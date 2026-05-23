# Google Go Style Guide

## TL;DR

In **November 2022**, Google publicly released their internal Go style guide at https://google.github.io/styleguide/go/. It's the first formally-published, modern, rule-oriented style guide from the Go team's home company. It's split into three documents: **Style Guide** (mandatory at Google), **Style Decisions** (rationale for non-obvious calls), and **Best Practices** (recommendations). Unlike Effective Go (which gives principles), this document **gives rules** — what's required, what's discouraged, with examples of each. The single biggest gotcha: **it's tuned for Google's monorepo at Google's scale**. Some rules (mandatory `init()` patterns, internal logging conventions, specific testing frameworks) reflect Google's environment more than the public Go community. Treat it as authoritative but adapt to your context.

## Mental Model

```
   Google's published style materials (2022+):
   
   ┌──────────────────────────────────────────────────────────┐
   │ Style Guide                                                │
   │   - The rules. Required at Google.                          │
   │   - Backed by automated checks where possible.              │
   └──────────────────────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Style Decisions                                              │
   │   - Explains "why" for non-obvious rules.                   │
   │   - Trade-offs that didn't make it into the Style Guide.    │
   └──────────────────────────────────────────────────────────┘
                     ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Best Practices                                                │
   │   - Patterns Google teams use; not strictly required.       │
   └──────────────────────────────────────────────────────────┘
```

The whole guide is publicly browsable; sections of interest below.

## Syntax & Basic Usage

There's no syntax; this page summarizes the Google style guide's notable rules.

## Deep Dive

### Authorship

The guide is curated by **the Go Readability program at Google** — engineers who review Go code for "readability" (a peer-review process for joining the trusted-reviewers list internally). The public release reflects 10+ years of accumulated practice on Google's internal monorepo, which contains tens of millions of LOC of Go.

Authors include Ian Lance Taylor, Bryan C. Mills, Robert Griesemer, and others on the Go team.

### Structure

The guide has three documents, each split into many sections. The high-level topics:

- Naming.
- Doc comments.
- Errors.
- Logging.
- Testing.
- Concurrency.
- Generics.
- Imports and packages.
- Style: formatting, control flow.

It's a *long* read (50+ pages printed). Browse by topic.

### Key rules (a curated selection)

#### Naming

- **Underscores discouraged** in identifiers, except `_test.go` filenames, `Test*`, `Benchmark*`, `Example*` functions.
- **Receiver names short**: `m`, `t`, `s` — 1-2 letters, not `this` or `self`.
- **Consistent receiver names** across methods on a type.
- **Don't repeat package name in symbol names**: `chess.Board`, not `chess.ChessBoard`.
- **Initialisms in identifiers**: all caps (`URLParser`, `userID`, `HTMLEncoder`).
- **Variables: short names for short lifetimes** (`i`, `err`); longer for longer-lived (`numConnections`).

#### Imports

- **Group**: stdlib, then external, then internal (own module).
- **Blank lines separate groups**.
- **Alphabetical within group**.
- **No dot imports** outside specific test patterns.
- **Aliases only when necessary**: name conflicts or path-vs-package mismatches.

#### Doc comments

- **Every exported identifier has a doc comment.**
- **Start with the identifier's name**: "Open opens a file..."
- **Complete sentences**.
- **Package comment in `doc.go`** for non-trivial packages.

#### Errors

- **Lowercase, no trailing punctuation**: `errors.New("cannot do thing")`, not `"Cannot do thing."` or `"Cannot do thing!"`.
- **No "failed to": just say what failed**: `"open file %s: %w"`, not `"failed to open file %s: %w"`.
- **Wrap with `%w`** when adding context.
- **Use sentinel errors** for cases callers programmatically inspect.
- **`errors.Is` / `errors.As`** for inspection.

#### Logging

- Google uses **`klog`** internally (a glog fork). External users should use `slog` (1.21+).
- **Structured logging** preferred over printf-style.
- **Don't log and return the same error** — pick one. Each error should be handled once.

#### Receivers

- **Pointer receivers** when:
  - The method mutates the receiver.
  - The receiver is large.
  - There's any chance of mutation in the future.
- **Value receivers** when the type is small and immutable (like `time.Time`).
- **Be consistent**: one type, one receiver kind.

#### Type assertions

- Use **comma-ok** form: `v, ok := x.(T)`.
- If you know the type for sure, single-value form is OK but rare.

