# Table-Driven Tests

## TL;DR

A **table-driven test** is a Go idiom where each test case is an entry in a slice of structs and the test loops over them. Instead of N copy-pasted `TestXxx` functions or N nested `if got != want { ... }` blocks, you write one assertion engine that consumes structured cases. The pattern is universal in stdlib and in production Go code because it: (1) makes adding a case a one-line change, (2) keeps fixture data adjacent to behavior, (3) plays naturally with `t.Run` for per-case subtests, and (4) lets `t.Parallel` parallelize cases trivially. The canonical form is a slice of anonymous structs with `name`, `input`, and `want` fields, iterated inside `TestXxx`. Since Go 1.22 the **loop variable per-iteration scoping fix** removed the long-standing closure trap, so capturing `tt` in `t.Run(tt.name, ...)` is now safe.

## Mental Model

```
   tests := []struct {
       name  string
       input X
       want  Y
   }{
       {"case A", inA, outA},
       {"case B", inB, outB},
       {"case C", inC, outC},
   }

   for _, tt := range tests {
       t.Run(tt.name, func(t *testing.T) {
           // arrange (using tt.input)
           // act    (call the code under test)
           // assert (compare to tt.want)
       })
   }
```

Each row of the table becomes one subtest. Adding a case = adding a row.

## Syntax & Basic Usage

```go
func TestReverse(t *testing.T) {
    tests := []struct {
        name string
        in   string
        want string
    }{
        {"empty", "", ""},
        {"ascii", "abc", "cba"},
        {"unicode", "héllo", "olléh"},
        {"emoji", "👍🏽", "🏽👍"},   // demonstrates a bug — see Anti-Patterns
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Reverse(tt.in)
            if got != tt.want {
                t.Errorf("Reverse(%q) = %q, want %q", tt.in, got, tt.want)
            }
        })
    }
}
```

Run:

```bash
$ go test -v ./...
=== RUN   TestReverse
=== RUN   TestReverse/empty
=== RUN   TestReverse/ascii
=== RUN   TestReverse/unicode
=== RUN   TestReverse/emoji
    reverse_test.go:18: Reverse("👍🏽") = ..., want "🏽👍"
--- FAIL: TestReverse (0.00s)
    --- PASS: TestReverse/empty
    --- PASS: TestReverse/ascii
    --- PASS: TestReverse/unicode
    --- FAIL: TestReverse/emoji
```

Each subtest's pass/fail is independent.

## Deep Dive

### Anatomy of a table-driven test

```go
func TestThing(t *testing.T) {
    // 1. Table definition: per-row what changes.
    tests := []struct {
        name    string
        input   InputType
        want    OutputType
        wantErr error          // or bool, or string suffix, or errors.Is target
    }{
        // 2. Cases.
        {"case 1", in1, want1, nil},
        {"case 2", in2, want2, ErrFoo},
    }

    // 3. Loop with subtests.
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Thing(tt.input)

            // 4. Error handling.
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Fatalf("err = %v, want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected err: %v", err)
            }

            // 5. Value comparison.
            if got != tt.want {
                t.Errorf("Thing(%v) = %v, want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

### The 1.22 loopvar fix

Pre-Go 1.22, this code was wrong:

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // tt was the same variable across iterations;
        // by the time the goroutine ran, tt held the last value
    })
}
```

You had to write `tt := tt` (the famous "ahem, please re-declare it" trick) before `t.Run`.

Since **Go 1.22**, the loop variable is scoped per-iteration. The code above is now correct without the workaround. If your `go.mod` says `go 1.22` or later, you get the safe semantics. Older modules get the legacy behavior. (See `02-language-basics/06-control-flow.md` for the language change.)

Older codebases may still have `tt := tt` lines — harmless but no longer required.

### Test names with special characters

`t.Run("name with spaces", ...)` produces `TestX/name_with_spaces` (spaces become underscores). Slashes inside the name become subtest separators (a `/` in the name creates a nested level). For non-printable inputs:

```go
{name: fmt.Sprintf("%q", in), in: in, want: ...}
```

