# Bounds-Check Elimination

## TL;DR

Every slice, array, and string indexing operation in Go is bounds-checked: at runtime, the index is compared to the length, and an out-of-range panic is raised on failure. The compiler's **`prove` SSA pass** tracks value ranges and proves checks redundant where possible, deleting them. A typical hot loop can shed all its bounds checks if you write the loop *idiomatically*. The single biggest gotcha: **`prove` reasons over SSA values, not source variables** — slight rewrites that look equivalent ("`s[i+1]`" vs "`s[i] then s[i]`") can flip BCE on or off. Verify with `-gcflags=-d=ssa/check_bce/debug=1` or by reading `ssa.html`.

## Mental Model

```
   for i := 0; i < len(s); i++ {
       _ = s[i]    ← bounds check: i < len(s)
   }                ─────────  prove pass:
                              "i" was just shown < len(s); deleted.

   for i := range s {
       _ = s[i]   ← same; bound is the iterator invariant.
   }

   for i := 0; i < n; i++ {
       _ = s[i]   ← cannot eliminate unless prove knows n ≤ len(s).
   }              ─────────  Add `_ = s[n-1]` first to hoist the check.

   prove tracks facts:
       i ≥ 0                       (from the init)
       i < len(s)                  (from the loop cond)
       len(s) is non-negative      (always)
   Combined: s[i] is safe.
```

The bounds check itself is a tiny conditional + a call to `runtime.panicIndex` on failure. On a modern CPU it's typically a single fused-compare+branch; cost is dominated by the branch predictor. Cumulative cost in a tight loop is real (~10–30% on memory-bound numeric code).

## Syntax & Basic Usage

```go
package main

import "fmt"

func sum(s []int) int {
	var total int
	for i := 0; i < len(s); i++ {
		total += s[i]
	}
	return total
}

func main() {
	fmt.Println(sum([]int{1, 2, 3}))
}
```

Inspect BCE:

```bash
$ go build -gcflags='-d=ssa/check_bce/debug=1' .
./main.go:8:13: Found IsInBounds  # bounds check that's still emitted
```

`-d=ssa/check_bce/debug=1` lists every bounds check the compiler kept. The goal is "none in your inner loop".

## Deep Dive

### Where bounds checks come from

Every `a[i]`, `a[i:j]`, `a[i:j:k]`, and `string[i]` operation produces a compiler-inserted check. In SSA they show up as `IsInBounds` and `IsSliceInBounds` ops:

```
v10 = IsInBounds <bool> v_idx v_len  // i < len(a)?
If v10 then b_ok else b_panic
```

If the check passes, execution continues; otherwise the program calls `runtime.panicIndex` / `runtime.panicSliceB`. The fast path is one compare and one predicted-not-taken branch.

### The `prove` pass

`cmd/compile/internal/ssa/prove.go` performs **value-range analysis** over SSA. It maintains "facts" — relational invariants between SSA values — and propagates them through control flow. Key fact types:

- `v ≥ c` or `v > c` (relative to a constant).
- `v < u` (relative to another value).
- `v == c` (after a comparison).
- `v ∈ [lo, hi]` (range info, also used for unsigned overflow reasoning).

The pass walks the dominator tree. Each conditional branch contributes facts on its taken arm. When `prove` encounters an `IsInBounds v_idx v_len`, it tries to prove `0 ≤ v_idx < v_len` from the current fact set. If it succeeds, the check is rewritten to a constant `true`, and subsequent passes (`opt`, `deadcode`) prune it.

### What `prove` can reason about

- **Loop induction variables**: in `for i := 0; i < n; i++`, `prove` knows `0 ≤ i < n` throughout the body.
- **Length invariants**: `len(s) ≥ 0` always.
- **Earlier checks**: if a previous `s[k]` succeeded and the analyzer can show no aliasing changes `len(s)`, then `k < len(s)` is still true.
- **Slice expressions**: `t := s[a:b]` gives `len(t) == b - a` and `len(t) ≤ len(s)`.
- **Index arithmetic**: `s[i+1]` is safe if `i+1 < len(s)`; sometimes needs hoisting.

### What `prove` cannot

- **Cross-function reasoning** beyond inlining. If a function takes a length parameter `n` and uses `s[i] for i < n`, the compiler doesn't know `n ≤ len(s)` unless the function is inlined OR you tell it via a leading expression.
- **Aliasing through interfaces or unsafe.**
- **Arithmetic overflow**: `s[i*2]` may be unsafe if `i*2` overflows.
- **Complex relations across multiple loops** that share a structural invariant the analyzer can't see.

