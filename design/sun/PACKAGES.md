# Sun packages
How sun is cut so the one rule that matters, never lose a job and never fire early, sits in one package
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Sun is small, and most of it is ordinary: a database of jobs, an index by due time, a stream to barns, an intake from pond. A bug in almost any of it makes a job late, or makes a barn busier than its neighbor, and the next tick or the next lease expiry fixes it. Only one kind of bug can't be fixed by waiting: a job that is lost, fired before it was due, fired to two barns at once with both believed, or retried forever. That is the state machine of a job's life, and it gets one package, `sun_lease`, which is the only R0 package in sun.

Everything that decides *when* is pushed out of `sun_lease` into packages that propose times: `sun_recur` turns schedules into occurrences, `sun_queue` keeps what is due next, `sun_dispatch` picks a barn. `sun_lease` takes their proposals and enforces the invariants: a transition happens only from the state it expects, and every transition is durable before anyone hears about it. So a reviewer reading `sun_lease` in full has read everything that can lose a job.

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `sun_job` | `homestead::sun::job` | The job and schedule model: fields, caps (payload 4 KiB, a year out, 20 attempts), validation, the lease token. Pure. | `common_core`, `common_json` | R1 |
| `sun_lease` | `homestead::sun::lease` | The job state machine: pending → leased → done / pending (backoff) / dead; expiry; heartbeat; release on disconnect. Durable through a `JobStore` interface before it answers. | `sun_job`, `common_core` | R0 |
| `sun_db` | `homestead::sun::db` | SQLite schema and statements: jobs by due, leases by `lease-until`, keys per tenant, schedules, dead letters, intake dedupe keys. Implements `JobStore`. | `sun_job`, `common_sqlite` | R1 |
| `sun_queue` | `homestead::sun::queue` | The in-memory timer wheel over the next window of due jobs, refilled from `sun_db`. Proposes "due now"; never changes state. | `sun_job`, `common_core` | R1 |
| `sun_recur` | `homestead::sun::recur` | Recurring schedules to occurrences: `every`, `at`, zones through C++20 `std::chrono` tzdb, stable jitter, DST collapse and shift. Pure. | `sun_job`, `common_core` | R1 |
| `sun_dispatch` | `homestead::sun::dispatch` | Barn subscriptions and their credit; picks a barn for a due job (credit first, tenant warmth second, avoid the last failing barn). | `sun_job`, `common_core` | R1 |
| `sun_intake` | `homestead::sun::intake` | Drains `pond.outbox.lease {to: sun}` (Q12.1) and `pond.deletions.subscribe` (Q12.2); dedupes by element key; schedules and cancels through `sun_lease`. | `sun_lease`, `sun_db`, `pond_client` | R1 |
| `sun_control` | `homestead::sun::control` | The control API's logic: schedule put and drop, dead letter list, read, redrive, discard, status, tenant delete. | `sun_lease`, `sun_db`, `sun_recur` | R1 |
| `sun_report` | `homestead::sun::report` | Counts per tenant and route, lateness, the `timers` ranger stream, problem aggregation. | `sun_job`, `common_log`, `common_problem` | R2 |
| `sun_proto` | `homestead::sun::proto` | `sun.fbs` and generated code. | `common_wire` | R3 |
| `sun_server` | `homestead::sun::server` | Binds `fire.*` to `sun_dispatch` and `sun_lease`, `control.*` to `sun_control`; checks callers. | `sun_proto`, core | R2 |
| `sun_client` | `homestead::sun::client` | The `Sun` interface barn uses (`fire`, `ack`, `heartbeat`) and rancher uses (`control`), its wire implementation, and contract tests. | `sun_proto`, `common_wire` | R2 |
| `sun_fake` | `homestead::sun::fake` | An in-memory sun with a manual clock: schedule, fire to a credited subscriber, ack, expire. | `sun_client`, `sun_job` | R2 |
| `sun_system` | `homestead::sun::system` | The composition root. | everything above, `pond_client`, `rancher_client`, `common_config` | R2 |
| `sun_main` | | The binary. | `sun_system` | R3 |

```
                         sun_main
                            |
                        sun_system --------------------- pond_client, rancher_client, common_log
                       /    |     \
              sun_server  sun_intake  sun_control   sun_report
               /    \         |      /    |    \
      sun_proto  sun_dispatch  |     /   sun_recur  sun_db (implements JobStore)
                     |        sun_lease ----------------+
                     |       /                          |
                 sun_queue  /                            |
                     \     /                              |
                      sun_job  <---------------------------+
                         |
                   common_core, common_json
```

`sun_lease` depends on nothing but `sun_job` and `common_core`. It sees storage through `JobStore` and time through `Clock&`, so its tests run every transition under a manual clock against an in-memory store, and again against `sun_db`.

## Quality seeds

