# Farmhand API
The command line, the inventory, the plan, and how farmhand talks to its agent
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Farmhand's API is a command line a person runs, an inventory file they keep, a plan they read, and exit codes a script can trust. Under it is a small protocol between `farmhand` on the operator's machine and `farmhand-agent` on the server, carried over SSH's stdin and stdout. Farmhand doesn't speak homestead's wire protocol, because it has to work before any certificate exists and while any service is broken. The one exception is that `add-tenant` and `keys rotate` call rancher's operator tools over HTTPS, like any operator.

## The commands

| command | changes the machine? | does |
| --- | --- | --- |
| `farmhand plan HOST` | no | Facts, then the diff, printed. |
| `farmhand deploy HOST [--yes] [--include-pond] [--build DIR]` | yes | Plan, confirm, gate, install, restart, ready or roll back, outside check, audit. |
| `farmhand rollback HOST` | yes | Puts every `.prev` and `.before-deploy` from the last deploy back, and restarts those services. |
| `farmhand add-tenant HOST DOMAIN [--tenant-name N] [--owner EMAIL] [--postal LINE] [--www yes\|no]` | through rancher | A new tenant and its domains, checked through Caddy. |
| `farmhand certs [issue\|renew\|revoke HOST\|list]` | on the next deploy | The deployment CA and per-service host certificates. |
| `farmhand keys rotate [--step N]`, `farmhand keys status` | yes | The cellar wrap key rotation, step by step and resumable. |
| `farmhand backup HOST [SERVICE]` | no (writes to object storage) | Runs backups now. |
| `farmhand backup drill HOST SERVICE` | a temp directory | Proves a restore works. |
| `farmhand restore HOST SERVICE --at DATE` | yes | Restores one service's database. |
| `farmhand audit HOST [--strict]` | no | Advisory findings and questions. |
| `farmhand facts HOST` | no | The raw facts, as JSON, for debugging. |

Every command takes `--inventory FILE` (default `./homestead.json`) and `--json`, which prints machine-readable output and asks nothing. Without a terminal, any question farmhand would ask is an error naming the flag that answers it, so a script never hangs.

### Exit codes

| code | means |
| --- | --- |
| 0 | Done, or for `plan`: nothing to change. |
| 1 | `plan` only: changes are pending. |
| 2 | Refused by the gate: a config, a Caddyfile, or a missing answer. Nothing changed. |
| 3 | Rolled back: something wasn't ready, and the machine is as it was. |
| 4 | Rollback failed: the machine is in a mixed state; the journal says exactly where. |
| 5 | Can't reach the machine, or another deploy holds the lock. |
| 6 | `audit --strict` only: findings remain. |
| 64 | Bad arguments or inventory (`EX_USAGE`). |

## The inventory

`homestead.json`, strict like every homestead config: defaults for everything, unknown keys refused.

| field | meaning |
| --- | --- |
| `deployment` | A name, like `prod`. Certificates, backups, and keys are kept per deployment. |
| `machines.$name.ssh` | `user@host` for SSH. The user needs sudo. |
| `machines.$name.address` | The private address peers dial. |
| `machines.$name.services` | Which services run there. Each service runs on exactly one machine, except barn, which may run on many. |
| `machines.$name.edge` | `caddy` and `stalwart`: whether this machine faces the internet. |
| `platform.ops-domain`, `platform.mail-domain` | The platform's own names. |
| `object-storage` | Endpoint and bucket for backups, WAL shipping, archives, and silo. Credentials are cellar secrets, not inventory. |
| `backups.at`, `backups.keep-days` | When the nightly timers run, and how long copies are kept. |
| `ca.dir` | Where the CA lives on the operator's machine. |
| `services.$service` | Optional overrides of a service's rendered config, which farmhand passes through after the service's own `check`. |

Farmhand checks the inventory before connecting: every service placed exactly once (barn excepted), names valid, peers reachable on the private network, at least one edge machine.

## The plan

The human form, from `farmhand plan home-1`:

