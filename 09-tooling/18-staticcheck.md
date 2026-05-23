# `staticcheck` — The High-Confidence Linter

## TL;DR

**`staticcheck`**, by Dominik Honnef, is the third-party linter Go developers reach for when `go vet` isn't enough. It's high-confidence: every check has a stable ID (`SA1019`, `S1000`, `ST1003`, `QF1001`) and a documented rationale. Findings *are* almost always real bugs or unambiguously inferior code. The checks split into four families: **SA** (staticcheck/correctness), **S** (gosimple/simplifications), **ST** (stylecheck/style), and **QF** (quickfix/refactor). It can run standalone, as a `-vettool=` for `go vet`, embedded in `gopls`, or wrapped by `golangci-lint`. The biggest reason to know it independently is the **`SA*` catalog** — it's a curriculum in "things gc will accept but you shouldn't write".

## Mental Model

```
   staticcheck ./...
        │
        ▼
   load packages via go/packages   (once)
        │
        ▼
   for each enabled check:
        ├─ SA*  ── correctness (deprecations, deadlocks, unreachable, ...)
        ├─ S*   ── simplifications (`if a {return true} else {return false}` → `return a`)
        ├─ ST*  ── style/naming (exported funcs need comments, error vars named ErrFoo, ...)
        └─ QF*  ── quickfix-able refactors (apply with -fix)
        │
        ▼
   emit diagnostics; exit non-zero if any
```

A check fires only when staticcheck is *confident*. False positives exist but are rare and worth filing as bugs.

## Syntax & Basic Usage

```bash
$ go install honnef.co/go/tools/cmd/staticcheck@2024.1.1
$ staticcheck ./...
$ staticcheck -f text ./...
$ staticcheck -f json ./...
$ staticcheck -f stylish ./...
$ staticcheck -checks "all,-ST1003" ./...    # all except ST1003
$ staticcheck -checks "SA*,S1000" ./...      # specific
$ staticcheck -explain SA1019                 # show what a check does
$ staticcheck -list-checks
$ staticcheck -show-ignored ./...            # include silenced findings
$ staticcheck -tags=integration ./...
$ staticcheck -unused.whole-program ./...     # whole-program dead-code analysis
$ staticcheck -fix ./...                      # apply quickfixes (QF* only)
```

## Deep Dive

### The four check families

**SA (Staticcheck / Correctness)** — bugs and likely-incorrect code:

| ID       | What it catches                                                     |
|----------|---------------------------------------------------------------------|
| `SA1000` | Invalid regex (won't compile at runtime).                            |
| `SA1006` | `Printf` with no format string but trailing args.                    |
| `SA1019` | Use of a `Deprecated:`-marked function/var.                          |
| `SA1029` | Inappropriate context.Value key types (non-comparable, basic types). |
| `SA2000` | `sync.WaitGroup.Add` inside the goroutine it counts.                 |
| `SA4006` | Value assigned but never read (more aggressive than `ineffassign`).  |
| `SA4017` | Discarded result of a function with no side effects.                 |
| `SA5000` | Possible nil pointer dereference.                                     |
| `SA5008` | Invalid struct tag.                                                   |
| `SA5012` | Passing odd-sized slice to a function expecting even.                 |
| `SA6005` | `strings.ToLower(s) == strings.ToLower(t)`: use `strings.EqualFold`. |
| `SA9003` | Empty `if` branch.                                                    |

`staticcheck -list-checks` lists ~150 SA rules.

**S (gosimple)** — simplifications:

| ID      | What it catches                                                  |
|---------|------------------------------------------------------------------|
| `S1000` | `select{}` for a single case → use direct receive.               |
| `S1002` | `if b == true` → `if b`.                                          |
| `S1004` | `bytes.Equal(a, []byte("x"))` → `string(a) == "x"`.              |
| `S1011` | `for _, x := range s { dst = append(dst, x) }` → `append(dst, s...)`. |
| `S1019` | `make([]T, 0)` → `[]T{}`.                                         |
| `S1023` | Redundant `return` at end of void func.                           |
| `S1025` | Don't `Sprintf` to convert a string to a string.                  |
| `S1030` | `string(b)` then bytes again → just use `b`.                      |
| `S1038` | `Errorf` with no formatting verbs → `errors.New`.                 |

**ST (stylecheck)** — style and naming:

| ID      | What it catches                                                  |
|---------|------------------------------------------------------------------|
| `ST1000` | Package has no documentation comment.                             |
| `ST1003` | Naming convention (use `MixedCaps`, not `snake_case`).            |
| `ST1005` | Error string starts with capital letter or ends with punctuation. |
| `ST1006` | Receiver names should be 1-2 chars.                               |
| `ST1011` | Loop variable shadows outer.                                      |
| `ST1015` | Default case in `switch` should be first or last.                 |
| `ST1016` | Inconsistent receiver name within a type.                         |
| `ST1018` | Avoid zero-width or control characters in string literals.        |
| `ST1019` | Don't import the same package twice.                              |
| `ST1020` | Exported function comment should start with function name.        |
| `ST1023` | Redundant explicit type in var declaration.                       |

ST checks are stylistic; teams pick which to enable.

**QF (quickfix)** — applicable refactors:

| ID      | What it does                                                     |
|---------|------------------------------------------------------------------|
| `QF1001` | Apply De Morgan's law (`!(a && b)` → `!a || !b`).               |
| `QF1002` | Convert untagged switch to tagged.                                |
| `QF1003` | Convert if/else chain to switch.                                  |
| `QF1006` | Lift `if` into the surrounding loop condition.                    |
| `QF1009` | Use `time.Time.Equal` instead of `==`.                            |
| `QF1011` | Omit redundant type in var declaration.                            |

`staticcheck -fix` applies QF auto-fixes.

### `SA1019` — deprecation tracker

```go
import "io/ioutil"          // deprecated in 1.16

ioutil.ReadFile("...")      // SA1019: io/ioutil is deprecated; use os and io packages instead
```

Staticcheck reads `// Deprecated:` markers in any package — yours, stdlib, third-party. Any call to a deprecated symbol fires `SA1019`. This is the single most useful check in big codebases.

To suppress for one call:

```go
//lint:ignore SA1019 // legacy code path; migrate in v2
data, _ := ioutil.ReadFile(path)
```

### Suppression syntax

```go
// Suppress a single line:
//lint:ignore SA1019 reason
data := callDeprecated()

// Suppress an entire file:
//lint:file-ignore SA1019 // whole file uses an old API

// Suppress a function:
//lint:ignore SA4006,SA1019 reason
func foo() { ... }
```

The reason after the IDs is mandatory; staticcheck warns if you omit it.

### Configuration

Staticcheck reads `staticcheck.conf` in the module root or any subdirectory:

```toml
checks = ["all", "-ST1000", "-ST1020", "-ST1021", "-ST1022"]
initialisms = ["ACL", "API", "ASCII", "CPU", "CSS", "DNS", "EOF", "GUID", "HTML", "HTTP", "HTTPS", "ID", "IP", "JSON", "QPS", "RAM", "RPC", "SLA", "SMTP", "SQL", "SSH", "TCP", "TLS", "TTL", "UDP", "UI", "GID", "UID", "UUID", "URI", "URL", "UTF8", "VM", "XML", "XMPP", "XSRF", "XSS"]
dot_import_whitelist = []
http_status_code_whitelist = ["200", "400", "404", "500"]
```

`checks = ["all"]` enables every check; `-ID` disables. Pin lists per-directory by placing a `staticcheck.conf` there.

### Embedding via `golangci-lint`

```yaml
linters:
  enable:
    - staticcheck
    - gosimple
    - stylecheck
    - unused
linters-settings:
  staticcheck:
    checks: ["all"]
  stylecheck:
    checks: ["all", "-ST1000"]
```

`golangci-lint` shares the same checker code; you don't need a separate `staticcheck` install if you're already running `golangci-lint`.

### `go vet -vettool=staticcheck`

```bash
$ go vet -vettool=$(which staticcheck) ./...
```

Or via `go test`:

```bash
$ go test -vet=off ./...        # disable built-in vet
$ staticcheck ./...             # then run staticcheck
```

Some teams stack both: built-in vet's high-signal checks + staticcheck's deeper analysis.

### `gopls` integration

```jsonc
"gopls": { "ui.diagnostic.staticcheck": true }
```

When enabled, gopls runs `SA*`, `S*`, and `ST*` checks inline as you edit. The single biggest editor-side productivity boost.

### Unused-code analysis

```bash
$ staticcheck -unused.whole-program ./...
```

`U1000` (unused symbol) is the unused-code check. Default mode is per-package; `--whole-program` looks across the whole module. Slower but catches more.

### Performance

- Standalone `staticcheck ./...` on a medium project: 30 s – 2 min.
- Via `golangci-lint`: shares the type-check; ~30% extra over base.
- Inline via `gopls`: latency hidden in editor's incremental analyzer.

### Versioning

Staticcheck releases are dated: `2024.1.1`, `2024.1.0`, `2023.1.7`, etc. Each release pins compatibility with one or two Go versions. Check the release notes for compatibility.

```bash
$ go install honnef.co/go/tools/cmd/staticcheck@2024.1.1
$ staticcheck -version
2024.1.1 (go1.26)
```

### CI usage

```yaml
- run: go install honnef.co/go/tools/cmd/staticcheck@2024.1.1
- run: staticcheck ./...
```

Or via `dominikh/staticcheck-action`:

```yaml
- uses: dominikh/staticcheck-action@v1.3.0
  with:
    version: "2024.1.1"
```

### Output formats

```bash
$ staticcheck -f text ./...        # default
$ staticcheck -f json ./...        # machine-readable
$ staticcheck -f stylish ./...     # grouped, color
$ staticcheck -f binary ./...      # protocol-buffer-style (for piping into tools)
$ staticcheck -f github ./...      # GH Actions annotations
$ staticcheck -f sarif ./...       # SARIF for code-scanning tools
```

### Self-update

```bash
$ go install honnef.co/go/tools/cmd/staticcheck@latest
```

Or pin in `go.mod`'s `tool` directive (1.24+):

```
tool honnef.co/go/tools/cmd/staticcheck
```

Then `go tool staticcheck ./...` uses the pinned version.

## Standard Library Hooks

- `golang.org/x/tools/go/analysis` — staticcheck wraps each rule as an `analysis.Analyzer`.
- `golang.org/x/tools/go/packages` — package loading.
- `honnef.co/go/tools/analysis/code` — internal analysis utilities.
- `honnef.co/go/tools/staticcheck`, `simple`, `stylecheck`, `quickfix`, `unused` — the four rule packages.

## Real-World Patterns

### 1. CI invocation

```yaml
- uses: dominikh/staticcheck-action@v1.3.0
  with: { version: "2024.1.1" }
```

### 2. Per-rule suppression in legacy code

```go
//lint:ignore SA1019 // legacy code path scheduled for removal in v3
data, _ := ioutil.ReadFile(path)
```

### 3. Module-wide config

```toml
# staticcheck.conf
checks = ["all", "-ST1000", "-ST1020", "-ST1021", "-ST1022"]
```

Disables the four `ST1xxx` checks that require package-level doc comments many teams find pedantic.

### 4. Tracking new deprecations

After upgrading stdlib (e.g., 1.21 deprecated several `math/rand` funcs in favor of `math/rand/v2`), `SA1019` fires across affected calls. Migrate one package at a time.

### 5. Stack with `golangci-lint`

```yaml
linters:
  enable: [staticcheck, gosimple, stylecheck, unused]
```

`golangci-lint` is the wrapper; `staticcheck` provides the rules. One config, one CLI.

### 6. Editor integration

VS Code (`.vscode/settings.json`):

```jsonc
{
  "gopls": { "ui.diagnostic.staticcheck": true }
}
```

### 7. `go tool staticcheck` (1.24+)

```bash
$ go get -tool honnef.co/go/tools/cmd/staticcheck@2024.1.1
$ go tool staticcheck ./...
```

## Anti-Patterns & Gotchas

**Disabling `SA1019` globally.** You lose visibility into deprecations. If a specific deprecation is acceptable, suppress *that* call site, not the rule.

**Running `staticcheck` and `golangci-lint`'s staticcheck linter separately.** Wastes a type-check pass. Pick one entry point.

**Adding `//lint:ignore` without a reason.** Staticcheck warns. Always include the rationale.

**Running with `checks = ["all"]` and ignoring noisy ST rules.** `ST1000` (package needs a comment) fires across every internal-only package. Disable explicitly if your team doesn't care.

**Pinning to `@latest`.** Bumps every install; behavior drifts. Pin a release.

**Treating `S*` (simplification) as mandatory.** Many `S*` rules are stylistic; some idiomatic Go intentionally takes the "long form" for clarity. Apply judgment.

**Confusing `SA4006` (unused write) with `ineffassign`.** They overlap but `SA4006` is more aggressive. Pick one if you don't want duplicate findings.

**Trusting `U1000` in a library.** Unused code in a library may be intentional public API for downstream consumers. Use `-unused.whole-program` only on application binaries.

**Letting `staticcheck.conf` and `golangci-lint`'s settings drift.** If both are present, you have two truths. Consolidate (typically use `golangci-lint`'s config).

