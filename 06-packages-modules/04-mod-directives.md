# go.mod Directives — replace, exclude, retract, toolchain, godebug

## TL;DR

Beyond `module`, `go`, and `require`, the `go.mod` grammar has **five** less-common directives that solve specific operational problems: `replace` (substitute a module path/version), `exclude` (refuse a specific version), `retract` (declare your own published version unsafe), `toolchain` (require a specific Go toolchain, since 1.21), and `godebug` (pin runtime-behavior flags, since 1.21). The single biggest gotcha: **`replace` and `exclude` are only honored in the main module's `go.mod`** — if you depend on a library that has its own `replace`, those are ignored. This makes `replace` useful for end products and forks but unreliable as a propagating dep-management feature.

## Mental Model

```
   Your go.mod (main module):
       replace github.com/foo/bar v1.0.0 => ./local/bar          ← honored
       exclude github.com/baz/qux v1.4.0                          ← honored
       retract  v1.0.0-bad                                        ← affects YOUR consumers

   Library you depend on (its own go.mod):
       replace github.com/foo/bar v1.0.0 => ./local/bar          ← IGNORED
       exclude github.com/baz/qux v1.4.0                          ← IGNORED
       retract  v1.0.0-bad                                        ← visible to anyone using THAT lib

   toolchain go1.26.1            ← every module sees this; switches Go if needed
   godebug   default=go1.26      ← every module sees this; configures runtime
```

The asymmetry exists for a reason: a transitive `replace` would let any of your deps secretly rewrite the build. That breaks reproducibility and security. So only the *top* of the build (the main module) gets to override.

## Syntax & Basic Usage

```
module github.com/me/proj

go 1.26
toolchain go1.26.1

require (
    github.com/foo/bar v1.0.0
    github.com/x/y v0.5.0
)

replace github.com/foo/bar v1.0.0 => ./local/bar
replace github.com/x/y => github.com/me/y-fork v0.5.1

exclude github.com/buggy/lib v1.3.0
exclude github.com/buggy/lib v1.3.1

retract v0.1.0  // had a security bug
retract [v0.2.0, v0.2.5]  // range
retract (
    v0.3.0
    v0.3.1
)

godebug (
    httpmuxgo121=1
    panicnil=1
)
```

## Deep Dive

### `replace`

Two forms:

```
replace <old-path> [<old-version>] => <new-path> [<new-version>]
```

```
replace github.com/foo/bar => ./local/bar                       // dir replacement
replace github.com/foo/bar v1.0.0 => ./local/bar                // version-specific
replace github.com/foo/bar => github.com/me/bar-fork v1.2.3     // module replacement
replace github.com/foo/bar v1.0.0 => github.com/me/bar v1.2.3   // both specific
```

Semantics:

- **Dir replacement** (`=> ./path`): use the local checkout's source instead of downloading the upstream. Path is relative to the `go.mod` containing the directive.
- **Module replacement** (`=> github.com/me/fork v1.2.3`): use a different module entirely as the source. Useful for forks.
- The replacement module's own `go.mod` is read (its requires, etc.) — but its `replace` directives are NOT applied.

Use cases:

1. **Local development**: develop a library and its consumer simultaneously.
2. **Forks**: published bug fix not yet upstream.
3. **Forced backport**: pin a security-patched version.
4. **Renamed/moved modules**: pre-modules code that hasn't migrated.

In multi-module monorepos with active local development, prefer `go.work` (`use` directive) over `replace` — `go.work` is per-developer, doesn't pollute `go.mod`, and isn't shipped to consumers.

Replacement with no version on the LHS replaces **all versions** of the module:

```
replace github.com/foo/bar => ./local/bar     // every version uses ./local/bar
```

With a version on the LHS, replacement is version-specific:

```
replace github.com/foo/bar v1.0.0 => ./local/bar
// Only the v1.0.0 require resolves to ./local/bar; other versions resolve normally.
```

### `exclude`

```
exclude github.com/buggy/lib v1.3.0
```

Tells MVS "never select this version". When a require would resolve to v1.3.0, MVS picks the next available version up (v1.3.1, v1.4.0, etc.).

Used when:
- A specific release has a critical bug or vulnerability.
- A version was published with the wrong dependencies.
- Compatibility issues with your code that the upstream hasn't fixed.

Like `replace`, only the main module's `exclude` is honored. If your dep `A` excludes a version that your transitive dep `B` requires, `B`'s require wins.

### `retract`

```
retract v0.1.0
retract [v0.2.0, v0.2.5]
retract (
    v0.3.0 // exact
    v0.3.1 // exact
)
```

