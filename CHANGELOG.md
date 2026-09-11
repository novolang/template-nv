# Changelog

All notable changes to template-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tmplspec` — `TmplSpec`, the one format spec three dialects parse
  into, with `check_for` saying which of them can spell a given one,
  and the padding arithmetic as integer algebra at `@tier(embedded)`.
- `tmplsub` — `${name}`, `{{name}}` and `{name}` with the syntax, the
  escape and the missing-name policy as values.
- `tmplprintf` — C's dialect, with the argument count checked before a
  byte is rendered and `%n` refused by name.
- `tmplpy` — Python's `str.format` mini-language: field paths,
  conversions, and one level of nesting.
- `tmplrust` — Rust's `format!` spec, and `translate` between all
  three.
- `tmplval` — the value tree a field path walks.
- `tmplerr` — every refusal with the byte it is at, and
  `is_format_fault` to tell a bad string from a bad call.

### Known

- **`TmplSpec` is the load-bearing interface.** `%-08.3f`,
  `{:<08.3f}` and `{:<08.3}` are one decision written three ways, so
  there are three parsers and one renderer.
- **The spec type is the UNION of the three**, so a spec can hold a
  combination its dialect cannot spell; `check_for` answers which, and
  `translate` refuses rather than approximating.
- **`tmplsub` has no control flow**, which is the line with tera-nv:
  the moment a template needs a condition it needs tera-nv.
- **The substitution does not rescan its own output**, so a
  user-supplied value cannot reach into the context.
- **There is no auto-escaping.** HTML escaping is html-nv's, applied by
  the caller.
- **The argument count is checked before rendering**, and `%n` is
  refused by name — the two bug classes C's printf has.
- **The C length modifiers are parsed and ignored**, because this
  language has one integer width; `has_length_modifier` says when a
  string relied on another.
- **`@tier(embedded)` is claimed for `tmplspec` and `tmplprintf`'s
  integer half**, and `tests/embedded_probe.nv` builds it for a
  Cortex-M4. The three that build lists are not claimed.
- **No reflection**, so a value tree is built by hand; the cost is
  named rather than worked around.
- **One dependency**, numfmt-nv, which is what makes the device claim
  possible.