### The "length hoist" trick

```go
// Compiler can't prove s[i]/s[i+1]/s[i+2] inside the loop are safe
// unless it sees the length up-front.

// Original — three checks per iteration:
for i := 0; i < n; i++ {
    a := s[i]
    b := s[i+1]
    c := s[i+2]
    use(a, b, c)
}

// Hoist a check; prove deduces the others:
_ = s[n+2]  // panic now if n+2 ≥ len(s); but also tells prove len(s) > n+2
for i := 0; i < n; i++ {
    a := s[i]
    b := s[i+1]
    c := s[i+2]
    use(a, b, c)
}
```

Pre-loop expression doubles as a guard *and* a fact for `prove`. Common in numeric code.

### `range` is your friend

```go
// Range loop: prove deduces "i < len(s)" exactly.
for i := range s {
    _ = s[i]
}

// Index loop with len(s): also fine.
for i := 0; i < len(s); i++ {
    _ = s[i]
}

// Index loop with a passed-in n: BCE depends.
func loop(s []int, n int) {
    for i := 0; i < n; i++ {
        _ = s[i]  // BCE: only if prove knows n ≤ len(s)
    }
}

// Fix:
func loop2(s []int, n int) {
    if n > len(s) {
        n = len(s)
    }
    s = s[:n] // now s's len IS n
    for i := 0; i < len(s); i++ {
        _ = s[i] // BCE
    }
}
```

### Re-slicing as a BCE technique

```go
// Process the first 16 bytes:
func first16(buf []byte) [16]byte {
    var out [16]byte
    b := buf[:16] // panic now if buf is too short
    for i := 0; i < 16; i++ {
        out[i] = b[i]  // BCE: prove knows i < 16 ≤ len(b)
    }
    return out
}
```

The slice expression is the single point of failure. Inside the loop the compiler proves every check.

### Multi-dimensional indexing

```go
// 2D index s[i][j] — two checks per access.
for i := range s {
    for j := range s[i] {
        _ = s[i][j]
    }
}

// Hoist the row:
for i := range s {
    row := s[i]
    for j := range row {
        _ = row[j]
    }
}
```

The hoist replaces `(check(i), index(s,i), check(j), index(row,j))` with `(check(i), index(s,i), check(j), index(row,j))` but the inner check now uses a fresh-from-deref length that the compiler tracks better. Quantitatively: ~10–30% speedup on dense matrix loops.

### Strings are array-like

`string[i]` is bounds-checked exactly like `[]byte[i]`. `prove` treats them identically.

### `bce` builds since Go 1.5

The `prove` pass was introduced in Go 1.7, refined heavily through 1.18+. Modern Go has very strong BCE; loops that needed manual coaxing in 2017 often need none today. Always recheck on version bumps.

### Generics and BCE

Generic functions are stenciled per GC shape; each stencil compiles separately. BCE works inside the stencil. If a generic uses `len(s)` where `s` is a type-parameter slice, `prove` knows `len` semantics.

### `IsInBounds` vs `IsSliceInBounds`

Index `a[i]`: `IsInBounds i n` checks `0 ≤ i < n`.
Slice `a[i:j]`: two checks, `0 ≤ i ≤ j ≤ n` (allowed to equal cap).

The latter is `IsSliceInBounds`. `prove` handles both.

### Inspecting decisions

```bash
# Where are bounds checks left?
go build -gcflags='-d=ssa/check_bce/debug=1' ./...

# Detailed SSA dump for a hot function
GOSSAFUNC=HotLoop go build .
# Open ssa.html — look at the "prove" column for facts and at "lower" or
# "deadcode" for whether IsInBounds is gone.
```

The reported lines are file:line of the SOURCE indexing operation, not the SSA op.

### What's the cost of an un-eliminated bounds check?

On modern x86-64 with well-predicted branches: ~0.3–1 ns per check. In a tight loop iterating millions of times, that's 1–5% wall time. In math kernels (matrix multiply, FFT), eliminating bounds checks routinely yields 10–30% speedups because each iteration has multiple indices.

CPU effects:
- Branch predictor cache pressure.
- Instruction decoder bandwidth.
- Pipeline width — a fused-compare-branch competes with the actual arithmetic op for slots.

