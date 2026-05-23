# Doc Comments — Markdown-ish Since 1.19

## TL;DR

Go's documentation system has always been **doc comments — plain comments immediately before declarations**. Since **Go 1.19**, doc comments support a curated subset of Markdown-like syntax: `# Headings`, `[doc links]`, code blocks, lists, links to other identifiers via `[Type.Method]`. The format is documented at https://go.dev/doc/comment. `gofmt` since 1.19 reformats doc comments into canonical form. The single biggest gotcha: **the format isn't full Markdown**. No bold or italic, no tables, no inline HTML. The constraints exist to keep `go doc` output clean across terminals while letting `pkg.go.dev` render richly.

## Mental Model

```
   Three layers of doc:
   
   1. Package doc — once per package, in `doc.go` or above any package clause.
   2. Identifier doc — above each exported type, function, method, var, const.
   3. Inline comments — anywhere; not part of the doc system.
   
   Rendering:
   
       go doc pkg              → terminal output (plain)
       go doc pkg.Identifier   → focused on one symbol
       godoc -http=:6060       → local web server (legacy)
       pkg.go.dev              → public; rich Markdown-like rendering
```

The same source text powers terminal and web rendering.

## Syntax & Basic Usage

```go
// Package cache provides an in-memory key-value cache with
// optional expiration and size limits.
//
// # Basic usage
//
// Create a cache with [New] and call [Cache.Get] / [Cache.Set]:
//
//	c := cache.New(cache.WithMaxEntries(1000))
//	c.Set("key", "value")
//	v, ok := c.Get("key")
//
// # Concurrency
//
// All [Cache] methods are safe for concurrent use.
package cache

// Cache stores recent values with optional expiration.
//
// Cache is safe for concurrent use by multiple goroutines.
type Cache struct{ /* ... */ }

// New returns an empty cache configured by opts.
// See [Option] for available configuration.
func New(opts ...Option) *Cache { return &Cache{} }
```

`pkg.go.dev/<module>/cache` renders this with headings, code blocks, and identifier links. `go doc cache` shows a plain-text version.

## Deep Dive

### History

- Doc comments existed from Go 1.0. `gofmt` did light formatting; rendering was plain.
- **Go 1.19** (Aug 2022) introduced **structured doc comments**: a documented format with headings, lists, links, code blocks. Russ Cox led the proposal. See https://go.dev/blog/godoc-markdown.
- **gofmt 1.19+** rewrites doc comments to canonical form: blank lines between paragraphs, normalized indentation.
- **pkg.go.dev** renders 1.19+ comments richly; older terminals see plain text.

### What's supported

#### Paragraphs

Blank-line-separated. Wrapped naturally for terminal output; rendered with `<p>` on web.

```go
// A paragraph.
//
// A second paragraph after a blank line.
```

#### Headings

`#` followed by space and text. Section heading.

```go
// # Usage
//
// Call Foo to bar.
```

`go doc` displays as plain heading; `pkg.go.dev` renders as `<h3>`.

Only `#` (one level); no `##` or `###` for sub-headings.

#### Doc links

Cross-references to other identifiers:

```go
// See [Cache.Get] for retrieving values.
// See [io.Reader] for the interface.
// See [github.com/example/foo.Bar] for fully-qualified.
```

Renderers resolve these to actual links on pkg.go.dev. `go doc` shows them as-is.

#### URL links

Plain URLs are auto-linked:

```go
// See https://example.com for details.
```

Or with link text:

```go
// See [the documentation] for details.
//
// [the documentation]: https://example.com
```

The second form is renderer-specific; the simple URL form works everywhere.

#### Code blocks

Indented by a tab or four spaces:

```go
// Example:
//
//	c := cache.New()
//	c.Set("k", "v")
```

`gofmt` 1.19+ uses tabs by convention. The block must follow a blank-line preceding line.

#### Lists

Numbered or bulleted:

```go
// Steps:
//
//  1. First step.
//  2. Second step.
//
// Items:
//
//   - Alpha
//   - Beta
```

Indentation must be consistent within the list.

### What's NOT supported

- **Bold** (`**word**`).
- **Italic** (`*word*`).
- **Inline code** (`` `code` ``) — code blocks only.
- **Tables**.
- **Inline HTML**.
- **Images**.

The format is intentionally restricted to be terminal-renderable.

### Canonical format via gofmt

```go
//   Originally:
//   ===========
//
// This is a    poorly-formatted doc comment.

//   After gofmt 1.19+:
//
// This is a poorly-formatted doc comment.
```

`gofmt` normalizes:
- Trims trailing whitespace.
- Ensures blank lines between paragraphs.
- Reformats lists to canonical indentation.
- Removes extra spaces.

