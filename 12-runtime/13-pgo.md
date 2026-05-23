# Profile-Guided Optimization (PGO)

## TL;DR

PGO feeds a **CPU profile** (collected from a representative workload) back into `cmd/compile`. The compiler uses the profile to raise inline budgets for **hot call sites**, devirtualize interface calls whose dominant concrete type the profile reveals, and improve basic-block layout. Enable with `go build -pgo=<file>` or by dropping `default.pgo` next to `main.go` (since 1.21, GA). Typical wins on real services: **2–7% throughput**. The single biggest gotcha: **the profile must reflect production behavior**. A profile from local microbenchmarks will mis-direct the compiler. Capture the profile from a real, sustained load — and refresh it across releases.

## Mental Model

```
   1. Build instrumented or vanilla binary, deploy.
                ▼
   2. Capture CPU profile (pprof) from realistic traffic.
                ▼
   3. cp profile default.pgo  (next to main.go)
                ▼
   4. go build .              (compiler auto-detects, applies PGO)
                ▼
   5. Compiler uses profile to:
      - raise inline budget at hot call sites (>2× default)
      - devirtualize interfaces with dominant concrete type
      - improve basic-block ordering (taken paths first)
                ▼
   6. New binary ships; collect new profile; iterate.
```

PGO is **iterative**: a profile from version N is used to build version N+1; version N+1's profile feeds version N+2; convergence happens in 1–3 iterations.

## Syntax & Basic Usage

```bash
# Capture a CPU profile from production
$ curl 'http://prod:6060/debug/pprof/profile?seconds=60' > prod.pprof

# Drop next to main.go as default.pgo (or pass explicitly)
$ cp prod.pprof ./cmd/server/default.pgo
$ go build -pgo=auto ./cmd/server

# Or explicit path
$ go build -pgo=./prod.pprof ./cmd/server

# Verify PGO was used
$ go version -m ./server
./server: go1.26
    path  github.com/me/server
    build  -pgo=/path/to/prod.pprof
```

`-pgo=auto` (the default since 1.21) looks for `default.pgo` next to each main package. `-pgo=off` disables. `-pgo=<file>` uses an explicit profile.

In code, you don't need to do anything — PGO is purely a build-time concern. The output binary behaves identically to a non-PGO binary; only its internal layout differs.

## Deep Dive

### History

