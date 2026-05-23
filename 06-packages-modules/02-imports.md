# Imports — Paths, Aliases, Blank, Dot

## TL;DR

An `import` declaration adds another package's exported identifiers to the current file's namespace under that package's name (not its last path segment). You can **alias** (`import foo "bar"`), do a **blank import** (`_`) to run only its `init()`s, or a **dot import** (`.`) to merge identifiers into the current package — the last is almost universally a bad idea outside of tests. `gofmt` groups imports into stdlib / external / internal blocks; `goimports` (or `gopls`) auto-adds and removes them. The single biggest gotcha: **import paths are case-sensitive and must match the canonical module path exactly** — `github.com/Org/Repo` and `github.com/org/repo` are different paths to Go (and modules require lowercase).

## Mental Model

```
   import (                              ← block form (preferred)
       "fmt"                             stdlib
       "io"
   
       "github.com/x/y"                  external module
       "github.com/x/z"
   
       "myproj/internal/store"           own module's package
   )

   import alias "github.com/x/y"         alias: refer to as `alias.Foo`
   import _ "github.com/x/y"             blank: only run init(); identifiers not added
   import . "github.com/x/y"             dot: identifiers added unqualified (avoid)

   gofmt groups imports separated by blank lines.
   goimports / gopls reorder and add/remove for you.
```

