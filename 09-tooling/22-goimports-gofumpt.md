# `goimports` and `gofumpt` — Formatter Supersets

## TL;DR

**`goimports`** is `gofmt` *plus* automatic import management: it adds missing imports, removes unused ones, and rearranges them into standard-library and third-party groups (separated by a blank line). It's the de-facto save-action for most Go editors. **`gofumpt`** is `gofmt` *plus* stricter formatting: it removes redundant `()` around expressions, normalizes spacing around `*` in pointer types, enforces a single empty line between top-level declarations, simplifies `if err != nil { return err }` chains, and applies several other opinions. Both produce gofmt-compatible output (you can pipe their output through `gofmt -d` and get no diff). They are *additive* on top of gofmt's rules — never opposed. Picking between them per-repo (or both: gofumpt + goimports composes since gofumpt v0.4) is a one-time team decision.

## Mental Model

```
   .go source
        │
        ▼
   ┌───────────────────────────────────┐
   │  gofmt         ── canonical rules │
   ├───────────────────────────────────┤
   │  goimports     ── + import fixing │
   │      (also runs gofmt internally) │
   ├───────────────────────────────────┤
   │  gofumpt       ── + stricter rules│
   │      (also runs gofmt internally) │
   ├───────────────────────────────────┤
   │  gofumpt -extra ── extra rules    │
   │      (some opinionated)           │
   └───────────────────────────────────┘
        │
        ▼
   reformatted source

   Composable: gofumpt then goimports = both sets of rules.
```

Both tools wrap `gofmt`; they re-apply its rules and then add their own. You always end up with gofmt-clean output as a subset of the result.

## Syntax & Basic Usage

```bash
# goimports
$ go install golang.org/x/tools/cmd/goimports@latest
$ goimports -w main.go               # write in place
$ goimports -d main.go               # diff
$ goimports -l .                     # list files needing changes (CI)
$ goimports -local github.com/me/proj -w .    # third group: local module
$ goimports -srcdir=./pkg main.go    # set context for resolving imports

# gofumpt
$ go install mvdan.cc/gofumpt@latest
$ gofumpt -w main.go
$ gofumpt -d main.go
$ gofumpt -l .                       # CI lint mode
$ gofumpt -extra -w main.go          # apply extra (opinionated) rules
$ gofumpt -lang=1.26 -w main.go      # version-aware rules
$ gofumpt -version
```

## Deep Dive

### What `goimports` adds beyond gofmt

1. **Add missing imports** by scanning for unresolved identifiers and matching them against the stdlib + module deps.
2. **Remove unused imports**.
3. **Group imports**: stdlib first, then third-party (and optionally a local group via `-local`).
4. **Sort within each group** alphabetically.
5. **Apply gofmt** for everything else.

```go
// before
package main

import (
    "github.com/x/y"
    "fmt"
)

func main() {
    fmt.Println(y.Foo(strings.Title("x")))
}
```

```go
// after `goimports -w`:
package main

import (
    "fmt"
    "strings"

    "github.com/x/y"
)

func main() {
    fmt.Println(y.Foo(strings.Title("x")))
}
```

Blank line separates stdlib from third-party.

### `-local` for in-house module group

```bash
$ goimports -local github.com/me/proj -w .
```

Adds a third group for `github.com/me/proj/...`:

```go
import (
    "fmt"
    "strings"

    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"

    "github.com/me/proj/internal/auth"
    "github.com/me/proj/internal/db"
)
```

Commonly set in `.golangci.yml` (goimports linter) or editor config to match team convention.

### How goimports finds imports

When you have `fmt.Println(strings.Title("x"))` with `strings` not imported, goimports needs to figure out which package provides `strings.Title`. It consults:

1. **Stdlib index** — built-in mapping of common identifiers to stdlib packages.
2. **Module cache** — scans `$GOMODCACHE` for packages that provide the identifier.
3. **Current module** — packages within the same module.

If multiple packages match, goimports picks the first found (typically stdlib wins). If none match, the identifier stays unresolved (compile error remains).