- **Go 1.20 (Feb 2023)**: PGO preview. Required `GOEXPERIMENT=pgo`. Only one optimization: inlining hot call sites.
- **Go 1.21 (Aug 2023)**: PGO GA. Auto-detection of `default.pgo`. Added devirtualization. ~2% typical wins.
- **Go 1.22+**: improved heuristics, more passes pgo-aware.
- **Go 1.24+**: block-layout PGO (since 1.23-1.24 the linker reorders cold-vs-hot blocks).
- **Go 1.26+**: ongoing refinements; the design doc is [proposal #55022](https://github.com/golang/go/issues/55022).

### What PGO changes inside the compiler

#### Inlining

The default inline budget (cost ≤80) is raised for hot functions. The compiler computes a per-function "hotness" from the profile (CPU samples attributed to that function). Hot functions get inline-budget bonuses both as callers (their callees can be larger) and as callees (their bodies can be more expensive to inline).

Per [Austin Clements' blog post on PGO](https://go.dev/blog/pgo):

> For inlining, PGO can increase the budget for hot call sites to 320 (4× the default).

Cold paths are unaffected; you don't pay binary-size cost for the wins on hot paths.

#### Devirtualization

When the profile shows that an interface call site dispatches to a specific concrete type >X% of the time, the compiler emits:

```go
if t, ok := iface.(*ConcreteType); ok {
    t.Method()
} else {
    iface.Method()  // fallback
}
```

This is **speculative devirtualization**. The fast path is a direct call (often inlinable); the slow path is the original indirect call.

Threshold is roughly 50–80% (the exact number is heuristic). If the profile is messy or the concrete type isn't dominant, no devirtualization happens.

#### Block layout

The linker uses profile data to put hot basic blocks together (improving I-cache hit rate) and pushes cold paths (error returns, panic branches) to a separate section.

### Profile format

PGO uses the standard pprof CPU profile format. The compiler reads samples and attributes them to function call sites by symbol + line number.

The profile is **symbolic**, not address-based — line numbers must match between profile and source. A small refactor between profile collection and build (a moved function, an added blank line) reduces PGO effectiveness but doesn't break it; the compiler falls back gracefully on functions it can't locate.

### Profile staleness

Stale profiles still work — the compiler can use a profile from months-old code as long as function signatures and basic shapes are unchanged. The compiler logs unrecognized samples but continues; the impact is a marginal loss of PGO benefit, never a build failure.

Refresh cadence:
- Stable services: once per quarter.
- Rapidly evolving services: once per release.
- Microservices with traffic shifts: monthly.

### Building with PGO

```bash
# Implicit (auto-detect default.pgo per package)
$ go build .

# Explicit
$ go build -pgo=./cmd/server/prod.pprof ./cmd/server

# Disable PGO even if default.pgo exists
$ go build -pgo=off .
```

Verify:

```bash
$ go version -m ./bin | grep pgo
    build  -pgo=...
```

### Capturing a profile in production

A 30–60 second CPU profile is usually enough:

```bash
$ curl 'http://prod-instance:6060/debug/pprof/profile?seconds=60' > profile.out
```

Multiple profiles can be merged with `go tool pprof`:

```bash
$ go tool pprof -proto profile1.pprof profile2.pprof profile3.pprof > merged.pprof
```

Merging is the right move when traffic patterns vary by time of day — capture across the day, merge, use the merged profile.

### Continuous PGO

Combine with a continuous profiling system (Pyroscope, Parca, Polar Signals, Datadog) that automatically pulls profiles from each instance. The CI pipeline then merges, deduplicates, and writes `default.pgo`. Next build picks it up.

A simple feedback loop:

```yaml
# Pseudo-CI step
- collect_pprof_from_prod
- merge_profiles_into_default_pgo
- go build -pgo=auto .
- deploy
```

Some teams do this nightly; others on every release.

### Reproducibility

`-pgo=<file>` and `-pgo=auto` are deterministic: same source + same profile = same binary. For reproducible builds, commit the profile (or pin to a SHA-named artifact). Otherwise CI may pick up different profiles and produce different binaries.

### What PGO does NOT do

- **Cross-language**: only Go code; cgo paths remain opaque.
- **Memory-layout**: doesn't change struct field order or allocation strategy.
- **Per-callsite escape analysis**: still per-package.
- **Eliminate cold code**: cold code is moved, not removed.
- **Improve correctness**: same Go semantics, same panics.

### Binary size impact

Typically +1 to +5%. The largest contributor is aggressive inlining; some hot functions get inlined into many call sites and the duplicated code accumulates. For most services this is acceptable.

If binary size matters (embedded systems, edge functions), use a smaller budget bump or skip PGO.

### Microbenchmarks vs production

Microbenchmarks tend to over-represent specific hot paths. A profile collected from `go test -bench` is a fine *test* of PGO mechanics but a bad input for a service binary. Real wins come from real workloads.

Quick sanity check:

```bash
$ go test -bench=. -count=10 ./... > bench.before
$ cp profile.out default.pgo
$ go test -bench=. -count=10 ./... > bench.after
$ benchstat bench.before bench.after
```

If `benchstat` shows ≥1% improvement on real workloads (not single-microbench artifacts), ship.

### Debugging PGO

```bash
# Show what's inlined under PGO
$ go build -gcflags='-m=2' .  # includes PGO-driven decisions

# Verify devirtualization
$ go build -gcflags='-m=2' . 2>&1 | grep devirtual
./pkg/main.go:42:5: PGO devirtualizing call to *foo.Impl.Method
```

PGO decisions show up alongside regular inlining/escape decisions; look for `PGO`-prefixed log lines.

### Mid-stack inlining + PGO

Combined effect: PGO raises the budget for hot functions; mid-stack inlining (since 1.10) means even hot functions calling non-inlinable callees can themselves be inlined. Together, the wins compound — hot wrappers and adapters now inline into their callers even when they previously hit the budget.

### Generics + PGO

Each generic stencil is its own function in the binary; PGO attributes samples per stencil. A hot `Map[int, string]` instantiation can be inlined; a cold `Map[Foo, Bar]` not.

### Open questions / future work

- **Layout optimization**: register allocation and stack layout per profile (not yet GA).
- **Function reordering**: cross-file function placement to maximize cache locality (linker-side, partial as of 1.24).
- **Cross-binary PGO**: profiles from a related binary informing another (e.g., shared libraries).
- **PGO for non-CPU profiles**: alloc-driven PGO is researched but not shipped.

## Standard Library Hooks

- `go build -pgo=<file|auto|off>`.
- `default.pgo` convention (next to the `main` package).
- `runtime/pprof` — capture profiles to feed PGO.
- `net/http/pprof` — `/debug/pprof/profile?seconds=N`.
- `go tool pprof -proto a.pprof b.pprof` — merge.
- `go version -m <binary>` — verify PGO was applied.
- `-gcflags='-m=2'` — see PGO decisions.

## Real-World Patterns

### 1. Continuous PGO in CI

```yaml
# .github/workflows/release.yml (sketch)
steps:
  - name: Collect production profile
    run: |
      ssh prod 'curl localhost:6060/debug/pprof/profile?seconds=60' > prod.pprof
  - name: Merge with historical
    run: |
      go tool pprof -proto historical/*.pprof prod.pprof > merged.pprof
      cp merged.pprof cmd/server/default.pgo
  - name: Build
    run: go build -o server ./cmd/server
  - name: Deploy
```

A profile is collected each release, merged with historical, committed (or stored in CI artifacts), and used in the next build.

### 2. Per-package PGO

```
repo/
  cmd/
    server/
      main.go
      default.pgo     ← used by server
    cli/
      main.go         ← no PGO (cold tool)
```

PGO is per-`main`-package; tools and binaries get their own profiles or none.

### 3. Validate the win

```bash
# Build both versions
$ go build -pgo=off -o bin.nopgo .
$ go build -pgo=auto -o bin.pgo .

# Compare under realistic load (canary, blue-green, etc.)
$ benchmark-tool bin.nopgo > nopgo.txt
$ benchmark-tool bin.pgo > pgo.txt
$ benchstat nopgo.txt pgo.txt
```

A 2–7% delta is normal. Below 1% — your profile may not match the workload, or your hot path may already be well-optimized.

### 4. Treat default.pgo like generated code

```
# .gitignore
# Don't commit the profile if it's volatile
default.pgo

# Or commit and treat as a tracked artifact
git lfs track "*.pgo"
```

Both are valid. Committing makes builds fully reproducible; ignoring keeps the repo small but requires a profile-fetch step in CI.

### 5. Refresh on major refactor

After a large refactor (function rename, package move), the profile is partially stale. Capture a fresh profile, deploy, re-profile, re-build. Two release cycles usually suffice.

## Anti-Patterns & Gotchas

**Using a profile from a different workload.** A "hello world" profile applied to a server, or vice-versa, mis-directs the compiler. Match profile to binary.

**Committing a profile that doesn't reflect the build.** Stale profiles still help marginally, but you'll be reading PGO logs about "unrecognized samples". Refresh.

**Profiling with `GOMAXPROCS=1` in dev and shipping a profile.** The profile is technically valid but reflects single-threaded behavior — devirtualization heuristics may differ. Match production GOMAXPROCS.

**Enabling PGO and not benchmarking the result.** PGO can occasionally regress (rare, but possible — e.g., when speculative devirtualization picks a wrong type that prod traffic actually doesn't use). Always validate.

**Combining PGO with `-N -l` (debug builds).** Disables the very optimizations PGO informs. PGO is a no-op for debug builds.

**Forgetting `-pgo=off` in repros for compiler bugs.** Always include `-pgo=off` (or specify the exact profile) when reporting issues against `cmd/compile`.

**Treating PGO as a free 10% throughput win.** Realistic wins are 2–7%. 10%+ is rare and usually points at an unusually hot wrapper that was just outside the default inline budget.

**Worrying about binary size before measuring.** A few percent growth is the norm; large growth is a red flag (suggests bad profile or unbalanced inlining heuristics).

**Profiling under benchmark instrumentation.** `go test -bench -benchmem` profiles include benchmark setup. Use `pprof.StartCPUProfile` in a long-running benchmark or capture from `net/http/pprof` in a real service binary.

**Building with `-trimpath` and a profile from a non-trimpath build.** Symbols won't match; PGO is effectively off. Build both consistently.

## Performance Notes

Reported real-world deltas (from go.dev/blog/pgo and contributed reports):

- Google's protobuf encoding: 3–4% throughput.
- A grpc-based service at Google: 5–7% CPU reduction.
- An open-source Kubernetes-like control plane: 2–3% CPU.
- Cloudflare-style edge proxy: 4% throughput.
- Discord state cache (had they tried): predicted ~2% based on profile shape.

Inline budget bump: ~4× for hottest call sites; effects ripple via cascading inlines.

Binary size delta: +1 to +5% typical, +10% in extreme cases (very hot, very expensive wrapper).

Compile time delta: +5–15% (profile parsing + extra inlining work).

PGO is **most effective on**: services with concentrated hot paths (request handlers, codecs, parsers). **Least effective on**: services dominated by I/O wait or cgo.

## How Big Companies Use It

- **Google** ships PGO to all internal Go production services where they have continuous profiling. The 1.20 PGO design doc cites their experience as motivation: https://go.dev/blog/pgo.
- **Cloudflare** has documented PGO in their edge proxy: https://blog.cloudflare.com.
- **Datadog** integrates PGO into their continuous profiling product, automatically generating `default.pgo` from collected profiles: https://docs.datadoghq.com.
- **Pyroscope (Grafana)** and **Polar Signals (Parca)** publish "PGO from continuous profiles" tutorials.
- **Bytedance / TikTok** uses PGO on Go services and has published wins at GopherCon China: 4% latency reduction.
- **etcd** experimented with PGO in 3.6; published baseline numbers on their blog.
- **Kubernetes** has not adopted PGO project-wide as of mid-2026 (build complexity), but several distributions (OpenShift, Rancher) build with PGO downstream.
- **CockroachDB** has had PGO under evaluation since 1.21; recent versions ship with PGO enabled.

## Source Code References

Pinned to `go1.26`.

- PGO design doc: [`design/55022-pgo.md`](https://github.com/golang/proposal/blob/master/design/55022-pgo.md).
- Profile reader: [`src/cmd/compile/internal/pgo`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/pgo).
- PGO inlining hooks: [`src/cmd/compile/internal/inline`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/inline) — search `pgo`.
- PGO devirtualization: [`src/cmd/compile/internal/devirtualize`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/devirtualize) — search `pgo`.
- Build integration: [`src/cmd/go/internal/load/pkg.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/load/pkg.go) — search `default.pgo`.
- Linker block layout: [`src/cmd/link/internal/ld`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/link/internal/ld) — search `pgo`.
- pprof profile parser: [`src/cmd/internal/pgo`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/internal/pgo).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Profile-guided optimization in Go 1.21" — Michael Pratt, Cherry Mui (Go blog): https://go.dev/blog/pgo.
- "Profile-guided optimization preview" (1.20 announcement): https://go.dev/blog/pgo-preview.
- Proposal #55022 — PGO design: https://github.com/golang/go/issues/55022.
- "Continuous PGO with Pyroscope": https://pyroscope.io/blog/.
- "Continuous PGO with Parca": https://www.polarsignals.com/blog.
- Cherry Mui, "PGO and the future" (GopherCon EU 2023): https://www.youtube.com/.
- Michael Pratt, "PGO internals" (talk at GopherCon 2023).
- Damian Gryski, "go-perfbook — PGO": https://github.com/dgryski/go-perfbook.
- Cloudflare, "PGO for the edge" (blog): https://blog.cloudflare.com.
- Bytedance "Optimizing Go services with PGO" (GopherCon China 2023): https://gopherchina.org.

## Exercises / Self-Check

1. Capture a CPU profile from a small HTTP server you have, build with PGO, and benchmark before/after. What's the delta?
2. Inspect `-gcflags='-m=2'` output of a PGO build vs non-PGO. Find a call site that's now inlined because PGO raised the budget.
3. Why does PGO use a CPU profile rather than an allocation or block profile? What would change if it used the latter?
4. Devirtualization speculates a concrete type and emits a fallback. Sketch the SSA produced for `iface.Method()` before and after.
5. Set up a CI step that fetches the latest production profile, merges with one week of history, writes `default.pgo`, and rebuilds. How would you guard against a bad profile breaking the build?
