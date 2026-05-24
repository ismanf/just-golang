# `net/http/httptest` — Testing HTTP

## TL;DR

`net/http/httptest` is the stdlib's HTTP testing toolkit. The two workhorses: **`httptest.NewServer(handler)`** spins up an in-process HTTP server bound to a random port, exposes `.URL`, and you make real HTTP requests to it (full stack: TCP, transport, your handler); **`httptest.NewRecorder()`** is an `http.ResponseWriter` implementation that captures status, body, and headers in memory, letting you call `handler.ServeHTTP(rec, req)` directly without any network. **`httptest.NewRequest`** constructs an `*http.Request` without going through a real server. Use the server for end-to-end-style tests (you want middleware, TLS handshake, real `*http.Client` behavior); use the recorder for fast unit tests of individual handlers. `httptest.NewTLSServer` brings up an HTTPS server with a self-signed cert; pair with `server.Client()` for a pre-configured client that trusts it.

## Mental Model

```
   Unit-style: NewRecorder + NewRequest        End-to-end-style: NewServer
   ─────────────────────────────────           ───────────────────────────
   handler := http.HandlerFunc(myHandler)      srv := httptest.NewServer(handler)
   rec := httptest.NewRecorder()               defer srv.Close()
   req := httptest.NewRequest("GET", "/x", nil)
   handler.ServeHTTP(rec, req)                 resp, err := http.Get(srv.URL + "/x")
                                                defer resp.Body.Close()
   rec.Code      → 200
   rec.Body      → bytes.Buffer
   rec.Header()  → http.Header
```

Recorder: in-memory, ~1 µs per call. Server: real network, ~100 µs per call. Pick the smaller tool first.

## Syntax & Basic Usage

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"
)

// Recorder style
func TestHandler(t *testing.T) {
    h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("hello"))
    })

    rec := httptest.NewRecorder()
    req := httptest.NewRequest("GET", "/", nil)
    h.ServeHTTP(rec, req)

    if rec.Code != http.StatusOK {
        t.Errorf("status = %d, want 200", rec.Code)
    }
    if rec.Body.String() != "hello" {
        t.Errorf("body = %q, want %q", rec.Body.String(), "hello")
    }
}

// Server style
func TestRoundTrip(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(myHandler))
    defer srv.Close()

    resp, err := http.Get(srv.URL + "/users/42")
    if err != nil { t.Fatal(err) }
    defer resp.Body.Close()
    // ...
}

// TLS
srv := httptest.NewTLSServer(handler)
defer srv.Close()
client := srv.Client()              // trusts the self-signed cert
resp, _ := client.Get(srv.URL)
```

## Deep Dive

### `httptest.NewRecorder`

```go
type ResponseRecorder struct {
    Code      int
    HeaderMap http.Header
    Body      *bytes.Buffer
    Flushed   bool
    // ...
}

func NewRecorder() *ResponseRecorder
```

`Recorder` implements `http.ResponseWriter` plus a few extras:

- `Code` — last `WriteHeader` value (default 200).
- `Body` — accumulated `Write` bytes.
- `HeaderMap` — call `rec.Header()` to read.
- `Flushed` — true if `Flush` was called.
- `Result()` — returns `*http.Response`, useful for code that expects `Response` shape.

```go
resp := rec.Result()
body, _ := io.ReadAll(resp.Body)
resp.Body.Close()
```

`Result()` snapshots the recorder state. Use when you want to feed the response into code that expects an `*http.Response`.

### `httptest.NewRequest`

```go
func NewRequest(method, target string, body io.Reader) *http.Request
```

Constructs a request with sensible defaults:

- `Method`, `URL` (parsed from `target`).
- `Proto: "HTTP/1.1"`, `ProtoMajor: 1`, `ProtoMinor: 1`.
- `RemoteAddr: "192.0.2.1:1234"` (a `RFC 5737`/`RFC 3849` reserved IP).
- `Body` if non-nil.
- `Host` from URL or `example.com` if missing.

```go
req := httptest.NewRequest("POST", "/users", strings.NewReader(`{"name":"alice"}`))
req.Header.Set("Content-Type", "application/json")
```

Differences from `http.NewRequest`: `httptest.NewRequest` panics on error (test code; failure is a test bug) and sets `RemoteAddr` for handlers that read it.

### `httptest.NewServer`

```go
func NewServer(handler http.Handler) *Server
```

Spins up a `net.Listener` on a random port (typically 127.0.0.1:<rand>) and serves `handler`. `srv.URL` is the http://host:port prefix.

```go
srv := httptest.NewServer(handler)
defer srv.Close()
resp, _ := http.Get(srv.URL + "/path")
```

**Always `defer srv.Close()`** — otherwise the listener leaks. (`go vet`'s `httpresponse` analyzer doesn't catch this directly; reviewers do.)

The server uses the default `http.Server`; all middleware/timeouts/limits the handler relies on apply.

### `httptest.NewUnstartedServer`

```go
srv := httptest.NewUnstartedServer(handler)
srv.Listener = customListener         // e.g., Unix socket
srv.Config.ReadHeaderTimeout = time.Second
srv.Start()
defer srv.Close()
```

When you need to customize `Listener` or `Config` before `Start`. Otherwise use `NewServer`.

### `httptest.NewTLSServer`

```go
srv := httptest.NewTLSServer(handler)
defer srv.Close()

