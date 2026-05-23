# `context`

## TL;DR

`context.Context` is the standard way to propagate cancellation, deadlines, and request-scoped values across API boundaries and between goroutines. Every blocking or long-running function should accept `ctx context.Context` as its first parameter. The single biggest gotcha: `context.Value` is meant for **request-scoped** values (request ID, auth token), not for passing optional parameters — abusing it produces magical, untestable code. Second gotcha: `WithCancel`, `WithTimeout`, `WithDeadline` return a `CancelFunc` that **must** be called, or the parent's child list (and any goroutines waiting on `Done`) leaks until the parent itself cancels.

## Mental Model

```
ctx tree:

  Background()        <- root, never canceled
       |
   WithCancel()       <- cancelCtx (has cancel func, Done chan)
       |
   WithTimeout(5s)    <- timerCtx (cancelCtx + timer)
       |
   WithValue("k","v") <- valueCtx (key-value map of one entry)
       |
   passed to RPC, db, etc.

When a parent cancels, every descendant cancels too (post-order).
ctx.Done() closes; ctx.Err() returns Canceled, DeadlineExceeded, or
a cause from context.WithCancelCause / WithDeadlineCause (since 1.20).
```

A Context is **immutable** — `WithX` returns a *new* derived Context. Trees are reference-counted by parent pointers; canceling a node walks down the children list and cancels each.

## Syntax & Basic Usage

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(200 * time.Millisecond):
		fmt.Println("slept full")
	case <-ctx.Done():
		fmt.Println("ctx:", ctx.Err())
	}
	// Output:
	// ctx: context deadline exceeded
}
```

The `defer cancel()` is mandatory — even when the timeout fires, you must release the timer.

## Deep Dive

### The interface

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

Four methods. `Done` returns the canonical cancellation channel; `Err` returns nil while live, then `Canceled` or `DeadlineExceeded` once `Done` closes. `Value` walks up the parent chain.

### `context.Background` vs `context.TODO`

- `Background()` — the root; use at the top of `main`, request handlers, init.
- `TODO()` — exactly identical except its identity is "I haven't decided what context belongs here yet." Linters can flag `TODO` as something to address. **Never** use `nil` as a context — many functions panic.

### `WithCancel`

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
go work(ctx)
```

`cancel` is idempotent. Calling it twice is fine; the second is a no-op. Failing to call it leaks the child until the parent cancels (the parent tracks children to propagate cancellation; the entry stays as long as the child is alive).

### `WithTimeout` / `WithDeadline`

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
ctx, cancel := context.WithDeadline(parent, time.Now().Add(2*time.Second))
```

Same thing, different units. Both schedule a timer that fires `cancel()` automatically; you still must `defer cancel()` to release the timer immediately on early return.

### `WithValue`

```go
ctx := context.WithValue(parent, key, value)
v := ctx.Value(key) // walks up parent chain
```

**Use unexported key types** to prevent collisions:

```go
type ctxKey struct{}
var reqIDKey = ctxKey{} // ctxKey-typed sentinel
ctx = context.WithValue(ctx, reqIDKey, "abc-123")
```

Don't use strings as keys. They collide across packages.

`Value` is O(depth) — each lookup walks the parent chain. Don't put hot-path data in there.

### Cancellation propagation

Cancellation flows from parent to child, never up. Canceling a child does not cancel its parent. Each cancelCtx maintains a `children` set; on cancel, it closes its own `Done` channel and recursively cancels every entry in `children`, then drops the set (so the GC can reclaim them).

### `WithCancelCause` / `WithDeadlineCause` / `WithTimeoutCause` (since 1.20)

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(errors.New("user disconnected"))

<-ctx.Done()
fmt.Println(ctx.Err())           // context canceled
fmt.Println(context.Cause(ctx))  // user disconnected
```

`Err` still returns the standard sentinel (`Canceled` / `DeadlineExceeded`) for backward compatibility; `context.Cause(ctx)` returns the specific reason. Use cause when you need to plumb a useful error up to the caller.

### `context.AfterFunc` (since 1.21)

```go
stop := context.AfterFunc(ctx, func() {
	log.Println("ctx canceled; cleaning up")
})
defer stop()
```

