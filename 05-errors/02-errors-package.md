# The `errors` Package — `Is`, `As`, `Join`, `Unwrap`

## TL;DR

The `errors` package is the runtime for Go's error model. `Is` walks the wrap chain looking for a target by identity; `As` walks it looking for a target by type and binds it; `Join` (since 1.20) combines several errors into one; `Unwrap` exposes the next link. Use `errors.Is`/`errors.As` instead of `==` and type assertions whenever the error could have been wrapped. Implement `Is(target error) bool`, `As(target any) bool`, or `Unwrap() error` / `Unwrap() []error` on your own types only when the default behavior is wrong.

## Mental Model

```
A wrapped error is a linked list (or tree, for multi-wrap):

    err ──► wrap1 ──► wrap2 ──► sql.ErrNoRows
               │
               └─► implements Unwrap() error

errors.Is(err, sql.ErrNoRows)
    walks the chain calling Unwrap until it finds == sql.ErrNoRows
    (or calls a custom Is on any node along the way)

errors.As(err, &target)
    walks the chain looking for a node assignable to *target's type
    when found, copies it in and returns true

errors.Join(a, b, c)
    returns an error whose Unwrap() returns []error{a, b, c}
    walks become tree walks; Is/As handle that transparently
```

The chain is a runtime data structure assembled by `fmt.Errorf("%w", ...)` and custom `Unwrap` methods. Nothing in the language enforces it — it is a convention plus four functions in `errors`.

## Syntax & Basic Usage

```go
package main

import (
	"errors"
	"fmt"
	"io"
	"os"
)

func main() {
	// errors.Is: identity over the chain
	err := fmt.Errorf("read config: %w", io.EOF)
	fmt.Println(errors.Is(err, io.EOF)) // true

	// errors.As: type extraction
	_, openErr := os.Open("/no/such/file")
	wrapped := fmt.Errorf("startup: %w", openErr)
	var perr *os.PathError
	if errors.As(wrapped, &perr) {
		fmt.Println("path:", perr.Path)
	}

	// errors.Join (1.20+): combine multiple
	joined := errors.Join(io.EOF, os.ErrPermission)
	fmt.Println(errors.Is(joined, io.EOF))             // true
	fmt.Println(errors.Is(joined, os.ErrPermission))   // true

	// Output:
	// true
	// path: /no/such/file
	// true
	// true
}
```

## Deep Dive

### `errors.New`

```go
func New(text string) error {
	return &errorString{text}
}
type errorString struct{ s string }
func (e *errorString) Error() string { return e.s }
```

Returns a unique pointer every call. Two `errors.New("x")` calls compare unequal — by design. Sentinels are unique because of the pointer identity.

### `errors.Unwrap`

```go
func Unwrap(err error) error {
	u, ok := err.(interface{ Unwrap() error })
	if !ok { return nil }
	return u.Unwrap()
}
```

Returns the *single* wrapped error, or nil if none. There is **no** `Unwrap` overload for the multi case at the package function level — for a multi-wrap node, `errors.Unwrap` returns nil and you must use `errors.Is`/`As` (or call the method `Unwrap() []error` directly).

### `errors.Is` — walking and matching

The algorithm in pseudocode:

```
Is(err, target):
    if target is comparable and err == target: return true
    while err != nil:
        if err has method Is(target) bool and err.Is(target): return true
        switch x := err.(type):
            case interface{ Unwrap() error }:   err = x.Unwrap()
            case interface{ Unwrap() []error }: for each child: if Is(child, target) return true; return false
            default: return false
    return false
```

Key consequences:

- A custom `Is(target) bool` method lets a single concrete error match many sentinels (e.g., `os.ErrNotExist` matches `*os.PathError` wrapping `syscall.ENOENT`).
- `target` is compared by `==`, so a custom non-comparable target (e.g., a struct containing a slice) will panic. Use a sentinel pointer.
- Cycle-free assumption: a custom `Unwrap` that returns itself causes infinite recursion. Don't.

### `errors.As` — walking and binding

```
As(err, target):
    if target is nil or not a non-nil pointer: panic
    targetType = *target's type (must be interface or implement error)
    while err != nil:
        if reflect.TypeOf(err) is assignable to targetType:
            *target = err
            return true
        if err has method As(target any) bool and err.As(target): return true
        unwrap as in Is and recurse on children
    return false
```

`target` must be a non-nil pointer to either an interface or to a type implementing `error`. The function panics on misuse so the failure surfaces during development, not silently at runtime. Bind site:

```go
var perr *fs.PathError
if errors.As(err, &perr) {
    // perr is set; safe to read perr.Op, perr.Path, perr.Err
}
```

### `errors.Join` (since 1.20)