client := srv.Client()                // *http.Client preconfigured to trust srv's cert
resp, _ := client.Get(srv.URL)        // srv.URL starts with https://
```

`srv.Client()` returns a client whose `Transport` has the self-signed cert in its `RootCAs`. Use this client; `http.Get` (with the default transport) won't trust it.

For your own custom client:

```go
client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},  // dev only!
    },
}
```

Or copy the cert from `srv.Certificate()`.

### `srv.Client()`

```go
func (s *Server) Client() *http.Client
```

Returns a client with sensible defaults for talking to *this* server. For `httptest.NewServer`, basically a plain client; for `httptest.NewTLSServer`, includes the self-signed cert trust. Always prefer `srv.Client()` over making your own.

### Testing middleware

```go
func TestAuthMiddleware(t *testing.T) {
    inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    h := AuthMiddleware(inner)

    t.Run("no token", func(t *testing.T) {
        rec := httptest.NewRecorder()
        req := httptest.NewRequest("GET", "/", nil)
        h.ServeHTTP(rec, req)
        if rec.Code != http.StatusUnauthorized {
            t.Errorf("status = %d, want 401", rec.Code)
        }
    })

    t.Run("valid token", func(t *testing.T) {
        rec := httptest.NewRecorder()
        req := httptest.NewRequest("GET", "/", nil)
        req.Header.Set("Authorization", "Bearer valid")
        h.ServeHTTP(rec, req)
        if rec.Code != http.StatusOK {
            t.Errorf("status = %d, want 200", rec.Code)
        }
    })
}
```

### Testing a client (mocking the server)

```go
func TestClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/api/v1/users/42" {
            t.Errorf("path = %s", r.URL.Path)
        }
        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"name":"alice"}`))
    }))
    defer srv.Close()

    c := NewClient(srv.URL)
    user, err := c.GetUser(context.Background(), 42)
    if err != nil { t.Fatal(err) }
    if user.Name != "alice" {
        t.Errorf("name = %q", user.Name)
    }
}
```

The server impersonates the real upstream; the client makes real HTTP calls.

### Context in handler tests

```go
func TestX(t *testing.T) {
    rec := httptest.NewRecorder()
    req := httptest.NewRequest("GET", "/", nil)

    ctx := context.WithValue(req.Context(), "user", "alice")
    req = req.WithContext(ctx)

    handler.ServeHTTP(rec, req)
}
```

For 1.24+, `t.Context()` provides a test-scoped ctx:

```go
req := httptest.NewRequest("GET", "/", nil).WithContext(t.Context())
```

### Streaming responses

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    f := w.(http.Flusher)
    for i := 0; i < 5; i++ {
        fmt.Fprintf(w, "chunk %d\n", i)
        f.Flush()
        time.Sleep(10 * time.Millisecond)
    }
}))
defer srv.Close()

resp, _ := http.Get(srv.URL)
defer resp.Body.Close()
scanner := bufio.NewScanner(resp.Body)
for scanner.Scan() {
    t.Logf("got: %s", scanner.Text())
}
```

`Recorder` supports `Flush()` (sets `Flushed = true`); for actual streaming tests you need the full server.

### `httptest.NewServer` and timeouts

The default `httptest.Server`'s `http.Server.ReadHeaderTimeout` is **unset** (0 = unlimited). For slowloris-resistant tests:

```go
srv := httptest.NewUnstartedServer(handler)
srv.Config.ReadHeaderTimeout = 5 * time.Second
srv.Start()
```

### Cookies and sessions

```go
srv := httptest.NewServer(handler)
defer srv.Close()

jar, _ := cookiejar.New(nil)
client := srv.Client()
client.Jar = jar

resp1, _ := client.Get(srv.URL + "/login")
resp2, _ := client.Get(srv.URL + "/profile")    // jar sends cookies from resp1
```

### Testing with multipart form

```go
var buf bytes.Buffer
w := multipart.NewWriter(&buf)
part, _ := w.CreateFormFile("file", "test.txt")
part.Write([]byte("contents"))
w.Close()

