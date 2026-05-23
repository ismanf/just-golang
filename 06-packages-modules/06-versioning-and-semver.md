# Versioning and Semver — `/v2+`, Pre-Release Tags

## TL;DR

Go modules use **Semantic Versioning 2.0.0** (`vMAJOR.MINOR.PATCH`) with one Go-specific twist: **major version ≥ 2 requires a `/vN` suffix in the module path** (`github.com/me/lib/v2`, not `github.com/me/lib`). The Go tool enforces this — a v2.x.x tag without the suffix in `go.mod` is rejected (with the `+incompatible` legacy escape). **Pre-release tags** (`v1.2.3-rc1`, `v1.2.3-beta.1`) are valid but excluded from `@latest`. **Pseudo-versions** (`v0.0.0-20240115...`) name untagged commits. The single biggest gotcha: **the major-version-suffix rule is the source of >90% of "weird module errors"** — `import "github.com/me/lib"` vs `import "github.com/me/lib/v2"` is a hard renaming break, not a soft migration.

## Mental Model

```
   v0.x.x     → pre-1.0; anything can change
   v1.x.x     → stable; minor = additive, patch = bugfix
   v2.x.x+    → REQUIRES /v2 suffix in module path AND import path
                module github.com/me/lib/v2
                import "github.com/me/lib/v2"

   v1.2.3-rc.1, v1.2.3-beta, v1.2.3-alpha.4
                → pre-release; sorted as pre-release < release
                → excluded from go get @latest

   v0.0.0-20240115T120000-abc123def456
                → pseudo-version for untagged commits
                → ordered chronologically

   v2.0.0+incompatible
                → escape hatch for pre-modules v2 libraries
                → discouraged in new code
```

The semver rules are mostly standard. The `/vN` suffix is what makes Go modules unusual — it baked the major version into the import path itself, so v1 and v2 of a library can coexist in one binary.

## Syntax & Basic Usage

```bash
$ git tag v1.2.3              # release tag
$ git tag v1.2.3-rc.1         # release candidate
$ git tag v2.0.0              # major bump — requires module path change!
$ git push --tags
```

Module declaring v2:

```
// github.com/me/lib v2.x.x
module github.com/me/lib/v2

go 1.26
```

Consumer of v2:

```go
import "github.com/me/lib/v2"
```

Consumer of v1 (same project):

```go
import "github.com/me/lib"           // still resolves to v1
```

You can have both at once:

```go
import (
    libv1 "github.com/me/lib"
    libv2 "github.com/me/lib/v2"
)
```

## Deep Dive

### Semver basics

`vMAJOR.MINOR.PATCH`:

- **MAJOR**: incompatible API changes (removed function, changed signature, changed semantics).
- **MINOR**: backward-compatible additions (new function, new optional parameter struct field).
- **PATCH**: backward-compatible bug fixes only.

A "v" prefix is required by Go (`v1.2.3`, not `1.2.3`). Most Git ecosystems accept both; Go tooling requires the `v`.

Comparison: `v1.2.3 < v1.2.10 < v1.3.0 < v2.0.0`. Components are integers, not strings — `1.2.10` > `1.2.9`.

### The `/vN` rule

Once you go to v2+, the module path must end in `/vN`:

```
v0.x.x, v1.x.x  →  module github.com/me/lib
v2.x.x          →  module github.com/me/lib/v2
v3.x.x          →  module github.com/me/lib/v3
...
```

Two reasons:

1. **Two majors in one binary**: v1 and v2 of the same library can both be imported simultaneously. Their import paths differ, so they don't conflict.
2. **Forced migration**: a breaking change forces downstream code to update *every import line*, which is the right place to confront the breakage.

The `/vN` suffix is in:
- The `module` directive in `go.mod`.
- Every `import` line.

It is **not** in the Git tag (`v2.0.0`, not `v2/v2.0.0`). Tags don't carry path info; the path is asserted by `go.mod`.

### Where the `/vN` lives in the repo

Two common layouts:

**Option A — major version in subdirectory:**

