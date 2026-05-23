# Experimental Arenas — Why They Were Pulled

## TL;DR

The `arena` package was added under `GOEXPERIMENT=arenas` in **Go 1.20** as a way to allocate many objects in a single region and free them all at once — bypassing the GC for short-lived bulk workloads. After roughly two years of experimentation, the proposal was **declined** (parked indefinitely) in March 2024 (see [issue #51317](https://github.com/golang/go/issues/51317)). It still works under `GOEXPERIMENT=arenas` for those who patched it back in, but is **not** going into the standard library. The single biggest gotcha: **arenas were declined not because they were a bad idea, but because the safety guarantees were hard to enforce**. Use-after-free across arena boundaries could compromise the runtime. The community alternatives — `[]byte` arenas, `sync.Pool`, off-heap encoding — solve the same problems with less invasive machinery.

## Mental Model

```
   Normal allocation:
   
   x := new(T)            // mallocgc; tracked by GC
   // ... later, GC marks unreachable, sweep frees.

   Arena allocation (proposed, experimental):
   
   a := arena.NewArena()
   defer a.Free()                   // explicit bulk-free
   
   for i := 0; i < N; i++ {
       x := arena.New[T](a)         // allocated in `a`
       process(x)
   }
   // All Ts freed when `a.Free()` runs. No GC scan of these.

   The objects in `a` are visible to the runtime (their pointers are
   tracked) but their *backing memory* is reclaimed all at once.
```

The design point: amortize allocator cost across many objects, then dump them en masse. Skip the GC for the lifetime of the arena.

## Syntax & Basic Usage

Under `GOEXPERIMENT=arenas` (1.20):

```go
//go:build goexperiment.arenas

package main

import (
	"arena"
	"fmt"
)

type T struct{ x, y int }

func main() {
	a := arena.NewArena()
	defer a.Free()

	xs := make([]*T, 0, 1000)
	for i := 0; i < 1000; i++ {
		t := arena.New[T](a)
		t.x = i
		t.y = i * 2
		xs = append(xs, t)
	}

	fmt.Println(xs[42].x, xs[42].y)
}
```

`arena.New[T](a)` allocates a `*T` in arena `a`. When `a.Free()` runs, every object allocated from it is reclaimed at once. The slice header is *not* in the arena unless you use `arena.MakeSlice`.

Compile:

```bash
$ GOEXPERIMENT=arenas go build .
```

Without the experiment flag, the package isn't importable.

## Deep Dive

### History

- **2022** ([#51317](https://github.com/golang/go/issues/51317)): proposal opened by Dan Scales, Cherry Mui, others.
- **Go 1.20 (Feb 2023)**: shipped as `GOEXPERIMENT=arenas`.
- **2023**: production trials at Google internal services; modest wins on specific workloads.
- **Mar 2024**: Austin Clements posted "arenas declined". Reasons:
  1. Safety: hard to make use-after-free impossible without runtime checks (which negate the perf win).
  2. API complexity creep: `arena.MakeSlice`, `arena.New`, lifetime tracking through closures, etc.
  3. Alternatives (sync.Pool, manual arenas) cover most use cases adequately.
  4. The Go team felt the savings (typically <10% on benchmarks) didn't justify a permanent stdlib API.
- **2024+**: code still exists under `GOEXPERIMENT=arenas` for users willing to live without future stability guarantees.

### Why arenas seemed promising

Workloads that benefit:
- **Burst allocations**: web request that creates 1000 short-lived objects.
- **Decoders**: parsing a wire format produces many objects all freed together.
- **Compaction**: building a large in-memory structure then writing it to disk.

In these cases, the per-object GC cost (mark + sweep) is wasted; the natural lifetime is "this batch".

### Why arenas failed

1. **Pointer escape across boundaries**. If a pointer from an arena leaks into a long-lived structure, freeing the arena creates a dangling pointer. The runtime can detect some leaks but not all.

2. **GC interaction**. The arena's memory must be scanned by the GC (any of its objects might point to GC-tracked heap). The savings are in *free*, not in *scan*. For pointer-dense data, the win is small.

3. **API ergonomics**. `arena.New[T](a)` is more verbose than `new(T)`. Slice handling required new helpers. Channels and maps couldn't easily be arena-allocated.

4. **Safer alternatives existed**. `sync.Pool` for reuse; `[]byte` arenas for bulk allocation; off-heap codecs (sqlc-generated, protobuf) for performance.

5. **Cost of a stable API**. Once shipped, the Go team owns its bugs forever. The team prefers fewer stable surfaces.

### The declined-but-not-removed status

Per Austin Clements' comments on the issue:

> "We are no longer planning to ship `arena` in the standard library. The package will remain available under GOEXPERIMENT=arenas for those who find it useful, but it is unsupported."

So:
- The code is still in `src/arena/` in the Go tree.
- `GOEXPERIMENT=arenas` still builds it.
- No stability promise; future Go versions may remove or break it.
- Don't depend on it in production new code.

### Production users

Some Go shops did use arenas under experiment:

- **Google internal**: documented modest wins on specific RPC handlers (5-10% lower allocator CPU on burst workloads).
- **One-off benchmarks**: people on /r/golang and Hacker News showed contrived 30%+ wins.
- **Network parsers**: where every parsed message dies together with the connection.

Few "real" deployments survive past the decline announcement.

### What to use instead

#### `sync.Pool` — for object reuse

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(input []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() { buf.Reset(); bufPool.Put(buf) }()
    // use buf
}
```

Best when objects are reused often. Cleared at GC; you re-fill on demand.

#### `[]byte` arena — for bulk allocation

```go
type arena struct {
    buf []byte
}

func (a *arena) Alloc(n int) []byte {
    if cap(a.buf)-len(a.buf) < n {
        a.buf = make([]byte, len(a.buf), max(cap(a.buf)*2, len(a.buf)+n))
    }
    a.buf = append(a.buf, make([]byte, n)...)
    return a.buf[len(a.buf)-n:]
}
```

Allocate `T`s by reinterpreting slices of `[]byte`. No GC overhead (the slice is one object); manual lifetime.

Common in CockroachDB's `coldata`, Dgraph's posting list, Badger's value log.

#### Off-heap codecs

Generate code that operates directly on `[]byte` (protobuf, Cap'n Proto, sqlc, FlatBuffers). No per-row allocations.

#### Reuse in iterators

`iter.Seq` (1.23+) lets pipelines avoid allocating intermediate slices.

### Pointer rules (when arenas worked)

The experimental arena package documented these rules:

- A pointer into an arena cannot outlive the arena.
- Pointers from arena memory to GC-tracked memory are fine.
- Pointers from GC-tracked memory to arena memory are **not allowed** (would be invalidated when arena frees).
- Channels, maps, and interfaces stored in arena memory: complex; restricted.

Violations were *runtime panics* or *use-after-free* depending on detection.

### Performance numbers (the experimental ones)

From the experimental rollout:

- Pure allocation throughput: 2-3× faster than `new(T)` for small T.
- Free cost: O(1) per arena instead of O(N) sweep.
- Total wins on real workloads: 5-15% in best cases.
- Memory usage: roughly the same; arenas don't reduce live size, just defer free.

Modest. The arguments against the API complexity outweighed these wins.

## Standard Library Hooks

The `arena` package is not in the stable stdlib. Its API was:

- `arena.NewArena() *Arena`.
- `arena.New[T](a *Arena) *T`.
- `arena.MakeSlice[T](a *Arena, len, cap int) []T`.
- `(*Arena).Free()`.

Don't import in code that needs to work across Go versions.

## Real-World Patterns

The "what to do instead" patterns:

### 1. sync.Pool for short-lived buffers

```go
var pool = sync.Pool{New: func() any { return new(parser) }}

func parse(input []byte) Result {
    p := pool.Get().(*parser)
    defer func() { p.reset(); pool.Put(p) }()
    return p.run(input)
}
```

### 2. byte arena for many small objects

```go
type Person struct {
    name [32]byte  // fixed size, no string header
    age  uint8
}

type arena struct {
    persons [1024]Person
    n       int
}

func (a *arena) NewPerson(name string, age uint8) *Person {
    if a.n >= len(a.persons) { /* grow or panic */ }
    p := &a.persons[a.n]
    a.n++
    copy(p.name[:], name)
    p.age = age
    return p
}
```

Backing storage is one allocation; "free" happens when the arena goes out of scope.

### 3. Reused decoder state

```go
type decoder struct {
    scratch []byte
    nodes   []node
}

func (d *decoder) Reset() {
    d.scratch = d.scratch[:0]
    d.nodes = d.nodes[:0]
}

func (d *decoder) Decode(input []byte) (*Result, error) {
    d.Reset()
    // populate d.scratch, d.nodes; return view into them
    return &Result{}, nil
}
```

Decoder lives across many calls; per-call state is recycled.

### 4. Generic generic-decoder

```go
type Decoder[T any] struct {
    pool sync.Pool
}

func (d *Decoder[T]) Get() *T {
    v, _ := d.pool.Get().(*T)
    if v == nil { v = new(T) }
    return v
}

func (d *Decoder[T]) Put(v *T) {
    var zero T
    *v = zero
    d.pool.Put(v)
}
```

### 5. Profile to find allocator hotspots

```bash
$ go test -bench=. -benchmem -memprofile=mem.prof
$ go tool pprof -alloc_space mem.prof
(pprof) top
(pprof) list HotFunction
```

Don't reach for arenas (or any optimization) before profiling.

## Anti-Patterns & Gotchas

**Building an arena library in user code that mimics `arena`.** Mostly futile; the value Go's arena offered was tight runtime integration. Without it, you're just allocating `[]byte` arenas — which is fine, but call it that.

**Storing `*T` from a custom arena across goroutines without explicit sync.** Same hazards.

**Assuming "arena = no GC".** GC still scans pointers *within* arena objects (any object that points to GC memory contributes to scan time).

**Importing `arena` in shared library code.** Anyone using your library now needs `GOEXPERIMENT=arenas`. Don't.

**Hoping arenas come back.** No active plan. Build around alternatives.

**Mixing arena and `runtime.SetFinalizer`.** Confusing semantics; the experimental package never supported finalizers cleanly.

**Skipping benchmarks before optimizing**. Most workloads don't need arenas. Profile first.

## Performance Notes

(Historical, from the experimental period.)

- Arena allocation: ~ns per object (bump allocator).
- Standard `new(T)`: ~10-30 ns + GC bookkeeping.
- Arena free: O(1) per arena instance.
- GC mark cost on arena: same per-pointer as elsewhere.

Alternatives:
- `sync.Pool` Get + Put: ~50 ns each, on warm path.
- byte arena: ~ns per object after warmup; manual.

## How Big Companies Used It

While experimental, very few production deployments. Reports came from:

- Google internal services that profiled and tried.
- Hobbyist benchmarks demonstrating peak speedups.
- Talk audiences at GopherCon (2023) discussing experiences.

Post-decline (2024+), production use is rare; companies that wanted arenas built bespoke ones.

## Source Code References

Pinned to `go1.26`.

- `arena` package (under `GOEXPERIMENT=arenas`): [`src/arena/arena.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/arena/arena.go).
- Runtime support: [`src/runtime/arena.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/arena.go).
- Issue #51317 (proposal): https://github.com/golang/go/issues/51317.
- Decline summary: Austin Clements' comment in March 2024 on the same issue.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Issue #51317 (proposal + decline): https://github.com/golang/go/issues/51317.
- "Memory arenas in Go 1.20" (release notes): https://go.dev/doc/go1.20.
- "Why arenas were declined" — community summaries on Hacker News.
- Dan Scales, "Arena allocation in Go" — design talks.
- "Allocator alternatives in Go" — Klauspost, Dgryski blog posts.
- The arena experiment vs Rust's typed arenas / Zig's allocators: comparative analyses.
- "When sync.Pool isn't enough" — community blog posts.

## Exercises / Self-Check

1. Write a `[]byte`-based arena that supports `NewPerson(name string, age uint8) *Person`. Compare allocations to `new(Person)` via `-benchmem`.
2. Read the decline comment by Austin Clements on issue #51317. Identify the three biggest stated reasons.
3. Implement a parser whose per-call state is reused via `sync.Pool`. Measure GC pressure with and without the pool.
4. Why does Go's arena experiment require the GC to still scan arena memory? What could allow skipping the scan entirely?
5. Argue for or against re-proposing arenas in a future version. What would need to change to satisfy the team's safety concerns?
