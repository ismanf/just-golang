# The Race Detector

## TL;DR

`go test -race`, `go run -race`, and `go build -race` enable Go's race detector, which is a thin wrapper over Google's **ThreadSanitizer (TSan)**. At runtime it instruments every memory access to track happens-before edges; any pair of conflicting accesses (one is a write, no edge between them) is reported with both stack traces. **No false positives**, but **false negatives are common** — a race that didn't fire on this run is still a bug. Run it in CI on every test. Single biggest gotcha: the race detector has ~5–10× CPU and ~2× memory overhead, and binary size grows ~2–10× — so do not ship `-race` binaries to production except in deliberate canaries.

## Mental Model

```
go test -race
   |
   v
compiler inserts __tsan_read/write/acquire/release calls at each access
   |
   v
runtime/race links libtsan (precompiled per OS/arch)
   |
   v
program runs:
   - every read/write checked against the shadow memory
   - every channel send/recv, lock/unlock, atomic op
     emits an acquire/release event
   - on detected race -> print report and (by default) crash
```

The race detector "wins" by being correct, not by being fast. It is the closest thing Go has to a proof of memory safety for concurrent code.

## Syntax & Basic Usage

```
go test -race ./...
go test -race -count=10 -run TestFlaky
go build -race -o myserver ./cmd/myserver
go run -race main.go
```

A racy program:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var x int
	var wg sync.WaitGroup
	for range 2 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			x++ // RACE: concurrent write to x
		}()
	}
	wg.Wait()
	fmt.Println(x)
}
```

`go run -race main.go` produces:

```
==================
WARNING: DATA RACE
Read at 0x00c0000180e0 by goroutine 7:
  main.main.func1()
      /tmp/race.go:13 +0x37

Previous write at 0x00c0000180e0 by goroutine 6:
  main.main.func1()
      /tmp/race.go:13 +0x4c
==================
```

The detector knows the address, the access kind (read/write), the goroutine, and the stack of *both* sides of the race.

## Deep Dive

### How it works under the hood

The race detector instruments memory accesses at compile time. For each access, the compiler emits a call into the TSan runtime, which:
1. Looks up shadow memory associated with the address.
2. Checks the access against the current goroutine's "vector clock" (a per-goroutine timestamp on every other goroutine's view).
3. If the access conflicts with a previous one without a happens-before edge → report.
4. Updates the shadow.

Synchronization primitives emit explicit acquire/release operations:
- `mu.Lock` → `tsan_acquire(mu)`.
- `mu.Unlock` → `tsan_release(mu)`.
- Channel send → `tsan_release(chan)`. Receive → `tsan_acquire(chan)`.
- Atomic load → acquire; store → release; CAS → both.
- `go f()` → release(g) on parent, acquire(g) on child.

So happens-before is reified into vector-clock operations.

### Vector clocks and overhead

Each goroutine carries a vector clock of N entries (one per goroutine). On each sync op, the clocks merge per the HB rules. This is why the race detector's memory grows roughly with goroutine count × tracked memory.

In practice: 5–10× slowdown, 2× memory, 2–10× binary size. The slowdown is per-access, so allocation-heavy hot loops feel it more.

### Build constraints

`-race` requires CGo (TSan is a C library). It's supported on:
- linux/amd64, linux/arm64, linux/ppc64le, linux/s390x
- darwin/amd64, darwin/arm64
- freebsd/amd64
- netbsd/amd64
- openbsd/amd64
- windows/amd64

If you target a platform that doesn't support `-race`, just disable it on that build (`go test -race` in CI on Linux usually suffices).

### What it does NOT catch

- **Races on memory it didn't observe.** If a code path isn't exercised, the race isn't reported.
- **Bugs that are not races.** Deadlocks, livelocks, missing-cancellation leaks — race detector won't help. Use `goleak`, `pprof`, deadlock detectors.
- **Misuse of atomic ops in a way that's allowed by the memory model.** E.g., publishing a half-built struct via atomic.Pointer — there's no race, but the data is wrong.
- **Races in Cgo C code.** Only Go-instrumented code is checked.
- **Races in code compiled without `-race`.** If you link a third-party `.a` not built with `-race`, instrumented Go code can race with uninstrumented C / asm and not be reported.

### False negatives are about coverage, not correctness

If `-race` doesn't fire on a run, it means *this run* did not exhibit a race. A different interleaving might. **Run tests many times** with `-count=N` (or use `stress` from `golang.org/x/tools/cmd/stress`) to increase coverage. Add `runtime.GOMAXPROCS(1)` and then `(4)` and then `(16)` in CI to vary scheduling.

### Exit behavior

By default, the race detector prints a report and the program continues. Set `GORACE="halt_on_error=1"` (or pass via `-gcflags`) to crash on first race — this is what CI should use:

```
GORACE="halt_on_error=1" go test -race ./...
```

### `GORACE` knobs

```
GORACE="log_path=/tmp/race halt_on_error=1 history_size=7"
```

- `log_path`: write reports to files instead of stderr.
- `halt_on_error`: 1 = crash on first race.
- `history_size`: 0..7, depth of access history kept per goroutine. Higher = better reports, more memory.
- `exitcode`: exit code when a race is found (default 66).
- `strip_path_prefix`: trim source paths in reports.

### Production canaries

Because the slowdown is significant, you don't run `-race` on all production traffic. But running it on **1% of traffic** (or in a single canary pod) for 24 hours can catch real races that escaped tests. Tailscale, CockroachDB, and Cloudflare have all written about this.

### The "memory races" report format

```
WARNING: DATA RACE
Read at 0x... by goroutine N:
   <stack>
