# Go Modules — go.mod, go.sum, MVS

## TL;DR

A **module** is a collection of related Go packages versioned as a unit, identified by a **module path** (its import prefix) and recorded in a **`go.mod`** file at its root. The Go tool resolves dependencies using **Minimum Version Selection (MVS)** — for each module required transitively, it picks the *highest* version that satisfies the *minimum* requirements; the result is deterministic and reproducible. **`go.sum`** stores cryptographic hashes of every dependency for tamper detection. The single biggest gotcha: **MVS does not pick "latest"; it picks the highest version anyone in the dep tree requires**. A direct require of `v1.2.0` plus a transitive require of `v1.5.0` results in `v1.5.0`. Upgrades are explicit via `go get` or `go mod tidy`.

## Mental Model

```
   project root
   ├── go.mod                          ← declares module path, deps, toolchain
   │     module github.com/me/proj
   │     go 1.26
   │     require (
   │         github.com/x/y v1.2.3
   │         github.com/x/z v2.0.0+incompatible
   │     )
   │
   ├── go.sum                          ← cryptographic hashes, never edit by hand
   │     github.com/x/y v1.2.3 h1:abc...
   │     github.com/x/y v1.2.3/go.mod h1:def...
   │
   ├── main.go
   └── ...

   The Go tool reads go.mod, resolves the dep graph via MVS,
   downloads zips from GOPROXY, verifies against go.sum,
   extracts into $GOMODCACHE, then compiles.
```

A module is the unit of versioning; a *package* is the unit of compilation. Inside a module, packages share the module's version. Outside the module, importers see the module's selected version.

## Syntax & Basic Usage

Initialize a module:

```bash
$ go mod init github.com/me/proj
# creates go.mod
```

Add dependencies (implicitly via `go build`/`go test`, or explicitly):

```bash
$ go get github.com/x/y@v1.2.3
$ go get github.com/x/y@latest
$ go get github.com/x/y@master   # specific branch
```

Tidy and verify:

```bash
$ go mod tidy            # remove unused, add missing
$ go mod download        # populate cache without building
$ go mod verify          # re-check go.sum against cache
```

Minimal `go.mod`:

```
module github.com/me/proj

go 1.26

require (
    github.com/google/uuid v1.6.0
    github.com/stretchr/testify v1.9.0
)
```

## Deep Dive

### `go.mod` structure

```
module github.com/me/proj           // module path

go 1.26                              // language version (and minimum Go for this module)

toolchain go1.26.1                   // optional: required toolchain version (since 1.21)

require (
    github.com/google/uuid v1.6.0           // direct dep
    github.com/jackc/pgx/v5 v5.5.0          // direct dep
    github.com/cespare/xxhash/v2 v2.2.0     // indirect
        // indirect
)

require github.com/cespare/xxhash/v2 v2.2.0 // indirect

replace github.com/foo/bar => ../local/bar    // local override
replace github.com/foo/bar => github.com/me/bar-fork v1.0.0

exclude github.com/x/y v1.4.0                // refuse this version

retract v1.0.0                                // declare a retraction (your own module)

godebug (
    default=go1.26
    httpmuxgo121=1
)
```

### Module path

The module path:

1. Is a string that **looks like an import path** — typically the public location.
2. Determines the *import prefix* for all packages in the module.
3. Must match the version-suffix rule for major version ≥2 (e.g., `github.com/me/proj/v2`).
4. Is lowercased for the `domain/owner/repo` prefix (proxy convention).

Public modules typically use the repo's location as the module path. Internal/private modules can use any path (`my.company/foo/bar`) as long as the resolver can find sources (via `GOPROXY`, `GOPRIVATE`, vanity tags, etc.).

### The `go` directive

```
go 1.26
```

Declares:
1. The **language version** the module is written against (1.26 features available).
2. The **minimum Go version** required to build this module.
3. **Compatibility defaults** — some `GODEBUG` settings flip based on this value.

Since 1.21 the `go` line is *strictly enforced*: a Go 1.20 toolchain cannot compile a module with `go 1.21`. Before 1.21 it was advisory.

