# Sun
Temporal jobs, fired at barns, retried until they succeed or die
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's timers were part of the host. A box declared `by_timer` sources in its manifest, the host planned fires per box per tenant, and the cluster design had to partition timer work with consistent hashing so a thousand tenants' nightly sweeps didn't stampede one machine. Timers were tangled with data (a timer read a source table), with ownership (a partition had a lease), and with the manifest (a timer was a route with a strategy).

Sun is timers as their own service, and it is shaped like a queue: **register a job for a time, and when the time comes, sun hands it to a barn, which runs it and acknowledges it.** If the barn doesn't acknowledge within the job's lease, at most 30 minutes, sun gives it to another barn. If a job fails too many times, it goes to a dead letter box where an operator or agent can see why. That is Amazon SQS with a due time, and it is boring on purpose.

Sun knows tenants, timer route names, times, and payloads. It doesn't know boxes, buckets, or what a job does. Barn turns a fired job into a request on a timer route, and from there a timer is evaluated like any other request.

## The shape

```
   rancher ---- system timers (recurring) ---+
                                             v
   barn ---- scheduled jobs (pond outbox) --> +------------------+
                                              | sun (one machine)|
                                              |  jobs by due time|
                                              |  leases, attempts|
                                              |  dead letter box |
                                              +------------------+
                                                 |  fire (with credit)
                       +-------------------------+------------------------+
                       v                         v                        v
                    barn-1                    barn-2                   barn-3
                 run the timer route, then ack or fail the job
```

## A job

```json
{
	"job": "4294968001",
	"tenant": "bobs_plumbing",
	"route": "timer:send_reminder",
	"due": 1762959600000,
	"payload": { "job_id": "4294967312" },
	"key": "reminder/4294967312",
	"lease-ms": 300000,
	"max-attempts": 5
}
```

| field | meaning |
| --- | --- |
| job | An id from pond's counter. Also the idempotency key of every pond write the job makes. |
| tenant | Whose job it is. Set by barn or rancher, never by a box. |
| route | The timer route that runs it, by name. Rancher resolves the name to a service, an export, objects, and a grant. |
| due | When it may first fire. Sun fires no earlier than due, and as soon after as it can. |
| payload | A small JSON document, capped in size. Its fields can bind objects on the timer route. |
| key | Optional, per tenant. Scheduling the same key again replaces the pending job, and cancelling by key removes it, so "remind me about job 42" is one job however many times it is scheduled. |
| lease-ms | How long a barn has to acknowledge before the job is retried elsewhere. At most, and by default, 30 minutes; a route may set less. |
| max-attempts | How many deliveries before the job goes to the dead letter box. Default 5. |

## Registering jobs

A timer is a temporal route, and it comes in two forms:

| form | from | how | runs for | for |
| --- | --- | --- | --- | --- |
| Scheduled job | A box | A route lists the timer routes it may schedule (`"schedules": ["timer:send_reminder"]`). The box asks to schedule or cancel, by key; the request is an effect in pond's outbox, committed with the request's write, and a barn delivers it to sun. So a job exists if and only if the write that scheduled it landed. | Seconds, like any request. | One-shot jobs that follow from data: a reminder, a retry of an outbound call, an expiry. |
| System timer | Rancher | A recurring schedule on a timer route, like `"every": "1d", "at": "03:00", "zone": "America/Los_Angeles"`, for the tenant that has the route. | As long as it needs: minutes or hours. | Sweeps, reports, digests, maintenance. |

A timer route's objects may bind payload fields (`{payload.job_id}`), so it trusts the routes that may schedule it. Rancher shows that chain as one widening when a route's `schedules` list grows.

A recurring schedule is not a job; sun creates each occurrence as a job when it comes due. Many tenants' schedules at the same hour are spread with jitter across a window, so a thousand nightly sweeps are a thousand jobs over minutes, not a thousand at once.

**Daylight saving.** A schedule in a zone fires once per occurrence, whatever the clock does: in an hour that happens twice, it fires the first time only; for an hour that doesn't happen, it fires once at the next valid time. Either way sun writes a log record (to [ranger](../ranger/IDEA.md)) saying which occurrence was shifted or collapsed and why.

## Delivery

Barns subscribe to sun over the internal protocol, each saying how many timer events it can run at once (its credit). When a job comes due:

