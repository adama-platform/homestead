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
| SSE | A long-lived GET on a typed path | Events. Barn subscribes to pond's feed for the route's objects and tables, and runs the box's export for each change it should push. |
| MCP | A tool call on a domain's `/mcp` | A tool result. The tool list is the routes rancher published for that domain, filtered by the caller's grants. |
| Email | A message to an address on a mail-enabled domain, handed to barn by a mail receiver | Accept or reject, with objects resolved from the sender and recipient. |
| Timer | A job fired by sun, naming a tenant, a timer route, and a payload | An acknowledgment to sun: done, or failed with a reason. |

## Learning what to serve

Barn learns everything from rancher. At start, it takes a snapshot of the live configuration at a seq, pulls the modules it names (by hash, cached on local disk forever), compiles them, and then follows rancher's config log from that seq.

**Barn doesn't serve until it has heard from rancher.** A starting barn fails its load balancer health check, and doesn't subscribe to sun, until it holds a complete snapshot, has compiled every live box version, and has leased an id range from pond. If rancher goes away later, barn keeps serving the last configuration it had, and reports how stale it is.

## Timer events

Barn subscribes to sun, offering credit for how many timer events it can run at once. A fired job arrives with its tenant, timer route, payload, and attempt; barn evaluates it exactly like a request on that route (objects may bind payload fields), with the job's id as the pond write's idempotency key. When the box finishes, barn acknowledges `done`, or `failed` with the reason, and sun retries or dead-letters the job (see [sun](../sun/IDEA.md)). A box on a route that lists `schedules` may ask barn to schedule or cancel jobs by key; barn sends them to sun after the request's pond write lands.

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
	"write": { "seq": 4472, "reruns": 0 }, "hold": { "users@1": "applied in 38ms" },
	"outcome": 200, "ms": { "route": 0.1, "permit": 0.2, "load": 1.4, "box": 3.1, "write": 2.2, "hold": 38 },
	"fuel": 412000
}
```

The record holds keys, seqs, names, and timings, never documents, bodies, or parameters, so it can be kept and shown to agents. It answers the questions that are hardest to answer later: which route matched and why, which version ran and why, what the box was allowed to touch, and where the time went. Records are written locally and shipped to a log sink; the destination is an open question.

## Failures per route

Barn counts, per tenant, route, and box version: requests, outcomes by class (refused by permit, refused by routing rules, trapped, out of fuel or memory, timed out, write conflict, hold timed out), and latency and fuel histograms. Timer routes add done, failed, and expired. The counts are reported to rancher every few seconds, where they drive health, ramp guardrails, and problems.

## Isolation is configurable

Boxes are stateless, so anything kept between requests is an optimization, and the isolation level trades safety against speed per box (an operator can raise it per route or per tenant).

| level | an instance is | can one request see another's leftovers? |
| --- | --- | --- |
| fresh | Created for one request and thrown away | Never. The default. |
| reset | Pooled, and its memory restored to a snapshot before each reuse | Never, as long as the reset is complete. Much cheaper than fresh. |
| tenant | Kept warm and reused for requests of the same tenant and box version | Yes, within one tenant. |
| shared | Kept warm and reused for any request to that box version | Yes, across tenants. Only for boxes holding no tenant data. |

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

Everything that goes wrong in barn is a problem record for rancher's problem log, not only a log line: a trap, a fuel or memory limit, a merge to a slot the route made read-only, a compare-and-set that kept conflicting, a query fork refused, a hold that timed out. Each names the box version, route, slot or table, and a request id, so an agent can see the failure and the code that caused it together (see [problems](../common/PROBLEMS.md)).

## What barn is not

- **Not storage.** Not even a cache that matters for correctness.
- **Not a control plane.** It follows rancher's log, proxies rancher's API, and decides nothing about what is deployed.
- **Not a timer.** Sun decides when; barn only runs what sun fires.
- **Not an identity system.** Permit decides who a caller is; barn checks the route's requirement against what permit said.
- **Not a TLS terminator for the internet, or a container host.**

## Open questions

1. **The component model, or a hand-written ABI?** Iron is small enough to write by hand in C or WAT; WIT would give bindings in many languages for free.
2. **Objects a route can't know in advance.** A box that writes a variable number of line items needs `new*` slots, or a slot over child ids listed in a parent object. Both widen the iron path and should wait for a real box.
3. **How many reruns?** A hot object can make compare-and-set conflict often. A small bound, with the conflict reported as a problem, is the start.
4. **Where do routing logs go?** Local files shipped to object storage are cheap; a log service agents can query is more useful. A sample of full records plus every failure is a middle ground.
5. **How does SSE fan out?** A thousand clients watching one bucket shouldn't run a box a thousand times per change.
6. **Does `reset` need engine support?** How well wasmtime restores linear memory from a snapshot decides whether `reset` is the practical default.

## The invariants

- **Barn is a pure evaluator.** Rancher controls the routes; barn routes, enforces, and builds the ABI.
- **Barn holds no durable state.** Any barn can answer any request.
- **Barn serves nothing until it has heard from rancher.** No snapshot, no traffic and no timer events.
- **Every timer event is acknowledged, or its lease expires.** Sun decides what happens next.
- **Every routing decision is logged.** Keys, versions, reasons, and timings; never data.
- **A box writes only through iron.** It is handed objects, and for each it merges or doesn't.
- **Buckets at deploy, objects at runtime.** The box's ruleset bounds the buckets; the route resolves the exact keys.
- **A box never chooses a key.** Ids come from the caller, a bound parameter, a name, or a fresh mint.
- **A request's writes are one compare-and-set write.** All of them, or none if the box fails.
- **Routing owns the grants.** Barn sends the route's grant with every query, and fork enforces it.
- **Every hold is bounded.** A timeout answers and is reported.
- **Warm state is only an optimization.** Every box answers the same from a fresh instance.
- **A bad box costs only its own routes.** Errors, latency, and problem records, never a breach.
