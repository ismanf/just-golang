# The `testify` Controversy

## TL;DR

**`github.com/stretchr/testify`** is the most-imported third-party test helper in Go. It provides three sub-packages: **`assert`** (continue-on-fail assertions), **`require`** (stop-on-fail), and **`mock`** (the mocking library mockery generates against). It also has **`suite`** (xUnit-style test suites). Why some teams ban it: (1) the API encourages the assertion *style* of testing, which Go's design intentionally discourages — Go testing is about computing-then-checking with explicit `if`, not DSL-style `assert.Equal(t, a, b)`; (2) it makes failure messages less informative than well-written `if got != want { t.Errorf("Foo(%v) = %v, want %v", in, got, want) }`; (3) the `mock` package's API is rough, prompting most teams to use `mockery` to generate against it (more layers); (4) `suite`'s xUnit style is foreign to Go idiom. Why other teams *love* it: it shortens tests, the assertion library is familiar to engineers from other languages, and `require` is genuinely cleaner than the equivalent `if err != nil { t.Fatal(err) }` pattern. This is a real, persistent debate in the Go community.

## Mental Model

```
   Plain testing                          testify
   ─────────────────                      ─────────
   if got != want {                       assert.Equal(t, want, got)
       t.Errorf("got %v, want %v",
           got, want)
   }

   if err != nil {                        require.NoError(t, err)
       t.Fatalf("unexpected: %v", err)
   }

   Decision tree:
       Is your team OK with assertion DSL?   ── yes ── use testify
                                              ── no ──┐
       Do you want richer failure msgs?              ─── stick with plain
       Do you want minimal test deps?
       Do you want stdlib-only?
```

The "right" answer depends on team preference; technically both work. This page lays out the tradeoffs so you can make an informed choice.

## Syntax & Basic Usage

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestSomething(t *testing.T) {
    got, err := DoStuff()
    require.NoError(t, err)               // fail fast
    assert.Equal(t, 42, got)              // continue on fail
    assert.NotNil(t, got.Inner)
    assert.Greater(t, got.Count, 0)
    assert.Contains(t, got.Name, "alice")
    assert.True(t, got.Valid)
    assert.WithinDuration(t, expected, got.Time, time.Second)
}
```

Compare to plain:

```go
func TestSomething(t *testing.T) {
    got, err := DoStuff()
    if err != nil { t.Fatal(err) }
    if got != 42 { t.Errorf("got %v, want 42", got) }
    if got.Inner == nil { t.Errorf("Inner is nil") }
    if got.Count <= 0 { t.Errorf("Count = %d, want > 0", got.Count) }
    if !strings.Contains(got.Name, "alice") {
        t.Errorf("Name = %q, want to contain 'alice'", got.Name)
    }
    // ...
}
```

## Deep Dive

### What `testify` provides

**`assert`** — checks that don't stop the test on failure:

```go
assert.Equal(t, expected, actual)
assert.NotEqual(t, ...)
assert.Nil(t, x)
assert.NotNil(t, x)
assert.True(t, b)
assert.False(t, b)
assert.Empty(t, s)
assert.NotEmpty(t, s)
assert.Len(t, s, 3)
assert.Contains(t, s, "substring")
assert.ElementsMatch(t, a, b)      // same elements, any order
assert.Greater(t, a, b)
assert.Less(t, a, b)
assert.InDelta(t, a, b, 0.01)      // float fuzzy
assert.WithinDuration(t, a, b, time.Second)
assert.Panics(t, func() { ... })
assert.NotPanics(t, func() { ... })
assert.Error(t, err)
assert.NoError(t, err)
assert.ErrorIs(t, err, target)
assert.ErrorAs(t, err, &target)
assert.ErrorContains(t, err, "string")
// ... ~80 more
```

Each returns `bool` so you can chain:

```go
if assert.NotNil(t, got) {
    assert.Equal(t, "alice", got.Name)
}
```

**`require`** — same set, but each calls `t.FailNow()` on failure (stops the test):

```go
require.NoError(t, err)        // if err, t.Fatal-style stop
require.NotNil(t, got)         // similar
```

Use `require` for preconditions (you can't continue without them) and `assert` for independent checks (one failure shouldn't hide others).

**`mock`** — manual mock construction:

```go
type MockStore struct { mock.Mock }
func (m *MockStore) Get(id string) (User, error) {
    args := m.Called(id)
    return args.Get(0).(User), args.Error(1)
}

