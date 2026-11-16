# Packages
How homestead is cut into packages, how they connect, and the order a person reviews them in
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's best engineering decision was the package: one directory, one namespace, one CMake target, one QUALITY.md, its own tests. It turned "the host" into a few dozen things an agent could hold in its head one at a time, and it let a pull request be a meaningful change to one package instead of a thousand files. Homestead keeps the package and takes it further, because homestead has a second goal goober didn't: **a person can't review everything, so the code has to say what deserves review first.**

So every package declares three things beside its code:

1. **Its contract**: the public headers in `include/`, which are all another package may see.
2. **Its quality bar**: the QUALITY.md, saying what it promises, how that is tested, and what it will never do.
3. **Its review tier**: how bad a bug in it would be, which decides who reads a change to it and how closely.

An agent working on a package loads that package's QUALITY.md, its own code, and the public headers of the packages it lists. Nothing else is in scope, and the build enforces it. A person's review queue is sorted by tier, so the handful of packages where a bug is a breach get read line by line, and the many where a bug is a wrong log line get their tests read.

Each service's packages are in `design/$service/PACKAGES.md`. This doc holds what they share: the anatomy of a package, the tiers, the kinds of package, the common packages, and how services connect.

## A package

```
src/pond_wal/
  CMakeLists.txt            the target, and the only dependencies it may use
  QUALITY.md                what it promises, its tier, how it is tested, what it won't do
  include/homestead/pond/wal/
    wal.h                   the public contract: types, interfaces, free functions
  src/
    segment.cpp             private: nothing outside the package includes these
    group_commit.cpp
  test/
    wal_test.cpp            GoogleTest, through CTest, against the public headers
    recovery_test.cpp
```

- **Directory == namespace == library.** `pond_wal` is `homestead::pond::wal`, builds `homestead_pond_wal`, and its headers are included as `<homestead/pond/wal/wal.h>`.
- **Dependencies are declared, and only declared ones link.** `target_link_libraries` is the dependency list; a check in CI fails a package that includes a header from a package it doesn't list.
- **Tests use the public headers.** A test that needs a private header is a sign the contract is missing something.
- **Small.** A package that passes about 3,000 lines of code, or grows a second responsibility, splits. The split is a new directory, not a subdirectory.

### QUALITY.md

Every QUALITY.md has the same five headings, so an agent knows where to look and a reviewer knows what to check:

```markdown
# pond_wal
Tier: R0 (a bug loses acknowledged writes)

## Promise
A write acknowledged has been fsynced. Replay after any crash yields exactly the acknowledged records, in order.

## Contract
`Wal::append(Record) -> Future<Position>`, `Wal::replay(from, fn)`, `Wal::truncate(before)`. Single writer; many readers.

## Tests
Crash at every fsync boundary (fault-injecting filesystem). Torn tails. Group commit under 10k concurrent appends.
Replay is idempotent. Property test: any interleaving of append/crash/replay preserves the acknowledged prefix.

## Won't
Know what a tenant, bucket, or document is. Decide what to write. Compress.

## Watch
Group commit latency; segment count; the oldest unshipped segment.
```

`Watch` names what the package reports when it runs (metrics, problems, log lines), so an operator knows where its health shows up.

## Review tiers

| tier | a bug here is | a change needs | examples |
| --- | --- | --- | --- |
| **R0** boundary | A breach, a lost acknowledged write, or a person's data reaching someone else | A person reads every line of the diff and its tests, and a second agent reviews adversarially first. Fuzz or property tests are required, not optional. | `fork_check`, `barn_iron`, `pond_wal`, `permit_token`, `common_tls` |
| **R1** core | A wrong answer, a stuck request, a broken feature, recoverable without data loss | A person reads the QUALITY.md diff, the public header diff, and the tests; samples the code. | `pond_tailer`, `barn_policy`, `sun_recur`, `store_compile` |
| **R2** plumbing | Something slow, noisy, or unavailable, caught by tests or by watching | A person reads the QUALITY.md diff and skims; an agent reviews the rest. | `pond_server`, `rancher_health`, `barker_domain` |
| **R3** leaf | Something cosmetic, or a generated or boring adaptation of a library | Tests pass and an agent reviews. A person sees it only in the weekly summary. | `$service_main`, `common_json`, `$service_proto` (generated) |

