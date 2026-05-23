# Finalizers and Cleanups — SetFinalizer, AddCleanup

## TL;DR

A **finalizer** is a function the GC schedules when an object becomes unreachable, useful for releasing non-Go resources (FDs, C pointers, memory-mapped regions). `runtime.SetFinalizer(obj, fn)` attaches one; `runtime.AddCleanup(obj, fn, arg)` (since 1.24) is the modern replacement that avoids resurrection bugs and supports multiple cleanups per object. Both **defer object freeing by one GC cycle** (the finalizer's argument must survive that cycle). The single biggest gotcha: **finalizers are not destructors**. They run on a single goroutine, eventually (not promptly), in arbitrary order; if the program exits before GC, they don't run at all. Use `defer` for deterministic cleanup; reserve finalizers for safety nets.

## Mental Model

```
   GC cycle N:                  GC cycle N+1:
   ┌────────────────────┐       ┌────────────────────┐
   │ obj becomes        │       │ finalizer runs     │
   │ unreachable        │       │ (queue → goroutine)│
   │                    │       │                    │
   │ but: has finalizer │       │ obj's memory       │
   │ → kept alive!      │       │ finally freed      │
   │ → finalizer queued │       │   (next cycle      │
   │                    │       │    or sweep)       │
   └────────────────────┘       └────────────────────┘

   AddCleanup is similar but:
     - cleanup function does NOT see the object
     - object can be freed in cycle N+1, immediately
     - no resurrection possible
```

A finalizer fires when the runtime detects the object is unreachable AND has a finalizer. The finalizer's existence makes the object reachable (it's a root), so the object survives the cycle that found it; the finalizer runs concurrently after; the object is collected in a *later* cycle.

A cleanup is similar but the runtime is forbidden to give the cleanup access to the object (it only sees the `arg` you provide), so resurrection can't happen.

## Syntax & Basic Usage

### `SetFinalizer` (since Go 1.0)

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

type Resource struct {
	fd int
}

func newResource(fd int) *Resource {
	r := &Resource{fd: fd}
	runtime.SetFinalizer(r, func(r *Resource) {
		fmt.Println("finalizer: closing fd", r.fd)
		// syscall.Close(r.fd) — real cleanup
	})
	return r
}

func main() {
	_ = newResource(42)
	runtime.GC()
	time.Sleep(100 * time.Millisecond) // give finalizer goroutine a chance
	runtime.GC()                       // second cycle frees the memory

	// Output:
	// finalizer: closing fd 42
}
```

### `AddCleanup` (since Go 1.24)

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

type Buffer struct {
	data []byte
}

func newBuffer(n int) *Buffer {
	b := &Buffer{data: make([]byte, n)}
	// arg is captured by value — the cleanup never sees `b`
	runtime.AddCleanup(b, func(fd int) {
		fmt.Println("cleanup: would release fd", fd)
	}, 99)
	return b
}

func main() {
	_ = newBuffer(1024)
	runtime.GC()
	time.Sleep(100 * time.Millisecond)
	// Output:
	// cleanup: would release fd 99
}
```

The cleanup function takes only `arg` — it has no reference back to the object. The runtime can free `b` *in the same cycle* the cleanup is scheduled.

## Deep Dive

### Why finalizers defer collection by a cycle

Suppose `obj` has finalizer `f(*Obj)`. When GC finds `obj` unreachable:

1. `f` is queued; `f` holds a reference to `obj` (its argument).
2. So `obj` is now reachable (via the queue).
3. The cycle ends with `obj` still alive.
4. The finalizer goroutine runs `f(obj)` — at this point `obj` may legally still be used.
5. After `f` returns, the finalizer is cleared; `obj` is again a candidate for collection.
6. The next GC cycle frees `obj`'s memory.

So: an object with a finalizer survives at least 2 GC cycles after its last user reference. Memory usage for finalizer-laden objects is roughly doubled relative to un-finalizer-laden equivalents.

### Resurrection

A finalizer can store its argument back into a global, making the object reachable again. The GC doesn't refire the finalizer (each obj has at most one chance unless you re-`SetFinalizer`). This pattern is occasionally useful but more often a source of confusion. `AddCleanup` forbids it by construction.

### Why `AddCleanup` exists

Problems with `SetFinalizer`:
1. **Resurrection**: easy to write by accident (capturing `obj` in the finalizer closure that ends up in a global).
2. **Single finalizer per object**: `SetFinalizer(obj, f2)` replaces `f1`. Hard to attach cleanups from multiple layers.
3. **Order**: when `A` and `B` reference each other, neither becomes unreachable; cycles never finalize. With `AddCleanup`, the system holds a separate reference to the arg, so reachability of args is independent.
4. **API ergonomics**: the finalizer signature `func(*T)` forces a pointer-to-the-finalized-type.

