# Range-Over-Func — Advanced Patterns

## TL;DR

Range-over-func, shipped in Go 1.23 (proposal [#61897](https://github.com/golang/go/issues/61897)), lets you write `for v := range myFn` where `myFn` is one of:
- `func(yield func() bool)` — no value (rare).
- `func(yield func(V) bool)` — `iter.Seq[V]`.
- `func(yield func(K, V) bool)` — `iter.Seq2[K, V]`.

The iterator function calls `yield(value)`; if `yield` returns `false`, iteration is aborted (e.g., `break`). It's Go's *push-style* iterator, distinct from `iter.Pull` (the pull-style adapter — see `24-frontier/02-coroutines-and-iter-pull.md`). The single biggest gotcha: **`yield` must be checked**. Returning early without checking `yield`'s result means you keep doing work the consumer doesn't want. Always `if !yield(v) { return }`.

## Mental Model

```
   Consumer:
       for v := range producer {
           if v == "stop" { break }   // breaks loop
           use(v)
       }
   
   Producer:
       func producer(yield func(string) bool) {
           for _, v := range source {
               if !yield(v) { return }    // consumer broke; we stop
           }
       }
   
   Flow:
       Each `yield(v)` enters consumer's body once.
       Consumer's loop runs, possibly modifies state, then returns to yield.
       If yield returns false, producer should stop iteration.
       Defer in producer fires on function exit (whenever that happens).
```

This is **synchronous** push iteration — no goroutine, no channel. The yield function is a closure that runs the consumer's body.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"iter"
)

// A simple iterator producing the first n integers.
func count(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) { return }
		}
	}
}

func main() {
	for v := range count(5) {
		if v == 3 { break }
		fmt.Println(v)
	}
	// Output:
	// 0
	// 1
	// 2
}
```

`count(5)` returns `iter.Seq[int]`. The `for ... range` loop drives it. `break` makes `yield` return false; the iterator function exits.

## Deep Dive

### The three iterator signatures

```go
// Used as `for range seq` (no value)
type SeqUnary func(yield func() bool)

// Used as `for v := range seq` (one value)
type Seq[V any] func(yield func(V) bool)        // == iter.Seq[V]

// Used as `for k, v := range seq` (two values)
type Seq2[K, V any] func(yield func(K, V) bool) // == iter.Seq2[K, V]
```

The compiler matches the receiving range expression to the right signature.

### Why "push" iterators

Most Go developers had used channels for streaming. Range-over-func is the *push* version of an iterator API: the producer hands values to the consumer one at a time.

Pros over channels:
- No goroutine; cheaper.
- No leak risk (no channel to close).
- Cleaner cancellation (`yield` returns false; producer reacts).

Cons:
- Producer and consumer run in the same goroutine; producer is blocked while consumer's body runs.
- Hard to compose with select (no channels).
- Different mental model: producer code can be hard to read with the `yield` parameter.

### `yield` return value semantics

```go
for v := range seq {
    if condition { break }
    use(v)
}
```

When `break` (or `return`) happens inside the loop, the compiler arranges for `yield(v)` to return `false`. The iterator must observe this and stop:

```go
func seq(yield func(int) bool) {
    for i := 0; i < 100; i++ {
        if !yield(i) { return }  // CRUCIAL — abandon early
        // ... cleanup, more work ...
    }
}
```

Forgetting to check `yield`'s return value means you keep doing work after the consumer is gone. That work usually has side effects (open files, network calls); skipping the check leaks.

### Cleanup with `defer`

```go
func fileLines(path string) iter.Seq[string] {
    return func(yield func(string) bool) {
        f, err := os.Open(path)
        if err != nil {
            yield("error: " + err.Error())
            return
        }
        defer f.Close()        // runs when the iterator function returns

        sc := bufio.NewScanner(f)
        for sc.Scan() {
            if !yield(sc.Text()) { return }
        }
    }
}

for line := range fileLines("/etc/hosts") {
    if strings.HasPrefix(line, "#") { continue }
    fmt.Println(line)
}
```

`defer f.Close()` runs when the iterator function exits — whether normally, via `yield` returning false, or via panic.

This is *the* clean-cleanup mechanism for range-over-func. Use it.

### Iterator transformations

Chainable transforms:

```go
func Map[T, U any](seq iter.Seq[T], f func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range seq {
            if !yield(f(v)) { return }
        }
    }
}

func Filter[T any](seq iter.Seq[T], pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range seq {
            if pred(v) {
                if !yield(v) { return }
            }
        }
    }
}

