# Goroutine Internals — the `g` Struct, Stacks, Preemption

## TL;DR

A goroutine is a `runtime.g` struct plus a **contiguous, growable stack** that starts at 8 KiB and can grow to 1 GiB. `g` holds the saved register state (`g.sched`), the stack bounds, a status word, and the M and P that own it. Spawning costs ~1 µs; switching costs ~150 ns; stacks grow by *copying* (the runtime rewrites every pointer into the old stack). Since Go 1.14 the runtime can **preempt asynchronously** via `SIGURG`, so a goroutine can be paused anywhere — including inside a tight CPU loop with no function calls. The single biggest gotcha: a leaked goroutine costs ≥8 KiB *plus* anything its stack frame retains; in long-running services these add up to gigabytes.

## Mental Model

```
   ┌────────────────────── goroutine ──────────────────────┐
   │                                                       │
   │   runtime.g {                                         │
   │       stack    [lo, hi]   ← bounds of this G's stack  │
   │       stackguard0         ← overflow check word       │
   │       _panic *_panic      ← active panics             │
   │       _defer *_defer      ← deferred function list    │
   │       m  *m               ← OS thread currently here  │
   │       sched gobuf {       ← saved registers           │
   │           sp, pc, bp,                                 │
   │           g  (for asm)                                │
   │       }                                               │
   │       atomicstatus uint32 ← _Gidle .. _Gdead          │
   │       goid     uint64                                 │
   │       waitreason waitReason                           │
   │       waiting  *sudog                                 │
   │   }                                                   │
   │                                                       │
   │   ┌─────────── stack ───────────┐                     │
   │   │ frame N                     │ hi → ┐              │
   │   │ frame N-1                   │      │              │
   │   │ ...                         │      ▼ grows down   │
   │   │ frame 0                     │                     │
   │   │ stackguard0 check word      │ lo                  │
   │   └─────────────────────────────┘                     │
   └───────────────────────────────────────────────────────┘
```

A goroutine never executes on its own — it's bound to an M (OS thread) which is bound to a P (logical processor). The G can be in one of several states; transitions are tracked atomically.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(3)
	for i := 0; i < 3; i++ {
		i := i // capture (Go 1.22+ no longer needed for range loops)
		go func() {
			defer wg.Done()
			fmt.Printf("goroutine %d, NumGoroutine=%d\n", i, runtime.NumGoroutine())
		}()
	}
	wg.Wait()
}
```

`go` is a single keyword: the compiler emits `runtime.newproc(fn, args)` which allocates a `g`, copies the arguments onto its new stack, sets it to `_Grunnable`, and enqueues it on the current P's run queue.

## Deep Dive

### The `g` struct, fields that matter

```go
// runtime/runtime2.go (excerpt; BSD-3 © The Go Authors)
type g struct {
    stack        stack    // [lo, hi)
    stackguard0  uintptr  // checked at every function prologue
    stackguard1  uintptr  // checked in cgo prologues

    _panic       *_panic
    _defer       *_defer

    m            *m       // current M (nil if not running)
    sched        gobuf    // saved sp/pc/bp/lr/ctxt/ret/g
    syscallsp    uintptr
    syscallpc    uintptr

    stktopsp     uintptr
    param        unsafe.Pointer

    atomicstatus atomic.Uint32  // _Gidle .. _Gdead
    stackLock    uint32
    goid         uint64
    schedlink    guintptr        // intrusive linked list
    waitsince    int64
    waitreason   waitReason

    preempt      bool
    preemptStop  bool
    preemptShrink bool

    // ... (many more fields for tracing, scan state, profiling)
}
```

`sched` is the register file. On a context switch, the M's `gogo` assembly:
1. Saves the running G's registers into its `sched`.
2. Loads the next G's `sched` into registers.
3. Jumps to `sched.pc`.

This is a coroutine switch in ~50–150 ns — no kernel transition. The M's *operating-system* stack is separate from the G's stack; it's switched at the same time.

### Goroutine states (`atomicstatus`)

```go
// runtime/runtime2.go (excerpt)
const (
    _Gidle      = iota // 0: uninitialized
    _Grunnable         // 1: on a run queue, not running
    _Grunning          // 2: executing
    _Gsyscall          // 3: in a syscall
    _Gwaiting          // 4: blocked (chan, lock, IO, GC, ...)
    _Gmoribund_unused
    _Gdead             // 6: unused / exited
    _Genqueue_unused
    _Gcopystack        // 8: stack being copied
    _Gpreempted        // 9: paused for async preempt
    _Gscan             // 0x1000: combined with another state, GC scanning stack
)
```

The `_Gscan` bit is OR'd with the actual state — `_Gscanrunning`, `_Gscanwaiting`, etc. It locks the G against state changes while GC walks its stack.

### Spawning — `newproc`

```go
// runtime/proc.go (excerpt; conceptual)
func newproc(fn *funcval) {
    gp := getg()        // current G
    pc := getcallerpc()
    systemstack(func() {
        newg := newproc1(fn, gp, pc)
        runqput(gp.m.p.ptr(), newg, true) // local run queue, "next" slot
    })
}
```

Key cost: `newproc1` either finds a dead G to reuse (from the per-P gFree list, populated when goroutines exit) or allocates fresh. Reusing is ~300 ns; fresh allocation is ~1 µs. Stacks are reused too — the freed G's stack is recycled.

### Stack growth — contiguous, copying

Each function prologue starts with:

```
TEXT main.foo(SB), 0, $48-0
    MOVQ TLS, AX            // load g pointer
    CMPQ SP, 16(AX)         // SP < g.stackguard0?
    JLS  morestack          // overflow → grow
    SUBQ $48, SP            // reserve frame
