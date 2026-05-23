# `go tool trace` — The Execution Tracer

## TL;DR

The execution tracer records **every interesting runtime event**: goroutine create/start/stop/block/unblock, GC cycles, syscalls, network polls, GOMAXPROCS changes, user-defined regions and tasks. Where `pprof` answers "where is time spent?", the tracer answers "*why* did this goroutine wait?". Trace data is captured via `runtime/trace.Start(w)` or by hitting `/debug/pprof/trace?seconds=N`; analyzed via `go tool trace`. The format and tracer were **rewritten in Go 1.21** (it had been ~unchanged since 1.5) — the new tracer is lower-overhead (~1%), supports flight-recorder mode (continuous, in-memory buffer), and produces a smaller more parseable file. The viewer launches a browser UI with timelines, goroutine analysis, network/syscall blocking views, and (1.22+) "minimum mutator utilization" GC graphs.

## Mental Model

```
   running process
        │
        ├─ runtime/trace.Start(w)            ── manual
        ├─ /debug/pprof/trace?seconds=N      ── HTTP
        ├─ runtime/trace.NewTask + .Region   ── user annotations
        │
        ▼
   trace events stream  (since 1.21: more compact, per-M buffers, flight recorder)
        │
        ▼
   trace file
        │
        ▼
   go tool trace trace.out
        │
        ▼
   browser UI:
        ├─ View trace             ── chrome://tracing-style timeline
        ├─ Goroutine analysis     ── per-G time breakdown
        ├─ Network blocking       ── poller events
        ├─ Sync blocking          ── chan/mutex
        ├─ Syscall blocking       ── kernel waits
        ├─ Scheduler latency      ── runqueue waits
        ├─ User-defined tasks     ── from runtime/trace
        └─ Goroutine creation     ── who created whom
```

The tracer is *event-based*, not sample-based. Every event is logged with a timestamp; the viewer reconstructs the timeline.

## Syntax & Basic Usage

```bash
# Collect
$ go test -trace=trace.out ./pkg
$ curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5'

# Programmatic
import "runtime/trace"
f, _ := os.Create("trace.out")
defer f.Close()
trace.Start(f)
defer trace.Stop()

# Analyze
$ go tool trace trace.out          # opens browser UI
$ go tool trace -http=:8080 trace.out
$ go tool trace -pprof=net trace.out > net.pprof    # extract sub-profile
$ go tool trace -pprof=sync trace.out > sync.pprof
$ go tool trace -pprof=syscall trace.out > sys.pprof
$ go tool trace -pprof=sched trace.out > sched.pprof
```

## Deep Dive

### The 1.21 tracer rewrite

Before 1.21, the tracer:

- Wrote events into a single global buffer (lock contention).
- Stopped the world to start/stop.
- Cost ~30% CPU during capture.
- Produced files that ballooned: 100 MB/s for a busy service.
- Could only be enabled via `runtime/trace.Start`.

After 1.21:

- Per-M (per-OS-thread) buffers — no global lock during capture.
- No stop-the-world.
- Overhead reduced to ~1%.
- Smaller, more compressible files (~10 MB/s).
- Supports **flight recorder** mode: a circular in-memory buffer that you dump on demand.
- Forward-compatible event schema.

The on-disk format changed; old `go tool trace` (1.20 and earlier) can't read 1.21+ files. Always use the same major version for capture and analysis.

### Flight recorder (1.22+)

```go
import "runtime/trace"

fr := trace.NewFlightRecorder()
fr.Start()
defer fr.Stop()

// On demand (e.g., on a panic or high-latency request):
f, _ := os.Create("flight.out")
fr.WriteTo(f)
f.Close()
```

Keeps the last N seconds of trace events in memory. When something interesting happens (panic, slow request, alert), dump the buffer — you get the events leading up to the moment, not a fresh capture starting from now.

Tunable buffer size and duration:

