# `os` and `os/exec` — Processes, Files, Environment

## TL;DR

`os` is the platform-portable wrapper for files, environment variables, processes, signals, and stdio. `os/exec` runs subprocesses with controlled environment, args, stdio piping, and timeouts. The 1.24 `os.Root` API is the modern answer to path-traversal safety (replaces hand-rolled `filepath.Join` validation). Long-running services almost always use `os.Signal` + `context` for graceful shutdown.

## Mental Model

```
os.File ── implements io.Reader, Writer, Closer, Seeker, ReaderAt, WriterAt, fs.File
        ── wraps an OS file descriptor (Unix) or HANDLE (Windows)

exec.Cmd (struct, not interface):
        ── configure: Path, Args, Env, Dir, Stdin/Stdout/Stderr, Cancel
        ── execute:  cmd.Run()  or  cmd.Start() then cmd.Wait()
        ── interrogate: cmd.ProcessState, cmd.ExitCode(), cmd.Err

os.Root (1.24+):
        ── opaque sandbox handle anchored at one directory
        ── Open/OpenRoot/Stat refuse paths that escape the anchor
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println("PID:", os.Getpid())
	home, _ := os.UserHomeDir()
	fmt.Println("HOME:", home)
	if err := os.WriteFile("/tmp/hello.txt", []byte("hi\n"), 0o644); err != nil {
		fmt.Println(err)
	}
	// Output (varies):
	// PID: 12345
	// HOME: /Users/you
}
```

```go
import "os/exec"

out, err := exec.Command("git", "rev-parse", "HEAD").Output()
if err != nil { /* may be *exec.ExitError */ }
fmt.Println(string(out))
```

## Deep Dive

### File operations

- `os.Open` — read-only.
- `os.Create` — read/write, truncate, mode 0666 before umask.
- `os.OpenFile(name, flag, mode)` — full control. Flags: `O_RDONLY|O_WRONLY|O_RDWR`, `O_APPEND`, `O_CREATE`, `O_EXCL`, `O_TRUNC`, `O_SYNC`.
- `os.ReadFile` / `os.WriteFile` — whole-file convenience.
- `os.Remove`, `os.RemoveAll`, `os.Rename`, `os.Symlink`, `os.Link`, `os.Truncate`, `os.Chmod`, `os.Chown`.
- `os.MkdirAll` — `mkdir -p`.
- `os.TempDir`, `os.CreateTemp(dir, pattern)`, `os.MkdirTemp(dir, pattern)`.

### Environment

- `os.Getenv(key)` — `""` if missing (no error).
- `os.LookupEnv(key) (string, bool)` — distinguish unset from empty.
- `os.Setenv`, `os.Unsetenv`, `os.Clearenv`, `os.Environ() []string`.
- `os.ExpandEnv("$HOME/data")` — Unix-style expansion.

### Process info

- `os.Getpid`, `os.Getppid`.
- `os.Hostname`.
- `os.UserCacheDir`, `os.UserConfigDir`, `os.UserHomeDir`.

### `os.Args` and `os.Exit`

```go
if len(os.Args) < 2 { os.Exit(2) }
```

`os.Exit` does NOT run deferred functions. Always prefer `return` from `main` if you have cleanup.

### `os.Root` (since 1.24) — path-safe filesystem access

```go
root, err := os.OpenRoot("/var/data")
if err != nil { return err }
defer root.Close()

f, err := root.Open("user/cache.bin")
// "..", absolute paths, symlinks escaping /var/data → fail with error
```

Operations relative to a `Root`:

- `root.Open`, `root.OpenFile`, `root.Create`.
- `root.Stat`, `root.Mkdir`, `root.MkdirAll`, `root.Remove`, `root.RemoveAll`, `root.Rename`.
- `root.OpenRoot(name)` — open a subdirectory as its own Root.

Why it matters: hand-rolled `filepath.Join(base, untrusted)` + `strings.HasPrefix` checks are notoriously hard to get right (TOCTOU, symlinks, Windows reserved names). `os.Root` uses platform syscalls (`openat`, `O_NOFOLLOW`, `RESOLVE_BENEATH`) to make the check race-free.

### `os/exec` — spawning subprocesses

```go
cmd := exec.Command("ls", "-la", "/tmp")
cmd.Dir = "/tmp"
cmd.Env = append(os.Environ(), "LANG=C")
out, err := cmd.Output()
```

Patterns:

- `cmd.Output()` — captures stdout; returns `*exec.ExitError` (which contains `Stderr` since 1.6) on non-zero exit.
- `cmd.CombinedOutput()` — stdout + stderr interleaved.
- `cmd.Run()` — runs and waits, no captured output.
- `cmd.Start()` + `cmd.Wait()` — for piping or async management.

### Piping stdio

```go
cmd := exec.Command("grep", "ERROR")
cmd.Stdin = strings.NewReader(largeLog)
cmd.Stdout = os.Stdout
cmd.Run()
```

Or:

```go
stdout, _ := cmd.StdoutPipe()
cmd.Start()
sc := bufio.NewScanner(stdout)
for sc.Scan() { handle(sc.Text()) }
cmd.Wait()
```

### Context-aware execution (since 1.7, improved through 1.20)

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
out, err := exec.CommandContext(ctx, "slowcmd").Output()
```

If the context is cancelled, the subprocess receives a signal (default `os.Kill`). Customize with `cmd.Cancel` (since 1.20) and `cmd.WaitDelay`.

### `cmd.Cancel` and `cmd.WaitDelay` (since 1.20)

```go
cmd := exec.CommandContext(ctx, "long-running")
cmd.Cancel = func() error { return cmd.Process.Signal(syscall.SIGTERM) }
cmd.WaitDelay = 3 * time.Second // after SIGTERM, if still alive: SIGKILL
```

Graceful shutdown for child processes: SIGTERM first, escalate to SIGKILL after a grace period.

### `exec.LookPath` and PATH safety

`exec.LookPath("git")` resolves a binary on PATH. Avoid running untrusted binary names without validating; on Windows, the resolution rules differ enough that 1.19+ requires absolute paths in some cases (CVE-2022-30580 era hardening).

### Standard streams

`os.Stdin`, `os.Stdout`, `os.Stderr` are `*os.File`. Replace them in tests by wrapping/swapping; the `os` variables are mutable.

## Standard Library Hooks

- `io/fs` — `os.DirFS` and `os.Root` produce `fs.FS` implementations.
- `path/filepath` — OS-aware path manipulation.
- `os/signal` — signal handling (separate page).
- `os/user` — user/group lookups.
- `syscall` — when `os` doesn't cover a primitive.
- `golang.org/x/sys/unix` — modern syscall package for niche OS features.

## Real-World Patterns

### 1. Atomic file replace

```go
import (
	"os"
	"path/filepath"
)

func atomicWrite(path string, data []byte) error {
	dir := filepath.Dir(path)
	tmp, err := os.CreateTemp(dir, ".tmp-*")
	if err != nil { return err }
	if _, err := tmp.Write(data); err != nil { tmp.Close(); os.Remove(tmp.Name()); return err }
	if err := tmp.Sync(); err != nil { tmp.Close(); os.Remove(tmp.Name()); return err }
	if err := tmp.Close(); err != nil { os.Remove(tmp.Name()); return err }
	return os.Rename(tmp.Name(), path)
}
```

Use case: configuration writes that must be all-or-nothing across crashes.

### 2. Subprocess with timeout, capture stderr separately

```go
import (
	"bytes"
	"context"
	"errors"
	"os/exec"
	"time"
)

func runWithTimeout(ctx context.Context, name string, args ...string) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	var stdout, stderr bytes.Buffer
	cmd := exec.CommandContext(ctx, name, args...)
	cmd.Stdout = &stdout
	cmd.Stderr = &stderr
	if err := cmd.Run(); err != nil {
		var xerr *exec.ExitError
		if errors.As(err, &xerr) {
			return "", fmt.Errorf("%s exit %d: %s", name, xerr.ExitCode(), stderr.String())
		}
		return "", err
	}
	return stdout.String(), nil
}
```

Use case: CLI wrappers (helm, kubectl, terraform shellouts).

### 3. Untrusted path read via `os.Root`

```go
import "os"

root, err := os.OpenRoot("/srv/userdata")
if err != nil { return err }
defer root.Close()