### The `toolchain` directive (since 1.21)

```
toolchain go1.26.1
```

Asks `go` to switch to this exact toolchain version if the current one is older. The Go command downloads it on demand (managed via `GOTOOLCHAIN`).

```
GOTOOLCHAIN=auto     # default: respect go.mod/toolchain
GOTOOLCHAIN=local    # never download
GOTOOLCHAIN=go1.26.1 # force
```

This decoupled the language version from the binary version: a module can require Go 1.26's language features even if your installed Go is 1.25, as long as `GOTOOLCHAIN=auto` and the network is reachable.

### Direct vs indirect requires

```
require (
    github.com/google/uuid v1.6.0           // direct: imported by your code
    github.com/cespare/xxhash/v2 v2.2.0 // indirect
                                            // not directly imported; only deps need it
)
```

`// indirect` means "this dependency is included only because some other dep needs it". `go mod tidy` adds/removes the marker automatically.

Why track indirects? So `go.mod` records the entire dep tree's *minimum* versions — needed for MVS to give a reproducible build without re-walking every transitive `go.mod`.

### `go.sum`

```
github.com/google/uuid v1.6.0 h1:abc...=
github.com/google/uuid v1.6.0/go.mod h1:xyz...=
```

Two entries per module-version:
- The first is the hash of the module's source zip.
- The second is the hash of just the `go.mod` file (for cheap verification without downloading the full source).

