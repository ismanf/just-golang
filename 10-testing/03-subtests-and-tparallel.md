# Subtests and `t.Parallel`

## TL;DR

`t.Run("name", func(t *testing.T) { ... })` creates a **subtest**: a nested `*testing.T` with its own pass/fail state, its own `t.Cleanup` stack, its own name in failure output. Subtests structure related assertions, integrate with `-run` filtering (`-run TestX/case_name`), and are the unit at which `t.Parallel` opts in. `t.Parallel()` tells the test runner "I'm safe to run alongside other parallel tests"; the runner pauses the test until the parent's serial work is done, then schedules it concurrently with other parallel tests (up to `-parallel=N`, default `GOMAXPROCS`). The two **traps** are: (1) the pre-1.22 loop-variable capture bug (now fixed for `go 1.22+` modules), and (2) the fact that a parent test does **not** wait for its parallel subtests to finish before its own function returns — `t.Cleanup` and `defer` in the parent fire *before* the parallel subtests run.

## Mental Model

```
   func TestOuter(t *testing.T) {
       setup()
       defer teardown()              // Runs immediately when TestOuter returns,
                                     // BEFORE parallel subtests execute.
       t.Cleanup(cleanup)            // SAME — fires before subtests.

       t.Run("A", func(t *testing.T) {
           t.Parallel()
           // ... runs concurrently with B, C
       })
       t.Run("B", func(t *testing.T) {
           t.Parallel()
       })
       t.Run("C", func(t *testing.T) { /* serial */ })
   }

   Execution order:
     1. setup()
     2. C runs serially (registered with no t.Parallel)
     3. TestOuter's body returns
     4. teardown() and cleanup() fire
     5. A and B fan out (now scheduled by the runner)
```

If you need cleanup *after* parallel subtests, register it from **inside** them, or wrap the parallel block in a serial subtest that the parent waits for.

## Syntax & Basic Usage

```go
func TestX(t *testing.T) {
    t.Run("subA", func(t *testing.T) { /* ... */ })

    t.Run("group", func(t *testing.T) {
        t.Run("nested1", func(t *testing.T) { /* ... */ })
        t.Run("nested2", func(t *testing.T) { /* ... */ })
    })

    t.Run("parallel", func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

Run:

```bash
$ go test -v ./...
=== RUN   TestX/subA
=== RUN   TestX/group
=== RUN   TestX/group/nested1
=== RUN   TestX/group/nested2
=== RUN   TestX/parallel
=== PAUSE TestX/parallel
...
=== CONT  TestX/parallel

