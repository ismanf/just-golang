# `go work` — Multi-Module Workspaces

## TL;DR

A **workspace** (introduced in Go 1.18) lets you edit several modules together as if they were one module, without committing throwaway `replace` directives. The marker file is **`go.work`**: it lists the local module directories you want active. When `go.work` is present, the `go` tool *overrides* normal module resolution for those paths — your build sees the in-tree code instead of the proxy-fetched version. Crucial points: (1) `go.work` should **never** be committed to a published library (it changes builds for downstream consumers in unintended ways), (2) `go.work` is the right tool for multi-module mono-repos and for live-editing a dep, (3) `go work sync` propagates dep choices from the workspace back into each module's `go.mod`, and (4) since 1.22, `go work vendor` writes a workspace-level `vendor/`.

## Mental Model

```
   /repo/
   ├── go.work                  ← workspace file (active modules)
   │     go 1.26
   │     use (
   │         ./api
   │         ./worker
   │         ./shared
   │     )
   │
   ├── api/         ── go.mod (github.com/me/api)
   ├── worker/      ── go.mod (github.com/me/worker, requires github.com/me/shared)
   └── shared/      ── go.mod (github.com/me/shared)


   With go.work present, an import of github.com/me/shared from worker/
   resolves to ../shared/ on disk, NOT a proxy fetch.

   Without go.work, worker/ would fetch github.com/me/shared per its go.mod.
```

A workspace is *additive* over `go.mod`: each listed module keeps its own `go.mod`, and the workspace's job is just to redirect specific imports to the in-tree copy.

## Syntax & Basic Usage

```bash
$ go work init ./api ./worker ./shared
# creates go.work in current dir

$ go work use ./newmodule           # add a module
$ go work use -r .                  # recursively add all modules under .
$ go work edit -dropuse=./old       # remove
$ go work edit -replace=github.com/foo/bar=../bar
$ go work edit -go=1.26
$ go work sync                      # push deps from workspace back to each go.mod
$ go work vendor                    # 1.22+: vendor across the workspace
$ go env GOWORK                     # path to go.work (or "off")
```

Minimal `go.work`:

```
go 1.26

use (
    ./api
    ./worker
    ./shared
)

replace github.com/some/upstream => ../forks/upstream
```

## Deep Dive

### What `go.work` actually does

When Go is invoked in a directory under a `go.work`, the workspace's `use` list joins all the listed module paths into the build's *main module set*. Every import resolves to the first match across all main modules; only if no main module provides the import does the resolver fall back to the regular dep graph.

The dependency selection is still MVS-driven, but it considers all `go.mod`s under the `use` list as if they were one combined module's requires. The selected versions feed every build inside the workspace, regardless of which module you're standing in.

Implementation: [`src/cmd/go/internal/workcmd`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/workcmd).

### `go work init`

```bash
$ go work init ./api ./worker ./shared
```

Creates `go.work` listing the three directories. Each arg is a relative path to a directory that already contains a `go.mod`.

Without args, creates an empty workspace; add modules later via `go work use`.

### `go work use`

```bash
$ go work use ./newpkg               # add
$ go work use -r ./services          # recursively add every go.mod under ./services
$ go work edit -dropuse=./oldpkg     # remove
```

`-r` is the bulk option for monorepos with many modules. It scans for `go.mod` files and adds each as a `use` entry.

### `go work edit`

The programmatic interface, mirroring `go mod edit`:

```bash
$ go work edit -use=./newpkg
$ go work edit -dropuse=./oldpkg
$ go work edit -replace=github.com/foo/bar=../bar
$ go work edit -dropreplace=github.com/foo/bar
$ go work edit -go=1.26
$ go work edit -toolchain=go1.26.1
$ go work edit -godebug=httpmuxgo121=1
$ go work edit -json                 # dump as JSON
$ go work edit -fmt                  # canonicalize
$ go work edit -print                # write to stdout
```

### `go work sync`

```bash
$ go work sync
```

After the workspace's MVS resolves to a set of dep versions, `sync` writes those back into each `use`d module's `go.mod`. Useful when you've been developing with the workspace overriding versions and want each module's `go.mod` to reflect the current truth before publishing or before tagging.

Common cycle:

1. Edit code across `api/`, `worker/`, `shared/` with `go.work` active.
2. `go test ./...` in each.
3. `go work sync` — pushes resolved deps to each `go.mod`.
4. Commit `go.mod` updates per module.
5. (Maybe) delete `go.work` if you're done.

### `go work vendor` (since 1.22)

```bash
$ go work vendor
```

Writes a single `vendor/` at the workspace root containing every dep of every `use`d module. Builds inside the workspace default to `-mod=vendor` after this runs.

Workspace vendoring is *not* the same as per-module vendoring (`go mod vendor`). The workspace vendor merges everything; the per-module ones each have their own.

### `go env GOWORK`

