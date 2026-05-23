# `iter` — Iterators (Range-over-Func, since 1.23)

## TL;DR

Go 1.23 added `range` over functions: `for v := range seqFunc { ... }` calls the function with a yield callback. The `iter` package defines the canonical signatures `iter.Seq[V]` and `iter.Seq2[K,V]`. Use iterators for lazy, composable streams (map/filter/take pipelines) without needing to materialize slices. Standard library APIs (`slices.All`, `maps.Keys`, `strings.SplitSeq` in 1.24+) increasingly expose iterators.

## Mental Model

```
iter.Seq[V]  = func(yield func(V) bool)
iter.Seq2[K,V] = func(yield func(K, V) bool)

The function calls yield once per element.
yield returns false → caller bailed; stop.
yield returns true → keep going.

range over the function:
    for v := range seq {
        // body
    }
calls seq(yield) where yield wraps the body.
break/return inside body → yield returns false → seq stops.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"iter"
)

func upTo(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) { return }
		}
	}
}

func main() {
	for v := range upTo(3) {
		fmt.Println(v)
	}
	// Output:
	// 0
	// 1
	// 2
}
```

## Deep Dive

### Defining a Seq

```go
type Seq[V any] func(yield func(V) bool)

func Filter[V any](s iter.Seq[V], pred func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range s {
			if !pred(v) { continue }
			if !yield(v) { return }
		}
	}
}
```

### `Seq2` for key-value

```go
func Enumerate[V any](s iter.Seq[V]) iter.Seq2[int, V] {
	return func(yield func(int, V) bool) {
		i := 0
		for v := range s {
			if !yield(i, v) { return }
			i++
		}
	}
}

for i, v := range Enumerate(upTo(3)) {
	fmt.Println(i, v)
}
```

### `iter.Pull` and `iter.Pull2`

Convert push-style to pull-style:

```go
next, stop := iter.Pull(upTo(5))
defer stop()
for {
	v, ok := next()
	if !ok { break }
	fmt.Println(v)
}
```

Lets you advance an iterator step-by-step or interleave multiple iterators. The implementation uses goroutines under the hood (via coroutines).

### Stdlib iterator producers

- `slices.All(s)` → `iter.Seq2[int, V]` — index/value pairs.
- `slices.Values(s)` → `iter.Seq[V]`.
- `slices.Backward(s)` → `iter.Seq2[int, V]` (reverse).
- `slices.Chunk(s, n)` → `iter.Seq[[]V]`.
- `maps.All(m)` → `iter.Seq2[K, V]`.
- `maps.Keys(m)`, `maps.Values(m)` → `iter.Seq[K]`, `iter.Seq[V]`.
- `strings.SplitSeq(s, sep)` (1.24+) → `iter.Seq[string]`.
- `strings.FieldsSeq(s)` (1.24+) → `iter.Seq[string]`.
- `bytes.SplitSeq`, `bytes.FieldsSeq` (1.24+).

### Stdlib iterator consumers

- `slices.Collect(seq)` → `[]V`.
- `slices.AppendSeq(dst, seq)` → `[]V`.
- `slices.Sorted(seq)`, `slices.SortedFunc(seq, cmp)`.
- `maps.Collect(seq2)` → `map[K]V`.
- `maps.Insert(m, seq2)` → mutate m.

### Composing pipelines

```go
import "iter"
import "slices"

func main() {
	seq := slices.Values([]int{1,2,3,4,5,6,7,8,9,10})
	even := Filter(seq, func(n int) bool { return n%2 == 0 })
	doubled := Map(even, func(n int) int { return n * 2 })
	for v := range doubled {
		fmt.Println(v)
	}
}

func Map[A, B any](s iter.Seq[A], f func(A) B) iter.Seq[B] {
	return func(yield func(B) bool) {
		for a := range s {
			if !yield(f(a)) { return }
		}
	}
}
```

Each step is lazy; no slice allocated.

### `break` behavior

When the loop body `break`s or returns, `yield` returns false. The iterator must return promptly — don't continue work after `yield` returns false.

### Side effects and cleanup

```go
func openLines(path string) iter.Seq[string] {
	return func(yield func(string) bool) {
		f, err := os.Open(path)
		if err != nil { return }
		defer f.Close()
		sc := bufio.NewScanner(f)
		for sc.Scan() {
			if !yield(sc.Text()) { return }
		}
	}
}
```

The `defer` runs on early termination as well — cleanup is built in.

### Errors

Iterators don't carry errors. Common patterns:

- `iter.Seq2[V, error]` — yield value-or-error pairs.
- Set a field on a parent struct: `var err error; for v := range producer.Seq() { ... } if producer.Err != nil { ... }` (like `bufio.Scanner`).

