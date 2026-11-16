# Store API
The tools agents write a site with, and the compiled releases barn serves
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Store has two audiences. **Agents and operators** write a tenant's site through tools: read its shape in one call, put and patch pages, components, layouts, and queries, render and lint without storing, stage a change across many files, release, and propose a promotion. **Barn** serves what store compiled: it fetches a release by hash, once, and executes its page plans per request. Store is never on the request path for a visitor, so its wire surface is small: the release fetch, the tools contract (`store.tools.list` and `store.tools.call`, see [porch](../porch/API.md#the-tools-contract)), which porch in barn serves as MCP at `/mcp/content` and the other role paths and as HTTP at `/_ops/api/{tool}`, and `store.http.serve`, which now carries only upload tickets.

The contract between store and barn is the release format, defined once in the `store_plan` library that both link. It is the most important part of this doc, because it is what makes "store compiles, barn serves" true.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `store.release.fetch` | call (chunked answer) | barn | no | A compiled release's bytes, by hash |
| `store.tools.list` | call | barn (porch) | no | Store's tool catalog at a version, each tool tagged with its roles |
| `store.tools.call` | call | barn (porch) | when the tool does | The tool's result; the caller is re-checked with `porch_roles` |
| `store.http.serve` | call | barn | yes | An `HttpResponse` for an upload ticket under `/_ops/site/upload/`, and nothing else |

Store's outward calls:

| to | method | for |
| --- | --- | --- |
| rancher | `rancher.config.subscribe` (kinds `table.*`, `release.*`, `tenant.*`) | Table versions to check queries against; which release is live where |
| rancher | `rancher.tables.get` | Columns and types for a table version, on demand |
| rancher | `rancher.release.submit` | A release's routes, grants, and named queries, which put it live on beta |
| rancher | `rancher.release.propose` | A promotion request a person approves |
| rancher | `rancher.problems.report` | Lint findings, guard refusals |
| ranger | `ranger.ingest` | Writes, releases, tool calls |

## Serving releases

### `store.release.fetch`

```json
{ "tenant": "bobs_plumbing", "release": "sha256:aa01c3...", "offset": 0 }
```

The answer is a stream of chunks of at most 1 MiB, `{ "offset": 0, "bytes": "...", "last": false }`, then a `Status`. The bytes are one release file in the `store_plan` format; barn checks the hash, may resume from an offset, and caches the release forever. `not_found` means no such release; `refused` means the release isn't the named tenant's.

Barn learns a release exists from rancher's `release.put` event, never from store, so a release barn serves is always one rancher checked.

### The release format

A release is one FlatBuffer, defined in `store_plan`:

```
namespace homestead.store.plan;

table Release {
  hash: string;              // sha256 of the canonical bytes, its id
  tenant: string;
  kit: string;               // the pinned hash of the built-in kit and components (Q9.7)
  pages: [Page];             // sorted by url, for binary search
  redirects: [Redirect];     // /p.md -> /p, aliases, front matter redirects
  statics: [Static];         // css, js, fonts: url, content type, hash; bytes in blobs
  blobs: [Blob];             // content-addressed bytes, deduplicated
  components: [Component];   // compiled Mustache, for slots rendered per request
  queries: [string];         // the named queries this release uses, by name
}

table Page {
  url: string;
  kind: Kind;                // markdown, fragment, stitched, document
  requirements: [string];    // permit clauses, checked by barn
  per_viewer: bool;          // depends on user.*, signed_in, or a per-viewer route include
  etag: string;              // for pages with no slots, the hash of the bytes
  deadline_ms: uint16;       // default 250
  parts: [Part];             // in order
}

union Part { Segment, QuerySlot, RouteSlot, EchoSlot, IfSlot }

table Segment   { blob: uint32; }                                   // literal bytes
table QuerySlot { query: string; component: uint16; params: [Bind]; empty: uint32; fallback: uint32; timeout_ms: uint16; }
table RouteSlot { path: string; component: uint16; fallback: uint32; timeout_ms: uint16; }
table EchoSlot  { var: string; }                                    // page.*, site.*, user.name, request.path, query.*, now.*
table IfSlot    { predicate: string; arg: string; then: [Part]; else: [Part]; }
table Bind      { name: string; literal: string; from: string; }    // from: "query.city"
```

| part | barn does |
| --- | --- |
| Segment | Writes the bytes. Layout, nav with its active link, footer, head, literal components, and markdown are already in them. |
| QuerySlot | Runs the named query (through its cache) with the bound values as SQL parameters, renders each row through the component, writes `empty` for no rows and `fallback` on failure or timeout. |
| RouteSlot | Calls a read-only GET route on this tenant as the same caller, renders its JSON through the component. |
| EchoSlot | Writes one escaped value from the fixed namespace. |
| IfSlot | Evaluates `signed_in`, `grant`, or `query` (present), and writes one branch. |

- **A plain page is one Segment.** Barn serves it from memory with its ETag.
- **At most 16 slots per page,** and an `IfSlot`'s branches count toward it. A RouteSlot never targets a stitched page.
- **Every value is escaped by the renderer,** and component output is never scanned for directives.
- **The format only grows.** Fields are added, never changed, so an older barn serves a newer release's known parts and refuses (with a problem) a page whose parts it doesn't know.

## The site tools

Store implements the tools contract: `store.tools.list` publishes these tools with their roles, and `store.tools.call` runs one for a caller porch has already filtered. Porch serves them on the tenant's `ops.` domain as MCP (`/mcp`, and `/mcp/content`, `/mcp/deploy`, `/mcp/data` by role) and as HTTP (`/_ops/api/{tool}`). Store re-checks every call with `porch_roles` against the tool's roles, the caller's grants, and the role path, and refuses what porch shouldn't have sent. Every tool answers compactly; `verbose: true` adds full text.

