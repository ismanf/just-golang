# Generics — Fundamentals

## TL;DR

Generics arrived in **Go 1.18** as **type parameters** on functions and types: `func F[T any](x T) T`. Constraints (interfaces, now extended with type-set syntax `~T | T | ...`) restrict which types `T` can be. Methods cannot introduce new type parameters — only the type itself can. Under the hood, the compiler uses **GC shape stenciling**: types that share a memory layout share a single function body, identified by a runtime *dictionary*. This trades minor call overhead for tiny binary size compared to monomorphization.

## Mental Model

```
func Max[T cmp.Ordered](a, b T) T {
    if a > b { return a }
    return b
}

call site:               Max[int](3, 5)
compiler instantiates:   one body shared by all "pointer-shaped" Ts,
                         another by all "scalar 8-byte" Ts, etc.
runtime carries:         a dictionary {*type, method ptrs, comparators}
                         passed implicitly to each generic function
```

You write one function. The compiler emits a small number of shape-shared bodies. Each instantiation gets a dictionary that supplies type-specific operations (e.g., the right `Less` for `cmp.Ordered`).

## Syntax & Basic Usage

```go
package main

import (
	"cmp"
	"fmt"
)

// Generic function — single type parameter T constrained to cmp.Ordered.
func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

// Generic type.
type Pair[A, B any] struct {
	First  A
	Second B
}

func main() {
	fmt.Println(Max(3, 5))                      // T inferred as int
	fmt.Println(Max("apple", "banana"))         // T inferred as string
	fmt.Println(Max[float64](2.5, 1.5))         // T explicit

	p := Pair[string, int]{First: "x", Second: 1}
	fmt.Println(p)
	// Output:
	// 5
	// banana
	// 2.5
	// {x 1}
}
```

## Deep Dive

### Type parameter syntax

A type parameter list appears in square brackets between the function/type name and the regular parameter list:

```go
func F[T any](x T) T          // one parameter, "any" constraint
func G[K comparable, V any](m map[K]V) []K { /* ... */ }
type Container[T any] struct{ data []T }
```

Constraint names are interfaces (or interface literals, see next page). `any` is the unconstrained constraint.

### Type inference

The compiler can usually infer type parameters from arguments:

```go
Max(3, 5)        // infers T = int
slices.Sort(nums) // infers element type from the slice
```

Inference may fail when there's no argument to anchor `T`:

```go
func Zero[T any]() T { var z T; return z }
// Zero()       // INVALID — cannot infer T
Zero[int]()     // OK
```

Inference was significantly improved in Go 1.21+: it now handles assignability, constraint-driven inference, and inference across method calls more aggressively.

### Type parameters on methods? No.

```go
type Container[T any] struct{ items []T }

func (c *Container[T]) Add(v T) { c.items = append(c.items, v) }
// func (c *Container[T]) Map[U any](f func(T) U) []U { ... } // INVALID
```

Methods cannot declare *additional* type parameters. They inherit the receiver type's parameters and that's it. The workaround is a package-level generic function:

```go
func MapContainer[T, U any](c *Container[T], f func(T) U) []U {
	out := make([]U, len(c.items))
	for i, v := range c.items {
		out[i] = f(v)
	}
	return out
}
```

### Instantiation

A generic function/type is **instantiated** at each call site by binding type parameters to concrete types. The result behaves like a normal function/type:

```go
type IntPair = Pair[int, int]      // type alias to specific instantiation (since 1.24 with type parameters)
var p IntPair = Pair[int, int]{1, 2}
```

`IntPair` is a regular type from this point on.

### GC shape stenciling

