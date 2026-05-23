# Distributed Tracing — OpenTelemetry Go

## TL;DR

OpenTelemetry (OTel) is the CNCF-graduated observability standard for traces, metrics, and logs. The Go SDK (`go.opentelemetry.io/otel`) is mature, stable for traces and metrics (since 2023), and stable for logs (since 2024). Three concepts you must internalise: a **trace** is a tree of **spans**; a **span** has start/end time, attributes, events, and parent-child relationships; **propagation** carries the current trace context across process boundaries via HTTP headers (`traceparent`, `tracestate`) or gRPC metadata. The collection pipeline is **SDK in your process → OTLP (gRPC or HTTP/protobuf) → OTel Collector → vendor backend** (Jaeger, Tempo, Honeycomb, Datadog, Lightstep, New Relic, …). The Collector decouples your apps from any one vendor and is where you batch, sample, filter, and route. The single biggest gotcha: **the default `AlwaysSample()` sampler in dev becomes "ship 100% of prod traces" in prod** — your storage bill goes vertical. Always configure `ParentBased(TraceIDRatioBased(0.01))` or smarter (tail sampling at the Collector) before going to prod.

## Mental Model

```
   Service A ──HTTP w/ traceparent────► Service B ──gRPC──► Service C
       │                                    │                   │
       └─ Span("A.handler")                 └─ Span("B.work")   └─ Span("C.lookup")
                  │                                  │                  │
                  └──────────────────────────────────┴──────────────────┘
                              All three spans share a trace_id;
                              parent_span_id links them.
                                          │
                                          ▼
                          ┌─────────────────────────────────┐
                          │  OTel SDK in each process        │
                          │  - TracerProvider                │
                          │  - BatchSpanProcessor            │
                          │  - OTLPExporter (gRPC or HTTP)   │
                          └─────────────────────────────────┘
                                          │
                                          ▼
                          ┌─────────────────────────────────┐
                          │   OTel Collector                 │
                          │   - receivers (OTLP, Jaeger, ...) │
                          │   - processors (batch, sample)    │
                          │   - exporters (Tempo, Honeycomb)  │
                          └─────────────────────────────────┘
```

## Setup

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
       go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

### Initialize a TracerProvider

```go
import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func initTracer(ctx context.Context) (func(context.Context) error, error) {
    exp, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),  // typical inside the cluster
    )
    if err != nil { return nil, err }

    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("billing"),
            semconv.ServiceVersion("v1.4.2"),
            semconv.DeploymentEnvironmentName("prod"),
        ),
        resource.WithHost(),
        resource.WithOSType(),
    )

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp, sdktrace.WithBatchTimeout(5*time.Second)),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(0.01),  // 1% head sampling
        )),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    return tp.Shutdown, nil
}
```

```go
func main() {
    ctx := context.Background()
    shutdown, err := initTracer(ctx)
    if err != nil { log.Fatal(err) }
    defer shutdown(ctx)

    // ... your server
}
```

Three knobs you tune most often: **exporter endpoint**, **sampling**, **resource attributes** (the service identity). `semconv` ships canonical attribute keys — use them; vendor backends index by these.

## Creating Spans

```go
tracer := otel.Tracer("github.com/example/billing")  // module-scoped tracer

func processOrder(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "processOrder",
        trace.WithAttributes(
            attribute.String("order.id", orderID),
            attribute.Int("retry", 0),
        ),
    )
    defer span.End()

    // ... do work; sub-spans inherit ctx
    if err := validate(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "validation failed")
        return err
    }
    span.SetStatus(codes.Ok, "")
    return nil
}
```

Rules:

- **`ctx` flows through every function** that participates in tracing. If you take `context.Context`, you should propagate the trace.
- **`defer span.End()`** immediately after `Start`. Forgetting this leaves orphan spans.
- **`SetStatus(codes.Error, ...)`** marks the span as failed for visualizations.
- **`RecordError(err)`** adds an exception event with stack trace.

### Attributes