### Every write

| field | in | meaning |
| --- | --- | --- |
| `if_etag` | request | Refuse with `conflict` unless the file is still at this ETag. Required for edits; absent on creates. |
| `dry_run` | request | Check, compile, and lint as if written; store nothing. |
| `etag` | answer | The file's new ETag. |
| `changed` | answer | The URLs whose compiled plans changed, so an agent knows what to look at. |
| `findings` | answer | Lint and guard findings, each `{ "rule", "level", "path", "at", "hint" }`. `error` findings refuse the write. |

### Tools

| tool | roles | request | answer |
| --- | --- | --- | --- |
| `site_guide` | content | `{}` | The store model in one page of text. |
| `site_map` | content, data, deploy | `{}` | Menus, layouts, components with parameter signatures, named queries with their params, and every page's URL, title, and kind. |
| `site_put`, `site_patch` | content | `{ "body" }` or `{ "old", "new" }`, with `if_etag` | `etag`, `changed`, `findings` |
| `page_put` | content | `{ "url", "kind", "body", "if_etag"? }` | Removes any other file answering the same URL. |
| `page_patch` | content | `{ "url", "old", "new", "if_etag" }` | Exact string replacement; `invalid` if `old` doesn't occur exactly once. |
| `page_get` | content | `{ "url" }` | `{ "url", "kind", "etag", "body" }` |
| `page_delete` | content | `{ "url", "if_etag" }` | `changed`, plus links that now break |
| `component_put` | content | `{ "name", "body", "if_etag"? }` | Refused if the front matter's params don't cover the template, or a page now misses a required param. |
| `layout_put` | content | `{ "name", "body", "if_etag"? }` | Refused unless `{{{body}}}` appears once. |
| `query_put` | content, data | `{ "name", "body", "if_etag"? }` | Checked with fork against the table versions; the verdict's refusal is the finding. |
| `render` | content | `{ "url", "as"?: "signed_out" \| { "grants": [...] }, "overlay"?: { "path": "body" } }` | `{ "status", "etag", "segments-changed", "slots": [ { "kind", "ms", "ok" } ], "findings" }`; HTML with `verbose`. |
| `lint` | content | `{ "paths"?, "overlay"? }` | Findings over the tree or over unsaved files |
| `stage` | content | `{ "stage"?, "files": { "path": "body" \| null } }` | `{ "stage": "4294971001", "findings" }`; staged files are layered, nothing is written. |
| `apply` | content | `{ "stage", "if_etags"? }` | Writes every staged file or none. |
| `attachment_put` | content | `{ "url": "https://..." }` or `{ "ticket": true }` | `{ "id", "path": "/attachment/12.jpg" }`, or an upload ticket: a one-time URL under `/_ops/site/upload/` |
| `attachment_list`, `attachment_delete` | content | `{}` / `{ "id" }` | Each attachment with the pages that use it; delete refuses one in use unless `force`. |
| `release` | deploy | `{ "note"? }` | `{ "release": "sha256:...", "config-seq" }`: snapshot, compile, guard, submit to rancher; beta serves it. |
| `diff` | content, deploy | `{ "from", "to" }` | Pages added, removed, and changed (text diffs), new origins, scripts, forms, queries, grants, route includes. |
| `promote` | deploy | `{ "release", "note"? }` | `{ "promotion", "url" }` from `rancher.release.propose`. A person approves it in rancher. |

`promote` is person-approved in rancher: the tool proposes, and the descriptor marks the approval (in rancher's catalog) `person: true`. `release` is `deploy` because it puts a snapshot live on beta's routes; a content agent's writes reach beta through the working tree without it.

```json
{ "tool": "page_patch", "args": { "url": "/about", "old": "since 1987", "new": "since 1986", "if_etag": "\"b7e1\"" } }
```

```json
{ "etag": "\"c402\"", "changed": ["/about"], "findings": [] }
```

### The working tree is beta

Every successful write recompiles only the plans it touched and updates the working tree. Beta serves the working tree: store submits it as a working release after each apply, debounced to one submit per few seconds. A named `release` is the same snapshot, kept and diffable, which is what promotion moves.

## Codes and problems

Wire codes are [network's](../common/NETWORK.md#codes); tools map them to HTTP as rancher's do. The problems store raises:

| problem | when |
| --- | --- |
| `store.guard_refused` | A release or write had an inline script, a foreign form target, a password field outside identity screens, an undeclared origin, or something shaped like a secret. |
| `store.query_refused` | Fork refused a named query against the table version. |
| `store.chrome_copy` | A document redraws the nav or footer, or markup repeats across pages. |
| `store.theme_drift` | Styles or values not from `theme`. |
| `store.orphan_page` | A page reachable from nothing. |
| `store.release_refused` | Rancher refused a release's routes. |
| `stitch.slot_failed` | Reported by barn: a slot keeps failing or timing out. |

## The invariants

- **Barn fetches only releases rancher named,** by hash, once.
- **The release format is one definition,** in `store_plan`, linked by store and barn, and it only grows.
- **Every write is checked when written:** Mustache parses, params are declared, queries pass fork, the guard passes.
- **Every edit takes an `if_etag`,** and a multi-file change is staged and applied all or nothing.
- **An agent can propose a promotion, never make one.**
- **Every tool names its roles, and store re-checks them on every call.** A content agent on `/mcp/content` never reaches `release` or `promote`.
- **`store.http.serve` carries upload tickets only.** Every other operator act is a tool.
- **Store is off the request path.** Store going down stops edits, not the site.
