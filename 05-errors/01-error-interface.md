# The `error` Interface — Why Errors Are Values

## TL;DR

`error` is a one-method interface (`Error() string`) declared in the language's universe block. Go has no exceptions: failure is a regular return value, almost always the last one, that the caller must inspect. The "errors are values" philosophy means you can store them, compare them, wrap them, and pass them across goroutines like any other value — but it also means the `if err != nil` boilerplate is non-negotiable, and the typed-nil interface trap (returning a `nil` pointer that satisfies `error`) is the single most common bug.

## Mental Model

```
error is just:

    type error interface {
        Error() string
    }

A value that satisfies it is two words at runtime:

    +--------+--------+
    | *itab  | *data  |    (the standard iface header — see Part 4)
    +--------+--------+

"nil error" = both words are nil.
A non-nil itab with a nil data pointer is a NON-nil error
  (even if the concrete pointer it wraps is nil).
```

Errors flow up the call stack as ordinary return values. Nothing in the runtime treats `error` specially — the compiler does not insert recovery code, the GC does not scan errors differently, panics are an unrelated mechanism (see `05-panic-and-recover.md`).

## Syntax & Basic Usage

```go
package main

import (
	"errors"
	"fmt"
)

// A function returns an error as its last return value.
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("divide by zero")
	}
	return a / b, nil
}

func main() {
	q, err := divide(10, 0)
	if err != nil {
		fmt.Println("error:", err)
		return
	}
	fmt.Println("result:", q)
	// Output:
	// error: divide by zero
}
```

Two stdlib constructors cover 90% of cases: `errors.New(string)` for static messages, `fmt.Errorf(format, args...)` for formatted ones. Anything beyond that — typed errors, wrapping, joining — is built on top.

## Deep Dive

### The declaration

`error` is one of the few predeclared identifiers in the language spec, defined as if it were:

```go
type error interface {
	Error() string
}
```

It lives in the universe block (no import needed) and is the only interface in that block other than `comparable` and `any`. See the spec under "Errors": https://go.dev/ref/spec#Errors.

### `errors.New` vs `fmt.Errorf`

```go
// errors.New — backing type is *errorString, identity-comparable
var ErrClosed = errors.New("conn closed")

// fmt.Errorf — also returns *errorString unless %w is present,
// in which case it returns *fmt.wrapError (single %w) or *fmt.wrapErrors (multi %w, 1.20+)
err := fmt.Errorf("read %s: %w", path, io.EOF)
```

`errors.New` allocates an `*errorString` with the passed message. Two calls with the same string produce **different** pointers — equality is by identity, not content. That is the whole point: sentinels like `io.EOF` are unique because of the `*errorString` address, not the text.

### The typed-nil trap (the canonical Go bug)

```go
type MyErr struct{ Code int }
func (e *MyErr) Error() string { return fmt.Sprintf("code=%d", e.Code) }

func find() error {
	var e *MyErr      // nil concrete pointer
	if conditionOK() {
		return nil    // OK
	}
	return e          // BUG: interface header has a non-nil itab, nil data
}

err := find()
if err != nil {
	// Always true when conditionOK is false — even though e was nil.
}
```

The cause is the two-word `iface` representation. `err == nil` is true only when **both** words are zero. Assigning a typed nil to an `error` produces a header with a non-nil `*itab` and a nil `*data`, which is not equal to a bare `nil`. Cure: always return the untyped `nil` explicitly, or never use typed nils.

`go vet` flags some forms of this (`nilness` analyzer). It does not catch all of them. The lint `nilerr` in `golangci-lint` covers more cases.

### Implementing a custom error type

```go
package httpx

import "fmt"

type StatusError struct {
	Code int
	Body string
}

func (e *StatusError) Error() string {
	return fmt.Sprintf("http %d: %s", e.Code, e.Body)
}
```

Conventions for custom errors:

