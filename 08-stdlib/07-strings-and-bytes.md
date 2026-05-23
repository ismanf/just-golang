# `strings` and `bytes` — Text and Byte-Slice Manipulation

## TL;DR

`strings` and `bytes` mirror each other: same functions, one operates on immutable `string`, the other on mutable `[]byte`. Use `strings.Builder` to assemble strings without quadratic allocation; use `bytes.Buffer` for byte assembly. Hot paths almost always benefit from staying in `[]byte` and converting once at the boundary — every `string([]byte)` and `[]byte(string)` is a copy unless the compiler proves otherwise.

## Mental Model

```
string  : immutable, header = {*data, len}; cheap to slice (no copy); compare with ==
[]byte  : mutable,   header = {*data, len, cap}; can append; can pool
                     ↕ conversion copies, unless compiler eliminates it
```

The dual `strings`/`bytes` packages let you pick the right type for the job.

## Syntax & Basic Usage

```go
package main

import (
	"bytes"
	"fmt"
	"strings"
)

func main() {
	fmt.Println(strings.Contains("hello", "ell"))     // true
	fmt.Println(strings.Split("a,b,c", ","))          // [a b c]
	fmt.Println(strings.ReplaceAll("a.b.c", ".", "-")) // a-b-c

	var b strings.Builder
	for i := 0; i < 5; i++ {
		fmt.Fprintf(&b, "n=%d ", i)
	}
	fmt.Println(b.String())

	buf := bytes.NewBufferString("init")
	buf.WriteByte(' ')
	buf.WriteString("more")
	fmt.Println(buf.String())

	// Output:
	// true
	// [a b c]
	// a-b-c
	// n=0 n=1 n=2 n=3 n=4 
	// init more
}
```

## Deep Dive

### The dual API

Almost every function in `strings` has a `bytes` twin: `strings.Contains` ↔ `bytes.Contains`, `strings.Index` ↔ `bytes.Index`, etc. Differences:

- `bytes` has `bytes.Buffer` (mutable); `strings` has `strings.Builder` (also mutable, but write-only).
- `bytes.Reader` and `strings.Reader` implement `io.Reader`/`io.Seeker`/`io.ReaderAt`.
- `bytes` deals in byte slices; `strings` in immutable UTF-8 strings.

### `strings.Builder` (since 1.10) — the right way to concatenate

```go
var b strings.Builder
b.Grow(1024)               // pre-allocate
for _, s := range items {
	b.WriteString(s)
	b.WriteByte('\n')
}
result := b.String()       // no copy; reuses Builder's buffer
```

Properties:

- Internal buffer reuses growth like `[]byte`. `Grow(n)` reserves capacity.
- `String()` returns without copying (the Builder is unusable after, since it can't be reset and then yield the same buffer safely; `Reset()` discards).
- Cheaper than `+=` in a loop (which is O(n²)).
- Cheaper than `fmt.Sprintf` for known shapes.

Never copy a `strings.Builder` — its internal pointer points back into itself and copying invalidates the address.

### `bytes.Buffer` — the workhorse

```go
var buf bytes.Buffer
buf.Write([]byte{1, 2, 3})
buf.WriteByte(0xff)
fmt.Fprintf(&buf, "x=%d", 42)
data := buf.Bytes() // direct view; valid until next mutation
```

Implements `io.Reader`, `io.Writer`, `io.ByteReader`, `io.ByteWriter`. Use anywhere a `Writer` is expected.

Reusable: `buf.Reset()` keeps the underlying capacity but length 0.

### Conversion costs

```go
b := []byte("hello") // copies into a new buffer
s := string(b)        // copies again

// Hot path optimization the compiler does:
m := make(map[string]int)
m[string(b)] = 1     // compiler may avoid the copy for the lookup key
n := m[string(b)]    // same — see 13-performance/08
```

`string(byteSlice)` allocates and copies — except in specific patterns recognized by the compiler. The `unsafe`-based shortcuts (`unsafe.String`, `unsafe.SliceData`) were added in 1.20 for cases where you truly need zero-copy and know the lifetime is safe.

### Searching

```go
strings.Index(s, "x")           // first index, or -1
strings.LastIndex(s, "x")
strings.IndexByte(s, 'x')       // faster for single byte
strings.IndexRune(s, '世')      // faster for single rune
strings.ContainsAny(s, "aeiou") // any of these chars
strings.Count(s, "x")
```

`IndexByte` uses SIMD on x86/ARM. Prefer it for single-character checks.

### Splitting and joining

```go
strings.Split("a,b,c", ",")       // [a b c]
strings.SplitN("a,b,c", ",", 2)   // [a b,c]
strings.SplitAfter("a.b.c", ".")  // [a. b. c]
strings.Fields("a   b\tc\n")      // [a b c] — splits on unicode whitespace
strings.Join([]string{"a","b"}, "-") // a-b
```

`Split` allocates a slice per call; `Fields` allocates plus iterates runes. For a tight loop, `bufio.Scanner` with `ScanWords` may be cheaper.

### Trim, replace, case

```go
strings.TrimSpace("  hi  ")            // "hi"
strings.Trim("xxhi xx", "x ")          // "hi" — set of chars
strings.TrimPrefix("file:foo", "file:") // "foo"
strings.TrimSuffix(s, ".log")
strings.ReplaceAll(s, "x", "y")
strings.Replace(s, "x", "y", 1)        // first occurrence only

strings.ToLower(s)        // unicode-aware
strings.ToUpper(s)
strings.EqualFold(a, b)   // case-insensitive == — UTF-8 aware
```

`EqualFold` is the right comparison for ASCII-or-Unicode case-insensitive equality. `strings.ToLower(a) == strings.ToLower(b)` allocates twice and gets some Unicode cases wrong.

### `strings.Cut` (since 1.18)

```go
key, val, ok := strings.Cut("k=v", "=")
```

Replaces the awkward `Split` + length-check pattern. There's also `strings.CutPrefix` and `strings.CutSuffix` (1.20+).

### `bytes.Reader` and `strings.Reader`

`io.Reader` adapters around in-memory data. Useful when an API wants an `io.Reader` and you have a string.

```go
http.NewRequest("POST", url, strings.NewReader(body))
```

### `bytes.Equal`

```go
bytes.Equal(a, b) // optimized; safe for nil slices
```

Faster than `string(a) == string(b)` (no allocation).

### `strings.NewReplacer`

```go
r := strings.NewReplacer("&", "&amp;", "<", "&lt;", ">", "&gt;")
escaped := r.Replace(s)
```

Compiles a multi-pattern replacer once; reuse for many strings. Cheaper than chained `ReplaceAll` calls.

## Standard Library Hooks

- `unicode`, `unicode/utf8`, `unicode/utf16` for rune-level ops.
- `regexp` for regex matching (slower than `Index*` for fixed patterns).
- `io` for streaming text.
- `strconv` for number ↔ string.
- `text/scanner`, `text/tabwriter` for higher-level text processing.

## Real-World Patterns

### 1. Build a query string efficiently

```go
var b strings.Builder
b.Grow(256)
first := true
for k, v := range params {
	if !first { b.WriteByte('&') }
	first = false
	b.WriteString(url.QueryEscape(k))
	b.WriteByte('=')
	b.WriteString(url.QueryEscape(v))
}
url := base + "?" + b.String()
```

Use case: HTTP client request builders.

### 2. Stream-process a large CSV-ish line

```go
const sep = ','
for s := line; len(s) > 0; {
	idx := strings.IndexByte(s, sep)
	var field string
	if idx < 0 {
		field, s = s, ""
	} else {
		field, s = s[:idx], s[idx+1:]
	}
	process(field)
}
```

Use case: hot-path parsing where allocating a `[]string` from `Split` is too expensive.

### 3. Reusable buffer pool with `sync.Pool`

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func format(item Item) string {
	b := bufPool.Get().(*bytes.Buffer)
	defer func() { b.Reset(); bufPool.Put(b) }()
	fmt.Fprintf(b, "%s/%d", item.Name, item.Count)
	return b.String()
}
```

Use case: per-request formatting in HTTP handlers.

### 4. Replace many tokens in one pass

```go
r := strings.NewReplacer(
	"{name}", user.Name,
	"{id}", strconv.Itoa(user.ID),
	"{role}", user.Role,
)
rendered := r.Replace(template)
```

Use case: tiny templating without bringing in `text/template`.

### 5. Validate input quickly

```go
if !strings.HasPrefix(token, "Bearer ") {
	http.Error(w, "missing bearer", http.StatusUnauthorized)
	return
}
auth := strings.TrimPrefix(token, "Bearer ")
```

Use case: HTTP middleware.

## Anti-Patterns & Gotchas

**`s += "x"` in a loop.** O(n²) allocations. Use `strings.Builder`.

**Concatenating with `+` across many tokens.** Each `+` allocates. Build with Builder, or use `strings.Join`.

**`string(b) == string(b2)`** when `bytes.Equal(b, b2)` is cheaper.

**`strings.ToLower(a) == strings.ToLower(b)`.** Use `strings.EqualFold`.

**Splitting then taking only the first element.** Use `strings.Cut`.

**`strings.Builder` shared across goroutines.** Not safe; wrap with a mutex or use one per goroutine.

**Copying a `strings.Builder` value.** Invalid; internal pointer breaks.

**`bytes.Buffer.Bytes()` retained past the next mutation.** The slice is invalidated.

**`fmt.Sprintf("%s%s", a, b)`** when `a + b` would do.

**Treating `len(s)` as a count of characters.** It's bytes. Use `utf8.RuneCountInString`.

## Performance Notes

- `strings.IndexByte` is SIMD-accelerated on x86/ARM (since 1.5+).
- `strings.Builder` allocates internal `[]byte` that grows similarly to `append` (≈ 2x doubling, geometric).
- `bytes.Buffer` does likewise.
- `string([]byte)` conversion = `mallocgc(len) + memcpy(len)`. ~ns per byte at large sizes.
- `unsafe.String(&b[0], len(b))` (1.20+) — zero-copy view of a byte slice as string. Use only when you guarantee `b` is not mutated for the string's lifetime.
- `strings.NewReplacer` builds a trie once; per-replace it's O(input length) with constant factor of the trie depth.

## How Big Companies Use It

- **Hugo** uses `strings.Builder` for HTML rendering; explicit `Grow` for templates.
- **Caddy** uses `strings.NewReplacer` for placeholder expansion in config.
- **Prometheus client** uses `bytes.Buffer` pools to format metric text exposition zero-allocation.
- **gopls** uses `strings.Cut` heavily for parsing source positions like `file:line:col`.

## Source Code References

Pinned to `go1.26`.

- `strings`: [`src/strings/strings.go`](https://github.com/golang/go/blob/master/src/strings/strings.go), `builder.go`, `replace.go`, `reader.go`.
- `bytes`: [`src/bytes/bytes.go`](https://github.com/golang/go/blob/master/src/bytes/bytes.go), `buffer.go`, `reader.go`.
- SIMD `IndexByte` implementations: [`src/internal/bytealg/indexbyte_amd64.s`](https://github.com/golang/go/blob/master/src/internal/bytealg/indexbyte_amd64.s).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/strings, https://pkg.go.dev/bytes.
- Go blog, "Strings, bytes, runes and characters in Go": https://go.dev/blog/strings.
- Damian Gryski, "High-performance allocator-free string handling".

## Exercises / Self-Check

1. Benchmark `s += x` vs `b.WriteString(x)` in a loop of 1000 iterations.
2. Compare `strings.ToLower(a) == strings.ToLower(b)` against `strings.EqualFold(a, b)` for "İstanbul" vs "istanbul".
3. Write a function that splits a path into segments without allocating a `[]string`. Hint: callback per segment.
4. Use `strings.NewReplacer` to expand `{NAME}` placeholders. Measure speed vs chained `ReplaceAll`.
5. Why does `var b strings.Builder; b.WriteString("x"); var c = b` then `c.String()` give weird results? Trace.
