# `go build` — Compiling Go Programs

## TL;DR

`go build` compiles the named packages and writes an executable (for `main`) or discards the result after type-checking (for libraries). It is **incremental** — every compile unit is keyed by its inputs (source content, build tags, compiler version, `-gcflags`, dependencies' export data) and cached under `$GOCACHE`. A cached hit returns instantly. Reproducible builds depend on three flags: **`-trimpath`** (strip absolute paths from the binary), **`-buildvcs`** (embed or omit VCS info), and **`-buildmode`** (pick the output format: `exe`, `pie`, `c-archive`, `c-shared`, `plugin`, `shared`, `archive`). On Go 1.26 the default linker is the internal one for `exe` mode; PIE binaries on Linux still need the external linker when CGO is on. Output goes to `./<pkg>` in the current directory unless `-o` redirects it; libraries leave nothing on disk by design.

## Mental Model

```
   go build ./...
        │
        ▼
   load packages  ──►  resolve deps via go.mod / go.work
        │                    │
        ▼                    ▼
   compute action graph (one node per .a file to produce)
        │
        ▼
   for each action:
        ├─ hash inputs (src + flags + dep hashes + toolchain id)
        ├─ look up $GOCACHE/<hash>
        │     hit  → reuse .a, skip compile
        │     miss → run compile or link, write to cache
        ▼
   final link  ──► writes binary to ./<pkg>  (or $GOBIN for install)
```

The cache is content-addressed: changing a comment in one file invalidates only that file's compile action (and the link that consumes it). Changing a `-gcflags` value invalidates every action whose flags hash changed.

## Syntax & Basic Usage

```bash
$ go build                  # build current dir, write ./<dir>
$ go build .                # same
$ go build ./...            # build every package in the module
$ go build -o bin/app ./cmd/app
$ go build -v ./...         # print package names as they compile
$ go build -x ./cmd/app     # print every command the toolchain runs
$ go build -n ./cmd/app     # like -x but don't actually run
```

`go build` without arguments builds the package in the current directory; with a `main` package, it writes a binary named after the directory. A non-main package builds silently (cache populates, no output).

```bash
$ ls
main.go  go.mod
$ go build
$ ls
app   main.go  go.mod
```

The binary name comes from the **directory containing `main`**, not the `.go` file name.

## Deep Dive

### The action graph

`go build` is internally a DAG of *actions*: load, compile, link, install, vet, test, etc. Each action lists its inputs (files, flags, dependency outputs) and produces an output (an archive `.a`, a binary, a piece of metadata). Run `go build -x` to dump the script the toolchain executes; `go build -debug-actiongraph=/tmp/g.json` (since 1.18) emits the graph as JSON for tools like `actiongraph` to visualize.

Implementation: [`src/cmd/go/internal/work/action.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/work/action.go).

### Build cache (`$GOCACHE`)

```bash
$ go env GOCACHE
/Users/me/Library/Caches/go-build         # macOS
/home/me/.cache/go-build                  # Linux
%LocalAppData%\go-build                   # Windows
```

Each cache entry is two files under `<hash[0:2]>/<hash>-a` (the artifact) and `<hash>-d` (metadata). Default size cap is 5 GiB; LRU eviction runs lazily. `go clean -cache` wipes it; `go clean -testcache` wipes just test-result entries.

```bash
$ du -sh "$(go env GOCACHE)"
1.4G    /Users/me/Library/Caches/go-build
$ go clean -cache
```

The hash inputs include: file content, file permissions (executable bit only), `GOOS`/`GOARCH`, `CGO_ENABLED`, `GOEXPERIMENT`, compiler version, every `-gcflags`/`-ldflags`/`-asmflags`/`-tags`, and every dependency action's hash. Anything that can change the output is in the hash; anything that cannot (the current time, the working directory after `-trimpath`) is excluded.

Source: [`src/cmd/go/internal/cache/hash.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/cache/hash.go).

### `-trimpath`

```bash
$ go build -trimpath -o app ./cmd/app
```

Strips file-system prefixes from the compiled binary. Without it, every `runtime.FuncForPC(...).FileLine(...)` call and every panic stack trace contains `/Users/me/projects/...`. With it, paths look like `github.com/me/proj/internal/foo/bar.go` (the module path + the in-module relative path) and `$GOROOT/src/...`.

Two reasons to set it:

1. **Reproducibility** — two machines with different `$HOME` produce byte-identical binaries.
2. **Privacy/security** — leaks of build-machine layout vanish from stack traces.

Best practice: set in CI builds (`go build -trimpath`) and in `goreleaser` configs. Don't set during local debugging — keeping absolute paths helps your IDE jump to source.

### `-buildvcs`

Since 1.18 the linker embeds VCS metadata into `runtime/debug.BuildInfo` (commit hash, dirty flag, timestamp). Control via:

```bash
$ go build -buildvcs=true  ./cmd/app   # embed (default if .git found)
$ go build -buildvcs=false ./cmd/app   # omit
$ go build -buildvcs=auto  ./cmd/app   # embed if VCS metadata is available (default)
```

Read it back:

```go
bi, _ := debug.ReadBuildInfo()
for _, s := range bi.Settings {
    if s.Key == "vcs.revision" { fmt.Println("commit:", s.Value) }
    if s.Key == "vcs.modified" { fmt.Println("dirty:",  s.Value) }
}
```

Or from outside the process: `go version -m ./app`. Worth disabling only when you need bit-identical builds and don't want the commit hash to participate in the hash.

### `-buildmode`

| Mode         | Output                          | Use case                                       |
|--------------|---------------------------------|------------------------------------------------|
| `exe`        | Statically linked executable    | Default                                        |
| `pie`        | Position-independent executable | Default on darwin/arm64, hardening on linux   |
| `c-archive`  | `.a` callable from C            | Embed Go into C programs (static link)         |
| `c-shared`   | `.so`/`.dylib`/`.dll` for C     | Embed Go via shared lib                        |
| `shared`     | Shared object of std + deps     | Used by `-linkshared` (rare)                   |
| `plugin`     | `.so` loadable via `plugin.Open`| Hot-pluggable Go (linux/darwin only)           |
| `archive`    | Single-package `.a`             | Internal use                                   |

```bash
$ go build -buildmode=c-archive -o libgo.a ./cmd/embed
$ go build -buildmode=pie       -o app    ./cmd/app
$ go build -buildmode=plugin    -o p.so   ./cmd/myplugin
```

PIE adds ~5% startup cost and is required on hardened Linux distros. `plugin` mode has unpleasant restrictions (must match every package version exactly across the host and the plugin) — see `11-low-level/09-build-tags.md` for the long story.

### `-tags`

```bash
$ go build -tags=integration,prod ./...
```

Each tag becomes available to `//go:build` constraints (`//go:build integration && !test`). Tags are *additive*: the default tag set is empty plus the `GOOS`/`GOARCH` implicit tags. See `11-low-level/09-build-tags.md` for the constraint mini-language.

### `-gcflags`, `-ldflags`, `-asmflags`, `-gccgoflags`

Pass flags to the underlying tools. Use the *package-pattern* form to scope:

```bash
$ go build -gcflags=all=-N\ -l ./...                  # disable optimization+inlining for all
$ go build -gcflags="github.com/me/...=-m" ./...       # print inlining decisions for my packages
$ go build -ldflags="-s -w" ./cmd/app                  # strip debug info + symbol table
$ go build -ldflags="-X main.version=1.2.3" ./cmd/app  # set a string variable at link time
```

The `pkg=flag1\ flag2` form (with `=`) scopes flags to a pattern; without it, flags apply to *all* packages in the build. `-X importpath.var=value` requires the variable to be a `var x = "default"` of type string in the named import path.

See `11-low-level/02-reflect-performance.md` for what `-gcflags=-m` produces (escape analysis output).

### `-mod` and `-modcacherw`

```bash
$ go build -mod=mod        ./...   # may update go.mod
$ go build -mod=readonly   ./...   # error if go.mod would change (CI default)
$ go build -mod=vendor     ./...   # use vendor/ exclusively
$ go build -modcacherw     ./...   # leave $GOMODCACHE writable (default: read-only)
```

`-mod=readonly` is what CI should set: a build that needs to edit `go.mod` is a sign that someone forgot `go mod tidy`. `-modcacherw` exists because the default read-only module cache breaks naive `rm -rf` cleanup.

### `-race`, `-msan`, `-asan`

```bash
$ go build -race  ./...    # link the race detector
$ go build -msan  ./...    # MemorySanitizer (linux/amd64,arm64; CGO required)
$ go build -asan  ./...    # AddressSanitizer (Go 1.18+; CGO required)
```

`-race` instruments memory accesses; binaries run ~2–10× slower and use ~5–10× more RAM. Always run integration tests with `-race`; never ship to prod with it. See `07-concurrency/15-race-detector.md` for the long story.

### `-cover` for production binaries (since 1.20)

```bash
$ go build -cover -o app ./cmd/app
$ GOCOVERDIR=/tmp/cov ./app             # binary writes coverage on exit
$ go tool covdata textfmt -i /tmp/cov -o cov.txt
$ go tool cover -html=cov.txt
```

Allows measuring coverage of long-running services in production-like environments, not just `go test`. See `10-testing/11-coverage.md`.

### `-pgo` (profile-guided optimization, since 1.21; default since 1.21)

```bash
$ go build -pgo=auto ./cmd/app          # auto-discover default.pgo next to main.go
$ go build -pgo=cpu.pprof ./cmd/app     # specific profile
$ go build -pgo=off ./cmd/app           # disable
```

A `default.pgo` file next to `package main` is picked up automatically. PGO tends to deliver 2–7% speedups on hot-path-heavy workloads; cost is one extra compile pass.

### `-C dir` (since 1.20)

```bash
$ go build -C /path/to/proj ./cmd/app
```

Equivalent to `(cd /path/to/proj && go build ./cmd/app)` but cleaner in scripts.

### Cross-compilation

```bash
$ GOOS=linux   GOARCH=amd64 go build -o app-linux  ./cmd/app
$ GOOS=darwin  GOARCH=arm64 go build -o app-mac    ./cmd/app
$ GOOS=windows GOARCH=amd64 go build -o app.exe    ./cmd/app
$ GOOS=js      GOARCH=wasm  go build -o app.wasm   ./cmd/app
```

Pure Go cross-compiles for free (no toolchain swap). CGO complicates things — you need a cross-compiling C toolchain (`CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 …`).

### `GOAMD64`, `GOARM`, `GOPPC64`, `GORISCV64`

```bash
$ GOAMD64=v3 go build ./cmd/app
```

Microarchitecture levels (since 1.18 for amd64). `v1` is the baseline (SSE2); `v3` enables AVX2; `v4` enables AVX-512. Each level adds ~2–8% on numerics. Default is `v1` for compatibility.

### Linking the standard library statically vs. dynamically

Go's standard library is statically linked into every binary — that's why `hello world` is 1.8 MB. To shrink:

```bash
$ go build -ldflags="-s -w" ./cmd/app                   # -s: strip symbol table; -w: strip DWARF
$ upx --best app                                         # external packer
```

`-s -w` reduces size by ~30%; UPX shaves another 60% at the cost of slower startup and broken `runtime/debug.BuildInfo`.

## Standard Library Hooks

- `runtime/debug.ReadBuildInfo` — read embedded module/VCS info at runtime.
- `debug/buildinfo` — read it from any binary on disk.
- `runtime.GOOS`, `runtime.GOARCH` — current target.
- `runtime.Version()` — toolchain version string (`go1.26.1`).
- `os.Executable()` — path to the running binary.
- `go/build` — the older (pre-modules) package for parsing build constraints; mostly kept for `gopls` and analysis tools.

## Real-World Patterns

### 1. Reproducible release binary

```bash
$ go build \
    -trimpath \
    -buildvcs=true \
    -ldflags="-s -w -X main.version=$VERSION -X main.commit=$COMMIT -buildid=" \
    -o dist/app \
    ./cmd/app
```

`-buildid=` zeros out the random build ID for byte-identical reproducibility across machines.

### 2. Smaller binary

```bash
$ go build -trimpath -ldflags="-s -w" -o app ./cmd/app
$ ls -lh app                          # ~25% smaller than default
```

Don't pair with `-race` (race-instrumented binaries need symbols).

### 3. Reading build info from inside the binary

```go
package main

import (
    "fmt"
    "runtime/debug"
)

func main() {
    bi, _ := debug.ReadBuildInfo()
    fmt.Println("module:", bi.Main.Path)
    fmt.Println("go:    ", bi.GoVersion)
    for _, s := range bi.Settings {
        switch s.Key {
        case "vcs.revision", "vcs.time", "vcs.modified", "GOOS", "GOARCH":
            fmt.Printf("%-15s %s\n", s.Key+":", s.Value)
        }
    }
}
```

### 4. Per-package gcflags

```bash
$ go build -gcflags="github.com/me/proj/internal/hot=-l" ./...
```

Disable inlining only in one package — useful when profiling and you want stable function boundaries.

### 5. Multi-platform release (no goreleaser)

```bash
for os in linux darwin windows; do
  for arch in amd64 arm64; do
    [[ "$os" == "windows" && "$arch" == "arm64" ]] && continue
    out="dist/app-${os}-${arch}"
    [[ "$os" == "windows" ]] && out="${out}.exe"
    GOOS=$os GOARCH=$arch go build -trimpath -o "$out" ./cmd/app
  done
done
```

## Anti-Patterns & Gotchas

**Forgetting `-trimpath` on release builds.** Stack traces in production logs reveal your build server's path layout. Set it in CI.

**Stripping with `-s -w` then trying to use a debugger.** `dlv` needs DWARF. If you stripped, you'll need to rebuild.

**Pinning `GOFLAGS="-trimpath"` in `.envrc`.** This is fine in CI but hurts local debugging: your IDE can't follow `go-to-definition` into `$GOROOT` if paths are stripped.

**Using `go install` instead of `go build` for application binaries.** `go install` writes to `$GOBIN` (usually `$HOME/go/bin`) and is meant for tools. Application binaries belong in `dist/` or `bin/` next to the project.

**Trusting the absence of `-buildvcs=false`.** If `.git` doesn't exist (e.g., a CI checkout that deleted it), `vcs.revision` is empty silently. Verify with `go version -m`.

**Building a `plugin` and a host that don't share the exact same module versions.** `plugin.Open` panics at runtime, far from the build that caused it. Plugins are fragile; prefer `gRPC` subprocesses.

**`-gcflags="-m"` without a package pattern.** Floods stdout with inlining decisions for every package in the build, including the standard library. Always scope: `-gcflags="github.com/me/...=-m"`.

**Assuming `go build` is silent on warnings.** It is — `go vet` is what runs analyzers. If you want lint feedback during build, integrate `staticcheck` or `golangci-lint` separately.

**Setting `CGO_ENABLED=0` in `goreleaser` without checking.** Disabling CGO breaks `net.LookupHost` (falls back to pure-Go DNS, which is fine) and any sqlite driver that needs it. Default-on for the host platform, off for cross-compile.

**Mixing `-race` and release builds.** A race build is 2–10× slower. Always a separate target (e.g., `make race-test`).

**Forgetting that `GOAMD64=v3` produces SIGILL on old CPUs.** Setting it for a binary distributed to customers may crash on pre-Haswell hardware. Default `v1` unless you control the deployment target.

## Performance Notes

- Cold build of a 50k-LoC project: ~5–15 s depending on dep count.
- Warm build (cache hit): <100 ms even for large projects.
- Linking dominates: linker takes 30–50% of a clean rebuild on large binaries.
- Internal linker (default): single-threaded, fast for typical sizes.
- External linker (`-extld`): needed for CGO; slower but supports more buildmodes.
- `-race` build: ~2× compile time, ~3–5× binary size, ~2–10× runtime cost.
- `-pgo=auto`: adds one extra compile pass per package (about +10–20% build time) for 2–7% runtime speedup.
- `-trimpath` is free at build time; no runtime cost.

Cache hit rate matters most. Running `go clean -cache` on a CI runner that doesn't share cache between jobs is a build-time disaster — most CI systems support cache persistence (GitHub Actions `actions/setup-go` does it automatically).

## How Big Companies Use It

- **Google** uses Bazel for most internal builds, but external Go releases use `go build` with `-trimpath` and reproducibility verified across machines: https://go.dev/blog/rebuild.
- **Uber** builds 400+ Go services with a shared module cache and `-trimpath -buildvcs=true` for traceability: https://github.com/uber-go/guide.
- **Cloudflare** ships statically linked Linux binaries with `-ldflags="-s -w"` and a custom symbol-stripping pass: https://blog.cloudflare.com/tag/golang/.
- **Tailscale** uses `-pgo=auto` since 1.21 on `tailscaled`, reporting a 4–5% throughput gain on encryption-heavy paths: https://tailscale.com/blog.
- **HashiCorp** signs every release binary with `cosign` and publishes the `go build` flags used as part of provenance: https://www.hashicorp.com/blog/announcing-hashicorp-product-security-improvements.
- **CockroachDB** builds for `GOAMD64=v3` for their cloud offering since they control the deployment hardware: https://github.com/cockroachdb/cockroach.
- **Discord's Go services** rely on `goreleaser` driving `go build` with `-trimpath` and per-platform `GOAMD64` tuning.

## Source Code References

Pinned to `go1.26`.

- `go build` command: [`src/cmd/go/internal/work/build.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/work/build.go).
- Action graph & exec: [`src/cmd/go/internal/work/exec.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/work/exec.go).
- Build cache: [`src/cmd/go/internal/cache`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/cache).
- Hash inputs: [`src/cmd/go/internal/cache/hash.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/cache/hash.go).
- Build info embedding: [`src/cmd/go/internal/load/pkg.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/load/pkg.go) (search for `buildvcs`).
- `runtime/debug.ReadBuildInfo`: [`src/runtime/debug/mod.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/debug/mod.go).
- `debug/buildinfo`: [`src/debug/buildinfo/buildinfo.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/debug/buildinfo/buildinfo.go).
- Build modes: [`src/cmd/go/internal/work/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/work/init.go).
- PGO support: [`src/cmd/compile/internal/pgo`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/compile/internal/pgo).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Compile and install packages and dependencies": https://pkg.go.dev/cmd/go#hdr-Compile_and_install_packages_and_dependencies.
- "Go Reproducibility" (Russ Cox): https://go.dev/blog/rebuild.
- "Build Cache" deep dive: https://pkg.go.dev/cmd/go#hdr-Build_and_test_caching.
- "PGO in Go" (Michael Pratt): https://go.dev/blog/pgo.
- "Reducing Go binary size" (Liz Rice, Filippo Valsorda etc.): https://github.com/golang/go/wiki/CompilerOptimizations.
- "GOAMD64 microarchitecture levels": https://github.com/golang/go/wiki/MinimumRequirements#amd64.
- "Cross-compiling Go" (Dave Cheney): https://dave.cheney.net/2015/08/22/cross-compilation-with-go-1-5.

## Exercises / Self-Check

1. Build the same binary twice on two machines with different `$HOME` values. Without `-trimpath`, are they byte-identical? With `-trimpath -ldflags="-buildid="`?
2. Use `go build -x ./cmd/app` and identify the three highest-level toolchain commands (compile, asm, link). Which one took the longest?
3. Set `-gcflags="all=-m"` on a small program and find one function that escapes to the heap. Adjust the code so it no longer escapes.
4. Build a binary with and without `-pgo=auto` after collecting a CPU profile. Measure the runtime delta on the same workload.
5. Use `go version -m ./bin` on a release binary you trust and verify that `vcs.modified=false`. What does it mean if it's `true`?
