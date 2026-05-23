# Error Tracking — Sentry, Bugsnag Patterns

## TL;DR

Error tracking is a category of observability separate from logs, metrics, and traces. The job: **deduplicate similar errors across thousands of occurrences, attach actionable context (stack trace, breadcrumbs, environment), and notify the right human**. Sentry and Bugsnag are the dominant commercial SaaS; Highlight, Honeybadger, Rollbar, and Errortrace are alternatives. The Go ecosystem has well-maintained SDKs for all of them. Three patterns you must internalise: **wrap your panics with a recovery middleware** that captures the stack and reports — then re-panics or returns 500; **enrich errors with context** (`errors.Join`, `fmt.Errorf` with `%w`, or a structured error type) so the tracker can group them; and **attach breadcrumbs** (log-like trail of events leading to the error). The single biggest gotcha: **Go's idiomatic `if err != nil { return err }` is a black hole for error tracking unless you also capture once at a known boundary** (handler, scheduled job, message consumer). Capturing every wrapped error inside the call tree creates duplicate reports of the same event. Go 1.20+'s `errors.Join` and `errors.Is/As` make the unwrapping reliable; Go 1.23+'s stack-pinned errors (`runtime.Pinner`-related improvements) make traces more reliable in error tracking.

## Mental Model

```
        Your service                  Error tracker (Sentry/Bugsnag)
        ────────────                  ────────────────────────────
                                                │
   ┌──────────────────┐                         │
   │ Recovery         │                         │
   │ middleware       │── captures panic + ────►│  group by fingerprint
   │ + 500 response   │   stack + breadcrumbs    │  notify if new
   └──────────────────┘                          │
                                                 │
   ┌──────────────────┐                         │
   │ Handler-level    │── sentry.CaptureErr ───►│  link errors to releases
   │ "this is an      │   (err) + context       │  show user-impact %
   │ unexpected err"  │                         │
   └──────────────────┘                         │
                                                 │
   ┌──────────────────┐                         │
   │ Expected errors  │── log only, NOT ───────►│  (don't pollute the tracker)
   │ (404, 401, ...)  │   captured              │
   └──────────────────┘                         │
```

The key insight: **not every `error` is an error worth tracking.** Validation errors, expected 4xx, retryable failures — these belong in logs and metrics. The error tracker is for "something is wrong and a human should look."

## Setting Up Sentry

```bash
go get github.com/getsentry/sentry-go
```

```go
import (
    "log"
    "time"
    "github.com/getsentry/sentry-go"
)

func main() {
    err := sentry.Init(sentry.ClientOptions{
        Dsn:              "https://abcdef@sentry.io/123",
        Environment:      "prod",
        Release:          "billing@v1.4.2",
        TracesSampleRate: 0.1,                  // performance traces, 10%
        AttachStacktrace: true,
        EnableTracing:    false,                // unless you've decided to use Sentry for traces
        SampleRate:       1.0,                  // 100% of errors
        BeforeSend: func(e *sentry.Event, hint *sentry.EventHint) *sentry.Event {
            // Last-mile redaction — strip secrets that snuck in
            return scrubSecrets(e)
        },
    })
    if err != nil { log.Fatal(err) }
    defer sentry.Flush(2 * time.Second)  // critical for short-lived processes!

    // ... your application
}
```

Critical: **`sentry.Flush`** on shutdown. The SDK queues events in memory; without flush, you lose the last second or two of errors on `os.Exit`. For long-running servers this is less important; for CLI tools and jobs, it's essential.

## HTTP Recovery Middleware

```go
import (
    "net/http"
    "github.com/getsentry/sentry-go"
)

func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                hub := sentry.GetHubFromContext(r.Context())
                if hub == nil { hub = sentry.CurrentHub() }
                hub.WithScope(func(s *sentry.Scope) {
                    s.SetTag("route", r.URL.Path)
                    s.SetExtra("method", r.Method)
                    s.SetExtra("ua", r.UserAgent())
                    hub.RecoverWithContext(r.Context(), rec)
                })
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

Use Sentry's `sentryhttp.New(...).Handle(...)` to get this plus other defaults out-of-the-box. Order in the middleware chain matters: recovery should be **outermost** so it wraps every other middleware (auth, logging, etc.).

## Capturing Non-Panic Errors

```go
func processOrder(ctx context.Context, id string) error {
    if err := validate(id); err != nil {
        // Expected — log and return, do NOT capture
        slog.WarnContext(ctx, "validation failed", "id", id, "err", err)
        return err
    }
    if err := charge(ctx, id); err != nil {
        // Unexpected — capture
        hub := sentry.GetHubFromContext(ctx)
        hub.WithScope(func(s *sentry.Scope) {
            s.SetTag("order.id", id)
            hub.CaptureException(err)
        })
        return fmt.Errorf("charge: %w", err)
    }
    return nil
}
```

The rule: **capture once, at the place where you stop knowing what to do.** A library function that wraps an error and returns shouldn't capture; the top-level handler that returns 500 should.

## Breadcrumbs

Breadcrumbs are timestamped log-like events leading up to an error. They give context: "the user did X, then Y, then the error happened during Z."

```go
sentry.AddBreadcrumb(&sentry.Breadcrumb{
    Category: "auth",
    Message:  "user logged in",
    Level:    sentry.LevelInfo,
    Data:     map[string]interface{}{"user_id": uid},
})

