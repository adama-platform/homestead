# Sun API
Fire subscriptions for barns, an intake from pond's outbox, and a control API for rancher
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Sun is a queue with a due time, and its API is a queue's API. Barns subscribe and are handed jobs as fast as they hand back credit; they acknowledge each job, or renew its lease while a long system job runs. Jobs arrive from two places: scheduled jobs from pond's outbox, which sun drains itself, and recurring schedules from rancher. Rancher also reads and redrives the dead letter box. Nothing else talks to sun, and sun talks to nothing in the data path.

## The surface

| method | kind | callers | changes state? | answers |
| --- | --- | --- | --- | --- |
| `sun.fire.subscribe` | stream | barn | yes (leases) | Fired jobs, one per message of credit |
| `sun.fire.ack` | call | barn | yes | The job's next state: done, retrying at a time, or dead |
| `sun.fire.heartbeat` | call | barn | yes | The lease's new `lease-until` |
| `sun.control.schedule.put`, `.drop` | call | rancher | yes | The recurring schedule and its next occurrence |
| `sun.control.dead.list`, `.read` | call | rancher | no | Dead jobs, with every attempt |
| `sun.control.dead.redrive`, `.discard` | call | rancher | yes | What moved |
| `sun.control.jobs.list` | call | rancher | no | Pending and leased jobs for a tenant and route |
| `sun.control.status` | call | rancher | no | Counts per tenant and route; lateness |
| `sun.control.tenant.delete` | call | rancher | yes | Jobs, schedules, and dead letters removed |

Sun also calls out:

| method | to | for |
| --- | --- | --- |
| `pond.outbox.lease`, `pond.outbox.ack` | pond | Schedule and cancel effects addressed `to: "sun"` (Q12.1) |
| `pond.deletions.subscribe` | pond | Tenant deletions, and users whose jobs carry their id (Q12.2) |
| `pond.ids.lease` | pond | Job ids |
| `rancher.health.report` | rancher | Counts per tenant and timer route, every few seconds |
| `rancher.problems.report` | rancher | Dead jobs, late fires, a barn failing what others succeed at |
| `ranger.ingest` | ranger | The `timers` stream |

## Firing: `sun.fire.subscribe`

A barn opens one stream to sun once it holds rancher's snapshot (barn serves nothing before then, timers included). The stream's message credit is the number of timer events the barn can run at once: it opens with a window, and sends one credit back each time it finishes a job.

```json
{ "barn": "barn-7", "window": 32, "tenants": null }
```

```json
{ "job": "4294968001", "tenant": "bobs_plumbing", "route": "timer:send_reminder",
  "payload": { "job_id": "4294967312" }, "attempt": 2, "kind": "scheduled",
  "due": 1762959600000, "fired": 1762959600410, "lease-until": 1762959900410,
  "deadline-ms": 10000, "lease": "4294968001/2" }
```

