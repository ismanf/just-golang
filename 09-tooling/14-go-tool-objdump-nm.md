# `go tool objdump` and `go tool nm` — Binary Inspection

## TL;DR

`go tool nm` lists every symbol in a Go binary (or `.a` archive): functions, global variables, type metadata, runtime tables. `go tool objdump` disassembles ranges of the binary back into Go assembly (Plan 9 dialect) or native instructions, optionally interleaved with source lines. They share the same symbol-table backend (`debug/gosym`). Use them to: figure out which symbol is bloating a binary, confirm a function got inlined (= no symbol in `nm` output), inspect generated machine code for hot paths, debug `//go:linkname` collisions, and verify what `-ldflags="-s"` actually stripped. Alternatives exist (`llvm-objdump`, `objdump` from binutils) but Go's variants understand Go-specific metadata that the generic tools miss.

## Mental Model

```
   Go binary  (ELF / Mach-O / PE / Wasm)
        │
        ├─ .text          ── code
        ├─ .rodata        ── strings, type info, type.* metadata
        ├─ .data          ── initialized globals
        ├─ .bss           ── zero-init globals
        ├─ .gopclntab     ── PC→func/line tables (Go-specific)
        ├─ .gosymtab      ── symbol table (Go-specific, often empty since 1.2+)
        ├─ .typelink      ── type info pointers
        ├─ .itablink      ── interface table pointers
        ├─ .gcdata/.gcbss ── GC bitmaps for live pointers
        ├─ DWARF          ── debug info (unless stripped with -w)
        └─ buildid        ── content hash
        │
        ▼
   go tool nm bin         ── list all symbols (addr, size, type, name)
   go tool objdump bin    ── disassemble (default: Go assembly with source)
```

`nm` and `objdump` read the binary directly; they don't run it.

## Syntax & Basic Usage

```bash
# nm
$ go tool nm ./bin                       # all symbols
$ go tool nm -size ./bin                 # include sizes
$ go tool nm -sort=size ./bin            # sort by size (asc); -sort=-size desc
$ go tool nm -sort=address ./bin
$ go tool nm -type ./bin                 # filter by symbol type
$ go tool nm -n ./bin                    # numeric sort by address
$ go tool nm pkg.a                       # works on archives too

# objdump
$ go tool objdump ./bin                  # disassemble everything (huge)
$ go tool objdump -s 'main\.Hot' ./bin   # only symbols matching regex
$ go tool objdump -S ./bin               # show source interleaved (needs DWARF)
$ go tool objdump -s 'main\.Hot' -gnu ./bin    # show GNU-style assembly too
```

## Deep Dive

### `nm` symbol types

The single letter at the start of each `nm` line tells you the symbol kind:

| Letter | Meaning                                                       |
|--------|---------------------------------------------------------------|
| `T`    | Text (code) — global function.                                |
| `t`    | Text — local function.                                        |
| `R`    | Read-only data.                                               |
| `D`    | Initialized data.                                             |
| `B`    | BSS — zero-initialized data.                                  |
| `U`    | Undefined — referenced but defined elsewhere (cgo, runtime).  |
| `S`    | Static symbol.                                                |
| `?`    | Unknown.                                                       |

Example output:

```bash
$ go tool nm -size ./bin | head
   1000   4096 t runtime.gcWork.balance
   2000   8192 T runtime.mallocgc
  10000   2048 R type.[]int
  20000     16 D main.config
  30000      8 B main.mu
```

Columns: address, size, type, name.

### Top-N largest symbols

```bash
$ go tool nm -size ./bin | sort -k 2 -nr | head -20
```

Useful when chasing binary bloat. Common culprits:

- Generated protobuf code (`proto.Marshal` for many messages).
- Reflection metadata (`type.*` symbols).
- String tables from `gofmt` output of generated code.
- Templates baked in via `embed.FS`.

### `objdump` output format

```bash
$ go tool objdump -s 'main\.Hot' ./bin
TEXT main.Hot(SB) /Users/me/pkg/main.go
  main.go:10        0x10500            65488b0c2530000000   MOVQ GS:0x30, CX
  main.go:10        0x10509            483b6110             CMPQ 0x10(CX), SP
  main.go:10        0x1050d            762d                 JBE 0x1053c
  main.go:11        0x1050f            4883ec18             SUBQ $0x18, SP
  ...
```

