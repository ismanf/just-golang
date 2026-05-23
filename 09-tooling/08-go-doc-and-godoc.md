# `go doc` and `pkg.go.dev` — Documentation

## TL;DR

`go doc` prints package/symbol documentation extracted from source comments. It works **without a network** — it reads the local source under `$GOROOT` and `$GOMODCACHE`. The web equivalent is **pkg.go.dev**, which serves the same documentation rendered from the public module proxy. Both share a single rule: a doc comment is a comment immediately preceding the declared identifier with no blank line between them. Since Go 1.19, doc comments have a *minimal markdown-like syntax* (headings via lines starting with `#`, links via `[Text]` or `[Text]: URL`, code blocks via indent, lists via `-`/`*`/numbered). The legacy `godoc` web server is deprecated; for an offline browser, use `pkgsite` (the binary behind `pkg.go.dev`) via `go install golang.org/x/pkgsite/cmd/pkgsite@latest`.

## Mental Model

```
   source file
        │
        ▼
   parser preserves comments as CommentGroup attached to Decl/Spec nodes
        │
        ▼
   go/doc.NewFromFiles / go/doc/comment parses the comment as a Doc tree
        │
        ▼
   ┌──────────────────────────────────────┐
   │ go doc          (terminal renderer)   │
   │ pkg.go.dev      (HTML renderer)       │
   │ pkgsite local   (HTML renderer)       │
   │ gopls hover     (Markdown renderer)   │
   └──────────────────────────────────────┘
```

The doc comment grammar is the same everywhere; only the renderer changes.

## Syntax & Basic Usage

```bash
$ go doc                          # current package overview
$ go doc fmt                      # package's exported API
$ go doc fmt.Println              # one symbol
$ go doc -all fmt                 # everything (including unexported with -u)
$ go doc -u fmt internalFunc      # show an unexported symbol
$ go doc -src fmt.Println         # print source code of the symbol
$ go doc -short fmt               # one-line summaries
$ go doc -cmd cmd/go              # include command package's main
$ go doc encoding/json.Marshal    # path-qualified
$ go doc .                        # current pkg
```

Browse offline via pkgsite:

```bash
$ go install golang.org/x/pkgsite/cmd/pkgsite@latest
$ pkgsite -open ./                # opens http://localhost:8080 for the local module
```

## Deep Dive

### Anatomy of a doc comment

```go
// Package math provides basic constants and mathematical functions.
//
// This package does not guarantee bit-identical results across architectures.
package math
```

- A **package comment** is a comment on the `package` clause. In a multi-file package, only one file should carry it (conventionally `doc.go`).
- A **declaration comment** precedes a top-level `var`, `const`, `type`, or `func` with no blank line between.

```go
// Pi is the ratio of a circle's circumference to its diameter.
const Pi = 3.14159265358979323846
```

For a group:

```go
// Constants for time conversion.
const (
    SecondsPerMinute = 60
    MinutesPerHour   = 60
)
```

The first sentence (terminated by `.`) is used as the short summary in `go doc -short` output and in pkg.go.dev's package index.

### Doc Comments 1.19 syntax

