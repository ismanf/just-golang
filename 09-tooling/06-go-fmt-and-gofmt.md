# `gofmt` and `go fmt` — The Canonical Formatter

## TL;DR

`gofmt` is a standalone binary that reformats Go source to the canonical style. `go fmt ./...` is just a thin wrapper that runs `gofmt -l -w` on the files in matched packages. The defining feature is **non-configurability**: there are no style options to argue about. Every Go program in every repo formats the same way. `gofmt` rewrites whitespace, normalizes imports' relative order *within a group* (it does *not* rearrange import groups — `goimports` does), aligns struct field tags, and applies a small set of mechanical simplifications when invoked with `-s`. It uses the same AST and parser the compiler uses (`go/parser`, `go/ast`, `go/printer`), so anything `gofmt` produces is guaranteed parseable.

## Mental Model

```
   source file (any whitespace, any spacing)
        │
        ▼
   go/scanner  →  tokens
        │
        ▼
   go/parser   →  AST (with comments)
        │
        ▼
   (optional)  gofmt -r rewrites,  -s simplifications
        │
        ▼
   go/printer  →  canonical output
        │
        ▼
   write back (-w) or print (default) or diff (-d)
```

The formatter does not interpret meaning beyond what the parser sees. Comments are anchored to AST nodes (the `*ast.CommentGroup` next to each statement); they get re-placed by `go/printer` rather than stitched back textually.

## Syntax & Basic Usage

```bash
$ gofmt main.go                    # print formatted version to stdout
$ gofmt -w main.go                 # write in place
$ gofmt -d main.go                 # show diff against current
$ gofmt -l .                       # list files that would change (good for CI)
$ gofmt -s -w main.go              # apply simplifications + write
$ gofmt -r 'a[b:len(a)] -> a[b:]' -w *.go      # apply a rewrite pattern
$ gofmt -e main.go                 # report all errors (not just first 10)

$ go fmt                           # gofmt -l -w on current dir
$ go fmt ./...                     # all packages in module
$ go fmt -n ./...                  # dry-run (print commands)
$ go fmt -x ./...                  # also print commands as they run
```

## Deep Dive

### What `gofmt` actually changes

Mechanical, non-controversial:

1. **Indentation** — tabs, one per nesting level. (Yes, tabs. Always tabs.)
2. **Spacing around operators** — single space, no space around `*` in pointer types.
3. **Alignment** — struct fields, comments at end-of-line, import statements within a group.
4. **Comment normalization** — single-space after `//`, `/* */` block comments preserved.
5. **Brace placement** — opening braces on the same line; never on the next line.
6. **Removing trailing whitespace** and ensuring a final newline.
7. **Sorting imports within a parenthesized group** alphabetically. (Groups are preserved as-is.)
8. **Reformatting tag strings on struct fields** so columns align.

What it does **not** do:

- Add or remove blank lines (except trailing).
- Move imports between groups (no first-party/third-party separation).
- Rename or reorder identifiers.
- Add or fix imports — that's `goimports`.

### The `-s` simplifications

```bash
$ gofmt -s -w *.go
```

Applies six small rewrites:

| Before                              | After                       |
|-------------------------------------|-----------------------------|
| `s[a:len(s)]`                       | `s[a:]`                     |
| `[]T{T{x}, T{y}}`                   | `[]T{{x}, {y}}`             |
| `&T{x}` inside `[]*T{...}` literal  | `{x}` (composite literal type elision) |
| `for k, _ := range m`               | `for k := range m`          |
| `for _ = range m`                   | `for range m`               |
| `&x{}` in some slice/map literals   | `{}`                        |

Idiomatic Go style assumes `gofmt -s`; most teams put it in `pre-commit`.

### The `-r` rewrites

```bash
$ gofmt -r 'oldName(a) -> newName(a)' -w *.go
$ gofmt -r 'fmt.Println(a) -> log.Println(a)' -w *.go
```

A `-r` rule is `pattern -> replacement` where lowercase identifiers (`a`, `b`, `x`, `y`) are *wildcards* matching any expression. Use for one-off mechanical refactors; `gopls`/`gorename` handle smarter renames.

### `go fmt` vs. `gofmt`

`go fmt` is a wrapper: `go fmt ./...` runs `gofmt -l -w` on each matched package. It cannot pass arbitrary `gofmt` flags. If you want `-s`, run `gofmt` directly:

```bash
$ gofmt -s -l -w .
$ find . -name '*.go' -not -path './vendor/*' -exec gofmt -s -l -w {} +
```

`go fmt` ignores `vendor/` and other ignored dirs (`_*`, `.*`); raw `gofmt` doesn't, so target paths carefully.

### CI usage

```bash
$ test -z "$(gofmt -l .)"
```

If `gofmt -l` prints any filenames, the test fails (`-z` checks for empty string). With simplification:

```bash
$ test -z "$(gofmt -s -l .)"
```

