# Pond packages
How pond is cut so the parts that can lose data are small enough to read in full
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Pond is the one service whose bugs lose people's data, so it is cut around one question: **which code, if wrong, loses an acknowledged write, leaks a row, or reuses an id?** That code goes in R0 packages, and everything else is pushed out of them. The WAL knows bytes and positions, not tenants. The write validator knows the rules a write must pass, not how it is stored. The applier folds a record into a tenant file, not how the file's statements are written. The query gate wraps fork and SQLite's locks, not caching or waiting. What's left (tailers, feeds, the outbox, shipping, control) is R1: a bug there gives a wrong or late answer that a rebuild or a redelivery repairs, because tables are a function of documents and every delivery is idempotent.

A person reads six packages line by line: `pond_wal`, `pond_write`, `pond_apply`, `pond_ids`, `pond_query`, and `pond_replica`. Everything else they read through its QUALITY.md and tests.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `pond_proto` | `homestead::pond::proto` | `pond.fbs` and generated code | `common_wire` | R3 |
| `pond_model` | `homestead::pond::model` | Plain types: Key, Op, Write, Effect, Record, TableDefinition, Change. No I/O. | `common_core` | R1 |
| `pond_wal` | `homestead::pond::wal` | Segments, append, group commit, fsync, recovery reads, truncation. Bytes and positions only. | `common_core` | R0 |
| `pond_record` | `homestead::pond::record` | Encodes a validated write as a WAL record and back; versioned, checksummed | `pond_model` | R0 |
| `pond_catalog` | `homestead::pond::catalog` | `_pond.db`: tenants and their state, tailer definitions, named queries, the deletions log | `common_sqlite`, `pond_model` | R1 |
| `pond_ids` | `homestead::pond::ids` | The counter and ranges in `_pond.db`; the set of vended ranges a write checks against; restore floor | `common_sqlite`, `common_core` | R0 |
| `pond_tenantdb` | `homestead::pond::tenantdb` | A tenant file: its system tables, prepared statements, connection pools (one writer, many read-only) | `common_sqlite`, `pond_model` | R1 |
| `pond_write` | `homestead::pond::write` | Validation and sequencing: tenant state, vended ids, dead keys, `if-seq`, idempotency, limits; assigns the seq | `pond_model`, `pond_ids`, `pond_tenantdb` | R0 |
| `pond_apply` | `homestead::pond::apply` | Folds a record into a tenant file in one transaction (documents, names, changes, outbox, cursor); replay by seq | `pond_record`, `pond_tenantdb`, `common_merge` | R0 |
| `pond_tailer` | `homestead::pond::tailer` | Builds and maintains `name@version` tables from `_changes`; joins by id; type checks to problems | `pond_tenantdb`, `pond_model`, `common_problem` | R1 |
| `pond_query` | `homestead::pond::query` | The gate: fork's check, the read-only connection, authorizer, `stmt_readonly`, progress handler, row cap; named query preparation | `fork_api`, `pond_tenantdb` | R0 |
| `pond_wait` | `homestead::pond::wait` | Bounded waits on tenant seq and table cursors, for holds, `min_seq`, and `pond.wait` | `pond_model`, `common_core` | R1 |
| `pond_feed` | `homestead::pond::feed` | Subscriptions over `_changes`: filters by bucket and id, table progress, resets | `pond_tenantdb`, `pond_wait` | R1 |
| `pond_outbox` | `homestead::pond::outbox` | Leases over `_outbox` per consumer, order per scope, acks, dead elements | `pond_tenantdb`, `common_core` | R1 |
| `pond_deletions` | `homestead::pond::deletions` | User delete as one write; the cross-tenant deletions log and its subscription | `pond_write`, `pond_catalog` | R1 |
| `pond_replica` | `homestead::pond::replica` | Chain replication of WAL records: head, middle, tail; epochs; durable positions | `pond_wal`, `common_wire` | R0 |
| `pond_ship` | `homestead::pond::ship` | Nightly snapshots and continuous WAL shipping to object storage; restore | `pond_wal`, `pond_tenantdb`, `common_blob` | R1 |
| `pond_control` | `homestead::pond::control` | The control API's logic: tenants, buckets, tables, named queries, status; idempotent by content | `pond_catalog`, `pond_tailer`, `pond_query`, `pond_deletions` | R1 |
| `pond_engine` | `homestead::pond::engine` | The per-tenant pipeline: write queue, applier thread, tailer scheduling; what `pond_server` calls | `pond_write`, `pond_wal`, `pond_apply`, `pond_tailer`, `pond_feed`, `pond_wait` | R1 |
| `pond_server` | `homestead::pond::server` | Binds `pond.*` methods to the engine and control, checks caller services | `pond_proto`, `pond_engine`, `pond_control`, `pond_outbox`, `pond_replica` | R2 |
| `pond_client` | `homestead::pond::client` | The `Pond` interface, its wire implementation, and the contract test suite | `pond_proto`, `common_wire` | R2 |
| `pond_fake` | `homestead::pond::fake` | An in-memory `Pond`: seqs, `if-seq`, idempotency, names, feeds, outbox; passes the contract tests | `pond_client`, `common_merge` | R2 |
| `pond_system` | `homestead::pond::system` | The composition root | everything above | R2 |
| `pond_main` | | The binary | `pond_system` | R3 |