// ... later, an error occurs ...
sentry.CaptureException(err)
// → report contains the breadcrumb trail
```

Patterns:
- HTTP middleware that logs every inbound request as a breadcrumb at INFO.
- Database wrapper that adds breadcrumbs for every query (carefully — don't include SQL with secrets).
- External API client that breadcrumbs every outbound call.

Sentry retains the **last 100 breadcrumbs per Hub** (configurable). The Hub is request-scoped via `sentry.GetHubFromContext`.

## Hubs and Scopes

The SDK's three layers:

- **Client** — connection, transport, DSN, sample rate.
- **Hub** — request-scoped state container. One per goroutine/request.
- **Scope** — attached to a Hub, contains tags, user, breadcrumbs, contexts.

```go
// Per-request hub
hub := sentry.CurrentHub().Clone()
r = r.WithContext(sentry.SetHubOnContext(r.Context(), hub))

// In handlers
hub := sentry.GetHubFromContext(r.Context())
hub.Scope().SetUser(sentry.User{ID: userID, Email: email})
```

The shared hub anti-pattern: using `sentry.CurrentHub()` everywhere leaks breadcrumbs and tags across requests because they're stored on the global default hub. Always clone or per-request.

## User & Tag Context

```go
hub.WithScope(func(s *sentry.Scope) {
    s.SetUser(sentry.User{ID: "42", Email: "a@b", IPAddress: "{{auto}}"})
    s.SetTag("plan", "enterprise")
    s.SetTag("region", "us-east-1")
    s.SetContext("device", map[string]interface{}{
        "browser": r.UserAgent(),
    })
    hub.CaptureException(err)
})
```

Tags are indexed and filterable in the UI. User context lets you compute "how many users were affected by this error?" — among the most useful metrics for prioritisation.

## Sentry Performance / Tracing

```go
span := sentry.StartSpan(ctx, "db.query",
    sentry.WithTransactionName("GET /users/:id"),
)
defer span.Finish()
// ... do work
```

If you've adopted OTel, you typically don't also use Sentry for tracing — pick one. Sentry's `sentryotel.NewSentrySpanProcessor()` bridges OTel traces into Sentry's UI for projects where Sentry is the primary observability tool.

## Bugsnag — A Brief Equivalence

Bugsnag's Go SDK has a near-identical mental model:

```go
import "github.com/bugsnag/bugsnag-go/v2"

bugsnag.Configure(bugsnag.Configuration{
    APIKey:          "...",
    ReleaseStage:    "production",
    AppVersion:      "v1.4.2",
})

