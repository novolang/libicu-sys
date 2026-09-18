# libicu-sys

The International Components for Unicode, ICU, is the reference
implementation of the Unicode standard and its algorithms. Its C API
is documented in the
[ICU4C user guide](https://unicode-org.github.io/icu/userguide/), and
its common library, libicuuc, holds the character properties, the
normalizers, the text boundary iterators and the charset converters.
This package declares fifty-eight of that library's entry points to
novo-lang, one declaration each.

Every function here is a declaration of a function in libicuuc. The
package contains no logic of its own, and it does nothing without the C
library installed. The fifty-eight entry points are the ones a program
needs to read the Unicode character database, normalize text, find its
word and sentence boundaries and convert it between charsets. The
section "What is not included" says what a program still cannot do with
them alone.

**This release binds ICU 74 and no other major version.** ICU renames
every exported C function to `<name>_<major>`, so the library exports
`u_strlen_74` and exports nothing called `u_strlen`. The suffix is
`U_ICU_VERSION_SUFFIX` in `unicode/uvernum.h`, and it is `_74` in an
ICU 74 build. Every declaration here names a `_74` symbol. Installing a
different major version of ICU leaves the package resolving against
nothing.

## What it is

A **code point** is a number the Unicode standard gives to a character,
written U+0041 for the capital letter A. There are rather more than a
million of them, and about a seventh are assigned.

The **Unicode character database** records what each code point is. The
**general category** says whether it is an uppercase letter, a decimal
digit, a space separator or something else. A **binary property** is a
yes-or-no fact about it, such as Alphabetic or White_Space. A **script**
is the writing system it belongs to. ISO 15924 names each one with
four letters: Latn, Cyrl, Arab, Hani. ICU numbers the scripts itself,
and `uscript_getShortName` is the call that answers the four letters.

**UTF-16** is the encoding ICU works in. A code point below U+10000 is
one sixteen-bit unit, and one above it is a **surrogate pair** of two
units. ICU calls the unit a `UChar`, and every length and capacity in
this package counts units and not bytes.

**Normalization** decides whether two strings that look the same are
the same. The letter é can be written as one code point or as an e
followed by a combining acute accent. **Form C** composes: it prefers
the single code point. **Form D** decomposes: it prefers the letter and
the mark. The **compatibility forms**, KC and KD, go further and
replace a character with the sequence it is a formatting variant of, so
the ligature ﬁ becomes two letters. That mapping loses a distinction
and cannot be undone. Unicode Standard Annex 15 specifies all four.

A **break iterator** finds the places in a text where it may be cut:
between characters, between words, between sentences, or where a line
may be wrapped. The rules differ by language, so an iterator is opened
for a locale. Unicode Standard Annex 29 specifies the default rules.

A **converter** turns bytes in a charset into UTF-16 and back. ICU
carries the tables for several hundred charsets, and finds one by its
name or by any of its registered aliases.

A **status code** is how every fallible call in ICU reports. The caller
passes the address of a `UErrorCode`, a four-byte integer that starts
at zero, and the call writes a code into it. Zero is success, a
positive code is a failure and a negative code is a warning.

## Install

```
novo pkg add libicu-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package `libicu-dev`:

```
sudo apt install libicu-dev
```

On macOS the Homebrew formula is `icu4c`. On other systems the library
builds from the ICU4C source.

The version this package binds is ICU 74. On Debian and Ubuntu that is
the `libicu74` package.

## Example

A word counted, normalized and read back:

```novo ignore
use libicu

fn main() [io, ffi]
    // A status slot: four bytes ICU writes into, of the eight
    // `ptr.alloc_word` reserves and zeroes.
    let st = ptr.alloc_word()

    // "cafe" and a combining acute accent, laid out as UTF-16.  Each
    // write covers the high half the one before it left zero.
    let src = ptr.alloc(16)
    ptr.write_i32(src, 0x0063)
    ptr.write_i32(src + 2, 0x0061)
    ptr.write_i32(src + 4, 0x0066)
    ptr.write_i32(src + 6, 0x0065)
    ptr.write_i32(src + 8, 0x0301)
    println("${libicu.u_strlen(src)} code units")

    // Form C joins the e and the mark into one code point.
    let nfc = libicu.unorm2_get_nfc_instance(st)
    let dest = ptr.alloc(32)
    let n = libicu.unorm2_normalize(nfc, src, 5, dest, 16, st)
    if ptr.read_word(st) != 0
        println(ptr.read_str(libicu.u_error_name(ptr.read_word(st))))
        return
    println("${n} code units after composing")

    // And back out as UTF-8.
    ptr.write_word(st, 0)
    let out = ptr.alloc(32)
    let _ = libicu.u_str_to_utf8(out, 32, 0, dest, n, st)
    println(ptr.read_str(out))

    ptr.free(out)
    ptr.free(dest)
    ptr.free(src)
    ptr.free(st)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libicu_tests.nv`, where the round trip is asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libicu` | Every entry point, in seven groups: versions and status codes, UTF-16 strings, the character properties, the scripts, normalization, break iteration and the charset converters. |

The seven groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Versions and status codes | 5 | Reports the library version and the Unicode version, and names a status code. |
| UTF-16 strings | 8 | Converts to and from UTF-8, measures, compares and changes the case of a string. |
| Character properties | 14 | Answers what a code point is, what it maps to and what it is named. |
| Scripts | 3 | Answers which writing system a code point belongs to, and names it. |
| Normalization | 10 | Composes and decomposes, in the canonical and the compatibility forms. |
| Break iteration | 10 | Walks the character, word, line and sentence boundaries of a text. |
| Charset converters | 8 | Converts bytes between a charset and UTF-16, in one call or with a converter the caller keeps. |

## How to choose an entry point

`u_charType` and `u_hasBinaryProperty` are for one code point at a
time. They read the character database and allocate nothing.

`u_toupper` and `u_tolower` are the **simple** mappings, one code point
to one code point. `u_strToUpper` and `u_strToLower` are the **full**
mappings over a whole string, and they are the correct ones: the German
sharp s upper-cases to two letters, and only the string form can say
so. The string form also takes a locale, which matters for Turkish and
Azerbaijani.

`u_strFoldCase` is for comparing. Case folding maps every casing of the
same text to one form, belongs to no locale, and is not a substitute
for lower-casing text a reader will see.

`ucnv_convert` converts in one call and opens no converter. `ucnv_open`
with `ucnv_toUChars` and `ucnv_fromUChars` is for the second and every
later buffer in the same charset.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **Every symbol carries the ICU major version.** `symbol =` on each
   declaration ends in `_74`, because ICU renames its exported
   functions and the library holds no symbol without the suffix. The
   suffix is `U_ICU_VERSION_SUFFIX` in `unicode/uvernum.h`. A program
   built against this package needs ICU 74 installed.
3. **A status slot is four bytes the caller owns.** `ptr.alloc_word`
   reserves eight and zeroes them, which is `U_ZERO_ERROR`, and
   `ptr.read_word` reads the code back. Set the slot to zero before
   each call. A call handed a code that is already a failure returns at
   once and does nothing.
4. **A status code is a number with a sign.** Zero is success, a
   positive code is a failure, and a negative code is a warning the
   caller may ignore. `U_BUFFER_OVERFLOW_ERROR`, the code a destination
   that is too small gives, is 15.
5. **A length counts UTF-16 code units, never bytes.** A buffer of
   `n` units is `n * 2` bytes of `ptr.alloc`. A length of -1 says the
   string is null-terminated.
6. **A call that fills a buffer answers the length it needed**, whether
   or not it fitted. When it did not fit the status holds
   `U_BUFFER_OVERFLOW_ERROR` and the answer is the size to allocate.
7. **An entry point that answers a C `int32_t` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number.
   `u_strCompare`, `u_digit`, `ubrk_next` and `ubrk_previous` are the
   ones that need it.
8. **An entry point that answers one byte answers it in one byte, and
   the bits above it are not cleared.** A `UBool` is `int8_t`, so write
   `as i8` before comparing one with 1. `u_isalpha`,
   `u_hasBinaryProperty`, `unorm2_isNormalized` and
   `unorm2_hasBoundaryBefore` each answer one.
   `unorm2_getCombiningClass` answers an unsigned byte instead, where
   `as i8` would read 230 as -26. Write `% 256` for that one.
9. **A locale is a `Str` and can never be the null pointer.** ICU reads
   a null locale as the process default. Pass `""` for the root locale,
   which is the nearest thing this package can express.
10. **A handle this package opens is a handle the caller closes.**
    `ubrk_open` is released by `ubrk_close` and `ucnv_open` by
    `ucnv_close`. Neither is released when the program ends.
11. **A `const char *` answer is memory the library owns.**
    `u_errorName`, `uscript_getName`, `uscript_getShortName`,
    `ucnv_getAvailableName` and `ucnv_getName` each answer such an
    address. Read it with `ptr.read_str` and free nothing. The name a
    converter answers lives as long as that converter.
12. **The five normalizer instances are singletons the library owns.**
    `unorm2_close` on one of them frees an object the library keeps for
    the life of the process, and the next use of it faults. Do not
    close them.
13. **A break iterator keeps no copy of its text.** The UTF-16 string
    passed to `ubrk_open` or `ubrk_setText` must stay where it is for
    as long as the iterator is used.
14. **`ubrk_next` and `ubrk_previous` answer -1 when there is no
    further boundary.** The value is `UBRK_DONE`, and it needs
    `as i32`.
15. **A word break tells you what it ended.** `ubrk_getRuleStatus`
    answers 0 for a break that ends no word, 100 for a number, 200 for
    a word of letters and 300 for one of kana. This is how a program
    drops the spaces out of a word walk.
16. **The property numbers are numbers**, because the C header spells
    them as enumerations.

    | Constant | Number | What it is |
    | --- | --- | --- |
    | `UCHAR_ALPHABETIC` | 0 | the binary Alphabetic property |
    | `UCHAR_IDEOGRAPHIC` | 17 | the binary Ideographic property |
    | `UCHAR_LOWERCASE` | 22 | the binary Lowercase property |
    | `UCHAR_UPPERCASE` | 30 | the binary Uppercase property |
    | `UCHAR_WHITE_SPACE` | 31 | the binary White_Space property |
    | `UCHAR_BIDI_CLASS` | 0x1000 | the enumerated Bidi_Class property |
    | `UCHAR_GENERAL_CATEGORY` | 0x1005 | the enumerated General_Category property |
    | `UCHAR_SCRIPT` | 0x100A | the enumerated Script property |

17. **The general categories are numbers too.** 1 is an uppercase
    letter, 2 a lowercase letter, 6 a non-spacing mark, 9 a decimal
    digit, 12 a space separator and 0 unassigned. Unicode Standard
    Annex 44 table 12 names them all.
18. **A break iterator type is 0, 1, 2 or 3**: characters, words,
    lines, sentences.
19. **`u_getNumericValue` answers -123456789.0** for a code point that
    stands for no number. The constant is `U_NO_NUMERIC_VALUE`.

## What is not included

- **The internationalisation library.** ICU ships as five shared
  libraries, and a `sys` package wraps one. `libicui18n` holds the
  collation (`ucol_`), the number and date formatters (`unum_`,
  `udat_`), the calendars (`ucal_`), the transliterators (`utrans_`),
  the message formatter (`umsg_`) and the regular expressions
  (`uregex_`). None of them are here. A program that has to sort or
  format text needs a second package.
- **The C++ API.** `icu::UnicodeString`, `icu::Normalizer2`,
  `icu::Collator` and the rest are C++ classes whose symbols are
  name-mangled and whose methods take and return objects by value.
- **The converter callbacks.** `ucnv_setToUCallBack` and
  `ucnv_setFromUCallBack` take C function pointers, and a novo-lang
  function is not one. A converter keeps its default behaviour for a
  character the charset cannot represent.
- **`UText`.** The `utext_*` family is driven by a table of C function
  pointers the caller supplies.
- **The default locale and the default converter.** A null
  `const char *` selects the process default in C, and a novo-lang
  `Str` cannot be the null pointer.
- **The sets.** `uset_open` and the `USet` family are in `libicuuc`
  and are not declared here. `unorm2_openFiltered` takes a `USet`, so
  the filtered normalizers are absent with them.
- **The data loading controls.** `u_setDataDirectory`,
  `udata_setCommonData` and `u_cleanup` are left out. This release
  reads the data the library finds for itself.

## Related packages

`unicode-nv` is the Unicode character database and its algorithms
written in novo-lang, with no C library. It is the package to reach for
first: it builds for a microcontroller and for WebAssembly, where this
one does not, and it carries the tables a program usually needs.
`unicode-nv` is planned and not published yet.

Choose this package when the program needs ICU's own tables, its
locale-sensitive break rules, or its several hundred charset
converters. The rest of ICU has no novo-lang port planned at all, which
means the collation, the formatters and the calendars.

## Tests

`tests/libicu_tests.nv` holds eleven tests over the fifty-eight entry
points. They call the C library, so `novo test` needs ICU 74 installed
and linkable:

```
novo test tests/libicu_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

Every test works in memory over text the suite lays out itself, so the
suite reads and writes nothing and needs no privileges. The suite
asserts that the library reports version 74 and Unicode 15.1, that
UTF-8 converts to UTF-16 and back byte for byte, that a string
upper-cases and case-folds to the expected text, that A is an uppercase
Latin letter with the Alphabetic and Uppercase properties, that
U+0041's Unicode name round-trips through `u_charName` and
`u_charFromName`, that the decomposed form of café composes to four
code units and decomposes back to five, that the ligature ﬁ is one code
point in Form C and two in Form KC, that a word iterator finds the
boundaries of a three-word sentence with the right rule statuses, and
that a Latin-1 converter turns three bytes into three code units and
back.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

ICU itself is distributed under the Unicode licence, and installing it
is the reader's own step.
