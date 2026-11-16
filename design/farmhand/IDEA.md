# Farmhand
Look at one machine, say what would change, then change it and prove it works
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's deploy script was one of its best tools. `just deploy you@server` built the binary, copied it, and ran a bootstrap that installed what the host needed. It checked the new config before anything restarted (the gate). It restarted and either got `ready` within a minute or put the previous build back (`.prev`). Then it checked the server through its public name, the way people reach it, and finished with an advisory audit that asked for whatever was missing. Every step that edited a config copied it first (`*.before-*`) and restored it if the result was refused. `--add-tenant` launched a site the same way (goober machine/046 §2, machine/028).

Homestead has more pieces: up to nine services, each its own binary with its own config, database, systemd unit, and certificate, plus Caddy, Stalwart, the cellar vault, and the deployment's CA. [Common](../common.md#isolation) already promises that "a deploy knows which packages changed, and so which services' binaries changed, and restarts only those." Farmhand is what keeps that promise.

**Farmhand is deployment tooling, not a service.** It runs from the operator's machine and reaches a server over SSH. It copies the build and a small `farmhand-agent` binary, and the agent works out **the diff of that one machine**: what the inventory says should be there, against what is there. Farmhand shows the plan. On `deploy`, it applies the plan the way goober did: gate, install, restart only what changed in a safe order, ready or roll back, check from outside, then audit.

### Why not a long-running service

- **Deploys are rare and a person's act.** They need root (systemd, Caddy, users, the vault key), and goober's deploy asked for the human's sudo password on purpose. A daemon with root on every machine, listening for instructions, would be the most valuable target in the system and would do nothing most of the time.
- **It must work when nothing else does.** The first install, a broken rancher, and a lost CA all happen before or outside the platform. A tool that needs the platform to fix the platform can't.
- **The agent exists only during a run.** `farmhand-agent` is copied, run over SSH, and exits. Nothing listens between deploys.

## The inventory

One file, `homestead.json`, kept in the operator's repository, describes the deployment. It is desired state, not a script:

```json
{
	"deployment": "prod",
	"machines": {
		"home-1": {
			"ssh": "deploy@home-1.example.net",
			"address": "10.0.0.2",
			"services": ["rancher", "pond", "permit", "sun", "barker", "ranger", "store", "barn"],
			"edge": { "caddy": true, "stalwart": true }
		}
	},
	"platform": { "ops-domain": "ops.homestead.example", "mail-domain": "mail.homestead.example" },
	"object-storage": { "endpoint": "https://s3.us-west-2.amazonaws.com", "bucket": "homestead-prod" },
	"backups": { "at": "03:15", "keep-days": 30 },
	"ca": { "dir": "~/.config/homestead/prod/ca" }
}
```

