# Unit Tests — The `testing` Package

## TL;DR

A Go unit test is a function `func TestXxx(t *testing.T)` in a file ending `_test.go` in the package being tested (or a `_test` companion package for black-box tests). The `testing.T` API is intentionally small: **`t.Errorf` / `t.Fatalf`** report failures (Errorf continues, Fatalf stops); **`t.Run`** creates a subtest; **`t.Parallel`** opts into concurrent execution; **`t.Cleanup`** registers a deferred-style teardown; **`t.Setenv`** sets an env var that auto-resets; **`t.TempDir`** returns a per-test temp directory auto-deleted; **`t.Helper`** marks a function as a test helper so failure locations point to the call site; **`t.Skip` / `t.Skipf`** mark a test skipped. Go has no built-in assertion library (`assert.Equal(...)` is a `testify` add-on; see `10-testing/12-testify-controversy.md`). The idiom is: compute, compare with `!=` or `reflect.DeepEqual`, call `t.Errorf("got X, want Y", got, want)`.

## Mental Model

```
   foo_test.go
        │
        ▼
   func TestFoo(t *testing.T) {
       ───────────────
       setup (t.TempDir, t.Setenv, ...)
       ───────────────
       got := callCode()
       if got != want {
           t.Errorf("Foo: got=%v want=%v", got, want)
       }
       ───────────────
       cleanup (t.Cleanup, defer fn(), ...)
   }
        │
        ▼
   go test ./...   ── builds package_test binary, runs each TestX
```

`*testing.T` is a per-test handle: each test gets its own. Calling `t.Fatal` stops *that test* (and its subtests); other tests in the package continue.

## Syntax & Basic Usage

```go
// file: math/sqrt.go
package math

func Sqrt(x float64) float64 { /* ... */ }

// file: math/sqrt_test.go
package math

import "testing"

func TestSqrt(t *testing.T) {
    got := Sqrt(4)
    if got != 2 {
        t.Errorf("Sqrt(4) = %v, want 2", got)
    }
}
```

Run:

```bash
$ go test                       # current package
$ go test ./...                 # all packages
$ go test -run TestSqrt ./...
$ go test -v ./...
$ go test -count=1 ./...        # bust test cache
```

## Deep Dive

### Naming and discovery

A test function is **discovered** if and only if:

1. It lives in a `*_test.go` file.
2. It has the signature `func TestXxx(t *testing.T)` where `Xxx` starts with an uppercase letter (or is empty: just `Test` is *not* allowed since 1.13; needs at least one more char like `Test_` for older idioms).
3. It is in `package <name>` or `package <name>_test`.

The `_test` package suffix is the **black-box** flavor: tests can only use the package's exported API. Used to enforce that exported APIs are sufficient and to avoid import cycles.

```go
// math/sqrt_test.go
package math          // white-box: sees unexported

// math/sqrt_ext_test.go
package math_test     // black-box: only uses exported
import "github.com/me/math"
```

Both styles can coexist in the same directory.

### `T` methods (the core surface)

```go
t.Errorf(format string, args ...any)  // log failure, keep running
t.Error(args ...any)
t.Fatalf(format string, args ...any)  // log failure, runtime.Goexit
t.Fatal(args ...any)
t.Fail()                               // mark failed without logging
t.FailNow()                            // == t.Fail then runtime.Goexit
t.Failed() bool                        // has anything failed so far?
t.Log(args ...any)                     // log, shown with -v or on failure
t.Logf(format string, args ...any)
t.Skip(args ...any)                    // mark skipped, stop
t.Skipf(format string, args ...any)
t.SkipNow()
t.Skipped() bool
t.Helper()                              // adjust attribution upward in stack
t.Name() string                        // current test name (with subtest path)
t.Cleanup(f func())                    // register teardown (LIFO)
t.Setenv(key, value string)            // set env var; auto-reset on test end
t.TempDir() string                     // per-test temp dir; auto-deleted
t.Chdir(dir string)                    // (1.24+) change CWD; auto-reset
t.Context() context.Context            // (1.24+) test-scoped context, cancelled at end
t.Deadline() (time.Time, bool)         // test's deadline if any
t.Parallel()                           // run concurrently with other Parallel tests
t.Run(name string, f func(t *testing.T)) bool   // subtest
```

### `t.Helper`

```go
func mustReadAll(t *testing.T, r io.Reader) []byte {
    t.Helper()                          // <—
    b, err := io.ReadAll(r)
    if err != nil {
        t.Fatalf("read: %v", err)
    }
    return b
}

func TestRead(t *testing.T) {
    data := mustReadAll(t, openFile(t))
    // failure reports line of TestRead, not mustReadAll
}
```

