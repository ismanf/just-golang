# Stack vs Heap — Escape Analysis

## TL;DR

Go allocates on the **stack** by default and on the **heap** only when escape analysis proves a value could outlive the function. The decision is made by the compiler, not the programmer — there's no `new` vs `make` distinction at the language level (both can land on either). Read the analysis with `go build -gcflags=-m=2`. The single biggest gotcha: **storing into an interface, channel, map, slice header, or any global makes the value escape**, no matter how innocent the call looks. Stack allocation costs ~1 ns; heap allocation costs ~30 ns plus GC pressure. Eliminating escapes is the single biggest performance lever in idiomatic Go.

## Mental Model

```
Function call stack:                    Heap (managed by GC):
+--------------------------+            +----------------------+
| caller frame             |            |  span (class N)      |
|   - locals (cheap)       |            |  +---+---+---+---+   |
|   - return addr          |            |  |obj|obj|obj|obj|   |
+--------------------------+            |  +---+---+---+---+   |
| callee frame             |            +----------------------+
|   - locals (cheap)       |                ^
|   - large arrays?     ───┼──may escape────+
+--------------------------+

Each goroutine has its own stack (starts at 8 KiB, grows up to 1 GiB).
Stack frames pop on return — values inside die instantly, no GC.

A value escapes when the compiler cannot prove it dies with the frame.
```

The compiler's escape analyzer is a pointer-flow inference over SSA. It tracks where each address goes: parameters → return values, struct fields, channel sends, interface boxes, closures, package-level variables. Any path that leaves the function's call tree forces a heap allocation.

## Syntax & Basic Usage

```go
package main

import "fmt"

type Point struct{ X, Y int }

// stays on the stack: p never escapes
func sumStack() int {
	p := Point{X: 1, Y: 2}
	return p.X + p.Y
}

// escapes: returning a pointer to a local
func newPointEscapes() *Point {
	p := Point{X: 1, Y: 2}
	return &p
}

func main() {
	fmt.Println(sumStack(), newPointEscapes())
}
```

Inspect:

```
$ go build -gcflags=-m=2 .
./main.go:8:2: p does not escape
./main.go:14:2: moved to heap: p
```

The reported "moved to heap" is canonical phrasing. `=-m=2` gives the reasons; `-m=1` is one line per decision.

## Deep Dive

### What forces an escape

The compiler is conservative: when it can't prove a value's lifetime is bounded by the function, it heap-allocates. Common triggers:

1. **Returning a pointer** to a local.
2. **Assigning to an `interface{}` / `any`**, including printf's `...any`. The interface box stores a pointer; the value must outlive the call frame because the interface itself might.
3. **Sending into a channel.** The receiver is in another goroutine, so the sent value must live independently.
4. **Capturing by a closure** that escapes (returned, sent, stored). A closure that's only called locally and not stored is itself stack-allocated.
5. **Storing into a slice or map.** The backing array is on the heap; values you place there go with it (for elements that are themselves pointers, the *pointee* must live as long as the slice).
6. **Calling a method on an interface value** where the method receiver is the local — the dynamic dispatch loses static information.
7. **Variadic `...any`.** `fmt.Println(x)` heap-allocates an `[]any{x}` and an interface for `x`.
8. **`unsafe.Pointer` round-trips** that the analyzer can't follow.
9. **Large stack frames.** Anything > a per-architecture threshold (~10 MiB) gets moved to heap to keep stacks bounded.
10. **Address taken and passed to a non-leaf call** that the analyzer can't inline through.

### What doesn't force an escape

- Returning a value (not a pointer).
- Calling a *concrete*, inlinable method.
- Passing a pointer to a function that doesn't store it anywhere outside its frame ("pointer doesn't escape" — visible in `-m=2` as `p does not escape`).
- Stack-allocated closures whose captures don't outlive the closure call.
- Slices and maps that are local *and* don't escape themselves: `m := make(map[int]int)` inside a function that doesn't return or store `m` can stay on the stack (since Go 1.5 for small maps, broader since 1.20 with smaller backing).

### Reading `-gcflags=-m`

```
$ go build -gcflags="-m=2 -l" pkg.go   # -l disables inlining for clarity
./pkg.go:5:6: cannot inline f: marked go:noinline
./pkg.go:6:2: p escapes to heap:
./pkg.go:6:2:   flow: ~r0 = &p:
./pkg.go:6:2:     from &p (address-of) at ./pkg.go:8:9
./pkg.go:6:2:     from return &p (return) at ./pkg.go:8:2
```

