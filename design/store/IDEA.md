# Store
A site's content, its chrome written once, and pages that stitch in live data with one request
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober turned into a CMS almost by accident, and that part worked. Agents are very good at shipping markdown and whole HTML pages, and a site an agent can write over MCP, read back, and publish in a minute is a real product. What didn't work was everything *between* pages. Every page carried its own copy of the navigation, the footer, and half the stylesheet, so the site drifted a little more with every edit. Adding one nav link once took 23 writes: 22 whole documents re-put, plus `site.json` (goober machine/027). Partials fixed the nav, but by then the content tree, templates, partials, data files, listings, feeds, and attachments had piled up as features of the host, all beside routing and identity.

Store is the content layer on its own, built around one observation: **tokens spent on chrome are tokens not spent on content, and every copy of the chrome is a chance to get it wrong.** So store makes the shared parts of a site (navigation, layout, footer, components, theme) data that is written once and referenced everywhere. A page carries only what makes it that page. Full fidelity stays: an agent can still write any HTML it wants, and the cheap, consistent path is just the easy one.

Store also adds what goober deliberately left out: pages that pull in live data. A new content type, `.shtml`, uses server-side includes to stitch a named query or a box route's answer into a page. One HTTP request, no WASM for the page itself, and evaluators small enough to run in microseconds.

Store holds a tenant's site and compiles it into releases. Barn serves releases. Rancher decides which release answers which domain.

## Where store sits

| store talks to | for | how |
| --- | --- | --- |
| barn | Serving: barn pulls compiled releases by hash and renders them | Barn subscribes to rancher's config log, which names releases; it fetches each one from store once and caches it forever. |
| rancher | Every release becomes routes on the tenant's domains, under the `site` service | Store submits a release's routes, grants, and named queries; rancher checks them like any other routes. |
| pond | Nothing directly | `.shtml` queries run through barn, under a grant rancher approved. |
| agents, operators | Writing, previewing, linting, and releasing | Store's tools, tagged with roles, served by [porch](../porch/IDEA.md) in barn as MCP at `/mcp/content` (and the other role paths) and as HTTP at `/_ops/api/{tool}`. |

Store is one machine with its own SQLite database for files, versions, and releases, plus a blob directory (S3-shaped, as in goober) for content-addressed bytes. It never sees a visitor's request.

## The site tree

Each tenant has one tree. Every path is text an agent can read and write, except attachments.

| path | what it is | served |
| --- | --- | --- |
| `site.json` | The site: name, domain, menus, footer, theme, defaults for head tags | Never directly; it feeds everything else. |
| `*.md` | A markdown page | Rendered through its layout. |
| `*.html` | A fragment (with front matter, wrapped in its layout) or a document (no front matter, served as written) | Rendered, or as written. |
| `*.shtml` | A stitched page: a fragment that may include live data (see below) | Rendered per request. |
| `_layouts/$name.html` | Mustache layouts: the head, the chrome, a `{{{body}}}` slot | Never. |
| `_components/$name.html` | Mustache snippets with parameters: a card, a hero, a call to action, the nav | Never. |
| `_queries/$name.sql` | Named, read-only SQL with front matter naming its tables and limits | Never. |
| `_data/$name.json` | Data for layouts and components | Never. |
| `*.css`, `*.js`, fonts, icons | Static files | As written, content-addressed. |
| `/attachment/$id.$ext` | Images and documents, numbered, sniffed from their bytes | Streamed, immutable. |

Goober's rules carry over: a path segment starting with `_` is never served, a page's URL is its path without the extension, `index` is its directory's URL, `/p.md` redirects to `/p`, and one URL answers from exactly one file.

## The chrome is data

`site.json` is the single place a site's shared shape lives:

```json
{
	"name": "Bob's Plumbing",
	"menus": {
		"main": [
			{ "label": "Home", "href": "/", "exact": true },
			{ "label": "Services", "href": "/services" },
			{ "label": "Jobs", "href": "/jobs", "signed-in": true },
			{ "label": "Call us", "href": "tel:+15550100", "class": "cta" }
		],
		"footer": [
			{ "label": "About", "href": "/about" },
			{ "label": "Privacy", "href": "/privacy" }
		]
	},
	"theme": {
		"color-ink": "#1d2433", "color-accent": "#0a6cff", "color-paper": "#fdfcf9",
		"font-body": "system-ui", "font-display": "Georgia, serif",
		"radius": "6px", "measure": "42rem"
	},
	"head": { "description": "Family owned since 1987, serving the whole valley.", "image": "/attachment/1.jpg" },
	"layout": "base"
}
```

