# `os/signal` — Signal Handling and Graceful Shutdown

## TL;DR

`os/signal` delivers OS signals to a Go channel. `signal.Notify(ch, sigs...)` registers; `signal.Stop(ch)` deregisters; `signal.Reset` restores defaults. The 1.16 `signal.NotifyContext` is the modern entry point: it returns a `context.Context` that's cancelled when SIGINT or SIGTERM arrive — the canonical "graceful shutdown" recipe. Don't do work inside the signal handler goroutine; deliver to a channel and react in normal code.

## Mental Model

```
OS sends SIGTERM ──► Go runtime catches it ──► signal package fans out to registered channels
                                                          │
                              ┌───────────────────────────┴────────────┐
                              ▼                                        ▼
                   ch1 <- syscall.SIGTERM               ctx.Done() closes (NotifyContext)

Your code:
  ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
  defer stop()
  <- start servers with ctx, they Shutdown() when ctx.Done()
```

## Syntax & Basic Usage

```go
package main

import (
	"context"
	"fmt"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(),
		syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	fmt.Println("running; Ctrl-C to stop")
	select {
	case <-ctx.Done():
		fmt.Println("shutdown signal received")
	case <-time.After(60 * time.Second):
		fmt.Println("timeout")
	}
}
```

## Deep Dive

### `signal.Notify`

```go
ch := make(chan os.Signal, 1) // BUFFERED — non-buffered drops signals
signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
go func() {
	sig := <-ch
	cleanup()
}()
```

The channel must be buffered. The signal delivery is non-blocking: if the channel is full, the signal is dropped. Buffer size 1 is enough for the "first signal wins" pattern.

### `signal.NotifyContext` (since 1.16) — modern idiom

```go
ctx, stop := signal.NotifyContext(parent, signals...)
defer stop()
```

Returns a context cancelled when any of the listed signals fire. `stop()` releases the registration. Pair with `http.Server.Shutdown(ctx)`, `grpc.Server.GracefulStop()`, worker `select { case <-ctx.Done() }`.

### `signal.Stop` and `signal.Reset`

- `signal.Stop(ch)` — remove `ch` from the registration list; the channel is no longer delivered signals.
- `signal.Reset(sigs...)` — restore default OS handler (SIGTERM → terminate, etc.).
- `signal.Ignore(sigs...)` — make signals no-ops.

### Common signals and their meanings (Unix)

| Signal | Default | Typical use |
|--------|---------|-------------|
| `SIGINT` (2) | terminate | Ctrl-C; gracefully shut down |
| `SIGTERM` (15) | terminate | `kill <pid>`, container shutdown; graceful |
| `SIGHUP` (1) | terminate | Reload config; reopen log files |
| `SIGQUIT` (3) | core dump | Go's special: dumps all goroutine stacks |
| `SIGUSR1`/`SIGUSR2` | terminate | App-defined; toggle debug, rotate logs |
| `SIGKILL` (9) | terminate, *uncatchable* | Force kill — no Go handler runs |
| `SIGSTOP` | stop, *uncatchable* | Pause |

`SIGKILL` and `SIGSTOP` cannot be caught — never count on cleanup running. Plan for crash-safety separately.

### Windows

Windows has a small subset: `os.Interrupt` (mapped from Ctrl-C / Ctrl-Break) and a couple of console signals. Most Unix signals are inapplicable. Cross-platform code typically registers `os.Interrupt` and `syscall.SIGTERM` (which is delivered by container runtimes even on Windows containers).

### Default Go behavior on SIGINT/SIGTERM

If you don't call `signal.Notify`, the Go runtime kills the process on these signals (after running any deferred runtime cleanup but not user defers). Once you `Notify`, Go suppresses the default — you own the response.

### `SIGQUIT` and goroutine dumps

Sending `SIGQUIT` (Ctrl-\) to a Go program writes all goroutine stacks to stderr and dies. Set `GOTRACEBACK=all` to include runtime goroutines.

```go
// Restore Go's default SIGQUIT handler even if you've called signal.Notify elsewhere.
signal.Reset(syscall.SIGQUIT)
```

### SIGHUP for config reload

```go
hup := make(chan os.Signal, 1)
signal.Notify(hup, syscall.SIGHUP)
go func() {
	for range hup {
		reloadConfig()
	}
}()
```

Used by daemons (nginx-style: `kill -HUP <pid>` reloads without restart).

### Forwarding signals to subprocesses

If your Go binary is a process supervisor (init, container entrypoint), it must forward signals to children. Don't catch SIGTERM, exit, and leave the child alive.

```go
sig := <-sigCh
cmd.Process.Signal(sig)
```

### The double-signal pattern

Many CLIs accept one Ctrl-C as "shut down gracefully" and a second as "force quit":

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

