# Tenants
Who owns the data, and how a request finds out
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's tenant rules were among its best ideas: a tenant comes only from the domain a request arrived on, a domain belongs to exactly one tenant, and nothing a caller types can choose a tenant. What went wrong was the number of databases hanging off a tenant. Every strategy could name its own database, a database could belong to a fixed tenant no domain mapped to, and so a tenant was a directory of files whose shape depended on every box ever deployed for it.

Homestead keeps the rules and collapses the shape: **one tenant is one website, and one database.** A tenant is a single site, like `example.com`, not a business: a business with two products has two tenants. Everything a tenant owns in pond (its buckets, their documents, its names, the tables tailed from them, and its change feed) lives in one SQLite file, so a tenant is the unit of isolation, ordering, placement, backup, and deletion, all at once.

## A tenant

| field | example | meaning |
| --- | --- | --- |
| name | bobs_plumbing | A chosen name, `[a-z][a-z0-9_]*`, at most 63 characters, never starting with `_`. Unique forever: a deleted tenant's name is not reused. |
| id | "4294967311" | A globally unique id from pond's counter, like everything else. |
| primary domain | bobsplumbing.com | The one name the tenant is known by: its website. |
| state | active, suspended, deleted | Suspended refuses writes and serves reads; deleted is gone. |

## One tenant, one website

A tenant is a website, and its domains are that site's names:

| domain | what it is |
| --- | --- |
| `example.com` | The primary domain: the product, in production. |
| `www.example.com` | A redirect to the primary domain, and any other alias like it. |
| `beta.example.com` | The product in beta, where every change lands first (see [rancher](../rancher/IDEA.md#beta-and-production)). |
| `ops.example.com` | The operating surface for the tenant's owner, operators, and agents. |

All of them are the same tenant, the same realm, and the same database. Beta and production differ in which box versions and routes answer, not in whose data they read. Ownership across tenants (one owner with several sites) isn't modeled; each tenant has its own owners in its own realm.

Tenants are created through rancher, which records the tenant as configuration and asks pond to create its database and permit to create its realm. Pond holds the tenant's data, permit holds its identities and grants in its one database (keyed by realm, shared by every tenant), and rancher holds everything about how the tenant is reached.

## One tenant, one database

```
pond storage/
  _pond.db                       management: tenants, buckets, tailers, grants, ids
  wal/                           the write-ahead log, shared by every tenant
  tenants/
    bobs_plumbing.db             everything bobs_plumbing owns in pond
    annes_landscaping.db         everything annes_landscaping owns
```

Inside a tenant's database:

| table | holds |
| --- | --- |
| `_buckets` | Every bucket the tenant has. A bucket is one row here, which is why buckets are cheap. |
| `_documents` | Every document in every bucket, keyed by bucket and id. |
| `_names` | Strings bound to ids, shared by every bucket and table in the tenant. |
| `_changes` | The tenant's change feed: seq, bucket, id, and op, kept for a window. |
| `_idempotency` | Recent idempotency keys and the seq each produced. |
| `_cursors` | How far each tailer has applied. |
| chosen names | The versioned tables tailers build from buckets, like `jobs@2` and `users@1`. |

What one database buys:

- **No cross-tenant query can be written.** A query runs on a connection to one tenant's file, so there is nothing else for it to name.
- **One tenant, one sequence.** Every write a tenant receives gets the next number in that tenant's seq, so a tenant's history has one order and subscribers see it in that order.
- **Moving a tenant is moving a file and a tail.** That is what makes the later cluster design ("a tenant lives on a single machine") cheap.
- **Forgetting a tenant is deleting a file**, plus its rows in permit, its WAL records, and its backups.
- **Quotas have one place to measure**: the file's size and its documents' count.

**How large can a tenant get?** As large as one machine. These are not sprawling products, so one SQLite file on one machine is enough for any tenant we expect. When a tenant outgrows it, it gets a bigger database, and that is a problem to meet when one arrives.

What it costs: a product that wants data shared across tenants has to put it in a tenant of its own and read it through a separate grant. That is deliberate. Goober's fixed-tenant databases were how shared data quietly crept in.

## Deletion

Deletion removes rows. It isn't a tombstone or a soft delete.

- **Every write is annotated with a user id.** Pond's WAL records, with each write, the user id it was made for (`!user_id`), or 0 when the write is the system's or the tenant's own. The annotation flows with the write into `_changes` and every tailed table that keeps it.
- **Deleting a user deletes everything written for them.** A bulk delete by user id removes that user's documents from every bucket, and the deletion flows from pond to every table tailed from them, to permit's rows for that user, and to the associated databases of other services, through the same feeds that carry any other change.
- **Deleting a tenant deletes its file**, its rows in permit, and its configuration in rancher.
- **Backups age out.** A deleted row survives only in backups and shipped WAL segments, until they pass their retention.

## Rules carried from goober

- **A domain belongs to exactly one tenant.** The registry of domains lives in rancher, and barns hold a copy.
- **A tenant has exactly one primary domain.** Every alias it owns (like `www.`) redirects there, so links, cookies, passkeys, and mail agree on one name. Its `beta.` and `ops.` subdomains are working surfaces of the same tenant, not aliases.
- **Callers can't choose their tenant.** The tenant comes from the domain registry, for the domain a request arrived on. Nothing in a path, query, header, or body sets it.
- **The tenant is a reserved value.** Written `!tenant`, set only by barn after it has resolved the domain, and passed to pond by barn, never by a box. A box asks pond for "the jobs bucket," and barn decides whose.
- **System names start with `_`.** A tenant can't be named `_pond`, and a system table can't collide with a tenant's table.
- **A tenant is an identity realm.** Its accounts, sessions, and grants live in permit and never cross into another tenant.
- **Host operators aren't tenant members.** Operator power is its own namespace (goober's `host:` roles), it is audited per tenant reached, and it never satisfies a tenant's own grants.
- **Domains can require hard verification.** A shared deployment proves control of a domain through DNS before serving it, so a forgotten domain can't be taken over.
- **Quotas belong to operators.** Documents, bytes, and request rates per tenant are set by operators in rancher, with red lines at 80% and 95%, and enforced in pond and barn at 100%.

## Rules that are new

- **A bucket belongs to exactly one tenant.** There are no shared buckets.
- **An id lives in one tenant, and may appear in many of its buckets.** A user's id keys `profiles`, `addresses`, and `settings` alike, but never a bucket in another tenant.
- **A tenant has one database.** Not one per box, strategy, or product. A second product is a second website, and so a second tenant.
- **Pond never infers a tenant.** Every call to pond names its tenant explicitly, from a trusted caller (barn or rancher), and pond refuses a call whose tenant doesn't exist or is suspended for that operation.

## The invariants

- **A tenant comes only from the domain registry.** No caller can choose one.
- **A domain belongs to exactly one tenant, and a tenant has exactly one primary domain.**
- **One tenant, one website.** Its primary domain, its redirects, beta, and ops all reach the same tenant.
- **One tenant, one database, one sequence.** A tenant is the unit of isolation, order, placement, backup, and deletion.
- **A bucket, and every key in it, belongs to exactly one tenant.**
- **A tenant is one realm in permit.** Identity never crosses tenants.
- **Operator power is separate and audited.** It never passes for a tenant's own permission.
- **Deletion removes rows.** Every write carries its user id (0 for the system or tenant), so deleting a user deletes what was written for them, everywhere it flowed.
