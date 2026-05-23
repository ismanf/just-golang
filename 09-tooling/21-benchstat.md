# `benchstat` — Statistically Comparing Benchmarks

## TL;DR

**`benchstat`** (by Russ Cox, in `golang.org/x/perf`) reads `go test -bench=... -count=N` output and answers the single question that matters for performance regressions: *is the difference between these two runs statistically significant, or could it be noise?*. Single-run benchmark numbers ("`old: 100 ns/op`, `new: 90 ns/op` — 10% faster!") are nearly useless without variance information. `benchstat` computes mean, ±standard deviation, and runs a Mann-Whitney U test (the default in 0.0.0-20231012+; prior versions used the Welch t-test) to report a `p`-value. If `p < 0.05`, the difference is unlikely to be noise. Workflow: run `-count=10` on the baseline, save; run `-count=10` on the change, save; `benchstat old.txt new.txt` shows the diff with confidence.

## Mental Model

```
   git checkout main
   go test -bench=. -count=10 ./pkg > old.txt
   git checkout feature
   go test -bench=. -count=10 ./pkg > new.txt
   benchstat old.txt new.txt
        │
        ▼
   For each benchmark:
        ├─ compute mean(old), mean(new)
        ├─ compute σ(old), σ(new)
        ├─ run Mann-Whitney U test → p-value
        └─ format:    old time/op    new time/op    delta    p-value
```

The default 10 runs balance time and statistical power. More runs = tighter intervals; you typically need 6+ for the U test to yield meaningful `p < 0.05` results.

## Syntax & Basic Usage

```bash
$ go install golang.org/x/perf/cmd/benchstat@latest

# Generate baselines
$ go test -bench=. -benchmem -count=10 ./pkg | tee old.txt

# Make a change, run again
$ go test -bench=. -benchmem -count=10 ./pkg | tee new.txt

# Compare
$ benchstat old.txt new.txt
$ benchstat -confidence=0.95 old.txt new.txt
$ benchstat -delta-test=utest old.txt new.txt          # default
$ benchstat -delta-test=ttest old.txt new.txt          # Welch t-test
$ benchstat -delta-test=none old.txt new.txt           # no test, just diff
$ benchstat -row=/lat -col=name old.txt new.txt        # custom row/column
$ benchstat -format=csv old.txt new.txt
$ benchstat old.txt new.txt 'experimental.txt'         # 3-way compare
```

## Deep Dive

### Reading benchstat output

```
$ benchstat old.txt new.txt
goos: darwin
goarch: arm64
pkg: github.com/me/pkg
                  │   old.txt   │              new.txt              │
                  │   sec/op    │   sec/op     vs base              │
Encode/small-8      125.3n ± 2%   118.7n ± 1%  -5.27% (p=0.000 n=10)
Encode/medium-8     1.213µ ± 1%   1.165µ ± 2%  -3.96% (p=0.001 n=10)
Encode/large-8      12.45µ ± 3%   12.41µ ± 5%       ~ (p=0.838 n=10)
geomean             1.235µ        1.183µ       -4.21%
```

Columns:

- **Benchmark name** (slash-separated for sub-benchmarks, `-N` for `GOMAXPROCS`).
- **Per-file values** (here `old.txt`, `new.txt`): mean ± standard deviation as a percentage.
- **Delta column** (vs. base): percent change, `p`-value, and sample count `n`.

Interpretation:

- `~ (p=0.838)` — no significant difference.
- `-5.27% (p=0.000)` — 5.27% faster, highly significant.
- `+12% (p=0.04)` — 12% slower, marginally significant.

`geomean` (geometric mean) gives an overall sense; for performance work, focus on individual benchmark deltas.

### Why `-count=10`

The Mann-Whitney U test requires roughly 6+ samples per group to have power. 10 is a comfortable default. Fewer samples → broader confidence intervals → fewer "significant" results.

For high-variance benchmarks (e.g., GC-heavy), more samples help. `-count=20` or `-count=50` for noisy workloads.