$ go test -run 'TestX/group/nested1' ./...
$ go test -parallel=8 ./...
$ go test -p=4 ./...                # parallel across packages, not within
```

## Deep Dive

### `t.Run` mechanics

```go
ok := t.Run("name", func(t *testing.T) { ... })
```

`t.Run`:
1. Synchronously creates a child `*testing.T` whose name is `parent/name`.
2. If the child does **not** call `t.Parallel`, runs the body serially and returns its pass/fail.
3. If the child **does** call `t.Parallel`, registers it with the scheduler, returns immediately, returns `true` if the test eventually passes (but this is misleading — see below).

Return value: `true` if the subtest passed at the time `Run` returned. For parallel subtests, that's *not* a complete picture; treat `t.Run`'s return as advisory only when subtests are serial.

### `t.Parallel` semantics

```go
func TestX(t *testing.T) {
    t.Parallel()           // outer test itself runs in parallel with siblings
    t.Run("a", func(t *testing.T) {
        t.Parallel()       // subtest runs in parallel with other parallel subtests
    })
}
```

The runner has two phases per test:

1. **Serial** until `t.Parallel` is called.
2. **Parallel**: the test pauses, joins a pool, and resumes when scheduler picks it.

`-parallel=N` caps how many parallel-marked tests run simultaneously. Default `GOMAXPROCS`. Note: `-p=M` is different — it's the number of **packages** compiled and tested concurrently.

### Parent vs. child timing

The often-surprising order:

```go
func TestX(t *testing.T) {
    t.Cleanup(func() { fmt.Println("outer cleanup") })

    t.Run("sub", func(t *testing.T) {
        t.Cleanup(func() { fmt.Println("inner cleanup") })
        t.Parallel()
        fmt.Println("sub body")
    })

    fmt.Println("outer body end")
}
```

Output:

```
outer body end
outer cleanup           ← outer cleanup fires before parallel sub
sub body                ← sub runs after parent's serial code returns
inner cleanup
```

If you need outer cleanup to run *after* the parallel subtests, structure as:

```go
func TestX(t *testing.T) {
    rsrc := setup()
    t.Cleanup(rsrc.Close)        // fires before parallel subs (problem!)

    t.Run("group", func(t *testing.T) {        // wrapper subtest
        // group is serial, so it waits for parallel children
        t.Run("a", func(t *testing.T) { t.Parallel(); /* uses rsrc */ })
        t.Run("b", func(t *testing.T) { t.Parallel(); /* uses rsrc */ })
    })
    // when group's t.Run returns, all parallel children have completed
    // outer cleanup runs after this point — safe
}
```

The "wrapper subtest" trick is the standard pattern for shared resources across parallel siblings.

### The loop-variable trap (pre-1.22)

```go
// Go 1.21 and earlier:
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // tt is captured by reference; by the time this goroutine runs,
        // the loop has finished and tt holds the LAST element
    })
}
```

Workaround:

```go
for _, tt := range tests {
    tt := tt                      // shadow with a per-iteration copy
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

Since **Go 1.22**, the loop variable is per-iteration (governed by `go.mod`'s `go` directive). New code can drop the workaround.

`go vet` (`loopclosure` analyzer) flagged this bug; with 1.22 semantics, the analyzer is silent.

### `-parallel` vs. `-p`

| Flag         | What it controls                                                      |
|--------------|-----------------------------------------------------------------------|
| `-parallel=N`| Cap on parallel **tests within a package** (set by `t.Parallel`).     |
| `-p=M`       | Cap on **packages** being built/tested in parallel.                   |

Both default to `GOMAXPROCS`. For DB-backed tests sharing a single DB, lower `-parallel`:

```bash
$ go test -parallel=2 ./...     # only 2 parallel tests per package
```

### Sequential subtests still benefit

Even without parallelism, subtests:

- Give per-case names in failure output.
- Enable `-run TestX/case` targeting.
- Have independent `t.Cleanup` stacks.
- Don't stop sibling subtests when one fails.

So `t.Run` is worth using even when you don't plan to parallelize.

### `t.Skip` in subtests

```go
t.Run("integration", func(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping in short mode")
    }
    // ...
})
```

Only the subtest skips; siblings continue.

### Reusing `*testing.T` across goroutines

Inside a single test, spawn goroutines that call `t.Errorf` / `t.Log` freely — `T` is goroutine-safe for these methods. But:

- `t.FailNow` / `t.Fatal` / `t.Skip` must be called from the *test goroutine*. Calling them from a spawned goroutine panics with a "test.go: Goexit called from non-test goroutine" error (the `testinggoroutine` vet analyzer catches this).

```go
func TestX(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            if v := compute(); v != 0 {
                t.Errorf("compute = %v, want 0", v)  // OK
                // t.Fatalf — NOT OK
            }
        }()
    }
    wg.Wait()
}
```

### Cleanup ordering with parallel subtests

```go
func TestX(t *testing.T) {
    t.Cleanup(func() { fmt.Println("OUTER cleanup") })

    t.Run("parent", func(t *testing.T) {
        t.Cleanup(func() { fmt.Println("PARENT cleanup") })

        t.Run("child", func(t *testing.T) {
            t.Cleanup(func() { fmt.Println("CHILD cleanup") })
            t.Parallel()
        })

        // PARENT cleanup fires here — before child runs
    })

    // OUTER cleanup fires here — before parent's child runs
}
```

The cleanup stack is LIFO per `*testing.T`; tests *don't* await their parallel descendants for cleanup. Use the wrapper-subtest pattern to fix.

### `t.Parallel` and `t.Setenv`

```go
func TestX(t *testing.T) {
    t.Parallel()
    t.Setenv("FOO", "bar")    // panic
}
```

Setenv is process-global; running parallel tests with conflicting envs would race. Use one or the other.

### Subtest naming and `-run` regex

```bash
$ go test -run 'TestX/sub.*'              # all subs starting "sub"
$ go test -run 'TestX/sub$'               # exact sub
$ go test -run 'TestX/sub/nested'         # nested
$ go test -run 'TestX$/sub'               # outer anchored
```

The regex is matched against each path segment between slashes. Anchor with `$` to avoid prefix matches.

### Running a benchmark as a subtest

`b.Run("name", ...)` is the benchmark equivalent. See `10-testing/04-benchmarks.md`.

## Standard Library Hooks

- `testing.T.Run`, `T.Parallel`, `T.Cleanup`.
- `testing.B.Run` — benchmark subtests (same model).
- `testing.F.Add` (sort of) — fuzz seeds get reported as fuzz subtests.
- `cmd/vet` analyzers: `testinggoroutine` (Fatal from non-test goroutine), `loopclosure` (pre-1.22 capture bug).

## Real-World Patterns

### 1. Table-driven, parallel cases

```go
func TestEncoder(t *testing.T) {
    t.Parallel()
    tests := []struct{ name, in, want string }{ /* ... */ }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got := Encode(tt.in)
            if got != tt.want {
                t.Errorf("Encode(%q) = %q, want %q", tt.in, got, tt.want)
            }
        })
    }
}
```

### 2. Wrapper subtest for shared resource

```go
func TestDB(t *testing.T) {
    db := setupDB(t)             // expensive
    // problem: db closes via t.Cleanup before parallel sub-subs run

    t.Run("group", func(t *testing.T) {
        t.Run("insert", func(t *testing.T) { t.Parallel(); test(db) })
        t.Run("update", func(t *testing.T) { t.Parallel(); test(db) })
        t.Run("delete", func(t *testing.T) { t.Parallel(); test(db) })
    })
    // group blocks here for parallel children; db.Close runs after
}
```

### 3. Grouping by feature

```go
func TestUser(t *testing.T) {
    t.Run("creation", func(t *testing.T) {
        t.Run("valid", testValidCreation)
        t.Run("missing fields", testMissingFields)
    })
    t.Run("authentication", func(t *testing.T) {
        t.Run("valid", testValidAuth)
        t.Run("expired", testExpiredAuth)
    })
}
```

### 4. Skipping per-case under -short

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        if tt.slow && testing.Short() {
            t.Skip("slow case skipped in -short")
        }
        // ...
    })
}
```

### 5. Goroutine assertions

```go
func TestConcurrent(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            if compute(i) != i*2 {
                t.Errorf("compute(%d) wrong", i)
            }
        }(i)
    }
    wg.Wait()
}
```

### 6. Resetting state per case

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        store := newStore()
        t.Cleanup(store.Close)
        // ...
    })
}
```

