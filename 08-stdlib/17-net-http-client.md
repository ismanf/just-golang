# `net/http` — Client Side

## TL;DR

The default `http.Client` and `http.DefaultClient` have **no timeout** — a hung server can stall your code forever. Always construct your own `http.Client` with a `Timeout` (or use `http.NewRequestWithContext` with a deadlined context). Reuse one `http.Client` (or its `Transport`) across the program — each holds the connection pool. The single most common bug: forgetting `defer resp.Body.Close()`, which leaks the connection so the pool can't reuse it.

## Mental Model

```
http.Client
  ├─ Transport (http.RoundTripper, default: http.DefaultTransport)
  ├─ Timeout (whole-request including body read)
  └─ CheckRedirect, Jar (cookies)

Transport
  ├─ Connection pool (per host)
  ├─ MaxIdleConns, MaxIdleConnsPerHost, IdleConnTimeout
  └─ TLSHandshakeTimeout, DialContext, etc.

Request flow:
  NewRequestWithContext(ctx, ...) → Client.Do(req)
  → Transport.RoundTrip → returns *Response
  → defer resp.Body.Close()  (REQUIRED)
  → read resp.Body to EOF or Close before next request
```

## Syntax & Basic Usage

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"time"
)

func main() {
	client := &http.Client{Timeout: 10 * time.Second}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	req, _ := http.NewRequestWithContext(ctx, "GET", "https://go.dev", nil)
	resp, err := client.Do(req)
	if err != nil { panic(err) }
	defer resp.Body.Close()

	b, _ := io.ReadAll(resp.Body)
	fmt.Println(resp.Status, len(b), "bytes")
}
```

## Deep Dive

### Why `DefaultClient` is dangerous

```go
resp, err := http.Get(url) // uses DefaultClient — no timeout
```

A server that accepts the connection but never replies will block forever. Always use your own client:

```go
var client = &http.Client{Timeout: 30 * time.Second}
```

### `Timeout` vs context

`Client.Timeout` covers the whole transaction: connect, headers, body. Context with deadline composes cleanly with cancellation:

```go
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req) // ctx cancellation aborts in-flight
```

Use context for per-request deadlines; use `Client.Timeout` as a global ceiling.

### Connection pooling

`http.Transport` keeps idle connections per host. Tune for high RPS:

```go
tr := &http.Transport{
	MaxIdleConns:        200,
	MaxIdleConnsPerHost: 100, // default is 2 — usually too low
	IdleConnTimeout:     90 * time.Second,
	DialContext: (&net.Dialer{
		Timeout:   5 * time.Second,
		KeepAlive: 30 * time.Second,
	}).DialContext,
	TLSHandshakeTimeout:   5 * time.Second,
	ResponseHeaderTimeout: 10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
	ForceAttemptHTTP2:     true,
}
client := &http.Client{Transport: tr, Timeout: 30 * time.Second}
```

Reuse `client` across the program. Constructing a new client per request defeats pooling.

### The `defer Body.Close()` rule

```go
resp, err := client.Do(req)
if err != nil { return err }
defer resp.Body.Close()
```

If you return without closing, the connection is held forever and the pool can't reclaim it. Eventually you exhaust `MaxIdleConnsPerHost`.

Also: **read the body to EOF** before closing, otherwise the connection cannot be reused even after Close:

```go
io.Copy(io.Discard, resp.Body)
resp.Body.Close()
```

`json.NewDecoder(resp.Body).Decode(&v)` reads only what's needed; drain explicitly for partial decoding.

### Status codes

```go
if resp.StatusCode >= 400 {
	body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
	return fmt.Errorf("status %d: %s", resp.StatusCode, body)
}
```

`Client.Do` returns an error only for network-level failures, not HTTP status codes. Always check `resp.StatusCode`.

### Redirects

Default: follows up to 10 redirects.

```go
client.CheckRedirect = func(req *http.Request, via []*http.Request) error {
	if len(via) >= 5 { return http.ErrUseLastResponse }
	return nil
}
```

Return `http.ErrUseLastResponse` to short-circuit.

### Cookies

```go
jar, _ := cookiejar.New(nil)
client := &http.Client{Jar: jar}
```

Without a `Jar`, cookies are ignored.

### Headers

```go
req, _ := http.NewRequestWithContext(ctx, "POST", url, body)
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Authorization", "Bearer "+token)
req.Header.Set("User-Agent", "myapp/1.0")
```

Default User-Agent: `Go-http-client/1.1` — set a real one in production.

### Request body and retry replay

```go
body := strings.NewReader(jsonStr)
req, _ := http.NewRequestWithContext(ctx, "POST", url, body)
```

`http.NewRequest` accepts `io.Reader`. For retryable requests, body must be replayable — use `bytes.Reader`, `strings.Reader`, or set `GetBody`:

```go
req.GetBody = func() (io.ReadCloser, error) {
	return io.NopCloser(bytes.NewReader(data)), nil
}
```

### HTTP/2

Transparent: servers advertising `h2` via ALPN are upgraded automatically.

### Streaming responses

```go
resp, _ := client.Do(req)
defer resp.Body.Close()
dec := json.NewDecoder(resp.Body)
for dec.More() {
	var item Item
	dec.Decode(&item)
	process(item)
}
```

Constant memory regardless of response size.

### Proxies

`http.ProxyFromEnvironment` reads `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`. Manual:

```go
tr.Proxy = http.ProxyURL(mustParseURL("http://proxy.example.com:3128"))
```

## Standard Library Hooks

- `net.Dialer` — under `Transport.DialContext`.
- `crypto/tls` — `Transport.TLSClientConfig`.
- `net/http/cookiejar` — cookie jar.
- `net/http/httputil` — `DumpRequest`/`DumpResponse`.
- `net/http/httptest` — test server and recorder.

## Real-World Patterns

### 1. Reusable JSON client wrapper

```go
type API struct { c *http.Client; base string }

