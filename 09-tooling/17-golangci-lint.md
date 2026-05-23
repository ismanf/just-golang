# `golangci-lint` — The Meta-Linter

## TL;DR

**`golangci-lint`** is a lint *aggregator*: it loads a Go package once (via `go/packages`) and runs many linters against the same AST/type-checked input, reusing work and caching results aggressively. Out of the box (1.55+) it ships 70+ linters including `govet`, `staticcheck`, `gosimple`, `gosec`, `errcheck`, `revive`, `gocritic`, `gocyclo`, `prealloc`, `bodyclose`, `nilerr`, `nestif`, `wsl`, and so on. Configuration is YAML (`.golangci.yml`); you typically enable/disable linters, set thresholds, exclude rules per-path, and define output preferences. The single most important practice is **picking your linters deliberately** — running every linter at default settings produces thousands of findings and trains your team to ignore the tool. Pick 8–15 that match your goals; turn off the rest.

## Mental Model

```
   golangci-lint run ./...
        │
        ▼
   load packages via go/packages   (once)
        │
        ▼
   feed AST + types into:
        ├─ govet           (built-in vet analyzers)
        ├─ staticcheck     (SA*, ST*, etc.)
        ├─ errcheck        (unchecked error returns)
        ├─ gosec           (security smells)
        ├─ revive          (style + lint, configurable)
        ├─ gocritic        (large set of opinionated checks)
        ├─ ineffassign     (assignments that aren't used)
        ├─ ...
        │
        ▼
   aggregate findings → filter by config → format → exit non-zero if any
```

The key win: one type-check pass for many linters. Running each linter separately would cost 10–20× as much CPU.

## Syntax & Basic Usage

```bash
$ golangci-lint run                          # current dir
$ golangci-lint run ./...                    # whole module
$ golangci-lint run --fix                    # auto-apply fixes where supported
$ golangci-lint run --new                    # only new findings since main branch
$ golangci-lint run --new-from-rev=origin/main
$ golangci-lint run --timeout=5m             # default 1m; bump for big repos
$ golangci-lint run --out-format=json
$ golangci-lint run -E stylecheck            # enable extra linter
$ golangci-lint run -D errcheck              # disable a linter
$ golangci-lint linters                      # list enabled/disabled
$ golangci-lint help linters                 # list every available linter
$ golangci-lint config                       # show effective config
$ golangci-lint cache status
$ golangci-lint cache clean
```

## Deep Dive

### Installation

```bash
$ # binary (recommended for CI)
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.59.0

$ # via go install (slower, downloads deps)
$ go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.59.0

$ # Homebrew (macOS / Linux)
$ brew install golangci-lint
```

Pin a version. CI must match local for reproducible findings.

### Configuration: `.golangci.yml`

```yaml
run:
  timeout: 5m
  go: '1.26'
  build-tags:
    - integration
  modules-download-mode: readonly

linters:
  disable-all: true
  enable:
    - govet
    - staticcheck
    - gosimple
    - ineffassign
    - errcheck
    - unused
    - bodyclose
    - gocyclo
    - revive

linters-settings:
  gocyclo:
    min-complexity: 15
  errcheck:
    check-type-assertions: true
    check-blank: true
  revive:
    rules:
      - name: var-naming
      - name: exported

issues:
  exclude-rules:
    - path: _test\.go
      linters: [errcheck, gosec]
    - path: internal/proto/
      linters: [staticcheck, gocritic]
  max-issues-per-linter: 0
  max-same-issues: 0
  new-from-rev: origin/main      # in CI, only fail on new findings

output:
  formats:
    - format: colored-line-number
    - format: github-actions       # in CI
```

The `disable-all: true` + explicit `enable` list is the recommended pattern. The alternative (enabling defaults plus extras) drifts when `golangci-lint` updates its default set.

### Linter catalog (curated)

These are the linters worth seriously considering:

| Linter        | What it catches                                                              |
|---------------|------------------------------------------------------------------------------|
| `govet`       | Built-in `go vet`'s default analyzers.                                        |
| `staticcheck` | The `SA*` family — high-confidence correctness checks.                        |
| `gosimple`    | The `S*` family — simplifications (`if a {return true} else ...`).            |
| `stylecheck`  | The `ST*` family — style + naming.                                            |
| `unused`      | Dead code, unused funcs/types/vars.                                           |
| `ineffassign` | Assigned-but-never-read variables.                                            |
| `errcheck`    | Unchecked error returns.                                                      |
| `bodyclose`   | `resp.Body` not closed.                                                       |
| `gocyclo`     | Cyclomatic complexity per func.                                               |
| `gocognit`    | Cognitive complexity (subtly different).                                      |
| `gosec`       | Security smells (weak crypto, SQL injection patterns, etc.).                  |
| `revive`      | Configurable replacement for the deprecated `golint`.                          |
| `gocritic`    | Big bag of opinionated checks; pick rules carefully.                          |
| `nilerr`      | `return nil` after `err := ...; if err != nil`.                                |
| `nilnil`      | Returning `nil, nil` (almost always a bug).                                    |
| `errorlint`   | `errors.Is`/`As` vs. `==`/type assertion on errors.                            |
| `prealloc`    | Append in a loop that should `make([]T, 0, n)` first.                          |
| `nestif`      | Deeply nested `if` blocks.                                                     |
| `wsl`         | Whitespace style (controversial).                                              |
| `dupl`        | Duplicate code blocks above a threshold.                                        |
| `lll`         | Long lines (Go has no line-length rule; rarely used).                          |
| `funlen`      | Function length thresholds.                                                    |
| `cyclop`      | Cyclomatic complexity with per-pkg thresholds.                                 |
| `exhaustive`  | `switch` statements missing const cases (great with `stringer`-generated).     |
| `containedctx`| `context.Context` stored in a struct (anti-pattern).                            |
| `contextcheck`| Calls that should pass `context.Context` but use `Background()`.                |
| `noctx`       | HTTP/SQL calls without context.                                                |
| `revive`      | Replaces deprecated `golint`; configurable rule list.                          |
| `gofumpt`     | Stricter `gofmt` (see `09-tooling/22-goimports-gofumpt.md`).                   |
| `forbidigo`   | Forbid specific identifiers (e.g., `fmt.Println` in production).               |
| `depguard`    | Forbid imports of specific packages (e.g., `log` in favor of `slog`).          |

`golangci-lint help linters` shows the full list with current enabled/disabled status.

### Exclude rules

```yaml
issues:
  exclude-rules:
    - path: _test\.go
      linters: [errcheck, gosec, dupl]
    - path: internal/generated/
      linters: [staticcheck, gocritic, gofumpt]
    - text: "G404"           # weak random
      linters: [gosec]
    - linters: [revive]
      text: "should have comment"
      source: "^//go:generate"
```

`path` is a regex on file path. `text`/`source` match against the finding's message/source line. Use sparingly; broad excludes hide real bugs.

For one-off skips, in-source comments:

```go
//nolint:errcheck                // pin a single linter
_, _ = f.Close()

//nolint:errcheck,gosec          // pin multiple

//nolint                          // skip ALL linters; avoid
```

Always include a comment explaining why:

```go
//nolint:gosec // G304: path is from trusted config, not user input
```

### `--fix` and auto-fixers

```bash
$ golangci-lint run --fix
```

Applies fixes from linters that support them. Most of `gofumpt`, `goimports`, `gofmt`, some `gocritic` rules, and a few staticcheck rules have fixers.

CI typically runs without `--fix` (just reports). Locally, run with `--fix` then review the diff.

### `--new` and `--new-from-rev`

```bash
$ golangci-lint run --new-from-rev=origin/main
```

Only report findings introduced since `origin/main`. Critical for adopting `golangci-lint` on a legacy codebase: don't try to fix every existing issue at once.

### CI patterns

GitHub Actions:

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-go@v5
  with: { go-version: '1.26' }
- uses: golangci/golangci-lint-action@v6
  with:
    version: v1.59.0
    args: --timeout=10m
```

For `--new`:

```yaml
- uses: golangci/golangci-lint-action@v6
  with:
    version: v1.59.0
    args: --new-from-rev=origin/${{ github.base_ref }}
