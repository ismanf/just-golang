# `testing/synctest`

## TL;DR

`testing/synctest` (stable in **Go 1.25**, experimental in 1.24 behind `GOEXPERIMENT=synctest`) runs a function in an isolated "bubble" where time is a **fake** clock controlled by the runtime, and the test waits for all goroutines in the bubble to become idle before advancing. This makes concurrency tests **deterministic** — `time.Sleep(time.Hour)` returns immediately, races between timers and goroutines play out reproducibly, and there's no real wall-clock waiting. The biggest gotcha: the bubble's notion of "idle" excludes goroutines that are blocked on real I/O (network, file). If your code-under-test reads from a real socket inside the bubble, the bubble will deadlock-detect; either fake the I/O or restructure.

## Mental Model

```
synctest.Run(func() {
   // inside this bubble:
   //   - time.Now() returns fake time, starting at a fixed epoch
   //   - time.Sleep/After/NewTimer/Ticker use fake clock
   //   - all goroutines in this call are members of the bubble
   //   - synctest.Wait() blocks until all bubble Gs are "durably blocked"
   //   - then the next pending fake timer fires
})
```

A bubble is a goroutine group with:
1. A **fake clock** — `time.Now`, `time.Sleep`, `time.After`, `time.NewTimer`, `time.NewTicker`, `time.AfterFunc` all use it.
2. A **durable-block predicate** — the bubble's goroutines must all be in synchronization waits (channel/lock/select/cond/wg) that involve only other bubble goroutines. Then `synctest.Wait` returns and the bubble advances time.

When the bubble's `Run` callback returns, the bubble dies; if any of its goroutines haven't finished, `Run` panics with a deadlock report.

## Syntax & Basic Usage

```go
package mypkg

import (
	"testing"
	"testing/synctest"
	"time"
)

func TestTickerEmitsEvery100ms(t *testing.T) {
	synctest.Run(func() {
		got := []time.Time{}
		done := make(chan struct{})

		go func() {
			t := time.NewTicker(100 * time.Millisecond)
			defer t.Stop()
			for i := 0; i < 5; i++ {
				got = append(got, <-t.C)
			}
			close(done)
		}()

		<-done
		if len(got) != 5 {
			t.Fatalf("got %d ticks", len(got))
		}
	})
}
```

No actual half-second wait — the bubble fast-forwards the fake clock and the test returns immediately.

## Deep Dive

### Two functions

`testing/synctest` exports exactly two:

```go
func Run(f func())  // run f in a new bubble; block until all bubble Gs finish.
func Wait()         // wait for all bubble Gs to be durably blocked, then advance time.
```

You typically only need `Run`. `Wait` is for when you want to assert intermediate state.

### Fake clock semantics

Inside a bubble:
- `time.Now()` starts at a fixed epoch (the same across runs — currently `2000-01-01 00:00:00 UTC`).
- `time.Sleep(d)` blocks the bubble goroutine on a fake timer; the bubble advances the fake clock to fire the next due timer when all bubble Gs are durably blocked.
- `time.After`, `time.NewTimer`, `time.NewTicker`, `time.AfterFunc` likewise.
- `time.Since(t)` works correctly because `time.Now()` is consistent.

Outside the bubble, time is real. Don't share clocks between bubbles and the outside.

### Durable blocking

A goroutine is "durably blocked" when it's:
- Receiving from / sending on a channel that only bubble Gs can interact with.
- Waiting on a mutex/cond/wg that only bubble Gs can release.
- Blocked on `<-time.After(...)` etc. inside the bubble.

It is **not** durably blocked when:
- Doing real I/O (read/write on a real fd).
- Holding a syscall or sleeping in syscall-land.
- Waiting on a channel that an outside-bubble goroutine could send to.

The bubble advances time only when *all* bubble goroutines are durably blocked. If one is in real I/O, the bubble waits indefinitely → eventual deadlock panic.

### Deadlock detection

If the bubble reaches a state where every G is durably blocked **and** there are no pending fake timers to fire, the bubble panics with a goroutine dump. This is the classic concurrency deadlock — your code is genuinely stuck. The bubble surfaces it immediately instead of letting the test hang.

### `synctest.Wait()`

```go
synctest.Run(func() {
	go producer()
	go consumer()
	synctest.Wait() // block until both are stuck waiting
	// inspect state, assert invariants
})
```

`Wait` returns when every bubble goroutine is durably blocked. Useful for two-phase tests: spin up workers, wait for them to settle, fire a stimulus, wait again, assert.

### What can't go in a bubble

