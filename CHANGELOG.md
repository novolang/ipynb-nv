# Changelog

All notable changes to ipynb-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-10-12

The implementation. Every body published as `todo()` in 0.0.1 is
written, and `tests/ipynb_tests.nv` — the interface release's own suite
— passes with its assertions unchanged; what changed in that file is
its header, which no longer describes a red run.

### Added

- `tests/ipynb_edge_tests.nv` — 56 tests over the four claims the API
  suite cannot reach: what becomes of a member NO code in this package
  mentions when a notebook is read and written back; the cell
  identifier rule at BOTH versions, 4.4 reading and writing without the
  member and 4.5 requiring it; each of the four output kinds, and the
  refusal a kind's missing member produces; and the split and the join
  of a multiline string with an empty line in the middle and a second
  one with no trailing newline. Every refusal is pinned by its code and
  by the JSON Pointer it carries, the identifier alphabet by the bytes
  at both ends of each of its ranges, and the writer's four choices by
  the exact bytes of a notebook that has a cell, a stream output and a
  two-line source — the empty notebook the API suite writes has no
  nested object and could not pin either of the first two.
- **A stream's `text` is read in both of its spellings.** nbformat
  defines a multiline string as a string OR as an array of lines, and
  THIS PACKAGE WRITES THE ARRAY FORM — so a reader that knew only the
  string form loses the text of every stream in a file it has written
  itself, and reports a member that is plainly there as missing. It
  did; it no longer does. One `nbread.multiline_member` is now the
  reading for a cell's `source`, a stream's `text` and a text MIME
  payload.
- **An `execution_count` that is not an integer is reported, and
  recovered as never-run.** `std.json` has one number kind and its
  `to_int` TRUNCATES, so a file carrying `1.5` was written back with
  `1` — a small loss, and a save makes it permanent.
  `NbBadExecutionCount` is the report and `f == math.floor(f)` the
  check.
- Every line under `src/` is executed by the two suites but two, which
  carry `// cov: skip` with the reason written out: a `json.get` for a
  name `json.keys` has just reported cannot answer `None`.

### Changed

- **`write` and `write_at` part ways at 4.5, and the rule is stated in
  one place now.** 0.0.1's `write_at` doc comment said the requested
  version drops the cell identifiers, and said nothing about writing a
  notebook at the version it already carries. The writer now claims
  both halves and follows them: writing at the notebook's own version
  INVENTS nothing and DISCARDS nothing, so a 4.5 notebook keeps every
  identifier and a 4.4 notebook keeps whichever ones it has; and
  `write_at` below 4.5 is a CONVERSION that drops them, as nbformat's
  own `downgrade` does. `write_cell` keeps them too.
- **`x == false` is written `not x` at the eleven sites the lint's
  `redundant-bool` rule names in `tests/ipynb_tests.nv`.** Every site
  is pre-existing — `git show HEAD:tests/ipynb_tests.nv` counts the
  same eleven — and `grep -rn '== false' tests/ src/` and the linter
  agree on the set, so the rewrite is the whole of it. No predicate
  changed.

### Known

- **A member no code here mentions is dropped by the next save, and
  nothing reports it.** The model holds four members for a notebook, a
  cell's own set, and its kind's set for an output; a JSON member
  outside those has nowhere to live, so the reader neither carries it
  nor knows it was there. `metadata` and every MIME payload are the
  exception — arbitrary JSON, carried whole — and a newer format's keys
  UNDER them come back. This is data loss in a diff a user will notice:
  `tests/ipynb_edge_tests.nv::test_a_member_no_code_mentions_is_not_written_back`
  is where it is stated, so that changing it would be a deliberate act
  with a failing test to answer.
- **A 4.6 document is read and written at 4.6, with the members a
  future minor adds to a CELL dropped.** The version is carried and
  written back (`test_metadata_from_a_newer_nbformat_survives_the_round_trip`),
  the document's `metadata` is carried whole, and a cell member this
  package has never heard of goes the way of the bullet above.
- **A present-but-empty `attachments` object is not written back.** A
  markdown cell whose file wrote `"attachments": {}` comes back without
  the member, because the writer emits it only when there is at least
  one attachment. Nothing is lost and a save of such a file is a diff,
  so `nbwrite.is_canonical` answers false for it.
- **A JSON document under a `+json` MIME type is read, carried and
  reported.** A charting tool writes its renderer's model under
  `application/vnd.plotly.v1+json`, which is a JSON value here and a
  multiline string in nbformat's schema. The payload survives the round
  trip untouched and is named as `bad-mime-payload`; the reading is not
  blocked and `read_strict` refuses, because the schema does.
