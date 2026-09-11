# toml-edit-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Format-preserving TOML editing. A document is parsed into items that
each remember the exact bytes they were written as; an edit rewrites
one item; rendering the document gives back every byte nobody touched,
identical.

- `tedititem` — `TeditRaw`, and what a piece of a document is;
- `teditkey` — keys, their three spellings, and dotted paths;
- `teditfmt` — the policy for what the editor **adds**, and reading one
  off a document;
- `teditdoc` — the document, and the six edits;
- `tediterr` — what it refuses;
- `teditconv` — the bridge to toml-nv's value tree.

```
novo pkg add toml-edit-nv
novo pkg build
novo test
```

## The one example that will work

This is `novo pkg add`. One dependency line goes into a manifest, and
nothing else in the file moves.

```novo ignore
use teditdoc
use tedititem
use teditfmt

fn add_dep(manifest: Str, name: Str, range: Str) -> Str
    let d  = teditdoc.parse(manifest)!
    let d2 = teditdoc.insert(d, "dependencies." + name,
                             tedititem.string_value(range),
                             teditfmt.manifest_policy())!
    teditdoc.render(d2)
```

And the claim about it is a line of test, not a paragraph of README:

```novo ignore
test.assert(list.len(teditdoc.edits(d2)) == 1)
```

## The load-bearing interface: `TeditRaw`

```novo ignore
pub enum TeditRaw
    TeditSpan(start: Int, end: Int)
    TeditMade(text: Str)
```

Two variants and no third. Every key, every value, every run of
whitespace and every comment in a parsed document is one of them, so
rendering is a concatenation of raws in order and the property the
whole package exists for —

> a byte the caller did not edit comes back identical

— is **structural** rather than promised. It cannot be broken by an
implementation that reflows a line or normalises a quote, because doing
either means replacing a `TeditSpan` with a `TeditMade`, and
`tedititem.origin_of` reports that to whoever asks. A test asserts the
property directly (`teditdoc.made_count(d) == 1`) instead of comparing
two files and hoping the diff noticed.

`teditdoc.edits` is the same fact from the other side: a list of
splices against the **original** offsets, ascending, which a caller
applies back to front with no bookkeeping — and which a language server
sends as a `textDocument/didChange`.

## Why an item tree, and not text with patched spans

The other design keeps the source, rewrites the span a `set` names, and
re-parses. toml-nv's `tomledit` is exactly that, and it is a perfectly
good answer for `set`. It cannot answer the three calls this package
exists for:

- **`insert`** has to decide what whitespace and what newline come
  *with* the new entry, and there is no span to patch, because the
  entry was never there. A policy has to be consulted; a design with no
  item model has nowhere to consult one from.
- **`rename`** moves a key and leaves the value, its decor and its
  trailing comment where they were. Over raw text that is two splices
  whose offsets depend on each other, with a re-scan of the line to
  find where the key ended and the whitespace began.
- **A `[dependencies]` table that does not exist yet** has to be
  created in the right *place* — at the end of the file, not inside
  `[package]` — and "the right place" is a statement about the item
  tree.

So the model is a tree of items each carrying its decor, and the text
is what the tree renders to rather than what it is.

## The second decision worth arguing: a formatting policy that can be *read off the file*

An editor needs a formatter for exactly one thing: the bytes of
something new. Nothing that was already in the document is subject to a
style, because a `TeditSpan` is not something a style can change — so
unlike every "formatter with a preserve flag", there is no path in this
package from a policy to an existing byte.

What the policy then has to get right is making a new line look like it
belongs, and a fixed policy cannot: a manifest that writes `key = value`
and one that writes `key=value` should each get a line that matches.
`teditfmt.infer(source)` answers the policy a document's own habits
imply — counted across the file rather than sampled from the first
pair — and `teditfmt.infer_confidence` says how much evidence there
was, because a four-line file gives almost none and a tool previewing
its edit should be able to say so.

`teditfmt.manifest_policy()` is the deliberate opposite: `TeditAppend`
and `TeditHeaderTable` pinned, so that a `novo.toml` whose
`[dependencies]` happens to be alphabetical today does not start
sorting itself, and one with no tables at all does not grow a brace.