Single machine first, and every service on it, which is what the ports in [network](../common/NETWORK.md#ports) were chosen for. The inventory supports several machines from the start: a machine lists the services it runs, peers are computed from the inventory, and `farmhand deploy` takes one host at a time. Tenant domains are not in the inventory: they live in rancher's domain registry, and Caddy asks barn about them (see below).

## What one machine's diff covers

| resource | desired from | checked by the agent | applied by |
| --- | --- | --- | --- |
| Packages | A fixed list (Caddy, Stalwart, SQLite tools) | Installed versions | The package manager |
| Users and directories | One user per service; data, config, spool, and vault directories | Existence, owner, mode | Create or fix; **never delete a data directory** |
| Service binaries | The build, by build id | `$service --build-id` | Copy, keep the old one as `.prev` |
| Service configs | Rendered from the inventory (peers, ports, paths, object storage) | Content hash | Write after a `.before-deploy` copy |
| systemd units and timers | One unit per service; backup timers | Unit content, enabled, active | Write, `daemon-reload`, enable |
| Wire certificates | One per service per host, `homestead://$service/$host`, from the deployment CA | Issuer, SAN, days left | Issue on the operator's machine, copy, SIGHUP |
| The cellar vault key | The current (and during rotation, previous) wrap key | Present, 0400, owned by the service user, key id | Copy, SIGHUP |
| Caddy | One site block: on-demand TLS whose `ask` is barn's domain check, proxying to barns | Caddyfile hash, `caddy validate` | Write after `.before-deploy`, validate, reload |
| Stalwart | Mail in for the platform and tenant domains, handing to barn over LMTP | Config hash | Write, reload |
| Kernel settings | What goober learned, like multipath TCP off | `sysctl` values | Write a sysctl drop-in, apply |
| Firewall | Wire ports only from peers in the inventory; 80, 443, and 25 from anywhere | Rules | Apply |

The agent gathers facts read-only, and the plan is computed from facts and inventory by a pure function, so `plan` never changes anything and the same facts always give the same plan.

## A deploy

```
farmhand deploy home-1
  1. build      cmake --preset server (or use --build DIR)
  2. connect    ssh, copy farmhand-agent and the build to a fresh staging dir (mktemp -d)
  3. facts      the agent reads the machine
  4. plan       diff; shown; stop here unless --yes or the operator says yes
  5. gate       each changed service's new binary runs `check --config` on its new config;
                caddy validate; a refusal stops the deploy with nothing changed
  6. install    .prev and .before-deploy copies, then binaries, configs, units, certs, keys
  7. restart    only changed services, in order: ranger, rancher, permit, sun, barker, store,
                barns (one at a time, drained by failing health then GOAWAY), pond last
  8. ready      each restarted service answers ready within a minute, or the deploy rolls
                back everything it restarted to .prev and .before-deploy, and fails
  9. outside    through Caddy and public names: the platform ops domain, a tenant per
                readiness file, the TLS ask, mail in on 25
 10. audit      advisory findings and questions, as goober's audit; never fails the deploy
```

- **Pond restarts last, and rarely.** Pond holds the data, and its code changes least. A plan that restarts pond says so on its own line, and `deploy` asks for it separately (`--include-pond` without a terminal). Pond's own shutdown flushes and checkpoints its WAL (see [pond](../pond/IDEA.md#backups)).
- **Order follows dependencies.** Ranger first, so the others' spools drain into it. Rancher next, because barns won't serve without its snapshot. Barns before pond, so nothing waits on a pond that is restarting.
- **A restart drains.** Every service sends GOAWAY on SIGTERM and finishes what it accepted. A barn fails its health check first, so Caddy stops sending to it before it drains.
- **The run survives a dropped SSH.** The agent runs the apply in a transient systemd unit and keeps a journal on the machine. Reconnecting with `farmhand deploy` resumes from the journal or finishes the rollback. A lock file refuses a second concurrent deploy.

## A new tenant

`farmhand add-tenant HOST DOMAIN` is goober's `--add-tenant`, made smaller by on-demand TLS. Caddy issues a certificate for any domain barn says yes to, and barn says yes from rancher's domain registry, so adding a tenant edits no Caddyfile:

1. Ask for what's missing (tenant name suggested from the domain, first owner, postal address, `www.` redirect), or take them as flags without a terminal.
2. Show the DNS records the names need (A or AAAA to the edge, MX when mail in is on) and whether each resolves here yet.
3. Call rancher's operator tools, as a host operator, to create the tenant (and with it its pond database and permit realm) and register `DOMAIN`, `www.DOMAIN`, `beta.DOMAIN`, and `ops.DOMAIN`.
4. Wait up to a minute for each name's health through Caddy. A certificate that isn't there yet because DNS isn't pointed is a note, not a failure.

## Certificates

`farmhand certs` runs the deployment's CA on the operator's machine. The CA's private key never goes to a server.

- **One certificate per service per host,** with the SAN `homestead://$service/$host`, as [network](../common/NETWORK.md#security) requires. Each is valid 90 days and re-issued by any plan that finds fewer than 30 days left.
- **Services reload on SIGHUP**, so issuing never restarts anything.
- **`farmhand certs revoke HOST`** removes a host from every peer's allowed list on the next deploy, for a machine that is lost.

## Rotating the cellar wrap key

`farmhand keys rotate` is the practiced routine [cellar](../cellar/IDEA.md#rotation) describes:

1. `cellar keygen` makes a new wrap key pair on the operator's machine.
2. A deploy step copies the new private key into every host's vault beside the old one and sends SIGHUP, so every service holds both.
3. Farmhand publishes the new public key through rancher's operator tools, and rancher seals every new secret to it.
4. On one host holding both keys, `cellar reseal` re-seals every secret rancher holds. Rancher stores the results and publishes them, and a sweep re-seals pond's `_credentials`.
5. When rancher reports nothing sealed to the old key, a final deploy removes the old private key from every vault.

Each step is resumable, and `farmhand keys status` says which step a deployment is on.

## Backups of everything that isn't pond

Pond ships its own WAL and snapshots, and ranger archives its own days. The other databases (rancher, permit, sun, barker, store, ranger's index, and the CA's issued list) need the same care. Farmhand installs, per service, a systemd timer that runs `$service backup` at `backups.at`. That command uses SQLite's online backup API, seals the copy with cellar to the offline backup key, and puts it in object storage under `backups/$service/$host/$date.sealed`, with write-only credentials, as goober insisted.

- **`farmhand backup HOST`** runs them now.
- **`farmhand backup drill HOST SERVICE`** proves a restore works. It fetches the newest copy, opens it with the offline backup key on the operator's machine, restores it into a temp directory on the host, and runs `$service check-db` against it. The audit reports any service without a successful drill in 30 days.
- **`farmhand restore HOST SERVICE --at DATE`** is the real thing: stop the service, keep the current database as `.before-restore`, restore, start, and check ready.

## The audit

`farmhand audit HOST` is goober's audit, kept advisory: it never fails a deploy, and it asks for what's missing. Each finding offers yes, not now, or never, and a silenced check stays visibly silenced. It checks for:

- a platform operator, and their passkey;
- object storage reachable, with credentials that can't read or delete backups;
- a backup drill in the last 30 days for every service;
- the vault key present everywhere it should be, and no host still holding a retired key;
- certificates with more than 30 days left;
- disk free on every data directory (goober's 256 MiB floor), and the age of the oldest unshipped WAL segment;
- tenants without a postal address or a `www.` redirect;
- the clock synchronized.

## Open questions

1. **C++, or a script?** The lean is C++20 with `common_*`, like everything else, with the plan engine pure and tested against a fake host, so the deploy logic is held to the same standard as the services it deploys. A shell or Python script is faster to start and is what goober had, but it has no tests worth the name.
2. **Several machines at once.** `deploy` takes one host. A deployment with several needs an order across hosts (the edge last, the pond holder rarely). The lean is one host per command, with `farmhand deploy --all` running hosts in an order computed from the inventory.
3. **Who may run it.** Applying is a person's act, since it needs sudo. `farmhand plan` is read-only and could be run by an agent with a restricted SSH key. The lean is yes for `plan` and `audit`, no for everything else.
4. **Where the CA lives.** On the operator's machine is simple and keeps the key offline. Losing that machine loses the CA, so the CA directory needs its own sealed backup.

## The invariants

- **Plan never changes anything.** The diff is computed from read-only facts by a pure function.
- **The gate comes first.** Every new binary checks its new config before anything restarts.
- **Ready, or rolled back.** Every restarted service is ready within a minute, or everything the deploy restarted goes back to `.prev` and `.before-deploy`.
- **Only what changed restarts,** in dependency order, drained, with pond last and only when asked.
- **A deploy is proven from outside,** through public names, the way people reach the site.
- **The audit advises; it never fails a deploy.**
- **No data directory is ever deleted by farmhand.**
- **Nothing listens between deploys.** The agent runs only while a person runs farmhand.
- **Private keys for the CA and backups never live on a server.**
