# Functional Go — How Far You Can Push It

## TL;DR

Go is not a functional language, but it has **first-class functions, closures, and (since 1.18) generics** — enough to express many functional patterns: `Map`, `Filter`, `Reduce`, function composition, currying. Since 1.23, **range-over-func** and `iter.Seq` make lazy-evaluation pipelines first-class. The single biggest gotcha: **the Go standard library deliberately doesn't ship `slices.Map`/`slices.Filter`/`slices.Reduce`** — the team's stance, articulated by Russ Cox and others, is that *explicit `for` loops are clearer and more idiomatic*. If you write a functional pipeline, you're swimming against the language's current; usually the right call is a plain loop. Reserve functional patterns for genuinely declarative scenarios (lazy iteration, function-as-data, composition of small transforms).

## Mental Model

```
   Imperative Go (idiomatic):
   
   var result []int
   for _, x := range xs {
       if x % 2 == 0 {
           result = append(result, x*x)
       }
   }

   Functional Go (works, but not idiomatic):
   
   result := Map(Filter(xs, isEven), square)

   Lazy / iterator-based (1.23+):
   
   for v := range xform(xs) {
       result = append(result, v)
   }
   // where xform produces values one at a time via iter.Seq
```

The functional style is more concise; the imperative style is more readable for most Go programmers — and importantly more profile-friendly (no closure allocation, no function-call indirection).

## Syntax & Basic Usage

Generic helpers (1.18+):

```go
package fp

func Map[T, U any](in []T, f func(T) U) []U {
	out := make([]U, len(in))
	for i, v := range in {
		out[i] = f(v)
	}
	return out
}

func Filter[T any](in []T, pred func(T) bool) []T {
	out := in[:0:0]
	for _, v := range in {
		if pred(v) {
			out = append(out, v)
		}
	}
	return out
}

func Reduce[T, U any](in []T, initial U, f func(U, T) U) U {
	acc := initial
	for _, v := range in {
		acc = f(acc, v)
	}
	return acc
}
```

Usage:

```go
nums := []int{1, 2, 3, 4, 5}
doubled := Map(nums, func(x int) int { return x * 2 })
evens := Filter(nums, func(x int) bool { return x%2 == 0 })
sum := Reduce(nums, 0, func(a, b int) int { return a + b })
```

Works, but compare:

```go
doubled := make([]int, 0, len(nums))
for _, x := range nums { doubled = append(doubled, x*2) }
```

For most Go developers, the loop is faster to read and write.

## Deep Dive

### Why Go doesn't ship `Map`/`Filter` in stdlib

The Go team has discussed this many times. Russ Cox's position (paraphrased from various golang-nuts threads):

- A `for` loop is universal; everyone reads them.
- `Map(xs, f)` requires the reader to: (1) know the helper exists, (2) trust it's correct, (3) hold the helper's implementation in mind to reason about it.
- The for loop has no hidden allocation; `Map` allocates an output slice.
- Adding `slices.Map` would suggest "this is the One True Way", which the team doesn't endorse for transformations.

`slices` (since 1.21) does have helpers — `slices.Contains`, `slices.Index`, `slices.Sort`, `slices.Reverse` — but they're not transformations. They're operations the loop can't trivially express.

### What you DO get in the stdlib

#### Function values

```go
type Pred func(int) bool

isEven := func(x int) bool { return x%2 == 0 }
```

Pass functions as arguments; assign to variables.

#### Closures

```go
func counter() func() int {
    n := 0
    return func() int { n++; return n }
}

c := counter()
c() // 1
c() // 2
```

#### Higher-order functions in stdlib

- `sort.Slice(s, less)` — pass a comparator.
- `slices.SortFunc(s, cmp)` — same.
- `slices.IndexFunc(s, pred)`.
- `cmp.Or(...)` — first non-zero.

#### `iter.Seq` and range-over-func (1.23+)

