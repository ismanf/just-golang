# `unicode`, `unicode/utf8`, `unicode/utf16` — Text at the Code-Point Level

## TL;DR

Go strings are sequences of bytes that happen to be UTF-8 by convention. `unicode` exposes Unicode properties (`IsLetter`, `IsDigit`, `IsSpace`, scripts, categories). `unicode/utf8` decodes/encodes runes from/to bytes. `unicode/utf16` is for the legacy UTF-16 world (Windows APIs, JavaScript). Iterate strings with `range` to get runes; iterate bytes with index loops only when you know the data is ASCII.

## Mental Model

```
"héllo"                                        // UTF-8 bytes
└─ len() = 6 (h=1, é=2, l=1, l=1, o=1)        // byte length, not rune count
└─ utf8.RuneCountInString = 5                  // actual rune count
└─ range over string → (i, r) where i is BYTE index, r is rune

range "héllo":  (0,'h') (1,'é') (3,'l') (4,'l') (5,'o')
                                  ^ skipped 2 because 'é' was 2 bytes
```

UTF-8 self-synchronizes: from any byte you can resync to the next rune boundary by scanning forward until you see a byte with high bits `10xxxxxx` (continuation) ending.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"unicode"
	"unicode/utf8"
)

func main() {
	s := "héllo, 世界"
	fmt.Println(len(s))                      // 13 bytes
	fmt.Println(utf8.RuneCountInString(s))   // 9 runes

	for i, r := range s {
		fmt.Printf("%d %c (%d)\n", i, r, r)
	}

	fmt.Println(unicode.IsLetter('A'))  // true
	fmt.Println(unicode.IsDigit('7'))   // true
	fmt.Println(unicode.IsSpace(' '))   // true
	// Output:
	// 13
	// 9
	// 0 h (104)
	// 1 é (233)
	// 3 l (108)
	// 4 l (108)
	// 5 o (111)
	// 6 , (44)
	// 7   (32)
	// 8 世 (19990)
	// 11 界 (30028)
}
```

## Deep Dive

### Runes and bytes

A `rune` is `int32`. UTF-8 encodes a rune in 1-4 bytes:

| Range          | Bytes | First byte starts with |
|----------------|-------|------------------------|
| U+0000–U+007F  | 1     | `0xxxxxxx` (ASCII)     |
| U+0080–U+07FF  | 2     | `110xxxxx`             |
| U+0800–U+FFFF  | 3     | `1110xxxx`             |
| U+10000–U+10FFFF | 4   | `11110xxx`             |

Continuation bytes start with `10xxxxxx`.

### `utf8` API

- `utf8.RuneLen(r) int` — bytes needed.
- `utf8.EncodeRune(buf []byte, r rune) int` — write rune to buf.
- `utf8.DecodeRune(buf []byte) (r rune, size int)` — read rune from start of buf.
- `utf8.DecodeLastRune(buf)` — read last rune.
- `utf8.DecodeRuneInString(s)` — string version.
- `utf8.RuneCount` / `RuneCountInString`.
- `utf8.Valid` / `ValidString` — check well-formedness.
- `utf8.RuneError` (= `'�'`) — returned for invalid bytes.
- `utf8.UTFMax` (= 4) — max bytes per rune.

### Iterating: `range` vs indexed

```go
// Correct rune iteration:
for i, r := range s {
	// i is BYTE index of start of rune; r is the rune
}

// Wrong (yields bytes, broken on multi-byte runes):
for i := 0; i < len(s); i++ {
	r := rune(s[i]) // WRONG for non-ASCII
}

// Manual rune iteration:
for i := 0; i < len(s); {
	r, size := utf8.DecodeRuneInString(s[i:])
	// process r
	i += size
}
```

### Slicing strings

`s[i:j]` is byte-based. Slicing across a rune boundary yields invalid UTF-8.

```go
s := "héllo"
fmt.Println(s[:2])  // "h\xc3" — broken
```

To slice by runes: convert to `[]rune` first (allocates) or compute byte offsets with `utf8.DecodeRuneInString`.

### `unicode` predicates

- Categorical: `IsLetter`, `IsDigit`, `IsNumber`, `IsLower`, `IsUpper`, `IsSpace`, `IsPunct`, `IsControl`, `IsPrint`, `IsGraphic`.
- Script-specific: `IsOneOf(unicode.Han, r)`, `Is(unicode.Latin, r)`.
- Case mapping: `ToLower(r)`, `ToUpper(r)`, `ToTitle(r)`, `SimpleFold(r)`.

### `unicode.RangeTable` and custom sets

```go
hexDigit := &unicode.RangeTable{
	R16: []unicode.Range16{{'0','9',1}, {'A','F',1}, {'a','f',1}},
}
if unicode.Is(hexDigit, r) { ... }
```

Used by `strings.Trim` and `strings.Map` for char-set predicates.

### `unicode/utf16`

For Windows API interop and JS-style strings:

```go
runes := utf16.Decode([]uint16{0xd83d, 0xde00}) // surrogate pair → 😀
encoded := utf16.Encode(runes)
```

Surrogate pairs encode runes above U+FFFF as two 16-bit code units.

### Normalization

The stdlib does NOT handle normalization (NFC/NFD/NFKC/NFKD). Use `golang.org/x/text/unicode/norm`:

```go
import "golang.org/x/text/unicode/norm"
nfc := norm.NFC.String("é")  // single code point
nfd := norm.NFD.String("é")  // e + combining acute
```

Critical for comparing user input where the same visible character may have two byte sequences.

### Grapheme clusters

A "user-perceived character" (grapheme) can span multiple runes (emoji ZWJ sequences, combining marks). The stdlib doesn't handle this; use `golang.org/x/text/unicode/segment` or `rivo/uniseg`. `len("👨‍👩‍👧‍👦") == 25` (UTF-8 bytes); `utf8.RuneCountInString` gives 7; grapheme count is 1.

## Standard Library Hooks

- `strings`, `bytes` — use `unicode/utf8` internally for rune-aware ops (`ToLower`, `Fields`, `EqualFold`).
- `regexp` — `RE2` syntax with Unicode support; `\p{L}` for letters etc.
- `golang.org/x/text` — normalization, collation, transformation, segmentation (not stdlib but the canonical extension).

## Real-World Patterns

### 1. Rune-aware truncation

```go
import "unicode/utf8"