```go
joined := errors.Join(err1, err2, err3) // nil args are dropped
```

Returns an error whose `Error()` is `err1.Error() + "\n" + err2.Error() + "\n" + ...`, and whose `Unwrap() []error` returns the slice. `errors.Is`/`errors.As` walk the slice.

`errors.Join(nil, nil, nil)` returns `nil`. Useful when accumulating in a loop:

```go
var errs []error
for _, item := range items {
	if err := process(item); err != nil {
		errs = append(errs, fmt.Errorf("%s: %w", item.Name, err))
	}
}
return errors.Join(errs...) // nil if all succeeded
```

### Multi-wrap with `fmt.Errorf` (since 1.20)

```go
err := fmt.Errorf("step failed: %w; cleanup also failed: %w", primaryErr, cleanupErr)
```

The resulting error implements `Unwrap() []error` returning the two wrapped errors in order. Use it when *two* causes are both relevant — typically primary + cleanup, or main + transient retry.

### Implementing `Is` on your own type

```go
type CodeErr struct{ Code int; Msg string }

func (e *CodeErr) Error() string { return fmt.Sprintf("%d: %s", e.Code, e.Msg) }

func (e *CodeErr) Is(target error) bool {
	t, ok := target.(*CodeErr)
	if !ok { return false }
	return e.Code == t.Code
}

// Usage:
errors.Is(&CodeErr{Code: 404, Msg: "x"}, &CodeErr{Code: 404})
// true — only Code is compared
```

Defining `Is` overrides the default `==` comparison. Useful for "match by category" semantics. The classic example is `os.PathError.Is(os.ErrNotExist)` which walks into the wrapped `syscall.Errno` and compares.

### Implementing `As` on your own type

Rarely needed. The default behavior (assignability check) handles most cases. You override `As` when you want to *synthesize* a target — e.g., extract a sub-field as if it were the target type:

```go
func (e *RPCErr) As(target any) bool {
	if t, ok := target.(**StatusError); ok {
		*t = e.inner
		return true
	}
	return false
}
```

### `errors.Unwrap` vs the method form

The function `errors.Unwrap` only knows about the single-error form. To inspect a multi-wrap node, use type assertion:

```go
if mw, ok := err.(interface{ Unwrap() []error }); ok {
	for _, child := range mw.Unwrap() {
		// inspect each
	}
}
```

`errors.Is` and `errors.As` handle this transparently.

### Comparable target requirement

`errors.Is(err, target)` does `err == target` somewhere along the way. If `target`'s dynamic type is non-comparable (contains a slice/map/function), this panics. Sentinels are always pointers to ensure they are comparable and unique.

```go
var ErrBad = errors.New("bad") // *errorString — comparable, unique

// Never:
var ErrBad = []error{errors.New("a"), errors.New("b")} // wrong type entirely
```

### What `nil` does at each entry point

- `errors.New("")` returns a non-nil error with empty message — usually a bug.
- `errors.Unwrap(nil)` returns `nil`.
- `errors.Is(nil, target)` returns `target == nil`.
- `errors.Is(err, nil)` returns true only if `err == nil`.
- `errors.As(nil, &x)` returns `false` (but still validates `&x`).
- `errors.Join()` and `errors.Join(nil, nil)` return `nil`.

## Standard Library Hooks

- `os.IsNotExist`, `os.IsExist`, `os.IsPermission`, `os.IsTimeout` — predicates that predate `errors.Is`. Prefer `errors.Is(err, fs.ErrNotExist)` in new code; the `os.Is*` functions are kept for compatibility.
- `os.ErrNotExist`, `os.ErrExist`, `os.ErrPermission`, `os.ErrClosed`, `os.ErrDeadlineExceeded` — sentinels for `errors.Is`.
- `fs.ErrNotExist`, `fs.ErrExist`, `fs.ErrPermission`, `fs.ErrInvalid`, `fs.ErrClosed` — the `io/fs` mirrors, preferred for filesystem-agnostic code.
- `context.Canceled`, `context.DeadlineExceeded` — match with `errors.Is`.
- `io.EOF`, `io.ErrUnexpectedEOF`, `io.ErrShortWrite`, `io.ErrClosedPipe`.
- `net.ErrClosed` (1.16+) for closed-connection detection.
- `http.ErrServerClosed` for graceful-shutdown detection in `http.Server.ListenAndServe`.
- `sql.ErrNoRows`, `sql.ErrTxDone`, `sql.ErrConnDone`.
- `syscall.Errno` implements `Is` so that errnos compare correctly across wrap layers.
- `*fs.PathError` and `*os.PathError` (same type since 1.16) implement `Unwrap`.

## Real-World Patterns

