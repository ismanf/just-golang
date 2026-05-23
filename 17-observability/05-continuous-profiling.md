# Continuous Profiling — Parca, Pyroscope, Polar Signals

## TL;DR

`net/http/pprof` ships with Go and lets you grab a profile *on demand*. Continuous profiling extends this idea: **scrape every running process every N seconds, store the deltas, give engineers a flame-graph view across time and across the fleet**. The three serious open-source options are **Parca** (Polar Signals), **Pyroscope** (now part of Grafana), and **Polar Signals Cloud** (commercial Parca). All three speak the standard Go pprof protocol (just hit `/debug/pprof/profile?seconds=10` periodically). The killer feature isn't capturing profiles — it's **pprof labels**: attach `route=/users/{id}`, `tenant=acme`, `feature_flag=v2` to the goroutines doing work, then filter your flame graphs by these labels in the UI. The single biggest gotcha: **CPU profile cost is non-zero** (~1–3% overhead at default 100Hz sampling, but reaches 5–10% at higher rates), and **heap profiles use real memory** (~tens of MB per profile of a busy process). Default cadence: 10-second CPU profile every 60 seconds; heap snapshot every 5 minutes. Go 1.26 ships eBPF-based delta-encoded profile uploads in the contrib OTel profiler (still experimental), and the `runtime/pprof` package gained `Profile.WriteToWithOptions` for output compaction.

## Mental Model

```
   Service instance
   ───────────────
       │
       │  /debug/pprof/profile?seconds=10  (CPU, 100 Hz sampled)
       │  /debug/pprof/heap                (current heap allocations)
       │  /debug/pprof/goroutine           (current goroutine stacks)
       │  /debug/pprof/allocs              (total allocations since start)
       │  /debug/pprof/mutex               (contention)
       │  /debug/pprof/block               (blocking events)
       │  /debug/pprof/threadcreate        (OS threads)
       ▼
   ┌─────────────────────────────────────────┐
   │  Continuous profiler (Parca / Pyroscope) │
   │  - scrapes every 10–60s                  │
   │  - delta-encodes against previous sample │
   │  - stores compressed pprof per instance  │
   └─────────────────────────────────────────┘
                  │
                  ▼  query: "flame graph for service=billing, p99 last 1h"
   ┌─────────────────────────────────────────┐
   │  UI: flame graphs, source-line attribution,
   │      diff between deploys, label filters     │
   └─────────────────────────────────────────┘
```

Two facts:

1. **A pprof profile is a `profile.Profile` protobuf** (`google/pprof`). All tools speak it. You're never locked in.
2. **Sampling is statistical.** A 10-second CPU profile at 100Hz captures ~1000 samples. Functions that consume <0.1% of CPU may not appear at all. This is intentional — overhead must stay low.

## Enable pprof in Your Service

```go
import (
    "net/http"
    _ "net/http/pprof"  // registers handlers at /debug/pprof/
)

func main() {
    go func() {
        // Bind pprof to a non-public port
        log.Fatal(http.ListenAndServe("localhost:6060", nil))
    }()
    // ... rest of your service
}
```

The blank import registers handlers on `http.DefaultServeMux`. For a service that uses a custom mux, register them explicitly:

```go
import "net/http/pprof"

mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
mux.Handle("/debug/pprof/heap", pprof.Handler("heap"))
mux.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine"))
mux.Handle("/debug/pprof/block", pprof.Handler("block"))
mux.Handle("/debug/pprof/mutex", pprof.Handler("mutex"))
mux.Handle("/debug/pprof/allocs", pprof.Handler("allocs"))
```

### Enabling block + mutex profiles (off by default)

```go
import "runtime"

runtime.SetBlockProfileRate(1)         // every blocking event
runtime.SetMutexProfileFraction(1)     // every contended unlock
```

Both can be expensive at high rates. Common production values: `SetBlockProfileRate(100_000_000)` (sample 1 of every 100ms blocking) and `SetMutexProfileFraction(100)` (1% of contended unlocks).

## The Profiles

