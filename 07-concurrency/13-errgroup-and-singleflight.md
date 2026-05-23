# `errgroup` and `singleflight`

## TL;DR

`errgroup.Group` is `sync.WaitGroup` plus first-error capture and (with `WithContext`) cancellation propagation. It is the standard tool for "fan out N parallel jobs, fail fast on the first error." `singleflight.Group` deduplicates concurrent calls with the same key — N callers asking for the same expensive thing get one fetch and N shared results, eliminating cache stampedes. Both live in `golang.org/x/sync` and need `go get golang.org/x/sync`. The biggest gotcha: **errgroup's "limit" feature (`SetLimit`) is per-Group, not per-key**, and `Go` *panics* if you call it after `Wait` returns — design your call graph accordingly. For singleflight: the shared error in shared callers is the *same* error — wrap before logging if callers must distinguish.

## Mental Model

```
errgroup.Group
  Go(f func() error)   <- spawn; first f returning error cancels ctx (if WithContext)
  Wait() error          <- block until all return; yield first error
  SetLimit(n int)       <- cap concurrent Go calls (since 1.18)
  TryGo(f) bool         <- non-blocking spawn under SetLimit

singleflight.Group
  Do(key, fn) (v, err, shared bool)        <- block; first caller runs fn, others wait
  DoChan(key, fn) <-chan Result            <- async version
  Forget(key)                              <- remove cached in-flight entry

Behind the scenes: a map[key]*call where *call holds the WaitGroup and result.
The first arriver creates the call; subsequent arrivers attach. When fn returns,
all attached callers wake with the (shared) result.
```

## Syntax & Basic Usage

```bash
go get golang.org/x/sync
```

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"golang.org/x/sync/errgroup"
)

func main() {
	urls := []string{"https://go.dev", "https://example.com", "https://invalid.invalid"}

	g, ctx := errgroup.WithContext(context.Background())
	for _, u := range urls {
		g.Go(func() error {
			req, _ := http.NewRequestWithContext(ctx, "GET", u, nil)
			resp, err := http.DefaultClient.Do(req)
			if err != nil { return err }
			resp.Body.Close()
			return nil
		})
	}
	if err := g.Wait(); err != nil {
		fmt.Println("first error:", err)
	}
}
```

The `ctx` returned by `WithContext` is canceled the moment any `Go` callback returns a non-nil error (or panics). Other in-flight callbacks observe the cancellation and exit early.

```go
package main

import (
	"fmt"
	"sync"

	"golang.org/x/sync/singleflight"
)

func main() {
	var sf singleflight.Group
	var wg sync.WaitGroup
	for range 5 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			v, err, shared := sf.Do("user:42", func() (any, error) {
				return loadUser(42)
			})
			fmt.Println(v, err, shared)
		}()
	}
	wg.Wait()
}

func loadUser(int) (any, error) { return "ALICE", nil }
```

Five goroutines call `Do` with the same key; `loadUser` runs **once**; all five receive `"ALICE"`. `shared` is `true` for the four that piggybacked (and possibly the first, depending on timing).

## Deep Dive

### `errgroup.Group` lifecycle

```go
g := &errgroup.Group{}             // independent group
g, ctx := errgroup.WithContext(parent) // canceling group cancels ctx

