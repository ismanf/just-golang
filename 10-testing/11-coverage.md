# Coverage — `go test -cover`, Integration-Test Coverage (1.20+)

## TL;DR

`go test -cover` instruments your code at *block* level: each basic block of source becomes a counter that increments when execution reaches it. After the test, the framework reports the fraction of blocks reached. Pass `-coverprofile=cov.out` to dump the data and analyze with **`go tool cover -html=cov.out`** (browser) or **`go tool cover -func=cov.out`** (per-func percentage). Three coverage modes: **`set`** (was reached? boolean, default), **`count`** (how many times?), **`atomic`** (count under `-race`). The big 1.20 addition is **coverage from production-style binaries**: build with `-cover -o app`, run with `GOCOVERDIR=/cov ./app`, the binary writes coverage data on exit, then merge with **`go tool covdata`**. This lets integration tests, end-to-end tests, and even production traffic contribute to coverage numbers. Avoid the trap of treating coverage % as a code-quality target — it measures execution, not correctness.

## Mental Model

```
   Source file (instrumented)
        │
        ▼
   each basic block has a counter:
        if x > 0 { counter[42]++; ... }   ← `set` mode
        for i := ... { counter[43]++; ... }
        │
        ▼
   on exit:
        ├─ go test -coverprofile=cov.out   ← writes profile
        │
   OR
        ├─ ./binary (built with -cover)    ← writes to $GOCOVERDIR on exit
        │
        ▼
   profile file (text format)
        github.com/me/pkg/foo.go:10.2,12.3 1 1
        github.com/me/pkg/foo.go:14.5,16.7 1 0
                                      │   │ └ count
                                      │   └ statement count
                                      └ end pos
        │
        ▼
   go tool cover -html=cov.out         (browser, colored)
   go tool cover -func=cov.out         (per-func table)
   go tool covdata merge ...           (1.20+ for binary-mode)
```

A `0` count means the block was never reached; non-zero = reached. `count` mode reports the actual hit count.

## Syntax & Basic Usage

```bash
# Standard test coverage
$ go test -cover ./...
$ go test -coverprofile=cov.out ./...
$ go test -coverpkg=./... -coverprofile=cov.out ./...        # cover deps too
$ go tool cover -func=cov.out                                  # text per-func
$ go tool cover -html=cov.out                                  # opens browser
$ go tool cover -html=cov.out -o cov.html                      # writes file

# Coverage from binaries (1.20+)
$ go build -cover -o app ./cmd/app
$ mkdir /tmp/cov && GOCOVERDIR=/tmp/cov ./app                  # binary writes on exit
$ go tool covdata percent -i /tmp/cov                          # quick summary
$ go tool covdata textfmt -i /tmp/cov -o cov.out               # to legacy text format
$ go tool covdata merge -i /tmp/cov1,/tmp/cov2 -o /tmp/merged  # merge runs

# Modes
$ go test -covermode=set ./...        # default
$ go test -covermode=count ./...
$ go test -covermode=atomic ./...     # required with -race
```

## Deep Dive

### Block-level instrumentation

The Go compiler (`cmd/cover`) walks the AST and inserts a counter increment at the start of every basic block. A basic block is a straight-line sequence of statements with one entry and one exit (the next branch). Block boundaries: `if`/`else`, `for`, `case`, `defer`, `goto`.

For:

```go
func F(x int) int {
    if x > 0 {
        return x * 2
    }
    return 0
}
```

Two blocks: the `return x * 2` branch and the `return 0` branch. Each gets a counter.

The instrumented binary writes the counters and the block-position table to the profile on exit.

### Profile format

```
mode: set
github.com/me/pkg/foo.go:10.2,12.3 1 1
github.com/me/pkg/foo.go:14.5,16.7 1 0
```

- Line 1: mode (`set`, `count`, `atomic`).
- Each subsequent line: `filename:startline.startcol,endline.endcol statementCount hitCount`.

`statementCount` is how many statements the block covers (for display); `hitCount` is the counter.

In `count` mode, hitCount is the actual number; in `set`, it's 0 or 1.

### Cover modes

| Mode      | Counter type | Concurrent safe | Use case |
|-----------|--------------|-----------------|----------|
| `set`     | `int8`        | No (race)        | Default; "was this reached?" |
| `count`   | `uint32`      | No (race)        | Hot-spot analysis |
| `atomic`  | `uint32` atomic | Yes            | Required with `-race`; production binaries |

