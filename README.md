# template-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The small, safe string-template family — the one that is **not** Jinja.
Two halves:

- **substitution**, which is `${name}` and `{{name}}` and nothing else;
- **formatting**, which is C's `printf`, Python's `str.format`
  mini-language and Rust's `format!` spec, all three parsed into one
  typed spec and rendered by one renderer into a caller's buffer.

```
novo pkg add template-nv
novo pkg build
novo test
```

- `tmplspec` — the one format spec, and the padding arithmetic;
- `tmplsub` — `${name}` and `{{name}}`;
- `tmplprintf` — C's dialect;
- `tmplpy` — Python's `str.format`;
- `tmplrust` — Rust's `format!`, and the translation between all three;
- `tmplval` — the values a template is rendered against;
- `tmplerr` — what is wrong, and where.

## The one example that will work

```novo ignore
use tmplprintf
use tmplval

// A report line, in the dialect whoever wrote the configuration knows.
fn row(name: Str, total: Float) -> Str
    match tmplprintf.format("%-20s %10.2f", [TmplStr(name), TmplFloat(total)])
        Ok(line) => line
        Err(e)   => e.message()
```

## The load-bearing interface: `TmplSpec`

```novo ignore
pub struct TmplSpec
    fill: Int
    align: TmplAlign
    sign: TmplSign
    alternate: Bool
    zero_pad: Bool
    width: TmplCount
    precision: TmplCount
    grouping: Int
    kind: TmplKind
```

Why are three format languages one package? **Because they are three
syntaxes over one semantics.** Line them up:

```text
%-08.3f          printf
{:<08.3f}        Python
{:<08.3}         Rust
```

Every one says: left-align, pad with zeros, at least eight columns
wide, three digits after the point, fixed notation. The *spellings*
differ — where the fill goes, whether the alignment is a flag or a
suffix, whether the type letter is required — and the **decision** is
identical. So there are three parsers and one renderer, and `TmplSpec`
is the value between them.

What that buys is concrete. A program can read a format from a
configuration file in whichever dialect its users know. The renderer's
test suite is written once and covers all three. And the twenty edge
cases of padding — a negative number under zero-fill, a precision that
truncates a string, a width smaller than the content — are decided in
one place rather than in three places that will drift.

What it costs is said out loud: `TmplSpec` is the **union** of what the
three can express, so a spec can hold a combination its own dialect
cannot spell. `check_for` is the function that answers which, every
parser calls it, and `translate` refuses rather than approximating — a
grouping separator has no Rust spelling, and a template that silently
lost it would render a different document.

## Where the line with tera-nv is

`tmplsub` has **no control flow**. No `{% if %}`, no loops, no filters,
no includes, no expressions. A name is looked up and its text is
substituted. That is the whole language.

That is the line: **the moment a template needs a condition it needs
tera-nv, and the moment it does not, this is cheaper.** Cheaper to
read, cheaper to audit, and impossible to make recurse. Python keeps
`string.Template` in its standard library beside Jinja for exactly this
reason — the thing people reach a full engine for is usually a message
with three names in it.

Three differences follow from it, and each is a decision rather than an
omission:

- **The substitution does not rescan its own output.** A value
  containing `${x}` is inserted verbatim and is not looked up again.
  tera-nv's interface records the opposite behaviour as a defect in
  what it graduated from; this package does not repeat it — which also
  means a user-supplied value cannot reach into the context.
- **There is no auto-escaping.** `TmplEscape` is about the *template's*
  own syntax — how to write a literal `$` — and not about the value's
  destination. HTML escaping is html-nv's, applied by the caller to the
  value, because a template package that escaped on its own initiative
  would be guessing its output is a web page. tera-nv's output usually
  *is* one, so tera-nv escapes by default and says so.
- **A missing name is a decision.** `TmplMissingRefuse` is the default,
  because a template with a typo in a name otherwise renders a hole
  nobody sees.

## Two bug classes this port of printf does not have

C's `printf` is variadic and untyped. `printf("%d %d", 1)` reads
whatever was next on the stack, and `printf(user_string)` is a remote
read primitive.

Here the arguments are a typed list. The count is checked **before a
byte is rendered** — `TmplTooFewArguments` carries both numbers — and
`%d` against a string is `TmplWrongValueKind` rather than a garbage
number. A format string from outside can do nothing worse than fail to
render, and `is_safe_external` is the check to run on one before it is
even parsed.

