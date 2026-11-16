# Fork packages
A SQL checker cut so the security boundary is three small packages and a lot of tests
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Fork is a library first. It is its own CMake project at `src/fork/`, depending on nothing but C++20 and GoogleTest (and Boost.JSON for one optional package), with an installed target `fork::fork` and one public header, `<fork/fork.h>` (see [the API](API.md)). Homestead consumes it like third-party code, which is what Q12.5 leans toward: fork can be built, tested, fuzzed, and released without the rest of the tree, and an agent working on it needs nothing else in its head.

Inside, the cut follows where a bug becomes a breach. **Parsing** can only be wrong by refusing too much or building the wrong tree, and the wrong tree is caught later, so it is R1. **Checking** (does this tree only read what was granted?), **rewriting** (bind versions, inject scopes, cap rows), and **printing** (the bytes SQLite will run) are where a bug lets a box read what it shouldn't, so those three are R0, and each is kept small: the checker doesn't print, the printer doesn't decide, and the rewriter doesn't check. The tree, the lexer, the grant types, the codecs, and the CLI are pushed out to R1 and R2.

Fork's namespace stays `fork::` rather than `homestead::fork::`, because it is a library meant to stand alone; the package directory still names its namespace (`fork_check` is `fork::check_` internally, and only the facade's names are public).

## The packages

| package | namespace | does | depends on | tier |
| --- | --- | --- | --- | --- |
| `fork_ast` | `fork::ast` | The tree: statements, select cores, expressions, table references, positions. Arena-allocated plain data. | std | R1 |
| `fork_lex` | `fork::lex` | Text to tokens: keywords, identifiers (quoted and bare), literals with size caps, `:name` and `?` parameters, `!` reserved names | std | R1 |
| `fork_parse` | `fork::parse` | Tokens to tree, recursive descent, the accepted subset only, with positions for every refusal | `fork_lex`, `fork_ast` | R1 |
| `fork_grant` | `fork::grant` | `Grant`, `TableDefinition`, `Limits`, the known function list with types; grant self-validation | `fork_ast`, `fork_parse` (for scopes) | R1 |
| `fork_check` | `fork::check_` | Tree x grant x definitions -> refusal or a resolved tree: names, aliases, scopes of names, versions, columns, functions, types, joins, no `*` | `fork_ast`, `fork_grant` | R0 |
| `fork_rewrite` | `fork::rewrite` | A resolved tree -> the tree to print: physical table names, scope subqueries at every reference, LIMIT injected or lowered, parameters numbered | `fork_ast`, `fork_grant` | R0 |
| `fork_print` | `fork::print` | Tree -> canonical SQL for a dialect: quoting, precedence, deterministic output | `fork_ast` | R0 |
| `fork_api` | `fork` | The facade: `check`, `Checker`, `Verdict`, `Refusal`, refusal codes and hints. The only package consumers include. | all of the above | R1 (its header is R0) |
| `fork_json` | `fork::json` | Grant, definitions, and verdict codecs; unknown keys refused | `fork_api`, Boost.JSON | R2 |
| `fork_corpus` | | Data only: gold queries with their grants, definitions, verdicts, and canonical SQL; the injection and confusion corpus | | R1 |
| `fork_fuzz` | | libFuzzer targets: lex, parse, check, and parse(print(parse(x))) stability | `fork_api` | R2 |
| `fork_main` | | The `fork` command line: `check`, `print`, `explain` | `fork_api`, `fork_json` | R3 |

## Dependencies

```
                 fork_main      fork_fuzz
                     |      \      |
                 fork_json   \     |
                     |        v    v
                     +-----> fork_api  <----- consumers: pond_query, rancher_check,
                            /   |    \                   store_query, barn_cache
                 fork_check fork_rewrite fork_print
                      \       |        /
                       fork_grant     /
                       /    \        /
              fork_parse     fork_ast
                  |            ^
               fork_lex -------+

         fork_corpus: data read by every package's tests
```

`fork_print` depends only on the tree, so it can be tested (and trusted) without any notion of grants: the printer's only job is to say exactly what the tree says.

## QUALITY.md seeds

