# `sync.Mutex` and `sync.RWMutex`

## TL;DR

`sync.Mutex` is a non-reentrant, FIFO-ish exclusive lock with a fast path that compiles to a single CAS. `sync.RWMutex` allows many readers *or* one writer. The single biggest gotcha: **`RWMutex` is often slower than `Mutex`** unless the read critical section is large (microseconds or more) and read-heavy. Profile before reaching for it. Second gotcha: locks must not be copied — `go vet` catches most cases via `copylocks`.

## Mental Model

```
sync.Mutex (24 bytes):
    state uint32    // bit 0: locked, bit 1: woken, bit 2: starving,
                    // bits 3..: waiter count
    sema  uint32    // semaphore for parking

Fast path: CAS(state, 0, 1) — uncontended, ~10 ns.
Slow path: lockSlow — spin a few times, then runtime_SemacquireMutex.

sync.RWMutex (40 bytes):
    w           sync.Mutex   // serializes writers
    writerSem   uint32       // writer waits for active readers
    readerSem   uint32       // readers wait for departing writer
    readerCount atomic.Int32 // negative when a writer is pending
    readerWait  atomic.Int32 // readers a pending writer is waiting for
```

A reader fast-path is an atomic add on `readerCount`; if positive, no writer is pending — done. If negative, a writer is queued and the reader parks.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}

func main() {
	var wg sync.WaitGroup
	c := &Counter{}
	for range 100 {
		wg.Add(1)
		go func() { defer wg.Done(); c.Inc() }()
	}
	wg.Wait()
	fmt.Println(c.Value())
	// Output:
	// 100
}
```

The zero value of a `sync.Mutex` is an unlocked mutex — no constructor needed.

## Deep Dive

### Non-reentrant

A goroutine that locks a mutex it already holds **deadlocks**. There is no "reentrant lock" in the stdlib. The rationale: re-entrant locks paper over confused ownership; if you need re-entry, your call graph is wrong. Refactor: extract the inner work into a private method that assumes the lock is held (`fooLocked()` is the convention).

### Starvation mode (since Go 1.9)

Without intervention, a long queue of waiters can starve while incoming goroutines barge the lock during their spin. Go 1.9 introduced **starvation mode**:
- If a waiter is queued for more than **1 ms**, the mutex switches to starvation mode.
- In starvation mode, the lock is handed directly to the front-of-queue waiter; barging is disabled.
- The handoff persists until the waiter who got the lock observes either (a) it was the last waiter, or (b) it waited less than 1 ms — then normal mode resumes.

This gives bounded tail latency without sacrificing the fast-path throughput.

### Spinning

On contention, `lockSlow` will spin briefly (`runtime.sync_runtime_canSpin`) when:
- Running on a multi-core machine,
- There are other Ps idle,
- The spin count is below a threshold,
- We are not in starvation mode.

Spinning trades CPU for latency by avoiding a context switch. Tightly bounded; not a livelock risk.

### `TryLock` (since 1.18)

```go
if mu.TryLock() {
	defer mu.Unlock()
	// got it
} else {
	// someone else holds it
}
```

`TryLock` is "rarely the right thing" per the docs — it tempts you to invent your own scheduling. Real use cases: lock-free metrics, opportunistic flushes.

### `RWMutex` — reader counting

```go
var rw sync.RWMutex
rw.RLock(); defer rw.RUnlock() // reader
rw.Lock();  defer rw.Unlock()  // writer
```

Multiple readers can hold simultaneously, but a single writer excludes all readers and other writers. The implementation:
- `RLock`: atomic `++readerCount`. If negative (a writer is queued), park on `readerSem`.
- `Lock`: acquire `w` (writer Mutex) to serialize writers. Then `readerCount -= maxReaders` (making it negative); waits for outstanding readers on `writerSem`.
- `Unlock`: restore `readerCount`; wake all readers that were parked.

### Writer starvation

A pure-reader-heavy workload can starve writers — readers keep arriving and incrementing `readerCount` before the writer gets a slot. Mitigation: once `Lock` is called, *new* `RLock`ers will park because `readerCount` is negative — so the writer waits only for the readers already in flight. This is the writer-preference policy. Old readers still finish; new ones wait.

### When `RWMutex` is slower than `Mutex`

The reader fast path is still an atomic op on a shared cache line. With many cores, that cache line ping-pongs under heavy read contention — sometimes *worse* than the same line under `Mutex.Lock` because of the additional bookkeeping. Rule of thumb (corroborated by Damian Gryski's benchmarks):
- Read critical section < ~1 µs: `Mutex` usually wins.
- Read-heavy ratio > 100:1 *and* critical section > a few µs: `RWMutex` wins.
- Mixed workload: try `Mutex` first; it's simpler and you'll often discover the lock isn't the bottleneck.

For truly hot read paths, consider `atomic.Pointer[T]` with copy-on-write, or `sync/atomic` directly. See `07-sync-atomic.md`.

### Don't copy locks

```go
type Bad struct{ mu sync.Mutex }
b1 := Bad{}
b2 := b1 // COPIES the lock — both now have independent state
```

After the copy, locking `b1.mu` does not lock `b2.mu`. The `copylocks` analyzer in `go vet` flags this. The standard fix: store a pointer (`*Bad`) or embed `*sync.Mutex`. Better still, keep the lock as the **first** field and make the type explicitly non-copyable by embedding `noCopy` (an internal Go-team idiom):

```go
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

