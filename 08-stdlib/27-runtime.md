# `runtime` — Talking to the Go Runtime

## TL;DR

`runtime` exposes scheduler, GC, and memory introspection: `GOMAXPROCS`, `Gosched`, `NumGoroutine`, `ReadMemStats`, `SetFinalizer`, `Caller`, `Stack`. Most user code touches it rarely — the runtime tunes itself well. Reach for it when profiling, when implementing finalizers (and now `AddCleanup` since 1.24), when implementing low-level utilities (panic recovery, traceback generators), or when you genuinely need to bypass the runtime's defaults.

## Mental Model

```
runtime
   ├─ Scheduler:   GOMAXPROCS, Gosched, LockOSThread
   ├─ Goroutines:  NumGoroutine, Goexit
   ├─ Memory/GC:   ReadMemStats, GC, KeepAlive, SetFinalizer, AddCleanup (1.24+)
   ├─ Stack:       Caller, Callers, CallersFrames, Stack
   ├─ Build:       Version, GOOS, GOARCH, GOROOT
   └─ Internals:   Breakpoint, MemProfileRate, BlockProfileRate
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("Go version:", runtime.Version())
	fmt.Println("GOOS/ARCH:", runtime.GOOS, runtime.GOARCH)
	fmt.Println("CPUs:", runtime.NumCPU())
	fmt.Println("Goroutines:", runtime.NumGoroutine())
	// Output (varies):
	// Go version: go1.26
	// GOOS/ARCH: linux amd64
	// CPUs: 12
	// Goroutines: 1
}
```

## Deep Dive

### `GOMAXPROCS`

```go
runtime.GOMAXPROCS(0)    // returns current
runtime.GOMAXPROCS(4)    // set to 4; returns old
```

Default = `NumCPU()`. In containers without `cpu.cfs_quota` awareness, this used to over-report. Since 1.16-ish and especially with `go.uber.org/automaxprocs`, container-aware GOMAXPROCS is common.

Since 1.25, the runtime can read `cgroup` CPU limits on Linux and adjust automatically (opt-in via `GODEBUG`).

### `Gosched`

```go
runtime.Gosched()
```

Yields the processor, allowing other goroutines to run. The scheduler is preemptive since 1.14, so this is almost never needed — except in pathological tight loops without function calls.

### `LockOSThread` / `UnlockOSThread`

```go
runtime.LockOSThread()
defer runtime.UnlockOSThread()
```

Pins the goroutine to one OS thread. Use case: thread-local state required by C code (OpenGL, some GUI toolkits), or signal handling tied to a thread.

### Memory stats

```go
var m runtime.MemStats
runtime.ReadMemStats(&m)
fmt.Println("alloc:", m.Alloc, "sys:", m.Sys, "gc cycles:", m.NumGC)
```

`MemStats` is large; reading it stops the world briefly. Don't poll every second; use `runtime/metrics` (modern) instead.

### `GC` and `KeepAlive`

```go
runtime.GC()             // force a GC
runtime.KeepAlive(x)     // ensure x stays reachable past this point
```

`KeepAlive` matters when `x` is referenced only by `unsafe.Pointer` or by C code via cgo — the GC can otherwise reclaim `x` while C still uses it.

### Finalizers and cleanup (1.24+)

```go
runtime.SetFinalizer(obj, func(o *T) { o.Close() })
runtime.AddCleanup(obj, func(state cleanupState) { cleanup(state) }, state)
```

**Finalizers are unreliable** — they run at unpredictable times, may not run before exit, can resurrect objects, and prevent prompt collection. Use them only as a last resort.

`runtime.AddCleanup` (1.24+) fixes most finalizer issues:

- Cleanup doesn't resurrect the object.
- Multiple cleanups per object allowed.
- Function takes a separate state argument (no risk of capturing the object and pinning it).

```go
import "runtime"

type File struct{ fd int }

func Open(path string) *File {
	f := &File{fd: open(path)}
	runtime.AddCleanup(f, func(fd int) { close(fd) }, f.fd)
	return f
}
```

### Stack traces

```go
buf := make([]byte, 8192)
n := runtime.Stack(buf, false) // current goroutine only
fmt.Println(string(buf[:n]))

n = runtime.Stack(buf, true)   // all goroutines
```

Used by `debug.Stack`, panic recover, profilers.

### `Caller` and `Callers`

```go
pc, file, line, ok := runtime.Caller(1) // 1 frame up
fmt.Println(file, line)

pcs := make([]uintptr, 32)
n := runtime.Callers(0, pcs)
frames := runtime.CallersFrames(pcs[:n])
for {
	f, more := frames.Next()
	fmt.Println(f.File, f.Line, f.Function)
	if !more { break }
}
```

Used by log libraries to inject source position.

### `runtime/metrics` (since 1.16, preferred over MemStats)

```go
import "runtime/metrics"

samples := []metrics.Sample{
	{Name: "/memory/classes/heap/free:bytes"},
	{Name: "/sched/goroutines:goroutines"},
}
metrics.Read(samples)
for _, s := range samples {
	fmt.Println(s.Name, s.Value)
}
```