Hashes are computed via the `Hash1` algorithm: SHA-256 of a canonical file listing. Implemented in [`golang.org/x/mod/sumdb/dirhash`](https://github.com/golang/mod/blob/master/sumdb/dirhash/hash.go).

When `go build` downloads a module, it hashes the result and verifies against `go.sum`. Mismatch → fatal error:

```
verifying module: checksum mismatch
        downloaded: h1:...
        go.sum:     h1:...
```

Indicates tampering, mirror corruption, or a rewritten history. Don't bypass without understanding.

### Checksum database (`sum.golang.org`)

When a module-version is *not* in `go.sum` (i.e., new add), the Go tool consults `https://sum.golang.org` to fetch the canonical hash and only then writes to `go.sum`. The checksum DB is an append-only Merkle log; tampering requires colluding with Google.

Disable per `GOSUMDB=off` (do this only for private modules) or set `GONOSUMCHECK=1` (legacy; modern is `GONOSUMDB` and `GOPRIVATE`).

### MVS — Minimum Version Selection

Russ Cox's algorithm. For each module in the dependency graph:

```
selected[m] = max( required[m] ∪ {v : some other module requires m@v} )
```

Steps:

1. Read your module's `go.mod` — direct requires.
2. For each required `m@v`, read `m@v`'s `go.mod` — its requires become candidates.
3. Repeat transitively.
4. For each module ever requested, pick the **max** version anyone in the tree requested.
5. The set of `(module, version)` pairs is your build list.

Critical properties:

- **Deterministic**: given the same input `go.mod`s, MVS always produces the same build list.
- **Minimal**: you only get versions someone explicitly requested. No "latest" surprise.
- **Stable**: adding a new dep that needs `m@v1.2.0` won't bump your `m` if it's already at `v1.5.0`.

The "latest" idea isn't part of MVS. `go get m@latest` resolves "latest" *once*, writes it to `go.mod`, then MVS uses the recorded number.

Implementation: [`src/cmd/go/internal/mvs/mvs.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/mvs/mvs.go).

### Why not SAT/CSP solvers?

Most package managers (Cargo, npm, pip) use SAT solvers to find "compatible" versions given constraints. Pros: more flexibility. Cons: nondeterministic, slow, sensitive to constraint formulation. Go modules trade flexibility for predictability: every build is reproducible, and resolution is linear in the dep graph size.

Russ Cox's argument is at https://research.swtch.com/vgo-mvs.

### `+incompatible`

```
require github.com/x/y v2.0.0+incompatible
```

A pre-modules major-version-2 module that doesn't follow the `/v2` import-path rule (see `06-packages-modules/06-versioning-and-semver.md`). The `+incompatible` tag says "I'm using this old-style v2; I know it doesn't fit the rule".

Rare in modern code. Most v2+ libraries have migrated to `module github.com/x/y/v2`.

### `go.mod` editing

`go mod edit` programmatically modifies `go.mod`:

```bash
$ go mod edit -require=github.com/x/y@v1.2.3
$ go mod edit -require=github.com/x/y@v1.2.3 -droprequire=github.com/old/dep
$ go mod edit -replace=github.com/foo/bar=../local/bar
$ go mod edit -go=1.26
$ go mod edit -toolchain=go1.26.1
$ go mod edit -json     # show as JSON (for scripts)
```

For one-off edits, hand-edit `go.mod` then run `go mod tidy`.

### `go mod tidy`

Walks the import graph from your `main` packages and all tests, then:

1. Adds any missing direct requires.
2. Removes any required modules that nothing imports.
3. Marks each require `// indirect` or not.
4. Adds missing `go.sum` entries.
5. Removes stale `go.sum` entries.

Run after every dep change. CI typically runs `go mod tidy && git diff --exit-code` to fail builds whose `go.mod` is out of sync.

Since 1.17, `go mod tidy -compat=1.17` (or `-compat=<version>`) preserves backward compatibility with the named Go version.

### `go get` semantics

```bash
$ go get github.com/x/y                  # add or upgrade to latest matching
$ go get github.com/x/y@v1.2.3           # specific version
$ go get github.com/x/y@latest           # latest tagged version
$ go get github.com/x/y@upgrade          # like @latest but only upgrade
$ go get github.com/x/y@master           # specific branch
$ go get github.com/x/y@a1b2c3d          # specific commit
$ go get github.com/x/y@none             # remove (and dependencies)
$ go get -u                              # upgrade everything in current module
$ go get -u=patch                        # upgrade patch versions only
```

`go get` in module mode (since 1.16) modifies `go.mod`. Pre-1.16 it could also build/install; that's now `go install` only.

### Vendor mode

If `vendor/` exists and contains a valid `modules.txt`:

```bash
$ go build -mod=vendor      # use vendor/ exclusively
$ go build -mod=mod         # ignore vendor/
$ go build -mod=readonly    # error on go.mod changes
```

Defaults vary; since `go.mod` `go ≥ 1.14` and `vendor/` exists, `-mod=vendor` is default. See `06-packages-modules/07-vendoring.md`.

### `GOPROXY`, `GOSUMDB`, `GOPRIVATE`

```
GOPROXY=https://proxy.golang.org,direct    # default
GOSUMDB=sum.golang.org                      # default
GOPRIVATE=*.corp.example.com,internal.lan   # bypass proxy + sumdb for these
```

`GOPRIVATE` is the easy switch: any module path matching its globs skips the public proxy and sumdb. Replaces the older `GONOSUMCHECK`. See `06-packages-modules/08-private-modules-and-goproxy.md`.

### Reading from `runtime/debug.ReadBuildInfo`

The linker embeds `go.mod` info into binaries:

```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	bi, ok := debug.ReadBuildInfo()
	if !ok {
		return
	}
	fmt.Println("module:", bi.Main.Path, bi.Main.Version)
	for _, d := range bi.Deps {
		fmt.Println("dep:", d.Path, d.Version)
	}
	for _, s := range bi.Settings {
		fmt.Println("setting:", s.Key, "=", s.Value)
	}
}
```

`go version -m ./bin` reads the same data without running the binary. Useful for vulnerability scanning (`govulncheck`) and license auditing.

### Pseudo-versions

When a require points at a commit (no tag), `go.mod` records a synthetic version:

```
require github.com/x/y v0.0.0-20240115120000-abc123def456
```

Format: `v0.0.0-YYYYMMDDhhmmss-shortcommit`. Sortable; the proxy resolves them via `https://proxy/<module>/@v/<pseudo>.info`. Specified at https://go.dev/ref/mod#pseudo-versions.

### `go.work` interaction

In a workspace (see `06-packages-modules/05-workspaces.md`), `go.work` overrides `go.mod` for selected modules — useful for editing multiple modules together. The workspace `use` directive lists local module dirs.

### `go list -m`

Inspect the build list:

```bash
$ go list -m all                    # everything MVS selected
$ go list -m -u all                 # mark available upgrades
$ go list -m -json all              # full JSON
$ go list -m -versions github.com/x/y   # available versions
```

`go list -m -json all` is the input format many tools want (security scanners, license auditors).

### `go mod graph`

```bash
$ go mod graph
github.com/me/proj github.com/x/y@v1.2.3
github.com/me/proj github.com/cespare/xxhash/v2@v2.2.0
github.com/x/y@v1.2.3 github.com/cespare/xxhash/v2@v2.1.0
```

One edge per line: `requirer requiree@version`. Use for debugging "why is this version selected".

### `go mod why`

```bash
$ go mod why github.com/cespare/xxhash/v2
# github.com/cespare/xxhash/v2
github.com/me/proj/internal/cache
github.com/cespare/xxhash/v2
```

Shows the import path that leads to a require. If a dep can be removed, this reports `(main module does not need package ...)`.

## Standard Library Hooks

- `go mod init/tidy/download/verify/why/graph/edit`.
- `go get`, `go install`.
- `go list -m`.
- `runtime/debug.ReadBuildInfo` — modules at runtime.
- `debug/buildinfo` — read modules from a binary on disk.
- `golang.org/x/mod` (semver, modfile, module, sumdb) — programmatic module manipulation.
- `golang.org/x/tools/go/packages` — analysis with full module context.
- `govulncheck` — vulnerability scanning over deps.

## Real-World Patterns

### 1. Initialize and add a dep

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

`tidy` adds `github.com/google/uuid` to `go.mod` and writes hashes to `go.sum`.

### 2. Pin an exact version

```bash
$ go get github.com/lib/pq@v1.10.9
$ git diff go.mod
+require github.com/lib/pq v1.10.9
```

A specific tagged version. Commit `go.mod` + `go.sum`.

### 3. Local-replace for development

```
replace github.com/me/util => ../util
```

Lets `go build` use a sibling checkout instead of the proxy-fetched version. Common for multi-module monorepos; use `go.work` instead in 1.18+.

### 4. CI tidy check

```yaml
- name: Verify tidy
  run: |
    go mod tidy
    git diff --exit-code go.mod go.sum
```

If a contributor forgot to tidy, CI fails. Saves "why is this require here?" debates.

### 5. Bump everything

```bash
$ go get -u ./...
$ go mod tidy
$ go test ./...
```

Tries all upgradable deps to latest minor/patch. Pair with tests; commit when green.

## Anti-Patterns & Gotchas

**Editing `go.sum` by hand.** It's machine-generated. `go mod tidy` will overwrite your edits.

**Committing `go.sum` but ignoring `go.mod`.** They're a pair. Always commit together.

**Running `go get -u` and committing without testing.** Major-version-locked upgrades sometimes break APIs. Always test after upgrade.

**`replace` directives that point at filesystem paths in `go.mod`.** Breaks CI on every machine that doesn't share the path. Use `go.work` for local dev; `replace` for forks pointing at remote modules.

**Pinning a specific commit when a tag is available.** Pseudo-versions are uglier and tools (Renovate, Dependabot) may not upgrade them. Prefer tagged versions.

**`GOPROXY=direct` in a corporate network.** Every fetch hits public VCS hosts; slow and brittle. Set up an internal proxy (Athens, JFrog) or use `proxy.golang.org`.

**Disabling sumdb (`GOSUMDB=off`) globally.** Removes supply-chain protection. Disable only for `GOPRIVATE` paths.

**Forgetting that `go.mod`'s `go` directive is enforced (since 1.21).** A toolchain older than declared can't build the module unless `GOTOOLCHAIN=auto` lets it fetch a newer one.

**Using `+incompatible` for new code.** Only legitimate when wrapping an old library; new modules should use proper `/v2` paths.

**Relying on `go get` to install binaries.** Since 1.16, `go get` is module-management; `go install foo@latest` installs binaries.

**Committing the module cache or `vendor/` without modules.txt.** `vendor/` only works with the modules.txt manifest. Use `go mod vendor` to generate.

**Treating `// indirect` as "I can remove this".** Indirect deps are still required at build time. Removing them breaks builds.

## Performance Notes

- `go mod tidy` walking 200 modules: ~1–5 s on a warm cache; ~30 s on cold.
- MVS resolution: O(deps × revisions); usually ms.
- Module download (per dep): ~50–500 ms over proxy.
- `go.sum` verification: ~ms per dep.
- `runtime/debug.ReadBuildInfo`: <µs (reads from binary metadata).
- `go list -m all`: ~100 ms for medium projects.
- `go mod graph`: similar.

A clean build (no module cache) of a 200-dep project: ~10–30 s for downloads alone; on a warm cache, sub-second.

## How Big Companies Use It

- **Kubernetes** uses a complex multi-module repo with hundreds of indirect deps; their `go.mod` is 100+ lines: https://github.com/kubernetes/kubernetes/blob/master/go.mod.
- **The Go team** at Google uses an internal proxy mirror plus aggressive `GOPRIVATE` for first-party code: https://go.dev/ref/mod#environment-variables.
- **Uber** documents their MVS-driven dep upgrade flow: https://github.com/uber-go/guide.
- **CockroachDB** uses `go mod tidy` checks in CI plus Renovate for automated upgrades: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses `replace` for development forks of upstream libs, then unblocks once upstream merges: https://github.com/tailscale/tailscale.
- **Discord's Go services** use Athens as an internal proxy for reproducibility across staging/prod.
- **HashiCorp** ships `go.sum` with every release and signs releases via `cosign` to detect tampering across the supply chain.

## Source Code References

Pinned to `go1.26`.

- `go.mod` parser: [`golang.org/x/mod/modfile`](https://github.com/golang/mod/tree/master/modfile) (vendored into `src/cmd/go/internal/modfile`).
- MVS algorithm: [`src/cmd/go/internal/mvs/mvs.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/mvs/mvs.go).
- Module loader: [`src/cmd/go/internal/modload`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/modload).
- Module fetcher: [`src/cmd/go/internal/modfetch`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/modfetch).
- `go.sum` hash algorithm: [`golang.org/x/mod/sumdb/dirhash`](https://github.com/golang/mod/blob/master/sumdb/dirhash/hash.go).
- Proxy protocol implementation: [`src/cmd/go/internal/modfetch/proxy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/proxy.go).
- `debug/buildinfo`: [`src/debug/buildinfo/buildinfo.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/debug/buildinfo/buildinfo.go).
- Toolchain switching: [`src/cmd/go/internal/toolchain`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/toolchain).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go Modules Reference": https://go.dev/ref/mod.
- Russ Cox, "Versioned Go (vgo)" — the original design series: https://research.swtch.com/vgo.
- Russ Cox, "Minimal Version Selection": https://research.swtch.com/vgo-mvs.
- Bryan C. Mills, "Go modules at scale" (talks): https://github.com/bcmills.
- Jay Conrod, "Modules: dependency management for the future" (Go blog): https://go.dev/blog/using-go-modules.
- "Go 1.21 toolchain proposal" (Russ Cox): https://go.googlesource.com/proposal/+/refs/heads/master/design/57001-gotoolchain.md.
- "Checksum database design": https://research.swtch.com/tlog.
- govulncheck: https://go.dev/security/vuln.

## Exercises / Self-Check

1. You have direct `require github.com/x/y v1.2.0` and a transitive dep that needs `v1.5.0`. What version does MVS pick? Why?
2. Write a minimal `go.mod` for a module `github.com/me/proj` requiring Go 1.26 and a single direct dep `github.com/google/uuid v1.6.0`. Include the toolchain directive.
3. The proxy returns a 200 for `github.com/x/y@v1.0.0` but the hash differs from `go.sum`. What does `go build` report? What are three plausible causes?
4. Run `go mod why github.com/some/transitive-dep` on a real project. Interpret the output.
5. Use `runtime/debug.ReadBuildInfo` to write a tiny endpoint that returns the running binary's `go.mod` deps as JSON.
