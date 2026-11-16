# Ranger
Every service's log lines, in one place, searchable while they happen
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's logs were files on the host: access, egress, mail, agents, db, audit, and one per box, each line a JSON object with a common envelope, hash-chained, rotated daily, archived sealed to the backup bucket, and searched with a grep-like `log_search` over the last few days. That worked because there was one host. Homestead has many barns and a handful of single-machine services, and barn writes one routing record per request whose destination was left open. Grepping six machines' files is not a tool an agent can use.

Ranger is the log service. **Every service streams its log lines to ranger; ranger keeps them, chains them, and answers tail, search, and grep over all of them at once, in real time.** Operator tooling and agents read the same API. A request id finds a request's lines in barn, pond, permit, sun, and barker in one call.

Ranger keeps raw records. It doesn't group, deduplicate, alert, or decide what is wrong. That is rancher's problem log: problems are grouped and written for the agent that will fix them; logs are every line, for whoever needs to see exactly what happened.

## The shape

```
   barn  barn  barn   pond   rancher   permit   sun   barker
     \     |    /       |       |         |      |      |
      \    |   /        |       |         |      |      |
       +---+--+---------+-------+---------+------+------+
                        |  internal protocol, batched, with credit
                        v
               +----------------------+
               | ranger (one machine) |
               |  segments per hour   |
               |  index: time, tenant,|
               |   request, route     |
               |  chain per stream    |
               +----------------------+
                 |                  |
       tail / search / grep     archive (sealed)
       (via rancher's API, MCP)  to object storage
```

## A record

```json
{
	"at": "2026-11-13T17:02:11.123Z",
	"service": "barn",
	"host": "barn-7",
	"log": "routing",
	"seq": 88213004,
	"tenant": "bobs_plumbing",
	"request": "9f3c...",
	"fields": {
		"route": "PUT bobsplumbing.com/me/address",
		"version": "accounts@sha256:77ab02...",
		"picked-by": "split 5% by user",
		"objects": { "address": "addresses/4294967100 @4471 merge" },
		"outcome": 200,
		"ms": { "box": 3.1, "write": 2.2, "hold": 38 }
	},
	"chain": "5b1e0c7a9d..."
}
```

| field | meaning |
| --- | --- |
| at | When the line was written, at the source, in UTC with milliseconds. |
| service, host | Who wrote it. |
| log | Which stream: see below. |
| seq | The line's number in its stream on that host, never reused. With service, host, and log, it identifies the line, so a line sent twice is kept once. |
| tenant | Optional. Present whenever the line is about one tenant, which is what scopes it. |
| request | Optional. The request, timer job, or message id that ties lines across services. |
| fields | The event's own fields, structured. |
| chain | A hash over the previous line's chain and this line, per stream, as in goober. |

**Never tenant data.** A record holds keys, seqs, names, versions, outcomes, and timings, never documents, bodies, query parameters, or tokens, as barn's invariant already says of routing records. A person's email address is a keyed hash and its domain. Services enforce this when they write; ranger enforces it again on ingest by refusing fields it knows are bodies, and by turning anything that looks like an address in free text into `[address <hash>]`.

## The streams

| log | written by | one line per |
| --- | --- | --- |
| routing | barn | Request or timer event: which route, which version and why, what was touched, where the time went. |
| egress | barn | Outbound fetch from a box: service, method, status, latency, bytes. |
| mail | barker | Send, attempt, bounce, complaint: recipient hash, template, provider id, outcome. |
| agents | barn | Tool call over a site's `/mcp`: client, tool, subject, outcome. |
| writes | pond | Applied write: tenant, buckets, keys, seq, size. Never documents. |
| tailers | pond | Tailer progress, rebuilds, and refusals. |
| timers | sun | Schedule, fire, ack, expiry, dead letter. |
| control | rancher | Configuration change: who, what object, the config seq. Diffs stay in rancher; the line points at them. |
| identity | permit | Sign-in, failed sign-in, grant, revoke (addresses hashed). |
| audit | every service | Security events an auditor asks for: operator actions, auth failures, starts and stops, and every read of ranger itself. |

A new stream is a new name; ranger needs no schema for it, only the envelope.

## Ingest

Services send batches over the internal protocol, with credit, like barns subscribing to sun. Each service writes locally first, to a small spool, and ships from there:

- **Never in the request path.** A request never waits on ranger. Writing a line is appending to the local spool.
- **At least once, kept once.** A batch is acknowledged after ranger makes it durable. A batch resent after a lost ack is deduplicated by `(service, host, log, seq)`.
- **Ranger down.** The spool grows, up to a bound (default 1 GiB or 24 hours), then the oldest lines are dropped, counted, and reported as a problem when ranger returns. A line lost that way leaves a gap in the chain, which says so.
- **Order.** Lines from one stream on one host arrive in seq order. Across hosts, ranger orders by `at`, and tail shows lines as they arrive.