func TestX(t *testing.T) {
    m := &MockStore{}
    m.On("Get", "42").Return(User{Name: "alice"}, nil)
    // ...
    m.AssertExpectations(t)
}
```

Most teams use `mockery` to generate against this (or use `gomock` / hand-rolled instead). See `10-testing/09-mocking-strategies.md`.

**`suite`** — xUnit-style test classes:

```go
type MySuite struct { suite.Suite; db *sql.DB }
func (s *MySuite) SetupSuite()    { s.db = openDB() }
func (s *MySuite) TearDownSuite() { s.db.Close() }
func (s *MySuite) SetupTest()     { s.db.Exec("TRUNCATE users") }
func (s *MySuite) TestInsert()    { /* ... */ }
func TestMySuite(t *testing.T)    { suite.Run(t, new(MySuite)) }
```

xUnit-flavored; Go ships nothing like this. Some teams find it familiar; others find it foreign.

### The case *for* testify

1. **Familiar to engineers from other languages** — Java, Python, JS have similar libraries.
2. **`require.NoError(t, err)` is genuinely cleaner** than the four-line `if err != nil { t.Fatal(err) }`.
3. **80+ assertion helpers** save typing for common patterns (`ElementsMatch`, `WithinDuration`).
4. **Less boilerplate** in tests means more tests written.
5. **Universal**: 86,000+ Go modules import it; well-supported.

### The case *against* testify

1. **Less informative failure messages by default**:

   ```
   assert.Equal(t, 42, got)
   // failure:
   // Error Trace:    foo_test.go:15
   // Error:          Not equal:
   //                 expected: 42
   //                 actual  : 41
   ```

   vs.

   ```go
   if got != 42 {
       t.Errorf("Process(input=%v) = %d, want 42", input, got)
   }
   // failure:
   // foo_test.go:15: Process(input={a:1,b:2}) = 41, want 42
   ```

   The plain version tells you *what input* produced the wrong result. testify's default doesn't (you can add a message: `assert.Equal(t, 42, got, "input=%v", input)`, but most don't).

2. **Encourages "assertion thinking" over "compute-check thinking"**. Go style: do the work, then check. testify nudges toward chained assertions, which can mask the question "what does this test actually verify?".

3. **Argument order is easy to flip**: `assert.Equal(t, expected, actual)` — many engineers write `actual, expected` by accident. Failures then read backwards.

4. **`mock` API is rough**. Methods like `m.Called(args)` return `mock.Arguments`; you call `.Get(0).(MyType)` and lose type safety. `mockery` papers over this; that's *more* layers.

5. **`suite.Run` doesn't integrate cleanly with `t.Run`**. Subtests work but the syntax is awkward.

6. **A heavy dependency for testing** — adds ~50 transitive deps to test packages.

7. **The Go authors and many idiomatic-Go advocates don't use it.** The stdlib uses zero testify; Tailscale bans it; CockroachDB minimizes use. This isn't an argument by itself, but it's a signal that mature Go projects often opt out.

### What style-guides say

**Uber Go Style Guide**:
> Avoid using `assert.Equal` style assertion libraries because they reduce clarity and make tests harder to understand at a glance. Prefer plain `if` checks.
([source](https://github.com/uber-go/guide/blob/master/style.md))

**Google Go Style**:
> Prefer the standard testing package. Use third-party libraries (like testify) only when they provide clear value beyond what the standard library offers.
([source](https://google.github.io/styleguide/go))

**Effective Go** (Go team):
> Use the standard testing package. Express test expectations in idiomatic Go.

**Most public Go libraries**:
> Some use testify; some don't. Most stdlib-replacement libraries (cmp, errors, slog) use plain testing in their own tests.

### Practical decision framework

If your team is:

- **Just starting**: go without testify; learn idiomatic patterns first.
- **Mixed background, lots of Java/Python**: testify smooths the transition; later teams can migrate off.
- **Already using testify**: don't churn migrating off; the value of consistency exceeds the value of stylistic purity.
- **Building a public library**: stdlib testing — your users don't need testify in their deps.
- **Building an internal app**: either is fine.

### The `require` exception

Even teams that ban `assert` often allow `require`. Reason:

```go
// plain
if err != nil { t.Fatal(err) }