```go
span.SetAttributes(
    attribute.String("http.method", "POST"),
    attribute.Int("http.status_code", 200),
    attribute.Bool("cache.hit", true),
    attribute.StringSlice("tags", []string{"premium", "us"}),
)
```

Use `semconv.*` constants where they exist (e.g., `semconv.HTTPRequestMethodKey.String("POST")`); vendor backends correlate them across services.

### Events

```go
span.AddEvent("cache.refresh", trace.WithAttributes(
    attribute.Int("evicted", 17),
))
```

Events are timestamped log-like points within a span. Use sparingly — sub-spans are usually better when work is non-trivial.

## HTTP Instrumentation

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Server side — wrap your handler
http.Handle("/api/", otelhttp.NewHandler(apiMux, "api"))

// Client side — wrap the transport
client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
    Timeout: 10 * time.Second,
}
```

`otelhttp` reads the `traceparent` header on incoming requests, starts a server span as a child of the incoming context, and propagates context outbound on every request through the wrapped client.

## gRPC Instrumentation

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

// Server
grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))

// Client
conn, _ := grpc.NewClient("backend:50051",
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

Note: `StatsHandler` replaced the older `UnaryServerInterceptor` / `StreamServerInterceptor` approach in 2023; use `NewServerHandler`.

## Propagation

The W3C `traceparent` header looks like:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
            ver  trace_id (32 hex)              span_id (16 hex)   flags
```

`tracestate` adds vendor-specific data (Honeycomb sampler, Datadog tags, etc.).

OTel ships propagators for HTTP, gRPC, NATS, Kafka, AMQP. For an in-house RPC, register your own `TextMapPropagator`:

```go
type carrier map[string]string
func (c carrier) Get(key string) string { return c[key] }
func (c carrier) Set(key, val string)   { c[key] = val }
func (c carrier) Keys() []string {
    out := make([]string, 0, len(c))
    for k := range c { out = append(out, k) }
    return out
}

// Inject (sender)
hdrs := carrier{}
otel.GetTextMapPropagator().Inject(ctx, hdrs)
// Send hdrs alongside RPC

// Extract (receiver)
ctx = otel.GetTextMapPropagator().Extract(ctx, hdrs)
```

## Sampling

Sampling is **the** observability cost lever. Three layers:

### 1. Head sampling — at SDK init

```go
sdktrace.WithSampler(sdktrace.ParentBased(
    sdktrace.TraceIDRatioBased(0.01),  // 1%
))
```

`ParentBased` says "honour the parent's sampling decision, falling back to the inner sampler at trace roots." Critical: without `ParentBased`, an entry-point service samples 1% but downstream services sample independently — and you lose visibility into the cross-service trace whenever they diverge.

### 2. Always-on for errors

```go
type errorSampler struct{ base sdktrace.Sampler }

func (s errorSampler) ShouldSample(p sdktrace.SamplingParameters) sdktrace.SamplingResult {
    // Always sample if the request URL signals an error path
    for _, a := range p.Attributes {
        if a.Key == "http.status_code" && a.Value.AsInt64() >= 500 {
            return sdktrace.SamplingResult{Decision: sdktrace.RecordAndSample}
        }
    }
    return s.base.ShouldSample(p)
}
func (errorSampler) Description() string { return "errorSampler" }
```

Issue: head samplers don't know the span's outcome yet; they decide at `Start`. For "sample all errors," prefer **tail sampling at the Collector**:

### 3. Tail sampling — at the Collector

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
```

Collector buffers spans of a trace for `decision_wait` (typically 10–30s), then decides. Lets you keep 1% of normal traces plus 100% of error/slow traces. Cost: Collector memory scales with trace volume × decision_wait.

## Span Links

When one operation joins multiple traces (fan-in, batch processing):

```go
// Process a batch of N events, each of which has its own incoming trace
links := make([]trace.Link, len(events))
for i, e := range events {
    links[i] = trace.Link{SpanContext: e.SpanContext, Attributes: ...}
}

