# `net/http` — Server Side

## TL;DR

`net/http` provides a production HTTP/1.1 and HTTP/2 server with `http.Server`, `http.Handler`, `http.ServeMux`, plus utility middleware via composition. The 1.22 router upgrade made `ServeMux` capable of method-and-pattern routing (`POST /users/{id}`) — eliminating most reasons to reach for a third-party router. Graceful shutdown via `Server.Shutdown(ctx)` is mandatory for production.

## Mental Model

```
http.Server
  ├─ Addr, Handler, TLSConfig
  ├─ ReadTimeout, WriteTimeout, IdleTimeout, ReadHeaderTimeout
  └─ Shutdown(ctx) — drain in-flight, refuse new

http.Handler { ServeHTTP(w, r) }
  ├─ http.HandlerFunc — function adapter
  └─ http.ServeMux — pattern→handler routing, 1.22 with methods + wildcards

Middleware = a function that takes a Handler and returns a Handler.

per-request: r.Context(); r.Body (io.ReadCloser); w.WriteHeader(code) then w.Write(body)
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /hello/{name}", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "hi, %s\n", r.PathValue("name"))
	})
	srv := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}
	_ = srv.ListenAndServe()
	// Output: <none — listens until killed>
}
```

## Deep Dive

### `http.ServeMux` 1.22 routing

```go
mux.HandleFunc("GET /", indexHandler)
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("DELETE /users/{id}", deleteUser)

// Wildcards:
mux.HandleFunc("GET /files/{path...}", serveFile) // multi-segment

// Host matching:
mux.HandleFunc("GET admin.example.com/", adminPage)
```

Precedence rules: longer/more-specific patterns win. Path values via `r.PathValue("id")`.

### `http.Handler` and middleware

```go
type Handler interface { ServeHTTP(http.ResponseWriter, *http.Request) }

func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		slog.Info("req", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
	})
}

mux := http.NewServeMux()
mux.HandleFunc("GET /", index)
handler := logging(mux) // wrap once
```

Compose middleware as `auth(logging(metrics(mux)))`.

### Graceful shutdown

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

srv := &http.Server{Addr: ":8080", Handler: mux}
errCh := make(chan error, 1)
go func() { errCh <- srv.ListenAndServe() }()

select {
case <-ctx.Done():
case err := <-errCh:
	return err
}

shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
return srv.Shutdown(shutdownCtx) // drains in-flight requests
```

### Timeouts (production essentials)

```go
srv := &http.Server{
	Addr:              ":8080",
	ReadHeaderTimeout: 10 * time.Second,  // CRITICAL — limits Slowloris
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      30 * time.Second,
	IdleTimeout:       120 * time.Second,
	MaxHeaderBytes:    1 << 20,
}
```

Without `ReadHeaderTimeout`, a malicious client can hold a connection open by trickling header bytes (Slowloris). Set this even if you set `ReadTimeout`.

### `http.ResponseWriter` interface

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
w.Write(body)
```

`WriteHeader` must be called before `Write`. Once `Write` is called, headers are flushed and cannot be modified. Setting status to 200 without calling `WriteHeader` is implicit.

Optional interfaces (type-assert to discover):

- `http.Flusher` — `Flush()` for streaming responses (SSE).
- `http.Hijacker` — `Hijack()` to take over the connection (WebSocket upgrades).
- `http.Pusher` — HTTP/2 server push (rarely used; pushed by browsers).
- `http.CloseNotifier` (deprecated; use `r.Context().Done()`).

### Streaming responses (Server-Sent Events)

```go
func sse(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	flusher, ok := w.(http.Flusher)
	if !ok { http.Error(w, "no flush", http.StatusInternalServerError); return }
	for i := 0; i < 10; i++ {
		select {
		case <-r.Context().Done(): return
		default:
		}
		fmt.Fprintf(w, "data: %d\n\n", i)
		flusher.Flush()
		time.Sleep(time.Second)
	}
}
```

### Request body

`r.Body` is an `io.ReadCloser`. Always:

- Close (handler frame's `defer r.Body.Close()` — also handled by server, but explicit is safer).
- Bound (`io.LimitReader(r.Body, maxBytes)` to prevent OOM).
- Parse incrementally (`json.NewDecoder(r.Body).Decode(&v)`).

### HTTPS

```go
srv.ListenAndServeTLS("cert.pem", "key.pem")
```

For ACME / Let's Encrypt: `golang.org/x/crypto/acme/autocert`.

### HTTP/2

Enabled automatically when serving TLS. For h2c (cleartext HTTP/2), wrap with `golang.org/x/net/http2/h2c`.

### Context

`r.Context()` is canceled when the client disconnects or `Shutdown` is in progress. Pass it to downstream calls so they cancel too.

## Standard Library Hooks

- `net` underlies HTTP server transport.
- `crypto/tls` for HTTPS.
- `log/slog` for structured request logging.
- `context.Context` propagation.
- `golang.org/x/net/http2` for low-level HTTP/2 server config; `/h2c` for cleartext h2.
- `golang.org/x/crypto/acme/autocert` for automated TLS certs.

## Real-World Patterns

### 1. JSON API endpoint with timeout

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
	defer cancel()

	var in CreateUser
	dec := json.NewDecoder(io.LimitReader(r.Body, 1<<20))
	dec.DisallowUnknownFields()
	if err := dec.Decode(&in); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest); return
	}
	u, err := svc.Create(ctx, in)
	if err != nil { httpErr(w, err); return }
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(u)
}
```

### 2. Middleware chain helper

```go
type Middleware func(http.Handler) http.Handler