```go
import "iter"

func Filter[T any](s []T, pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for _, v := range s {
            if pred(v) {
                if !yield(v) { return }
            }
        }
    }
}

func Map[T, U any](s iter.Seq[T], f func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range s {
            if !yield(f(v)) { return }
        }
    }
}

// Usage:
nums := []int{1, 2, 3, 4, 5}
for v := range Map(Filter(slices.Values(nums), isEven), square) {
    fmt.Println(v)
}
```

Lazy: no intermediate slices. Each yielded value flows through filter then map. Range-over-func is THE big functional-shaped addition to Go in 1.23.

### Currying and partial application

```go
func Curry[A, B, C any](f func(A, B) C) func(A) func(B) C {
    return func(a A) func(B) C {
        return func(b B) C {
            return f(a, b)
        }
    }
}

add := func(a, b int) int { return a + b }
add5 := Curry(add)(5)
result := add5(3) // 8
```

Mechanically possible; almost never useful. Idiomatic Go uses a closure directly:

```go
add5 := func(b int) int { return 5 + b }
```

### Function composition

```go
func Compose[A, B, C any](g func(B) C, f func(A) B) func(A) C {
    return func(a A) C { return g(f(a)) }
}

double := func(x int) int { return x * 2 }
addOne := func(x int) int { return x + 1 }
doubleThenAddOne := Compose(addOne, double)
doubleThenAddOne(5) // 11
```

Rare in Go. `iter.Seq` chaining is the more natural form.

### Immutable data

Go has no built-in immutability. Convention: don't expose setters; return new values rather than mutating.

```go
type Point struct { X, Y int }

func (p Point) Translate(dx, dy int) Point {
    return Point{X: p.X + dx, Y: p.Y + dy}
}
```

Value receivers + return-by-value gives copy-semantics. Good for small structs; expensive for large ones (slices, maps).

### Tail-call optimization

Go does NOT do tail-call optimization. Recursive solutions blow the stack:

```go
func sum(xs []int) int {
    if len(xs) == 0 { return 0 }
    return xs[0] + sum(xs[1:])
}
// Works for small slices; stack-overflow for huge ones.
```

Convert recursion to iteration in Go. Always.

### Memoization

Cache results of pure functions:

```go
func Memoize[K comparable, V any](f func(K) V) func(K) V {
    cache := map[K]V{}
    var mu sync.RWMutex
    return func(k K) V {
        mu.RLock()
        if v, ok := cache[k]; ok { mu.RUnlock(); return v }
        mu.RUnlock()
        mu.Lock()
        defer mu.Unlock()
        if v, ok := cache[k]; ok { return v }  // double-check
        v := f(k)
        cache[k] = v
        return v
    }
}

slowFib := func(n int) int { /* recursive Fibonacci */ return 0 }
fib := Memoize(slowFib)
```

Useful for expensive pure computations. For caching across services, use a real cache.

### Streams via channels

Go has channels — natural for streaming pipelines:

```go
func Range(start, end int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := start; i < end; i++ {
            out <- i
        }
    }()
    return out
}

func MapChan[T, U any](in <-chan T, f func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for v := range in {
            out <- f(v)
        }
    }()
    return out
}

// Usage:
for v := range MapChan(Range(0, 10), func(x int) int { return x * x }) {
    fmt.Println(v)
}
```

Pre-1.23 this was the way to do lazy pipelines. Post-1.23, `iter.Seq` is cheaper (no goroutine per stage). Channels remain useful for cross-goroutine streaming.

### Functional error handling

Imperative Go: `if err != nil { return err }` everywhere.

"Functional" alternative: `Result[T]` types via generics.

```go
type Result[T any] struct {
    Value T
    Err   error
}

func Try[T any](f func() (T, error)) Result[T] {
    v, err := f()
    return Result[T]{v, err}
}

func (r Result[T]) Map(f func(T) T) Result[T] {
    if r.Err != nil { return r }
    return Result[T]{f(r.Value), nil}
}

func (r Result[T]) FlatMap(f func(T) Result[T]) Result[T] {
    if r.Err != nil { return r }
    return f(r.Value)
}
```

