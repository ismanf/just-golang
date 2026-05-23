# Scheduler Deep Dive — G-M-P, netpoller, sysmon, async preemption

## TL;DR

The Go scheduler is a **work-stealing, multi-level-feedback scheduler** organized as **G-M-P**: any number of goroutines (`G`), some number of OS threads (`M`), exactly `GOMAXPROCS` logical processors (`P`). Each P has a 256-slot local run queue plus a `runnext` slot for fast hand-offs; an overflow **global queue** plus the **netpoller** wake list handle the rest. **Sysmon** (a single threadless M) does background work: preempt long-running Gs via SIGURG, hand off Ps stuck in syscalls, poll the network. **Async preemption** (since 1.14) means even a `for {}` yields. **Container-aware GOMAXPROCS** (since 1.25) reads cgroup CPU quotas. The single biggest gotcha: a blocking syscall **does not occupy a P** — only an M; thousands of socket-blocked Gs cost zero Ps.

This page goes one level deeper than `07-concurrency/11-scheduler-internals.md`. Read that first.

## Mental Model

```
        Sysmon (Mₛ, no P) ──── SIGURG ───┐
        ┌──────────────────────┐         │
        │ - preemption polls   │         │
        │ - syscall handoff    │         │
        │ - netpoll(timeout=0) │         │
        │ - GC pacing          │         │
        └──────────────────────┘         │
                                         ▼
        ┌────────────────────────────────────────────────────────────┐
        │  Ps (logical processors, GOMAXPROCS count)                 │
        │                                                             │
        │  P0          P1          P2          P3                    │
        │  ┌──┐        ┌──┐        ┌──┐        ┌──┐                  │
        │  │M0│        │M1│        │M2│        │M3│                  │
        │  └─┬┘        └─┬┘        └─┬┘        └─┬┘                  │
        │   ▼            ▼            ▼            ▼                 │
        │  runq[256]   runq[256]   runq[256]   runq[256]             │
        │  runnext     runnext     runnext     runnext               │
        │  mcache      mcache      mcache      mcache                │
        └────────────────────────────────────────────────────────────┘

        Global run queue (lock-protected, overflow)
        Netpoller ready list (woken by epoll/kqueue/IOCP)
        timers heap (per-P since 1.14)
        gFree per-P pool (recycled `g` structs)

        Other Ms:
            Ms in syscall (M without a P; M holds the goroutine)
            Ms idle (parked on a futex)
```

The runtime's job is to keep the Ps fed. The scheduling loop on each M does this; sysmon does it for stragglers.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
	fmt.Println("NumCPU:", runtime.NumCPU())
	// In Go 1.25+, GOMAXPROCS defaults to min(NumCPU, cgroup CPU limit).
}
```

Override:

```
GOMAXPROCS=4   # explicit
GOMAXPROCS=0   # use legacy default (NumCPU, ignore cgroup)
```

`runtime.GOMAXPROCS(n)` at runtime resizes the P count; live Ms migrate.

## Deep Dive

### The scheduling loop

Every M runs `schedule()` after finishing a G. The full algorithm, paraphrased from `src/runtime/proc.go`:

```
schedule():
  1. If sysmon set p.runSafePointFn, run it (e.g., GC sync point).
  2. If GC needs assist work, do a chunk.
  3. If trace stop-the-world is pending, ack and park.
  4. Every schedtick % 61 == 0, try the global queue first (fairness).
  5. Try p.runnext slot.
  6. Try the P's local runq.
  7. findrunnable():
       a. global queue (if any)
       b. netpoll (non-blocking)
       c. work-steal from a random P (take half)
       d. (rare) if nothing found: park M on noteclear/notesleep.
  8. Execute the chosen G: gogo() — load registers, jump to PC.
