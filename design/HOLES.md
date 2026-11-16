# Holes
The jobs goober's host did that no homestead service owns yet, and names for the ones that need a building
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified. Names are suggestions; the farm theme is kept so every service still reads as one place.

## The idea

Homestead split goober's host into pond, fork, barn, rancher, permit, sun, store, barker, and ranger. That covers data, SQL, compute, routing, identity, time, content, outbound mail, and logs. But goober's host did more than that, and some of what it did is already assumed by the new docs without anyone owning it: permit and barker both keep keys "by reference to the key store," and no doc defines a key store; store says user uploads "belong with boxes and pond," and pond says blobs "go to object storage"; ranger says it is "not metrics," and rancher would have to become a time-series database to drive guardrails.

This doc walks goober's design (`history/goober-design/human` and `machine`) and lists every role with no owner, what goober did about it, whether it deserves its own service or belongs inside an existing one, and a name. A hole that is a package or a section in an existing doc is listed too, so nothing falls between the buildings.

## The holes at a glance

| role | goober | today in homestead | suggestion | name | when |
| --- | --- | --- | --- | --- | --- |
| Keys and secrets | human/064, human/325 | Referenced by permit, barker, barn's fetch; owned by nobody | A service | **cellar** | Now: blocks permit and barker |
| Files people upload | human/060 | Refused by store and by pond | A service | **silo** | Before the first product with uploads |
| Outbound calls, providers, crawling | human/030, human/330, human/800, machine/017 | `barn_fetch` to declared origins, with no credentials, caching, or politeness | A service | **hound** | Before the first box that calls a paid API |
| The owner's panel | human/300, machine/028, human/999 | An operator API and MCP in rancher, no screen | A service (a site) | **porch** | Before the first promotion a person reviews |
| Metrics over time | machine/044, machine/048, version_health | Counts pushed to rancher, kept nowhere over time | A service | **gauge** | Before ramps with guardrails |
| Running homestead itself | human/099, human/663, machine/046 | Assumed ("a deploy knows which packages changed"); owned by nobody | A service | **farmhand** | Before a second machine |
| The bus and event routes | human/040, human/039, sources and event routes | Pond's feed for data, SSE fan-out in barn; no topics, no "run a route when this changes" | A service | **creek** | When a product needs it |
| Skills and playbooks | machine/050, CHANGES.md / `whats_new` | `site_guide` in store; nothing for the platform | A service, small | **almanac** | Soon: agents are the workforce |
| Source and builds | human/521, machine/020, human/110 | Boxes arrive as finished modules from somewhere | A service | **shed** | When agents write boxes without a laptop |
| Telephony | machine/025 | Nothing | A service | **rooster** | When a tenant needs a number |
| Stateful boxes | human/400 | Barn is stateless by design | Not yet | **stable** | Only when a real product needs it |

## Services

### cellar: keys and secrets

**Goober.** Keys lived in their own SQLite database, encrypted per secret with a data key, data keys wrapped by a root key in a vault that was touched only at start and at rotation (human/064). Operators managed them with host roles (human/325). Keys covered reply-token signing, OAuth and session secrets, outbound credentials by name (`keys/tenant/{!tenant}/ses`), per-tenant backup keys, and per-user keys for services a person connects. None ever entered a box.

**The hole.** Homestead already depends on it. Permit's signing keys, barker's DKIM keys and provider credentials, the reply-token key barker shares with barn, backup encryption for pond and ranger, and every credential a box's fetch needs are all "by reference to the key store." Each service holding its own would scatter the most sensitive bytes in the system across six databases with six rotation stories.

**The job.** Hold every secret, encrypted at rest under envelope encryption; hand a secret only to the service allowed to use it, only over mutual TLS, and only into memory; rotate on schedule with old and new both valid through an overlap; audit every read to ranger. Per-tenant and per-user secrets (a tenant's own SES account, a person's calendar token) are scoped like everything else.

**Why its own service.** It is the one place where a bug is total, so it should be tiny, all R0, and changed rarely, like pond. Every caller gets a `cellar_client` and a fake.

**Other names.** strongbox, safe, lockbox.

