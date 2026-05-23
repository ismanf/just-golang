# `go tool pprof` — Profile Viewer

## TL;DR

`pprof` (originally from Google's perftools, now part of Go) consumes profile files in the protobuf-based `profile.proto` format and renders them as text, graphs, flamegraphs, or an interactive web UI. Go produces profiles three ways: (1) `go test -cpuprofile=cpu.out`, (2) `net/http/pprof` HTTP endpoints on a running service (`/debug/pprof/profile`, `/debug/pprof/heap`, etc.), (3) programmatic via `runtime/pprof`. The profile types are **CPU**, **heap** (live + alloc), **goroutine**, **block**, **mutex**, **threadcreate**, **allocs**, and the unified **fgprof** (1.18+ via `runtime/pprof.WriteHeapProfile` style). `pprof`'s killer feature is `-http=:8080`: a web UI with flamegraph, source-annotated listing, peek view, and call-graph. Read it as: **CPU profiles are sampled at ~100Hz; heap profiles are sampled at every 512KB allocation by default; counts are estimates**.

## Mental Model

```
   running process / test
        │
        ├─ runtime/pprof.StartCPUProfile() → cpu.out
        ├─ runtime/pprof.WriteHeapProfile() → heap.out
        ├─ net/http/pprof handlers → /debug/pprof/profile, /heap, /goroutine, ...
        │
        ▼
   profile.proto encoded file (gzipped)
        │
        ▼
   go tool pprof <profile>     ── interactive REPL
   go tool pprof -http=:8080   ── web UI (flamegraph, graph, source view)
   go tool pprof -top          ── batch text output
   go tool pprof -list Regexp  ── source-annotated listing
   go tool pprof -base old new ── differential profile
```

A profile file is just samples (PCs + count/cost). pprof's job is to symbolicate (PC → function), aggregate, and render.

## Syntax & Basic Usage

```bash
# Collect via go test
$ go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out -blockprofile=block.out ./pkg

# Collect from running service
$ curl -o cpu.out http://localhost:6060/debug/pprof/profile?seconds=30
$ curl -o heap.out http://localhost:6060/debug/pprof/heap
$ curl -o goroutine.out http://localhost:6060/debug/pprof/goroutine
$ curl -o block.out http://localhost:6060/debug/pprof/block
$ curl -o mutex.out http://localhost:6060/debug/pprof/mutex
$ curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5'

# Analyze
$ go tool pprof cpu.out
(pprof) top
(pprof) top -cum
(pprof) list main.Hot
(pprof) web                       # opens .svg call graph in browser
(pprof) flamegraph                # opens flamegraph in browser
(pprof) traces                    # call stacks
(pprof) peek funcName             # callers/callees of funcName
(pprof) disasm funcName

# Web UI (preferred)
$ go tool pprof -http=:8080 cpu.out

# Compare two profiles
$ go tool pprof -http=:8080 -base old.out new.out

# Save text reports
$ go tool pprof -top cpu.out > top.txt
$ go tool pprof -tree -cum cpu.out > tree.txt
$ go tool pprof -svg cpu.out > graph.svg
$ go tool pprof -png cpu.out > graph.png
```

## Deep Dive

### Enabling `net/http/pprof`

```go
import (
    "net/http"
    _ "net/http/pprof"          // registers handlers on DefaultServeMux
)

func main() {
    go http.ListenAndServe("localhost:6060", nil)
    // ... rest of your service
}
```

The blank import registers `/debug/pprof/*` on `http.DefaultServeMux`. Always bind to `localhost` (or behind a firewall/VPN); these endpoints expose internals.

For an isolated mux:

```go
import "net/http/pprof"

func setupDebug(mux *http.ServeMux) {
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
}
```

### Available endpoints

| Endpoint                   | Profile type   | Typical cost / duration       |
|----------------------------|----------------|-------------------------------|
| `/debug/pprof/profile`     | CPU            | `?seconds=30` (default 30 s) |
| `/debug/pprof/heap`        | Heap (live)    | Instant snapshot              |
| `/debug/pprof/allocs`      | Heap (alloc)   | Instant snapshot              |
| `/debug/pprof/goroutine`   | Goroutine      | Instant; `?debug=2` for stacks|
| `/debug/pprof/block`       | Blocking       | Requires `runtime.SetBlockProfileRate` |
| `/debug/pprof/mutex`       | Mutex          | Requires `runtime.SetMutexProfileFraction` |
| `/debug/pprof/threadcreate`| Thread create  | Instant                       |
| `/debug/pprof/trace`       | Execution trace| `?seconds=5`; use `go tool trace` not pprof |
| `/debug/pprof/cmdline`     | Process args   | Instant                       |
| `/debug/pprof/symbol`      | PC→func        | Used by pprof internally      |

Block and mutex profiles need explicit enabling:

```go
runtime.SetBlockProfileRate(1)         // record every blocking event
runtime.SetMutexProfileFraction(1)     // record every mutex contention
```

Rate 1 = every event; higher numbers sample (1 in N).

### CPU profile sampling

CPU profiles sample at ~100Hz (10ms cadence). Each sample is the current PC across all OS threads. Long enough sampling (30+ s) → useful aggregates; short bursts (<5 s) → noise.

The kernel posts SIGPROF; the Go runtime's signal handler captures the stack via `runtime.SetCPUProfileRate`. Default rate is 100Hz; change via `runtime/pprof.SetCPUProfileRate(N)` (rarely needed).

### Heap profile sampling

Heap profiles record an allocation every `MemProfileRate` bytes (default 512KB). The recorded stack is the alloc site; pprof aggregates them.

```go
runtime.MemProfileRate = 1   // record every allocation (expensive)
runtime.MemProfileRate = 0   // disable heap profiling
```

The default (~512KB) is fine for most production use; rate 1 gives accurate small-object analysis at substantial runtime cost.

Two views from a heap profile:

- **inuse_space / inuse_objects** — what's live right now.
- **alloc_space / alloc_objects** — cumulative over the program's life.

```bash
$ go tool pprof -http=:8080 heap.out
# Switch the SAMPLE drop-down in the web UI:
#   inuse_space (default)
#   inuse_objects
#   alloc_space
#   alloc_objects
```

`inuse_*` shows current memory pressure; `alloc_*` shows GC pressure (allocation rate).

### The interactive REPL

```bash
$ go tool pprof cpu.out
(pprof) top10
Showing nodes accounting for 4.50s, 90% of 5.00s total
Dropped 25 nodes (cum <= 0.025s)
      flat  flat%   sum%        cum   cum%
     1.80s 36.00% 36.00%      1.80s 36.00%  runtime.mallocgc
     0.80s 16.00% 52.00%      0.80s 16.00%  runtime.scanobject
     0.50s 10.00% 62.00%      0.50s 10.00%  runtime.writeBarrier
     ...
```

- **flat** — time spent in this function alone.
- **cum** — time in this function and its callees.
- **flat%** — flat as fraction of total.
- **sum%** — cumulative `flat%` going down the list.

```
(pprof) top -cum         # sort by cumulative
(pprof) top10 -cum
(pprof) list HotFunc     # annotated source
(pprof) web              # graph in browser (needs graphviz: brew install graphviz)
(pprof) traces           # raw stacks
(pprof) peek HotFunc     # who calls HotFunc, who HotFunc calls
(pprof) disasm HotFunc   # native assembly with sample counts
(pprof) tags             # for profiles with tags (e.g., http handler labels)
(pprof) help
```

### The web UI

```bash
$ go tool pprof -http=:8080 cpu.out
```

Tabs:

- **Top** — sortable table (same as REPL `top`).
- **Graph** — call graph; nodes sized by `flat`, edges by call cost.
- **Flame Graph** — wide bars at the bottom show hot leaves; click to zoom.
- **Peek** — caller/callee fan-in/out for a chosen node.
- **Source** — annotated source listing.
- **Disassembly** — annotated machine code.

The flame graph reads bottom-up: leaves at top, root at bottom (Go's flamegraph orientation; Brendan Gregg's original is inverted — pprof flipped it).

### Differential profiles

```bash
$ go tool pprof -http=:8080 -base old.out new.out
```

Each cell in the report becomes `(new - old)`. Negative numbers are improvements. Used to compare "before optimization" vs. "after".

For benchmarks, prefer `benchstat` (`09-tooling/21-benchstat.md`) since it does the statistical work; for production-shaped profiles, `-base` is the right tool.

### Pprof tags (labels)

```go
import "runtime/pprof"

ctx := pprof.WithLabels(ctx, pprof.Labels("handler", "GET /users"))
pprof.Do(ctx, ctx.Value("labels").(pprof.LabelSet), func(ctx context.Context) {
    // work
})
```

Or simpler, with the `pprof.SetGoroutineLabels(ctx)` form:

```go
pprof.Do(ctx, pprof.Labels("handler", "GET /users"), func(ctx context.Context) {
    handleUsers(ctx, w, r)
})
```

In the web UI, the **Top** tab gains a "Tags" filter that lets you slice the profile by label value. Critical for multi-handler services: "how much CPU does each route consume?".

### Saving profiles for later

```bash
$ # collect once, view later
$ curl -o cpu.out http://localhost:6060/debug/pprof/profile?seconds=30
$ go tool pprof -http=:8080 cpu.out

$ # for the binary as well (improves symbolication):
$ go tool pprof -http=:8080 ./bin cpu.out
```

Passing the binary helps when the profile lacks full symbol info (e.g., stripped or PGO-changed builds).

### Tracing & pprof

`/debug/pprof/trace?seconds=5` produces a **runtime trace**, not a pprof profile. Open with `go tool trace`, not `pprof`. See `09-tooling/13-go-tool-trace.md`.

### Comparing CPU vs. trace

| Tool        | Granularity     | Use case                                                |
|-------------|-----------------|---------------------------------------------------------|
| `pprof` CPU | 10 ms samples   | "What functions burn time?"                             |
| `trace`     | Per-event       | "What's blocking goroutines? Scheduling? GC? Network?"  |

Use both: pprof tells you where time goes; trace tells you why goroutines wait.

### Profile size

- CPU profile (30s, busy service): 50–500 KB gzipped.
- Heap profile: 10–200 KB.
- Goroutine profile: ~1 KB per goroutine.
- Trace: 1–10 MB per second of capture.

### Programmatic profiling

```go
import (
    "os"
    "runtime/pprof"
)

func profileCPU(path string, fn func()) error {
    f, err := os.Create(path)
    if err != nil { return err }
    defer f.Close()
    if err := pprof.StartCPUProfile(f); err != nil { return err }
    defer pprof.StopCPUProfile()
    fn()
    return nil
}
```

For heap:

```go
runtime.GC()  // get accurate live set
f, _ := os.Create("heap.out")
defer f.Close()
pprof.Lookup("heap").WriteTo(f, 0)
```

### `pprof` and PGO

A CPU profile produced by `pprof` is *exactly* the input format for `-pgo`. The workflow:

```bash
$ curl -o default.pgo http://prod/debug/pprof/profile?seconds=60
$ go build -pgo=auto -o app ./cmd/app
```

`go build` picks up `default.pgo` if it's next to `main.go` (`-pgo=auto`'s rule).

