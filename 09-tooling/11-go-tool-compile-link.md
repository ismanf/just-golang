# `go tool compile` and `go tool link` — Inspecting the Backend

## TL;DR

`go tool compile` is **gc**, the official Go compiler. It reads `.go` files, type-checks them, lowers to SSA, optimizes, and emits a Plan 9-style object file (`.a`). `go tool link` is **gld**, the linker (internal by default; external for cgo). Together they're invoked by `go build` once per package (`compile`) plus once for the final binary (`link`). You rarely call them directly — but you frequently pass flags through via **`-gcflags`** and **`-ldflags`**. The most useful introspection flags are: `-gcflags=-m` (escape analysis decisions), `-gcflags=-S` (assembly listing), `-gcflags=-d=ssa/check/on` (SSA passes), `-gcflags=-l` (disable inlining), `-ldflags=-s -w` (strip symbols+DWARF), and `-ldflags=-X import/path.var=value` (set a string at link time). Understanding the SSA dump (`GOSSAFUNC=fn go build`) is the unlock for optimization questions.

## Mental Model

```
   .go files
        │
        ▼
   go tool compile         (one invocation per package)
        │   ├─ scanner / parser   ── go/scanner, go/parser
        │   ├─ type check          ── go/types-like internal checker
        │   ├─ desugar / walk      ── lower syntax to intermediate form
        │   ├─ SSA build           ── per-function SSA graphs
        │   ├─ SSA passes          ── inline, escape, devirt, opt, sched, regalloc, ...
        │   └─ object emit         ── .a archive (Plan 9 object format)
        ▼
   .a files for each package
        │
        ▼
   go tool link             (one invocation per binary)
        │   ├─ load .a files
        │   ├─ resolve symbols
        │   ├─ lay out sections (.text .data .rodata .typelink ...)
        │   ├─ apply relocations
        │   └─ emit ELF/Mach-O/PE
        ▼
   binary
```

