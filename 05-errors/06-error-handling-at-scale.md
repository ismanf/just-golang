# Error Handling at Scale — Kubernetes, Docker, CockroachDB, gRPC

## TL;DR

Large Go codebases all confront the same problems the stdlib leaves open: typed structured errors that survive RPC; stack traces; redaction of PII; classification (`IsNotFound`, `IsRetryable`, `IsAuth`); mapping to HTTP/gRPC status codes; consolidating multi-source failures; and observability hooks for metrics and tracing. They each solved it slightly differently — Kubernetes with `*StatusError` + `IsXxx` predicates, Docker with the behavior-based `errdefs`, CockroachDB with the `cockroachdb/errors` superset, gRPC with `*status.Status`. The common pattern across all of them: **classify at the source, wrap with `%w` for chain integrity, surface a stable category at the boundary, and log the full chain only at the outermost edge**.

## Mental Model

```
                      (your code)
        ┌─────────────────────────────────────┐
        │   internal layer                    │
        │     return fmt.Errorf("%w", err)    │  ← wrap-and-bubble
        │                                     │
        │   service layer                     │
        │     return errors.Join(...)         │  ← combine partial failures
        │                                     │
        │   boundary layer (HTTP / gRPC)      │
        │     code := classify(err)           │  ← map chain to category
        │     log full chain ← once           │  ← observability
        │     return user-safe response       │  ← redact internals
        └─────────────────────────────────────┘
                      │
                      ▼
                  client sees:
                    - HTTP 404 / gRPC NotFound
                    - request-id for support
                    - sanitized message
```

The big-system patterns are layered repetitions of `02-errors-package.md` and `03-error-wrapping.md` applied with discipline.

## Syntax & Basic Usage

A minimal "scale-ready" error layout:

```go
package errx

import (
	"errors"
	"fmt"
)

// 1. Stable categorical codes — the public API for classification.
type Kind int

const (
	KindUnknown Kind = iota
	KindNotFound
	KindAlreadyExists
	KindInvalid
	KindUnauthorized
	KindForbidden
	KindConflict
	KindTransient
	KindInternal
)

// 2. Structured error type that participates in the wrap chain.
type Err struct {
	Kind    Kind
	Op      string            // logical operation, e.g., "store.GetUser"
	Fields  map[string]string // structured context for logs
	Err     error
}

func (e *Err) Error() string {
	if e.Err == nil {
		return fmt.Sprintf("%s: %s", e.Op, kindName(e.Kind))
	}
	return fmt.Sprintf("%s: %s: %s", e.Op, kindName(e.Kind), e.Err)
}
func (e *Err) Unwrap() error { return e.Err }

// 3. Predicates the boundary layer calls.
func Is(err error, k Kind) bool {
	var e *Err
	if errors.As(err, &e) {
		return e.Kind == k
	}
	return false
}

func kindName(k Kind) string { /* … */ return "" }
```

The shape is intentionally close to what every large project ended up with after a few years of iteration.

## Deep Dive

### Why the stdlib alone isn't enough at scale

`fmt.Errorf("%w", err)` plus `errors.Is`/`errors.As` covers identity and structure. It does **not** cover:

1. **Stack traces**: where did this error originate?
2. **Structured context**: machine-readable key/value, not just string fragments.
3. **Redaction**: production logs must not leak PII or secrets that user input may contain.
4. **Categorization for transport**: HTTP status codes, gRPC status codes, error codes for clients.
5. **Telemetry hooks**: counters, traces, alerts keyed by error category.
6. **Multi-error consolidation across boundaries**: `errors.Join` works in-process; sending five concurrent failures over a wire format is a separate problem.
7. **Round-tripping**: an error generated on server A, sent over RPC, decoded on client B, then re-wrapped — preserving identity/category.

Every large Go project adds layers for these. The good news: they converge.

### Kubernetes — typed `*StatusError` + `IsXxx` predicates

`k8s.io/apimachinery/pkg/api/errors` exposes one primary typed error:

```go
type StatusError struct {
	ErrStatus metav1.Status // structured: code, reason, details, message
}

func (e *StatusError) Error() string { return e.ErrStatus.Message }
func (e *StatusError) Status() metav1.Status { return e.ErrStatus }
```

And a fleet of predicates:

```go
func IsNotFound(err error) bool
func IsAlreadyExists(err error) bool
func IsConflict(err error) bool
func IsForbidden(err error) bool
func IsServerTimeout(err error) bool
// ...many more
```

