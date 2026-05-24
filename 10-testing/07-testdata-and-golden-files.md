# `testdata/` and Golden Files

## TL;DR

The `testdata/` directory is **special to the Go toolchain**: it's the conventional place to put fixtures, sample inputs, and expected outputs that tests load at runtime. `go build` and `go vet` ignore `testdata/` (no compilation, no analysis), so anything inside — broken JSON, syntactically invalid Go, weird binary files — is safe. **Golden files** are a pattern built on top: instead of writing `if got != "long string..." { t.Errorf(...) }`, you store the expected output in `testdata/golden/Foo.txt` and compare the test's output to the file. When intentional changes occur, regenerate goldens with a `-update` flag (a convention, not a stdlib feature). This pattern shines for: text renderers (template output), compilers (AST/IR dumps), formatters, code generators, anything that produces large, structured output. The single rule: **goldens must be committed** to track expected-output drift.

## Mental Model

```
   pkg/
     foo.go
     foo_test.go
     testdata/
       inputs/
         case1.json
         case2.json
       golden/
         case1.txt           ← expected output for case 1
         case2.txt
```

Test reads input from `testdata/inputs/`, calls the code under test, compares output to `testdata/golden/`. To accept new expected output, run with a flag and the test rewrites the golden.

## Syntax & Basic Usage

```go
// foo_test.go
package foo

import (
    "flag"
    "os"
    "path/filepath"
    "testing"
)

var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    cases := []string{"case1", "case2", "case3"}
    for _, name := range cases {
        t.Run(name, func(t *testing.T) {
            in, err := os.ReadFile(filepath.Join("testdata", "inputs", name+".json"))
            if err != nil { t.Fatal(err) }

            got, err := Render(in)
            if err != nil { t.Fatal(err) }

            goldenPath := filepath.Join("testdata", "golden", name+".txt")
            if *update {
                if err := os.WriteFile(goldenPath, got, 0o644); err != nil { t.Fatal(err) }
                return
            }
            want, err := os.ReadFile(goldenPath)
            if err != nil { t.Fatalf("missing golden: %v", err) }
            if !bytes.Equal(got, want) {
                t.Errorf("output mismatch\n--- got ---\n%s\n--- want ---\n%s", got, want)
            }
        })
    }
}
```

Run:

```bash
$ go test ./pkg              # normal: compare
$ go test ./pkg -update      # regenerate goldens; review with git diff
$ git diff testdata/golden   # see what changed
```

## Deep Dive

### Why `testdata/` is special

