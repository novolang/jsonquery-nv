# Changelog

All notable changes to jsonquery-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Fixed

- `jqlang.parse` refuses programs nested more than 128 deep with `JqUnexpectedByte`.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-11

The **implementation**.  Every signature the interface release published
now has a body, the eight suites run green under `novo test`, and nothing
in the interface changed: the effect rows are the ones 0.0.1 published,
and every function is still effect-free.

### Added

- `jqlang.parse` and `parse_with`: the grammar, with `def`, `reduce`,
  `foreach`, destructuring `as`, `try`/`catch`, `//` and string
  interpolation in the tree, every node keeping the byte offset it
  started at.  A bare `..` parses as the `recurse` builtin, because
  jq's recursive descent yields the input itself and RFC 9535's
  descendant segment does not.  A quoted name inside `["…"]` is copied
  into the RFC 9535 text the path delegate reads.  Builtins resolve at
  parse time, so an unknown name is a fault with a caret rather than a
  surprise on the four hundred thousandth record.
- `jqlang.render`, `render_filter`, `position`, `free_variables`,
  `defined_names`, `called_builtins`, `is_path_expression`, `depth` and
  `path_fault`, each answered from the tree.
- `jqbuiltin`: the table filled in — 69 name-and-arity entries, with
  `takes_filter_argument`, `may_stream`, `resolve` and the refusals
  published as data with a reason each.
- `jqeval.run`, `run_with`, `run_stream` and `apply`, over jq's own
  rules: the total order, truthiness of everything but `null` and
  `false`, and a budget that stops a program that does not stop itself.
  `select_path` applies the four places jq disagrees with RFC 9535 in
  one function; `select_path_rfc` is the delegate unchanged.
- `jqfmt`: the CSV typing rule, the column arithmetic, the grid, the
  aligned table, CSV out, and the three renderings.  `pretty` takes the
  indent rather than deciding it.
- `tests/jqquery_edge_tests.nv`, a sixth suite for the readings the
  other five leave implicit — a blank CSV cell versus the text `null`,
  a short record's missing cell, the column-refusal sentences, the
  padding rule, `pretty` at indent 0, and the parse-time refusal of a
  function jq has and this package does not.
- `tests/jqcover_tests.nv`, a seventh suite that walks the SURFACE
  rather than the vectors: every entry of the builtin table, every
  production of the grammar, every node kind of the evaluator, each
  with an input that carries it to the arm it covers, so a row added
  and not wired fails even before the publish gate's coverage line
  reads it.
- `tests/jqarms_tests.nv`, an eighth suite for the half of every rule a
  program cannot spell: a hand-built tree for the arms the parser never
  produces, the fault a whole-array builtin carries out, the escape the
  scanner reads backwards, the invalid UTF-8 no jq domain reaches, and
  one row per budget reading with the ceiling named in the row.

### Known

- **`del`, `children` and the assignment operators are not
  implemented.**  `del(.a)` and `children` answer an unknown function;
  `|=`, `+=`, `del` by assignment need a writing half over
  `jpeval.path_steps` that this package does not carry.
- **`test` is a SEARCH over I-Regexp, and I-Regexp has no anchors.**
  `test("b")` matches `"abc"` and `test("^a+$")` matches nothing, which
  is what RFC 9485 defines and not what a PCRE habit expects.  The
  regex family that needs capture groups, and the family that needs a
  machine, are refused by name with the reason in
  `jqbuiltin.absence_reason`.
- **The readings jq's manual does not write down, measured and
  asserted in the edge suite**: a number literal with an exponent is a
  number (`1e3` is `1000`); `..` includes the input itself; `join`
  renders `null` as the empty string; `add` over nothing is `null` and
  not an error; `keys` sorts and `keys_unsorted` is document order;
  `.[]?` over a scalar is nothing while `.a` over a number is an error.
- **The readings `reduce` and `foreach` follow, measured against jq
  1.7**: an update that produces nothing sets the state to `null` and
  the fold goes on with the next item, so `reduce [1,2][] as $x (0;
  empty)` is `null` and not nothing; an init with several outputs folds
  once per output, so `reduce (1,2) as $x (0,10; . + $x)` is `3, 13`; a
  fault from the SOURCE loses the state a `reduce` had reached, while a
  `foreach` keeps the outputs it already made; and the extract is not
  run for an item whose update was empty.  Asserted in the arms suite.
- **What a ceiling does is measured too, in the arms suite**: an array
  construction spends the output budget before it makes its one output,
  so `[range(0;10)]` under `max_outputs = 2` answers nothing and
  `over-budget` while `range(0;10)` answers `0, 1, 2` and
  `over-budget`; the values a stream already emitted are kept and the
  ceiling is reported beside them; and a `reduce` whose source the
  ceiling cut off answers no state at all.
- **`limit` and `first` collect their filter argument**, so
  `limit(1; range(0; 1000000))` answers `0` *and* reports
  `over-budget`, where jq answers `0` alone, and the cost is paid in
  full.  Filed as
  `bugs/orbit/limit-and-first-collect-their-filter-argument.md`;
  answering jq's way needs a consumer-stop state `Run` does not carry,
  which is why it is a caveat here rather than a body.