type Counter struct {
	_  noCopy
	mu sync.Mutex
	n  int
}
```

`go vet` honors the `noCopy` marker.

### Lock ordering

Acquiring multiple locks in different orders across goroutines deadlocks. The convention: define a strict global order (e.g., always lock `a` before `b`), and document it.

If you need to lock two arbitrary instances, sort by address:

```go
func lockBoth(a, b *Counter) {
	if uintptr(unsafe.Pointer(a)) < uintptr(unsafe.Pointer(b)) {
		a.mu.Lock(); b.mu.Lock()
	} else {
		b.mu.Lock(); a.mu.Lock()
	}
}
```

(`a == b` collapses to a single lock, which is reentrant — disaster. Add an `if a == b` guard.)

### `defer` is fine, mostly

`defer mu.Unlock()` was historically ~100 ns. Since Go 1.14 (open-coded defers), it's ~few ns for the common case. Use `defer` for clarity unless profiling proves otherwise.

## Standard Library Hooks

- `sync.Mutex`, `sync.RWMutex` — the two locks.
- `sync.Locker` interface — `Lock()` / `Unlock()`. Both `*Mutex` and `*RWMutex` implement it; `*RWMutex.RLocker()` returns a `Locker` adapter for the read side.
- `sync.Cond` — pairs with a `Locker`. See `05-sync-waitgroup-once-cond.md`.
- `go vet -copylocks` — flags struct copies and func-by-value passes that copy a lock.
- `runtime.SetMutexProfileFraction` — enables mutex contention profiling. View with `go tool pprof http://.../debug/pprof/mutex`.

## Real-World Patterns

### 1. Read-mostly cache with `RWMutex`

```go
package cache

import "sync"

type Cache[K comparable, V any] struct {
	mu sync.RWMutex
	m  map[K]V
}

func New[K comparable, V any]() *Cache[K, V] {
	return &Cache[K, V]{m: make(map[K]V)}
}

func (c *Cache[K, V]) Get(k K) (V, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	v, ok := c.m[k]
	return v, ok
}

func (c *Cache[K, V]) Set(k K, v V) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[k] = v
}
```

Classic shape. For read rates so high that the `RWMutex` line itself bottlenecks, swap to `atomic.Pointer[map[K]V]` with copy-on-write.

### 2. Copy-on-write replacement

```go
package cache

import "sync/atomic"

type COW[K comparable, V any] struct {
	p atomic.Pointer[map[K]V]
}

func (c *COW[K, V]) Get(k K) (V, bool) {
	m := *c.p.Load()
	v, ok := m[k]
	return v, ok
}

func (c *COW[K, V]) Set(k K, v V) {
	for {
		old := c.p.Load()
		next := make(map[K]V, len(*old)+1)
		for kk, vv := range *old {
			next[kk] = vv
		}
		next[k] = v
		if c.p.CompareAndSwap(old, &next) {
			return
		}
	}
}
```

Reads are lock-free. Writes are O(n) — only acceptable when reads vastly outnumber writes. Great for routing tables, feature flags, config maps.

### 3. Locked private struct, exposed via methods only

```go
type State struct {
	mu      sync.Mutex
	online  map[string]bool
	updated time.Time
}

func (s *State) MarkOnline(id string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.online[id] = true
	s.updated = time.Now()
}

func (s *State) Snapshot() (map[string]bool, time.Time) {
	s.mu.Lock()
	defer s.mu.Unlock()
	out := maps.Clone(s.online) // since 1.21
	return out, s.updated
}
```

Snapshot returns *clones* so the caller can't race on shared map. The lock never escapes.

### 4. Hand-rolled "fooLocked"

```go
func (s *State) Touch(id string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.touchLocked(id)
}

// touchLocked must be called with s.mu held.
func (s *State) touchLocked(id string) {
	s.updated = time.Now()
	s.online[id] = true
}
```

Naming convention `xxxLocked` documents the invariant. The Go runtime itself uses this style.

### 5. Lock-free counter via `atomic.Int64`

