# Netflix — Go in a Java-First Shop

## TL;DR

Netflix's backend reputation is famously **Java + JVM** — Hystrix, Zuul, Eureka, RxJava, Spring Cloud — but Go has carved out specific niches. Notable Go usage at Netflix: **Rend** (Memcached protocol proxy and L1/L2 caching), **ChaosMonkey** v2 (chaos engineering), parts of the **Spinnaker** delivery platform, **Bless** (SSH CA service), and various infrastructure agents. Netflix open-sourced many tools; their Go projects influence the broader ecosystem. The single biggest gotcha: **Netflix Go is mostly *infrastructure tooling*, not user-facing services**. Their video streaming and recommendation paths remain Java/JVM. Reading Netflix engineering blogs gives the impression Go is everywhere; the reality is "Go for ops tools, Java for the data plane".

## Mental Model

```
   Netflix simplified architecture:
   
   ┌──────────────────────────────────────────────────────────┐
   │  Edge / client tier                                       │
   │   - Zuul 2 (Java) — API gateway                           │
   │   - Eureka, Ribbon — service discovery + LB (Java)        │
   └─────────────────┬────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Microservices tier                                       │
   │   - 1000+ services, mostly Java (Spring Boot, RxJava)     │
   │   - Cassandra, EVCache (Memcached) for state              │
   └─────────────────┬────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Infrastructure / ops tier (Go appears here)              │
   │   - Rend: cache proxy + L1/L2 logic                       │
   │   - Spinnaker: pipelines + deployment                     │
   │   - Bless: SSH CA                                         │
   │   - ChaosMonkey, Chaos Kong: failure injection            │
   │   - Various agents on every node                          │
   └──────────────────────────────────────────────────────────┘
```

The pattern: Java for business logic; Go for "high-fan-out infrastructure where a single-binary deploy beats JVM startup".

## Syntax & Basic Usage

A Rend-style cache proxy skeleton:

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"net"
)

func handle(ctx context.Context, c net.Conn) {
	defer c.Close()
	r := bufio.NewReader(c)
	for {
		line, err := r.ReadString('\n')
		if err != nil { return }
		// Parse memcached text protocol: "get key\r\n"
		fmt.Fprintf(c, "VALUE %s 0 4\r\nDATA\r\nEND\r\n", "key")
		_ = line
	}
}

