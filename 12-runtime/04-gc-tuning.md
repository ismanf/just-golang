# GC Tuning — GOGC, GOMEMLIMIT, SetGCPercent

## TL;DR

Go's GC has exactly **two tuning knobs**: `GOGC` (target heap growth ratio, default 100) and `GOMEMLIMIT` (soft total-memory cap, default off, since 1.19). Almost everything else you can do is a side effect of these two. Use **GOMEMLIMIT in containers** (cap = ~90% of cgroup limit) and **GOGC in long-running daemons** to trade memory for CPU. The single biggest gotcha: GOMEMLIMIT is *soft*. If the program genuinely needs more memory than the limit, the GC will run **back-to-back** consuming up to 50% of CPU (the "death spiral"), but it will not refuse allocations — your container will eventually OOM-kill.

## Mental Model

```
            Heap size over time, GOGC=100:
            
   bytes
     ▲
     │            HeapGoal = 2 × HeapMarked
     │    ┌─────╲             ┌─────╲             ┌─────╲
     │   ╱       ╲___        ╱       ╲___        ╱       ╲___
     │  ╱            sweep  ╱            sweep  ╱            sweep
     │ ╱  ▲mark            ╱  ▲mark            ╱  ▲mark
     │╱   │  trigger       ╱   │  trigger      ╱   │  trigger
     └────┴───────────────┴────────────────────────────────────► time
          
                       Each cycle: live heap doubles, then drops back to marked size.
                       Higher GOGC → wider sawteeth (more memory, less CPU).
                       Lower GOGC  → narrower sawteeth (less memory, more CPU).
                       
            With GOMEMLIMIT:
            
   bytes
     ▲────────────────── soft limit ────────────────────────────
     │   ╲╱╲╱╲╱╲╱  ← cycles compress as we approach the limit
     │  ╱
     │ ╱
     │╱
     └────────────────────────────────────────────────────────► time
```