func Take[T any](seq iter.Seq[T], n int) iter.Seq[T] {
    return func(yield func(T) bool) {
        i := 0
        for v := range seq {
            if i >= n { return }
            if !yield(v) { return }
            i++
        }
    }
}
```

Each is a function returning an `iter.Seq[T]`. Compose: `Take(Filter(Map(seq, f), pred), 10)`. Lazy: no intermediate slice.

### Iterator from existing collections

`slices.Values`, `slices.All`, `maps.Keys`, `maps.Values`, `maps.All` return iterators directly. Examples:

```go
import (
    "iter"
    "slices"
    "maps"
)

s := []int{10, 20, 30}
for i, v := range slices.All(s) {  // iter.Seq2[int, int]
    fmt.Println(i, v)
}

m := map[string]int{"a": 1, "b": 2}
for k := range maps.Keys(m) { fmt.Println(k) }
for v := range maps.Values(m) { fmt.Println(v) }
for k, v := range maps.All(m) { fmt.Println(k, v) }
```

These compose with `Map`/`Filter` cleanly.

### Stateful iterators

```go
func dedup[T comparable](seq iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        seen := map[T]struct{}{}
        for v := range seq {
            if _, ok := seen[v]; ok { continue }
            seen[v] = struct{}{}
            if !yield(v) { return }
        }
    }
}
```

State (the `seen` map) lives in the iterator function's closure. Each call to the iterator creates a fresh map.

### Generator-style iterators

```go
func fibonacci() iter.Seq[int] {
    return func(yield func(int) bool) {
        a, b := 0, 1
        for {
            if !yield(a) { return }
            a, b = b, a+b
        }
    }
}

for n := range Take(fibonacci(), 10) {
    fmt.Println(n)
}
```

Infinite iterator + bounded `Take` = first 10 Fibonacci numbers. Mostly clean expression.

### Tree / graph traversal

```go
type Node struct {
    Value    int
    Children []*Node
}

func (n *Node) Walk() iter.Seq[int] {
    return func(yield func(int) bool) {
        var walk func(*Node) bool
        walk = func(node *Node) bool {
            if node == nil { return true }
            if !yield(node.Value) { return false }
            for _, c := range node.Children {
                if !walk(c) { return false }
            }
            return true
        }
        walk(n)
    }
}

for v := range tree.Walk() {
    fmt.Println(v)
}
```

Recursive walk; `yield` propagates through the recursion via the returned bool.

### Two-value iterators (`iter.Seq2`)

```go
func enumerate[T any](seq iter.Seq[T]) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        i := 0
        for v := range seq {
            if !yield(i, v) { return }
            i++
        }
    }
}

for i, v := range enumerate(slices.Values([]string{"a", "b", "c"})) {
    fmt.Println(i, v)
}
```

### Combining iterators

```go
func chain[T any](seqs ...iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        for _, s := range seqs {
            for v := range s {
                if !yield(v) { return }
            }
        }
    }
}

all := chain(slices.Values([]int{1, 2}), slices.Values([]int{3, 4}))
for v := range all { fmt.Println(v) }
```

Chain multiple iterators into one stream.

### Iterating with errors

Range-over-func doesn't have built-in error propagation. Conventions:

#### Option 1: return error as `iter.Seq2[T, error]`

```go
func lines(path string) iter.Seq2[string, error] {
    return func(yield func(string, error) bool) {
        f, err := os.Open(path)
        if err != nil {
            yield("", err)
            return
        }
        defer f.Close()
        sc := bufio.NewScanner(f)
        for sc.Scan() {
            if !yield(sc.Text(), nil) { return }
        }
        if err := sc.Err(); err != nil {
            yield("", err)
        }
    }
}

for line, err := range lines("/etc/hosts") {
    if err != nil { /* handle */; break }
    fmt.Println(line)
}
```

Each iteration is `(value, error)`. Caller checks error per iteration.

#### Option 2: store error on a wrapper

```go
type Stream struct {
    seq iter.Seq[string]
    err error
}

