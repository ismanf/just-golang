# `slices`, `maps`, `cmp` — Generic Helpers (since 1.21)

## TL;DR

`slices`, `maps`, and `cmp` are the modern generic counterparts to scattered helpers in older Go. They replace most uses of `sort.Slice`, hand-rolled find/contains/dedupe loops, and ad-hoc map iteration. Use them as defaults: `slices.Sort`, `slices.Contains`, `slices.Index`, `slices.Equal`, `maps.Equal`, `cmp.Compare`, `cmp.Or`. Faster than reflection-based predecessors; less code at the call site.

## Mental Model

```
slices: ordered/unordered ops on []T
   ├─ Sort, SortFunc, SortStableFunc
   ├─ Contains, ContainsFunc, Index, IndexFunc, LastIndex
   ├─ Equal, EqualFunc, Compare, CompareFunc
   ├─ Reverse, Replace, Insert, Delete, DeleteFunc
   ├─ BinarySearch, BinarySearchFunc
   ├─ Compact, CompactFunc (dedupe consecutive equals)
   ├─ Clone, Concat, Grow, Clip
   ├─ Max, Min, MaxFunc, MinFunc
   └─ All, Values, Chunk, Backward, Sorted (iter helpers, 1.23+)

maps:
   ├─ Equal, EqualFunc, Clone, Copy
   ├─ DeleteFunc, Keys (iter, 1.23+), Values, All, Collect, Insert
   └─ no map sorting (maps have no order)

cmp:
   ├─ Compare[T Ordered](a, b) int   → -1, 0, +1
   ├─ Or[T comparable](vals ...T) T   → first non-zero
   ├─ Less[T Ordered], Equal
   └─ Ordered constraint
```

## Syntax & Basic Usage

```go
package main

import (
	"cmp"
	"fmt"
	"maps"
	"slices"
)

func main() {
	s := []int{3, 1, 4, 1, 5, 9, 2, 6}
	slices.Sort(s)
	fmt.Println(s)
	fmt.Println(slices.Contains(s, 4))
	fmt.Println(slices.Index(s, 5))

	m := map[string]int{"a": 1, "b": 2}
	m2 := maps.Clone(m)
	fmt.Println(maps.Equal(m, m2))

	type Person struct{ Name string; Age int }
	people := []Person{{"B", 30}, {"A", 30}, {"A", 25}}
	slices.SortFunc(people, func(x, y Person) int {
		return cmp.Or(cmp.Compare(x.Name, y.Name), cmp.Compare(x.Age, y.Age))
	})
	fmt.Println(people)
	// Output:
	// [1 1 2 3 4 5 6 9]
	// true
	// 4
	// true
	// [{A 25} {A 30} {B 30}]
}
```

## Deep Dive

### `slices`

Most-used:

- `Sort(s)` — in-place, ordered types.
- `SortFunc(s, cmp)` — custom comparator returning -1/0/+1.
- `SortStable*` — preserve equal-element order.
- `Contains(s, v)` / `ContainsFunc(s, pred)`.
- `Index(s, v)` / `IndexFunc(s, pred)`; `LastIndex*`.
- `Equal(a, b)` / `EqualFunc(a, b, eq)`.
- `Compare(a, b)` — lex compare returning -1/0/+1.
- `Reverse(s)` — in-place.
- `Replace(s, i, j, v...)` — replace s[i:j] with v.
- `Insert(s, i, v...)` — insert at i, returns new slice.
- `Delete(s, i, j)` — remove s[i:j], returns new slice.
- `DeleteFunc(s, pred)` — remove matching elements.
- `Compact(s)` — remove **consecutive** duplicates (sort first for global dedupe).
- `BinarySearch(s, v)` — returns (index, found).
- `Clone(s)` — shallow copy.
- `Concat(s1, s2, ...)` — new slice with all elements.
- `Grow(s, n)` — ensure capacity for n more.
- `Clip(s)` — set cap = len (release unused capacity).
- `Max(s)`, `Min(s)`, `MaxFunc`, `MinFunc`.

Iterator helpers (1.23+): `All`, `Values`, `Backward`, `Chunk`, `Sorted`, `SortedFunc`, `Collect`.

### `maps`

- `Equal(a, b)` / `EqualFunc(a, b, eq)`.
- `Clone(m)` — shallow copy of map.
- `Copy(dst, src)` — copy all entries from src to dst.
- `DeleteFunc(m, pred)` — delete matching keys.

Iterator helpers (1.23+): `Keys`, `Values`, `All`, `Collect`, `Insert`.

No sort: maps are unordered.

### `cmp`

- `Compare[T Ordered](a, b T) int` — generic three-way.
- `Less[T Ordered](a, b T) bool`.
- `Equal[T comparable](a, b T) bool`.
- `Or[T comparable](vals ...T) T` — first non-zero argument.
- `Ordered` constraint — `int*`, `uint*`, `float*`, `string`.

### Generic constraints used here