```go
fr := trace.NewFlightRecorder()
fr.SetSize(time.Minute, 64<<20)   // 1 minute or 64 MB, whichever first
fr.Start()
```

Flight recorder is the killer feature for production tracing: you don't pay for "always-on full capture" but can grab a snapshot when needed.

### User-defined tasks and regions

```go
import "runtime/trace"

func handleRequest(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    ctx, task := trace.NewTask(ctx, "handleRequest")
    defer task.End()

    region := trace.StartRegion(ctx, "lookup")
    user := lookupUser(ctx, r.URL.Path)
    region.End()

    region = trace.StartRegion(ctx, "render")
    renderUser(w, user)
    region.End()
}
```

**Tasks** are long-lived operations (often per-request); **regions** are shorter spans. The trace viewer's "User-defined tasks" tab groups events by task and shows each region as a labeled block. Critical for understanding "where did this 50 ms request spend its time?".

### Logs

```go
trace.Log(ctx, "user_id", "42")
```

Annotates the trace with a key/value pair attached to the current task. Visible in the viewer's task detail.

### Browser UI tabs

After `go tool trace trace.out` opens your browser:

| Tab                          | What you see                                                       |
|------------------------------|--------------------------------------------------------------------|
| View trace                   | Chrome-tracing-style timeline; one row per processor (P).         |
| Goroutine analysis           | Per-goroutine time breakdown (running, runnable, blocked).        |
| Network blocking             | Network poller events.                                            |
| Synchronization blocking     | Channel/mutex waits.                                              |
| Syscall blocking             | System call durations.                                            |
| Scheduler latency profile    | Time from "ready" to "running".                                   |
| User-defined tasks           | Your `trace.NewTask` calls.                                       |
| User-defined regions         | Your `trace.StartRegion` calls.                                   |
| Minimum mutator utilization  | (1.22+) GC pause percentile graph.                                |

### Reading the timeline

Each row in the main "View trace" is a processor (P), not a goroutine. A bar on the row represents a goroutine running on that P. Click a bar to see the goroutine's stack at that point. Right-click to navigate (zoom, follow goroutine, etc.).

Vertical pink bars are GC cycles. Bars labeled `GC (background sweep)` or `STW (mark termination)` show GC phases.

Below the P rows: "Goroutines" lane shows the running count over time; "Heap" lane shows heap size; "Threads" lane shows OS thread count.

### Goroutine analysis

```
Click "Goroutine analysis" → table of goroutines by ID/name
Click a name → time breakdown:
   Execution time:    150ms
   Sync block time:    50ms       ← waiting on a mutex or channel
   Sched wait time:    20ms       ← runqueue wait
   Syscall block time: 80ms       ← in kernel
   GC pause time:      10ms
```

The biggest blocking category is where to look. "Sync block" → check channel/mutex usage. "Syscall block" → I/O is dominating. "Sched wait" → too many runnable goroutines for the available Ps.

### Extracted profiles (`-pprof=`)

```bash
$ go tool trace -pprof=net  trace.out > net.pprof
$ go tool pprof -http=:8080 net.pprof
```

Generates a pprof-compatible profile of blocking events. Useful when you want to use pprof's flame graph for blocking analysis instead of the trace UI.

Available extractions: `net`, `sync`, `syscall`, `sched`.

### Trace from `go test`

```bash
$ go test -trace=trace.out -bench=. ./pkg
$ go tool trace trace.out
```

Profiling a benchmark gives you a precisely-bounded workload. Combine with `-cpuprofile` for both views.

### Tracing in production

Three modes:

1. **On-demand HTTP**: `curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5'`. Captures 5 s; analyze locally.
2. **Continuous to disk**: rare; trace files are big. Use only when chasing a specific reproducible issue.
3. **Flight recorder**: keeps last N seconds in RAM; dump on signal or trigger. Best for production "what happened just before this alert?".

### File size

