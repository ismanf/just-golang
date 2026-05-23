# Panic and Recover — When (Rarely) to Use Them

## TL;DR

`panic` unwinds the goroutine's stack, running deferred functions on the way; `recover` (only valid inside a deferred function) stops the unwind and yields the panic value. Panics are for *unrecoverable programmer errors* (nil dereferences, out-of-bounds, "this can't happen"), package init failures, and a handful of legitimate cross-stack control transfers (parser bailouts, server handlers). They are **not** the error mechanism — that's the `error` interface. The discipline: panic in the rarest of cases, recover only at goroutine entry points, and never use them to imitate try/catch.

## Mental Model

```
goroutine stack (top is current)

    ┌──────────────────┐
    │  panic("boom")   │  ← raises a *runtime._panic on g
    ├──────────────────┤
    │  f()             │  ← unwinds, runs defers (newest first)
    │   defer g()      │
    │   defer h()      │
    ├──────────────────┤
    │  caller          │  ← keeps unwinding through callers
    └──────────────────┘
       │
       ▼
    If a deferred function calls recover():
       - returns the panic value as any
       - panicking flag cleared on the goroutine
       - normal flow resumes from the recover-er's return
    If no recover():
       - stack walk reaches main / goroutine top
       - runtime prints the panic + tracebacks of all goroutines
       - process exits with status 2
```

Panic and recover are **goroutine-local**. A panic in goroutine X cannot be recovered in goroutine Y. If goroutine X doesn't recover, the whole process dies.

## Syntax & Basic Usage

```go
package main

import "fmt"

func safe(fn func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic: %v", r)
		}
	}()
	fn()
	return nil
}

func main() {
	err := safe(func() { panic("kaboom") })
	fmt.Println(err)
	// Output:
	// panic: kaboom
}
```

The `recover()` call only returns non-nil when called *directly* from a deferred function during a panic. Otherwise it returns `nil`.

## Deep Dive

### What can be passed to `panic`

`panic` takes an `any`. Convention is to pass an `error` (or a string for ad-hoc panics):

```go
panic(errors.New("invariant: empty key"))
panic("unreachable")
panic(&InvariantViolation{Op: "checkInvariants"})
```

