# `context` — Cancellation, Deadlines, Values (Revisited)

## TL;DR

`context.Context` is the explicit cancellation/deadline/values value passed as the first argument through every Go API that may need to be cancelled. Use `context.WithCancel`, `WithTimeout`, `WithDeadline`, `WithCancelCause` (1.20+), `AfterFunc` (1.21+). Never store a context in a struct — pass it. The single most important thing: **propagate, don't ignore**. Every blocking call (DB query, HTTP request, RPC) takes a context; pass `r.Context()` through.

## Mental Model

```
ctx0 = context.Background()  (root; never cancels)
                  │
        WithTimeout/Cancel/Deadline/Value
                  │
                  ▼
ctx1 with parent ctx0   →   ctx1.Done() closes when parent does OR own cancel/deadline fires

ctx.Done()       returns a <-chan struct{} that closes on cancel
ctx.Err()        returns ctx.Canceled or ctx.DeadlineExceeded or nil
ctx.Value(key)   returns associated value (rarely used; for opaque IDs)
ctx.Deadline()   returns deadline if any
```

## Syntax & Basic Usage

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(100 * time.Millisecond):
		fmt.Println("did work")
	case <-ctx.Done():
		fmt.Println("cancelled:", ctx.Err())
	}
	// Output:
	// cancelled: context deadline exceeded
}
```

## Deep Dive

### Constructors

- `context.Background()` — root; for `main`, init, tests. Never cancelled.
- `context.TODO()` — placeholder; same as Background but signals "I haven't decided yet."
- `WithCancel(parent) (ctx, cancel)` — manual cancel.
- `WithTimeout(parent, d) (ctx, cancel)` — auto-cancels after `d`.
- `WithDeadline(parent, t) (ctx, cancel)` — auto-cancels at absolute time.
- `WithValue(parent, key, val)` — attach an opaque value.
- `WithCancelCause(parent) (ctx, cancel)` (1.20+) — `cancel(err)` records a reason.
- `WithoutCancel(parent)` (1.21+) — strip cancellation but keep values.
- `WithDeadlineCause`, `WithTimeoutCause` (1.21+) — set the deadline plus a cause.

### Cancellation propagation

```go
parent, parentCancel := context.WithCancel(context.Background())
child, _ := context.WithTimeout(parent, 5*time.Second)

parentCancel() // child.Done() fires immediately
```

Children cancel when *any* ancestor cancels.

### `context.Cause` (1.20+)

```go
ctx, cancel := context.WithCancelCause(context.Background())
cancel(errors.New("quota exceeded"))
fmt.Println(ctx.Err())          // context canceled
fmt.Println(context.Cause(ctx)) // quota exceeded
```

`Err()` always returns one of the two sentinels; `Cause()` returns the actual error.

### `context.AfterFunc` (1.21+)

```go
stop := context.AfterFunc(ctx, func() {
	cleanup()
})
defer stop() // optional; cancels the registration if not yet fired
```

Runs `f` when `ctx` is cancelled. Useful for cleanup that should happen on cancel without spawning a goroutine.

### Values — used sparingly

```go
type ctxKey int
const userKey ctxKey = 1

ctx = context.WithValue(ctx, userKey, user)
u := ctx.Value(userKey).(User)
```

Use unexported types for keys to prevent collisions. Reserve `WithValue` for request-scoped opaque data (request ID, trace ID, tenant, user); never for optional function arguments.

### Always call `cancel`

```go
ctx, cancel := context.WithTimeout(parent, d)
defer cancel()
```

If you don't, the timer (or parent-watcher goroutine) leaks until the parent cancels. `go vet` (the `lostcancel` check) catches the common forms.

### `select` with `ctx.Done()`

```go
select {
case <-ctx.Done(): return ctx.Err()
case v := <-input:  process(v)
case <-time.After(d): handleTimeout()
}
```

`ctx.Done()` returning a channel makes context composable with any other select.

### Propagating to APIs

Every API that may block should take `ctx` as its first argument:

```go
func Get(ctx context.Context, id string) (*Item, error)
```

Inside, pass it through:

```go
row := db.QueryRowContext(ctx, query, id)
resp, err := client.Do(req.WithContext(ctx))
```

### `context.WithoutCancel` (1.21+)

```go
detached := context.WithoutCancel(parent)
go logSomeStuff(detached) // shouldn't be cancelled when parent is
```

Use to spawn audit/log work that must complete even after the request returns.

## Standard Library Hooks

- `net/http.Request.Context()` — request context.
- `net/http.Request.WithContext(ctx)` — replace context.
- `database/sql.QueryContext`, `ExecContext`, `PrepareContext`, `BeginTx(ctx, ...)`.
- `os/exec.CommandContext`.
- `net.Dialer.DialContext`, `net.Resolver.LookupHost(ctx, ...)`.
- `signal.NotifyContext` — context that cancels on signal.

## Real-World Patterns

### 1. Request timeout propagating to DB

```go
func (s *Service) Get(ctx context.Context, id string) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()
	return s.db.GetUser(ctx, id)
}

