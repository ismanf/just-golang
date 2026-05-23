# Type Assertions and Type Switches

## TL;DR

A **type assertion** extracts the concrete (or another interface) value out of an interface: `v, ok := iface.(Concrete)`. A **type switch** dispatches on the dynamic type held by an interface: `switch x := iface.(type) { case A: ...; case B: ... }`. Both are O(1) — a pointer compare on the type descriptor. Always use the comma-ok form on untrusted input; the panic-form crashes the goroutine on failure.

## Mental Model

```
iface  ──► [ *itab/_type | *data ]
                 │
                 ▼
            descriptor for the concrete type
                 │
       Type assertion: compare descriptor with target type's descriptor.
       Equal → unbox; not equal → ok=false / panic.
```

It's just two pointer-equality checks underneath. No magic, no traversal.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	var i any = "hello"

	// comma-ok form — safe
	s, ok := i.(string)
	fmt.Println(s, ok) // hello true

	n, ok := i.(int)
	fmt.Println(n, ok) // 0 false

	// panic form — only when you're certain
	s = i.(string)
	fmt.Println(s)

	// type switch
	switch v := i.(type) {
	case string:
		fmt.Printf("string len=%d\n", len(v))
	case int:
		fmt.Printf("int %d\n", v)
	case nil:
		fmt.Println("nil")
	case fmt.Stringer:
		fmt.Println(v.String())
	default:
		fmt.Printf("other %T\n", v)
	}
	// Output:
	// hello true
	// 0 false
	// hello
	// string len=5
}
```

The two forms compile to the same kind of check; the comma-ok variant just returns the result instead of panicking.

## Deep Dive

### Asserting to a concrete type

```go
v, ok := iface.(*MyStruct)
```

Runtime compares `iface`'s type descriptor with `*MyStruct`'s. Match → `v` is the unboxed pointer. No match → `v = nil`, `ok = false`.

### Asserting to another interface

```go
type Closer interface{ Close() error }

v, ok := iface.(Closer)
```

Runtime checks whether the dynamic type of `iface` has all of `Closer`'s methods. The first time this check fires for a given (dynamic-type, target-interface) pair, the runtime computes a new `itab` and caches it. Subsequent assertions are pointer compares.

### The comma-ok form is mandatory for safety

```go
v := iface.(int)  // panic if not an int — "interface conversion: ..."
```

Use the panic form only when you constructed the interface yourself two lines ago. For anything coming from a parameter, a map, JSON, a channel, etc., always:

```go
v, ok := iface.(int)
if !ok {
	return fmt.Errorf("expected int, got %T", iface)
}
```

### Type switch with multiple cases

```go
switch v := x.(type) {
case int, int64:
	// v has type "any" here — multiple types in one case fall back to the interface
case string:
	// v is string
case nil:
	// v is the nil interface
case Stringer:
	// matches if x.dynamic type satisfies Stringer
}
```

When a case lists more than one type, `v` reverts to the switch's interface type (because the compiler can't pick a single concrete type). Single-type cases bind `v` as that exact type.

### Order matters with interface cases

```go
switch v := x.(type) {
case error:           // matches anything implementing error
case *MyErr:          // unreachable for *MyErr, because *MyErr satisfies error
}
```

Specific concrete cases should come before general interface cases.

### `default` case

```go
switch v := x.(type) {
case string:
	_ = v
default:
	return fmt.Errorf("unsupported type %T", v)
}
```

In `default`, `v` has the switch's interface type (not the dynamic type). To get the dynamic type, use `fmt.Sprintf("%T", v)` or `reflect.TypeOf(v)`.

### Asserting against a generic type parameter — you can't, directly

```go
func F[T any](x any) (T, bool) {
	v, ok := x.(T) // OK since 1.18 — works on instantiated T at runtime
	return v, ok
}
```

The assertion happens at runtime with the instantiated `T`'s type descriptor.

### `errors.As` is a typed assertion in disguise

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	// pathErr is now bound to the matching wrapped error
}
```

