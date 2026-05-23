# HashiCorp — Terraform, Vault, Consul, Nomad, Packer, Boundary, Waypoint, Vagrant

## TL;DR

**HashiCorp** is the most prolific Go user in the operations / infrastructure space. Their stack — **Terraform** (infrastructure as code), **Vault** (secrets), **Consul** (service mesh + KV), **Nomad** (orchestrator), **Packer** (image building), **Boundary** (zero-trust access), **Waypoint** (app deploys) — is essentially all Go. The company is responsible for shipping more Go binaries into enterprises than anyone except Google. They wrote and open-sourced many widely-reused Go libraries: **`go-multierror`**, **`hclog`**, **`raft`**, **`memberlist`**, **`serf`**, **`go-version`**, **`yamux`**. The single biggest gotcha: **HashiCorp's products are mature, opinionated, and large**, which means upgrades, plugins, and modifications can be heavy lifts; their Go style is also distinctive (HCL config files, plugin-via-gRPC architecture) and differs from "modern" Go idioms in places.

## Mental Model

```
   HashiCorp open-source stack (all Go):
   
   ┌────────────────────────────────────────────────────────────┐
   │  Terraform — declarative IaC                                │
   │   - core CLI in Go                                          │
   │   - providers as out-of-process gRPC plugins                │
   │   - state file in S3/Consul/Postgres                        │
   ├────────────────────────────────────────────────────────────┤
   │  Vault — secrets store                                      │
   │   - HA via Raft                                              │
   │   - dozens of auth/secret backends as plugins                │
   ├────────────────────────────────────────────────────────────┤
   │  Consul — service mesh + KV + health checks                  │
   │   - Raft for consensus                                       │
   │   - gossip via Serf for membership                           │
   ├────────────────────────────────────────────────────────────┤
   │  Nomad — workload orchestrator                                │
   │   - Raft for consensus                                       │
   │   - drivers: Docker, raw_exec, java, qemu                    │
   ├────────────────────────────────────────────────────────────┤
   │  Packer, Boundary, Waypoint, Vagrant, Sentinel — others       │
   └────────────────────────────────────────────────────────────┘
   
   Common substrate (HashiCorp libraries):
       - hashicorp/raft         — Raft consensus
       - hashicorp/memberlist   — gossip membership
       - hashicorp/serf         — service discovery
       - hashicorp/go-plugin    — gRPC-based plugin framework
       - hashicorp/hcl          — config language
       - hashicorp/go-multierror, hclog, go-version, yamux, etc.
```

The libraries are reused across the products — and adopted by hundreds of unrelated projects (Kubernetes, MinIO, CockroachDB, Caddy, Traefik all use one or more).

## Syntax & Basic Usage

Use HashiCorp's Raft library:

```go
package main

import (
	"net"
	"os"
	"time"

	"github.com/hashicorp/raft"
	raftboltdb "github.com/hashicorp/raft-boltdb/v2"
)

type fsm struct{}

func (f *fsm) Apply(l *raft.Log) any                { return nil }
func (f *fsm) Snapshot() (raft.FSMSnapshot, error)  { return nil, nil }
func (f *fsm) Restore(snap io.ReadCloser) error     { return nil }

func main() {
	cfg := raft.DefaultConfig()
	cfg.LocalID = "node-1"

	store, _ := raftboltdb.NewBoltStore("/tmp/raft.db")
	snap, _ := raft.NewFileSnapshotStore("/tmp/snapshots", 1, os.Stderr)

	addr, _ := net.ResolveTCPAddr("tcp", "127.0.0.1:7000")
	tr, _ := raft.NewTCPTransport("127.0.0.1:7000", addr, 3, 10*time.Second, os.Stderr)

	r, _ := raft.NewRaft(cfg, &fsm{}, store, store, snap, tr)
	r.BootstrapCluster(raft.Configuration{
		Servers: []raft.Server{{ID: "node-1", Address: "127.0.0.1:7000"}},
	})

	select {}
}

// (Sketch — see hashicorp/raft examples for full code.)
```

