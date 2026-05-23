# `go env` — Environment Variables

## TL;DR

`go env` prints every variable the `go` command consults. Some are set by the user (`GOOS`, `GOARCH`, `GOPROXY`); others are computed (`GOROOT`, `GOPATH`, `GOTOOLDIR`); a handful default to OS conventions (`GOCACHE`, `GOMODCACHE`). `go env -w KEY=VAL` writes values to `$GOENV` (typically `~/.config/go/env`), which persists across shells — this is the canonical way to set, say, an internal `GOPROXY` once for every Go project on the machine. The variable surface is large (60+ on a 1.26 install) but a small subset matters daily: `GOPROXY`, `GOPRIVATE`, `GOSUMDB`, `GOFLAGS`, `GOCACHE`, `GOMODCACHE`, `CGO_ENABLED`, `GOOS`/`GOARCH`, `GOMAXPROCS` (runtime, not build), and `GOTOOLCHAIN`. `GODEBUG` is a related but distinct namespace: it controls runtime behavior, not the toolchain.

## Mental Model

```
   go env
        │
        ├─ User-settable    ── GOPROXY, GOPRIVATE, GOFLAGS, GOTOOLCHAIN, ...
        ├─ OS-derived        ── HOME, OS, ARCH, ...
        ├─ Computed          ── GOROOT (binary location), GOPATH (default ~/go), GOTOOLDIR
        └─ Sticky (in $GOENV)── written by `go env -w`, survives shell restarts

   Precedence (highest first):
        1. Shell env (`GOPROXY=foo go build`)
        2. $GOENV file (`go env -w GOPROXY=foo`)
        3. Defaults
```

`go env -w` writes to `$GOENV`, which is `${XDG_CONFIG_HOME:-$HOME/.config}/go/env`. Settings here persist; shell env wins on a per-invocation basis.

## Syntax & Basic Usage

```bash
$ go env                          # print all variables (a lot of them)
$ go env GOPROXY                  # print one
$ go env GOPROXY GOPRIVATE        # print several
$ go env -json                    # machine-readable
$ go env -json GOPROXY GOCACHE

$ go env -w GOPROXY=https://internal-proxy,direct
$ go env -w CGO_ENABLED=0
$ go env -u GOPROXY               # unset (revert to default)
$ cat "$(go env GOENV)"           # see what's pinned
```

## Deep Dive

### The big four: `GOPROXY`, `GOPRIVATE`, `GOSUMDB`, `GONOSUMCHECK`

```bash
$ go env GOPROXY
https://proxy.golang.org,direct
$ go env GOPRIVATE
""
$ go env GOSUMDB
sum.golang.org
```

- **GOPROXY**: comma-separated proxy URLs to query for module downloads. `direct` means "go directly to the VCS". `off` means "no network — fail if the cache misses".
- **GOPRIVATE**: comma-separated globs of module paths that should *bypass* `GOPROXY` and `GOSUMDB`. Set this for internal/corporate modules.
- **GOSUMDB**: the checksum database to consult for new module-versions. `off` disables (only do this for `GOPRIVATE` paths).
- **GONOSUMCHECK** / **GONOSUMDB**: legacy; superseded by `GOPRIVATE`.

Typical corporate setup:

```bash
$ go env -w \
    GOPROXY='https://athens.internal,https://proxy.golang.org,direct' \
    GOPRIVATE='*.corp.example.com,github.com/example-corp/*'
```

See `06-packages-modules/08-private-modules-and-goproxy.md` for the long story.

### `GOFLAGS`

```bash
$ go env -w GOFLAGS='-mod=readonly -trimpath'
```

Flags inserted into *every* `go` command. Useful for:

- `-mod=readonly` — CI safety (fail on `go.mod` edits).
- `-trimpath` — strip absolute paths in build output.
- `-tags=...` — apply build tags by default.
- `-count=1` — bust the test cache on every `go test`.

Per-shell override: `GOFLAGS='-mod=mod' go build`.

### `GOTOOLCHAIN`

Introduced in 1.21. Controls toolchain switching:

```bash
$ go env GOTOOLCHAIN
auto                  # default: respect go.mod's toolchain line
```

| Value          | Meaning                                                           |
|----------------|-------------------------------------------------------------------|
| `auto`         | Switch to the toolchain `go.mod` requires; download if needed.    |
| `local`        | Always use the installed toolchain; error if `go.mod` needs newer.|
| `path`         | Use whatever `go` is on `$PATH`.                                   |
| `go1.26.1`     | Use exactly this version (download if not present).               |
| `go1.26.1+auto`| Pin a floor; allow upgrades above it per go.mod.                  |