#### Concurrency

- **Don't start a goroutine without knowing how it will stop.**
- **Avoid goroutine leaks** via context cancellation.
- **Use `errgroup.WithContext`** for parallel work that may fail.

#### Generics

- **Use generics when you have multiple concrete types** doing the same thing.
- **Don't use generics for `interface{}`-style wide-open types**.
- **Constraints should be expressive**: `comparable`, `Ordered`, etc.

#### Testing

- **`testing.T` not panic-driven** — fail with `t.Errorf`, `t.Fatalf`.
- **Table-driven tests** for similar cases.
- **Subtests** (`t.Run("name", ...)`) for organization.
- **`t.Helper()`** in helper functions.
- **`testing/synctest`** (1.25+) for time-dependent tests.

### Notable Google-isms

#### Internal flag culture

Google's Go has its own flag library (`google.golang.org/grpc/...`-style). External Go is mostly `flag` stdlib or `cobra/viper`. The style guide notes this but doesn't dwell.

#### Mandatory readability review

Google has a process where new engineers must have one Go change approved by a "readability reviewer" before they can `LGTM` Go code alone. The style guide is the rule book for that process.

#### `go vet` and `staticcheck` are baseline

Google requires both clean before submit. External shops should at minimum use `go vet` + `golangci-lint`.

### Things the style guide explicitly forbids

- **`init()` for non-trivial work**.
- **Goroutine-per-request without bounds**.
- **Returning the wrong-shaped error** (e.g., panic from a `(T, error)` function).
- **Mutex copy via value-receiver methods** on types that embed `sync.Mutex`.
- **`Lock()` without paired `defer Unlock()`** in most cases.
- **`recover()` outside of `defer`d functions**.
- **Channels-with-direction confusion**: prefer `chan<-` / `<-chan` in function signatures.

### Things the style guide endorses

- **Functional options** for constructors (`19-patterns/01-functional-options.md`).
- **Accept interfaces, return structs** ("19-patterns/06-clean-hexagonal.md").
- **Small interfaces** ("io.Reader, io.Writer, io.Closer").
- **Constructor pattern**: `NewX(...) *X`.
- **`context.Context` first parameter** for I/O methods.
- **`error` last return value**.
- **`gofmt`** always.

### Where Google diverges from the broader community

- **Logging**: Google uses klog (internal); community uses slog.
- **Errors**: more "production-shape" guidance (where to log; specific verbs); less wrapping than Uber.
- **Testing**: prefers stdlib `testing`; less testify/gomock.
- **Mocking**: Google uses dependency injection (wire) + interfaces + handwritten fakes; less reflection-based mocking.

### Tools that enforce

Google has internal tools matching the guide; some publicly available:

- **`gofmt`** — formatting.
- **`go vet`** — basic checks.
- **`staticcheck`** — many style and bug checks.
- **`golangci-lint`** — aggregates linters; can be configured per Google rules.
- **`go-critic`** — additional opinions.

External Google open-source projects (Kubernetes, gRPC-Go) use various of these in CI.

### The "Style Decisions" document

Each contested decision has a rationale. Example: "why no `Get` prefix?" — because the field is already named, the prefix adds noise and Go's `Method()` reads as "the noun".

Worth reading the rationale; helps when applying rules in edge cases.

### Adapting to your team

Google-scale rules don't all fit small teams:

- Mandatory readability reviews: probably overkill for <50 Go engineers.
- klog: skip; use slog.
- Internal flag libs: skip.
- Internal DI: use Wire/Fx or manual.

But broadly, the formatting/naming/comment/error rules are universal.

### Cross-reference with Uber, Code Review Comments

Most rules agree with Uber's guide and the Code Review Comments wiki. Where they diverge, the Google guide is more recent and arguably more authoritative; Uber's is more practical and dense with antipattern examples.

If you read just one: **the Google guide.** If you read two: **add the Uber guide** for antipattern coverage. If you read three: **add the Code Review Comments wiki** for compactness.

### Personal opinions vs guide

The guide doesn't say "always use named returns" or "never use them"; it explains *when* each is appropriate. Treat as a decision-support document, not a checklist.

### What the guide doesn't cover

- Architecture and project layout (cmd/, internal/, pkg/).
- Microservice patterns.
- Deployment.
- Specific frameworks (chi, gin, echo, gRPC, etc.).
- Database access patterns.
- Observability beyond logging basics.

