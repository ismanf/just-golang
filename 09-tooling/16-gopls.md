# `gopls` — The Official Go Language Server

## TL;DR

**`gopls`** (pronounced "go please") is the Language Server Protocol implementation for Go, written and maintained by the Go team. Every modern editor (VS Code, Neovim, Emacs, GoLand-replacement plugins, Zed) uses it for hover info, completion, go-to-definition, find-references, rename, format-on-save, code actions (extract function, fill struct, organize imports), inline diagnostics (compile errors, vet warnings, staticcheck if enabled), workspace symbol search, and incremental analysis. It's the single common denominator for Go tooling outside the build — there is *no* alternative LSP shipped by the Go team. Configure via your editor's settings, but the canonical place is the `gopls` settings object passed via LSP initialization. Run `gopls -h` for CLI use (rare); the daemon is normally invoked by your editor and stays resident for the editor's session.

## Mental Model

```
   Editor (VS Code / nvim / Emacs / Zed)
        │
        ▼
   LSP JSON-RPC over stdio
        │
        ▼
   gopls daemon
        ├─ load packages   ── go/packages over current workspace
        ├─ build index     ── symbols, references, types
        ├─ run analyzers   ── go vet, optionally staticcheck
        ├─ hot-reload      ── incremental on file save
        ▼
   responses: hover, completion, diagnostics, code actions, formatting, ...
```

`gopls` understands modules, workspaces (`go.work`), build tags, generated code, and tests. It re-uses the build cache; first-load is dominated by parsing+type-checking, subsequent edits are incremental.

## Syntax & Basic Usage

```bash
$ go install golang.org/x/tools/gopls@latest    # install
$ gopls version                                  # check version
$ gopls -h                                       # cli help
$ gopls api-json                                 # dump full config schema as JSON
$ gopls check ./...                              # batch run analyzers
$ gopls codelens ./...                           # list available code lenses
$ gopls definition path:line:col                 # CLI go-to-definition
$ gopls references path:line:col                 # CLI find-refs
$ gopls fix ./...                                # apply quickfixes batch
$ gopls semtok path                              # semantic tokens for syntax highlighting
$ gopls remote start                             # run as a network daemon (rare)
```

Most users never run `gopls` from CLI — the editor handles it.

## Deep Dive

### Editor integration

| Editor          | How to enable                                                          |
|-----------------|------------------------------------------------------------------------|
| VS Code         | Install "Go" extension; auto-installs gopls.                           |
| Neovim          | `nvim-lspconfig` + `lua require'lspconfig'.gopls.setup{}`.             |
| Emacs           | `eglot` or `lsp-mode` + gopls on PATH.                                 |
| Zed             | Built-in; auto-downloads.                                              |
| Sublime Text    | LSP plugin + gopls.                                                    |
| Vim (classic)   | `vim-go` (uses gopls under the hood) or `vim-lsp`.                     |
| GoLand          | Uses its own analyzer (not gopls); but supports LSP optionally.        |

### Capabilities (the LSP surface)

- **Hover**: type info, doc comment, link to docs.
- **Completion**: identifiers, struct field literals, snippets.
- **Signature help**: parameter info while typing inside a call.
- **Definition / Type definition / Implementation**.
- **References**: find every usage across workspace.
- **Rename**: project-wide rename of an identifier.
- **Code actions**: organize imports, fill struct, extract function/variable, generate test, add tags, simplify expression.
- **Diagnostics**: compile errors, vet warnings, staticcheck (if enabled), inlay hints for type info.
- **Formatting**: `gofmt` + `goimports` integrated.
- **Workspace symbol search**: `Cmd+T` / `Ctrl+T` in most editors.
- **Document symbols**: outline view.
- **Folding ranges**: collapse blocks.
- **Code lens**: "run test", "generate" buttons inline.
- **Semantic tokens**: precise syntax highlighting based on type info.
- **Call hierarchy**: callers and callees of a function (LSP 3.16+).

### Configuration schema

`gopls` accepts a nested config object. Common keys:

```jsonc
{
  "gopls": {
    "ui.semanticTokens": true,
    "ui.completion.usePlaceholders": true,
    "ui.completion.experimentalPostfixCompletions": true,
    "ui.diagnostic.staticcheck": true,
    "ui.diagnostic.analyses": {
      "shadow": true,
      "fieldalignment": false,
      "unusedwrite": true,
      "nilness": true
    },
    "ui.codelenses": {
      "gc_details": false,
      "generate": true,
      "regenerate_cgo": true,
      "tidy": true,
      "upgrade_dependency": true,
      "vendor": true,
      "run_govulncheck": true
    },
    "ui.inlayhint.hints": {
      "assignVariableTypes": true,
      "compositeLiteralFields": true,
      "compositeLiteralTypes": true,
      "constantValues": true,
      "functionTypeParameters": true,
      "parameterNames": true,
      "rangeVariableTypes": true
    },
    "ui.documentation.linksInHover": true,
    "build.buildFlags": ["-tags=integration"],
    "build.env": {"GOPRIVATE": "*.corp.example.com"},
    "build.directoryFilters": ["-vendor", "-third_party"]
  }
}
```

Dump the full schema: `gopls api-json | jq '.Options.User'`.

### Static analysis integration

`gopls` runs `go vet`'s default analyzers automatically. Enable additional ones:

```jsonc
"ui.diagnostic.analyses": {
  "shadow": true,           // variable shadowing
  "fieldalignment": true,   // struct memory layout
  "nilness": true,           // simple nil-flow analysis
  "unusedwrite": true,       // unused writes
  "unusedparams": false      // off by default; noisy
}
```

For **staticcheck** integration:

```jsonc
"ui.diagnostic.staticcheck": true
```

Runs the staticcheck `SA*` checks inline. See `09-tooling/18-staticcheck.md`.

### Code actions ("quick fixes")

When the cursor is on a diagnostic, the editor offers actions:

- **Organize imports** — adds missing, removes unused, sorts groups.
- **Fill struct** — generate field assignments for a `T{}` literal.
- **Extract function / variable** — refactor selection.
- **Add tags** — generate struct tags (json, yaml, etc.) from a tag pattern.
- **Generate test** — create a `Test*` skeleton for the function at the cursor.
- **Implement interface** — generate method stubs for `var _ I = (*T)(nil)`.
- **Add nil check** — wrap with `if err != nil`.

VS Code surfaces these via Cmd+. (Mac) or Ctrl+. (others).

### Inlay hints

```jsonc
"ui.inlayhint.hints": {
  "parameterNames": true,
  "assignVariableTypes": true
}
```

Editor shows greyed-out hints inline:

```go
foo.Bar(/* name: */ "x", /* count: */ 3)
n := /* int */ len(s)
```

Helpful when reading unfamiliar code; some teams find them noisy.

### Code lenses

Inline buttons rendered above relevant declarations:

- **Run test** above each `Test*` function.
- **Generate** above each `//go:generate` directive.
- **go mod tidy / upgrade** above the `module` line of `go.mod`.

Enable per-lens:

```jsonc
"ui.codelenses": {
  "generate": true,
  "test": true,
  "tidy": true,
  "vendor": false,
  "run_govulncheck": true
}
```

`run_govulncheck` (since gopls v0.13) runs `govulncheck` over the module on demand.

### Workspaces and build tags

`gopls` understands `go.work` automatically — it loads every `use`d module. For build tags:

```jsonc
"build.buildFlags": ["-tags=integration,prod"]
```

Without this, files behind those tags appear unanalyzed.

### Generated files

Files starting with `// Code generated ... DO NOT EDIT.` are marked as generated. `gopls`:

- Skips most refactoring inside them.
- Doesn't suggest manual edits via code actions.
- Still type-checks (errors are reported).

### Performance: memory and CPU

On a large module (Kubernetes-sized, ~5M LoC), `gopls` uses:

- 2–4 GB RAM at idle (full type-checked index).
- 1–5% CPU at idle.
- ~10–60 s for initial workspace load.

Per-file diagnostics on save: <500 ms typical.

Reduce memory:

```jsonc
"ui.semanticTokens": false           // skip semantic highlighting
"ui.diagnostic.staticcheck": false   // skip staticcheck
"build.directoryFilters": ["-vendor"]
```

### `gopls` CLI mode

```bash
$ gopls check ./...                  # batch analyze (CI use)
$ gopls fix ./...                    # batch apply quickfixes
$ gopls workspace_symbol "MyType"    # find a symbol across workspace
$ gopls definition main.go:10:5      # CLI go-to-definition
$ gopls semtok main.go               # semantic tokens (JSON)
```

