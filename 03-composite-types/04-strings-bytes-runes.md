# Strings, Bytes, Runes

## TL;DR

A Go `string` is an **immutable, read-only slice of bytes** with a 2-word header (data pointer + length). It has no encoding *enforced*, but the standard library and source files treat strings as UTF-8. A `byte` is `uint8`; a `rune` is `int32` and represents a single Unicode code point. Indexing returns a byte. Ranging yields runes. The cheapest "string operation" is no operation; the most expensive is unnecessary `string ↔ []byte` conversion. Use `strings.Builder` to assemble large strings.

## Mental Model

```
s := "héllo"   // 6 bytes on disk: 68 c3 a9 6c 6c 6f
                                    h   é     l  l  o

string header (16 bytes on 64-bit):
+---------+-----+
|   ptr   | len |    <- read-only, points into rodata or heap
+---------+-----+
        len(s) == 6 (BYTES, not runes)

Indexing:           s[1] == 0xc3        // byte
Range-over-string:  yields (0,'h'),(1,'é'),(3,'l'),(4,'l'),(5,'o')  // runes + byte offsets
```

Key invariant: **length is in bytes**, indexing yields bytes, ranging yields runes plus the byte offset of the rune's first byte. If you confuse these three you will produce buggy text-handling code.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "héllo"
	fmt.Println(len(s))                          // 6 (bytes)
	fmt.Println(utf8.RuneCountInString(s))       // 5 (runes)
	fmt.Printf("%q %x\n", s[1], s[1])            // '\xc3' c3 — second byte of 'é'

	for i, r := range s {
		fmt.Printf("byte %d -> rune %q (U+%04X)\n", i, r, r)
	}
	// Output:
	// 6
	// 5
	// 'Ã' c3
	// byte 0 -> rune 'h' (U+0068)
	// byte 1 -> rune 'é' (U+00E9)
	// byte 3 -> rune 'l' (U+006C)
	// byte 4 -> rune 'l' (U+006C)
	// byte 5 -> rune 'o' (U+006F)
}
```

Raw string literals use backticks and ignore escape sequences:

```go
path := `C:\Users\me\Documents`  // no \U / \n processing
re := `^\d+$`
```

## Deep Dive

### Why immutable

The compiler can put string literals in read-only memory (rodata), share identical literals across packages, and pass them by value without copying the bytes — only the header. Mutating would break all of that. `[]byte(s)` and `string(b)` copy the bytes by default (with a few compiler-recognized exceptions, see Performance Notes).

### `byte`, `rune`, `int32`, `uint8`

- `byte` is an alias for `uint8`.
- `rune` is an alias for `int32` and stores a Unicode code point (a number in 0..0x10FFFF).
- `'a'` is an untyped rune constant. `byte('a')` is 0x61.

A `string` is a sequence of bytes. UTF-8 says how to encode a run of code points into bytes; Go's source files are UTF-8 by spec.

### Range loop over a string

```go
for i, r := range s { ... }
```

- `i` is the **byte index** where the current rune starts.
- `r` is the rune (a `int32`).
- If the bytes at `s[i:]` are not valid UTF-8, `r` is `utf8.RuneError` (U+FFFD) and the loop advances one byte.

By comparison `for i := 0; i < len(s); i++` walks bytes, not runes.

### Conversions and their costs

```go
s := "hello"
b := []byte(s)   // allocates a new []byte, copies len(s) bytes
s2 := string(b)  // allocates a new string, copies len(b) bytes
r := []rune(s)   // allocates a []int32 of utf8.RuneCountInString(s) entries
r0 := []byte("a")[0] // single byte: still allocates a []byte unless the compiler optimizes
```

The compiler **does** optimize a few specific patterns to avoid the alloc:

- `m[string(b)]` where `m` is a map and `b` is `[]byte` — no alloc.
- `for i, c := range string(b)` — no alloc.
- String comparison `string(b) == "literal"` — no alloc.

Outside of those, plan for the copy.

### `unsafe.String` and `unsafe.SliceData` (since 1.20)

For zero-copy conversion when you know the bytes won't be mutated:

```go
import "unsafe"

func b2s(b []byte) string {
	return unsafe.String(unsafe.SliceData(b), len(b))
}
```

This is the modern, supported replacement for the old `*(*string)(unsafe.Pointer(&b))` trick. **Mutating `b` after this call is undefined behavior** — Go's runtime assumes strings are immutable.

### `strings.Builder`

The right way to build a string in a loop:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var sb strings.Builder
	sb.Grow(64) // optional pre-size hint
	for i := 0; i < 3; i++ {
		fmt.Fprintf(&sb, "item-%d ", i)
	}
	fmt.Println(sb.String())
	// Output: item-0 item-1 item-2 
}
```