ctx, span := tracer.Start(ctx, "batch.process", trace.WithLinks(links...))
```

A span can have at most a few hundred links in practice (vendor limits vary). Use for: Kafka consumer batches, scatter-gather queries, cron-style processing of many inputs.

## Baggage

```go
import "go.opentelemetry.io/otel/baggage"

b, _ := baggage.Parse("user.id=42,tier=enterprise")
ctx = baggage.ContextWithBaggage(ctx, b)

// Anywhere downstream
b := baggage.FromContext(ctx)
member := b.Member("user.id")
```

Baggage propagates key-value pairs across all services in a trace. Useful for routing/sampling decisions ("always sample requests from `tier=enterprise`"). Beware: baggage is sent on every cross-process call; over-stuffing means bigger headers.

## Metrics (Brief)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
)

exp, _ := otlpmetricgrpc.New(ctx)
mp := sdkmetric.NewMeterProvider(
    sdkmetric.WithReader(sdkmetric.NewPeriodicReader(exp, sdkmetric.WithInterval(30*time.Second))),
    sdkmetric.WithResource(res),
)
otel.SetMeterProvider(mp)

meter := otel.Meter("github.com/example/billing")
requests, _ := meter.Int64Counter("http.requests")
requests.Add(ctx, 1,
    metric.WithAttributes(
        attribute.String("method", "GET"),
        attribute.String("route", "/users/{id}"),
    ),
)
```

OTel metrics overlap with Prometheus client_golang. Common patterns:

- **OTel-only**: use OTel SDK for metrics + traces, export via OTLP, let the Collector's `prometheusexporter` expose `/metrics` for Prometheus scrape.
- **Prometheus + OTel**: keep `client_golang` for metrics (mature ecosystem, exemplars), add OTel for traces. Bridge exemplars via `otelmetric/prometheus` exporter.

## Logs (Stable in 2024)

```go
import "go.opentelemetry.io/contrib/bridges/otelslog"

logger := otelslog.NewLogger("github.com/example/billing")
logger.InfoContext(ctx, "order processed", "id", orderID)
```

`otelslog` bridges `log/slog` to OTel's log SDK. Logs gain trace_id/span_id correlation automatically when the context carries a span.

## Anti-Patterns & Gotchas

**`AlwaysSample` in production.** Storage costs explode. Default to 1–5% head sampling + tail sampling for errors.

**Forgetting `defer span.End()`.** Leaves orphan spans; SDK may pool unbounded memory.

**Span names with high cardinality.** `Tracer.Start(ctx, "GET /users/" + id)` blows up the span-name dimension. Use the route template.

**Skipping `ctx` through internal functions.** No ctx = no trace propagation = orphan spans. Always take and forward `ctx`.

**Attributes containing entire request bodies.** OTel attributes are not log fields. Keep them small; events for "important moments"; sub-spans for "non-trivial work."

**Mixing `otel.GetTracerProvider().Tracer("...")` everywhere** with module name typos. Use one constant `var tracer = otel.Tracer("...")` at package scope.

**Shipping spans synchronously per call.** Always `WithBatcher`, never `WithSyncer` (the simple-span-processor) in production.

**Missing `Shutdown` on exit.** Buffered spans in the batcher get dropped. Always `defer shutdown(ctx)` with a deadline.

**Tracer per request.** Tracers are cheap but not free; cache one per package.

**Exporting to a Collector you don't run.** Collector ownership matters; ensure it has resource limits, backpressure, and an alert when receiver buffers fill.

**Cross-process clock skew.** Spans can appear to "start before parent" in the UI. Live with it; or run NTP/PTP rigorously.

**Sampling decision drifting across versions.** Document the sampler config; treat changes as a deploy event.

**Forgetting that the Collector is a SPOF.** Run it as a sidecar (per-pod) for resilience, or as a DaemonSet, with redundant downstream exporters.

**Exposing `traceparent` headers in error responses to users.** Leaks internal trace IDs. Strip at the egress proxy.

