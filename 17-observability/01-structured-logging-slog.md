# Structured Logging — `log/slog`

## TL;DR

`log/slog` (Go 1.21+) is the standard library's official answer to structured logging — what users were reaching for `logrus`, `zerolog`, `zap`, and `uber-go/zap` for since 2015. Three core types: a **`Logger`** wraps a **`Handler`** that processes **`Record`s** (timestamp + level + message + attrs). Two built-in handlers (`TextHandler` for line-oriented dev logs, `JSONHandler` for ingest), one extensibility hook (`Handler` interface — implement it for OTel, Loki, Honeycomb, whatever). Three ergonomics that matter most: **`slog.With(...)`** for derived loggers that carry context (request ID, trace ID); **`LogAttrs`** (allocation-free fast path) vs `Info`/`Debug`/etc. (variadic, convenient, ~3x slower); and **`LogValuer`** for lazy/redacted attribute formatting. Go 1.26 makes `slog.Default()` the default logger across more stdlib packages, adds `slog.DiscardHandler`, lands the long-discussed `Handler.WithGroup` semantics fix, and ships `slog.SetLogLoggerLevel` for taming the `log` package bridge. The single biggest gotcha: **`slog.Info("msg", "key1", val1, "key2", val2)` is variadic-`any`-typed**, so every value escapes to the heap. For hot paths, use `slog.LogAttrs(ctx, level, msg, slog.String("k", v))`.

## Mental Model

```
   slog.Info("user logged in", "id", 42)
              │
              ▼
   ┌─────────────────┐
   │  Logger          │  ── slog.With("svc","auth") → child Logger
   │   .handler ──────┼──►──────────┐
   │   .ctx-attrs ────┼──►──────────┤
   └─────────────────┘              ▼
                          ┌─────────────────┐
                          │   Handler        │  (you can swap this)
                          │   - Enabled(lvl) │
                          │   - Handle(rec)  │
                          │   - WithAttrs    │
                          │   - WithGroup    │
                          └─────────────────┘
                                  │
                       ┌──────────┼──────────┐
                       ▼          ▼          ▼
                  TextHandler  JSONHandler  YourCustomHandler
                  (dev)        (prod)       (OTel/Loki/Honeycomb)
```

Two invariants:

1. **`Logger` is a thin handle.** It holds a `Handler` and zero or more "preformatted" attrs. Cheap to copy. Pass by value.
2. **`Handler` is where work happens.** Most extensibility is "wrap a handler with a decorator." The stdlib's `JSONHandler` is ~200 lines — short enough to read.

## Setup — The First Few Lines

```go
import (
    "log/slog"
    "os"
)

func main() {
    h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        AddSource: true,   // file:line — slightly expensive, but cheap relative to JSON marshal
    })
    slog.SetDefault(slog.New(h))

    slog.Info("starting", "version", "v1.4.2", "pid", os.Getpid())
}
```

Output:

```json
{"time":"2026-05-23T09:00:00Z","level":"INFO","source":{"function":"main.main","file":"/app/main.go","line":15},"msg":"starting","version":"v1.4.2","pid":12345}
```

For local dev:

```go
h := slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelDebug})
```

```
time=2026-05-23T09:00:00.000Z level=INFO msg=starting version=v1.4.2 pid=12345
```

## Levels

```go
const (
    LevelDebug Level = -4
    LevelInfo  Level = 0
    LevelWarn  Level = 4
    LevelError Level = 8
)
```

Levels are `int` — you can interpolate (`slog.Level(2)` is "info+notice"). Custom levels are first-class; many production systems use `LevelInfo + 4 = "NOTICE"` or `LevelError + 4 = "FATAL"`.

### Dynamic level

```go
var levelVar slog.LevelVar  // atomic — thread-safe
levelVar.Set(slog.LevelInfo)

h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: &levelVar})

// Later, from a /loglevel HTTP handler:
levelVar.Set(slog.LevelDebug)
```

A `LevelVar` is the recommended pattern for runtime-tunable log levels — no logger swap, no race.

## Attrs — The Allocation Boundary

```go
// Variadic any — convenient, slower (escapes to heap)
slog.Info("processed", "user", uid, "rows", n)

// LogAttrs — explicit attrs, no any-boxing, fast
slog.LogAttrs(ctx, slog.LevelInfo, "processed",
    slog.Int64("user", uid),
    slog.Int("rows", n),
)
```

Benchmark on a typical machine:

