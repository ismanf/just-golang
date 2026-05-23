# Goroutines

## TL;DR

A goroutine is a function that runs concurrently with other goroutines, multiplexed by the Go runtime onto a small pool of OS threads. You spawn one with `go f(args)`. It starts with a tiny (~2 KiB) growable stack and costs roughly an order of magnitude less than an OS thread. The single biggest gotcha: `go` returns immediately; the parent has no built-in handle to wait on, recover from, or cancel the child — you must supply that scaffolding (`sync.WaitGroup`, `context.Context`, channels, `errgroup`, or `WaitGroup.Go` in 1.25+).

## Mental Model

```
+-------------------+          +-------------------+
|   main goroutine  |          |  goroutine #42    |
+---------+---------+          +---------+---------+
          |                              |
          |  go work(x)                  |
          +----------------------------->+ (scheduled into local run queue of a P)
          |                              |
   continues immediately          eventually runs on some M (OS thread)
```

The Go scheduler is the **G-M-P** model: a `G` is a goroutine, an `M` is an OS thread, a `P` is a logical processor (a scheduling context). At any moment, at most `GOMAXPROCS` Ps are running Gs on Ms in parallel. When a G blocks on I/O, syscall, channel, or lock, the runtime parks it and schedules another G on the same M (or hands the P off to a different M). See `11-scheduler-internals.md` for the full picture.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := range 5 { // since 1.22: range-over-int
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			fmt.Println("hello from", n)
		}(i)
	}
	wg.Wait()
	// Output (order non-deterministic):
	// hello from 0
	// hello from 1
	// hello from 2
	// hello from 3
	// hello from 4
}
```

Since Go 1.22 the loop variable `i` is per-iteration, so passing it explicitly is no longer required for correctness — but doing so is still the clearest idiom. Since Go 1.25 you can write `wg.Go(func() { ... })` and skip the `Add`/`Done` boilerplate (see `05-sync-waitgroup-once-cond.md`).

## Deep Dive

### Lifecycle

A goroutine's life:
1. Creation by `go` (compiled to `runtime.newproc`).
2. Placed on the creating P's local run queue.
3. Scheduled on some M when its turn comes (the runtime may also work-steal it from another P's queue).
4. Runs until it returns, blocks, or is preempted.
5. On return, `runtime.goexit` cleans up its stack and signals the scheduler.

Returning from `main` terminates the program *and every goroutine* — even ones mid-syscall. Goroutines do not get a chance to run deferred functions when `main` returns. If you need clean shutdown, you orchestrate it.

### Stacks grow

A goroutine starts with a ~2 KiB stack (`_StackMin` in `runtime/stack.go`). At each function prologue the compiler inserts a stack-bound check; if the new frame would overflow, the runtime allocates a bigger stack (doubling), copies the old frames over, rewrites pointers into the stack, and resumes. The old stack is freed. This is why pointers to stack-allocated locals are safe even across grows.

Stacks can also shrink during GC if more than 1/4 of the stack is unused.

The hard ceiling is `runtime/debug.SetMaxStack` (default 1 GiB on 64-bit). Hit it and you get a runtime panic: `runtime: goroutine stack exceeds 1000000000-byte limit`.

### Spawning cost

Spawning a goroutine is on the order of ~1 µs and ~2 KiB. Spawning an OS thread is ~10–100 µs and the thread costs ~2 MiB of virtual address space. You can comfortably have hundreds of thousands to a few million goroutines on a beefy server; you cannot have a million OS threads. That asymmetry is the whole pitch.

### Preemption

Pre-1.14, a goroutine could only be preempted at function call boundaries (the same stack-bound check that triggers growth). A `for {}` with no calls would peg a core forever. Since **Go 1.14**, the runtime uses **asynchronous preemption** via signals (`SIGURG` on Unix). The `sysmon` thread sends the signal; the signal handler steals the goroutine's PC, redirects to `asyncPreempt`, and the scheduler decides whether to yield.

### `go` evaluates arguments at call time

```go
x := 1
go func() { fmt.Println(x) }() // captures x by reference
x = 2

