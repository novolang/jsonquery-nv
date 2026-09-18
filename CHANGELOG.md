# Changelog

All notable changes to jsonquery-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
