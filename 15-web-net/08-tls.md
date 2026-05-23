# TLS — `crypto/tls`, ALPN, mTLS

## TL;DR

`crypto/tls` is the Go standard library's TLS 1.2/1.3 implementation — pure Go, audited, default for `net/http`'s HTTPS. Go 1.26 ships TLS 1.3 by default, AES-GCM + ChaCha20-Poly1305 only, X25519 + ML-KEM-768 (post-quantum hybrid) curve preferences, and stricter session ticket rotation. The mental model is a small struct, **`*tls.Config`**, that you attach to a server (`Server.TLSConfig`) or a transport (`http.Transport.TLSClientConfig`). Five disciplines distinguish working TLS from broken-in-prod TLS: **minimum TLS version 1.2 (1.3 preferred)** — never 1.0/1.1; **automated certificate management** via ACME (Let's Encrypt) rather than year-long manual renewals; **ALPN negotiation** for HTTP/2 (the entire reason your `http.Transport` can do h2 over HTTPS); **mTLS** for service-to-service auth in zero-trust meshes; and **never** `InsecureSkipVerify: true` in production. The single biggest gotcha: **shipping with self-signed certs in dev and forgetting to verify them in prod-bound code**. The fix is making `InsecureSkipVerify` a compile-time-guarded toggle that fails CI if it's true on a non-test build.

## Mental Model

```
   ┌─────────────────────────────┐
   │  *tls.Config                 │
   │  ────────────                │
   │  MinVersion: TLS12+          │
   │  Certificates / GetCert      │
   │  ClientAuth (mTLS)           │
   │  ClientCAs                   │
   │  RootCAs                     │
   │  ServerName (SNI)            │
   │  NextProtos (ALPN)           │
   │  CurvePreferences            │
   │  CipherSuites (1.2 only)     │
   └─────────────────────────────┘
            │
            │ attach to
            ▼
   Server side                       Client side
   ───────────                       ───────────
   http.Server.TLSConfig             http.Transport.TLSClientConfig
   tls.Listen(...)                   tls.Dial(...)
```

Two invariants worth remembering:

1. **`*tls.Config` is shared across many connections.** Don't mutate after first use; clone before changing.
2. **Hostname verification is critical.** Without it, anyone with a valid cert (for any name) can impersonate your server. `tls.Config.ServerName` (set automatically by `tls.Dial`) is what makes verification meaningful.

## Server: HTTPS in 4 Lines

```go
srv := &http.Server{
    Addr:    ":443",
    Handler: handler,
}
log.Fatal(srv.ListenAndServeTLS("server.crt", "server.key"))
```

Defaults (Go 1.26): TLS 1.2 minimum, TLS 1.3 preferred, modern cipher suites only, HTTP/2 via ALPN.

## Server: Explicit Config

```go
cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
if err != nil { log.Fatal(err) }

cfg := &tls.Config{
    MinVersion: tls.VersionTLS13,
    Certificates: []tls.Certificate{cert},
    CurvePreferences: []tls.CurveID{
        tls.X25519MLKEM768,   // 1.24+: post-quantum hybrid
        tls.X25519,
        tls.CurveP256,
    },
    NextProtos: []string{"h2", "http/1.1"},
}

srv := &http.Server{
    Addr:      ":443",
    Handler:   handler,
    TLSConfig: cfg,
}
srv.ListenAndServeTLS("", "")  // empty strings — cert comes from cfg
```

Set `MinVersion` explicitly — even though defaults are good, you want the audit trail in code.

## Multiple Certificates / SNI

`GetCertificate` is the dynamic hook:

```go
cfg := &tls.Config{
    GetCertificate: func(hello *tls.ClientHelloInfo) (*tls.Certificate, error) {
        switch hello.ServerName {
        case "api.example.com":
            return &apiCert, nil
        case "admin.example.com":
            return &adminCert, nil
        default:
            return &defaultCert, nil
        }
    },
}
```

The client's SNI (Server Name Indication) is the `ServerName` field. Used for virtual hosting on a single IP.

## Automated Certificates: `autocert` (Let's Encrypt)

```go
import "golang.org/x/crypto/acme/autocert"

m := &autocert.Manager{
    Prompt:     autocert.AcceptTOS,
    HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
    Cache:      autocert.DirCache("/var/lib/autocert"),
    Email:      "ops@example.com",
}

srv := &http.Server{
    Addr:      ":443",
    Handler:   handler,
    TLSConfig: m.TLSConfig(),
}

// HTTP-01 challenge listener (redirects everything else to HTTPS)
go http.ListenAndServe(":80", m.HTTPHandler(nil))

log.Fatal(srv.ListenAndServeTLS("", ""))
```

`autocert.Manager`:

- Issues a certificate the first time SNI hits a whitelisted hostname.
- Renews 30 days before expiry.
- Stores certs in `DirCache` (or use `S3`, `redis` adapters).
- Uses HTTP-01 challenge by default (TLS-ALPN-01 also supported via `m.TLSConfig().GetCertificate`).

For multi-instance deployments: share the cache. `autocert.DirCache` on a shared filesystem (EFS, GCS Fuse) or a Redis-backed cache.

For richer ACME needs (DNS-01, wildcard certs), use **Caddy** as a frontend or **lego** (https://github.com/go-acme/lego) inside your app.

## TLS Versions

```go
tls.VersionTLS10  // disabled in Go 1.22+ by default
tls.VersionTLS11  // disabled in Go 1.22+ by default
tls.VersionTLS12  // good
tls.VersionTLS13  // preferred
```

Don't enable 1.0/1.1; they have known weaknesses (BEAST, POODLE) and are removed from major browsers.

```go
cfg := &tls.Config{
    MinVersion: tls.VersionTLS12,
    MaxVersion: tls.VersionTLS13,  // optional — usually leave unset
}
```

For new public-facing services in 2026, **`MinVersion: tls.VersionTLS13`** is the right baseline unless a known client requires 1.2.

## Cipher Suites

TLS 1.3 ciphers are not configurable (intentional — the IETF removed all the foot-guns). Go uses:
- `TLS_AES_128_GCM_SHA256`
- `TLS_AES_256_GCM_SHA384`
- `TLS_CHACHA20_POLY1305_SHA256`

TLS 1.2 ciphers are configurable but Go's defaults are sane:

```go
cfg.CipherSuites = []uint16{
    tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
    tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
    tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
    tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
    tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
}
```

Mostly leave `CipherSuites: nil` and let Go pick. The defaults track current best practice.

## ALPN — Application-Layer Protocol Negotiation

ALPN is how HTTP/2 (and HTTP/3 over QUIC) is selected: during the TLS handshake, the client advertises supported protocols (`h2`, `http/1.1`), the server picks one.

```go
cfg.NextProtos = []string{"h2", "http/1.1"}
```

Server picks the *first* protocol it supports that the client offered. Without `NextProtos`, you get HTTP/1.1.

ALPN is also how gRPC negotiates inside TLS: `h2` (gRPC over HTTP/2 over TLS). And how HTTP/3 (`h3`) is negotiated over QUIC.

## Client-Side TLS

```go
tr := &http.Transport{
    TLSClientConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
        RootCAs:    customCAPool,        // override system trust store
        ServerName: "api.example.com",   // override if dialing by IP
    },
    ForceAttemptHTTP2: true,
}
client := &http.Client{Transport: tr}
```

Three things to know:

- **`RootCAs`** — if `nil`, system CA store. Use to pin to a custom CA (e.g., internal PKI).
- **`ServerName`** — auto-set from URL, but override if you dial by IP.
- **`InsecureSkipVerify: true`** — **don't**. See below.

## Certificate Pinning

For high-assurance scenarios, pin the server's certificate or public key:

```go
expectedFingerprint := mustHex("...sha256 of leaf cert DER...")

tr := &http.Transport{
    TLSClientConfig: &tls.Config{
        VerifyPeerCertificate: func(rawCerts [][]byte, verifiedChains [][]*x509.Certificate) error {
            if len(rawCerts) == 0 { return errors.New("no cert") }
            sum := sha256.Sum256(rawCerts[0])
            if !bytes.Equal(sum[:], expectedFingerprint) {
                return errors.New("cert pin mismatch")
            }
            return nil
        },
    },
}
```

Caveat: pinning makes cert rotation painful. Only pin if the threat model justifies it (mobile app talking to a known backend, financial protocol).

## Mutual TLS (mTLS)

Server requires the client to present a certificate signed by a trusted CA.

```go
caPEM, _ := os.ReadFile("ca.crt")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(caPEM)

cfg := &tls.Config{
    Certificates: []tls.Certificate{serverCert},
    ClientAuth:   tls.RequireAndVerifyClientCert,
    ClientCAs:    caPool,
    MinVersion:   tls.VersionTLS13,
}

srv := &http.Server{Addr: ":443", TLSConfig: cfg, Handler: handler}
srv.ListenAndServeTLS("", "")
```

`ClientAuth` values:

| Value | Meaning |
|-------|---------|
| `NoClientCert` | Don't ask (default) |
| `RequestClientCert` | Ask but accept missing |
| `RequireAnyClientCert` | Must present; don't verify |
| `VerifyClientCertIfGiven` | If presented, must verify |
| `RequireAndVerifyClientCert` | Required and verified |

Production mTLS uses `RequireAndVerifyClientCert`.

### Reading client identity in the handler

```go
func handler(w http.ResponseWriter, r *http.Request) {
    if len(r.TLS.PeerCertificates) == 0 {
        http.Error(w, "no client cert", 401)
        return
    }
    clientCert := r.TLS.PeerCertificates[0]
    cn := clientCert.Subject.CommonName
    // ... authorize by CN, SAN, OID extensions
}
```

For SPIFFE-style identity, parse the SVID from `clientCert.URIs[0]` (e.g., `spiffe://example.com/billing`).

### Client side of mTLS

```go
clientCert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
caPEM, _ := os.ReadFile("ca.crt")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(caPEM)

tr := &http.Transport{
    TLSClientConfig: &tls.Config{
        Certificates: []tls.Certificate{clientCert},
        RootCAs:      caPool,
    },
}
```

## Session Resumption

TLS 1.3 supports session tickets — the client can resume a TLS session without a full handshake, saving 1 RTT.

```go
cfg.SessionTicketsDisabled = false   // default — on
// Custom ticket key (32 bytes) for multi-instance sharing:
cfg.SetSessionTicketKeys([][32]byte{key1, key2})
```

For multiple instances behind a load balancer:

- Share `SessionTicketKey` across instances.
- Rotate keys daily; keep last N keys in the slice for resumption to work across rotations.
- Without sharing: clients resuming on a different instance fall back to full handshake (still works; slower).

## Performance Knobs

```go
cfg.PreferServerCipherSuites = true   // ignored in TLS 1.3 (no negotiation)
cfg.ClientSessionCache = tls.NewLRUClientSessionCache(0)   // client-side resumption
```

Go's TLS implementation is competitive with OpenSSL on modern hardware (AES-NI). For ultra-high-throughput TLS termination, dedicated proxies (Caddy, Envoy, HAProxy) may outperform marginally.

## ALPN-Based Routing (One Port, Multiple Protocols)

```go
ln, _ := tls.Listen("tcp", ":443", cfg)
for {
    conn, _ := ln.Accept()
    go func(c net.Conn) {
        tlsConn := c.(*tls.Conn)
        if err := tlsConn.Handshake(); err != nil { c.Close(); return }
        switch tlsConn.ConnectionState().NegotiatedProtocol {
        case "h2":
            h2Server.ServeConn(tlsConn, ...)
        case "http/1.1":
            // serve via net/http
        case "myproto":
            // custom protocol
        }
    }(conn)
}
```

`tls.Conn.ConnectionState()` exposes everything negotiated: version, cipher, ALPN, SNI, peer cert.

## Configuration Patterns

### Per-environment

```go
func tlsConfig(env string) *tls.Config {
    cfg := &tls.Config{MinVersion: tls.VersionTLS12}
    switch env {
    case "prod":
        cfg.MinVersion = tls.VersionTLS13
    case "dev":
        // okay to be permissive in dev
    }
    return cfg
}
```

### Hot-reload certificates

```go
type CertReloader struct {
    mu   sync.RWMutex
    cert *tls.Certificate
}

func (r *CertReloader) GetCertificate(_ *tls.ClientHelloInfo) (*tls.Certificate, error) {
    r.mu.RLock(); defer r.mu.RUnlock()
    return r.cert, nil
}

func (r *CertReloader) reload(certFile, keyFile string) error {
    c, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil { return err }
    r.mu.Lock(); r.cert = &c; r.mu.Unlock()
    return nil
}

// On SIGHUP, call reloader.reload(...)
cfg.GetCertificate = reloader.GetCertificate
```

Replaces the cert without a server restart. Works for autocert-style cache misses too.

## ACME / Caddy / cert-manager

Three production patterns:

1. **`autocert` in your Go binary** — simplest; great for single-instance.
2. **Caddy as TLS-terminating proxy** — Caddy automates ACME for you; your Go server speaks plain HTTP.
3. **`cert-manager` in Kubernetes** — issues certs as Secrets; your pod mounts them.

For multi-instance services, options 2 and 3 scale better.

## Anti-Patterns & Gotchas

**`InsecureSkipVerify: true` in production.** The certificate is now decorative.

**Storing cert+key in environment variables.** Use file mounts, not env vars (visible in `ps`, K8s API).

**Forgetting OCSP stapling / CT.** Modern CAs require Certificate Transparency. `autocert` handles this; manual cert installs may not.

**Loading certs once at startup and never reloading.** Renewed certs sit on disk unused until restart. Use a reloader.

**Treating `Certificates` and `GetCertificate` as interchangeable.** `GetCertificate` runs per-handshake; expensive lookups will slow handshakes.

**ALPN list with HTTP/2 listed but not actually wired to `http2.Server`.** Negotiation succeeds, but no handler is ready. Go's `net/http` handles this automatically; custom code may not.

**Pinning to a leaf certificate.** Renewal breaks the pin. Pin to a CA or to a public key (SPKI hash).

**Disabling certificate hostname verification "to simplify dev."** Make a self-signed CA, add it to `RootCAs`, and trust your dev cert properly.

**Calling `tls.Config{}` to construct then mutating after first use.** A connection in progress may have a stale view. Clone with `cfg.Clone()` then mutate.

**Using TLS 1.0/1.1 because some client said they need it.** Get the spec; usually they need TLS 1.2.

**Mixing `Certificates` and `GetCertificate`.** When both set, `GetCertificate` wins.

**Ignoring `tls.RecordHeaderError` in logs.** It's "the client sent garbage" — usually a non-TLS scan, not an emergency. Log at DEBUG, not ERROR.

**No session resumption key sharing across instances.** Multi-instance terminations resumeing through one LB end up doing full handshakes every time.

**Trusting `r.TLS.PeerCertificates[0].Subject.CommonName` for authorisation.** CN is deprecated; use SAN (Subject Alternative Names) or URI SANs (SPIFFE IDs).

**Not preloading the CT log inclusion checks.** Some browsers refuse certs not in a CT log. Use a reputable CA + autocert.

## Performance Notes

- **TLS 1.3 handshake**: 1 RTT (or 0 RTT with session tickets and PSK).
- **TLS 1.2 handshake**: 2 RTT.
- **AES-NI accelerated AES-GCM**: 3-5 GB/s per core.
- **ChaCha20-Poly1305 (no AES-NI)**: 1-2 GB/s per core.
- **Cert loading at startup**: <10ms for 100 certs.
- **`GetCertificate` callback per handshake**: keep it <1ms or you bottleneck.
- **Memory per TLS conn**: ~5-50 KB depending on buffers.

## How Big Companies Use It

- **Cloudflare** runs TLS termination at scale on Go; their `cfssl` is a popular cert-management library.
- **Caddy** is entirely Go + autocert + custom; powers many production sites.
- **Let's Encrypt** itself (Boulder) is Go.
- **Tailscale** uses TLS via WireGuard underneath, but exposes TLS endpoints for some traffic.
- **Lyft / Envoy** (C++; Go control plane) uses Go for `cert-manager`-style automation.
- **HashiCorp Vault** PKI engine is Go; many services use it as their internal CA.
- **Kubernetes** uses `crypto/tls` everywhere (`kube-apiserver`, etcd, kubelet); mTLS is standard between components.
- **Step CA** (smallstep) — open-source CA in Go, ACME-compatible.

## Source Code References

- `crypto/tls`: https://github.com/golang/go/tree/master/src/crypto/tls.
- `crypto/x509`: https://github.com/golang/go/tree/master/src/crypto/x509.
- `golang.org/x/crypto/acme/autocert`: https://github.com/golang/crypto/tree/master/acme/autocert.
- `lego` (ACME library): https://github.com/go-acme/lego.
- Caddy: https://github.com/caddyserver/caddy.
- cert-manager: https://github.com/cert-manager/cert-manager.
- `cfssl`: https://github.com/cloudflare/cfssl.
- Step CA: https://github.com/smallstep/certificates.
- Boulder (Let's Encrypt): https://github.com/letsencrypt/boulder.

## Further Reading

- "Bulletproof SSL and TLS" (Ivan Ristić) — the reference.
- "TLS Mastery" (Michael Lucas) — practical, modern.
- Filippo Valsorda's blog (former Go security team lead): https://blog.filippo.io/.
- Cloudflare blog on TLS: https://blog.cloudflare.com/tag/tls/.
- "So you want to expose Go on the Internet" (Cloudflare).
- "TLS 1.3 explained" (Filippo Valsorda).
- Mozilla SSL Configuration Generator: https://ssl-config.mozilla.org/.
- "Post-Quantum TLS in 2024" — Cloudflare blog.

## Exercises / Self-Check

1. Set up a Go HTTPS server with `autocert`. Confirm it issues a Let's Encrypt cert for a real domain you control.
2. Configure mTLS: server requires + verifies client cert. Use `curl --cert client.crt --key client.key` to test.
3. Pin a client's certificate via `VerifyPeerCertificate`. Renew the server cert; observe the pin failure. Adjust strategy.
4. Use SNI-based routing to serve two domains with two certs on one IP:port.
5. Implement a hot-reload `GetCertificate`. Rotate certs without restart; confirm by inspecting the served cert (`openssl s_client`).
6. Disable TLS 1.2; verify `MinVersion: tls.VersionTLS13` rejects older clients.
7. Enable HTTP/2: confirm `NextProtos: ["h2","http/1.1"]` negotiates `h2` with `curl --http2`.
8. Share session ticket keys across two Go instances behind a single LB. Verify resumption works across instances.
