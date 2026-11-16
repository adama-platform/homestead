# Pond
Cheap buckets of documents, a WAL, and SQLite tables tailed from them
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober tried to be the whole operating system in one binary, and data was where that hurt the most. Every box declared strategies, schemas, and migrations, and the host had to understand all of them at once, so a change to how data was stored was a change to the manifest, the checker, the generator, and the host together. The good parts were buried in there anyway: ids unique everywhere, writes that merge rather than replace, append-only history that is never migrated, and views rebuilt by replaying what happened. Pond takes those parts and makes them the whole data layer.

Pond does two things. It makes it easy to **capture** domain data: a tenant's data lives in buckets that are as cheap as S3 buckets, a bucket holds documents keyed by ids, and documents only change by MERGE and DELETE. And it makes it easy to **ask** questions of that data: tailers project one or more buckets into ordinary SQLite tables, in as many versions as are useful, which can be dropped and rebuilt on the fly, and boxes query them with read-only SQL that pond checks with fork and runs itself.

Pond owns everything about data. Connections, transactions, durability, ordering, change feeds, and ids all live behind pond's API, and every query is stateless. The write path belongs to pond alone; the only SQL anyone else can run is a read.

## The shape

```
     MERGE / DELETE (from barn)                  SELECT (from barn)
              |                                        |
              v                                        v
   +-----------------------+                 +-------------------+
   | WAL (fsync, then ack) |                 | fork: parse+check |
   +-----------------------+                 +-------------------+
              |                                        |
              v                                        v
   +------------------------------------------------------------+
   | tenant database (one SQLite file per tenant)               |
   |   _buckets     name -> bucket number, kind                 |
   |   _documents   (bucket, id) -> doc, seq                    |
   |   _names       string -> id                                |
   |   _changes     seq, bucket, id, op, user --> pubsub feed   |
   |   _outbox      effects committed with a write, until done  |
   |   tables       name@version, tailed from 1+ buckets        |
   +------------------------------------------------------------+

   _pond.db   tenants, tailer definitions, the id counter and ranges
```

| piece | what it is | what it promises |
| --- | --- | --- |
| A bucket | A named collection of documents in one tenant | Documents in it change only by MERGE and DELETE. A bucket costs one row until it holds something. |
| An id | A 64-bit integer from pond's counter | It names one thing (a user, a job) anywhere. Many buckets can hold a document for the same id. |
| The WAL | An append-only log of every mutation, shared by every tenant on the machine | A write acknowledged is a write that survives a crash. |
| The documents | `_documents`: the current fold of every MERGE per bucket and id | Reading a key gives the result of every acknowledged write to it, in order. |
| A tailer | A projection from one or more buckets into one versioned SQLite table | The table is a function of the documents, and can be rebuilt at any time. |
| The feed | The tenant's change log: which keys changed, in order | A subscriber sees every change once, in seq order, or learns it fell behind. |
| The id service | The counter, the ranges it vends, and each tenant's names | An id is minted once; a name always means the same id while it is bound. |
| The outbox | Effects on other services (a grant, a schedule, a mail), committed in the same write as the data | An effect exists if and only if its write landed, and is delivered at least once. |

## Buckets are cheap

A bucket is a name in one tenant, like `profiles` in `bobs_plumbing`. It is a row in the tenant's `_buckets` table and nothing else: every bucket's documents live in the one `_documents` table, keyed by bucket number and id, so a tenant can hold ten buckets or ten thousand and an empty bucket costs nothing.

- **A bucket exists once it is written or configured.** A write to a bucket the tenant doesn't have yet creates it, in the same WAL record. Barn only writes buckets a box's write ruleset names (see barn), so the ruleset is the gate, not a creation step.
- **A bucket has no schema.** A schema is a promise about the writers, and pond only promises things about the writes. Shape lives in tailer definitions, where it can change without touching a document.
- **Dropping a bucket is one write.** Its documents are deleted in the background; its name is tombstoned.
- **A bucket has a kind.** `document` is the first and only kind to start. `log` (append-only events) and `cache` (a time to live, not in the WAL) are candidates for later. Timers are not a bucket kind; they belong to [sun](../sun/IDEA.md).

