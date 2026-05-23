# `defer` — Stack Semantics, Open-Coded Defers, Argument Capture

## TL;DR

`defer` schedules a function call to run when the surrounding function returns — whether normally, via `return`, or because of a `panic`. It is Go's structured-cleanup primitive, replacing C++'s RAII destructors and Java's `try`/`finally`. Three rules cover almost every use: (1) deferred calls run in **LIFO** order (last defer runs first); (2) **arguments are evaluated at the `defer` statement**, not when the deferred function actually runs — `defer fmt.Println(x)` captures `x` *now*, even if `x` changes before return; (3) **deferred calls run after the return value has been computed but before the function actually returns** — they can mutate named return values. Go 1.14 added **open-coded defers** — when the compiler sees a small, fixed number of defers in a function (and no defers in a loop), it inlines them at the return points, eliminating the historical ~50ns-per-defer overhead. Modern defers are nearly free (~3-5 ns) for the common case. The single biggest gotcha: **`defer` in a loop accumulates** — `for ... { defer f.Close() }` creates one deferred call per iteration, all firing at function exit. If you're closing per-iteration resources, wrap each iteration in its own function (or call `Close` explicitly).

## Mental Model

```
   func process() error {
       f, _ := os.Open("a")           // ─┐
       defer f.Close()                //  │ scheduled at function exit
                                      //  │
       g, _ := os.Open("b")           // ─┤
       defer g.Close()                //  │ scheduled at function exit
                                      //  │
       // ... work ...                //  │
       return nil                     //  │
   }                                  //  ▼
        ↓
   At function exit (LIFO):
       1. g.Close()   ← last-deferred runs first
       2. f.Close()
```

## Basic Usage

```go
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil { return nil, err }
    defer f.Close()
    return io.ReadAll(f)
}
```

The classic pattern: acquire a resource, defer its cleanup, use it. The cleanup runs whether you return early, return at the end, or panic.

## LIFO Order

```go
func order() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
// Prints: 3, 2, 1
```

Each `defer` pushes to an internal stack; at return, the stack is popped. This means newer cleanups (which may depend on older resources) run first.

```go
db.BeginTx(ctx)
defer tx.Rollback()    // 2nd — runs first; safe to call after Commit (no-op)

// ... do work ...
if err := tx.Commit(); err != nil { return err }
return nil
```

## Argument Capture

```go
x := 10
defer fmt.Println(x)   // captures x = 10
x = 20
// At return: prints "10"
```

The arguments are evaluated immediately at the `defer` statement. The function call is deferred; the arguments are not.

To capture the **current** value at return time, use a closure:

```go
x := 10
defer func() { fmt.Println(x) }()    // captures x by reference
x = 20
// At return: prints "20"
```

This distinction is the source of many bugs. When in doubt: a bare `defer f(x)` snapshots `x`; a `defer func() { f(x) }()` reads the current `x`.

## Named Returns + Defer

```go
func split(s string) (left, right string) {
    defer func() {
        if left == "" { left = "<empty>" }
    }()

    parts := strings.SplitN(s, "=", 2)
    if len(parts) == 2 {
        left, right = parts[0], parts[1]
    }
    return
}
```

A deferred function can **mutate named return values**. This is sometimes used to wrap errors:

```go
func process(path string) (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("process %q: %w", path, err)
        }
    }()

    // ... can return raw errors; the defer wraps them
}
```

Note: this only works with **named** returns. For unnamed returns, the value has already been assigned to a temporary at the `return` statement; the defer can't see or change it.

## `defer` in Loops

```go
// BAD — accumulates deferreds; all run at function exit
func processAll(paths []string) error {
    for _, p := range paths {
        f, err := os.Open(p)
        if err != nil { return err }
        defer f.Close()       // 1000 defers if len(paths) == 1000
        // use f
    }
    return nil
}
```

Each iteration adds a defer. If the loop runs N times, N file handles stay open until the function returns. Often a leak.

Fix 1: per-iteration function:

```go
func processAll(paths []string) error {
    for _, p := range paths {
        if err := processOne(p); err != nil { return err }
    }
    return nil
}

func processOne(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close()
    // use f
    return nil
}
```