req := httptest.NewRequest("POST", "/upload", &buf)
req.Header.Set("Content-Type", w.FormDataContentType())

rec := httptest.NewRecorder()
handler.ServeHTTP(rec, req)
```

### Subtests for many cases

```go
tests := []struct {
    name   string
    method string
    path   string
    body   string
    want   int
}{
    {"GET ok", "GET", "/users/42", "", 200},
    {"POST ok", "POST", "/users", `{"name":"a"}`, 201},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        rec := httptest.NewRecorder()
        req := httptest.NewRequest(tt.method, tt.path, strings.NewReader(tt.body))
        handler.ServeHTTP(rec, req)
        if rec.Code != tt.want {
            t.Errorf("status = %d, want %d", rec.Code, tt.want)
        }
    })
}
```

### `httptest` and `httputil.DumpRequest`/`DumpResponse`

```go
b, _ := httputil.DumpRequest(req, true)
t.Logf("request:\n%s", b)
```

Useful for debugging tests; produces RFC-style HTTP text.

### `httptest.NewServer` vs. mock libraries

Libraries like `gock`, `httpmock`, `responder` intercept transport-level calls. They're useful when:

- You can't control how the client builds requests.
- You need response sequencing logic (return 500, then 200).

For most cases, `httptest.NewServer` is simpler and more accurate (full stack).

## Standard Library Hooks

- `net/http/httptest` — recorder, request, server, TLS server.
- `net/http/httputil.DumpRequest`, `DumpResponse` — dump for debugging.
- `net/http.Handler`, `HandlerFunc` — interface implemented by your code.
- `net/http.Client` — for making test-time requests.
- `net/http/cookiejar` — for cookie-based sessions.
- `mime/multipart` — for form uploads.
- `crypto/tls` — for custom TLS clients.

## Real-World Patterns

### 1. Recorder for a unit test

```go
func TestHelloHandler(t *testing.T) {
    rec := httptest.NewRecorder()
    req := httptest.NewRequest("GET", "/hello", nil)
    HelloHandler(rec, req)
    if rec.Body.String() != "hello\n" {
        t.Errorf("got %q", rec.Body.String())
    }
}
```

### 2. Server for an integration test

```go
func TestUserAPI(t *testing.T) {
    srv := httptest.NewServer(NewRouter(db))
    defer srv.Close()

    resp, _ := http.Post(srv.URL+"/users", "application/json",
        strings.NewReader(`{"name":"alice"}`))
    defer resp.Body.Close()
    if resp.StatusCode != 201 {
        t.Errorf("status = %d", resp.StatusCode)
    }
}
```

### 3. Mock upstream

```go
func TestExternalClient(t *testing.T) {
    upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"result":42}`))
    }))
    defer upstream.Close()

    c := NewClient(upstream.URL)
    got, err := c.Fetch(context.Background())
    if err != nil { t.Fatal(err) }
    if got != 42 { t.Errorf("got %d", got) }
}
```

### 4. TLS test

```go
func TestHTTPS(t *testing.T) {
    srv := httptest.NewTLSServer(handler)
    defer srv.Close()
    resp, err := srv.Client().Get(srv.URL + "/")
    if err != nil { t.Fatal(err) }
    resp.Body.Close()
}
```

### 5. Streaming SSE

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    f := w.(http.Flusher)
    for i := 0; i < 3; i++ {
        fmt.Fprintf(w, "data: %d\n\n", i)
        f.Flush()
    }
}))
defer srv.Close()

resp, _ := http.Get(srv.URL)
defer resp.Body.Close()
io.Copy(os.Stdout, resp.Body)
```

### 6. Multiple endpoints

```go
mux := http.NewServeMux()
mux.HandleFunc("/users", usersHandler)
mux.HandleFunc("/posts", postsHandler)
srv := httptest.NewServer(mux)
defer srv.Close()
```

### 7. Inspect request inside a mock server

```go
var gotMethod, gotPath string
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    gotMethod = r.Method
    gotPath = r.URL.Path
    w.WriteHeader(http.StatusOK)
}))
defer srv.Close()

c.Trigger()  // your code under test makes a request

