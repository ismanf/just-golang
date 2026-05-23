# InfluxDB — Time-Series Database in Go (and the Rust pivot)

## TL;DR

**InfluxDB** is one of the longest-running Go databases — InfluxDB v1 (2013) and v2 (2019) were Go from the start, optimized for high-rate time-series ingestion. Their **TSM** (Time-Structured Merge) storage engine pioneered Go-side LSM-tree implementations for time series. In 2020, InfluxData announced **InfluxDB IOx** (later renamed back to InfluxDB 3.0), a **Rust rewrite using Apache Arrow + Parquet + DataFusion**. The reasons mirror Discord's: GC pauses, lack of zero-copy, and the desire for SIMD-heavy vectorized execution. The single biggest gotcha: **InfluxDB v1/v2 (Go) and InfluxDB 3.0 (Rust) coexist in production**. Go didn't fail at time-series — it scaled to many years of production use. The Rust rewrite is about a different tier of analytical workloads (Parquet-on-disk columnar, not LSM row-store).

## Mental Model

```
   InfluxDB v1/v2 (Go) — focused on operational metrics:
   
   ┌──────────────────────────────────────────────────────────┐
   │  HTTP write API: line protocol                            │
   │   measurement,tag=value field=42 1721481600000000000      │
   └──────────────┬───────────────────────────────────────────┘
                  ▼
   ┌──────────────────────────────────────────────────────────┐
   │  TSM (Time-Structured Merge) engine                       │
   │   - WAL (Write-Ahead Log) on disk                         │
   │   - in-memory cache (recent writes)                       │
   │   - compacted .tsm files: encoded blocks per series       │
   │   - TSI (Time-Series Index): inverted index for tags      │
   └──────────────────────────────────────────────────────────┘
                  ▲
                  │
   ┌──────────────────────────────────────────────────────────┐
   │  Query layer (InfluxQL, Flux)                             │
   │   - planner expands queries to series-level reads          │
   │   - aggregation in Go                                       │
   └──────────────────────────────────────────────────────────┘
   
   InfluxDB 3.0 (Rust) — analytical, Parquet-based:
   
   ┌──────────────────────────────────────────────────────────┐
   │  Apache Arrow in-memory columnar                          │
   │  Parquet on disk (object storage)                          │
   │  DataFusion SQL engine                                    │
   │  No GC; SIMD throughout                                    │
   └──────────────────────────────────────────────────────────┘
```

The Go version remains supported and widely deployed. New work is on 3.0.

## Syntax & Basic Usage

Write to InfluxDB v2:

```go
package main

import (
	"context"

	influxdb2 "github.com/influxdata/influxdb-client-go/v2"
	"github.com/influxdata/influxdb-client-go/v2/api/write"
)

func main() {
	client := influxdb2.NewClient("http://localhost:8086", "my-token")
	defer client.Close()

	writeAPI := client.WriteAPIBlocking("my-org", "my-bucket")

	p := write.NewPoint("temperature",
		map[string]string{"sensor": "S1", "room": "A"},
		map[string]any{"value": 22.5},
		nil,
	)
	_ = writeAPI.WritePoint(context.Background(), p)
}
```

The line protocol format is `measurement,tag=value field=number timestamp`. Simple, well-suited to high-rate metric ingestion.

## Deep Dive

### History

**2013**: InfluxDB v1 launched. Initial storage was BoltDB; quickly outgrew it.
**2015**: TSM (Time-Structured Merge) engine introduced, custom-built in Go.
**2017**: TSI (Time-Series Index) for tag cardinality scaling.
**2019**: InfluxDB 2.0 — incorporates Flux (dataflow query language), Telegraf, Chronograf.
**2020**: IOx project announced — Rust rewrite using Arrow.
**2023**: InfluxDB 3.0 released as IOx-based product.

### TSM engine

