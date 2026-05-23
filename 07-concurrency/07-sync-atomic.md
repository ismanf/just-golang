# `sync/atomic`

## TL;DR

`sync/atomic` provides lock-free atomic operations on integers, pointers, and bools. Since **Go 1.19** there are typed wrappers (`atomic.Int64`, `atomic.Pointer[T]`, `atomic.Bool`) that prevent the classic alignment and mistyping bugs. **Go's atomics are sequentially consistent** (since 1.19, formalized in the memory model), so you don't need to reason about acquire/release — but they're still slower than a plain non-atomic op, and they don't serialize anything *other* than themselves. The biggest gotcha: an atomic on field A and a non-atomic on field B don't synchronize; if any goroutine writes a memory location without atomics or a mutex while another reads it, the race detector will fire and the result is undefined.

## Mental Model

```
atomic.Int64
+----------+
| _ noCopy |
| v int64  |  // 8-byte aligned
+----------+

Add(d):    LOCK XADD on x86
Load():    MOV (with appropriate barrier on weak-memory archs)
Store(v):  MOV + barrier / XCHG depending on arch
CAS(o,n):  LOCK CMPXCHG on x86
Swap(v):   XCHG

All atomic ops are total-order (sequentially consistent) across goroutines.
Non-atomic ops on the same memory are a data race.
```

The runtime guarantees 8-byte alignment for the typed atomic structs even on 32-bit architectures — the legacy `atomic.AddInt64(&x, 1)` on a non-aligned `*int64` would crash on 32-bit ARM.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n atomic.Int64

	var wg sync.WaitGroup
	for range 100 {
		wg.Go(func() { n.Add(1) }) // since 1.25
	}
	wg.Wait()

	fmt.Println(n.Load())
	// Output:
	// 100
}
```

Typed atomics (`atomic.Int32`, `Int64`, `Uint32`, `Uint64`, `Uintptr`, `Bool`, `Pointer[T]`) shipped in **Go 1.19** and are the preferred API. The legacy free functions (`atomic.AddInt64`, `atomic.LoadPointer`, etc.) still work but are less safe (no alignment guarantee, easy to misuse).

## Deep Dive

### The full type set (Go 1.19+)

```go
atomic.Int32     // Add, And, Or (1.23+), CompareAndSwap, Load, Store, Swap
atomic.Int64     // 8-byte aligned even on 32-bit
atomic.Uint32
atomic.Uint64
atomic.Uintptr
atomic.Bool      // Load, Store, Swap, CompareAndSwap
atomic.Pointer[T] // generic; T can be any type
atomic.Value      // pre-generic; stores any, requires consistent type
```

The methods (typed forms):

```go
v.Load() T
v.Store(new T)
v.Add(delta T) T          // returns new value (Int*/Uint*)
v.Swap(new T) (old T)
v.CompareAndSwap(old, new T) bool
v.And(mask T) T          // since 1.23 — returns old, atomically ANDs
v.Or(mask T) T           // since 1.23 — returns old, atomically ORs
```

### Sequential consistency (since 1.19)

Before Go 1.19 the memory model said atomics provided "the same semantics as the corresponding non-atomic operations" plus visibility. After Go 1.19, the memory model explicitly says: **atomic operations are sequentially consistent**. That means all atomic operations across all goroutines appear in some total global order consistent with program order. No fences, no acquire/release distinction in user code — just call the op.

This is stronger than C++ `memory_order_seq_cst` because it doesn't require an opt-in. Go made the choice deliberately (Russ Cox's memory model essay): the goal is to be easy to reason about, even at a modest perf cost.

### Alignment requirements (legacy functions)

```go
type Bad struct {
	a int32     // 4-byte aligned
	b int64     // on 32-bit: not 8-byte aligned!
}
atomic.AddInt64(&Bad{}.b, 1) // panic on 32-bit ARM
```

Typed atomics (`atomic.Int64`) embed alignment padding under the hood, so this trap goes away. **Use typed atomics.**

### `atomic.Pointer[T]` and the load-modify-publish pattern

```go
package config

import "sync/atomic"

type Config struct{ Host string }

var current atomic.Pointer[Config]

func Load() *Config         { return current.Load() }
func Replace(c *Config)     { current.Store(c) }
func Update(f func(*Config) *Config) {
	for {
		old := current.Load()
		new := f(old)
		if current.CompareAndSwap(old, new) {
			return
		}
	}
}
```

This is the canonical "copy-on-write" / "RCU lite" pattern in Go. Readers are zero-overhead (one atomic load); writers build a new value and CAS it in. Frequent writes have to retry but starvation is bounded by the write rate.

### `atomic.Value` (legacy)

```go
var v atomic.Value
v.Store(&Config{Host: "x"})
c := v.Load().(*Config)
```

Pre-generics. The first `Store` fixes the dynamic type; subsequent `Store`s of a different type panic. Since 1.19 prefer `atomic.Pointer[T]` for type safety.

### `And` and `Or` (since 1.23)

```go
var flags atomic.Uint32
old := flags.Or(1 << READY) // set the READY bit atomically; returns old value
if old & (1 << READY) == 0 {
	// we set it, do init
}
```

Pre-1.23 you'd write a CAS loop:

```go
for {
	old := flags.Load()
	if flags.CompareAndSwap(old, old | (1<<READY)) { break }
}
```

`Or`/`And` map to single instructions (`LOCK OR`, `LOCK AND` on x86; `LDADD` family on ARMv8.1+).

### Atomics don't serialize non-atomic memory

```go
var ready atomic.Bool
var data string