Runs `func` in its own goroutine when `ctx` is canceled. `stop()` removes the registration. This is *much* cheaper than a watchdog goroutine that does `<-ctx.Done()`; it doesn't allocate a goroutine until the cancel happens.

### `context.WithoutCancel` (since 1.21)

```go
ctx2 := context.WithoutCancel(parent)
```

`ctx2` inherits values from `parent` but has no deadline and is never canceled. Used when you want to spawn a background task that should outlive its triggering request — for example, asynchronously fire an audit log after returning to the caller.

### Context immutability

A Context is a value, but the cancelCtx has internal mutable state (children set, error). You may pass a Context to any number of goroutines safely. The only mutation is via the `cancel` closure returned at creation time.

### "Context tree" lifetime rules

1. Always derive a child from the appropriate parent — usually the one your function received as `ctx`.
2. Always `defer cancel()` for any `WithCancel`/`WithTimeout`/`WithDeadline`.
3. Never store a Context in a struct unless that struct *is* the request (rare). Pass it as a function parameter.
4. The Context is always the first parameter: `func F(ctx context.Context, ...)`.
5. Don't pass `nil`. Use `context.Background()` or `context.TODO()`.

## Standard Library Hooks

- `net/http` — every request handler gets a `Context` via `r.Context()` that cancels when the connection is closed.
- `database/sql` — every method has a `Context` variant (`QueryContext`, `ExecContext`). Use them; the non-Context versions are essentially deprecated.
- `os/exec` — `exec.CommandContext(ctx, ...)` kills the process when `ctx` is canceled.
- `signal.NotifyContext(parent, signals...)` returns a Context that cancels on the first signal.
- `golang.org/x/sync/errgroup.WithContext` — propagates cancellation to all workers.
- `runtime/pprof.Do(ctx, labels, f)` attaches labels and runs `f`, useful for cross-goroutine profiling.

## Real-World Patterns

### 1. Request-scoped timeout

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	rows, err := db.QueryContext(ctx, "SELECT ...")
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}
	defer rows.Close()
	// ...
}
```

The handler derives a tighter timeout from the request's own context. If the client disconnects, `r.Context()` cancels, and so does `ctx`; if 2 s pass, `ctx` cancels alone.

### 2. Signal-cancelled main

```go
package main

import (
	"context"
	"os/signal"
	"syscall"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	if err := run(ctx); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context) error {
	srv := &http.Server{Addr: ":8080"}
	go func() {
		<-ctx.Done()
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		srv.Shutdown(shutdownCtx)
	}()
	return srv.ListenAndServe()
}
```

Modern Go idiom: SIGINT/SIGTERM cancels `ctx`, the goroutine triggers a graceful shutdown with its own 5 s budget. Note `WithoutCancel(ctx)` would also work for the shutdown context.

### 3. AfterFunc for cleanup (1.21+)

```go
func startUploader(ctx context.Context, f *os.File) {
	context.AfterFunc(ctx, func() {
		_ = f.Close()
	})
}
```

Replaces the older `go func() { <-ctx.Done(); f.Close() }()`. No idle goroutine.

### 4. Propagate values for tracing (sparingly)

```go
type traceKey struct{}

func WithTraceID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, traceKey{}, id)
}

func TraceID(ctx context.Context) string {
	id, _ := ctx.Value(traceKey{}).(string)
	return id
}
```

Use Context.Value for things like request IDs, auth principals, locales — values whose lifetime matches the request. **Not** for repository handles, loggers, or other dependencies that belong in the function signature or a struct.

### 5. Cause for distinguishing why we stopped (1.20+)

```go
ctx, cancel := context.WithCancelCause(parent)
go func() {
	if err := watch(); err != nil {
		cancel(err)
	}
}()