Each predicate extracts the underlying `*StatusError` (handling wrapped chains via `errors.As`) and compares `ErrStatus.Reason` against a known constant.

**Why this design.** Kubernetes errors must round-trip through the API server, etcd, kubectl, and dozens of controllers. The `metav1.Status` is the wire format. The predicates give caller code a stable, readable shape independent of how errors are constructed. Predicates also predate `errors.Is` and remain the idiomatic call site.

**Lessons:**

- One typed error with an enum (`Reason`) beats 30 sentinels.
- Predicates are the ergonomic surface; underlying type is the source of truth.
- The wire format (`metav1.Status`) is what makes "errors over RPC" possible.

### Docker / Moby — `errdefs` behavior interfaces

`github.com/docker/docker/errdefs` defines a set of behavior interfaces:

```go
type ErrNotFound interface { error; NotFound() }
type ErrConflict interface { error; Conflict() }
type ErrForbidden interface { error; Forbidden() }
// ...
```

And predicates that test for them:

```go
func IsNotFound(err error) bool {
	_, ok := getImplementer(err).(ErrNotFound)
	return ok
}
```

Helpers wrap concrete errors to give them the right interface:

```go
err := errdefs.NotFound(fmt.Errorf("image %q not found", name))
```

The HTTP API layer in `moby` then dispatches purely on behavior:

```go
func httpStatusFromError(err error) int {
	switch {
	case errdefs.IsNotFound(err):    return http.StatusNotFound
	case errdefs.IsConflict(err):    return http.StatusConflict
	case errdefs.IsForbidden(err):   return http.StatusForbidden
	// ...
	default:                          return http.StatusInternalServerError
	}
}
```

**Why this design.** Docker's daemon has dozens of subsystems (image, container, network, volume). Each constructs its own concrete error types. Behavior interfaces let the HTTP layer classify without knowing any of them, and let subsystems evolve their internal types without breaking the boundary.

**Lessons:**

- Opaque-by-behavior scales beautifully: producers stay decoupled from consumers.
- One small package (`errdefs`) at the boundary is enough; the rest of the codebase stays plain.
- Wrappers like `errdefs.NotFound(err)` are how you "attach" a behavior to an existing typed error.

### CockroachDB — the `cockroachdb/errors` superset

`github.com/cockroachdb/errors` is the most comprehensive third-party errors library:

```go
err := errors.Wrap(srcErr, "executing distributed transaction")
err = errors.WithDetail(err, "txn-id: 0xabcd")
err = errors.WithHint(err, "Retry with a fresh transaction")
err = errors.WithStack(err) // captures runtime.Callers

// Inspect:
errors.IsAny(err, sql.ErrNoRows, ErrTxConflict)
errors.GetSafeDetails(err) // PII-redacted details for telemetry
errors.FlattenDetails(err) // for user output
```

Properties beyond stdlib:

- **Stack traces** captured by `WithStack` (cheap; only Callers, not full frame resolution).
- **Hints and details** — structured strings split into safe (telemetry-safe) and unsafe (may contain PII).
- **Safe redaction** via `redact.RedactableString`, integrating with CockroachLabs' redaction library.
- **Wire format**: encode/decode errors over gRPC with full chain preservation.
- **Compatibility**: implements `Unwrap`, `Is`, `As`; stdlib operators work over Cockroach errors and vice versa.

**Why this design.** CockroachDB is a distributed SQL engine. An error on node A may originate from a Postgres-compatible parse failure, propagate through query distribution, hit a transient KV-layer conflict on node B, and finally return to a psql client expecting a Postgres error code. Every layer needs to add context without losing the original cause, and the final output must redact tenant data from the operator's logs.

**Lessons:**

- For genuinely distributed systems, you outgrow the stdlib — but you stay *compatible* with it. Cockroach's errors satisfy `errors.Is`/`errors.As` perfectly.
- Stack traces are expensive but invaluable for cross-node debugging; capture lazily.
- Redaction must be a first-class concern, not a "we'll add that later."

### gRPC — `*status.Status` and the code/message split

`google.golang.org/grpc/status` exposes:

```go
import "google.golang.org/grpc/status"
import "google.golang.org/grpc/codes"

err := status.Errorf(codes.NotFound, "user %q not found", id)

// Server side: the framework converts errors at the RPC boundary.
// Client side:
if st, ok := status.FromError(err); ok {
	switch st.Code() {
	case codes.NotFound: handleMissing()
	case codes.Unavailable: retry()
	}
}
```

Properties:

