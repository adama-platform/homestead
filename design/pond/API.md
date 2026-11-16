# Pond API
Every call pond answers, who may make it, and what comes back
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Pond's API is the schema in `pond_proto` plus the `Pond` interface in `pond_client`. It has four faces, and each has its own short list of callers: the **data API** that barn (and rancher) use on every request, the **service API** that permit, sun, and barker use for ids, outbox elements, and deletions, the **replica API** that other ponds use, and the **control API** that only rancher uses. Every method rides the internal protocol described in [network](../common/NETWORK.md): a deadline on every call, an idempotency key on every call that changes state, and subscriptions that resume by seq.

Examples are written in flatc's JSON form of the FlatBuffers messages. Ids are int64 on the wire; flatc prints them as numbers, and the client hands them to JSON callers as strings.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `pond.get` | call | barn, rancher | no | Documents and their seqs, for keys |
| `pond.write` | call | barn, rancher | yes | The write's seq, and how a hold went |
| `pond.query` | call | barn, rancher | no | Rows, the tables read, the seq reflected |
| `pond.names` | call | barn, rancher | yes, with `create` | Ids for names |
| `pond.wait` | call | barn, rancher | no | Whether tables reached a seq, and how far each got |
| `pond.ids.lease` | call | barn, rancher, permit, sun, barker | yes | A range of fresh ids |
| `pond.feed.subscribe` | stream | barn, rancher | no | Key changes and table progress, in seq order |
| `pond.events.subscribe` | stream | barn | yes (leases) | Events for event routes, leased to this barn, as credit allows |
| `pond.events.ack` | call | barn | yes | Events marked done, failed, or skipped |
| `pond.outbox.lease` | stream | permit, sun, barker (Q12.1) | yes | Outbox elements addressed to the caller, under a lease |
| `pond.outbox.ack` | call | permit, sun, barker (Q12.1) | yes | Elements marked delivered or dead |
| `pond.deletions.subscribe` | stream | permit, sun, barker (Q12.2) | no | User and tenant deletions, in order |
| `pond.replica.append` | push | pond | yes | The highest WAL position made durable on the replica |
| `pond.control.tenant.create`, `.suspend`, `.resume`, `.delete` | call | rancher | yes | The tenant's state |
| `pond.control.user.delete` | call | rancher | yes | The deletion's seq |
| `pond.control.queries.put` | call | rancher | yes | The named queries prepared |
| `pond.control.bucket.configure`, `.drop` | call | rancher | yes | The bucket's state, or the drop's seq |
| `pond.control.table.put`, `.drop`, `.rebuild` | call | rancher | yes | The table version's state |
| `pond.control.events.put`, `.drop` | call | rancher | yes | The event route's cursor state |
| `pond.control.events.dead.list`, `.redrive`, `.discard` | call | rancher | list no; others yes | Dead events, or how many were redriven or discarded |
| `pond.control.status` | call | rancher | no | Per tenant: seq, sizes, table lag, event route lag, WAL position |

`pond.control.tenant.resume` is the inverse of suspend; the IDEA's control table implies it. Every call except `get`, `query`, `wait`, `status`, and the subscriptions requires `Open.idempotency`.

## Common fields

| field | on | meaning |
| --- | --- | --- |
| tenant | every tenant call | The tenant by name. Pond refuses a tenant that doesn't exist (`not_found`), is deleted (`gone`), or is suspended for this operation (`refused`, problem `pond.tenant_suspended`). |
| user | writes | The user id the write is for (`!user_id`), or 0 for the system or the tenant. |
| seq | answers | The tenant's sequence number the answer reflects. |

