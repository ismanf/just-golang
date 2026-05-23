# Maps

## TL;DR

A map is Go's built-in hash table: `map[K]V` where `K` is any comparable type. As of Go 1.24, the runtime implementation is a **Swiss table** (replacing the older bucket-array layout). Maps are reference-like — assigning a map copies a header that points to the same underlying table. Iteration order is randomized on purpose. You **cannot take the address of a map value** (`&m[k]` is a compile error), and you cannot use a slice, map, or func as a key.

## Mental Model

```
m := map[string]int{}

map header (a *hmap pointer):
    ┌──────────┐
    │   *hmap  │───► table metadata + groups of slots
    └──────────┘

Swiss-table group (since 1.24, 8 entries per group):
    ┌─────────────┬─────────────┬───────────────────────┐
    │ ctrl bytes  │ keys[0..7]  │ values[0..7]          │
    └─────────────┴─────────────┴───────────────────────┘
       (8 bytes)
```

`ctrl` bytes tell the lookup whether a slot is empty / deleted / occupied (and a 7-bit hash fingerprint for fast skip). Lookup hashes the key, scans the relevant group's ctrl byte with a SIMD-like batch compare, and probes onward on collision.

You don't need to know all of this to use a map — but it explains why iteration order is unstable and why `delete` doesn't shrink the table.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	// zero value of a map is nil — reads OK, writes panic
	var nilMap map[string]int
	_ = nilMap["missing"] // returns 0, fine
	// nilMap["x"] = 1    // panics: assignment to entry in nil map

	m := map[string]int{"a": 1, "b": 2}
	m["c"] = 3
	v, ok := m["d"] // comma-ok: v=0, ok=false
	delete(m, "a")

	fmt.Println(len(m), v, ok)
	// Output:
	// 2 0 false

	preSized := make(map[string]int, 1000) // hint for capacity
	preSized["x"] = 1
}
```

`make(map[K]V, hint)` pre-allocates space for ~`hint` entries. It's a hint, not a hard cap.

## Deep Dive

### Comparability of keys

Keys must be **comparable**:
- All numeric, bool, string, pointer, channel, interface (if the dynamic type is comparable).
- Structs and arrays whose fields are all comparable.
- Not: slices, maps, functions. Type-assertion on an interface key with a non-comparable dynamic type panics at runtime.

```go
type bad struct{ s []int }
m := map[any]int{}
m[bad{nil}] = 1 // compiles, but ...
m[bad{nil}]      // panics: runtime error: hash of unhashable type
```

### `nil` map vs empty map

```go
var nilMap map[string]int  // nil
empty := map[string]int{}  // non-nil

len(nilMap) == 0           // true
len(empty) == 0            // true
nilMap == nil              // true
empty == nil               // false
```

Both are fine to read from and range over. Only the empty one is safe to write to.

### Iteration order is randomized

```go
m := map[int]int{0: 0, 1: 1, 2: 2, 3: 3, 4: 4}
for k := range m {
	fmt.Print(k, " ")
}
// Output is intentionally non-deterministic across runs.
```

This was added in Go 1.0 specifically to discourage relying on ordering. Tests that print map contents must sort first.

### Values are not addressable

```go
type Counter struct{ N int }
m := map[string]Counter{}
m["x"] = Counter{}
// m["x"].N++ // compile error: cannot assign to struct field m["x"].N in map
```

Two idiomatic fixes:

```go
// (a) Read-modify-write the whole value
c := m["x"]
c.N++
m["x"] = c

