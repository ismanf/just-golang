# govulncheck in CI — Vulnerability Scanning for Go

## TL;DR

`govulncheck` is the Go team's official vulnerability scanner. It reads your module graph and your compiled call graph, then queries the **Go vulnerability database** (`vuln.go.dev`) to report only vulnerabilities your code *actually reaches* — not every CVE in your `go.sum`. That call-graph filter is the killer feature: it routinely drops noise by 80–95% compared to SCA tools that match by package version alone. The standard CI invocation is `govulncheck ./...`; it exits non-zero on findings, prints stable text by default, and emits JSON with `-format=json` for parsing. Go 1.26+ extends it with **binary scanning** (`govulncheck -mode=binary ./mybin`), **OSV-format output** (`-format=openvex` for VEX statements), and an integrated mode in `go test` and `go vet` for IDE wiring. The single biggest gotcha: `govulncheck` analyses Go code only — your CGo, your container base image, and your JavaScript bundle are invisible to it.

## Mental Model

```
                  Your project              govulncheck                Go vuln DB
                  ───────────               ───────────                ──────────
   go.mod ────┐                      ┌──► query for affected
              ├──► module list ──────┤    versions, by module
   go.sum ────┘                      │
                                     ├──► call-graph build
   *.go ─────► SSA call graph ──────┤    (which symbols do you
              (only reachable code)  │     actually call?)
                                     │
                                     └──► intersect ──► report only
                                                          REACHABLE
                                                          vulnerabilities
```

Two key invariants:

1. **Module match alone is not enough.** A package can be vulnerable in version X but the vulnerable symbol is never called by your binary. `govulncheck` will still *list* it (at a lower severity) but distinguish "called" from "imported but not called."
2. **Static analysis ≠ runtime guarantees.** Reflection, `go:linkname`, plugins, and CGo edges defeat reachability. Vulnerabilities behind those edges may be silently missed.

## Installation and Basic Usage

```bash
# Install (Go 1.18+, but track the latest)
go install golang.org/x/vuln/cmd/govulncheck@latest

# Run from your module root
govulncheck ./...

# Specific package
govulncheck ./cmd/server

# Scan a compiled binary (Go 1.26+ formalised; available since 1.21)
govulncheck -mode=binary ./bin/server

# JSON output for CI parsing
govulncheck -format=json ./... > vulns.json

# OpenVEX output (1.26+) — produces machine-readable VEX statements
govulncheck -format=openvex ./... > vex.json

# Show all imports, not just reachable ones
govulncheck -show=verbose ./...
```

### Reading the human output

```
Vulnerability #1: GO-2024-2598
    HTTP/2 stream reset DoS in golang.org/x/net/http2
  More info: https://pkg.go.dev/vuln/GO-2024-2598
  Module: golang.org/x/net
    Found in: golang.org/x/net@v0.20.0
    Fixed in: golang.org/x/net@v0.23.0
    Example traces found:
      #1: server/handler.go:42:18: handler.ServeHTTP calls http2.Server.ServeConn
```

The "Example traces" block is the reachability proof. If it's absent under "Your code is affected by 0 vulnerabilities," the issue is informational — present in your module graph but not invoked.

## Exit Codes (Critical for CI)

`govulncheck` exit codes are deterministic and CI-friendly:

| Exit | Meaning |
|------|---------|
| `0`  | No reachable vulnerabilities found |
| `1`  | Tooling error (bad args, build failure) |
| `2`  | (Reserved) |
| `3`  | At least one vulnerability is **affecting** your code |

Older versions used `0`/`1` only — exit `3` was added so pipelines can distinguish "scan ran, found nothing" from "scan ran, found bugs" from "scan itself broke." Always check for exit `3`, not just non-zero:

```bash
govulncheck ./...
case $? in
  0) echo "clean"; exit 0 ;;
  3) echo "vulns!"; exit 1 ;;
  *) echo "scanner broken"; exit 1 ;;
esac
```

## CI Integration — GitHub Actions

```yaml
# .github/workflows/govulncheck.yml
name: govulncheck
on:
  push: { branches: [main] }
  pull_request:
  schedule: [{ cron: "0 6 * * *" }]   # daily, to catch new CVEs

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.26.x'
          check-latest: true
      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - name: Scan
        run: govulncheck -format=json ./... | tee vulns.json
      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: vulns.sarif }
```

