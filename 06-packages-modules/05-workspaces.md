# Go Workspaces — `go.work` (since 1.18)

## TL;DR

A **workspace** is a `go.work` file that lists multiple local modules to build together, overriding their normal versions with the local filesystem checkouts. It replaces the old "edit every `go.mod` with `replace` directives" workflow for multi-module development. `go.work` is **machine/developer-local** by convention — keep it out of version control or generate it on demand. The Go tool detects `go.work` in any ancestor directory and uses it automatically. The single biggest gotcha: **`go.work` is only honored by *commands*, not by published modules** — your `go.mod` is what consumers see, so a workspace fixing a build locally doesn't fix anything for users.

## Mental Model

```
   monorepo/
   ├── go.work                          ← workspace file (often gitignored)
   ├── api/
   │   ├── go.mod                       module github.com/me/api
   │   └── api.go
   ├── server/
   │   ├── go.mod                       module github.com/me/server
   │   └── server.go     (imports github.com/me/api)
   └── client/
       ├── go.mod                       module github.com/me/client
       └── client.go     (imports github.com/me/api)

   go.work:
       go 1.26
       use (
           ./api
           ./server
           ./client
       )

   With go.work in place:
     - `go build` in server/ uses ../api source, NOT proxy-fetched api.
     - Same for client/.
     - Changes in api/ are visible immediately, no version bumps.
```

`go.work` overlays selected modules at filesystem locations. The Go tool builds against the union — it sees `api`, `server`, `client` all as local, with cross-references resolved to local paths.

## Syntax & Basic Usage

```bash
$ go work init ./api ./server ./client
# generates go.work
```

```
go 1.26

use (
    ./api
    ./server
    ./client
)
```

Other commands:

```bash
$ go work use ./newmodule      # add a module
$ go work use -r .              # add every go.mod under .
$ go work edit -dropuse=./old   # remove
$ go work sync                  # propagate workspace's effective versions to each go.mod
$ go work edit -go=1.26         # bump go directive
```

When `go.work` is present, every `go` command uses it. Override with `-workfile=off`:

```bash
$ go build -workfile=off ./...   # ignore go.work for this invocation
```

Or set `GOWORK=off`:

```bash
$ GOWORK=off go build ./...
```

## Deep Dive

### Why workspaces exist

Before Go 1.18, multi-module local development required:

```
// server/go.mod
require github.com/me/api v0.0.0
replace github.com/me/api => ../api
```

Each developer adds this `replace`. Committed to the repo, it breaks for anyone who's not at that path. Not committed, every developer adds it manually. CI strips it before publish. Painful.

`go.work` solves this with a file *outside* `go.mod` whose authority is local-only. Drop it in your dev tree; commit it to `.gitignore` (or commit it and document the layout).

### The `go.work` file

```
go 1.26
toolchain go1.26.1

use (
    ./moduleA
    ./moduleB
    ../sibling/moduleC      // relative paths fine
    /absolute/path/moduleD
)

replace github.com/foo/bar => github.com/me/fork v1.2.3
```

Grammar:

- `go <version>`: minimum workspace version. Must be ≥ each `use`'d module's `go` directive (with `auto` toolchain promotion).
- `toolchain go<version>`: workspace toolchain pin.
- `use <path>`: a directory containing a `go.mod`. The path is relative to `go.work`'s directory.
- `replace`: same as in `go.mod`; applies across the workspace, overriding any module-level replace.

### `use` semantics

Each `use ./module` brings that module's source into the workspace at the listed path. Imports across `use`'d modules resolve locally:

- Module `github.com/me/api` at `./api`.
- Module `github.com/me/server` at `./server`.
- `server` imports `github.com/me/api/types`.
- Resolution: `./api/types` (local).

What `use` ignores:

- The module's `require` directives for the *other use'd modules*. Versions are irrelevant; sources are used as-is.
- The module's `replace`/`exclude`/`retract` directives, with one caveat: `replace`s pointing at *modules outside the workspace* are still applied within that module's source. So if `./api`'s `go.mod` says `replace github.com/x/y => ./fork`, that holds when building `./api`.

### `go work sync`

```bash
$ go work sync
```

Propagates the workspace's effective MVS resolution to each `use`'d module's `go.mod`. After a workspace dev cycle, when you're ready to publish, `go work sync` writes the actual selected versions back into each `go.mod`, ensuring published modules will reproduce the build.

Useful when the workspace pulls in newer indirect deps than any individual `go.mod` knew about.

### `GOWORK`

```
GOWORK=auto     # default: find go.work in ancestor dirs
GOWORK=off      # disable workspace mode entirely
GOWORK=/path/to/go.work  # explicit path
```

`GOWORK=off` is the safety valve when you want module mode regardless of `go.work`. CI commonly sets `GOWORK=off` to verify that each module builds standalone.

### `go.work.sum`

When a workspace adds a new external dep (one not in any `use`'d module's `go.sum`), the Go tool writes its hash to `go.work.sum`. This is to `go.work` what `go.sum` is to `go.mod`. Commit it if you commit `go.work`.