Each line: source line ref, address, raw bytes, Plan-9-style mnemonic. The first three lines of nearly every function are the stack-growth check (`MOVQ GS:0x30`, `CMPQ`, `JBE` to `morestack_noctxt`).

With `-gnu`:

```
  main.go:10  0x10500  65488b0c2530000000  MOVQ GS:0x30, CX // mov %gs:0x30, %rcx
```

The `// mov %gs:0x30, %rcx` is the AT&T/GNU dialect. Useful if you're more familiar with GCC's output.

### Checking if a function inlined

If `go build` inlined `Foo` into all its callers, the binary has no `Foo` symbol — `nm` won't find it.

```bash
$ go tool nm ./bin | grep -F 'main.Foo'
$ # (no output) → fully inlined
```

To confirm, build with `-gcflags=-m`:

```bash
$ go build -gcflags="-m" ./pkg 2>&1 | grep "inlined call to Foo"
```

Or block inlining and rebuild:

```bash
$ go build -gcflags="-l" -o app ./...
$ go tool nm ./app | grep main.Foo   # now visible
```

### Filtering with `-s`

```bash
$ go tool objdump -s '^main\.' ./bin
$ go tool objdump -s 'github\.com/me/pkg\.' ./bin
$ go tool objdump -s 'init$' ./bin     # show every init() function
```

Regex is anchored at the *symbol name*; use `^` and `$` to anchor.

### Source-line interleaving

```bash
$ go tool objdump -S ./bin > listing.txt
```

`-S` needs DWARF info — i.e., the binary was **not** stripped with `-w`. Output is enormous; combine with `-s` to scope.

### Reading `.gopclntab`

The PC→function/line table is what `runtime.FuncForPC` consults. Pre-1.2, `.gosymtab` carried this; since 1.2, `.gopclntab` is canonical. `nm` lists its symbols (`runtime.pclntab`, `runtime.funcdata.0`, etc.). `objdump` doesn't disassemble it (not code).

Programmatic access: `debug/gosym.NewTable(symtab, pcln)`.

### Symbol types under `nm -type`

```bash
$ go tool nm -type ./bin | head
T 0x100000  main.main
T 0x101000  main.Init
B 0x300000  main.cfg
D 0x301000  main.version
```

`-type` shows the same letter as default plus the address column moved. Useful when scripting.

### Inspecting archives

`.a` files (the per-package compiled output that `go build -work` produces):

```bash
$ go build -work ./pkg
WORK=/tmp/go-build123
$ ls /tmp/go-build123/b001
_pkg_.a
$ go tool nm /tmp/go-build123/b001/_pkg_.a | head
```

Lets you see per-package symbols before they're merged into the binary.

### `objdump` and PGO

PGO (Profile-Guided Optimization) may inline or reorder functions. To see what the PGO build produced vs. the baseline:

```bash
$ go build -o baseline ./cmd/app
$ go build -pgo=cpu.pprof -o pgo ./cmd/app
$ diff <(go tool nm -sort=name baseline | awk '{print $3}') \
       <(go tool nm -sort=name pgo      | awk '{print $3}')
```

Symbols missing in `pgo` were probably inlined; new symbols mean code was specialized.

### Stripped binaries

After `go build -ldflags="-s -w"`:

```bash
$ go tool nm ./bin
go tool nm: ./bin: no symbol table
```

`-s` strips the symbol table; `nm` errors out. `objdump` still works but loses the symbol→PC mapping. `runtime/debug.ReadBuildInfo` data and `.gopclntab` are preserved (so stack traces still work) — only the developer-facing symbol table is gone.

### `objdump` for native `.so`/`.dylib`

For non-Go binaries (cgo wrappers, plugins compiled separately), prefer `llvm-objdump` or `objdump` from binutils. Go's `objdump` doesn't understand standard ELF sections beyond what Go needs.

### Cross-platform

`go tool nm` / `objdump` work on any Go binary regardless of host platform — they understand ELF (Linux/BSD), Mach-O (macOS), PE (Windows), and Wasm. So you can disassemble a Linux binary on a Mac, or vice versa.

