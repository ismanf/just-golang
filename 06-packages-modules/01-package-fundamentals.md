# Package Fundamentals — Names, Exports, `init`

## TL;DR

A Go **package** is a directory of `.go` files sharing the same `package <name>` clause. Identifiers starting with an uppercase letter are **exported**; everything else is package-private. Each package is compiled as a unit; the **import path** (`github.com/foo/bar/baz`) is how callers reference it, while the **package name** (last segment, used in code) is independent. `init()` functions run once per package, in file order within a package and in dependency order across packages. The single biggest gotcha: **the package name need not match the last segment of the import path** — `import "k8s.io/api/core/v1"` is actually `package v1`, and you'll routinely write `corev1.Pod` after aliasing.

## Mental Model

```
   module: github.com/org/proj             (one go.mod here)
   ├── go.mod
   ├── main.go                              package main           (import path = github.com/org/proj)
   ├── server/
   │   ├── server.go                        package server         (import path = github.com/org/proj/server)
   │   └── server_test.go                   package server         (or package server_test)
   └── internal/
       └── store/
           └── store.go                     package store          (importable only by /github.com/org/proj/...)

   Each directory == one package. No nested packages within a single dir.
   All .go files in a dir must share the same `package X` declaration.
```

A package is the smallest unit of compilation and dependency tracking. Imports are between *packages*; references inside a package are unqualified.

## Syntax & Basic Usage

```go
// File: greet/greet.go
package greet

// Exported (capital G): callable from other packages.
func Hello(name string) string {
	return "hello, " + name
}

// Unexported (lower-case l): package-private.
func log(msg string) {
	// only callable from within `greet`
}
```

```go
// File: main.go
package main

import (
	"fmt"
	"github.com/me/example/greet"
)

func main() {
	fmt.Println(greet.Hello("world"))
	// fmt.Println(greet.log("x"))  // compile error: greet.log unexported
}
```

## Deep Dive

### Naming conventions

The Go style guide is unambiguous:

- **Package names are lower-case, single word, no underscores or mixedCaps**: `bytes`, `httptest`, `bufio`. Exceptions in the stdlib (`go/build`) are rare and historical.
- **Avoid generic names**: `util`, `common`, `helpers`, `misc` — these aggregate unrelated code and degrade discoverability.
- **Don't stutter**: `bytes.Buffer`, not `bytes.BytesBuffer`. The package name is part of the qualified identifier.
- **The name should be a noun and describe what the package provides**: `http`, `parser`, `cache`.

The full set: https://go.dev/blog/package-names and https://go.dev/doc/effective_go#package-names.

### Import path vs package name

```go
// go.mod
module github.com/me/example

// example/util/v2/v2.go
package util       // package name (used in code)
                   // import path is "github.com/me/example/util/v2"
```

The import path is the canonical address. The package name is what callers type. Mismatches require an alias:

```go
import vutil "github.com/me/example/util/v2"
```

In `go vet`'s eyes, an import alias is fine only when there's actual ambiguity. For matching names, alias is noise.

### Exported identifiers

Go's only access control is the **case rule**:

```go
package store

type Item struct {        // exported
	ID    string          // exported field
	value string          // unexported field
}

func New() *Item    { ... } // exported constructor
func (i *Item) save() error // unexported method
```

There is no `private`, `protected`, `friend`. The package is the access boundary.

Sub-rules:
- Struct fields and methods follow the same case rule.
- An exported type can have unexported fields; you'll see the type's name but not its internals.
- An unexported type can have exported fields/methods — callers in the same package can use them, but external code can't even *name* the type.
- A type can be unexported while its constructor is exported (`*store.item` is not name-able outside, but `store.New() *item` is the idiom for "opaque type with factory"). The `*item` is returned as an unexported type — callers store it via `:=`.

### `internal/` directories

A package whose import path contains the segment `internal` is importable only by packages **rooted at the parent of `internal/`**. From the spec:

> An "internal" package may only be imported by packages whose import paths begin with the path up to and including the parent of the "internal" directory.

```
github.com/me/proj/
├── internal/store/    ← importable by github.com/me/proj/...
├── server/
│   └── internal/auth/ ← importable by github.com/me/proj/server/...
└── cmd/cli/           ← can import github.com/me/proj/internal/store
```

Use `internal/` for everything that's not a published API. A common pattern: only `cmd/...` and the module root are external API; everything else lives under `internal/`.

### `init()` functions

