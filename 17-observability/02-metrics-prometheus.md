# Metrics — Prometheus `client_golang`

## TL;DR

Prometheus is the de-facto open-source metrics ecosystem; its Go client (`github.com/prometheus/client_golang`) is the canonical instrumentation library. Four metric types you must internalise: **Counter** (monotonic; rate per second; HTTP requests, errors), **Gauge** (snap-shot, can go up or down; queue depth, memory in use), **Histogram** (pre-bucketed observation counts plus sum and count; request latency), and **Summary** (client-side quantiles; legacy — avoid for new code). The library scrapes `/metrics` HTTP endpoint in Prometheus text or OpenMetrics format. The single biggest gotcha across every team I've seen instrument with this library: **label cardinality**. A label with millions of unique values (user ID, request path with IDs, full URL) creates millions of time series, blows up your Prometheus server, and explains why your bill is what it is. The fix is a small, finite label set (`method`, `route`, `status_class`) plus exemplars (which point to high-cardinality trace IDs externally). Go 1.26 doesn't change Prometheus itself, but the OTel metrics SDK is now mature enough that many teams deploy OTel-only and bridge to Prometheus at the Collector — covered in the next page.

## Mental Model

```
                        ┌──────────────────────────────────┐
   Your Go app          │  prometheus.Registry              │
                        │   ├─ Counter:    http_requests_total
   Counters ─inc()────►│   ├─ Gauge:      goroutines
   Gauges   ─set()────►│   ├─ Histogram:  http_request_seconds
                        │   └─ Summary:    legacy             │
                        └──────────────────────────────────┘
                                       │
                                       │ HTTP GET /metrics
                                       ▼
                          ┌────────────────────────┐
                          │  promhttp.Handler()     │  scrape endpoint
                          └────────────────────────┘
                                       ▲
                                       │ scrape (15s default)
                                       │
                          ┌────────────────────────┐
                          │   Prometheus server     │
                          │   - TSDB, alerting     │
                          └────────────────────────┘
```

Two facts to keep in mind:

1. **Metrics are pull, not push.** Prometheus calls *you* every 15s by default. Your job is to keep a cheap, current snapshot ready.
2. **Each distinct combination of label values is a separate time series.** Cardinality is a budget; pretend you have a million series total and never exceed it.

## Install

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
```

```bash
go get github.com/prometheus/client_golang@latest
```

## Counter

A monotonically increasing number. Always derive a rate in PromQL with `rate()` or `increase()`.

```go
var httpRequests = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests.",
    },
    []string{"method", "route", "status_class"},
)

func init() {
    prometheus.MustRegister(httpRequests)
}

func handler(w http.ResponseWriter, r *http.Request) {
    statusClass := "2xx"
    // ... do work ...
    httpRequests.WithLabelValues(r.Method, "/users/{id}", statusClass).Inc()
}
```

Naming convention:
- Snake_case, ASCII.
- Always `_total` suffix for counters (Prometheus best practice; some tools assume it).
- Unit-bearing names: `_bytes`, `_seconds`, `_total`. Never milliseconds.

## Gauge

A point-in-time value that can go up or down.

```go
var queueDepth = prometheus.NewGauge(prometheus.GaugeOpts{
    Name: "task_queue_depth",
    Help: "Number of tasks pending.",
})

queueDepth.Set(float64(q.Len()))
queueDepth.Inc()
queueDepth.Dec()
queueDepth.Add(5)
```

Common gauges:
- `process_resident_memory_bytes`
- `goroutines_count`
- `open_connections`
- `cache_size_bytes`

## Histogram

Pre-bucketed observation counts. Critical for latency and size distributions.

```go
var httpDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "http_request_duration_seconds",
        Help: "HTTP request latency.",
        Buckets: prometheus.ExponentialBuckets(0.001, 2, 12), // 1ms → 4s
    },
    []string{"route", "status_class"},
)

