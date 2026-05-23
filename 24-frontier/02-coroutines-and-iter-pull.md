# Coroutines and `iter.Pull`

## TL;DR

Since Go 1.23, the `iter` package exposes **`iter.Pull`** and **`iter.Pull2`** — adapters that turn a **push-based iterator** (`iter.Seq`) into a **pull-based iterator** by running it on a *coroutine* (a goroutine with custom park/resume semantics). The implementation is built on top of new runtime primitives — `coro.go` — that schedule the iterator goroutine cooperatively: the consumer "pulls" by calling a `next` function, which transfers control to the iterator until it yields, then transfers back. The single biggest gotcha: **`Pull` returns a `stop` function you MUST call** (often via `defer`), or the underlying coroutine leaks until GC notices. Coroutines are NOT general-purpose — they're a focused mechanism specifically for converting between iterator shapes.

## Mental Model

```
   Push iterator (iter.Seq):
       The producer calls yield(v); consumer is a `for v := range seq`.
       Loop control flows producer → consumer → producer (each yield is a "push").
   
   Pull iterator (via iter.Pull):
       The consumer calls next() to get the next value.
       Loop control flows consumer → producer (next) → consumer (one value at a time).
   
   Conversion happens by running the producer on a coroutine:
   
   ┌────────────────────────┐         ┌──────────────────────┐
   │ Consumer (main goroutine) │      │ Producer (coroutine) │
   │                         │       │                       │
   │ v, ok := next()  ───────┼──────▶│ yield(v) runs, then   │
   │                         │       │   blocks until next() │
   │ ◀───────── value back ──┼───────│                       │
   │                         │       │                       │
   │ v, ok = next()  ────────┼──────▶│ resumes after yield   │
   │                         │       │ yield(next v)         │
   └────────────────────────┘         └──────────────────────┘
   
   stop() — releases the coroutine if the consumer abandons early.
```

`iter.Pull` is the bridge between Go's range-over-func (push) and APIs that want explicit `next()`-style stepping (pull) — common in merge-sort, zip, take/while patterns.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"iter"
	"slices"
)

func main() {
	seq := slices.Values([]int{1, 2, 3, 4, 5})

	next, stop := iter.Pull(seq)
	defer stop()                              // VITAL — releases coroutine

	for {
		v, ok := next()
		if !ok { break }
		fmt.Println(v)
	}
}
```

`iter.Pull(seq)` returns:
- `next func() (T, bool)` — call to get the next value. `ok=false` means iteration done.
- `stop func()` — call when you're finished, even if you didn't exhaust the iterator.

Without `defer stop()`, the producer coroutine is parked indefinitely (until GC eventually frees it via a finalizer).

For `iter.Seq2`:

```go
next, stop := iter.Pull2(maps.All(m))
defer stop()
for {
    k, v, ok := next()
    if !ok { break }
    use(k, v)
}
```

## Deep Dive

### Why pull when you have push?

Many algorithms naturally express as **lockstep iteration over multiple sequences**:

- **zip**: `(a[0], b[0]), (a[1], b[1]), ...`
- **merge** (sorted): take next from whichever side is smaller.
- **take/while**: consume until a condition.
- **chunked**: take N at a time, then resume.

With push iteration alone, these are awkward: you'd have to buffer one sequence into a slice, then walk both. Pull lets you advance each side independently.

```go
func zip[A, B any](as iter.Seq[A], bs iter.Seq[B]) iter.Seq2[A, B] {
    return func(yield func(A, B) bool) {
        nextA, stopA := iter.Pull(as)
        defer stopA()
        nextB, stopB := iter.Pull(bs)
        defer stopB()
        for {
            a, okA := nextA()
            b, okB := nextB()
            if !okA || !okB { return }
            if !yield(a, b) { return }
        }
    }
}
```

`zip` over two push iterators using pull. Each call to `nextA`/`nextB` advances exactly one side.

### Under the hood: coroutines

Go runtime added `coro.go` ([`src/runtime/coro.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/coro.go)) for this purpose. A coroutine is a goroutine with restricted scheduling:

