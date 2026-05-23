# Caddy and Traefik — Modern Web Servers in Go

## TL;DR

**Caddy** and **Traefik** are the two flagship Go web servers / reverse proxies. Caddy (Matt Holt, 2015) is a general-purpose HTTPS server with automatic Let's Encrypt; Traefik (Containous → Traefik Labs, 2016) is a dynamic reverse proxy designed for Docker/Kubernetes service discovery. Both pioneered **"HTTPS by default"** and **automatic certificate management** in mainstream deployment. Both are single static binaries with rich configuration. The single biggest gotcha: **they're *not* general substitutes for NGINX/HAProxy in every dimension** — Caddy and Traefik prioritize *developer experience* and *automatic ops* over raw throughput. NGINX's worker-per-CPU C model still wins at the highest QPS; Caddy/Traefik win on operability, configuration ergonomics, and TLS automation.

## Mental Model

```
   Caddy:
   ┌──────────────────────────────────────────────────────────┐
   │  Caddyfile (DSL) or JSON config                           │
   │                                                            │
   │   example.com {                                            │
   │     reverse_proxy localhost:8080                           │
   │   }                                                         │
   ├──────────────────────────────────────────────────────────┤
   │  HTTP/HTTPS/HTTP3 server (Go net/http + quic-go)          │
   │  TLS auto-management via ACME (Let's Encrypt, ZeroSSL)     │
   │  Storage: filesystem default; can be Consul, S3, etc.      │
   │  Modules (Go plugins, statically compiled)                 │
   └──────────────────────────────────────────────────────────┘
   
   Traefik:
   ┌──────────────────────────────────────────────────────────┐
   │  Static config (file/env/CLI) + dynamic (from "providers")│
   │   - Providers: Docker, Kubernetes Ingress/CRD,            │
   │     Consul, etcd, file, KV stores                         │
   │   - Auto-discover services by container labels            │
   ├──────────────────────────────────────────────────────────┤
   │  HTTP/HTTPS/TCP/UDP routers + middlewares                  │
   │  TLS via ACME                                              │
   │  Tracing, metrics, dashboards built-in                     │
   └──────────────────────────────────────────────────────────┘
```

Caddy is "type a config, get HTTPS". Traefik is "label your containers, get HTTPS routing".

## Syntax & Basic Usage

### Caddy

`Caddyfile`:

```
example.com {
    reverse_proxy localhost:8080
}

api.example.com {
    reverse_proxy localhost:8081
    rate_limit {
        rate 100r/s
    }
}
```

Run: `caddy run`. Caddy fetches a Let's Encrypt cert on first request (provided DNS points at the box), renews automatically. No additional config needed.

### Traefik

`docker-compose.yml` with labels:

```yaml
services:
  traefik:
    image: traefik:v3
    command:
      - --providers.docker
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.email=me@example.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
    ports: ["80:80", "443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - le:/letsencrypt

  myapp:
    image: myapp:latest
    labels:
      - traefik.http.routers.myapp.rule=Host(`example.com`)
      - traefik.http.routers.myapp.tls.certresolver=le
```

Traefik discovers the `myapp` container via labels, routes HTTPS traffic to it, manages the cert.

## Deep Dive

### Caddy architecture

Caddy is a **modular** Go server. Core (~50k LOC) is small; almost everything (TLS, routing, middlewares, transports) is a module statically linked at build time. Modules implement registered interfaces; the JSON config selects which to use.

Key components:
- `caddyhttp`: HTTP server.
- `caddytls`: TLS automation (ACME).
- `reverse_proxy`: load balance + health checks.
- `file_server`: static file serving.
- `templates`: in-process HTML templates.
- `replace_response`, `rewrite`, `header`, `encode`: middlewares.

Custom modules: Go packages that register on import. `xcaddy` builds a Caddy binary with selected modules.

### Caddy's TLS automation

