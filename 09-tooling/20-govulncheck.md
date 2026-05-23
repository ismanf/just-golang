# `govulncheck` — Vulnerability Scanning

## TL;DR

**`govulncheck`** is the Go team's official vulnerability scanner. It compares your module's build graph against the **Go vulnerability database** (`vuln.go.dev`) and reports only vulnerabilities whose vulnerable *function* is reachable from your code — not just present in `go.mod`. This call-graph-aware design eliminates the noise of typical "dep has a CVE" scanners. It works on source (`govulncheck ./...`) or on a built binary (`govulncheck ./bin`). The vulnerability DB is curated by the Go security team; sources include CVE feeds, GitHub Security Advisories, and direct module-owner reports. Run it in CI, as a pre-commit step, or via `gopls`'s `run_govulncheck` code lens.

## Mental Model

```
   govulncheck ./...
        │
        ▼
   load packages (go/packages)        ── what your build pulls in
        │
        ▼
   build call graph                    ── what your code actually invokes
        │
        ▼
   compare against vuln.go.dev:
        ├─ vulnerability per module-version
        ├─ vulnerable symbol(s) per CVE
        │
        ▼
   report only CVEs whose vulnerable symbol is REACHABLE
        │
        ▼
   exit non-zero if any reachable; 0 if not
```

A CVE in a dep that your code never calls is **not reported as a finding** (logged as informational, but no error). That alone dramatically reduces noise compared to generic SCA tools.

## Syntax & Basic Usage

```bash
$ go install golang.org/x/vuln/cmd/govulncheck@latest

$ govulncheck ./...                          # source mode
$ govulncheck -show=verbose ./...
$ govulncheck -json ./...                    # machine-readable
$ govulncheck -mode=binary ./bin             # binary mode
$ govulncheck -mode=source ./...             # explicit source mode (default)
$ govulncheck -tags=integration ./...        # respect build tags
$ govulncheck -test ./...                    # include test deps (default off)
$ govulncheck -C /path/to/proj ./...
$ govulncheck -scan=module ./...             # module-level only (faster, noisier)
$ govulncheck -scan=symbol ./...             # symbol-level (default)
$ govulncheck -version
```

## Deep Dive

### Why call-graph awareness matters

Traditional SCA tools say: "Your `go.mod` has `golang.org/x/crypto v0.8.0`, which has CVE-2023-XXXX." But if you never call the vulnerable function, the CVE doesn't apply to your binary.

`govulncheck` says: "Your `main.handler` calls `dep.Foo`, which calls `vulnerable.Bar`. Path:"

```
Vulnerability #1: GO-2023-1234
    Crash via malformed input.
  Found in: golang.org/x/crypto@v0.8.0
  Fixed in: golang.org/x/crypto@v0.15.0
  Affected: github.com/me/proj/cmd/app

    Call stacks in your code:
      cmd/app/main.go:42:5: main.main calls dep.Foo
      dep/dep.go:15:3: dep.Foo calls vulnerable.Bar
      vendor/.../vulnerable.go:30:1: vulnerable.Bar
```

Now you know: (1) this CVE actually affects you; (2) where in your code the chain originates; (3) what to upgrade to.

### Source vs. binary mode

```bash
$ govulncheck -mode=source ./...      # default: needs source
$ govulncheck -mode=binary ./bin      # built artifact
```

**Source mode**:
- Reads `.go` files; builds full call graph.
- Symbol-level precision.
- Needs build dependencies present.

**Binary mode**:
- Reads `runtime/debug.BuildInfo` from binary.
- Only module-level precision (since the call graph isn't reconstructable from the binary alone).
- Useful when scanning a binary you didn't build (e.g., a vendor's binary).

### Output formats

```bash
$ govulncheck ./...                    # human-readable
$ govulncheck -show=verbose ./...      # include reasoning
$ govulncheck -show=color ./...        # colorized
$ govulncheck -show=traces ./...       # full call chain depth
$ govulncheck -json ./...              # machine-readable
```

JSON output is consumable by SARIF converters and security dashboards:

```bash
$ govulncheck -json ./... | jq '.OSV[]'
```

### The vulnerability database

`vuln.go.dev` mirrors the OSV format. Each entry:

```yaml
id: GO-2023-1234
modified: 2023-05-01T00:00:00Z
published: 2023-04-15T00:00:00Z
aliases: [CVE-2023-12345, GHSA-xxxx]
summary: |
  Crash via malformed input
affected:
  - package:
      name: golang.org/x/crypto
      ecosystem: Go
    ranges:
      - type: SEMVER
        events:
          - introduced: 0
          - fixed: 0.15.0
    ecosystem_specific:
      imports:
        - path: golang.org/x/crypto/ssh
          symbols: [ParseDSAPrivateKey]
references:
  - type: FIX
    url: https://go.dev/cl/123456
```

