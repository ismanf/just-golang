# `log/slog` — Structured Logging

## TL;DR

`log/slog` (since 1.21) is the stdlib structured logger: leveled, attribute-based, with pluggable handlers (text, JSON, or your own). It replaces ad-hoc use of the old `log` package for any service-grade application. Construct attributes with `slog.String`, `slog.Int`, `slog.Any`; group with `slog.Group`; create child loggers with `.With(...)`. JSON handler is the production default — feed to ELK, Loki, CloudWatch, Datadog.

## Mental Model

```
slog.Logger
   ├─ Handler (text | json | custom)
   ├─ Default attributes (via .With)
   └─ Methods: Debug, Info, Warn, Error, Log (custom level)
                 │
                 ▼
            Record { Time, Level, Message, []Attr }
                 │
                 ▼
            Handler.Handle(ctx, Record)
                 │
                 ▼
            Bytes to Writer (or anywhere)

Attr = (Key, Value)
Value can be: String, Int, Float, Bool, Duration, Time, Any, Group, LogValuer
```

## Syntax & Basic Usage

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})
	logger := slog.New(h)
	slog.SetDefault(logger)

	slog.Info("user logged in", "user_id", 42, "method", "oauth")
	slog.Debug("debug detail", slog.Group("req", "method", "GET", "path", "/x"))
	// Output (JSON):
	// {"time":"...","level":"INFO","msg":"user logged in","user_id":42,"method":"oauth"}
	// {"time":"...","level":"DEBUG","msg":"debug detail","req":{"method":"GET","path":"/x"}}
}
```

## Deep Dive

### Handlers

- `slog.NewTextHandler(w, opts)` — human-readable `key=value` lines.
- `slog.NewJSONHandler(w, opts)` — newline-delimited JSON, suitable for log aggregators.
- Custom: implement `slog.Handler` interface for routing to OTel, Loki, etc.

### `HandlerOptions`

```go
opts := &slog.HandlerOptions{
	Level: slog.LevelDebug,
	AddSource: true,             // include file:line in output
	ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
		if a.Key == "password" { return slog.Attr{} } // drop
		return a
	},
}
```

`ReplaceAttr` is the redaction hook.

### Attribute constructors

```go
slog.String("key", "v")
slog.Int("n", 42)
slog.Int64("big", 1<<40)
slog.Float64("ratio", 0.5)
slog.Bool("ok", true)
slog.Duration("elapsed", time.Since(start))
slog.Time("at", time.Now())
slog.Any("err", err)             // for arbitrary types; falls back to reflection
slog.Group("req", "id", 1, "method", "GET")
```

Prefer the typed ones — they avoid reflection.

### Variadic attrs vs `slog.Attr`

```go
slog.Info("msg", "k", "v", "n", 1)    // alternating key-value pairs (any type)
slog.LogAttrs(ctx, slog.LevelInfo, "msg",
	slog.String("k", "v"),
	slog.Int("n", 1),
)                                       // typed; zero reflection
```

`LogAttrs` is the fast path.

### Child loggers via `With`

```go
reqLogger := logger.With("request_id", reqID)
reqLogger.Info("started")
reqLogger.Info("ended", "status", 200)
```

Both lines include `request_id`.

### Levels

Built-in: `LevelDebug` (-4), `LevelInfo` (0), `LevelWarn` (4), `LevelError` (8). Custom: any `slog.Level` (int).

```go
slog.Log(ctx, slog.Level(2), "between info and warn", ...)
```

### Dynamic level adjustment

```go
var level slog.LevelVar // initially LevelInfo
level.Set(slog.LevelDebug)
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: &level})
```

Pointer-to-`LevelVar` lets you change level at runtime (e.g., via SIGUSR1).

### Context-aware logging

```go
slog.InfoContext(ctx, "msg", "k", "v")
```

Handlers receive `ctx`; custom handlers can pull request IDs, trace IDs, tenant IDs from it (OTel integration commonly does this).

### `LogValuer` for lazy/structured values

```go
type User struct{ ID int; Name string }
func (u User) LogValue() slog.Value {
	return slog.GroupValue(slog.Int("id", u.ID), slog.String("name", u.Name))
}

slog.Info("login", "user", u)  // emits user={id=1 name=Ada}
```

Use case: hide PII at log-time without changing call sites.

### Replacing the `log` package

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)
log.Println("legacy") // routed through slog
```

`slog.SetDefault` also redirects the stdlib `log` to it.

## Standard Library Hooks

- `context.Context` — `InfoContext`/`ErrorContext` etc. for handlers that want it.
- `io.Writer` for handler output.
- `time.Time` for `Time` attrs; `time.Duration` for `Duration`.

## Real-World Patterns

### 1. Production JSON logging with redaction