The `flow:` lines trace the pointer's path step-by-step. If you can break any link in the chain (don't return the pointer, don't store it in an interface, etc.), the escape disappears.

### Stack growth

Goroutine stacks start at **8 KiB** (since 1.4; was 2 KiB in 1.2). Each function prologue checks `if SP < stackguard0 { morestack() }`. If the check fails, the runtime allocates a stack twice as large, copies frames, fixes pointers in those frames, and resumes. This is **contiguous stack** (since 1.4). Max stack size is 1 GiB on 64-bit (`runtime.SetMaxStack`).

The copy involves rewriting *all pointers into the old stack*. Doable because the compiler emits stack maps describing where pointers live in each frame. See `src/runtime/stack.go`, function `copystack`.

Stack growth is amortized O(1) per allocation but a single growth can be a hundred microseconds. If you see `runtime.morestack` in profiles, recursion or large frame allocation is suspect.

### Stack shrinking

If a stack uses less than 1/4 of its capacity after GC, the runtime halves it. This prevents the "rich getting richer" effect where one big call permanently inflates every goroutine.

### Heap allocation cost breakdown

```
mallocgc(size, type, needzero):
  ~10 ns   classify, mcache lookup, bump freelist
  ~5 ns    zero memory (needzero=true)
  ~10 ns   write GC bitmap bits
  ~5 ns    sampling, profiling hooks
```

Plus the *amortized* cost of GC scanning the object on every mark phase. For a noscan tiny object, ~20 ns total. For a pointer-bearing 1 KiB object, the marginal cost extends into GC.

Stack allocation is just `SP -= size` — a single arithmetic op, sub-nanosecond.

### Escape analysis is not whole-program

Each package is analyzed independently. Calls into other packages assume the worst (parameters escape) unless the callee is *inlined*. This is why `//go:noinline` annotations or large bodies that don't get inlined can suddenly cause escapes that disappear after refactoring.

Profile-guided optimization (since 1.20) can hint the compiler to inline hot calls, indirectly fixing escapes. See `12-runtime/13-pgo.md`.

### The `runtime.KeepAlive` exception

If you need a heap value to *not* be collected before a certain point (e.g., a file descriptor referenced via `unsafe`), `runtime.KeepAlive(x)` forces the analyzer to treat `x` as live until that point. It doesn't change escape; it changes the GC's view of liveness.

### Special pseudo-functions that don't escape arguments

The compiler hard-codes "leaf" status for a few well-known functions even without inlining. As of 1.26:
- `len`, `cap`, `unsafe.Sizeof`, `unsafe.Alignof`, `unsafe.Offsetof`, `runtime.KeepAlive`.
- Some intrinsics (`bits.LeadingZeros64`, etc.).

### When the analyzer is wrong (or you wish it were)

It's never *unsound* — it always over-approximates. But it over-approximates: a function that *would* be safe on the stack might be heap-allocated because the analyzer can't prove safety. Workarounds:

1. **Inline manually** by writing the call site inline.
2. **Use a sized stack buffer** and bound the slice: `var buf [256]byte; s := buf[:n]`. The backing is on stack; the slice header is too if the slice itself doesn't escape.
3. **Pass a pre-allocated destination** instead of returning a new value.
4. **Avoid `any` in hot paths.** If you must, use generics — type parameters preserve the concrete type through the call, often turning would-be escapes into stack stays.

## Standard Library Hooks

- `runtime.KeepAlive(x any)` — extend visibility of `x` to GC up to this call.
- `runtime.SetMaxStack(n int) int` — change the 1 GiB stack cap.
- `runtime.Stack(buf []byte, all bool)` — dump stacks (also see `-gcflags=-m`).
- `runtime/debug.SetGCPercent(-1)` — disable GC, useful for measuring allocator-only overhead in benchmarks.
- `runtime.MemStats.StackInuse` — total stack memory in use across all goroutines.
- `runtime.MemStats.Mallocs / Frees` — cumulative heap allocation counts.

## Real-World Patterns

### 1. Reuse a buffer to keep it off the heap

```go
package main

import "fmt"

type Writer struct {
	buf [256]byte
}

// w.buf stays on Writer's allocation — no per-call allocation.
func (w *Writer) Format(x int) string {
	n := fmtInt(w.buf[:0], x)
	return string(w.buf[:n])
}

func fmtInt(b []byte, x int) int {
	// imagine itoa here
	b = append(b, '4', '2')
	_ = x
	return len(b)
}

func main() {
	var w Writer
	fmt.Println(w.Format(42))
}
```

