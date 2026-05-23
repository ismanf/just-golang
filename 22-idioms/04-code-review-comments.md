# Code Review Comments — the Canonical Wiki

## TL;DR

The **Go Code Review Comments** wiki page (https://go.dev/wiki/CodeReviewComments) is a compact, one-page checklist of common style and design issues that Go reviewers flag. Maintained by the Go team, it's been the de-facto checklist for "what reviewers say" since ~2014. The list complements Effective Go (principles) and the Google/Uber style guides (rules) by being **review-ready**: every entry is the size of a code-review comment. The single biggest gotcha: **it predates much of modern Go** (no generics, no slog, no context-deep idioms) and updates have slowed since the Google style guide was published in 2022. Still, the list captures common review feedback that experienced Go reviewers consistently give.

## Mental Model

```
   Code Review Comments wiki is short — read it all.
   The page lists ~30 common rules; each is a line or paragraph.
   
   Pattern of each entry:
       1. State the rule.
       2. Brief reason.
       3. Example (sometimes).
   
   What reviewers actually type in PRs:
       "see CodeReviewComments: <heading>"
   
   Common topics:
     - Errors                        - Naming
     - Doc comments                  - Receivers
     - Goroutines                    - Imports
     - Variable shadowing            - Function names
     - In-band errors                - Mixed caps
```

## Syntax & Basic Usage

Browse https://go.dev/wiki/CodeReviewComments. It's a single page, ~30 sections. Below: notable entries with examples.

## Deep Dive

### Notable entries

#### Gofmt

> Run gofmt on your code to automatically fix the majority of mechanical style issues.

Non-negotiable. CI should reject non-gofmt code.

#### Comment Sentences

Doc comments should be **full sentences** starting with the identifier's name:

```go
// Open opens the named file for reading.
func Open(name string) (*File, error)
```

#### Contexts

`context.Context` is the **first parameter** of any function that may block or take time:

```go
func Do(ctx context.Context, args ...string) error {}
```

Don't pass context as a struct field unless the struct is short-lived (per-request).

#### Copying

Copying a struct that contains a `sync.Mutex` or any uncopyable type is a bug. `go vet -copylocks` warns.

```go
type Counter struct { mu sync.Mutex; n int }

func (c Counter) Inc() { /* TYPO — pointer receiver needed */ }

// Compiler doesn't catch this; go vet does.
```

Use pointer receivers when the type has a `Mutex`.

#### Declaring Empty Slices

When declaring an empty slice, prefer:

```go
var s []int        // nil slice
```

Over:

```go
s := []int{}       // non-nil but length 0
```

Reason: easier to compare to nil for "no value". Both work in most contexts.

#### Crypto Rand

Use `crypto/rand` for any security-sensitive randomness:

```go
import "crypto/rand"

b := make([]byte, 32)
_, _ = rand.Read(b)
```

`math/rand` (now `math/rand/v2` since 1.22) is not cryptographically secure.

#### Doc Comments

All top-level exports get doc comments. The first sentence is the docstring:

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp object that can be used to match against text.
func Compile(expr string) (*Regexp, error)
```

#### Don't Panic

Don't panic for ordinary errors. Return them:

```go
// Bad
if x < 0 { panic("negative") }

// Good
if x < 0 { return fmt.Errorf("invalid: %d < 0", x) }
```

Panic only for truly unrecoverable conditions (programmer errors, init failures).

#### Error Strings

- Lowercase.
- No trailing punctuation.

```go
// Bad
errors.New("Something failed.")
errors.New("Something failed!")

// Good
errors.New("something failed")
```

Reason: error strings often appear in the middle of longer messages.

#### Examples

Add `Example_X` functions in `*_test.go`. They:
- Become testable examples.
- Appear in package documentation.
- Are verified to compile and run.

```go
func ExampleHello() {
    fmt.Println(Hello("World"))
    // Output: Hello, World!
}
```

#### Goroutine Lifetimes

> When you spawn goroutines, make it clear when — or whether — they exit.

Don't leave goroutines without a clear termination strategy. Use `context.Context` for cancellation:

```go
func (s *Server) start(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case msg := <-s.in:
                s.handle(msg)
            }
        }
    }()
}
```

#### Handle Errors

Don't discard errors with `_ =` unless you're sure it's safe:

```go
// Bad
_ = json.NewEncoder(w).Encode(v)