Implemented via [`certmagic`](https://github.com/caddyserver/certmagic) — Caddy's TLS automation library, separable from Caddy. CertMagic:
- Implements ACME client (RFC 8555).
- Manages certificate lifecycle: issuance, renewal, OCSP stapling.
- Stores certs/keys atomically; replicates across instances when configured.

Used by:
- Caddy.
- Several third-party Go services that want HTTPS without bothering with cert ops.

### Caddy as a library

```go
package main

import "github.com/caddyserver/caddy/v2/caddyconfig/caddyfile"
import "github.com/caddyserver/caddy/v2"

func main() {
    cfg := []byte(`{
        "apps": {
            "http": {
                "servers": {
                    "srv0": {
                        "listen": [":443"],
                        "routes": [{
                            "match": [{"host": ["example.com"]}],
                            "handle": [{"handler": "reverse_proxy", "upstreams": [{"dial": "localhost:8080"}]}]
                        }]
                    }
                }
            }
        }
    }`)
    _ = caddy.Load(cfg, true)
    select {}
}
```

Embed Caddy in your own Go program. Used by some teams to ship a Go service that *includes* its own reverse proxy.

### Traefik architecture

Traefik is **provider-driven**. The static config tells Traefik *where to look*; dynamic config (from Docker labels, Kubernetes CRDs, Consul KV, etc.) tells Traefik *what to route*.

Internal model:
- **EntryPoints**: ports Traefik listens on (`:80`, `:443`, `:tcp/3306`).
- **Routers**: match requests (Host, Path, Headers); apply middlewares; route to a Service.
- **Services**: load-balance across endpoints; supports sticky, mirroring, weighted.
- **Middlewares**: auth, rate limit, retry, headers, redirect.
- **TLS Stores**: cert per host, ACME or static.

### Traefik providers

Providers watch external state and feed dynamic config in:

- **Docker**: parses container labels.
- **Kubernetes Ingress** (v1).
- **Kubernetes CRD**: `IngressRoute`, `Middleware`, etc.
- **Consul Catalog**, **Consul KV**.
- **etcd**, **Redis**, **ZooKeeper**.
- **File**: watches a YAML/TOML.
- **HTTP**: pulls from an HTTP endpoint periodically.

Architecturally elegant: writing a new provider is implementing a Go interface that emits events.

### Performance comparison

NGINX (C, worker-per-CPU): 1M+ RPS on a 16-core box.
Caddy (Go): ~200–500k RPS on similar hardware.
Traefik (Go): ~100–300k RPS.

Why the gap? Go's `net/http` is good but not C-with-event-loop level. The encoding cost of headers, the GC pressure of per-request allocations, the lack of zero-copy syscalls all add up.

For most workloads, this gap doesn't matter — your origin servers are slower than any of these proxies. For mega-scale edge (Cloudflare, Fastly), C/Rust still wins; that's why Cloudflare moved NGINX to Pingora (see `20-big-tech/04-cloudflare.md`).

### Caddy plugin ecosystem

`xcaddy build` lets you compile a Caddy binary with selected plugins:

```bash
$ xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/mholt/caddy-l4
```

Resulting binary has DNS-01 ACME support via Cloudflare API and L4 (TCP) proxying.

### Traefik enterprise

Traefik EE (paid) adds:
- Distributed authentication.
- HA across multi-region.
- Multi-team RBAC.
- Hosted control plane.

The OSS version is feature-rich on its own; most users stop there.

### Caddy 1 vs Caddy 2

Caddy 1 (2015–2019) was a different architecture. Caddy 2 (2020+) rewrote everything around the JSON config + module system. The Caddyfile still exists but is converted to JSON internally.

### Real-world deployments

#### Caddy

- **DuckDuckGo**: documented Caddy use.
- **Hugo blog**: static-site hosting.
- **Substack-like services**: TLS-by-default for user domains.
- **Several universities**: campus HTTPS.

#### Traefik

- **Kubernetes Ingress**: probably the second-most-deployed ingress after NGINX-Ingress.
- **Docker Swarm**: classic Traefik environment.
- **Hetzner Cloud, DigitalOcean, Scaleway**: as recommended ingress.
- **Many CI/CD pipelines**: dynamic routing for ephemeral environments.

### Lessons for Go developers

#### 1. Modular runtime via reflection

Caddy's module system uses `caddy.RegisterModule(Module)`. Modules implement an interface; the JSON config picks them by name. Plugin without subprocess; statically compiled.

#### 2. Provider pattern

Traefik's "watch external state, push config in" pattern generalizes well — used by service meshes, ingress controllers, GitOps tools.

#### 3. ACME in Go

CertMagic and `lego` (https://github.com/go-acme/lego) are mature ACME clients. Use them; don't roll your own.

#### 4. HTTP/3 and QUIC

Caddy was an early adopter of HTTP/3 via `quic-go`. Production-ready in current versions.

#### 5. Active health checks

Both implement active health checks (periodic probing) on upstreams. Useful pattern for any Go service that load-balances.

## Standard Library Hooks

- `net/http`: foundational.
- `crypto/tls`: TLS sessions.
- `crypto/x509`: cert handling.
- `context`: per-request lifecycle.
- `golang.org/x/sync/errgroup`: parallel work.
- `gopkg.in/yaml.v3`: config.
- `quic-go`: HTTP/3.

## Real-World Patterns

### 1. Caddyfile for HTTPS reverse proxy

```
api.example.com {
    @auth header Authorization Bearer*
    handle @auth {
        reverse_proxy backend:8080
    }
    handle {
        respond "auth required" 401
    }
}
```

### 2. Traefik with Kubernetes Ingress

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
spec:
  entryPoints: [websecure]
  routes:
  - match: Host(`example.com`)
    kind: Rule
    services:
    - name: myapp-svc
      port: 80
  tls:
    certResolver: le
```

### 3. Embed Caddy in a Go binary

```go
// Generate JSON config at startup and load:
import "github.com/caddyserver/caddy/v2"

func startProxy() error {
    return caddy.Load(myJSONConfig, true)
}
```

Useful for self-contained binaries that include their own TLS terminator.

### 4. CertMagic standalone

```go
import "github.com/caddyserver/certmagic"

certmagic.DefaultACME.Agreed = true
certmagic.DefaultACME.Email = "me@example.com"

if err := certmagic.HTTPS([]string{"example.com"}, mux); err != nil {
    log.Fatal(err)
}
```

`certmagic.HTTPS` wraps your `http.Handler` with TLS auto-management. One line of code; full HTTPS.

### 5. Traefik dynamic provider

```yaml
# providers.file (dynamic.yaml)
http:
  routers:
    myrouter:
      rule: Host(`example.com`)
      service: mysvc
  services:
    mysvc:
      loadBalancer:
        servers:
        - url: http://localhost:8080
```

Edit the file → Traefik reloads dynamically. No restart.

## Anti-Patterns & Gotchas

**Comparing Caddy/Traefik to NGINX on raw QPS.** Different design goals. Optimize where it matters.

**Running ACME on a low-DNS-TTL public domain without HTTP/TLS-ALPN challenge support.** Renewals fail. Use DNS-01 with a DNS provider plugin.

**Editing Caddy JSON config by hand.** Use Caddyfile (easier to read) or programmatic generation.

**Trusting Caddy's defaults for high-traffic prod.** Tune buffer sizes, transport idle timeouts, max connections.

**Using Traefik's Docker labels with too-long label values.** Some Docker label parsers truncate. Use file/CRD provider for complex routes.

**Mixing Traefik EntryPoint and explicit IngressRoute TLS.** Conflicts; pick one source of truth.

**Self-signing certs in dev with Caddy.** It actually has an `internal` CA option — use it instead of manual self-signing.

**Forgetting Traefik's `[providers.kubernetesIngress]` vs `[providers.kubernetesCRD]`** are different sources. Pick one.

**Trying to bypass Caddy's JSON config layer.** Don't; it's the canonical config format. Caddyfile is sugar.

**Running multiple Caddy instances pointing at the same ACME storage without coordination.** Use `certmagic`'s S3 backend or similar for HA.

## Performance Notes

- Caddy RPS (simple proxy, 16 cores): ~200–500k.
- Traefik RPS: ~100–300k.
- TLS handshake throughput: 10k+/sec per core.
- Reload latency: ms to seconds depending on cert count.
- HTTP/3 throughput: 70–90% of HTTP/2 in Go (encryption-CPU bound).
- Memory per active connection: ~30 KiB.

## How Big Companies Use It

- **Stripe**: documented use of Caddy for some internal services.
- **DigitalOcean App Platform**: uses Caddy.
- **fly.io**: heavily uses Caddy patterns.
- **Hetzner Cloud Marketplace**: ships Traefik.
- **Rancher**: ships Traefik as default ingress.
- **k3s** (lightweight K8s by Rancher): bundles Traefik.
- **Adyen**: Traefik in production.
- **The Go team**: golang.org / pkg.go.dev are served by Caddy.

## Source Code References

- Caddy: [`caddyserver/caddy`](https://github.com/caddyserver/caddy).
- CertMagic: [`caddyserver/certmagic`](https://github.com/caddyserver/certmagic).
- xcaddy (build tool): [`caddyserver/xcaddy`](https://github.com/caddyserver/xcaddy).
- Traefik: [`traefik/traefik`](https://github.com/traefik/traefik).
- lego (ACME library): [`go-acme/lego`](https://github.com/go-acme/lego).
- quic-go: [`quic-go/quic-go`](https://github.com/quic-go/quic-go).

(Apache-2.0; MIT)

## Further Reading

- Caddy docs: https://caddyserver.com/docs/.
- Matt Holt blog: https://matt.life.
- "How Caddy works": https://caddyserver.com/docs/architecture.
- Traefik docs: https://doc.traefik.io/traefik/.
- "Traefik vs NGINX vs HAProxy" comparisons: various.
- CertMagic ACME deep-dive: https://caddyserver.com/docs/automatic-https.
- ACME (RFC 8555): https://datatracker.ietf.org/doc/html/rfc8555.
- HTTP/3 in Go: https://caddyserver.com/docs/v2-upgrade.

## Exercises / Self-Check

1. Stand up Caddy with one upstream and watch the ACME flow with `--debug`. What steps does it run for a fresh cert?
2. Configure Traefik to discover Docker containers. Add labels to two containers and watch the routes appear in Traefik's dashboard.
3. Use CertMagic standalone in a Go program: wrap an `http.ServeMux` with HTTPS, automatic cert from Let's Encrypt staging.
4. Benchmark Caddy vs Traefik vs NGINX on the same hardware. Where do they diverge? Where do they tie?
5. Write a Caddy module that implements a custom middleware (e.g., log request bodies above some size). Build with `xcaddy`.