start := time.Now()
// ... handle ...
httpDuration.WithLabelValues("/users/{id}", "2xx").Observe(time.Since(start).Seconds())
```

Buckets: 8–12 is typical. Each bucket is a *separate time series*; 20 buckets × 50 routes × 5 status_classes = 5000 series.

**Choosing buckets:**
- `prometheus.DefBuckets` (`.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10`) is sensible for HTTP latency.
- `ExponentialBuckets(start, factor, count)` for power-of-two growth.
- `LinearBuckets(start, width, count)` for uniform spacing.
- Custom: cover your real distribution; over-bucketing the boring middle wastes series.

**Quantiles from histograms:** PromQL `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` — the p99 is *server-side approximated* from buckets. Better than client-side summaries because: (a) aggregate-able across instances, (b) constant memory, (c) cardinality-friendly.

### Native histograms (Prometheus 2.40+, client_golang 1.13+)

```go
prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "http_request_duration_seconds",
        Help: "...",
        NativeHistogramBucketFactor: 1.1,   // ~10% resolution
        NativeHistogramMaxBucketNumber: 100,
    },
    []string{"route"},
)
```

Native histograms encode buckets as exponential schema → tens of buckets per series, ~100× less storage than classic bucketed histograms with similar resolution. Production-recommended for new instrumentation since 2024.

## Summary

```go
var latency = prometheus.NewSummary(prometheus.SummaryOpts{
    Name: "request_latency_seconds",
    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
})
latency.Observe(d.Seconds())
```

Avoid for new code. Quantiles are computed *per instance*, can't be aggregated across replicas, and use bounded memory but at significant CPU. Histogram + `histogram_quantile()` is almost always better.

## The Scrape Endpoint

```go
http.Handle("/metrics", promhttp.Handler())
log.Fatal(http.ListenAndServe(":9090", nil))
```

In production:

```go
mux := http.NewServeMux()
mux.Handle("/metrics", promhttp.HandlerFor(
    prometheus.DefaultGatherer,
    promhttp.HandlerOpts{
        EnableOpenMetrics: true,                    // exemplars need this
        MaxRequestsInFlight: 5,                     // protect scrape endpoint
        Timeout: 10 * time.Second,                  // bound scrape duration
    },
))
go http.ListenAndServe(":9090", mux)
```

Note: **separate port for `/metrics`** (e.g., 9090) keeps it off the public path. Most production K8s setups expose 9090 only to the Prometheus pod via NetworkPolicy.

## Default Collectors

```go
import "github.com/prometheus/client_golang/prometheus/collectors"

prometheus.MustRegister(collectors.NewGoCollector(
    collectors.WithGoCollections(collectors.GoRuntimeMetricsCollection),
))
prometheus.MustRegister(collectors.NewProcessCollector(
    collectors.ProcessCollectorOpts{},
))
prometheus.MustRegister(collectors.NewBuildInfoCollector())
```

The `GoRuntimeMetricsCollection` exposes the full `runtime/metrics` series (covered in `04-runtime-metrics.md`). Defaults to a smaller subset for backward compatibility.

## Exemplars

Link a histogram observation to a trace ID:

```go
httpDuration.WithLabelValues("/users/{id}", "2xx").(prometheus.ExemplarObserver).
    ObserveWithExemplar(d.Seconds(), prometheus.Labels{
        "trace_id": span.SpanContext().TraceID().String(),
    })
```

With `EnableOpenMetrics: true` on the handler, Prometheus stores exemplars per bucket. Grafana renders them as dots on the latency panel; clicking jumps to the matching trace. This is the bridge between metrics and traces — extremely high-leverage for SLO debugging.

## HTTP Middleware (the canonical pattern)

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

mux := http.NewServeMux()
mux.Handle("/api/users", promhttp.InstrumentHandlerCounter(
    httpRequestsByCode,
    promhttp.InstrumentHandlerDuration(
        httpDuration,
        userHandler,
    ),
))
```

For chi/echo/gin frameworks, use:
- `chi`: `chiprometheus.NewMiddleware`
- `echo`: `echoprometheus.NewMiddleware`
- `gin`: `gin-contrib/prometheus`
- Or write your own — it's ~30 lines.