x = 1
go func(v int) { fmt.Println(v) }(x) // copies 1 at the go statement
x = 2
```

The first prints either `1` or `2` depending on scheduling. The second always prints `1`. Same closure-vs-argument distinction as any function call, but it bites more under concurrency.

### Panic in a goroutine kills the program

A panic that propagates out of any goroutine — not just `main` — crashes the entire process. There is no parent-child exception channel. Either install a `defer recover()` at the top of every spawned function, or design so that those goroutines truly cannot panic. See `05-panic-and-recover.md`.

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("worker panic: %v\n%s", r, debug.Stack())
		}
	}()
	work()
}()
```

## Standard Library Hooks

- `sync.WaitGroup` — wait for a set of goroutines to finish. `WaitGroup.Go` (1.25+) replaces the `Add`/`Done` dance.
- `context.Context` — cancellation and deadline propagation. Every long-lived goroutine should accept a `ctx`.
- `runtime.NumGoroutine()` — current count. Useful in tests and leak detectors.
- `runtime.Goexit()` — terminate the current goroutine after running its deferred functions. Does *not* kill the program; `main` calling `Goexit` is the one case where the program exits with `no goroutines` deadlock.
- `runtime.Gosched()` — voluntarily yield. Rarely needed since 1.14 preemption.
- `runtime.LockOSThread()` / `UnlockOSThread()` — pin a goroutine to its current OS thread. Required for OpenGL, some signal handling, and `runtime.SetMutexProfileFraction` callsites that touch thread-local state in C.
- `runtime/pprof.Do(ctx, labels, fn)` — attach pprof labels to whatever goroutine runs `fn`.
- `golang.org/x/sync/errgroup` — `WaitGroup` plus first-error capture plus context cancellation. See `13-errgroup-and-singleflight.md`.

## Real-World Patterns

### 1. Detached worker that respects shutdown

```go
package main

import (
	"context"
	"log/slog"
	"time"
)

func startReaper(ctx context.Context, every time.Duration, do func()) {
	go func() {
		t := time.NewTicker(every)
		defer t.Stop()
		for {
			select {
			case <-ctx.Done():
				slog.Info("reaper stopping", "cause", context.Cause(ctx))
				return
			case <-t.C:
				do()
			}
		}
	}()
}
```

Note the `select { case <-ctx.Done(): return }` — that branch is the only thing standing between you and a goroutine leak.

### 2. Bounded fan-out

```go
package main

import "sync"

func parallelMap[T, U any](in []T, n int, f func(T) U) []U {
	out := make([]U, len(in))
	sem := make(chan struct{}, n)
	var wg sync.WaitGroup
	for i, v := range in {
		wg.Add(1)
		sem <- struct{}{}
		go func(i int, v T) {
			defer wg.Done()
			defer func() { <-sem }()
			out[i] = f(v)
		}(i, v)
	}
	wg.Wait()
	return out
}
```

A buffered channel as a counting semaphore limits concurrency to `n`. See `12-concurrency-patterns.md` for the canonical worker-pool version.

