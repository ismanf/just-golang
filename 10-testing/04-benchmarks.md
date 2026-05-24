# Benchmarks — `testing.B`, `b.Loop` (1.24+), `b.N` (legacy)

## TL;DR

A Go benchmark is `func BenchmarkXxx(b *testing.B)` in a `*_test.go` file. Until Go 1.24, you wrote `for i := 0; i < b.N; i++ { /* code */ }` and the framework increased `b.N` until the body ran for ≥1 second of wall time. Go **1.24 introduced `b.Loop()`**, a new iteration construct that fixes long-standing accuracy issues with `b.N` (compile-time constant folding, dead-code elimination, inlining inconsistencies). The recommended modern form is `for b.Loop() { /* code */ }`. Other essential APIs: `b.ResetTimer()` (drop setup time), `b.StopTimer/StartTimer` (exclude regions), `b.ReportAllocs()` (always-on alloc reporting), `b.ReportMetric(v, "unit/op")` (custom metrics), `b.RunParallel(pb *testing.PB)` (load-test–style concurrent benchmarks). Compare runs statistically with `benchstat` (`09-tooling/21-benchstat.md`).

## Mental Model

```
   BenchmarkX
        │
        ▼
   framework picks b.N (legacy) or runs b.Loop() (1.24+)
        │
        ▼
   measure wall time, allocations, bytes/op
        │
        ▼
   if total runtime < 1s, double N (legacy) or extend run (b.Loop)
        │
        ▼
   emit:
   BenchmarkX-8   1000000   1245 ns/op   120 B/op   3 allocs/op
                  │         │            │          │
                  │         │            │          └─ allocs per iter (with -benchmem)
                  │         │            └─ bytes per iter (with -benchmem)
                  │         └─ time per iter
                  └─ iterations executed
```

`-8` is `GOMAXPROCS`. Iterations are not "how many times your code ran in 1 second" — they're "how many we ran to reach 1 s of total measurement".

## Syntax & Basic Usage

```go
// Legacy form (pre-1.24 or for compatibility)
func BenchmarkOldStyle(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Do()
    }
}

// New form (1.24+)
func BenchmarkNewStyle(b *testing.B) {
    for b.Loop() {
        Do()
    }
}

// With setup
func BenchmarkWithSetup(b *testing.B) {
    data := generateInput()
    b.ResetTimer()                  // exclude setup
    for b.Loop() {
        Do(data)
    }
}

// Sub-benchmarks
func BenchmarkSizes(b *testing.B) {
    for _, n := range []int{1, 10, 100, 1000} {
        b.Run(fmt.Sprintf("N=%d", n), func(b *testing.B) {
            for b.Loop() {
                Process(n)
            }
        })
    }
}

// Parallel
func BenchmarkConcurrent(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Do()
        }
    })
}
```

Run:

```bash
$ go test -bench=. ./pkg
$ go test -bench=BenchmarkX -benchmem ./pkg
$ go test -bench=. -benchtime=10s ./pkg
$ go test -bench=. -benchtime=100x ./pkg     # exactly 100 iterations
$ go test -bench=. -count=10 ./pkg
$ go test -bench=. -cpu=1,4,8 ./pkg
$ go test -bench=. -cpuprofile=cpu.out ./pkg
```

## Deep Dive

### `b.Loop()` (Go 1.24+)

```go
func BenchmarkX(b *testing.B) {
    for b.Loop() {
        Do()
    }
}
```

What `b.Loop` does differently from `for i := 0; i < b.N; i++`:

1. **Hoists allocations** out of the loop body so they don't contaminate per-iteration cost.
2. **Prevents the compiler from constant-folding** the loop body (which could make `b.N` irrelevant).
3. **Coordinates with PGO and inlining** to keep optimization realistic across iterations.
4. **Allows custom iteration counts** that change between runs without breaking the benchmark.

The mechanic: `b.Loop()` returns `true` while iterations remain, decrements internally, and returns `false` once done. The framework treats the function as a closure over the loop count, which lets the compiler not over-optimize away the body.

