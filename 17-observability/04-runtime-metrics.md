# Runtime Metrics — `runtime/metrics`

## TL;DR

`runtime/metrics` (Go 1.16+, expanded heavily through 1.26) is the modern, stable, schema-versioned interface for asking the Go runtime "what are you doing?" It exposes ~50 named metrics across GC, scheduler, memory, mutex, and CPU subsystems — every one with a documented name, unit, kind (counter, gauge, histogram), and stability guarantee. It superseded `runtime.MemStats` (still works, but frozen and incomplete) and `debug.GCStats` (still works, still narrow). The killer feature: **distributions are histograms with high-resolution exponential buckets**, so you get real GC-pause percentiles rather than the legacy "last 256 pauses in a circular buffer." The single biggest gotcha: **`metrics.Read` is not free** — each call walks ~50 metric descriptions and copies values; calling it from a tight scrape loop without batching can become measurable. Pre-allocate a `[]Sample` slice once, reuse on every read. Go 1.23 added `/sched/pauses/total/{gc,other}:seconds` (separating GC pauses from non-GC STW). Go 1.25 added `/gc/scan/heap:bytes` and `/sync/mutex/wait/total:seconds` (per-mutex wait — the mutex contention metric the runtime never used to expose). Go 1.26 finalises greenteagc-related metrics under `/gc/greentea/*`.

## Mental Model

```
   Your process
        │
        │ runtime/metrics.Read([]Sample) ─► fills in *all* values
        ▼
   ┌──────────────────────────────┐
   │  Runtime internal counters    │
   │  ├─ GC: pauses, scanned, ...   │
   │  ├─ scheduler: goroutines, ...  │
   │  ├─ memory: classes, alloc, ... │
   │  ├─ mutex: wait, ...            │
   │  └─ cpu: total, scavenge, ...   │
   └──────────────────────────────┘
        │
        ▼
   Expose via Prometheus collector,
   OTel meter, or whatever your stack uses.
```

Two facts:

1. **Metrics are described by string names with units after a colon**: `/gc/cycles/total:gc-cycles`, `/memory/classes/heap/free:bytes`. The unit suffix matters for downstream conversion.
2. **The API is by name** — you ask "what metrics exist?" via `metrics.All()`, then sample by name. Forward-compatible: when Go adds new metrics, you pick them up automatically.

## Basic Usage

```go
import "runtime/metrics"

// One-time: list everything available
for _, d := range metrics.All() {
    fmt.Printf("%-50s  %-12s  cumulative=%v\n", d.Name, d.Kind, d.Cumulative)
}
```

```go
// Repeated sampling
desc := []metrics.Sample{
    {Name: "/gc/cycles/total:gc-cycles"},
    {Name: "/sched/goroutines:goroutines"},
    {Name: "/memory/classes/heap/free:bytes"},
    {Name: "/gc/pauses:seconds"},          // distribution
}

metrics.Read(desc)

for _, s := range desc {
    switch s.Value.Kind() {
    case metrics.KindUint64:
        fmt.Println(s.Name, s.Value.Uint64())
    case metrics.KindFloat64:
        fmt.Println(s.Name, s.Value.Float64())
    case metrics.KindFloat64Histogram:
        h := s.Value.Float64Histogram()
        fmt.Println(s.Name, "p50=", quantile(h, 0.5), "p99=", quantile(h, 0.99))
    case metrics.KindBad:
        fmt.Println(s.Name, "unsupported on this Go version")
    }
}
```

The `KindBad` case is critical: `runtime/metrics` is **forward-compatible across versions** by silently returning `KindBad` for unknown names. Your code keeps working when you upgrade.

## The 30 Metrics That Matter (Go 1.26)

### GC

