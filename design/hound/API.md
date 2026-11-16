# Hound API
The C++ library barn links, the box interface it backs, the command line, and the case format
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Hound has no wire API: nothing calls it over the network. It has four surfaces, and each has one reader:

| surface | used by | section |
| --- | --- | --- |
| `hound@1`, the WIT interface | boxes, through barn | [The box interface](#the-box-interface) |
| The C++ library | barn's `barn_egress`, and the CLI | [The library](#the-library) |
| `hound`, the command line | people, agents, and CI validating an integration | [The command line](#the-command-line) |
| Cases: Postman collections with a `hound` extension | the CLI | [Cases](#cases) |

Every surface reports refusals with the same codes, listed in [Refusals](#refusals).

## The box interface

`hound@1` is a WIT interface barn implements with this library. A box imports it only if it declares `egress`.

```wit
package homestead:hound@1;

interface egress {
  record header { name: string, value: string }

  record request {
    method: string,                 // "GET", "POST", ...
    path: string,                   // "/api/v3/athlete/activities", never a URL
    query: list<tuple<string, string>>,
    headers: list<header>,
    body: list<u8>,
  }

  record response {
    status: u16,
    headers: list<header>,
    body: list<u8>,
    truncated: bool,                // the body passed the binding's cap
    ms: u32,
  }

  variant error {
    refused(string),                // a refusal code: "policy.path_outside_prefix"
    not-connected,                  // oauth2-user and the caller has no tokens for this name
    rate-limited(u32),              // retry after this many ms
    timeout,
    unreachable(string),            // DNS, TCP, or TLS failed; the reason, no secrets
  }

  send: func(name: string, req: request) -> result<response, error>;

  // Only when the binding attaches a spec. params is JSON: path, query, header, and body
  // parameters by their names in the spec ("body" for the request body).
  call: func(name: string, operation-id: string, params: string) -> result<response, error>;

  connected: func(name: string) -> bool;
}
```

- **A non-2xx answer is a response, not an error.** A 404 from Strava is the box's to read. `error` is only for what hound decided or couldn't do.
- **`name` is the binding's name**, the same string the box lists in its `egress` ruleset.
- **Every call counts against the request's deadline and fuel**, and the time spent waiting is reported in barn's routing record (`ms.egress`).

## The library

Namespaces follow the packages (see [PACKAGES](PACKAGES.md)). The headers barn includes are below, shortened.

### Request and response (`hound_request`)

```cpp
namespace homestead::hound::request {
struct Header { std::string name, value; };
struct Request {
  std::string method, path;
  std::vector<std::pair<std::string, std::string>> query;
  std::vector<Header> headers;
  std::string body;
};
struct Response {
  uint16_t status;
  std::vector<Header> headers;
  std::string body;
  bool truncated;
  std::chrono::milliseconds elapsed;
};
}
```

### Policy (`hound_policy`)

```cpp
namespace homestead::hound::policy {
struct Binding {                         // the route's egress entry, parsed by rancher_model
  std::string name;
  Origin origin;                         // scheme, host, port; https unless the deployment allows http for dev
  std::vector<std::string> prefixes;     // each starts and ends with '/'
  std::set<std::string> methods;
  CredentialRef credential;              // kind, secret name, header or query parameter name
  std::optional<SpecRef> spec;
  std::optional<Rate> rate;
  Caps caps;                             // body bytes in and out, time
};

struct Allowed { std::string normalized_path; };
struct Refused { std::string code; std::string detail; };   // detail names rules, never values

// Pure: no I/O, no clock. The only R0 decision in hound that isn't credentials.
std::variant<Allowed, Refused> check(const Binding&, const request::Request&);
std::variant<Allowed, Refused> check_redirect(const Binding&, std::string_view location);
}
```

### Secrets and tokens (`hound_auth`)

```cpp
namespace homestead::hound::auth {
class SecretSource {                     // cellar implements it in barn; the CLI reads the environment file
 public:
  virtual ~SecretSource() = default;
  virtual std::optional<Secret> get(std::string_view tenant, std::string_view name) = 0;
};

struct Tokens { Secret access; Secret refresh; int64_t expires_at_ms; std::string scope; };

class TokenStore {                       // barn implements it over pond's _credentials bucket
 public:
  virtual ~TokenStore() = default;
  virtual Future<std::optional<Versioned<Tokens>>> load(Tenant, UserId, std::string_view name) = 0;
  virtual Future<StoreResult> save(Tenant, UserId, std::string_view name, const Tokens&, Seq if_seq) = 0;
};

// Adds the credential to an allowed request. Refuses when the box set anything the credential owns.
Future<std::variant<request::Request, policy::Refused>>
attach(const policy::Binding&, request::Request, const Caller&, SecretSource&, TokenStore&, Clock&);
}
```

`Secret` is a type that can't be printed, logged, or copied into a `std::string` without an explicit `reveal()` call, and `reveal()` appears only in `hound_auth`.

### Specs (`hound_openapi`)

```cpp
namespace homestead::hound::openapi {
class Spec {
 public:
  static Result<Spec> parse(std::string_view json_or_yaml);       // OpenAPI 3.0 and 3.1
  const Operation* match(std::string_view method, std::string_view path) const;
  const Operation* operation(std::string_view operation_id) const;
  Result<request::Request> build(const Operation&, const boost::json::value& params) const;
  std::vector<Finding> validate_request(const Operation&, const request::Request&) const;
  std::vector<Finding> validate_response(const Operation&, const request::Response&) const;
};
}
```

### The client (`hound_client`)

```cpp
namespace homestead::hound::client {
struct Exchange {                        // what barn logs to ranger's egress stream
  std::string name, method, normalized_path, operation_id;
  uint16_t status; std::chrono::milliseconds elapsed;
  uint64_t bytes_out, bytes_in; bool truncated; int redirects;
  std::optional<std::string> refused;
};

class Client {
 public:
  Client(common::http::Client&, rate::Limiter&, Clock&, record::Recorder* = nullptr);
  // check -> spec (if any) -> attach -> rate -> send -> follow redirects within policy -> cap
  Future<std::variant<request::Response, Error>>
  send(const policy::Binding&, const std::optional<openapi::Spec>&, request::Request,
       const Caller&, auth::SecretSource&, auth::TokenStore&, Deadline, Exchange& out);
};
}
```

`Caller` is the tenant, the user id (0 when signed out), and the request id, which barn already has. Barn holds one `Client` per process; it is thread-safe.

## Refusals

| code | when | sent? |
| --- | --- | --- |
| `policy.unknown_name` | The box called a name its route didn't bind. | No |
| `policy.absolute_url` | The path carries a scheme, a host, or `//` at its start. | No |
| `policy.path_ambiguous` | Encoded slashes, dot segments, `//` inside, backslashes, control characters, or a second decoding would change it. | No |
| `policy.path_outside_prefix` | The normalized path is under no listed prefix. | No |
| `policy.method_not_allowed` | The method isn't listed. | No |
| `policy.header_reserved` | The box set a header hound owns (`Authorization`, `Cookie`, `Host`, `Proxy-*`, ...). | No |
| `policy.credential_param` | The box set the query parameter the credential uses. | No |
| `policy.redirect_outside` | A redirect left the origin or prefixes; the 3xx goes to the box. | The first request only |
| `auth.secret_missing` | The binding names a secret cellar doesn't have. | No |
| `auth.not_connected` | `oauth2-user` and the caller has no tokens. | No |
| `auth.refresh_failed` | The provider refused the refresh; the tokens are marked stale. | The refresh only |
| `spec.no_operation` | Strict mode, and no operation matches. | No |
| `spec.invalid_request` | Strict mode, and parameters or body fail the schema. | No |
| `rate.limited` | The bucket is empty, or the provider's headers say wait. | No |
| `cap.body_out` | The request body passes the cap. | No |

`hound_policy` checks in a fixed order (absolute URL, ambiguity, prefix, method, headers, credential parameter) and answers the first refusal, so a request breaking several rules always gets the same code. Each is reported as a problem named `hound.` plus the code, fingerprinted by route, box version, and name. `spec.response_mismatch` is reported but never refuses.

## The command line

```
hound run <collection> [--env <file>]... [--folder <name>]... [--allow-writes]
                       [--record <dir> | --replay <dir>] [--save-env] [--json] [--bail]
hound call --spec <openapi> --op <operationId> [--env <file>] [--param k=v]... [--binding <file>]
hound import <openapi> [--base-url <url>] > collection.json
hound check-policy --binding <file> --method <m> --path <p> [--header k:v]...
hound lint <collection>                 # parse, list skipped scripts and unknown auth types
```

| command | does |
| --- | --- |
| `run` | Runs every request in order (folders in order, depth first), applying captures, expectations, and policy checks. |
| `call` | One operation by its `operationId`, built from the spec, under an optional binding. |
| `import` | An OpenAPI spec to a Postman v2.1 collection, one folder per tag, with `hound` expectations for status and response schema. |
| `check-policy` | A request against a binding, offline: allowed with the normalized path, or the refusal code. Rancher's route checks and agents use it to test bindings without a network. |
| `lint` | What hound would ignore in a collection, before running it. |

- **Environment files** are Postman environments. Several `--env` files layer, later wins; a variable from the process environment named `HOUND_<NAME>` wins over files, so secrets can stay out of files in CI.
- **Secrets never print.** Variables marked `"type": "secret"` in the environment, and anything captured from a field named like a token, are shown as `[secret]` in output and recordings.
- **Captures stay in memory** unless `--save-env` writes them back to the last environment file, which matters for providers like Strava that rotate refresh tokens.
- **Writes are off by default.** An item with `"writes": true`, or any method but GET, HEAD, and OPTIONS without `"writes": false`, is skipped unless `--allow-writes`.

| exit | means |
| --- | --- |
| 0 | Everything ran passed (skips don't fail). |
| 1 | An expectation failed. |
| 2 | A policy case was not refused, or was refused with a different code. |
| 3 | A request couldn't be sent (DNS, TLS, timeout). |
| 4 | Usage or a file that doesn't parse. |

Output is one line per item: outcome, name, method and path, status, and time; then a rate line when the provider sent rate headers; then a summary. `--json` writes one JSON object per item instead, for agents.

## Cases

A case file is a **Postman Collection v2.1**, read as is: folders, requests, `{{variables}}` in URLs, headers and bodies, collection and folder variables, and the `apikey`, `bearer`, `basic`, and `oauth2` auth types (with a refresh token). Postman's JavaScript (`event` scripts) isn't run; each script is listed as skipped. `.http` files (JetBrains and VS Code REST Client) and Hurl files are candidates for later imports.

Hound's own instructions ride in a `hound` key, which Postman ignores on import and keeps on export:

```json
{
	"name": "recent activities",
	"request": { "method": "GET", "url": "{{base}}/athlete/activities?per_page=5" },
	"hound": {
		"expect": {
			"status": 200,
			"headers": { "content-type": { "op": "matches", "value": "^application/json" } },
			"json": [
				{ "path": "$", "op": "type", "value": "array" },
				{ "path": "$[0].id", "op": "exists" }
			],
			"schema": "response of getLoggedInAthleteActivities"
		},
		"capture": { "activity_id": "$[0].id" },
		"writes": false
	}
}
```

| key | on | means |
| --- | --- | --- |
| `egress` | the collection | Bindings, by name, exactly as a route writes them. Items that name one (`"binding": "strava"`) are run through `hound_policy` first, as barn would. |
| `spec` | the collection | A path to an OpenAPI file, for `schema` expectations and `call` items. |
| `binding` | an item | Run this item as a box would. When the URL's origin is the binding's, the rest of it is the path the box sends; any other origin is what a box sending a URL looks like, and is `policy.absolute_url`. Then the policy decides, the item's Postman auth is ignored, and the binding's credential is attached. |
| `expect.status` | an item | A number, or a list of numbers. |
| `expect.headers` | an item | Header name to a value, or to `{op, value}`. |
| `expect.json` | an item | JSONPath checks: `exists`, `absent`, `eq`, `ne`, `type` (`string`, `number`, `integer`, `boolean`, `array`, `object`, `null`), `gte`, `lte`, `matches` (a bounded regex), `len_gte`. |
| `expect.schema` | an item | `response of <operationId>`: the body must match the spec's schema for that status. |
| `expect.refused` | an item | A refusal code. The item passes only if hound refuses it with that code and sends nothing. |
| `capture` | an item | Variable name to JSONPath; the value is available to later items as `{{name}}`. |
| `writes` | an item or folder | True if the item changes the world. Skipped without `--allow-writes`. |
| `secret` | a capture | `{"var": "$.access_token", "secret": true}` keeps it out of output and recordings. |

The examples for HEVY and Strava are in [cases](cases/README.md).

## The invariants

- **One set of refusal codes** for boxes, the library, problems, and the CLI.
- **A non-2xx answer is the box's; only hound's own decisions are errors.**
- **`hound_policy::check` is pure**, so the same function decides in barn, in rancher's checks, and in `check-policy`.
- **A `Secret` is revealed in one package.** Nothing else can print one.
- **Cases are Postman collections.** Anything hound adds lives under `hound` and is ignored by Postman.
- **Writes don't run by accident.** Off unless asked for.