// (b) Make the map hold pointers
mp := map[string]*Counter{"x": {}}
mp["x"].N++ // fine
```

`map[K]*V` is the right choice when V is large or you genuinely want shared mutation.

### `delete` doesn't shrink

```go
m := make(map[int]int, 1000)
for i := 0; i < 1000; i++ { m[i] = i }
for k := range m { delete(m, k) }
// len(m) == 0, but the table still owns ~1000 slots of memory
```

Pre-1.21 the only way to reclaim was `m = map[int]int{}`. As of 1.21+ and especially the 1.24 Swiss-table rework, the runtime can reuse slots more efficiently, but it still doesn't compact aggressively. For pathological churn, periodically replace the map.

### Concurrency

Maps are **not safe for concurrent use**. The runtime detects concurrent read+write and crashes with `fatal error: concurrent map read and map write`. Use `sync.RWMutex` + `map`, or `sync.Map` (good for write-once read-many disjoint key sets), or shard.

### `clear` builtin (since 1.21)

```go
m := map[int]int{1: 1, 2: 2}
clear(m)
fmt.Println(len(m)) // 0
```

Equivalent to `for k := range m { delete(m, k) }` but the runtime can do it in one shot.

## Standard Library Hooks

- `maps` package (since 1.21): `Clone`, `Copy`, `DeleteFunc`, `Equal`, `EqualFunc`, `Keys` (returns `iter.Seq` since 1.23), `Values`.
- `cmp.Compare` for sorting by map values.
- `encoding/json`: marshals maps with `string`-convertible keys; iteration order is **sorted** in JSON output for determinism (this is special-cased in the encoder).
- `sync.Map`: specialized for two access patterns; not a general-purpose concurrent map. Read its doc carefully.
- `hash/maphash`: hash a value with the same algorithm Go uses for map keys.

## Real-World Patterns

### 1. Set via `map[T]struct{}`

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(v T)       { s[v] = struct{}{} }
func (s Set[T]) Has(v T) bool  { _, ok := s[v]; return ok }
func (s Set[T]) Delete(v T)    { delete(s, v) }
```

`struct{}` occupies zero bytes. Don't use `map[T]bool` for sets in hot paths.

### 2. Counter with `map[K]int`

```go
counts := map[string]int{}
for _, word := range words {
	counts[word]++ // zero value of int is 0, so this works on first sight
}
```

The zero-value semantics make this idiomatic. No need to check existence first.

### 3. Group-by

```go
import "slices"

func groupBy[K comparable, V any](items []V, keyFn func(V) K) map[K][]V {
	out := make(map[K][]V, len(items)/4) // rough hint
	for _, item := range items {
		k := keyFn(item)
		out[k] = append(out[k], item)
	}
	return out
}

// usage:
people := []Person{...}
byCity := groupBy(people, func(p Person) string { return p.City })
_ = slices.Sorted(maps.Keys(byCity)) // deterministic iteration
```

### 4. Sharded map for concurrent workloads

```go
type Shard[K comparable, V any] struct {
	mu sync.RWMutex
	m  map[K]V
}

type Sharded[K comparable, V any] struct {
	shards [16]Shard[K, V]
	hash   func(K) uint64
}

func (s *Sharded[K, V]) shard(k K) *Shard[K, V] {
	return &s.shards[s.hash(k)%uint64(len(s.shards))]
}
```

For workloads with high contention, 16–64 shards usually outperforms `sync.Map`.

### 5. Deterministic JSON-like output

```go
import (
	"fmt"
	"slices"
	"maps"
)

func print(m map[string]int) {
	keys := slices.Sorted(maps.Keys(m))
	for _, k := range keys {
		fmt.Printf("%s=%d\n", k, m[k])
	}
}
```

## Anti-Patterns & Gotchas

**Concurrent access without synchronization.** The runtime will crash you. There is no "it usually works."

**`map[string]Foo` where Foo is a fat struct, then `m[k].Field`.** Compile error. Either store pointers (`map[string]*Foo`) or read-modify-write.

**`len(m)` for capacity planning.** `len` is the number of live entries, not the bucket count. You can't query capacity.

**Iterating and `delete`-ing**, expecting to see/skip current key. The spec allows the iterator to see or skip a deleted-during-iteration key. Don't write code that depends on either.

**Iterating and `delete`-ing all keys to "reset" the map.** Use `clear(m)`.