| Form | Allocs | ns/op |
|------|--------|-------|
| `slog.Info("msg", "k", v)` | 4–8 | ~700 |
| `slog.LogAttrs(...)` | 0–2 | ~250 |
| `JSONHandler` final write | (varies) | ~500 |

For per-request logs in a 100k-req/s service, prefer `LogAttrs`. For occasional startup/shutdown logs, the variadic form is fine.

### All attr constructors

```go
slog.String("name", "alice")
slog.Int("n", 42)
slog.Int64("ts", time.Now().Unix())
slog.Uint64("size", uint64(1<<32))
slog.Float64("f", 3.14)
slog.Bool("ok", true)
slog.Time("at", time.Now())          // formats per handler
slog.Duration("took", elapsed)
slog.Any("custom", someStruct)
slog.Group("http",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.Int("status", 200),
)
```

## Context — Derived Loggers

```go
log := slog.With(
    slog.String("svc", "billing"),
    slog.String("region", "us-east-1"),
)

log.Info("invoiced", "user", 42)
// → { "svc":"billing", "region":"us-east-1", "msg":"invoiced", "user":42 }
```

`With` returns a new `*Logger`. The attrs are *pre-formatted* by the handler (`WithAttrs` is the optimization hook), so deep `With` chains have no per-call overhead.

### Per-request logger via `context.Context`

```go
type loggerKey struct{}

func WithLogger(ctx context.Context, l *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey{}, l)
}
func FromContext(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok { return l }
    return slog.Default()
}

func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log := slog.With(
            "req_id", r.Header.Get("X-Request-ID"),
            "method", r.Method,
            "path", r.URL.Path,
        )
        r = r.WithContext(WithLogger(r.Context(), log))
        next.ServeHTTP(w, r)
    })
}

// In a handler:
func handler(w http.ResponseWriter, r *http.Request) {
    log := FromContext(r.Context())
    log.Info("query result", "rows", 17)
}
```

A contextless `slog.Default()` is a sign of a missed correlation opportunity in request-scoped code.

## Groups — Nested Structure

```go
slog.LogAttrs(ctx, slog.LevelInfo, "request",
    slog.Group("client",
        slog.String("ip", r.RemoteAddr),
        slog.String("ua", r.UserAgent()),
    ),
    slog.Group("server",
        slog.Int("status", 200),
        slog.Duration("dur", elapsed),
    ),
)
```

JSON output:

```json
{"msg":"request","client":{"ip":"1.2.3.4","ua":"curl/8"},"server":{"status":200,"dur":12000000}}
```

`Logger.WithGroup("g").Info(...)` nests *all* subsequent attrs under `"g"`. Useful for namespaced subsystem logs.

## LogValuer — Lazy Formatting & Redaction

```go
type User struct {
    ID    int64
    Email string
    Pass  string  // secret
}

// User.LogValue() called only if the log line is emitted
func (u User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Int64("id", u.ID),
        slog.String("email", u.Email),
        // Pass intentionally omitted
    )
}

slog.Info("login", "user", u)
// → { "msg":"login", "user":{"id":42,"email":"a@b"} }
```

Two benefits: **redaction** (sensitive fields can't accidentally appear); **laziness** (LogValue is only called if the handler's level admits the record — debug logs that aren't emitted cost ~ten ns).

### Redaction wrapper pattern

```go
type Secret string

func (s Secret) LogValue() slog.Value {
    return slog.StringValue("[REDACTED]")
}

type Config struct {
    User string
    Pass Secret
}

slog.Info("config", "cfg", Config{"alice", "hunter2"})
// → "cfg":{"User":"alice","Pass":"[REDACTED]"}
```

## Custom Handlers

The `Handler` interface:

```go
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

### Decorating an existing handler

```go
type ctxHandler struct{ slog.Handler }

func (h ctxHandler) Handle(ctx context.Context, r slog.Record) error {
    if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
        r.AddAttrs(
            slog.String("trace_id", span.SpanContext().TraceID().String()),
            slog.String("span_id", span.SpanContext().SpanID().String()),
        )
    }
    return h.Handler.Handle(ctx, r)
}

slog.SetDefault(slog.New(ctxHandler{slog.NewJSONHandler(os.Stdout, nil)}))
```

Every log now carries trace correlation, with no caller-side change.

### Multi-handler fanout

```go
type multi struct{ hs []slog.Handler }