// testify/require
require.NoError(t, err)
```

`require.NoError` saves typing and the message is fine. The argument-order issue doesn't apply (NoError has just one arg). Some teams permit `require.NoError` and `require.NotNil` while banning the rest.

### Migrating away from testify

If you decide to remove testify:

1. **`assert.NoError` / `require.NoError`** → `if err != nil { t.Fatal(err) }`.
2. **`assert.Equal(t, want, got)`** → `if got != want { t.Errorf("Foo: got %v, want %v", got, want) }`.
3. **`assert.True`/`False`** → `if !cond { t.Errorf("cond failed: %v", v) }`.
4. **`assert.Contains`** → `if !strings.Contains(s, sub) { ... }`.
5. **`mock`** → `gomock` or hand-rolled fakes.
6. **`suite`** → table-driven or per-test setup helpers.

Tools like [`brunoga/testify-cleanup`](https://github.com/brunoga/testify-cleanup) (community) can automate part of this. Or do it by hand; it's mechanical.

### `assert.Equal` with non-comparable types

```go
assert.Equal(t, []int{1,2,3}, got)         // works (uses reflect.DeepEqual)
assert.Equal(t, map[string]int{...}, got)  // works
```

The trade: clean syntax, but the failure message just says "expected vs. actual"; the diff is shallow. For deep structures, `cmp.Diff` (in plain testing) produces much better output.

### `Eventually` and `Never`

```go
assert.Eventually(t, func() bool { return condition() }, 5*time.Second, 100*time.Millisecond)
assert.Never(t, ..., 1*time.Second, 100*time.Millisecond)
```

Polls until condition true (or false, for `Never`). Useful when waiting for async work. The plain alternative:

```go
deadline := time.Now().Add(5 * time.Second)
for time.Now().Before(deadline) {
    if condition() { return }
    time.Sleep(100 * time.Millisecond)
}
t.Fatal("condition never became true")
```

`Eventually` is one of the few testify helpers most "ban testify" teams admit is genuinely useful.

### `testify` vs. `go-cmp`

For complex struct comparisons, `cmp.Diff` is better than `assert.Equal`:

```go
// testify
assert.Equal(t, expected, actual)
// failure:
//   expected: User{Name: "alice", ...}
//   actual:   User{Name: "bob", ...}

// cmp
if diff := cmp.Diff(expected, actual); diff != "" {
    t.Errorf("mismatch (-want +got):\n%s", diff)
}
// failure:
//   mismatch (-want +got):
//   {
//     Name: "alice",
//   - Name: "bob",
//   }
```

The `cmp` diff is unified and points at the *specific* field that differs.

## Standard Library Hooks

testify is third-party; no stdlib hooks. Alternatives in stdlib:

- `testing.T` — manual assertions.
- `errors.Is`, `errors.As` — error matching.
- `reflect.DeepEqual` — deep equality.
- `testing/quick.Check` — basic property-based.

Third-party alternatives:

- `github.com/google/go-cmp/cmp` — deep diffs (Google-maintained).
- `gotest.tools/v3/assert` — alternative assertion library; less DSL-heavy.
- `github.com/matryer/is` — minimalist assertion (1-letter package name).

## Real-World Patterns

### 1. Hybrid: `require` for setup, plain for assertions

```go
import "github.com/stretchr/testify/require"

func TestX(t *testing.T) {
    db, err := openDB()
    require.NoError(t, err)
    defer db.Close()

    got, err := repo.Get("42")
    require.NoError(t, err)

    // Plain assertions for the actual check:
    if got.Name != "alice" {
        t.Errorf("Name = %q, want %q", got.Name, "alice")
    }
}
```

Some teams accept this middle ground.

### 2. Pure testify

```go
import "github.com/stretchr/testify/assert"

func TestX(t *testing.T) {
    got, err := DoStuff()
    assert.NoError(t, err)
    assert.Equal(t, expected, got)
    assert.NotEmpty(t, got.Items)
}
```

Familiar to many; concise.

### 3. Pure plain

```go
func TestX(t *testing.T) {
    got, err := DoStuff()
    if err != nil { t.Fatalf("DoStuff: %v", err) }
    if got.Name != "alice" {
        t.Errorf("Name = %q, want %q", got.Name, "alice")
    }
}
```

Explicit; no DSL.

### 4. Plain + cmp

```go
import "github.com/google/go-cmp/cmp"

func TestX(t *testing.T) {
    got, err := DoStuff()
    if err != nil { t.Fatalf("DoStuff: %v", err) }
    if diff := cmp.Diff(expected, got); diff != "" {
        t.Errorf("DoStuff mismatch (-want +got):\n%s", diff)
    }
}
```

Common compromise: stdlib for asserts, `cmp` for diffs.

### 5. Eventually pattern

```go
import "github.com/stretchr/testify/assert"

assert.Eventually(t, func() bool {
    return checkAsync()
}, 5*time.Second, 100*time.Millisecond, "async never converged")
```

Even testify-skeptical teams sometimes pull this in.

### 6. Forbid via linter

```yaml
# .golangci.yml
linters:
  enable: [depguard]
linters-settings:
  depguard:
    rules:
      main:
        deny:
          - pkg: "github.com/stretchr/testify"
            desc: "use stdlib testing"