The pacer continuously adjusts when to trigger so that the cycle finishes near the goal. GOGC sets the goal; GOMEMLIMIT clamps it from above.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	prev := debug.SetGCPercent(50) // GOGC=50: trigger at 1.5× live size
	fmt.Println("previous GOGC:", prev)

	debug.SetMemoryLimit(4 << 30) // 4 GiB
	// Calling SetMemoryLimit(-1) disables the limit (default).
}
```

Environment alternatives (read at startup):

```
GOGC=200            # less frequent GC; more memory, less CPU
GOGC=off            # disable GC entirely
GOMEMLIMIT=3GiB     # 3 GiB soft cap. Units: B, KiB, MiB, GiB, TiB
GOMEMLIMIT=off      # disable (default)
```

`GOGC=off` is equivalent to `SetGCPercent(-1)`. `GOMEMLIMIT` accepts `KB/MB/GB` (decimal) or `KiB/MiB/GiB` (binary).

## Deep Dive

### GOGC, formally

Per `runtime/debug` documentation:

> GOGC sets the initial garbage collection target percentage. A collection is triggered when the ratio of freshly allocated data to live data remaining after the previous collection reaches this percentage.

```
HeapGoal = HeapMarked × (1 + GOGC/100)
```

- `GOGC=100` (default): `HeapGoal = 2 × HeapMarked`.
- `GOGC=50`: `HeapGoal = 1.5 × HeapMarked`.
- `GOGC=200`: `HeapGoal = 3 × HeapMarked`.
- `GOGC=off` or `SetGCPercent(-1)`: `HeapGoal = ∞`, GC never auto-triggers.

`HeapMarked` is "bytes still live after the previous mark". So GOGC tunes the *amplitude* of the sawtooth.

### GOMEMLIMIT, formally

Per `runtime/debug.SetMemoryLimit`:

> The memory limit affects the runtime's behavior in the same way as GOGC, but the GOGC mechanism still applies: the runtime will trigger a GC when it has allocated GOGC% more heap than the last live size, OR when total memory approaches the limit.

The exact rule:

```
HeapGoal = min(
    HeapMarked × (1 + GOGC/100),
    memoryLimit - nonHeapMemory
)
```

`nonHeapMemory` includes stacks, runtime metadata, GC bookkeeping, and so on. The runtime reserves margin for these so heap doesn't claim the whole budget.

When `HeapGoal` would force more frequent GC than GOGC suggests, GOGC effectively rises. When memory is plentiful, GOGC dominates and GOMEMLIMIT is inert.

### The 50% CPU limit (death-spiral cap)

If the program is allocating faster than GC can reclaim, the pacer would in principle run GC nonstop. The runtime caps GC CPU at 50% (`gcCPULimiter`, since 1.19) to ensure the mutator can still make forward progress. Beyond that, the heap is allowed to grow past the goal (and past the soft limit) until the mutator catches up or the kernel OOM-kills the process. Watch for this with `runtime/metrics` series `/gc/cpu/limiter/last-enabled:gc-cycle`.

### Pacer signals

The pacer publishes a few stats useful for tuning:

```go
// runtime/metrics
"/gc/cycles/automatic:gc-cycles"            // auto-triggered cycles
"/gc/cycles/forced:gc-cycles"               // runtime.GC() calls
"/gc/heap/goal:bytes"                       // current goal
"/gc/heap/live:bytes"                       // current estimate of live heap
"/gc/heap/tiny/allocs:objects"              // tiny allocs (no per-object overhead)
"/gc/pauses:seconds"                        // histogram
"/gc/scan/heap:bytes"                       // bytes scanned by mark
"/gc/scan/stack:bytes"                      // stack bytes scanned
"/gc/cpu/limiter/last-enabled:gc-cycle"     // cycle where the 50% limiter kicked in
"/memory/classes/total:bytes"               // total runtime memory
```

If `/gc/cpu/limiter/last-enabled` advances, you're hitting the 50% cap — either raise GOMEMLIMIT, lower allocation pressure, or scale out.

### When to lower GOGC

Lower GOGC (e.g., 25–50) helps when:
- RSS is the bottleneck (limited container, expensive RAM).
- Live heap is dominated by long-lived objects that don't churn (you want the goal to track them tightly).
- You can afford 1–2× more GC CPU.

### When to raise GOGC

Raise GOGC (e.g., 200–500) helps when:
- CPU is the bottleneck.
- Allocations are bursty and short-lived (most die before goal).
- You have plenty of RAM headroom.

### When to use GOMEMLIMIT instead of GOGC

GOMEMLIMIT is the right knob when you don't know the live heap a priori. In a container, you know the RSS budget, not the working set. Setting GOGC alone might over- or under-shoot the budget. GOMEMLIMIT auto-adapts: when live heap is small, GC pacing is unchanged; when live heap grows toward the limit, GC accelerates.

Common deployment: **`GOMEMLIMIT=<90% of container limit>`, `GOGC=100`** (default). The limit prevents OOM; GOGC handles the steady state.

### When NOT to use GOMEMLIMIT

- **Short-lived processes** (CLI tools, batch jobs): GOMEMLIMIT only matters across many GC cycles.
- **Workloads where peak heap > limit consistently**: you'll spiral. Either scale out, optimize, or accept the OOM risk.
- **Mixed-Go-and-cgo programs that allocate much memory in C**: the cgo side doesn't count against the limit, leaving Go starved while total RSS is fine.

### GC and `runtime.GC()`

`runtime.GC()` triggers a synchronous cycle. Useful in:
- Tests (assert a finalizer ran).
- Bench setup (mark `b.ReportAllocs` baseline).
- Pre-measurement reset.

Never use it in production hot paths. It blocks until the cycle completes (mark + sweep on the heap to date).

### Interaction with sync.Pool

`sync.Pool` evicts at GC. With high GOGC (less frequent GC) the pool retains more memory; with low GOGC it gets cleared more aggressively. If `sync.Pool` is the centerpiece of a hot path, GOGC tuning visibly affects allocator throughput.

### Interaction with stacks

Stack sizes don't count against `HeapMarked` but do count against `GOMEMLIMIT`. A program with 1M goroutines holding 1 GiB of stack memory leaves less budget for the heap. `runtime/metrics` `/memory/classes/stacks/...` shows this.

### Interaction with the scavenger

The scavenger (background goroutine) returns idle pages to the OS via `madvise(MADV_DONTNEED)`. With `GOMEMLIMIT` set, scavenging is more aggressive — the runtime wants RSS, not just heap, near the budget. With `GOMEMLIMIT=off`, scavenging is more conservative; the runtime keeps idle pages around assuming the program will reuse them.

You can force scavenge with `runtime/debug.FreeOSMemory()`.

### `GODEBUG` knobs that affect GC

- `gctrace=1` — per-cycle stderr log.
- `gcpacertrace=1` — pacer-internals log (very verbose).
- `gccheckmark=1` — second STW mark to verify the first; for debugging GC bugs.
- `scavtrace=1` — scavenger activity.
- `madvdontneed=0` — use `MADV_FREE` instead of `MADV_DONTNEED` (pre-1.16 behavior).
- `clobberfree=1` — fill freed memory with garbage for use-after-free detection.
- `gcstoptheworld=N` — debug: force more STW (N=1 mark, N=2 mark+sweep).
- `inittrace=1` — log init function durations (1.16+).

### Forcing GC at boundaries

A useful pattern in batch jobs:

```go
for _, batch := range batches {
    process(batch)
    runtime.GC()              // cycle now while we're idle
    debug.FreeOSMemory()      // return pages to OS
}
```

Pauses are concentrated at known points instead of mid-batch.

## Standard Library Hooks

- `runtime/debug.SetGCPercent(percent int) int` — returns previous value.
- `runtime/debug.SetMemoryLimit(limit int64) int64` — returns previous value. `math.MaxInt64` is "off".
- `runtime/debug.SetGCPercent(-1)` — disable GC.
- `runtime/debug.FreeOSMemory()` — synchronous sweep + scavenge.
- `runtime.GC()` — synchronous cycle.
- `runtime.ReadMemStats(&m)` — STW snapshot.
- `runtime/metrics` — non-STW counters and histograms.

## Real-World Patterns

### 1. Container-friendly default

```go
package main