| Profile | What it shows | Cost | Typical cadence |
|---------|---------------|------|-----------------|
| `profile` (CPU) | Where CPU time was spent | ~1–3% per active profile | 10s every 60s |
| `heap` | Live + total allocated memory by call site | snapshot, ~10ms | every 30–60s |
| `allocs` | Cumulative allocations since process start | snapshot | rare; for trend lines |
| `goroutine` | Current goroutines and their stacks | snapshot, scales with N goroutines | on demand or every minute |
| `block` | Goroutines blocked on syscall/channel/mutex | sampled per `SetBlockProfileRate` | every 60s if rate enabled |
| `mutex` | Contended mutex acquisitions | sampled per `SetMutexProfileFraction` | every 60s if rate enabled |
| `threadcreate` | OS thread creations | snapshot | rare |
| `trace` | Per-event execution trace (very expensive) | ~10–30%! | manual debugging only |

## CPU Profile in Production

```go
// Manual capture from CLI
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

// In the UI
(pprof) top10
(pprof) list myFunction
(pprof) web        # opens flame graph in browser
```

For continuous profiling, the scraper does this for you.

## Heap Profile

```bash
go tool pprof http://localhost:6060/debug/pprof/heap

# Show top 10 by inuse_space (default)
(pprof) top10

# Switch to "what was allocated total" (alloc_space)
(pprof) sample_index=alloc_space
(pprof) top10
```

Four sample types:
- `alloc_objects` — count of objects allocated since start
- `alloc_space` — bytes allocated since start
- `inuse_objects` — currently live object count
- `inuse_space` — currently live bytes

For diagnosing leaks: trend `inuse_space` over time, then capture two heap profiles and diff:

```bash
go tool pprof -base before.pb.gz after.pb.gz
```

## Goroutine Profile

```bash
# Quick textual view
curl localhost:6060/debug/pprof/goroutine?debug=2 | less
```

`debug=2` produces a full text dump with one stack per goroutine — extremely useful when you suspect goroutine leaks or deadlocks. `debug=1` produces aggregated stacks; `debug=0` (default) is the protobuf for pprof tools.

## pprof Labels — The Superpower

```go
import "runtime/pprof"

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := pprof.WithLabels(r.Context(), pprof.Labels(
        "route", chi.RouteContext(r.Context()).RoutePattern(),
        "tenant", r.Header.Get("X-Tenant"),
        "method", r.Method,
    ))
    pprof.SetGoroutineLabels(ctx)
    // ... do work — all subsequent samples will be tagged
    process(ctx, r)
}

func process(ctx context.Context, r *http.Request) {
    // pprof.Do attaches labels for the duration of the function
    pprof.Do(ctx, pprof.Labels("phase", "validate"), func(ctx context.Context) {
        validate(r)
    })
    pprof.Do(ctx, pprof.Labels("phase", "persist"), func(ctx context.Context) {
        persist(ctx, r)
    })
}
```

Now in the UI:

- "Show flame graph filtered by `route=/users/{id}`"
- "Compare CPU between `tenant=acme` and `tenant=other`"
- "Where does the `phase=persist` work concentrate?"

Labels are **per-goroutine** and inherited by goroutines spawned while a label is active. They appear in `profile.Sample.Label` in the pprof protobuf.

Cost: ~ns to set, no allocation if labels are constants.

## Parca

