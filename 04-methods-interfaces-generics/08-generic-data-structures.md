# Generic Data Structures

## TL;DR

Pre-1.18 Go had two ways to write a "container of T": ship a code-generated package per type, or use `any` and box everything. Generics let you write the structure once, in full type safety, with no boxing for the element type. The cost is a small dictionary-driven indirection. Three canonical examples worth knowing by heart: **Stack**, **Ring Buffer**, **LRU Cache**. Each demonstrates a different concern (mutation, fixed capacity, hash + linked-list combo).

## Mental Model

```
type Container[T any] struct { /* private state */ }

Methods cannot add new type parameters; they inherit T from the container.
Storage layout: same as the equivalent non-generic version with T substituted.
Operations: shape-shared bodies; method dispatch uses the receiver's dictionary.
```

A generic data structure is "the non-generic version with the type filled in." Mentally substitute the type parameter and the code reads as if you'd written it for `int` or `string`.

## Syntax & Basic Usage

We'll build three production-style data structures below. They share the same skeleton:

```go
package ds

type Container[T any] struct{ /* state */ }

func New[T any](args ...any) *Container[T] { /* ... */ return nil }

func (c *Container[T]) Op() { /* ... */ }
```

## Deep Dive — Three Reference Implementations

### 1. Stack

```go
package ds

// Stack is a generic LIFO stack.
type Stack[T any] struct{ items []T }

// New returns an empty stack.
func NewStack[T any](initialCap int) *Stack[T] {
	return &Stack[T]{items: make([]T, 0, initialCap)}
}

// Len returns the number of items currently on the stack.
func (s *Stack[T]) Len() int { return len(s.items) }

// Push adds v to the top.
func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

// Pop removes and returns the top item. Returns the zero value and false if empty.
func (s *Stack[T]) Pop() (T, bool) {
	n := len(s.items)
	if n == 0 {
		var zero T
		return zero, false
	}
	v := s.items[n-1]
	var zero T
	s.items[n-1] = zero  // critical: zero the slot to allow GC
	s.items = s.items[:n-1]
	return v, true
}

// Peek returns the top item without removing it.
func (s *Stack[T]) Peek() (T, bool) {
	if len(s.items) == 0 {
		var zero T
		return zero, false
	}
	return s.items[len(s.items)-1], true
}
```

Key detail: **zeroing the popped slot**. Without it, when `T` contains pointers, the removed value remains reachable through the backing array — a subtle memory leak. The slice header shrinks but the array doesn't.

Usage:

```go
package main

import "fmt"

func main() {
	s := NewStack[int](4)
	s.Push(1); s.Push(2); s.Push(3)
	for {
		v, ok := s.Pop()
		if !ok { break }
		fmt.Println(v)
	}
	// Output:
	// 3
	// 2
	// 1
}
```

### 2. Ring Buffer (fixed-capacity, lock-free single-producer single-consumer is harder; here: simple mutexed)

```go
package ds

import "sync"

// RingBuffer is a fixed-capacity FIFO. Overflow drops the oldest entries.
type RingBuffer[T any] struct {
	mu    sync.Mutex
	buf   []T
	head  int // next slot to write
	tail  int // next slot to read
	count int // number of valid items
}

func NewRingBuffer[T any](cap int) *RingBuffer[T] {
	if cap <= 0 {
		panic("ring buffer: capacity must be positive")
	}
	return &RingBuffer[T]{buf: make([]T, cap)}
}

func (r *RingBuffer[T]) Push(v T) (dropped T, evicted bool) {
	r.mu.Lock()
	defer r.mu.Unlock()
	if r.count == len(r.buf) {
		// Full: evict oldest.
		dropped = r.buf[r.tail]
		r.tail = (r.tail + 1) % len(r.buf)
		evicted = true
		r.count--
	}
	r.buf[r.head] = v
	r.head = (r.head + 1) % len(r.buf)
	r.count++
	return
}

func (r *RingBuffer[T]) Pop() (T, bool) {
	r.mu.Lock()
	defer r.mu.Unlock()
	if r.count == 0 {
		var zero T
		return zero, false
	}
	v := r.buf[r.tail]
	var zero T
	r.buf[r.tail] = zero
	r.tail = (r.tail + 1) % len(r.buf)
	r.count--
	return v, true
}

func (r *RingBuffer[T]) Len() int {
	r.mu.Lock()
	defer r.mu.Unlock()
	return r.count
}
```

