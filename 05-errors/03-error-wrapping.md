# Error Wrapping — `%w`, `Unwrap`, and the Wrap Chain

## TL;DR

Wrapping an error means returning a new error that *contains* the original, accessible via `Unwrap()`. The `%w` verb in `fmt.Errorf` is the idiomatic constructor; since 1.20 you can use it multiple times in one call. A wrap chain lets callers add layered context (`"open config: read file: permission denied"`) while preserving identity (`errors.Is(err, fs.ErrPermission)`). The discipline is: **wrap once per layer with one fragment of context, never wrap silently, never wrap without `%w` if the cause matters**.

## Mental Model

```
fmt.Errorf("open %s: %w", path, fs.ErrPermission)

constructs:

    *fmt.wrapError {
        msg: "open /etc/cfg: permission denied",
        err: fs.ErrPermission,
    }

Unwrap() error → returns fs.ErrPermission

Chain of three layers:

    *wrapError{ "service start: ", *wrapError{ "open /etc/cfg: ", fs.ErrPermission } }
                       │                              │                   │
                       └─ added by main               └─ added by config  └─ original
```

Each `%w` introduces one chain link. `errors.Is`/`errors.As` walk the chain. Reading the final `.Error()` produces a human-readable string of all layers joined by `": "`.

## Syntax & Basic Usage

```go
package main

import (
	"errors"
	"fmt"
	"io/fs"
)

func readConfig(path string) error {
	// Pretend we tried to open the file and got a real error.
	return fmt.Errorf("read config %q: %w", path, fs.ErrPermission)
}

func start() error {
	if err := readConfig("/etc/app.cfg"); err != nil {
		return fmt.Errorf("startup: %w", err)
	}
	return nil
}

func main() {
	err := start()
	fmt.Println(err)
	fmt.Println("is perm:", errors.Is(err, fs.ErrPermission))
	// Output:
	// startup: read config "/etc/app.cfg": permission denied
	// is perm: true
}
```

## Deep Dive

### `%w` versus `%v` versus `%s`

```go
%v  → formats the error value; chain link NOT created
%s  → same as %v for errors
%w  → formats AND embeds the wrapped error in the chain
```

If you write `fmt.Errorf("ctx: %v", inner)`, the resulting error is a plain `*errorString`. `errors.Is(out, inner)` returns false. If you write `fmt.Errorf("ctx: %w", inner)`, the resulting error is a `*fmt.wrapError` whose `Unwrap()` returns `inner`. This is the one decision every layer makes: do you want callers to be able to inspect the cause, or are you collapsing it to a string?

Rule of thumb: **almost always `%w`**. Use `%v` only when you specifically want to hide the cause (e.g., sanitizing across a trust boundary).

### Single-`%w` constructor

```go
// fmt.Errorf turns one %w into *fmt.wrapError
err := fmt.Errorf("step %d: %w", n, cause)

// Equivalent hand-rolled:
type wrapErr struct {
	msg string
	err error
}
func (w *wrapErr) Error() string  { return w.msg }
func (w *wrapErr) Unwrap() error  { return w.err }
```

`fmt.Errorf` enforces at most one `%w` *per format directive position* in older Go; since 1.20 multiple are allowed.

### Multi-`%w` constructor (since 1.20)

```go
err := fmt.Errorf("primary: %w; cleanup: %w", primary, cleanup)
// Backing type: *fmt.wrapErrors with Unwrap() []error
```

The wrapper implements `Unwrap() []error`, the multi-error variant `errors.Is`/`errors.As` understand. Use this when *both* errors are independently meaningful — typically the original failure plus an error from the deferred cleanup that ran afterward.

```go
func doWork(r io.ReadCloser) (err error) {
	defer func() {
		if cerr := r.Close(); cerr != nil {
			err = fmt.Errorf("%w; close: %w", err, cerr)
		}
	}()
	// ... use r ...
	return process(r)
}
```

