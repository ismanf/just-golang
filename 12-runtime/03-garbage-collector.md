# Garbage Collector — Tricolor, Write Barriers, Soft Memory Limit

## TL;DR

Go's GC is a **concurrent, non-generational, non-compacting, tricolor mark-sweep** collector with a **hybrid Dijkstra/Yuasa write barrier**. It runs alongside the program with only two **stop-the-world** (STW) phases — both <100 µs on modern hardware. Pacing is driven by `GOGC` (default 100 — meaning "trigger when heap is 2× the live set after last GC") and `GOMEMLIMIT` (soft cap, since 1.19). As of Go 1.25+, the **Green Tea GC** is the production collector — a region-aware redesign that improves locality on large heaps. The single biggest gotcha: heap *size* is what GC tracks, not allocation *rate*; a workload that allocates fast but recycles into a sync.Pool stays cheap.

## Mental Model

```
   GC cycle phases (concurrent with the mutator):

   ┌──────────┐  ┌─────────┐  ┌────────────────┐  ┌─────────────┐  ┌──────────┐
   │ STW Mark │→ │  Mark   │→ │  STW Mark Term │→ │  Sweep      │→ │ (idle)   │
   │ Setup    │  │ (concur)│  │  (~µs to ms)   │  │ (concurrent)│  │          │
   │ (<100µs) │  │         │  │                │  │             │  │          │
   └──────────┘  └─────────┘  └────────────────┘  └─────────────┘  └──────────┘
        ↑                                                                 ↓
        └─────────  triggered when HeapLive ≥ HeapTrigger ←───────────────┘

   Tricolor abstraction (each object is one color at all times):

           WHITE — unmarked, candidate for sweep
           GREY  — marked, children not yet scanned (on GC work queue)
           BLACK — marked, children scanned

   Invariant maintained by the write barrier:
     "Black must not directly reference white" (otherwise white gets swept while live).
```

The collector runs the mutator and GC threads concurrently. Reachability is established by starting from roots (stacks, globals, registers, finalizer queues) and graying every object you reach. As you scan a grey object, you gray its referents and blacken the object. When the work queue empties, anything still white is unreachable.

## Syntax & Basic Usage

