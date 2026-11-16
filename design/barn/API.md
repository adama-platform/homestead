# Barn's API
The edge it answers, the ABI it hands a box, and what it calls on everyone else
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Barn is the only service with no wire listener: it dials out to everyone and nobody dials it (see [network](../common/NETWORK.md)). So barn's API isn't a FlatBuffers schema. It is three surfaces:

1. **The edge.** What Caddy and the mail receiver send barn, and what barn answers: HTTP, assets, SSE, MCP, LMTP, plus timer events and pond events that arrive on streams barn opened.
2. **The box ABI.** What a box imports and exports: `barn-abi@1` for everything that isn't a write, `iron@1` for the write path, `silo@1` for files in S3, and `hound@1` for third-party calls. This is the contract that must stay stable across barn versions, and it is where a box's power ends.
3. **The footprint.** What barn calls on pond, rancher, permit, sun, barker, store, and ranger. These are other services' APIs; listing them here says exactly how much of each one barn depends on.

| surface | from | transport | section |
| --- | --- | --- | --- |
| HTTP routes | Caddy | HTTP/1.1, port 8080 | [HTTP](#http) |
| Assets | Caddy | HTTP/1.1: `serve`, `upload`, and `attachments` routes | [Assets](#assets) |
| Reserved prefixes | Caddy | HTTP/1.1, proxied as `http.serve`, or local | [Reserved paths](#reserved-paths) |
| SSE | Caddy | HTTP/1.1, long-lived GET | [SSE](#sse) |
| MCP | Caddy | HTTP, `/mcp` and `/mcp/{role}` on a domain | [MCP](#mcp) |
| Operator tools | Caddy | HTTP JSON, `/_ops/api/{tool}` on `ops.` | [MCP](#mcp) |
| Mail | Stalwart | LMTP, port 2424 (Q12.3) | [Mail](#mail) |
| Timer events | sun | wire stream barn opened | [Timers](#timers) |
| Pond events | pond | wire stream barn opened | [Events](#events) |
| Health | the load balancer | HTTP, `/_barn/health` | [Health](#health) |
| `barn-abi@1` | a box | WIT imports | [The box ABI](#the-box-abi) |
| `iron@1` | a box | WIT imports | [iron@1](#iron1) |
| `silo@1` | a box | WIT imports | [silo@1](#silo1) |
| `hound@1` | a box | WIT imports | [hound@1](#hound1) |

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

### Assets

Three route forms reach S3 (see [barn](IDEA.md#assets-silo)). Each resolves its asset's prefix from the route, with the tenant always first.

**`serve`** is a GET that maps a path to a key under a prefix, with no box:

```
GET /media/photos/4294967100/a1b2.jpg
-> 200, etag: "9f3c...", accept-ranges: bytes, cache-control: (the route's), content-type: (sniffed at upload)
```

**`upload`** streams the body to S3, then optionally runs `then` with the stored object:

```
PUT /me/avatar   content-type: image/jpeg   (body: the bytes)
-> the `then` export's answer, or 201 with {"key": "4294967100/7e1d.jpg", "size": 182311, "type": "image/jpeg", "sha256": "..."}
```

**`attachments`** accepts `multipart/form-data` on a box route. File parts go to S3 first; the other fields become the body the box reads; the box gets the stored parts through `silo@1.attachments`.

| outcome | status | problem code |
| --- | --- | --- |
| Served | 200, 206 for a Range, 304 for a matching `If-None-Match` | |
| Stored by `upload` with no `then` | 201 | |
| No object at that key | 404 | |
| Key escapes the prefix after normalizing | 400, before S3 is touched | `silo.key_outside_prefix` |
| Over `max-bytes`, or more files than `max-files` | 413, the upload is aborted and its parts deleted | |
| Sniffed type not in `types` | 415, the object is deleted | `silo.type_refused` |
| Route grants no write | 405 | |
| S3 unavailable | 503 with `Retry-After` | `barn.dependency` |
| The box fails after attachments were stored | the box's failure status; barn deletes the parts, best effort | `silo.orphaned` if the delete fails |

**Presigned URLs.** A box may call `silo@1.presign` to hand a browser a URL for a direct PUT or GET to S3: within its prefix, with the route's `max-bytes` and `types` written into the signature's conditions, valid for at most 15 minutes. A presigned upload isn't sniffed by barn, so a box that cares checks with `head` and deletes what it doesn't want.

### Reserved paths

On every tenant domain, these prefixes never reach a box. A route that claims one is refused by rancher.

| prefix | goes to | how |
| --- | --- | --- |
| `/_permit/` | permit | `permit.http.serve` |
| `/_unsubscribe/`, `/_barker/` | barker | `barker.http.serve` |
| `/_ops/site/upload/` on `ops.` | store | `store.http.serve`, for upload tickets only |
| `/_ops/api/{tool}` on `ops.` | the operator tools | porch, in barn, forwarding to `rancher.tools.call` or `store.tools.call` |
| `/mcp`, `/mcp/{role}` on `ops.` | the operator tools | porch, in barn, see [MCP](#mcp) |
| `/mcp`, `/mcp/{role}` on any other domain | the tenant's MCP routes | barn itself, see [MCP](#mcp) |
| `/_hound/connect/{name}` | a person connecting a third-party account | barn and hound, see [Connecting an account](#connecting-an-account) |
| `/_barn/health` | barn | local |
| `/_barn/ask?domain=` | barn | local; Caddy's on-demand TLS check |

Proxying is opaque: barn resolves the tenant from the domain, strips what it strips for any request, caps the body (1 MiB; store uploads go by ticket), and forwards `HttpRequest` as NETWORK.md defines it. It never reads the body, adds credentials, or caches the answer unless the answer says `cache-control: public`. Rancher has no `http.serve`: its operator surface is tools, which porch serves.

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

`POST /mcp` speaks MCP over streamable HTTP, and `POST /mcp/{role}` is the same endpoint narrowed to one role.

| domain | `tools/list` | `tools/call` |
| --- | --- | --- |
| a tenant's | Its `mcp` routes whose `roles` include `{role}` (all of them on bare `/mcp`) and whose requirement the caller satisfies, each with its input schema | Evaluates the route exactly like HTTP; the box's response body is the tool result. |
| `ops.` | The operator tools from [porch](../porch/IDEA.md) tagged with `{role}`, intersected with the caller's `ops/{role}` grants (`ops/admin` holds every role) | Porch forwards to `rancher.tools.call` or `store.tools.call` with the caller's subject and grants; the owning service checks again. |

`/_ops/api/{tool}` on `ops.` is the same catalog as plain HTTP: `POST` a JSON body, get the tool's JSON answer, with the same filtering.

- **The path focuses; the grant decides.** A tool outside the path's role is not listed and is refused if called (`invalid`); a listed tool the caller isn't granted is refused (403 over HTTP, an MCP error over MCP).
- **Sub-agents.** A lead agent holding `ops/admin` can ask permit for an agent token with only `ops/content`, and hand it to a sub-agent that connects to `/mcp/content`. Permit guarantees an agent never holds more than its subject.
- **Authentication.** An agent uses an OAuth token from permit (`kind: agent`); without one, `tools/list` answers only routes requiring nothing, and the `WWW-Authenticate` header points at `/_permit/`.

### Connecting an account

`GET /_hound/connect/{name}` starts an OAuth authorization for the signed-in caller against the egress service `{name}` (as bound on a route of this tenant with `credential.kind: oauth2-user`), and `GET /_hound/connect/{name}/callback` finishes it. Barn exchanges the code with hound, seals the tokens with cellar, and writes them as a system write to `_credentials/{!user_id}` in pond. Then it redirects to the route's `connected` path. The box never sees the tokens; it sees only whether the caller is connected (`hound@1.connected`).

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

### Events

Barn opens `pond.events.subscribe`, one stream across tenants, with a window equal to the events it can run at once. Each event names a tenant, an event route, and `{seq, bucket, id, op, user, at}`, under a 30-second lease. Barn evaluates it as a request on that `event` route: objects bind `{event.id}` and `{event.user}`, and the pond write's idempotency key is `event:{route}:{seq}`.

| box result | barn sends pond |
| --- | --- |
| Returned 0, write landed (or no write) | `pond.events.ack {lease, outcome: done}` |
| Trap, limit, non-zero, conflict, dependency down | `pond.events.ack {lease, outcome: failed, reason}` with the problem code |
| The route is gone from this barn's config, or the tenant is suspended | `pond.events.ack {lease, outcome: skipped, reason}` |
| A beta-only event route | Runs as a shadow: merges recorded and dropped, then `done` |

Barn returns one message of credit after each ack. Events for one route and id arrive in order, and pond doesn't lease the next one until the previous is acknowledged.

### Health

`GET /_barn/health` answers 200 only when barn holds a complete rancher snapshot, has compiled every live box version, holds an id range, and its pond connection is up. Otherwise 503 with a JSON body naming what's missing and how stale the config is. Caddy and the load balancer use it; nothing else does.

`GET /_barn/ask?domain=example.com` answers 200 when the domain is in rancher's domain registry (any role: primary, redirect, beta, ops) and verified, and 404 otherwise. Caddy's on-demand TLS asks it before getting a certificate, so a tenant's domain gets one the moment rancher registers it, and nobody edits the Caddyfile ([farmhand](../farmhand/IDEA.md) sets the `ask` once). It is served only on loopback.

## The box ABI

A box is a WebAssembly component in the reactor model. Its world imports `barn-abi@1`; if it writes, `iron@1`; if it declares `assets`, `silo@1`; and if it declares `egress`, `hound@1`. It exports the functions its routes name. A box may import nothing else; rancher refuses the upload otherwise.

```wit
package homestead:barn@1.0.0;

interface request {
  record header { name: string, value: string }
  record request {
    id: string,                    // the request id
    protocol: string,              // "http", "sse", "mcp", "mail", "timer", "event"
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

interface util {
  log: func(level: u8, text: string);            // a box log line (Q11.1)
  random: func(len: u32) -> list<u8>;            // seeded per request; the seed is logged
}

world box {
  import request; import response; import query; import util;
}
```

| import | rule |
| --- | --- |
| `request.get` | The caller's raw token, cookies, and `x-homestead-*` are never present. On an event route, `protocol` is `"event"` and `params` carries `event.seq`, `event.bucket`, `event.id`, `event.op`, `event.user`, and `event.at`. |
| `query.named` | Runs one of the route's named queries; params are JSON, bound, never spliced. Reserved params come from barn. |
| `query.sql` | Ad-hoc SQL under the route's grant, checked by fork in pond on every call. Refusals return the fork error text and become a problem. |
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
// a box that declares assets or egress includes `import silo;` or `import hound;` (below) in its world
```

- **A box can only name slots.** No bucket, no id, no tenant appears in any call.
- **Errors are returned, not trapped.** A box that ignores `read-only` still writes nothing to that slot, and barn records `iron.slot_read_only` as a problem.
- **Effects are iron's**, not barn-abi's, because they share the write's fate. `grant` substitutes the route's bound parameters into its `gives` and refuses anything else; `mail` checks the template renders, the recipient isn't suppressed, and the per-address limit.
- **Caps:** at most 64 slots per route, 1 MiB of composed patch per slot, 4 MiB per write, 32 effects per write.

### silo@1

```wit
interface silo {
  type asset = string;             // a name from the route's assets
  record object { key: string, size: u64, type: string, sha256: string, etag: string, modified-ms: s64 }
  record page { objects: list<object>, cursor: option<string> }
  record attachment { field: string, key: string, size: u64, type: string, sha256: string }

  head: func(a: asset, key: string) -> result<option<object>, silo-error>;
  get: func(a: asset, key: string, offset: u64, len: u32) -> result<list<u8>, silo-error>;
  put: func(a: asset, key: string, bytes: list<u8>) -> result<object, silo-error>;      // small objects, at most 4 MiB
  %list: func(a: asset, under: string, cursor: option<string>, limit: u32) -> result<page, silo-error>;
  delete: func(a: asset, key: string) -> result<_, silo-error>;
  presign: func(a: asset, key: string, method: string, ttl-ms: u32) -> result<string, silo-error>;
  attachments: func() -> list<attachment>;     // what barn stored for this request, before the box ran

  enum silo-error {
    unknown-asset, key-invalid, outside-prefix, not-granted, too-large, type-refused, not-found, unavailable,
  }
}
```

- **Keys are relative to the asset's resolved prefix.** `put("photos", "a1b2.jpg")` on a route whose prefix is `photos/{!user_id}/` writes `t/{tenant}/photos/4294967100/a1b2.jpg`; the box never sees the left part.
- **Every call checks `access`.** `list` needs `list`, `put` and `presign` with PUT need `write`, `delete` needs `delete`, and `head`, `get`, and GET presigns need `read`.
- **Silo writes are not iron writes.** An object written by `put` exists whether or not the box's pond write lands. A box that needs the two to agree writes the object first and its reference in the pond write, and sweeps unreferenced objects on a timer.
- **On a shadow run,** `put`, `delete`, and PUT presigns return `not-granted`.

### hound@1

```wit
interface hound {
  type service = string;           // a name from the route's egress
  record header { name: string, value: string }
  record request { method: string, path: string, query: string, headers: list<header>, body: list<u8> }
  record response { status: u16, headers: list<header>, body: list<u8> }

  send: func(s: service, r: request) -> result<response, hound-error>;
  call: func(s: service, operation-id: string, params: list<u8>, body: list<u8>) -> result<response, hound-error>;  // needs a spec
  connected: func(s: service) -> bool;      // for oauth2-user credentials: has the caller connected an account?

  enum hound-error {
    unknown-service, method-refused, outside-prefix, header-refused, not-connected,
    spec-missing, operation-unknown, params-invalid, too-large, rate-limited, timeout, unavailable,
  }
}
```

| rule | |
| --- | --- |
| Origin | Fixed by the service name. `path` is a path, never a URL. |
| Path | Normalized, then required to start with one of the route's `prefixes`. |
| Method | One of the route's `methods`. |
| Headers | Any, except `Authorization`, `Cookie`, `Host`, `Proxy-*`, and the credential's own header, which hound sets. |
| Redirects | Followed only within the origin and a prefix; otherwise returned as the response. |
| Caps | Request 1 MiB, response 8 MiB, time within the box's remaining deadline, the route's `rate` per tenant. |
| Shadow | Only services the route marks `effect-free`; others return `method-refused`. |
| Logs | One `egress` line per exchange: service, method, path template, status, ms, bytes. Never bodies or headers. |

The full design of hound, including OpenAPI handling, credential kinds, and its test runner, is in [hound](../hound/IDEA.md).

### Exports

| export | signature | when |
| --- | --- | --- |
| a route's `export` | `func() -> s32` | Every request on the route. 0 commits the staged write; anything else discards it and answers what the box set, or 500. |
| a route's `then` | `func(hold-timed-out: bool) -> s32` | After the write is durable and held tables applied (or timed out). It may read and query; any iron write call returns `read-only`. |
| a streaming export | `func(seq: u64, change: list<u8>) -> s32` | Once per pushed change (per group, or per viewer). |
| an event route's `export` | `func() -> s32`, the event in `request.get` | Once per leased event. 0 means `done`; anything else means `failed`. |
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
| hound calls | 8 per request | barn | `hound` returns `rate-limited` |
| silo calls | 64 per request | barn | `silo` returns `too-large` |
| attachments | route `max-files`, `max-bytes` each | route | 413 before the box wakes |

## The footprint

What barn calls on each service. Every one goes through that service's `_client` interface, and every one has a fake in barn's tests.

| service | barn uses | for |
| --- | --- | --- |
| pond | `pond.get`, `pond.write`, `pond.query`, `pond.names`, `pond.wait`, `pond.ids.lease`, `pond.feed.subscribe`, `pond.events.subscribe`, `pond.events.ack` | Objects, the write, SQL, name slots, held answers, minting, cache invalidation and SSE, event routes |
| rancher | `rancher.config.subscribe`, `rancher.box.fetch`, `rancher.health.report`, `rancher.problems.report`, `rancher.tools.list`, `rancher.tools.call` | Routes, policies, and sealed secrets; modules; counts; problems; the operator tools porch serves (including sun's, barker's, and ranger's) |
| permit | `permit.keys.subscribe`, `permit.grants.get`, `permit.http.serve` | Verifying tokens, live grants on a cache miss, sign-in pages |
| sun | `sun.fire.subscribe`, `sun.fire.ack`, `sun.fire.heartbeat` | Timer events |
| barker | `barker.suppression.subscribe`, `barker.thread.note`, `barker.bounce.ingest`, `barker.http.serve` | Refusing suppressed mail at the call, threads, bounces, unsubscribe |
| store | `store.release.fetch`, `store.tools.list`, `store.tools.call`, `store.http.serve` | Compiled releases, site tools through porch, upload tickets |
| S3 | the S3 API: get, put (multipart), head, list, delete, presign | Assets, under `t/{tenant}/`, with credentials unwrapped by cellar |
| third-party origins | HTTPS, through hound | A box's egress, to the origins and prefixes its route binds |
| ranger | `ranger.ingest` | Routing, egress, and agents lines, from the spool |

Under Q12.1's lean, barn does not drain the outbox and calls no `permit.effects.apply` or `sun.jobs.*`; if the answer goes the other way, those join this table.

## The invariants

- **Nothing calls barn but Caddy, the mail receiver, and the load balancer's health check.** Everything else arrives on streams barn opened.
- **Reserved prefixes never reach a box,** and proxying never interprets what it carries.
- **Barn owns the statuses that aren't the box's.** A conflict is 409, a trap 503, a deadline 504, and each is counted per route.
- **A box imports only `barn-abi@1`, `iron@1`, `silo@1`, and `hound@1`.** No WASI, no clock, no sockets.
- **A box names assets and keys, never S3 buckets or tenants.** Every key is checked under the route's prefix after normalizing.
- **A box names egress services, never hosts or credentials.** Origin, prefixes, methods, and the credential are the route's.
- **Events are acknowledged or leased again,** and written once by `event:{route}:{seq}`.
- **`/mcp/{role}` narrows; `ops/{role}` grants decide.**
- **A box names slots, never keys.** No bucket, id, or tenant crosses the ABI into barn.
- **Effects are iron calls,** and share the write's fate.
- **ABI errors are returned, not trapped,** and every refusal is a problem.
- **Barn's footprint on every other service is the table above,** through `_client` interfaces with fakes.