### Custom types with `Unwrap() error`

```go
type StepErr struct {
	Step string
	Err  error
}

func (e *StepErr) Error() string { return e.Step + ": " + e.Err.Error() }
func (e *StepErr) Unwrap() error { return e.Err }
```

Useful when the wrapper carries structured fields (not just a string) you want callers to extract via `errors.As`.

### Custom types with `Unwrap() []error` (since 1.20)

```go
type MultiErr struct{ Errs []error }

func (m *MultiErr) Error() string {
	parts := make([]string, len(m.Errs))
	for i, e := range m.Errs { parts[i] = e.Error() }
	return strings.Join(parts, "; ")
}
func (m *MultiErr) Unwrap() []error { return m.Errs }
```

`errors.Is`/`errors.As` walk all children. Note: `errors.Unwrap(m)` returns `nil` because the package-level function only knows the single form — that's by design.

### Wrapping does not chain by default

Defining `Error()` on your type does **not** automatically wrap anything. You must add `Unwrap()` explicitly:

```go
// Holds an inner error but is INVISIBLE to errors.Is — no Unwrap method.
type Bad struct{ Inner error }
func (b *Bad) Error() string { return "outer: " + b.Inner.Error() }

errors.Is(&Bad{Inner: io.EOF}, io.EOF) // false
```

Always pair the `Error()` that mentions the inner with `Unwrap()` that exposes it.

### Where the context fragment goes

Convention: the context fragment goes **before** `%w`, ending in `": "`. The `%w` formats the inner error, which already begins with a lowercase letter and contains no terminal punctuation. The result chains cleanly:

```
"startup: open /etc/cfg: permission denied"
```

If every layer added "Error: " or "Failure: " or capitalized prefixes, the chain would read awkwardly. The lowercase-no-punctuation convention exists precisely so wraps compose.

### What context to add

Add exactly the information *this layer* uniquely knows. A SQL driver returning `sql.ErrNoRows` already says "no rows" — your store layer's job is to add the *query intent* ("get user X: …"), not to restate "no rows". The HTTP handler then adds the *request scope*.

```go
// Storage layer:
return fmt.Errorf("get user %q: %w", id, err) // adds "which user"

// Service layer:
return fmt.Errorf("auth login: %w", err)      // adds "which operation"

// HTTP layer: usually maps to a status, no further wrapping needed
```

Avoid layers that add no new information (`return fmt.Errorf("error: %w", err)` is pure noise).

### Wrapping breaks `==`

```go
err := fmt.Errorf("ctx: %w", io.EOF)
err == io.EOF                  // false
errors.Is(err, io.EOF)         // true
```

This is the most-quoted reason to always use `errors.Is` in code that touches wrapped errors.

### Wrapping and JSON / serialization

The stdlib has no built-in JSON encoding for errors. The chain is a runtime structure; you cannot serialize and round-trip it with `encoding/json` and expect identity to survive. Libraries that need wire-format errors (gRPC, JSON-RPC, REST APIs) define their own structured error types with codes/messages.

When you do log, structured loggers like `log/slog` write the error's `.Error()` string. The chain is collapsed to one string for the log. If you want individual fragments separately, log them with separate keys (`slog.String("op", "open"); slog.String("path", path); slog.Any("err", err)`).

### Wrapping and stack traces

The stdlib does **not** capture stack traces when wrapping. `fmt.Errorf("%w", err)` records a string, not a `runtime.Caller`. If you want stack traces, you either:

1. Use a third-party error library (`github.com/cockroachdb/errors`, `github.com/pkg/errors`, `github.com/go-errors/errors`).
2. Capture them yourself in custom error types using `runtime.Callers`.
3. Rely on the panic mechanism (separate page).

Russ Cox's reasoning: stack traces are *useful* but they tempt overuse, balloon allocations, and discourage descriptive context. The stdlib stayed minimal.

### Inspecting a chain (debugging)

