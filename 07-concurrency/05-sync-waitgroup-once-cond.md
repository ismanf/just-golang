# `sync.WaitGroup`, `sync.Once`, `sync.Cond`

## TL;DR

`sync.WaitGroup` counts in-flight goroutines so the parent can wait for "all done". `sync.Once` runs a function exactly once across all callers, regardless of contention. `sync.Cond` is a condition variable for the (rare) cases when channels don't fit — wait for a predicate to become true. Since **Go 1.25**, `WaitGroup.Go(func())` lets you skip the manual `Add`/`Done` dance. The biggest gotcha across all three: do **not** copy them after first use.

## Mental Model

```
WaitGroup
+---------------------------------------+
| state atomic.Uint64                   |  // packs counter (high 32) + waiters (low 32)
| sema  uint32                          |
+---------------------------------------+
Add(n): counter += n. If counter < 0 → panic. If counter==0 and waiters>0, wake them.
Done():  Add(-1).
Wait():  if counter==0 return. else waiters++; park on sema.

Once
+--------------+
| done atomic  |  // 0 or 1
| m    Mutex   |
+--------------+
Do(f): fast path: if done==1, return. slow path: lock, double-check, run f, set done=1.

Cond
+--------------+
| L   Locker   |  // user-supplied (Mutex or RWMutex)
| notify list  |  // FIFO waiters
+--------------+
Wait(): c.L.Unlock(); park; c.L.Lock() — caller must hold L when calling.
Signal(): wake one waiter. Broadcast(): wake all.
```

## Syntax & Basic Usage

### WaitGroup — classic API

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := range 3 {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			fmt.Println("done", n)
		}(i)
	}
	wg.Wait()
	// Output (order non-deterministic):
	// done 0
	// done 1
	// done 2
}
```

### WaitGroup.Go — since Go 1.25

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := range 3 {
		wg.Go(func() {
			fmt.Println("done", i)
		})
	}
	wg.Wait()
}
```

`wg.Go(fn)` is `wg.Add(1)` + `go func() { defer wg.Done(); fn() }()`. Eliminates the easiest concurrency bug — forgotten `Add` or misplaced `defer Done`. Note that the `i` capture here is per-iteration (since Go 1.22).

### Once

```go
package main

import (
	"fmt"
	"sync"
)

var (
	once sync.Once
	cfg  *Config
)

type Config struct{ Host string }

func Get() *Config {
	once.Do(func() {
		cfg = &Config{Host: "example.com"}
	})
	return cfg
}

func main() { fmt.Println(Get().Host) }
```

### Cond

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var mu sync.Mutex
	cond := sync.NewCond(&mu)
	ready := false

	go func() {
		mu.Lock()
		ready = true
		cond.Broadcast()
		mu.Unlock()
	}()

	mu.Lock()
	for !ready {
		cond.Wait()
	}
	mu.Unlock()
	fmt.Println("ready")
}
```

The `for !ready` loop is mandatory — `Wait` can return spuriously, and the predicate must be re-checked.

## Deep Dive

### WaitGroup internals

`state` packs a 32-bit counter and a 32-bit waiter count into one `atomic.Uint64`. `Add(delta)` does an atomic add on the high 32 bits, then if the counter hit zero and waiters > 0, it wakes them via the semaphore. `Wait` atomically increments the low 32 bits and parks.

Pre-1.25, the most common WaitGroup bug was calling `wg.Add(1)` **inside** the goroutine instead of before `go`:

```go
go func() {
	wg.Add(1)            // RACE: may run after wg.Wait()
	defer wg.Done()
	...
}()
```

The race: if all current goroutines finish and `Wait` observes counter=0, it returns — then your goroutine `Add`s and never gets waited on. The `wg.Go` method makes this category of bug structurally impossible because the `Add` happens before the `go` returns.

### WaitGroup reuse

You can reuse a `WaitGroup` after `Wait` returns — but only if there are no pending `Add` calls that haven't been observed. The safe sequence: `Add → spawn → Done → Wait → Add → spawn → Done → Wait → ...`. Interleaving `Add` and `Wait` racily is the same bug as before.

### WaitGroup.Go and panics

`WaitGroup.Go` documentation explicitly states: if the function panics, the panic propagates after `Done` is called. The wrapped goroutine still calls `Done`, so `Wait` will return — but the program will then crash from the panic. If you need to capture errors, use `errgroup.Group.Go` instead (see `13-errgroup-and-singleflight.md`).

### Once internals — the fast path

```go
type Once struct {
	done atomic.Uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if o.done.Load() == 0 {
		o.doSlow(f)
	}
}
```

The atomic load is ~3 ns on modern hardware. The slow path takes the mutex, re-checks `done`, runs `f`, then atomically stores 1. The double-check is the standard pattern.

If `f` panics, `done` is **still set to 1** — subsequent calls do nothing. The panic still propagates from the call that ran `f`. So `Once` does not retry on failure. If you need retry-until-success, use `sync.Mutex` directly and an explicit "did it succeed?" flag.

### `Once.OnceFunc`, `OnceValue`, `OnceValues` (since 1.21)

```go
init := sync.OnceFunc(func() { log.Println("once!") })
get := sync.OnceValue(func() *Config { return loadConfig() })
get2 := sync.OnceValues(func() (*Config, error) { return loadCfg() })

