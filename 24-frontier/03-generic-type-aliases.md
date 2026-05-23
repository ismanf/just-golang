# Generic Type Aliases (since Go 1.24)

## TL;DR

Type aliases (`type X = Y`) have existed since Go 1.9, but **only for non-generic types**. Since **Go 1.24** (proposal [#46477](https://github.com/golang/go/issues/46477)), you can alias **generic types** with their own type parameters: `type Set[T comparable] = map[T]struct{}`. This closes a long-standing gap that forced ugly workarounds — defining wrapper types just to give a friendly name to a parameterized type. The single biggest gotcha: **a generic alias is still an alias, not a new type**. You can't define methods on it (methods require a distinct type, not an alias), and types unify through aliases — `Set[int]` and `map[int]struct{}` are *the same type*, not similar ones.

## Mental Model

```
   Plain alias (since 1.9):
       type ID = string                     // ID and string are identical types
       
   Generic alias (since 1.24):
       type Set[T comparable] = map[T]struct{}
       
       Set[int]    == map[int]struct{}        (same type)
       Set[string] == map[string]struct{}     (same type)
       
   Non-alias generic type definition (existed since 1.18):
       type Set[T comparable] map[T]struct{}  // distinct from map[T]struct{}
       
       Set[int] != map[int]struct{}            (different types; can have methods)
```

The alias gives you a shorter name; the definition gives you a *new type* you can hang methods on. Different tools for different jobs.

## Syntax & Basic Usage

```go
package main

import "fmt"

// Generic alias
type Pair[A, B any] = struct {
	First  A
	Second B
}

// Generic alias for stdlib generic type
type Set[T comparable] = map[T]struct{}

// Using them:
func main() {
	p := Pair[int, string]{First: 1, Second: "hi"}
	fmt.Println(p)

	s := Set[int]{1: {}, 2: {}, 3: {}}
	fmt.Println(s)

	// Same underlying type — interchangeable:
	var raw map[int]struct{} = s
	_ = raw
}
```

`Pair[int, string]` and `struct{First int; Second string}` are identical. The alias is "syntactic sugar" only.

## Deep Dive

### Why this took so long

Generics shipped in 1.18. Type aliases shipped in 1.9. Generic aliases — the natural intersection — were *deliberately deferred* because:

1. **The 1.18 generics design** focused on declaring new types. Aliases were considered nice-to-have.
2. **The grammar was tricky**: a parser had to distinguish `type X[T] = ...` (alias) from `type X[T] ...` (new type).
3. **Type-identity rules** needed careful spec work for aliases that name parameterized types.
4. **Cross-package alias behavior** with generic parameters needed test coverage.

The proposal was open from 2021; 1.24 (Feb 2025) finally shipped it. Russ Cox blogged about the unfinished business of generic aliases in 2022 — three years later, it's done.

### Aliases vs definitions

```go
// Alias — same type
type Set[T comparable] = map[T]struct{}

// Definition — new type
type Set[T comparable] map[T]struct{}
```

| Aspect | Alias | Definition |
|---|---|---|
| Same underlying type? | yes (literally same) | new type |
| Can define methods? | no (alias) | yes |
| Interchangeable with underlying? | yes | no |
| Type identity in interfaces | matches underlying | matches the new type |
| Use case | shorter names, gradual migration | new abstractions with behavior |

### When to use alias

1. **Migration**: rename a type without breaking callers.
   ```go
   // old name kept as alias during transition
   type OldName[T any] = NewName[T]
   ```

2. **Friendly names** for stdlib generics:
   ```go
   type Slice[T any] = []T
   type Map[K comparable, V any] = map[K]V
   ```
   (Marginally useful; rarely worth the indirection.)

3. **Re-exporting types** from one package via another:
   ```go
   // in package pubapi
   import "myorg/internal/impl"
   type Service[T any] = impl.Service[T]
   ```
   Callers import `pubapi.Service[T]`; the actual type lives in `impl.Service[T]`.

4. **Constraint shortcuts**:
   ```go
   type Hashable = comparable  // not generic but conceptually similar
   ```

### When to use definition

1. **Adding methods** to a parameterized type.
2. **Distinct identity** — preventing accidental conflation.
3. **Domain modeling** where the new type represents a concept (Set[T] as a set, not a map).

### Aliases CAN'T have methods

```go
type Set[T comparable] = map[T]struct{}

// COMPILE ERROR: cannot define new methods on non-local type map
func (s Set[T]) Add(v T) { s[v] = struct{}{} }
```

To get methods, use a definition:

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(v T)     { s[v] = struct{}{} }
func (s Set[T]) Contains(v T) bool { _, ok := s[v]; return ok }
```

### Aliases and type parameters

The alias declares its own type parameters; they may differ in count or constraint from what the right-hand side actually has:

```go
type Reduce[T any] = []T  // T is the element type
```

```go
type Pair2[A any] = Pair[A, A]  // re-parameterize Pair to one type
```

Pair2[int] is the same as Pair[int, int].

You can also bind parameters partially via composition:

```go
type StringSet = Set[string]                     // fully specialized; no params
type IntPair = Pair[int, int]
```

These aliases are not generic; just shorthand.

### Cross-package aliases

```go
// pkg/internal/big.go
package internal
type BigThing[T any] struct{ /* ... */ }

// pkg/public/api.go
package public
import "myorg/pkg/internal"
type Thing[T any] = internal.BigThing[T]
```

External users see `public.Thing[int]`. The underlying type is `internal.BigThing[int]`. They're identical.

This is the canonical use case in 1.24+: hide implementation packages while re-exporting types.

### Alias chains

```go
type A[T any] = []T
type B[T any] = A[T]
type C[T any] = B[T]
```

`C[int]` and `[]int` are the same type. Alias chains compress to one.

### Interaction with reflection

`reflect.TypeOf(make(Set[int]))` returns `map[int]struct{}` because Set[int] *is* map[int]struct{}. There's no separate reflect representation for the alias.

Definitions are different:
```go
type Set[T comparable] map[T]struct{}
reflect.TypeOf(Set[int]{})  // pkg.Set[int]
```

### Methods through type definitions on aliased types — possible after a layer

You can't add methods to an alias *directly*, but you can add them to the underlying type if you own it:

```go
package mypkg

type Inner[T any] struct{ X T }
func (i *Inner[T]) Get() T { return i.X }

// External alias for re-export
type Outer[T any] = Inner[T]

// callers can still invoke methods on Outer:
o := &Outer[int]{X: 42}
_ = o.Get()
```

Methods belong to `Inner[T]`; through alias, they're accessible via `Outer[T]`.

### Embedding constraints

You can embed generic aliases in other types:

```go
type WithLog[T any] struct {
    Set[T]              // embed generic alias
    log slog.Logger
}
```

But: since Set is `map[T]struct{}`, embedding it doesn't give you methods unless Set is a definition (with methods).

### Common stdlib usages

As of Go 1.26+:
- `slices.SortFunc[E any]` accepts `func(a, b E) int` — no aliases used internally, but library authors can now alias for clarity.
- `sync.Map` is not generic (still); future plans may use generic aliases for ergonomics.
- `maps.Values[K comparable, V any](m map[K]V) iter.Seq[V]` — no alias, but in principle could.

### Migration strategy

Renaming a generic type without breaking users:

```go
// Old version
package mypkg
type Result[T any] struct { /* ... */ }

// New version: rename to Outcome, keep Result as alias
package mypkg

type Outcome[T any] struct { /* ... */ }
type Result[T any] = Outcome[T]  // backward compat
```

External code calling `mypkg.Result[X]` still works; new code uses `mypkg.Outcome[X]`. Drop `Result` in a major version bump.

### Constraints in alias parameters

```go
type Numeric[T int | int64 | float64] = T

// Use:
var x Numeric[int] = 42
```

The constraint applies. Compile error if you try `Numeric[string]`.

### Forward declarations / cycles

Aliases can reference each other within limits:

```go
type A[T any] = B[T]    // OK if B is defined
type B[T any] struct{ Val T }
```

Direct cycles are an error:

```go
type X[T any] = X[T]    // compile error: cycle
```

### Use in interfaces

```go
type Container[T any] interface {
    Add(T)
    Contains(T) bool
}

type IntContainer = Container[int]   // alias to specialized interface
```

`IntContainer` is the same interface as `Container[int]`.

## Standard Library Hooks

The 1.24 release notes mention generic aliases. Stdlib usage is being added gradually as packages adopt the feature.

- `iter.Seq`, `iter.Seq2`: not yet aliased; could be.
- `slices`, `maps`: their helper signatures could be sugared with aliases.
- New stdlib helpers (1.26+): may introduce aliases for ergonomics.

## Real-World Patterns

### 1. Re-export from internal package

```go
// internal/store/store.go
package store
type Repository[T any] struct { /* ... */ }

// pkg/api/store.go
package api
import "myorg/internal/store"
type Repository[T any] = store.Repository[T]
```

Users import `api.Repository[T]`; can't import `internal/...`.

### 2. Friendly name for verbose generic

```go
type Result[T any] = struct {
    Value T
    Err   error
}

// vs
type Result[T any] struct {
    Value T
    Err   error
}
```

Alias version is interchangeable with a literal struct; definition version has its own identity. Choose based on whether you want methods.

### 3. Specialize a generic for common case

```go
type StringSet = Set[string]    // pre-specialized; no params at use site
type IntSet = Set[int]

// Use:
var s StringSet = StringSet{}
s["alice"] = struct{}{}
```

### 4. Gradual rename

```go
// Phase 1: introduce new name
type NewName[T any] struct{ /* ... */ }
type OldName[T any] = NewName[T]

// Phase 2 (next major version): drop OldName
type NewName[T any] struct{ /* ... */ }
```

Callers migrate at their own pace.

### 5. Compose constraints

```go
type Ordered = interface{ int | int64 | float32 | float64 | string }
type Container[T Ordered] struct{ items []T }

// Alternative without alias:
type Container[T int | int64 | float32 | float64 | string] struct{ items []T }
```

Alias makes the constraint reusable.

## Anti-Patterns & Gotchas

**Trying to add methods to an alias.** Compile error; switch to definition.

**Aliasing for the sake of it.** Each alias adds a name; readers must learn it. `Set[T] = map[T]struct{}` saves three characters and costs vocabulary.

**Long alias chains.** `A → B → C → D` reads awkwardly. Flatten when possible.

**Aliases that change semantics.** Reordering type parameters in an alias is technically legal but confusing.

**Re-exporting alias in many packages.** Two packages aliasing the same internal type might be intentional, or might indicate broken visibility design.

**Expecting type identity to differ.** `Set[int]` *is* `map[int]struct{}`. Code that depends on them being distinct fails.

**Using aliases to "fix" import cycles.** Doesn't help; an alias still depends on the aliased package.

**Generic aliases for primitive types.** `type Int = int` is the old non-generic form; `type IntSlice = []int` is fine but not generic. Generic aliases shine only with parameters.

## Performance Notes

Aliases are compile-time only. Zero runtime cost; the compiler resolves them to underlying types.

Compile times: aliases add no notable overhead. The type checker resolves them as it goes.

## How Big Companies Use It

1.24 is recent (Feb 2025). Adoption examples are early:

- **The Go team** discussed using aliases in stdlib for sugar across `iter`, `slices`, `maps`.
- **Kubernetes**: migration paths for legacy generic-by-hand patterns could use aliases.
- **gRPC-Go**: discussed for naming long generic types in the codegen output.
- **Library authors generally**: re-export from `internal/` becomes cleaner.

Production posts are still light; this is genuinely new territory.

## Source Code References

Pinned to `go1.26`.

- Spec change (type aliases with type parameters): https://go.dev/ref/spec#Alias_declarations.
- Proposal #46477: https://github.com/golang/go/issues/46477.
- Type checker handling: [`src/go/types/`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/types).
- Type identity rules: [`src/cmd/compile/internal/types2/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/types2).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Go 1.24 release notes (alias section): https://go.dev/doc/go1.24.
- Proposal #46477: https://github.com/golang/go/issues/46477.
- "Generic type aliases" — design discussion: golang/proposal repo.
- "Type aliases in Go" — Go blog (1.9 era, for context): https://go.dev/blog/type-aliases.
- Russ Cox, "Generics — unfinished business" (2022): https://research.swtch.com.

## Exercises / Self-Check

1. Declare `type Pair[A, B any] = struct{ First A; Second B }`. Verify `Pair[int, string]{}` is interchangeable with `struct{First int; Second string}{}`.
2. Try defining a method on `Pair[A, B]`. Why does it fail?
3. Use a generic alias to re-export an `internal` type. Confirm callers can use it as if it lived in your public package.
4. Rename a generic type while keeping the old name as a deprecation alias. Demonstrate the migration with a fake caller.
5. When would you choose `type X[T any] = Y[T]` over `type X[T any] Y[T]`? Articulate the engineering rule.
