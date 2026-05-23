# runtime/pprof Internals — How the Profilers Work

## TL;DR

Go ships **five built-in profilers**: CPU (signal-driven sampling at 100 Hz), heap (allocation-sampled), goroutine (full or sampled stacks), block (sampled blocking events on channels/mutexes), and mutex (sampled contention). Each writes to a **protobuf** in the pprof format, readable by `go tool pprof`. CPU profiling installs a SIGPROF timer that interrupts goroutines and records the current stack; allocation profiling samples ~1 in every 512 KiB of allocations and stores the stack alongside the allocation. The single biggest gotcha: the **CPU profiler relies on SIGPROF**, which the OS may deliver to *any* thread — including Ms in syscalls or cgo. If your hot work is in cgo, the profiler can blame the wrong thing.

## Mental Model

```
   Timer-driven (CPU profile):
   ─────────────────────────────
   OS sets ITIMER_PROF, fires SIGPROF every 10 ms (default).
   Signal arrives on a random M:
       sigprof handler →
           gentraceback(currentPC, currentSP) →
           write (count, stack_id) to a lock-free buffer.
   Background goroutine (profBuf reader):
       drain buffer → encode as pprof.Profile → io.Writer.

   Allocation-driven (heap profile):
   ─────────────────────────────────
   On every allocation, mcache.nextSample counts down.
   When zero: capture stack, store in mprof.mbucket. Reset counter to
   exponentially-distributed next sample point.
   At profile time: walk buckets, emit pprof.

   Event-driven (block, mutex profile):
   ────────────────────────────────────
   When sync primitives experience contention or block, the runtime
   checks a sampling rate (1/N). If sampled, recordStack + accumulate
   duration into a per-stack bucket.

   Goroutine profile:
   ──────────────────
   On request: gentraceback every G, write each to the profile.
   (Full STW for 'all' mode; sampled mode is lighter.)
```

All profilers use the same `gentraceback` machinery (see `12-runtime/08-traceback-and-panic.md`).

## Syntax & Basic Usage

```go
package main

import (
	"os"
	"runtime/pprof"
)

func main() {
	// CPU profile
	f, _ := os.Create("cpu.prof")
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()
	defer f.Close()

	work()

	// Heap snapshot
	hf, _ := os.Create("heap.prof")
	pprof.WriteHeapProfile(hf)
	hf.Close()
}

func work() {}
```

For HTTP-enabled introspection (the canonical pattern):

```go
package main

import (
	_ "net/http/pprof"
	"net/http"
)

func main() {
	go http.ListenAndServe("localhost:6060", nil)
	// app code
}
```

Then:

```
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30   # CPU
go tool pprof http://localhost:6060/debug/pprof/heap                  # heap
go tool pprof http://localhost:6060/debug/pprof/goroutine             # goroutines
go tool pprof http://localhost:6060/debug/pprof/allocs                # allocations
go tool pprof http://localhost:6060/debug/pprof/block                 # blocking
go tool pprof http://localhost:6060/debug/pprof/mutex                 # mutex contention
```

## Deep Dive

### CPU profiling

`pprof.StartCPUProfile(w)`:

1. Sets `SetCPUProfileRate(100)` — 100 Hz default (10 ms period).
2. Installs SIGPROF handler.
3. OS arms `setitimer(ITIMER_PROF, ...)` on every M (since profile is per-thread).
4. Spawns a background goroutine to drain the profile buffer and write to `w`.

The signal handler (`sigprof` in `runtime/proc.go`):

```go
// runtime/proc.go (excerpt; conceptual)
func sigprof(pc, sp, lr uintptr, gp *g, mp *m) {
    if !haveLockToProfile { return }
    var stk [maxStack]uintptr
    n := gentraceback(pc, sp, lr, gp, 0, &stk[0], maxStack, ...)
    cpuprof.add(gp, stk[:n])
}
```

`cpuprof.add` appends `(gp, stk[:n])` to a **lock-free ring buffer** (`runtime.profBuf`). The reader goroutine pops batches and emits to the writer.

