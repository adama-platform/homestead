# Rancher
The control plane: boxes that exist, routes that decide what they may touch
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's manifest was a single document per box that said everything: routes, data strategies, schemas, roles, mail, timers, services, limits, and how it all fit together. Every new idea became a new section, every section had to be checked against every other, and deploying anything meant re-deriving the whole world. The manifest became the bottleneck.

Rancher replaces the manifest with a service mesh. **A box is a unit that just exists**: a module, identified by its hash, with an ABI version, its exports, and its write ruleset (the buckets it may ever write). **Routes are managed at the API level**, and a route is where every decision about a request lives: who may call it (a permit requirement), which objects it may touch (iron slots), which tables it may read (a grant), and which service answers it. A service's traffic policy then says which box versions answer: one version, a weighted A/B split, a shadow, or a ramp. Deploying a box and sending traffic to it are two different acts, which is how every service mesh already works.

Rancher is one machine. It holds configuration, drives pond's and permit's control APIs, signals barns, and collects problems. It never serves a tenant's request and never holds a tenant's data.

## What rancher holds

| object | example | what it says |
| --- | --- | --- |
| Tenant | bobs_plumbing | That a tenant exists, its state, and its quotas. Rancher creates it in pond and permit. |
| Domain | bobsplumbing.com -> bobs_plumbing | The domain registry: which tenant a domain provides, primary or redirect, and its verification. |
| Table | jobs@2, from bucket jobs | A versioned tailer definition, applied to pond in the tenants that use it. |
| Box | accounts@sha256:9f3c... | A module, its exports, and its write ruleset. Immutable once uploaded. |
| Service | accounts | A logical endpoint with a traffic policy over box versions. |
| Route | PUT bobsplumbing.com/me/address -> accounts.put_address | Permit requirement, objects, grant, hold, and the service and export that answer. |
| Barn | barn-7 | A registered barn: its address, its config seq, its health, and its failure counts per route. |
| Schedule | timer:nightly_digest every 1d at 03:00, all tenants | A recurring schedule on a timer route, applied to sun. |
| Problem | tailer.type_mismatch on users@1.zip | Something that went wrong, grouped, for operators and agents. |

Rancher keeps these in its own SQLite database, and every change to them is an event in rancher's config log.

## A box

```json
{
	"box": "accounts",
	"module": "sha256:9f3c41...",
	"abi": ["barn-abi@1", "iron@1"],
	"exports": ["put_address", "get_me", "render_me"],
	"writes": ["addresses", "profiles", "settings"],
	"fetch": { "geocoder": { "url": "https://geo.example", "effect": "none" } },
	"isolation": "reset",
	"limits": { "fuel": 100000000, "memory-max-bytes": 67108864, "timeout-ms": 1000 }
}
```

That is all a box declares. `writes` is its write ruleset: the buckets it may ever merge into or delete from, in any tenant. It doesn't name a tenant, a domain, a URL, a table, or a role. A box can be uploaded, checked, and sit unused forever.

Uploading a box checks that the module imports only the ABIs it lists and exports what it says. A box version never changes; a new module is a new version, and a new version that widens `writes` says so, so a person approving it sees the widening.

## A route

```json
{
	"route": "PUT bobsplumbing.com/me/address",
	"service": "accounts",
	"export": "put_address",
	"then": "render_me",
	"permit": { "any": ["signed_in"] },
	"objects": {
		"address": { "bucket": "addresses", "id": "{!user_id}", "access": "merge" },
		"profile": { "bucket": "profiles", "id": "{!user_id}", "access": "read" }
	},
	"grant": {
		"tables": { "users": { "version": 1, "columns": ["id", "name", "city"], "scope": "id = :!user_id" } },
		"limits": { "max-rows": 1, "max-ms": 20 }
	},
	"hold": { "tables": ["users@1"], "timeout-ms": 500 },
	"limits": { "rate": "10/s", "body-max-bytes": 4096 }
}
```

| field | decides | checked against |
| --- | --- | --- |
| permit | Who may call: a requirement over permit grants, like `owner` or `admin/project_{project_id}` | Permit's grant syntax; every `{param}` an object uses must be bound here. |
| objects | Exactly which keys the box may read, merge, or delete in this request | Every merge or delete slot's bucket must be in the `writes` of every box version the service may run. |
| grant | Which tables, versions, and columns the box's SQL may read, and which rows | Fork, against the table versions' columns. |
| hold, then | Whether the answer waits for tables to catch up, and what renders it | The held tables exist; `then` is an export. |

Routing is where security lives. Rancher checks a route when it is created, and again when a service's policy adds a box version, so a route can never give a box a write its ruleset doesn't allow, and a ramp can never send traffic to a version that would violate a route.

Routes cover every protocol barn speaks:

| protocol | route key |
| --- | --- |
| HTTP | verb, domain, typed path |
| SSE | `sse`, domain, typed path |
| MCP | `mcp`, domain, tool name |
| Email | `mail`, local part, domain (or every mail-enabled domain) |
| Timer | `timer`, name; fired by sun, with objects that may bind payload fields |

