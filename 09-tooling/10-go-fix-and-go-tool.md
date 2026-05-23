# `go fix` and `go tool` — Migrations and Tool Invocation

## TL;DR

`go fix` is the toolchain's migration helper: it rewrites old API usage to current API usage when the Go team retires or replaces something. Historically it handled big jumps (Go 1.0 → 1.1, `golang.org/x/net/context` → `context`, etc.); since modules took over, most fixes are minor and `go fix` is run only when release notes call it out. **`go tool`** is the dispatch command for invoking toolchain binaries living under `$GOROOT/pkg/tool/$GOOS_$GOARCH/` — the compiler (`compile`), linker (`link`), assembler (`asm`), `pprof`, `trace`, `cover`, `objdump`, `nm`, `vet`, `cgo`, `addr2line`, `pack`, `buildid`, `dist`, `fix`, `test2json`, `covdata`, and more. Since Go 1.24, `go tool <name>` *also* resolves names from the `go.mod` `tool` directive, replacing the old `tools.go` pattern. So `go tool stringer -type=Color` Just Works once `stringer` is in `go.mod`'s `tool` block.

## Mental Model

```
   go tool                 ── dispatches to:
        ├─ built-in tools  ── $GOROOT/pkg/tool/<GOOS>_<GOARCH>/{compile,link,vet,pprof,...}
        └─ module tools    ── resolved from go.mod `tool` directive (1.24+)

   go fix                  ── runs cmd/fix on the named packages
        └─ applies registered AST rewrites for known API migrations
```

`go tool` is the umbrella for "any Go-distributed binary that isn't `go` itself". `go fix` is one specific tool, exposed both as `go fix` and `go tool fix`.

## Syntax & Basic Usage

```bash
# go tool
$ go tool                          # list available tools (1.24+ shows module-pinned ones too)
$ go tool compile -V               # compiler version
$ go tool pprof cpu.out
$ go tool trace trace.out
$ go tool cover -html=cov.out
$ go tool nm ./bin
$ go tool objdump ./bin
$ go tool dist list                # supported GOOS/GOARCH pairs
$ go tool addr2line ./bin
$ go tool buildid ./bin
$ go tool vet ./...
$ go tool stringer -type=Color     # 1.24+, if `tool` directive points to stringer
$ go tool cgo -godefs types.go     # generate Go from C decls

# go fix
$ go fix ./...                     # run registered fixers on all packages
$ go fix -diff ./...               # 1.22+: show what would change
$ go tool fix -force=context ./... # 1.22+: force a specific fix even if not auto-detected
$ go tool fix -help
$ go tool fix -r=context ./...     # 1.22+: equivalent to -force
```

## Deep Dive

### `go tool` — discovery

```bash
$ go tool
addr2line
asm
buildid
cgo
compile
covdata
cover
dist
doc
fix
link
nm
objdump
pack
pprof
test2json
trace
vet

# 1.24+ also lists module-pinned tools:
golang.org/x/tools/cmd/stringer (in module github.com/me/proj)
github.com/golang/mock/mockgen (in module github.com/me/proj)
```

The output mixes:

1. **Built-in tools** in `$GOROOT/pkg/tool/<GOOS>_<GOARCH>/`. These ship with Go and are versioned with the toolchain.
2. **Module tools** (since 1.24) declared via the `tool` directive in `go.mod`.

### Built-in tools (the canonical list)

