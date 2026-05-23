# `runtime/debug` — GC Tuning, Build Info, Stack Dumps

## TL;DR

`runtime/debug` exposes ops-grade controls: `SetGCPercent`, `SetMemoryLimit` (the soft `GOMEMLIMIT`, 1.19+), `ReadBuildInfo` for the binary's module info, `Stack`/`PrintStack` for debug dumps, `FreeOSMemory` to return RSS aggressively. Use it to ship version info in `/version` endpoints, to tune the GC for memory-constrained or latency-sensitive workloads, and to capture diagnostics on shutdown.

## Mental Model

```
runtime/debug
   ├─ GC: SetGCPercent, SetMemoryLimit, FreeOSMemory
   ├─ Stack: Stack(), PrintStack()
   ├─ Build: ReadBuildInfo() → module versions, build tags, vcs revision
   └─ Crash: SetTraceback, SetPanicOnFault
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	info, ok := debug.ReadBuildInfo()
	if ok {
		fmt.Println("module:", info.Main.Path, info.Main.Version)
		for _, s := range info.Settings {
			if s.Key == "vcs.revision" { fmt.Println("rev:", s.Value) }
		}
	}
}
```

## Deep Dive

### `SetGCPercent`

```go
old := debug.SetGCPercent(50) // GC when heap grows 50% past last GC; default 100
debug.SetGCPercent(-1)         // disable GC (don't unless you know why)
```

Lower → more frequent, lower steady-state heap, higher CPU. Higher → less frequent, larger heap, lower CPU.

Tradeoff knob; default fits most workloads.

### `SetMemoryLimit` (since 1.19)

```go
debug.SetMemoryLimit(2 << 30) // 2 GiB soft limit
```

The GC tries to keep total heap + scavenger + runtime below this. Soft means it may exceed transiently if doing so prevents an OOM-kill. Effectively prevents the runaway-heap scenario.

Also settable via `GOMEMLIMIT=2GiB` env var.

### `ReadBuildInfo` (since 1.18)

```go
info, _ := debug.ReadBuildInfo()
fmt.Println(info.GoVersion)            // go1.26
fmt.Println(info.Main.Path)            // module path
fmt.Println(info.Main.Version)         // module version (often "(devel)" for local builds)
for _, dep := range info.Deps {
	fmt.Println(dep.Path, dep.Version)
}
for _, s := range info.Settings {
	// vcs (git/hg), vcs.revision, vcs.time, vcs.modified, GOOS, GOARCH, -ldflags, etc.
	fmt.Println(s.Key, s.Value)
}
```

`vcs.revision` is the git commit. Captured automatically when `go build` is run inside a git checkout (since 1.18 with `-buildvcs=true`, default).

### Stack dumps

```go
debug.PrintStack()        // current goroutine, to stderr
b := debug.Stack()         // returns []byte
```

Common in panic handlers: `defer func() { if r := recover(); r != nil { slog.Error("panic", "v", r, "stack", string(debug.Stack())) } }()`.

### `FreeOSMemory`

```go
debug.FreeOSMemory()
```

Forces the scavenger to return unused memory pages to the OS. Useful after a burst of allocation that won't recur. Modern Go does this automatically; explicit calls are rare.

### `SetTraceback`

```go
debug.SetTraceback("all") // include all goroutines on panic
```

Equivalent to `GOTRACEBACK=all` env var. Values: `none`, `single`, `all`, `system`, `crash`.

### `SetPanicOnFault`

```go
old := debug.SetPanicOnFault(true)
defer debug.SetPanicOnFault(old)
```

Makes faults from `unsafe`/`syscall` (e.g., mapped file pages disappearing) recoverable via `recover` instead of crashing. Use rarely, with care.

## Standard Library Hooks

- `runtime` — underlies `debug` functions.
- `runtime/metrics` for fine-grained runtime data.

## Real-World Patterns

### 1. `/version` endpoint