For new code, **always use `b.Loop()`**. The legacy `for b.N` form works but the framework's accuracy guarantees are weaker.

### `b.N` (legacy)

```go
func BenchmarkOld(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Do()
    }
}
```

How the framework picks `b.N`:

1. Run with N=1 to estimate per-iter cost.
2. Estimate how many iterations would fill `benchtime` (default 1s).
3. Run with that N.
4. If too short, double and retry.

This had two problems: (1) the body's cost can vary with N if the compiler decides to vectorize differently, (2) `i := 0; i < b.N; i++` becomes a dead loop if the body has no observable side effects — the compiler may elide it. The fix has been to use the result of the body somehow:

```go
var sink int
func BenchmarkOld(b *testing.B) {
    for i := 0; i < b.N; i++ {
        sink = compute(i)
    }
}
```

`sink` is a package-level var the compiler can't prove dead. `b.Loop` makes this dance unnecessary.

### `b.ResetTimer()`, `b.StopTimer()`, `b.StartTimer()`

```go
func BenchmarkX(b *testing.B) {
    data := loadFixture()
    b.ResetTimer()                  // start clock now

    for b.Loop() {
        b.StopTimer()
        in := prepareInput()        // not measured
        b.StartTimer()

        out := Process(in)

        b.StopTimer()
        verify(out)                  // not measured
        b.StartTimer()
    }
}
```

`StopTimer`/`StartTimer` around setup/verification within the loop. Use sparingly — they have measurable cost (~100ns); over-using them distorts the very measurement they're meant to protect.

### `b.ReportAllocs()`

```go
func BenchmarkX(b *testing.B) {
    b.ReportAllocs()             // include B/op and allocs/op regardless of -benchmem
    for b.Loop() {
        Do()
    }
}
```

