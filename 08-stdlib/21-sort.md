# `sort` — Sorting (Mostly Superseded by `slices.Sort`)

## TL;DR

`sort` is the original sorting package: `sort.Slice`, `sort.SliceStable`, `sort.Sort` with `sort.Interface`. Since 1.21, `slices.Sort` and friends are faster (no reflection) and easier (generic). New code should reach for `slices` first; `sort` remains for custom comparators that don't fit a simple `cmp.Compare` and for the `sort.Search` binary search helper.

## Mental Model

```
sort.Slice(s, less)         // closure-based; reflection at runtime
sort.SliceStable(s, less)    // stable variant
sort.Sort(sort.Interface)    // implement Len/Less/Swap; pre-generics era

slices.Sort(s)               // 1.21+: ordered types, generic, fast
slices.SortFunc(s, cmp)      // custom comparator returning -1/0/+1
slices.SortStableFunc(s, cmp)
sort.Search(n, f)            // binary search; still useful
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"slices"
	"sort"
)

type Person struct{ Name string; Age int }

func main() {
	people := []Person{{"Bob", 30}, {"Ada", 25}, {"Eve", 25}}

	// Modern (1.21+):
	slices.SortFunc(people, func(a, b Person) int {
		if a.Age != b.Age { return a.Age - b.Age }
		return strings.Compare(a.Name, b.Name)
	})

	// Legacy:
	sort.Slice(people, func(i, j int) bool {
		if people[i].Age != people[j].Age {
			return people[i].Age < people[j].Age
		}
		return people[i].Name < people[j].Name
	})

	fmt.Println(people)
	// Output:
	// [{Ada 25} {Eve 25} {Bob 30}]
}
```

## Deep Dive

### `sort.Interface` (the original)

```go
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

type byAge []Person
func (b byAge) Len() int           { return len(b) }
func (b byAge) Less(i, j int) bool { return b[i].Age < b[j].Age }
func (b byAge) Swap(i, j int)      { b[i], b[j] = b[j], b[i] }

sort.Sort(byAge(people))
sort.Stable(byAge(people))
```

Verbose but zero-reflection. Pre-generics, this was the only way.

### `sort.Slice` (1.8+)

```go
sort.Slice(people, func(i, j int) bool { return people[i].Age < people[j].Age })
sort.SliceStable(people, less)
sort.SliceIsSorted(people, less)
```

Reflection-based; slower than `sort.Sort` (interface dispatch overhead).

### `slices.Sort*` (1.21+, modern)

```go
slices.Sort([]int{3,1,2})                                  // ordered types
slices.SortFunc(items, func(a, b Item) int {
	return cmp.Compare(a.ID, b.ID)
})
slices.SortStableFunc(items, less)
slices.IsSorted(s)
slices.IsSortedFunc(s, cmp)
slices.SortStable(s)  // (with cmp.Ordered constraint)
```

Comparator returns int: negative = a<b, 0 = equal, positive = a>b. Use `cmp.Compare(a, b)` from `cmp` package for ordered values.

### `cmp.Compare` and `cmp.Or` (1.21+)

```go
import "cmp"

slices.SortFunc(people, func(a, b Person) int {
	return cmp.Or(
		cmp.Compare(a.Age, b.Age),
		cmp.Compare(a.Name, b.Name),
	)
})
```

`cmp.Or` returns the first non-zero argument — perfect for multi-key sorts.

### Stable vs unstable

Unstable sorts may reorder equal elements; stable sorts preserve original order. Stable is slightly slower. Use stable when secondary order matters (e.g., already sorted by timestamp, now sorting by user — want user clustering with timestamps still ascending).

### `sort.Search` and `slices.BinarySearch`

```go
// sort.Search: predicate-based, classic
i := sort.Search(len(a), func(i int) bool { return a[i] >= target })
if i < len(a) && a[i] == target { /* found at i */ }

// slices.BinarySearch (1.21+): typed
i, found := slices.BinarySearch([]int{1,3,5,7}, 5)
i, found := slices.BinarySearchFunc(items, target, cmpFn)
```

The slice must be pre-sorted with the same ordering.

### Type-specific sort helpers (legacy)

```go
sort.Ints([]int{...})
sort.Strings([]string{...})
sort.Float64s([]float64{...})
```

`slices.Sort` covers all of these uniformly since 1.21.

## Standard Library Hooks