func main() {
	ln, _ := net.Listen("tcp", ":11211")
	for {
		c, err := ln.Accept()
		if err != nil { return }
		go handle(context.Background(), c)
	}
}
```

Rend speaks the Memcached binary + text protocols on the front and back. Its job: deduplicate hot keys, do L1/L2 routing, manage compression and chunking transparently.

## Deep Dive

### Why Go at Netflix (the specific cases)

Netflix's published reasoning, post-by-post:

#### Rend

[Rend](https://github.com/Netflix/rend) is Netflix's high-performance Memcached server / proxy. From their 2016 post:

- **Why not stay in Java**: per-process JVM overhead (multi-hundred-MB) made co-locating many cache instances on one host wasteful.
- **Why Go**: predictable memory, fast startup, easy concurrent I/O.
- **Outcome**: Rend handles **multi-million ops/sec per instance** with low overhead. Used in production fronting EVCache.

#### ChaosMonkey v2

[ChaosMonkey](https://github.com/Netflix/chaosmonkey) v2 (rewritten in Go from Java) is the chaos-engineering tool that randomly terminates instances to test failure handling. From their post:

- v1 (Java, on Spring Cloud): heavy, hard to deploy.
- v2 (Go): single binary, runs as a sidecar in Spinnaker pipelines, configurable per-app.

#### Bless

[Bless](https://github.com/Netflix/bless) is Netflix's SSH Certificate Authority — issues short-lived SSH certificates instead of long-lived keys. Originally Python on Lambda; sister projects in Go for CLI tooling.

#### Spinnaker

[Spinnaker](https://github.com/spinnaker/spinnaker) is the multi-cloud delivery platform. Mostly Java (Spring) but with Go components for some sidecars and the `clouddriver-aws` plugins; Spinnaker's CLI `spin` is Go.

### Rend architecture in detail

Rend sits in front of EVCache (Memcached). Reasons:

1. **Chunking**: Memcached's max value size is 1 MiB. Rend transparently splits larger values across multiple keys; recombines on read.
2. **L1/L2 caching**: a fast local L1 (small, in-process) fronts the EVCache cluster (L2). Rend routes get/set decisions.
3. **Compression**: zstd or LZ4 on the wire, transparent to clients.
4. **Protocol translation**: clients can use text or binary Memcached protocols; Rend bridges.
5. **Health checking**: actively probes backend EVCache nodes; routes around degraded ones.

Implementation highlights:
- Single-process Go server, async I/O via goroutine-per-connection.
- Lock-free request queues.
- Heavy use of `sync.Pool` for buffers.
- ~200k lines of careful Go.

### Performance work

Netflix's blog has multiple deep dives. Key learnings:

#### 1. Goroutine accounting under bursty load

Rend tracks active connection counts; spawning unbounded goroutines led to scheduler pressure under burst.

#### 2. Memory allocator tuning

Like Twitch, Discord, Cloudflare — Netflix engineers profile allocator pressure carefully. `sync.Pool` for hot buffers; arena-style allocation for protocol parsing.

#### 3. cgo for binary memcached protocol

For the highest-throughput path, some parsing dropped into hand-written `unsafe` code. Decisions documented in 2017 GopherCon talks.

### Netflix's stance: "Use what fits"

Netflix's open-source culture means lots of public material, but it's biased toward the *interesting* (Go) cases. Reading the volume of posts, you'd think Go is dominant; reality is Java still runs the bulk of traffic.

Netflix engineers in talks have summarized:

> "We don't have a language religion. Java is where the bulk of our developers and Spring expertise live. Go is excellent for sidecars, agents, and infra services where a single binary is the right deploy unit."

### Other Netflix Go projects

#### Atlas (open-source)

[Atlas](https://github.com/Netflix/atlas) — Netflix's time-series monitoring system. Primarily Scala. Their backend exposes Atlas as Prometheus-compatible; some adapters are Go.

#### Dyno

[Dyno](https://github.com/Netflix/dyno) — multi-region Dynomite client. Java; not Go.

#### Hystrix (deprecated)

Java circuit breaker library. Replaced by Resilience4j. The pattern (circuit breaker) has Go equivalents (Sony's `gobreaker`).

#### Spectator

Metrics library for Atlas. Spectator is Java; Spectator-py for Python; some Go shims exist for sidecar tools.

#### Spinnaker-Halyard

Spinnaker installer; mixed Java + Go.

#### Conductor (deprecated)

Netflix's workflow orchestrator. Java; their workflow engineering team eventually moved on (Cadence/Temporal won the broader ecosystem).

### Netflix and the Go community

Several Netflix engineers have given GopherCon talks on Go performance, including:
- Brian Bockelman (Caching).
- Various posts on Rend, ChaosMonkey, Bless.

Netflix's open-source approach influenced patterns elsewhere — chaos engineering, immutable deploys, blue-green via Spinnaker.

### The Java-dominated reality

Netflix has ~700 microservices. The vast majority are Java/Spring Boot. Their CI/CD, observability, and dependency injection patterns are JVM-shaped. Go enters where:
- Resource overhead matters (cache proxies).
- Operational tools need fast iteration.
- Polyglot infrastructure benefits from Go's deployment story.

This is *the* big-tech use case for Go beyond Kubernetes-centric companies.

## Standard Library Hooks

For a Netflix-style cache proxy in Go:

- `net`, `bufio`, `io`: TCP plumbing.
- `encoding/binary`: binary protocol parsing.
- `sync.Pool`: buffer reuse.
- `compress/gzip` + third-party `klauspost/compress` for LZ4/Zstd.
- `runtime/pprof`: continuous profiling.
- `context`: per-connection timeouts.
- `golang.org/x/sync/singleflight`: deduplicate concurrent cache fills.

## Real-World Patterns

### 1. L1/L2 cache with singleflight

```go
import "golang.org/x/sync/singleflight"

type LayeredCache struct {
    l1, l2 Cache
    g      singleflight.Group
}

func (c *LayeredCache) Get(ctx context.Context, key string) ([]byte, error) {
    if v, ok := c.l1.Get(key); ok {
        return v, nil
    }
    v, err, _ := c.g.Do(key, func() (any, error) {
        if v, ok := c.l2.Get(key); ok {
            c.l1.Set(key, v.([]byte))
            return v, nil
        }
        v, err := backend.Fetch(ctx, key)
        if err == nil {
            c.l2.Set(key, v)
            c.l1.Set(key, v)
        }
        return v, err
    })
    if err != nil { return nil, err }
    return v.([]byte), nil
}
```

`singleflight` ensures only one concurrent request actually hits the backend per key.

### 2. Chunked value support

```go
const maxChunk = 1 << 20 // 1 MiB

func putLarge(c Cache, key string, val []byte) error {
    n := (len(val) + maxChunk - 1) / maxChunk
    for i := 0; i < n; i++ {
        chunkKey := fmt.Sprintf("%s:chunk:%d", key, i)
        end := (i + 1) * maxChunk
        if end > len(val) { end = len(val) }
        if err := c.Set(chunkKey, val[i*maxChunk:end]); err != nil {
            return err
        }
    }
    return c.Set(key+":meta", []byte(fmt.Sprintf("%d", n)))
}
```

Transparent splitting; readers reassemble via the metadata key.

### 3. Bounded TCP listener

```go
sem := make(chan struct{}, 10000)
for {
    c, err := ln.Accept()
    if err != nil { return }
    sem <- struct{}{}
    go func() {
        defer func() { <-sem }()
        handle(c)
    }()
}
```

Caps concurrent connections at 10k. Crucial for memory predictability in a cache proxy.

### 4. Configurable circuit breaker

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "evcache",
    MaxRequests: 100,
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
})

result, err := cb.Execute(func() (any, error) {
    return backend.Fetch(ctx, key)
})
```