### 1. Predicate the caller actually uses

```go
package store

import (
	"database/sql"
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("not found")

func (s *Store) Get(id string) (*Row, error) {
	row := s.db.QueryRow("SELECT * FROM rows WHERE id=$1", id)
	var r Row
	if err := row.Scan(&r); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("id %q: %w", id, ErrNotFound)
		}
		return nil, fmt.Errorf("scan: %w", err)
	}
	return &r, nil
}

// HTTP handler:
if errors.Is(err, store.ErrNotFound) {
	http.Error(w, err.Error(), http.StatusNotFound)
	return
}
```

The store hides `database/sql` details and exports its own sentinel. Callers couple to *one* package, not to the SQL driver.

### 2. Pulling structured information via `errors.As`

```go
import (
	"encoding/json"
	"errors"
)

err := json.Unmarshal(blob, &v)
var syn *json.SyntaxError
if errors.As(err, &syn) {
	logger.Error("bad JSON", "offset", syn.Offset)
}
var ut *json.UnmarshalTypeError
if errors.As(err, &ut) {
	logger.Error("type mismatch", "field", ut.Field, "want", ut.Type, "got", ut.Value)
}
```

`errors.As` is what makes typed errors usable through wrap chains.

### 3. Accumulating with `errors.Join`

```go
func validate(form Form) error {
	var errs []error
	if form.Name == "" {
		errs = append(errs, errors.New("name required"))
	}
	if !validEmail(form.Email) {
		errs = append(errs, fmt.Errorf("email %q: %w", form.Email, ErrInvalidEmail))
	}
	if form.Age < 0 {
		errs = append(errs, errors.New("age must be non-negative"))
	}
	return errors.Join(errs...) // nil if all OK
}

// Caller:
if err := validate(f); err != nil {
	for _, e := range err.(interface{ Unwrap() []error }).Unwrap() {
		fmt.Println("-", e)
	}
}
```

`errors.Join`'s nil-pruning means you can blindly append and the no-op case still returns `nil`.

### 4. Custom `Is` for category matching

```go
type HTTPErr struct{ Status int }

func (e *HTTPErr) Error() string { return http.StatusText(e.Status) }

func (e *HTTPErr) Is(target error) bool {
	t, ok := target.(*HTTPErr)
	if !ok { return false }
	if t.Status == 0 { return true } // any HTTPErr matches the wildcard
	return e.Status == t.Status
}

var (
	ErrAnyHTTP = &HTTPErr{}                 // wildcard
	ErrNotFnd  = &HTTPErr{Status: 404}
)

errors.Is(&HTTPErr{Status: 500}, ErrAnyHTTP) // true
errors.Is(&HTTPErr{Status: 500}, ErrNotFnd)  // false
```

### 5. `errors.As` to a behavior interface

```go
type Temporary interface{ Temporary() bool }

var t Temporary
if errors.As(err, &t) && t.Temporary() {
	retry()
}
```

This is exactly what `net.Error.Temporary()` was for (now deprecated in favor of `os.IsTimeout` / `errors.Is(err, context.DeadlineExceeded)`). The pattern itself remains useful for your own retryable/idempotent classifications.

## Anti-Patterns & Gotchas

**`err == target` after wrapping.** Always false. Use `errors.Is`.

**Custom `Is` that calls `errors.Is` on the target.** Infinite recursion. The contract is: compare your own state to `target`, return bool; do not re-enter the `errors` machinery.

**`errors.As` with the wrong pointer type.** `errors.As(err, perr)` where `perr` is `*os.PathError` (not `**os.PathError`) panics. Always pass `&x`, never `x`.

**Forgetting `%w`.** `fmt.Errorf("read: %v", err)` returns a plain `*errorString` with the message but no wrap link. `errors.Is`/`errors.As` cannot see through `%v`.

**Multiple `%w` confusion.** Pre-1.20, `fmt.Errorf("%w %w", a, b)` was an error. From 1.20, it's valid and produces a multi-wrap. Old code reviewers sometimes flag the new form as a bug.

**`errors.Join` for everything.** Joining unrelated errors makes the message unreadable. Use it when the errors are siblings of the same conceptual operation, not as a generic concatenator.

**Targeting a non-comparable type with `Is`.** `errors.Is(err, someStructWithSlice{})` panics. Make sentinels pointer-based.

**Sentinel proliferation.** Exporting 50 sentinels just to be granular often beats the same job done with one typed error carrying a `Code` field. See `04-sentinel-vs-typed-vs-opaque.md`.

**Returning `errors.Join(nil)`.** Returns `nil`, which surprises people who expected an empty-but-non-nil error. The behavior is documented and correct; surprise comes from not reading the docs.

