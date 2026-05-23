# `select`

## TL;DR

`select` is a multiplexer over channel operations. It blocks until at least one case can proceed, then runs exactly one — chosen **pseudo-randomly** among the ready cases. With a `default` clause it becomes non-blocking. The single biggest gotcha: when no cases are ready and there is no `default`, `select` blocks forever; with a `nil` channel a case becomes silently disabled (a feature, not a bug — exploit it).

## Mental Model

```
select {
case v := <-a:    // ready if a has a value or is closed
case b <- x:      // ready if b has buffer room or a parked receiver
case <-ctx.Done():// ready if ctx is canceled
default:          // ready always (makes the select non-blocking)
}
```

`selectgo` (runtime) evaluates each case to find which are ready right now. If at least one is ready, it picks one at random and executes it. If none are ready, it parks the goroutine on **all** the channels involved, and the first one to become ready wakes it. Then it re-checks and runs that case.

Key consequences:
- Pseudo-random fairness → no priority among cases. If you need priority, use nested selects.
- Parking on N channels has setup/teardown cost ~O(N). Selects with hundreds of cases are slow; use `reflect.Select` only if you must.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tick := time.Tick(50 * time.Millisecond)
	boom := time.After(150 * time.Millisecond)
	for {
		select {
		case <-tick:
			fmt.Println("tick")
		case <-boom:
			fmt.Println("BOOM!")
			return
		}
	}
	// Output:
	// tick
	// tick
	// BOOM!
}
```

(Output is timing-dependent; this example is a classic Tour of Go demo. In production prefer `time.NewTicker` + `defer ticker.Stop()` to avoid the `time.Tick` leak.)

## Deep Dive

### Pseudo-random selection

The Go spec says: "If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection." The PRNG state is per-`selectgo` invocation; do not rely on patterns. The fairness is statistical, not absolute.

Implementation: `selectgo` shuffles a `[ncases]uint16` poll order with Fisher-Yates, then walks it.

### `default` makes it non-blocking

```go
select {
case v := <-ch:
	use(v)
default:
	// no value available right now
}
```

Without `default`, the goroutine parks. With `default`, the goroutine never parks; if no other case is ready, `default` runs immediately. Use `default` for try-recv / try-send semantics.

### `nil` channel disables a case

```go
var done <-chan struct{}        // nil
if shouldWatch {
	done = ctx.Done()           // now non-nil
}
select {
case <-done:                    // disabled when done == nil
case msg := <-queue:
}
```

A send to or receive from a nil channel blocks forever, so `select` will never pick that case. Setting a channel variable to `nil` is a common way to remove a stage from a pipeline once it's exhausted:

```go
for in != nil || pending {
	select {
	case v, ok := <-in:
		if !ok { in = nil; continue }
		pending = append(pending, v)
	case out <- pending[0]:
		pending = pending[1:]
	}
}
```

This pattern is widely used in pipelines and merge-sort-style fan-ins.

### Empty `select{}` — block forever

`select {}` with no cases blocks the calling goroutine forever. Useful in `main` when the program is driven by background goroutines (e.g., an HTTP server is already calling `ListenAndServe` in a separate goroutine and you want `main` to not return).

### Timeouts with `time.After`

```go
select {
case res := <-resCh:
	use(res)
case <-time.After(2 * time.Second):
	return errTimeout
}
```

`time.After` creates a single-shot timer + goroutine. **It does not leak by definition, but it is not freed early**: if `resCh` wins immediately, the timer keeps running until 2 s elapses, holding the timer slot. In a tight loop this is wasteful; in user-request paths it's fine. For hot paths use `NewTimer`:

```go
t := time.NewTimer(2 * time.Second)
defer t.Stop()
select {
case res := <-resCh:
	use(res)
case <-t.C:
	return errTimeout
}
```

(Go 1.23 reworked timers: `t.Stop()` does not need the old `<-t.C` drain trick anymore — see the Go 1.23 release notes.)

### Send case

```go
select {
case ch <- v:
	// sent
case <-ctx.Done():
	return ctx.Err()
}
```

A send case fires only if the channel has room (or an immediate matched receiver). Combining a send case with a `ctx.Done()` case is the canonical way to make every send cancellable.

### Receive vs receive-with-ok

```go
select {
case v, ok := <-ch:
	if !ok { /* channel closed */ }
	use(v)
}
```

The two-value form is the only way to distinguish "closed" from "zero value sent". If you'd like the case to *only* fire on a real value, then exit on close, set `ch = nil` once `!ok` to disable that case.

### Repeat readiness, fairness, and priority

`select` has **no priority**. If you need priority — e.g., "always drain `quit` before `work`" — nest selects:

```go
for {
	select {
	case <-quit:
		return
	default:
	}
	select {
	case <-quit:
		return
	case w := <-work:
		handle(w)
	}
}
```

The first inner select with `default` is a non-blocking try-receive on `quit`; if it fires, exit. Otherwise fall through to the normal blocking select.

### `reflect.Select`

For dynamic, runtime-determined sets of cases:

```go
import "reflect"