```bash
$ go env GOWORK
/Users/me/repo/go.work
$ GOWORK=off go build ./...     # ignore workspace, build as if it didn't exist
$ GOWORK=/some/path/go.work go build ./...
```

The `auto` value (default) discovers `go.work` by walking up from cwd. `off` disables workspace mode entirely. An explicit path uses that file regardless of cwd.

When debugging "why isn't my build seeing my local copy of `shared`?", check `go env GOWORK` first.

### `go.work.sum`

Just like `go.sum` but for the workspace. Hashes for any modules whose `go.mod`s the workspace had to consult during MVS but which aren't selected. Generally machine-managed; commit alongside `go.work` if you commit `go.work`.

### Interaction with `go.mod` `replace`

If `go.work` has a `replace` and a `go.mod` has a different `replace` for the same module, **the workspace `replace` wins**. This is the documented behavior and is what lets you point everything at a local fork in one place.

If `go.work` has no replace but a `go.mod` does, the `go.mod` replace applies.

### What's NOT a workspace

- A multi-package single module is *not* a workspace. There's one `go.mod`; just use sub-packages.
- A `go.mod` with `replace ../sibling` is *not* a workspace; it's the old, manual approach (and breaks CI on machines without the path).

Use a workspace when:

- You have multiple `go.mod` files in one repo (multi-module mono-repo).
- You want to live-edit a dep alongside the consuming code.
- You want CI to test the whole repo as one unit.

### When NOT to commit `go.work`

For a **published library**, never commit `go.work`. If you publish module `github.com/me/lib`, anyone who clones the repo and runs `go build` from inside `lib/` will see your workspace overrides — they'll be using `replace` lines they didn't write, fetching local paths that don't exist on their machine. Breaks builds.

For an **application monorepo**, committing `go.work` is fine and expected — it pins the workspace structure for everyone.

The convention: `.gitignore` `go.work*` in library repos; commit them in app repos.

### `go work` and per-module `go.sum`

Each `use`d module still has its own `go.sum`. The workspace doesn't replace them; `go work sync` keeps them current with the workspace-resolved versions.

If a module's `go.sum` drifts from workspace truth, `go build` will refuse to compile that module until either `go.sum` updates or the workspace is removed.

### CI in a workspace

```yaml
- run: go test ./...               # tests every package across all workspace modules
- run: go vet ./...
- run: go work sync                 # ensure no drift
- run: git diff --exit-code         # fail if sync changed anything
```

`./...` in a workspace expands across every `use`d module. Single CI step covers them all.

## Standard Library Hooks

- `golang.org/x/mod/modfile.ParseWork` — programmatic `go.work` parsing.
- `golang.org/x/tools/go/packages` — workspace-aware analysis (set `Mode` to include `NeedModule`).
- `runtime/debug.ReadBuildInfo` — at runtime, sees the workspace-resolved versions (not the per-module ones if they differed).
- `cmd/go/internal/workcmd` — internal; the source if you're writing a workspace-aware tool.

## Real-World Patterns

### 1. Edit a dep live

```bash
$ git clone https://github.com/me/myapp
$ git clone https://github.com/me/mylib
$ cd myapp
$ go work init . ../mylib
$ vim ../mylib/foo.go       # edit
$ go test ./...             # picks up the edit immediately
```

No commits, no `replace` edits. When done, remove `go.work` or move to PR-land.

### 2. Monorepo workspace at repo root

```bash
$ tree
.
├── go.work
├── apps/
│   ├── api/      (go.mod: github.com/me/api)
│   └── web/      (go.mod: github.com/me/web)
└── libs/
    ├── auth/     (go.mod: github.com/me/auth)
    └── db/       (go.mod: github.com/me/db)

$ cat go.work
go 1.26
use (
    ./apps/api
    ./apps/web
    ./libs/auth
    ./libs/db
)
```

`go test ./...` from the repo root tests everything.

### 3. Recursive workspace setup

```bash
$ go work init
$ go work use -r .
```

Walks the tree and adds every `go.mod` it finds.

### 4. Sync after a coordinated change

```bash
$ # editing both libs/auth and apps/api
$ go test ./...
$ go work sync
$ git diff -- apps/api/go.mod libs/auth/go.mod
$ git add -A && git commit
```

### 5. Workspace vendor for a release

```bash
$ go work vendor
$ git add vendor/ go.work go.work.sum
$ git commit -m "vendor workspace deps"
```

Build then ignores `$GOMODCACHE` entirely. Useful for hermetic CI in air-gapped environments.

### 6. Toggling workspace off for a release build

```bash
$ GOWORK=off go build -trimpath -o app ./apps/api
```

Ensures the release uses the published dep versions, not local edits.

## Anti-Patterns & Gotchas

**Committing `go.work` to a library repo.** Downstream users get unexpected overrides. `.gitignore go.work*` for libraries.

**Forgetting `go work sync` before tagging a module.** The `go.mod` you tag may not reflect the versions you actually tested with.