```
github.com/me/lib/
├── go.mod                    module github.com/me/lib   (v1)
├── lib.go
└── v2/
    ├── go.mod                module github.com/me/lib/v2
    └── lib.go
```

Pros: v1 and v2 sources visible side-by-side. Easy to share code between them (`internal/`).
Cons: more boilerplate, weird-looking directory structure.

**Option B — major version on a branch:**

```
github.com/me/lib/  (branch: v2)
├── go.mod          module github.com/me/lib/v2
├── lib.go

github.com/me/lib/  (branch: main, default)
├── go.mod          module github.com/me/lib
├── lib.go
```

Pros: clean tree; v1 and v2 are independent branches.
Cons: cross-version refactor is harder; tags must be on the right branch.

Both are valid. Most popular libraries use **Option A** because it allows v1 maintenance from the same checkout.

### `+incompatible`

Some pre-modules libraries published v2+ tags without following the `/vN` rule. The Go tool tolerates these with the `+incompatible` suffix:

```
require github.com/x/y v2.1.0+incompatible
```

The `+incompatible` is a *build metadata* in semver terms. The Go tool interprets it as "you're using v2 of an old-style module that doesn't follow the rule; both sides agree to ignore the conflict".

Limitations:
- Discouraged for new modules. Adds friction; consumers must include `+incompatible`.
- Once a `+incompatible` module migrates to `/v2`, both paths exist simultaneously for transition.

### Pre-release versions

```
v1.0.0-alpha
v1.0.0-alpha.1
v1.0.0-beta
v1.0.0-rc.1
v1.0.0
```

Pre-release ordering (per semver spec):

- A version with a pre-release tag is **less** than the same version without (`v1.0.0-rc.1 < v1.0.0`).
- Pre-release identifiers compare lexically (numeric < alphanumeric).
- `v1.0.0-alpha.1 < v1.0.0-alpha.2 < v1.0.0-alpha.10` (numeric comparison).
- `v1.0.0-alpha < v1.0.0-beta < v1.0.0-rc`.

Practical impact:

- `go get @latest` ignores pre-release versions by default. To get one explicitly: `go get github.com/x/y@v1.0.0-rc.1`.
- `go list -m -versions github.com/x/y` shows all, including pre-releases.

### Pseudo-versions

When a require points at an untagged commit (or a tag-less branch), `go.mod` records a synthetic version:

```
require github.com/x/y v0.0.0-20240115120000-abc123def456
```

Three forms:

1. `vX.0.0-yyyymmddhhmmss-shortsha` — commit on a branch with no tags.
2. `vX.Y.Z-pre.0.yyyymmddhhmmss-shortsha` — commit between two pre-release tags.
3. `vX.Y.(Z+1)-0.yyyymmddhhmmss-shortsha` — commit after a release tag (the next "patch in progress").

The timestamp is UTC, in the commit. The shortsha is the first 12 hex chars of the commit. The Go tool resolves a pseudo-version by asking the proxy:

```
GET https://proxy.golang.org/github.com/x/y/@v/v0.0.0-20240115120000-abc123def456.info
```

Response: JSON with `Version`, `Time` (full timestamp), commit sha.

### Pseudo-version vs tag

Pseudo-versions are an *escape hatch* for "I don't have a tag yet". Once a real tag exists at that commit, MVS prefers the tag (it sorts higher in the version order).

Pseudo-versions are useful for:
- Bleeding-edge dev against unreleased libraries.
- Pinning a fork at a specific commit when the fork doesn't tag.
- Reproducible builds against `main`.

Avoid in production where possible — pseudo-versions are ugly, opaque to humans, and require the source proxy to resolve.

### Stability promise

**Pre-1.0 (v0.x.x)**: no compatibility guarantee. Anything can change.
**v1.x.x and beyond**: minor and patch versions must remain backward-compatible per semver. Breaking changes require a major bump.

This is a convention, not enforced by the compiler. The Go tooling assumes it: `go get -u` happily upgrades within a major version. A library that breaks compat in a minor release is "doing semver wrong" and will surprise users.

