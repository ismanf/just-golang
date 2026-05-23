# Concurrency Patterns

## TL;DR

There are roughly half a dozen concurrency patterns that cover 95% of production Go code: pipelines, fan-out/fan-in, worker pools, bounded parallelism via semaphore, generator (channel-producing) goroutines, and request/response with `errgroup`. The single biggest gotcha across all of them: **every goroutine you spawn must have an answer to "how does it stop?"** — usually a `context.Context` or a closed `done` channel. Without that, every pattern leaks.

## Mental Model

```
generator     -->  stage1  -->  stage2  -->  sink         (pipeline)

source --> [fan-out to N workers] --> [fan-in to merger] --> consumer

source --> [sem <- struct{}{}; go work(); <-sem]         (bounded parallelism)

ctx --> errgroup.Go(...) --> errgroup.Go(...) --> g.Wait() (request/response)

producer --> bounded channel --> consumer                (backpressure)
```

Every arrow above is a channel or `errgroup`. Every node is a goroutine. Every box has a "stop" input (context or channel close).

## Syntax & Basic Usage

The minimum building blocks are repeated below in every pattern. To save space, the patterns themselves assume:
- `context.Context` is passed in for cancellation.
- `errgroup` (`golang.org/x/sync/errgroup`) is available; install with `go get golang.org/x/sync`.

## Deep Dive

### Pattern 1: Generator

A goroutine that owns and writes to a channel, closing it when done.

```go
package main

import (
	"context"
	"fmt"
)

func numbers(ctx context.Context, n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 0; i < n; i++ {
			select {
			case <-ctx.Done():
				return
			case out <- i:
			}
		}
	}()
	return out
}

func main() {
	for v := range numbers(context.Background(), 5) {
		fmt.Println(v)
	}
	// Output:
	// 0
	// 1
	// 2
	// 3
	// 4
}
```

The producer **always** owns the close. The `select` is essential — without it, an abandoned consumer leaks the producer forever.

### Pattern 2: Pipeline

Chain generators by passing the upstream channel to the next stage.

```go
package main

import "context"

func square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case <-ctx.Done():
				return
			case out <- v * v:
			}
		}
	}()
	return out
}

// Usage:
// ctx, cancel := context.WithCancel(context.Background())
// defer cancel()
// for v := range square(ctx, square(ctx, numbers(ctx, 5))) { ... }
```

Cancellation propagates because closing upstream causes each downstream `range` to exit, which closes its own output. Every stage cleans up after itself.

### Pattern 3: Fan-out

Multiple workers consume from the same input channel.

```go
package main

import (
	"context"
	"sync"
)

func fanOut[T, U any](ctx context.Context, n int, in <-chan T, f func(T) U) <-chan U {
	out := make(chan U)
	var wg sync.WaitGroup
	for range n {
		wg.Go(func() {
			for v := range in {
				select {
				case <-ctx.Done():
					return
				case out <- f(v):
				}
			}
		})
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

`WaitGroup.Go` is Go 1.25+. Pre-1.25 use `wg.Add(1)` + `defer wg.Done()`. The trick: only the dedicated closer goroutine closes `out`, after every worker has finished — never close from within a worker.

### Pattern 4: Fan-in (merge)

Multiple producer channels, one consumer channel.

```go
package main

import "sync"

