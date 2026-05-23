# HTTP/3 and QUIC — `quic-go`

## TL;DR

HTTP/3 is the third major version of HTTP, running over **QUIC** (a new transport built on UDP with TLS 1.3 baked in) instead of TCP+TLS. Key benefits over HTTP/2-over-TCP: **no TCP head-of-line blocking** (QUIC streams are independent at the transport layer; one slow stream doesn't stall others); **0-RTT connection setup** (resume a prior connection without round trips); **connection migration** (your conn ID survives an IP change — phone switching from WiFi to LTE doesn't drop streams); **encrypted transport headers** (not just payload). The Go ecosystem implements HTTP/3 via **`quic-go/quic-go`** (the only serious option) and its companion **`quic-go/http3`** package. The stdlib `net/http` does **not** speak HTTP/3 as of Go 1.26; there's an experimental package in `x/net/http3` tracking acceptance. The single biggest gotcha: **UDP/QUIC is blocked or deprioritised by many corporate firewalls and middleboxes**. Always serve HTTP/3 alongside HTTP/2 — advertise via the `Alt-Svc` header and let clients downgrade gracefully. Operational cost: QUIC servers use significantly more CPU than HTTP/2 (~2-3x for crypto) because there's no kernel offload yet.

## Mental Model

```
   Old stack (HTTP/1.1, HTTP/2)
   ─────────────────────────────
   Application: HTTP semantics
   TLS:         TLS 1.2 / 1.3
   Transport:   TCP — single ordered byte stream, kernel-managed
   Network:     IP

   New stack (HTTP/3)
   ──────────────────
   Application: HTTP semantics
   QUIC:        - encrypts transport headers (TLS 1.3 inside)
                - multiple independent streams (no HoL blocking)
                - connection migration via Connection ID
                - 0-RTT resumption
                - kernel-bypass: all in userspace today
   Transport:   UDP
   Network:     IP
```

Two key implications:

1. **All TLS happens inside QUIC.** No separate TLS layer; QUIC handshake = TLS 1.3 handshake.
2. **Userspace transport.** This is why CPU is higher than TCP+TLS — the kernel can't do AES offload across UDP packets the same way.

## When to Use HTTP/3

- **Mobile clients** with frequent network changes (WiFi ↔ LTE). Connection migration is a real win.
- **High-RTT links** (3G, satellite, transcontinental). 0-RTT and reduced handshake matter.
- **Lossy networks** where HTTP/2-over-TCP's HoL blocking hurts (mobile, congested Wi-Fi).
- **Anti-censorship / firewall traversal** scenarios — QUIC over 443/udp is harder to fingerprint than TCP+TLS.

When NOT:

- **Internal DC traffic**. Latency is sub-ms, no packet loss, kernel TCP+TLS is faster. Use HTTP/2.
- **CPU-constrained backends**. QUIC's CPU overhead is the deal-breaker.
- **Behind a strict corporate firewall**. UDP/443 often blocked.
- **You don't already need TLS 1.3 only**. HTTP/3 requires it; 1.2 isn't an option.

## Setup

```bash
go get github.com/quic-go/quic-go/http3
```

```go
import (
    "github.com/quic-go/quic-go/http3"
)
```

`quic-go` ships under several modules:

- `github.com/quic-go/quic-go` — the QUIC implementation.
- `github.com/quic-go/quic-go/http3` — HTTP/3 over QUIC.
- `github.com/quic-go/qpack` — header compression (QPACK, the HTTP/2-HPACK successor).

## Minimal HTTP/3 Server

```go
package main

import (
    "crypto/tls"
    "log"
    "net/http"

    "github.com/quic-go/quic-go/http3"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        proto := r.Proto                     // HTTP/3.0
        w.Header().Set("Alt-Svc", `h3=":443"; ma=86400`)
        w.Write([]byte("hello over " + proto + "\n"))
    })

    cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
    cfg := &tls.Config{
        Certificates: []tls.Certificate{cert},
        NextProtos:   []string{"h3"},          // ALPN identifier for HTTP/3
    }

    srv := &http3.Server{
        Addr:      ":443",
        Handler:   mux,
        TLSConfig: cfg,
    }
    log.Fatal(srv.ListenAndServe())
}
```

