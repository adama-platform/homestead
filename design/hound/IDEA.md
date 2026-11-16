# Hound
A library barn uses to fetch from the world: the box builds the request, and the route decides where it may go
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober owned the network a box saw. Services were named in the manifest, typed by OpenAPI, and given credentials on the way out that the box never saw (human/030). Stability (human/330) wanted products to outlive their vendors, and crawling (human/800) wanted the host to be a polite robot. The instinct was right; the cost was that every service call went through generated code and a manifest section, so calling a new API meant changing the host's view of the world.

Hound keeps the instinct and drops the generator. **A box has full control of the request it sends: method, path, query, headers, and body. The only permission is where it may go: an origin and the URI prefixes under it, bound by the route and checked at runtime.** Hound is the library that enforces that, attaches the credential the box never sees, speaks OpenAPI when a spec is attached, and keeps the calls honest: capped, rate limited, logged, and replayable.

Hound is a library, not a service. Barn links it and runs it in the request, so an outbound call costs one hop, not two. The same library runs in the `hound` command line tool, so the exact code that serves boxes can be pointed at HEVY or Strava from a laptop, with a Postman collection, and checked.

## Where hound sits

```
   box  --hound@1 send/call-->  barn (barn_egress)
                                  |  the route's egress binding, the tenant, the user
                                  v
                       +---------------------------+
                       | hound (library, in barn)  |
                       |  policy: origin, prefix,  |
                       |    method, headers        |
                       |  auth: credential from    |
                       |    cellar / _credentials  |
                       |  openapi: match, validate |
                       |  rate, caps, redirects    |
                       +---------------------------+
                                  |  HTTPS
                                  v
                    api.hevyapp.com, www.strava.com, ...

   hound CLI (laptop, CI) -- the same library, with Postman collections and environments
```

| hound talks to | for | how |
| --- | --- | --- |
| The world | The box's request | HTTPS through `common_http`'s client, certificates verified. |
| cellar | App secrets (API keys, OAuth client secrets) | The `SecretSource` interface; barn passes cellar's implementation. |
| pond | Per-user OAuth tokens, sealed | Through barn: hound asks barn's `TokenStore` for `_credentials/{user}`, and hands back refreshed tokens for barn to write. |
| ranger | One `egress` line per exchange | Through barn's log, never directly. |

## The permission

The rule follows barn's: **names at deploy, bindings at runtime.**

| layer | where | says | checked |
| --- | --- | --- | --- |
| The box's egress ruleset | The box, in rancher: `"egress": ["strava"]` | The names this module may ever call. | At deploy: a route can't bind a name the box doesn't list. |
| The route's binding | The route, in rancher | What the name means for this route: origin, prefixes, methods, credential, spec, rate. | At runtime, on every call. |

```json
"egress": {
	"strava": {
		"origin": "https://www.strava.com",
		"prefixes": ["/api/v3/"],
		"methods": ["GET"],
		"credential": { "kind": "oauth2-user", "secret": "tenant/strava_app" },
		"spec": "sha256:5e1c...",
		"rate": "100/15m"
	}
}
```

- **The origin is fixed by the name.** The box sends a path, never a URL. An absolute URL, a scheme, a host, or a port in what the box sends is refused.
- **The path must sit under a listed prefix after normalization.** Hound percent-decodes once, refuses encoded slashes and dot segments (`..`, `.`, `%2e%2e`), collapses nothing (`//` is refused rather than repaired), and only then compares against each prefix on a segment boundary, so `/api/v3/` never matches `/api/v3x/`.
- **The method must be listed.** A route that only reads lists `GET`.
- **Some headers belong to hound.** The box can't set `Authorization`, `Cookie`, `Host`, `Proxy-*`, `Forwarded`, `X-Forwarded-*`, `Content-Length`, `Transfer-Encoding`, `Connection`, or the header or query parameter the credential uses. Everything else is the box's.
- **Redirects stay inside.** A redirect is followed only when its target is the same origin and under a listed prefix and method rules allow it; otherwise the 3xx is handed to the box as a response.
- **Answers are capped.** A response body is cut at the binding's limit (default 4 MiB), the whole exchange at the route's deadline less what the box already used (default 5 s), and both are reported.

A refusal is decided locally, before a byte leaves the machine, answers the box an error it can read, and is a problem (`hound.policy.path_outside_prefix`, and so on: the refusal code under `hound.`) naming the route, the box version, and the name.

## Credentials

The box never holds a secret, so it can't leak one, log one, or send one somewhere else.

| kind | hound attaches | from |
| --- | --- | --- |
| none | Nothing | |
| api-key | A header (`api-key: ...`) or a query parameter, as the binding names | cellar, by secret name |
| bearer | `Authorization: Bearer ...` | cellar |
| basic | `Authorization: Basic ...` | cellar |
| oauth2-client | A token from the client credentials grant, cached until it expires | cellar holds the client id and secret |
| oauth2-user | The calling user's access token, refreshed when it expires | the user's sealed tokens in pond's reserved `_credentials` bucket; the app's client id and secret in cellar |

Secret names are scoped: `tenant/strava_app` is the tenant's own, `system/...` the platform's. A route can name only its tenant's secrets and the system secrets an operator shared with it.

### Per-user OAuth

Strava, Google Calendar, and most consumer APIs act for one person, so the token belongs to the person, not the tenant.

