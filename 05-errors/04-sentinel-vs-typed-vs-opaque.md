# Sentinel vs Typed vs Opaque Errors — Dave Cheney's Taxonomy

## TL;DR

Three ways to expose an error from a package: a **sentinel** (an exported `var Err... = errors.New(...)`), a **typed error** (an exported struct implementing `error`), or an **opaque error** (just an `error` value whose concrete type is unexported, inspected via behavior). Each commits the caller to a different coupling. Dave Cheney's rule: **prefer opaque + behavior**; reach for typed errors when callers genuinely need structured fields; reach for sentinels only for terminal states or where comparison is the whole point (`io.EOF`). The trap is using the strongest coupling (sentinels everywhere) when the weakest would do.

## Mental Model

```
 Coupling weakens  ──────────────────────────────────►
       strong                                    weak

   SENTINEL          TYPED ERROR         OPAQUE ERROR
   var ErrX = ...    type FooErr struct  return errors.New(...)
   errors.Is(err,X)  errors.As(err,&fe)  if behaviorIface, ok := ...

   Caller binds to:
   * variable name   * type identity     * a method / behavior
   * package import  * package import    * (no package import needed if behavior is shared)
```

The choice is about *what callers are allowed to depend on*. Once exported, a sentinel value or a typed error becomes part of your package's API surface forever; an opaque error stays free to refactor.

## Syntax & Basic Usage

```go
package fileops

import (
	"errors"
	"fmt"
	"io/fs"
)

// 1. SENTINEL — exported variable, callers compare with errors.Is.
var ErrCorrupt = errors.New("fileops: archive is corrupt")

// 2. TYPED — exported struct, callers extract fields with errors.As.
type ChecksumErr struct {
	Want, Got [32]byte
	At        int64
}

func (e *ChecksumErr) Error() string {
	return fmt.Sprintf("fileops: checksum mismatch at %d", e.At)
}

// 3. OPAQUE — unexported concrete type, callers ONLY know it's an error.
func ReadHeader(r fs.File) error {
	// returns a *headerErr (unexported); callers can't type-assert.
	return &headerErr{op: "ReadHeader", err: errors.New("truncated")}
}

type headerErr struct {
	op  string
	err error
}
func (e *headerErr) Error() string { return e.op + ": " + e.err.Error() }
func (e *headerErr) Unwrap() error { return e.err }
```

## Deep Dive

### Sentinel errors

```go
package io
var EOF = errors.New("EOF")
```

**Properties:**

- Identity by pointer. `io.EOF == io.EOF` is true because both sides refer to the same `*errorString`.
- Callers compare with `errors.Is(err, io.EOF)` (or `err == io.EOF` for un-wrapped direct returns).
- Exported as a package-level `var`, becoming part of the public API.
- Cannot carry structured data — the error message is fixed.

**When to use:** terminal/well-known conditions that have *no* extra structure to communicate. `io.EOF` (the stream is over), `sql.ErrNoRows` (no row matched), `context.Canceled` (cancellation), `http.ErrServerClosed` (graceful shutdown). When the entire signal is "this specific named condition occurred."

**When to avoid:** rich failures with parameters (a 404 for *what URL*? a parse error at *what offset*?). Don't define 30 sentinels to represent variations of one structured condition; use a typed error with a field.

**Subtle traps:**