```go
import (
	"encoding/json"
	"net/http"
	"runtime/debug"
)

func version(w http.ResponseWriter, r *http.Request) {
	info, _ := debug.ReadBuildInfo()
	out := map[string]any{
		"go_version": info.GoVersion,
		"module":     info.Main.Path,
		"version":    info.Main.Version,
	}
	for _, s := range info.Settings {
		if s.Key == "vcs.revision" { out["revision"] = s.Value }
		if s.Key == "vcs.time"     { out["build_time"] = s.Value }
	}
	json.NewEncoder(w).Encode(out)
}
```

Use case: every service should expose its version for ops.

### 2. Memory-limited container

```go
import "runtime/debug"

func init() {
	// Respect Kubernetes memory limit, leaving 10% headroom.
	if limit := readCgroupMemLimit(); limit > 0 {
		debug.SetMemoryLimit(int64(float64(limit) * 0.9))
	}
}
```

Or just `GOMEMLIMIT=900MiB` in the pod spec.

### 3. Panic handler with stack

```go
func recoverer(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rv := recover(); rv != nil {
				slog.Error("panic",
					"value", rv,
					"stack", string(debug.Stack()),
				)
				http.Error(w, "internal error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

### 4. Latency-sensitive GC tuning

```go
debug.SetGCPercent(20)      // more-frequent, smaller pauses
debug.SetMemoryLimit(4 << 30)
```

Use case: real-time trading, game servers, latency SLO targets.

### 5. Dump all goroutines on SIGUSR2

```go
go func() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGUSR2)
	for range c {
		buf := make([]byte, 64<<10)
		n := runtime.Stack(buf, true)
		fmt.Fprintln(os.Stderr, string(buf[:n]))
	}
}()
```

Use case: diagnose hung services without killing them.

## Anti-Patterns & Gotchas

**`SetGCPercent(-1)` in production.** Heap grows unbounded.

**Calling `FreeOSMemory` after every allocation burst.** Wastes CPU; scavenger handles it.

**Hardcoding `GOMEMLIMIT` based on host memory** instead of container limit.

**Logging `debug.Stack()` on every error.** Expensive; reserve for panics and fatal paths.

**Trying to detect dev builds with `info.Main.Version == "(devel)"`** without also checking `vcs.modified`.

**Setting `SetTraceback("none")` to hide secrets in logs.** Better to redact at log layer; you still want stacks during dev.

## Performance Notes

- `SetGCPercent` and `SetMemoryLimit` take effect on the next GC cycle.
- `ReadBuildInfo` is cheap; results are baked into the binary.
- `debug.Stack` walks the current goroutine — microseconds.
- `runtime.Stack(buf, true)` for all goroutines: hundreds of microseconds to milliseconds, depending on count.

## How Big Companies Use It

- **Discord** wrote about lowering `GOGC` to reduce GC tail latency (the famous post about removing 50ms p99 spikes).
- **Cloudflare** tunes `GOMEMLIMIT` per Worker.
- **Caddy** exposes `/debug/buildinfo` on its admin endpoint.
- **Cockroach** ships build info via SQL `SELECT version()`.

## Source Code References

Pinned to `go1.26`.

- `runtime/debug`: [`src/runtime/debug/`](https://github.com/golang/go/tree/master/src/runtime/debug).
- `SetGCPercent`, `SetMemoryLimit`: same dir.
- `ReadBuildInfo`: [`src/runtime/debug/mod.go`](https://github.com/golang/go/blob/master/src/runtime/debug/mod.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/runtime/debug.
- Go blog, "Memory limit and GC" (1.19): https://go.dev/doc/go1.19#runtime.
- Discord, "Why Discord is switching from Go to Rust" (2020) — context for GC tuning.

## Exercises / Self-Check

1. Build a `/version` endpoint using `ReadBuildInfo`. Confirm the git revision shows in the output.
2. Set `GOMEMLIMIT=200MiB` (via env) and observe `GODEBUG=gctrace=1` traces.
3. Print all goroutine stacks on SIGUSR2.
4. Try `SetGCPercent(10)` and observe more frequent GC cycles via `gctrace`.
5. Read the Discord GC blog post; identify which `runtime/debug` knobs they exercised.