if gotMethod != "POST" || gotPath != "/webhook" {
    t.Errorf("got %s %s", gotMethod, gotPath)
}
```

## Anti-Patterns & Gotchas

**Forgetting `defer srv.Close()`.** Listener leaks; CI gradually exhausts file descriptors.

**Forgetting `defer resp.Body.Close()`.** Connection isn't returned to the pool; eventually exhausts the transport. `go vet`'s `httpresponse` analyzer catches it.

**Using `http.DefaultClient` against `httptest.NewTLSServer`.** Default client doesn't trust the self-signed cert; request fails with TLS handshake error. Use `srv.Client()`.

**Hardcoding ports in tests.** `httptest.NewServer` picks a random port; use `srv.URL`. Hardcoded ports flake on busy machines.

**Reusing one recorder across multiple `ServeHTTP` calls.** State accumulates (headers, body). Create a new `NewRecorder()` per call.

**Using `http.NewRequest` instead of `httptest.NewRequest`.** Both work, but `httptest.NewRequest` panics on error (test-style) and sets `RemoteAddr` (handlers that read it work).

**Treating `Recorder.Code = 0` as a real response.** If your handler never calls `WriteHeader`, `rec.Code` defaults to 200 (in the recorder), but the real `http.Server` would also default to 200. Don't rely on 0 to mean "no response".

**Slow tests using `httptest.NewServer` when `Recorder` would do.** Recorder is ~100× faster. Use it for pure-handler tests.

**Asserting on `rec.HeaderMap` directly.** Use `rec.Header()` (the standard accessor) to handle the canonical-case lookup.

**Concurrent writes to a `Recorder`.** Not goroutine-safe. If your handler spawns goroutines that write, the recorder may panic. Use a server for that scenario.

**Forgetting timeouts on the server's `http.Server`.** Default is no timeout; slow clients can hang test goroutines. Set `ReadHeaderTimeout` via `NewUnstartedServer`.

**Tests that depend on real DNS / outbound network.** Use `httptest.NewServer` for the upstream; don't hit real endpoints.

## Performance Notes

- `Recorder` per call: ~1 µs.
- `NewServer` startup: ~100 µs.
- `NewTLSServer` startup: ~10 ms (TLS handshake setup).
- Round trip via `srv.URL`: ~100 µs (localhost).
- Closing server: <1 ms.

For benchmarks: use `Recorder` to measure pure handler logic; use `NewServer` only when measuring transport-related behavior.

## How Big Companies Use It

- **Google** uses `httptest` for stdlib's `net/http` tests and for many internal Go services: https://github.com/golang/go/tree/master/src/net/http/httptest.
- **Kubernetes** uses `httptest.NewServer` heavily for `client-go` tests: https://github.com/kubernetes/kubernetes.
- **Uber** uses `httptest` for HTTP-handling code; some teams wrap it with mock-injection helpers: https://github.com/uber-go.
- **HashiCorp** uses `httptest` for Terraform provider HTTP fixtures: https://github.com/hashicorp/terraform-provider-aws.
- **CockroachDB** uses `httptest` for their admin UI tests: https://github.com/cockroachdb/cockroach.
- **Cloudflare** uses `httptest` for HTTP/2 and HTTP/3 handler tests: https://blog.cloudflare.com.
- **Tailscale** uses `httptest` for control-plane API tests: https://github.com/tailscale/tailscale.

## Source Code References

Pinned to `go1.26`.

- Package: [`src/net/http/httptest`](https://github.com/golang/go/tree/release-branch.go1.26/src/net/http/httptest).
- `Recorder`: [`src/net/http/httptest/recorder.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/httptest/recorder.go).
- `Server`: [`src/net/http/httptest/server.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/httptest/server.go).
- `NewRequest`: [`src/net/http/httptest/httptest.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/httptest/httptest.go).
- TLS cert generation: [`src/net/http/internal/testcert/testcert.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/http/internal/testcert/testcert.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Package httptest": https://pkg.go.dev/net/http/httptest.
- "Testing HTTP handlers in Go" (Alex Edwards): https://www.alexedwards.net/blog/testing-handlers.
- "How to test HTTP servers in Go" (Mat Ryer): https://medium.com/@matryer/5-simple-tips-and-tricks-for-writing-unit-tests-in-golang-619653f90742.
- "HTTPS testing with self-signed certs" (Filippo Valsorda): https://words.filippo.io.
- "Writing testable HTTP services" (Gopheracademy): https://blog.gopheracademy.com.

## Exercises / Self-Check

1. Write a handler that returns JSON; test with both `Recorder` and `NewServer`. Compare runtimes.
2. Mock an upstream service with `httptest.NewServer`. Verify your client sends the right path/method.
3. Use `NewTLSServer` and `srv.Client()` to make an HTTPS request. Why doesn't `http.Get(srv.URL)` work?
4. Add cookie-based session handling and test login → access flow via `cookiejar`.
5. Stream Server-Sent Events; test that flush makes data visible incrementally.