CLI use is mostly for CI, e.g., `gopls check ./...` as an extra lint pass.

### `gopls vulncheck` and `govulncheck`

`gopls` integrates `govulncheck` (`09-tooling/20-govulncheck.md`) under the `run_govulncheck` code lens, but it doesn't replace the standalone command.

### Caching and invalidation

`gopls` shares the same on-disk caches as `go build` (`$GOCACHE`, `$GOMODCACHE`). Edits don't invalidate the on-disk cache until you `go build`. Saving a file makes `gopls` re-parse just that file plus dependents in memory; the on-disk cache catches up on the next `go build`/`go test`.

### Distributed mode

```bash
$ gopls serve -listen=tcp://localhost:8081
```

Runs `gopls` as a long-lived server other editors connect to. Rare; mostly used for shared development environments (CodeSandbox, GitHub Codespaces). Most setups run an in-process gopls per editor.

### Versioning

`gopls` is versioned independently from Go: `gopls/v0.16.0` etc. Upgrade with:

```bash
$ go install golang.org/x/tools/gopls@latest
```

A `gopls` released against Go 1.26 typically supports Go 1.25 and 1.26 toolchains.

## Standard Library Hooks

- `golang.org/x/tools/go/analysis` — analyzer framework.
- `golang.org/x/tools/go/packages` — workspace-aware package loader.
- `golang.org/x/tools/internal/lsp` — the LSP server (internal but the source).
- `go/types`, `go/ast`, `go/parser`, `go/format` — the core language-server primitives.
- `golang.org/x/tools/refactor/rename` — rename logic.
- `golang.org/x/tools/imports` — `goimports` (used for "organize imports").

## Real-World Patterns

### 1. VS Code minimum config

```jsonc
{
  "go.useLanguageServer": true,
  "[go]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "golang.go",
    "editor.codeActionsOnSave": { "source.organizeImports": "explicit" }
  },
  "gopls": {
    "ui.diagnostic.staticcheck": true,
    "ui.completion.usePlaceholders": true
  }
}
```

### 2. Neovim minimum config

```lua
require'lspconfig'.gopls.setup{
  settings = {
    gopls = {
      analyses = { shadow = true, unusedwrite = true },
      staticcheck = true,
      hints = { parameterNames = true, assignVariableTypes = true },
    }
  }
}
```

### 3. Enable struct field alignment hints

```jsonc
"gopls": {
  "ui.diagnostic.analyses": { "fieldalignment": true }
}
```

Will surface struct layout warnings inline. Apply via code action.

### 4. Workspace with build tags

```jsonc
"gopls": {
  "build.buildFlags": ["-tags=integration"]
}
```

### 5. Per-project `gopls` settings

Create `.editorconfig`-style or per-project `.vscode/settings.json` to override.

### 6. CI gopls check

```yaml
- run: go install golang.org/x/tools/gopls@latest
- run: gopls check ./...
```

Useful for ensuring gopls-only diagnostics (not in `go vet` defaults) don't regress.

### 7. Run govulncheck via code lens

```jsonc
"gopls": { "ui.codelenses": { "run_govulncheck": true } }
```

Click "Run govulncheck" on `go.mod` to see vulnerable deps inline.

## Anti-Patterns & Gotchas

**Running multiple `gopls` instances per editor window.** Some setups accidentally launch a gopls per file. Check process list; gopls should be one per workspace root.

**Old gopls + new Go toolchain.** Mismatch can cause crashes on new syntax (e.g., type params). Update gopls when updating Go.

**Configuring `ui.diagnostic.analyses` to enable everything.** Many analyzers are off by default for false-positive rates. Enable selectively; review for noise.

**Letting `gopls` analyze `vendor/` and `node_modules/`.** Wastes memory; usually wrong. Use `build.directoryFilters` to exclude.

**Editing `go.mod` outside the editor while gopls is running.** Sometimes gopls misses external edits; restart the LSP if diagnostics get stale.

**Treating gopls inlay hints as ground truth.** They reflect type inference; if your type is `any`, the hint shows `interface{}` — useful but not always actionable.

**Using gopls's `fill struct` on a giant struct.** Will fill every field with zero values; cleaner code is to write only the ones you set. Use sparingly.

**Disabling format-on-save because "it changes my code".** That's the point. If formatting is causing churn, your team is divided between `gofmt` and `gofumpt`; pick one.