[Russ Cox's proposal](https://go.dev/doc/comment) standardized a small superset of plain text:

**Headings** — a paragraph starting with `#` (single `#`; not `##`):

```go
// # Overview
//
// The Foo package does X.
//
// # Concurrency
//
// All exported functions are safe for concurrent use.
```

**Lists** — bullets (`-`, `*`, `+`) or numbered, with subsequent lines indented:

```go
// Supported modes:
//   - read
//   - write
//   - append
```

**Code blocks** — indented:

```go
//     out, err := json.Marshal(v)
//     if err != nil { return err }
```

**Inline links** — `[Text]` referencing a link reference, or `[Text]: URL` for the reference itself:

```go
// See [json.Marshal] for the structure of input.
//
// More at [Go blog].
//
// [Go blog]: https://go.dev/blog
```

**Auto-links** — bare URLs are linkified.

**Doc-link to symbols** — `[Name]` resolves to a symbol in the current package; `[pkg.Name]` resolves cross-package.

### `go doc` flags

```bash
$ go doc -u pkg            # include unexported names
$ go doc -all pkg          # include detailed content (examples, types' methods)
$ go doc -src pkg.Sym      # print the source
$ go doc -short pkg        # one-line summaries
$ go doc -cmd pkg          # treat main pkg as having exported main
$ go doc -c                # match case (default: ignore)
$ go doc -C dir pkg        # like 'go build -C dir'
```

Resolution order for arg `X`:

1. Current package + name X.
2. Standard library: `X`.
3. Module cache: `path/X`.

So `go doc Marshal` finds `encoding/json.Marshal` (or any other top-level `Marshal` in std).

### Examples (`ExampleFoo`)

```go
// File: example_test.go
package mypkg_test

import (
    "fmt"
    "github.com/me/mypkg"
)

func ExampleHello() {
    fmt.Println(mypkg.Hello())
    // Output: hello
}
```

`go doc -all` includes example output; pkg.go.dev renders them inline with "Run" buttons. The `// Output:` comment is checked by `go test` — if the code prints something else, the example fails.

Examples named:

- `ExampleFoo` — for function `Foo`.
- `ExampleType_Method` — for method.
- `Example_named` — labeled example for a package overview.

### `go doc` for commands

```bash
$ go doc cmd/go
```

Prints the package's documentation. For tools, conventionally that's the user-facing reference (`go help <topic>` mirrors `go doc cmd/go`'s sections).

The `-cmd` flag is needed when documenting a `main` package's exported (but uncalled) helpers.

### pkg.go.dev mechanics

[pkg.go.dev](https://pkg.go.dev/) is run by Google and fetches module zips from `proxy.golang.org`. To get a module published:

1. Tag a version (`git tag v1.0.0; git push --tags`).
2. Fetch it once via `proxy.golang.org` — `go install module@v1.0.0` from anywhere will trigger this.
3. Wait ~10 minutes; pkg.go.dev mirrors.

Once mirrored, the version is **immutable** — modules cannot be unpublished. Use `retract` (`06-packages-modules/04-mod-directives.md`) to mark bad versions.

### `pkgsite` for offline / private modules

```bash
$ go install golang.org/x/pkgsite/cmd/pkgsite@latest
$ pkgsite -http=:8080 -gopath_mode -mode local
```

Serves a local mirror of pkg.go.dev, useful for air-gapped envs or private modules (pkg.go.dev only indexes public, mirror-reachable code).

For a single project:

```bash
$ cd /my/project
$ pkgsite -open .
```

Opens a browser to your module's docs.

### Deprecated `godoc`

The old `godoc` server (binary at `golang.org/x/tools/cmd/godoc`) is unmaintained. Use `pkgsite`. The `godoc.org` website redirects to pkg.go.dev.

### Linking in doc comments

`[json.Marshal]` (with bracket but no URL) is a *doc link*; the renderer resolves it to the right page. Works for:

- `[Name]` — current package, exported symbol.
- `[Type.Method]` — method.
- `[pkg.Name]` — cross-package.
- `[pkg.Type.Method]` — cross-package method.
- `[Text](URL)` — *not* supported; use the link-reference form below.

For external links:

```go
// See the [Go blog] for more.
//
// [Go blog]: https://go.dev/blog
```

### Deprecation markers

```go
// Deprecated: Use NewThing instead.
func OldThing() {}
```

The exact form `Deprecated:` (line-leading, single space) is recognized by tools (gopls flags calls, pkg.go.dev shows a "deprecated" banner, `staticcheck` checks `SA1019`).

### `package doc.go` convention

Put package-level docs in a `doc.go` file:

```go
// Package mypkg does X.
//
// # Overview
//
// ...
package mypkg
```

Keeps it findable; survives even after the implementation files churn.

### BUG and TODO conventions

```go
// BUG(rsc): The implementation is wrong when N > 100.
```

`BUG(name)` is recognized; pkg.go.dev shows them as a separate section.

`TODO` and `XXX` are not — they're just comments.

### `go doc` and version selection

Inside a module, `go doc pkg` uses the *selected* version of `pkg` (whatever MVS picked). Outside a module, it uses the latest cached. To pin:

```bash
$ go doc go/types@v0.20.0    # not supported (no @ syntax)
```

There's no `@version` form for `go doc`. Use pkg.go.dev's URL: `https://pkg.go.dev/go/types@v0.20.0`.

## Standard Library Hooks

- `go/doc` — parse documentation from `*ast.Package`.
- `go/doc/comment` — parse and render the 1.19 comment syntax.
- `go/printer` — render code excerpts.
- `golang.org/x/pkgsite` — the pkg.go.dev server source.
- `golang.org/x/tools/cmd/godoc` — the legacy server; do not deploy new instances.
- `gopls` — uses `go/doc` for hover/completion info.

## Real-World Patterns

### 1. Local lookup

```bash
$ go doc strings.Builder
$ go doc -src strings.Builder.WriteByte
```

### 2. Look up a third-party symbol

```bash
$ go doc github.com/google/uuid.NewString
```

Reads from `$GOMODCACHE`. Run `go mod download` first if cache is cold.

### 3. Document your package

```go
// Package cache is an in-memory key-value store with TTL.
//
// # Overview
//
// Use [New] to construct a Cache. Items are sharded across N buckets
// for concurrent access.
//
// # Eviction
//
// Items expire after their TTL or when the cache exceeds its size limit.
//
// # Example
//
//     c := cache.New(cache.WithSize(1024))
//     c.Set("k", "v", time.Minute)
//     v, ok := c.Get("k")
//
// [New]: #New
package cache
```

### 4. Open local pkgsite for review

```bash
$ pkgsite -open .
```

See your docs the way pkg.go.dev will render them.

### 5. CI lint for missing docs

A common `golangci-lint` linter is `revive` or `golint` (deprecated but still bundled). Rule: every exported identifier must have a doc comment starting with the identifier name.

```go
// Bad: missing comment.
func New() *Cache { ... }

// Good: starts with the function name.
// New returns a new Cache.
func New() *Cache { ... }
```

### 6. Deprecated banner

```go
// MarshalIndent is like Marshal but applies Indent to format the output.
//
// Deprecated: Use Marshal followed by Indent.
func MarshalIndent(v any, prefix, indent string) ([]byte, error) { ... }
```

`go vet` doesn't flag this; `staticcheck`'s `SA1019` does.

## Anti-Patterns & Gotchas

**Blank line between doc comment and declaration.** Breaks the binding; the comment becomes orphaned.

```go
// This won't be the doc comment.

func Foo() {}
```

**Forgetting the identifier in the first sentence.** Convention is `// FuncName does X.` not `// Does X.` — tools like `revive` flag it.

**Using markdown that isn't in the 1.19 syntax.** Bold (`**`), italics (`_`), tables — not supported. Render as plain text.

**Putting package docs in multiple files.** Only one wins (the first `package` clause encountered by the parser; usually `doc.go`).

**Documenting unexported symbols visibly.** `go doc -u` shows them; pkg.go.dev does not. Avoid spending time on docstrings that won't reach users.

**Linking to a symbol with a typo.** `[Foo]` to a non-existent symbol renders as literal text without a warning. `gopls` catches this in hover.

**Forgetting examples have `// Output:` checked.** `ExampleX` without an `// Output:` comment is just a compile check; with one, `go test` verifies. Use it to keep examples honest.

**Confusing `BUG(name)` with `TODO`.** Only `BUG(name)` is recognized; arbitrary tags are ignored.

**Old `godoc` muscle memory.** `godoc -http=:6060` still works for some workflows but the docs it produces differ from pkg.go.dev. Migrate to `pkgsite`.

**Trusting pkg.go.dev to index private modules.** It doesn't. Run `pkgsite` locally or via internal infra.

**Doc comments containing back-tick code (Markdown style).** Not supported; use indented code blocks.

## Performance Notes

- `go doc` lookup of a single symbol: <100 ms.
- `go doc -all pkg`: <500 ms for medium packages.
- `pkgsite` first request: ~1 s (parses, caches).
- `pkgsite` subsequent: <100 ms.
- pkg.go.dev mirroring after a new tag: ~5–30 minutes.

No measurable build-time cost for doc comments — the compiler ignores them. Only `go/doc` consumers care.

## How Big Companies Use It

- **Google** maintains pkg.go.dev: https://pkg.go.dev/about.
- **Kubernetes** uses pkg.go.dev for public API docs and an internal pkgsite mirror for `k8s.io/internal/*`: https://github.com/kubernetes/kubernetes.
- **Uber** runs an internal pkgsite mirror for private modules: https://github.com/uber-go.
- **HashiCorp** publishes Terraform provider SDK docs via pkg.go.dev: https://pkg.go.dev/github.com/hashicorp/terraform-plugin-framework.
- **CockroachDB** uses pkg.go.dev for the `cockroach-go` client lib and a custom Sphinx-style site for the database itself: https://pkg.go.dev/github.com/cockroachdb/cockroach-go.
- **Tailscale** publishes `tsnet` docs on pkg.go.dev as the primary developer reference: https://pkg.go.dev/tailscale.com/tsnet.
- **The Go team** writes `doc.go` files in every `golang.org/x/*` repo as the canonical package docs: https://github.com/golang/tools.

## Source Code References

Pinned to `go1.26`.

- `go doc` command: [`src/cmd/doc`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/doc).
- `go/doc` package: [`src/go/doc`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/doc).
- `go/doc/comment` (1.19 syntax): [`src/go/doc/comment`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/doc/comment).
- pkgsite: [`golang.org/x/pkgsite`](https://github.com/golang/pkgsite).
- pkg.go.dev frontend: [`golang.org/x/pkgsite/internal/frontend`](https://github.com/golang/pkgsite/tree/master/internal/frontend).
- Legacy godoc: [`golang.org/x/tools/cmd/godoc`](https://github.com/golang/tools/tree/master/cmd/godoc).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Go Doc Comments" (Russ Cox): https://go.dev/doc/comment.
- "Command doc": https://pkg.go.dev/cmd/doc.
- "Adding modules to pkg.go.dev": https://pkg.go.dev/about.
- pkgsite README: https://github.com/golang/pkgsite.
- "Effective Go — Commentary": https://go.dev/doc/effective_go#commentary.
- Andrew Gerrand, "Godoc: documenting Go code": https://go.dev/blog/godoc (historical; covers pre-1.19 syntax).

## Exercises / Self-Check

1. Write a package with a `doc.go` file that uses headings, lists, code blocks, and a doc link. Verify the rendering via `pkgsite -open .`.
2. Add a `// Deprecated:` line to a function. Check that pkg.go.dev renders a deprecation banner (or simulate locally with pkgsite).
3. Use `go doc -src` to read the source of a stdlib function. Why is this useful before depending on undocumented behavior?
4. Write an `ExampleX` with an `// Output:` comment. Break the example. What does `go test` report?
5. Tag a new version of a public module. How long until pkg.go.dev picks it up? What URL would show v1.2.3 specifically?
