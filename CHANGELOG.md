# Changelog

All notable changes to toml-edit-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