- Wire-format errors: `codes.Code` enum + UTF-8 message + optional `Details` (protobuf Any).
- `status.FromError` is the canonical extractor, equivalent to `errors.As` for the status type.
- Compatible with `errors.Is`: `errors.Is(err, status.Error(codes.NotFound, ""))` works.

**Lessons:**

- When errors cross trust boundaries, a small enumerated code set (`codes`) beats free-form types.
- `Details` (protobuf Any) lets you attach structured supplementary info without expanding the enum.
- Treating "error" and "RPC status" as the same value type means client/server code uses one mechanism.

### HashiCorp — `hashicorp/go-multierror`

Predates `errors.Join` by years; still used in Terraform, Vault, Nomad:

```go
var result error
for _, plugin := range plugins {
	if err := plugin.Validate(); err != nil {
		result = multierror.Append(result, fmt.Errorf("plugin %s: %w", plugin.Name, err))
	}
}
return result
```

Adds an `ErrorFormat` hook so the joined output can be customized for the `terraform plan` UI. Mostly superseded by stdlib `errors.Join` for greenfield code; in legacy projects, swap is trivial because both implement `Unwrap() []error`.

### Boundary mapping in practice

The HTTP/gRPC boundary is where chain → category → response happens. A common skeleton:

```go
func writeError(w http.ResponseWriter, r *http.Request, err error) {
	// 1. Telemetry: count + trace.
	metrics.Inc("http.error", tagFor(err))
	span := trace.SpanFromContext(r.Context())
	span.RecordError(err)

	// 2. Log the full chain ONCE.
	requestID := middleware.GetReqID(r.Context())
	slog.Error("http error",
		"method", r.Method, "path", r.URL.Path, "request_id", requestID,
		"err", err, // stringifies the chain
		// "stack", string(debug.Stack()), // optional
	)

	// 3. Classify and respond.
	status, msg := classify(err)
	// Optionally include the request_id so support can correlate.
	http.Error(w, fmt.Sprintf("%s (request_id=%s)", msg, requestID), status)
}

func classify(err error) (int, string) {
	switch {
	case errors.Is(err, store.ErrNotFound):       return 404, "not found"
	case errors.Is(err, store.ErrConflict):       return 409, "conflict"
	case errors.Is(err, auth.ErrUnauthorized):    return 401, "unauthorized"
	case errors.Is(err, validate.ErrBadInput):    return 400, "invalid input"
	case errors.Is(err, context.DeadlineExceeded):return 504, "timeout"
	default:                                       return 500, "internal error"
	}
}
```

Three things to internalize:

1. **Classify by category, not message.** Never `if strings.Contains(err.Error(), "duplicate key")`.
2. **Log the chain once, at the boundary.** Every internal layer wraps; the boundary is the one place that writes a full record.
3. **Return user-safe messages.** The user gets a category + a request ID. Internals stay in logs.

### Stack traces — when worth the cost

The stdlib doesn't capture stack traces because they're expensive and tempt overuse. Production systems that face complex async flow (queues, distributed work) usually capture stacks:

- **At the error origin** (when the typed error is first constructed), via `runtime.Callers`.
- **Lazily formatted** (`runtime.CallersFrames`) only when logged.
- **One per chain**, not one per wrap layer. Adding stacks at every layer multiplies cost without information.

If you don't already have a reason for traces, you don't need them. When you do, use a library (`cockroachdb/errors`, `pkg/errors`) rather than reinventing.

### Multi-error in distributed/concurrent code

```go
import "golang.org/x/sync/errgroup"

func processAll(ctx context.Context, items []Item) error {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8)
	for _, it := range items {
		it := it
		g.Go(func() error { return process(ctx, it) })
	}
	return g.Wait() // returns the first non-nil error; cancels others
}
```

`errgroup` *cancels siblings on first error*. If you need all errors:

```go
import "errors"
import "sync"

func processAll(ctx context.Context, items []Item) error {
	var (
		mu   sync.Mutex
		errs []error
		wg   sync.WaitGroup
	)
	wg.Add(len(items))
	for _, it := range items {
		it := it
		go func() {
			defer wg.Done()
			if err := process(ctx, it); err != nil {
				mu.Lock(); errs = append(errs, err); mu.Unlock()
			}
		}()
	}
	wg.Wait()
	return errors.Join(errs...)
}
```

Or use `sync.WaitGroup.Go` (1.25+) for less boilerplate. The pattern of `collect-then-join` is universal in batch/parallel processing.

### Redaction