```

If `SP < stackguard0` (the stack is too small for this frame), the prologue jumps to `morestack`, which calls `newstack`:

```go
// runtime/stack.go (excerpt; conceptual)
func newstack() {
    gp := getg().m.curg
    oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize * 2
    if newsize > maxstacksize { // default 1 GiB
        throw("stack overflow")
    }
    copystack(gp, newsize)
}
```

`copystack`:
1. Allocates a new stack 2× the old size.
2. Adjusts each frame's pointers using the **stack maps** the compiler emitted.
3. Frees the old stack.

Two consequences:
- **All pointers to stack data are rewritten.** This is why you can't store a `uintptr` to a stack location across a function call — the location may have moved.
- **Stack grows are O(n)** in stack size. A function that recurses to 10 MiB triggers ~10 doublings, each more expensive than the last.

### Stack shrink

After GC, if the stack is using ≤25% of its capacity, the runtime *shrinks* it (also via `copystack`). Prevents long-lived goroutines from holding inflated stacks forever.

### Stack bounds and `stackguard0`

`stackguard0` is normally `lo + StackGuard` (where `StackGuard = 928`). The runtime can also set it to a magic sentinel `stackPreempt = 0xfffffade` to force the prologue check to fail — that's the **cooperative preemption** mechanism, used as a fallback when async preemption isn't applicable.

### Preemption mechanisms

**Cooperative (pre-1.14, still used).** At any function-call boundary, the prologue check trips, runs `morestack`, which sees `stackguard0 == stackPreempt`, and yields to the scheduler. Tight loops with no calls (e.g., `for i := 0; i < 1e9; i++ {}` not calling anything inlinable) never yield this way.

**Asynchronous (1.14+).** Sysmon sends `SIGURG` to the OS thread running a G that's been running >10 ms. The signal handler (`sigPreempt`) saves the user-mode register state, jumps into `asyncPreempt`, which redirects PC to `gopreempt_m`, which yields to the scheduler. Implemented in [`src/runtime/preempt.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/preempt.go) and per-arch `asm_*.s`.

**GC-driven preemption.** During GC, the runtime sets `preempt=true` and `stackguard0=stackPreempt` on every G. The next prologue trip yields and lets the GC scan that G's stack.

### Async preemption safety points