`AddCleanup`:
- Allows multiple cleanups on one object (additive).
- `arg` can be any type; commonly an FD or a slice.
- Returns a `Cleanup` value; `cleanup.Stop()` cancels it.
- The cleanup function cannot reach back to the object.

### Cleanup Stop and pinning

```go
// runtime/mfinal.go (since 1.24)
type Cleanup struct { ... }
func (c Cleanup) Stop()
```

```go
package main

import "runtime"

func main() {
	b := make([]byte, 1024)
	c := runtime.AddCleanup(&b, func(_ int) { /* ... */ }, 0)
	// Decide later not to clean:
	c.Stop()
	_ = b
}
```

The cleanup is *removed* from the queue; if `b` is unreachable later, no callback runs.

### What finalizers/cleanups are good for

- Closing OS resources (FDs, mmaps) as a safety net when explicit `Close()` was forgotten.
- Releasing C-side memory tied to a Go handle.
- Decrementing a refcount in a shared store.
- Logging diagnostic info when a goroutine-bound resource leaks.

### What they're bad for

- Releasing **scarce** resources (DB connections, large mmaps) — finalizer delay is unbounded. Use `defer Close()`.
- Anything that must run before exit — finalizers do not run on `os.Exit` or panic-induced termination.
- Anything dependent on order — finalizers run on a single goroutine in arbitrary order.
- Anything time-sensitive — finalizers run "eventually". With `GOGC=off` or low pressure, "eventually" can be never.

### Constraints

- `SetFinalizer(obj, fn)`: `obj` must be a pointer to a heap-allocated object. The runtime panics if `obj` is a stack-allocated pointer (the analyzer normally prevents this, but escape analysis can miss things — `obj` is moved to heap when you call `SetFinalizer`).
- `fn`'s signature: `func(T)` where `T == typeof(obj)`. Variadic args are not allowed.
- `obj` must not be the result of `&literal{}` where the literal's lifetime is unknown — use `new(T)` or `&T{}` with a clear escape path.
- `AddCleanup`: `obj` must be a pointer; `arg` can be any type but is captured **by value**.

### Re-setting

`SetFinalizer(obj, nil)` removes the finalizer. `SetFinalizer(obj, fn)` after an earlier set replaces it. Each `obj` has one slot.

`AddCleanup` is additive; each call adds a separate cleanup.

### Finalizer goroutine

A single goroutine (`runfinq`) drains the queue and calls finalizers. If one finalizer blocks (e.g., waits on a channel), all others stall behind it. Keep finalizers **short and non-blocking**. Send to a buffered channel for async work:

```go
package main

import "runtime"

type R struct{}

var work = make(chan func(), 1024)

func init() {
	go func() { for f := range work { f() } }()
}

func newR() *R {
	r := &R{}
	runtime.SetFinalizer(r, func(r *R) {
		select {
		case work <- func() { /* slow cleanup */ }:
		default: /* drop */
		}
	})
	return r
}
```

### Order on exit

When the program exits, finalizers may or may not run. The runtime makes a best-effort pass at the very end (`runfinq` continues until the queue drains), but signal-induced or `os.Exit` termination skips it entirely.

If you need final cleanup, use `defer` in `main` or `signal.NotifyContext`.

### Interaction with weak pointers

`weak.Make[T]` (since 1.24) gives you a `weak.Pointer[T]` that doesn't keep its referent alive. Often used together with `AddCleanup`: a cache holds weak refs, a cleanup runs when the strong reference (e.g., the original `*T`) is gone.

```go
package main

import (
	"runtime"
	"sync"
	"weak"
)

type Cache[K comparable, V any] struct {
	mu sync.Mutex
	m  map[K]weak.Pointer[V]
}

func (c *Cache[K, V]) Put(k K, v *V) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.m == nil {
		c.m = map[K]weak.Pointer[V]{}
	}
	c.m[k] = weak.Make(v)
	// When v is unreachable elsewhere, weak ref goes nil and we GC the map entry.
	runtime.AddCleanup(v, func(k K) {
		c.mu.Lock()
		delete(c.m, k)
		c.mu.Unlock()
	}, k)
}

func (c *Cache[K, V]) Get(k K) *V {
	c.mu.Lock()
	defer c.mu.Unlock()
	if p, ok := c.m[k]; ok {
		return p.Value()
	}
	return nil
}
```

This pattern is detailed in `08-stdlib/34-weak.md`.

## Standard Library Hooks

