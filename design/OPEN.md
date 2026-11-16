# Open questions
Every undecided question across the homestead design, in one place
------

> Status: collected 2026-11-11 from the open questions in every design doc. Questions that
> appeared in more than one doc are merged, and each lists its sources. Write your answer
> under **Answer:**. Once answered, the decision moves into the source docs and the question
> leaves this file. 2026-11-13: the first round of answers moved into the source docs.

Each question has a **lean**: what the docs currently assume if nobody answers. Leaving an answer empty means "go with the lean for now."

## 3. Data (pond and fork)

### Q3.2 How long is history kept?
*Source:* pond/IDEA.md. The WAL is truncated once applied, and `_changes` is kept for a window. Keeping every write forever, as WAL segments in object storage, is valuable and is also a privacy problem.
*Lean:* a window only, for now.

**Answer:**


## 9. Store

### Q9.1 Does the `.shtml` name stay?
*Source:* store/IDEA.md. It is historical and agents know it; a `.html` page with `stitch: true` in its front matter would avoid a new extension.
*Lean:* `.shtml`.

**Answer:**

### Q9.2 Directives in a box's own HTML?
*Source:* store/IDEA.md. Goober let a box opt in with `"includes": true`.
*Lean:* components only, no live includes, so a box can't widen its reads through a page.

**Answer:**

### Q9.3 Can a component include one component?
*Source:* store/IDEA.md. Two levels would let a `card-grid` use `card`, at the cost of loop checks.
*Lean:* one level only.

**Answer:**

### Q9.4 A/B by page, or only by release?
*Source:* store/IDEA.md. A split on one page would let an agent test a headline without forking the site.
*Lean:* by release only.

**Answer:**

### Q9.5 Are route includes shared unless marked per-viewer?
*Source:* store/IDEA.md. Mirrors the SSE fan-out rule.
*Lean:* yes.

**Answer:**

### Q9.6 Where do attachment bytes live?
*Source:* store/IDEA.md. Store's own blob directory now; pond may grow object storage for uploads later, and the two could share it.
*Lean:* store's blob directory.

**Answer:**

### Q9.7 Is the built-in kit pinned per release?
*Source:* store/IDEA.md. The kit and built-in components are host code; pinning by hash means an upgrade never changes a deployed page.
*Lean:* yes, pinned by hash.

**Answer:**

## 10. Mail (barker)

### Q10.1 Who renders mail?
*Source:* barker/IDEA.md. Barn rendering at the call catches errors early; barker rendering at send keeps barn smaller.
*Lean:* render in both, against one template version.

**Answer:**

### Q10.2 Platform fallback for auth mail
*Source:* barker/IDEA.md. Sending a business's sign-in codes from the platform's domain is reliable but looks like someone else's mail.
*Lean:* fall back to the platform domain only while the tenant's domain is unverified.

**Answer:**

### Q10.3 Bulk mail at all?
*Source:* barker/IDEA.md. Goober grew categories and lists for outreach.
*Lean:* refuse bulk until a real product needs it.

**Answer:**

### Q10.4 DKIM keys: barker's or the provider's?
*Source:* barker/IDEA.md. Holding them makes switching providers seamless; SES Easy DKIM is less to run.
*Lean:* undecided.

**Answer:**

### Q10.5 Templates per language
*Source:* barker/IDEA.md.
*Lean:* wait for real need.

**Answer:**

### Q10.6 Bodies in the outbox?
*Source:* barker/IDEA.md. Template, vars, and version are smaller and keep personal text out of pond's history.
*Lean:* template, vars, and version only.

**Answer:**

## 11. Logs (ranger)

### Q11.1 Box logs
*Source:* ranger/IDEA.md. A box's own log text is tenant-authored and can carry personal data, which breaks "never tenant data."
*Lean:* a separate, shorter-lived stream with its own scope.

**Answer:**

### Q11.2 Sampling routing records
*Source:* ranger/IDEA.md.
*Lean:* one line per request; every failure plus a sample of successes if disk demands it.

**Answer:**

### Q11.3 Indexing
*Source:* ranger/IDEA.md. A full-text index makes grep fast and ranger much larger.
*Lean:* four indexed fields and bounded scans.

**Answer:**

### Q11.4 Searching archives
*Source:* ranger/IDEA.md.
*Lean:* restore a sealed day to search it.

**Answer:**

### Q11.5 A cluster of rangers
*Source:* ranger/IDEA.md.
*Lean:* partition by tenant, like pond, when one machine isn't enough.

**Answer:**

### Q11.6 Chain across hosts
*Source:* ranger/IDEA.md.
*Lean:* chains per stream per host.

**Answer:**

## 12. Network and packages

### Q12.1 Who drains the outbox?
*Source:* pond/IDEA.md says barns drain `_outbox` under a lease and call permit, sun, and barker; barker/IDEA.md says barker leases its own mail elements from pond. common/NETWORK.md needs one answer to fill in its method table.
*Lean:* each consuming service drains its own slice (`pond.outbox.lease {to: "sun"}`), as barker's doc says. Barn stays a pure evaluator with no delivery duty, the consumer that deduplicates is the one that leases, and permit, sun, and barker each need a `pond_client` anyway (for id ranges). Pond's doc and invariant would change from "delivered by barn" to "delivered by its consumer."