The compiler does **not** generate a unique body per instantiation (that's monomorphization, à la C++/Rust). Instead, types that share a "GC shape" (size, pointerness, alignment) share a function body. The body receives an implicit runtime *dictionary* describing the concrete type's operations.

Practically:
- `Max[int]` and `Max[int64]` share a body on 64-bit (same shape).
- `Max[string]` is a separate body (2-word, contains a pointer).
- Dictionary lookup adds a small overhead vs hand-written code, on the order of nanoseconds.

This design keeps binary size small but means generics are typically a *touch* slower than hand-written equivalents in microbenchmarks. For 99% of code this is invisible; for tight loops over basic types, profile and consider specialization.

### Constraints are interfaces with extensions

```go
type Number interface {
	~int | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
	var s T
	for _, x := range xs {
		s += x
	}
	return s
}
```

The `~int` means "any type whose underlying type is `int`" — so `type Celsius int` satisfies `Number`. See the dedicated constraints page.

### Specifying constraints inline

```go
func Min[T interface{ ~int | ~float64 }](a, b T) T {
	if a < b {
		return a
	}
	return b
}
```

Equivalent to named constraints, just inline. Use named for readability when reused.

### Generic types with method requirements

```go
type Hasher[T any] interface {
	Hash() uint64
}

func Index[T Hasher[T]](xs []T) map[uint64]T { /* ... */ }
```

`T Hasher[T]` means "T must satisfy the `Hasher[T]` interface, parameterized by itself." This pattern enables "curiously recurring" generics for things like builders that return the receiver.

### Multiple type parameters

```go
func Map[K comparable, V, W any](m map[K]V, f func(V) W) map[K]W {
	out := make(map[K]W, len(m))
	for k, v := range m {
		out[k] = f(v)
	}
	return out
}
```

Order matters for inference: parameters that can be inferred from earlier args should come first.

### Generic type parameters on aliases (since 1.24)

```go
type IntSlice = []int      // simple alias (always existed)
type GenSlice[T any] = []T // parameterized alias (since 1.24)
```

Until 1.24, generic aliases were experimental behind `GOEXPERIMENT=aliastypeparams`.

## Standard Library Hooks

- `slices` (1.21+): `Sort`, `SortFunc`, `Min`, `Max`, `Contains`, `Index`, `Delete`, `Insert`, `Equal`, `Clone`, `Compact`, `Concat`, `Reverse`, `BinarySearch`, etc.
- `maps` (1.21+): `Clone`, `Copy`, `Keys`, `Values`, `Equal`, `EqualFunc`, `DeleteFunc`.
- `cmp` (1.21+): `Compare`, `Less`, `Or`, `Ordered` constraint.
- `sync.OnceValue`, `sync.OnceValues` (1.21+): lazy initialization with typed result.
- `sync.Map` is **not** generic (legacy API). Use a typed wrapper if you need type safety.
- `atomic.Pointer[T]`, `atomic.Int64`, `atomic.Bool`, etc. (1.19+) — typed atomic primitives.
- `iter.Seq[T]`, `iter.Seq2[K, V]` (1.23+) — generic iterator types for range-over-func.
- `unique.Handle[T]` (1.23+) — value interning.
- `weak.Pointer[T]` (1.24+) — weak references.

## Real-World Patterns

### 1. `slices.SortFunc` for custom ordering

```go
import (
	"cmp"
	"slices"
)

type Person struct {
	Name string
	Age  int
}

func main() {
	people := []Person{{"Alice", 30}, {"Bob", 25}, {"Cathy", 30}}
	slices.SortFunc(people, func(a, b Person) int {
		return cmp.Or(
			cmp.Compare(a.Age, b.Age),
			cmp.Compare(a.Name, b.Name),
		)
	})
}
```

`cmp.Or` chains compares lexicographically — your secondary sort key falls through when the primary ties.

### 2. Typed result memoization

```go
import "sync"

func Memoize[K comparable, V any](f func(K) V) func(K) V {
	var mu sync.Mutex
	cache := map[K]V{}
	return func(k K) V {
		mu.Lock()
		defer mu.Unlock()
		if v, ok := cache[k]; ok {
			return v
		}
		v := f(k)
		cache[k] = v
		return v
	}
}

slowSquare := Memoize(func(x int) int { /* expensive */ return x * x })
```

### 3. Generic option pattern

```go
type Option[T any] func(*T)

func With[T any, V any](field *V, value V) Option[T] {
	return func(t *T) { *field = value }
}

// Better: per-config functional options remain idiomatic; the truly generic
// version above is rarely worth the complexity. Most teams keep concrete options.
```

### 4. Result/Either type

```go
type Result[T any] struct {
	Value T
	Err   error
}

func Try[T any](fn func() (T, error)) Result[T] {
	v, err := fn()
	return Result[T]{Value: v, Err: err}
}
```

Idiomatic Go usually prefers multiple returns over `Result` types, but the pattern shows up in channel-based pipelines where carrying error + value through a single message is convenient.

### 5. Generic LRU cache (more in the data-structures page)

```go
type LRU[K comparable, V any] struct {
	cap   int
	items map[K]V
	// ... order tracking
}

func (l *LRU[K, V]) Get(k K) (V, bool) { /* ... */ }
func (l *LRU[K, V]) Put(k K, v V)      { /* ... */ }
```

Typed cache, no `any` boxing, no per-element allocation for the value.

## Anti-Patterns & Gotchas

**Over-generification.** If you only ever call `F[int]`, the function should not be generic. Resist the urge to "future-proof" with `[T any]`.

**Method type parameters that don't exist.** Compile error. If you need them, switch to a package-level function or design the receiver type differently.

**Implicit instantiation failure on `Zero[T]()`** when `T` cannot be inferred. Always provide explicit type arguments when there's no anchor.

**Forgetting `any` is shorthand for an empty interface.** `func F[T any](x T)` accepts anything — even values that don't satisfy what you actually need. Add a more specific constraint.

**Constraint with `~` everywhere "to be safe".** `~int` is for when callers might pass `type ID int`. If you don't expect that, omit `~`. Adding it later is a non-breaking change; removing it is breaking.

**Generic struct with unexported type params from another package.** You cannot use unexported types from other packages as type arguments. Use exported types or interfaces.

**Reaching for generics when an interface would do.** If you have one method, an interface is simpler and idiomatic. Generics shine when you want to preserve element types across operations (slices, maps, channels) or avoid boxing.

**Premature performance fears.** Generics in Go are roughly as fast as interface-based code, and *much* faster than `reflect`-based code. But they're slower than hand-specialized code in microbenchmarks. Profile, don't guess.

## Performance Notes

- Generic function calls have a small overhead vs hand-written: typically 1-5 ns from dictionary access, occasionally more for methods on interface constraints.
- No interface boxing on call site if `T` is a concrete type — the value is passed by value (or pointer) directly.
- GC-shape sharing limits code bloat. Binary growth from generics is typically minimal compared to monomorphization.
- The dictionary is passed implicitly; no observable allocation per call.
- Inlining is supported for many generic functions, especially trivial ones (`Min`, `Max`, simple accessors). Use `-gcflags='-m'` to verify.
- For ultimate speed (rare), the canonical answer is to write a specialized non-generic version alongside the generic one and pick at the call site.

## How Big Companies Use It

- **Standard library**: The `slices`/`maps`/`cmp` packages are the most ubiquitous generic code in production Go. Their adoption guided runtime tuning.
- **Cockroach** uses generics in `pkg/util/syncutil` for typed atomic pointers, and in batch-builder helpers.
- **Tailscale's `tsweb` and `syncs` packages** use generics for caches and lock-protected typed values: https://github.com/tailscale/tailscale/tree/main/syncs.
- **Caddy 2** moved several internal pools to typed `sync.Pool[T]` wrappers built on generics.
- **Discord's state service** post-mortem cited generics as one tool they used to type their hot paths without resorting to `any`.
- **`samber/lo`** is a popular (though sometimes overused) lodash-style generic helper library.

## Source Code References

Pinned to `go1.26`.

- Type parameter parsing: [`src/go/parser/parser.go`](https://github.com/golang/go/blob/master/src/go/parser/parser.go).
- Type checker for generics: [`src/go/types/instantiate.go`](https://github.com/golang/go/blob/master/src/go/types/instantiate.go).
- Compiler instantiation: [`src/cmd/compile/internal/typecheck/iexport.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/typecheck/iexport.go) — search "dictionary".
- Runtime dictionary support: [`src/cmd/compile/internal/reflectdata/reflect.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/reflectdata/reflect.go).
- `slices` implementations: [`src/slices/slices.go`](https://github.com/golang/go/blob/master/src/slices/slices.go), [`src/slices/sort.go`](https://github.com/golang/go/blob/master/src/slices/sort.go).
- `cmp` package: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go).

## Further Reading

- Spec, "Type parameters": https://go.dev/ref/spec#Type_parameters
- Go blog, "An Introduction To Generics": https://go.dev/blog/intro-generics
- Go blog, "Why generics?": https://go.dev/blog/why-generics
- "Type Parameters Proposal": https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md
- Go 1.18 release notes: https://go.dev/doc/go1.18
- Ian Lance Taylor talks on generics: https://www.youtube.com/results?search_query=ian+lance+taylor+generics+go
- "GC shape stenciling" design doc: https://go.googlesource.com/proposal/+/refs/heads/master/design/generics-implementation-gcshape.md
- PlanetScale blog, "Faster JSON parsing with generics": https://planetscale.com/blog (search "generics")

## Exercises / Self-Check

1. Write `Filter[T any](s []T, pred func(T) bool) []T`. Why doesn't `slices.Filter` exist in the standard library? (Hint: read the proposal discussion.)
2. Why can't you write `func (c *Container[T]) Map[U any](f func(T) U) []U`? Convert it to a package-level function.
3. Show that `func F[T any]() T` cannot be called without explicit type args. Add an argument to enable inference; what changed?
4. Build a benchmark comparing `slices.Sort` (generic) with the legacy `sort.Ints`. Where does the difference come from? Look at `-gcflags='-m'`.
5. Define a `Repository[T any]` interface with `Get`, `Put`, `Delete`. Why can't this be a method-set interface with generic methods?