```go
for cur := err; cur != nil; cur = errors.Unwrap(cur) {
	fmt.Printf("%T: %s\n", cur, cur)
}
```

For multi-wrap nodes, manually type-assert to `interface{ Unwrap() []error }`. There is no `errors.Walk` in the stdlib; write the walker if you need it.

### `errors.Unwrap` and the multi case

`errors.Unwrap(err)` returns `nil` if `err`'s `Unwrap` is the multi form. This is intentional — the single function can't return a slice — and is the reason most code uses `errors.Is`/`errors.As` rather than walking by hand.

## Standard Library Hooks

- `fmt.Errorf` with `%w` (single since 1.13, multi since 1.20).
- `errors.Unwrap`, `errors.Is`, `errors.As`, `errors.Join`.
- `*os.PathError`, `*os.LinkError`, `*os.SyscallError` — wrap `syscall.Errno` and friends.
- `*net.OpError`, `*net.DNSError` — wrap the underlying cause; expose it via `Unwrap`.
- `*url.Error` — wraps the transport error from `net/http`.
- `*exec.ExitError` — wraps a `*os.ProcessState`; `Stderr` field holds captured output.
- `*tls.RecordHeaderError`, `*tls.CertificateVerificationError`.
- `*json.SyntaxError`, `*json.UnmarshalTypeError`.
- `database/sql` does NOT generally wrap driver errors — be careful, you often see raw driver errors.

## Real-World Patterns

### 1. Wrap-at-every-layer with terse context

```go
package payments

import (
	"context"
	"errors"
	"fmt"
)

func (s *Service) Charge(ctx context.Context, userID, sku string) error {
	user, err := s.users.Get(ctx, userID)
	if err != nil {
		return fmt.Errorf("get user %q: %w", userID, err)
	}
	price, err := s.catalog.Price(ctx, sku)
	if err != nil {
		return fmt.Errorf("price %q: %w", sku, err)
	}
	if err := s.gateway.Capture(ctx, user.CardID, price); err != nil {
		return fmt.Errorf("capture %s: %w", price, err)
	}
	return nil
}
```

A failure surfaces as `"price \"sku-1\": http 502: bad gateway"`. The caller still does `errors.Is(err, ErrCardDeclined)` to decide whether to retry.

### 2. Cleanup wrap on deferred close (1.20+)

```go
func parse(path string) (_ *Doc, err error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer func() {
		if cerr := f.Close(); cerr != nil {
			if err == nil {
				err = fmt.Errorf("close %s: %w", path, cerr)
			} else {
				err = fmt.Errorf("%w; close %s: %w", err, path, cerr)
			}
		}
	}()
	return decode(f)
}
```

If both the read and the close fail, both are visible to `errors.Is`. If only one fails, the caller sees a normal single-wrap chain.

### 3. Typed wrapper to expose structured fields

```go
type RetryableErr struct {
	After time.Duration
	Err   error
}

func (e *RetryableErr) Error() string { return fmt.Sprintf("retry after %s: %s", e.After, e.Err) }
func (e *RetryableErr) Unwrap() error { return e.Err }

func (s *Service) Call() error {
	if err := s.upstream.Do(); err != nil {
		if rl, ok := err.(*RateLimitErr); ok {
			return &RetryableErr{After: rl.Retry, Err: err}
		}
		return err
	}
	return nil
}

// Caller:
var rerr *RetryableErr
if errors.As(err, &rerr) {
	time.Sleep(rerr.After)
	return s.Call()
}
```

The wrapper *adds* a field (`After`) that callers can extract without parsing strings.

### 4. Selective unwrap — present the cause without leaking internals

```go
// Inside an HTTP handler, mask internal details across a trust boundary
if err := svc.Do(); err != nil {
	logger.Error("svc.Do failed", "err", err) // full chain to logs
	if errors.Is(err, ErrNotFound) {
		http.Error(w, "not found", http.StatusNotFound)
	} else {
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}
```

