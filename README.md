# jsonquery-nv

[jq](https://jqlang.github.io/jq/manual/v1.7/) is a command-line JSON
processor. Its manual describes it as "like `sed` for JSON data", and a jq
program as a **filter**: "it takes an input, and produces an output". This
package brings jq's program language to novo-lang. A program is parsed into a
typed value, and that value is run over the JSON value the standard library's
`std.json` produces. The paths inside a program are
[jsonpath-nv](https://novo-lang.org/packages/jsonpath-nv)'s, so a jq path and a
JSONPath query are the same walk over the same document.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **filter** takes one JSON value and produces a **stream** of zero or more JSON
values. That is the whole model. `.name` produces exactly one value.
`select(.age > 30)` produces zero or one. `.users[]` produces one per element of
an array. All three are filters, and because they are the same kind of thing
they compose.

Filters compose two ways. The **pipe**, `f | g`, feeds every output of `f` to
`g` in turn. The **comma**, `f, g`, produces the outputs of `f` and then those
of `g`. Square brackets collect a stream back into one value, so
`[.users[] | select(.age > 30) | .name]` is a single expression that answers one
array.

A **path** is the part of a filter that names places in the input: `.a.b`,
`.[0]`, `.users[1:3]`, `..`. A filter every one of whose outputs came from
somewhere in the input is a **path expression**, and only a path expression can
be handed to `path(f)`. `.a` is one. `.a + 1` is not.

A **builtin** is a filter with a name, such as `length`, `map(f)` or
`range(n)`. jq overloads a builtin name on the number of arguments it is given,
so `range(n)`, `range(from; upto)` and `range(from; upto; by)` are three
different filters. The number of arguments is the **arity**, and a builtin in
this package is identified by the pair of name and arity.

A jq program can fail while running, and it can fail after it has already
produced output. `.users[] | (.name, error("stop"))` emits the first name and
then raises. A run therefore answers both halves: the values that were produced
and the error that stopped it.

A runtime error in jq is itself a JSON value. `error({code: 4})` raises an
object, and `try f catch .code` binds that object and reads a member out of it.

## Install

```
novo pkg add jsonquery-nv
```

## Example

```novo
use std.json
use jqlang
use jqeval
use jqfmt

fn main() [io]
    // Parse the program once. Everything that can be wrong with a program
    // is wrong here: an unknown builtin, a wrong arity, an unbound $name.
    match jqlang.parse(".users[] | select(.age >= 30) | .name")
        Err(f) => println("the program is bad at byte ${f.at}")
        Ok(p)  =>
            // Read the document with the standard library's JSON parser.
            match json.parse("{\"users\":[{\"name\":\"ada\",\"age\":36},{\"name\":\"bob\",\"age\":24}]}")
                None    => println("the input is not JSON")
                Some(v) =>
                    // Run the program. The outcome carries the values it
                    // produced and the error it stopped on, if any.
                    let out = jqeval.run(p, v)
                    for value in out.values
                        println(jqfmt.raw(value))
                    // ada

                    // jq's own exit code: 0 for output, 1 for none, 5 for a failure.
                    println("${jqeval.exit_code(out, false)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `jqlang` | A jq program as a typed value, and the parser that answers one. Every node carries the byte offset it starts at. |
| `jqbuiltin` | The builtin table as data: one entry per name and arity, with its signature, its summary, which of its arguments are filters, and the names this package refuses with a reason. |
| `jqeval` | Running a program over a value: the outcome, the bindings, the evaluation limits, jq's total order, jq's truthiness, and the two path calls. |
| `jqerror` | The two error types. One is a program that did not parse. The other is a filter that failed while running. |
| `jqfmt` | Turning a JSON value into text: compact, pretty and raw, a table, CSV out, and the rule that gives a CSV cell a type. |

## How to choose an entry point

**Parse once, run many times.** `jqlang.parse` is where every mistake in a
program is caught, so a service that accepts programs from outside parses at the
edge. `jqeval.run` then runs the parsed program over one value.

**`jqeval.run_with` is `run` plus an environment.** The environment carries the
`$name` bindings a `--arg` flag would supply and the limits the run is held to.
`jqeval.run_stream` runs one program over a list of documents and answers one
outcome each.

**`jqlang.parse_with` is for a program that reads `$names` the caller will
bind.** Passing the names up front stops them being reported as unbound.

**`jqeval.select_path` follows jq's rules and `jqeval.select_path_rfc` follows
RFC 9535's.** The two disagree in four places, listed below. Pick the first for
a tool whose users know jq, and the second for a tool that also speaks JSONPath
and must answer the same way everywhere.

**`jqfmt.compact`, `.pretty` and `.raw` are the three renderings.** `raw` prints
a string without its quotes and everything else as JSON, which is what a shell
pipeline wants. `jqfmt.tabular` and `.csv_out` answer `None` for a value that is
not a table, and `jqfmt.table_problem` says why.

## The rules a user needs

1. **A filter answers a stream, not a value.** Zero outputs, one, or many. jq
   manual, "Invoking jq".
2. **Reaching into a container that has no such place is `null`. Reaching into
   something that is not a container is an error.** This is jq's rule, and it is
   where jq and RFC 9535 disagree. jq manual, "Basic filters"; RFC 9535 section
   2.3.

   | The path | jq, and this package | RFC 9535 |
   | --- | --- | --- |
   | `.foo` on an object without `foo` | `null` | selects nothing |
   | `.[3]` on a shorter array | `null` | selects nothing |
   | `.foo` on `null` | `null` | selects nothing |
   | `.foo` on a number, `.[0]` on an object, `.[]` on a string | an error | selects nothing |

3. **`.[]` on `null` is an error, even though `.foo` on `null` is `null`.** jq
   answers the two differently, and this package keeps that. jq manual, "Basic
   filters".
4. **A run answers the values it produced and the error it stopped on.**
   `JqOutcome` has both fields, because a filter that fails has usually already
   produced output. Read `values` for the rows and `fault` for the failure.
5. **A runtime error is a JSON value.** `JqError.value` is what `catch` binds.
   For `error(v)` it is exactly what the program passed. For every other kind it
   is the string jq would have printed, as a JSON string. jq manual, "Error
   Suppression / Optional Operator".
6. **`f?` is `try f` with no handler**, and the parser builds the same node for
   both. A round trip through `jqlang.render` prints `try f`. jq manual, "try-catch".
7. **Only `null` and `false` are falsy.** `0`, `""` and `[]` are all true, so
   `0 and 1` is true. jq manual, "if-then-else".
8. **Comparison never fails.** jq orders every pair of values:
   `null < false < true < numbers < strings < arrays < objects`, so `1 < "a"` is
   true. RFC 9535 section 2.3.5.2.2 orders only like against like.
   `jqeval.order` is jq's order. jq manual, "sort, sort_by, group_by".
9. **`keys` sorts and `keys_unsorted` does not.** `keys` answers an object's
   member names in sorted order. `keys_unsorted` answers them in document order,
   which is what a table's column order needs. jq manual, "keys, keys_unsorted".
10. **A builtin is a name and an arity.** `jqbuiltin.resolve("range", 2)` finds
    exactly one entry. A call with a count no form has is a parse fault naming
    the counts that do exist, from `jqbuiltin.arities_of`.
11. **Everything wrong with a program is wrong at parse time.** An unknown
    builtin, a wrong arity, an unbound `$name` and a path RFC 9535 refuses are
    all `JqFault`, with half-open byte offsets `at` and `to` into the program
    text. After a successful parse a program can only be slow.
12. **`test/1` matches with I-Regexp, not PCRE.** RFC 9485 has no anchors, no
    backreferences, no lookaround, no lazy quantifiers and no capture groups.
    A pattern that works here matches the same strings in every conforming
    implementation.
13. **Numbers are `Float`, however they were written.** JSON has one number
    type, so a program that says `> 30` matches a value stored as `30.5`.
14. **`{}` and `null` are the same value to every typed accessor of
    `std.json`.** Only the rendering separates them, so `length` over an empty
    object and over `null` are two answers that cost a render to tell apart.
15. **A run is bounded.** `JqLimits` caps the outputs produced, the filter
    applications made, and the depth recursed. A refusal for exceeding a limit is not
    catchable by `try`. `jqeval.default_limits` is the set a caller gets when it
    does not choose one.
16. **A CSV cell gets the narrowest type whose rendering is byte-identical to
    the cell's own text.** `120` is the number 120. `00417` is a string, because
    `417` does not render back as `00417`. `1.50` is a string for the same
    reason. An empty cell is `null`. `jqfmt.round_trips` is that rule as a
    predicate.

## What is not included

- **Six regex builtins: `match`, `capture`, `scan`, `splits`, `sub` and
  `gsub`, and `test/2`.** Each needs capture groups or match offsets, and
  I-Regexp has no capture semantics at all. `jqbuiltin.absent_named` and
  `jqbuiltin.absence_reason` publish the refusals as data, so a program that
  writes `gsub` is told what is missing rather than that the name is unknown.
- **The assignment operators `=`, `|=`, `+=` and `del` by assignment.** They
  need a path to write to rather than a value to read. `path(f)` and `getpath(p)`
  are here, so the reading half of paths is complete.
- **The builtins that read a machine: `env`, `$ENV`, `input`, `inputs`,
  `input_line_number`, `now`, `localtime`, `strftime`, `halt_error` and
  `debug`.** Nothing in this package opens a file, reads the environment or
  consults a clock. `jqbuiltin.absent_for_layer` says which absences a package
  with those permissions could close and which no package can.
- **`@base64` and the other `@` formats, and the SQL-style operators.**
- **A command line.** This package has no argv, no file reading, no standard
  input and no output. It answers text and values.
- **Running on a microcontroller.** `std.json`, the value this package queries,
  does not build for a device with no heap allocator. There is no probe here and
  there cannot be one until it does.

## Related packages

- [jsonpath-nv](https://novo-lang.org/packages/jsonpath-nv) is the RFC 9535
  JSONPath implementation this package's paths are. It answers a nodelist and
  never fails. Use it directly for a tool that speaks JSONPath and nothing else.
- [table-nv](https://novo-lang.org/packages/table-nv) draws a table for a
  terminal, with widths, borders and alignment. `jqfmt.grid` answers the cells
  and leaves the drawing to a caller.
- `std.json` in the standard library parses and renders JSON. It is where the
  value this package queries comes from, and `json.stringify` is what
  `jqfmt.compact` is written over.

## Tests

```bash
novo test tests                            # every suite
novo test tests/jqlang_tests.nv            # what parses, what is refused, where the caret goes
novo test tests/jqbuiltin_tests.nv         # the builtin table, as data
novo test tests/jqeval_tests.nv            # the evaluator and the four divergences
novo test tests/jqerror_tests.nv           # what each error type can say
novo test tests/jqfmt_tests.nv             # the CSV typing rule and the renderings
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
jsonquery-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests
are the specification the implementation will have to satisfy.

The reference is jq 1.7. Its manual is the specification for the grammar, the
builtin table and the rule that separates `null` from an error, and its own test
suite, `tests/jq.test`, is a list of a program, an input and the exact outputs it
must produce.

The suite asserts that every entry in the builtin table resolves from its own
name and arity, that a missing field is `null` and indexing a number is an
error, that an error arriving after output keeps the output, that `try`/`catch`
binds an error value, that `//` swallows a failure on its left, and that a CSV
cell takes the narrowest type that renders back as itself.

## Implementation status

| Item | Implemented |
| --- | --- |
| `jqlang.JqProgram`, `.JqFilter`, `.JqCallee`, `.JqLiteral`, `.JqBinOp`, `.JqField`, `.JqPattern`, `.JqPatternEntry`, `.JqStrPart` | declared |
| `jqlang.parse`, `.parse_with`, `.program`, `.field` | no |
| `jqlang.render`, `.render_filter`, `.position` | no |
| `jqlang.free_variables`, `.defined_names`, `.called_builtins`, `.is_path_expression`, `.depth`, `.path_fault` | no |
| `jqbuiltin.JqBuiltin`, the 69 name-and-arity entries | declared |
| `jqbuiltin.all`, `.name`, `.arity`, `.signature`, `.summary` | no |
| `jqbuiltin.resolve`, `.is_known_name`, `.arities_of` | no |
| `jqbuiltin.takes_filter_argument`, `.may_stream` | no |
| `jqbuiltin.absent_named`, `.absence_reason`, `.absent_for_layer` | no |
| `jqeval.JqOutcome`, `.JqLimits`, `.JqBinding`, `.JqEnv`, `.JqKind` | declared |
| `jqeval.default_limits`, `.empty_env`, `.env_with`, `.bind`, `.lookup` | no |
| `jqeval.run`, `.run_with`, `.run_stream`, `.apply` | no |
| `jqeval.select_path`, `.select_path_rfc` | no |
| `jqeval.produced`, `.raised`, `.failed`, `.exit_code` | no |
| `jqeval.kind`, `.type_name`, `.truthy`, `.order`, `.equal`, `.null_value` | no |
| `jqerror.JqFaultKind`, `.JqFault`, `.JqErrorKind`, `.JqError` | declared |
| `jqerror.fault`, `.fault_kind_name`, `.fault_message` | no |
| `jqerror.error`, `.error_kind_name`, `.error_message`, `.error_value`, `.is_catchable` | no |
| `jqfmt.JqShape` | declared |
| `jqfmt.cell_value`, `.round_trips`, `.csv_to_json` | no |
| `jqfmt.shape`, `.columns`, `.cell_text`, `.grid`, `.table_problem`, `.tabular`, `.csv_out` | no |
| `jqfmt.compact`, `.pretty`, `.raw` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