import (
	"os"
	"runtime/debug"
	"strconv"
)

func init() {
	// If GOMEMLIMIT isn't set in env, fall back to ~90% of explicit MEM_LIMIT_MIB.
	if os.Getenv("GOMEMLIMIT") == "" {
		if s := os.Getenv("MEM_LIMIT_MIB"); s != "" {
			if mib, err := strconv.ParseInt(s, 10, 64); err == nil {
				debug.SetMemoryLimit(mib * 1 << 20 * 9 / 10)
			}
		}
	}
}
```

In Kubernetes pods, surface `resources.limits.memory` as `MEM_LIMIT_MIB` via the downward API.

### 2. Throughput-favoring batch job

```go
package main

import "runtime/debug"

func main() {
	debug.SetGCPercent(500) // 6× live; very lazy GC
	// Or, for pure-extract jobs:
	// debug.SetGCPercent(-1) // off; only collect at exit
	process()
}

func process() {}
```

`GOGC=500` is reasonable when peak memory headroom is plenty and you want to minimize GC overhead. For pure data-pipeline binaries that allocate monotonically, `off` is fine — you exit before it matters.

### 3. Latency-favoring API server

```go
package main

import "runtime/debug"

func init() {
	debug.SetGCPercent(50)   // smaller sawtooth = lower tail latency
	debug.SetMemoryLimit(1800 << 20) // 1.8 GiB cap in a 2 GiB container
}
```

Trades ~30% more GC CPU for typically 20–40% lower p99 latency. Confirm with load tests.

### 4. Periodically force GC for predictable pauses

```go
package main

import (
	"runtime"
	"time"
)

func main() {
	go func() {
		t := time.NewTicker(30 * time.Second)
		defer t.Stop()
		for range t.C {
			runtime.GC()
		}
	}()
	serve()
}

func serve() {}
```

Used in latency-sensitive services to keep heap close to live size — auto pacing can let heap grow toward `2 × live` and trigger at inconvenient moments. Forcing a cycle every N seconds smooths the heap curve.

### 5. Tune from observed metrics

```go
package main

import (
	"runtime/debug"
	"runtime/metrics"
)

func liveHeapMiB() uint64 {
	s := []metrics.Sample{{Name: "/gc/heap/live:bytes"}}
	metrics.Read(s)
	return s[0].Value.Uint64() >> 20
}

