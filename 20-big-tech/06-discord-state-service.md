# Discord — The State Service, the GC Saga, and the Move to Rust

## TL;DR

Discord's **"Read States" service** (which tracks the unread/read status of every channel for every user) was originally Go. In **2020 they migrated it to Rust** and published "Why Discord is switching from Go to Rust" — one of the most-read Go performance post-mortems ever. The reason wasn't Go itself; it was the **GC behavior on a pointer-dense in-memory cache** holding 100M+ items: garbage collection scanned the entire heap on every cycle, and tail-latency spikes hit every two minutes (the GC's forced periodic collection). The Rust rewrite eliminated GC; latency dropped from p99 ~125 ms to p99 <2 ms. The single biggest gotcha (and the one most readers miss): **Discord still uses Go heavily elsewhere**. The Rust move was for one specific service whose workload didn't fit Go's GC; many Discord backend services remain Go.

## Mental Model

```
   Read States service (pre-Rust, 2018–2020):
   
   ┌──────────────────────────────────────────────────────────┐
   │  In-memory LRU cache (in Go)                              │
   │   - 100M+ entries                                          │
   │   - each entry: ~150 bytes                                 │
   │   - aggregate heap: ~15 GiB live                           │
   │   - heavily pointer-bearing (struct fields with strings,   │
   │     map values, time.Time)                                 │
   ├──────────────────────────────────────────────────────────┤
   │  Every 2 minutes (the GC's forced periodic collection):    │
   │   - GC scans the ENTIRE heap                                │
   │   - mark phase: ~125 ms latency spike                      │
   │   - even with GOGC=off, runtime forces collection every    │
   │     2 minutes (runtime/proc.go: forcegcperiod)             │
   └──────────────────────────────────────────────────────────┘

   Rust rewrite (2020+):
       - same workload, no GC
       - p99 ~2 ms steady
       - half the memory (Rust's tighter layout)
       - all the latency spikes gone
```

The post-mortem became cited shorthand for "Go's GC has scale limits on pointer-heavy in-memory caches". The technical details are nuanced; the headline reading isn't always accurate.

## Syntax & Basic Usage

A Go cache like Discord's Read States (simplified):

```go
package main

import (
	"sync"
	"time"
)

type ReadState struct {
	ChannelID string
	UserID    string
	LastSeen  time.Time
	Mentions  int
}

type Cache struct {
	mu sync.RWMutex
	m  map[string]*ReadState // 100M+ entries — pointer-heavy
}

func (c *Cache) Get(key string) *ReadState {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.m[key]
}

func (c *Cache) Set(key string, rs *ReadState) {
	c.mu.Lock()
	c.m[key] = rs
	c.mu.Unlock()
}
```

Every `*ReadState` is a pointer; `ChannelID`, `UserID` are strings (also pointers under the hood); `time.Time` carries pointers (location). The GC has to walk all of this on every mark cycle.

## Deep Dive

### The actual problem

Discord's Read States service answers: "for user U, in channel C, what's the latest message you've seen?" There's one entry per (user, channel). With 100M+ users and many channels each, the cardinality reaches billions in theory; in practice the working set is "active users × their channels", say 100M+ entries kept hot.

The architecture:
- Service reads from Cassandra on cache miss.
- Service writes back when users open channels.
- Cache must be fast (sub-ms) because every message read updates state.
- Service eventually flushes to Cassandra.

Memory layout:

```go
type ReadState struct {
    ID         string     // 16 bytes header + heap allocation
    LastMsgID  string     // ditto
    LastSeenAt time.Time  // 24 bytes, includes *time.Location pointer
    MentionCount int      // 8 bytes
    // ... more fields
}
```

A `string` in Go is a `(ptr, len)` pair (16 bytes) plus the heap-allocated byte array. `time.Time` has 24 bytes and a `*Location` pointer. Each `*ReadState` in the map is itself a pointer; the map's bucket points to it; the map's bucket array has pointer-bearing fields.

Net effect: 100M entries × 5-10 pointer-bearing fields each = ~1 billion pointers the GC must scan on every mark cycle.

### The 2-minute periodic GC

Even with `GOGC=off`, Go's runtime forces a GC every 2 minutes (`forcegcperiod` in `runtime/proc.go`):

```go
// runtime/proc.go (excerpt; BSD-3 © The Go Authors)
const forcegcperiod = 2 * 60 * 1e9 // 2 minutes in nanoseconds

// In sysmon:
if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
    // force a GC
}
```

Discord's pre-Rust deployment had set `GOGC=off` to avoid GC during normal operation — but the periodic forced GC still hit every 2 minutes, producing the regular latency spikes they observed.

### Why GC at 15 GiB scan size hurt so much

Pre-Go-1.5 GC was stop-the-world. Post-1.5 it's mostly concurrent, but:
- Mark phase still consumes ~25% of one CPU.
- Mark *work* scales with the *live, scan-bearing* heap.
- Pointer-dense heaps make mark expensive.

For 15 GiB of pointer-bearing data:
- Mark CPU: significant (~25% of cores while running).
- Mark assist (when allocations outpace background mark): adds latency.
- STW phases (mark-setup + mark-term): typically <1 ms but on a 1B-pointer heap they can spike.

The latency spike Discord saw (p99 ~125 ms) wasn't STW per se — it was the combination of mark CPU, GC assist, and concurrent contention on the mutexes their cache used. The post is honest about the multifactor cause.

### What Discord tried before Rust

Per the post:

1. **`GOGC=off`**: stopped auto-GC. Eliminated minor pauses; the 2-minute forced GC remained.
2. **`debug.SetGCPercent(800)`**: reduced frequency. Helped, didn't eliminate.
3. **Caching only frequently-accessed entries**: reduced cardinality. Helped, but the working set was inherently large.
4. **Off-heap caches** (custom byte-slice-backed structures): didn't fit their data model.
5. **Manual sharding** (multiple Go processes each holding a piece): didn't help — each process still had millions of entries.

None gave the consistent latency they wanted.

### Why Rust

Rust offers:
- No GC; deterministic destruction via ownership.
- Tighter memory layout (no per-`struct` runtime metadata).
- Same compiled-binary deployment.
- Acceptable productivity for this team.

The migration took several months. The Rust version uses similar data structures (LRU cache, mutex-shared map) but no garbage collector.

### What the Discord post does NOT say

The post is **specific to this one service**. It does not say:
- "Don't use Go."
- "Go's GC is broken."
- "All caches should be Rust."

Discord engineers still write Go for many services. The post identifies a workload — large, pointer-heavy in-memory cache with strict tail latency — that doesn't fit Go's GC.

### What the post DOES say (worth quoting)

> "We didn't choose Rust over Go because Go is bad. We just hit a specific class of workload where Rust's lack of GC made the engineering tradeoff easier."

> "If we hadn't been running such a pointer-heavy cache in process, Go would have been fine."

### Modern Go improvements that *would have helped*

The post was written for Go 1.13–1.14 era. Modern Go has materially better GC:

- **Go 1.19+ `GOMEMLIMIT`**: prevents OOM spirals.
- **Go 1.22+ per-type GC bitmaps**: faster mark.
- **Go 1.25+ Green Tea GC**: region-aware marking; better locality on large heaps.
- **Per-P timers since 1.14**: better tail latency in general.

It's likely (but not certain) that Discord's Read States service would have been viable in Go 1.25+. The team has not publicly revisited the decision.

### The pattern: noscan-friendly layout

What Discord would have wanted (in Go) is a **noscan-friendly layout** — data the GC doesn't have to scan. The fix in Go:

```go
// Bad: pointer-dense
type RowPtr struct {
    Key  *string
    Vals *[]int64
}

// Good: noscan
type RowFlat struct {
    Key  [32]byte
    Vals [16]int64
}
```

A noscan span (no pointer-bearing fields) is invisible to the GC's mark phase. 15 GiB of noscan data = near-zero GC scan cost.

But it requires:
- Fixed-size key/value layouts (no `string`, no `time.Time`, no maps).
- Manual offset arithmetic for variable-length fields.
- Byte-array arenas for buffer-backed strings.

This is the pattern CockroachDB's `coldata`, Dgraph's posting lists, and Badger's value log all use. It works; it's effort.

### Lessons for Go developers

#### 1. Large in-memory caches need noscan-friendly layout

If you have hundreds of GB of cache data, design upfront so the GC doesn't scan it. Use `[]byte` arenas, fixed-size structs, integer keys.

#### 2. Profile GC time, not just CPU

`runtime/metrics` `/gc/cpu/time:cpu-seconds` and `/gc/pauses:seconds` show what GC is doing. If GC CPU >20%, your heap shape is suspect.

#### 3. Latency-critical workloads ≠ throughput-critical workloads

Go's GC is *throughput-optimized*. A workload requiring p99 <5 ms on a 15 GiB heap is at the edge of what's achievable.

#### 4. Sometimes the right answer is another language

Engineering judgment: if the rewrite cost is reasonable and the gains are large, switch. Discord did. Most teams don't need to.

### What Discord uses Go for today

(As of public 2024 information.)

- API gateway and Worker services.
- Most user-facing backend services.
- Voice infrastructure (some).
- Internal tooling.

Rust is reserved for high-throughput, latency-critical paths (Read States, certain media services).

## Standard Library Hooks

Tools to investigate GC behavior like Discord's:

- `GODEBUG=gctrace=1` — per-cycle log.
- `GODEBUG=gcpacertrace=1` — pacer internals.
- `runtime/metrics`:
  - `/gc/heap/live:bytes` — live heap.
  - `/gc/scan/heap:bytes` — scanned per cycle.
  - `/gc/pauses:seconds` — pause histogram.
  - `/gc/cpu/total:cpu-seconds` — cumulative GC CPU.
- `runtime.ReadMemStats`: comprehensive snapshot (STW cost).
- `runtime/debug.SetGCPercent`, `SetMemoryLimit`.
- `runtime/pprof` heap profile to find allocation sites.

## Real-World Patterns

### 1. Noscan-friendly cache entry

```go
package cache

// Bad: every entry has pointers; GC visits each.
type EntryPtr struct {
    Key, Val string
    Tags     []string
    Updated  time.Time
}

// Good: fixed bytes, no pointers; noscan span.
type EntryFlat struct {
    Key      [32]byte
    Val      [128]byte
    TagCount uint8
    Tags     [8][16]byte
    Updated  int64 // unix nanos
}
```

100M `EntryFlat` records: ~16 GiB but **0 pointers for the GC to scan**. Mark cost: near-zero.

### 2. Byte-arena for variable-length strings

```go
type Arena struct {
    buf []byte
}

func (a *Arena) Append(s string) (offset, length uint32) {
    o := uint32(len(a.buf))
    a.buf = append(a.buf, s...)
    return o, uint32(len(s))
}

type Entry struct {
    KeyOffset, KeyLength uint32
    ValOffset, ValLength uint32
}

// Lookup: a.buf[entry.KeyOffset : entry.KeyOffset+entry.KeyLength]
```

`[]byte` is a single allocation with no internal pointers (its bytes are not pointers). All your string-like data lives there; entries hold offsets.

### 3. Force GC to characterize behavior

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	debug.SetGCPercent(-1) // disable auto-GC

	// Populate 10M entries
	m := make(map[int]*entry)
	for i := 0; i < 10_000_000; i++ {
		m[i] = &entry{x: i, y: i + 1}
	}

	start := time.Now()
	runtime.GC()
	fmt.Println("forced GC took:", time.Since(start))

	var s runtime.MemStats
	runtime.ReadMemStats(&s)
	fmt.Println("scanned bytes:", s.GCSys+s.HeapAlloc)
}

