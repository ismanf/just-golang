# Scheduler Internals

## TL;DR

The Go scheduler is the **G-M-P** model: `G` is a goroutine, `M` is an OS thread, `P` is a logical processor (a scheduling context). There are at most `GOMAXPROCS` Ps; each P has its own local run queue of Gs and is bound to at most one running M at a time. When a P's queue is empty it **work-steals** from other Ps. The runtime preempts running goroutines **asynchronously** via signals (since Go 1.14), so even a `for {}` loop yields. The single biggest gotcha: blocking syscalls park the M (not the P), letting another M take the P — this is how thousands of blocked goroutines coexist with `GOMAXPROCS=8`. Sysmon and the netpoller are the two background threads that keep this all responsive.

## Mental Model

```
        Gs (goroutines)
        ----------------
        G1  G2  G3  G4  G5 G6 G7 ...   (millions possible)

        Ps (logical processors, count = GOMAXPROCS)
        ---------------------------------------------
        P0 [local run queue: G1, G3]
        P1 [local run queue: G2]
        P2 [local run queue: empty]   <- will work-steal from P0 or P1
        P3 [local run queue: G5, G7]

        Ms (OS threads, count is dynamic, usually > GOMAXPROCS)
        --------------------------------------------------------
        M0 running G1 on P0
        M1 running G2 on P1
        M2 idle (parked) — woken when work arrives
        M3 stuck in a blocking syscall (no P bound; P given to someone else)

        Global run queue (rare, overflow)
        Network poller (netpoll) — wakes Gs blocked on I/O
        sysmon (system monitor) — preempts long-running Gs, releases stuck Ps
```

