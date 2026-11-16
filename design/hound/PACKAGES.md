# Hound packages
How hound is cut, where the R0 lines are, and how barn wires it in
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Hound has two places where a bug is a breach: **deciding whether a request may go** (a path that escapes its prefix reaches an endpoint nobody granted) and **attaching a credential** (a token sent to the wrong place, or exposed to the box). Those are `hound_policy` and `hound_auth`, both R0, both kept small. `hound_policy` is a pure function over a binding and a request, so it can be read in full and fuzzed without a network; `hound_auth` is the only package that can reveal a `Secret`.

Everything else is pushed out: HTTP plumbing to `hound_client`, OpenAPI's size to `hound_openapi`, rates to `hound_rate`, and the developer tooling (Postman, cases, recordings, the fake upstream, the CLI) to R2 and R3, where a bug costs a confusing test run, not a leak.

Hound is a library in homestead's sense (see [packages](../common/PACKAGES.md#library-packages)), but not a pure one like fork: `hound_client` does I/O. So barn links hound's packages as its own core packages, owned by the hound team, with `hound_policy` and `hound_auth` reviewed at R0 wherever they're linked.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `hound_request` | `homestead::hound::request` | Request and response as plain data; header names case-insensitive | `common_core` | R1 |
| `hound_policy` | `homestead::hound::policy` | `Binding`; path normalization; origin, prefix, method, and header rules; redirect checks. Pure. | `hound_request`, `common_core` | R0 |
| `hound_auth` | `homestead::hound::auth` | `Secret`, `SecretSource`, `TokenStore`, credential attachment for every kind, OAuth client credentials, user token refresh with compare-and-set, the connect flow's code exchange | `hound_policy`, `common_crypto`, `common_http` | R0 |
| `hound_openapi` | `homestead::hound::openapi` | OpenAPI 3.0 and 3.1: parse (JSON and YAML), `$ref` resolution with depth caps, operation matching, building requests from `operationId`, request and response validation | `hound_request`, `common_json` | R1 |
| `hound_rate` | `homestead::hound::rate` | Token buckets per binding and tenant, shares across barns, tightening from `Retry-After` and `X-RateLimit-*` | `common_core` | R1 |
| `hound_client` | `homestead::hound::client` | The pipeline: check, validate, attach, rate, send, redirect within policy, cap, `Exchange` | all of the above, `common_http` | R1 |
| `hound_record` | `homestead::hound::record` | Record and replay exchanges, with secrets replaced by placeholders; matching for shadows and `route_try` | `hound_request`, `common_json` | R2 |
| `hound_postman` | `homestead::hound::postman` | Postman Collection v2.1 and environment import, the `hound` extension, export from an OpenAPI spec | `hound_request`, `hound_openapi`, `common_json` | R2 |
| `hound_cases` | `homestead::hound::cases` | The case runner: variables, captures, expectations, JSONPath subset, writes gating, output | `hound_client`, `hound_postman`, `hound_record` | R2 |
| `hound_fake` | `homestead::hound::fake` | A scriptable fake upstream on the in-memory network: routes, rate headers, redirects, OAuth token endpoints | `common_http`, `common_test` | R2 |
| `hound_main` | | The `hound` binary | `hound_cases`, `hound_openapi` | R3 |

Third-party: a YAML parser for specs (yaml-cpp), wrapped by `hound_openapi` only. Everything else is already in `common_*`.

## Dependencies

```
                         hound_main
                             |
                        hound_cases ------------------+
                        /     |     \                 |
              hound_postman   |    hound_record       |
                    |         |                       |
                    |    hound_client  <-- barn_egress (barn)
                    |   /   |    |    \
        hound_openapi   |  hound_rate  hound_auth  --> common_crypto, common_http
                 \      |              /
                  \  hound_policy  ---+
                   \     |
                  hound_request --> common_core

  hound_fake: tests only, used by hound_client, hound_cases, and barn's tests
```

## Quality seeds

### hound_policy (R0)

- **Promise.** A request is allowed only if its method is listed, its path is unambiguous and, once normalized, under a listed prefix on a segment boundary, and it sets no reserved header or credential parameter. A redirect is allowed only within the same rules.
- **Tests.** A table of paths with verdicts: every encoding of `..` and `.`, `%2f`, `%252f` (double encoding), `//`, backslashes, null bytes, Unicode lookalikes, trailing-dot hosts, uppercase percent escapes, prefix boundary cases (`/api/v3` vs `/api/v3x`). Absolute URLs in every form. Reserved headers in every case. Fuzzed: `check` never allows a path that a reference normalizer (RFC 3986 § 5.2.4) places outside every prefix. Property: allowed paths round-trip through normalization unchanged.
- **Won't.** Do I/O, read a clock, repair a path, or know what a credential's value is.

