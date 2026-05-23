# When Not To Use Generics

## TL;DR

Russ Cox's rule: **"when in doubt, don't."** Generics in Go are a tool for preserving type information across operations on **containers, iterators, and algorithms over arbitrary element types**. They are *not* a substitute for interfaces (when you want behavior polymorphism), nor a way to reduce typing (Go prefers explicitness). Reach for generics when you'd otherwise box into `any`, code-generate per type, or duplicate identical code across types. Otherwise, leave them out.

## Mental Model

```
Ask three questions before adding [T any]:

  1. Do callers benefit from preserving the element type?
     (e.g., slices.Index returns the matching T, not any)
  
  2. Would the non-generic version use `any` and force callers to type-assert?
     (e.g., sync.Map's `Get` returns `any`; a generic wrapper avoids the cast)
  
  3. Are there two or more types I currently write identical code for?
     (e.g., min/max for int and float64)

If "no" to all three, you probably don't need generics.
```

The decision is rarely "is this elegant?" It's "does this preserve type information that the user cares about?"

## Syntax & Basic Usage — None

This page is conceptual. The "right" syntax is whichever of the alternatives below fits your case.

## Deep Dive

### Russ Cox's three rules (paraphrased from his 2022 GopherCon talk)

1. **If you find yourself writing the exact same code multiple times, where the only difference is the type, consider generics.**
2. **If your implementation must reference an element type (the "T" you'd parameterize) repeatedly, consider generics.**
3. **Otherwise, prefer interfaces.** Most polymorphism in Go is behavior-based; that's what interfaces are for.

### When generics are right

- **Containers** (`Stack[T]`, `Queue[T]`, `Set[T]`, `LRU[K,V]`).
- **Algorithms over arbitrary element types** (`slices.Sort`, `Map`, `Filter`, `Reduce`).
- **Numeric utilities** (`Min`, `Max`, `Clamp`, `Sum`, `Abs`).
- **Wrapping `any`-shaped APIs** (typed `sync.Pool`, typed `atomic.Value`, typed cache).
- **Functional helpers when type preservation matters** (`Memoize[K,V]`, `OnceValue[T]`).

### When generics are wrong

#### 1. You'd be better served by an interface

```go
// Smell: generic function that calls one method on T.
func Render[T interface{ Render() string }](items []T) []string {
	out := make([]string, len(items))
	for i, x := range items { out[i] = x.Render() }
	return out
}

// Cleaner: just take an interface slice.
type Renderer interface{ Render() string }
func Render(items []Renderer) []string {
	out := make([]string, len(items))
	for i, x := range items { out[i] = x.Render() }
	return out
}
```

The generic version "preserves T" but the caller doesn't care. If the only operation in the body is dispatching on `T`'s method, you wanted an interface.

#### 2. You only ever call it with one type

```go
func sumInts[T int](xs []T) T { /* ... */ }   // delete the [T int]
```

If `T` has exactly one instantiation, drop the parameter. YAGNI.

#### 3. You're using generics to fake operator overloading

```go
func Add[T int | string](a, b T) T { return a + b }
```

Now `Add` exists. Does `Add(1, 2)` add up faster than `1 + 2`? No, and you've added a dictionary lookup. This is a fun toy, not a useful tool.

#### 4. Constraint elaboration over real expressiveness

```go
type Number interface {
	~int | ~int8 | /* 12 more lines */
}
```

If every package defines its own `Number`, you've created friction without adding type safety. Centralize, or just use `cmp.Ordered` from stdlib (since 1.21).

#### 5. Code that was clearer with concrete types

```go
// Bad — looks "generic" but only inside the package.
func processFoo[T Foo](xs []T) { /* uses only Foo's methods */ }

// Better — concrete.
func processFoo(xs []Foo) { /* ... */ }
```

Generics let you preserve the type *across an API*. If callers always know the type, generics don't help them.

#### 6. Generics-as-templates for "code generation"

```go
type Map[K comparable, V any] struct {
	m map[K]V
	mu sync.RWMutex
}
```

Looks reasonable. But if you only ever instantiate `Map[string, int]` and `Map[uint64, *User]`, you've made the type harder to read for callers — every usage now reads `Map[string, int]` everywhere. Sometimes a concrete `UserMap` or `StringIntMap` is friendlier.

### The Russ Cox quote

From his 2022 GopherCon talk "Compatibility: How Go Programs Keep Working":

> "Write Go programs by writing code, not by defining types."

And from the proposal discussion:

> "Generics are a tool to be used when needed, not a tool to be used because they exist."

### A practical decision tree

```
Do you write the same code 2+ times for different types?
├── Yes → Consider generics.
└── No
    ├── Are callers forced to type-assert because you use `any`?
    │   ├── Yes → Consider generics.
    │   └── No → Use interfaces or concrete types.
    └── Do you need a container/iterator/algorithm to preserve T across calls?
        ├── Yes → Generics.
        └── No → Interfaces.
```

### What generics replaced (and didn't)

Generics killed:

- `gen`-style code-generation tools (`genny`, `gengo`) for typed containers.
- Most uses of `interface{}` in container packages.
- The `reflect`-based "convert this map to a slice of keys" pattern.

Generics did **not** kill:

- Standard interfaces (`io.Reader`, `error`, `fmt.Stringer`). These remain the right tool for behavior polymorphism.
- `any` in legacy APIs (`json.Marshal`, `fmt.Println`, `context.Value`) — these need true heterogeneity.
- Code generation for cases generics still can't express: variant types, struct-field projection, exhaustive matching.

### What generics still can't do (in 1.26)

- **No methods with their own type parameters.** Many functional-style APIs hit this wall.
- **No const generics.** Array lengths still cannot be parameterized.
- **No higher-kinded types.** You can't write `Functor[F[_], A, B]`.
- **No specialization.** Every instantiation goes through the same body; you cannot define an optimized version for `T = int`.
- **No "type-classes" beyond what type-set interfaces allow.**

If you find yourself wanting these, you probably want a different language or a code-gen step, not more clever generics.

## Standard Library Hooks

The Go team's own gating decisions on what was added to the standard library after generics:

- **Added:** `slices`, `maps`, `cmp` — containers and algorithms where type preservation was the whole point.
- **Added:** `sync.OnceValue[T]`, `atomic.Pointer[T]` — replace `any`-shaped APIs with typed ones.
- **Deliberately not added:** generic `sync.Map[K,V]`, generic `sync.Pool[T]` — these would touch core runtime/concurrency primitives and the team chose to wait. (Several proposals exist; none merged as of 1.26.)
- **Deliberately not added:** `slices.Filter`, `slices.Map`, `slices.Reduce`. Discussion: with the new iterator (range-over-func, 1.23), `iter.Seq`-based stream operations are preferred over eager slice operations. Watch the relevant proposals (https://go.dev/issue/61898).

## Real-World Patterns (When Not To)

### 1. Don't make every helper generic

```go
// Probably overgeneric:
func FirstOrZero[T any](s []T) T {
	if len(s) == 0 { var z T; return z }
	return s[0]
}

// Usually fine to just write the concrete version once per caller, or use slices.First-style helpers.
```

### 2. Don't wrap stdlib `any`-typed APIs preemptively

```go
// Smell: typed sync.Pool wrapper that the rest of the codebase never actually uses.
type Pool[T any] struct { /* ... */ }
```

Build the wrapper when you have **two or more call sites** that benefit. Until then, the `.(*T)` assertion at the use site is cheaper than another package to maintain.

### 3. Don't constrain by methods if an interface is already in play

```go
// Worse:
func Process[T interface{ Validate() error }](xs []T) error { /* ... */ }

// Better:
type Validatable interface{ Validate() error }
func Process(xs []Validatable) error { /* ... */ }
```

If the body never *creates* a `T` and never returns one, you don't need the type parameter.

### 4. Don't use generics to "fix" `error` handling

Functional-error patterns (`Result[T]`, `Either[L,R]`) are a frequent generic toy. They work in Go but fight every existing tool, library, and idiom. Go's `(value, error)` return tuple is the convention; stay with it unless you're prototyping or writing a heavily functional internal DSL.

### 5. Don't make APIs generic that callers will only use with `any`

```go
func Run[T any](fn func(T) error) error { /* ... */ }

Run(func(x any) error { /* ... */ }) // T=any anyway
```

If the only practical instantiation is `any`, drop the parameter and write `func(any) error`.

## Anti-Patterns & Gotchas (recap of "don'ts" from this page)

- Generics where an interface would do.
- Generics for a function called with one type.
- Generics that obscure the API for the sake of "future flexibility."
- Reinventing `cmp.Ordered` per package.
- Functional reduce/map chains in lieu of a `for` loop.
- Method-bag interfaces converted to generic constraints — same problems, less readable.
- Generic wrappers around `sync.Map`/`sync.Pool` written speculatively.

## Performance Notes

- Sometimes generics **lose** on raw performance vs hand-specialized code (e.g., a generic `Sum` over numeric types is slightly slower than `func SumInt(xs []int) int`). The difference is usually irrelevant; if it isn't, write the hand-specialized version alongside.
- Generics are reliably **faster** than `any` + assertion in hot paths — no boxing, no type-check per element.
- Generics produce slightly larger binaries than `interface{}`-based code but smaller than full monomorphization. Compared to interface-based, the binary growth is typically a few percent.
- If a generic version compiles to identical assembly to the hand-written (often true for trivial wrappers), there's no perf difference.

## How Big Companies Use It (Restraint Examples)

- **Standard library** stays conservative. `slices.Filter`/`Map`/`Reduce` were declined deliberately to avoid encouraging functional-pipeline code that hurts readability.
- **Tailscale's `tsweb` package** uses generics where they preserve type info (caches, atomic values) but not for general HTTP handlers — those stay interface-typed.
- **CockroachDB's code style guide** advises generics only when they replace boxing or repeated code; method-bag interfaces stay as interfaces.
- **Google's internal style guide** (per the public-facing `google/styleguide/go`) treats generics as "expert-level": OK in stdlib-like utility packages, discouraged in service code where concrete types are clearer.

## Source Code References

Pinned to `go1.26`.

- `slices` package — clean, minimal use of generics: [`src/slices/slices.go`](https://github.com/golang/go/blob/master/src/slices/slices.go).
- The `Filter`/`Map`/`Reduce` proposals (declined or pending): https://go.dev/issue/61898.
- `cmp` package — small, focused: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go).
- Counter-example: `container/list` still uses `any`. The team has not converted it; consensus is "if you need it typed, write your own."

## Further Reading

- Russ Cox, "When To Use Generics" talk (GopherCon 2022): https://go.dev/blog/when-generics (companion blog).
- "When To Use Generics" blog: https://go.dev/blog/when-generics.
- Ian Lance Taylor, "Generics — proposal and design": https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md.
- Robert Griesemer talks on type parameters: https://www.youtube.com/results?search_query=griesemer+generics.
- Dave Cheney, "Why generics?" (skeptic-friendly): https://dave.cheney.net/2018/11/12/go-2-and-generics.
- Go FAQ — historical "Why doesn't Go have generics?" entry (now mostly historical): https://go.dev/doc/faq.

## Exercises / Self-Check

1. Take a generic function you've recently written. Try the interface-based alternative. Which reads better at the call site?
2. Find a piece of code in your project that uses `any` + type assertion in a hot path. Would generics remove the assertion without adding complexity?
3. Read the `slices` package source. Identify three functions where the team chose generics and two operations they declined. What's the pattern?
4. Re-implement a `Filter` function generically, then with an iterator (`iter.Seq[T]`, 1.23+). Which fits Go's idioms better, and why was no `slices.Filter` added to stdlib?
5. Convert a generic API in your codebase that has exactly one instantiation back to concrete. Does the codebase get clearer?