When backend failures spike, the breaker opens for `Timeout`, short-circuits all calls. Allows the backend to recover.

### 5. ChaosMonkey-style instance termination

```go
type Terminator interface {
    Kill(ctx context.Context, instanceID string) error
}

func chaos(t Terminator, instances []string, probability float64) {
    for _, inst := range instances {
        if rand.Float64() < probability {
            log.Printf("chaos: terminating %s", inst)
            _ = t.Kill(context.TODO(), inst)
        }
    }
}
```

ChaosMonkey runs daily; deliberate failure-injection forces every team to be prepared.

## Anti-Patterns & Gotchas

**One Go process for every microservice "to follow Netflix's style".** Their style is Java. Go is for specific niches.

**Hand-rolling a Memcached proxy when Rend exists.** Open-source it: https://github.com/Netflix/rend.

**Sidecar-everything pattern.** Each language has its own sidecar; ops overhead grows. Service meshes (Linkerd, Istio) consolidate.

**Going Go without observability.** Netflix's infra requires Spectator/Atlas integration; sidecar libraries exist for this.

**Forgetting that Spring's `@Inject` doesn't exist in Go.** Use fx (Uber) or wire (Google) — see `19-patterns/03-dependency-injection.md`.

**Skipping graceful shutdown.** Cache proxies must drain in-flight requests on SIGTERM.

**Trusting client retry alone.** Implement circuit breakers; clients retrying every 100 ms will kill recovering backends.

**Building infra without ChaosMonkey-style tests.** Failure injection in staging catches what passive tests miss.

**Trying to replace Spinnaker.** Hard. Use it; contribute upstream.

## Performance Notes

(From public Netflix posts / benchmarks; rough.)

- Rend throughput: 1M+ ops/sec per instance.
- Rend latency: p99 <1 ms (intra-AZ).
- ChaosMonkey: lightweight; runs as a daemon, low resource cost.
- Bless: low QPS (SSH cert issuance is rare); latency irrelevant.
- Spinnaker pipelines: minutes-to-hours.
- EVCache fleet aggregate: trillions of requests / day.

## How Big Companies Use It (Netflix-influenced patterns)

- **Lyft**: their `clouddriver` integrations with Spinnaker.
- **Box**: cache proxy patterns informed by Rend.
- **Slack**: chaos engineering culture inspired by Netflix.
- **Airbnb**: deploys via Spinnaker.
- **Adobe**: chaos engineering tooling, partly Go.
- **Cisco**: heavy Spinnaker user, sidecar agents in Go.
- **Walmart Labs**: Go-based caches modeled after Rend.
- **Capital One**: Spinnaker + Bless adoption.

## Source Code References

- Rend: [`Netflix/rend`](https://github.com/Netflix/rend).
- ChaosMonkey (v2): [`Netflix/chaosmonkey`](https://github.com/Netflix/chaosmonkey).
- Bless (SSH CA): [`Netflix/bless`](https://github.com/Netflix/bless).
- Spinnaker: [`spinnaker/spinnaker`](https://github.com/spinnaker/spinnaker).
- spin (Spinnaker CLI, Go): [`spinnaker/spin`](https://github.com/spinnaker/spin).
- Atlas (Scala, with adapters): [`Netflix/atlas`](https://github.com/Netflix/atlas).
- All Netflix open-source: [`github.com/Netflix`](https://github.com/Netflix).
- sony/gobreaker (used by many): [`github.com/sony/gobreaker`](https://github.com/sony/gobreaker).
- golang.org/x/sync/singleflight (used in caching): [`golang.org/x/sync`](https://github.com/golang/sync).

## Further Reading

- "Application Auto Scaling with Chaos Monkey" — Netflix blog: https://netflixtechblog.com.
- "Rend: A Memcached server in Go" (announcement post): https://netflixtechblog.com/.
- "Resilience Engineering at Netflix" (chaos talks).
- "Spinnaker: Hello Open Source CD" — https://spinnaker.io.
- Adrian Cockcroft, "Microservices at Netflix" (talks).
- Brian Bockelman, "EVCache architecture": https://netflixtechblog.com/.
- "Why we built our own Memcached server" — Rend deep dive.
- Bless overview: https://github.com/Netflix/bless.

## Exercises / Self-Check

1. Implement a Memcached text-protocol proxy that returns cached values from an in-process map; benchmark single-threaded ops/sec.
2. Add chunking: values larger than 1 MiB are split across N sub-keys with a metadata key listing them.
3. Implement a `golang.org/x/sync/singleflight`-based loader for a hot cache. Verify with a load test that only one backend call happens per key under concurrent demand.
4. Write a ChaosMonkey-style daemon that lists Kubernetes pods in a namespace and randomly deletes one every minute with probability 0.05. Add a safety: never delete the last instance of a deployment.
5. Why does Netflix keep Java for the data plane but Go for cache proxies? Articulate the engineering trade-off in two sentences.