The import is resolved by `go/build` (legacy) or the modules resolver (modern) at compile time. The resolver consults `go.mod`, `go.sum`, the module cache (`$GOMODCACHE`, default `$GOPATH/pkg/mod`), and possibly a proxy (`GOPROXY`, default `https://proxy.golang.org`).

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	fmt.Println(strings.ToUpper("hi"))
	// Output: HI
}
```

Single-line:

```go
import "fmt"
```

The block form is conventional even for a single import — gofmt won't change it, but adding a second import later is cleaner.

## Deep Dive

### Import path resolution

The path in quotes is the import path, not a filesystem path:

```go
import "github.com/me/project/cache"
```

How it's resolved:

1. Look in **vendor/** if the project has a `vendor` directory and `go build -mod=vendor` is in effect.
2. Otherwise consult `go.mod`: which module does this path belong to?
3. Find the version recorded in `go.sum`.
4. Look in the **module cache** ($GOMODCACHE / $GOPATH/pkg/mod).
5. If not cached, fetch from `GOPROXY` (default proxy.golang.org), VCS, or both.

The resolution code lives in [`src/cmd/go/internal/modload`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/go/internal/modload).

### Aliases

```go
import (
	xnet "golang.org/x/net/context"   // alias: refer as `xnet`
	corev1 "k8s.io/api/core/v1"        // canonical: import path ends in v1
)
```

When to alias:

- **Name collision**: importing `crypto/rand` and `math/rand` both have package name `rand`; one must alias.
- **Mismatched package vs path**: `k8s.io/api/core/v1` package name is `v1`; `corev1` is the standard alias.
- **Disambiguation**: shortening or domain-clarifying (`pq "github.com/lib/pq"`).

When NOT to alias:

- **Stylistic preference**: `httpkit "github.com/x/httpkit"` adds noise when `httpkit` is also the package name.
- **Just because you can**: aliases become magic strings; readers must check imports to know what `xy.Foo` is.

### Blank imports

```go
import (
	"database/sql"

	_ "github.com/lib/pq"        // registers postgres driver via init()
	_ "image/png"                 // registers PNG decoder
	_ "embed"                     // enables //go:embed directives
)
```

Blank import semantics:
- Identifier `_` is the blank identifier; you can't reference the package.
- The package is *imported* — its files compile, its `init()` runs.
- Doesn't affect type checking of your file (you can't use any names).

Common uses:
- **Driver registration**: `database/sql`, `image/*`, `crypto/*`.
- **Side effects required by directives**: `//go:embed` needs `_ "embed"`.
- **Plugin-like extension**: `_ "myapp/extensions/foo"` includes feature foo's init-time registration.

Misuse: blank-importing for "just in case". Every blank import costs init time and binary bytes.

### Dot imports

```go
import . "math"
import . "github.com/onsi/ginkgo/v2"  // Ginkgo test framework
```

Dot imports merge the imported package's exported identifiers into the current package's namespace:

```go
package main

import . "fmt"

func main() {
	Println("hi")  // not fmt.Println
}
```

Pros: terse.
Cons: readers can't tell where an identifier comes from. Conflicts with future stdlib additions. Tools (gopls, vet, refactors) fight you.

The Go style guide effectively bans dot imports outside of tests. Even in tests, only Ginkgo-style DSLs really benefit; everywhere else, `assert.Equal` reads fine.

### Import path canonicalization

```go
import "github.com/MyOrg/MyRepo"   // Wrong: case-sensitive
import "github.com/myorg/myrepo"   // Right
```

The module proxy lower-cases the first three path elements (the "domain/owner/repo" prefix); import paths must use the canonical case. Mixed case worked in GOPATH mode; modules require strict matching.

Vanity import paths (`gopkg.in/...`, `go.uber.org/...`) are resolved by an HTTP `<meta>` tag served at the path — see `go help importpath`.

### Import order and grouping

Conventional grouping (enforced by `goimports`):

```go
import (
	// 1. Standard library
	"context"
	"fmt"
	"io"

	// 2. External modules (alphabetical)
	"github.com/google/uuid"
	"golang.org/x/sync/errgroup"

	// 3. Same module's internal packages
	"github.com/me/proj/internal/store"
)
```

`gofmt` (plain) preserves blank lines between groups but doesn't enforce grouping. `goimports` and `gopls` enforce it.

Three-group layout (stdlib / external / internal) is the most common. Two-group (stdlib / everything else) is also acceptable. Single-group is fine but loses scannability.

### `gopls` and auto-imports

`gopls` (the official Go LSP) auto-adds and removes imports on save:

- Type `regexp.MustCompile(...)` — `gopls` adds `import "regexp"`.
- Remove the last use of `bytes` — `gopls` removes the import.
- Sort and group on save.

Pre-`gopls` era used `goimports` separately. Modern toolchains have `gopls` integrated into VS Code, GoLand, Neovim, Zed.

### Unused imports

Go compilation **fails** on unused imports:

```
./main.go:5:8: "io" imported and not used
```

Reason: large codebases historically accumulated stale imports; making it a compile error keeps things clean. The blank import (`_ "io"`) is the official "I want this import even though I don't reference it" mechanism.

### Vanity import paths

```go
import "go.uber.org/zap"
```

Resolved by fetching `https://go.uber.org/zap?go-get=1` and parsing:

```html
<meta name="go-import" content="go.uber.org/zap git https://github.com/uber-go/zap">
```

The Go tool then clones from the git URL. Pros: stable import path independent of hosting changes. Cons: needs DNS + HTTP; corp proxies sometimes block.

The set of common vanity hosts: `golang.org/x/...` (Go team semi-official packages), `gopkg.in/...`, `go.uber.org/...`, `cloud.google.com/...`, `k8s.io/...`.

### Cycles

```
package a → imports b → imports a   // error
```

Compile error: `import cycle not allowed`. Fix by refactoring; see `06-packages-modules/01-package-fundamentals.md`.

### Test-only imports

```go
// file: x_test.go
package x

import "testing"
```

Test files can import test-only packages (`testing`, `testing/quick`, `testing/iotest`). These don't affect non-test builds.

### External test packages

```go
// file: x_test.go
package x_test    // not "package x"

import (
	"testing"
	"myproj/x"
)
```

Useful when tests should only use the public API. `x_test` is a sibling package that compiles into the same test binary as `x`.

### `import C`

```go
/*
#include <stdlib.h>
*/
import "C"
```

Special: enables cgo. `C` is a pseudo-package referring to the C namespace. See `11-low-level/05-cgo.md`. The import "C" must be alone on its line, preceded by a comment with C code.

### `embed`

```go
import _ "embed"

//go:embed banner.txt
var banner string
```

The `embed` package's existence is the side-effect signal that enables the `//go:embed` directive. Without the blank import, the compiler rejects the directive. See `08-stdlib/30-embed.md`.

### `internal/` boundary enforcement

```go
// In github.com/me/proj:
import "github.com/me/proj/internal/x"     // OK
// In github.com/other/proj:
import "github.com/me/proj/internal/x"     // compile error
```

Enforced at import resolution; see `06-packages-modules/01-package-fundamentals.md`.

### `vendor/`

If a `vendor/` directory exists with a `vendor/modules.txt`, modern Go uses it by default when:

- `-mod=vendor` is set (often the default).
- `go.mod`'s `go` directive is ≥ 1.14.