Why a ring buffer? Bounded memory regardless of producer rate, no allocation on push after construction, friendly to cache. Real-world example: in-process telemetry buffers, log batching, retry queues.

### 3. LRU Cache (map + doubly linked list)

```go
package ds

// LRU is a generic least-recently-used cache.
// Implemented as a hash map for O(1) lookup plus a doubly linked list
// for O(1) move-to-front and eviction-from-tail.
type LRU[K comparable, V any] struct {
	cap  int
	m    map[K]*lruNode[K, V]
	head *lruNode[K, V] // most recently used
	tail *lruNode[K, V] // least recently used
}

type lruNode[K comparable, V any] struct {
	k          K
	v          V
	prev, next *lruNode[K, V]
}

func NewLRU[K comparable, V any](cap int) *LRU[K, V] {
	if cap <= 0 {
		panic("lru: capacity must be positive")
	}
	return &LRU[K, V]{
		cap: cap,
		m:   make(map[K]*lruNode[K, V], cap),
	}
}

// Get returns the value for k and bumps it to most-recently-used.
func (c *LRU[K, V]) Get(k K) (V, bool) {
	n, ok := c.m[k]
	if !ok {
		var zero V
		return zero, false
	}
	c.moveToFront(n)
	return n.v, true
}

// Put inserts or updates k -> v. Evicts the LRU entry if over capacity.
func (c *LRU[K, V]) Put(k K, v V) {
	if n, ok := c.m[k]; ok {
		n.v = v
		c.moveToFront(n)
		return
	}
	n := &lruNode[K, V]{k: k, v: v}
	c.m[k] = n
	c.pushFront(n)
	if len(c.m) > c.cap {
		c.evictTail()
	}
}

func (c *LRU[K, V]) Len() int { return len(c.m) }

// --- helpers ---

func (c *LRU[K, V]) pushFront(n *lruNode[K, V]) {
	n.prev = nil
	n.next = c.head
	if c.head != nil {
		c.head.prev = n
	}
	c.head = n
	if c.tail == nil {
		c.tail = n
	}
}

func (c *LRU[K, V]) detach(n *lruNode[K, V]) {
	if n.prev != nil {
		n.prev.next = n.next
	} else {
		c.head = n.next
	}
	if n.next != nil {
		n.next.prev = n.prev
	} else {
		c.tail = n.prev
	}
	n.prev, n.next = nil, nil
}

func (c *LRU[K, V]) moveToFront(n *lruNode[K, V]) {
	if c.head == n {
		return
	}
	c.detach(n)
	c.pushFront(n)
}

func (c *LRU[K, V]) evictTail() {
	if c.tail == nil {
		return
	}
	t := c.tail
	c.detach(t)
	delete(c.m, t.k)
}
```

Usage:

```go
cache := NewLRU[string, []byte](1024)
cache.Put("k1", []byte("v1"))
if v, ok := cache.Get("k1"); ok {
	_ = v
}
```

Not goroutine-safe by design — wrap in `sync.Mutex` or `sync.RWMutex` if needed. Adding the mutex turns every Get into a write (because of `moveToFront`), so prefer `Mutex` over `RWMutex` for LRUs.

## Standard Library Hooks

- `container/heap` — generic-ish (uses `heap.Interface`, predates type parameters). Build a typed heap by wrapping.
- `container/list` — doubly linked list using `any`. Useful template; consider rolling your own typed version for performance-critical code.
- `slices`, `maps` — generic helpers for the most common operations.
- `sync.Pool[T]` does **not** exist in stdlib (`sync.Pool` uses `any`). Many projects build typed wrappers.
- `sync.OnceValue[T]`, `sync.OnceValues[T1, T2]` — typed lazy initialization (1.21+).
- `atomic.Pointer[T]`, `atomic.Int64`, `atomic.Bool` — typed atomic primitives (1.19+).
- `unique.Handle[T]` — handle-based interning of comparable values (1.23+).
- `iter.Seq[T]`, `iter.Seq2[K, V]` — generic iterator types (1.23+).

## Real-World Patterns

### 1. Typed `sync.Pool` wrapper

