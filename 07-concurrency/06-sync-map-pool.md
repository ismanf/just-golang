# `sync.Map` and `sync.Pool`

## TL;DR

`sync.Map` is a concurrent map specialized for two workloads: (a) keys written once and read many times, and (b) disjoint key sets per goroutine. For everything else, `map[K]V + sync.Mutex` is faster and clearer. `sync.Pool` is a per-P free list for *temporary* objects — the GC can drain it on any cycle, so it's a hint, not a guarantee. The biggest gotchas: `sync.Map` is **not generic** (key/value are `any`), and `sync.Pool` items can vanish at any time, so never assume what's in there.

## Mental Model

```
sync.Map
+----------------------+      +----------------------+
| read (atomic.Pointer | ---> | readOnly             |  fast read path
|       to readOnly)   |      |   m: map[any]*entry  |  (no lock)
| dirty map[any]*entry |      |   amended: bool      |
| misses int           |      +----------------------+
| mu     Mutex         |
+----------------------+
                              dirty map: holds new keys not yet promoted.
                              On enough misses, dirty is promoted to read.

sync.Pool
+-------------------------------+
| local []poolLocalInternal     |  // one entry per P
|   private any                 |  // P's private slot (no atomics)
|   shared  poolChain           |  // double-ended queue, work-stealable
| New func() any                |  // factory
+-------------------------------+
Get: try private → shared → steal from other Ps → call New.
Put: prefer private slot → push to shared.
At every GC, the runtime drains all locals into a "victim cache",
and the previous victim cache is freed.
```

## Syntax & Basic Usage

### sync.Map

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map
	m.Store("a", 1)
	m.Store("b", 2)

	if v, ok := m.Load("a"); ok {
		fmt.Println(v)
	}

	m.Range(func(k, v any) bool {
		fmt.Println(k, v)
		return true // continue
	})
	// Output (order non-deterministic):
	// 1
	// a 1
	// b 2
}
```

The `func(k, v any) bool` return false to stop iterating. `sync.Map.Range` does **not** lock the whole map — it takes a snapshot of the read map and iterates that, plus any dirty entries; concurrent updates may or may not be visible.

### sync.Pool

```go
package main

import (
	"bytes"
	"fmt"
	"sync"
)

var bufPool = sync.Pool{
	New: func() any { return new(bytes.Buffer) },
}

func main() {
	buf := bufPool.Get().(*bytes.Buffer)
	defer bufPool.Put(buf)
	buf.Reset()                  // ALWAYS reset before use
	buf.WriteString("hello")
	fmt.Println(buf.String())
}
```

Every `Get` may return a fresh-from-`New` object or a recycled one; you can't tell. Always reset state before use.

## Deep Dive

### sync.Map — two-map design

The data structure is two maps:

- **read** (immutable, swap-in via `atomic.Pointer`): the fast-path. Reads under `read` are lock-free.
- **dirty** (mutable, mutex-protected): holds entries not yet in `read`, plus deletions in progress.

The misses counter increments every time a read falls through to `dirty`. After enough misses, the runtime "promotes" `dirty` to a new `read` (a single atomic swap), and a fresh empty `dirty` starts collecting new keys.

This shape is excellent when:
- The key set stabilizes quickly (mostly reads).
- Different goroutines mostly read different keys (no false sharing on a hot key).

It's bad when:
- Keys churn rapidly — promotion is amortized but each promotion costs O(n).
- Writes dominate — every write at minimum locks `mu`.
- Iteration is frequent — `Range` is not free.

### `sync.Map` API in full

```go
m.Store(key, value)           // unconditional set
v, ok := m.Load(key)
v, loaded := m.LoadOrStore(k, v) // atomic "get or set"
v, loaded := m.LoadAndDelete(k)  // atomic "get and delete"
m.Delete(k)
m.Range(func(k, v any) bool { return true })

// since Go 1.20:
swapped := m.CompareAndSwap(k, old, new)
deleted := m.CompareAndDelete(k, old)
m.Swap(k, v)                  // returns previous value
```

These atomic compound ops let you build lock-free state machines without falling back to a Mutex.

### `sync.Map` and `any` (no generics)

`sync.Map`'s API is `any`-typed. This was a deliberate decision; making it generic would require an incompatible API change. In practice, wrap it:

```go
type StringIntMap struct{ m sync.Map }

