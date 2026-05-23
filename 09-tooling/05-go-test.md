# `go test` — The Testing Driver

## TL;DR

`go test` compiles a package's source plus its `_test.go` files into a one-off binary, links in the `testing` framework, runs the binary, and reports results. The flag surface is huge but cleanly partitioned: **selection** (`-run`, `-skip`, `-tags`), **iteration** (`-count`, `-race`, `-shuffle`, `-cpu`), **output** (`-v`, `-json`, `-bench`, `-benchtime`, `-benchmem`), **coverage** (`-cover`, `-coverprofile`, `-coverpkg`), **profiling** (`-cpuprofile`, `-memprofile`, `-blockprofile`, `-mutexprofile`, `-trace`), and **caching/timeouts** (`-timeout`, `-count=1` to defeat cache, `-failfast`). Test binaries are cached: re-running `go test` with no source changes is a near-instant cache hit. Test artifacts (profiles, coverage, the binary itself) are *not* in the cache — they only appear when you ask via `-c` or one of the `-*profile` flags. Part 10 covers writing tests; this page covers driving them.

## Mental Model

```
   go test ./pkg
       │
       ▼
   build action: pkg + pkg_test.go + testing framework  →  pkg.test binary
       │
       ▼
   look up $GOCACHE for "test result for this pkg + flags + env"
       ├─ hit  → print cached output, exit
       └─ miss → exec pkg.test with flags
                     │
                     ▼
                 testing.Main runs:
                   - filter via -run/-skip
                   - for each Test, b.N each Benchmark, etc.
                   - emit textual or -json output
                     │
                     ▼
                 write artifacts (profiles, coverprofile) if requested
                     │
                     ▼
                 exit 0 (pass) / 1 (fail) / 2 (build error)
       ▼
   if exit 0, cache result keyed by source+flags+env
```

The test binary is just a regular Go program with `package main` synthesized; you can also build it standalone via `go test -c` and run it directly (useful for cross-compiled tests).

## Syntax & Basic Usage

```bash
$ go test                       # current package
$ go test ./...                 # all packages, recursively
$ go test ./pkg/...
$ go test -v ./...              # verbose: show every test name
$ go test -run TestFoo ./...    # only tests matching regex
$ go test -run 'TestFoo/sub'    # subtest selector
$ go test -count=5 ./...        # run each test 5 times
$ go test -count=1 ./...        # disable cache (forces re-run)
$ go test -race ./...           # race detector
$ go test -timeout=30s ./...    # per-binary timeout (default 10m)
$ go test -short ./...          # set testing.Short()=true (suite-level convention)
$ go test -bench=. ./...        # run benchmarks
$ go test -cover ./...          # coverage summary
$ go test -coverprofile=c.out   # write coverage data
$ go test -c -o pkg.test ./pkg  # compile without running
```

## Deep Dive

### Selection: `-run` and `-skip`

`-run` and `-skip` (1.20+) are regex matchers against the slash-joined test path. Examples:

```bash
$ go test -run TestFoo
$ go test -run 'TestFoo|TestBar'
$ go test -run 'TestFoo/SubA'
$ go test -run 'TestFoo$'                 # exact match (anchor)
$ go test -skip 'TestSlow'                # run everything except matching
$ go test -run 'TestFoo' -skip 'Slow'     # combine
```

Subtests created via `t.Run("Sub", ...)` show up as `TestFoo/Sub`. Use `/` in the regex to drill in. Special characters need escaping (`go test -run 'TestPort\.8080'`).

### `-count` and test caching

```bash
$ go test ./pkg          # first run: builds + executes
$ go test ./pkg          # second run: cached, instant
$ go test -count=1 ./pkg # force re-run (disables cache for this invocation)
$ go test -count=10 ./pkg# run each test 10 times
```

Cache key includes: source, build flags, env vars used by the test (Go records reads of `os.Getenv` automatically), and the package's deps. Modify a `.go` file → cache invalidates. Modify an env var the test reads → cache invalidates.

The cache does *not* trip on:

- `time.Now()` results (Go can't introspect).
- External file changes outside the package source (read via `os.Open(...)`).
- Network state.

For tests that depend on external state, set `-count=1` or use `t.Setenv("CACHE_KEY", ...)` to add a poison input.

Wipe the test cache:

```bash
$ go clean -testcache
```

### `-race` — data race detector

```bash
$ go test -race ./...
```

Instruments memory accesses; flags concurrent unsynchronized read+write to the same memory. Slowdown: 2–10×. Memory: ~5–10× larger. Always run `-race` at least in CI. See `07-concurrency/15-race-detector.md` for the deep dive.

A race report:

```
WARNING: DATA RACE
Write at 0x00c0000c8050 by goroutine 7:
  main.(*Counter).Add()
      /src/main.go:14 +0x44

Previous read at 0x00c0000c8050 by goroutine 6:
  main.(*Counter).Get()
      /src/main.go:18 +0x22
```

Exit code is 1 if any races fired. `GORACE=halt_on_error=1` (env var) stops after the first.

### `-cpu`

```bash
$ go test -cpu=1,2,4 ./...
```

Runs each test once per `GOMAXPROCS` value. Useful for catching concurrency bugs that only show up under parallelism. Combine with `-race`.

### `-shuffle` (since 1.17)

```bash
$ go test -shuffle=on ./...
$ go test -shuffle=1234 ./...     # specific seed for reproducibility
```

Randomizes test execution order to surface order-dependent tests. The `on` form picks a random seed and prints it.

### `-v` and `-json`

```bash
$ go test -v ./...      # human-readable, includes PASS/FAIL per test
$ go test -json ./...   # one JSON object per event, line-delimited
```

`-json` output:

```json
{"Time":"2026-05-23T...","Action":"run","Package":"github.com/me/pkg","Test":"TestFoo"}
{"Time":"...","Action":"output","Package":"...","Test":"TestFoo","Output":"=== RUN   TestFoo\n"}
{"Time":"...","Action":"pass","Package":"...","Test":"TestFoo","Elapsed":0.002}
```

`tparse`, `gotestsum`, and other UIs consume `-json`. CI dashboards typically expect it.

### Benchmarks: `-bench`, `-benchtime`, `-benchmem`, `-cpu`

```bash
$ go test -bench=.                 # run all benchmarks
$ go test -bench=BenchmarkFoo      # specific
$ go test -bench=. -benchmem        # include allocations
$ go test -bench=. -benchtime=10s   # run for ~10s each (default 1s)
$ go test -bench=. -benchtime=5x    # run exactly 5 iterations
$ go test -bench=. -count=10        # run benchmark suite 10x for stats
$ go test -bench=. -cpu=1,4,8       # benchmark at different GOMAXPROCS
```

Default output:

```
BenchmarkFoo-8   1000000  1245 ns/op  120 B/op  3 allocs/op
```

Save and compare with `benchstat` (`09-tooling/21-benchstat.md`).

Since 1.24, `b.Loop()` replaces `for i := 0; i < b.N; i++` and is more accurate (auto-adjusts allocations, recompiles inlined helpers). See `10-testing/04-benchmarks.md`.

### `-coverprofile`, `-coverpkg`, `-covermode`

```bash
$ go test -cover ./...
$ go test -coverprofile=c.out ./...
$ go test -coverprofile=c.out -covermode=count ./...
$ go test -coverprofile=c.out -coverpkg=./... ./...
```

`-cover` prints a summary; `-coverprofile` writes a file consumable by `go tool cover`:

```bash
$ go tool cover -func=c.out         # per-function coverage
$ go tool cover -html=c.out         # browser view
$ go tool cover -html=c.out -o cov.html
```

`-covermode`:

- `set` (default): line was reached (boolean).
- `count`: how many times.
- `atomic`: like `count` but safe under `-race`.

`-coverpkg` controls *which packages* contribute to coverage. Default is the packages being tested; setting `-coverpkg=./...` measures coverage of the whole module from this test run — useful when one package's tests exercise others.

Since 1.20, you can also collect coverage from a `-cover`-instrumented production binary via `GOCOVERDIR`; see `10-testing/11-coverage.md`.

### Profiling flags

```bash
$ go test -cpuprofile=cpu.out ./pkg
$ go test -memprofile=mem.out ./pkg
$ go test -blockprofile=block.out ./pkg
$ go test -mutexprofile=mutex.out ./pkg
$ go test -trace=trace.out ./pkg
```

These write profiles for `go tool pprof` (or `go tool trace`):

```bash
$ go tool pprof -http=:8080 cpu.out
$ go tool trace trace.out
```

Profiling flags imply `-c`-style build — the test binary stays in the working dir (often named `pkg.test`).

### `-c`: compile test binary without running

```bash
$ go test -c -o pkg.test ./pkg
$ ./pkg.test -test.v -test.run TestFoo
```

Useful for cross-compiling tests:

```bash
$ GOOS=linux GOARCH=arm64 go test -c -o pkg.test ./pkg
$ scp pkg.test target:/tmp/ && ssh target /tmp/pkg.test
```

The binary takes `-test.*` flags (with the `test.` prefix) — different from how the `go test` wrapper accepts them.

### `-failfast`

```bash
$ go test -failfast ./...
```

Stops as soon as one test fails. Within a single package it stops after that test; with `./...` it stops on the first failing package.

### `-timeout`

```bash
$ go test -timeout=30s ./...
$ go test -timeout=0 ./...      # disable (dangerous in CI)
```

Default is 10 minutes per package. When the timeout fires, the test binary is killed with a stack dump of every goroutine — invaluable for debugging deadlocks.

### `-parallel`

```bash
$ go test -parallel=4 ./...
```

Caps the number of tests calling `t.Parallel()` that run concurrently within a single package. Default is `GOMAXPROCS`. Lowering helps when parallel tests fight for a shared resource (DB pool, port).

### Build flags pass through

Most `go build` flags also work with `go test`:

```bash
$ go test -tags=integration ./...
$ go test -gcflags="all=-l" ./...
$ go test -ldflags="-X main.flag=on" ./...
$ go test -trimpath ./...
$ go test -mod=vendor ./...
$ go test -race -pgo=auto ./...
```

### Env vars that affect tests

- `GOMAXPROCS` — concurrency cap.
- `GOTRACEBACK` — `none`/`single`/`all`/`system`/`crash`; controls stack dumps.
- `GORACE` — race detector options (`halt_on_error=1`, `history_size=7`, etc.).
- `GODEBUG` — many runtime knobs.
- `GOEXPERIMENT` — feature flags (e.g., `loopvar` pre-1.22).
- `GOFLAGS` — default flags for `go` invocations.

Anything the test reads via `os.Getenv` enters the cache key automatically.

### `go test ./...` in a workspace

In a workspace (`09-tooling/04-go-work.md`), `./...` expands across every `use`d module. One command, full coverage.

## Standard Library Hooks

- `testing.T`, `testing.B`, `testing.F`, `testing.PB`, `testing.M` — the test API surface.
- `testing/quick` — basic property-based testing.
- `testing/iotest` — io-related test helpers.
- `testing/fstest` — virtual filesystem for tests.
- `testing/synctest` — synthetic-time testing (1.24+; see `10-testing/13-testing-synctest.md`).
- `runtime/pprof` — programmatic profiles.
- `runtime/trace` — programmatic tracing.

## Real-World Patterns

### 1. Standard local loop

```bash
$ go test -race -count=1 ./...
```

`-race` for safety, `-count=1` to bust the cache so you see the current state.

### 2. CI lint+test

```yaml
- run: go test -race -shuffle=on -coverprofile=cov.out ./...
- run: go tool cover -func=cov.out | tail -1
- uses: codecov/codecov-action@v4
  with: { files: cov.out }
```

### 3. Bench comparison

```bash
$ git checkout main
$ go test -bench=. -benchmem -count=10 ./pkg > old.txt
$ git checkout feature
$ go test -bench=. -benchmem -count=10 ./pkg > new.txt
$ benchstat old.txt new.txt
```

`benchstat` reports mean, ±, and whether the delta is statistically significant.

### 4. Single subtest

```bash
$ go test -run 'TestUser/CreateThenDelete' ./...
```

### 5. Cross-compiled test

```bash
$ GOOS=linux GOARCH=amd64 go test -c -o pkg.test ./pkg
$ docker run --rm -v "$PWD":/w -w /w alpine /w/pkg.test
```

### 6. Trace on a hot path

```bash
$ go test -bench=BenchmarkPath -trace=trace.out -benchtime=5s ./pkg
$ go tool trace trace.out
```

### 7. Test with a custom build tag

```go
// integration_test.go
//go:build integration
package mypkg_test
```

```bash
$ go test ./...                     # skips integration tests
$ go test -tags=integration ./...   # includes them
```

## Anti-Patterns & Gotchas

**Using `-count=1` always.** You give up the cache; CI gets ~5× slower for no gain. Use `-count=1` only when you suspect cache pollution.

**Forgetting `-race` in CI.** Races often manifest only under load and only on multi-core. Always at least one CI job with `-race`.

**Running benchmarks without `-count`.** A single benchmark run has too much variance to compare against another. `-count=10` + `benchstat` is the minimum for a defensible comparison.

**Using `-bench` and `-cover` together.** Coverage instrumentation distorts benchmarks. Profile *or* cover, not both.

**Forgetting `-benchmem`.** Without it, you only see ns/op; the bigger story (allocations) is hidden.

**Timeout too short.** Default 10m is fine for unit tests. Long-running integration tests need `-timeout=30m` or more.

**Timeout disabled (`-timeout=0`).** A hung test runs forever; CI hangs until a job-level timeout kills it (much harder to debug).

**Filtering with `-run` and forgetting it matches the slash path.** `go test -run TestFoo` matches every test named `TestFoo` *and* every subtest like `TestBar/TestFoo`. Use `-run 'TestFoo$'` to anchor.

**Expecting `-shuffle=on` to be deterministic.** It picks a fresh seed each run. For reproducibility, capture the seed it prints and pass it back: `-shuffle=1234567890`.

**Hitting `t.Parallel()` from inside a `t.Run` whose parent already finished.** Common in table-driven loops. See `10-testing/03-subtests-and-tparallel.md`.

**Caching a test that depends on external state.** Wrap external reads in helper code that calls `t.Setenv("...", ...)` or otherwise registers an env-var read, so the cache invalidates correctly.

**Trusting `-coverprofile` to cover code in other packages.** Without `-coverpkg=./...`, only the *test's own package* contributes to the profile.

## Performance Notes

- Cached re-run of `go test ./...` (no source changes): <500 ms even for huge projects.
- First-time `go test ./...` for a 200-pkg module: 30s – 5min depending on test density.
- `-race` slowdown: 2–10× wall time, ~5× memory.
- `-cover` overhead: ~3% on test runtime; binary ~2× larger.
- `-bench=.`: each benchmark gets ~1 s of wall time by default; `-benchtime=1x` runs it exactly once.
- `-trace`: low overhead (~1%) during collection but produces big files (~10 MB/s).
- `-cpuprofile`: ~5% overhead; samples at 100Hz.

The fastest test loop: well-structured packages with cached deps and `-count=1` only when necessary.

## How Big Companies Use It

- **Google** runs `go test -race -shuffle=on` across every `golang.org/x/*` repo as part of TryBots: https://go.dev/doc/contribute.
- **Kubernetes** uses `go test -v -count=1 ./...` in their hack scripts plus `gotestsum` to format output: https://github.com/kubernetes/kubernetes.
- **Uber** runs benchmarks in CI with `-count=20` and `benchstat` to gate performance regressions: https://github.com/uber-go/zap.
- **Cloudflare** uses build-tagged integration tests (`-tags=integration`) gated behind a separate CI stage: https://blog.cloudflare.com/tag/golang/.
- **HashiCorp** runs `go test -race -timeout=30m ./...` per service; Terraform's test suite is ~20m on a beefy runner: https://github.com/hashicorp/terraform.
- **CockroachDB** stripes tests across hundreds of CI runners, each `go test`ing a few packages, then merging coverage: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses `-shuffle=on` and reports the seed in failure messages so flaky tests are reproducible: https://github.com/tailscale/tailscale.

## Source Code References

Pinned to `go1.26`.

- `go test`: [`src/cmd/go/internal/test/test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/test/test.go).
- Test binary main: [`src/testing/testing.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/testing.go).
- Benchmark machinery: [`src/testing/benchmark.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/benchmark.go).
- Fuzz machinery: [`src/testing/fuzz.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/fuzz.go).
- Test cache: [`src/cmd/go/internal/test/testcache.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/test/testcache.go).
- Coverage support: [`src/cmd/cover`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/cover).
- Race detector wiring: [`src/runtime/race`](https://github.com/golang/go/tree/release-branch.go1.26/src/runtime/race).
- JSON output: [`src/cmd/internal/test2json`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/internal/test2json).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Testing flags" (Go docs): https://pkg.go.dev/cmd/go#hdr-Testing_flags.
- "Testing package": https://pkg.go.dev/testing.
- "Coverage profiling support for integration tests" (Go blog): https://go.dev/blog/integration-test-coverage.
- "PGO in Go" (Michael Pratt): https://go.dev/blog/pgo.
- "Subtests and sub-benchmarks" (Marcel van Lohuizen): https://go.dev/blog/subtests.
- Dave Cheney, "How to write benchmarks in Go": https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go.

## Exercises / Self-Check

1. Run `go test` twice in a row on a package with no changes. Confirm the second run is cached. Now `touch` a `.go` file — what happens?
2. Use `-run 'TestX$'` vs. `-run 'TestX'`. Construct test names where the two filter differently.
3. Profile a benchmark with `-cpuprofile=cpu.out` and view it in `go tool pprof -http=:8080 cpu.out`. Identify the hottest leaf.
4. Run `go test -race -shuffle=on -count=10 ./...` on a project. Did any flaky tests surface? Capture the seed and reproduce.
5. Write a coverage report that includes packages outside the one being tested, using `-coverpkg=./...`.
