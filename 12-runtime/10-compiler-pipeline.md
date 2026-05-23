# Compiler Pipeline — Parse → Typecheck → SSA → Assembly

## TL;DR

`cmd/compile` is a multi-pass compiler: **parse** Go source into an AST, **type-check** with full inference, lower to an **internal IR** (since 1.16, the `ir` package), translate that to **SSA** (static single assignment), run dozens of SSA optimization passes, then a per-architecture lowering pass produces **machine code** which the linker assembles into the binary. The whole thing usually finishes in under a second per package. The single biggest gotcha: **the compiler is per-package**; inlining and escape analysis cross function boundaries within a package but only see *exported, inlinable* function bodies across packages — refactor that crosses a package boundary may invisibly change optimization decisions.

## Mental Model

```
  .go file(s)
       │
       ▼
  ┌──────────────┐
  │   Parser     │  cmd/compile/internal/syntax   → syntax tree
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Type checker │  cmd/compile/internal/types2   → typed AST (irgen)
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │     IR       │  cmd/compile/internal/ir       → typed function bodies
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Devirt/Inline│  cmd/compile/internal/inline   → expand qualified calls
  │ Escape       │  cmd/compile/internal/escape   → heap vs stack decisions
  │ Walk         │  cmd/compile/internal/walk     → lower complex ops
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │     SSA      │  cmd/compile/internal/ssa      → IR in SSA form
  └──────┬───────┘
         │  ~50 passes: dead code, CSE, BCE, prove, lower, regalloc, ...
         ▼
  ┌──────────────┐
  │  Per-arch    │  cmd/compile/internal/{amd64,arm64,...}
  │  emit asm    │                                  → object files
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │   Linker     │  cmd/link                       → executable
  └──────────────┘
```

You can inspect any stage with build flags. The compiler is heavily instrumented; reading its output is the closest thing Go has to a "decompiler" REPL.

## Syntax & Basic Usage

```go
package main

import "fmt"

func add(a, b int) int { return a + b }

func main() { fmt.Println(add(2, 3)) }
```

Inspect at various stages:

```bash
# 1. Type-check + escape analysis output
go build -gcflags='-m=2' .

# 2. SSA dump — produces ssa.html for inspection in a browser
GOSSAFUNC=add go build .
# Open ssa.html — shows every SSA pass side-by-side.

# 3. Final assembly
go build -gcflags='-S' . 2>asm.s

# 4. Object disassembly
go tool objdump -s '^main\.add$' ./executable
```

Common `-gcflags`:

| Flag | Effect |
|---|---|
| `-m=N` | escape analysis verbosity, 1–5 |
| `-S` | emit assembly to stderr |
| `-l` | disable inlining |
| `-N` | disable optimizations |
| `-d=ssa/<pass>/dump=<func>` | dump SSA after `<pass>` for `<func>` |
| `-d=ssa/<pass>/debug=N` | enable per-pass diagnostics |
| `-d=loopvar=2` | trace 1.22 loop-var transformation |

`GOSSAFUNC=funcname` is the most useful: it writes `ssa.html` with every pass annotated.

## Deep Dive

### Stage 1 — Parse

`cmd/compile/internal/syntax` is a *hand-written* recursive-descent parser. It produces a syntax tree of `syntax.Node` interfaces. Key properties:

- Independent of `go/ast`. The standard library's `go/ast` is for tools (gofmt, gopls); the compiler has its own faster, simpler tree.
- The parser is single-pass and resolves nothing — every name is a string until later.
- Source positions are interned per-file to keep nodes small.

### Stage 2 — Type checking

`cmd/compile/internal/types2` is the canonical Go type checker — shared with `go/types`. It performs:

- Name resolution.
- Type inference (since 1.18 includes generics).
- Constant folding for untyped constants (full arbitrary-precision via `go/constant`).
- Method-set computation.
- Interface satisfaction checking.

Output: a typed AST where every `Expr` has a `Type()` and `tv.Value` for constants.

The `types2` package is the second iteration; pre-1.18 the compiler had its own checker (`typecheck`). The unification with `go/types` was driven by the need for one source of truth for generics.

### Stage 3 — IR

After type-checking, the compiler lowers to **`ir`** (since 1.16). `ir.Node` is the new internal representation; it's typed and ready for optimization passes that *want* a tree (escape analysis, inlining, walk).

```go
// cmd/compile/internal/ir/node.go (excerpt; BSD-3 © The Go Authors)
type Node interface {
    Pos() src.XPos
    Type() *types.Type
    Op() Op
    Editable
    ...
}
```

Each `Op` (`OADD`, `OCALL`, `OINDEX`, ...) corresponds to a Go operation. The tree is mutable in-place; passes rewrite it.

### Stage 4 — Devirtualization and inlining