### What `gofumpt` adds beyond gofmt

The full list is at https://github.com/mvdan/gofumpt#added-rules; highlights:

1. **No extra leading or trailing blank lines** inside functions or composite literals.
2. **No empty line at the start of a block**.
3. **No `_ = x` for ignored returns** when `x` is just a value.
4. **No redundant parentheses** around expressions.
5. **Spacing around `*` in pointer types**: `func foo() *int` (no space around).
6. **`var x int = 0` → `var x int`** (zero value is implicit).
7. **One empty line between top-level decls**.
8. **Short variable declarations preferred** in test setup (only when clearly safe).
9. **`if err != nil { return err }` simplifications** are *not* applied automatically (controversial; available with `-extra`).

`-extra` adds opinionated rules many teams find too aggressive.

```go
// before
package main

import (
    "fmt"
)

func main() {
    var x int = 0    // gofumpt: drop `int = 0`

    if  ( x ==0  ) {  // gofumpt: redundant parens; spacing
        fmt.Println("zero")
    }
}
```

```go
// after `gofumpt -w`:
package main

import "fmt"

func main() {
    var x int

    if x == 0 {
        fmt.Println("zero")
    }
}
```

### Composing gofumpt + goimports

```bash
$ gofumpt -w main.go
$ goimports -w main.go
```

Or, since gofumpt v0.4, gofumpt's binary can drive both:

```bash
$ gofumpt -w -extra main.go            # gofumpt only
$ # for imports too, run goimports after, or use:
$ goimports -w main.go && gofumpt -w main.go
```

Order matters only slightly: gofumpt then goimports is a common choice because goimports also normalizes whitespace, and gofumpt then re-canonicalizes.

### Editor integration

**VS Code** (`.vscode/settings.json`):

```jsonc
{
  "go.formatTool": "goimports",         // or "gofumpt"
  "go.formatFlags": ["-local", "github.com/me/proj"],
  "[go]": {
    "editor.formatOnSave": true
  }
}
```

For gofumpt integration via gopls:

```jsonc
"gopls": {
  "formatting.gofumpt": true,
  "formatting.local": "github.com/me/proj"
}
```

gopls runs both gofumpt and goimports under "Format Document" when `formatting.gofumpt` is true.

**Neovim** (`nvim-lspconfig`):

```lua
require'lspconfig'.gopls.setup{
  settings = {
    gopls = {
      gofumpt = true,
      ["local"] = "github.com/me/proj",
    }
  }
}
```

### `golangci-lint` integration

```yaml
linters:
  enable:
    - goimports
    - gofumpt
linters-settings:
  goimports:
    local-prefixes: github.com/me/proj
  gofumpt:
    extra-rules: false
    lang-version: "1.26"
```

If both are enabled, the order is goimports then gofumpt by linter design; output is canonical regardless.

### `gofumpt` and Go version

```bash
$ gofumpt -lang=1.26 -w .
```

The `-lang` flag enables rules that depend on language features:

- 1.21+: simplifications using `min`/`max` built-ins.
- 1.22+: range-over-int loop simplifications.
- 1.26+: any 1.26-specific rules.

Read from `go.mod`'s `go` line if `-lang` is unset.

### CI usage

```yaml
# goimports check
- run: |
    out=$(goimports -local github.com/me/proj -l .)
    if [ -n "$out" ]; then echo "$out"; exit 1; fi

# gofumpt check
- run: |
    out=$(gofumpt -l .)
    if [ -n "$out" ]; then echo "$out"; exit 1; fi
```

Or via `golangci-lint`.

### Why `goimports` is on the standard editor save action

Most Go editors run goimports (not raw gofmt) on save because:

- It catches forgotten imports as you type.
- It removes leftover imports after deleting code.
- It keeps the import block sorted automatically.
- Output is still gofmt-clean (subset).

The cost: ~100ms per save for large files (scans the import resolver).

### Why `gofumpt` is sometimes contentious

gofumpt's whole point is *more opinionated* formatting than gofmt. Some teams welcome the consistency; others see it as overstepping. Common debates:

- Does `var x = 5` need to become `x := 5` (gofumpt's `-extra` rule)?
- Does the empty line between every top-level decl reduce density too much?
- Should comment placement be normalized?

Daniel Martí, the author, has been responsive to feedback; gofumpt opts out of the most contentious rules unless `-extra` is set.

### `goimports` and modules

goimports respects `go.mod`: it only imports from packages your module can resolve. If a needed identifier is in a package not in your `go.mod`, goimports adds the package but the build fails until `go mod tidy` brings it in.

Workflow:

```bash
$ # paste code that uses uuid
$ goimports -w main.go       # adds "github.com/google/uuid"
$ go mod tidy                 # adds the require to go.mod
$ go build ./...
```

### Versioning

```bash
$ goimports -version          # not available; check via go install
$ gofumpt -version
v0.6.0
```

`goimports` lives in `golang.org/x/tools`; pin via `go install golang.org/x/tools/cmd/goimports@v0.20.0`.
`gofumpt` lives at `mvdan.cc/gofumpt`; pin via `go install mvdan.cc/gofumpt@v0.6.0`.

### `goimports` and generated files

goimports doesn't skip generated files automatically. Generators should produce gofmt-clean output before goimports runs — or generators should call `go/format.Source` themselves.

To exclude from goimports in CI:

```bash
$ goimports -l $(find . -name '*.go' -not -name '*.gen.go' -not -path './vendor/*')
```

## Standard Library Hooks

- `golang.org/x/tools/imports` — programmatic goimports.
- `mvdan.cc/gofumpt/format` — programmatic gofumpt.
- `go/format`, `go/printer` — underlying gofmt.
- `go/parser`, `go/ast` — AST manipulation.

Programmatic goimports:

```go
import "golang.org/x/tools/imports"

opts := &imports.Options{
    Comments:  true,
    TabIndent: true,
    TabWidth:  8,
    FormatOnly: false,
}
formatted, err := imports.Process("main.go", src, opts)
```

## Real-World Patterns

### 1. Editor on save with `-local`

```jsonc
"go.formatTool": "goimports",
"go.formatFlags": ["-local", "github.com/me/proj"]
```

### 2. Pre-commit hook

```bash
#!/usr/bin/env bash
files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.go$' || true)
[ -z "$files" ] && exit 0
out=$(goimports -local github.com/me/proj -l $files)
if [ -n "$out" ]; then
    echo "Run goimports on:"
    echo "$out"
    exit 1
fi
```

### 3. CI gate

```yaml
- name: goimports
  run: |
    out=$(goimports -local github.com/me/proj -l .)
    if [ -n "$out" ]; then echo "$out"; exit 1; fi
```

### 4. gofumpt-only team

```yaml
linters:
  enable: [gofumpt]
linters-settings:
  gofumpt:
    extra-rules: false
```

### 5. Both gofumpt and goimports

```bash
$ gofumpt -w . && goimports -local github.com/me/proj -w .
```

### 6. Repo `.editorconfig`

```ini
[*.go]
indent_style = tab
indent_size = 8
trim_trailing_whitespace = true
insert_final_newline = true
```

Editor-agnostic baseline; gofmt enforces these regardless.

## Anti-Patterns & Gotchas

**Running `goimports` without `-local`.** Local-module imports get lumped with third-party. Set `-local` to match `module` in `go.mod`.

**Switching between `gofmt`, `goimports`, and `gofumpt` per developer.** Constant diff churn. Pick one team-wide.

**Configuring gopls and editor separately.** Editor runs gofmt; gopls returns gofumpt — they fight. Set both to the same tool.

**Editing an import block by hand.** goimports/gofumpt will rewrite it on save. Either fix once via the tool or set the file to skip formatting (rare).

**Using `gofumpt -extra` without team agreement.** The `-extra` rules are opinionated; surprise PR comments follow. Discuss before enabling.

**Trusting goimports to find any package by identifier.** It searches the module cache; if your dep isn't in `$GOMODCACHE`, the import isn't added. Run `go mod download` first.

**Not pinning tool versions.** `goimports` at `@latest` may behave differently between contributors. Pin via `go install pkg@v1.2.3` or `tool` directive (1.24+).

**Formatting vendor/ directories.** Most teams skip. Add `vendor/` to exclude lists.

**Running goimports on `*.tmpl` files thinking they're Go.** They're not; goimports errors. Use file extensions or globs.

**Treating gofumpt as a linter.** It's a formatter. Lint output is "this file would change"; fix by running with `-w`.

**Mixing `gofumpt` and `gofmt` saves in the same repo.** Round-trip churn: gofumpt adds rules, gofmt doesn't undo them but also doesn't enforce them. Pick one.

## Performance Notes

- `goimports` on a 1000-LoC file: ~30–100 ms.
- `gofumpt` on a 1000-LoC file: ~10–50 ms.
- `goimports -l .` on a 100k-LoC project: ~5–15 s (scans imports across module).
- `gofumpt -l .` on the same: ~2–5 s.

Editor-on-save runs are fast enough that you don't notice; CI runs are bounded by `go/packages` loading time.

## How Big Companies Use It

- **Google** uses `goimports` on every save; gofumpt is not standard in google3 but used in some open-source repos: https://github.com/golang/tools.
- **Kubernetes** uses `goimports` with `-local k8s.io`; gofumpt is not enforced: https://github.com/kubernetes/kubernetes.
- **Uber** uses `goimports` + gofumpt; documented in their Go style guide: https://github.com/uber-go/guide.
- **Cloudflare** uses `goimports`; some teams add gofumpt via golangci-lint: https://blog.cloudflare.com.
- **CockroachDB** uses `crlfmt` (their internal stricter formatter) which superset includes goimports semantics: https://github.com/cockroachdb/crlfmt.
- **HashiCorp** uses `goimports` + golangci-lint enforcement: https://github.com/hashicorp/terraform.
- **Tailscale** uses `goimports` + `gofumpt` via golangci-lint: https://github.com/tailscale/tailscale.
- **Discord's Go services** use gofumpt for consistency across team members.

## Source Code References

- `goimports`: [`golang.org/x/tools/cmd/goimports`](https://github.com/golang/tools/tree/master/cmd/goimports).
- `goimports` library: [`golang.org/x/tools/imports`](https://github.com/golang/tools/tree/master/imports).
- `gofumpt`: [`mvdan.cc/gofumpt`](https://github.com/mvdan/gofumpt).
- `gofumpt` format pkg: [`mvdan.cc/gofumpt/format`](https://github.com/mvdan/gofumpt/tree/master/format).
- gopls formatting: [`golang.org/x/tools/gopls/internal/lsp/source/format.go`](https://github.com/golang/tools/blob/master/gopls/internal/lsp/source/format.go).

(BSD-3-Clause © The Go Authors for goimports; BSD-3-Clause © Daniel Martí for gofumpt.)

## Further Reading

- "goimports docs": https://pkg.go.dev/golang.org/x/tools/cmd/goimports.
- "gofumpt added rules": https://github.com/mvdan/gofumpt#added-rules.
- "Using gofumpt": https://github.com/mvdan/gofumpt#installation.
- "Go formatting in VS Code": https://github.com/golang/vscode-go/wiki/features.
- "Uber Go style guide — Formatting": https://github.com/uber-go/guide/blob/master/style.md.
- Daniel Martí, "gofumpt: A stricter gofmt": https://mvdan.cc/blog.

## Exercises / Self-Check

1. Write a file with unused imports and missing imports. Run `goimports -w`. What got added/removed?
2. Use `-local github.com/me/proj` and see the three-group import block.
3. Run `gofumpt -d` on a file. Which rules fired? Are they all uncontroversial?
4. Enable `formatting.gofumpt: true` in gopls. Save a file with `var x int = 0`. What happens?
5. Compare the output of `gofmt` vs `gofumpt` vs `goimports` on the same file. Which is the strictest superset?
