# CNCF Go Projects — Prometheus, etcd, containerd, Helm, CoreDNS, Linkerd

## TL;DR

The **Cloud Native Computing Foundation (CNCF)** hosts the bulk of modern cloud-native open source — and **the vast majority of CNCF graduated/incubating projects are written in Go**. Notables: **Prometheus** (metrics), **etcd** (distributed KV, the heart of Kubernetes), **containerd** (container runtime), **Helm** (Kubernetes package manager), **CoreDNS** (DNS server), **Linkerd** (service mesh — Rust data plane + Go control plane), **Argo**, **Flux**, **Crossplane**, **Cilium** (data plane is C+eBPF, control plane Go). The single biggest gotcha: **CNCF "Go" projects often have non-Go components for performance-critical data planes** — Cilium's eBPF, Envoy's C++, Linkerd's Rust proxy. Go dominates **control planes** and **agents**; the highest-throughput data path is frequently another language.

## Mental Model

```
   CNCF project landscape (where Go appears):
   
   ┌──────────────────────────────────────────────────────────┐
   │  Graduated, mostly Go:                                    │
   │   - Kubernetes        (mostly Go; see Part 20 entry)      │
   │   - Prometheus        (Go everywhere)                      │
   │   - etcd              (Go)                                 │
   │   - containerd        (Go; runc in Go too)                 │
   │   - Helm              (Go)                                 │
   │   - CoreDNS           (Go)                                 │
   │   - Argo (CD, Workflows, Rollouts, Events) — Go            │
   │   - Flux              (Go)                                 │
   │   - Crossplane        (Go)                                 │
   │   - OpenTelemetry collector (Go)                            │
   │   - Cortex / Thanos / Mimir (Go, Prometheus-compatible)     │
   ├──────────────────────────────────────────────────────────┤
   │  Mixed (Go + other):                                       │
   │   - Linkerd            (control plane Go; proxy Rust)      │
   │   - Cilium             (control plane Go; data plane eBPF) │
   │   - Envoy              (mostly C++; some Go contrib for     │
   │                          xDS server and Go filters)         │
   │   - Istio              (control plane Go; data plane Envoy) │
   └──────────────────────────────────────────────────────────┘
```

Go's stronghold is the **control plane**: APIs, reconcilers, schedulers, agents.

## Syntax & Basic Usage

A trivial Prometheus exposition:

```go
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	requests = promauto.NewCounter(prometheus.CounterOpts{
		Name: "http_requests_total",
		Help: "Total number of HTTP requests.",
	})
)

func main() {
	http.Handle("/metrics", promhttp.Handler())
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		requests.Inc()
		_, _ = w.Write([]byte("ok"))
	})
	http.ListenAndServe(":8080", nil)
}
```

Prometheus scrapes `/metrics`, parses the exposition format, stores in TSDB.

## Deep Dive

### Prometheus