g.Go(fn func() error)              // spawn
g.SetLimit(n)                      // max in-flight (since 1.18). -1 = unlimited.
g.TryGo(fn) bool                   // non-blocking spawn under SetLimit
err := g.Wait()                    // wait for all; return first error
```

`Go` calls `wg.Add(1)` and spawns a goroutine that calls `fn` and `wg.Done`. The first non-nil `err` is captured via `sync.Once`; if `WithContext` was used, that error also triggers `cancel()`.

`Wait` blocks on `wg.Wait`, then runs `cancel()` (idempotent) for cleanliness, and returns the captured error.

### `SetLimit` semantics

```go
g := &errgroup.Group{}
g.SetLimit(4) // at most 4 concurrent
```

After `SetLimit(n)`, `Go` blocks if `n` goroutines are already running. `TryGo` returns `false` instead of blocking. Set the limit **before** the first `Go` call (else panic in some versions; undefined in others). To disable, pass `-1`.

Use this when fan-out shouldn't overload an external dependency: 4 simultaneous DB queries, 10 concurrent HTTP fetches, etc.

### `errgroup.WithContext` and cancellation

```go
g, ctx := errgroup.WithContext(parent)
```

The returned `ctx` is canceled when:
- The first `Go` callback returns a non-nil error, OR
- `parent` is canceled.

All in-flight callbacks see `<-ctx.Done()`. If you write `g.Go(func() error { req, _ := http.NewRequestWithContext(ctx, ...); ... })`, the cancellation propagates into the HTTP request and unblocks it. **This is the whole point of `WithContext`** — without it, slow tasks keep running after a fast one fails.

### `errgroup` panics from `Go` are recovered (since 1.21+ in `x/sync`)

Old behavior: a panic in `Go` callback crashed the process. Recent `x/sync` versions recover the panic and re-throw it from `Wait` as a `PanicError`. Check the package version (`go list -m -versions golang.org/x/sync`).

### `errgroup` vs `sync.WaitGroup`

| Need                                | Use                  |
|-------------------------------------|----------------------|
| Just wait for N to finish, no error | `sync.WaitGroup.Go`  |
| Wait + capture first error          | `errgroup.Group`     |
| ... and cancel siblings on error    | `errgroup.WithContext` |
| ... and limit concurrency           | `errgroup.SetLimit`  |

`WaitGroup.Go` (Go 1.25+) covers the simplest case. `errgroup` covers everything else.

### `singleflight.Do` — request coalescing

```go
v, err, shared := sf.Do(key, fn)
```

- If no in-flight call exists for `key`, this caller runs `fn`. Others arriving meanwhile wait.
- All callers (the runner included) wake with the *same* `(v, err)`.
- `shared` is `true` if more than one caller participated.

After `fn` returns, the entry is removed from the in-flight map. Next caller starts fresh. So singleflight is *coalescing*, not *caching*. If you want a cache, layer one on top.

### `singleflight.DoChan`

```go
ch := sf.DoChan(key, fn)
select {
case res := <-ch:
	// res.Val, res.Err, res.Shared
case <-ctx.Done():
	// don't wait anymore; but fn keeps running for other waiters
}
```

`DoChan` lets you abandon a wait. The in-flight `fn` continues for other waiters; if there were no other waiters, you've still committed to running `fn` to completion. To cancel `fn` itself, `Forget(key)` and have `fn` honor a context — but the running call doesn't see the new context.

### `singleflight.Forget`

```go
sf.Forget("user:42")
```

Removes the in-flight entry so the next `Do` runs `fn` afresh. Use after a `Do` returned a stale or error result you don't want subsequent callers to receive (they wouldn't anyway — entry is removed when `fn` returns — but `Forget` is useful for the edge case where the entry is still in-flight due to a retry policy).

### Shared error semantics

```go
v, err, shared := sf.Do("k", expensive)
// If shared==true, err is the SAME error VALUE as every other caller saw.
```

This bites if callers do something like `err = errors.Wrap(err, "in handler X")` — they're wrapping the shared error. Logging the wrapped error per caller is fine; mutating it isn't (errors should be immutable, but custom error types sometimes aren't).

### Failures cache implications

Singleflight does **not** cache failures. If `fn` returns an error, every concurrent caller sees that error, but the *next* caller (after the call returns) gets a fresh attempt. This is good for transient errors, bad for genuinely-broken keys — you'll hammer the dependency until it stops failing. Layer your own negative-cache.

## Standard Library Hooks

- `golang.org/x/sync/errgroup`
- `golang.org/x/sync/singleflight`
- `golang.org/x/sync/semaphore` (the weighted-semaphore companion to bounded errgroup)
- `golang.org/x/sync/syncmap` (predecessor to `sync.Map`, kept as alias)
- `context.Context` — pair every errgroup with a Context.

## Real-World Patterns

### 1. Concurrent fetch with cancel on first error

```go
package main

import (
	"context"
	"encoding/json"
	"net/http"

	"golang.org/x/sync/errgroup"
)