```

### Aggregating output

```bash
$ golangci-lint run --out-format=json | jq '.Issues[].Text'
```

Built-in formats: `colored-line-number`, `line-number`, `json`, `junit-xml`, `checkstyle`, `code-climate`, `github-actions`, `teamcity`, `sarif`. Some tools (CodeQL ingestion, GH PR annotations) want specific formats.

### Cache

`golangci-lint` keeps a cache in `$HOME/.cache/golangci-lint/` (Linux) or platform equivalents. It speeds repeat runs ~10×. Wipe with `golangci-lint cache clean`.

In CI, persist the cache between runs:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/golangci-lint
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-golangci-${{ hashFiles('**/go.sum', '.golangci.yml') }}
```

### Choosing linters: a starter set

For a fresh project:

```yaml
linters:
  disable-all: true
  enable:
    - govet
    - staticcheck
    - gosimple
    - unused
    - ineffassign
    - errcheck
    - bodyclose
    - errorlint
    - revive
    - gocritic
    - gosec
```

About 10. Add `gocyclo` if you want complexity budgets; `exhaustive` if you have enum-like consts; `noctx` if you're strict about context propagation. Don't enable `wsl`, `lll`, `funlen` until your team agrees — they're style choices.

### Migration from `golint` / `gometalinter`

`golint` is deprecated; replace with `revive` (configurable) or `stylecheck` (the `ST*` family from staticcheck).

`gometalinter` is unmaintained; `golangci-lint` is its successor.

### Per-package config

There isn't a per-package config file. Use `issues.exclude-rules` with `path:` regexes:

```yaml
issues:
  exclude-rules:
    - path: ^internal/legacy/
      linters: [errcheck, gocyclo]
```

### `go run` vs. installed binary

`go run github.com/golangci/golangci-lint/cmd/golangci-lint@v1.59.0 run ./...` works but is slow (compiles every invocation). For day-to-day, install once.

## Standard Library Hooks

- `golang.org/x/tools/go/analysis` — the underlying analyzer framework most linters use.
- `golang.org/x/tools/go/packages` — package loading.
- `go/ast`, `go/types`, `go/token` — AST and types.
- Many individual linter modules: `honnef.co/go/tools/...` (staticcheck/gosimple/stylecheck/unused), `github.com/kisielk/errcheck`, `github.com/timakin/bodyclose`, etc.

## Real-World Patterns

### 1. Minimal config

```yaml
linters:
  disable-all: true
  enable:
    - govet
    - staticcheck
    - errcheck
    - ineffassign
    - unused
```

### 2. Legacy-codebase adoption

```bash
$ golangci-lint run --new-from-rev=origin/main ./...
```

CI only fails on new findings; gradually fix the existing ones.

### 3. Pre-commit hook

```bash
#!/bin/sh
# .git/hooks/pre-commit
golangci-lint run --new-from-rev=HEAD || exit 1
```

### 4. Per-path excludes

```yaml
issues:
  exclude-rules:
    - path: _test\.go
      linters: [errcheck, gosec, dupl]
    - path: vendor/
      linters: [all]
```

### 5. Forbid an import

```yaml
linters: { enable: [depguard] }
linters-settings:
  depguard:
    rules:
      main:
        deny:
          - pkg: "log"
            desc: "use log/slog instead"
          - pkg: "math/rand"
            desc: "use math/rand/v2"
```

### 6. Forbid an identifier

```yaml
linters: { enable: [forbidigo] }
linters-settings:
  forbidigo:
    forbid:
      - 'fmt\.Print.*'    # no printf in production
      - 'os\.Exit'        # no exit outside main
```

### 7. Make a Makefile target

```makefile
.PHONY: lint
lint:
	golangci-lint run --timeout=5m ./...

.PHONY: lint-fix
lint-fix:
	golangci-lint run --fix ./...
```

## Anti-Patterns & Gotchas

**Enabling every linter.** Hundreds of findings; team learns to ignore the tool. Start with 8–15 deliberate choices.

**Auto-fixing in CI.** `--fix` modifies files; CI should report, not edit. Run locally; commit the diff.

**Pinning to `master`.** Behavior drifts between versions. Pin a tag (`v1.59.0`).

**Long `//nolint` lines without justification.** Future readers can't tell if it's intentional. Always add a reason.

**Treating `gocyclo` thresholds as gospel.** A complex switch in a parser is necessary complexity. Tune per package via `exclude-rules`.

**Ignoring `staticcheck.SA*` warnings.** They're high-confidence. If a `SA*` rule false-positives often, file an upstream issue rather than disable.