Or use a Make target:

```makefile
fmt-check:
	@out=$$(gofmt -s -l .); \
	if [ -n "$$out" ]; then echo "gofmt issues:"; echo "$$out"; exit 1; fi
```

### Editor integration

Every editor with Go support runs `gofmt` on save. `gopls` (`09-tooling/16-gopls.md`) provides a "format on save" command equivalent to `gofmt -s` plus `goimports`.

VS Code: `"go.formatTool": "gofmt"` (or `"goimports"` / `"gofumpt"`).
Vim/Neovim with `vim-go`: `:GoFmt` runs `gofmt`; `:GoImports` runs `goimports`.
GoLand: built-in, configurable to use `gofumpt`.

### `go/printer` modes

`gofmt` uses `go/printer.Mode = UseSpaces | TabIndent` with `Tabwidth = 8`. You can override programmatically:

```go
import "go/printer"

var cfg = &printer.Config{
    Mode:     printer.UseSpaces | printer.TabIndent,
    Tabwidth: 8,
}
```

But Don't. The whole point of `gofmt` is one style.

### Why no configuration?

Rob Pike, in a 2010 design note:

> Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.

Time spent arguing about brace placement, tab width, or line length is time not spent on the substance of the code. By removing the option, Go removed the debate. Strictly, the *only* style decision Go encodes outside `gofmt` is the recommendation in [Effective Go](https://go.dev/doc/effective_go) — and even there, most prescriptions just describe what `gofmt` does.

### Subtle: comment placement

A trailing-line comment binds to the line above it:

```go
x := 1 // initial value
```

A leading comment binds to the next statement:

```go
// initialize
x := 1
```

`gofmt` preserves which form you used. To force the leading style, put the comment on its own line.

### Subtle: alignment in declarations

```go
var (
    a int
    bb string
    ccc bool
)
```

`gofmt` aligns the types because they're in a single `var` block with no blank line. A blank line **breaks** the alignment group:

```go
var (
    a int
    bb string

    ccc bool       // separate alignment group
)
```

Same rule for struct fields and tag strings. Use blank lines deliberately.

### Behavior on syntax errors

```bash
$ gofmt broken.go
broken.go:5:1: expected '}', found 'EOF'
```

Exit code 2 if parsing failed. `-e` makes it print *all* errors (default is the first 10).

### `gofmt -d` for review

```bash
$ gofmt -d main.go
diff old/main.go new/main.go
--- old/main.go
+++ new/main.go
@@ -3,5 +3,5 @@
 func main(){
-x:=1
-fmt.Println(x)
+	x := 1
+	fmt.Println(x)
 }
```

Useful in code review to see what `gofmt` would do without writing.

## Standard Library Hooks

- `go/parser.ParseFile` — read source into AST.
- `go/printer.Fprint` — write AST back.
- `go/format.Source([]byte) ([]byte, error)` — the high-level "format this source" entry point.
- `go/format.Node(w io.Writer, fset *token.FileSet, node any) error` — format a single AST node.
- `go/ast/astutil` — AST manipulation helpers (used by `goimports`).
- `go/token` — position information.

In-process formatter:

```go
package main

import (
    "fmt"
    "go/format"
)

func main() {
    src := []byte("package main\nfunc main(){fmt.Println(1)}")
    out, err := format.Source(src)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(out))
}
```

## Real-World Patterns

### 1. Pre-commit hook

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.go$' || true)
if [ -n "$files" ]; then
    out=$(gofmt -s -l $files)
    if [ -n "$out" ]; then
        echo "Run gofmt on:"
        echo "$out"
        exit 1
    fi
fi
```

### 2. CI gate

```yaml
- run: |
    diff -u <(echo -n) <(gofmt -s -d .)
```

Prints the diff and fails if non-empty.

### 3. One-shot rewrite

Rename `Foo` to `Bar` across all files (use `gopls` for proper rename, but `gofmt -r` works for trivial cases):

```bash
$ gofmt -r 'Foo(a) -> Bar(a)' -w **/*.go
```

### 4. Format generated code

```go
//go:generate stringer -type=Color
package types
```

The generated file is already `gofmt`-clean; if your generator emits raw strings, format them via `go/format.Source` before writing.

### 5. Format from inside a tool

```go
import "go/format"

formatted, err := format.Source(rawBytes)
if err != nil {
    // syntactic error in input
}
os.WriteFile(path, formatted, 0644)
```

### 6. Editor-on-save

VS Code default config:

```json
"[go]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "golang.go"
},
"go.formatTool": "goimports"
```

(`goimports` runs `gofmt` *and* fixes imports.)

## Anti-Patterns & Gotchas

**Configuring an alternative formatter without team agreement.** `gofumpt` is stricter than `gofmt`; mixing the two in one repo creates churn. Pick one per repo.

**Running `gofmt` *and* `gofumpt` in different editors on the same files.** Each "fixes" the other's output, looping forever. Pick one tool, share the config.

**Editing `.go` files in a non-go-aware editor and committing.** Trailing whitespace, tabs vs. spaces, line endings get wrong. Always run through `gofmt` (or use Go-aware editor).

**Skipping `-s`.** Modern Go style assumes simplification was applied. Reviewers will ask you to apply it anyway.

**Trying to disable `gofmt` "for one file".** There's no escape hatch. If you have a reason to deviate (generated code with specific layout), put it in a build-tagged file that you generate fresh; don't fight the formatter.

**Mistaking `gofmt`'s import ordering for `goimports`'s.** `gofmt` only sorts *within* an existing import group. Adding/removing imports and creating standard-library / third-party groups is `goimports` work.

**Re-formatting on every diff because two team members run different `gofmt` versions.** Cross-version drift is rare but real. Pin Go toolchain version in `go.mod`'s `toolchain` line.

**Running `go fmt` from a script that expects exit codes.** `go fmt` exits 0 if it ran (it ran the underlying `gofmt -l -w`). Use raw `gofmt -l` and check stdout for a non-empty result.

**Treating `gofmt` errors as "stylistic warnings".** They're parser errors: the file is broken. Fix the syntax.

**Running `gofmt` on third-party `vendor/`.** Don't. Vendored code should be byte-identical to upstream.

**Forgetting `gofmt` doesn't touch line length.** Go has no line-length rule. Tools that try to wrap long lines (`golines`) operate outside `gofmt` and have their own opinions.

## Performance Notes

- `gofmt` on a 1000-LoC file: ~5–20 ms.
- `gofmt -l .` on a 100k-LoC project: <1 s.
- `go fmt ./...` (which wraps gofmt): same plus package-loading overhead.
- `gofmt -d`: similar timing, but the diff render adds a bit.
- `go/format.Source` from inside a tool: <1 ms for small inputs; scales linearly with file size.

`gofmt` is the cheapest tool in the toolchain. There is no reason not to run it on every save.

## How Big Companies Use It

- **Google** runs `gofmt -s` as a presubmit on every `golang.org/x/*` repo: https://go.dev/doc/contribute.
- **Kubernetes** enforces `gofmt -s` via the `hack/verify-gofmt.sh` script: https://github.com/kubernetes/kubernetes.
- **Uber** runs `gofmt -s -l` plus `goimports` in CI for every Go repo: https://github.com/uber-go/guide.
- **CockroachDB** uses `gofmt -s` plus `crlfmt` (their internal stricter formatter) for `goimports`-style grouping: https://github.com/cockroachdb/crlfmt.
- **HashiCorp** uses `gofmt -s` plus `goimports` in their CI standard template: https://github.com/hashicorp/terraform.
- **Discord** chose `gofumpt` over `gofmt` for stricter consistency; documented in their internal style guide.
- **Tailscale** runs `gofmt -s` and a `staticcheck` pass on every PR: https://github.com/tailscale/tailscale.

## Source Code References

Pinned to `go1.26`.

- `gofmt` command: [`src/cmd/gofmt`](https://github.com/golang/go/tree/release-branch.go1.26/src/cmd/gofmt).
- Simplifier: [`src/cmd/gofmt/simplify.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/gofmt/simplify.go).
- Rewriter: [`src/cmd/gofmt/rewrite.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/gofmt/rewrite.go).
- `go fmt` wrapper: [`src/cmd/go/internal/fmtcmd/fmt.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/fmtcmd/fmt.go).
- `go/printer`: [`src/go/printer`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/printer).
- `go/format`: [`src/go/format/format.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/go/format/format.go).
- `go/parser`: [`src/go/parser`](https://github.com/golang/go/tree/release-branch.go1.26/src/go/parser).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command gofmt": https://pkg.go.dev/cmd/gofmt.
- "Go formatting" (Robert Griesemer, on `go/printer`): https://research.swtch.com/gofmt.
- "Effective Go — Formatting": https://go.dev/doc/effective_go#formatting.
- "The cost of opinions" (Brad Fitzpatrick on `gofmt`): https://talks.golang.org/2014/research.slide.
- `gofumpt` (stricter superset): https://github.com/mvdan/gofumpt.
- `goimports` (gofmt + imports): https://pkg.go.dev/golang.org/x/tools/cmd/goimports.

## Exercises / Self-Check

1. Write a Go file with deliberately misaligned struct field tags. Run `gofmt -d`. What columns does it align?
2. Apply `gofmt -r 'len(a) == 0 -> a == nil'` to a file. Why is this rewrite *unsafe* in general?
3. Use `go/format.Source` from a small Go program to format a string of source code. What error does an invalid input produce?
4. Find one place in your codebase where `gofmt -s` would simplify. Why does the simplification not change semantics?
5. Why does `gofmt` use tabs for indentation? (Hint: editor configurability without re-formatting.)
