# Problems
Elevating what went wrong to an agent that can fix it
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Homestead moves a lot of checking to runtime on purpose. A box's ruleset bounds its buckets at deploy, but the route decides its objects per request. Tables are tailed asynchronously from documents nobody schema-checked. Grants are checked by fork when a query arrives. That trade keeps every layer small, and it means failures show up later and in more places: a document whose field doesn't fit a table, a merge to a read-only slot, a query naming a column the grant doesn't have, a tailer that can't keep up.

A failure an agent can't see is a failure nobody fixes. So every service elevates what goes wrong as a **problem**: a structured, grouped, deduplicated record that says what broke, where, how often, and what would fix it, readable by an agent over MCP. Logs are still kept, for people. Problems are for the thing that will write the next version.

## A problem

```json
{
	"problem": "tailer.type_mismatch",
	"fingerprint": "users@1/zip/string",
	"tenant": "bobs_plumbing",
	"source": "pond",
	"where": { "table": "users@1", "column": "zip", "bucket": "addresses" },
	"expected": "int64",
	"found": "string",
	"count": 312,
	"first-seen": 1762870000000,
	"last-seen": 1762874400000,
	"examples": [{ "bucket": "addresses", "id": "4294967311", "seq": 4471, "shape": { "zip": "string" } }],
	"caused-by": { "box": "accounts@sha256:77ab02...", "route": "PUT bobsplumbing.com/me/address" },
	"hint": "Column zip in users@1 is int64, but 312 documents in addresses hold a string. Declare zip as text in a new version of users, or write numbers.",
	"state": "open"
}
```

| field | meaning |
| --- | --- |
| problem | A stable code from a fixed catalog, like `tailer.type_mismatch` or `iron.slot_read_only`. |
| fingerprint | What makes two occurrences the same problem, so a million failures are one record with a count. |
| where | The table, column, bucket, route, slot, or box export involved, by name. |
| examples | A few occurrences, with ids, seqs, and request ids, and the *shape* of the data, not its values. |
| caused-by | The box version and route that wrote the data or ran the code, when it is known. |
| hint | A sentence that says what would fix it, written for an agent. |
| state | open, acknowledged, or resolved. A resolved problem that happens again reopens. |

## Where problems come from

| source | examples |
| --- | --- |
| pond | A field of the wrong type for a table; a tailer falling behind; a hold or wait timing out; a write refused for a stale `if-seq` or a dead key. |
| fork | A query naming a table, version, column, or function its grant doesn't have; SQL outside the accepted subset. |
| barn | A trap, a fuel, memory, or deadline limit; a merge to a read-only slot; a compare-and-set conflict on a hot object; a fetch to an undeclared address. |
| permit | A requirement nobody can satisfy; a parameter an object uses that the requirement doesn't bind. |
| rancher | A route that would give a box a write outside its ruleset; a guardrail breached during a ramp. |

## How they flow

1. **Report.** Each service reports problems to rancher in batches, already grouped by fingerprint, so a hot failure costs a counter, not a message per request.
2. **Group.** Rancher keeps the problem log in its own database, one record per fingerprint, with counts and the latest examples.
3. **Read.** Operators and agents read problems through rancher's API and MCP tools: list by tenant, box, route, or table; read one; subscribe to new ones.
4. **Act.** A problem names what caused it, so the agent that owns the box version can read the problem, change the code or put a new table version, try it, and ship it.
5. **Resolve.** A problem is resolved by a person or agent, or automatically when the version that caused it is no longer live and it stops happening.

## Keeping problems

Open problems are kept as long as they are open. A resolved problem keeps its examples for 30 days, long enough to compare the version that caused it with the one that fixed it, and is then compacted to its counts, first and last seen, and the versions involved.

## Rules

- **Shapes, not values.** A problem never carries a document's data, a query's parameters, or a request body. It carries ids, seqs, request ids, and types. An operator can look up the rest with the access that requires.
- **Problems are bounded.** Every service caps the fingerprints it tracks, and over the cap, reports one problem saying so.
- **Every refusal is a problem.** If a service refuses something at runtime that a deploy-time check could have caught, it is reported, so the check can be added.
- **Problems only report.** A problem never acts on its own: it doesn't slow a route, roll back a version, or pause a tailer. Acting is a person's or agent's decision (a ramp's guardrails, which do roll back, are rancher's, not the problem log's).
- **Problems are for operators and their agents.** Tenants don't see their own problems yet, since a problem can reveal the platform's internals.
- **The catalog is fixed.** A new kind of problem is a new code with a documented meaning and hint, so agents can learn the catalog.

## The invariants

- **Nothing goes wrong silently.** Every runtime refusal and failure is a problem.
- **A problem names its cause.** Where, what, how often, and which box version and route, when known.
- **Problems carry shapes, not data.**
- **One fingerprint, one record.** Counts grow; the log doesn't.
