# Frameworks — chi, gin, echo, fiber

## TL;DR

Go's stdlib `net/http` is good enough for most production services — especially since 1.22 added method-aware routing. The four community frameworks people reach for are: **chi** (the idiomatic-Go choice; thin layer over `net/http`; `http.Handler` everywhere), **gin** (most popular by stars; opinionated context; mature middleware ecosystem), **echo** (similar to gin but a slightly cleaner API), and **fiber** (built on `fasthttp` instead of `net/http`; fastest in benchmarks but **incompatible with the entire `net/http` ecosystem**). The single biggest decision: **stay inside `net/http`** (stdlib, chi, echo, gin all do) or **leave it for `fasthttp`** (fiber). The first choice gives you the entire Go ecosystem — middleware, observability tools, security scanners, HTTP/2, server hijack. The second buys 1.5-3x raw throughput at the cost of every library that assumes `http.Handler`. For most apps, *the bottleneck is your database, not your router*, and that benchmark advantage evaporates in practice. Go 1.22+ `ServeMux` makes "use the stdlib" a stronger option than it was in 2020. If you do reach for a framework, **chi** is the safest bet — it doesn't fight `net/http` and you can drop it without a rewrite.

## Mental Model

```
   Stdlib net/http
   ───────────────
        │
        ├─► chi          (decorator pattern; http.Handler in/out)
        ├─► gin          (own Context type; ResponseWriter wrapped)
        ├─► echo         (own Context type; similar to gin)
        │
        └─► fasthttp ◄──── fiber (rewrites HTTP from scratch for perf)
```

Two invariants:

1. **Frameworks that wrap `net/http` interoperate**. Mix chi routes with raw `http.Handler`; use `otelhttp`, `httptest`, `httputil.ReverseProxy` freely.
2. **`fasthttp` is its own world**. The standard libraries don't work. The middleware doesn't compose. You need fiber-specific everything.

## Side-by-Side: Same Endpoint

### Stdlib (Go 1.22+)

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    fmt.Fprintf(w, "user %s\n", id)
})
http.ListenAndServe(":8080", mux)
```

### chi

```go
import "github.com/go-chi/chi/v5"

r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Get("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    fmt.Fprintf(w, "user %s\n", id)
})
http.ListenAndServe(":8080", r)
```

### gin

```go
import "github.com/gin-gonic/gin"

r := gin.New()
r.Use(gin.Recovery(), gin.Logger())
r.GET("/users/:id", func(c *gin.Context) {
    c.String(200, "user %s\n", c.Param("id"))
})
r.Run(":8080")
```

### echo

```go
import "github.com/labstack/echo/v4"

e := echo.New()
e.Use(echomw.Recover(), echomw.Logger())
e.GET("/users/:id", func(c echo.Context) error {
    return c.String(200, "user "+c.Param("id"))
})
e.Start(":8080")
```

### fiber

```go
import "github.com/gofiber/fiber/v2"

