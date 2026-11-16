# Rancher packages
How the control plane is cut, and where a person must read every line
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Rancher is where security lives: every route says who may call, which objects a request may touch, and which tables it may read, and rancher is the one place those decisions are checked before a barn ever sees them. So rancher's R0 line is drawn around **the checks and the review**, not around the database or the API. Three packages are R0: `rancher_route` (is this route well-formed, and does it collide with another), `rancher_check` (does this route, with every box version its service may run, stay inside what was granted), and `rancher_promote` (does a person see every widening before production changes). Everything else (storing configuration, publishing the log, applying to pond and permit, counting health, serving HTTP) is plumbing that those three are called from, and it is kept out of them.

The checks are pure functions over `rancher_model` values. They read no database, call no service, and know no clock, which is what keeps them small enough to read in full and test exhaustively. Fork's verdicts are an input to `rancher_check`, computed by the caller and passed in.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `rancher_model` | `homestead::rancher::model` | **Library.** Tenants, domains, boxes, services, policies, routes, tables, schedules as plain data, with FlatBuffers encoding. Linked by barn and store. | `common_core`, `rancher_proto` | R1 |
| `rancher_proto` | `homestead::rancher::proto` | The `.fbs` schema and generated code | `common_wire` | R3 |
| `rancher_db` | `homestead::rancher::db` | Rancher's SQLite schema and statements: objects by key and env, the config log, problems, promotions | `common_sqlite`, `rancher_model` | R2 |
| `rancher_route` | `homestead::rancher::route` | Route keys, typed paths, the one-route-or-none rule, overlap detection | `rancher_model` | R0 |
| `rancher_check` | `homestead::rancher::check` | Ruleset vs objects for every version, bound params, grants and named queries against fork verdicts, the schedules widening chain, `widens` sentences | `rancher_model`, `rancher_route`, `fork_api` | R0 |
| `rancher_promote` | `homestead::rancher::promote` | The beta/production diff, partial promotion closure, the approval rule | `rancher_model`, `rancher_check` | R0 |
| `rancher_configlog` | `homestead::rancher::configlog` | Seq'd events, snapshots, the subscriber fan-out, `reset` | `rancher_db`, `rancher_model` | R1 |
| `rancher_box` | `homestead::rancher::box` | Upload: hash, store the module, inspect WASM imports and exports against `abi` | `rancher_model`, `common_crypto`, `common_blob` | R1 |
| `rancher_policy` | `homestead::rancher::policy` | Weights, shadows, ramps, guardrails, automatic rollback | `rancher_model`, `common_core` | R1 |
| `rancher_tenant` | `homestead::rancher::tenant` | Tenants, domains and their roles, DNS verification | `rancher_model`, `rancher_db` | R1 |
| `rancher_apply` | `homestead::rancher::apply` | The reconciler: drives pond, permit, sun, and barker control calls until they land | `rancher_db`, `pond_client`, `permit_client`, `sun_client`, `barker_client` | R1 |
| `rancher_ops` | `homestead::rancher::ops` | The operator tools: dry run, `if_seq`, `widens`, undo, scopes, the proxies to ranger, sun, and barker | most core packages, `ranger_client` | R1 |
| `rancher_problems` | `homestead::rancher::problems` | The problem log: group by fingerprint, cap, compact, reopen | `rancher_db`, `common_problem` | R2 |
| `rancher_health` | `homestead::rancher::health` | The barn registry, per-route counts, readiness of box versions, staleness | `rancher_db`, `rancher_model` | R2 |
| `rancher_http` | `homestead::rancher::http` | The operator API as HTTP endpoints and MCP tools, over `http.serve` | `rancher_ops`, `common_http` | R2 |
| `rancher_server` | `homestead::rancher::server` | Binds wire methods to core, checks the caller's service | `rancher_proto`, core packages | R2 |
| `rancher_client` | `homestead::rancher::client` | The `Rancher` interface and its wire implementation | `rancher_proto`, `rancher_model`, `common_wire` | R2 |
| `rancher_fake` | `homestead::rancher::fake` | An in-memory rancher: config log, box fetch, report sinks | `rancher_client` | R2 |
| `rancher_system` | `homestead::rancher::system` | The composition root | everything above | R2 |
| `rancher_main` | | The binary | `rancher_system` | R3 |

