# Fork
Parse the SQL, check it against what was granted, and refuse the rest
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Pond lets boxes ask real questions of their data in SQL, which is the language every agent already writes well. That is only safe if pond knows, before SQLite sees a query, that the query only reads, and only reads what the box was granted. Fork is that check. It is a library that parses a query into a tree, walks the tree against a table permission set, and answers either a verdict with a canonical query to run, or a refusal that says why.

The write path never goes through SQL. Pond owns every write, through MERGE and DELETE on buckets, so the SQL fork accepts is only ever a read. That makes fork's job narrow enough to be done well: it doesn't have to understand INSERT, UPDATE, triggers, or transactions, only SELECT, and only a subset of it.

## Where fork runs

| caller | when | why |
| --- | --- | --- |
| pond | On every query, before it is prepared | The real gate. Barn is trusted infrastructure, but a box wrote the SQL. |
| rancher | When a route is put, or a table version changes | To tell an operator or agent, at deploy time, that a route's grant names a table, version, or column that doesn't exist, that a route's named query is refused, or that dropping a version would break a route. |
| barn (optional) | Before sending a query to pond | Only as an optimization: refusing early, and keying its cache by the canonical query. Never as the gate. |

## A permission set

A permission set (a grant) is the list of tables, versions, and columns a box may read in the tenant's database, plus limits. **Routing owns grants**: each route carries one (see rancher), barn sends it to pond with every query, and fork enforces it there. The same box can therefore read `jobs@1` on one route and `jobs@2` on another.

```json
{
	"tables": {
		"jobs": { "version": 2, "columns": ["id", "status", "customer_id", "finished_at"] },
		"customers": {
			"version": 1,
			"columns": ["id", "name"],
			"scope": "owner_id = :!user_id"
		}
	},
	"functions": ["count", "sum", "min", "max", "lower", "coalesce"],
	"limits": { "max-rows": 1000, "max-ms": 50, "max-joins": 4 }
}
```