func (s *Stream) Iter() iter.Seq[string] { return s.seq }
func (s *Stream) Err() error             { return s.err }
```

Iterator sets `s.err` on failure; consumer checks after loop.

The community is still settling on conventions; both styles appear.

### Compile-time vs runtime checks

The Go compiler validates iterator function signatures at compile time. A function `func(yield func(int) bool)` is recognized as `iter.Seq[int]`.

But: forgetting `if !yield(v) { return }` is not a compile error. Only runtime behavior (extra work after break) reveals it. Linters are starting to flag this.

### Re-entrancy and reset

An `iter.Seq[T]` can be iterated multiple times. Each `for range` call invokes the iterator function fresh:

```go
seq := count(3)
for v := range seq { fmt.Println(v) }  // 0 1 2
for v := range seq { fmt.Println(v) }  // 0 1 2 again
```

State inside the iterator is created fresh each iteration. If the iterator wraps a file, opening it twice; if wrapping a channel, draining it twice (which fails after the first if the channel is closed).

### Performance characteristics

Range-over-func is **inlined** in many cases by the compiler. Performance is close to a hand-written for loop:

- Per-yield call: ~1-2 ns.
- Function-call overhead for the iterator: amortized.
- No closure allocation per iteration if yield is recognized as inline.

Compared to channels: 10-100× faster.
Compared to slice append + range: similar speed; advantage is laziness (no intermediate slice).

### Compatibility gotcha

Range-over-func requires `go 1.23` in `go.mod`. With an older `go` directive, the compiler rejects iterator literals.

```
./main.go:5:6: range over s (variable of type func(yield func(int) bool)) requires go1.23 or later
```

Bump your `go.mod`'s `go` directive.

### `for range function` syntax

You can write:

```go
for v := range func(yield func(int) bool) {
    for i := 0; i < 5; i++ {
        if !yield(i) { return }
    }
} {
    fmt.Println(v)
}
```

Inline iterator function. Useful in rare cases; usually a named function is cleaner.

## Standard Library Hooks

- `iter` package: [`pkg.go.dev/iter`](https://pkg.go.dev/iter).
- `slices.All`, `slices.Values`, `slices.Backward`, `slices.Sorted`, etc.
- `maps.All`, `maps.Keys`, `maps.Values`.
- `strings.Lines` (1.24+).
- `strings.SplitSeq`, `bytes.SplitSeq` (1.24+) — lazy alternatives to `Split`.
- `bufio.Scanner` is *not* an iterator; wrap it.
- Third-party adapters: `samber/lo` and others.

## Real-World Patterns

### 1. Read-and-process file line by line

```go
func nonComments(path string) iter.Seq2[string, error] {
    return func(yield func(string, error) bool) {
        f, err := os.Open(path)
        if err != nil { yield("", err); return }
        defer f.Close()
        sc := bufio.NewScanner(f)
        for sc.Scan() {
            line := sc.Text()
            if strings.HasPrefix(line, "#") || line == "" { continue }
            if !yield(line, nil) { return }
        }
        if err := sc.Err(); err != nil { yield("", err) }
    }
}

for line, err := range nonComments("/etc/hosts") {
    if err != nil { log.Println(err); break }
    process(line)
}
```

### 2. Paginated API as iterator

```go
func listAll(client *Client) iter.Seq[Item] {
    return func(yield func(Item) bool) {
        cursor := ""
        for {
            page, next, err := client.List(cursor)
            if err != nil { return }    // could yield with error variant
            for _, item := range page {
                if !yield(item) { return }
            }
            if next == "" { return }
            cursor = next
        }
    }
}

for item := range listAll(client) {
    process(item)
    if shouldStop { break }
}
```

Lazy pagination: only fetches the next page when needed.

### 3. Database query results

```go
func queryAll(ctx context.Context, db *sql.DB, q string, args ...any) iter.Seq2[User, error] {
    return func(yield func(User, error) bool) {
        rows, err := db.QueryContext(ctx, q, args...)
        if err != nil { yield(User{}, err); return }
        defer rows.Close()
        for rows.Next() {
            var u User
            if err := rows.Scan(&u.ID, &u.Name); err != nil {
                yield(User{}, err); return
            }
            if !yield(u, nil) { return }
        }
        if err := rows.Err(); err != nil { yield(User{}, err) }
    }
}
```

`defer rows.Close()` cleans up even on early break.

### 4. Composed transforms

```go
import (
    "iter"
    "slices"
)

words := []string{"hello", "world", "foo", "bar"}
pipeline := Take(
    Map(
        Filter(
            slices.Values(words),
            func(s string) bool { return len(s) <= 4 },
        ),
        strings.ToUpper,
    ),
    2,
)