```go
type Counter struct{ n atomic.Int64 }

func (c *Counter) Inc() int64 { return c.n.Add(1) }
func (c *Counter) Load() int64 { return c.n.Load() }
```

When the only operation is integer arithmetic, skip the mutex entirely. See `07-sync-atomic.md`.

## Anti-Patterns & Gotchas

**Copying a struct that contains a mutex.** `go vet -copylocks` will catch most of these. The standard fix is to pass pointers.

**Lock held across a blocking call.** Holding `mu.Lock()` while reading from a slow network or sleeping pins every other contender. Keep critical sections small.

**Locking around `chan` operations.** A channel is already thread-safe. Locking around a send/recv is usually redundant and can introduce deadlocks (lock held, channel full, no one can drain because they're blocked on the lock).

**Recursive locking.** Deadlocks. Use the `xxxLocked` split.

**Forgetting to `Unlock` on every path.** `defer mu.Unlock()` immediately after `Lock()`. Worth a few ns to never get this wrong.

**Using `RWMutex` everywhere "because it's faster for reads".** It isn't, for short critical sections. Measure.

**Acquiring multiple locks in inconsistent orders.** Deadlock waiting to happen. Define a global ordering or use address-sorted locking.

**Embedding a Mutex in an exported type.** Now callers can `Lock`/`Unlock` it from outside, often breaking your invariants. Make the field unexported.

**Initializing a mutex with `new(sync.Mutex)` "to be safe".** Unnecessary — the zero value works.

## Performance Notes

- Uncontended `Lock`/`Unlock`: ~10–15 ns.
- Contended `Lock` that has to park: ~µs scale (the context switch dominates).
- `RWMutex.RLock`/`RUnlock` uncontended: ~15–20 ns (two atomic ops).
- `RWMutex.Lock` uncontended: ~25 ns.
- Cache-line ping-pong on a single `Mutex` from many cores can dominate everything else — splitting into multiple sharded mutexes is the standard fix (see `sync.Map` and `xsync.MPMC`).
- Mutex profile (`runtime.SetMutexProfileFraction(1)`) reports time goroutines spent blocked on locks. Cheap to enable in dev; sample (e.g., 1/100) in prod.

## How Big Companies Use It

- **Kubernetes** scales `RWMutex` to its limit in the API server's storage cacher. Some hot paths were rewritten to use `atomic.Pointer` for read-mostly state.
- **CockroachDB** uses sharded `sync.Mutex` across rangefeed registries; one global mutex would dominate.
- **Cloudflare's logd** documented how `RWMutex` contention showed up in mutex profile and was solved by switching to atomics: https://blog.cloudflare.com/.
- **etcd's mvcc store** uses fine-grained locks per key range to avoid global contention.
- **Discord's GC blog** mentions lock contention as a confounding factor in their Go-vs-Rust analysis: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.

## Source Code References

Pinned to `go1.26`.

- `Mutex` and `RWMutex` implementation: [`src/sync/mutex.go`](https://github.com/golang/go/blob/master/src/sync/mutex.go), [`src/sync/rwmutex.go`](https://github.com/golang/go/blob/master/src/sync/rwmutex.go).
- Starvation-mode design doc: https://go.dev/cl/34310 (Dmitry Vyukov, 2016).
- Runtime semaphore (`runtime_SemacquireMutex`): [`src/runtime/sema.go`](https://github.com/golang/go/blob/master/src/runtime/sema.go).
- `copylocks` analyzer: [`src/cmd/vet/copylock.go`](https://github.com/golang/go/blob/master/src/cmd/vet/copylock.go) (and the underlying `golang.org/x/tools/go/analysis/passes/copylock`).
- Mutex profile: [`src/runtime/mprof.go`](https://github.com/golang/go/blob/master/src/runtime/mprof.go).

## Further Reading

- Russ Cox, "Go Mutexes": https://research.swtch.com/mutex (and follow-ups)
- Dmitry Vyukov, starvation-mode CL description: https://go.dev/cl/34310
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns"
- Damian Gryski, "Mutex vs RWMutex" benchmarks: https://github.com/dgryski/go-perfbook
- Filippo Valsorda, "Go's sync.Mutex tour": https://words.filippo.io/
- Go memory model — sync: https://go.dev/ref/mem#sync

## Exercises / Self-Check

1. Why is `sync.Mutex` not reentrant? Sketch the deadlock that happens when you try.
2. Implement a sharded counter using N `Mutex`-protected sub-counters. When does the sharding help, and when does it not?
3. Profile (with `-bench` and `-mutexprofile`) a tight loop using `RWMutex` for a 50 ns critical section. Compare to `Mutex`. What do you see?
4. Why does `RWMutex.Lock` set `readerCount` negative? What invariant does that establish?
5. Show how `noCopy` and `go vet -copylocks` interact to prevent accidental copies.
