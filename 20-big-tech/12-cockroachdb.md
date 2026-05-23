# CockroachDB — Distributed SQL Built in Go

## TL;DR

**CockroachDB** ("CRDB") is a globally-distributed SQL database — geographically replicated, ACID, Postgres-wire-compatible. It is one of the largest Go applications by line count (**~2M+ LOC**), and arguably the most ambitious Go-based database. Built by Spencer Kimball, Peter Mattis, and Ben Darnell — ex-Google engineers who built Colossus and shaped GFS — CRDB is essentially **Spanner in Go**, MIT-licensed (until 2019; now BSL like HashiCorp). The single biggest gotcha: **CockroachDB pushes Go to its limits on the database data plane** — they wrote their own columnar in-memory format (`coldata`), a custom Raft fork, a Pebble (LevelDB-style) storage engine, and aggressive sync.Pool patterns to make Go performant enough for OLTP. Several of their open-source projects (Pebble, Raft, Cockroach-CLI) are widely reused.

## Mental Model

```
   CockroachDB cluster (each node a single Go binary):
   
   ┌──────────────────────────────────────────────────────────┐
   │  Postgres-compatible wire protocol (SQL clients)          │
   ├──────────────────────────────────────────────────────────┤
   │  SQL layer:                                                │
   │   - parser (yacc-style, code-generated)                   │
   │   - planner (DistSQL: distributed query planner)          │
   │   - executor: row-based AND vectorized (coldata)          │
   ├──────────────────────────────────────────────────────────┤
   │  Distribution / KV layer:                                  │
   │   - splits data into "ranges" (~512 MiB each)             │
   │   - each range is a Raft group with 3+ replicas           │
   │   - leases for read optimization                           │
   ├──────────────────────────────────────────────────────────┤
   │  Storage engine:                                            │
   │   - Pebble (LSM tree, RocksDB-inspired, written in Go)      │
   │   - mmap'd manifest, sstable files                         │
   ├──────────────────────────────────────────────────────────┤
   │  Transport:                                                 │
   │   - gRPC between nodes                                      │
   │   - Raft messages, lease renewals, replication              │
   └──────────────────────────────────────────────────────────┘
```

Each node holds a slice of every range it's part of; queries are distributed by the SQL planner. ACID is achieved via Spanner-style multi-version concurrency control with hybrid logical clocks.

## Syntax & Basic Usage

CRDB is Postgres-wire-compatible; any Postgres client works:

```go
package main

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5"
)

func main() {
	ctx := context.Background()
	conn, err := pgx.Connect(ctx, "postgresql://root@localhost:26257/defaultdb?sslmode=disable")
	if err != nil { panic(err) }
	defer conn.Close(ctx)

	_, err = conn.Exec(ctx, "CREATE TABLE IF NOT EXISTS kv (k STRING PRIMARY KEY, v STRING)")
	if err != nil { panic(err) }

	_, err = conn.Exec(ctx, "INSERT INTO kv VALUES ('hello', 'world') ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v")
	if err != nil { panic(err) }

	var v string
	err = conn.QueryRow(ctx, "SELECT v FROM kv WHERE k=$1", "hello").Scan(&v)
	if err != nil { panic(err) }
	fmt.Println("v=", v)
}
```

Drop-in for Postgres. CockroachDB ships its own driver (`cockroach-go`) too, with retry helpers.

## Deep Dive

### Why Go (CockroachDB's reasoning)

The founders, in their 2014 announcement, explained:

- **Wanted a single static binary**.
- **Strong concurrency primitives** — every Raft group is many goroutines.
- **Productivity** — building Spanner-like semantics in C++ would have taken years more.
- **Acceptable performance** — modern Go GC was already adequate (this was 1.4-era).

CRDB's CTO Ben Darnell has spoken about the trade-off: "Go's performance is enough; we save engineering time. We've never seriously considered rewriting in C++."

### Pebble — the storage engine

