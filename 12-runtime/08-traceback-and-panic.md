# Traceback and Panic — How the Runtime Unwinds

## TL;DR

A **traceback** is the runtime walking the stack frame-by-frame, mapping each return PC to a source location via **PCDATA** tables emitted by the compiler. It's used by panics, `runtime.Stack`, the profiler, and the goroutine dump on SIGQUIT. **Panic** flips the goroutine into an unwinding state that calls each deferred function in LIFO order; if no `recover` swallows it, the runtime prints a panic message + traceback for every goroutine and `exit(2)`s. The single biggest gotcha: `recover()` only works in a function **directly called by `defer`** — not in a helper called from the deferred function, and not after a goroutine started inside the deferred function.

## Mental Model

```
Stack growing down ↓

  +────────────────────+
  │  panic frame       │ ← runtime.gopanic
  +────────────────────+
  │  user frame B      │ ← was running when panic happened
  +────────────────────+
  │  user frame A      │ ← called B
  +────────────────────+   ↑
  │  main goroutine    │   │ defers walk this direction (oldest last)
  +────────────────────+   │ until one of them calls recover()
                           │ or the stack is exhausted

Each frame stores:
  - return PC (where the caller will resume)
  - frame size (compiler-emitted, in PCDATA)
  - argument layout (PCDATA)
  - stack-map bitmaps (which words are pointers)

Traceback uses these tables to walk frame N from frame N+1's
saved return PC.
```

The traceback machinery is the same whether you called `runtime.Stack`, hit a panic, sent SIGQUIT, or the GC needed to scan a stack: all roads lead through `gentraceback` in `runtime/traceback.go`.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	defer func() {
		if r := recover(); r != nil {
			buf := make([]byte, 1<<16)
			n := runtime.Stack(buf, false) // false = this goroutine only
			fmt.Printf("recovered: %v\n%s\n", r, buf[:n])
		}
	}()
	a()
}

func a() { b() }
func b() { c() }
func c() { panic("boom") }
```

Output (abbreviated):

```
recovered: boom
goroutine 1 [running]:
main.main.func1()
   /tmp/x.go:9 +0x65
panic({0x..., 0x...})
   runtime/panic.go:818 +0x...
main.c(...)
   /tmp/x.go:17
main.b(...)
   /tmp/x.go:16
main.a(...)
   /tmp/x.go:15
main.main()
   /tmp/x.go:11 +0x...
```

## Deep Dive

### `panic` lifecycle

```go
// runtime/panic.go (excerpt; BSD-3 © The Go Authors)
func gopanic(e any) {
    gp := getg()
    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    for {
        d := gp._defer
        if d == nil {
            break
        }
        // ... pop deferred call, invoke it ...
        if p.recovered {
            // recover() set this; resume normally
            gp._panic = p.link
            // jump to the deferred function's caller
            mcall(recovery)
        }
    }

    // No recover happened: fatal.
    fatalpanic(gp._panic)
}
```

Roughly:
1. Build a `_panic` record, push onto `g._panic` stack.
2. While the goroutine has deferred functions:
   a. Pop the topmost defer.
   b. Run its function.
   c. If the function called `recover`, mark `p.recovered`, jump back to the deferred function's frame's caller (via `recovery` assembly).
3. If we exhaust defers without recovery, the runtime walks every goroutine, prints tracebacks, and `exit(2)`.

The key trick: `recover` only sets a flag (`p.recovered = true` plus return-value plumbing). The actual stack unwind happens via `recovery`, which manipulates registers and SP to "return" from the function the deferred call was inside.

### Why `recover` must be a *direct* defer call

The flow above pops a defer, runs it, and only the popped defer's function can set `recovered`. If the defer calls a helper that calls `recover`, the helper sees `p.panicking` but cannot rewind to the deferred function's caller — by the time recovery wants to jump, the helper's frame is in the way.

```go
package main

import "fmt"

// WORKS: recover is in a function called *directly* by defer.
func ok() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("ok:", r)
		}
	}()
	panic("x")
}

// FAILS: recover is in a helper, not directly in the defer.
func helper() any { return recover() }
func bad() {
	defer helper()
	panic("y") // not recovered; helper's recover() returns nil
}