Wrap chain stays intact internally; the user-facing message is sanitized.

### 5. Walker for diagnostics

```go
func chain(err error) []error {
	var out []error
	for err != nil {
		out = append(out, err)
		switch x := err.(type) {
		case interface{ Unwrap() error }:
			err = x.Unwrap()
		case interface{ Unwrap() []error }:
			out = append(out, x.Unwrap()...)
			return out
		default:
			return out
		}
	}
	return out
}
```

Useful inside debug endpoints, not on the hot path.

## Anti-Patterns & Gotchas

**Wrapping with `%v` and being confused that `errors.Is` returns false.** `%v` does not wrap; use `%w`.

**Double-wrapping with the same context.**

```go
if err := step(); err != nil {
	log.Println("step failed:", err)
	return fmt.Errorf("step failed: %w", err) // "step failed" twice in the log
}
```

Pick one. Almost always: don't log, just return.

**Wrapping an `error` that's already wrapped in a way that re-states the cause.**

```go
return fmt.Errorf("open: %w (caused by %s)", err, err) // "caused by" duplicates the chain
```

Trust the chain. One `%w`, end of story.

**Wrapping `nil`.**

```go
return fmt.Errorf("step: %w", nil) // returns a non-nil error wrapping nil
```

The `nil` is preserved by `Unwrap()` (returns nil), but the outer error reads `"step: %!w(<nil>)"` — clearly a bug. Always check the inner before wrapping.

**Multi-`%w` with mixed concerns.** `fmt.Errorf("%w; %w", primaryErr, totallyUnrelatedErr)` is technically legal but produces an error that's hard to reason about. Use `errors.Join` if the errors are siblings; use multi-`%w` only for genuinely related primary+secondary pairs (operation+cleanup).

**Adding a wrapping layer for "consistency" with no new information.**

```go
return fmt.Errorf("error: %w", err) // pure noise
```

Either add real context (op name, key, file, identifier) or return the inner directly.

**Custom wrapper without `Unwrap`.** Forgetting `Unwrap` makes the wrapper opaque to `errors.Is`/`errors.As`. The first thing to check when "`errors.Is` doesn't find my sentinel" is whether each wrapper in the chain implements `Unwrap`.

**Wrapping that hides programmer errors.** `fmt.Errorf("decode: %w", err)` wrapping an `out of memory` is misleading. Some failures (panics, OOM, asserts) should propagate unmolested.

**Capitalizing or terminating wrap context.** `fmt.Errorf("Decode FAILED.: %w", err)` produces awkward concatenations. Lowercase, no trailing period, just context + `: `.

**Wrap chain depth > 7-ish.** Beyond that, the message becomes unreadable and `errors.Is` walks slow down. If you're wrapping at every method, that's too many layers.

## Performance Notes

- `fmt.Errorf("ctx: %w", err)` allocates the `*fmt.wrapError` (16 bytes) plus the formatted message string. Roughly 80–120 ns/op.
- Multi-`%w` allocates the wrapper plus the `[]error` slice plus the message. Comparable cost to a single wrap when N=2.
- `errors.Join(a, b)` is similar: one allocation for the join wrapper, one for the slice.
- A wrap chain of N links is O(N) memory and O(N) traversal for `errors.Is`/`errors.As`. Negligible at N<10.
- Calling `.Error()` on a deep chain re-formats every layer; if you log it many times, cache the string.
- Hot paths returning errors *should* return pre-allocated sentinels rather than `fmt.Errorf` whenever possible. Wrap once at the boundary.

A useful benchmark:

```go
func BenchmarkWrap(b *testing.B) {
	for b.Loop() {
		_ = fmt.Errorf("ctx: %w", io.EOF)
	}
}
```

≈ 100 ns/op, 2 allocs (one for the wrapper, one for the message).

## How Big Companies Use It

