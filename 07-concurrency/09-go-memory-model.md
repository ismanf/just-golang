# The Go Memory Model

## TL;DR

The Go memory model defines when one goroutine is **guaranteed** to observe writes made by another. The key relation is **happens-before**: if event A happens-before event B, B sees the effects of A. Synchronization primitives (channels, `sync.*`, atomics, `go`/`Goexit`, `init`) all establish happens-before edges; nothing else does. If you read a memory location while another goroutine writes it without such an edge, that is a **data race**, and the program's behavior is undefined. Since **Go 1.19** atomics are explicitly sequentially consistent. The biggest gotcha: "I haven't observed the bug" is not a proof of correctness — the race detector is the proof.

## Mental Model

```
program order:        A1 -> A2 -> A3 (within one goroutine)

happens-before edges (across goroutines) come ONLY from:
  - channel send  HB  matching receive
  - close(ch)     HB  receive that returns ok=false
  - unlock(mu)    HB  next lock(mu)
  - RUnlock       HB  next Lock
  - sync.Once.Do  HB  every later Do return
  - WaitGroup.Done HB Wait return
  - sync.Cond.Signal/Broadcast HB matching Wait return
  - the n-th atomic on x HB  the (n+1)-th atomic on x (total order, since 1.19)
  - go f()        HB  the start of f
  - end of init() HB  start of main()
  - end of goroutine's last statement HB nothing (no implicit join)

Without an HB edge, "later in wall-clock time" does NOT imply "visible".
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"sync"
)

// Bad: data race
func badExample() {
	var x int
	var done bool
	go func() {
		x = 42
		done = true
	}()
	for !done { }
	fmt.Println(x) // race; may print 0, 42, or anything
}

// Good: explicit synchronization
func goodExample() {
	var x int
	var mu sync.Mutex
	var done bool
	go func() {
		mu.Lock(); x = 42; done = true; mu.Unlock()
	}()
	for {
		mu.Lock()
		d := done
		v := x
		mu.Unlock()
		if d { fmt.Println(v); return }
	}
}
```

`badExample` has *no* happens-before edge from the writer's `x = 42` to the reader's read of `x`. The race detector flags both `done` and `x`. The fact that `for !done {}` "eventually" sees `done=true` on x86 is an artifact of cache coherence — *not* a guarantee from the model.

## Deep Dive

### Happens-before, formally

For two events A and B, A happens-before B if any of:
- A and B are in the same goroutine and A appears before B in program order.
- (Synchronization edges as listed above.)
- Transitivity: A HB B and B HB C → A HB C.

Two events conflict if they access the same memory location and at least one is a write. A program has a **data race** when two conflicting events have no happens-before relationship between them. **Race-free programs see sequentially consistent execution.** Racy programs see whatever the compiler and hardware decide — including out-of-thin-air torn reads on some architectures, though Go's spec carefully prohibits the worst forms.

### The channel rules

- A send on a channel happens-before the corresponding receive *completes*.
- The k-th receive from a channel with capacity C happens-before the (k+C)-th send completes.
- A close of a channel happens-before a receive that observes the channel as closed (returning ok=false / the zero value).

The first rule covers unbuffered handoffs. The second covers buffered channels: filling the buffer creates a back-pressure edge that lets a sender know the buffer drained. The third makes `close` a safe broadcast.

### The mutex rules

- For any `*Mutex` or `*RWMutex` variable `m`: call n of `m.Unlock` happens-before call n+1 of `m.Lock`.
- For any `*RWMutex`: for any call to `m.RLock`, there is an n such that the `m.RLock` happens-after call n of `Unlock`, and the matching `m.RUnlock` happens-before call n+1 of `m.Lock`.

So: anything done before `Unlock` is visible to whoever next calls `Lock`. Anything done before a writer's `Unlock` is visible to whoever next `RLock`s.

### The once rule

`o.Do(f)`: the completion of the (only) call to `f` happens-before any subsequent `o.Do` returns. So the result of `f` is safely published to every caller.

### The atomic rule (Go 1.19+)

Operations on `sync/atomic` variables (and the typed wrappers) are part of a single total order that is consistent with the happens-before order. In other words: sequential consistency. This is stronger than the C++ default and stronger than what Go promised pre-1.19.

This means:

```go
var x atomic.Int64
var y atomic.Int64

// Goroutine 1:
x.Store(1)
r1 := y.Load()

// Goroutine 2:
y.Store(1)
r2 := x.Load()

// At least one of r1, r2 must be 1.
```

Pre-1.19 the spec did not guarantee that — both could be 0 (independent reordering). Since 1.19, the total order forbids it.

### The goroutine rule

`go f(args)`: the `go` statement happens-before the start of `f`. Captured variables and arguments are safely visible to the new goroutine.