## Storage and retention

Ranger keeps each stream in hourly segments, compressed once closed, with a small index per segment by time, tenant, request, and route. Searching by those is a lookup; searching by anything else is a bounded scan.

| where | how long | what |
| --- | --- | --- |
| On ranger's disk | 7 days | Every segment, searchable. |
| Object storage | 400 days | Each closed day, sealed with the backup key (a secret opened with [cellar](../cellar/IDEA.md)), with a manifest of its first and last seq, its chains, its line count, and its hash. Restorable for search, not searched in place. |

A day's archive and its manifest let anyone check that no segment is missing and no line was changed. `log_verify` walks the chains and answers the first break.

Deletion (Q2.4): a user's bulk delete flows through pond, but ranger holds no user data to delete, only hashes and ids. A tenant's deletion drops its lines from the local segments; archives age out.

## Reading

Reads go through rancher's operator API, which proxies ranger's, so operator tooling and agents (over MCP) use one surface with one set of scopes.

| tool | does |
| --- | --- |
| `log_tail {logs?, tenant?, filter?}` | Follows matching lines as they arrive, from now, or from a cursor. The live view. |
| `log_search {logs?, tenant?, request?, route?, fields?, pattern?, regex?, since?, until?, limit?, before?}` | Newest first, paged by cursor. Indexed fields first, then substring or bounded regex over the raw line. Defaults to the last hour. |
| `log_trace {request}` | Every line for one request id, across every service, in time order. The usual first move when a problem names a request. |
| `log_count {logs, tenant?, group-by, since?}` | Counts by a field over a window, for "how often" without reading lines. |
| `log_verify {log, host?, days?}` | Recomputes the chains and answers the first break. |

**Bounds.** A regex is at most 200 characters with no backreferences or nested repeats; a search reads at most 2 GiB and runs at most 10 seconds, and says when it stopped early; a tail is capped in lines per second and drops (and counts) beyond it. Every read is itself an `audit` line.

**Scopes.** A host operator sees everything. An operator focused on tenants sees only lines with those tenants, and of the untenanted lines only their own audit actions. Tenants don't read ranger (Q8.1: not yet).

## Logs and problems

| | ranger (logs) | rancher (problems) |
| --- | --- | --- |
| Unit | A line | A problem: grouped by fingerprint, counted |
| Written | For every event | When something goes wrong |
| Reader | Whoever needs exactly what happened | The agent that will write the fix |
| Lifetime | 7 days hot, 400 archived | Until fixed or discarded |
| Link | A problem's examples carry request ids | `log_trace` on that id shows the whole story |

A problem says "312 documents in addresses hold a string where users@1 wants int64." Ranger says what happened on each of those requests.

## What ranger is not

- **Not metrics.** Failure counts per route go from barn and sun to rancher every few seconds, for health and guardrails. Ranger's `log_count` is for investigation, not for dashboards that must never miss.
- **Not the problem log.** It never groups, deduplicates, or alerts.
- **Not a store of tenant data.** Only ids, keys, names, hashes, and timings.
- **Not in the request path.** Nothing waits on it, and nothing breaks without it.
- **Not a place for diffs.** Rancher keeps configuration diffs; ranger points at them.

## Open questions

1. **Box logs.** Goober kept a log per box with the box's own text. That text is tenant-authored and could carry personal data, which breaks "never tenant data." Leave it out, keep it in a separate, shorter-lived stream with its own scope, or turn it into structured fields only?
2. **Sampling routing records.** One line per request is the design. At high traffic, keeping every failure plus a sample of successes may be enough; the cost of one machine's disk decides.
3. **Indexing.** Four indexed fields and bounded scans is the start. A full-text index would make grep fast and ranger much larger.
4. **Searching archives.** Restoring a sealed day to search it is slow but simple. Searching archives in place needs a different format.
5. **A cluster of rangers.** Partitioning by tenant, like pond, once one machine isn't enough; untenanted lines go to one partition.
6. **Chain across hosts.** Chains are per stream per host. A single chain over all of ranger's ingest would prove more and couple every writer.

## The invariants

- **Nothing waits on ranger.** Lines go to a local spool first.
- **A line is kept once.** Identified by service, host, log, and seq.
- **Lines carry no tenant data.** Keys, names, versions, outcomes, timings, and hashes only.
- **Every stream is chained.** A changed, removed, or inserted line breaks the chain where it happened.
- **Every read of ranger is audited.**
- **Reads are bounded.** Time, bytes, and regex shape all have limits, and an early stop says so.
- **Logs are raw; problems are grouped.** Ranger never decides what is wrong.
