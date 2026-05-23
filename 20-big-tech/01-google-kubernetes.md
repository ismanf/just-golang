# Google & Kubernetes — Go at Container-Orchestration Scale

## TL;DR

**Kubernetes** (k8s) is the canonical "Go at scale" project: ~2 million lines of Go, ~3700 contributors, run by virtually every cloud platform. Born at Google in 2014 as the open-source successor to **Borg** (C++) and **Omega** (Go pilot), Kubernetes is **entirely Go** — apiserver, kubelet, scheduler, controllers, every client. Go was chosen because Brendan Burns, Joe Beda, and Craig McLuckie wanted a faster developer cycle than Borg's C++ allowed, with strong concurrency primitives for a control-plane workload. The single biggest scale pain: **Kubernetes' apiserver is a stateless cache over etcd**, and its **runtime memory dominates** — a 5000-node cluster apiserver routinely runs at 20–40 GiB resident, almost entirely informer-cache and watch-event buffers.

## Mental Model

```
                              Kubernetes control plane (Go)
   ┌───────────────────────────────────────────────────────────────┐
   │  kube-apiserver                                                │
   │    - REST + watch over etcd                                    │
   │    - validates, authz, admits, persists                        │
   │    - informer-style watch cache (in-process)                   │
   └──────────┬──────────────────┬───────────────────┬──────────────┘
              ▼                  ▼                   ▼
       kube-controller-mgr  kube-scheduler     kube-cloud-mgr
       (deploy, rs, sa,     (placement)       (LB, routes,
        endpoint, ...)                          volumes)

              etcd (Go) — Raft-replicated KV store
   ┌──────────────────────────────────────────────┐
   │  /registry/pods/<ns>/<name>                  │
   │  /registry/services/...                      │
   │  Watch streams keyed by resourceVersion      │
   └──────────────────────────────────────────────┘

              Worker nodes
   ┌───────────────────────────────────────────────┐
   │  kubelet — node agent: starts containers      │
   │  kube-proxy — iptables/IPVS for Service VIPs  │
   │  CNI plugin (often Go: Calico, Cilium, ...)   │
   │  CRI runtime (containerd/CRI-O — also Go)     │
   └───────────────────────────────────────────────┘
```

The control plane is a set of **independent Go binaries** coordinating via the apiserver. The apiserver is the **only** component that talks to etcd. Everything else uses watch-based informers that subscribe to changes and react.

## Syntax & Basic Usage

A minimal controller using `client-go`:

```go
package main

import (
	"context"
	"fmt"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	cfg, err := clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
	if err != nil { panic(err) }
	cs, err := kubernetes.NewForConfig(cfg)
	if err != nil { panic(err) }

	factory := informers.NewSharedInformerFactory(cs, 30*time.Second)
	pods := factory.Core().V1().Pods().Informer()
	pods.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj any) {
			p := obj.(*corev1.Pod)
			fmt.Println("ADD", p.Namespace, p.Name)
		},
		UpdateFunc: func(_, obj any) {
			p := obj.(*corev1.Pod)
			fmt.Println("UPDATE", p.Namespace, p.Name, p.Status.Phase)
		},
	})

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	factory.Start(ctx.Done())
	factory.WaitForCacheSync(ctx.Done())

	_ = metav1.ObjectMeta{} // silence unused
	<-ctx.Done()
}
```

`client-go` is the canonical Go client. The pattern — informer + work queue + reconciler — is what every k8s controller looks like.

## Deep Dive

### Why Go (the original story)

In 2013, Joe Beda and Brendan Burns prototyped what became Kubernetes inside Google's CADL ("Container as a Linux Daemon") group. Borg was C++; Borglet (the node agent) was 250k+ LOC of C++ with painful build times. Google's internal Omega project had experimented with Go.

The trade-off Beda articulated in his "Borg, Omega, and Kubernetes" paper (ACM Queue, 2016):

- **C++**: max performance, slow iteration, hard for new contributors.
- **Java**: GC pauses problematic for control plane.
- **Go**: faster iteration than C++, predictable GC, strong stdlib for HTTP/JSON.

Beda has said in talks that "we chose Go because we wanted Kubernetes to be open source from day one, and we wanted a language that newcomers could be productive in within a week".

### The informer cache

Every controller subscribes to the apiserver via **watch** — a long-poll HTTP stream of JSON-encoded `WatchEvent`s. The client-side **informer** maintains an in-memory snapshot of the watched resources, indexed by namespace/name. When events arrive, the informer updates the cache and dispatches to event handlers.

Key consequences:

- Controllers never query the apiserver in their hot loop — they read the local cache.
- The apiserver maintains a similar in-memory **watch cache** to serve watches efficiently.
- Memory grows linearly with the number of objects: 100k pods × 5 KiB average = 500 MiB just for pod cache, multiplied across every controller that watches pods.