cases := []reflect.SelectCase{
	{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(a)},
	{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(b)},
}
chosen, recv, ok := reflect.Select(cases)
```

Slow — ~10–100× the cost of static `select` because of reflection. Only use when the channel set isn't known at compile time (e.g., a watchdog that subscribes to N user-registered channels).

## Standard Library Hooks

- `time.After`, `time.NewTimer`, `time.NewTicker` — timed cases.
- `context.Context.Done()` — cancellation case.
- `signal.Notify` — channel-delivered OS signals, perfect for a top-level select.
- `reflect.Select` — dynamic case set.
- `runtime/trace` — selects show up as scheduler events in `go tool trace`.

## Real-World Patterns

### 1. The canonical cancellable worker loop

```go
package main

import (
	"context"
	"log/slog"
)

func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			slog.Info("worker exiting", "err", context.Cause(ctx))
			return
		case j, ok := <-jobs:
			if !ok {
				return
			}
			handle(ctx, j)
		}
	}
}

type Job struct{}
func handle(context.Context, Job) {}
```

Two cases, both essential. Every long-lived goroutine has this shape.

### 2. Non-blocking try-send / try-recv

```go
select {
case events <- e:
default:
	// queue full; drop event, increment metric
	metrics.Dropped.Inc()
}
```

Backpressure made explicit. The `default` makes "fail-fast" the deliberate choice. Without it, the send would block and silently degrade latency.

### 3. Disable a case dynamically

```go
package main

import "context"

func merge[T any](ctx context.Context, a, b <-chan T) <-chan T {
	out := make(chan T)
	go func() {
		defer close(out)
		for a != nil || b != nil {
			select {
			case <-ctx.Done():
				return
			case v, ok := <-a:
				if !ok { a = nil; continue }
				out <- v
			case v, ok := <-b:
				if !ok { b = nil; continue }
				out <- v
			}
		}
	}()
	return out
}
```

When one source closes, set its variable to `nil` and the case never fires again. The loop exits when both are `nil`.

### 4. Heartbeat with timeout

```go
func runWithHeartbeat(ctx context.Context, do func() (Result, error)) (Result, error) {
	done := make(chan struct{})
	var res Result
	var err error
	go func() {
		defer close(done)
		res, err = do()
	}()
	t := time.NewTicker(5 * time.Second)
	defer t.Stop()
	for {
		select {
		case <-done:
			return res, err
		case <-t.C:
			slog.Info("still working...")
		case <-ctx.Done():
			return Result{}, ctx.Err()
		}
	}
}

