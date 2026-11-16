# Rancher API
What barns follow, what every service reports, and what operators and agents change
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Rancher has three audiences, and its API is cut the same way. **Barns** follow it: a snapshot of the live configuration, then every change in order, and the modules and OpenAPI specs it names. **Every service** reports to it: problems, and from barn and sun, health counts. **Operators and agents** change it: tenants, domains, boxes, services, routes, tables, schedules, secrets, promotions, and the problem log. The first two are wire methods, small and fixed. The third is the operator tools, which are many and grow, so they ride two wire methods, `rancher.tools.list` and `rancher.tools.call`, which [porch](../porch/IDEA.md) (a library inside barn) serves to people and agents over MCP and HTTP on `ops.` domains, filtered by role.

Rancher also calls out: it is the only caller of pond's, permit's, sun's, and barker's control APIs, and the proxy for ranger's reads. Those calls are listed at the end, so the whole control plane is visible in one place.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `rancher.config.subscribe` | stream | barn, barker, store | no | A snapshot at a seq, then config events in seq order |
| `rancher.box.fetch` | call (chunked answer) | barn | no | A module's bytes by hash |
| `rancher.spec.fetch` | call (chunked answer) | barn | no | An OpenAPI spec's bytes by hash |
| `rancher.health.report` | push | barn, sun | yes (counts) | Acknowledged seq |
| `rancher.problems.report` | push | every service | yes (the problem log) | Acknowledged seq |
| `rancher.tables.get` | call | store | no | Table versions with columns and types |
| `rancher.release.submit` | call | store | yes (beta) | The release's routes, checked, and the config seq that serves them |
| `rancher.release.propose` | call | store | yes (a pending promotion) | A promotion id for a person to approve |
| `rancher.tools.list` | call | barn (porch) | no | Tool descriptors, with roles, including the sun, barker, and ranger tools rancher proxies |
| `rancher.tools.call` | call | barn (porch) | depends on the tool | The tool's answer, with `seq`, `widens`, `undo`, and `problems` for changing tools |

`rancher.tools.*` replaces the earlier `rancher.http.serve`: rancher no longer speaks HTTP or MCP at all, porch does.