Don't fight it; run `gofmt`.

### Package doc

The doc comment that immediately precedes a `package` declaration is the **package doc**. Convention: put it in `doc.go`:

```go
// Package retry implements exponential-backoff retry helpers.
//
// # Basics
//
// Use [Do] for a simple retry loop:
//
//	err := retry.Do(ctx, func() error { return work() })
//
// # Policies
//
// Configure backoff via [Policy].
package retry
```

`doc.go` contains only the doc and the `package` clause. No code. Tests live elsewhere.

Multi-file packages: only one file should have the package doc. `gofmt` doesn't enforce this; the team should.

### Identifier doc

Every **exported** identifier should have a doc comment. The first sentence should begin with the identifier's name:

```go
// Cache stores recent values.
type Cache struct{ /* ... */ }

// Open opens the named file for reading.
func Open(name string) (*File, error) { /* ... */ }

// MaxIters is the maximum allowed iteration count.
const MaxIters = 100
```

The first sentence (up to the first period followed by whitespace) is the **synopsis**. Used in `go doc` summaries and pkg.go.dev's package overview.

### `_test.go` examples become docs

`ExampleX` test functions appear in package documentation:

```go
func ExampleHello() {
    fmt.Println(Hello("World"))
    // Output: Hello, World!
}
```

`go test` verifies the output. `pkg.go.dev` shows it under `Hello`'s doc.

For multi-line examples or examples per method:

```go
func ExampleCache_Get() { /* ... */ }     // method example
func ExampleNew()        { /* ... */ }     // function example
func Example()           { /* ... */ }     // top-level example
```

Naming: `Example`, `ExampleX`, `ExampleType_Method`, `ExampleType_Method_named`.

### `go doc` command

```bash
$ go doc fmt              # package summary
$ go doc fmt.Println      # one identifier
$ go doc -all fmt         # all identifiers, all docs
$ go doc -short fmt       # just synopses
$ go doc -src fmt.Println # source of the function
```

Useful for command-line exploration.

### pkg.go.dev

The canonical web rendering. Auto-indexes every published Go module. Search by module path:

```
https://pkg.go.dev/github.com/me/myproj/mypkg
```

Doc comments render with:
- Headings as `<h2>/h3`.
- Code blocks with syntax highlighting.
- Doc links as hyperlinks.
- Examples in expandable boxes.

Anyone consuming your library reads pkg.go.dev. Write for it.

### Conventions

#### Synopsis

The first sentence should be a *concise summary*:

```go
// Open opens the named file.
```

Not:

```go
// This is a function that takes a filename as a string and returns a *File and an error...
```

#### Imperative voice for functions

> "Read reads ..."
> "Parse parses ..."
> "Compile compiles ..."

#### Cross-references

Use doc links liberally:

```go
// Get returns the value for key, or [ErrNotFound] if missing.
// See [Cache.Set] for insertion.
```

#### Code blocks for usage

Functions whose usage isn't obvious benefit from a code block:

```go
// Process runs the work and returns the result.
//
// Example:
//
//	result, err := Process(ctx, input)
//	if err != nil {
//	    return err
//	}
//	use(result)
func Process(ctx context.Context, input []byte) (Result, error)
```

### Linting doc comments

- `revive` rule `package-comments` checks for package doc.
- `revive` `exported` flags missing doc on exported.
- `staticcheck` `ST1000` (package doc), `ST1020` (exported doc).
- `golint` (deprecated; folded into revive).

Configure in `.golangci.yml`:

```yaml
linters-settings:
  revive:
    rules:
      - name: package-comments
      - name: exported
```

### When NOT to write a doc comment

Internal (unexported) identifiers don't strictly need doc comments. Use judgment:
- Self-explanatory names + small scope: skip.
- Subtle behavior, gotchas: explain.

Over-documentation reads badly:

```go
// i is the iterator.
i := 0
```

Skip. The variable name and context are clear.

### `// Deprecated:` markers

Used by `pkg.go.dev` and tools to flag deprecated APIs:

```go
// Foo does something.
//
// Deprecated: Use [Bar] instead.
func Foo() {}
```

`gopls` and `golangci-lint` flag callers of deprecated APIs.

### Build constraints

Build constraints come **before** doc comments and are separated:

```go
//go:build linux

// Package mylinuxpkg provides Linux-specific helpers.
package mylinuxpkg
```

Blank line between `//go:build` and the doc comment. `//go:build` is not a doc.

### Multiple paragraphs

Use blank lines:

```go
// Open opens the named file for reading.
//
// On success, Open returns a *File whose Close method must be called
// to release resources.
//
// If the file does not exist, Open returns [ErrNotExist].
func Open(name string) (*File, error)
```