From that, store builds what goober asked agents to build by hand:

- **Menus render through the `nav` component**, built in and overridable at `_components/nav.html`. Store marks the active link for every page *at release time*, since a static page's path is known (goober's `data-match` rules: longest prefix wins, `exact` matches only itself). A `signed-in` item is dropped for visitors. Adding a link is one line in one file.
- **The theme becomes `/site.css`**: CSS custom properties from `theme`, plus a small built-in kit of classes (`.stack`, `.grid`, `.card`, `.button`, `.prose`, `.cta`) written against those properties. Changing the accent color is one value, not a find and replace across pages.
- **Head tags come from defaults**: title pattern, description, social image, canonical URL, JSON-LD, built from `site.json` and front matter. A page only says what differs.

## Components

A component is a named Mustache snippet that takes parameters. Goober's partials became components, and the agent-facing contract is the point: each component starts with front matter declaring its parameters, so the whole catalog can be handed to an agent in a few hundred tokens.

```html
---
summary: A linked card with an optional image and badge
params:
  title: { required: true }
  href: { required: true }
  image: {}
  badge: {}
  text: {}
---
<a class="card" href="{{href}}">
  {{#image}}<img src="{{image}}" alt="">{{/image}}
  <h3>{{title}}{{#badge}} <span class="badge">{{badge}}</span>{{/badge}}</h3>
  {{#text}}<p>{{text}}</p>{{/text}}
</a>
```

Every value is escaped. A component can't include another component, so nothing loops. That's goober's one-level rule, and it keeps the evaluator trivial. Store refuses an include that names an unknown component or leaves out a required parameter when the page is written, not when it's served.

## Pages

Pages keep goober's three kinds and add a fourth:

| kind | file | what it carries | tokens to write |
| --- | --- | --- | --- |
| markdown | `.md` | Words, with front matter | The fewest. The default for prose. |
| fragment | `.html` with front matter | The page's own markup; layout, nav, and footer come from the site | Only the body. |
| stitched | `.shtml` | A fragment plus directives that include components and live data | Only the body, plus one line per include. |
| document | `.html` without front matter | A whole page, served as written | All of it. For pages that really are one-offs. |

Documents stay because fidelity matters: a landing page with its own art direction should be possible. But a document can still use the chrome: `<!--#include component="nav"-->` works in a document, and store's lint warns about a document that redraws the nav or footer by hand.

Front matter is goober's: `title`, `layout`, `date`, `summary`, `image`, `tags`, `order`, `draft`, `requirements`, `redirect`, and listings (`list`, `sort`, `per-page`, `feed`). Listings, the sitemap, `llms.txt`, Atom feeds, and error pages all come from the tree, so store computes them at release, and they cost nothing per request.

## Stitched pages: `.shtml`

A stitched page is a fragment whose body may carry directives. The syntax is Apache's server-side includes, `<!--#directive key="value"-->`, on purpose. It's decades old, it's all over the training data, an agent writes it correctly the first time, and a browser shows it as a comment if it ever escapes unexpanded. The content type even gets its historical name: `text/x-server-parsed-html`.

```html
---
title: Recent jobs
grant: [recent_jobs]
---
<h1>Jobs finished this week</h1>

<!--#if signed_in-->
  <p>Welcome back, <!--#echo var="user.name"-->.</p>
<!--#endif-->

<div class="grid">
  <!--#include query="recent_jobs" component="card" city="{query.city}" limit="12"
               empty="No jobs yet this week."-->
</div>

<!--#include route="/weather/today" component="weather" fallback="" timeout-ms="80"-->
```

The directive set is small and fixed:

| directive | does | evaluator |
| --- | --- | --- |
| `include component="x" k="v"...` | Renders a component with literal parameters | Mustache, at release when every value is literal; per request otherwise |
| `include query="x" component="y" ...` | Runs a named query, rendering each row through the component (row columns are its parameters) | Pond, through barn's query cache |
| `include route="/path" component="y"` | Calls a GET route on this tenant as the same caller; renders its JSON through the component | Barn's normal route path, in-process |
| `echo var="..."` | Writes one escaped value from a fixed namespace: `page.*`, `site.*`, `user.name`, `request.path`, `query.*`, `now.year`, `now.date` | A lookup |
| `if` / `else` / `endif` | Keeps a block when a predicate holds: `signed_in`, `grant="admin"`, `query="q"` (present) | A lookup |

There are no variables to set, no loops except a query's rows, no arithmetic, and no nesting of includes. An included component's output is never scanned for directives, and a route's JSON is always rendered through a component, so a box can't inject markup into the site's pages. Values bound into a query (`city="{query.city}"`) are bound as SQL parameters, never spliced.