```go
type Pool[T any] struct {
	p   sync.Pool
	new func() *T
}

func NewPool[T any](new func() *T) *Pool[T] {
	return &Pool[T]{p: sync.Pool{New: func() any { return new() }}, new: new}
}

func (p *Pool[T]) Get() *T  { return p.p.Get().(*T) }
func (p *Pool[T]) Put(x *T) { p.p.Put(x) }
```

Removes the assertion on every `Get`; same runtime behavior.

### 2. Typed atomic value

```go
type Atomic[T any] struct{ p atomic.Pointer[T] }

func (a *Atomic[T]) Load() *T  { return a.p.Load() }
func (a *Atomic[T]) Store(v *T) { a.p.Store(v) }
func (a *Atomic[T]) Swap(v *T) *T { return a.p.Swap(v) }
```

For "copy-on-write" hot-reloadable config: load is a single atomic read.

### 3. Set with generic deduplication

```go
type Set[T comparable] map[T]struct{}

func NewSet[T comparable](items ...T) Set[T] {
	s := make(Set[T], len(items))
	for _, it := range items { s[it] = struct{}{} }
	return s
}

func (s Set[T]) Add(v T)          { s[v] = struct{}{} }
func (s Set[T]) Has(v T) bool     { _, ok := s[v]; return ok }
func (s Set[T]) Delete(v T)       { delete(s, v) }

func (s Set[T]) Union(o Set[T]) Set[T] {
	out := make(Set[T], len(s)+len(o))
	for v := range s { out[v] = struct{}{} }
	for v := range o { out[v] = struct{}{} }
	return out
}
```

### 4. Priority queue (typed heap)

```go
import "container/heap"

type Item[T any] struct {
	Value    T
	Priority int
	index    int
}

type pq[T any] []*Item[T]

func (q pq[T]) Len() int            { return len(q) }
func (q pq[T]) Less(i, j int) bool  { return q[i].Priority < q[j].Priority }
func (q pq[T]) Swap(i, j int)       { q[i], q[j] = q[j], q[i]; q[i].index = i; q[j].index = j }
func (q *pq[T]) Push(x any)         { *q = append(*q, x.(*Item[T])) }
func (q *pq[T]) Pop() any           { old := *q; n := len(old); x := old[n-1]; *q = old[:n-1]; return x }

type PriorityQueue[T any] struct{ h pq[T] }

func (p *PriorityQueue[T]) Push(v T, priority int) {
	heap.Push(&p.h, &Item[T]{Value: v, Priority: priority})
}

func (p *PriorityQueue[T]) Pop() (T, bool) {
	if p.h.Len() == 0 { var z T; return z, false }
	it := heap.Pop(&p.h).(*Item[T])
	return it.Value, true
}
```

The `container/heap` API still uses `any` for `Push`/`Pop`, so you box once per operation. Acceptable for almost all use cases.

### 5. Deque (double-ended queue)

```go
type Deque[T any] struct {
	buf  []T
	head int
	tail int
	size int
}

func NewDeque[T any]() *Deque[T] { return &Deque[T]{buf: make([]T, 8)} }

func (d *Deque[T]) Len() int { return d.size }

func (d *Deque[T]) PushBack(v T) {
	if d.size == len(d.buf) { d.resize() }
	d.buf[d.tail] = v
	d.tail = (d.tail + 1) % len(d.buf)
	d.size++
}

func (d *Deque[T]) PopFront() (T, bool) {
	if d.size == 0 { var z T; return z, false }
	v := d.buf[d.head]
	var z T
	d.buf[d.head] = z
	d.head = (d.head + 1) % len(d.buf)
	d.size--
	return v, true
}

func (d *Deque[T]) resize() {
	n := len(d.buf) * 2
	newBuf := make([]T, n)
	for i := 0; i < d.size; i++ {
		newBuf[i] = d.buf[(d.head+i)%len(d.buf)]
	}
	d.buf = newBuf
	d.head, d.tail = 0, d.size
}
```

## Anti-Patterns & Gotchas

**Forgetting to zero popped slots when `T` contains pointers.** Leak. Pop from slice-based structures must reset the slot.

**Storing `T` instead of `*T` for large structs.** Each container slot holds the full struct; copies are expensive. Consider `*T` for large types.

**Locking inside every operation of a structure that's used in a single goroutine.** Unnecessary contention. Provide unlocked and locked variants, or document the threading model.