The runtime maintains: per-P run queues (256 entries each), a global run queue (overflow + initial spawn), a netpoller's ready-list, and a sysmon thread that does the housekeeping (preemption signals, releasing Ps held by syscalls, GC pacing).

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("GOMAXPROCS =", runtime.GOMAXPROCS(0))
	fmt.Println("NumCPU =", runtime.NumCPU())
	fmt.Println("NumGoroutine =", runtime.NumGoroutine())
}
```

`GOMAXPROCS(n)` sets the number of Ps. `0` returns the current value. As of **Go 1.5**, the default is `runtime.NumCPU()`. As of **Go 1.25** the runtime is **container-aware**: `GOMAXPROCS` defaults to the lower of NumCPU and the cgroup CPU quota.

## Deep Dive

### G — the goroutine struct

`runtime.g` (in `src/runtime/runtime2.go`) holds:
- `stack`: low/high addresses of the contiguous stack.
- `stackguard0`: stack overflow check value (used by function prologues).
- `m`: the M currently running this G (nil if not running).
- `sched`: saved register context (PC, SP, BP).
- `atomicstatus`: state (idle, runnable, running, syscall, waiting, dead, etc.).
- `goid`: monotonically-increasing ID.

A G is just a struct; goroutines are this struct plus a stack. Spawning one is `runtime.newproc`: allocate a `g`, copy the function arguments to its stack, enqueue.

### M — the OS thread struct

`runtime.m` holds:
- `g0`: the M's *system* goroutine, used for scheduling and stack switches (its stack is the OS thread's stack).
- `curg`: the currently running user G.
- `p`: the P this M is bound to (nil if running without one, e.g., in a syscall).
- `nextp`: a P this M will pick up after returning from syscall.

The runtime caps the number of Ms at `runtime.SetMaxThreads(10000)` by default. Hitting that limit panics with `runtime: program exceeds 10000-thread limit`.

### P — the logical processor

`runtime.p` holds:
- `runq`: a 256-entry circular buffer (the local run queue).
- `runnext`: a single-slot "next G to run" optimization for fast self-yields.
- `m`: the M currently bound to this P.
- `mcache`: a per-P allocator cache (this is why allocator paths can be lock-free).
- `gcCache`: GC scratch.

`GOMAXPROCS` Ps are created at startup (and adjusted on `runtime.GOMAXPROCS(n)`).

### The scheduling loop

`schedule()` (in `runtime/proc.go`) is what an M runs after it finishes a G:

1. Look at the P's `runnext` slot — if set, run that G.
2. Try the local run queue.
3. Every 61 ticks, check the global run queue to ensure fairness.
4. Try the netpoller — any I/O-ready Gs?
5. Work-steal: pick a random other P, take half its queue.
6. If nothing found, the M parks itself (`stopm`) waiting to be woken.

`runnext` exists for a specific reason: when goroutine A spawns B and immediately blocks (e.g., `<-ch`), the very next G to run on the same P should usually be B — direct hand-off avoids a global queue trip.

### Work stealing

A P with an empty queue spins through random other Ps, stealing half of each victim's runq. This balances load without central coordination. Cost: a few atomic ops per steal. The randomization avoids hot-spotting.

### Asynchronous preemption (Go 1.14+)

Pre-1.14, a goroutine could only be preempted at function call boundaries (the stack-bound check that also triggers stack growth). Tight CPU loops with no calls could starve everything else and block GC.

Since 1.14, sysmon sends `SIGURG` to a thread running a G that has been running >10 ms. The signal handler saves state, jumps to `asyncPreempt`, which forces the goroutine into `morestack` (the same path that grows the stack), and from there the scheduler decides whether to actually yield.

The signal-based approach is necessary because Go cannot insert software interrupts everywhere — it would bloat code and slow common paths.

### Syscalls and the P handoff

When a goroutine enters a blocking syscall (`read`, `write`, `accept`, `connect`, etc.), the runtime detaches the P from the M:

1. M's P becomes available — sysmon (or a wake-up from the syscall's M itself) hands it to another M.
2. The original M is now "in syscall, no P". When the syscall returns, the M tries to reacquire its old P; if it can't (someone else took it), it gets any P, or enqueues the G on the global queue.

This is why a server can have thousands of goroutines blocked on `read` while only `GOMAXPROCS` are running CPU-bound code: the Ps are recycled.

### The netpoller

Network I/O (and timers since 1.14) uses non-blocking syscalls. When a goroutine wants to read from a socket and the socket isn't ready, the runtime:
1. Registers the FD with epoll/kqueue/IOCP.
2. Parks the G (status → `_Gwaiting`).
3. Frees the M's P for other work.

A dedicated netpoller routine periodically calls `netpoll(0)` (non-blocking) to wake any Gs whose FDs are ready. The scheduler also runs `netpoll(blocking)` when idle — it parks the M on epoll until something happens.

So blocking on a socket costs zero CPU and zero OS threads; it costs only one parked goroutine.

### Sysmon

Sysmon (system monitor) runs on a special M without a P. Its job:
1. **Preemption**: send SIGURG to long-running Gs.
2. **Sysmon-mediated handoff**: if a P has been "in syscall" for >10 µs, hand the P to a fresh M.
3. **Network poll**: periodically poll the netpoller.
4. **GC pacing**: trigger forced GC if it's been >2 minutes.

Sysmon polls in an exponentially-increasing sleep loop — fast when busy, slow when idle.

### The global run queue

A backup for when local queues overflow (256 entries each). The global queue has a lock and is slower than local enqueue. Most workloads barely touch it; if you spawn millions of Gs at once, you'll see global-queue contention.

### GOMAXPROCS in containers (Go 1.25+)

Pre-1.25, `runtime.NumCPU()` returned the host CPU count, leading to over-subscription in containers (`GOMAXPROCS=64` in a 2-core cgroup → terrible). The fix was either Uber's `automaxprocs` library, or manual `GOMAXPROCS=2`. **Go 1.25** makes the runtime container-aware by default: it reads `/sys/fs/cgroup/cpu.max` (cgroup v2) and clamps `GOMAXPROCS` accordingly. You can opt out with `GOMAXPROCS=0` (legacy default) or set it explicitly.

### Goroutine state lifecycle

```
_Gidle  -> _Grunnable  -> _Grunning  -> _Gwaiting (chan, lock, IO)
                       -> _Gsyscall (in syscall, may still own P briefly)
                       -> _Gdead    (exited)
```

You can see states in goroutine dumps (`SIGQUIT` or `runtime.Stack`):

```
goroutine 7 [chan receive]:
main.consume(...)
   /tmp/main.go:42 +0x123