- Pointer receiver if the type carries state and must be compared by identity.
- Value receiver if the type is small and immutable (rare).
- Exported types start with `Err...` only when they are singletons (sentinels); typed errors that take fields use plain CamelCase like `StatusError`, `*os.PathError`.
- The message starts lowercase, no trailing punctuation. Reason: errors are often wrapped: `fmt.Errorf("opening %s: %w", path, err)` should read naturally.

### Idiomatic check shape

```go
if err := step(); err != nil {
	return fmt.Errorf("step: %w", err)
}
```

Two things to internalize:

1. `if`-with-init keeps `err` scoped to the branch. The variable does not bleed past.
2. Almost every check returns immediately. "Happy path on the left" — indented `if err == nil { ... }` blocks are an anti-pattern.

### Errors as comparable values

Because errors are values, you can:

- Store them in maps and slices.
- Compare them with `==` (when the dynamic type is comparable).
- Send them over channels.
- Marshal them (with care — `error` has no JSON encoding by default).

Identity comparison is the foundation of sentinel errors:

```go
if err == io.EOF { ... }            // pre-1.13 style, still valid for direct returns
if errors.Is(err, io.EOF) { ... }   // modern, walks the wrap chain
```

Prefer `errors.Is` for any code that might be called with wrapped errors (covered in `02-errors-package.md`).

### Where the `error` value lives in memory

An `error` interface value is two machine words. The concrete error data behind it usually lives on the heap (interface assignment of a non-pointer value causes a boxing allocation; pointer values get reused as-is). Pre-allocated sentinels (`io.EOF`, `sql.ErrNoRows`) live in package-level globals: no per-call allocation.

```go
var ErrTimeout = errors.New("timeout") // allocated once at init
```

Hot loops that return errors should prefer sentinels or pre-allocated typed errors over `fmt.Errorf` to avoid allocating per call.

### The cost of `fmt.Errorf`

`fmt.Errorf` uses reflection-driven formatting plus an allocation for the resulting `*errorString`/`*wrapError`. On the error path that does not matter. But code that returns errors at millions of calls per second (parsers, lexers, low-level codecs) often switches to a custom error type with no formatting and reuses an instance.

### What `error.Error()` should and should not do

`Error()` should return a short, lowercase, machine-parseable string. It should not:

- Allocate giant strings (no embedded stack traces).
- Format with the time of day or PIDs (log layer's job).
- Cause side effects.
- Panic.

The runtime calls `Error()` from `fmt` printing, log frameworks, the panic handler, and tests. A panicking `Error()` is hard to debug because the stack trace points at the printer, not the bug.

## Standard Library Hooks

- `errors` — `New`, `Is`, `As`, `Join`, `Unwrap`. The vocabulary of the error system.
- `fmt.Errorf` — formatted errors, supports the `%w` verb for wrapping (and multi-`%w` since 1.20).
- `io.EOF`, `io.ErrUnexpectedEOF`, `io.ErrClosedPipe` — canonical sentinels.
- `os` — `*PathError`, `*LinkError`, `*SyscallError`, plus `os.ErrNotExist`, `os.ErrExist`, `os.ErrPermission`. Use `errors.Is` against the `Err...` sentinels, not against the wrapper types.
- `syscall.Errno` — POSIX errnos as an `error` type.
- `net.Error`, `net.OpError`, `net.DNSError` — wrap with `Temporary()` (deprecated) and `Timeout()` methods.
- `context.Canceled`, `context.DeadlineExceeded` — the two canonical context sentinels.
- `sql.ErrNoRows`, `sql.ErrTxDone`, `sql.ErrConnDone` — `database/sql` sentinels.
- `encoding/json.SyntaxError`, `*json.UnmarshalTypeError` — typed errors with location info.

## Real-World Patterns

### 1. Add context with `%w` at every layer

```go
package store

import (
	"database/sql"
	"errors"
	"fmt"
)

func (s *Store) GetUser(id string) (*User, error) {
	row := s.db.QueryRow("SELECT name FROM users WHERE id = $1", id)
	var u User
	if err := row.Scan(&u.Name); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("user %q: %w", id, ErrUserNotFound)
		}
		return nil, fmt.Errorf("scan user %q: %w", id, err)
	}
	return &u, nil
}

var ErrUserNotFound = errors.New("user not found")
```

Each layer adds *one* fragment of context (`scan user "abc": no such row`). The caller can still match on `ErrUserNotFound` via `errors.Is`.

### 2. Sentinel for a known terminal condition

```go
package iter

import "errors"

var ErrDone = errors.New("iter: done")

type Iter[T any] struct{ /* ... */ }

func (it *Iter[T]) Next() (T, error) {
	var zero T
	if it.exhausted() {
		return zero, ErrDone
	}
	return it.advance(), nil
}

// Caller:
for {
	v, err := it.Next()
	if errors.Is(err, ErrDone) {
		break
	}
	if err != nil {
		return err
	}
	consume(v)
}
```

Sentinels work well for *terminal* conditions like `io.EOF` and `iter.ErrDone`. They work poorly for rich failures (no fields to inspect).

### 3. Typed error for structured information

```go
package validate

import "fmt"

type FieldError struct {
	Field string
	Value any
	Rule  string
}

func (e *FieldError) Error() string {
	return fmt.Sprintf("validate %s=%v: %s", e.Field, e.Value, e.Rule)
}

func Email(s string) error {
	if !strings.Contains(s, "@") {
		return &FieldError{Field: "email", Value: s, Rule: "must contain @"}
	}
	return nil
}
```

Callers extract details with `errors.As(err, &fe)`. See `04-sentinel-vs-typed-vs-opaque.md` for when this is the wrong choice.

## Anti-Patterns & Gotchas

**Typed nil.** Covered above. Always return bare `nil` from a function whose return type is `error`.

**Ignoring errors via `_`.** Sometimes valid (`_ = file.Close()` on a read-only file), but if you write `_ = json.NewEncoder(w).Encode(x)` you've thrown away every failure mode that matters. Convention: comment why you are discarding.

**Logging *and* returning.** Common code-review smell:

```go
if err != nil {
	log.Printf("read failed: %v", err) // logged here…
	return err                          // …and the caller will log it again
}
```

Pick one. Usually return-and-let-the-caller-log; log only at the boundary (HTTP handler, main, goroutine entry point).

**`if err == nil` happy path indented.** Reverse it.

**`fmt.Errorf` without `%w` when wrapping.** `fmt.Errorf("read: %v", err)` loses the chain. `errors.Is(returnedErr, io.EOF)` returns false. Use `%w` unless you specifically want to *hide* the cause from callers.

**Custom error types whose `Error()` panics on nil receiver.** Always handle the nil case or never pass nil.

**Comparing wrapped errors with `==`.** Will not match after a wrap. Use `errors.Is`.

**`error.Error()` containing PII or secrets.** Errors get logged. Redact at construction, not at log time.

**Pre-1.13 wrapping libraries (`pkg/errors`).** Still works, but its `Wrap`/`Cause` predates `%w`/`errors.Is`/`errors.As`. New code should use stdlib. (The library's author archived it in 2021.)

## Performance Notes

- A sentinel `error` is one heap allocation total, performed at package init.
- `errors.New(s)` allocates a `*errorString` plus the string itself. Cache sentinels at package level.
- `fmt.Errorf` without `%w` allocates the formatted message + an `*errorString`. With `%w`, also a `*wrapError`. Single-digit hundred nanoseconds typical.
- An interface assignment of a non-pointer concrete error (e.g., `var e error = StatusError{...}` with value receiver) boxes the value to the heap. Pointer receivers avoid this.
- Hot paths that *can* return an error but usually return `nil`: the `nil` interface value is two literal zero words; no allocation. Free.
- Wrapping deep chains is cheap to construct but `errors.Is`/`errors.As` walk the chain linearly. Don't wrap 50 layers deep.

Benchmark sketch:

```go
func BenchmarkErrorsNew(b *testing.B) {
	for b.Loop() { // since 1.24
		_ = errors.New("oops")
	}
}
func BenchmarkSentinel(b *testing.B) {
	var sink error
	for b.Loop() {
		sink = ErrTimeout
	}
	_ = sink
}
```

On a modern x86 box: `errors.New` ≈ 40 ns/op + 2 allocs; sentinel assignment ≈ 0.3 ns/op + 0 allocs.

## How Big Companies Use It

- **Kubernetes** treats `error` as the universal return channel; the project's `k8s.io/apimachinery/pkg/api/errors` exposes typed `StatusError` plus `IsNotFound`, `IsAlreadyExists`, `IsConflict` predicates — see `06-error-handling-at-scale.md`.
- **CockroachDB** built `github.com/cockroachdb/errors` because the stdlib lacked stack traces, structured details, and redaction. Read the project's `errors/README.md` for the reasoning: https://github.com/cockroachdb/errors.
- **Docker / Moby** wraps every Docker daemon error in `errdefs` typed categories (`ErrNotFound`, `ErrConflict`, `ErrUnauthorized`) so the HTTP layer can map them to status codes without each handler knowing the details: https://github.com/moby/moby/tree/master/errdefs.
- **HashiCorp** uses `hashicorp/go-multierror` (pre-`errors.Join`) heavily in Terraform plan/apply.
- **gRPC-Go** uses typed `*status.Status` errors paired with `status.FromError` — a domain-specific take on the "error as value" pattern with explicit status codes.

## Source Code References

Pinned to `go1.26`.

- The `error` interface is in the universe block: [`src/builtin/builtin.go`](https://github.com/golang/go/blob/master/src/builtin/builtin.go) (search for `type error`).
- `errors` package: [`src/errors/errors.go`](https://github.com/golang/go/blob/master/src/errors/errors.go), [`src/errors/wrap.go`](https://github.com/golang/go/blob/master/src/errors/wrap.go), [`src/errors/join.go`](https://github.com/golang/go/blob/master/src/errors/join.go).
- `fmt.Errorf` and `%w`: [`src/fmt/errors.go`](https://github.com/golang/go/blob/master/src/fmt/errors.go).
- The `*errorString` type returned by `errors.New`: [`src/errors/errors.go`](https://github.com/golang/go/blob/master/src/errors/errors.go), lines ~10–35.
- Canonical typed nil issue: https://go.dev/doc/faq#nil_error.

License: Go source is BSD-3 (Copyright the Go Authors). Cite when copying snippets.

## Further Reading

- Spec, "Errors": https://go.dev/ref/spec#Errors and "Predeclared identifiers": https://go.dev/ref/spec#Predeclared_identifiers.
- Go blog, "Error handling and Go" (Andrew Gerrand, 2011): https://go.dev/blog/error-handling-and-go — still the canonical introduction.
- Go blog, "Errors are values" (Rob Pike, 2015): https://go.dev/blog/errors-are-values.
- Go blog, "Working with Errors in Go 1.13": https://go.dev/blog/go1.13-errors — origin of `%w`, `errors.Is`, `errors.As`.
- Go blog, "Working with multiple errors" (2023, for `errors.Join`): https://go.dev/blog/errors-join.
- Dave Cheney, "Don't just check errors, handle them gracefully": https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully.
- Russ Cox, on the error proposal review (2019): https://research.swtch.com/go2019.
- Go FAQ, "Why does my nil error value not equal nil?": https://go.dev/doc/faq#nil_error.

## Exercises / Self-Check

1. Implement `Stringer` and `error` for the same type. What happens with `fmt.Println(v)` — which method is called?
2. Write a function that returns `*MyErr` typed but declared with `error` return type. Trigger the typed-nil bug, then fix it. Explain in one sentence which word of the `iface` header is non-nil.
3. Why does the standard library use both `*PathError` (typed) and `io.EOF` (sentinel) instead of picking one style? When would a third option, behavior assertion (`net.Error.Timeout()`), be better?
4. Take the loop `for { v, err := it.Next(); if errors.Is(err, ErrDone) { break }; ... }`. Convert it to a `range`-over-func (`iter.Seq2`) since 1.23. How does error reporting change?
5. Benchmark `errors.New("x")` vs a package-level sentinel inside a tight loop. By how much does the sentinel win, and why?
