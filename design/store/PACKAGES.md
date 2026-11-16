# Store packages
How the content layer is cut, and where pages are kept from executing
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Store compiles text an agent wrote into plans barn executes for strangers. The two places a mistake turns into a breach are the **directive parser**, which decides what in a page is live, and the **guard**, which decides whether a page may ship scripts, forms, or foreign origins. Those are store's R0 packages, `store_ssi` and `store_guard`, and both are pure: bytes in, a tree or a verdict out. Everything around them (the tree, front matter, markdown, compiling, releases, attachments, the tools) is R1 or R2, and most of it is careful but ordinary text processing.

The plan format barn executes is a library, `store_plan`, so there is one definition of a release that both sides link. It is R1: a bug there serves a wrong page, but the guarantees that matter (escaping, no directives in component output, slot limits) are enforced by the renderer in barn and the compiler in store, each tested against the same plans.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `store_proto` | `homestead::store::proto` | The `.fbs` schema and generated code for `release.fetch`, `tools.list`, `tools.call` (with `porch_contract`'s types), and `http.serve` for upload tickets | `common_wire`, `porch_contract` | R3 |
| `store_plan` | `homestead::store::plan` | **Library.** The release format: pages, parts, segments, slots, blobs; build and read. Linked by barn. | `common_core` | R1 |
| `store_db` | `homestead::store::db` | Store's SQLite: files and their versions, stages, releases, attachments; blobs by hash in `common_blob` | `common_sqlite`, `common_blob` | R2 |
| `store_tree` | `homestead::store::tree` | Paths, file kinds, URL rules (`_` never served, `index`, one URL one file, `.md` redirects) | `common_core` | R1 |
| `store_frontmatter` | `homestead::store::frontmatter` | Front matter parsing and validation for pages, components, and queries | `common_json` | R2 |
| `store_markdown` | `homestead::store::markdown` | Markdown to HTML through md4c, raw HTML escaped or allowed per kind | md4c | R2 |
| `store_component` | `homestead::store::component` | Components and layouts: declared params, required checks, one level only | `common_mustache`, `store_frontmatter` | R1 |
| `store_ssi` | `homestead::store::ssi` | The five directives: parse `<!--#...-->` into a tree, refuse anything else | `common_core` | R0 |
| `store_query` | `homestead::store::query` | Named query files: front matter, fork check against table versions | `fork_api`, `store_frontmatter` | R1 |
| `store_guard` | `homestead::store::guard` | Rogue-content checks: scripts, forms, password and card fields, origins, secrets, CSP, sizes | `common_core` | R0 |
| `store_compile` | `homestead::store::compile` | Tree to plans: layouts, nav active links, head, literal components, listings, feeds, sitemap, `llms.txt`, `/site.css` | `store_tree`, `store_markdown`, `store_component`, `store_ssi`, `store_query`, `store_plan` | R1 |
| `store_lint` | `homestead::store::lint` | Consistency: chrome copies, repeated markup, theme drift, orphans, page lint | `store_tree`, `store_plan` | R2 |
| `store_release` | `homestead::store::release` | Snapshot, canonical bytes and hash, diff in plain words, submit and propose | `store_compile`, `store_guard`, `store_db`, `rancher_client` | R1 |
| `store_attach` | `homestead::store::attach` | Attachments: fetch by URL, upload tickets, sniff types from bytes, numbering, usage | `store_db`, `common_http` | R1 |
| `store_tools` | `homestead::store::tools` | Implements the tools contract: the catalog with each tool's roles, the role re-check with `porch_roles` on every call, and the handlers: `if_etag`, dry run, stage and apply, render with overlays, findings | most core packages, `porch_contract`, `porch_roles` | R1 |
| `store_server` | `homestead::store::server` | Binds `release.fetch`, `tools.list`, `tools.call`, and `http.serve` (upload tickets only), checks the caller is barn | `store_proto`, `store_tools`, `store_attach`, `store_db` | R2 |
| `store_client` | `homestead::store::client` | The `Store` interface and its wire implementation | `store_proto`, `common_wire` | R2 |
| `store_fake` | `homestead::store::fake` | An in-memory store serving releases built in the test | `store_client`, `store_plan` | R2 |
| `store_system` | `homestead::store::system` | The composition root | everything above | R2 |
| `store_main` | | The binary | `store_system` | R3 |

## Dependencies

```
                               store_main
                                   |
                              store_system
                  +----------------+---------------+
                  |                                |
            store_server                     rancher_client
                  |
             store_tools ----------------+------------------+
              /    |     \               |                  |
   store_release  store_lint  store_attach                 store_db
      /     \          \                                    |
 store_guard store_compile -----+--------------+        common_blob
             /    |      \       \              \
   store_tree  store_markdown  store_component  store_query   store_ssi
                                  |                 |
                          common_mustache        fork_api
                     store_compile ---> store_plan <--- barn (links it)
```

`store_ssi` and `store_guard` depend on nothing but `common_core`. They take bytes and answer a tree or a verdict.

## Quality seeds

### store_ssi (R0)
- **Promise.** Only the five directives, with only their documented keys, are recognized; anything else that looks like a directive is a refusal, not text. Values bound into queries are parameters, never spliced. Nothing nests: no include inside an include, `if` blocks only around markup and slots.
- **Tests.** A gold table of pages with expected trees and refusals; fuzzed nightly; round trip (parse, print, parse) stable; adversarial inputs (unterminated comments, `--` inside values, mixed quotes, directives in attribute values, in `<script>`, in components).
- **Won't.** Evaluate anything, or know about queries, routes, or components beyond their names.

### store_guard (R0)
- **Promise.** A release with an inline script, a script from outside the site's own files, a form posting outside the tenant's routes, a password or card field outside identity screens, an undeclared external origin, or something shaped like a secret is refused, with the path and position. Documents get the strict CSP unless `site.json` opts a path out.
- **Tests.** One fixture per rule and per evasion (event handler attributes, `javascript:` URLs, `srcdoc`, SVG scripts, encoded schemes, `formaction`); the goober rogue-content corpus; fuzzed HTML.
- **Won't.** Fix content, or warn where it should refuse.

### store_plan (R1, library)
- **Promise.** A release is one canonical FlatBuffer whose hash is its id; building the same tree twice yields the same bytes; readers built against an older schema refuse unknown parts rather than skip them.
- **Tests.** Round trips; canonical bytes stable across runs; compatibility with the previous release's schema; a shared fixture set barn's renderer also runs.
- **Won't.** Render, escape, or do I/O.

### store_tree (R1)
- **Promise.** One URL answers from exactly one file; `_` segments are never served; `index` and extension rules match goober's.
- **Tests.** Tables of paths and URLs; collisions refused on put.
- **Won't.** Read file contents.

### store_component (R1)
- **Promise.** Every component's params are declared; required params are present at every use; a component includes no component; every value is escaped except `{{{content}}}` and `{{{body}}}` in layouts.
- **Tests.** Missing params, unknown components, nested includes, escaping of every value kind.
- **Won't.** Run queries or call routes.

### store_query (R1)
- **Promise.** A named query is stored only with a passing fork verdict against the table version it names, and its params' types are declared.
- **Tests.** Verdicts against fake table definitions; a table version change rechecks every query that names it.
- **Won't.** Run a query.

### store_compile (R1)
- **Promise.** Everything decidable at release is decided at release: nav active state, literal components, listings, feeds, sitemap. A page without slots is one segment; a stitched page has at most 16 slots.
- **Tests.** Gold sites compiled to plans; the plan rendered by barn's renderer equals a reference render.
- **Won't.** Decide whether content is allowed (`store_guard`) or tasteful (`store_lint`).

### store_release (R1)
- **Promise.** A release is immutable and named by its hash; its diff lists every page change and every new origin, script, form, query, grant, and route include; submission to rancher carries every route and named query the release needs.
- **Tests.** Diff completeness on generated pairs; resubmitting is idempotent.
- **Won't.** Promote; it proposes.

### store_tools (R1)
- **Promise.** Every tool is in the catalog with its roles; every call is re-checked with `porch_roles` before a handler runs; every changing tool takes `dry_run` and every edit `if_etag`; a stage applies all or nothing.
- **Tests.** The contract test suite from `porch_fake` against store's source; a role-path call for a tool outside that role refused even when porch forwards it; dry runs change nothing.
- **Won't.** Verify tokens (barn did), or decide content rules (`store_guard`, `store_lint`).

### store_attach (R1)
- **Promise.** An attachment's type comes from its bytes, never its name; tickets are one-time and expire; an attachment in use isn't deleted without `force`.
- **Tests.** Polyglot files, mismatched extensions, oversized uploads, expired tickets.
- **Won't.** Store files people upload while using a product (that is pond and boxes).

## How it's wired

`store_system` builds: the clock and network, `common_blob`, `store_db`, the compilers and checkers, the `rancher_client` (with a config subscription for table versions and live releases), `store_release`, `store_attach`, `store_tools`, and `store_server`.

- **Threads.** One Asio context for the wire; one writer thread for `store_db`; a compile pool sized to cores, since a write recompiles only its plans but a release compiles the site.
- **Start.** Open the database, subscribe to rancher's config and wait for the snapshot (queries can't be checked without table versions), then listen. `release.fetch` serves from the database without rancher.
- **Shutdown.** GOAWAY, finish in-flight applies and releases, flush the pending beta submit, close.

