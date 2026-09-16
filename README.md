# template-nv

A string template is text with named holes in it, filled from values a
program supplies. A format specification is the other half of the same
job: how one value is written, in how many columns, with how many
decimal places. This package implements both for novo-lang. The
substitution follows Python's
[`string.Template`](https://docs.python.org/3/library/string.html#template-strings),
and the three format dialects follow C's
[`printf(3)`](https://man7.org/linux/man-pages/man3/printf.3.html),
Python's
[format specification mini-language](https://docs.python.org/3/library/string.html#format-specification-mini-language)
and Rust's
[`std::fmt`](https://doc.rust-lang.org/std/fmt/). It is built on
[numfmt-nv](https://novo-lang.org/packages/numfmt-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

**Substitution** is the smaller half. A template is text with `${name}`
or `{{name}}` in it, a name is looked up in a context the caller
supplies, and its text is put in the hole. There is no other language:
no conditions, no loops, no filters, no includes and no expressions.

**Formatting** is the larger half. A **format specification** is the
part after the colon in `{:<08.3f}`, or the part after the percent sign
in `%-08.3f`. It carries the **fill** character, the **alignment**, how
a **sign** is written, whether the **alternate form** adds a `0x` or a
`0b` prefix, the minimum **width** of the field, the **precision**, a
digit **grouping** separator, and the **kind**, which is the conversion
letter such as `d`, `x`, `f` or `e`.

Three dialects spell that specification three ways and mean the same
thing:

| Dialect | Written | What it says |
| --- | --- | --- |
| C `printf` | `%-08.3f` | Left-aligned, zero-filled, at least 8 columns, 3 decimal places, fixed notation |
| Python `str.format` | `{:<08.3f}` | The same |
| Rust `format!` | `{:<08.3}` | The same |

So there are three parsers, one specification type and one renderer.

**Precision** means two different things, and both are the
specification's. On a number it is how many digits follow the point. On
a string it **truncates**: `%.3s` of `hello` is `hel`.

A **width smaller than the content does not truncate.** Every dialect
grows the field instead.

## Install

```
novo pkg add template-nv
```

## Example

```novo
use tmplval
use tmplsub
use tmplprintf

fn main() [io]
    // The values a template is rendered against, built by hand: the
    // language has no reflection, so there is one line per field.
    let context = tmplval.map_of([
        tmplval.pair("name", TmplStr("Ada")),
        tmplval.pair("total", TmplFloat(12.5))])

    // Substitution only: a name is looked up and its text inserted.
    // There are no conditions, no loops and no filters. The backslash
    // escapes the dollar for novo's own string interpolation.
    match tmplsub.substitute("Hello, \${name}!", context,
                             tmplsub.default_policy())
        Err(e) => println(e.message())
        Ok(s)  => println(s)

    // The printf dialect, over a typed argument list. The count and
    // the types are checked before a byte is rendered.
    match tmplprintf.format("%-20s %10.2f", [TmplStr("Ada"), TmplFloat(12.5)])
        Err(e) => println(e.message())
        Ok(line) => println(line)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: template-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `tmplerr` | Every reason a template or a format could not be used, with the byte offset it is about. |
| `tmplval` | The values a template is rendered against, as a tree of strings, numbers, lists and maps. |
| `tmplspec` | The one format specification, the padding arithmetic, and the conversion between dialect spellings. |
| `tmplsub` | `${name}` and `{{name}}` substitution, and the policy that chooses between them. |
| `tmplprintf` | C's dialect: the parser, the renderer, and the allocation-free integer and string field arithmetic. |
| `tmplpy` | Python's `str.format`: the field syntax, the paths and the conversions. |
| `tmplrust` | Rust's `format!`, and the translation of a format string between the three dialects. |

## How to choose an entry point

**`tmplsub.substitute` is one call for a message with names in it.**
`tmplsub.parse` then `render` is the same when the template is used
more than once.

**`tmplprintf.format`, `tmplpy.format` and `tmplrust.format` are one
call each in their own dialect.** Each also has a `parse` and a
`render`, for a format used repeatedly, a `render_into` that appends to
a buffer you own, and a `render_len` that answers the length first.

**`tmplrust.translate` converts a format string between dialects.** It
refuses rather than approximating. `is_translatable` asks first.

**`tmplspec.parse_spec` and `format_spec` read and write one
specification in a named dialect.** Use them when the caller stores a
specification rather than a whole format string.

**`tmplprintf.int_len` and `int_byte` are the device path.** They
answer how many bytes a formatted integer occupies in its field, and
what byte `i` is, so firmware writes them straight to a peripheral. See
"Running on a microcontroller".

**`tmplprintf.is_safe_external` checks a format string that came from
outside** before it is even parsed.

## The rules a user needs

1. **Arguments are a typed list and the count is checked before
   anything is rendered.** `TmplTooFewArguments` carries both numbers.
   C's `printf("%d %d", 1)` reads whatever was next on the stack; that
   cannot happen here.
2. **A conversion applied to the wrong kind of value is refused.**
   `%d` against a string is `TmplWrongValueKind` rather than a garbage
   number.
3. **`%n` is refused by name.** It is the conversion that writes the
   byte count back through a pointer, and it is why format-string bugs
   are write primitives rather than only read ones.
   `TmplWriteBackRefused` is what a format string carrying one meets, a
   refusal rather than a silent absence.
4. **A boolean is not a number.** C prints one as `1`. A template that
   did the same would render a checkbox as a digit.
5. **The C length modifiers are parsed and ignored.** `%lld`, `%zu` and
   `%Lf` occur in real format strings, and this language has one
   integer width and one float width. `has_length_modifier` finds the
   conversions that were relying on a width that is not here.
6. **`TmplSpec` is the union of what the three dialects can express.**
   A specification can therefore hold a combination its own dialect
   cannot spell. `tmplspec.check_for` answers which, every parser calls
   it, and `tmplrust.translate` refuses rather than approximating. A
   grouping separator has no Rust spelling, and a template that
   silently lost it would render a different document.
7. **A width smaller than the content grows the field.** It never
   truncates. That is the rule that stops a log line losing the end of
   a number.
8. **Precision truncates a string.** `%.3s` of `hello` is `hel`.
9. **Centring puts the odd pad byte on the right.** That is Python's
   rule and Rust's, and a caller that split it the other way would
   centre a column one byte differently from every other
   implementation.
10. **Substitution does not rescan its own output.** A value containing
    `${x}` is inserted verbatim and is not looked up again, so a
    user-supplied value cannot reach into the context.
11. **A missing name is refused by default.** `TmplMissingRefuse` is
    the default policy, because a template with a typo in a name
    otherwise renders a hole nobody sees.
    `tmplsub.missing_names` lists them for a caller that would rather
    check first.
12. **There is no automatic escaping of values.** `TmplEscape` is about
    the template's own syntax, which is how to write a literal `$`, and
    not about where the output is going. A caller rendering into HTML
    escapes the value itself with
    [html-nv](https://novo-lang.org/packages/html-nv).
13. **Python's unquoted subscript is a string key unless every
    character is a digit.** `{d[1]}` is index 1 and `{d[one]}` is the
    key `one`. The mini-language has no quoting at all.
14. **`tmplval` trees are built by hand, one line per field.** The
    language has no reflection. `tmplval.map_of`, `pair`, `list_of` and
    `with` are the constructors.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers the field arithmetic in `tmplspec` and the
integer and string field functions in `tmplprintf`. Those are the
functions marked `@tier(embedded)`, and nothing else in the package is
claimed.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release and produced an executable.

The device shape is a loop the firmware owns. The package answers how
many bytes the field is and what byte `i` of it is, and the firmware
writes each one:

```novo ignore
for i in 0 .. tmplprintf.int_len(v, 10, 6, 0, false)
    hal.uart.write_byte(tmplprintf.int_byte(i, v, 10, 6, 1, 0, false, 32))
```

This is what "render into the caller's buffer" means on a device.
`bytes.set_u8` is not available at the embedded tier, so `render_into`
is a host call whatever its buffer belongs to.
[numfmt-nv](https://novo-lang.org/packages/numfmt-nv)'s `digits` module
draws the same line for a bare number, and this package follows it, so
a reader who knows one knows the other.

`tmplsub`, `tmplpy` and `tmplrust` are outside the claim. All three
build lists of pieces and return a `Str`, and a device that needs them
needs a heap.

**What the probe proves today is narrower than what it will prove.**
Every body under `src/` is a `todo()`, so what links is the signatures
and the types. There is no padding arithmetic in the binary yet, so the
probe does not show the renderer is allocation-free. It shows that
nothing in the shape of the surface needs an allocator or a host.

The device functions take an `Int` where the host ones take an enum,
because that is the spelling a value-tier caller can use. Alignment is
0 for left, 1 for right, 2 for centre and 3 for after-sign, and sign is
0 for minus only, 1 for plus and 2 for space.
`tmplspec.effective_align` converts between the two on the host side.

## What is not included

- **Control flow.** No conditions, no loops, no filters, no includes
  and no expressions. The moment a template needs one of those it needs
  [tera-nv](https://novo-lang.org/packages/tera-nv).
- **Automatic escaping.** See rule 12.
- **Reflection.** See rule 14.
- **Printing.** Every renderer returns text or takes the caller's
  buffer. A library that printed would be deciding for the program that
  embeds it that something appears on a console.
- **Reading a template from a file.** The source arrives as a string.
- **`%n`.** See rule 3.
- **Integer and float widths beyond the language's own.** See rule 5.
- **Locale-aware number formatting.** A grouping separator is a
  character in the specification, not a locale lookup.
  [i18n-nv](https://novo-lang.org/packages/i18n-nv) is the package for
  localised numbers and dates.

## Related packages

- [numfmt-nv](https://novo-lang.org/packages/numfmt-nv) is the only
  dependency. `%d` is an integer-to-decimal conversion and `%.3f` is a
  shortest-round-trip float conversion, and neither is small. Its
  `digits.digit_count` and `digits.digit_byte` are the allocation-free
  shape this package's device half is built on, so the dependency is
  what lets those functions build for a microcontroller with no heap
  allocator.
- [tera-nv](https://novo-lang.org/packages/tera-nv) is the full
  template engine, with conditions, loops, filters and inheritance, and
  it escapes its output by default because its output is usually a web
  page. Use it when a template needs a condition. Use this one when it
  does not: it is cheaper to read, cheaper to audit, and cannot recurse.
- [html-nv](https://novo-lang.org/packages/html-nv) escapes a value for
  an HTML document. A caller applies it to the value before
  substitution.
- [i18n-nv](https://novo-lang.org/packages/i18n-nv) is message
  catalogues with grammar in them, for text a person reads in their own
  language.

## Tests

The references are glibc's `printf(3)` for the C dialect, CPython's
`str.format` for the second, Rust's `std::fmt` for the third, and
Python's `string.Template` for the substitution half. The expected
strings are what those produce.

The cases chosen are the ones where implementations differ by one byte:
a negative number under the zero flag, a precision applied to a string,
an alternate form applied to zero, a width smaller than the content,
and a centred field with an odd number of pad bytes.

```bash
novo test tests/tmplspec_tests.nv     # 7 tests: the one spec and the padding rules
novo test tests/tmplsub_tests.nv      # 7 tests: substitution and the missing-name policy
novo test tests/tmplprintf_tests.nv   # 8 tests: the C dialect and its refusals
novo test tests/tmplfield_tests.nv    # 9 tests: the Python and Rust field syntax
```

The suite asserts that the three dialects parse to the same
specification, that a translation with no spelling in the target
dialect is refused, that too few arguments is caught before any output,
that `%d` against a string is refused, that `%n` is refused by name,
that a width smaller than the content grows the field, that a precision
truncates a string, that a centred field puts the odd byte on the
right, and that a substituted value containing template syntax is not
rescanned.

`tests/embedded_probe.nv` is not a test suite. It is the program that
shows the `@tier(embedded)` functions build for a microcontroller with
no heap allocator, built for a Cortex-M4.

The tests compile today and fail at run, each on the
`not implemented: template-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

Nothing is implemented. The table lists the surface an implementation
has to fill.

| Item | Implemented |
| --- | --- |
| `tmplerr.offset_of`, `.is_format_fault`, `.underline`, `TmplError.message` | no |
| `tmplval.walk`, `.get`, `.at`, `.len`, `.keys` | no |
| `tmplval.kind_name`, `.is_numeric`, `.as_text`, `.as_debug`, `.as_int`, `.as_float` | no |
| `tmplval.pair`, `.map_of`, `.list_of`, `.with` | no |
| `tmplspec.default_spec`, `.parse_spec`, `.format_spec`, `.check_for`, `.resolve` | no |
| `tmplspec.count_arity`, `.width_of`, `.precision_of`, `.effective_align` | no |
| `tmplspec.is_numeric`, `.radix_of`, `.alternate_prefix` | no |
| `tmplspec.field_len`, `.pad_before`, `.pad_after`, `.field_byte`, `.content_index` | no |
| `tmplspec.sign_byte`, `.fill_len` | no |
| `tmplsub.default_policy`, `.brace_policy`, `.is_valid_name`, `.escape_literal` | no |
| `tmplsub.parse`, `.render`, `.render_into`, `.render_len`, `.substitute` | no |
| `tmplsub.names`, `.name_count`, `.missing_names` | no |
| `tmplprintf.parse`, `.render`, `.render_into`, `.render_len`, `.format` | no |
| `tmplprintf.arity_of`, `.argument_kinds`, `.is_safe_external`, `.has_length_modifier` | no |
| `tmplprintf.int_len`, `.int_byte`, `.str_field_len`, `.str_field_index` | no |
| `tmplpy.parse`, `.render`, `.render_into`, `.render_len`, `.format` | no |
| `tmplpy.parse_field`, `.parse_path`, `.format_field`, `.convert`, `.escape_literal` | no |
| `tmplpy.field_count`, `.names_of`, `.uses_automatic_numbering` | no |
| `tmplrust.parse`, `.render`, `.render_into`, `.render_len`, `.format` | no |
| `tmplrust.field_count`, `.names_of`, `.escape_literal`, `.specs_of` | no |
| `tmplrust.translate`, `.is_translatable` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