```go
// File: cache/cache.go
package cache

import "os"

var defaultSize = 1024

func init() {
	if v := os.Getenv("CACHE_SIZE"); v != "" {
		// parse v, set defaultSize
	}
}

func init() { // multiple init() per file is legal
	// more setup
}
```

Rules:

- `init()` takes no args, returns nothing.
- A file can have multiple `init()`s; they run in source order within the file.
- Files in a package are processed in **lexical filename order** (a.go before b.go) for inits.
- Across packages, inits run in **import-DAG order**: a package's inits run *after* all transitive imports have initialized.
- `init()` runs exactly once per process per package.
- `init()` cannot be called from regular code.

Use `init()` sparingly. Common legit uses:

- Register a driver (`sql.Register("mysql", &Driver{})`).
- Validate env-derived configuration once (panic on bad config).
- Pre-compile a regex into a package-level var (but a `var re = regexp.MustCompile(...)` is often cleaner).

Anti-patterns: opening connections, doing I/O, reading config files. Inits run before `main`, so failures kill the process without a clean stack.

### Package-level variable initialization

```go
package x

var (
	a = compute()
	b = a + 1     // ordered by dependency, not file order
	c int         // zero value: 0
)

func compute() int { return 42 }
```

Rules:

- The compiler does **dependency-driven init order**: `b` depends on `a`, so `a` is initialized first regardless of source order.
- If two vars have no dependency between them, source order wins.
- All package-level vars are initialized **before any `init()` runs** in that file.
- Across files in the same package: vars in lexically-earlier filenames init before vars in later filenames (assuming no cross-file dependencies).