func (m multi) Enabled(ctx context.Context, l slog.Level) bool {
    for _, h := range m.hs { if h.Enabled(ctx, l) { return true } }
    return false
}
func (m multi) Handle(ctx context.Context, r slog.Record) error {
    for _, h := range m.hs {
        if h.Enabled(ctx, r.Level) {
            // Each handler needs its own clone — Record is mutable
            if err := h.Handle(ctx, r.Clone()); err != nil { return err }
        }
    }
    return nil
}
func (m multi) WithAttrs(a []slog.Attr) slog.Handler {
    out := make([]slog.Handler, len(m.hs))
    for i, h := range m.hs { out[i] = h.WithAttrs(a) }
    return multi{out}
}
func (m multi) WithGroup(g string) slog.Handler { /* analogous */ }
```

Common use: write JSON to stdout for the log pipeline, and a more verbose Text format to a local file during dev.

## Sampling, Rate-Limiting, and Dedup

The stdlib doesn't ship them — but they're easy decorators:

```go
type rateLimited struct {
    slog.Handler
    sem *semaphore.Weighted
}

func (h rateLimited) Handle(ctx context.Context, r slog.Record) error {
    if !h.sem.TryAcquire(1) {
        return nil  // drop
    }
    defer h.sem.Release(1)
    return h.Handler.Handle(ctx, r)
}
```

For sampling, take every N-th `LevelDebug` record:

```go
type sampled struct {
    slog.Handler
    every int64
    count atomic.Int64
}
func (h *sampled) Handle(ctx context.Context, r slog.Record) error {
    if r.Level == slog.LevelDebug {
        if h.count.Add(1) % h.every != 0 { return nil }
    }
    return h.Handler.Handle(ctx, r)
}
```

## Integration With Older Code

```go
// log → slog bridge: legacy log.Printf goes through slog
slog.SetLogLoggerLevel(slog.LevelInfo)
log.Print("legacy message")
// → emitted via slog.Default() at LevelInfo