The GC has no API for "collect this"; you only control pacing and observe results.

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	debug.SetGCPercent(100)              // GOGC=100 (default)
	debug.SetMemoryLimit(2 << 30)        // 2 GiB soft cap
	runtime.GC()                         // synchronous: trigger a full cycle now

	var s runtime.MemStats
	runtime.ReadMemStats(&s)
	pause := time.Duration(s.PauseNs[(s.NumGC+255)%256])
	fmt.Printf("NumGC=%d Pause=%v HeapAlloc=%dMiB\n",
		s.NumGC, pause, s.HeapAlloc>>20,
	)
}
```

Trace per-cycle output:

```
$ GODEBUG=gctrace=1 ./bin
gc 1 @0.014s 1%: 0.012+0.78+0.020 ms clock, 0.10+0.10/0.62/1.5+0.16 ms cpu, 4->5->2 MB, 5 MB goal, 0 MB stacks, 0 MB globals, 8 P
```

Fields: `gc N @T s P%`, `STW1+MARK+STW2 ms clock`, `cpu`, `heap-before -> heap-during -> heap-after MB`, `goal MB`, `stacks/globals`, `Ps`.

## Deep Dive

### Tricolor invariant

The collector marks objects grey, then black. The mutator can race with this:

- **Mutator writes** a black object's field to point to a white object. Now black → white, violating the invariant. The white object would be missed.

To prevent that, every pointer write while GC is active runs the **write barrier**, a tiny snippet of code injected by the compiler.

### The hybrid write barrier (Go 1.8+)

Pre-1.8 Go used a *stack rescan* during STW: at the end of marking, every goroutine stack was rescanned to catch pointers stored during concurrent marking. With deep stacks (databases, RPC services) this STW grew to seconds.

1.8 replaced rescan with **Dijkstra + Yuasa hybrid**:

```go
// runtime/mwbbuf.go (excerpt; conceptual)
// On write *slot = ptr:
if writeBarrier.enabled {
    shade(ptr)        // Dijkstra: any pointer being written gets shaded
    shade(*slot)      // Yuasa:    the previous value gets shaded too
}
*slot = ptr
```

Both arms exist because each guards a different invariant:

- **Dijkstra** (shade target): maintains "black does not reference white". Sufficient if all roots (stacks) are blackened at start.
- **Yuasa** (shade source): catches the case where a pointer is overwritten *before* GC reaches that field. Eliminates the need for stack rescan.

The pair lets the runtime mark stacks once, at goroutine start, and never re-scan them — so STW termination is constant-time. This is Austin Clements' design from [proposal #17503](https://go.dev/issue/17503).

The compiler inlines a check `if writeBarrier.enabled` into every pointer write; when GC isn't running, the cost is one load + branch (negligible).

### Pacing

The pacer's goal: complete a GC cycle exactly as heap size reaches **HeapGoal**, where:

```
HeapGoal ≈ HeapMarked × (1 + GOGC/100)
```

`GOGC=100` (default) means heap grows to 2× live size before triggering. `GOGC=200` means 3× (less frequent GC, more memory). `GOGC=off` disables GC entirely.

The runtime estimates marking rate (`gcController.assistWorkPerByte`) and starts GC early enough that mark finishes by the time the heap hits the goal. Mid-cycle, if the mutator is allocating faster than marking, **GC assist** kicks in: the allocator helps mark proportional to bytes it's about to allocate. This is what `assist` time means in gctrace output.

`GOMEMLIMIT` (since 1.19, `runtime/debug.SetMemoryLimit`) is a *soft* cap. When total runtime memory approaches it, the pacer trades CPU for memory: more frequent GC, smaller goal, more aggressive scavenging. Once exceeded, GC runs back-to-back ("death spiral"); the limit is *soft* because GC can't refuse allocations the program needs.

### STW phases

There are two:

1. **STW Mark Setup** (~<10 µs): freeze every goroutine, enable write barriers, gather root pointers' addresses.
2. **STW Mark Termination** (~<100 µs): finish marking remaining work, swap the write-barrier-protected state, disable barriers.

Both go through `stopTheWorldWithSema` / `startTheWorldWithSema`. With async preemption (1.14+) the time to actually stop all Gs is bounded by the signal latency (~10–100 µs).

### Sweep

Sweep is **lazy and concurrent**. As goroutines allocate, they sweep a few spans first (proportional to alloc size). Background sweepers also exist. Sweep frees objects' slots in their spans and returns empty spans to mheap.

A span isn't truly "freed" until **mheap_.freeSpan** marks it; even then the physical pages stay reserved. The *scavenger* (separate background activity) returns idle pages to the OS via `madvise(MADV_DONTNEED)`.

### Marking

The mark phase scans the **type bitmap** for each object — one bit per word indicating "is this word a pointer?" The scanner is type-aware; it never treats integers as pointers (no conservative scanning). This is what `noscan` spans bypass entirely: their objects are guaranteed pointer-free.

Roots scanned:
- All goroutine stacks (using compiler-emitted stack maps).
- Globals (one bitmap per package).
- Finalizer / cleanup queue entries.
- The map of allocated heap arenas (for span-level metadata).

After roots are grey-listed, mark workers (`gcBgMarkWorker`, one per P) dequeue grey objects, scan their pointers, gray any unmarked referents.

### Mark workers and dedicated/fractional Ps

The pacer chooses how many Ps are "dedicated" to marking vs "fractional" (split time with the mutator) vs "idle" (only mark when the P would otherwise be idle). At ~25% GC CPU target (`gcCPULimiter`), the pacer reserves one P every four for marking when needed.

You can see this with `GOMAXPROCS=8 GODEBUG=gctrace=1 ./bin`: under heavy load, gctrace's `cpu` field reports `STW+(mark assist+mark dedicated+mark fractional)+STW2 ms cpu`.

### Green Tea GC (1.25+)

The mark phase pre-1.25 traverses the heap in a graph-order that thrashes cache. Green Tea (Michael Knyszek, proposal [#73581](https://github.com/golang/go/issues/73581)) reorganizes marking around **regions** — contiguous chunks of memory marked together. This:
- Reduces L2/L3 misses during mark.
- Trades a bit of per-region bookkeeping for much better cache locality.
- Allows opportunistic compaction within a region (still non-compacting *across* the heap).

In 1.25 it's the default; you can revert with `GOEXPERIMENT=nogreenteagc` (for reproducing pre-1.25 behavior in benchmarks).

### Finalizers and cleanups

Finalizer-queued objects can't be collected in the cycle that finds them unreachable. They get an extra cycle to run their finalizer; that often resurrects them. See `12-runtime/07-finalizers-cleanups.md`. `runtime.AddCleanup` (since 1.24) is a saner replacement.

### What the GC does NOT do

- **No moving / compaction.** Pointers never change addresses (cgo and unsafe code depend on this).
- **No generations.** Every cycle scans everything reachable.
- **No reference counting.** No per-object overhead beyond the type bitmap.
- **No backup scanner.** It's precise — conservative roots would be unsound with stack copying.

### GC and unsafe

If you stash a `uintptr` derived from a pointer and the GC runs, the uintptr is *not* tracked. You can lose the object. The rule: convert to uintptr and back **atomically** — only inside a single expression. See `11-low-level/03-unsafe-pointer.md`.

## Standard Library Hooks

- `runtime.GC()` — force a full cycle, block until done. Useful only in tests and forcing cleanup of finalizers.
- `runtime/debug.SetGCPercent(percent int) int` — set GOGC programmatically; -1 disables.
- `runtime/debug.SetMemoryLimit(limit int64) int64` — set soft cap (1.19+).
- `runtime/debug.ReadGCStats(stats *GCStats)` — historical pause times.
- `runtime.MemStats` — comprehensive counters via `ReadMemStats` (STW cost).
- `runtime/metrics` — same data without STW (preferred for monitoring; sample names listed at [pkg.go.dev/runtime/metrics](https://pkg.go.dev/runtime/metrics)).
- `runtime.SetFinalizer(obj, fn)` — schedule cleanup at unreachability (legacy; prefer `runtime.AddCleanup`).
- `runtime.AddCleanup(obj, fn, arg)` (since 1.24) — cleaner alternative.
- `runtime.KeepAlive(x)` — pin GC liveness.
- `GODEBUG=gctrace=1` — per-GC stderr line.
- `GODEBUG=gcpacertrace=1` — pacer internals (very verbose).
- `GODEBUG=clobberfree=1` — fill freed memory with 0xdeadbeef for use-after-free detection.

## Real-World Patterns

### 1. Set GOMEMLIMIT in a container

```go
package main