defer bugsnag.AutoNotify()  // recover + notify on panic
```

Notify on demand:

```go
bugsnag.Notify(err,
    bugsnag.MetaData{"order": {"id": orderID}},
    bugsnag.User{Id: userID, Email: email},
)
```

Migrating between trackers is mechanical at the SDK level — both speak panic-recover + structured error metadata. The real lock-in is the UI, integrations (PagerDuty, Slack, GitHub), and historical data.

## Other Tools — One-Line Summaries

- **Highlight** (highlight.io) — open-source, self-hostable; combines errors + session replay + logs.
- **Rollbar** — older incumbent; strong on stack trace presentation.
- **Honeybadger** — simple, Ruby-first heritage; Go SDK is functional.
- **Errortrack** — used inside several big-tech companies, mostly closed.
- **Cloud-native**: Datadog, New Relic, AppDynamics — error tracking as a feature of broader APM.

## Error Grouping ("Fingerprinting")

Trackers group identical-or-similar errors into a single issue. Default fingerprint: top of stack + exception type. This works well for panics; mediocre for `fmt.Errorf` chains where the file/line is in your error helper, not the underlying problem.

Customise:

```go
hub.WithScope(func(s *sentry.Scope) {
    s.SetFingerprint([]string{"{{ default }}", "db.connection", err.Error()})
    hub.CaptureException(err)
})
```

`{{ default }}` keeps the default heuristics and appends your strings. Use a stable identifier (error code, not error message — messages with variable strings break grouping).

## Release Tracking

```go
sentry.Init(sentry.ClientOptions{
    Release: "billing@" + buildVersion + "+" + buildCommit,
})
```

`Release` ties errors to commits. New errors after a release flag as regressions; resolved errors that return in a later release are re-opened automatically.

Pair with **source-map upload** (or in Go, dSYM/symbol upload) so stack traces show source lines. Sentry CLI:

```bash
sentry-cli releases new billing@v1.4.2
sentry-cli releases set-commits billing@v1.4.2 --auto
sentry-cli upload-dif --org acme --project billing ./bin/billing
```

## Sampling Errors

```go
sentry.Init(sentry.ClientOptions{
    SampleRate: 0.1,  // 10% of errors reported
})
```

Avoid in most cases — errors are typically rare enough that 100% is fine. Use only if you're hitting quota and have many duplicates of the same error (in which case grouping should help anyway).

Smarter: filter in `BeforeSend`:

```go
BeforeSend: func(e *sentry.Event, hint *sentry.EventHint) *sentry.Event {
    if e.Exception != nil && e.Exception[0].Value == "context canceled" {
        return nil  // drop these
    }
    return e
}
```

## Structured Errors

```go
type AppError struct {
    Code    string
    Message string
    Cause   error
}

func (e *AppError) Error() string { return e.Code + ": " + e.Message }
func (e *AppError) Unwrap() error { return e.Cause }

// Sentry can use this via BeforeSend to set tags
BeforeSend: func(e *sentry.Event, hint *sentry.EventHint) *sentry.Event {
    if ae, ok := hint.OriginalException.(*AppError); ok {
        e.Tags["error.code"] = ae.Code
        e.Fingerprint = []string{ae.Code}
    }
    return e
}
```

Structured errors with stable codes make fingerprinting deterministic.

## Slog Integration

Connect `log/slog` errors to Sentry:

```go
type sentryHandler struct{ slog.Handler }