## Keys: buckets share ids

A key is a bucket and an id. The id comes from pond's counter, is unique across every tenant, and is a string in JSON (see [common](../common.md#ids)). Because the key includes the bucket, **many buckets can hold a document for the same id**. That is how an entity is spread across buckets by facet:

```
id 4294967311 (a user)
  profiles/4294967311    { "name": "Alice", "avatar": "..." }
  addresses/4294967311   { "street": "1 Main St", "city": "Tacoma" }
  settings/4294967311    { "theme": "dark" }
```

Each facet is written by the routes that own it, can be granted on its own, and is read without loading the others. A table can join them back together by id (see [Tables from several buckets](#tables-from-several-buckets)).

- **An id lives in one tenant.** A key only ever exists in the tenant it was first written in.
- **Ids are minted, never chosen.** Pond refuses a write to an id it never vended. Which ids a box may write is decided by routing, per object, at runtime (see barn and rancher).
- **A deleted key is dead.** DELETE removes a bucket's document for an id. Writing to that same key again is refused; the id lives on in its other buckets.

## Names: strings to ids

Plenty of things already have a name: an email address, a SKU, a Stripe customer id, a slug. Pond's id service turns a name into an id, once, forever, so a route can address "the customer whose email is bob@example.com" as a key.

```json
{ "tenant": "bobs_plumbing", "names": ["email:bob@example.com", "sku:A-100"], "create": true }
```

```json
{ "ids": { "email:bob@example.com": "4294967390", "sku:A-100": "4294967391" } }
```

- **One table per tenant, shared by every bucket and table.** `_names` maps a name to an id, so `email:bob@example.com` is the same id whether it is used in `customers`, `invoices`, or a table joining them.
- **A name never changes meaning.** Once bound, a name's id is permanent for as long as the name exists, which makes the mapping safe to cache.
- **Names make creates idempotent.** "Create the customer for this email" run twice resolves the same id both times and merges into the same document, so a retried request can't make a duplicate.
- **Names are forgotten with their user.** A name bound for a user carries that user's id, and deleting the user deletes the name's row (see [Deletion](#deletion)). Its id is never vended again, so nothing that cached the old mapping can land on someone else's data; if the same string is bound later, it gets a fresh id.
- **Names are namespaced by convention.** `kind:value`, so `email:` and `sku:` never collide. Pond stores the exact bytes; normalizing (lowercasing an email) is the caller's choice, made once in the route.
- **Binding a name is a write.** It goes through the WAL with the tenant's next seq, so a crash can't forget a name that was answered.

## Writes

Every mutation is a MERGE or a DELETE. A batch of them for one tenant is one write: it lands in the WAL as one record, gets one seq, and is applied all or nothing.

```json
{
	"tenant": "bobs_plumbing",
	"idempotency": "req-5f2a9c",
	"user": "4294967311",
	"ops": [
		{ "op": "merge", "bucket": "addresses", "id": "4294967311", "if-seq": 4471,
		  "doc": { "city": "Tacoma", "zip": "98402" } },
		{ "op": "merge", "bucket": "jobs", "id": "4294967312", "if-seq": 0,
		  "doc": { "status": "open", "customer_id": "4294967311" } },
		{ "op": "delete", "bucket": "drafts", "id": "4294967305" }
	],
	"hold": { "tables": ["users@1"], "timeout-ms": 500 }
}
```

```json
{ "seq": 4472, "held": true }
```

| field | meaning |
| --- | --- |
| op | `merge` or `delete`. There is no put that replaces a document. |
| doc | A plain JSON merge, RFC 7386: objects merge recursively, values (arrays included) replace, and null removes a field. A MERGE to a key with no document creates it. |
| if-seq | Optional. The op only lands if the key's last write is still at this seq; 0 means "only if it doesn't exist yet." Barn sets it from the seq it read, so every box write is compare-and-set. |
| idempotency | A key the writer chooses, so a retry is a no-op that answers the original seq. |
| user | The user id the write was made for (barn's `!user_id`), or 0 when it is the system's or the tenant's own. Recorded in the WAL and `_changes`, and what a delete by user follows (see [Deletion](#deletion)). |
| effects | Optional. Outbox elements committed with the write: grants for permit, jobs for sun, mail for barker (see [The outbox](#the-outbox)). |
| hold | Optional. Answer only after these tables have applied this write, or after the timeout, saying which. |

A merge is the right default because old code and new code can coexist: an old writer that never heard of a field can't erase it. RFC 7386 replaces arrays whole, and that is accepted: pond assumes most documents have a single writer within their domain (one user's profile, one job's owner), so two writers adding to one list at once is rare, and compare-and-set catches it when it happens. There are no counters, sets, or list merges.

## Durability

Pond is crash-proof by its WAL. A write's life:

1. **Validate.** The tenant accepts writes, every id was vended, no key is dead, every `if-seq` matches, and the idempotency key is new.
2. **Append.** The write is appended to the WAL with the tenant's next seq. Writes from many tenants share fsyncs (group commit), which is where vertical scale comes from.
3. **Acknowledge.** Once the fsync returns, the write is durable. Pond answers then, or after its hold.
4. **Apply.** The tenant's applier folds the write into `_buckets`, `_documents`, `_names`, and `_changes` in one SQLite transaction, and records the WAL position it reached.
5. **Tail.** Tailers pick the change up from `_changes`, asynchronously, and update their tables.
6. **Truncate.** A WAL segment is deleted once every tenant has applied past it and checkpointed, and once it has been shipped (see [Backups](#backups)).

After a crash, pond replays the WAL from each tenant's recorded position. Applying and tailing are idempotent by seq, so replay is safe to repeat.

## Backups

Every tenant is snapshotted nightly, and the WAL is shipped continuously to object storage between snapshots, so a tenant can be restored to its last snapshot plus every shipped write after it. Restoring a tenant is copying its snapshot file and replaying the shipped WAL from the snapshot's seq, the same replay a crash uses.

Pond is the service that holds the data, so it is the one that should restart least. Deploys know which services changed and restart only those (see [common](../common.md#isolation)); pond's shutdown flushes its WAL, checkpoints each tenant, and hands off cleanly, and its code is kept narrow so that most deploys never touch it.

## Deletion

Deletion removes rows. Every WAL record carries the user id the write was made for, or 0 for the system or the tenant, so pond can answer "everything written for this user":

| call | does |
| --- | --- |
| `delete_user {tenant, user}` | Deletes every document written for that user, in every bucket, and every name bound for them, as one write with its own seq. |

The deletion flows like any other change: tailers drop the rows, the feed names the deleted keys, and the services that keep their own copies (permit, sun, barker) follow the feed and remove theirs. What remains is in nightly snapshots and shipped WAL segments, until they age out of retention.

## Tables

A tailer turns documents into a SQLite table that fork and SQLite can query. A table has a name and a version, like `jobs@2`, and several versions of a table can exist at once.

```json
{
	"table": "jobs",
	"version": 2,
	"from": { "job": "jobs" },
	"columns": {
		"id": { "type": "id" },
		"status": { "from": "job:/status", "type": "text" },
		"customer_id": { "from": "job:/customer_id", "type": "id" },
		"finished_at": { "from": "job:/finished_at", "type": "int64" }
	},
	"indexes": { "by_status": ["status", "finished_at"] }
}
```

- **Tables are disposable.** A table is a function of its buckets, so pond can drop it and rebuild it from `_documents` at any time.
- **Versions live side by side.** A new shape is a new version, built beside the old one from `_documents` and caught up from `_changes`. Routes choose which version their queries read, so `jobs@1` serves old box versions while `jobs@2` serves new ones, and the old version is dropped when no route names it. There are no migrations, only rebuilds.
- **Only tailers write tables.** No SQL from outside pond can write one.
- **A tailer has a cursor.** Each table knows the seq it has applied, so a read can wait for exactly the tables it needs.
- **One document, one row.** A tailer turns each document (or each id, across joined buckets) into at most one row. Exploding an array into child rows (a job's line items) is on the roadmap, and waits for a real need.
- **A document that doesn't fit is a problem, not a stop.** A missing field is null; a field of the wrong type is null and is reported as a problem (see [Problems](#problems)).

### Tables from several buckets

Since buckets share ids, a table can join several buckets by id, and pond keeps the join materialized:

```json
{
	"table": "users",
	"version": 1,
	"from": { "profile": "profiles", "address": "addresses", "settings": "settings" },
	"rows": "profile",
	"columns": {
		"id": { "type": "id" },
		"name": { "from": "profile:/name", "type": "text" },
		"city": { "from": "address:/city", "type": "text" },
		"theme": { "from": "settings:/theme", "type": "text", "default": "light" }
	}
}
```

`rows` names the bucket whose ids decide which rows exist (here, one row per profile), or `any` for a row whenever any bucket has the id. A change to any source bucket recomputes only that id's row, so the join costs one lookup per bucket per change, never a scan. This is where the "user -> profile, user -> address" shape pays off: the facets are written separately and read together.

## Queries

A query is stateless: it carries everything pond needs, including the grant routing gave it, and pond holds nothing open afterward.

```json
{
	"tenant": "bobs_plumbing",
	"grant": {
		"tables": { "jobs": { "version": 2, "columns": ["id", "status", "finished_at"] } },
		"limits": { "max-rows": 1000, "max-ms": 50 }
	},
	"sql": "SELECT id, status FROM jobs WHERE status = :status ORDER BY finished_at DESC",
	"params": { "status": "open" },
	"reserved": { "!user_id": "4294967100" },
	"min-seq": 4472
}
```

1. **Check.** Fork parses the SQL against the grant, maps each table to its version, and returns canonical SQL and the tables it reads, or a refusal.
2. **Wait.** With `min-seq`, pond waits, up to a bound, until every table the query reads has applied that seq.
3. **Run.** Pond runs fork's SQL on a pooled read-only connection to the tenant's database, with SQLite's authorizer, a row cap, and a time cap.
4. **Answer.** Rows, with ids as strings, and the seq the answer reflects.

Pond doesn't store grants. Routing owns them (see rancher), barn sends the grant with each query, and pond trusts barn as it trusts it with the tenant. Reading a key doesn't need SQL: `get` returns a bucket's document for an id, and its seq.

### Named queries

Most of a route's queries are known when it is deployed, so a route registers them by name, and rancher checks each one with fork once, at deploy:

```json
"queries": {
	"open_jobs": "SELECT id, status FROM jobs WHERE status = 'open' AND customer_id = :!user_id ORDER BY finished_at DESC"
}
```

Pond receives the checked, canonical SQL with the route's configuration, prepares each named query once per tenant connection, and a box runs one by name with its parameters. A named query skips parsing and checking on every request, and its plan is cached. Ad-hoc SQL stays available, checked on every call, for agents exploring and for queries a box builds at runtime.

## Waiting for tailers

Tailers are asynchronous, so a write is durable before its tables catch up. Anyone who needs the tables current can wait:

| how | when to use it |
| --- | --- |
| `hold` on a write | Answer the write only once the named tables have applied it. Barn uses this when a route holds its response. |
| `min-seq` on a query | Read tables at least as new as a write the caller already saw. |
| `wait {tenant, seq, tables, timeout-ms}` | Block until the tables reach a seq, without reading. |

Every wait is bounded. A wait that times out answers that it timed out and how far each table got, and is reported as a problem when it keeps happening, since a tailer that can't keep up is something an operator or agent should see.

## Pubsub

Each tenant's `_changes` table is its change feed, and pond publishes it. A subscriber names a tenant, the buckets it cares about, and the seq to start from:

```
feed.subscribe { tenant: bobs_plumbing, buckets: [jobs, addresses], from: 4470 }   (a wire stream)
feed.subscribe { tenant: bobs_plumbing, buckets: [jobs], ids: [4294967312], from: 4470 }

{ "seq": 4471, "bucket": "addresses", "id": "4294967311", "op": "merge" }
{ "seq": 4472, "bucket": "jobs", "id": "4294967312", "op": "merge" }
{ "seq": 4472, "bucket": "drafts", "id": "4294967305", "op": "delete" }
{ "seq": 4473, "table": "users@1", "applied": 4472 }
```

- **Filter by bucket, then by id.** A subscription names the buckets it wants, and optionally the exact ids within them, so an SSE client watching one job gets that job's changes and nothing else from a busy bucket. Watching one entity is the common real-time case, so the id filter is applied in pond, not by the subscriber.
- **Changes name keys, not contents.** A subscriber that wants the document reads it. The feed stays small and can't leak a field a reader wasn't granted.
- **Tables report progress.** A subscriber can ask for table progress too, which is how barn learns a cached query answer is stale and when a held response can go.
- **Resuming is by seq.** `_changes` is kept for a window, and a subscriber asking for a seq older than the window gets a `reset`, which tells it to drop everything it cached and start from now.

## The outbox

A request can do more than write data: it can grant a role in permit, schedule a job in sun, or send mail through barker. Those are other services, and they don't share a transaction with pond. Goober's answer was the outbox, and homestead keeps it: **an effect is written into pond in the same write as the data**, so it exists if and only if the write landed.

```json
"effects": [
	{ "to": "permit", "op": "grant", "subject": "4294967100", "grant": "editor/project_42" },
	{ "to": "sun", "op": "schedule", "route": "timer:send_reminder", "at": 1762900000000, "payload": { "job_id": "4294967312" } },
	{ "to": "barker", "op": "send", "template": "job_opened", "to-user": "4294967311" }
]
```

- **One write, one bundle.** The write's ops and its effects land in the same WAL record and the same seq. If the box fails, neither happens.
- **Durable before acknowledged.** Because a committed bundle is a promise to the world, its WAL record is chain-replicated to a small chain of pond replicas (two or three machines) before pond acknowledges it, so losing the pond machine can't lose an effect it promised. The same chain is a warm standby for the tenant's data.
- **Pond coordinates; barn executes.** Committed effects go into the tenant's `_outbox` table. Barns drain it under a lease, call permit, sun, or barker, and mark each element done. A barn that dies mid-delivery loses its lease, and another barn picks the element up.
- **At least once, made once.** Every element carries an idempotency key derived from the write's seq and its position in the bundle, and permit, sun, and barker deduplicate by it.
- **In order per scope.** Elements to the same service for the same subject are delivered in commit order.
- **Every element ends in exactly one place.** Delivered, or dead lettered after its deadline with every attempt recorded, and raised as a problem.

This makes pond the transaction coordinator for the other services, which is deliberate: pond already owns the only durable order a tenant has.

## The id service

Pond's management database holds goober's counter and every range vended:

```sql
CREATE TABLE _id_counter (next_id INTEGER NOT NULL);       -- starts at 2^32 + 1
CREATE TABLE _id_ranges (
	id_start INTEGER PRIMARY KEY,
	length INTEGER NOT NULL,
	vended_to TEXT NOT NULL,                                -- the barn host
	vended_at INTEGER NOT NULL
);
```

A barn host leases a range (like 10,000 ids) and mints from it in memory. The counter moves forward in the same transaction that records the range, before the range is returned. A restore moves the counter past the highest id in any restored tenant plus a safety margin, so no id that ever left pond is minted again. Names (above) mint from the same counter.

## Problems

Pond doesn't throw away what goes wrong. A document a tailer can't fit, a write refused for a stale `if-seq`, a query fork refused, a hold that timed out, or a tailer falling behind is a problem record, reported up to rancher's problem log, as described in [problems](../common/PROBLEMS.md). Each one names the table, column, bucket, and seq involved, so an agent can see what broke and why without reading pond's logs.

## The control layer

Pond's control API manages `_pond.db`, and only rancher calls it.

| call | does |
| --- | --- |
| tenant create, suspend, delete | Creates or removes the tenant's database file and its row. |
| user delete | Deletes everything written for one user id (see [Deletion](#deletion)). |
| queries put | Prepares a route's named queries, already checked by fork in rancher. |
| bucket configure, drop | Sets a bucket's kind and retention, or drops it with one write. |
| table put, drop, rebuild | Defines a table version from one or more buckets, builds it, drops it. |
| status | Per tenant: seq, buckets, documents, bytes, each table's lag, the WAL position. |

## Scale

A tenant is bound to one machine, and that is the plan, not a stage: these products are not sprawling, and a tenant that outgrows one machine gets a bigger database. Pond starts as one machine, scaled vertically: one WAL with group commit, one applier per active tenant, a pool of read connections per tenant, and every tenant in its own file. The path to a cluster follows from "one tenant, one database": a tenant lives on exactly one pond machine at a time, owned under a lease with an epoch, and moving a tenant means copying its file, replaying its WAL tail on the new machine, and flipping the lease. Barns find a tenant's pond from a placement map rancher publishes.

## What pond is not

- **Not a SQL write path.** Writes are MERGE and DELETE on keys. SQL reads tables.
- **Not a schema registry.** Shapes live in table versions, and a new shape is a new version.
- **Not an authority on who may do what.** Routing decides which keys a box may write and which tables it may read; permit decides who a caller is. Pond trusts barn and rancher.
- **Not a place for blobs.** Large files go to object storage, and a document holds a reference.
- **Not stateful for its readers.** No sessions, cursors, or open transactions survive a call.

## Inspirations

| thing | what we take from it |
| --- | --- |
| Goober's primary keys | One counter, ranges leased once, ids unique everywhere and strings in JSON. |
| Goober's sources and event routes | Append-only truth, and new views made by replaying it. |
| Amazon S3 | Buckets so cheap nobody thinks twice about making one. |
| RFC 7386 JSON Merge Patch | The only merge, because it is simple and every agent already knows it. |
| Goober's outbox | Effects committed with the data, delivered at least once with stable keys. |
| Chain replication | Durability for the bundle of effects, without a consensus protocol. |
| Kafka compacted topics | A change log keyed by key, consumed from an offset. |
| Entity-component systems | One id, many facets, each stored and changed on its own. |

## Open questions

1. **How long is history kept?** The WAL is truncated once applied and shipped, and `_changes` is kept for a window. Keeping every write forever (WAL segments in object storage) is valuable and is also a privacy problem.

## The invariants

- **Writes are MERGE and DELETE.** No replace, and no SQL writes.
- **A write acknowledged has been fsynced to the WAL.** A crash loses nothing that was acknowledged.
- **One tenant, one database, one sequence.** The seq is the order every reader sees.
- **A key is a bucket and an id.** Buckets share ids; ids are minted, never chosen; a deleted key is never reused.
- **Buckets are cheap.** An empty bucket costs a row, and creating one is a write.
- **A name means one id while it is bound.** Within its tenant, shared by every bucket and table; an id is never vended twice.
- **Tables are a function of buckets.** Versioned, rebuilt rather than migrated, and written only by tailers.
- **Queries are read-only, checked by fork, and stateless.** Pond owns every connection.
- **Every wait is bounded.** And a wait that keeps timing out is a problem someone sees.
- **Only rancher calls the control API.**
- **Every write names its user.** 0 for the system or tenant; deleting a user deletes what was written for them.
- **An effect exists if and only if its write landed.** Effects are committed with the data, chain-replicated before acknowledgment, and delivered at least once by barn.