This is specified in [Package initialization](https://go.dev/ref/spec#Package_initialization).

### Multiple files in a package

Splitting a package across multiple files is fine and common:

```
cache/
├── cache.go        // package cache — main API
├── eviction.go     // package cache — eviction policy
└── cache_test.go   // package cache or package cache_test
```

All non-test files must share the same `package` clause. Test files can use `package <name>_test` to live "alongside" the package (external tests).

### Build constraints (file-level)

```go
//go:build linux && amd64

package mypkg

// This file is compiled only on linux/amd64.
```

The constraint must appear *above* the `package` clause, separated by a blank line. Pre-1.17 used `// +build linux,amd64`; modern code uses `//go:build`. `go fix` migrates old form. See `11-low-level/09-build-tags.md`.

Filename suffixes also imply build constraints:
- `_GOOS.go`, `_GOARCH.go`, `_GOOS_GOARCH.go` (e.g., `file_linux_amd64.go`).
- `_test.go` — test-only.

### Documentation comments

```go
// Package greet provides simple greeting functions.
//
// Hello returns a greeting suitable for printing.
package greet
```

The first sentence of the package doc is what `pkg.go.dev` and `go doc` show in indexes. Convention:

- Package doc: above `package` clause in one file (typically `doc.go`).
- Exported function/method doc: above the declaration, beginning with the function's name.
- Use complete sentences. Markdown-ish formatting since Go 1.19 (links, lists, headings — see `22-idioms/06-doc-comments.md`).

### `doc.go`

For larger packages, a dedicated file `doc.go` holds only the package comment:

```go
// Package greet provides ...
//
// # Usage
//
// Call greet.Hello with a name:
//
//	greet.Hello("world")
package greet
```

No code, just documentation. Clean separation.

### `main` package

`package main` is special:

- Produces an executable (rather than a library).
- Must contain `func main() {}`.
- Import path is the path of the `main` package itself (e.g., `github.com/me/proj/cmd/server`).
- A module can have many `main` packages under `cmd/`, each a separate binary.

### Compilation model

Each package is compiled once per build into an object archive (`.a`). The compiler emits:

- Type information.
- Function bodies (with inlining metadata if applicable).
- A symbol table.
- GC bitmaps for types and globals.

`go build` reuses the build cache at `$GOCACHE` (default `~/.cache/go-build`). Re-compilation happens only when source or build flags change. See `12-runtime/10-compiler-pipeline.md`.

### Circular imports

Forbidden. The Go compiler will refuse a build with:

```
import cycle not allowed:
  package a imports b which imports a
```

Common workarounds:

- Move shared definitions into a third package.
- Use interfaces in the upstream package; satisfy in downstream.
- Restructure: cyclic imports usually indicate a layering problem.

### Identifier visibility nuances

- A type can be **unexported but contain exported fields**: the package returns it as an opaque value (`func New() unexportedType`), but in-package code can read fields. External code can use the value via the exported interface returned by `New`, but can't write field literals.
- `_` is a special blank identifier; can be used as a name to mean "unused" but cannot be referenced.

### Reserved names

Avoid `Error` as a field (it shadows the `error` interface name in error messages). Avoid `String` as a non-`fmt.Stringer` method (causes infinite recursion in `fmt.Sprintf("%v", x)` if `String()` calls Sprintf).

### Package paths in the stdlib

Standard library packages are imported without a hostname:

```go
import "fmt"             // stdlib
import "encoding/json"   // stdlib
import "github.com/x/y"  // module
```

Anything without a `.` in the first path element is treated as stdlib. The compiler maintains a fixed list (`go list std`).

## Standard Library Hooks

- `go list -e -json <pkg>` — JSON metadata about a package.
- `go doc <pkg>` — print package documentation.
- `go doc <pkg> <ident>` — documentation for one identifier.
- `go vet ./...` — static analysis (catches some package-design issues).
- `golang.org/x/tools/cmd/godoc` — local doc server (legacy; modern is `pkg.go.dev`).
- `go/build`, `go/ast`, `go/types` — tools-side analysis.
- `runtime/debug.ReadBuildInfo` — modules linked into the binary at runtime.
- `go test -run` — runs tests within a single package.

## Real-World Patterns

### 1. The `internal/` boundary

```
github.com/me/api/
├── README.md
├── go.mod
├── api.go                   ← exports: Client, NewClient
├── internal/
│   ├── transport/
│   │   └── transport.go     ← private; not importable externally
│   └── auth/
│       └── auth.go
└── cmd/cli/
    └── main.go              ← can import internal/...
```

Anything you don't want third parties depending on goes under `internal/`. Refactoring `internal/...` is then "your business" — no compatibility guarantee.

### 2. `doc.go` for package documentation

```go
// Package retry implements exponential-backoff retry helpers.
//
// # Basics
//
// Use [Do] for a typical retry loop:
//
//	err := retry.Do(ctx, func() error { return work() })
//
// # Tuning
//
// The default policy backs off 50ms, 100ms, 200ms, ...
// Override with [Policy].
package retry
```

No code; only docs. `pkg.go.dev` renders this verbatim.

### 3. Package-private opaque types

```go
package store

type session struct {
	id string
}

// New returns an opaque session value. External callers can pass it
// back to other store functions but can't construct or inspect it.
func New() *session { return &session{id: "..."} }

// Use takes a session previously returned by New.
func Use(s *session) string { return s.id }
```

External users see:

```go
s := store.New()
store.Use(s)
// Cannot: store.session{} — type name is unexported.
```

Useful when you want a handle type without exposing its representation.

### 4. `init()` for driver registration

```go
package main

import (
	"database/sql"

	_ "github.com/lib/pq"   // pq's init() calls sql.Register("postgres", ...)
)

func main() {
	db, _ := sql.Open("postgres", "...")
	_ = db
}
```

The blank import (`_`) is the canonical idiom for "I want this package's init() to run but don't reference its API". See `06-packages-modules/02-imports.md`.

### 5. Multiple binaries under one module

```
github.com/me/proj/
├── go.mod
├── internal/...
├── cmd/
│   ├── server/main.go    ← go build ./cmd/server
│   ├── cli/main.go       ← go build ./cmd/cli
│   └── migrate/main.go   ← go build ./cmd/migrate
└── pkg/...               ← libraries
```

Each `cmd/x/main.go` is its own `main` package. Shared code lives in `internal/` or under the module root.

## Anti-Patterns & Gotchas

**Naming packages `util`, `common`, `helpers`.** Becomes a dumping ground. Split by purpose: `slogutil`, `httputil`, `idutil`.

**Using `init()` for I/O.** Failures kill the process pre-`main`. Move to lazy initialization or explicit setup functions.

**Cyclic imports.** Usually a sign two packages should be one, or that a third "core" package should hold shared types.

**Exporting fields you don't want callers to depend on.** Once exported, the type is part of your API.

**Package-level mutable state.** Globals are convenient but make testing hard and concurrency dangerous. Prefer constructor-returned state.

**`package util` with `Util*` functions** — stutter. `func util.UtilFormat` is bad; `func util.Format` is fine, but `util.` is still uninformative. Be specific.

**`init()` that depends on init order across packages.** Within a package, file order is well-defined; across packages it's the import DAG. Anything more subtle is a bug waiting.

**Multiple package names in one directory.** Forbidden (except `_test` external tests). The compiler errors.

**Naming a file `*_test.go` when it has runtime code.** It'll be excluded from builds and only included in `go test`. Surprises.

**Forgetting `//go:build` constraints when adding OS-specific code.** Will compile on all OSes and fail at link or runtime.

**Putting all code in `package main`.** Makes nothing reusable, nothing testable. Even single-binary projects benefit from one library package + a thin `main`.

**Exporting a type *just* for tests.** Use `export_test.go` to expose internals to *internal* tests without changing the public API:

```go
// File: pkg/export_test.go (compiled only for tests)
package pkg

var Internal = internal
```

## Performance Notes

- Package compilation cost is roughly linear in source size; ~10 ms per 10k LOC.
- `init()` overhead is per-package, run once. Each `init` typically <µs unless it does real work.
- Package-level variables are initialized lazily *within their package's init* but eagerly relative to `main`.
- Cross-package optimization (inlining, escape analysis) requires the callee's export data — the smaller and simpler your exported API, the better the compiler can reason.
- Excessive package count slows builds linearly. The Go team rebuilds Kubernetes (~6M LOC) in minutes; thousands of trivial packages would slow it more than fewer larger ones.
- `_ "github.com/foo/bar"` (blank import) still costs the import's init() + any package-level allocations. Don't blank-import casually.

## How Big Companies Use It

- **Kubernetes** uses `internal/` heavily and aggressive `pkg/`-vs-`cmd/` separation: https://github.com/kubernetes/kubernetes.
- **Google's Go style guide** mandates package naming conventions: https://google.github.io/styleguide/go/decisions#package-names.
- **Uber's Go style guide** has additional rules around package layout: https://github.com/uber-go/guide.
- **gRPC-Go** exposes a minimal top-level API and stuffs internals under `internal/`: https://github.com/grpc/grpc-go.
- **CockroachDB** publishes `cockroachdb/cockroach` with hundreds of packages; their layout is a teaching artifact for large Go projects: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses `internal/` and `doc.go` files in every package: https://github.com/tailscale/tailscale.
- **HashiCorp Vault** has `vault/` (private internals) + `api/` (public client lib) + `command/` (CLI): https://github.com/hashicorp/vault.

## Source Code References

Pinned to `go1.26`.

- Package init order (spec): https://go.dev/ref/spec#Package_initialization.
- `internal/` rule (spec/tools): [`src/cmd/go/internal/modload/import.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/import.go) — search `internal`.
- Build packaging: [`src/go/build/build.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/build/build.go).
- Compiler entry point: [`src/cmd/compile/internal/gc/main.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/compile/internal/gc/main.go).
- Stdlib package list: [`src/go/build/syslist.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/build/syslist.go) + `go list std`.
- Build constraints: [`src/go/build/constraint/expr.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/build/constraint/expr.go).
- Doc comment parser: [`src/go/doc/comment`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/doc/comment).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Effective Go — Package names": https://go.dev/doc/effective_go#package-names.
- "Style guideline for Go packages" (Dave Cheney): https://rakyll.org/style-packages/.
- "Standard package layout" — golang-standards/project-layout (controversial): https://github.com/golang-standards/project-layout.
- "Package design: cohesive abstraction" — Bill Kennedy (Ardan Labs): https://www.ardanlabs.com/blog/2017/02/package-oriented-design.html.
- "Internal packages" — Go 1.4 design: https://go.googlesource.com/proposal/+/master/design/12063-go-list-test.md.
- Google Go style guide — Package naming: https://google.github.io/styleguide/go/decisions#package-names.
- Russ Cox, "Go modules — design": https://research.swtch.com/vgo-intro.
- "Doc comments" (Go 1.19 update): https://go.dev/doc/comment.

## Exercises / Self-Check

1. A package's import path is `github.com/me/api/internal/store/v2`. Who can import it, and what's its package name likely to be?
2. You have `var x = f()` and `func f() int { return y }` and `var y = 10` in the same file. What's the initialization order?
3. Why is `init()` not callable from regular code? What problem would arise if it were?
4. Sketch a directory layout for a project with three binaries (server, CLI, migrator) sharing common storage and auth packages, all hidden from external users.
5. Construct a minimal example where `package main` and a library package have a circular import dependency. Fix it by introducing a third package.