**Using `replace ../sibling` in `go.mod` instead of `go.work`.** The `go.mod` form requires every contributor to have the same filesystem layout. Workspaces don't.

**Building from inside a workspace, expecting publishable behavior.** Workspace overrides may hide a missing dep. Test with `GOWORK=off` before tagging.

**Adding a module to `go.work` without a `go.mod` in it.** `go work use` requires each entry to point at an existing module. Error if missing.

**Workspace with modules at incompatible Go versions.** If `apps/api`'s `go.mod` says `go 1.21` and `libs/auth`'s says `go 1.26`, the workspace's `go` line (e.g., `1.26`) governs the build. The 1.21 module may break if it uses 1.26-only features inadvertently.

**Forgetting `go.work.sum`.** It's the workspace equivalent of `go.sum`. Commit when you commit `go.work`.

**Manually editing `go.work` and forgetting `go work sync`.** The per-module `go.sum`s may go stale, leading to "checksum mismatch" on next build.

**Workspace plus `tools.go` (1.24-era tools).** Workspaces and the `tool` directive in `go.mod` coexist fine, but `go work` doesn't have a `tool` directive of its own. Tools live in the per-module `go.mod` even in a workspace.

**Workspace `replace` pointing at a path outside the workspace.** Works, but couples the workspace to a machine-specific layout. Prefer in-workspace `use` entries.

**Using a workspace as a "build everything" shortcut without intent.** If two modules genuinely should not share deps (different upgrade cadence, different audiences), forcing them into one workspace forces MVS to reconcile their requires — sometimes unhelpfully.

## Performance Notes

- `go work init`: <100 ms.
- `go work use -r .` on a 50-module repo: ~200 ms (filesystem walk).
- `go work sync` warm: ~1 s for ~10 modules.
- `go work vendor`: scales with total dep size; 1–10 s for medium projects.
- `go env GOWORK` (the discovery walk): <10 ms.

In a workspace, `go test ./...` rebuilds nothing extra vs. running tests per-module sequentially — the cache hits across modules.

## How Big Companies Use It

- **Google** uses internal Bazel for most monorepo work, but external projects like `golang.org/x/tools` use `go.work` for cross-module dev: https://go.googlesource.com/tools.
- **Kubernetes** uses a non-workspace multi-module repo with hand-managed `replace` directives; they avoided `go.work` because some build steps predate 1.18: https://github.com/kubernetes/kubernetes.
- **Tailscale** uses `go.work` for live-editing of `tsnet` against the main `tailscale` repo: https://github.com/tailscale/tailscale.
- **CockroachDB** considered workspaces for their multi-module migration but stuck with Bazel: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses `go.work` in some new repos (e.g., `consul-net-rpc`) but Terraform itself remains a single module: https://github.com/hashicorp/terraform.
- **The Go team** maintains `golang.org/x/*` with workspaces during cross-cutting changes: https://github.com/golang/tools.
- **Discord** keeps each Go service as its own module and uses an "umbrella" `go.work` per developer machine for cross-service refactors.

## Source Code References

Pinned to `go1.26`.

- `go work` umbrella: [`src/cmd/go/internal/workcmd`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/workcmd).
- `go work init`: [`src/cmd/go/internal/workcmd/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/workcmd/init.go).
- `go work use`: [`src/cmd/go/internal/workcmd/use.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/workcmd/use.go).
- `go work sync`: [`src/cmd/go/internal/workcmd/sync.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/workcmd/sync.go).
- `go work vendor`: [`src/cmd/go/internal/workcmd/vendor.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/workcmd/vendor.go).
- Workspace loader: [`src/cmd/go/internal/modload/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/init.go) (search `goWorkfile`).
- `go.work` parser: [`golang.org/x/mod/modfile`](https://github.com/golang/mod/blob/master/modfile/work.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Workspaces" (Go ref): https://go.dev/ref/mod#workspaces.
- "Get familiar with workspaces" (Go blog): https://go.dev/blog/get-familiar-with-workspaces.
- Original design doc — Russ Cox: https://go.googlesource.com/proposal/+/master/design/45713-workspace.md.
- "Multi-module workspaces in Go 1.18" (Beyang Liu): https://about.sourcegraph.com/blog/go-1-18-multi-module-workspaces.
- "Go workspaces tutorial" (Go docs): https://go.dev/doc/tutorial/workspaces.

## Exercises / Self-Check

1. Create a workspace with two modules where `module-a` depends on `module-b`. Edit `module-b` and confirm `module-a`'s tests see the change immediately.
2. Run `GOWORK=off go test ./...` from inside a workspace. Does it fall back to the proxy? Why is this useful before tagging?
3. Add a `replace` directive in `go.work` and a conflicting one in a `go.mod`. Which wins? Verify via `go list -m -json`.
4. Use `go work vendor`. Compare the resulting `vendor/` to per-module `go mod vendor` output.
5. Why does `.gitignore go.work` belong in a library repo but not in an application repo?
