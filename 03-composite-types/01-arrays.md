# Arrays

## TL;DR

Arrays in Go are **fixed-size, value-typed** sequences of homogeneous elements. The size is part of the type — `[3]int` and `[4]int` are different types. You almost never want a bare array in application code; you want a slice. Arrays show up when size is a compile-time invariant: cryptographic digests, fixed protocol headers, lookup tables, and as the backing storage you take a slice over.

## Mental Model

```
[5]int
+----+----+----+----+----+
|  0 |  0 |  0 |  0 |  0 |   <-- a single value, copied on assignment
+----+----+----+----+----+
 idx0 idx1 idx2 idx3 idx4
```

An `[N]T` is laid out contiguously, sized exactly `N * sizeof(T)`, and lives wherever you put it (stack, heap inside a struct, etc.). It is **not** a reference type — passing an array to a function copies every element.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	var zeroed [5]int                  // [0 0 0 0 0]
	literal := [3]string{"a", "b", "c"}
	inferred := [...]int{1, 2, 3, 4}   // length deduced -> [4]int
	sparse := [10]int{2: 99, 7: 42}    // index:value pairs

	fmt.Println(zeroed, literal, inferred, sparse)
	fmt.Printf("len(inferred)=%d cap(inferred)=%d\n", len(inferred), cap(inferred))
	// Output:
	// [0 0 0 0 0] [a b c] [1 2 3 4] [0 0 99 0 0 0 0 42 0 0]
	// len(inferred)=4 cap(inferred)=4
}
```

`len` and `cap` both return the (compile-time-known) length. The compiler folds them to constants.

## Deep Dive

### Value semantics

Every assignment, function parameter, and `range` copies the whole array.

```go
package main

import "fmt"

func zero(a [4]int) { a[0] = 0 }

func main() {
	x := [4]int{1, 2, 3, 4}
	zero(x)
	fmt.Println(x[0]) // 1 — x was copied
	// Output: 1
}
```

Pass `*[4]int` (or a slice) if you want the callee to mutate the caller's data. Pointers to arrays are *not* generic — `*[4]int` and `*[5]int` are still different types.

### Size is part of the type

```go
var a [3]int
var b [4]int
// a = b // compile error: cannot use b (variable of type [4]int) as [3]int value
```

This is why generic code over "an array of any length" needs either a slice (`[]int`) or, since 1.18, a type parameter for the length:

```go
func Sum[N int, T ~int | ~float64](a [N]T) T {
	// N is a type parameter only on the type — Go does NOT support const generics
	// directly. The N here is just a regular type parameter; this won't compile.
	// See "Anti-Patterns" below.
	var s T
	for _, v := range a {
		s += v
	}
	return s
}
```

Go has **no const-generics**. You cannot parameterize a function over an array length. The idiomatic move is to take a slice.

### Comparability

Arrays are comparable iff their element type is comparable. `==` compares element-wise:

```go
[3]int{1, 2, 3} == [3]int{1, 2, 3} // true
[2]string{"a", "b"} == [2]string{"a", "b"} // true
// [2]map[string]int{} == [2]map[string]int{} // compile error: maps not comparable
```

This makes arrays usable as map keys — a real-world trick for caching by fixed-size keys (e.g. SHA-256 digests, IPv6 addresses pre-`netip`).

### Range produces index + value copy

```go
arr := [3]struct{ x int }{{1}, {2}, {3}}
for i, v := range arr {
	v.x = 99       // mutates the copy, not arr[i]
	_ = i
}
// arr is unchanged
```

Use `arr[i]` directly, or `range &arr` to range over a pointer (still copies the element though). To mutate in place: `for i := range arr { arr[i].x = 99 }`.

## Standard Library Hooks

- `crypto/sha256.Sum256(data []byte) [32]byte` — returns an array, not a slice, so it's stack-allocatable and hashable as a map key.
- `crypto/md5.Sum`, `crypto/sha1.Sum`, `crypto/sha512.Sum512` — same pattern.
- `net/netip.Addr` — wraps `[16]byte` internally to hold v4 or v6 without an allocation.
- `encoding/binary.Read` / `Write` — work with fixed-size arrays for wire formats.

## Real-World Patterns

### 1. Hash as map key

```go
package main

import (
	"crypto/sha256"
	"fmt"
)

type Cache map[[32]byte][]byte

func (c Cache) Put(payload []byte) {
	key := sha256.Sum256(payload) // [32]byte, no alloc
	c[key] = payload
}

func main() {
	c := Cache{}
	c.Put([]byte("hello"))
	fmt.Println(len(c)) // Output: 1
}
```

Because `[32]byte` is comparable, you can use it directly as a map key. A slice (`[]byte`) cannot be a key.

### 2. Lookup table baked into the binary

```go
package crc