import "runtime/debug"

func main() {
	// Container limit is 2 GiB. Reserve ~10% headroom for non-heap memory
	// (stacks, runtime metadata, cgo).
	debug.SetMemoryLimit(1800 << 20)
	// ...
}
```

Or, set environment variable: `GOMEMLIMIT=1800MiB`. The runtime now treats this as the budget; GC will run more aggressively as you approach it, but it won't OOM the container.

### 2. Disable GC for short tools

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	debug.SetGCPercent(-1) // off: max throughput, no GC cost
	// build a large index, exit
	idx := make(map[string]int, 1<<20)
	for i := 0; i < 1<<20; i++ {
		idx[fmt.Sprintf("k%d", i)] = i
	}
	_ = idx
}
```

Useful for batch / build tools whose memory profile is monotonically growing — there's nothing to collect. Saves significant wall time on millions-of-allocation workloads.

### 3. Pacing-friendly batch processing

```go
package main

import "runtime"

func processBatch(items []Item) {
	defer runtime.GC() // force GC at known boundary, not mid-batch
	for _, it := range items {
		handle(it)
	}
}

type Item struct{}
func handle(Item) {}
```

Forcing GC at quiet points keeps tail latency from spiking mid-batch when GC happens to trigger.

### 4. Monitor live heap without STW

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func liveHeapBytes() uint64 {
	s := []metrics.Sample{{Name: "/memory/classes/heap/objects:bytes"}}
	metrics.Read(s)
	return s[0].Value.Uint64()
}