The Raft library is widely used in production by Consul, Nomad, Vault, etcd-like systems, and many homegrown databases.

## Deep Dive

### Why Go (HashiCorp's reasoning)

Mitchell Hashimoto's blog posts from 2014–2016 describe the choice: Vagrant was Ruby (slow, dependency-heavy); Packer was the first Go rewrite. The reasons cited:

- **Single static binary per OS/arch** — distribution is trivial.
- **Cross-compilation** — same source ships to every platform.
- **Strong stdlib for HTTP, crypto, networking** — needed for everything HashiCorp builds.
- **Reasonable concurrency** — agents, clients, servers all benefit.
- **Modular plugin model** — gRPC-based plugins (their `go-plugin` framework) allows independent versioning.

By 2015, HashiCorp standardized on Go for all new products. Terraform, Consul, Vault, and Nomad all followed.

### Terraform

[Terraform](https://github.com/hashicorp/terraform) is HashiCorp's flagship. ~500k LOC of Go (core + helpers). Architecture:

- **CLI** parses HCL config files.
- **Core** builds a DAG of resources, computes a plan, applies in dependency order.
- **Providers** (AWS, Azure, GCP, Kubernetes, etc.) are *separate binaries* loaded via gRPC plugins.

Each provider is its own Go module shipping its own binary; Terraform launches it as a subprocess; communication via gRPC over Unix sockets.

Key Go patterns:
- `context.Context` propagated through every plan/apply step.
- `terraform-plugin-sdk` / `terraform-plugin-framework` (modern) for provider authors.
- Heavy use of `hcl` for config parsing.
- State management with concurrency-safe locks (DynamoDB, S3, Consul, Postgres).

#### Provider plugins via `go-plugin`

`hashicorp/go-plugin` lets you run plugins as separate processes. Lifecycle:
1. Main binary spawns plugin binary.
2. Plugin opens a Unix socket; advertises a gRPC service.
3. Main binary dials the socket; calls gRPC methods.
4. On unload, main sends SIGTERM.

Benefits: plugins crash independently. Providers can be written in any language with gRPC support. Drawbacks: subprocess overhead per provider.

### Vault

[Vault](https://github.com/hashicorp/vault) is HashiCorp's secrets management system. ~200k LOC of Go. Architecture:

- **Storage backends**: Raft (default), Consul, etcd, dynamodb, ...
- **Auth methods**: token, AWS IAM, Kubernetes, LDAP, GCP, ...
- **Secrets engines**: KV, transit (encryption-as-a-service), PKI, AWS secrets, database creds, ...
- **Sealing**: Vault is "sealed" on startup; needs unseal keys to operate.

Modern Vault clusters use **integrated Raft storage** (HashiCorp Raft library) — eliminates the need for external Consul.

Vault's Go style:
- Each backend is a plugin (via `go-plugin`).
- Heavy use of `context.Context` and `vault/sdk` framework.
- Audit log to multiple sinks (file, syslog, socket).
- `crypto/x509` heavily used for PKI engine.

### Consul

[Consul](https://github.com/hashicorp/consul) provides service discovery, KV, health checks, mesh networking. ~300k LOC. Architecture:

- **Servers**: 3 or 5 nodes; Raft consensus.
- **Clients**: agents on every node; gossip via Serf.
- **DNS interface**: nodes resolve `service.consul` → IPs.
- **HTTP API**: full CRUD.

Consul popularized the "service discovery agent" pattern.

Notable libraries spun out of Consul:
- `hashicorp/raft`.
- `hashicorp/serf` (gossip).
- `hashicorp/memberlist` (Serf's foundation).
- `hashicorp/go-version`.
- `hashicorp/hcl`.

### Nomad

[Nomad](https://github.com/hashicorp/nomad) is HashiCorp's workload orchestrator — alternative to Kubernetes. ~300k LOC. Architecture:

- **Servers**: Raft consensus on job state.
- **Clients**: agents on every node; run task drivers.
- **Drivers**: Docker, raw_exec, qemu, java, exec, all Go.

Nomad's design point: simpler than Kubernetes, single binary, supports non-container workloads natively. Used by Cloudflare for edge deploys, Roblox for game servers, others.

### Packer

[Packer](https://github.com/hashicorp/packer) builds machine images (AMI, GCE images, Docker, Vagrant). ~100k LOC. Architecture: plugins for each cloud/platform, JSON/HCL templates.

### Boundary

[Boundary](https://github.com/hashicorp/boundary) is identity-based access to remote resources. ~150k LOC. Postgres-backed. Designed as a Tailscale alternative — but PaaS-style, not VPN-style.

### Waypoint

[Waypoint](https://github.com/hashicorp/waypoint) is an application deployment workflow tool. Wraps Docker, Kubernetes, Nomad, AWS Lambda behind a unified `waypoint up` command.

### Sentinel

Closed-source policy-as-code engine. Used in Terraform Cloud/Enterprise. Mentioned for completeness.

### The library ecosystem (reused everywhere)

#### `hashicorp/raft`

The most cited Go Raft implementation. Used by:
- Consul.
- Nomad.
- Vault.
- InfluxDB IFQL.
- Many homegrown systems.

#### `hashicorp/serf` and `hashicorp/memberlist`

SWIM-protocol gossip implementations. Used by:
- Consul.
- Nomad.
- Linkerd (older versions).
- Many homegrown clustering systems.

#### `hashicorp/go-plugin`

gRPC-based plugin framework. Used by:
- Terraform.
- Vault.
- Various security tools.

#### `hashicorp/hcl`

HashiCorp Configuration Language. JSON-compatible but human-friendly. Used by:
- Terraform.
- Nomad.
- Packer.
- Several non-HashiCorp tools.

#### `hashicorp/go-multierror`

Multi-error aggregation. Pre-`errors.Join`. Used widely; many projects haven't migrated.

#### `hashicorp/hclog`

Structured leveled logger. Used by all HashiCorp products. zap-like but predates zap.

#### `hashicorp/yamux`

Stream multiplexing over a single TCP connection. Used by Vault, Boundary, others.

#### `hashicorp/go-version`

Semver-compatible version comparison. Used in CI tools, package managers.

#### `hashicorp/golang-lru`

LRU cache (and 2Q variants). Used in many projects unrelated to HashiCorp.

### HashiCorp's Go style

Distinctive elements (as of mid-2020s codebases):

- **`hclog` instead of stdlib `log`**. Older than zap; structured logging.
- **`go-multierror` instead of `errors.Join`**. Older code still uses it.
- **`context.Background()` more often than freshly-derived contexts**. Pragmatic in CLI-shaped programs.
- **`init()` for plugin registration**. Heavier than modern style.
- **Imports grouped by source** (stdlib, third-party, internal). 3-group style.
- **`flag` package, not `cobra`** in older binaries; `cobra` in newer.

Newer HashiCorp code (2022+) increasingly uses modern Go idioms (`errors.Join`, `slog`, generics). Migration is gradual.

### Open-source posture

HashiCorp historically open-sourced all core products (MPL-2.0). In **August 2023** they relicensed Terraform, Consul, Vault, Nomad, Packer, Boundary, Waypoint, Vagrant to **BSL (Business Source License)** — non-OSI-approved. Sparked the **OpenTofu** fork of Terraform (https://opentofu.org). HashiCorp libraries (raft, serf, hcl, go-plugin) remained MPL.

In **2024**, IBM acquired HashiCorp.

### Lessons for Go projects

- **Plugin via gRPC** is mature; consider it for extensibility.
- **Raft + gossip** are well-trodden paths; the library exists.
- **Single-binary distribution** is the HashiCorp success formula.
- **HCL** is a credible alternative to YAML for config.
- **Cross-compilation discipline**: every HashiCorp tool ships for every platform.

## Standard Library Hooks

- `net/http`, `crypto/tls`: all server endpoints.
- `crypto/x509`, `crypto/rsa`, `crypto/ecdsa`: PKI in Vault, Consul, Boundary.
- `flag` / `spf13/cobra`: CLIs.
- `context`: pervasive.
- `os/signal`: graceful shutdown.
- `runtime/pprof`: debug surfaces.
- `database/sql` + drivers: state in Vault/Boundary.
- `golang.org/x/sync/errgroup`: concurrent work.

## Real-World Patterns

### 1. Service discovery with Consul

```go
import "github.com/hashicorp/consul/api"

cli, _ := api.NewClient(api.DefaultConfig())
_ = cli.Agent().ServiceRegister(&api.AgentServiceRegistration{
    Name: "my-service",
    Port: 8080,
    Check: &api.AgentServiceCheck{
        HTTP:     "http://localhost:8080/health",
        Interval: "10s",
    },
})

services, _, _ := cli.Health().Service("my-service", "", true, nil)
for _, s := range services {
    fmt.Println(s.Service.Address, s.Service.Port)
}
```

### 2. Distributed lock via Consul KV

```go
sess, _ := cli.Session().Create(&api.SessionEntry{
    Name: "my-lock",
    TTL:  "15s",
}, nil)

locked, _, _ := cli.KV().Acquire(&api.KVPair{
    Key:     "lock/my-resource",
    Session: sess,
}, nil)

if locked {
    defer cli.KV().Release(&api.KVPair{Key: "lock/my-resource", Session: sess}, nil)
    // critical section
}
```

### 3. Vault dynamic database credentials

```go
import "github.com/hashicorp/vault/api"

vc, _ := api.NewClient(api.DefaultConfig())
vc.SetToken("hvs.token...")

secret, _ := vc.Logical().Read("database/creds/my-role")
user := secret.Data["username"].(string)
pass := secret.Data["password"].(string)
// connect to DB with these short-lived creds
```

Vault issues credentials valid for a few minutes; rotates automatically.

### 4. Terraform provider skeleton

```go
package main

import (
	"github.com/hashicorp/terraform-plugin-framework/provider"
	"github.com/hashicorp/terraform-plugin-framework/providerserver"
)

type myProvider struct{}

func (p *myProvider) Schema(/* ... */)   { /* ... */ }
func (p *myProvider) Configure(/* ... */) { /* ... */ }
// ... Resources, DataSources, etc.

func main() {
	providerserver.Serve(nil, func() provider.Provider { return &myProvider{} },
		providerserver.ServeOpts{Address: "example.com/me/myprovider"})
}
```

`terraform-plugin-framework` is the modern provider SDK; an older `terraform-plugin-sdk` v2 still has wide use.

### 5. Embedding Raft for HA

The Raft snippet from "Syntax & Basic Usage" above. Used by CockroachDB, etcd-alternatives, custom databases.

## Anti-Patterns & Gotchas

**Using `hashicorp/go-multierror` in new code.** Use `errors.Join` (1.20+).

**Pinning to old `hashicorp/hclog`.** It still works but slog (1.21+) covers most needs.

**Treating Terraform state as ephemeral.** It's authoritative; corrupted state is catastrophic. Always use remote state with locking.

**Running Vault without HA from day one.** Single-node Vault loses data on host failure.

**Mixing Consul Connect (mesh) and a separate sidecar mesh.** Pick one.

**Storing huge values in Consul KV.** It's a coordinator, not a database. >512 KiB values cause Raft latency spikes.

**Plugin protocol versioning mistakes.** Terraform's plugin protocol changes; old providers may not load.

**Running multiple HashiCorp products' embedded Raft on the same disk without I/O isolation.** Consul + Nomad + Vault all writing to the same disk fight for fsync.

**Ignoring `hcl` parsing errors.** They're often informative — point to the wrong column.

**Treating Boundary as a Tailscale replacement.** Different model; Boundary is PaaS-style identity-based, not always-on mesh.

## Performance Notes

(Rough estimates, vary by deployment.)

- Consul gossip convergence (3-node cluster): seconds.
- Consul Raft commit latency: ~10–50 ms (intra-DC).
- Vault read latency (cached): <1 ms.
- Vault write latency: ~10–100 ms.
- Nomad allocation start: seconds for Docker driver, sub-second for raw_exec.
- Terraform plan time: scales with resource count; ~10–60 s typical.
- Packer image build: minutes (depends on what you build).

## How Big Companies Use It

- **Cloudflare**: Nomad for edge deploys.
- **GitHub**: Consul (older), Vault.
- **Roblox**: Nomad for game servers.
- **CircleCI**: Nomad for build runners.
- **Cruise (Autonomous)**: Consul + Terraform.
- **Stripe**: Vault for secrets management.
- **Walmart, Capital One**: Vault, Terraform.
- **Adobe**: Consul, Terraform.
- **JPMorgan, Goldman Sachs**: Vault.
- **Most YC startups**: Terraform.

Terraform alone is in tens of thousands of production deployments.

## Source Code References

All HashiCorp open source on GitHub:

- Terraform: [`hashicorp/terraform`](https://github.com/hashicorp/terraform).
- Vault: [`hashicorp/vault`](https://github.com/hashicorp/vault).
- Consul: [`hashicorp/consul`](https://github.com/hashicorp/consul).
- Nomad: [`hashicorp/nomad`](https://github.com/hashicorp/nomad).
- Packer: [`hashicorp/packer`](https://github.com/hashicorp/packer).
- Boundary: [`hashicorp/boundary`](https://github.com/hashicorp/boundary).
- Waypoint: [`hashicorp/waypoint`](https://github.com/hashicorp/waypoint).
- Vagrant: [`hashicorp/vagrant`](https://github.com/hashicorp/vagrant) (Ruby, not Go).
- raft: [`hashicorp/raft`](https://github.com/hashicorp/raft).
- serf: [`hashicorp/serf`](https://github.com/hashicorp/serf).
- memberlist: [`hashicorp/memberlist`](https://github.com/hashicorp/memberlist).
- go-plugin: [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin).
- hcl: [`hashicorp/hcl`](https://github.com/hashicorp/hcl).
- hclog: [`hashicorp/hclog`](https://github.com/hashicorp/hclog).
- yamux: [`hashicorp/yamux`](https://github.com/hashicorp/yamux).
- go-multierror: [`hashicorp/go-multierror`](https://github.com/hashicorp/go-multierror).
- golang-lru: [`hashicorp/golang-lru`](https://github.com/hashicorp/golang-lru).
- OpenTofu (Terraform fork): [`opentofu/opentofu`](https://github.com/opentofu/opentofu).

(Licenses vary; mostly MPL-2.0 for libraries, BSL since 2023 for products.)

## Further Reading

- "The Tao of HashiCorp": https://www.hashicorp.com/tao-of-hashicorp.
- "Vault architecture": https://developer.hashicorp.com/vault/docs/internals/architecture.
- "Consul architecture": https://developer.hashicorp.com/consul/docs/architecture.
- "Nomad architecture": https://developer.hashicorp.com/nomad/docs/concepts/architecture.
- Mitchell Hashimoto's blog: https://mitchellh.com/.
- HashiCorp blog: https://www.hashicorp.com/blog.
- "Raft consensus" — Diego Ongaro paper: https://raft.github.io/.
- "Lessons learned with Raft" — HashiCorp blog.
- "Terraform Provider Development Framework": https://developer.hashicorp.com/terraform/plugin/framework.
- "Why HashiCorp uses BSL": https://www.hashicorp.com/license-faq.

## Exercises / Self-Check

1. Use `hashicorp/raft` to build a minimal in-memory KV replicated across 3 nodes. Demonstrate that a write commits only after majority ack.
2. Register a service with Consul (in dev mode); discover it from another Go program via the `consul/api` client.
3. Write a Terraform provider skeleton using `terraform-plugin-framework`. Implement one resource that "creates" a local file.
4. Compare `hclog` to `slog` for a typical log statement: features, performance, ergonomics. Which would you choose for a new project?
5. Why does HashiCorp use out-of-process plugins (gRPC subprocess) rather than dynamic loading (`plugin` package)? List three reasons.