Imports resolve from `vendor/` first. See `06-packages-modules/07-vendoring.md`.

### Module cache layout

```
$GOMODCACHE/                       (default: $GOPATH/pkg/mod)
├── cache/
│   ├── download/
│   │   └── github.com/me/foo/@v/  ← zip files keyed by version
│   └── lock
└── github.com/me/foo@v1.2.3/      ← extracted sources, read-only
```

Read-only by default; `go mod tidy` and friends manage it. `go clean -modcache` clears it.

### `$GOPROXY` and module fetches

```
GOPROXY=https://proxy.golang.org,direct  (default)
GOPROXY=https://proxy.golang.org|https://athens.mycorp/,direct
```

Comma-separated proxies are tried in order; `direct` falls back to VCS. Pipe (`|`) suppresses fallback on 404/410 only — proxies that 5xx force fallback to next. See `06-packages-modules/08-private-modules-and-goproxy.md`.

## Standard Library Hooks

- `go list -e -json <pkg>` — show resolved imports, version, dir.
- `go mod why <pkg>` — explain why a package is in `go.mod`.
- `go mod graph` — full module dependency graph.
- `go env GOMODCACHE GOPROXY GOPRIVATE` — relevant env vars.
- `goimports` (`golang.org/x/tools/cmd/goimports`) — auto-add/sort.
- `gopls` — LSP server doing the same plus more.
- `go/ast.File.Imports` — programmatic access in tools.

## Real-World Patterns

### 1. Driver registration

```go
package main

import (
	"database/sql"
	"log"

	_ "github.com/jackc/pgx/v5/stdlib"  // registers "pgx" driver
)

func main() {
	db, err := sql.Open("pgx", "postgres://localhost/db")
	if err != nil {
		log.Fatal(err)
	}
	_ = db
}
```

The blank import causes `pgx`'s `init()` to call `sql.Register("pgx", &Driver{})`. `sql.Open("pgx", ...)` finds it.

### 2. Aliased imports for clarity

```go
import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

var pod = corev1.Pod{
	ObjectMeta: metav1.ObjectMeta{Name: "foo"},
}
```

Kubernetes is the canonical aliasing case — the import paths end in `v1` and `v1`, but you need to distinguish `core/v1` from `apps/v1` etc.

### 3. Side-effect import for embed

```go
package web

import (
	_ "embed"
	"net/http"
)

//go:embed assets
var assets embed.FS  // requires the import above

func ServeAssets(mux *http.ServeMux) {
	mux.Handle("/assets/", http.FileServer(http.FS(assets)))
}
```

### 4. Internal helpers re-imported externally

```go
// github.com/me/api/pkg/types/types.go
package types

type Request struct { /* ... */ }

// github.com/me/api/client/client.go
package client

import "github.com/me/api/pkg/types"

func Send(r types.Request) error { /* ... */ }
```

Move shared types up; users of `client` get `types.Request` without depending on internals.

### 5. Resolve a vanity path manually

```bash
$ curl -s 'https://go.uber.org/zap?go-get=1' | grep '<meta name="go-import"'
<meta name="go-import" content="go.uber.org/zap git https://github.com/uber-go/zap">
```

Useful for debugging proxy failures.

## Anti-Patterns & Gotchas

**Dot-importing for brevity.** Future grep is impossible. Avoid.

**Aliasing every import.** Aliases should be rare and motivated. Frequent aliasing makes code unreadable.

**Blank-importing entire packages to "be safe".** Each costs init time and binary size. Only blank-import when a side-effect is required.

**Ignoring case in import paths.** `github.com/Org/repo` and `github.com/org/repo` are different paths; the proxy canonicalizes; sometimes you hit a mismatch and `go mod tidy` fights you.

**Importing `embed` without the `//go:embed` directive.** The compiler doesn't reject it but you've added an unused package. (It will, however, complain if you have a directive without the import.)

**Forgetting `_test.go` excludes from normal builds.** Tools and scripts that import a `_test.go`-defined helper fail when run outside `go test`.

**Modifying files in the module cache.** They're read-only on purpose. `go.sum` checksums detect tampering; CI fails. Use `replace` directives or `vendor/` for local edits.

**Stale `import "C"` after removing all cgo usage.** Compile error: "C" referenced but no preceding cgo C code. Remove the import.