### Code samples that compile

If a code block in a doc comment looks like Go code, ideally it should compile. Make it an `ExampleX` function elsewhere, which guarantees compilation:

```go
func ExampleOpen() {
    f, err := os.Open("/etc/hosts")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer f.Close()
    // ...
}
```

The example appears under `Open`'s doc on pkg.go.dev.

### Migrating older doc comments

Pre-1.19 comments often used ad-hoc formatting (asterisks for emphasis, weird indentation). Running `gofmt` on 1.19+ rewrites them to canonical form.

Backward compatibility: pre-1.19 readers see the original text; 1.19+ renderers parse the new structure. No code breaks.

## Standard Library Hooks

- `go doc` command: terminal documentation.
- `pkg.go.dev`: web rendering.
- `go/doc/comment` package (1.19+): programmatic parsing/rendering.
- `gofmt`: canonical formatting.
- `golangci-lint`'s rules: linting.

## Real-World Patterns

### 1. Package doc in `doc.go`

```go
// doc.go
//
// Package retry implements exponential-backoff retry helpers.
//
// # Quick start
//
// Use [Do] for the simple case:
//
//	err := retry.Do(ctx, func() error { return work() })
//
// # Customization
//
// Provide a [Policy] for custom backoff.
package retry
```

### 2. Exported type with cross-link

```go
// Policy configures retry behavior.
//
// See [DefaultPolicy] for sensible defaults; [WithBackoff] to customize.
type Policy struct { /* ... */ }
```

### 3. Constructor with example

```go
// New returns a configured [Cache].
//
// Example:
//
//	c := cache.New(cache.WithMaxEntries(1000), cache.WithTTL(5*time.Minute))
//	defer c.Close()
func New(opts ...Option) *Cache { /* ... */ }
```

### 4. Deprecation marker

```go
// OldAPI returns the legacy result.
//
// Deprecated: Use [NewAPI] for the modern interface.
func OldAPI() Result { /* ... */ }
```

### 5. Example function as documentation

```go
func Example_basic() {
    c := cache.New()
    c.Set("hello", "world")
    v, _ := c.Get("hello")
    fmt.Println(v)
    // Output: world
}
```

Tested + documented.

## Anti-Patterns & Gotchas

**Missing doc on exported identifiers.** Common review complaint.

**First sentence doesn't start with the identifier.** Tools and humans expect it.

**Long-winded synopsis.** Shorter is better; details below.

**Markdown sytnax assumed to work.** `**bold**` and `*italic*` render as `*bold*` literally.

**Tables.** Not supported.

**HTML.** Not supported.

**Code block without tab/4-space indent.** Renders as a paragraph.

**`// Deprecated:` lowercase.** Should be exactly `Deprecated:` capitalized.

**Doc on internal functions in shared library.** Bloats the docs; users don't see them anyway.

**Trailing whitespace.** `gofmt` cleans.

**Lots of `*X*` to fake emphasis.** Renders as literal asterisks.

## Performance Notes

Doc comments are compile-time; zero runtime impact. `go doc` parses them once.

## How Big Companies Use It

- **Standard library**: every export documented.
- **gRPC-Go, Kubernetes, CockroachDB**: enforced via CI.
- **Public open source**: missing docs → bad reputation.
- **Internal company code**: variable; some enforce, some don't.

## Source Code References

- Doc comment spec: https://go.dev/doc/comment.
- `go/doc/comment`: [`src/go/doc/comment/`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/doc/comment).
- `gofmt`: [`src/cmd/gofmt/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/gofmt).
- `go doc`: [`src/cmd/doc/`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/doc).
- pkg.go.dev: https://pkg.go.dev.
- `revive` rules: https://github.com/mgechev/revive.
- `staticcheck` ST1000 family: https://staticcheck.dev/docs/checks.

## Further Reading

- "Go Doc Comments" (official): https://go.dev/doc/comment.
- "Markdown-style godoc" Go blog (1.19): https://go.dev/blog/godoc-markdown.
- "Effective Go § Commentary".
- "Writing Go documentation": various community write-ups.
- pkg.go.dev itself: examples in action.
- "go doc" command reference.

## Exercises / Self-Check

1. Add structured doc comments to a small Go package. View on pkg.go.dev locally via `gopls` or `godoc -http=:6060`.
2. Find an exported function in your codebase missing doc. Write a comment that starts with the function name and includes a code example.
3. Use `[Type.Method]` doc-link syntax to cross-reference within a package. Verify it renders as a link on pkg.go.dev.
4. Try `gofmt`-ing an old doc comment with unconventional formatting. Compare before/after.
5. Mark a function `Deprecated:` and run a project that calls it. Does `gopls` warn you?