- `slices` — modern generic sort.
- `cmp` — `Compare`, `Or`, `Ordered` constraint.
- `sort.Reverse(data)` — wraps `sort.Interface` to invert.
- `sort.SearchInts`, `SearchStrings`, `SearchFloat64s` — typed binary searches.

## Real-World Patterns

### 1. Multi-key sort

```go
slices.SortFunc(events, func(a, b Event) int {
	return cmp.Or(
		cmp.Compare(a.Day, b.Day),
		cmp.Compare(b.Priority, a.Priority), // priority DESC
		cmp.Compare(a.ID, b.ID),
	)
})
```

Use case: scheduler ordering, report rows.

### 2. Sort and dedupe

```go
slices.Sort(s)
s = slices.Compact(s) // removes consecutive duplicates
```

Use case: building a unique sorted set without a map.

### 3. Top-K with partial sort

```go
// Stdlib has no partial sort; container/heap is the typical answer.
// For small K and small N, just sort:
slices.SortFunc(items, func(a, b Item) int { return cmp.Compare(b.Score, a.Score) })
top := items[:min(k, len(items))]
```

Use case: leaderboards.

### 4. Binary search for insertion

```go
i, _ := slices.BinarySearch(sorted, value)
sorted = slices.Insert(sorted, i, value)
```

Use case: maintaining a sorted slice with occasional inserts.

### 5. Reverse iteration

```go
slices.Reverse(s) // in-place
```

Or use indexing: `for i := len(s)-1; i >= 0; i--`.

## Anti-Patterns & Gotchas

**`sort.Slice` in new code.** Use `slices.SortFunc`; faster and clearer.

**`a[i] - a[j]` for `int` comparators when values can overflow.** Use `cmp.Compare`.

**Sorting a map** by iterating. Maps have no order; extract to a slice first.

**Comparator that's not transitive.** Undefined behavior; can loop forever.

**Comparator with side effects.** Indeterminate.

**`sort.Sort` on a slice expected to remain stable.** Use `Stable` variants.

**Binary search on unsorted data.** Returns garbage.

**`sort.SliceStable` thinking it's faster than `sort.Slice`.** It's slower.

**Sorting floats containing NaN.** NaN comparisons are weird; filter first or use `cmp` carefully.

## Performance Notes

- `sort.Sort` ≈ `slices.Sort` for primitives; `slices.Sort` wins on complex types thanks to inlining and no interface dispatch.
- `sort.Slice` is ~30% slower than `slices.SortFunc` due to reflection in `Less`.
- Underlying algorithm is pattern-defeating quicksort (pdqsort) since 1.19 — ~2× faster than the old introsort on average.
- Stable sort uses block merge; slightly more memory, slightly slower.
- For very small slices (< 12), insertion sort is used internally.

## How Big Companies Use It

- **gopls** sorts diagnostics and code actions using `slices.SortFunc`.
- **Cockroach** sorts query plans with multi-key comparators (`cmp.Or`).
- **Prometheus** sorts label sets for canonical encoding.
- **Kubernetes** has migrated many sort sites from `sort.Slice` to `slices.SortFunc`.

## Source Code References

Pinned to `go1.26`.

- `sort`: [`src/sort/sort.go`](https://github.com/golang/go/blob/master/src/sort/sort.go), `slice.go`, `search.go`.
- pdqsort implementation: [`src/sort/zsortfunc.go`](https://github.com/golang/go/blob/master/src/sort/zsortfunc.go).
- `slices.Sort`: [`src/slices/sort.go`](https://github.com/golang/go/blob/master/src/slices/sort.go).
- `cmp.Compare`: [`src/cmp/cmp.go`](https://github.com/golang/go/blob/master/src/cmp/cmp.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/sort, /slices, /cmp.
- Go blog, "Generic data structures and Go 1.21": https://go.dev/blog/comparable.
- pdqsort paper: https://arxiv.org/abs/2106.05123.

## Exercises / Self-Check

1. Convert a `sort.Slice` call to `slices.SortFunc` using `cmp.Compare`.
2. Implement multi-key sort (3 keys) with `cmp.Or`.
3. Use `slices.BinarySearch` to insert into a sorted slice maintaining order.
4. Sort a slice of structs containing floats with potential NaN values. Handle NaN explicitly.
5. Benchmark `sort.Slice` vs `slices.SortFunc` on 1M random ints.
