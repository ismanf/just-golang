# Pointers

## TL;DR

A pointer in Go is a typed memory address. There is **no pointer arithmetic** (without `unsafe`), no implicit conversions between pointer types, and a `nil` pointer is the zero value of every pointer type. The compiler decides where the pointed-to value lives — stack or heap — via **escape analysis**, not via syntax. You take pointers to mutate, to share (avoid copying a big struct), to express "absent" with `nil`, and to satisfy method sets that require a pointer receiver. Methods on `nil` pointers compile and run — they only panic if they dereference.

## Mental Model

```
x := 42                  ┌──────────────────┐
p := &x          ─────►  │   x: int = 42    │  on stack (or heap, if x escapes)
                         └──────────────────┘
                                ▲
                                │
                   p: *int  ────┘

*p == 42       // dereference reads
*p = 7         // dereference writes (x is now 7)
p = nil        // safe; *p afterwards is a runtime panic
```

A pointer is a single word (8 bytes on 64-bit). The "escape" question — does the pointed-to value live on the stack or heap — is decided by the compiler, not the keyword you used.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	x := 10
	p := &x          // p has type *int
	*p = 20          // mutates x
	fmt.Println(x)   // 20

	var np *int      // nil
	fmt.Println(np == nil) // true

	n := new(int)    // allocates an int, returns *int (n != nil, *n == 0)
	*n = 99

	s := &struct{ A int }{A: 5} // composite literal address: alloc + ptr
	fmt.Println(s.A) // 5 (no need to write (*s).A — Go auto-derefs for fields)

	fmt.Println(*n, s.A)
	// Output:
	// 20
	// true
	// 99 5
}
```

`&T{...}` is more idiomatic than `new(T)` followed by field assignments.

## Deep Dive

### No pointer arithmetic

```go
p := &arr[0]
p++              // compile error
p = p + 1        // compile error
```

The escape hatch is `unsafe.Pointer` and conversions through `uintptr` — see the `unsafe` page for the six valid patterns.

### Auto-dereference for fields and methods

```go
type S struct{ X int }
func (s *S) Bump() { s.X++ }

p := &S{}
p.X       // == (*p).X — Go inserts the deref
p.Bump()  // ok
```

Same goes for methods with value receivers — Go takes the address for you. But only when the operand is **addressable**. `f().Method()` where `f` returns a value and `Method` is on `*T` is a compile error.

### Pointer receivers vs value receivers

```go
type Counter struct{ n int }

func (c Counter) ValueBump()  { c.n++ }  // mutates a copy; pointless
func (c *Counter) PtrBump()   { c.n++ }  // mutates the original

var c Counter
c.ValueBump() // c.n still 0
c.PtrBump()   // c.n == 1
```

Rules of thumb:
- If the method mutates, use `*T`.
- If `T` is large, use `*T` to avoid the copy.
- If `T` contains a mutex or any `sync.*` field, use `*T` (copying a mutex is a bug).
- Be consistent within a type: don't mix value and pointer receivers.
- If `T` should satisfy an interface, the **method set of `*T` includes both value and pointer methods, but the method set of `T` only includes value methods.** This matters for interface satisfaction.

### Nil pointer methods

```go
type Logger struct{ prefix string }
func (l *Logger) Log(msg string) {
	if l == nil {
		return // null object pattern
	}
	fmt.Println(l.prefix, msg)
}

