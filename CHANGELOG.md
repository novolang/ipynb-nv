# Changelog

All notable changes to ipynb-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
