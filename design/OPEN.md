# Open questions
Every undecided question across the homestead design, in one place
------

> Status: collected 2026-11-11 from the open questions in every design doc. Questions that
> appeared in more than one doc are merged, and each lists its sources. Write your answer
> under **Answer:**. Once answered, the decision moves into the source docs and the question
> leaves this file.

Each question has a **lean**: what the docs currently assume if nobody answers. Leaving an answer empty means "go with the lean for now."

## 0. Terms

### Q0.1 What is the iron ABI?
*Source:* barn/IDEA.md. "Iron" doesn't appear anywhere in goober's design. The docs assume `iron@1` is a new, small, versioned boundary for the write path only: the route resolves object slots, and for each one the box reads it and then either merges, deletes (if allowed), or does nothing. Reads and everything else are the ordinary `barn-abi@1`.
*Lean:* as described.

**Answer:**

## 1. Engineering

### Q1.1 `hs::` or `homestead::`?
*Source:* common.md. The short one reads better in code; the long one can't collide with anything.
*Lean:* `hs::`.

**Answer:**

### Q1.2 One repository build, or one per service?
*Source:* common.md.
*Lean:* one CMake tree with every package, while nothing is deployed separately.

**Answer:**

### Q1.3 Which payload encoding for the internal protocol?
*Source:* common.md. FlatBuffers reads without copying and evolves by adding fields; Protocol Buffers is the most familiar; a hand-written codec avoids a dependency and a generator.
*Lean:* FlatBuffers, with our own framing and multiplexing on Asio.

**Answer:**

### Q1.4 Without TLS, is the handshake's service name enough?
*Source:* common.md. A shared secret per deployment would stop two environments from being wired together by accident, at the cost of one more setting.
*Lean:* service name only.

**Answer:**

### Q1.5 Component model (WIT) or a hand-written ABI?
*Source:* barn/IDEA.md. WIT gives bindings in many languages for free; iron is small enough to write by hand in C or WAT.
*Lean:* hand-written C header for `barn-abi@1` and `iron@1`, WIT later if languages demand it.

**Answer:**

## 2. Tenants and identity

### Q2.1 Is a tenant always a business?
*Source:* common/TENANT.md. One business running two products may want two tenants with one owner, which needs an owner identity above the tenant.
*Lean:* a tenant is a realm and a database; ownership across tenants waits.

**Answer:**

### Q2.2 Permit: one database, or one file per realm?
*Sources:* common/TENANT.md, permit/IDEA.md. One file keyed by realm is simple. A file per realm matches pond's "one tenant, one database," so deleting a tenant is deleting two files.
*Lean:* one database.

**Answer:**

### Q2.3 How large can one tenant get?
*Source:* common/TENANT.md. One file is simple until a tenant needs more than one machine's disk or write rate.
*Lean:* the tenant is the cluster's unit; very large tenants are a problem to meet when one arrives.

**Answer:**

### Q2.4 What does deletion promise?
*Sources:* common/TENANT.md, pond/IDEA.md (forgetting names). A WAL, backups, and `_names` remember what a deleted tenant or user forgot. Crypto-shredding with a per-tenant key makes tenant deletion final everywhere; replacing a forgotten name's string with a hash keeps its id stable and drops the data.
*Lean:* both, later.

**Answer:**

### Q2.5 Long lists of grants
*Source:* permit/IDEA.md. A user who edits ten thousand projects shouldn't make every request load ten thousand strings.
*Lean:* check a requirement against permit's index for the bound parameter, not the whole list.

**Answer:**

### Q2.6 Who sends mail?
*Source:* permit/IDEA.md. Email sign-in codes need a sender before homestead has a mail design.
*Lean:* a per-tenant provider configured in rancher, called by permit directly.

**Answer:**

### Q2.7 Operator sign-in
*Source:* permit/IDEA.md. Goober let host operators sign in on any domain and drop into a tenant.
*Lean:* keep it, and design audit and consent before building it.

**Answer:**

## 3. Data (pond and fork)

### Q3.1 Which merges?
*Source:* pond/IDEA.md. Is RFC 7386 enough to start, or do counters, sets, and lists need to come first so two writers adding to one list don't lose each other's work?
*Lean:* RFC 7386 first.

**Answer:**

### Q3.2 How long is history kept?
*Source:* pond/IDEA.md. The WAL is truncated once applied, and `_changes` is kept for a window. Keeping every write forever, as WAL segments in object storage, is valuable and is also a privacy problem.
*Lean:* a window only, for now.

**Answer:**