It walks the wrap chain and does the equivalent of `assertion-or-skip` at each level. Don't reimplement it.

### Asserting nil interface

```go
var x any
v, ok := x.(string) // v="", ok=false
```

No panic — comma-ok form. Without comma-ok:

```go
var x any
_ = x.(string) // panic: interface conversion: interface {} is nil, not string
```

### Type assertion on the result of a function

```go
v := mustReturnInterface().(*Foo) // panic-form, unaddressable receiver
```

Allowed — the result is an interface value, fine for assertion. But the result of an assertion is not addressable, so `&result` is a compile error in some contexts.

## Standard Library Hooks

- `errors.As(err, &target)` — the production-grade wrapper for "is there an X in this error chain?"
- `reflect.TypeOf(any) reflect.Type` and `reflect.ValueOf(any).Kind()` — slower but works without a known target type.
- `fmt.Sprintf("%T", x)` — prints the dynamic type, including package path.
- `net/http.Hijacker`, `http.Flusher`, `http.Pusher` — `ResponseWriter` is type-asserted at runtime to detect optional capabilities.
- `io.WriterTo`, `io.ReaderFrom` — adapter detection in `io.Copy`.

## Real-World Patterns

### 1. Capability detection on standard types

```go
func tryFlush(w http.ResponseWriter) {
	if f, ok := w.(http.Flusher); ok {
		f.Flush()
	}
}
```

`http.ResponseWriter` is intentionally small; optional capabilities surface via assertions.

### 2. Walking a tagged-union AST

```go
type Expr interface{ exprNode() }

type IntLit struct{ N int };       func (IntLit) exprNode() {}
type StrLit struct{ S string };    func (StrLit) exprNode() {}
type Add    struct{ L, R Expr };   func (Add)    exprNode() {}

func Eval(e Expr) any {
	switch v := e.(type) {
	case IntLit:
		return v.N
	case StrLit:
		return v.S
	case Add:
		l := Eval(v.L).(int)
		r := Eval(v.R).(int)
		return l + r
	default:
		panic(fmt.Sprintf("unknown expr %T", v))
	}
}
```

The `default` panic catches new variants that forgot to extend `Eval`. Pair with a sealed interface to make adding variants impossible outside the package.

### 3. Decoding `map[string]any` from JSON

```go
func getString(m map[string]any, key string) (string, error) {
	v, ok := m[key]
	if !ok {
		return "", fmt.Errorf("missing %q", key)
	}
	s, ok := v.(string)
	if !ok {
		return "", fmt.Errorf("%q: want string, got %T", key, v)
	}
	return s, nil
}
```

Wrap repetitive assertions in tiny accessors. Don't sprinkle `.(string)` everywhere.

### 4. Error inspection with `errors.As`

```go
_, err := os.Open("/no/such")
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Printf("op=%s path=%s err=%v\n", pathErr.Op, pathErr.Path, pathErr.Err)
}
```

### 5. Generic helper that asserts at the boundary

```go
func MustAs[T any](v any) T {
	t, ok := v.(T)
	if !ok {
		panic(fmt.Sprintf("expected %T, got %T", *new(T), v))
	}
	return t
}

cfg := MustAs[*Config](registry.Get("main"))
```

Concentrate the panic at the entry point; downstream code uses the typed value.

## Anti-Patterns & Gotchas

**Bare assertion on input you didn't construct.** `iface.(T)` panics the goroutine. Always comma-ok.

**Asserting to an interface that the dynamic type doesn't satisfy at runtime even though it does at compile time.** This is the typed-nil trap — `var p *T; var e error = p; e.(*T)` returns `(nil, true)`; the assertion *succeeds* but you get a nil pointer.

**`switch x := f().(type)` when `f` returns a value type, not interface.** Type switches only work on interface values.

**Forgetting the `default` case** and treating the switch as exhaustive. The compiler doesn't enforce exhaustiveness; add a `default` panic or an external linter (`exhaustruct`/`exhaustive`).