f, err := root.Open(userSuppliedPath) // "../../etc/passwd" → error
if err != nil { return err }
defer f.Close()
```

Use case: serving user-uploaded files from a sandboxed directory.

### 4. Environment-driven config

```go
func mustEnv(key string) string {
	v, ok := os.LookupEnv(key)
	if !ok { log.Fatalf("missing env %s", key) }
	return v
}
```

Use case: 12-factor app config; matches Docker/K8s deploy patterns.

### 5. Streaming stdout from a long-running subprocess

```go
cmd := exec.CommandContext(ctx, "tail", "-f", "/var/log/syslog")
stdout, _ := cmd.StdoutPipe()
if err := cmd.Start(); err != nil { return err }
go func() {
	defer stdout.Close()
	sc := bufio.NewScanner(stdout)
	for sc.Scan() { process(sc.Text()) }
}()
return cmd.Wait()
```

Use case: log forwarders, dev-tool watchers.

## Anti-Patterns & Gotchas

**`os.Exit` instead of returning from main.** Skips defers; leaks tempfiles.

**`os.RemoveAll` on user-controlled input.** Always validate the path is inside the intended directory; use `os.Root` if possible.

**Running shell commands with `exec.Command("sh", "-c", userInput)`.** Shell injection. Never. Construct argv directly.

**Ignoring `cmd.Wait()` errors.** Resource leak: the OS keeps zombie process records.

**`os.Open` of an empty path.** Returns an error; don't pass user-controlled empty strings without checking.

**`filepath.Join(base, untrusted)` for path safety.** Not a security boundary — `..` collapses, but symlinks escape. Use `os.Root`.

**Capturing stdout *and* setting `cmd.Stdout` to your own writer.** Mutually exclusive: `Output()` won't work if `Stdout` is already set.

**Forgetting that `cmd.Env = nil` means "inherit parent env"**. `cmd.Env = []string{}` means "empty env" — different.

**`os.Setenv` in tests** without `t.Cleanup`. Mutates global state; later tests see the leftover.

**Comparing `exec.ExitError` with `==`.** Use `errors.As` to extract.

## Performance Notes

- `os.ReadFile` reads the whole file in one go (allocates a buffer the size of the file). For multi-GB files, stream.
- `os.WriteFile` opens with `O_TRUNC|O_CREATE|O_WRONLY`; equivalent to `Create` + `Write`.
- `cmd.Start()` is fork+exec on Linux (`clone(CLONE_VM)` since modern Go). Cost: ~milliseconds.
- Pipe IO between Go and subprocess goes through OS pipes — typical throughput hundreds of MB/s.
- `os.Root` uses `openat2` with `RESOLVE_BENEATH` on Linux when available; falls back to `openat` + verification otherwise.
- Frequent `os.Stat` in a hot loop is expensive — cache or use `fs.WalkDir` which uses `DirEntry`.

## How Big Companies Use It

- **Docker / containerd** uses `os/exec` extensively to manage container runtimes (`runc`, `containerd-shim`).
- **Kubernetes kubelet** uses `os/exec` to invoke container runtime and CNI plugins; uses `context` everywhere for cancellation.
- **HashiCorp Terraform** wraps provider plugin subprocesses with `os/exec`; gRPC-over-stdio between Terraform core and plugins.
- **GitHub Actions runner** (Go) uses `exec.CommandContext` with cancel hooks for step timeouts and cancellations.
- **Caddy** uses `os.Root` (1.24+) for serving user-uploaded files safely; before 1.24 it had hand-rolled path validation.

## Source Code References

Pinned to `go1.26`.

- `os.File`: [`src/os/file.go`](https://github.com/golang/go/blob/master/src/os/file.go).
- `os.Root`: [`src/os/root.go`](https://github.com/golang/go/blob/master/src/os/root.go).
- `exec.Cmd`: [`src/os/exec/exec.go`](https://github.com/golang/go/blob/master/src/os/exec/exec.go).
- Linux `openat2` use: [`src/internal/poll/sendfile_linux.go`](https://github.com/golang/go/blob/master/src/internal/poll) and around.
- `cmd.Cancel` / `WaitDelay`: introduced in 1.20, [release notes](https://go.dev/doc/go1.20#os-exec).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/os, https://pkg.go.dev/os/exec.
- Go 1.24 release notes on `os.Root`: https://go.dev/doc/go1.24.
- Filippo Valsorda, "Go's path/filepath is not for security" (CVE writeups, ~2022).
- Russ Cox, "The Go Programming Language Specification: Predeclared identifiers" (covers `os.Args`).

## Exercises / Self-Check

1. Implement atomic-write (above). Test that a crash between `Sync` and `Rename` leaves the original file intact.
2. Use `os.Root` to safely serve files from `/tmp/userdata`. Show that `..` paths fail.
3. Write a small `grep` clone with `exec.Command` calling the system grep, then rewrite it in pure Go with `bufio.Scanner`. Compare ergonomics and performance.
4. Demonstrate `cmd.Cancel` + `cmd.WaitDelay` by running `sleep 60` and cancelling after 1 second. Show the SIGTERM-then-SIGKILL escalation.
5. Trace through what happens when `os.WriteFile("/dev/full", ...)` is called. Which error do you get? When?