func truncateRunes(s string, n int) string {
	if utf8.RuneCountInString(s) <= n {
		return s
	}
	out := make([]byte, 0, len(s))
	i := 0
	for j, r := range s {
		if i >= n { _ = j; break }
		out = utf8.AppendRune(out, r)
		i++
	}
	return string(out)
}
```

Use case: tweet-style char limits, table column truncation.

### 2. Validate UTF-8 from untrusted input

```go
if !utf8.ValidString(input) {
	return errors.New("input not valid UTF-8")
}
```

Use case: HTTP body sanitization, log ingestion.

### 3. Case-insensitive comparison without allocating

```go
strings.EqualFold(a, b) // internally rune-walks both strings with SimpleFold
```

Avoid `strings.ToLower(a) == strings.ToLower(b)` which allocates twice.

### 4. Replace control characters

```go
import (
	"strings"
	"unicode"
)

func sanitize(s string) string {
	return strings.Map(func(r rune) rune {
		if unicode.IsControl(r) && r != '\n' && r != '\t' {
			return -1 // drop
		}
		return r
	}, s)
}
```

Use case: terminal output sanitization, log scrubbing.

### 5. Decode mixed UTF-8/legacy input

```go
for i := 0; i < len(buf); {
	r, size := utf8.DecodeRune(buf[i:])
	if r == utf8.RuneError && size == 1 {
		// Invalid byte; skip or replace
		i++
		continue
	}
	process(r)
	i += size
}
```

Use case: parsing log files with mixed encodings.

## Anti-Patterns & Gotchas

**Indexing strings by byte for character ops.** `s[2]` is a byte, not a rune.

**Comparing strings with different normalization forms.** "café" can be NFC or NFD; `==` fails.

**`len(s)` to count "characters".** It's bytes. Even runes may not equal user-perceived characters (graphemes).

**Slicing across rune boundaries.** Produces invalid UTF-8.

**`strings.ToUpper("İ")` expectations.** Turkish dotless-I handling differs from Latin; stdlib does basic case mapping, not locale-aware.

**Treating `[]rune(s)` as cheap.** It allocates and copies. Only do it when needed.

**Mixing `byte` and `rune` arithmetic.** `'A' + 1 == 'B'` only for ASCII; not for arbitrary runes.

**Using `unicode/utf16` when not interfacing with UTF-16 systems.** Almost everything in Go and on the network is UTF-8.

**Forgetting `golang.org/x/text/unicode/norm` for user identity comparison.** Usernames, search terms, dedup.

## Performance Notes

- `range` over a string is the fastest rune iteration (compiler-inlined `utf8.DecodeRune`).
- `utf8.RuneCountInString` is O(N) bytes; for ASCII data, equals `len(s)`.
- `[]rune(s)` allocates `len(s)*4` bytes worst case (one int32 per rune).
- `utf8.ValidString` is SIMD-accelerated on x86 (since 1.16+).
- Case mapping via `unicode.ToLower`/`ToUpper` is a table lookup; cheap per rune.

## How Big Companies Use It

- **Go compiler itself** uses `unicode/utf8` to parse Go source (identifiers can include Unicode letters).
- **Hugo** uses `golang.org/x/text/unicode/norm` for slug normalization.
- **Docker** uses `utf8.Valid` to sanitize container log streams.
- **Cockroach** normalizes SQL identifiers via `golang.org/x/text/unicode/norm`.

## Source Code References

Pinned to `go1.26`.

- `unicode/utf8`: [`src/unicode/utf8/utf8.go`](https://github.com/golang/go/blob/master/src/unicode/utf8/utf8.go).
- `unicode` properties tables (generated): [`src/unicode/tables.go`](https://github.com/golang/go/blob/master/src/unicode/tables.go).
- `unicode.SimpleFold`: [`src/unicode/letter.go`](https://github.com/golang/go/blob/master/src/unicode/letter.go).
- `unicode/utf16`: [`src/unicode/utf16/utf16.go`](https://github.com/golang/go/blob/master/src/unicode/utf16/utf16.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/unicode, /unicode/utf8, /unicode/utf16.
- Go blog, "Strings, bytes, runes and characters in Go": https://go.dev/blog/strings.
- "UTF-8: Bits, Bytes, and Benefits" (Rob Pike, 2003): https://research.swtch.com/utf8.
- Unicode Standard Annex #29 (grapheme cluster boundaries): https://unicode.org/reports/tr29/.

## Exercises / Self-Check

1. Write `truncateRunes(s, n)` (above) and test with a string containing emoji.
2. Why does `len("世界") == 6`? Decode the bytes manually.
3. Compare two strings that differ only in NFC vs NFD normalization. Show that `==` says false and `norm.NFC.String(a) == norm.NFC.String(b)` says true.
4. Replace all control characters from a string except `\n` and `\t`.
5. Implement a function that splits a string at the first invalid UTF-8 byte. Use `utf8.DecodeRune` in a loop.
