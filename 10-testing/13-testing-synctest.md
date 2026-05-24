# `testing/synctest` — Synthetic-Time Concurrency Testing (1.24+)

## TL;DR

**`testing/synctest`** (added experimentally in Go 1.24, graduating in 1.25) gives you **synthetic time** for tests that exercise concurrent or time-based code. Inside a `synctest.Run(func() { ... })` (or 1.25's `synctest.Test(t, func(t *testing.T) { ... })`) bubble, `time.Sleep`, `time.NewTimer`, `time.NewTicker`, `time.After`, and `context.WithTimeout` advance only when **all goroutines are blocked on time**; the runtime advances the synthetic clock to the next pending timer instantly. Result: a `time.Sleep(1 * time.Hour)` test runs in microseconds, and complex multi-goroutine coordination is deterministic (no flaky polling, no real wall-clock waits). The bubble has strict rules: all goroutines you spawn must finish (or be blocked on synctest-aware primitives) before the bubble exits. Non-synctest-aware blocking (network, raw `runtime.Gosched`, `select{}` without a time arm) is *not* coordinated and will deadlock the bubble.

## Mental Model

```
   synctest.Run(func() {                          (or 1.25's synctest.Test)
       go worker()
       go scheduler()
       time.Sleep(1 * time.Hour)     ── synthetic
       // ... at the end, all goroutines must be terminated or
       // blocked on synctest-aware primitives
   })

   Inside the bubble:
       ┌───────────────────────────────────────────┐
       │  Synthetic clock                           │
       │  All time.* primitives use it             │
       │  Advances only when ALL goroutines are    │
       │  blocked on time (or other synctest sync) │
       │                                            │
       │  Result: tests with sleeps run instantly. │
       │  No race between "did the goroutine wake?"│
       │  and "did the test check?".                │
       └───────────────────────────────────────────┘
```

The runtime detects "everyone's waiting on time" → fast-forwards to the earliest timer → fires it → repeats. Wall-clock progress is irrelevant.

## Syntax & Basic Usage

```go
// Go 1.24 (experimental)
import "testing/synctest"

func TestSleepInstant(t *testing.T) {
    synctest.Run(func() {
        start := time.Now()
        time.Sleep(time.Hour)
        if time.Since(start) != time.Hour {
            t.Errorf("expected 1h synthetic, got %v", time.Since(start))
        }
    })
}

// Go 1.25 (graduated; preferred form)
func TestSleepInstant(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        start := time.Now()
        time.Sleep(time.Hour)
        if time.Since(start) != time.Hour {
            t.Errorf("expected 1h synthetic, got %v", time.Since(start))
        }
    })
}
```

Build/run:

```bash
# Go 1.24
$ GOEXPERIMENT=synctest go test ./...

# Go 1.25+
$ go test ./...
```

The 1.24 version requires the `GOEXPERIMENT=synctest` env var; 1.25 made it a regular feature.

## Deep Dive

### What synctest replaces

The classic anti-pattern:

```go
func TestEventualConsistency(t *testing.T) {
    cache := NewCache(100 * time.Millisecond)
    cache.Set("key", "value")

    time.Sleep(150 * time.Millisecond)    // pad for safety; flaky

    if _, ok := cache.Get("key"); ok {
        t.Errorf("entry should have expired")
    }
}
```

Problems:
- Test takes 150 ms wall time.
- On slow CI, 150 ms might not be enough → flaky.
- On fast machines, you waste 150 ms × number-of-tests.

With synctest:

```go
func TestEventualConsistency(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        cache := NewCache(100 * time.Millisecond)
        cache.Set("key", "value")

        time.Sleep(150 * time.Millisecond)    // synthetic; instant
        synctest.Wait()                        // wait for goroutines to settle (1.25)

        if _, ok := cache.Get("key"); ok {
            t.Errorf("entry should have expired")
        }
    })
}
```

Runtime: microseconds. Deterministic.

### What's covered

Inside a bubble, these use synthetic time:

- `time.Now`
- `time.Sleep`
- `time.NewTimer`, `time.AfterFunc`, `time.NewTicker`
- `time.After`
- `time.Until`, `time.Since`
- `context.WithTimeout`, `context.WithDeadline`
- Channel operations involving `time.After` arms.

Outside the bubble: real time.

### `synctest.Wait` (1.25+)

```go
synctest.Test(t, func(t *testing.T) {
    go worker()
    synctest.Wait()    // block until all bubble-goroutines are blocked or done
    // assertions
})
```

`Wait` returns when **every goroutine in the bubble is in a "blocked" state**: blocked on a channel, a mutex, a time call, or finished. Useful for "let the goroutines settle before I check".

Without `Wait`, you'd need ad-hoc synchronization (a channel signal, a `sync.WaitGroup`).

### The "all goroutines blocked" trigger

Synthetic time only advances when **every goroutine in the bubble is blocked**. If any goroutine is actively running (e.g., in a busy loop), the clock doesn't move:

```go
synctest.Test(t, func(t *testing.T) {
    go func() {
        for { /* busy loop */ }       // never blocks; clock stuck
    }()
    time.Sleep(time.Hour)             // never returns
})
```

Will hang. The bubble has a 1-minute wall-clock timeout to detect this.

### What doesn't work in a bubble

- **`net` package I/O**: real network blocks the goroutine in the kernel; synctest can't see it.
- **`os/exec`** child processes: outside the bubble.
- **`runtime.LockOSThread`**: not synctest-aware.
- **CGO calls**: opaque to the scheduler.
- **`syscall`-based file I/O**: real blocking.

For these, the bubble pattern is "do the real work outside, do the timing inside":

```go
synctest.Test(t, func(t *testing.T) {
    // Inside: timer + coordination logic
    ticker := time.NewTicker(time.Hour)
    select {
    case <-ticker.C:
        // ...
    }
})
```

### `synctest.Run` vs. `synctest.Test`

- **`synctest.Run(func())`** — the original 1.24 form; doesn't accept a `*testing.T`.
- **`synctest.Test(t, func(t *testing.T))`** — 1.25's form; integrates with the test framework. Inside the func, the `t` is a special variant that runs cleanups inside the bubble.

Prefer `synctest.Test` going forward; `Run` may eventually be deprecated.

### Goroutine accounting

When the bubble exits, **all goroutines spawned inside it must have finished** or be blocked on synctest-aware primitives. Otherwise the bubble panics:

```
panic: synctest: goroutine has not finished
```

Pattern: use `synctest.Wait` before bubble exit if you have lingering goroutines that *will* eventually block (e.g., a periodic worker).

For workers you want to stop:

```go
ctx, cancel := context.WithCancel(context.Background())
go worker(ctx)
// ... test logic
cancel()
synctest.Wait()    // wait for worker to exit
```

### Testing context timeouts

```go
synctest.Test(t, func(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    err := doWork(ctx)
    if !errors.Is(err, context.DeadlineExceeded) {
        t.Errorf("err = %v, want DeadlineExceeded", err)
    }
})
```

Runs instantly. Real time would wait 1 second per test.

### Testing tickers

```go
synctest.Test(t, func(t *testing.T) {
    ticks := make(chan time.Time, 10)
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()
    go func() {
        for t := range ticker.C { ticks <- t }
    }()

    time.Sleep(5 * time.Minute)
    synctest.Wait()

    if len(ticks) != 5 {
        t.Errorf("got %d ticks, want 5", len(ticks))
    }
})
```

Five ticks happen instantly; verified deterministically.

### Replacing clock-injection mocks

Pre-synctest pattern: inject a `Clock` interface, fake it for tests (`10-testing/09-mocking-strategies.md`).

```go
type Clock interface { Now() time.Time; Sleep(d time.Duration) }

func New(c Clock) *Cache { ... }

// test
var fakeNow time.Time
c := &fakeClock{now: &fakeNow}
cache := New(c)
fakeNow = fakeNow.Add(time.Hour)
```

With synctest, you don't need the interface:

```go
synctest.Test(t, func(t *testing.T) {
    cache := New()                  // uses real time.Now/time.Sleep
    time.Sleep(time.Hour)            // synthetic
    // ...
})
```

The production code uses standard `time` calls; tests get fake time for free.

### Interaction with `t.Cleanup`

In 1.25's `synctest.Test`, the `t.Cleanup` registered inside the bubble runs inside the bubble. Synthetic time still applies during cleanup.

In 1.24's `Run`, you don't have `t` directly; manage cleanup with `defer` inside the func.

### Performance

A test that would sleep 10 minutes in real time runs in microseconds (synchronization + bookkeeping). For a benchmark of timer-heavy code, synctest can run thousands of "minutes" per second.

### When *not* to use synctest

- Tests that exercise real I/O timing (network latency, disk).
- Tests that verify wall-clock behavior intentionally (e.g., a metrics counter that resets on real-time intervals).
- Tests that depend on the OS scheduler (rare; usually code smell).

For everything else with timers: synctest.

### `synctest.Wait` vs. `time.Sleep`

```go
// Anti-pattern
go worker()
time.Sleep(time.Hour)        // hope worker is done by now

// Correct
go worker()
synctest.Wait()              // wait for worker to be blocked or done
```

`time.Sleep` advances the clock but doesn't guarantee goroutines have run. `Wait` does.

### Real time inside a bubble

There's no escape hatch (intentionally). If you need real time, exit the bubble:

```go
// Real time outside:
realStart := time.Now()
synctest.Test(t, func(t *testing.T) {
    time.Sleep(time.Hour)
})
realDuration := time.Since(realStart)
// realDuration is typically a few ms.
```

### Race detector

```bash
$ go test -race ./...
```

Works inside synctest bubbles. Useful for detecting actual races in the code under test (synctest doesn't introduce races itself).

### Comparing to alternatives

| Approach              | Pros                       | Cons                                 |
|-----------------------|----------------------------|--------------------------------------|
| Clock injection (mock)| Works on any Go version    | Pollutes production code with interface |
| `time.Sleep` padding  | No deps                    | Flaky, slow                          |
| `synctest`            | Real `time.*`, instant     | Requires Go 1.24+ / 1.25+            |
| Time-mocking libraries (`clockwork`) | Test-only abstraction | Still needs DI in prod          |

Synctest is the cleanest option when available.

## Standard Library Hooks

- `testing/synctest.Run` — bubble entry (1.24+).
- `testing/synctest.Test` — bubble + `*testing.T` (1.25+).
- `testing/synctest.Wait` — wait for goroutine quiescence (1.25+).
- `time.*` — synctest-aware inside the bubble.
- `context.WithTimeout`/`WithDeadline` — synctest-aware.
- Channel `select` with `time.After` arms — synctest-aware.

## Real-World Patterns

### 1. Cache expiry

```go
func TestCacheExpiry(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        c := NewCache(time.Minute)
        c.Set("k", "v")
        time.Sleep(2 * time.Minute)
        synctest.Wait()
        if _, ok := c.Get("k"); ok {
            t.Error("should be expired")
        }
    })
}
```

### 2. Retry with backoff

```go
func TestRetryBackoff(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        var attempts int
        start := time.Now()
        err := WithRetry(func() error {
            attempts++
            if attempts < 5 { return errors.New("fail") }
            return nil
        }, time.Second, 5*time.Second)
        if err != nil { t.Fatal(err) }
        if attempts != 5 {
            t.Errorf("attempts = %d, want 5", attempts)
        }
        // backoff total: 1+2+4+8 = 15s synthetic
        if d := time.Since(start); d != 15*time.Second {
            t.Errorf("elapsed = %v, want 15s", d)
        }
    })
}
```

### 3. Rate limiter

```go
func TestRateLimiter(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        rl := NewLimiter(10, time.Second)
        for i := 0; i < 10; i++ {
            if !rl.Allow() { t.Errorf("call %d denied", i) }
        }
        if rl.Allow() { t.Errorf("11th call should be denied") }
        time.Sleep(time.Second)
        if !rl.Allow() { t.Errorf("after refill, should allow") }
    })
}
```

### 4. Context timeout

```go
func TestContextTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        err := SlowOperation(ctx)
        if !errors.Is(err, context.DeadlineExceeded) {
            t.Errorf("err = %v", err)
        }
    })
}
```

### 5. Periodic worker

```go
func TestWorker(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        var count atomic.Int32
        ctx, cancel := context.WithCancel(context.Background())
        go func() {
            tick := time.NewTicker(time.Minute)
            defer tick.Stop()
            for {
                select {
                case <-ctx.Done(): return
                case <-tick.C: count.Add(1)
                }
            }
        }()

        time.Sleep(10 * time.Minute)
        synctest.Wait()
        cancel()
        synctest.Wait()    // wait for goroutine to exit

        if got := count.Load(); got != 10 {
            t.Errorf("count = %d, want 10", got)
        }
    })
}
```

### 6. Migration from clockwork

```go
// before (clockwork)
clock := clockwork.NewFakeClock()
svc := NewService(clock)
clock.Advance(time.Hour)
svc.Check()

// after (synctest)
synctest.Test(t, func(t *testing.T) {
    svc := NewService()           // production constructor
    time.Sleep(time.Hour)
    synctest.Wait()
    svc.Check()
})
```

### 7. CI guard

```yaml
# Go 1.24
- run: GOEXPERIMENT=synctest go test ./...

# Go 1.25+
- run: go test ./...
```

## Anti-Patterns & Gotchas

**Busy loops inside the bubble.** Clock never advances; bubble hangs. Eliminate any spin/poll/`runtime.Gosched`.

**Real network calls inside the bubble.** The kernel block isn't visible; bubble may deadlock. Mock/stub network calls.

**Forgetting `synctest.Wait` before assertions.** Goroutines may not have run yet; race between assertion and worker.

**Leaving goroutines unfinished at bubble exit.** Bubble panics. Use cancel + Wait to drain.

**Mixing real-clock and synthetic-clock assertions.** `time.Since(start)` inside the bubble is synthetic; outside, real. Don't compare.

**Calling `synctest.Wait` when nothing is running.** Returns immediately; not harmful but signals confusion.

**Assuming `t.Parallel()` works inside the bubble.** Parallelism isn't synctest-aware in the same way; avoid.

**Using `time.Sleep(0)` to "yield".** Real Go uses `runtime.Gosched()`; in synctest, the clock doesn't advance for 0 duration. Use `synctest.Wait` instead.

**Forgetting that `time.AfterFunc` schedules a goroutine.** That goroutine must finish or be blocked at bubble exit.

**Synctest with `-race` disabled.** You miss real races. Run with `-race`.

**Expecting synctest to fix wall-clock races (mutex contention, scheduler decisions).** It fixes time-based flakes; other races still need normal techniques.

**Importing synctest pre-1.24.** Compile error. Guard with build tags if you support older Go.

## Performance Notes

- Inside-bubble `time.Sleep(1h)`: microseconds (clock advance + scheduler tick).
- `synctest.Wait` cost: O(active goroutines).
- Bubble setup: ~µs.
- Real-time tests of timer-heavy code: replaced with synctest, ~1000× faster.

For test suites with timers, synctest is the single biggest test-speedup tool added to Go in years.

## How Big Companies Use It

- **The Go team** dogfooded synctest for stdlib's `time`, `net/http`, `context` tests: https://github.com/golang/go.
- **Tailscale** adopted synctest for `wireguard-go`'s timer-heavy logic: https://tailscale.com/blog.
- **Cloudflare** uses synctest for HTTP/2 idle-timeout tests: https://blog.cloudflare.com.
- **CockroachDB** evaluating for replacing custom clock injection: https://github.com/cockroachdb/cockroach.
- **HashiCorp** adopting in Consul/Vault for lease-expiry tests: https://github.com/hashicorp/consul.
- **Discord** uses synctest for rate-limiter tests.
- **Many small projects** migrating from `clockwork` and `bouk/monkey`.

## Source Code References

Pinned to `go1.26`.

- `testing/synctest`: [`src/testing/synctest`](https://github.com/golang/go/tree/release-branch.go1.26/src/testing/synctest).
- Runtime integration: [`src/runtime/synctest.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/synctest.go).
- Time integration: [`src/time/sleep.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/time/sleep.go) (search `synctest`).
- Context integration: [`src/context/context.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/context/context.go).
- Channel select handling: [`src/runtime/chan.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/chan.go) (search `synctest`).
- Proposal: https://go.googlesource.com/proposal/+/master/design/67434-synctest.md.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Testing concurrent code with synctest" (Damien Neil, Go blog): https://go.dev/blog/synctest.
- "Proposal: testing/synctest" (Damien Neil): https://github.com/golang/go/issues/67434.
- "Go 1.25 release notes — synctest": https://tip.golang.org/doc/go1.25.
- "Faster, deterministic time-based tests in Go" (various blog posts): search "go synctest tutorial".
- "Clock injection alternatives" (clockwork): https://github.com/jonboulle/clockwork.

## Exercises / Self-Check

1. Convert a `time.Sleep`-based test to synctest. Measure the wall-time speedup.
2. Test a rate limiter without clock injection. Use `synctest.Test` + real `time.Now`.
3. Find a clockwork-using test in your codebase. Migrate to synctest.
4. Test a context timeout. Without synctest, the test takes the timeout duration; with synctest, microseconds.
5. Cause a bubble panic by leaving a goroutine running. Diagnose and fix.