### 7. Run only one parallel case

```bash
$ go test -run 'TestX/specific$' ./...
```

`t.Parallel()` still applies but with one test, the test runs alone.

## Anti-Patterns & Gotchas

**Cleanup in parent of parallel subtests.** Outer cleanup fires before parallel children. Wrap children in a serial subtest if you need ordering.

**Calling `t.Fatal` from a spawned goroutine.** Panics. Use `t.Errorf` plus a flag channel.

**Forgetting `t.Parallel` in subtests for a parallelized parent.** The parent declared parallel; subtests didn't — they run serially within the parallel parent.

**Pre-1.22 loop var capture.** Add `tt := tt` or bump `go.mod` to 1.22+.

**Running `t.Parallel` and `t.Setenv` in the same test.** Panics. Pick one.

**Assuming `t.Run` returns true after parallel subtest passes.** It returns immediately for parallel children; pass/fail isn't determined yet.

**Calling `t.Skip` after spawning work.** The work isn't cancelled by Skip. Skip early.

**Heavy `setup` in parallel subtests.** Each subtest pays the cost. Hoist common setup to the parent (with a wrapper subtest to manage timing).

**Long subtest names with newlines or special chars.** Renders awkwardly; `-run` regex gets ugly. Keep names short and `[a-z_-]`-friendly.