```go
type Ordered interface {
	~int | ~int8 | ... | ~uint | ... | ~float32 | ~float64 | ~string
}
```

Tilde `~` includes named types (e.g., `type ID int`).

### When `slices.Sort` vs `sort.Slice`

`slices.Sort` is generic, faster (no reflection), and works on any `Ordered` type. `sort.Slice` is reflection-based and slower; use only when you can't write a typed `SortFunc` (rare).

### `slices.Delete` and aliasing

```go
s := []int{1, 2, 3, 4, 5}
s = slices.Delete(s, 1, 3) // removes indices 1,2 → s == [1, 4, 5]
```

Mutates `s`; reassign to capture the new length. Capacity unchanged.

### `slices.Compact` vs `slices.CompactFunc`

```go
s := []int{1, 1, 2, 3, 3, 3, 4}
s = slices.Compact(s)  // [1, 2, 3, 4]
```

Only removes **consecutive** duplicates. For global dedupe: `slices.Sort(s); s = slices.Compact(s)`.

## Standard Library Hooks

- `sort` — older, partially superseded.
- `iter` — iterator producers/consumers.

## Real-World Patterns

### 1. Unique sorted slice

```go
slices.Sort(s)
s = slices.Compact(s) // dedupe
```

Use case: building a unique list without a map.

### 2. Find an item

```go
i := slices.IndexFunc(users, func(u User) bool { return u.ID == target })
if i >= 0 { return &users[i], nil }
```

### 3. Multi-key sort

```go
slices.SortFunc(events, func(a, b Event) int {
	return cmp.Or(
		cmp.Compare(a.Priority, b.Priority),
		cmp.Compare(a.Time.UnixNano(), b.Time.UnixNano()),
	)
})
```

### 4. Map equality in tests

```go
if !maps.Equal(got, want) { t.Errorf("got %v, want %v", got, want) }
```

Faster than `reflect.DeepEqual` and clearer.

### 5. Chunked processing

```go
import "slices"

for batch := range slices.Chunk(items, 100) {
	if err := bulkInsert(batch); err != nil { return err }
}
```

(Requires 1.23+ for `slices.Chunk` returning iterator.)

### 6. Top-K

```go
slices.SortFunc(items, func(a, b Item) int { return cmp.Compare(b.Score, a.Score) })
top := items[:min(k, len(items))]
```

## Anti-Patterns & Gotchas

**`sort.Slice` in new code.** Use `slices.SortFunc`.

**`slices.Compact` without sorting first** when you wanted global dedupe.

**`slices.Delete` and forgetting to assign back.** Original `s` retains zeroed tail.

**Comparing slices with `==`.** Compile error. Use `slices.Equal`.

**Comparing maps with `==`.** Same. Use `maps.Equal`.

**`reflect.DeepEqual` on simple slices/maps.** Slow; use `slices.Equal`/`maps.Equal`.

**Mutating a slice while iterating.** Same trap as anywhere; iterate over a clone if needed.

**`slices.Clone` for deep copy.** It's shallow.

**`cmp.Compare` on `float64` with NaN.** Treats NaN as smaller; surprises possible.

## Performance Notes

- `slices.Sort` ≈ `sort.Sort` for primitives; better than `sort.Slice` (no reflection).
- `slices.Contains` is O(N) linear scan; for repeated lookups, use a map (`map[T]struct{}`).
- `slices.BinarySearch` is O(log N).
- `slices.Delete` shifts elements; O(N).
- `maps.Clone` is O(N) over entries.
- Generic dispatch is monomorphized for built-in types — no boxing, no interface tables for `slices.Sort([]int)`.

## How Big Companies Use It

- **Kubernetes** has migrated many internal sites from `sort.Slice` and hand-rolled find loops.
- **Cockroach** uses `cmp.Or` for tie-breaking in query plans.
- **gopls** uses `slices.Contains`/`IndexFunc` extensively.
- **HashiCorp Terraform** uses `slices.Equal` for plan comparisons.

## Source Code References

Pinned to `go1.26`.

- `slices`: [`src/slices/slices.go`](https://github.com/golang/go/blob/master/src/slices/slices.go), `sort.go`, `iter.go`.
- `maps`: [`src/maps/maps.go`](https://github.com/golang/go/blob/master/src/maps/maps.go), `iter.go`.
- `cmp`: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/slices, /maps, /cmp.
- Go 1.21 release notes: https://go.dev/doc/go1.21.
- Go blog, "Generic data structures and Go 1.21".

## Exercises / Self-Check

1. Dedupe a `[]string`: sort then compact.
2. Multi-key sort with `cmp.Or` across three keys.
3. Test map equality with `maps.Equal`; compare speed to `reflect.DeepEqual`.
4. Use `slices.BinarySearch` to maintain a sorted slice with insertions.
5. Convert a `sort.Slice` call from an existing codebase to `slices.SortFunc`. Measure diff in lines and benchmark.
