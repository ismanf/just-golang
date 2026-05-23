# Builder Pattern

## TL;DR

The **builder pattern** constructs a complex object in steps, returning a value (or pointer) at the end. In Go it appears in two main flavors: **fluent builders** (`b.WithX(...).WithY(...).Build()`) and **method-chaining configurators** (`req.Header.Set(...); req.URL = ...`). Go's idiomatic preference is usually **functional options** (see `19-patterns/01-functional-options.md`) — but builders win when the construction is **multi-step**, **stateful**, or **needs to support both reads and writes during construction**. The single biggest gotcha: **Go has no method overloading**, so builders can't naturally accept variable types per call; either every step has a unique method name, or you genericize, or you fall back to options.

## Mental Model

```
   Pattern shape:
   
   result := NewBuilder().              ← create
               WithA(1).                ← step
               WithB("hi").             ← step
               WithC(true).             ← step
               Build()                  ← finalize → result

   Each step returns the builder (or a new builder) so you can chain.
   Build() performs validation, returns the constructed object,
   and (often) makes the builder unusable.
```

In Java/C# this pattern is universal. In Go it's used selectively — typically when the object can be constructed in many sequences, or when the build process involves side effects (file I/O, validation).

## Syntax & Basic Usage

```go
package query

import (
	"errors"
	"fmt"
	"strings"
)

type Query struct {
	table   string
	columns []string
	where   []string
	orderBy string
	limit   int
}

type QueryBuilder struct {
	q   *Query
	err error
}

func NewQueryBuilder(table string) *QueryBuilder {
	return &QueryBuilder{q: &Query{table: table}}
}

func (b *QueryBuilder) Select(cols ...string) *QueryBuilder {
	if b.err != nil { return b }
	if len(cols) == 0 {
		b.err = errors.New("at least one column required")
		return b
	}
	b.q.columns = cols
	return b
}

func (b *QueryBuilder) Where(cond string) *QueryBuilder {
	if b.err != nil { return b }
	b.q.where = append(b.q.where, cond)
	return b
}

func (b *QueryBuilder) Limit(n int) *QueryBuilder {
	if b.err != nil { return b }
	if n < 0 {
		b.err = fmt.Errorf("limit must be ≥0, got %d", n)
		return b
	}
	b.q.limit = n
	return b
}

func (b *QueryBuilder) Build() (string, error) {
	if b.err != nil { return "", b.err }
	if len(b.q.columns) == 0 { return "", errors.New("Select required") }

	sb := strings.Builder{}
	sb.WriteString("SELECT " + strings.Join(b.q.columns, ", "))
	sb.WriteString(" FROM " + b.q.table)
	if len(b.q.where) > 0 {
		sb.WriteString(" WHERE " + strings.Join(b.q.where, " AND "))
	}
	if b.q.limit > 0 {
		sb.WriteString(fmt.Sprintf(" LIMIT %d", b.q.limit))
	}
	return sb.String(), nil
}

func main() {
	sql, err := NewQueryBuilder("users").
		Select("id", "name").
		Where("active = true").
		Where("age > 21").
		Limit(50).
		Build()
	if err != nil { panic(err) }
	fmt.Println(sql)
	// Output: SELECT id, name FROM users WHERE active = true AND age > 21 LIMIT 50
}
```

Errors accumulate in `b.err`; later calls short-circuit. `Build()` returns the first.

## Deep Dive

### Fluent vs imperative builders

**Fluent**: methods return the builder, allow chaining.
```go
qb.Select(...).Where(...).Build()
```

**Imperative**: methods return `void`/nothing; builder is built piece by piece.
```go
qb := New()
qb.Select(...)
qb.Where(...)
result := qb.Build()
```

Go programs sometimes use imperative when a step's return value is something else (an error, a sub-builder). Fluent is more popular when the call chain is short.

### Error handling in fluent builders

Two common approaches:

#### Errors accumulate on the builder