for w := range pipeline {
    fmt.Println(w)
}
// Output:
// FOO
// BAR
```

Concise pipeline; lazy.

### 5. Cancellation via context

```go
func cancellable[T any](ctx context.Context, seq iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range seq {
            select {
            case <-ctx.Done(): return
            default:
            }
            if !yield(v) { return }
        }
    }
}
```

Wraps any iterator with context-cancellation. Stops at the next yield after cancel.

## Anti-Patterns & Gotchas

**Forgetting `if !yield(v) { return }`.** You keep doing work after the consumer broke out. Side effects leak.

**Using channels for "iterator" when range-over-func suffices.** Channels are heavier and prone to leak.

**Long-lived iterators holding open resources.** Document; users must call `defer close(...)` or use early-break carefully.

**Iterating over a non-reusable iterator twice.** Sometimes the producer reads a file, network stream, etc. Document.

**Hidden state across yields.** Each `yield(v)` re-enters the loop's body. If the producer mutates shared state in a surprising way, the consumer might be in an inconsistent state.

**Iterators that allocate per call.** Each yield should be cheap. If you build a slice and yield items from it, you've defeated laziness.

**Returning errors via panic.** Don't. Use `iter.Seq2[T, error]` or stored-error pattern.

**Combining `iter.Pull` and `iter.Seq` carelessly.** Pull's coroutine plus a Seq's recursion can stack deeply.

**Synchronous iterator that blocks on I/O without ctx.** No way to cancel cleanly; consumer breaks but I/O is in progress.

**Calling `yield(v)` from a goroutine other than the iterator's.** The yield function captures the consumer's loop state; calling from another goroutine is unsafe.

## Performance Notes

(1.23+ benchmarks; vary by workload.)

- Simple `for i := 0; i < n; i++` for loop: baseline.
- Range over slice: similar speed.
- Range over `slices.Values(s)`: similar (within 1-5%).
- Custom iterator function: similar, sometimes 5-10% slower due to function call overhead (compiler often inlines).
- Composed pipeline (Map ∘ Filter ∘ Take): roughly 2-3× faster than allocating intermediate slices.
- Channel-based equivalent: 10-100× slower.

`go test -bench` will tell you for your workload.

## How Big Companies Use It

1.23 is recent (Aug 2023); adoption is growing:

- **The Go team** added `iter.Pull`, `slices.Values`, `maps.All`, `strings.Lines`, `bytes.SplitSeq` to stdlib.
- **CockroachDB**: experimenting with iterators for SQL operators.
- **Library authors**: most popular libraries are gradually adding `iter.Seq` overloads.
- **Klauspost's compress libs**: lazy decoder pipelines.
- **OpenTelemetry Go**: lazy metric collection.

The pattern is universally available; idioms are still settling.

## Source Code References

Pinned to `go1.26`.

- iter package: [`src/iter/iter.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/iter/iter.go).
- slices iterators: [`src/slices/iter.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/slices/iter.go).
- maps iterators: [`src/maps/iter.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/maps/iter.go).
- Compiler rangefunc support: [`src/cmd/compile/internal/rangefunc/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/rangefunc).
- strings.Lines etc.: [`src/strings/strings.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/strings/strings.go).
- bytes.SplitSeq: [`src/bytes/bytes.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/bytes/bytes.go).
- Proposal #61897: https://github.com/golang/go/issues/61897.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Range Over Function Types" (Go 1.23 release blog): https://go.dev/blog/range-functions.
- "Iterators in Go 1.23" — official doc updates.
- Proposal #61897: https://github.com/golang/go/issues/61897.
- "Custom iterators in Go" — community write-ups, 2023-2024.
- "Functional Go and iter.Seq" — talks at GopherCon 2024.
- Russ Cox, "Coroutines for Go": https://research.swtch.com/coro.
- Russ Cox, "Storing data in iterators" — research.swtch.com.
- Damian Gryski, "Range-over-func benchmarks": https://github.com/dgryski/go-perfbook.

## Exercises / Self-Check

1. Write an iterator that yields the prime numbers up to N. Use `Take` to get the first 10 primes.
2. Implement `lines(io.Reader) iter.Seq2[string, error]` that scans line by line and yields errors when the underlying reader fails.
3. Demonstrate the `if !yield(v) { return }` gotcha: write an iterator without the check, and show the side effects (e.g., a counter) that leak after break.
4. Compose Map ∘ Filter ∘ Take on a 1M-element source. Compare to slice-based equivalent for memory and time.
5. Implement `enumerate` (yields `(index, value)`). Why is `iter.Seq2[int, T]` the right return type rather than `iter.Seq[Pair[int, T]]`?