type entry struct{ x, y int }
```

Useful diagnostic: how long does a full mark take at YOUR heap size?

### 4. Shard the cache across goroutines

```go
const shards = 256

type ShardedCache struct {
    parts [shards]struct {
        mu sync.RWMutex
        m  map[string]*ReadState
    }
}

func (c *ShardedCache) shard(key string) *struct {
    mu sync.RWMutex
    m  map[string]*ReadState
} {
    // hash key, take shard
    return &c.parts[hash(key)%shards]
}
```

Sharding doesn't reduce GC scan time (the heap is the same), but it reduces lock contention. Helps with one symptom Discord saw.

### 5. Monitor with `runtime/metrics`

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func report() {
	samples := []metrics.Sample{
		{Name: "/gc/heap/live:bytes"},
		{Name: "/gc/scan/heap:bytes"},
		{Name: "/gc/pauses:seconds"},
		{Name: "/gc/cpu/total:cpu-seconds"},
	}
	metrics.Read(samples)
	for _, s := range samples {
		fmt.Println(s.Name, s.Value.Kind(), s.Value)
	}
}
```

No-STW, fine-grained.

## Anti-Patterns & Gotchas

**Reading "Why Discord is switching" and panicking.** Most workloads don't need Rust. Profile YOUR app first.