Fix 2: closure:

```go
for _, p := range paths {
    func() {
        f, err := os.Open(p)
        if err != nil { /* ... */ }
        defer f.Close()
        // ...
    }()
}
```

Fix 3: explicit close at end of iteration:

```go
for _, p := range paths {
    f, err := os.Open(p)
    if err != nil { return err }
    // use f
    f.Close()
}
```

## Open-Coded Defers (Go 1.14+)

Before Go 1.14, every `defer` incurred ~50-60 ns of overhead (allocating a `_defer` struct in a linked list). Microbenchmarks made `defer` look prohibitively expensive for hot paths.

Go 1.14 introduced **open-coded defers**: when a function has ≤8 defers, none in a loop, and all are "regular" calls, the compiler inlines them at every return point and panic path. Overhead drops to ~3-5 ns — essentially free.

Conditions that disable open-coding:

- Defer inside a loop.
- More than 8 defers in a function.
- Defer used reflectively (`reflect.DeferFunc`, rare).
- Some other corner cases.

In practice: if you write idiomatic per-function defers, the overhead is negligible. Profile before assuming it's slow.

## `defer` and `panic`

```go
func mayPanic() {
    defer fmt.Println("cleanup")
    panic("boom")
}
// Prints "cleanup", then crashes.
```

Defers run during panic propagation, in LIFO order. This is **the** mechanism for `recover`:

```go
func safeProcess() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()

    risky()
    return nil
}
```

`recover()` only works from inside a deferred function. Outside a defer, it returns `nil`. The panic value is whatever was passed to `panic()`. Covered in `05-errors`.

## `defer` and `os.Exit`

```go
func main() {
    defer fmt.Println("cleanup")
    os.Exit(1)    // CLEANUP NEVER RUNS
}
```

`os.Exit` terminates immediately, bypassing defers. Same for `log.Fatal` (which calls `os.Exit(1)`). Use `os.Exit` only at the very top of `main`, after you've done your own cleanup.

## `defer` with Method Calls on Pointer Receivers

```go
type Resource struct { name string }
func (r *Resource) Close() { fmt.Println("closing", r.name) }

func process() {
    r := &Resource{name: "A"}
    defer r.Close()
    r = &Resource{name: "B"}
    // At return: prints "closing A"  (the original r was captured)
}
```

The **receiver** is captured at defer time, just like arguments. Reassigning `r` after the defer doesn't change which resource gets closed.

## Performance Patterns

```go
// Hot loop: avoid defer
func hot() {
    for i := 0; i < 1000000; i++ {
        f := acquire()
        // ... work ...
        f.release()    // explicit, not defer
    }
}

// Cold function: defer is fine
func cold() error {
    f, _ := os.Open("x")
    defer f.Close()
    // ...
}
```

For very hot loops (microseconds per iteration), even the ~3-5 ns of open-coded defer might matter. Otherwise, prefer `defer` for safety.

## `defer` for Locks

```go
mu.Lock()
defer mu.Unlock()
// ... work ...
```

Idiomatic. Ensures unlock even on early return or panic.

For very hot critical sections where panic-safety isn't needed:

```go
mu.Lock()
// ... very tight work ...
mu.Unlock()
```

But measure first.

## `defer` for Timers / Profiling

```go
func instrumented() {
    start := time.Now()
    defer func() { log.Printf("took %s", time.Since(start)) }()
    // ... work ...
}
```

Common pattern for tracing function durations.

```go
defer trace.StartRegion(ctx, "compute").End()
```

`runtime/trace`'s `StartRegion` returns a value with an `End()` method designed for this pattern.

## Anti-Patterns & Gotchas

**`defer` in a loop without per-iteration scope.** Accumulates.

**Forgetting that args are captured at defer time.** `defer log.Printf("%d", counter)` prints the value of `counter` at defer time.

**Calling `defer` on a nil function pointer.** Allowed at defer time (the call is deferred); panics at return time.

**`defer recover()` outside a deferred function.** No effect.

**Mutating an unnamed return via defer.** Doesn't work; the return value is already captured at the `return` statement.