go func() {
	data = "hello"
	ready.Store(true)
}()
for !ready.Load() {}
fmt.Println(data) // RACE per the race detector? Actually NO — see below
```

Because Go atomics are sequentially consistent, the `ready.Store(true)` happens-after `data = "hello"`, and the receiver's `ready.Load() == true` happens-after both. So this **is** safe per the memory model. The race detector accepts it. **But** if you have a *non-atomic* write to `data` and another goroutine reads `data` without going through any atomic or channel, that's a race regardless.

The rule of thumb: atomics give you a happens-before edge from the store to a later load of the same atomic variable. Use that edge to synchronize a publish — e.g., set a pointer atomically after fully constructing the pointee.

### CAS loops and ABA

```go
for {
	old := node.next.Load()
	new := compute(old)
	if node.next.CompareAndSwap(old, new) { break }
}
```

CAS-based lock-free structures suffer from the "ABA problem": a value changes from A to B and back to A, and a CAS-based observer thinks nothing happened. In Go, the GC mostly saves you — the old `A` object can't be reclaimed while a goroutine still holds a reference — but if you're encoding state into integer values, beware.

### Atomics vs Mutex

A `Mutex` acquire is ~10 ns uncontended; an atomic op is ~1–5 ns. The win is real when:
- The critical section is a single integer update or pointer swap.
- Contention is high enough that mutex queueing dominates.

The lose is also real:
- An atomic on every iteration of a hot loop can be slower than a Mutex held once around the loop (cache-line ping-pong).
- Compound state (touching multiple fields) needs a Mutex; atomics protect one location at a time.

### `runtime/internal/atomic` and `unsafe.Pointer`

The runtime uses a lower-level atomic package internally with more arch-specific ops. User code should stick to `sync/atomic`. If you need to atomically swap an `unsafe.Pointer`, use `atomic.Pointer[T]` with `T` being whatever you're pointing at — the API hides the unsafe.

## Standard Library Hooks

- `sync/atomic` — the primitives.
- `sync.Map` and `sync.Pool` use atomics internally on the hot paths.
- `runtime/metrics` exposes atomic counters for runtime stats.
- `expvar` exports atomic-friendly counter types (`expvar.Int`, `expvar.Float`) — they wrap atomics.
- `runtime.Gosched()` if you find yourself spinning on a CAS loop — usually a sign you should use a mutex instead.

## Real-World Patterns

### 1. Lock-free counter

```go
type Stats struct {
	Requests atomic.Uint64
	Errors   atomic.Uint64
}

func (s *Stats) Hit(err bool) {
	s.Requests.Add(1)
	if err {
		s.Errors.Add(1)
	}
}
```

Drop-in for the classic "increment a counter from many goroutines". Zero contention except cache-line ping-pong.

### 2. Atomic publication of read-mostly config

```go
package featureflags

import "sync/atomic"

type Flags struct{ EnableX, EnableY bool }

var current atomic.Pointer[Flags]

func init() { current.Store(&Flags{}) }

func Get() *Flags          { return current.Load() }
func Set(f *Flags)         { current.Store(f) }
```

Readers do a single atomic load; updates publish a complete new `Flags`. Readers never see a half-updated state.

### 3. Once flag without `sync.Once`

```go
var initDone atomic.Bool

func ensureInit() {
	if initDone.Load() { return }
	doInit()
	initDone.Store(true)
}
```

Wrong! Two goroutines can both observe `false`, both call `doInit`. For "exactly once" you need `sync.Once`. Atomics give you "*at least* once with safe publication" if you arrange it carefully — usually just use `sync.Once`.

### 4. Atomic-bit flag set (since 1.23)

```go
const (
	StateReady = 1 << iota
	StateClosed
)

var state atomic.Uint32

func MarkReady() {
	old := state.Or(StateReady)
	if old & StateReady == 0 {
		log.Println("transitioned to Ready")
	}
}
```

`Or` returns the old value, so you can detect whether *you* were the one who set the bit.

### 5. Spinlock (rarely a good idea)

```go
type Spinlock struct{ v atomic.Int32 }

