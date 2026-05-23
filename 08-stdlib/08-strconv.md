# `strconv` — Strings ↔ Numbers, Quoting, Booleans

## TL;DR

`strconv` converts between strings and primitive types. It's faster, more precise, and more controllable than `fmt`. Use `Atoi`/`Itoa` for base-10 ints, `ParseFloat`/`FormatFloat` for floats with precision control, `Quote`/`Unquote` for Go-syntax string escaping, `Append*` variants to fill a byte buffer without allocation.

## Mental Model

```
fmt.Sprintf("%d", n)        → reflection-based, slow, allocs
strconv.Itoa(n)             → optimized digit loop, often allocation-free
strconv.AppendInt(buf, n, 10) → no return value alloc, appends to buf

Parse* returns (value, error). Format* returns string. Append* appends to []byte.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	n, err := strconv.Atoi("42")
	fmt.Println(n, err) // 42 <nil>

	s := strconv.Itoa(99) // "99"
	fmt.Println(s)

	f, _ := strconv.ParseFloat("3.14159", 64)
	fmt.Println(f)

	b, _ := strconv.ParseBool("true")
	fmt.Println(b)

	q := strconv.Quote(`hello "world"`)
	fmt.Println(q)

	// Output:
	// 42 <nil>
	// 99
	// 3.14159
	// true
	// "hello \"world\""
}
```

## Deep Dive

### Integer parsing

- `Atoi(s)` — equivalent to `ParseInt(s, 10, 0)` truncated to `int`.
- `ParseInt(s, base, bitSize)` — base 0 for auto-detect (`0x`/`0b`/`0o` prefix). bitSize 0 = platform int.
- `ParseUint(s, base, bitSize)`.

Errors: `*strconv.NumError` with fields `Func`, `Num`, `Err`. The `Err` is typically `ErrSyntax` or `ErrRange`.

```go
_, err := strconv.Atoi("hello")
var nerr *strconv.NumError
if errors.As(err, &nerr) {
	fmt.Println(nerr.Func, nerr.Num, nerr.Err)
	// Atoi hello invalid syntax
}
```

### Integer formatting

- `Itoa(int)` — base 10.
- `FormatInt(i int64, base)` / `FormatUint`.
- `AppendInt(buf, i, base)` / `AppendUint` — append to `[]byte` without allocating a string.

### Float parsing

- `ParseFloat(s, bitSize)` — bitSize 32 or 64. Returns `float64` either way (cast for 32).
- Supports `1.5e10`, `0x1.8p3` (hex floats), `Inf`, `+Inf`, `-Inf`, `NaN`.

### Float formatting

```go
strconv.FormatFloat(3.14, 'f', 2, 64)  // "3.14"
strconv.FormatFloat(3.14, 'e', 4, 64)  // "3.1400e+00"
strconv.FormatFloat(3.14, 'g', -1, 64) // "3.14" — shortest round-trippable
```

Format characters: `'b'` (binary exponent), `'e'`, `'E'` (scientific), `'f'` (decimal), `'g'`, `'G'` (auto-choose), `'x'`, `'X'` (hex with binary exponent).

`prec = -1` for `'g'` means "shortest representation that round-trips."

### Quoting

```go
strconv.Quote("hi\nthere")          // `"hi\nthere"`
strconv.QuoteToASCII("héllo")       // `"héllo"`
strconv.QuoteRune('世')             // `'世'`
strconv.Unquote(`"hi\nthere"`)      // "hi\nthere", nil
strconv.AppendQuote(buf, s)         // append to buffer
```

Use case: generating Go source code, escaping for JSON-like formats.

### Booleans

```go
strconv.ParseBool("1")     // true
strconv.ParseBool("True")  // true
strconv.ParseBool("yes")   // ERROR — not accepted
strconv.FormatBool(true)   // "true"
```

Accepted: `1`, `t`, `T`, `TRUE`, `true`, `True`, `0`, `f`, `F`, `FALSE`, `false`, `False`. Nothing else.

### `Append*` for zero-alloc formatting

```go
var buf []byte
buf = strconv.AppendInt(buf, 42, 10)
buf = append(buf, ',')
buf = strconv.AppendFloat(buf, 3.14, 'f', 2, 64)
// buf = "42,3.14"
```

Used heavily in serializers (JSON encoder, Prometheus exposition).

### Width and padding — NOT here

`strconv` has no width/padding options. Use `fmt.Sprintf("%05d", n)` for padding, or build manually.

### Hex strings

For raw hex bytes, prefer `encoding/hex` (single-byte at a time and Decode/Encode). `strconv.FormatInt(..., 16)` is for integer-to-hex.

## Standard Library Hooks

- `fmt` — calls `strconv` internally for numeric verbs.
- `encoding/json` — `strconv.ParseFloat` for numbers, `Quote`/`Unquote` for strings.
- `text/template`, `html/template` — convert via `fmt`, ultimately `strconv`.
- `net/url` — parses ports as ints via `strconv`.

## Real-World Patterns

### 1. Hot-path int formatting