### silo: files people upload

**Goober.** Uploads were their own flow: an upload row before any bytes, bytes to disk or S3, preprocessing by sniffed content type (images resized and stripped of GPS and metadata, archives checked for zip bombs), a mark-and-sweep that collected uploads no row adopted, and resumable uploads planned (human/060).

**The hole.** Store holds a site's own media and refuses product uploads. Pond refuses blobs and says a document holds a reference. Nothing accepts a profile photo, a receipt PDF, or a job's site photos, and nothing collects the orphans.

**The job.** Accept uploads through barn on a reserved path with a ticket a route minted; sniff, strip, resize, and refuse; store content-addressed bytes in object storage per tenant; let a box attach an upload to an object slot (the attach rides the request's pond write, so an upload is adopted if and only if the write landed); sweep the unadopted; serve downloads through barn under the route's permit requirement; delete a user's uploads when pond's deletions say so.

**Why its own service.** Uploads are big, slow, and adversarial bytes; keeping them out of barn keeps barn's memory and threads for boxes, and keeping them out of pond keeps the WAL small. Store's attachments could move onto silo later (store/IDEA Q9.6).

**Other names.** crib (a corn crib), bin, hayloft.

### hound: everything that goes out to the world

**Goober.** The host owned the network a box saw (human/030): named services, OpenAPI types, credentials attached on the way out, never visible to the box. Stability (human/330) went further: host capabilities with long-lived contracts (`send-email@1`) and swappable providers, and OpenAPI services that could be redirected to an internal replacement by configuration, so a product could outlive its vendors. Crawling (human/800) made the host a good robot: robots.txt, rate per site, identification, caching, and extraction on a schedule.

**The hole.** `barn_fetch` sends a box's request to an origin the box declared. It can't attach a credential (there is no cellar), cache, throttle per upstream, honor robots.txt, record and replay for shadows and tests, or redirect a dead vendor to a replacement. Every barn doing it alone means every barn holds credentials and no barn knows the total rate to an upstream.

**The job.** Be the one egress: barn sends a box's fetch to hound with the route's declared service name; hound attaches the credential from cellar, enforces per-upstream and per-tenant rates, caches what may be cached, records exchanges for shadow comparison and `route_try`, redirects a service to its configured provider, and for crawls, fetches politely on sun's schedule. Every exchange is a line in ranger's `egress` stream.

**Why its own service.** Credentials leave cellar to one place instead of every barn, and rates are counted in one place. Barker stays separate: mail has its own rules (suppression, DKIM, bounces) and is already designed.

**Other names.** wagon (it goes to town and back), scout, courier.

### porch: the owner's panel

**Goober.** An operator panel on an `ops.` subdomain, look-only, with a change log, every box, traffic and data flow views, a database explorer, sitemap, performance, agents, content, and logs (human/300, machine/028); approvals queued for a person (human/305); and a native mobile terminal into one domain (human/999).

**The hole.** HUMAN.md's whole bet is that the person sits at the boundary: reviews the promotion diff by its shape, watches the problem log, and decides what agents work on. Rancher, ranger, store, and sun all expose operator APIs and MCP tools, but there is nothing for a person to look at, and a person won't review a quarter's worth of change through curl.

**The job.** A site, served on each `ops.` domain and the platform's own, that reads only through the operator APIs (rancher, and through it ranger, sun, barker, store, gauge): the promotion review, the problem log, traces by request id, traffic and guardrails, dead letters, the site's pages, skills. Its writes are the few that are a person's: approve a promotion, roll back, redrive. It holds no state of its own beyond preferences.

**Why its own service.** It is the human surface, and it changes for reasons none of the platform services do. Building it as a store site plus a few boxes, served by barn like any tenant, would also prove homestead can build its own tools; the lean is to try that first.

**Other names.** stoop, window, lookout.

### gauge: metrics over time

**Goober.** `version_health` compared each route's failures and latency against the version it replaced, `perf_report` and the heat study (machine/048) measured where time went, and health was bounded by reading local access logs (machine/040 § 4).

**The hole.** Barn and sun push counts to rancher every few seconds, and rancher's guardrails need "error rate of this version over the last ten minutes against the last one." Ranger refuses to be metrics. Keeping a time series in rancher would put an always-growing, write-heavy store inside the control plane that is meant to stay small and boring.

**The job.** Take counter and histogram pushes from every service (per tenant, route, box version, table, upstream); keep them at a few resolutions with fixed retention; answer windowed queries for rancher's guardrails, porch's charts, and agents (`version_health` again, as a query). No alerting: problems stay rancher's.

**Why its own service.** Its write rate and storage shape are unlike every other service. Rancher keeps only what a guardrail needs right now, read from gauge.

**Other names.** almanac (if almanac isn't skills), weathervane, ledger.

### farmhand: running homestead itself

**Goober.** One host behind Caddy, with stalwart for mail and S3 for backups (human/099); nightly sealed backups of everything plus continuous database backups (human/663); operations reviews that asked whether a person could start, stop, upgrade, roll back, back up, restore, and keep the disk from filling (machine/044, machine/046).

**The hole.** Common.md says "a deploy knows which packages changed, and so which services' binaries changed, and restarts only those," but nothing does the deploying. Pond backs itself up; rancher, permit, sun, barker, store, and their SQLite files don't. Caddy needs to know which domains to get certificates for (rancher's domain registry), the CA needs to issue host certificates for the wire, and Stalwart needs configuration.

**The job.** Know the machines and which services run where; roll out a new build service by service in a safe order (pond last and rarely); back up every non-pond database and test restores on a schedule; issue and rotate wire certificates; answer Caddy's on-demand TLS "ask" from the domain registry; watch disks. It is the platform's operator, not a tenant's.

**Why its own service.** Every other service is ignorant of the others' existence beyond its clients; something has to hold the whole picture, and it shouldn't be rancher, which is per-tenant configuration.

**Other names.** foreman, hand, steward.

### creek: the bus and event routes

**Goober.** A bus that carried signals, not data: "this topic is now at seq N," coalesced per subscriber window, so pub/sub never went quadratic (human/040). Internal routes let one box call another's route directly, never from outside (human/039). Sources and event routes ran a box when data changed.

**The hole.** Pond's feed tells barn which keys changed, and SSE fan-out runs a box per change per group, which covers watching an object. There are no topics that aren't a bucket, no coalescing across many writers ("this room is now at N"), and no way to say "when a document lands in `orders`, run `timer:fulfil`" except a box remembering to schedule it in every write.

**The job.** Topics per tenant carrying only seqs, coalesced; event routes that rancher declares (a bucket or topic, a filter, a route to run) and creek fires through sun-like delivery to barns, at least once, keyed by the change's seq.

**Why its own service, maybe.** Event routes could be a feature of sun (fire on a change instead of a time) and topics a feature of pond's feed. The lean is to try that first and split creek out only if the two grow apart.

**Other names.** brook, ditch, bell.

### almanac: skills and playbooks

**Goober.** Skills (how to do one thing well) and playbooks (what to do when something happens), written by agents for roles, searchable and readable in ops, edited by the operator agent (machine/050); `whats_new` read from CHANGES.md so agents learned what a build added.

**The hole.** Store has `site_guide` for one job. The thousand features HUMAN.md expects agents to ship need the knowledge of how to ship them on homestead: deploy a box, put a table version, read a problem, write a route. Without a home, it lives in prompts nobody versions.

**The job.** Skills and playbooks per role, versioned, searchable over MCP, written by agents and approved by a person like any other promotion; `whats_new` per service per build; each problem code linked to the playbook that fixes it.

**Why its own service.** It is small enough to be a store site with a few routes, which is the lean; it is listed because nobody owns it.

**Other names.** primer, ledger, handbook.

### shed: source and builds

**Goober.** The host kept source: check out, change, build on the host, deploy, and the source stays next to what it built, so the next conversation starts where the last ended (human/521, machine/020). Code generators targeted C and Rust (human/110); host libraries (`time@1`, `ics@1`) gave boxes heavy lifting (machine/831).

**The hole.** Rancher accepts a module by hash. Where the module came from, which source built it, and how an agent with no laptop builds one, is unowned. WIT replaces goober's generators, but someone still has to own the box SDKs and the toolchain images.

**The job.** Projects with history; builds in a sandbox from a pinned toolchain (Rust and C to a WIT component), reproducible so a module's hash proves its source; the box SDKs and starter templates; uploads to rancher with the source revision attached, so a promotion diff can show source, not only hashes.

**Why its own service.** Builds are heavy and untrusted, and belong far from barn and rancher. A team with laptops and CI may never need it; agents without one will.

**Other names.** workshop, forge, smithy.

### rooster: telephony

**Goober.** A proposal: each tenant rents numbers from a carrier API; texts and calls arrive as routes, as mail does; boxes send texts through the outbox, as they send mail (machine/025).

**The hole.** Nothing.

**The job.** Barker's shape for SMS: numbers per tenant, sends from the outbox with limits, opt-out (STOP) as suppression, delivery receipts; inbound texts handed to barn as `sms` routes.

**Why its own service.** Carrier rules (registration, opt-out law, number ownership) are their own world. Folding it into barker as a second channel is the alternative, and reasonable if barker's queue and suppression carry over cleanly.

**Other names.** crier, bell, horn.

### stable: stateful boxes (not yet)

**Goober.** Workers versus services: a service box owned one key on exactly one host under a lease, kept state in memory as a cache, rehydrated from durable data on loss, and was found through a fleet registry (human/400).

**Today.** Barn is stateless on purpose, and the docs say correctness never depends on warm state. The `tenant` isolation level is the closest thing. Named here so it is a decision, not an accident: a game room, a collaborative document, or a live auction would want it.

**Other names.** pen, corral.

## Holes that aren't services

| hole | goober | where it belongs |
| --- | --- | --- |
| Runtime flags (freeze orders without a new version) | machine/039 | Rancher: a flag set per service, declared by the box, changed by an audited operator call, read through `barn-abi`. |
| Typed request bodies (validate before the box wakes) | human/036, human/037, machine/011 | Rancher routes: a body schema per route, checked by barn; WIT covers the box side. |
| One route, many audiences (JSON or HTML by `Accept`) | human/070 | Barn, rendering a box's JSON through a store component, the way `.shtml` route includes already do. |
| Box-to-box calls | human/039 | Barn: an `internal` route protocol, in-process, with a callers list on the route. |
| Host libraries (`time@1`, `ics@1`) | machine/831 | Barn: library imports beside `barn-abi@1`, each its own package and tier. |
| Caddy's on-demand TLS check | human/099, human/015 | Barn answers the ask from rancher's domain registry; farmhand configures Caddy. |
| Approvals of individual changes | human/305, machine/035 | Deliberately replaced by beta and bulk promotion; porch is where a person does it. |
| Installable apps (PWA) | machine/782 | Store: a manifest and service worker generated from `site.json`. |
| Quotas | human/390 | Covered: rancher sets them, pond and barn enforce them. Gauge would show them. |

## Kept out on purpose

- **The manifest.** Replaced by boxes that just exist and routes that decide (rancher).
- **Code generation from schemas.** Replaced by WIT bindings and table versions.
- **One process for everything.** The reason for this document.

## Suggested order

1. **cellar**, because permit and barker already reference it, and nothing else should be designed around a placeholder.
2. **porch** and **gauge**, because the owner's review and automatic rollback are homestead's whole safety story, and both need them.
3. **farmhand**, before a second machine or a first outage.
4. **hound** and **silo**, with the first product that calls a paid API or takes a photo.
5. **almanac**, as soon as more than one agent is shipping.
6. **creek**, **shed**, **rooster**, and **stable**, each when a real product asks.

## The invariants

- **Every job goober's host did has an owner, or a written reason it doesn't.**
- **A role referenced by another doc is never left undefined.** The key store is the first to fix.
- **A new service follows the same rules:** its own database, a `_client` and a fake, an API.md, a PACKAGES.md with tiers, and a farm name.
- **A hole that fits an existing service goes there first,** and splits out only when it grows apart.