func main() {
	ok()
	defer func() { recover() }()
	bad()
}
```

The rule: `recover` returns a non-nil value *only when called directly from a deferred function*. Otherwise it returns `nil` even while a panic is in progress.

### Defers: open-coded vs heap

Pre-1.14 defers were always heap-allocated `_defer` records linked off `g._defer`. Each `defer` cost ~50 ns plus a heap allocation.

Since 1.14, defers in functions with ≤8 simple defers (no defer inside loop, no condition that varies the count) are **open-coded** by the compiler: a bitmask + inline calls. Zero-allocation, ~5 ns per defer.

You can check with `go build -gcflags=-d=defer`:

```
$ go build -gcflags=-d=defer pkg.go
./pkg.go:10:2: open-coded defer
./pkg.go:18:2: stack-allocated defer
./pkg.go:25:2: heap-allocated defer
```

Stack-allocated (`_defer` on the goroutine's stack) is the middle case — used when the defer count is fixed but >8 or the function is too complex for open-coding.

### `_defer` struct

```go
// runtime/runtime2.go (excerpt; BSD-3)
type _defer struct {
    started   bool
    heap      bool
    openDefer bool

    sp     uintptr  // sp at defer time
    pc     uintptr  // return pc after the deferred call
    fn     func()   // the deferred function

    _panic *_panic
    link   *_defer
    ...
}
```

`sp` and `pc` let the runtime restore the call site exactly when unwinding.

### `gentraceback` — the unifying walker

```go
// runtime/traceback.go (excerpt; conceptual)
func gentraceback(pc0, sp0 uintptr, lr0 uintptr, gp *g, ...) int {
    for f := findfunc(pc0); f.valid(); f = findfunc(callerPC) {
        // Decode f's pcdata to find frame size and locals.
        // Print or store frame info.
        // Advance pc, sp to caller frame.
    }
}
```

Inputs: starting PC and SP. Output: count of frames walked, optional callback per frame.

Used by:
- `runtime.Caller`, `runtime.Callers` — `runtime/extern.go`.
- `runtime.Stack` — for SIGQUIT or user dumps.
- `runtime/pprof` — for CPU and goroutine profiles.
- GC mark — to walk pointers in each frame.
- `panic` — to print frames during a fatal panic.

The walker uses **pcdata tables** (compiler-emitted, in `cmd/internal/obj/pcln.go`) to map a PC to:
- The function it belongs to (via `findfunc`).
- Frame size at that PC.
- File and line number.
- Live pointer bitmap (for GC).
- Inlining tree (for "in-function" stacks).

### Inlining and tracebacks

If `f` is inlined into `g`, the traceback should still show `f`. The compiler emits an **inlining tree** with each PC's "physical" frame plus "virtual" callers. The runtime walks both:

```
goroutine 1 [running]:
main.add(...)
    /tmp/x.go:5
main.sum(...)        ← inlined; shown but no physical frame
    /tmp/x.go:9
main.main()
    /tmp/x.go:12
```

Pre-1.12 inlining hid frames. Modern Go always shows them.

### Stack maps for GC

Each PC has a bitmap describing which stack-frame words are pointers. The GC walks every goroutine's stack and uses these maps to find roots. Without them, GC would have to be conservative (treat every word as a maybe-pointer), which would prevent compacting and prevent precise stack copying.

Emitted by `cmd/compile/internal/liveness/plive.go`, packed into the function's metadata.

### `runtime.Caller` and friends

```go
// runtime/extern.go (canonical signatures)
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
func Callers(skip int, pc []uintptr) int
func FuncForPC(pc uintptr) *Func
func (*Func) FileLine(pc uintptr) (file string, line int)
func (*Func) Name() string
```

Usage:

```go
package main

import (
	"fmt"
	"runtime"
)

func logCaller() {
	pcs := make([]uintptr, 8)
	n := runtime.Callers(1, pcs) // skip Callers itself
	frames := runtime.CallersFrames(pcs[:n])
	for {
		f, more := frames.Next()
		fmt.Printf("  %s\n    %s:%d\n", f.Function, f.File, f.Line)
		if !more {
			break
		}
	}
}

func main() { logCaller() }
```

`runtime.CallersFrames` is the modern API; it handles inlining correctly. The older `runtime.FuncForPC` doesn't.

### `panic` printing format

When a panic kills the program:

```
panic: <message>

goroutine 1 [running]:
<traceback>

goroutine 2 [chan receive]:
<traceback>

...
```

All goroutines are dumped by default (since 1.6). `GOTRACEBACK` controls verbosity:

```
GOTRACEBACK=none    # no stack
GOTRACEBACK=single  # only current goroutine
GOTRACEBACK=all     # all user goroutines (default)
GOTRACEBACK=system  # all + runtime frames
GOTRACEBACK=crash   # all + runtime + abort()→core dump
```

`GOTRACEBACK=crash` is the right setting in production: you get a core dump for post-mortem with delve.

### SIGQUIT

`Ctrl-\` (SIGQUIT) on Unix prints a goroutine dump and exits. Useful for stuck programs:

```
^\SIGQUIT: quit
PC=0x4...
goroutine 1 [chan receive]:
main.main()
   /tmp/x.go:10