`pond_record` was split from `pond_wal` so the WAL stays ignorant of what a write is; `pond_wait` was split from the feed so holds, `min_seq`, and `pond.wait` share one bounded waiter; `pond_engine` holds the threading so no R0 package owns a thread.

## Dependencies

```
                         pond_main
                             |
                        pond_system
           +-----------------+------------------+
      pond_server        pond_ship         (rancher_client, ranger via common_log)
     /    |    \   \
pond_engine pond_control pond_outbox pond_replica
  |   |  \     |     \                   |
  |   |   \  pond_deletions  pond_query  pond_wal
  |   |    \     |            |  \
  | pond_feed  pond_write   fork_api  pond_tenantdb
  |   |          |   \                  |
  | pond_wait  pond_ids pond_tenantdb  common_sqlite
  |
pond_apply --- pond_record --- pond_model --- common_core
   |
common_merge

pond_client --- pond_proto --- common_wire          pond_fake --- pond_client
```

## QUALITY.md seeds

### pond_wal (R0)
- **Promise:** a record whose append future resolved is on disk and is returned by every replay, in order; nothing unacknowledged is ever returned as acknowledged.
- **Tests:** a fault-injecting filesystem (`common_test`) crashing at every write and fsync; torn tails cut back; group commit with 10k concurrent appends; truncate never removes a position someone hasn't released.
- **Won't:** know tenants or documents, compress, or own threads beyond its fsync loop.

### pond_record (R0)
- **Promise:** decode(encode(w)) == w for every valid write; a corrupt record is detected by its checksum and never applied.
- **Tests:** round trips, a corpus of records from every past format version, fuzzed decode.
- **Won't:** validate business rules.

### pond_ids (R0)
- **Promise:** no id is ever vended twice, across crashes and restores; ids start at 2^32 + 1; a write's ids can be checked as vended in O(log ranges).
- **Tests:** crash between counter move and answer; repeated idempotency keys; restore floor past the highest restored id; property test over random lease/crash sequences.
- **Won't:** know tenants; bind names (that is a write).

### pond_write (R0)
- **Promise:** a write that passes is valid against the tenant's state at the seq it gets; one that fails changes nothing; idempotent repeats answer the original seq.
- **Tests:** a table of every refusal rule; concurrent conflicting writes to one key (exactly one wins); idempotency across restart.
- **Won't:** touch the WAL or the tenant file directly; it decides, the engine does.

### pond_apply (R0)
- **Promise:** applying a record is one SQLite transaction that moves the tenant cursor; replaying any record twice is a no-op; documents equal the ordered fold of every acknowledged merge.
- **Tests:** model-based test against an in-memory fold; crash mid-transaction; user delete removes exactly that user's rows and names.
- **Won't:** tail, publish, or decide anything.

### pond_query (R0)
- **Promise:** SQLite only ever prepares fork's canonical SQL, on a read-only connection, under an authorizer that allows exactly the tables and columns fork reported, and the grant's row and time caps.
- **Tests:** fork's injection corpus run end to end against a real tenant file; a deliberately broken fork (a test double that passes everything) must still be stopped by the authorizer; timeouts.
- **Won't:** cache answers, store grants, or wait (that is `pond_wait`).

### pond_replica (R0)
- **Promise:** a position acknowledged by the tail is durable on every chain member; a stale epoch is refused.
- **Tests:** in-memory transport with severed links, reordered acks, a head replaced mid-batch.
- **Won't:** choose the chain (rancher's placement does).

### R1 seeds