The handler is a regular `http.Handler` — same interface as HTTP/1.1 and HTTP/2. Path params, middleware, response writing — all standard.

`http3.Server` binds a **UDP** listener (vs `http.Server`'s TCP). You typically want both protocols:

## Serve HTTP/2 + HTTP/3 Simultaneously

```go
mux := http.NewServeMux()
mux.HandleFunc("/", handler)

cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
tlsCfg := &tls.Config{Certificates: []tls.Certificate{cert}}

// HTTP/3 over UDP/443
h3srv := &http3.Server{
    Addr:      ":443",
    Handler:   addAltSvc(mux),
    TLSConfig: &tls.Config{
        Certificates: tlsCfg.Certificates,
        NextProtos:   []string{"h3"},
    },
}

// HTTP/1.1 + HTTP/2 over TCP/443
h2srv := &http.Server{
    Addr:      ":443",
    Handler:   addAltSvc(mux),
    TLSConfig: &tls.Config{
        Certificates: tlsCfg.Certificates,
        NextProtos:   []string{"h2", "http/1.1"},
    },
}

go h3srv.ListenAndServe()
go h2srv.ListenAndServeTLS("", "")

select {}
```

`addAltSvc` middleware advertises HTTP/3 on every response:

```go
func addAltSvc(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Alt-Svc", `h3=":443"; ma=86400, h2=":443"; ma=86400`)
        h.ServeHTTP(w, r)
    })
}
```

Browsers and modern clients see `Alt-Svc`, remember it, and prefer HTTP/3 on subsequent requests. First request is always over HTTP/2; subsequent ones can be over HTTP/3.

## HTTP/3 Client

```go
import "github.com/quic-go/quic-go/http3"

client := &http.Client{
    Transport: &http3.RoundTripper{
        TLSClientConfig: &tls.Config{
            // standard TLS config
        },
    },
}

resp, err := client.Get("https://example.com/")
```

The `http3.RoundTripper` is a drop-in replacement for `http.Transport`'s role in `http.Client`. From the application's perspective, nothing else changes.

For a client that opportunistically upgrades to HTTP/3 after seeing `Alt-Svc`:

```go
// Use github.com/quic-go/quic-go/http3 with the experimental "AltSvc" aware
// roundtripper, OR use a wrapping RoundTripper:

type AltSvcRoundTripper struct {
    h2 http.RoundTripper
    h3 http.RoundTripper
    altSvc sync.Map // host -> "h3"
}

func (r *AltSvcRoundTripper) RoundTrip(req *http.Request) (*http.Response, error) {
    if _, ok := r.altSvc.Load(req.URL.Host); ok {
        return r.h3.RoundTrip(req)
    }
    resp, err := r.h2.RoundTrip(req)
    if err != nil { return nil, err }
    if as := resp.Header.Get("Alt-Svc"); strings.Contains(as, "h3=") {
        r.altSvc.Store(req.URL.Host, "h3")
    }
    return resp, nil
}
```

## Streams in QUIC

QUIC has two stream types:

- **Bidirectional streams** — request-response (HTTP/3 uses these).
- **Unidirectional streams** — control / push.

The handler-level abstraction (`http.Handler` over `http3.Server`) hides this. To use QUIC directly for custom protocols:

```go
listener, _ := quic.ListenAddr("0.0.0.0:443", tlsCfg, nil)
for {
    sess, _ := listener.Accept(context.Background())
    go func(sess quic.Connection) {
        for {
            stream, _ := sess.AcceptStream(context.Background())
            go handleStream(stream)
        }
    }(sess)
}
```

`quic.Stream` is a `net.Conn`-like duplex. Each stream has independent flow control. Useful for custom RPC or WebTransport-like APIs.

## Performance Tuning

```go
qcfg := &quic.Config{
    MaxIncomingStreams:    1000,
    MaxIncomingUniStreams: 100,
    MaxIdleTimeout:        30 * time.Second,
    KeepAlivePeriod:       15 * time.Second,
    InitialStreamReceiveWindow: 512 * 1024,
    MaxStreamReceiveWindow:     6 * 1024 * 1024,
}

h3srv := &http3.Server{
    Addr:       ":443",
    Handler:    mux,
    TLSConfig:  tlsCfg,
    QUICConfig: qcfg,
}
```

- **`KeepAlivePeriod`** — send keep-alive PINGs; mandatory to prevent NAT timeout (UDP-NAT timeouts can be as short as 30s).
- **Receive windows** — analogous to HTTP/2 flow control; bump for high-BDP (long-fat) networks.
- **`MaxIncomingStreams`** — DoS protection cap.

UDP send-buffer tuning at the OS level matters:

```bash
sysctl -w net.core.rmem_max=2500000
sysctl -w net.core.wmem_max=2500000
```

Without this, packet drops at the receiver can throttle QUIC throughput.

## 0-RTT Resumption

```go
// Client side
qcfg := &quic.Config{
    EnableEarlyData: true,
}

// First connection — full 1-RTT handshake.
// Subsequent connections (with session ticket) — 0-RTT
// Application data sent in the very first packet.
```

Caveat: **0-RTT data is replayable**. An attacker who captures it can resend it. Servers must treat 0-RTT data as potentially-replayed and reject anything non-idempotent. Most servers default to "0-RTT for GET only; other methods downgrade to 1-RTT."

## Connection Migration

QUIC connections are identified by a **Connection ID**, not (IP, port). A client roaming from WiFi to LTE keeps the same conn ID; the server sees the source IP change and updates its routing table.

Migration works *out of the box* with `quic-go` for client-driven migration. Server-side roaming (server's IP changes) is rarer and less mature.

## QPACK (Header Compression)

HTTP/2 uses HPACK; HTTP/3 uses QPACK (https://www.rfc-editor.org/rfc/rfc9204) — designed to be safe under reordering. Don't worry about it; `quic-go` handles it.

## WebTransport (Browser ↔ Server Streams)

WebTransport is a new browser API for bidirectional streams over HTTP/3. Like WebSocket but multiplexed: a browser can have many independent streams to a server in one QUIC connection.

```go
import "github.com/quic-go/webtransport-go"

s := &webtransport.Server{
    H3: http3.Server{TLSConfig: tlsCfg},
}
mux.HandleFunc("/wt", func(w http.ResponseWriter, r *http.Request) {
    conn, _ := s.Upgrade(w, r)
    for {
        stream, _ := conn.AcceptStream(r.Context())
        // handle
    }
})
```

Still experimental in browsers but landing across Chrome/Edge/Firefox in 2024-2025. The pure-Go server side is production-ready via `webtransport-go`.

## Anti-Patterns & Gotchas

**Serving HTTP/3 only.** Many networks block UDP/443. Always pair with HTTP/2 + `Alt-Svc`.

**Skipping `KeepAlivePeriod`.** NAT entries expire (often 30s for UDP); connections die silently.

**Treating 0-RTT as safe for any method.** Replayable. Allow only idempotent operations.

**Forgetting `SO_REUSEPORT` for multi-core scaling.** A single `http3.Server` instance is single-threaded on the listener loop; throughput is gated.

**No `QUICConfig.MaxIncomingStreams` limit.** DoS via stream flooding.

**Logging connection ID without redaction.** It's an identity hint for tracking.

**Running QUIC behind an L4 LB that doesn't know about Connection IDs.** Packets to the same conn may get routed to different backends. Use an L7 LB (Envoy, HAProxy with QUIC support) or sticky-by-conn-ID.

**Assuming HTTP/3 means TCP is dead.** Most internal traffic stays on TCP+HTTP/2 for the foreseeable future.

**Trusting QUIC-only firewalls to block QUIC.** Encrypted handshake; deep packet inspection is hard. Don't rely on FW rules alone.

**Using QUIC streams as application-level connections.** Streams are cheap (kilobytes), but they're not free. Pool / reuse like HTTP/2.

**Forgetting Go's `quic-go` is a separate library, not stdlib.** Stability and API changes are tracked there, not in Go's release notes.

**Running QUIC in a CGo-heavy binary.** Crypto cost is real; CGo on top makes it worse.

**Not measuring CPU.** TCP+TLS is hardware-accelerated; QUIC isn't (yet). Plan for 2-3x CPU.

## Performance Notes

(Modern hardware, single core, large transfers.)

| Metric | HTTP/2 over TCP+TLS | HTTP/3 over QUIC |
|--------|---------------------|-------------------|
| Handshake (cold) | 2 RTT (TCP + TLS 1.3) | 1 RTT (combined) |
| Handshake (warm/resume) | 0-1 RTT | 0 RTT |
| Throughput per stream | ~2-3 GB/s | ~0.5-1 GB/s |
| Throughput aggregate | ~5-10 GB/s (multi-core) | ~1-3 GB/s |
| CPU at 1 GB/s | ~20-30% one core | ~50-80% one core |
| Memory per conn | ~50 KB | ~100 KB |
| Loss tolerance | Per-conn HoL | Per-stream — no HoL |

Linux kernels are gaining UDP GSO/GRO and AES offload paths for QUIC; expect the gap to narrow through 2026-27.

## How Big Companies Use It

- **Google** invented QUIC and runs it at massive scale (Search, YouTube, Gmail). The Go reference impl is `quic-go`, separate from Google's internal C++ impl.
- **Cloudflare** has supported QUIC + HTTP/3 since 2019. Their Go services use `quic-go` selectively.
- **Akamai**, **Fastly** offer HTTP/3 across their CDNs.
- **Meta** uses QUIC (`mvfst` — C++) for Facebook/Instagram; some Go services use `quic-go` for tooling.
- **Apple** uses QUIC for FaceTime / iCloud (Network.framework — Swift/C; not Go).
- **WebTransport-go** is used by streaming/media startups and some game backends.
- **AdGuard / Tailscale** experiment with QUIC for their tunnels.
- **Caddy** server has HTTP/3 support via `quic-go`.

In the Go ecosystem specifically, HTTP/3 adoption is **opportunistic** — most teams keep HTTP/2 as the primary and add HTTP/3 for mobile-heavy or edge-CDN scenarios.

## Source Code References

- `quic-go`: https://github.com/quic-go/quic-go.
- HTTP/3 RFC 9114: https://www.rfc-editor.org/rfc/rfc9114.html.
- QUIC RFC 9000: https://www.rfc-editor.org/rfc/rfc9000.html.
- QPACK RFC 9204: https://www.rfc-editor.org/rfc/rfc9204.html.
- TLS 1.3 RFC 8446: https://www.rfc-editor.org/rfc/rfc8446.html.
- WebTransport draft: https://www.ietf.org/archive/id/draft-ietf-webtrans-overview-04.html.
- `webtransport-go`: https://github.com/quic-go/webtransport-go.
- Cloudflare's `quiche` (Rust, but documents QUIC well): https://github.com/cloudflare/quiche.

## Further Reading

- "HTTP/3 explained": https://http3-explained.haxx.se/ (Daniel Stenberg, curl maintainer).
- Cloudflare blog tag: https://blog.cloudflare.com/tag/quic/.
- Robin Marx's QUIC blog: https://calendar.perfplanet.com/2020/quic-and-http-3-performance/.
- "QUIC for the Application Developer": https://datatracker.ietf.org/doc/html/draft-ietf-quic-applicability-00.
- "Real-Time Communication with WebRTC vs QUIC" (various 2023-24 talks).
- Google's QUIC overview: https://www.chromium.org/quic/.
- "How HTTP/3 works at high level" — MDN.

## Exercises / Self-Check

1. Set up a minimal `http3.Server` with a self-signed cert. Verify HTTP/3 with `curl --http3 -k https://localhost:443/`.
2. Add `Alt-Svc` advertisement on an HTTP/2 server. From a browser, verify the next page load uses HTTP/3 (check DevTools → Network → Protocol).
3. Tune `MaxIncomingStreams` and `KeepAlivePeriod`. Stress-test with `h3spec` or `quiche-client`.
4. Measure CPU at 1 Gbit/s on HTTP/2 vs HTTP/3 on the same hardware. Quantify the QUIC overhead.
5. Simulate mobile network migration: change the client's source IP mid-stream. Confirm the QUIC connection survives via Connection ID.
6. Enable 0-RTT for GET requests only. Send a POST via 0-RTT; confirm the server rejects or downgrades.
7. Build a small custom protocol on raw `quic.Stream`. Compare framing/code complexity vs WebSocket.
8. Deploy HTTP/3 behind a UDP-aware L4 LB. Test that connections survive an LB backend rotation (sticky-by-Conn-ID).