There is **no** happens-before edge from the end of a goroutine back to anything in the parent. If you want to know a goroutine has finished, use `WaitGroup`, channel, `errgroup`, etc.

### init order

For two packages P and Q where P imports Q: all of Q's `init()` functions complete before any of P's begin. `package main`'s `init` completes before `main` starts. Concurrent reads of package-level variables initialized in `init` are safe because the writes happen before `main` (and thus before any goroutines `main` spawns).

### What is NOT a happens-before edge

- A `for` loop with no synchronization. `for !done {}` is the textbook race.
- A `time.Sleep`. Sleeping does not establish memory visibility.
- A `runtime.Gosched`. Same.
- `fmt.Println`. Yes, it locks internally, but you can't rely on that for cross-goroutine visibility of *other* variables (it's an implementation detail, plus the lock is in a different module).
- "Wall-clock time elapsed" — *especially* on weak-memory ISAs (ARM64, POWER), where store buffers and write reordering can hide updates indefinitely.

### Word tearing

Go does not guarantee that reads/writes of types larger than the word size are atomic. A `struct{ a, b int }` read concurrently with a write can see a half-updated state on any architecture. Even an `int64` on 32-bit is not atomic without `sync/atomic`. Use atomics or locks.

For aligned `int` and pointer-sized values, single reads/writes are themselves not torn on supported architectures — but that does **not** make them safe; a non-atomic read of a value written without a barrier may still see a stale value, and the compiler may reorder it. **There is no benign data race in Go.**

### Compiler and CPU reordering

The compiler may reorder reads and writes within a goroutine as long as the as-if rule holds — that is, the goroutine's own observed behavior is preserved. The CPU may reorder loads and stores too. The memory model says: across goroutines, only happens-before edges constrain what is observable. Without them, anything goes.

### The race detector is the standard

The Go race detector (`go test -race`, `go build -race`) uses ThreadSanitizer to track happens-before edges at runtime. If a data race is observed, it's reported with both stack traces. **It can have false negatives** (a race not observed at this run is still a bug), but **no false positives** — a flagged race is real.

Best practice: run all tests with `-race` in CI. Many bugs only show up under `-race` because TSan slows the program enough to interleave threads differently.

## Standard Library Hooks

- Every `sync.*` and `sync/atomic` primitive establishes an edge.
- Every channel op establishes an edge.
- `runtime.Goexit` does **not** establish a join edge to the parent; the goroutine just dies.
- `runtime.GC` is not a synchronization primitive (it does internal barriers, but you cannot use it for happens-before in your code).
- `time.Sleep` is not a synchronization primitive.
- The `go vet` `atomic` analyzer catches some common atomic misuses.
- `runtime.SetCPUProfileRate`, `runtime.MemProfileRate` etc. are not synchronization primitives.

## Real-World Patterns

### 1. Publish via atomic.Pointer

```go
package config

import "sync/atomic"

var current atomic.Pointer[Config]

type Config struct{ Host string }

func Load() *Config         { return current.Load() }
func Replace(c *Config)     { current.Store(c) }
```

A reader who calls `Load()` and then dereferences `Host` is safe: the atomic load establishes happens-after the atomic store, and the writer fully populated the `Config` value before calling `Store`. The dereference picks up a published, never-mutated `Host`.

### 2. Publish via close

```go
type Result struct{ value int }

func producer() (<-chan struct{}, *Result) {
	done := make(chan struct{})
	r := &Result{}
	go func() {
		r.value = compute()
		close(done) // close HB any receive on `done`
	}()
	return done, r
}

func consume() int {
	done, r := producer()
	<-done
	return r.value // safe: close HB this receive HB this read
}
```

`close(done)` happens-after the `r.value = compute()`. The consumer's `<-done` happens-after the close. By transitivity, the consumer's read of `r.value` happens-after the producer's write.

### 3. Publish via WaitGroup

```go
func crawl(urls []string) []Result {
	var wg sync.WaitGroup
	results := make([]Result, len(urls))
	for i, u := range urls {
		wg.Go(func() { results[i] = fetch(u) })
	}
	wg.Wait()
	return results // safe: each Done HB Wait return; main can read results
}
```

Each goroutine's `wg.Done()` happens-before the parent's `wg.Wait()` return. Each goroutine writes to its own index, so no two goroutines conflict.

### 4. Mutex-protected hand-off

```go
type Buffer struct {
	mu  sync.Mutex
	buf []byte
}

func (b *Buffer) Set(p []byte) {
	b.mu.Lock()
	b.buf = append(b.buf[:0], p...)
	b.mu.Unlock()
}

func (b *Buffer) Get() []byte {
	b.mu.Lock()
	defer b.mu.Unlock()
	return slices.Clone(b.buf) // copy so caller can't race
}
```

The classic "guard everything with a Mutex" pattern is correct by construction: any pair of Set/Get is serialized through `mu`.

