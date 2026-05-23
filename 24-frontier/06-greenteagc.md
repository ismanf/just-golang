# Green Tea GC — The 1.25+ GC Redesign

## TL;DR

**Green Tea GC** (Michael Knyszek, proposal [#73581](https://github.com/golang/go/issues/73581)) is the biggest GC redesign since Go 1.5's concurrent mark-sweep. Shipped as default in **Go 1.25** (Aug 2025), it reorganizes mark-phase traversal around **memory regions** to dramatically improve **cache locality** on large heaps. The mark phase scans live objects more cache-efficiently, reducing CPU time and tail latency on heaps from a few GiB up to hundreds. Headline numbers from Google's published benchmarks: **5-15% lower mark CPU**, **2-5% lower p99 latency** on large heap workloads. The single biggest gotcha: **Green Tea doesn't change the GC's contract** — it's still tricolor mark-sweep with a hybrid write barrier; just the *traversal order* changes. Workloads that struggled with Go's GC for fundamental reasons (Discord-style pointer-heavy caches) still struggle, just less.

## Mental Model

```
   Pre-Green Tea mark traversal (1.4-1.24):
   
       Mark workers process objects in graph order (BFS-ish).
       Each pointer dereference may touch arbitrary memory in the heap.
       Cache misses are frequent on large heaps.
       L2/L3 miss rates dominate mark CPU time.
   
   Green Tea mark traversal (1.25+):
   
       Mark workers process objects within a "region" (cache-line-aligned
       chunk of memory) before moving to the next.
       Pointers leaving a region are queued for later.
       Sequential memory access within each region; predictable cache hits.
       
       ┌─────────┐   ┌─────────┐   ┌─────────┐
       │ Region 0│ → │ Region 1│ → │ Region 2│ ...
       │  scan   │   │  scan   │   │  scan   │
       │ in-place│   │ in-place│   │ in-place│
       └────┬────┘   └────┬────┘   └────┬────┘
            │             │             │
            └───── outgoing pointers ───┘
                 queued in a work buffer
                 processed by region order

   Same set of objects scanned, but in a cache-friendlier order.
```

The trick: Sort GC work by *physical address* (region) so each worker accesses memory linearly. The end-to-end set of marked objects is identical to before; the traversal is just faster.

## Syntax & Basic Usage

There's no API. Green Tea is the default in Go 1.25+; you don't opt in. Opt **out** via:

```bash
GOEXPERIMENT=nogreenteagc go build .
```

Reverts to the pre-1.25 mark algorithm. Useful for A/B testing or reproducing pre-1.25 benchmarks.

Observe with the usual GC trace:

```bash
GODEBUG=gctrace=1 ./myapp
```

The output format is unchanged; the *numbers* differ.

## Deep Dive

### Motivation

Pre-1.25 GC mark phase walked the heap in graph order (work queue of grey objects → scan their pointers → grey their referents). Memory access pattern: ~random.

On modern CPUs:
- L1 cache: ~30 KiB, ~1 ns access.
- L2 cache: ~256 KiB-1 MiB, ~3 ns.
- L3 cache: ~4-32 MiB, ~10 ns.
- DRAM: GiB+, ~100 ns.

For a 10 GiB heap, random access misses L3 frequently. Mark CPU is dominated by memory stalls, not work.

Green Tea reorganizes traversal so each worker accesses a few KiB-MiB region at a time → mostly L2/L3 hits → 3-10× faster per byte scanned.

### Design

The proposal is detailed at [#73581](https://github.com/golang/go/issues/73581). Key pieces:

#### Regions

The heap is logically divided into **regions** of a few hundred KiB to a few MiB each (chosen at startup based on cache sizes). Each region knows its *base address* and *size*.

#### Region-local work queues

Each P (logical processor) has a queue of regions it's responsible for. Mark workers pop regions from this queue.

#### In-region scanning

When a worker picks a region, it scans every grey object *in that region* before moving on. Pointers found are sorted:

- **Local pointers** (point within the same region) → process in-place.
- **Remote pointers** (point to another region) → queue for that region's worker.

#### Cross-region work distribution

Workers exchange remote pointers via lock-free queues. The scheduling tries to keep regions local to a single worker for the duration.

#### Locality benefits

Each worker spends most of its time inside one region, hitting cache. Cross-region traffic is amortized.

### Why this is hard

- **Maintaining tricolor invariant**: even with reordered work, no object can be "white when scanned-from black". Green Tea uses the same write barrier; just shuffles the worklist.
- **Avoiding stalls**: a worker can run out of "local" regions; needs efficient inter-worker handoff.
- **Region size tuning**: too small and cross-region traffic dominates; too big and locality is lost. Adaptive sizing.
- **Compatibility with existing GC modes**: must work alongside soft memory limit, GC pacer, etc.

### Performance gains

Per the proposal and follow-up posts:

| Workload | Mark CPU change | p99 latency change |
|---|---|---|
| Small heap (<1 GiB) | ~0% | ~0% |
| Medium heap (1-10 GiB) | -5 to -10% | -2 to -5% |
| Large heap (10-100 GiB) | -10 to -15% | -3 to -8% |
| Huge heap (>100 GiB) | -15 to -20% | up to -10% |

Workload types that benefit most:
- Pointer-dense caches (Discord-style, before Discord went Rust).
- Large in-memory databases.
- Apiservers with many cached objects.

Workload types with smaller gains:
- Noscan-heavy heaps (no pointers to walk).
- Small heaps that fit in L3 anyway.
- Allocation-heavy / live-set-small (mark cost is low to begin with).

### Compatibility

- All existing GC tuning (`GOGC`, `GOMEMLIMIT`, `debug.SetMemoryLimit`) works unchanged.
- `runtime.GC()` is unchanged in semantics.
- `runtime/metrics` series unchanged; values shift slightly.
- `gctrace` output format unchanged.
- Write barriers unchanged.
- Pacer logic mostly unchanged (slight retuning).

You can update to Go 1.25 and observe gains without changing code or config.

### Comparison to other GCs

Green Tea is **not** generational. Go's GC remains non-generational. Pros: simpler, no per-allocation write barriers for promotions. Cons: every cycle scans everything reachable.

Generational GCs (Java, .NET) collect young generations more often, old generations less. Many workloads benefit. The Go team has explicitly declined to add generations; the engineering cost and added complexity didn't justify it given alternative wins (Green Tea is one such win).

### What Green Tea is NOT

- **Not a generational GC**. Same scan-everything-reachable contract.
- **Not a compacting GC**. Pointers don't move; cgo and unsafe code keep working.
- **Not a reference-counting GC**.
- **Not a parallel-GC-only**. Still concurrent with mutator.
- **Not a no-STW**. The two STW phases (mark start, mark term) remain; their cost was already <1 ms typically.

### Adoption notes

Google internal services upgrade-tested Green Tea before 1.25 ship; reported wins inline with the published benchmarks. No major regressions reported.

For your service:
- Upgrade to 1.25+.
- Run for a few days under load.
- Compare `runtime/metrics` series (`/gc/cpu/total:cpu-seconds`, `/gc/pauses:seconds`).
- If wins disappoint, try `GOEXPERIMENT=nogreenteagc` to confirm GC was the bottleneck.

### How the GC team measures wins

The Go team maintains a benchmark suite:
- Kubernetes apiserver under load.
- Prometheus TSDB ingestion.
- A synthetic "Discord-like" cache.
- Stdlib's own benchmarks.

Green Tea was tested across all; the published numbers are the *worst-case* of these benchmarks, not the best.

### Future work

The proposal mentions follow-ups:
- Cross-region inlining of "trivial" pointers.
- Adaptive region sizing based on observed locality.
- Potential combination with other GC innovations (page reclamation, scavenger improvements).

The roadmap is one improvement per major release for the foreseeable future.

### Working alongside soft memory limit

`GOMEMLIMIT` (1.19+) interacts with Green Tea. The pacer recalibrates with new mark CPU costs; goal-pressure logic is slightly retuned.

Under memory pressure (near GOMEMLIMIT), Green Tea's lower mark CPU means more headroom for the mutator. Net effect: GOMEMLIMIT-pressured workloads are more responsive.

### Tuning post-Green Tea

Mostly unchanged. The same advice applies:
- Set `GOMEMLIMIT` in containers (~90% of cgroup limit).
- Tune `GOGC` for memory vs CPU trade-off.
- Avoid pointer-dense long-lived heaps.
- Use sync.Pool, byte arenas, noscan-friendly layouts for hot paths.

Green Tea moves the bar; doesn't eliminate the considerations.

### The naming

"Green Tea" is a tongue-in-cheek codename. The original "milk tea" / "boba" naming theme threaded through several internal Go projects.

## Standard Library Hooks

No new public API. Observe:

- `GODEBUG=gctrace=1` — per-cycle stderr.
- `GODEBUG=gcpacertrace=1` — pacer internals.
- `runtime/metrics`:
  - `/gc/cpu/total:cpu-seconds`
  - `/gc/cpu/limiter/last-enabled:gc-cycle`
  - `/gc/scan/heap:bytes`
  - `/gc/pauses:seconds`
- `runtime.MemStats` — same fields, different numbers.

Opt out:
- `GOEXPERIMENT=nogreenteagc` (build-time).

## Real-World Patterns

### 1. Profile mark CPU with/without

```bash
# With Green Tea (default in 1.25+)
GODEBUG=gctrace=1 ./myapp 2>green.log

# Without
GOEXPERIMENT=nogreenteagc GODEBUG=gctrace=1 ./myapp 2>nogreen.log

# Compare mark times
grep '^gc ' green.log | awk '{print $3}'
grep '^gc ' nogreen.log | awk '{print $3}'
```

### 2. Monitor in production

```go
package main

import (
    "fmt"
    "runtime/metrics"
)

func reportGC() {
    samples := []metrics.Sample{
        {Name: "/gc/cpu/total:cpu-seconds"},
        {Name: "/gc/scan/heap:bytes"},
        {Name: "/gc/pauses:seconds"},
    }
    metrics.Read(samples)
    for _, s := range samples {
        fmt.Println(s.Name, s.Value)
    }
}
```

Read periodically; track over time.

### 3. Combine with PGO

Green Tea + PGO + GOMEMLIMIT is the modern Go performance stack:

```bash
$ GOMEMLIMIT=1800MiB go build -pgo=auto -o app .
$ ./app  # Go 1.25+, Green Tea by default
```

Each contributes a few percent. Together: typically 5-15% throughput improvement vs unoptimized.

### 4. Tune GOGC after upgrading

After moving to 1.25:

```bash
# Before: GOGC=50 to keep latency low
# After: GOGC=75 might be acceptable; Green Tea has shaved mark CPU
```

Try lifting GOGC; measure latency. Often you can let heap grow more without paying the latency tax you used to.

### 5. Validate with load test

```bash
# Pre-upgrade
$ vegeta attack -duration=10m | tee pre.bin
$ vegeta report < pre.bin

# Upgrade Go to 1.25, redeploy
$ vegeta attack -duration=10m | tee post.bin
$ vegeta report < post.bin
```

Compare p50/p95/p99. Real-world workloads vary.

## Anti-Patterns & Gotchas

**Expecting Green Tea to "fix" pointer-dense heaps.** It reduces cost; doesn't change fundamentals. If your service was GC-bound at scale, look at heap layout.

**Disabling Green Tea with `GOEXPERIMENT=nogreenteagc` in production "for stability".** Green Tea has been heavily tested; no known regressions. Don't.

**Skipping load tests after Go upgrade.** Always validate. Even improvements can shift behavior.

**Reading old GC tuning advice.** Most still applies; some heuristics have changed slightly. Re-test rather than trust 2020 blog posts.

**Treating mark CPU savings as latency-only wins.** They're CPU. Use the freed CPU for the mutator (more throughput) or accept reduced cluster footprint.

**Assuming Green Tea will help small heaps.** It won't (much). Their L3 hit rate was already fine.

**Mixing benchmarks from pre-1.25 and 1.25+ Go versions.** Always note the Go version on benchmark output.

## Performance Notes

(From Michael Knyszek's published material.)

- Mark CPU reduction: 5-15% typical on large heaps.
- p99 latency reduction: 2-5% on representative server workloads.
- Steady-state heap: unchanged.
- Allocations/sec: unchanged.
- Goroutine throughput: unchanged.
- Binary size: negligible increase.

Workload sensitivity:
- Pointer density: more pointers → more gain.
- Heap size: larger → more gain.
- Allocation rate: orthogonal; doesn't change Green Tea behavior.

## How Big Companies Use It

Green Tea ships in 1.25 (Aug 2025). Adopters are upgrading through 2026:

- **Google internal**: tested extensively pre-ship; production rollout in 2025.
- **Cloudflare**: documented evaluation, similar wins to other large-heap workloads.
- **CockroachDB**: ships with 1.25+ for new releases.
- **Tailscale**: upgrades quickly; reports as part of normal release notes.
- **Kubernetes**: tracks Go minimums; 1.25 is in the support window.

Most workloads see 2-7% improvement; pointer-heavy ones see more.

## Source Code References

Pinned to `go1.26`.

- Green Tea mark code: [`src/runtime/mgcmark_greenteagc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcmark_greenteagc.go).
- GC pacer: [`src/runtime/mgcpacer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgcpacer.go).
- Main GC loop: [`src/runtime/mgc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mgc.go).
- Proposal #73581: https://github.com/golang/go/issues/73581.
- Design notes: [`design/73581-green-tea-gc.md`](https://github.com/golang/proposal/blob/master/design/73581-green-tea-gc.md).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Proposal #73581 (Green Tea GC): https://github.com/golang/go/issues/73581.
- "A Guide to the Go Garbage Collector" (general docs, updated for 1.25): https://go.dev/doc/gc-guide.
- Michael Knyszek, "Green Tea GC" — talks at GopherCon 2025.
- Austin Clements, "Getting to Go: Journey of the Go GC" (2018 — historical context): https://go.dev/blog/ismmkeynote.
- "Inside Go's GC" — Damian Gryski blog series.
- "GC regions in Go 1.25" — release notes: https://go.dev/doc/go1.25.
- Russ Cox, "Backwards compatibility and GC changes" — research.swtch.com.
- Michael Pratt + Cherry Mui, "Profile-guided optimization in Go" (related GC adjacent work): https://go.dev/blog/pgo.

## Exercises / Self-Check

1. Upgrade a real Go service from 1.24 to 1.25. Compare `/gc/cpu/total:cpu-seconds` over a week. What's the change?
2. Why does Green Tea help large heaps more than small? Walk through the cache argument.
3. Construct a workload (pointer-dense in-memory cache) where Green Tea gives a measurable win. Build with and without (`GOEXPERIMENT=nogreenteagc`). Quantify.
4. Green Tea is NOT generational. What design tradeoffs led the Go team to keep their non-generational GC? Cite the proposal.
5. Does Green Tea reduce STW pauses? Why or why not?