func (s *StringIntMap) Store(k string, v int) { s.m.Store(k, v) }
func (s *StringIntMap) Load(k string) (int, bool) {
	v, ok := s.m.Load(k)
	if !ok { return 0, false }
	return v.(int), true
}
```

The boxing overhead (allocating an `int` into an `any`) usually dominates the lock-free win for small values. **Always benchmark `sync.Map` vs `map+RWMutex` for your workload.** The Go team's docs explicitly say: "Most code should use a plain Go map ... with separate locking ... and only consider sync.Map after measurement."

### sync.Pool — per-P locals + GC interaction

A pool maintains one `poolLocal` per P. Each `poolLocal` has:
- A `private` slot (one item, no atomics — only the owning P touches it).
- A `shared` deque, which other Ps can steal from when their own private/shared are empty.

`Get`:
1. Look in this P's private slot.
2. If empty, pop from this P's shared deque.
3. If empty, steal from another P's shared deque.
4. If still empty, call `New` (or return nil if `New` is nil).

`Put`:
1. If this P's private slot is empty, place there.
2. Otherwise push to shared.

**At every GC cycle**, the runtime drains all `poolLocal`s into a "victim cache". The previous victim cache is discarded. So an item survives at most 2 GC cycles. This bounds memory use but means you cannot rely on the pool for long-term caching.

### Why pool drains on GC

`sync.Pool` is designed for transient allocations — `*bytes.Buffer`, `*[]byte` slabs, `*json.Encoder`. The GC drain prevents an unbounded steady-state pool from looking like a memory leak. The 2-cycle victim window is an empirical compromise: long enough that a busy server hits the cache most of the time, short enough that a quiescent server reclaims memory.

### Pool — sizing reset, never grow without bound

A common mistake: putting back oversized buffers that then get reused for tiny jobs, holding huge backing arrays alive.

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func writeReply(w io.Writer, msg string) {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	buf.WriteString(msg)
	buf.WriteTo(w)
	if buf.Cap() > 64*1024 { // don't pool huge buffers
		return
	}
	bufPool.Put(buf)
}
```

This pattern (drop oversized) is standard in `net/http`'s response writer pool, `fmt`'s scratch pool, etc.

### Pool — never store pointers in pooled objects across uses

```go
type Worker struct{ conn *net.Conn }

var pool = sync.Pool{New: func() any { return &Worker{} }}
```

Don't. The next `Get` will hand the pooled `Worker` with the **previous** `conn` still attached — pinning a closed connection, or worse, leaking it. Reset every field before `Put`, or zero the whole struct.

## Standard Library Hooks

- `sync.Map` (since 1.9; CAS/Swap additions in 1.20).
- `sync.Pool` (since 1.3).
- `bytes.Buffer` paired with `sync.Pool` is the canonical example.
- `runtime.GC()` will drain the pool — useful in tests to expose lifetime bugs.
- `runtime/debug.SetGCPercent(-1)` disables GC; useful to inspect pool behavior without churn.
- `golang.org/x/sync/syncmap` was the predecessor and is now an alias.

## Real-World Patterns

### 1. Pooled scratch buffer

```go
package main

import (
	"bytes"
	"io"
	"sync"
)

var bufPool = sync.Pool{
	New: func() any { return &bytes.Buffer{} },
}

func gzipFile(dst io.Writer, src io.Reader) error {
	buf := bufPool.Get().(*bytes.Buffer)
	defer func() {
		if buf.Cap() <= 1<<20 { // 1 MiB threshold
			buf.Reset()
			bufPool.Put(buf)
		}
	}()
	if _, err := buf.ReadFrom(src); err != nil {
		return err
	}
	// ... gzip(buf, dst) ...
	return nil
}
```

The threshold prevents one giant request from poisoning the pool with multi-MiB buffers.

### 2. fmt-style printer

```go
package myfmt

import (
	"fmt"
	"strings"
	"sync"
)

type printer struct{ b strings.Builder }

var printerPool = sync.Pool{New: func() any { return &printer{} }}

func Sprintf(format string, args ...any) string {
	p := printerPool.Get().(*printer)
	defer func() { p.b.Reset(); printerPool.Put(p) }()
	fmt.Fprintf(&p.b, format, args...)
	return p.b.String() // String() copies; safe to recycle after
}
```

Matches the structure inside the real `fmt` package — see `src/fmt/print.go`.

### 3. sync.Map for connection-by-id

```go
type Server struct {
	conns sync.Map // map[id]*Conn
}

func (s *Server) Add(c *Conn)         { s.conns.Store(c.id, c) }
func (s *Server) Remove(id string)    { s.conns.Delete(id) }
func (s *Server) Lookup(id string) *Conn {
	v, ok := s.conns.Load(id)
	if !ok { return nil }
	return v.(*Conn)
}
```

Good `sync.Map` workload: each connection's goroutine reads its own entry (no contention on any one key), and additions are infrequent compared to lookups.

### 4. LoadOrStore for singletons-by-key

```go
var instances sync.Map // map[string]*Instance

func Get(name string) *Instance {
	if v, ok := instances.Load(name); ok {
		return v.(*Instance)
	}
	v, _ := instances.LoadOrStore(name, newInstance(name))
	return v.(*Instance)
}
```

Note this constructs `newInstance(name)` even if another goroutine wins the race. If construction is expensive, prefer `singleflight.Group` (see `13-errgroup-and-singleflight.md`).

### 5. CompareAndSwap for state machines (1.20+)

```go
type State int

const (
	Idle State = iota
	Running
	Stopped
)

var states sync.Map // map[string]State

func MarkRunning(id string) bool {
	return states.CompareAndSwap(id, Idle, Running)
}
```

Lock-free state transitions. The CAS replaced manual "Load, check, Store" sequences which were inherently racy.

## Anti-Patterns & Gotchas