### Q3.3 What else can a tailer do?
*Source:* pond/IDEA.md. Exploding an array into child rows (a job's line items) is the obvious next shape after one row per id, and after joins of several buckets by id.
*Lean:* wait for a real need.

**Answer:**

### Q3.4 Does the feed filter by key?
*Source:* pond/IDEA.md. Subscribing to one id would let barn's SSE clients watch one entity without a bucket's whole feed.
*Lean:* buckets only to start.

**Answer:**

### Q3.5 Backups
*Source:* pond/IDEA.md. Tenant snapshots plus the WAL tail, shipped continuously (like Litestream) or nightly.
*Lean:* nightly snapshots plus continuous WAL shipping.

**Answer:**

### Q3.6 Which SQL parser?
*Source:* fork/IDEA.md. A hand-written recursive-descent parser for a small subset is easy to read and strict; SQLite's Lemon grammar is complete but describes far more than fork wants.
*Lean:* hand-written, with SQLite's grammar as the reference.

**Answer:**

### Q3.7 Does fork check against table definitions?
*Source:* fork/IDEA.md. The grant alone says what a box may read; the table's real columns also catch typos and types.
*Lean:* yes, in both rancher (at deploy) and pond (at query time).

**Answer:**

### Q3.8 Richer grants
*Source:* fork/IDEA.md. Scopes cover "only my rows." Masking a column, or exposing an aggregate but not the rows under it, is a richer model.
*Lean:* scopes only.

**Answer:**

### Q3.9 Window and JSON functions in SQL?
*Source:* fork/IDEA.md.
*Lean:* wait until a real box needs them.

**Answer:**

### Q3.10 Prepared, named queries?
*Source:* fork/IDEA.md. A route could register its queries at deploy, so rancher checks them once and pond prepares them once. Ad-hoc SQL stays available for agents.
*Lean:* later, as an optimization.

**Answer:**

## 4. Barn

### Q4.1 Objects a route can't know in advance
*Source:* barn/IDEA.md. A box writing a variable number of line items needs `new*` slots, or a slot over child ids listed in a parent object. Both widen iron.
*Lean:* wait for a real box.

**Answer:**

### Q4.2 How many reruns on a compare-and-set conflict?
*Source:* barn/IDEA.md.
*Lean:* a small bound (3), with repeated conflicts reported as a problem.

**Answer:**

### Q4.3 Where do routing logs go?
*Source:* barn/IDEA.md. Local files shipped to object storage are cheap; a log service agents can query is more useful.
*Lean:* every failure plus a sample of successes, shipped to object storage, with recent records queryable through rancher.

**Answer:**

### Q4.4 How does SSE fan out?
*Source:* barn/IDEA.md. A thousand clients watching one bucket shouldn't run a box a thousand times per change.
*Lean:* run the export once per change per distinct set of subscription parameters.

**Answer:**

### Q4.5 Is `reset` isolation practical?
*Source:* barn/IDEA.md. It depends on how well wasmtime restores linear memory from a snapshot.
*Lean:* `fresh` is the default until `reset` is measured.

**Answer:**

## 5. Control (rancher)

### Q5.1 Does rancher's state belong in pond?
*Source:* rancher/IDEA.md. A reserved `_rancher` tenant gives a WAL, a feed, and backups for free, and couples the control plane to the data plane's health.
*Lean:* rancher's own SQLite.

**Answer:**

### Q5.2 Automatic rollback?
*Source:* rancher/IDEA.md.
*Lean:* guardrails stop a ramp and raise a problem; a person or agent rolls back.

**Answer:**

### Q5.3 Who may change what?
*Source:* rancher/IDEA.md. Approvals for operators and agents: which changes, especially widenings of a box's `writes`, a route's objects, and grants, need a person.
*Lean:* widenings wait for a person; everything else goes.

**Answer:**

### Q5.4 Routes per tenant, or per domain pattern?
*Source:* rancher/IDEA.md. A product serving many tenants wants one route on `*`, with the tenant from the domain; a single business wants routes on its own domain.
*Lean:* both, with the specific domain winning over `*`.

**Answer:**

## 6. Timers (sun)

### Q6.1 Who may schedule what, and what can a payload bind?
*Sources:* rancher/IDEA.md, sun/IDEA.md. A route's `schedules` list lets its box schedule timer routes, and a timer route can resolve objects from payload fields (`{payload.job_id}`), so it trusts whatever route scheduled the job.
*Lean:* allowed, and rancher shows the whole chain ("PUT /jobs can schedule send_reminder, which merges into jobs") as one widening.

**Answer:**

### Q6.2 Long jobs
*Source:* sun/IDEA.md. A box runs for seconds; the 30-minute lease covers queueing and retries.
*Lean:* a job that needs longer schedules its own continuation.

**Answer:**

### Q6.3 Time zones and daylight saving
*Source:* sun/IDEA.md. A recurring schedule in a zone needs a rule for the hour that happens twice and the hour that doesn't happen.
*Lean:* fire once in a repeated hour, and at the next valid time for a skipped one.

**Answer:**

### Q6.4 A cluster of suns
*Source:* sun/IDEA.md.
*Lean:* partition jobs by tenant, like pond, when one machine isn't enough.

**Answer:**

## 7. Effects across services

### Q7.1 Writes, grants, and schedules in one request
*Sources:* permit/IDEA.md, sun/IDEA.md. A request can write to pond, grant a role in permit, and schedule a job in sun: three services, no transaction.
*Lean:* barn applies grants and schedules only after the pond write lands, each idempotent by request id. If "exists if and only if the write landed" becomes necessary, pond holds an outbox that permit and sun drain.

**Answer:**

## 8. Problems

### Q8.1 Does a tenant see its own problems?
*Source:* common/PROBLEMS.md. A business's owner or agent would benefit, and problems must not reveal the platform's internals.
*Lean:* not yet.

**Answer:**

### Q8.2 Do problems ever act?
*Source:* common/PROBLEMS.md.
*Lean:* only report; acting is a person's or agent's decision.

**Answer:**

### Q8.3 How long are problems kept?
*Source:* common/PROBLEMS.md.
*Lean:* resolved problems are compacted to counts after 30 days.

**Answer:**