// Good
if err := json.NewEncoder(w).Encode(v); err != nil {
    log.Printf("encode: %v", err)
}
```

Some cases (`fmt.Println`, `w.Close()` in deferred) the error is consistently uninteresting; document and ignore explicitly.

#### Imports

Group as: stdlib, external, internal. Use `goimports` to enforce.

```go
import (
    "fmt"
    "io"

    "github.com/foo/bar"

    "myorg/internal/x"
)
```

#### Import Blank

Allowed only with a comment explaining why:

```go
import (
    _ "github.com/lib/pq" // PostgreSQL driver
)
```

#### Import Dot

Effectively forbidden outside specific test patterns. See `06-packages-modules/02-imports.md`.

#### In-Band Errors

Don't encode errors as sentinel values in a typed return:

```go
// Bad — returns -1 for "not found"
func Find(name string) int

// Good — explicit error
func Find(name string) (int, error)
```

#### Indent Error Flow

```go
// Bad — happy path indented
if err == nil {
    // ... lots of code ...
}
return err

// Good — early return
if err != nil {
    return err
}
// ... lots of code ...
```

Early returns flatten the indentation.

#### Initialisms

Acronyms keep their case throughout:

```go
URLParser   // not UrlParser
userID      // not userId
HTMLEncoder // not HtmlEncoder
```

`go vet` flags some violations; `revive` catches more.

#### Interfaces

Define interfaces where they're **consumed**, not where they're implemented.

```go
// Bad
package userstore
type User struct{ /* ... */ }
type UserStore interface { Get(id string) (User, error) }    // declared with impl

// Good
package svc
type UserStore interface { Get(id string) (User, error) }    // declared where used
```

The implementation just satisfies the interface; doesn't import it.

#### Line Length

No hard rule. ~100 characters is the de-facto threshold. Wrap long lines naturally.

#### Mixed Caps

Use `mixedCaps` or `MixedCaps`, never `snake_case` or `kebab-case`. Exception: filenames (`some_helper.go`) and tags.

#### Named Result Parameters

Use sparingly:
- Public API where they document the meaning of returns.
- `defer`-based error capture.

```go
func (f *Foo) Compute() (result int, err error) {
    defer func() {
        if r := recover(); r != nil { err = fmt.Errorf("%v", r) }
    }()
    /* ... */
    return
}
```

But don't use them for terse one-line functions.

#### Naked Returns

A `return` with no values, relying on named results, is "naked":

```go
func (f *Foo) Compute() (result int, err error) {
    result = 42
    return   // naked
}
```

OK in short functions; avoid in long ones.

#### Package Comments

Every non-trivial package should have a comment before `package X`:

```go
// Package retry implements retry policies and helpers.
package retry
```

Or in a separate `doc.go`.

#### Package Names

Short, lowercase, single word. See `06-packages-modules/01-package-fundamentals.md`.

#### Pass Values

Don't take `*T` for arguments just to avoid copying. For small types, pass by value:

```go
// Bad
func (s *Server) handle(req *Request)

// Good (if Request is small)
func (s *Server) handle(req Request)
```

For large structs or for mutation, pointers are fine.

#### Receiver Names

Short — typically the first letter of the type, lowercase:

```go
func (s *Server) Start() error { /* ... */ }
func (c *Cache) Get(key string) ([]byte, error) { /* ... */ }
```

Never `this` or `self`.

#### Receiver Type

Consistent within one type. Don't mix value and pointer receivers.

#### Synchronous Functions

Prefer synchronous functions that block on completion over async functions that take callbacks. Synchronous code is easier to test:

```go
// Prefer
func DownloadAll(urls []string) []Result

// Over
func DownloadAll(urls []string, callback func(Result))
```

Async patterns are sometimes necessary; default sync.

#### Useful Test Failures

`t.Errorf` should include actual + expected:

```go
// Bad
if got != want { t.Errorf("wrong") }

