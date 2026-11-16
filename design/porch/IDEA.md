# Porch
The operator tools, gathered into one MCP surface in barn, and cut by role so each agent sees only its job
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's operator surface was one MCP server with every tool on it: deploys, pages, databases, mail, logs, keys. An agent connected to it saw a hundred tools, most of them nothing to do with the job it was given, and the person couldn't hand an agent "just the content" without handing it everything. HUMAN.md's first lesson is that an agent must be focused, and the second is that organizational limits apply to agents too. A tool list is a job description, so it should be one.

Porch is the operator surface as a **library that barn links**. It holds no state and owns no tools. The services that implement tools (rancher, and through it sun, barker, and ranger; and store) publish them over the wire as `$service.tools.list` and `$service.tools.call`, each tool tagged with the roles it serves. Porch gathers those catalogs, serves them as MCP and as plain HTTP JSON on every `ops.` domain, and **filters them by role**:

```
https://ops.bobsplumbing.com/mcp            every tool the caller's grants allow
https://ops.bobsplumbing.com/mcp/content    pages, components, layouts, queries, renders, attachments
https://ops.bobsplumbing.com/mcp/deploy     boxes, routes, services, ramps, releases, promotions proposed
https://ops.bobsplumbing.com/mcp/data       tables, named queries, buckets, query explorer
https://ops.bobsplumbing.com/mcp/observe    problems, logs, traces, health
https://ops.bobsplumbing.com/mcp/mail       domains, templates, dead letters, suppression
https://ops.bobsplumbing.com/mcp/identity   members, grants, operator entries, audit
```

That is how a hierarchy of agents works: a lead agent holds `ops/admin` and connects to `/mcp`; it hands a content agent a token for `ops/content`, which connects to `/mcp/content` and sees a dozen tools about pages. The content agent can't deploy a box, not because it was told not to, but because the tool isn't there and the grant isn't held.

## Where porch sits

```
  agent (MCP client)                person (browser)
        |                                 |
        v                                 v
   Caddy -> barn ------------------------ barn
             |  barn_caller: token verified (permit_token), grants known
             |  porch: route /mcp/{role} or /_ops/api/{tool}
             |    porch_roles: may this caller see / call this?
             |    porch_catalog: which service owns this tool?
             |
             +--> rancher.tools.call  (rancher's own tools; sun's, barker's, ranger's through rancher)
             +--> store.tools.call    (the site tools)
             +--> barn's own routes   (on tenant domains: the tenant's mcp routes)
```

| porch talks to | for | how |
| --- | --- | --- |
| barn | The verified caller, the tenant, the request id, the HTTP request | In process: barn calls porch with a `Caller` it already built |
| rancher | Its catalog and its calls, and the role tree | `rancher.tools.list`, `rancher.tools.call`, through the `rancher_client` interface barn gives it |
| store | Its catalog and its calls | `store.tools.list`, `store.tools.call`, through `store_client` |
| ranger | One `agents` line per tool call | Barn's `common_log` spool |

## Roles

A role is a job an agent or a person does on a site. Rancher holds the role tree per deployment, as configuration, and publishes it on its config log:

| role | for | contains |
| --- | --- | --- |
| admin | The owner, and the lead agent | every role |
| deploy | Boxes, routes, services and policies, table versions, releases, proposing promotions | |
| content | Pages, layouts, components, attachments, renders, lint, stage and apply | |
| data | Tables, named queries, buckets, ad-hoc reads | |
| observe | Problems, logs, traces, health, dead letters (read) | |
| mail | Sending domains, templates, dead letters, suppression | |
| identity | Members, grants, invites, operator entries, audit | |

- **A role is held as a grant**, `ops/{role}`, in the tenant's realm in permit. `ops/admin` satisfies every role. Host operators hold `host:` roles, and `host:admin` satisfies every role on every `ops.` domain, audited per tenant entered.
- **A tool names its roles.** `page_patch` is `content`; `query_put` is `content` and `data`; `release` is `deploy`. A tool with several roles is visible under each.
- **The tree only grows by configuration.** A new role is a config change in rancher, reviewed like any other, and tools tagged with it appear under it on the next catalog version.

## Focus and security are two different checks

| check | asks | decided by | on failure |
| --- | --- | --- | --- |
| The path role (`/mcp/content`) | What is this connection for? | The URL the agent was given | It narrows the catalog. It never widens anything. |
| The caller's grants (`ops/content`) | What may this caller do? | Permit's token, checked in barn | A role in the path the caller doesn't hold answers 403 at connect; a tool outside the caller's grants doesn't exist for them. |
| The owning service's re-check | Did porch get it right? | rancher or store, from the `Caller` sent with every call | `refused`, and a problem, because porch and the owner disagreed. |

