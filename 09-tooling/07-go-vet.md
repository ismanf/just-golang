# `go vet` — Built-In Static Analyzers

## TL;DR

`go vet` runs a set of bundled static-analysis checks on Go source. The checks catch suspicious but legal constructs: `Printf` format mismatches, struct tag typos, lock-by-value bugs, unreachable code, accidental shadowing, time-format mistakes (`stdTimeFormat` checker), useless conversions, and many more. It does *not* check style (that's `gofmt`), nor performance, nor third-party patterns (that's `staticcheck`/`golangci-lint`). `go test` runs the *default* subset of `go vet` automatically before running tests since Go 1.10 — failing `vet` aborts the test before any test code executes. The vet binary is the canonical surface for running `analysis.Analyzer` plugins; everything `staticcheck` and `gopls` use is built on the same `golang.org/x/tools/go/analysis` framework.

## Mental Model

```
   go vet ./...
        │
        ▼
   load packages via go/packages
        │
        ▼
   for each pkg:
        run every enabled Analyzer
             │
             ├─ printf       (format string vs. args)
             ├─ shadow       (variable shadowing)
             ├─ structtag    (tag syntax)
             ├─ atomic       (atomic.AddX misuse)
             ├─ copylocks    (sync.Mutex copied by value)
             ├─ httpresponse (resp.Body.Close before err check)
             ├─ ...          (~30 default analyzers)
        │
        ▼
   print diagnostics to stderr
   exit 0 if clean, 1 otherwise
```

Analyzers run on the same `go/analysis.Pass` API third-party tools use. `go vet -vettool=staticcheck` is a real, supported way to substitute another tool with the same interface.

## Syntax & Basic Usage

```bash
$ go vet ./...                       # run default analyzers on every package
$ go vet -json ./...                 # machine-readable output
$ go vet -shadow ./...               # enable a non-default analyzer
$ go vet -shadow=false ./...         # disable a default analyzer
$ go vet -vettool=$(which staticcheck) ./...   # use staticcheck as drop-in
$ go vet help                        # list analyzer names
$ go vet help printf                 # detail on one analyzer
$ go tool vet ./...                  # equivalent for non-go-cmd workflows
```

`go test` runs a subset automatically:

```bash
$ go test ./...     # runs `atomic, bools, buildtag, directive, errorsas, ...` first
$ go test -vet=off ./...    # disable the pre-test vet
$ go test -vet=printf,shadow ./...   # custom subset
$ go test -vet=all ./...             # run every default analyzer
```

## Deep Dive

### The default analyzer set (Go 1.26)

Roughly 30 checks. Important ones (full list via `go vet help`):

