# Constraints and Type Sets

## TL;DR

A **constraint** in Go generics is just an interface — but interfaces were extended in 1.18 to describe **type sets**: not only "any type with these methods" but also "any type whose underlying type is in this list." The new syntax adds two operators: `|` (union of types) and `~T` ("any type whose underlying type is `T`"). Predefined: `any` (no constraint), `comparable` (supports `==`), and `cmp.Ordered` (since 1.21) for ordering. Use named constraints for reuse; inline for one-offs.

## Mental Model

```
A constraint is an interface. An interface describes a TYPE SET.

Pre-1.18:
    interface { Read([]byte) (int, error) }
    Set = "all types whose method set contains Read([]byte)(int,error)"

Since 1.18 (constraint-only syntax):
    interface { ~int | ~int64 | ~float64 }
    Set = "all types whose underlying type is int, int64, or float64"

    interface {
        ~string                          // type-set element
        Len() int                        // method requirement
    }
    Set = INTERSECTION of:
        - types whose underlying type is string
        - types with a Len() int method
```

Type sets generalize "interface satisfaction." A type satisfies the constraint iff it's in the set.

## Syntax & Basic Usage

```go
package main

import (
	"cmp"
	"fmt"
)

// Predefined "any" — no restriction.
func Identity[T any](x T) T { return x }

// "comparable" — supports == and !=
func Equal[T comparable](a, b T) bool { return a == b }

// "cmp.Ordered" — supports <, <=, >, >=, ==, !=
func Max[T cmp.Ordered](a, b T) T {
	if a > b { return a }
	return b
}

// Inline type-set constraint.
func Sum[T interface{ ~int | ~float64 }](xs []T) T {
	var s T
	for _, x := range xs { s += x }
	return s
}

// Named constraint, reusable.
type Number interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
	~float32 | ~float64
}

func Avg[T Number](xs []T) float64 {
	var s T
	for _, x := range xs { s += x }
	return float64(s) / float64(len(xs))
}

func main() {
	type Celsius float64
	temps := []Celsius{20, 22, 25}
	fmt.Println(Avg(temps)) // 22.333...
	// Output:
	// 22.333333333333332
}
```

`Celsius`'s underlying type is `float64`, so `~float64` catches it. Without `~`, only `float64` itself would qualify.

## Deep Dive

### The `~` (tilde) operator

```go
type ID int
type Number interface { int | int64 }       // ID does NOT satisfy this
type Number2 interface { ~int | ~int64 }    // ID DOES satisfy this
```

Without `~`, the constraint matches only the listed types exactly. With `~T`, it matches any defined type whose underlying type is `T`. Almost all real-world constraints want `~`.

### The `|` (union) operator

```go
type Signed interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}
```

A union of type-set elements. The compiler computes the union and uses it to validate operations:

- Arithmetic operators (`+`, `-`, `*`, `/`): allowed if **all** members of the set support them.
- Comparison operators: same — all members must support.
- The actual operation at runtime is dispatched via the dictionary.

### `comparable`

Predeclared. Means the type supports `==` and `!=`. Includes all numeric, bool, string, pointer, channel, interface-with-comparable-dynamic-types, and structs/arrays of comparable elements.

```go
func Contains[T comparable](s []T, x T) bool {
	for _, v := range s { if v == x { return true } }
	return false
}
```

