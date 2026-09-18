# Changelog

All notable changes to libicu-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: fifty-eight entry points of the libicuuc C API, one
`@ffi` declaration each, and no logic.

### Added

- `libicu` — the whole common-library surface, in seven groups.
  - Versions and status codes: `u_init`, `u_getVersion`,
    `u_getUnicodeVersion`, `u_versionToString` and `u_errorName`.
  - UTF-16 strings and their UTF-8 conversion: `u_strlen`,
    `u_countChar32`, `u_strFromUTF8`, `u_strToUTF8`, `u_strCompare`,
    `u_strToUpper`, `u_strToLower` and `u_strFoldCase`.
  - The character properties: `u_charType`, `u_hasBinaryProperty`,
    `u_getIntPropertyValue`, `u_charDirection`, `u_isalpha`,
    `u_isdigit`, `u_isspace`, `u_toupper`, `u_tolower`, `u_totitle`,
    `u_charName`, `u_charFromName`, `u_getNumericValue` and `u_digit`.
  - The scripts: `uscript_getScript`, `uscript_getName` and
    `uscript_getShortName`.
  - Normalization: the five normalizer singletons,
    `unorm2_normalize`, `unorm2_isNormalized`,
    `unorm2_getCombiningClass`, `unorm2_hasBoundaryBefore` and
    `unorm2_close`.
  - Break iteration: `ubrk_open`, `ubrk_close`, `ubrk_setText`, the
    five movement calls, `ubrk_getRuleStatus` and
    `ubrk_countAvailable`.
  - The charset converters: `ucnv_open`, `ucnv_close`,
    `ucnv_convert`, `ucnv_toUChars`, `ucnv_fromUChars`,
    `ucnv_getName`, `ucnv_countAvailable` and
    `ucnv_getAvailableName`.
- `tests/libicu_tests.nv` — eleven tests over the fifty-eight entry
  points. They call the C library, so they need ICU 74 installed.
  Every test works in memory over text the suite lays out itself, so
  the suite reads and writes nothing and needs no privileges.

### Every symbol carries the ICU major version

ICU renames each exported C function to `<name>_<major>`. The suffix is
`U_ICU_VERSION_SUFFIX` in `unicode/uvernum.h`. The library exports
`u_strlen_74` and exports nothing called `u_strlen`, so a `symbol =`
that leaves the suffix off resolves against nothing. Every
declaration in this release therefore names a `_74` symbol, and this
package is a binding over ICU 74. A different major version of ICU
needs a new release of this package. The alternative, an ICU built
with `--disable-renaming`, is not what Debian, Ubuntu, Homebrew or
Fedora ship.

### Named as missing

**The internationalisation library.** ICU ships as five shared
libraries. `libicuuc` holds the character properties, the
normalizers, the break iterators and the converters; `libicui18n`
holds the collation, the number and date formatters, the calendars,
the transliterators and the regular expressions; `libicudata` holds
the tables and exports no entry point of its own; `libicuio` is a
stdio layer and `libicutu` is the toolchain. A `sys` package wraps
exactly one library, and this one wraps the common library. Every
symbol beginning `ucol_`, `unum_`, `udat_`, `ucal_`, `utrans_`,
`umsg_` and `uregex_` is therefore absent, and a program that has to
sort or format needs a second package, which this release does not
provide.

**The C++ API.** `icu::UnicodeString`, `icu::Normalizer2`,
`icu::BreakIterator` and the rest are C++ classes. Their symbols are
name-mangled and their methods take and return objects by value, and
the novo-lang foreign function interface passes integers, floats and
strings.

**The callback interfaces.** `ucnv_setToUCallBack` and
`ucnv_setFromUCallBack` take C function pointers, and a novo-lang
function is not one. A converter therefore keeps whatever it does by
default with a character the charset cannot represent.

**`UText`.** The `utext_*` family is driven by a table of C function
pointers the caller supplies, which is the whole point of it.

**The default locale and the default converter.** Passing the null
pointer where ICU takes a `const char *locale` or a
`const char *converterName` selects the process default. A novo-lang
`Str` cannot be the null pointer, so the calls here take `""` for the
root locale and a charset name for the converter, and the default is
not reachable.

**The sets.** `uset_open` and the `USet` family are in `libicuuc` and
are not declared. `unorm2_openFiltered` takes a `USet`, so the filtered
normalizers are absent with them.

### Unverified in one respect

The suite passes against the ICU 74 installed on the machine that
built this release. Nothing in it has been run against another major
version, and by construction nothing in it can be.
