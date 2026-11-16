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