The **scheduled daily run** is essential: a clean main-branch scan today does not mean tomorrow's database doesn't list a new CVE for code you haven't touched. CI on PRs catches new code; cron catches new disclosures.

### GitLab CI

```yaml
govulncheck:
  image: golang:1.26
  script:
    - go install golang.org/x/vuln/cmd/govulncheck@latest
    - govulncheck -format=json ./... > vulns.json
  artifacts:
    when: always
    reports:
      # Convert JSON to GitLab SAST report if desired
      sast: vulns.json
  allow_failure: false
```

## JSON Output Schema

The JSON output is a stream of newline-delimited messages, each with a `"config"`, `"progress"`, `"sbom"`, `"osv"`, or `"finding"` key. The relevant one is `"finding"`:

```json
{"finding":{
  "osv":"GO-2024-2598",
  "fixed_version":"v0.23.0",
  "trace":[
    {"module":"my-app","package":"main","function":"main"},
    {"module":"golang.org/x/net","package":"http2",
     "function":"Server.ServeConn","version":"v0.20.0"}
  ]
}}
```

Trace depth correlates with severity — a finding with `trace` length 1 means the vuln symbol is reachable from your entrypoint; length 0 means imported-but-not-called (advisory).

### Parsing with jq

```bash
# All reachable findings
govulncheck -format=json ./... | \
  jq -r 'select(.finding != null and (.finding.trace | length) > 0) | .finding.osv'

# Findings grouped by module
govulncheck -format=json ./... | \
  jq -s 'map(select(.finding)) | group_by(.finding.trace[-1].module)'
```

## Binary Mode — Scanning What You Ship

```bash
# Build with full symbol info
go build -trimpath -o bin/server ./cmd/server
govulncheck -mode=binary bin/server
```

Binary mode is invaluable for:

- **CI/CD post-build scanning** — confirm the artifact you're about to push to prod has no reachable vulns.
- **Third-party binaries** — you don't have the source, but you have the binary.
- **Container image scans** — pair with `cosign attest` to attach the report as an attestation.

Caveats:

- Stripped binaries (`-ldflags="-s -w"`) lose symbol info → less precise reachability.
- Plugin systems (`plugin.Open`) hide cross-module edges.
- Binary mode is *call-graph approximated* — it sees the symbol table but cannot always resolve dynamic dispatch through interfaces; it errs on the side of "called" (false positive over false negative).

## Go 1.26+ Enhancements

### Integrated `go test -vuln`

```bash
# 1.26+: vulnerability scan piggybacks on go test
go test -vuln ./...
```

When a test depends on a package with a vulnerable symbol that the test code reaches, the test fails with a vulnerability error in addition to its usual pass/fail.

### `go vet -vettool` chaining

```bash
go vet -vettool=$(which govulncheck) ./...
```

Embeds govulncheck as a vet check. IDE plugins that surface `go vet` results (gopls) thus highlight vulns inline.

### OpenVEX output

VEX (**Vulnerability Exploitability eXchange**) is an OASIS standard for stating "vuln X is in component Y but is not exploitable because Z." Govulncheck 1.26+ can emit a VEX statement for every imported-but-not-reachable vuln, suitable for attestation to your SBOM. Pair with `cosign attest --type vex`.

### Tracking only first-party reachability

```bash
govulncheck -test ./...           # include test code in graph
govulncheck -tags=integration ./...  # build tags
```

## Anti-Patterns & Gotchas

**Treating non-reachable vulns as "no risk."** They become reachable the moment you `import` a new helper. Fix them — just don't gate releases on them.

**Pinning govulncheck to an old release.** The Go vuln DB updates daily; an old binary can still query it, but its bug fixes (call-graph accuracy) regress with age.

**Running only on PR.** New CVEs land continuously. Always pair with a daily cron.

**Ignoring CGo.** `govulncheck` ignores `import "C"` code. Pair with `trivy` or `grype` for native deps.

**Ignoring base images.** Your `FROM ubuntu:22.04` has CVEs `govulncheck` cannot see. Layer in `trivy fs` or `grype` for container layers.

**Allow-listing by CVE ID without justification.** If you exempt `GO-2024-XXXX`, write the exemption with the reachability proof or VEX justification inline in your config.

