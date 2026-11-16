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
| Domain | bobsplumbing.com -> bobs_plumbing | The domain registry: which tenant a domain provides, as primary, redirect, beta, or ops, and its verification. |
| Table | jobs@2, from bucket jobs | A versioned tailer definition, applied to pond in the tenants that use it. |
| Box | accounts@sha256:9f3c... | A module, its exports, and its rulesets: the buckets it may write, the asset names and egress names it may use. Immutable once uploaded. |
| Spec | strava@sha256:41d0... | An OpenAPI document an egress binding names, stored by hash like a module and fetched by barns. |
| Secret | tenant/bobs_plumbing/hevy_api_key | A sealed secret, which rancher stores and publishes but can never open (see [Secrets](#secrets)). |
| Role | content, data, deploy, ... | The operator role tree, which decides what each `/mcp/{role}` endpoint offers. |
| Service | accounts | A logical endpoint with a traffic policy over box versions. |
| Route | PUT bobsplumbing.com/me/address -> accounts.put_address | Permit requirement, objects, grant, hold, and the service and export that answer. |
| Barn | barn-7 | A registered barn: its address, its config seq, its health, and its failure counts per route. |
| Schedule | timer:nightly_digest every 1d at 03:00 | A system timer: a recurring schedule on a timer route, applied to sun. |
| Promotion | beta seq 9120 -> production | A reviewed bundle of every change between production and beta, promoted at once. |
| Problem | tailer.type_mismatch on users@1.zip | Something that went wrong, grouped, for operators and agents. |

Rancher keeps these in its own SQLite database, not in pond, so the control plane never depends on the data plane's health. Every change to them is an event in rancher's config log.

## Beta and production

Every tenant has two environments, and rancher holds a full configuration for each: routes, services and their policies, table versions in use, and schedules.

| environment | served on | changes |
| --- | --- | --- |
| beta | `beta.example.com` | Land immediately. An operator or agent puts a route, uploads a box, moves a policy, and beta serves it on the next config seq, with no approval. |
| production | `example.com` | Only by promotion. |

**Promotion is a bulk review.** Rancher shows the whole diff between production and beta as one document: boxes added and their `writes`, routes added, changed, and removed, every widening of objects and grants, table versions added and dropped, policy changes, schedules, and the problems beta raised along the way. A person reviews it by its shape and promotes it in one act, or promotes part of it. Promotion is a config change like any other, so it is in the log, it is reversible, and production keeps answering while barns apply it.

Both environments are the same tenant: the same realm in permit, the same database in pond. Beta and production differ in which box versions and routes answer, not in whose data they read, so beta is where changes meet real data before they meet everyone. Table versions that beta needs are built beside production's, as any table version is.

Agents work in beta. The person owns production.

## A box

```json
{
	"box": "accounts",
	"module": "sha256:9f3c41...",
	"abi": ["barn-abi@1", "iron@1"],
	"exports": ["put_address", "get_me", "render_me"],
	"topology": "request-response",
	"writes": ["addresses", "profiles", "settings"],
	"assets": ["photos"],
	"egress": ["geocoder"],
	"isolation": "reset",
	"limits": { "fuel": 100000000, "memory-max-bytes": 67108864, "timeout-ms": 1000 }
}
```

That is all a box declares. `topology` is request-response or streaming (see [barn](../barn/IDEA.md#the-box-contract)). `writes` is its write ruleset: the buckets it may ever merge into or delete from, in any tenant. `assets` names the S3 asset areas it may ever touch, and `egress` names the third-party services it may ever call; neither says where they are. It doesn't name a tenant, a domain, a URL, a prefix, a table, or a role. A box can be uploaded, checked, and sit unused forever.

The three rulesets work the same way: **names at deploy, places at runtime.** The box says "photos" and "geocoder"; each route says which S3 prefix "photos" is for this request, and which origin, URI prefixes, and credential "geocoder" is. A box can't reach an asset area or a service it didn't declare, and a route can't hand it one.

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
	"queries": { "me": "SELECT id, name, city FROM users WHERE id = :!user_id" },
	"hold": { "tables": ["users@1"], "timeout-ms": 500 },
	"assets": {
		"photos": { "prefix": "photos/{!user_id}/", "access": ["read", "write", "list", "delete"],
		            "max-bytes": 10485760, "types": ["image/*"] }
	},
	"egress": {
		"geocoder": { "origin": "https://geo.example", "prefixes": ["/v1/geocode"], "methods": ["GET"],
		              "credential": { "kind": "api-key", "header": "x-api-key", "secret": "tenant/bobs_plumbing/geocoder" },
		              "spec": "sha256:41d0...", "rate": "10/s" }
	},
	"limits": { "rate": "10/s", "body-max-bytes": 4096 }
}
```

| field | decides | checked against |
| --- | --- | --- |
| permit | Who may call: a requirement over permit grants, like `owner` or `admin/project_{project_id}` | Permit's grant syntax; every `{param}` an object uses must be bound here. |
| objects | Exactly which keys the box may read, merge, or delete in this request | Every merge or delete slot's bucket must be in the `writes` of every box version the service may run. |
| grant | Which tables, versions, and columns the box's SQL may read, and which rows | Fork, against the table versions' columns. |
| queries | Named queries the box runs by name, checked once at deploy and prepared once by pond | Fork, against the grant and the table versions' columns. |
| hold, then | Whether the answer waits for tables to catch up, and what renders it | The held tables exist; `then` is an export. |
| assets | Which S3 prefix each asset name means for this request, and what the box may do there (read, write, list, delete), with size and type caps | Every name is in the `assets` of every box version the service may run; every `{param}` in a prefix is bound by the permit requirement, as for objects; barn always adds the tenant's own prefix in front. |
| egress | Which origin, URI prefixes, and methods each egress name may reach, which credential barn attaches, which OpenAPI spec describes it, and its rate | Every name is in the `egress` of every box version; the origin is https; prefixes are normalized and non-empty; the secret exists and is sealed for barn; the spec hash was uploaded. |

Routing is where security lives. Rancher checks a route when it is created, and again when a service's policy adds a box version, so a route can never give a box a write its ruleset doesn't allow, and a ramp can never send traffic to a version that would violate a route.

Routes cover every protocol barn speaks:

| protocol | route key |
| --- | --- |
| HTTP | verb, domain, typed path |
| SSE | `sse`, domain, typed path |
| MCP | `mcp`, domain, tool name |
| Email | `mail`, local part, domain (or every mail-enabled domain) |
| Timer | `timer`, name; fired by sun, with objects that may bind payload fields |
| Event | `event`, name; fired by pond's event tailer on a bucket's changes, with objects that may bind `{event.id}` and `{event.user}` |

A route can also answer without a box, for bytes in S3 (silo, inside barn):

| form | on | does |
| --- | --- | --- |
| `"serve": { "asset": "media", "key": "{*path}" }` | GET | Streams the object under the asset's prefix; no box wakes. |
| `"upload": { "asset": "photos", "key": "{uuid}.{ext}" }` | PUT, POST | Streams the body to S3 under the prefix, with the route's size and type caps; an optional `then` export sees what was stored. |
| `"attachments": { "asset": "photos", "max-files": 10 }` | a box route | Barn stores each multipart part first, and the box gets their keys, sizes, and types instead of bytes. |

A box with `write` on an asset may also presign an upload for a browser, inside its prefix. Garbage collection is the box's job: a box granted `list` and `delete` sweeps what its own documents no longer name.

The conflict rules come from goober: the domain is the primary key, types count when comparing paths, every request resolves to exactly one route or none, and a route that overlaps an existing one is refused unless it comes with a traffic decision.

## Event routes

Pond is the event bus (see [pond](../pond/IDEA.md#event-routes)): an event route is a tailer over a bucket's changes whose sink is a box. Rancher owns the route like any other, and applies only its `on` to pond with `pond.control.events.put`.

```json
{
	"route": "event order_placed",
	"service": "fulfilment",
	"export": "on_order",
	"on": { "bucket": "orders", "ops": ["merge"] },
	"objects": {
		"order": { "bucket": "orders", "id": "{event.id}", "access": "read" },
		"stock": { "bucket": "reservations", "id": "{event.id}", "access": "merge" }
	}
}
```

Rancher checks three more things for an event route:

- **No loops.** If any box version the service may run can write the bucket the route listens to, the route (or the policy that adds that version) is refused with `route.event_loop`. Chains across buckets are allowed, and the promotion diff shows each chain as one widening: "a merge into `orders` runs `on_order`, which merges into `reservations`."
- **One environment.** Both environments see every write to the tenant's data, so an event route is live in exactly one. A new event route in beta runs as a shadow: barns run it, record its merges, and discard them. Promotion moves it to production, where it takes over the cursor pond already keeps, so nothing is missed or run twice.
- **Objects bind only the event.** `{event.id}` and `{event.user}` are the only sources besides reserved values and `new`; an event route has no caller and no permit requirement.

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
| guardrails | Thresholds that, when breached, roll a ramp back automatically and raise a problem. |

**Rollback is automatic.** When a version on a ramp breaches its service's guardrails, rancher returns its weight to the previous version, publishes that policy, and raises a problem naming the version, the guardrail, and the numbers. Rolling forward again is a person's or agent's decision.

## Table versions

Because routes name the table versions their grants read, rancher always knows which versions are in use. Shipping a new shape is three steps, none of them a migration:

1. **Put `jobs@3`.** Pond builds it beside `jobs@2` from the documents, in every tenant that has the bucket.
2. **Move routes.** A route's grant moves to version 3, usually together with the box version that expects it.
3. **Drop `jobs@2`** once no route names it. Rancher refuses the drop while one does.

Tables that join several buckets by id work the same way, so a new materialized view across `profiles` and `addresses` is a table put and a route change.

## Routes belong to a tenant

Every route belongs to exactly one tenant, on that tenant's own domain. There are no routes on `*` shared by many tenants: each tenant is a distinct product with a distinct domain, so a box serving two tenants is simply named by routes in both. This keeps a route's decisions (who, which objects, which tables) specific to the product it serves.

## Timer routes and schedules

A timer is a temporal route, as in goober: a route keyed `timer`, name, with its own objects and grant. It can be started two ways:

| form | set by | runs for |
| --- | --- | --- |
| System timer | A schedule in rancher, like "run `nightly_digest` every day at 03:00" | As long as the job needs, within its lease; for sweeps, digests, and maintenance. |
| Scheduled job | A box, on a route that lists the timer route in `schedules`, through pond's outbox | Seconds, like any request. |

A route's `schedules` list is a widening, and rancher shows the chain as one: "PUT /jobs can schedule `send_reminder`, which merges into `jobs`," so a person approving it sees everything the scheduling route can cause.

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

Rancher is the only caller of pond's, permit's, and sun's control APIs. Creating a tenant creates its pond database and its permit realm; putting a table version applies it to pond; putting a route registers its named queries, checked by fork, with pond; putting an event route applies its cursor to pond; putting a recurring schedule on a timer route applies it to sun. Reading and redriving sun's dead letter box goes through rancher's API too. Every apply is idempotent and retried until it lands, so rancher can always re-apply its view.

## Problems

Rancher keeps the problem log: pond, barn, permit, sun, and rancher's own checks report problems to it, rancher groups them by fingerprint, and operators and agents read them through the API. That is described in [problems](../common/PROBLEMS.md).

## Specs

An egress binding names an OpenAPI document by hash. Rancher stores specs like modules: uploaded once, immutable, content-addressed, published to barns in the config log, and fetched by barns with `rancher.spec.fetch`. Barn's [hound](../hound/IDEA.md) uses a spec to validate a box's request against the operation it matches, and to explain a refusal by operation id. A spec never widens anything: the origin, prefixes, and methods on the route are the permission.

## Secrets

Rancher keeps every secret a service or route needs (a tenant's Hevy API key, Strava's OAuth app, the platform's SES credentials), and it can't read any of them. Secrets follow goober's vault design, carried in the [cellar](../cellar/IDEA.md) library:

- **Write-only.** `secret_put` seals the value to the deployment's public wrap key, with cellar's seal function, the moment it arrives. Rancher stores the ciphertext and a fingerprint, never the value, and no tool answers it back.
- **Named and scoped.** `system/ses`, `tenant/bobs_plumbing/hevy_api_key`. A secret lists the services that may open it (`for: ["barn"]`), and the config log carries it only to those subscribers.
- **Opened at use.** Barn, permit, or barker open a secret with the root key cellar holds, in memory, for the call that needs it.
- **Per user, in pond.** Tokens a person grants (Strava's OAuth refresh token) are sealed the same way, but they change at runtime, so barn keeps them in pond's reserved `_credentials` bucket, not in configuration.

## Roles and the operator tools

Every operator tool belongs to the service that implements it. Rancher's (routes, boxes, promotions, problems, secrets) and store's (pages, components, releases) are listed and called over the wire with `rancher.tools.list`/`rancher.tools.call` and `store.tools.list`/`store.tools.call`. Rancher also proxies sun's, barker's, and ranger's tools through its own, so dead letters and logs keep rancher's audit and scoping. People and agents reach them through [porch](../porch/IDEA.md), a library inside barn that serves them over MCP and HTTP on `ops.` domains. Rancher is never on the internet.

A tool says which roles it serves:

```json
{ "name": "page_patch", "summary": "Replace exact text in a page", "roles": ["content"], "changes": true, "input": { } }
```

Rancher holds the role tree for the deployment. It starts as `admin`, which contains `deploy`, `content`, `data`, `observe`, `mail`, and `identity`. A role is held as a grant, `ops/{role}`, so an agent's token carries exactly the roles its parent gave it, and never more than its subject has. `/mcp/content` offers the content tools the caller's grants allow; `/mcp` offers every tool they allow. The path focuses an agent; the grant secures it. Tenant `mcp` routes may carry `"roles": [...]` too, so `/mcp/{role}` filters a product's own tools the same way.

## The operator API

Rancher's tools are how people and agents run the platform: upload a box, put a route, try it, shadow it, ramp it, roll it back, put a table version, add a domain, read problems, set secrets, and review and promote beta to production. They are offered over the internal protocol and reached through porch, with goober's engineering practices: every changing call has a dry run, says what it widens, takes an `if-seq`, and has an undo.

## What rancher is not

- **Not in the request path.** If rancher is down, barns keep serving the last config they had.
- **Not a data store.** It holds configuration about tenants, never their data.
- **Not a manifest.** A box says almost nothing; routes say what each request may do, one route at a time.

## The invariants

- **A box just exists.** Immutable, content-addressed, and declaring only its exports and the buckets it may write.
- **Routing owns security.** Who may call, which objects, which tables: all on the route, all checked when the route or a policy changes.
- **No route gives a box a write outside its ruleset.** For every version the service may run.
- **Names at deploy, places at runtime.** A box declares the buckets, asset names, and egress names it may use; a route binds each to keys, an S3 prefix, or an origin and URI prefixes, and every `{param}` is bound by the permit requirement.
- **An event route can't feed itself,** and runs in exactly one environment; promotion hands it the cursor.
- **Secrets are write-only.** Rancher seals and stores them, and can never open one.
- **The path focuses, the grant secures.** `/mcp/{role}` narrows the tools offered; `ops/{role}` grants decide what a caller may call.
- **Table versions are dropped only when no route reads them.**
- **Every request resolves to exactly one route, or none.**
- **A shadow never changes the world.**
- **Changes land in beta; production moves only by promotion.** Every promotion is a reviewed bulk diff.
- **A breached guardrail rolls back automatically.**
- **A route belongs to one tenant.** No route spans tenants.
- **Barns learn everything from the config log, in order.** A pull always comes before the traffic that needs it.
- **Rancher is the only caller of pond's, permit's, and sun's control APIs, and is off the request path.** Its tools reach people and agents only through porch, inside barn.