Mechanically works. Hardly anyone uses this in Go. The community settled on `if err != nil` for a reason — explicit handling is clearer.

### Pure functions

A pure function: same inputs → same output, no side effects. Easy to test, easy to parallelize.

Idiomatic Go: prefer pure functions when reasonable, but don't contort the design.

```go
// Pure
func taxTotal(items []Item, rate float64) float64 {
    var total float64
    for _, it := range items { total += it.Price * rate }
    return total
}

// Side effects (not pure)
func saveOrder(o Order) error { /* writes DB */ return nil }
```

Mix as needed.

### When functional patterns help

- **Iterator chains** (1.23+) for clean lazy transformations.
- **Function-as-data** for callback-style APIs (sort comparator, HTTP handlers).
- **Decorator pattern** (middleware) is functional in essence.
- **Memoization** of expensive pure work.

### When they hurt

- **Replacing a 3-line for loop with `Map(Filter(...))`**: extra reading effort.
- **Manual currying** in a language without partial-application sugar.
- **`Result[T]` to "fix" error handling**: fights the language.
- **Recursion**: no TCO; blows the stack.
- **Excessive higher-order helpers** that obscure intent.

### Things Go specifically doesn't have

- Pattern matching (1.x — there's no plan; type switches are the closest).
- Tail-call optimization.
- Lazy data structures (other than iter.Seq).
- Algebraic data types (sum types via interfaces are awkward).
- Monads (you can encode them; nobody does).
- Persistent collections (immutable trees).

## Standard Library Hooks

- `sort.Slice`, `sort.SliceStable`: function comparator.
- `slices.SortFunc`, `slices.SortStableFunc`, `slices.BinarySearchFunc` (1.21+): generic + function args.
- `slices.IndexFunc`, `slices.ContainsFunc`.
- `iter.Seq`, `iter.Seq2` (1.23+): pull-based iteration.
- `cmp.Or` (1.22+): first non-zero value.
- `maps.Keys`, `maps.Values` (return iter.Seq since 1.23).
- `strings.Map`: per-rune transformation function.
- `bytes.Map`: same for bytes.
- `regexp.ReplaceAllStringFunc`.
- `http.HandlerFunc`: function-as-handler.

## Real-World Patterns

### 1. Functional config (functional options)

Already covered (`19-patterns/01-functional-options.md`). The poster-child functional pattern in Go.

```go
srv := New("addr", WithTimeout(30*time.Second), WithLogger(log))
```

### 2. Middleware as function composition

Already covered (`19-patterns/04-middleware-chain.md`).

```go
handler := Chain(Recover, Logging, Auth)(myHandler)
```

### 3. iter.Seq pipeline (1.23+)

```go
import (
    "iter"
    "slices"
)

func evens(s iter.Seq[int]) iter.Seq[int] {
    return func(yield func(int) bool) {
        for v := range s {
            if v%2 == 0 {
                if !yield(v) { return }
            }
        }
    }
}

func squared(s iter.Seq[int]) iter.Seq[int] {
    return func(yield func(int) bool) {
        for v := range s {
            if !yield(v * v) { return }
        }
    }
}

func sum(s iter.Seq[int]) int {
    var t int
    for v := range s { t += v }
    return t
}

func main() {
    total := sum(squared(evens(slices.Values([]int{1, 2, 3, 4, 5, 6}))))
    fmt.Println(total) // 4 + 16 + 36 = 56
}
```

Lazy; no intermediate slices.

### 4. Pure function library

```go
package money

func Add(a, b Amount) Amount   { return Amount{cents: a.cents + b.cents} }
func Mul(a Amount, n int) Amount { return Amount{cents: a.cents * n} }
func Pct(a Amount, p float64) Amount { return Amount{cents: int64(float64(a.cents) * p / 100)} }
```

Pure, easy to test, easy to compose.

### 5. Decorator with closures

```go
func WithRetry(f func(ctx context.Context) error, max int) func(ctx context.Context) error {
    return func(ctx context.Context) error {
        var err error
        for i := 0; i < max; i++ {
            if err = f(ctx); err == nil { return nil }
        }
        return err
    }
}

doWork := WithRetry(actualWork, 3)
```

Closure captures `actualWork`; returned function is itself a closure ready to be called.

## Anti-Patterns & Gotchas

**Replacing every for loop with `Map`/`Filter`.** Reduces readability without functional gain.

**Deep currying.** Cute exercise, never useful in Go.

**Returning closures that capture large state.** They become hidden owners of memory; profile if you suspect leaks.

**Memoization without bounded cache.** Memory grows; needs LRU or expiration.

**Recursion without iteration fallback.** Blows the stack on real inputs.

**`Result[T]`-style monad chains.** Fights Go's idiom; the next developer will hate it.

**Using channels for transformations (`MapChan`/`FilterChan`) post-1.23.** `iter.Seq` is cheaper.

**Higher-order helpers in a hot loop.** Each call may allocate the closure. Inline.

**Pure functions that secretly mutate.** Document and audit; tests catch obvious cases.

**Algebraic data types via empty interfaces.** Type-switch-as-pattern-match is unsafe. Use closed type sets via package-level interfaces with `isType()` methods.

## Performance Notes

- For-loop: zero overhead.
- `Map`/`Filter` helper: one function call per element + maybe one closure per call.
- `iter.Seq` (1.23+): function call per yield; no closure alloc if `yield` is statically known.
- Generic dispatch: typically zero overhead after monomorphization.
- Channel-based pipeline (pre-1.23): goroutine per stage, channel send per element. Often 10× slower than iter.Seq.
- Closure allocation: ~24 bytes typical; can dominate hot paths.

`go test -benchmem` is your friend.

## How Big Companies Use It

- **Functional options** — universal across the ecosystem.
- **Middleware** — universal in HTTP/RPC servers.
- **iter.Seq adoption** — early days; expected to spread (1.23+).
- **Memoization** — quietly common; rarely advertised.
- **Pure-function discipline** — strong in financial / domain-modeling Go (Stripe, banking).
- **Channel pipelines** — historically common; replaced by iter.Seq in modern code.

## Source Code References

- `iter` package (1.23+): [`src/iter/iter.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/iter/iter.go).
- `slices` package: [`src/slices/slices.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/slices/slices.go).
- `maps` package: [`src/maps/maps.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/maps/maps.go).
- `strings.Map`, `bytes.Map`: [`src/strings/strings.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/strings/strings.go).
- Generic `cmp.Or`: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmp/cmp.go).
- A community generic library: [`samber/lo`](https://github.com/samber/lo) — controversial; uses functional-Go heavily.
- `golang.org/x/exp/slices` (predecessor of stdlib slices).

## Further Reading

- "Range Over Function Types" (Go 1.23 release): https://go.dev/blog/range-functions.
- Russ Cox, "Why Generics?" — context for the design: https://go.dev/blog/intro-generics.
- "Go Generics" (FAQ-style discussions on slices.Map): golang-nuts archives.
- "Functional Go" — Tomáš Beránek talks.
- Dave Cheney, "Don't use functional options for everything": https://dave.cheney.net.
- "Effective Go": composition over inheritance (general guidance).
- Bill Kennedy, "When generics are NOT needed in Go".
- `samber/lo` README — the case for functional Go; community reaction.

## Exercises / Self-Check

1. Rewrite a 3-line `for` loop using `Map`/`Filter`. Compare readability and benchmarks.
2. Build a lazy pipeline with `iter.Seq`: take a slice of integers, filter even, square, take 5. Run benchmarks against the equivalent for loop.
3. Implement memoization with bounded cache (e.g., `Memoize` with LRU). Test under concurrent access.
4. Why does Go not have tail-call optimization? Construct a recursive function that fails, then rewrite as iteration.
5. Argue both sides of `samber/lo`'s value for a 100k-LOC Go codebase. Where does it help? Where does it hurt?