A tool is visible on `/mcp/{role}` when the tool is tagged with that role and the caller holds it; on `/mcp`, when the caller holds any of the tool's roles. Calling a tool that isn't visible answers as if it didn't exist. The path is focus; the grants are the security; the owner's re-check is the second lock.

## A hierarchy of agents

| step | who | how |
| --- | --- | --- |
| 1. Connect | The lead agent | OAuth 2.1 with PKCE against `/_permit/`, consent naming `ops/admin` on `ops.bobsplumbing.com` |
| 2. Delegate | The lead agent | A token exchange at `/_permit/oauth/token` for `ops/content`, one hour, for "rewrite the services pages" |
| 3. Work | The content agent | Connects to `/mcp/content` with the child token; sees content tools only |
| 4. End | Either | The child expires, or the lead revokes it; revoking the lead revokes every child |

The rules are permit's (see [permit](../permit/IDEA.md#operator-roles-and-agent-hierarchy)): a child holds a subset of its parent's grants, lives no longer than its parent, is at most three levels deep, and only a parent holding a role that delegates (`admin` and `deploy` to start) can mint one. Every tool call records the whole lineage, so the person reading the log sees "the content agent, minted by the lead agent, acting for Bob."

## Built-in tools

Porch adds two tools to every role, so an agent's first call answers "what am I for":

| tool | answers |
| --- | --- |
| `guide` | What this role is for, in a page, its tools with one line each, and the playbook for the common job. Written in rancher's role configuration, so the person tunes it. |
| `whats_new` | What the tools under this role gained or changed in recent builds, from the notes each service publishes with its catalog. |

## Tool calls are ordinary operator calls

Porch passes a call through untouched, so the conventions rancher's and store's tools already follow hold everywhere:

- **Dry run.** Every changing tool takes `dry_run`, and porch never sets or strips it.
- **`if_seq` and `if_etag`.** Concurrency control is the owner's; porch forwards the argument.
- **Widenings.** A changing tool answers what it widens, and porch puts `widens` in the MCP result's text first, so an agent reads it before anything else.
- **Problems.** A refusal comes back with its problem code and hint, as an MCP tool error the agent can act on.
- **Tracing.** Every call writes one `agents` line to ranger: tool, role path, subject, token id, lineage, the owning service, outcome, and timing, never arguments or answers. The request id rides the wire, so `log_trace` shows the call in barn, rancher, and pond together.

## The person's panel, later

A panel for the owner is the same tools read by a person: the promotion diff, the problem log, traces, ramps. The lean is to build it as a **store site on the platform's own tenant**, with a few pages and scripts that call `/_ops/api/{tool}` with the person's session cookie, so homestead builds its own tools out of its own parts. Porch needs nothing new for it: the HTTP form, the session cookie through barn's caller, and the same role filter (a person holding `ops/content` gets a content panel).

## On tenant domains

A tenant's own `mcp` routes (barn's MCP for the product's users and their agents) get the same treatment: a route may carry `"roles": ["support"]`, and `/mcp/support` on `bobsplumbing.com` lists only those routes, filtered by the route's permit requirement as before. Porch's filter runs with barn's route table as its catalog source. The roles there are the tenant's own, not `ops/` roles.

## What porch is not

- **Not a tool owner.** A tool lives in the service that does the work; porch only gathers and filters.
- **Not stateful.** It caches catalogs by version and holds MCP sessions in memory; a barn restart costs a reconnect.
- **Not the authority.** Permit says who; the owner re-checks; porch's filter is the first lock, not the only one.
- **Not a service.** It runs in every barn, so the operator surface scales with barn and needs no new machine.

## The invariants

- **Tools are owned by the service that does the work.** Porch gathers, filters, and forwards.
- **A role path only narrows.** `/mcp/{role}` never shows or allows a tool the caller's grants don't.
- **Grants decide; owners re-check.** Every `tools.call` carries the caller, and the owner refuses what porch shouldn't have sent.
- **A role is a grant, `ops/{role}`, in the tenant's realm.** `ops/admin` holds every role; `host:admin` holds every role on every domain, audited.
- **An agent never holds more than its parent.** Sub-agents are subsets, shorter-lived, at most three deep, and revoked with their parent.
- **Every tool call is one line in ranger,** with the lineage, never the arguments.
- **Dry runs, `if_seq`, and widenings pass through untouched.**