| package | promise | tests | won't |
| --- | --- | --- | --- |
| `pond_model` | Plain, comparable types with name and size validation | Table tests | Do I/O |
| `pond_catalog` | `_pond.db` reflects every control call exactly once | Repeat every control call; reopen | Hold tenant data |
| `pond_tenantdb` | One writer, pooled readers, the system schema created once and versioned | Schema upgrade from every prior version | Decide what to write |
| `pond_tailer` | A table equals a rebuild from documents at the same cursor | Incremental vs full rebuild equality, joins over three buckets, type mismatches become problems | Stop on a bad document |
| `pond_wait` | Every wait ends by its bound and says how far each table got | Manual clock; wakeups on progress | Spin |
| `pond_feed` | Every change once, in seq order, filtered, or a `reset` | Slow subscribers, resubscribe from every seq | Send contents |
| `pond_outbox` | Each element is leased to one consumer at a time, in order per scope, until acked or dead | Lease expiry, competing consumers, out-of-order acks | Interpret bodies |
| `pond_deletions` | Deleting a user removes their documents and names in one write and publishes one event | Cross-check against the WAL's user annotations | Delete other services' copies |
| `pond_ship` | Restore = snapshot + shipped WAL equals the live tenant at that seq | Restore round trip in `common_blob`'s local implementation | Ship unfinished segments |
| `pond_control` | Every call is idempotent by content and refuses a changed definition under the same version | Re-apply rancher's whole view twice | Run without rancher's checks |
| `pond_engine` | Per tenant, writes are applied in seq order and tailers never block writes | Many tenants, one hot tenant, shutdown mid-load | Validate or encode |

## How it's wired

`pond_system` builds, in order: the config; `pond_catalog` and `pond_ids` on `_pond.db`; `pond_wal` (and its replay); an engine per tenant, lazily, which opens the tenant's `pond_tenantdb` and replays from its cursor; `pond_replica` (as head, middle, or tail, from config); `pond_ship`; `pond_control`; `pond_server` on the wire listener; and the clients pond dials (rancher for problems, ranger through `common_log`'s spool).

| threads | run |
| --- | --- |
| 1 network `io_context`, N threads | `common_wire`, `pond_server` decoding, feed fan-out |
| 1 WAL thread | Group commit and fsync |
| 1 applier per active tenant, from a pool | `pond_apply`, then `pond_tailer` and feed publish for that tenant |
| A read pool | `pond_query` runs, so a slow query never blocks a write |
| 1 shipper | Snapshots and WAL shipping |

**Start:** replay the WAL into every tenant past its cursor, rebuild tailers that are behind, then open the listener. Pond answers `unavailable` until replay finishes.
**Shutdown:** GOAWAY, finish in-flight writes, fsync, apply everything, checkpoint each tenant, ship the tail, close. Pond is the service that restarts least; this path is tested in `sim_world` on every change to `pond_system`.

## Review queue

1. **R0**, read line by line: `pond_wal`, `pond_record`, `pond_ids`, `pond_write`, `pond_apply`, `pond_query`, `pond_replica`.
2. **R1**, read contract, QUALITY.md, and tests: `pond_model`, `pond_engine`, `pond_tenantdb`, `pond_tailer`, `pond_wait`, `pond_feed`, `pond_outbox`, `pond_deletions`, `pond_catalog`, `pond_ship`, `pond_control`.
3. **R2/R3**: `pond_server`, `pond_client`, `pond_fake`, `pond_system`, `pond_proto`, `pond_main`.

Build order, with what can go in parallel on each line:

1. `pond_model`, `pond_wal`, `pond_proto` (and fork, independently)
2. `pond_record`, `pond_ids`, `pond_tenantdb`, `pond_catalog`, `pond_client` + `pond_fake` (unblocks barn)
3. `pond_write`, `pond_apply`, `pond_wait`, `pond_query`
4. `pond_tailer`, `pond_feed`, `pond_outbox`, `pond_deletions`, `pond_replica`, `pond_ship`
5. `pond_engine`, `pond_control`
6. `pond_server`, `pond_system`, `pond_main`

`pond_client` and `pond_fake` land early so barn, permit, sun, and barker can build against the contract before pond exists.

## The invariants

- **No R0 package owns a thread or knows the wire.** Threads live in `pond_engine` and the system; the wire in `pond_server` and `pond_replica`'s transport.
- **The WAL knows bytes; the record knows writes; the validator knows rules; the applier knows the file.** Each in its own package.
- **SQLite prepares only fork's output, and only `pond_query` prepares reads.**
- **Tables, feeds, and the outbox are R1** because a rebuild or a redelivery repairs them.
- **`pond_fake` passes the same contract tests as pond.**