**Running `staticcheck -fix` in CI.** Like other auto-fixers, fixing in CI silently rewrites code. Use locally and commit.

**Ignoring `SA9003` (empty if branch).** Often a comment is there explaining why nothing happens — but staticcheck still flags it. If intentional, add `// empty by design` and suppress.

## Performance Notes

- Cold `staticcheck ./...` on 100k LoC: 30–90 s.
- Warm: 5–15 s.
- Memory: 500 MB – 2 GB.
- Adds ~10–30% to `golangci-lint` runtime when enabled there.

Caching is on by default at `$HOME/.cache/staticcheck/`.

## How Big Companies Use It

- **Google** uses `staticcheck` checks via gopls; many internal codebases enable the full `SA*` set: https://github.com/dominikh/go-tools.
- **Kubernetes** uses `staticcheck` via `golangci-lint`; specific `SA1019` checks gate deprecation cleanups: https://github.com/kubernetes/kubernetes.
- **Uber** runs `staticcheck` as part of their internal lint stack: https://github.com/uber-go.
- **CockroachDB** uses `staticcheck` + custom rules; CI fails on any new `SA*` finding: https://github.com/cockroachdb/cockroach.
- **HashiCorp** uses `staticcheck` via `golangci-lint` across Terraform repos: https://github.com/hashicorp/terraform.
- **Tailscale** uses `staticcheck` + their custom analyzer `depaware`: https://github.com/tailscale/depaware.
- **Discord** uses `staticcheck` baseline with team-level enables for additional `ST*` checks.