**No correlation between logs and traces.** Wire `slog` to inject `trace_id`/`span_id`; this single change makes incident debugging 10× faster.

## Performance Notes

- **Span creation** (start + attrs + end): ~1–3 µs.
- **Span batching** flush: 5 second default; configure `WithBatchTimeout`.
- **OTLP gRPC export** of 1000 spans batch: ~5–15 ms.
- **Memory** per pending span: ~1–2 KB.
- **Network** at 1% sampling, 10k req/s: ~10 KB/s outbound (cheap).
- **Network** at 100% sampling, 10k req/s: ~1 MB/s (and growing fast with span depth).

Profile your sampler: every `ShouldSample` runs per trace start. A complex custom sampler with regex/JSON in the hot path is a perf footgun.

## How Big Companies Use It

- **Google** initiated OpenCensus (predecessor to OTel); Go OTel SDK is led/maintained heavily by Google.
- **Honeycomb** built early on OTel; their Go SDK + Beeline are public references.
- **Lightstep / ServiceNow** drives much of OTel spec; their Go reference is a good model.
- **New Relic** uses OTel as primary ingest for Go apps.
- **Datadog** ships `dd-trace-go` natively, with OTel interop via `otlpreceiver` in their Agent.
- **Grafana Labs** runs **Tempo** (OTel-native trace backend); their Go services emit OTel.
- **Cloudflare** uses OTel + Tempo internally; spans-per-day measured in trillions.
- **Shopify** uses OTel for cross-language tracing across Ruby/Go/Node.
- **Uber** Jaeger was the origin of much of OTel's tracing concepts.
- **Kubernetes** (kube-apiserver, kubelet) supports OTLP egress since 1.22+.

## Source Code References

- OTel Go core: https://github.com/open-telemetry/opentelemetry-go.
- OTel Go contrib (instrumentation libraries): https://github.com/open-telemetry/opentelemetry-go-contrib.
- otlphttp/otlpgrpc exporters: https://github.com/open-telemetry/opentelemetry-go/tree/main/exporters/otlp.
- OTel Collector: https://github.com/open-telemetry/opentelemetry-collector.
- Semantic conventions: https://github.com/open-telemetry/semantic-conventions.
- Jaeger: https://github.com/jaegertracing/jaeger.
- Tempo: https://github.com/grafana/tempo.

## Further Reading

- "OpenTelemetry Specification": https://opentelemetry.io/docs/specs/.
- Charity Majors, *Observability Engineering* (O'Reilly) — frames trace-centric debugging.
- "Distributed Tracing in Practice" (Parker, Spoonhower, Mace, Sigelman) — the canonical book.
- "Why You Should Use OpenTelemetry in 2024" (Reinhold Bauer, ad-hoc but widely cited).
- "W3C TraceContext": https://www.w3.org/TR/trace-context/.
- "Sampling in distributed traces" (Honeycomb blog series).
- "Tail sampling vs head sampling" (Lightstep blog).
- "Trace-driven observability" (Charity Majors talks).

## Exercises / Self-Check

1. Set up a TracerProvider with OTLP gRPC exporter and `ParentBased(TraceIDRatioBased(0.1))`. Send 100 requests through a 3-service hop and confirm trace continuity in your backend.
2. Add `otelhttp.NewHandler` to your server and `otelhttp.NewTransport` to your client. Verify `traceparent` is sent + received.
3. Configure tail sampling at the Collector: 100% errors, 100% slow (>1s), 1% normal. Generate mixed traffic and verify the policy.
4. Write a custom propagator for an in-house RPC protocol. Test inject/extract round-trip.
5. Add OTel attributes to a span using only `semconv` constants. Confirm your vendor recognizes them.
6. Wire `otelslog` to bridge `log/slog` into OTel logs. Verify logs are correlated with their containing span in your backend UI.
7. Build a span-link example: a Kafka consumer that links a single batch.process span to N producer spans.
8. Benchmark span creation overhead with 0, 3, 10 attributes. Decide if your hot path can afford it.