- It runs on the same M (OS thread) as its caller during a `next()` call.
- When the coroutine reaches a `yield`, it parks; control returns to the caller.
- The next `next()` resumes the coroutine.
- `stop()` cancels the coroutine cooperatively.

Implementation: a coroutine is internally a `*g` with `coroexit` semantics. The "switch" between consumer and coroutine is a register save/restore — *not* a full goroutine context switch through the scheduler.

### Cost characteristics

- `iter.Pull` setup: starts a coroutine; ~1 µs.
- Per-`next()`: ~10-50 ns coroutine switch (same M; no scheduler involvement).
- `stop()`: ~µs.
- Goroutine allocation: ~1 µs for the underlying `g`; reused via pool.

For comparison: a channel-based pull would be ~µs per step (full goroutine switch + channel ops). Coroutines are ~10× faster than channels for this pattern.

### `stop()` is mandatory

If the consumer abandons iteration mid-way and never calls `stop()`, the producer coroutine is **parked forever** until the iterator value is garbage-collected. The runtime sets a finalizer that calls `stop()` on GC, but that's a best-effort safety net.

Always:

```go
next, stop := iter.Pull(seq)
defer stop()
```

Forgetting it is essentially a goroutine leak. Linters and `go vet` flag missing `defer stop()` in some cases; static analysis isn't complete yet.

### `iter.Pull` vs channels

Old pattern (channel-based pull):

```go
func toChan[T any](s []T) <-chan T {
    c := make(chan T)
    go func() {
        defer close(c)
        for _, v := range s {
            c <- v
        }
    }()
    return c
}

c := toChan([]int{1, 2, 3})
for v := range c {
    fmt.Println(v)
}
```

Issues:
- Full goroutine per channel; expensive.
- `c <- v` is a scheduler-mediated context switch.
- Goroutine leaks if consumer doesn't drain.

`iter.Pull` solves all three: cheaper switch, automatic stop, single coroutine.

### Pull from a Pull (chaining)

You can pull from a sequence that was created by pulling — coroutine on top of coroutine:

```go
func skip[T any](seq iter.Seq[T], n int) iter.Seq[T] {
    return func(yield func(T) bool) {
        next, stop := iter.Pull(seq)
        defer stop()
        for i := 0; i < n; i++ {
            if _, ok := next(); !ok { return }
        }
        for {
            v, ok := next()
            if !ok { return }
            if !yield(v) { return }
        }
    }
}
```

`skip(seq, 5)` pulls 5 values and discards them, then yields the rest.

Coroutines stack; each `iter.Pull` adds one level. Several levels deep is fine; hundreds might add up.

### `iter.Pull2` for Seq2

```go
import "iter"

next, stop := iter.Pull2[K, V](someSeq2)
defer stop()
for {
    k, v, ok := next()
    if !ok { break }
    use(k, v)
}
```

Same semantics, three returns from `next()`.

### Coroutines are NOT exposed publicly

There's no `runtime.NewCoro()`-style API. Coroutines are an *implementation detail* of `iter.Pull` (and possibly future iterator helpers). Don't try to abstract them yourself.

This is deliberate: the Go team wants to expose just enough to make iter.Pull work, not promise a general coroutine API.

### Future use of coroutines

Some discussed but not committed extensions:
- Generator-style functions (Python `yield`-shaped).
- Async / promise patterns.

None are committed. The current footprint (one `Pull`/`Pull2` per package using iterators) is what we have.

### Compared to other languages

- **Python generators** (`yield` keyword): closest analog. Python's `yield` is the function-level coroutine; iter.Pull provides the same shape but through library API, not syntax.
- **Lua coroutines**: full coroutines with custom resume/yield. Go's are restricted.
- **C++ coroutines** (C++20): syntactic sugar with deep compiler integration. Go's are simpler, library-based.
- **Kotlin coroutines**: have similar two-way control flow but are higher-level (suspend functions, structured concurrency).

### Goroutine leaks via missed `stop()`

Real leak case:

```go
func first[T any](seq iter.Seq[T]) (T, bool) {
    next, _ := iter.Pull(seq)
    // FORGOT defer stop()!
    return next()
}
```