With `-race`, the framework auto-upgrades to `atomic`.

### `-coverpkg`

By default, `go test -cover ./pkg` only counts coverage **of the package being tested**. To count packages outside the test package:

```bash
$ go test -coverpkg=./... -cover ./...
$ go test -coverpkg=github.com/me/pkg,github.com/me/util -cover ./tests
```

Critical for: integration tests in a separate package, tests that exercise multiple packages, cross-package code paths.

The downside: bigger profiles, slower instrumentation.

### `go tool cover -html`

```bash
$ go tool cover -html=cov.out
```

Opens a browser with the source files colored:

- **Green**: covered.
- **Red**: not covered.
- **Grey**: not instrumented (often non-Go like generated assembly).

The page has a dropdown to pick files. Save to disk:

```bash
$ go tool cover -html=cov.out -o cov.html
```

Useful in CI for uploading as a build artifact.

### `go tool cover -func`

```bash
$ go tool cover -func=cov.out
github.com/me/pkg/foo.go:10:    F        100.0%
github.com/me/pkg/foo.go:20:    G         50.0%
total:                          (statements) 75.0%
```

Per-function percentage; last line is total. Pipe to filters:

```bash
$ go tool cover -func=cov.out | awk '$3+0 < 50'    # functions <50% covered
$ go tool cover -func=cov.out | grep -v 100.0
```

### Coverage from production binaries (1.20+)

The big new feature in Go 1.20: build a binary with `-cover` instead of just running `go test -cover`:

```bash
$ go build -cover -o app ./cmd/app
$ mkdir /tmp/cov
$ GOCOVERDIR=/tmp/cov ./app
# ... exercise the app via real traffic, integration tests, manual smoke testing
# on exit:
$ ls /tmp/cov
covcounters.<hash>.<pid>.<timestamp>
covmeta.<hash>
```

Two file types:
- **covmeta**: package/function/block layout (one per binary).
- **covcounters**: hit counts (one per process invocation).

Inspect:

```bash
$ go tool covdata percent -i /tmp/cov
$ go tool covdata textfmt -i /tmp/cov -o cov.out
$ go tool cover -html=cov.out
```

### `go tool covdata`

Subcommands:

```bash
$ go tool covdata percent -i /tmp/cov              # overall %
$ go tool covdata pkglist -i /tmp/cov              # list packages
$ go tool covdata func -i /tmp/cov                 # per-func %
$ go tool covdata textfmt -i /tmp/cov -o cov.out   # convert to legacy text format
$ go tool covdata merge -i /tmp/a,/tmp/b -o /tmp/merged   # merge multiple runs
$ go tool covdata subtract -i /tmp/all -base /tmp/baseline -o /tmp/diff
```

`merge` is the workhorse for combining unit-test coverage (`go test -coverprofile`) with binary-mode coverage (from prod-style runs).

### Merging unit + integration coverage

```bash
# Unit coverage (legacy text format)
$ go test -coverprofile=unit.out -coverpkg=./... ./...

# Integration coverage (binary mode)
$ go build -cover -o app ./cmd/app
$ mkdir /tmp/cov
$ GOCOVERDIR=/tmp/cov ./run-integration-tests.sh

# Convert binary to text
$ go tool covdata textfmt -i /tmp/cov -o integration.out

# Merge text profiles (manual; covdata merge is for binary format)
$ grep -h -v '^mode:' unit.out integration.out | sort -u > merged.out
$ ( head -1 unit.out; cat merged.out ) > final.out
$ go tool cover -html=final.out
```

Slightly awkward; tools like `gocovmerge` (third-party) automate it.

### Coverage in CI

GitHub Actions with Codecov:

```yaml
- run: go test -coverprofile=cov.out -coverpkg=./... ./...
- uses: codecov/codecov-action@v4
  with:
    files: cov.out
```

For diff coverage (only new lines):

```bash
$ go test -coverprofile=cov.out ./...
$ # use github.com/wadey/gocovmerge or similar to integrate with PR diff
```

### Coverage and table-driven tests

Each case in a table-driven test contributes to coverage. So:

```go
tests := []struct{ in, want int }{
    {0, 0},
    {1, 1},
    {-1, 1},
}
for _, tt := range tests {
    if got := Abs(tt.in); got != tt.want { /* ... */ }
}
```

— exercises the three branches of `Abs`. Adding cases is the cheapest way to raise coverage.

### Coverage as a metric

The single biggest mistake: **chasing coverage %**. Coverage measures execution, not assertion. A test can:

- Call every function with `_ = F()` and ignore the result — 100% coverage, zero correctness.
- Cover every branch with the same input — high coverage, no edge-case testing.

Coverage is a **floor**, not a ceiling. "We have 80%" doesn't mean the code works; "we have 0%" definitively means it's untested.

Useful goals:
- **No regression**: per-PR coverage doesn't drop.
- **Critical paths**: assert specific high-risk functions are 100% covered.
- **New code**: PR adds X lines of code; expect Y% to be tested.

Bad goals:
- "Reach 80% overall."
- "Block PRs under 75% file coverage."

### Generated code

```go
// Code generated by protoc-gen-go. DO NOT EDIT.
```

Don't measure generated code. Exclude via:

```bash
$ go test -coverpkg=$(go list ./... | grep -v '/generated/') ./...
```

Or per-file via `-tags`. Or use a wrapper like `gocov-xml` that filters.

### Cover doesn't count `_test.go`