```

Step 4 is the **61-tick fairness check**: without it, a P that always finds work in its local queue might never check globals. 61 is chosen because it's prime and roughly the number of `schedule()` calls per CPU-millisecond at typical workloads.

### `runnext` — the LIFO hand-off slot

```go
// runtime/runtime2.go (excerpt)
type p struct {
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    runnext  guintptr // single-slot LIFO
    ...
}
```

When a goroutine A spawns B and immediately blocks (`go b(); <-ch`), the runtime wants the next-running G to be B (data-flow locality). It puts B in `runnext`. The scheduler always checks `runnext` first. If `runnext` is full, the previous occupant is moved to the tail of the local queue.

`runnext` exists *only* to optimize this pattern. Misuse: a hot loop spawning short-lived Gs will starve other queued Gs because each spawn keeps refilling `runnext`. The runtime has a heuristic (`runnext` doesn't get stolen via work-stealing immediately) but the asymmetry can show up in benchmarks.

### Work stealing

```go
// runtime/proc.go (paraphrased)
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {
    // Take half (rounded up) of victim's local queue.
    // Atomic CAS on the victim's head; copy entries to our queue.
    // Optionally take runnext if `stealRunNextG`.
}
```

Stealing is *random*: pick a P uniformly. After 4 random attempts, the M parks. Randomization avoids hot-spotting any one victim. Stealing half (not all) maintains the local queue as a stable working set.

`runnext` is normally **not** stolen until the victim has been running it for >3 µs — otherwise heavy spawn-then-block workloads thrash.

### The global queue

A doubly-linked list under `sched.lock`. Used when:
- A goroutine is created and the spawner's local queue is full.
- The runtime starts goroutines on behalf of background tasks.
- Stolen goroutines spill over.

Local queues are 256 entries — overflow goes to the global queue (in batches of 128). Most workloads barely touch the global queue.

### Netpoller

The netpoller is **non-blocking I/O**. Every network FD is in epoll (Linux) / kqueue (BSD/macOS) / IOCP (Windows) / `event ports` (Solaris). When a G calls `Read` on a socket and there's no data:

1. The runtime registers the FD (if not already) for "ready" events.
2. The G goes to `_Gwaiting` with `waitreason=waitReasonIOWait`.
3. The M is freed (or returns to scheduling).

When the network thread (any M can do it) calls `netpoll(timeout)`:

```go
// runtime/netpoll.go (excerpt)
func netpoll(delay int64) (gList, int32) {
    // epoll_wait / kevent / ...
    // For each ready FD: call goready on the parked G.
}
```

Two variants are called:
- **`netpoll(0)` — non-blocking poll** — invoked from sysmon and from `findrunnable` when looking for work.
- **`netpoll(blocking)` — long park** — invoked by an idle M as the last resort before going to sleep.

The "network thread" isn't a dedicated thread; any M can poll. This keeps tail latency low.

### Timers

Pre-1.14, timers were a single global heap with global lock. Severe contention with many timers.

Since 1.14, **timers are per-P**: each P has its own four-heap (one for active, one for adjusting, etc., compressed in 1.23). The scheduler checks the P's timer heap inline:

```
findrunnable():
    checkTimers(pp, now)  // fire any expired
    ...
```

Re-spec'd in [issue #6239](https://go.dev/issue/6239). Modern Go programs with millions of timers (HTTP timeouts, gRPC deadlines) scale because each P handles its own.

### Sysmon

Sysmon is one thread, no P, started at runtime init:

```go
// runtime/proc.go (excerpt; conceptual)
func sysmon() {
    for {
        usleep(delay)               // exponential backoff
        retake(now)                 // preempt or syscall-handoff
        if netpollinited() {
            netpollBreak() / netpoll(0)
        }
        forcegcperiod := 2*60*1e9   // 2 minutes
        if now > lastgc + forcegcperiod {
            gcStart(...)             // forced periodic GC
        }
        scvg.signal()                // wake scavenger
    }
}
```

`retake`:
- For each P, check if its current G has been running >10 ms. If so, send SIGURG (async preempt).
- For each P "in syscall" for >10 µs without progress, hand the P to a fresh M so the syscall's M can carry on detached.

The exponential backoff: sysmon sleeps 20 µs initially, doubling up to 10 ms when idle. Active runtime → 20 µs polls; idle runtime → 10 ms polls (no wasted CPU).

### Asynchronous preemption — implementation

Three pieces:

**1. Per-arch `asyncPreempt` shim** in `src/runtime/asm_*.s`:

```asm
TEXT ·asyncPreempt(SB),NOSPLIT|NOFRAME,$0-0
    // save user registers
    // call asyncPreempt2
    // restore and return