type User struct{ ID int }
type Order struct{ ID int }

func loadDashboard(ctx context.Context, userID int) (*User, []Order, error) {
	g, ctx := errgroup.WithContext(ctx)
	var u User
	var orders []Order

	g.Go(func() error {
		req, _ := http.NewRequestWithContext(ctx, "GET", "/users/"+itoa(userID), nil)
		resp, err := http.DefaultClient.Do(req)
		if err != nil { return err }
		defer resp.Body.Close()
		return json.NewDecoder(resp.Body).Decode(&u)
	})
	g.Go(func() error {
		req, _ := http.NewRequestWithContext(ctx, "GET", "/orders?user="+itoa(userID), nil)
		resp, err := http.DefaultClient.Do(req)
		if err != nil { return err }
		defer resp.Body.Close()
		return json.NewDecoder(resp.Body).Decode(&orders)
	})

	if err := g.Wait(); err != nil { return nil, nil, err }
	return &u, orders, nil
}

func itoa(int) string { return "" }
```

Two parallel calls, fail fast. The slower call sees the context cancel as soon as the faster one fails.

### 2. Bounded concurrent batch

```go
func processAll(ctx context.Context, items []Item) error {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8) // at most 8 in flight
	for _, it := range items {
		g.Go(func() error {
			return processOne(ctx, it)
		})
	}
	return g.Wait()
}
func processOne(context.Context, Item) error { return nil }
type Item struct{}
```

`SetLimit(8)` makes `g.Go` block until a slot is free. Cleaner than a manual semaphore.

### 3. singleflight in front of a DB

```go
type Loader struct {
	db *sql.DB
	sf singleflight.Group
}

