# Porch packages
A small library in barn, with one tiny R0 check at its center
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Porch is mostly plumbing: gather catalogs, speak MCP, forward calls. The one place a bug matters is the question "may this caller see and call this tool, through this path?", because a wrong answer hands a content agent the deploy tools. That question is `porch_roles`, kept to a pure function over a role tree, a tool's roles, a path role, and a grant list, and it is the only R0 package. It is also a library the owners (rancher, store) link to re-check every call, so the first lock and the second lock are the same code and can't drift.

Everything else is R1 or R2: the MCP protocol is careful work but can't grant anything `porch_roles` refused, and the HTTP form is a thin adaptation of it.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `porch_roles` | `homestead::porch::roles` | **Library.** The role tree (parse, `contains`, `delegates`), `holds(grants, role)`, `visible(tool, path_role, grants)`. Pure. Linked by barn, rancher, store, permit. | `common_core`, `common_json` | R0 |
| `porch_contract` | `homestead::porch::contract` | **Library.** `porch.fbs`: `ToolDescriptor`, `Catalog`, `Caller`, `CallRequest`, `CallAnswer`, and their JSON forms. Included by `rancher_proto` and `store_proto`. | `common_wire`, `common_json` | R1 |
| `porch_catalog` | `homestead::porch::catalog` | Sources, per-owner caches by version, merging, name clash detection, invalidation, the visible view per caller and path | `porch_roles`, `porch_contract`, `common_core` | R1 |
| `porch_dispatch` | `homestead::porch::dispatch` | A call: find the owner, check `visible`, build the `CallRequest`, forward with the deadline left, map codes, write the `agents` line | `porch_catalog`, `common_log` | R1 |
| `porch_mcp` | `homestead::porch::mcp` | MCP over streamable HTTP: JSON-RPC, sessions, SSE notifications, paging, the 401/403 challenges, protected resource metadata | `porch_dispatch`, `common_http`, `common_json` | R1 |
| `porch_http` | `homestead::porch::http` | `/_ops/api`: list and call as JSON, cookie `Origin` check, status mapping | `porch_dispatch`, `common_http` | R2 |
| `porch_builtin` | `homestead::porch::builtin` | `guide` and `whats_new` from the role tree and catalog news | `porch_catalog` | R2 |
| `porch` | `homestead::porch` | The facade barn links: `Porch`, `ToolSource`, `Config`; routes a request to `porch_mcp` or `porch_http` | all of the above | R2 |
| `porch_fake` | `homestead::porch::fake` | A `ToolSource` with tools declared in the test, recording calls; and a contract test suite every `ToolSource` must pass | `porch_contract` | R2 |

Porch has no proto, server, client, system, or main: it is a library, and barn is its composition root. The wire methods it calls are rancher's and store's, declared in their protos using `porch_contract`'s types.

## Dependencies

```
                 barn_system
                     |
                   porch  (facade)
                 /        \
           porch_mcp     porch_http
                 \        /
              porch_dispatch ----> common_log
                     |
              porch_catalog   porch_builtin
                /        \        |
       porch_roles   porch_contract
           ^   ^            ^   ^
           |   |            |   |
   rancher_tools  store_tools   rancher_proto, store_proto
   permit_agents (the role tree for delegation)
```

## Quality seeds

### porch_roles (R0, library)
- **Promise.** `visible` is true only when the tool is tagged with a role the caller holds, and, with a path role, only when that role is among the tool's roles and the caller holds it. `holds` follows `contains` in the tree and nothing else; `ops/admin` holds every role; `host:admin` holds every role; no other grant implies a role. A tool with no roles is visible to `admin` only. An unknown role in the path is visible to no one.
- **Tests.** The full table of (tree, tool roles, path role, grants) for the seven starting roles; property tests: adding a path role never makes more tools visible; removing a grant never makes more tools visible; a tree change that only adds roles never hides a tool from `admin`. Hostile inputs: `ops/` with empty role, `ops/admin/x`, case and unicode lookalikes, `host:` roles on tenant tools.
- **Won't.** Verify tokens, read the clock, or know any tool by name.

### porch_contract (R1, library)
- **Promise.** One schema for descriptors, callers, and calls; fields only added; JSON and FlatBuffers forms round trip.
- **Tests.** Round trips; decoding the previous release's messages.
- **Won't.** Decide anything.

### porch_catalog (R1)
- **Promise.** The view a caller gets is exactly the tools `porch_roles` says are visible, from the newest catalog each owner answered; a name two owners publish is shown once, from the first owner, and reported; an owner that is down serves its last catalog, marked stale.
- **Tests.** Against `porch_fake` sources: version moves, unchanged answers, clashes, an owner down at start (no catalog, no tools, a problem), invalidation by config seq.
- **Won't.** Filter by anything but `porch_roles`.

### porch_dispatch (R1)
- **Promise.** A call reaches an owner only if `visible` said yes for this caller and path; arguments go through byte for byte; the deadline passed is what is left; one `agents` line per call, with lineage and without arguments.
- **Tests.** Invisible tools answer `not_found` and never reach a source; argument passthrough including `dry_run` and `if_seq`; code mapping; log lines checked for absent argument values.
- **Won't.** Retry a changing call on its own.

### porch_mcp (R1)
- **Promise.** Conforms to MCP's streamable HTTP transport: JSON-RPC ids echoed, sessions issued and expired, `tools/list_changed` sent when the visible set changes, 401 and 403 with the right `WWW-Authenticate`.
- **Tests.** A recorded conversation from a reference MCP client per role path; session expiry under a manual clock; list changes on catalog moves; malformed JSON-RPC; fuzzed request bodies.
- **Won't.** Hold anything a restart can't lose.

## How it's wired

Barn's `barn_system` builds two `Porch` instances: one for `ops.` domains, with `ToolSource`s over `rancher_client` and `store_client`; one for tenant domains, with a source over barn's route table (tenant `mcp` routes and their `roles`). `barn_edge` sends `/mcp`, `/mcp/{role}`, `/_ops/api`, and the protected resource metadata paths to the right one, after `barn_caller` has built the `Caller`. Rancher's config events of kind `catalog.put` and `role.put` call `invalidate`.

Porch runs on barn's network threads; it never blocks, since every source call is a future. MCP sessions are a bounded map (default 10,000 per barn, oldest idle evicted).

## Review queue

| tier | packages |
| --- | --- |
| R0 | `porch_roles` (also reviewed as part of every linker: barn, rancher, store, permit) |
| R1 | `porch_contract`, `porch_catalog`, `porch_dispatch`, `porch_mcp` |
| R2 | `porch_http`, `porch_builtin`, `porch`, `porch_fake` |

Build order: `porch_roles` and `porch_contract` first, since rancher's and store's tool packages and permit's delegation depend on them; then `porch_fake` and `porch_catalog`; then `porch_dispatch`; then `porch_mcp`, `porch_http`, `porch_builtin` in parallel; then the facade. Barn integrates last, against `porch_fake` until rancher and store answer `tools.list`.

## The invariants

- **One R0 package, pure, shared by porch and every owner.** The first and second locks are the same code.
- **Porch is a library; barn is its root.** No state that matters, no binary.
- **No tool reaches an owner unless `porch_roles` said visible.**
- **Arguments and answers pass through untouched.**