**Answer:**

### Q12.2 How do deletions reach permit, sun, and barker?
*Source:* common/TENANT.md and pond/IDEA.md say a user's deletion "flows through the same feeds," but the feed is per tenant and names keys, and those services keep their own copies keyed by user and tenant.
*Lean:* pond publishes one cross-tenant stream, `pond.deletions.subscribe`, of user and tenant deletions with seqs, which permit, sun, and barker follow. The alternative is rancher calling each service's `user.delete` in turn, which is simpler but makes rancher the place a missed deletion hides.

**Answer:**

### Q12.3 How does the mail receiver hand mail to barn?
*Source:* barn/IDEA.md and barker/IDEA.md say Stalwart hands inbound mail to barn, without saying how.
*Lean:* LMTP, on barn's port 2424, so a refusal from a mail route is an SMTP-level answer the sender sees. Bounces and feedback reports to the bounce address are routed by barn to `barker.bounce.ingest`.

**Answer:**

### Q12.4 Does `homestead_dev` exist?
*Source:* common/PACKAGES.md. A single binary running every service's system over real sockets on one machine makes the dev loop one config file, at the cost of a composition production never runs.
*Lean:* yes, built on `sim_world`, never deployed.

**Answer:**

### Q12.5 Is fork standalone?
*Source:* fork/PACKAGES.md. Fork could depend on `common_core`, like every other package, or on nothing but the standard library, so it builds and ships as a library outside homestead.
*Lean:* standalone: no `common_*`, its own CMake project, consumed by homestead like third-party code, with the namespace `fork::` rather than `homestead::fork::`.

**Answer:**

### Q12.6 How long is pond's deletions log kept?
*Source:* pond/API.md. `pond.deletions.subscribe` resumes by seq, so the log needs a window.
*Lean:* 30 days. A subscriber further behind gets `reset` and a `pond.deletions_reset` problem, since a missed deletion is a privacy failure someone must look at.

**Answer:**

### Q12.7 Are outbox effects opaque to pond?
*Source:* pond/API.md. pond/IDEA.md shows typed effects (`op`, `subject`, `grant`); the API carries an opaque body with a `to` and an ordering `scope`, and idempotency keys `seq/index`.
*Lean:* opaque, so pond never learns permit's, sun's, or barker's messages, and a new effect needs no pond change.

**Answer:**

### Q12.8 Box interfaces: WIT throughout, with effects in `iron@1`?
*Source:* barn/API.md. barn/IDEA.md lists C-style `iron_*(slot, ptr, len)` imports and no imports for grants, schedules, or mail. The API sketches WIT functions, and puts grant, revoke, schedule, cancel, and mail in `iron@1`, since they happen only if the write lands.
*Lean:* yes to both; barn/IDEA.md's import table moves to WIT.

**Answer:**

### Q12.9 Barn's per-request caps
*Source:* barn/API.md: 64 object slots, 1 MiB of patch per slot, 4 MiB per write, 32 effects, 32 queries, 8 fetches, SSE streams closed after 1 hour or when the caller's grants change.
*Lean:* adopt provisionally, tunable per route by operators.

**Answer:**

### Q12.10 Signing keys: per deployment or per realm?
*Source:* permit/API.md. permit/IDEA.md implies keys per realm; the API uses one key set per deployment, with the realm a signed claim barn checks against the domain's tenant.
*Lean:* per deployment, which keeps barn's key feed small; per-realm keys only if a realm must be exportable with its keys.

**Answer:**

### Q12.11 Keep capability and invite token text out of pond?
*Source:* permit/API.md. A `mint` effect could return the token text into the box's write, or return an id the holder exchanges once at `/_permit/`.
*Lean:* an id only, so a bearer secret never sits in pond's WAL, history, or backups.

**Answer:**

### Q12.12 Sun's `skipped` outcome
*Source:* sun/API.md adds `skipped` to `done` and `failed`, for a job whose route is gone or tenant is suspended; it dead-letters at once instead of retrying.
*Lean:* keep it.

**Answer:**

### Q12.13 How does beta see store's working tree?
*Source:* store/API.md. Beta serves the working tree, but barn only serves releases named in rancher's config log.
*Lean:* store auto-submits a debounced "working release" every few seconds, under its own config kind (`release.working`) so it doesn't crowd the log that promotions are reviewed from.

**Answer:**

### Q12.14 Partial promotions
*Source:* rancher/API.md. Promoting part of beta can leave a route pointing at a box or table version production doesn't have.
*Lean:* rancher refuses a partial promotion whose closure is incomplete and names what must come with it; only a person's session can approve one.

**Answer:**

### Q12.15 A barn that doesn't understand part of a release
*Source:* store/API.md. A newer store can compile a plan part an older barn doesn't know.
*Lean:* the barn refuses that page (not the release) and raises a problem, rather than skipping the part silently.

**Answer:**

### Q12.16 Who can lift a complaint suppression?
*Source:* barker/API.md.
*Lean:* only a host operator, never the tenant's own operator, since a complaint is the recipient's word.

**Answer:**