func merge[T any](cs ...<-chan T) <-chan T {
	out := make(chan T)
	var wg sync.WaitGroup
	for _, c := range cs {
		wg.Go(func() {
			for v := range c {
				out <- v
			}
		})
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

Same closing discipline: a dedicated closer waits for every input to drain, then closes the output.

### Pattern 5: Worker pool

A fixed pool of workers, a job queue, an optional results queue. The most common production shape.

```go
package main

import (
	"context"
	"sync"
)

type Job struct{ ID int }
type Result struct{ ID int; Err error }

func WorkerPool(ctx context.Context, n int, jobs <-chan Job, do func(context.Context, Job) error) <-chan Result {
	out := make(chan Result)
	var wg sync.WaitGroup
	for w := 0; w < n; w++ {
		wg.Go(func() {
			for j := range jobs {
				err := do(ctx, j)
				select {
				case <-ctx.Done():
					return
				case out <- Result{ID: j.ID, Err: err}:
				}
			}
		})
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

The pool size `n` caps concurrency. The job channel can be buffered for backpressure. The result channel must be drained by the caller or the workers block.

### Pattern 6: Bounded parallelism via semaphore

Use a buffered channel as a counting semaphore when the pool shape doesn't fit.

```go
package main

import "sync"

func ParallelMap[T, U any](in []T, max int, f func(T) U) []U {
	out := make([]U, len(in))
	sem := make(chan struct{}, max)
	var wg sync.WaitGroup
	for i, v := range in {
		wg.Add(1)
		sem <- struct{}{} // acquire
		go func(i int, v T) {
			defer wg.Done()
			defer func() { <-sem }() // release
			out[i] = f(v)
		}(i, v)
	}
	wg.Wait()
	return out
}
```

Each goroutine writes to its own slot, so no race. Up to `max` run at once. For variable concurrency, `golang.org/x/sync/semaphore.Weighted` lets you weight different jobs differently.

### Pattern 7: errgroup — first-error fan-out

```go
package main

import (
	"context"
	"golang.org/x/sync/errgroup"
	"net/http"
)

func fetchAll(ctx context.Context, urls []string) error {
	g, ctx := errgroup.WithContext(ctx)
	for _, u := range urls {
		g.Go(func() error {
			req, _ := http.NewRequestWithContext(ctx, "GET", u, nil)
			resp, err := http.DefaultClient.Do(req)
			if err != nil {
				return err
			}
			resp.Body.Close()
			return nil
		})
	}
	return g.Wait()
}
```

`errgroup.WithContext` returns a derived `ctx` that cancels when the first error is returned. Other in-flight workers see the cancellation and exit. `g.Wait()` returns the first non-nil error. See `13-errgroup-and-singleflight.md`.

### Pattern 8: Or-channel — return when any source signals

```go
func or(channels ...<-chan struct{}) <-chan struct{} {
	switch len(channels) {
	case 0: return nil
	case 1: return channels[0]
	}
	out := make(chan struct{})
	go func() {
		defer close(out)
		switch len(channels) {
		case 2:
			select {
			case <-channels[0]:
			case <-channels[1]:
			}
		default:
			select {
			case <-channels[0]:
			case <-channels[1]:
			case <-channels[2]:
			case <-or(append(channels[3:], out)...):
			}
		}
	}()
	return out
}
```

Returns a channel that closes as soon as any of `channels` closes. From Sameer Ajmani's pipelines blog. Useful for combining several `done` signals.

### Pattern 9: Tee — duplicate one channel to two

```go
func tee[T any](ctx context.Context, in <-chan T) (<-chan T, <-chan T) {
	out1 := make(chan T)
	out2 := make(chan T)
	go func() {
		defer close(out1); defer close(out2)
		for v := range in {
			a, b := out1, out2
			for i := 0; i < 2; i++ {
				select {
				case <-ctx.Done():
					return
				case a <- v: a = nil
				case b <- v: b = nil
				}
			}
		}
	}()
	return out1, out2
}
```

The `a = nil` / `b = nil` trick disables a case after that side has received. Both consumers must keep up; if one stalls, the producer stalls.

### Pattern 10: Rate limiter

```go
import "time"

ticker := time.NewTicker(time.Second / 100) // 100 req/s
defer ticker.Stop()
for req := range requests {
	<-ticker.C
	go handle(req)
}
```

For more sophisticated rate limiting use `golang.org/x/time/rate.Limiter` (token bucket):

```go
import "golang.org/x/time/rate"

l := rate.NewLimiter(100, 200) // 100 req/s, burst 200
for req := range requests {
	if err := l.Wait(ctx); err != nil { return err }
	go handle(req)
}
```

### Pattern 11: Backpressure via bounded channel

```go
queue := make(chan Job, 1000) // backlog of at most 1000
go func() {
	for j := range incoming {
		select {
		case queue <- j:
		default:
			// queue full; drop or shed load
			metrics.Dropped.Inc()
		}
	}
}()
```

A bounded queue is a backpressure mechanism, not an infinite buffer. The `default` makes "drop on overflow" the deliberate policy; without it, the producer blocks, which can be the right answer too.

### Pattern 12: Heartbeat / liveness

```go
func work(ctx context.Context, pulse <-chan time.Time) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-pulse:
			// send heartbeat to observer
		default:
			// do unit of work
		}
	}
}
```

For tests, the pulse channel makes "is the worker alive?" testable without sleeping. See `synctest` (in `14-testing-synctest.md`).

## Standard Library Hooks

- Channels, `select`, and `close` — the building blocks of every pattern here.
- `context.Context` — the cancellation signal woven into every long-lived goroutine.
- `sync.WaitGroup` (and `WaitGroup.Go` in 1.25+) — fan-out coordination.
- `sync.Once` / `sync.OnceFunc` / `OnceValue` — single-execution patterns.
- `time.NewTicker` / `time.NewTimer` — rate limiting and heartbeats; remember to `Stop()`.
- `golang.org/x/sync/errgroup` — fan-out with first-error capture and cancellation.
- `golang.org/x/sync/semaphore.Weighted` — weighted bounded parallelism.
- `golang.org/x/sync/singleflight` — request coalescing.
- `golang.org/x/time/rate` — token-bucket rate limiter.
- `runtime/pprof.Do` + `pprof.Labels` — tag goroutines for profiling visibility.
- `testing/synctest` (1.25+) — deterministic tests for these patterns.

## Real-World Patterns

### 1. Concurrent batch insert

```go
func insertAll(ctx context.Context, db *sql.DB, items []Item) error {
	g, ctx := errgroup.WithContext(ctx)
	sem := make(chan struct{}, 10) // 10 concurrent inserts
	for _, it := range items {
		g.Go(func() error {
			select {
			case sem <- struct{}{}:
				defer func() { <-sem }()
			case <-ctx.Done():
				return ctx.Err()
			}
			_, err := db.ExecContext(ctx, "INSERT INTO t VALUES(?)", it.Value)
			return err
		})
	}
	return g.Wait()
}
```

Bounded concurrency + first-error cancellation + context-aware acquisition.

### 2. Crawler with depth limit

```go
type Page struct{ URL string; Links []string }