Concatenating with `+=` in a loop is O(n²) — each operation allocates a new string. `Builder` is O(n) amortized.

### Empty string vs nil

There is no nil string; `var s string` is `""` and `s == ""` is `true`. The zero value of `string` is a header with `ptr=nil, len=0`. Comparing strings is by value (length first, then `memcmp`).

### Comparison and equality

```go
"abc" == "abc"       // true; first checks len, then memcmp
"abc" < "abd"        // true; lexicographic byte-wise (NOT locale-aware)
strings.EqualFold(a, b) // ASCII case-insensitive
```

Locale-aware collation is in `golang.org/x/text/collate`.

## Standard Library Hooks

- `strings`: `Builder`, `Contains`, `Index`, `Split`, `Join`, `Replace`, `ToLower`/`ToUpper` (ASCII; for Unicode use `cases.Title` from `x/text`), `EqualFold`, `Cut`, `Fields`, `TrimSpace`, `Repeat`, `NewReader`, `Map`.
- `bytes`: mirror of `strings` for `[]byte`. Includes `bytes.Buffer` (predates `strings.Builder` and is read-write).
- `strconv`: `Quote`, `Unquote`, `Itoa`, `Atoi`, `FormatFloat`, etc.
- `unicode`: `IsLetter`, `IsDigit`, `IsSpace`, etc.
- `unicode/utf8`: `RuneCountInString`, `DecodeRuneInString`, `ValidString`, `RuneLen`.
- `fmt`: `%s`, `%q`, `%x`, `%v` verbs.
- `regexp`: RE2-based, no backreferences, linear time. `regexp.MustCompile` for package-level.
- `golang.org/x/text`: full Unicode normalization, collation, transformations.

## Real-World Patterns

### 1. Building a query string without allocations in a loop

```go
package main

import (
	"net/url"
	"strings"
)

func encode(params map[string]string) string {
	var sb strings.Builder
	sb.Grow(64 * len(params))
	first := true
	for k, v := range params {
		if !first {
			sb.WriteByte('&')
		}
		first = false
		sb.WriteString(url.QueryEscape(k))
		sb.WriteByte('=')
		sb.WriteString(url.QueryEscape(v))
	}
	return sb.String()
}
```

### 2. Cheap substring checks on hot paths

```go
// strings.Contains and strings.HasPrefix already do len-checks
// before falling into the inner loop. Don't over-engineer.
if strings.HasPrefix(line, "ERROR ") { handleError(line[6:]) }
```

`line[6:]` is a sub-string — header reslice, no copy.

### 3. UTF-8 validation at the edge

```go
import "unicode/utf8"

func mustUTF8(s string) error {
	if !utf8.ValidString(s) {
		return fmt.Errorf("non-UTF8 input")
	}
	return nil
}
```

Reject early; downstream code can assume validity.

### 4. Trim with a custom rune predicate

```go
trimmed := strings.TrimFunc(input, func(r rune) bool {
	return !unicode.IsLetter(r) && !unicode.IsDigit(r)
})
```

### 5. Hot-path map lookup keyed on bytes (avoiding the alloc)

```go
var cache = map[string][]byte{}

func lookup(key []byte) ([]byte, bool) {
	v, ok := cache[string(key)] // compiler optimizes: no alloc
	return v, ok
}
```

The conversion `string(key)` inside a map index expression is special-cased by the compiler.

### 6. `strings.Cut` for one-shot split

```go
host, port, ok := strings.Cut("example.com:443", ":")
// host="example.com", port="443", ok=true
```

Faster and clearer than `strings.SplitN(s, ":", 2)`.

## Anti-Patterns & Gotchas

**`s[i]` returns a byte, not a rune.** `"é"[0]` is `0xc3`, not `'é'`.

**`len(s)` is bytes, not runes.** Use `utf8.RuneCountInString`.

**Indexing into a UTF-8 string by integer.** `s[0:1]` of `"héllo"` produces `"h"`; `s[1:2]` produces a single byte of a multi-byte rune — invalid UTF-8.

**Concatenating with `+=` in a loop.** Use `strings.Builder` (or `bytes.Buffer` for byte output).

**`string(intValue)` does not stringify the number.** `string(65)` is `"A"`. Use `strconv.Itoa` or `fmt.Sprint`. (Go vet warns about this.)

