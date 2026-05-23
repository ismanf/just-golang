# Closures — Capture Semantics and the Loop Variable Trap (1.22 Fix)

## TL;DR

A **closure** is a function value that captures variables from its enclosing scope. In Go, closures capture variables **by reference** — the closure holds a pointer to the captured variable's storage, not a copy of its value at capture time. This is what makes closures power middleware, event handlers, deferred work, and goroutine setups. It is also the source of **the single most-asked Go interview question**: "what does this loop print?" The notorious loop-variable trap — pre-Go-1.22, the `i` in `for i := 0; i < 3; i++` was the *same variable* across all iterations, so closures captured all-the-same-thing and saw the final value `3`. Go 1.22 (Feb 2024) **changed loop semantics**: each iteration of `for i := 0; i < N; i++` and `for i, v := range x` now gets its own copy of `i` and `v`. Code that previously printed `3, 3, 3` now prints `0, 1, 2`. This is the most significant language-behaviour change since Go 1.0 — and it required a module-level `go 1.22` directive to opt in (older modules keep the old behaviour). The single biggest gotcha that remains: **closures still capture mutable variables by reference**, so if you assign to the captured variable after creating the closure, the closure sees the new value — loop fix or not.

## Mental Model

```
   counter := 0                          counter
   add := func() { counter++ }            ▲
   add()                                  │ shared variable
   add()                                  │ (closed-over)
   fmt.Println(counter)   // 2            │
                                          │
                                          │
   closure value:                         │
   ┌────────────────┐                     │
   │ fn pointer ────┼─► code              │
   │ env ───────────┼─► [&counter] ──────┘
   └────────────────┘
```

Three invariants:

1. **Capture is by reference**, not by value (for *variables* — captured *constants* are baked in).
2. **A closure keeps its captured variables alive** even after the enclosing function returns. This is why the variables escape to the heap.
3. **Each call to the outer function creates a new closure** with a new copy of the enclosing variables.

## A Simple Closure

```go
func makeAdder(x int) func(int) int {
    return func(y int) int { return x + y }
}

add5 := makeAdder(5)
add5(3)    // 8
add5(10)   // 15

add7 := makeAdder(7)
add7(3)    // 10
```

Each call to `makeAdder` creates a fresh `x`. `add5`'s `x` is 5; `add7`'s `x` is 7. They don't share.

The closure holds a reference to `x`. Since `x` outlives `makeAdder`'s stack frame (the returned function uses it), `x` escapes to the heap. Escape analysis: see `12-runtime`.

## Mutation Through Captures

```go
func makeCounter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

c := makeCounter()
c()   // 1
c()   // 2
c()   // 3
```

`n` is captured by reference; each call mutates it. The closure *is* the state holder.

If you wanted independent counters from one factory:

```go
c1 := makeCounter()
c2 := makeCounter()
c1()  // 1
c1()  // 2
c2()  // 1   ← independent
```

Each `makeCounter()` allocates a fresh `n`.

## The Loop Variable Trap (Pre-1.22 Behaviour)

```go
// Pre-Go-1.22
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs { f() }
// Output: 3 3 3
```

Why? Pre-1.22, **`i` was one variable**, scoped to the entire `for` statement. All three closures captured the same `i`. After the loop, `i == 3`. All three print 3.

The same applied to range:

```go
for _, v := range []int{1, 2, 3} {
    go func() { fmt.Println(v) }()
}
// Pre-1.22: prints 3, 3, 3 (in some order)
// (v is reused; goroutines see whatever v is when they happen to run)
```

The fix pre-1.22 was to **rebind**:

```go
for i := 0; i < 3; i++ {
    i := i    // new variable, shadowing the outer i
    funcs = append(funcs, func() { fmt.Println(i) })
}
```

Or pass as a parameter:

```go
for i := 0; i < 3; i++ {
    go func(i int) { fmt.Println(i) }(i)
}
```

This bug bit *every* Go developer eventually. It was the canonical Go gotcha for ~12 years.

## The 1.22 Change

```go
// Go 1.22+
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs { f() }
// Output: 0 1 2  ← fixed!
```

The loop variables `i` and `v` (in the range form) are now **freshly scoped per iteration**. Each closure captures *its own* `i`.

How to opt in: your `go.mod`'s `go` directive must be `go 1.22` or later. Modules with `go 1.21` or older keep the old semantics — the change is intentionally backward-compatible at the module level.

```go
// go.mod
module example.com/myapp
go 1.22    // ← this enables the new semantics
```

Check: `go env GOEXPERIMENT` and the build version. Most CI logs will tell you.

### What didn't change