| Metric | Kind | Use |
|--------|------|-----|
| `/gc/cycles/total:gc-cycles` | counter (uint64) | Total GC cycles run |
| `/gc/cycles/automatic:gc-cycles` | counter | GC cycles triggered by heap growth (not by `runtime.GC()`) |
| `/gc/cycles/forced:gc-cycles` | counter | Explicit `runtime.GC()` calls |
| `/gc/heap/allocs:bytes` | counter | Total bytes allocated (sum of all live + dead) |
| `/gc/heap/allocs:objects` | counter | Total object count allocated |
| `/gc/heap/frees:bytes` | counter | Total bytes freed |
| `/gc/heap/frees:objects` | counter | Total object count freed |
| `/gc/heap/goal:bytes` | gauge | Next-cycle heap-size target |
| `/gc/heap/live:bytes` | gauge | Live heap at last GC |
| `/gc/heap/objects:objects` | gauge | Live object count |
| `/gc/heap/tiny/allocs:objects` | counter | Tiny-allocator (≤16B) bytes |
| `/gc/pauses:seconds` | histogram | STW pause durations |
| `/gc/scan/heap:bytes` | counter (1.25+) | Bytes scanned per GC for liveness |
| `/gc/scan/stack:bytes` | counter (1.25+) | Stack bytes scanned per GC |
| `/gc/scan/total:bytes` | counter (1.25+) | Total scan work |
| `/gc/stack/starting-size:bytes` | gauge | Initial goroutine stack size |
| `/gc/greentea/...` | various (1.26+) | Green Tea GC-specific |

### Scheduler

| Metric | Use |
|--------|-----|
| `/sched/goroutines:goroutines` | Current live goroutines |
| `/sched/latencies:seconds` | Histogram of time goroutine was runnable but not running |
| `/sched/pauses/stopping/gc:seconds` | Time to stop the world for GC (1.23+) |
| `/sched/pauses/stopping/other:seconds` | Time to stop the world for non-GC reasons (1.23+) |
| `/sched/pauses/total/gc:seconds` | Total GC-induced STW |
| `/sched/pauses/total/other:seconds` | Total non-GC STW |

### Memory classes (decomposes the heap)

| Metric | Use |
|--------|-----|
| `/memory/classes/total:bytes` | Total memory under runtime control |
| `/memory/classes/heap/free:bytes` | Heap unused by user code |
| `/memory/classes/heap/objects:bytes` | Heap occupied by live + dead user objects |
| `/memory/classes/heap/released:bytes` | Released back to OS (madvise DONTNEED) |
| `/memory/classes/heap/stacks:bytes` | Goroutine stacks (allocated from heap) |
| `/memory/classes/heap/unused:bytes` | Slab/span overhead |
| `/memory/classes/metadata/mcache/*:bytes` | Per-P metadata |
| `/memory/classes/os-stacks:bytes` | OS thread stacks |
| `/memory/classes/profiling/*:bytes` | Profiling buffers |
| `/memory/classes/other:bytes` | Everything else |

### Mutex (1.25+ — the big addition)

| Metric | Use |
|--------|-----|
| `/sync/mutex/wait/total:seconds` | Cumulative time goroutines spent blocked on `sync.Mutex` |

Until 1.25, you could see *contention* via blocking profiles but not *aggregate wait time*. This metric is gold for capacity planning.

### CPU

| Metric | Use |
|--------|-----|
| `/cpu/classes/total:cpu-seconds` | Total CPU consumed by the process |
| `/cpu/classes/gc/total:cpu-seconds` | CPU spent in GC |
| `/cpu/classes/gc/mark/assist:cpu-seconds` | GC assist time (sign of insufficient GC capacity) |
| `/cpu/classes/gc/mark/dedicated:cpu-seconds` | Dedicated GC marker goroutines |
| `/cpu/classes/gc/mark/idle:cpu-seconds` | Idle marker time (cheap, soaked-up cycles) |
| `/cpu/classes/gc/pause:cpu-seconds` | STW cycles |
| `/cpu/classes/scavenge/*:cpu-seconds` | Background scavenge to OS |
| `/cpu/classes/idle:cpu-seconds` | Wall-clock idle |
| `/cpu/classes/user:cpu-seconds` | All non-GC, non-idle |

The `gc/mark/assist` series is the canonical "is allocation pressure overwhelming GC?" signal. High assist ratios mean user goroutines are being conscripted to help GC — capacity for actual work is being eaten.

## Histogram Helpers

```go
func quantile(h *metrics.Float64Histogram, q float64) float64 {
    // Cumulative count
    total := uint64(0)
    for _, c := range h.Counts { total += c }
    target := uint64(float64(total) * q)
    cum := uint64(0)
    for i, c := range h.Counts {
        cum += c
        if cum >= target {
            // Linear interpolation between bucket bounds
            lo, hi := h.Buckets[i], h.Buckets[i+1]
            return (lo + hi) / 2
        }
    }
    return h.Buckets[len(h.Buckets)-1]
}
```

`Float64Histogram.Buckets` has length `len(Counts)+1` — bucket `i` covers `[Buckets[i], Buckets[i+1])`. `Buckets[0]` may be `-Inf`, `Buckets[len-1]` may be `+Inf`.