Use a stable, descriptive name; `-run` filtering uses these names.

### Indexed vs. named cases

```go
// Indexed (legacy)
for i, tt := range tests {
    if got := f(tt.in); got != tt.want {
        t.Errorf("case %d: f(%v) = %v, want %v", i, tt.in, got, tt.want)
    }
}

// Named with t.Run (preferred)
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) { ... })
}
```

Named is preferred because: failure output is clearer; `-run TestX/Specific` targets one case; subtest parallelism is opt-in per case.

### Per-case `t.Parallel`

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()                    // each case runs in parallel
        got := Slow(tt.in)
        if got != tt.want { /* ... */ }
    })
}
```

Combined with `-parallel=N`, this fans out. See `10-testing/03-subtests-and-tparallel.md`.

### Map-keyed tables

Some teams use a `map[string]struct{ ... }` instead of a slice:

```go
tests := map[string]struct{ in, want string }{
    "empty":   {"", ""},
    "ascii":   {"abc", "cba"},
}
for name, tt := range tests {
    t.Run(name, func(t *testing.T) { /* ... */ })
}
```

Pros: name lives next to the data; can't have duplicate names.
Cons: iteration order is random — bad if tests depend on order, but they shouldn't.

### Testing functions with errors

```go
tests := []struct {
    name    string
    in      string
    want    int
    wantErr bool
}{
    {"valid", "42", 42, false},
    {"invalid", "abc", 0, true},
    {"empty", "", 0, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := strconv.Atoi(tt.in)
        if (err != nil) != tt.wantErr {
            t.Fatalf("err = %v, wantErr = %v", err, tt.wantErr)
        }
        if got != tt.want {
            t.Errorf("got %d, want %d", got, tt.want)
        }
    })
}
```

A more refined version uses sentinel errors:

```go
wantErr error
// ...
{"empty", "", 0, strconv.ErrSyntax},
// ...
if !errors.Is(err, tt.wantErr) {
    t.Fatalf("err = %v, want %v", err, tt.wantErr)
}
```

### Helper extraction

When the table fields are repetitive, extract a helper:

```go
type testCase struct {
    name string
    in   Input
    want Output
}

func runCases(t *testing.T, cases []testCase) {
    t.Helper()
    for _, tt := range cases {
        t.Run(tt.name, func(t *testing.T) {
            got := Do(tt.in)
            if got != tt.want {
                t.Errorf("Do(%v) = %v, want %v", tt.in, got, tt.want)
            }
        })
    }
}

func TestDoBasic(t *testing.T)   { runCases(t, basicCases) }
func TestDoEdgeCase(t *testing.T) { runCases(t, edgeCases) }
```

Use only when several tests share the assertion engine; otherwise inline is clearer.

### Generated tables

Tables work well with generated inputs:

```go
tests := []struct{ in, want int }{}
for i := 0; i < 100; i++ {
    tests = append(tests, struct{ in, want int }{i, i * 2})
}
for _, tt := range tests { /* ... */ }
```

Combine with `testing/quick` or `testing.F` (fuzzing) for property-based generation.

### Test name conflicts

Two cases with the same name get suffixed `#01`, `#02`:

```go
{"a", ...},
{"a", ...},     // becomes TestX/a#01
```

Avoid; use unique names so `-run TestX/specific` is unambiguous.

### `go test -run TestX/case_name`

```bash
$ go test -v -run 'TestReverse/unicode' ./...
```

The slash separates outer test from subtest; the regex matches against the full path. Anchor with `$`:

```bash
$ go test -run 'TestReverse/ascii$' ./...   # exactly "ascii" subtest
```

### When *not* to use a table

- The cases differ in setup (different DBs, different mocks). Each case becomes its own `TestX` so setup stays clear.
- The case count is 2 and the cases share little. Two `TestFoo` and `TestBar` are clearer.
- The cases need wildly different assertions. A table forces uniformity; if your assertions differ in kind, separate tests are better.

### Hybrid: table + helper