func chain(h http.Handler, mw ...Middleware) http.Handler {
	for i := len(mw) - 1; i >= 0; i-- {
		h = mw[i](h)
	}
	return h
}

handler := chain(mux, recoverer, logging, requestID, cors)
```

### 3. Panic-recover middleware

```go
func recoverer(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rv := recover(); rv != nil {
				slog.Error("panic", "v", rv, "stack", string(debug.Stack()))
				http.Error(w, "internal error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

### 4. Static file serving with custom headers

```go
fs := http.FileServer(http.Dir("./public"))
mux.Handle("GET /static/", http.StripPrefix("/static/", fs))
```

For embedded files:

```go
//go:embed assets/*
var assets embed.FS
mux.Handle("GET /assets/", http.FileServer(http.FS(assets)))
```

### 5. Health and readiness endpoints

```go
mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
})
mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
	if !ready.Load() { http.Error(w, "not ready", http.StatusServiceUnavailable); return }
	w.WriteHeader(http.StatusOK)
})
```

Use case: Kubernetes liveness/readiness probes.

## Anti-Patterns & Gotchas

**Default `http.Server{}` in production.** No timeouts → Slowloris attack vector. Always set timeouts.

**Calling `w.WriteHeader` twice.** Logs a warning; only first call counts.

**Writing to `w` after the handler returns** (via a goroutine that outlives the request). The connection is gone.

**Ignoring `r.Body.Close()`.** Server closes for you, but explicit `defer r.Body.Close()` is required if you Hijack.

**Reading `r.Body` without bounding.** Memory exhaustion.

**Mutating `r.Header` and expecting the response to use it.** That's a request header; response uses `w.Header()`.

**Building a custom router when `ServeMux` 1.22 covers your needs.** Gin/Chi/Echo are useful, but stdlib is now competitive.

**Forgetting `Shutdown` on SIGTERM.** Active requests get dropped; Kubernetes deploys leave clients with 502s.

**Long-running handlers without checking `r.Context().Done()`.** Doesn't honor client disconnect.

**Returning sensitive errors directly to client.** Wrap and sanitize at the boundary.

**HTTP/2 + Hijack.** Hijack is not supported on h2 connections. Detect with `r.ProtoMajor == 2`.

## Performance Notes

- One goroutine per connection (HTTP/1.1) or per stream (HTTP/2). Cheap; scales to millions.
- Reuse a `bufio` writer in the response? `net/http` already does internally.
- For high RPS, profile allocations: header maps and request parsing dominate.
- `http.NewRequest` (client side) and request parsing both reuse buffer pools.
- `sync.Pool` for response objects is a common micro-optimization in frameworks; stdlib already does it for the connection state.

## How Big Companies Use It

- **Caddy** is built on `net/http` (with its own router); production reverse proxy.
- **Kubernetes** apiserver: stdlib mux for the core HTTP loop, custom routing on top.
- **HashiCorp Vault / Consul** use stdlib server with custom middleware.
- **Tailscale control plane** uses stdlib + 1.22 routing exclusively.
- **GitHub's Go services** standardize on stdlib `net/http` with internal middleware libraries.

## Source Code References

Pinned to `go1.26`.

- `net/http` server: [`src/net/http/server.go`](https://github.com/golang/go/blob/master/src/net/http/server.go).
- `ServeMux` 1.22 routing: [`src/net/http/pattern.go`](https://github.com/golang/go/blob/master/src/net/http/pattern.go), `routing_tree.go`.
- HTTP/2 internals: [`src/net/http/h2_bundle.go`](https://github.com/golang/go/blob/master/src/net/http/h2_bundle.go) (auto-generated from `golang.org/x/net/http2`).
- `Server.Shutdown`: same file.

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/net/http.
- Go 1.22 release notes (routing): https://go.dev/doc/go1.22.
- Filippo Valsorda's HTTP timeout post: https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/.
- Go blog, "HTTP/2 Server Push" (historical context).

## Exercises / Self-Check

1. Build a `GET /users/{id}` endpoint using stdlib 1.22 routing. Return JSON.
2. Add `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`. Show the response on a Slowloris-style attack.
3. Write graceful shutdown that drains in-flight requests with a 30-second budget.
4. Implement Server-Sent Events that emits events until client disconnects.
5. Why is `Hijack` not supported on HTTP/2? Look up the protocol design.