## Standard Library Hooks

- `runtime/pprof` — programmatic profile capture.
- `net/http/pprof` — HTTP handlers.
- `runtime` — `MemProfileRate`, `SetBlockProfileRate`, `SetMutexProfileFraction`.
- `github.com/google/pprof` — the upstream pprof viewer (Go ships a vendored copy).
- `runtime/trace` — the trace counterpart (not pprof but related).

## Real-World Patterns

### 1. Profile a busy handler

```bash
$ curl -o cpu.out 'http://localhost:6060/debug/pprof/profile?seconds=30'
$ go tool pprof -http=:8080 cpu.out
```

In the web UI, jump to flame graph → identify the widest leaf at the top.

### 2. Identify a memory leak

```bash
$ # capture two heap snapshots, 10 minutes apart
$ curl -o heap1.out http://localhost:6060/debug/pprof/heap
$ sleep 600
$ curl -o heap2.out http://localhost:6060/debug/pprof/heap
$ go tool pprof -http=:8080 -base heap1.out heap2.out
```

Differential view shows what grew.

### 3. Find goroutine leaks

```bash
$ curl -o g.out 'http://localhost:6060/debug/pprof/goroutine?debug=1'
$ go tool pprof -top g.out
```

A large count from a single creation site (often `go func() {}` in a request handler) is the smoking gun.