<-ctx.Done()
if cause := context.Cause(ctx); cause != nil && !errors.Is(cause, context.Canceled) {
	log.Printf("watcher died: %v", cause)
}
```

Allows the canceller to attach an error that the consumer can inspect via `context.Cause`.

## Anti-Patterns & Gotchas

**Storing a Context in a struct.** "ctx in struct" is a Go-team-recommended anti-pattern: it pins a lifetime to a struct that doesn't necessarily share it. Pass `ctx` as the first parameter of every function.

**Passing `nil` as a Context.** Many functions panic; some quietly do nothing. Use `context.Background()` or `context.TODO()`.

**Forgetting to call `cancel`.** `defer cancel()` immediately after creating the child. `go vet`'s `lostcancel` analyzer catches most cases.

**Using `context.Value` for required arguments.** Magical. Untestable. Use a function parameter or a struct field.

**Using a `string` as a context key.** Collides across packages. Use an unexported struct type as the key.

**Re-using a Context across requests.** The Context's lifetime *is* the request. Reusing leaks values and cancellation semantics.

**Cancelling the parent to cancel a child.** Cancellation flows down, not up. Cancel the child directly.

**Treating `Background()` and `TODO()` as different at runtime.** They're behaviorally identical. The distinction is documentation.

**Ignoring `ctx.Err()` in long-running work.** A goroutine that never checks `<-ctx.Done()` and never makes a Context-aware call will not stop on cancel. Cancellation is *cooperative*.

**`context.WithoutCancel(nil)`.** Panics. Same as any other context op on nil.

## Performance Notes

- Creating a `cancelCtx`: ~150 ns (allocates a struct and a channel).
- Creating a `timerCtx`: ~250 ns (extra timer allocation).
- `ctx.Done()`: returns the cached channel; no allocation.
- `ctx.Value(k)`: O(depth of the value chain). Deep value stacks (10+) start to matter.
- `cancel()`: O(N children).
- `context.AfterFunc` allocates a small registration node but **no goroutine** until the cancel actually fires.
- For tight loops, avoid creating a child Context per iteration. Hoist it out.

## How Big Companies Use It

- **`net/http`** propagates request cancellation via `Request.Context()` — every server-side request has a per-connection Context that cancels when the client disconnects.
- **`gRPC-Go`** uses Context for cancellation, deadlines, and metadata propagation; every RPC has a Context. See [`google.golang.org/grpc`](https://github.com/grpc/grpc-go).
- **Kubernetes `client-go`** uses Context heavily; informers and reflectors take a Context and stop when it's canceled.
- **CockroachDB** uses Context for query cancellation, tracing spans, and distributed timeouts.
- **OpenTelemetry-Go** stores span state in the Context — the textbook example of when Context.Value is appropriate.
- **Vault** uses Context for vault.RequestContext to bind auth and timeout.

## Source Code References

Pinned to `go1.26`.

- `context` package: [`src/context/context.go`](https://github.com/golang/go/blob/master/src/context/context.go). One file, well-commented.
- `WithCancelCause`: same file, search `WithCancelCause`.
- `AfterFunc`: same file, search `AfterFunc`.
- `WithoutCancel`: same file, search `WithoutCancel`.
- `signal.NotifyContext`: [`src/os/signal/signal.go`](https://github.com/golang/go/blob/master/src/os/signal/signal.go).
- `lostcancel` analyzer: [`src/cmd/vet/lostcancel.go`](https://github.com/golang/go/blob/master/src/cmd/vet/lostcancel.go) (and `golang.org/x/tools/go/analysis/passes/lostcancel`).

## Further Reading

- Go blog, "Go Concurrency Patterns: Context" (Sameer Ajmani): https://go.dev/blog/context
- Sameer Ajmani's original Context design: https://go.dev/talks/2014/gotham-context.slide
- Proposal: `context.WithCancelCause`: https://go.dev/issue/51365
- Proposal: `context.AfterFunc`: https://go.dev/issue/57928
- Proposal: `context.WithoutCancel`: https://go.dev/issue/40221
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns" — context as the primary cancellation mechanism
- Dave Cheney, "Context isn't for cancellation" (counterpoint): https://dave.cheney.net/2017/01/26/context-is-for-cancelation
- Filippo Valsorda, "Context-aware Go": https://words.filippo.io/

## Exercises / Self-Check

1. Why must you always call `cancel` from `WithCancel`/`WithTimeout`/`WithDeadline`? What leaks if you don't?
2. Convert `func F(ctx context.Context) { log := ctx.Value(logKey{}).(Logger); ... }` to a less magical API. What did you gain?
3. Implement `context.AfterFunc` using only the older API (`<-ctx.Done()` goroutine). What's the cost difference?
4. Why is `context.Cause(ctx)` necessary when `ctx.Err()` already exists?
5. What happens if you call `cancel(err)` (from `WithCancelCause`) twice with different errors? Why?