**Manual import sorting.** `gopls`/`goimports` do this perfectly; manual ordering generates churn in code reviews.

**`go.mod` says version X but `go.sum` is missing entries.** Run `go mod download` or `go mod tidy`.

**Importing third-party packages just for one function.** Each adds compile time and supply-chain risk. Consider copying into `internal/` (with attribution) for trivial cases.

**Forgetting that vanity paths break in air-gapped CI.** Either run a proxy (Athens) that pre-fetches everything, or vendor.

## Performance Notes

- Import resolution at build time: ~ms per module on cache hit; ~hundreds of ms per module on first fetch.
- Module cache disk usage: a typical service depends on hundreds of modules totaling 1–5 GiB extracted; the proxy zips are 50–500 MiB.
- Init() chain depth: 100+ in large projects; sub-ms aggregate if inits are well-behaved.
- Binary size impact: every imported package contributes type info + function bodies. The linker drops unreachable symbols; cold subpackages cost almost nothing.
- Blank imports cost the same as regular imports at build time; can be slightly more at runtime if their `init()` does work.

`go build -x` shows every compile + link step. `go list -deps ./... | wc -l` counts your import closure.

## How Big Companies Use It

- **Kubernetes** uses ~1000 imports per binary; aliased `corev1`, `metav1`, etc. throughout: https://github.com/kubernetes/kubernetes.
- **gRPC-Go** keeps its public API minimal; ~10 top-level packages, many blank imports for codec registration: https://github.com/grpc/grpc-go.
- **CockroachDB** has a strict import-grouping policy enforced via golangci-lint `gci`: https://github.com/cockroachdb/cockroach.
- **Uber's Go style guide** mandates import grouping: https://github.com/uber-go/guide/blob/master/style.md#import-group-ordering.
- **Cloud Native Computing Foundation projects** (Prometheus, etcd, containerd) all use 3-group import ordering.
- **Tailscale**'s `tsweb` is built around `init()`-time HTTP handler registration: blank imports of subpackages register routes on a central mux.

## Source Code References

Pinned to `go1.26`.

- Import resolution: [`src/cmd/go/internal/modload/import.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/import.go).
- Vanity path discovery (`go-get=1`): [`src/cmd/go/internal/vcs/vcs.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/vcs/vcs.go).
- Module proxy client: [`src/cmd/go/internal/modfetch/proxy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/proxy.go).
- Build constraints: [`src/go/build/constraint/expr.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/build/constraint/expr.go).
- Standard library list: [`src/go/build/syslist.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/build/syslist.go).
- `embed` package: [`src/embed/embed.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/embed/embed.go).
- `goimports` source: [`golang.org/x/tools/cmd/goimports`](https://github.com/golang/tools/tree/master/cmd/goimports).
- `gopls`: [`golang.org/x/tools/gopls`](https://github.com/golang/tools/tree/master/gopls).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "How to Write Go Code — Importing packages": https://go.dev/doc/code#ImportingPackages.
- "Customizing go get" (vanity import paths): https://go.dev/cmd/go/#hdr-Remote_import_paths.
- Russ Cox, "Go modules — go.mod, proxies, sum DB": https://research.swtch.com/vgo-tour.
- "Effective Go — Imports": https://go.dev/doc/effective_go#imports.
- Uber Go Style Guide — Import Group Ordering: https://github.com/uber-go/guide.
- `goimports` README: https://pkg.go.dev/golang.org/x/tools/cmd/goimports.
- `gopls` import organization: https://github.com/golang/tools/blob/master/gopls/doc/features/formatting.md.
- Dave Cheney, "Avoid dot imports": https://dave.cheney.net/2016/07/09/an-anthology-of-the-go-faq-anti-patterns-anti-pattern-1-dot-imports.

## Exercises / Self-Check

1. You have `import "crypto/rand"` and need `math/rand` too. How do you write both imports cleanly?
2. A package compiles fine but a blank import to it causes a panic at startup. What's the most likely cause?
3. Set up a vanity import path for `go.example.com/util` pointing at a private GitHub repo. What HTTP response does the Go tool expect at `https://go.example.com/util?go-get=1`?
4. Why is `import . "fmt"` a compile-time error inside the `fmt` package's own test files, but works fine in external `fmt_test` tests? (Hint: think about identifier clashes.)
5. Write a tool that walks a directory of Go files and lists all blank imports. Use `go/parser` and `ast.File.Imports`.