For an air-gapped CI: `GOTOOLCHAIN=local`.

### `GOROOT`, `GOPATH`, `GOBIN`

- **GOROOT**: where Go itself lives (`/usr/local/go` or similar). Computed from `go`'s binary location; rarely worth setting manually.
- **GOPATH**: legacy workspace root; defaults to `~/go`. Even in module mode, `$GOPATH/pkg/mod` is the default `GOMODCACHE` and `$GOPATH/bin` is the default `GOBIN`.
- **GOBIN**: where `go install` writes binaries. Defaults to `$GOPATH/bin`. Many setups override to `~/.local/bin` or similar.

```bash
$ go env -w GOBIN=$HOME/.local/bin
$ export PATH=$HOME/.local/bin:$PATH
```

### `GOCACHE` and `GOMODCACHE`

```bash
$ go env GOCACHE
/Users/me/Library/Caches/go-build       # macOS
$ go env GOMODCACHE
/Users/me/go/pkg/mod
```

- **GOCACHE**: build cache (see `09-tooling/01-go-build.md`). Capped at 5 GiB by default.
- **GOMODCACHE**: module download cache.

Wipe with `go clean -cache` and `go clean -modcache`. In Docker, mount both to persist across container runs:

```bash
$ docker run -v $HOME/.cache/go-build:/root/.cache/go-build \
             -v $HOME/go/pkg/mod:/go/pkg/mod \
             -w /src golang:1.26 go build ./...
```

### `GOOS` and `GOARCH`

The target platform. Cross-compile by setting both:

```bash
$ GOOS=linux GOARCH=arm64 go build -o app
```

Combined list: `go tool dist list` (`09-tooling/10-go-fix-and-go-tool.md`).

### `GOAMD64`, `GOARM`, `GOPPC64`, `GORISCV64`

Microarchitecture levels for the corresponding `GOARCH`:

- **GOAMD64**: `v1` (baseline, SSE2), `v2` (POPCNT, SSE4.2), `v3` (AVX2, default for Go ≥1.22 builds on amd64v3-capable hosts), `v4` (AVX-512).
- **GOARM**: `5`, `6`, `7` (default 7 for new builds).
- **GOPPC64**: `power8`, `power9`, `power10`.
- **GORISCV64**: `rva20u64`, `rva22u64`.

```bash
$ GOAMD64=v3 go build ./...
```

Each level gates the compiler to emit those instructions; the resulting binary is incompatible with lower-tier CPUs.

### `CGO_ENABLED`

```bash
$ go env CGO_ENABLED
1
```

`1` = cgo is allowed; the toolchain will use the system C compiler when needed. `0` = disable cgo; certain packages (`net.LookupHost`, sqlite drivers) fall back to pure-Go implementations or fail at build.

For cross-compiling, default flips to `0` unless `CC` is set explicitly:

```bash
$ CGO_ENABLED=1 CC=aarch64-linux-gnu-gcc GOOS=linux GOARCH=arm64 go build ./...
```

### `CC`, `CXX`, `AR`, `CGO_CFLAGS`, `CGO_LDFLAGS`

C-toolchain configuration for cgo. Defaults to `cc`/`c++`/`ar` from `$PATH`; override per project via env or `#cgo` directives in source.

### `GO111MODULE`

Legacy. In Go ≥1.16, modules are always on; `GO111MODULE=off` is essentially gone. Default is `on`; `auto` is treated as `on`. You'd only see this set if a script is carrying ancient configs forward.

### `GOEXPERIMENT`

Feature flags for the compiler/runtime:

```bash
$ GOEXPERIMENT=loopvar go build ./...        # pre-1.22, opt into 1.22 loopvar semantics
$ GOEXPERIMENT=arenas go build ./...         # opt into memory arenas (experimental)
$ GOEXPERIMENT=cgocheck2 go build ./...      # stricter cgo pointer checks
$ go env GOEXPERIMENT
""                                            # default: none
```