```

Mechanically enforces the ban.

### 7. Allowlist `require` only

```yaml
linters-settings:
  depguard:
    rules:
      main:
        deny:
          - pkg: "github.com/stretchr/testify/assert"
          - pkg: "github.com/stretchr/testify/mock"
          - pkg: "github.com/stretchr/testify/suite"
```

`require` is allowed.

## Anti-Patterns & Gotchas

**`assert.Equal(t, actual, expected)` reversed args.** The signature is `Equal(t, expected, actual)`. Reversed → failure message is backwards.

**Cluttering tests with both `assert` and `require` without rationale.** Pick a pattern.

**Using `assert.Nil(t, err)` for error checks.** Use `assert.NoError(t, err)` — clearer intent.

**Chained `assert` calls assuming earlier ones fail-fast.** They don't (that's `require`). The test will keep running and may panic on a `nil`.

**Importing both `testify/assert` and `cmp` in the same test.** Pick the diff style; mixing is confusing.

**`assert.Equal` on floats.** Float comparison is treacherous; use `assert.InDelta` or `assert.InEpsilon`.

**`assert.Equal(t, "string", string(got))` for byte slices.** Works but `bytes.Equal(want, got)` is more honest about what's compared.

**`require.NoError(t, nil)`-style checks against literals.** Should never pass anything but a real error.

**Cleaning up via testify's `suite.TearDownTest` instead of `t.Cleanup`.** Subtests in suites don't always cooperate well; prefer `t.Cleanup`.

**Treating "testify provides X helper" as "always use it"**. The right tool depends on the test. Sometimes plain is clearer.

**Migrating from testify in a giant single PR.** Touch hundreds of files; reviewers can't read it. Migrate package-by-package.

## Performance Notes

- testify overhead per assert call: ~100 ns (reflection).
- Plain `if got != want`: ~ns (no overhead).
- For a 10k-assertion test suite, testify adds ~1ms total; not measurable.

Performance isn't the reason to choose or avoid testify; readability is.

## How Big Companies Use It

- **Google** does not use testify in stdlib or `golang.org/x/*` repos: https://github.com/golang/go.
- **Kubernetes** uses testify selectively; many tests are plain: https://github.com/kubernetes/kubernetes.
- **Uber** discourages testify in their style guide; uses cmp + plain: https://github.com/uber-go/guide.
- **HashiCorp** uses testify in many Terraform repos: https://github.com/hashicorp/terraform.
- **CockroachDB** uses minimal testify; mostly plain + cmp: https://github.com/cockroachdb/cockroach.
- **Tailscale** explicitly bans testify in CONTRIBUTING.md: https://github.com/tailscale/tailscale.
- **Docker / Moby** uses testify pervasively: https://github.com/moby/moby.
- **Discord's Go services** allow `require` for setup, plain for assertions.

## Source Code References

- testify: [`github.com/stretchr/testify`](https://github.com/stretchr/testify).
- assert package: [`assert`](https://github.com/stretchr/testify/tree/master/assert).
- require package: [`require`](https://github.com/stretchr/testify/tree/master/require).
- mock package: [`mock`](https://github.com/stretchr/testify/tree/master/mock).
- suite package: [`suite`](https://github.com/stretchr/testify/tree/master/suite).
- cmp (alternative for diffs): [`github.com/google/go-cmp`](https://github.com/google/go-cmp).
- gotest.tools/assert: [`gotest.tools`](https://github.com/gotestyourself/gotest.tools).
- is (minimalist alternative): [`github.com/matryer/is`](https://github.com/matryer/is).

(MIT License for testify; BSD-3-Clause for cmp.)

## Further Reading

- "Uber Go Style Guide — Assertions": https://github.com/uber-go/guide/blob/master/style.md.
- "Why I don't use testify" (multiple bloggers): search "go testify anti-pattern".
- "Tests should be terrible" (Mat Ryer): https://medium.com/@matryer.
- "testify documentation": https://pkg.go.dev/github.com/stretchr/testify.
- "Plain testing vs. testify" (Reddit threads, repeatedly): https://reddit.com/r/golang.
- Russ Cox, "Testing in Go" (informal preference for stdlib): various Go talks.

## Exercises / Self-Check

1. Rewrite a testify test in plain style. Is the failure message richer or poorer?
2. Find an `assert.Equal(t, actual, expected)` call (reversed args). What does its failure message say?
3. Set up `depguard` to ban testify in a small project. How many files need changes?
4. Use `cmp.Diff` for a complex struct comparison. Compare to `assert.Equal`.
5. Survey your team: how many people prefer testify vs. plain? Why? Are the answers correlated with prior language experience?