**Using a map for ordered data.** Maps are unordered. If you need order, keep a `[]K` of insertion order alongside, or use a sorted structure.

**Storing very large values (`map[string][1<<20]byte`).** Each grow rehashes and copies every value. Box with a pointer.

**Hashing too many similar keys** (e.g., adversarial input). Go's hash function is seeded per-map at creation, so DoS via key collision is hard but not impossible. For untrusted inputs, prefer `maphash` with explicit seeds.

## Performance Notes

- Pre-size: `make(map[K]V, n)` saves several rehash cycles when `n` is large.
- Reading from a `nil` map is allocation-free and fast; writing to `nil` is a runtime panic.
- The Swiss-table rewrite (1.24) cut lookup latency ~30% on typical workloads and improved cache behavior; see the Go team's announcement.
- `map[int]V` is faster than `map[string]V` for the same number of entries — string hashing is more expensive.
- `delete` doesn't free; if you cycle billions of keys through one map, replace it occasionally.
- For read-mostly maps, `sync.Map` only wins when the working set is stable and reads dominate writes. Benchmark before reaching for it.
- `map[K]struct{}` beats `map[K]bool` slightly on memory and not at all on speed.

## How Big Companies Use It

- **The Go team's own Swiss-table redesign**, championed by the runtime team, was driven by Google's internal services: https://go.dev/blog/swisstable.
- **Kubernetes' `cache` package** ([`staging/src/k8s.io/client-go/tools/cache`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/client-go/tools/cache)) uses sharded maps under `sync.Mutex` for the informer indexers — production-grade reference.
- **Caddy's reverse proxy** uses `sync.Map` for upstream pools.
- **Prometheus's TSDB** uses `map[uint64]series` keyed by FNV-hashed labels — read [`tsdb/head.go`](https://github.com/prometheus/prometheus/blob/main/tsdb/head.go).
- **etcd's `lease` package** uses `map[LeaseID]*Lease` with explicit RWMutex.

## Source Code References

Pinned to `go1.26`.

- Swiss-table implementation (since 1.24): [`src/internal/runtime/maps`](https://github.com/golang/go/tree/master/src/internal/runtime/maps).
- Legacy bucket-based map (kept for fallback / arena types): [`src/runtime/map.go`](https://github.com/golang/go/blob/master/src/runtime/map.go).
- Map header / hash seed: same file, search `hmap`.
- `maps` package: [`src/maps/maps.go`](https://github.com/golang/go/blob/master/src/maps/maps.go).
- `sync.Map`: [`src/sync/map.go`](https://github.com/golang/go/blob/master/src/sync/map.go) — read the comment at the top; the data structure is unusual.
- Concurrent map crash detection: search `mapaccess` for `throw("concurrent map read and map write")`.

## Further Reading

- Go blog, "Inside the map implementation" (talk by Keith Randall): https://www.youtube.com/watch?v=Tl7mi9QmLns
- Swiss-table design (Google original): https://abseil.io/about/design/swisstables
- Go 1.24 release notes (Swiss tables): https://go.dev/doc/go1.24
- Spec, "Map types": https://go.dev/ref/spec#Map_types
- Dave Cheney, "If a map isn't a reference variable, what is it?": https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it
- `sync.Map` design discussion: https://go.dev/issue/18177

## Exercises / Self-Check

1. Why does `m["x"].field = y` fail to compile for `m map[string]Struct`? What two fixes work?
2. Write a `topK[K comparable](counts map[K]int, k int) []K` that returns the k most-frequent keys deterministically.
3. Build a benchmark comparing `map[int]struct{}` vs `map[int]bool` for a set of size 1M. Do you see a difference?
4. Implement a sharded concurrent map and benchmark it against `sync.Map` under 80% reads / 20% writes.
5. What does `delete(m, k)` do to `cap`/memory usage? Demonstrate with `runtime.ReadMemStats` before and after deleting all keys.