- **A pull request takes the tier of the highest package it touches.** The review queue is sorted by tier, then by age, so R0 changes never wait behind R3 ones and R3 changes never steal attention from R0 ones.
- **Changing a tier is itself an R0 change.** Lowering one is how scrutiny would quietly disappear, so it gets the scrutiny.
- **Tiers push code outward.** If an R0 package grows code that isn't boundary code, that code moves to a lower-tier package the R0 one depends on, so the R0 package stays small enough to read in full. Most of the design work in the per-service docs is exactly that: making the R0 packages tiny.
- **Public headers of R0 packages are R0.** A contract change in `fork_check` is read by a person even when the package under it is R1.

## Kinds of package

Every package is one of these kinds, and the kind decides what it may depend on:

| kind | named | is | may depend on |
| --- | --- | --- | --- |
| common | `common_*` | Leaves every service shares: clocks, ids, JSON, the wire, SQLite, logs | Third-party code and other `common_*` below it |
| library | `fork_*`, and a service's pure library packages (`permit_token`, `store_plan`, ...) | Pure code another service is allowed to link: no I/O, no threads, no clients | `common_*` (and fork: nothing at all, see [fork](../fork/PACKAGES.md)) |
| core | `$service_*` | The service's own logic, each one responsibility | `common_*`, libraries, its own service's core packages, other services' `_client` interfaces |
| proto | `$service_proto` | The service's FlatBuffers schema and generated code | `common_wire` |
| server | `$service_server` | Binds the proto's methods to core interfaces, checks the caller's service | proto, core |
| client | `$service_client` | The abstract interface other services program against, and its wire implementation | proto, `common_wire` |
| fake | `$service_fake` | An in-memory implementation of the client interface, for other services' tests | the client's interface, `common_*` |
| system | `$service_system` | The composition root: reads the config, builds everything, owns it | everything in its service |
| main | `$service_main` | The binary: one argument, a config file | system |

```
       common_*  <-----------------------------------------------+
          ^                                                       |
       libraries (fork_*, permit_token, store_plan, ...)          |
          ^                                                       |
  $service core packages  ---uses interface of--->  $other_client ---> $other_proto ---> common_wire
          ^                                              ^
  $service_server ---> $service_proto                    |
          ^                                         $other_fake (tests only)
  $service_system  (builds core, server, and the clients it dials)
          ^
  $service_main
```

### Library packages

A library package is code another service runs in its own process, which is the one exception to "services meet only at the wire." It earns the exception by being **pure**: data and functions, no sockets, no files, no threads, no clock it didn't get passed. The cases:

| library | owned by | linked by | why it isn't a call |
| --- | --- | --- | --- |
| `fork_*` | fork | pond, rancher, store, barn (optional) | A SQL check on every query can't be a network hop, and it must give the same verdict everywhere. |
| `permit_token`, `permit_require` | permit | barn | Barn verifies every token and checks every requirement locally; permit is off the request path. Signing is in a separate target barn never links, so no barn can mint a token. |
| `barker_reply` | barker | barn | Barn verifies reply tokens that barker minted, with a shared key, without calling barker. |
| `store_plan` | store | barn | Barn executes the page plans store compiles; the format must be one definition. |
| `rancher_model` | rancher | barn, store | Routes, services, and policies as data, so barn's routing and rancher's checks read one definition. |
| `common_mustache` | common | store, barker, barn | One template engine, one escaping rule. |
| `cellar_*` | cellar | barn, permit, barker, pond, ranger; rancher (seal only) | Secrets are sealed in rancher and opened at the moment of use with the host's vault key; a network hop for a plaintext secret would be a worse boundary. See [cellar](../cellar/PACKAGES.md). |
| `hound_*` | hound | barn, and the `hound` CLI | A box's outbound call runs on its request, under its deadline and its route's grant. See [hound](../hound/PACKAGES.md). |
| `porch_*` | porch | barn | Operator tools belong to the services that implement them; porch only gathers, filters by role, and speaks MCP at barn's door. See [porch](../porch/PACKAGES.md). |