```

The handler is installed by default; you can intercept it with `signal.Notify` to dump without exiting.

### Fatal errors vs panics

Some runtime conditions produce **fatal errors** (`fatal error: ...`) which can't be recovered:

- Concurrent map write/read (`fatal error: concurrent map writes`).
- Stack overflow (`runtime: goroutine stack exceeds 1000000000-byte limit`).
- Out-of-memory.
- Some atomic misuse.

These call `throw` in the runtime; the goroutine doesn't get a chance to defer/recover. Print + dump + exit.

### `recover` return value

```go
package main

import "fmt"

func main() {
	defer func() {
		switch r := recover().(type) {
		case nil:
			// no panic
		case error:
			fmt.Println("error panic:", r)
		case string:
			fmt.Println("string panic:", r)
		default:
			fmt.Println("other panic:", r)
		}
	}()
	panic(fmt.Errorf("oops"))
}
```

Any value can be passed to `panic`. Type-assert in `recover` to decide.

### `runtime/debug.Stack` and `runtime/debug.PrintStack`

`runtime.Stack(buf, false)` is low-level. `runtime/debug.Stack()` returns the formatted dump as a `[]byte`; `runtime/debug.PrintStack()` writes it to stderr. Slightly higher-level convenience.

## Standard Library Hooks

- `panic(v any)`, `recover() any`, `defer ...`.
- `runtime.Stack(buf []byte, all bool) int`.
- `runtime.Caller(skip int)`.
- `runtime.Callers(skip int, pc []uintptr) int`.
- `runtime.CallersFrames(pcs []uintptr) *Frames`.
- `runtime.FuncForPC(pc uintptr) *Func`.
- `runtime.SetCgoTraceback(...)` — register cgo unwinder.
- `runtime/debug.Stack()`, `runtime/debug.PrintStack()`.
- `runtime/debug.SetPanicOnFault(enabled bool) bool` — turn faults into panics.
- `GOTRACEBACK` env var.

## Real-World Patterns

### 1. Panic-safe goroutine spawner

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func Go(fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Fprintf(stderr, "panic in goroutine: %v\n%s\n", r, debug.Stack())
				// metric.Increment("panic")
			}
		}()
		fn()
	}()
}

var stderr writer

type writer interface{ Write(p []byte) (int, error) }
```

Wraps every goroutine spawn with a panic catcher. Unrecovered panics in goroutines kill the process; this turns them into logged errors.

### 2. Stack dump on signal (production)

```go
package main

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

func main() {}
```

`kill -USR1 <pid>` dumps every goroutine without killing the process. Indispensable for diagnosing stuck production servers.

### 3. Wrap an error with stack

```go
package errs

import (
	"fmt"
	"runtime"
)

type Err struct {
	msg  string
	stack [16]uintptr
	n     int
}

func New(msg string) *Err {
	e := &Err{msg: msg}
	e.n = runtime.Callers(2, e.stack[:])
	return e
}

func (e *Err) Error() string { return e.msg }

func (e *Err) Trace() string {
	frames := runtime.CallersFrames(e.stack[:e.n])
	out := ""
	for {
		f, more := frames.Next()
		out += fmt.Sprintf("  %s\n    %s:%d\n", f.Function, f.File, f.Line)
		if !more { break }
	}
	return out
}
```

Capture stack at error-creation, format on demand. Skip 2 to omit `Callers` and `New` themselves. (For production use prefer `pkg/errors` or stdlib `errors.Join` with care.)

### 4. Convert SIGSEGV to recoverable panic

```go
package main

import "runtime/debug"

func init() {
	debug.SetPanicOnFault(true)
	// Now invalid memory access becomes a panic instead of a runtime crash.
}
```

Useful when reading mmap'd files where the OS may unmap behind your back — turn SIGBUS into a recoverable panic.

### 5. Verbose tracebacks in production

```
GOTRACEBACK=crash ./app
```

On panic, dump every goroutine + runtime frames, then abort to drop a core. Pair with `delve core <bin> <core>` for post-mortem.

## Anti-Patterns & Gotchas

**Using `panic` for normal control flow.** Panics are expensive (~µs each) and confuse readers. Use errors. Exception: very deep recursion where unwinding is the cheapest way to bail (e.g., parser recovery).

**`recover` in a helper called from defer.** Returns nil; doesn't recover. Must be in the deferred function itself.

**Goroutine started inside a `defer`.** That goroutine's panic doesn't propagate. Catch within the goroutine.

**Re-panic from a recovered function.** `recover()` then `panic(r)` works, but the second panic resets defer iteration. Usually you want to log and return instead.

**Letting a goroutine panic in production without catching it.** Kills the process. Always wrap goroutine entry points.

**`runtime.Stack(buf, true)` at high frequency.** STW-equivalent for stack walking; with 1M goroutines this is many ms. Use `runtime/pprof` goroutine profile for sampling.