Naive: log `err.Error()`. Problem: user email, query strings, internal IDs end up in production logs, sometimes shipped to third-party SaaS.

Patterns:

- **Construct errors with no PII**: `fmt.Errorf("validate input: %w", err)` instead of `fmt.Errorf("validate %s: %w", userEmail, err)`.
- **Attach safe vs unsafe details**: the cockroachdb/errors approach, where `GetSafeDetails(err)` returns only the bits marked safe for telemetry.
- **Redact at the logger**: `slog.Handler` can transform attributes; libraries like `cockroachdb/redact` integrate.

Pick one and apply uniformly. Mixed strategies leak.

### Error budgets and SLOs

Operationally, errors aren't just "bug or not"; they categorize differently for SLO accounting:

- **User errors** (400-class, client retried bad input) — don't count against the SLO.
- **Service errors** (500-class) — count.
- **Transient errors** (503/504, retried successfully) — may count partially.

Your error category needs to encode this. The HTTP/gRPC code does the job at the boundary; internally, a `Kind` enum (above) lets you split metrics:

```go
errCounter.WithLabelValues(kindName(kindOf(err))).Inc()
```

## Standard Library Hooks

- `errors.Is`, `errors.As`, `errors.Join`, `errors.Unwrap` — the foundation.
- `fmt.Errorf` with `%w` and multi-`%w` — the wrapping primitive.
- `golang.org/x/sync/errgroup` — cancel-on-first-error parallel execution.
- `golang.org/x/sync/singleflight` — coalescing duplicate requests (errors broadcast to all waiters).
- `log/slog` — structured logging; errors stringify via `Error()`, attach a chain via custom `LogValuer`.
- `context` — `context.Cause(ctx)` (1.20+) returns the cancellation cause, often used as the error to propagate.
- `runtime/debug.Stack` — for stack snapshots at recovery / panic points.

## Real-World Patterns

### 1. Per-package error helpers

```go
package store

import (
	"errors"
	"fmt"
	"yourorg/errx"
)

func wrap(op string, err error, kind errx.Kind) error {
	if err == nil { return nil }
	return &errx.Err{Kind: kind, Op: "store." + op, Err: err}
}

func (s *Store) Get(id string) (*Row, error) {
	row, err := s.db.QueryRow(...).Scan(...)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, wrap("Get", err, errx.KindNotFound)
	}
	if err != nil {
		return nil, wrap("Get", err, errx.KindInternal)
	}
	return row, nil
}
```

Every package has a `wrap` helper. Boilerplate small; consistency huge.

### 2. Boundary error mapper

```go
package httpapi

import (
	"errors"
	"net/http"
	"yourorg/errx"
)

func toHTTPStatus(err error) int {
	switch {
	case errx.Is(err, errx.KindNotFound):       return http.StatusNotFound
	case errx.Is(err, errx.KindAlreadyExists):  return http.StatusConflict
	case errx.Is(err, errx.KindInvalid):        return http.StatusBadRequest
	case errx.Is(err, errx.KindUnauthorized):   return http.StatusUnauthorized
	case errx.Is(err, errx.KindForbidden):      return http.StatusForbidden
	case errx.Is(err, errx.KindConflict):       return http.StatusConflict
	case errx.Is(err, errx.KindTransient):      return http.StatusServiceUnavailable
	case errors.Is(err, context.DeadlineExceeded): return http.StatusGatewayTimeout
	}
	return http.StatusInternalServerError
}
```

Two-place dependency: every internal package uses `errx.Err{Kind: ...}` to tag, the boundary uses `toHTTPStatus` to map.

### 3. gRPC error converter

```go
import "google.golang.org/grpc/status"
import "google.golang.org/grpc/codes"

func toGRPCStatus(err error) error {
	switch {
	case errx.Is(err, errx.KindNotFound):
		return status.Error(codes.NotFound, err.Error())
	case errx.Is(err, errx.KindAlreadyExists):
		return status.Error(codes.AlreadyExists, err.Error())
	// ...
	}
	return status.Error(codes.Internal, "internal error")
}
```

Server-side interceptor calls this on every returned error.

### 4. Retry middleware keyed on category

```go
type retryable interface{ Retryable() bool }

func WithRetry(op func() error) error {
	for n := 0; n < 5; n++ {
		err := op()
		if err == nil { return nil }
		if errors.Is(err, context.Canceled) { return err }
		var r retryable
		if !errors.As(err, &r) || !r.Retryable() {
			return err
		}
		time.Sleep(backoff(n))
	}
	return errors.New("retry: exhausted")
}
```