When the runtime prints the panic, it formats the value with `%v` (or uses the type's `Error()` for errors, `String()` for `Stringer`s). The recover-er gets the value back as `any` and must type-switch:

```go
defer func() {
	switch r := recover().(type) {
	case nil: // not panicking
	case error:
		log.Error("recovered", "err", r)
	case string:
		log.Error("recovered", "msg", r)
	default:
		log.Error("recovered", "value", r)
	}
}()
```

### `recover` only inside deferred functions

```go
func wrong() {
	if r := recover(); r != nil { // always nil — not in a defer
		log.Println(r)
	}
	panic("x") // happens after the useless recover; propagates
}
```

`recover` returns the panic value **only** when it executes during stack unwinding. Calling it outside a defer, or from a function called by a defer (one frame too deep), returns `nil`.

The deferred function may call other functions that *return* the panic value; what matters is which frame invokes `recover`. Modern usage almost always: `defer func() { if r := recover(); r != nil { ... } }()`.

### Re-panicking

Inside a deferred recover, you can decide to forward the panic:

```go
defer func() {
	r := recover()
	if r == nil { return }
	if shouldHandle(r) {
		// handle locally
		return
	}
	panic(r) // re-raise
}()
```

The re-panic propagates with the original value but a new stack origin (the re-panic site). Original goroutine tracebacks are lost unless you captured them. For production tracebacks, use `runtime/debug.Stack()` to snapshot before re-panicking.

### Deferred order during a panic

Deferred functions run in **LIFO** order, just like a normal return. The panic value is observable by each defer through `recover` (only the deferred function that successfully recovers stops propagation). After recovery, deferreds *below* the recovering frame still run; deferreds *above* (in the unwound part of the stack) have already run.

```go
func f() {
	defer fmt.Println("f-defer-1")
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()
	defer fmt.Println("f-defer-3")
	g()
}
func g() {
	defer fmt.Println("g-defer")
	panic("p")
}
// Output:
// g-defer
// f-defer-3
// recovered: p
// f-defer-1
```

### Panic + named return value

A recover-er can mutate named returns to convert a panic into an error:

```go
func parse(s string) (n int, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("parse: %v", r)
		}
	}()
	return mustParse(s), nil
}
```

This is the "convert panic to error at API boundary" idiom — used in `encoding/json`, parser packages, and HTTP middleware.

### `runtime.Goexit`

```go
runtime.Goexit() // terminates the goroutine, runs deferreds, no recover
```

Unwinds the goroutine like a panic but cannot be recovered. Deferred functions still run. Used internally by `t.FailNow()` in `testing` to stop a test goroutine without killing the process.

### Panics during package init

A panic in an `init()` function aborts package initialization, which aborts the whole process before `main`. Useful for invariants that *must* be true at startup ("this binary was built without a required build tag"). No way to recover at this stage from user code.

### Panics inside deferred functions

A panic inside a defer that triggers during another panic produces "panic during panic." Both values are printed, the second supersedes the first for the runtime exit, and both stacks are shown:

```go
func main() {
	defer func() { panic("second") }()
	panic("first")
}
// Output (to stderr):
// panic: first [recovered]
//     panic: second
// [stack traces]
```

This pattern is almost always a bug. If a deferred cleanup might panic, recover inside it.

### `recover` in a goroutine you launch

The goroutine you launch with `go func() { ... }()` does NOT inherit the recover handlers of the launcher. If it panics, only its own deferred recovers can stop the process:

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			log.Error("worker panic", "err", r)
		}
	}()
	work()
}()
```

Forgetting this kills servers. Every goroutine launched from user code should have a top-level recover, *or* a strong argument why it can't panic. Standard pattern: wrap goroutine entry points with a small helper.

```go
func safeGo(label string, fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				log.Error("goroutine panic", "label", label, "err", r, "stack", string(debug.Stack()))
			}
		}()
		fn()
	}()
}
```

### When the runtime itself panics

Several runtime conditions are panics: nil-pointer dereference, integer divide-by-zero, slice-out-of-range, type assertion without comma-ok, closing a closed channel, unrecoverable map race (rare). These look identical to user panics — `recover` catches them. **You should not.** A nil deref is a bug. Recovering and continuing risks operating on corrupt invariants.

The legitimate exception: a top-level recover in an HTTP handler so one bad request doesn't kill the server *while logging the panic loudly*.

### `runtime.Stack` and `runtime/debug.Stack`

Recovery loses the stack. Capture before:

```go
defer func() {
	if r := recover(); r != nil {
		stk := debug.Stack()         // []byte
		log.Error("panic", "v", r, "stack", string(stk))
	}
}()
```

Without `debug.Stack()`, you get the panic value but not where it came from — significantly harder to debug.

### Cross-stack control flow (the rare legitimate use)

A recursive parser deep in the call stack can `panic(parseError{...})` to unwind to the top-level parse function, which recovers and converts to an error return. This is the pattern used by `text/template`, `encoding/gob`, and several stdlib packages — for performance (no need to check err at every recursion level) and clarity.

```go
type parseErr string

func (p *parser) errorf(format string, args ...any) {
	panic(parseErr(fmt.Sprintf(format, args...)))
}