func (l *Loader) GetUser(ctx context.Context, id int) (*User, error) {
	key := fmt.Sprintf("user:%d", id)
	v, err, _ := l.sf.Do(key, func() (any, error) {
		var u User
		if err := l.db.QueryRowContext(ctx, "SELECT id FROM users WHERE id=?", id).Scan(&u.ID); err != nil {
			return nil, err
		}
		return &u, nil
	})
	if err != nil { return nil, err }
	return v.(*User), nil
}
```

If 100 requests for the same user arrive in the same millisecond, the DB sees 1 query. Adopted in nearly every Go service with a backing store.

### 4. Cache + singleflight (avoid stampede after eviction)

```go
func (c *Cache) Get(ctx context.Context, key string) ([]byte, error) {
	if v, ok := c.local.Get(key); ok {
		return v, nil
	}
	v, err, _ := c.sf.Do(key, func() (any, error) {
		return c.fetchAndStore(ctx, key)
	})
	if err != nil { return nil, err }
	return v.([]byte), nil
}
```

After a TTL eviction, hundreds of concurrent reads would each refetch. `singleflight.Do` collapses them into one. Classic anti-stampede pattern.

### 5. `DoChan` with timeout

```go
func (l *Loader) GetUserTimeout(ctx context.Context, id int) (*User, error) {
	key := "user:" + strconv.Itoa(id)
	ch := l.sf.DoChan(key, func() (any, error) { return loadFromDB(id) })
	select {
	case r := <-ch:
		if r.Err != nil { return nil, r.Err }
		return r.Val.(*User), nil
	case <-ctx.Done():
		// don't Forget(key) — let other waiters still get the result
		return nil, ctx.Err()
	}
}
func loadFromDB(int) (any, error) { return nil, nil }
```

Lets a slow caller bail without affecting other waiters.

## Anti-Patterns & Gotchas

**Calling `g.Go` after `g.Wait` returned.** Panics. Errgroup is a one-shot. Re-create for a new round.

**Using `errgroup.Group` (without WithContext) for fail-fast.** Other workers keep running until they finish; `Wait` returns the first error but doesn't cancel anything. Use `WithContext`.

**Forgetting that `errgroup.Go` doesn't take a context.** The callback closes over the `ctx` from `WithContext`. If you pass each callback a different context, you've defeated the purpose.

**Setting `SetLimit` after the first `Go`.** Undefined/panic. Set it once at construction.

**Singleflight with side effects in `fn`.** `fn` runs once for many callers. If it mutates the world per-call (e.g., increments a metric), the metric is wrong.

**Caching errors via singleflight.** It doesn't. Layer your own negative cache or expect retries.

**Using singleflight as a long-term cache.** It only deduplicates *in-flight* calls. After return, the entry is gone. Add a cache.

**Forgetting that `shared==true` callers receive the *same* mutable object.** If `fn` returns a `*Slice` and one caller appends, others see it. Return immutable values or clone.

**Letting one slow singleflight call block all callers indefinitely.** If `fn` hangs, every caller hangs. Always bound `fn` with a context and add a timeout in the caller via `DoChan`.

**Using `errgroup` to wait for goroutines that never error.** That's `sync.WaitGroup` (or `WaitGroup.Go` in 1.25+). Don't reach for errgroup unless you need error capture.

## Performance Notes

- `errgroup.Go` overhead: one atomic increment + goroutine spawn — sub-µs.
- `errgroup.Wait`: a single channel-close + `wg.Wait`.
- `singleflight.Do` first arriver: lock + map insert + `fn` call.
- `singleflight.Do` follower: lock + map lookup + park on shared WaitGroup. ~µs.
- `DoChan` adds one channel allocation per caller (not per key).
- The internal map in `singleflight.Group` is mutex-protected; for very high-key throughput (>1M ops/sec) shard the group by hash.
- `SetLimit` uses a buffered channel as a semaphore — same characteristics as the manual pattern.

## How Big Companies Use It

- **Kubernetes** uses `errgroup` in `client-go/util` and many controllers for parallel reconciles.
- **gRPC-Go** uses `errgroup` for stream coordination.
- **Vault** uses `singleflight` for token lookups so that bursts don't overwhelm the backend.
- **Caddy** uses `singleflight` for certificate issuance to coalesce ACME requests under load.
- **Buf (`buf.build`)** uses `errgroup.SetLimit` for bounded protoc parallelism.
- **CockroachDB** uses both heavily — `singleflight` for descriptor cache fills, `errgroup` for distributed table reads.
- **Original `singleflight`** was extracted from `groupcache` (Brad Fitzpatrick at Google), which itself was the inspiration for memcache/redis stampede prevention everywhere.

## Source Code References

Pinned to `golang.org/x/sync` latest.

- `errgroup`: [`errgroup/errgroup.go`](https://github.com/golang/sync/blob/master/errgroup/errgroup.go). Single small file.
- `singleflight`: [`singleflight/singleflight.go`](https://github.com/golang/sync/blob/master/singleflight/singleflight.go).
- `semaphore.Weighted`: [`semaphore/semaphore.go`](https://github.com/golang/sync/blob/master/semaphore/semaphore.go).
- Original `groupcache` `singleflight`: [`groupcache/singleflight`](https://github.com/golang/groupcache/tree/master/singleflight).

## Further Reading

- Proposal: `errgroup`: https://go.dev/issue/53757 (and earlier discussion threads)
- Proposal: `errgroup.SetLimit`: https://go.dev/issue/27837
- Brad Fitzpatrick, "groupcache" presentation: https://www.youtube.com/watch?v=eIxVqL7M8aA
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns" (covers errgroup): https://www.youtube.com/watch?v=5zXAHh5tJqQ
- Sameer Ajmani, "Go Concurrency Patterns": https://go.dev/talks/2012/concurrency.slide
- Filippo Valsorda on cache stampedes: https://words.filippo.io/
- "Stampede defense in Go" (Cloudflare): https://blog.cloudflare.com/.

## Exercises / Self-Check

1. Why does `errgroup.WithContext` matter? Construct a slow-task-vs-fast-fail scenario where the difference is observable.
2. Implement a poor-man's `singleflight` using `sync.Map` and `sync.WaitGroup`. What's the trickiest race?
3. Show how to combine `errgroup.SetLimit(N)` with `errgroup.WithContext` for bounded fail-fast fan-out.
4. Singleflight: why is "share error value" a footgun? Sketch a wrapper that wraps with per-caller context before returning.
5. Two callers `Do("k", fn)` simultaneously. `fn` panics. What does each caller see? What is the group's state after?