**Assuming GC is your problem because tail latency is bad.** It might be locks, channels, or external systems. Use `runtime/trace` to disambiguate.

**Adding more memory hoping GC behaves better.** Often makes it worse — more heap to scan, longer marks.

**Setting `GOGC=off` "to save GC CPU".** Not a fix. The forced 2-minute collection still runs; memory grows unbounded otherwise.

**Trying to layout-tune `string`-heavy data.** Strings always have pointers. Use `[]byte` arenas.

**Using `interface{}` / `any` as map values for "flexibility".** Each value is an interface header + heap alloc; pointer-dense by construction.

**Pinning entire query result objects in a cache.** Cache the minimum bytes you need; let the rest GC.

**Skipping benchmarks because "Go GC is fine".** At <10 GiB heap, yes. At 100 GiB pointer-dense, no.

**Treating "Rust vs Go" as a generalized question.** It's workload-specific.

**Assuming Discord's exact problem applies to your service.** Their cache had specific cardinality, latency, and access patterns. Yours probably doesn't.

## Performance Notes

Per Discord's post (Go 1.13–1.14 era):

- 15 GiB live heap, pointer-dense.
- p50 latency: <1 ms (cache hit).
- p99 latency: ~125 ms (during GC mark / assist).
- GC frequency: every ~2 min (forced periodic).
- GC mark CPU: ~30% of one core during cycles.

