# Ranger packages
How ranger is cut, and which parts a person reads line by line
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Ranger is storage and search for lines, and most of it is ordinary: segments, an index, a scan with bounds, an archive. Two things are not ordinary. **Privacy**: ranger is the one place every service's lines meet, so if a body or an address slips through a writer, ranger's ingest is the last fence before it is kept for 400 days. **Integrity**: the chain is the only reason a log line can be believed after an incident, so the code that verifies it can't be wrong.

So the R0 packages are exactly those two fences, cut small: `ranger_record` (validate the envelope, refuse body fields, redact addresses) and `ranger_chain` (verify chains and record gaps, with the chain function itself owned by `common_log` so writers and ranger share one definition). Scope checks on reads are the third security question, and they live in `ranger_scope`, also R0, as a pure predicate over a line. Everything else (where bytes sit, how fast a search is) is R1 or R2, because its failure is a slow or incomplete answer, which ranger already says out loud.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `ranger_proto` | `homestead::ranger::proto` | `ranger.fbs` and generated code | `common_wire` | R3 |
| `ranger_record` | `homestead::ranger::record` | Validate a line's envelope; refuse known body fields; redact addresses in free text; check service and host against the caller | `common_log`, `common_core` | R0 |
| `ranger_chain` | `homestead::ranger::chain` | Per-stream chain state: verify each line against the previous, record gaps, answer `verify` over segments | `common_log` | R0 |
| `ranger_scope` | `homestead::ranger::scope` | `visible(scope, line) -> bool`: host, tenants, untenanted audit lines of the operator | `ranger_proto` | R0 |
| `ranger_segment` | `homestead::ranger::segment` | Hourly segment files per stream: append, fsync, close, compress with zstd, rewrite for a purge | `common_core`, zstd | R1 |
| `ranger_index` | `homestead::ranger::index` | Per-segment index by time, tenant, request, and route; dedupe high-water marks per stream | `common_sqlite`, `ranger_segment` | R1 |
| `ranger_search` | `homestead::ranger::search` | Search, trace, and count: index lookups, then a bounded scan with RE2; bytes, time, and limit stops | `ranger_index`, `ranger_segment`, `ranger_scope`, RE2 | R1 |
| `ranger_archive` | `homestead::ranger::archive` | Seal a closed day with the backup key, write its manifest (first and last seq, chains, counts, hash), store it, age it out | `common_blob`, `common_crypto`, `ranger_segment` | R1 |
| `ranger_audit` | `homestead::ranger::audit` | Write an `audit` line for every read, purge, and gap, through ranger's own `common_log` | `common_log` | R1 |
| `ranger_ingest` | `homestead::ranger::ingest` | The ingest pipeline: record check, dedupe, chain, append, ack durable seqs | `ranger_record`, `ranger_chain`, `ranger_segment`, `ranger_index`, `ranger_audit` | R1 |
| `ranger_tail` | `homestead::ranger::tail` | Live fan-out of ingested lines to tail streams, with filters, rate caps, and dropped counts | `ranger_scope` | R2 |
| `ranger_server` | `homestead::ranger::server` | Binds `ranger.*` methods; ingest from any service, reads and control from rancher only | `ranger_proto`, core | R2 |
| `ranger_client` | `homestead::ranger::client` | The shipper every service runs: read `common_log`'s spool, push batches, drop what is durable; and rancher's read interface | `ranger_proto`, `common_wire`, `common_log` | R1 |
| `ranger_fake` | `homestead::ranger::fake` | In-memory ranger: keeps lines, answers search and trace naively, checks chains | `ranger_client` | R2 |
| `ranger_system` | `homestead::ranger::system` | The composition root | everything | R2 |
| `ranger_main` | | The binary | `ranger_system` | R3 |

`ranger_client` is R1, not R2, because every service in homestead runs it: a bug there loses or duplicates every service's lines.

```
                              ranger_main
                                  |
                             ranger_system
              +-------------+-----+------+--------------+-----------+
              |             |            |              |           |
        ranger_server  ranger_ingest  ranger_search  ranger_tail  ranger_archive
              |          /   |   \      |      \          |           |
        ranger_proto  record chain  audit  ranger_index  ranger_scope  common_blob
              |        (R0)  (R0)    |        |          (R0)
         common_wire     \    |      |   ranger_segment
                          common_log --+      |
                                             zstd

   every service: common_log (spool, envelope, chain) --> ranger_client (shipper) --> wire
```

## Quality seeds

### ranger_record (R0)
- **Promise.** A line is kept only if its envelope is complete, its service and host are the caller's, it names no refused field, and every address in free text has been replaced by its hash. A refused line is counted by stream and field, never kept.
- **Tests.** A table of envelopes, valid and each way invalid; every refused field name, nested and at the top; an address corpus (internationalized, quoted local parts, addresses inside URLs) that must all redact; a fuzzed line parser.
- **Won't.** Store, index, or chain anything; decide which fields a service may log.