```go
type Lines struct { Err error }
func (l *Lines) Seq(r io.Reader) iter.Seq[string] {
	return func(yield func(string) bool) {
		sc := bufio.NewScanner(r)
		for sc.Scan() {
			if !yield(sc.Text()) { return }
		}
		l.Err = sc.Err()
	}
}
```

## Standard Library Hooks

- `iter` — `Seq`, `Seq2`, `Pull`, `Pull2`.
- `slices` — many iterator producers/consumers.
- `maps` — `All`, `Keys`, `Values`, `Collect`, `Insert`.
- `strings`, `bytes` (1.24+) — `SplitSeq`, `FieldsSeq`.

## Real-World Patterns

### 1. Lazy filter-map-take pipeline

```go
seq := slices.Values(users)
seq = Filter(seq, func(u User) bool { return u.Active })
ids := Map(seq, func(u User) int { return u.ID })
first10 := Take(ids, 10)
for id := range first10 { fmt.Println(id) }
```

No intermediate slices; total work is bounded by 10.

### 2. Read a file line by line as an iterator

```go
for line := range openLines("log.txt") {
	process(line)
}
```

Cleaner than `bufio.Scanner` boilerplate.

### 3. Chunk a slice

```go
import "slices"

for batch := range slices.Chunk(items, 100) {
	bulkInsert(batch)
}
```

Use case: paginating database inserts.

### 4. Reverse iteration

```go
for i, v := range slices.Backward(s) {
	fmt.Println(i, v)
}
```

### 5. Concurrent producer with `iter.Pull`

```go
next, stop := iter.Pull(stream)
defer stop()
for {
	v, ok := next()
	if !ok { break }
	select {
	case <-ctx.Done(): return ctx.Err()
	case out <- v:
	}
}
```

Use case: bridging a push-style iterator into a channel consumer.

## Anti-Patterns & Gotchas

**Ignoring `yield`'s return value.** Iterator keeps running after the consumer bailed.

**Calling `yield` in a goroutine.** Undefined behavior; must be in the iterator's call stack.

**Iterators with side effects that don't clean up on early return.** Use `defer` inside the iterator function.

**Storing the `iter.Seq` and iterating twice.** Some iterators are single-pass (the source is consumed). Document expectations.

**Forgetting that `iter.Pull` starts a goroutine.** Always call `stop` (defer it).

**Conflating `iter.Seq` with `chan`.** Iterators are synchronous; channels are concurrent.

**Returning errors via panic/recover in iterators.** Bad practice; carry errors in a struct field.

## Performance Notes

- `range`-over-func has overhead comparable to a function call per element (~ns).
- Slightly slower than `for i := range slice` (no inlining of `yield` body in many cases yet).
- `iter.Pull` allocates two goroutines and channels — usable but not zero-cost. Profile if used in hot loops.
- Pipelines compose without allocating intermediate slices — wins for streaming over very large inputs.

## How Big Companies Use It

- Adoption is recent (1.23 is the first stable release). Expect uptake to accelerate as 1.23+ becomes baseline.
- **`gopls`** uses iterators in newer analysis paths.
- **stdlib itself** — `slices` and `maps` packages are getting iterator counterparts steadily.
- **`x/exp`** community packages have many iterator helpers.

## Source Code References

Pinned to `go1.26`.

- `iter`: [`src/iter/iter.go`](https://github.com/golang/go/blob/master/src/iter/iter.go).
- Range-over-func in the compiler: [`src/cmd/compile/internal/rangefunc/`](https://github.com/golang/go/tree/master/src/cmd/compile/internal/rangefunc).
- `iter.Pull` coroutine implementation: [`src/runtime/coro.go`](https://github.com/golang/go/blob/master/src/runtime/coro.go).
- `slices.All`, `Values`, `Chunk`: [`src/slices/iter.go`](https://github.com/golang/go/blob/master/src/slices/iter.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/iter.
- Go 1.23 release notes — range over function: https://go.dev/doc/go1.23#language.
- Russ Cox, "Coroutines for Go" (2023): https://research.swtch.com/coro.

## Exercises / Self-Check

1. Write generic `Map[A,B]`, `Filter[V]`, `Take[V]`. Chain them.
2. Implement an iterator over file lines that propagates `Scanner.Err()` via a struct field.
3. Use `iter.Pull` to interleave two iterators round-robin.
4. Convert a function that returns `[]T` to one that returns `iter.Seq[T]`. Show memory savings on large inputs.
5. Why does the iterator stop when `yield` returns false? Trace through how `break` translates to that.
