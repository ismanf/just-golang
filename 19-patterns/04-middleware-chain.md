# Middleware Chain Pattern

## TL;DR

A **middleware** is a function that wraps a handler, adding behavior **before** and/or **after** the wrapped handler runs. In Go's HTTP world, the canonical signature is `func(http.Handler) http.Handler` — given an inner handler, return a new handler that intercepts requests. Chaining means composing many middlewares: `Chain(loggingMW, authMW, rateLimitMW)(yourHandler)`. The stdlib has no built-in chain helper; routers (chi, gin, echo) provide one, or you write a trivial one in ~10 lines. The single biggest gotcha: **middleware order matters and is non-obvious**. The first middleware in your chain is the *outermost* (runs first on entry, last on exit). Misordering recovery-from-panic, logging, and auth will produce hard-to-debug behavior.

## Mental Model

```
   Request → MW1 → MW2 → MW3 → Handler → MW3 → MW2 → MW1 → Response
             (entry side)              (exit side)

   Each middleware wraps the next. In code:
   
   Chain(MW1, MW2, MW3, handler) ==
       MW1(MW2(MW3(handler)))
   
   When a request comes in:
       MW1.ServeHTTP — does pre-work, calls inner
         MW2.ServeHTTP — does pre-work, calls inner
           MW3.ServeHTTP — does pre-work, calls inner
             handler.ServeHTTP — handles request
           MW3 post-work
         MW2 post-work
       MW1 post-work
```

The function-composition shape is what makes the pattern elegant: each middleware sees the inner as opaque; composition is associative.

## Syntax & Basic Usage

```go
package main

import (
	"log/slog"
	"net/http"
	"time"
)

// Middleware is a function that wraps an http.Handler.
type Middleware func(http.Handler) http.Handler

func LoggingMW(log *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			next.ServeHTTP(w, r)
			log.Info("req", "method", r.Method, "path", r.URL.Path, "dur", time.Since(start))
		})
	}
}

func RecoverMW(log *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if rec := recover(); rec != nil {
					log.Error("panic", "panic", rec)
					http.Error(w, "internal error", 500)
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// Chain composes middlewares in the order given (first one is outermost).
func Chain(mws ...Middleware) Middleware {
	return func(final http.Handler) http.Handler {
		for i := len(mws) - 1; i >= 0; i-- {
			final = mws[i](final)
		}
		return final
	}
}

func main() {
	log := slog.Default()
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("hi"))
	})
	chain := Chain(RecoverMW(log), LoggingMW(log))
	http.ListenAndServe(":8080", chain(mux))
}
```

`Chain(A, B)(h)` means A wraps B wraps h. A runs first/last; B runs second/second-to-last.

## Deep Dive

### The canonical signature

```go
type Middleware func(http.Handler) http.Handler
```

A middleware receives the next handler in the chain and returns a wrapping handler. This is **decorator** in GoF terms.

Variants exist:
- `func(http.HandlerFunc) http.HandlerFunc`: same but on the HandlerFunc type.
- `func(handler) handler` for custom (non-http) types.

`http.Handler` is just `ServeHTTP(http.ResponseWriter, *http.Request)`. Anything implementing that fits.

### Why "outermost first" is the convention

Reading `Chain(A, B, C)`:

```
Chain(A, B, C) = A(B(C(handler)))
```

A's `ServeHTTP` runs first on entry. A is "outermost". This matches Express.js (Node), Rack (Ruby), WSGI (Python) — the convention is universal.

Implementation: build the chain right-to-left, so the *last* middleware wraps the handler first; A wraps last:

```go
func Chain(mws ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        for i := len(mws) - 1; i >= 0; i-- {
            final = mws[i](final)
        }
        return final
    }
}
```

### Pre-work vs post-work

```go
func MyMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ---- pre-work (before inner) ----
        next.ServeHTTP(w, r)
        // ---- post-work (after inner) ----
    })
}
```

Pre-work: logging start, auth check, rate-limit check.
Post-work: logging end, metric emission, response transformation.

