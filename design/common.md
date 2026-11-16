# Common
The engineering decisions every homestead service shares
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's engineering practice held up better than its product design. C++ with a small set of mainstream libraries, one package per directory, a QUALITY.md per package, no globals, and a single config file were all choices that made the code legible to agents and reviewable by a person. Homestead keeps them, and applies them to several services instead of one host, which adds one new rule: services only know each other through a client package and a wire protocol.

## Foundation

| decision | value | carried from |
| --- | --- | --- |
| Language | C++20 for every service and library. | goober (machine/005.FirstVersion.md) |
| Build | CMake with Ninja, through `CMakePresets.json`; GoogleTest through CTest. | goober |
| Third-party baseline | Boost (Asio and Beast for HTTP, Boost.JSON for JSON), SQLite 3, GoogleTest. wasmtime through its C++ API, in barn only. | goober |
| Durable storage | SQLite, in WAL mode, for every service's own state. Pond adds its own write-ahead log in front of SQLite for client writes. | goober, narrowed |
| Wire | Between services, a multiplexed binary protocol over TCP, with FlatBuffers payloads and optional mutual TLS. HTTP only at the edge, in barn. See [The internal protocol](#the-internal-protocol). | new; goober's PHILOSOPHY § 7 asked for it |
| Ids | 64-bit, positive, globally unique, minted from ranges that pond vends, plus names that pond binds to ids. Strings in JSON. See [Ids](#ids). | goober (human/021.PrimaryKeys.md), extended |
| Failures | Elevated as structured problems an agent can act on, not only logged. See [PROBLEMS.md](common/PROBLEMS.md). | new |

## Packages

Every package is one directory under `src/`. The full rules (package anatomy, review tiers, kinds of package, the common packages, and how services are wired together) are in [packages](common/PACKAGES.md), and each service lists its own in `design/$service/PACKAGES.md`. The underscore in a package name is a namespace under `homestead::`, so `pond_wal` is `homestead::pond::wal` and builds the static library `homestead_pond_wal`. The long namespace can't collide with anything, and code inside a package rarely spells it out. Names are lean.

```
src/
  common_core/        homestead::common::core      clocks, errors, ids, names
  common_json/        homestead::common::json      thin Boost.JSON helpers, ids as strings
  common_http/        homestead::common::http      server and client on Beast, SSE (barn's edge)
  common_wire/        homestead::common::wire      the internal protocol: frames, streams, flow control
  common_tls/         homestead::common::tls       optional mutual TLS for the wire, host certificates
  common_sqlite/      homestead::common::sqlite    RAII over sqlite3, statements, pragmas
  fork_ast/           fork::ast                    the SQL tree (fork is a standalone library, see fork/PACKAGES.md)
  fork_parse/         fork::parse                  text -> tree
  fork_check/         fork::check_                 tree x permission set -> verdict
  fork_print/         fork::print                  tree -> canonical SQL
  pond_wal/           homestead::pond::wal
  pond_bucket/        homestead::pond::bucket
  pond_tailer/        homestead::pond::tailer
  pond_query/         homestead::pond::query
  pond_feed/          homestead::pond::feed
  pond_ids/           homestead::pond::ids         the counter, ranges, and names
  pond_client/        homestead::pond::client      the only way other services reach pond
  pond_system/        homestead::pond::system      composition root
  pond_main/                                       the binary
  barn_iron/          homestead::barn::iron        the write boundary: objects, merges
  barn_...  rancher_...  permit_...          the same shape
```

Each package has:

- **A `QUALITY.md`** that says what the package promises, how it is tested, and what "good" means for it. `common_http` is a faithful, boring adaptation of HTTP; `fork_check` is a security boundary and is tested like one.
- **Its own CMake target and tests.** A package lists its dependencies in its `CMakeLists.txt`, and anything not listed can't be included.
- **One clear responsibility.** Changes land per package, so a pull request is a meaningful change to one or a few packages, never a thousand files.

### Isolation

Every package is isolated, and so is every service. A package builds and tests on its own, from its own target, with only the packages it lists. A service is its own tree of packages, like goober's host was, with its own composition root and binary, and it builds, tests, and deploys without building any other service. A shared top-level CMake tree is only a convenience for building everything at once; nothing depends on it.

Isolation is also what makes deployment smooth. A deploy knows which packages changed, and so which services' binaries changed, and restarts only those. Each service has its own shutdown logic (barn drains requests, sun hands back leases, pond flushes and checkpoints its WAL), and pond, which holds the data, is built to change rarely, so most deploys never restart it.

### Dependency direction

Dependencies flow one way: `common_*` -> library packages -> service packages -> `$service_system` -> `$service_main`.

| rule | why |
| --- | --- |
| `common_*` depends only on third-party code. | Leaves stay reusable and testable alone. |
| `fork_*` depends on nothing but the standard library (Q12.5's lean). | The SQL checker is a pure function and must stay one, and it ships as a library outside homestead. |
| A service reaches another service only through its `_client` package. | barn includes `pond_client`, never `pond_bucket`. The wire protocol is the boundary, so a service can be rewritten, split, or clustered without its callers noticing. |
| Only `$service_system` wires packages together. | One composition root per binary, reachable from exactly one object. |

## Philosophy, kept

These come straight from goober's `machine/PHILOSOPHY.md`, because they were right.

- **No globals. Ever.** No singletons, mutable statics, or self-registering registries. Every piece of state is owned by something reachable from one root, `System`, and a test can build several systems in one process to simulate several machines.
- **Time, randomness, the filesystem root, and the network are injected.** Code takes a `Clock&`, never reads the clock itself. Tests use manual clocks, ephemeral ports, temp directories, and no sleeps.
- **One argument: a config file.** No flags and no environment side channels. Every setting has a conventional default, a config file states only what differs, and unknown keys are errors. Each package defines a plain `Config` struct; only the composition root knows the file format.
- **Lean on good libraries; write less.** Wrappers impose our boundaries (RAII, error types, naming) and don't re-abstract the library.
- **Reasonable tests, not exhaustive tests.** Test behavior at package boundaries and the edges that would hurt.
- **Built for AI-assisted development.** Predictable structure (directory == namespace == library), mainstream tools, small files, the why written down.

## Names

- **Chosen names** (tenants, buckets, tables, boxes, routes, services) match `[a-z][a-z0-9_]*`, at most 63 characters.
- **System-owned names start with `_`**, like the `_pond` management database or a `_documents` table, so a chosen name can never collide with one.
- **Names are refused, not repaired.** A bad name is an error that says what the rule is.

## Ids

Every object any service keys by an id gets one from a single counter that pond owns, following goober's primary key design.

| level | holds | gets more from |
| --- | --- | --- |
| Pond's counter | The next unused id, durable in pond's management database | Nothing; it only moves forward. |
| A barn host (or permit, or sun) | A leased range, like 10,000 ids | Pond, through `pond_client`. |
| A request | Ids minted one at a time from its barn's range | Its barn, in memory. |

- **An id names exactly one thing, anywhere.** A user, a job, a customer. That one thing may have a document in many buckets of its tenant (`profiles/42`, `addresses/42`), which is how data is split by facet.
- **A name is a string bound to an id, forever.** Pond binds names like `email:bob@example.com` per tenant, so an outside identifier always maps to the same id, and a create keyed by a name is naturally idempotent.
- **The first id is 2^32 + 1.** Code that truncates an id to 32 bits fails from the first id, in testing, as goober decided.
- **A range is vended once.** The counter moves forward durably before a range leaves pond, ranges are never reused, and gaps are fine.
- **Ids are strings in JSON**, and int64 everywhere else.
- **A box never chooses an id.** Its route resolves every object it may touch, from the caller, a bound parameter, a name, or a fresh mint.

## Services

| service | scale | owns |
| --- | --- | --- |
| pond | One machine, vertical first; a cluster later, with each tenant on exactly one machine | All durable tenant data, the WAL, the id counter |
| rancher | One machine | Boxes, routes (and with them every grant), traffic, domains, tenants as configuration, the problem log |
| permit | One machine | Identity, authentication, and grants per tenant, in its own database |
| barn | Horizontal and stateless | Nothing durable; caches only |
| sun | One machine | Temporal jobs, their leases, and the dead letter box |
| barker | One machine | Sending every email, for every service (see [barker](barker/IDEA.md)) |
| ranger | One machine | Logs from every service, searchable in real time (see [ranger](ranger/IDEA.md)) |
| store | One machine | Each tenant's site: pages, layouts, components, named queries, attachments, and immutable releases (see [store](store/IDEA.md)) |

Only barn is reachable from the internet, behind Caddy, which does TLS. Every other connection is the internal protocol.

## The internal protocol

Services talk to each other constantly: every request is a permit check, an object load, a query, and a write, every barn follows feeds from pond, rancher, permit, and sun, and sun pushes timer events. HTTP and JSON would spend most of that on parsing and connection setup. So the internal protocol is a binary, multiplexed protocol over long-lived TCP connections, in `common_wire`, built on Asio, and shared by every service.

| piece | design |
| --- | --- |
| Connection | One long-lived TCP connection (or a few) between each pair of machines, with a handshake that names the protocol version and the peer's service. |
| Frames | Length-prefixed: a stream id, a frame type (open, data, close, error, credit, ping), flags, and a payload. |
| Streams | Many concurrent streams per connection, so one slow query never blocks another call. A stream is a single call (request and response) or a long-lived subscription (a feed, timer events). |
| Flow control | Credit per stream and per connection, so a slow consumer slows its own stream, never the connection. Sun's timer credit is the same mechanism. |
| Payloads | FlatBuffers, from one schema per service in the repo, read without copying, with documents (JSON) carried as opaque bytes inside. Fields are only ever added, so old and new peers interoperate. Framing and multiplexing are our own, on Asio. |
| Calls | Every call has a deadline, carried with it, and an idempotency key where it changes state. |
| Pipelining | Writes are batched per connection, and pond can group commit them; a barn sends many tenants' writes on one connection. |

Each service publishes its calls through its `_client` package, which is the only code that knows the encoding. A service's API is the schema plus the client, written up in `design/$service/API.md`. The map of every connection, the frame layout, and who may call what are in [network](common/NETWORK.md).

### Security is optional, and pluggable

On one machine, or a private network, the internal protocol can run in plaintext. When services spread across machines, each host plugs in a certificate:

```json
{
	"wire": {
		"listen": { "address": "0.0.0.0", "port": 7400 },
		"tls": {
			"cert": "/etc/homestead/host.crt",
			"key": "/etc/homestead/host.key",
			"ca": "/etc/homestead/ca.crt",
			"require": true
		}
	}
}
```

- **With `tls`,** every connection is mutual TLS. A peer is accepted only if its certificate chains to the configured CA, and the certificate names the peer's service (like `barn` or `rancher`), which decides what it may call: only rancher may call pond's, permit's, and sun's control APIs.
- **Without `tls`,** the same service names are sent in the handshake and the same call rules are checked, but nothing is proven. That is for one machine or a network you trust. For now the service name is the whole handshake, with no shared secret per deployment; TLS is the planned answer to wiring two environments together by accident.
- **`require`** refuses plaintext peers, so a cluster can be moved to TLS one host at a time and then locked.
- **Certificates are reloaded** on SIGHUP, like the rest of the config, so rotation needs no restart.

## Docs

Design docs live in `design/$service/`, and each starts as an IDEA.md: a title, a one-line subtitle, a status line, the idea, then tables and JSON examples, open questions, and a closing list of invariants. A doc that grows past one idea splits into a new doc instead of a new section.
