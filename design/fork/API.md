# Fork API
The C++ library every SQL read in homestead passes through
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Fork is a library, not a service: it has no wire API, no threads, and no I/O. Its API is a small set of C++ headers under `<fork/...>`, one pure function at the center (**text, a grant, and table definitions in; a verdict out**), and a command-line tool so agents can ask the same question without writing C++. Following Q12.5's lean, fork depends on nothing but the C++20 standard library (and Boost.JSON, only in the optional JSON component), builds as its own CMake project, and is consumed by homestead the way homestead consumes SQLite: as a versioned, installed dependency.

Consumers include exactly one header, `<fork/fork.h>`. Everything else (the tree, the lexer, the parser, the rewriter) is an implementation detail with its own package and tests, and may change in any release.

## The surface

| header | gives | stability |
| --- | --- | --- |
| `<fork/fork.h>` | `fork::check`, `fork::Checker`, and the types below | Stable; semver |
| `<fork/grant.h>` | `Grant`, `TableGrant`, `Limits`, `TableDefinition`, `Column`, `Type`, `Definitions` | Stable |
| `<fork/verdict.h>` | `Verdict`, `Accepted`, `Refusal`, `RefusalCode` | Stable; codes only added |
| `<fork/dialect.h>` | `Dialect` (`sqlite` today) | Stable; dialects only added |
| `<fork/json.h>` | Grant, definition, and verdict codecs (component `fork::json`) | Stable |
| `fork` (the binary) | `fork check`, `fork print`, `fork explain` | Stable flags; output is JSON |

## The function

```cpp
#include <fork/fork.h>

fork::Grant grant = ...;              // from the route
fork::Definitions defs = ...;         // the table versions' columns, from the tailers
fork::Verdict v = fork::check(
    "SELECT id, status FROM jobs WHERE status = :status ORDER BY finished_at DESC",
    grant, defs, {.dialect = fork::Dialect::sqlite});

if (auto* ok = std::get_if<fork::Accepted>(&v)) {
  ok->sql;          // canonical SQL to prepare: the only text SQLite ever sees
  ok->params;       // positional order of bound names: {"status"}
  ok->reserved;     // reserved names used: {"!user_id"} (from scopes or the query)
  ok->tables;       // physical tables read: {"jobs@2"}
  ok->columns;      // per logical table, the columns read
} else {
  auto& no = std::get<fork::Refusal>(v);
  no.code;          // fork::RefusalCode::column_not_granted
  no.at;            // byte offset in the text
  no.table, no.column;
  no.hint;          // a sentence for an agent
}
```

```cpp
namespace fork {
struct Options {
  Dialect dialect = Dialect::sqlite;
  std::size_t max_text_bytes = 16 * 1024;
  std::size_t max_literal_bytes = 1024;
  std::size_t max_depth = 64;            // expression and subquery nesting
};
Verdict check(std::string_view sql, const Grant&, const Definitions&, const Options& = {});

// For callers that check many queries against one grant: validates the grant
// and definitions once (a scope that doesn't parse, a granted column that doesn't exist).
class Checker {
 public:
  static std::variant<Checker, Refusal> make(Grant, Definitions, Options = {});
  Verdict check(std::string_view sql) const;
};
}
```

- **`check` never throws** for any input text. Bad text is a `Refusal`; only allocation failure escapes.
- **`Checker::make` checks the grant itself**: every scope parses and uses only reserved parameters and literals, every granted column exists in the definition with a known type, every function is one fork knows. Rancher calls it when a route is put; a bad grant is refused there, not on the first request.

## Grants and definitions

```cpp
namespace fork {
enum class Type { id, int64, real, text, boolean, blob };

struct Column { std::string name; Type type; bool nullable = true; };
struct TableDefinition { std::string name; int version; std::vector<Column> columns; };
using Definitions = std::vector<TableDefinition>;     // every version the grant may name

struct TableGrant {
  int version;
  std::vector<std::string> columns;                   // never "*"
  std::optional<std::string> scope;                   // a WHERE over reserved params and literals
};
struct Limits { std::uint32_t max_rows = 1000, max_ms = 50, max_joins = 4; };
struct Grant {
  std::map<std::string, TableGrant> tables;           // by logical name
  std::vector<std::string> functions;                 // from fork::known_functions()
  Limits limits;
};
std::span<const std::string_view> known_functions();
}
```

