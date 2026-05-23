# `go mod` — Every Subcommand

## TL;DR

`go mod` is the umbrella for module-file management. Twelve subcommands divide cleanly into four groups: **lifecycle** (`init`, `tidy`, `download`, `verify`), **inspection** (`graph`, `why`, `edit -json`), **mutation** (`edit`, `vendor`), and **diagnostics** (`why`). The two you'll run every day are `init` (once) and `tidy` (after every dep change). The two that ship reproducibility are `verify` (re-hash the cache against `go.sum`) and `download` (populate the cache from `go.sum` without compiling). Everything `go mod` does also happens implicitly when `go build`/`go test` notices a missing dep or a stale require — `go mod` is just the explicit, scriptable form.

## Mental Model

```
                        ┌─────────────────────┐
                        │   go mod init       │ ── creates go.mod
                        └─────────────────────┘
                                  │
   ┌──────────────────────────────┼──────────────────────────────┐
   ▼                              ▼                              ▼
 LIFECYCLE                    INSPECTION                    MUTATION
 ──────────                   ───────────                   ────────
 go mod tidy                  go mod graph                  go mod edit
 go mod download              go mod why                    go mod vendor
 go mod verify                go mod edit -json
                              (also `go list -m`)
```

`go mod` operates on `go.mod`, `go.sum`, the module cache (`$GOMODCACHE`), and `vendor/`. It never compiles code — that's `go build`. It never runs tests — that's `go test`. Single responsibility per subcommand.

## Syntax & Basic Usage

```bash
$ go mod init github.com/me/proj
$ go mod tidy                       # add missing, drop unused
$ go mod tidy -compat=1.21          # keep working under older Go
$ go mod download                   # populate cache for all required mods
$ go mod download -json all         # JSON output of downloaded mods
$ go mod verify                     # re-hash cache against go.sum
$ go mod graph                      # dep edges, one per line
$ go mod why github.com/x/y         # show import chain
$ go mod why -m github.com/x/y      # module-level (not import-level)
$ go mod edit -require=github.com/x/y@v1.2.3
$ go mod edit -json                 # dump go.mod as JSON
$ go mod vendor                     # write vendor/ from current deps
$ go mod vendor -e                  # continue on errors
```

## Deep Dive

### `go mod init`

```bash
$ go mod init github.com/me/proj
go: creating new go.mod: module github.com/me/proj
```

Creates `go.mod` with just the module path and the current toolchain's `go` line. Optional second form, without a path, infers from `$GOPATH/src/<path>` or VCS metadata (rarely useful).

Run once per module. Running again refuses if `go.mod` exists.

### `go mod tidy`

The workhorse. Walks the import graph from all `main` packages **and all tests** (including ones gated by build tags it can see), then:

1. Adds any direct require missing from `go.mod`.
2. Removes any require not transitively needed.
3. Marks each require with `// indirect` (or removes the marker).
4. Adds missing `go.sum` entries.
5. Removes stale `go.sum` entries.
6. Downloads any module-version it had to consult.

Flags worth knowing:

```bash
$ go mod tidy                  # quiet
$ go mod tidy -v               # report which modules were added/removed
$ go mod tidy -e               # continue past errors (best-effort)
$ go mod tidy -compat=1.21     # preserve module graph readable to Go 1.21
$ go mod tidy -x               # print every command (download URLs etc.)
$ go mod tidy -diff            # 1.23+: report what *would* change without writing
```

`-compat=1.x` is for libraries that want to keep older Go users happy: it preserves enough indirect entries so `go 1.x` toolchains don't need to re-resolve the graph. Without it, dropping unneeded indirects can make older toolchains hit MVS errors.

CI pattern:

```yaml
- run: go mod tidy
- run: git diff --exit-code go.mod go.sum
```

Fail builds whose contributors forgot to tidy.

### `go mod download`

```bash
$ go mod download                # all modules in build list
$ go mod download example.com/x  # just one
$ go mod download -json all      # machine-readable
$ go mod download -x             # show URLs
```