The instrumentation skips `_test.go` files (you don't measure test coverage of tests). External test packages (`package foo_test`) contribute to `foo`'s coverage when run.

### Hot-path analysis via `count`

```bash
$ go test -covermode=count -coverprofile=cov.out ./pkg
$ go tool cover -func=cov.out | sort -k 3 -n -r | head
```

Sorting by count shows hot vs. cold blocks. Useful for finding never-tested or barely-tested branches.

### Coverage with `-race`

```bash
$ go test -race -covermode=atomic -coverprofile=cov.out ./...
```

Atomic mode is required for thread safety; framework switches automatically.

### Coverage and benchmarks

`go test -bench` doesn't produce coverage; benchmarks are about performance. Coverage and benchmarks are orthogonal.

## Standard Library Hooks

- `cmd/cover` — the instrumenter.
- `cmd/covdata` (1.20+) — manage binary coverage.
- `cmd/cover` HTML/text renderers.
- `testing.M.Run` — invokes cover-aware code path.
- `runtime/coverage` (1.20+) — programmatic flush of counters.

For programmatic flush in a long-running service:

```go
import "runtime/coverage"

func dumpCoverage(dir string) {
    coverage.WriteCountersDir(dir)
    coverage.WriteMetaDir(dir)
}
```

Allows periodic flushes from a `-cover`-built binary (without waiting for exit).

## Real-World Patterns

### 1. Local fast loop

```bash
$ go test -cover ./pkg
ok  github.com/me/pkg  0.123s  coverage: 78.4% of statements
```

### 2. HTML drill-down

```bash
$ go test -coverprofile=cov.out ./pkg
$ go tool cover -html=cov.out
```

Click through files; find red blocks.

### 3. Module-wide

```bash
$ go test -coverpkg=./... -coverprofile=cov.out ./...
$ go tool cover -func=cov.out | tail
```

### 4. Per-function check

```bash
$ go tool cover -func=cov.out | awk '/main\./ && $3+0 < 80'
```

Fails functions in `main.*` under 80%.

### 5. Integration coverage

```bash
$ go build -cover -o app ./cmd/app
$ mkdir /tmp/cov
$ GOCOVERDIR=/tmp/cov ./app
$ # ... run tests against the binary
$ # Ctrl-C the app
$ go tool covdata percent -i /tmp/cov
```

### 6. Merge runs

```bash
$ go tool covdata merge -i /tmp/cov1,/tmp/cov2 -o /tmp/merged
$ go tool covdata percent -i /tmp/merged
```

### 7. Codecov upload

```yaml
- run: go test -race -coverprofile=cov.out -covermode=atomic ./...
- uses: codecov/codecov-action@v4
```

## Anti-Patterns & Gotchas

**Optimizing for coverage %.** Easy to game; doesn't measure correctness.

**No `-coverpkg`.** Tests in `pkg/foo_test` don't credit `pkg/foo` for coverage. Add `-coverpkg=./...`.

**`-covermode=count` with `-race`.** Will switch to atomic anyway; pass `-covermode=atomic` to be explicit.

**Including generated code.** Inflates denominator; teams add tests for generated code that produce no value. Exclude.

**Forgetting to merge runs.** Unit + integration give 60% combined; either alone shows 30%. Merge before reporting.

**Running `-cover` on benchmarks.** Distorts ns/op (instrumentation adds work). Use separate runs.

**Treating "Branch coverage" claims at face value.** Go does *statement* coverage, not branch. A `if x && y` is one block; both subconditions count together.

**`go tool cover -html` open from a binary built with `-trimpath`.** Source paths are relative; browser can't find files. Either skip `-trimpath` for cover runs or run the HTML from the source tree.

**Coverage from `GOCOVERDIR` collisions.** Multiple processes writing to the same directory concurrently can corrupt counters. Each process should write to its own subdir.

**Forgetting that `GOCOVERDIR` requires the binary built with `-cover`.** Setting it on an unrelated binary does nothing.

**Coverage CI gate that fails on small drift.** A 0.5% drop after a refactor causes false alarms. Gate on absolute thresholds, not deltas.

**Mixing `-coverpkg=./...` and benchmarks.** Coverage instrumentation slows benchmarks; isolate.

**Using `-coverprofile` and `-cpuprofile` together.** Coverage instrumentation pollutes the CPU profile.

## Performance Notes

- Coverage instrumentation overhead: ~3–10% on test runtime.
- Atomic mode: ~5% slower than set.
- Coverage profile file size: ~10–100 KB per package.
- `go tool cover -html`: instant for small profiles; <5 s for huge ones.
- `go tool covdata merge`: ms per source file.
- Binary-mode coverage: ~10% extra binary size, ~3% runtime.

For prod-traffic coverage (long-running), keep instrumentation always on; the cost is modest.

## How Big Companies Use It

- **Google** uses coverage internally; the `cmd/cover` tool comes from Google: https://github.com/golang/go/tree/master/src/cmd/cover.
- **Kubernetes** tracks coverage via Codecov; PRs see a coverage badge: https://github.com/kubernetes/kubernetes.
- **Uber** uses coverage thresholds per package: https://github.com/uber-go.
- **CockroachDB** uses coverage including integration tests via 1.20 binary mode: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses Codecov in Terraform: https://github.com/hashicorp/terraform.
- **Tailscale** uses coverage but doesn't gate on it; focus on critical-path tests: https://github.com/tailscale/tailscale.
- **The Go team** uses coverage selectively for new features: https://github.com/golang/go.

## Source Code References

Pinned to `go1.26`.

- `cmd/cover`: [`src/cmd/cover`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/cover).
- `cmd/covdata`: [`src/cmd/covdata`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/covdata).
- Cover compiler integration: [`src/cmd/compile/internal/coverage`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/coverage).
- `runtime/coverage` (1.20+): [`src/runtime/coverage`](https://github.com/golang/go/tree/release-branch.go1.26/src/runtime/coverage).
- Profile parser: [`src/cmd/cover/profile.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/cover/profile.go).
- `internal/coverage`: [`src/internal/coverage`](https://github.com/golang/go/tree/release-branch.go1.26/src/internal/coverage).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "The cover story" (Rob Pike, Go blog): https://go.dev/blog/cover.
- "Code coverage for Go integration tests" (Go 1.20 blog): https://go.dev/blog/integration-test-coverage.
- "go tool cover docs": https://pkg.go.dev/cmd/cover.
- "go tool covdata docs": https://pkg.go.dev/cmd/covdata.
- "Coverage profiling support" (Than McIntosh): https://go.googlesource.com/proposal/+/master/design/51430-revised-coverage-design.md.
- "Why 100% coverage is a bad goal" (various): https://research.swtch.com.

## Exercises / Self-Check

1. Run `go test -cover ./...` and identify the package with lowest coverage. Drill in with `-html`.
2. Add `-coverpkg=./...` and re-run. How does the number change?
3. Build a binary with `-cover`, run it under integration tests, merge coverage with unit tests.
4. Use `-covermode=count` to find the hottest 5 functions.
5. Try to game coverage: write a test that calls 10 functions, discards results. Note the coverage % vs. test value.