Authors declare versions of *their own module* as withdrawn:

```
// go.mod of github.com/me/lib
retract v1.2.0  // had a data-corruption bug
```

When a user runs `go get github.com/me/lib@latest`, the Go tool reads the latest tagged version's `go.mod`, observes the retraction, and selects a non-retracted version.

A retracted version *can* still be downloaded if explicitly requested (`go get github.com/me/lib@v1.2.0` works), but with a warning. Existing `go.mod`s pinning v1.2.0 keep working — retraction doesn't break builds, just discourages.

**Critical**: retraction must be published in a *later* tag than the retracted one. The Go tool reads `retract` from the highest available version's `go.mod`.

Example flow:
1. Tag v1.2.0. Discover bug.
2. Tag v1.2.1 with a fix.
3. Add `retract v1.2.0` in v1.2.1's `go.mod` (or in a later v1.2.2 if you missed it).
4. Publish v1.2.2 with the retraction.

`go list -m -retracted all` shows retractions.

### `toolchain`

```
go 1.26
toolchain go1.26.1
```

(Since Go 1.21.)

Says "build me with at least `go1.26.1`". The Go command, by default, will *download* and switch to this toolchain if the current one is older. Controlled by `GOTOOLCHAIN`:

```
GOTOOLCHAIN=auto         # default: respect go.mod's `toolchain` directive (and `go` directive)
GOTOOLCHAIN=local        # never download; error if mismatch
GOTOOLCHAIN=go1.26.1     # force a specific toolchain
GOTOOLCHAIN=go1.26.1+auto  # at least 1.26.1, but allow newer if go.mod requires
GOTOOLCHAIN=path         # use whatever's in PATH (closest to legacy)
```

The toolchain is downloaded via `https://proxy.golang.org` (as a special "module-like" artifact) into `$GOTOOLCHAIN`.

Practical use:

- Library authors: set `go 1.26` + `toolchain go1.26.1` to require a known-good toolchain.
- Application authors: same; the build is reproducible across machines without per-machine Go installs.
- CI: `GOTOOLCHAIN=auto` (default) for normal builds; `GOTOOLCHAIN=local` to detect drift.

### `godebug` (since 1.21)

```
godebug (
    httpmuxgo121=1
    panicnil=1
)
```

Sets `GODEBUG` values at build time, scoped to this module's binaries. These flags control runtime behavior — typically to opt in/out of newer-version defaults.

Examples:

- `httpmuxgo121=1`: use Go 1.21's `http.ServeMux` semantics (no pattern matching) even on 1.22+ toolchains.
- `panicnil=1`: panic with nil (legacy behavior) instead of `*runtime.PanicNilError` (the 1.21+ default).
- `randautoseed=0`: opt out of `math/rand`'s automatic seeding (pre-1.20 behavior).

Setting `godebug` in `go.mod` makes the behavior part of the build artifact — different from `GODEBUG=...` at runtime, which can be overridden per-invocation.

Without a `godebug` line, defaults follow the `go` directive: `go 1.26` selects 1.26-era defaults; `go 1.20` selects 1.20-era defaults. This is the "Go compatibility promise" mechanism — code written against 1.20 keeps its 1.20-era behavior even when run on 1.26.

See https://go.dev/doc/godebug for the full flag list.

### Order of directives

`go.mod` does not enforce a strict directive order, but conventional grouping:

```
module github.com/me/proj

go 1.26
toolchain go1.26.1

require (
    ...
)

require (
    ...    // indirect
)

replace (
    ...
)

exclude (
    ...
)

retract (
    ...
)

godebug (
    ...
)
```

`go mod tidy` rewrites `go.mod` in this order.

### Comments

```
require github.com/x/y v1.2.3 // indirect, kept for plugin

retract v1.0.0 // CVE-2024-XXXX: SQL injection
```

End-of-line comments are preserved across `go mod tidy`. Block comments above directives are also preserved (mostly). Use them; future you will thank present you.

### Removing a directive

`go mod edit -dropreplace=github.com/foo/bar` and similar `-drop*` flags work programmatically. Or hand-edit and tidy.

### Limitations of `replace`

- Cannot replace the main module itself.
- Cannot replace a module to a different *module path* (rare; happens in major-version-bump scenarios). Actually, you can: `replace foo => bar v1.0.0` is legal as long as `bar`'s `go.mod` declares `module foo`. The "real" module path of the replacement must match. Tricky.
- Replace doesn't propagate through `vendor/`; the vendored content is whatever was vendored.
- Multiple replaces of the same module + version is an error.

