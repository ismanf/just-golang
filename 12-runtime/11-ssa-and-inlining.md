# SSA and Inlining — Budget, Mid-Stack, Devirtualization

## TL;DR

After type-checking, `cmd/compile` lowers Go to **SSA (Static Single Assignment)** and runs ~50 passes. The most consequential pre-SSA pass is **inlining**: it splices small callees' bodies into callers, enabling escape analysis, bounds-check elimination, and constant propagation across the boundary. The inline budget is **80 hairy-call units** (configurable via `-l=N` and per-call directives). **Mid-stack inlining** (since 1.10) inlines functions whose own callees aren't all inlinable — previously inlining stopped at the first non-trivial callee. **Devirtualization** (improved each release) turns interface calls into direct calls when the concrete type is known. The single biggest gotcha: **functions that look small can be too "hairy"** — `defer`, `recover`, `range over func`, large switches, `select`, and closures all cost a lot of budget.

## Mental Model

```
   func Caller() {
       x := Callee(y)       ← inline candidate
       Other(x)             ← non-inlinable
   }
   func Callee(v int) int { return v * 2 }

   Before inlining:
       Caller calls Callee, Callee returns.
       Escape analysis sees Callee's body in isolation.

   After inlining:
       Caller body becomes:
         x := y * 2
         Other(x)
       Escape analysis runs over Caller with Callee's body merged.
       Constant propagation, dead-code elim, BCE happen across what
       used to be a call boundary.

   Mid-stack inlining (since 1.10):
       Even if Callee itself contains a call to Other (non-inlinable),
       Callee can still be inlined into Caller. Earlier versions stopped.
```

The pre-SSA `inline` pass walks every function, identifies candidates, and *substitutes* their bodies (with type-substituted parameters). The result feeds escape analysis, then SSA generation.

## Syntax & Basic Usage

```go
package main

import "fmt"

//go:noinline
func slow(x int) int { return x*x + 3 }

func fast(x int) int { return x*x + 3 }

func main() {
	fmt.Println(slow(2), fast(2))
}
```

Inspect:

```bash
$ go build -gcflags='-m=2' .
./main.go:5:6: cannot inline slow: marked go:noinline
./main.go:7:6: can inline fast with cost 9 as: func(int) int { return x * x + 3 }
./main.go:10:21: inlining call to fast
```

## Deep Dive

### Why inline at all

Direct benefits:

1. **Eliminate call overhead.** A non-inlined call costs ~5–10 ns (prologue, regalloc preservation, return).
2. **Enable cross-function constant folding.** If `Callee(7)` is inlined and 7 is constant, the body simplifies.
3. **Enable better escape analysis.** A pointer that escapes "through" a function call doesn't escape if the function is inlined.
4. **Enable BCE.** Bounds checks can be eliminated when the analyzer sees both producer (slice length) and consumer (index check).
5. **Enable devirtualization.** A call through an interface, if inlined, may show the concrete type.

Drawbacks:

1. **Code size growth.** Each inlining adds bytes to the binary.
2. **Compile-time growth.** Inlined functions get re-analyzed in each call site.
3. **I-cache pressure.** Very large hot functions can outrun L1 instruction cache.

The compiler picks a balance via the cost budget.

### The cost model

`cmd/compile/internal/inline/inl.go` walks a function and sums:

- Each `OOP` (arithmetic, comparison, indexing, etc.): 1.
- A `CALL` to a known-inlinable function: 57 (the prologue cost).
- A `CALL` to an unknown / non-inlinable function: 57 plus call-arg cost.
- A non-trivial expression (e.g., type assertion, type switch): higher.
- `defer`: usually disqualifies entirely or adds large cost.
- `recover`, `go`, `select`, `for-range` over function: large additions.

The default budget is **80**. Beyond that the function is rejected.

You can change the budget per compile with `-gcflags=-l=N` (N=0 disables inlining entirely; N=1 default; -gcflags='-l -l -l -l' raises budget repeatedly, four-deep being the historical "force inline everything"). But changing it globally is rarely what you want.

### "Hairy" features that disqualify

Functions are rejected outright (regardless of budget) if they contain:

- `go` statements (spawning goroutines).
- `select` (the inliner doesn't model it).
- `range over func` iterators (since 1.23): treated like a closure call.
- A non-Go body (assembly, runtime-internal).
- Certain `unsafe` operations.
- Recursive calls (recursion guards inlining).
- `defer` until 1.14; since 1.14, simple defers (open-coded) are inlinable.

You can sometimes refactor: split the hairy part into a separate helper, leave the hot body simple.

### Mid-stack inlining

Pre-1.10: if `f` called `g`, and `g` was not inlinable, then `f` was effectively not inlinable either — the chain broke at the first non-trivial callee.

Since 1.10: `f` can be inlined into its callers even if `g` (called inside `f`) is not. The recursive expansion stops at the non-inlinable `g`, but `f`'s body still merges into the caller.

This change made many common patterns (small wrappers around `runtime.assertI2T2`, for example) inlinable for the first time.

### Devirtualization

A call `iface.Method()` is, at the IR level, an indirect call through the interface's itab. If escape analysis or value propagation can determine the *concrete* type at this site, the inliner rewrites the call to `*ConcreteType.Method`. Then the direct call can also be inlined.

```go
package main

import "fmt"

type Stringer interface{ String() string }
type MyType struct{ s string }
func (m *MyType) String() string { return m.s }

func print(s Stringer) { fmt.Println(s.String()) }

func main() {
	m := &MyType{s: "hi"}
	print(m) // since 1.21+, devirtualizer recognizes m's type
}
```

`-gcflags='-m=2'` shows:

```
devirtualizing m.String to (*MyType).String
inlining call to (*MyType).String
```

Type assertions also cooperate:

```go
if v, ok := x.(*MyType); ok {
    v.Method() // direct call after assertion
}
```

In 1.21+ the compiler propagates concrete types through `defer`, simple branches, and `errors.As`-style patterns more aggressively.

### PGO and inlining

Profile-guided optimization (since 1.20, GA 1.21) feeds a CPU profile into the compiler; hot call sites get a *higher inline budget*. See `12-runtime/13-pgo.md`. Typical effect: 2–5% throughput improvement on real services.

### SSA-side optimizations enabled by inlining

After inlining, the SSA passes get a richer graph:

- **`opt`**: peephole rewrites (strength reduction, `x*2 → x<<1`).
- **`prove`**: value range analysis — produces evidence for BCE and other facts.
- **`cse`**: common subexpression elimination across inlined boundary.
- **`dse`**: dead store elimination — irrelevant stores from the inlined body.
- **`loop rotation, hoisting`**: invariants from inlined math.
- **`decompose`**: slice/interface/struct split into scalars.

Inlining isn't the optimizer's only entry, but without it most passes can't see across function boundaries.

### Compiler directives that affect inlining

| Directive | Effect |
|---|---|
| `//go:noinline` | Never inline this function. |
| `//go:nosplit` | No stack-overflow check; for tiny runtime helpers. |
| `//go:noescape` | Inputs do not escape (assembly stubs). |
| `//go:nocheckptr` | Skip the cgo pointer check. |
| `//go:registerparams` (experimental) | Use register-based ABI. |

There's no `//go:inline` — you can't *force* inlining. The closest is removing whatever disqualifies the function.

### Generics and inlining

Generic functions are stenciled per GC shape (see `12-runtime/10-compiler-pipeline.md`). Each stencil is its own function with its own cost evaluation. Inlining works as for non-generic code, but the cost can balloon if a generic uses many capabilities (interface constraints, type-switches over type-parameters).

### Closures

Closures are typically:
- Stack-allocated and inlined if the closure escapes only to known callees that inline.
- Heap-allocated when stored, returned, or sent on channels — still inlinable if the function it's called within is.

Closure cost is high in the inline budget (typically ≥20) because of the captured-variable handling.

### Cross-package inlining

The compiler emits each package's inlinable function bodies in the package's export data (`.a` file). At compile time of a downstream package, the inliner has access to these bodies. Closer inspection:

- The exported function body must be *visible* in export data — only happens if the function is inlinable.
- The downstream package must be rebuilt after the upstream changes for the new body to take effect (the build system handles this).
- A function annotated `//go:noinline` is not exported.

This is why subtle changes (an added `defer`, a switch statement) in a small upstream helper can de-inline downstream call sites — and you'd only notice by re-running benchmarks.

### Reading SSA

```bash
GOSSAFUNC=Foo go build .
# open ssa.html
```

Each pass column shows the SSA after that pass. Hover over a value to see its definition. Common things to look for:

- **After `early opt`**: did inlining happen? Look for the callee's instructions in the caller.
- **After `prove`**: are bounds-check operations still present?
- **After `lower`**: arch-specific opcodes (`AMD64ADDQ`, etc.).
- **After `regalloc`**: each value has an assigned register or stack slot.

`-gcflags='-d=ssa/<pass>/dump=Foo'` dumps a specific pass to stderr instead.

### Inlining and benchmarks

Microbenchmarks can be dominated by call overhead. Inlining can shave 10–30% from a 5-ns op. Always check:

```bash
$ go test -bench=. -gcflags='-m' ./...
```

If `bench` shows surprising results, suspect an inlining decision change. The first thing to check is `cannot inline ...` output.

### When NOT to inline

- **Cold paths**: error handling, init. The compiler does this automatically (low call frequency).
- **Very large functions called once**: marginal benefit, large binary growth.
- **Functions with high register pressure** that would spill in the caller.

The compiler's heuristics handle these; manual intervention is rare.

## Standard Library Hooks

- `-gcflags='-m'`, `-gcflags='-m=2'`, `-gcflags='-m=3'`: inlining + escape decisions.
- `-gcflags='-l'`: disable inlining (one level); `-l -l -l -l` (four levels) historically raised the budget. Current convention: `-gcflags='all=-l'` to disable everywhere.
- `-gcflags='-d=ssa/<pass>/dump=Func'`: dump SSA after a pass.
- `GOSSAFUNC=Func go build .`: write `ssa.html` for one function.
- `runtime.CallersFrames`: handles inlined frames in tracebacks (since 1.12).
- `go tool objdump`: disassemble final binary.
- `go tool nm`: symbol table.
- `go build -pgo=cpu.prof`: enable PGO inlining decisions.

## Real-World Patterns

### 1. Verify a hot function is inlined

```go
package hot

func Hash(b []byte) uint64 {
	h := uint64(1469598103934665603)
	for _, c := range b {
		h ^= uint64(c)
		h *= 1099511628211
	}
	return h
}
```

```bash
$ go build -gcflags='-m=2' ./hot
./hash.go:3:6: can inline Hash with cost 38 as: ...
```

Cost 38 — comfortably under 80. Good. Callers will get the body inlined.

### 2. Refactor away from a "hairy" feature

```go
// Before: defer disqualifies inlining.
func Process(p *T) error {
	defer p.unlock()
	return p.work()
}

// After: move defer outside the hot path.
func Process(p *T) error {
	err := p.work()
	p.unlock()
	return err
}
```

(Use this only if `work` cannot panic; otherwise the defer is correct.)

### 3. Make an interface call direct

```go
package main

type Doer interface{ Do() }
type Impl struct{}
func (Impl) Do() {}

// Slow: indirect call.
func runAny(d Doer) { d.Do() }

// Fast: concrete type, inlinable.
func runImpl(d Impl) { d.Do() }
```

When you know the concrete type, take it. Devirtualization is great but not always possible.

### 4. Use generics for monomorphization-ish behavior

```go
package main

import "fmt"

func Max[T int | int64 | float64](a, b T) T {
	if a > b {
		return a
	}
	return b
}

func main() {
	fmt.Println(Max(1, 2), Max(1.5, 0.3))
}
```

Each instantiation gets its own stencil (or shared shape). Generally inlines.

### 5. PGO for inlining

```bash
$ go build -o bin .
$ ./bin -cpuprofile=cpu.prof &     # collect profile under realistic load
$ kill %1; wait
$ cp cpu.prof default.pgo
$ go build -pgo=auto .             # next build uses default.pgo
```

The compiler then inlines hot call sites that would otherwise miss the budget. Reproducible 2–5% wins on big services.

## Anti-Patterns & Gotchas

**Sprinkling `//go:noinline` to "improve profiling".** It does; it also slows your code. Use `pprof.Labels` instead.

**Building a wrapper around every stdlib call.** Wrappers are usually inlinable, but each function you write adds source noise. The 80-cost ceiling means moderate wrappers compose fine; absurdly small ones don't help much.

**Refactoring purely to fit the inline budget.** Test first, refactor only if there's actual speedup. Microoptimizations rarely move the needle.

**Forcing a budget increase via `-l -l -l -l` in production builds.** It's a debug knob; produces unmaintainable assumptions.

**Believing `interface.Method()` is always direct after devirtualization.** It is when the analyzer can prove the type. In data-driven code (vtable dispatch, registry patterns), it can't. Profile first.

**Reading `-m=2` and assuming the bottom line ("does not escape") is the answer.** Read the `flow:` lines; sometimes "does not escape" is true *despite* an inlining failure that's the real cost.

**Trying to inline a function that contains `select`.** Won't happen. Restructure to a helper called from one arm of the select.

**Confusing "inlinable" with "always inlined".** A function can be marked inlinable in export data but still not inlined at a specific call site (e.g., loop unroll has already grown the caller past its own budget). Inspect each site.

**Pre-1.10 mental model.** "If `g` is not inlinable, then `f` can't be either." Wrong since 1.10; let the compiler decide.

**Forgetting to rebuild downstream after changing upstream.** `go build` handles it; `go test ./...` handles it; manual `go vet` of individual packages does not. Stale builds can give surprising profiling results.

## Performance Notes

- A small inlined function (cost ≤20): saves ~5–10 ns per call.
- A medium inlined function (cost 30–60): saves ~10–20 ns; depending on caller layout, may pay back in cache.
- A function devirtualized + inlined: saves ~10 ns plus enables further opt.
- Inline budget breach by 1 unit can mean a 20% throughput delta in a tight benchmark.
- `-l` (no inlining) typically 2–10× slowdown on benchmarks; varies wildly.
- PGO inlining: 2–5% throughput on real services; less on microbenchmarks.
- Binary size cost of inlining: typically 1–5%; aggressive inlining (PGO heavy) can be 10%+.

## How Big Companies Use It

- **Google** internal Go services use PGO routinely; cited 2–7% wins across many services: https://go.dev/blog/pgo.
- **Uber's go-perfbook** documents inlining triage: https://github.com/dgryski/go-perfbook.
- **CockroachDB** has annotated hot paths with `//go:noinline` for profiling clarity, then removed annotations once benchmarks were stable: https://www.cockroachlabs.com/blog/.
- **Cloudflare** has documented production wins from inlining + devirtualization in their edge proxy: https://blog.cloudflare.com/scaling-go-applications/.
- **Tailscale**'s `magicsock` benchmarks check inlining decisions in CI to guard regressions: https://github.com/tailscale/tailscale.
- **etcd** uses `go vet` analyzers that flag functions that grew past the inline budget unexpectedly.
- **Discord's hot paths** were heavily inline-tuned before the migration to Rust; their post-mortem credits "we'd already squeezed all the inlining we could".

## Source Code References

Pinned to `go1.26`.

- Inliner: [`src/cmd/compile/internal/inline`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/inline).
- Cost model: [`src/cmd/compile/internal/inline/inl.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/inline/inl.go) — search `inlineMaxBudget`.
- Devirtualization: [`src/cmd/compile/internal/devirtualize`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/devirtualize).
- SSA passes: [`src/cmd/compile/internal/ssa/compile.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/compile.go).
- SSA opt rules: [`src/cmd/compile/internal/ssa/_gen/generic.rules`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/_gen/generic.rules).
- Prove pass (BCE-enabler): [`src/cmd/compile/internal/ssa/prove.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/prove.go).
- PGO integration: [`src/cmd/compile/internal/pgo`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/pgo).
- Generic stenciling: [`src/cmd/compile/internal/typecheck/subr.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/typecheck/subr.go) and `noder/`.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- Keith Randall, "Inside the Go compiler — SSA backend" (GopherCon 2018): https://www.youtube.com/watch?v=uTMvKVma5ms.
- David Lazar, "Mid-stack inlining" (Go blog): https://go.dev/blog/inlining.
- Austin Clements & Cherry Mui, "Profile-guided optimization" (Go blog): https://go.dev/blog/pgo.
- Damian Gryski, "go-perfbook — inlining": https://github.com/dgryski/go-perfbook.
- "Go SSA" wiki: https://github.com/golang/go/wiki/CompilerOptimizations.
- "Notes on Go's inliner" — Will Newton blog: https://williamhepburn.com.
- Khoa Truong, "Devirtualization in Go 1.21": https://go.dev/doc/go1.21.
- Russ Cox, "Generics implementation" notes: https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md.

## Exercises / Self-Check

1. Take a function with `defer m.Unlock()` and confirm with `-m=2` whether it inlines. Refactor the `defer` away and re-check.
2. Why is the inline budget unit "57 per call"? What's the cost model trying to approximate?
3. Construct an interface call site that the compiler devirtualizes, and one that it doesn't. What's the structural difference?
4. Run `GOSSAFUNC=Hot go build .` on a hot function. Find the pass where a bounds check is eliminated. Identify the SSA evidence used.
5. Build a binary with and without PGO on a small server; compare `go tool objdump` output for the same function. What changed?