Critical: use **route templates** (`/users/{id}`), not raw paths (`/users/123`). The raw path turns user IDs into label values and explodes cardinality.

## Cardinality — The Most Important Rule

```go
// BAD — millions of series
httpRequests.WithLabelValues(r.Method, r.URL.Path, ...).Inc()

// GOOD — bounded series
route := chi.RouteContext(r.Context()).RoutePattern() // "/users/{id}"
httpRequests.WithLabelValues(r.Method, route, statusClass(code)).Inc()
```

Rules of thumb:

- **No user IDs, no UUIDs, no SHA hashes** in labels.
- **Status as class** (`"2xx"`, `"4xx"`, `"5xx"`) not raw `"200"`.
- **Routes as templates** not as concrete URLs.
- **Country, region** — fine. **City** — usually too many.
- **Customer/tenant** — depends. If you have 50, fine. 50,000 — no.
- **HTTP user-agent** — extremely high cardinality; never label by it.

Audit periodically: `count by (__name__) ({__name__=~".+"})` in Prometheus shows your top-N series.

## Custom Collectors

Implement `prometheus.Collector` when you need to emit metrics from an existing data source (database table, file, in-memory map) without proxying through a Gauge:

```go
type cacheCollector struct {
    cache *lru.Cache
    desc  *prometheus.Desc
}

func (c *cacheCollector) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.desc
}
func (c *cacheCollector) Collect(ch chan<- prometheus.Metric) {
    ch <- prometheus.MustNewConstMetric(
        c.desc, prometheus.GaugeValue, float64(c.cache.Len()),
    )
}

prometheus.MustRegister(&cacheCollector{
    cache: c,
    desc: prometheus.NewDesc("cache_entries", "Cache entry count", nil, nil),
})
```

The `Collect` method runs at scrape time. Useful for cheap "ask the data source for its current value at scrape time" — avoids you running a goroutine that periodically updates a Gauge.

## Push Gateway (for batch jobs)

Cron jobs, finite-lifetime processes — Prometheus can't scrape what isn't running. Push to a `pushgateway`:

```go
import "github.com/prometheus/client_golang/prometheus/push"

err := push.New("http://pushgw:9091", "nightly_backup").
    Collector(backupBytes).
    Push()
```

Only for **batch-style instance-less metrics**. Not a general "push instead of pull" facility. Long-running services should scrape.

## Anti-Patterns & Gotchas

**Per-request `prometheus.NewCounter`.** That creates and registers a new metric every call → memory leak + Prometheus errors on duplicate registration. Create once at init, reuse forever.

**`WithLabelValues(r.URL.Path)`.** As above — cardinality killer.

**Logging `_seconds_milliseconds`** (mixed units in name + ms in observation). Always observe in *seconds* and put it in the name.

**Summary instead of histogram for SLOs.** Quantiles from per-instance summaries can't be aggregated meaningfully. Histogram + `histogram_quantile`.

**Buckets that don't cover your p99.** If p99 is 800ms and your top bucket is 250ms, p99 is reported as `+Inf`. Always include buckets up to 10× your worst-case expected latency.

**Forgetting `_total`** suffix on counters. PromQL recording rules and many UIs detect counters by this suffix.

**Two metrics with the same name but different label sets.** Re-registration panics. Use one consistent label schema.

**Holding labels with secrets.** `user_email` as a label exposes PII in `/metrics`.

**Resetting counters.** `Counter.Reset()` exists but breaks `rate()` calculations. Don't.

**Histogram observations in seconds-as-int.** `d.Milliseconds()` is `int64`; convert to `float64(d) / float64(time.Second)` or `d.Seconds()` — never integer truncate.

**Exposing `/metrics` publicly.** Internal data, attack surface, secrets-via-labels-via-bugs. NetworkPolicy or basic auth.

**Counting "active users."** That's a gauge — not a counter. Counters only go up.

**`prometheus.MustRegister(c)` repeated on hot path.** It panics on duplicate. Use `sync.Once` if dynamically registering.

**Mixing Prometheus and OTel metrics without bridge.** Pick one canonical source per series; otherwise you double-count. OTel Collector's `prometheusexporter` can serve both.