**Using `sync.Map` reflexively.** It is slower than `map + Mutex` for the common write-heavy or balanced workload. The docs say so explicitly. Benchmark.

**Iterating with `sync.Map.Range` and expecting a snapshot.** It's "best effort." Concurrent writers may or may not show up. If you need a consistent snapshot, take a lock and copy out.

**Not resetting a pooled object.** Next user sees stale state. Reset on `Get` or right before `Put`.

**Putting back over-large objects.** Pool bloat. Cap with a threshold.

**Holding pool objects across goroutines or beyond a single request.** The pool may drain them; the next `Get` may hand the same object to someone else.

**Storing pointers to pooled objects in long-lived structures.** The object can be `Put` back while still referenced — race city.

**Calling `Put(nil)`.** Panics (`sync: Pool.Put with nil value` in some versions; silently broken in others). Don't.

**Assuming a pooled object is always recycled.** Under load spikes, every `Get` may call `New`. Pool is an optimization, not a guarantee.

**Storing comparable-but-not-hashable types as `sync.Map` keys.** Slices/maps/funcs are not comparable; they panic at runtime when used as a key.

**Using `sync.Map` for ordered iteration.** It has no order, and `Range` won't give you one. Use a different structure.

## Performance Notes

- `sync.Map.Load` hit on `read`: ~10–20 ns (single atomic load + map lookup).
- `sync.Map.Load` miss (fall through to dirty): ~50–100 ns plus mutex.
- `sync.Map.Store` of an existing key in `read`: ~10 ns (CAS on entry).
- `sync.Map.Store` of a new key: ~100 ns plus possible promotion cost.
- `sync.Pool.Get` hit on private slot: ~5–10 ns. With stealing: ~50 ns.
- `sync.Pool.Put`: ~5 ns.
- Boxing cost for `sync.Map`: a non-pointer value type may allocate on every `Store`. Pointer-valued maps avoid this.
- `sync.Pool` removes ~80% of allocations from `bytes.Buffer`-heavy paths — but if your buffers are tiny (~16 bytes), the pool overhead can be a wash.

## How Big Companies Use It

- **`net/http`** uses `sync.Pool` for `*Response`, `*Request.Form`, header maps, and TLS scratch buffers.
- **`fmt`** uses `sync.Pool` for `*pp` (printer state). Read [`src/fmt/print.go`](https://github.com/golang/go/blob/master/src/fmt/print.go).
- **`encoding/json`** uses `sync.Pool` for encoder/decoder state.
- **`gRPC-Go`** uses `sync.Pool` for buffer reuse in the framer.
- **Kubernetes `apiserver`** uses both `sync.Map` (for connection tracking) and `sync.Pool` (for proto buffers).
- **CockroachDB** uses `sync.Pool` heavily in `pkg/util/encoding` for binary marshal scratch space.
- **Dgraph's badger** uses `sync.Pool` for value-log scratch buffers.

## Source Code References

Pinned to `go1.26`.

- `sync.Map`: [`src/sync/map.go`](https://github.com/golang/go/blob/master/src/sync/map.go). The file is heavily commented — read top to bottom.
- `sync.Pool`: [`src/sync/pool.go`](https://github.com/golang/go/blob/master/src/sync/pool.go), with `poolChain` in [`src/sync/poolqueue.go`](https://github.com/golang/go/blob/master/src/sync/poolqueue.go).
- Runtime hooks for pool GC drain: [`src/runtime/mgc.go`](https://github.com/golang/go/blob/master/src/runtime/mgc.go), search `poolcleanup`.
- `sync.Map.CompareAndSwap` proposal: https://go.dev/issue/51972 (added in 1.20).
- Example real-world usage of `sync.Pool`: [`src/fmt/print.go`](https://github.com/golang/go/blob/master/src/fmt/print.go) — `ppFree`.

## Further Reading

- Go blog, "Behind the scenes of sync.Pool" (Bryan C. Mills, indirectly): https://go.dev/blog/
- Original `sync.Map` design doc (Bryan C. Mills): https://docs.google.com/document/d/1tNX5BzKZP-9bMc-1G6lpdYy7CTXNu5wY-vCk1Vyj4Pk/
- Russ Cox on `sync.Pool` and the victim cache: https://research.swtch.com/
- Damian Gryski, "sync.Map vs map+Mutex benchmarks": https://github.com/dgryski/go-perfbook
- Dmitry Vyukov, "Scalable Go concurrent map design": commit history of `sync.Map`.
- Filippo Valsorda, "sync.Pool in detail": https://words.filippo.io/

## Exercises / Self-Check

1. When does `sync.Map` beat `map[K]V + sync.RWMutex`? Construct a workload where each wins.
2. Why does `sync.Pool` drain at every GC? What does the victim cache add?
3. Write a generic `Map[K comparable, V any]` wrapper over `sync.Map`. What's the overhead?
4. Profile a fmt.Sprintf-heavy program with and without `sync.Pool` for the printer state. How does allocs/op change?
5. Why might `LoadOrStore(k, newExpensive())` waste work compared to `singleflight.Do`?
