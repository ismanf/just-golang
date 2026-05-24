# Examples as Tests — `ExampleXxx`

## TL;DR

A Go **example function** is `func ExampleXxx() { ... }` in a `*_test.go` file. When the function ends with a `// Output: <expected>` comment (or `// Unordered output:`), the `testing` framework runs it and verifies its stdout matches. Examples serve **three purposes** simultaneously: (1) documentation — pkg.go.dev renders them inline with a "Run" button, (2) compile-time guarantees — examples are compiled with the test binary; an example that calls a renamed API breaks the build, (3) light executable tests — they run with `go test`. Naming binds the example to the symbol: `ExampleFoo` → function `Foo`; `ExampleType_Method` → method; `Example_named` → labeled (no symbol). They are not a substitute for unit tests; treat them as documentation that happens to be verified.

## Mental Model

```
   ExampleFoo
        │
        ▼
   compiled into the test binary
        │
        ▼
   if `// Output:` comment present:
        ├─ run the function
        ├─ capture stdout
        ├─ trim trailing space; compare to comment
        ├─ pass / fail like any test
        │
        ▼
   pkg.go.dev / go doc:
        ├─ extract source as documentation
        └─ render with "Run" button on web
```

`go doc -all` and pkg.go.dev display examples in-line with the symbol they document. Without an `Output` comment, the function only compiles — it's a "code documentation" snippet, not a test.

## Syntax & Basic Usage

```go
package strings_test

import (
    "fmt"
    "strings"
)

func ExampleToUpper() {
    fmt.Println(strings.ToUpper("hello"))
    // Output: HELLO
}

func ExampleToLower() {
    fmt.Println(strings.ToLower("HI"))
    // Output: hi
}

func ExampleBuilder_WriteString() {
    var b strings.Builder
    b.WriteString("hello, ")
    b.WriteString("world")
    fmt.Println(b.String())
    // Output: hello, world
}

func Example_overview() {
    // A package-level overview example, shown at the top of the doc page.
    fmt.Println("welcome")
    // Output: welcome
}
```

Run:

```bash
$ go test ./...                          # examples with Output run
$ go test -run ExampleToUpper ./...
$ go test -v ./...                       # show example pass/fail
```

## Deep Dive

### Naming conventions

| Form                           | Documents                              |
|--------------------------------|----------------------------------------|
| `func ExampleFunc()`           | Function `Func`.                       |
| `func ExampleType()`           | Type `Type`.                            |
| `func ExampleType_Method()`    | Method `(Type) Method`.                |
| `func ExampleType_field()`     | Field `field` of type `Type` (lowercase → field). |
| `func ExampleFunc_suffix()`    | Another example of `Func` (suffix after `_`). Lowercase suffix. |
| `func Example()`               | Whole package overview.                |
| `func Example_suffix()`        | Labeled package example.                |

```go
func ExampleSort()                 // for Sort
func ExampleSort_descending()      // additional Sort example
func ExampleSlice_Pop()            // method
func Example()                     // package-level
func Example_quickstart()          // package-level, labeled "quickstart"
```

Convention: the suffix after the second `_` is lowercase to distinguish from method names.

### Output verification

```go
func ExampleAdd() {
    fmt.Println(Add(2, 3))
    // Output: 5
}
```

The framework:

1. Captures stdout (and stderr — see below).
2. Trims trailing whitespace on each line.
3. Compares to the comment lines after `// Output:`.

```go
func ExampleMultiline() {
    fmt.Println("hello")
    fmt.Println("world")
    // Output:
    // hello
    // world
}
```

The leading `// Output:` line introduces the block; subsequent `// ` lines are the expected output. Indentation in the comment is normalized.

### `Unordered output`

For map iteration or concurrent code where order isn't deterministic:

```go
func ExampleMapKeys() {
    m := map[string]int{"a": 1, "b": 2, "c": 3}
    for k := range m {
        fmt.Println(k)
    }
    // Unordered output:
    // a
    // b
    // c
}
```

The framework sorts both got and want lines before comparing.

### Examples without `Output:`

```go
func ExampleNoOutput() {
    // shown in docs but not run as a test
    f := openFile()
    defer f.Close()
}
```

These compile (catching breakage if the API changes) and render in docs, but `go test` doesn't execute them. Use when:

- Output is non-deterministic (network call, time-dependent).
- The example is illustrative but actually running it isn't practical (requires a database).
- You want the documentation without the runtime guarantee.

### Why examples *are* tests (in the compile sense)

```go
func ExampleX() {
    var x = NewX()
    x.OldMethod()       // if OldMethod is renamed, this breaks the build
}
```