go func() {
	<-ctx.Done()
	stop() // unregister; next signal kills the process via default handler
}()
```

## Standard Library Hooks

- `context.Context` for cancellation propagation.
- `net/http.Server.Shutdown(ctx)` — drains in-flight requests.
- `database/sql.DB.Close()` — should be called on shutdown.
- `runtime.GOTRACEBACK` — environment variable that controls SIGQUIT dump content.

## Real-World Patterns

### 1. HTTP server graceful shutdown

```go
import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os/signal"
	"syscall"
	"time"
)

func runServer(addr string, h http.Handler) error {
	ctx, stop := signal.NotifyContext(context.Background(),
		syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{Addr: addr, Handler: h}
	errCh := make(chan error, 1)
	go func() { errCh <- srv.ListenAndServe() }()

	select {
	case <-ctx.Done():
		slog.Info("shutting down")
	case err := <-errCh:
		return err
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return err
	}
	if err := <-errCh; err != nil && !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

Use case: every production HTTP service.

### 2. Worker that drains on signal

```go
func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			return
		case j := <-jobs:
			handle(j)
		}
	}
}

// In main:
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
defer stop()
var wg sync.WaitGroup
for i := 0; i < 8; i++ {
	wg.Add(1)
	go func() { defer wg.Done(); worker(ctx, jobs) }()
}
wg.Wait()
```

Use case: message queue consumers, background processors.

### 3. SIGHUP reload

```go
hup := make(chan os.Signal, 1)
signal.Notify(hup, syscall.SIGHUP)
go func() {
	for range hup {
		newCfg, err := loadConfig(path)
		if err != nil { slog.Error("reload failed", "err", err); continue }
		atomic.StorePointer(&cfgPtr, unsafe.Pointer(newCfg))
		slog.Info("config reloaded")
	}
}()
```

Use case: long-running daemons whose config must change without restart.

### 4. Log rotation trigger

```go
usr1 := make(chan os.Signal, 1)
signal.Notify(usr1, syscall.SIGUSR1)
go func() {
	for range usr1 {
		rotateLogFile()
	}
}()
```

Use case: companion of `logrotate(8)` which sends SIGUSR1 after renaming the log file.

## Anti-Patterns & Gotchas

**Unbuffered signal channel.** Signals get dropped silently. Always buffer ≥ 1.

**Heavy work in the signal goroutine.** That goroutine should read from the channel and dispatch — not call your entire shutdown sequence inline.

**Catching SIGKILL.** Impossible. Don't even register it.

**Not unregistering before exit.** `signal.Stop` (or `stop` from `NotifyContext`) prevents leftover handlers from firing during shutdown of subsystems.

**Catching SIGTERM but never propagating to children.** Supervisors zombify their children.

**Treating Ctrl-C as panic.** Use a normal shutdown path; panicking from a signal handler is a runtime error.

**Assuming Windows has Unix signals.** It doesn't. Only `os.Interrupt` is portable.

**Mixing `signal.Notify(ch, ...)` with `signal.NotifyContext`** for the same signals without thinking. The first registration wins per signal; subsequent ones add channels, not replace.

**Calling `signal.Reset` from a goroutine while another is calling `Notify`.** Race.

## Performance Notes

- Signal delivery is rare and cheap; ignore performance.
- `signal.Notify` adds the channel to a small global table — O(N) lookup at signal time, fine for N < 100.
- `NotifyContext` allocates a context plus a goroutine; one-time cost.

## How Big Companies Use It

- **Kubernetes** components (kubelet, kube-proxy, controllers) all use `signal.NotifyContext` for shutdown.
- **Caddy** uses SIGUSR1 for log reopen, SIGHUP for config reload.
- **HashiCorp Vault** uses SIGHUP for reload and SIGUSR1 for dumping internal state.
- **Container runtimes** (containerd, runc) propagate signals from the runtime to the contained process and back.
- **`go test`** itself catches SIGINT to print partial test results before exiting.

## Source Code References

Pinned to `go1.26`.

- `signal` package: [`src/os/signal/signal.go`](https://github.com/golang/go/blob/master/src/os/signal/signal.go).
- `NotifyContext`: same file (search the function).
- Signal handling in the runtime: [`src/runtime/signal_unix.go`](https://github.com/golang/go/blob/master/src/runtime/signal_unix.go).
- `http.Server.Shutdown`: [`src/net/http/server.go`](https://github.com/golang/go/blob/master/src/net/http/server.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/os/signal.
- Go blog, "Signal Handling" (older but relevant): https://go.dev/blog.
- Tail.dev / Tailscale on graceful shutdown best practices: https://tailscale.com/blog.

## Exercises / Self-Check

1. Write an HTTP server that handles SIGTERM by calling `Shutdown` with a 10-second timeout. Test by sending `kill` and watching in-flight requests complete.
2. Implement a config reloader using SIGHUP and `atomic.Pointer[Config]`.
3. Send SIGQUIT to a running Go program (Ctrl-\). What do you see? How does it differ with `GOTRACEBACK=all`?
4. Build a CLI that exits cleanly on first Ctrl-C and force-quits on second. Verify by interrupting twice quickly.
5. Why must the signal channel be buffered? Demonstrate with code that drops signals when unbuffered.