### 3. Safe recover wrapper

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func Go(name string, fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("goroutine %s panicked: %v\n%s\n", name, r, debug.Stack())
			}
		}()
		fn()
	}()
}
```

A house-style `Go(...)` like this is in nearly every production codebase. It pairs well with labels:

```go
import "runtime/pprof"
pprof.Do(ctx, pprof.Labels("worker", name), func(ctx context.Context) { fn(ctx) })
```

so `pprof` shows you which worker is hot.

## Anti-Patterns & Gotchas

**Spawning unbounded goroutines from a request handler.** `for _, x := range items { go process(x) }` on a 100k-item slice will allocate 100k goroutines. Use a bounded worker pool.

**No way to wait, no way to cancel.** `go work()` and forget. If `work` runs forever or holds resources, you've leaked. Always pair `go` with a `WaitGroup`/`errgroup` and a `ctx`.

**Capturing the loop variable pre-1.22.** `for i := 0; i < n; i++ { go func() { use(i) }() }` printed `n` over and over before Go 1.22 because all closures captured the same `i`. Since 1.22 the variable is per-iteration so this works. **But** the same trap still exists for non-`for`-loop closures over later-mutated variables. When in doubt, pass as an argument.

**Calling `runtime.Goexit()` in `main`.** It runs deferred functions then terminates the goroutine. If no other goroutine is keeping the program alive you get `fatal error: no goroutines (main called runtime.Goexit) - deadlock!`.

**Assuming `GOMAXPROCS` goroutines run truly in parallel.** They do, but blocking calls (syscalls, channels, locks) park them and free the P for someone else. CPU-bound work is what hits the parallelism limit.

**Forgetting that an uncaught panic kills the world.** Library code that spawns goroutines without `recover` is a foot-cannon for its users.

**Sleeping to "wait" for a goroutine.** `go f(); time.Sleep(time.Second); ...` is a flake generator. Use a `WaitGroup`, channel, or `errgroup`.

## Performance Notes

- Spawn cost: ~1 µs on modern hardware, dominated by `runtime.newproc` and the `g` struct allocation.
- Context-switch cost: ~200–500 ns for a goroutine-to-goroutine switch on the same M (cooperative). Async preemption via signal is more like a few µs.
- The `g` struct itself is ~256 bytes; the initial stack is 2 KiB. So a goroutine "costs" ~2.25 KiB at minimum.
- The scheduler's local run queue is 256 entries per P. Overflowing goes to the global queue, which has a lock and is slower. Don't enqueue millions of goroutines at once.
- `runtime.Gosched` is much cheaper than `time.Sleep(0)` if you actually want to yield — but you almost never need either.
- Idle goroutines (parked on channel/lock) cost nothing CPU-side, but their stacks are still resident memory until GC shrinks them.

## How Big Companies Use It

- **Kubernetes API server** spawns one goroutine per long-poll watch. The 1.21 GC and scheduler changes were partly tuned around its goroutine count profile. See [`apiserver/pkg/storage/cacher`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/apiserver/pkg/storage/cacher).
- **Docker/containerd** uses one goroutine per running container for I/O multiplexing; the `shim` design is a deliberate goroutine-budget choice.
- **Tailscale's `wgengine`** uses goroutine-per-flow for WireGuard packet processing. Brad Fitzpatrick has written about deliberately keeping goroutine counts predictable.
- **Cloudflare** documented goroutine-explosion incidents in their HTTP/2 stack: https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/.
- **Discord's "switching from Go to Rust"** post is partly a story about millions of goroutines stressing the (pre-1.20) GC: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.

## Source Code References

Pinned to `go1.26` — substitute the real release tag when consulting.

- `go` statement codegen → `runtime.newproc`: [`src/runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go) (search for `func newproc`).
- The `g` struct: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/master/src/runtime/runtime2.go) (search `type g struct`).
- Stack growth: [`src/runtime/stack.go`](https://github.com/golang/go/blob/master/src/runtime/stack.go), function `newstack`, `copystack`, `morestack`.
- Asynchronous preemption: [`src/runtime/preempt.go`](https://github.com/golang/go/blob/master/src/runtime/preempt.go).
- `Goexit`: [`src/runtime/panic.go`](https://github.com/golang/go/blob/master/src/runtime/panic.go), function `Goexit`.
- `runtime.LockOSThread`: [`src/runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go), search `LockOSThread`.

## Further Reading

- Go spec, "Go statements": https://go.dev/ref/spec#Go_statements
- Go memory model: https://go.dev/ref/mem
- Dmitry Vyukov, "Scalable Go Scheduler Design Doc" (2012): https://golang.org/s/go11sched
- Austin Clements et al., "Non-cooperative goroutine preemption" (proposal #24543): https://go.dev/issue/24543
- Russ Cox, "Goroutines, threads, and stacks" (research!rsc): https://research.swtch.com/gostack
- Dave Cheney, "Five things that make Go fast" (goroutines section): https://dave.cheney.net/2014/06/07/five-things-that-make-go-fast
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns" (GopherCon 2018): https://www.youtube.com/watch?v=5zXAHh5tJqQ
- Filippo Valsorda, "How Go's stack grows": https://blog.filippo.io/the-curious-case-of-the-yellow-truck/

## Exercises / Self-Check

1. What is printed by `for i := 0; i < 3; i++ { go func() { fmt.Println(i) }() }; time.Sleep(time.Second)` on Go 1.22+? On 1.21? Why?
2. Write `SafeGo(fn func()) <-chan error` that runs `fn` in a goroutine, recovers any panic, and returns a channel that yields nil or the panic value as an error.
3. A program creates a goroutine that blocks on `<-make(chan struct{})`. The main function returns. Does the goroutine ever run its `defer`? Why?
4. Sketch the scheduler state (Gs, Ms, Ps, run queues) after `go work()` is called four times with `GOMAXPROCS=2`.
5. Why does asynchronous preemption use a signal instead of a flag the goroutine checks itself?