## Anti-Patterns & Gotchas

**"It works on x86, must be fine."** x86 has Total Store Order, the friendliest model. On ARM64 or POWER, the same code may fail. The Go memory model is what is portable; the hardware is just an implementation.

**Using `time.Sleep` to "wait" for a goroutine.** No HB edge. Flaky on faster machines, broken on slower ones.

**Reading and writing a `bool` flag without sync.** "It's just a bool, surely it's atomic." Not portable, and the compiler can hoist the read out of the loop (`for !done {}` becomes `for !registerValue {}`). Use `atomic.Bool`.

**Embedding a `Mutex` and forgetting the value-copy ban.** Copying the struct silently splits the lock. `go vet -copylocks` catches it.

**Closing a channel for synchronization, then sending after.** Closing wakes everyone; subsequent sends panic. Use a "done" channel separate from data channels.

**Using `runtime.Gosched()` to "let other goroutines see".** Not a barrier. Adds noise.

**Multiple writers to a shared slice.** Even appending is a race; the slice header is three words, not atomic.

**Believing `var v atomic.Pointer[T]` means dereferences are atomic.** The pointer **load** is atomic; what the pointer points at is whatever the writer published. If the writer mutates the pointee after publishing, readers race.

**Caching `ctx.Done()` and assuming reads are atomic.** Reads from a channel are atomic but the chan value itself is not — capture once, use everywhere.

## Performance Notes

- Synchronization is not free: each atomic, lock, or channel op has a cost. But "skip the sync to go faster" is wrong — you trade correctness for nothing on a busy box because cache coherence will dominate anyway.
- Happens-before edges on x86 are usually plain MOV (for atomics) + `LOCK` prefix where needed. On ARM64 they cost more (dmb instructions).
- The race detector adds ~5–10× CPU and ~2× memory in tests. Reserve `-race` for CI and dev.
- Avoiding races at the design level (immutable data, owned-by-one-goroutine, channels) often gives the best perf because it removes both the mutex contention and the cache-line bouncing.

## How Big Companies Use It

- **Russ Cox's "Updating the Go Memory Model"** (2022) is *the* primary source for the modern model: https://research.swtch.com/gomm.
- **Kubernetes** has internal docs requiring `-race` in CI; race regressions are blocking.
- **CockroachDB** maintains a "memory model lints" doc and ensures every shared variable has documented synchronization.
- **Cloudflare** runs production binaries built with `-race` in a small fraction of pods to catch latent races.
- **The Go team itself** found multiple races in the runtime by enabling TSan; see commit history under [`runtime/race`](https://github.com/golang/go/tree/master/src/runtime/race).
- **Tailscale** has a strict "no benign data races" policy; the codebase passes `-race` everywhere.

## Source Code References

Pinned to `go1.26`.

- The memory model document: [`doc/articles/race_detector.html`](https://github.com/golang/go/blob/master/doc/articles/race_detector.html) and the live one at https://go.dev/ref/mem.
- Race detector integration: [`src/runtime/race/`](https://github.com/golang/go/tree/master/src/runtime/race).
- Atomic ordering implementation: [`src/sync/atomic/`](https://github.com/golang/go/tree/master/src/sync/atomic).
- Channel happens-before in code: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go) — read the `// raceacquire` calls.
- WaitGroup HB: [`src/sync/waitgroup.go`](https://github.com/golang/go/blob/master/src/sync/waitgroup.go) — `race.Acquire`/`race.Release` calls.

## Further Reading

- Go memory model (canonical): https://go.dev/ref/mem
- Russ Cox, "Updating the Go Memory Model" (2022): https://research.swtch.com/gomm
- Russ Cox, "Hardware memory models" (2021): https://research.swtch.com/hwmm
- Russ Cox, "Programming language memory models" (2021): https://research.swtch.com/plmm
- Hans Boehm, "How to miscompile programs with 'benign' data races" (PLDI 2011)
- Dmitry Vyukov, "Race detector design": https://github.com/golang/go/blob/master/src/runtime/race/README
- Sergey Adam, ThreadSanitizer paper: https://research.google/pubs/pub35604/

## Exercises / Self-Check

1. Sketch the happens-before edges in: G1 does `mu.Lock(); x=1; mu.Unlock()`; G2 does `mu.Lock(); print(x); mu.Unlock()`. Where is the edge?
2. Why is `for !done {}` (with `done` a plain `bool` set by another goroutine) broken even on x86?
3. Given an unbuffered channel and a single send/recv pair, prove that anything the sender wrote before the send is visible to the receiver after the recv.
4. What changed between the pre-1.19 and 1.19+ atomic semantics? Construct an example where it matters.
5. Run `go test -race` on a deliberately racy program. What does the report look like and how do you map it back to the source?
