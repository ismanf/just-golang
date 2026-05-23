# Functional Options — Rob Pike's Pattern

## TL;DR

The **functional options pattern** is Go's idiomatic way to handle optional, extensible constructor parameters. A constructor takes variadic `...Option` values, where each `Option` is a function `func(*T)` (or `func(*T) error`) that mutates the receiver. Coined by **Rob Pike** in his 2014 blog post "Self-referential functions and the design of options", the pattern is now standard across the Go stdlib (`grpc.NewServer`, `oteltrace.WithSampler`) and most well-designed Go libraries. The single biggest gotcha: **functional options are NOT cheap**. Each option is a closure that may allocate; on hot constructors (called per request), they can dominate. Use the pattern for *coarse* configuration (creating a server, a client, a parser); not for per-call parameters.

## Mental Model

```
   Without functional options (becomes unwieldy):
   
   srv := NewServer(addr, port, readTimeout, writeTimeout,
                    maxConns, tlsCfg, logger, ...)

   With functional options:
   
   srv := NewServer(addr,
       WithPort(8080),
       WithReadTimeout(30*time.Second),
       WithTLS(tlsCfg),
       WithLogger(logger),
   )

   Each option is a closure capturing its argument; called once
   inside NewServer to mutate the server struct.
```

The pattern's defining property: **adding a new option is a purely additive change** — existing callers don't break, and you don't pass `nil`/`0`/empty-string sentinels for parameters you don't care about.

## Syntax & Basic Usage

```go
package server

import (
	"crypto/tls"
	"net"
	"time"
)

type Server struct {
	addr         string
	readTimeout  time.Duration
	writeTimeout time.Duration
	tls          *tls.Config
}

// Option configures a Server.
type Option func(*Server)

func WithReadTimeout(d time.Duration) Option {
	return func(s *Server) { s.readTimeout = d }
}

func WithWriteTimeout(d time.Duration) Option {
	return func(s *Server) { s.writeTimeout = d }
}

func WithTLS(cfg *tls.Config) Option {
	return func(s *Server) { s.tls = cfg }
}

func NewServer(addr string, opts ...Option) *Server {
	s := &Server{
		addr:         addr,
		readTimeout:  30 * time.Second,  // default
		writeTimeout: 30 * time.Second,  // default
	}
	for _, opt := range opts {
		opt(s)
	}
	return s
}

func main() {
	_ = NewServer(":8080",
		WithReadTimeout(60*time.Second),
		WithTLS(&tls.Config{}),
	)
	_ = net.Listener(nil)
}
```

Caller writes only the options they care about. Defaults fill in the rest.

## Deep Dive

### Why not a config struct?

Alternative pattern:

```go
type ServerConfig struct {
    Addr         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    TLS          *tls.Config
}

func NewServer(cfg ServerConfig) *Server { /* ... */ }

// Usage:
srv := NewServer(ServerConfig{
    Addr:        ":8080",
    ReadTimeout: 60 * time.Second,
})
```

Trade-offs:

| Aspect | Functional options | Config struct |
|---|---|---|
| Adding a new field | additive, no caller breaks | additive but exposes zero-value |
| Default vs explicit zero | clear (option not passed) | ambiguous (was 0 set or default?) |
| Multiple-layer composition | natural | requires merge logic |
| Validation | inside each option | requires post-construction check |
| Discoverability | godoc lists each `WithX` | one struct |
| IDE autocomplete | shows options as you type | shows fields |
| Cost per call | one closure allocation per option | none |
| Order matters | sometimes (last `With` wins) | no |

Functional options win when:
- The API surface grows over time.
- "Set" vs "default" distinction matters.
- Cross-cutting options (logger, telemetry) appear across many types.

Config struct wins when:
- The constructor is in the hot path.
- You want all-or-nothing serialization (YAML→struct).
- Defaults are obvious from zero-values.

Many APIs use **both**: a `Config` struct *plus* `Option` functions that operate on it. gRPC does this.

### Option signatures: with or without error?

```go
type Option func(*Server)
type Option func(*Server) error
```

`error` lets options validate (`WithReadTimeout(-1*time.Second)` could return an error). Trade-off: `NewServer` must now return `error`, propagating complexity.