Use `defer` for post-work that must run even on panic:

```go
func MyMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer postWork()
        preWork()
        next.ServeHTTP(w, r)
    })
}
```

### Short-circuiting

A middleware can refuse to call `next.ServeHTTP` and return a response itself:

```go
func AuthMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", 401)
            return  // don't call next
        }
        next.ServeHTTP(w, r)
    })
}
```

Auth, rate limit, CORS, conditional fetches — all short-circuit when needed.

### Modifying request / response

```go
func RequestIDMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-Id")
        if id == "" {
            id = uuid.NewString()
        }
        // Carry through context
        ctx := context.WithValue(r.Context(), reqIDKey{}, id)
        r = r.WithContext(ctx)
        w.Header().Set("X-Request-Id", id)
        next.ServeHTTP(w, r)
    })
}

type reqIDKey struct{}
func GetRequestID(ctx context.Context) string {
    v, _ := ctx.Value(reqIDKey{}).(string)
    return v
}
```

`r.WithContext(ctx)` returns a *new* `*http.Request` with extended context. The middleware passes that to `next`.

### Wrapping ResponseWriter

To inspect the response (status code, byte count) you must wrap the writer:

```go
type statusWriter struct {
    http.ResponseWriter
    status int
    size   int
}

func (sw *statusWriter) WriteHeader(code int) {
    sw.status = code
    sw.ResponseWriter.WriteHeader(code)
}

func (sw *statusWriter) Write(b []byte) (int, error) {
    if sw.status == 0 { sw.status = 200 }
    n, err := sw.ResponseWriter.Write(b)
    sw.size += n
    return n, err
}
```

Now `LoggingMW` can log `sw.status` and `sw.size`.

**Caveat**: wrapping ResponseWriter loses any optional interfaces (`http.Hijacker`, `http.Flusher`, `http.Pusher`). Either ignore them or implement passthroughs. `go-chi/chi/middleware` has helpers.

### Per-route middlewares

Many routers let you attach middlewares per route or per group:

```go
// chi
r := chi.NewRouter()
r.Use(RecoverMW, LoggingMW)

r.Group(func(r chi.Router) {
    r.Use(AuthMW)
    r.Get("/api/profile", profileHandler)
})

r.Get("/public", publicHandler)
```

`/api/profile` runs Recover + Logging + Auth. `/public` runs only Recover + Logging.

### Standard order

A widely-used HTTP middleware order, outermost-first:

1. **Recover** (catch panics).
2. **Request ID / correlation**.
3. **Logging** (entry + exit time).
4. **Tracing / metrics** (span start).
5. **Rate limit / circuit breaker**.
6. **Authentication**.
7. **Authorization**.
8. **CORS / CSP / security headers**.
9. **Compression**.
10. **Validation** (per-handler typically).
11. **Handler**.

Recover at top so it catches every panic. Logging early to log even failed requests.

### Middleware vs decorator

Functionally equivalent; "middleware" is HTTP-specific terminology. The pattern works for any handler shape:

```go
type RPCHandler func(ctx context.Context, req Request) (Response, error)
type RPCMiddleware func(RPCHandler) RPCHandler

func RPCRecoverMW(next RPCHandler) RPCHandler {
    return func(ctx context.Context, req Request) (resp Response, err error) {
        defer func() {
            if rec := recover(); rec != nil {
                err = fmt.Errorf("panic: %v", rec)
            }
        }()
        return next(ctx, req)
    }
}
```

gRPC's interceptors are middleware for gRPC handlers.

### Generic middleware (1.18+)

```go
type Handler[Req, Resp any] func(ctx context.Context, req Req) (Resp, error)
type Middleware[Req, Resp any] func(Handler[Req, Resp]) Handler[Req, Resp]
```

Used in some modern Go libraries to give type-safe RPC middleware.

### Middleware that's actually NOT a middleware

Some people call **interceptors** or **hooks** middleware. They have different signatures:

- gRPC unary interceptor: `func(ctx, req, info, handler) (resp, err)`.
- gRPC stream interceptor: different signature.
- Functional callbacks (`OnRequest`, `OnResponse`): more event-driven.

Conceptually similar; not interchangeable code-wise.

### Pitfalls of inflexible chain helpers

A "smart" chain helper:

```go
chain := Chain(A, B, C)(handler)
```

But what if you want **conditional middleware**?

```go
mws := []Middleware{Recover, Logging}
if authRequired {
    mws = append(mws, Auth)
}
chain := Chain(mws...)(handler)
```

This works because `Chain` accepts variadic. Many homegrown chains hardcode 3 or 5 — limiting.

### Per-request middleware state

Middlewares are reused across all requests. **Never** stash request-specific state in middleware-level vars; use the request context.

```go
// WRONG: shared state
var lastUser string
func MW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        lastUser = r.Header.Get("X-User")  // RACE
        next.ServeHTTP(w, r)
    })
}

// RIGHT: context
func MW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.WithValue(r.Context(), userKey{}, r.Header.Get("X-User"))
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Testing middlewares

Each middleware is a function; testable in isolation:

```go
func TestRecoverMW(t *testing.T) {
    var logged bool
    log := slog.New(slog.NewTextHandler(io.Discard, nil))
    h := RecoverMW(log)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        panic("boom")
    }))

    req := httptest.NewRequest("GET", "/", nil)
    rec := httptest.NewRecorder()
    h.ServeHTTP(rec, req)

    if rec.Code != 500 { t.Fatalf("got %d, want 500", rec.Code) }
    _ = logged
}
```

`httptest.NewRecorder` provides a fake ResponseWriter.

## Standard Library Hooks

- `http.Handler`, `http.HandlerFunc`: the canonical types.
- `http.Hijacker`, `http.Flusher`, `http.Pusher`: optional interfaces; preserve when wrapping.
- `context.WithValue`, `context.Context`: per-request data.
- `httptest.NewRecorder`, `httptest.NewRequest`: testing.
- `slog`: structured logging.

## Real-World Patterns

### 1. Recover + log + metrics chain

```go
func RecoverMW(log *slog.Logger, metrics Metrics) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rec := recover(); rec != nil {
                    metrics.Inc("panic")
                    log.Error("panic", "panic", rec, "stack", debug.Stack())
                    http.Error(w, "internal error", 500)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}
```

### 2. Timeout middleware

```go
func TimeoutMW(d time.Duration) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), d)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

Or use `http.TimeoutHandler` (stdlib).

### 3. Authn middleware

```go
func AuthMW(tokens TokenStore) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tok := r.Header.Get("Authorization")
            user, ok := tokens.Lookup(tok)
            if !ok {
                http.Error(w, "unauthorized", 401)
                return
            }
            ctx := context.WithValue(r.Context(), userKey{}, user)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### 4. Compression middleware

```go
func GzipMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
            next.ServeHTTP(w, r)
            return
        }
        w.Header().Set("Content-Encoding", "gzip")
        gz := gzip.NewWriter(w)
        defer gz.Close()
        gzw := gzipResponseWriter{Writer: gz, ResponseWriter: w}
        next.ServeHTTP(gzw, r)
    })
}

type gzipResponseWriter struct {
    io.Writer
    http.ResponseWriter
}

func (g gzipResponseWriter) Write(b []byte) (int, error) { return g.Writer.Write(b) }
```

### 5. Rate limit middleware (token bucket)

```go
import "golang.org/x/time/rate"