- **Kubernetes** wraps internal errors using `fmt.Errorf("%w", ...)` heavily; the `controller-runtime` library uses wrap chains so reconcilers can distinguish transient (`apierrors.IsConflict`) from terminal failures.
- **CockroachDB** uses its own `errors.Wrap`/`errors.WithDetail`/`errors.WithHint` which preserve stack traces and structured details across nodes. Reasoning: a distributed SQL engine needs to send rich error info over RPC, which stdlib cannot do. See https://github.com/cockroachdb/errors.
- **HashiCorp Vault** wraps with context for audit trails — every error includes the path, token policy, and remote address.
- **Docker / Moby** uses `errdefs` to *categorize* errors (NotFound, Conflict, Forbidden) by wrapping; the categorization survives transport over the Docker API.
- **gRPC-Go** uses `status.Errorf(codes.NotFound, "user %s", id)` plus `errors.Is(err, status.Error(codes.NotFound, ""))` to combine wrap-style identity with explicit RPC status codes.
- **Tailscale's `tsnet`** wraps OS errors with `tsnet:` prefixes so log scraping can attribute failures to the right subsystem without spelunking into stack traces.

## Source Code References

Pinned to `go1.26`.

- `fmt.Errorf` implementation: [`src/fmt/errors.go`](https://github.com/golang/go/blob/master/src/fmt/errors.go) — read the whole file, it's short.
- `*fmt.wrapError` and `*fmt.wrapErrors` types: same file.
- `errors.Unwrap` for both forms: [`src/errors/wrap.go`](https://github.com/golang/go/blob/master/src/errors/wrap.go).
- `*os.PathError.Unwrap`: [`src/io/fs/fs.go`](https://github.com/golang/go/blob/master/src/io/fs/fs.go).
- `*url.Error.Unwrap`: [`src/net/url/url.go`](https://github.com/golang/go/blob/master/src/net/url/url.go).
- `*exec.ExitError`: [`src/os/exec/exec.go`](https://github.com/golang/go/blob/master/src/os/exec/exec.go).
- Proposal: "fmt: %w for multiple errors" (1.20): https://github.com/golang/go/issues/53435 (companion to `errors.Join`).

Go source is BSD-3 licensed; cite when copying.

## Further Reading

- Go blog, "Working with Errors in Go 1.13": https://go.dev/blog/go1.13-errors.
- Go blog, "Working with multiple errors" (1.20): https://go.dev/blog/errors-join.
- Go release notes 1.13: https://go.dev/doc/go1.13#error_wrapping.
- Go release notes 1.20: https://go.dev/doc/go1.20 — search for `errors`.
- Dave Cheney, "Stack traces and the errors package": https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package.
- CockroachDB blog, "Errors Library": https://www.cockroachlabs.com/blog/error-handling-go/.
- Russ Cox, "Go 2 Error Inspection (Problem Overview)" (2018): https://go.googlesource.com/proposal/+/master/design/29934-error-values.md.
- Bryan C. Mills, "Go internals: error wrapping" (talk transcripts on go.dev/blog).

## Exercises / Self-Check

1. Write a function that opens a file and decodes JSON; on failure, wrap with the operation name and file path. Then write a test that asserts `errors.Is(err, fs.ErrNotExist)` for a missing file.
2. Demonstrate the multi-`%w` form in a deferred close. Show that both errors are visible via `errors.Is`.
3. Make a custom error type `*StepErr` with `Step string` and `Err error`. Implement `Unwrap`. Show that `errors.As(err, &stepErr)` works when wrapped twice through `fmt.Errorf`.
4. Build a `Chain(err)` walker that returns `[]error` flattening single and multi forms. Test it against a chain mixing `fmt.Errorf("%w", ...)`, `errors.Join(...)`, and a custom multi-wrapper.
5. Take a chain of depth 1, 5, 25. Benchmark `errors.Is(chain, sentinel)`. Plot the cost. Where does walking become measurable?