func main() {
	fmt.Println("live:", liveHeapBytes())
}
```

`runtime/metrics` (since 1.16) has hundreds of named gauges/histograms with no STW cost. Prefer it over `ReadMemStats` for production monitoring. See `08-stdlib/27-runtime.md`.

### 5. Investigate a pause

```
$ GODEBUG=gctrace=1,gcpacertrace=1 ./bin 2> trace.log
$ grep '^gc ' trace.log | awk '{print $5}' | sort -n | tail
```

Top 10 wall-clock pause times across all cycles. If `STW1` and `STW2` are small but total mark time is huge, you're CPU-bound on marking; if assist time dominates, the mutator is outrunning the marker (raise GOGC or reduce alloc rate).

## Anti-Patterns & Gotchas

**Treating `runtime.GC()` as a performance tool.** It blocks. Use it only in tests or before measurements.

**Setting `GOMEMLIMIT` to the container's hard limit.** Leave ≥10% headroom for non-heap allocations (stacks, runtime, cgo). Hitting the soft limit causes GC pressure; hitting the hard limit causes OOM.

**Disabling GC and forgetting to re-enable.** `SetGCPercent(-1)` is permanent. Server processes leak memory until `SetGCPercent(100)` runs.

**Pinning huge objects with `runtime.KeepAlive` past their actual liveness.** Defeats the whole point of GC. Use only when interacting with unsafe / cgo.

**Allocating in a tight loop that never escapes a function.** Allocation rate itself is fine; the issue is *live heap*. A loop that allocates and discards stays small. A loop that retains everything in a slice grows the goal.

**Believing `MADV_FREE` (pre-1.16) means memory is "released".** RSS stayed up because Linux didn't reclaim until pressure. Use `madvdontneed=1` (default 1.16+).

**Reading `MemStats.PauseNs` as an average.** It's a 256-element ring of recent pauses. Aggregate properly: `runtime/metrics` exposes a histogram (`/gc/pauses:seconds`).

**Heap-fragmenting via many sync.Pools.** Pools accelerate alloc and recycle, but if you have hundreds of pools each holding ~1 MiB of unused buffers, the heap stays large between cycles.

**Triggering GC manually under `GOMAXPROCS=1`.** Mark workers compete with the mutator on the single P; pauses can be far longer than expected.

**Trying to time GC for "off-hours".** Modern Go GC is concurrent; the STW pauses are sub-millisecond. Tuning for "low-traffic windows" is rarely worth it.

## Performance Notes

- STW pause (mark setup + mark term combined): typically <100 µs, ~1 ms in pathological cases.
- Mark CPU cost: ~25% of one core (target); scales with heap *scan* size, not allocation rate.
- Sweep CPU cost: amortized into allocator; ~1–2% under typical load.
- Write barrier overhead during GC: 1–10% CPU on pointer-heavy workloads.
- Scavenger pause cost on the M it runs on: negligible; runs as background goroutine.
- GOGC=50 (more frequent GC): ~20% more CPU, ~30% less RSS.
- GOGC=200 (less frequent GC): ~10% less CPU, ~50% more RSS.
- GOMEMLIMIT pressure: above ~95% utilization, GC CPU rises sharply (the pacer accelerates).
- Green Tea (1.25+) typical improvement on >10 GiB heaps: 5–15% lower mark CPU, 2–5% lower p99 latency.
- A 1 GiB *all-pointer* heap takes ~150 ms to mark single-threaded; concurrency divides that across mark workers.
- A 1 GiB *noscan* heap takes near-zero to mark — the GC skips spans entirely.

## How Big Companies Use It

- **Discord's "Why we're switching from Go to Rust"** is the canonical pointer-heavy-heap horror story: 100 GB live, ~50 GiB scanned every cycle, 100 ms+ p99 pauses — https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **Twitch's "Go GC: Solving the Latency Problem"** describes the engineering behind getting an IRC server's p99 below 10 ms; one of the loudest pre-1.8 pacer change advocates: https://blog.twitch.tv/en/2016/07/05/gos-march-to-low-latency-gc-a6fa96f06eb7/.
- **Cloudflare** uses `GOMEMLIMIT` aggressively on edge boxes to keep RSS under cgroup limits: https://blog.cloudflare.com/go-memory-limits/.
- **Uber's M3DB** time-series database documents per-shard GC tuning: https://eng.uber.com/m3/.
- **CockroachDB** ships with per-node GC pacing knobs and documents the cost model: https://www.cockroachlabs.com/docs/stable/cluster-settings.html.
- **Bytedance / TikTok** has a CGO-light reimplementation of grpc to avoid write barrier overhead on hot RPC paths (talked about at GopherCon China 2023).
- **Google's Vitess** uses `runtime.GC()` after each catch-up cycle to keep MVCC-heavy heaps from growing unboundedly.

## Source Code References

Pinned to `go1.26`.

- Top-level GC loop: [`src/runtime/mgc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgc.go) — search `gcStart`, `gcMarkDone`, `gcMarkTermination`.
- Pacer: [`src/runtime/mgcpacer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcpacer.go).
- Mark workers: [`src/runtime/mgcmark.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcmark.go).
- Sweep: [`src/runtime/mgcsweep.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcsweep.go).
- Write barriers (compiler intrinsic emitter): [`src/cmd/compile/internal/ssa/writebarrier.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/writebarrier.go).
- Write barrier buffer (runtime side): [`src/runtime/mwbbuf.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mwbbuf.go).
- Scavenger: [`src/runtime/mgcscavenge.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcscavenge.go).
- Green Tea: [`src/runtime/mgcmark_greenteagc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcmark_greenteagc.go) (1.25+).
- Soft memory limit: [`src/runtime/mgclimit.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgclimit.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Austin Clements, "Getting to Go: The Journey of Go's Garbage Collector" (GopherCon 2018): https://go.dev/blog/ismmkeynote
- Austin Clements, "Eliminate STW stack re-scanning" (proposal #17503): https://go.dev/issue/17503
- Rick Hudson, "Go GC: Latency Problem Solved" (GopherCon 2015): https://www.youtube.com/watch?v=aiv1JOfMjm0
- Michael Knyszek, "Soft memory limits" (proposal #48409): https://github.com/golang/proposal/blob/master/design/48409-soft-memory-limit.md
- Michael Knyszek, "Green Tea GC" (proposal #73581): https://github.com/golang/go/issues/73581
- "Go's Memory Model" (canonical doc): https://go.dev/ref/mem
- "Concurrent GC algorithms" — Jones, Hosking, Moss, *The Garbage Collection Handbook*.
- Dmitry Vyukov, "Memory barriers and the Go memory model": http://www.1024cores.net/home/lock-free-algorithms/so-what-is-a-memory-model
- Rhys Hiltner (Twitch), "An ode to the garbage collector": https://about.sourcegraph.com/podcast/rhys-hiltner

## Exercises / Self-Check

1. Sketch the four GC phases on a timeline. For each, mark whether the mutator is stopped and what work is happening.
2. Why does the hybrid write barrier need both Dijkstra and Yuasa? What goes wrong with just one?
3. A program runs `GODEBUG=gctrace=1` and shows assist time > mark dedicated time in every cycle. What does that tell you about the workload, and how would you fix it?
4. Compute the relationship between `GOGC`, `HeapMarked`, and `HeapGoal`. If a service has `HeapMarked=4 GiB` and `GOGC=200`, what's the goal?
5. Set `GOMEMLIMIT=500MiB` on a program that needs ~600 MiB live heap. Predict the behavior. Run it and confirm with `GODEBUG=gctrace=1`.
