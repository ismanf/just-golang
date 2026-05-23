# MinIO — S3-Compatible Object Storage in Go

## TL;DR

**MinIO** is the leading open-source S3-compatible object storage system, entirely in Go. Single binary, drop into a Kubernetes pod or bare metal, and you have an S3 endpoint serving petabytes. AS of 2024 it claims tens of thousands of production deployments and **hundreds of exabytes** of total stored capacity globally. MinIO is notable for hitting hardware limits — they routinely benchmark **read/write throughput at ~200 GiB/s per cluster** on dense NVMe arrays. The single biggest gotcha: **MinIO's parallelism story changes everything about how you write Go for I/O-bound workloads** — they push goroutines, syscalls, and `splice/sendfile` so hard that they hit Linux kernel bottlenecks before Go's runtime. Their open-source erasure-coding library (`klauspost/reedsolomon`) is used by virtually every Go data-storage project.

## Mental Model

```
   MinIO cluster:
   
   ┌──────────────────────────────────────────────────────────┐
   │  Client (AWS SDK, mc, boto3 — speaks S3 API)              │
   └─────────────────┬────────────────────────────────────────┘
                     ▼ HTTPS, S3 v4 signatures
   ┌──────────────────────────────────────────────────────────┐
   │  MinIO nodes (single Go binary each)                      │
   │   - Each node serves the S3 API                            │
   │   - Coordinated via consistent hashing                     │
   │   - Erasure coding across disks (within or across nodes)   │
   │   - "Pools": groups of nodes; scale out by adding pools    │
   ├──────────────────────────────────────────────────────────┤
   │  Storage tier:                                              │
   │   - direct file-per-object on each disk                    │
   │   - object split into N data + M parity shards             │
   │   - O_DIRECT for hot paths                                  │
   │   - sendfile / splice on read                              │
   └──────────────────────────────────────────────────────────┘

   Multi-site: replication, bucket policies, ILM (lifecycle).
```

A 4-node MinIO cluster with 16 disks per node gives you N+M erasure coding (typically 8+4). Each object survives M disk failures. Scale capacity by adding pools; reads spread across the active pool.

## Syntax & Basic Usage

S3-compatible. Use any AWS SDK:

```go
package main

import (
	"context"
	"fmt"

	"github.com/minio/minio-go/v7"
	"github.com/minio/minio-go/v7/pkg/credentials"
)

func main() {
	ctx := context.Background()
	cli, err := minio.New("play.min.io", &minio.Options{
		Creds:  credentials.NewStaticV4("Q3AM3UQ867SPQQA43P2F", "zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG", ""),
		Secure: true,
	})
	if err != nil { panic(err) }

	_, _ = cli.MakeBucket(ctx, "test-bucket", minio.MakeBucketOptions{})
	fmt.Println("bucket created")

	// Upload
	info, _ := cli.FPutObject(ctx, "test-bucket", "hello.txt", "/etc/hostname", minio.PutObjectOptions{})
	fmt.Println("uploaded:", info.Size, "bytes")
}
```

`minio-go` is the official Go client; works against MinIO, AWS S3, GCS-S3, Wasabi, etc.

## Deep Dive

### Why Go (MinIO's reasoning)

AB Periasamy (founder, ex-GlusterFS) has stated:
- **Single static binary** = trivial to deploy on any Linux.
- **Cross-platform** with one codebase.
- **gRPC + HTTP stdlib** = no protocol-stack engineering needed.
- **Goroutines map well to "one per concurrent request"**.

Performance was the main risk. The answer: aggressive Go optimization plus heavy use of Linux primitives via cgo (rare) and syscalls (frequent).

### Erasure coding

MinIO uses **Reed-Solomon erasure coding**. Default: 8 data + 4 parity (8+4). An object is split into 8 shards, 4 parity shards are computed, and all 12 are written to 12 different disks (or 12 different nodes in larger deployments).