Retryable-ness is a property of the error, set when constructed (`&errx.Err{Kind: KindTransient}` could implement `Retryable() bool { return true }`).

### 5. Slog handler that flattens the chain

```go
type chainAttr struct{ err error }

func (c chainAttr) LogValue() slog.Value {
	var nodes []slog.Attr
	i := 0
	for cur := c.err; cur != nil; cur = errors.Unwrap(cur) {
		nodes = append(nodes, slog.String(fmt.Sprintf("layer_%d", i), cur.Error()))
		i++
	}
	return slog.GroupValue(nodes...)
}

slog.Error("op failed", "err", chainAttr{err: err})
```

Lets log aggregators index each layer separately.

## Anti-Patterns & Gotchas

**No category, just messages.** Without a stable categorical enum, the boundary layer cannot map to status codes without string-matching, and metrics turn into infinite-cardinality nightmares.

**Logging at every layer.** Each wrap layer logging the same error means one failure produces 6 log lines. Log once, at the boundary.

**Category mapping in 14 places.** If `IsNotFound`-style switches live in every handler, they drift. Centralize.

**Including PII in error messages.** "user 'foo@bar.com' not found" becomes 1M production log lines containing emails. Construct messages with category + opaque ID; log details separately as redacted attributes.

**Re-wrapping at the boundary.** The boundary's job is to log and respond, not to wrap further. Wrapping again obscures the original layer history.

**Custom error types in microservice APIs without a stdlib bridge.** If your internal `*Err` doesn't implement `Unwrap`, callers' `errors.Is(err, sql.ErrNoRows)` returns false. Always implement `Unwrap`.

**Treating gRPC `*status.Status` as the only error type.** It's the wire format. Internally, keep your richer typed errors; convert at the boundary. Inverting this loses information.

**`fmt.Errorf("v=%v", v)` where `v` is a request object.** Inadvertent dump of the entire request body into the error message → into logs.

**Multi-error chain with 50 entries.** `errors.Join` over millions of items produces an error whose `.Error()` is megabytes. Cap the count or summarize.

**Forgetting context.Canceled in retry loops.** Retrying when the user has cancelled wastes resources. Always: `if errors.Is(err, context.Canceled) { return err }` first.

**Mixing redacted and unredacted in logs.** If one log line includes `email` and another redacts it, scrubbing pipelines break. Pick one model.

**Two error libraries in one process.** Stdlib + `pkg/errors` + `cockroachdb/errors` in the same binary leads to chains where `errors.Is` partially works. Standardize on one.

## Performance Notes

- Per-error allocation count and bytes matter at high QPS. A boundary handler producing 5 wrap layers + a stack trace per request can be a non-trivial GC source.
- `cockroachdb/errors.WithStack` is cheap (Callers only); resolving frames into strings happens at log time.
- `errors.Join` over N items allocates the wrapper + a slice. For tiny N (≤ 4), negligible. For large N, consider a custom summarizing wrapper.
- Avoid string formatting in the error path until the boundary. `fmt.Errorf` is fine for context but reformat-on-log is wasteful — pass attributes to `slog` instead.
- HTTP recovery middleware that calls `debug.Stack()` on every panic is fine because panics are rare; doing it on every error is not.
- Category classification (`errors.As(err, &e)`) is one reflect-based assignability check per chain link. At 100k QPS with chain depth 5, that's 500k checks/sec — measurable but typically <1% of CPU.
- Pre-allocated category instances (`errMissingTenant := &Err{Kind: KindNotFound, Op: "tenant"}`) save allocations on hot paths where the context doesn't vary.

## How Big Companies Use It

