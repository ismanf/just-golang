# Anti-Case Studies — Teams That Moved Off Go

## TL;DR

Not every team that started with Go stayed. **Discord** (cache service → Rust), **Cloudflare** (proxy data plane → Rust), **InfluxData** (time-series engine → Rust), **Figma** (multiplayer state → Rust), **1Password** (CLI → Rust), and others have published detailed post-mortems. Reading them in aggregate, the pattern is consistent: **Go is "rewritten away from" mostly for one of two reasons** — (1) **GC at scale on pointer-heavy in-memory data**, (2) **the lack of zero-copy primitives and SIMD ecosystem** for analytic-style workloads. The single biggest gotcha when reading these posts: **none of them say "Go is bad"**. They say "Go was acceptable for years; we hit a scale or workload where another language is better; we're moving that specific piece, not everything". Most companies that "moved off Go" still have huge Go fleets elsewhere.

## Mental Model

```
   Migration patterns:
   
   ┌──────────────────────────────────────────────────────────┐
   │  Reason 1: Garbage Collection at scale                    │
   │   - Discord: pointer-heavy cache, p99 spikes from GC      │
   │   - Cloudflare: NGINX-replacement, per-conn determinism   │
   │   - 1Password: CLI startup time + binary size             │
   ├──────────────────────────────────────────────────────────┤
   │  Reason 2: Analytic workloads need zero-copy + SIMD       │
   │   - InfluxDB IOx: Arrow/Parquet ecosystem in Rust         │
   │   - DuckDB: never used Go; C++ for SIMD-heavy SQL          │
   │   - ClickHouse: never Go; C++ vectorized                  │
   ├──────────────────────────────────────────────────────────┤
   │  Reason 3: Embedded / ultra-resource-constrained          │
   │   - Some embedded Linux: Rust for footprint               │
   │   - WASM environments: Rust has better Wasm tools         │
   ├──────────────────────────────────────────────────────────┤
   │  Reason 4: Existing codebase / team expertise             │
   │   - Java shops staying Java rather than migrating          │
   │   - C++ shops keeping C++ for cross-team velocity          │
   └──────────────────────────────────────────────────────────┘
```

The migrations are usually **partial**: one critical service, not the company.

## Syntax & Basic Usage

This page is a case-study summary, not API documentation. The "syntax" is the public engineering blog post — read them. Every reference is linked below.

## Deep Dive

### Discord: Go → Rust for Read States

Already covered in detail in `20-big-tech/06-discord-state-service.md`. Quick summary:
- 100M+ entries in a pointer-heavy in-memory LRU.
- Periodic GC cycle every 2 minutes caused 100+ ms p99 spikes.
- Rust rewrite: p99 dropped to <2 ms.
- Discord still uses Go for many other services.

Lesson: **single-process caches with billions of GC-visible pointers** are outside Go's sweet spot.

### Cloudflare: NGINX/Lua → Rust (Pingora)

Covered in `20-big-tech/04-cloudflare.md`. Quick summary:
- The data plane proxy was C+Lua (NGINX); not Go.
- Cloudflare rewrote it in Rust (Pingora) because:
  - **Per-connection memory** in NGINX was high.
  - **Lua's GC** had its own latency issues.
  - **Tokio's async I/O** mapped well.
- Cloudflare explicitly says: "We still use Go widely; we just moved the proxy data plane".

Lesson: at edge data-plane scale (millions of QPS, microsecond budgets), Go is mid-tier. C/C++/Rust still win.

### InfluxData: TSM (Go) → IOx (Rust)

Covered in `20-big-tech/13-influxdb.md`. Quick summary:
- v1/v2 (Go) is production for years.
- IOx (Rust) targets analytical, columnar, Arrow-based workloads.
- Reasons:
  - Apache Arrow's center of gravity is Rust.
  - Parquet libraries are more mature in Rust.
  - DataFusion (Arrow SQL engine) is Rust.
  - GC pauses at high cardinality.

Lesson: for **vectorized SQL on columnar data**, the Rust + Arrow ecosystem is currently ahead.

### Figma: Multiplayer state → Rust

Figma's multiplayer collab engine was originally TypeScript on Node. They moved the document state to a server in Rust. Some of their internal services were in Go; the multiplayer-state service specifically chose Rust.

