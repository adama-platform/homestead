# Porch API
The tools contract services implement, the MCP and HTTP surface barn serves, and the C++ library between them
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Porch has three contracts, and each has a different reader:

1. **The tools contract**: `$service.tools.list` and `$service.tools.call` on the wire. Rancher and store implement it; porch is its only caller. Its messages are one schema, in the `porch_contract` library, which both owners and porch link, so a tool is described the same way everywhere.
2. **The public surface**: MCP at `/mcp` and `/mcp/{role}`, and HTTP JSON at `/_ops/api/{tool}`, on every `ops.` domain. Agents and the person's panel use it.
3. **The library**: the C++ interface barn calls with a request and a verified caller.

## The surface

| method or path | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `rancher.tools.list`, `store.tools.list` | call | barn (porch) | no | The service's catalog at a version |
| `rancher.tools.call`, `store.tools.call` | call | barn (porch) | when the tool does | The tool's result, or a status with a problem |
| `POST /mcp`, `POST /mcp/{role}` on `ops.` | HTTP (MCP, streamable) | agents | when a tool does | JSON-RPC answers, or an SSE stream of them |
| `GET /mcp`, `GET /mcp/{role}` on `ops.` | HTTP (SSE) | agents | no | Server notifications: `tools/list_changed` |
| `DELETE /mcp/...` | HTTP | agents | no | Ends the MCP session |
| `GET /_ops/api` | HTTP | panel, scripts | no | The visible catalog as JSON |
| `POST /_ops/api/{tool}` | HTTP | panel, scripts | when the tool does | The tool's result as JSON |
| `GET /.well-known/oauth-protected-resource/mcp/{role}` | HTTP | agents | no | Protected resource metadata naming permit and the scope `ops/{role}` |

Rancher proxies the tools of sun (`dead_*`, schedules), barker (`mail_*`), and ranger (`log_*`) in its own catalog, so those services keep their only caller rancher, and porch talks to two owners.

## The tools contract

### A tool

```json
{
	"name": "page_patch",
	"owner": "store",
	"summary": "Replace one exact string in a page, guarded by its ETag.",
	"input_schema": { "type": "object", "required": ["url", "old", "new", "if_etag"],
	                  "properties": { "url": { "type": "string" }, "old": { "type": "string" },
	                                  "new": { "type": "string" }, "if_etag": { "type": "string" },
	                                  "dry_run": { "type": "boolean" } } },
	"roles": ["content"],
	"changes": true
}
```

| field | meaning |
| --- | --- |
| name | Unique across the deployment, `[a-z][a-z0-9_]*`. Porch refuses a catalog that repeats a name another owner has, and reports `porch.tool_name_clash`. |
| summary | One line, for `tools/list` and `guide`. |
| input_schema | JSON Schema for the arguments. Porch doesn't validate against it; the owner does. |
| roles | The roles the tool serves, from the role tree. A tool with no roles is visible only to `admin`. |
| changes | Whether the tool can change anything. Shown to MCP clients as `readOnlyHint: false`, and required to accept `dry_run`. |

### `$service.tools.list`

```json
{ "tenant": "bobs_plumbing", "have": "rancher:8815" }
```

```json
{
	"version": "rancher:8821",
	"tools": [ { "name": "route_put", "roles": ["deploy"], "changes": true, "...": "..." } ],
	"roles": { "admin": { "contains": ["*"], "delegates": true, "about": "..." },
	           "content": { "contains": [], "delegates": false, "about": "Pages, layouts, ..." } },
	"news": [ { "build": "2026-11-16.2", "roles": ["deploy"], "text": "route_try takes `as` for any grant set." } ]
}
```

- **`have`** is the version porch holds; an owner whose catalog hasn't moved answers `{ "version": same, "unchanged": true }`.
- **Versions** are opaque strings an owner chooses. Rancher's is its config seq, so a role tree change or a tool change moves it; store's moves with its build.
- **Only rancher answers `roles`.** The role tree is configuration, and rancher holds it. Store answers tools and news only.
- **Catalogs don't vary by tenant** today; `tenant` is there so one could (a tenant without mail routes needn't see mail tools), and porch caches per owner, per tenant when an owner says so.