- **Google / Kubernetes**: `apimachinery/pkg/api/errors` + the `metav1.Status` wire format. Custom controllers add their own typed errors and rely on the predicates. See https://github.com/kubernetes/apimachinery/tree/master/pkg/api/errors.
- **Docker / Moby**: `errdefs` behavior interfaces, no central typed errors. Each subsystem wraps its own errors with `errdefs.NotFound(...)` etc. See https://github.com/moby/moby/tree/master/errdefs.
- **CockroachDB**: `cockroachdb/errors` everywhere; integrates with `cockroachdb/redact` and the wire-format encoder for distributed errors. Reading their internal post-mortems shows error handling as a first-class design concern. See https://github.com/cockroachdb/errors and https://www.cockroachlabs.com/blog/error-handling-go/.
- **Tailscale**: a small `tsweb`/error helpers package; categories are HTTP-status-shaped. Reasoning: every error eventually becomes an HTTP response, so encode it that way from the start. See public engineering posts at https://tailscale.com/blog.
- **HashiCorp**: pre-stdlib `multierror` heavily used; recent code starts to migrate to `errors.Join`. The Terraform/`hashicorp/hcl` packages combine errors with rich diagnostics (`hcl.Diagnostics`) that double as both error chains and source-location-aware UI.
- **Cloudflare**: heavy use of `errors.Is`/`As` + a behavior interface called `Status() int` that maps to HTTP. Per public blog posts on the worker runtime.
- **Discord / Twitch**: pragmatic mix of stdlib + per-team helpers; consolidated into a single error library once the org grew enough to need uniform observability.
- **Uber**: `uber-go/multierr` (their multi-error predecessor); Uber Go style guide (https://github.com/uber-go/guide) is one of the better written specifications for production Go error handling, even if you disagree with parts.
- **gRPC ecosystem**: `*status.Status` as the boundary type; codes mapped to HTTP status by `grpc-gateway`. The combination is the de facto pattern for service-mesh-era Go.

## Source Code References

Pinned to `go1.26` for stdlib; third-party links pin to latest tagged releases.

- Kubernetes apimachinery errors: https://github.com/kubernetes/apimachinery/blob/master/pkg/api/errors/errors.go.
- Docker errdefs (interfaces): https://github.com/moby/moby/blob/master/errdefs/defs.go.
- Docker errdefs (helpers): https://github.com/moby/moby/blob/master/errdefs/helpers.go.
- CockroachDB errors library: https://github.com/cockroachdb/errors/tree/master/errbase.
- CockroachDB redactable strings: https://github.com/cockroachdb/redact.
- gRPC status: https://github.com/grpc/grpc-go/blob/master/status/status.go.
- HashiCorp go-multierror: https://github.com/hashicorp/go-multierror.
- Uber multierr (predecessor to `errors.Join`): https://github.com/uber-go/multierr.
- Stdlib `golang.org/x/sync/errgroup`: https://pkg.go.dev/golang.org/x/sync/errgroup.
- Stdlib `errors.Join`: [`src/errors/join.go`](https://github.com/golang/go/blob/master/src/errors/join.go).

All linked Go source is BSD-3 licensed. Third-party libraries carry their own licenses (most: Apache 2.0 or MIT); check before vendoring.

## Further Reading

- Go blog, "Working with multiple errors" (1.20): https://go.dev/blog/errors-join.
- Cockroach Labs, "Error handling and Go": https://www.cockroachlabs.com/blog/error-handling-go/.
- Kubernetes design — "Error handling and rejection": https://kubernetes.io/docs/concepts/overview/kubernetes-api/.
- "Errors are values" (Pike, 2015): https://go.dev/blog/errors-are-values.
- Dave Cheney, "Don't just check errors, handle them gracefully": https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully.
- Google Go style guide on errors: https://google.github.io/styleguide/go/decisions#errors.
- Uber Go style guide on errors: https://github.com/uber-go/guide/blob/master/style.md#errors.
- Filippo Valsorda, "Errors in production Go": various talks at GopherCon EU.
- "Go error handling at Twitch" (engineering blog): https://blog.twitch.tv (search for "error handling").
- gRPC status codes overview: https://grpc.github.io/grpc/core/md_doc_statuscodes.html.
- Russ Cox, design retrospective on `errors`: https://research.swtch.com/go2019.

## Exercises / Self-Check

1. Design a 4-layer call stack: HTTP handler → service → repository → database driver. Pick a single error category (`NotFound`) and trace where it gets tagged, wrapped, classified, and logged. Now do the same for `Transient` and explain how retry middleware interacts with each layer.
2. Take a small package of yours that returns plain `fmt.Errorf` everywhere. Introduce an `errx` package with a `Kind` enum, a wrap helper, and predicates. Convert one boundary handler to use it. Measure the diff in code volume.
3. Write a `slog.Handler` adapter that, given an `error` attribute, expands it into N attributes for each chain layer. Test with `errors.Join` and `fmt.Errorf("%w; %w", ...)`.
4. Compare `errors.Join` and `hashicorp/go-multierror` on the same workload. What does multierror give you that stdlib doesn't? Is it enough to justify the dependency?
5. Read the source of Docker's `errdefs/helpers.go` (under 100 lines). Replicate the same pattern in your own codebase: a producer wraps `Foo(err)`, a consumer tests `IsFoo(err)`. Why is producer/consumer decoupling stronger here than with a typed error?
