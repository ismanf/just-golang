# Slices

## TL;DR

A slice is a **3-word descriptor** (pointer, length, capacity) over a backing array. It is Go's default sequence type. The single biggest gotcha: two slices can share the same backing array, so `append` to one can silently mutate the other — until `append` triggers a reallocation, at which point they diverge. Master the header model, the growth rule, and `slices.Clip` and most slice bugs vanish.

## Mental Model

```
sl := arr[2:5:7]

backing array (cap=10):
+----+----+----+----+----+----+----+----+----+----+
| 00 | 01 | 02 | 03 | 04 | 05 | 06 | 07 | 08 | 09 |
+----+----+----+----+----+----+----+----+----+----+
            ^             ^         ^
            |             |         |
            ptr        ptr+len   ptr+cap
            len=3, cap=5

slice header (24 bytes on 64-bit):
+---------+-----+-----+
|   ptr   | len | cap |
+---------+-----+-----+
```

The slice itself is value-typed (three words copied on assignment), but it *points into* shared memory. That is the entire mental model. Everything else falls out of it.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	var nilSlice []int                  // nil, len=0, cap=0
	empty := []int{}                    // non-nil, len=0, cap=0
	literal := []int{10, 20, 30}        // len=3, cap=3
	made := make([]int, 5)              // len=5, cap=5, all zero
	madeCap := make([]int, 0, 16)       // len=0, cap=16

	sub := literal[1:3]                 // len=2, cap=2
	bound := literal[1:3:3]             // 3-index form: len=2, cap=2 (no over-share)

	fmt.Println(nilSlice == nil, empty == nil)
	fmt.Println(made, madeCap, sub, bound)
	// Output:
	// true false
	// [0 0 0 0 0] [] [20 30] [20 30]
}
```

`nil` and `empty` slices behave identically for `len`, `cap`, `range`, and `append`. They differ only under `== nil`. Prefer `len(s) == 0`.

## Deep Dive

### The header

Defined (conceptually) in `reflect.SliceHeader`, but the canonical runtime layout is in `runtime/slice.go`:

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

`reflect.SliceHeader` exists but is **deprecated as of 1.21** — use `unsafe.Slice` / `unsafe.SliceData` instead.

### `append` semantics and the growth rule

```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 4)
	for i := 0; i < 10; i++ {
		old := cap(s)
		s = append(s, i)
		if cap(s) != old {
			fmt.Printf("grew at len=%d: cap %d -> %d\n", len(s), old, cap(s))
		}
	}
	// Output:
	// grew at len=5: cap 4 -> 8
	// grew at len=9: cap 8 -> 16
}
```

Pre-1.18 the rule was "double until 1024, then +25%". Since 1.18 the growth is **smoothed** and rounded to fit size classes; see `growslice` in `runtime/slice.go`. Don't depend on exact numbers, but expect roughly geometric growth.

### The aliasing trap

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4}
	b := a[:2]
	b = append(b, 99) // writes into a's backing array at index 2!
	fmt.Println(a, b)
	// Output:
	// [1 2 99 4] [1 2 99]
}
```

`b` had `cap=4`, room to grow without reallocating, so `append` reused the same memory. To avoid: use the three-index form to clip capacity.

```go
b := a[:2:2]            // cap=2 — next append must reallocate
b = append(b, 99)
fmt.Println(a, b)
// a is unchanged: [1 2 3 4]
// b: [1 2 99]
```

Or use `slices.Clip(a[:2])` for the same effect.

### Passing slices to functions

The header is passed by value, but the backing array is shared. So callee writes via `s[i] = ...` are visible; `s = append(s, ...)` is **not** visible unless you return the new header.

```go
func badAppend(s []int)         { s = append(s, 99) }      // caller sees nothing
func goodAppend(s []int) []int  { return append(s, 99) }   // canonical
```

### Slices of slices, and the `[][]T` table pattern

```go
grid := make([][]int, 3)
for i := range grid {
	grid[i] = make([]int, 4)
}
```