- `runtime.SetFinalizer(obj any, finalizer any)`.
- `runtime.AddCleanup(ptr *T, cleanup func(arg ArgT), arg ArgT) Cleanup` (since 1.24).
- `runtime.Cleanup.Stop()` (since 1.24).
- `runtime.KeepAlive(x any)` — prevent premature finalization.
- `weak.Make(*T) weak.Pointer[T]` (since 1.24).
- `runtime.MemStats.NumFinalizer` / `runtime/metrics` `/gc/finalizers:objects` — count of objects with finalizers.
- `runtime.GC()` — manual cycle, forces finalizer queue advancement.

## Real-World Patterns

### 1. Safety-net for forgotten `Close`

```go
package main

import (
	"fmt"
	"runtime"
)

type File struct {
	fd     int
	closed bool
}

func openFile(fd int) *File {
	f := &File{fd: fd}
	runtime.AddCleanup(f, func(fd int) {
		fmt.Fprintf(stderr, "leak: file %d not Closed\n", fd)
		// syscall.Close(fd)
	}, fd)
	return f
}

func (f *File) Close() error {
	if f.closed {
		return nil
	}
	f.closed = true
	// syscall.Close(f.fd)
	return nil
}

var stderr writer

type writer interface{ Write(p []byte) (int, error) }
```

The cleanup *logs* a leak. It's a debugging aid, not a substitute for `defer f.Close()`. The cleanup is removed by `Close()` calling `runtime.Cleanup.Stop()` (real impl would store the `Cleanup` in the `File`).

### 2. Releasing C-allocated memory

```go
package main

/*
#include <stdlib.h>
*/
import "C"
import (
	"runtime"
	"unsafe"
)

type CBuffer struct {
	ptr unsafe.Pointer
	len int
}

func NewCBuffer(n int) *CBuffer {
	p := C.malloc(C.size_t(n))
	b := &CBuffer{ptr: p, len: n}
	runtime.AddCleanup(b, func(p unsafe.Pointer) {
		C.free(p)
	}, p)
	return b
}
```

When the Go-side `*CBuffer` is unreachable and the cleanup fires, `C.free` runs. Without this, leaked Go handles would leak C memory indefinitely.

### 3. Multiple cleanups per object (only with AddCleanup)

```go
package main

import "runtime"

type Conn struct{}

func newConn() *Conn {
	c := &Conn{}
	runtime.AddCleanup(c, decrementMetric, "conn.opened")
	runtime.AddCleanup(c, logLeak, "conn")
	return c
}

func decrementMetric(name string) { /* ... */ }
func logLeak(kind string)         { /* ... */ }
```

`SetFinalizer` would let you set only one; with `AddCleanup`, every layer can attach its own.

### 4. Pin to prevent premature finalization

```go
package main

import (
	"runtime"
	"unsafe"
)

func writeWithRawFd(fd int) {
	// raw syscall; we need fd to outlive the call
	ptr := unsafe.Pointer(&fd)
	doSyscall(ptr)
	runtime.KeepAlive(fd) // pin until here
}

func doSyscall(unsafe.Pointer) {}
```

If `fd` had a finalizer attached and the analyzer didn't see the syscall reading it, the finalizer could run between the syscall starting and finishing. `KeepAlive` blocks that.

### 5. Removing cleanup on explicit close

```go
package main

import "runtime"

type Buf struct {
	data    []byte
	cleanup runtime.Cleanup
}

func NewBuf(n int) *Buf {
	b := &Buf{data: make([]byte, n)}
	b.cleanup = runtime.AddCleanup(b, func(int) {
		// log leak
	}, 0)
	return b
}

func (b *Buf) Free() {
	b.cleanup.Stop()
	b.data = nil
}
```

`Free` removes the leak-warning cleanup, then drops the data.

## Anti-Patterns & Gotchas

**Relying on finalizers for resource release.** Connections, locks, FDs need `defer Close()`. Finalizers run *eventually*, not soon.

**Long-running finalizers.** Block the finalizer goroutine; subsequent cleanups stall.

**Finalizers that synchronously call external systems.** Same problem; bonus risk of deadlock if the external system depends on Go code that's waiting on GC.

**Finalizers that resurrect their object.** Legal but confusing — the second time the object becomes unreachable, no finalizer runs.

**Finalizer on a struct value.** `SetFinalizer(&v)` where `v` is a `struct` literal that's about to escape: the analyzer must see it escape. Use `new(T)` or `&T{}` returned from a function that's expected to allocate.

**Cyclic references between finalizable objects.** `A.b = b; b.a = a`; both have finalizers. Neither is unreachable from the other; finalizers never run. Use `AddCleanup` and explicit `nil`-out, or `weak.Pointer`.

**Assuming finalizers run before `os.Exit`.** They don't. Run cleanup in `defer` blocks reachable from `main`.