Lose any 4 disks → still readable. The math comes from `klauspost/reedsolomon` (https://github.com/klauspost/reedsolomon), Klaus Post's SIMD-accelerated library that MinIO sponsored.

`klauspost/reedsolomon` benchmarks: ~10 GiB/s encode on a modern CPU using AVX2; ~25 GiB/s on AVX512.

### O_DIRECT and sendfile

For bulk transfers, MinIO uses:
- `O_DIRECT` on `open()` to bypass the page cache (the data is going somewhere else; caching wastes RAM).
- `splice(2)` / `sendfile(2)` to move bytes from disk to socket without copying through user space.

This requires `unsafe`, alignment, and careful Go code that doesn't violate Go's runtime assumptions. MinIO has documented this in their performance posts.

### Heal scanner

A background goroutine walks every object, verifies hashes, rebuilds missing/corrupt shards. Runs continuously; pacing tunable.

### Multipart uploads

For large objects, S3 supports multipart upload: client uploads parts in parallel; server assembles. MinIO supports this with parallel goroutines writing shards across disks/nodes.

### Lifecycle (ILM)

Object lifecycle rules: tier to lower-cost storage after N days, expire after M days. Implemented as background reconcilers.

### Bucket replication

MinIO supports active-active and active-passive replication across sites. Implementation: each write generates a replication event; a worker pool replicates to destination buckets; conflicts resolved by versioning + timestamps.

### IAM and policies

S3 IAM-compatible policies. MinIO has its own IAM (users, groups, policies) or can federate to OIDC.

### Hardware orientation

MinIO's published documentation strongly recommends:
- Direct-attached storage (NVMe, SATA SSD, JBOD). Not RAID; MinIO does its own redundancy.
- Each disk a separate filesystem (XFS).
- Heavy parallelism: 16+ disks per node typical.

### Go performance lessons documented

#### 1. Goroutine cost at I/O scale

Hundreds of thousands of goroutines is fine; millions is fine; tens of millions starts hurting. MinIO caps per-request goroutines.

#### 2. sync.Pool everywhere

Hot buffers: read/write buffers (1 MiB+), HTTP body buffers, Reed-Solomon working buffers. Pooled obsessively.

#### 3. Avoiding allocator pressure during heavy I/O

Profiling shows even seemingly innocuous code (string concatenation in error paths) becomes a bottleneck at petabyte/day throughput.

#### 4. GC pause tracking

MinIO publishes per-cycle GC stats via `runtime/metrics`. At their scale, even 5 ms pauses are noticed.

#### 5. `runtime/debug.SetMemoryLimit`

Used widely post-1.19. MinIO documents the configuration in their docs.

### KES — key encryption server

[KES](https://github.com/minio/kes) is MinIO's tiny key-management-service. Designed for high concurrency on a single binary. Talks to Vault, AWS KMS, Azure Key Vault as backends; serves keys to MinIO for object encryption.

### MinIO Console

The web UI (https://github.com/minio/console) is a separate React app + Go backend.

### `mc` — the MinIO CLI

[mc](https://github.com/minio/mc) is a swiss-army-knife S3 CLI. Speaks any S3-compatible endpoint, including AWS S3. Used by many even when their primary storage isn't MinIO.

### Mirror, sync, replication

`mc mirror`, `mc cp -r`, MinIO's bucket replication, and the LakeFS-style features — all built on top of the same Go internals.

### Performance numbers

MinIO regularly publishes benchmarks. Headline: ~325 GiB/s read on 32 nodes of dense NVMe. Per-node: ~10 GiB/s sustained.

These numbers depend hugely on hardware. The point is that Go can achieve them; the bottleneck moves to NIC and disk.

### Comparison to Ceph

Ceph is also distributed object storage, written in C++. Pros: mature, more storage models (block, file, object). Cons: complex to operate. MinIO trades feature breadth for operational simplicity.

### Licensing

MinIO was Apache-2.0 for years. In 2021 they switched to AGPLv3. Cloud distributors and self-hosters can use freely; embedded SaaS uses generally require a commercial license. The AGPL move was deliberate — to prevent cloud providers from monetizing without contributing.

## Standard Library Hooks

- `net/http`: S3 API server.
- `crypto/sha256` (hardware-accelerated): per-block hashing.
- `crypto/aes-gcm`: object encryption at rest.
- `os`, `syscall`, `golang.org/x/sys/unix`: O_DIRECT, splice, sendfile.
- `sync.Pool`: buffer reuse.
- `bufio`, `io`: bulk transfers.
- `context`: per-request lifecycle.
- `runtime/pprof`, `runtime/metrics`: continuous profiling.

## Real-World Patterns

### 1. Minimal object PUT/GET via SDK

```go
cli, _ := minio.New("localhost:9000", &minio.Options{
    Creds:  credentials.NewStaticV4("admin", "password", ""),
    Secure: false,
})

// Upload
_, _ = cli.PutObject(ctx, "bucket", "key", strings.NewReader("data"), 4, minio.PutObjectOptions{})

// Download
obj, _ := cli.GetObject(ctx, "bucket", "key", minio.GetObjectOptions{})
defer obj.Close()
io.Copy(os.Stdout, obj)
```

### 2. Multipart upload for large objects

```go
// Parts are bookkept automatically by the SDK for objects > 64 MiB
info, _ := cli.PutObject(ctx, "bucket", "big.dat", largeFile, fileSize,
    minio.PutObjectOptions{PartSize: 64 * 1024 * 1024})
fmt.Println("uploaded:", info.Size)
```

### 3. Server-side encryption (SSE-S3)

```go
opts := minio.PutObjectOptions{
    ServerSideEncryption: encrypt.NewSSE(),
}
cli.PutObject(ctx, "bucket", "secret.txt", reader, size, opts)
```

### 4. Reed-Solomon encoding (klauspost)

```go
import "github.com/klauspost/reedsolomon"

enc, _ := reedsolomon.New(8, 4)
shards, _ := enc.Split(data)
_ = enc.Encode(shards)

// Lose up to 4 shards; rebuild from remaining 8.
if err := enc.Reconstruct(shards); err != nil { /* unrecoverable */ }
```

`klauspost/reedsolomon` is the workhorse library — same one MinIO uses internally.

### 5. Streaming with sendfile (conceptual)

```go
// Sketch — MinIO does this internally
file, _ := os.Open(path)
conn := w.(http.ResponseWriter).(http.Hijacker).Hijack()
// On Linux, io.Copy from *os.File to *net.TCPConn triggers sendfile internally.
io.Copy(conn, file)
```

Go's stdlib detects compatible file/socket pairs and dispatches to `sendfile` automatically.

## Anti-Patterns & Gotchas

**Running MinIO on a single disk for production.** No redundancy; one disk failure = data loss. Use erasure-coded mode (4+ disks minimum).

**Mixing disk sizes.** MinIO assumes uniform disk size per pool. Smaller disks fill first.

**Using RAID under MinIO.** Double redundancy is wasteful; let MinIO erasure-code on JBOD.

**Skipping `xfs` for production.** `ext4` works but has bigger overhead on huge directories.

**Reaching for AWS S3 SDK with MinIO endpoints.** Works, but the official `minio-go` has better support for MinIO-specific features.

**Forgetting bucket versioning.** Without versioning, accidental delete is permanent. Enable on production buckets.

**Heavy small-file workloads.** Object storage assumes objects are "large" (MiB+). 1M tiny files are inefficient.

**Skipping ILM rules.** Without lifecycle, you'll pay for cold data forever.

**Treating MinIO as a substitute for S3 in cost.** It's free-as-in-software but costs you hardware + ops.

**Mixing AGPL-licensed MinIO into proprietary embedded products.** Requires source release. Read the license.

## Performance Notes

(MinIO-published benchmarks; vary by hardware.)

- Single-node read throughput (NVMe, 16 disks): 10 GiB/s.
- 32-node cluster read: ~325 GiB/s aggregate.
- Single-object PUT latency (small): 5–20 ms.
- Multipart upload (large): network-bound.
- Erasure-encode (8+4) on AVX2 CPU: ~10 GiB/s per core.
- Read sendfile bandwidth: 10 GiB/s+ per node.
- GC pauses tuned via `GOMEMLIMIT`: <1 ms typical.

## How Big Companies Use It

- **Bloomberg**: petabyte-scale archive on MinIO.
- **Pinterest**: documented MinIO use for ML pipelines.
- **NVIDIA**: AI training data on MinIO.
- **HKMA, Bank of America (rumored)**: regulated archive.
- **Many federal agencies**: air-gapped S3.
- **Major telcos**: HLS video archival.
- **Hyperscale cloud-edge** providers.

## Source Code References

- MinIO: [`minio/minio`](https://github.com/minio/minio).
- minio-go (Go client): [`minio/minio-go`](https://github.com/minio/minio-go).
- mc (CLI): [`minio/mc`](https://github.com/minio/mc).
- KES: [`minio/kes`](https://github.com/minio/kes).
- klauspost/reedsolomon: [`klauspost/reedsolomon`](https://github.com/klauspost/reedsolomon).
- klauspost/compress: [`klauspost/compress`](https://github.com/klauspost/compress).
- MinIO Console (UI + backend): [`minio/console`](https://github.com/minio/console).

(AGPLv3 for MinIO; Apache-2.0 for klauspost libs; varies.)

## Further Reading

- MinIO docs: https://min.io/docs/minio/linux/index.html.
- "MinIO Architecture": https://min.io/docs/minio/linux/operations/concepts/architecture.html.
- "Erasure coding": https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html.
- Klaus Post's blog (Reed-Solomon optimization): https://klauspost.io.
- "Scaling MinIO" — talks at various conferences.
- AB Periasamy talks at GopherCon, KubeCon.
- "AGPL vs Apache" — MinIO's licensing rationale: https://min.io/.
- "Direct I/O in Go" — community write-ups.

## Exercises / Self-Check

1. Stand up a 4-node MinIO cluster on Docker. Upload an object; manually corrupt one shard; verify the heal scanner reconstructs it.
2. Use `klauspost/reedsolomon` to encode and decode a 1 GiB random byte stream. Measure throughput; compare to your CPU's theoretical AVX2 max.
3. Why does MinIO use `O_DIRECT`? In a Go service that doesn't use it, what would be the symptom under sustained bulk reads?
4. Compare MinIO to AWS S3 for a workload of 10M 4 KiB files. Where does each become inefficient?
5. Implement a simple S3 client that uploads a file to MinIO with server-side encryption (SSE-S3) and verifies the data is encrypted at rest.