For these, look elsewhere (this book, vendor docs, the broader community).

## Standard Library Hooks

- `gofmt`, `goimports`, `go vet`, `staticcheck`, `golangci-lint`, `gopls`.
- Adoption is generally tooling-enforced.

## Real-World Patterns

### 1. Receiver naming consistency

```go
type Server struct{ /* ... */ }
func (s *Server) Start() error { /* ... */ }
func (s *Server) Stop() error  { /* ... */ }
// All methods use `s`. Don't introduce `srv` or `this` somewhere.
```

### 2. Error messages

```go
// Bad
return errors.New("Failed to open file: " + path)
// Good
return fmt.Errorf("open %s: %w", path, err)
```

### 3. Import grouping

```go
import (
    "context"
    "fmt"

    "github.com/google/uuid"
    "golang.org/x/sync/errgroup"

    "myorg/myproj/internal/store"
)
```

Three groups, blank lines between. Alphabetical within.

### 4. Doc comments per export

```go
// Cache stores recent values for fast lookup.
//
// Cache is safe for concurrent use by multiple goroutines.
type Cache struct{ /* ... */ }

// Get returns the value for key, or nil if not present.
func (c *Cache) Get(key string) *Value { /* ... */ }
```

### 5. Lint config

```yaml
# .golangci.yml — sample for Google-style enforcement
linters:
  enable:
    - gofmt
    - goimports
    - govet
    - staticcheck
    - revive          # uses Google-style rules
    - misspell
    - errcheck
    - gocritic
```

## Anti-Patterns & Gotchas

**Memorizing rules without rationale.** Read the Style Decisions doc.

**Treating Google's logging norms (klog) as universal.** For new code, use slog.

**Cargo-culting "no underscores".** Test filenames need them (`foo_test.go`).

**Internal-vs-external mismatch.** The public guide is what's authoritative for external code.

**Skipping the readability rationale.** Some rules look weird; the rationale clarifies.

**Mixing receiver kinds.** A type with both `(s Server)` and `(s *Server)` methods is confusing.

**`Get`-prefixed accessors.** `func (u *User) GetID() string` is non-Go.

**Lowercase initialisms.** `userId` should be `userID`.

**Forgetting `context` parameter.** Long-running or I/O functions should accept `ctx context.Context`.

**Ignoring `go vet` warnings.** Cheap to fix; expensive to leave.

## Performance Notes

Style choices generally don't affect performance. Some do:
- `errors.New("...")` allocates once; cache sentinel errors.
- `fmt.Errorf("%w", err)` allocates a wrapper; minimal in normal paths.
- `slog`'s structured logging is faster than `log.Printf` in modern Go.

## How Big Companies Use It

- **Google**: enforced via readability program and internal tooling.
- **Kubernetes** (Google-incubated): broadly follows Google style.
- **gRPC-Go**: Google style, Google-style imports and comments.
- **Open source projects with Google authorship**: aligned.
- **Most other Go shops**: cite the guide as a reference, adapt per team.

## Source Code References

- Google Go style guide: https://google.github.io/styleguide/go/.
- Repo: https://github.com/google/styleguide.
- Code Review Comments wiki: https://go.dev/wiki/CodeReviewComments.
- Effective Go: https://go.dev/doc/effective_go.

## Further Reading

- "Google Go style guide" public release announcement: 2022 Go blog.
- Bryan C. Mills, "What's new in the Google Go style guide" (talks).
- "Why we wrote a style guide for Go" — Google blog.
- Robert Griesemer, Rob Pike, Ken Thompson interviews on Go design philosophy.
- "The Go Programming Language Style Guide" (Wikipedia summary).
- Damian Gryski, "go-perfbook — style and idioms": https://github.com/dgryski/go-perfbook.
- Various GopherCon talks referencing the guide.

## Exercises / Self-Check

1. Read Google's "Style Decisions" section on receivers. Apply it to a type in your codebase that has mixed receivers.
2. Find one rule in the Google guide that conflicts with Uber's. Which one would you adopt? Why?
3. Configure golangci-lint to enforce Google-style rules. Run on your codebase; review failures.
4. The guide forbids `init()` for non-trivial work. Find an `init()` in your code; argue for keeping it or refactoring.
5. Why is "lowercase error messages" the rule? What's the Style Decisions rationale?