### 4. Per-handler labels

```go
func wrap(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        labels := pprof.Labels("handler", r.URL.Path)
        pprof.Do(r.Context(), labels, func(ctx context.Context) {
            h.ServeHTTP(w, r.WithContext(ctx))
        })
    })
}
```

Then filter by tag in the web UI to attribute CPU per route.

### 5. Continuous profiling

Tools like [pyroscope](https://pyroscope.io) or [parca](https://www.parca.dev) scrape `/debug/pprof/` periodically and store profiles for trending. Lets you spot regressions hours/days after deploy.

### 6. Bench-driven profile

```bash
$ go test -bench=BenchmarkHot -cpuprofile=cpu.out -count=10 -benchtime=5s ./pkg
$ go tool pprof -http=:8080 cpu.out
```

The benchmark runs 50 s total (5 s × 10), giving enough samples for high-confidence analysis.

### 7. Save a profile, ship the binary

```bash
$ scp prod:/debug/pprof/profile cpu.out
$ scp prod:/usr/local/bin/myapp ./
$ go tool pprof -http=:8080 ./myapp cpu.out
```

The binary gives better symbol resolution than what the profile carries.

## Anti-Patterns & Gotchas

**Short CPU profiles (<5 s).** Too few samples → noise. 30 s minimum for actionable data.

**Block/mutex profiling left at rate 1 in production.** Each blocking event records a stack — high overhead. Use a fraction (e.g., `SetBlockProfileRate(1000000)` for 1 in 1M events).

**Heap profile rate 1 in production.** Records every allocation; can 10× a service's CPU cost. Leave default.

**Interpreting `flat` and `cum` interchangeably.** `flat` is in-function; `cum` is in-function + callees. A function with high `cum` but low `flat` is a parent; the children are the work.

**Forgetting `runtime.GC()` before heap snapshot.** Without it, `inuse_space` includes unreachable-but-not-yet-collected memory. Triggers can be misleading.

**Comparing profiles with `diff` instead of `pprof -base`.** Profile files are protobuf-gzipped; diffing them produces gibberish. Use `pprof -base old new`.

**Running `pprof -http=:8080` without graphviz installed.** The "Graph" tab renders empty. `brew install graphviz` (or `apt install graphviz`).

**Exposing `/debug/pprof/*` on a public interface.** Anyone can DoS your service via expensive profile collections or read sensitive command-line args. Always bind localhost or behind auth.

**Trusting CPU profiles when CGO is involved.** `pprof` may not symbolicate C code; cgo functions appear as `cgo[abi]_*`. Use `perf` for the C side.

**Profile from one short request expected to show the request's bottleneck.** Insufficient samples. Drive load with `wrk`/`hey` for the duration of the profile.

**Reading flame graphs as call graphs.** They're not. Flame graphs show aggregated stacks; siblings aren't sequential calls but separate observed stacks at different sampling moments.

**Confusing CPU profile percentages with wall-clock.** A profile's "20% in foo" means "20% of *sampled* CPU time"; if your service is mostly idle, that's 20% of a tiny number. Always note the absolute total at the top of `top`.

## Performance Notes

- CPU profile overhead: ~3% during sampling.
- Heap profile overhead: <1% at default rate.
- Block/mutex profile at rate 1: 5–30% slowdown depending on contention.
- Goroutine profile: O(N) snapshot; N = goroutine count. Fast for thousands; slow for millions.
- pprof web UI: <1 s startup for typical profiles.
- Symbolication: <100 ms with binary present; can take seconds without.

## How Big Companies Use It

- **Google** uses pprof throughout internal services; some teams ingest profiles into BigQuery for trending: https://github.com/google/pprof.
- **Uber** open-sourced [pyroscope](https://github.com/grafana/pyroscope) (now under Grafana) for continuous profiling of their Go fleet: https://pyroscope.io.
- **Cloudflare** uses pprof + custom scrapers to profile their Workers Go runtime: https://blog.cloudflare.com/profiling-go.
- **Datadog** scrapes `/debug/pprof/*` and ships [their continuous profiler](https://docs.datadoghq.com/profiler).
- **CockroachDB** publishes their pprof workflows in cluster diagnostics: https://www.cockroachlabs.com/docs/stable/cluster-setup-troubleshooting.html.
- **The Go team** uses pprof on the toolchain itself (`go build`); profile-guided optimization (PGO) is built on this pipeline: https://go.dev/blog/pgo.
- **Discord** integrates pprof with their internal devstack — `go tool pprof -http` is the first line of investigation for any service latency regression.

## Source Code References

Pinned to `go1.26`.

- `runtime/pprof`: [`src/runtime/pprof`](https://github.com/golang/go/tree/release-branch.go1.26/src/runtime/pprof).
- `net/http/pprof`: [`src/net/http/pprof`](https://github.com/golang/go/tree/release-branch.go1.26/src/net/http/pprof).
- Runtime CPU profiler: [`src/runtime/cpuprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/cpuprof.go).
- Heap profile: [`src/runtime/mprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mprof.go).
- pprof viewer (vendored): [`src/cmd/vendor/github.com/google/pprof`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/vendor/github.com/google/pprof).
- Upstream pprof: [`github.com/google/pprof`](https://github.com/google/pprof).

(BSD-3-Clause © The Go Authors; Apache-2.0 © Google for upstream pprof.)

## Further Reading

- "Profiling Go programs" (Russ Cox, Go blog): https://go.dev/blog/pprof.
- "pprof documentation": https://github.com/google/pprof/blob/main/doc/README.md.
- "Continuous Profiling in Production" (Felix Geisendörfer): https://www.youtube.com/watch?v=PaY7Ru5q5oM.
- "How to use Go's profiler" (Datadog): https://www.datadoghq.com/blog/go-profiling-best-practices.
- "Memory profiling in Go" (Maxim Vladimirsky): https://medium.com/@vCabbage/go-best-practices-memory-profiling.
- Brendan Gregg, "Flame graphs": https://www.brendangregg.com/flamegraphs.html.

## Exercises / Self-Check

1. Add `net/http/pprof` to a small service. Capture 30 s of CPU profile during load and identify the top function by `cum` time.
2. Capture two heap snapshots 5 minutes apart from an idle service. Use `-base` to compare. Why might `inuse_space` change even when no requests are served?
3. Enable mutex profiling at rate 1000 and find a contention hotspot.
4. Wrap an HTTP handler in `pprof.Do` with a label set. Confirm the label appears in the web UI's Tags filter.
5. Take a CPU profile and feed it back as `-pgo=auto`. Rebuild and measure the runtime delta.