**Using `b.Run` on a single benchmark case.** Adds nesting for no value. Top-level `BenchmarkX` is simpler.

**Globally enabling `t.Parallel` without thinking about resources.** Parallelism over DB-backed tests can OOM the DB or exhaust connection pools.

**Relying on subtest ordering.** `t.Run` *registers* subtests in order; with parallelism, completion order is undefined.

## Performance Notes

- `t.Run` overhead per call: ~10 µs.
- `t.Parallel` overhead: ~50 µs (scheduling cost).
- Per-test fixture: dominated by setup, not test framework.
- `t.Cleanup` register: ~ns; runs in LIFO O(N) at test end.
- `-parallel=GOMAXPROCS` is default; useful to lower for resource-bound tests.
- Stop-the-world events between subtests: none; subtests share the package's test binary.

For test suites where each test spends most of its time on I/O (network, DB), `t.Parallel` can yield 5–10× wall-time speedups.

## How Big Companies Use It

- **Google** uses subtests + parallel extensively in stdlib and internal Go: https://go.googlesource.com.
- **Kubernetes** uses subtests for grouping; parallelism is conservative due to shared API server fixtures: https://github.com/kubernetes/kubernetes.
- **Uber** uses parallel subtests in `zap` and `dig`; documented in their style guide: https://github.com/uber-go/guide.
- **CockroachDB** uses parallel subtests + per-test transaction rollback for DB tests: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses parallel subtests in Terraform's helper test pattern: https://github.com/hashicorp/terraform.
- **Tailscale** uses parallel subtests + their internal `tstest` helpers: https://github.com/tailscale/tailscale.
- **The Go team** uses subtests in stdlib (`encoding/json`, `net/http`, `time`): https://github.com/golang/go.

## Source Code References

Pinned to `go1.26`.

- `T.Run`: [`src/testing/sub_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/sub_test.go), `src/testing/testing.go`.
- `T.Parallel`: [`src/testing/testing.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/testing.go).
- Scheduler: [`src/testing/run_example.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/run_example.go) and `testing.go` (search `release`, `match`).
- Loop-var change (Go 1.22): https://go.dev/wiki/LoopvarExperiment.
- `loopclosure` analyzer: [`golang.org/x/tools/go/analysis/passes/loopclosure`](https://github.com/golang/tools/tree/master/go/analysis/passes/loopclosure).
- `testinggoroutine` analyzer: [`golang.org/x/tools/go/analysis/passes/testinggoroutine`](https://github.com/golang/tools/tree/master/go/analysis/passes/testinggoroutine).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Using Subtests and Sub-benchmarks" (Marcel van Lohuizen, Go blog): https://go.dev/blog/subtests.
- "Go 1.22 loop variable" (Russ Cox): https://go.dev/blog/loopvar-preview.
- "Parallel testing in Go" (Filippo Valsorda): https://blog.filippo.io.
- "Testing best practices" (Mat Ryer): https://medium.com/@matryer.
- Dave Cheney, "Don't use `t.Parallel` everywhere": https://dave.cheney.net.

## Exercises / Self-Check

1. Write a `TestX` that runs three parallel subtests sharing a resource. Use the wrapper-subtest pattern to ensure cleanup runs after them all.
2. Trigger the pre-1.22 loop-var bug intentionally (set `go.mod`'s `go` to 1.21). Observe; then bump to 1.22.
3. Use `-parallel=1` and `-parallel=GOMAXPROCS` on a slow integration test. Time the difference.
4. Spawn 10 goroutines inside a test. Call `t.Errorf` from each; observe that all 10 errors are reported.
5. Use `-run TestX/specific$` to target one subtest. Why does the `$` matter?
