# HTTP Client Patterns — Transport Reuse, Retries, Circuit Breakers

## TL;DR

`http.Client` is **stateful**. It owns a `Transport` which owns a **connection pool**. Reusing the same client (or at minimum the same `Transport`) across all calls is the difference between 50k req/s and 500 req/s. Two-thirds of "Go is slow" reports trace to one of three mistakes: `http.Get` everywhere (uses `DefaultClient` and `DefaultTransport` — fine, but undiscoverable), constructing a new `http.Client` per request (defeats pooling), or `resp.Body.Close()` skipped on the error path (leaks connections). Resilience patterns layer on top: **idempotent retries with exponential backoff and jitter**, **circuit breakers** to fail fast when a downstream is dead, and **bounded timeouts at every level** — connect, TLS handshake, response header, total request. The single biggest gotcha: **`resp, err := http.Get(...)` and you only check `err != nil` — but on error, `resp` may still be non-nil and need its body closed**. Even after that, `_, _ = io.Copy(io.Discard, resp.Body)` before `Close` is required to make the connection reusable on the pool — otherwise it's a one-shot.

## Mental Model

```
   http.Client
   ───────────
       │
       │  .Timeout  ──── whole request budget
       │  .Transport
       ▼
   http.Transport
   ──────────────
       │
       │  DialContext            ── TCP connect
       │  TLSClientConfig        ── TLS settings
       │  MaxIdleConns           ── pool size total
       │  MaxIdleConnsPerHost    ── per-host limit
       │  MaxConnsPerHost        ── per-host concurrency cap
       │  IdleConnTimeout        ── how long pooled conns live idle
       │  TLSHandshakeTimeout    ── TLS handshake budget
       │  ResponseHeaderTimeout  ── time to first byte
       │  ExpectContinueTimeout  ── 100-Continue wait
       │  ForceAttemptHTTP2      ── ALPN h2 negotiation
       ▼
   Connection Pool (per host)
       │   conn ── reused on next request to the same host
       │   conn
       └── conn
```

Three invariants:

1. **One `Client` per service or globally.** Build once, pass around.
2. **`Transport` is the real engine.** Tuning lives here.
3. **`resp.Body.Close()` is mandatory, always, on every code path.** No exceptions.

## Bare Minimum (For Quick Scripts)

```go
resp, err := http.Get("https://example.com")
if err != nil { return err }
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body)   // drain so the conn returns to the pool
```

`http.Get` uses `http.DefaultClient` which uses `http.DefaultTransport`. Fine for one-offs. Don't use in libraries.

## Production Client

```go
package httpclient

import (
    "net"
    "net/http"
    "time"
)

var Transport = &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   5 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    MaxIdleConnsPerHost:   10,
    MaxConnsPerHost:       50,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   5 * time.Second,
    ResponseHeaderTimeout: 10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}

var Client = &http.Client{
    Transport: Transport,
    Timeout:   30 * time.Second,
}
```

What each knob means:

- **`Dialer.Timeout`** — how long to wait for TCP connect. Production: 2-5s.
- **`MaxIdleConns`** — global cap on idle pooled conns. Default 100; raise if many distinct hosts.
- **`MaxIdleConnsPerHost`** — default **2** (yes, two — almost certainly too low). Raise to 10-100 depending on concurrency.
- **`MaxConnsPerHost`** — total concurrent connections per host. Default unlimited; cap to limit fan-out.
- **`IdleConnTimeout`** — how long an unused pooled conn lives. Default 90s; matches typical LB idle timeouts.
- **`TLSHandshakeTimeout`** — TLS handshake budget. 5s is generous.
- **`ResponseHeaderTimeout`** — time to receive response headers (TTFB). Often the right place to fail fast.
- **`ForceAttemptHTTP2`** — opt-in to h2 when ALPN agrees. Default `true` for HTTPS in modern Go.

The most-commonly-wrong knob: **`MaxIdleConnsPerHost = 2`**. Under any concurrency, you constantly tear down and re-establish connections to the same host — death by handshake.

## Per-Request Context (Cancellation)