Low-overhead, structured. Discover with `metrics.All()`.

### `Goexit`

```go
runtime.Goexit() // terminate current goroutine, run defers, ignored by recover
```

Used by `testing.T.FailNow()`.

## Standard Library Hooks

- `runtime/debug` — `Stack`, `PrintStack`, `BuildInfo`, GC tuning.
- `runtime/pprof` — profiling.
- `runtime/metrics` — modern metrics.
- `runtime/trace` — execution tracer.

## Real-World Patterns

### 1. Container-aware GOMAXPROCS

```go
import _ "go.uber.org/automaxprocs"
```

Auto-detect cgroup CPU quota. Since Go 1.25, the runtime does this natively with a `GODEBUG` flag.

### 2. Logger that injects source line

```go
func logCaller(format string, args ...any) {
	_, file, line, _ := runtime.Caller(1)
	fmt.Printf("%s:%d: "+format+"\n", append([]any{file, line}, args...)...)
}
```

(`slog` with `AddSource: true` does this for you now.)

### 3. Stack snapshot on panic

```go
defer func() {
	if r := recover(); r != nil {
		buf := make([]byte, 4096)
		n := runtime.Stack(buf, false)
		log.Printf("panic %v\n%s", r, buf[:n])
	}
}()
```

### 4. Cleanup for unsafe-pointer-allocated buffer

```go
type Buf struct { ptr unsafe.Pointer; len int }

func New(n int) *Buf {
	b := &Buf{ptr: C.malloc(C.size_t(n)), len: n}
	runtime.AddCleanup(b, func(p unsafe.Pointer) { C.free(p) }, b.ptr)
	return b
}
```

Use case: wrap a C allocation in a Go object with deterministic-ish cleanup.

### 5. Periodic memory metrics

```go
import "runtime/metrics"

func dumpMetrics() {
	samples := []metrics.Sample{
		{Name: "/memory/classes/heap/objects:bytes"},
		{Name: "/gc/heap/objects:objects"},
		{Name: "/sched/goroutines:goroutines"},
	}
	for range time.Tick(30 * time.Second) {
		metrics.Read(samples)
		for _, s := range samples { fmt.Println(s.Name, s.Value) }
	}
}
```

## Anti-Patterns & Gotchas

**`runtime.GC()` in production code.** Wastes CPU; trust the GC.

**`runtime.Gosched` to "fix" performance.** Almost never the right answer.

**`SetFinalizer` as the primary cleanup mechanism.** Use `defer` or `AddCleanup`.

**Forgetting `runtime.KeepAlive` after cgo or unsafe pointer usage.** Premature GC.

**Reading `MemStats` in a hot loop.** STW pauses.

**Capturing the finalized object inside its finalizer closure.** Resurrects it.

**`LockOSThread` without matching `UnlockOSThread`.** Goroutine never returns to the pool of M's.

**Comparing `runtime.Version()` strings naively.** "go1.10" vs "go1.9" — string compare gives wrong order.

## Performance Notes

- `runtime.NumGoroutine()` is O(1).
- `runtime.NumCPU()` is cached at startup.
- `MemStats.Read` is a STW operation; microseconds to milliseconds.
- `runtime/metrics` samples are very cheap.
- Finalizers add cost: each finalized object is scheduled and processed by a dedicated goroutine.
- `KeepAlive` compiles to a no-op or single instruction.

## How Big Companies Use It

- **Uber** wrote `automaxprocs` to fix the CPU-quota mismatch in Kubernetes.
- **Cockroach** uses `runtime/metrics` heavily for self-monitoring.
- **Kubernetes** uses `runtime.Stack` in its panic-recovery middleware.
- **Tailscale** uses `LockOSThread` for some networking syscall sequences.
- **Caddy** uses `runtime/debug.SetGCPercent` for tuning per workload.

## Source Code References

Pinned to `go1.26`.

- `runtime`: [`src/runtime/`](https://github.com/golang/go/tree/master/src/runtime).
- `runtime/metrics`: [`src/runtime/metrics/`](https://github.com/golang/go/tree/master/src/runtime/metrics).
- `runtime/debug`: [`src/runtime/debug/`](https://github.com/golang/go/tree/master/src/runtime/debug).
- `AddCleanup` (1.24): [`src/runtime/mfinal.go`](https://github.com/golang/go/blob/master/src/runtime/mfinal.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/runtime, /runtime/debug, /runtime/metrics.
- Go 1.24 release notes — `AddCleanup`: https://go.dev/doc/go1.24.
- "Automatic Memory Management in Go" — Go GC design docs.

## Exercises / Self-Check

1. Print all goroutine stacks at SIGUSR2. Wire up `signal.Notify` + `runtime.Stack`.
2. Use `runtime/metrics` to print heap-in-use and goroutine count every 5 seconds.
3. Compare `runtime.SetFinalizer` and `runtime.AddCleanup`. Demonstrate one downside of finalizers (resurrection or delayed reclamation).
4. Set `GOMAXPROCS=1` and run a tight loop in one goroutine plus a tick in another. Why does the tick still fire (since 1.14)?
5. Read your binary's build info with `debug.ReadBuildInfo()`.
