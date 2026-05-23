# HTTP Server Patterns — Routing, Middleware, Graceful Shutdown

## TL;DR

`net/http` is one of Go's flagship packages — a production-grade HTTP/1.1 + HTTP/2 server in the standard library, no framework required. Three pieces compose the entire model: a **`http.Handler`** interface (`ServeHTTP(w, r)`), a **`*http.ServeMux`** for routing (Go 1.22+ supports method+path patterns natively, ending most reasons to reach for `chi`/`gorilla/mux`), and a **`*http.Server`** that orchestrates listeners, timeouts, TLS, and graceful shutdown. The mental shift since Go 1.22 is that the stdlib mux is now genuinely competitive — you get `GET /users/{id}` and path parameters via `r.PathValue("id")`. Middleware in Go is just function composition (no special framework primitive), and graceful shutdown is `srv.Shutdown(ctx)` with an outer SIGTERM listener. Go 1.26 tightens defaults: `Server.ReadHeaderTimeout` defaults to a non-zero value, `MaxHeaderBytes` is enforced more strictly, and the new `Server.ListenAndServeTLS` honours OS-level `SO_REUSEPORT` when available via a new `ListenConfig` field. The single biggest gotcha that still haunts production: **zero `WriteTimeout` means infinite write time** — a single slow-loris client can hold a goroutine indefinitely until you set it.

## Mental Model

```
   Listener (TCP)                 *http.Server
       │                          ─────────────
       │                          ReadHeaderTimeout
       │                          ReadTimeout
       │                          WriteTimeout
       │                          IdleTimeout
       │                          Handler ───► ServeMux
       │                                            │
       ▼                                            ▼
   accept ──► Serve(rw, req) ──► middleware ──► routed Handler
                                  chain
```

Two invariants worth tattooing:

1. **One goroutine per request.** Cheap (~8 KB stack), but unbounded by default — set `Server.MaxConnsPerHost`-equivalent limits at the LB or via a semaphore.
2. **`http.Handler` is `func(ResponseWriter, *Request)`.** Everything else — middleware, routing, frameworks — is composition on top of that one interface.

## Minimal Server

```go
package main

import (
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/plain")
        w.Write([]byte("hello\n"))
    })

    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

Three lines of routing, zero dependencies, HTTP/1.1 + HTTP/2 over h2c if you want it. This is the baseline.

## Production Server Skeleton

```go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    registerRoutes(mux)

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           withMiddleware(mux),
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       30 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
        MaxHeaderBytes:    1 << 20,
        ErrorLog:          slog.NewLogLogger(slog.Default().Handler(), slog.LevelError),
    }

    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        slog.Info("listening", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("server failed", "err", err)
            os.Exit(1)
        }
    }()

    <-ctx.Done()
    slog.Info("shutting down")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("graceful shutdown failed", "err", err)
        srv.Close()
    }
}
```

What this gives you:

- Bounded timeouts (no slow-loris).
- SIGINT/SIGTERM-aware shutdown.
- `slog` for error logs (so panic stacks etc. land in structured output).
- A 20s grace window for in-flight requests during deploys.
- `http.ErrServerClosed` ignored on shutdown (it's normal).

## Routing (1.22+)

Go 1.22 added method matching, path parameters, and wildcards to the standard `ServeMux`. This is the routing API now:

```go
mux.HandleFunc("GET /users/{id}",        getUser)
mux.HandleFunc("POST /users",            createUser)
mux.HandleFunc("DELETE /users/{id}",     deleteUser)

mux.HandleFunc("GET /files/{path...}",   serveFile)   // greedy wildcard