Even without `Output:`, the example is compiled with the test binary. Renaming a function breaks `go build ./...` of the test package. That alone is reason enough to write examples for every exported function: refactors can't silently break docs.

### stdout vs. stderr

```go
func ExampleX() {
    fmt.Fprintln(os.Stderr, "this is captured too")
    fmt.Println("regular")
    // Output: regular
    // this is captured too
}
```

Both streams merge into the captured output, in the order written. Mind interleaving when concurrency is involved.

### Example placement

Examples live in `*_test.go` files. By convention:

- Examples of public APIs go in `<file>_test.go` next to the implementation.
- Package-level overview examples go in `example_test.go` or `doc_test.go`.

Either `package foo` (white-box) or `package foo_test` (black-box) works. Black-box is preferred because the example demonstrates how an outside user would call the API.

```go
// foo/example_test.go
package foo_test

import (
    "fmt"
    "github.com/me/proj/foo"
)

func ExampleNew() {
    f := foo.New()
    fmt.Println(f.Name)
    // Output: defaultName
}
```

### Examples in pkg.go.dev

[pkg.go.dev](https://pkg.go.dev) renders each example next to the symbol it documents. The "Run" button on the web invokes the Go Playground, executing the example interactively. Examples are the single best onboarding doc for a library.

Try: https://pkg.go.dev/strings#example-Builder — see the Builder example, click "Run", get instant feedback.

### Output formatting tips

- Use `fmt.Println` (not `fmt.Print`) — easier line matching.
- Be deterministic — avoid `time.Now()`, `rand`, maps in ordered output.
- Use `Unordered output:` for maps/concurrent.
- Avoid trailing whitespace in output strings; the comparator trims, but it's confusing.

### Examples that show errors

```go
func ExampleParse_error() {
    _, err := Parse("invalid")
    fmt.Println(err)
    // Output: parse: invalid input
}
```

Useful for documenting error messages.

### Multiple examples per symbol

```go
func ExampleSort()                  // primary
func ExampleSort_strings()          // for []string
func ExampleSort_descending()       // descending order
```

pkg.go.dev shows them all under `Sort`'s docs, labeled with the suffix.

### What examples can't do

- Take parameters (they're `func()` — no args).
- Use `*testing.T` (use `TestXxx` for that).
- Assert custom messages (the only check is stdout match).
- Run subtests.
- Skip dynamically (without `Output:`, they don't run).

If you need any of these, write a regular test.

### Comment immediately precedes output line

```go
func ExampleX() {
    fmt.Println("foo")

    // This is a regular comment.

    // Output: foo
}
```

The `// Output:` line must be the *last* comment block in the function. Lines after it become the expected output. Earlier comments are ignored by the parser.

### Examples and `_test.go` build mode

```go
// +build !nogen
package foo
```

Build tags on `*_test.go` files apply to examples too. Useful to skip examples that require a network:

```go
//go:build integration

package foo_test

import "testing"

func ExampleRemote() { /* ... */ }
```

`go test` without `-tags=integration` skips the file entirely.

### `go test -v` output

```
=== RUN   ExampleHello
--- PASS: ExampleHello (0.00s)
=== RUN   ExampleSort_descending
--- FAIL: ExampleSort_descending (0.00s)
got:
    [3 2 1]
want:
    [3 2 1 0]
```

Failing examples show diff-style output.

## Standard Library Hooks

- `testing.Example` (internal) — represents an example.
- `testing.runExamples` — runs them.
- `go/doc.Package.Examples` — extracts examples for documentation.
- `go/doc/comment` — renders example bodies in docs.
- pkg.go.dev's frontend — fetches and runs examples via the Playground.

## Real-World Patterns

### 1. Function example

```go
func ExampleFields() {
    fmt.Printf("%q\n", strings.Fields("  foo  bar  baz   "))
    // Output: ["foo" "bar" "baz"]
}
```

### 2. Method example

```go
func ExampleBuilder_WriteString() {
    var b strings.Builder
    b.WriteString("Hello, ")
    b.WriteString("World!")
    fmt.Println(b.String())
    // Output: Hello, World!
}
```

### 3. Package-level overview

```go
// Package strset provides a thread-safe set of strings.
package strset

// Example_overview is rendered at the top of pkg.go.dev's strset page.
func Example_overview() {
    s := strset.New()
    s.Add("a"); s.Add("b"); s.Add("a")
    fmt.Println(s.Len())
    // Output: 2
}
```

### 4. Unordered output for maps

```go
func ExampleHeaders() {
    h := map[string]string{"X-Auth": "token", "Content-Type": "json"}
    for k, v := range h {
        fmt.Printf("%s=%s\n", k, v)
    }
    // Unordered output:
    // X-Auth=token
    // Content-Type=json
}
```

### 5. Example without output (illustrative only)

```go
func ExampleNew() {
    db := New("conn-string")
    defer db.Close()
    // ... real work
}
```

Compiles; renders in docs; doesn't run as a test (no `// Output:`).

### 6. Error example

```go
func ExampleParse_invalid() {
    _, err := Parse(`{`)
    fmt.Println(err)
    // Output: unexpected EOF
}
```

### 7. Building docs site

```bash
$ pkgsite -open .         # serves docs with examples
```

See `09-tooling/08-go-doc-and-godoc.md`.

## Anti-Patterns & Gotchas

**Non-deterministic output without `Unordered output:`.** Map iteration, goroutine prints — produce flaky tests. Use sorted output or `Unordered output:`.

**`time.Now()` or random in output.** Same problem.

**Examples that read from external sources.** Network/file system calls — flaky CI. Skip via `// Output:` omission or build tag.

**Comments after `// Output:`.** Anything after that line is treated as expected output. Don't add explanatory comments below.

**Trailing whitespace mismatches.** The framework trims, but a stray space in the source can confuse you. Lint with `gofmt`.

**Forgetting `package foo_test` for black-box.** White-box examples work but don't demonstrate the actual import path users will write.

**Multi-line output without proper `//` prefix.** Each output line needs `// ` (slash-slash space). `//hello` (no space) is parsed as `hello`, often a mismatch.

**Examples that take parameters.** Compile error — examples must be `func()`.

**Examples that test edge cases.** That's what `TestXxx` is for. Examples should illustrate the *common* case.

**Examples on unexported APIs.** Not shown in docs; misuse of the form.

**Example with `// Output:` that's actually expected to fail.** No way to assert "this should error" cleanly; use a regular test.

**Examples printing internal-format strings (memory addresses, pointers).** Non-deterministic; replace with named formatters or skip output.

**Many examples on the same symbol with non-descriptive suffixes.** `Sort_a`, `Sort_b` — confusing. Use meaningful suffixes (`Sort_descending`, `Sort_strings`).

## Performance Notes

- Example runtime: same as a unit test of equivalent size; usually <1 ms.
- Examples without `Output:`: compiled only; zero runtime cost.
- Caching: same as tests; cached across `go test` invocations.

Negligible cost; the value is documentation accuracy.

## How Big Companies Use It

- **Google** uses examples extensively in stdlib; every public symbol typically has one: https://pkg.go.dev/strings.
- **Kubernetes** uses examples in `k8s.io/client-go` and `k8s.io/apimachinery` to demonstrate usage patterns: https://pkg.go.dev/k8s.io/client-go.
- **Uber's `zap`** has rich examples covering structured logging patterns: https://pkg.go.dev/go.uber.org/zap#pkg-examples.
- **HashiCorp's `terraform-plugin-framework`** uses examples for provider authors: https://pkg.go.dev/github.com/hashicorp/terraform-plugin-framework.
- **CockroachDB's `cockroach-go` client** ships examples for connection management: https://pkg.go.dev/github.com/cockroachdb/cockroach-go.
- **Tailscale's `tsnet`** uses examples as the primary user-facing intro: https://pkg.go.dev/tailscale.com/tsnet.
- **The Go team** mandates examples for new stdlib APIs: https://go.dev/doc/contribute.

## Source Code References

Pinned to `go1.26`.

- Example runner: [`src/testing/example.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/example.go).
- `runExamples`: [`src/testing/run_example.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/testing/run_example.go).
- Example parser (in go/doc): [`src/go/doc/example.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/doc/example.go).
- pkgsite example rendering: [`golang.org/x/pkgsite/internal/godoc/dochtml`](https://github.com/golang/pkgsite/tree/master/internal/godoc/dochtml).
- Playground integration: [`golang.org/x/tools/godoc/redirect`](https://github.com/golang/tools/tree/master/godoc).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Testable Examples in Go" (Andrew Gerrand, Go blog): https://go.dev/blog/examples.
- "Go Doc Comments" (Russ Cox): https://go.dev/doc/comment#examples.
- "How to write Go examples": https://pkg.go.dev/testing#hdr-Examples.
- "Examples in pkg.go.dev": https://pkg.go.dev/about#examples.
- "Effective Go — Examples": https://go.dev/doc/effective_go.

## Exercises / Self-Check

1. Write an `Example` for a function in your code with `// Output:`. Confirm `go test -v` shows it pass.
2. Change a method's name. Does the example break the build? (It should.)
3. Use `// Unordered output:` for an example that prints map entries. Why is this needed?
4. Add a package-level `Example_overview` to your library. Render it locally with `pkgsite -open .`.
5. Write an example without `// Output:`. Does `go test` run it? Does it appear in docs?
