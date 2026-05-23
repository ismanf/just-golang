# Goroutine Leaks

## TL;DR

A goroutine leak is a goroutine that the runtime can no longer ever schedule to completion — it's parked on a channel that will never receive, a lock that will never unlock, a context that will never cancel, or just an infinite loop with no exit. Leaked goroutines don't get GC'd (the runtime considers them live), so they hold their stacks (≥2 KiB), their captured variables, and anything reachable from those. The single biggest gotcha: **send/receive on an unbuffered channel without a `select { case <-ctx.Done(): }` arm**. The canonical detection tool is Uber's `uber-go/goleak`. In production: `runtime.NumGoroutine()`, `pprof goroutine` profile, and `SIGQUIT` dump.

## Mental Model

```
A goroutine is leaked if:
    no other goroutine can ever cause it to wake AND it has no way to exit on its own.

Common shapes:

    send to unbuffered chan, no receiver       <- producer parked forever
    receive from chan that's never written      <- consumer parked forever
    waiting on a mutex held by a dead goroutine <- pure deadlock
    select with no ctx.Done() arm + slow consumer
    `time.Tick(d)` in a function that returns   <- ticker leaks until process exit
    `for range ch` where the producer abandons close

Each leaked G holds:
    its stack (~2-8 KiB)
    its goid record
    everything its closure captures (slices, maps, conns, etc.)
```

The runtime cannot detect a leak — it just sees a parked G. Detection is your job, via tools.

## Syntax & Basic Usage

```go
// Bad: classic generator leak
func Numbers() <-chan int {
	out := make(chan int)
	go func() {
		for i := 0; ; i++ {
			out <- i // blocks forever if consumer abandons
		}
	}()
	return out
}

// Fix: thread context, exit on cancel
func Numbers(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 0; ; i++ {
			select {
			case <-ctx.Done():
				return
			case out <- i:
			}
		}
	}()
	return out
}
```

Both compile. Only the second cleans up when the caller stops reading.

## Deep Dive

### Why leaks aren't reclaimed

The Go runtime keeps every G alive until it returns (or `runtime.Goexit`). A parked G is "waiting", not "dead". The GC scans its stack and treats everything reachable as roots. Until the G is woken and exits, its memory stays.

If a server leaks one G per request, at 1000 RPS for a day you have ~86M parked Gs — somewhere around 200 GiB of stack memory if average stack is 2 KiB, plus everything those Gs captured. In practice the process OOMs long before that.

### Top causes

1. **Unbuffered send with no receiver.** The producer goroutine is parked indefinitely.
2. **Receive from a channel no one closes.** Consumer parked.
3. **`select` with no `ctx.Done()` arm.** Same as the above when conditions change.
4. **`time.Tick` or `time.After` in a function that exits early.** The underlying timer goroutine doesn't know the caller is gone.
5. **Calling `wg.Wait()` when one `wg.Add` was never matched by a `Done`.** Hangs forever.
6. **HTTP request without `defer resp.Body.Close()`.** Holds the connection and the read goroutine.
7. **Database rows without `defer rows.Close()`.** Locks a row in the pool.
8. **`for { select { ... } }` with no exit branch.** Eternal worker.
9. **Goroutines spawned from a request handler that outlive the request.** Common in fire-and-forget logging or audit dispatch.
10. **Deadlock between mutexes acquired in different orders.** All involved goroutines parked.

### Detection in tests: `uber-go/goleak`

```go
package mypkg

import (
	"testing"
	"go.uber.org/goleak"
)

func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m)
}
```

`goleak.VerifyTestMain` runs the test suite, then after the last test counts goroutines. Any G left over (excluding a small allowlist of framework Gs) is reported as a leak with its stack. This catches "test leaked, test passed" — the highest-signal leaks because they're reproducible.

For a single test:

```go
func TestThing(t *testing.T) {
	defer goleak.VerifyNone(t)
	// ... test ...
}
```

### Detection in production: `runtime.NumGoroutine`

```go
import "runtime"
import "log/slog"

go func() {
	for range time.Tick(30 * time.Second) {
		slog.Info("goroutine count", "n", runtime.NumGoroutine())
	}
}()
```

A monotonically growing count over hours is a leak signature. Plot it in Prometheus / Grafana — every Go service should have this metric.

### Detection in production: `pprof goroutine` profile

```
go tool pprof http://localhost:6060/debug/pprof/goroutine
(pprof) top
(pprof) list <function-name>
```

The goroutine profile is a histogram of goroutines grouped by stack. The largest bucket points at the leak: "you have 100,000 goroutines all parked at line X of function Y."

With `?debug=2`:

```
http://localhost:6060/debug/pprof/goroutine?debug=2
```

you get full stacks. With `?debug=1` you get aggregated counts.

### Detection: `SIGQUIT` dump (no pprof needed)

```
kill -QUIT <pid>
```

The process prints every goroutine's stack to stderr and exits. Useful in a pinch when you don't have pprof endpoints exposed. For non-exit dumps install `SIGUSR1` (see `11-scheduler-internals.md`).

### `GODEBUG=schedtrace` and `gctrace`

