# Changelog

All notable changes to toml-edit-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tedititem` — `TeditRaw` and its two variants, the decor model, the
  five item kinds, a value that keeps its `repr` beside its meaning,
  and the constructors for a new value.
- `teditkey` — the three key spellings as a value, `key_eq` over
  decoded names, and dotted paths with the one place a string path and
  a key list disagree stated rather than discovered.
- `teditfmt` — `TeditPolicy`, the three named policies, and `infer`
  with a confidence.
- `teditdoc` — `parse`, `render`, `render_to<W: Write[e]>`, the reads,
  and the six edits: `set`, `insert`, `upsert`, `remove`, `rename`,
  `retype`.
- `tediterr` — eight refusals, each a refusal rather than a repair.
- `teditconv` — `to_value`, `from_value` under a policy, and
  `apply_value` / `replace_value` as two names rather than one flag.

### Known

- **`TeditRaw` is the load-bearing interface.** Every byte a render
  emits is either a span into the original or text this package made,
  so "untouched bytes come back identical" is a property of the type
  and `made_count` is how a test asserts it.
- **`teditdoc.edits` answers the splice list**, against the original's
  offsets, ascending — which makes `novo pkg add`'s "one line, nothing
  else moves" a one-line assertion.
- **A formatting policy can be read off the document** (`teditfmt.infer`),
  so a new line matches the file it lands in; `manifest_policy()` pins
  the two decisions a manifest must not leave to inference.
- **No `TeditPlacement` variant sorts an existing table**, deliberately.
- **`@tier(embedded)` is not claimed**, and the README says why: three
  heap structures whose size is the file's.
- **The overlap with toml-nv's `tomledit` is named in the README**, with
  a table of what each does, and the decision about the duplicate is
  left to the owner of both rows rather than taken here.
- One dependency, `toml-nv ^0.0.3`, for the value tree and nothing else.
- The scaffold's `src/toml_edit.nv` was dropped: `toml_edit` is not a
  module name this package wants, and `tomledit` is toml-nv's.

### Design notes

- An item tree rather than source text with patched spans. Patching
  spans answers `set` and cannot answer `insert` (there is no span to
  patch and a policy has to be consulted), `rename` (two splices whose
  offsets depend on each other) or creating a table that does not yet
  exist (where "the right place" is a statement about the tree).
- `from_value` exists although toml-nv's `tomledit` refuses the
  direction. The argument for refusing holds for a package with no
  formatting policy. Here the policy that generates a file is the same
  value that governs every later edit, so a generated lock file or a
  scaffolded manifest stays consistent with itself.
- `replace_value` is a separate name rather than a flag on
  `apply_value`. A `true` at a call site does not tell a reviewer that
  a config file's unmentioned half is about to go.
- Two things of `toml_edit`'s are deliberately not ported: mutation
  through a mutable reference, because this package answers new
  documents, and the `Document` deref that makes `doc["a"]["b"]` work
  and a missing key a panic.
- The overlap with toml-nv's `tomledit` is real and unresolved. Both
  packages are interfaces with no bodies. The defensible outcomes are
  that toml-nv's stays as the small path for a caller that only sets a
  value, or that it is dropped in favour of a toml-edit-nv dependency
  before either is implemented. Two implementations of decor
  preservation in one registry is not defensible. The decision belongs
  to whoever owns both.
- `novo pkg add`'s consumer is a pair, not a call. `manifest_insert_dep`
  and `manifest_key_pos` in the compiler scan the same file twice with
  two hand-written scanners. `teditdoc.insert` and
  `teditdoc.position_of` replace both or neither.