```go
type Builder struct { err error /* ... */ }

func (b *Builder) X() *Builder {
    if b.err != nil { return b }
    // do work
    return b
}

func (b *Builder) Build() (Result, error) {
    if b.err != nil { return Result{}, b.err }
    return b.actuallyBuild()
}
```

Used above. Pro: caller can chain without per-step error checks. Con: errors are silent until `Build()`.

#### Each step returns `(T, error)`

```go
b, err := New().Select("id")
if err != nil { return err }
b, err = b.Where("active")
if err != nil { return err }
```

Pro: errors visible immediately. Con: chaining becomes ugly; might as well not be fluent.

Go community generally prefers **accumulating errors**, as long as `Build()` is the only place errors materialize.

### Validation in `Build()`

```go
func (b *Builder) Build() (Result, error) {
    if b.err != nil { return Result{}, b.err }
    if b.q.table == "" { return Result{}, errors.New("table required") }
    if b.q.limit > 1000 { return Result{}, errors.New("limit too high") }
    return b.q, nil
}
```

Cross-field validation lives in `Build()` because individual steps don't have full context.

### One-shot vs reusable builders

```go
// One-shot: Build() consumes the builder.
func (b *Builder) Build() (Result, error) {
    if b.consumed { return Result{}, errors.New("builder already consumed") }
    b.consumed = true
    // ...
}

// Reusable: Build() can be called multiple times.
func (b *Builder) Build() (Result, error) {
    return Result{...}, nil  // doesn't touch b
}
```

Reusable lets you set partial defaults and reuse the builder. One-shot lets you safely use mutable internals.

Most Go builders are reusable; the `Build()` produces an immutable result.

### Pointer vs value receivers

Builders are almost always implemented with **pointer receivers**:

```go
func (b *Builder) Select(...) *Builder { /*...*/ return b }
```

Why? Value receivers would copy the builder on every method call, losing accumulated state.

Exception: **immutable builders** that return a *new* builder per step:

```go
func (b Builder) Select(...) Builder {
    nb := b
    nb.columns = append(nb.columns, ...)  // careful with slice sharing
    return nb
}
```

Immutable is harder to get right (slice aliasing) but enables thread-safety and reuse. Rare in Go.

### Method names matter

Without overloading, every distinguishable operation needs a unique method:

```go
// Bad: ambiguous; both add things to "where"
.Where(x)
.Where(y)

// Better: distinguish
.AndWhere(x)
.OrWhere(y)
```

For a SQL builder, multiple WHERE conditions usually compose with AND; OR needs an explicit name.

### Generic builders

Since Go 1.18:

```go
type Builder[T any] struct{ value T }

func New[T any]() *Builder[T] { return &Builder[T]{} }
func (b *Builder[T]) Set(v T) *Builder[T] { b.value = v; return b }
func (b *Builder[T]) Build() T { return b.value }
```

Useful for *frameworks* (e.g., generic container builders). Most domain-specific builders use concrete types.

### Builder vs functional options

| Aspect | Builder | Functional options |
|---|---|---|
| Visual style | `b.X().Y().Build()` | `New(WithX(), WithY())` |
| Error handling | Build() | each option or constructor |
| Reusable | yes, naturally | yes |
| Adding fields | additive | additive |
| Allocation | one builder | one closure per option |
| Mental model | step-by-step | declarative |
| Conditional steps | `if cond { b.X() }` | `if cond { opts = append(opts, ...) }` |
| IDE autocomplete | shows methods after `.` | shows `WithX` functions |

Builders win for:
- Multi-step, stateful construction.
- DSL-like APIs (SQL, query languages).
- When intermediate states are meaningful.

Options win for:
- Configuration of a single object.
- Most server / client setups.

### Test data builders

A very common Go usage: building test fixtures.