```
home-1 (deploy@home-1.example.net), build 2026-11-16.3 (was 2026-11-14.1)

  ~ binary   barn       2026-11-14.1 -> 2026-11-16.3     restart (drain)
  ~ binary   rancher    2026-11-14.1 -> 2026-11-16.3     restart
  = binary   pond       unchanged
  ~ config   barn       peers.store added
  + unit     store-backup.timer
  ~ cert     permit     19 days left -> reissue           reload
  ~ caddy    Caddyfile  ask endpoint path changed          validate, reload
  ! sysctl   net.mptcp.enabled = 1 (want 0)

  restarts: rancher, barn          reloads: permit, caddy
  pond: not restarted
```

`+` adds, `~` changes, `-` removes (never a data directory), `!` corrects drift that someone made by hand.

The `--json` form is the same plan, one object:

```json
{
	"host": "home-1",
	"build": { "from": "2026-11-14.1", "to": "2026-11-16.3" },
	"changes": [
		{ "kind": "binary", "target": "barn", "from": "2026-11-14.1", "to": "2026-11-16.3", "effect": "restart", "drain": true },
		{ "kind": "config", "target": "barn", "summary": "peers.store added", "hash": { "from": "9a1c...", "to": "44e0..." } },
		{ "kind": "unit", "target": "store-backup.timer", "op": "add" },
		{ "kind": "cert", "target": "permit", "days-left": 19, "effect": "reload" },
		{ "kind": "caddy", "target": "Caddyfile", "effect": "reload", "validate": true },
		{ "kind": "sysctl", "target": "net.mptcp.enabled", "from": "1", "to": "0", "drift": true }
	],
	"restarts": ["rancher", "barn"],
	"reloads": ["permit", "caddy"],
	"pond": "unchanged"
}
```

A plan has an id (a hash of the facts and the inventory). `deploy` refuses to apply a plan whose facts changed between showing and applying, and plans again.

## The agent protocol

```
operator machine                                      server
farmhand  --ssh-->  sudo -n /var/lib/homestead/farmhand/stage/XXXX/farmhand-agent --stdio
          <-- { "hello": { "agent": "2026-11-16.3", "host": "home-1" } }
          --> { "id": 1, "facts": {} }
          <-- { "id": 1, "ok": { ...facts... } }
          --> { "id": 2, "apply": { "plan": "p-7c21...", "steps": [ ... ] } }
          <-- { "id": 2, "progress": { "step": 4, "of": 11, "doing": "gate barn" } }
          <-- { "id": 2, "ok": { "journal": "/var/lib/homestead/farmhand/journal/2026-11-16T18-02Z" } }
```

- **JSON lines over stdin and stdout.** Readable when something goes wrong, and needing nothing but SSH. Stderr carries the agent's own log.
- **Requests:** `facts`, `apply`, `resume`, `rollback`, `check` (run one service's `check --config`), `ready` (wait for one service), `backup`, `restore`, `drill`.
- **The agent is staged per run** in a fresh `mktemp -d` directory owned by root, which fixes goober's fixed `/tmp` race (goober machine/013 S4-7), and removed at the end.
- **`apply` runs detached** in a transient systemd unit (`systemd-run`), writing every step to the journal before and after doing it, so a dropped connection can be resumed with `resume` or undone with `rollback`.
- **The agent never decides.** It reports facts and executes steps it was given. The plan is computed on the operator's machine, in the same pure code that `plan` runs.

## Readiness

Each service answers readiness the same way, so the agent needs no per-service knowledge:

| service | ready means |
| --- | --- |
| every wire service | `$service ready` on the host: a local HELLO and `status` call that answers ok |
| barn | `GET /_barn/health` is 200, which needs a rancher snapshot, compiled boxes, and an id range ([barn](../barn/IDEA.md#learning-what-to-serve)) |
| Caddy, Stalwart | Their own status commands, then the outside check |

## The invariants

- **Every command without a terminal either acts on flags or fails naming the missing flag.** Nothing hangs.
- **`plan` and `--json` describe the same changes,** and a plan whose facts moved is never applied.
- **The exit code says whether the machine changed,** and whether it is in a known state.
- **The agent only reports and executes.** Plans are computed on the operator's machine.
- **Every apply step is journaled before and after,** so any run can be resumed or rolled back.