- Code that does real network I/O (`net.Dial`, `http.Get`).
- Code that calls `runtime.Goexit` in a non-bubble goroutine.
- Code that uses non-Go scheduling primitives (cgo callbacks that take a long time).
- Multiple bubbles running at the same time in the same test (they have to be sequential).

For network code, replace `net.Dialer` with `httptest.Server` or an in-memory net.Conn pair (`net.Pipe()` works inside a bubble because both ends are bubble Gs).

### History

- Proposed: https://go.dev/issue/67434 (Russ Cox / Damien Neil, 2024).
- Landed in **Go 1.24** behind `GOEXPERIMENT=synctest`, under the path `testing/synctest`.
- Stabilized in **Go 1.25** with no experiment flag needed.

Pre-1.24, this functionality required third-party libraries (`benbjohnson/clock`, `jonboulle/clockwork`) that all required your code to take an injectable clock interface. `synctest` works on stdlib `time` without any code changes.

### Bubble-aware sleeping under net/http

The Go team has been making more stdlib packages bubble-aware. `net.Pipe` works. `time.AfterFunc` works. `context.WithTimeout` works (because the underlying timer is bubble-aware). `net/http` doesn't yet fully work with real sockets, but `httptest.NewServer` paired with `net.Pipe` is the workaround.

### Compatibility with `t.Parallel`

A `t.Parallel` test inside a `synctest.Run` makes no sense — the bubble has only one entry point. You can have many tests each call `synctest.Run` and run them in parallel; that's fine — each bubble is independent.

### Bubble nesting

You cannot nest `synctest.Run` calls — calling `Run` from inside a bubble panics. If you need to run a sub-bubble, structure your test so each `Run` is at the top level.

## Standard Library Hooks

- `testing/synctest.Run` and `Wait`.
- `time.*` (Now, Sleep, After, NewTimer, NewTicker, AfterFunc) — bubble-aware.
- `context.WithTimeout/WithDeadline/AfterFunc` — bubble-aware via the time package.
- `sync.Mutex`, `sync.Cond`, channels, `sync.WaitGroup`, `errgroup` — all work; their blocks count as durable.
- `net.Pipe` — in-memory pair, both ends inside the bubble.
- `httptest` — partially bubble-friendly via `net.Pipe`.

## Real-World Patterns

### 1. Timer/ticker assertion

```go
func TestRateLimiter(t *testing.T) {
	synctest.Run(func() {
		l := newLimiter(10 * time.Millisecond) // tick every 10ms
		got := 0
		done := make(chan struct{})
		go func() {
			for range 5 {
				l.Wait()
				got++
			}
			close(done)
		}()
		<-done
		// fake clock advanced 50ms; test takes ~0 wall time
		if got != 5 { t.Fatal("ticked wrong") }
	})
}
```

### 2. Timeout coverage without slow tests

```go
func TestRequestTimesOutAfter5s(t *testing.T) {
	synctest.Run(func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		err := doWork(ctx)
		if !errors.Is(err, context.DeadlineExceeded) {
			t.Fatalf("got %v", err)
		}
	})
}
```

The 5-second deadline expires in zero wall-clock time. Tests for timeouts become tractable instead of `t.Skip("too slow")`.

### 3. Race against the clock

```go
func TestSlowResponseLosesToFastFailure(t *testing.T) {
	synctest.Run(func() {
		ctx, cancel := context.WithCancel(context.Background())
		defer cancel()
		results := make(chan error, 2)
		go func() {
			time.Sleep(time.Hour) // fake — won't actually wait
			results <- nil
		}()
		go func() {
			time.Sleep(time.Second)
			cancel()
			results <- errors.New("failed fast")
		}()
		var errs []error
		errs = append(errs, <-results)
		errs = append(errs, <-results)
		// Inspect interleaving deterministically
	})
}
```

Determinism: the bubble fires timers in order, so this test always plays out the same way.

### 4. Cond + state machine

```go
func TestStateTransitions(t *testing.T) {
	synctest.Run(func() {
		sm := newStateMachine()
		go sm.Run()
		synctest.Wait() // both Goroutines settle into their initial waits
		sm.Trigger("connect")
		synctest.Wait()
		if sm.State != Connected { t.Fatal("expected Connected") }
		sm.Trigger("disconnect")
		synctest.Wait()
		if sm.State != Idle { t.Fatal("expected Idle") }
	})
}
```

`Wait` lets you stop and inspect between stimuli — no `time.Sleep(50ms)` flakes.

### 5. Backoff + jitter without waiting