There is no `TeditPlacement` variant that sorts an existing table.
Sorting is a whole-file rewrite wearing an insert's clothes.

## The overlap with toml-nv, named

toml-nv 0.0.3 ships a `tomledit` module with a `TomlEdit` type that
preserves formatting. That overlap is real and this package does not
pretend otherwise; the two are a **simple path and a complete one**,
and the differences are checkable rather than a matter of taste:

| | toml-nv `tomledit` | toml-edit-nv |
| --- | --- | --- |
| model | source text + spans, re-parsed per edit | item tree with decor |
| `set` | yes | yes |
| `insert` a key that is absent | appends at the end of its table, no policy | `TeditPlacement`, and creates the parent tables |
| `rename` | — | yes |
| a formatting policy for what is added | — | `TeditPolicy`, inferable from the document |
| inline vs standard table | — | `retype`, refusing what would change meaning |
| array of tables | — | `TeditArrayTable` |
| what an edit changed | `is_unchanged`, a boolean | `edits`, the splice list |
| `from_value` | refused, for a stated reason | under a policy, for the stated reason |

**What should happen to toml-nv's `tomledit` is a question for its
maintainers, and this lane did not answer it.** Both packages are
interfaces; neither has a body. The defensible outcomes are that
toml-nv's stays as the small path for a caller that only ever sets a
value, or that it is dropped in favour of a `toml-edit-nv` dependency
before either is implemented. What is *not* defensible is two
implementations of decor preservation in one registry, and this README
is where that is on the record.

The name collision is already binding: `tomledit` and `TomlEdit` are
toml-nv's, so this package's modules and types carry a `tedit` / `Tedit`
prefix. `docs/publishing.md` § Public type names are globally unique is
the rule, and it cost this package its most obvious names.

## The direction of the dependency

`teditconv` is the whole of it. A document can always answer a value
and a value cannot answer a document, so toml-edit-nv depends on
toml-nv and never the reverse.

`teditconv.apply_value` is the call that earns the crossing: it writes
a subtree into an existing document and touches only the entries whose
values differ, so a config merged from a template produces a diff the
size of the difference rather than the size of the file. Rendering the
merged tree instead would have rewritten every byte.

`replace_value` is the same call that also deletes what the tree does
not carry, and it is a **separate name rather than a flag** — reading a
`true` at a call site does not tell a reviewer that a config file's
unmentioned half is about to go.

## The layer, and why

`core`. A document is a string the caller already holds, an edit is
arithmetic over byte ranges into it, and rendering is a concatenation.
Nothing is read, nothing is written, no clock is consulted.

The one place a stream could have entered is writing an edited document
out, and that is the effect-polymorphic shape:

```novo ignore
pub fn render_to<W: Write[e]>(w: W, d: TeditDoc) -> ?IoError [e]
```

The clause is `[e]`, bound by the caller's `Write` impl, so a file
costs `[fs]`, a terminal costs `[io]` and an in-memory buffer costs
nothing — and this package has spent none of them.
`docs/publishing.md` § How a `core` package takes a stream from its
host is the rule.

Every edit answers a **new** document: `[mutate]` is a host effect and
there is none here, which is also what makes `edits` meaningful,
because the document before an edit is still a document and the two can
be compared.

## `@tier(embedded)` is not claimed

Deliberately. A document holds its whole source text plus one item per
entry, and the splice list grows with the edits — three heap
structures whose size is the file's. A device that wanted to change one
field of a configuration blob wants a different surface (find the key,
overwrite in place, fixed capacity), not this one with an annotation on
it. The honest form is the absence of the claim, and
`docs/publishing.md`'s rule — a device claim is built, not asserted —
means a claim made here would have to be kept by a package whose whole
job is building lists.

## The reference implementation