[Prometheus](https://github.com/prometheus/prometheus) is the de-facto metrics system for Kubernetes-era operations. ~150k LOC Go. Architecture:

- **Server**: TSDB on disk (block-based; mmap'd indices).
- **Scrape loop**: pulls `/metrics` from configured targets.
- **PromQL**: query language; evaluator in Go.
- **Alertmanager** (separate binary): aggregates and routes alerts.

Notable performance work:
- TSDB compaction is heavy on memory; tuned with `GOMEMLIMIT`.
- Scrape-loop concurrency tuned per target count.
- PromQL evaluator uses interface-driven plan tree; some optimization rounds (constant folding, range query optimization).

### etcd

[etcd](https://github.com/etcd-io/etcd) is the distributed KV store underneath Kubernetes. ~120k LOC Go. Architecture:

- **Raft**: own implementation, not `hashicorp/raft`. Predates and influenced it.
- **MVCC store**: BoltDB underneath.
- **gRPC API**: read, write, watch, transaction.
- **Lease**: TTL-based keys.

etcd's own Raft implementation ([`go.etcd.io/raft`](https://github.com/etcd-io/raft)) is a separate library. Used by CockroachDB and others.

Performance: ~10k writes/sec, ~50k reads/sec on a 3-node cluster.

### containerd

[containerd](https://github.com/containerd/containerd) is the container runtime under Docker, Kubernetes, and others. ~200k LOC Go. Architecture:

- **gRPC API**: clients (Docker, kubelet) talk to containerd.
- **Snapshotter**: manages container filesystems (overlay, btrfs, ...).
- **Runtime shim**: spawns `runc` (also Go) to start container processes.
- **CRI plugin**: implements Kubernetes' CRI directly inside containerd.

runc itself is Go; uses cgo for clone() / namespace setup.

### Helm

[Helm](https://github.com/helm/helm) is the package manager for Kubernetes. ~60k LOC Go. Architecture:

- Charts (YAML templates) rendered with Go's `text/template`.
- Releases stored as Kubernetes secrets or configmaps.
- CLI in Go; library importable.

Helm 3 (current) removed the server-side Tiller component; everything client-side.

### CoreDNS

[CoreDNS](https://github.com/coredns/coredns) is the DNS server in every Kubernetes cluster (replaced kube-dns). Plugin-driven; each feature is a Go plugin.

Architecture: a chain of plugins; each plugin can handle, transform, or pass through a query. Used as authoritative DNS for service discovery in Kubernetes.

### Linkerd

[Linkerd](https://github.com/linkerd/linkerd2) is a service mesh. Two parts:
- **Control plane** (Go): destination service, identity service, proxy injector.
- **Data plane** (Rust): the `linkerd2-proxy` runs as a sidecar per pod.

The Rust data plane was chosen for performance and predictability; the Go control plane is conventional Kubernetes-style.

### Cilium

[Cilium](https://github.com/cilium/cilium) is eBPF-based networking and security for Kubernetes. Architecture:
- **eBPF programs** (C, compiled to BPF bytecode): kernel-side packet processing.
- **Cilium agent** (Go): runs on every node; manages eBPF programs, talks to apiserver.
- **Operator** (Go): cluster-wide reconciler.

Cilium's eBPF data plane is the hot path; the agent is "just" a control plane.

### Argo, Flux, Crossplane

GitOps and Kubernetes-extension projects — all Go, all controller-runtime-based:

- **Argo CD**: declarative GitOps.
- **Argo Workflows**: workflow engine for Kubernetes.
- **Argo Rollouts**: progressive delivery (canary, blue-green).
- **Flux**: GitOps toolkit.
- **Crossplane**: cloud resources as Kubernetes custom resources.

### OpenTelemetry Collector

[OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector) is the vendor-neutral observability data pipeline. ~200k LOC Go. Receives telemetry, processes, exports to backends. Pluggable receivers + exporters; >100 plugins.

### Cortex, Thanos, Mimir

Prometheus-compatible long-term storage:
- **Cortex** (CNCF): multi-tenant, horizontally-scalable.
- **Thanos** (CNCF): adds object-storage backend; less scaling, simpler.
- **Mimir** (Grafana Labs, not in CNCF): fork of Cortex.

All Go; all share Prometheus-derived TSDB code.

### Envoy

[Envoy](https://github.com/envoyproxy/envoy) is the C++ proxy that powers Istio, Consul Connect, AWS App Mesh. Not Go — but the **xDS server** (the control plane that pushes config to Envoy) is often Go: [`go-control-plane`](https://github.com/envoyproxy/go-control-plane). Many Istio-related tools and admission controllers are also Go.

### Patterns common across CNCF Go projects

#### 1. Controller-runtime / Operator pattern

See `20-big-tech/01-google-kubernetes.md`. Most CNCF tools that extend Kubernetes are built on [`sigs.k8s.io/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime).

#### 2. Cobra for CLI

Almost every Go CNCF binary uses `github.com/spf13/cobra` for CLI parsing. Kubernetes, Helm, etcd-ctl, Hugo, GitHub CLI, all use cobra.

#### 3. Viper for config

`github.com/spf13/viper` — pairs with Cobra. Reads YAML/JSON/TOML/env vars.

#### 4. Klog / logr / slog

Kubernetes uses [`klog`](https://github.com/kubernetes/klog); CNCF projects increasingly migrate to [`logr`](https://github.com/go-logr/logr) (a structured-logging interface) or stdlib `slog`.

#### 5. gRPC

`google.golang.org/grpc` is universal. Pretty much every CNCF project speaks gRPC for some interface.

#### 6. Prometheus client

`github.com/prometheus/client_golang/prometheus` is the universal metrics library. Every project exposes `/metrics`.

#### 7. OpenTelemetry

`go.opentelemetry.io/otel` for tracing. Replacing OpenTracing across the ecosystem.

### Why CNCF is Go-heavy

- **Kubernetes set the precedent.** Anything Kubernetes-adjacent inherits the same ecosystem.
- **Cobra/Viper/controller-runtime are Go-only**. Switching language means rebuilding tooling.
- **Distribution as single static binary** matches what Kubernetes administrators want.
- **Go culture of writing operators** is mature (Kubebuilder, Operator SDK).
- **Performance is rarely the bottleneck for control planes**. Compile + iterate speed beats per-µs.

### Go versions and CNCF

CNCF projects typically support the **last two Go minor versions**. As Go releases (~every 6 months), projects upgrade within months. The Kubernetes minimum `go` directive is the de-facto floor; everyone tracks it.

### Cross-project patterns

#### Helm chart for any CNCF project

Most CNCF projects ship as Helm charts. Their teams maintain official ones; community variants exist.

#### Operator + CRD

Almost every project has an Operator. Crossplane provides infrastructure-as-K8s; Argo, Flux, ... all install via operators.

#### Cosign + SLSA

CNCF projects increasingly sign releases with [`cosign`](https://github.com/sigstore/cosign) and publish SLSA provenance. Cosign itself is Go.

## Standard Library Hooks

Shared across CNCF Go projects:

- `net/http`: APIs, exposition formats.
- `crypto/tls`: every connection.
- `context`: pervasive.
- `database/sql`: where state isn't etcd.
- `encoding/json`: configs, manifests.
- `gopkg.in/yaml.v3`: YAML for Kubernetes configs.
- `runtime/pprof`: production debugging.
- `runtime/debug.SetMemoryLimit`: container memory.

## Real-World Patterns

### 1. Prometheus instrumentation

```go
var (
    rps = promauto.NewCounter(prometheus.CounterOpts{Name: "rps", Help: "requests"})
    latency = promauto.NewHistogram(prometheus.HistogramOpts{
        Name: "latency_seconds", Help: "request latency",
        Buckets: prometheus.DefBuckets,
    })
)

func handler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(latency)
    defer timer.ObserveDuration()
    rps.Inc()
    w.Write([]byte("hello"))
}
```

Time → histogram. Counter for raw count. Prometheus scrapes both.

### 2. etcd watch

```go
import "go.etcd.io/etcd/client/v3"

cli, _ := clientv3.New(clientv3.Config{Endpoints: []string{"localhost:2379"}})
defer cli.Close()

watchChan := cli.Watch(ctx, "/config/", clientv3.WithPrefix())
for resp := range watchChan {
    for _, ev := range resp.Events {
        fmt.Printf("%s %s -> %s\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
    }
}
```

Streaming watch over a prefix. Classic Kubernetes-style usage.

### 3. Helm Go SDK

```go
import "helm.sh/helm/v3/pkg/action"

settings := cli.New()
cfg := new(action.Configuration)
cfg.Init(settings.RESTClientGetter(), settings.Namespace(), "secret", log.Printf)

install := action.NewInstall(cfg)
install.ReleaseName = "my-release"
chart, _ := loader.Load("./mychart")
rel, _ := install.Run(chart, map[string]any{"replicas": 3})
fmt.Println("installed:", rel.Name)
```

Helm as a library; not just CLI.

### 4. CoreDNS plugin

```go
package myplugin

import (
    "github.com/coredns/coredns/plugin"
    "github.com/coredns/coredns/request"
    "github.com/miekg/dns"
)

type MyPlugin struct{ Next plugin.Handler }

func (p MyPlugin) ServeDNS(ctx context.Context, w dns.ResponseWriter, r *dns.Msg) (int, error) {
    state := request.Request{W: w, Req: r}
    if state.QName() == "magic.example." {
        m := new(dns.Msg)
        m.SetReply(r)
        m.Answer = []dns.RR{...}
        w.WriteMsg(m)
        return dns.RcodeSuccess, nil
    }
    return plugin.NextOrFailure(p.Name(), p.Next, ctx, w, r)
}

func (p MyPlugin) Name() string { return "myplugin" }
```

Plug into a Corefile; CoreDNS chain-of-responsibility.

### 5. OTel collector receiver/exporter

```go
// Pseudo: define a receiver Factory and Exporter Factory in Go
// then register via the otel collector builder
```

Full doc at https://opentelemetry.io/docs/collector/.

## Anti-Patterns & Gotchas

**Building an Operator for everything.** Operators are useful for stateful, reconciled resources. For a simple deploy, just `kubectl apply`.

**Not exposing `/metrics`.** CNCF norm: every binary exposes Prometheus metrics. Skipping it makes it hard to operate.

**Skipping pprof.** Same norm: expose `/debug/pprof/`. Production debugging is invaluable.

**Custom RPC over gRPC.** Pick gRPC; the ecosystem assumes it.

**Custom config format.** Use YAML + JSON unmarshaling; CNCF tooling assumes them.

**Forking instead of plugin.** Many projects (CoreDNS, Crossplane) have plugin systems. Use them.

**Old `gopkg.in/yaml.v2`.** Use v3; v2 has known quirks.

**Manual CRD validation.** OpenAPI schema validation via apiserver is free; use it.

**Skipping `controller-runtime`.** Hand-rolled informers are error-prone. Use the framework.

**Treating CNCF "graduated" as quality stamp.** It's stability/community. Performance and correctness vary; benchmark for your use.

## Performance Notes

(Order-of-magnitude estimates.)

- Prometheus scrape: ms-level overhead per target; ~1k targets at 15 s interval is fine.
- etcd cluster: 10k writes/sec.
- containerd container start: ~100–500 ms (image local).
- Helm install: seconds for moderate charts.
- CoreDNS QPS: 100k+ per pod.
- Linkerd proxy CPU per pod: ~1–5 mCPU baseline; scales with RPS.
- Cilium agent per node: ~50–200 MiB memory.

## How Big Companies Use CNCF Tools

- **Most Fortune 500** running Kubernetes use Prometheus + Helm + ArgoCD or Flux.
- **Goldman Sachs**: published Kubernetes + Prometheus deployments.
- **Salesforce**: builds on containerd directly.
- **Spotify**: heavy Helm + Prometheus user.
- **Reddit**: Kubernetes + CNCF stack.
- **Cloudflare**: Cilium for K8s networking on some clusters.
- **Datadog, Honeycomb**: support OpenTelemetry as ingestion target.
- **Bloomberg, JPMorgan**: heavy users of CNCF observability.

## Source Code References

All open source, mostly Apache-2.0.

- Prometheus: [`prometheus/prometheus`](https://github.com/prometheus/prometheus).
- Prometheus client_golang: [`prometheus/client_golang`](https://github.com/prometheus/client_golang).
- etcd: [`etcd-io/etcd`](https://github.com/etcd-io/etcd).
- etcd raft: [`etcd-io/raft`](https://github.com/etcd-io/raft).
- containerd: [`containerd/containerd`](https://github.com/containerd/containerd).
- runc: [`opencontainers/runc`](https://github.com/opencontainers/runc).
- Helm: [`helm/helm`](https://github.com/helm/helm).
- CoreDNS: [`coredns/coredns`](https://github.com/coredns/coredns).
- Linkerd2 (control plane Go): [`linkerd/linkerd2`](https://github.com/linkerd/linkerd2).
- Cilium: [`cilium/cilium`](https://github.com/cilium/cilium).
- Argo CD: [`argoproj/argo-cd`](https://github.com/argoproj/argo-cd).
- Argo Workflows: [`argoproj/argo-workflows`](https://github.com/argoproj/argo-workflows).
- Flux: [`fluxcd/flux2`](https://github.com/fluxcd/flux2).
- Crossplane: [`crossplane/crossplane`](https://github.com/crossplane/crossplane).
- OTel Collector: [`open-telemetry/opentelemetry-collector`](https://github.com/open-telemetry/opentelemetry-collector).
- Cortex: [`cortexproject/cortex`](https://github.com/cortexproject/cortex).
- Thanos: [`thanos-io/thanos`](https://github.com/thanos-io/thanos).
- Mimir (Grafana): [`grafana/mimir`](https://github.com/grafana/mimir).
- Envoy Go control plane: [`envoyproxy/go-control-plane`](https://github.com/envoyproxy/go-control-plane).
- controller-runtime: [`kubernetes-sigs/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime).
- Cosign: [`sigstore/cosign`](https://github.com/sigstore/cosign).

## Further Reading

- CNCF landscape: https://landscape.cncf.io.
- CNCF graduated projects: https://www.cncf.io/projects/.
- Prometheus design: https://prometheus.io/docs/concepts/.
- etcd architecture: https://etcd.io/docs/v3.5/learning/.
- containerd architecture: https://containerd.io/docs/.
- "Helm 3 design" (no Tiller): https://helm.sh/blog/helm-3-released/.
- "Kubernetes API conventions": https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md.
- "Linkerd: data plane vs control plane" — William Morgan: https://linkerd.io.
- "Cilium and eBPF" — talks: https://cilium.io.
- OpenTelemetry docs: https://opentelemetry.io/docs/.
- OperatorHub (community): https://operatorhub.io.

## Exercises / Self-Check

1. Why is Linkerd's data plane Rust but its control plane Go? Articulate the trade-off.
2. Build a tiny Prometheus exporter that exposes a process's open file count as a gauge. Verify with `curl /metrics`.
3. Use etcd's watch API to react to a key prefix. Compare its semantics to a Kubernetes informer.
4. Implement a CoreDNS plugin that resolves `*.foo.local` to a fixed IP. Build, configure with a Corefile, test with `dig`.
5. Survey the imports of one CNCF Go project (say, ArgoCD). Identify which dependencies are also Go-team, HashiCorp-team, Kubernetes-team, or grassroots community.