**Forgetting `runtime.KeepAlive` for unsafe interop.** GC may finalize an object whose pointer is held only as `uintptr`. Worst case: use-after-free in C.

**Setting many finalizers in a tight loop.** Each finalizer registration costs ~µs; for 10M objects this is 10s. Use `sync.Pool` or arena patterns.

**Using `SetFinalizer(nil, nil)` thinking it clears all.** Per-object only; pass the specific object.

## Performance Notes

- `SetFinalizer` cost: ~1 µs (allocates a `specialfinalizer`, hooks into the span).
- `AddCleanup` cost: ~1 µs.
- `Cleanup.Stop`: O(1), atomic CAS.
- Finalizer trigger latency: 2 GC cycles minimum. With `GOGC=100` and steady load, ~ms to ~s.
- Finalizer goroutine throughput: ~10–100k finalizers/sec (depending on per-finalizer work).
- Memory overhead per finalizable object: ~32 bytes (the `special` record in the span).
- Programs with millions of finalizers see noticeable mark-time increase (the queue is a root).

`runtime/metrics`:
- `/gc/finalizers/queue:objects` — currently queued.
- `/gc/cleanups/queue:objects` — same for cleanups (1.24+).

## How Big Companies Use It

- **Tailscale** uses `AddCleanup` for `netaddr.IPSet` cache invalidation: https://github.com/tailscale/tailscale.
- **CockroachDB** uses finalizers as leak detectors on internal buffer pools — releases to prod if a buffer makes it to GC without being recycled: https://www.cockroachlabs.com/blog/.
- **Dgraph / Badger** uses cleanups (since 1.24) to release mmap regions when handles go out of scope: https://github.com/dgraph-io/badger.
- **gRPC-Go** historically used finalizers on `*grpc.ClientConn` to log leaks; the modern version logs with `slog` and uses `AddCleanup`: https://github.com/grpc/grpc-go.
- **Kubernetes' apimachinery** uses finalizers (the *Kubernetes* "finalizer" string mechanism, not Go runtime finalizers) — naming overlap is confusing; the Go-runtime kind is rarely used in K8s code.
- **etcd** uses finalizers on lease objects in pre-1.24 code; modern code is migrating to `AddCleanup`: https://github.com/etcd-io/etcd.
- **The Go runtime itself** uses finalizers internally for `os.File` (closing the FD if the user forgot to Close), `os/exec` (reaping zombie processes), and `net.Conn` (closing the underlying FD).

## Source Code References

Pinned to `go1.26`.

- Finalizer machinery: [`src/runtime/mfinal.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mfinal.go).
- `runfinq` (finalizer goroutine): same file, function `runfinq`.
- `SetFinalizer`: same file, function `SetFinalizer`.
- `AddCleanup`: same file (1.24+), function `AddCleanup`.
- Cleanup `Stop`: same file, method `Stop`.
- GC interaction (special records): [`src/runtime/mheap.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mheap.go) — search `special` and `specialfinalizer`.
- `runtime.KeepAlive`: same file as finalizer; minimal compile-time intrinsic.
- Weak pointers: [`src/weak/weak.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/weak/weak.go) (1.24+).
- `os.File` finalizer example: [`src/os/file_unix.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/os/file_unix.go) — search `SetFinalizer`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "AddCleanup" proposal #67535: https://github.com/golang/go/issues/67535.
- "Weak pointers" proposal #67552: https://github.com/golang/go/issues/67552.
- Go 1.24 release notes (cleanup + weak): https://go.dev/doc/go1.24.
- Russ Cox, "Go's Concurrent GC and finalizers" (research.swtch): https://research.swtch.com/.
- Go FAQ "Why does my program use so much memory? Finalizers": https://go.dev/doc/faq#Why_does_my_program_use_so_much_memory.
- Damian Gryski, "Comparison: SetFinalizer vs AddCleanup vs explicit close": https://github.com/dgryski/go-perfbook.
- "Releasing scarce resources" — Go wiki section on finalizers vs defer: https://go.dev/wiki.

## Exercises / Self-Check

1. Sketch the GC cycle interaction: an object `o` with finalizer becomes unreachable in cycle N. Mark each event (queue, run, free) on a timeline.
2. Take a small program with a finalizer that resurrects its object (stores it in a global). Predict the behavior, then run with `GODEBUG=gctrace=1`. What survives, what's freed?
3. Why does `runtime.AddCleanup`'s cleanup function take `arg` by value rather than a pointer to the finalized object?
4. Implement a leak-detector `Conn` wrapper that uses `AddCleanup` to log when `Close` was never called. Use `Cleanup.Stop` on explicit close.
5. Write a tiny benchmark comparing `SetFinalizer` and `AddCleanup` overhead per allocation. Predict which is faster and why.