### fork_check (R0)
- **Promise:** a tree is resolved only if every table, column, and function it names is granted and defined, every name binds to exactly one source, and nothing writes; otherwise it is refused with a code and a position.
- **Tests:** the gold corpus (every refusal code exercised at least five ways); the confusion corpus (aliases shadowing tables, correlated subqueries naming outer columns, CTEs named like granted tables, quoted identifiers with odd bytes, case folding); a model test that walks every column reference in an accepted tree and asserts it is in the grant.
- **Won't:** print, rewrite, or look at text.

### fork_rewrite (R0)
- **Promise:** every reference to a scoped table, at any depth (joins, subqueries, CTEs), is replaced by its scope subquery; every table carries its granted physical name; the outermost LIMIT is at most `max_rows`.
- **Tests:** for each accepted corpus query, an independent tree walk confirms no unscoped reference survives; LIMIT cases (none, larger, smaller, parameterized, with OFFSET).
- **Won't:** decide whether a query is allowed.

### fork_print (R0)
- **Promise:** the printed SQL parses (by fork and by SQLite) to the same tree; output is deterministic; every identifier is quoted and every value is a parameter or an escaped literal.
- **Tests:** parse(print(t)) == t over the corpus and the fuzzer; SQLite's own `EXPLAIN` of printed output reads only the expected tables (a test-only check against a real SQLite); quoting of every byte value.
- **Won't:** accept a tree fork_check didn't resolve (it takes a distinct resolved type).

### R1 seeds

| package | promise | tests | won't |
| --- | --- | --- | --- |
| `fork_ast` | Plain data with positions; equality and hashing | Construction and equality tables | Validate |
| `fork_lex` | Every input yields tokens or a positioned error, in linear time | Fuzzed; every literal and identifier form; size caps | Allocate per byte |
| `fork_parse` | Every input in the subset parses to its tree; everything else is `syntax` or `unsupported` with a position | Gold corpus; fuzzed; depth limits; known-hostile inputs from SQLite's test suite | Accept anything outside the subset to "be helpful" |
| `fork_grant` | A grant that validates has parseable scopes over reserved params, known functions, and defined columns | Table tests per rule | Know about routes or JSON |
| `fork_api` | `check` never throws and maps every internal failure to a code and hint | Every corpus case through the facade | Expose internal types |
| `fork_corpus` | Every case is reviewed and named; additions only | A test that every case is used | Hold anything generated |

## How it's wired

There is no system and no main loop. The composition is the facade: `fork::check` lexes, parses, checks, rewrites, and prints, in that order, on the caller's thread, with one arena per call. `fork::Checker` precomputes the validated grant and a definition index once.

Homestead links `fork::fork` into `pond_query` (the gate), `rancher_check` (deploy-time checks), `store_query` (named queries in a site), and optionally `barn_cache`. Each consumer's QUALITY.md lists the fork version it was tested against; upgrading fork is a pull request that bumps that version and reruns those consumers' tests, and it is reviewed at R0 because it changes what the gate accepts.

## Review queue

1. **R0**, read line by line: `fork_check`, `fork_rewrite`, `fork_print`, and the `<fork/fork.h>` header.
2. **R1**: `fork_parse`, `fork_lex`, `fork_grant`, `fork_api`, `fork_ast`, and every addition to `fork_corpus` (a new case is a new rule someone believed).
3. **R2/R3**: `fork_json`, `fork_fuzz`, `fork_main`.

Build order, each line parallel:

1. `fork_ast`, `fork_lex`, `fork_corpus` (the first cases)
2. `fork_parse`, `fork_print` (round trips start here)
3. `fork_grant`
4. `fork_check`, `fork_rewrite`
5. `fork_api`, `fork_json`, `fork_fuzz`, `fork_main`

Consumers can start against `fork_api`'s header and a stub that refuses everything, so pond and rancher are never blocked on fork's internals.

## The invariants

- **Fork builds alone**, with the standard library and GoogleTest, and installs as `fork::fork`.
- **Consumers include `<fork/fork.h>` and nothing else.**
- **Check, rewrite, and print are separate R0 packages**, and none does another's job.
- **The printer takes only resolved trees**, so unchecked text can't reach it by type.
- **The corpus only grows.** Every refusal and every bug fixed becomes a case.
- **Changing fork is reviewed at R0 in every consumer**, because it changes what the gate accepts.