Without `t.Helper`, the failure line is inside `mustReadAll` — useless. With it, the line reported is the caller's.

### `t.Cleanup`

```go
func TestX(t *testing.T) {
    f, err := os.CreateTemp("", "")
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { os.Remove(f.Name()); f.Close() })

    // ... test
}
```

Run **after** the test completes (pass, fail, or skip), in LIFO order. Survives panics (caught and reported as failure). Preferred over `defer` because:

- Runs even if the test calls `t.Fatal` (defer doesn't run after `runtime.Goexit` inside a sub-helper).
- Cooperates with subtests: each `t.Cleanup` registered in a subtest runs at the subtest's end.

### `t.TempDir`

```go
func TestX(t *testing.T) {
    dir := t.TempDir()
    // dir is /tmp/TestX1234567890/001
    // auto-removed on test end
}
```

Path includes the test name and a counter. Removed via `t.Cleanup` (so works even on `t.Fatal`).

### `t.Setenv` and `t.Chdir`

```go
func TestX(t *testing.T) {
    t.Setenv("API_KEY", "test")
    t.Chdir(t.TempDir())                // 1.24+
    // env and cwd restored at test end
}
```

`t.Setenv` fails if the test is parallel (env vars are process-wide; parallel tests would race).

### `t.Context` (1.24+)

```go
func TestX(t *testing.T) {
    ctx := t.Context()
    db, err := sql.Open(...)
    if err != nil { t.Fatal(err) }
    row := db.QueryRowContext(ctx, "SELECT 1")
    // ctx is cancelled when the test ends
}
```

Pre-1.24, idiom was `ctx, cancel := context.WithCancel(context.Background()); t.Cleanup(cancel)`. The new helper removes boilerplate.

### Black-box tests for circular avoidance

If your test imports a package that depends on the one being tested, you'll have a cycle. Use `_test` package:

```go
// db/db.go
package db
func Connect() ... { ... }

// db/db_test.go
package db_test

import (
    "testing"
    "github.com/me/db"
    "github.com/me/db/testdb"     // helpers, depends on db
)

func TestConnect(t *testing.T) {
    conn := db.Connect()
    testdb.Use(conn)
}
```

`testdb` can import `db`; `db` is *not* imported into the white-box test side.

### `t.Run` and `t.Parallel` interaction

```go
func TestFoo(t *testing.T) {
    t.Run("subA", func(t *testing.T) { /* sub */ })
    t.Run("subB", func(t *testing.T) { /* sub */ })
}
```

If subtests call `t.Parallel`, the parent's outer code waits for them. Details in `10-testing/03-subtests-and-tparallel.md`.

### `testing.M` and `TestMain`

```go
func TestMain(m *testing.M) {
    setup()
    code := m.Run()
    teardown()
    os.Exit(code)
}
```

Lets you do package-wide setup/teardown. If defined, `TestMain` is the entry point instead of running tests directly; you must call `m.Run()`. Don't `os.Exit` from defers — they don't run after `os.Exit`. Pre-1.15 the call was awkward; modern idiom is:

```go
func TestMain(m *testing.M) {
    setup()
    defer teardown()
    if code := m.Run(); code != 0 {
        teardown()       // belt + braces
        os.Exit(code)
    }
}
```

Use sparingly. Per-test `t.Cleanup` is usually enough.

### `t.Run` as a structuring tool

Even without parallelism, `t.Run` groups related assertions for cleaner output:

```go
func TestUser(t *testing.T) {
    t.Run("Create", func(t *testing.T) { /* ... */ })
    t.Run("Update", func(t *testing.T) { /* ... */ })
    t.Run("Delete", func(t *testing.T) { /* ... */ })
}
```

Failed subtests don't block sibling subtests; output shows `--- FAIL: TestUser/Update`.

### Test caching

`go test` caches passing test results (`05-go-test.md`). A test is re-run only if:

- Source under the package changed.
- Build flags differ.
- An env var the test read changed.

If your test depends on external state Go can't detect (a database, a clock), use `-count=1` or `t.Setenv("CACHE_KEY", time.Now().String())` to defeat the cache.

### Skipping

```go
func TestNetwork(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping network test in short mode")
    }
    // ...
}
```

`testing.Short()` returns true when `-short` is passed. Convention: long-running tests opt out under `-short`.

For build-time skips (OS, arch, build tag), use build tags instead:

```go
//go:build linux

package foo
```

### `*testing.B` and `*testing.F`

```go
func BenchmarkX(b *testing.B) { /* ... */ }
func FuzzX(f *testing.F)      { /* ... */ }
```

Same package conventions. Covered in `10-testing/04-benchmarks.md` and `10-testing/05-fuzzing.md`.

### Comparing values

There's no built-in `assertEqual`. Options:

```go
// Simple values
if got != want {
    t.Errorf("got %v, want %v", got, want)
}

// Slices / maps / structs
if !reflect.DeepEqual(got, want) {
    t.Errorf("got %v, want %v", got, want)
}

// Structured diff with go-cmp (preferred for complex)
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("mismatch (-want +got):\n%s", diff)
}
```

[`github.com/google/go-cmp/cmp`](https://pkg.go.dev/github.com/google/go-cmp/cmp) is the *de facto* deep-compare; it produces unified diffs and supports `cmpopts.IgnoreFields(...)`, `cmpopts.EquateApproxTime(...)`, etc.

### Failure message conventions

```go
t.Errorf("Sqrt(%v) = %v, want %v", input, got, want)
```

Always include: the input, the got value, the want value. Some teams prefer:

```go
t.Errorf("Sqrt(%v): got %v, want %v", input, got, want)
```

The `name: got X, want Y` form is the most-cited. Avoid bare `t.Error("failed")` — when the test fails six weeks later, that message is useless.

### Output buffering

`t.Log` output is buffered per test. Without `-v`, it's printed only on failure. With `-v`, every `t.Log` is shown immediately. Stdout/stderr writes (`fmt.Println`) bypass this and appear interleaved — usually undesirable.

### `t.Parallel` + `t.Setenv` mutual exclusion

```go
func TestX(t *testing.T) {
    t.Parallel()
    t.Setenv("FOO", "bar")   // panic: cannot use t.Setenv in parallel tests
}
```

The runtime catches this. Workaround: don't `t.Parallel` tests that need env state.

## Standard Library Hooks

- `testing` — `T`, `B`, `F`, `M`, `TB`, `PB`.
- `testing/iotest` — broken/limit/timed-out reader/writer helpers.
- `testing/fstest` — virtual filesystem (`fs.FS`).
- `testing/synctest` (1.24+) — synthetic-time tests for concurrency (`10-testing/13-testing-synctest.md`).
- `testing/quick` — basic property-based testing.
- `reflect` — `DeepEqual`.
- `github.com/google/go-cmp/cmp` — third-party deep-compare (Apache-licensed Google project; widely used).

## Real-World Patterns

### 1. Simple unit test

```go
func TestParse(t *testing.T) {
    in := `{"name":"alice"}`
    var u User
    if err := json.Unmarshal([]byte(in), &u); err != nil {
        t.Fatalf("unmarshal: %v", err)
    }
    if u.Name != "alice" {
        t.Errorf("Name = %q, want %q", u.Name, "alice")
    }
}
```

### 2. With helper and cleanup

```go
func setupDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite", t.TempDir()+"/test.db")
    if err != nil { t.Fatalf("open: %v", err) }
    t.Cleanup(func() { db.Close() })
    return db
}

func TestInsert(t *testing.T) {
    db := setupDB(t)
    _, err := db.Exec("CREATE TABLE x (id INT)")
    if err != nil { t.Fatalf("create: %v", err) }
    // ...
}
```

### 3. TestMain with setup

```go
func TestMain(m *testing.M) {
    container := startPostgres()
    defer container.Stop()
    os.Setenv("DB_URL", container.URL)
    os.Exit(m.Run())
}
```

### 4. Black-box test

```go
// pkg/foo/foo_test.go
package foo_test

import (
    "testing"
    "github.com/me/proj/pkg/foo"
)

func TestExportedAPI(t *testing.T) {
    got := foo.Do(1)
    if got != 2 { t.Errorf("Do(1) = %d, want 2", got) }
}
```

### 5. Cmp with options

```go
import (
    "github.com/google/go-cmp/cmp"
    "github.com/google/go-cmp/cmp/cmpopts"
)

if diff := cmp.Diff(want, got,
    cmpopts.IgnoreFields(User{}, "CreatedAt"),
    cmpopts.EquateEmpty(),
); diff != "" {
    t.Errorf("User mismatch (-want +got):\n%s", diff)
}
```

### 6. Skipping based on platform

```go
func TestSyscall(t *testing.T) {
    if runtime.GOOS != "linux" {
        t.Skip("Linux-only syscall test")
    }
    // ...
}
```

### 7. Cleanup that survives Fatal

```go
func TestX(t *testing.T) {
    rsrc := acquire()
    t.Cleanup(rsrc.Release)        // runs even if t.Fatal below fires

    if !rsrc.Ready() {
        t.Fatal("not ready")
    }
}
```

## Anti-Patterns & Gotchas

**Using `defer` for cleanup in tests.** `defer` doesn't run after `t.Fatal` (which calls `runtime.Goexit`). Use `t.Cleanup`.

**`fmt.Println` instead of `t.Log`.** Bypasses buffering; output interleaves across parallel tests. Always use `t.Log`/`t.Logf`.

**Comparing slices with `==`.** Compile error for slices (and you'd want deep equality anyway). Use `reflect.DeepEqual` or `cmp.Diff`.

**Failure messages without inputs.** `t.Errorf("failed")` is useless six weeks later. Always include `(input -> got vs. want)`.

**Mixing `t.Errorf` and `t.Fatal` without thought.** `Fatal` stops the test; subsequent assertions don't run. Use for unrecoverable failures (setup), `Errorf` for assertion-style.

**Forgetting `t.Helper` in helper funcs.** Failure attributions point to the helper, not the test that called it.

**`TestMain` without calling `m.Run()`.** No tests run.

**`os.Exit` from a `TestMain` defer.** Defers don't run after `os.Exit`. Use the LIFO pattern shown above.

**Black-box tests that re-export internal helpers.** If a test needs unexported access, white-box is appropriate; don't add public APIs just to test.

**Tests that assume process-global state.** Two tests setting `os.Setenv` race when `-parallel` is on. Use `t.Setenv`.

**Caching assumptions for tests that depend on external state.** `go test` happily caches a passing result; if the cached result is wrong, you won't notice. Add `-count=1` or `t.Setenv` to invalidate.

**Long `if/then/else` chains in tests instead of table-driven.** See `10-testing/02-table-driven.md`.

**Asserting on string error messages (`err.Error() == "..."`).** Brittle. Use `errors.Is` / `errors.As` against sentinel errors or types.

## Performance Notes

- `go test` per-test overhead: ~µs.
- `t.Run` overhead: ~10 µs.
- `t.Cleanup` registration: ~ns.
- `t.TempDir` create+cleanup: ~ms.
- `t.Setenv` cost: dominated by mutex; ~µs.
- Test cache hit: <500 ms for full project.
- `cmp.Diff` for a large struct: 10–100 µs (allocates).

Most tests run in microseconds; macro-time goes to setup/teardown of external dependencies (DB, network).

## How Big Companies Use It

- **Google** uses standard `testing` plus `cmp` extensively; `cmp` originated at Google: https://github.com/google/go-cmp.
- **Kubernetes** uses `testing` + `cmp` + ginkgo (for e2e specs): https://github.com/kubernetes/kubernetes/blob/master/test.
- **Uber** uses `testing` + `cmp` + `testify` (where teams agree); their Go style guide discourages testify in libraries: https://github.com/uber-go/guide/blob/master/style.md.
- **HashiCorp** uses standard `testing` plus `cmp`; some Terraform code uses testify: https://github.com/hashicorp/terraform.
- **CockroachDB** uses standard `testing` + `cmp` + custom `testutils` for cluster setup: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses standard `testing` + `cmp`; explicitly no testify (style choice): https://github.com/tailscale/tailscale.
- **The Go team** uses pure `testing`; no external libs: https://github.com/golang/go/tree/master/src.

## Source Code References

Pinned to `go1.26`.

- `testing` package: [`src/testing/testing.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/testing.go).
- `T` methods: same file (search `func (c *common)` and `func (t *T)`).
- Test cache: [`src/cmd/go/internal/test/testcache.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/test/testcache.go).
- `t.TempDir`: [`src/testing/testing.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/testing.go) (search `TempDir`).
- `t.Setenv`: same file (search `Setenv`).
- `t.Context`: same file (search `Context`).
- `TestMain`: same file (search `func (m *M)`).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Package testing": https://pkg.go.dev/testing.
- "Go testing — Standard Library Documentation": https://pkg.go.dev/testing.
- "How to write tests" (Go docs): https://go.dev/doc/tutorial/add-a-test.
- "Effective Go — Tests": https://go.dev/doc/effective_go#testing.
- "Useful Go testing patterns" (Dave Cheney): https://dave.cheney.net/2019/05/07/prefer-table-driven-tests.
- "go-cmp documentation": https://pkg.go.dev/github.com/google/go-cmp/cmp.
- "Testify vs. plain testing" (Mat Ryer): https://blog.gobyexample.com/testing.

## Exercises / Self-Check

1. Write a test for `strings.ToUpper`. Cover one passing case and one failure case. Confirm `-v` prints both.
2. Use `t.Helper` in an assertion helper. Verify the failure location points to the test, not the helper.
3. Convert a `defer cleanup()` to `t.Cleanup(cleanup)`. Add a `t.Fatal` call before the cleanup; observe both run.
4. Write a black-box test (`package _test`) that imports the package being tested. Why is this useful?
5. Use `cmp.Diff` to compare two structs ignoring one timestamp field via `cmpopts.IgnoreFields`.