`compile` knows about one package at a time (plus deps' export data). `link` is the only stage that sees the whole program.

## Syntax & Basic Usage

```bash
# Inspecting compile decisions (use via go build for path-resolution)
$ go build -gcflags="all=-m" ./pkg 2>&1 | grep escapes
$ go build -gcflags="all=-S" ./pkg 2>&1 | less        # assembly listing
$ go build -gcflags="all=-l" ./pkg                    # disable inlining
$ go build -gcflags="all=-N -l" ./pkg                 # disable optimization + inlining (debug-friendly)
$ go build -gcflags=-m=2 ./pkg                        # more verbose inlining
$ go build -gcflags="-d=ssa/check/on" ./pkg           # SSA debug

# SSA dump per function
$ GOSSAFUNC=Foo go build ./pkg
$ # opens ssa.html in cwd

# Linker
$ go build -ldflags="-s -w" ./cmd/app                 # strip
$ go build -ldflags="-X main.Version=1.0" ./cmd/app   # set string var
$ go build -ldflags="-buildid=" ./cmd/app             # zero build id

# Direct invocation (rare; mostly internal)
$ go tool compile -V
$ go tool compile -S foo.go > foo.s
$ go tool link -V
```

## Deep Dive

### `-gcflags` syntax recap

```bash
$ go build -gcflags="<pkg-pattern>=<flags>" ./...
$ go build -gcflags="all=-m -l" ./...                # all packages
$ go build -gcflags="github.com/me/...=-m" ./...     # scoped
$ go build -gcflags=-m ./pkg                         # unscoped (only the named pkg)
```

The pattern form (`pkg=flags`) is required when you want to scope; unscoped flags apply only to the *root* packages of the build, *not* dependencies.

### `-m`: escape analysis

```bash
$ go build -gcflags="-m" ./pkg 2>&1
./foo.go:3:6: can inline f
./foo.go:7:6: cannot inline g: function too complex: cost 84 exceeds budget 80
./foo.go:12:9: &x escapes to heap
./foo.go:12:10: moved to heap: x
```

`-m=2` adds more detail:

```bash
./foo.go:12:9: &x escapes to heap:
   flow: ~r0 = &x:
     from &x (address-of) at ./foo.go:12:9
     from return &x (return) at ./foo.go:12:2
```

Read top-down: the *flow* tells you how the escape was inferred. Killing one of those flow steps drops the variable to the stack.

See `12-runtime/04-escape-analysis.md` for the algorithm.

### `-S`: assembly listing

```bash
$ go build -gcflags=-S=foo ./pkg 2>asm.txt
```

The `-S=foo` form restricts to functions whose name matches the regex (here, `foo`).

Output is **Go assembly** (Plan 9 dialect), not native x86/ARM directly. Each line ends with `// PC=<num>` showing the bytecode offset; native instructions are listed as comments on each Go-assembly line.

Useful for confirming:

- Whether a hot loop got vectorized.
- How many spills/reloads happen.
- Whether bounds checks were elided.

For raw native disassembly, use `go tool objdump` after building (`09-tooling/14-go-tool-objdump-nm.md`).

### `-l`, `-N` — disable optimizations

```bash
$ go build -gcflags="all=-N -l" -o app ./cmd/app
```

`-l` disables inlining (one `l`; multiple `-l`s tune the inliner: `-l -l` disables a different stage). `-N` disables optimization entirely. Combine for debugger-friendly builds; `dlv debug` sets these by default.

### `-d`: compiler debug flags

```bash
$ go build -gcflags="-d=ssa/prove/debug=1" ./pkg
$ go build -gcflags="-d=loopvar=2" ./pkg          # legacy 1.22-related diagnostic
$ go build -gcflags="-d=checkptr=1" ./pkg         # unsafe.Pointer rule checks
$ go build -gcflags="-d=nilrange=1" ./pkg
```

A long list of debug knobs lives in `cmd/compile/internal/base/debug.go`. Most are for compiler hackers; `checkptr` is useful when chasing unsafe-pointer bugs (the race detector enables it by default).

### `GOSSAFUNC` — per-function SSA dump

```bash
$ GOSSAFUNC=Foo go build ./pkg
ssa.html generated
```

Writes `ssa.html` in cwd showing **every SSA pass** for function `Foo`: initial CFG, after inline, after escape, after `decompose`, after `lower`, after `regalloc`, after `schedule`, etc. Each pass is a click-through.

Used by compiler hackers; also handy when chasing why a function isn't getting a specific optimization.

`GOSSAFUNC=Foo+` (with `+`) prints to stdout instead of html.

### `-pgo` flag (1.21+)

```bash
$ go build -pgo=cpu.pprof ./pkg
$ go build -pgo=auto ./pkg          # auto-pickup default.pgo
```

PGO supplies a CPU profile to the compiler so it can:

- Inline hot functions even past the normal budget.
- Devirtualize interface calls based on observed types.
- (Future) Reorder basic blocks by frequency.

2–7% wins typical. The profile must come from a representative production workload.

### `-trimpath` interaction

`go build -trimpath` strips file-system prefixes from object files. The compile-time effect: `runtime.FuncForPC(...).FileLine(...)` returns module-relative paths instead of absolute. See `09-tooling/01-go-build.md`.

### `-ldflags` essentials

```bash
$ go build -ldflags="-s -w" ./cmd/app
$ go build -ldflags="-X main.Version=1.2.3" ./cmd/app
$ go build -ldflags="-X github.com/me/pkg.flag=true" ./cmd/app
$ go build -ldflags="-buildid=" ./cmd/app
$ go build -ldflags="-extldflags=-static" ./cmd/app    # static link with external linker (cgo)
$ go build -ldflags="-linkmode=external" ./cmd/app     # force external linker
$ go build -ldflags="-checklinkname=0" ./cmd/app       # 1.23+: relax linkname restrictions
```

| Flag                  | Effect                                                            |
|-----------------------|-------------------------------------------------------------------|
| `-s`                  | Strip symbol table (saves ~10–25%).                              |
| `-w`                  | Strip DWARF debug info (saves more). Breaks debuggers.            |
| `-X path.var=value`   | Set a `var x = "..."` of type string in the named import path.   |
| `-buildid=`           | Override build ID (empty for reproducibility).                    |
| `-linkmode=internal`  | Use Go's linker (default for pure Go).                            |
| `-linkmode=external`  | Use the platform linker (gcc/lld; needed for some cgo cases).     |
| `-extldflags=...`     | Flags passed to the external linker.                              |
| `-checklinkname=0`    | (1.23+) Disable restrictions on `//go:linkname`. Discouraged.     |
| `-r=path1:path2`      | Embed an rpath into the binary (cgo / shared libs).               |

### `-X` requires the var to exist

```go
package main
var Version = "unknown"

func main() { println(Version) }
```

```bash
$ go build -ldflags="-X main.Version=1.0.0" -o app
$ ./app
1.0.0
```

The var must be:

- A `var` (not `const`).
- Of type `string`.
- At the top level of the named package (no inner-scope or method-receiver).

### Internal vs. external linking

| Mode      | Used by                            | Notes                                   |
|-----------|------------------------------------|-----------------------------------------|
| internal  | Pure Go (default)                  | Fast, single-threaded; ships with Go.   |
| external  | CGO, `-buildmode=c-archive`, etc.  | Calls `ld`/`lld` from the system.       |

`go build -ldflags="-linkmode=external"` forces external. Useful if you want symbols laid out by `lld` for a specific build profile.

### Build ID

Every binary embeds a build ID derived from the inputs' hashes:

```bash
$ go tool buildid ./app
abc123...XYZ/main.a@... + link inputs hash
```

Used to:

- Identify cache entries.
- Distinguish builds in distributed systems.
- Anchor PGO profiles (so they match the binary).

Empty (`-ldflags="-buildid="`) means "byte-identical reproducibility" but loses cache identification.

### Compile/link versions

```bash
$ go tool compile -V
compile version go1.26
$ go tool link -V
link version go1.26
```

The toolchain version is embedded into every compiled object; a 1.26 compiler's output is *not* linkable with a 1.25 linker.

### Object file inspection

A compiled package is an archive (`.a`). Read with `go tool pack` (rarely useful) or via `go build -work`:

```bash
$ go build -work ./pkg
WORK=/tmp/go-build123
...
$ ls /tmp/go-build123/b001
_pkg_.a
```

`b001/_pkg_.a` is the archive. Inspect symbols:

```bash
$ go tool nm /tmp/go-build123/b001/_pkg_.a | head
```

### Reading the SSA HTML

When `GOSSAFUNC=Foo go build` writes `ssa.html`, each pass shows:

- **Phi nodes** (`v1 = phi(v2, v3)`).
- **Block layout** (b1, b2, ...).
- **Lowering decisions** (`v4 = MOVQload`).
- **Spills/reloads** (`v5 = AMD64MOVQload` from memory).

You're looking for:

- Did `prove` (range analysis) elide bounds checks?
- Did `dse` (dead store elimination) remove the assignment?
- Did `regalloc` spill the variable you cared about?

Compiler hackers stare at this; mortals stare at `-gcflags=-m` first.

### Bounds-check elimination (BCE)

```bash
$ go build -gcflags="-d=ssa/check_bce/debug=1" ./pkg
```

Prints which bounds checks were elided. A loop like:

```go
for i := 0; i < len(s); i++ { _ = s[i] }
```

— bounds check elided. A loop like:

```go
for i := 0; i < n; i++ { _ = s[i] }
```

— bounds check stays (compiler can't prove `n <= len(s)`).

### Inlining budget

Inlining is gated by a budget (default ~80 nodes). Print decisions:

```bash
$ go build -gcflags="-m -m" ./pkg 2>&1 | grep inline
```

Disable for a specific function:

```go
//go:noinline
func Foo() { ... }
```

Force inlining (rare; the compiler usually knows better):

Use `//go:inline` (1.22+) on `func` declarations to tell the compiler to ignore the budget. Use sparingly; over-inlining causes icache thrashing.

### Compile speed

The compiler is single-threaded *within* a package but parallelizes *across* packages. `go build` schedules one compile per CPU core. For massive monorepos, this is the dominant build cost.

Tracing the build:

```bash
$ go build -x ./... 2> trace.log
$ grep '^cd ' trace.log | wc -l         # number of package compiles
```

### Link speed

The linker is mostly single-threaded; large binaries (100MB+) take seconds. Since 1.20, the linker uses concurrent symbol resolution; since 1.21, it can deduplicate identical functions across packages.

Reduce link cost by:

- Splitting one giant binary into multiple smaller ones.
- Stripping (`-s -w`).
- Avoiding pulling in giant deps (reflect, generated proto code).

## Standard Library Hooks

- `cmd/compile/internal/*` — the compiler source.
- `cmd/link/internal/*` — the linker source.
- `go/types`, `go/parser`, `go/ast` — the same parser/typechecker libs (pre-1.26 the compiler had its own; modern compiler uses go/types internally).
- `runtime/pprof` — profiles used by `-pgo`.
- `cmd/internal/obj` — object file format.

## Real-World Patterns

### 1. Find an escaping allocation

```bash
$ go build -gcflags=-m ./pkg 2>&1 | grep "escapes to heap" | head
```

Then look at the call site, decide if the allocation is necessary, refactor.

### 2. Set version at link time

```go
// cmd/app/main.go
package main
var (
    Version = "dev"
    Commit  = "unknown"
)
```

```bash
$ go build -ldflags="-X main.Version=$(git describe --tags) -X main.Commit=$(git rev-parse --short HEAD)" -o app
```

### 3. Strip binary

```bash
$ go build -trimpath -ldflags="-s -w" -o app ./cmd/app
```

### 4. PGO-optimized build

```bash
$ # collect profile from prod
$ curl http://prod/debug/pprof/profile?seconds=30 > default.pgo
$ # rebuild
$ go build -pgo=auto -o app ./cmd/app
```

### 5. SSA dump

```bash
$ GOSSAFUNC=BenchmarkHot go build ./pkg
$ open ssa.html
```

### 6. Disable inlining for accurate profiles

```bash
$ go build -gcflags="all=-l" -o app ./cmd/app
$ # now stack traces show every function boundary
```

### 7. Static cgo binary

```bash
$ CGO_ENABLED=1 go build \
    -ldflags="-linkmode=external -extldflags=-static" \
    -o app ./cmd/app
```

(Requires a static glibc or musl; typical Linux builds use Alpine + musl.)

## Anti-Patterns & Gotchas

**`-ldflags="-s -w"` then debugging in dlv.** Stripped binaries have no DWARF; debuggers can't symbolicate. Keep symbols for builds you'll debug.

**`-X` on a `const` or method-receiver variable.** Silently does nothing (or fails). Var must be top-level, string-typed, in a named package.

**`-gcflags=-m` without scoping.** Floods stdout for every package in the build, including stdlib. Always scope: `-gcflags="github.com/me/...=-m"`.

**Confusing `-gcflags="-l"` (no inlining) with `-l` to the linker (no idea what that means).** `gcflags` is for compile; `ldflags` is for link.

**Trusting `-gcflags=-m` of one package as the whole picture.** Decisions in dep packages affect your code. Use `all=-m` for full view.

**Using `//go:noinline` widely to "improve stack traces".** Lower-cost: build with `-gcflags="all=-l"` only during debugging sessions.

**Setting `-buildid=` in CI without understanding cache impact.** Builds become bit-identical, but the cache key derived from build ID changes; you may get unexpected re-builds.

**Trying to interpret SSA output without reading the SSA passes doc.** It's dense and assumes compiler-internals knowledge. Start with `-m`; only escalate to SSA when needed.

**`-linkmode=external` without a working external linker.** Fails with cryptic errors. Either install `ld` from binutils (Linux) or accept internal linking.

**Building with `-N -l` and shipping.** Strips optimization; the binary is 2–3× slower. Use only for `dlv`-friendly builds.

**Stripping DWARF (`-w`) and then trying to profile.** Profiling needs symbols for `pprof` to translate PCs to functions. Keep DWARF for production binaries you'll profile.

**Forgetting `-trimpath` makes paths in profiles relative.** Some tools (web profile viewers) need absolute paths to fetch source. Decide once: trimpath for prod, no trimpath for dev.

## Performance Notes

- Single-package compile: ~50–300 ms.
- Whole stdlib compile (warm cache): <500 ms; cold: ~5 s.
- Linker: ~50 ms for tiny binaries; up to 30 s for a 200MB binary.
- `-S` output: ~100× the file size of the source; pipe to a pager.
- `GOSSAFUNC` HTML: 1–10 MB per function; opens in seconds.
- `-pgo=auto` cost: +10–20% build time, 2–7% runtime speedup.
- `-trimpath`: free.
- `-s -w`: free at build time, saves binary size.

The compile/link phase scales poorly with deps. Trimming imports often beats compiler flag-tweaking for build-time savings.

## How Big Companies Use It

- **Google** uses `-pgo=auto` on internal hot services; PGO profiles are continuously refreshed: https://go.dev/blog/pgo.
- **Kubernetes** uses `-gcflags=-l` in debug builds for stack trace fidelity: https://github.com/kubernetes/kubernetes.
- **Uber** uses `-ldflags="-X"` for version embedding plus `-trimpath`: https://eng.uber.com.
- **Tailscale** uses `GOSSAFUNC` and `-gcflags=-m` to verify hot-path optimizations on `wireguard-go`: https://tailscale.com/blog.
- **Cloudflare** uses `-buildmode=c-archive` and `-linkmode=external` for embedding Go into nginx/lua: https://blog.cloudflare.com/go-cgo-and-the-pluggable-runtime.
- **CockroachDB** uses `-pgo=auto` and a custom CI step to refresh production profiles: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses `-ldflags="-s -w -X main.Version=..."` for every Terraform release: https://github.com/hashicorp/terraform.

## Source Code References

Pinned to `go1.26`.

- Compiler (gc): [`src/cmd/compile`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile).
- SSA passes: [`src/cmd/compile/internal/ssa`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/ssa).
- Escape analysis: [`src/cmd/compile/internal/escape`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/escape).
- Inliner: [`src/cmd/compile/internal/inline`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/inline).
- PGO: [`src/cmd/compile/internal/pgo`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/pgo).
- Linker: [`src/cmd/link`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/link).
- Object format: [`src/cmd/internal/obj`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/internal/obj).
- Build IDs: [`src/cmd/internal/buildid`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/internal/buildid).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Introduction to the Go compiler" (Keith Randall, GopherCon): https://www.youtube.com/watch?v=KINIAgRpkDA.
- "Go SSA backend" (Keith Randall): https://docs.google.com/presentation/d/1Cj_xn4zVxk6gP8B83iDIPdHt2gjSh9zphZ9JLAYYqaA.
- "Inlining in the Go compiler" (Dave Cheney): https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go.
- "Profile-Guided Optimization in Go 1.21" (Michael Pratt): https://go.dev/blog/pgo.
- "Reducing Go binary size" (various, Go wiki): https://github.com/golang/go/wiki/CompilerOptimizations.
- "Go internal linker" (Cherry Mui): https://go.dev/blog/internal-linker.
- "Bounds check elimination" (Dave Cheney): https://dave.cheney.net/2017/01/19/the-empty-struct.

## Exercises / Self-Check

1. Use `-gcflags=-m` on a function that returns a `*T`. Why does the pointer's pointee escape?
2. Build with `GOSSAFUNC=Hot ./pkg` and open `ssa.html`. Find a pass where a phi node is introduced.
3. Add `var Version string` and use `-ldflags="-X main.Version=..."` to inject a release tag. Confirm via `./app`.
4. Compare `-gcflags="all=-l"` vs. no flag on a benchmark. Which is slower, by how much?
5. Strip with `-ldflags="-s -w"` and inspect with `go tool nm`. What symbols remain? What's missing?