```
GODEBUG=schedtrace=1000 ./myserver
```

Prints scheduler stats every 1000 ms, including goroutine count, run queue lengths, etc. Useful early-warning.

### Categorizing what you see

When you find a large goroutine pile, the stacks tell you:
- `chan receive` → consumer waiting; producer is gone or never wrote.
- `chan send` → producer waiting; consumer drained too slowly or is gone.
- `select` → multiple waits; check what arms exist.
- `IO wait` → real I/O. Usually not a leak unless the socket is dead and `SetReadDeadline` was never set.
- `semacquire` → mutex contention.
- `sync.WaitGroup.Wait` → missing `Done` somewhere.

### The `time.Tick` leak

```go
func process(d time.Duration) {
	for range time.Tick(d) {
		work()
	}
} // returns? the ticker goroutine survives
```

`time.Tick` creates a ticker goroutine and never stops it; if `process` returns the goroutine is unreferenced — but it doesn't get GC'd because it's still sending into the channel it owns and the runtime keeps it alive. **Always use `time.NewTicker` + `defer t.Stop()`.**

### Cancellation discipline

Every goroutine that lives longer than its caller's stack frame must accept a `context.Context` or a closeable `done` channel. Period. Code review: if you see `go f()` without one of those, ask "how does it stop?"

### Subroutine cancellation propagation

When a parent goroutine cancels, the cancellation should reach every descendant. `context.WithCancel`'s children list does this for context, but you also need:
- Every blocking call to take a `ctx` parameter.
- Every `select` to have a `<-ctx.Done()` arm.
- Long CPU loops to check `ctx.Err()` periodically.

A subroutine that takes a `ctx` but never checks it is a leak waiting to happen.

### Anti-pattern: "I'll just defer cancel later"

```go
ctx := context.Background()
go work(ctx) // no way to cancel work
```

If `ctx` is `Background`, `ctx.Done()` is a nil channel — `<-ctx.Done()` blocks forever. The worker can never exit on cancellation. Use `WithCancel`.

### Anti-pattern: receive-from-closed in a loop

```go
for {
	v, ok := <-ch
	if !ok { ... }
	use(v)
}
```

If you never break/return after `!ok`, you spin reading zero values from the closed channel forever — 100% CPU. Always break/return on close.

## Standard Library Hooks

- `runtime.NumGoroutine() int` — leak signal.
- `runtime.Stack(buf, all)` — programmatic dump.
- `runtime/pprof.Lookup("goroutine").WriteTo(w, debug)` — pprof's goroutine profile.
- `net/http/pprof` — `/debug/pprof/goroutine` endpoint. Just `import _ "net/http/pprof"`.
- `context` — every long-lived G's exit signal.
- `time.Ticker`, `time.Timer` — always `Stop` them.
- `GODEBUG=schedtrace=1000` — runtime stats env var.
- `go.uber.org/goleak` — test-time detector.
- `testing/synctest` — deterministic-time tests where leaks surface as deadlock panics inside the bubble.

## Real-World Patterns

### 1. Leaking sender

```go
// LEAK
func emit(values []int) <-chan int {
	out := make(chan int)
	go func() {
		for _, v := range values {
			out <- v // blocks forever if consumer reads <len(values) items
		}
		close(out)
	}()
	return out
}

// FIX
func emit(ctx context.Context, values []int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, v := range values {
			select {
			case <-ctx.Done(): return
			case out <- v:
			}
		}
	}()
	return out
}
```

### 2. Leaking receiver

```go
// LEAK
go func() {
	for v := range ch { // if ch is never closed, this goroutine never exits
		handle(v)
	}
}()

// FIX
go func() {
	for {
		select {
		case <-ctx.Done(): return
		case v, ok := <-ch:
			if !ok { return }
			handle(v)
		}
	}
}()
```

### 3. Detecting an HTTP body leak

```go
resp, err := http.Get(url) // forgot Body.Close
if err != nil { return }
// ... use resp.Body partially ...
// resp.Body's underlying connection's read goroutine leaks until the conn is closed
```

`go vet` (with `bodyclose` analyzer enabled via `staticcheck` or `bodyclose` standalone) catches this. **Always** `defer resp.Body.Close()` and **always** drain (`io.Copy(io.Discard, resp.Body)`) before close if you want connection reuse.

### 4. WaitGroup leak

```go
// LEAK
var wg sync.WaitGroup
for _, item := range items {
	wg.Add(1)
	go func() {
		// forgot defer wg.Done()
		if shouldSkip(item) { return } // wg.Wait() will hang
		work(item)
		wg.Done()
	}()
}
wg.Wait()

// FIX
for _, item := range items {
	wg.Go(func() { // Go 1.25+: defer wg.Done() built in
		if shouldSkip(item) { return }
		work(item)
	})
}
```

### 5. Goroutine-pinning the connection