app := fiber.New()
app.Use(logger.New(), recover.New())
app.Get("/users/:id", func(c *fiber.Ctx) error {
    return c.SendString("user " + c.Params("id"))
})
app.Listen(":8080")
```

Five lines each; the surface differences are small. The deeper differences are in middleware shape, error model, and what happens at scale.

## chi — The Idiomatic Choice

`github.com/go-chi/chi/v5`. Thin layer over `net/http`. Roughly 1k LoC core; the rest is optional middleware.

Strengths:

- **`http.Handler` everywhere**. Middleware signature is `func(http.Handler) http.Handler` — same as raw stdlib. You can drop chi and your handlers still work.
- **Sub-routers + group middleware**:
  ```go
  r := chi.NewRouter()
  r.Use(commonMW)
  r.Route("/api", func(r chi.Router) {
      r.Use(apiMW)
      r.Get("/users", listUsers)
      r.Mount("/admin", adminRouter())
  })
  ```
- **Per-pattern timeout**: `r.Use(middleware.Timeout(60*time.Second))`.
- **Compatibility**: any stdlib `Handler`/`HandlerFunc`, any framework's middleware that takes `http.Handler`.

Weaknesses:

- No batteries-included batteries. Validation, binding, rendering — you write or import.
- Slower than gin/echo on micro-benchmarks (rarely matters).

Since Go 1.22's mux gained method matching, **chi's main remaining advantage is sub-router with prefix middleware** (which the stdlib still doesn't have ergonomically).

## gin — The Popular Choice

`github.com/gin-gonic/gin`. Most stars; most blog posts; most Stack Overflow answers.

Strengths:

- **`*gin.Context`** carries everything: params, query, headers, body binding, response helpers. Convenient.
- **Binding + validation**: `c.ShouldBindJSON(&req)` with struct tags + `go-playground/validator` integration.
- **Rendering helpers**: `c.JSON(...)`, `c.XML(...)`, `c.HTML(...)`.
- **Middleware ecosystem**: extensive (auth, CORS, rate limiting, sessions).
- **Performance**: among the fastest `net/http`-compatible options.

Weaknesses:

- **`*gin.Context` is gin-specific.** Your handlers are no longer plug-compatible with stdlib. If you decide to leave gin, every handler needs a rewrite.
- **Error handling**: returning an error from a handler isn't first-class; you `c.Error(err)` and check at the middleware layer. Slightly awkward.
- **Default routing trie** can panic on conflicting routes if you're not careful.

## echo — The Cleaner Alternative

`github.com/labstack/echo`. Similar feature set to gin, slightly cleaner conventions:

- **Handlers return `error`** — more idiomatic Go. The framework picks up the error and the global error handler decides response.
- **Type-safe binding** via reflection and tags.
- **Strong middleware story** with the same per-route / per-group composition gin has.
- **WebSocket support** built in.

Many teams choose echo over gin for the error-return convention alone.

## fiber — The Fast But Different One

`github.com/gofiber/fiber`. Built on `valyala/fasthttp`, which is a from-scratch HTTP server that pools everything (request, response, header maps) to minimise allocation.

Strengths:

- **Throughput**: 1.5-3x ahead of `net/http` in TechEmpower benchmarks.
- **Memory**: lower allocation profile.
- **Familiar API** (Express.js-inspired) for ex-Node devs.

Major caveats:

- **Not `net/http`-compatible.** `c.Body()` returns a `[]byte` you cannot keep past the handler — it's pooled. Misuse leaks pool entries or corrupts subsequent requests.
- **No HTTP/2** (until very recently, and still incomplete). For HTTP/3 / TLS 1.3 features, you're often behind.
- **Ecosystem split**: middleware must be fiber-specific. `otelhttp`, `httputil.ReverseProxy`, every `http.Handler`-based piece does not apply.
- **Testing**: stdlib `httptest` doesn't work directly.
- **`net.Listen` is replaced by fasthttp's listener**: hijacking semantics differ.

When fiber is worth it:

- You profiled; HTTP layer is the actual bottleneck (rare).
- Your app is a thin pass-through (proxy, edge); database isn't gating throughput.
- You can live without HTTP/2/3 and stdlib-compatible middleware.

When fiber is the wrong choice:

- You want `otelhttp`, prom middleware, or any of the standard ecosystem.
- You expect to swap pieces over time.
- You have any `[]byte`-aliasing footguns in the team.

## Quick Decision Tree

```
   Need raw throughput, willing to give up ecosystem?
   ──► fiber

   Already on Go 1.22, modest routing needs, want zero deps?
   ──► net/http stdlib

   Want sub-routers + per-group middleware + stay stdlib-compatible?
   ──► chi

   Want batteries (binding, validation, rendering, big eco)?
   ──► gin   or   echo
        (echo if you prefer handler-returns-error)
```

## Feature Matrix

| Feature                          | net/http | chi  | gin  | echo | fiber |
|----------------------------------|----------|------|------|------|-------|
| `http.Handler`-compatible        | yes      | yes  | wraps| wraps| no    |
| Method routing                   | 1.22+    | yes  | yes  | yes  | yes   |
| Path params (`/users/{id}`)      | 1.22+    | yes  | yes  | yes  | yes   |
| Wildcards (`/files/{p...}`)      | 1.22+    | yes  | yes  | yes  | yes   |
| Sub-routers / groups             | manual   | yes  | yes  | yes  | yes   |
| Per-group middleware             | manual   | yes  | yes  | yes  | yes   |
| Built-in JSON binding+validation | no       | no   | yes  | yes  | yes   |
| Built-in error model             | no       | no   | partial | yes (handlers return error) | partial |
| HTTP/2                           | yes      | yes  | yes  | yes  | partial |
| WebSocket                        | external | external | external | yes | yes  |
| Server-Sent Events               | manual   | yes  | yes  | yes  | yes   |
| Stdlib `httptest`                | yes      | yes  | yes  | yes  | adapter |
| OTel via `otelhttp`              | yes      | yes  | yes  | yes  | fiber-specific |
| Stable since                     | 2009     | 2016 | 2014 | 2016 | 2020  |

## Middleware Composition

```go
// chi — same as stdlib
func mw(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ...
        next.ServeHTTP(w, r)
    })
}