```

The bracket `[chan receive]` is the state.

### Stack growth and the schedtick

Stack growth checks happen at function prologue. The check is `if SP < stackguard0 { morestack() }`. `stackguard0` is sometimes set to `stackPreempt` (a magic value) to force a `morestack` call even when there's plenty of stack — this is the older cooperative preemption mechanism, still used as a backup.

## Standard Library Hooks

- `runtime.GOMAXPROCS(n int) int` — get/set P count.
- `runtime.NumCPU() int` — host (or cgroup-aware in 1.25+) CPU count.
- `runtime.NumGoroutine() int` — current live goroutines.
- `runtime.Gosched()` — voluntary yield. Rarely needed since async preemption.
- `runtime.LockOSThread()` / `UnlockOSThread()` — pin the calling goroutine to its OS thread (for OpenGL, JNI, signal handlers).
- `runtime.Stack(buf []byte, all bool) int` — dump goroutine stacks.
- `runtime.SetMaxThreads(n int) int` — change OS thread cap.
- `runtime/trace` — execution tracer; see `09-tooling/13-go-tool-trace.md`.
- `runtime/pprof` — goroutine profiles, mutex profiles, block profiles.

## Real-World Patterns

### 1. Pin to OS thread for OpenGL / FFI

```go
package main

import "runtime"

func init() {
	runtime.LockOSThread() // OpenGL must be called from the OS thread that owns the context
}

func main() {
	// ... renderer ...
}
```

`LockOSThread` ensures every call in this goroutine runs on the same OS thread for the goroutine's lifetime. The thread is also marked: when the goroutine exits, the thread is terminated (since 1.10), avoiding leaks.

### 2. Honor container CPU limits

```go
package main

import "runtime"

func main() {
	// Go 1.25+: nothing to do — runtime is cgroup-aware.
	// Pre-1.25: use go.uber.org/automaxprocs
	_ = runtime.GOMAXPROCS(0)
}
```

### 3. Stack dump on hang

```go
import (
	"os"
	"os/signal"
	"runtime"
	"syscall"
)