- **`for cond {}`** doesn't define variables, so no change.
- **`for { }`** (infinite) doesn't define variables.
- **Other variables declared inside the loop body**: behaviour unchanged (always fresh per iteration).
- **`switch`, `if`-init**: not affected; they already had per-statement scope.

### What the toolchain does

The compiler under `go 1.22+` rewrites:

```go
for i := 0; i < n; i++ { body(i) }
```

to (conceptually):

```go
for i := 0; i < n; i++ {
    i := i           // implicit shadow
    body(i)
}
```

There's no runtime cost in the common case where the closure isn't escaping — escape analysis handles it. If the closure does escape (passed to a goroutine, stored), a small heap allocation per iteration occurs for the new `i`. The same as you'd have written explicitly.

### Migration

For most code: do nothing. The new semantics is what you wanted in the first place. Tests that depended on the old behaviour (rare but exists) need updating.

For libraries: bump `go.mod` to `go 1.22+` and verify. Use `go vet`'s `loopclosure` analyzer to find old-style code.

## Still-Relevant Gotchas (Post-1.22)

```go
// Mutation after closure capture STILL applies
x := 1
f := func() { fmt.Println(x) }
x = 99
f()    // 99 — not 1
```

This is correct closure semantics, not a bug. If you want to snapshot:

```go
x := 1
val := x
f := func() { fmt.Println(val) }
x = 99
f()    // 1
```

Or with the function parameter pattern:

```go
f := func(v int) func() { return func() { fmt.Println(v) } }(x)
x = 99
f()    // 1
```

### Defer arguments are NOT captured by reference

```go
x := 1
defer fmt.Println(x)   // captures 1
x = 99
// At return: prints 1
```

Defer arg eval happens at the defer statement. But a closure inside defer:

```go
x := 1
defer func() { fmt.Println(x) }()
x = 99
// At return: prints 99
```

See `07-defer.md`.

## Common Closure Patterns

### Middleware

```go
func withLogging(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println(r.URL.Path)
        h.ServeHTTP(w, r)
    })
}
```

`h` is captured. Each call to `withLogging` produces a new closure that knows which handler to forward to.

### Functional options

```go
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}
```

Returns a closure that captures `d`. The closure is later invoked against a `Server`.

### Memoization

```go
func memoize(f func(int) int) func(int) int {
    cache := map[int]int{}
    var mu sync.Mutex
    return func(x int) int {
        mu.Lock(); defer mu.Unlock()
        if v, ok := cache[x]; ok { return v }
        v := f(x)
        cache[x] = v
        return v
    }
}
```

The cache and mutex are captured. Each `memoize` call produces an independent memoized version.

### Worker / pump

```go
func startWorker(ch <-chan job) {
    go func() {
        for j := range ch {
            process(j)
        }
    }()
}
```

`ch` is captured; the goroutine reads from it forever.

## Performance — Heap vs Stack

A closure that doesn't escape (lifetime bounded by the enclosing function) can have its captured variables on the stack — zero allocation. Example:

```go
func compute(xs []int) int {
    f := func(x int) int { return x * 2 }
    total := 0
    for _, x := range xs { total += f(x) }
    return total
}
```

`f` doesn't escape; the compiler may inline it entirely. Verify with `go build -gcflags="-m"`.

A closure that escapes:

```go
func adder() func(int) int {
    n := 0
    return func(x int) int { n += x; return n }
}
```

`n` must outlive `adder`'s frame → escapes to heap → one allocation per `adder()` call.

In the common middleware pattern (`return func() {...}`), the closure escapes. The cost is one allocation; usually negligible.

## Closures and Goroutines

```go
for _, v := range items {
    go func() { process(v) }()
}
// Pre-1.22: ALL goroutines see the same v (often the last)
// Post-1.22 with go 1.22+: each goroutine sees its own v
```

If your `go.mod` targets `go 1.22+` and you've verified your `go vet -loopclosure` is clean, the post-1.22 pattern is safe. Older modules: rebind explicitly.

## Closures vs Methods on Bound Receivers

```go
type Counter struct { n int }
func (c *Counter) Inc() { c.n++ }

c := &Counter{}
inc := c.Inc   // method value — captures c
inc(); inc()   // c.n is now 2
```

A **method value** like `c.Inc` is conceptually a closure: it captures the receiver. The compiler may emit a tiny stub allocation per method-value assignment.

`(*Counter).Inc` (note the type) is a **method expression** — not bound, takes the receiver as an explicit first arg:

```go
fn := (*Counter).Inc
fn(c)   // equivalent to c.Inc()
```

Method expressions don't capture; no allocation. Useful when you want a generic "given a receiver, do this."

## Anti-Patterns & Gotchas

**Pre-1.22 loop variable in goroutine without rebind.** Famous bug. Use `go 1.22+` or rebind.