Previous write at 0x... by goroutine M:
   <stack>
Goroutine N (running) created at:
   <stack>
Goroutine M (finished) created at:
   <stack>
```

The "created at" stacks help you trace why a goroutine exists at all — invaluable for "where did this come from?" races.

### What about `-msan` and `-asan`?

Go supports `-msan` (memory sanitizer, uninitialized reads in C linked code) and `-asan` (address sanitizer, out-of-bounds in C/Go) since 1.18+. They're complementary to `-race`. Most pure-Go projects only need `-race`.

## Standard Library Hooks

- `runtime/race` — the Go-side TSan glue. Not for direct user use; `sync` and `runtime` call into it.
- `go test -race` — the canonical use.
- `GORACE` env var — config.
- `golang.org/x/tools/cmd/stress` — run a test repeatedly until it fails, often paired with `-race`.
- `go vet -copylocks`, `-shadow` etc. — static analyses that catch some race causes before the runtime does.

## Real-World Patterns

### 1. CI matrix with -race

```yaml
# .github/workflows/test.yml
- run: go test -race -count=3 -shuffle=on ./...
  env:
    GORACE: "halt_on_error=1"
```

`-shuffle=on` (since 1.17) randomizes test order; `-count=3` runs each three times. Combined with `-race`, this catches the long tail of "only races every fourth run" bugs.

### 2. stress for flake hunting

```
go test -race -c -o test.bin ./mypkg
stress -p 4 ./test.bin -test.run TestFlaky
```

`stress` runs the binary in parallel, in a loop, until something fails. The compiled-once approach avoids re-linking each iteration.

### 3. Race-canary in prod

```go
//go:build race

package main

import "runtime"

func init() { runtime.GC() } // your race-build-only init
```

A build with `//go:build race` files lets you change behavior under `-race` — for example, registering a Sentry hook for race reports.

### 4. Annotated mutex doc

```go
// State is safe for concurrent use under State.mu.
//
// Goroutines must NOT call any method while holding mu.
type State struct {
	mu      sync.Mutex
	online  map[string]bool
}
```

Document the locking discipline. The race detector enforces *correctness* of the discipline you write; the doc tells humans what that discipline is.

### 5. `_ = ctx.Done()` after refactor

Sometimes the race detector reports a leak that boils down to "this goroutine accessed `x` after `Wait()` returned." The fix is usually a missing `<-ctx.Done()` or `wg.Wait()` before reading shared state. The detector points exactly where.

## Anti-Patterns & Gotchas

**Skipping `-race` because "it's slow."** Slow in CI, free in correctness. Run nightly if cost matters.

**Disabling `-race` for "flaky" tests.** A `-race`-only flake is a real race. Find it.

**Believing absence of report = absence of race.** No. Add `-count=N` and stress to widen coverage.