Reasons (from Figma's blog):
- Predictable latency under heavy CRDT (conflict-free replicated data type) workloads.
- Zero-copy parsing of binary protocol.
- Strong type system for the complex CRDT logic.
- No GC pauses on long-lived state.

Figma's blog: https://www.figma.com/blog/.

### 1Password: CLI → Rust

1Password's CLI (`op`) was Go. In their 2021 announcement they moved to Rust. Reasons cited:

- **Startup time**: a Go binary cold-start was ~50–100 ms on macOS. The CLI is invoked thousands of times in shell scripts.
- **Binary size**: Go binaries are ~10 MiB minimum; Rust comparable but with more tunability.
- **Memory safety guarantees**: stronger than Go for handling secrets in memory.

The 1Password CLI is a fairly small program; the migration was tractable. Many features (sync with cloud, etc.) are still in Go elsewhere in the stack.

### Other documented migrations

#### Cockroach Labs: RocksDB (C++) → Pebble (Go)

Notable because it's a **Go → Go-equivalent** migration. CockroachDB used RocksDB (C++) via cgo; cgo overhead was significant. They wrote Pebble in pure Go to eliminate the boundary. **Going *to* Go**, not away from.

#### Aurora (Discord's later effort)

After the Read States migration, Discord's "Aurora" message-search system was also Rust. Same reasoning as Read States: cache with strict latency requirements.

#### Mozilla: Some Servo components

Servo (Mozilla's experimental browser engine) was Rust from the start. Not Go-to-Rust, but Mozilla's earlier internal tools were Python/C++; the shift to Rust was driven by browser engine needs (memory safety, parallelism, low-latency).

#### Twitter: Various

Twitter migrated parts of their backend from Ruby on Rails to Scala (JVM) circa 2010. Go wasn't the destination. Mentioning for completeness — "moved off Go" stories are rare; "moved off other languages to Go" is far more common.

#### Spectre: Switched from Go to Python

Smaller-scale example: some AI/ML teams that prototyped infra in Go switched to Python for the ML side because:
- Tensor / numpy ecosystem.
- Researcher productivity.
- Most ML training is Python regardless.

Inference is sometimes Go-back; training Python.

### What these stories have in common

1. **Latency-critical service** with strict tail-latency SLOs (sub-millisecond p99).
2. **Long-lived state in memory** with many pointer-bearing objects.
3. **Per-instance scale** in the tens of GiB heap.
4. **Workload that benefits from manual memory layout** or zero-copy parsing.
5. **Team had Rust expertise** or appetite for it.

If any of these isn't true, the migration likely didn't happen.

### What these stories don't say

- "Go is slow." It's not, generally.
- "Go is poorly designed." It's not.
- "We rewrote everything." They didn't.
- "Everyone should rewrite in Rust." They don't claim this.

The "Go vs Rust" framing in the broader community is more contentious than the case studies themselves. The post-mortems are nuanced; the takes about them often aren't.

### Reading anti-case-studies well

- **Check the date.** Many were written about Go 1.13–1.16 era. Go 1.22+ has materially better GC.
- **Check the workload.** A cache is not a control plane. A SQL engine is not a CRUD app.
- **Check the scope.** "Service X moved" ≠ "company moved".
- **Check the team's expertise.** If they hired 10 Rust developers, they're invested in Rust.

### When NOT to follow the migration

- Your service is <10 GiB heap.
- Your p99 latency target is >10 ms.
- Your team is fluent in Go but not Rust.
- Your service is more I/O- than CPU-bound.
- Your service is not in the hottest critical path.

In all these cases, the rewrite cost (months to years) almost certainly exceeds the gain.

### When the migration is worth considering

- You're certain Go's GC is the bottleneck (proven via `runtime/metrics`, not vibes).
- Your team has Rust experience or strong appetite.
- The service is small and well-bounded.
- The performance gain is large (10×+).

### Lessons for Go practitioners

#### 1. Profile before considering migration

`runtime/metrics`, `pprof`, `runtime/trace`. Identify the actual bottleneck. It might not be GC.

#### 2. Try modern Go optimizations first

- `GOMEMLIMIT` (1.19+).
- Green Tea GC (1.25+).
- Noscan-friendly layouts (byte arenas, fixed-size structs).
- `sync.Pool` discipline.
- PGO (1.21+).

These together can yield 30%+ improvements without changing language.

#### 3. Consider partial rewrites

If a single hot path is the issue, isolate it. Sometimes a Go service can use cgo into a Rust library for the hot path while remaining Go elsewhere.

#### 4. Accept that Go has scale limits

Like any language, it does. Knowing them is a sign of maturity.

#### 5. Don't generalize

A success or failure story is data, not a verdict. Your workload might be different.

## Standard Library Hooks

Tools for evaluating "is this a Go workload":

- `runtime/metrics` `/gc/cpu/total:cpu-seconds`, `/gc/pauses:seconds`.
- `runtime/pprof` heap and CPU profiles.
- `runtime/trace` execution traces.
- `GODEBUG=gctrace=1` for raw GC behavior.
- `runtime/debug.SetMemoryLimit`.

Use these *before* migrating away.

## Real-World Patterns

### 1. Quantify GC overhead

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func main() {
	samples := []metrics.Sample{
		{Name: "/gc/cpu/total:cpu-seconds"},
		{Name: "/sched/latencies:seconds"},
		{Name: "/gc/pauses:seconds"},
	}
	metrics.Read(samples)
	for _, s := range samples {
		fmt.Println(s.Name, s.Value)
	}
}
```

Read these in production over weeks. If GC CPU is <5% and pause p99 is <10 ms, GC is not your bottleneck.

### 2. Test a noscan-friendly redesign

Before rewriting in Rust, redesign the hot data structure in Go to use `[]byte` arenas and fixed-size structs. Compare. Sometimes you get 80% of the win for 10% of the effort.

### 3. Pin GOMEMLIMIT and observe

In containers, setting `GOMEMLIMIT` to ~90% of cgroup limit often produces significant tail-latency improvements without code changes.

### 4. PGO

```bash
$ go build -pgo=auto ./...
```

For hot services, 2–7% throughput gain — cheap.

### 5. Isolate the hot path

If a single function is 80% of CPU, that's a candidate for cgo + a small Rust library. The rest of the service stays Go. Best of both.

## Anti-Patterns & Gotchas

**Reading "Why Discord switched" and concluding "Go is bad for caches".** It's specifically bad for that very-particular shape of cache.

**Migrating to Rust because Hacker News upvoted a post.** Validate first.

**Migrating because the team is bored.** The cost is high; the gain must be proportional.

**Migrating without measurement.** Without a baseline, you don't know the win.

**Believing the new language won't have its own issues.** Rust has compile-time pain, ecosystem gaps, hiring friction.

**Assuming "rewrite in Rust" projects take 3 months.** They take 12+ for non-trivial services.

**Reading these posts as fashion vs. data.** They're data; treat as such.

**Treating Go's stable features (>10 years) as legacy.** Mature, not legacy.

**Ignoring modern Go releases.** A 2018-Go-vs-2024-Rust comparison is unfair.

**Communicating the migration as "we hate Go".** Almost always the post-mortem is honest about Go's strengths.

## Performance Notes

Reported numbers from the case studies:

- **Discord Read States**: p99 125 ms → 2 ms (60× improvement).
- **Cloudflare Pingora**: 70% less CPU, 67% less memory than the NGINX+Lua predecessor.
- **InfluxDB IOx**: claims orders of magnitude on analytical queries (different workload than v2).
- **1Password CLI**: startup 50 ms → ~10 ms.
- **Figma multiplayer**: latency dramatically reduced; specific numbers not public.

These are big wins — but they came from rewriting *and* re-architecting; some of the gain is from the re-architect alone.

## How Big Companies Use It (the broader picture)

Even the companies that moved a piece away still have huge Go fleets. Examples:

- **Discord**: still Go for API, gateway, analytics.
- **Cloudflare**: still Go for control plane, DNS (RRDNS), Workers control plane.
- **InfluxData**: Telegraf is Go and dominant; v1/v2 still production.
- **Figma**: Go elsewhere in their stack.
- **1Password**: server-side services are Go.
- **Mozilla**: many tools Go.

Migration ≠ abandonment.

## Source Code References

Anti-case-study source material:

- "Why Discord is switching from Go to Rust": https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- "Why we built Pingora": https://blog.cloudflare.com/pingora-open-source/.
- "Announcing InfluxDB IOx": https://www.influxdata.com/blog/announcing-influxdb-iox/.
- "Why we use Rust at Figma": https://www.figma.com/blog/.
- "1Password CLI 2.0" (Rust migration announcement): https://blog.1password.com/.

## Further Reading

- The case studies themselves (above).
- "A Guide to the Go Garbage Collector": https://go.dev/doc/gc-guide.
- "Profile-guided optimization in Go": https://go.dev/blog/pgo.
- Russ Cox, "Go's compatibility promise": https://go.dev/doc/go1compat.
- "When NOT to use Go" — various blog posts; calibrate your expectations.
- Tef ("the boring problem"), "Programming as theory building" (general).
- "Choose Boring Technology" — Dan McKinley: https://boringtechnology.club.
- Brian Goetz, "Java's evolution" — relevant for "do we even need a rewrite?" thinking.

## Exercises / Self-Check

1. Pick one of the migration posts (Discord, Cloudflare, etc.). Identify the *specific* metric that drove the decision. Could modern Go (1.22+) have changed it?
2. Take a Go service you maintain. Capture `/gc/cpu/total:cpu-seconds` and `/gc/pauses:seconds` over a week. Plot. Is GC actually your bottleneck?
3. Argue the case for *not* migrating an in-house Go cache service that hits 50 ms p99 GC spikes once an hour. What would you try first?
4. The Rust ecosystem has Tokio and Arrow. What are the Go equivalents? Where do they fall short?
5. Write a one-page "should we migrate?" decision template covering: workload type, current bottleneck, team expertise, ROI estimate, alternatives in current Go.