### R0

**sun_lease**
- *Promise:* a job is never lost: every job is pending, leased to exactly one lease token, done, or dead, and every transition is durable before it is answered or announced. A job is never leased before `due`. A lease token that isn't current can't complete, fail, or extend a job. A job reaches the dead letter box after `max-attempts` deliveries without `done`, never sooner and never later.
- *Tests:* every transition from every state, including the ones that must be refused; an expiry racing an ack; a disconnect releasing leases mid-run; a crash (fault-injecting store) between each durable step and its answer, then recovery; a property test that random sequences of schedule, fire, ack, heartbeat, expire, crash, and recover preserve the invariant above; heartbeat past a schedule's deadline.
- *Won't:* choose a barn, compute a due time, or know what a route is.

### R1

- **sun_job.** *Promise:* every job sun stores passes the caps, and a lease token names one attempt. *Tests:* each cap at its edge; tokens round-trip. *Won't:* hold state.
- **sun_db.** *Promise:* `JobStore` is transactional per transition; a job key is unique among pending jobs per tenant; indexes serve "next due" and "next expiry" without scans. *Tests:* `JobStore` contract suite (shared with the in-memory store); a million future jobs cost disk, not memory. *Won't:* decide transitions.
- **sun_queue.** *Promise:* every job due within the window is proposed no earlier than due and within a tick after; the window refills before it runs dry. *Tests:* manual clock; jobs added inside the current window; refill under a burst. *Won't:* change a job's state.
- **sun_recur.** *Promise:* each occurrence of a schedule fires once, whatever the clock does; jitter is stable per tenant and schedule. *Tests:* gold tables of (schedule, zone, from) → next occurrences across DST transitions in several zones (spring forward, fall back, zones with 30-minute shifts, a zone whose rules change); `every` grammar refusals. *Won't:* read the system clock or the system's zone.
- **sun_dispatch.** *Promise:* no barn is handed more jobs than its credit; a due job goes to some barn with credit if any exists. *Tests:* credit accounting under subscribe, disconnect, and resubscribe; fairness across barns; avoidance of a failing barn without starvation. *Won't:* lease (it asks `sun_lease`).
- **sun_intake.** *Promise:* each outbox element is applied once, in commit order per tenant and job key, and acknowledged to pond only after the job is durable. *Tests:* redelivery; crash between apply and ack; schedule then cancel; deletions. *Won't:* validate beyond `sun_job`.
- **sun_control.** *Promise:* every change is idempotent and re-applying rancher's whole view converges. *Tests:* put twice; drop a schedule with a leased occurrence; redrive and discard by route. *Won't:* reach a job except through `sun_lease`.

## How it's wired

`sun_system` builds, in order: the config and clock; `sun_db`; `sun_lease` over `sun_db`; recovery (`sun_lease` releases every lease whose barn connection is gone, which after a restart is all of them, making those jobs due now); `sun_queue`, filled from `sun_db`; `sun_recur` and the schedules; `sun_dispatch`; `sun_report`; the clients (pond, rancher); `sun_intake`; and last `sun_server`.

| thread | runs |
| --- | --- |
| network (Asio) | the wire, subscriptions, encode and decode |
| tick (one) | `sun_queue` ticks, expiries, recurring occurrence creation; proposes to `sun_lease` |
| store writer (one) | every `JobStore` transaction, so transitions are serialized and durable in order |

**Start.** Sun accepts `fire.subscribe` only after recovery and an id range from pond. **Shutdown.** GOAWAY; stop leasing; wait briefly for in-flight acks; release remaining leases durably (they become due now, for the next sun); stop intake (unacked elements return to pond); close the database.

## Review queue

1. **`sun_lease`**, read in full by a person on every change, with its property test.
2. **`sun_recur`**, by its gold tables: a change to an expected occurrence is a change a person reads.
3. **The other R1 packages**: QUALITY.md, headers, tests.
4. **R2 and R3** by agent review.

**Build order for parallel agents.** `sun_job` and `sun_proto` first. Then `sun_lease`, `sun_db`, `sun_queue`, `sun_recur`, `sun_dispatch`, and `sun_client` in parallel (`sun_lease` against an in-memory store until `sun_db` lands). Then `sun_fake`, `sun_intake`, `sun_control`, `sun_report`, `sun_server`. Last `sun_system` and `sun_main`. Barn's timer work starts against `sun_fake` as soon as `sun_client` exists.

## The invariants

- **`sun_lease` is the only R0 package,** and the only code that changes a job's state.
- **Packages that compute times only propose them.** `sun_queue`, `sun_recur`, and `sun_dispatch` never write.
- **Every transition is durable before it is answered.**
- **Time is injected.** No package reads the system clock or the system's zone.
- **`JobStore` has one contract suite,** passed by the in-memory store and by `sun_db`.