```go
func aUser() *UserBuilder { return &UserBuilder{name: "Alice", age: 30} }

func (b *UserBuilder) Named(n string) *UserBuilder { b.name = n; return b }
func (b *UserBuilder) Aged(a int) *UserBuilder { b.age = a; return b }
func (b *UserBuilder) Build() User { return User{Name: b.name, Age: b.age} }

// In tests:
u := aUser().Named("Bob").Aged(25).Build()
```

Sometimes called **Object Mother + Builder hybrid**. Wonderfully readable in tests; defaults reduce noise.

### Builder for byte construction (the stdlib one)

`strings.Builder` and `bytes.Buffer` are the most-used builders in Go — but they're imperative:

```go
var sb strings.Builder
sb.WriteString("hello")
sb.WriteString(", world")
out := sb.String()
```

Not fluent; you call methods for their side effects. The "builder" terminology still applies.

### Hot-path use

Builders allocate the builder struct (~32 bytes) plus per-step allocations (if any). For hot paths, avoid the pattern or use a `sync.Pool` of builders.

```go
var builderPool = sync.Pool{New: func() any { return &QueryBuilder{} }}

func New(table string) *QueryBuilder {
    b := builderPool.Get().(*QueryBuilder)
    b.q = &Query{table: table}
    return b
}

func (b *QueryBuilder) Build() (string, error) {
    defer func() {
        *b = QueryBuilder{}     // reset
        builderPool.Put(b)
    }()
    // ... build ...
}
```

Trades clarity for performance. Use only when measured.

## Standard Library Hooks

- `strings.Builder` — append text to a buffer; `String()` returns.
- `bytes.Buffer` — same for bytes; reusable.
- `net/url.Values` — kind of a builder for query strings.
- `crypto/x509.CertificateRequest` — populate then `CreateCertificateRequest`.
- `database/sql.Stmt` (kind of, after `Prepare`).
- `text/template.Template`, `html/template.Template` — `New(...).Parse(...).Funcs(...)`.
- `flag.NewFlagSet`, `flag.Var(...)`.
- `go/build.Default`, `go/build.Context` (legacy).

## Real-World Patterns

### 1. SQL query builder

```go
sql, err := NewSelect("users").
    Columns("id", "name").
    Where("active = ?", true).
    OrderBy("created_at DESC").
    Limit(100).
    Build()
```

Used by many ORMs and query libraries (`squirrel`, `goqu`, `sqlx` extensions).

### 2. HTTP request builder

```go
req, err := New().
    Method(http.MethodPost).
    URL("https://api.example.com/v1/orders").
    Header("Authorization", "Bearer "+token).
    JSON(payload).
    Build()
resp, err := http.DefaultClient.Do(req)
```

Cleaner than constructing an `*http.Request` directly with multiple statements.

### 3. Test data builder

```go
order := anOrder().
    For(aCustomer().Named("Alice").Build()).
    With(anItem().SKU("ABC").Build()).
    With(anItem().SKU("DEF").Qty(2).Build()).
    PaidWith("CARD").
    Build()
```

Tests read like English.

### 4. Builder with sub-builders

```go
package email

type Builder struct {
    from, to    string
    subject     string
    body        string
}

type AttachmentBuilder struct {
    parent *Builder
    name   string
    data   []byte
}

func (b *Builder) Attach() *AttachmentBuilder {
    return &AttachmentBuilder{parent: b}
}
func (ab *AttachmentBuilder) Named(n string) *AttachmentBuilder { ab.name = n; return ab }
func (ab *AttachmentBuilder) Data(d []byte) *AttachmentBuilder { ab.data = d; return ab }
func (ab *AttachmentBuilder) Done() *Builder {
    // add attachment to parent
    return ab.parent
}

// Usage:
b := New().
    To("alice@example.com").
    Subject("hello").
    Body("hi there").
    Attach().Named("file.pdf").Data(pdfBytes).Done().
    Build()
```

Sub-builders give you nested context without losing the chain.

### 5. Conditional steps