## Standard Library Hooks

- `-gcflags='-d=ssa/check_bce/debug=1'` — emit a line per remaining bounds check.
- `-gcflags='-d=ssa/prove/debug=N'` — `prove` pass diagnostics; N=1–5.
- `GOSSAFUNC=Foo go build .` — full SSA dump.
- `-gcflags='-B'` — disable BCE entirely (debugging tool).
- `-gcflags='-N'` — disable optimizations (also disables prove).
- `runtime.panicIndex` — runtime function called on bounds-check failure (visible in disassembly).
- `slices.SortFunc` and friends are BCE-friendly by design (1.21+).

## Real-World Patterns

### 1. Idiomatic range-over-slice

```go
package main

func dot(a, b []float64) float64 {
	if len(a) != len(b) {
		panic("size")
	}
	var sum float64
	for i := range a {
		sum += a[i] * b[i] // a: BCE. b: only if prove deduces len(b) ≥ len(a).
	}
	return sum
}
```

The compiler often eliminates both, but to be safe:

```go
b = b[:len(a)] // panics if too short; prove now knows len(b) ≥ len(a)
for i := range a {
    sum += a[i] * b[i]
}
```

### 2. Process a fixed-size block

```go
package main

func process16(b []byte) {
	if len(b) < 16 {
		return
	}
	_ = b[15] // single check; prove now knows len(b) ≥ 16
	for i := 0; i < 16; i++ {
		b[i]++
	}
}
```

### 3. Read four bytes as big-endian

```go
package main

import "encoding/binary"

// Hand-written without bce:
func u32(b []byte) uint32 {
	return uint32(b[0])<<24 | uint32(b[1])<<16 | uint32(b[2])<<8 | uint32(b[3])
}

// With hoisted check:
func u32fast(b []byte) uint32 {
	_ = b[3]                             // single check
	return uint32(b[0])<<24 | uint32(b[1])<<16 | uint32(b[2])<<8 | uint32(b[3])
}

// Better: use the stdlib (which the compiler recognizes specially):
func u32best(b []byte) uint32 {
	return binary.BigEndian.Uint32(b)    // intrinsic, no bounds check after first
}
```

`encoding/binary`'s methods include a hoisted check; the compiler also has intrinsic recognition since 1.18.

### 4. Matrix loop optimization

```go
package main

func matmul(a, b, c [][]float64, n int) {
	for i := 0; i < n; i++ {
		ai := a[i]
		ci := c[i]
		for j := 0; j < n; j++ {
			var s float64
			for k := 0; k < n; k++ {
				s += ai[k] * b[k][j]
			}
			ci[j] = s
		}
	}
}
```

Hoisting `a[i]` and `c[i]` to local slices `ai`, `ci` lets the compiler prove the inner-loop indices safely. The remaining `b[k][j]` is harder; restructure to `b[k]` outer for cache anyway.

### 5. Verify BCE in a hot loop

```bash
$ go test -bench=BenchmarkHot -gcflags='-d=ssa/check_bce/debug=1' ./...
hot_test.go:42: Found IsInBounds
```

A surviving `IsInBounds` is the smoking gun. Look at line 42 and adjust.

## Anti-Patterns & Gotchas

**Rewriting `for i := range s` to `for i := 0; i < len(s); i++` and back, hoping for speedup.** Both are equivalent for BCE. Profile, don't guess.

**Storing `len(s)` in a local and looping `for i := 0; i < n; i++ { _ = s[i] }`.** Defeats `prove` — it doesn't relate `n` to `len(s)` anymore. Use `for i := 0; i < len(s); i++` or `range`.

**Hoisting checks with `_ = s[n-1]` but using `n` after a slice operation changes `s`.** The fact is invalidated by the re-assignment. Hoist again or re-slice.

**Assuming BCE means "no overhead".** The compare is gone; the index computation is not. Strided / unpredictable accesses still pay cache costs.

**Using `-gcflags=-B` in production.** Disabling all bounds checks is **unsafe**: a bad index can read arbitrary memory or panic the runtime. Only use in benchmarks to compare; never ship.

**Relying on BCE for unsigned underflow safety.** `s[i-1]` where `i` is `uint` and `i==0` will wrap to a huge value; bounds check catches it, but you might have expected a Go-style "negative index" panic. Use signed types when subtracting.