```go
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

Always `NewRequestWithContext`. The `ctx` cancels the in-flight read/write, releases the goroutine, and is propagated by `otelhttp`/middleware for tracing.

`Client.Timeout` is a separate, outer deadline. The minimum of `ctx` deadline and `Client.Timeout` wins. Use `ctx` for per-call budgets; `Client.Timeout` for the absolute upper bound.

## The Body-Close Footgun

```go
// WRONG — leaks the connection
resp, err := client.Do(req)
if err != nil {
    return err
}
if resp.StatusCode != 200 {
    return errBad  // body never closed!
}
defer resp.Body.Close()
```

Fix: defer immediately after error check.

```go
resp, err := client.Do(req)
if err != nil { return err }
defer resp.Body.Close()
if resp.StatusCode != 200 {
    return errBad
}
```

But that's still not enough — if you don't read the body, the connection can't be reused:

```go
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

Or, for the common pattern of "I want to read until I decide it's an error":

```go
defer resp.Body.Close()
// ...
if resp.StatusCode != 200 {
    io.Copy(io.Discard, resp.Body)   // drain explicitly
    return errBad
}
```

The drain matters because the `Transport` returns a connection to the pool *only* when the response body is fully consumed and closed. A partial read + close = closed connection = a fresh TCP+TLS handshake next request.

### Body-on-error subtlety

Even when `client.Do` returns `err != nil`, `resp` may be non-nil (e.g., redirect handling fails after a 30x). Defensive:

```go
resp, err := client.Do(req)
if resp != nil {
    defer resp.Body.Close()
}
if err != nil {
    return err
}
```

## Reading and Capping Bodies

```go
body, err := io.ReadAll(io.LimitReader(resp.Body, 10<<20)) // 10 MB cap
```

Without the limit, a malicious or buggy upstream can OOM you.

For JSON:

```go
dec := json.NewDecoder(io.LimitReader(resp.Body, 10<<20))
dec.DisallowUnknownFields()
err := dec.Decode(&out)
```

For large streams, decode incrementally — don't `ReadAll`.

## Retries (Idempotent Only)

```go
import (
    "math/rand/v2"
    "time"
)

func doWithRetry(ctx context.Context, client *http.Client, req *http.Request) (*http.Response, error) {
    const maxAttempts = 4
    var lastErr error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(1<<attempt) * 100 * time.Millisecond
            jitter := time.Duration(rand.Int64N(int64(backoff)))
            select {
            case <-time.After(backoff + jitter):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        reqClone := req.Clone(ctx)
        resp, err := client.Do(reqClone)
        if err != nil {
            lastErr = err
            if !isRetryable(err) { return nil, err }
            continue
        }
        if shouldRetryStatus(resp.StatusCode) {
            io.Copy(io.Discard, resp.Body)
            resp.Body.Close()
            lastErr = fmt.Errorf("status %d", resp.StatusCode)
            continue
        }
        return resp, nil
    }
    return nil, fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}

func shouldRetryStatus(code int) bool {
    return code == 429 || code == 502 || code == 503 || code == 504
}

func isRetryable(err error) bool {
    var ne net.Error
    if errors.As(err, &ne) && ne.Timeout() {
        return true
    }
    if errors.Is(err, io.EOF) || errors.Is(err, io.ErrUnexpectedEOF) {
        return true   // server closed mid-response
    }
    return false
}
```

Rules:

- **Only retry idempotent methods** — GET, HEAD, PUT (idempotent if writes are designed that way), DELETE. Never POST unless you carry an idempotency key.
- **Exponential backoff with jitter** — without jitter, retries from thousands of clients synchronise into a thundering herd.
- **Cap total attempts AND total time** — a long retry loop turns a transient downstream blip into a long client-side hang.
- **Respect `Retry-After`** header on 429/503:
  ```go
  if d := resp.Header.Get("Retry-After"); d != "" { /* parse, sleep */ }
  ```
- **Distinguish "request never sent" from "request maybe sent."** A dial failure means safe to retry; a write failure mid-request means the server *may* have processed it — retry only with an idempotency key.

## Idempotency Keys

For non-idempotent endpoints that you still want to retry safely:

```go
req.Header.Set("Idempotency-Key", uuid.NewString())
```

Server-side, dedup by key + body hash within a TTL. Stripe popularised this pattern; most modern payment/financial APIs support it.

## Circuit Breakers

When a downstream is dead, retrying is harmful — you pile up timeouts. Circuit breakers short-circuit calls to a known-bad target.

States:

- **Closed** — calls pass through; failures counted.
- **Open** — all calls fail fast (e.g., return `ErrCircuitOpen`); no network.
- **Half-Open** — after cooldown, allow one trial; success → Closed; failure → Open.

Sketch:

```go
type Breaker struct {
    mu          sync.Mutex
    state       int   // 0=closed, 1=open, 2=half
    failures    int
    threshold   int
    openedAt    time.Time
    cooldown    time.Duration
}

func (b *Breaker) Allow() bool {
    b.mu.Lock(); defer b.mu.Unlock()
    switch b.state {
    case 0: return true
    case 1:
        if time.Since(b.openedAt) > b.cooldown {
            b.state = 2; return true
        }
        return false
    case 2: return true
    }
    return false
}

func (b *Breaker) Success() {
    b.mu.Lock(); defer b.mu.Unlock()
    b.failures = 0; b.state = 0
}

func (b *Breaker) Failure() {
    b.mu.Lock(); defer b.mu.Unlock()
    b.failures++
    if b.state == 2 || b.failures >= b.threshold {
        b.state = 1; b.openedAt = time.Now()
    }
}
```

Libraries: `sony/gobreaker` (clean, popular), `failsafe-go` (modern, generic-typed). Rolling-window failure counts are often better than fixed thresholds.

## Rate Limiting (Client Side)

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(100), 10)  // 100/s, burst 10

ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
if err := limiter.Wait(ctx); err != nil {
    return err
}
// send request
```

Pair with circuit breaker: rate limit *normal* operation, breaker *failure* mode.

## Connection Pool Tuning by Workload

| Workload | `MaxIdleConnsPerHost` | `MaxConnsPerHost` | `IdleConnTimeout` |
|----------|----------------------|-------------------|-------------------|
| Low-rate (a few req/s) | 2 (default fine) | unlimited | 90s |
| Internal RPC, high-rate | 100-500 | 500-1000 | 60-120s |
| Many distinct hosts (proxy/scraper) | 5-10 | 20 | 30-60s |
| Single downstream, bursty | match peak concurrency | 1.5x peak | 90s |

Profile: if `netstat`/`ss` shows lots of `TIME_WAIT`, you're tearing down too aggressively. If you see `Connection refused` under burst, your `MaxConnsPerHost` is too low or the downstream is rate-limiting connections.

## HTTP/2 Specifics

Default Go client negotiates HTTP/2 over HTTPS via ALPN. One TCP connection multiplexes many streams. Implications:

- **Per-host connection count is typically 1** for HTTP/2 — multiplexing replaces parallel connections.
- **`MaxConcurrentStreams`** server setting bounds in-flight streams per connection.
- **`Transport.DisableKeepAlives = true` forces HTTP/1.1**, defeating multiplexing. Don't.
- **Server-side flow control** can stall heavy uploads. Tune `http2.Server.MaxReadFrameSize` if you push large payloads.

For HTTP/3, see `09-http3-quic.md`.

## Anti-Patterns & Gotchas

**`http.Get` everywhere with no timeout.** `http.DefaultClient.Timeout == 0` = no overall timeout. Every call can hang forever.

**Building a fresh `http.Client` per call.** Connection pool is per-Transport. New Transport = no pool benefit.

**Forgetting `resp.Body.Close()` on error path.** Connection leak; eventually file-descriptor exhaustion or pool starvation.

**Not draining the body.** Connection cannot be reused. You think you have a pool; you have many short-lived TCP connections.

**`MaxIdleConnsPerHost = 2` (the default)** in a high-concurrency service. The most common silent perf bug.

**Retrying POSTs without idempotency keys.** Double-charges, double-emails.

**Retry storms after a downstream restart.** No circuit breaker = thousands of clients hammering on cold caches.

**Using `Client.Timeout` without per-request `ctx`.** No way to cancel a slow call from outside.

**`req.Body = ...` reused across retries.** `client.Do` consumes the body. Use `req.GetBody = func() ...` so the transport can re-create the body on retry.

**Logging request URLs with credentials.** `https://user:pass@host/path` — strip before log.

**Trusting `http.Response.ContentLength`.** Can be `-1` (unknown for chunked).

**Using `httptest.NewServer` for unit tests but not propagating the real `http.Client` semantics.** Tests pass; prod fails. Test with `httptest` *and* with a real network simulator (`gomock`, `httpmock`, `nethttp.ServeMux` in tests).