var l *Logger
l.Log("hi") // compiles, runs, prints nothing
```

A method call on a nil pointer compiles. It only panics if the body dereferences `l` (e.g., `l.prefix`). This enables a lightweight null-object pattern.

### Escape analysis

```go
func newInt() *int {
	x := 5
	return &x  // x escapes to the heap
}
```

The compiler proves `x` outlives the call frame and allocates it on the heap. Inspect with:

```
go build -gcflags='-m' ./...
# ./main.go:NN:5: moved to heap: x
```

Common escape triggers: returning `&local`, storing `&local` in an interface, sending `&local` on a channel. Often, escape doesn't matter — but in hot paths it does.

### Pointers and interfaces

Storing a pointer in an interface boxes the pointer. `nil` of a concrete pointer type stored in an interface is **not** equal to a `nil` interface:

```go
var p *int       // nil *int
var i any = p    // non-nil interface holding (type=*int, value=nil)
fmt.Println(i == nil) // false — this is the famous "typed nil" trap
```

Don't return `var p *MyError; return p` from a function whose return type is `error`. Return `nil` explicitly.

### `new` vs `&T{}`

```go
p1 := new(MyStruct)      // pointer to zero-valued MyStruct
p2 := &MyStruct{}        // same — pointer to zero-valued literal
p3 := &MyStruct{Name: "x"} // can't do this with new in one line
```

Prefer `&T{}` (or `&T{Field: v}`). `new` is mostly used for built-in types: `new(int)`, `new(bytes.Buffer)`.

### Comparability

```go
a, b := 1, 1
&a == &b           // false: different addresses
&a == &a           // true
p1 := &MyStruct{}
p2 := p1
p1 == p2           // true
p1 == nil          // false
```

Pointer equality compares addresses, not the values pointed to. To compare pointed-to values, dereference.

### Two pointers, same target

```go
x := 5
p := &x
q := &x
*p = 10
fmt.Println(*q) // 10
```

Mundane, but newcomers ask this constantly. Both pointers refer to the same memory.

## Standard Library Hooks

- `unsafe.Pointer` — bypass the type system. See the dedicated `unsafe` page.
- `reflect.PointerTo(t)` — get `*t`'s reflect.Type.
- `runtime.SetFinalizer(p, fn)` — attach a cleanup function. (Deprecated in favor of `runtime.AddCleanup` since 1.24.)
- `runtime.KeepAlive(p)` — keep an object reachable past where the optimizer might otherwise drop it.
- `atomic.Pointer[T]` (since 1.19) — typed atomic pointer ops.
- `weak.Pointer[T]` (since 1.24) — weak reference that doesn't prevent GC.

## Real-World Patterns

### 1. Optional fields via pointer

```go
type UpdateUser struct {
	Name  *string  // nil means "don't change"
	Email *string
	Age   *int
}

func ptr[T any](v T) *T { return &v }

req := UpdateUser{
	Email: ptr("new@example.com"),
}
```

Mirrors how `encoding/json` distinguishes "absent" (`nil`) from "present but zero" (`*Name == ""`).

### 2. Mutable receiver

```go
type Counter struct{ n int }
func (c *Counter) Inc()  { c.n++ }
func (c *Counter) Val() int { return c.n }
```

If you skip the pointer receiver here, `Inc` does nothing observable.

### 3. Atomic pointer for lock-free read-mostly cache

```go
import "sync/atomic"

type Config struct{ /* big */ }
var cfg atomic.Pointer[Config]

func Get() *Config       { return cfg.Load() }
func Replace(c *Config)  { cfg.Store(c) }
```

Many readers, occasional writer, zero locking on the read path.

### 4. Builder returning `*T` for chaining

```go
type Req struct { /* ... */ }
func (r *Req) WithHeader(k, v string) *Req { /* ... */ return r }

req := (&Req{}).WithHeader("X", "1").WithHeader("Y", "2")
```

### 5. Null-object via nil-safe methods

```go
type metric interface{ Inc() }

type noopMetric struct{}
func (*noopMetric) Inc() {}

