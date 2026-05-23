# Cloudflare — Edge Compute in Go, BoringCrypto, RRDNS

## TL;DR

Cloudflare runs a **global edge network** spanning 300+ cities; significant portions of the edge — DNS, TLS termination, WAF preprocessing, Workers control plane — are written in Go. Notable Cloudflare Go projects: **RRDNS** (authoritative DNS, now Go-based), **cfssl** (TLS PKI tooling), **tableflip** (zero-downtime upgrades), **redoctober** (split-knowledge crypto), **goflow** (NetFlow/sFlow ingestion), **BoringCrypto** patches (FedRAMP-compliant TLS via Google's BoringSSL-in-Go). Their Workers runtime is C++/V8, but everything *around* it — orchestration, control plane, analytics — is Go. The single biggest gotcha at Cloudflare scale: **garbage collection at the edge is unforgiving**. A 5 ms GC pause on a 1M-RPS server produces a noticeable spike across millions of users; the team has documented multiple GC-driven incidents and contributed to several Go GC improvements.

## Mental Model

```
   Cloudflare edge node (one of ~3000 cities):
   
   ┌──────────────────────────────────────────────────────────┐
   │  NGINX / Pingora (reverse proxy) — historically C/Lua,    │
   │   now mostly Rust (Pingora)                                │
   ├──────────────────────────────────────────────────────────┤
   │  Go services (edge "sidecars"):                            │
   │   - RRDNS: authoritative + recursive DNS                   │
   │   - Workers control plane                                  │
   │   - Logpush: log streaming                                 │
   │   - Argo: smart routing                                    │
   │   - Argo Tunnel (cloudflared)                              │
   │   - Cache invalidation                                     │
   ├──────────────────────────────────────────────────────────┤
   │  Workers Runtime (V8-based JS/Wasm, C++)                   │
   └──────────────────────────────────────────────────────────┘
                            ▲
                            │  control & analytics flow
                            ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Core data plane (Go services):                            │
   │   - Quicksilver (KV propagation to edge)                   │
   │   - Tunnel control plane                                   │
   │   - Magic Transit / Magic WAN routing                       │
   │   - Analytics & dashboards                                  │
   └──────────────────────────────────────────────────────────┘
```

Go isn't the *packet-pushing* tier (that's Rust/C++ for max throughput); Go is the **glue and control plane** plus several data-handling services where developer velocity matters more than per-microsecond latency.

## Syntax & Basic Usage

A Cloudflare-style zero-downtime reload server using `tableflip`:

```go
package main

import (
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"

	"github.com/cloudflare/tableflip"
)

func main() {
	upg, err := tableflip.New(tableflip.Options{})
	if err != nil { log.Fatal(err) }
	defer upg.Stop()

	go func() {
		sig := make(chan os.Signal, 1)
		signal.Notify(sig, syscall.SIGHUP)
		for range sig {
			_ = upg.Upgrade()
		}
	}()

	ln, err := upg.Listen("tcp", ":8080")
	if err != nil { log.Fatal(err) }

	srv := &http.Server{}
	go srv.Serve(ln)

	if err := upg.Ready(); err != nil { log.Fatal(err) }

	<-upg.Exit()
	_ = srv.Shutdown(nil)
}
```

`tableflip` lets you SIGHUP a process; a new child binds the same socket via SCM_RIGHTS, finishes init, signals "ready", and the parent exits. No dropped connections.

## Deep Dive

### Why Go at Cloudflare

Cloudflare's blog has documented Go's role over a decade. Key drivers:

- **Productivity for control-plane services**: API servers, config pipelines, log pushers.
- **Predictable runtime characteristics**: better than Node's tail; better than Java's startup.
- **Single binary deploys** simplify packaging for ~3000 edge sites.
- **Strong stdlib for networking**: DNS, TLS, HTTP/2/3.
- **Existing community libs** (DNS, crypto, BGP).

Cloudflare engineers (Filippo Valsorda, Marek Vavruša, Vlad Krasnov, Brendan Coll) have published extensively on Go's role.

### RRDNS

