# Tailscale — A Whole VPN in Go

## TL;DR

**Tailscale** is a mesh VPN built entirely on **WireGuard** plus a Go control plane and client. Everything but the WireGuard kernel module is Go: the `tailscaled` daemon, the `tailscale` CLI, the **DERP** relay servers, the control plane, the `tsnet` library that embeds Tailscale into any Go binary, the macOS/iOS/Android non-UI layers. The team's stack is famous in the Go community for **using Go idiomatically, hard** — every Go-feature blog post (`go.uber.org/automaxprocs` predecessor, `goleak` adoption, `gVisor netstack` as user-space TCP/IP) shows up in Tailscale's code. The single biggest gotcha: **Tailscale's "VPN" is mostly a NAT-traversal and key-distribution problem, not a packet-pushing problem**; their `tailscaled` does little in the packet-data path and most in the *control* path (peer discovery, key rotation, NAT punching, ACL evaluation).

## Mental Model

```
   Tailscale node (your laptop, server, phone):
   
   ┌──────────────────────────────────────────────────────────┐
   │  Your apps (any language)                                 │
   │    use 100.x.y.z addresses (Tailscale's CGNAT range)      │
   └─────────────┬────────────────────────────────────────────┘
                 ▼ (kernel route 100.0.0.0/8)
   ┌──────────────────────────────────────────────────────────┐
   │  tailscaled (Go)                                          │
   │    - WireGuard userspace OR kernel WireGuard              │
   │    - magicsock: NAT-traversal UDP plumbing                │
   │    - control: speaks to login.tailscale.com               │
   │    - DNS proxy (MagicDNS)                                  │
   │    - ACLs, exit-node routing                                │
   └─────────────┬────────────────────────────────────────────┘
                 ▼
       UDP packets (encrypted) over the public internet
                 │
                 ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Other Tailscale nodes (direct P2P when possible),         │
   │  DERP relays as fallback when both sides are double-NAT'd │
   └──────────────────────────────────────────────────────────┘

   Control plane:
   ┌──────────────────────────────────────────────────────────┐
   │  login.tailscale.com (Go, hosted by Tailscale Inc)        │
   │    - issues node keys, distributes peer maps              │
   │    - WebSocket "noise" sessions to each node              │
   └──────────────────────────────────────────────────────────┘
```

The control plane sees no packet data — only the encrypted metadata needed to coordinate keys. Customer payloads stay end-to-end encrypted via WireGuard.

## Syntax & Basic Usage

`tsnet` embeds Tailscale into any Go binary:

```go
package main

import (
	"fmt"
	"io"
	"net/http"

	"tailscale.com/tsnet"
)

func main() {
	s := &tsnet.Server{
		Hostname: "my-app",          // appears in the Tailscale admin UI
		AuthKey:  "tskey-auth-...",  // ephemeral auth from the admin panel
	}
	defer s.Close()

	ln, err := s.Listen("tcp", ":80")
	if err != nil { panic(err) }

	http.Serve(ln, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		who, _ := s.WhoIs(r.Context(), r.RemoteAddr)
		fmt.Fprintf(w, "hello %s (via Tailscale)\n", who.UserProfile.LoginName)
		_ = io.Discard
	}))
}
```

Your service is reachable at `http://my-app/` from any other device on your Tailscale network. No firewall holes; no certificates.

## Deep Dive

### Founders and language choice

Tailscale was founded in 2019 by **Brad Fitzpatrick** (creator of LiveJournal, memcached, OAuth; former Go team member at Google) and **David Crawshaw** (former Go team member, contributed to mobile + cgo). Plus David Carney and Avery Pennarun (CEO).

Two of the four founders were former Go-team engineers. The language choice was deliberate, but never religious. Brad's quote (paraphrased from various talks): "We picked Go because it has the best mix of network programming primitives and a deployment story across every OS we care about."

### WireGuard — kernel and userspace

WireGuard is the data-plane crypto. Two implementations:

1. **Linux kernel WireGuard**: maximum performance; requires kernel module / Linux ≥5.6. Used when available.
2. **Userspace `wireguard-go`**: Go port of WireGuard by Jason Donenfeld. Bundled in `tailscaled` for non-Linux platforms.

`wireguard-go` does encryption in user space via Go's `golang.org/x/crypto/chacha20poly1305`. Less performant than kernel WireGuard (~2–5 Gbps vs ~10+ Gbps) but works on macOS, Windows, BSDs.

### magicsock — NAT-traversal