**Trying to use gopls to replace `staticcheck` entirely.** gopls integrates *some* staticcheck via opt-in; full coverage still requires running `staticcheck` separately (or via `golangci-lint`).

**Trusting gopls's "rename" across a workspace with multiple modules.** Cross-module rename works in a workspace; without `go.work`, references in other modules are missed.

**Setting `build.buildFlags` differently from the build system.** Editor sees files behind tag A; CI sees tag B. Keep them in sync via a shared config (e.g., `Makefile` derives both).

**Running `gopls` as a Snap/Flatpak under restricted permissions.** Sandboxing breaks access to `$GOMODCACHE`. Use the standard install.

## Performance Notes

- Workspace load (kubernetes-scale): 30–90 s.
- Workspace load (typical microservice): 1–5 s.
- Memory at idle (large workspace): 2–4 GB.
- Memory at idle (small): 200–500 MB.
- Hover/completion latency: <100 ms.
- Diagnostics on save: <500 ms.
- Workspace-wide rename (5 files): <500 ms; (500 files): few seconds.
- Find-references on common identifier: <1 s typical.

If memory grows unbounded, file a bug — known regressions land in patch releases.

## How Big Companies Use It

- **Google** uses `gopls` company-wide; it's the Go team's primary developer tool: https://github.com/golang/tools/tree/master/gopls.
- **Kubernetes** developers use `gopls` with custom `build.directoryFilters` to exclude generated boilerplate: https://github.com/kubernetes/kubernetes.
- **Uber** uses gopls plus internal config templates distributed via `dotfiles`: https://eng.uber.com.
- **Cloudflare** uses gopls with custom analyzers loaded via `-vettool` proxies: https://blog.cloudflare.com.
- **HashiCorp** uses gopls plus per-project `.vscode/settings.json` for tag-based builds: https://github.com/hashicorp/terraform.
- **Tailscale** uses gopls plus `staticcheck` integration; they author `depaware` as a custom analyzer: https://github.com/tailscale.
- **The Go team** uses gopls to test gopls (eat your own dogfood): https://github.com/golang/tools/tree/master/gopls.

## Source Code References

Pinned to `golang.org/x/tools/gopls` master (versioned independently of Go).

- gopls source: [`golang.org/x/tools/gopls`](https://github.com/golang/tools/tree/master/gopls).
- LSP server: [`golang.org/x/tools/gopls/internal/lsp`](https://github.com/golang/tools/tree/master/gopls/internal/lsp).
- Analysis integration: [`golang.org/x/tools/gopls/internal/lsp/source`](https://github.com/golang/tools/tree/master/gopls/internal/lsp/source).
- Code actions: [`golang.org/x/tools/gopls/internal/lsp/source/code_action.go`](https://github.com/golang/tools/blob/master/gopls/internal/lsp/source/code_action.go).
- Configuration: [`golang.org/x/tools/gopls/internal/settings`](https://github.com/golang/tools/tree/master/gopls/internal/settings).
- Analyzers: [`golang.org/x/tools/go/analysis/passes`](https://github.com/golang/tools/tree/master/go/analysis/passes).
- govulncheck integration: [`golang.org/x/tools/gopls/internal/lsp/source/vulncheck`](https://github.com/golang/tools/tree/master/gopls/internal/lsp/source).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "gopls user guide": https://github.com/golang/tools/blob/master/gopls/doc/user.md.
- "gopls settings": https://github.com/golang/tools/blob/master/gopls/doc/settings.md.
- "gopls release notes": https://github.com/golang/tools/blob/master/gopls/CHANGELOG.md.
- "Editor plugins for gopls": https://github.com/golang/tools/blob/master/gopls/doc/integration.md.
- "gopls performance": https://github.com/golang/tools/blob/master/gopls/doc/troubleshooting.md.
- "Workspace symbols and references" (Heschi Kreinick): https://go.dev/blog/gopls-scale.

## Exercises / Self-Check

1. Find the `gopls api-json` output for one option you've never seen. Try it in your editor.
2. Enable `fieldalignment` in `ui.diagnostic.analyses`. Find one struct that benefits from reordering.
3. Use "Generate test" code action on a function. Compare the skeleton it produces to what you'd write by hand.
4. Trigger a "Fill struct" code action. When is it helpful, and when is it noise?
5. Enable `staticcheck` in gopls. What new diagnostics appear that `go vet` didn't show?
