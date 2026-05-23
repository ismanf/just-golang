# Channels vs Mutexes

## TL;DR

"Don't communicate by sharing memory; share memory by communicating" is the most-quoted Go slogan and the most-misapplied. The truth: **channels are about transferring ownership**; **mutexes are about guarding state**. For a counter, a cache, a registry, or any "shared piece of memory that many goroutines read and update," a `sync.Mutex` is simpler, smaller, and usually faster. For a pipeline, a worker handoff, or a cancellation signal, a channel is clearer. The single biggest gotcha: people reach for channels because they're "more Go-ish," then end up implementing a hand-rolled mutex out of `chan struct{}` of capacity 1 — slower and harder to read than `sync.Mutex`.

## Mental Model

```
state to protect?                      data to move?
       |                                       |
       v                                       v
  +----------+                          +-------------+
  |  Mutex   |                          |  Channel    |
  +----------+                          +-------------+
  - one location                        - producer/consumer
  - many readers/writers                - ownership transfer
  - read-modify-write                   - signaling
  - 10-50 ns per op                     - 50-300 ns per op
  - sharded if hot                      - select for multiplexing
```

A Mutex is shorter (in code), faster (per op), and obvious in intent for state protection. A channel composes cleanly with `select`, carries data with synchronization, and naturally handles cancellation.

## Syntax & Basic Usage

### Mutex-protected counter

```go
package main

import "sync"

type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc()       { c.mu.Lock(); c.n++; c.mu.Unlock() }
func (c *Counter) Value() int { c.mu.Lock(); defer c.mu.Unlock(); return c.n }
```

### Channel-orchestrated counter (don't do this)

```go
type ChanCounter struct {
	inc chan struct{}
	get chan chan int
}

func NewChanCounter() *ChanCounter {
	c := &ChanCounter{inc: make(chan struct{}), get: make(chan chan int)}
	go func() {
		n := 0
		for {
			select {
			case <-c.inc:
				n++
			case reply := <-c.get:
				reply <- n
			}
		}
	}()
	return c
}

func (c *ChanCounter) Inc()       { c.inc <- struct{}{} }
func (c *ChanCounter) Value() int { r := make(chan int); c.get <- r; return <-r }
```

This works. It's also ~10–50× slower, allocates per-call, leaks a goroutine forever (no cancellation), and obscures the intent. The Mutex version is the right answer.

### Channel as ownership transfer (do this)

```go
type Job struct{}
func worker(jobs <-chan Job) {
	for j := range jobs {
		process(j)
	}
}
func process(Job) {}
```

Here the channel really does transfer a `Job` from producer to one of many workers. There is no shared state to lock; the job is owned by whoever currently holds it. The channel is the right tool.

## Deep Dive

### The slogan, restated

Rob Pike's original phrasing was deliberately provocative; the Go team's later refinement is on the Go wiki:

> Channels are a good fit for transferring ownership of data, distributing units of work, and communicating asynchronous results. Mutexes are a good fit for caches, registries, and any state that's shared and frequently mutated.

The wiki adds: "Use whichever is most expressive and/or most simple."

### Cost comparison

Rough numbers on modern x86 (your mileage varies):

| Op                                       | Cost     |
|------------------------------------------|----------|
| `mu.Lock()` / `Unlock()` uncontended     | ~10–15 ns |
| `atomic.Int64.Add(1)`                    | ~5 ns    |
| Unbuffered chan send + recv (matched)    | ~150 ns  |
| Buffered chan send (room) / recv (item)  | ~50 ns   |
| Select with 2 cases, one ready           | ~50 ns   |
| Select with 2 cases, parking + wake      | ~µs      |

Channels are inherently more expensive because they're built **on top of** mutexes, plus the goroutine park/wake bookkeeping.

### Profiling proof points

`sync.Map`'s docs explicitly say "most code should use a plain map ... with separate locking." The Go team profiled the alternatives. Same conclusion for channels-as-locks.

Damian Gryski's `go-perfbook` and Dmitry Vyukov's benchmarks show 5–50× channel-vs-mutex overhead for guard-state workloads.

### Where channels really win

1. **Pipelines.** Stages naturally compose with channels; mutexes don't.
2. **Cancellation.** `<-ctx.Done()` is a channel; mixing into a `select` with other channels is the killer feature.
3. **Bounded queues with backpressure.** A buffered channel of capacity N is a perfectly good bounded queue. Hand-rolling one with a mutex + slice is more code.
4. **Producer-consumer hand-off.** When the work item travels from one goroutine to another, the channel both transfers and synchronizes.
5. **Broadcast via close.** `close(done)` wakes every waiter exactly once; cheap to implement with a channel, awkward with mutex + cond.

### Where mutexes really win