mux.HandleFunc("/", notFound)                          // anything else
```

Path values:

```go
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
}
```

Precedence rules (Go 1.22):

- More specific patterns win (`GET /users/me` beats `GET /users/{id}`).
- Method-bearing patterns beat method-less ones.
- The mux *rejects* registration if there's a conflict it can't resolve (compile-time-ish safety).

### When you outgrow stdlib mux

You probably won't. But reasons to reach for `chi` or `echo` or `gin`:

- Sub-routers with prefix-bound middleware.
- Pattern parameter constraints (regex per segment).
- Per-route OpenAPI annotations.
- "Trie performance" — rarely matters; stdlib mux is fast enough for 100k req/s.

The frameworks themselves are covered in `07-frameworks.md`.

## Middleware — Just Function Composition

```go
type Middleware func(http.Handler) http.Handler

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rec := &statusRecorder{ResponseWriter: w, status: 200}
        next.ServeHTTP(rec, r)
        slog.InfoContext(r.Context(), "request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", rec.status,
            "dur", time.Since(start),
        )
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.ErrorContext(r.Context(), "panic",
                    "err", rec,
                    "stack", string(debug.Stack()),
                )
                http.Error(w, "internal server error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

func withMiddleware(h http.Handler) http.Handler {
    return chain(h,
        recoveryMiddleware,   // outermost — catches everything
        loggingMiddleware,
        timeoutMiddleware,
        cors,
        auth,
    )
}
```

Order matters:

- **Recovery outermost** — catches panics from every other middleware.
- **Logging next** — even auth failures get logged.
- **Timeout/cancellation** — before any handler-specific work.
- **CORS** — before auth so preflights aren't blocked.
- **Auth** — last general gate; per-route mw can layer on.

### A note on `statusRecorder`

```go
type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(c int) {
    r.status = c
    r.ResponseWriter.WriteHeader(c)
}
```

The default `ResponseWriter` doesn't expose the status code post-write. Wrap to record. If you also need body size, count bytes in `Write`.

### Hijacking, flushing, pushing — wrap carefully

If your wrapper hides `http.Hijacker`, `http.Flusher`, or `http.Pusher`, WebSocket upgrades and SSE break. Use the **`http.NewResponseController`** approach (Go 1.20+):

```go
func handler(w http.ResponseWriter, r *http.Request) {
    rc := http.NewResponseController(w)
    if err := rc.SetWriteDeadline(time.Now().Add(30*time.Second)); err != nil {
        // wrapper doesn't expose it; degrade gracefully
    }
    rc.Flush()
}
```

`ResponseController` traverses wrapped writers using the `Unwrap() http.ResponseWriter` method that your wrappers should implement.

## Timeouts — The Big One

```go
srv := &http.Server{
    ReadHeaderTimeout: 5 * time.Second,    // attacker holding headers? cut.
    ReadTimeout:       30 * time.Second,   // whole request, including body
    WriteTimeout:      30 * time.Second,   // whole response
    IdleTimeout:       120 * time.Second,  // keep-alive idle
}
```

Pre-1.26: `WriteTimeout = 0` meant infinite. Post-1.26: defaults to a finite value on `ListenAndServe`. You can still set `0` explicitly for streaming endpoints (SSE, long-polling), but do so per-handler via `http.NewResponseController.SetWriteDeadline`.

### Per-handler timeout vs server timeout

`http.TimeoutHandler` wraps a handler with a deadline; on timeout, it writes a 503 (configurable). It doesn't cancel the goroutine — that's your handler's responsibility:

```go
mux.Handle("GET /slow",
    http.TimeoutHandler(slowHandler, 5*time.Second, "request timed out\n"))

func slowHandler(w http.ResponseWriter, r *http.Request) {
    // Always check ctx
    select {
    case <-r.Context().Done():
        return
    case <-time.After(10 * time.Second):
        w.Write([]byte("done"))
    }
}
```

The `*http.Request.Context()` is canceled when the client disconnects or the server times out — propagate it into every downstream call (DB, HTTP client, RPC).

## Graceful Shutdown

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

// ... server starts in goroutine ...

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

`Shutdown` stops accepting new connections, waits for in-flight to finish, returns when either all are done or `ctx` expires. The grace window should match your LB's connection draining (often 15-30s).

### Two-phase shutdown for K8s

In Kubernetes, the recommended sequence is:

1. **Pod gets SIGTERM** + removed from Service endpoints.
2. **App switches readiness probe to fail**, so LB stops sending new requests.
3. **Sleep briefly** to let in-flight LB connection-draining complete.
4. **Call `Shutdown`** to drain server-side.

```go
<-ctx.Done()
readiness.SetReady(false)             // your readiness flag
time.Sleep(5 * time.Second)            // drain LB
srv.Shutdown(shutdownCtx)
```

Without the readiness flip, you race the LB and 5xx the last few requests.

### Connection draining vs goroutine leaks

`Shutdown` waits for in-flight HTTP requests but **doesn't cancel them**. A handler stuck in `time.Sleep(1 hour)` will keep the shutdown blocked. Always propagate `r.Context()` and respond to cancellation.

## Streaming Responses (SSE, long-polling)

```go
func sse(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    rc := http.NewResponseController(w)
    rc.SetWriteDeadline(time.Time{})   // no write timeout for streaming

    for {
        select {
        case <-r.Context().Done():
            return
        case msg := <-source:
            fmt.Fprintf(w, "data: %s\n\n", msg)
            if err := rc.Flush(); err != nil {
                return
            }
        }
    }
}
```

Flush is mandatory — without it, output sits in the Transport buffer.

## h2c — HTTP/2 Without TLS

Default `http.ListenAndServe` is HTTP/1.1 (and HTTP/2 over TLS via `ListenAndServeTLS`). For HTTP/2 over plaintext (rare but useful inside a mesh):

```go
import "golang.org/x/net/http2"
import "golang.org/x/net/http2/h2c"

h2s := &http2.Server{}
srv := &http.Server{
    Addr:    ":8080",
    Handler: h2c.NewHandler(myHandler, h2s),
}
```

Mostly used behind Envoy/Linkerd where TLS terminates at the mesh sidecar.

## Anti-Patterns & Gotchas

**Zero `WriteTimeout`.** Pre-1.26 default; one slow-loris holds a goroutine forever.

**`http.HandleFunc` on the global `http.DefaultServeMux`.** Library code or transitive dependencies can pollute it. Always use your own `mux := http.NewServeMux()`.

**Hand-rolled context propagation.** Use `r.Context()` everywhere. Never `context.Background()` inside a handler — you sever the cancellation chain.

**Forgetting to close `r.Body`.** The server handles this for you. The bug is on the *client* side. (Covered in `02-http-client-patterns.md`.)

**Reading request body without `MaxBytesReader`.** A client can POST 100 GB. Always:
```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)  // 1 MB cap
```

**Returning 200 with an empty body for errors.** Always set status before write. Always write a reason.

**Using `http.Error` for arbitrary text + Content-Type that conflicts.** `http.Error` writes `text/plain; charset=utf-8`. Don't override after the fact.

**Panicking instead of returning errors.** Panics are for unrecoverable bugs. Validation errors are 4xx with bodies.

**Calling `WriteHeader` more than once.** Logged as a warning, second call ignored. Often happens when middleware writes then handler also writes.

**Concurrent writes to one `ResponseWriter`.** Not safe. Serialize.

**Locking `r.Body` reads across goroutines.** Body is a stream; one reader.

**Forgetting CORS preflight.** Browsers send `OPTIONS` with `Access-Control-Request-*`. Your CORS middleware must handle the preflight before auth runs.

**Logging full request bodies.** PII leak; log size + content-type.

**Returning `r.RemoteAddr` as the client IP behind a reverse proxy.** Use `X-Forwarded-For` or `X-Real-IP` only if you trust the proxy; otherwise spoofable.

**Calling `srv.Close()` instead of `Shutdown`.** Forces immediate disconnect of in-flight requests; clients see TCP resets.

**Holding mutable state in package globals.** Race-prone. Bind to a struct (`type App struct { db *sql.DB; ... }`) and attach handlers as methods.

## Performance Notes

- **Per-request goroutine** stack: ~2-8 KB initial; grows on demand.
- **Empty handler throughput**: ~200–400k req/s on a modern core; bottlenecked by epoll + syscalls.
- **Routing overhead** (stdlib mux 1.22+): ~100-300 ns per match.
- **Middleware chain** of 5 frames: ~100-200 ns extra.
- **`slog.LogAttrs` per request**: ~250 ns.
- **`http.MaxBytesReader`**: free until you read past the cap.

The stdlib server is rarely the bottleneck. The DB or downstream service is.

## How Big Companies Use It

- **Caddy** (entirely Go) uses `net/http` with a custom listener wrapper for TLS multiplexing.
- **Cloudflare** runs `net/http` at the edge for many internal services; for the highest-throughput tier they tune `Transport`, accept-loop concurrency, and run multiple processes behind `SO_REUSEPORT`.
- **Kubernetes** uses `net/http` for `kube-apiserver` — extended with auth/audit middleware but the bones are stdlib.
- **Tailscale**'s `tsnet` exposes a `net/http`-compatible listener inside a Tailnet — pure stdlib API.
- **Grafana** ships a custom `httpserver` package on top of `net/http`.
- **Mattermost** uses stdlib `net/http` + a chi-style mux.
- **GitHub** runs Go HTTP services on stdlib; chi for the routing in newer services.

## Source Code References

- `net/http` server: https://github.com/golang/go/blob/master/src/net/http/server.go.
- `net/http` ServeMux: https://github.com/golang/go/blob/master/src/net/http/mux.go.
- 1.22 mux design: https://go.dev/blog/routing-enhancements.
- `http/httputil` (reverse proxy, dump request): https://github.com/golang/go/tree/master/src/net/http/httputil.
- `http.ResponseController`: https://github.com/golang/go/blob/master/src/net/http/responsecontroller.go.
- chi (popular drop-in router): https://github.com/go-chi/chi.

## Further Reading

- Filippo Valsorda, "So you want to expose Go on the Internet": https://blog.cloudflare.com/exposing-go-on-the-internet/.
- "Go 1.22 routing enhancements": https://go.dev/blog/routing-enhancements.
- "The complete guide to Go net/http timeouts" (Cloudflare blog).
- "Graceful shutdown of HTTP servers in Go" (various Medium posts; pattern is everywhere).
- "Hello, HTTP/2 and h2c" (Brad Fitzpatrick talk).
- "Go: Best practices for production environments" (Peter Bourgon).

## Exercises / Self-Check

1. Build a server with 1.22-style routing (`GET /users/{id}`). Confirm path parameters via `r.PathValue`.
2. Wire recovery + logging + timeout middleware. Trigger a panic in a handler and confirm the recovery middleware logs and responds 500.
3. Configure `ReadHeaderTimeout: 5*time.Second`. Use `nc` to open a connection and send headers slowly; confirm the server closes after 5s.
4. Implement graceful shutdown with 20s window. Send a long-running request; SIGTERM the server; confirm the request completes before exit.
5. Build an SSE endpoint. Use `http.NewResponseController.Flush` to push events. Verify with `curl -N`.
6. Wrap `ResponseWriter` to record status and bytes. Implement `Unwrap()` so `http.NewResponseController` still works.
7. Cap request bodies with `http.MaxBytesReader(w, r.Body, 1<<20)`. Send a 10 MB body; confirm 413 / read error.
8. Run two Go servers on the same port via `SO_REUSEPORT` (using `net.ListenConfig.Control`). Confirm both accept connections.
