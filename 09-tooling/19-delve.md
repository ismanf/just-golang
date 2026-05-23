# `dlv` — Delve, the Go Debugger

## TL;DR

**Delve** (binary name `dlv`) is the Go debugger. Unlike GDB, it understands Go semantics natively: goroutines, channels, defers, interface types, `runtime.G` structures, package-private variables. The core workflows are: **`dlv debug`** (build and run with debug info), **`dlv test`** (debug a test binary), **`dlv attach <pid>`** (live process), **`dlv exec ./bin`** (already-built binary), and **`dlv core ./bin core.dump`** (post-mortem from a core file). The headless mode (`dlv exec --headless --listen=:2345 --api-version=2 ./bin`) is what IDE plugins (VS Code, GoLand, Neovim) connect to via JSON-RPC. Delve compiles your code with `-gcflags="all=-N -l"` (no optimization, no inlining) for accurate stepping; this slows execution but makes variables predictable.

## Mental Model

```
   ┌────────────────────────────────────────┐
   │  dlv  (the CLI / RPC server)           │
   ├────────────────────────────────────────┤
   │  ptrace (Linux) / Mach (macOS) / DbgHelp (Windows) │
   │  reads target process state directly   │
   ├────────────────────────────────────────┤
   │  DWARF parser + .gopclntab + runtime knowledge     │
   │  resolves PCs → file:line, type info, goroutines   │
   └────────────────────────────────────────┘
                  │
                  ▼
              ┌────────────┐
              │  Target    │  ── compiled with -N -l (no opt, no inline)
              │  Go binary │
              └────────────┘
```

`dlv` is a regular debugger that happens to know Go's runtime: it can list goroutines, inspect channel state, follow defers, decode interface values, etc.

## Syntax & Basic Usage

```bash
$ go install github.com/go-delve/delve/cmd/dlv@latest

$ dlv debug                          # build + run main in current dir
$ dlv debug ./cmd/app
$ dlv debug -- -flag1 -flag2 arg     # pass args to your program
$ dlv test                           # debug test in current dir
$ dlv test -- -test.run=TestFoo
$ dlv exec ./prebuilt-binary
$ dlv attach 12345                   # by pid
$ dlv core ./bin /tmp/core.dump

# Headless (for IDE)
$ dlv debug --headless --listen=:2345 --api-version=2 --accept-multiclient ./cmd/app
$ dlv connect localhost:2345

# Inside dlv prompt:
(dlv) break main.foo
(dlv) break /pkg/file.go:42
(dlv) continue
(dlv) next         # step over
(dlv) step         # step into
(dlv) stepout      # step out
(dlv) print user
(dlv) p user.Name
(dlv) locals
(dlv) args
(dlv) goroutines
(dlv) goroutine 12 bt
(dlv) condition 1 user.ID == 42       # conditional breakpoint
(dlv) trace main.foo                  # tracepoint (no stop, just log)
(dlv) regs
(dlv) disassemble
(dlv) help
(dlv) exit
```

## Deep Dive

### Building for debug

`dlv debug` / `dlv test` build with:

```bash
$ go build -gcflags="all=-N -l" -o __debug_bin .
```

- `-N`: no optimization → variables aren't dead-code-eliminated; expressions evaluate without rearrangement.
- `-l`: no inlining → stack traces show every function call.

To debug an *optimized* binary (closer to production):

```bash
$ dlv exec --build-flags="-tags=prod" ./bin
```

— but stepping will be jumpy and some variables will be "optimized out".

### Headless mode and IDE integration

```bash
$ dlv debug --headless --listen=:2345 --api-version=2 --accept-multiclient ./cmd/app
```

`--headless` exposes JSON-RPC; IDEs connect to it. `--accept-multiclient` allows reconnects (useful when an IDE crashes mid-session). `--api-version=2` is current; v1 is deprecated.

VS Code's Go extension launches `dlv` like this automatically when you start a debug session.

### Breakpoints

```
(dlv) break main.foo                 # by function
(dlv) break /pkg/file.go:42          # by file:line
(dlv) break /pkg/file.go:42 if user.ID > 100   # conditional
(dlv) breakpoints                    # list
(dlv) clear 2                        # remove breakpoint #2
(dlv) clearall
```