// gin
func mw(c *gin.Context) {
    // ... pre
    c.Next()
    // ... post
}

// echo
func mw(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // ... pre
        err := next(c)
        // ... post
        return err
    }
}

// fiber
func mw(c *fiber.Ctx) error {
    // ... pre
    err := c.Next()
    // ... post
    return err
}
```

chi's signature is `http.Handler → http.Handler`. Any stdlib middleware works. That's the killer ecosystem property.

## Request Binding

### gin

```go
type Req struct {
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"gte=18,lte=120"`
}
var req Req
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(400, gin.H{"error": err.Error()})
    return
}
```

### echo

```go
var req Req
if err := c.Bind(&req); err != nil { return err }
if err := c.Validate(&req); err != nil { return err }
```

### chi / stdlib

You write the 6-line decode-and-validate yourself, or use `gorilla/schema` for query strings + `go-playground/validator` for body:

```go
import "github.com/go-playground/validator/v10"
var validate = validator.New()

var req Req
if err := json.NewDecoder(r.Body).Decode(&req); err != nil { ... }
if err := validate.Struct(&req); err != nil { ... }
```

## Error Handling

### gin

```go
r.GET("/", func(c *gin.Context) {
    if err := doSomething(); err != nil {
        c.AbortWithError(500, err)
        return
    }
    c.JSON(200, gin.H{"ok": true})
})

// Middleware looks at c.Errors after the chain
r.Use(func(c *gin.Context) {
    c.Next()
    for _, e := range c.Errors {
        slog.Error("err", "err", e.Err)
    }
})
```

### echo

```go
e.HTTPErrorHandler = func(err error, c echo.Context) {
    code := 500
    if he, ok := err.(*echo.HTTPError); ok { code = he.Code }
    c.JSON(code, map[string]string{"error": err.Error()})
}

e.GET("/", func(c echo.Context) error {
    if err := doSomething(); err != nil {
        return echo.NewHTTPError(500, "internal")
    }
    return c.JSON(200, map[string]bool{"ok": true})
})
```

echo's "handlers return error" model is the most idiomatic Go.

## When Performance Actually Matters

Benchmarks: TechEmpower shows fiber ahead, gin/echo a tier below, chi just under, stdlib ~5-15% behind the leaders for hello-world. But:

- A typical service spends >90% of its time in the database, RPCs, JSON encoding.
- The router itself adds <1% of request latency.
- The "fast" frameworks' advantage is in micro-benchmarks (return a 6-byte string with no work).

Where the framework choice *does* affect perf:

- **Allocations per request**: gin/echo/chi all do per-request allocs for `*gin.Context`, route lookup tables. Fiber pools.
- **Hot routing tries** with many routes: chi's trie is fine to 10k routes; gin's slightly slower; fiber's faster. Real services rarely have >500 routes.
- **JSON encoding**: orthogonal to the framework; benchmark `encoding/json` vs `bytedance/sonic` vs `goccy/go-json` independently.

Operational advice: **don't pick a framework for performance unless you have profiled and proven HTTP is your bottleneck**.

## Testing

### stdlib / chi / gin / echo

```go
import "net/http/httptest"

func TestUserGet(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/42", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    if w.Code != 200 { t.Fatal(w.Code) }
}
```

Same code works for stdlib, chi, gin (via `router.Handler()`), echo (via `e.ServeHTTP`).

### fiber

```go
import "github.com/gofiber/fiber/v2"
// No httptest. Use fiber's app.Test():
resp, _ := app.Test(httptest.NewRequest("GET", "/users/42", nil))
```

Adapter exists, but it's not the same.

## Anti-Patterns & Gotchas

**Reaching for a framework "for performance" without profiling.** 9 times in 10 your bottleneck is elsewhere.

**Mixing fiber with `net/http`-shaped libraries.** Subtle bugs, panics on pooled-buffer reuse.

**Holding `c.Body()` bytes past handler in fiber.** Pool reuse; bytes get overwritten.

**Storing `*gin.Context` in goroutines** — its lifetime is the request. Use `c.Copy()` if you must.

**Concurrent writes to `*gin.Context`.** Not safe.

**Using framework-specific error wrappers when you also have `slog`+Sentry middleware that expects errors.** Convert at the boundary.

**Mounting too many routers.** Some frameworks have O(N) lookup in mount table. Profile if >100.

**Trusting `c.ClientIP()` blindly.** Most frameworks read `X-Forwarded-For` unconditionally — spoofable unless you configure trusted proxies.

**Forgetting middleware order matters.** Recovery should be outermost. CORS before auth.

**Hot-swapping frameworks "for cleanliness."** Migrations cost more than perceived benefit. Choose once and stick.

**Choosing fiber for a backend service that talks to Postgres.** The bottleneck is Postgres.

**Mixing gin and chi via `http.Handler` adapters but losing context.** `*gin.Context` doesn't survive across adapters.

**Skipping HTTPS in dev because the framework's TLS story is "complicated."** All four support `ListenAndServeTLS`-equivalent. Don't skip.

## Performance Notes

(TechEmpower-style hello-world; YMMV; rounded.)

| Framework | RPS (single core, hello) | Allocs/req |
|-----------|--------------------------|-----------|
| stdlib    | ~150k | 4-8 |
| chi       | ~140k | 4-8 |
| gin       | ~170k | 3-5 |
| echo      | ~170k | 3-5 |
| fiber     | ~350k | 0-1 |

Real services with DB calls converge — most run at 1-10k RPS regardless of framework.

## How Big Companies Use It

- **Google** (internally) uses an internal stack mostly; external Go services use stdlib `net/http`.
- **Cloudflare** uses stdlib + custom middleware; some teams use chi.
- **Uber** has a mix; many Go services use stdlib + custom routing.
- **DoorDash** uses gin for many services.
- **TikTok / ByteDance** uses both gin and their own `hertz` framework (similar style, in-house).
- **Shopify** Storefront uses stdlib + chi-style routing for Go services.
- **GitHub** uses chi for most Go services.
- **Twitch** uses chi internally.
- **Discord** uses gin.
- **Tailscale** uses stdlib + a small custom routing layer.
- **Kubernetes** uses stdlib `net/http` (`kube-apiserver` has a heavy custom layer on top).

The bigger / older the company, the more likely they use stdlib + chi. The younger / blog-driven / "stack starter" the company, the more likely gin or echo.

## Source Code References

- chi: https://github.com/go-chi/chi.
- gin: https://github.com/gin-gonic/gin.
- echo: https://github.com/labstack/echo.
- fiber: https://github.com/gofiber/fiber.
- fasthttp: https://github.com/valyala/fasthttp.
- ByteDance hertz: https://github.com/cloudwego/hertz.
- TechEmpower benchmarks: https://www.techempower.com/benchmarks/.

## Further Reading

- "Go 1.22 routing enhancements": https://go.dev/blog/routing-enhancements.
- "Choosing a Go web framework" — Peter Bourgon, various engineering blogs.
- "Why I don't use a Go framework" (multiple posts; gist: stdlib is enough).
- chi docs: https://go-chi.io/.
- gin docs: https://gin-gonic.com/docs/.
- echo docs: https://echo.labstack.com/.
- fiber docs: https://docs.gofiber.io/.

## Exercises / Self-Check

1. Build the same "hello user" endpoint in stdlib (1.22), chi, gin, echo, fiber. Compare code length and code style.
2. Profile each under `wrk -c 100 -d 30s`. Measure RPS and p99 latency for hello-world. Then add a 5ms `time.Sleep` (simulating DB) and re-measure — observe convergence.
3. Migrate a chi service to gin (or vice versa). Note what breaks. How long did it take?
4. Use `otelhttp` middleware with chi. Try the same with fiber — observe the incompatibility.
5. Implement a request-binding + validation pattern in chi using `validator/v10`. Compare lines of code to gin's `ShouldBindJSON`.
6. Try fiber's `c.Body()` bytes saved past handler — confirm they get corrupted on the next request.
7. Build a graceful-shutdown story in each framework. Note which gives you the cleanest path.
8. For a real service in your org, list the dependencies that assume `http.Handler`. Estimate the rewrite cost if you switched to fiber.