A 5-second trace on a moderately busy service: ~50 MB pre-1.21, ~5 MB on 1.21+. Compressed (gzip): ~1 MB. Don't ship uncompressed.

### Tracing in cgo / external code

Goroutines blocked in cgo show as "syscall block". The trace can't see inside C code; the timeline gap is opaque. Use `perf trace` on the C side for cgo-heavy workloads.

### `GODEBUG=tracefpunwindoff=1`

Disables the frame-pointer-based unwinder used by the 1.21+ tracer on amd64/arm64; falls back to gentry-style. Slower but more compatible. You'd only set this when chasing a tracer bug.

## Standard Library Hooks

- `runtime/trace` — programmatic trace start/stop, tasks, regions, logs.
- `runtime/trace.NewFlightRecorder` (1.22+) — circular buffer.
- `net/http/pprof.Trace` — HTTP handler.
- `runtime/pprof` — different but related; see `09-tooling/12-go-tool-pprof.md`.
- `cmd/trace` — the viewer source.

## Real-World Patterns

### 1. Quick capture during load

```bash
$ # in another terminal: drive load
$ wrk -t4 -c100 -d10s http://localhost:8080/
$ # capture
$ curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5'
$ go tool trace trace.out
```

### 2. Per-request task

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, task := trace.NewTask(r.Context(), "handler:"+r.URL.Path)
    defer task.End()
    serve(w, r.WithContext(ctx))
}
```

In the viewer, "User-defined tasks" groups by name → see distribution of request times.

### 3. Investigating a tail-latency spike

Enable flight recorder; on a high-latency request, dump the buffer:

```go
fr := trace.NewFlightRecorder()
fr.SetSize(30*time.Second, 64<<20)
fr.Start()