Conditional breakpoints evaluate Go expressions against the current frame. Functions can be called in conditions but with side-effect caveats (changing program state, allocating, etc.).

### Tracepoints

```
(dlv) trace main.foo
```

A tracepoint *doesn't stop* execution but logs each hit. Useful for "how many times did this run?" without performance impact of `fmt.Println`. Comparable to dtrace / bpftrace probes.

### Stepping

| Command   | What it does                                                |
|-----------|-------------------------------------------------------------|
| `next` / `n`   | Step over (don't enter calls).                          |
| `step` / `s`   | Step into.                                                |
| `stepout` / `so` | Step out of current function.                          |
| `continue` / `c` | Resume until next break.                                |
| `restart` / `r`  | Restart program (preserves breakpoints).                |

### Variable inspection

```
(dlv) print user
main.User {ID: 42, Name: "alice", Roles: []string len: 2, cap: 4, ["admin","user"]}
(dlv) p user.Name
"alice"
(dlv) p len(user.Roles)
2
(dlv) p user.Roles[0]
"admin"
(dlv) locals
(dlv) args
(dlv) print *user
(dlv) display user                   # auto-print on every stop
```

Print supports Go expressions: indexing, slicing, dereferencing, method calls (with caveats).

### Goroutines

```
(dlv) goroutines
* Goroutine 1 - User: ./main.go:42 main.foo (0x10500) [running]
  Goroutine 2 - User: ./worker.go:15 worker.Run (0x10800) [chan receive]
  Goroutine 3 - User: ./net.go:31 net.Read (0x10900) [IO wait]

(dlv) goroutine 2
Switched from 1 to 2 (thread 12345)
(dlv) bt
0  0x000000000043a8d0 in runtime.gopark
   at /usr/local/go/src/runtime/proc.go:380
1  0x000000000043a8d0 in runtime.chansend
   at /usr/local/go/src/runtime/chan.go:259
2  0x0000000000800200 in worker.Run
   at /worker.go:15
```

`goroutines` lists all; `goroutine N` switches to one; `bt` shows its stack.

Filter: `goroutines -running`, `goroutines -t 'main.handler'` (filter by function on stack).

### Channel inspection

```
(dlv) p myChannel
chan int { value: 42, qcount: 0, dataqsiz: 0, ... }
```

Channels print as a struct with internal fields. Useful for diagnosing deadlocks: a goroutine blocked on `chan send` with `qcount == dataqsiz` is waiting on a full buffer.

### Defers

```
(dlv) deferred
0: defer func1() at ./main.go:50
1: defer cleanup() at ./main.go:55
```

Lists pending defers for the current goroutine. Helpful when investigating why cleanup didn't run.

### Watchpoints

```
(dlv) watch -w &x        # break on write
(dlv) watch -r &x        # break on read
(dlv) watch -rw &x
```

Hardware watchpoints (limited count — usually 4 on amd64). Useful for "what is mutating this variable?".

### Disassembly

```
(dlv) disassemble
TEXT main.foo(SB) /pkg/main.go
   main.go:10  ► CMPQ ...
   main.go:11    MOVQ ...
```

The `►` marks the current PC. Step at instruction level with `stepi`/`nexti`.

### Core file debugging

```bash
$ ulimit -c unlimited           # allow core dumps
$ # crash the program; OS writes /tmp/core.<pid>
$ dlv core ./bin /tmp/core.12345
(dlv) goroutines
(dlv) goroutine 1 bt
```

Post-mortem inspection without a live process. Necessary for production-only crashes.

For Go-specific stack traces from a crash, also use `GOTRACEBACK=crash ./bin` which writes a core file automatically on panic.

### Attaching to a running process

```bash
$ dlv attach 12345
```

On Linux, requires either:

- Running as root, or
- `CAP_SYS_PTRACE` capability, or
- `/proc/sys/kernel/yama/ptrace_scope` set to `0`, or
- The target process must be a child of dlv (e.g., spawned with `--accept-multiclient` mode).

`dlv detach -k` detaches and leaves the process running; without `-k`, the process is killed on detach.

### Remote debugging

```bash
# On target (e.g., a container)
$ dlv exec --headless --listen=:2345 --api-version=2 --accept-multiclient ./bin

# Locally
$ dlv connect target-host:2345
```

Pair with port-forwarding for k8s pods: `kubectl port-forward pod/foo 2345:2345`.

### `dlv test`

```bash
$ dlv test ./pkg                          # debug all tests
$ dlv test ./pkg -- -test.run TestFoo
```

Compiles a test binary with `-N -l`, runs, attaches `dlv`. Same prompt as `dlv debug`.

### Source-level features

```
(dlv) list /pkg/file.go:42        # print source
(dlv) list                         # print around current PC
(dlv) frame 2                      # switch to stack frame 2
(dlv) up / down                    # navigate frames
(dlv) regs                         # CPU registers
(dlv) stack -full                  # extra-verbose backtrace
```

### Function calls from the prompt

```
(dlv) call foo(42)
```

Calls a function in the target process. Comes with caveats: must not deadlock, allocations can affect GC, etc. Disabled by default for safety in some builds; enable with `-allow-non-terminal-function-calls=true`.

### Configuration

`~/.config/dlv/config.yml`:

```yaml
aliases:
  bp: breakpoints
  d: disassemble

substitute-path:
  - {from: /build/src, to: /Users/me/src}

max-string-len: 1024
max-array-values: 100
show-location-expr: false
```

`substitute-path` is critical for binaries compiled on another machine — maps the embedded paths to your local checkout.

## Standard Library Hooks

- `runtime/debug.PrintStack()`, `runtime.Stack(buf, all)` — programmatic stack dumps.
- `runtime.SetCrashOutput` (1.23+) — redirect crash logs.
- `os/signal` — signals for graceful shutdown / debug triggers.
- `runtime.GC()`, `runtime.GOMAXPROCS()` — runtime introspection.

Many `dlv` features rely on the runtime exposing internal data structures; the `runtime` package's `_g_`, `m`, `p` types are what `dlv` parses.

## Real-World Patterns

### 1. Quick debug

```bash
$ dlv debug ./cmd/myapp
(dlv) break main.run
(dlv) c
(dlv) p config
(dlv) c
```

### 2. Debug a failing test

```bash
$ dlv test ./pkg -- -test.run TestFoo
(dlv) break /pkg/foo.go:42
(dlv) c
(dlv) p result
```

### 3. Attach to k8s pod

```bash
$ kubectl exec -it pod/foo -- dlv attach 1 --headless --listen=:2345 --api-version=2 --accept-multiclient
$ kubectl port-forward pod/foo 2345:2345
$ dlv connect localhost:2345
```

The target binary must have been built with debug info (no `-ldflags="-s -w"`).

### 4. Post-mortem from a panic

```bash
$ GOTRACEBACK=crash ./bin             # crash writes core file
$ dlv core ./bin core.123
(dlv) goroutines
(dlv) goroutine 1 bt
```

### 5. Tracepoint instead of `fmt.Println`

```
(dlv) trace main.handler
(dlv) continue
> main.handler called from ...
> main.handler called from ...
```

No code changes; no rebuild; toggle off when done.

### 6. Conditional breakpoint to catch a specific user

```
(dlv) break /pkg/handler.go:30 if r.URL.Path == "/users/42"
```

### 7. VS Code launch config

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug app",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/app"
    }
  ]
}
```

VS Code's Go extension manages dlv invocation.

## Anti-Patterns & Gotchas

**Debugging a `-ldflags="-s -w"` binary.** No DWARF; dlv prints "no source info". Build for debugging without those flags.

**Attaching without `ptrace_scope=0`.** Linux denies the syscall. Either root, capabilities, or kernel setting.

**Calling functions that block.** `call myBlocker()` deadlocks the debugger. Avoid in production debug sessions.

**Setting watchpoints on stack variables.** When the variable goes out of scope, the watchpoint fires on whatever new value occupies that address — confusing. Watch heap-allocated values.

**Stepping through optimized code.** Variables show as "optimized out"; line numbers jump unexpectedly. Build with `-N -l` (the dlv default).

**Forgetting `substitute-path` for remote/container debugging.** Source paths in the binary point to `/build/...`; your local checkout is `/home/me/...`. Map them in config.

**Detaching without `-k`.** Kills the target. Use `dlv detach -k` to leave it running.

**Headless mode on a public interface.** Anyone connecting can read memory and call functions. Bind to localhost or behind VPN.

**Confusing `step` and `next`.** `step` enters every call (including stdlib); `next` stays at the current call depth. Most of the time you want `next`.

**Trying to debug a stripped binary.** No DWARF → variable inspection limited. Use `objdump`/`nm` instead.

**Long-running `dlv attach` on a production service.** The target is stopped while at a breakpoint. Don't break on hot paths in prod.

**Expecting `dlv` to interpret `go func() {}` as creating a goroutine you can step into.** It can; use `step` on the `go` keyword or set a breakpoint inside the closure and `continue`.

## Performance Notes

- `dlv debug` build: 2–10 s (compiles with `-N -l`).
- Breakpoint trap overhead: ~10µs (negligible).
- Stepping a single line: <50ms.
- Headless RPC roundtrip: <10ms locally.
- Print variable: <50ms for simple types; longer for deeply nested.
- Attach to running process: <500ms.

`dlv` slows the target only at breakpoints and during inspection; between breaks the program runs full speed (but compiled with `-N -l`, so ~30–50% slower than optimized).

## How Big Companies Use It

- **Google** uses Delve internally for Go service debugging; the project receives sustained Google engineering: https://github.com/go-delve/delve.
- **Uber** uses dlv for production-incident debugging via remote attach on canary pods: https://eng.uber.com.
- **Cloudflare** uses dlv for live debugging of edge workers; the headless mode runs in their containers: https://blog.cloudflare.com.
- **HashiCorp** distributes a dlv-friendly debug image alongside Terraform for plugin developers: https://github.com/hashicorp/terraform-plugin-sdk.
- **CockroachDB** uses dlv core-file analysis for post-mortem investigation: https://www.cockroachlabs.com/docs.
- **Tailscale** uses dlv attach for live debugging of `tailscaled`: https://tailscale.com/blog.
- **GoLand / VS Code Go** plugins are dlv frontends.

## Source Code References

`dlv` is at `github.com/go-delve/delve`.

- Source: [`github.com/go-delve/delve`](https://github.com/go-delve/delve).
- CLI: [`cmd/dlv`](https://github.com/go-delve/delve/tree/master/cmd/dlv).
- Process control: [`pkg/proc`](https://github.com/go-delve/delve/tree/master/pkg/proc).
- DWARF: [`pkg/dwarf`](https://github.com/go-delve/delve/tree/master/pkg/dwarf).
- Service / JSON-RPC: [`service`](https://github.com/go-delve/delve/tree/master/service).
- Go runtime knowledge: [`pkg/proc/g.go`](https://github.com/go-delve/delve/blob/master/pkg/proc/g.go).

(MIT License.)

## Further Reading

- "Delve docs": https://github.com/go-delve/delve/tree/master/Documentation.
- "Getting started with Delve" (Derek Parker): https://github.com/derekparker/delve-tutorial.
- "Debugging Go programs with Delve" (Alex Edwards): https://www.alexedwards.net/blog/an-introduction-to-debugging-with-delve.
- "Debugging Go with VS Code": https://github.com/golang/vscode-go/wiki/debugging.
- "Production debugging in Go" (Sasha Goldshtein): https://blog.sentry.io/debugging-tail-latency-with-delve.
- "Core dumps in Go" (Filippo Valsorda): https://github.com/golang/go/wiki/CoreDumpDebugging.

## Exercises / Self-Check

1. Start a `dlv debug` session, set a conditional breakpoint on a function, and trigger it only for specific input.
2. Use `goroutines` and `goroutine N bt` to inspect a deadlocked program. Identify the blocked channel operation.
3. Generate a core dump (`GOTRACEBACK=crash`) on a panic. Analyze with `dlv core`.
4. Set up `dlv` headless on a container; connect from your local machine. Configure `substitute-path` so source paths resolve.
5. Use a tracepoint to count how many times a function is called during a load test.