### `$service.tools.call`

```json
{
	"tool": "page_patch",
	"args": { "url": "/about", "old": "since 1987", "new": "since 1986", "if_etag": "\"b7e1\"" },
	"caller": {
		"realm": "bobs_plumbing", "tenant": "bobs_plumbing",
		"subject": "4294967100", "kind": "agent",
		"grants": ["ops/content"], "role_path": "content",
		"token": "4294967901", "lineage": ["4294967555", "4294967890"]
	},
	"catalog": "store:2026-11-16.2"
}
```

```json
{ "result": { "etag": "\"c402\"", "changed": ["/about"], "findings": [] }, "widens": [] }
```

| field | meaning |
| --- | --- |
| args | The arguments exactly as the client sent them, including `dry_run`, `if_seq`, and `if_etag`. |
| caller | Who is calling, as barn verified it. `grants` is the token's or subject's grants; `role_path` is the role in the URL, or null for `/mcp`; `lineage` is the token's ancestor sessions, root first. |
| catalog | The version porch routed by. An owner whose tool moved answers `invalid` with problem `porch.catalog_stale`, and porch refetches. |
| result | The tool's answer, JSON. |
| widens | Sentences for every widening a changing tool made (or would make, on a dry run). |

The owner re-checks the caller with `porch_roles` before it does anything: the tool's roles, intersected with the role path when there is one, must include one the caller holds. A call that fails the re-check is `refused` with problem `porch.role_disagreement`, because it means porch's filter and the owner's disagree.

| code | when | MCP | HTTP |
| --- | --- | --- | --- |
| ok | | result | 200 |
| invalid | Arguments don't match the schema; stale catalog | tool error | 400 |
| refused | The caller may not; a person-only tool called by an agent | tool error | 403 |
| not_found | No such tool, or the thing it names | tool error | 404 |
| conflict | `if_seq` or `if_etag` moved | tool error | 409 |
| unavailable, overloaded | The owner is down or busy | tool error with `retry_after_ms` | 503, 429 |
| deadline | The call ran out of time; changing tools may have applied | tool error | 504 |

Tool errors are MCP results with `isError: true`, whose text starts with the problem code and hint, so an agent reads what to fix. JSON-RPC errors are only for protocol mistakes.

### Person-only tools

Some acts are a person's by design: approving a promotion, lifting a mail block, dropping data. Their descriptors carry `"person": true`. Porch lists them to agents (so an agent knows they exist and can ask), and the owner refuses them unless `caller.kind` is `user` or `operator` with a session made by a person (not `agent`). The refusal says who can do it.

## MCP

Porch speaks MCP over streamable HTTP, one endpoint per role path.

| JSON-RPC method | porch does |
| --- | --- |
| `initialize` | Answers server info (`homestead-porch`, the build), `capabilities.tools.listChanged: true`, and instructions: the role's `guide` in one paragraph. Issues an `Mcp-Session-Id`. |
| `tools/list` | The visible tools: `guide` and `whats_new` first, then the role's tools in catalog order. Paged by cursor at 100. |
| `tools/call` | Built-ins answer in porch; everything else goes to the owner's `tools.call`. |
| `ping` | Answers. |
| `notifications/tools/list_changed` | Sent on the session's SSE stream when an owner's catalog version moves and the visible set changed. |

- **Auth.** Every request carries `Authorization: Bearer <agent token>`. Without one, or with one that fails, porch answers 401 with `WWW-Authenticate: Bearer resource_metadata="https://ops.bobsplumbing.com/.well-known/oauth-protected-resource/mcp/content"`. That metadata names `https://ops.bobsplumbing.com/_permit/` as the authorization server and `ops/content` as the scope, so an MCP client connecting to `/mcp/content` asks permit's consent screen for exactly that role.
- **A role the caller doesn't hold** answers 403 with `WWW-Authenticate: Bearer error="insufficient_scope", scope="ops/content"`.
- **Sessions** live in the barn that issued them, for 30 minutes idle. A session reaching another barn (no stickiness) is refused with 404, and MCP clients reinitialize, as the spec expects. Nothing about a session matters for correctness.
- **Tool names are plain** (`page_patch`), never prefixed by role or owner, so a tool has one name in every role and in the HTTP form.