// db.GetUser internally calls QueryRowContext(ctx, ...) — the SQL driver
// cancels the in-flight query when ctx is done.
```

### 2. Goroutine that exits on cancel

```go
func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			return
		case j := <-jobs:
			handle(ctx, j)
		}
	}
}
```

### 3. `errgroup` with shared context

```go
import "golang.org/x/sync/errgroup"

g, gCtx := errgroup.WithContext(ctx)
for _, item := range items {
	item := item
	g.Go(func() error { return process(gCtx, item) })
}
return g.Wait() // cancels gCtx on first error
```

### 4. Cause-tracking cancellation

```go
ctx, cancel := context.WithCancelCause(ctx)
defer cancel(nil)

go func() {
	if err := watch(); err != nil {
		cancel(fmt.Errorf("watcher: %w", err))
	}
}()

<-ctx.Done()
return context.Cause(ctx) // real reason
```

### 5. AfterFunc for cleanup

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
stop := context.AfterFunc(ctx, func() {
	rollback()
})
defer stop()
return doWork(ctx)
```

If `doWork` cancels via timeout, `rollback` fires; if `doWork` returns successfully, `stop` cancels the registration.

## Anti-Patterns & Gotchas

**Storing `context.Context` in a struct.** Pass it as the first argument.

**`context.Background()` deep inside a request handler.** Loses cancellation propagation.

**Ignoring `cancel`.** Resource leak.

**Using `WithValue` for required parameters.** Hide dependencies; refactor to typed args.

**Untyped string keys for `WithValue`.** Collide across packages. Use unexported types.

**Passing `nil` context.** Always panics or breaks downstream.

**Context as a "global state" smuggle channel.** It's for request-scoped opacity, not for config.

**`ctx.Done()` polled without `select`.** Spin-loop. Always select.

**Forgetting `WithoutCancel` for background work that outlives the parent.** Spawning a goroutine with the request context leads to truncated work.

**`context.WithTimeout(ctx, 0)` thinking it disables.** It cancels immediately.

## Performance Notes

- Constructing a context: allocates a small struct (~80 bytes).
- `ctx.Done()` returns a channel; allocated lazily on first call.
- `ctx.Value` is a linked-list walk through ancestors; O(depth). Don't put hundreds of values.
- Cancellation propagation goes through goroutines waiting on the parent's channel — efficient.
- `AfterFunc` registers a small callback; no goroutine spawned unless the function fires.

## How Big Companies Use It

- **Kubernetes** APIs are context-first; controllers cancel via parent context on shutdown.
- **gRPC-Go** uses context everywhere: deadline metadata is propagated across RPC boundaries.
- **CockroachDB** uses `WithCancelCause` to attach SQL error codes to query cancellation.
- **Discord** uses `WithoutCancel` to detach analytics writes from request lifetime.
- **Tailscale** uses context for per-connection cancellation in their derp relay.

## Source Code References

Pinned to `go1.26`.

- `context`: [`src/context/context.go`](https://github.com/golang/go/blob/master/src/context/context.go).
- `WithCancelCause` / `Cause`: same file (1.20+).
- `AfterFunc` / `WithoutCancel`: same file (1.21+).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/context.
- Go blog, "Context": https://go.dev/blog/context.
- Sameer Ajmani, "Go Concurrency Patterns: Context" (2014).
- Sameer Ajmani, "context-aware APIs in Go".

## Exercises / Self-Check

1. Implement a function `WithRetry(ctx, fn, n)` that retries with backoff, respecting ctx cancellation.
2. Show that `context.WithCancelCause` lets you preserve the underlying error after Cancel.
3. Use `context.AfterFunc` to schedule a cleanup callback. Verify it fires on cancel and doesn't fire if stop is called.
4. Build a request middleware that puts a unique request ID in context; downstream handlers extract it.
5. Why is `time.AfterFunc(d, fn)` not enough — when do you need `context.AfterFunc(ctx, fn)`?