func wrap(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        h.ServeHTTP(w, r)
        if d := time.Since(start); d > 500*time.Millisecond {
            f, _ := os.Create(fmt.Sprintf("slow-%d.trace", time.Now().Unix()))
            fr.WriteTo(f); f.Close()
        }
    })
}
```

### 4. Extract sync-blocking profile

```bash
$ go tool trace -pprof=sync trace.out > sync.pprof
$ go tool pprof -http=:8081 sync.pprof
```

### 5. GC analysis

In the viewer, "View trace" → look for pink GC bars and their duration; "Minimum mutator utilization" (1.22+) → shows percentile of CPU available to user code over windows of various lengths. Low MMU = GC starvation.

### 6. Find a goroutine leak

In the viewer's "Goroutines" lane: an unbounded climb over time = leak. Cross-check with `/debug/pprof/goroutine?debug=1` for stacks.

### 7. Confirm `GOMAXPROCS` is right

Watch the P count in the viewer. If many goroutines are "sched wait" (in runqueue but not running), you may need more Ps. If Ps are idle, GOMAXPROCS is fine — work is I/O bound, not CPU.

## Anti-Patterns & Gotchas

**Tracing for too long in production.** A 60-second trace is 60 MB+ uncompressed; the file is harder to load. Aim for 5–15 s captures.

**Tracing without driving load.** A 5-s trace of an idle service shows nothing useful. Generate traffic.

**Confusing P count with goroutine count.** The viewer has one row per P (usually = `GOMAXPROCS`), not per goroutine. Use "Goroutine analysis" to see per-G.

**Using flight recorder without sizing the buffer.** Default may be too small (events fall off the back); too large wastes RAM. Tune based on event rate.

**Trusting "syscall block time" without context.** A long syscall block could be normal (epoll_wait on a quiet socket). Compare to peers.

**Old trace viewer on new files.** Pre-1.21 viewer can't read 1.21+ traces. Match toolchain versions.

**Forgetting `runtime.GC()` before short benchmark traces.** GC events from the previous workload pollute the trace. Trigger a GC before starting.

**Trying to read megabyte traces in a memory-constrained machine.** The viewer loads the whole file. Use a beefy local machine, not the production box.

**Tracing through a tunnel without compression.** Trace files are highly compressible. `gzip` before copying off the box.

**Expecting tracer to symbolicate cgo.** It can't; cgo time appears as "syscall block" with no function info on the C side.

**User tasks with thousands of regions per task.** Each region is an event; lots of events make the viewer sluggish. Use sparingly inside hot loops.

## Performance Notes

- Tracer overhead (1.21+): ~1% wall time.
- Tracer overhead (pre-1.21): ~30% wall time.
- File size (1.21+): ~10 MB/s for a busy service; ~1 MB/s compressed.
- Capture latency: ~5 ms to start, ~5 ms to stop.
- Flight recorder: marginal extra cost over normal tracing.
- Viewer startup: 1–10 s for typical files; can take minutes for 1GB+ traces.

`runtime/trace.Log` is cheap (~ns); `trace.NewTask` slightly more (~10 ns).

## How Big Companies Use It

- **Google** uses the tracer heavily; the 1.21 rewrite was driven by their internal scale: https://go.dev/blog/execution-traces.
- **Uber** uses traces alongside their internal profiling system to debug latency tail issues in `cadence` and `temporal`: https://github.com/temporalio/temporal.
- **Cloudflare** uses flight recorder mode in their Workers Go runtime to capture rare panics with full context: https://blog.cloudflare.com.
- **Datadog** scrapes `/debug/pprof/trace` periodically as part of their APM offering: https://docs.datadoghq.com.
- **CockroachDB** integrates traces into their cluster debug bundle: https://www.cockroachlabs.com.
- **The Go team** uses traces to optimize the runtime (scheduler tweaks, GC tuning): https://go.dev/blog.
- **Tailscale** uses flight recorder to investigate intermittent peer-connection latency: https://tailscale.com/blog.

## Source Code References

Pinned to `go1.26`.

- `runtime/trace`: [`src/runtime/trace`](https://github.com/golang/go/tree/release-branch.go1.26/src/runtime/trace) and [`src/runtime/traceback.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/traceback.go).
- Tracer (1.21+): [`src/runtime/traceruntime.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/traceruntime.go).
- Trace viewer: [`src/cmd/trace`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/trace).
- Trace v2 parser: [`src/internal/trace`](https://github.com/golang/go/tree/release-branch.go1.26/src/internal/trace).
- Flight recorder: [`src/runtime/trace/flightrecorder.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/trace/flightrecorder.go).
- HTTP handler: [`src/net/http/pprof/pprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/pprof/pprof.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Execution traces in Go 1.21" (Michael Knyszek): https://go.dev/blog/execution-traces.
- "An introduction to the Go execution tracer" (Dmitry Vyukov): https://go.dev/blog/execution-trace.
- "Go execution tracer design doc" (Michael Knyszek): https://go.googlesource.com/proposal/+/master/design/68272-execution-traces.md.
- "Flight recorder proposal": https://go.googlesource.com/proposal/+/master/design/63185-trace-flight-recorder.md.
- "Profiling and execution tracing" (Felix Geisendörfer): https://www.youtube.com/watch?v=Z4FvSWgFcGY.

## Exercises / Self-Check

1. Add `trace.NewTask` + `trace.StartRegion` to an HTTP handler. Capture a trace, find the task in the User-defined tasks tab, and identify which region dominates.
2. Set up a flight recorder with a 30-second buffer. Trigger a dump on the first request that takes >100 ms. What do you see in the seconds leading up to it?
3. Run `go tool trace -pprof=sync trace.out` and analyze the resulting profile. Which lock contended most?
4. View the timeline during a GC cycle. How long was the STW phase? Compare to the cycle's total wall time.
5. Capture a 5-second trace before and after switching from `sync.Mutex` to `sync.RWMutex` in a hot path. Did "Synchronization blocking" decrease?