### ranger_chain (R0)
- **Promise.** A stream's chain is verified with `common_log::chain`, the same function writers use; a break is reported at the first line that doesn't chain; a seq jump is a gap, recorded and distinguished from a break.
- **Tests.** Gold chains; a line changed, removed, inserted, or reordered at every position; gaps of every size; restart in the middle of a stream; verify across segment boundaries and an archived day.
- **Won't.** Compute chains for writers, repair a break, or drop a line.

### ranger_scope (R0)
- **Promise.** `visible` is true only for lines a scope may see; a filter on a read can only narrow it.
- **Tests.** Every scope against tenanted, untenanted, and audit lines; operator mismatches; an exhaustive small-world property test.
- **Won't.** Know where a scope came from.

### ranger_segment, ranger_index, ranger_search, ranger_archive, ranger_audit, ranger_ingest, ranger_client (R1)
| package | promise | tests | won't |
| --- | --- | --- | --- |
| `ranger_segment` | An acked line is on disk; a closed segment never changes except by a purge rewrite, which is atomic | Crash at every append; purge with a crash mid-rewrite | Parse fields |
| `ranger_index` | An indexed lookup finds every line with that value; dedupe never drops a new line | Index vs scan agreement on random data | Search free text |
| `ranger_search` | Every answer is within bounds and says why it stopped; regex shapes outside the limit are refused | Bounds at each limit; RE2 shape refusals; cursor paging stability | Apply scope itself (asks `ranger_scope`) |
| `ranger_archive` | A sealed day plus its manifest proves no segment is missing or changed | Tamper each byte of an archive; restore and verify | Search archives in place |
| `ranger_audit` | Every read, purge, and gap writes one audit line before it answers | One per call, under failures | Read anything |
| `ranger_ingest` | Ack only what is durable; a line is kept once | Duplicate and reordered batches; kill between append and ack | Answer reads |
| `ranger_client` | A spooled line is shipped until acked durable, and deleted only then; nothing on a writer's path waits | Ranger down for a day; spool overflow counts; restarts mid-batch | Block the caller |

## How it's wired

`ranger_system` builds: the config, the segment directory, `ranger_index`, `ranger_chain` (loading each stream's last chain and seq from the newest segments), `ranger_audit`, `ranger_ingest`, `ranger_search`, `ranger_tail`, `ranger_archive` on a `common_blob`, a `rancher_client` problem sink, and `ranger_server` on the listener.

| thread | runs |
| --- | --- |
| network (Asio) | The listener; tail fan-out |
| one ingest thread | Record check, dedupe, chain, append, index; one writer per segment |
| search pool (2 by default) | Search, trace, count, verify; each read bounded by its own time and bytes |
| archive thread | Compress closed hours, seal closed days, age out |

**Start:** load chain state, then listen, so every ingested line chains from what is on disk. **Shutdown:** GOAWAY, finish acked appends, fsync, close the open hour's segments without compressing (they compress on next start). Writers keep spooling while ranger is gone, so a restart loses nothing.

Ranger's own lines (audit) go through its own `common_log` into its own spool and back in through ingest, like any service, so they are chained the same way.

## Review queue

| tier | packages | read |
| --- | --- | --- |
| R0 | `ranger_record`, `ranger_chain`, `ranger_scope`; and `common_log`'s chain and redaction files | Every line, after an adversarial agent review |
| R1 | `ranger_ingest`, `ranger_client`, `ranger_segment`, `ranger_index`, `ranger_search`, `ranger_archive`, `ranger_audit` | QUALITY.md, headers, and tests |
| R2 | `ranger_tail`, `ranger_server`, `ranger_fake`, `ranger_system` | QUALITY.md, skimmed |
| R3 | `ranger_proto`, `ranger_main` | Agent review |

Build order:

1. `ranger_proto`, `ranger_record`, `ranger_chain`, `ranger_segment` (only `common_*` beneath them).
2. `ranger_scope`, `ranger_index`, `ranger_audit`, `ranger_client` then `ranger_fake` (every other service can start shipping to the fake here).
3. `ranger_ingest`, `ranger_search`, `ranger_tail`, `ranger_archive`.
4. `ranger_server`, `ranger_system`, `ranger_main`, then ranger in `sim_world`.

## The invariants

- **Two fences are R0, and small:** what a line may contain, and whether its chain holds. A third, scope, is a pure predicate.
- **One chain function**, in `common_log`, used by every writer and by ranger.
- **One writer per segment.**
- **`ranger_client` runs in every service, and never blocks one.**
- **Ranger's own reads are audited through the same path as everyone's lines.**