**`-race` in benchmarks.** Useless — benchmarks measure speed, race detector slows them by 5×. Bench without; race-test the same code paths separately.

**Shipping `-race` binaries to prod.** Acceptable for canaries; not for the whole fleet. Operational overhead aside, error budgets get eaten by GC pauses doubled by race overhead.

**Racing on `time.Now()`.** The detector knows about `time` package internals. Some packages have race-suppressed special cases (`runtime/race.Disable/Enable`); user code should not call those.

**Mistaking the report's goroutine ID for OS thread ID.** They are independent. The numbers reset per program.

**Forgetting that a race report can name a *finished* goroutine.** The "previous write by goroutine M" goroutine may be gone; the detector still has the access record.

**Building a `c-shared` library with `-race` and linking from C.** Possible but rare. Make sure your C code follows the threading rules TSan expects.

**Suppressing reports with `//go:nocheckptr` or `//go:nosplit`.** These directives have nothing to do with the race detector; they're unrelated runtime hints.

## Performance Notes

- Slowdown: 5–10× CPU (varies). Memory-heavy workloads suffer the most.
- Memory: ~2× heap, plus shadow memory proportional to addressable memory touched.
- Binary size: 2–10×. Symbols are not stripped by default.
- Some operations (channel sends, atomic ops) cost a few hundred ns extra because of the TSan call.
- Real-world experience: Kubernetes API server runs ~3× slower under `-race`; CockroachDB ~5×; small services often closer to 2×.

## How Big Companies Use It

- **The Go team itself**: every CL passes `go test -race ./...` for the runtime and stdlib. Many runtime races have been found this way.
- **Kubernetes**: `-race` runs in CI; the Kubernetes team has documented finding subtle races in informers and controllers via the detector.
- **CockroachDB**: every CI run includes `-race`. They've blogged about finding distributed-systems-level races: https://www.cockroachlabs.com/blog/.
- **Cloudflare**: race-detector canaries in production, see https://blog.cloudflare.com/.
- **Tailscale**: full-suite `-race` plus `go test -count=20` for known-flaky tests, per Brad Fitzpatrick's GopherCon talks.
- **Discord** has called out the race detector as essential during their Go years: https://discord.com/blog/.
- **Uber's `goleak`** (a leak detector) complements `-race` by catching goroutine leaks the detector wouldn't see.

## Source Code References

Pinned to `go1.26`.

- Go integration: [`src/runtime/race/`](https://github.com/golang/go/tree/master/src/runtime/race) — README is excellent.
- Compiler instrumentation: [`src/cmd/compile/internal/ssagen/race.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/race.go).
- Race acquire/release calls inside sync: e.g. [`src/sync/mutex.go`](https://github.com/golang/go/blob/master/src/sync/mutex.go), search `race.Acquire`.
- Channel races: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go), search `raceacquire`.
- The TSan C library is upstream LLVM: https://github.com/llvm/llvm-project/tree/main/compiler-rt/lib/tsan.

## Further Reading

- Go blog, "Introducing the Go race detector" (Dmitry Vyukov, 2013): https://go.dev/blog/race-detector
- Race detector internals (Dmitry Vyukov's notes): https://github.com/golang/go/blob/master/src/runtime/race/README
- ThreadSanitizer paper (Serebryany & Iskhodzhanov, 2009): https://research.google/pubs/pub35604/
- Russ Cox, "Updating the Go Memory Model" (2022): https://research.swtch.com/gomm
- `golang.org/x/tools/cmd/stress`: https://pkg.go.dev/golang.org/x/tools/cmd/stress
- Uber's `goleak` package: https://github.com/uber-go/goleak
- Dave Cheney, "Why you should always run tests with -race": https://dave.cheney.net/

## Exercises / Self-Check

1. Write a one-goroutine program that has a race but only fires reliably under high concurrency. How would `-count=N` help?
2. Why does the race detector use vector clocks rather than a single global counter?
3. What's the difference between `-race` and `-msan`? When would you want each?
4. Construct a program where the race detector produces no output but the program is incorrect. What does that say about race vs other concurrency bugs?
5. Profile your test suite under `-race`. Where does the cost go? Is the matrix tradeoff worth it for your team?