### Named queries

A query is a file, in the language agents already write best:

```sql
---
tables: { jobs: { version: 2, columns: [id, title, city, finished_at, photo] } }
params: { city: { type: text, optional: true } }
limits: { max-rows: 50, max-ms: 20 }
---
SELECT id, title AS title, '/jobs/' || id AS href, photo AS image, city AS badge
FROM jobs
WHERE finished_at > :now - 604800000 AND (:city IS NULL OR city = :city)
ORDER BY finished_at DESC
```

This is the prepared, named query the design adopted for routes: fork checks it once, at release, against the table version's real columns, and pond prepares it once. A page's `grant` names the queries it uses. Rancher turns those names into the grant on the page's route, so routing still owns grants, and a page can't read anything its route wasn't given. Aliasing columns to a component's parameter names (`AS title`) is how one card component serves many queries.

### One request, evaluated fast

Store compiles every page at release into a **plan**: a list of literal byte segments and slots. Layout, nav (with its active link already marked), footer, head, literal components, and markdown are all folded into the segments, so a plain page compiles to one segment and is served from memory with its ETag. A stitched page compiles to segments plus a few slots.

Per request, barn:

1. resolves the route to the release and page plan, and checks the page's `requirements` with permit's answer;
2. starts every slot at once: queries to pond (or barn's cache), route includes in-process;
3. waits for each slot up to its `timeout-ms` (default 100), within the page's deadline (default 250 ms);
4. writes segments and finished slots in order. A slot that fails or times out writes its `fallback` (default: nothing), and barn records a problem naming the page, the slot, and why.

Limits keep the shape honest: at most 16 dynamic slots per page, and a route include may not itself be a stitched page. A page never writes: route includes must be GET routes whose objects are all read-only, and rancher refuses the release otherwise.

**Caching follows what the page used.** A page whose slots don't depend on the viewer (no `user.*`, no `signed_in`, no route include marked per-viewer) is cached by release, path, and the query values it read. Its entry is dropped when pond's feed moves a table one of its queries read, which is exactly the machinery barn already uses for query answers. A page that depends on the viewer is `private, no-cache`, with `Vary: Cookie`. Either way, the ETag is the hash of the bytes served.

## Beta and production

A tenant is one website, with `beta.` and `ops.` beside it, and changes land in beta immediately. Store follows that exactly:

- **Every write updates the working tree**, and store recompiles only the plans the write touched. The working tree is always live at `beta.example.com`, so an agent sees its change on the real site seconds after making it.
- **A release is a snapshot of the tree**, named by its hash and immutable. Rancher's `site` service has a traffic policy over releases, just like box versions: one release live, an A/B split by user or IP ("which home page gets more calls"), and rollback in one step, because the previous release stays loaded. Shadowing doesn't apply to content.
- **Promotion is the bulk review.** Promoting beta to production shows a person one diff: pages added, removed, and changed (a text diff of each), plus every new external origin, script, form, named query, grant, and route include. That's the review HUMAN.md describes. Nobody reads every page, but a page that grew a login form in a release about a typo stands out immediately.

Goober's rogue-content checks run on every release: no inline scripts, and scripts only from the site's own files; forms submit only to the tenant's routes; no password or card fields outside identity screens; external origins must be declared in `site.json`; nothing that looks like a secret; size limits; broken links. Documents get the strict CSP unless `site.json` opts out per path. That closes the gap goober left open.

## Spending tokens on content

Every tool answers compactly by default, with the full text only when asked (`verbose`). The design target is that the common edits cost tokens in proportion to the change, not to the site.

| task | goober | store |
| --- | --- | --- |
| Learn a site | `page_list`, then read templates and pages one by one | `site_map`: menus, layouts, the component catalog with parameter signatures, named queries, and every page's URL, title, and kind, in one call |
| Add a nav link | 23 writes before partials; one partial re-put after | One line in `site.json` |
| Change a paragraph | Re-put the whole page | `page_patch`: exact old and new strings against the page's ETag |
| Restyle the site | Edit CSS in every page that inlined it | One value in `theme` |
| A grid of 12 cards from data | 12 hand-written blocks, stale on the next change | One `include query` line, always current |
| Check the result | `page_render` returned the HTML | `render` returns status, ETag, lint findings, and which segments changed; the HTML only with `verbose` |
| Ship ten files | Several calls, each checked alone | `stage` then `apply`, all checked before any is written |
| Upload an image | Base64 in a tool call, often cut off | `attachment_put` from a URL, or a one-time upload ticket |

## Consistency is linted

Store's lint runs on every write and every release. Its findings are problems in rancher's problem log, so the agent that caused one is the agent that sees it.

| finding | why it matters |
| --- | --- |
| A document that redraws the nav or footer | Chrome copies drift; include the component instead. |
| The same block of markup (512 bytes or more, normalized) on three or more pages | It should be a component; the finding suggests a name and parameters. |
| A `<style>` block or more than a few `style=` attributes on a page | Page-level CSS drifts from the theme; use the kit or the theme. |
| A colour or font that isn't from `theme` | Same reason, caught at the value. |
| A page reachable from no menu, listing, or link | Orphans are usually mistakes. |
| An unknown component, a missing required parameter, an unrendered `{{` | Broken now, so refused at write. |
| Title or description outside the strict lengths, missing or duplicate JSON-LD | Goober's strict page lint, kept. |
| A slot that keeps failing or timing out in production | A broken data dependency, seen with the page that has it. |

## The operator API

| tool | does |
| --- | --- |
| `site_guide` | The model of this document in one page, for an agent's first call. |
| `site_map` | The whole site's shape, compactly (see above). |
| `site_put`, `site_patch` | Write or patch `site.json`. |
| `page_put`, `page_patch`, `page_get`, `page_delete` | Pages of every kind, by URL. A put removes any other file answering the same URL. |
| `component_put`, `layout_put`, `query_put` | Each checked when written: Mustache parses, parameters are declared, SQL passes fork against the table version. |
| `render` | Renders a URL from the working tree (or with unsaved files layered over it), signed in or `as: "signed_out"`, without storing anything. |
| `lint` | Runs the checks over the tree or over unsaved files. |
| `stage`, `apply` | Build a multi-file change, then write it all or nothing. |
| `attachment_put`, `attachment_list`, `attachment_delete` | Media, with what uses each one. |
| `release`, `diff` | Snapshot the working tree; diff any two releases in plain words. |
| `promote` | Ask to move production to a release. It needs a person's approval through rancher, so an agent can propose a promotion but can't make one. |

## What store is not

- **Not a renderer on the request path.** Store compiles; barn serves. Store going down stops edits, not the site.
- **Not a database for user content.** Comments, bookings, and profiles are domain data in pond, written by boxes. A page shows them through a named query.
- **Not a template language.** Mustache and five directives are the whole vocabulary. Anything that needs logic is a box route, included by a page.
- **Not a file host for uploads.** Attachments are the site's own media, set by operators and agents. Files people upload while using a product belong with boxes and pond.

## Open questions

1. **Does the `.shtml` name stay?** It's historical and agents know it. A `.html` page with a `stitch: true` front matter key would avoid a new extension.
2. **Directives in a box's own HTML?** Goober let a box opt in with `"includes": true`. The lean is components only, no live includes, so a box can't widen its own reads through a page.
3. **Can a component include one component?** One level is simple. Two levels would let a `card-grid` use `card`, but loops then need checking.
4. **A/B by page, or only by release?** A whole release is simplest. A split on one page would let an agent test a headline without forking the site.
5. **Per-viewer route includes.** The lean is that a route include is shared unless the route is marked per-viewer, mirroring the SSE fan-out rule. Is that the right default for pages?
6. **Where do attachment bytes live?** Store's blob directory is the start. Pond may grow object storage for uploads later, and the two could share it.
7. **Should the built-in kit be versioned and pinned?** The kit and the built-in components are host code. Goober pinned them so an upgrade never changed a deployed page; a release could pin the kit's hash.

## The invariants

- **The chrome is written once.** Menus, layout, footer, theme, and head defaults live in `site.json`, layouts, and components, never copied into pages.
- **Full fidelity, cheap consistency.** Any HTML is possible; the shared path is the short one, and lint points back to it.
- **Everything decidable at release is decided at release.** Layouts, nav state, literal components, listings, feeds, and the sitemap cost nothing per request.
- **One request per page.** Live data is stitched in by barn, in parallel, under a deadline, with a fallback for each slot.
- **Pages can't execute and can't write.** Markdown, Mustache, and five directives; every value escaped; route includes are read-only GETs rendered through components.
- **Routing owns a page's reads.** Named queries are checked by fork at release and granted by rancher on the page's route.
- **Releases are immutable, and their id is their hash.** Beta is the working tree, production is a release, and rollback is one step.
- **Promotion is a person's act.** Agents write, preview, release, and propose; a person approves the diff.
- **A failing slot costs only its slot.** It writes its fallback and becomes a problem; the rest of the page still serves.