Parca (https://www.parca.dev) is open-source, OSS-licensed, scraping-based. Architecture mirrors Prometheus: a central Parca server scrapes pprof endpoints periodically, stores delta-encoded profiles, exposes a query UI.

```yaml
# parca.yaml
scrape_configs:
  - job_name: 'go-services'
    scrape_interval: 30s
    profiling_config:
      pprof_config:
        memory:    { enabled: true,  path: /debug/pprof/heap     }
        block:     { enabled: false                              }
        goroutine: { enabled: true,  path: /debug/pprof/goroutine }
        mutex:     { enabled: false                              }
        process_cpu: { enabled: true, delta: true, path: /debug/pprof/profile, seconds: 30 }
    static_configs:
      - targets: ['app1:6060', 'app2:6060']
```

Parca + the `parca-agent` eBPF-based off-CPU profiler captures profiles without modifying the binary — useful for short-lived processes or processes you can't add pprof to (e.g., older Go versions).

## Pyroscope (Grafana)

Pyroscope (https://pyroscope.io, now part of Grafana) is open-source, integrates natively into Grafana's UI. Either:

1. **Pull mode** — Pyroscope scrapes `/debug/pprof/*`.
2. **Push mode** — your app uses the Pyroscope Go client to push profiles.

```go
import "github.com/grafana/pyroscope-go"

profiler, _ := pyroscope.Start(pyroscope.Config{
    ApplicationName: "billing.prod",
    ServerAddress:   "http://pyroscope:4040",
    Logger:          pyroscope.StandardLogger,
    Tags:            map[string]string{"region": "us-east-1"},
    ProfileTypes: []pyroscope.ProfileType{
        pyroscope.ProfileCPU,
        pyroscope.ProfileAllocObjects,
        pyroscope.ProfileAllocSpace,
        pyroscope.ProfileInuseObjects,
        pyroscope.ProfileInuseSpace,
    },
})
defer profiler.Stop()
```

The Pyroscope client respects pprof labels (`pyroscope.TagWrapper`). The major operational benefit: tight Grafana integration — exemplars from a Prometheus latency panel can jump to the flame graph at that exact time window.

## Polar Signals (Commercial)

Polar Signals Cloud is the commercial offering from the Parca team — managed Parca + extras (better diff UI, deploy-correlation, more retention, SAML/SSO). Same protocol; switching is `s/parca/polar-signals/` in config.

## Diffing Two Profiles

```bash
# Capture before and after a deploy
curl http://app:6060/debug/pprof/profile?seconds=60 > before.pb.gz
# ... deploy v2 ...
curl http://app:6060/debug/pprof/profile?seconds=60 > after.pb.gz

# Diff
go tool pprof -base before.pb.gz after.pb.gz
(pprof) web
```

Functions in red consumed *more* CPU after; functions in green consumed less. The single most useful technique for evaluating performance impact of a deploy.

Continuous profilers automate this: every flame graph view has "compare to N hours/days ago" or "compare across deploys."

## Symbolisation

For pprof to display function names, the binary needs symbols. By default Go binaries include them. **Don't strip with `-ldflags="-s -w"`** if you want readable flame graphs from continuous profiling. If you must strip for size reasons, keep the unstripped binary in a symbol server (`debuginfod`, Parca's debuginfo upload).

```bash
# Strip and keep separate symbols
go build -o app ./cmd/app
objcopy --only-keep-debug app app.debug
strip app
# Upload app.debug to your symbol server
```

## Securing pprof Endpoints

**Never expose `/debug/pprof/*` to the public internet.** They reveal source code structure, sometimes config, and allow expensive CPU profiles that DoS your service.

- Bind to `localhost` and use a kubectl port-forward or sidecar scraper.
- Behind mTLS or a dedicated metrics network.
- NetworkPolicy in Kubernetes restricting `:6060` to the profiler service account.

## Anti-Patterns & Gotchas

**Exposing `/debug/pprof/` publicly.** CVE-class mistake.

**Profiling a stripped binary.** Flame graph shows hex addresses.

**Running CPU profile back-to-back-to-back.** Overlapping profiles ~double overhead. Use the profiler's scheduling.

**Forgetting `SetBlockProfileRate(0)` (off) is the default.** You won't see channel/mutex waits in `block` profile until you turn it on.

**Setting `SetMutexProfileFraction(1)` in prod.** 100% sampling on a contended path is expensive. Use `100` (1%) or `1000` (0.1%).

**Trusting CPU profile of <5s.** Statistical noise dominates short profiles. 30s minimum for meaningful analysis.

**Profile via `httptest.Server`-only handler.** The `_ "net/http/pprof"` blank import registers on `DefaultServeMux` — your test server may not include it.

**Comparing `inuse_space` between deploys without GC behavior context.** The new code may simply have triggered GC less often.

**Using `goroutine` profile to find leaks without a baseline.** "10,000 goroutines" might be normal for a high-fanout service. Compare to your steady-state baseline.

**Symbolising client-side for huge profiles** — Parca/Pyroscope symbolise server-side, much faster.

**Re-scraping CPU at 1Hz.** That's a continuous load test. 1-per-minute or 1-per-30s is plenty.

**Using `runtime.ReadMemStats` for "live heap" in dashboards.** It's the post-last-GC snapshot. Use `/gc/heap/live:bytes` from `runtime/metrics` or current heap from continuous profiling.

**Forgetting pprof labels can include secrets if you're careless.** Don't put session tokens or PII in label values.

**Profile capture from inside a `panic` recovery.** The runtime is in a degraded state; you'll get partial data.

## Performance Notes

- **Default CPU profile (100Hz, 10s):** ~1–3% overhead during the 10s window. Off when not profiling.
- **High-frequency CPU profile (500Hz):** ~5–10% overhead — set `SetCPUProfileRate(500)`.
- **Heap profile snapshot:** ~10ms, allocates ~10–50 MB temporarily (proportional to live heap).
- **Goroutine profile of 100k goroutines:** ~50–200 ms.
- **pprof labels:** <100 ns per `WithLabels` + `SetGoroutineLabels`.
- **Block + mutex sampling at 1%:** <0.5% overhead.
- **eBPF agent (Parca-agent):** ~0.5% CPU overhead, no code change required.

## Real Workflow

1. **Always-on continuous profiling**: 30s CPU + heap every 60s, no labels at startup.
2. **Add pprof labels at the request edge**: `route`, `method`, `tenant_tier`. Never high-cardinality (`user_id`, `request_id`).
3. **Alert on profile-derived signals**: e.g., GC CPU >5% (computable from `runtime/metrics` and shown in the profiler too).
4. **At incident time**: open the profiler, filter by `route` for the affected endpoint, compare to last week.
5. **During code review** for performance-sensitive PRs: run `pprof -base` on a benchmark.

## How Big Companies Use It

- **Google** invented pprof and runs continuous profiling internally (Google-Wide Profiling / GWP). Public talks discuss extracting performance signal at fleet scale.
- **Polar Signals** is the company building Parca; founded by ex-Red Hat / ex-Prometheus engineers (Frederic Branczyk).
- **Grafana Labs** acquired Pyroscope; the product is now first-class in Grafana Cloud.
- **Datadog** ships continuous profiling for Go natively in their Agent.
- **Cloudflare** uses pprof + Parca extensively; their blog has detailed posts on profile-driven Go optimisation.
- **Discord** uses continuous profiling for Go's `boltdb` performance work and JIT/GC tuning.
- **Uber** invented Pyroscope-class tooling (now public via Pyroscope OSS) for their Go fleet.
- **Tailscale** uses pprof labels per node for cross-fleet correlation; profiles uploaded to Grafana Cloud Profiles.
- **Bytedance** built their own continuous profiler around pprof for their Go services.

## Source Code References

- `net/http/pprof`: https://github.com/golang/go/tree/master/src/net/http/pprof.
- `runtime/pprof`: https://github.com/golang/go/tree/master/src/runtime/pprof.
- `google/pprof` (the protobuf and tooling): https://github.com/google/pprof.
- Parca: https://github.com/parca-dev/parca.
- Parca Agent (eBPF): https://github.com/parca-dev/parca-agent.
- Pyroscope: https://github.com/grafana/pyroscope.
- Pyroscope Go client: https://github.com/grafana/pyroscope-go.
- OTel profiles proto (forthcoming standard): https://github.com/open-telemetry/oteps/pull/239.

## Further Reading

- Russ Cox, "Profiling Go programs": https://go.dev/blog/pprof.
- Felix Geisendörfer, "The busy developer's guide to Go profiling, tracing and observability": https://github.com/DataDog/go-profiler-notes.
- Frederic Branczyk, "Continuous Profiling: An Introduction" (Polar Signals blog).
- Liz Fong-Jones, "Observability, profiling, and Go" (Honeycomb talks).
- "Continuous Profiling" chapter in the SRE Workbook.
- "Differential Profiling" (Google patent, 2009) — diff-based profiling.

## Exercises / Self-Check

1. Enable `net/http/pprof` on a service. Pull a 30-second CPU profile with `go tool pprof http://.../debug/pprof/profile?seconds=30`. Identify the top 3 hottest functions.
2. Add pprof labels at your request edge (`route`, `tenant_tier`). Capture a profile under mixed load. Filter the flame graph by one label value in the UI.
3. Deploy Parca via Helm. Configure it to scrape your dev service every 30s. Watch profile retention over a day.
4. Take two heap profiles 10 minutes apart on a leaking service. Use `pprof -base` to identify the leak.
5. Turn on `SetBlockProfileRate(100_000)` and capture a block profile under contention. Confirm channel sends/receives show up.
6. Compare two binaries (before/after an optimisation). Diff their CPU profiles. Decide whether the change was a net win.
7. Add the Pyroscope Go client to your service in push mode. Confirm profiles appear in Grafana with your tags.
8. Strip your binary; observe that flame graphs become unusable. Unstrip; restore.