```

**2. Signal handler** `sigPreempt` in `signal_unix.go`:

```go
func sigPreempt(...) {
    // is GC-safe-point active?
    // is current PC marked safe-to-preempt?
    if !canAsyncPreempt(...) {
        return
    }
    // redirect PC to runtime.asyncPreempt
}
```

**3. Compiler-emitted safe-point tables** (in func data) — the linker stitches them into the binary; the runtime consults them in `canAsyncPreempt`.

If the signal hits an unsafe PC (register-stale, mid-writebarrier, in cgo), the signal handler returns without rewriting PC. Sysmon will try again later.

This is why some functions have `//go:nosplit`: they have no preemption safe-point and the prologue stack-check is skipped. They must be short.

### Container awareness (1.25+)

Pre-1.25: `runtime.NumCPU()` and the default GOMAXPROCS came from `sched_getaffinity` on Linux — the host's CPU count, regardless of cgroup quota. In a 2-core container on a 64-core host, the runtime saw 64. Result: heavy contention, garbage scheduling, more goroutines per P than P slots.

The community fix was Uber's `automaxprocs` library: read `/sys/fs/cgroup/cpu.cfs_quota_us` divided by `cpu.cfs_period_us` (v1) or `cpu.max` (v2), use as GOMAXPROCS.

Go 1.25 imports this logic into the runtime. The default GOMAXPROCS is now:

```
min(NumCPU(), ceil(cgroup_quota / cgroup_period))
```

You can opt out by setting `GOMAXPROCS=0` (returns to pre-1.25 default) or explicit `GOMAXPROCS=N`.

### Sched tick and randomization

`p.schedtick` counts each call to `schedule()` on that P. The 61-tick fairness check and various stat samplers use it. `p.fastrand` is a per-P PRNG used for work-stealing victim choice and goroutine sample selection in profiles — keeps the choices uncorrelated across Ps.

### M lifecycle

```
mstart (kernel-created thread) →
  schedule() → execute G's →
  if no work, stopm() → parked on note →
  woken by startm() / handoff →
  ... loop
```

Ms are *not* one-per-P. There are at least `GOMAXPROCS` Ms; more are spawned as Gs enter syscalls. Total cap is `runtime.SetMaxThreads(10000)` by default. Hitting the limit panics.

### Special goroutines and Ps

- **`g0` (per M)**: system goroutine, runs scheduler code with the M's actual OS stack. Switches to user G's stack to run user code.
- **`gsignal` (per M)**: tiny goroutine on which signal handlers run.
- **`Mₛ` (sysmon)**: holds no P. Single instance.
- **Template thread**: pre-allocated M for handling new OS threads created by `clone(2)` from cgo.

## Standard Library Hooks

- `runtime.GOMAXPROCS(n int) int`.
- `runtime.NumCPU() int` (cgroup-aware in 1.25+).
- `runtime.NumGoroutine() int`.
- `runtime.Gosched()` — voluntary yield (rarely useful since 1.14).
- `runtime.LockOSThread()`/`UnlockOSThread()`.
- `runtime.SetMaxThreads(n int) int`.
- `runtime.GoroutineProfile(p []runtime.StackRecord) (int, bool)`.
- `runtime/trace.Start(w io.Writer) error` — execution tracer; see `09-tooling/13-go-tool-trace.md`.
- `runtime/metrics` series:
  - `/sched/goroutines:goroutines` — total goroutines.
  - `/sched/latencies:seconds` — runnable-to-running latency histogram.
  - `/sched/gomaxprocs:threads` — current P count.
- `GODEBUG=schedtrace=1000` — every 1 s, scheduler summary line.
- `GODEBUG=scheddetail=1` — extended `schedtrace` showing every G's state.

## Real-World Patterns

### 1. Diagnose scheduler latency