Populates `$GOMODCACHE` from `$GOPROXY` and verifies against `go.sum`. Doesn't compile, doesn't touch `go.mod`/`go.sum`. Useful as a Docker layer:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app ./cmd/app
```

The `download` step caches independently of source code, so application changes don't re-download deps.

`go mod download -json` output:

```json
{
    "Path":    "github.com/google/uuid",
    "Version": "v1.6.0",
    "Info":    "/Users/me/go/pkg/mod/cache/download/github.com/google/uuid/@v/v1.6.0.info",
    "GoMod":   "/Users/me/go/pkg/mod/cache/download/github.com/google/uuid/@v/v1.6.0.mod",
    "Zip":     "/Users/me/go/pkg/mod/cache/download/github.com/google/uuid/@v/v1.6.0.zip",
    "Dir":     "/Users/me/go/pkg/mod/github.com/google/uuid@v1.6.0",
    "Sum":     "h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0="
}
```

### `go mod verify`

```bash
$ go mod verify
all modules verified
```

Re-hashes every module in the build list against `go.sum`. Catches:

- Corrupted module cache (disk error).
- Tampered-with proxy.
- A `replace` directive that hides a malicious payload.

Quick; usually <1 s. Run in CI before build to catch supply-chain weirdness.

### `go mod graph`

```bash
$ go mod graph
github.com/me/proj github.com/x/y@v1.2.3
github.com/me/proj github.com/cespare/xxhash/v2@v2.2.0
github.com/x/y@v1.2.3 github.com/cespare/xxhash/v2@v2.1.0
```

One edge per line: `requirer requiree@version`. Use to debug "where does this version come from":

```bash
$ go mod graph | grep xxhash
github.com/me/proj github.com/cespare/xxhash/v2@v2.2.0
github.com/x/y@v1.2.3 github.com/cespare/xxhash/v2@v2.1.0
```

(Selected version is the max, `v2.2.0`.)

### `go mod why`

```bash
$ go mod why github.com/cespare/xxhash/v2
# github.com/cespare/xxhash/v2
github.com/me/proj/internal/cache
github.com/cespare/xxhash/v2
```

Lines after `# pkg` show the import chain leading to it. With `-m`:

```bash
$ go mod why -m github.com/cespare/xxhash/v2
# github.com/cespare/xxhash/v2
(main module does not need module github.com/cespare/xxhash/v2)
```

If a module is no longer needed, `go mod tidy` will remove it. `go mod why` is the diagnostic.

### `go mod edit`

Programmatic edits to `go.mod`. Useful in scripts that don't want to risk hand-parsing:

```bash
$ go mod edit -require=github.com/x/y@v1.2.3
$ go mod edit -droprequire=github.com/old/dep
$ go mod edit -replace=github.com/foo/bar=../local/bar
$ go mod edit -replace=github.com/foo/bar=github.com/me/bar-fork@v1.0.0
$ go mod edit -dropreplace=github.com/foo/bar
$ go mod edit -exclude=github.com/x/y@v1.4.0
$ go mod edit -dropexclude=github.com/x/y@v1.4.0
$ go mod edit -retract=v1.0.0
$ go mod edit -dropretract=v1.0.0
$ go mod edit -go=1.26
$ go mod edit -toolchain=go1.26.1
$ go mod edit -godebug=httpmuxgo121=1
$ go mod edit -tool=golang.org/x/tools/cmd/stringer    # 1.24+
$ go mod edit -droptool=golang.org/x/tools/cmd/stringer
$ go mod edit -json                                    # dump as JSON
$ go mod edit -fmt                                     # canonicalize formatting
$ go mod edit -print                                   # write to stdout instead of file
```

