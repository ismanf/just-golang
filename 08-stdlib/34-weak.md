# `weak` — Weak Pointers (since 1.24)

## TL;DR

`weak.Pointer[T]` holds a reference that does NOT prevent garbage collection. Call `.Value()` to get a strong `*T` (nil if the value was already collected). Use for caches and lookup tables that should let unused entries be reclaimed automatically — the canonical example being `unique.Make` itself (built on weak pointers).

## Mental Model

```
strong *T : reachability through this counts; GC keeps T alive
weak.Pointer[T] : does NOT count toward reachability; T can be collected
                   .Value() returns *T if still alive, nil if collected

Use case: associate metadata with an object without preventing collection.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"runtime"
	"weak"
)

func main() {
	x := new(int); *x = 42
	w := weak.Make(x)
	fmt.Println(*w.Value())  // 42

	x = nil
	runtime.GC()              // may or may not actually collect; depends
	runtime.GC()
	got := w.Value()
	fmt.Println(got == nil)   // true if collected
}
```

## Deep Dive

### `weak.Make` and `weak.Pointer[T]`

```go
type Pointer[T any] struct{ /* unexported */ }
func Make[T any](p *T) Pointer[T]
func (Pointer[T]) Value() *T
```

`Value()` returns the strong pointer if the object is still alive, nil otherwise.

### When weak pointers help

- **Caches** where entries can be regenerated. Once nobody else uses them, free the memory.
- **Object → metadata mappings.** E.g., `map[*Object]MetaData` keeps Objects alive forever; weak keys would let them die naturally.
- **Interning systems.** Exactly what `unique` does internally.
- **Cycle-breaking.** In certain back-pointer scenarios where strong references would create a leak.

### What weak pointers don't help with

- Manual memory management. Go has none; weak pointers don't change that.
- Avoiding pinning during cgo calls. Use `runtime.KeepAlive` and pointer rules instead.

### `weak` + finalizers / cleanup

A weak pointer plus `runtime.AddCleanup` (also 1.24) replaces most finalizer use cases:

```go
type cached struct{ data []byte }

func get(key string) *cached {
	if v := strongCache[key]; v != nil { return v }
	if w := weakCache[key]; w.Value() != nil { return w.Value() }
	v := load(key)
	weakCache[key] = weak.Make(v)
	runtime.AddCleanup(v, func(k string) { delete(weakCache, k) }, key)
	return v
}
```

When the loaded value's last strong reference goes away, cleanup runs and removes the weak entry from the cache.

### Caveat: GC unpredictability

The GC decides when to collect. Tests using `runtime.GC()` to force collection of a weak ref are flaky in CI — you usually need `runtime.GC()` twice plus a `runtime.Gosched()`. Don't write code whose correctness depends on weak collection timing.

### Comparison

| | strong *T | weak.Pointer[T] |
|---|-----------|-----------------|
| Keeps T alive | yes | no |
| Direct deref | `*p` | `w.Value(); *p` |
| Cost | none | small (header lookup) |

## Standard Library Hooks

- `runtime.AddCleanup` (1.24) — schedule code when an object is collected. Pairs naturally with weak.
- `unique` — built on top of `weak`.

## Real-World Patterns

### 1. Soft cache

```go
type Soft[K comparable, V any] struct {
	mu sync.Mutex
	m  map[K]weak.Pointer[V]
}

func (c *Soft[K, V]) Get(key K) (*V, bool) {
	c.mu.Lock(); defer c.mu.Unlock()
	w, ok := c.m[key]
	if !ok { return nil, false }
	v := w.Value()
	return v, v != nil
}

func (c *Soft[K, V]) Put(key K, v *V) {
	c.mu.Lock(); defer c.mu.Unlock()
	if c.m == nil { c.m = map[K]weak.Pointer[V]{} }
	c.m[key] = weak.Make(v)
	runtime.AddCleanup(v, func(k K) {
		c.mu.Lock(); delete(c.m, k); c.mu.Unlock()
	}, key)
}
```