| field | meaning |
| --- | --- |
| job | The job's id. Also the idempotency key of every pond write the run makes. |
| tenant, route, payload | What to run. Barn resolves the route from its own config. |
| attempt | 1 on the first fire. |
| kind | `scheduled` (seconds, the route's limits) or `system` (long, renewed by heartbeat). |
| lease-until | When sun will hand the job to another barn if it hears nothing. |
| deadline-ms | How long the run may take: the route's limits for scheduled jobs, the schedule's deadline for system ones. |
| lease | The lease token: job and attempt. Acks and heartbeats name it, so a stale barn's ack for an old attempt is refused. |

- **One message of credit, one job.** Sun never sends a barn more jobs than it credited. Sun picks among barns with credit, preferring one that recently ran the tenant's jobs.
- **A dropped connection releases its leases at once.** Those jobs are due again immediately, and sun prefers a different barn.
- **`tenants`** is optional, for a barn pool dedicated to some tenants; empty means any.

## Acknowledging: `sun.fire.ack`

```json
{ "lease": "4294968001/2", "outcome": "failed", "reason": "box.trap",
  "request": "9f3c...", "version": "accounts@sha256:77ab02..." }
```

```json
{ "state": "retrying", "due": 1762959660410, "attempt": 2, "max-attempts": 5 }
```

| outcome | sun does |
| --- | --- |
| `done` | Completes the job. A recurring occurrence makes the next one. |
| `failed` | Makes the job due again after backoff (30 s doubling, at most 1 h), or dead-letters it at `max-attempts`. `reason` is a problem code. |
| `skipped` | Barn couldn't run it for a reason that isn't the box's (the route no longer exists, the tenant is suspended). Dead-lettered at once with the reason, so an operator sees it. |

| code | when |
| --- | --- |
| `conflict` | The lease isn't current: it expired and the job went elsewhere. The barn's run still wrote with the job's id as idempotency key, so nothing happened twice. |
| `not_found` | The job was cancelled or its tenant deleted while it ran. |

The ack is idempotent by lease token: a repeated ack answers the same state.

## Renewing: `sun.fire.heartbeat`

```json
{ "lease": "4294968105/1" }
```

```json
{ "lease-until": 1762961400000 }
```

Only `system` jobs may heartbeat. Each heartbeat sets `lease-until` to now plus the lease (at most 30 minutes), until the schedule's deadline, after which sun answers `deadline` and the next expiry retries the job. A barn heartbeats at a third of the lease.

## Intake: schedules from boxes

Under the lean of [Q12.1](../OPEN.md#q121-who-drains-the-outbox), sun drains its own slice of pond's outbox with `pond.outbox.lease { to: "sun" }`. If the answer goes the other way, barn calls `sun.jobs.schedule` and `sun.jobs.cancel` with the same payloads and keys.

```json
{ "key": "bobs_plumbing/4472/1", "tenant": "bobs_plumbing", "seq": 4472,
  "effect": { "op": "schedule", "route": "timer:send_reminder", "at": 1762959600000,
              "payload": { "job_id": "4294967312" }, "job-key": "reminder/4294967312",
              "user": "4294967311" } }
```

| op | sun does |
| --- | --- |
| `schedule` | Creates the job, or replaces the pending job with the same `job-key` in that tenant. A leased job with that key isn't interrupted; the new one waits behind it. |
| `cancel` | Removes the pending job with that `job-key`, if any. |

- **Durable before acknowledged.** Sun writes the job, then acks the element to pond.
- **Deduplicated by the element's key**, kept 30 days.
- **Elements for one tenant and job key are applied in commit order**, so schedule-then-cancel ends cancelled.
- **`user`** is the write's user id, kept on the job so a user's deletion removes their pending jobs.
- **A payload over the cap (4 KiB), a time more than a year out, or an unknown route shape** is acknowledged as refused and raised as `sun.schedule_refused`.

## Deletions

Under the lean of [Q12.2](../OPEN.md#q122-how-do-deletions-reach-permit-sun-and-barker), sun follows `pond.deletions.subscribe`: a user deletion removes pending and dead jobs carrying that user id; a tenant deletion removes the tenant's jobs, schedules, and dead letters. A leased job is allowed to finish; its writes go to pond, which refuses them for a deleted tenant.

## The control API

Only rancher calls `sun.control.*`, and operators and agents reach it through rancher's operator API.

### Recurring schedules

```json
{ "method": "sun.control.schedule.put", "tenant": "bobs_plumbing", "name": "nightly_digest",
  "route": "timer:nightly_digest", "every": "1d", "at": "03:00", "zone": "America/Los_Angeles",
  "jitter-ms": 600000, "deadline-ms": 3600000, "max-attempts": 3, "if-seq": 112 }
```

```json
{ "seq": 113, "next": 1762945200000 }
```

| field | meaning |
| --- | --- |
| every, at, zone | `every` is `1h`, `1d`, `1w`, or a cron-like `"mon,thu"`; `at` is local time in `zone`. |
| jitter-ms | Spread across a window so many tenants' sweeps don't fire together. Stable per tenant and schedule. |
| deadline-ms | How long one occurrence may run, renewed by heartbeat. Default 1 hour. |
| if-seq | Rancher's compare-and-set on the schedule. |

`schedule.drop` removes the schedule; an occurrence already leased finishes. Rancher re-applies its whole view on start, and every put is idempotent.

### The dead letter box

```json
{ "method": "sun.control.dead.list", "tenant": "bobs_plumbing", "route": "timer:send_reminder", "limit": 50 }
```

```json
{ "jobs": [{ "job": "4294968001", "route": "timer:send_reminder", "died": 1762963000000,
             "attempts": [{ "barn": "barn-7", "at": 1762959600410, "reason": "box.trap",
                            "request": "9f3c...", "version": "accounts@sha256:77ab02..." }] }],
  "next": null }
```

`dead.redrive {jobs | route, due?}` makes them due again with fresh attempts; `dead.discard {jobs | route}` removes them. Both take idempotency keys and answer counts.

## Problems

| code | when |
| --- | --- |
| `sun.job_dead` | A job reached the dead letter box. Fingerprint: tenant, route, reason. |
| `sun.fire_late` | Jobs are firing more than a minute after due: barns lack credit. |
| `sun.barn_failing` | A barn fails jobs that succeed on other barns. |
| `sun.schedule_refused` | An outbox element sun couldn't accept. |
| `sun.dst_shift` | Logged, not a problem: an occurrence was collapsed or shifted, written to the `timers` stream. |

## The schema, in part

```
namespace homestead.sun;

enum JobKind : ubyte { scheduled, system }
table Fired {
  job: uint64; tenant: string; route: string; payload: [ubyte];   // JSON bytes
  attempt: uint16; kind: JobKind; due: int64; fired: int64;
  lease_until: int64; deadline_ms: uint32; lease: string;
}
table FireSubscribe { barn: string; tenants: [string]; }

enum Outcome : ubyte { done, failed, skipped }
table Ack { lease: string; outcome: Outcome; reason: string; request: string; version: string; }
enum JobState : ubyte { done, retrying, dead }
table AckAnswer { state: JobState; due: int64; attempt: uint16; max_attempts: uint16; }

table Heartbeat { lease: string; }
table HeartbeatAnswer { lease_until: int64; }
```

## The invariants

- **One message of credit is one job.** A barn is never handed more than it said it could run.
- **Acks and heartbeats name the lease,** so a stale barn can't complete or extend someone else's attempt.
- **A dropped connection releases its leases at once.**
- **Jobs arrive from pond's outbox or from rancher.** Never from a barn's own word about a tenant.
- **A job is durable before its element is acknowledged to pond.**
- **Only rancher calls the control API.**
- **Every dead job keeps every attempt,** with barn, reason, request, and box version.