Each row is independently allocated. A common micro-optimization is to allocate one big slice and re-slice:

```go
flat := make([]int, 3*4)
grid := make([][]int, 3)
for i := range grid {
	grid[i] = flat[i*4 : (i+1)*4 : (i+1)*4]
}
```

One allocation instead of four, better cache locality, but rows now share backing memory — be careful with `append`.

### Iteration

```go
for i, v := range s { /* ... */ }
```

`v` is a copy of `s[i]`. To mutate in place, use `s[i]`. Since Go 1.22, `i` and `v` are scoped per-iteration (each iteration is a new variable), which fixes the classic loop-variable closure bug.

## Standard Library Hooks

- `slices` package (since 1.21): `Sort`, `SortFunc`, `Index`, `Contains`, `Delete`, `Insert`, `Reverse`, `Clone`, `Clip`, `Grow`, `Compact`, `Equal`, `BinarySearch`, `Min`, `Max`, `Concat`.
- `slices.Clip(s)` returns `s[:len(s):len(s)]` — drops excess capacity, prevents accidental sharing.
- `slices.Grow(s, n)` ensures `cap(s) >= len(s)+n` with at most one allocation.
- `bytes` and `strings` packages — slice-shaped APIs.
- `sort.Slice` (predates generics; still works but `slices.SortFunc` is preferred now).
- `unsafe.Slice(ptr, len)` builds a slice from a pointer + length.
- `unsafe.SliceData(s)` returns the pointer to the first element (1.20+).

## Real-World Patterns

### 1. Reuse a buffer across iterations

```go
package main

import (
	"bufio"
	"os"
)

func processLines(r *os.File) error {
	buf := make([]byte, 0, 64*1024) // grow once, reuse forever
	sc := bufio.NewScanner(r)
	for sc.Scan() {
		buf = append(buf[:0], sc.Bytes()...) // reset len, keep cap
		_ = handle(buf)
	}
	return sc.Err()
}

func handle(line []byte) error { _ = line; return nil }
```

`buf[:0]` is the idiomatic "truncate to empty without freeing memory."

### 2. Deletion preserving order (since 1.21)

```go
import "slices"

s := []int{1, 2, 3, 4, 5}
s = slices.Delete(s, 1, 3) // remove indexes 1..2
// s == [1 4 5]
```

Pre-generics this was `s = append(s[:i], s[j:]...)` — same operation, just less obvious.

### 3. Unordered delete (O(1))

```go
func deleteFast[T any](s []T, i int) []T {
	last := len(s) - 1
	s[i] = s[last]
	var zero T
	s[last] = zero // important: avoid leaking a pointer
	return s[:last]
}
```

Zeroing the tail matters when `T` contains pointers — otherwise the deleted element is still reachable from the backing array and GC can't collect it.

### 4. Stack of T with explicit pool

```go
type Stack[T any] struct{ s []T }

func (st *Stack[T]) Push(v T) { st.s = append(st.s, v) }
func (st *Stack[T]) Pop() (T, bool) {
	if len(st.s) == 0 {
		var zero T
		return zero, false
	}
	n := len(st.s) - 1
	v := st.s[n]
	var zero T
	st.s[n] = zero
	st.s = st.s[:n]
	return v, true
}
```

### 5. Producing a defensive copy

```go
func (e *Event) Tags() []string {
	return slices.Clone(e.tags) // caller can mutate without affecting Event
}
```

`slices.Clone` is `append([]T(nil), s...)` written clearly.

## Anti-Patterns & Gotchas

**Returning a sub-slice of a giant buffer.** Holding `s[:10]` of a 10MB read keeps the whole 10MB alive. Fix: `slices.Clone(s[:10])` to copy out.

**`append` reused the array, my other slice changed.** See "aliasing trap" above. Use the three-index slice or `slices.Clip`.

**Forgetting `s = append(s, ...)`.** `append` returns a (potentially new) header. Always assign.

**`for _, v := range s { v.field = x }`.** `v` is a copy. Use `s[i].field = x`.

**Comparing slices with `==`.** Compile error. Use `slices.Equal`.