**Long chains of type assertions** in app code. Often a sign that you should accept a more specific type, generate code, or move to generics.

**`switch v := x.(type) { case A, B: v.SomeField }`.** Compile error — `v` has interface type when the case lists multiple types.

**Assertion vs conversion confusion.** `string(b)` (b is `[]byte`) is a conversion. `i.(string)` (i is `any`) is an assertion. Different operators (parens vs no dot prefix), different rules.

**Reflective alternatives in hot paths.** `reflect.TypeOf(x) == reflect.TypeOf("")` is much slower than `_, ok := x.(string)`.

## Performance Notes

- Type assertions to a concrete type: one pointer compare against the type descriptor. Inlinable.
- Type assertions to an interface: first call computes and caches the `itab`; subsequent calls are a single pointer compare. The first-call cost is small but real.
- Type switch with N concrete cases compiles to N pointer compares; the compiler may use a hash dispatch for large N.
- Type switch over interfaces (each case is an interface type) does the more expensive method-set check, but each result is cached as an `itab`.
- No allocation occurs for an assertion — you're reading bits out of an existing interface header.
- Panicking on a failed assertion is slow (stack walk, deferred recovery), so the comma-ok form is strictly cheaper on the failure path.

## How Big Companies Use It

- **Kubernetes `runtime.Object`-conformant types** are repeatedly type-asserted (or reflected) inside generic codecs. The `client-go` informer cache stores `runtime.Object` and downcasts at the read side with `.(*v1.Pod)` etc.
- **Cockroach's `roachpb`** uses tagged-union messages with type switches for request dispatch.
- **`net/http`'s capability detection** of `Hijacker`/`Flusher`/`Pusher` is the textbook example of assertion-based optional capabilities — both used and copied widely.
- **HashiCorp Vault** uses type assertions in its `logical.Backend` dispatch to route to the right handler for `Storage`-shaped vs `Pluggable`-shaped backends.
- **`encoding/json`** does runtime type switching on the destination interface value to pick the fastest decoder for each kind.

## Source Code References

Pinned to `go1.26`.

- Type assertion lowering and codegen: [`src/cmd/compile/internal/walk/convert.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/convert.go).
- `itab` lookup and caching: [`src/runtime/iface.go`](https://github.com/golang/go/blob/master/src/runtime/iface.go) — functions `getitab`, `assertI2I`, `assertI2T`.
- Type switch codegen: [`src/cmd/compile/internal/walk/switch.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/switch.go).
- `errors.As` and `errors.Is`: [`src/errors/wrap.go`](https://github.com/golang/go/blob/master/src/errors/wrap.go).
- Capability detection in HTTP server: [`src/net/http/server.go`](https://github.com/golang/go/blob/master/src/net/http/server.go) — search `Hijacker`.

## Further Reading

- Spec, "Type assertions": https://go.dev/ref/spec#Type_assertions
- Spec, "Type switches": https://go.dev/ref/spec#Switch_statements (subsection "Type switches")
- Go blog, "Errors are values": https://go.dev/blog/errors-are-values
- "Working with errors in Go 1.13": https://go.dev/blog/go1.13-errors
- Russ Cox, "Go Data Structures: Interfaces": https://research.swtch.com/interfaces

## Exercises / Self-Check

1. Write `func GetInt(m map[string]any, k string) (int, error)` handling both `int` and `float64` (the JSON case).
2. Replace this with an `errors.As`-based version: a loop manually calling `errors.Unwrap` and checking `*fs.PathError`.
3. Build a type switch handling 5 concrete types and one default. Add a 6th type; how should the compiler / vet / linter help you remember to update the switch?
4. Why does `v := iface.(T); &v` sometimes fail to compile? When is the result of an assertion addressable?
5. Benchmark `_, ok := x.(*MyStruct)` vs `reflect.TypeOf(x) == reflect.TypeOf((*MyStruct)(nil))`. Explain the difference.