| Tool         | Purpose                                                                  |
|--------------|--------------------------------------------------------------------------|
| `compile`    | The Go compiler (gc). `go build` invokes it per package.                 |
| `link`       | The Go linker (gld/internal). Produces the final binary.                 |
| `asm`        | The Go assembler (Plan 9 dialect). For `*.s` files.                      |
| `cgo`        | C↔Go interop preprocessor. Generates `_cgo_*.go` and `_cgo_export.c`.    |
| `vet`        | Static analysis. Equivalent to `go vet` (no `tool` prefix needed).        |
| `pprof`      | Profile viewer; `09-tooling/12-go-tool-pprof.md`.                         |
| `trace`      | Execution tracer viewer; `09-tooling/13-go-tool-trace.md`.                |
| `cover`      | Coverage reports from `-coverprofile=` output.                            |
| `covdata`    | Tool for `-cover`-instrumented binary outputs (`GOCOVERDIR`).             |
| `nm`         | Symbol listing (like `nm(1)`); `09-tooling/14-go-tool-objdump-nm.md`.    |
| `objdump`    | Disassembly; same file as nm.                                             |
| `addr2line`  | PC → file:line. Used internally by stack-trace post-processing.           |
| `pack`       | Archive tool for `.a` files (Plan 9 style, used by linker).               |
| `buildid`    | Read/write build IDs in binaries.                                         |
| `dist`       | Bootstrap and metadata. `go tool dist list` shows valid GOOS/GOARCH.      |
| `doc`        | Same as `go doc` (no `tool` prefix needed normally).                      |
| `fix`        | Same as `go fix`. The actual rewriter lives here.                         |
| `test2json`  | Converts `go test -v` output to `-json` format. Used internally.          |

For most users, only `pprof`, `trace`, `cover`, and `vet` come up day-to-day. The compiler/linker are invoked by `go build` for you; you only reach for them when you want to inspect their output (e.g., `go tool compile -S file.go` to see assembly).

### `go tool compile`

```bash
$ go tool compile -S file.go      # print assembly
$ go tool compile -m file.go      # print optimization decisions
$ go tool compile -m=2 file.go    # more detailed
$ go tool compile -W file.go      # show type-checked AST
$ go tool compile -live file.go   # show liveness analysis
$ go tool compile -d=ssa/check/on file.go    # SSA debug
```

These are the same flags you'd pass via `-gcflags=...` in `go build`. Running `compile` directly is rare — usually you want `go build -gcflags=-m ./pkg` instead.

Full flag list:

```bash
$ go tool compile -h
```

### `go tool link`

```bash
$ go tool link -s -w file.a
```

Same as `-ldflags=-s -w` in `go build`. Read symbols via `go tool nm`; disassemble via `go tool objdump`. See `09-tooling/11-go-tool-compile-link.md`.

### `go tool vet`

```bash
$ go tool vet ./...
```

The actual analyzer. `go vet ./...` is a wrapper that resolves package paths and invokes the same binary. The output is identical.

### `go tool dist list`

```bash
$ go tool dist list | head
aix/ppc64
android/386
android/amd64
android/arm
android/arm64
darwin/amd64
darwin/arm64
dragonfly/amd64
freebsd/386
freebsd/amd64
...
```

Lists supported platforms. Filter to first-class ports:

```bash
$ go tool dist list -json | jq '.[] | select(.FirstClass==true) | "\(.GOOS)/\(.GOARCH)"'
```

First-class ports (Linux/amd64, Linux/arm64, darwin/amd64, darwin/arm64, windows/amd64) get full release-blocking CI; others may have lower priority for bugfixes.

### Module tools (1.24+)

Declare tools in `go.mod`:

```
module github.com/me/proj

go 1.26

tool (
    golang.org/x/tools/cmd/stringer
    github.com/golang/mock/mockgen
    honnef.co/go/tools/cmd/staticcheck
)
```

`go mod tidy` figures out the version (uses `latest` if none requested via `go get`). Then:

```bash
$ go tool stringer -type=Color
$ go tool mockgen -source=foo.go
$ go tool staticcheck ./...
```

Each command runs the version pinned in `go.mod`. Cached after first build; reruns are fast.

### Migrating from `tools.go` to `tool` directive

Before 1.24:

```go
//go:build tools
package tools
import (
    _ "golang.org/x/tools/cmd/stringer"
    _ "github.com/golang/mock/mockgen"
)
```

After 1.24:

```bash
$ go get -tool golang.org/x/tools/cmd/stringer@latest
$ go get -tool github.com/golang/mock/mockgen@latest
$ rm tools.go
```

`go.mod` gains the `tool (...)` block. `go get -tool` is the imperative way; `go mod edit -tool=...` is the scriptable way.

### `go fix` — what it does today

Historically `go fix` rewrote stale APIs (Go 1.0 → 1.1 type changes; `golang.org/x/net/context` → `context`; `go/doc.Filter` signature change). Each rewrite is a registered "fix" in `cmd/fix`.