**Believing BCE works across function calls.** It doesn't, unless inlined. Pass slices, not lengths-and-pointers.

**Optimizing BCE before profiling.** If your hot path isn't bounds-check-bound, you're shaving 1% from something that doesn't matter.

**Adding hoist comments that go stale.** A future refactor reorders the function and the hoist is no longer first. Use `_ = s[N]` near the loop, not far away.

## Performance Notes

- One bounds check: ~0.3–1 ns (fused compare + predicted-not-taken branch).
- Loop with one BCE-failed check per iteration: ~5–20% overhead vs eliminated.
- Loop with multiple BCE-failed checks per iteration: 20–40% overhead.
- Hot numeric kernels (FFT, matmul) typically gain 10–30% from full BCE.
- `binary.BigEndian.Uint32` etc.: 0 bounds checks after hoist, single load.
- `copy(dst, src)`: 0 in-loop checks; uses `memmove`.
- `append`: a bounds check on capacity (handled in `growslice`).

`benchstat` comparing builds with and without `-B`:

```
name      old time/op  new time/op  delta
Hot       4.2ns ± 1%   3.6ns ± 1%   -14%
```

A 14% delta is BCE-bound. If the delta is <2%, BCE isn't your bottleneck.

## How Big Companies Use It

- **Klauspost's reedsolomon** library does aggressive BCE hoisting for erasure-coding loops: https://github.com/klauspost/reedsolomon.
- **`encoding/json`** v2 (1.26 candidate) uses explicit length checks and `_ = b[n-1]` hoists in hot decoders.
- **`crypto/aes`** — AES round implementation hoists checks once per block: https://github.com/golang/go/blob/master/src/crypto/aes/.
- **`runtime/internal/atomic`** — pure machine code, no checks at all.
- **CockroachDB's coldata package** — columnar data structures with explicit hoists for cache-line-friendly traversal: https://github.com/cockroachdb/cockroach.
- **InfluxDB's tsm storage** — bulk-decode loops are BCE-tuned for throughput: https://github.com/influxdata/influxdb.
- **Cilium** datapath helpers — BCE-tuned packet parsing in user-space tools.
- **Tailscale's wire packet codecs** — checked once, parsed without bounds.

## Source Code References

Pinned to `go1.26`.

- `prove` pass: [`src/cmd/compile/internal/ssa/prove.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/prove.go).
- `IsInBounds` definition: [`src/cmd/compile/internal/ssa/_gen/genericOps.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/_gen/genericOps.go) (search `IsInBounds`).
- Slice bounds: [`src/cmd/compile/internal/ssa/_gen/generic.rules`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/_gen/generic.rules) — search `IsSliceInBounds`.
- `runtime.panicIndex`: [`src/runtime/panic.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/panic.go).
- BCE diagnostics: [`src/cmd/compile/internal/ssa/check_bce.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/check_bce.go).
- Bounds-check intrinsic recognition (binary.BigEndian.Uint32 etc.): [`src/cmd/compile/internal/ssagen/intrinsics.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssagen/intrinsics.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Bounds Check Elimination" — Go wiki: https://github.com/golang/go/wiki/CompilerOptimizations#bounds-check-elimination.
- Brad Fitzpatrick, "Eliminating bounds checks in a Go vector library": https://research.swtch.com/.
- Damian Gryski, "go-perfbook — bce": https://github.com/dgryski/go-perfbook.
- Klauspost, "Bounds Check Hoisting" examples: https://klauspost.io.
- "Loop Bounds Check Elimination" — proposal #15324: https://go.dev/issue/15324.
- "Bound check elimination in Go" — Adrian Hesketh blog (illustrated walkthrough): https://infrequently.org.
- Cherry Mui, "SSA backend internals" — talks on the `prove` pass.

## Exercises / Self-Check

1. Write a loop that the compiler can't eliminate checks for. Add a hoist that satisfies prove and verify with `-d=ssa/check_bce/debug=1`.
2. Why does `s[:n]` followed by `for i := range s` BCE perfectly, while `for i := 0; i < n; i++ { _ = s[i] }` may not?
3. Implement a function that reads 8 bytes as little-endian uint64 with exactly one bounds check. Show the assembly to confirm.
4. Take a published BCE-tuned function from `klauspost/reedsolomon`. Identify each hoist and explain what fact it provides to `prove`.
5. Profile a hot benchmark with and without `-B`. If the delta is small, what would you optimize next?