`toml_edit` (Rust). The shape ported is its `Document` / `Item` /
`Value` / `Table` / `InlineTable` / `ArrayOfTables` split, its `Decor`
of prefix and suffix, and its `Formatted<T>` — a value that keeps its
`repr` beside its meaning, which is why `0x2A`, `42` and `4_2` are one
integer and three files here too. `toml_edit`'s own `RawString`, which
is either a span into the original or an owned string, is `TeditRaw`.

Two things are deliberately **not** ported. `toml_edit` mutates a
document in place through `&mut`; this package answers a new document,
because `[mutate]` is a host effect and `core` has none. And
`toml_edit`'s `Document` derefs to its root table, which makes
`doc["a"]["b"]` work and makes a missing key a panic; here the path is
a value and a miss is a `Result`.

The test vectors are `toml-test`'s valid and invalid corpora for the
parse half, and `toml_edit`'s own `test_parse` / `test_edit` round-trip
cases for the preservation half. The one vector this package adds is
its own: `novo.toml`, edited, compared byte for byte outside the
inserted line.

## The consumers, and what adopting this would take

**`novo pkg add`** (`compiler/bin/novo.ml`, `manifest_insert_dep`) is
the first and the one the design was measured against. It is 40 lines
that read the file into a line array, find the `[dependencies]` header,
scan forward for the last non-blank line before the next header, and
splice a string in. It exists in that shape because the whole-manifest
re-serialiser it replaced (`manifest_to_string`) dropped `stability`,
`category`, `tags`, `repository`, `maintainers`, `[docs.pages]` and
every comment — and on an interface package the casualty was
`stability = "draft"`, which the next `novo pkg publish` requires, so
the publish was refused for a line the tool itself had deleted.

`teditdoc.insert` with `teditfmt.manifest_policy()` is that function,
and it gains three things the line scanner cannot have: it creates
`[dependencies]` in the right place when the table is absent (the
scanner appends a table at the end of the file whatever is there), it
refuses a dependency that is already present instead of adding a second
line for it, and it knows that a `#` inside a string is not a comment.

**`manifest_key_pos`** — the function that answers `novo.toml:13:1` for
a refusal — is `teditdoc.position_of`, and the difference is the same
one: it scans lines for `^key`, so a key inside a `[docs.pages]` table
with the same name as a `[package]` key answers the wrong line.

**A dependency bot**, a lock-file writer and `novo pkg init --force`
are the same call with a different policy.

**config-nv** (planned, `tooling`/`host`) writes settings back to a
file it did not create, which is `teditconv.apply_value` exactly.

## What a row wanted to widen

Nothing widened. Every function in this package is `[]` except
`render_to`, whose row is its caller's.

Three things the plan's row did not say, which this lane found:

1. **The row's own job is partly already done.** toml-nv 0.0.3 ships a
   `tomledit` module. The plan's note — "format-preserving edits, which
   `novo pkg` needs" — reads as though nothing on the registry does
   any of it. § The overlap with toml-nv above is the finding, and the
   decision belongs to whoever owns both rows.
2. **The name the row implies is taken.** `tomledit`, `TomlEdit`,
   `TomlValue`, `TomlPair`, `TomlError`, `TomlType` and `TomlStyle` are
   toml-nv's, so the obvious module and type names for this package
   were unavailable before a line was written. The `tedit` prefix is
   what is left, and it is in the manifest and the README rather than
   discovered by a reader.
3. **`novo pkg add`'s consumer is a pair, not a call.**
   `manifest_insert_dep` and `manifest_key_pos` are the same file
   scanned twice by two hand-written scanners, and only one of them is
   in the plan's note. Adopting this package replaces both or neither.

## The surface

| module | `pub fn` | `pub struct` | `pub enum` |
| --- | --- | --- | --- |
| `tedititem` | 28 | 3 | 4 |
| `teditkey` | 17 | 1 | 2 |
| `teditfmt` | 10 | 1 | 3 |
| `teditdoc` | 31 | 2 | 0 |
| `tediterr` | 6 | 0 | 1 |
| `teditconv` | 8 | 0 | 0 |
| **total** | **100** | **7** | **10** (45 variants) |

Two `impl Error` blocks, for `TeditError` and `TeditKeyError`.
