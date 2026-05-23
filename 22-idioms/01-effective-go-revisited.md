# Effective Go Revisited

## TL;DR

**Effective Go** (https://go.dev/doc/effective_go) is the canonical document for Go style, written by the Go team in the early 1.x era. It covers formatting, naming, control structures, functions, data, interfaces, concurrency, errors, and more. It hasn't been *substantially* updated since ~2013, so some sections feel dated against modern Go (no generics, no slog, no iter.Seq). But the **principles** — Go formatting is non-negotiable, package names are short and lowercase, errors are values, interfaces are small, accept interfaces return structs, communicate by sharing memory not the reverse — remain definitive. The single biggest gotcha: **read Effective Go AND its successors** (Google's Go style guide, the Code Review wiki, Uber's guide). Effective Go is foundational; the others fill modern gaps.

## Mental Model

```
   The Go style ecosystem:
   
   ┌──────────────────────────────────────────────────────────┐
   │ Effective Go (2009–2013, Go team)                         │
   │   - Foundational; principles, not rules                   │
   │   - Hasn't been updated for generics, slog, iter           │
   └─────────────────┬────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Google Go Style Guide (2022, public)                       │
   │   - Rule-based, modern, includes generics                  │
   └──────────────────────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Uber Go Style Guide (2018–, community)                     │
   │   - Practical, dense, full of antipatterns                 │
   └──────────────────────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Code Review Comments wiki (2014–, Go team)                  │
   │   - Single page; catalog of common review notes              │
   └──────────────────────────────────────────────────────────┘
```

`gofmt` enforces formatting. `go vet` enforces some semantics. The rest is convention.

## Syntax & Basic Usage

There's no syntax to demonstrate; this page surveys the document's recommendations and what's changed in modern Go.

## Deep Dive

### Formatting

**Rule**: run `gofmt`. Always. No discussions.

Standard layout:
- Tabs for indentation (not spaces).
- Semicolons inserted by the lexer (don't write them).
- Braces required (`if` body must be braced).
- One statement per line (no `i++; j--`).
- Imports grouped: stdlib, third-party, internal.

Modern tools that supersede:
- `gofmt` — the canonical formatter.
- `goimports` — adds/removes imports.
- `gofumpt` — stricter superset of gofmt.
- `gopls` — does both inside editors.

### Commentary

Doc comments (now backed by https://go.dev/doc/comment, the 1.19+ doc comment format) appear before declarations and start with the declared name:

```go
// Open opens the named file for reading.
func Open(name string) (*File, error)
```

For packages: a single comment before the `package` clause; conventionally in `doc.go`.

Modern: doc comments support light Markdown-like formatting (headings via `#`, links, lists). See `22-idioms/06-doc-comments.md`.

### Names

#### Package names

- Short, lowercase, single word.
- `bytes`, `bufio`, `fmt`, `httptest`.
- No underscores or mixedCaps.
- The package name is part of identifiers: `bytes.Buffer`, not `bytes.BytesBuffer` (stutter).

#### Getters

Don't prefix with `Get`. Field `Owner` has accessor `Owner()`, not `GetOwner()`.

#### Setters

Use `SetX`. `SetOwner(o)` is fine.

#### Interface names

A one-method interface ends in `-er`: `Reader`, `Writer`, `Closer`, `Stringer`. Multi-method interfaces don't follow this rule strictly (`http.Handler`).

#### MixedCaps

`HTTPServer`, not `Http_Server` or `http_server`. Acronyms stay all-caps in identifier names: `URLParser`, `userID`.

### Semicolons

Auto-inserted. The placement of `{` matters:

```go
// Compile error: semicolon inserted before {
if x > 0
{
    foo()
}

// Correct
if x > 0 {
    foo()
}
```

### Control structures

#### `if`

Initialization clause allowed:

```go
if err := f(); err != nil {
    return err
}
```

Scope `err` to the `if` body.

#### `for`

Three forms:
- `for { }` — infinite.
- `for cond { }` — while-style.
- `for init; cond; post { }` — C-style.
- `for k, v := range coll { }` — range.

There's no `while`; `for` covers it.

#### `switch`

- No `break` required; cases don't fall through.
- Use `fallthrough` keyword if you want to fall through.
- `switch` with no expression is `switch true` — useful for switch-on-condition:

```go
switch {
case x < 0:
    // ...
case x == 0:
    // ...
case x > 0:
    // ...
}
```

#### Type switch

```go
switch v := x.(type) {
case int:    // v is int
case string: // v is string
default:     // v is original interface
}
```

### Functions

#### Multiple returns

Idiomatic. `(value, error)` is the canonical pattern:

```go
func Parse(s string) (int, error) {
    // ...
}
```

#### Named returns

```go
func (f *Foo) Sum(items []int) (total int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panicked: %v", r)
        }
    }()
    for _, x := range items { total += x }
    return
}
```

Useful for documentation; can be terse and surprising in long functions.

#### Defer

Runs in LIFO order on function exit. Use for cleanup:

```go
f, _ := os.Open(name)
defer f.Close()
// ...
```

Modern Go (1.14+) has open-coded defers; defer cost is negligible for ≤8 simple defers.

### Data

#### Allocation: `new` vs `make`

- `new(T)` returns `*T` zeroed; works for any type.
- `make` only for slices, maps, channels; returns the value (not a pointer).

Use `&T{}` (struct literal) more often than `new(T)`.

#### Constructors / `NewX`

```go
type File struct { /* ... */ }

func NewFile(name string) (*File, error) {
    // ...
    return &File{...}, nil
}
```

Constructors are functions returning the new value. No `init` methods on the type.

#### Composite literals

```go
type Point struct{ X, Y int }

p := Point{1, 2}        // positional
p := Point{X: 1, Y: 2}  // named (preferred for non-trivial types)
p := Point{}            // zero
```

Slice/map literals:

```go
s := []int{1, 2, 3}
m := map[string]int{"a": 1, "b": 2}
```

#### Allocation pattern: struct vs map

When something can be a `struct` with named fields, prefer struct over `map[string]any`.

### Slices

Mostly covered separately (`03-composite-types/02-slices.md`). Key Effective Go points:

- A slice is a "view" over a backing array.
- `append` may reallocate; assign the result.
- Pass-by-value of a slice is cheap (header is 3 words).

### Maps

- Map values are not addressable: `m["key"].field = x` won't compile if value is struct.
- Map iteration is randomized; don't rely on order.

### Channels

The Effective Go advice: **"Do not communicate by sharing memory; share memory by communicating."**

In practice this is half-aspirational. Mutexes and atomics are common in real Go; the channel idiom suits some workflows but not all. See `07-concurrency/15-channels-vs-mutexes.md`.

#### Channel patterns

```go
ch := make(chan int)
go func() { ch <- 42 }()
v := <-ch
```

Goroutine spawned; communicates via channel.

#### Select

```go
select {
case v := <-ch1:
    // ...
case ch2 <- 7:
    // ...
case <-time.After(1*time.Second):
    // timeout
}
```

### Methods

#### Pointer vs value receivers

Effective Go advice (still good):
- Use pointer if the method mutates the receiver.
- Use pointer if the receiver is large.
- Use value if the type is small and immutable (like `time.Time`).
- **Be consistent** within one type: don't mix value and pointer receivers.

#### Method sets

- Methods on `T` are in the method set of both `T` and `*T`.
- Methods on `*T` are only in the method set of `*T`.

Implication: an interface requiring a pointer-receiver method can only be satisfied by `*T`, not `T`.

### Interfaces

#### Small interfaces

The Go community has settled on **"interfaces should be small"**. The standard library exemplars:

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }
```

Compose into bigger ones:

```go
type ReadWriter interface { Reader; Writer }
type ReadCloser interface { Reader; Closer }
```

#### Accept interfaces, return structs

Functions take interface arguments (for testability and flexibility); return concrete types (so the caller has full information).

```go
// Good
func Process(r io.Reader) (*Result, error)

// Less good
func Process(f *os.File) (Result, error)
```

#### Empty interface (`any`)

`any` is an alias for `interface{}` (since 1.18). Holds anything. Use sparingly; type assertions and reflection are runtime concerns.

### Concurrency

Effective Go's canonical example: a simple producer-consumer with channels. Modern Go has more options:

- `sync.WaitGroup`, `WaitGroup.Go` (1.25+).
- `errgroup` (`golang.org/x/sync/errgroup`).
- `context.Context` for cancellation.
- `sync.Once`, `OnceValue`, `OnceValues`.
- `sync.Map`, `sync.Pool`.

The advice "share memory by communicating" applies when natural. Otherwise mutex is fine.

### Errors

Effective Go's section is brief: errors are values implementing the `error` interface.

Modern Go (1.13+) added:
- `%w` verb for wrapping.
- `errors.Is`, `errors.As`, `errors.Join`.
- Structured errors via custom types.

See `05-errors/` chapters.

### What's missing in Effective Go (since 2013)

- Generics (1.18+).
- `slog` structured logging (1.21+).
- `iter.Seq` and range-over-func (1.23+).
- `context.Context` patterns (got slight update, still light).
- Module / dependency management (predates modules).
- Profiling and pprof workflows.
- Testing patterns beyond basics.
- Production-shape concerns (graceful shutdown, GOMEMLIMIT, observability).

For these, supplement with:
- Google's Go style guide.
- Uber's Go style guide.
- The release notes per Go version.
- This book :)

### Treat Effective Go as principles, not law

The Go community has evolved past some specific recommendations:
- "Don't put `if err != nil` on the same line" — many ignore.
- "Use channels for everything" — practice mixed mutex + channel.
- "Avoid named returns" — many use them tactically.

Read for principles (small interfaces, accept-interfaces-return-structs, gofmt is law); don't memorize every example.

## Standard Library Hooks

- `gofmt` — formatting.
- `goimports` — imports.
- `go vet` — basic static checks.
- `golangci-lint` — broader linting.
- `gopls` — IDE-level checks.

## Real-World Patterns

### 1. Always gofmt

```bash
$ gofmt -w .   # rewrite in place
$ gofmt -l .   # list non-conformant files
$ goimports -w . # gofmt + manage imports
```

Most editors run on save.

### 2. Tabs for code, spaces for alignment

Go code uses tabs for indentation; gofmt aligns columns with spaces. Don't fight it.

### 3. One file per type — usually

Not a hard rule. Small types are fine in shared files. Large types deserve their own file.

### 4. Test files alongside

`foo.go` + `foo_test.go` in the same package. Same package == `package foo`; external == `package foo_test`.

### 5. Package as the unit of cohesion

Each package serves one purpose. `time` does time. `bytes` does bytes. Subpackages by sub-concern, not by file.

## Anti-Patterns & Gotchas

**Skipping gofmt.** Always run it. Reviewers will demand it; CI will fail.

**Stuttering package names.** `bytes.BytesBuffer` reads as junk; `bytes.Buffer` is right.

**Get-prefixed accessors.** `GetOwner()` is non-Go. `Owner()` is right.

**Long-lived `interface{}` parameters in hot paths.** Use generics where possible.

**Mixing receiver kinds on one type.** Either all value, all pointer, or consistently mixed at known points.

**Channels-everywhere when mutex would do.** Channels are heavier than RWMutex. Use whichever fits.

**Skipping doc comments on exported identifiers.** `golint` (deprecated but accurate) flagged this; `golangci-lint` enforces.

**Custom formatting in your editor.** Don't override gofmt's choices.

**Hand-written `String()` for types that aren't fmt.Stringer.** Calling `Sprintf("%v", x)` will infinitely recurse.

**Ignoring `go vet`.** Free and helpful.

## Performance Notes

Style choices generally don't affect performance. Some do:
- `fmt.Sprintf` allocates; `strconv.AppendInt` doesn't.
- Channel vs mutex: similar at low contention; mutex faster at high.
- Generics: zero overhead after monomorphization.
- Defer (1.14+): ~5ns for open-coded; previously ~50ns.

## How Big Companies Use It

- **Google internal**: Effective Go is required reading.
- **Uber**: Uber Go Style Guide is a "+1" layer on top.
- **Most Go shops**: Effective Go + golangci-lint config.
- **Education**: every Go course starts with Effective Go.

## Source Code References

- Effective Go: https://go.dev/doc/effective_go.
- Go FAQ: https://go.dev/doc/faq.
- Code Review Comments wiki: https://go.dev/wiki/CodeReviewComments.
- Google's Go style guide: https://google.github.io/styleguide/go/.
- Uber's Go style guide: https://github.com/uber-go/guide.
- The gofmt source: [`src/cmd/gofmt/gofmt.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/gofmt/gofmt.go).
- The go/format library: [`src/go/format/format.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/format/format.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Effective Go": https://go.dev/doc/effective_go.
- "Go FAQ": https://go.dev/doc/faq.
- Dave Cheney, "Practical Go" (lecture series): https://dave.cheney.net/practical-go.
- Bill Kennedy (Ardan Labs) blog: https://www.ardanlabs.com/blog/.
- Mat Ryer's talks on idiomatic Go.
- "Go Programming Language" — Donovan & Kernighan (book).
- "The Go Programming Language Specification" — formal reference: https://go.dev/ref/spec.

## Exercises / Self-Check

1. Read Effective Go's section on Interfaces. Identify one piece of advice that's been refined or overridden by modern community practice.
2. Find a Go file in your codebase that doesn't `gofmt` clean. Why?
3. Effective Go uses channels heavily in examples. Find a place in your code where mutex would be clearer. Justify the choice.
4. Effective Go pre-dates generics. Sketch how the "interface for collections" section would change with 1.18+ in mind.
5. Read the Code Review Comments wiki. List five concrete rules not stated in Effective Go.