- **jq's library-defined helpers are absent as well** — `repeat`,
  `walk`, `isempty`, the type filters (`numbers`, `strings`, `values`,
  `nulls`, `booleans`, `arrays`, `objects`, `iterables`, `scalars`) and
  the rest of the family jq writes in jq answer an unknown function.  `jqbuiltin.absent_named` publishes the
  17 names that carry a written reason — `match`, `capture`, `scan`,
  `splits`, `sub`, `gsub`, `test`, `env`, `$ENV`, `input`, `inputs`,
  `input_line_number`, `now`, `localtime`, `strftime`, `halt_error`
  and `debug`; a name outside that list is simply not in the table.
- **`std.json`'s two limits are unchanged** from 0.0.1: `{}` and
  `null` answer the same thing to every typed accessor, and there is no
  deep equality, which is why `jqeval.kind`, `equal` and `order` exist.

### Changed

- **The parser's three early-returning functions answer through a single
  exit.**  On this toolchain an early return skips the release of a
  call-bound local whose last use the tail moves into the answer, so
  every malformed `reduce`, `foreach` and `as` pattern leaked one
  `JqPattern` — every verdict green and `the run leaked`, exit 23.
  `parse_reduce_at`, `parse_foreach_at` and `parse_postfix` now bind
  their answer to a `var` local and answer that local, which is
  behaviour-identical and clean under `novo test --isolate`.  Filed as
  `bugs/memory-perceus/an-early-return-skips-the-release-of-a-call-bound-local-the-tail-moves.md`;
  collapse each back to its `return` when that filing closes.
- **A fault raised inside a whole-array builtin now reaches the
  caller.**  `any`, `all`, `until`, `while`, `map_values` and the
  `_by` family used to answer a value (or nothing) when a later element
  raised, which is the one thing a jq caller cannot write a `try`
  around; each now carries the first fault out.
- **`reduce` and `foreach` were fitted to jq's readings for an empty
  update, for several init outputs and for where a fault arrives**, all
  measured against jq 1.7.  A first draft answered nothing for an update
  that produced nothing and dropped a source's fault once a state had
  been reached, and its doc block claimed jq's answer for both; the
  bodies now fold once per init output, set the state to `null` when an
  update produces nothing and carry the first fault out in the order jq
  reports it.  The arms suite pins every reading (`reduce (1,2) as $x
  (0,10; . + $x)` is `3, 13`, `reduce (1, error("x")) as $x (0,5; .)`
  is the error and no state).
- **Two private helpers no caller reached were removed rather than
  excused.**  `whole_number` in `jqeval` (its callers ask `whole_int` or
  `is_whole`) and `scan_plain_text` in `jqlang` (every string this parse
  reads goes through `scan_text_until` or `parse_string_at`) were dead
  code the first draft left behind.  A `cov: skip` marker excuses the
  line below it and the block that line opens, so excusing a whole
  function means putting the marker over its `fn` line, which would
  split the function from its doc block; deleting the dead helpers
  keeps every remaining marker over one arm or one block.  Ten markers
  remain, and the publish gate reads each one.
- **`contains` over arrays follows jq's rule rather than a
  subsequence**: the array contains another when each of that one's
  elements is contained, recursively, in *some* element of the first,
  in any order — `[0,1,2,3] | contains([3,1])` is `true`.  `inside` is
  the same function with the arguments the other way round.
- **`range` refuses a bound that is not a number** instead of reading
  it as zero, which is jq's own rule (`range(0; "2")` is an error
  there); a fractional bound is left as it is, so `range(0.5; 2)` is
  `0.5, 1.5`.
- **A string with an invalid escape no longer stops the scanner
  advancing**, so `"\\q"` is one `bad-escape` fault instead of a run
  that does not end.  Found while probing the fault table.
- **`jqfmt.csv_out` renders CSV itself instead of calling
  `csv.stringify`.**  On this toolchain `csv.stringify` leaks one heap
  `Str` per input cell, so every `novo test` run of the CSV suite
  printed every verdict green and exited 23.  The writer is
  `needs_quotes` / `csv_field` / `render_csv` in `src/jqfmt.nv`, quotes
  exactly what RFC 4180 § 2.6–2.7 requires, and `csv.parse` reads its
  output back.  Filed as
  `bugs/stdlib/csv-stringify-leaks-one-str-per-input-cell.md`; revert
  the helper for the stdlib call when that filing closes.
- **`x == false` rewritten to `not <expr>`** in the five suites, which is
  the `redundant-bool` rule the linter suggests.  All 18 sites were
  pre-existing (`git show HEAD:tests/<file>` carries the same count)
  and the rewrite cannot change a predicate; the edge suite added with
  this release is written `not <expr>` from the start.
- **Two locals named `env` in `tests/jqeval_tests.nv` renamed to
  `scope`**, which the compiler reported as `W0002 binding 'env'
  shadows stdlib module 'env'`.  The binding a `JqEnv` is held in is
  still the same value; no assertion changed.