- Once exported, you cannot change the message text without breaking callers who match by string (which they shouldn't, but do).
- Sentinels mutate carelessly: `io.EOF.Error = "different"` — there's nothing preventing it in Go. (`*errorString` has an unexported field, so this is mostly hypothetical, but be aware.)
- Sentinels imported across module boundaries pin two packages together at the import level forever.

### Typed errors

```go
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
func (e *PathError) Unwrap() error { return e.Err }
```

**Properties:**

- Carry structured fields.
- Callers extract with `errors.As(err, &pe)`.
- Type becomes part of your package's public API.
- Can implement custom `Is` to also match sentinels (so a `*PathError` wrapping `syscall.ENOENT` matches `fs.ErrNotExist`).

**When to use:** when callers must inspect specific fields to decide what to do. HTTP status codes, validation errors with field names, parse errors with offsets, retryable errors with retry-after durations. Whenever the error has parameters and at least one caller cares about them.

**When to avoid:** when no caller actually inspects the fields. If your typed error is only ever checked for "did it fail," you've exported a type you don't need to. Use opaque or sentinel.

**Subtle traps:**

- Once exported, every field is API. Add fields freely, but renaming or removing is breaking.
- A typed error with pointer receiver methods does not satisfy an interface if you create a value-typed instance (`PathError{...}` vs `&PathError{...}`).
- Implementing `Is` and `As` thoughtfully often lets a typed error replace a fleet of sentinels.

### Opaque errors

```go
package store

import "errors"

func (s *Store) Get(id string) error {
	return errors.New("store: lookup failed") // caller knows only that an error occurred
}
```

Or, more usefully, an opaque error that exposes *behavior*:

```go
package store

import "fmt"

type retryableErr struct{ msg string; after time.Duration }
func (e *retryableErr) Error() string         { return e.msg }
func (e *retryableErr) RetryAfter() time.Duration { return e.after }

// Caller (in another package):
type retryable interface{ RetryAfter() time.Duration }

if r, ok := err.(retryable); ok {
	time.Sleep(r.RetryAfter())
	return Try()
}
```

**Properties:**

- Concrete type is unexported. Callers cannot type-assert to it.
- Coupling is to a *behavior interface* (`retryable`, `timeouter`), often declared at the *caller's* side.
- The package author retains freedom to change the concrete implementation.

**When to use:** by default, whenever the failure does not need to be inspected with `errors.Is` or `errors.As`. When callers care about a property (timeout? retryable? user-fixable?), expose a small interface they can test for.

**When to avoid:** when callers truly need identity-equality (then a sentinel is honest about the coupling) or structured fields (then a typed error is honest).

**Subtle traps:**

- Behavior interfaces must be re-declared at every caller. That's the price of decoupling and usually worth it.
- Behavior must be stable: if `RetryAfter()` becomes an irrelevant method, you've broken every caller depending on it.
- Don't accidentally expose the concrete type by returning `*retryableErr` from an exported function signature; declare the return type as `error`.

### The `Cheney rule`

Dave Cheney's order of preference (from "Don't just check errors, handle them gracefully", 2016):

> 1. Handle by behavior, not by type or identity.
> 2. If behavior is insufficient, handle by type.
> 3. Use sentinel values as a last resort.

The reasoning: sentinels create the strongest coupling (caller imports your package just for a `var`), typed errors create a moderate coupling (caller imports your package for a `type`), behavior-based opaque errors create the weakest (caller invents the interface locally; multiple unrelated packages can all be inspected with the same predicate). Weakest coupling that does the job = best.

### Where the stdlib breaks the rule

`io.EOF` is a sentinel because callers needed to test for it long before `errors.As` existed. `*net.OpError`, `*os.PathError`, `*json.SyntaxError` are typed because callers genuinely needed fields. The stdlib uses all three styles — pragmatically, not dogmatically.

What it *doesn't* do is define `var ErrInvalid, ErrTooLarge, ErrEmpty, ErrMissing, ErrMalformed, ...` as a wall of sentinels for one operation. When you see 20 exported sentinels in a package, it's usually 19 too many.

### Combining the patterns

Real packages mix all three:

```go
package store

// Sentinel for the terminal "doesn't exist" condition.
var ErrNotFound = errors.New("store: not found")

// Typed error for validation with structured fields.
type ValidationErr struct {
	Field string
	Rule  string
}
func (e *ValidationErr) Error() string { return ... }

// Opaque, behavior-based for retries — caller defines the interface.
type transientErr struct{ err error; after time.Duration }
// (not exported; expose via a method)
```

Caller of all three:

```go
err := store.Save(item)
switch {
case errors.Is(err, store.ErrNotFound):
	create()
case errors.As(err, &validErr):
	report(validErr.Field, validErr.Rule)
default:
	type retryable interface{ RetryAfter() time.Duration }
	if r, ok := err.(retryable); ok {
		time.Sleep(r.RetryAfter())
		retry()
	}
}
```

### How to decide, in practice

Walk the questions in order:

1. **Will callers do something different *only* because this exact named condition occurred?** (e.g., "stop iterating because we hit EOF"). → Sentinel.
2. **Will callers extract a specific field to decide what to do?** (e.g., "retry after this duration", "show user this field is invalid"). → Typed error.
3. **Will callers ask "is this kind of failure?"** (transient, permission, user-fixable). → Opaque error + behavior interface.
4. **Will callers only log and bubble up?** → Opaque error, no exported type at all.

Most "errors" in your codebase fall in case 4. Default to opaque.

## Standard Library Hooks

- Sentinels: `io.EOF`, `io.ErrUnexpectedEOF`, `io.ErrShortWrite`, `io.ErrClosedPipe`, `context.Canceled`, `context.DeadlineExceeded`, `sql.ErrNoRows`, `sql.ErrTxDone`, `sql.ErrConnDone`, `http.ErrServerClosed`, `fs.ErrNotExist`/`fs.ErrExist`/`fs.ErrPermission`/`fs.ErrInvalid`/`fs.ErrClosed`, `net.ErrClosed`, `os.ErrDeadlineExceeded`.
- Typed errors: `*fs.PathError`, `*os.LinkError`, `*os.SyscallError`, `*net.OpError`, `*net.DNSError`, `*net.AddrError`, `*url.Error`, `*json.SyntaxError`, `*json.UnmarshalTypeError`, `*exec.ExitError`, `*tls.RecordHeaderError`, `*tls.CertificateVerificationError`.
- Behavior interfaces (historical): `net.Error` (with `Timeout()`/`Temporary()`); `Temporary()` is deprecated but the *pattern* remains valid. Implementations should check for `context.DeadlineExceeded` and `os.IsTimeout(err)` instead.
- Opaque examples: most errors returned from `net/http` request execution that are not `*url.Error`.

## Real-World Patterns

### 1. Behavior interface declared at the caller

```go
// Package retry (caller) doesn't import package store at all.
type retryable interface{ RetryAfter() time.Duration }

func Do(op func() error) error {
	for attempt := 0; attempt < 5; attempt++ {
		err := op()
		if err == nil { return nil }
		var r retryable
		if errors.As(err, &r) {
			time.Sleep(r.RetryAfter())
			continue
		}
		return err
	}
	return errors.New("retry: exhausted")
}
```

`store`, `httpclient`, `grpcclient` — any package can implement `RetryAfter()` and slot in without coordinating.

### 2. Typed error with custom `Is` to match a sentinel

```go
var ErrNotFound = errors.New("not found")

type NotFoundErr struct{ Key string }

func (e *NotFoundErr) Error() string { return fmt.Sprintf("not found: %s", e.Key) }
func (e *NotFoundErr) Is(target error) bool { return target == ErrNotFound }
```

Callers can use either `errors.Is(err, ErrNotFound)` for "any not-found" or `errors.As(err, &nfe)` to get the key. The typed error is a strict superset; the sentinel is the rendezvous point.

### 3. Sentinel-free package via a single typed error with code

```go
type Code int
const (
	CodeUnknown Code = iota
	CodeNotFound
	CodeConflict
	CodeUnauthorized
	CodeInvalid
)

type APIErr struct {
	Code Code
	Msg  string
}
func (e *APIErr) Error() string { return e.Msg }

// Caller:
var ae *APIErr
if errors.As(err, &ae) && ae.Code == CodeNotFound { ... }
```

Replaces five sentinels with one type + an enum. The trade-off: callers must know the `Code` enum (also exported). For internal packages this is a clean compression; for libraries with many third-party callers, named predicates are easier to discover.

### 4. Hybrid: store-level sentinels mapped to a category interface

```go
package httpapi

import (
	"errors"
	"net/http"
	"store"
)

type httpStatusErr interface{ HTTPStatus() int }

func writeErr(w http.ResponseWriter, err error) {
	var hse httpStatusErr
	if errors.As(err, &hse) {
		http.Error(w, err.Error(), hse.HTTPStatus())
		return
	}
	switch {
	case errors.Is(err, store.ErrNotFound):
		http.Error(w, "not found", http.StatusNotFound)
	case errors.Is(err, store.ErrConflict):
		http.Error(w, "conflict", http.StatusConflict)
	default:
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}
```

Behavior interface (`httpStatusErr`) tried first, sentinel fallback for the legacy package. Real systems are full of this.

## Anti-Patterns & Gotchas

**Sentinel-everything.** A package with 25 exported `var Err...` lines is shouting "I have no design here." Group what's grouped, structure what's structured.

**Typed errors no one inspects.** If no test calls `errors.As(err, &myType)`, the type's existence is overhead. Either delete it or document why callers should be using it.

**Opaque-everything with no behavior.** "Just return `errors.New(...)`" is too far the other way. Callers downstream need *some* hook to make decisions. Add a behavior interface as soon as a real caller needs one.

**Implementing `Is` that recurses through `errors.Is`.** Infinite loop. Compare only your own state; never call back into `errors.Is`/`errors.As`.

**Exporting a typed error's fields you didn't mean to commit to.** Once `type FooErr struct{ Foo, Bar, Baz string }` exists, all three fields are API forever. Start with the smallest set.

**Behavior interface declared in the producing package.** Defeats the purpose: now callers import you anyway. Declare the interface at the caller, or in a neutral shared package.

**`net.Error.Temporary()` cargo-culted.** The method has been semantically meaningless for years and is deprecated. Use `errors.Is(err, context.DeadlineExceeded)` or `os.IsTimeout`.

**String-matching the error message.** `if strings.Contains(err.Error(), "not found")`. This breaks the moment a wrap layer changes wording. Use one of the three patterns above.

**Mixing pointer and value receiver for the same typed error.** Decide once. Pointer is the safe default; you can change `Error()` to a value receiver, but going back is messy.

**Naming sentinels without the package prefix in their text.** `errors.New("not found")` instead of `errors.New("store: not found")`. When the error bubbles up through five layers and gets logged, you'll be glad you said which package emitted it.

## Performance Notes

- Sentinels: zero per-call cost. One allocation at init.
- Typed errors: one allocation per error (the struct), unless you pre-allocate a singleton (sentinel-with-fields-zeroed).
- Opaque errors: same as typed, plus the indirection through a behavior interface lookup on inspection.
- Behavior lookup via `errors.As(err, &iface)` is reflect-based and ~10× slower than `errors.Is(err, sentinel)`. Still nanoseconds; matters only in hot inspection loops.
- Wrapping does not change the cost of *creating* the underlying typed/sentinel/opaque error; it adds a `*fmt.wrapError` per layer.
- Pre-allocated typed errors with shared fields are an option for hot paths: `var errBadInput = &ValidationErr{Field: "input"}` then `return errBadInput`. Loses per-call context but saves the alloc.

## How Big Companies Use It

- **Kubernetes** (`k8s.io/apimachinery/pkg/api/errors`): typed `*StatusError` with a `Status()` method returning structured metadata. Plus `IsNotFound`, `IsAlreadyExists`, `IsConflict` predicates that behavior-test the status code. Combines all three patterns in one package.
- **Docker/Moby `errdefs`**: pure behavior style. Errors implement `IsNotFound()`, `IsConflict()`, `IsForbidden()`, etc. methods. The HTTP API layer dispatches solely on these methods, never on identity or type. Beautiful example of opaque-by-behavior at scale.
- **CockroachDB `cockroachdb/errors`**: rich typed errors with hints, details, stack traces, redaction. Every error is structured. Justification: distributed SQL needs to send precise diagnostic info between nodes and back to clients.
- **HashiCorp Vault**: heavy sentinel usage for permission/policy decisions (`logical.ErrUnsupportedOperation`, `logical.ErrInvalidRequest`); the audit log relies on them.
- **gRPC-Go**: typed `*status.Status` with `Code()` and `Message()` methods; conventional behavior + identity hybrid via `status.FromError`.
- **Cloudflare** internal libraries (when public): tend to opaque + behavior at the boundary, with sentinels only where stdlib already established a name (`io.EOF`, `net.ErrClosed`).

## Source Code References

Pinned to `go1.26`.

- Stdlib sentinel example: [`src/io/io.go`](https://github.com/golang/go/blob/master/src/io/io.go) — `var EOF = errors.New("EOF")`.
- Typed error example: [`src/io/fs/fs.go`](https://github.com/golang/go/blob/master/src/io/fs/fs.go) — `type PathError struct {...}`.
- Typed error with custom `Is`: [`src/syscall/syscall_unix.go`](https://github.com/golang/go/blob/master/src/syscall/syscall_unix.go) — `func (e Errno) Is(target error) bool`.
- Behavior interface (historical): [`src/net/net.go`](https://github.com/golang/go/blob/master/src/net/net.go) — `type Error interface { error; Timeout() bool; Temporary() bool }`.
- Docker `errdefs` (third-party, BSD-like): https://github.com/moby/moby/tree/master/errdefs.
- Kubernetes errors package: https://github.com/kubernetes/apimachinery/tree/master/pkg/api/errors.

Go source is BSD-3 licensed; cite when copying.

## Further Reading

- Dave Cheney, "Don't just check errors, handle them gracefully" (2016): https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully — the canonical taxonomy.
- Dave Cheney, "Inspecting errors": https://dave.cheney.net/2014/12/24/inspecting-errors.
- Dave Cheney, "Error handling in Go" (GopherCon Singapore 2017): https://dave.cheney.net/2017/01/23/the-error-handling-deep-dive-with-go-1-9-and-go-2.
- Go blog, "Working with Errors in Go 1.13": https://go.dev/blog/go1.13-errors — discusses when `errors.Is`/`As` make sentinel-vs-typed easier to mix.
- Russ Cox, "Error Handling — Problem Overview" (Go 2 draft, 2018): https://go.googlesource.com/proposal/+/master/design/29934-error-values.md.
- Google Go style guide on errors: https://google.github.io/styleguide/go/decisions#errors.
- Uber Go style guide on errors: https://github.com/uber-go/guide/blob/master/style.md#errors.
- Cockroach errors library blog post: https://www.cockroachlabs.com/blog/error-handling-go/.

## Exercises / Self-Check

1. List five `var Err...` sentinels in the stdlib you've used. For each, argue whether a typed error or a behavior interface would have served better, and why the stdlib chose a sentinel.
2. Rewrite a package you've written that exports 5+ sentinels using a single typed error with a `Code` enum. Did callers get simpler or harder? When would you regret the consolidation?
3. Implement an opaque retryable error in package `producer`. In package `consumer`, declare a `retryable` interface locally and use `errors.As` to check. Add a second producer that also implements `RetryAfter()` and show the consumer needs no changes.
4. Take `*os.PathError` and explain how it bridges typed + sentinel: callers can `errors.As(err, &pe)` to get fields *and* `errors.Is(err, fs.ErrNotExist)` to test the category. Trace through the `Is` implementation.
5. Find a `net.Error.Temporary()` call in code you have. Replace it with an `errors.Is` or `os.IsTimeout` equivalent. Document any behavior change.