Note: until Go 1.20, you could not use `comparable` as a constraint argument to a parameter typed as a non-`comparable` interface. The rules loosened in 1.20+ — see [the proposal](https://go.dev/issue/56548).

### `cmp.Ordered` (since 1.21)

```go
package cmp

type Ordered interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
	~float32 | ~float64 |
	~string
}
```

Defined in [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go). Use it whenever you'd otherwise write a "Number-or-String" constraint by hand.

### Intersection (multiple elements in one interface)

```go
type Stringy interface {
	~string
	Len() int          // (illustrative — basic types don't have methods)
}
```

Multiple lines in an interface form an intersection. To satisfy it, a type must be in **all** of the listed type sets and have **all** the listed methods. Some intersections are empty (e.g., requiring both `~int` and a method — basic types have no methods), which means no type can satisfy them and the constraint is useless. The compiler doesn't reject impossible constraints by themselves, but no concrete type will instantiate them.

### Methods on constrained types

```go
type Lener interface {
	Len() int
}

func TotalLen[T Lener](xs []T) int {
	total := 0
	for _, x := range xs { total += x.Len() }
	return total
}
```

A method-set constraint works exactly like a regular interface — the difference is that the type parameter is checked at compile time per instantiation.

### Self-referential constraints

```go
type Lesser[T any] interface {
	Less(T) bool
}

func Sort[T Lesser[T]](s []T) { /* ... */ }
```

`T` is constrained to "any type that has a `Less(T) bool` method comparing against itself." This is how `slices.SortStableFunc` etc. could be written before `cmp` showed up.

### `constraints` package — historical

`golang.org/x/exp/constraints` (and the brief `constraints` proposal) predates the stdlib `cmp.Ordered`. Most of what you used to import from `constraints.Ordered` is now in `cmp.Ordered`. The `Signed`/`Unsigned`/`Integer`/`Float` aliases still live in `x/exp/constraints`; for stdlib use, define your own.

### Constraint type inference

```go
func Min[T cmp.Ordered](a, b T) T { ... }

Min(1, 2)         // T=int inferred
Min(1, 2.0)       // ERROR: ambiguous — int vs float64
Min[float64](1, 2.0) // OK
```

Mixed-type literals don't infer well. Annotate explicitly when needed.

### Constraint syntax inside generic types

```go
type SortedMap[K cmp.Ordered, V any] struct { /* ... */ }
```

Same rules apply.

### `any` is `interface{}` is the empty type set's complement

`any` is the interface with the empty method set and no type-set elements — every type satisfies it.

## Standard Library Hooks

- `cmp.Ordered` — the workhorse numeric/string constraint (1.21+).
- `comparable` — predeclared.
- `any` — predeclared, alias for `interface{}`.
- `cmp.Compare`, `cmp.Less`, `cmp.Or` — work with `Ordered`.
- `slices` package — sort/binary-search functions use `cmp.Ordered`.
- `golang.org/x/exp/constraints` — historical; `Signed`, `Unsigned`, `Integer`, `Float`, `Complex` aliases.

## Real-World Patterns

### 1. Roll-your-own `Number` constraint

```go
type Numeric interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
	~float32 | ~float64
}

func Clamp[T Numeric](x, lo, hi T) T {
	if x < lo { return lo }
	if x > hi { return hi }
	return x
}
```

Don't include `~complex64 | ~complex128` unless you actually want complex numbers — they don't support `<`.

### 2. Constraint with method requirement

```go
type Identifiable interface {
	ID() string
}

func IndexByID[T Identifiable](items []T) map[string]T {
	out := make(map[string]T, len(items))
	for _, it := range items {
		out[it.ID()] = it
	}
	return out
}
```

### 3. Curiously recurring (self-referential) for builders

```go
type Builder[T any] interface {
	Reset() T  // returns the receiver type, typed
}

func ResetAll[T Builder[T]](bs []T) {
	for _, b := range bs { b.Reset() }
}
```

`T Builder[T]` says "T must have a `Reset() T` method." Concrete types `type Q struct{}; func (q *Q) Reset() *Q { ... }` instantiate as `T = *Q`.

### 4. Restricting to comparable for set behavior

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(v T)       { s[v] = struct{}{} }
func (s Set[T]) Contains(v T) bool { _, ok := s[v]; return ok }

func NewSet[T comparable](items ...T) Set[T] {
	s := make(Set[T], len(items))
	for _, it := range items { s.Add(it) }
	return s
}
```

### 5. Generic comparison helper

```go
import "cmp"

func MinBy[T any, K cmp.Ordered](xs []T, key func(T) K) T {
	if len(xs) == 0 { var z T; return z }
	min := xs[0]
	mk := key(min)
	for _, x := range xs[1:] {
		k := key(x)
		if k < mk {
			mk, min = k, x
		}
	}
	return min
}

oldest := MinBy(people, func(p Person) int { return p.BirthYear })
```

## Anti-Patterns & Gotchas

**Forgetting `~`.** Your constraint accepts `int` but rejects `type ID int`. Almost always you want `~`.

**`comparable` interface trap pre-1.20.** Older Go versions rejected `T comparable` in some valid contexts. Update Go and the issue evaporates.

**Including `complex64`/`complex128` in `~int | ... | ~complex128`** then using `<`. Complex numbers don't have an order. The compiler will reject the operation only when you write it inside the generic body.

**Defining a constraint with both a type set and a method requirement on a basic type.** Basic types have no methods — the intersection is empty. Either drop the type-set element or restructure (require the method, accept any type that has it).

**Re-deriving `cmp.Ordered` instead of importing it.** Use the stdlib.

**Inline constraints everywhere.** Repetitive. Define a named constraint when reused.

**Using `any` and then `reflect`-ing inside the body.** That's not generic programming, that's runtime introspection in disguise. Tighten the constraint.

**Trying to write `T comparable` then doing `<` on `T`.** `comparable` permits `==`/`!=` only. Use `cmp.Ordered` for ordering.

## Performance Notes

- Constraints exist at compile time; they have no runtime cost beyond the dictionary mechanism (covered in the fundamentals page).
- Method-bearing constraints add itab-style dictionaries with method pointers. Calls through them are roughly as fast as calls through interfaces.
- Type-set constraints (`~int | ~int64`) allow direct operations — no boxing, no method dispatch. The compiler emits per-shape bodies that operate on the raw bits.
- Adding `~` does not affect performance; it just widens the set at compile time.
- The runtime cost of generics scales with constraint complexity only in the sense that more elaborate constraints generate more elaborate dictionaries; in steady-state hot code, the difference is negligible.

## How Big Companies Use It

- **Standard library** is the largest user. `slices.Sort` uses `cmp.Ordered`; `slices.SortFunc` uses `any`; `maps.Equal` uses `comparable` for keys and `comparable` for values (or `EqualFunc` for non-comparable).
- **Google's internal codebases** (per Ian Lance Taylor's talks) standardized on a small set of constraints, importing them from a central package rather than redeclaring locally.
- **Tailscale** uses `cmp.Ordered`-backed generic helpers in their `tsweb` and `syncs` packages.
- **PlanetScale and others** writing high-perf SQL drivers use type-set constraints to avoid `any` boxing in column-encoding hot paths.
- **`samber/lo`** is the most widely-imported third-party library to use generic constraints heavily — useful as a real-world catalog of patterns (and over-patterns).

## Source Code References

Pinned to `go1.26`.

- `cmp.Ordered` definition: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go).
- `comparable` builtin: [`src/builtin/builtin.go`](https://github.com/golang/go/blob/master/src/builtin/builtin.go).
- Type-set / constraint type checking: [`src/go/types/typeset.go`](https://github.com/golang/go/blob/master/src/go/types/typeset.go).
- Union type implementation: [`src/go/types/union.go`](https://github.com/golang/go/blob/master/src/go/types/union.go).
- Constraint enforcement on operators: [`src/go/types/expr.go`](https://github.com/golang/go/blob/master/src/go/types/expr.go).
- `golang.org/x/exp/constraints` (historical): https://pkg.go.dev/golang.org/x/exp/constraints.

## Further Reading

- Spec, "Interface types — Type sets": https://go.dev/ref/spec#Interface_types
- Spec, "Type constraints": https://go.dev/ref/spec#Type_constraints
- Go blog, "An Introduction To Generics" (covers constraints): https://go.dev/blog/intro-generics
- "Type Parameters Proposal" sections on type sets: https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md
- Go 1.21 release notes on `cmp` and `slices`: https://go.dev/doc/go1.21
- Robert Griesemer's talks on type parameters: https://www.youtube.com/watch?v=TborQFPY2IM
- "Constraints and the underlying type" blog: search go.dev/blog for "underlying type"

## Exercises / Self-Check

1. Define a `Stringer` constraint (the methodful one) and a `~string` constraint (the type-set one). Why can't a type satisfy both at once unless you contrive it?
2. Write `Sum[T cmp.Ordered](xs []T) T`. Does it compile? Why not? (Hint: `cmp.Ordered` allows `<` but `Sum` needs `+`.)
3. Build a constraint `Numeric` that accepts only signed numerics. Test that `uint8` fails to instantiate.
4. Why does `func F[T comparable](x T) { return x < x }` fail to compile?
5. Use `~` to make your `Money` (`type Money int64`) usable with a generic `Sum`. Show what happens when you drop the `~`.
