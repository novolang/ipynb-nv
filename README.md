# ipynb-nv

A Jupyter notebook is a document that holds prose, source code and the
output that code produced, in one JSON file with the extension
`.ipynb`. Its format is specified by
[nbformat](https://nbformat.readthedocs.io/en/latest/format_description.html),
and this package reads, checks and writes the current version of it,
4.5, in novo-lang.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a notebook is

A `.ipynb` file is one JSON object with four members.

| Member | What it holds |
| --- | --- |
| `nbformat` | The major version of the format. `4` for everything current |
| `nbformat_minor` | The minor version. `5` is the current one |
| `metadata` | A JSON object. The kernel and the language are named here, and every tool keeps its own keys here |
| `cells` | An array of cells, in the order they are shown |

A **cell** is one JSON object. Its `cell_type` is `"code"`,
`"markdown"` or `"raw"`, and its `source` is the text it holds. A code
cell also has `outputs`, an array of what running it produced, and an
`execution_count`, which is the number shown beside it or `null` if it
has never run. A markdown cell may have `attachments`, which are files
its text refers to as `attachment:<name>`.

From version 4.5, every cell also has an `id`: between one and
sixty-four characters of letters, digits, `-` and `_`, unique within
the notebook. Before 4.5 the member did not exist, and a cell could
only be named by its position, which changes whenever a cell above it
is inserted.

An **output** is one of four shapes, named by its `output_type`.

| `output_type` | What it holds |
| --- | --- |
| `stream` | A `name`, which is `"stdout"` or `"stderr"`, and the `text` written to it |
| `display_data` | A MIME bundle, and metadata about it |
| `execute_result` | A MIME bundle, metadata, and the execution count it belongs to |
| `error` | An exception name, its value, and the traceback as the kernel formatted it |

A **MIME bundle** is a JSON object whose members are MIME types and
whose values are the same thing rendered each of those ways: a table as
`text/html` and as `text/plain`, a chart as `image/png` and as
`application/json`. A consumer picks the rendering it can show. A
payload under `text/*` is text; a payload under `application/json` or a
type ending in `+json` is a JSON value; anything else is base64 text.

nbformat calls a `source`, a stream's `text` and a text MIME payload a
**multiline string**, and defines it as *either* a JSON string *or* a
JSON array of strings whose concatenation is that string. Both
spellings are in real files.

## Install

```
novo pkg add ipynb-nv
```

## Example

```novo
use nbread
use nbcell
use nbnotebook
use nbwrite
use nbvalidate

fn main() [io]
    // The text of a .ipynb file, which the caller read from disk.
    let text = "{\"cells\":[],\"metadata\":{},\"nbformat\":4,\"nbformat_minor\":5}"

    match nbread.read(text)
        Err(e) => println("not a notebook: ${e.message()}")
        Ok(r) =>
            // Anything the file did that the format does not allow.
            // The notebook is still here; this is a list, not a refusal.
            println("${list.len(r.issues)} issue(s)")

            // Add a cell. The identifier is the caller's, because
            // generating one needs randomness this package cannot have.
            let nb = nbnotebook.append(r.notebook,
                                       nbcell.code_cell("a1b2c3d4", "print(1)"))

            // Check it against the rules for the version it carries.
            println("${list.len(nbvalidate.check(nb))} rule(s) broken")

            // Write it back in the byte-for-byte shape nbformat writes.
            match nbwrite.write(nb)
                Err(e) => println("cannot write: ${e.message()}")
                Ok(out) => println(out)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: ipynb-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `nberror` | Every refusal, each carrying the JSON Pointer of the member at fault. |
| `nbcell` | A cell, the four output shapes, the MIME bundle and the attachment, and the split that turns a text into the lines a file holds. |
| `nbnotebook` | The document: its version, its metadata and its cells, with the calls that add, move and clear them. |
| `nbread` | Text to a notebook, tolerant, with every recovery reported. |
| `nbwrite` | A notebook to text, in the shape nbformat's own writer produces. |
| `nbvalidate` | The shape rules the format's schema enforces, as a list of complaints. |

## How to choose an entry point

**`nbread.read` opens a notebook a person has to be able to fix.** It
refuses only what makes the document unreadable and hands back a
notebook with every shape complaint beside it. A notebook editor, a
viewer and a repair tool all want this one.

**`nbread.read_strict` refuses a notebook the format does not allow.**
The notebook it answers satisfies its own version by construction, so a
caller never has to check. A pipeline that must not accept a malformed
file wants this one.

**`nbread.format_version` reads the two version members alone.** Use it
before opening a large file, and in a tool that supports more than one
format.

**`nbwrite.write` produces the file.** `nbwrite.canonicalise` is read
and write in one call, which is what a repository hook and a `--fix`
flag do. `nbwrite.is_canonical` asks whether writing would change
anything.

**`nbvalidate.check` reports every rule a notebook breaks.**
`nbvalidate.blocking_issues` reports only the ones that stop it being
written, which is the set `nbwrite.write` refuses over.

## The rules a user needs

1. **The reader is tolerant and the validator is not.** `nbread.read`
   refuses only four things: text that is not JSON, a top-level value
   that is not an object, a missing `nbformat`, and an `nbformat` whose
   major version is not 4. Everything else is an issue on a notebook it
   still hands back.
2. **Each recovery is named.** A missing `metadata` becomes an empty
   object; a missing `cells` becomes no cells; a cell with no `id`
   keeps an empty one; a cell with an unknown `cell_type` is dropped;
   an output with an unknown `output_type` is dropped; an
   `execution_count` that is not an integer becomes none. Each one is
   an issue in the result.
3. **A dropped cell is a lost cell.** A `cell_type` this package does
   not know could be anybody's source code, and guessing at it is worse
   than reporting it gone.
4. **A multiline string is read from either spelling and written as an
   array of lines.** The two spellings hold the same text, so the
   package holds one `Str`. `nbcell.split_lines` is the split the
   writer performs, and each line keeps its own `\n`.
5. **The writer's output is byte-defined.** Members are written in
   sorted key order, indentation is one space per level, multiline
   strings are arrays of lines, and the file ends with a newline. Those
   are nbformat's own four choices, so a notebook this package writes
   is the file Jupyter would have written.
6. **A file that was not already canonical does not come back byte for
   byte.** It comes back canonical. `nbwrite.is_canonical` says in
   advance whether the first save will produce a diff;
   `nbwrite.canonicalise` is idempotent from the second call onwards.
7. **A cell identifier is one to sixty-four characters** of letters,
   digits, `-` and `_`, and is unique within the notebook.
   `nbcell.is_valid_cell_id` is the check, and
   `nbnotebook.duplicate_ids` finds the collisions.
8. **A fresh cell identifier comes from the caller.** Generating one
   needs a random source and this package declares no effects.
   `nbcell.cell_id` sets it, and `nbnotebook.upgraded` takes as many as
   `nbnotebook.cells_without_id` says it needs.
9. **The version is carried, not assumed.** A notebook read at 4.4 is
   written back as 4.4. `nbnotebook.upgraded` is the deliberate change,
   and `nbwrite.write_at` saves at a version other than the notebook's
   own.
10. **A notebook that does not satisfy its own version is not
    written.** A 4.5 notebook with a cell that has no identifier is
    `NbNotWritableAt`, because writing it would produce a file claiming
    a version it does not meet.
11. **`execution_count` of zero and of none are different.** Zero is a
    cell that ran first. None is a cell that never ran.
12. **Clearing outputs clears the execution counts too.**
    `nbnotebook.cleared` does both, because a notebook with outputs
    removed and counts left is valid and confusing.
13. **A MIME bundle's order is the file's and means nothing.** Which
    rendering is best is the renderer's decision, which is what
    `nbcell.bundle_prefer` takes a preference list for.
14. **Everything outside `text/*`, `application/json` and the
    `+json`-suffixed types is base64.** `nbcell.is_base64_mime` is the
    rule, and `nbcell.bundle_bytes` decodes.
15. **A cell of one kind may not carry another kind's members.** A
    markdown cell has no `outputs`, a raw cell has neither `outputs`
    nor `execution_count`. `nbvalidate.allowed_members` and
    `nbvalidate.required_members` state it.
16. **Every issue is reported, not the first.** A person handed one
    complaint fixes it and is handed the next; a person handed all of
    them fixes the file once.

## What is not included

- **Running a cell.** Execution is a kernel: a process, a socket and
  the Jupyter messaging protocol. This package declares no effects and
  starts nothing. It reads and writes the document a kernel's output
  ends up in.
- **The Jupyter messaging protocol.** The ZeroMQ channels a front end
  and a kernel talk over are a different specification.
- **nbformat versions 1, 2 and 3.** They are different documents, and
  a reader that half-understood one would be worse than one that says
  so. `NbUnsupportedFormat` names the version it was given.
- **A JSON Schema engine.** nbformat ships a schema per version, and
  this package writes the rules out instead. A schema engine reports a
  notebook's problems in the schema's vocabulary, and a person fixing a
  notebook needs them in the notebook's.
- **Rendering.** Turning a markdown cell into HTML, or an `image/png`
  payload into pixels, belongs to the tool that draws them.
- **Reading the file.** Opening a path costs `[fs]`. The caller reads
  the text and hands it over.
- **A diff between two notebooks.** Matching cells across a save is
  what the 4.5 identifiers are for, and a diff tool built on them is a
  package of its own.

## Related packages

- [json-nv and the standard library's `std.json`](https://novo-lang.org/packages/jsonpath-nv)
  hold the JSON values this package carries for everything the format
  leaves open.
- [markdown-nv](https://novo-lang.org/packages/markdown-nv) renders the
  source of a markdown cell.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is the encoding
  a binary MIME payload is written in.
- [mime-nv](https://novo-lang.org/packages/mime-nv) parses and compares
  the MIME types a bundle's members are keyed by.
- [diff-nv](https://novo-lang.org/packages/diff-nv) compares the text of
  two cells once a caller has matched them by identifier.

## Test vectors

The normative source is the nbformat format description and the JSON
Schema it ships for version 4.5, together with the release note that
introduced the cell identifier.

The oracle is **nbformat's own test suite**: the `.ipynb` files it ships
— including one that is valid, one that is not, and one with no minor
version — each of which its `validate` accepts or rejects, and its
round-trip test, which reads a file, writes it and compares the bytes.
The cases in this package's suite are those files reduced to the
smallest document that shows the same rule, and the whole directory
becomes a generated run beside the hand suite when the implementation
lands.

```bash
novo test tests/ipynb_tests.nv    # the document, the cells, the writer, the rules
```

The suite asserts that a notebook with a cell missing its identifier
opens with one issue and is refused by the strict reading, that a cell
with an unknown type is dropped and reported, that both spellings of a
multiline string read as the same text, that the split keeps each
line's newline, that the writer sorts its keys and ends with a newline,
that canonicalising is idempotent from the second call, that a 4.5
notebook with an identifierless cell is not written, and that the cell
identifier rule applies at 4.5 and not at 4.4.

The tests compile today and fail at run, each on the
`not implemented: ipynb-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `nbcell.NbCell`, `.NbOutput`, `.NbMimeBundle`, `nbnotebook.NbNotebook` and the other types | the types are declared |
| `nberror.path_of`, `.code_of`, `.is_fatal`, `NbFault.message` | no |
| `nbcell.code_cell`, `.markdown_cell`, `.raw_cell`, `.kind_name`, `.kind_named` | no |
| `nbcell.is_valid_cell_id`, `.cell_id`, `.with_outputs`, `.cleared`, `.with_execution_count`, `.with_source`, `.with_metadata`, `.with_attachment`, `.attachment` | no |
| `nbcell.stdout_text`, `.first_error`, `.output_type_name` | no |
| `nbcell.bundle`, `.bundle_with`, `.bundle_get`, `.bundle_mimes`, `.bundle_prefer`, `.bundle_text`, `.bundle_bytes`, `.is_base64_mime` | no |
| `nbcell.split_lines`, `.join_lines` | no |
| `nbnotebook.notebook`, `.notebook_at`, `.cell_count`, `.cell_at`, `.cell_by_id`, `.index_of` | no |
| `nbnotebook.append`, `.insert`, `.replace_at`, `.remove_at`, `.with_cells`, `.with_metadata` | no |
| `nbnotebook.kernel_name`, `.language_name`, `.cleared`, `.code_source`, `.cells_of_kind` | no |
| `nbnotebook.upgraded`, `.cells_without_id`, `.duplicate_ids` | no |
| `nbread.read`, `.read_json`, `.read_strict`, `.format_version`, `.is_notebook` | no |
| `nbread.read_cell`, `.read_output`, `.read_bundle`, `.multiline` | no |
| `nbwrite.write`, `.write_at`, `.to_json`, `.write_cell` | no |
| `nbwrite.is_canonical`, `.canonicalise`, `.member_order`, `.indent_width` | no |
| `nbvalidate.check`, `.check_at`, `.first_issue`, `.is_valid`, `.blocking_issues` | no |
| `nbvalidate.check_cell`, `.check_output`, `.check_bundle`, `.first_issue_of_cell`, `.issue_text` | no |
| `nbvalidate.allowed_members`, `.required_members` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