1. **Per-key cache or registry.** Map + mutex is ~50 lines of clear code. A channel version is a small goroutine + RPC-like API.
2. **Compound state updates.** `total += amount; lastUpdated = now()`. One critical section, two writes.
3. **Read-heavy, write-rare.** `sync.RWMutex` or copy-on-write with `atomic.Pointer`. Channels can't beat this.
4. **Pure counters.** `atomic.Int64`. Don't even reach for the mutex.
5. **Lazy initialization.** `sync.Once` (or `sync.OnceValue`) is purpose-built.

### "Don't communicate by sharing memory" — when to ignore it

The slogan is true when you have multiple goroutines that need to **transfer ownership** of a value. It's misleading when you have a single piece of state that many goroutines need to *read and update*. In the latter case, you are sharing memory; pretending you aren't by hiding it behind a channel doesn't make it not so — it just adds a hop.

### Channels can become the bottleneck

A single channel between N producers and one consumer serializes everything at the channel. If your consumer can do 1M ops/sec but you need 10M, sharding the channel (multiple consumers, each reading from a dedicated channel, with producers hash-distributing) is the typical fix. Same is true of a hot mutex — shard.

### Mixed designs

It's common to use both:

```go
type Server struct {
	mu     sync.Mutex
	conns  map[string]*Conn
	events chan Event
}
```

A mutex guards the `conns` map (a registry). A channel transfers events from one place to another. Each primitive does what it's best at.

### "Confine to one goroutine"

A third option, often the best: **don't share state**. Give each goroutine its own copy or partition of the data. No mutex, no channel, no contention. The producer of state has no contenders. This is the model behind, e.g., per-P caches (`mcache`, `sync.Pool` locals).

## Standard Library Hooks

- `sync.Mutex`, `sync.RWMutex` — for guarding state.
- `sync/atomic` — for single-word state.
- Channels — for ownership transfer and signaling.
- `select` — multiplexing channels (and timeouts).
- `golang.org/x/sync/errgroup` — combines goroutines, errors, and context. Uses neither model in user code; under the hood it's a `WaitGroup` plus an atomic error.
- `sync.Cond` — third option for "wait for predicate" with shared state.

## Real-World Patterns

### 1. Cache (mutex wins)

```go
type Cache[K comparable, V any] struct {
	mu sync.RWMutex
	m  map[K]V
}

func (c *Cache[K, V]) Get(k K) (V, bool) {
	c.mu.RLock(); defer c.mu.RUnlock()
	v, ok := c.m[k]
	return v, ok
}
func (c *Cache[K, V]) Set(k K, v V) {
	c.mu.Lock(); defer c.mu.Unlock()
	c.m[k] = v
}
```

A channel-orchestrated version would route every Get and Set through a single owner goroutine. That bottlenecks at one core and adds latency. Mutex is right.

### 2. Pipeline (channel wins)

```go
func pipeline(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case <-ctx.Done(): return
			case out <- v*v:
			}
		}
	}()
	return out
}
```

Each stage owns its output. Mutex makes no sense here — there's no shared mutable state.

### 3. Worker pool with backpressure (channel wins)

```go
jobs := make(chan Job, 1000)

for w := 0; w < 16; w++ {
	go func() {
		for j := range jobs {
			process(j)
		}
	}()
}

for _, j := range incoming {
	jobs <- j
}
close(jobs)
```

The channel is both the queue and the synchronization. A mutex-protected slice would need its own condition variable to block producers and wake consumers — reinventing the channel.

### 4. Counter (atomics win, mutex second, channel terrible)

```go
type Counter struct{ n atomic.Int64 }
func (c *Counter) Inc() { c.n.Add(1) }
func (c *Counter) Value() int64 { return c.n.Load() }
```

For a single integer, `atomic` beats `Mutex` beats `channel` by 5×, 10×, and 50× respectively.

### 5. Done signal (channel wins)

```go
done := make(chan struct{})
go func() {
	work()
	close(done)
}()
<-done
```

`close(done)` is a broadcast. With a mutex + cond you'd need `cond.Broadcast()` plus careful predicate-loop handling. Channel is cleaner.

### 6. Bounded semaphore (either works)

```go
sem := make(chan struct{}, 10)
sem <- struct{}{}
defer func() { <-sem }()
work()
```

vs.

```go
import "golang.org/x/sync/semaphore"
s := semaphore.NewWeighted(10)
s.Acquire(ctx, 1); defer s.Release(1)
work()
```

The channel version is more idiomatic; the `semaphore.Weighted` version supports context-aware acquire and weights. Both are fine.

## Anti-Patterns & Gotchas

**"It's more Go-like to use a channel."** No. It's more Go-like to pick the right tool. Mutex on a counter is idiomatic.