```go
func TestParse(t *testing.T) {
    t.Run("valid inputs", func(t *testing.T) {
        for _, tt := range validCases {
            t.Run(tt.name, func(t *testing.T) { assertParses(t, tt) })
        }
    })
    t.Run("invalid inputs", func(t *testing.T) {
        for _, tt := range invalidCases {
            t.Run(tt.name, func(t *testing.T) { assertFails(t, tt) })
        }
    })
}
```

Nested subtests, two tables, clear grouping in output.

### Test ordering

Cases run in **slice order** (with `map[string]` they run in random order). Tests should be order-independent — don't rely on case 2 seeing state from case 1. Use `t.Cleanup` to reset.

## Standard Library Hooks

- `testing.T.Run` — the subtest machinery.
- `testing.T.Parallel` — opt into concurrent execution.
- `testing/quick.Check` — basic property-based.
- `testing.F.Add` — fuzz seeds (a kind of one-row table; see `10-testing/05-fuzzing.md`).

## Real-World Patterns

### 1. Parser tests

```go
tests := []struct {
    name string
    in   string
    want Expr
    err  error
}{
    {"int literal", "42", &IntLit{42}, nil},
    {"add", "1+2", &BinOp{"+", &IntLit{1}, &IntLit{2}}, nil},
    {"syntax error", "1++", nil, ErrSyntax},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Parse(tt.in)
        if !errors.Is(err, tt.err) { t.Fatalf("err = %v, want %v", err, tt.err) }
        if diff := cmp.Diff(tt.want, got); diff != "" {
            t.Errorf("Parse(%q) mismatch (-want +got):\n%s", tt.in, diff)
        }
    })
}
```

### 2. HTTP handler tests

```go
tests := []struct {
    name   string
    method string
    path   string
    body   string
    want   int     // status code
}{
    {"GET ok", "GET", "/users/42", "", 200},
    {"POST ok", "POST", "/users", `{"name":"a"}`, 201},
    {"DELETE not-found", "DELETE", "/users/99", "", 404},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        req := httptest.NewRequest(tt.method, tt.path, strings.NewReader(tt.body))
        w := httptest.NewRecorder()
        handler.ServeHTTP(w, req)
        if w.Code != tt.want {
            t.Errorf("status = %d, want %d", w.Code, tt.want)
        }
    })
}
```

### 3. Map-keyed for clarity

```go
tests := map[string]struct{ in, want int }{
    "zero":     {0, 0},
    "positive": {5, 25},
    "negative": {-3, 9},
}
for name, tt := range tests {
    t.Run(name, func(t *testing.T) {
        if got := Square(tt.in); got != tt.want {
            t.Errorf("Square(%d) = %d, want %d", tt.in, got, tt.want)
        }
    })
}
```

### 4. With go-cmp for complex structs

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := Transform(tt.in)
        if diff := cmp.Diff(tt.want, got,
            cmpopts.IgnoreFields(User{}, "CreatedAt"),
        ); diff != "" {
            t.Errorf("mismatch (-want +got):\n%s", diff)
        }
    })
}
```

### 5. Parallel cases

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()       // safe in Go 1.22+ without `tt := tt`
        // ...
    })
}
```

### 6. Negative-name conventions

Some teams prefix failing-case names with `err_` or `invalid_`:

```go
{"err_missing_field", ...},
{"err_negative_count", ...},
```

Easy to grep, easy to filter (`-run 'TestX/err_'`).

### 7. Shared setup per table

```go
func TestUser(t *testing.T) {
    db := setupDB(t)        // once
    tests := []struct{ ... }{ /* ... */ }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            seedTx(t, db, tt.fixture)
            // assert
        })
    }
}
```

## Anti-Patterns & Gotchas

**Pre-1.22 loop closure trap.** If your `go.mod` says `go 1.21`, you still need `tt := tt` before `t.Run` for parallel subtests. Easiest fix: bump `go` directive.

**One giant table with disparate cases.** When half the rows test addition and half test subtraction, split into two tables.

**Hidden state between cases.** Mutating a shared global in one case affects the next. Use `t.Cleanup` per case to reset.