### Symbol size accuracy

`nm -size` shows *function* size for `T`/`t` symbols. For `R`/`D`/`B`, it shows the data size. For runtime tables, sizes may be approximate (depends on linker padding).

### Stack-frame size

To find a function's stack size, look at its prologue in `objdump`:

```
SUBQ $0x18, SP
```

— 24-byte frame. Or use the (undocumented) `go tool nm -size` which sometimes encodes frame size for text symbols.

For a structured view, build with `-gcflags=-m -m=2` and look for `stack frame size:` lines.

## Standard Library Hooks

- `debug/elf`, `debug/macho`, `debug/pe` — parse ELF, Mach-O, PE.
- `debug/dwarf` — DWARF debug info reader.
- `debug/gosym` — Go-specific symbol table reader.
- `debug/buildinfo` — read build/module info from a binary.
- `cmd/objdump`, `cmd/nm` — the source for these tools.

Reading symbols programmatically:

```go
package main

import (
    "debug/elf"
    "fmt"
    "os"
    "sort"
)

func main() {
    f, _ := elf.Open(os.Args[1])
    defer f.Close()
    syms, _ := f.Symbols()
    sort.Slice(syms, func(i, j int) bool { return syms[i].Size > syms[j].Size })
    for _, s := range syms[:20] {
        fmt.Printf("%-50s %d\n", s.Name, s.Size)
    }
}
```

## Real-World Patterns

### 1. Find the 10 biggest symbols

```bash
$ go tool nm -size ./bin | sort -k 2 -nr | head -10
```

Identify whether bloat is generated code, reflection, embedded assets, or just a giant function.

### 2. Bloat budget in CI

```bash
$ size_bytes=$(go tool nm -size ./bin | awk '{s+=$2} END{print s}')
$ test "$size_bytes" -lt 50000000 || (echo "binary too big: $size_bytes"; exit 1)
```

### 3. Confirm an inlining

```bash
$ go tool nm ./bin | grep -F 'github.com/me/pkg.shortHelper'
$ # empty = inlined; present = not inlined
```

### 4. Look at a function's machine code

```bash
$ go tool objdump -s 'github\.com/me/pkg\.Hot' ./bin | less
```

### 5. Find type metadata bloat

```bash
$ go tool nm -size ./bin | grep '^.*R type\.' | sort -k 2 -nr | head
```

Often points at deeply nested generic instantiations or large slices/maps of complex types.

### 6. Diff symbols between builds

```bash
$ go build -o old ./cmd/app
$ # make a change
$ go build -o new ./cmd/app
$ diff <(go tool nm -sort=name old | awk '{print $3}') \
       <(go tool nm -sort=name new | awk '{print $3}')
```

New symbols are new code; removed symbols are inlined or dead-code-eliminated.

### 7. Inspect runtime function

```bash
$ go tool objdump -s 'runtime\.mallocgc' ./bin | head -50
```

See how the GC allocator was lowered to native instructions.

## Anti-Patterns & Gotchas

**Running `nm` on a stripped binary expecting symbols.** `-ldflags="-s"` strips the symbol table; `nm` errors. Either build without `-s` for inspection or use `objdump` (which still works without full symbols).

**Using `-S` on a `-w`-stripped binary.** DWARF info is gone; source interleaving silently degrades. Build without `-w`.

**Trusting `nm`'s sizes for runtime symbols.** Runtime tables (`.gopclntab`, `.typelink`) span symbols in ways `nm` doesn't always represent precisely.

**Disassembling without filtering.** `go tool objdump ./bin` outputs the entire binary — tens of MB. Always use `-s '<regex>'` to scope.

**Reading objdump output as canonical assembly.** It's Plan 9 dialect. Cross-reference with `-gnu` for AT&T or use `llvm-objdump` for Intel syntax.

**Confusing `T` and `t`.** `T` is exported; `t` is package-local. Both are code; just different linkage.

**Running `nm` on a `vendor/` archive.** Vendor dirs don't contain `.a` files — those live in `$GOCACHE`. To inspect, `go build -work` and look under the temp dir.