Comma-separated list. The set changes per release; check `go doc internal/goexperiment` (or [the source](https://github.com/golang/go/blob/master/src/internal/goexperiment/flags.go)).

### `GODEBUG`

A *runtime* variable (not toolchain). Tunes the runtime:

```bash
$ GODEBUG=gctrace=1 ./app           # log every GC
$ GODEBUG=schedtrace=1000 ./app     # log scheduler stats every 1000ms
$ GODEBUG=allocfreetrace=1 ./app    # log every alloc/free (very noisy)
$ GODEBUG=httpmuxgo121=1 ./app      # restore pre-1.22 mux behavior
$ GODEBUG=panicnil=1 ./app          # restore pre-1.21 nil-panic behavior
```

See `12-runtime/14-godebug.md` for a full catalog.

Importantly, `go env` doesn't list `GODEBUG` because it's not a build/toolchain variable — it's read by the runtime at process start.

### `GOMAXPROCS`

Also runtime, not build. Sets the OS-thread parallelism:

```bash
$ GOMAXPROCS=4 ./app
```

In containers, since Go 1.5 default = number of logical CPUs the kernel reports. In cgroup-limited containers, that can over-count; use [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs) to clamp to cgroup limits. (Go 1.25+ reads cgroup CPU quota natively.)

### `GOROOT_FINAL`

Used when the Go install is built in one location but moves to another (Linux distro packaging). Sets the runtime-reported `GOROOT` independent of where binaries actually live.

### Setting via `go env -w`

```bash
$ go env -w GOPROXY=https://internal.proxy
$ cat $(go env GOENV)
GOPROXY=https://internal.proxy
```

`go env -u KEY` removes the line; the variable reverts to default.

### Common non-obvious variables

| Variable                  | Purpose                                                          |
|---------------------------|------------------------------------------------------------------|
| `GOTMPDIR`                | Temp dir for `go build`/`go run`. Default: `$TMPDIR`.            |
| `GOWORK`                  | Path to `go.work`. `off` disables workspace mode.                |
| `GOINSECURE`              | Globs of modules to fetch over plain HTTP/insecure HTTPS.        |
| `GOVCS`                   | Per-prefix VCS allow-list (e.g., `*:git`).                       |
| `GOTRACEBACK`             | Runtime: `none`/`single`/`all`/`system`/`crash`.                 |
| `GOSSAFUNC`               | Build: dump SSA HTML for matching function name.                 |
| `GOMEMLIMIT`              | (1.19+) Soft memory limit for the GC; e.g., `8GiB`.              |
| `GOGC`                    | (Runtime) GC target percentage; default 100. Set to `off` to disable. |
| `GODEBUG_DISCARD_DEFINES` | Hidden flag for debugging compiler issues.                        |

### `go env` inside a Dockerfile

```dockerfile
FROM golang:1.26
RUN go env -w GOPROXY=https://athens.internal
```

The `go env -w` persists into the image; subsequent `go build`s honor it.

### `go env -json` for tooling

```bash
$ go env -json | jq .GOMODCACHE
"/Users/me/go/pkg/mod"
```

Useful in scripts; avoids parsing the human-readable output.

## Standard Library Hooks

- `os.Getenv` — read env vars at runtime.
- `runtime.GOOS`, `runtime.GOARCH` — compile-time platform.
- `runtime.GOMAXPROCS()` — read/set the runtime parallelism.
- `runtime.GOROOT()` — Go install path.
- `runtime/debug.ReadBuildInfo()` — build settings (some env vars are recorded here).
- `internal/goexperiment` — gated feature flags.

## Real-World Patterns

### 1. Corporate-wide module proxy

```bash
$ go env -w GOPROXY='https://athens.corp,https://proxy.golang.org,direct'
$ go env -w GOPRIVATE='*.corp.example.com'
$ go env -w GOSUMDB=off                # only for GOPRIVATE; better to leave default
```

### 2. Reproducible CI defaults

```bash
$ go env -w GOFLAGS='-mod=readonly -trimpath'
$ go env -w GOTOOLCHAIN=local           # don't auto-download in CI
```

### 3. Cross-compile

```bash
$ GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -trimpath -o app-linux-arm64 ./cmd/app
```

### 4. Tune microarchitecture

```bash
$ GOAMD64=v3 go build -o app-v3 ./cmd/app
```

### 5. Reset to defaults

```bash
$ go env -u GOPROXY
$ go env -u CGO_ENABLED
```

### 6. Inspect what CI is using

```yaml
- run: go env
- run: go version
```

Print at the top of CI for debuggability.

### 7. Cache mounts in Docker

```dockerfile
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go build -o app ./cmd/app
```

BuildKit caches `GOCACHE` and `GOMODCACHE` between builds.

## Anti-Patterns & Gotchas

**Setting `GOPROXY=off` system-wide.** Breaks any fresh module fetch. Use `GOTOOLCHAIN=local` for air-gapped CI instead, and pre-warm the cache.

**`GOSUMDB=off` globally.** Removes supply-chain protection. Scope to `GOPRIVATE`.

**Forgetting `go env -w` persists.** Sets via `-w` survive shell restarts; troubleshooting "why is `go build` slow today?" often reveals an old `GOPROXY=direct` left behind.

**Mixing shell env and `$GOENV` and not knowing which won.** Shell env wins per-invocation. `go env GOPROXY` always shows the effective value.

**Setting `CGO_ENABLED=1` for cross-compile without `CC=...`.** Build fails with cryptic linker errors. Either `CGO_ENABLED=0` or set the cross C toolchain.

**Tuning `GOMAXPROCS` for a benchmark to "more cores" without measuring.** May reduce throughput if work is memory-bound. Profile first.

**Confusing `GODEBUG` (runtime) and `GOEXPERIMENT` (compile-time).** `GODEBUG` is read at process start; `GOEXPERIMENT` is read by the compiler/linker.

**Setting `GOAMD64=v4` and shipping to customers.** Breaks on pre-Tigerlake CPUs. Default `v1` unless you control deployment.

**Relying on `go env GOROOT` for "where Go is".** A custom packager may set `GOROOT_FINAL`; `GOROOT` may differ from where the binary actually sits. For tooling, use `runtime.GOROOT()`.

**Forgetting `GOWORK=off` for release builds.** A leftover `go.work` may shadow upstream dep versions. Set `GOWORK=off` in CI for release artifacts.

**Setting `GOFLAGS=-count=1` permanently.** Disables test cache; CI takes 5× longer. Set per-test when needed.

**Hardcoding `GOPATH` for module-era projects.** Modules don't need a workspace; `GOPATH` only matters as the default location for `GOMODCACHE`/`GOBIN`.

## Performance Notes

- `go env`: <50 ms.
- `go env -w`: <100 ms (writes a few bytes to disk).
- `go env -json`: same as `go env`.
- Runtime cost of an env var: same as `os.Getenv` (~ns).

No build-time penalty from setting env vars; the toolchain's behavior changes accordingly but env-var reads are negligible.

## How Big Companies Use It

- **Google** mandates `GOFLAGS='-trimpath'` for release builds and internal proxy via `GOPROXY`: https://go.dev/doc/contribute.
- **Uber** uses `GOTOOLCHAIN=local` plus an internal Athens proxy; CI fails on unknown modules: https://eng.uber.com.
- **CockroachDB** ships a `Makefile` that pre-sets `GOFLAGS`, `CGO_ENABLED`, and `GOEXPERIMENT` for their cluster: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses `GOPRIVATE=*.hashicorp.com` for internal Terraform plugins: https://github.com/hashicorp/terraform.
- **Tailscale** uses `GOMEMLIMIT` on `tailscaled` to keep RSS bounded: https://tailscale.com/blog.
- **Cloudflare** sets `GOAMD64=v3` for their Workers Go runtime since their fleet is Tigerlake+ uniform: https://blog.cloudflare.com.
- **Discord's Go services** use `GOMAXPROCS` capped via `automaxprocs` plus `GODEBUG=madvdontneed=1` for memory release.

## Source Code References

Pinned to `go1.26`.

- `go env` command: [`src/cmd/go/internal/envcmd/env.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/envcmd/env.go).
- Variable defaults: [`src/cmd/go/internal/cfg/cfg.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/cfg/cfg.go).
- `$GOENV` reading: [`src/cmd/go/internal/cfg/cfg.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/cfg/cfg.go) (search `EnvFile`).
- `GOTOOLCHAIN`: [`src/cmd/go/internal/toolchain`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/toolchain).
- `GOEXPERIMENT`: [`src/internal/goexperiment`](https://github.com/golang/go/tree/release-branch.go1.26/src/internal/goexperiment).
- `GODEBUG` (runtime): [`src/runtime/runtime1.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/runtime/runtime1.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command go — Environment variables": https://pkg.go.dev/cmd/go#hdr-Environment_variables.
- "GOTOOLCHAIN" (1.21 release): https://go.dev/doc/go1.21#toolchain.
- "GOEXPERIMENT flags": https://pkg.go.dev/internal/goexperiment.
- "GODEBUG reference": https://pkg.go.dev/runtime#hdr-Environment_Variables.
- "Module configuration" (private modules): https://go.dev/ref/mod#private-modules.
- "Configure Go for an Internet-restricted environment": https://go.dev/wiki/GoGetPrivateGoEnv.

## Exercises / Self-Check

1. Print only the env vars whose names start with `GO`: `go env | grep -E '^GO'`. Which are set vs. defaulted?
2. Set `GOFLAGS='-mod=readonly'` via `go env -w`. Now run `go get`. What happens?
3. Configure `GOPROXY` and `GOPRIVATE` for a hypothetical internal company `acme.corp`. Show your full command.
4. Cross-compile to `linux/arm64` with cgo enabled. Which env vars do you set?
5. `GODEBUG=gctrace=1 ./app` produces output — what columns? Why isn't `GODEBUG` listed in `go env`?
