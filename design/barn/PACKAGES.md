# Barn's packages
A pure evaluator cut so the ironclad parts fit on a page
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Barn is the largest service, because it is where every protocol, every box, and every other service's client meet. It is also where a bug is most likely to become a breach: a box is untrusted code, and barn is what stands between it and every tenant's data. So barn is cut along one question: **which code, if wrong, lets a box or a caller touch something the route didn't give it?** That code is R0, and it is kept as small as it can be made:

- **`barn_caller`** decides who the caller is and whether the route's requirement holds.
- **`barn_objects`** turns slots into keys.
- **`barn_iron`** turns a box's staged calls into exactly one compare-and-set write.
- **`barn_abi`** copies bytes across the guest boundary.
- **`barn_wasm`** confines the box: isolation levels, resets, and limits.

Everything else is R1 or lower, because its worst bug is a wrong answer or a slow one, not a reach. Routing (`barn_route`, `barn_policy`) picks which route and version answer, but every route it can pick was checked by rancher, and the R0 packages enforce that route's rules whatever was picked. The cache is R1 because it holds only what pond already decided a caller may see under that grant, and its keys always include the tenant and grant. The evaluator (`barn_eval`) is R1 because it only sequences R0 decisions; it may not make one.

Barn has **no `barn_server` and no `barn_proto`**. Nothing dials barn on the wire, so it has no schema to publish, and no `barn_client` or `barn_fake` exists for anyone to depend on.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `barn_config` | `homestead::barn::config` | Holds the live configuration: takes rancher's snapshot, follows the config log in order, and publishes immutable `View`s (routes, services, domains, releases) by config seq | `rancher_client`, `rancher_model`, `common_core` | R1 |
| `barn_route` | `homestead::barn::route` | Domain to tenant and environment (prod, beta, ops); request to exactly one route by verb, domain, and typed path; `mcp`, `mail`, `timer`, `sse` keys | `rancher_model`, `common_core` | R1 |
| `barn_policy` | `homestead::barn::policy` | Picks a box version by weights and a stable hash of user, ip, or request; names shadows | `rancher_model`, `common_crypto` | R1 |
| `barn_caller` | `homestead::barn::caller` | Verifies tokens with permit's keys, holds the grants cache and revocations, evaluates the route's requirement, binds parameters | `permit_token`, `permit_require`, `permit_client`, `common_core` | R0 |
| `barn_objects` | `homestead::barn::objects` | Resolves each slot to a key: reserved values, bound params only, `new` from `barn_ids`, `name:` through pond; loads documents and seqs | `rancher_model`, `barn_ids`, `pond_client` | R0 |
| `barn_ids` | `homestead::barn::ids` | Leases id ranges from pond and mints from them; leases the next range before the current one runs out | `pond_client`, `common_core` | R1 |
| `barn_iron` | `homestead::barn::iron` | Stages merges, deletes, holds, and effects per slot; composes patches; builds one pond write with `if-seq` per object and the request id as idempotency key | `common_merge`, `rancher_model`, `pond_client` (types only) | R0 |
| `barn_abi` | `homestead::barn::abi` | The host side of `barn-abi@1` and `iron@1`: bounds-checked copies in and out of guest memory, argument decoding, calls into the request context | `barn_iron`, `common_core`, wasmtime (types) | R0 |
| `barn_wasm` | `homestead::barn::wasm` | wasmtime engine and compile cache, instance pools per isolation level, reset to snapshot with verification, fuel, memory, epoch deadlines | `barn_modules`, wasmtime | R0 |
| `barn_modules` | `homestead::barn::modules` | Fetches modules from rancher by hash, verifies the hash, caches them on disk forever | `rancher_client`, `common_crypto` | R2 |
| `barn_cache` | `homestead::barn::cache` | Objects by key, names forever, query answers by tenant, grant, canonical SQL, and params; invalidation from pond's feed; reset drops a tenant | `pond_client`, `common_core` | R1 |
| `barn_eval` | `homestead::barn::eval` | The pipeline: domain, caller, route, version, objects, box, write, hold, then, encode; shadows; outcomes to statuses | every core package above, `pond_client` | R1 |
| `barn_query` | `homestead::barn::query` | Runs `query.named` and `query.sql` for a box: attaches the route's grant and reserved params, goes through the cache, optional early fork check | `pond_client`, `barn_cache`, `fork_api` (optional) | R1 |
| `barn_fetch` | `homestead::barn::fetch` | Egress for `fetch.call`: declared origins only, shadow-safe check, timeouts, `egress` lines | `common_http`, `rancher_model`, `common_log` | R1 |
| `barn_edge` | `homestead::barn::edge` | The HTTP server on 8080: parsing, header stripping and ownership, reserved prefixes, health, statuses | `common_http`, `barn_eval`, `barn_proxy` | R1 |
| `barn_proxy` | `homestead::barn::proxy` | Carries reserved-prefix requests opaquely as `http.serve` to rancher, permit, barker, store | `common_http`, the four `_client`s | R1 |
| `barn_sse` | `homestead::barn::sse` | Streams: feed subscriptions per route group, fan-out by `per-viewer`, resume by `Last-Event-ID`, close on revocation | `pond_client`, `barn_eval` | R1 |
| `barn_mcp` | `homestead::barn::mcp` | MCP over streamable HTTP: `tools/list` from routes and grants, `tools/call` through `barn_eval` | `common_http`, `barn_eval` | R2 |
| `barn_mail` | `homestead::barn::mail` | LMTP on 2424, recipient to route, reply tokens, bounces to barker, thread notes | `barker_reply`, `barker_client`, `barn_eval` | R1 |
| `barn_timer` | `homestead::barn::timer` | `sun.fire.subscribe` with credit, evaluation as timer requests, acks and heartbeats | `sun_client`, `barn_eval` | R1 |
| `barn_stitch` | `homestead::barn::stitch` | Fetches releases from store, serves page plans, runs slots in parallel under deadlines with fallbacks, page caching | `store_plan`, `store_client`, `common_mustache`, `barn_query`, `barn_eval` | R1 |
| `barn_routelog` | `homestead::barn::routelog` | One routing record per request; failure counts per tenant, route, and version, reported to rancher | `common_log`, `common_problem`, `rancher_client` | R2 |
| `barn_system` | `homestead::barn::system` | The composition root | everything | R2 |
| `barn_main` | | The binary: one argument, a config file | `barn_system` | R3 |