Every changing wire call carries an idempotency key in `Open`, as [network](../common/NETWORK.md#calls) requires.

## Following the configuration

### `rancher.config.subscribe`

```json
{ "from": 8812, "env": ["beta", "production"], "kinds": [] }
```

| field | meaning |
| --- | --- |
| from | The last seq the subscriber applied, or 0 for a cold start. 0, or a seq older than the log keeps, starts with a snapshot. |
| env | Which environments to follow. Barn follows both; store follows only what it needs. |
| kinds | Optional filter. Barker asks for `template.*` and `domain.*`; store for `table.*` and `release.*`. Empty is everything. |

The stream starts with a snapshot when `from` is 0 or too old, then follows the log:

```json
{ "snapshot": { "seq": 8812, "chunk": 0, "last": false, "objects": [ { "kind": "route", "env": "production", "route": { } } ] } }
{ "snapshot": { "seq": 8812, "chunk": 7, "last": true, "objects": [ ] } }
{ "event": { "seq": 8813, "kind": "box.pull", "box": "accounts@sha256:77ab02..." } }
{ "event": { "seq": 8814, "kind": "service.policy", "env": "beta", "tenant": "bobs_plumbing", "service": "accounts", "policy": { } } }
{ "event": { "seq": 8815, "kind": "route.put", "env": "beta", "tenant": "bobs_plumbing", "route": { } } }
```

| event kind | carries | barn does |
| --- | --- | --- |
| `tenant.put`, `tenant.state` | A tenant, its state and quotas | Serves, suspends, or refuses its domains |
| `domain.put`, `domain.drop` | A domain, its tenant, and its role: primary, redirect, beta, ops | Updates the domain registry |
| `box.pull` | A box version barns must compile before traffic | Fetches, compiles, reports ready in `health.report` |
| `spec.put` | An OpenAPI spec by hash, for egress bindings | Fetches it with `spec.fetch` before a route that names it goes live |
| `secret.put`, `secret.drop` | A sealed secret by name, sent only to the services in its `for` list | Keeps the ciphertext; opens it with cellar at use |
| `role.put` | The deployment's role tree (also followed by permit, for consent and delegation) | Porch filters `/mcp/{role}` with it |
| `catalog.put` | A tool owner's catalog moved to a new version (a rancher or store build) | Porch drops that owner's cached catalog |
| `box.drop` | A version no policy names any more | Frees its instances; keeps the module on disk |
| `service.policy` | A service's policy in one environment | Picks versions by the new weights |
| `route.put`, `route.drop` | A whole route, in one environment | Rebuilds the routing table |
| `table.put`, `table.drop` | A table version, for grant resolution | Nothing beyond caching |
| `schedule.put`, `schedule.drop` | A recurring schedule (for barn's timer routes) | Nothing; sun fires |
| `route.put` for an `event` route | The route, with `env` and `shadow: true` while it is beta-only | Runs events pond leases to it; a shadow's merges are recorded and discarded |
| `release.put`, `release.policy` | A store release and the `site` service's policy | Fetches the release from store, then serves it |
| `template.put` | A mail template | Barn checks box mail against it; barker renders it |
| `limits.put` | Rate and quota limits per tenant and route | Enforces them |
| `reset` | The subscriber's `from` was too old | Drops its view and waits for the snapshot that follows |

- **Events are whole objects, not diffs.** A `route.put` carries the entire route, so applying the log is replacing by key.
- **A pull comes before traffic.** Rancher publishes a `service.policy` that names a version only after every healthy barn has reported that version ready.
- **Barns report what they hold.** A barn's `health.report` carries the seq it applied, so rancher knows how stale each barn is.

### `rancher.box.fetch`

```json
{ "module": "sha256:9f3c41...", "offset": 0 }
```

The answer is a stream of chunks of at most 1 MiB, each `{ "offset": 0, "bytes": "...", "last": false }`, then a `Status`. Modules are content-addressed, so a barn checks the hash of what it received and may resume from an offset. `not_found` means the module was never uploaded.

### `rancher.spec.fetch`

The same shape as `box.fetch`: `{ "spec": "sha256:41d0...", "offset": 0 }`, answered in chunks. Barn checks the hash and caches specs forever.

## Reports

### `rancher.problems.report`

A push stream. Each batch is already grouped by fingerprint at the source (`common_problem`):

```json
{
	"seq": 412,
	"service": "pond", "host": "pond-1",
	"problems": [
		{
			"problem": "tailer.type_mismatch", "fingerprint": "users@1/zip/string",
			"tenant": "bobs_plumbing", "count": 312, "first-seen": 1762870000000, "last-seen": 1762874400000,
			"where": { "table": "users@1", "column": "zip", "bucket": "addresses" },
			"examples": [ { "bucket": "addresses", "id": "4294967311", "seq": 4471, "shape": { "zip": "string" } } ],
			"caused-by": { "box": "accounts@sha256:77ab02...", "route": "PUT bobsplumbing.com/me/address" }
		}
	]
}
```

Rancher answers `{ "acked": 412 }` once the batch is in its database. A problem code not in the catalog is accepted and filed as `catalog.unknown_code`, so a newer service never loses a report to an older rancher.

### `rancher.health.report`

```json
{
	"seq": 9001, "service": "barn", "host": "barn-7", "config-seq": 8815,
	"ready": ["accounts@sha256:77ab02..."],
	"window-ms": 5000,
	"routes": [
		{ "tenant": "bobs_plumbing", "env": "production", "route": "PUT bobsplumbing.com/me/address",
		  "version": "accounts@sha256:77ab02...",
		  "outcomes": { "ok": 412, "permit": 3, "trap": 0, "fuel": 0, "timeout": 1, "conflict": 2, "hold-timeout": 0 },
		  "ms": { "p50": 4.1, "p99": 41.0 }, "fuel": { "p99": 512000 } }
	]
}
```

Sun's report has the same shape with timer outcomes (`fired`, `done`, `failed`, `expired`, `dead`) and lateness. These counts drive guardrails, so they are a push with acknowledgment, not a log.

## Store's calls

| method | request | answer |
| --- | --- | --- |
| `rancher.tables.get` | `{ "tenant": "bobs_plumbing", "tables": ["jobs@2"] }`, or no list for every version in use | `{ "tables": [ { "table": "jobs", "version": 2, "columns": { "id": "id", "status": "text" } } ] }` |
| `rancher.release.submit` | `{ "tenant", "release": "sha256:...", "routes": [ ... ], "queries": { ... } }` | `{ "ok": true, "config-seq": 9120, "widens": [ ... ] }`, or `invalid` with every refused route and why |
| `rancher.release.propose` | `{ "tenant", "release": "sha256:...", "note": "..." }` | `{ "promotion": "4294970001", "url": "https://ops.bobsplumbing.com/_ops/promotions/4294970001" }` |

A submitted release lands in beta at once, as the `site` service's live version there. Production moves only when a person approves a proposed promotion.

## The operator tools

### `rancher.tools.list` and `rancher.tools.call`

```json
{ "caller": { "tenant": "bobs_plumbing", "subject": "4294967100", "kind": "agent", "grants": ["ops/content", "ops/observe"] } }
```

```json
{ "tools": [
	{ "name": "problem_list", "summary": "Problems, newest first", "roles": ["observe"], "changes": false, "input": { } },
	{ "name": "route_put", "summary": "Put a route in beta", "roles": ["deploy"], "changes": true, "input": { } } ],
  "roles": { "admin": ["deploy", "content", "data", "observe", "mail", "identity"] } }
```

```json
{ "tool": "route_put", "args": { "env": "beta", "route": { }, "dry_run": true },
  "caller": { "tenant": "bobs_plumbing", "subject": "4294967100", "kind": "agent", "grants": ["ops/deploy"] } }
```

| field | meaning |
| --- | --- |
| caller | Who porch says is calling: tenant (from the domain), subject, token kind, and grants, all verified by barn. Rancher trusts it from barn and nobody else. |
| roles | A tool's roles. A caller may call a tool only if it holds `ops/{role}` for one of them, or `ops/admin`, or is a host operator; rancher checks again on every call, whatever porch filtered. |
| changes | Whether the tool changes anything; changing tools take the fields below and are refused for read-only tokens. |
| input | The tool's JSON Schema, the same for MCP and HTTP. |

`tools.list` answers every tool the caller may call, with the role tree; porch narrows it by the `/mcp/{role}` path. Tools rancher proxies (sun's and barker's dead letters, ranger's logs) appear in the list with their own roles (`observe`, `mail`) and are forwarded by rancher with the caller's scope.

Porch serves these tools at `/mcp`, `/mcp/{role}`, and `/_ops/api/{tool}` on a tenant's `ops.` domain (and the platform's own ops domain for host operators). Each tool is one HTTP endpoint with a JSON body; the MCP tool and the endpoint share a name and a schema.

### Every changing call

Goober's engineering practice, applied uniformly:

| field | in | meaning |
| --- | --- | --- |
| `dry_run` | request | Check everything, store nothing, answer exactly as if it had happened. |
| `if_seq` | request | Refuse with `conflict` unless the object is still at this config seq. Required for edits; absent only on creates. |
| `env` | request | `beta` (the default) or `production`. Production refuses every call except `promote`, `rollback`, and `policy` changes that only lower risk. |
| `seq` | answer | The config seq the change landed at. |
| `widens` | answer | Every way the change adds reach, as sentences: a new bucket in `writes`, an object slot moved from read to merge, a column granted, a route made public, a schedules chain. |
| `undo` | answer | The call that reverses it, ready to send. |
| `problems` | answer | Problems already open on the objects touched, so an agent sees them before it builds on them. |

```json
{
	"seq": 8815,
	"widens": ["PUT bobsplumbing.com/me/address may now merge into addresses/{!user_id} (was read)."],
	"undo": { "tool": "route_put", "args": { "env": "beta", "route": { "...": "the previous route" }, "if_seq": 8815 } },
	"problems": []
}
```

### Tools

| group | tool | does |
| --- | --- | --- |
| guide | `ops_guide` | The model of the platform in one page, for an agent's first call. |
| tenants | `tenant_create`, `tenant_get`, `tenant_suspend`, `tenant_delete`, `tenant_quotas_put` | Host operators only. Create drives pond and permit. |
| domains | `domain_put`, `domain_drop`, `domain_verify` | Add a domain with a role; verification answers the DNS record to publish and whether it is there. |
| boxes | `box_upload`, `box_get`, `box_list`, `box_diff` | Upload checks imports against `abi`, exports, and `writes`; answers `widens` against the previous version of the same box name. |
| services | `service_put`, `policy_put`, `ramp`, `shadow`, `rollback`, `service_health` | `ramp` moves weight in steps with guardrails; `rollback` returns to the previous policy. |
| routes | `route_put`, `route_drop`, `route_get`, `route_list`, `route_check`, `route_try` | `route_check` is the full check without storing; `route_try` runs a route on any version, dark ones included, through a barn, with writes discarded. Event routes are tried with a synthetic event. |
| events | `event_status`, `event_dead_list`, `event_redrive`, `event_discard` | Each event route's cursor lag, and its dead events, through `pond.control.events.*`. Roles: `data`, `observe`. |
| specs | `spec_upload`, `spec_get`, `spec_list` | OpenAPI documents for egress, stored by hash. Role: `deploy`. |
| secrets | `secret_put`, `secret_list`, `secret_drop`, `secret_rotate` | Write-only: put seals the value with cellar and stores ciphertext; list answers names, `for`, fingerprints, and dates, never values; rotate puts a new value under the same name and keeps the previous one until barns confirm. Roles: `admin`, or `deploy` for tenant secrets. |
| roles | `role_list`, `role_put` | The role tree. Host operators only. |
| tables | `table_put`, `table_drop`, `table_status` | Put builds a version beside the others; drop is refused while a route names it. |
| schedules | `schedule_put`, `schedule_drop` | Recurring timers on timer routes, applied to sun. |
| promotion | `promotion_diff`, `promotion_create`, `promotion_approve`, `promotion_reject`, `promotion_list` | The bulk review: one diff, approved whole or in part, by a person. |
| problems | `problem_list`, `problem_get`, `problem_ack`, `problem_resolve`, `problem_subscribe` | The problem log. Subscribe is SSE over HTTP and a resource subscription over MCP. |
| dead letters | `dead_list`, `dead_redrive`, `dead_discard` | `{ "from": "sun" | "barker" }`, proxied to `sun.control.*` and `barker.control.*`. |
| mail | `mail_send`, `mail_domain_check`, `template_put`, `template_test` | Operator mail and template tests through `barker.send`; templates are configuration. |
| logs | `log_tail`, `log_search`, `log_trace`, `log_count`, `log_verify` | Proxied to ranger with the caller's scope; every read is audited. |

### Promotions

```json
{ "tool": "promotion_diff", "args": { "tenant": "bobs_plumbing" } }
```

```json
{
	"from": { "env": "production", "seq": 8790 },
	"to": { "env": "beta", "seq": 9120 },
	"boxes": [ { "box": "accounts@sha256:77ab02...", "widens": ["writes settings"] } ],
	"routes": { "added": ["POST bobsplumbing.com/jobs"], "changed": ["PUT bobsplumbing.com/me/address"], "removed": [] },
	"widens": ["POST bobsplumbing.com/jobs can schedule timer:send_reminder, which merges into jobs."],
	"tables": { "added": ["jobs@3"], "dropped": [] },
	"policies": ["accounts: 95/5 split"],
	"releases": [ { "site": "sha256:aa01...", "pages": { "added": 2, "changed": 5, "removed": 0 } } ],
	"problems": [ { "problem": "iron.cas_conflict", "count": 14, "route": "PUT bobsplumbing.com/me/address" } ],
	"items": ["box:accounts@sha256:77ab02...", "route:POST bobsplumbing.com/jobs", "..."]
}
```

`promotion_approve { promotion, items? }` promotes everything, or the listed items. Rancher refuses a partial promotion that would leave production inconsistent (a route whose box version or table version isn't promoted with it) and says which items must come along. Approving requires a person's session; an agent token may create and read promotions, never approve them.

### Problems

`problem_list { tenant?, source?, box?, route?, table?, state?, since? }` answers problems newest first, paged by cursor, with counts and hints. `problem_get` adds every example. `problem_resolve` marks one resolved; it reopens on its own if it happens again.

## Rancher's outward calls

| to | methods | for |
| --- | --- | --- |
| pond | `pond.control.tenant.create`, `.suspend`, `.delete`, `pond.control.user.delete`, `pond.control.queries.put`, `pond.control.bucket.configure`, `.drop`, `pond.control.table.put`, `.drop`, `.rebuild`, `pond.control.events.put`, `.drop`, `pond.control.events.dead.*`, `pond.control.status`, `pond.ids.lease` | Tenants, users, named queries, tables, event cursors and dead events, and ids for tenants, promotions, and problems |
| permit | `permit.control.realm.create`, `.suspend`, `.delete`, `permit.control.user.delete`, `permit.control.operator.*`, `permit.control.audit.read` | Realms, operators, and audit |
| sun | `sun.control.schedule.put`, `.drop`, `sun.control.dead.*`, `sun.control.status` | Recurring schedules and the dead letter box |
| barker | `barker.send`, `barker.control.domain.check`, `barker.control.dead.*` | Operator mail and test sends, sender domains, dead letters |
| ranger | `ranger.search`, `ranger.tail`, `ranger.trace`, `ranger.count`, `ranger.verify` | Log reads for operators and agents |

Every apply is idempotent, keyed by the config seq that caused it, and retried by `rancher_apply` until it lands, so rancher can always re-apply its whole view after any failure.

## Codes and problems

Wire codes are [network's](../common/NETWORK.md#codes). Operator calls map them to HTTP: `invalid` 400, `refused` 403, `not_found` 404, `conflict` 409, `gone` 410, `overloaded` 429, `unavailable` 503. The problems rancher raises itself:

| problem | when |
| --- | --- |
| `route.ruleset_violation` | A route or policy would give a box a write outside its `writes`. |
| `route.unbound_param` | An object uses a `{param}` the permit requirement doesn't bind. |
| `route.overlap` | A route overlaps another without a traffic decision. |
| `route.grant_refused` | Fork refused a grant or named query against the table versions. |
| `route.event_loop` | An event route's service can write the bucket it listens to. |
| `route.asset_unbound` | A route binds an asset name its box versions don't declare, or a prefix uses an unbound `{param}`. |
| `route.egress_invalid` | An egress binding names an undeclared name, a non-https origin, an empty prefix, a missing spec, or a secret not sealed for barn. |
| `secret.missing` | A route or service names a secret nobody has put. |
| `table.drop_in_use` | A drop names a version a route still reads. |
| `ramp.guardrail` | A ramp breached its guardrails and was rolled back. |
| `apply.lagging` | A control apply to pond, permit, sun, or barker keeps failing. |
| `barn.stale` | A barn's config seq is far behind, or it stopped reporting. |
| `promotion.partial_refused` | A partial promotion would break production. |

## The schema, in part

```
namespace homestead.rancher;

enum Env : ubyte { beta, production }

table ConfigEvent {
  seq: uint64;
  kind: string;            // "route.put", "service.policy", ...
  env: Env;
  tenant: string;          // absent for platform-wide events
  object: [ubyte];         // a rancher_model FlatBuffer: Route, Service, Box, Domain, Table, ...
}

table Snapshot { seq: uint64; chunk: uint32; last: bool; objects: [ConfigEvent]; }

union ConfigMessage { Snapshot, ConfigEvent, Reset }

table Tool { name: string; summary: string; roles: [string]; changes: bool; input: [ubyte]; }   // input: JSON Schema
table Caller { tenant: string; subject: string; kind: string; grants: [string]; }
table ToolsList { caller: Caller; }
table ToolsListAnswer { tools: [Tool]; roles: [RoleNode]; }
table ToolCall { tool: string; args: [ubyte]; caller: Caller; }                 // args: JSON
table ToolAnswer { result: [ubyte]; seq: uint64; widens: [string]; undo: [ubyte]; problems: [ubyte]; }

table SealedSecret { name: string; for: [string]; fingerprint: string; sealed: [ubyte]; previous: [ubyte]; }

table ProblemBatch { seq: uint64; service: string; host: string; problems: [homestead.problem.Problem]; }
table HealthReport { seq: uint64; service: string; host: string; config_seq: uint64;
                     ready: [string]; window_ms: uint32; routes: [RouteCounts]; }
```

The model types (`Route`, `Service`, `Policy`, `Box`, `Domain`, `Table`) live in `rancher_model`, which barn and store link, so the bytes in a `ConfigEvent` mean the same thing to every reader.

## The invariants

- **Barns learn everything from `config.subscribe`.** A snapshot at a seq, then every event in order, each a whole object.
- **A pull comes before traffic.** No policy names a version until barns report it ready.
- **Every changing operator call has a dry run, an `if_seq`, a list of what it widens, and an undo.**
- **Production changes only by promotion,** and only a person approves one.
- **Reports are acknowledged.** Problems and health counts are pushes resent until rancher has them.
- **Rancher is the only caller of pond's, permit's, sun's, and barker's control APIs,** and every apply is idempotent.
- **The operator tools are never on the internet directly.** Porch, inside barn, serves them, and rancher checks the caller's roles again on every call.
- **Secrets cross the config log sealed,** only to the services allowed to open them; no tool answers a secret's value.
- **An event route is checked like any other,** plus no loops, and lives in one environment.