The tenant in a payload is trusted only from barn or rancher (see [network](../common/NETWORK.md#security)); the service API derives the tenant from the element or deletion it hands out, never from the caller.

## Reading keys: `pond.get`

```json
{ "tenant": "bobs_plumbing",
  "keys": [ { "bucket": "addresses", "id": 4294967311 }, { "bucket": "profiles", "id": 4294967311 } ] }
```

```json
{ "seq": 4472,
  "docs": [
    { "bucket": "addresses", "id": 4294967311, "seq": 4471, "state": "live", "doc": "{\"city\":\"Tacoma\"}" },
    { "bucket": "profiles", "id": 4294967311, "seq": 0, "state": "absent" } ] }
```

| field | meaning |
| --- | --- |
| keys | At most 64 per call. |
| state | `live`, `absent` (never written; seq 0, so `if-seq: 0` creates it), or `dead` (deleted; a write is refused with `gone`). |
| doc | JSON bytes, opaque to the wire. |
| min-seq | Optional. Answer only once the tenant has applied this seq, bounded by the deadline. |

## Writing: `pond.write`

```json
{ "tenant": "bobs_plumbing", "user": 4294967311,
  "ops": [
    { "op": "merge", "bucket": "addresses", "id": 4294967311, "if_seq": 4471, "doc": "{\"zip\":\"98402\"}" },
    { "op": "merge", "bucket": "jobs", "id": 4294967312, "if_seq": 0, "doc": "{\"status\":\"open\"}" },
    { "op": "delete", "bucket": "drafts", "id": 4294967305 } ],
  "names": [ { "name": "email:bob@example.com", "id": 4294967390 } ],
  "effects": [
    { "to": "sun", "scope": "reminder/4294967312", "body": "{\"op\":\"schedule\",\"route\":\"timer:send_reminder\",\"at\":1762900000000}" } ],
  "hold": { "tables": ["users@1"], "timeout_ms": 500 } }
```

```json
{ "seq": 4472, "hold": { "held": true, "tables": [ { "table": "users@1", "applied": 4472 } ] } }
```

| field | meaning |
| --- | --- |
| ops | MERGE (RFC 7386) or DELETE, at most 256 per write, one WAL record, one seq, all or nothing. |
| if_seq | Optional per op; 0 means "only if absent." A mismatch refuses the whole write with `conflict` and problem `pond.if_seq_stale`; `detail` names the key and its current seq. |
| names | Bindings minted by `pond.names` in the same request and committed with this write, so a name and the document it keys land together. |
| effects | Outbox elements. `to` is `permit`, `sun`, or `barker`; `scope` orders elements per consumer (commit order within a scope); `body` is the consumer's own message, opaque to pond. |
| hold | Answer once these tables applied the write, or at `timeout_ms`, saying which. A timeout is still `ok`, with `held: false`, and is reported as `pond.hold_timeout`. |
| `Open.idempotency` | The write's key. A repeat answers the original seq. Barn uses the request id; sun's jobs use the job id. |

Bucket names starting with `_` are refused with `invalid` unless the write comes from barn's system path (`system: true`, which barn sets only for `_credentials` writes it makes itself, never for a box's ops).

Refusals: `invalid` (malformed patch, too many ops, a bucket name that breaks the name rules), `refused` with `pond.id_not_vended` (an id pond never leased out), `gone` with `pond.key_dead`, `conflict`, `refused` with `pond.tenant_suspended`, `overloaded` when the tenant's write queue is full.

An acknowledged write has been fsynced to the WAL and, when it carries effects, appended on the replica chain.

## Querying: `pond.query`

```json
{ "tenant": "bobs_plumbing",
  "grant": "{\"tables\":{\"jobs\":{\"version\":2,\"columns\":[\"id\",\"status\",\"finished_at\"]}},\"limits\":{\"max-rows\":1000,\"max-ms\":50}}",
  "sql": "SELECT id, status FROM jobs WHERE status = :status ORDER BY finished_at DESC",
  "params": [ { "name": "status", "text": "open" } ],
  "reserved": [ { "name": "!user_id", "int": 4294967100 } ],
  "min_seq": 4472 }
```

A named query sends `"named": { "route": "PUT bobsplumbing.com/me/address", "config_seq": 8815, "name": "me" }` instead of `grant` and `sql`; pond runs the canonical SQL rancher registered with `pond.control.queries.put`.

```json
{ "seq": 4472, "tables": [ { "table": "jobs@2", "applied": 4472 } ],
  "columns": ["id", "status"],
  "rows": [ { "cells": [ { "id": 4294967312 }, { "text": "open" } ] } ],
  "truncated": false, "ms": 1.8 }
```

| field | meaning |
| --- | --- |
| grant | The route's grant, as fork's JSON. Pond stores no grants. |
| params, reserved | Typed bound values. A reserved name (`!user_id`, `!tenant`) in `params` is refused. |
| min_seq | Wait (bounded by the deadline and the grant's limits) until every table the query reads has applied this seq. |
| tables | What fork said the query reads, and how far each had applied; barn keys its cache invalidation on these. |
| truncated | The row cap cut the answer. |

Refusals: `refused` with fork's refusal code as the problem (`fork.column_not_granted`, ...) and fork's refusal in `detail`; `deadline` with `pond.query_timeout` when the grant's `max-ms` stopped it; `not_found` with `pond.named_query_unknown`.

## Names: `pond.names`

```json
{ "tenant": "bobs_plumbing", "names": ["email:bob@example.com", "sku:A-100"], "create": true, "user": 0 }
```

```json
{ "ids": [ { "name": "email:bob@example.com", "id": 4294967390, "created": false },
           { "name": "sku:A-100", "id": 4294967391, "created": true } ] }
```

Without `create`, an unbound name is absent from the answer. With `create`, new bindings are written through the WAL with the tenant's next seq before the answer, and `user` is what a later `user.delete` follows. Names are cacheable forever while bound.

## Waiting: `pond.wait`

```json
{ "tenant": "bobs_plumbing", "seq": 4472, "tables": ["users@1", "jobs@2"], "timeout_ms": 500 }
```

```json
{ "reached": false, "tables": [ { "table": "users@1", "applied": 4472 }, { "table": "jobs@2", "applied": 4469 } ] }
```

## Ids: `pond.ids.lease`

```json
{ "holder": "barn-7", "count": 10000 }
```

```json
{ "start": 4294980001, "count": 10000 }
```

`count` is at most 100,000. The counter moves forward durably before the answer; a repeat of the same idempotency key answers the same range. Ids are tenant-free: a range is leased to a host, and the first write that uses an id binds it to its tenant.

## The feed: `pond.feed.subscribe`

```json
{ "tenant": "bobs_plumbing", "buckets": ["jobs", "addresses"], "ids": [], "tables": ["users@1"], "from": 4470 }
```

Events, one per DATA message:

```json
{ "seq": 4471, "change": { "bucket": "addresses", "id": 4294967311, "op": "merge" } }
{ "seq": 4472, "change": { "bucket": "drafts", "id": 4294967305, "op": "delete" } }
{ "seq": 4472, "progress": { "table": "users@1", "applied": 4472 } }
{ "seq": 4480, "reset": { "oldest": 4400 } }
```

Filters apply in pond: buckets (empty means all), then ids. Changes name keys, never contents. A `from` older than `_changes` keeps answers one `reset` and continues from now. A barn opens one subscription per tenant it caches and widens it by reopening; at most 1,024 ids per subscription.

## Events: `pond.events.subscribe` and `pond.events.ack`

Pond is the event bus: an event route is a tailer over `_changes` whose sink is a box (see [pond](IDEA.md#event-routes)). A barn opens one subscription across every tenant, and its credit is how many events it can run at once.

```json
{ "holder": "barn-7", "lease_ms": 30000 }
```

Events on the stream, one per message, as credit allows:

```json
{ "event": { "lease": "e-91f2", "tenant": "bobs_plumbing", "route": "order_placed",
  "seq": 4472, "bucket": "orders", "id": 4294967312, "op": "merge", "user": 4294967311,
  "at": 1762870000123, "attempt": 1 } }
```

```json
{ "acks": [ { "lease": "e-91f2", "outcome": "done" },
            { "lease": "e-91f3", "outcome": "failed", "reason": "box.trap" },
            { "lease": "e-91f4", "outcome": "skipped", "reason": "route_not_live" } ] }
```

| field | meaning |
| --- | --- |
| route | The event route's name, as rancher put it. Barn resolves it to a service, export, objects, and grant from its own configuration. |
| seq, bucket, id, op, user, at | The change: a key, never a document. |
| lease | Held until acknowledged, `lease_ms` passes, or the stream closes; then the event goes to another barn. |
| outcome | `done` moves the cursor; `failed` retries after a backoff (30 s, doubling, at most 10 minutes), dead after 5 attempts; `skipped` (the route isn't live here, the tenant is suspended) is done without running and is logged. |

- **Order:** an event for (route, id) isn't handed out while an earlier one for the same route and id is leased or waiting. Different ids run in parallel.
- **Idempotency:** the box's write carries `event:{route}:{seq}`, so a redelivery can't write twice.
- **A dead event** is raised as `pond.event_dead` with the route, key, seq, and every attempt's reason, and waits for rancher.

## The service API

### The outbox (assuming Q12.1's lean: each consumer drains its own slice)

```json
{ "to": "barker", "lease_ms": 60000 }
```

Events on the stream, as credit allows:

```json
{ "element": { "tenant": "bobs_plumbing", "seq": 4472, "index": 0, "key": "4472/0",
  "scope": "mail/ann", "attempt": 1, "body": "{...}" } }
```

```json
{ "acks": [ { "tenant": "bobs_plumbing", "key": "4472/0", "outcome": "done" },
            { "tenant": "bobs_plumbing", "key": "4472/1", "outcome": "dead", "reason": "template_missing" } ] }
```

- **The stream holds the lease.** Elements on it are leased to the subscriber until `lease_ms` passes without an ack or the stream closes; then they are handed to the next subscriber for that `to`.
- **In order per scope.** An element is not handed out while an earlier element in the same tenant and scope is unacknowledged.
- **The key** is `seq/index`, stable across redeliveries; the consumer deduplicates by it.
- `dead` is the consumer saying the element can never be delivered; pond keeps it in `_outbox` as dead and raises `pond.outbox_dead`.

### Deletions (assuming Q12.2's lean: one cross-tenant stream)

```json
{ "from": 1201 }
```

```json
{ "seq": 1202, "user": { "tenant": "bobs_plumbing", "user": 4294967100, "tenant_seq": 5120 } }
{ "seq": 1203, "tenant": { "tenant": "annes_landscaping" } }
```

The deletions log has its own seq in `_pond.db`, kept for 30 days; a `from` older than that gets `reset`, and the subscriber must reconcile against pond (a reset here is a problem, `pond.deletions_reset`, since it can leave data behind).

## The replica API: `pond.replica.append`

A push from the chain head to the next pond. Each batch is WAL records with their positions; the replica appends and fsyncs, forwards down the chain, and acknowledges the highest position durable at the tail. The head acknowledges an effect-carrying write only after the tail acknowledged it. Only callers whose certificate names `pond` may push.

```json
{ "epoch": 3, "records": [ { "position": 918223, "bytes": "..." } ] }
```

```json
{ "durable": 918223 }
```

A record from an older epoch is refused with `refused` and problem `pond.replica_epoch_stale`.

## The control API

Only rancher. Every call is idempotent by key and by content, so rancher can always re-apply its view.

| method | request | answer |
| --- | --- | --- |
| `tenant.create` | `{ tenant, id }` | `{ state: "active" }`; a second create with the same id is `ok` |
| `tenant.suspend`, `.resume` | `{ tenant, writes: bool, reads: bool }` | `{ state }` |
| `tenant.delete` | `{ tenant }` | `{ state: "deleted" }`; the file is removed, a deletion event is published |
| `user.delete` | `{ tenant, user }` | `{ seq }` of the one write that deletes everything written for that user |
| `queries.put` | `{ tenant, route, config_seq, queries: [{ name, sql, tables }] }` | `{ prepared: [name] }`; the SQL is fork's canonical output, checked by rancher |
| `bucket.configure` | `{ tenant, bucket, kind, retention_ms? }` | `{ bucket }` |
| `bucket.drop` | `{ tenant, bucket }` | `{ seq }` |
| `table.put` | `{ tenant?, definition }` (the tailer JSON; no tenant means every tenant with the source buckets) | `{ table, version, state: "building" or "live", applied }` |
| `table.drop` | `{ tenant?, table, version }` | `{ state: "dropped" }` |
| `table.rebuild` | `{ tenant, table, version }` | `{ state: "building" }` |
| `status` | `{ tenant? }` | `{ tenants: [{ tenant, seq, buckets, documents, bytes, tables: [{ table, applied, lag }], wal_position }] }` |

| `events.put` | `{ tenant?, name, bucket, ops: ["merge", "delete"], start: "now" or "beginning" }` | `{ name, cursor }`; the same definition again is `ok`; a changed `bucket` or `ops` under the same name is `conflict` |
| `events.drop` | `{ tenant?, name }` | `{ state: "dropped" }`; leased events finish, nothing new is handed out |
| `events.dead.list` | `{ tenant, name?, limit, after? }` | `{ dead: [{ name, seq, bucket, id, op, attempts: [{ holder, at, reason }] }], next }` |
| `events.dead.redrive`, `.discard` | `{ tenant, name, seqs? }` (no seqs: all) | `{ count }` |

`events.put` with `start: "beginning"` replays what `_changes` still holds; older history is not replayed. On promotion, rancher moves an event route from beta's shadow to production without a new cursor, so it neither misses nor repeats changes.

`table.put` with a definition that differs from an existing version of the same number is `conflict` with `pond.table_version_changed`: a new shape is a new version.

## Central types

```
namespace homestead.pond;

table Key { bucket: string; id: long; }

enum OpKind : ubyte { merge, delete }
table Op { op: OpKind; bucket: string; id: long; if_seq: long = -1; doc: [ubyte]; }   // -1: no check

table Effect { to: string; scope: string; body: [ubyte]; }
table Hold { tables: [string]; timeout_ms: uint32; }

table Write {
  tenant: string; user: long;
  ops: [Op]; names: [NameBinding]; effects: [Effect]; hold: Hold;
}
table WriteAnswer { seq: long; hold: HoldAnswer; }

table Query {
  tenant: string; grant: [ubyte]; sql: string; named: NamedRef;
  params: [Value]; reserved: [Value]; min_seq: long;
}

table Change { bucket: string; id: long; op: OpKind; }
table Progress { table: string; applied: long; }
table Reset { oldest: long; }
union FeedBody { Change, Progress, Reset }
table FeedEvent { seq: long; body: FeedBody; }

enum EventOutcome : ubyte { done, failed, skipped }
table Event {
  lease: string; tenant: string; route: string;
  seq: long; bucket: string; id: long; op: OpKind; user: long; at: long; attempt: uint16;
}
table EventAck { lease: string; outcome: EventOutcome; reason: string; }
```

## Problem codes pond raises

| code | when |
| --- | --- |
| `pond.if_seq_stale` | A write lost compare-and-set. Counted per route by barn too. |
| `pond.key_dead` | A write to a deleted key. |
| `pond.id_not_vended` | A write to an id pond never leased. Always a bug upstream. |
| `pond.tenant_suspended` | A refused call to a suspended tenant. |
| `pond.hold_timeout`, `pond.wait_timeout` | A hold or wait ran out. |
| `pond.query_timeout` | A query hit its `max-ms`. |
| `tailer.type_mismatch`, `tailer.behind` | A document doesn't fit a column; a tailer can't keep up. |
| `pond.outbox_dead`, `pond.outbox_stuck` | An element was dead-lettered; an element has waited past its deadline. |
| `pond.replica_lag`, `pond.replica_epoch_stale` | The chain is slow; a stale head pushed. |
| `pond.event_dead`, `pond.event_behind` | An event ran out of attempts; an event route's cursor is falling behind its bucket. |
| `pond.deletions_reset` | A deletion subscriber fell behind the kept log. |
| `fork.*` | Passed through from fork's refusal catalog. |

## The invariants

- **Every changing call carries an idempotency key, and a repeat answers the original.**
- **A write answered `ok` has been fsynced, and replicated when it carries effects.**
- **The tenant comes from barn or rancher.** The service API never takes one from its caller.
- **Feeds name keys, never contents,** and a gap is a `reset`.
- **Pond is the event bus.** Event routes are tailers whose sink is a box; every matching change is leased to one barn at a time until done, skipped, or dead.
- **Boxes never write `_` buckets.** Only barn's own system writes reach `_credentials`.
- **Only rancher calls `pond.control.*`; only pond calls `pond.replica.*`.**
- **Queries carry their grant.** Pond stores none, and runs only fork's output.