| field | meaning |
| --- | --- |
| tables | The only tables a query may name, the version of each it reads, and the only columns it may read. `"*"` is not allowed; columns are listed. The box writes `jobs`; fork rewrites it to the physical table for `jobs@2`. |
| scope | Optional. A filter fork attaches to every reference to that table, so a box only ever sees the rows the scope allows. See [Scopes](#scopes). |
| functions | The functions a query may call, from a fixed list fork knows to be pure and cheap. |
| limits | Caps pond enforces when it runs the query: rows returned, time spent, and the size of the query itself. |

## Table definitions

Fork checks a query against the grant and against the table's real definition: the columns the tailer defines, with their types. The grant says what a box may read; the definition catches a column that was granted but doesn't exist, a typo, and a comparison between incompatible types. Rancher has both at deploy time, and pond has both at query time, so fork runs with both in each place.

## What fork accepts

A single statement, from a SELECT subset written in a common dialect: the SQL that SQLite, PostgreSQL, and MySQL all read the same way. Fork accepts only what is portable, and prints canonical SQL that it can render for each, so a table that someday moves to a bigger database keeps its queries. SQLite is the only target today.

- `SELECT` with named columns, expressions, aliases, and `DISTINCT`.
- `FROM` with tables, inner and left joins, and subqueries.
- `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, and `OFFSET`.
- `WITH` for non-recursive common table expressions.
- Bound parameters only, written `:name` or `?`. Reserved parameters, like `:!user_id` and `:!tenant`, are filled by barn and can't be bound by a box.

Everything else is refused: every statement but SELECT, `PRAGMA`, `ATTACH`, recursive CTEs, window functions, JSON functions, syntax only one engine understands, any function not in the grant, any table or column not in the grant, `*` (because it would read columns that weren't granted), and any literal string over a size limit.

## The verdict

```json
{
	"ok": true,
	"sql": "SELECT j.id, j.status FROM \"jobs@2\" AS j WHERE j.status = ?1 ORDER BY j.finished_at DESC LIMIT 1000",
	"params": ["status"],
	"tables": ["jobs@2"],
	"columns": { "jobs": ["id", "status", "finished_at"] }
}
```

Fork never hands pond the text a box wrote. It prints the tree back out as canonical SQL, which means:

- **What runs is what was checked.** A trick the parser and SQLite read differently can't survive the round trip, since SQLite only ever sees fork's output.
- **Limits are written in.** The grant's `max-rows` becomes a `LIMIT`, or lowers the one the box wrote.
- **Versions are resolved.** The box names tables by their logical name, and fork binds each to the version its route granted, so table versions can change under a box without the box changing.
- **The tables a query reads are known.** Pond uses them to wait for exactly the tailers the query depends on, and barn uses them to know which changes invalidate a cached answer.
- **Equal queries look equal.** Canonical text is a good cache key.

A refusal names the rule and the place, and is reported as a problem (see [problems](../common/PROBLEMS.md)) with the route and box version that sent it, so an agent sees a refused query as something to fix:

```json
{ "ok": false, "error": "column_not_granted", "table": "customers@1", "column": "email", "at": 31,
  "hint": "customers@1 is granted columns id, name. Add email to the route's grant, or stop selecting it." }
```

## Scopes

A scope is a WHERE clause the grant owns. Fork replaces every reference to a scoped table with a subquery that applies it, so the box's query can't avoid it with a join, an alias, or a subquery of its own:

```sql
-- the box wrote
SELECT c.name FROM customers AS c
-- pond runs
SELECT c.name FROM (SELECT id, name FROM customers WHERE owner_id = :!user_id) AS c
```

Scopes may only use reserved parameters and literals, since a scope that took a box's parameter would be a filter the box controls.

Grants are row-centric: a scope decides which rows, and the column list decides which columns, and that is the whole model. There is no column masking or aggregate-only access. When data needs different isolation (a public view of a profile beside a private one), a tailer splits it into a separate table with only the columns that audience may see, and a route grants that table.

## Named queries

A route can register its queries by name (see [pond](../pond/IDEA.md#named-queries)). Rancher runs each through fork when the route is put, and stores the verdict's canonical SQL with the route, so a named query is checked once, prepared once by pond, and run by name with parameters. A named query whose table version changes is checked again; one that no longer passes blocks the change. Ad-hoc SQL is still checked on every call.

## Two locks

Fork is the first lock. Pond still runs every query with the second:

- **A read-only connection**, opened with `SQLITE_OPEN_READONLY` and `PRAGMA query_only`.
- **An authorizer** (`sqlite3_set_authorizer`) that allows only reads of the tables and columns fork said the query uses, and denies everything else.
- **`sqlite3_stmt_readonly`** checked after prepare.
- **A progress handler** that stops a query at the grant's `max-ms`.

If fork has a bug, SQLite's own view of the statement still has to agree before a single row is read.

## Packages

| package | does |
| --- | --- |
| fork_ast | The tree: statements, expressions, table references. Plain data. |
| fork_parse | Text to tree, for the accepted subset, with positions for errors. |
| fork_check | Tree and grant to verdict: names, functions, scopes, limits. |
| fork_print | Tree to canonical SQL. |

Fork depends only on `common_*`. It is a pure function from text and a grant to a verdict.

The parser is hand-written: recursive descent over the accepted subset, easy to read and strict by construction. Its tests are gold-plated, because fork is a security boundary: tables of queries with the verdict and canonical SQL each must get, every refusal rule exercised, a corpus of known SQL injection and confusion tricks, and round trips (parse, print, parse) that must be stable.

## Later

Window functions and JSON functions are useful and large, and they wait for a real need.

## The invariants

- **Fork only ever accepts reads.** One SELECT, no PRAGMA, no ATTACH, nothing that writes.
- **Only granted tables, versions, columns, and functions.** Listed explicitly on the route; never `*`.
- **Every refusal is a problem.** With a hint an agent can act on.
- **SQLite only sees fork's output.** The text a box wrote never runs.
- **Scopes can't be escaped.** Every reference to a scoped table carries its filter.
- **Reserved parameters come from barn.** A box can't bind `!user_id` or `!tenant`.
- **Checked against grant and definition.** A granted column must also exist in the table version, with a compatible type.
- **Portable SQL only.** What fork accepts reads the same in SQLite, PostgreSQL, and MySQL.
- **Two locks.** Fork's check, then SQLite's authorizer on a read-only connection.