- **`jqerror.JqFault.subject`'s doc corrected**: a `JqBadPath` fault
  carries the RFC 9535 path text it was refused for, not the empty
  string, which is what `jqlang.path_fault` hands back to the delegate.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- jsonpath-nv: `^0.0.2` to `^0.1.0`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`JqFault` now declares the `impl Error` its own `Result` positions
require.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `Result<_, jqerror.JqFault>` was accepted
across modules with no impl anywhere.  The impl is the signature this
package always meant; nothing else about the interface changed.
`JqError` needs none: a runtime error travels in `jqeval.JqOutcome`,
never in a `Result`.

`jsonpath-nv` is now taken at `^0.0.2`, the version whose `JpFault`
declares the same impl — under the caret rule `^0.0.x` means exactly
that version, so `^0.0.1` would have resolved a `JpFault` that is not
an `Error` and `jqlang.select_path`'s delegate would not build.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `jqlang` — a jq program as a typed value, with every filter node
  carrying the byte offset it started at, and the parse that answers
  one. `def`, `reduce`, `foreach`, destructuring `as` patterns and
  string interpolation are in the tree; paths are `jpquery.JpQuery`.
- `jqbuiltin` — 68 builtins as data, one variant per name AND arity,
  with the filter-argument flag an evaluator cannot work without and
  the refusals published as data with a reason each.
- `jqeval` — `JqOutcome`, `run`, `run_stream`, `apply`, jq's total
  order, jq's truthiness, and the two path calls that name which rules
  they follow.
- `jqerror` — `JqFault` for a program that does not parse and `JqError`
  for a filter that failed, which are two types because only the second
  is catchable and only the second carries a value.
- `jqfmt` — the CSV typing rule, the column arithmetic, and the three
  renderings, so `orbit/nq` keeps only its CLI.

### Known

- **`JqOutcome` is the load-bearing interface.** A jq filter that fails
  has usually already produced output, and a `Result` has nowhere to put
  both halves. It is also what makes `try`/`catch`, `//` and `f?`
  implementable rather than approximated.
- **A runtime error is a VALUE.** `error({code: 4})` raises an object
  and `catch .code` reads a member out of it, so `JqError.value` is a
  `JsonValueH` and the message is a rendering of it.
- **The builtin table is data.** One variant per name and arity, because
  `range/1` and `range/2` are different filters and an error message has
  to be able to say which one the program asked for.
- **Paths are jsonpath-nv's**, and the four places jq's answers differ
  from RFC 9535's are a table in the README applied in one function.
- **Six regex builtins are absent** — `match`, `capture`, `scan`,
  `splits`, `sub`, `gsub`, and `test/2` — because I-Regexp has no
  capture semantics. A `core` engine answering match positions is the
  missing row that would close them; no layer closes them otherwise.
- **The assignment operators are absent**, deliberately: they need
  `jpeval.path_steps` and a writing half this interface does not have.
- **No device claim.** `std.json` is refused at `@tier(embedded)`, so
  there is no probe and there cannot be one until the value this queries
  builds for a microcontroller.
- **One dependency**, jsonpath-nv, by registry range.

### Design notes

- **`orbit/nq` is the package this interface was cut out of.** nq keeps
  `src/main.nv` — argv, several files or standard input, the `--nd` and
  `-s` input modes, stdout, and the exit code a shell script branches
  on. Everything in `src/query.nv`, `src/eval.nv` and `src/table.nv` is
  replaced: `query.parse` by `jqlang.parse`, `query.show` by
  `jqlang.render`, `eval.run` and `eval.EvalOut` by `jqeval.run` and
  `jqeval.JqOutcome`, `eval.cmp` by `jqeval.order`, and every symbol in
  `table.nv` by the same name in `jqfmt`. What nq gains is array and
  object construction, arithmetic, `reduce`, `foreach`, `try`/`catch`,
  `//`, `as` bindings, string interpolation, `def`, and the path forms
  its three-step scanner never had. What nq loses is PCRE in `~=`,
  because `test` here is I-Regexp; portability was chosen over
  `(?i)`. Three of nq's behaviours stay its own, because they are about
  a stream of documents rather than about a program: the `--nd`
  line-per-document reader, `-s` slurping, and the rule that a
  malformed record on line 2 stops the run before line 1 is printed.
- **The standard library's JSON value has three limits this interface
  is shaped around**, all filed against the toolchain as
  `std-json-cannot-tell-an-empty-object-from-null-and-has-no-deep-equality`.
  `{}` and `null` answer the same thing to every typed accessor, so
  `jqeval.kind` falls back to the rendering for that pair alone. There
  is no deep equality, which is part of why `jqeval.equal` and
  `jqeval.order` exist. Reaching one element of an array materialises
  all of it, because `json.to_list` is the only way in. `json.is_null`,
  `json.type_of` and `json.equals` are the smallest additions that
  would close the filing.
- **The six regex builtins are a missing capability rather than a
  choice.** Closing them needs a regular-expression engine with no
  machine effects that answers match positions and capture groups.
  `jpregex` answers a `Bool`, and `std.regex` is PCRE-shaped, so a jq
  program using it here would answer differently against a conforming
  implementation across a wire.
