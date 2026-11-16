# Holes
The jobs goober's host did that no homestead service owned, and where each one landed
------

> Status: the first pass (2026-11-16) listed eleven holes; the author decided them the same day. What remains open is at the end.

## The idea

Homestead split goober's host into pond, fork, barn, rancher, permit, sun, store, barker, and ranger. That covered data, SQL, compute, routing, identity, time, content, outbound mail, and logs, but goober's host did more, and some of it was already assumed without an owner: permit and barker kept keys "by reference to the key store," store refused user uploads, and ranger refused to be metrics.

This doc walked goober's design (`history/goober-design/human` and `machine`) for every role with no owner. The author's rule for the answers: **prefer a library inside barn, or a feature of a service that already owns the data, over a new building.** Only telephony became a new service.

## Decided

| role | goober | decision | name | design |
| --- | --- | --- | --- | --- |
| Keys and secrets | human/064, human/325 | A library, linked by barn and every service that holds a key; goober's vault design, with secrets write-only through rancher | **cellar** | [cellar](cellar/IDEA.md) |
| Files people upload | human/060 | A function of barn: routes map to S3 prefixes, upload to them, and hand attachments to boxes; boxes own garbage collection | **silo** | [barn](barn/IDEA.md) |
| Outbound calls | human/030, human/330, human/800 | A library barn uses to speak to third parties, OpenAPI first; a box controls the request, the route grants a domain and URI prefixes; a CLI runs Postman-compatible test cases (HEVY and Strava first) | **hound** | [hound](hound/IDEA.md) |
| The operator surface | human/300, machine/028, human/410 | A library barn imports to serve every service's tools over MCP and HTTP, filtered by role (`/mcp/content`), so agents can form a hierarchy | **porch** | [porch](porch/IDEA.md) |
| Running homestead itself | human/099, human/663, machine/046 | Deploy tooling, like goober's deploy script: diff one machine, then deploy and set everything up | **farmhand** | [farmhand](farmhand/IDEA.md) |
| The bus and event routes | human/040, human/039 | No bus. **Pond is the event bus**: it is the data source, and change data capture through a tailer is the design. Boxes listen through `event` routes, which pond drives with an event tailer per route | (none) | [pond](pond/IDEA.md), [barn](barn/IDEA.md) |
| Telephony | machine/025 | A service, in barker's shape: numbers per tenant, texts out through the outbox, texts and calls in as barn routes | **rooster** | not yet written |
| Source and builds | human/521, machine/020 | **Removed.** Agents need a laptop; keeping source and building in the machine was a whole mess of beans in goober | | |
| Stateful boxes | human/400 | **Removed.** Barn stays stateless; homestead won't take on leases, rehydration, and a fleet registry | | |

### Why each landed where it did

- **cellar is a library** because every use of a secret is inside a process that already exists (barn calling S3 or Strava, permit signing, barker signing DKIM), and a network hop to fetch a plaintext secret is a worse boundary than a sealed one opened locally with the host's vault key. Rancher seals; only hosts with the vault key open.
- **silo is barn's** because reading and writing S3 under a route's rules is the same work as reading and writing pond under a route's rules: resolve what the request may touch, check it at runtime, and let the box decide. Several ways to upload (a plain upload route, attachments on a box route, a presigned URL a box mints) are all one prefix rule.
- **hound is a library** because an outbound call is part of a box's request, under the box's deadline, and belongs on the request's thread, with the route's grant. Its CLI makes the same code testable from a laptop against real providers.
- **porch is a library** because the tools already belong to the services that implement them. Porch only gathers them, filters them by role, and speaks MCP, and barn is already the one door from the internet.
- **pond is the bus** because every event worth reacting to is a write, and pond already has the only ordered log of writes, a cursor per consumer (tailers), and a feed. A second bus would be a second copy of the truth.

## Holes that aren't services

| hole | goober | where it belongs |
| --- | --- | --- |
| Runtime flags (freeze orders without a new version) | machine/039 | Rancher: a flag set per service, declared by the box, changed by an audited operator tool, read through `barn-abi`. |
| Typed request bodies (validate before the box wakes) | human/036, human/037 | Rancher routes: a body schema per route, checked by barn. |
| One route, many audiences (JSON or HTML by `Accept`) | human/070 | Barn, rendering a box's JSON through a store component, as `.shtml` route includes do. |
| Box-to-box calls | human/039 | Barn: an `internal` route protocol, in-process, with a callers list on the route. |
| Host libraries (`time@1`, `ics@1`) | machine/831 | Barn: library imports beside `barn-abi@1`, each its own package and tier. |
| Caddy's on-demand TLS check | human/099 | Barn answers the ask from rancher's domain registry; farmhand configures Caddy. |
| Approvals of individual changes | human/305 | Replaced by beta and bulk promotion. |
| Installable apps (PWA) | machine/782 | Store: a manifest and service worker generated from `site.json`. |
| Quotas | human/390 | Covered: rancher sets them, pond and barn enforce them. |

## Still open

| role | goober | lean | name |
| --- | --- | --- | --- |
| Metrics over time (version health, guardrail windows, performance) | machine/044, machine/048 | A small service; rancher keeps only what a guardrail needs now and reads windows from it | **gauge** |
| Skills and playbooks for agents and people | machine/050 | A store site with a few routes, served through porch's `guide` tool per role | **almanac** |
| Telephony's design | machine/025 | Write rooster's IDEA when a tenant needs a number | **rooster** |

## The invariants

- **Every job goober's host did has an owner, or a written reason it doesn't.**
- **A role referenced by another doc is never left undefined.**
- **Library or feature before service.** A new building needs a reason a library or an existing owner can't carry.
- **A new service or library follows the same rules:** an IDEA, an API.md, a PACKAGES.md with tiers, and a farm name.