```
$ GODEBUG=schedtrace=100,scheddetail=1 ./bin
SCHED 100ms: gomaxprocs=4 idleprocs=0 threads=12 spinningthreads=0 needspinning=0 idlethreads=2 runqueue=0 ...
  P0: status=1 schedtick=1234 syscalltick=12 m=5 runqsize=3 gfreecnt=15
  P1: ...
  G1: status=4(IO wait) m=-1 lockedm=-1
  ...
```

Useful fields:
- `gomaxprocs / idleprocs`: are Ps idle while Gs queue?
- `runqueue`: global queue depth (should be 0 most of the time).
- Per-P `runqsize`: which Ps are loaded.
- Per-G `status`: where Gs are stuck.

### 2. Capture an execution trace

```go
package main

import (
	"os"
	"runtime/trace"
)

func main() {
	f, _ := os.Create("trace.out")
	defer f.Close()
	trace.Start(f)
	defer trace.Stop()
	// ... workload ...
}
```

Then: `go tool trace trace.out`. The UI shows each P's timeline, every G transition, GC activity, syscall blockers. Read more at `09-tooling/13-go-tool-trace.md`.

### 3. Make a long-running cgo call cooperative

```go
// In a C function called from cgo:
// for (long i = 0; i < N; i++) {
//     work();
//     if (i % 100000 == 0) sched_yield();   // explicit OS yield
// }
```

A long cgo call holds an M without async-preempt access (signals are tricky cross-language). Either chunk the work in Go and call cgo many times, or yield from C.

### 4. Reserve a P for a critical loop

There's no direct API. Closest pattern:

```go
package main

import (
	"runtime"
)

func criticalLoop() {
	runtime.LockOSThread()
	// This G never moves; the M is dedicated to it.
	// Other Gs use the remaining Ps.
	for { tick() }
}

func tick() {}
```

Note: the *P* isn't dedicated; only the *M*. If the OS schedules the M elsewhere, your loop pauses. For real isolation use `taskset`/`cpuset` at the OS level.

### 5. Tune for high-fan-out workloads

```go
package main

import "runtime"

func init() {
	// In a container with 8 CPUs and fan-out workload:
	// Default GOMAXPROCS=8 is correct.
	// Raising it doesn't help (would oversubscribe).
	// Lowering it (e.g., 4) leaves capacity for sidecars but increases tail latency.
	_ = runtime.GOMAXPROCS(0)
}
```

The right value is *almost always* the default. Don't second-guess without measurement.

## Anti-Patterns & Gotchas

**Raising GOMAXPROCS to make the program faster.** Cores don't multiply; you just oversubscribe. CPU-bound code with `GOMAXPROCS > NumCPU` thrashes context switches.

**Calling `runtime.Gosched()` "to help" the scheduler.** Adds scheduler cost; async preemption handles it.

**Storing per-P state in user code.** No public API to identify the current P. The closest hack is reading `runtime` internals via `//go:linkname`, which breaks across versions. Use per-CPU patterns (`sync.Pool` is per-P, indirectly) instead.

**Long-running cgo calls.** They hold an M and (initially) a P. Sysmon eventually hands the P off after 10 µs, but startup latency is added. Either chunk the call or accept the cost.

**Burst-spawning a million goroutines.** Overflows local queues → global queue → contention. Use a semaphore (`make(chan struct{}, N)`) or `errgroup.WithLimit`.

**Mixing `LockOSThread` with `GOMAXPROCS=1`.** The locked M owns the only P; no other Gs run. Net effect: serialized execution.

**Long-blocking time.Sleep in a syscall-shaped pattern.** `time.Sleep` uses the per-P timer heap and parks the G via netpoller's eventfd — no M is wasted. Don't worry about that one.

**Trying to schedule with priorities.** No priority concept. All Gs are equal; ordering is FIFO within a queue and pseudorandom across Ps.

**Setting GOMAXPROCS=1 to "remove concurrency bugs".** Removes *parallelism*, not concurrency. Goroutines still interleave on the single P; race conditions still happen.

## Performance Notes