The owner of a library is the only team that changes it, and a change to one is reviewed at the higher of its own tier and its linker's. A library's QUALITY.md lists who links it.

## The common packages

| package | namespace | does | tier |
| --- | --- | --- | --- |
| `common_core` | `homestead::common::core` | `Clock` (and `ManualClock`), `Rng`, `Id` (int64, 2^32+1 floor, string form), names (`[a-z][a-z0-9_]*`, refused not repaired), `Result<T>`, `Error` | R1 |
| `common_config` | `homestead::common::config` | Strict JSON config to plain structs: defaults, unknown keys are errors, SIGHUP reload hooks | R2 |
| `common_json` | `homestead::common::json` | Thin Boost.JSON helpers; ids as strings in, int64 out | R3 |
| `common_merge` | `homestead::common::merge` | RFC 7386 apply and compose, size and depth caps. Barn composes a box's merges with it; pond applies them with it. | R1 |
| `common_sqlite` | `homestead::common::sqlite` | RAII over sqlite3: connections, statements, pragmas, WAL mode, read-only opens, the authorizer hook | R1 |
| `common_crypto` | `homestead::common::crypto` | Ed25519 sign and verify, HMAC, keyed hashes, SHA-256, secure random, over OpenSSL. No home-grown crypto. | R0 |
| `common_tls` | `homestead::common::tls` | Mutual TLS contexts, the `homestead://service/host` identity, reload on SIGHUP | R0 |
| `common_wire` | `homestead::common::wire` | The internal protocol: frames, connections, streams, credit, calls, subscriptions, pushes, the in-memory transport, the method table check | R0 (frame parsing), with the rest held to R1 |
| `common_http` | `homestead::common::http` | HTTP/1.1 server and client on Beast, SSE, the `HttpRequest`/`HttpResponse` used by `http.serve` | R1 |
| `common_blob` | `homestead::common::blob` | An S3-shaped object store: put, get, list, delete, with a local directory implementation and an S3 one | R2 |
| `common_log` | `homestead::common::log` | The ranger record envelope, per-stream seq, the hash chain, redaction of known body fields, the local spool | R1 (the chain and redaction are R0, and live in their own files with their own tests) |
| `common_problem` | `homestead::common::problem` | The problem record, the catalog of codes, a local aggregator by fingerprint, and a `Sink` interface | R2 |
| `common_mustache` | `homestead::common::mustache` | Mustache, escaping by default, no partials (components are a store concept) | R1 |
| `common_test` | `homestead::common::test` | Temp directories, ephemeral ports, a fault-injecting filesystem, a manual clock harness. Test-only. | R3 |

Third-party code, all mainstream: Boost (Asio, Beast, JSON), SQLite 3, OpenSSL 3, FlatBuffers, GoogleTest, and per service: wasmtime (barn), RE2 and zstd (ranger), md4c (store). hound needs a YAML parser for OpenAPI (yaml-cpp). Each is wrapped by one package, which is the only one that includes it.

## How everything connects

### Through interfaces, built in one place

Every service another service calls is seen through an abstract interface in its `_client` package. Core packages take that interface by reference; they never know whether it is the wire, a fake, or (in a test) the real service in the same process.