**Skipping `req.Close = false`.** Default. Setting `Close = true` makes every request use a one-shot conn — opt-in only when you want it.

**Forgetting that `Transport.CloseIdleConnections()` exists.** Useful when rotating credentials/IPs.

**Naked `client.Do(req)` in a library** without taking a `context.Context` from the caller. Always thread context.

## Performance Notes

| Operation | Cost |
|-----------|------|
| TCP connect (same DC) | 0.5-2 ms |
| TLS handshake (TLS 1.3, 1-RTT) | 1-3 ms |
| TLS resumption (session ticket) | sub-ms |
| HTTP/1.1 request, pooled conn | 0.5-5 ms RTT-bound |
| HTTP/2 stream over existing conn | RTT-bound + ~10 µs framing |
| `client.Do` overhead in Go | ~50 µs (excluding network) |
| Cold pool, 100 concurrent → same host | bursts of dial + handshake; expect 100ms+ |

Connection reuse savings: a cold call costs 5-10 ms of overhead (TCP + TLS); a pooled call costs ~RTT only. At 10k req/s, that's the difference between "fine" and "fire."

## How Big Companies Use It

- **Cloudflare** runs millions of outbound calls/sec; their internal `httpclient` wrappers tune `Transport` aggressively and use circuit breakers extensively.
- **Stripe** popularised idempotency keys for HTTP retries; their Go SDK exposes the pattern.
- **AWS SDK for Go** v2 ships its own retry/circuit logic; supports adaptive retries with token-bucket per host.
- **Tailscale** uses a single global `http.Client` per binary, threaded via DI.
- **Caddy** uses `net/http` with per-upstream connection pools tuned by deployment.
- **Kubernetes client-go** has been the de facto reference for "well-tuned outbound HTTP" — circuit breaking, rate limiting, exponential backoff.
- **Datadog's `dd-trace-go`** wraps the standard `http.RoundTripper` for tracing, preserving pool semantics.

## Source Code References

- `net/http.Client`: https://github.com/golang/go/blob/master/src/net/http/client.go.
- `net/http.Transport`: https://github.com/golang/go/blob/master/src/net/http/transport.go.
- `net/http/httputil.ReverseProxy`: https://github.com/golang/go/blob/master/src/net/http/httputil/reverseproxy.go.
- `golang.org/x/time/rate`: https://github.com/golang/time.
- `sony/gobreaker`: https://github.com/sony/gobreaker.
- `failsafe-go`: https://github.com/failsafe-go/failsafe-go.
- AWS SDK v2 retry: https://github.com/aws/aws-sdk-go-v2/tree/main/aws/retry.
- HTTP/2 in Go: https://github.com/golang/go/tree/master/src/net/http and `golang.org/x/net/http2`.

## Further Reading

- "The complete guide to Go net/http timeouts" (Cloudflare blog).
- Filippo Valsorda, "So you want to expose Go on the Internet": https://blog.cloudflare.com/exposing-go-on-the-internet/.
- "Building resilient services in Go" (DoorDash blog).
- "Failure modes of HTTP retries" (Marc Brooker, AWS).
- "The Calculus of Service Availability" (Treynor et al., Google SRE).
- "Idempotency keys" (Stripe Engineering blog).
- "Circuit breakers and rate limiters" (Sony Engineering blog).

## Exercises / Self-Check

1. Build a global `http.Client` with tuned Transport. Make 1000 concurrent calls to one host; compare connection count (via `netstat`) with default vs tuned.
2. Skip `resp.Body.Close()` on the error path; run a load test; confirm file-descriptor count grows.
3. Skip the `io.Copy(io.Discard, resp.Body)` drain; measure connection reuse rate via `Transport.MaxIdleConnsPerHost` saturation.
4. Add exponential backoff with jitter. Simulate a flaky upstream (50% 503); verify retries succeed and overall latency is bounded.
5. Implement a circuit breaker. Take a downstream offline; confirm the breaker opens within N failures and short-circuits subsequent calls.
6. Use `golang.org/x/time/rate` to cap outbound at 100 req/s. Confirm under burst that calls block but don't drop.
7. Issue an HTTP/2 client request to an h2 server. Inspect with Wireshark; confirm a single TCP conn carries many streams.
8. Add `Idempotency-Key` to a POST retry. Build a server that dedupes; confirm a retried POST executes exactly once.