```go
// LEAK: the goroutine reading from net.Conn never exits if the peer hangs
go func() {
	for {
		var b [1024]byte
		_, _ = conn.Read(b[:]) // blocks forever; no deadline
	}
}()

// FIX
conn.SetReadDeadline(time.Now().Add(30 * time.Second))
// or accept ctx and call conn.SetDeadline when ctx fires:
go func() {
	<-ctx.Done()
	conn.Close() // unblocks the Read with an error
}()
```

`net.Conn.Close()` unblocks pending Read/Write with an error — that's the canonical way to cancel real I/O.

## Anti-Patterns & Gotchas

**`go f()` with no exit story.** Every goroutine needs an exit. Document it in the comment if it's not obvious.

**Long-lived goroutine that doesn't take a context.** Refactor.

**`time.After` / `time.Tick` in a function that returns.** Use `NewTimer` / `NewTicker` with `defer Stop()`.

**Forgetting `defer resp.Body.Close()`.** Leaks the connection's read goroutine and a file descriptor.

**Worker pool's job channel never closed.** Workers `range` forever.

**Goroutine that captures a huge variable.** Even if the goroutine eventually exits, while it lives it pins the captured memory. Watch closures over big slices/maps.

**Goroutine that holds a mutex in a deadlock.** All other contenders also leak.

**Spawning a goroutine inside a hot path without limit.** Even bounded leaks (eventually exit) can spike memory; rate-limit or pool.

**Trusting `runtime.NumGoroutine` to spot small leaks.** A leak of 1 goroutine per hour is invisible at one-minute resolution. Watch the trend over days.

**`goleak.VerifyTestMain` skipping background goroutines.** It allowlists common stdlib Gs but you may need to add ignores for your framework — keep the list minimal so real leaks don't slip through.

## Performance Notes

- Each leaked goroutine: 2 KiB stack minimum, ~256 B G struct, plus captured-variable pinning.
- Each leak adds GC scan work — even parked goroutines have their stacks scanned (since they may be runnable).
- `pprof goroutine` profile at debug=1 is cheap (just counters); debug=2 dumps stacks and can be hundreds of MiB if you have millions of Gs.
- `runtime.Stack(buf, true)` allocates `buf` on every call and walks every G — don't call it in a hot path.

## How Big Companies Use It

- **Uber's `goleak`** (https://github.com/uber-go/goleak) is the de facto standard. Their team has talked about how it found 100+ leaks in their codebase over time.
- **Kubernetes** runs `goleak`-style checks on key controllers; the API server has goroutine-count metrics in its standard dashboard.
- **CockroachDB** has internal tooling for goroutine-count diffs across CI runs.
- **Cloudflare's edge** monitors `go_goroutines` (Prometheus default metric); alerts fire on sustained growth.
- **Discord's incident retros** mention goroutine leaks as a common production trap: https://discord.com/blog/.
- **Bryan C. Mills's "Rethinking Classical Concurrency Patterns"** is largely an extended argument about how to design APIs that *can't* leak: https://www.youtube.com/watch?v=5zXAHh5tJqQ.
- **Google's gRPC-Go** has a long history of leak fixes around stream lifecycle — search the repo for "leak" in commit messages.

## Source Code References

Pinned to `go1.26`.

- Goroutine state and counters: [`src/runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go), search `allgs`.
- `runtime.NumGoroutine`: same file.
- Goroutine profile generation: [`src/runtime/mprof.go`](https://github.com/golang/go/blob/master/src/runtime/mprof.go), search `goroutineProfile`.
- `pprof` HTTP endpoint: [`src/net/http/pprof/pprof.go`](https://github.com/golang/go/blob/master/src/net/http/pprof/pprof.go).
- `goleak`: [`github.com/uber-go/goleak`](https://github.com/uber-go/goleak).
- `bodyclose` analyzer: [`github.com/timakin/bodyclose`](https://github.com/timakin/bodyclose).

## Further Reading

- Bryan C. Mills, "Rethinking Classical Concurrency Patterns" (GopherCon 2018): https://www.youtube.com/watch?v=5zXAHh5tJqQ
- Sameer Ajmani, "Go Concurrency Patterns: Pipelines and cancellation": https://go.dev/blog/pipelines
- Uber, "Goroutine leaks: prevention, detection, and recovery" (blog): https://www.uber.com/blog/
- Dave Cheney, "Never start a goroutine without knowing how it will stop" (Practical Go): https://dave.cheney.net/practical-go
- Filippo Valsorda, "Pprof tutorial": https://filippo.io/
- "Visualizing goroutines with pprof" (Go blog and various): https://go.dev/blog/pprof
- `gops` for live introspection: https://github.com/google/gops

## Exercises / Self-Check

1. Identify the leak in: `func Tick() { for range time.Tick(time.Second) { log.Println("tick") } }` called once and then `return`ed.
2. Write a test that uses `goleak.VerifyTestMain` to fail when a deliberately-leaking generator runs.
3. Construct an HTTP handler that leaks a goroutine per request without `body.Close()`. Visualize with `pprof goroutine`.
4. Why does `net.Conn.Close` unblock a parked `Read`? What's the canonical pattern for canceling I/O via context?
5. Sketch the goroutine profile output for a worker pool that leaks one G per malformed job. What does the "fix" stack trace look like?