The Go tool [explicitly ignores](https://pkg.go.dev/cmd/go#hdr-Package_lists_and_patterns) directories named `testdata` (and any directory starting with `.` or `_`) when walking package paths. Implications:

- Files under `testdata/` are **not compiled** — they can be invalid Go.
- `go vet` and analyzers **skip them**.
- `go mod tidy` doesn't try to resolve imports referenced only in `testdata/`.
- `golangci-lint` skips them by default.

So you can put a deliberately malformed `.go` file in `testdata/` to test your formatter or parser; the build tool won't choke.

```
mypkg/
  parser.go
  parser_test.go
  testdata/
    valid/
      hello.go         ← valid Go for parser to parse
    invalid/
      bad-syntax.go    ← intentionally malformed; compiler ignores
```

### Loading testdata

Tests' working directory is the **package directory** (where the `_test.go` files live), so `testdata` resolves relative to it:

```go
data, err := os.ReadFile("testdata/inputs/foo.json")
```

If you run `go test ./...` from a higher dir, `testdata` still resolves correctly because each package's tests start in its own dir.

For cross-platform safety, use `filepath.Join`:

```go
path := filepath.Join("testdata", "inputs", "foo.json")
```

### `embed.FS` alternative

Since Go 1.16, you can embed fixtures at build time:

```go
//go:embed testdata/inputs/*.json
var fixtures embed.FS

func TestX(t *testing.T) {
    data, err := fixtures.ReadFile("testdata/inputs/case1.json")
    // ...
}
```

Pros: no filesystem at test time; works from `go test`-built binaries shipped elsewhere.
Cons: doubles binary size; rebuild needed when fixtures change.

For most tests, plain `os.ReadFile` is preferred (faster iteration, no rebuild).

### Golden file pattern, expanded

```go
var update = flag.Bool("update", false, "update golden files")

func golden(t *testing.T, name string, got []byte) {
    t.Helper()
    path := filepath.Join("testdata", "golden", name+".txt")
    if *update {
        if err := os.MkdirAll(filepath.Dir(path), 0o755); err != nil { t.Fatal(err) }
        if err := os.WriteFile(path, got, 0o644); err != nil { t.Fatal(err) }
        return
    }
    want, err := os.ReadFile(path)
    if err != nil {
        t.Fatalf("golden file %s: %v\nrun `go test -update` to create", path, err)
    }
    if !bytes.Equal(got, want) {
        t.Errorf("golden mismatch for %s\n--- want ---\n%s\n--- got ---\n%s",
            name, want, got)
    }
}
```

Now any test can call `golden(t, "TestX", got)`. Review changes with `git diff` after `-update`.

### Flag conventions

Pick one name and stick with it across the repo:

```bash
$ go test -update       # most common
$ go test -rewrite      # also seen
$ go test -gen          # less common
```

The `*update` is a package-level `*bool` registered via `flag.Bool`. `go test` parses these flags as part of the test binary's args.

### Sanity in `-update` mode

```go
if *update && testing.Short() {
    t.Skip("don't update goldens in -short")
}
```

Or fail loudly if goldens were updated alongside other tests passing:

```go
if *update {
    t.Log("regenerated golden; commit testdata/")
}
```

### Structured diffs

For complex outputs (multi-line, JSON, AST), `bytes.Equal` works but the failure message is awful. Use `go-cmp`:

```go
import "github.com/google/go-cmp/cmp"

if diff := cmp.Diff(string(want), string(got)); diff != "" {
    t.Errorf("golden mismatch (-want +got):\n%s", diff)
}
```

Or for JSON specifically:

```go
var wantV, gotV any
json.Unmarshal(want, &wantV); json.Unmarshal(got, &gotV)
if diff := cmp.Diff(wantV, gotV); diff != "" {
    t.Errorf("...")
}
```

### Subdirectories under testdata

Organize by category:

```
testdata/
  parser/
    inputs/
      simple.go
      complex.go
    golden/
      simple.txt
      complex.txt
  formatter/
    inputs/
    golden/
```

Each test groups its data. Avoid one giant flat dir.

### Naming convention

Conventional patterns:

- `testdata/golden/<TestName>.txt` — one golden per test.
- `testdata/golden/<TestName>_<subtest>.txt` — one per subtest.
- `testdata/golden/<TestName>/<subtest>.txt` — nested dir per test.

Pick one; consistency matters more than the specific choice.

### Generator pattern

Some teams write a separate `generate_test.go` file with `-update`-style logic separated from the assertion. Cleaner; less risk of `*update` flag accidentally regenerating in CI.

### CI safety

```yaml
- run: go test ./...
- run: |
    if [ -n "$(git status --porcelain testdata/)" ]; then
      echo "testdata changed; commit golden updates"
      git diff testdata/
      exit 1
    fi
```

If golden regen happens via a CI-only flag, this catches accidental drift.

### Stdlib uses `testdata` heavily

```bash
$ find $(go env GOROOT)/src -name testdata -type d | head
src/cmd/asm/internal/asm/testdata
src/cmd/compile/internal/test/testdata
src/cmd/cover/testdata
src/cmd/gofmt/testdata
src/cmd/internal/buildid/testdata
src/cmd/internal/cov/testdata
src/cmd/objdump/testdata
src/compress/testdata
src/crypto/x509/testdata
src/debug/buildinfo/testdata
```

Most stdlib packages with parsers, formatters, or readers have testdata trees.

### Files inside `testdata/fuzz/`

A specific use of testdata: persisted fuzz failures (`10-testing/05-fuzzing.md`):

```
testdata/fuzz/FuzzParse/abc123def456
```

Format is engine-defined. Don't hand-edit; the engine reads them as seeds.

### Binary fixtures

Binary files belong in `testdata/` too:

```
testdata/binaries/sample.elf
testdata/audio/sample.wav
testdata/images/test.png
```

`os.ReadFile` returns `[]byte`; treat opaquely.

### `testdata/` and version control

Commit everything in `testdata/`. The only exception is generated noise (e.g., temp files from a misbehaving test) — those shouldn't be in `testdata` in the first place.

For very large fixtures (videos, multi-MB binaries), consider Git LFS; small text files are fine in regular Git.

## Standard Library Hooks

- `os.ReadFile`, `os.WriteFile` — load/store fixtures.
- `path/filepath.Join` — cross-platform paths.
- `embed.FS` — embed fixtures (Go 1.16+).
- `testing/fstest.MapFS` — virtual FS for fixture-style tests.
- `github.com/google/go-cmp/cmp` — diff goldens.

## Real-World Patterns

### 1. Golden text output

```go
func TestRender(t *testing.T) {
    in := loadFixture(t, "input.json")
    got := Render(in)
    golden(t, "TestRender", got)
}
```

### 2. Many cases, many goldens

```go
func TestFormat(t *testing.T) {
    matches, _ := filepath.Glob("testdata/inputs/*.go")
    for _, m := range matches {
        name := strings.TrimSuffix(filepath.Base(m), ".go")
        t.Run(name, func(t *testing.T) {
            in, _ := os.ReadFile(m)
            got, _ := Format(in)
            goldenPath := filepath.Join("testdata", "golden", name+".out")
            // ... compare or update
        })
    }
}
```

The `Glob` pattern means adding a new fixture = adding a new test case automatically.

### 3. JSON comparison

```go
got, _ := json.MarshalIndent(result, "", "  ")
want, _ := os.ReadFile("testdata/golden/result.json")
if diff := cmp.Diff(string(want), string(got)); diff != "" {
    if *update { os.WriteFile("testdata/golden/result.json", got, 0o644); return }
    t.Errorf("mismatch:\n%s", diff)
}
```

### 4. Update workflow

```bash
$ go test ./pkg                    # detect drift
$ go test ./pkg -update            # accept drift
$ git diff testdata/golden         # human review
$ git add testdata/golden && git commit
```

### 5. With embed (for distribution)

```go
//go:embed testdata/golden/*.json
var goldens embed.FS

func TestX(t *testing.T) {
    want, _ := goldens.ReadFile("testdata/golden/TestX.json")
    // ...
}
```

Test binary contains the goldens; useful when shipping a test suite separately.

### 6. CI gate

```yaml
- run: go test ./...
- run: |
    test -z "$(git status --porcelain)" || (
      echo "uncommitted changes; perhaps testdata regenerated?"
      git diff
      exit 1
    )
```

### 7. Per-OS goldens

```go
path := filepath.Join("testdata", "golden", runtime.GOOS, name+".txt")
```

Some outputs differ by OS (path separators, line endings). Branch the golden by GOOS.

## Anti-Patterns & Gotchas

**Forgetting to commit `testdata/`.** Tests pass locally; fail in CI because the golden is missing.

**Auto-running `-update` in CI.** Hides the drift; goldens evolve silently. Always require human review.

**Goldens that contain non-deterministic data.** Timestamps, ports, UUIDs. Normalize before writing.

**Comparing with `==` on `[]byte`.** Compile error; use `bytes.Equal`.

**Reading testdata with absolute paths.** Breaks when tests run from a different directory. Use relative paths starting with `testdata/`.

**Failure messages that print 50KB of text.** Use `cmp.Diff` for compact diffs.

**Putting test data outside `testdata/`.** Then `go build` tries to compile it. Always nest under `testdata/`.

**Embedding goldens via `embed.FS` then trying to update them.** Rebuilding for every change is slow. Stick with filesystem reads for goldens.

**Letting goldens grow without review.** Each `-update` should be a deliberate, reviewed change. CI gate prevents accidental drift.

**Mixing `testdata/` for fixtures and for fuzz corpus.** They coexist; just keep them in separate subdirs (`testdata/inputs/`, `testdata/fuzz/`).

**Encoding implementation details into goldens.** If your golden captures memory addresses or pointer values, every run produces a diff. Strip those before comparing.

**Treating goldens as the *spec*.** They're a snapshot of current behavior, not the desired one. A test passes today because the behavior matches what was captured — not because the behavior is correct. Pair with assertion-style tests for invariants.

**Putting `*update` definitions in multiple test files.** Conflicting flag definitions panic. Define once per package.

## Performance Notes

- `os.ReadFile` for typical fixtures: <1 ms.
- `bytes.Equal` on 100KB: <100 µs.
- `cmp.Diff` on 100KB strings: 10–100 ms (allocates).
- `-update` adds disk writes; minor overall.

Goldens don't slow tests meaningfully; the bottleneck is usually the code under test.

## How Big Companies Use It

- **The Go team** uses `testdata/` heavily in stdlib: `cmd/gofmt`, `cmd/compile`, `crypto/x509`, etc.: https://github.com/golang/go.
- **Kubernetes** uses testdata + golden files for `kubectl` output stability: https://github.com/kubernetes/kubernetes.
- **Cockroach** uses goldens for their SQL parser, optimizer, and plan output: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses goldens for Terraform plan output: https://github.com/hashicorp/terraform.
- **Uber** uses goldens for their `zap` logger formatting tests: https://github.com/uber-go/zap.
- **Tailscale** uses testdata for protocol parser fixtures: https://github.com/tailscale/tailscale.
- **Cloudflare** uses golden files for HTTP/3 frame parsing tests: https://github.com/cloudflare.

## Source Code References

Pinned to `go1.26`.

- `go test` testdata handling: [`src/cmd/go/internal/load/pkg.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/load/pkg.go) (search `testdata`).
- `embed`: [`src/embed/embed.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/embed/embed.go).
- `testing/fstest`: [`src/testing/fstest`](https://github.com/golang/go/tree/release-branch.go1.26/src/testing/fstest).
- Example golden tests in stdlib: [`src/cmd/gofmt/gofmt_test.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/gofmt/gofmt_test.go).
- `cmp` package: [`github.com/google/go-cmp`](https://github.com/google/go-cmp).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Package testdata convention": https://pkg.go.dev/cmd/go#hdr-Package_lists_and_patterns.
- "Golden file testing in Go" (Andrew Gerrand, original talk): https://github.com/golang/go/wiki/TestComments.
- "Using golden files for tests" (Alex Edwards): https://www.alexedwards.net/blog/golden-files.
- "Snapshot testing in Go" (Mitchell Hashimoto): https://mitchellh.com.
- `embed` package: https://pkg.go.dev/embed.
- "Go test fixtures with testdata" (Mat Ryer): https://blog.gobyexample.com/testing.

## Exercises / Self-Check

1. Convert an existing test with a long inline `want` string to a golden file. Add the `-update` flag.
2. Compare bytes-vs-`cmp.Diff` failure messages on a 1KB string. Which is easier to read?
3. Glob `testdata/inputs/*.json`; auto-generate a subtest per file. Add a new file; confirm it becomes a test.
4. Embed goldens via `//go:embed`. What changes about the dev workflow?
5. Set up a CI gate that fails if `testdata/` has uncommitted changes after `go test ./...`.