### The Mann-Whitney U test

A non-parametric test that compares two samples without assuming normal distribution. Robust to:

- Outliers (one slow run won't dominate).
- Skewed distributions (often the case for benchmarks where most runs cluster but a few are GC-slow).
- Small sample sizes.

Welch's t-test (pre-default) assumes normality; benchstat moved to U test for these robustness reasons.

`benchstat -delta-test=ttest` brings back Welch if you want it (e.g., for compatibility with old workflows).

### Multiple metrics

```bash
$ go test -bench=. -benchmem -count=10 ./pkg > out.txt
```

`-benchmem` adds B/op and allocs/op. benchstat renders one table per metric:

```
sec/op:
Encode/small-8   125.3n ± 2%   118.7n ± 1%  -5.27% ...

B/op:
Encode/small-8   64.00 ± 0%    64.00 ± 0%   ~       (p=1.000 n=10)

allocs/op:
Encode/small-8   1.000 ± 0%    1.000 ± 0%   ~       (p=1.000 n=10)
```

`-row=/lat,/B,/op` etc. (regex matching) selects which metrics to print.

### Three-way comparisons

```bash
$ benchstat baseline.txt with-cache.txt with-cache-and-pool.txt
```

Three columns; delta is vs. the first file. Useful for "show me how each optimization step contributed".

### Filtering benchmarks

```bash
$ go test -bench=BenchmarkEncode -count=10 ./pkg > out.txt
```

Only the named benchmark. Combine with `-run=^$` to skip unit tests in mixed packages:

```bash
$ go test -run=^$ -bench=BenchmarkEncode -count=10 ./pkg > out.txt
```

### Sub-benchmarks

```go
func BenchmarkEncode(b *testing.B) {
    for _, size := range []int{1, 10, 100} {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ { encode(size) }
        })
    }
}
```

Produces `BenchmarkEncode/size=1-8`, `/size=10-8`, `/size=100-8`. benchstat handles each separately.

### `b.Loop()` (since 1.24)

```go
func BenchmarkFoo(b *testing.B) {
    for b.Loop() {
        // ...
    }
}
```

Replaces `for i := 0; i < b.N; i++`. Auto-adjusts iteration count; integrates with PGO and inlining; output looks the same to benchstat.

### Custom rows/columns

```bash
$ benchstat -row='/throughput' -col='name' new.txt old.txt
```

The `-row=` and `-col=` patterns reshape the table. Useful for many-benchmark, many-version comparisons.

### Filter benchmark names

```bash
$ benchstat -ignore=pkg -filter='.unit:ns/op' old.txt new.txt
```

`-filter` is a query language for picking subsets. `-ignore` strips fields you don't want as table cells.

### Output formats

```bash
$ benchstat old.txt new.txt                  # default: ASCII table
$ benchstat -format=csv old.txt new.txt      # for spreadsheets
$ benchstat -format=html old.txt new.txt     # render to HTML
$ benchstat -table=name -row=/lat old.txt new.txt
```

CSV/HTML are useful for dashboards.

### Reading the noise (`± X%`)

Standard deviation as a percentage of the mean. Lower is better:

- **< 1%**: very stable; CPU-bound, well-warmed.
- **1–5%**: normal for warmed-up benchmarks.
- **> 10%**: noisy; consider more iterations or controlling external factors (GC, thermal throttling, other processes).

If old.txt and new.txt show vastly different noise levels, something changed in benchmark *stability*, not just speed.

### Reducing noise

- `b.ReportAllocs()` if not using `-benchmem`.
- `b.ResetTimer()` after setup.
- Run on a quiet machine (close browser tabs).
- `taskset` (Linux) or `cpuset` to pin to a CPU.
- Disable Turbo Boost (Linux: `echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo`).
- Set `GOMAXPROCS` explicitly.
- Run multiple `-count=10` batches and concat with `cat`.

### Combining batches

```bash
$ cat batch1.txt batch2.txt batch3.txt > combined.txt
$ benchstat combined.txt
```

Concatenating raw `go test -bench` output is fine — benchstat parses each `Benchmark` line and aggregates.

### CI gating

```yaml
- run: go test -bench=. -benchmem -count=10 ./pkg > new.txt
- run: |
    git stash
    git checkout main
    go test -bench=. -benchmem -count=10 ./pkg > old.txt
    git checkout -
    git stash pop
- run: benchstat old.txt new.txt > result.txt
- run: |
    if grep -qE '\+[0-9]+\.[0-9]+%.*\(p=0\.0[0-4][0-9]?' result.txt; then
      echo "performance regression"; exit 1
    fi
```

Crude; better tools (e.g., GitHub bot integrations) exist for this.

### `cont` benchmark (pre-1.24 only)

Older Go versions had a `BenchmarkXxx-N` suffix automatic insertion based on `GOMAXPROCS`. 1.24 changed semantics; benchstat handles both formats.

### Statistical power and N

- `-count=5`: roughly the minimum; many comparisons will report `~`.
- `-count=10`: standard.
- `-count=20`: tighter intervals, more "significant" results.
- `-count=50`: usually overkill; useful for very noisy benchmarks.

`p < 0.05` is the conventional cutoff but benchstat just reports the value; you decide.

## Standard Library Hooks

- `testing.B` — benchmark API.
- `b.Loop()` (1.24+) — new iteration loop.
- `b.ReportAllocs()` — alloc reporting.
- `b.ReportMetric()` — custom metrics.
- `golang.org/x/perf/benchstat` — the library.
- `golang.org/x/perf/benchproc` — benchmark name parser.
- `golang.org/x/perf/benchmath` — statistical functions.

Custom metric:

```go
func BenchmarkX(b *testing.B) {
    for b.Loop() {
        // ...
    }
    b.ReportMetric(throughput, "ops/sec")
}
```

benchstat picks up the custom metric automatically.

## Real-World Patterns

### 1. Standard A/B

```bash
$ git checkout main; go test -bench=. -count=10 ./pkg > old.txt
$ git checkout feature; go test -bench=. -count=10 ./pkg > new.txt
$ benchstat old.txt new.txt
```

### 2. Three-way

```bash
$ benchstat baseline.txt cache.txt cache-pool.txt
```

### 3. Per-CPU comparison

```bash
$ go test -bench=. -cpu=1,4,8 -count=10 ./pkg > out.txt
$ benchstat out.txt
```

Sub-benchmarks include `-N` suffix for each `GOMAXPROCS`; benchstat shows each.

### 4. Custom metric — throughput

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

```
$ go test -bench=BenchmarkServe -count=10 ./pkg
BenchmarkServe-8   ...   ...   1234 MB/sec
```

### 5. Long bench session

```bash
$ for i in $(seq 1 5); do
    go test -bench=. -count=10 ./pkg >> all.txt
done
$ benchstat all.txt        # 50 samples per benchmark
```

### 6. Save trends over commits

```bash
$ git log --oneline | while read commit msg; do
    git checkout "$commit"
    go test -bench=. -count=10 ./pkg > "perf-$commit.txt"
done
$ benchstat perf-*.txt
```

Useful for "when did performance regress?".

### 7. Visual dashboards

```bash
$ benchstat -format=csv old.txt new.txt | python plot.py
```

Or use `benchviz` / `benchsuite` / Grafana datasources.

## Anti-Patterns & Gotchas

**Comparing a single `-count=1` run.** No variance; the delta is meaningless. Always `-count=10`+.

**Running benchmarks on a battery-powered laptop.** Thermal throttling distorts timings. Plug in; close apps.

**Running benchmarks immediately after waking from sleep.** Frequency scaling hasn't settled. Warm up first.

**Comparing across machines.** Different CPUs, different baselines. Always same machine for both runs.

**Trusting `geomean` as the only metric.** It can mask outliers; one benchmark that doubled time and one that halved cancel out. Look at individual benchmarks.

**Ignoring `± X%` noise on findings.** A `-50% (p=0.05)` change with `±40%` noise is barely significant. Re-run with `-count=30`.

**Re-running `go test -bench` without `-count=1` in a sequence.** The test cache may skip subsequent runs; force fresh: `-count=10`.

**Adding new benchmarks between old and new.** benchstat aligns by name; new benchmarks in `new.txt` not in `old.txt` show as missing baseline. Either keep both runs aligned or use `-row=/specific`.

**Comparing benchmarks across different Go versions.** Compiler changes affect performance independently of your code. Either keep Go version fixed for the comparison or call out the version in the report.

**Confusing "0.00% delta, p=0.001" as significant.** 0% delta is no change; `p` of the delta can still be tiny if variance is tiny. Read the delta first.

**Forgetting to `b.ResetTimer()` after setup.** Setup time pollutes the benchmark; old.txt looks fast, new.txt looks fast if setup is reproducible — but absolute numbers lie.

**Using benchstat output as the only proof of an optimization.** Verify with a real workload too; micro-benchmarks can mislead.

## Performance Notes

- `benchstat` runtime: <1 s for typical files.
- Memory: trivial; parses then computes.
- benchmark generation cost: `-count=10` ≈ 10× `-count=1` per benchmark.

`-count=10` of a 1 s benchmark with 20 sub-benchmarks: ~200 s. Plan time accordingly.

## How Big Companies Use It

- **Google** uses benchstat across the Go team for every performance-touching CL: https://go.googlesource.com.
- **Uber** uses benchstat in pre-merge gates for `zap`, `dig`, and other libraries: https://github.com/uber-go.
- **Cloudflare** uses benchstat to compare HTTP/2 vs. HTTP/3 implementations in their Go stack: https://blog.cloudflare.com.
- **CockroachDB** runs nightly benchstat comparisons to spot regressions: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses benchstat for `wireguard-go` perf work: https://tailscale.com/blog.
- **HashiCorp** uses benchstat sporadically; Vault's crypto paths track perf via it: https://github.com/hashicorp/vault.
- **The Go team** uses benchstat for every compiler/runtime change that affects perf: https://github.com/golang/go.

## Source Code References

`benchstat` lives at `golang.org/x/perf`.

- Source: [`golang.org/x/perf`](https://github.com/golang/perf).
- benchstat cmd: [`cmd/benchstat`](https://github.com/golang/perf/tree/master/cmd/benchstat).
- Library: [`benchmath`](https://github.com/golang/perf/tree/master/benchmath), [`benchproc`](https://github.com/golang/perf/tree/master/benchproc), [`benchstat`](https://github.com/golang/perf/tree/master/benchstat).
- Benchmark output format: [`benchfmt`](https://github.com/golang/perf/tree/master/benchfmt).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "benchstat documentation": https://pkg.go.dev/golang.org/x/perf/cmd/benchstat.
- "Statistically rigorous Java performance evaluation" (Georges et al.; foundational): https://dri.es/files/oopsla07-georges.pdf.
- "How to benchmark a Go function" (Dave Cheney): https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go.
- "Reducing benchmark noise" (perf wiki): https://github.com/golang/go/wiki/CompilerOptimizations.
- "b.Loop in Go 1.24" (Michael Knyszek): https://go.dev/blog/testing-b-loop.
- "Benchmarking and profiling" (Filippo Valsorda): https://blog.filippo.io.

## Exercises / Self-Check

1. Run `-count=10` of an existing benchmark twice without changing code. Compare with benchstat. Are the differences significant?
2. Use `-cpu=1,4,8` and `-count=10`. Does parallelism help your benchmark? Read the `p`-values.
3. Add a `b.ReportMetric` for throughput and re-run benchstat. Does the throughput metric appear?
4. Compare a benchmark on a battery vs. plugged-in machine. How much does noise differ?
5. Convert a `for i := 0; i < b.N; i++` loop to `for b.Loop()`. Does benchstat report any change?