```
POST /mcp/content
Authorization: Bearer v1.k7.eyJ2IjoxLCJpZCI6...
Mcp-Session-Id: 7f1c...
{"jsonrpc":"2.0","id":4,"method":"tools/call",
 "params":{"name":"page_patch","arguments":{"url":"/about","old":"since 1987","new":"since 1986","if_etag":"\"b7e1\""}}}
```

```
{"jsonrpc":"2.0","id":4,"result":{"content":[{"type":"text","text":"{\"etag\":\"\\\"c402\\\"\",\"changed\":[\"/about\"],\"findings\":[]}"}],
 "structuredContent":{"etag":"\"c402\"","changed":["/about"],"findings":[]},"isError":false}}
```

## HTTP JSON

For the panel and for scripts: the same tools, the same filter, without MCP.

- `GET /_ops/api` lists the visible tools; `?role=content` narrows it like the MCP path.
- `POST /_ops/api/{tool}` takes the arguments as the body and answers the result, with the status table above. A role can be named with `?role=content`, and narrows exactly as the path does.
- **Auth** is a bearer token or the person's `__Host-session` cookie. Cookie requests must carry an `Origin` equal to the `ops.` domain, so a page elsewhere can't drive a person's session.
- **The answer** is `{ "result": ..., "widens": [...] }`, or `{ "problem", "message", "hint" }` on failure.

## Built-ins

| tool | args | answers |
| --- | --- | --- |
| `guide` | `{}` | The role's `about` from the role tree, its tools with summaries, and which tools are person-only. On `/mcp`, every role the caller holds. |
| `whats_new` | `{ "since"?: "build" }` | `news` entries from every owner whose roles intersect the caller's view, newest first. |

## The library

```cpp
namespace homestead::porch {

struct Caller {                       // built by barn_caller from a verified token
  std::string realm, tenant;
  Id subject; permit::token::Kind kind;
  std::vector<std::string> grants;    // token or live grants
  Id token; std::vector<Id> lineage;
  bool person;                        // a person's session, not an agent
};

class ToolSource {                    // rancher, store, and on tenant domains barn's route table
 public:
  virtual ~ToolSource() = default;
  virtual Future<Catalog> list(const std::string& tenant, const std::string& have, Deadline) = 0;
  virtual Future<CallAnswer> call(const CallRequest&, Deadline) = 0;
};

class Porch {
 public:
  Porch(Config, std::vector<ToolSource*>, Clock&, log::Logger&);
  // barn hands every request under /mcp or /_ops/api on an ops. domain here
  Future<http::Response> serve(const http::Request&, const Caller&, std::string_view request_id);
  // the config seq moved: catalogs older than this are refetched on next use
  void invalidate(std::string_view owner, std::string_view version);
};
}
```

- **Barn builds one `Porch`** with sources wrapping `rancher_client` and `store_client`, and a second, for tenant domains, with a source over its own route table.
- **Porch never verifies a token.** Barn has already, with `permit_token`; porch receives a `Caller`.
- **Catalogs are cached per owner and version.** Rancher's `catalog.put` config event calls `invalidate`; a stale call answer refetches too, so a missed event costs one round trip.

## Problems

| code | when |
| --- | --- |
| `porch.tool_name_clash` | Two owners publish the same tool name; the later one's tool is hidden. |
| `porch.role_unknown` | A tool names a role the tree doesn't have; it is visible only to `admin`. |
| `porch.role_disagreement` | An owner refused a call porch allowed. |
| `porch.catalog_stale` | An owner refused a call routed by an old catalog, more than once in a minute. |
| `porch.owner_unavailable` | An owner's `tools.list` failed; porch serves the last catalog it had and says how old. |

## The invariants

- **One descriptor, one name, everywhere.** A tool is the same in every role, in MCP, and in HTTP.
- **The role path only narrows.** Visibility is the path role intersected with the caller's grants.
- **The owner re-checks every call** with the same `porch_roles` library porch uses.
- **Porch passes arguments and answers through untouched,** and puts widenings first.
- **Person-only tools are listed to agents and refused to them.**
- **Every call is traced with its lineage, never its arguments.**