func (s *Spinlock) Lock() {
	for !s.v.CompareAndSwap(0, 1) {
		runtime.Gosched()
	}
}
func (s *Spinlock) Unlock() { s.v.Store(0) }
```

`Gosched` prevents starving the scheduler. Even so, `sync.Mutex` is usually faster because it has a tuned spin-then-park backoff. Use a spinlock only inside the runtime or for very short, very hot critical sections.

## Anti-Patterns & Gotchas

**Mixing atomic and non-atomic access to the same variable.** Race condition, race detector fires, behavior undefined. **All accesses must use atomic or none must.**

**Using a legacy free function on an unaligned field.** Panic on 32-bit ARM. Use typed atomics — they handle alignment automatically.

**Using atomics to publish a multi-field struct.** A `*T` swap publishes the pointer atomically, but individual field stores in the pointee are not synchronized unless the pointee was built before the swap. Build, then publish.

**Atomic for "exactly once" semantics.** Use `sync.Once` (or `OnceFunc`). Atomic is "at least once" or "at most once" — not "exactly once" without more work.

**`atomic.Pointer[T]` followed by reading fields without copying.** Once you have the loaded pointer, any reader can be working with stale fields. Treat the loaded `*T` as a snapshot; don't dereference fields concurrently with writes elsewhere unless those fields are themselves immutable.

**CAS loop on a hot variable from many cores.** Cache-line bouncing makes the loop spin orders of magnitude longer than expected. Shard the variable or use a mutex.

**`atomic.Value.Store` of a different concrete type.** Panic. Use `atomic.Pointer[T]` for type safety.

**Forgetting that `Add` returns the *new* value, not the old.** `n.Add(1)` returns the post-increment count. `Swap(0)` returns the old (pre-swap) value. Read the docs.

**Believing atomics replace mutexes everywhere.** They protect one location. Compound invariants (e.g., "balance = sum of transactions") need a mutex.

## Performance Notes

- `atomic.Int64.Add`: ~5 ns uncontended, ~50 ns under heavy contention.
- `atomic.Int64.Load`: ~1–2 ns; on amd64 it's a plain MOV.
- `atomic.Pointer[T].Load`: same as `Load` on `uintptr`.
- `CompareAndSwap`: ~5 ns uncontended; can loop under contention.
- Hot-counter contention scales like 1/cores after the cache line saturates. **Sharded counters** (`[NumCPU()]atomic.Int64`) often win at >4 cores: each core touches its own line.
- `Or`/`And` (1.23+) are one instruction on amd64/arm64 with the right ISA extensions; otherwise lowered to a CAS loop in the runtime.
- `atomic.Pointer[T]` has the same overhead as `atomic.Uintptr` plus the safety guarantees.

## How Big Companies Use It

- **`net/http`** uses `atomic.Int64` for active connection counts and `atomic.Pointer[Transport]` for swap-on-update.
- **CockroachDB** uses `atomic.Pointer[ClusterSettings]` for live config updates without locking the hot read path.
- **Prometheus** uses `atomic.Uint64` for every counter and gauge. Sharded counters are documented in their perf docs.
- **etcd**'s metrics use atomics on lease counters.
- **gRPC-Go** uses atomic.Pointer for resolver and balancer updates.
- **Tailscale**'s connection state uses `atomic.Bool` for lifecycle flags.
- **Cloudflare**'s Quiche-in-Go and tubular use atomics extensively in the hot path: https://blog.cloudflare.com/tubular-fixing-the-socket-api-with-ebpf/.

## Source Code References

Pinned to `go1.26`.

- The typed wrappers: [`src/sync/atomic/type.go`](https://github.com/golang/go/blob/master/src/sync/atomic/type.go).
- The legacy free functions: [`src/sync/atomic/doc.go`](https://github.com/golang/go/blob/master/src/sync/atomic/doc.go).
- Assembly implementations: `src/sync/atomic/asm_amd64.s`, `asm_arm64.s`, etc.
- Memory model wording on atomics: [`go.dev/ref/mem`](https://go.dev/ref/mem) — "The APIs in the `sync/atomic` package are collectively 'atomic operations' ... are sequentially consistent."
- `Or`/`And` proposal: https://go.dev/issue/61395 (1.23).
- `atomic.Pointer[T]` proposal: https://go.dev/issue/47141 (1.19).

## Further Reading

- Russ Cox, "Updating the Go Memory Model" (2022): https://research.swtch.com/gomm
- Go 1.19 memory model release note: https://go.dev/doc/go1.19#atomic_types
- Hans Boehm, "Threads cannot be implemented as a library" — the canonical paper on why memory models exist
- Damian Gryski, "Atomic counters: don't roll your own": https://github.com/dgryski/go-perfbook
- Filippo Valsorda on the lock-free settings pattern: https://words.filippo.io/
- Dmitry Vyukov, "Lock-free data structures in Go": various design docs and CL discussions

## Exercises / Self-Check

1. Why are Go atomics sequentially consistent rather than acquire/release?
2. Convert a `var mu sync.Mutex; var cfg *Config` lazy-init to `atomic.Pointer[Config]` + `sync.Once`. What changes for readers?
3. Implement a sharded `Counter` with one `atomic.Uint64` per P (use `runtime.GOMAXPROCS`). Benchmark vs a single counter.
4. Show a CAS-based push for a lock-free stack. Where does ABA come from in C; why is it usually safe in Go?
5. Why does `atomic.Value.Store(otherType)` panic? Why does `atomic.Pointer[T]` not need that check?
