# `unique` — Value Interning (since 1.23)

## TL;DR

`unique.Make[T](v)` returns a `Handle[T]` — a stable pointer to a canonical copy of `v`. Two equal values produce two handles whose underlying address is identical, so `h1 == h2` is a pointer comparison. Saves memory when many duplicate string/struct values exist; speeds up equality of large composite keys. The runtime garbage-collects unreferenced canonical values automatically.

## Mental Model

```
unique.Make("hello")  →  Handle{ptr to canonical "hello"}
unique.Make("hello")  →  Handle{same ptr}

Handle equality = pointer equality = O(1)
Underlying type comparison happened ONCE at intern time.

GC: when no Handle references a canonical value, it's collected.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"unique"
)

func main() {
	h1 := unique.Make("hello")
	h2 := unique.Make("hello")
	fmt.Println(h1 == h2)         // true — same canonical value
	fmt.Println(h1.Value())        // "hello"
	fmt.Println(unsafe.Sizeof(h1)) // 8 bytes (a pointer)
	// Output:
	// true
	// hello
	// 8
}
```

## Deep Dive

### `Handle[T comparable]`

```go
type Handle[T comparable] struct{ /* unexported */ }
func Make[T comparable](v T) Handle[T]
func (h Handle[T]) Value() T
```

`T` must be comparable. Handle is itself comparable (`==`) — that's the point.

### Use cases

- **String pools** — many objects share the same string label (HTTP headers, log keys, gRPC method names, JSON field names).
- **Tag-like values** — enums, country codes, currency codes.
- **Composite keys** — `Make(struct{...})` lets you use the handle as a map key, comparing in O(1) regardless of struct size.

### Memory savings

Suppose you have 1M log entries each with a `Method` field that's one of ~20 values:

```go
// Without unique: 1M * 24 bytes (string header) + many copies of the underlying string = a lot
// With unique:    1M * 8 bytes  (handle)        + 20 canonical copies                = much less

type Log struct {
	Time   time.Time
	Method unique.Handle[string]
	Path   unique.Handle[string]
	Status int
}
```

### Garbage collection

Canonical values are weak-referenced (built on `weak.Pointer`, the 1.24 internal feature). When all handles to a value go away, the value is reclaimed.

If you re-`Make(v)` after the value was GCed, you get a fresh canonical copy (different pointer). Most code doesn't notice, but be aware: handle identity is stable only while at least one handle exists.

### Equality and ordering

Handles compare by pointer for equality. They do NOT support `<`/`>` meaningfully — the pointer order is arbitrary. To order by the underlying value, call `.Value()` and compare.

### Map keys

```go
counts := map[unique.Handle[string]]int{}
counts[unique.Make("GET")]++
counts[unique.Make("GET")]++   // same key
// counts[unique.Make("GET")] == 2
```

Hashing the handle hashes the pointer — much faster than hashing a long string.

### Struct interning

```go
type Tag struct{ Key, Value string }
hTag := unique.Make(Tag{Key: "env", Value: "prod"})
```

Useful for label sets in metrics systems.

### Compared to `sync.Map` cache

A handcrafted intern cache with `sync.Map[string]string` works but lacks GC integration. `unique` adds:

- Automatic weak-reference cleanup.
- Built-in thread safety.
- Zero allocation on hit (after first insert).

## Standard Library Hooks

- Built on the internal weak pointer machinery (since 1.24, but `unique` is 1.23+).
- Works with any `comparable` type.

## Real-World Patterns

### 1. Intern HTTP methods and paths in a request log

```go
type Entry struct {
	When   time.Time
	Method unique.Handle[string]
	Path   unique.Handle[string]
	Status int
}

func recordRequest(r *http.Request, status int) {
	log = append(log, Entry{
		When:   time.Now(),
		Method: unique.Make(r.Method),
		Path:   unique.Make(r.URL.Path),
		Status: status,
	})
}
```

Use case: in-process request audit log; 100 MB → 20 MB savings on duplicate strings.

### 2. Prometheus-style label sets

```go
type LabelSet struct{ Pairs string } // pre-encoded canonical "k1=v1,k2=v2"

type Metric struct {
	Name unique.Handle[string]
	Labels unique.Handle[LabelSet]
}
```

Comparing two `Metric` values is now O(1) pointer compare.

### 3. Cache lookup on a heavy key

```go
type Key struct{ Tenant, User, Region string }

var cache = map[unique.Handle[Key]]Value{}

h := unique.Make(Key{Tenant: t, User: u, Region: r})
if v, ok := cache[h]; ok { return v }
```

Use case: caching where the key is a multi-field struct.

### 4. Symbol table in a parser

```go
type Sym = unique.Handle[string]

func (l *Lexer) Intern(s string) Sym { return unique.Make(s) }
```

Identifiers in source code are heavily duplicated; interning at lex time speeds downstream comparisons.

### 5. Enum-like constants from string config

```go
var (
	StateOpen  = unique.Make("open")
	StateClose = unique.Make("close")
)

func setState(s string) { state.Store(unique.Make(s)) }
func isOpen() bool      { return state.Load() == StateOpen }
```

## Anti-Patterns & Gotchas

**Interning truly unique values.** Defeats the purpose; pure overhead.

**Interning values from untrusted input without bounds.** Memory amplification: each unique value sticks until GC. Bound the cardinality at the boundary.

**Storing `Handle` long-term and expecting `Value()` to always return a stable address.** `Value()` returns the canonical value, but if GC reclaimed it and you re-Make later, a new canonical is created.

**Using `Handle` as `<` key in a `[]Handle`-sorted slice.** Pointer order is arbitrary; not meaningful.

**Interning types with embedded pointers/maps/slices** without thinking about equality semantics. `comparable` constraint catches some of this, but pointer-fields compare by identity, not value.

**Forgetting that `Handle` is a value type holding a pointer.** Passing by value is cheap; pointers to Handle are silly.

## Performance Notes

- `Make`: hash the value, look up in an internal sync.Map-like structure; allocate canonical copy only on miss.
- `Value()`: pointer dereference.
- `==` on handles: pointer compare, instant.
- For very large interns (millions of unique values), the internal map grows accordingly. Bound input cardinality.

## How Big Companies Use It

- Adoption is recent; expect interning to show up in:
  - **Prometheus** label sets (community discussions ongoing).
  - **slog handlers** for attribute keys (some are already considering).
  - **Trace span exporters** (OpenTelemetry).
- **gopls** uses interned types for symbol tables — pre-`unique`, they had hand-rolled versions.

## Source Code References

Pinned to `go1.26`.

- `unique`: [`src/unique/`](https://github.com/golang/go/tree/master/src/unique).
- Backing weak pointer (1.24): [`src/weak/`](https://github.com/golang/go/tree/master/src/weak).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/unique.
- Go 1.23 release notes — unique package: https://go.dev/doc/go1.23.
- Russ Cox, "Coroutines and weak references" research notes (background for 1.23/1.24 features).

## Exercises / Self-Check

1. Intern a slice of duplicated strings; measure memory savings vs storing raw strings.
2. Use `Handle[string]` as a map key. Verify O(1) hash.
3. Build a tag struct, intern it, use as map key for counts.
4. Demonstrate GC: drop all handles to a value, force GC, re-Make — observe identity may differ (timing-dependent).
5. Why does `Make` require `T comparable`? What types are excluded?
