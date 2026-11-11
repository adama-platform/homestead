# homestead
A SlopBox orchestrator, harvested from goober
------

Homestead is a small ecosystem for an agentic platform to target. It is a research project at core, and its job is the same one goober had: let agents (and people) write small, isolated WebAssembly boxes, put them on the internet behind HTTP, SSE, MCP, and email, and give them data to work with, without anyone having to trust what is inside a box. We call those boxes SlopBoxes, since we assume the code inside was written quickly, by a machine, and is wrong in ways nobody has noticed yet. The orchestrator's job is to make that fine.

## The stage we are in

Goober (history/goober-design) tried to do all of this in one host, behind one manifest. The manifest kept growing a section for every idea (strategies, schemas, identity, mail, timers, approvals, migrations, traffic), and the host had to understand all of it at once, so the manifest became the bottleneck for every change. Many of the individual ideas were good. Putting them all in one place was not.

Homestead is the harvest. We read goober, keep what held up, leave what didn't, and split what is left into layers that each do one thing and know nothing about the layers above them. Nothing here is built yet. Every design doc is an IDEA.md, written to be argued with.

| keep from goober | leave behind |
| --- | --- |
| The box: an isolated WASM module with no POSIX, reached through declared capabilities | The manifest that described everything a box would ever touch |
| A strict boundary, now only on the write path: a box merges into the objects its route resolved, or doesn't | A generated, typed boundary over every table a box could read or write |
| Per-domain identity realms and `/`-scoped role strings | Identity, grants, and requirements folded into the host and the manifest |
| Globally unique 64-bit ids, vended in ranges, strings in JSON | A database per strategy, many databases per tenant |
| The domain registry as the only source of a tenant | One host process that owns identity, mail, timers, data, and routing |
| Merges and append-only history; never migrate what happened | Schema migrations as a deploy step |
| Shadow and A/B traffic, refused route conflicts | Traffic decisions buried inside box deployment |
| C++, one package per directory, QUALITY.md per package, no globals | Generated code and checkers that had to understand the whole world |

## The layers

```
        clients (browsers, agents over MCP, mail)
                        |
                        v
                 Caddy (TLS, LB)
                        |
          +-------------+-------------+
          |             |             |
       +------+      +------+      +------+
       | barn |      | barn |      | barn |      many, stateless:
       +------+      +------+      +------+      run WASM, cache reads
          |  \          |  \          |  \
          |   \_________|___\_________|___\_____ subscribe to config
          |             |             |         \
          v             v             v          v
                         +-------+
                         |  sun  |  timers: fire at barns, retry, dead letter
                         +-------+
   +------------------------------------+   +-----------+   +---------+
   | pond                               |   | rancher   |   | permit  |
   |  buckets (MERGE / DELETE) -> WAL   |<--|  boxes,   |-->|  users, |
   |  tailers -> sqlite3 tables         |   |  routes,  |   |  tokens,|
   |  read-only SQL, checked by fork    |   |  tenants, |   |  grants |
   |  pubsub, id leases, names          |   |  problems |   +---------+
   +------------------------------------+   +-----------+   one machine
            one machine (for now)            one machine
```

| piece | what it is | scale | design |
| --- | --- | --- | --- |
| pond | The data layer. Cheap buckets of documents keyed by globally unique ids (shared across buckets), written only by MERGE and DELETE, made durable by a WAL, tailed into versioned per-tenant SQLite tables (including joins of several buckets), read by stateless read-only SQL, and published as a change feed. Pond also vends id ranges and binds names to ids. | One machine, scaled vertically; later a cluster where each tenant lives on exactly one machine. | [design/pond/IDEA.md](design/pond/IDEA.md) |
| fork | A library that parses SQL and checks it against a table permission set, so only read-only queries over granted tables and columns ever reach SQLite. | A library, linked into pond and rancher. | [design/fork/IDEA.md](design/fork/IDEA.md) |
| barn | The WASM host, and a pure evaluator of rancher's routes. Speaks HTTP, SSE, MCP, email, and timer events, routes each one to a box version, hands the box the objects its route allows through the iron write boundary, runs it at a configurable isolation level, and caches reads using pond's change feed. Logs every routing decision and counts failures per route. Proxies the control plane to rancher and permit. | Horizontal; any barn answers any request. | [design/barn/IDEA.md](design/barn/IDEA.md) |
| rancher | The control plane. Holds boxes (and the buckets each may write), routes (and with them every permit requirement, object, and grant), traffic policies (shadow, A/B, ramps), domains, tenants, and the problem log. Drives pond and permit, and signals barns. | One machine. | [design/rancher/IDEA.md](design/rancher/IDEA.md) |
| sun | Timers. Holds temporal jobs (one-shot from boxes, recurring from rancher), fires them at subscribed barns, and retries SQS-style: a 30-minute lease, retries on another barn, and a dead letter box. | One machine. | [design/sun/IDEA.md](design/sun/IDEA.md) |
| permit | Identity: authentication, accounts, sessions, and token-based grants per tenant, in its own SQLite database. The same grants secure identity and products. | One machine. | [design/permit/IDEA.md](design/permit/IDEA.md) |

Services talk to each other over one multiplexed binary protocol on TCP, with optional mutual TLS from a host certificate. Shared rules live in [design/common.md](design/common.md) (engineering), [design/common/TENANT.md](design/common/TENANT.md) (tenants), and [design/common/PROBLEMS.md](design/common/PROBLEMS.md) (elevating failures to agents).

## Repository map

| path | what is there |
| --- | --- |
| design/ | The homestead design, one directory per service, plus the common rules. |
| history/goober-design/ | Goober's design, kept for reference and harvesting. Inspiration only; nothing in it binds homestead. |