**Hand-rolling a mutex with a `chan struct{}` of capacity 1.** Slower, no `TryLock`, no starvation mode, no priority inversion handling, no `go vet -copylocks`. Use `sync.Mutex`.

**Channel-orchestrating a registry.** Every read becomes an RPC. Use a mutex + map.

**Mutex around a pipeline.** You've serialized everything. Use channels.

**Comparing benchmarks of channel vs mutex *under contention* without thinking about contention shape.** A "fair" benchmark also has to test the actual workload — if your real workload is read-heavy, RWMutex or atomic.Pointer wins. If it's producer/consumer, channels win.

**Forgetting that channel sends and receives can park.** A "fast" channel becomes a slow channel the moment it has to block. Profile real workloads.

**Holding a mutex across a channel send.** Recipe for deadlock if the receiver tries to call back into the locked code.

**Using a channel to invalidate a cache entry, when a CAS would do.** "Atomic single-word update" deserves atomics, not a goroutine + channel.

**Reaching for `sync.Cond` instead of a channel when channels would do.** Channels usually win for "wait for ready" patterns because they integrate with `select`.

## Performance Notes

- Per-op cost ratio (uncontended): atomic ~1 < mutex ~3 < buffered chan ~15 < unbuffered chan ~30 < select with park/wake ~100.
- Under heavy contention, every primitive degrades: the mutex spins-then-parks (best), the channel parks immediately (more expensive), the atomic loops on CAS (forever, in pathological cases).
- A "hot" mutex on a 16-core box can cost ~µs per op due to cache-line bouncing — and a channel will do the same or worse for the same workload.
- Sharding works for both: N mutexes or N channels, hashed by key.
- Trace your program (`go test -bench -trace=trace.out`) to see where time goes. The visualization makes the right choice obvious.

## How Big Companies Use It

- **`net/http`** uses both: a Mutex for the server's connection set; channels for `Server.Serve`'s shutdown signal.
- **Kubernetes `client-go`** workqueues use a Mutex + condition variable + Set, *not* channels — because they need O(1) "is key already queued?" which is awkward over channels.
- **etcd raft** uses channels heavily for the `Node` interface (proposals, ready events) and mutexes inside the raft state machine itself.
- **CockroachDB** uses sharded Mutexes for hot per-range state; channels for SQL stream merges.
- **Cloudflare's blog "Scaling Go applications"** dedicates a section to "don't use channels for what mutexes do": https://blog.cloudflare.com/scaling-go-applications/.
- **Bryan C. Mills's "Rethinking Classical Concurrency Patterns"** explicitly de-emphasizes channels for state and prefers mutex/atomic for many patterns: https://www.youtube.com/watch?v=5zXAHh5tJqQ.
- **Discord's "switching from Go to Rust"** post mentions both channel and mutex paths in their hot loop: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.

## Source Code References

Pinned to `go1.26`.

- `sync.Mutex` implementation: [`src/sync/mutex.go`](https://github.com/golang/go/blob/master/src/sync/mutex.go).
- Channel implementation: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go).
- `select` implementation: [`src/runtime/select.go`](https://github.com/golang/go/blob/master/src/runtime/select.go).
- `sync/atomic`: [`src/sync/atomic/`](https://github.com/golang/go/tree/master/src/sync/atomic).
- `golang.org/x/sync/semaphore` (channel-backed semaphore reference): [`semaphore/semaphore.go`](https://github.com/golang/sync/blob/master/semaphore/semaphore.go).

## Further Reading

- Go wiki, "Mutex Or Channel?": https://go.dev/wiki/MutexOrChannel
- Rob Pike, "Share Memory By Communicating": https://go.dev/blog/codelab-share
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns": https://www.youtube.com/watch?v=5zXAHh5tJqQ
- Dave Cheney, "Channels are not enough or what would you really want from a concurrent language?" (2014): https://dave.cheney.net/
- Damian Gryski, "go-perfbook" — mutex vs channel: https://github.com/dgryski/go-perfbook
- Filippo Valsorda on choosing primitives: https://words.filippo.io/
- Sameer Ajmani, "Go Concurrency Patterns": https://go.dev/talks/2012/concurrency.slide

## Exercises / Self-Check

1. Implement a thread-safe registry (`Register(k, v)`, `Lookup(k)`, `Delete(k)`) using a Mutex. Then implement it using a single owner goroutine + channels. Benchmark both. What did you observe?
2. Why is a `chan struct{}` of capacity 1 a worse mutex than `sync.Mutex`?
3. Convert a worker pool with mutex-protected slice queue to one using a buffered channel. Does the API stay the same?
4. When would `sync.Cond` beat both? Give an example.
5. Profile a contended `sync.Mutex` vs `sync.RWMutex` vs `atomic.Pointer` for a read-mostly map. Which wins at what read:write ratio?