Practical advice:
- For options that can't fail (set a field, register a callback): use `func(*T)`.
- For options that can fail (parse a string, validate a range): use `func(*T) error`.
- If a single option in your API can fail, all should use the same signature; mixing is awkward.

### Mutually exclusive options

```go
// Only one of WithSyncMode or WithAsyncMode can be active.
func WithSyncMode() Option {
    return func(s *Server) {
        s.mode = ModeSync
    }
}

func WithAsyncMode() Option {
    return func(s *Server) {
        s.mode = ModeAsync
    }
}

// Document: "last one wins".
```

Last-write-wins is the conventional resolution. Document it explicitly.

### Optional options with state

An option that captures complex state:

```go
type rateLimit struct {
    qps   int
    burst int
}

func WithRateLimit(qps, burst int) Option {
    return func(s *Server) {
        s.limit = &rateLimit{qps: qps, burst: burst}
    }
}
```

The closure captures `qps` and `burst`. The Server gets a pointer to the new struct.

### Options that interact

Two options that conflict at validation time:

```go
func WithReadTimeout(d time.Duration) Option {
    return func(s *Server) {
        if s.idleTimeout > 0 && d > s.idleTimeout {
            // can't directly return error from func(*T)
            s.readTimeout = d // do it anyway; validate in NewServer
        } else {
            s.readTimeout = d
        }
    }
}
```

If validation crosses options, do it once after all options are applied:

```go
func NewServer(addr string, opts ...Option) (*Server, error) {
    s := &Server{addr: addr}
    for _, opt := range opts { opt(s) }
    if err := s.validate(); err != nil { return nil, err }
    return s, nil
}
```

### Generic options

Since Go 1.18:

```go
type Option[T any] func(*T)

func WithName[T interface{ SetName(string) }](name string) Option[T] {
    return func(t *T) { (*t).SetName(name) }
}
```

Rarely needed; concrete types usually suffice. Generic options are useful in library frameworks that operate on many different types.

### Interface-based options (rare)

```go
type Option interface {
    apply(*Server)
}

type readTimeoutOption time.Duration

func (o readTimeoutOption) apply(s *Server) {
    s.readTimeout = time.Duration(o)
}

func WithReadTimeout(d time.Duration) Option {
    return readTimeoutOption(d)
}
```

Heavier syntax; benefit is that options are *values* (printable, comparable). gRPC uses an interface-based variant for this reason — the framework introspects options.

### Hierarchical options

When an option's "type" is itself an option:

```go
type TLSOption func(*tls.Config)

func WithMinTLSVersion(v uint16) TLSOption {
    return func(c *tls.Config) { c.MinVersion = v }
}

func WithTLS(opts ...TLSOption) Option {
    return func(s *Server) {
        cfg := &tls.Config{}
        for _, opt := range opts { opt(cfg) }
        s.tls = cfg
    }
}

// Usage:
srv := NewServer(":443",
    WithTLS(WithMinTLSVersion(tls.VersionTLS13)),
)
```

Nested. Often overkill — flatten if you can.

### Allocation cost

Each `WithX(value)` call is a closure (a small heap allocation, ~16-32 bytes). For a constructor with 10 options, that's ~160-320 bytes of garbage per call.

For server/client construction (called once at startup), this is irrelevant.

For per-request constructors (e.g., creating a `parser` per HTTP request), this is real overhead. Either:
- Cache the parser per goroutine via `sync.Pool`.
- Use a config struct.
- Avoid the pattern in hot paths.

Benchmarking:

```go
func BenchmarkOptions(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = NewServer(":80",
            WithReadTimeout(time.Second),
            WithWriteTimeout(time.Second),
        )
    }
}
```

Typical: ~100 ns/op, ~3 allocs (one per option + the struct).

### Naming convention

The Go community standard:

- `WithFoo(foo Foo)`: set a value.
- `EnableFoo()`, `DisableFoo()`: toggle a bool.
- `AddFoo(foo Foo)`: append to a slice.
- `SetFoo(...)` is *not* canonical for options (it's for setters on existing objects).

Be consistent.

### Where the pattern shines in the stdlib & community

- `golang.org/x/exp/...`: many libraries.
- `google.golang.org/grpc.NewServer`, `grpc.Dial`.
- `go.opentelemetry.io/otel` — every component uses options.
- `database/sql.Open` (kind of — Conn options via `sql.Conn.Raw`).
- `crypto/tls.Config` is a struct (predates the pattern's spread); `crypto/tls.Dial` doesn't take options yet.
- HTTP middleware libraries (chi, gin) use the pattern for router options.

### When NOT to use it

- **Per-call**: `Save(opts ...SaveOption)` called in a tight loop allocates.
- **One- or two-parameter functions**: `NewBuffer(size int)` doesn't need options.
- **Internal types**: options are an API design tool; for unexported types, just use a struct.
- **Configuration loaded from YAML/JSON**: the struct is your source of truth.

## Standard Library Hooks

The stdlib doesn't have a "functional options" type — it's a pattern, not an API. But many packages follow it:

- `golang.org/x/oauth2`, `google.golang.org/grpc`, `go.opentelemetry.io/otel`.
- `github.com/spf13/cobra` for commands (`cobra.Command` fields, not options strictly).
- `github.com/jackc/pgx/v5`: `pgx.ConnConfig` is a struct, but its `WithFooHook` helpers are options-style.

## Real-World Patterns

### 1. HTTP client builder

```go
package httpcli

import (
	"net/http"
	"time"
)

type Client struct {
	hc      *http.Client
	timeout time.Duration
	retries int
}

type Option func(*Client)

func WithTimeout(d time.Duration) Option { return func(c *Client) { c.timeout = d } }
func WithRetries(n int) Option           { return func(c *Client) { c.retries = n } }
func WithTransport(t http.RoundTripper) Option {
	return func(c *Client) { c.hc.Transport = t }
}

func New(opts ...Option) *Client {
	c := &Client{
		hc:      &http.Client{Timeout: 30 * time.Second},
		retries: 3,
	}
	for _, opt := range opts { opt(c) }
	return c
}
```

### 2. Logger with structured options

```go
package logger

import "log/slog"

type Logger struct {
	level slog.Level
	attrs []slog.Attr
}

type Option func(*Logger)

func WithLevel(l slog.Level) Option {
	return func(lg *Logger) { lg.level = l }
}

func WithAttrs(attrs ...slog.Attr) Option {
	return func(lg *Logger) { lg.attrs = append(lg.attrs, attrs...) }
}

func New(opts ...Option) *Logger {
	lg := &Logger{level: slog.LevelInfo}
	for _, opt := range opts { opt(lg) }
	return lg
}
```

### 3. Database client with validation

```go
package db

import (
	"errors"
	"time"
)

type Client struct {
	dsn       string
	maxConns  int
	idleConns int
	timeout   time.Duration
}

type Option func(*Client) error

func WithMaxConns(n int) Option {
	return func(c *Client) error {
		if n <= 0 { return errors.New("maxConns must be positive") }
		c.maxConns = n
		return nil
	}
}

func WithIdleConns(n int) Option {
	return func(c *Client) error {
		if n < 0 { return errors.New("idleConns must be non-negative") }
		c.idleConns = n
		return nil
	}
}

func New(dsn string, opts ...Option) (*Client, error) {
	c := &Client{dsn: dsn, maxConns: 10, idleConns: 2, timeout: 30 * time.Second}
	for _, opt := range opts {
		if err := opt(c); err != nil { return nil, err }
	}
	if c.idleConns > c.maxConns {
		return nil, errors.New("idleConns cannot exceed maxConns")
	}
	return c, nil
}
```

Cross-option validation happens after all options are applied.

### 4. Hierarchical option groups

```go
package server

import "crypto/tls"

type Server struct {
	addr string
	tls  *tls.Config
}

type Option func(*Server)

type TLSOpt func(*tls.Config)

func MinTLS13() TLSOpt {
	return func(c *tls.Config) { c.MinVersion = tls.VersionTLS13 }
}

func WithCipherSuites(suites []uint16) TLSOpt {
	return func(c *tls.Config) { c.CipherSuites = suites }
}

func WithTLS(opts ...TLSOpt) Option {
	return func(s *Server) {
		s.tls = &tls.Config{}
		for _, opt := range opts { opt(s.tls) }
	}
}
```

### 5. Combine options for reuse

```go
func ProductionDefaults() []Option {
	return []Option{
		WithTimeout(30 * time.Second),
		WithRetries(5),
		WithMetrics(prom.DefaultRegistry),
	}
}

func main() {
	c := New(append(ProductionDefaults(), WithTimeout(60*time.Second))...)
	_ = c
}
```

Reusable option bundles; last `WithTimeout` wins.

## Anti-Patterns & Gotchas

**Using options for required parameters.** `addr` should be a regular argument; options are for *optional* config.

**Options that allocate large structures.** Each option is a closure. Hidden heap allocations.

**Returning errors from options when the constructor doesn't.** Mixing signatures forces awkward error handling.

**Order-dependent options that aren't documented.** Surprising behavior.

**Mutually exclusive options without clear "last wins" semantics.** Document precisely.

**Recursive option construction.** `WithOptions(opts ...Option) Option` — fine for combinator-style, but make the precedence rules explicit.

**Stuffing every parameter through options.** A constructor with 30 options is harder to use than a struct.

**Allocating in a hot loop.** `NewParser(WithStrict(true))` called per HTTP request → real GC pressure. Use a config struct or sync.Pool.

**Options that mutate global state.** They should mutate the receiver only.

**Forgetting to apply options in the constructor.** A constructor that ignores `opts...` silently breaks.

**Generic options before generics existed.** `interface{}`-typed options force type assertions; ugly. Use concrete types.

**Documenting only `func WithFoo(Foo)` without explaining its effect.** Each option should have a godoc explaining what it changes and the default.

## Performance Notes

- One option: ~16-32 bytes heap (the closure).
- 10-option constructor call: ~100 ns + ~250 bytes.
- Hot-path constructor calls: avoid the pattern.
- Server / client startup: cost is invisible.
- PGO and inlining can sometimes eliminate option allocations if the constructor inlines and the options are statically known.

## How Big Companies Use It

- **The Go team's gRPC package**: every option is `func(*serverOptions)`-style. https://github.com/grpc/grpc-go.
- **OpenTelemetry Go**: `oteltrace.NewTracerProvider(oteltrace.WithSampler(...), oteltrace.WithBatcher(...))`. https://github.com/open-telemetry/opentelemetry-go.
- **Uber's zap** (logging): `zap.NewProduction(zap.AddCaller(), zap.AddStacktrace(zap.ErrorLevel))`. https://github.com/uber-go/zap.
- **HashiCorp Vault** plugin SDK: options for every storage backend.
- **CockroachDB**: many internal modules.
- **Kubernetes client-go**: option-style constructors throughout.

## Source Code References

The pattern itself isn't in the standard library. References to implementations:

- gRPC Go server options: [`grpc-go/server.go`](https://github.com/grpc/grpc-go/blob/master/server.go) — search `ServerOption`.
- OpenTelemetry tracer options: [`opentelemetry-go/sdk/trace/`](https://github.com/open-telemetry/opentelemetry-go/tree/main/sdk/trace).
- Uber zap options: [`uber-go/zap/options.go`](https://github.com/uber-go/zap/blob/master/options.go).
- Caddy module options: [`caddy/v2/caddyconfig/`](https://github.com/caddyserver/caddy).

## Further Reading

- Rob Pike, "Self-referential functions and the design of options" (2014): https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html.
- Dave Cheney, "Functional options for friendly APIs": https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis.
- Kelsey Hightower, "Functional options pattern in Go": various talks.
- Uber Go style guide: https://github.com/uber-go/guide.
- Bryan Mills, "API design — options vs structs" — Gophers Slack archives.
- Effective Go — Functions (general): https://go.dev/doc/effective_go#functions.

## Exercises / Self-Check

1. Write a `NewCache` constructor with options for `MaxEntries`, `TTL`, `Eviction`. Add a new option `WithMetrics(prom.Registry)` without breaking callers.
2. Convert a Server constructor with 15 parameters into the functional options pattern. Compare ergonomics.
3. Benchmark `NewClient(WithA, WithB, WithC)` vs `NewClientCfg(Config{...})`. How many allocations differ?
4. Implement options that can return errors (validation). How does the constructor's signature change?
5. Why does gRPC use an interface-based option pattern (`type ServerOption interface { apply(...) }`) rather than plain `func(*Server)`? What does it enable?