```
                                 barn_main
                                     |
                                 barn_system
          +-------------+-----------+-----------+-------------+------------+
       barn_edge    barn_sse    barn_mcp    barn_mail    barn_timer   barn_stitch
          |  \          |           |           |            |            |
     barn_proxy +-------+-----------+-----------+------------+------------+
                                     |
                                 barn_eval  (R1: sequences, decides nothing)
     +-----------+-----------+-------+-------+-----------+------------+-----------+
 barn_route  barn_policy  barn_caller  barn_objects  barn_wasm   barn_query  barn_fetch
     |           |          (R0)       (R0)  |       (R0) |           |
     +-----+-----+           |          barn_ids   barn_abi (R0)   barn_cache
           |                 |                       |
     barn_config      permit_token            barn_iron (R0)
           |          permit_require                 |
     rancher_model                             common_merge
           \__________ _client interfaces: pond, rancher, permit, sun, barker, store _____/
                                     |
                         common_wire, common_log, common_*
```

## Quality seeds

### R0

**barn_caller.** *Promise:* a request's subject and grants are exactly what a valid, unrevoked token from this tenant's realm says, and a requirement is satisfied only by a clause after substituting the route's typed parameters; host roles satisfy only host clauses. *Tests:* tokens from another realm, expired, revoked mid-request, signed by a rotated-out key, and forged; every clause shape; parameters with `/`, `:`, and `_` in them; fuzzed token bytes. *Won't:* call permit on the hot path when the cache is warm; decide objects.

**barn_objects.** *Promise:* a slot's id comes only from a reserved value, a parameter the requirement bound, a fresh mint, or a name pond bound; nothing a box or caller sends can choose another. *Tests:* a table of slot definitions against requests that try every unbound parameter, a path param with a different type, a name slot that would create on a read-only slot. *Won't:* load anything the route didn't name; cache across tenants.

**barn_iron.** *Promise:* a request's staged calls become exactly one pond write: every op on a resolved slot with merge or delete access, `if-seq` equal to the seq loaded, the request id as idempotency key, and effects only from the route's `gives`, `schedules`, and mail services; a failed or shadow run produces none. *Tests:* property tests over random call sequences comparing the composed patch with sequential RFC 7386 application; every `iron-error`; caps. *Won't:* talk to pond, run a box, retry a conflict.

**barn_abi.** *Promise:* no host call reads or writes guest memory outside the range it was given, and every host call's arguments are validated before use. *Tests:* fuzzed pointers and lengths at every import, zero-length, overflow, and out-of-bounds ranges; generated WIT bindings checked against the published `barn-abi@1` and `iron@1`. *Won't:* hold state between requests; decide policy.

**barn_wasm.** *Promise:* a box gets only the imports its world declares; fuel, memory, and deadline are always enforced; `reset` restores memory and globals to the snapshot byte for byte, verified on a sample of resets and on every reset in tests; `tenant` and `shared` never mix what they promise not to. *Tests:* hostile modules (infinite loops, memory growth, deep recursion, stray imports, a box that stashes data in globals and checks for it after a reset); reset cost measured in a benchmark that fails on regression. *Won't:* know about routes, tenants beyond a pool key, or pond.

### R1

**barn_config.** *Promise:* a `View` is a consistent cut at one config seq; events apply in order; a gap means a resnapshot, never a skip. *Tests:* replayed logs, gaps, a pull arriving before the policy that needs it. *Won't:* change configuration.

**barn_route.** *Promise:* every request resolves to exactly one route or none, with goober's conflict rules (types count). *Tests:* a corpus of routes and requests with expected matches. *Won't:* check permission.