Useful when you want allocation counts reported even without the `-benchmem` flag (e.g., a CI that doesn't always pass `-benchmem`).

### `b.ReportMetric(value, "unit")`

```go
func BenchmarkServe(b *testing.B) {
    var bytes int64
    for b.Loop() {
        n, _ := w.Write(payload)
        bytes += int64(n)
    }
    b.ReportMetric(float64(bytes)/b.Elapsed().Seconds()/(1<<20), "MB/sec")
}
```

Output:

```
BenchmarkServe-8   ...   ...   1234 MB/sec
```

`benchstat` (`09-tooling/21-benchstat.md`) handles custom metrics natively. Use for domain-relevant numbers (throughput, p99 latency, etc.).

### `b.RunParallel`

```go
func BenchmarkConcurrent(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            handle(request)
        }
    })
}
```

Spawns `GOMAXPROCS` goroutines (override with `b.SetParallelism(N)`); each goroutine pulls iterations from a shared counter `pb`. Use to benchmark concurrent code (mutex contention, atomic ops, channel send/recv) the way it'd actually run.

`b.RunParallel`'s `for pb.Next()` is the parallel analog to `for b.Loop()` — same iteration semantics, distributed across goroutines.

### Sub-benchmarks

```go
func BenchmarkEncode(b *testing.B) {
    for _, size := range []int{1, 10, 100, 1000} {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            payload := make([]byte, size)
            b.SetBytes(int64(size))           // reports MB/s
            for b.Loop() {
                Encode(payload)
            }
        })
    }
}
```

Output:

```
BenchmarkEncode/size=1-8       2000000   683 ns/op   1.46 MB/s
BenchmarkEncode/size=10-8      1500000   821 ns/op  12.18 MB/s
BenchmarkEncode/size=100-8      800000  1455 ns/op  68.73 MB/s
BenchmarkEncode/size=1000-8     200000  6234 ns/op 160.42 MB/s
```

`b.SetBytes(N)` declares "this iteration processed N bytes" and the framework computes throughput.

### `-benchtime`

```bash
$ go test -bench=. -benchtime=1s ./pkg          # default
$ go test -bench=. -benchtime=10s ./pkg         # longer = more samples = less noise
$ go test -bench=. -benchtime=100x ./pkg        # exactly 100 iterations
$ go test -bench=. -benchtime=1000x ./pkg
```

Use `Ns` (seconds) for time-bounded, `Nx` (iterations) for fixed-count. Fixed-count is useful when you want every run to do the same amount of work (CPU-bound, no warmup).

### `-count`

```bash
$ go test -bench=. -benchmem -count=10 ./pkg | tee out.txt
```

Runs each benchmark 10 times. Needed for `benchstat` to compute variance. 10 is the de-facto minimum; more for noisy benchmarks.

### `-cpu`

```bash
$ go test -bench=. -cpu=1,4,8 ./pkg
```

Runs each benchmark at each `GOMAXPROCS` value. Output suffix `-N` shows the value. Useful for parallel benchmarks to see how they scale.

### Avoiding compiler optimizations

```go
var sink int

func BenchmarkX(b *testing.B) {
    for b.Loop() {
        sink = compute()       // result observable; compiler can't elide
    }
}
```

`b.Loop` reduces the need for `sink` tricks, but for purely side-effect-free code (math functions) you may still need it. The 2021 [Felix Geisendörfer post](https://github.com/golang/go/issues/27400) lists the patterns; modern code uses package-level vars or the `runtime.KeepAlive` family.

### `runtime.KeepAlive`

```go
func BenchmarkX(b *testing.B) {
    for b.Loop() {
        v := allocate()
        runtime.KeepAlive(v)   // tell compiler this value is live until here
    }
}
```

Prevents the compiler from freeing `v` before the loop iteration completes (which would defeat allocation measurement).

### Memory allocation reporting

```bash
$ go test -bench=. -benchmem ./pkg
BenchmarkX-8   1000000   1245 ns/op   120 B/op   3 allocs/op
```

`120 B/op` = bytes allocated per iteration; `3 allocs/op` = number of allocations. Both are usually more informative than raw ns/op for high-allocation workloads.

To reduce allocations:

- `sync.Pool` for short-lived objects.
- Pre-allocate slices/maps with `make([]T, 0, n)`.
- Use `*bytes.Buffer.Reset()` instead of allocating new buffers.

### Profiling alongside benchmarks

```bash
$ go test -bench=BenchmarkX -cpuprofile=cpu.out ./pkg
$ go tool pprof -http=:8080 cpu.out
```

Other profile flags: `-memprofile`, `-blockprofile`, `-mutexprofile`, `-trace`. See `09-tooling/12-go-tool-pprof.md`.

### `benchstat` workflow

```bash
$ go test -bench=. -benchmem -count=10 ./pkg > old.txt
$ # make changes
$ go test -bench=. -benchmem -count=10 ./pkg > new.txt
$ benchstat old.txt new.txt
```

Reports mean, ±σ, and a `p`-value for the difference. Detailed in `09-tooling/21-benchstat.md`.

### `b.Elapsed()` and `b.Loop` interaction

```go
func BenchmarkX(b *testing.B) {
    for b.Loop() {
        Do()
    }
    elapsed := b.Elapsed()
    b.ReportMetric(float64(work)/elapsed.Seconds(), "items/sec")
}
```

`b.Elapsed()` returns wall time spent in the measured portion (excluding `b.StopTimer` regions). Use for custom throughput metrics.

### What `b.Loop` doesn't do

- Doesn't run the body in a separate goroutine (use `b.RunParallel` for that).
- Doesn't warm up the function call (the first iteration may be slower).
- Doesn't compensate for thermal throttling, GC pauses, or kernel scheduling.

For high-precision benchmarks, run on a quiet machine, pin to a CPU (`taskset` on Linux), disable Turbo Boost.

### Common benchmark patterns to avoid

```go
// BAD: body has no observable effect; compiler may elide
func BenchmarkBad(b *testing.B) {
    for b.Loop() {
        _ = expensive()
    }
}
```

```go
// GOOD: pin the result so compiler must compute it
var sink int
func BenchmarkGood(b *testing.B) {
    var x int
    for b.Loop() {
        x = expensive()
    }
    sink = x          // observed at the end
}
```

`b.Loop` mostly handles this, but for pure functions, belt + braces.

### Comparing approaches

| Form                    | Pros                                | Cons                                |
|-------------------------|-------------------------------------|-------------------------------------|
| `for i := 0; i < b.N; i++` | Works on all Go versions          | Compiler may over-optimize; manual sink needed |
| `for b.Loop()`           | Modern; better accuracy            | Requires Go 1.24+                   |
| `b.RunParallel`          | Tests concurrent code              | Different concurrency model than prod |

Use `b.Loop` for new code; keep `b.N` only for backward compatibility.

## Standard Library Hooks

- `testing.B` — all the methods.
- `testing.B.Loop` (1.24+).
- `testing.B.ResetTimer`, `StopTimer`, `StartTimer`.
- `testing.B.ReportMetric`, `ReportAllocs`, `SetBytes`.
- `testing.B.RunParallel`, `SetParallelism`.
- `testing.B.Elapsed` (1.20+).
- `testing.PB` — parallel-iteration handle.
- `runtime.KeepAlive` — prevent optimization.
- `runtime/debug.SetGCPercent` — control GC for benchmark stability.
- `runtime/pprof` — programmatic profiling alongside benchmarks.

## Real-World Patterns

### 1. Modern microbenchmark

```go
func BenchmarkHash(b *testing.B) {
    data := []byte("hello world")
    b.ResetTimer()
    for b.Loop() {
        sha256.Sum256(data)
    }
}
```

### 2. Sized sub-benchmarks

```go
func BenchmarkEncodeSized(b *testing.B) {
    for _, n := range []int{1 << 8, 1 << 12, 1 << 16, 1 << 20} {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := make([]byte, n)
            b.SetBytes(int64(n))
            for b.Loop() {
                Encode(data)
            }
        })
    }
}
```

### 3. Allocation profile

```go
func BenchmarkAlloc(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        s := make([]int, 1000)
        sink = s[len(s)-1]
    }
}
```

### 4. Parallel handler

```go
func BenchmarkServer(b *testing.B) {
    h := http.HandlerFunc(myHandler)
    s := httptest.NewServer(h)
    defer s.Close()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            resp, _ := http.Get(s.URL + "/")
            resp.Body.Close()
        }
    })
}
```

### 5. Compare two implementations

```go
func BenchmarkSort(b *testing.B) {
    data := generateData(10000)
    b.Run("Stable", func(b *testing.B) {
        for b.Loop() {
            c := append([]int(nil), data...)
            sort.SliceStable(c, func(i, j int) bool { return c[i] < c[j] })
        }
    })
    b.Run("Unstable", func(b *testing.B) {
        for b.Loop() {
            c := append([]int(nil), data...)
            sort.Slice(c, func(i, j int) bool { return c[i] < c[j] })
        }
    })
}
```

### 6. Custom metric

```go
func BenchmarkThroughput(b *testing.B) {
    var bytes int64
    for b.Loop() {
        n := Process()
        bytes += int64(n)
    }
    b.ReportMetric(float64(bytes)/b.Elapsed().Seconds()/(1<<20), "MB/sec")
}
```

### 7. Benchstat in CI

```yaml
- run: go test -bench=. -benchmem -count=10 ./pkg > new.txt
- run: |
    git stash
    git checkout main
    go test -bench=. -benchmem -count=10 ./pkg > old.txt
    git checkout -; git stash pop
- run: benchstat old.txt new.txt > result.txt
- run: cat result.txt
```

## Anti-Patterns & Gotchas

**Benchmark body has no observable side effect.** Compiler may elide. Use `b.Loop` (1.24+) or a package-level sink.

**Setup inside the loop.** Costs include setup. `b.ResetTimer` after setup; `b.StopTimer`/`StartTimer` around per-iter setup.

**Comparing one-run numbers.** Variance is too high. `-count=10` minimum; use `benchstat`.

**Benchmarking on a noisy machine.** Browser, video calls, indexing service — all distort. Quiet the box.

**Running benchmarks without `-benchmem`.** You see ns/op only; allocation behavior hidden.

**Long benchmarks with `-benchtime=1s` (default).** Slow benchmarks complete few iterations; numbers are noisy. Increase: `-benchtime=10s`.

**Using `b.N` directly in computations.** `for i := 0; i < b.N; i++ { sum += i }` — `sum` becomes O(N²); your benchmark doesn't measure what you think.

**Benchmarking pure functions without warmup.** First call may pay JIT-like costs (icache, branch predictor cold). Modern Go has no JIT, but cold-cache effects exist.

**Mixing `b.RunParallel` with `b.SetBytes`.** Bytes are measured globally; throughput numbers can mislead. Verify with a known-good baseline.

**Skipping `runtime.KeepAlive` for allocation-tracked benchmarks.** Compiler may free objects mid-iteration; allocation counts drop spuriously.

**Comparing benchmarks across Go versions.** Compiler changes affect timing; isolate.

**Benchmarking on a battery-powered laptop.** Thermal throttling distorts. Plug in.

**Forgetting that `b.Loop` requires Go 1.24+.** Older toolchains: compile error.

## Performance Notes

- Benchmark overhead per iteration: <10 ns (timer + counter).
- `b.RunParallel` overhead: ~50 ns per goroutine sync.
- `-benchmem` doesn't slow the benchmark much; ~5%.
- `-cpuprofile` overhead: ~5%.
- `-trace`: low overhead during capture but big files.

Reasonable benchmarks target 100ns – 1µs per iteration for high precision; longer iterations dilute timer noise but lose to fewer samples.

## How Big Companies Use It

- **Google** uses benchmarks extensively in stdlib; the `runtime`, `encoding/json`, and `crypto` packages have hundreds: https://github.com/golang/go.
- **Uber** uses benchmarks in `zap` (logger) for sub-microsecond per-call costs: https://github.com/uber-go/zap.
- **Cloudflare** uses benchmarks for their HTTP/3 implementation; benchstat in CI: https://blog.cloudflare.com.
- **CockroachDB** uses benchmarks gated by benchstat to catch regressions per PR: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses benchmarks in Vault's crypto paths: https://github.com/hashicorp/vault.
- **Tailscale** uses benchmarks for `wireguard-go`: https://tailscale.com/blog.
- **The Go team** uses `b.Loop` aggressively since 1.24; runtime + compiler PRs include benchmarks: https://github.com/golang/go.

## Source Code References

Pinned to `go1.26`.

- `testing.B`: [`src/testing/benchmark.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/benchmark.go).
- `b.Loop`: same file (search `func (b *B) Loop`).
- `b.RunParallel`, `PB`: same file (search `RunParallel`, `PB`).
- `b.ReportMetric`, `ReportAllocs`: same file.
- Benchmark iteration logic: [`src/testing/benchmark.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/benchmark.go) (search `launch`).
- benchstat: [`golang.org/x/perf/cmd/benchstat`](https://github.com/golang/perf/tree/master/cmd/benchstat).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Benchmark in Go 1.24" (Michael Knyszek): https://go.dev/blog/testing-b-loop.
- "How to write benchmarks in Go" (Dave Cheney): https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go.
- "Reliable benchmarking" (Brendan Gregg): https://www.brendangregg.com/blog/2018-02-09/kpti-kaiser-meltdown-performance.html.
- "Sub-benchmarks" (Marcel van Lohuizen): https://go.dev/blog/subtests.
- "Compiler optimizations and benchmarks" (Filippo Valsorda): https://words.filippo.io.
- "Statistically rigorous Java performance evaluation" (Georges et al.): https://dri.es/files/oopsla07-georges.pdf.

## Exercises / Self-Check

1. Rewrite a `for i := 0; i < b.N; i++` benchmark using `for b.Loop()`. Compare ns/op (with `benchstat`).
2. Add `b.SetBytes(N)` and observe the throughput column appear.
3. Write a `b.RunParallel` benchmark for a `sync.Mutex`-protected operation. Compare to single-threaded with `-cpu=1,8`.
4. Use `b.ReportMetric` to add a custom "items/sec" column to a benchmark.
5. Profile a benchmark with `-cpuprofile`. Identify the hottest function in `go tool pprof`.
