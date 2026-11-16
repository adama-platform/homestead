# Network
Who talks to whom, over what, and what the bytes look like
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Homestead is a handful of single-machine services and a fleet of stateless barns, and almost all of their traffic is to each other: every request is a token check, an object load, a query, and a write, and every barn follows feeds from pond, rancher, permit, sun, and barker. [Common](../common.md#the-internal-protocol) settled the shape: one binary, multiplexed protocol over long-lived TCP, FlatBuffers payloads, optional mutual TLS. This doc pins it down: the map of every connection, the bytes on the wire, the rules every call and subscription follows, and who may call what.

Three rules carry most of the weight:

1. **The internet reaches barn, and only barn.** Caddy terminates TLS and speaks plain HTTP to barns. Every other service is on the private network, and reaches the internet only for its one job (barker to a mail provider, pond and ranger to object storage, barn to S3 for a tenant's assets and to the third-party origins a route grants a box, through hound).
2. **Barn only dials out.** Barn has no internal listener. It dials pond, rancher, permit, sun, barker, store, and ranger, and everything those services push to barn (config, feeds, timer events, suppression lists) arrives on streams barn opened. A barn is added by starting it and removed by stopping it; nobody keeps a list of barns to call.
3. **Only the dialer opens streams.** A connection has a dialer and a listener. The listener answers and pushes, but never opens a stream, so every stream has one obvious owner and one obvious place to cancel it.

## The map

```
                                 the internet
                                      |
               browsers, agents (MCP) | mail from the world
                     |                |            |
                     v                |            v
              +-------------+         |     +-------------+
              |    Caddy    |  HTTPS  |     |  Stalwart   |  SMTP in
              +-------------+         |     +-------------+
                     | HTTP           |            | LMTP (Q12.3)
                     v                |            v
   +--------------------------------------------------------------+
   |  barn  barn  barn  ...   (stateless, horizontal, dial out)   |-----> third-party APIs (hound)
   +--------------------------------------------------------------+-----> S3 (silo: assets)
     |      |        |        |        |        |        |
     |pond  |rancher |permit  |sun     |barker  |store   |ranger
     v      v        v        v        v        v        v
 +------+ +-------+ +------+ +-----+ +------+ +-----+ +------+
 | pond | |rancher| |permit| | sun | |barker| |store| |ranger|
 +------+ +-------+ +------+ +-----+ +------+ +-----+ +------+
   |  ^      |  |      |  |    |       |  |      |      |
   |  |      |  |      |  +----|------>+  |      |      +--> object storage (archives)
   |  +------+  |      |       |          +--> SES / SMTP
   |  control   +---> permit, sun, barker, ranger: control and reads
   |
   +--> pond replicas (chain), object storage (snapshots, WAL)

   every service --> ranger   (log batches, from a local spool)
   pond --> barn              (events for event routes, on streams barn opened)
   every service --> rancher  (problem reports; barn and sun also health counts)
   permit, sun, barker, rancher --> pond  (id ranges; deletions; outbox, see Q12.1)
   store --> rancher (table definitions, releases);  permit --> barker (auth mail)
```

### Every edge

The dialer is on the left. "call" is a request and one answer; "stream" is a subscription the dialer opened and the listener pushes on; "push" is a stream the dialer sends on with acknowledgments back.

| dialer | listener | for | kind | methods |
| --- | --- | --- | --- | --- |
| Caddy | barn | Every request from the internet | HTTP/1.1 | the edge (see [barn's API](../barn/API.md)) |
| Stalwart | barn | Inbound mail, including bounces | LMTP (Q12.3) | the mail edge |
| barn | pond | Reads, writes, queries, names, ids, waits | call | `pond.get`, `pond.write`, `pond.query`, `pond.names`, `pond.ids.lease`, `pond.wait` |
| barn | pond | Cache invalidation, SSE, held answers | stream | `pond.feed.subscribe` |
| barn | pond | Events for `event` routes, leased per route, acknowledged per event | stream, call | `pond.events.subscribe`, `pond.events.ack` |
| barn | rancher | Configuration, modules | stream, call | `rancher.config.subscribe`, `rancher.box.fetch` |
| barn, sun | rancher | Health and failure counts, problems | push | `rancher.health.report`, `rancher.problems.report` |
| barn | rancher, store | Operator tools, served by porch inside barn as MCP and HTTP | call | `rancher.tools.list`, `rancher.tools.call`, `store.tools.list`, `store.tools.call` |
| barn | rancher | OpenAPI specs for egress bindings, by hash | call | `rancher.spec.fetch` |
| barn | permit, barker, store | The proxied pages: sign-in and consent, unsubscribe, store's upload tickets | call | `$service.http.serve` |
| barn | permit | Signing keys, revocations, grant changes; live grants | stream, call | `permit.keys.subscribe`, `permit.grants.get` |
| barn | sun | Timer events to run, with credit | stream | `sun.fire.subscribe`, `sun.fire.ack`, `sun.fire.heartbeat` |
| barn | barker | Suppression hashes; thread notes | stream, call | `barker.suppression.subscribe`, `barker.thread.note`, `barker.bounce.ingest` |
| barn | store | Compiled releases, by hash | call | `store.release.fetch` |
| barn, every service | ranger | Log lines | push | `ranger.ingest` |
| every service | rancher | Problems | push | `rancher.problems.report` |
| rancher | pond | The control API | call | `pond.control.*` |
| rancher | permit | Realms, users, operators, audit | call | `permit.control.*` |
| rancher | sun | Recurring schedules, dead letters | call | `sun.control.*` |
| rancher | barker | Operator mail, domains, dead letters | call | `barker.send`, `barker.control.*` |
| rancher | ranger | Log reads, proxied for operators and agents; a deleted tenant's purge | call, stream | `ranger.search`, `ranger.tail`, `ranger.trace`, `ranger.count`, `ranger.verify`, `ranger.control.*` |
| permit, sun, barker, rancher | pond | Id ranges | call | `pond.ids.lease` |
| permit, sun, barker | pond | User and tenant deletions | stream | `pond.deletions.subscribe` (Q12.2) |
| sun, permit, barker | pond | Outbox elements addressed to them | stream | `pond.outbox.lease`, `pond.outbox.ack` (Q12.1) |
| permit | barker | Sign-in codes and grant notices | call | `barker.send` |
| store | rancher | Table definitions; release routes and promotion requests | call | `rancher.tables.get`, `rancher.release.submit`, `rancher.release.propose` |
| store, barker, permit | rancher | Templates, table versions, and the role tree, as configuration | stream | `rancher.config.subscribe` |
| pond | pond | Chain replication of the WAL | push | `pond.replica.append` |

Outside the wire, barn reaches S3 (assets, under a route's prefix rules, with credentials from cellar) and the third-party origins an egress binding names (through hound), and a browser may PUT or GET straight to S3 with a URL a box presigned within its prefix. Farmhand reaches machines over SSH to deploy; it is a tool, not a peer.

Nothing else talks. In particular: **no service but barn and rancher calls pond's data API**, **only rancher calls a control API**, and **nothing calls barn**.

### Ports

Every service listens on its own port, so the whole system can run on one machine, and the same port everywhere else, so a config never has to guess.

| service | listens on | for |
| --- | --- | --- |
| barn | 8080 | HTTP from Caddy, and `/_barn/health` for the load balancer |
| barn | 2424 | LMTP from the mail receiver |
| rancher | 7401 | wire |
| pond | 7402 | wire (clients, and replicas from other ponds) |
| permit | 7403 | wire |
| sun | 7404 | wire |
| barker | 7405 | wire |
| ranger | 7406 | wire |
| store | 7407 | wire |

### Finding each other

A service's config file names the address of every service it dials. There is no discovery service, and no DNS trick: six single-machine services are six lines of config.

```json
{
	"peers": {
		"rancher": "10.0.0.2:7401",
		"pond": "10.0.0.3:7402",
		"permit": "10.0.0.2:7403",
		"sun": "10.0.0.2:7404",
		"barker": "10.0.0.2:7405",
		"ranger": "10.0.0.4:7406",
		"store": "10.0.0.2:7407"
	}
}
```

The one exception is pond, later. When pond becomes a cluster, the `pond` peer is a seed: the dialer asks it for the placement map (which pond machine holds which tenant, under which lease epoch), which rancher publishes, and opens connections to the ponds it needs. A call that reaches the wrong pond is refused with `moved`, naming the right one. Until then, there is one pond and the map has one entry.

## The protocol

The protocol lives in `common_wire`, on Asio. Every service uses it the same way, through its `_client` package (see [packages](PACKAGES.md)), so nobody but `common_wire` knows what a frame looks like.

### A connection

```
dialer                                            listener
  | -- TCP connect (then TLS, if configured) ------> |
  | -- preface "HSTD" u16 version u16 0 ----------->  |
  | -- HELLO { service, host, build, limits } ---->  |
  |  <------ HELLO { service, host, build, limits } |
  | -- OPEN stream 1 { method, deadline, ... } --->  |
  | -- DATA stream 1 [END] request --------------->  |
  | -- OPEN stream 3 ...                             |   many streams at once
  |  <---------------- DATA stream 1 [END] answer -- |
  |  <---------------------- CLOSE stream 1 { ok } - |
  | -- PING ----------------------------------------> |   every 10 s when idle
  |  <-------------------------------------- PONG -- |
```

- **One connection per pair of processes**, by default, opened at start and kept. A dialer may open a few more (configurable, default 1) when one connection's window is the bottleneck; streams are spread across them.
- **The preface** is 8 bytes: the ASCII magic `HSTD`, the protocol version (`1`), and two reserved bytes. A listener that doesn't recognize the magic or the version closes the connection, so an HTTP client or a future protocol fails at once rather than half-working.
- **HELLO** is the first frame each way, on stream 0. It names the peer's service (`barn`, `rancher`, ...), its host, its build, and its limits. Nothing else may be sent before the peer's HELLO arrives.
- **Keepalive.** Each side PINGs after 10 seconds of silence and closes the connection after 30 seconds without hearing anything. Closing fails every open stream with `unavailable`.
- **Reconnects** back off from 100 ms, doubling, to 5 seconds, with jitter. Subscriptions resume from the last seq they saw (see [Subscriptions](#subscriptions)).
- **GOAWAY** is a graceful close: the listener says the last stream id it will finish, finishes those, and refuses new ones with `unavailable`, so a restart drains instead of dropping. Every service sends it on shutdown.

### Frames

Every frame is a 12-byte header and a payload. Integers are little-endian, like FlatBuffers.

```
 0               4               8       9       10              12
 +---------------+---------------+-------+-------+---------------+------------------+
 | length  (u32) | stream  (u32) | type  | flags | reserved (0)  | payload ...      |
 +---------------+---------------+-------+-------+---------------+------------------+
   payload bytes    0 = the          u8      u8       u16           length bytes
   after header     connection
```

| type | code | stream | sent by | payload |
| --- | --- | --- | --- | --- |
| HELLO | 0x01 | 0 | both, first | `Hello` |
| OPEN | 0x02 | new, odd, increasing | dialer | `Open` |
| DATA | 0x03 | open | both | one message (or part of one), a FlatBuffer of the method's type |
| CLOSE | 0x04 | open | listener | `Status`: the stream is finished, and how |
| CANCEL | 0x05 | open | dialer | empty: the dialer gave up; the listener stops work and sends nothing more |
| CREDIT | 0x06 | 0 or open | both | `u32`: more bytes (stream 0) or more messages (a stream) the sender may receive |
| PING | 0x07 | 0 | both | 8 opaque bytes |
| PONG | 0x08 | 0 | both | the PING's 8 bytes |
| GOAWAY | 0x09 | 0 | both | `GoAway { last_stream, reason }` |

| flag | bit | on | means |
| --- | --- | --- | --- |
| END | 0x01 | DATA | The sender's last message on this stream. A call's request and its answer each carry END. |
| MORE | 0x02 | DATA | The message continues in the next DATA frame on this stream. |

- **A frame is at most 1 MiB** of payload; a message larger than that is split with MORE. **A message is at most 16 MiB** (the HELLO limits may lower either). Anything bigger (a box module, a compiled release, a tenant snapshot) is a stream of chunk messages, never one message.
- **An unknown frame type, a frame on stream 0 that belongs on a stream, or a stream id that goes backward** is a protocol error: the receiver sends GOAWAY with the reason and closes the connection.
- **Odd stream ids, from the dialer only.** Even ids are reserved, in case a later version lets listeners open streams.

### The envelope

The connection-level messages are one FlatBuffers schema in `common_wire`:

```
namespace homestead.wire;

table Hello {
  service: string;          // "barn", "pond", ... checked against the TLS certificate when present
  host: string;             // "barn-7"
  build: string;            // the build id, logged, never trusted
  max_frame: uint32;        // default 1 MiB
  max_message: uint32;      // default 16 MiB
  window: uint32;           // connection byte credit this side grants, default 16 MiB
}

table Open {
  method: string;           // "pond.write"
  deadline_ms: uint32;      // remaining budget when sent, relative, so clocks needn't agree
  request: string;          // the request, timer job, or message id that ties lines in ranger
  idempotency: string;      // required on every method that changes state
  tenant: string;           // for logs, limits, and placement; the payload still names it
  window: uint32;           // message credit the dialer grants for this stream (subscriptions)
}

enum Code : ubyte {
  ok, invalid, refused, not_found, conflict, gone,
  unavailable, overloaded, deadline, moved, reset, unimplemented, internal
}

table Status {
  code: Code;
  problem: string;          // a code from the problem catalog, like "iron.slot_read_only"
  message: string;          // for people and agents; never tenant data
  retry_after_ms: uint32;   // a hint, with overloaded and unavailable
  detail: [ubyte];          // optional, a FlatBuffer the method's schema names
}

table GoAway { last_stream: uint32; reason: string; }
```

Each service's own messages live in its `$service_proto` package, one `.fbs` per service, in `namespace homestead.$service`. A method names its request and answer types there.

### Calls

A call is one stream: OPEN, one DATA from the dialer with END, then from the listener one DATA with END and a CLOSE with `ok`, or only a CLOSE with an error.

- **Every call has a deadline.** `deadline_ms` is the time the dialer has left, and the listener subtracts its own queueing from it. Work past the deadline is abandoned and answered `deadline`. A call that makes further calls (barn's request to pond) passes on what is left, never more. There is no "no deadline."
- **Every call that changes state carries an idempotency key.** The listener remembers recent keys (per method, for a window the method documents) and answers a repeat with the original answer, so a dialer may always retry after `unavailable` or `deadline`. A changing call without a key is refused with `invalid`.
- **Compare-and-set where it matters.** Calls that change a versioned thing take an `if-seq` in their payload and answer `conflict` when it moved.
- **Cancel is cooperative.** CANCEL tells the listener to stop; the dialer treats the call as failed at once and never waits for an answer.

### Subscriptions

A subscription is a stream the dialer opens with `from` (a seq) and a message window, and the listener pushes events on until one side ends it.

- **Resume by seq.** Every event carries the seq it reflects; a dialer that reconnects opens a new stream `from` the last seq it processed. Events are delivered in seq order, at least once.
- **Reset, not silence.** When `from` is older than what the listener keeps, the listener sends one `reset` event and continues from now; the subscriber drops what it derived and starts over. A subscriber never silently misses an event.
- **Credit is the brakes.** The listener sends at most `window` messages beyond what the dialer has credited back. A slow subscriber slows only its own stream, never the connection. A listener that has buffered too much for a stalled subscriber closes that stream with `overloaded`, and the subscriber resumes by seq.
- **Credit can mean capacity.** For sun's `fire.subscribe`, a message of credit is one timer event a barn can run at once, so load balancing is just the credit each barn hands back.
- **Snapshots first.** Subscriptions to configuration-like state (rancher's config, permit's keys, barker's suppression list) start with a snapshot at a seq and then follow the log from that seq, so a cold subscriber needs no second call.

### Pushes

A push is a stream the dialer sends on: log batches to ranger, problem and health reports to rancher, WAL records to a pond replica. Each batch is a DATA message with a seq; the listener sends back a small acknowledgment message naming the highest seq it made durable, and credit for more. The dialer keeps unacknowledged batches (ranger's spool, problem buffers) and resends them after a reconnect, and the listener deduplicates by seq.

### Codes

| code | means | retry? |
| --- | --- | --- |
| ok | Done. | |
| invalid | The request is malformed or breaks a rule; the message says which. | No. Fix the request. |
| refused | The caller may not do this: wrong service, suspended tenant, a grant or rule that says no. | No. |
| not_found | The thing named doesn't exist. | No. |
| conflict | An `if-seq` or compare-and-set lost. | Only after re-reading. |
| gone | The thing existed and was deleted (a dead key, a dropped table version). | No. |
| unavailable | The listener can't serve now: starting, draining, or a dependency is down. | Yes, with backoff, same idempotency key. |
| overloaded | Too much in flight; `retry_after_ms` says when to come back. | Yes, after the hint. |
| deadline | The deadline passed before the work finished. Changing calls may or may not have happened. | Yes, same idempotency key. |
| moved | This pond doesn't hold the tenant; `detail` names the one that does. | Yes, there. |
| reset | A subscription's `from` is too old. | Resubscribe from now. |
| unimplemented | Unknown method: the peer is older or newer. | No. |
| internal | A bug. Reported as a problem by the listener. | Once, then give up. |

Every non-ok code from a service carries a `problem` code from the [problem catalog](PROBLEMS.md) when one applies, so the dialer can report it with its own context without inventing a second name.

## Who may call what

The listener checks the caller's service on every OPEN against its method table. With TLS, the service comes from the certificate; without TLS, it comes from HELLO, and nothing is proven (see [Security](#security)).

| method family | barn | rancher | pond | permit | sun | barker | store | ranger |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `pond.get`, `write`, `query`, `names`, `wait`, `feed.*` | yes | yes | | | | | | |
| `pond.events.*` | yes | | | | | | | |
| `pond.ids.lease` | yes | yes | | yes | yes | yes | | |
| `pond.outbox.*`, `pond.deletions.*` | (Q12.1) | | | yes | yes | yes | | |
| `pond.replica.*` | | | yes | | | | | |
| `pond.control.*` | | yes | | | | | | |
| `rancher.config.subscribe` | yes | | | yes | | yes | yes | |
| `rancher.box.fetch`, `spec.fetch`, `tools.*` | yes | | | | | | | |
| `rancher.health.report` | yes | | | | yes | | | |
| `rancher.problems.report` | yes | | yes | yes | yes | yes | yes | yes |
| `rancher.tables.get`, `release.*` | | | | | | | yes | |
| `permit.keys.*`, `grants.get`, `http.serve` | yes | | | | | | | |
| `permit.effects.apply` | (Q12.1) | | | | | | | |
| `permit.control.*` | | yes | | | | | | |
| `sun.fire.*` | yes | | | | | | | |
| `sun.jobs.*` | (Q12.1) | | | | | | | |
| `sun.control.*` | | yes | | | | | | |
| `barker.send` | | yes | | yes | | | | |
| `barker.suppression.*`, `thread.note`, `bounce.ingest`, `http.serve` | yes | | | | | | | |
| `barker.control.*` | | yes | | | | | | |
| `store.release.fetch` | yes | | | | | | | |
| `store.tools.*`, `http.serve` | yes | | | | | | | |
| `ranger.ingest` | yes | yes | yes | yes | yes | yes | yes | |
| `ranger.search`, `tail`, `trace`, `count`, `verify`, `control.*` | | yes | | | | | | |

A refused OPEN is answered `refused` and reported as a problem, since a service calling something it shouldn't is a bug or an attack, and either is worth seeing.

### The proxied control plane

Two kinds of control-plane traffic reach services through barn.

**Operator tools** are typed. Each service that owns tools (rancher, and store for the site; rancher also fronts sun's, barker's, and ranger's) answers `tools.list` with descriptors (`name`, `summary`, `input_schema`, `roles`, `changes`) and `tools.call` with the caller attached. [Porch](../porch/IDEA.md), a library inside barn, gathers the catalogs and serves them as MCP at `/mcp` and `/mcp/{role}`, and as JSON at `/_ops/api/{tool}`, on `ops.` domains. The role in the path narrows the catalog; the caller's `ops/{role}` grants decide; the owning service checks again.

**Pages** are opaque: sign-in pages, consent screens, unsubscribe links, and store's upload tickets. Barn doesn't interpret them. It carries each request as an `http.serve` call:

```
table HttpRequest {
  method: string; path: string; query: string;
  headers: [Header];        // hop-by-hop headers and anything starting with x-homestead- removed
  body: [ubyte];
  domain: string;           // the domain it arrived on
  tenant: string;           // resolved by barn from the domain registry; null for the platform
  peer: string;             // the client address Caddy reported
}
table HttpResponse { status: uint16; headers: [Header]; body: [ubyte]; }
```

| prefix on a tenant's domain | goes to |
| --- | --- |
| `/_permit/` | permit: sign-in, passkeys, consent, OAuth endpoints |
| `/_unsubscribe/`, `/_barker/` | barker: one-click unsubscribe, provider notifications |
| `/_ops/site/upload/` on `ops.` | store: one-time upload tickets for site attachments |
| `/mcp`, `/mcp/{role}`, `/_ops/api/` on `ops.` | not proxied: porch, in barn, calls `rancher.tools.*` and `store.tools.*` |
| `/_hound/connect/` | not proxied: barn and hound run the OAuth connect flow for an egress binding |

Barn resolves the tenant from the domain before it proxies, exactly as for a box route, so a proxied service gets a tenant it can trust and never takes one from a path or header. Bodies are capped (1 MiB, larger for store's uploads by ticket), and the proxied service sees only what the request carried; barn adds no credentials of its own.

## Security

Security on the wire is optional and pluggable, as [common](../common.md#security-is-optional-and-pluggable) says. This section is what that means in bytes.

- **Plaintext** is for one machine or a private network the operator trusts. HELLO names the service and the listener applies the method table, but a liar is believed.
- **Mutual TLS** (TLS 1.3 only, through OpenSSL under Asio) wraps the connection before the preface. Each host has a certificate from the deployment's CA whose subject alternative name is a URI, `homestead://$service/$host`, like `homestead://barn/barn-7`. The listener takes the caller's service from that URI, refuses a HELLO that names a different one, and applies the method table to the certificate's service. A host certificate can name one service; a machine running several services has one certificate each.
- **`require`** refuses plaintext peers, so a deployment moves to TLS one host at a time and then locks.
- **Rotation** is a SIGHUP: new connections use the new certificate, and existing ones live out their lifetime (at most 24 hours; connections older than that are drained with GOAWAY and reopened).
- **Secrets ride only sealed.** Rancher's config log carries secrets sealed to the deployment's wrap key, and only to the services a secret names; only a host whose vault holds the unwrap key can open one, with [cellar](../cellar/IDEA.md).
- **No secret ever rides in the envelope.** Tokens a box's caller presented are verified in barn and never forwarded; what crosses the wire is a subject id and grants, which the listener already trusts barn to say.
- **The tenant on the wire is barn's or rancher's word.** Pond, permit, sun, and barker trust the tenant a barn or rancher names, and refuse one from any other caller. `Open.tenant` is for logs and placement only; authority is the payload's, checked against the caller's service.

## Budgets

Rough numbers to design against, for a barn answering one request that reads two objects, runs one query, and writes:

| hop | calls | budget |
| --- | --- | --- |
| barn -> permit | 0 (tokens verify locally; grants cached from the feed) | 0 ms |
| barn -> pond, load objects | 1 call per object, or 0 from cache | < 1 ms each on a LAN |
| barn -> pond, query | 1 call, or 0 from cache | < 2 ms for an indexed read |
| barn -> pond, write | 1 call, fsync with group commit | 2-5 ms |
| barn -> ranger | 0 on the request path (spooled) | 0 ms |

Nothing on the request path waits on rancher, ranger, store, sun, or barker. Losing any of them degrades what they own (config changes, logs, edits, timers, mail) and nothing else.

## Testing the network

- **`common_wire` has an in-memory transport** with the same frames, so a test builds several services' systems in one process, connects them with loopback pairs, and drives time with a manual clock, as goober's philosophy asks. Fault injection (drop, delay, reorder across connections, sever) is a property of the loopback, not a mock.
- **Each `_proto` has round-trip tests**, and a compatibility test that decodes messages written by the previous release's schema, so "fields are only ever added" is checked, not hoped.
- **The frame parser is fuzzed.** It is the first code that touches bytes from another machine.

## Open questions

See [OPEN.md](../OPEN.md#12-network-and-packages): who drains the outbox (Q12.1), how deletions reach the services that keep copies (Q12.2), and how the mail receiver hands mail to barn (Q12.3).

## The invariants

- **Only barn is on the internet.** Behind Caddy, which does TLS.
- **Barn only dials out.** Nothing calls barn but Caddy and the mail receiver.
- **Only the dialer opens streams.** The listener answers and pushes.
- **Every call has a deadline, and every changing call has an idempotency key.** Retrying with the same key is always safe.
- **Subscriptions resume by seq, and a gap is a `reset`, never silence.**
- **Credit is per stream.** A slow consumer slows only itself.
- **Every OPEN is checked against the caller's service.** With TLS, the service is proven by the certificate.
- **The tenant on the wire comes from barn or rancher.** Never from anyone else, and never from `Open`.
- **Nothing on the request path waits on rancher, ranger, store, sun, or barker.**
- **Only `common_wire` knows what a frame looks like, and only a service's `_client` and `_server` know its schema.**