The unused `stop` is dropped; the coroutine remains parked. The iterator's `*coro` is GC-rooted via internal references until something explicitly frees it.

Always pair `iter.Pull` with `defer stop()`.

### `range` over iter.Pull?

You can't `range next, stop := iter.Pull(...)`. The `range` keyword works on push iterators. Pull's whole point is non-range usage.

Convention: if you can use `range seq` directly, do that. Use `Pull` only when you need explicit `next()`.

### Performance characteristics

```go
func BenchmarkPullVsChannel(b *testing.B) {
    s := []int{1, 2, 3, /* ... */ }
    b.Run("pull", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            next, stop := iter.Pull(slices.Values(s))
            for {
                _, ok := next()
                if !ok { break }
            }
            stop()
        }
    })
    b.Run("channel", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            c := make(chan int, 0)
            go func() {
                for _, v := range s { c <- v }
                close(c)
            }()
            for range c {}
        }
    })
}
```

Typical: pull ~10× faster than channel for short sequences.

## Standard Library Hooks

- `iter.Pull[T](seq iter.Seq[T]) (next func() (T, bool), stop func())`.
- `iter.Pull2[K, V](seq iter.Seq2[K, V]) (next func() (K, V, bool), stop func())`.
- `iter.Seq`, `iter.Seq2` — the push side.
- `slices.Values`, `slices.All`, `maps.All`, `maps.Keys`, `maps.Values` — return iter.Seq / iter.Seq2.
- `runtime/coro.go`: internal, not for direct use.

## Real-World Patterns

### 1. Zip two sequences

```go
func zip[A, B any](as iter.Seq[A], bs iter.Seq[B]) iter.Seq2[A, B] {
    return func(yield func(A, B) bool) {
        nextA, stopA := iter.Pull(as)
        defer stopA()
        nextB, stopB := iter.Pull(bs)
        defer stopB()
        for {
            a, okA := nextA()
            b, okB := nextB()
            if !okA || !okB { return }
            if !yield(a, b) { return }
        }
    }
}
```

### 2. Merge sorted sequences

```go
func merge[T cmp.Ordered](as, bs iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        nextA, stopA := iter.Pull(as)
        defer stopA()
        nextB, stopB := iter.Pull(bs)
        defer stopB()

        a, okA := nextA()
        b, okB := nextB()
        for okA && okB {
            if a <= b {
                if !yield(a) { return }
                a, okA = nextA()
            } else {
                if !yield(b) { return }
                b, okB = nextB()
            }
        }
        for okA { if !yield(a) { return }; a, okA = nextA() }
        for okB { if !yield(b) { return }; b, okB = nextB() }
    }
}
```

Classic two-finger merge over push iterators.

### 3. Chunked iteration

```go
func chunked[T any](seq iter.Seq[T], n int) iter.Seq[[]T] {
    return func(yield func([]T) bool) {
        next, stop := iter.Pull(seq)
        defer stop()
        for {
            chunk := make([]T, 0, n)
            for i := 0; i < n; i++ {
                v, ok := next()
                if !ok { break }
                chunk = append(chunk, v)
            }
            if len(chunk) == 0 { return }
            if !yield(chunk) { return }
        }
    }
}
```

Pulls N items at a time, yields slices.

### 4. Diff iteration (two sorted sets)

```go
func diff[T cmp.Ordered](a, b iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        nextA, stopA := iter.Pull(a)
        defer stopA()
        nextB, stopB := iter.Pull(b)
        defer stopB()
        va, okA := nextA()
        vb, okB := nextB()
        for okA && okB {
            switch {
            case va < vb:
                if !yield(va) { return }
                va, okA = nextA()
            case va == vb:
                va, okA = nextA()
                vb, okB = nextB()
            case va > vb:
                vb, okB = nextB()
            }
        }
        for okA { if !yield(va) { return }; va, okA = nextA() }
    }
}
```

Yields elements in `a` not in `b` (both sorted).

### 5. Take/while pattern