**Excluding `_test.go` from everything.** Tests deserve linting too — they're part of the codebase. Exclude only narrow categories that don't make sense in test code (e.g., `gosec` for cryptographic test fixtures).

**Letting `wsl` (whitespace style) decide your style.** Highly opinionated; many teams hate it. Try it but be ready to disable.

**Running `golangci-lint` and `staticcheck` separately.** You're paying for two type-checks. Use `golangci-lint`'s built-in `staticcheck` linter.

**No timeout in CI.** Default is 1 minute; large repos run longer. Set `--timeout=10m` or in YAML.

**Per-file config files.** They don't exist; use `exclude-rules` with `path:` regexes.

**`disable-all: true` + `enable:` *and* expecting future linters to be auto-included.** That's why you used `disable-all: true`. New defaults are opt-in.

**Trusting `forbidigo`/`depguard` to enforce architecture.** They catch direct usage but not transitive (a forbidden package imported by an allowed wrapper). Combine with code review.

## Performance Notes

- First run (cold cache): 30 s – 5 min for medium projects.
- Warm cache: 5–30 s.
- Memory: 500 MB – 4 GB depending on linter set and project size.
- `--new-from-rev` saves time on incremental runs but still must load packages.
- Each enabled linter adds 5–20% to runtime.

If running over 5 minutes routinely, prune linters or split into `lint-fast` (subset, runs on every PR) and `lint-full` (everything, runs nightly).

## How Big Companies Use It

- **Kubernetes** uses `golangci-lint` with a curated subset; their `.golangci.yml` is in the repo: https://github.com/kubernetes/kubernetes/blob/master/hack/golangci.yaml.
- **Uber** uses `golangci-lint` with custom-built linters loaded via `-vettool`: https://github.com/uber-go/guide.
- **HashiCorp** ships a baseline `.golangci.yml` across Terraform provider repos: https://github.com/hashicorp/terraform.
- **CockroachDB** uses `golangci-lint` plus custom `crl-lint` (their internal linter): https://github.com/cockroachdb/cockroach.
- **Cloudflare** uses `golangci-lint --new-from-rev` to enforce on PRs without rewriting legacy code: https://blog.cloudflare.com.
- **Discord** uses a strict `golangci-lint` config gated by `--new` for monorepo-scale linting.
- **Tailscale** uses `golangci-lint` + `depaware` (their custom analyzer) for import-graph enforcement: https://github.com/tailscale/depaware.

## Source Code References

`golangci-lint` is independently versioned.

- Source: [`github.com/golangci/golangci-lint`](https://github.com/golangci/golangci-lint).
- Linter wrappers: [`pkg/golinters`](https://github.com/golangci/golangci-lint/tree/master/pkg/golinters).
- Config schema: [`pkg/config`](https://github.com/golangci/golangci-lint/tree/master/pkg/config).
- Cache impl: [`internal/cache`](https://github.com/golangci/golangci-lint/tree/master/internal/cache).
- Loader: [`pkg/lint`](https://github.com/golangci/golangci-lint/tree/master/pkg/lint).
- GH action: [`golangci/golangci-lint-action`](https://github.com/golangci/golangci-lint-action).

(GPL-3.0 license; some linters under their own licenses.)

## Further Reading

- "golangci-lint docs": https://golangci-lint.run.
- "Configuration reference": https://golangci-lint.run/usage/configuration.
- "Linters list": https://golangci-lint.run/usage/linters.
- "Performance" (golangci-lint blog): https://golangci-lint.run/usage/performance.
- "How to use golangci-lint" (Dan Eddy): https://www.gopherguides.com/articles/how-to-use-golangci-lint.
- "Standardized lint configurations" (Stripe): https://stripe.com/blog/online-migrations.

## Exercises / Self-Check

1. Adopt `golangci-lint` on a project with `disable-all: true` and an explicit `enable:` list of 8 linters. What did you pick? Why?
2. Run with `--new-from-rev=origin/main` on a feature branch. Compare to a full run — what's the difference?
3. Add a `depguard` rule forbidding `log` in favor of `log/slog`. Test by importing `log` somewhere; does it fire?
4. Add a `//nolint:errcheck // <reason>` exclusion and verify the linter respects it.
5. Configure `gocyclo` with a complexity threshold of 20. Find one function that exceeds it; refactor.