Histograms are **cumulative**: each `Read` overwrites with absolute totals since process start. For per-interval deltas, subtract the previous snapshot.

## Mapping to Prometheus

The Prometheus `client_golang` library has a built-in collector for runtime metrics:

```go
import "github.com/prometheus/client_golang/prometheus/collectors"

prometheus.MustRegister(collectors.NewGoCollector(
    collectors.WithGoCollections(collectors.GoRuntimeMetricsCollection),
))
```

This exposes every `runtime/metrics` series as Prometheus metrics under the `go_` prefix. The names are mapped automatically: `/gc/heap/allocs:bytes` becomes `go_gc_heap_allocs_bytes_total`. Histograms (`/gc/pauses:seconds`) become Prometheus native histograms.

For OTel:

```go
import "go.opentelemetry.io/contrib/instrumentation/runtime"

runtime.Start(runtime.WithMinimumReadMemStatsInterval(time.Second))
```

The OTel runtime instrumentation lazily reads `runtime/metrics` and emits via the configured `MeterProvider`.

## When to Read

Reading is cheap *enough* — ~10–50 µs depending on how many distributions you sample — but not free:

- **Prometheus scrape interval** (15s default): read once per scrape, inside the collector.
- **OTel periodic exporter**: read once per export interval (typically 30–60s).
- **Custom logging** (e.g., for SLO tracking): cadence to your needs; >1Hz is usually overkill.

Avoid:

- Reading in hot paths.
- Reading more often than your scrape interval.
- Reading just one metric — the API is "fill a slice" so the cost is mostly fixed.

## Custom Sample Slice — The Pattern

```go
type runtimeMetrics struct {
    samples []metrics.Sample
}

func newRuntimeMetrics(names []string) *runtimeMetrics {
    s := make([]metrics.Sample, len(names))
    for i, n := range names { s[i].Name = n }
    return &runtimeMetrics{samples: s}
}

func (rm *runtimeMetrics) Update() {
    metrics.Read(rm.samples)  // fills in-place
}

func (rm *runtimeMetrics) Goroutines() uint64 {
    return rm.samples[0].Value.Uint64()  // assuming index 0 is /sched/goroutines:goroutines
}
```

Allocate once at startup; reuse the slice on every read. Avoids per-call allocation.

## Useful Derived Quantities

```go
// GC CPU fraction — fraction of total CPU spent in GC since start
gc := rm.samples[i_gc_total].Value.Float64()
tot := rm.samples[i_cpu_total].Value.Float64()
gcFrac := gc / tot   // expect <0.05 (5%) in healthy services

// Heap fragmentation
free := rm.samples[i_heap_free].Value.Uint64()
released := rm.samples[i_heap_released].Value.Uint64()
unreleased := free - released   // memory the runtime is holding but not using

// Live heap growth rate
// (Take periodic snapshots of /gc/heap/live:bytes)
```

The GC CPU fraction is one of the strongest signals for "do I need to increase `GOGC` or `GOMEMLIMIT`?"

## Anti-Patterns & Gotchas

**Using `runtime.MemStats` for new code.** Frozen, incomplete, induces a STW on `ReadMemStats`. Use `runtime/metrics`.

**Reading on a hot path.** Pin the call to your scrape or export interval.

**Allocating `[]Sample` per call.** Allocate once.

**Computing percentiles in process from histograms then exposing percentiles.** Lose aggregation across replicas. Export the *histogram* (Prometheus native histograms preserve buckets) and quantile in PromQL.

**Mixing pre-1.21 names** in 1.22+ code. Names are stable but new ones are added. Check `metrics.Description.Cumulative` and ignore `KindBad`.

**Treating `/gc/heap/live:bytes` as "current heap."** It's the live heap *at last GC*. Allocations after that aren't included until the next cycle.