init(); init() // logs once
c := get()     // loads once; subsequent calls return the cached value
```

Cleaner than the original `var once sync.Once; var v T` boilerplate. The wrapped function is called exactly once; the captured result (or error) is memoized and returned to all callers.

### Cond — why it's rare

`Cond` is for "wait for a predicate", which is what channels already do well. Use `Cond` when:
- The predicate involves shared mutable state that's already mutex-protected.
- You want `Broadcast` semantics with cheap wakeup of many waiters.
- A channel would require N "ready" channels per consumer or other awkwardness.

The Go team has historically discouraged it (Russ Cox: "I'm not sure I've ever used `sync.Cond`"), but it's idiomatic for things like bounded queues that can't be expressed as a single channel.

### Cond pitfalls

- **Must hold the lock when calling Wait, Signal, Broadcast**. (Wait release-acquires the lock; Signal/Broadcast don't require it but you should hold it to make the predicate atomic vs the wake.)
- **Spurious wakeups happen.** Always re-check the predicate in a `for` loop.
- **Cond doesn't notify late joiners.** A goroutine that calls `Wait` after a `Broadcast` will wait for the next one. Channels (with close) broadcast permanently.
- **Cannot be copied after first use.** Same as the others.

### Don't copy these

All three types embed `noCopy` (or its equivalent state). `go vet -copylocks` flags accidental copies. The fix is to store pointers or to keep them in a fixed allocation that you only pass by pointer.

## Standard Library Hooks

- `sync.WaitGroup`, `sync.WaitGroup.Go` (1.25+).
- `sync.Once`, `sync.OnceFunc`, `sync.OnceValue`, `sync.OnceValues` (1.21+).
- `sync.Cond`, `sync.NewCond`.
- `golang.org/x/sync/errgroup` — `Group.Go(func() error) error` if you also need first-error capture and context cancellation.
- `runtime/debug.SetTraceback("all")` — when a `WaitGroup.Wait` hangs, this dumps every goroutine's stack on signal.

## Real-World Patterns

### 1. Spawn-and-wait with WaitGroup.Go (1.25+)

```go
package main

import (
	"net/http"
	"sync"
)

func warmUp(urls []string) {
	var wg sync.WaitGroup
	for _, u := range urls {
		wg.Go(func() {
			_, _ = http.Get(u) // ignore errors
		})
	}
	wg.Wait()
}
```

Pre-1.25 you'd write `wg.Add(1)` and `defer wg.Done()` for each spawn. The 1.25 form is one line and structurally safe.

### 2. Lazy singleton

```go
package config

import "sync"

var load = sync.OnceValue(func() *Config {
	return mustLoad()
})

func Get() *Config { return load() }
```

`OnceValue` produces a function with the same `Get`-shape any client code expects. No package-level mutable state visible to clients.

### 3. Cond-driven bounded queue

```go
package queue

import "sync"

type Bounded[T any] struct {
	mu    sync.Mutex
	notEmpty, notFull *sync.Cond
	items []T
	cap   int
}

func New[T any](cap int) *Bounded[T] {
	b := &Bounded[T]{cap: cap}
	b.notEmpty = sync.NewCond(&b.mu)
	b.notFull = sync.NewCond(&b.mu)
	return b
}

func (b *Bounded[T]) Push(v T) {
	b.mu.Lock()
	for len(b.items) == b.cap {
		b.notFull.Wait()
	}
	b.items = append(b.items, v)
	b.notEmpty.Signal()
	b.mu.Unlock()
}

func (b *Bounded[T]) Pop() T {
	b.mu.Lock()
	for len(b.items) == 0 {
		b.notEmpty.Wait()
	}
	v := b.items[0]
	b.items = b.items[1:]
	b.notFull.Signal()
	b.mu.Unlock()
	return v
}
```

A buffered channel does the same job in fewer lines, **but**: this version exposes `Len`, allows arbitrary policy on the internal slice, and lets multiple consumers wake selectively via `Signal` vs `Broadcast`.

### 4. Fan-out + result collection

```go
func crawl(urls []string) []Result {
	var wg sync.WaitGroup
	results := make([]Result, len(urls))
	for i, u := range urls {
		wg.Go(func() {
			results[i] = fetch(u) // each goroutine writes its own slot — no race
		})
	}
	wg.Wait()
	return results
}
```

Each goroutine writes to its own index of the pre-allocated slice. No mutex needed because no two goroutines touch the same element. This pattern is widely used and a common interview question.

### 5. Idempotent shutdown with OnceFunc

```go
type Server struct {
	stop chan struct{}
	once sync.Once
}

func (s *Server) Stop() {
	s.once.Do(func() { close(s.stop) })
}
```

Or with 1.21:

```go
type Server struct {
	stop chan struct{}
	Stop func()
}