- Scheduler decision (find next G from runnext or local queue): ~50 ns.
- Work-steal attempt: ~µs (CAS-dominated).
- Park/unpark M: ~5–10 µs (futex round-trip).
- Sysmon polling interval: 20 µs (busy) to 10 ms (idle).
- Netpoll round-trip (epoll_wait + goready): ~10–100 µs.
- Async preempt signal latency: ~10 µs from sysmon decision to G yield.
- Per-P timer fire: ~µs.
- GC safepoint sync (STW start): bounded by signal latency × num Ps; typically <100 µs at GOMAXPROCS=8.
- Global queue acquire (lock contention): ~µs at low load, scales to ~ms under millions of ops/sec.

## How Big Companies Use It

- **The Go team** uses Kubernetes apiserver, Prometheus, and Bazel as scheduler regression suites: https://golang.org/s/go11sched (historical context).
- **Uber `automaxprocs`** introduced cgroup-aware GOMAXPROCS years before 1.25: https://github.com/uber-go/automaxprocs.
- **Cloudflare** documented scheduler hot-spots in their edge: https://blog.cloudflare.com/scaling-go-applications/.
- **Discord** post-mortems describe scheduler latency contributing to tail latency; their state-cache GC issues had a partial scheduler explanation: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **Kubernetes** profiles scheduler hot paths regularly; the 1.14 async preempt landed a measurable apiserver latency win.
- **Tailscale** uses `LockOSThread` for `magicsock` and documents the trade-offs: https://github.com/tailscale/tailscale.
- **YouTube ad bidder** (Google internal, Austin Clements GopherCon 2018) profiled the scheduler with millions of timers; informed the 1.14 per-P timers redesign.

## Source Code References

Pinned to `go1.26`.

- Scheduling loop: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go) — `schedule`, `findrunnable`, `execute`.
- Work-stealing: same file, `runqget`, `runqgrab`, `runqsteal`.
- Sysmon: same file, function `sysmon`.
- Async preempt: [`src/runtime/preempt.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/preempt.go), per-arch `asyncpreempt_*.s`.
- Signal preempt: [`src/runtime/signal_unix.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/signal_unix.go).
- Netpoller: [`src/runtime/netpoll.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/netpoll.go) and per-OS impls (`netpoll_epoll.go`, `netpoll_kqueue.go`, `netpoll_windows.go`).
- Timers: [`src/runtime/time.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/time.go).
- Cgroup awareness (1.25+): [`src/runtime/cgroup_linux.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/cgroup_linux.go).
- GMP struct definitions: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/runtime2.go).
- Schedtrace formatter: [`src/runtime/proc.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/proc.go) — `schedtrace`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Dmitry Vyukov, "Scalable Go Scheduler Design": https://golang.org/s/go11sched.
- Dmitry Vyukov, "Go Preemptive Scheduler" follow-up: https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/.
- Austin Clements, "Non-cooperative goroutine preemption" (proposal #24543): https://go.dev/issue/24543.
- "Scheduler in Go" (Madhav Jivrajani, GopherCon EU 2022): https://www.youtube.com/watch?v=_clOpANrGgw.
- Kavya Joshi, "The Scheduler Saga" (GopherCon 2018): https://www.youtube.com/watch?v=YHRO5WQGh0k.
- "Per-P timers in Go 1.14" (Ian Lance Taylor): https://go.dev/doc/go1.14#runtime.
- Ardan Labs scheduler series: https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html.
- "Go's work-stealing scheduler" — Madhav Jivrajani blog: https://rakyll.org/scheduler/.

## Exercises / Self-Check

1. With `GOMAXPROCS=4`, you start 1M goroutines all doing `time.Sleep(1*time.Second)`. How many Ps are busy? How many Ms exist?
2. Sketch the path of a `net.Conn.Read` call from G's perspective: queues, threads, parking and waking. Where does the M block, if at all?
3. Why is the 61-tick global-queue check needed? What workload exposes the bug if you removed it?
4. Sysmon decides to preempt a G via SIGURG. Trace the events through the signal handler, register save, and PC redirect. What can cause the preemption to be skipped?
5. With `GOMAXPROCS=8` in a 2-core cgroup container (Go 1.25+), what's the actual GOMAXPROCS? What if you opt out?
