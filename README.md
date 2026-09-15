# toml-edit-nv

[TOML](https://toml.io/en/v1.0.0) is a configuration file format whose
specification describes it as "a minimal configuration file format that's easy
to read due to obvious semantics". This package edits a TOML document without
disturbing how it was written. A document is parsed into items that each
remember the exact bytes they came from. An edit rewrites one item. Rendering
gives back every byte nobody touched, identical.

The model is the Rust crate [toml_edit](https://docs.rs/toml_edit), whose
`Document`, `Item`, `Decor` and `RawString` this package ports.
[config-nv](https://novo-lang.org/packages/config-nv) reads configuration and
does not write it. This is the package that writes.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **document** is the text of a TOML file plus a list of the items parsed out
of it. `TeditDoc` keeps the original text whole and never changes it. Every
edit answers a new document.

An **item** is one entry: a key, what it holds, and the bytes around both.
There are five kinds. A key-value pair is `port = 8080`. A standard table is
introduced by a header line, `[server]`. An inline table is written between
braces, `server = { port = 8080 }`. An element of an array of tables is written
`[[bin]]`. An implicit table is the one a dotted header creates: `[a.b.c]`
declares `a` and `a.b` without either appearing in the file, and an implicit
table renders as nothing at all.

**Decor** is the whitespace and the comments around an item. The prefix is what
came before it: the blank lines above a header, the indentation before a key,
a comment on its own line. The suffix is what came after it on the same line: a
trailing comment and the spaces before it. `toml_edit` uses the same word.

A **raw** is a run of bytes in a document, and it has two forms and no third. A
span is a range of the original text. A made raw is text this package produced.
Rendering is a concatenation of raws in order, so a byte the caller did not edit
comes back identical because it is the same byte.

A **splice** is one change: an offset range in the original text and the text to
put there. `teditdoc.edits` answers the list of them, and a caller applying them
itself needs no bookkeeping.

A **policy** decides what new bytes look like: the spaces around an `=`, how a
new string is quoted, the indentation under a header, where in a table a new
entry lands, and whether a new sub-table is written inline or as a header.
Nothing already in the document is subject to a policy.

## Install

```
novo pkg add toml-edit-nv
```

## Example

```novo
use std.list
use teditdoc
use tedititem
use teditfmt

fn main() [io]
    // A manifest as a tool would read it off disk, comments and blank lines and all.
    let text = "# the package\n[package]\nname = \"app\"\n\n[dependencies]\ntoml-nv = \"^0.0.3\"\n"

    match teditdoc.parse(text)
        Err(_) => println("that text is not TOML")
        Ok(d)  =>
            // Add one dependency. The policy governs only the bytes of the new line.
            match teditdoc.insert(d, "dependencies.crypto-nv",
                                  tedititem.string_value("^0.1.3"),
                                  teditfmt.manifest_policy())
                Err(_) => println("that key is already there")
                Ok(d2) =>
                    // One splice, against the original text's offsets.
                    println("${list.len(teditdoc.edits(d2))}")

                    // The new line, and the original bytes everywhere else.
                    println(teditdoc.render(d2))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `tedititem` | What a piece of a document is: the two forms of a raw, the decor, the five item kinds, a value that keeps its written spelling beside its meaning, and the constructors for a new value. |
| `teditkey` | Keys, their three spellings, comparing two by meaning, rendering one by style, and dotted paths. |
| `teditfmt` | The policy for what an edit adds, three named policies, and reading a policy off a document's own habits. |
| `teditdoc` | The document: parsing, rendering, the reads, and the six edits. |
| `tediterr` | The eight refusals, with the position for the one that has one. |
| `teditconv` | The crossing to toml-nv's value tree: reading a document out as data, and writing data back into a document. |

## How to choose an entry point

**A tool changing one entry calls `teditdoc.set`, `insert` or `upsert`.** `set`
requires the path to exist. `insert` requires it to be absent. `upsert` accepts
either, and it is the only lenient call in the package.

**A tool restructuring a file calls `remove`, `rename` or `retype`.** `rename`
moves a key and leaves the value, its decor and its trailing comment where they
were. `retype` changes a table between its inline and header spellings, and
refuses a change that would alter what the file means.

**A tool that wants the data rather than the document calls
`teditconv.to_value`.** It answers toml-nv's value tree with the trivia dropped.

**A tool merging data into a file calls `teditconv.apply_value`.** It writes a
subtree into an existing document and touches only the entries whose values
differ, so the diff is the size of the difference rather than the size of the
file. `replace_value` also deletes what the incoming tree does not carry.

**A tool that needs the changes rather than the text calls `teditdoc.edits`.**
It answers the splices against the original offsets, ascending, which is what a
language server sends as a `textDocument/didChange`.

For the output, `render` answers a string, `render_bytes` answers bytes,
`render_into` appends into a buffer the caller sized, and `render_to` writes
into any sink implementing `Write`.

## The rules a user needs

1. **A byte the caller did not edit comes back identical.** Every raw in a
   rendered document is either a span into the original or text this package
   made. `tedititem.origin_of` says which, and `teditdoc.made_count` counts the
   second kind.
2. **`teditdoc.edits` answers splices against the original text's offsets**,
   ascending. A caller applying them itself applies them back to front.
3. **The three key spellings name the same key.** TOML admits `name`, `"name"`
   and `'name'`. `teditkey.key_eq` compares the decoded names, and `render`
   writes the style the key carries. TOML v1.0.0, "Keys".
4. **A dot inside a quoted key is not a separator.** `a.b` is two segments and
   `"a.b"` is one key whose name contains a dot. `teditkey.path_segments`
   answers a `Result` for this reason, and `path_of` takes a list of keys.
   TOML v1.0.0, "Keys".
5. **A scalar keeps the bytes it was written as beside its meaning.** `0x2A`,
   `42` and `4_2` are one integer and three files. `tedititem.int_of` answers
   42 for all three, and `repr_of` answers the file's own bytes. TOML v1.0.0,
   "Integer".
6. **A string's decoded value and its written form are two different reads.**
   `path = "C:\\tmp"` is `C:\tmp` from `str_of` and `"C:\\tmp"` from `repr_of`.
7. **A newline belongs to the prefix of what follows, not to the suffix of what
   precedes.** Without that rule, deleting the last key of a table takes the
   blank line above the next table with it, and the file closes up by one line
   every time a tool runs.
8. **A policy governs new bytes only.** There is no path in this package from a
   policy to an existing byte.
9. **`teditfmt.infer` reads a policy off the document.** It counts what the
   file already does across the whole file rather than sampling the first pair.
   `infer_confidence` says how much evidence there was, which is almost none for
   a four-line file. `defaults()` is the fallback.
10. **`teditfmt.manifest_policy` pins two decisions.** New entries are appended,
    and new tables are written as headers. A manifest whose `[dependencies]`
    happens to be alphabetical today does not start sorting itself, and one with
    no inline tables does not grow a brace.
11. **No placement sorts an existing table.** `TeditSorted` inserts in the
    position the existing order implies, and falls back to appending when the
    table is not in that order. A half-sorted table has no correct insertion
    point.
12. **This package refuses rather than repairs.** A parse that cannot account
    for a byte refuses. `set` on an absent path refuses. `insert` on a present
    one refuses. A value whose written form does not read back as the kind it
    claimed refuses at the edit rather than at the next reader.
13. **`retype` refuses a change that would alter meaning.** An inline table has
    no spelling that can contain a header table, so `[a.b.c]` cannot become
    `b = { c = … }` while `[a.b.d]` stays where it is. TOML v1.0.0, "Inline
    Table".
14. **A parse error carries a line and a column, and an edit error does not.**
    A path that names nothing has no line in the file. `tediterr.error_line`
    answers 0 for those seven, and `render` leaves the colons out.
15. **Every edit answers a new document.** Nothing is changed in place, so the
    document before an edit is still a document and the two can be compared.
16. **`teditconv.to_value` is onto and not one-to-one.** A standard table and an
    inline table both become a table. An array of tables becomes an array of
    tables' values. This is why the other direction needs a policy.

## What is not included

- **Reading or writing a file.** A document is a string the caller already
  holds. `teditdoc.render_to` writes into a sink the caller supplies, and it
  costs whatever that sink costs.
- **A formatter.** There is no call that restyles a document. Formatting
  applies to new bytes only.
- **Sorting a table.** Sorting is a whole-file rewrite wearing an insert's
  clothes. A caller who wants it renders the keys itself.
- **Editing in place.** Every edit answers a new document, which is what makes
  the splice list meaningful.
- **Indexing a document like a map.** `toml_edit`'s `Document` allows
  `doc["a"]["b"]`, and a missing key there is a panic. Here a path is a value
  and a miss is a refusal.
- **Running on a microcontroller.** A document holds its whole source text, one
  item per entry, and a splice list that grows with the edits. All three are
  the size of the file. A device that wants to change one field of a
  configuration blob wants a fixed-capacity surface, not this one.
- **Parsing TOML into data.** [toml-nv](https://novo-lang.org/packages/toml-nv)
  does that, and `teditconv` is the crossing between the two.

## Related packages

- [toml-nv](https://novo-lang.org/packages/toml-nv) is the TOML value tree: it
  answers what a file means, and two files holding the same data with different
  comments are the same value. This package answers how a file is written, and
  those two files are different documents. A document can always produce a
  value, and a value cannot produce a document, so the dependency runs one way.
  toml-nv also ships a `tomledit` module that preserves formatting across a
  `set`. It is the smaller path: it has no `rename`, no `retype`, no formatting
  policy, and no splice list, and it appends a key it has to create rather than
  placing it. The names `tomledit`, `TomlEdit`, `TomlValue` and `TomlError` are
  toml-nv's, which is why every module and type here carries a `tedit` or
  `Tedit` prefix.
- [config-nv](https://novo-lang.org/packages/config-nv) reads configuration off
  files, the environment and the command line. It does not write. A program
  that writes settings back into a file it did not create calls
  `teditconv.apply_value`.
- [config-core-nv](https://novo-lang.org/packages/config-core-nv) merges
  configuration layers and looks values up by dotted path. Its paths and this
  package's are both dotted and are not the same grammar: this one follows
  TOML's key rules, with quoting rather than backslash escapes.
- [changelog-nv](https://novo-lang.org/packages/changelog-nv) does for a
  `CHANGELOG.md` what this package does for a `novo.toml`. It edits a file and
  keeps everything it did not change.
- `std.toml` in the standard library parses TOML into a value and renders a
  value back. It has no document model, so rendering a parsed file returns the
  canonical spelling rather than the author's.

## Tests

```bash
novo test tests                            # every suite
novo test tests/teditdoc_tests.nv          # the worked example, and one splice
novo test tests/tedititem_tests.nv         # which bytes came from where
novo test tests/teditkey_tests.nv          # three spellings of one key
novo test tests/teditsurface_tests.nv      # the rest of the surface
novo test tests/teditconv_tests.nv         # the crossing to toml-nv
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
toml-edit-nv.<module>.<fn>` panic, because every body is a `todo()`. The
`teditconv` suite reaches panics from toml-nv as well, which is an interface
too.

The parse cases come from [toml-test](https://github.com/toml-lang/toml-test)'s
valid and invalid corpora. The preservation cases come from `toml_edit`'s own
round-trip tests. One fixture is this project's own `novo.toml`, edited and
compared byte for byte outside the inserted line.

The suite asserts that adding a dependency to a manifest produces exactly one
splice, that a freshly parsed document reports every raw as original, that
`name`, `"name"` and `'name'` compare equal while each renders as itself, and
that `teditconv.apply_value` produces a diff the size of the difference rather
than the size of the file.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tedititem.TeditOrigin`, `.TeditRaw`, `.TeditDecor`, `.TeditItemKind`, `.TeditValueKind`, `.TeditItem`, `.TeditNewValue` | declared |
| `tedititem`'s five raw functions, from `raw_text` to `made` | no |
| `tedititem`'s decor and comment readers, from `decor_of` to `suffix_comment` | no |
| `tedititem`'s value readers, from `kind_of` to `value_untouched` | no |
| `tedititem`'s nine value constructors, from `string_value` to `new_value_ok` | no |
| `teditkey.TeditKeyStyle`, `.TeditKey`, `.TeditKeyError`, `impl Error for TeditKeyError` | declared |
| `teditkey`'s seventeen functions, from `key` to `key_error_at` | no |
| `teditfmt.TeditPlacement`, `.TeditTableStyle`, `.TeditStringStyle`, `.TeditPolicy` | declared |
| `teditfmt`'s ten functions, from `defaults` to `indent_for` | no |
| `teditdoc.TeditDoc`, `.TeditSplice` | declared |
| `teditdoc.parse`, `.parse_bytes` | no |
| `teditdoc.render`, `.render_bytes`, `.render_into`, `.render_len`, `.render_to` | no |
| `teditdoc.is_untouched`, `.edits`, `.made_count`, `.raw_str` | no |
| `teditdoc`'s reads, from `get` to `position_of` | no |
| `teditdoc.set`, `.insert`, `.upsert`, `.remove`, `.rename`, `.retype` | no |
| `teditdoc.set_comment`, `.comment_at` | no |
| `tediterr.TeditError`, `impl Error for TeditError` | declared |
| `tediterr.error_line`, `.error_col`, `.error_path`, `.is_parse_error`, `.render`, `.from_key_error` | no |
| `teditconv`'s eight functions, from `to_value` to `value_eq` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