**Capturing a huge struct in a closure that escapes.** Holds the struct alive for the closure's lifetime. Capture only what you need.

**Closures inside hot loops** that allocate. Profile; might be the cost.

**Mutating captured variables from multiple goroutines.** Data race. Use a mutex or atomic.

**Closure that captures `*T` and runs after the `*T` is GC'd via some other path** — impossible in Go (the closure holds a reference, keeping it alive), but check that you don't accidentally hold onto a `*T` you wanted to free.

**Defer + closure that captures `err`** to log it — the closure sees the latest `err`, including any later overwrite. Sometimes desired, sometimes a bug.

**Method values in tight loops** — small allocation per assignment. Cache or use method expressions.

**Forgetting that the *closure value* itself is just a pointer + env**: copying a closure variable is cheap; calling the closure may not be (function call + indirect jump).

**Returning a closure that captures a `sync.WaitGroup` by value.** Not what you want — `WaitGroup` is a struct; capturing by value is a copy. Capture `*sync.WaitGroup`.

**Recursive closures**: you have to declare the variable first:
```go
var fact func(int) int
fact = func(n int) int {
    if n <= 1 { return 1 }
    return n * fact(n - 1)
}
```
Otherwise the closure can't refer to itself.

**Assuming the captured variable's address is stable.** It is for the closure's lifetime, but two different closures from the same factory have different addresses.

**Forgetting that the `range` over a map / channel returns copies of values**. Modifying `v` in `for _, v := range m` doesn't change the map.

## Performance Notes

- **Closure call overhead**: ~1-2 ns vs direct call (indirect jump, cache-friendly if hot).
- **Closure value size**: 16 bytes (function pointer + env pointer) on 64-bit.
- **Escape**: closure with captured vars that outlives the function → heap allocation of an env struct (size varies with what's captured).
- **Stack closures (non-escaping)**: zero alloc; may be inlined.
- **Pre-1.22 vs post-1.22 loop bind**: post-1.22 may incur one alloc per iteration *if* the closure escapes. If it doesn't, escape analysis catches it; zero cost.

Check escape: `go build -gcflags="-m" your/pkg`. Look for `... escapes to heap` annotations.

## How Big Companies Use It

- **Google's internal style guide** notes the 1.22 change and recommends bumping `go.mod` once it's released. Discourages relying on closure capture for performance-critical loops.
- **Kubernetes** had hundreds of loopvar-captures that triggered the change; the project upgraded to `go 1.22` and fixed remaining linter findings.
- **Uber's Go style guide** mandates `loopclosure` linter clean.
- **CockroachDB** uses memoization closures for SQL expression compilation.
- **HashiCorp** uses functional-options closures throughout config builders.
- **Tailscale** uses closures for goroutine setup in their daemon; relies on `go 1.22+` semantics.

## Source Code References

- Go spec — Function literals: https://go.dev/ref/spec#Function_literals.
- 1.22 loop variable change spec: https://go.dev/ref/spec#For_statements (look for "scope" sections).
- Loop variable proposal: https://github.com/golang/go/issues/60078.
- Migration guide: https://go.dev/wiki/LoopvarExperiment.
- `loopclosure` analyzer: https://github.com/golang/tools/tree/master/go/analysis/passes/loopclosure.
- Escape analysis: https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/.

## Further Reading

- Russ Cox, "Fixing for loops in Go 1.22": https://go.dev/blog/loopvar-preview.
- David Crawshaw, "Closures and the loop variable" (Go team writing).
- Go 1.22 release notes — language: https://go.dev/doc/go1.22#language.
- Dave Cheney, "Practical Go" — touches on closure escape.
- Brad Fitzpatrick, GopherCon "Go performance" talks (closure escape examples).

## Exercises / Self-Check

1. Build a `makeAdder(x)` factory. Verify two adders with different `x` are independent.
2. Write the pre-1.22 loop-variable bug: `for i := 0; i < 3; i++ { fns = append(fns, func() { fmt.Println(i) }) }`. Build with `go 1.21` directive — observe `3 3 3`. Bump to `go 1.22+` — observe `0 1 2`.
3. Capture a slice in a closure. Mutate the slice from the outer scope. Verify the closure sees the mutation.
4. Build a memoize wrapper. Test that the cache persists across calls.
5. Write a recursive closure with `var f func(int) int` declared first.
6. Use `go build -gcflags="-m"` to check whether a tiny closure escapes. Try restructuring to avoid escape.
7. Capture a `*sync.WaitGroup` in a goroutine launched in a loop. Confirm `wg.Done()` runs as expected (not duplicated, not skipped).
8. Convert a method value (`c.Inc`) to a method expression (`(*Counter).Inc`). Use both in a slice of `func()` and `func(*Counter)` respectively.