### CI: build with and without workspace

```yaml
- name: Build with workspace
  run: go build ./...

- name: Build modules standalone (without workspace)
  run: |
    cd api && GOWORK=off go build ./...
    cd ../server && GOWORK=off go build ./...
    cd ../client && GOWORK=off go build ./...
```

The first catches workspace-only bugs. The second catches "I forgot to update `api`'s `go.mod` before publishing".

### Multi-module repositories

Modern Go monorepos typically look like:

```
monorepo/
├── go.work                  (committed or gitignored)
├── apps/
│   ├── api/    (go.mod)
│   └── web/    (go.mod)
├── libs/
│   ├── auth/   (go.mod)
│   ├── cache/  (go.mod)
│   └── log/    (go.mod)
└── tools/
    └── codegen/  (go.mod)
```

Each app/lib is a separate module — independently versioned, separately released. The workspace makes local dev tractable.

### Single-module monorepos

Alternative: one `go.mod` at the root, internal packages as directories. No workspace needed. Trade-off:

- Single-module: simpler, all-internal, no versioning concerns.
- Multi-module: lets each library version independently; consumers can import a subset.

A library that's published to external users almost always wants its own module. Pure internal monorepos often choose single-module for simplicity.

### Workspace replace vs module replace

Both directives can appear. Precedence:

1. **Workspace `replace`** — applied across the build.
2. **Main module's `replace`** — applied if no workspace replace.
3. (Other modules' `replace`s are ignored, with or without workspace.)

If both define a replace for the same module, workspace wins.

### Build cache and workspace

The build cache is keyed by source content. When source is `./api` (local), the cache key changes on every edit. This makes workspace builds slightly noisier in CI (more cache misses) but doesn't affect correctness.

### Tests

`go test ./...` in workspace mode resolves all imports through the workspace. Cross-module integration tests just work without manual setup.

### `gopls` and workspaces

`gopls` reads `go.work` directly:

```
{
    "gopls": {
        "expandWorkspaceToModule": false
    }
}
```

VS Code / GoLand / Neovim auto-detect `go.work`. Renaming a symbol in `./api` updates references in `./server` and `./client`.

### Detecting workspace mode at runtime

```go
package main

import (
    "fmt"
    "runtime/debug"
)

func main() {
    bi, _ := debug.ReadBuildInfo()
    for _, s := range bi.Settings {
        if s.Key == "GOWORK" {
            fmt.Println("workspace mode:", s.Value)
        }
    }
}
```

The `GOWORK` build setting records whether the binary was built in workspace mode. Visible in `go version -m <binary>`.

## Standard Library Hooks

- `go work init [dirs...]`.
- `go work use [-r] [dirs...]`.
- `go work edit -use, -dropuse, -replace, -dropreplace, -go, -toolchain`.
- `go work sync`.
- `GOWORK` env var (auto / off / path).
- `-workfile=off` flag.
- `runtime/debug.ReadBuildInfo` — see `GOWORK` setting.
- `golang.org/x/mod/modfile` and `.../workspace` for programmatic editing.

## Real-World Patterns

### 1. Develop a library + consumer together

```bash
$ ls
api/    server/
$ go work init ./api ./server
$ cat go.work
go 1.26

use (
    ./api
    ./server
)

$ # Edit api/api.go, then build server
$ cd server && go build .   # uses local ../api source
```

No `replace` needed in `server/go.mod`; nothing to commit and revert.

### 2. Recursively add modules

```bash
$ find . -name go.mod
./api/go.mod
./libs/auth/go.mod
./libs/cache/go.mod
./server/go.mod

$ go work init
$ go work use -r .          # add every go.mod under .
$ cat go.work
go 1.26

use (
    ./api
    ./libs/auth
    ./libs/cache
    ./server
)
```

### 3. CI guard against workspace-only builds

```yaml
# .github/workflows/test.yml
jobs:
  with-workspace:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.26' }
      - run: go work init ./api ./server ./libs/*
      - run: go test ./...

  modules-standalone:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.26' }
      - env: { GOWORK: off }
        run: |
          for m in api server libs/auth libs/cache; do
            (cd $m && go test ./...)
          done
```

Two passes: workspace mode (fast, integrated) and standalone (verifies publishability).

### 4. Workspace replace for a fork shared across modules

```
// go.work
go 1.26

use (
    ./api
    ./server
)

replace github.com/buggy/lib => github.com/me/lib-fix v1.0.0
```

Both `api` and `server` get the forked `lib`, without editing each `go.mod`.

### 5. Generated `go.work` for monorepos

```bash
# scripts/setup-workspace.sh
go work init
go work use -r .
```

Committed script, gitignored `go.work`. New contributors run `./scripts/setup-workspace.sh` once.

## Anti-Patterns & Gotchas

**Committing `go.work` with absolute paths.** Breaks for anyone whose checkout isn't at the same path. Use only relative paths in committed `go.work`.

**Forgetting `go.work` exists and publishing a module.** The published `go.mod` doesn't see the workspace; downstream users get whatever versions are in `go.mod`. Run `go work sync` (or `GOWORK=off go test`) before tagging a release.

**Workspaces with stale `use` directives.** A `use ./removed-module` for a deleted directory errors:

```
go: invalid use ./removed-module: no go.mod file at the specified path
```

Edit `go.work` to remove.

**Treating `go.work` as a deployment artifact.** It's a dev convenience. Production builds use `go.mod` + `go.sum` only.

**Multiple `go.work` files in the same directory tree.** The Go tool picks the closest ancestor. Nested workspaces (`go.work` inside a `use`'d module's tree) are confusing and rarely needed.

**Mixing `replace` in `go.mod` and `go.work`.** Workspace replace wins. If you forget the workspace replace exists, you'll be puzzled why the module's own `replace` is ignored.

**Workspace `go` directive lower than any `use`'d module's.** Compile error: the workspace must support the most demanding member.

**Believing workspaces speed up imports.** They don't — local resolution is no faster than cache-resolved fetch. They just remove the version-bumping ceremony.

**Forgetting to add `go.work.sum`.** If you commit `go.work`, also commit `go.work.sum` (and vice versa); they pair.

**Workspace as a substitute for `internal/`.** A workspace exposes modules to each other but doesn't make them "private". Use `internal/` for visibility scope.

## Performance Notes

- Workspace parsing: <ms, trivial.
- `use ./mod` rewrites resolution to the local dir; no proxy fetch.
- `go build` in workspace: same speed as without, given a warm cache.
- `go work sync`: O(modules × deps); seconds on large workspaces.
- `gopls` indexes all `use`'d modules — memory cost scales linearly.

For small workspaces (≤10 modules) overhead is negligible. For huge workspaces (100+ modules), `gopls` and `go list ./...` slow noticeably; consider `GOWORK=off` for single-module tasks.

## How Big Companies Use It

- **Kubernetes** is technically a single-module mega-repo but uses `go.work`-style overlays in its `staging/` setup (which predates `go.work` but inspired aspects of it): https://github.com/kubernetes/kubernetes/blob/master/staging/README.md.
- **The Go team** uses `go.work` for the `golang.org/x/...` repos to develop multiple sub-modules together: https://go.googlesource.com/.
- **Tailscale** has documented multi-module workflows with `go.work` for `tailscale/corp` (internal) vs `tailscale/tailscale` (open source): https://github.com/tailscale.
- **CockroachDB** uses `go.work` for cross-module local builds during refactors: https://github.com/cockroachdb/cockroach.
- **Cloudflare** open-source projects often use multi-module repos with `go.work` for shared libraries: https://github.com/cloudflare.
- **Uber's Go monorepo** is single-module historically but documents `go.work` for selective splits.

## Source Code References

Pinned to `go1.26`.

- Workspace file parser: [`golang.org/x/mod/modfile/work.go`](https://github.com/golang/mod/blob/master/modfile/work.go).
- Workspace loading: [`src/cmd/go/internal/modload/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/init.go) — search `workFilePath`.
- `go work` command: [`src/cmd/go/internal/workcmd/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/workcmd).
- Workspace resolution: [`src/cmd/go/internal/modload/buildlist.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/buildlist.go) — search `workspace`.
- `go.work.sum` handling: [`src/cmd/go/internal/modload/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/init.go).
- gopls workspace support: [`golang.org/x/tools/gopls/internal/cache/workspace.go`](https://github.com/golang/tools/blob/master/gopls/internal/cache/workspace.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Tutorial: Get started with multi-module workspaces": https://go.dev/doc/tutorial/workspaces.
- "Go Workspaces in Go 1.18" (Go blog): https://go.dev/blog/get-familiar-with-workspaces.
- Bryan C. Mills, "Multi-module workspaces" design doc: https://go.googlesource.com/proposal/+/refs/heads/master/design/45713-workspace.md.
- Russ Cox, "Go modules — workspaces": https://research.swtch.com/.
- "Multi-module workflow" Go wiki: https://go.dev/wiki/MultiModuleRepositories.
- Brad Fitzpatrick, "Tailscale's open-source monorepo" (talks): https://tailscale.com/blog/.
- `go help work`: built-in docs.

## Exercises / Self-Check

1. You have `./api` and `./server` modules. `server` imports `github.com/me/api`, currently pinned to `v1.0.0` in `server/go.mod`. Initialize a workspace and verify local `./api` is used.
2. Write a CI step that builds your monorepo in workspace mode AND validates each module builds standalone.
3. Why might you choose to commit `go.work` to version control? Why might you choose to gitignore it?
4. A teammate published `./libs/auth` with a feature you needed locally — but the workspace makes it harder to reproduce their bug. How do you build server in module-only mode without removing `go.work`?
5. Add a workspace `replace github.com/x/y => ./forks/y`. Verify with `go list -m all` that the replacement is active across all `use`'d modules.