`devirtualize` (per call site): if an interface call's concrete type is known at the site (from escape/dataflow analysis), rewrite it to a direct call.

`inline` (per call): if the callee fits the **inline budget** (cost ≤ 80 hairy-call units), splice its body in. Beyond a fixed depth (default 4), recursion stops.

The inline budget is heuristic: each `OOP` costs 1 unit; a `CALL` to a non-inlinable function costs 57; etc. The 80-unit cap was chosen empirically. Functions decorated `//go:inline` are forced (still subject to recursion bounds); `//go:noinline` skipped.

### Stage 5 — Escape analysis

See `12-runtime/02-stack-vs-heap.md`. Runs after inlining so the analyzer sees through inlined calls. Marks every node "stack" or "heap".

### Stage 6 — Walk

`walk` lowers high-level operations to runtime calls:

```
range over slice    → for-loop with index increment
range over chan     → calls to runtime.chanrecv
map iteration       → calls to runtime.mapiternext
make(map)           → call to runtime.makemap
append(s, x...)     → call to runtime.growslice + memmove
defer (heap path)   → call to runtime.deferproc / deferreturn
new(T)              → call to runtime.newobject (or stack alloc)
go f()              → call to runtime.newproc
```

After `walk`, the IR is "low-level" — no high-level constructs remain, just `OCALL`, `OAS`, `OIF`, arithmetic, and memory ops.

### Stage 7 — SSA generation

`cmd/compile/internal/ssa` converts the lowered IR to SSA form:

- Each variable gets versioned: `x` becomes `x_1`, `x_2`, ...
- φ functions reconcile multiple definitions at join points.
- Memory is its own SSA value (each store produces a new memory version).

```
  v3 = ADD <int> v1 v2
  v4 = STORE <mem> {p} v3 v0
  goto block 1
  
  block 1:
    v5 = PHI <mem> v4 v6 ...
```

Each SSA `Value` has an `Op`, type, args, and aux info. The graph of values is the function.

### Stage 8 — SSA passes (~50 of them)

Major passes (order matters; see `cmd/compile/internal/ssa/compile.go`):

1. **early phielim, early copyelim** — basic cleanup.
2. **deadcode** — remove unreachable values.
3. **opt** — peephole pattern rewrites (e.g., `x*2 → x<<1`).
4. **generic.rules** application — hundreds of architecture-independent rewrites.
5. **prove** — value range / bounds analysis; emits BCE evidence.
6. **fuse** — merge adjacent basic blocks.
7. **dse** (dead store elim) — remove unused writes.
8. **cse** (common subexpression elim).
9. **loop rotate, loop invariant code motion.**
10. **decompose** — split aggregates (slices → ptr/len/cap; interfaces → itab/data).
11. **schedule** — order values within blocks.
12. **lower** — arch-specific. Replaces `Add64` with `AMD64ADDQ`, etc.
13. **regalloc** — register allocation (graph coloring, ~Chaitin-style).
14. **stackalloc** — assign stack slots to spilled values.
15. **trim** — last cleanup.
16. **layout** — basic block ordering.

`GOSSAFUNC=add` dumps every pass for `add` — drop into the resulting `ssa.html` and click through.

### Stage 9 — Code emission

Per-arch backend (`cmd/compile/internal/amd64`, `arm64`, etc.) walks SSA in scheduled order, emitting machine instructions via `cmd/internal/obj` (Go's plan-9-flavored assembler). Output is a `.o` object file (Go's own object format, not ELF/Mach-O until link time).

### Stage 10 — Linker

`cmd/link` is a *whole-program* linker:

