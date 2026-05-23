# Uber — 3500+ Go Microservices, Cadence, Jaeger

## TL;DR

Uber runs one of the largest Go fleets outside Google: **thousands of Go microservices** powering rides, eats, freight, financial services. Major Go-built systems include **Jaeger** (distributed tracing — donated to CNCF), **Cadence** (workflow orchestration — Temporal's parent), **Aresdb** (GPU-accelerated time-series DB), **Zap** (structured logging), **Fx** (DI framework), **Yarpc** (RPC framework), and **Geofence** (geospatial indexing). Go was chosen in 2015 as the standard for new services. The single biggest gotcha: **Uber's scale exposed many of Go's early performance edges** — container-unaware GOMAXPROCS, GC pause hotspots on pointer-heavy heaps, the per-goroutine cost of "one goroutine per request" patterns at millions of QPS. Many of Uber's open-source libraries (`automaxprocs`, `zap`, `goleak`) exist because they solved problems first observed at Uber's scale.

## Mental Model

```
   Uber technology stack (2016–2026):
   
   ┌──────────────────────────────────────────────────────────┐
   │  Front-line services (Go)                                │
   │   - rider/driver app backends                            │
   │   - dispatch (matching), pricing, payments               │
   │   - thousands of microservices, mostly Go                │
   └─────────┬─────────────────────┬────────────────────────────┘
             ▼                     ▼
   ┌────────────────────┐  ┌─────────────────────┐
   │  Cadence/Temporal  │  │  Jaeger              │
   │  workflow engine   │  │  distributed tracing │
   │  (Go server, Go SDK)│  │  (Go collector,      │
   │                    │  │  Go agent, Go UI)    │
   └────────────────────┘  └─────────────────────┘
             │
             ▼
   ┌─────────────────────────────────────────────────────────┐
   │  Data infrastructure                                     │
   │   - Cassandra (Java, accessed via Go gocql client)       │
   │   - Kafka (Java, Go consumers/producers)                 │
   │   - M3DB (Go time-series DB)                             │
   │   - AresDB (Go + CUDA)                                   │
   └─────────────────────────────────────────────────────────┘
```

The microservice mesh is glued together by **YARPC** (Uber's RPC framework, Go-native, supports HTTP/Thrift/gRPC) and traced through Jaeger.

## Syntax & Basic Usage

A representative Uber-style service skeleton:

```go
package main

import (
	"context"

	"go.uber.org/fx"
	"go.uber.org/zap"
)

type Config struct {
	Port int `yaml:"port"`
}

type Server struct {
	log *zap.Logger
	cfg Config
}

func NewServer(log *zap.Logger, cfg Config) *Server {
	return &Server{log: log, cfg: cfg}
}

func (s *Server) Start(ctx context.Context) error {
	s.log.Info("starting", zap.Int("port", s.cfg.Port))
	return nil
}

func main() {
	app := fx.New(
		fx.Provide(
			zap.NewProduction,
			func() Config { return Config{Port: 8080} },
			NewServer,
		),
		fx.Invoke(func(s *Server) {
			_ = s.Start(context.Background())
		}),
	)
	app.Run()
}
```

`fx` provides dependency injection; `zap` is structured logging. Both are Uber-built and widely adopted outside Uber.

## Deep Dive

### Why Go (the Uber story)

Uber's adoption is documented in posts by Ankit Jain, Tyler Treat, and others. The 2015–2016 transition was driven by:

- **Node.js issues at scale**: dispatch (the matching service) was Node; suffered from event-loop saturation under load. Latency tail was unacceptable for ride matching.
- **Python's GIL** for compute-heavy backend work.
- **Java's startup time** and heap memory cost.

Go offered:
- Predictable latency vs Node's tail.
- Real parallelism vs Python's GIL.
- Sub-second startup vs Java's many-second JVM warmup.
- Single static binary deploys.

By 2018 Uber's "production" stack defaulted to Go for new services. As of 2024 their internal estimates put Go at thousands of services and millions of QPS.

### Cadence (and Temporal)

[Cadence](https://github.com/uber/cadence) is Uber's workflow-as-code engine. You write a workflow function in Go (or Java); Cadence runs it with **durable execution** — state survives crashes, retries are automatic, code can sleep for days.

```go
package workflows

import (
	"time"

	"go.uber.org/cadence/workflow"
)

func OrderWorkflow(ctx workflow.Context, orderID string) error {
	ao := workflow.ActivityOptions{
		ScheduleToCloseTimeout: 24 * time.Hour,
		HeartbeatTimeout:       30 * time.Second,
	}
	ctx = workflow.WithActivityOptions(ctx, ao)

	var charge ChargeResult
	if err := workflow.ExecuteActivity(ctx, ChargeCard, orderID).Get(ctx, &charge); err != nil {
		return err
	}

	if err := workflow.ExecuteActivity(ctx, ShipOrder, orderID).Get(ctx, nil); err != nil {
		// Compensate
		_ = workflow.ExecuteActivity(ctx, RefundCard, charge.TxnID).Get(ctx, nil)
		return err
	}

	// Sleep 7 days, then send a follow-up.
	_ = workflow.Sleep(ctx, 7*24*time.Hour)
	return workflow.ExecuteActivity(ctx, SendFollowup, orderID).Get(ctx, nil)
}
```

The trick: every Cadence operation (`ExecuteActivity`, `Sleep`, `WithActivityOptions`) is **deterministically replayable**. The workflow function may run hundreds of times during its lifetime, each time replaying past decisions from history. Activities (side effects) run exactly once.

Cadence became Temporal (https://temporal.io) — same architecture, same Go SDK, run as a SaaS. Uber still uses Cadence internally.

### Jaeger

[Jaeger](https://github.com/jaegertracing/jaeger) is Uber's distributed tracing platform — donated to CNCF in 2017. Components:

- **Agent** (Go): runs on every host, receives spans from local apps via UDP.
- **Collector** (Go): receives spans from agents, validates, writes to storage.
- **Query** (Go): reads from storage, serves the UI.
- **Storage**: Cassandra, Elasticsearch, or Memory (dev).

Jaeger pioneered W3C TraceContext support; its OpenTracing client became the basis for OpenTelemetry's Go SDK.

### M3DB

[M3DB](https://github.com/m3db/m3) is Uber's time-series database, replacing Graphite. Written in Go. Designed for:

- Ingestion of millions of data points per second.
- Multi-tenant tag-based queries (Prometheus-compatible).
- Long-term retention with downsampling.

The Go implementation is notable for aggressive allocator tuning — custom `xpool` package replaces `sync.Pool` for some hot paths, with stricter lifetime control.

### Zap — structured logging

[zap](https://github.com/uber-go/zap) was open-sourced in 2016 because Uber needed logging at millions of requests per second and `log` / `logrus` / `glog` were not fast enough.

Design points:
- Zero allocation in the structured field path.
- Pre-encoded fields where possible.
- Two APIs: `*zap.Logger` (typed fields, zero alloc) and `*zap.SugaredLogger` (printf-like, some alloc).

Benchmarks (from zap's README):
- zap structured: ~250 ns/op, 0 allocs.
- logrus: ~3.5 µs/op, 50+ allocs.
- standard `log`: ~3 µs/op, ~10 allocs.

The lib's APIs are very type-driven:

```go
logger.Info("processing order",
    zap.String("order_id", id),
    zap.Int("items", n),
    zap.Duration("elapsed", elapsed),
)
```

The trade-off: less ergonomic than `logger.Infof("%s %d", ...)`, but ~10x faster.

### Fx — dependency injection

[fx](https://github.com/uber-go/fx) is Uber's DI framework, built on top of [dig](https://github.com/uber-go/dig). Pattern: declare constructors, fx wires them.

```go
fx.New(
    fx.Provide(NewLogger),         // *zap.Logger
    fx.Provide(NewHTTPServer),     // *http.Server, depends on *zap.Logger
    fx.Provide(NewMetrics),
    fx.Invoke(func(*http.Server) {}),
).Run()
```

The constructor's parameters are dependencies; fx topologically sorts and instantiates. Used internally at Uber for almost every Go service. Alternative: Google's [wire](https://github.com/google/wire) (compile-time codegen instead of runtime reflection).

### automaxprocs

[automaxprocs](https://github.com/uber-go/automaxprocs) addresses the issue: pre-Go-1.25, `runtime.GOMAXPROCS` defaulted to `runtime.NumCPU()` — the *host* CPU count, ignoring cgroup quotas. In a 2-CPU container on a 64-CPU host, `GOMAXPROCS=64` led to terrible context-switch performance.

```go
import _ "go.uber.org/automaxprocs"

// In main package, side-effect import sets GOMAXPROCS at startup
// based on cgroup CPU quota.

func main() { /* ... */ }
```

Uber published this in 2017. Go 1.25 finally made the behavior built-in.

### goleak

[goleak](https://github.com/uber-go/goleak) detects goroutine leaks in tests:

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

After all tests run, goleak fails if any goroutines (other than the test runner's own) are still alive. Catches "I started a goroutine and forgot to cancel" bugs at CI time. Used by every Go service team at Uber.

### YARPC

YARPC ("Yet Another RPC") is Uber's polyglot RPC framework. Supports:
- Thrift (legacy, dominant pre-2020).
- HTTP/JSON.
- gRPC.

Uber's Cadence and several internal services still use Thrift via YARPC. Migration to plain gRPC has been gradual.

### TChannel

[TChannel](https://github.com/uber/tchannel-go) was Uber's pre-YARPC RPC protocol — multiplexed, bidirectional, sub-millisecond. Largely replaced by gRPC + YARPC adapters now.

### AresDB

[AresDB](https://github.com/uber/aresdb) is Uber's GPU-accelerated time-series store. Go orchestration layer + CUDA compute. Used for sub-second OLAP on real-time data (driver heatmaps, ETA tracking).

### Geofence

Uber published several geospatial libraries:
- [H3](https://github.com/uber/h3-go): hexagonal hierarchical geospatial index, with Go bindings.
- Internal "city-shard" routing relies on H3 for partition keys.

### Performance lessons learned

#### 1. Allocator pressure dominates GC pause

Hot paths that allocate per request (JSON unmarshal, string concatenation) trigger GC under load. Zap, custom protobuf code, and sync.Pool patterns reduced allocations.

#### 2. Goroutine count grows with QPS

"One goroutine per request" works at 1k QPS, struggles at 1M QPS. Uber's internal codegen tends to produce bounded worker pools; `errgroup.WithLimit` (1.21+) helps.

#### 3. Tail latency from GC

Pre-Go-1.8 STW pauses of >100 ms were unacceptable. Uber heavily contributed to feedback that drove the 1.8 hybrid write barrier work. Modern Go GC pauses (sub-millisecond) are not the bottleneck anymore.

#### 4. CPU spikes from container CFS throttling

CFS quotas plus over-subscribed CPUs caused "CPU starvation" spikes that looked like GC pauses but weren't. automaxprocs + careful Go CFS-aware sysmon (1.14+) help.

### Cultural patterns

Uber's Go style guide (https://github.com/uber-go/guide) is widely adopted outside Uber:
- Error wrapping with `errors.Is/As`, no `pkg/errors`.
- Constructors return `*T`, not `T`.
- Initialization via fx.
- Tests use `goleak`.
- Logging via zap.
- No global state; configs via constructor.

## Standard Library Hooks

Common stdlib uses across Uber's Go ecosystem:

- `net/http`: HTTP servers; sometimes wrapped by YARPC.
- `context`: ubiquitous; every RPC carries one.
- `database/sql`: paired with `pgx` or `gocql` for Cassandra.
- `sync.Pool`: per-request buffer reuse.
- `runtime/pprof` + Jaeger spans: combined for "where did this request spend its time".
- `runtime/debug.SetMemoryLimit`: Uber rolled out widely after 1.19.
- `runtime/metrics`: feeds custom Prometheus exporters.

## Real-World Patterns

### 1. Service skeleton with fx + zap

```go
package main

import (
	"context"
	"net/http"

	_ "go.uber.org/automaxprocs"
	"go.uber.org/fx"
	"go.uber.org/zap"
)

func NewMux(log *zap.Logger) *http.ServeMux {
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		log.Info("health check")
		w.WriteHeader(200)
	})
	return mux
}

func NewServer(lc fx.Lifecycle, mux *http.ServeMux, log *zap.Logger) *http.Server {
	srv := &http.Server{Addr: ":8080", Handler: mux}
	lc.Append(fx.Hook{
		OnStart: func(_ context.Context) error {
			go func() {
				if err := srv.ListenAndServe(); err != http.ErrServerClosed {
					log.Fatal("server", zap.Error(err))
				}
			}()
			return nil
		},
		OnStop: func(ctx context.Context) error { return srv.Shutdown(ctx) },
	})
	return srv
}

func main() {
	fx.New(
		fx.Provide(zap.NewProduction),
		fx.Provide(NewMux),
		fx.Provide(NewServer),
		fx.Invoke(func(*http.Server) {}),
	).Run()
}
```

### 2. Cadence workflow

```go
func ReportFraudWorkflow(ctx workflow.Context, txID string) error {
	ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		ScheduleToCloseTimeout: time.Hour,
	})
	var score float64
	if err := workflow.ExecuteActivity(ctx, ScoreFraud, txID).Get(ctx, &score); err != nil {
		return err
	}
	if score > 0.8 {
		return workflow.ExecuteActivity(ctx, FreezeAccount, txID).Get(ctx, nil)
	}
	return nil
}
```

Replayable, durable, retried automatically.

### 3. Jaeger client (OpenTelemetry compatible)

```go
import (
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/jaeger"
	"go.opentelemetry.io/otel/sdk/trace"
)

func setup() func() {
	exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(jaeger.WithEndpoint("http://jaeger:14268/api/traces")))
	tp := trace.NewTracerProvider(trace.WithBatcher(exp))
	otel.SetTracerProvider(tp)
	return func() { _ = tp.Shutdown(context.Background()) }
}
```

### 4. goleak in tests

```go
func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m,
		goleak.IgnoreTopFunction("internal/poll.runtime_pollWait"),
	)
}
```

CI catches goroutine leaks per package.

### 5. errgroup with bounded concurrency

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)
g.SetLimit(20) // cap concurrent goroutines

for _, item := range items {
	g.Go(func() error {
		return process(ctx, item)
	})
}
if err := g.Wait(); err != nil { return err }
```

Uber-style: never unbounded fan-out.

## Anti-Patterns & Gotchas

**Logging with `fmt.Sprintf`.** Zap's typed fields are dramatically faster — Uber-scale services would otherwise spend significant CPU on formatting.

**Goroutine-per-request without bounds.** At millions of QPS, this saturates the run queue. Use worker pools.

**Using `runtime.GOMAXPROCS = runtime.NumCPU()` manually.** Pre-1.25, this was the bug `automaxprocs` fixed. Post-1.25, the runtime does it; either way, don't override.

**Cadence workflows with non-deterministic code.** Calling `time.Now()`, `rand.Intn`, or accessing globals breaks replay. Use `workflow.Now`, `workflow.NewRandom`.

**Storing huge data in workflow history.** Cadence persists every workflow event; large payloads bloat the history table. Pass IDs, fetch data in activities.

**Trusting Jaeger sampling at 100%.** Cost (network + storage) is significant. Adaptive sampling (1% baseline, 100% for errors) is the norm.

**Skipping fx and rolling your own DI.** Works for a 5-file project; doesn't scale. The discipline of constructor-based wiring pays off.

**Treating zap as drop-in for logrus.** API is different (typed fields). Migration must be deliberate.

**Forgetting `defer cancel()` after `context.WithCancel`.** Goroutine leaks; goleak catches in tests; production sees memory growth.

**Using `sync.Map` everywhere "for concurrency".** It's slower than `RWMutex+map` for low contention. Use only for write-once-read-many patterns.

## Performance Notes

Approximate numbers from Uber's posts and benchmarks:

- Zap structured log: ~250 ns/op, 0 allocs.
- Jaeger span emission (sampled at 1%): negligible.
- Cadence task workflow worker: ~10k workflows/sec per worker.
- YARPC Thrift RPC: ~50–100 µs intra-DC.
- gRPC over HTTP/2: similar.
- Go service warm-up: ~1 s.
- Container CPU throttling with automaxprocs: tail latency improves 30–50%.
- M3DB ingestion: millions of points/sec per node.

## How Other Big Companies Use Uber's Tools

- **Temporal Technologies** (started by Cadence's authors): SaaS version of Cadence. Customers include Datadog, Coinbase, Snap.
- **Stripe** uses zap.
- **Cloudflare** uses zap, goleak.
- **Tailscale** uses goleak in tests.
- **GitHub** has used Jaeger in internal observability stacks.
- **DataDog** internally uses Cadence concepts for their workflow systems.
- **The Go team** itself uses `goleak` in some toolchain test suites.

## Source Code References

- Cadence: [`uber/cadence`](https://github.com/uber/cadence).
- Temporal Go SDK (Cadence successor): [`temporalio/sdk-go`](https://github.com/temporalio/sdk-go).
- Jaeger: [`jaegertracing/jaeger`](https://github.com/jaegertracing/jaeger).
- M3DB: [`m3db/m3`](https://github.com/m3db/m3).
- AresDB: [`uber/aresdb`](https://github.com/uber/aresdb).
- Zap: [`uber-go/zap`](https://github.com/uber-go/zap).
- Fx: [`uber-go/fx`](https://github.com/uber-go/fx).
- Dig: [`uber-go/dig`](https://github.com/uber-go/dig).
- goleak: [`uber-go/goleak`](https://github.com/uber-go/goleak).
- automaxprocs: [`uber-go/automaxprocs`](https://github.com/uber-go/automaxprocs).
- TChannel: [`uber/tchannel-go`](https://github.com/uber/tchannel-go).
- H3 (geospatial): [`uber/h3-go`](https://github.com/uber/h3-go).
- Go style guide: [`uber-go/guide`](https://github.com/uber-go/guide).

(Apache-2.0 and MIT licenses; per-project.)

## Further Reading

- "How We Built Uber Engineering's Highest Query per Second Service Using Go" (2018): https://eng.uber.com/go-geofence/.
- "Cadence at Uber" — Maxim Fateev (creator) talks: https://www.youtube.com/c/TemporalIO.
- "Jaeger: Open source distributed tracing" — Yuri Shkuro (creator): https://www.uber.com/blog/distributed-tracing/.
- "Zap: Blazing fast, structured, leveled logging" — uber-go/zap README.
- "How Uber tests microservices with Go" (talks).
- "Uber's experience scaling Go" — GopherCon talks (various years).
- Tyler Treat, "Beyond microservices" — multi-year retrospective: https://bravenewgeek.com.
- Uber Go style guide: https://github.com/uber-go/guide/blob/master/style.md.

## Exercises / Self-Check

1. Why does Cadence require workflow code to be deterministic? What happens during a replay, and what kinds of common Go patterns break it?
2. Compare a tight log statement using `log.Printf("%s %d", x, y)` vs `zap.Logger.Info("msg", zap.String("x", x), zap.Int("y", y))`. Why is the latter faster?
3. Set up automaxprocs in a container with `cpu: "0.5"` limit. Verify `runtime.GOMAXPROCS(0)` returns 1, not the host count. Now compare with `GOMAXPROCS=0` on Go 1.25+ — do you still need the library?
4. Write a Cadence-style workflow (using Temporal SDK) that charges a card, ships, and refunds on failure. Verify the workflow survives a worker restart.
5. Why is `goleak.VerifyTestMain` better than per-test `goleak.VerifyNone`? When would you prefer the latter?