| Analyzer        | What it catches                                                           |
|-----------------|---------------------------------------------------------------------------|
| `assign`        | Useless self-assignments (`x = x`).                                       |
| `atomic`        | Misuse of `sync/atomic`: passing non-aligned values, `AddX` to a literal. |
| `bools`         | Mistakes in boolean expressions (`a == nil || a == nil`).                 |
| `buildtag`      | Invalid `//go:build` or `// +build` constraint syntax.                    |
| `cgocall`       | Calling `C.<fn>` with Go pointers in violation of cgo rules.              |
| `composites`    | Composite literals using unnamed fields when the struct comes from another package. |
| `copylocks`     | `sync.Mutex`/`sync.RWMutex` passed/returned by value.                     |
| `defers`        | `time.Since(t)` in a `defer fn(...)` evaluated at defer time, not at run. |
| `directive`     | Misplaced `//go:` directives.                                             |
| `errorsas`      | `errors.As(err, &target)` where target isn't a non-nil pointer.           |
| `framepointer`  | Assembly frame pointer issues.                                            |
| `httpresponse`  | `defer resp.Body.Close()` before checking `err`.                          |
| `ifaceassert`   | Impossible interface assertions (`x.(T)` where T can't satisfy x).        |
| `loopclosure`   | (Pre-1.22) capturing loop variable in goroutine. Less relevant since 1.22 loopvar change. |
| `lostcancel`    | `ctx, cancel := context.WithCancel(...)` where `cancel` is never called.  |
| `nilfunc`       | Comparing a function to nil (almost never what you want).                 |
| `printf`        | `Printf("%s", n)` mismatches.                                             |
| `shift`         | Integer shift exceeding type width.                                       |
| `slog`          | Misuse of `log/slog` (odd number of args, type-mismatched keys).           |
| `stdmethods`    | Methods that look like interface methods but have the wrong signature.    |
| `stdversion`    | (1.21+) Use of stdlib API newer than the module's `go` directive.         |
| `stringintconv` | `string(myInt)` (you probably wanted `strconv.Itoa`).                      |
| `structtag`     | Struct tag syntax errors (`json:foo`, missing quotes).                    |
| `testinggoroutine` | Calling `t.Fatal` from a non-test goroutine.                            |
| `tests`         | Mistakes in test function signatures (`func TestFoo(t *testing.B)`).       |
| `timeformat`    | `time.Parse("2006-02-01")` (wrong month/day order vs. reference time).    |
| `unmarshal`     | Passing a non-pointer to `json.Unmarshal`.                                 |
| `unreachable`   | Code after `return`/`panic`/`os.Exit`.                                     |
| `unsafeptr`     | Conversions to/from `unsafe.Pointer` that aren't one of the 6 valid patterns. |
| `unusedresult`  | Discarding the result of `fmt.Errorf`, `errors.New`, etc.                  |

Plus a few non-default ones:

- `shadow` — variable shadowing (off by default; lots of false positives in idiomatic code).
- `fieldalignment` — struct fields not optimally ordered for memory (off by default).
- `nilness` — nil dereferences detected via simple flow analysis (off by default; needs `-vettool=...`).
- `reflectvaluecompare` — `==` on `reflect.Value` (off by default).

Enable per-flag: `go vet -shadow ./...`. Enable all: not a simple flag; use `staticcheck` or `golangci-lint`.

### The `printf` analyzer (most-used)

```go
fmt.Printf("user %s id %d", user.ID, user.Name)
//                    │            └── wrong: name should be %s, id is int
//                    └── prints int as %s, garbage output
```

`go vet` catches:

```
./main.go:5: Printf format %s has arg user.ID of wrong type int
./main.go:5: Printf format %d has arg user.Name of wrong type string
```

It understands all of `%s`, `%d`, `%v`, `%T`, `%w` (error wrapping), `%q`, `%x`, etc., and tracks `wrapped` formatters defined in your own code via `//go:fix` notation:

```go
// Logf is a printf wrapper.
//
//go:noinline
func Logf(format string, args ...any) {
    fmt.Printf(format, args...)
}
```

Vet recognizes wrappers ending in `f` automatically. For wrappers with other names:

```go
// MyLog logs a formatted message.
// func MyLog(string, ...any)     <-- doc-recognized
func MyLog(format string, args ...any) { ... }
```

(Older convention: put `// vet:"printf"` in the doc; modern Go uses simple name suffixes.)

### `copylocks`

```go
var mu sync.Mutex
func f(m sync.Mutex) {}   // vet: f passes lock by value: sync.Mutex
```

Once a `sync.Mutex` is copied, its internal state diverges from the original. The check covers `sync.Mutex`, `sync.RWMutex`, `sync.Once`, `sync.WaitGroup`, `sync/atomic` types, and `noCopy`-marked structs.

### `lostcancel`

```go
func f() {
    ctx, cancel := context.WithCancel(context.Background())
    // forgot to defer cancel()
    doWork(ctx)
}
// vet: the cancel function returned by context.WithCancel should be called
```

A leaked `cancel` keeps the context goroutine alive forever. The check covers `WithCancel`, `WithTimeout`, `WithDeadline`, and `WithCancelCause` (1.20+).

### `httpresponse`

```go
resp, err := http.Get(url)
defer resp.Body.Close()   // vet: using resp before checking err
if err != nil { return err }
```

`resp` is nil on error; deferring before the err check panics. Correct:

```go
resp, err := http.Get(url)
if err != nil { return err }
defer resp.Body.Close()
```

### `timeformat`

```go
t.Format("2006-02-01")   // vet: 2006-02-01 should be 2006-01-02
```

The reference time is `Mon Jan 2 15:04:05 MST 2006`. Anything other than `2006-01-02` for "year-month-day" is suspicious.

### `slog`

```go
slog.Info("hello", "user")          // vet: odd number of arguments
slog.Info("hello", 42, "world")     // vet: key "42" is not a string
```

Keys must be strings (or `slog.Attr` values); pairs must be balanced. See `08-stdlib/19-log-slog.md`.

### `stdversion`

```go
// go.mod: go 1.20
slices.Sort(s)   // vet: slices.Sort requires go1.21 or later
```

The module's `go` directive enforces a min version (since 1.21). `stdversion` catches use of newer APIs.

### `shadow` (opt-in)

```go
$ go vet -shadow ./...
```

```go
err := f()
if err != nil {
    err := g()   // vet: declaration of "err" shadows declaration at ...
    if err != nil { ... }
}
```

Lots of idiomatic Go shadows on purpose (re-declaring `err` in nested scopes is fine). Many teams skip `shadow`.

### `fieldalignment` (opt-in, via `go vet -vettool=fieldalignment`)

```go
type Bad struct {
    a bool   // 1 byte + 7 bytes padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 bytes padding
}            // total: 24 bytes
```

Suggests reordering to `int64, bool, bool` → 16 bytes (or 10 + 2 padding). Useful for memory-sensitive code; noisy for everyday structs.

### Integration with `go test`

Since Go 1.10, `go test` runs a *high-confidence* subset of vet analyzers before running tests. The subset is hard-coded; it's smaller than the full default to avoid surprising test failures.

The list (as of 1.26): `atomic, bools, buildtag, directive, errorsas, ifaceassert, nilfunc, printf, stringintconv`.

```bash
$ go test -vet=off ./...                # disable pre-test vet
$ go test -vet=all ./...                # run the full default set
$ go test -vet=printf,shadow ./...      # arbitrary list
```

If vet fails, test exits before running any test. The failure mode is intentional: a `Printf` typo is a fast feedback signal that you don't want hidden behind a passing test.

### `go vet -vettool=`

```bash
$ go install honnef.co/go/tools/cmd/staticcheck@latest
$ go vet -vettool=$(which staticcheck) ./...
```

Any binary that implements the `analysis.Analyzer` protocol can run via `-vettool`. This is how `staticcheck` slots into `go test -vet=...`:

```bash
$ go test -vet=off ./...     # disable built-in vet
$ staticcheck ./...           # run staticcheck separately
```

(Or, more commonly, run both — built-in vet for speed, staticcheck for depth.)

### `-json` output

```bash
$ go vet -json ./... 2>&1 | jq .
```

Per-package, per-analyzer output for tooling:

```json
{
  "github.com/me/pkg": {
    "printf": [
      {
        "posn": "/src/main.go:5:6",
        "message": "Printf format %s has arg n of wrong type int"
      }
    ]
  }
}
```

Useful for ingest into static-analysis dashboards.

### Writing a custom analyzer

```go
package myanalyzer

import (
    "go/ast"
    "golang.org/x/tools/go/analysis"
    "golang.org/x/tools/go/analysis/passes/inspect"
    "golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
    Name:     "noPanic",
    Doc:      "report calls to panic()",
    Requires: []*analysis.Analyzer{inspect.Analyzer},
    Run:      run,
}

func run(pass *analysis.Pass) (any, error) {
    insp := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)
    insp.Preorder([]ast.Node{(*ast.CallExpr)(nil)}, func(n ast.Node) {
        call := n.(*ast.CallExpr)
        if id, ok := call.Fun.(*ast.Ident); ok && id.Name == "panic" {
            pass.Reportf(call.Pos(), "do not call panic()")
        }
    })
    return nil, nil
}
```

Compile a binary that exports it (via `unitchecker.Main(myanalyzer.Analyzer)`) and use it as `-vettool`. See `golang.org/x/tools/go/analysis/cmd/singlechecker`.

## Standard Library Hooks

- `golang.org/x/tools/go/analysis` — the framework (Analyzer, Pass, Diagnostic).
- `golang.org/x/tools/go/analysis/passes/*` — bundled analyzers' source (printf, copylocks, etc.).
- `golang.org/x/tools/go/analysis/singlechecker` — runner for a single analyzer.
- `golang.org/x/tools/go/analysis/multichecker` — runner for a bundle.
- `golang.org/x/tools/go/analysis/unitchecker` — `-vettool`-compatible runner.
- `go/ast`, `go/types`, `go/token`, `go/parser` — the analysis primitives.

## Real-World Patterns

### 1. CI gate

```yaml
- run: go vet ./...
```

Fast; should always pass.

### 2. CI gate with extra analyzers

```yaml
- run: go vet -shadow ./...
- run: go vet -vettool=$(go env GOPATH)/bin/fieldalignment ./...
```

### 3. Disable pre-test vet (e.g., for legacy code)

```bash
$ go test -vet=off ./legacy/...
```

But fix it for new code.

### 4. Vet only specific packages

```bash
$ go vet ./cmd/... ./internal/...
```

### 5. Per-analyzer tuning

```bash
$ go vet -printf=false ./vendor/...   # skip vendored
$ go vet -shadow ./internal/...
```

### 6. Vet a single file

```bash
$ go vet single.go    # works for fragments
```

## Anti-Patterns & Gotchas

**Disabling `vet` in `go test` (`-vet=off`).** Loses fast `Printf` and `errors.As` feedback. Almost never worth it.

**Trusting `vet` to catch every bug.** It's a small set of high-confidence checks; serious linting belongs to `staticcheck`/`golangci-lint`.

**Mixing custom analyzers without versioning.** A custom `-vettool` checked into a repo without version pinning breaks across machines. Pin via `go install pkg@v1.2.3` and commit the path.

**Enabling `shadow` and `fieldalignment` without preparing reviewers.** Both produce lots of suggestions; introduce gradually with team buy-in.

**Putting `printf`-wrapping functions without a recognizable name suffix.** `Logf`, `Errorf` are seen; `Log`, `Error` aren't. Add `f` to the name to enable format-string checking.

**Forgetting that `go vet` runs on *packages*, not files.** A package with a build-tag–excluded file may not be analyzed. Pass `-tags=...` to match the build.

**Trusting `vet` clean code to be `staticcheck` clean.** They overlap minimally. Run both.

**Vet warnings on auto-generated code.** Use `//go:generate` headers to exclude (`-skip-tests` doesn't exist; gate via build tags or `linename:nolint` markers via `golangci-lint`).

**Confusing `go vet` exit code with test result.** `go vet` exits 1 on any finding; `go test` aborts but reports the vet finding clearly. Always check stderr.

**Vendor code triggering vet findings.** `go vet ./...` skips `vendor/` by default. If you `go vet ./vendor/...` it analyzes it — usually noise.

**Custom analyzers writing to a package they don't own.** Analyzers must not modify source; they only emit diagnostics. Fixers belong to `gopls`/`gofix`.

## Performance Notes

- `go vet ./...` on a 100k-LoC project: ~5–15 s warm cache.
- Cold cache (just-cloned repo): 30–90 s due to package loading.
- Per-analyzer cost varies: `printf` is cheap; `nilness` (flow-based) is the slowest in the default set.
- `staticcheck` (full set) on the same project: 30 s – 3 min.
- `golangci-lint` on the same project: typically 1–10 min depending on enabled linters.

Vet results are cached in `$GOCACHE` keyed by analyzer + package; re-runs without source changes are <1 s.

## How Big Companies Use It

- **Google** runs `go vet` plus several internal analyzers on every change: https://go.dev/doc/contribute.
- **Kubernetes** runs `go vet` plus `golangci-lint` in pre-submit: https://github.com/kubernetes/kubernetes.
- **Uber** runs `go vet` + `staticcheck` + an internal `lint` aggregator: https://github.com/uber-go/guide.
- **CockroachDB** runs `go vet` plus a custom analyzer suite (`crl-lint`) that includes locking conventions: https://github.com/cockroachdb/cockroach.
- **HashiCorp** runs `go vet -all ./...` in CI plus dedicated analyzers for HCL parsing: https://github.com/hashicorp/terraform.
- **The Go team** dogfoods `go vet` plus `staticcheck` plus several xtools analyzers: https://github.com/golang/go/blob/master/src/cmd/vet.
- **Tailscale** ships a custom analyzer (`depaware`) that gates which packages can import which: https://github.com/tailscale/depaware.

## Source Code References

Pinned to `go1.26`.

- `go vet` command: [`src/cmd/vet`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/vet).
- Bundled analyzers: [`golang.org/x/tools/go/analysis/passes`](https://github.com/golang/tools/tree/master/go/analysis/passes).
- `printf` analyzer: [`golang.org/x/tools/go/analysis/passes/printf`](https://github.com/golang/tools/tree/master/go/analysis/passes/printf).
- `copylocks` analyzer: [`golang.org/x/tools/go/analysis/passes/copylocks`](https://github.com/golang/tools/tree/master/go/analysis/passes/copylocks).
- Analysis framework: [`golang.org/x/tools/go/analysis`](https://github.com/golang/tools/tree/master/go/analysis).
- `go test -vet=` wiring: [`src/cmd/go/internal/test/test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/test/test.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command vet": https://pkg.go.dev/cmd/vet.
- "go/analysis package": https://pkg.go.dev/golang.org/x/tools/go/analysis.
- "Writing useful Go analyzers" (Alan Donovan): https://go.dev/blog/analysis.
- "go vet — code static analysis" (Go blog): https://go.dev/blog/govet.
- "Staticcheck checks" (overlaps with vet): https://staticcheck.dev/docs/checks.

## Exercises / Self-Check

1. Write code that mismatches a `Printf` format. Run `go vet`. Now write a wrapper `func MyLogf(format string, args ...any)` that calls `fmt.Printf` — does `vet` see through it?
2. Trigger the `lostcancel` analyzer. Fix it. What goroutine leak did you prevent?
3. Run `go vet -shadow ./...` on your project. Are the findings false positives, real bugs, or stylistic?
4. Install `fieldalignment` and run it on a struct-heavy package. Which suggestion saves the most memory?
5. Write a one-analyzer `singlechecker` that forbids calls to `os.Exit` outside `func main`.