func New() *Server {
	s := &Server{stop: make(chan struct{})}
	s.Stop = sync.OnceFunc(func() { close(s.stop) })
	return s
}
```

The second form makes the idempotency obvious at the call site.

## Anti-Patterns & Gotchas

**`wg.Add(1)` inside the goroutine.** Race; possibly the most-common WaitGroup bug. Use `wg.Go` (1.25+) or always `Add` before `go`.

**Calling `wg.Wait` then `wg.Add` from the same goroutine without synchronization.** Same race.

**Negative counter panic.** Calling `Done` more times than `Add` panics with `sync: negative WaitGroup counter`.

**Forgetting `defer wg.Done()`.** If the goroutine body can panic or return early, `Done` never runs and `Wait` hangs. Always `defer`.

**Once that's expected to retry on failure.** `Once.Do(f)` sets `done` even if `f` panics or fails. If you want retry semantics, don't use `Once` — use a mutex with explicit success tracking.

**Cond without a for-loop predicate check.** Spurious wakeup or interleaved state changes will burn you. Always `for !predicate { c.Wait() }`.

**Cond.Signal without holding the lock.** The waiter could observe stale predicate state. Always hold the associated lock during signal/broadcast unless you've reasoned very carefully about the ordering.

**Copying a `WaitGroup`/`Once`/`Cond` by value.** Two independent objects; locking one doesn't affect the other. `go vet` catches most of these via `copylocks`.

**Re-using a `Once` across resets.** Once it's done, it stays done forever. If you need resettable, build your own with mutex + flag.

**Passing `wg` by value to a goroutine.** Common pre-pointer-discipline bug. Pass `*sync.WaitGroup` or capture by closure.

## Performance Notes

- `WaitGroup.Add`/`Done`: one atomic add each, ~5 ns. `Wait` on counter==0: ~3 ns. With waiters parked: scheduler cost.
- `Once.Do` fast path: one atomic load, ~3 ns. Slow path runs the function plus mutex overhead.
- `Cond.Signal`: one wake, ~µs. `Broadcast`: one wake per waiter.
- `OnceValue` / `OnceValues` return cached values without taking the mutex on every call — the fast path is identical to `Once`.
- `WaitGroup.Go` is essentially the same cost as the hand-written equivalent; the wrapper is inlined.

## How Big Companies Use It

- **Kubernetes** uses `sync.WaitGroup` heavily in controllers and informer shutdown.
- **gRPC-Go** uses `sync.Once` for one-time stream initialization and shutdown.
- **etcd** uses `sync.Cond` in a few places where multiple goroutines wait on raft state transitions.
- **Docker** uses `OnceFunc`/`OnceValue` since 1.21 in many init paths — search `sync.OnceValue` in https://github.com/moby/moby.
- **HashiCorp's `consul`** uses `sync.WaitGroup` to coordinate leader-election goroutines.

## Source Code References

Pinned to `go1.26`.

- `WaitGroup`, including `Go` (1.25+): [`src/sync/waitgroup.go`](https://github.com/golang/go/blob/master/src/sync/waitgroup.go).
- `Once` and `OnceFunc`/`OnceValue`/`OnceValues`: [`src/sync/once.go`](https://github.com/golang/go/blob/master/src/sync/once.go), [`src/sync/oncefunc.go`](https://github.com/golang/go/blob/master/src/sync/oncefunc.go).
- `Cond`: [`src/sync/cond.go`](https://github.com/golang/go/blob/master/src/sync/cond.go).
- `noCopy` marker: [`src/sync/cond.go`](https://github.com/golang/go/blob/master/src/sync/cond.go), bottom.
- Runtime notify list (Cond's wait queue): [`src/runtime/sema.go`](https://github.com/golang/go/blob/master/src/runtime/sema.go), `notifyList`.

## Further Reading

- Proposal: `sync.WaitGroup.Go`: https://go.dev/issue/63796
- Proposal: `sync.OnceFunc/Value/Values`: https://go.dev/issue/56102
- Go memory model — sync section: https://go.dev/ref/mem#sync
- Russ Cox on lazy singletons: https://research.swtch.com/lazyinit
- Bryan C. Mills, "Concurrency hazards": https://github.com/bcmills/go-concurrency-patterns
- Dmitry Vyukov, "Sync.WaitGroup internals": comments in the source
- Dave Cheney, "Practical Go: Real world advice for writing maintainable Go programs": https://dave.cheney.net/practical-go

## Exercises / Self-Check

1. Write a function that fans out to N workers, collects results, and returns them in input order, using `WaitGroup.Go`.
2. Why is `wg.Add(1)` inside the goroutine racy? Construct the interleaving that breaks it.
3. Implement a resettable Once. Why can't `sync.Once` be reset?
4. When would you reach for `Cond` over `make(chan struct{})`?
5. Convert `var once sync.Once; var cfg *Config; func Get() *Config { once.Do(func(){ cfg = load() }); return cfg }` to `sync.OnceValue`.