**barn_policy.** *Promise:* the same split key always gets the same version under the same policy; weights converge. *Tests:* distribution tests, stability across restarts. *Won't:* roll back (rancher does).

**barn_ids.** *Promise:* an id is minted once; a range is never used after barn restarts. *Tests:* exhaustion, lease failure, concurrency. *Won't:* persist anything.

**barn_cache.** *Promise:* a cached answer is never served after pond reports a change past it, and keys always carry tenant and grant. *Tests:* feed reorderings, resets, eviction under memory pressure. *Won't:* be required for correctness.

**barn_eval.** *Promise:* the nine steps run in order, a failure at any step writes nothing, and every outcome maps to one status and one counter. *Tests:* against fakes of every client, one test per row of the status table. *Won't:* make an access decision itself.

**barn_query, barn_fetch, barn_proxy, barn_edge, barn_sse, barn_mail, barn_timer, barn_stitch.** Each promises its protocol faithfully and its limits always; each is tested against fakes and the in-memory network. `barn_proxy` *won't* read a body or add a credential; `barn_fetch` *won't* reach an undeclared origin; `barn_timer` *won't* hold more events than its credit; `barn_stitch` *won't* let a slot's output be scanned for directives.

## How it's wired

`barn_system` builds, in this order:

1. **Clients.** One wire connection to each peer: rancher, pond, permit, sun, barker, store, ranger. `common_log`'s spool starts shipping to ranger.
2. **Config.** `barn_config` opens `rancher.config.subscribe` and waits for the snapshot.
3. **Modules.** `barn_modules` fetches every live box version named in the snapshot; `barn_wasm` compiles them and takes reset snapshots.
4. **Ids.** `barn_ids` leases its first range.
5. **Subscriptions.** `permit.keys.subscribe`, `barker.suppression.subscribe`, and `pond.feed.subscribe` for cache invalidation.
6. **Ready.** Health turns 200. Only now does `barn_timer` open `sun.fire.subscribe`, and only now does Caddy send traffic.

Barn serves nothing until step 6: no snapshot, compiled boxes, and id range means a failing health check and no timer credit.

**Threads.** As in goober, two pools so network work and box work never steal from each other:

| pool | runs | size |
| --- | --- | --- |
| network | one `io_context`: the edge, LMTP, every wire connection, the cache, timers | a few threads |
| box | box calls, run to completion on a box thread, optionally pinned to cores | one per core given to boxes |

A request is parsed on a network thread, posted to the box pool when it needs a box, and its pond calls are async on the network pool. Credit to sun is the box pool's free capacity.

**Shutdown.** Health turns 503; barn stops granting sun credit and acks or fails what it holds; it finishes in-flight requests up to their deadlines; SSE streams close with a retry hint; it sends GOAWAY on every connection; the spool flushes to ranger. An unused id range is simply abandoned (gaps are fine).

## Review queue

| order | packages | tier |
| --- | --- | --- |
| 1 | `barn_iron`, `barn_abi`, `barn_objects`, `barn_caller`, `barn_wasm` | R0: every line, adversarial agent review first, fuzzing required |
| 2 | `barn_eval`, `barn_config`, `barn_route`, `barn_policy`, `barn_cache`, `barn_ids`, `barn_query`, `barn_proxy`, `barn_fetch` | R1, the request path |
| 3 | `barn_edge`, `barn_sse`, `barn_mail`, `barn_timer`, `barn_stitch` | R1, the protocols |
| 4 | `barn_mcp`, `barn_modules`, `barn_routelog`, `barn_system` | R2 |
| 5 | `barn_main` | R3 |

**Build order.** Agents can work in parallel along these lanes, each against fakes, meeting only at `barn_eval`:

1. `barn_iron` and `barn_abi` (the ABI, against a test harness box), with `barn_wasm` beside them.
2. `barn_config`, `barn_route`, `barn_policy` (against recorded config logs).
3. `barn_caller` (against `permit_token` test vectors), `barn_objects` and `barn_ids` (against `pond_fake`).
4. `barn_cache` and `barn_query` (against `pond_fake`).
5. `barn_eval`, then `barn_edge` and `barn_proxy`, which make the first HTTP request work end to end in `sim_world`.
6. The protocols (`barn_sse`, `barn_mcp`, `barn_mail`, `barn_timer`, `barn_stitch`, `barn_fetch`) in any order.

## The invariants

- **The R0 line is reach.** Code that decides who the caller is, which keys a box gets, what it writes, or what it can touch in memory is R0; nothing else is.
- **`barn_eval` sequences, it doesn't decide.** Every access decision lives in an R0 package.
- **No `barn_server`, `barn_proto`, `barn_client`, or `barn_fake`.** Nothing dials barn.
- **Every other service is reached through its `_client` interface,** and every one has a fake barn's tests use.
- **Network threads never run a box; box threads never block on the network.**
- **Barn serves nothing until it has a snapshot, every live box compiled, and an id range.**
- **Shutdown drains.** Requests finish, timer events are acked or failed, and the spool flushes.