**Using a slice as a map key.** Compile error — slices aren't comparable. Use an array or a string.

**Mutating an element of a slice held by `[]any` after type assertion to a value type.** The assertion copies. Mutate via pointer.

**Trusting `cap` to predict growth.** Growth strategy is an implementation detail. Don't write tests that pin it.

## Performance Notes

- Pre-size with `make([]T, 0, n)` when `n` is known. Each grow is `len*sizeof(T)` of `memmove` plus an allocation.
- For element types containing pointers, the backing array is scanned by the GC. Smaller capacities = less scan work.
- `copy(dst, src)` is a `memmove`. Faster than a loop. Returns `min(len(dst), len(src))`.
- `append(a, b...)` is fastest when `cap(a) >= len(a)+len(b)`; otherwise it allocates and `memmove`s.
- `string(byteSlice)` and `[]byte(s)` allocate. The compiler optimizes a few specific call sites (`m[string(b)]`, `for _, r := range string(b)`) to avoid the alloc.
- Slice-of-pointer (`[]*T`) vs slice-of-struct (`[]T`): the latter is denser, friendlier to cache, but each element copy is the full struct size. Profile before deciding.

## How Big Companies Use It

- **Discord's 1.21 GC blog** profiles slice-of-pointer-vs-slice-of-value trade-offs in their state service: https://discord.com/blog/why-discord-is-switching-from-go-to-rust (note this also describes why slices weren't enough).
- **Cloudflare's `golog`/`golz4` byte-slice reuse patterns** are textbook examples of avoiding GC pressure via `s[:0]` re-truncation.
- **Kubernetes `pkg/util/slice`** and the `apimachinery` repo are saturated with slice manipulation — read [`staging/src/k8s.io/apimachinery/pkg/util/sets`](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/apimachinery/pkg/util/sets) for production-grade examples.
- **CockroachDB's `util/buildutil`** uses bump-allocator-like patterns over a giant `[]byte`.
- **InfluxDB's TSM engine** stores time-series points in `[]byte` and slices them with explicit length prefixes to minimize allocs.

## Source Code References

Pinned to `go1.26` — substitute the actual release tag when consulting.

- Growth algorithm and `growslice`: [`src/runtime/slice.go`](https://github.com/golang/go/blob/master/src/runtime/slice.go) — read `growslice`, `roundupsize`, and `nextslicecap`.
- `copy` builtin: same file, function `slicecopy`.
- `append` codegen (compiler): [`src/cmd/compile/internal/ssagen/ssa.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go), search `OAPPEND`.
- `slices` package: [`src/slices/slices.go`](https://github.com/golang/go/blob/master/src/slices/slices.go).
- `unsafe.Slice` / `unsafe.SliceData`: [`src/unsafe/unsafe.go`](https://github.com/golang/go/blob/master/src/unsafe/unsafe.go).

## Further Reading

- Go blog, "Go Slices: usage and internals": https://go.dev/blog/slices-intro
- Go blog, "Arrays, slices (and strings): The mechanics of 'append'": https://go.dev/blog/slices
- Russ Cox, "Go Data Structures": https://research.swtch.com/godata
- Dave Cheney, "Slices from the ground up": https://dave.cheney.net/2018/07/12/slices-from-the-ground-up
- Damian Gryski, "go-perfbook" on slice perf: https://github.com/dgryski/go-perfbook
- Proposal for `slices` package: https://go.dev/issue/45955

## Exercises / Self-Check

1. Given `a := []int{1,2,3,4,5}` and `b := a[1:3]`, what does `cap(b)` print? Why?
2. Write a function `Dedup[T comparable](s []T) []T` that removes adjacent duplicates in place and returns the truncated slice.
3. Use `slices.Clip` to fix this leak: a function that takes `data []byte`, returns `data[:headerLen]`, and the caller holds the result forever.
4. Why does `append([]int(nil), 1, 2, 3)` work? What is the header before and after?
5. Implement an O(1) delete-unordered. Why must you zero the tail element if `T = *Foo`?