```go
func TestBackoffEventuallySucceeds(t *testing.T) {
	synctest.Run(func() {
		attempts := 0
		err := backoff(context.Background(), func() error {
			attempts++
			if attempts < 5 { return errors.New("nope") }
			return nil
		})
		if err != nil { t.Fatal(err) }
		if attempts != 5 { t.Fatalf("got %d", attempts) }
	})
}
```

A backoff that sleeps 100ms, 200ms, 400ms, 800ms, 1.6s would be a 3.1s test. In a bubble it's instantaneous.

## Anti-Patterns & Gotchas

**Real I/O inside a bubble.** If your code-under-test calls `net.Dial("tcp", "example.com:80")`, the goroutine isn't durably blocked (it's in syscall land), so the bubble hangs. Inject a `net.Conn` from `net.Pipe`.

**File I/O inside a bubble.** Same as network — real file syscalls aren't durable blocks. For tests, use `testing/fstest` or `afero`-style in-memory FS.

**Mixing real time with fake.** Don't pass values from `time.Now()` inside the bubble to code outside, or vice versa.

**Forgetting that goroutines spawned by your code are in the bubble.** Every `go` statement made by a goroutine inside the bubble joins the bubble. That's usually what you want — but it means a goroutine you didn't expect to control is suddenly governed by the fake clock.

**Calling `Wait` before any goroutine has had a chance to block.** Doesn't matter much — `Wait` will return immediately if everyone is already idle. But it's a hint that the test structure is off.

**Calling `synctest.Run` from outside `go test`.** The package is in `testing/`, intended for tests. It works in `main`, but you usually shouldn't.

**Treating "the bubble hangs" as a synctest bug.** It's almost always a real deadlock in the code or a "blocked on real I/O" mistake.

**Nesting `synctest.Run`.** Panics.

## Performance Notes

- A bubble has near-zero overhead beyond what a real test would have, minus the wall-clock wait. A 60-second timeout test becomes a few µs.
- Fake-clock timer events are O(log n) via a heap.
- Memory: one bubble allocates a small struct (~100 bytes) plus per-G bookkeeping.
- CI savings can be substantial: rewriting timeout/backoff/retry tests with synctest can cut a CI run from minutes to seconds.

## How Big Companies Use It

- **The Go team** uses `synctest` internally in `net/http`, `database/sql`, and `cmd/go` tests since 1.24.
- **Tailscale** has migrated many of their goroutine-heavy tests to synctest. They write about how it eliminated whole categories of flake.
- **CockroachDB** is gradually adopting synctest for raft and SQL session tests.
- **`net/http`'s tests for `http.Transport` connection reuse and idle timeouts** are now bubble-based.
- **`context` tests** for `WithTimeout`/`AfterFunc` use synctest extensively.
- Various open-source projects: `cenkalti/backoff` test suite, `prometheus/client_golang` rate-limit tests.

## Source Code References

Pinned to `go1.26`.

- Package: [`src/testing/synctest/synctest.go`](https://github.com/golang/go/blob/master/src/testing/synctest/synctest.go).
- Runtime support: [`src/runtime/synctest.go`](https://github.com/golang/go/blob/master/src/runtime/synctest.go).
- Bubble-aware `time` integration: [`src/time/sleep.go`](https://github.com/golang/go/blob/master/src/time/sleep.go) — search `synctest`.
- Bubble-aware `context.AfterFunc`: [`src/context/context.go`](https://github.com/golang/go/blob/master/src/context/context.go).
- Proposal: https://go.dev/issue/67434.

## Further Reading

- Proposal (Russ Cox, Damien Neil): https://go.dev/issue/67434
- Go 1.24 release notes (experimental): https://go.dev/doc/go1.24#testing-synctest
- Go 1.25 release notes (stable): https://go.dev/doc/go1.25 (search for `synctest`)
- Damien Neil's writeup on synctest at https://research.swtch.com/ (commentary)
- Tailscale, "Goodbye, flaky tests": https://tailscale.com/blog (recent post on synctest adoption)
- benbjohnson/clock (pre-stdlib alternative): https://github.com/benbjohnson/clock
- jonboulle/clockwork: https://github.com/jonboulle/clockwork

## Exercises / Self-Check

1. Rewrite a test that uses `time.Sleep(2 * time.Second)` for retry to use `synctest.Run`.
2. Why does the bubble panic when all Gs are blocked and no timers are pending? What does that detect?
3. Construct a synctest test that races a 1-hour timer against a 1-second timer. Which fires first? Why is it deterministic?
4. What needs to change in code that calls `net.Dial("tcp", "...")` to be testable under synctest?
5. How does `synctest.Wait` differ from `time.Sleep(0)` or `runtime.Gosched()`?
