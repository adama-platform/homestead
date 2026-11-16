# Barn's API
The edge it answers, the ABI it hands a box, and what it calls on everyone else
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Barn is the only service with no wire listener: it dials out to everyone and nobody dials it (see [network](../common/NETWORK.md)). So barn's API isn't a FlatBuffers schema. It is three surfaces:

1. **The edge.** What Caddy and the mail receiver send barn, and what barn answers: HTTP, SSE, MCP, LMTP, plus timer events that arrive on a stream barn opened to sun.
2. **The box ABI.** What a box imports and exports: `barn-abi@1` for everything that isn't a write, and `iron@1` for the write path. This is the contract that must stay stable across barn versions, and it is where a box's power ends.
3. **The footprint.** What barn calls on pond, rancher, permit, sun, barker, store, and ranger. These are other services' APIs; listing them here says exactly how much of each one barn depends on.

| surface | from | transport | section |
| --- | --- | --- | --- |
| HTTP routes | Caddy | HTTP/1.1, port 8080 | [HTTP](#http) |
| Reserved prefixes | Caddy | HTTP/1.1, proxied as `http.serve` | [Reserved paths](#reserved-paths) |
| SSE | Caddy | HTTP/1.1, long-lived GET | [SSE](#sse) |
| MCP | Caddy | HTTP, `/mcp` on a domain | [MCP](#mcp) |
| Mail | Stalwart | LMTP, port 2424 (Q12.3) | [Mail](#mail) |
| Timer events | sun | wire stream barn opened | [Timers](#timers) |
| Health | the load balancer | HTTP, `/_barn/health` | [Health](#health) |
| `barn-abi@1` | a box | WIT imports | [The box ABI](#the-box-abi) |
| `iron@1` | a box | WIT imports | [iron@1](#iron1) |

## The edge

### HTTP

Every request goes through the same pipeline (see [barn](IDEA.md#a-requests-life)). The domain picks the tenant, the route picks the box, and the outcome is mapped to a status barn owns. A box chooses its own status only when it ran and returned successfully.

| outcome | status | body | problem code |
| --- | --- | --- | --- |
| The box answered | the box's | the box's | |
| Unknown domain | 421 | empty | |
| No route matched | 404 | barn's | |
| Not signed in, and the route requires it | 401 (browsers: 303 to `/_permit/sign-in`) | barn's | |
| Signed in, requirement not met | 403 | barn's | `permit.requirement_unmet` (counted, not a problem per request) |
| Body over the route's `body-max-bytes` | 413, before the box wakes | barn's | |
| Rate limit | 429 with `Retry-After` | barn's | |
| Tenant suspended, on a writing route | 423 | barn's | |
| Compare-and-set conflict on the write | 409 | barn's | `iron.conflict` |
| Trap, non-zero return, fuel or memory limit | 503 | barn's | `box.trap`, `box.fuel`, `box.memory` |
| Deadline | 504 | barn's | `box.deadline` |
| Response over the cap, after the box ran | 500 | barn's | `box.response_too_large` |
| Pond, permit, or a dependency unavailable | 503 with `Retry-After` | barn's | `barn.dependency` |
| Barn is not ready (no snapshot yet) | never: the health check keeps traffic away | | |

A hold that times out is not an error: the write landed, the response goes, and `then` sees `hold.timed-out = true`.

**Headers barn owns.** Barn sets `content-length`, `connection`, `transfer-encoding`, `date`, `etag` for stitched and static pages, `set-cookie` for permit's session cookie only, and `x-homestead-request` (the request id, which `log_trace` takes). A box may set any other response header; one it tries to set from that list, or starting with `x-homestead-`, is refused at the call.

**Headers barn strips.** Incoming `x-homestead-*`, `forwarded` and `x-forwarded-*` other than the ones Caddy sets, and hop-by-hop headers. The box sees the client address Caddy reported, as a field, not a header it could be fooled by.

### Reserved paths

On every tenant domain, these prefixes never reach a box. A route that claims one is refused by rancher.

| prefix | goes to | how |
| --- | --- | --- |
| `/_permit/` | permit | `permit.http.serve` |
| `/_unsubscribe/`, `/_barker/` | barker | `barker.http.serve` |
| `/_ops/` on `ops.` | rancher | `rancher.http.serve` |
| `/_ops/site/` on `ops.` | store | `store.http.serve` |
| `/mcp` on `ops.` | rancher's operator tools, and store's site tools | `rancher.http.serve` (rancher fans site tools to store) |
| `/mcp` on any other domain | the tenant's MCP routes | barn itself, see [MCP](#mcp) |
| `/_barn/health` | barn | local |

Proxying is opaque: barn resolves the tenant from the domain, strips what it strips for any request, caps the body (1 MiB; store uploads go by ticket), and forwards `HttpRequest` as NETWORK.md defines it. It never reads the body, adds credentials, or caches the answer unless the answer says `cache-control: public`.

### SSE

A GET on an `sse` route opens a stream. Barn evaluates the route once at open (permit, objects, grant), subscribes to pond's feed for the route's object keys and held tables, and calls the box's streaming export for each change.

```
GET /jobs/4294967312  (accept: text/event-stream)

event: change
id: 4472
data: {"status":"done"}
```

- **The event id is the pond seq.** A reconnect with `Last-Event-ID` resumes from it; a `reset` from pond becomes one `event: reset` and the box's export runs once with the current state.
- **Fan-out** follows `per-viewer` (see [barn](IDEA.md#sse-fan-out)).
- **Streams close** when the caller's grants change (permit's feed), when the route changes, or after the route's `max-stream-ms` (default 1 hour), so a revoked caller doesn't keep listening.

### MCP

`POST /mcp` on a tenant domain speaks MCP over streamable HTTP. `tools/list` is the tenant's `mcp` routes that the caller's grants satisfy, with each route's input schema. `tools/call` evaluates the route exactly like HTTP; the box's response body is the tool result. An agent authenticates with an OAuth token from permit (`kind: agent`); without one, `tools/list` answers only routes requiring nothing, and the `WWW-Authenticate` header points at `/_permit/`.

### Mail

The receiver delivers each message over LMTP to port 2424 (the lean in Q12.3). Barn answers per recipient:

| case | answer |
| --- | --- |
| A `mail` route on that address and domain accepted it | 250 |
| A `reply+<token>@` address whose token verifies (`barker_reply`) | the route the token names, with its claims as context; 250 or the route's refusal |
| The bounce address | `barker.bounce.ingest`; 250 |
| No route, a forged token, a box refusal | 550 with a short reason |
| Barn not ready, or a dependency down | 451, so the sender retries |

Accepted mail from a sender is noted with `barker.thread.note` (hashes only), so replies get the thread allowance.

### Timers

Barn opens `sun.fire.subscribe` with a window equal to the timer events it can run at once, and evaluates each fired job as a request on its `timer` route, with the job's id as the pond write's idempotency key and payload fields available to object slots.

| box result | barn sends sun |
| --- | --- |
| Returned 0, write landed (or no write) | `sun.fire.ack {job, done}` |
| Trap, limit, non-zero, conflict, dependency down | `sun.fire.ack {job, failed, reason}` with the problem code |
| Still running a system job | `sun.fire.heartbeat {job}` every lease/3 |

Barn returns one message of credit after each ack.

### Health

`GET /_barn/health` answers 200 only when barn holds a complete rancher snapshot, has compiled every live box version, holds an id range, and its pond connection is up. Otherwise 503 with a JSON body naming what's missing and how stale the config is. Caddy and the load balancer use it; nothing else does.

## The box ABI

A box is a WebAssembly component in the reactor model. Its world imports `barn-abi@1` and, if it writes, `iron@1`, and exports the functions its routes name. A box may import nothing else; rancher refuses the upload otherwise.

```wit
package homestead:barn@1.0.0;

interface request {
  record header { name: string, value: string }
  record request {
    id: string,                    // the request id
    protocol: string,              // "http", "sse", "mcp", "mail", "timer"
    method: string, path: string, query: string,
    params: list<tuple<string, string>>,   // typed path and query params the route declared
    headers: list<header>,
    caller: option<string>,        // !user_id as a string, absent for a visitor or a shared SSE run
    grants: list<string>,          // the caller's grants that the route's requirement names
    now-ms: s64,                   // fixed once per request
  }
  get: func() -> request;
  body: func(offset: u64, len: u32) -> list<u8>;   // read the body in pieces
}

interface response {
  set-status: func(status: u16) -> result<_, string>;
  set-header: func(name: string, value: string) -> result<_, string>;  // owned headers refused
  write: func(bytes: list<u8>) -> result<_, string>;                   // appends; capped
  event: func(name: string, data: list<u8>) -> result<_, string>;      // streaming topology only
}

interface query {
  record answer { rows: list<u8>, seq: u64, cached: bool }   // rows: JSON array, ids as strings
  named: func(name: string, params: list<u8>) -> result<answer, string>;
  sql: func(text: string, params: list<u8>) -> result<answer, string>;
}

interface fetch {
  record reply { status: u16, headers: list<tuple<string, string>>, body: list<u8> }
  call: func(service: string, method: string, path: string,
             headers: list<tuple<string, string>>, body: list<u8>) -> result<reply, string>;
}

interface util {
  log: func(level: u8, text: string);            // a box log line (Q11.1)
  random: func(len: u32) -> list<u8>;            // seeded per request; the seed is logged
}

world box {
  import request; import response; import query; import fetch; import util;
}
```

| import | rule |
| --- | --- |
| `request.get` | The caller's raw token, cookies, and `x-homestead-*` are never present. |
| `query.named` | Runs one of the route's named queries; params are JSON, bound, never spliced. Reserved params come from barn. |
| `query.sql` | Ad-hoc SQL under the route's grant, checked by fork in pond on every call. Refusals return the fork error text and become a problem. |
| `fetch.call` | Only a service declared in the box's `fetch`, to its declared origin. On a shadow run, only effect-free ones. Each call is an `egress` line. |
| `util.random` | Deterministic per request, so a request can be replayed. |
| time | `request.now-ms`, fixed per request. There is no clock import. |

### iron@1

```wit
interface iron {
  type slot = string;              // a name from the route's objects

  read: func(s: slot) -> result<option<list<u8>>, iron-error>;   // JSON, none if no document
  id: func(s: slot) -> result<string, iron-error>;
  merge: func(s: slot, patch: list<u8>) -> result<_, iron-error>;  // RFC 7386, composed in order
  delete: func(s: slot) -> result<_, iron-error>;
  hold: func(table: string) -> result<_, iron-error>;              // e.g. "users@1"

  // effects, committed in the same pond write, or not at all
  grant: func(subject: string, role: string) -> result<_, iron-error>;    // role from route `gives`
  revoke: func(subject: string, role: string) -> result<_, iron-error>;
  schedule: func(route: string, key: string, at-ms: s64, payload: list<u8>) -> result<_, iron-error>;
  cancel: func(route: string, key: string) -> result<_, iron-error>;
  mail: func(service: string, to: string, template: string, vars: list<u8>) -> result<_, iron-error>;

  enum iron-error {
    unknown-slot, read-only, not-deletable, patch-invalid, patch-too-large,
    table-not-granted, role-not-given, route-not-schedulable, mail-refused, too-many-effects,
  }
}

world writer { include box; import iron; }
```

- **A box can only name slots.** No bucket, no id, no tenant appears in any call.
- **Errors are returned, not trapped.** A box that ignores `read-only` still writes nothing to that slot, and barn records `iron.slot_read_only` as a problem.
- **Effects are iron's**, not barn-abi's, because they share the write's fate. `grant` substitutes the route's bound parameters into its `gives` and refuses anything else; `mail` checks the template renders, the recipient isn't suppressed, and the per-address limit.
- **Caps:** at most 64 slots per route, 1 MiB of composed patch per slot, 4 MiB per write, 32 effects per write.

### Exports

| export | signature | when |
| --- | --- | --- |
| a route's `export` | `func() -> s32` | Every request on the route. 0 commits the staged write; anything else discards it and answers what the box set, or 500. |
| a route's `then` | `func(hold-timed-out: bool) -> s32` | After the write is durable and held tables applied (or timed out). It may read and query; any iron write call returns `read-only`. |
| a streaming export | `func(seq: u64, change: list<u8>) -> s32` | Once per pushed change (per group, or per viewer). |
| `init` (optional) | `func() -> s32` | Once after instantiation, before the reset snapshot is taken. |

### Limits

| limit | default | from | when hit |
| --- | --- | --- | --- |
| fuel | 100,000,000 | box, route may lower | trap; 503 |
| memory | 64 MiB | box | trap; 503 |
| deadline | 1,000 ms, host calls included | box, route may lower | killed; 504 |
| request body | route `body-max-bytes` | route | 413 before the box wakes |
| response | 8 MiB | barn | 500 |
| queries | 32 per request | barn | `query` returns an error |
| fetches | 8 per request | barn | `fetch` returns an error |

## The footprint

What barn calls on each service. Every one goes through that service's `_client` interface, and every one has a fake in barn's tests.

| service | barn uses | for |
| --- | --- | --- |
| pond | `pond.get`, `pond.write`, `pond.query`, `pond.names`, `pond.wait`, `pond.ids.lease`, `pond.feed.subscribe` | Objects, the write, SQL, name slots, held answers, minting, cache invalidation and SSE |
| rancher | `rancher.config.subscribe`, `rancher.box.fetch`, `rancher.health.report`, `rancher.problems.report`, `rancher.http.serve` | Routes and policies, modules, counts, problems, the operator proxy |
| permit | `permit.keys.subscribe`, `permit.grants.get`, `permit.http.serve` | Verifying tokens, live grants on a cache miss, sign-in pages |
| sun | `sun.fire.subscribe`, `sun.fire.ack`, `sun.fire.heartbeat` | Timer events |
| barker | `barker.suppression.subscribe`, `barker.thread.note`, `barker.bounce.ingest`, `barker.http.serve` | Refusing suppressed mail at the call, threads, bounces, unsubscribe |
| store | `store.release.fetch`, `store.http.serve` | Compiled releases, site tools |
| ranger | `ranger.ingest` | Routing, egress, and agents lines, from the spool |

Under Q12.1's lean, barn does not drain the outbox and calls no `permit.effects.apply` or `sun.jobs.*`; if the answer goes the other way, those join this table.

## The invariants

- **Nothing calls barn but Caddy, the mail receiver, and the load balancer's health check.** Everything else arrives on streams barn opened.
- **Reserved prefixes never reach a box,** and proxying never interprets what it carries.
- **Barn owns the statuses that aren't the box's.** A conflict is 409, a trap 503, a deadline 504, and each is counted per route.
- **A box imports only `barn-abi@1` and `iron@1`.** No WASI, no clock, no sockets.
- **A box names slots, never keys.** No bucket, id, or tenant crosses the ABI into barn.
- **Effects are iron calls,** and share the write's fate.
- **ABI errors are returned, not trapped,** and every refusal is a problem.
- **Barn's footprint on every other service is the table above,** through `_client` interfaces with fakes.