func RateLimitMW(perIP int) Middleware {
    var (
        mu      sync.Mutex
        limiters = map[string]*rate.Limiter{}
    )
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr
            mu.Lock()
            l, ok := limiters[ip]
            if !ok {
                l = rate.NewLimiter(rate.Limit(perIP), perIP)
                limiters[ip] = l
            }
            mu.Unlock()
            if !l.Allow() {
                http.Error(w, "too many", 429)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

For production, evict idle IPs; use `golang.org/x/time/rate.Limiter`.

## Anti-Patterns & Gotchas

**Putting Recover middleware second-from-outermost.** A panic in the outer middleware (e.g., RequestID) won't be caught. Recover must be outermost.

**Forgetting that order is non-commutative.** `Logging(Auth(handler))` logs every request; `Auth(Logging(handler))` only logs authenticated requests.

**Modifying request without `r.WithContext(ctx)` or copy.** Mutating shared state across goroutines.

**Wrapping ResponseWriter and dropping optional interfaces.** Hijack (websocket upgrade), Flush (SSE), Push (HTTP/2) silently break.

**Middleware that writes to `w` AND calls `next`.** Two responses → undefined behavior. Either short-circuit OR pass through.

**Sharing state via package-level vars.** Race conditions galore.

**Per-request allocation in middleware.** Each middleware that calls `r.WithContext` allocates a request. Acceptable but watch for hot paths.

**Mixing concrete and interface middleware types.** Pick `Middleware = func(http.Handler) http.Handler`; stick with it.

**Stacking middlewares from different libraries.** `gin.HandlerFunc` and `http.Handler` aren't interchangeable. Routers each have their own.

**Deeply-nested chains with no abstraction.** Use a chain builder; 8 levels of manual wrapping is unreadable.

## Performance Notes

- Per-middleware overhead: ~ns (one function call).
- 10-middleware chain: ~10-20 ns + per-middleware work.
- Context derivation: ~50 ns (creates new context struct).
- ResponseWriter wrapping: ~ns indirection.
- Compression (gzip): bound by encoding cost; ~10-50 MiB/s.
- Rate limiter check: ~50-200 ns.

Middleware itself is not the bottleneck. Per-middleware *logic* is.

## How Big Companies Use It

- **Every Go HTTP service**: middleware is universal.
- **Kubernetes apiserver**: complex chain with auth, audit, admission webhooks.
- **gRPC servers**: interceptors are the equivalent.
- **Caddy** (web server): middleware-driven by design.
- **Stripe Go SDK**: HTTP client middleware for retries, idempotency.
- **The official Go module proxy** uses middleware patterns for auth and logging.
- **Most YC startups**: chi or gin with their middleware ecosystems.

## Source Code References

- chi middleware: [`go-chi/chi/middleware/`](https://github.com/go-chi/chi/tree/master/middleware) — best reference implementations.
- Negroni (older): [`urfave/negroni`](https://github.com/urfave/negroni).
- Echo middleware: [`labstack/echo/middleware/`](https://github.com/labstack/echo/tree/master/middleware).
- gin middleware: [`gin-gonic/gin`](https://github.com/gin-gonic/gin).
- gRPC interceptors: [`grpc-go/interceptor.go`](https://github.com/grpc/grpc-go/blob/master/interceptor.go).
- `http.TimeoutHandler`: [`src/net/http/server.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/server.go).

## Further Reading

- "Writing Middleware in #golang" — Justinas Stankevičius: https://justinas.org/writing-http-middleware-in-go.
- chi documentation: https://github.com/go-chi/chi.
- "Middleware in HTTP servers" — Mat Ryer: blog series.
- "gRPC interceptors" — official docs.
- Effective Go — composition.
- "Composable HTTP handlers" — multiple community posts.

## Exercises / Self-Check

1. Write a middleware that records request bodies above 1 MiB to disk for debugging. Where in the chain should it sit?
2. Why is `Recover` typically outermost? Construct a case where putting it inner would silently swallow errors.
3. Wrap an `http.ResponseWriter` so it preserves `http.Flusher` and `http.Hijacker` interfaces. Test by serving Server-Sent Events through your wrapper.
4. Compare manual chain composition `A(B(C(h)))` to `Chain(A, B, C)(h)`. When does the chain helper help?
5. Convert your chain to also accept conditional middleware: `Chain(A, B, IfDev(C), D)`. Sketch the API.