```cpp
// src/pond_client/include/homestead/pond/client/pond.h
namespace homestead::pond::client {
class Pond {
 public:
  virtual ~Pond() = default;
  virtual Future<GetAnswer> get(const Get&, Deadline) = 0;
  virtual Future<WriteAnswer> write(const Write&, Deadline) = 0;
  virtual Future<QueryAnswer> query(const Query&, Deadline) = 0;
  virtual Subscription feed(const FeedRequest&, FeedHandler&) = 0;
  // ...
};
std::unique_ptr<Pond> connect(common::wire::Peer&);   // the wire implementation
}
```

```cpp
// src/barn_system/src/system.cpp: the only place barn's parts meet
System::System(const Config& config, Clock& clock, Network& net)
    : pond_peer_(net.dial(config.peers.pond, "pond")),
      pond_(pond::client::connect(pond_peer_)),
      permit_(permit::client::connect(net.dial(config.peers.permit, "permit"))),
      cache_(config.cache, clock),
      iron_(clock),
      eval_(config.eval, *pond_, *permit_, cache_, iron_, clock),
      edge_(config.edge, eval_, net) {}
```

- **`Network&` is injected**, like the clock. Production passes a TCP network; tests pass the in-memory one.
- **Fakes are first-class.** `pond_fake` is an in-memory pond that honors the same contract (seqs, `if-seq`, idempotency, feeds), with its own tests against the same contract test suite the real pond passes. Barn's evaluator is tested against `pond_fake` without a pond.
- **Contract tests live with the client.** `pond_client` ships a `contract_test` suite that any `Pond` must pass: the wire client against a real pond, and the fake. That is how the fake stays honest.

### A call, end to end, in packages

```
barn_eval        pond::client::Pond::write(...)              (interface)
pond_client      encode with pond_proto, open a stream       (R2)
common_wire      frames, credit, deadline                    (R0/R1)
   ... TCP, maybe TLS (common_tls) ...
common_wire      check the caller's service against the method table
pond_server      decode, map to pond core                    (R2)
pond_write       validate ids, dead keys, if-seq, idempotency (R0)
pond_wal         append, group commit, fsync                 (R0)
pond_server      encode the answer
```

Each line is one package, one responsibility, and one test suite, and the tiers say which lines a person must read.

### One world in one process

`sim_world` is a test-only package above every `$service_system`. It builds rancher, pond, permit, sun, barker, ranger, store, and any number of barns in one process, wired by the in-memory network, with one manual clock. End-to-end tests (put a route, upload a box, send a request, fire a timer, promote beta) run there in milliseconds, with fault injection (sever pond, stall ranger, kill a barn mid-request) as a test parameter. It is goober's "several systems in one process to simulate several machines," applied to the whole platform.

`sim_world` is also the dev loop: `homestead_dev` is a binary that runs the same composition over real sockets on one machine, with the dev mail provider and a local blob directory, so an agent can run the whole platform with one config file. Production never runs it; every service there is its own binary, deployed alone (see [common](../common.md#isolation)).

## Building and testing

- **Every package builds and tests alone**: `cmake --build --preset dev --target homestead_pond_wal_test`.
- **Every service builds alone**: its system, its main, and the closure of what they list.
- **CI runs, per pull request,** the tests of every changed package and of every package that depends on one, then `sim_world` when a `_proto`, `_client`, or `_system` changed.
- **Fuzzing** is a target per R0 parser (`common_wire` frames, `fork_parse`, `barn_abi` pointers, `permit_token`, `store_ssi`, `barker_bounce`), run nightly and on every change to that package.

## The invariants

- **Directory == namespace == library**, with a QUALITY.md and a tier.
- **Only declared dependencies link**, and only public headers are visible.
- **Services meet at the wire,** through a `_client` interface, except for pure library packages their owner publishes.
- **Every client has a fake, and both pass the client's contract tests.**
- **One composition root per binary.** Only `$service_system` wires packages together.
- **A pull request takes the tier of the highest package it touches**, and the review queue is sorted by tier.
- **R0 packages stay small enough to read in full.** Code that isn't boundary code moves out.