[Pebble](https://github.com/cockroachdb/pebble) is CockroachDB's LSM-tree storage engine. Originally CRDB used **RocksDB** (C++) via cgo; the cgo overhead and GC interaction cost were significant.

In 2020 they finished Pebble — a pure-Go RocksDB-format-compatible engine. Pros:
- No cgo overhead.
- Tighter Go integration; better profiling.
- ~10% throughput improvement on OLTP.

Pebble is now used outside CRDB (Etcd has experimented; some homegrown DBs use it).

### Raft fork

CRDB doesn't use HashiCorp Raft directly; they use a heavily customized fork of etcd's Raft (`go.etcd.io/raft`). Reasons:
- They run hundreds of thousands of Raft groups per cluster (one per range). HashiCorp's Raft is single-group.
- They need batch-and-pipeline optimizations not in upstream.

The fork lives in [`cockroachdb/cockroach/pkg/raft`](https://github.com/cockroachdb/cockroach/tree/master/pkg/raft).

### Hundreds of thousands of Raft groups

A 1 TiB cluster has ~2000 ranges (512 MiB each) × 3 replicas = 6000 Raft logs. A 100 TiB cluster has 600k Raft logs. Each is its own state machine with leader, followers, log, snapshot.

This is unprecedented Raft scale. CRDB invested years optimizing:
- **Quotient cache** for Raft state to avoid full re-fetches.
- **Coalesced heartbeats** across many groups on the same nodes.
- **Concurrent log application** within a single group.

### coldata — Go's most aggressive columnar layout

[`pkg/col/coldata`](https://github.com/cockroachdb/cockroach/tree/master/pkg/col/coldata) is CockroachDB's vectorized execution engine. Conventional row-based SQL is slow per-row; columnar processes batches.

The trick in Go: layout columns as `[]int64`, `[]string`, `[]float64` — flat arrays, no pointer-bearing structs. Each column is a `[]byte` or fixed-array `[]T`. The GC barely scans these.

```go
type Vec struct {
    typ  *types.T
    col  []int64       // or []float64, []string, etc.
    nulls Nulls         // bitmap of NULLs
}

type Batch struct {
    cols []Vec
    n    int   // number of rows
}
```

A batch is 1024 rows. Operators process batches; the inner loop sees flat slices, gets full BCE, and incurs ~zero GC pressure.

DistSQL operators (filters, aggregates, joins) are written for both row and columnar mode. Vectorized mode is faster for analytical-style queries by 5–10×.

### sync.Pool everywhere

CRDB uses `sync.Pool` heavily for:
- KV key/value byte buffers.
- Protocol message structs.
- Iterator state.
- Batch scratch space.

The teams document `sync.Pool`'s GC interaction (pool clears on every GC) and tune accordingly: deeper pools for hot paths, explicit `Reset()` methods.

### DistSQL — distributed SQL execution

CRDB's planner produces a *physical plan* that spans multiple nodes. Each node runs a piece. Streams move data between operators.

```
Client → SQL gateway → planner → "flow" of operators across nodes:
   Node A: scan range_1, filter, send to Node B
   Node B: scan range_2, filter, merge with stream from A, project
   Node B → Client
```

Each operator is a `Processor` interface; flows are scheduled by node-local schedulers. Internally similar in structure to Apache Spark or Flink, but for SQL.

### Postgres wire-protocol compatibility

CRDB speaks Postgres's wire protocol. Every Postgres client (`psql`, `pgx`, `lib/pq`, JDBC, etc.) works without modification. There are SQL dialect differences but most apps "just work".

### Online schema changes

Adding a column to a 100 TiB table without downtime: CRDB does this via background work. Implementation is heavy — many edge cases. Their blog has multiple posts on the algorithm.

### Operations

`cockroach` binary is the same for server and CLI. `cockroach start`, `cockroach sql`, `cockroach init`. Cluster topology configured via flags.

### License

CRDB was Apache 2.0 until 2019, when they relicensed to **BSL (Business Source License)** with a 3-year delayed conversion to Apache. Set the precedent later followed by HashiCorp. The community has not forked CRDB the way Terraform was forked.

### Performance characteristics

CRDB is not a top-tier OLTP benchmark performer (Postgres, MySQL single-node beat it) — but it's competitive *given replication overhead*. Strengths:
- Strong consistency across DC.
- Survival of zone/region failures.
- Online operations (scale-out, schema change, version upgrade).

Throughput on a 3-node cluster: 50k–100k TPS on simple workloads.

### Things they've documented

- **GC pause investigation**: per-range Raft state created pointer-dense heaps.
- **JSON unmarshal hot path**: replaced with custom decoder for hot fields.
- **gRPC tuning**: `MaxConcurrentStreams`, custom flow control.
- **Pebble compaction GC interaction**: bursty allocations during major compactions.

### Test infrastructure

CRDB has one of the largest Go test suites: **half a million test cases**. They've built:
- `roachtest`: cloud-based integration tests.
- `roachprod`: cluster provisioning for tests.
- Property-based testing via custom DSLs.
- Jepsen testing for distributed-system safety (publicly conducted).

## Standard Library Hooks

- `database/sql`-style interfaces externally (clients).
- `crypto/tls`: client connections.
- `encoding/binary`: serialization.
- `sync.Pool`, `sync.RWMutex`, `sync/atomic`: hot paths.
- `runtime/pprof`: continuous profiling.
- `golang.org/x/sync/errgroup`: parallel work.
- `google.golang.org/grpc`: node-to-node.
- `bufio`: client connection I/O.

## Real-World Patterns

### 1. Embed Pebble as a KV store

```go
import "github.com/cockroachdb/pebble"

db, _ := pebble.Open("/tmp/db", &pebble.Options{})
defer db.Close()

_ = db.Set([]byte("key"), []byte("value"), pebble.Sync)

v, closer, _ := db.Get([]byte("key"))
fmt.Println(string(v))
closer.Close()
```

Pebble is standalone — usable as a Go embedded LSM-tree DB, similar to BoltDB but LSM-shaped.

### 2. Postgres-compatible client

```go
import "github.com/jackc/pgx/v5"
conn, _ := pgx.Connect(ctx, "postgresql://user@host:26257/db")
```

CRDB's docs recommend pgx. Their own `github.com/cockroachdb/cockroach-go/crdb` library adds automatic retry on serializable-restart errors.

### 3. Transactional retry

```go
import "github.com/cockroachdb/cockroach-go/v2/crdb/crdbpgx"

err := crdbpgx.ExecuteTx(ctx, conn, pgx.TxOptions{}, func(tx pgx.Tx) error {
    if _, err := tx.Exec(ctx, "UPDATE a SET v=v+1 WHERE k='x'"); err != nil { return err }
    if _, err := tx.Exec(ctx, "UPDATE b SET v=v-1 WHERE k='x'"); err != nil { return err }
    return nil
})
```

`ExecuteTx` automatically retries on serialization restart errors — a CRDB-specific concern.

### 4. Coldata-style vectorized operator (conceptual)

```go
type FilterOp struct {
    pred  func(v int64) bool
    input Iterator
}

func (f *FilterOp) Next() *Batch {
    batch := f.input.Next()
    out := 0
    for i := 0; i < batch.n; i++ {
        if f.pred(batch.cols[0].col[i]) {
            batch.cols[0].col[out] = batch.cols[0].col[i]
            out++
        }
    }
    batch.n = out
    return batch
}
```

Operates on entire batch; BCE friendly; near-zero GC.

### 5. Raft-style replication via etcd-io/raft

```go
import "go.etcd.io/raft/v3"

n := raft.StartNode(&raft.Config{
    ID: 1, ElectionTick: 10, HeartbeatTick: 1,
    Storage: raft.NewMemoryStorage(),
}, []raft.Peer{{ID: 1}, {ID: 2}, {ID: 3}})
defer n.Stop()

// Propose a value to the cluster:
_ = n.Propose(ctx, []byte("hello"))
```

CRDB's fork extends this for hundreds-of-thousands-of-groups scale; for normal use, etcd's upstream Raft is fine.

## Anti-Patterns & Gotchas

**Treating CRDB as drop-in Postgres.** Strong consistency + serializable isolation default means more retries than Postgres at the same workload.

**Doing 1-row inserts in a tight loop.** Round-trip latency dominates. Use batched inserts or COPY.

**Joining across many ranges with WHERE on non-indexed columns.** DistSQL scatters reads; latency adds. Add indexes.

**Trusting `pgx`'s defaults.** Connection pool sizing, statement caching, TLS — all need tuning.

**Skipping retry helpers.** Serializable transactions can be aborted; the client must retry. `crdbpgx.ExecuteTx` does this.

**Running 2-node CRDB.** Raft needs majority; 2 means 1-failure = unavailable. Use 3+ nodes.

**Using Pebble for tiny embedded apps.** It's LSM-tree shaped — good for write-heavy workloads, overkill for simple K/V. BoltDB / Badger may be simpler.

**Hand-rolling vectorized SQL evaluation.** Use a query engine if you can; CRDB's coldata is years of work.

**Forgetting `Closer.Close()` on Pebble Gets.** Each `Get` returns an io.Closer; not closing leaks memory.

**Treating CRDB as cheap. It's resource-hungry; 3-node minimum, lots of disk, lots of RAM.**

## Performance Notes

(From CRDB documentation and benchmarks.)

- 3-node simple OLTP: 50–100k TPS.
- Read p99: 1–5 ms intra-DC.
- Write p99: 10–50 ms (Raft commit + fsync).
- Across-region p99: ~hundreds of ms (speed of light dominated).
- Pebble write throughput: ~100 MiB/s sustained.
- Vectorized scan: 5–10× row-based.

## How Big Companies Use It

- **Shipt**, **Bose**, **Comcast**, **Lush**: CockroachDB Cloud customers.
- **Several large banks**: as a regulated, ACID, multi-region store.
- **CRDB-style architectures** influenced YugabyteDB, FaunaDB, Spanner clones.

Pebble alone has spread:
- **CockroachDB**: original use.
- **Etcd**: experimental backend.
- **Several homegrown event stores**.

## Source Code References

CockroachDB main repo: [`cockroachdb/cockroach`](https://github.com/cockroachdb/cockroach).

- Pebble: [`cockroachdb/pebble`](https://github.com/cockroachdb/pebble).
- coldata (vectorized): [`cockroach/pkg/col/coldata`](https://github.com/cockroachdb/cockroach/tree/master/pkg/col/coldata).
- DistSQL: [`cockroach/pkg/sql/distsql`](https://github.com/cockroachdb/cockroach/tree/master/pkg/sql/execinfra).
- Raft fork: [`cockroach/pkg/raft`](https://github.com/cockroachdb/cockroach/tree/master/pkg/raft).
- KV layer: [`cockroach/pkg/kv`](https://github.com/cockroachdb/cockroach/tree/master/pkg/kv).
- SQL parser (Yacc-generated): [`cockroach/pkg/sql/parser`](https://github.com/cockroachdb/cockroach/tree/master/pkg/sql/parser).
- cockroach-go (driver helpers): [`cockroachdb/cockroach-go`](https://github.com/cockroachdb/cockroach-go).
- roachtest, roachprod (test infra): [`cockroach/pkg/cmd/roachtest`](https://github.com/cockroachdb/cockroach/tree/master/pkg/cmd/roachtest).
- etcd-io/raft (upstream): [`etcd-io/raft`](https://github.com/etcd-io/raft).

(BSL since 2019 for CRDB; Apache-2.0 for Pebble.)

## Further Reading

- "Why we built CockroachDB" — Spencer Kimball: https://www.cockroachlabs.com/blog/why-we-built-cockroachdb/.
- "Why Go?" — CockroachDB blog: https://www.cockroachlabs.com/blog/.
- "Pebble: a new RocksDB inspired key-value store" — https://www.cockroachlabs.com/blog/pebble-rocksdb-kv-store/.
- "Vectorized SQL execution" — https://www.cockroachlabs.com/blog/vectorized-execution-engine/.
- "Living with millions of Raft groups": engineering blog post.
- "Online schema change" series: https://www.cockroachlabs.com/blog/.
- Jepsen reports on CockroachDB: https://jepsen.io/analyses.
- "Spanner: Google's globally distributed database" — OSDI 2012.
- Ben Darnell on Distributed Systems: GopherCon talks.

## Exercises / Self-Check

1. Embed Pebble in a Go program. Compare its write throughput to BoltDB and Badger for sequential and random writes.
2. Build a tiny vectorized filter that operates on `[]int64`. Benchmark against a row-based equivalent. Where does the difference come from?
3. Why does CockroachDB use a fork of etcd's Raft rather than HashiCorp's? Identify two CRDB-specific Raft optimizations.
4. Write a client that retries CockroachDB transactions on serialization restart. What error codes signal "retry"?
5. CRDB targets thousands of Raft groups per node. What goroutine and memory cost would that imply with a naive implementation? What optimizations help?