**Forgetting that linker symbol names use the import path with `.` instead of `/`.** `github.com/me/pkg.Foo` becomes... `github.com/me/pkg.Foo`. (Some other languages mangle differently; Go doesn't.) But generic types may have angle brackets in their `nm` output (`pkg.Foo[int]`).

**Trying to use `objdump` to find race conditions.** It shows code, not runtime behavior. Use `-race` (`07-concurrency/15-race-detector.md`) for that.

**Diff'ing symbol lists across builds without sorting.** Order depends on link order; sort first.

**Including symbol-size budgets in CI without context.** A 1 MB binary bloat may be a new dep with legitimate value. Add notes when expanding the budget.

## Performance Notes

- `go tool nm` on a 100 MB binary: ~1–5 s.
- `go tool objdump` (full binary): 10–60 s.
- Scoped `objdump -s 'pattern'`: <1 s.
- `nm -size`: same cost as `nm`; sorting is O(N log N) in symbol count.
- Reading via `debug/elf` programmatically: ms per binary; allocate carefully for huge ones.

Sub-second turnaround for scoped queries; multi-second for full sweeps.

## How Big Companies Use It

- **Google** uses internal forks of `nm`/`objdump` for binary size monitoring across Go services: https://go.googlesource.com.
- **Kubernetes** uses `go tool nm` in build size reports: https://github.com/kubernetes/kubernetes.
- **Cloudflare** uses `objdump` to verify that hot paths in their HTTP/3 implementation didn't regress on inlining: https://blog.cloudflare.com/tag/golang.
- **Tailscale** uses [`depaware`](https://github.com/tailscale/depaware) to track dep-induced binary size growth: https://github.com/tailscale/depaware.
- **CockroachDB** monitors binary size per release; uses `nm` to attribute growth to deps: https://github.com/cockroachdb/cockroach.
- **HashiCorp** ships size budgets for Terraform plugin binaries: https://github.com/hashicorp/terraform.
- **The Go team** uses `nm`/`objdump` extensively during compiler-optimization work: https://github.com/golang/go.

## Source Code References

Pinned to `go1.26`.

- `go tool nm`: [`src/cmd/nm`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/nm).
- `go tool objdump`: [`src/cmd/objdump`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/objdump).
- Disassembler: [`src/cmd/internal/obj`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/internal/obj).
- `debug/elf`: [`src/debug/elf`](https://github.com/golang/go/tree/release-branch.go1.26/src/debug/elf).
- `debug/macho`: [`src/debug/macho`](https://github.com/golang/go/tree/release-branch.go1.26/src/debug/macho).
- `debug/pe`: [`src/debug/pe`](https://github.com/golang/go/tree/release-branch.go1.26/src/debug/pe).
- `debug/gosym`: [`src/debug/gosym`](https://github.com/golang/go/tree/release-branch.go1.26/src/debug/gosym).
- `debug/dwarf`: [`src/debug/dwarf`](https://github.com/golang/go/tree/release-branch.go1.26/src/debug/dwarf).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command nm": https://pkg.go.dev/cmd/nm.
- "Command objdump": https://pkg.go.dev/cmd/objdump.
- "Inside the Go compiler — generating code" (Keith Randall): https://www.youtube.com/watch?v=KINIAgRpkDA.
- "How a Go program's binary is built" (Dave Cheney): https://dave.cheney.net/2014/09/14/go-list-your-swiss-army-knife.
- "Reducing the size of Go binaries" (Filippo Valsorda, Mat Ryer): https://github.com/golang/go/wiki/CompilerOptimizations.
- `debug/gosym` design note (Russ Cox): https://research.swtch.com/symbol.

## Exercises / Self-Check

1. Build a small Go program, run `go tool nm -size`, sort by size. What's the biggest symbol? Why?
2. Find a small function in your code that should be inlined. Confirm via `nm` (no symbol) and via `-gcflags=-m` (inline decision).
3. Disassemble a hot loop with `go tool objdump -s 'pattern'`. Are bounds checks present? (Look for `runtime.panicIndex` calls.)
4. Build the same program with and without `-ldflags="-s -w"`. Compare `nm` output and binary size.
5. Write a tiny Go program that uses `debug/elf` to print the 20 largest symbols of a binary. Compare to `nm -size | sort -k 2 -nr | head -20`.
