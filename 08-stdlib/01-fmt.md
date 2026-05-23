# `fmt` — Formatted I/O

## TL;DR

`fmt` is Go's printf family: `Print*`, `Sprintf`, `Fprintf`, `Errorf`, `Scan*`. It uses reflection-based verbs (`%v`, `%d`, `%s`, `%w`, `%+v`, `%#v`, `%T`) plus three customization hooks: `Stringer`, `GoStringer`, and `Formatter`. Verb resolution at runtime makes `fmt` flexible but allocates and is measurably slower than purpose-built formatters — never put it in a hot inner loop where allocations matter.

## Mental Model

```
fmt.Fprintf(w, "%-10s %5d\n", name, count)
            │       │   │
            │       │   └── width 5
            │       └────── left-justified width 10
            └──────────────  → goes to w (io.Writer)

Verbs walk arguments left-to-right.
If an arg has String()/Error()/Format(), fmt calls it.
Otherwise fmt reflects.
```

Three layers: the *verb* dictates formatting; the *type* of the arg may override via `Stringer`/`Formatter`; the *destination* (`io.Writer`, buffer, string) is chosen by the function name.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println("plain")                       // newline auto
	fmt.Printf("name=%s age=%d\n", "Ada", 37) // formatted, to stdout
	s := fmt.Sprintf("hex=%#x", 255)            // returns string
	fmt.Fprintln(os.Stderr, "warning")          // to any io.Writer
	fmt.Println(s)
	// Output:
	// plain
	// name=Ada age=37
	// hex=0xff
	// warning  (this line goes to stderr)
}
```

## Deep Dive

### Verb cheat sheet (most-used)

| Verb | Meaning                              |
|------|--------------------------------------|
| `%v` | default format                       |
| `%+v`| struct with field names              |
| `%#v`| Go-syntax representation             |
| `%T` | type of the value                    |
| `%d` | decimal int                          |
| `%b` `%o` `%x` `%X` | binary, octal, hex      |
| `%c` | rune as character                    |
| `%U` | Unicode "U+1234" format              |
| `%e` `%f` `%g` | float forms                |
| `%s` | string / []byte / Stringer           |
| `%q` | double-quoted Go-syntax string       |
| `%p` | pointer in hex with `0x`             |
| `%w` | wrap an error (only in `Errorf`)    |

Width/precision: `%5d`, `%-5d` (left), `%05d` (zero-pad), `%.2f`, `%*d` (width from arg).

### `%v` vs `%+v` vs `%#v`

```go
type P struct{ X, Y int }
p := P{1, 2}
fmt.Printf("%v\n", p)   // {1 2}
fmt.Printf("%+v\n", p)  // {X:1 Y:2}
fmt.Printf("%#v\n", p)  // main.P{X:1, Y:2}
```

`%#v` is invaluable in tests: copy-paste the output back into Go source.

### Customization hooks

```go
type Money struct{ Cents int64 }

func (m Money) String() string {
	return fmt.Sprintf("$%d.%02d", m.Cents/100, m.Cents%100)
}
// %v / %s / Println all call String().

type RawBytes []byte
func (b RawBytes) GoString() string { return fmt.Sprintf("%x", []byte(b)) }
// %#v calls GoString().

// Full control with Formatter: low-level access to the State.
type Color struct{ R, G, B uint8 }
func (c Color) Format(s fmt.State, verb rune) {
	switch verb {
	case 'v', 's':
		fmt.Fprintf(s, "#%02x%02x%02x", c.R, c.G, c.B)
	case 'd':
		fmt.Fprintf(s, "(%d,%d,%d)", c.R, c.G, c.B)
	}
}
```

The `error` interface acts like `Stringer` for `%v`/`%s`: `fmt.Println(err)` calls `err.Error()`.

### `%w` and error wrapping

`%w` is special: only valid inside `fmt.Errorf`. It wraps the argument so `errors.Is` / `errors.As` walk through. Multi-`%w` is allowed since 1.20. See `05-errors/03-error-wrapping.md`.

### Scanning

`fmt.Scan*` reads from stdin or a string; mostly used in interview problems and toy programs. Production input parsing should use `bufio.Scanner`, `encoding/json`, or `strconv`.

### `Print` family

| Family   | Destination          | Newline | Formatting |
|----------|----------------------|---------|------------|
| `Print`  | stdout                | no      | spaces between args |
| `Println`| stdout                | yes     | spaces between args |
| `Printf` | stdout                | no      | verbs      |
| `Fprint*`| any `io.Writer`       | varies  | varies     |
| `Sprint*`| returns string        | varies  | varies     |
| `Errorf` | returns error         | no      | verbs (`%w` allowed) |

### How `fmt` resolves a verb

Pseudocode:

```
for each arg matched to a verb:
    if arg implements Formatter: call Format(state, verb); continue
    if verb is %v or %s or %q and arg implements Stringer/Error: call String()/Error()
    else: reflect on arg's kind and format
```

Implication: a custom `Format` overrides everything; a `String()` overrides default reflect formatting; reflection is the slow fallback.

### Padding and alignment for tables

```go
fmt.Printf("%-20s %10d\n", "items processed", 4823)
// "items processed         4823"
```

For aligned multi-row tables, prefer `text/tabwriter`:

```go
w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
fmt.Fprintln(w, "Name\tAge\tCity")
fmt.Fprintln(w, "Ada\t37\tLondon")
fmt.Fprintln(w, "Linus\t54\tPortland")
w.Flush()
```

## Standard Library Hooks