```go
func takeWhile[T any](seq iter.Seq[T], pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        next, stop := iter.Pull(seq)
        defer stop()
        for {
            v, ok := next()
            if !ok { return }
            if !pred(v) { return }
            if !yield(v) { return }
        }
    }
}
```

Yields until the predicate fails, then stops; the coroutine is canceled by stop.

## Anti-Patterns & Gotchas

**Forgetting `defer stop()`.** Goroutine leak.

**Calling `stop()` then `next()`.** After stop, next returns `(zero, false)` forever. Safe but pointless.

**Calling `next()` on a goroutine other than the one that created the Pull.** Pull is *not* goroutine-safe; the runtime enforces same-goroutine access.

**Pulling from a Pull from a Pull from a Pull (very deep).** Each adds overhead; benchmarks needed.

**Using `iter.Pull` when push iteration suffices.** A simple `for v := range seq` is cheaper.

**Pulling from a sequence that has side effects you don't want.** Push iteration may have started side effects (e.g., HTTP fetch) that you'd want to skip. Stop early via `stop()`.

**Iterating across a sequence that has internal cancellation logic.** The sequence's `yield` return value matters; ignoring it via Pull can prevent cleanup.

**Trying to use coroutines for general async work.** Not their purpose. Use goroutines + channels / `context.Context`.

**Sharing `next`/`stop` across goroutines.** Single-goroutine API. Treat them like file handles.

**Long-running Pull where the producer holds heavy state.** Memory pinned while coroutine is parked.

## Performance Notes

(Approximate, 1.23 era.)

- `iter.Pull` setup: ~µs (allocates coroutine).
- `next()` call: ~10-50 ns.
- `stop()` call: ~µs.
- Channel-based equivalent: ~µs per step.
- Range-over-func (push only): ~ns per yield.

If your sequence is short and consumed in full, push (`for v := range seq`) is optimal.
If you need explicit `next()` semantics, `iter.Pull` is materially faster than channels.

## How Big Companies Use It

`iter.Pull` is 1.23+; adoption is just starting (as of mid-2026):

- **The Go team**: `slices.SortFunc` and friends rely on push iteration; pull is used in new helpers.
- **Klauspost's compression libs**: experimenting with iter.Pull for decoder state.
- **CockroachDB**: evaluating for some streaming SQL operators.
- **Standard library internal**: helpers in `database/sql` reportedly use it for some streaming results.
- **gRPC-Go**: still uses channels in most places; iter.Pull adoption discussed but not widespread.

## Source Code References

Pinned to `go1.26`.

- `iter.Pull` / `iter.Pull2`: [`src/iter/iter.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/iter/iter.go).
- Coroutine runtime: [`src/runtime/coro.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/coro.go).
- range-over-func compiler support: [`src/cmd/compile/internal/rangefunc/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/rangefunc).
- Proposal #61897 (range-over-func): https://github.com/golang/go/issues/61897.
- Proposal #61405 (iter.Pull): https://github.com/golang/go/issues/61405.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Range Over Function Types" (Go 1.23 release blog): https://go.dev/blog/range-functions.
- Russ Cox, "Coroutines for Go": https://research.swtch.com/coro.
- "iter package" docs: https://pkg.go.dev/iter.
- Proposal #61405: https://github.com/golang/go/issues/61405.
- "Iterators in Go 1.23" — various community write-ups.
- "Coroutines vs channels in Go" — benchmarks: blog posts, 2023-2024.
- Damian Gryski, "Range over function performance" — go-perfbook: https://github.com/dgryski/go-perfbook.

## Exercises / Self-Check

1. Implement `zip` from scratch using `iter.Pull`. Verify it stops correctly when either side ends.
2. Why is `defer stop()` required? Construct a code path where omitting it leaks a goroutine until GC.
3. Compare `iter.Pull` to a channel-based equivalent on a 1M-element sequence. Measure throughput.
4. Implement `chunked(seq, 100)` that yields slices of 100 elements. Test with iterators that don't divide evenly.
5. Why is the coroutine API in `iter.Pull` not exposed as a general `runtime.NewCoro` function? Articulate the design rationale.