[TSM](https://github.com/influxdata/influxdb/tree/master/tsdb/engine/tsm1) is an LSM-tree variant tuned for time series:

- Each series (a `measurement,tag1=value1,...` combo) has its own column.
- Data points are encoded with delta + gorilla-style float compression.
- Writes go to WAL + in-memory cache.
- Background compaction merges into `.tsm` files.

Key Go optimizations:
- `sync.Pool` for compaction buffers.
- Custom int + float encoders (delta, run-length, ZSTD).
- Aggressive pre-sizing of slices.

### TSI — Time-Series Index

InfluxDB's "high cardinality" problem: with N tag-value combinations, the in-memory index size grows linearly. TSI moves the index to disk in a tree-of-bitmaps format, allowing queries over millions of unique series with bounded memory.

Implementation: roaring bitmaps for tag-value sets; FST (Finite State Transducers) for prefix-search of tag keys. Both ported to Go.

### Flux — dataflow query language

[Flux](https://github.com/influxdata/flux) is InfluxData's pipeline language. Pipes-and-filters; column-oriented:

```flux
from(bucket: "telegraf")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> mean()
```

Compiles to a DAG of operators, executed in Go. Heavy use of generics-pre-1.18 (interfaces + type switches; 1.18+ migrated some to generics).

### Telegraf

[Telegraf](https://github.com/influxdata/telegraf) is InfluxData's metrics collection agent. Plugin-based, ~200+ input plugins, ~50+ output plugins. Used widely outside InfluxDB — Telegraf can write to Prometheus, Kafka, anything.

Single binary; deploys on every host; runs lightweight Go plugins.

### Why Go (initially)

Paul Dix (founder/CTO) wrote in early posts:

- **Single binary deployment**: customers run InfluxDB on bare metal, VMs, containers.
- **Reasonable performance for write-heavy workloads**.
- **Strong stdlib for HTTP and concurrency**.
- **Hiring**: Go developers were easier to find than C++.

### Why Rust (the IOx move)

Paul Dix's 2020 post "InfluxDB IOx: Rust + Arrow" lists:

- **Apache Arrow ecosystem**: columnar, zero-copy, SIMD-native. Go's ecosystem here is thinner.
- **GC pauses on huge heaps**: time-series with high cardinality created pointer-dense state.
- **DataFusion**: Arrow-native SQL engine in Rust.
- **Parquet**: native Rust support; Go's Parquet libs are less mature.
- **Tokio's async I/O**: predictable.

The post is balanced — Dix doesn't say "Go was bad". He says "Arrow's center of gravity is Rust; if we want SIMD-vectorized SQL, that's where the work is".

### What InfluxDB Go projects taught the ecosystem

#### Gorilla compression (port from Facebook's paper)

Time-series-specific encoding: delta-of-delta for timestamps, XOR for floats. Implemented in Go in TSM. Widely studied; appears in other Go TSDBs (Prometheus, M3DB).

#### LSM tuning at write-heavy scale

TSM's compaction-throttling logic became reference for other Go LSM-style engines.

#### High-cardinality index design

TSI's roaring bitmaps + FST hybrid set patterns followed by Pebble, Cortex, others.

### Concurrency patterns

InfluxDB v1/v2 was a sustained Go-concurrency tutorial in itself:
- Each shard had its own goroutines for write, query, compaction.
- Backpressure via bounded channels.
- Per-CPU GOMAXPROCS-aware sharding.

### Performance characteristics

InfluxDB v2 OSS (single-node):
- Sustained write: ~500k points/sec.
- Query latency: ms-level for recent data, seconds for old.
- Memory per series: ~150 bytes.
- 10M unique series before performance degrades (single-node OSS).

InfluxDB Enterprise (multi-node):
- 10M+ writes/sec.
- 100M+ unique series.

These are real numbers from running deployments.

### Where Go still wins (in InfluxData's stack)

- **Telegraf**: agent on every host. Single static binary. Go.
- **InfluxDB v1/v2**: still in production for millions of users.
- **CLI tooling**: `influx`, `influxd`. Go.
- **Plugins to other systems**: Kafka adapters, etc.

The Go portfolio is mature and supported. The Rust portion is for new analytical workloads.

### Lessons

1. **Go is good for write-heavy time-series.** TSM proves this.
2. **For analytical SQL on columnar data, Rust+Arrow is currently easier.** Until Go's Arrow ecosystem matures.
3. **Migrations cost years.** IOx was announced 2020, released 2023.
4. **The "Go vs Rust" framing is workload-specific.** Don't generalize.

## Standard Library Hooks

- `net/http`: APIs.
- `encoding/binary`, `encoding/json`: serialization.
- `sort`: per-series ordering.
- `sync.Pool`: hot path buffers.
- `compress/snappy` (third-party), `klauspost/compress`: data compression.
- `bufio`, `io`: WAL handling.
- `os`, `syscall`: file system for TSM files.
- `runtime/pprof`: continuous profiling.

## Real-World Patterns

### 1. InfluxDB Go client (v2 SDK)

```go
client := influxdb2.NewClient(url, token)
defer client.Close()

writeAPI := client.WriteAPI("org", "bucket")
defer writeAPI.Flush()

for _, m := range metrics {
    p := write.NewPoint(m.Name, m.Tags, m.Fields, time.Now())
    writeAPI.WritePoint(p)
}
```

Async batch writer; tunable batch size, flush interval.

### 2. Telegraf-style input plugin

```go
type CPUInput struct{}

func (c *CPUInput) Gather(acc telegraf.Accumulator) error {
    pct, _ := cpu.Percent(0, false)
    acc.AddFields("cpu", map[string]any{"percent": pct[0]}, nil)
    return nil
}
```

Built-in plugin framework; metrics flow from plugins to outputs.

### 3. Flux query

```flux
from(bucket: "metrics")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "http_requests")
  |> aggregateWindow(every: 30s, fn: count)
```

Pipeline expressed declaratively; executed by Flux runtime.

### 4. Embedded TSDB use

```go
import "github.com/influxdata/influxdb/tsdb/engine/tsm1"

// Note: not a stable embed API; this is illustrative.
// In practice, InfluxDB is run as a separate server.
```

Unlike Pebble/Badger, TSM isn't a fully separable embedded engine. Run InfluxDB as a service.

### 5. Migration from v2 to 3.0

Both speak SQL/Flux. Migration: dual-write for a period, then cut over reads. Tooling for export/import provided by InfluxData.

## Anti-Patterns & Gotchas

**Million-cardinality tags.** A tag like `request_id` (millions of unique values) blows up the index. Use fields or aggregation.

**Storing massive payloads as field values.** Time-series databases optimize for many small datapoints, not few big ones.

**Treating InfluxDB as a relational DB.** It's not; joins are limited.

**Naive write batching.** Use the client's batch writer; per-point HTTP POST is far too slow.

**Skipping retention policies.** Without retention, disks fill. Configure RP per bucket.

**Running v1.x in 2026.** Out of active development. Plan upgrades.

**Using InfluxQL in v2.** Flux is the future; v2 still supports InfluxQL via translation but new features are Flux-only.

**Comparing v2 Go with 3.0 Rust head-to-head.** Different storage models; different optimization targets.

## Performance Notes

(InfluxDB v2 OSS, single node, modest hardware.)

- Sustained writes: ~500k points/sec.
- Compaction throughput: ~100 MiB/s.
- Series cardinality before degradation: ~10M.
- Query latency (recent): ~ms.
- Memory: a few GiB to tens of GiB depending on cardinality.

## How Big Companies Use It

- **Tesla**: documented at GopherCon as an InfluxDB user for vehicle telemetry.
- **Cisco**: networking telemetry.
- **Atlassian, GitLab, Splunk**: dev metrics.
- **Ferrari, McLaren**: car-racing telemetry.
- **NASA, CERN**: scientific data streams.
- **Many gaming companies**: player + server metrics.

Telegraf is even more widely deployed than InfluxDB — many companies use Telegraf to ship to Prometheus, Datadog, etc.

## Source Code References

- InfluxDB v2: [`influxdata/influxdb`](https://github.com/influxdata/influxdb).
- TSM engine: [`influxdata/influxdb/tsdb/engine/tsm1`](https://github.com/influxdata/influxdb/tree/master/tsdb/engine/tsm1).
- Flux: [`influxdata/flux`](https://github.com/influxdata/flux).
- Telegraf: [`influxdata/telegraf`](https://github.com/influxdata/telegraf).
- Go client (v2): [`influxdata/influxdb-client-go`](https://github.com/influxdata/influxdb-client-go).
- InfluxDB IOx (Rust): [`influxdata/influxdb_iox`](https://github.com/influxdata/influxdb_iox) — for comparison.
- Apache Arrow Go: [`apache/arrow-go`](https://github.com/apache/arrow-go).

(MIT License, mostly.)

## Further Reading

- "InfluxDB IOx: Rust + Arrow" — Paul Dix: https://www.influxdata.com/blog/announcing-influxdb-iox/.
- "Why InfluxDB chose Rust": https://www.influxdata.com/blog/.
- Original TSM engine post: https://www.influxdata.com/blog/.
- Gorilla compression paper (Facebook): https://www.vldb.org/pvldb/vol8/p1816-teller.pdf.
- "Apache Arrow Go" documentation: https://arrow.apache.org/docs/go/.
- "Flux: Querying Time-Series Data" — InfluxData blog.
- Paul Dix talks at GopherCon: various years.
- "Comparing TSDBs": Prometheus vs InfluxDB vs others, multiple blog posts.

## Exercises / Self-Check

1. Why is high-cardinality tag explosion a problem for time-series DBs? Sketch the storage layout that makes it costly.
2. Implement Gorilla-style XOR compression for `[]float64` in Go. Compare size to raw bytes.
3. Read the InfluxDB IOx announcement. What specific Go-related issues do they cite, and which apply to your workloads?
4. Use the InfluxDB Go client to write 1M points. Tune the batch writer. Where's the bottleneck?
5. Compare InfluxDB's TSM to Prometheus's TSDB: how do they differ in storage layout, indexing, and tradeoffs?