The JSON form, which rancher stores on routes and barn sends to pond, is the one in [fork's idea](IDEA.md#a-permission-set). `fork::json` reads and writes it, refusing unknown keys.

## Verdicts

```cpp
namespace fork {
struct Accepted {
  std::string sql;
  std::vector<std::string> params;                    // ?1..?n map to these names
  std::vector<std::string> reserved;
  std::vector<std::string> tables;                    // "jobs@2"
  std::map<std::string, std::vector<std::string>> columns;
  std::uint32_t max_rows, max_ms;                     // the limits the caller must enforce
};
struct Refusal {
  RefusalCode code; std::size_t at;
  std::string table, column, function, hint;
};
using Verdict = std::variant<Accepted, Refusal>;
}
```

The canonical SQL binds every parameter positionally (`?1`, `?2`, ...), quotes every identifier, names each physical table (`"jobs@2"`), wraps scoped tables in their subquery, and carries a `LIMIT` no larger than `max_rows`. Printing is deterministic: two texts that parse to the same tree print the same bytes, so the output is a cache key.

### Refusal codes

Every code becomes a problem (`fork.$code`) when a service reports it.

| code | when |
| --- | --- |
| `syntax` | The text doesn't parse in the accepted subset. |
| `not_select` | Any statement but one SELECT: INSERT, PRAGMA, ATTACH, a second statement. |
| `unsupported` | Valid SQL outside the subset: recursive CTEs, window functions, JSON functions, engine-specific syntax. |
| `star` | `*` or `t.*`. |
| `table_not_granted` | A table the grant doesn't name. |
| `column_not_granted` | A column the grant doesn't list for that table. |
| `column_unknown` | A granted column the table version doesn't define. |
| `version_unknown` | The grant names a version the definitions don't have. |
| `function_not_granted` | A function outside the grant's list. |
| `function_unknown` | A function fork doesn't know. |
| `type_mismatch` | A comparison or function over incompatible types. |
| `reserved_bound` | A query binds a reserved name the caller must supply (only checked in `bind`, below). |
| `scope_invalid` | A scope that doesn't parse or uses a non-reserved parameter (from `Checker::make`). |
| `too_large` | Text, literal, depth, or joins over the limits. |
| `ambiguous` | A column name that two tables in scope both have, unqualified. |

## Dialects

`Dialect::sqlite` is the only target today. The accepted subset is portable on purpose (see [the idea](IDEA.md#what-fork-accepts)), so adding `postgres` or `mysql` is a new printer and its gold tests, never a change to what is accepted. A query accepted for one dialect is accepted for all.

## The command line

For agents, and for anyone debugging a refusal without a running pond:

```
fork check --grant route-grant.json --defs tables.json "SELECT id, status FROM jobs WHERE status = :s"
fork print --dialect sqlite "select ID from jobs"          # parse and print, no grant
fork explain --grant g.json --defs d.json "..."            # the tree, the scopes applied, the verdict
```

Output is the verdict as JSON, the same shape services report:

```json
{ "ok": false, "error": "column_not_granted", "table": "customers@1", "column": "email", "at": 31,
  "hint": "customers@1 is granted columns id, name. Add email to the route's grant, or stop selecting it." }
```

Exit status: 0 accepted, 1 refused, 2 bad arguments.

## Threads, memory, and cost

- **Pure and reentrant.** `check` touches no globals; a `Checker` is immutable after `make` and may be shared across threads.
- **Bounded.** Every input limit in `Options` caps time and memory; checking is linear in the text for the accepted subset. A query at the limits checks in well under a millisecond.
- **Allocation.** The tree is arena-allocated per call and freed at return; nothing outlives the verdict but its strings.

## Versioning

- **Semver on the stable headers.** A minor release may add refusal codes, functions to `known_functions()`, dialects, and fields with defaults. A major release is the only thing that may accept less or change canonical output.
- **Canonical output is versioned.** `fork::canonical_version()` says which printer made a string; rancher stores it with each named query, and re-checks named queries when it changes.
- **No ABI promise.** Fork is linked statically into each service and rebuilt with it; the promise is source compatibility.

## How homestead uses it

| consumer | calls | with |
| --- | --- | --- |
| pond (`pond_query`) | `check` on every ad-hoc query; prepares `Accepted::sql`; feeds `tables` and `columns` to the authorizer and the waiter | The grant from barn and pond's own table definitions |
| rancher | `Checker::make` when a route, grant, or table version changes; `check` on every named query, storing the canonical SQL | Its route config and the tailer definitions it holds |
| store | `check` on every `_queries/*.sql` at write and at release | Definitions from `rancher.tables.get` |
| barn (optional) | `check` before sending ad-hoc SQL, to refuse early and key its cache by `Accepted::sql` | The route's grant and definitions from the config log. Never the gate. |

## The invariants

- **One pure function at the center.** Text, grant, and definitions in; verdict out; no I/O.
- **Only `<fork/fork.h>` is a contract.** Every other header may change.
- **Accepted means a single, read-only SELECT over granted columns**, printed canonically, with scopes applied and a LIMIT written in.
- **Every refusal has a code, a place, and a hint.**
- **The same inputs give the same bytes,** in every process that links fork.
- **Fork depends on the standard library only.** JSON is an optional component.