**Computed `want` in the table.** `{"x", 1, 1+1}` — when the test fails, "what was the want?" requires mental arithmetic. Compute literal values where possible.

**Long anonymous struct definitions.** When the struct has 10 fields, name it:

```go
type caseFor_X struct { ... }
var xCases = []caseFor_X{ ... }
```

**`fmt.Sprintf` for case names of complex inputs.** Hard to grep. Use stable, descriptive names ("empty map", "nil receiver").

**Treating `wantErr bool` as enough.** "Got an error, wanted an error" doesn't say *which* error. Use sentinel errors or `errors.As` checks.

**Map iteration order assumptions.** If cases must run in a specific order (rare), use a slice. If they don't, map is fine.

**Forgetting `t.Run` — single-loop test with N assertions.** Without `t.Run`, a failure in case 3 doesn't tell you it's case 3 in any clean way. Use subtests.

**Hard-coded test indices in error messages.** When you reorder rows, the message lies. Use `tt.name`, not array index.

**Mixing `t.Errorf` and `t.Fatalf` in a case without thought.** Use `Fatal` for "can't continue this case"; `Errorf` for "note this and move on within the case".

**Tables that depend on order-dependent helpers.** A `setupDB` helper that grows state across cases is dangerous in parallel tests. Each case should be self-contained.

## Performance Notes

- Per-case `t.Run` overhead: ~10 µs.
- Subtest cache effects: subtests are cached collectively as part of the parent's cache key.
- Map-keyed table iteration: slightly slower than slice due to randomization; negligible at typical sizes.
- Table-driven vs. N copy-pasted tests: identical runtime; tables save *human* time.

## How Big Companies Use It

- **Google** uses table-driven tests universally; their style guide formalizes it: https://google.github.io/styleguide/go/decisions#table-driven-tests.
- **Kubernetes** uses table-driven tests across the codebase; `cmd/kubectl/.../*_test.go` is full of them: https://github.com/kubernetes/kubernetes.
- **Uber** documents table-driven as the default pattern: https://github.com/uber-go/guide/blob/master/style.md.
- **HashiCorp** uses table-driven heavily in Terraform provider tests: https://github.com/hashicorp/terraform.
- **CockroachDB** uses table-driven tests across their SQL parser and execution code: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses table-driven plus `cmp` for diff-style failures: https://github.com/tailscale/tailscale.
- **The Go team** uses table-driven everywhere in stdlib (`time`, `encoding/json`, `strings`): https://github.com/golang/go.

## Source Code References

Many stdlib tests exemplify table-driven; some canonical references:

- `time` parsing tests: [`src/time/format_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/time/format_test.go).
- `strings` tests: [`src/strings/strings_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/strings/strings_test.go).
- `encoding/json` tests: [`src/encoding/json/decode_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/encoding/json/decode_test.go).
- `t.Run`: [`src/testing/sub_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/sub_test.go) (implementation in `src/testing/testing.go`).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Prefer table driven tests" (Dave Cheney): https://dave.cheney.net/2019/05/07/prefer-table-driven-tests.
- "Subtests and sub-benchmarks" (Marcel van Lohuizen, Go blog): https://go.dev/blog/subtests.
- "Google Go style — Table tests": https://google.github.io/styleguide/go/decisions#table-driven-tests.
- "How I write Go tests" (Mat Ryer): https://medium.com/@matryer.
- "Testing patterns in the Go standard library" (Brad Fitzpatrick talk).

## Exercises / Self-Check

1. Convert a copy-pasted set of three `TestFoo*` functions into one table-driven `TestFoo`.
2. Add a case to a table-driven test that you expect to fail. Run it. Does the failure output point at the right row?
3. Convert a slice-based table to a map-keyed table. Compare iteration order over multiple runs.
4. Add `t.Parallel()` inside the subtest. Run with `-parallel=4`. What's the speedup?
5. In a Go 1.21 module, parallelize subtests without `tt := tt`. Observe the bug. Now bump to Go 1.22; observe the fix.