// slog → log bridge: stdlib log handler wraps slog (rare)
```

Go 1.26 makes the bridge bidirectional defaults more predictable; `SetLogLoggerLevel` is the right hook for old `log.Print` calls in dependencies.

## Anti-Patterns & Gotchas

**`fmt.Sprintf` inside log args.** Defeats structure: `slog.Info(fmt.Sprintf("user %d done", uid))` is no better than `log.Printf`. Use `slog.Info("user done", "uid", uid)`.

**Logging in tight loops.** Each call is ~250ns–1µs minimum. A million-call loop costs seconds. Sample, aggregate, or hoist out.

**Unbounded attr cardinality.** Logging `user_id` is fine; logging full request bodies turns your log pipeline into Pinterest.

**Mixing variadic forms in hot paths.** `slog.Info` allocates; `slog.LogAttrs` doesn't. Pick the right one.

**Calling `slog.Default()` mid-request without `WithContext`.** You lose trace correlation. Build a request-scoped logger in middleware and propagate.

**Handler that mutates `r.AddAttrs` and forwards.** Fine inside *your* `Handle`, but if you also wrap, the same `Record` may be handed to multiple downstream handlers. Clone before mutation if fanning out.

**Logging secrets/PII directly.** `slog.Info("login", "user", user)` where `user` has a `password` field — until you implement `LogValuer`, the password serializes. Use typed wrappers.

**`slog.Any` for everything.** It boxes to `interface{}` and routes through reflection in the handler. Prefer specific constructors.

**Forgetting context cancellation in async logging.** A buffered/async handler that flushes on close: if you call `os.Exit(0)` you skip the flush. Use `runtime.SetFinalizer` carefully or explicit shutdown.

**Sending logs over the network synchronously per call.** Disastrous tail-latency. Use a local file or stdout + sidecar (Fluentbit, Vector, OTel Collector) to ship.

**Setting `AddSource: true` in a 100k-req/s service without measuring.** Each `runtime.Callers` walk is ~hundreds of ns. Usually fine; sometimes the regression matters.

**Logging `time.Now()` again from inside an attr.** `slog.Record` already has the timestamp. Don't double-log.

**Treating `slog.Error` as "error log + return."** It does not return an error. Pattern: `slog.ErrorContext(ctx, "failed", "err", err); return err`.

## Performance Notes

(Approximate, modern x86, default `JSONHandler`, writing to `io.Discard`.)

| Operation | ns/op | Allocs |
|-----------|-------|--------|
| `slog.Info("msg")` no attrs | ~120 | 0 |
| `slog.Info("msg", "k", "v")` | ~700 | 4 |
| `slog.LogAttrs(...3 attrs)` | ~250 | 0 |
| `slog.With(...).Info(...)` | ~150 + ~700 (latter per call) | (varies) |
| `JSONHandler` 10 attrs to file | ~2–4µs | varies |
| `TextHandler` 10 attrs to file | ~1.5–3µs | varies |
| `Enabled(level)` short-circuit (disabled) | ~5 | 0 |

For a hot loop: branch on `slog.Default().Enabled(ctx, slog.LevelDebug)` before constructing expensive attrs.

```go
if log.Enabled(ctx, slog.LevelDebug) {
    log.DebugContext(ctx, "details", "expensive", computeExpensive())
}
```

## Production Configuration Cheat Sheet

```go
func setupLogger(env string) *slog.Logger {
    var lvl slog.LevelVar
    switch env {
    case "prod":  lvl.Set(slog.LevelInfo)
    case "dev":   lvl.Set(slog.LevelDebug)
    case "test":  lvl.Set(slog.LevelWarn)
    }

    opts := &slog.HandlerOptions{
        Level:     &lvl,
        AddSource: env != "prod",   // skip in prod for perf
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Redact known sensitive keys
            switch a.Key {
            case "password", "token", "authorization":
                return slog.String(a.Key, "[REDACTED]")
            }
            return a
        },
    }

    var h slog.Handler = slog.NewJSONHandler(os.Stdout, opts)
    if env == "dev" {
        h = slog.NewTextHandler(os.Stdout, opts)
    }
    h = withTraceContext(h)         // your OTel decorator
    return slog.New(h)
}
```

## How Big Companies Use It

- **Google** internal services moved to `log/slog`-shaped APIs years before stdlib slog existed; the proposal was led by Jonathan Amsterdam (Google).
- **Cloudflare** publishes `cloudflare/slog-multi` patterns and uses slog with custom OTel-aware handlers across their edge.
- **Datadog**'s `dd-trace-go` integrates with slog via `ddtrace/contrib/log/slog` for trace correlation.
- **Honeycomb** ships an OTel-aware slog handler in `honeycombio/otel-config-go`.
- **HashiCorp** (Consul, Vault, Nomad) historically used `hashicorp/go-hclog` — newer subprojects integrate slog where the dep is shippable.
- **Tailscale** uses a custom `tailscale.com/logger` (predates slog) but converging on slog-compatible patterns.
- **Grafana Loki**'s recommended Go client is now slog-based.
- **Kubernetes** uses `klog` (historical); slog adoption in newer subprojects is increasing.

## Source Code References

- `log/slog`: https://github.com/golang/go/tree/master/src/log/slog.
- The `Handler` interface: https://github.com/golang/go/blob/master/src/log/slog/handler.go.
- `JSONHandler`: https://github.com/golang/go/blob/master/src/log/slog/json_handler.go.
- `TextHandler`: https://github.com/golang/go/blob/master/src/log/slog/text_handler.go.
- Original proposal: https://github.com/golang/go/issues/56345.
- Community extensions: https://github.com/samber/slog-multi, https://github.com/samber/slog-formatter.
- OTel integration: https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/bridges/otelslog.

## Further Reading

- "log/slog official guide" (Jonathan Amsterdam, Go team): https://go.dev/blog/slog.
- Go 1.21 release notes (slog section): https://go.dev/doc/go1.21#slog.
- "Structured logging with slog" (Jonathan Amsterdam, GopherCon talk).
- "Performance of log/slog" (benchmark posts, Eli Bendersky).
- "From logrus to slog" — many engineering blog migration write-ups.
- "Best practices for logging in Go" (12-Factor, OWASP Logging cheat sheet).

## Exercises / Self-Check

1. Set up a `JSONHandler` that writes to stdout with `AddSource` enabled. Compare output against a `TextHandler` writing the same record.
2. Write a `LogValuer` wrapper around a struct containing a secret. Verify the secret never appears in JSON or text output.
3. Build a handler decorator that adds OTel trace_id/span_id to every record. Wire it into your HTTP server.
4. Benchmark `slog.Info("k", v)` vs `slog.LogAttrs(..., slog.String("k", v))`. Confirm the allocation difference.
5. Implement a `LevelVar`-controlled `/loglevel` HTTP endpoint that lets you flip from INFO to DEBUG without restart.
6. Decorate a handler with rate-limiting (semaphore) and verify under load that excess records are dropped, not blocked.
7. Build a multi-handler that sends INFO+ to stdout JSON and DEBUG+ to a local file in TextHandler format. Confirm both write independently.
8. Wire `slog.SetLogLoggerLevel(slog.LevelWarn)` and call `log.Print("legacy")`. Confirm the record is processed by slog.
