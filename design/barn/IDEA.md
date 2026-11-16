# Barn
A stateless host for WebAssembly, with an iron write path and a cache in front of pond
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's host did everything: TLS checks, routing, identity, mail, timers, the bus, approvals, migrations, schemas, generated code, and storage, all behind one manifest. Its boundary tried to give a box typed access to every table it could touch, generated from schemas, which is why the boundary, the checker, and the generator all had to grow together. The parts that held up were the box itself and the entrance to it: an isolated WASM module with no POSIX inheritance, reached over HTTP, SSE, MCP, and email, with routes a person can read and conflicts an operator decides.

Barn is that entrance, and it stays simple in spirit: **barn is a pure evaluator.** Rancher controls every route, and barn only evaluates them. Barn has three jobs:

1. **Route to a box.** Find the one route a request (or a timer event) matches, and pick a box version by the service's traffic policy.
2. **Enforce the route's rules at runtime.** The permit requirement, the objects, the grant, the limits, and the hold.
3. **Build the ABI for the box.** Resolve and load the objects, lay out the request, run the box, and turn what it staged into one pond write.

It receives a request, asks permit who the caller is, finds the route, picks a box version, runs the box, and answers. The one boundary that has to be ironclad is the write path, and barn makes it small: **a box is handed the objects its route allows, and for each one it either merges or it doesn't.** Reads are SQL under the route's grant, run by pond. Around those three jobs, barn caches reads (using pond's feed to keep the cache honest), runs timer events from sun and event routes from pond, stores and serves a tenant's files in S3 under the route's prefixes (silo), speaks to third-party services through [hound](../hound/IDEA.md), serves the operator tools over MCP through [porch](../porch/IDEA.md), unwraps secrets at the moment of use with [cellar](../cellar/IDEA.md), logs every routing decision to ranger, counts failures per route, and is the proxy for permit's, barker's, and store's pages.

Barn holds no durable state. Any barn can answer any request, so capacity is more barns behind the load balancer, and losing one costs only the requests it was holding.

## Where barn sits

| barn talks to | for | how |
| --- | --- | --- |
| Caddy | Requests in, with TLS already done | Plain HTTP. |
| permit | Who a caller is, and the grants they hold | Verifies tokens locally; follows permit's revocation feed. |
| rancher | Routes, services, domains, box versions to pull or drop | Subscribes to rancher's config log; fetches modules by hash. |
| permit, barker, store | Sign-in and consent pages, unsubscribe links, upload tickets | Proxied from the internet as `http.serve`; barn never interprets them. |
| rancher, store | The operator tools, served as MCP and HTTP by porch inside barn | `rancher.tools.*` and `store.tools.*`; rancher proxies sun's, barker's, and ranger's tools. |
| pond | Reads, writes, queries, names, id ranges, the feed, and events | `pond_client`, scoped to the request's tenant; `pond.events.subscribe` for event routes. |
| S3 (object storage) | A tenant's assets: uploads, attachments, served files | Barn's own S3 client, under the tenant's prefix, with credentials from cellar. |
| third-party APIs | A box's outbound calls | Hound, a library in barn, to the origin and prefixes the route binds. |
| sun | Timer events to run, and one-shot jobs boxes schedule | Subscribes with credit; acknowledges each event. Schedules arrive through pond's outbox. |
| permit, sun, barker | Effects committed with a write: grants, schedules, mail | Drains pond's outbox under a lease (see [pond](../pond/IDEA.md#the-outbox)). |
| ranger | Routing logs and service logs | Streams records over the internal protocol (see [ranger](../ranger/IDEA.md)). |

Everything but Caddy speaks the internal protocol: multiplexed binary frames over TCP, with optional mutual TLS (see [common](../common.md#the-internal-protocol)).

## A request's life

```
  client -> Caddy (TLS) -> barn
                             |
                     1. resolve the domain to a tenant
                     2. resolve the caller with permit (!user_id, grants)
                     3. match the route; check the route's permit requirement
                     4. pick a box version by the service's policy
                     5. resolve the route's objects to keys and load them; resolve its
                        asset prefixes and egress names; stream any attachments to S3
                     6. run the box (iron): it reads objects, queries, calls silo and
                        hound, and merges or not
                     7. send its merges and effects to pond as one write (compare-and-set)
                     8. hold if the route says so, or run the box's "then" export
                     9. encode the response
                             |
  client <- Caddy <- barn <--+
```

A shadow version gets a copy of the same request; its answer is compared, and its merges are recorded and dropped.

## The box contract

A box is a WebAssembly component, described in WIT, built in the reactor model: it is instantiated once, exports functions barn calls (one per route export), and returns from each call rather than running a `main` to completion. The reactor model is what lets barn keep instances warm, reset them, and call a second export (`then`) on the same box, and WIT gives every language bindings for free.

| interface | what it is |
| --- | --- |
| `barn-abi@1` | Everything that isn't a write: the request, the response, queries, logging, time, and randomness. |
| `iron@1` | The write path. |
| `silo@1` | Files in S3 under the prefixes the route bound (see [Assets](#assets-silo)). |
| `hound@1` | Calls to third-party services the route bound (see [Egress](#egress-hound)). |

A box also declares its **topology**, which decides how barn calls it:

| topology | the box | used for |
| --- | --- | --- |
| request-response | Gets one call per request or event and returns one answer | HTTP, MCP, email, timer, and event routes. The default. |
| streaming | Gets a call per change or message on a long-lived connection, and emits events | SSE, and later bi-directional protocols. |

## The iron write path

Iron is less an API than a contract: the interface between barn and the box stays stable across barn versions, and it carries harder restrictions around data writes than anything else a box can do. It is versioned like `iron@1`, and it is deliberately small, because the write path is only a mutation: a set of objects, each of which a box may merge into, or leave alone.

### Two layers of constraint

| layer | where it is set | what it says | when it is checked |
| --- | --- | --- | --- |
| The box's write ruleset | Uploaded with the box, in rancher | The buckets this module may ever write, like `["addresses", "profiles", "jobs"]` | At deploy: a route can't give a box an object in a bucket outside its ruleset. |
| The route's objects | The route, in rancher | Exactly which keys this request may touch, resolved from the request | At runtime, per request: a box can only write the keys barn resolved. |

So a box is constrained at the bucket layer when it is deployed, and per object when it runs. A box that was granted `addresses` can't write a stranger's address, because the route resolved `addresses/{!user_id}` and that is the only address slot it has.

### Objects

A route names its objects as slots, each with a bucket, an id source, and an access:

```json
"objects": {
	"address": { "bucket": "addresses", "id": "{!user_id}", "access": "merge" },
	"profile": { "bucket": "profiles", "id": "{!user_id}", "access": "read" },
	"customer": { "bucket": "customers", "id": "name:email/{email}", "access": "merge" },
	"job": { "bucket": "jobs", "id": "new", "access": "merge" }
}
```

| id source | resolves to | allowed when |
| --- | --- | --- |
| `{!user_id}` and other reserved values | The caller, from permit | Always; the caller can't choose it. |
| `{param}` | A typed path or query parameter | The route's permit requirement binds that parameter, like `admin/project_{project_id}`, so the caller holds a grant for that very object. |
| `new` | A fresh id from barn's leased range | Always; written with `if-seq: 0`, so it can only create. |
| `name:kind/{param}` | The id pond's name service binds to that string | As for `{param}`, and the name is created on first use only if the slot merges. |

| access | the box may |
| --- | --- |
| read | Read the object. |
| merge | Read it, and merge into it or not. |
| delete | Read it, merge, or delete it. |

### The calls

| import | does |
| --- | --- |
| `iron_object_read(slot, ptr, len)` | Copies the object's current document (JSON, or empty if none) into the box's memory. |
| `iron_object_id(slot, ptr, len)` | The object's id, as a string, so a box can return a new job's id. |
| `iron_merge(slot, ptr, len)` | Stages a merge patch for the object. Merges to one slot compose in order. |
| `iron_delete(slot)` | Stages a delete, if the slot allows it. |
| `iron_hold(table_ptr, len)` | Asks barn not to answer until that table has applied this request's write. |

Everything else a box can do (read the request, respond, query, log, time, randomness) is the ordinary `barn-abi@1`, and none of it writes to pond. Files are `silo@1` and outbound calls are `hound@1`, each bounded by its own route grant.

### What barn does with the merges

When the box returns successfully, barn turns every staged merge and delete into one pond write, with `if-seq` set to the seq each object had when barn loaded it. That gives every box compare-and-set for free:

- **A conflict is refused outright.** If another request changed an object in between, pond refuses the write, and barn doesn't rerun the box: it answers the request with a conflict (409 over HTTP), and counts it per route. Retrying is the box's or the client's decision, which keeps back pressure where the contention is instead of hiding it inside barn.
- **A failure writes nothing.** A trap, a timeout, or a non-zero return discards every staged merge.
- **Retries are idempotent.** The pond write carries the request id as its idempotency key.
- **Effects ride along.** Grants (a route's `gives`), schedules (a route's `schedules`), and mail the box asked for are staged as outbox effects and land in the same pond write, so they happen if and only if the merges do. After the write, barns drain the outbox and deliver them (see [pond](../pond/IDEA.md#the-outbox)).

## Waiting for propagation

Tailers run asynchronously, so a box that wrote may answer before its tables reflect the write. A route or a box can hold back the response instead:

| how | does |
| --- | --- |
| `"hold": { "tables": ["users@1"], "timeout-ms": 500 }` on a route | Barn sends the write with a hold, and answers only once pond says those tables applied it. |
| `iron_hold(table)` in the box | The same, decided by the box at runtime. |
| A `then` export on the route | After the write is durable and held tables have caught up, barn runs a second export of the same box, which can query fresh tables to render the answer. |

```json
"route": "POST bobsplumbing.com/jobs",
"export": "create_job",
"then": "render_job_list",
"hold": { "tables": ["jobs@2"], "timeout-ms": 500 }
```

A hold that times out still answers, with the timeout noted to the box's `then` export and reported as a problem, so a slow tailer shows up as something to fix rather than a hung request.

## Reads

A box reads its objects through iron, and everything else through SQL under the grant its route carries:

```json
"grant": {
	"tables": { "jobs": { "version": 2, "columns": ["id", "status", "customer_id", "finished_at"],
	                      "scope": "customer_id = :!user_id" } },
	"limits": { "max-rows": 200, "max-ms": 50 }
}
```

Routing owns the grant, so the same box can read `jobs@1` on one route and `jobs@2` on another, and moving a route to a new table version is a routing change. Barn sends the grant to pond with the query, and fork enforces it.

## Protocols

| protocol | in | out |
| --- | --- | --- |
| HTTP | Verb, domain, typed path, query, headers, body | Status, headers, body |
| SSE | A long-lived GET on a typed path, answered by a streaming box | Events. Barn subscribes to pond's feed for the route's objects (by bucket and id) and tables, and runs the box's export for each change it should push (see [SSE fan-out](#sse-fan-out)). |
| MCP | A tool call on a domain's `/mcp` or `/mcp/{role}` | A tool result. On a tenant domain, the tools are its `mcp` routes; on `ops.`, the operator tools from porch. Either way filtered by the path's role and the caller's grants (see [MCP and porch](#mcp-and-porch)). |
| Email | A message to an address on a mail-enabled domain, handed to barn by a mail receiver | Accept or reject, with objects resolved from the sender and recipient. |
| Timer | A job fired by sun, naming a tenant, a timer route, and a payload | An acknowledgment to sun: done, or failed with a reason. |
| Event | A change pond leased to barn: a seq, bucket, id, op, and user on an event route's bucket | An acknowledgment to pond: done, failed, or skipped (see [Event routes](#event-routes)). |
| Assets | A GET on a `serve` route, a body on an `upload` route, or multipart parts on a route with `attachments` | Bytes from S3 with ETag and Range, or the stored objects' keys, handed to the box (see [Assets](#assets-silo)). |

## SSE fan-out

A thousand clients watching one job shouldn't run a box a thousand times per change. So barn groups a route's subscribers by their distinct subscription parameters (the resolved object keys and bound path parameters), runs the box's export **once per change per group**, and fans the event out to every client in the group.

That is only safe when the event doesn't depend on who is watching. A route says so:

```json
"route": "sse bobsplumbing.com/jobs/{job_id}",
"per-viewer": false
```

| `per-viewer` | barn runs the export |
| --- | --- |
| false | Once per change per distinct set of subscription parameters, and shares the result across viewers. |
| true (the default) | Once per change per connection, since the event may depend on the caller's identity or grants. |

A route that declares `per-viewer: false` promises its box doesn't read the caller; barn enforces it by giving the shared run no caller at all.

## Learning what to serve

Barn learns everything from rancher. At start, it takes a snapshot of the live configuration at a seq, pulls the modules it names (by hash, cached on local disk forever), compiles them, and then follows rancher's config log from that seq.

**Barn doesn't serve until it has heard from rancher.** A starting barn fails its load balancer health check, and doesn't subscribe to sun, until it holds a complete snapshot, has compiled every live box version, and has leased an id range from pond. If rancher goes away later, barn keeps serving the last configuration it had, and reports how stale it is.

## Timer events

Barn subscribes to sun, offering credit for how many timer events it can run at once. A fired job arrives with its tenant, timer route, payload, and attempt; barn evaluates it exactly like a request on that route (objects may bind payload fields), with the job's id as the pond write's idempotency key. When the box finishes, barn acknowledges `done`, or `failed` with the reason, and sun retries or dead-letters the job (see [sun](../sun/IDEA.md)). A box on a route that lists `schedules` may ask barn to schedule or cancel jobs by key; they are committed as outbox effects with the request's pond write, and delivered to sun from there.

## Event routes

There is no separate bus. **Pond is the event source**, because every change already has a seq in a tenant's `_changes`, and an event route is a tailer whose sink is a box instead of a table. A route keyed `event` names a bucket and the ops it cares about:

```json
{
	"route": "event order_placed",
	"service": "fulfilment",
	"export": "on_order",
	"on": { "bucket": "orders", "ops": ["merge", "delete"] },
	"objects": {
		"order": { "bucket": "orders", "id": "{event.id}", "access": "read" },
		"shipment": { "bucket": "shipments", "id": "{event.id}", "access": "merge" }
	}
}
```

- **Pond keeps the cursor.** Each event route has a cursor per tenant in pond, like a tailer. Barn opens `pond.events.subscribe`, one stream across tenants, with credit equal to the events it can run at once, exactly as it takes timer events from sun. Pond leases each event to one barn for 30 seconds.
- **Keys, not contents.** An event is `{seq, bucket, id, op, user, at}`. Objects bind `{event.id}` and `{event.user}`, and the box reads what it needs through its slots, under its route's rules, so an event never carries a field the route wasn't given.
- **Acknowledged, or leased again.** Barn evaluates the event like a request and answers `pond.events.ack {lease, outcome, reason}` with `done`, `failed` (pond retries with backoff, then dead-letters and raises a problem), or `skipped` (the route is gone or the tenant suspended).
- **At least once, written once.** The box's write carries the idempotency key `event:{route}:{seq}`, so a redelivery can't apply twice.
- **Ordered per object.** Events for the same route and id arrive in seq order; different ids run in parallel.
- **Beta shadows.** An event route that exists only in beta runs as a shadow: its merges are recorded and dropped until it is promoted, since beta and production share one tenant's data.
- **SSE is unchanged.** Watching an object live stays on `pond.feed.subscribe`; event routes are for work, not for screens.

A route whose box writes the bucket it listens to can loop. Rancher refuses an event route whose service may write its own `on` bucket, unless the route says `"self-writes": true` and the box is trusted to stop.

## Assets (silo)

A tenant's files (uploads, attachments, photos, exports) live in S3, and barn is the only door to them. Silo isn't a service; it is a function of barn, with the same split as iron: **buckets of rights at deploy, prefixes at runtime.**

| layer | where | says |
| --- | --- | --- |
| The box's assets | Uploaded with the box | The asset names it may ever touch, like `"assets": ["photos"]`. |
| The route's assets | The route | The prefix each name resolves to for this request, and what the box may do there. |

```json
"assets": {
	"photos": {
		"prefix": "photos/{!user_id}/",
		"access": ["read", "write", "list", "delete"],
		"max-bytes": 10485760,
		"types": ["image/*"]
	}
}
```

- **The tenant is always the first prefix.** Barn writes `t/{tenant id}/photos/4294967100/...` in the deployment's bucket. A box names only an asset and a key under it, never the S3 bucket, the tenant, or the full key.
- **Keys are checked after normalizing.** No `..`, no empty segments, no control bytes; a key that doesn't land under the resolved prefix is refused. This check is the iron of silo, and it is R0.
- **Types come from the bytes.** Barn sniffs the content type of every stored object and refuses one outside `types`, whatever the client claimed.

There are several ways in, on purpose:

| form | on the route | what happens |
| --- | --- | --- |
| `serve` | `"serve": {"asset": "photos", "key": "{path}"}` on a GET | No box. Barn streams the object from S3, with ETag, Range, and the route's cache headers. |
| `upload` | `"upload": {"asset": "photos", "key": "{uuid}"}` on a PUT or POST | Barn streams the body to S3 under the prefix, then runs the optional `then` export with the object's `{key, size, type, sha256}`. |
| `attachments` | `"attachments": {"asset": "photos", "max-files": 5}` on a box route | Barn streams each multipart file part to S3 first, then runs the box with the other fields as the body and `[{field, key, size, type, sha256}]` as attachments. If the box fails, barn deletes them, best effort. |
| presign | `silo@1.presign` from a box | A short-lived URL for a direct PUT or GET to S3, within the box's prefix and limits, for large files a browser sends itself. |
| the ABI | `silo@1` from a box | `head`, ranged `get`, small `put`, `list` with a cursor, and `delete`, each only if the route's `access` allows it. |

**Garbage is the box's job.** Barn doesn't track which objects a document references. A box that wants orphans gone is granted `list` and `delete` and runs a sweep on a timer route, which also keeps the policy (how long an unreferenced upload lives, what "referenced" means) in product code, where it belongs. A user's deletion is the same: the product's deletion route lists and deletes that user's prefix.

The S3 endpoint and bucket are barn's configuration; the credentials are a sealed secret unwrapped by cellar.

## Egress (hound)

A box calling Strava or Hevy builds its own HTTP request: method, path, query, headers, and body. That is full control, and it is deliberate. What a box doesn't control is **where** the request goes or **what credential** it carries. Barn binds both on the route, and [hound](../hound/IDEA.md), a library inside barn, sends the request:

| layer | where | says |
| --- | --- | --- |
| The box's egress | Uploaded with the box | The service names it may ever call, like `"egress": ["strava"]`. Replaces goober's `fetch`. |
| The route's egress | The route | For each name: the origin, the path prefixes, the methods, the credential, an optional OpenAPI spec, and a rate. |

```json
"egress": {
	"strava": {
		"origin": "https://www.strava.com",
		"prefixes": ["/api/v3/"],
		"methods": ["GET"],
		"credential": { "kind": "oauth2-user", "secret": "tenant/strava_app" },
		"spec": "sha256:5e1a...",
		"rate": "100/15m"
	}
}
```

- **The origin is fixed by the name.** A box sends to `strava`, not to a URL, so it can't reach a host the route didn't give it.
- **The path must normalize under a prefix,** and the method must be listed. Redirects are followed only when they stay within the origin and a prefix.
- **The credential is hound's.** The box can't set `Authorization`, `Cookie`, `Host`, or the credential's header; hound adds them from cellar. A box never sees a key or a token.
- **Per-user tokens live in pond.** OAuth tokens a person grants (their Strava account) are sealed documents in the reserved bucket `_credentials`, keyed by user. When hound refreshes one, barn writes the new token as a system write. The connect flow lives at the reserved path `/_hound/connect/{name}` and its callback.
- **OpenAPI when there is a spec.** With `spec`, a box may call by operation id (`hound@1.call`), and hound builds and checks the request from the spec; without it, `send` is plain HTTP.
- **Caps and logs.** Request and response sizes, a time limit inside the box's deadline, the route's rate per tenant, and one `egress` line to ranger per exchange. A shadow run only calls names marked effect-free.

## MCP and porch

Barn speaks MCP in two places, and both can be narrowed by role so agents can form a hierarchy:

| where | tools | filtered by |
| --- | --- | --- |
| `/mcp` and `/mcp/{role}` on a tenant's domain | The tenant's `mcp` routes | The route's `roles` tags, then the caller's grants |
| `/mcp` and `/mcp/{role}` on `ops.` | The operator tools, from [porch](../porch/IDEA.md), a library in barn | The tool's roles, then the caller's `ops/{role}` grants |
| `/_ops/api/{tool}` on `ops.` | The same operator tools, as plain HTTP JSON | The same |

Porch aggregates `rancher.tools.list` and `store.tools.list` (rancher carries sun's, barker's, and ranger's), and forwards each call to the service that owns the tool, with the caller's subject and grants. **The path focuses; the grant decides.** `/mcp/content` lists only content tools, so an agent connected there has a small, sharp catalog; but a tool runs only if the caller holds `ops/content` (or `ops/admin`, which holds every role). A lead agent can ask permit for a narrower token for a sub-agent, and an agent never holds more than its subject, so the hierarchy is enforced by permit, not by convention.

## Secrets (cellar)

Barn needs secrets (S3 credentials, hound's API keys and OAuth client secrets, the reply-token key), and it holds none of them at rest. [Cellar](../cellar/IDEA.md), goober's vault design as a library, is linked into barn:

- **Sealed in the log.** Secrets arrive on rancher's config log sealed; rancher stores them but can't read them.
- **Unwrapped at the moment of use.** Barn unwraps a secret with the host's vault key when a call needs it, keeps the plaintext only in memory and only as long as the cache allows, and never gives it to a box.
- **Every use is countable.** Unwraps are counted per secret and logged to ranger's `audit` stream, never with the value.

## Routing logs

Because barn is where every routing decision is made, barn writes one structured record per request or timer event that explains it:

```json
{
	"request": "9f3c...", "at": 1762870000123, "protocol": "http",
	"domain": "bobsplumbing.com", "tenant": "bobs_plumbing",
	"route": "PUT bobsplumbing.com/me/address", "config-seq": 8815,
	"service": "accounts", "version": "accounts@sha256:77ab02...", "picked-by": "split 5% by user",
	"shadow": ["accounts@sha256:c0de99..."],
	"permit": { "subject": "4294967100", "kind": "user", "satisfied-by": "_signed_in" },
	"objects": { "address": "addresses/4294967100 @4471 merge", "profile": "profiles/4294967100 @4402 read" },
	"queries": [{ "tables": ["users@1"], "rows": 1, "ms": 2 }],
	"write": { "seq": 4472, "conflict": false }, "hold": { "users@1": "applied in 38ms" },
	"outcome": 200, "ms": { "route": 0.1, "permit": 0.2, "load": 1.4, "box": 3.1, "write": 2.2, "hold": 38 },
	"fuel": 412000
}
```

The record holds keys, seqs, names, and timings, never documents, bodies, or parameters, so it can be kept and shown to agents. It answers the questions that are hardest to answer later: which route matched and why, which version ran and why, what the box was allowed to touch, and where the time went. Records stream to [ranger](../ranger/IDEA.md), the log service every homestead service feeds, where operators, the operating tooling, and agents search and tail them in real time.

## Failures per route

Barn counts, per tenant, route, and box version: requests, outcomes by class (refused by permit, refused by routing rules, trapped, out of fuel or memory, timed out, write conflict, hold timed out), and latency and fuel histograms. Timer routes add done, failed, and expired. The counts are reported to rancher every few seconds, where they drive health, ramp guardrails, and problems.

## Isolation is configurable

Boxes are stateless, so anything kept between requests is an optimization, and the isolation level trades safety against speed per box (an operator can raise it per route or per tenant).

| level | an instance is | can one request see another's leftovers? |
| --- | --- | --- |
| fresh | Created for one request and thrown away | Never. The default. |
| reset | Pooled, and its memory restored to a snapshot before each reuse | Never, as long as the reset is complete. Much cheaper than fresh, and where the best performance comes from. |
| tenant | Kept warm and reused for requests of the same tenant and box version | Yes, within one tenant. |
| shared | Kept warm and reused for any request to that box version | Yes, across tenants. Only for boxes holding no tenant data. |

`reset` is a first-class level, available from the start, and measured: barn tracks the cost of each restore and verifies that a restored instance's memory matches its snapshot, so its speed is known and its isolation is checked, not assumed. The reactor model makes it natural: the snapshot is taken after instantiation and initialization, before the first call.

Whatever the level, correctness never depends on warm state, and the iron path doesn't change: a box can still only write the objects this request resolved.

| limit | caps | when hit |
| --- | --- | --- |
| fuel | Instructions executed | The instance traps; barn answers 503. |
| memory | Linear memory size | The instance traps; barn answers 503. |
| deadline | Wall time, including host calls | The instance is killed; barn answers 504. |
| body size | Bytes in and out | 413 before the box wakes, or 500 after. |

## Caching

- **Objects by key.** Loading a route's objects is served from the cache when the cached document is current; pond's feed drops a key when it changes.
- **Names forever.** A name's id never changes, so resolved names are cached without invalidation.
- **Query answers.** Keyed by tenant, grant, fork's canonical SQL, and parameters, and dropped when a table the answer read reports progress past it.
- **Grants from permit.** Cached per caller and dropped by permit's revocation feed.
- **A reset drops everything for that tenant.**

A barn with empty caches answers correctly, only slower. Stickiness by tenant at the load balancer warms caches, and nothing depends on it.

## Problems

Everything that goes wrong in barn is a problem record for rancher's problem log, not only a log line: a trap, a fuel or memory limit, a merge to a slot the route made read-only, a compare-and-set conflict on a hot object, a query fork refused, a hold that timed out, a silo key outside its prefix, an egress call outside its origin or prefixes, an event route that keeps failing. Each names the box version, route, slot or table, and a request id, so an agent can see the failure and the code that caused it together (see [problems](../common/PROBLEMS.md)).

## What barn is not

- **Not storage.** Not even a cache that matters for correctness. Assets live in S3 and documents in pond; barn holds the door, not the contents.
- **Not a bus.** Events come from pond's tailers; barn only runs the routes that listen to them.
- **Not a control plane.** It follows rancher's log, proxies rancher's API, and decides nothing about what is deployed.
- **Not a timer.** Sun decides when; barn only runs what sun fires.
- **Not an identity system.** Permit decides who a caller is; barn checks the route's requirement against what permit said.
- **Not a TLS terminator for the internet, or a container host.**

## Later

- **Objects a route can't know in advance.** A box that writes a variable number of line items needs `new*` slots, or a slot over child ids listed in a parent object. Both widen the iron path, and wait for a real box.

## The invariants

- **Barn is a pure evaluator.** Rancher controls the routes; barn routes, enforces, and builds the ABI.
- **Barn holds no durable state.** Any barn can answer any request.
- **Barn serves nothing until it has heard from rancher.** No snapshot, no traffic and no timer events.
- **Every timer event is acknowledged, or its lease expires.** Sun decides what happens next.
- **Every routing decision is logged, to ranger.** Keys, versions, reasons, and timings; never data.
- **A box writes only through iron.** It is handed objects, and for each it merges or doesn't.
- **Buckets at deploy, objects at runtime.** The box's ruleset bounds the buckets; the route resolves the exact keys.
- **A box never chooses a key.** Ids come from the caller, a bound parameter, a name, or a fresh mint.
- **A request's writes are one compare-and-set write.** All of them, or none if the box fails; a conflict is refused, never silently rerun.
- **Effects ride with the write.** Grants, schedules, and mail land in the same pond write as the merges, or not at all.
- **A box is a WIT component in the reactor model**, with a declared topology: request-response or streaming.
- **Routing owns the grants.** Barn sends the route's grant with every query, and fork enforces it.
- **Every hold is bounded.** A timeout answers and is reported.
- **Warm state is only an optimization.** Every box answers the same from a fresh instance.
- **A bad box costs only its own routes.** Errors, latency, and problem records, never a breach.
- **Events come from pond.** An event route is a tailer whose sink is a box: leased, acknowledged, written once by seq.
- **Assets at deploy, prefixes at runtime.** A box names assets and keys; barn adds the tenant and checks every key under the route's prefix.
- **Egress at deploy, origins and prefixes at runtime.** A box builds its own request, but never chooses the host or sees the credential.
- **The path focuses, the grant decides.** `/mcp/{role}` narrows what an agent sees; `ops/{role}` grants decide what it may run.
- **No secret at rest in barn, and none in a box.** Cellar unwraps at the moment of use.