The Go standard library itself has its own [compatibility promise](https://go.dev/doc/go1compat) — backward-compatible additions only within Go 1.x.

### Branches

Tags are versions. Branches are not versions, but you can refer to them:

```bash
$ go get github.com/x/y@main
$ go get github.com/x/y@master
$ go get github.com/x/y@release-1.2
```

The resolution: find the latest commit on that branch, generate a pseudo-version, write to `go.mod`. Subsequent builds use that pseudo-version, not "the current main HEAD". To re-pin, run `go get @branch` again.

### Commits

```bash
$ go get github.com/x/y@abc123def
```

Resolve to a pseudo-version pinned at that commit.

### `go list -m -versions`

```bash
$ go list -m -versions github.com/google/uuid
github.com/google/uuid v0.1.0 v0.1.1 v1.0.0 v1.1.0 ... v1.6.0
```

Lists every tagged version known to the proxy. Useful for upgrade planning.

### Module path != Git URL (always)

```
module path: github.com/me/lib
git URL:     https://github.com/me/lib.git
```

The Go tool maps module path → repo URL via the `go-get` HTTP discovery (or directly for github/gitlab/bitbucket). Vanity hosts (`go.uber.org/...`) need an HTML `<meta>` tag.

For modules in subdirectories of a repo:

```
module path: github.com/me/repo/api/v2
git URL:     https://github.com/me/repo.git
tag:         api/v2.0.0  (or api/v2/v2.0.0)
```

The tag's prefix encodes the subdirectory. The Go tool tries multiple tag patterns: `v2.0.0`, `api/v2.0.0`, `api/v2/v2.0.0`. The first match wins.

### Major version 0

`v0.x.x` is "unstable; expect breakage". The semver spec is more relaxed for v0; the Go convention is that any v0.x.x change is allowed.

Some libraries stay v0 forever (`v0.42.0`, `v0.43.0`, ...) as a way to signal "we don't promise compat". Others jump to v1 as soon as the API stabilizes. Both are accepted.

### Pre-1.0 import path

A module at `v0.x.x` or `v1.x.x` does NOT use a `/v0` or `/v1` suffix:

```
module github.com/me/lib       // any version 0.x.x or 1.x.x
```

Only `/v2`+ get the suffix.

### MVS and the major version

MVS resolves within a major. A direct require of `github.com/me/lib v1.2.0` and a transitive require of `github.com/me/lib v1.5.0` resolves to `v1.5.0` (MVS max).

But `github.com/me/lib` and `github.com/me/lib/v2` are **different modules**. MVS treats them independently; you can end up with both in your build list (and your binary).

```
require (
    github.com/me/lib v1.5.0
    github.com/me/lib/v2 v2.1.0
)
```

Both compile. Both link. Different Go types — `lib.Foo` ≠ `libv2.Foo`. Useful during migrations.

### `go mod tidy` and version selection

`go mod tidy` recomputes the build list, then writes the minimal `go.mod`/`go.sum`. It uses MVS; it doesn't upgrade unless forced.

`go get -u` is the explicit upgrade — bumps to latest minor/patch, then runs MVS, then writes updated `go.mod`.

### Retracted versions

A retracted version (see `06-packages-modules/04-mod-directives.md`) is still selectable but ignored by `@latest`:

```bash
$ go list -m -retracted github.com/me/lib
github.com/me/lib v1.6.0 (retracted)
$ go get github.com/me/lib@latest
# resolves to v1.5.0 if v1.6.0 is retracted
$ go get github.com/me/lib@v1.6.0
# Still possible; warning about retraction.
```

### Time-based vs explicit upgrades

`go get -u` upgrades to latest. Some teams pair with Renovate or Dependabot bots that auto-PR upgrades. Others manually audit. The semver model assumes upgrading within a major is safe; CVE-aware tools like `govulncheck` flag specific vulnerable versions for selective upgrade.

## Standard Library Hooks

- `go list -m -versions <pkg>` — all available versions.
- `go list -m -u all` — list with upgrade hints.
- `go get <pkg>@<version>` — set/upgrade.
- `go mod edit -require=<pkg>@<version>` — same, programmatically.
- `golang.org/x/mod/semver` — comparator/parser library.
- `golang.org/x/mod/module` — module path validation.
- Proxy info endpoint: `GET https://proxy.golang.org/<module>/@v/<version>.info`.
- Proxy list endpoint: `GET https://proxy.golang.org/<module>/@v/list`.

## Real-World Patterns

### 1. Compare versions programmatically

```go
package main

import (
	"fmt"

	"golang.org/x/mod/semver"
)

func main() {
	fmt.Println(semver.Compare("v1.2.3", "v1.2.10"))   // -1 (v1.2.3 < v1.2.10)
	fmt.Println(semver.Compare("v1.0.0-rc.1", "v1.0.0")) // -1
	fmt.Println(semver.IsValid("v1.2.3"))                // true
	fmt.Println(semver.MajorMinor("v1.2.3"))             // v1.2
	fmt.Println(semver.Canonical("v1"))                  // v1.0.0
}
```

`golang.org/x/mod/semver` follows the Go variant of semver (with the `v` prefix). Use it instead of regex hacks.

### 2. Publish v2 of a library (Option A)

```
github.com/me/lib/
├── go.mod              module github.com/me/lib
├── lib.go              package lib  (v1 API)
├── go.sum
└── v2/
    ├── go.mod          module github.com/me/lib/v2
    ├── lib.go          package lib  (v2 API)
    └── go.sum

# Tag both at once if cross-cutting fix:
$ git tag v1.4.5
$ git tag v2.0.1
$ git push --tags
```

### 3. Migrate from v1 to v2 in a consumer

```go
// Step 1: side-by-side
import (
    lib   "github.com/me/lib"
    libv2 "github.com/me/lib/v2"
)

// Step 2: migrate one call site at a time
// Step 3: remove the v1 import
```

The dual-import phase is the migration period. Don't try to flip everything at once on large codebases.

### 4. Force-pin to a pseudo-version

```bash
$ go get github.com/x/y@abc123def
$ cat go.mod
require github.com/x/y v0.0.0-20240115120000-abc123def456
```

Used when an upstream fix isn't tagged yet. Convert to a tagged version as soon as available.

### 5. Audit available versions

```bash
$ go list -m -versions github.com/google/uuid | tr ' ' '\n' | tail
v1.4.0
v1.5.0
v1.6.0
$ go get github.com/google/uuid@v1.6.0
$ go mod tidy
$ go test ./...
```

## Anti-Patterns & Gotchas

**Releasing v2.0.0 without changing the module path.** Consumers can't upgrade cleanly. Either follow `/v2` properly or accept `+incompatible` (legacy code only).

**Tagging "v1" or "v2" without minor/patch.** Invalid: must be `v1.0.0`, not `v1`. The Go tool will refuse.

**Pseudo-versions as a long-term pin.** Tag a release. Pseudo-versions defeat clarity, tooling, and audit.

**Pre-release tags accidentally used as "stable".** `go get @latest` skips them, but explicit pins to `-rc.1` will stick. Tag a final release after RC.

**Multiple majors used in one binary unintentionally.** Easy to do via transitive deps. Inspect with `go list -m all | grep github.com/me/lib`. May indicate diamond-dep problems.

**`/vN` in `go.mod` but not in imports.** Consumer code must use the `/vN` import path. The compiler enforces; the error is "no required module provides package..." — confusing if you don't suspect the suffix issue.

**Believing semver is enforced.** It isn't. The community trusts library authors to honor it. Breaking changes within a major are common bug reports against poorly-versioned libraries.

**Tagging `v0.x.x` indefinitely to "avoid responsibility".** Users still depend; "v0 means anything" doesn't excuse breakage. Tag v1 when you're ready to commit.

**Forgetting to push tags.** `git push` alone doesn't push tags. Use `git push --tags` or configure auto-push.

**Reusing a tag.** Don't. The Go proxy caches tags by name; replacing a tag forces consumers to bypass the cache (and detects mismatched hashes in `go.sum`).

**Mixing semver with date-based versioning.** Calver (`v2024.01.15`) is valid semver formally but breaks user expectations. Either pure semver or pure calver per the project's convention.

## Performance Notes

- `go list -m -versions`: ~ms per package on cache hit; ~hundreds of ms cold.
- `semver.Compare`: nanoseconds.
- Pseudo-version resolution: one proxy round-trip.
- Branch resolution: similar (resolves to a pseudo-version).
- MVS scales with version-count per module; rarely a bottleneck.

## How Big Companies Use It

- **Kubernetes** publishes `k8s.io/api`, `k8s.io/apimachinery`, etc. as multi-module repos with synchronized version tags: https://github.com/kubernetes/kubernetes/blob/master/staging/README.md.
- **The Go x/... repos** are pre-1.0 forever (`v0.x.x`) because they're "extensions, not stdlib promise": https://pkg.go.dev/golang.org/x.
- **gRPC-Go** is at `google.golang.org/grpc` v1; never tagged v2 despite many minor versions: https://github.com/grpc/grpc-go.
- **gocloud.dev** is at v0 for stability of its API surface, intentionally: https://gocloud.dev.
- **HashiCorp Vault SDK** uses `github.com/hashicorp/vault/api/v2` for v2 path migration: https://github.com/hashicorp/vault.
- **Uber's Zap** is at v1 since 2017 — semver-strict, tags every minor: https://github.com/uber-go/zap.
- **AWS SDK v2** uses `github.com/aws/aws-sdk-go-v2` (not `/v2` suffix; they chose a new repo): https://github.com/aws/aws-sdk-go-v2.

## Source Code References

Pinned to `go1.26`.

- Semver library: [`golang.org/x/mod/semver`](https://github.com/golang/mod/tree/master/semver).
- Module path validation: [`golang.org/x/mod/module`](https://github.com/golang/mod/blob/master/module/module.go).
- Major version suffix enforcement: [`src/cmd/go/internal/modload/import.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/import.go) — search `major`.
- Pseudo-version handling: [`src/cmd/go/internal/modfetch/pseudo.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/pseudo.go).
- `+incompatible` handling: [`src/cmd/go/internal/modfetch/coderepo.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/coderepo.go) — search `Incompatible`.
- Proxy info endpoint: [`src/cmd/go/internal/modfetch/proxy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/proxy.go).
- Module path canonical form: same module package.

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go Modules Reference — Versions": https://go.dev/ref/mod#versions.
- "Semantic Versioning 2.0.0": https://semver.org.
- Russ Cox, "Semantic versions and module paths" (vgo series): https://research.swtch.com/vgo-import.
- Russ Cox, "Going to v2 of a module": https://go.dev/blog/v2-go-modules.
- "Publishing Go Modules" (Go blog): https://go.dev/blog/publishing-go-modules.
- "Module version numbering": https://go.dev/doc/modules/version-numbers.
- "Go's compatibility promise (1.x)": https://go.dev/doc/go1compat.
- Bryan Mills, "How modules handle v2+" (talks): https://github.com/bcmills.

## Exercises / Self-Check

1. Sort these versions: `v1.0.0`, `v1.0.0-rc.1`, `v1.0.0-alpha.10`, `v1.0.0-alpha.2`, `v1.0.0-beta`. Show your reasoning.
2. Your library is at v1.5.0; you want to add a breaking change. Write the changes you'd make to `go.mod` and the repo layout for v2.0.0 (Option A — subdirectory).
3. What does `go get github.com/x/y@main` write to `go.mod`? Why is that better than a branch reference?
4. A library was tagged `v2.0.0` without changing its module path. How do you require it in a modern `go.mod`?
5. Use `golang.org/x/mod/semver` to write a CLI that takes two versions and prints "upgrade" / "downgrade" / "same".
