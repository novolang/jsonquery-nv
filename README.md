# jsonquery-nv

jq's program language as a typed value, parsed to positioned faults and
evaluated over the standard library's JSON value.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A jq program is not a path.  A path picks values out of a document; a jq
program is a pipeline of filters, each of which takes one value and
produces a stream of zero or more — so `.users[]` produces many,
`select(.age > 30)` produces zero or one, and `.name` produces exactly
one, and all three are the same kind of thing.  That model is what makes
`[.users[] | select(.age > 30) | .name]` one expression rather than
three passes.

This package is that model, split the way the layers want it:

| module | holds |
| --- | --- |
| `jqlang` | the program as a typed value, and the parser that answers one |
| `jqbuiltin` | the builtin table, as data with arity |
| `jqeval` | the evaluator: a program and a value in, an outcome out |
| `jqerror` | a parse fault and a runtime error, which are two types |
| `jqfmt` | the renderings, and the CSV typing rule |

Paths inside a program are **jsonpath-nv's**: `JqFPath` holds a
`jpquery.JpQuery` and the evaluator runs it through `jpeval.select`.
That is where the filter selectors, slices and descendant segments come
from, and it is also where jq and RFC 9535 disagree — the table below is
those four rows.

## Adding it, and checking it

```console
$ novo pkg add jsonquery-nv
$ novo pkg build
```

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The one example that will work

```novo
use std.json
use jqeval
use jqfmt
use jqlang

fn main() [io]
    match jqlang.parse(".users[] | select(.age >= 30) | .name")
        Err(f) =>
            println("bad program at ${f.at}")
        Ok(p)  =>
            match json.parse("{\"users\":[{\"name\":\"ada\",\"age\":36},{\"name\":\"bob\",\"age\":24}]}")
                None    => println("not json")
                Some(v) =>
                    let out = jqeval.run(p, v)
                    for value in out.values
                        println(jqfmt.raw(value))
                    // ada
```

## The load-bearing interface

`JqOutcome`, and the argument for it is one sentence: **a jq filter that
fails has usually already produced output, and a `Result` throws that
away.**

```novo norun:pseudo
pub struct JqOutcome
    values: [JsonValueH]
    fault: ?jqerror.JqError
```

```
.users[] | (.name, error("stop"))
```

emits the first name and then raises.  A signature answering
`Result<[JsonValueH], JqError>` has exactly two shapes to put that in —
the values without the error, or the error without the values — and both
of them lie.  With both fields, "it produced three rows and then failed
on the fourth" is a value a caller can hold, print the three rows from,
and exit non-zero on.

It is also what makes `try`/`catch` implementable rather than
approximated.  `try f catch h` runs `f`, **keeps** the outputs `f`
managed, and feeds `h` the error value if one arrived; an evaluator
whose inner call answered `Err` has already lost the outputs it needs to
keep.  The same shape carries `//`, whose left side swallows an error,
and `f?`, which is `try f` with no handler.

`orbit/nq`, which this comes out of, answers `EOk(vals) | EErr(msg)` and
can afford to: it has no `try`, no `error` and no `//`, so nothing in it
can produce output and then fail.  Adding any one of the three forces
this shape, and adding all three is what this package is.

## A runtime error is a VALUE, and that is why there are two error types

`jqerror.JqFault` is a program that does not parse.  It has a byte span
into the program text, nothing can catch it, and there is no program to
run.  `jqlang.parse` answers it, once.

`jqerror.JqError` is a filter that failed while running, and in jq a
runtime error **is a value**:

```
try error({code: 4, path: .p}) catch .code
```

raises an object and reads a member out of it.  So `JqError` carries a
`JsonValueH` and `error_message` is a *rendering* of that value rather
than the thing itself.  A port whose payload is a `Str` can express
`error("no")` and cannot express the line above, which is the shape
every non-trivial jq program that handles failures is written in.

The position survives into the runtime too: `JqError.at` is the byte
offset of the filter that raised, so a caller can point a caret at `.a`
in `.users[] | .a.b` when the run fails on a record forty thousand lines
in.  jq itself cannot do this; the offset is already in the parsed
program, so keeping it costs one field.

## The builtin table is data, not a `match` in the parser

Four callers need four different questions answered about a builtin, and
only a value can answer all four: an **error message** needs the arities
a name does have, so `range(1;2;3;4)` is told "range has 1, 2 and 3
arguments"; **completion** needs the whole list with a summary each; the
**evaluator** needs to know which arguments are filters rather than
values, because `map(.x)` runs its argument once per element and
`limit(3; f)` must not run its second argument until it wants the next
output; and a **refusal** needs a reason.

`orbit/nq` is the case in point.  Its parser hard-codes six names, its
error message hard-codes the same six in prose a few hundred lines away,
and the two are one edit from disagreeing.