**Assuming `/memory/classes/total:bytes` equals RSS.** It's runtime-tracked memory. RSS includes things the runtime doesn't see (mmap'd files, CGo).

**Forgetting that pause histograms include all pauses.** GC pauses + non-GC STW. Use the 1.23+ `/sched/pauses/total/gc:seconds` vs `/other:seconds` split.

**Sampling once at startup.** Most metrics are cumulative; you need deltas for rates.

**Ignoring `/cpu/classes/gc/mark/assist:cpu-seconds`.** High values say GC is keeping up *only* by stealing CPU from user goroutines. A key health signal.

**Exposing every histogram as Prometheus classic histogram with default buckets.** The default buckets are wrong for GC pauses (sub-ms) and wrong for scheduler latencies (sub-µs). Native histograms or custom buckets.

**Reading runtime metrics in a CGO-heavy binary.** They cover the Go runtime; the C side (malloc, OpenSSL, etc.) is invisible. Combine with `/proc/<pid>/status` or `procfs` for full picture.

## Performance Notes

- **`metrics.All()`**: ~50 µs (returns ~50 descriptions). Call once at startup.
- **`metrics.Read([]Sample)` with all ~50 metrics**: ~50–100 µs. With ~10 metrics: ~10–20 µs. Distributions dominate.
- **Allocation**: 0 if `[]Sample` is reused.
- **Histogram buckets**: typically 50–60 buckets per distribution; bucket bounds are precomputed inside the runtime.
- **STW cost**: zero. `runtime/metrics` does not pause the world. (`runtime.ReadMemStats` *did* until Go 1.16 and still locks for a moment in some versions.)

## How Big Companies Use It

- **Google** uses `runtime/metrics` internally to feed GC dashboards across their Go fleet; the API was designed by the runtime team to address operational gaps `MemStats` couldn't fill.
- **Cloudflare** uses `runtime/metrics` + Prometheus + Grafana for fleet-wide Go runtime monitoring; the `go_gc_*` series are key tuning signals during incidents.
- **Datadog**'s Go agent reads `runtime/metrics` for the `runtime.go.gc.*` family of metrics in their integration.
- **Grafana Labs** internal dashboards for Mimir/Tempo include GC pause p99, goroutine count, and mark assist time — all from `runtime/metrics`.
- **Discord** publicly attributed several Go-runtime-related performance fixes to insights gleaned from these metrics (notably scheduler latency vs goroutine count).
- **Twitch** uses runtime metrics for `GOMEMLIMIT` tuning across their Go services.
- **DoorDash** uses GC CPU fraction alerting (from `/cpu/classes/gc/*`) as a deploy-gate.

## Source Code References

- `runtime/metrics`: https://github.com/golang/go/tree/master/src/runtime/metrics.
- Documentation page (every metric with its semantics): https://pkg.go.dev/runtime/metrics.
- Implementation: https://github.com/golang/go/blob/master/src/runtime/metrics.go.
- Prometheus collector for runtime metrics: https://github.com/prometheus/client_golang/blob/main/prometheus/collectors/go_collector_latest.go.
- OTel runtime instrumentation: https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/runtime.
- Original proposal: https://github.com/golang/go/issues/37112.

## Further Reading

- "A Guide to the Go Garbage Collector": https://tip.golang.org/doc/gc-guide — uses `runtime/metrics` extensively for diagnosis.
- Michael Knyszek, "Performance and the Go Runtime" (GopherCon talks).
- "Go 1.21 GC improvements" (release notes and blog).
- Felix Geisendörfer, "Go pprof and profiling" series.
- Bryan C. Mills, "Understanding the Go scheduler" — talks.
- "GOMEMLIMIT in 1.19" (Michael Knyszek): https://go.dev/doc/gc-guide#Memory_limit.

## Exercises / Self-Check

1. List every metric on your Go version. Identify the ones unavailable from the deprecated `runtime.MemStats` API.
2. Build a Prometheus collector that exposes `/cpu/classes/gc/mark/assist:cpu-seconds` as a rate. Alert when >5% of CPU is going to GC assist.
3. Compute the p99 GC pause from `/gc/pauses:seconds`. Compare to `debug.GCStats` legacy values. Confirm the histogram-based answer is more accurate.
4. Track `/sync/mutex/wait/total:seconds` over a 10-minute load test. Correlate the slope with your throughput to estimate contention cost.
5. Subtract two snapshots of `/gc/heap/allocs:bytes` to compute allocation rate. Confirm against `go tool pprof -alloc_space`.
6. Wire OTel runtime instrumentation. Visualise GC CPU fraction in your tracing backend.
7. Read every histogram in a single sample. Benchmark `metrics.Read` with all-metrics vs just 5 metrics. Quantify the cost difference.
8. Trigger sustained allocation pressure (e.g., 1 GB/s of short-lived `[]byte`). Watch `/cpu/classes/gc/*` series react. Tune `GOGC` and `GOMEMLIMIT`; observe.