## Performance Notes

- **Counter.Inc()**: ~5–10 ns (atomic add).
- **Counter.WithLabelValues(...).Inc()**: ~50–100 ns (hash + atomic).
- **Histogram.Observe**: ~30–80 ns.
- **Scrape latency** for a typical app (~5k series): 1–20 ms; for 100k series: 50–300 ms.
- **Memory** per series: ~2 KB (varies by collector and Prometheus settings).
- **Cardinality budget** for an instance: 100k series is comfortable; 1M is painful; 10M will crash Prometheus.

Avoid `WithLabelValues` in extreme hot paths (>1M ops/sec). Cache the resolved metric:

```go
m := httpRequests.WithLabelValues("GET", "/x", "2xx")  // resolve once
for i := 0; i < 1e6; i++ { m.Inc() }                    // hot loop
```

## How Big Companies Use It

- **SoundCloud** (the original creator of Prometheus and `client_golang`) instrumented their Go fleet with this exact library; the conventions in their codebase shaped the public best-practice guide.
- **CoreOS/Red Hat OpenShift** runs Prometheus + Thanos in every cluster; instrumentation is `client_golang` end-to-end.
- **Grafana Labs** built **Mimir** (Prometheus-compatible TSDB) and uses `client_golang` for all internal Go services; many internal patterns are upstreamed.
- **Uber's M3** TSDB is Prometheus-compatible; Uber's Go services use `client_golang`.
- **Cloudflare** uses `client_golang` across their edge with Thanos for long-term storage.
- **Shopify** uses Datadog → Prometheus migration patterns documented publicly; instrumentation library is `client_golang`.
- **GitHub** runs Prometheus for their internal Go services with strict cardinality budgets enforced via CI checks.
- **Kubernetes itself** uses `client_golang` — every `kube-*` binary exposes `/metrics`.

## Source Code References

- `client_golang` repo: https://github.com/prometheus/client_golang.
- `prometheus` package (core API): https://github.com/prometheus/client_golang/tree/main/prometheus.
- `promhttp` package: https://github.com/prometheus/client_golang/tree/main/prometheus/promhttp.
- `collectors` package: https://github.com/prometheus/client_golang/tree/main/prometheus/collectors.
- Native histogram design doc: https://prometheus.io/docs/concepts/metric_types/#histogram.
- Prometheus server: https://github.com/prometheus/prometheus.

## Further Reading

- "Best practices: Metric and label naming": https://prometheus.io/docs/practices/naming/.
- "Histograms and summaries": https://prometheus.io/docs/practices/histograms/.
- Björn Rabenstein, "How to use Prometheus" talks (PromCon).
- Brian Brazil, *Prometheus: Up & Running* (O'Reilly).
- "USE method" (Brendan Gregg) and "RED method" (Tom Wilkie) — guides for which metrics to instrument.
- "SRE Workbook ch. 4 — SLO Engineering" — what to actually alert on.
- "Histograms with Prometheus: A Tale of Woe" — cautionary tales about bucket selection.

## Exercises / Self-Check

1. Instrument an HTTP handler with `Counter` (requests by method/route/status), `Histogram` (duration), and `Gauge` (in-flight requests). Scrape with `curl localhost:9090/metrics`.
2. Identify three labels in your app with high cardinality. Replace with bounded equivalents (`user_id` → `user_tier`, `path` → `route_pattern`).
3. Switch a classic histogram to a native histogram. Verify in Prometheus that the resolution is preserved and storage is lower.
4. Add OTel exemplars: link p99 latency observations to trace IDs. Confirm Grafana renders exemplar dots.
5. Write a custom `Collector` that exposes the current size of an in-memory cache at scrape time (no goroutine).
6. Build a Pushgateway flow for a nightly batch job. Confirm metrics expire after the next push of the same group.
7. Profile a hot loop with `WithLabelValues` resolved per-iteration vs hoisted once. Confirm the hoisted version saves O(N) hash lookups.
8. Set up an alert: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1`. Trigger it with a deliberately slow handler.