```go
b := New().Select("id")
if includeName { b = b.AndSelect("name") }
if userID != 0 { b = b.Where("user_id = ?", userID) }
sql, err := b.Build()
```

Loose-coupling between caller logic and builder chain. The builder doesn't know about conditions.

## Anti-Patterns & Gotchas

**Builder that's also the result.** Calling `Build()` should produce a distinct, often immutable, object. Otherwise the builder leaks into the rest of the program.

**Hidden state in step methods.** A builder that writes to a database mid-chain is a footgun. Steps should be pure mutations of builder state.

**Builders that allocate per step.** Each step that appends to a slice or map is fine; calling `Build()` repeatedly should be cheap if reusable.

**Returning a different type per step.** Fluent chains break: `b.X().Y().Z()` only works if each returns a chainable type.

**Mixing builder and direct setter.** Caller doesn't know whether to chain or assign. Pick one.

**Builders for two-parameter objects.** Overkill. `NewUser(name, age string) *User` is cleaner.

**Skipping `Build()` validation.** Let invalid objects escape, fails downstream.

**Method names that don't read well in chains.** `b.AddOneToTheCounter()` reads better as `b.Increment()`.

**Builders that retain references to caller-owned data.** A `b.WithBytes(largeSlice)` that keeps `largeSlice` alive prevents GC. Document or copy.

**Forgetting that builders aren't goroutine-safe.** Don't share a builder across goroutines unless you've explicitly designed for it.

## Performance Notes

- Builder struct: ~32-128 bytes typical.
- Per-step method call: ~ns (inlined).
- `strings.Builder` and `bytes.Buffer`: highly tuned; ~ns per Write.
- Pooled builders: negligible per use after warmup.
- Closure-allocating builders (rare): one alloc per use.

## How Big Companies Use It

- **Masterminds/squirrel** (popular SQL builder): https://github.com/Masterminds/squirrel — used by many Go services.
- **Uber's Cadence/Temporal**: builders for workflow definitions.
- **AWS SDK for Go v2**: many request structs follow builder-ish patterns (set field, call `Send`).
- **Google Cloud Go SDK**: similar.
- **CockroachDB internal**: lots of expression builders for SQL plans.
- **gqlgen** (GraphQL): builders for resolver registration.
- **gjson** / **sjson** (Tidwall): not literally builders but use the chained API style.

## Source Code References

- `strings.Builder`: [`src/strings/builder.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/strings/builder.go).
- `bytes.Buffer`: [`src/bytes/buffer.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/bytes/buffer.go).
- `text/template.New`: [`src/text/template/template.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/text/template/template.go).
- Masterminds/squirrel: [`github.com/Masterminds/squirrel`](https://github.com/Masterminds/squirrel).
- doug-martin/goqu: [`github.com/doug-martin/goqu`](https://github.com/doug-martin/goqu).
- url.Values: [`src/net/url/url.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/net/url/url.go) — search `Values`.

## Further Reading

- "Test data builders" — Nat Pryce, Steve Freeman, in their book *Growing Object-Oriented Software, Guided by Tests*.
- "The Builder Pattern in Go" — multiple blog posts.
- "API design — builder vs options" — Bryan Mills' contributions to golang-nuts.
- Effective Go — composition: https://go.dev/doc/effective_go.
- Refactoring Guru's Builder pattern (language-agnostic): https://refactoring.guru/design-patterns/builder.

## Exercises / Self-Check

1. Implement an HTTP request builder with chainable `Method`, `URL`, `Header`, `JSON(body)`, `Build()` methods. Build() returns `(*http.Request, error)`.
2. Convert your builder to be reusable: `Build()` should return a *new* request each call without mutating the builder.
3. Pick a Go function in your codebase with 5+ parameters. Convert to builder pattern. Compare readability.
4. Implement a test data builder for a `User` type with sensible defaults. Show a test that overrides only one field.
5. Pool a builder using `sync.Pool`. Compare allocation counts with and without via `-benchmem`.