// Good
if got != want { t.Errorf("Hash(%q) = %q; want %q", input, got, want) }
```

Reading the failure should diagnose without re-running.

#### Variable Names

- Shorter for shorter lifetimes (`i`, `j` in inner loops).
- Longer for longer-lived (`numActiveConnections`).
- Receivers: 1-2 chars.
- Loop indices: `i`, `j`, `k`.

### What's not in the wiki (post-2014 topics)

- Generics (1.18+).
- `slog` patterns (1.21+).
- `iter.Seq` (1.23+).
- Module conventions.
- Goroutine leak detection (`goleak`).
- Modern observability (OpenTelemetry).

For these, consult the Google/Uber guides and this book.

### How to use it

#### In code review

> "Per CodeReviewComments § Error Strings, please lowercase and remove the trailing period."

Short references; the reviewer doesn't have to re-explain.

#### As a checklist

Before submitting a PR, scroll through the wiki entries. Most reviews catch the same handful: gofmt, error strings, comment style, receiver naming, context placement.

#### As onboarding material

New Go engineers should read it once. Re-read after a few months — different items land differently with experience.

### Variation across teams

Each team interprets some entries. Common disagreements:

- "Synchronous over async" — some teams have specific async patterns (Cadence-style).
- "Pass values" — others always use pointers for consistency.
- "Naked returns" — some ban entirely.

Adapt to your team; keep the spirit.

### Relationship to Effective Go, Google Style, Uber Style

The wiki is the *most compact*. Google Style is the *most authoritative and modern*. Uber Style is the *most example-rich*. Effective Go is the *most foundational*.

Read order:
1. Effective Go (once).
2. Code Review Comments (once; re-read periodically).
3. Google Style Guide (browse, refer often).
4. Uber Style Guide (browse, adopt selectively).

### Tooling support

- `golint`: deprecated; many rules covered by `staticcheck`, `revive`.
- `revive`: configurable; many rule names match wiki entries (`receiver-naming`, `error-strings`).
- `staticcheck`: covers many rules (e.g., `SA1029` for context.Context-as-struct-field).
- `gopls`: integrates many checks.
- `golangci-lint`: aggregates all.

A typical `.golangci.yml` enabling these gives most of the wiki's rules for free.

### Sample golangci-lint config

```yaml
linters:
  enable:
    - gofmt
    - goimports
    - govet
    - staticcheck
    - revive
    - errcheck
    - misspell
    - unused

linters-settings:
  revive:
    rules:
      - name: error-strings
      - name: error-naming
      - name: package-comments
      - name: receiver-naming
      - name: var-naming
```

## Standard Library Hooks

- `gofmt`, `goimports`, `go vet`, `staticcheck`, `revive`, `golangci-lint`.
- `context.Context` first parameter convention.
- `errors.Is`/`As`/`Join`.
- `crypto/rand` (vs `math/rand/v2`).

## Real-World Patterns

### 1. Lower-case error strings

```go
return fmt.Errorf("read config: %w", err)
```

Not `"Read config: %w"` or `"Failed to read config: %w"`.

### 2. Context first

```go
func (db *DB) QueryRow(ctx context.Context, sql string, args ...any) *Row
```

### 3. Receiver naming consistency

```go
type Foo struct{}
func (f *Foo) A() {}
func (f *Foo) B() {}
// All `f`. Don't switch to `foo` or `this`.
```

### 4. Useful failure messages

```go
if got != want {
    t.Errorf("Parse(%q) = %v; want %v", input, got, want)
}
```

### 5. Early returns

```go
func process(input string) error {
    if input == "" { return errors.New("empty") }
    if !isValid(input) { return errors.New("invalid") }
    // happy path, unindented
    return nil
}
```

## Anti-Patterns & Gotchas

**Sentinel values for errors** ("returns -1 on not found"). Use `(T, error)`.

**Uppercase error strings.**

**Long indented happy paths**. Refactor to early returns.

**`Get`-prefixed accessors.** `Owner()`, not `GetOwner()`.

**Mixing receiver naming.** Pick one per type.

**Unused returns.** `_ = ...` should be explicit when ignoring is intentional.

**Generic-sounding test failures.** "FAIL" without context.

**Adopting all rules dogmatically.** The wiki is advice; some don't fit specific contexts.

**Skipping the wiki because "I've read Effective Go".** They overlap but the wiki has review-ready items.

## Performance Notes

Style choices generally don't affect performance. The wiki focuses on readability and correctness.

## How Big Companies Use It

Universally cited in Go code reviews. Most major Go shops have CI rules covering the wiki's recommendations.

## Source Code References

- The wiki itself: https://go.dev/wiki/CodeReviewComments.
- Source repo: https://github.com/golang/go/wiki.
- Related linters: [`mgechev/revive`](https://github.com/mgechev/revive), [`dominikh/go-tools`](https://github.com/dominikh/go-tools).
- `golangci-lint`: https://github.com/golangci/golangci-lint.

## Further Reading

- Code Review Comments wiki: https://go.dev/wiki/CodeReviewComments.
- "Common Mistakes" wiki: https://go.dev/wiki/CommonMistakes.
- Effective Go: https://go.dev/doc/effective_go.
- Google Style Guide: https://google.github.io/styleguide/go/.
- Uber Style Guide: https://github.com/uber-go/guide.
- Dave Cheney, "Practical Go" lecture: https://dave.cheney.net/practical-go.

## Exercises / Self-Check

1. Read the entire wiki page in one sitting. List five rules you don't already follow.
2. Configure `revive` to enforce a subset of the wiki's rules. Run on your codebase.
3. Find one PR review where you'd cite the wiki. Type the comment you'd post.
4. Why are error strings lowercase? What's the rationale (per the wiki)?
5. Compare the wiki's "Interfaces" entry to Google's style guide. Where do they agree? Diverge?