One variant per **name and arity**, because jq overloads on argument
count: `range/1`, `range/2` and `range/3` are three different filters,
and `jqbuiltin.resolve("range", 2)` cannot return the wrong one.

## Where jq and RFC 9535 answer the same path differently

Paths are jsonpath-nv's, so this table is the whole cost of that, and it
is applied in one function — `jqeval.select_path` — rather than spread
through an evaluator.  `jqeval.select_path_rfc` is the same walk with
the RFC's own answers, published beside it because the choice belongs to
the caller.

| the path | jq, and here | RFC 9535 |
| --- | --- | --- |
| `.foo` on an object without `foo` | `null` | selects nothing |
| `.[3]` on a shorter array | `null` | selects nothing |
| `.foo` on `null` | `null` | selects nothing |
| `.foo` on a number, `.[0]` on an object, `.[]` on a string | an **error** | selects nothing |

The last row has one exception and it is jq's: `.[]` against `null` is
an error even though `.foo` against `null` is not.  It looks
inconsistent because it is.  `orbit/nq` keeps it, its README says so,
and a user who knows jq must not have to relearn it.

Two more differences are not about paths and are worth stating here
because they are the two a reader trips over next:

- **`keys` sorts.**  jq's `keys` answers an object's member names in
  sorted order and `keys_unsorted` answers them in document order.
  RFC 9535 has no `keys` at all, and its wildcard visits members in
  document order.  Both are in the builtin table, separately, because
  `jqfmt.columns` needs the document order and a comparison needs the
  sort.
- **Comparison never fails.**  jq's `<` is a total order over every pair
  of values — `null < false < true < numbers < strings < arrays <
  objects` — so `1 < "a"` is true.  RFC 9535 § 2.3.5.2.2 orders only
  numbers against numbers and strings against strings and calls
  everything else false.  `jqeval.order` is jq's; the two agree
  everywhere the RFC has an opinion.

## The pattern language is I-Regexp, and six builtins are absent because of it

`test/1` matches with **I-Regexp** (RFC 9485) through jsonpath-nv's
`jpregex`, which has no anchors, no backreferences, no lookaround, no
lazy quantifiers and **no capture semantics at all**.  That is the point
of RFC 9485: what is left matches the same strings everywhere.

So the six jq builtins that need capture groups or match offsets are
absent, by name, with a reason: `match`, `capture`, `scan`, `splits`,
`sub`, `gsub` — and `test/2`, whose second argument is jq's flags
string, which is exactly what I-Regexp removed.  `jqbuiltin.absent_named`
and `jqbuiltin.absence_reason` publish that as data, because "unknown
function gsub" is the wrong answer for a name jq has: a user who wrote
`gsub` did not misspell anything, and telling them so sends them looking
for a typo.

**This is a missing row on the grid, and naming it is the honest form of
the refusal.**  What those six want is a `core` regular-expression
engine that answers match POSITIONS and capture groups — `jpregex`
answers a `Bool`, and `std.regex` is PCRE-shaped, so a jq program using
it here would work against this implementation and answer differently
against a conforming one across a wire.  Until such a package exists,
`jqbuiltin.absent_for_layer` says which absences a `host` package could
close (`env`, `input`, the clock) and which no layer can (`gsub` and its
five siblings).

## What else is absent, and why

- **The assignment operators** — `=`, `|=`, `+=`, and `del` by
  assignment.  Not hard, but a different half of jq: they need a path to
  *write* to rather than a value to read, and the piece that would carry
  it is `jpeval.path_steps`, which turns a normalized path back into a
  walk.  `path(f)` and `getpath(p)` are here, so the reading half is
  complete and the writing half is a named later step rather than a
  silence.
- **The host builtins** — `env`, `$ENV`, `input`, `inputs`, `now`,
  `strftime`, `halt_error`, `debug`.  A `core` package has no
  environment, no clock and no standard input, and a program that
  silently answered `null` for `env.HOME` would be worse than one that
  refuses.  These are the absences a `host` package built over this one
  could close.
- **`@base64` and the other `@` formats**, and SQL-style operators.  No
  consumer asks yet.

## What `orbit/nq` keeps, and what it takes

nq is the dogfood package this is cut out of.  The split is a file
boundary that already existed: `src/query.nv`, `src/eval.nv` and
`src/table.nv` are pure and `src/main.nv` is the only one that touches
the machine.

**nq keeps** `src/main.nv`, and after the swap that is the whole
package: argv, several files or standard input, the `--nd` and `-s`
input modes, stdout, and the exit code that answers "did that match
anything" so a shell script can branch on it.

**nq takes**, replacing code it has today:

| nq symbol | this package |
| --- | --- |
| `query.parse` / `query.Parsed` / `query.Stage` / `query.Step` / `query.Op` / `query.Lit` | `jqlang.parse`, `jqlang.JqProgram`, `jqlang.JqFilter`, `jqlang.JqBinOp`, `jqlang.JqLiteral` |
| `query.show` / `show_stage` / `show_path` / `show_op` / `show_lit` | `jqlang.render`, `jqlang.render_filter` |
| `eval.run` / `eval.apply` / `eval.apply_path` / `eval.EvalOut` | `jqeval.run`, `jqeval.apply`, `jqeval.select_path`, `jqeval.JqOutcome` |
| `eval.kind` / `eval.type_name` / `eval.K_NULL` … `eval.K_OBJ` | `jqeval.kind`, `jqeval.type_name`, `jqeval.JqKind` |
| `eval.cmp` / `eval.eq` / `eval.compare` | `jqeval.order`, `jqeval.equal` |
| `eval.jnull` | `jqeval.null_value` |
| `table.cell_value` / `table.csv_to_json` | `jqfmt.cell_value`, `jqfmt.round_trips`, `jqfmt.csv_to_json` |
| `table.columns` / `table.cell_text` / `table.grid` / `table.tabular` / `table.csv_out` | the same names in `jqfmt` |
| `table.shape` / `table.Shape` / `table.table_problem` | `jqfmt.shape`, `jqfmt.JqShape`, `jqfmt.table_problem` |
| `main.render` / `main.truthy` / `main.exit_code` | `jqfmt.compact` / `jqfmt.pretty` / `jqfmt.raw`, `jqeval.truthy`, `jqeval.exit_code` |

What nq **gains** by taking them: array and object construction (`[…]`
and `{…}` are parse errors in nq today), arithmetic, `reduce`,
`foreach`, `try`/`catch`, `//`, `as` bindings, string interpolation,
`def`, and every path form RFC 9535 has that nq's three-step scanner did
not — slices, descendant segments and filter selectors, so
`.users[?@.age > 30]` becomes expressible.

What nq **loses**, and it is a decision rather than a free upgrade:
`~=` accepts a PCRE pattern today through `std.regex`, and `test` here
accepts I-Regexp, which has no `(?i)`.  jsonpath-nv's README already
flagged that narrowing as nq's owner's call; it is the same call, and
the answer this package assumes is that portability wins.

Three of nq's `main.nv` behaviours are **not** taken and stay its own,
because they are about a stream of documents rather than about a
program: the `--nd` line-per-document reader, `-s` slurping, and the
rule that a malformed record on line 2 stops the run before line 1 is
printed.  `jqeval.run_stream` is the piece that supports the first of
them without making the decision.

## The layer, and why

`core`.  A parse of a string the caller typed, and a walk over a value
the caller already holds.  No function declares an effect, which is
checkable rather than a claim: the shard audit's `effect-budget` row
measures every `pub fn`'s declared row against `core`'s empty budget.

**No device claim, and the reason is not this package's arithmetic.**
`std.json` is refused at `@tier(embedded)`, so there is no
`tests/embedded_probe.nv` here and there cannot be one until the value
this queries builds for a microcontroller.  jsonpath-nv reached the same
wall for the same reason and filed it; a jq engine over a JSON value
that does not exist on the device would be a claim with nothing behind
it.

## What the standard library's JSON value cannot carry

Inherited from jsonpath-nv, which filed all of it, and repeated here
because it is load-bearing for two calls in this package:

- **`{}` and `null` are the same value to every typed accessor.**  Both
  answer zero keys and `None` from `to_str`, `to_int`, `to_float`,
  `to_bool` and `to_list`; only `json.stringify` separates them.  So
  `jqeval.kind` assembles the six types from the accessors and falls
  back to the rendering for exactly this pair, and `length` over `{}`
  and over `null` are two different answers that cost a render to tell
  apart.  `orbit/nq` carries a regression test called
  `test_empty_object_is_not_null`, and it moves here with the code.
- **There is no deep equality.**  `jqeval.equal` and `jqeval.order`
  exist partly because there is no `json.equals` to defer to.
- **Reaching one element of an array materialises all of it**, because
  `json.to_list` is the only way in.

All of it is filed against the toolchain as
`std-json-cannot-tell-an-empty-object-from-null-and-has-no-deep-equality`,
with `json.is_null`, `json.type_of` and `json.equals` as the smallest
things that would close it.

## The reference implementation

jq 1.7 (MIT) — its manual is the specification for the grammar, the
builtin table and the null-versus-error rule, and its own test suite
(`tests/jq.test`, a program, an input and the exact outputs) is the
oracle when the bodies land.  `orbit/nq` is the second reference: it is
this language's subset already written in novo-lang, with a hundred
assertions about the rules the two share, and those assertions are the
first thing `tests/jqeval_tests.nv` restates.

## Status

Every function is `todo()`.  The four suites under `tests/` are red on
`not implemented`, which is the expected result until the bodies land:

```console
$ novo test tests/jqlang_tests.nv
$ novo test tests/jqbuiltin_tests.nv
$ novo test tests/jqeval_tests.nv
$ novo test tests/jqfmt_tests.nv
```