```bash
$ go tool fix -help
fix: a tool for performing transitional code rewrites.

usage: fix [flags] [path ...]

flags:
  -diff   display diffs instead of rewriting
  -force=name1,name2   force these fixes (default: only auto-detected)
  -r=name1,name2       same as -force
```

List registered fixes:

```bash
$ go tool fix -help 2>&1 | grep -A 100 "Available fixes:"
```

Today the active set is small (a handful of fixes covering 1.0→1.1, 1.4→1.5, `context` migration, etc.). Run with `-diff` first to preview.

### `go fix` for big upgrades

When the Go team retires an API, the release notes call out `go tool fix -r=<name>`. For example, when `os.NewFile`'s signature changed years ago, the migration was `go tool fix -r=newfilesignature`.

Modern equivalents are often handled by `gopls`'s code actions or by `staticcheck`'s fix suggestions; `go fix` is the formal compiler-side path.

### `go tool` quirks

- `go tool <name>` searches `$GOROOT/pkg/tool/<GOOS>_<GOARCH>/<name>` first, then module-declared tools.
- Module tools shadow built-in names (uncommon but possible). Naming collision rules favor the explicit `go.mod` declaration.
- Tools can be invoked directly: `/usr/local/go/pkg/tool/darwin_arm64/compile`. The `go tool` wrapper just adds `$PATH` resolution.

### `go tool` arguments

Each tool has its own flag parser. They typically don't recognize `--` for separator (Plan-9 ancestry). Pass the tool's flags after the tool name:

```bash
$ go tool pprof -http=:8080 cpu.out
$ go tool nm -size ./bin
$ go tool objdump -s 'main\.foo' ./bin
$ go tool compile -p main -o foo.a foo.go
```

### Help

```bash
$ go tool <name> -h
$ go tool <name> -help
$ go help tool
```

`go help tool` documents the wrapper; individual tool help is `-h` after the tool name.

## Standard Library Hooks

- `cmd/fix` source — list of registered fixes: [`src/cmd/fix`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/fix).
- `golang.org/x/tools/go/analysis` — modern analysis framework (used by `vet`, `staticcheck`, etc.).
- `cmd/go/internal/tool` — tool dispatcher source.
- `cmd/go/internal/modload` — `tool` directive resolution.
- `internal/diff` — used by `go fix -diff`.

## Real-World Patterns

### 1. List tools

```bash
$ go tool
```

The first thing to run when wondering "what comes with Go?".

### 2. Inspect compiler output

```bash
$ go tool compile -S -B -wb=0 -spectre=all foo.go
$ # or, more commonly:
$ go build -gcflags=-S=foo ./pkg 2>asm.txt
```

### 3. View a profile

```bash
$ go tool pprof -http=:8080 cpu.out
```

### 4. Module tool invocation

```bash
$ go mod edit -tool=golang.org/x/tools/cmd/stringer
$ go mod tidy
$ go tool stringer -type=Color
```

### 5. Test 2 json conversion

```bash
$ go test -v ./... 2>&1 | go tool test2json
```

Useful for piping into UIs that want `-json` but the original run was `-v`.

### 6. Buildid inspection

```bash
$ go tool buildid ./bin
```

Returns the build ID embedded in the binary. Useful for cache invalidation in distribution systems.

### 7. Run `go fix` in diff mode

```bash
$ go fix -diff ./...
```

Preview rewrites; nothing written.

### 8. Cross-platform sanity check

```bash
$ go tool dist list | grep -c '/'
```

Counts supported `GOOS/GOARCH` pairs. Currently ~40+.

## Anti-Patterns & Gotchas

**Calling `go tool compile` directly to "build faster".** It doesn't; you skip the dep graph, the cache, and the linker. `go build` is the right entry point.

**Mixing `tools.go` and `tool` directive (1.24+).** Pick one. Confusing future contributors over which is authoritative.

**Pinning a `tool` directive to `@latest`.** `go mod tidy` may upgrade it whenever someone re-tidies. Pin a specific version via `go get -tool pkg@v1.2.3`.

**Forgetting `go tool` resolves built-in vs. module tools by name only.** A module tool named `compile` would shadow the built-in compiler — strange but allowed.

**Using `go fix` on modern code expecting magic.** Most APIs no longer have registered fixes; `gopls` and `staticcheck` cover what's left.

