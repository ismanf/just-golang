# Empty Interface & `any`

## TL;DR

`any` is an **alias for `interface{}`** introduced in Go 1.18 — exactly the same type, just a friendlier spelling. It holds a value of any type, paying the cost of an interface header (`eface`: type pointer + data pointer) and, often, a heap allocation to box the value. Use it when you genuinely need heterogeneous values (encoding, formatting, generic containers pre-1.18); avoid it when generics or a sealed interface would express the API better.

## Mental Model

```
var x any = 42
                      ┌──────────────┐
                      │   *_type     │ ──► descriptor for int
        x:            │   *data      │ ──► heap-allocated copy of 42
                      └──────────────┘
                          (eface)
```

`eface` differs from `iface` only in that it has no `itab` — no method table is needed because there are no methods to dispatch.

## Syntax & Basic Usage

```go
package main

import "fmt"

func describe(v any) {
	switch x := v.(type) {
	case nil:
		fmt.Println("nil")
	case string:
		fmt.Printf("string: %q\n", x)
	case int:
		fmt.Printf("int: %d\n", x)
	case []any:
		fmt.Printf("slice of any, len=%d\n", len(x))
	default:
		fmt.Printf("unknown type %T: %v\n", x, x)
	}
}

func main() {
	describe(nil)
	describe("hi")
	describe(42)
	describe([]any{1, "two", 3.0})
	describe(struct{ X int }{X: 1})
	// Output:
	// nil
	// string: "hi"
	// int: 42
	// slice of any, len=3
	// unknown type struct { X int }: {1}
}
```

`any` is identical to `interface{}`:

```go
var a any = 1
var i interface{} = a // no conversion needed
```

## Deep Dive

### Why the alias

Before 1.18, code was full of `interface{}` — visually noisy and easy to misread. The alias landed alongside generics, where unconstrained type parameters spell out as `any` (`func F[T any](...)`). Equivalence is exact: same type, same layout, same behavior.

```go
type alias = any
// Method sets, assignability, type assertions: all identical to interface{}.
```

### Boxing cost

Putting a value into `any` may allocate:

```go
var x any = 7      // small int: compiler may avoid the alloc (one-word boxed value)
var y any = "hi"   // string: header is two words; needs a heap copy
var z any = []int{1,2,3} // slice: three words; needs a heap copy
```