**`RWMutex` on an LRU cache.** Reads modify the recency order, so every "read" is a write. Use `Mutex`.

**Constraining `T comparable` when the structure doesn't need equality.** Over-constrains callers. Only require what you actually use.

**Building a "universal" container library.** Most teams reinvent Stack, Queue, Heap a few times — each tailored. The standard library deliberately keeps generics narrow for this reason.

**Methods that try to add type parameters.** Not possible. Refactor into package-level functions.

**Heavy use of pointer chasing in linked structures.** GC scans every pointer; an LRU of 1M entries with `*node` chains is hostile to the GC. Profile before scaling up.

## Performance Notes

- Generic containers are roughly as fast as the equivalent hand-written `[]T` / `map[K]V` based structures. The dictionary overhead is small.
- Memory layout for `[]T` inside `Container[T any]` is identical to a non-generic version's. No header bloat.
- Linked-list-style structures (LRU, doubly linked list) are pointer-heavy and harder on the GC. Where possible, use a slab/index-based representation backed by a single `[]node` slice; node "pointers" become indices.
- Map-based structures with `K = string` pay the string-hashing cost; for hot indexes consider `K = uint64` (e.g., FNV-hashed keys upstream).
- Each generic instantiation is a separate "shape group" body; small numbers of distinct instantiations stay cheap.

## How Big Companies Use It

- **HashiCorp's `golang-lru`** (https://github.com/hashicorp/golang-lru/tree/master/v2) is the canonical production-grade generic LRU. Read its `v2` API — it predates `unique.Handle` but illustrates the patterns.
- **Cockroach's `util/ring`** ring buffer types use generics in newer code; older parts use code generation.
- **Tailscale's `syncs` package** provides typed atomic and locked containers: https://github.com/tailscale/tailscale/tree/main/syncs.
- **`scylladb/go-set`** is a popular typed set library (https://github.com/scylladb/go-set), originally code-generated, now superseded by generics.
- **`elastic/go-freelru`** is a high-performance generic LRU using a single backing slice + map of indices — the slab-backed pattern.

## Source Code References

Pinned to `go1.26`.

- `slices.SortFunc` showing generic algorithm + dictionary pattern: [`src/slices/sort.go`](https://github.com/golang/go/blob/master/src/slices/sort.go).
- `sync.OnceValue` / `OnceValues`: [`src/sync/oncefunc.go`](https://github.com/golang/go/blob/master/src/sync/oncefunc.go).
- `atomic.Pointer[T]`: [`src/sync/atomic/type.go`](https://github.com/golang/go/blob/master/src/sync/atomic/type.go).
- `container/heap` (non-generic): [`src/container/heap/heap.go`](https://github.com/golang/go/blob/master/src/container/heap/heap.go).
- `container/list` (non-generic): [`src/container/list/list.go`](https://github.com/golang/go/blob/master/src/container/list/list.go).
- `hashicorp/golang-lru/v2`: https://github.com/hashicorp/golang-lru/tree/master/v2.
- `elastic/go-freelru`: https://github.com/elastic/go-freelru.

## Further Reading

- Go blog, "An Introduction To Generics" — sections on generic types: https://go.dev/blog/intro-generics
- "Generics and collections in Go" (talks, search): https://www.youtube.com/results?search_query=go+generics+collections
- Donovan & Kernighan, "The Go Programming Language" — Chapter 6 patterns hold up well; add generics to taste.
- HashiCorp `golang-lru` README: https://github.com/hashicorp/golang-lru
- "Designing Generic Data Structures in Go" (community write-ups, search go.dev/blog and dev.to)

## Exercises / Self-Check

1. Add a `Snapshot()` method to `Stack[T]` returning a defensive copy. Why is `slices.Clone` exactly what you want?
2. Make `LRU[K, V]` goroutine-safe with the minimum number of locks. Why does `RWMutex` not help?
3. Implement `Set[T comparable].Intersect(other Set[T]) Set[T]`. Pre-size the result correctly to avoid rehashing.
4. Convert the linked-list-based LRU to a slab-based one (single `[]node`, indices instead of pointers). Compare GC behavior with `runtime.GC` + `MemStats`.
5. Write a generic `Bounded[T]` that wraps a `chan T` with `Push(ctx, T) error` and `Pop(ctx) (T, error)` semantics. Use `context.Context` to time out blocked operations.
