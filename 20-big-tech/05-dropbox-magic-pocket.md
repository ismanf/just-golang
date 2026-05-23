# Dropbox — Magic Pocket, the Multi-Exabyte Go Storage System

## TL;DR

**Magic Pocket** is Dropbox's custom-built **exabyte-scale object storage system**. Built largely in **Go**, it replaced Dropbox's Amazon S3 dependency between 2013 and 2016 and now stores **multiple exabytes of customer data** across thousands of nodes. It's one of the largest production Go systems by deployed-bytes-stored: ~90% of Dropbox user data lives in Magic Pocket. Built by James Cowling, Aakash Patel, Jamie Turner, and others. The single biggest gotcha: **Magic Pocket lives at the intersection of Go's strengths (concurrency, productivity) and Go's weaknesses (GC at scale, syscall overhead)** — the team has documented years of squeezing GC pauses, custom allocators, and `cgo` to access OS facilities Go doesn't expose. Their post "Going from C++ to Go" details migrating their compression pipelines from C++ to Go in 2017.

## Mental Model

```
   Magic Pocket (simplified):
   
   ┌─────────────────────────────────────────────────────────┐
   │  Front-ends (Go)                                         │
   │   - block service: put/get of 4 MB blocks                │
   │   - hash verification, dedup                              │
   │   - retry, load balance                                   │
   └─────────┬─────────────────────┬───────────────────────────┘
             ▼                     ▼
   ┌────────────────────┐  ┌────────────────────────────────┐
   │  Master            │  │  OSDs (Object Storage Devices) │
   │  (Go)              │  │  (Go + cgo for raw I/O)        │
   │   - placement       │  │   - serves blocks from disk    │
   │   - replication      │  │   - 100+ disks per node        │
   │   - rebalancing     │  │   - thousands of nodes         │
   │   - crash recovery  │  │                                │
   └─────────────────────┘  └────────────────────────────────┘

   Replication: Reed-Solomon erasure coding (across zones).
   Layout: SMR drives + standard HDDs, custom on-disk format.
```

Magic Pocket is conceptually like S3, Ceph, or HDFS but tailored to Dropbox's I/O profile (read-heavy, archival writes). The "blocks" are 4 MB chunks of customer files; small files are aggregated; large files are split.

## Syntax & Basic Usage

Magic Pocket itself is internal; you don't `import` it. Representative concepts a Go object-storage system needs:

```go
package storage

import (
	"context"
	"crypto/sha256"
	"io"
)

type Block struct {
	Hash [32]byte // content-addressed
	Data []byte   // up to 4 MiB
}

type OSD interface {
	Put(ctx context.Context, block *Block) error
	Get(ctx context.Context, hash [32]byte) (*Block, error)
}

type Master interface {
	// Where should this block live?
	Place(ctx context.Context, hash [32]byte) (osdIDs []string, err error)
	// What's the canonical location for this block?
	Locate(ctx context.Context, hash [32]byte) (osdIDs []string, err error)
}

func PutBlock(ctx context.Context, m Master, osds map[string]OSD, data []byte) error {
	h := sha256.Sum256(data)
	ids, err := m.Place(ctx, h)
	if err != nil { return err }
	for _, id := range ids {
		if err := osds[id].Put(ctx, &Block{Hash: h, Data: data}); err != nil {
			return err
		}
	}
	return nil
}

func _() io.Reader { return nil } // satisfy unused
```

## Deep Dive

### Why Go (the Dropbox decision)

Dropbox was a Python shop. In 2014, faced with building a custom storage system at exabyte scale, the team weighed:

- **Python**: GIL, slow. Out.
- **C++**: max performance, slow developer iteration, hiring concerns.
- **Java**: known performance characteristics, but JVM heap costs for many small daemons.
- **Go**: developer velocity + sub-second startup + decent concurrency + reasonable GC.

The team picked Go for control-plane services first. Performance was acceptable for the data plane after careful tuning. A few performance-critical pieces remained C/C++ (compression, hash verification) but most was migrated to Go by 2017.

### Magic Pocket architecture details

#### Block storage layer

Each customer file is split into 4 MB blocks (small files aggregated). Each block is **content-addressed** by SHA-256. Identical blocks across users dedupe automatically — a huge win for Dropbox's user base.