**Catching `runtime.Error` from a fatal-error path.** You can't. `throw`-based errors bypass defer.

**Trusting goroutine numbers across versions.** They're informational; runtime renumbers freely.

**Relying on `recover()` in defers when the goroutine is in a syscall.** The defer runs after the syscall returns, but if the process is being killed by signal there's no chance.

**Using stack traces as error identity.** Two panics from different sites have different stacks; an `errors.Is` comparison should use sentinel/typed errors, not stack-based identity.

## Performance Notes

- Open-coded defer: ~5 ns per call.
- Stack-allocated `_defer`: ~20 ns, no allocation.
- Heap-allocated `_defer`: ~100 ns + heap alloc.
- Panic + recover (no real work): ~µs.
- `runtime.Caller(0)`: ~µs.
- `runtime.Callers` for N frames: ~100 ns per frame.
- `runtime.Stack` for a single G: ~10–100 µs depending on stack depth.
- `runtime.Stack(buf, all=true)` with 10k Gs: ~tens of ms.
- Goroutine profile (`pprof`): samples ~1/sec; constant overhead.

Modern code avoids the heap-defer path entirely — most functions fit open-coded. If `pprof` shows `runtime.deferproc` as a hot frame, the function has too many or conditional defers.

## How Big Companies Use It

- **gRPC-Go** wraps every server handler invocation in a recover that logs + returns an INTERNAL error: https://github.com/grpc/grpc-go.
- **Kubernetes apiserver** has `utilruntime.HandleCrash` that catches panics in goroutines fleet-wide: https://github.com/kubernetes/apimachinery/blob/master/pkg/util/runtime/runtime.go.
- **CockroachDB** uses `runtime.Stack` snapshots in their `crdb_internal.node_stack_traces` virtual table for online diagnostics: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses `runtime/debug.SetPanicOnFault` for mmap'd routing tables; documented at https://github.com/tailscale/tailscale.
- **Uber's Zap logger** has a built-in stack-capture mode using `runtime.Callers` and `CallersFrames`: https://github.com/uber-go/zap.
- **Sentry/Bugsnag Go SDKs** wrap goroutines with their own panic catcher that captures a `runtime.Stack` dump for reporting.

## Source Code References

Pinned to `go1.26`.

- Panic / recover / defer: [`src/runtime/panic.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/panic.go).
- Traceback walker: [`src/runtime/traceback.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/traceback.go) — `gentraceback`.
- `_defer` struct: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/runtime2.go) — search `type _defer`.
- Open-coded defer (compiler side): [`src/cmd/compile/internal/ssa/_gen/generic.rules`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/_gen/generic.rules) — search `OpenCodedDefer`.
- PCDATA + line tables: [`src/cmd/internal/obj/pcln.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/internal/obj/pcln.go).
- `runtime.Caller(s)` and `CallersFrames`: [`src/runtime/extern.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/extern.go), [`src/runtime/symtab.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/symtab.go).
- `runtime.Stack`: [`src/runtime/mprof.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mprof.go).
- `SetPanicOnFault`: [`src/runtime/debug/garbage.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/debug/garbage.go).
- Cgo traceback: [`src/runtime/traceback.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/traceback.go) — search `SetCgoTraceback`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Defer, Panic, and Recover" (official Go blog): https://go.dev/blog/defer-panic-and-recover.
- Keith Randall, "Open-coded defers": https://go.googlesource.com/proposal/+/master/design/34481-opencoded-defers.md.
- Russ Cox, "Go's traceback": informal but excellent at https://research.swtch.com/.
- Ian Lance Taylor, "Defer optimizations" Go blog: https://go.dev/blog/defer-overhead.
- "Stack maps and GC" — Austin Clements talk, GopherCon 2018: https://www.youtube.com/watch?v=KFls7P_-_h4.
- Dmitry Vyukov, "Go traceback internals": https://golang.org/s/go-traceback.
- "How Go panic works under the hood" — Cybozu inside: https://blog.cybozu.io/.
- `runtime/debug` package docs: https://pkg.go.dev/runtime/debug.

## Exercises / Self-Check

1. Why does `recover()` need to be called *directly* from a deferred function? What machinery would have to change to lift that restriction?
2. Write a small program where `recover` is called from a helper invoked by defer. Predict the behavior, then run it.
3. Inspect `go build -gcflags=-d=defer` on a function with one `defer`, with three defers, with eight, with nine, and with a defer inside a loop. Which are open-coded?
4. A SIGSEGV happens in your program. With `GOTRACEBACK=crash`, what do you find on disk? With `debug.SetPanicOnFault(true)`?
5. How does `gentraceback` know the size of each stack frame? What compiler-emitted data is involved, and what changes for an inlined call?