**`%n` is refused by name.** It is the conversion that writes the byte
count back through a pointer; it is why format-string bugs are write
primitives and not only read ones. `TmplWriteBackRefused` is what a
string carrying one meets — a refusal rather than an absence, because a
string with `%n` in it came from somewhere that expected it to work.

The C length modifiers (`%lld`, `%zu`, `%Lf`) are **parsed and
ignored**: parsed because real format strings have them, ignored
because this language has one integer width and one float width.
`has_length_modifier` is how a caller porting C code finds the strings
that were relying on a width that is not here.

## The device claim, and the half it covers

`@tier(embedded)` is claimed for **`tmplspec`'s field arithmetic and
`tmplprintf`'s integer renderer**, and `tests/embedded_probe.nv` builds
them for a Cortex-M4.

Firmware logs need this. A sensor's log line is `"[%6d] %s"` and a
UART, and the host-tier calls cannot help it: they return a `Str` or
write into a `[u8]`, and a device with no allocator has neither. What
it has is the same question numfmt-nv's `digits` answers for a bare
number — *how many bytes, and what is byte `i`?* — asked of a number
inside a padded field:

```novo ignore
for i in 0 .. tmplprintf.int_len(v, 10, 6, 0, false)
    hal.uart.write_byte(tmplprintf.int_byte(i, v, 10, 6, 1, 0, false, 32))
```

That **is** "render into a caller's buffer" in the only sense a device
has a buffer. `bytes.set_u8` is not on the embedded tier's admitted
surface, so `render_into` is an `@tier(app)` call whatever its buffer
belongs to; numfmt-nv draws exactly this line first and this package
follows it, so a reader who knows one knows the other.

`tmplsub`, `tmplpy` and `tmplrust` are **not** claimed. All three build
lists of pieces and return `Str`, by construction rather than by
accident, and a device that needs them needs a heap.

## The value tree, and the reflection that is not there

`{user.name}` and `{rows[0].total}` walk `tmplval`'s tree. A caller
builds that tree from its own structs **by hand**, one line per field,
because novo-lang has no reflection:

```novo ignore
let ctx = tmplval.map_of([
    tmplval.pair("user", tmplval.map_of([tmplval.pair("name", TmplStr(u.name))])),
    tmplval.pair("total", TmplFloat(order.total))])
```

That cost is real and it is named rather than worked around. The
alternative — a run-time walk over a `Serialize` value — would give the
library the fields a sample happened to populate and silently miss the
rest, which is the same limit openapi-nv's interface records for schema
derivation.

Two smaller rules that follow the references exactly:

- **Python's subscript rule.** An unquoted subscript is a *string key*
  unless every character is a digit: `{d[1]}` is the index 1 and
  `{d[one]}` is the key `"one"`, and neither takes quotes, because the
  mini-language has no quoting at all.
- **A bool is not a number.** C would take one as an integer and print
  `1`; a template that did so would render a checkbox as a digit, and
  the refusal is more useful than the coercion.

## The layer, and the one dependency

`core`. Formatting is arithmetic over bytes and a caller's own buffer:
nothing is read, nothing is written, and **a library does not print** —
every renderer here returns text or takes the buffer.

numfmt-nv is the only dependency, and it is the right one: `%d` is itoa
and `%.3f` is ryu, neither is small, and this registry already has a
measured port of both. It is `core`, it makes its own device claim, and
its `digits.digit_count` / `digits.digit_byte` are exactly the
allocation-free shape this package's embedded half is built on — so the
dependency is what makes the device claim possible rather than what
threatens it.

## The reference implementations

glibc's `printf(3)` for the C dialect, CPython's `str.format` for the
second, and Rust's `std::fmt` for the third, with `string.Template` for
the substitution half. The expected strings in `tests/` are what those
produce, and the cases chosen are the ones where implementations differ
by one byte: a negative number under the zero flag, a precision on a
string, an alternate form on zero, a width smaller than the content,
and a centred field with an odd number of pad bytes.

## Status

**NOT IMPLEMENTED — interface only.** `0.0.1`, `stability = "draft"`,
recorded `implemented = false` on the registry. The first
implementation is the `0.1.0` published over it.

```
novo pkg build     # clean: the signatures type-check and the rows fit
novo test          # red: every body is a todo()
```