[magicsock](https://github.com/tailscale/tailscale/tree/main/wgengine/magicsock) is Tailscale's NAT-traversal layer. It:

1. Multiplexes WireGuard packets over a single UDP socket per node.
2. Probes peers via UDP from multiple paths (direct, STUN, DERP).
3. Switches to the lowest-latency working path.
4. Falls back to DERP (relay) when no direct path works.

Standard NAT-traversal: STUN-derived endpoints + UDP hole-punching. magicsock optimizes for fast convergence — typical "first packet" round trip ~50 ms.

### DERP — Designated Encrypted Relay for Packets

[DERP servers](https://github.com/tailscale/tailscale/tree/main/derp) are Tailscale's last-resort relays. When two nodes can't establish a direct path (e.g., both behind symmetric NAT), they forward WireGuard packets through DERP. Encrypted end-to-end; DERP sees only encrypted blobs.

DERP is HTTP/2 + WebSocket-like. Each DERP server handles ~10k concurrent peers. Implemented entirely in Go.

### Control plane

The control plane (login.tailscale.com) is Go. It:

- Authenticates users (OAuth, OIDC, SSO).
- Stores public node keys.
- Distributes peer maps to each node ("here are your peers and their public keys").
- Pushes ACL changes.

Coordinates over **Noise IK** sessions (long-lived, mutually-authenticated, encrypted). The wire protocol is fully documented: https://tailscale.com/blog/tcp-vs-udp.

### MagicDNS

Tailscale's DNS server. Each node has a name like `laptop.your-tailnet.ts.net`; resolves to its 100.x.y.z address. Implemented in Go, integrated into `tailscaled`.

### ACLs

[Tailscale ACLs](https://tailscale.com/kb/1018/acls) are JSON / HuJSON policies evaluated on each node. The control plane distributes the policy; each node enforces locally. Evaluation in Go.

### Why Go-team people built it in Go

The founders have given many talks on this. Key reasons:

- **Cross-platform**: ships binaries for Linux/macOS/Windows/iOS/Android/FreeBSD.
- **Networking stdlib**: net + crypto + tls cover most needs.
- **Deployment ease**: single static binary; embed into containers and CLIs trivially.
- **Concurrency**: NAT-traversal is inherently concurrent — Goroutines are excellent.
- **Personal expertise**: Brad and David lived inside Go for years.

### Tailscale's open-source posture

Tailscale's Go client and server are open source (BSD-3). The control plane is open-source too (`headscale` is a third-party implementation). The hosted service is closed-source but the binary you run on your laptop is fully inspectable.

### Use of `gVisor` netstack

For platforms without TUN-device support (iOS, some Android), Tailscale uses **gVisor's `netstack`** as a user-space TCP/IP stack. The Tailscale client implements a virtual interface in user space; netstack provides the TCP/IP. Documented post: https://tailscale.com/blog/userspace-networking.

This is the cleanest production example of someone else using a piece of gVisor (see `20-big-tech/02-google-gvisor.md`).

### tsnet — embed Tailscale into Go binaries

`tsnet.Server` lets your Go program *be* a Tailscale node:

```go
import "tailscale.com/tsnet"

s := &tsnet.Server{Hostname: "my-service"}
ln, _ := s.Listen("tcp", ":80")
http.Serve(ln, mux)
```

Your service joins the Tailnet automatically; no separate `tailscaled`. Used for:
- Hosted services that want a Tailscale-only admin endpoint.
- CLIs that need to reach the Tailnet.
- CI runners that join a tailnet to access internal hosts.

### Go practices Tailscale popularized

- **`goleak` in tests**: every test verifies no goroutine leaks.
- **Strict context propagation**: ctx-first parameter, ctx through every call.
- **`defer` for cleanup**: heavy use; pre-1.14 it was costly, now negligible.
- **`go vet` strict + `staticcheck` clean**: every PR.
- **Internal `mem.RO` type**: an alternative to `[]byte` and `string` that's read-only and avoids copies in some paths.
- **`syncs.AtomicValue[T]`**: generic atomic value wrapper (since 1.18).
- **`util/multierr.Append`**: in-house multi-error type (predates `errors.Join`).
- **Heavy use of `golang.org/x/sync/errgroup`**.
- **Profiling endpoints on every binary**: `tsweb` exposes `/debug/vars` and `/debug/pprof/`.

### `tsweb` debug surface

Tailscale's [`tsweb`](https://github.com/tailscale/tailscale/tree/main/tsweb) package wraps standard `net/http/pprof` with structured access to:

- Pprof endpoints.
- Build info (`runtime/debug.ReadBuildInfo`).
- Liveness checks.
- A small landing page.

Every Tailscale binary exposes `tsweb`. Excellent for debugging in production.

### Cross-platform packaging

Tailscale ships binaries for:
- Linux (amd64, arm64, armv7, mips, mips64, ppc64le, riscv64).
- macOS (universal, signed).
- Windows (signed, MSI installer).
- iOS (Network Extension framework).
- Android.
- FreeBSD, OpenBSD.
- Synology / QNAP NAS.
- Various router OSes (OPNsense, OpenWRT).

All from one Go codebase + per-platform glue (TUN device handling, system DNS integration, GUI).

### The TUN device

On Unix, Tailscale binds to a `/dev/net/tun` virtual interface. Packets the kernel writes to this interface land in `tailscaled`'s read loop; packets `tailscaled` writes back go out to the network stack. TUN drivers are platform-specific (different syscalls on Linux vs macOS vs FreeBSD).

### Synchronization patterns

`tailscaled` is a long-running daemon with many concurrent flows. Common patterns:

#### 1. Event-driven state machine

```go
type State int
const (
    StateStopped State = iota
    StateStarting
    StateAuthRequired
    StateRunning
)

func (b *Backend) sm(events <-chan Event) {
    s := StateStopped
    for e := range events {
        s = transition(s, e)
    }
}
```

A single goroutine owns the state; events arrive on a channel.

#### 2. Atomic snapshots for read-mostly data

```go
type prefs struct{...}
var current atomic.Pointer[prefs]

func Read() *prefs { return current.Load() }
func Update(p *prefs) { current.Store(p) }
```

Readers never lock; writers do copy-on-write.

#### 3. Heavy use of `context.Context` everywhere

Every long-running operation accepts a `context.Context`. Shutdown is one cancellation away.

### Performance

`wireguard-go` (Go's port) is ~2–5 Gbps single-flow on a modern CPU. Kernel WireGuard is 10+ Gbps. Tailscale uses kernel WireGuard when available.

`tailscaled`'s CPU footprint at idle: <0.1% of a core. Under traffic: scales with throughput; userspace WireGuard dominates.

### Recent additions

- **SSH** server built in: Tailscale-authenticated SSH without keys.
- **Funnel**: expose `tsnet` services to the public internet on `*.ts.net`.
- **Serve**: HTTPS reverse-proxy from Tailscale node to local service.
- **Drive** (file sharing).
- **Taildrop** (file transfer).

All in Go.

## Standard Library Hooks

Tailscale stresses:

- `net`, `net/http`, `crypto/tls`: ubiquitous.
- `golang.org/x/crypto/chacha20poly1305`: WireGuard encryption.
- `golang.org/x/crypto/curve25519`: key exchange.
- `golang.org/x/sys/unix`, `golang.org/x/sys/windows`: per-platform syscalls.
- `gvisor.dev/gvisor/pkg/tcpip/...`: userspace TCP/IP for iOS/Android.
- `nhooyr.io/websocket` / `coder/websocket`: control-plane sessions.
- `runtime/debug.ReadBuildInfo`, `runtime/pprof`: ops surface.

## Real-World Patterns

### 1. Embedded Tailscale service (`tsnet`)

Shown above. Single binary becomes a Tailnet node.

### 2. Restricted admin endpoint

```go
s := &tsnet.Server{Hostname: "admin"}
ln, _ := s.Listen("tcp", ":80")

http.Serve(ln, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    who, err := s.WhoIs(r.Context(), r.RemoteAddr)
    if err != nil || who.UserProfile.LoginName != "alice@example.com" {
        http.Error(w, "forbidden", 403)
        return
    }
    // ... admin only
}))
```

Reachable only over Tailscale; identity is the Tailscale login.

### 3. Mesh service-to-service

Run multiple services with `tsnet`; each gets a `100.x.y.z` address. They reach each other by Tailscale name. No firewalls; no LB; no certs.

### 4. CI runner joining a Tailnet

```bash
$ tailscale up --authkey="$TS_AUTHKEY" --hostname="ci-runner-$JOB_ID"
$ ./run-tests.sh
$ tailscale logout
```

The CI runner is now a temporary Tailnet member; can reach internal staging hosts; vanishes when done.

### 5. Stream packets through the userspace stack

```go
import "gvisor.dev/gvisor/pkg/tcpip/transport/tcp"

// Pseudo: create a netstack stack
s := stack.New(stack.Options{
    NetworkProtocols:   []stack.NetworkProtocolFactory{ipv4.NewProtocol},
    TransportProtocols: []stack.TransportProtocolFactory{tcp.NewProtocol},
})
// Wire the stack to Tailscale's WireGuard interface
```

For iOS / Android where the kernel network stack isn't available.

## Anti-Patterns & Gotchas

**Treating Tailscale as a fast LAN.** It's a mesh VPN; routes packets over the public internet (mostly direct, sometimes DERP). Throughput is real but bounded by encryption + bandwidth.

**Using `tsnet` and exposing it on `0.0.0.0`.** The whole point is reachable-only-over-Tailscale. Bind to the `tsnet`-provided listener.

**Trusting `tsnet.Server.LocalClient()` without context.** Operations can block on control-plane.

**Embedding `tsnet` and forgetting `defer s.Close()`.** Leaks goroutines and a temporary auth key.

**Pre-creating long-lived auth keys.** Use ephemeral auth keys (`--ephemeral`) so the node disappears when idle.

**Assuming `wireguard-go` matches kernel performance.** It doesn't. Use kernel WireGuard where available.

**Hardcoding `100.x.y.z` addresses.** Tailscale changes them as nodes come and go. Use MagicDNS names.

**Skipping ACLs.** By default every node can reach every node. Configure ACLs to constrain.

**Trusting NAT-traversal in air-gapped networks.** Some networks block STUN. Plan for DERP fallback.

**Running `tailscaled` as non-root.** Some platforms (Linux, macOS) need it; others have userspace mode (`--tun=userspace-networking`).

## Performance Notes

(Tailscale-published.)

- Kernel WireGuard throughput: ~10+ Gbps single flow.
- `wireguard-go` (userspace): 2–5 Gbps single flow.
- DERP relay throughput per server: 1–5 Gbps aggregate.
- NAT-traversal convergence: typically <500 ms.
- `tailscaled` idle CPU: <0.1% per core.
- `tsnet` service overhead: ~30 MiB memory.

## How Big Companies Use It

- **Most YC startups**: Tailscale-by-default for internal infra.
- **GitHub**: documented Tailscale use for GitHub Actions runners.
- **HashiCorp**: internal use; Vault SSH integration with Tailscale SSH.
- **Mozilla**: documented uses for internal dev environments.
- **Atlassian**: connecting cloud → datacenter networks.
- **Stripe, Brex, Notion**: documented as customers.
- **Government clients** (Tailscale-published case studies).

Tailscale's open-source `tsnet` makes it easy to add Tailscale to any Go service, so unlike most "big company" stories the *technology* is widely reused even if the company is small.

## Source Code References

All open source (BSD-3).

- Main repo: [`tailscale/tailscale`](https://github.com/tailscale/tailscale).
- WireGuard userspace: [`tailscale/wireguard-go`](https://github.com/tailscale/wireguard-go).
- magicsock: [`tailscale/wgengine/magicsock/`](https://github.com/tailscale/tailscale/tree/main/wgengine/magicsock).
- DERP: [`tailscale/derp/`](https://github.com/tailscale/tailscale/tree/main/derp).
- tsnet: [`tailscale/tsnet/`](https://github.com/tailscale/tailscale/tree/main/tsnet).
- tsweb: [`tailscale/tsweb/`](https://github.com/tailscale/tailscale/tree/main/tsweb).
- Control plane Go client: [`tailscale/control/...`](https://github.com/tailscale/tailscale/tree/main/control).
- ACL evaluator: [`tailscale/wgengine/filter/`](https://github.com/tailscale/tailscale/tree/main/wgengine/filter).
- IPN (interface to UI): [`tailscale/ipn/`](https://github.com/tailscale/tailscale/tree/main/ipn).
- Netstack integration: [`tailscale/wgengine/netstack/`](https://github.com/tailscale/tailscale/tree/main/wgengine/netstack).
- headscale (third-party control plane): [`juanfont/headscale`](https://github.com/juanfont/headscale).

## Further Reading

- Tailscale blog: https://tailscale.com/blog.
- "How Tailscale Works": https://tailscale.com/blog/how-tailscale-works.
- "WireGuard from scratch in Go" — wireguard-go origin.
- "userspace networking" (using gVisor netstack): https://tailscale.com/blog/userspace-networking.
- "TCP vs UDP for control plane": https://tailscale.com/blog/tcp-vs-udp.
- "How NAT traversal works": https://tailscale.com/blog/how-nat-traversal-works.
- David Crawshaw's blog (Tailscale CTO): https://crawshaw.io.
- Brad Fitzpatrick talks at GopherCon (2022, 2023).
- "tsnet: Tailscale-as-a-library": https://tailscale.com/kb/1244/tsnet.
- Headscale documentation: https://headscale.net.

## Exercises / Self-Check

1. Embed `tsnet` in a small Go HTTP server. Reach it from another Tailscale node by hostname (MagicDNS).
2. Why do Tailscale's DERP servers exist if WireGuard provides end-to-end encryption? What problem do they solve?
3. Use `gVisor`'s netstack directly in a Go program (no Tailscale). What's its `Listen` API, and how does it differ from `net.Listen`?
4. Compare kernel WireGuard and `wireguard-go` userspace throughput on the same hardware. Where's the time spent in the userspace version?
5. Tailscale uses `atomic.Pointer[T]` (1.19+) for read-mostly state. Why is that preferable to `*T` + `sync.RWMutex`? Construct a benchmark.