### hound_auth (R0)

- **Promise.** A credential is attached only to a request `hound_policy` allowed, only for the binding's origin, in exactly the place the binding names; the box's own values in that place are refused, never merged. A `Secret` leaves this package only inside a request being sent. A refreshed token is saved before it is used again, and a lost compare-and-set uses the winner's token.
- **Tests.** Every credential kind against `hound_fake`'s token endpoints; refresh on expiry and on 401, once; concurrent refreshes; rotated refresh tokens (Strava's behavior); a refused refresh marks tokens stale; a redirect never carries the credential across origins; `Secret` can't be streamed or formatted (compile-time tests).
- **Won't.** Store anything itself, choose a secret name, or decide policy.

### hound_openapi (R1)

- **Promise.** Matches the operation a spec would match, builds the request the spec describes, and reports schema findings with JSON pointers.
- **Tests.** The OpenAPI 3.0 and 3.1 test suites' petstore variants; path templates with overlapping literals; `$ref` cycles refused; the Strava and HEVY specs (or what exists of them) round-tripped.
- **Won't.** Generate code, fetch remote `$ref`s, or allow anything `hound_policy` refused.

### hound_rate (R1)

- **Promise.** A binding's rate is never exceeded by the deployment as a whole by more than one barn's share, and the provider's headers only ever make it stricter.
- **Tests.** A manual clock; shares rebalancing as barns join and leave; `Retry-After` in seconds and as a date; Strava's two-window `X-RateLimit-Usage`.
- **Won't.** Queue. An empty bucket answers at once.

### hound_client (R1)

- **Promise.** Runs the pipeline in order, never sends what a step refused, caps bytes and time, verifies TLS, and fills `Exchange` for every call, refused ones included.
- **Tests.** Against `hound_fake`: redirects in and out of policy, slow bodies, oversized bodies, TLS failures, connection reuse, cancellation at the deadline.
- **Won't.** Decide policy, hold a secret past one call, or log a body.

### hound_request (R1)

- **Promise.** Plain data, with case-insensitive header lookup and no hidden normalization.
- **Tests.** Header handling; copies and moves.
- **Won't.** Parse URLs. That is `hound_policy`'s job, once.

## How it's wired

Hound has no system or main of its own in production; barn wires it.

- **`barn_system` builds** one `hound::rate::Limiter`, one `hound::client::Client` over barn's `common_http` client, cellar's `SecretSource`, and `barn_egress`'s `TokenStore` over `pond_client`. It loads specs from rancher's config log (by hash, cached like modules).
- **`barn_egress`** is barn's package that implements `hound@1` for boxes: it maps a box call to `Client::send` with the route's binding and the caller, adds `Exchange` to the routing record and to ranger's `egress` stream, and turns refusals into problems. It also serves `/_hound/connect/{name}` and `/_hound/disconnect/{name}`.
- **Threads.** Calls run on barn's network threads, with the box's call suspended or blocked as barn's box pool decides; hound owns no threads.
- **Rancher** links `hound_policy` (pure) to check that a route's bindings parse, that prefixes are well formed, and that every name is in the box's egress ruleset, at deploy.
- **The CLI** builds the same `Client` over a real network, an environment-file `SecretSource`, and an in-memory `TokenStore` seeded from the environment.

## Review queue

| tier | packages |
| --- | --- |
| R0 | `hound_policy`, `hound_auth` |
| R1 | `hound_client`, `hound_openapi`, `hound_rate`, `hound_request` |
| R2 | `hound_record`, `hound_postman`, `hound_cases`, `hound_fake` |
| R3 | `hound_main` |

Build order, so agents can work in parallel:

1. `hound_request`, then `hound_policy` and `hound_fake` in parallel.
2. `hound_auth`, `hound_rate`, and `hound_openapi` in parallel.
3. `hound_client`.
4. `hound_postman` and `hound_record` in parallel, then `hound_cases` and `hound_main`. The HEVY and Strava cases become the CLI's first acceptance tests.
5. `barn_egress` in barn, against `hound_fake`.

## The invariants

- **Two R0 packages, both small.** `hound_policy` is pure; `hound_auth` is the only place a secret is revealed.
- **Barn and the CLI link the same packages.** No test-only fork of the pipeline.
- **Hound owns no threads, no storage, and no wire API.** Barn supplies the network, the token store, and the secrets.
- **Every package has a QUALITY.md and builds alone**, as [packages](../common/PACKAGES.md) asks.