[RRDNS](https://github.com/cloudflare/rrdns) (Resolver Reasonable DNS) is Cloudflare's authoritative DNS server. Originally NSD-based (C), now Go.

Architecture:
- Parses incoming UDP/TCP queries.
- Looks up in an in-memory zone store (constantly updated by Quicksilver).
- Implements DNSSEC, ECS, geo-aware routing.
- Handles ~50M QPS globally.

Why Go: developer velocity for a critical service that must support new RR types, DNSSEC variants, attack mitigations frequently. The team writes "DNS is messy; we change the code every week".

### cloudflared (Argo Tunnel)

[cloudflared](https://github.com/cloudflare/cloudflared) is the open-source agent that connects an origin to Cloudflare's edge. Runs at customer infrastructure; opens outbound TLS tunnels to Cloudflare, eliminating the need for inbound firewall rules.

~30k LOC of Go. Notable patterns:
- `quic-go` for QUIC-based tunnels.
- gRPC streaming for control plane.
- `tableflip` for zero-downtime upgrades.
- Heavy use of `context` for graceful shutdown.

### tableflip

[tableflip](https://github.com/cloudflare/tableflip) implements graceful zero-downtime restart for Go servers. The pattern:

1. Parent process binds listening socket.
2. SIGHUP triggers upgrade.
3. Parent forks a child, passing the socket FD via SCM_RIGHTS.
4. Child runs init, calls `upg.Ready()`.
5. Parent stops accepting and drains existing connections, then exits.

Used by Cloudflare for every long-running Go service that must restart without dropping connections. Documented blog: https://blog.cloudflare.com/graceful-upgrades-in-go/.

### cfssl

[CFSSL](https://github.com/cloudflare/cfssl) is Cloudflare's PKI toolkit — CA management, signing, OCSP, CT log support. Used for internal certificate issuance at Cloudflare and adopted by Kubernetes (cert-manager, k3s).

Notable: implements multi-root CAs, automated cert rotation, ACME server, transparency log helpers. Pre-dates `crypto/x509`'s modern features; Cloudflare contributed many upstream changes.

### goflow

[goflow](https://github.com/cloudflare/goflow) (and goflow2) ingest NetFlow/sFlow/IPFIX from network devices, convert to Kafka events. Used in Cloudflare's network telemetry pipeline at multi-million-flow/sec rates.

### DNSCryptoUtility patterns

`golang.org/x/crypto`'s DNS-related modules came partly from Cloudflare contributions: ed25519, ECDSA-with-deterministic-k, post-quantum experiments. Filippo Valsorda (Go security lead, ex-Cloudflare) brought much of this work into the standard library.

### BoringCrypto

For FIPS-140 compliance, Cloudflare ships builds of Go that link Google's **BoringCrypto** — a FIPS-validated C cryptography library — replacing parts of `crypto/...`. This is the same project Google maintains for internal compliance.

The patches live in `dev.boringcrypto` branches in the Go repo. As of Go 1.24+ the **native FIPS mode** ([proposal #66218](https://github.com/golang/go/issues/66218)) is being integrated, replacing the BoringCrypto patch tree.

### Pingora migration

In 2022 Cloudflare migrated NGINX (Lua-heavy) to **Pingora**, a Rust framework, for the proxy hot path. The reason was per-connection memory, not Go: NGINX's worker-per-CPU model didn't share state cleanly; Lua's GC was unpredictable; Pingora's Tokio-based scheduling was a better fit.

Cloudflare's blog about the move explicitly notes "Go was not the right fit for the hottest data plane; we still use Go for control plane". See https://blog.cloudflare.com/pingora-open-source/.

### GOMEMLIMIT in production

Cloudflare deployed `GOMEMLIMIT` widely after Go 1.19. The blog post "Two months in production: Go 1.19's GOMEMLIMIT" documents:
- Setting `GOMEMLIMIT` to ~90% of cgroup memory limit.
- Watching for GC CPU spikes when approaching the limit.
- Net effect: fewer OOM kills, more predictable RSS.

### Workers control plane

Cloudflare Workers (the V8/Wasm runtime) is C++. But the **control plane** that deploys, configures, and observes Workers is Go: APIs, tier-2 routing, KV propagation. Hundreds of services.

### Common Cloudflare Go patterns

#### 1. Zero-downtime restart

```go
upg, _ := tableflip.New(tableflip.Options{})
ln, _ := upg.Listen("tcp", ":443")
go server.Serve(ln)
upg.Ready()
<-upg.Exit()
```

#### 2. Per-connection contexts with deadlines

Every connection gets a context with a request deadline; child operations (DNS lookups, backend calls) inherit. Tail-latency wins.

#### 3. Bounded fan-out

Edge requests trigger downstream work (logs, analytics). Cloudflare uses fixed-size goroutine pools to bound concurrency — same pattern Uber uses.

#### 4. Aggressive `sync.Pool` for per-request state

Hot paths reuse buffers, decoder state, parser state via `sync.Pool`. Hot enough to make `pprof` show "pool churn" in flame graphs.

#### 5. Custom JSON

Cloudflare wrote and uses fork-of-`json-iterator-go` patches and (more recently) tracks `encoding/json/v2`. The general direction: fast streaming JSON with strict schema validation.

### GC-driven incidents

Cloudflare's "post-mortems" on GC pauses are required reading:
- "Live-blogging a Go GC pause incident": catastrophic stop-the-world on a pre-1.8 deployment.
- "Go don't collect my garbage": detailed allocator analysis.

These contributed to the priority placed on STW reduction in Go 1.8 (hybrid write barrier), 1.14 (async preempt), and 1.25 (Green Tea GC).

### Cryptography expertise

Filippo Valsorda (now Go security lead, formerly Cloudflare) led many TLS improvements: TLS 1.3 prioritization, X25519 fixes, post-quantum experimentation. Cloudflare's Go fork at times shipped with patches months ahead of upstream.

## Standard Library Hooks

- `net`, `net/http`, `net/http/httputil`: edge proxies.
- `crypto/tls`, `crypto/x509`, `crypto/rand`: TLS termination.
- `golang.org/x/crypto/...`: extended cryptography.
- `context`: pervasive cancellation.
- `runtime/debug.SetMemoryLimit`: GOMEMLIMIT rollout.
- `runtime/pprof` + `net/http/pprof`: production profiling on every service.
- `quic-go` (third-party): HTTP/3, MASQUE.
- `golang.org/x/sync`: errgroup, singleflight.
- `encoding/json` + custom forks.

## Real-World Patterns

### 1. cloudflared minimal tunnel

```go
// Conceptually:
import (
	"context"

	"github.com/cloudflare/cloudflared/connection"
)

func main() {
	ctx := context.Background()
	cfg := connection.Config{
		Origin: "http://localhost:8080",
		Cred:   "your-tunnel-token",
	}
	conn, _ := connection.New(cfg)
	_ = conn.Serve(ctx)
}
```

A few lines: outbound QUIC tunnel from your host into Cloudflare; incoming Cloudflare requests flow back through.

### 2. Authoritative DNS server with `miekg/dns`

```go
import "github.com/miekg/dns"

func handler(w dns.ResponseWriter, r *dns.Msg) {
	m := new(dns.Msg)
	m.SetReply(r)
	for _, q := range r.Question {
		if q.Qtype == dns.TypeA {
			rr := &dns.A{
				Hdr: dns.RR_Header{Name: q.Name, Rrtype: dns.TypeA, Class: dns.ClassINET, Ttl: 60},
				A:   net.ParseIP("1.2.3.4"),
			}
			m.Answer = append(m.Answer, rr)
		}
	}
	_ = w.WriteMsg(m)
}

func main() {
	dns.HandleFunc(".", handler)
	_ = dns.ListenAndServe(":53", "udp", nil)
}
```

`miekg/dns` is the canonical DNS library for Go, used by RRDNS, CoreDNS, and many others.

### 3. NetFlow ingestion (goflow-style)

```go
import "github.com/netsampler/goflow2"

dec := goflow2.NewDecoder()
go func() {
	for msg := range incomingPackets {
		flows, _ := dec.Decode(msg)
		for _, f := range flows {
			kafka.Produce(f)
		}
	}
}()
```

### 4. TLS terminator with rotation

```go
// Pseudo: load certs from disk, watch for changes, swap atomically.
import "sync/atomic"

var certs atomic.Pointer[tls.Certificate]

func reload() {
	c, _ := tls.LoadX509KeyPair("cert.pem", "key.pem")
	certs.Store(&c)
}

cfg := &tls.Config{
	GetCertificate: func(_ *tls.ClientHelloInfo) (*tls.Certificate, error) {
		return certs.Load(), nil
	},
}
```

### 5. PGO build with collected profile

```bash
# Collect profile from prod (Cloudflare runs continuous profiling)
$ curl http://prod:6060/debug/pprof/profile?seconds=60 > prod.pprof
$ cp prod.pprof ./cmd/edge-service/default.pgo
$ go build ./cmd/edge-service
```

Cloudflare adopted PGO early in Go 1.21; documented 4–7% throughput wins.

## Anti-Patterns & Gotchas

**Goroutine-per-connection at >1M RPS.** Goroutine churn alone can be a measurable percentage of CPU. Use bounded pools.

**Heavy reflection in hot paths.** `encoding/json` reflection on every request adds up. Either pre-compile codecs (`easyjson`, `sonic`) or migrate to `encoding/json/v2`.

**Synchronous logging.** zap is fast; `log.Println` is slow + blocks. Cloudflare moved to structured async logging years ago.

**Forgetting `Close` on `*http.Response.Body`.** Connection pool leaks. At edge scale, hundreds of MB of stuck connections in days.

**`fmt.Sprintf` in TLS handshake fast path.** Real source of allocation profiling at Cloudflare. Use `strconv.Append*`.

**Trusting Linux's default `/dev/random`.** Use `crypto/rand`; on Linux it reads `getrandom(2)` which is correct.

**SIGTERM → instant exit.** Cloudflare patches every service with `signal.NotifyContext` + drain loops.

**Building without `-trimpath`.** Source paths leak into production binaries; profile symbols differ across machines. Standardize.

**Per-request DNS resolution.** The stdlib resolver is slow and caches poorly. Use `dnscache` or your own.

**Treating Go's HTTP/2 server as bulletproof.** It is robust but has had real CVEs (rapid reset, etc.). Stay on the latest patch release.

## Performance Notes

Approximate Cloudflare numbers from public posts:

- RRDNS query rate: ~50M QPS globally.
- Average DNS response latency: <1 ms.
- TLS handshake rate per edge node: 100k+ /sec.
- Workers cold start: ~5 ms (V8 isolate).
- cloudflared tunnel latency: ~ms over QUIC.
- Go GC pause typical: <500 µs (post-1.14).
- Service startup time (with tableflip): <1 s warmup before traffic.

## How Big Companies Use It

- **Cloudflare's own customers** (Discord, Shopify, Cisco, IBM, etc.) consume the platform; many run Go themselves.
- **NS1**, **DNSimple**, **Hurricane Electric**: other DNS providers using Go + miekg/dns.
- **cert-manager** in Kubernetes is built on CFSSL concepts: https://github.com/cert-manager/cert-manager.
- **Let's Encrypt** runs Boulder (Go ACME server): https://github.com/letsencrypt/boulder.
- **Akamai** uses Go for some edge management services (not the data plane).
- **Fastly** is mostly Rust now, but historically had Go in the control plane.

## Source Code References

- cloudflared: [`cloudflare/cloudflared`](https://github.com/cloudflare/cloudflared).
- tableflip: [`cloudflare/tableflip`](https://github.com/cloudflare/tableflip).
- CFSSL: [`cloudflare/cfssl`](https://github.com/cloudflare/cfssl).
- RRDNS-like (publicly: redoctober + ksp-cli): [`cloudflare/redoctober`](https://github.com/cloudflare/redoctober).
- circl (post-quantum crypto): [`cloudflare/circl`](https://github.com/cloudflare/circl).
- goflow / goflow2: [`netsampler/goflow2`](https://github.com/netsampler/goflow2).
- miekg/dns (used by RRDNS): [`miekg/dns`](https://github.com/miekg/dns).
- quic-go (used in cloudflared): [`quic-go/quic-go`](https://github.com/quic-go/quic-go).
- Pingora (Rust, for comparison): [`cloudflare/pingora`](https://github.com/cloudflare/pingora).

(BSD-3 / MIT / Apache-2.0; per project.)

## Further Reading

- "Graceful upgrades in Go" (tableflip): https://blog.cloudflare.com/graceful-upgrades-in-go/.
- "Two months in production: Go 1.19's GOMEMLIMIT": https://blog.cloudflare.com/two-go-memory-related-features/.
- "Go don't collect my garbage": https://blog.cloudflare.com/go-don-t-collect-my-garbage/.
- "Performance tuning a Go server": https://blog.cloudflare.com/.
- "RRDNS: Cloudflare's authoritative DNS server" — talks by Marek Vavruša.
- "Why Pingora?" (Rust migration of NGINX): https://blog.cloudflare.com/pingora-open-source/.
- Filippo Valsorda's blog (Go crypto / security): https://filippo.io.
- "How Cloudflare built BoringCrypto for Go": https://blog.cloudflare.com.
- Cloudflare Research blog (algorithms, post-quantum): https://blog.cloudflare.com/research/.
- "Securing a Go service in production" — Cloudflare engineering: https://blog.cloudflare.com.

## Exercises / Self-Check

1. Implement a minimal tableflip-style listener that survives SIGHUP. Verify the parent process exits *after* the child binds.
2. Read Cloudflare's "Go don't collect my garbage" post and identify three allocator-pressure causes they fixed. What's the equivalent in modern Go (1.22+)?
3. Set up a DNS server using `miekg/dns` that answers A queries from an in-memory map. Benchmark single-threaded QPS. Where does the bottleneck show up?
4. Build a TLS reverse proxy in Go that hot-reloads certificates atomically (no dropped connections). Confirm with a load test during reload.
5. Why does Cloudflare use `runtime/debug.SetMemoryLimit` rather than only `GOGC`? What are the trade-offs of each?
