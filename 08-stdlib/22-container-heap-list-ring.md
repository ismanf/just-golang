# `container/heap`, `container/list`, `container/ring`

## TL;DR

Three small data-structure packages from Go's pre-generics era. `container/heap` is a min-heap on an `heap.Interface` — useful for priority queues, top-K, and timer wheels. `container/list` is a doubly-linked list — almost never the right choice in modern Go (a slice is faster). `container/ring` is a circular list — niche, used for buffered streaming. With generics, you can write your own in 30 lines; community libraries (`gammazero/deque`, generic heaps) often beat the stdlib for ergonomics.

## Mental Model

```
heap.Interface = sort.Interface + Push(any) + Pop() any
     ├─ heap.Init(h)
     ├─ heap.Push(h, x)
     ├─ heap.Pop(h)    — returns the min (or max if you flip Less)
     └─ heap.Fix(h, i) — re-heapify after modifying h[i]

list.List: doubly-linked; PushBack/Front, MoveToBack, Remove
ring.Ring: circular doubly-linked; Next/Prev, Move(n), Do(fn)
```

## Syntax & Basic Usage

```go
package main

import (
	"container/heap"
	"fmt"
)

type IntHeap []int
func (h IntHeap) Len() int            { return len(h) }
func (h IntHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)         { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any           { old := *h; n := len(old); v := old[n-1]; *h = old[:n-1]; return v }

func main() {
	h := &IntHeap{5, 2, 8, 1}
	heap.Init(h)
	heap.Push(h, 3)
	for h.Len() > 0 {
		fmt.Println(heap.Pop(h))
	}
	// Output:
	// 1
	// 2
	// 3
	// 5
	// 8
}
```

## Deep Dive

### `container/heap`

The package gives you the algorithm; you bring the storage and the comparison. `heap.Push`/`heap.Pop` operate via your `Push`/`Pop` methods on the underlying slice.

Priority queue pattern:

```go
type Item struct{ Name string; Priority int; index int }
type PQ []*Item
func (pq PQ) Len() int               { return len(pq) }
func (pq PQ) Less(i, j int) bool     { return pq[i].Priority > pq[j].Priority } // max-heap
func (pq PQ) Swap(i, j int)          {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index, pq[j].index = i, j
}
func (pq *PQ) Push(x any) {
	n := len(*pq); item := x.(*Item); item.index = n; *pq = append(*pq, item)
}
func (pq *PQ) Pop() any {
	old := *pq; n := len(old); item := old[n-1]; old[n-1] = nil; *pq = old[:n-1]; return item
}
```

`Fix(h, i)` re-heapifies after you mutate `pq[i].Priority`.

`heap.Remove(h, i)` removes the item at index `i` in O(log n).

### `container/list`

```go
l := list.New()
e := l.PushBack(42)
l.PushFront(0)
for el := l.Front(); el != nil; el = el.Next() {
	fmt.Println(el.Value)
}
l.Remove(e)
```

Pros: O(1) insert/remove given an element.
Cons: Pointer chasing, cache-unfriendly, no random access, `Value any` boxes everything.

Modern alternative: `[]T` for most cases; `gammazero/deque` for double-ended.

### `container/ring`

```go
r := ring.New(5)
for i := 0; i < r.Len(); i++ {
	r.Value = i
	r = r.Next()
}
r.Do(func(v any) { fmt.Println(v) })
```

Useful for fixed-size circular buffers. Rarely needed.

## Standard Library Hooks

- `sort.Interface` — heap reuses `Less`/`Swap`/`Len`.
- `slices.Sort` — for one-shot sort instead of incremental heap.

## Real-World Patterns

### 1. Priority queue for scheduled tasks

```go
type Task struct { Run func(); At time.Time; idx int }
type TaskQueue []*Task
func (q TaskQueue) Len() int            { return len(q) }
func (q TaskQueue) Less(i, j int) bool  { return q[i].At.Before(q[j].At) }
func (q TaskQueue) Swap(i, j int)       { q[i], q[j] = q[j], q[i]; q[i].idx = i; q[j].idx = j }
func (q *TaskQueue) Push(x any)         { *q = append(*q, x.(*Task)); (*q)[len(*q)-1].idx = len(*q)-1 }
func (q *TaskQueue) Pop() any           { old := *q; n := len(old); t := old[n-1]; *q = old[:n-1]; return t }

// Scheduler loop:
for q.Len() > 0 {
	t := (*q)[0]
	d := time.Until(t.At)
	if d > 0 { time.Sleep(d) }
	heap.Pop(q)
	t.Run()
}
```