```go
opts := &slog.HandlerOptions{
	Level: slog.LevelInfo,
	ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
		switch a.Key {
		case "password", "token", "authorization":
			return slog.String(a.Key, "[REDACTED]")
		}
		return a
	},
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
slog.SetDefault(logger)
```

### 2. Per-request logger via middleware

```go
func withLogger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		l := slog.Default().With(
			"request_id", reqID(r),
			"method", r.Method,
			"path", r.URL.Path,
		)
		ctx := context.WithValue(r.Context(), loggerKey{}, l)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func logFor(ctx context.Context) *slog.Logger {
	if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok { return l }
	return slog.Default()
}
```

Use case: every handler logs with the request_id automatically.

### 3. Dynamic level via signal

```go
var level slog.LevelVar
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: &level}))
slog.SetDefault(logger)

go func() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGUSR1)
	for range c {
		if level.Level() == slog.LevelInfo {
			level.Set(slog.LevelDebug)
		} else {
			level.Set(slog.LevelInfo)
		}
	}
}()
```

Use case: toggle debug logging in production without restart.

### 4. Custom handler that forwards to OTel

```go
type otelHandler struct{ inner slog.Handler }

func (h *otelHandler) Handle(ctx context.Context, r slog.Record) error {
	if span := trace.SpanFromContext(ctx); span.IsRecording() {
		// attach attrs to span
	}
	return h.inner.Handle(ctx, r)
}
// implement Enabled, WithAttrs, WithGroup similarly
```

### 5. Lazy expensive serialization

```go
type bigStruct struct{ /* 100 fields */ }
func (b bigStruct) LogValue() slog.Value {
	// computed only if the record is actually emitted
	return slog.StringValue(fmt.Sprintf("%d items", len(b.Items)))
}
slog.Debug("state", "data", bigStruct{...})
```

If the level is Info, the `LogValue` call doesn't happen — saves expensive formatting.

## Anti-Patterns & Gotchas

**Using `fmt.Sprintf` to build log messages.** Pre-renders attrs; loses structure.

**Passing `slog.Any` for typed values.** Use the typed constructors — faster, less reflection.

**Unbalanced key/value args.** `slog.Info("msg", "key")` (no value) is detected at runtime; produces `!BADKEY=key`.

**Logging sensitive data.** Use `ReplaceAttr` or `LogValuer` to redact.

**Synchronous logging in hot path without buffering.** Wrap the writer with `bufio.Writer` and flush periodically.

**Creating loggers per request without `With`.** Allocates new logger; use `With` to derive cheaply.

**Mixing `slog` levels with custom int levels** without documenting.

**Forgetting that `slog.Group` attrs render differently** by handler — JSON nests, text uses `g.k=v` notation.

**Using `slog.Default()` everywhere** instead of a per-context logger. Hard to add request_id later.

## Performance Notes

- `slog.LogAttrs(ctx, level, msg, attrs...)` is the fastest API (no variadic boxing).
- JSON handler is roughly 2× slower than text handler.
- Disabled levels short-circuit: `slog.Debug(...)` with Info level is ~ns.
- `With(...)` allocates a new logger with cached attrs; cheap.
- For ultra-high RPS, third-party loggers (`zerolog`, `zap`) are still faster than stdlib; the gap has narrowed.

## How Big Companies Use It

- **Kubernetes** has been migrating from `klog` to `slog` for newer subsystems.
- **HashiCorp Vault** moved to `slog` for audit logs (since 1.21).
- **Tailscale** uses `slog` with a custom JSON handler that prepends node/peer context.
- **Caddy** uses its own `zap`-backed logger but supports `slog` interop.
- **gopls** uses `slog` for diagnostics.

## Source Code References

Pinned to `go1.26`.

- `log/slog`: [`src/log/slog/`](https://github.com/golang/go/tree/master/src/log/slog).
- Default JSON handler: [`src/log/slog/json_handler.go`](https://github.com/golang/go/blob/master/src/log/slog/json_handler.go).
- `LogValuer`: [`src/log/slog/value.go`](https://github.com/golang/go/blob/master/src/log/slog/value.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/log/slog.
- Go blog, "Structured Logging with slog": https://go.dev/blog/slog.
- Proposal #56345: https://github.com/golang/go/issues/56345.
- Jon Calhoun, "Logging with slog tutorials": https://www.calhoun.io.

## Exercises / Self-Check

1. Build a JSON handler with `AddSource: true`. Show the `source` field.
2. Implement `LogValue` on a `User` type to render `id` and `email_hash` (not the email).
3. Add a middleware that puts a per-request `slog.Logger` in the context.
4. Toggle level dynamically with a SIGUSR1 handler.
5. Benchmark `slog.Info("msg", "k", "v")` vs `slog.LogAttrs(ctx, Info, "msg", slog.String("k", "v"))`.