**Heavy work in a deferred function on a hot path.** Defer overhead is one thing; the function itself might be slow.

**Relying on `defer` after `os.Exit` / `log.Fatal`.** They don't run.

**Closing the same resource twice via overlapping defers and explicit Close.** Often harmless (Go's `io.Closer.Close` is usually idempotent), sometimes panics. Be explicit.

**`defer fmt.Println(err)` to "log on exit".** The `err` is captured at defer; later assignments invisible. Use a closure.

**Using `defer` for ordering-critical operations** without considering LIFO. The third defer runs first.

**`defer cancel()` from `context.WithCancel` placed before the err check.** Generally correct (cancelling a not-yet-used ctx is fine), but be aware.

**Massive numbers of defers in one function.** Beyond ~8, falls off the open-coded path. Refactor to separate functions.

**Using `defer` in tests for setup teardown** when `t.Cleanup` is available (1.14+). `t.Cleanup` runs in the right scope and handles subtests; defer doesn't.

**Deferring a method on a stack-allocated struct that may have been mutated.** Receiver captured at defer; mutations visible.

## Performance Notes

| Form | Cost (modern x86) |
|------|-------------------|
| Open-coded defer | ~3-5 ns |
| Non-open-coded (loop, >8 defers) | ~30-60 ns |
| `defer func() { ... }()` closure | same as above; closure may allocate |
| `defer` in a loop, 1000 iterations | 30k-60k ns total deferred work |

Profile with `go test -bench` if you suspect `defer` is hot. In the vast majority of code, it isn't.

The compiler heuristics for open-coding are documented in `cmd/compile/internal/ssagen/ssa.go` (`defer8` and related logic).

## How Big Companies Use It

- **Google's internal style guide** advocates `defer` for *every* resource acquisition, with rare exceptions for measured hot paths.
- **Kubernetes** uses `defer` extensively in informer setup, controller queues, and worker loops.
- **CockroachDB** uses `defer` for transaction cleanup, lock release, and tracing region ends.
- **Cloudflare** profiles defer impact in their proxy hot paths; uses explicit `Close` in the very innermost loops, `defer` everywhere else.
- **HashiCorp Vault** uses `defer` for telemetry timer recording.
- **Tailscale** uses `defer` heavily; their style guide explicitly recommends it over manual cleanup.

Defer is universally adopted. The only debate is "always defer" vs "explicit close in hot loops."

## Source Code References

- Go spec — Defer statements: https://go.dev/ref/spec#Defer_statements.
- Go runtime `_defer` struct: https://github.com/golang/go/blob/master/src/runtime/runtime2.go (search `_defer`).
- Open-coded defer pass: https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go.
- Defer implementation history: https://blog.golang.org/proposal/14939-runtime-deferred-funcs (Go 1.14 design doc).
- `testing.T.Cleanup`: https://pkg.go.dev/testing#T.Cleanup.

## Further Reading

- "Go 1.14 release notes" (defer perf improvements): https://go.dev/doc/go1.14.
- Dave Cheney, "Performance without the event loop" (touches on defer cost).
- Damian Gryski, "Defer overhead in Go": https://github.com/dgryski/go-perfbook.
- Effective Go — Defer: https://go.dev/doc/effective_go#defer.
- Russ Cox, design notes on open-coded defers.

## Exercises / Self-Check

1. Write a function with three `defer`s. Confirm LIFO order.
2. Demonstrate argument capture: `x := 1; defer fmt.Println(x); x = 2`. Confirm "1" prints.
3. Demonstrate closure capture: `defer func() { fmt.Println(x) }()`. Confirm "2" prints.
4. Open 1000 files in a loop with `defer f.Close()`. Count open file descriptors (`ls /proc/<pid>/fd | wc -l`). Refactor with per-iteration scope.
5. Use named returns + `defer` to wrap errors with context.
6. Benchmark a function with one defer vs the same function with explicit cleanup. Quantify open-coded defer cost.
7. Add `defer fmt.Println("end")` before `os.Exit(1)`. Confirm it never runs.
8. Use `defer` for a `sync.Mutex.Unlock`. Cause a panic inside the critical section; confirm the mutex still unlocks (no deadlock).