The conflict rules come from goober: the domain is the primary key, types count when comparing paths, every request resolves to exactly one route or none, and a route that overlaps an existing one is refused unless it comes with a traffic decision.

## Services and traffic

```json
{
	"service": "accounts",
	"policy": {
		"live": { "accounts@sha256:9f3c41...": 95, "accounts@sha256:77ab02...": 5 },
		"split-by": "user",
		"shadow": ["accounts@sha256:c0de99..."],
		"guardrails": { "error-rate": 0.02, "p99-ms": 300 }
	}
}
```

| policy field | meaning |
| --- | --- |
| live | The versions that really answer, with weights. |
| split-by | What the stable hash uses to keep a person on one version: user, ip, or request. |
| shadow | Versions that get a copy of each live request. Their merges are recorded and discarded, and their fetches run only when declared effect-free. |
| guardrails | Thresholds that stop a ramp and raise a problem. |

## Table versions

Because routes name the table versions their grants read, rancher always knows which versions are in use. Shipping a new shape is three steps, none of them a migration:

1. **Put `jobs@3`.** Pond builds it beside `jobs@2` from the documents, in every tenant that has the bucket.
2. **Move routes.** A route's grant moves to version 3, usually together with the box version that expects it.
3. **Drop `jobs@2`** once no route names it. Rancher refuses the drop while one does.

Tables that join several buckets by id work the same way, so a new materialized view across `profiles` and `addresses` is a table put and a route change.

## Signalling barns

Every change rancher accepts is an event with a seq in rancher's config log. Barns subscribe:

```
barn -> rancher   config.subscribe { from: 8812 }        (a wire stream)
rancher -> barn   { "seq": 8813, "kind": "box.pull", "box": "accounts@sha256:77ab02..." }
rancher -> barn   { "seq": 8814, "kind": "service.policy", "service": "accounts", "policy": { ... } }
rancher -> barn   { "seq": 8815, "kind": "route.put", "route": { ... } }
```

- **A cold barn** takes a snapshot at a seq, pulls the modules it names, and follows the log from that seq. It serves nothing, and takes no timer events, until it has.
- **A pull comes before traffic.** Rancher only publishes a policy that sends traffic to a version after barns report it ready.
- **Modules are content-addressed** and cached forever by hash.

## Driving pond, permit, and sun

Rancher is the only caller of pond's, permit's, and sun's control APIs. Creating a tenant creates its pond database and its permit realm; putting a table version applies it to pond; putting a recurring schedule on a timer route applies it to sun. Reading and redriving sun's dead letter box goes through rancher's API too. Every apply is idempotent and retried until it lands, so rancher can always re-apply its view.

## Problems

Rancher keeps the problem log: pond, barn, permit, sun, and rancher's own checks report problems to it, rancher groups them by fingerprint, and operators and agents read them through the API. That is described in [problems](../common/PROBLEMS.md).

## The operator API

Rancher's API is how people and agents run the platform: upload a box, put a route, try it, shadow it, ramp it, roll it back, put a table version, add a domain, read problems. It is offered over the internal protocol, and over HTTP and MCP through barn, with goober's engineering practices: every changing call has a dry run, says what it widens, takes an `if-seq`, and has an undo. Barn is its control plane proxy, so rancher is never on the internet.

## What rancher is not

- **Not in the request path.** If rancher is down, barns keep serving the last config they had.
- **Not a data store.** It holds configuration about tenants, never their data.
- **Not a manifest.** A box says almost nothing; routes say what each request may do, one route at a time.

## Open questions

1. **Does rancher's state belong in pond?** A reserved `_rancher` tenant would give rancher a WAL, a feed, and backups for free, and couple the control plane to the data plane's health.
2. **Automatic rollback?** Guardrails that stop a ramp and raise a problem, without reversing it, are the start.
3. **Who may schedule what?** A route's `schedules` list lets its box schedule timer routes; rancher should show the chain from the scheduling route to what the timer route writes as one widening.
4. **Who may change what?** Approvals for operators and agents: which changes, especially widenings of `writes`, objects, and grants, need a person.
5. **Routes per tenant, or per domain pattern?** A product serving many tenants wants one route on `*` with the tenant from the domain; a single business wants routes on its own domain.

## The invariants

- **A box just exists.** Immutable, content-addressed, and declaring only its exports and the buckets it may write.
- **Routing owns security.** Who may call, which objects, which tables: all on the route, all checked when the route or a policy changes.
- **No route gives a box a write outside its ruleset.** For every version the service may run.
- **Table versions are dropped only when no route reads them.**
- **Every request resolves to exactly one route, or none.**
- **A shadow never changes the world.**
- **Barns learn everything from the config log, in order.** A pull always comes before the traffic that needs it.
- **Rancher is the only caller of pond's, permit's, and sun's control APIs, and is off the request path.**