1. **Connect.** A person visits `/_hound/connect/strava` on the tenant's domain. Barn runs the authorization code flow (with PKCE when the provider supports it) using the app's client id from cellar, under the binding's scopes, and on the callback stores the tokens.
2. **Store.** Tokens are sealed by cellar and written to `_credentials/{user_id}` in the tenant's pond, under the name, as a system write annotated with the user. Boxes can't read `_` buckets; deleting the user deletes the tokens, like any of their data.
3. **Use.** A box on a route bound `oauth2-user` calls `send`; hound loads the sealed token through barn, attaches it, and if it has expired or the provider answers 401, refreshes once and retries once.
4. **Refresh.** Providers like Strava rotate refresh tokens, so the new pair is written back to pond with compare-and-set; a concurrent refresh that loses re-reads and uses the winner's token.
5. **Disconnect.** `/_hound/disconnect/strava` revokes at the provider when it can and deletes the tokens.

A box learns whether the caller is connected (`connected(name) -> bool`) and nothing more.

## OpenAPI

OpenAPI is the prime candidate for describing a service, because it already exists for most of them and every agent reads it. A spec is optional, attached to a binding by hash and uploaded through rancher like a box.

| with a spec, hound | so |
| --- | --- |
| Matches every `send` to an operation by method and path template | Requests to undocumented endpoints can be refused (`strict`) or only reported (`warn`, the default). |
| Validates parameters and the request body against the schema | A box's typo is an error before the provider sees it. |
| Offers `call(name, operation_id, params)` | A box can call `getLoggedInAthlete` without building a path. Hound fills the path, query, and body from `params` by the spec. |
| Can validate responses (off by default) | A provider that changed its contract shows up as a problem, `hound.spec.response_mismatch`, before a box breaks on it. |
| Lets the CLI generate a Postman collection | The test cases start from the spec. |

The policy still rules: an operation whose path falls outside the binding's prefixes is refused even if the spec has it.

## Rates and manners

- **Rates per upstream, per tenant.** A binding's `rate` (`100/15m`) is a token bucket shared by every barn through the route's tenant: each barn takes a share, and rancher's view of barns rebalances shares. A call over the rate fails at once with `rate.limited` rather than queueing inside a request.
- **The provider's word wins.** `Retry-After` and `X-RateLimit-*` headers (Strava's `X-RateLimit-Limit` and `X-RateLimit-Usage`) tighten the bucket until the window resets.
- **Identification.** Every request carries `User-Agent: homestead-hound/1 (+https://$primary_domain)`; a box may add to it, not replace it.
- **Crawling** (goober's good robot) is a mode a binding can turn on: robots.txt honored and cached, one request per second per host by default, conditional GETs. It waits for a real crawler product to be designed in full.

## Record and replay

Every exchange can be recorded as a pair of request and response with secrets and tokens replaced by placeholders. Records serve three things:

| use | how |
| --- | --- |
| Shadows | A shadow version's calls are answered from the live version's recording when the request matches, and never sent otherwise (unless the binding is declared effect-free, as rancher's shadow rule says). |
| `route_try` | A tried route can run against recorded answers, so trying a route never posts to Strava. |
| Tests | The CLI records a run (`--record dir`) and replays it (`--replay dir`), so a case suite runs in CI without credentials or network. |

## The command line

`hound` is the library with a front end, for people and agents validating an integration before a box uses it:

```
$ hound run cases/strava.postman.json --env strava.env.json --folder read
  ok    refresh token                 POST /oauth/token            200   212 ms
  ok    who am I                      GET  /api/v3/athlete         200    88 ms
  ok    recent activities             GET  /api/v3/athlete/act...  200   141 ms
  ok    one activity                  GET  /api/v3/activities/...  200   120 ms
  ok    policy: path outside prefix   refused policy.path_outside_prefix (not sent)
  rate  X-RateLimit-Usage 4,180 of 200,2000
5 passed, 0 failed, 1 skipped (writes; use --allow-writes)
```

It reads Postman collections and environments directly, so a collection someone already has works as is, and hound's own assertions ride in an extension key Postman ignores (see [API](API.md#cases)). The examples for HEVY and Strava are in [cases](cases/README.md).

## What hound is not

- **Not a service.** It runs inside barn and inside the CLI. Nothing calls hound over the wire.
- **Not a proxy for anything a box can name.** Only bindings a route declares, and only under their prefixes.
- **Not a code generator.** OpenAPI checks and fills requests; it doesn't generate box code.
- **Not mail.** Barker sends mail; hound speaks HTTP.
- **Not a place for secrets.** Cellar holds them; hound borrows them for one request.

## The invariants

- **The box controls the request; the route controls where it goes.** Origin by name, path under a prefix, method from a list.
- **Names at deploy, bindings at runtime.** A route can bind only names the box's egress ruleset lists.
- **Every refusal happens before a byte is sent.** And every refusal is a problem.
- **The box never sees a credential.** Hound attaches it and strips anything the box set in its place.
- **Paths are normalized once, then compared on segment boundaries.** Ambiguous paths are refused, never repaired.
- **Redirects never leave the binding.**
- **Every exchange is capped, rate limited, and logged** as keys and timings, never bodies or tokens.
- **Per-user tokens belong to the user.** Sealed in pond under their id, deleted with them.
- **The CLI and barn run the same library.** A case that passes on a laptop is a statement about production.