func (a *API) Get(ctx context.Context, path string, out any) error {
	req, _ := http.NewRequestWithContext(ctx, "GET", a.base+path, nil)
	resp, err := a.c.Do(req)
	if err != nil { return err }
	defer resp.Body.Close()
	if resp.StatusCode >= 400 {
		return fmt.Errorf("GET %s: status %d", path, resp.StatusCode)
	}
	return json.NewDecoder(resp.Body).Decode(out)
}
```

### 2. Retry with backoff and context

```go
func doWithRetry(ctx context.Context, c *http.Client, req *http.Request) (*http.Response, error) {
	delay := 200 * time.Millisecond
	for n := 0; n < 5; n++ {
		resp, err := c.Do(req)
		if err == nil && resp.StatusCode < 500 { return resp, nil }
		if resp != nil { io.Copy(io.Discard, resp.Body); resp.Body.Close() }
		select {
		case <-ctx.Done(): return nil, ctx.Err()
		case <-time.After(delay):
		}
		delay = min(delay*2, 5*time.Second)
	}
	return nil, errors.New("retry exhausted")
}
```

### 3. Streaming download with progress

```go
resp, _ := client.Do(req)
defer resp.Body.Close()
out, _ := os.Create("download.bin")
defer out.Close()

var read int64
buf := make([]byte, 32<<10)
for {
	n, err := resp.Body.Read(buf)
	if n > 0 {
		out.Write(buf[:n])
		read += int64(n)
		fmt.Printf("\r%d bytes", read)
	}
	if err == io.EOF { break }
	if err != nil { return err }
}
```

### 4. Rate-limited transport

```go
type rlTransport struct { rt http.RoundTripper; lim *rate.Limiter }

func (t *rlTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	if err := t.lim.Wait(req.Context()); err != nil { return nil, err }
	return t.rt.RoundTrip(req)
}

client := &http.Client{Transport: &rlTransport{
	rt: http.DefaultTransport,
	lim: rate.NewLimiter(10, 20),
}}
```

### 5. Mutual TLS

```go
cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
caPool := x509.NewCertPool()
ca, _ := os.ReadFile("ca.crt")
caPool.AppendCertsFromPEM(ca)

tr := &http.Transport{
	TLSClientConfig: &tls.Config{
		Certificates: []tls.Certificate{cert},
		RootCAs:      caPool,
	},
}
client := &http.Client{Transport: tr, Timeout: 30 * time.Second}
```

Use case: service-to-service auth in zero-trust networks.

## Anti-Patterns & Gotchas

**Using `http.Get` / `http.DefaultClient` in production.** No timeout.

**Constructing `http.Client` per request.** Loses connection pool.

**Forgetting `defer resp.Body.Close()`.** Connection leak.

**Returning with body not read to EOF.** Connection can't be reused.

**Treating a 500 status as success because `err == nil`.** Check `StatusCode`.

**`MaxIdleConnsPerHost = 2` (default) for an internal high-RPS API.** Raise it.

**Retry without `GetBody`.** Body can't be replayed.

**Hardcoded `InsecureSkipVerify: true`.** Disables TLS validation.

**Not setting `ResponseHeaderTimeout`.** Slow server can hold connections.

**Ignoring redirects carrying sensitive cookies.** Configure `CheckRedirect`.

## Performance Notes

- Connection reuse saves a TCP handshake (~1 RTT) and TLS handshake (~2 RTT) per request.
- HTTP/2 multiplexes streams; one connection per host suffices for high throughput.
- `Transport.DisableCompression = false` (default) auto-handles gzip.
- `WriteBufferSize`/`ReadBufferSize` tune for very large bodies.
- `crypto/tls` session resumption skips a handshake round-trip on subsequent connections.

## How Big Companies Use It

- **AWS SDK for Go v2** uses a shared `http.Client` per service with explicit timeouts and retry middleware.
- **Kubernetes client-go** tunes connection pools for apiserver workloads; mTLS via TLS config.
- **Cloudflare Workers (Go runtime)** uses custom transports for connection budgeting.
- **Tailscale** uses stdlib client with custom dialers routing through WireGuard or DERP relays.
- **HashiCorp Terraform** uses `cleanhttp.DefaultClient` to avoid sharing transports globally.

## Source Code References

Pinned to `go1.26`.

- `net/http` client: [`src/net/http/client.go`](https://github.com/golang/go/blob/master/src/net/http/client.go).
- `Transport`: [`src/net/http/transport.go`](https://github.com/golang/go/blob/master/src/net/http/transport.go).
- `Request`/`Response`: [`src/net/http/request.go`](https://github.com/golang/go/blob/master/src/net/http/request.go), `response.go`.
- HTTP/2 client: [`src/net/http/h2_bundle.go`](https://github.com/golang/go/blob/master/src/net/http/h2_bundle.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/net/http.
- Cloudflare, "The complete guide to Go net/http timeouts": https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/.
- "Don't use Go's default HTTP client": https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779.

## Exercises / Self-Check

1. Build a JSON-API client struct that takes a base URL and an `http.Client`. Implement `Get`, `Post`, `Put`, `Delete`.
2. Why does the connection pool not reuse a connection if you call `resp.Body.Close()` without reading to EOF? Trace through.
3. Implement retry with exponential backoff using `req.GetBody`. Test replay.
4. Tune `MaxIdleConnsPerHost`; observe load with `httptrace`.
5. Set up mutual TLS between client and server. Verify with `openssl s_client`.