Use case: result cache that doesn't bloat memory.

### 2. Object metadata side-table

```go
var debugInfo = map[weak.Pointer[Object]]Info{}

func register(o *Object, info Info) {
	debugInfo[weak.Make(o)] = info
}
```

If the Object is collected, the entry in `debugInfo` becomes orphaned (key.Value() is nil); reap periodically.

### 3. Build a tiny intern table

```go
var cache sync.Map // map[string]weak.Pointer[string]

func Intern(s string) *string {
	if w, ok := cache.Load(s); ok {
		if v := w.(weak.Pointer[string]).Value(); v != nil {
			return v
		}
	}
	p := &s
	cache.Store(s, weak.Make(p))
	return p
}
```

(Or just use `unique.Make`.)

### 4. Weak event listeners

```go
type Listener struct { /* ... */ }
type Pub struct {
	mu sync.Mutex
	listeners []weak.Pointer[Listener]
}
func (p *Pub) Subscribe(l *Listener) {
	p.mu.Lock(); p.listeners = append(p.listeners, weak.Make(l)); p.mu.Unlock()
}
func (p *Pub) Notify(ev Event) {
	p.mu.Lock(); defer p.mu.Unlock()
	live := p.listeners[:0]
	for _, w := range p.listeners {
		if v := w.Value(); v != nil {
			v.Handle(ev); live = append(live, w)
		}
	}
	p.listeners = live
}
```

Use case: avoid the classic "publisher keeps listeners alive" leak.

### 5. Cycle-aware tree node back-pointer

```go
type Node struct {
	Children []*Node
	Parent   weak.Pointer[Node]
}
```

Parent doesn't pin its subtree (children pin parent's *contents* via shared references, but the back-edge doesn't add reachability).

In Go this matters less than in languages with reference counting; the GC handles cycles. But weak parent links still make detached subtrees collectable as a whole.

## Anti-Patterns & Gotchas

**Treating `weak` as "free memory."** Doesn't free anything; just lets GC decide.

**Relying on weak collection timing.** GC schedule is not under your control.

**Forgetting to check `Value() != nil`.** Will deref a nil pointer.

**Using weak pointers for cgo memory.** Doesn't apply — C memory isn't GC-managed.

**Comparing `weak.Pointer` values with `==`.** Compares the small handle, not the underlying objects.

**Hand-rolling weak when `unique` would do.** For value interning, prefer `unique`.

## Performance Notes

- `weak.Make` is one allocation and a small global table update.
- `Value()` is one indirect load.
- Collected weak pointers leave stale entries until the application cleans them; use `AddCleanup` to do so.

## How Big Companies Use It

- **Stdlib `unique`** (1.23+) is built on `weak` — production validation right there.
- **Caches in distributed systems** are starting to adopt; expect uptake to grow.
- **gopls** considered weak for symbol tables; haven't seen final adoption yet.

## Source Code References

Pinned to `go1.26`.

- `weak`: [`src/weak/pointer.go`](https://github.com/golang/go/blob/master/src/weak/pointer.go).
- GC integration: [`src/runtime/mfinal.go`](https://github.com/golang/go/blob/master/src/runtime/mfinal.go).
- `unique` (consumer): [`src/unique/`](https://github.com/golang/go/tree/master/src/unique).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/weak.
- Go 1.24 release notes: https://go.dev/doc/go1.24.
- Proposal #67552 — weak pointers: https://github.com/golang/go/issues/67552.

## Exercises / Self-Check

1. Build a `Soft[K, V]` cache as above. Verify entries vanish after dropping strong refs + GC.
2. Implement a weak-listener pub/sub. Test that an unreferenced listener stops receiving events after GC.
3. Use `weak.Pointer[T]` to break a parent-back-reference cycle. Confirm subtree is collected when detached.
4. Why is `weak` paired so often with `runtime.AddCleanup`? Trace through a soft cache lifecycle.
5. Why isn't `weak` a replacement for explicit cache eviction in latency-sensitive code?