`fmt.Sprintf("%d", x)` allocates twice (the `[]any{x}` and the result string). The buffer-reuse version allocates once (the `string` conversion).

### 2. Generic helper avoids the `any` box

```go
package main

import "fmt"

// any-based: x is boxed, escapes.
func PrintAny(x any) { fmt.Println(x) }

// generic: T preserved; for value types, no boxing in the call.
func Print[T any](x T) { fmt.Println(x) } // still boxes inside Println,
                                          // but the caller-side box is gone

func main() {
	Print(42)
	Print("hi")
}
```

```
$ go build -gcflags=-m=2 ./...
./main.go:6:13: leaking param: x
./main.go:9:13: x does not escape
```

The improvement is at the *call site*; inside `fmt.Println` the value still escapes because that's how `fmt` works. For full payoff, write the formatter without `any`.

### 3. Pre-allocate, pass-in, no return-by-pointer

```go
package main

import "encoding/json"

type Doc struct {
	Title string
	Body  string
}

// Returns a pointer — d escapes.
func ParseBad(b []byte) (*Doc, error) {
	var d Doc
	return &d, json.Unmarshal(b, &d)
}

// Caller owns storage — no escape.
func ParseGood(b []byte, d *Doc) error {
	return json.Unmarshal(b, d)
}
```

The unmarshal target's pointer escapes anyway because `json.Unmarshal` is a reflection black-box, but the *struct itself* in `ParseGood` lives on the caller's frame, which is where you want it (typically a `for` loop reusing the same `Doc`).

### 4. Use the sized array trick for known bounds

```go
package main

import "encoding/hex"

func ToHex(x [16]byte) string {
	var out [32]byte // stays on stack
	hex.Encode(out[:], x[:])
	return string(out[:]) // single allocation (the string)
}
```

`out` is `[32]byte`, a value type. The slice header `out[:]` is also stack-resident because it doesn't escape past the `hex.Encode` call.

### 5. Stack-allocated closures

```go
package main

func sumEach(xs []int, f func(int)) {
	for _, x := range xs {
		f(x)
	}
}

func main() {
	total := 0
	sumEach([]int{1, 2, 3}, func(x int) { total += x }) // closure stays on stack
	_ = total
}
```

The closure captures `total` by reference. Because the closure value isn't stored or returned, the analyzer keeps it (and the captured variable) on the stack. If `sumEach` instead stored `f` in a global, both would escape.

## Anti-Patterns & Gotchas

**`fmt.Sprintf` in a hot loop.** Each call allocates the `...any` slice plus interface boxes plus the result. Use `strconv` directly or a reused `bytes.Buffer`.

**`json.Unmarshal` into a `*new(T)` returned from a function.** The `T` escapes both because of the pointer return and because `Unmarshal` uses reflection. Pass a caller-owned `*T`.