**Modifying a `[]byte` derived from `unsafe.String`/`unsafe.Slice`.** Undefined behavior. The runtime assumes strings are immutable.

**Calling `strings.ToLower` thinking it's Unicode-correct for all locales.** It's not (e.g., Turkish dotless 'ı'). Use `golang.org/x/text/cases` with a `language.Tag`.

**Using regex for simple prefix/suffix checks.** `strings.HasPrefix` is orders of magnitude faster.

**`bytes.Buffer.String()` returns a copy.** It's not zero-cost. For one-shot reads of buffered bytes, `Bytes()` is alloc-free (but be aware the slice is invalidated by further writes).

## Performance Notes

- String concat with `+` of two strings allocates `len(a)+len(b)` bytes. The compiler combines runs of `+` into one allocation, but a loop does not.
- `strings.Builder` writes into a `[]byte` and exposes `.String()` via `unsafe.String` — no final copy.
- `[]byte(s)` always allocates and copies; the only exception is the compiler-recognized patterns listed above.
- `string(b)` also allocates and copies, with the same exceptions.
- `len(s)` is O(1). `utf8.RuneCountInString` is O(n).
- `strings.Index` uses a SIMD-friendly implementation written in assembly on AMD64/ARM64.
- Strings have a 16-byte header (64-bit). Passing strings to functions is a 2-word copy, cheap.
- The compiler interns identical literals across packages; `"foo" == "foo"` (different occurrences) often hit the same pointer.

## How Big Companies Use It

- **Google's Go team** moved many internal APIs from `[]byte` to `string` once `unsafe.String`/`unsafe.SliceData` (1.20) made zero-copy interop ergonomic.
- **Cloudflare** wrote `golang/go` patches for SIMD `strings.Index` and `bytes.Equal` on AMD64; these contributions live in `src/internal/bytealg`.
- **Discord** (in their state service) discusses string-vs-bytes trade-offs and GC pressure: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **CockroachDB** uses an interning pool (`pkg/util/intern`) keyed by `string` to dedupe column names — classic memory-saving pattern.
- **Tailscale** keeps frequently-accessed configuration as preformatted `string` so it can be passed across goroutines without copies.

## Source Code References

Pinned to `go1.26`.

- String runtime ops (concat, conversions): [`src/runtime/string.go`](https://github.com/golang/go/blob/master/src/runtime/string.go).
- `strings.Builder`: [`src/strings/builder.go`](https://github.com/golang/go/blob/master/src/strings/builder.go).
- `bytes.Buffer`: [`src/bytes/buffer.go`](https://github.com/golang/go/blob/master/src/bytes/buffer.go).
- UTF-8 decoder: [`src/unicode/utf8/utf8.go`](https://github.com/golang/go/blob/master/src/unicode/utf8/utf8.go).
- `strings.Index` and family (with assembly stubs): [`src/internal/bytealg`](https://github.com/golang/go/tree/master/src/internal/bytealg).
- `unsafe.String` / `unsafe.SliceData`: [`src/unsafe/unsafe.go`](https://github.com/golang/go/blob/master/src/unsafe/unsafe.go).
- Compiler optimization for `m[string(b)]`: search `cmd/compile/internal/walk` for `OINDEXMAP` / `string(b)` patterns.

## Further Reading

- Go blog, "Strings, bytes, runes and characters in Go": https://go.dev/blog/strings
- Go blog, "Go strings, bytes, runes and characters": https://go.dev/blog/strings (Rob Pike)
- Spec, "String types": https://go.dev/ref/spec#String_types
- Spec, "Conversions to and from a string type": https://go.dev/ref/spec#Conversions_to_and_from_a_string_type
- Unicode FAQ: https://unicode.org/faq/utf_bom.html
- `golang.org/x/text` overview: https://pkg.go.dev/golang.org/x/text
- Russ Cox, "Regular Expression Matching Can Be Simple And Fast": https://swtch.com/~rsc/regexp/regexp1.html (motivates RE2, Go's regex engine)

## Exercises / Self-Check

1. Why does `len("日本語")` return 9 and not 3?
2. Convert `[]byte("hello")` to a `string` without allocation. What's the rule you must obey afterwards?
3. Write a function that reverses a UTF-8 string correctly. (Hint: reversing bytes won't work.)
4. Benchmark string concatenation 1000 times with `+=` vs `strings.Builder`. Plot the difference.
5. Why does `string(65)` give `"A"` and not `"65"`? Which `go vet` check fires?