1. Read all `.o` files (each one's symbol table, GC bitmaps, type info, function metadata).
2. Resolve symbols, perform relocations.
3. Eliminate dead code at the symbol level — functions never reachable from `main` are dropped.
4. Lay out data: globals (`.data`), code (`.text`), read-only (`.rodata`), GC bitmaps, type descriptors, function tables (`pclntab`).
5. Emit ELF/Mach-O/PE.

The linker also embeds:
- The runtime's pclntab (for tracebacks).
- The cgo wrappers if any.
- The Go-specific symbol table.
- A build ID (used by `go.mod`).

### Important compiler internals to know

#### `//go:linkname`

```go
//go:linkname myFn other/pkg.privateFn
func myFn() // body comes from privateFn
```

The linker resolves `myFn` to `other/pkg.privateFn` at link time. Used to access unexported runtime functions. Since 1.23, requires `--unsafe-linkname` or a special build tag for external packages.

#### `//go:noescape`

Tells escape analysis "this function does not allow its arguments to escape", overriding the analyzer's conservatism. Body must live elsewhere (assembly). Critical for performance of `runtime.memmove` and friends.

#### `//go:nosplit`

Function has no stack-overflow check in prologue. Used for tiny runtime helpers. Must use ≤StackSplit bytes of stack.

#### `//go:noinline`

Force-disable inlining for this function.

#### `//go:nocheckptr`

Disable the `cgo`-related pointer check (1.14+).

See `11-low-level/10-go-directives.md` for the full list.

### Generics: shape stenciling

The 1.18 generics implementation uses **GC shape stenciling**: one compiled function per "GC shape" of the type parameters, not per concrete type. So `Map[int, string]` and `Map[int64, []byte]` may share a body if they have the same pointer/non-pointer layout. The dictionary (passed implicitly) tells the body the concrete types where needed.

This is why generic code can be slower than hand-specialized — interface-like dispatch through the dictionary instead of monomorphized specialization.

### Build cache

`go build` caches package compilation in `$GOCACHE` (default `~/.cache/go-build`). Each cache entry is keyed by:

- Package source content hash.
- Compiler / linker version.
- Build flags (`-gcflags`, `-tags`, `GOOS`, `GOARCH`, etc.).
- Profile (if PGO).

A change to any source file invalidates that file's package and all downstream packages.

### `go tool compile` and `go tool link`

The toolchain dispatches to these binaries:

```bash
go tool compile -V                          # version
go tool compile -gcflags=... -o foo.o foo.go
go tool link -o exe foo.o
```

`go build` is a wrapper that orchestrates compile + link with cache and dependency tracking.

### Cross-compilation

```bash
GOOS=linux GOARCH=arm64 go build .
```

Triggers `compile-linux-arm64` toolchain selection. The per-arch backends are all linked into the same `compile` binary (the linker handles per-target asm-ing). See `14-build-deploy/01-cross-compilation.md`.

## Standard Library Hooks

- `go/parser`, `go/ast`, `go/types` — tools-side replicas (gofmt, gopls, etc.). NOT used by the compiler itself.
- `go/build`, `go/build/constraint` — directive parsing.
- `runtime.FuncForPC`, `runtime.CallersFrames` — read compiler-emitted metadata at runtime.
- `debug/buildinfo` — read the linker-embedded build info from a binary.
- `runtime/debug.ReadBuildInfo` — same, from inside the running process.
- `cmd/compile/internal/...` is **not** in the public API; only available when contributing to the Go toolchain.

## Real-World Patterns

### 1. Find why the compiler isn't inlining

```bash
$ go build -gcflags='-m=2' ./... 2>&1 | grep 'cannot inline'
./foo.go:42:6: cannot inline DoWork: function too complex: cost 142 exceeds budget 80
```

Approaches: split the function, move hot path to a helper, or annotate `//go:noinline` on cold paths so they're not counted.

### 2. Inspect SSA for a hot function

```bash
$ GOSSAFUNC=Process go build .
# open ssa.html
```

Walk passes left-to-right; you'll see arithmetic strength-reduction, BCE results, loop transformations. Particularly useful to verify that a hot loop has its bounds check eliminated.

### 3. Inspect final assembly

```bash
$ go build -gcflags='-S=2' . 2>asm.s
$ less asm.s
# Search for "main.Add STEXT" — your function's emitted instructions.
```

Use this when benchmarking — confirm that the inner loop is what you expect.

### 4. Build with PGO

```bash
$ go build -pgo=cpu.prof .
```

Compiler uses the profile to bias inlining toward hot call sites. See `12-runtime/13-pgo.md`.

### 5. Read build info from a binary

```bash
$ go version -m ./bin
./bin: go1.26
        path  github.com/me/app
        mod   github.com/me/app  v1.2.3  h1:...
        dep   github.com/some/dep v0.4.0 h1:...
        build -compiler=gc
        build CGO_ENABLED=1
        build GOOS=linux
```

Backed by `debug/buildinfo`. Use `runtime/debug.ReadBuildInfo` to access the same from inside the process.

## Anti-Patterns & Gotchas

**Optimizing without `-gcflags=-m`.** Assumptions about escape / inlining decisions are usually wrong. Always verify.

**Reading `-gcflags=-S` and trying to optimize at the assembly level.** Almost always: a Go-level refactor is easier and equally effective. Use `-S` to confirm, not to write.

**Disabling inlining via `//go:noinline` to "isolate" a function in profiles.** It works but penalizes performance. Better: use `pprof.Labels`, distinct function names, or trace events.

**Relying on cross-package inlining.** The compiler inlines across packages only if (a) the callee's body is exportable (small enough), (b) it's not `//go:noinline`, (c) the caller's package was rebuilt after the callee changed. Cross-module refactors can silently de-inline.

**Trying to override the inline budget.** `//go:inline` doesn't force past budget; only past discretionary choices. Massive functions remain non-inlined.

**Using `unsafe.Pointer` to "trick" escape analysis.** Sometimes works, often breaks under future compiler updates. Read the `unsafe` rules in `11-low-level/03-unsafe-pointer.md`.

**Forgetting that `-N -l` disables optimizations AND inlining.** Benchmarks built with `-gcflags='all=-N -l'` (the debugging build) are not comparable to production builds.

**Believing `go build` caches binaries on disk.** It caches *intermediate objects*, not final binaries. Rebuilding with the same inputs is fast but not instant.

**Reading `go.mod` to detect "the compiler version".** `go.mod`'s `go` directive declares *language version*. The actual compiler version is in `runtime.Version()` or `go version`.

## Performance Notes

- Parse: ~1 ms per 10k LOC.
- Type-check: ~10 ms per 10k LOC.
- SSA generation + opt + lower: ~30 ms per 10k LOC.
- Register allocation: O(n²) in IR size; can dominate large functions. Beyond ~100k IR ops per function, regalloc grinds; restructure.
- Final code generation: ~5 ms per 10k LOC.
- Link: O(total symbols); a 50 MB binary takes ~2 s to link.

A clean build of Kubernetes (~6M LOC of Go) takes ~3 minutes on a 32-core machine. Incremental rebuilds are seconds.

`go build -x` shows each invocation; `-work` keeps the temp dir for inspection.

## How Big Companies Use It

- **Bazel/rules_go** uses `go tool compile`/`go tool link` directly under the hood for incremental, hermetic, multi-language builds: https://github.com/bazelbuild/rules_go.
- **Google internal** uses an adapted toolchain that produces deterministic, hash-keyed outputs for distributed builds.
- **Buf (protobuf tooling)** instruments the compiler with custom `//go:linkname` patches to expose unexported `protoreflect` internals.
- **Cilium** (eBPF networking) consumes the compiler's SSA dump to verify hot-path optimizations on Cilium agent code: https://github.com/cilium/cilium.
- **Tailscale's `tsweb`** uses `runtime/debug.ReadBuildInfo` to surface version & module info on every server's `/debug/vars`.
- **govulncheck** consumes the binary's symbol table (compiler-emitted) to identify which vulnerable functions actually get called: https://go.dev/security/vuln.
- **gopls** runs the parser + types2 in-process for analysis; it does NOT invoke `cmd/compile`, but shares `types2`.

## Source Code References

Pinned to `go1.26`.

- Top-level pipeline: [`src/cmd/compile/internal/gc/main.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/gc/main.go).
- Syntax parser: [`src/cmd/compile/internal/syntax`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/syntax).
- Type checker: [`src/cmd/compile/internal/types2`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/types2).
- IR: [`src/cmd/compile/internal/ir`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/ir).
- Inliner: [`src/cmd/compile/internal/inline`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/inline).
- Escape: [`src/cmd/compile/internal/escape`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/escape).
- Walk: [`src/cmd/compile/internal/walk`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/walk).
- SSA: [`src/cmd/compile/internal/ssa`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/ssa).
- Per-arch backends: e.g., [`src/cmd/compile/internal/amd64`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/amd64).
- Linker: [`src/cmd/link`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/link).
- SSA pass list: [`src/cmd/compile/internal/ssa/compile.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/compile.go).
- SSA generic rules: [`src/cmd/compile/internal/ssa/_gen/generic.rules`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/ssa/_gen/generic.rules).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "The Go programming language: compiler" (informal index): https://go.dev/src/cmd/compile/README.
- Keith Randall, "Inside the Go compiler" (GopherCon 2018): https://www.youtube.com/watch?v=uTMvKVma5ms.
- "SSA backend in cmd/compile" design notes: https://go.googlesource.com/go/+/refs/heads/master/src/cmd/compile/README.md.
- Russ Cox, "Generics implementation" — design doc: https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md.
- Cherry Mui, "Register-based calling convention" proposal: https://go.googlesource.com/proposal/+/refs/heads/master/design/40724-register-calling.md.
- Cherry Mui, "Go linker": https://go.googlesource.com/proposal/+/refs/heads/master/design/27539-internal-linker.md.
- "Lessons from optimizing the Go compiler" — Austin Clements blog: https://github.com/golang/go/wiki.
- gopls / `golang.org/x/tools/go/...` source — practical examples of `types2` and `go/ast`.

## Exercises / Self-Check

1. Build a small program with `GOSSAFUNC=main go build .` and open `ssa.html`. Identify the pass that removes a bounds check.
2. Take a function the compiler refuses to inline. Inspect `-m=2`, find the cost, restructure to fit. Verify the new version is inlined.
3. `//go:linkname` a runtime function (say `runtime.nanotime`) into your package. Build and run; confirm the call works. (Note 1.23+ restrictions.)
4. Use `debug/buildinfo` to write a tiny tool that prints the `go.mod` version and build flags from any Go binary on disk.
5. Compare the compile time of `go build` cold vs warm cache on a medium-sized package. Where did the time go? (Hint: `go build -x`, time each step.)