After Rust migration:

- p50: <0.5 ms.
- p99: <2 ms.
- Memory usage: ~7 GiB (Rust's tighter layout).
- No GC.

Modern Go 1.25+ estimates (extrapolated, not from Discord):

- p99 likely 20–40 ms on the same heap shape.
- Green Tea GC + per-type bitmaps: ~30% lower mark CPU.
- Still not as predictable as Rust at this scale.

## How Big Companies Use It

The "Go-to-Rust for caches" pattern is followed by:

- **Discord** (Read States, voice processing): https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **Cloudflare** (Pingora proxy, but for different reasons): https://blog.cloudflare.com/pingora-open-source/.
- **Figma** (multiplayer state — also Rust): https://www.figma.com/blog/.
- **Dropbox** (compression — kept in Go after evaluation, see `20-big-tech/05-dropbox-magic-pocket.md`).
- **CockroachDB** (kept in Go but built `coldata` for noscan layout).
- **Tigris** (Rust storage engine, Go API layer): https://tigris.dev.

Most "we moved from Go to Rust" stories cite GC, not language complaints.

## Source Code References

Discord's services aren't open source. Related Go/Rust constructs:

- `runtime/proc.go` `forcegcperiod`: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go).
- GC pacer: [`src/runtime/mgcpacer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcpacer.go).
- Green Tea GC (1.25+): [`src/runtime/mgcmark_greenteagc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcmark_greenteagc.go).
- A reference noscan-cache pattern: `klauspost/bigcache` or `dgraph-io/ristretto`.
- `dgraph-io/ristretto`: [`github.com/dgraph-io/ristretto`](https://github.com/dgraph-io/ristretto) — Go cache with noscan-friendly layout.
- `allegro/bigcache`: [`github.com/allegro/bigcache`](https://github.com/allegro/bigcache) — `[]byte` arena cache for GC-friendliness.

## Further Reading

- "Why Discord is switching from Go to Rust" (the canonical post): https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- "Storing billions of messages: how Discord uses Cassandra and ScyllaDB": https://discord.com/blog/.
- "Using Rust to scale Elixir for 11 million concurrent users" (Discord's voice infra): https://discord.com/blog/.
- "Go GC: Solving the Latency Problem" (Twitch, similar problem space): https://blog.twitch.tv/en/2016/07/05/.
- "A Guide to the Go Garbage Collector" (official): https://go.dev/doc/gc-guide.
- "Memory layout for low-GC overhead" — Damian Gryski talks.
- Bigcache design: https://blog.allegro.tech/2016/03/writing-fast-cache-service-in-go.html.
- "Garbage Collection in Go: Part III" — Bill Kennedy: https://www.ardanlabs.com/blog/2019/05/garbage-collection-in-go-part3-gcpacing.html.

## Exercises / Self-Check

1. Build a Go program that allocates 1B `*int` in a map. Measure GC pause time. Now replace with `[1_000_000_000]int` flat array. Compare.
2. Read the Discord post and identify the specific Go version they were on. What GC improvements have shipped since that would partially mitigate their issue?
3. Implement a tiny cache where keys and values are stored in a `[]byte` arena. Lookup by binary search on a parallel `[]offset` index. Compare scan time to a `map[string]string`.
4. Why does `GOGC=off` not eliminate Go's GC entirely? What 2-minute mechanism remains, and where in `runtime/` is it?
5. Argue both sides of "Discord should have stayed on Go". What modern Go features (1.22+) would change the calculus, and what's still out of reach?