The operator tools live in `rancher_ops`, not `rancher_operator`: `operator` is a C++ keyword, and the directory must name the namespace.

## Dependencies

```
                           rancher_main
                                |
                          rancher_system
            +-------------+-----+-------+-------------+
            |             |             |             |
      rancher_server  rancher_http  rancher_apply  rancher_configlog
            |             |             |   \             |
            +------> rancher_ops   |    pond/permit/sun/barker_client
                     /   |    |    \    |
        rancher_promote  |    |  rancher_policy   rancher_box   rancher_tenant
              |          |    |                              \     /
        rancher_check ---+  rancher_problems  rancher_health  rancher_db
           /     \                                               |
  rancher_route  fork_api                                  common_sqlite
           \
        rancher_model ---> rancher_proto ---> common_wire
```

The R0 packages (`rancher_route`, `rancher_check`, `rancher_promote`) sit on `rancher_model` and `fork_api` alone. Nothing R0 reaches the database, the wire, or a clock.

## Quality seeds

### rancher_route (R0)
- **Promise.** Every request resolves to exactly one route or none. Typed paths compare by type (`/jobs/{id:int}` and `/jobs/new` don't overlap; `/jobs/{id}` and `/jobs/{slug}` do). A route overlapping another is refused unless a traffic decision names both.
- **Tests.** A table of path pairs with the expected overlap verdict; a property test that for any accepted route set and any generated request, at most one route matches. Barn's matcher runs the same tables.
- **Won't.** Store routes, know about boxes, or decide traffic.

### rancher_check (R0)
- **Promise.** No route gives any box version its service may run a merge or delete slot outside that version's `writes`. Every `{param}` an object uses is bound by the permit requirement. Every grant and named query passed fork against the table versions' columns. Every `schedules` entry is shown with what the scheduled timer route can write.
- **Tests.** One test per rule per direction (route put, policy put, box upload, table drop); a generated corpus of routes and policies checked against a slow, obviously correct reference checker; `widens` sentences gold-tested.
- **Won't.** Call fork itself (verdicts are inputs), read the database, or soften a refusal into a warning.

### rancher_promote (R0)
- **Promise.** The diff names every box, route, policy, table, schedule, release, and widening that differs between production and beta. A partial promotion is refused unless its closure (box versions, table versions, timer routes it schedules) comes with it. Only a person's session can approve.
- **Tests.** Diffs of generated environment pairs checked to be complete (applying the diff to production yields beta); partial closures; approval refused for agent, capability, and operator-without-tenant tokens.
- **Won't.** Apply anything (it answers a plan; `rancher_ops` applies it), or summarize away a widening.

### rancher_model (R1, library)
- **Promise.** One definition of every configuration object, round-tripped through FlatBuffers and JSON without loss. Linked by barn and store.
- **Tests.** Round trips; decoding objects written by the previous release's schema.
- **Won't.** Validate meaning (that is `rancher_route` and `rancher_check`), or do I/O.

### rancher_configlog (R1)
- **Promise.** Subscribers see every event once, in seq order, after a snapshot consistent at its seq; a `from` older than the log gets `reset` and a snapshot, never a gap.
- **Tests.** Concurrent writes and subscribes against the in-memory wire; snapshot equals replay of the log; resume at every seq.
- **Won't.** Decide what an event means to a barn.

### rancher_box (R1)
- **Promise.** A stored module's hash is its id; its imports are only the ABIs it lists; its exports include every one it names.
- **Tests.** Malformed, oversized, and lying modules; re-upload is idempotent.
- **Won't.** Run a module, or compile one.

### rancher_policy (R1)
- **Promise.** Weights sum to 100; split hashes are stable; a breached guardrail returns weight to the previous version within one health window and raises `ramp.guardrail`.
- **Tests.** Ramp scenarios driven by synthetic health windows on a manual clock.
- **Won't.** Roll forward on its own.

### rancher_tenant (R1)
- **Promise.** A domain belongs to exactly one tenant; a tenant has exactly one primary domain; hard-verified deployments don't serve unverified domains.
- **Tests.** Domain moves, collisions, verification with a fake resolver.
- **Won't.** Create pond databases or realms (it records intent; `rancher_apply` drives them).

### rancher_apply (R1)
- **Promise.** Every accepted change that pond, permit, sun, or barker must know is applied, idempotently, keyed by config seq, and retried until it lands; persistent failure is `apply.lagging`.
- **Tests.** Against the fakes, with injected failures and restarts mid-apply.
- **Won't.** Decide whether a change is allowed.

### rancher_ops (R1)
- **Promise.** Every changing tool supports `dry_run`, honors `if_seq`, answers `widens` and `undo`, and refuses production edits except promotion, rollback, and risk-lowering policy changes.
- **Tests.** Every tool with dry run equals the real answer minus storage; every undo restores the prior config; scope checks per token kind.
- **Won't.** Check routes itself; it calls `rancher_check`.

## How it's wired

`rancher_system` builds, in order: the clock and network, `rancher_db`, the model codecs, `rancher_configlog`, `rancher_problems`, `rancher_health`, the checks, `rancher_policy`, `rancher_tenant`, `rancher_box`, the clients to pond, permit, sun, barker, and ranger, `rancher_apply`, `rancher_ops`, `rancher_http`, and finally `rancher_server`, which opens the listener.

- **Threads.** One Asio context for the wire and HTTP; one writer thread for `rancher_db` (every change is a single serialized transaction that appends to the config log); a small pool for fork checks during route puts.
- **Start.** Open the database, replay nothing (the log is the database), start `rancher_apply`'s reconciliation of the whole view, then listen.
- **Shutdown.** GOAWAY to subscribers, finish in-flight transactions, stop the reconciler at a clean point, close the database. Barns keep serving their last config.

## Review queue

| tier | packages |
| --- | --- |
| R0 | `rancher_check`, `rancher_route`, `rancher_promote` |
| R1 | `rancher_model`, `rancher_configlog`, `rancher_ops`, `rancher_apply`, `rancher_policy`, `rancher_tenant`, `rancher_box` |
| R2 | `rancher_db`, `rancher_problems`, `rancher_health`, `rancher_http`, `rancher_server`, `rancher_client`, `rancher_fake`, `rancher_system` |
| R3 | `rancher_proto`, `rancher_main` |

Build order, so agents can work in parallel:

1. `rancher_proto`, `rancher_model`.
2. In parallel: `rancher_route`, `rancher_db`, `rancher_policy`, `rancher_box`, `rancher_client` + `rancher_fake` (unblocks barn and store).
3. In parallel: `rancher_check` (needs `fork_api`), `rancher_configlog`, `rancher_tenant`, `rancher_problems`, `rancher_health`.
4. `rancher_promote`, `rancher_apply` (needs the other services' fakes), `rancher_ops`.
5. `rancher_http`, `rancher_server`, `rancher_system`, `rancher_main`.

## The invariants

- **The checks are pure.** `rancher_route`, `rancher_check`, and `rancher_promote` read no database, call no service, and take no clock.
- **R0 is three packages,** and stays small enough to read in full.
- **One model.** Barn and store read configuration through `rancher_model`, never a copy of it.
- **Every change goes through `rancher_ops`,** so dry run, `if_seq`, `widens`, and undo can't be skipped by a new tool.
- **Only `rancher_apply` calls control APIs,** and only `rancher_system` builds it.