## Review queue

| tier | packages |
| --- | --- |
| R0 | `store_guard`, `store_ssi` |
| R1 | `store_plan`, `store_compile`, `store_release`, `store_component`, `store_query`, `store_tree`, `store_attach`, `store_tools` |
| R2 | `store_lint`, `store_db`, `store_frontmatter`, `store_markdown`, `store_server`, `store_client`, `store_fake`, `store_system` |
| R3 | `store_proto`, `store_main` |

Build order, so agents can work in parallel:

1. `store_plan` and `store_proto` first: barn's renderer is built against `store_plan` at the same time.
2. In parallel: `store_ssi`, `store_guard`, `store_tree`, `store_frontmatter`, `store_markdown`, `store_db`, `store_client` + `store_fake`.
3. In parallel: `store_component`, `store_query` (needs `fork_api`), `store_attach`.
4. `store_compile`, then `store_lint` and `store_release` (needs `rancher_fake`).
5. `store_tools` (needs `porch_roles` and `porch_contract`), `store_server`, `store_system`, `store_main`.

## The invariants

- **The parser and the guard are pure and R0.** Bytes in, a tree or a verdict out.
- **One release format,** in `store_plan`, linked by store and barn.
- **Every check runs at write and again at release.** A tool can't skip one.
- **Store never serves a visitor.** `store_server` answers barn only.