Not every PC is safe to preempt at — register-stale moments, unsafe pointer arithmetic, write-barrier-in-progress moments must be skipped. The compiler annotates per-PC "safe to preempt" bits in **PC-value tables** (stored in the function's metadata). When the signal hits an unsafe PC, the handler doesn't preempt; sysmon will retry shortly. See `src/runtime/asyncpreempt_*.s`.

### Goroutine IDs

`g.goid` is monotonically assigned. It's deliberately **not exposed** in the public API (no `runtime.GID()`) — the Go team's stance is that goroutine identity should not be used for control flow. People do extract it anyway (parse `runtime.Stack` output, or use `//go:linkname`), but it's discouraged.

The official reason: if `gid` were available, libraries would start writing goroutine-local storage, and any retrofit of work-stealing or stack copying that moved a goroutine across threads would break them.

### Where stacks live in memory

Goroutine stacks are allocated from a **stack cache** (`mcache.stackcache` for small sizes 2 KiB–32 KiB; `mheap.stackLarge` otherwise). They're not on the GC heap proper, but the GC does scan them. The `g0` (system goroutine) per M has the OS thread's stack instead.

### Sudog — the wait record

When a G blocks on a channel, mutex, condvar, or netpoll, the runtime allocates a `sudog` (or pulls from a per-P pool) and links it into the waiter list. The G itself transitions to `_Gwaiting`. Wakers walk the list, call `goready` on the sudog's G, return it to `_Grunnable`.

```go
// runtime/runtime2.go (excerpt)
type sudog struct {
    g           *g
    next, prev  *sudog
    elem        unsafe.Pointer // data for channels
    acquiretime int64
    releasetime int64
    ticket      uint32
    c           *hchan // channel
    // ... more
}
```

Sudogs are pooled per-P; a busy server with millions of channel operations sees zero allocation on the wait path.

### `runtime.Stack` and the dump format

```
goroutine 17 [select, 2 minutes]:
main.worker(0xc000010030)
    /tmp/x.go:18 +0x96
created by main.main in goroutine 1
    /tmp/x.go:10 +0x4a
```

The bracket holds the *waitreason* (e.g., `chan receive`, `select`, `sync.Mutex.Lock`) and optionally elapsed time (added by sysmon when blocked >1 minute). `created by ... in goroutine 1` (added in 1.22) shows the spawning goroutine — invaluable for tracing leaks.

### LockOSThread

`runtime.LockOSThread()` pins the calling G to its current M for the G's lifetime. The runtime cannot move that G to another M; the M will not run other Gs. When the G exits without unlocking, the M is destroyed (since 1.10) instead of returning to the pool — prevents leaks of OS-side state (signal masks, name, scheduler params).

### Implementation cost summary

- Spawn (with G reuse): ~300 ns.
- Spawn (with fresh G alloc): ~1 µs.
- Context switch: ~150 ns (savings vs OS thread switch which is ~1–10 µs).
- Stack growth (doubling 8 KiB → 16 KiB): ~5 µs.
- Stack growth (doubling 1 MiB → 2 MiB): ~ms.
- Sudog allocation (from pool): ~50 ns.
- Async preempt SIGURG round-trip: ~10 µs.

## Standard Library Hooks

- `runtime.NumGoroutine() int` — live count.
- `runtime.Stack(buf []byte, all bool) int` — dump stacks.
- `runtime.Gosched()` — voluntary yield (rare; async preempt usually suffices).
- `runtime.LockOSThread()`, `runtime.UnlockOSThread()` — pin to M.
- `runtime.Goexit()` — terminate the calling G after running defers; from any depth.
- `runtime.GoroutineProfile(p []runtime.StackRecord) (int, bool)` — sample stacks.
- `runtime/debug.SetMaxStack(n int) int` — change the 1 GiB cap.
- `runtime.SetMaxThreads(n int) int` — change the 10000 M cap.
- `runtime.SetFinalizer(obj, func)` — sketched in `12-runtime/07-finalizers-cleanups.md`.
- `runtime/pprof.Lookup("goroutine")` — full goroutine profile.

## Real-World Patterns

### 1. Goroutine leak detector

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func WatchLeaks(threshold int) {
	go func() {
		t := time.NewTicker(10 * time.Second)
		defer t.Stop()
		for range t.C {
			n := runtime.NumGoroutine()
			if n > threshold {
				buf := make([]byte, 1<<20)
				m := runtime.Stack(buf, true)
				fmt.Fprintf(stderr, "GOROUTINES=%d\n%s\n", n, buf[:m])
			}
		}
	}()
}

var stderr = mustStderr()

func mustStderr() interface {
	Write(p []byte) (int, error)
} {
	return nil // placeholder; in real code use os.Stderr
}
```

In production, plug to `os.Stderr` and Sentry/Datadog. Pair with `go.uber.org/goleak` in tests.

### 2. Bounded worker pool

```go
package main

import "sync"

func RunBounded[T any](items []T, parallelism int, fn func(T)) {
	sem := make(chan struct{}, parallelism)
	var wg sync.WaitGroup
	for _, it := range items {
		sem <- struct{}{}
		wg.Add(1)
		go func(it T) {
			defer wg.Done()
			defer func() { <-sem }()
			fn(it)
		}(it)
	}
	wg.Wait()
}
```

Caps live G count regardless of input size. Replaces unbounded `for _, it := range items { go fn(it) }`, which can spawn millions and saturate the run queue.

### 3. Pin to OS thread for FFI

```go
package main

import "runtime"

func init() {
	// Lock so OpenGL/JNI/CUDA calls run on the same OS thread.
	runtime.LockOSThread()
}

func main() {
	// All work in this goroutine inherits the lock.
	render()
}

func render() {}
```

### 4. Bounded stack for deep recursion

```go
package main

import "runtime/debug"

func init() {
	debug.SetMaxStack(64 << 20) // cap at 64 MiB instead of 1 GiB
}

func main() { /* avoids runaway recursion exhausting memory */ }
```

Useful for parsers / interpreters that handle untrusted input.

### 5. Spawn-time arg copy gotcha

```go
// Bug pre-1.22:
for i := 0; i < 5; i++ {
    go func() { print(i) }() // captured `i` is shared; usually prints 5 five times
}

// Fix pre-1.22, idiomatic:
for i := 0; i < 5; i++ {
    go func(i int) { print(i) }(i) // copy
}

// Go 1.22+: loop variable is per-iteration; original code is correct.
```

`go func(i int) { ... }(i)` works on all versions; relying on 1.22+ semantics requires `//go:build go1.22` or a `go.mod` minimum.

## Anti-Patterns & Gotchas

**Goroutine leaks via dangling receivers.** `go func() { <-ch }()` where nobody sends. The G parks in `_Gwaiting` forever. Detect with `goleak`.

**Spawning a goroutine per RPC without bounds.** Common in incoming-request handlers without semaphores. Under load you get millions of Gs, run queue saturation, and OOM from stack memory.

**Storing pointers to local stack variables across function calls.** If the stack grows (copies), the address changes. Use a pointer to a heap allocation instead.

**Using `runtime.NumGoroutine()` as a "lock contention" gauge.** It's just a count; says nothing about runnable vs waiting. Use `runtime/metrics` series `/sched/goroutines:goroutines`.

**`runtime.LockOSThread` without unlock.** Locks-without-unlock terminate the M when the G exits (1.10+). For pooled threads this is fine; for app-level threads (e.g., main) it's mostly harmless.

**Counting on a specific `goid`.** It's unstable across runs. The advice from the Go team has been consistent for 10+ years: pass context, not IDs.

**Recursing past `SetMaxStack`.** Causes runtime panic `runtime: goroutine stack exceeds N-byte limit`. Convert deep recursion to iteration.

**Using `runtime.Gosched()` "to be fair".** Almost never needed since 1.14. Adds scheduler overhead.

**Trying to interrupt a goroutine.** No API. Pass a `context.Context`, check `ctx.Err()` at safe points.

**Goroutines as cheap actors.** They're cheap, not free. 1M goroutines = ~8 GiB of stacks at minimum. For massively concurrent state-machine workloads, look at single-goroutine + state objects.

## Performance Notes

- 1M live goroutines: ~8 GiB stack memory (idle 8 KiB each) + ~64 KiB of `g`/sudog metadata each. Plus whatever each frame holds live.
- Spawn rate: ~1M Gs/sec sustainable on a single P; ~10M/sec across `GOMAXPROCS=8` (with G reuse warm).
- Context switch: ~150 ns user-mode; ~2 µs if it crosses M handoff (P swap).
- `runtime.Stack(buf, all=true)` cost: O(num goroutines × stack depth). At 1M Gs this is several hundred ms — STW. Use sparingly.
- Stack first growth (8 KiB → 16 KiB): ~5 µs.
- Stack shrink: same cost as a growth.
- Sudog from pool: ~50 ns; from heap: ~300 ns.

## How Big Companies Use It

- **Caddy** (web server) treats each connection as a goroutine; documents G memory analyses for high-connection-count deployments: https://caddyserver.com.
- **gRPC-Go** uses a "per-stream goroutine" model; the gRPC team has multiple posts on G accounting: https://grpc.io/blog/.
- **CockroachDB** caps per-request G fan-out via `errgroup.WithLimit` (1.21+) and `singleflight`; documented at https://www.cockroachlabs.com/docs/.
- **Tailscale**'s `magicsock` uses `LockOSThread` for tunnel I/O glue: https://github.com/tailscale/tailscale.
- **etcd** uses goleak in their CI for every PR; the pattern is documented in their CONTRIBUTING.
- **Uber** publishes the `go.uber.org/goleak` library: https://github.com/uber-go/goleak.
- **Discord** post-mortems describe how a state-cache goroutine's stack grew to 8 MiB and amplified GC mark times: https://discord.com/blog/.

## Source Code References

Pinned to `go1.26`.

- `g`, `m`, `p`, `sudog` definitions: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/runtime2.go).
- Goroutine creation: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go) — `newproc`, `newproc1`.
- Goroutine exit: same file, `goexit0`.
- Stack growth: [`src/runtime/stack.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/stack.go) — `newstack`, `copystack`, `shrinkstack`.
- Async preemption: [`src/runtime/preempt.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/preempt.go) and per-arch `asm_*.s` (e.g., `asyncpreempt_amd64.s`).
- Signal handling for preempt: [`src/runtime/signal_unix.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/signal_unix.go) — `sigPreempt`.
- LockOSThread: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go) — search `dolockOSThread`.
- Stack copy / pointer rewrite: [`src/runtime/stack.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/stack.go) — `adjustpointers`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Russ Cox, "Goroutines, threads, and stacks": https://research.swtch.com/gostack.
- Dmitry Vyukov, "Scalable Go Scheduler Design Doc": https://golang.org/s/go11sched.
- Austin Clements, "Non-cooperative goroutine preemption" (proposal #24543): https://go.dev/issue/24543.
- Keith Randall, "Contiguous stacks" (the 1.4 redesign): https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/.
- Dave Cheney, "Five things that make Go fast" (goroutine cost section): https://dave.cheney.net/2014/06/07/five-things-that-make-go-fast.
- Kavya Joshi, "The Scheduler Saga" (GopherCon 2018): https://www.youtube.com/watch?v=YHRO5WQGh0k.
- "Anatomy of goroutines" — Hyperledger Fabric blog (deep code-walk): https://hyperledger-fabric.readthedocs.io.
- `goleak`: https://github.com/uber-go/goleak.

## Exercises / Self-Check

1. Why does Go copy stacks instead of using "segmented" stacks (the pre-1.4 design)? What problem did segments cause?
2. A goroutine recurses 200 times, each frame using 4 KiB. How many stack growths happen? At what sizes?
3. Sketch the path from `runtime.SIGURG` arriving on a thread to the running goroutine actually yielding. What can prevent the yield?
4. Write a program with `go func() { <-make(chan int) }()` and `goleak.VerifyNone(t)`. What does goleak detect, and how does it know?
5. Why is `g.goid` private API? Name two refactoring scenarios in the runtime where exposing it would have made the change impossible.