func init() {
	go func() {
		ch := make(chan os.Signal, 1)
		signal.Notify(ch, syscall.SIGUSR1)
		for range ch {
			buf := make([]byte, 1<<20)
			n := runtime.Stack(buf, true)
			os.Stderr.Write(buf[:n])
		}
	}()
}
```

`kill -USR1 <pid>` dumps every goroutine's stack. Indispensable for diagnosing production hangs.

### 4. Yield for fairness in a CPU-bound loop (rarely needed)

```go
for i := 0; i < 1_000_000_000; i++ {
	work(i)
	if i % 10000 == 0 { runtime.Gosched() } // optional — async preemption handles it
}
```

Pre-1.14 this was the canonical way to let other Gs run inside a tight loop. Post-1.14 async preemption makes it almost always unnecessary.

### 5. Profile goroutine scheduling

```
go test -bench=. -trace=trace.out
go tool trace trace.out
```

The trace UI shows you which G is on which P at each instant, lifetime of each G, and where time goes. Essential for diagnosing tail latency.

## Anti-Patterns & Gotchas

**Setting `GOMAXPROCS` way higher than CPU cores.** No benefit; you just oversubscribe and waste cache. The default is correct.

**Setting `GOMAXPROCS=1` to "remove concurrency bugs".** Removes parallelism, not concurrency. Goroutines still interleave on a single P; race conditions still happen.

**Calling `runtime.Gosched()` "to be fair".** Almost never needed since 1.14. Overuse adds scheduler overhead with no gain.

**Spawning millions of goroutines all at once.** Overflows local queues into the global queue, which is locked and slow. Spread the spawn rate, or use a worker pool.

**`runtime.LockOSThread` and forgetting to `UnlockOSThread`.** If you `Lock` and the goroutine exits without unlocking, the OS thread is destroyed (since 1.10). If you do unlock before exit, the thread returns to the pool. Match them.

**Believing `NumGoroutine()` is exact.** It's a snapshot under contention; treat it as an estimate.

**Long-running CGo calls block a P.** A Cgo call holds the M (and may hold the P if the call is short). Long calls effectively reduce parallelism. Use it sparingly.

**Spinning in a `for {}` loop expecting fairness from `runtime.Gosched`.** It worked pre-1.14; today async preemption is more reliable.

**Trying to manage the scheduler from user code.** You can't bind a G to a P, you can't pin a G to a CPU, you can't priority-boost. Don't try.

## Performance Notes

- Schedule cost (M picks up next G from its local queue): ~50 ns.
- Work-steal cost: ~µs, dominated by atomic ops.
- Async preemption signal: ~1–2 µs latency from sysmon decision to G yielding.
- Syscall enter/exit handoff: ~µs because of the P swap.
- Netpoller wakeup: ~10–100 µs depending on OS and load.
- Goroutine spawn (newproc): ~1 µs.
- Stack growth (doublings): proportional to stack size, can be ms for huge stacks.
- Global queue contention: noticeable above ~1M goroutines/sec spawn rate.

## How Big Companies Use It

- **The Go team** uses Kubernetes, Prometheus, and Bazel as primary scheduler regressions tests — significant scheduler changes are profiled against these.
- **Uber's `automaxprocs`** library (https://github.com/uber-go/automaxprocs) introduced cgroup-aware GOMAXPROCS years before the runtime made it default — Go 1.25 closes that gap.
- **Cloudflare** has documented scheduler hot spots in their edge: https://blog.cloudflare.com/scaling-go-applications/.
- **Discord's GC blog** is partly a scheduler post: high G counts amplified GC pause times. The fix involved both GC tuning and reducing goroutine churn.
- **Kubernetes apiserver** profiles its scheduler frequently; the 1.14 async preemption was significant for them.
- **Tailscale** documents `runtime.LockOSThread` use for their kernel-side WireGuard glue.

## Source Code References

Pinned to `go1.26`.

- The scheduler core: [`src/runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go). Read `schedule`, `findrunnable`, `runqget`, `runqsteal`.
- G, M, P struct definitions: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/master/src/runtime/runtime2.go).
- Asynchronous preemption: [`src/runtime/preempt.go`](https://github.com/golang/go/blob/master/src/runtime/preempt.go), [`src/runtime/signal_unix.go`](https://github.com/golang/go/blob/master/src/runtime/signal_unix.go) (search `sigPreempt`).
- Netpoller: [`src/runtime/netpoll.go`](https://github.com/golang/go/blob/master/src/runtime/netpoll.go) and per-OS implementations (`netpoll_epoll.go`, `netpoll_kqueue.go`, `netpoll_windows.go`).
- Sysmon: [`src/runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go), function `sysmon`.
- GOMAXPROCS container-awareness (1.25+): [`src/runtime/cgroup_linux.go`](https://github.com/golang/go/blob/master/src/runtime/cgroup_linux.go).

## Further Reading

- Dmitry Vyukov, "Scalable Go Scheduler Design Doc" (2012, still the canonical source): https://golang.org/s/go11sched
- "Work-stealing scheduler" (Vyukov, 2013): https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/
- Austin Clements, "Non-cooperative goroutine preemption" (proposal #24543): https://go.dev/issue/24543
- Russ Cox, "Goroutines, threads, and stacks": https://research.swtch.com/gostack
- Russ Cox, "Scheduling multithreaded computations by work stealing" (Blumofe-Leiserson summary)
- Madhav Jivrajani, "Go scheduler internals" (GopherCon EU 2022): https://www.youtube.com/watch?v=YHRO5WQGh0k
- Kavya Joshi, "The Scheduler Saga" (GopherCon 2018): https://www.youtube.com/watch?v=YHRO5WQGh0k
- Ardanlabs' deep-dive scheduler series: https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html

## Exercises / Self-Check

1. With `GOMAXPROCS=2`, you spawn 10 goroutines that each do `time.Sleep(1*time.Second)`. How many OS threads exist for most of that second? Why?
2. Sketch the path of a blocking `net.Read` from the goroutine's perspective: which queues, which threads, where it parks, where it wakes.
3. Why does sysmon need to send a signal for preemption instead of relying on the goroutine to check a flag?
4. With Go 1.25, what happens if a container has CPU limit `1.5` (i.e., 150ms per 100ms)? What's `GOMAXPROCS`?
5. A program runs `go x()` 1,000,000 times in a tight loop. Where does the contention show up in the scheduler? How would you redesign to avoid it?