func Parse(s string) (_ *Tree, err error) {
	defer func() {
		if r := recover(); r != nil {
			if pe, ok := r.(parseErr); ok {
				err = errors.New(string(pe))
				return
			}
			panic(r) // unknown panic — re-raise
		}
	}()
	// ...
}
```

The `panic(r)` re-raise is essential: if it isn't your panic type, you must let it propagate. Catch-all `recover` is the bug.

### Performance: defer + recover are not free

A function with a `defer func() { recover() }()` cannot use the compiler's open-coded defer optimization in all cases. The defer is heap-allocated and runs through the generic defer machinery. Cost: tens of nanoseconds per call, plus an allocation in some compiler versions. For HTTP middleware running on every request, it's typically fine; for hot loops, don't add defer/recover to every iteration.

Pre-Go-1.14: defers were always heap-allocated. Since 1.14: open-coded defers eliminate the allocation in many simple cases, but `recover` complicates the inliner's decision.

## Standard Library Hooks

- `recover` (builtin) — only meaningful inside deferred functions.
- `panic` (builtin).
- `runtime.Goexit` — graceful goroutine termination, runs defers, ignored by recover.
- `runtime/debug.Stack` — snapshot the current goroutine's stack.
- `runtime/debug.PrintStack` — write the snapshot to stderr.
- `runtime.SetPanicOnFault` — control whether memory faults in `unsafe`/`syscall` produce recoverable panics or process aborts.
- `net/http`'s `http.Server` wraps each handler in a recover by default (since Go 1.0); panics are logged and the connection is closed.
- `testing.T` uses `runtime.Goexit` for `t.FailNow`.

## Real-World Patterns

### 1. HTTP handler panic guard

```go
func recoverPanic(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rv := recover(); rv != nil {
				slog.Error("http handler panic",
					"path", r.URL.Path,
					"method", r.Method,
					"panic", rv,
					"stack", string(debug.Stack()),
				)
				http.Error(w, "internal server error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

Go's `http.Server` already does this internally (`func (c *conn) serve` has a recover), but the default just logs to stderr and closes the connection. A middleware lets you write a proper 500 response and structured log entry.

### 2. Goroutine entry point wrapper

```go
func Go(name string, fn func()) {
	go func() {
		defer func() {
			if rv := recover(); rv != nil {
				slog.Error("goroutine panic",
					"name", name,
					"panic", rv,
					"stack", string(debug.Stack()),
				)
				panicCounter.WithLabelValues(name).Inc()
			}
		}()
		fn()
	}()
}
```

Use this *everywhere* you start a goroutine. The convention "every long-lived goroutine has a recover" is widely adopted in production Go codebases.

### 3. Parser bailout

```go
type lexErr string

func (l *Lexer) errorf(format string, args ...any) {
	panic(lexErr(fmt.Sprintf(format, args...)))
}

func Lex(input string) (toks []Token, err error) {
	defer func() {
		switch r := recover().(type) {
		case nil:
		case lexErr:
			err = errors.New(string(r))
		default:
			panic(r) // not our error
		}
	}()
	l := &Lexer{input: input}
	return l.run(), nil
}
```

Used inside a single package, with a private panic type, this is acceptable. Crossing package boundaries with panic is not — callers shouldn't have to recover.

### 4. Init-time invariant

```go
package db

var schemaVersion = mustParseVersion("schema_version.txt")

func mustParseVersion(path string) int {
	b, err := os.ReadFile(path)
	if err != nil {
		panic(fmt.Sprintf("db init: read %s: %v", path, err))
	}
	v, err := strconv.Atoi(strings.TrimSpace(string(b)))
	if err != nil {
		panic(fmt.Sprintf("db init: parse %s: %v", path, err))
	}
	return v
}
```

Failures here are deployment errors — the binary cannot function. Crash early, crash loudly.

### 5. `Must` constructors

```go
var tmpl = template.Must(template.New("page").Parse(`<h1>{{.Title}}</h1>`))
```

`template.Must` panics if its argument is a non-nil error. Idiomatic for package-level globals where failure is a programmer error (the template literal is malformed). Don't extend the pattern to runtime values — input from users belongs in error returns.

## Anti-Patterns & Gotchas

**Using panic as a normal error mechanism.** "I'll panic on bad input and recover in the caller" → no. Return `error`. Panics across package boundaries violate the principle of least surprise.

**Recovering everything everywhere.** A blanket `recover` at every layer hides bugs (nil derefs, races) that should crash. Recover at *boundaries only*: HTTP handler entry, goroutine entry, top of a parser. Inside, let panics propagate.

**Recovering without logging.** Silent recovery is the worst outcome: the bug exists but you'll never know. Always log the panic value and the stack.

**Forgetting to re-panic unknown values in a typed bailout.** `defer func() { recover() }()` swallows everything. If your bailout uses a custom `lexErr` type, `panic` anything else through.

**`recover()` outside a defer.** Returns `nil`. The check looks legitimate but does nothing. `go vet` (the `nilness` analyzer in some versions) catches some cases; the `errcheck` linter does not.

**Calling `recover` from a helper called by a defer.**

```go
defer cleanup() // does recover inside — DOESN'T WORK
```

`recover` must be in the deferred function itself, not a function it calls. Wrap in `defer func() { cleanup() }()` instead, with `cleanup` containing the recover, **or** call `recover` inline.

**Panicking goroutines killing the server.** Every `go` keyword needs a recover wrapper or a documented reason it can't panic.

**`panic(nil)`.** Since Go 1.21, `panic(nil)` is detected and converted to `panic(&runtime.PanicNilError{})`, making `recover()` return non-nil. Pre-1.21, `recover()` would return `nil` and the panic would silently continue — a famous footgun.

**Recover for control flow across many frames in production code.** Even the stdlib's parser bailouts limit this to *one package*. Spreading panic-based control flow across packages turns Go into a worse Python.

**Mixing `runtime.Goexit` and `recover`.** `Goexit` walks defers but isn't caught by recover. Don't try to recover from it.

**Deferred functions that panic during another panic.** Each cleanup defer must guard against its own panics, especially during recovery.

## Performance Notes

- `defer func() { recover() }()` adds nanoseconds-to-tens-of-nanoseconds of overhead per call. Open-coded defers (1.14+) help, but `recover` prevents the most aggressive optimization in some cases.
- The panic itself is very expensive: stack walk to find the deferred recover, formatting the value, unwinding. Microseconds, not nanoseconds. Don't use panics on the happy path.
- `debug.Stack()` is an order of magnitude slower than the recover itself (it formats every frame). Call it only in the recovery branch.
- Open-coded defer optimization: the compiler can inline a defer if there are ≤ 8 defers in the function and they're statically known. Recover-style defers usually qualify but check `go tool compile -d=...` if in doubt.
- A goroutine's stack is unwound by walking frames, freeing them lazily. The runtime cost scales with the depth of the call stack at panic time.

Benchmark sketch:

```go
func BenchmarkRecover(b *testing.B) {
	for b.Loop() {
		func() {
			defer func() { recover() }()
		}()
	}
}
// ~5-15 ns/op on modern x86, no allocs with open-coded defer.
```

## How Big Companies Use It

- **Kubernetes** wraps all controller goroutines with `utilruntime.HandleCrash` which recovers, logs to a registered crash-handler, and re-launches. Loss of one reconciler doesn't take down the apiserver.
- **HashiCorp Vault** wraps every plugin RPC call with recover; a panicking plugin returns an error rather than killing Vault.
- **Caddy** wraps each request handler with `defer recoverFromPanic` middleware that returns 500 + writes to the error log.
- **Docker** wraps every API request handler with a top-level recover; the resulting error is returned to the client as a 500 with a generic message and a request ID for log correlation.
- **gRPC-Go** has `grpc_recovery` (community) middleware for unary and streaming interceptors. Recommended in every production service.
- **CockroachDB** treats panics as bugs by default. Test infrastructure detects them; production servers crash and rely on the orchestrator to restart. The reasoning: a panic during a distributed transaction often leaves invariants violated, and continuing is more dangerous than dying.
- **encoding/json**, **encoding/gob**, **html/template**: parser bailouts converting panics to errors at the package boundary.

## Source Code References

Pinned to `go1.26`.

- `runtime.gopanic` and the panic data structure: [`src/runtime/panic.go`](https://github.com/golang/go/blob/master/src/runtime/panic.go).
- `recover` builtin: [`src/runtime/panic.go`](https://github.com/golang/go/blob/master/src/runtime/panic.go) — search `func gorecover`.
- `runtime.Goexit`: [`src/runtime/panic.go`](https://github.com/golang/go/blob/master/src/runtime/panic.go) — search `func Goexit`.
- Open-coded defer logic: [`src/cmd/compile/internal/ssagen/ssa.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go).
- `debug.Stack`: [`src/runtime/debug/stack.go`](https://github.com/golang/go/blob/master/src/runtime/debug/stack.go).
- `http.Server`'s default recover: [`src/net/http/server.go`](https://github.com/golang/go/blob/master/src/net/http/server.go) — search `defer func()` inside `(c *conn) serve`.
- `template.Must`: [`src/text/template/helper.go`](https://github.com/golang/go/blob/master/src/text/template/helper.go).
- 1.21 nil-panic change: https://go.dev/doc/go1.21#language — search `panic(nil)`.

Go source is BSD-3 licensed; cite when copying.

## Further Reading

- Spec, "Handling panics": https://go.dev/ref/spec#Handling_panics.
- Spec, "Run-time panics": https://go.dev/ref/spec#Run_time_panics.
- Go blog, "Defer, Panic, and Recover": https://go.dev/blog/defer-panic-and-recover.
- Go FAQ, "Why does Go not have exceptions?": https://go.dev/doc/faq#exceptions.
- Effective Go, "Panic" and "Recover" sections: https://go.dev/doc/effective_go#panic.
- Dave Cheney, "Why Go gets exceptions right" (2012, dated but useful): https://dave.cheney.net/2012/01/18/why-go-gets-exceptions-right.
- "Panicking gracefully" (Dmitri Shuralyov): https://dmitri.shuralyov.com/blog/40 — practical guidance on goroutine recovery.
- Google Go style guide on panic: https://google.github.io/styleguide/go/decisions#dont-panic.
- Uber style guide on panic: https://github.com/uber-go/guide/blob/master/style.md#dont-panic.

## Exercises / Self-Check

1. Write a `Safely(fn func()) (err error)` that runs `fn`, recovers any panic, and returns it as an error preserving the panic value (use type-switch). Test with both string and error panics.
2. Demonstrate `panic(nil)` behavior in Go 1.21+: write a test that recovers from `panic(nil)` and asserts the recovered value is `*runtime.PanicNilError`.
3. Build a goroutine pool whose workers all use a shared recover wrapper. Show that a panicking task does not kill the pool or other workers.
4. Trace through the parser-bailout pattern in `text/template`: find where the package panics with `template.ExecError` and where it recovers. Why does it use a typed panic value?
5. Benchmark a function with and without `defer func() { recover() }()`. Measure the overhead. At what call rate would adding/removing the recover make a measurable difference?