Use case: timer wheels, batch schedulers.

### 2. Top-K with min-heap

```go
// Keep the K largest items in a min-heap of size K.
h := &IntHeap{}
for _, x := range stream {
	if h.Len() < k {
		heap.Push(h, x)
	} else if x > (*h)[0] {
		(*h)[0] = x
		heap.Fix(h, 0)
	}
}
// (*h) now holds the top K
```

Use case: streaming analytics.

### 3. LRU cache with `list.List`

```go
type LRU struct {
	cap int
	m   map[string]*list.Element
	l   *list.List
}

func (c *LRU) Get(k string) (any, bool) {
	if e, ok := c.m[k]; ok {
		c.l.MoveToFront(e)
		return e.Value, true
	}
	return nil, false
}

func (c *LRU) Put(k string, v any) {
	if e, ok := c.m[k]; ok {
		e.Value = v; c.l.MoveToFront(e); return
	}
	c.m[k] = c.l.PushFront(v)
	if c.l.Len() > c.cap {
		old := c.l.Back(); c.l.Remove(old)
		// remove from map by tracking key inside element value
	}
}
```

For production, use `hashicorp/golang-lru` (generic, faster).

### 4. Fixed-size ring buffer for last-N events

```go
r := ring.New(100)
for ev := range stream {
	r.Value = ev
	r = r.Next()
}
// Dump last 100:
r.Do(func(v any) { if v != nil { fmt.Println(v) } })
```

Use case: keeping the last N log lines for diagnostics.

### 5. Discrete-event simulation

A priority queue keyed by event time. Pop the earliest, advance virtual clock, process, push generated events.

## Anti-Patterns & Gotchas

**`container/list` as a default.** Almost always slower than `[]T`.

**Forgetting `heap.Fix` after mutating an element.** Heap property broken silently.

**Calling `heap.Push` directly on the underlying slice.** Bypasses heap property; use `heap.Push(h, x)`.

**Using `container/heap` for a one-shot sort.** Use `slices.Sort`.

**`any` boxing in `container/list`.** Type assertions everywhere.

**Pop on empty heap.** Panics.

**Treating ring's `Len()` as element count of non-nil values.** It's the ring size; iterate and skip nils.

## Performance Notes

- `container/heap` ops are O(log n); constant factor decent.
- `container/list` Pop/Push are O(1) but pointer chasing kills cache locality.
- `container/ring` is similar to `list` cost-wise.
- A `[]T` deque (pop front via slice trick or `gammazero/deque`) outperforms `list` for nearly every workload.
- Generics let you write a typed heap in 50 lines that's faster than the stdlib (no `any` boxing).

## How Big Companies Use It

- **Kubernetes** uses `container/heap` in `client-go/util/workqueue` for delayed-retry queues.
- **etcd** uses heap for lease expiration tracking.
- **Cockroach** uses heap-based priority queues for query scheduling.
- **HashiCorp Nomad** uses heap for placement bin-packing.

## Source Code References

Pinned to `go1.26`.

- `container/heap`: [`src/container/heap/heap.go`](https://github.com/golang/go/blob/master/src/container/heap/heap.go).
- `container/list`: [`src/container/list/list.go`](https://github.com/golang/go/blob/master/src/container/list/list.go).
- `container/ring`: [`src/container/ring/ring.go`](https://github.com/golang/go/blob/master/src/container/ring/ring.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/container/heap, /list, /ring.
- "Why container/list is slow": community blog posts on cache-unfriendly linked lists.

## Exercises / Self-Check

1. Build a generic priority queue using generics; compare cost with stdlib `container/heap`.
2. Implement top-K with a size-K min-heap as above; test against a slow O(N log N) sort.
3. Build an LRU cache using `container/list` + map; then rewrite with a generic doubly-linked list. Benchmark.
4. Discrete-event simulation: simulate 1000 events with heap-based scheduling.
5. Why is `container/list` rarely the right answer in Go? Discuss cache locality and slice deques.