func main() {
	// Adapt GOGC based on observed live heap.
	if liveHeapMiB() > 1024 {
		debug.SetGCPercent(50)
	} else {
		debug.SetGCPercent(100)
	}
}
```

A heuristic feedback loop. Useful for processes whose live heap changes by 10× between modes (e.g., bulk import phase vs serving phase).

## Anti-Patterns & Gotchas

**Setting GOMEMLIMIT to the cgroup hard limit.** Non-heap memory (stacks, runtime, cgo) consumes 5–15% on top of heap. Set GOMEMLIMIT to ~90% to leave headroom.

**Setting both GOGC=off and GOMEMLIMIT.** Without GOGC the soft limit becomes the only trigger; you'll only collect at the brink. RSS sawtooths between live-heap and the limit. Use GOGC≥10 alongside.

**Tuning blindly.** `GODEBUG=gctrace=1` plus a load test is the minimum bar. Production gauges (`/gc/pauses:seconds` histogram) tell you whether tuning helped.

**Treating SetGCPercent as a per-cycle knob.** It's a global ratio. Changes mid-process affect the next pacer recalibration; transient changes are usually noise.

**Using runtime.GC() to "fix" memory growth.** Forcing GC doesn't free anything that wasn't already going to be freed. If RSS grows monotonically, you have a leak.

**Confusing leak with bloat.** A leak grows live heap forever; tighter GOGC won't help. Bloat means transient peaks; GOGC or pools can. Use heap profiles to distinguish.

**Forgetting that `runtime/metrics` is the right counter source.** `ReadMemStats` is STW. In production use `/gc/heap/live:bytes`, `/gc/pauses:seconds`, `/memory/classes/*`.

**Believing GOMEMLIMIT will OOM-protect you.** It's soft. If you genuinely need more, GC stalls (50% cap) then the OOM-killer gets you. Real protection requires either right-sizing or scaling out.

**`debug.SetGCPercent(0)`.** That's GOGC=0, meaning "collect on every byte" — pathological. To disable, use `-1`.

**Tuning without first reducing allocations.** Most workloads benefit far more from cutting allocation rate than from any GC tuning. Profile first.

## Performance Notes

Rules of thumb (varies by workload; always benchmark):

- GOGC 50 → ~30% lower p99 latency vs default, ~25% more GC CPU.
- GOGC 200 → ~30% more memory, ~15% less GC CPU.
- GOGC 500 → ~80% more memory, ~25% less GC CPU.
- GOGC off → max throughput, monotonically growing heap.
- GOMEMLIMIT pressure (>90% utilization): GC CPU climbs sharply.
- `runtime.GC()` cost: equivalent to one full cycle; varies with live heap.
- `debug.FreeOSMemory()`: cycle + scavenge, can hold the calling goroutine ~ms to ~s on huge heaps.

A workload generating 1 GiB/s of allocations with 10 MiB live heap will collect ~10 times per second under default GOGC, ~3 times per second under GOGC=300. The GC scans only live; allocation rate is mostly a write barrier cost.

## How Big Companies Use It

- **Cloudflare's `GOMEMLIMIT` deployment**: "Go 1.19's GOMEMLIMIT, two months in production" — https://blog.cloudflare.com/two-go-memory-related-features/.
- **Discord** post-Rust-migration documented the GC pain that motivated the move; their Go service used `GOGC=10` at one point, paying CPU for stable tail latency — https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **Uber's recommended defaults** in their Go style guide include guidance on `GOGC=200` for batch jobs and tighter values for online services.
- **Twitch's Go GC posts** chronicle pacer tuning over multiple Go versions: https://blog.twitch.tv/en/2016/07/05/gos-march-to-low-latency-gc-a6fa96f06eb7/.
- **CockroachDB** ships `GOGC=100` default but documents `GOGC=50` for memory-constrained nodes: https://www.cockroachlabs.com/docs/.
- **Tailscale**'s `tailscaled` uses `GOMEMLIMIT` aggressively to stay within tight embedded budgets.
- **Bytedance** documented `GOMEMLIMIT` saving them ~10% RSS across their fleet at GopherCon China 2023.

## Source Code References

Pinned to `go1.26`.

- Pacer logic: [`src/runtime/mgcpacer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcpacer.go).
- `SetGCPercent`: [`src/runtime/debug/garbage.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/debug/garbage.go).
- `SetMemoryLimit`: [`src/runtime/debug/garbage.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/debug/garbage.go) (search `SetMemoryLimit`).
- 50% CPU limiter: [`src/runtime/mgclimit.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgclimit.go).
- Pacer recalculation: [`src/runtime/mgcpacer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcpacer.go) — function `revise`.
- Soft limit reservations: [`src/runtime/mgc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgc.go) — search `memoryLimit`.
- Scavenger pacing: [`src/runtime/mgcscavenge.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcscavenge.go) — search `heapRetained` and `scavengeGoal`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "A Guide to the Go Garbage Collector": https://go.dev/doc/gc-guide (official; the single most useful doc on this topic).
- Michael Knyszek, "Respecting boundaries — GOMEMLIMIT" (GopherCon 2022): https://www.youtube.com/watch?v=We3yi2OtcnI.
- Michael Knyszek, "Soft memory limits" proposal: https://github.com/golang/proposal/blob/master/design/48409-soft-memory-limit.md.
- Austin Clements, "Go 1.5 concurrent GC pacing": https://golang.org/s/go15gcpacer.
- Rick Hudson, "Go's garbage collector journey" — chronicles pacer evolution: https://go.dev/blog/ismmkeynote.
- "Go memory management" — Povilas Versockas (GopherCon EU 2018): https://www.youtube.com/watch?v=jD3wCw3GIDo.
- Damian Gryski, "go-perfbook — GC tuning": https://github.com/dgryski/go-perfbook.

## Exercises / Self-Check

1. A service has live heap 500 MiB, runs in a 2 GiB container. What `GOMEMLIMIT` should you set? Why not 2 GiB?
2. `GOGC=200`. Live heap after last GC was 1 GiB. What is the next trigger heap size?
3. Your `gctrace` shows assist time consuming 40% of cycles. What knobs would you adjust first, and what would each do?
4. Why does `GOGC=off` not give the lowest GC CPU forever? (Hint: think about what `runtime/metrics` reports versus reality.)
5. With `GOMEMLIMIT=500MiB` and a workload whose live heap legitimately grows to 1 GiB, sketch the timeline: heap, GC CPU, RSS, and the final outcome.
