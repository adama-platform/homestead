# Farmhand packages
A pure planner, a careful executor, and a thin layer of SSH between them
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Farmhand runs as root on the machines that hold everything, so its bugs are of two kinds. A wrong plan restarts the wrong thing or misses a change, and the gate and readiness checks usually catch it. A wrong **apply** can delete what it shouldn't, leave a machine half-deployed, or hand out a key or certificate it shouldn't. So the cut is:

- **Planning is pure** (`farmhand_inventory`, `farmhand_resource`, `farmhand_plan`): facts and inventory in, an ordered list of steps out, with no I/O. It is R1, tested exhaustively with fixtures, and shared by `plan` and `deploy`, so what a person reads is what runs.
- **Applying is small and R0** (`farmhand_apply`): it executes steps, journals them, and rolls them back. It refuses whole classes of action (deleting a data directory, writing outside the paths the inventory owns) whatever the plan says.
- **Keys and certificates are R0** (`farmhand_ca`, `farmhand_keys`), and run only on the operator's machine.
- **Everything that touches the real world** (reading facts, SSH, systemd, Caddy, the outside check, the audit) is R2 adapters behind interfaces, so the planner and the executor are tested against `farmhand_fakehost`.

Farmhand is homestead's own tool, so it uses `common_*` like any service (the lean of [the idea's](IDEA.md#open-questions) first question).

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `farmhand_inventory` | `homestead::farmhand::inventory` | Parse and validate `homestead.json`; compute peers, ports, and per-service rendered configs | `common_config`, `common_json` | R1 |
| `farmhand_facts` | `homestead::farmhand::facts` | The `Facts` type, and reading them on a host through a `Host` interface: files, hashes, owners, units, build ids, sysctl, packages, certs | `common_core`, `common_crypto` | R2 |
| `farmhand_resource` | `homestead::farmhand::resource` | Each resource kind (package, user, directory, binary, config, unit, cert, vault key, caddy, stalwart, sysctl, firewall): desired vs actual -> changes | `farmhand_inventory`, `farmhand_facts` | R1 |
| `farmhand_plan` | `homestead::farmhand::plan` | Changes -> an ordered plan: gate steps, install, restart order (ranger ... barns, pond last), reloads, rollback steps; the plan id; human and JSON rendering | `farmhand_resource` | R1 |
| `farmhand_apply` | `homestead::farmhand::apply` | Executes a plan's steps through the `Host` interface with a journal; `.prev` and `.before-deploy`; ready-or-rollback; resume; refuses forbidden actions | `farmhand_plan`, `farmhand_facts` | R0 |
| `farmhand_host` | `homestead::farmhand::host` | The real `Host`: filesystem, systemd (`systemctl`, `systemd-run`), package manager, sysctl, firewall, `caddy validate`, service `check` and `ready` | `farmhand_facts` | R2 |
| `farmhand_fakehost` | `homestead::farmhand::fakehost` | An in-memory `Host` with fault injection (a service that never gets ready, a full disk, a dropped connection mid-step) | `farmhand_facts` | R2 |
| `farmhand_agent` | `homestead::farmhand::agent` | The server side: the JSON-lines protocol on stdin and stdout, dispatching to `farmhand_host` and `farmhand_apply`, the lock file, the staging directory | `farmhand_apply`, `farmhand_host` | R1 |
| `farmhand_ssh` | `homestead::farmhand::ssh` | The operator side of the protocol, over the system `ssh` binary (no SSH library), copying the agent and the build | `common_core` | R2 |
| `farmhand_ca` | `homestead::farmhand::ca` | The deployment CA: create, issue `homestead://$service/$host` certificates, renew, revoke, the issued list | `common_crypto`, `common_tls` | R0 |
| `farmhand_keys` | `homestead::farmhand::keys` | The cellar wrap key rotation, as resumable steps; drives `cellar` and rancher's operator tools | `cellar_seal`, `common_http` | R0 |
| `farmhand_backup` | `homestead::farmhand::backup` | Backup timers, drills, and restores for the non-pond databases; opens backups with the offline key on the operator's machine | `farmhand_plan`, `common_blob`, `cellar_seal` | R1 |
| `farmhand_check` | `homestead::farmhand::check` | The outside check: through Caddy and public names, the TLS ask, mail on 25, readiness files | `common_http` | R2 |
| `farmhand_audit` | `homestead::farmhand::audit` | Advisory findings, questions, and silences | `farmhand_facts`, `farmhand_inventory` | R2 |
| `farmhand_tenant` | `homestead::farmhand::tenant` | `add-tenant`: questions, DNS checks, rancher's operator tools, the health wait | `common_http` | R2 |
| `farmhand_cli` | `homestead::farmhand::cli` | Commands, flags, terminal questions, exit codes | everything operator-side | R2 |
| `farmhand_main` | | The `farmhand` binary | `farmhand_cli` | R3 |
| `farmhand_agent_main` | | The `farmhand-agent` binary | `farmhand_agent` | R3 |

## Dependencies

```
              operator's machine                                 the server

   farmhand_main                                           farmhand_agent_main
        |                                                          |
   farmhand_cli ---------------- JSON lines over ssh ------> farmhand_agent
     |   |   |   |   |   \                                    |          |
     |   |   |   |   |    farmhand_ssh                  farmhand_apply  farmhand_host
     |   |   |   |   farmhand_tenant, farmhand_check            |          |
     |   |   |   farmhand_audit                                 |          |
     |   |   farmhand_backup                                    |          |
     |   farmhand_ca, farmhand_keys (-> cellar_seal)            |          |
     |                                                          |          |
   farmhand_plan  <------------ the same pure code -------------+          |
        |                                                                  |
   farmhand_resource                                                       |
     |          \                                                          |
   farmhand_inventory   farmhand_facts  <----------------------------------+
                              ^
                        farmhand_fakehost (tests)
```

## Quality seeds

### farmhand_apply (R0)

- **Promise.** A step runs only if it is in the plan it was given. Every step is journaled before and after. A failed readiness check puts every service the deploy touched back to `.prev` and `.before-deploy` and restarts them. A resumed run never repeats a step whose after-record exists. It refuses, whatever the plan says, to delete or truncate a data directory, to write outside the paths the inventory owns, or to restart pond without the plan's explicit pond step.
- **Tests.** Against `farmhand_fakehost`: a crash after every step, then resume; a service that never gets ready, at every position in the restart order; a full disk during install; a rollback that itself fails, which ends with exit 4 and a journal naming the mixed state; a plan containing a forbidden step.
- **Won't.** Compute a plan, read the inventory, or talk SSH.

### farmhand_ca (R0)

- **Promise.** A certificate is issued only for a service and host in the inventory, with exactly one `homestead://$service/$host` SAN, from the deployment's own CA, and the CA's private key never leaves the operator's machine.
- **Tests.** Issued certificates verify against the CA and fail against another deployment's. Wrong SANs are impossible to request. Revocation removes a host from every peer's allowed list in the next plan.
- **Won't.** Copy anything to a server (`farmhand_apply` does that), or issue wildcards.

### farmhand_keys (R0)

- **Promise.** Rotation proceeds only in order (generate, distribute, publish, re-seal, retire), each step is resumable, and the old private key is retired only when rancher reports nothing sealed to it.
- **Tests.** Interruption at every step, then resume; a host that missed the distribute step blocks retirement; re-sealed secrets open with the new key and not the old.
- **Won't.** Print a private key, or hold one longer than the step that needs it.

### farmhand_inventory (R1)

- **Promise.** A valid inventory places every service exactly once (barn excepted) and renders configs that each service's own `check` accepts. Unknown keys are refused.
- **Tests.** Gold inventories and their rendered configs; every placement mistake has its own message.
- **Won't.** Connect anywhere.

### farmhand_resource and farmhand_plan (R1)

- **Promise.** The same facts and inventory always give the same plan. A plan restarts exactly the services whose binary, config, or unit changed, in dependency order, with pond last and only as its own step.
- **Tests.** A fixture table of facts and inventory to expected plans, including drift, a first install on a bare machine, a certificate near expiry, and nothing to do. The JSON and human renderings agree.
- **Won't.** Execute anything.

### farmhand_agent (R1)

- **Promise.** One deploy at a time per machine (the lock), a fresh root-owned staging directory per run, removed at the end, and every request answered or failed by id.
- **Tests.** Two concurrent agents; a garbled request line; a dropped stdin mid-apply leaves the detached unit running and resumable.
- **Won't.** Decide what to change.

### farmhand_backup (R1)

- **Promise.** Every non-pond database is backed up nightly with SQLite's online backup, sealed to the offline backup key, and stored with credentials that can't read it back. A drill proves a copy restores and passes the service's `check-db`.
- **Tests.** Backup during writes; a drill on a corrupted copy fails and names the copy; restore keeps `.before-restore`.
- **Won't.** Touch pond's data, which pond backs up itself.

## How it's wired

Farmhand has two composition roots, one on each side of SSH:

- **`farmhand_cli`** (the operator's side) builds the inventory, the planner, the SSH session, the CA, the key rotation, the backup and audit tools, and an HTTPS client for rancher's operator tools, with an injected clock and filesystem, like a service's system. It holds no state between runs except the CA directory and the per-deployment key directories.
- **`farmhand_agent`** (the server side) builds the real `Host`, the executor, and the journal, reads requests until stdin closes, and exits.

There are no threads beyond the one waiting on SSH output and the detached apply unit. A run is: read the inventory, connect, stage the agent, ask for facts, plan, confirm, send the plan, follow progress, outside check, audit, disconnect.

## Review queue

| tier | packages |
| --- | --- |
| R0 | `farmhand_apply`, `farmhand_ca`, `farmhand_keys` |
| R1 | `farmhand_plan`, `farmhand_resource`, `farmhand_inventory`, `farmhand_agent`, `farmhand_backup` |
| R2 | `farmhand_facts`, `farmhand_host`, `farmhand_fakehost`, `farmhand_ssh`, `farmhand_check`, `farmhand_audit`, `farmhand_tenant`, `farmhand_cli` |
| R3 | `farmhand_main`, `farmhand_agent_main` |

Build order, so agents can work in parallel:

1. `farmhand_inventory` and `farmhand_facts` (with `farmhand_fakehost`), in parallel.
2. `farmhand_resource`, then `farmhand_plan`. `plan` can ship here, read-only, against a real machine through a minimal agent.
3. `farmhand_apply` and `farmhand_host`, in parallel, then `farmhand_agent` and `farmhand_ssh`.
4. `farmhand_ca`, `farmhand_backup`, `farmhand_check`, `farmhand_audit`, and `farmhand_tenant`, in parallel.
5. `farmhand_keys`, once cellar and rancher's secret tools exist.

## The invariants

- **One planner for `plan` and `deploy`.** What a person reads is what runs.
- **The executor refuses forbidden actions whatever the plan says.** No data directory is deleted, nothing outside the inventory's paths is written, and pond restarts only as its own step.
- **Every R0 package is tested against a fake host that fails at every step.**
- **The CA and backup private keys stay on the operator's machine.**
- **Farmhand speaks SSH and JSON lines, not the wire protocol,** so it works when the platform doesn't.
