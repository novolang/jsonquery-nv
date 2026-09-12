# Changelog

All notable changes to jsonquery-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