func Crawl(ctx context.Context, root string, depth int) (<-chan Page, error) {
	out := make(chan Page, 100)
	g, ctx := errgroup.WithContext(ctx)
	sem := make(chan struct{}, 32)

	var visit func(url string, d int)
	visit = func(url string, d int) {
		if d > depth { return }
		g.Go(func() error {
			select {
			case sem <- struct{}{}:
				defer func() { <-sem }()
			case <-ctx.Done():
				return ctx.Err()
			}
			p, err := fetch(ctx, url)
			if err != nil { return err }
			select {
			case out <- p:
			case <-ctx.Done():
				return ctx.Err()
			}
			for _, l := range p.Links {
				visit(l, d+1)
			}
			return nil
		})
	}
	visit(root, 0)
	go func() { g.Wait(); close(out) }()
	return out, nil
}

func fetch(context.Context, string) (Page, error) { return Page{}, nil }
```

### 3. Job dispatcher with priority queues

```go
type Priority int
const (HighPrio Priority = iota; LowPrio)

func dispatch(ctx context.Context, high, low <-chan Job, do func(Job)) {
	for {
		// drain high-priority first
		select {
		case j := <-high:
			do(j)
			continue
		default:
		}
		select {
		case <-ctx.Done(): return
		case j := <-high: do(j)
		case j := <-low:  do(j)
		}
	}
}
```

The non-blocking `default` select makes high-priority work strictly preferred. Without it, the standard `select` is uniform-random and high jobs can be starved.

### 4. Cancel-on-first-success ("hedged request")

```go
func hedge(ctx context.Context, urls []string) (Result, error) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	type ret struct{ R Result; E error }
	results := make(chan ret, len(urls))
	for _, u := range urls {
		go func() {
			r, err := fetch(ctx, u)
			results <- ret{r, err}
		}()
	}
	for range urls {
		r := <-results
		if r.E == nil {
			cancel() // cancel the rest
			return r.R, nil
		}
	}
	return Result{}, errors.New("all failed")
}
```

Fire the same request at multiple replicas; take the first success; cancel the rest. Reduces tail latency at the cost of extra load.

### 5. Inactivity timeout

```go
func relay(ctx context.Context, in <-chan Msg, out chan<- Msg, idle time.Duration) error {
	t := time.NewTimer(idle)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-t.C:
			return errors.New("idle timeout")
		case m, ok := <-in:
			if !ok { return nil }
			if !t.Stop() {
				select { case <-t.C: default: } // pre-1.23 drain dance
			}
			t.Reset(idle)
			select {
			case out <- m:
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	}
}
```

Times out only if no messages flow for `idle`. Go 1.23 simplifies the `Stop`/`Reset` dance (no drain required).

## Anti-Patterns & Gotchas

**"Just spawn a goroutine."** Without a stop signal, you've leaked. Every spawn pairs with a context, a wait, or an explicit done channel.

**Closing a channel from a worker.** The owning goroutine closes — the one that knows when there's no more to send. Workers that close cause panics when other workers try to send.

**Worker pool that never closes the result channel.** Consumers `range` and block forever.

**Pipeline with no cancellation.** Closing upstream usually drains the system gracefully, but an idle stage on a slow consumer + a fast producer can stall everything. Use context.

**Unbounded fan-out.** `for _, x := range bigSlice { go process(x) }` spawns N goroutines. With N = millions, you'll OOM. Use a worker pool.

**Errgroup without canceling on first error.** Use `errgroup.WithContext`. The vanilla `errgroup.Group` waits for everyone to finish before returning; failures don't propagate cancellation.

**Channel-based mutexes.** Using `chan struct{}` of capacity 1 as a lock works but is slower than `sync.Mutex` and obscures the intent.

**Treating concurrency as the optimization.** If your work is I/O-bound, more goroutines help up to the OS-thread / FD / DB-conn limit. If it's CPU-bound, more goroutines than cores doesn't help. Profile first.

**Worker that never times out on send to a slow consumer.** Wrap every channel op in a `select` with `ctx.Done()` or a deadline.

## Performance Notes

- Worker pool size: start with `runtime.NumCPU()` for CPU-bound work, ~2× for mixed I/O.
- Channel buffer size: pick a power of two; 1024 is a common default. Bigger ≠ better — bigger buffers hide backpressure.
- Semaphore approach beats worker pool when jobs are heterogeneous (some short, some long): the semaphore lets short ones drain without waiting for long ones to release a worker slot.
- `errgroup`'s overhead is one mutex + one channel close per error — negligible.
- Tee/teeing N consumers from 1 producer: each consumer slowdown caps the producer. If you can't keep up, drop or sample.
- Trace your pipeline (`go test -bench -trace=trace.out; go tool trace trace.out`) — visualizing G activity often shows the bottleneck.

## How Big Companies Use It

- **Kubernetes `client-go` workqueues** are textbook worker-pool implementations: [`util/workqueue`](https://github.com/kubernetes/client-go/tree/master/util/workqueue).
- **Cloudflare's CDN** uses pipeline stages for request transformation; many of them are described in https://blog.cloudflare.com/scaling-go-applications/.
- **Prometheus scrape pool** is a worker pool with per-target goroutines: [`prometheus/scrape`](https://github.com/prometheus/prometheus/tree/main/scrape).
- **CockroachDB**'s SQL execution layer uses pipeline + fan-out for parallel scans.
- **gRPC-Go** uses fan-in for its server-side streaming response merger.
- **Tailscale's `wgengine`** uses bounded parallelism per packet flow.
- **Sameer Ajmani's "Pipelines and cancellation" Go blog** (2014) is canonical: https://go.dev/blog/pipelines.

## Source Code References

Pinned to `go1.26`.

- Channel ops underlying these patterns: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go).
- WaitGroup.Go (1.25+): [`src/sync/waitgroup.go`](https://github.com/golang/go/blob/master/src/sync/waitgroup.go).
- `errgroup`: [`golang.org/x/sync/errgroup`](https://pkg.go.dev/golang.org/x/sync/errgroup).
- `semaphore.Weighted`: [`golang.org/x/sync/semaphore`](https://pkg.go.dev/golang.org/x/sync/semaphore).
- `singleflight.Group`: [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight).
- Kubernetes workqueue (real-world worker pool): https://github.com/kubernetes/client-go/blob/master/util/workqueue/queue.go.

## Further Reading

- Sameer Ajmani, "Go Concurrency Patterns: Pipelines and cancellation": https://go.dev/blog/pipelines
- Rob Pike, "Go Concurrency Patterns" (Google I/O 2012): https://www.youtube.com/watch?v=f6kdp27TYZs
- Rob Pike, "Advanced Go Concurrency Patterns": https://www.youtube.com/watch?v=QDDwwePbDtw
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns" (GopherCon 2018): https://www.youtube.com/watch?v=5zXAHh5tJqQ
- Kavya Joshi, "Understanding Channels" (GopherCon 2017): https://www.youtube.com/watch?v=KBZlN0izeiY
- Katherine Cox-Buday, *Concurrency in Go* (O'Reilly book)
- Damian Gryski, "go-perfbook" — concurrency micro-benchmarks: https://github.com/dgryski/go-perfbook

## Exercises / Self-Check

1. Write a four-stage pipeline (gen → filter → map → reduce) that cancels cleanly on context.
2. Convert a "spawn per request" handler into a bounded worker pool with `errgroup`.
3. Implement `hedge` (cancel-on-first-success) with deadline semantics — return after `d` even if no replica succeeded.
4. Why does the dedicated closer goroutine pattern (`go func(){ wg.Wait(); close(out) }()`) exist? What goes wrong if any worker closes?
5. Build a tee that doesn't block the producer when one consumer is slow (hint: per-consumer buffered queues + drop policy).