**Hand-running `go tool link`.** The linker requires a precise set of `.a` files and metadata. Use `go build` and read `-ldflags=...`.

**Pasting `go tool nm` output into a bug report without the binary's build ID.** Symbols may differ across builds. Capture `go tool buildid ./bin` alongside.

**Treating `go tool compile -m` output as authoritative inlining decisions.** It is, for the inputs you compiled; production builds use different `-gcflags` (e.g., `-pgo=auto`), which can change inlining. Profile production with `pprof` for the real picture.

**Running `go tool dist list` and trusting all entries are tier-1.** Many are community-supported or experimental. Filter on `FirstClass: true`.

**Forgetting that `go tool fix` rewrites in place.** Run with `-diff` first or commit before running.

**Running `go tool covdata` without the right `GOCOVERDIR`.** The covdata tool reads the directory `-cover`-built binaries wrote to; if you point it at the wrong directory you get "no coverage data".

## Performance Notes

- `go tool` startup: ~50 ms (Go program init).
- `go tool compile` for a small file: ~100 ms.
- `go tool link` for a small binary: ~200 ms.
- `go tool vet` per package: ~50–200 ms (cached after first run).
- `go tool pprof`'s web UI starts in <1 s.
- `go fix` is fast — AST rewrite, no build.
- Module-tool first invocation: includes a build (~1–5 s); subsequent uses are cache hits.

## How Big Companies Use It

- **Google** uses `go tool` directly for internal release engineering — `go tool buildid` features in their binary-distribution pipelines: https://go.googlesource.com.
- **Kubernetes** uses `go tool` heavily inside `hack/*` scripts (`go tool dist list`, `go tool vet`): https://github.com/kubernetes/kubernetes/blob/master/hack/.
- **Uber** uses `go tool` to wrap profile collection in their internal devstack: https://eng.uber.com/measuring-performance-in-go.
- **CockroachDB** uses `go tool covdata` for cross-binary coverage merging: https://github.com/cockroachdb/cockroach.
- **HashiCorp** wraps `go tool nm` in scripts for binary size budgets (terraform plugin binaries): https://github.com/hashicorp/terraform.
- **The Go team** uses `cmd/fix` to ship migration scripts as part of major-version releases: https://go.dev/doc/go1.21.
- **Tailscale** uses the `tool` directive (1.24+) to pin internal codegen tools (`cloner`, `viewer`) without committing a `tools.go`: https://github.com/tailscale/tailscale.

## Source Code References

Pinned to `go1.26`.

- `go tool` dispatcher: [`src/cmd/go/internal/tool/tool.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/tool/tool.go).
- `go fix`: [`src/cmd/go/internal/fix`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/fix) and the underlying [`src/cmd/fix`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/fix).
- Built-in tool binaries: [`src/cmd/{compile,link,asm,cgo,vet,nm,objdump,addr2line,buildid,pack,dist,doc,fix,pprof,trace,cover,covdata,test2json}`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd).
- `tool` directive parser: [`src/cmd/go/internal/modload/modfile.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/modfile.go).
- `go get -tool`: [`src/cmd/go/internal/modget`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/modget).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command go — Tool": https://pkg.go.dev/cmd/go#hdr-Run_specified_go_tool.
- "Command go — Update packages to use new APIs": https://pkg.go.dev/cmd/go#hdr-Update_packages_to_use_new_APIs (i.e., `go fix`).
- "Go 1.24 tool directive": https://tip.golang.org/doc/go1.24#tools.
- "Inside the Go compiler" (Keith Randall): https://www.youtube.com/watch?v=KINIAgRpkDA.
- `cmd/fix` README: https://github.com/golang/go/tree/master/src/cmd/fix.

## Exercises / Self-Check

1. Run `go tool` and identify three tools you've never used. Look one up via `go tool <name> -h`.
2. Add `golang.org/x/tools/cmd/stringer` to your `go.mod` via `go get -tool`. Confirm `go tool stringer -type=Foo` runs the pinned version.
3. Run `go fix -diff ./...` on a small project. Does anything come up?
4. Use `go tool dist list -json` to count first-class platforms. How many?
5. Compare invoking `go vet ./...` vs. `go tool vet ./...`. What's the difference, if any?