func (h sentryHandler) Handle(ctx context.Context, r slog.Record) error {
    if r.Level >= slog.LevelError {
        hub := sentry.GetHubFromContext(ctx)
        if hub != nil {
            // Extract err attr if present
            var err error
            r.Attrs(func(a slog.Attr) bool {
                if a.Key == "err" {
                    if e, ok := a.Value.Any().(error); ok { err = e }
                }
                return true
            })
            if err != nil {
                hub.CaptureException(err)
            } else {
                hub.CaptureMessage(r.Message)
            }
        }
    }
    return h.Handler.Handle(ctx, r)
}
```

Wire it up:

```go
slog.SetDefault(slog.New(sentryHandler{slog.NewJSONHandler(os.Stdout, nil)}))
slog.ErrorContext(ctx, "charge failed", "err", err)
// → JSON log + Sentry event
```

Pattern: `slog.Error` is the "you also want Sentry" verb; `slog.Warn` is "log only."

## Anti-Patterns & Gotchas

**Capturing every wrapped error.** N nested handlers, all calling `sentry.CaptureException(err)` for the same root cause → N duplicate issues. Capture once, at the boundary.

**Capturing context cancellation.** `context.Canceled` is normal cleanup. Filter in `BeforeSend`.

**Capturing 4xx HTTP responses.** They're expected. Tracker should reflect *5xx + panics + cron failures*, not "user typed bad input."

**Logging the error and capturing it.** Pick one rule; otherwise grep for errors in logs and Sentry diverges. Convention: logs = mechanical record; Sentry = "this is a bug."

**Sharing the default Hub across goroutines.** Cross-request bleed of tags/breadcrumbs. Always clone per request.

**Setting `Release` to a hard-coded constant.** Then "regression in v1.4.2" never appears as a regression because you never published v1.4.3 to the tracker.

**Not flushing on shutdown.** Lost events.

**`AttachStacktrace: false` for non-panic captures.** You'll get the error message but no location. Always `true`.

**Putting secrets in breadcrumbs.** Especially DB queries with values. Sanitise.

**Capturing in tight loops.** A bad consumer that errors on every message floods the tracker. Add a circuit breaker.

**Treating Sentry as a metrics system.** It's not. Don't capture "user signed up" or "checkout completed" — those are events for an analytics pipeline.

**Bypassing the tracker for "library" errors.** Your DB wrapper's "connection refused" is precisely what should be captured — *once, at the application boundary*.

**Failing CI when Sentry is down.** Treat as best-effort; the build shouldn't break because the tracker is.

**Not setting up source maps / debug symbols.** Stack traces show `[fn0001]` and you can't fix the bug. Upload symbols on every release.

**Logging stack on `errors.Wrap`-style chains then capturing at top.** Sentry's grouping uses the *innermost* exception's stack; ensure your wrapping preserves the original via `%w`.

## Performance Notes

- **`sentry.CaptureException(err)`**: ~10–50 µs; serializes + queues. Network is async.
- **Breadcrumb addition**: <1 µs.
- **Recovery middleware overhead**: negligible until a panic actually happens.
- **`BeforeSend` callback**: runs on every event; keep it fast or move heavy work to a background goroutine.
- **Memory**: the SDK retains the last 100 breadcrumbs per Hub (~10 KB each = ~1 MB upper bound).
- **Network**: errors are small (~5–20 KB each). 1000 errors/day ≈ 20 MB.

## How Big Companies Use It

- **GitHub** uses Sentry across most products; Go services capture via the standard SDK with strict per-handler middleware patterns.
- **Stripe** built internal error tracking before vendor tools matured; convention since: every panic = paging event.
- **Shopify** uses Bugsnag historically; Go services adopted Sentry as their Go fleet grew.
- **Cloudflare** uses Sentry for many Go services; for the highest-volume edge they ship to internal pipelines (volume would exceed Sentry's quota economics).
- **Discord** uses Sentry for backend Go services with custom fingerprinting based on error codes.
- **Tailscale** uses a lightweight internal tracker plus reproducible-stack-via-pprof patterns; Sentry for their control plane.
- **Notion** uses Sentry with detailed `slog`-to-Sentry bridging.
- **Linear** uses Sentry tightly integrated with their deployment pipeline (regressions auto-tagged).

## Source Code References

- Sentry Go SDK: https://github.com/getsentry/sentry-go.
- sentryhttp middleware: https://github.com/getsentry/sentry-go/tree/master/http.
- sentryotel bridge: https://github.com/getsentry/sentry-go/tree/master/otel.
- Bugsnag Go SDK: https://github.com/bugsnag/bugsnag-go.
- Highlight Go: https://github.com/highlight/highlight/tree/main/sdk/highlight-go.
- Honeybadger Go: https://github.com/honeybadger-io/honeybadger-go.
- Rollbar Go: https://github.com/rollbar/rollbar-go.
- Go `errors` package (1.20+ Join): https://pkg.go.dev/errors.

## Further Reading

- "Error tracking best practices" (Sentry blog series): https://blog.sentry.io/.
- "12-Factor App XII: Admin Processes" — touches on capturing failures in batch jobs.
- "Errors are values" (Rob Pike): https://go.dev/blog/errors-are-values.
- "Don't just check errors, handle them gracefully" (Dave Cheney): https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully.
- "Working with Errors in Go 1.13" (Damien Neil, Jonathan Amsterdam): https://go.dev/blog/go1.13-errors.
- Charity Majors, *Observability Engineering* — chapter on error vs. metric vs. trace.

## Exercises / Self-Check

1. Wire Sentry into a Go HTTP server with a recovery middleware. Trigger a panic; confirm the event appears with the request URL and method as tags.
2. Add breadcrumbs to every DB query (carefully — no secret values). Trigger an error; confirm the breadcrumb trail shows the queries leading up to it.
3. Implement a `slog.Handler` that captures `slog.LevelError` records as Sentry events. Verify by emitting an error log; confirm Sentry sees it.
4. Add release tracking: `Release: app@<git-sha>`. Cause a regression in a deploy; observe it auto-tagged.
5. Set up custom fingerprinting by stable error code (`AppError.Code`). Confirm two different messages with the same code are grouped.
6. Filter out `context.Canceled` and `context.DeadlineExceeded` in `BeforeSend`. Confirm they no longer appear in the tracker.
7. Capture an error in a goroutine spawned from a handler. Use `sentry.GetHubFromContext(ctx)` (or clone) so tags from the request are preserved.
8. For a CLI tool that exits, add `defer sentry.Flush(2*time.Second)`. Crash deliberately; confirm the event still arrives.