```go
// Slow:  fmt.Sprintf("%d", id)   ~50 ns/op + 1 alloc
// Fast:  strconv.Itoa(id)        ~5 ns/op + sometimes 1 alloc
// Faster (no alloc): strconv.AppendInt(buf[:0], int64(id), 10)
```

Use case: high-throughput log/metric formatters.

### 2. Robust user input parsing

```go
func parsePort(s string) (uint16, error) {
	n, err := strconv.ParseUint(s, 10, 16)
	if err != nil { return 0, fmt.Errorf("port %q: %w", s, err) }
	if n == 0 { return 0, errors.New("port must be > 0") }
	return uint16(n), nil
}
```

Use case: CLI flag parsing, config reading.

### 3. Round-trip float serialization

```go
f := 0.1 + 0.2 // 0.30000000000000004
s := strconv.FormatFloat(f, 'g', -1, 64)
back, _ := strconv.ParseFloat(s, 64)
// back == f exactly
```

`'g'` with `prec = -1` guarantees round-trip. Use it for JSON-style float encoding.

### 4. Build a CSV line zero-alloc

```go
var buf []byte
for i, n := range nums {
	if i > 0 { buf = append(buf, ',') }
	buf = strconv.AppendInt(buf, n, 10)
}
buf = append(buf, '\n')
w.Write(buf)
```

Use case: data exporters.

### 5. Generate Go source

```go
fmt.Fprintf(&out, "var name = %s\n", strconv.Quote(value))
```

Use case: code generators (`stringer`, `protoc-gen-go`).

## Anti-Patterns & Gotchas

**`fmt.Sprintf("%d", n)` everywhere.** Reach for `strconv.Itoa` (and `Append*` in tight loops).

**Ignoring `strconv.NumError`.** Don't drop the error; users want to know what string failed.

**`strconv.Atoi(s) != 0` as a validity check.** `Atoi("0")` returns 0 with no error.

**Parsing floats from user input without checking range.** `ParseFloat` returns `ErrRange` for ±Inf — `errors.Is(err, strconv.ErrRange)`.

**Using `ParseInt` then converting to a smaller type.** Use `ParseInt(s, 10, 8)` so the bitSize check happens for free.

**`FormatBool` then comparing.** `if strconv.FormatBool(b) == "true"` is pointless; just use `b`.

**`strconv.Quote` for JSON strings.** Close but not exactly JSON-compliant; use `encoding/json` for JSON output.

**`AppendInt(buf, n, 10)` with `buf == nil` and forgetting to assign back.** Append may allocate; use the return value.

## Performance Notes

- `strconv.Itoa` for small ints: ~5 ns/op, 0 allocs (the result string may be interned in some cases; usually 1 small alloc).
- `strconv.AppendInt` to an existing buffer: ~5 ns/op, 0 allocs.
- `strconv.FormatFloat` is more expensive (Ryu algorithm since 1.12); single-digit microseconds for high-precision values.
- `ParseFloat` for `'g'`-format strings is fast; arbitrary scientific notation is slower.
- `fmt.Sprintf("%d", n)` allocates the pp struct from pool, but always allocates the result string.

## How Big Companies Use It

- **Prometheus client_golang** uses `strconv.AppendFloat` and `AppendInt` to build the exposition format byte-by-byte, no allocations on the metric write path.
- **Kubernetes** parses time, quantity, and resource limits with `strconv.ParseInt`/`ParseFloat` wrapped in domain validation.
- **Cockroach** uses `strconv.Quote` for SQL string literal generation in test fixtures.
- **gRPC-Go** uses `strconv.Itoa` for status code rendering in error messages.

## Source Code References

Pinned to `go1.26`.

- `strconv` core: [`src/strconv/atoi.go`](https://github.com/golang/go/blob/master/src/strconv/atoi.go), `itoa.go`, `atof.go`, `ftoa.go`, `quote.go`.
- `NumError`: [`src/strconv/atoi.go`](https://github.com/golang/go/blob/master/src/strconv/atoi.go).
- Ryu algorithm for floats: [`src/strconv/ftoa.go`](https://github.com/golang/go/blob/master/src/strconv/ftoa.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/strconv.
- Ryu paper, "Ryū: Fast Float-to-String Conversion" (Ulf Adams): https://dl.acm.org/doi/10.1145/3192366.3192369.

## Exercises / Self-Check

1. Benchmark `fmt.Sprintf("%d", n)` vs `strconv.Itoa(n)` vs `strconv.AppendInt(buf, int64(n), 10)`.
2. Why does `strconv.ParseInt("0xff", 0, 32)` succeed but `strconv.ParseInt("0xff", 10, 32)` fail?
3. Parse user input that may be `"true"`, `"yes"`, `"on"`, or `"1"` as booleans. Wrap `strconv.ParseBool` to add the missing cases.
4. Generate a Go source file with `strconv.Quote` for string constants. Verify the output compiles.
5. Show that `strconv.FormatFloat(0.1+0.2, 'g', -1, 64)` round-trips exactly through `ParseFloat`.