### `replace` in `go.work`

Workspaces support `replace` too (in the workspace's `go.work` file). Workspace-level replace overrides any in `go.mod`s under `use` directives. Useful for short-term overrides during multi-module work.

```
go 1.26

use (
    ./moduleA
    ./moduleB
)

replace github.com/foo/bar => ./forks/bar
```

### `retract` and the proxy

When you publish a retraction:

1. Tag a new version that includes `retract` directives.
2. Push the tag.
3. The proxy (`proxy.golang.org`) eventually re-fetches and notes the retraction.
4. `go get @latest` for downstream users skips retracted versions.

You can speed propagation by hitting `https://proxy.golang.org/<module>/@v/<version>.info` directly.

### CI patterns

Pin toolchains in `go.mod` for reproducibility:

```
go 1.26
toolchain go1.26.1
```

CI runs with `GOTOOLCHAIN=auto`; everyone gets `go1.26.1` regardless of their installed Go.

Detect drift:

```bash
$ GOTOOLCHAIN=local go build ./...
```

Fails if the local Go doesn't match. Use in a "verify-toolchain" CI job.

## Standard Library Hooks

- `go mod edit -replace`, `-dropreplace`, `-exclude`, `-dropexclude`, `-retract`, `-dropretract`, `-toolchain`, `-godebug`, `-dropgodebug`.
- `go mod tidy` — re-sorts and removes unused.
- `go list -m -retracted` — show retractions.
- `go list -m -versions` — show all versions, marks retracted.
- `golang.org/x/mod/modfile` — programmatic editing.
- `GOTOOLCHAIN` env var — toolchain switching.
- `GODEBUG` env var — runtime override of `godebug` directives.

## Real-World Patterns

### 1. Replace for a security backport

```
require github.com/lib/pq v1.10.9
replace github.com/lib/pq v1.10.9 => github.com/myorg/pq-secured v1.10.9-patch1
```

You consume the patched fork transparently. When upstream releases v1.10.10 with the fix, remove the replace.

### 2. Replace for monorepo local dev

```
// In services/api/go.mod
replace github.com/myorg/libs/auth => ../libs/auth
```

Better: use `go.work`:

```
// /go.work
use (
    ./services/api
    ./libs/auth
)
```

`go.work` is gitignored by convention; per-developer. `replace` in `go.mod` is committed and breaks for everyone else.

### 3. Retract a bad version

```bash
# After noticing v1.5.0 had a bug:
$ git tag v1.5.1
$ cat go.mod
module github.com/me/lib

go 1.26

retract v1.5.0  // serialization bug; use v1.5.1

$ git commit -am "retract v1.5.0"
$ git tag v1.5.1
$ git push --tags
```

Downstream `go get @latest` skips v1.5.0.

### 4. Toolchain pinning

```
// go.mod
go 1.26
toolchain go1.26.1
```

```yaml
# CI: install minimal Go; let toolchain directive fetch the right one
- uses: actions/setup-go@v5
  with:
    go-version: '1.21'   # any version ≥ 1.21 for toolchain support
- run: go build ./...
```

CI installs only the bootstrap Go; the `toolchain` directive fetches `go1.26.1` on demand. Same binary across all CI environments.

### 5. godebug for backward-compat

```
// go.mod of a library still depending on Go 1.21 ServeMux behavior
go 1.22

godebug (
    httpmuxgo121=1
)
```

The library forces 1.21 `http.ServeMux` semantics even when compiled with 1.22+. Buys time to migrate to 1.22's new routing patterns.

## Anti-Patterns & Gotchas

**Using `replace` for long-lived "vendor mode lite".** It works but rots — every dep upgrade has to re-pin the replace target. Eventually it diverges from upstream. Either contribute the fix upstream, or fork properly with module-path replacement.

**Replacing with a non-`go.mod` directory.** Your local path must contain a valid `go.mod`. Otherwise `go build` errors.

**`exclude` to "fix" a security issue.** `exclude` makes MVS skip a version, but you might end up with an *older* version that's worse. Use `replace` to a patched fork or upgrade past the bad version.

**Retracting without a higher tag.** Retraction is read from the *highest* version. If v1.2.0 is bad and you add `retract v1.2.0` directly to v1.2.0, downstream never sees it. Always tag a new version with the retract.

**Pinning `toolchain go1.26.0` and then patching `toolchain go1.26.1` only in some modules.** MVS picks the highest toolchain across all `go.mod`s in the build; the result is fine but can confuse "what toolchain am I using?" — print `runtime.Version()` to be sure.

**`godebug` for opaque experimental flags.** Use sparingly. Future-Go may remove the flag entirely; your build then fails.

**Committing `go.work` with `replace`s pointing at developers' home dirs.** `go.work` should be gitignored or contain only paths relative to the repo root.

**Believing transitive `replace`s work.** They don't. If you depend on library X that depends on library Y, and you want Y replaced, *you* must add the replace.

**`require` + `replace` to a different version mismatch.** `require github.com/x/y v1.2.0` plus `replace github.com/x/y v1.5.0 => ...`: the require is for v1.2.0; the replace is for v1.5.0; the replace never fires. Either match versions or omit the version on the replace LHS.

**Using `+incompatible` semantics in a replacement.** Replace `github.com/foo/bar v2.0.0+incompatible => github.com/me/fork v2.0.0` requires the fork's `go.mod` to declare `module github.com/foo/bar` (with the v2-prefix if applicable). Easy to misconfigure.

## Performance Notes

- Directives are parsed once per build; resolution is dominated by `require` handling.
- `replace` to a local dir skips proxy fetch; saves a few hundred ms per dep.
- `exclude` and `retract` add MVS iterations but are O(n) in deps; negligible.
- `toolchain` may trigger a Go download (~100 MiB, one-time per version).
- `godebug` directives don't affect build time, only runtime behavior.

## How Big Companies Use It

- **Tailscale** uses `replace` for short-term forks of dependencies they intend to upstream; documented in their CONTRIBUTING: https://github.com/tailscale/tailscale.
- **CockroachDB** uses `exclude` for known-bad versions of upstream deps: https://github.com/cockroachdb/cockroach/blob/master/go.mod.
- **Kubernetes** publishes its API libraries as separate modules and uses `replace` heavily for cross-module local builds in monorepo-mode: https://github.com/kubernetes/kubernetes.
- **HashiCorp** uses `retract` on internal CVEs: e.g., `vault@v1.x` retractions on known-vulnerable patch versions.
- **The Go team itself** pioneered `retract` with `golang.org/x/...` modules when they pushed broken versions during the 1.18 generics rollout.
- **gRPC-Go** uses `godebug` to maintain backward compat with older gRPC clients: https://github.com/grpc/grpc-go.
- **Cloudflare's edge** uses pinned toolchains (`toolchain` directive) for deterministic CI builds.

## Source Code References

Pinned to `go1.26`.

- `go.mod` parser/serializer: [`golang.org/x/mod/modfile/rule.go`](https://github.com/golang/mod/blob/master/modfile/rule.go).
- Replace handling: [`src/cmd/go/internal/modload/buildlist.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/buildlist.go) — search `replace`.
- Exclude handling: same file, search `exclude`.
- Retract: [`src/cmd/go/internal/modload/list.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/list.go) — search `Retracted`.
- Toolchain switching: [`src/cmd/go/internal/toolchain/select.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/toolchain/select.go).
- `godebug` directive: [`src/internal/godebug/godebug.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/internal/godebug/godebug.go).
- `go mod edit` implementation: [`src/cmd/go/internal/modcmd/edit.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/edit.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go Modules Reference" (full directive grammar): https://go.dev/ref/mod#go-mod-file.
- "Module retract directive": https://go.dev/ref/mod#go-mod-file-retract.
- Russ Cox, "Go 1.21 toolchains" proposal: https://go.googlesource.com/proposal/+/refs/heads/master/design/57001-gotoolchain.md.
- Russ Cox, "Go, Backwards Compatibility, and GODEBUG": https://go.dev/blog/compat.
- "godebug history" — Russ Cox: https://research.swtch.com/godebug-history.
- "When to use replace vs go.work" (Go FAQ-style): https://go.dev/wiki/MultiModuleRepositories.
- "Retract a Go module version": https://go.dev/wiki/Modules#how-to-retract-a-module-version.
- `go help mod`, `go help mod edit`: built-in docs.

## Exercises / Self-Check

1. You discover your dep `github.com/x/y v1.2.0` has a bug and the upstream is unresponsive. Sketch the `go.mod` for using your patched fork at `github.com/me/y-fork v1.2.0-fix1`.
2. Why must `retract` directives appear in a *later* tag than the version being retracted? What happens if they're in the same tag?
3. A library at `go 1.20` is consumed by your app at `go 1.26`. Which `godebug` defaults apply to the library's code?
4. With `toolchain go1.26.1`, `GOTOOLCHAIN=local`, and an installed `go1.25.0`, what happens when you run `go build`?
5. Compare `replace` in `go.mod` vs `use` in `go.work`. Give two cases where each is preferable.