- `text/tabwriter` for aligned columnar output.
- `strconv` for non-reflective number parsing/formatting (much faster).
- `io.Writer` is the universal sink: `os.Stdout`, `bytes.Buffer`, `strings.Builder`, `gzip.Writer`, `http.ResponseWriter`.
- `log` and `log/slog` use `fmt`-style verbs for their string forms.
- `encoding/json`'s `Marshal` uses `Stringer` only for specific types (e.g., `time.Time`).

## Real-World Patterns

### 1. Implement `Stringer` for log-friendly types

```go
type UserID int64

func (u UserID) String() string { return fmt.Sprintf("U%d", u) }

slog.Info("login", "user", UserID(42)) // logs user=U42
```

### 2. Build CLI output with `tabwriter`

```go
import "text/tabwriter"

func renderTable(rows [][]string) {
	w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
	for _, r := range rows {
		fmt.Fprintln(w, strings.Join(r, "\t"))
	}
	w.Flush()
}
```

Use case: `kubectl get pods` style command output.

### 3. Sprintf-free hot path

```go
// Slow:  fmt.Sprintf("%d", n)        // ~50 ns/op, 1 alloc
// Fast:  strconv.Itoa(n)             // ~5 ns/op, 0 allocs (for small n)
```

Hot serialization loops in metrics exporters (Prometheus, OTel) use `strconv` + `strings.Builder` explicitly to avoid `fmt`.

### 4. Reusable buffer with `fmt.Fprintf`

```go
var buf bytes.Buffer
for _, r := range records {
	buf.Reset()
	fmt.Fprintf(&buf, "%s,%d,%s\n", r.Name, r.Count, r.Time)
	if _, err := dst.Write(buf.Bytes()); err != nil { return err }
}
```

Reuses one buffer instead of building strings per iteration. `bytes.Buffer` implements `io.Writer`.

## Anti-Patterns & Gotchas

**`fmt.Sprintf("%s", err)`** when `err` is `nil` produces `"%!s(<nil>)"`. Check first.

**`fmt.Sprintf("%v", nil)`** produces `"<nil>"`; harmless but distinct.

**Forgetting `%w` for wrapping.** `fmt.Errorf("ctx: %v", err)` loses the chain.

**`fmt.Println(largeStruct)` in hot logs.** Each call reflects every field. Use `slog` with explicit attrs.

**Padding without specifying width on a Stringer.** `%-20s` on a `Stringer` works (it calls `String()` first, then pads).

**Mixing `Print` and `Println` randomly.** Pick one per code path; mixing produces awkward trailing spaces or no newlines.

**Reading-back `%#v`** to parse Go values — only works for simple types, not maps with pointers, channels, funcs.

**`fmt.Errorf("%w", nil)`** wraps a nil error: the resulting non-nil wrapper has `Unwrap() == nil`. Guard before wrapping.

## Performance Notes

- `fmt.Sprintf("%d", n)` ≈ 40–60 ns/op, 1 alloc. `strconv.Itoa(n)` ≈ 5 ns/op, 0 allocs.
- `fmt.Sprintf("%s %s", a, b)` builds an intermediate format machine; for known shapes, manual `a + " " + b` is faster.
- `fmt.Fprintln(os.Stdout, ...)` acquires a global mutex on `os.Stdout` (since the writer is shared). High-concurrency loggers prefer buffered or per-goroutine writers.
- Reflection cost dominates for structs with many fields. Cache the formatted output if it doesn't change.
- The `pp` pool inside `fmt` reuses a per-call state struct. You don't manage it, but it bounds allocation pressure.

## How Big Companies Use It

- **Kubernetes**: `kubectl` output combines `text/tabwriter` with `fmt`. Status messages use `Stringer` on resource types.
- **Docker**: container/image listing uses `tabwriter`; image IDs implement `Stringer` to render shortened SHA.
- **Prometheus `client_golang`**: avoids `fmt.Sprintf` in the metric write path; uses `strconv` + buffer pools.
- **Cockroach**: heavy use of `redact.Sprintf` (a `cockroachdb/redact` wrapper around `fmt`) to mark PII spans in formatted output.

## Source Code References

Pinned to `go1.26`.

- Core printer: [`src/fmt/print.go`](https://github.com/golang/go/blob/master/src/fmt/print.go).
- Scanner: [`src/fmt/scan.go`](https://github.com/golang/go/blob/master/src/fmt/scan.go).
- Errorf and `%w`: [`src/fmt/errors.go`](https://github.com/golang/go/blob/master/src/fmt/errors.go).
- `tabwriter`: [`src/text/tabwriter/tabwriter.go`](https://github.com/golang/go/blob/master/src/text/tabwriter/tabwriter.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/fmt.
- Effective Go, "Printing": https://go.dev/doc/effective_go#printing.
- Go blog, "JSON and Go" (touches on Stringer): https://go.dev/blog/json.
- Russ Cox, on the unreasonable effectiveness of small interfaces (Stringer is the canonical example).

## Exercises / Self-Check

1. Implement `Stringer` for a `time.Duration` wrapper that renders `1h30m` style. Compare `fmt.Println(d)` with `d.String()`.
2. Why does `fmt.Sprintf("%d", uint64(1<<63))` overflow on `%d` but not `%x`? Read the verb table to find out.
3. Benchmark `fmt.Sprintf("%d-%d", a, b)` against `strconv.Itoa(a) + "-" + strconv.Itoa(b)` for 1M iterations.
4. Build a `tabwriter`-aligned table from a `[]struct{ Name string; Score float64 }` slice. Format the score to 2 decimals.
5. Trace what `fmt.Println(err)` does when `err` is a custom type implementing both `Error()` and `Format(State, rune)`. Which wins?