The `-json` form is the right way for tools to *read* `go.mod` (don't regex it):

```json
{
    "Module": { "Path": "github.com/me/proj" },
    "Go":     "1.26",
    "Require": [
        { "Path": "github.com/google/uuid", "Version": "v1.6.0" }
    ]
}
```

### `go mod vendor`

```bash
$ go mod vendor
```

Walks the import graph, copies every package's source files (and `LICENSE`/`PATENTS`/`NOTICE`/`COPYING`-prefixed files) into `vendor/`, then writes `vendor/modules.txt` recording the module-version of each subdir.

After vendoring, builds default to `-mod=vendor` (use `vendor/` exclusively, ignore `$GOMODCACHE`). The full story is in `06-packages-modules/07-vendoring.md`.

```bash
$ go mod vendor -e         # don't fail on missing deps
$ go mod vendor -v         # report which deps were copied
```

### `go mod` interaction with `go.work`

In a workspace (`06-packages-modules/05-workspaces.md`), most `go mod` commands operate on the *workspace's effective module graph*. `go mod tidy` runs per-module — you must `cd` into each.

The exception is the workspace-aware command set: `go work init`, `go work use`, `go work sync`, `go work vendor` (1.22+). See `09-tooling/04-go-work.md`.

### `GOFLAGS` and `go mod`

```bash
$ export GOFLAGS="-mod=readonly"
$ go build ./...   # fails if it would edit go.mod
```

A safety net in CI: prevents accidental `go.mod` mutations during a build.

### `replace`, `exclude`, `retract`

Briefly (full coverage in `06-packages-modules/04-mod-directives.md`):

```
replace github.com/old => github.com/new v1.0.0    // remap
exclude github.com/x/y v1.4.0                       // refuse this version
retract v1.0.1                                       // mark this version of *my* module as bad
```

`go mod edit -retract=v1.0.1` adds the retract line; consumers see a warning when `go get`-ing v1.0.1.

### Hidden: `go mod verify` and `GOFLAGS=-insecure`

If `GOPRIVATE` includes a host, `GOSUMDB=off` is implied for that host. `go mod verify` still hashes against `go.sum` but doesn't consult the public sumdb. Disabling sumdb globally is a foot-gun (`GOSUMDB=off` for everything) — scope to `GOPRIVATE`.

### `go.sum` lifecycle

`go.sum` grows when new versions enter the graph (even versions *not selected* by MVS — Go records hashes for *every version it consulted*). `go mod tidy` prunes versions not in the current build list.

If `go.sum` has stale entries because contributors forgot to `tidy`, CI catches it via `git diff --exit-code go.sum`.

## Standard Library Hooks

- `runtime/debug.ReadBuildInfo` — read embedded module/dep info.
- `debug/buildinfo` — read it from any binary on disk.
- `golang.org/x/mod/modfile` — programmatic `go.mod` parsing.
- `golang.org/x/mod/module` — version + path utilities.
- `golang.org/x/mod/sumdb` — verify against the checksum DB.
- `golang.org/x/mod/zip` — read/write the module zip format.
- `golang.org/x/tools/go/packages` — analysis with full module context (calls `go list` under the hood).

## Real-World Patterns

### 1. Fresh module + add deps

```bash
$ mkdir myapp && cd myapp
$ go mod init github.com/me/myapp
$ cat > main.go <<EOF
package main
import "github.com/google/uuid"
func main() { println(uuid.NewString()) }
EOF
$ go mod tidy
$ go build .
```

### 2. CI tidy guard

```yaml
- run: go mod tidy
- run: git diff --exit-code -- go.mod go.sum
```

If a developer forgot to tidy, CI fails with a useful diff.

### 3. Reproducible Docker build

```dockerfile
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -o /out/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

Two cache layers: `go mod download` warms the module cache; the app build reuses it.

### 4. Bulk upgrade

```bash
$ go get -u ./...       # upgrade all direct + indirect to latest
$ go mod tidy
$ go test ./...
```

### 5. Replace for fork during review

```bash
$ go mod edit -replace=github.com/upstream/lib=github.com/me/lib-fork@v0.0.0-20250101120000-abc123def456
$ go mod tidy
```

When the upstream PR merges, drop the replace.

### 6. Discover where a version came from

```bash
$ go mod graph | grep '@v1.5.0'
$ go mod why github.com/x/y
```

### 7. Mass `go mod tidy` across multi-module repo

```bash
$ find . -name go.mod -execdir go mod tidy \;
```

(Or use `go work sync` if you have a workspace.)

## Anti-Patterns & Gotchas

**Editing `go.sum` by hand.** It's a generated artifact. `go mod tidy` will overwrite your edits.

**Running `go mod tidy` only in CI.** It edits `go.mod`/`go.sum`. Local devs need to run it before pushing. Make it part of `pre-commit` hooks or `Makefile`.

**`go mod tidy` with a network outage.** Tidy needs to consult the proxy for any version it hasn't seen. If you're offline and the cache lacks a version, it fails. Set `GOPROXY=off` in air-gapped CI and rely on a pre-warmed cache.

**Vendoring without `go mod tidy` first.** `go mod vendor` honors current `go.mod`. If `go.mod` is stale, `vendor/` is stale.

**Mixing `replace ../local` and CI.** Filesystem `replace` breaks on any machine without that path. Use `go.work` for local dev; reserve `replace` for remote forks.

**Forgetting `-compat=1.x` on libraries supporting older Go.** Tidying with Go 1.26 may remove indirects that Go 1.21 still needs to resolve the graph. Set `-compat` to your declared min version.

**`go mod vendor` *and* `go mod download` *and* `vendor/` checked into Git.** Pick one model: vendored or proxy-driven. Both is just storage churn.

**Disabling sumdb globally (`GOSUMDB=off`).** Removes supply-chain protection. Scope to `GOPRIVATE`.

**`go mod why` on a package vs. a module.** Default `go mod why` is package-level; for module-level use `-m`. Easy to misread output.

**Trusting `go mod tidy -e` in CI.** `-e` keeps going on errors; you may end up with an "almost tidy" `go.mod` that hides a real problem. Use only for diagnostics, never to ship.

**Running `go mod download all` in a tiny container.** All transitive deps land in the image; the image bloats by 100s of MiB. Multi-stage Docker builds (use `download` only in the builder stage) fix this.

## Performance Notes

- `go mod init`: <100 ms.
- `go mod tidy` warm: <1 s for small (~20 deps) projects; 1–5 s for medium (~100 deps); 5–30 s for large (~500+ deps).
- `go mod tidy` cold (no module cache): dominated by network; 10–60 s.
- `go mod download` warm: <1 s. Cold: dominated by network/proxy.
- `go mod verify`: ~1 ms per dep.
- `go mod graph`: ~100 ms.
- `go mod vendor`: 1–10 s depending on dep size.
- `go mod why`: <1 s.

Cache hits matter. Persist `$GOMODCACHE` (default `$GOPATH/pkg/mod`) across CI runs.

## How Big Companies Use It

- **Google** runs `go mod tidy` as a pre-submit gate on every `golang/*` repo: https://go.googlesource.com.
- **Kubernetes** uses a workspace-style multi-module repo and runs `go mod tidy` per-module with a custom verifier: https://github.com/kubernetes/kubernetes/blob/master/hack/update-vendor.sh.
- **Uber** uses an internal Athens proxy and pins `GOPROXY` to it; `go mod download` is the first step in every build: https://github.com/uber-go/guide.
- **HashiCorp** vendors every dep (`go mod vendor`) for reproducibility and audit; CI fails on missing vendor entries: https://github.com/hashicorp/terraform.
- **CockroachDB** ships `go.mod` + `go.sum` + Renovate; `go mod tidy` is enforced via CI: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses `replace` for in-flight upstream patches; their `go.mod` notes which `replace` blocks unblock when upstream merges: https://github.com/tailscale/tailscale.
- **Discord** mirrors public modules into an internal proxy via `athens` to harden against deletions of upstream packages.

## Source Code References

Pinned to `go1.26`.

- `go mod` umbrella: [`src/cmd/go/internal/modcmd`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/modcmd).
- `go mod init`: [`src/cmd/go/internal/modcmd/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/init.go).
- `go mod tidy`: [`src/cmd/go/internal/modcmd/tidy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/tidy.go).
- `go mod download`: [`src/cmd/go/internal/modcmd/download.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/download.go).
- `go mod verify`: [`src/cmd/go/internal/modcmd/verify.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/verify.go).
- `go mod graph`: [`src/cmd/go/internal/modcmd/graph.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/graph.go).
- `go mod why`: [`src/cmd/go/internal/modcmd/why.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/why.go).
- `go mod edit`: [`src/cmd/go/internal/modcmd/edit.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/edit.go).
- `go mod vendor`: [`src/cmd/go/internal/modcmd/vendor.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/vendor.go).
- modfile parser: [`golang.org/x/mod/modfile`](https://github.com/golang/mod/tree/master/modfile).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go Modules Reference — `go mod`": https://go.dev/ref/mod#go-mod-cmd.
- "Using `go mod tidy`" (Jay Conrod): https://go.dev/blog/using-go-modules.
- Russ Cox, "vgo design series": https://research.swtch.com/vgo.
- "Module mirror and checksum database" (Filippo Valsorda): https://go.dev/blog/module-mirror-launch.
- "Vendor directories" (Go docs): https://go.dev/ref/mod#vendoring.

## Exercises / Self-Check

1. After `go get github.com/x/y@v1.5.0`, run `go mod graph | grep x/y`. Why might multiple lines appear?
2. Run `go mod why -m github.com/some-transitive-dep`. What does the output mean if it says "main module does not need module"?
3. Vendor a project (`go mod vendor`) then delete `$GOMODCACHE` and build. Does it still work? Why?
4. Edit `go.mod` to add a `replace` pointing at a fork. Run `go mod tidy`. What does it do to `go.sum`?
5. Use `go mod edit -json` to write a script that prints all direct deps and their versions.