**Storing values in an `error` interface that wraps them in a struct.** The wrapped value escapes; the error itself often escapes too (it's returned). For "rich" errors, this is fine; for hot paths, prefer sentinel errors.

**Using `defer` to "clean up small things" in tight loops.** `defer` itself escapes its argument slice (the defer record is heap-allocated unless the function has open-coded defers, which only kick in for ≤8 simple cases). Move the cleanup to the loop's exit explicitly.

**Variadic functions called with a single value.** `func g(xs ...int)` called as `g(x)` allocates a 1-element slice. Use overloads or accept the cost.

**Believing inlining auto-fixes escapes.** Inlining helps because the analyzer sees through the call boundary — but only if the call is *actually* inlined. Functions over the 80-budget threshold aren't, and stay opaque. Use `//go:noinline` removal, smaller helpers, or PGO.

**Reading too much into `does not escape` without `=2`.** A `-m=1` "does not escape" can hide a different value that does. Use `=2` and trace each flow.

**Stack growth in deeply recursive code.** Each `morestack` is expensive. Tail-call-style iteration in a loop is much cheaper. Go has no TCO; rewrite recursion as iteration.

**Trying to "force" stack allocation via `runtime`-level hacks.** There isn't one. Either restructure to satisfy the analyzer, or accept the heap.

**`for ... range` over a channel allocates per-iteration if the value type is large and the receiver is a pointer.** Reuse the receive variable explicitly; consider chunking.

## Performance Notes

- Stack allocation: <1 ns (just SP adjustment).
- Heap allocation hot path: ~15–30 ns small objects, ~µs large.
- `morestack` (stack growth): hundreds of µs per growth.
- GC marginal cost per pointer-bearing word: ~1 ns per mark cycle.
- Interface boxing: ~10 ns (heap alloc + assignment) for non-pointer values.
- `fmt.Sprintf("%d", x)`: ~80 ns, 2 allocs.
- `strconv.AppendInt(b, x, 10)`: ~10 ns, 0 allocs.
- Channel send of pointer: no extra escape if value already heap; for stack-only value, send forces escape.

The fastest way to assess: `go test -bench=. -benchmem`. Watch `allocs/op` and `B/op`. Anything > 0 in a hot inner loop is suspicious.

## How Big Companies Use It

- **Discord** documents how their state service's pointer-heavy heap layout slammed GC mark times; the fix involved removing pointers from inner structs to keep them on noscan spans, and reducing heap churn by reusing pools: https://discord.com/blog/why-discord-is-switching-from-go-to-rust.
- **CockroachDB** documents per-RPC `*Args` reuse via `sync.Pool` and explicit `Reset()` methods to avoid escape from request handlers: https://www.cockroachlabs.com/blog/the-cost-of-allocation/.
- **Cloudflare** publishes profiles of their edge proxy with detailed `-gcflags=-m` triage; their proxy team has multiple posts on shaving allocations from request hot paths: https://blog.cloudflare.com/go-don-t-collect-my-garbage/.
- **Uber's Zap logger** is built around avoiding allocations during logging: pre-sized buffers, no `fmt`, no `any`-encoding. Benchmarks at https://github.com/uber-go/zap.
- **Google's Vitess** (sharded MySQL proxy) uses generics and pool patterns throughout to keep request handling free of avoidable escapes.
- **Tailscale** routinely shares optimization stories, including escape-driven refactors: https://tailscale.com/blog/.

## Source Code References

Pinned to `go1.26`.

- Escape analysis: [`src/cmd/compile/internal/escape/escape.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/escape/escape.go).
- Escape graph utilities: [`src/cmd/compile/internal/escape/graph.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/escape/graph.go).
- Stack copy / growth: [`src/runtime/stack.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/stack.go) — read `copystack`, `newstack`, `shrinkstack`.
- Stack maps (pointer locations per frame): [`src/runtime/mbitmap.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mbitmap.go) and emitted by [`src/cmd/compile/internal/liveness`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/liveness).
- `runtime.KeepAlive`: [`src/runtime/mfinal.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/mfinal.go).
- Stack allocation thresholds (`maxStackVarSize`, `maxImplicitStackVarSize`): [`src/cmd/compile/internal/escape/utils.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/escape/utils.go).

## Further Reading

- "Allocation efficiency in high-performance Go services" — Achille Roussel, Segment: https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/
- Dave Cheney, "Five things that make Go fast" (escape analysis section): https://dave.cheney.net/2014/06/07/five-things-that-make-go-fast
- Damian Gryski, "go-perfbook — heap and escape": https://github.com/dgryski/go-perfbook
- Bryan C. Mills, "Escape analysis in Go" thread (golang-nuts): https://groups.google.com/g/golang-nuts/c/0jUbeWHnVlw
- Russ Cox, "Goroutines, threads, and stacks": https://research.swtch.com/gostack
- Keith Randall, "Inside the Go compiler — the SSA backend" (GopherCon 2018): https://www.youtube.com/watch?v=uTMvKVma5ms
- Austin Clements, "Go's regabi" — register-based calling convention; relevant because the new ABI changed escape decisions: https://go.googlesource.com/proposal/+/master/design/40724-register-calling.md

## Exercises / Self-Check

1. Write a function that returns a `string` formatted from an `int` using zero heap allocations. Verify with `-benchmem`.
2. Take a program that builds a `map[string]int` via `m[s] = v` where `s = string(b)` and `b` is `[]byte`. Why does the compiler not allocate the string?
3. Add `//go:noinline` to a small helper that previously allowed the caller's locals to stay on the stack. Predict the new escape behavior, then confirm with `-m=2`.
4. Recursively walk a tree of `*Node`. The walk function takes a closure callback. Where does the closure live? When does it escape?
5. Set `GOGC=off` and benchmark a program that allocates 1M small objects vs one that uses a pool. Compare absolute allocator throughput separated from GC cost.