## Performance Notes

- `errors.Is` is O(depth-of-chain). Typical chains are 1–5 deep; the per-link cost is one type assertion and one comparison.
- `errors.As` adds a reflect-based assignability check per link. Cheap (`reflect.Type` comparison), but measurably slower than a sentinel `errors.Is`.
- `errors.Join` allocates the wrapper plus the slice. The wrapper's `Error()` method allocates a `strings.Builder`-sized buffer to format. Don't call `.Error()` in hot paths; reserve formatting for logging.
- `fmt.Errorf("%w", ...)` allocates a `*fmt.wrapError` (or `*fmt.wrapErrors` for multi). Roughly 80–120 ns/op on x86-64.
- `errors.Unwrap` is one interface assertion + one method call. Negligible.

Cost-sensitive APIs (parsers, packet handlers) often skip `fmt.Errorf` entirely and return pre-allocated sentinels.

## How Big Companies Use It

- **Kubernetes `k8s.io/apimachinery/pkg/api/errors`** layers `IsNotFound`, `IsAlreadyExists`, etc. on top of typed `*StatusError`. These predicates predate `errors.Is` and are kept for back-compat; new code increasingly switches to `errors.Is(err, apierrors.ErrXxx)` style.
- **Caddy** uses `errors.Is(err, fs.ErrNotExist)` in its file-server handler to distinguish 404 from real failures: https://github.com/caddyserver/caddy.
- **etcd** wraps gRPC errors with `errors.As(err, &rpctypes.EtcdError)` to extract structured codes that survive transport.
- **CockroachDB** wraps every error with its own `errors.WithStack`, `errors.WithDetail`, `errors.WithHint` from `github.com/cockroachdb/errors`, then exposes `Is`/`As`-compatible chains so callers can use stdlib operators.
- **gRPC-Go status errors**: `status.FromError(err)` is essentially `errors.As(err, &*status.Status)` plus fallbacks.

## Source Code References

Pinned to `go1.26`.

- `errors.New` and `*errorString`: [`src/errors/errors.go`](https://github.com/golang/go/blob/master/src/errors/errors.go).
- `errors.Is`, `errors.As`, `errors.Unwrap`: [`src/errors/wrap.go`](https://github.com/golang/go/blob/master/src/errors/wrap.go).
- `errors.Join` and its `joinError` type: [`src/errors/join.go`](https://github.com/golang/go/blob/master/src/errors/join.go).
- `fmt.Errorf` and `%w` handling: [`src/fmt/errors.go`](https://github.com/golang/go/blob/master/src/fmt/errors.go).
- `syscall.Errno.Is` implementation: [`src/syscall/syscall_unix.go`](https://github.com/golang/go/blob/master/src/syscall/syscall_unix.go) (search `func (e Errno) Is`).
- `os.PathError`/`fs.PathError` and its `Unwrap`: [`src/io/fs/fs.go`](https://github.com/golang/go/blob/master/src/io/fs/fs.go).

Go source is BSD-3 licensed; cite when snipping.

## Further Reading

- Spec, "Errors": https://go.dev/ref/spec#Errors.
- Go blog, "Working with Errors in Go 1.13": https://go.dev/blog/go1.13-errors — definitive on `Is`/`As`/`%w`.
- Go blog, "Working with multiple errors" (2023): https://go.dev/blog/errors-join.
- Proposal: "errors: add Wrap, Is, As" (Damien Neil, 2019): https://github.com/golang/go/issues/29934.
- Proposal: "errors: add Join" (Damien Neil et al., 2022): https://github.com/golang/go/issues/53435.
- Russ Cox, design retrospective on the error proposal: https://research.swtch.com/go2019.
- Dave Cheney, "Inspecting errors": https://dave.cheney.net/2014/12/24/inspecting-errors.
- pkg.go.dev: https://pkg.go.dev/errors.

## Exercises / Self-Check

1. Write a type `MultiCause` that wraps two errors and implements `Unwrap() []error`. Verify `errors.Is` finds either child. Then add a primary/secondary distinction so `Error()` reports the primary first.
2. `errors.As(err, &target)` panics when called wrong. List three calling shapes that panic and explain each.
3. Given `err := fmt.Errorf("A: %w; B: %w", io.EOF, os.ErrNotExist)`, what does `errors.Unwrap(err)` return? Why?
4. Implement a custom `Is` for an `*HTTPErr` so that `errors.Is(&HTTPErr{500}, &HTTPErr{0})` matches as a wildcard. What invariant must `Is` maintain to avoid infinite loops?
5. Benchmark `errors.Is(err, sentinel)` against `err == sentinel`. At what chain depth does `Is` cost 10× more than `==`?