**Building without `-trimpath` and then scanning the binary.** Stripped/symbol-poor binaries downgrade reachability quality. Always `-trimpath` + keep symbol table.

**Mixing `replace` directives that pin upstream to forks.** `govulncheck` queries the *original* module path; if your fork patched a vuln but the path is unchanged, govulncheck will still report it. Either rename the fork's module path or use a `retract` + version bump.

**Running only `govulncheck` on the root package.** `./...` is correct — you want the full module graph traversed.

**Forgetting cgo/plugins/reflection caveats.** Reachability is *static*. Anything dynamic is a blind spot.

**Failing CI on "no fix available" vulns without a process.** If there's no fix yet, you need a documented mitigation (firewall, WAF rule, code workaround). Don't just silence the alert.

## Performance Notes

- **Cold scan, medium repo (~200 deps):** ~10–30s.
- **Warm scan (with `GOCACHE` + `GOMODCACHE` populated):** ~2–5s.
- **Binary scan:** ~5–15s for a 50 MB binary.
- **DB fetch:** ~3 MB compressed, cached in `$GOPATH/pkg/mod/cache/download/sumdb/sum.golang.org` plus `~/.cache/govulncheck/`.
- **CI tip:** cache `~/go/pkg/mod` and `~/.cache/govulncheck` between runs — drops scan time by ~70%.

## How Big Companies Use It

- **Google** uses `govulncheck` internally as part of their Go monorepo CI; results feed into vuln dashboards.
- **GitHub** integrates `govulncheck` findings into the Dependabot security tab via the OSV format.
- **Sourcegraph** ships `govulncheck` results in their batch-change UI for fleet-wide vulnerability triage.
- **Cloudflare** runs `govulncheck` on every binary built in their internal Bazel-based Go pipeline, with binary-mode scans on the artifact post-build.
- **Tailscale** publishes their `govulncheck` results as part of release notes ("addresses GO-2024-XXXX").
- **HashiCorp** (Vault, Consul, Terraform) integrates govulncheck into their goreleaser pipeline with the `-format=openvex` output attached as a Sigstore attestation.

## Source Code References

- `govulncheck` source: https://github.com/golang/vuln (the entire `x/vuln` repository).
- Go vulnerability DB: https://vuln.go.dev — OSV-formatted JSON feed.
- DB ingestion pipeline: https://github.com/golang/vulndb — how reports become entries.
- Reachability core: `golang.org/x/vuln/internal/vulncheck` — SSA call-graph traversal.
- Integration tests: `golang.org/x/vuln/internal/test`.

## Further Reading

- Go team announcement: https://go.dev/blog/govulncheck.
- "Vulnerability Management for Go" (Filippo Valsorda, Julie Qiu): https://go.dev/blog/vuln.
- OSV-Schema specification: https://ossf.github.io/osv-schema/.
- VEX OASIS spec: https://github.com/openvex/spec.
- Russ Cox, "Surviving Software Dependencies" (2019): https://research.swtch.com/deps.
- Filippo Valsorda, "The Go Vulnerability Database": https://blog.filippo.io/.
- SLSA framework: https://slsa.dev — pair with govulncheck for full supply-chain attestation.

## Exercises / Self-Check

1. Create a Go module that imports `golang.org/x/net v0.20.0` but never calls into `http2`. Run `govulncheck ./...`. Does GO-2024-2598 appear, and is it marked as reachable?
2. Build a binary, strip symbols with `-ldflags="-s -w"`, and run `govulncheck -mode=binary`. Compare against an un-stripped build. Where does reachability degrade?
3. Write a GitHub Actions workflow that fails only on `exit 3`, treats `exit 0` as success, and posts a PR comment with the JSON-parsed list of OSV IDs.
4. Use `-format=openvex` to produce a VEX statement for a non-reachable vuln. Attach it to a container image via `cosign attest`.
5. Add a `replace` directive pinning a forked dependency. Confirm `govulncheck` still queries the original module path — what's the implication for forks of vulnerable packages?
6. Add reflection-based dispatch (`reflect.ValueOf(x).MethodByName(...)`) in a way that calls a vulnerable function. Does `govulncheck` detect it? Why or why not?
7. Run `govulncheck -test ./...` vs without `-test`. Identify a vuln that is reachable only from test code. Should that block a release?