type Result struct{}
```

`select` lets you mix the "tell me when you're done" channel with periodic heartbeats and a timeout, with no extra primitives.

### 5. Quit-channel with priority

```go
for {
	// drain quit first
	select {
	case <-quit:
		return
	default:
	}
	select {
	case <-quit:
		return
	case msg := <-in:
		out <- transform(msg)
	}
}
```

Without the priority pass, `select` would choose pseudo-uniformly between `quit` and `in`, so a busy `in` could let you process forever after `quit` is closed.

## Anti-Patterns & Gotchas

**Empty select in a request path.** `select {}` blocks forever. Useful in `main`; **never** in a handler — it pins a goroutine and OS resources permanently.

**`time.After` in a loop.** Each iteration allocates a new timer; old timers keep running. Use `time.NewTimer` + `t.Reset(d)` (pre-1.23, with the `Stop`/drain dance; 1.23+ no longer needs the drain).

**Forgotten `default` causes accidental blocking.** "Why is my goroutine stuck?" — because you `select`ed on a case that was never going to be ready, with no `default` and no cancellation case.

**Closed-channel receive flooding the select.** A closed channel is *always* ready (returns zero value). If you do not set its variable to `nil` after observing close, the `select` will busy-loop choosing that case forever.

**Priority by ordering.** Case order does not imply priority. Use nested selects.

**Send case panicking on closed channel.** `select { case ch <- v: ... }` will panic if `ch` is closed and the send case is chosen. The send-on-closed rule applies regardless of `select`.

**Mutating shared state inside cases without a lock.** A `select` does not serialize anything beyond the chosen op. State touched across cases needs its own synchronization.

**Believing the spec gives you fairness over time.** It gives you *per-call* uniform selection. A consistently-saturated `a` and rarely-ready `b` will still cause `a` to be picked far more often.

## Performance Notes

- A static `select` with N cases costs roughly O(N) per call for the readiness scan + shuffle. For N ≤ 8 it's a few tens of ns.
- The dynamic `reflect.Select` is ~10–100× slower because of reflection allocations.
- Parking a goroutine on M channels touches M `hchan` mutexes briefly. The cost is mostly in the wakeup path.
- `select { case <-time.After(d): ... }` allocates a runtime timer (~80 bytes) every call. In hot paths this dominates.
- A `select` over a single channel is the same as the bare receive — the compiler can sometimes optimize it, but writing it as `v := <-ch` is clearer.

## How Big Companies Use It

- **etcd's raft Node interface** is essentially one giant `for { select { ... } }` driving the state machine: [`raft/node.go`](https://github.com/etcd-io/raft/blob/main/node.go).
- **Kubernetes `client-go` Reflector** uses `select` to multiplex watch events, resync ticks, and stop signals: [`tools/cache/reflector.go`](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go).
- **NATS server** mixes channel-based per-connection select loops with lock-free hot paths.
- **Bryan C. Mills, "Rethinking Classical Concurrency Patterns"** has multiple `select`-driven cancellation examples that have shaped Go-team-recommended style.
- **CockroachDB** uses `select` heavily in `pkg/sql/distsql` and the raft loop.

## Source Code References

Pinned to `go1.26`.

- `selectgo`: [`src/runtime/select.go`](https://github.com/golang/go/blob/master/src/runtime/select.go) — read the file top to bottom; it's well-commented and not very long.
- Compiler lowering of `select` to `selectgo`: [`src/cmd/compile/internal/walk/select.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/select.go).
- `reflect.Select` driver: [`src/reflect/value.go`](https://github.com/golang/go/blob/master/src/reflect/value.go), search `func Select`.
- Spec: [`go/ref/spec#Select_statements`](https://go.dev/ref/spec#Select_statements).

## Further Reading

- Go spec, "Select statements": https://go.dev/ref/spec#Select_statements
- Russ Cox, "Go's select statement": https://research.swtch.com/godoc (and follow-ups on lock-free selects)
- Sameer Ajmani, "Pipelines and cancellation": https://go.dev/blog/pipelines
- Dmitry Vyukov, "Scalable Go scheduler design doc" — see the section on select wait queues
- Damian Gryski, "go-perfbook" — select micro-benchmarks: https://github.com/dgryski/go-perfbook
- Filippo Valsorda, "Timing the select" (informal): https://words.filippo.io/

## Exercises / Self-Check

1. Why does `select {}` block forever? In what runtime state is the goroutine left?
2. Write `TryRecv[T any](ch <-chan T) (T, bool)` using a one-case select with `default`.
3. Show how to drain multiple input channels until all are closed using nil-channel disabling.
4. A select has cases on `a` (closed) and `b` (open, no data). What runs? What if you `<-a` ten more times?
5. Explain why a priority "quit before work" requires *two* selects, not one with case order.