The compiler can elide the alloc for some cases (a single word that's a pointer; cached singletons for small ints; `nil`). Don't rely on the optimization — when in doubt, measure.

```go
package main

import (
	"runtime"
	"testing"
)

func BenchmarkBox(b *testing.B) {
	var x any
	for i := 0; i < b.N; i++ {
		x = i // forces escape to interface
	}
	runtime.KeepAlive(x)
}
// go test -bench=. -benchmem
// BenchmarkBox-12   1000000000   ~0.5 ns/op   0 B/op   0 allocs/op   (with small-int optimization)
```

Switch to `var x any = []int{i}` and the allocations and bytes-per-op explode.

### Reading back: assertions and switches

```go
v, ok := x.(int)             // comma-ok, never panics
v := x.(int)                 // panics if not int
switch v := x.(type) { ... } // exhaustive over types
```

A bare type assertion (`x.(int)` without comma-ok) panics with `interface conversion: interface {} is X, not Y`. Always use comma-ok in code that handles untrusted shapes.

### `any` and `nil`

```go
var x any
fmt.Println(x == nil) // true

var p *int
x = p
fmt.Println(x == nil) // false — typed nil
```

Same trap as with non-empty interfaces. The `eface`'s type pointer is non-nil even though its data is nil. See the interfaces page.

### `[]any` is **not** `[]T` for any T

```go
nums := []int{1, 2, 3}
// var anys []any = nums // compile error: cannot use nums (type []int) as type []any
```

Each element would need to be boxed individually. Convert manually:

```go
anys := make([]any, len(nums))
for i, v := range nums { anys[i] = v }
```

Variadic functions are forgiving:

```go
fmt.Println(1, "two", 3.0) // any... captures heterogeneous args
```

But explicit slice-of-`any` conversions remain manual.

### `map[K]any` and JSON

`encoding/json` decodes into `any` as:

- JSON null → `nil`
- JSON bool → `bool`
- JSON number → `float64` (NOT `int`, even for whole numbers!)
- JSON string → `string`
- JSON array → `[]any`
- JSON object → `map[string]any`

```go
var v any
_ = json.Unmarshal([]byte(`{"id":1,"name":"x"}`), &v)
m := v.(map[string]any)
id := int(m["id"].(float64)) // float64 → int — easy to forget
```

`json.Number` (when `Decoder.UseNumber()` is called) preserves number precision.

### `any` vs generics

```go
// any-flavored, runtime-typed:
func First(s []any) any { return s[0] }

// generic, compile-time-typed:
func First[T any](s []T) T { return s[0] }
```

The generic version preserves the element type, avoids boxing, and lets the compiler check the return is used correctly. Almost always preferable post-1.18.

### Identity with `any`

Two `any` values compare equal if both type and value match (per the comparable rules). Comparing two non-comparable dynamic types (slices, maps, funcs) **panics**:

```go
var a, b any = []int{1}, []int{1}
fmt.Println(a == b) // panic: runtime error: comparing uncomparable type []int
```

Use `reflect.DeepEqual` for those cases — but be aware it's slow and surprising.

## Standard Library Hooks

- `fmt.Println(args ...any)` and the entire `fmt` family — variadic `any` is the foundational use case.
- `encoding/json`, `encoding/xml`, `encoding/gob` — produce/consume `any` for unstructured data.
- `errors.Is(err, target error)`, `errors.As(err, &target)` — target is `any` under the hood via reflection.
- `reflect.ValueOf(any)` — bridge from `any` to runtime type info.
- `sync.Map`, `sync.Pool` — key/value typed as `any`.
- `log.Print`, `slog.Info(msg, key, val, ...)` — `any...` variadic.
- `context.WithValue(parent, key, val any)` — key/value are `any`; the canonical "do not abuse `any` for typed data" warning lives in its docs.

## Real-World Patterns

### 1. Heterogeneous arguments to a formatter

```go
slog.Info("user request",
	"user_id", uid,
	"path", r.URL.Path,
	"latency_ms", lat.Milliseconds(),
)
```

`any` is the only sane type for slog's key/value pairs because the value types vary. The cost is paid once per log call and easily dwarfed by the log handler's IO.

### 2. Configuration with dynamic schema

```go
type Config map[string]any

func (c Config) Int(key string) (int, bool) {
	v, ok := c[key]
	if !ok { return 0, false }
	switch x := v.(type) {
	case int:    return x, true
	case int64:  return int(x), true
	case float64: return int(x), true // JSON path
	}
	return 0, false
}
```

For TOML/YAML/JSON config where schema is loose, `map[string]any` keeps the parser simple and pushes typing to access sites.

### 3. Cache or pool of objects

```go
var pool = sync.Pool{
	New: func() any { return new(bytes.Buffer) },
}

func handle() {
	b := pool.Get().(*bytes.Buffer)
	defer pool.Put(b)
	b.Reset()
	// ...
}
```

`sync.Pool` predates generics; the API is locked to `any`. Type assert on `Get`.

### 4. Context value carry-over (with discipline)

```go
type requestIDKey struct{}

ctx = context.WithValue(ctx, requestIDKey{}, "abc123")

func RequestID(ctx context.Context) string {
	v, _ := ctx.Value(requestIDKey{}).(string)
	return v
}
```

The key should be an unexported type to avoid collisions. The value is `any` but you wrap access in a typed helper.

## Anti-Patterns & Gotchas

**`any` everywhere because "flexibility".** You've reinvented Python with worse ergonomics. Use generics or sealed interfaces.

**Returning `any` from your own APIs.** Forces every caller to type-assert. Either return a concrete type, return a sealed interface, or make the function generic.

**JSON `number → float64` forgetting.** `m["id"].(int)` panics; you have to assert `float64` and convert. Or use `json.Number` / generated types.

**Comparing `any` values without knowing the dynamic types are comparable.** Panic.

**`context.WithValue` overuse.** The docs are explicit: context values are for request-scoped data crossing API boundaries, not as a hidden parameter store. Type-assert at access. Don't store mutable state.

**Boxing in tight loops.** Every iteration `var x any = i` may allocate, even when you don't keep the reference. Convert to a generic helper or move the box outside the loop.

**Putting `any` in a generic parameter (`func F[T any]`) when you actually want a constrained set.** If you only ever call `F` with numbers, write `func F[T Number]` and skip the boxing-by-other-means.

## Performance Notes

- `var x any = T{}` may allocate `sizeof(T)` on the heap. The compiler can avoid the alloc if `T` is a pointer (1 word) or for some "always-same" values (`true`, `false`, small ints — the latter via a stash of pre-allocated ones).
- Type assertions are O(1) — a pointer compare on the type descriptor.
- Type switches with N cases compile to a sequence of compares; the compiler may use a jump table or hash-based dispatch for many cases.
- Reading from `any` doesn't allocate; storing into `any` may.
- `[]any` has 16 bytes per slot (header) + per-element box. A `[]int` of N elements is 8N bytes; the equivalent `[]any` is at least 16N bytes plus N alloc headers.
- `interface{}` and `any` are byte-identical at the type level — no perf difference.

## How Big Companies Use It

- **`slog` (introduced in 1.21)**, originating from Google's internal logging libraries, deliberately uses `any` for log values so handlers can do their own typing. Custom handlers like `slog.JSONHandler` dispatch via type switches per attribute.
- **HashiCorp's `hcl` (HashiCorp Configuration Language)** parser exposes `cty.Value` which is a tagged union — but at the Go API boundary, decoded values often surface as `any` through `map[string]any`.
- **OpenTelemetry Go SDK** uses `attribute.Value` (a hand-rolled tagged union) rather than `any` for performance — a case study in why the standard `any` cost is sometimes unacceptable.
- **Caddy's config system** parses JSON into `map[string]any`, then re-encodes into typed module configs. The flexibility-vs-cost trade-off is paid once at startup.
- **`text/template` and `html/template`** pass values as `any` through the template execution pipeline — necessary for arbitrary user data, but a known hot spot.

## Source Code References

Pinned to `go1.26`.

- `eface` definition: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/master/src/runtime/runtime2.go).
- Boxing into interface: [`src/runtime/iface.go`](https://github.com/golang/go/blob/master/src/runtime/iface.go), functions `convT*`.
- Small-integer cache (`staticuint64s`): [`src/runtime/iface.go`](https://github.com/golang/go/blob/master/src/runtime/iface.go), search `staticuint64s`.
- `any` alias declaration: [`src/builtin/builtin.go`](https://github.com/golang/go/blob/master/src/builtin/builtin.go) — `type any = interface{}`.
- `sync.Pool.Get` returning `any`: [`src/sync/pool.go`](https://github.com/golang/go/blob/master/src/sync/pool.go).
- `slog` attribute handling: [`src/log/slog/attr.go`](https://github.com/golang/go/blob/master/src/log/slog/attr.go).

## Further Reading

- Go 1.18 release notes (introducing `any`): https://go.dev/doc/go1.18
- Spec, "Interface types" with the empty case: https://go.dev/ref/spec#Interface_types
- Go FAQ, "Why does my nil error value not equal nil?": https://go.dev/doc/faq#nil_error
- Russ Cox, "Go Data Structures: Interfaces": https://research.swtch.com/interfaces
- Eli Bendersky, "Interfaces in Go": https://eli.thegreenplace.net/2018/go-internals-capturing-loop-variables-in-closures (and series)
- Discussion of `any` cost in Discord state service: https://discord.com/blog/why-discord-is-switching-from-go-to-rust

## Exercises / Self-Check

1. Write a `Coalesce(values ...any) any` returning the first non-nil arg. Watch out for typed nils.
2. Why does `json.Unmarshal([]byte("1"), &v)` give `v = float64(1)` when `v` is `any`? Re-do with `json.Decoder.UseNumber()` and inspect the dynamic type.
3. Benchmark `func F(xs []int) int { /* sum */ }` vs `func F(xs []any) int { /* sum */ }` for 10k elements. Where does the cost difference come from?
4. Demonstrate the comparison panic: build two `any` values holding equal slices, then `==`. Catch the panic with `recover`.
5. Convert this `func Map(xs []any, f func(any) any) []any` to a generic version preserving types. Compare benchmarks.
