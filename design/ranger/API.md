# Ranger API
How lines get in, how operators and agents read them, and the contract every writer keeps
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Ranger has two doors. **Every service pushes lines in**, through `ranger.ingest`, from a local spool, so a request never waits on ranger and nothing breaks without it. **Only rancher reads them out**, on behalf of operators and agents, so there is one surface with one set of scopes, and every read is audited. Ranger never calls anyone but rancher (problems) and object storage (archives).

Half of ranger's contract lives in the writers: the envelope, the per-stream seq, the hash chain, and the spool are in `common_log`, which every service links. This page documents both halves, because a line is only as trustworthy as the writer that chained it.

Ranger listens on 7406 (see [network](../common/NETWORK.md#ports)).

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `ranger.ingest` | push | every service | yes | acks with the highest seq made durable, per stream |
| `ranger.search` | call | rancher | no (writes an audit line) | lines, newest first, and a cursor |
| `ranger.tail` | stream | rancher | no (audited) | lines as they arrive |
| `ranger.trace` | call | rancher | no (audited) | every line for one request id, in time order |
| `ranger.count` | call | rancher | no (audited) | counts by a field over a window |
| `ranger.verify` | call | rancher | no (audited) | the first chain break, or none |
| `ranger.control.tenant.purge` | call | rancher | yes | lines and segments removed |
| `ranger.control.status` | call | rancher | no | ingest rates, spool gaps reported, disk, oldest segment, archive lag |

Every read carries a **scope**, set by rancher from the operator or agent behind it, never by the reader:

```json
{ "scope": { "operator": "4294960010", "host": false, "tenants": ["bobs_plumbing"] } }
```

A host scope sees everything. A tenant-focused scope sees lines with those tenants, and of untenanted lines only the `audit` lines of its own operator. Ranger applies the scope inside the read, before bounds, so a scope can't be widened by a filter.

## Ingest

```
service -> ranger   OPEN ranger.ingest { window: 8 }
service -> ranger   DATA { "batch": 551, "lines": [ ... up to 4096 lines or 1 MiB ... ] }
ranger -> service   DATA { "batch": 551, "durable": { "barn/barn-7/routing": 88213004, "barn/barn-7/egress": 1200 } }
ranger -> service   CREDIT 1
```

| field | meaning |
| --- | --- |
| batch | The shipper's own batch counter, so acks can be matched. |
| lines | Records in the envelope below, in seq order per stream. |
| durable | Per stream (`service/host/log`), the highest seq ranger has fsynced. The shipper drops spooled lines up to it. |

- **At least once, kept once.** A line is identified by `(service, host, log, seq)`; a resent line is dropped on ingest. Ranger keeps the highest seq per stream, so dedupe is one comparison per line.
- **The caller is checked.** The `service` in each line must equal the caller's service from the handshake (or certificate), and `host` the caller's host, so one service can't write lines as another.
- **Refused lines are counted, not silent.** A line that breaks the envelope, carries a known body field, or fails redaction is dropped, its seq is acknowledged so the shipper doesn't resend it forever, and the stream gets a `ranger.refused` count reported as a problem with the field's name.
- **Gaps are recorded.** A seq that jumps (the shipper's spool dropped lines while ranger was down) is kept, and ranger writes an `audit` line naming the stream and the gap. `ranger.verify` reports it as a gap, not a break.

## Reading

### search

```json
{
	"scope": { "operator": "4294960010", "host": true },
	"logs": ["routing"],
	"tenant": "bobs_plumbing",
	"route": "PUT bobsplumbing.com/me/address",
	"fields": { "outcome": 409 },
	"pattern": "conflict",
	"since": "2026-11-13T16:00:00Z",
	"limit": 100
}
```

```json
{
	"lines": [ { "at": "2026-11-13T17:02:11.123Z", "service": "barn", "host": "barn-7", "log": "routing", "seq": 88213004, "...": "..." } ],
	"cursor": "c2VnLTE3OjQ0MTI=",
	"stopped": null,
	"scanned": { "bytes": 41943040, "ms": 212 }
}
```

| field | meaning |
| --- | --- |
| logs, tenant, request, route | Indexed. Narrowing by these is a lookup. |
| fields | Exact matches on fields, checked during the scan. |
| pattern, regex | A substring, or an RE2 regex of at most 200 characters with no backreferences, over the raw line. |
| since, until | Defaults to the last hour. At most 7 days, the local retention. |
| before | A cursor from a previous answer, for the next page. |
| stopped | `null`, or `bytes`, `time`, or `limit`: why the search stopped early. A stop is never an error; it is said. |

### tail

`ranger.tail` takes the same filters (without `since`, `until`, and `before`, but with an optional `from` cursor) and pushes lines as they arrive. Rancher's credit paces it; ranger caps a tail at 500 lines a second and sends a `dropped` count in place of lines beyond that.

### trace, count, verify

```json
{ "scope": { "...": "..." }, "request": "9f3c..." }
```

`ranger.trace` answers every line with that request id across every service, in `at` order, from the request index. It is bounded by the same 10 seconds and 2 GiB.

```json
{ "scope": { "...": "..." }, "logs": ["routing"], "tenant": "bobs_plumbing", "group-by": "outcome", "since": "2026-11-13T16:00:00Z" }
```

```json
{ "counts": { "200": 18210, "409": 31, "503": 2 }, "stopped": null }
```

`ranger.verify {log, host?, days?}` recomputes each stream's chain over local segments (and archived days, when `days` reaches past 7 and they are restored) and answers the first break: the stream, the seq, and what it expected. Gaps recorded on ingest are listed separately, since a gap is a known loss and a break is a change.

## Tenant deletion

When a tenant is deleted, rancher calls `ranger.control.tenant.purge {tenant}`. Ranger drops that tenant's lines from local segments by rewriting the affected segments, and records an `audit` line saying how many lines and which segments. Archived days age out on their 400-day retention; they are sealed and not rewritten.

This is a rancher control call, not a `pond.deletions.subscribe` follower, deliberately: ranger holds no user data to delete under the lean of [Q12.2](../OPEN.md#q122-how-do-deletions-reach-permit-sun-and-barker), only ids and hashes, so a user's deletion needs nothing from ranger, and a tenant's deletion is already rancher's act. Ranger then depends on nothing but rancher, and a purge is idempotent by tenant.

## The writer contract

Every service links `common_log`. It is what makes a line trustworthy before ranger ever sees it.

```cpp
namespace homestead::common::log {
class Log {
 public:
  // Appends to the local spool. Never blocks on the network; never throws.
  void write(std::string_view log, Fields fields, Context ctx = {});
};
struct Context { std::optional<std::string> tenant; std::optional<std::string> request; };
}
```

| piece | rule |
| --- | --- |
| Envelope | `at` (UTC ms, from the injected clock), `service`, `host`, `log`, `seq`, `tenant?`, `request?`, `fields`, `chain`. |
| seq | Per `(log)` stream on this host, starting at 1, never reused, stored in the spool so a restart continues it. |
| chain | `SHA-256(previous chain || canonical JSON of the line without chain)`, hex. The first line of a stream chains from 32 zero bytes. The function is `common_log::chain`, and ranger verifies with the same one. |
| Redaction | Fields named `body`, `doc`, `params`, `token`, `password`, `cookie`, and `authorization` are refused at write time; an address in free text becomes `[address <hash>]`. Ranger repeats both checks on ingest. |
| Spool | An append-only file per stream under the service's data directory; bounded by 1 GiB or 24 hours, then the oldest lines are dropped and counted. |
| Shipper | `ranger_client` reads the spool, sends batches on one `ranger.ingest` push stream, and deletes what ranger acknowledged durable. |

```
namespace homestead.ranger;

table Line {
  at: int64; service: string; host: string; log: string; seq: uint64;
  tenant: string; request: string;
  fields: [ubyte];          // canonical JSON, as chained
  chain: [ubyte];           // 32 bytes
}
table IngestBatch { batch: uint64; lines: [Line]; }
table StreamAck { stream: string; durable: uint64; }
table IngestAck { batch: uint64; durable: [StreamAck]; }

table Scope { operator: int64; host: bool; tenants: [string]; }
table SearchRequest {
  scope: Scope; logs: [string]; tenant: string; request: string; route: string;
  fields: [ubyte]; pattern: string; regex: string;
  since: int64; until: int64; limit: uint32; before: string;
}
enum Stop : ubyte { none, bytes, time, limit }
table SearchAnswer { lines: [Line]; cursor: string; stopped: Stop; scanned_bytes: uint64; scanned_ms: uint32; }
```

## Codes and problems

| code | when |
| --- | --- |
| `invalid` | A malformed filter, a regex over the limits, a window past local retention. |
| `refused` | A read from anyone but rancher; a line whose service or host isn't the caller's. |
| `overloaded` | Disk is past its high-water mark; ingest pushes back with credit first, and refuses only after that. |

| problem | raised when |
| --- | --- |
| `ranger.refused` | Lines were dropped on ingest; names the stream and field. |
| `ranger.gap` | A stream's seq jumped: a spool overflowed while ranger was away. |
| `ranger.chain_break` | `verify`, or ingest itself, found a line that doesn't chain. |
| `ranger.archive_failed` | A closed day couldn't be sealed and stored. |
| `ranger.disk` | Local retention had to shrink below 7 days. |

## The invariants

- **Everyone writes; only rancher reads.** And every read is an `audit` line.
- **A line is identified by service, host, log, and seq**, and kept once.
- **A service can only write its own lines.** Checked against the caller's identity.
- **The chain is computed by the writer** and verified by ranger with the same function.
- **Nothing in a line is tenant data**, checked at write and again at ingest.
- **The scope comes from rancher, and is applied before any filter.**
- **Every read is bounded, and says when it stopped early.**