Informer plumbing lives in [`k8s.io/client-go/tools/cache`](https://github.com/kubernetes/client-go/tree/master/tools/cache).

### The work queue pattern

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

pods.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj any) {
        key, _ := cache.MetaNamespaceKeyFunc(obj)
        queue.Add(key)
    },
    // similar for Update, Delete
})

for {
    key, quit := queue.Get()
    if quit { return }
    if err := reconcile(key.(string)); err != nil {
        queue.AddRateLimited(key)  // requeue with exponential backoff
    } else {
        queue.Forget(key)
    }
    queue.Done(key)
}
```

The work queue:
- **Deduplicates** events: if the same key is added 100 times before being processed, it's processed once.
- **Rate-limits** retries: failures back off exponentially.
- **Concurrency control**: multiple workers pull from one queue.

This pattern is so universal that **controller-runtime** (used by every Operator SDK and Kubebuilder project) wraps it in a `Reconciler` interface.

### kubelet — node-side agent

The kubelet runs on every node. It:

1. Watches the apiserver for pods assigned to its node.
2. Pulls images via the **CRI** (Container Runtime Interface) — gRPC to containerd or CRI-O.
3. Sets up networking via **CNI** plugins (gRPC + CNI spec).
4. Mounts volumes via **CSI** plugins (gRPC).
5. Reports node + pod status back to the apiserver every ~10 s.

Kubelet is ~50k LOC of Go. The hot path is the **PLEG** (Pod Lifecycle Event Generator), which polls the CRI for container state changes and translates them into pod-status updates. PLEG has been a chronic source of latency complaints; recent versions use evented PLEG (gRPC streaming).

### Sustained scale numbers

Kubernetes' scalability targets ([SLOs published by sig-scalability](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)):

- 5000 nodes.
- 150,000 total pods.
- 300,000 total containers.
- 100 pods per node.
- 99% of pod startup latency <5 s for stateful pods.
- 99% of API call latency <1 s.

At 5000 nodes, the apiserver runs at 20–40 GiB resident memory. CPU at 8–16 cores. etcd at 4–16 GiB.

### Go pain points the Kubernetes team has hit

#### 1. GC during list-watch resync

Every 30 s by default, informers do a full resync (re-emit every object to handlers). Combined with watch event bursts, this generates millions of allocations per second, pushing GC.

Mitigations:
- `metav1.ListOptions.ResourceVersionMatch` and `bookmark` events.
- Pagination (`limit=500` per page).
- Watch events skip JSON encoding for some paths.
- Recent versions use **streaming list** (since k8s 1.27) which streams ListResponse without buffering the entire list.

#### 2. `encoding/json` reflection cost

apiserver historically spent significant CPU on JSON marshaling/unmarshaling. Mitigations:
- Pre-generated DeepCopy methods (autogen via `deepcopy-gen`).
- `json-iterator` library replaced `encoding/json` for hot paths.
- The "v1.24 json-iter removal" reverted to stdlib + protobuf for internal traffic.

#### 3. `protobuf` for internal API

Each k8s resource has both JSON (external) and Protocol Buffers (internal, since 1.4) encodings. Protobuf is ~10x faster to marshal. Watch traffic between apiserver and controllers uses protobuf where possible. Tools like `protoc-gen-gogo` (now archived) were used to generate efficient protobuf code.

#### 4. `klog` and structured logging

Kubernetes uses [klog](https://github.com/kubernetes/klog), a fork of `glog`, for logging. The migration to `slog`-compatible structured logging is ongoing as of Kubernetes 1.30+.

#### 5. Memory bloat from watch caches

The apiserver caches every watched object. With 1000 watches across all clients, the same object can effectively be cached many times (controllers each maintain their own copies). The "metrics-server" project demonstrates an alternative: don't watch every object, sample on demand.

#### 6. Goroutine explosion

Each watch consumes goroutines (sender + receiver). At 10k clients × 5 watches = 50k goroutines just for watch fanout, on top of per-request handlers. The team has documented goroutine growth as a scaling cliff: [sig-scalability test results](https://github.com/kubernetes/kubernetes/issues?q=label%3Asig%2Fscalability).

### Go versions used

Kubernetes commits to supporting the **last 3 minor versions** of Go. As of 2026, that's Go 1.24, 1.25, 1.26. The `go.mod` declares the minimum:

```
// kubernetes/go.mod
module k8s.io/kubernetes
go 1.26
```

The Kubernetes team upgrades Go aggressively — major Go releases often correspond to noticeable performance changes in apiserver/etcd benchmarks.

### Controller-runtime and Operator SDK

[`sigs.k8s.io/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime) is the framework most production Operators use:

```go
package main

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    corev1 "k8s.io/api/core/v1"
)

type Reconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *Reconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    var pod corev1.Pod
    if err := r.Get(ctx, req.NamespacedName, &pod); err != nil {
        return reconcile.Result{}, client.IgnoreNotFound(err)
    }
    // do work
    return reconcile.Result{}, nil
}

func main() {
    mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{})
    _ = ctrl.NewControllerManagedBy(mgr).
        For(&corev1.Pod{}).
        Complete(&Reconciler{Client: mgr.GetClient(), Scheme: mgr.GetScheme()})
    _ = mgr.Start(ctrl.SetupSignalHandler())
}
```

Built by the Kubernetes API Machinery SIG, controller-runtime hides informer/work-queue/handler plumbing. Most CNCF "operator" projects use it.

### Code generation

Kubernetes leans heavily on **codegen**: `deepcopy-gen`, `client-gen`, `informer-gen`, `lister-gen`. Each resource type spawns ~10k LOC of generated code (deep copies, typed clients, informers, listers). The build system regenerates on schema changes.

For external Operator developers, [Kubebuilder](https://book.kubebuilder.io) wraps these generators. The `controller-tools` repo holds the generators themselves.

### Apiserver memory tuning

The apiserver is the most memory-hungry component. Common knobs:

- `--watch-cache-sizes=pods=5000,services=1000`: cap watch cache entries.
- `--default-watch-cache-size`: default cap.
- `--max-mutating-requests-inflight`, `--max-requests-inflight`: API throttling.
- `--target-ram-mb`: hint to internal pacers.
- `GOGC` and `GOMEMLIMIT` (since 1.19): set via container env. Production runs typically use `GOMEMLIMIT=<container_limit * 0.9>`.

### How Google still uses Borg internally

Google never migrated to Kubernetes internally. **Borg** remains Google's production cluster manager. Kubernetes was open-sourced and developed externally; the internal control plane is unchanged.

That said, Google Cloud (GKE) is one of the largest Kubernetes operators and contributes heavily upstream. Bryan Cantrill once called this "Google's gift to the world" — they open-sourced the *concept* without exposing the *implementation*.

## Standard Library Hooks

The Kubernetes codebase touches almost every standard library package. Notable heavy users:

- `net/http`, `net/http/httputil`: apiserver + clients.
- `encoding/json`, `encoding/gob` (rare): wire formats.
- `crypto/tls`: every connection.
- `context`: deep context propagation through every layer.
- `sync.Map`, `sync.RWMutex`: informer-cache concurrency.
- `runtime`, `runtime/pprof`: heavy production profiling.
- `runtime/debug.SetMemoryLimit`: container-aware memory tuning.
- `os/signal` + `signal.NotifyContext`: graceful shutdown of components.

## Real-World Patterns

### 1. Watch-react-reconcile loop

Already shown above. Universal pattern.

### 2. Leader election

```go
import "k8s.io/client-go/tools/leaderelection"

lock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      "my-controller",
        Namespace: "kube-system",
    },
    Client: cs.CoordinationV1(),
}

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock:          lock,
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) { runController(ctx) },
        OnStoppedLeading: func() { os.Exit(0) },
    },
})
```

Active-passive HA with a heartbeat into a `Lease` resource. Used by every replicated controller in k8s (controller-manager, scheduler, cloud-controller-manager).

### 3. Server-side apply

Since 1.16, `kubectl apply` runs server-side. Field ownership is tracked at the apiserver. Multiple controllers can update the same object as long as they own disjoint fields.

### 4. Webhook admission

Custom admission via HTTPS webhooks:

```go
func mutate(w http.ResponseWriter, r *http.Request) {
    var review admissionv1.AdmissionReview
    json.NewDecoder(r.Body).Decode(&review)
    patches := []map[string]any{
        {"op": "add", "path": "/metadata/labels/mutated", "value": "true"},
    }
    pb, _ := json.Marshal(patches)
    review.Response = &admissionv1.AdmissionResponse{
        UID:     review.Request.UID,
        Allowed: true,
        Patch:   pb,
    }
    json.NewEncoder(w).Encode(review)
}
```

The apiserver POSTs `AdmissionReview` to your TLS webhook; you respond with a patch. Used by service meshes (Istio, Linkerd), policy engines (OPA Gatekeeper), and security tools (Falco).

### 5. CRDs + Operators

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.example.com
spec:
  group: example.com
  names:
    kind: Backup
    listKind: BackupList
    plural: backups
    singular: backup
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              source: { type: string }
```

Define a CRD; the apiserver now serves `/apis/example.com/v1/backups`. Write a Go Operator (controller-runtime) that reconciles `Backup` resources. The pattern has produced thousands of community Operators.

## Anti-Patterns & Gotchas

**Polling the apiserver in a hot loop.** Use watches; the cache is local.

**Forgetting `WaitForCacheSync` before reading from informers.** Will read empty results until sync completes.

**Long-running work in event handlers.** Handlers are called serially; block one and all others stall. Push keys into a work queue.

**Custom resources without status subresource.** Without it, status updates collide with spec updates causing optimistic-concurrency-control errors.

**Holding the apiserver client across goroutines without `client-go`'s retry mechanisms.** Hand-rolled retry is rarely as good.

**Using `corev1.Pod` directly across versions.** k8s API versions are not interchangeable; use a versioned client.

**Designing CRDs without considering watch traffic.** A CRD with 1M instances stresses every controller that watches it. Use label selectors and field selectors to scope watches.

**Skipping leader election.** Two active controllers will fight, each undoing the other's changes.

**Building Operators without test envs.** [envtest](https://book.kubebuilder.io/reference/envtest) starts an in-process apiserver + etcd for tests; without it, controllers are nearly impossible to test deterministically.

**Treating Operator memory growth as "k8s problem".** It's usually an informer cache holding too much. Use label/field selectors to scope.

## Performance Notes

- Apiserver request throughput: ~10k QPS on a single replica.
- Watch event throughput: ~50k events/sec total across all watches.
- etcd write throughput: ~10k ops/sec.
- Pod startup latency (image already pulled): p99 ~3–5 s.
- Pod startup latency (cold image): p99 5–30 s.
- Schedule decision: <100 ms for clusters up to 1000 nodes; ~500 ms at 5000.
- Apiserver memory: ~2 GiB per 10k pods + cache overhead.
- Kubelet memory per node: ~100–300 MiB.

Numbers from sig-scalability test suites; vary with cluster shape.

## How Other Big Companies Use It

- **Red Hat / OpenShift**: largest Kubernetes-derivative product; ships its own controllers and operators in Go.
- **Amazon EKS**, **Microsoft AKS**, **Google GKE**: managed offerings; significant upstream contributors.
- **Lyft**, **Pinterest**, **Spotify**: large in-house operators in Go for service management.
- **Shopify**: runs Kubernetes at Black Friday scale; published [blog series on tuning](https://shopify.engineering/).
- **CERN**: uses Kubernetes for LHC data processing.
- **Stack Overflow**: Kubernetes-hosted; published architecture blog series.

## Source Code References

Pinned to `kubernetes/kubernetes` v1.30+.

- Apiserver: [`kubernetes/staging/src/k8s.io/apiserver/`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/apiserver).
- Kubelet: [`kubernetes/pkg/kubelet/`](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet).
- Scheduler: [`kubernetes/pkg/scheduler/`](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler).
- client-go: [`kubernetes/staging/src/k8s.io/client-go/`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/client-go).
- Informer factory: [`tools/cache/shared_informer.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/tools/cache/shared_informer.go).
- Workqueue: [`util/workqueue/`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/client-go/util/workqueue).
- Leader election: [`tools/leaderelection/`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/client-go/tools/leaderelection).
- Watch protocol: [`apimachinery/pkg/watch/watch.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/watch/watch.go).
- controller-runtime: [`kubernetes-sigs/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime).
- Kubebuilder: [`kubernetes-sigs/kubebuilder`](https://github.com/kubernetes-sigs/kubebuilder).

(Apache-2.0 © The Kubernetes Authors.)

## Further Reading

- "Borg, Omega, and Kubernetes" (ACM Queue, 2016): https://queue.acm.org/detail.cfm?id=2898444.
- "Large-scale cluster management at Google with Borg" (EuroSys 2015): https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/.
- "Kubernetes: Up and Running" — Burns, Beda, Hightower (book).
- "Programming Kubernetes" — Hausenblas, Schimanski (book).
- Kubernetes design proposals: https://github.com/kubernetes/enhancements.
- sig-scalability SLOs and reports: https://github.com/kubernetes/community/tree/master/sig-scalability.
- Joe Beda, "The history of Kubernetes" (TGIK livestreams): https://github.com/heptio/tgik.
- Brendan Burns, "Designing Distributed Systems" (book).
- Daniel Smith, "API server architecture" (KubeCon talks): https://kccncna.com.

## Exercises / Self-Check

1. Sketch the data flow when `kubectl apply -f pod.yaml` is run. Where does the request travel, where is it persisted, and which controllers see it?
2. Why does the apiserver maintain a watch cache instead of streaming directly from etcd to clients?
3. Compute the memory cost of caching 100k pods of 5 KiB each across 50 controllers. Why is this only an upper bound?
4. Implement a minimal controller (informer + work queue + reconciler) that labels new pods with a timestamp. Verify with `kubectl describe`.
5. The kubelet's PLEG was historically a latency bottleneck. Why is "poll the CRI for state" expensive at scale, and what does evented PLEG change?