var m metric
if cfg.MetricsEnabled {
	m = realMetric()
} else {
	m = (*noopMetric)(nil) // typed nil, but Inc is safe
}
m.Inc()
```

Use sparingly — the typed-nil gotcha lurks if `m` ever escapes to a caller checking `m == nil`.

## Anti-Patterns & Gotchas

**Typed nil in an interface.** `var p *T; return p` from a function returning `error` makes `err != nil` for the caller. Return literal `nil` instead.

**Returning pointer to a local that didn't need to escape.** If you only want to mutate within the function, don't return the address; you push the alloc onto the heap.

**Copying a struct that embeds `sync.Mutex` or any `sync.*`.** The mutex's state copies too — both copies will fight over the same lock semantics across different memory. `go vet` catches this.

**Pointer to map value.** `&m[k]` is a compile error. There's no way to take an address of a map element because the map can rehash and move it.

**Comparing pointer-of-struct values with `==` when you meant deep equality.** Use `*p1 == *p2` or `reflect.DeepEqual`.

**`&x` where you don't actually need a pointer.** Heap allocation, GC pressure. If the value is small and you don't need to mutate, pass by value.

**Sharing a slice's backing array via `&slice[0]`** and then resizing the slice. The pointer may dangle after `append` triggers a realloc. Use `unsafe.SliceData(s)` and pin the slice with `runtime.KeepAlive`.

**Storing pointers to loop-local variables before Go 1.22.** The classic bug fixed by the per-iteration scoping in 1.22.

**Premature heapification:** writing `func New() *T { return &T{} }` for tiny value types. Sometimes returning `T` is the right call.

## Performance Notes

- Pointers are 8 bytes on 64-bit. A slice of pointers is denser to GC scan than a slice of fat structs (in terms of bytes scanned per element), but each element costs an extra deref to access.
- Heap allocation costs: scan cost (GC), allocation cost (mcache fastpath: ~10ns), reachability tracking.
- Escape analysis is your friend. `go build -gcflags='-m=2'` reports decisions.
- Methods with pointer receivers are inlinable in the same conditions as value receivers; receiver type does not by itself prevent inlining.
- A struct of all pointers (`[]*Foo`) is GC-scanned word by word. A struct of all scalars (`[]Foo` where `Foo` has no pointers) is **not scanned at all**. This matters for billions of objects.
- `atomic.Pointer[T]` is faster than `atomic.Value` for the same use case because it's typed (no `interface{}` boxing).

## How Big Companies Use It

- **Discord's state service** GC pause issues stemmed from a `map[Snowflake]*Member` with many millions of pointers, each scanned every GC cycle. They switched to a value-typed representation to slash scan time. https://discord.com/blog/why-discord-is-switching-from-go-to-rust
- **Cloudflare** uses `atomic.Pointer` heavily for hot-reloadable configs in their proxy. See the `pingora` predecessor articles (the Go ones, before the Rust rewrite).
- **Kubernetes** uses `*ObjectMeta` and friends pervasively — every API object is heap-allocated and shared through interfaces. The `client-go` informer pattern depends on pointer identity for shared caches.
- **CockroachDB** uses `*Replica` with atomic pointer swaps for leader changes.
- **Tailscale's `tsnet`** uses `atomic.Pointer[Config]` for live reconfig of the embedded node.

## Source Code References

Pinned to `go1.26`.

- Pointer type descriptor: [`src/internal/abi/type.go`](https://github.com/golang/go/blob/master/src/internal/abi/type.go) — `PtrType`.
- `atomic.Pointer[T]` (since 1.19): [`src/sync/atomic/type.go`](https://github.com/golang/go/blob/master/src/sync/atomic/type.go).
- `weak.Pointer[T]` (since 1.24): [`src/weak/pointer.go`](https://github.com/golang/go/blob/master/src/weak/pointer.go).
- Escape analysis (compiler): [`src/cmd/compile/internal/escape`](https://github.com/golang/go/tree/master/src/cmd/compile/internal/escape).
- `runtime.KeepAlive`: [`src/runtime/mfinal.go`](https://github.com/golang/go/blob/master/src/runtime/mfinal.go).
- `runtime.AddCleanup` (since 1.24): same file.
- Typed nil interface trap reference (test): [`test/nil.go`](https://github.com/golang/go/blob/master/test/nil.go).

## Further Reading

- Spec, "Pointer types": https://go.dev/ref/spec#Pointer_types
- Spec, "Method sets": https://go.dev/ref/spec#Method_sets
- Dave Cheney, "Pointers in Go": https://dave.cheney.net/2017/04/26/understand-go-pointers-in-less-than-800-words
- Dave Cheney, "Should methods be declared on T or *T?": https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t
- Go FAQ on typed nil: https://go.dev/doc/faq#nil_error
- Go Memory Model on atomic pointers: https://go.dev/ref/mem
- Escape analysis (Vincent Blanchon): https://medium.com/a-journey-with-go/go-introduction-to-the-escape-analysis-f7610174e890

## Exercises / Self-Check

1. Why is `var p *MyError; return p` from a function returning `error` a bug? Demonstrate with a runnable example.
2. Write a function that returns the address of a local variable. Use `-gcflags=-m` to confirm it escapes.
3. Compare the GC scan time for `[]Foo` vs `[]*Foo` with `Foo struct{ X int }` and a million elements. Use `runtime.GC()` + `runtime.ReadMemStats`.
4. Define a type with both value and pointer receivers. Pass it to an interface accepting one of them. Which compiles and why?
5. Replace a `mutex + value` config pattern with `atomic.Pointer[Config]`. Measure the read-path latency before and after.