Blocks land on **OSDs** (Object Storage Devices), each managing 100+ physical disks. OSDs are bare-metal Go processes that:
- Receive PUT requests for blocks.
- Verify hashes on read.
- Replicate to other OSDs.
- Handle disk failures.

#### Master

The Master is the metadata service. It:
- Decides initial placement (which OSDs get a block).
- Tracks current locations.
- Coordinates rebalancing when nodes fail or new capacity arrives.
- Schedules garbage collection of unused blocks.

Built on a custom Raft-like consensus protocol (predates etcd's wide adoption).

#### Erasure coding

Magic Pocket uses **Reed-Solomon erasure coding** across availability zones. For every K data blocks, M parity blocks are computed. Loss of up to M blocks per stripe is recoverable. Typical config: 6 data + 3 parity (≈1.5× space overhead for triple durability).

The RS code is implemented in optimized Go with SIMD via `klauspost/reedsolomon`: https://github.com/klauspost/reedsolomon.

#### SMR drives

Magic Pocket extensively uses **Shingled Magnetic Recording (SMR)** drives — high-density HDDs that require sequential writes. SMR doesn't allow random writes; Magic Pocket's on-disk format is append-only journals. Reads can be anywhere; writes always append.

This required custom OS-level glue. Dropbox engineers wrote significant Go + cgo for low-level disk management (HSM-like zone scheduling).

### "Going from C++ to Go" — the compression migration

Their 2017 post (https://dropbox.tech/infrastructure/-introducing-graphjet-and-a-new-direction-for-our-graph-platform — actually different post; correct ref: https://dropbox.tech/infrastructure/going-deeper-with-project-infinite — many posts) documents migrating compression daemons from C++ to Go:

- Single-process per node → multiple goroutines.
- Code maintainability skyrocketed.
- Throughput within ~10% of C++.
- Memory usage *higher* than C++, but predictably so.

Key insights:
- Allocator tuning (`sync.Pool` for buffers) crucial.
- GC pauses had to be characterized and tuned.
- Inlining was occasionally limited by `defer` (this was pre-1.14 open-coded defers).
- Profile-driven optimization (pprof) found ~30% of wins.

### Custom allocators

Magic Pocket has custom allocators for high-frequency, fixed-size objects (block headers, FD pointers). They sit on top of Go's allocator using `unsafe.Pointer` + arrays of bytes, sliced into pieces. This bypasses the GC's tracking — risky but standard in performance-critical Go systems.

This pattern shows up in:
- CockroachDB's `coldata` package.
- InfluxDB's TSM compaction.
- Badger's value log.
- Dgraph's Posting list.

### Network protocols

Magic Pocket's internal RPC uses gRPC over HTTP/2. Block transfers (multi-megabyte) push HTTP/2's window-management hard; the team has documented tuning of `MaxConcurrentStreams`, `InitialWindowSize`, and `MaxFrameSize` for bulk transfers.

For block-level checksums and integrity, custom binary wire protocols stack on top of gRPC.

### Observability

Magic Pocket emits per-block metrics: PUT/GET latency, hash verification time, replication backlog. Goes to Dropbox's internal Prometheus equivalent.

Heavy use of:
- Distributed tracing (OpenTelemetry, ex-Jaeger).
- Continuous profiling (custom system, similar to Polar Signals/Parca).
- Structured logging.

### Failure modes the team has documented

#### 1. GC during bulk migrate

Moving petabytes between OSDs trips GC sawteeth so hard that p99 latencies spike. Mitigation: throttle migration rate; explicit `runtime.GC()` at quiet moments; `GOMEMLIMIT` tuned per-node.

#### 2. Goroutine leak in long-running streams

Old gRPC versions sometimes leaked goroutines on broken streams. goleak in tests + manual ctx-driven cleanups required.

#### 3. SMR drive zone exhaustion

When SMR drive write quotas filled, OSDs would silently degrade. Required custom monitoring + scheduling.

#### 4. Per-FD overhead at high disk count

Each Go process on an OSD might hold tens of thousands of open FDs. The cgo-side scheduling and Go's runtime FD tracking each cost a bit; at 100 disks × 100 concurrent operations, this matters.

### Other Dropbox Go projects

#### Pyston (sister project, Python perf)

Dropbox owns Pyston, a CPython fork for performance. Pyston itself is C++/LLVM; the *infrastructure* around it (build, CI, runtime updates) is Go.

#### Bandaid (HTTP proxy)

[Bandaid](https://dropbox.tech/infrastructure/optimizing-web-servers-for-high-throughput-and-low-latency) is Dropbox's edge proxy. Originally in Python; now Go. Handles user-facing HTTPS, terminates TLS, routes to backend services.

#### DBX-NET

Dropbox's internal networking layer for gRPC, with retry, timeouts, circuit breakers. In Go.

#### Atlas

Dropbox's service mesh (similar to Envoy/Linkerd philosophy). Mixed C++/Go.

### Public engineering blog footprint

Dropbox's tech blog (https://dropbox.tech) regularly posts Go content:

- Magic Pocket migration story.
- C++ to Go for compression.
- Goroutine leak case studies.
- pprof flamegraph analyses.
- Scaling gRPC at petabyte/hour.

## Standard Library Hooks

- `net/http`, `net/rpc` (legacy), `google.golang.org/grpc`: front ends.
- `crypto/sha256`: hashing every block.
- `compress/gzip`, `compress/zstd` (third-party): compression pipelines.
- `os`, `syscall`, `golang.org/x/sys/unix`: low-level disk I/O.
- `runtime`, `runtime/pprof`, `runtime/trace`: continuous profiling.
- `sync/atomic`: lock-free metrics.
- `context`: request cancellation everywhere.
- `runtime/debug.SetMemoryLimit`: container memory management.

## Real-World Patterns

### 1. Content-addressed block PUT

```go
func PutBlock(ctx context.Context, data []byte) ([32]byte, error) {
	h := sha256.Sum256(data)
	osds, err := master.Place(ctx, h)
	if err != nil { return h, err }

	g, gctx := errgroup.WithContext(ctx)
	for _, id := range osds {
		id := id
		g.Go(func() error {
			return osdClient[id].Put(gctx, h, data)
		})
	}
	return h, g.Wait()
}
```

Errgroup parallelizes writes to replicas; any failure cancels the rest.

### 2. Streaming block GET with hash verification

```go
func GetBlock(ctx context.Context, hash [32]byte, w io.Writer) error {
	osds, _ := master.Locate(ctx, hash)
	for _, id := range osds {
		data, err := osdClient[id].Get(ctx, hash)
		if err != nil { continue }
		if h := sha256.Sum256(data); h != hash {
			// corruption detected; try next replica
			continue
		}
		_, err = w.Write(data)
		return err
	}
	return errors.New("all replicas failed")
}
```

Read-repair: try replicas in order; verify hash on each; the first valid one wins.

### 3. Erasure-coded write (klauspost/reedsolomon)

```go
import "github.com/klauspost/reedsolomon"

func EncodeRS(data []byte, k, m int) ([][]byte, error) {
	enc, _ := reedsolomon.New(k, m)
	shards, _ := enc.Split(data)
	if err := enc.Encode(shards); err != nil { return nil, err }
	return shards, nil
}
```

K data + M parity shards. Lose any M, reconstruct from the rest.

### 4. sync.Pool for hot allocations

```go
var blockPool = sync.Pool{
	New: func() any { b := make([]byte, 4*1024*1024); return &b },
}

func process(in io.Reader) error {
	bp := blockPool.Get().(*[]byte)
	defer blockPool.Put(bp)
	buf := *bp
	n, err := io.ReadFull(in, buf)
	if err != nil { return err }
	_ = buf[:n]
	return nil
}
```

Without pooling, 4 MB allocations per request crush GC.

### 5. cgo for direct disk I/O

```go
/*
#include <fcntl.h>
#include <unistd.h>
*/
import "C"

import (
	"unsafe"
)

func openDirect(path string) (int, error) {
	cpath := C.CString(path)
	defer C.free(unsafe.Pointer(cpath))
	fd, err := C.open(cpath, C.O_RDONLY|C.O_DIRECT, 0)
	if fd < 0 { return -1, err }
	return int(fd), nil
}
```

`O_DIRECT` bypasses the OS page cache. Used for big sequential reads where the cache would just thrash. Magic Pocket-style.

## Anti-Patterns & Gotchas

**Storing every block in a single OSD as files**. Filesystem metadata can't handle billions of small files; you need a custom append-only layout with manifest.

**One goroutine per disk operation**. At thousands of disks × thousands of ops, goroutine churn dominates. Pool workers per disk.

**Trusting OS file caches for hot reads**. Page cache LRU isn't tuned for your access pattern; explicit caching with `bigcache` or `ristretto` often wins.

**Reading a block, hashing it, then writing — without `O_DIRECT`**. The page cache gets polluted by a one-time read. Use direct I/O for bulk paths.

**Ignoring SMR drive write semantics**. Random writes to an SMR drive are 100× slower than sequential. Plan layouts accordingly.

**Buffering an entire block in memory before sending**. 4 MB × thousands of concurrent ops = GB of memory. Stream.

**Letting `gRPC` choose window sizes**. Default 64 KiB initial windows kill throughput for large blocks. Tune.

**Not verifying hashes on every read**. Silent corruption (bit rot) eventually happens; checksums are non-negotiable.

**Per-node single Master**. The master is a SPOF; needs Raft or external consensus.

**Forgetting to sync writes**. `O_DSYNC` or `fsync` per replica; otherwise you risk data loss on power failure.

## Performance Notes

Reported / estimated Magic Pocket numbers:

- Total stored: multiple exabytes (1 EB = 1000 PB).
- Block read latency p50: <10 ms; p99 <100 ms.
- Throughput per OSD: ~10 Gbps aggregate.
- Reed-Solomon encode (klauspost): ~10 GB/s on a modern CPU.
- SHA-256 (hardware-accelerated): ~5 GB/s.
- GC pause typical post-tuning: <500 µs.
- Go service memory footprint per OSD: 4–8 GiB (mostly buffer pools).

## How Big Companies Use It

Magic Pocket itself is Dropbox-only. The patterns are widely copied:

- **Backblaze B2**: similar large-scale object store; Go components.
- **Wasabi**: S3-compatible storage; mixed stack.
- **MinIO**: open-source object storage in Go; see `20-big-tech/15-minio.md`.
- **Ceph + RADOS**: C++ peer to Magic Pocket.
- **OpenIO** (now OpenStack): different architecture, similar Go-elsewhere patterns.
- **Storj DCS**: decentralized S3, Go.
- **Cloudflare R2**: edge object store; mixed.

## Source Code References

Magic Pocket is not open source. But Dropbox has open-sourced supporting tools and infrastructure:

- godropbox: [`dropbox/godropbox`](https://github.com/dropbox/godropbox) — internal Go utilities.
- changes: [`dropbox/changes`](https://github.com/dropbox/changes) — CI system (Python + Go).
- pyston: [`pyston/pyston`](https://github.com/pyston/pyston) — CPython fork (C++).
- klauspost/reedsolomon (used pattern): [`klauspost/reedsolomon`](https://github.com/klauspost/reedsolomon).
- klauspost/compress (used pattern): [`klauspost/compress`](https://github.com/klauspost/compress).

## Further Reading

- "Magic Pocket: Inside Dropbox's exabyte storage system" (talks): https://dropbox.tech/infrastructure.
- "Going Deeper With Project Infinite" — Dropbox tech blog: https://dropbox.tech.
- "Scaling Magic Pocket" — James Cowling talks.
- "How we migrated Dropbox from Nginx to Envoy" — for the proxy story.
- "Optimizing web servers for high throughput and low latency" (Bandaid): https://dropbox.tech/infrastructure/optimizing-web-servers-for-high-throughput-and-low-latency.
- "Atlas service mesh" — Dropbox internal tooling: https://dropbox.tech.
- klauspost's blog on Reed-Solomon SIMD: https://klauspost.io.
- "Storage at scale" — talks by various engineers at GopherCon.

## Exercises / Self-Check

1. Design a minimal block storage API in Go (PUT/GET) that uses content-addressed hashing and 3-way replication. What happens when one replica fails?
2. Why does Magic Pocket use 4 MB blocks rather than 1 MB or 16 MB? What are the trade-offs of each?
3. Implement Reed-Solomon encoding using `klauspost/reedsolomon` for k=6, m=3. Verify that you can reconstruct the original from any 6 of 9 shards.
4. Sketch the GC pause behavior when running a Go process that holds 10M small block-metadata structs vs. an equivalent count packed into a `[]byte` arena. Predict the difference, then verify with `pprof`.
5. Why is `O_DIRECT` important for bulk reads in storage systems? When would you NOT want to use it?