The `imports.symbols` list is the key: govulncheck checks if your code reaches any of those symbols. Source: [`github.com/golang/vulndb`](https://github.com/golang/vulndb).

### Submitting vulnerabilities

If you find a vuln in a Go module you maintain:

1. File at `https://github.com/golang/vulndb/issues/new`.
2. The Go security team reviews and assigns a `GO-YYYY-NNNN` ID.
3. Add the symbols + version ranges.
4. Once merged, `govulncheck` runs detect it.

For first-party (Go stdlib) vulnerabilities, the Go security team coordinates with CVE.

### `gopls` integration

```jsonc
"gopls": {
  "ui.codelenses": { "run_govulncheck": true }
}
```

Adds a "Run govulncheck" lens above the `module` line in `go.mod`. Click → runs scan → annotates `go.mod` with vulnerabilities and `code actions` for upgrades.

### CI usage

GitHub Actions:

```yaml
- uses: golang/govulncheck-action@v1
  with:
    go-version-file: go.mod
    work-dir: .
```

Or directly:

```yaml
- run: go install golang.org/x/vuln/cmd/govulncheck@latest
- run: govulncheck ./...
```

Some teams gate merges; others run nightly and file issues. CI gating is appropriate when you can quickly upgrade; nightly is better for "we'll fix on our cadence".

### Stdlib vulnerabilities

govulncheck checks the standard library too:

```
Vulnerability #1: GO-2024-1234
    Buffer overflow in net/http
  Found in: net/http@go1.25.0
  Fixed in: net/http@go1.25.3
  Affected: github.com/me/proj/cmd/app
```

When stdlib is the source, upgrade Go (not your `go.mod` requires). The `go.mod` `toolchain` line + `GOTOOLCHAIN=auto` lets `go build` auto-fetch the fixed version.

### Symbol-level vs. module-level scan

```bash
$ govulncheck -scan=symbol ./...     # default; precise, slower
$ govulncheck -scan=module ./...     # all CVEs in any dep version, fast
```

Module mode is what other SCA tools do; use only if you need speed and accept noise.

### `-test` flag

```bash
$ govulncheck -test ./...
```

Includes test-only dependencies in the scan. Useful for catching CVEs in tools like `testify` that might handle untrusted input in tests (security exposure is lower, but still).

### Offline mode

govulncheck downloads the vuln DB on each run by default. For air-gapped:

```bash
$ GOVULNDB=file:///path/to/mirror govulncheck ./...
```

Mirror the DB with:

```bash
$ git clone https://github.com/golang/vulndb /opt/vulndb
$ # serve via simple HTTP or use as filesystem path
```

### Performance

```bash
$ time govulncheck ./...
real    0m12.345s
user    0m20.000s
sys     0m1.500s
```

Time scales with dep graph size; large projects ~30 s – 2 min. Subsequent runs benefit from the build cache.

### Comparison with other scanners

| Tool                | Approach                           | Noise               |
|---------------------|------------------------------------|---------------------|
| `govulncheck`       | Call-graph + curated DB            | Low                  |
| `snyk` / `dependabot`| Dep-version vs. CVE feed          | Higher (no reachability) |
| `trivy` (with Go)   | Same as snyk; uses NVD feed        | Higher              |
| `osv-scanner`       | Dep-version vs. OSV (multi-language)| Higher              |

`govulncheck` is the only one with symbol-level reachability for Go. Use it for definitive "do I need to act?"; use generic tools for breadth across languages.

### Limitations

- **Reflection** (`reflect.Value.Call`) is a hole — calls invoked through reflection aren't tracked. govulncheck reports a `caller unknown` finding in that case.
- **`go:linkname`** can bypass the call graph; govulncheck may miss vulns reached via linkname.
- **CGO** boundaries: calls into C aren't analyzed.
- **Plugins** (`plugin.Open`): the loaded code isn't statically known.
- Symbol-level precision relies on accurate DB metadata; missing symbols in a DB entry fall back to module-level.

## Standard Library Hooks

- `golang.org/x/vuln/scan` — programmatic API.
- `golang.org/x/vuln/internal/govulncheck` — internal logic.
- `runtime/debug.ReadBuildInfo()` — what binary-mode scans read.
- `debug/buildinfo` — read build info from a file.
- `golang.org/x/tools/go/packages` — package loading.
- `golang.org/x/tools/go/callgraph` — call-graph construction.

## Real-World Patterns

### 1. CI gate

```yaml
- uses: golang/govulncheck-action@v1
```

Fails the build on any reachable vuln.

### 2. Scheduled scan

```yaml
on:
  schedule:
    - cron: '0 4 * * *'           # daily

jobs:
  vuln:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.26' }
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - run: govulncheck ./...
```

### 3. Scan released binaries

```bash
$ govulncheck -mode=binary /usr/local/bin/myapp
```

Useful for vendor binaries you don't have source for.

### 4. Pre-commit

```bash
#!/bin/sh
govulncheck ./... || (echo "fix vulns first"; exit 1)
```

### 5. JSON ingestion to a security dashboard

```bash
$ govulncheck -json ./... > scan.json
$ # convert to SARIF for GitHub code-scanning
$ jq -f to-sarif.jq scan.json > scan.sarif
```

### 6. Quick upgrade after a finding

```bash
$ govulncheck ./...
# ... reports CVE in dep X@v1.2.0; fixed in v1.2.3
$ go get dep@v1.2.3
$ go mod tidy
$ govulncheck ./...
no vulnerabilities found
```

### 7. Stdlib vuln → toolchain upgrade

```bash
$ govulncheck ./...
# reports net/http issue in go1.25.0
$ go mod edit -toolchain=go1.25.3
$ GOTOOLCHAIN=auto go build ./...
```

## Anti-Patterns & Gotchas

**Treating module-level scans as authoritative.** Without symbol info, you'll either ignore everything (noise) or upgrade for nothing. Use the default symbol mode.

**Running with `-mode=binary` on a built binary that was stripped.** Binary mode reads `buildinfo`, not symbols; it still works but may have less precision. Make sure `-buildvcs=auto` is on for traceability.

**Reflection-heavy code with `caller unknown` findings.** A `caller unknown` finding is uncertain — could be real, could be unreachable. Investigate manually.

**Ignoring stdlib CVEs because "I'll upgrade Go later".** Stdlib vulnerabilities are often network-facing (HTTP, TLS). Upgrade promptly; the toolchain mechanism (1.21+) makes it cheap.

**Pinning to old `govulncheck` versions.** The tool's accuracy improves over time. Pin DB freshness, not tool version.

**Running govulncheck only at release.** A CVE landing the day after release goes unnoticed for weeks. Daily scheduled scan or pre-merge gating.

**Using govulncheck as your only security tool.** It covers known CVEs in Go modules. Doesn't cover: secrets in code, SQL injection patterns, container vulns, transitively imported non-Go components. Layer with `gosec`, `trivy`, secret scanners.

**Disabling govulncheck because of a false positive.** File an issue at golang/vulndb. False positives are tracked and fixed.

**Not pinning the action version (`@v1` vs. `@v1.0.4`).** Pin the action; bumps can change behavior.

**Treating "found but not reachable" as "safe forever".** A future code change may make it reachable. Re-scan on every CI run.

**Scanning a project with stale `go.sum`.** govulncheck loads via `go/packages`; needs `go.sum` to verify. Run `go mod tidy` first.

## Performance Notes

- Source-mode scan: 30 s – 2 min for medium projects (~100 deps).
- Binary-mode scan: <5 s typically.
- Vuln DB download: ~5 MB, ~2 s.
- Memory: 200 MB – 1 GB depending on dep graph.

Cache the DB and `$GOMODCACHE` in CI for fast repeat scans.

## How Big Companies Use It

- **Google** uses govulncheck across Google's Go services as the primary supply-chain scanner: https://go.dev/security/vuln.
- **GitHub** integrates govulncheck into security advisories: https://github.blog/2022-09-06-govulncheck-is-now-available.
- **Kubernetes** runs govulncheck nightly; findings become issues filed against `kubernetes/kubernetes`: https://github.com/kubernetes/kubernetes.
- **Cloudflare** uses govulncheck as a release gate for their Go services: https://blog.cloudflare.com.
- **HashiCorp** uses govulncheck plus `trivy` (for OS-level CVEs in Docker images): https://github.com/hashicorp/terraform.
- **CockroachDB** runs govulncheck as part of nightly CI: https://github.com/cockroachdb/cockroach.
- **Tailscale** uses govulncheck plus an internal review for any `caller unknown` findings: https://tailscale.com/blog.

## Source Code References

`govulncheck` lives at `golang.org/x/vuln`.

- Source: [`golang.org/x/vuln`](https://github.com/golang/vuln).
- `govulncheck` cmd: [`cmd/govulncheck`](https://github.com/golang/vuln/tree/master/cmd/govulncheck).
- Scanner: [`scan`](https://github.com/golang/vuln/tree/master/scan).
- Internal call graph: [`internal/scan`](https://github.com/golang/vuln/tree/master/internal/scan).
- Vuln DB: [`golang/vulndb`](https://github.com/golang/vulndb).
- Web frontend (vuln.go.dev): [`golang/pkgsite-metrics`](https://github.com/golang/pkgsite-metrics).
- gopls integration: [`gopls/internal/vulncheck`](https://github.com/golang/tools/tree/master/gopls/internal/vulncheck).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "govulncheck reference": https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck.
- "Vulnerability Management for Go" (Julie Qiu): https://go.dev/blog/vuln.
- "Go vulnerability database": https://vuln.go.dev.
- "OSV format": https://ossf.github.io/osv-schema.
- "govulncheck design doc" (Roger Peppe): https://go.googlesource.com/proposal/+/master/design/57161-govulncheck.md.
- "Symbol-level vulnerability scanning" (Filippo Valsorda): https://words.filippo.io/govulncheck.

## Exercises / Self-Check

1. Run `govulncheck ./...` on a project. Are findings reachable from your code? Investigate one trace.
2. Use `-mode=binary` on a built binary. Compare findings to source mode.
3. Set up a GitHub Actions nightly govulncheck. How does it differ from a per-PR gate?
4. Find a stdlib CVE in `vuln.go.dev`. What Go version fixed it? How would you upgrade in `go.mod`?
5. Cause a `caller unknown` finding by adding a reflection call to a vulnerable symbol. How would you triage it?