#### Why 100 Hz?

ITIMER_PROF resolution is OS-dependent. 100 Hz is a balance: fine enough to catch ~10 ms hot regions, coarse enough that signal overhead is <1%. Increase with `runtime.SetCPUProfileRate(rate)` — up to a few thousand, beyond which signal storms appear.

#### Why SIGPROF can miss

- **Syscall blocking.** ITIMER_PROF counts only on-CPU time in the process. A thread sleeping in `read()` doesn't get SIGPROF — that's the correct semantics (it's not consuming CPU).
- **Cgo calls.** SIGPROF arrives on the thread; if the thread is in C code, Go's handler still runs but `gentraceback` may give a degenerate trace ("cgo" frame) unless you registered a C unwinder via `runtime.SetCgoTraceback`.
- **Brief functions.** A function that runs for 3 ms between two SIGPROF ticks may go entirely unrepresented. Long enough samples average out.
- **Signal queue overflow.** Linux drops signals if delivery is too fast. Symptom: `runtime: cpu profiling signal dropped` on stderr. Reduce rate.

### Heap profiling

Sampling is **proportional to allocation size**. On every `mallocgc`, the runtime decrements `mcache.nextSample` by the allocation's size. When it reaches zero, it captures a stack.

```go
// runtime/malloc.go (excerpt; conceptual)
if rate := MemProfileRate; rate > 0 {
    if c.nextSample -= size; c.nextSample <= 0 {
        profilealloc(c, x, size) // capture stack
        c.nextSample = nextSample() // exponential distribution
    }
}
```

`MemProfileRate` defaults to **512 KiB** (each 512 KiB of allocations triggers, on average, one sample). Set with `runtime.MemProfileRate = N` before allocations occur.

`nextSample()` draws from a Poisson-style exponential distribution so individual allocations have a probability proportional to their size — large allocations almost always sampled, tiny ones rarely.

Each sample stores: `(allocStack, inUseObjects, inUseBytes, allocObjects, allocBytes)`. Stored in **buckets** keyed by stack identity. `pprof` can report either `inuse_*` (live now) or `alloc_*` (cumulative).

### Block profile

Records goroutines that **blocked**. Captured on:

- Channel send/receive that didn't complete immediately.
- `sync.Mutex.Lock` that contended.
- `sync.WaitGroup.Wait` that blocked.
- `sync.Cond.Wait`.
- `runtime.gopark` more generally.
- Select that blocked.

```go
runtime.SetBlockProfileRate(rate int)
```

Argument is **expected block duration in nanoseconds**: only events that blocked for ≥ rate ns get sampled (probabilistically). `1` means sample everything (expensive); `10000` means ~10 µs events; 0 means off (default).

Each sample stores: `(stack, count, delay)`.

### Mutex profile

Records lock-holder contention — when a goroutine releases a contended mutex.

```go
runtime.SetMutexProfileFraction(rate int)
```

Argument is **1 in rate**: 1 = sample every contention, 100 = 1%, 0 = off.

Records: `(stack of unlocker, count, delay-of-waiters)`. Note: the stack is of the *unlocker* (the one that held the lock and made others wait), not the waiters. So a hot lock shows up under the function holding it, which is usually what you want.

### Goroutine profile

Two modes:

- `pprof.Lookup("goroutine").WriteTo(w, 0)` — proto pprof format, **STW** to walk all stacks consistently.
- `pprof.Lookup("goroutine").WriteTo(w, 1)` — text format with full stacks; same STW.
- `pprof.Lookup("goroutine").WriteTo(w, 2)` — text format, no STW (best-effort).

Cost is O(num goroutines × stack depth). On 1M-goroutine processes, ~100 ms STW. Use the no-STW mode for monitoring.

### `runtime/pprof` API

```go
// Predefined named profiles
pprof.Lookup("goroutine")
pprof.Lookup("heap")
pprof.Lookup("allocs")        // alias for heap
pprof.Lookup("threadcreate")
pprof.Lookup("block")
pprof.Lookup("mutex")

// CPU
pprof.StartCPUProfile(w io.Writer) error
pprof.StopCPUProfile()
pprof.SetCPUProfileRate(hz int)

// User profiles (since 1.4): labeled key/value sets attached to goroutines
pprof.SetGoroutineLabels(ctx context.Context)
pprof.Do(ctx context.Context, labels pprof.LabelSet, f func(ctx context.Context))
pprof.Labels("key", "value")
```

### Pprof labels

Attach labels to a goroutine for the duration of `pprof.Do`:

```go
package main

import (
	"context"
	"runtime/pprof"
)

func handle(ctx context.Context, req *Request) {
	pprof.Do(ctx, pprof.Labels("endpoint", req.Path, "user", req.UserID), func(ctx context.Context) {
		serve(ctx, req)
	})
}

type Request struct{ Path, UserID string }
func serve(context.Context, *Request) {}
```

CPU samples then carry these labels; `go tool pprof -tagfocus=endpoint=/api/v1 cpu.prof` filters to that subset. Crucial for production tracing.

### Buffer and writer side

The CPU profile buffer is `runtime.profBuf`, a lock-free ring. The reader goroutine ([`runtime/pprof/pprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/pprof/pprof.go)) drains and writes to the user's io.Writer. If the reader is slow, the buffer can drop entries — pprof emits `runtime: cpu profiling signal dropped` and the resulting profile is incomplete. The default size (~1 MiB) is enough for tens of seconds at 100 Hz.

### `runtime.SetCgoTraceback`

When SIGPROF arrives during a cgo call, `gentraceback` can't walk past the Go ⟷ C boundary by default. `SetCgoTraceback` registers C-side unwinder callbacks (typically backed by libunwind). Without it, profiles show CGO frames as opaque.

The bridge is implemented in [`src/runtime/traceback.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/traceback.go).

### Execution tracer (different from pprof)

`runtime/trace` is **event-based**, not sampled: every G state transition, syscall, GC phase, mark assist, and netpoll event is logged. The output is much bigger and meant for `go tool trace` (a UI). Not pprof format. Covered in `09-tooling/13-go-tool-trace.md`.

### Profile cost summary

| Profile | Cost |
|---|---|
| CPU (100 Hz) | ~1% steady |
| Heap (rate=512 KiB, default) | ~0.5% on allocator path |
| Heap (rate=1) | ~10–30%, every allocation samples |
| Block (rate=10000 ns) | ~negligible |
| Mutex (rate=100) | <1% |
| Goroutine (lookup) | STW O(NumG); ms-scale for 100k Gs |

### Encoding format

pprof's output is a `gzip(protobuf(profile.proto))`. The schema is at [github.com/google/pprof/proto](https://github.com/google/pprof/blob/master/proto/profile.proto). Each profile contains:

- A *symbolized* mapping (function names ↔ PCs).
- Sample types: `[("cpu","nanoseconds"), ("samples","count")]` etc.
- Samples: `(stack_id, [values], labels)`.
- Locations: per-PC info, including inlining info.

`go tool pprof` parses this and offers `top`, `list`, `web`, `peek`, `disasm` commands.

## Standard Library Hooks

- `runtime/pprof.StartCPUProfile`, `StopCPUProfile`, `SetCPUProfileRate`.
- `runtime/pprof.WriteHeapProfile`, `Lookup`, `Profile.WriteTo`.
- `runtime/pprof.Do`, `SetGoroutineLabels`, `Labels`, `LabelSet`.
- `runtime.SetBlockProfileRate`, `runtime.SetMutexProfileFraction`.
- `runtime.MemProfileRate` (variable).
- `runtime.SetCgoTraceback`.
- `net/http/pprof` — registers `/debug/pprof/*` handlers on `http.DefaultServeMux`.
- `runtime/trace.Start`, `Stop` — execution tracer.
- `go tool pprof` — view, summarize, compare profiles.

## Real-World Patterns

### 1. Always-on profiling endpoint

```go
package main

import (
	"net/http"
	_ "net/http/pprof"
)

func main() {
	go func() {
		_ = http.ListenAndServe("localhost:6060", nil)
	}()
	serve()
}

func serve() {}
```

Bind to `localhost` (or a UNIX socket) to avoid exposing it. In production, gate it behind a sidecar or VPN.

### 2. Periodically dump heap to disk

```go
package main

import (
	"fmt"
	"os"
	"runtime/pprof"
	"time"
)

func main() {
	go func() {
		t := time.NewTicker(5 * time.Minute)
		for range t.C {
			name := fmt.Sprintf("heap-%d.prof", time.Now().Unix())
			f, _ := os.Create(name)
			pprof.WriteHeapProfile(f)
			f.Close()
		}
	}()
	serve()
}

func serve() {}
```

Lets you diff two heap profiles across an OOM event: `go tool pprof -base heap-T0.prof heap-T1.prof`.

### 3. Label CPU samples by request

```go
package main

import (
	"context"
	"net/http"
	"runtime/pprof"
)

func Middleware(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		labels := pprof.Labels("endpoint", r.URL.Path, "method", r.Method)
		pprof.Do(r.Context(), labels, func(ctx context.Context) {
			h.ServeHTTP(w, r.WithContext(ctx))
		})
	})
}
```

Now `go tool pprof -tagfocus=endpoint=/api/v1/users cpu.prof` shows where one endpoint spends its CPU.

### 4. On-demand profile, capture-and-flush

```go
package main

import (
	"bytes"
	"net/http"
	"runtime/pprof"
	"time"
)

func cpuProfile30s(w http.ResponseWriter, r *http.Request) {
	buf := &bytes.Buffer{}
	if err := pprof.StartCPUProfile(buf); err != nil {
		http.Error(w, err.Error(), 500)
		return
	}
	time.Sleep(30 * time.Second)
	pprof.StopCPUProfile()
	w.Header().Set("Content-Type", "application/octet-stream")
	_, _ = w.Write(buf.Bytes())
}
```

Equivalent to `/debug/pprof/profile?seconds=30`. Useful when you can't enable `net/http/pprof` globally.

### 5. Continuous profiling (Pyroscope/Parca/Polar Signals)

These tools sample profiles every few seconds, ship them, and let you scroll through CPU/heap/block over time. Setup uses the standard pprof endpoints; the tools just collect and store. Typical deployment:

```yaml
# pyroscope agent config
target: my-go-app
url: http://my-go-app:6060/debug/pprof/profile
type: cpu
period: 10s
```

## Anti-Patterns & Gotchas

**Running CPU profile in production with rate=1000.** Signal storms, dropped samples, distorted results. Default 100 Hz is right.

**Trusting CPU profiles when most time is in cgo.** Install `SetCgoTraceback` and a C unwinder. Otherwise cgo time shows as opaque.

**`MemProfileRate=1` in production.** Each allocation samples — massive slowdown. Use default 512 KiB; lower only in dev.

**Block/mutex profile permanently on.** They're cheap but not free; enable when diagnosing, disable otherwise. Especially block profile with rate=1.

**Reading `goroutine?debug=2` on a stuck process at 1 Hz.** This is full STW; you'll worsen the stuck condition.

**`pprof` profile names interpreted as paths.** They're protobuf blobs. `go tool pprof file.pb.gz` (or any extension) — extension doesn't matter.

**Comparing profiles across binaries with different optimization flags.** Inlining changes the function tree; comparisons aren't apples-to-apples.

**Profiling a binary built with `-trimpath` but no `-buildvcs`.** Stack traces show file paths as the trimmed prefix, breaking symbolication. Either include `-buildvcs` or ship the source tree alongside.

**Sampling at high rates and forgetting to call `StopCPUProfile`.** The profile file remains incomplete; the goroutine continues writing. Always `defer pprof.StopCPUProfile()`.

**Believing block/mutex profile shows where the *waiters* are.** It shows the lock-holder's release point (for mutex) and the waker's site (for block). For waiter traces, use goroutine profile.

## Performance Notes

- CPU profile overhead at 100 Hz: ~0.5–1% per CPU.
- CPU profile overhead at 1000 Hz: ~5–10%, with signal-storm risk.
- Heap profile (default rate): ~0.5% added to allocator hot path.
- Heap profile rate=1: 10–30% slowdown on allocator-heavy workloads.
- Block profile (rate=10000 ns): negligible on typical workloads.
- Mutex profile (rate=100): <1% on contention-heavy workloads.
- Goroutine `?debug=2`: O(N × depth); 1M goroutines = ~100 ms STW.
- `runtime/trace`: 1–5% steady, far higher disk throughput. Use bursts of 30 s, not continuous.

## How Big Companies Use It

- **Google's continuous profiling** (Daven Cargo, GopherCon 2017) shipped pprof samples from production fleet-wide.
- **Pyroscope / Grafana** — open-source continuous profiling: https://pyroscope.io.
- **Polar Signals (Parca)** — pprof + eBPF profiling: https://www.polarsignals.com.
- **Datadog continuous profiler** — same pprof endpoints, multi-language support.
- **Cloudflare** documents block profile sampling rates for their edge: https://blog.cloudflare.com.
- **Discord** used pprof heap diffs to localize their GC-time growth that drove the Rust migration: https://discord.com/blog/.
- **Uber** open-sourced `uber-go/automaxprocs` partly to make pprof CPU profiles in containers reflect actual container CPU.
- **Tailscale** uses `pprof.Labels` for per-tunnel labeling: https://tailscale.com/blog/.
- **Kubernetes apiserver** ships pprof endpoints by default on port 10257 — invaluable for production diagnostics.

## Source Code References

Pinned to `go1.26`.

- pprof user-facing API: [`src/runtime/pprof/pprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/pprof/pprof.go).
- CPU profile internals: [`src/runtime/cpuprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/cpuprof.go).
- Heap profile (`mbucket`, `mprof_*`): [`src/runtime/mprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mprof.go).
- Block & mutex profile: same file, search `blocksample` and `mutexevent`.
- Signal-side sampling: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go) — function `sigprof`.
- Allocation sampling: [`src/runtime/malloc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/malloc.go) — search `nextSample`.
- profBuf (lock-free ring): [`src/runtime/profbuf.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/profbuf.go).
- Goroutine labels: [`src/runtime/pprof/label.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/pprof/label.go).
- `net/http/pprof`: [`src/net/http/pprof/pprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/pprof/pprof.go).
- pprof proto: https://github.com/google/pprof/blob/main/proto/profile.proto.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Profiling Go Programs" (official Go blog): https://go.dev/blog/pprof.
- Rhys Hiltner, "An introduction to `runtime/pprof`": https://about.sourcegraph.com/podcast/rhys-hiltner.
- Felix Geisendörfer, "Mutex profiles": https://felixge.de/.
- Damian Gryski, "go-perfbook — profiling": https://github.com/dgryski/go-perfbook.
- Polar Signals' parca docs: https://www.parca.dev/docs.
- Pyroscope's Go pprof guide: https://pyroscope.io/docs/golang/.
- Russ Cox, "Profile-guided optimization" — sets context for how pprof feeds PGO: https://go.dev/blog/pgo.
- Carlos Castillo, "Continuous profiling at scale" (Datadog blog): https://www.datadoghq.com/blog/engineering/.

## Exercises / Self-Check

1. You see `runtime: cpu profiling signal dropped` in stderr. What's happening and how do you fix it?
2. A heap profile shows huge allocations from `bufio.NewReader`. Is it actually `bufio.NewReader` that's growing the heap, or the caller? How do you disambiguate?
3. Set up `pprof.Do` labels for HTTP requests in a small server. Capture a CPU profile under load and filter to a single endpoint.
4. Why does the mutex profile attribute samples to the lock-holder rather than the waiter?
5. Run a CPU profile at 1000 Hz in a tight allocator loop. Observe the dropped-signal warning. Tune to the highest rate where you don't lose samples.