- **A float `std.json` spells in exponent form is written without
  Python's `+`.** `1e20` comes back as `1e20` where Python's `json`
  writes `1e+20`, so this package's bytes differ from nbformat's for a
  document carrying one, and the first save of a file nbformat wrote is
  a diff. Integers and ordinary floats are unaffected — `4.0` is
  written `4.0`.
- **`NbNotJson.at` is always `0`.** `std.json`'s parse refusal answers
  `None` and not an offset, so the member carries no information today;
  every site that builds the fault passes `0`.
- **`issues` is the reading's order and not a sorted list.** The
  reading's own faults come first, in document order, and everything
  `nbvalidate.check` finds — the identifier rules, a cell's forbidden
  members, the payload rules, the duplicate identifiers — is appended
  after them by a second walk. `read_strict` refuses on the first of
  that list; a caller that wants the fault at the smallest JSON Pointer
  sorts.
- **nbformat's own `tests/` directory is not generated into a run.**
  The suite carries those files reduced to the smallest document that
  shows the same rule, and a case is the TEXT of a notebook rather than
  a path, because reading a file costs `[fs]` and this package declares
  no effects. 0.0.1's Known list said the generated run would land with
  the implementation; it did not, and the README says why.
- **One assertion in `tests/ipynb_edge_tests.nv` calls
  `nbcell.bundle_mimes` where it would read `d.entries` off a pattern
  binding, as a workaround.** A field read off a binding in an enum arm
  is an internal compiler error in 0.20.2 when a sibling field of that
  variant is a handle type — `std.json`'s `JsonValueH`, which every
  bundle entry carries — and `novo build --check` passes the file it
  fires on. Filed in the toolchain's bugs tree as
  `type-system-meta/a-field-read-off-a-binding-in-an-enum-arm-is-an-e6000-ice-when-a-sibling-field`;
  the comment at the site names it, and the direct read goes back when
  the filing closes.

### Requires

- **`novo = ">= 0.20.2"`**, up from `>= 0.9.0`. 0.20.2 is the toolchain
  every suite in this release was run on, and the floor is the version
  the package's behaviour is verified against rather than a guess.

## [0.0.1]

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `nbread` and `nbvalidate` — the load-bearing interface, and it is a
  SPLIT rather than a function. A notebook comes out of a repository or
  off somebody's laptop, and a tool that refused to open one because
  its third cell was missing an identifier could not be the tool that
  fixes that notebook. So the reader refuses exactly four things — text
  that is not JSON, a top level that is not an object, a missing
  `nbformat`, a major version other than 4 — and everything else is a
  complaint beside a notebook it still hands back. The shape rules the
  format's schema enforces are `nbvalidate`'s, they answer EVERY issue
  rather than the first, and `nbread.read_strict` is the one call that
  makes them fatal. Each recovery the reader performs is named in its
  own documentation rather than left to be discovered.
- `nbwrite` — the output is byte-defined: sorted keys, one space of
  indentation, multiline strings as arrays of lines, a trailing
  newline. Those are nbformat's own four choices, and reproducing all
  four is what stops a repository learning which tool saved a notebook
  last. The array-of-lines form is the one that makes a notebook diff
  by the line rather than by the cell. A file that was not already
  canonical comes back canonical and not byte for byte, which is a
  deliberate one-way normalisation and is stated as such.
- `nbcell` — a multiline string is ONE text: both of nbformat's
  spellings read into one `Str`, because the split is not information
  about the document and a type that carried it would make every
  consumer handle two cases to read one string. `split_lines` is the
  writer's split, exposed. An output is four arms and not one struct
  with optional members, because a display carrying an `ename` is not a
  document nbformat permits.
- `nbnotebook` — the version is carried and not assumed, so a 4.4 file
  saved by this package is still 4.4. `upgraded` is the deliberate
  change, and it takes the fresh cell identifiers as an argument:
  generating one is random and a package with no effects may not be.
- `nberror` — sixteen reasons, each carrying the JSON Pointer of the
  member at fault, with `is_fatal` dividing the four that mean there is
  no document from the ones that mean the document is wrong.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  ipynb-nv.<module>.<fn>`.
- **nbformat's own test files are named but not generated.** The suite
  carries them reduced to the smallest document that shows each rule;
  the generated run over the whole directory lands with the
  implementation.
- **Execution is not here and is not coming.** A kernel is a process, a
  socket and the Jupyter messaging protocol, and this package declares
  no effects. It is the document a kernel's output ends up in.
- **No `tests/embedded_probe.nv`.** The absence is a claim not made
  rather than a claim skipped: a notebook is a tree of strings and
  `std.json` values, and `std.json` refuses the embedded tier outright.