// crc8Table is a precomputed 256-entry lookup, sized exactly at compile time.
// As an array, it lives in the read-only data segment; as a slice, it would
// need a backing array allocated at init.
var crc8Table = [256]byte{
	0x00, 0x07, 0x0e, 0x09, /* ... 252 more ... */
}

func Update(crc byte, p []byte) byte {
	for _, b := range p {
		crc = crc8Table[crc^b]
	}
	return crc
}
```

### 3. Fixed-size ring buffer with no allocations

```go
type Ring[T any] struct {
	buf  [64]T
	head int
	tail int
}

func (r *Ring[T]) Push(v T) bool {
	next := (r.head + 1) % len(r.buf)
	if next == r.tail {
		return false
	}
	r.buf[r.head] = v
	r.head = next
	return true
}
```

The `[64]T` lives inline inside the `Ring` struct — no separate heap allocation for the buffer.

## Anti-Patterns & Gotchas

**Passing huge arrays by value.** `func process(blob [1 << 20]byte)` copies a megabyte on every call. Pass `*[1 << 20]byte` or `[]byte`.

**Trying to parameterize by length.** Go has no const-generics. This does not compile:

```go
// func Pad[N int](s string) [N]byte { ... } // INVALID
```

Use a slice and check length at runtime, or use code generation for each fixed size.

**Confusing `[...]T{...}` with `[]T{...}`.** The first is an array, second a slice. Easy to misread.

**Mutating in `for _, v := range`.** `v` is a copy; assignments to `v` or `v.Field` don't reach back into the array.

**Using arrays where you mean slices.** Most APIs (`io.Reader`, `bytes.Buffer`, JSON encoding) want `[]byte`/`[]T`. Don't force arrays into them — `arr[:]` works but signals "this should have been a slice."

## Performance Notes

- Arrays smaller than the escape threshold stay on the stack. The compiler is aggressive here.
- Copying is a `memmove`; cost scales linearly. For `[8]int` it's effectively free, for `[1<<16]int` it isn't.
- Array compare is unrolled by the compiler for small sizes — equality of two `[16]byte` is a couple of MOV+CMP instructions.
- Array indexing has bounds checks. The compiler frequently elides them when the index is provably in range (`for i := range a`).

Inspect escape decisions with:

```
go build -gcflags='-m=2' ./...
```

## How Big Companies Use It

- **Tailscale** uses `netip.Addr` (a wrapper around `[16]byte`) end-to-end so address values are comparable, allocation-free, and safe to use as map keys. See `net/netip` design rationale: [go.dev/blog/netip-package](https://go.dev/blog/netip-package).
- **Cloudflare** uses fixed-size arrays in their TLS code (`crypto/tls` upstream contributions) for `ClientRandom`, `ServerRandom`, session ticket keys — all `[32]byte`.
- **CockroachDB** uses array-of-byte hashes as keys in their MVCC scan code.
- **Kubernetes** uses `[16]byte` UUIDs internally before converting to string for the API.

## Source Code References

Pinned to `go1.26` (substitute the actual release tag when writing).

- Runtime array type descriptor: [`src/runtime/type.go`](https://github.com/golang/go/blob/master/src/runtime/type.go) — `arraytype` struct.
- Array equality codegen: [`src/cmd/compile/internal/ssagen/ssa.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go), look for `OEQ`/`ONE` of `TARRAY`.
- `crypto/sha256.Sum256` returning `[Size]byte`: [`src/crypto/sha256/sha256.go`](https://github.com/golang/go/blob/master/src/crypto/sha256/sha256.go).
- `netip.Addr` layout: [`src/net/netip/netip.go`](https://github.com/golang/go/blob/master/src/net/netip/netip.go) — uses `uint128` (two uint64) but the spirit is identical.

## Further Reading

- Go spec, "Array types": https://go.dev/ref/spec#Array_types
- "Go Slices: usage and internals" — explains arrays as the foundation for slices: https://go.dev/blog/slices-intro
- Russ Cox, "Go Data Structures": https://research.swtch.com/godata
- Dave Cheney, "Arrays, slices and copy": https://dave.cheney.net/2018/07/12/slices-from-the-ground-up

## Exercises / Self-Check

1. Write a function `Reverse[T any](a *[8]T)` that reverses in place. Why does it take a pointer?
2. Benchmark `Sum([1024]int)` passed by value vs by pointer. At what size does the copy cost become measurable on your machine?
3. Why can `[3]int` be a map key but `[3][]int` cannot?
4. Use `go build -gcflags='-m'` on a function that returns a local `[16]byte`. Does it escape? Why or why not?
5. Implement a fixed-capacity LRU using two arrays (no slices, no maps) — what trade-offs do you hit?