## Source Code References

`staticcheck` is independently versioned at `honnef.co/go/tools`.

- Source: [`honnef.co/go/tools`](https://github.com/dominikh/go-tools).
- SA checks: [`staticcheck`](https://github.com/dominikh/go-tools/tree/master/staticcheck).
- S checks: [`simple`](https://github.com/dominikh/go-tools/tree/master/simple).
- ST checks: [`stylecheck`](https://github.com/dominikh/go-tools/tree/master/stylecheck).
- QF checks: [`quickfix`](https://github.com/dominikh/go-tools/tree/master/quickfix).
- Unused: [`unused`](https://github.com/dominikh/go-tools/tree/master/unused).
- Action: [`dominikh/staticcheck-action`](https://github.com/dominikh/staticcheck-action).

(MIT License.)

## Further Reading

- "staticcheck documentation": https://staticcheck.dev/docs.
- "Checks list with explanations": https://staticcheck.dev/docs/checks.
- "staticcheck configuration": https://staticcheck.dev/docs/configuration.
- "Why staticcheck doesn't have an explicit suppression option" (Dominik Honnef): https://staticcheck.dev/docs/configuration/#ignoring-problems.
- "Honnef's go/tools" (Dominik Honnef GopherCon talk): https://www.youtube.com/watch?v=OW01PfjeYwY.
- "Staticcheck migration guide": https://staticcheck.dev/docs/getting-started.

## Exercises / Self-Check

1. Run `staticcheck ./...` on a project that hasn't been linted. Pick the first `SA*` finding — is it a real bug?
2. Use `staticcheck -explain SA1019`. Where does the deprecation marker come from?
3. Find a function that fires `S1011` (loop append to slice). Apply the suggested transformation. Does it pass `staticcheck -fix`?
4. Enable `ST1003` (naming) on a legacy package. How many findings? How many are intentional?
5. Configure `staticcheck.conf` to enable `all` except the four `ST10xx` doc-related rules. Verify with `staticcheck -show-ignored`.