1. **Lease.** Sun picks a barn with free credit, increments the job's attempt, sets `lease-until = now + lease-ms`, and makes that durable.
2. **Fire.** Sun sends the barn the job. Barn resolves the timer route from its configuration from rancher and runs it like a request.
3. **Acknowledge.** Barn answers `done` or `failed` with a reason. `done` completes the job (a recurring occurrence schedules the next). `failed` makes the job due again after a backoff.
4. **Expire.** A job not acknowledged by `lease-until` is due again, and sun prefers a different barn. A barn whose connection drops has its leases released at once, rather than waiting the full lease.
5. **Dead letter.** A job whose attempts reach `max-attempts` without `done` moves to the dead letter box.

**Long system jobs.** A scheduled job runs in seconds, and its lease covers queueing and retries. A system job may run for much longer, so barn renews its lease with a heartbeat while it runs: the lease stays at most 30 minutes, so a barn that dies is noticed within one lease, but a healthy barn can keep a system job for as long as its schedule's deadline allows.

Delivery is at least once, and effects are at most once per job: every attempt writes to pond with the job's id as the idempotency key, so a retry after a lost acknowledgment can't apply its writes twice.

| setting | default | maximum |
| --- | --- | --- |
| lease | 30 minutes | 30 minutes, renewable by heartbeat for system jobs |
| scheduled job deadline | seconds, as the route's limits | the route's limits |
| system job deadline | 1 hour | set per schedule |
| attempts | 5 | 20 |
| backoff | 30 seconds, doubling | 1 hour between attempts |

## Load balancing

Sun spreads fires across barns by free credit, so a busy barn gets fewer jobs and a new barn starts taking them as soon as it subscribes. Sun may prefer a barn that recently ran jobs for the same tenant, which keeps barn's caches warm, but correctness never depends on it. A barn that keeps failing jobs that succeed elsewhere is reported as a problem.

## The dead letter box

A dead job keeps its payload, every attempt's barn, time, and failure reason, and the route and box version that ran it. Operators and agents can, through rancher's API (which proxies sun's):

- **Read** dead jobs by tenant and route.
- **Redrive** them: make them due now with fresh attempts, usually after shipping a fix.
- **Discard** them.

Every dead job is a problem record (see [problems](../common/PROBLEMS.md)), grouped by route and reason, so a route that starts dead-lettering is visible the moment it does.

## Failures per route

Sun counts, per tenant and timer route: jobs scheduled, fired, done, failed, expired, and dead-lettered, and lateness (how long after due a job fired). Barn counts the same per route and box version from its side. Both report to rancher, which uses them for health, ramp guardrails, and problems.

## Storage

Sun keeps its own SQLite database: pending jobs indexed by due time, deduplicated by the outbox element's idempotency key, leased jobs indexed by `lease-until`, recurring schedules, and the dead letter box. A job is durable before sun acknowledges scheduling it. In memory, sun holds the next window of due jobs in a timer wheel, refilled from the index, so a million future jobs cost disk, not memory. Sun's clock is injected, so tests run time by hand.

## Scale

Sun is one machine, and stays one machine. Tenants are bound to a single machine everywhere except barn's compute, and sun's work (jobs indexed by due time) is small next to pond's. If sun ever needs to split, it splits by tenant, like pond.

## What sun is not

- **Not a workflow engine.** A job is one fire of one route. A sequence of steps is a box that schedules its next job.
- **Not in the data path.** Sun never reads or writes pond; the timer route does.
- **Not precise to the millisecond.** Jobs fire no earlier than due, typically within a second after.
- **Not aware of boxes.** It knows route names; rancher and barn know the rest.

## The invariants

- **A job fires no earlier than due.**
- **A scheduled job exists if and only if the write that scheduled it landed.** It arrives through pond's outbox.
- **A job is leased to one barn at a time, for at most 30 minutes.** No acknowledgment means another barn gets it.
- **Delivery is at least once; effects are at most once.** Every attempt writes with the job's id as its idempotency key.
- **Each occurrence fires once.** A daylight saving shift never doubles or drops a fire, and is logged.
- **Too many failures means the dead letter box.** Never silent loss, never endless retries.
- **Every dead job is a problem.** Counted per route.
- **Sun never chooses a tenant.** Barn or rancher sets it, and a box never can.
