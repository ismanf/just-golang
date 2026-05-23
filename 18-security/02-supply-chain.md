# Supply Chain — Sigstore, SLSA, Module Checksum DB

## TL;DR

Go has had a first-class supply-chain story since 2019 — earlier than almost any other language ecosystem. Three load-bearing pieces: (1) **`go.sum` + the Go checksum database (`sum.golang.org`)** verifies that the bytes you download for module `M@v` match what every other Go user downloaded for `M@v` (a transparency log enforces immutability — even Google cannot quietly republish a different `M@v` without leaving a public, auditable trace); (2) **`GOPROXY`** decouples you from upstream version-control availability and gives you a place to enforce policy (allow-list, vuln-scan, mirror); (3) **the Go module proxy/sumdb pair** make Go's package install *deterministic and verifiable* — the cornerstone of SLSA. On top of that, **Sigstore** (`cosign`, `rekor`) and **SLSA** (Supply-chain Levels for Software Artifacts) build the *attestation* layer: cryptographic statements that "this binary was built from this commit by this builder." In Go 1.26+, the official Go toolchain itself is distributed as a Sigstore-signed, Rekor-logged artifact, and the recommended release flow attaches build provenance (SLSA L3) to every binary you publish.

## Mental Model

```
                ┌────────────────────────────────┐
   Developer ───┤  go mod tidy / go build         │
                │  ─ fetches modules via GOPROXY  │
                │  ─ verifies sums via GOSUMDB    │
                │  ─ records sums in go.sum       │
                └────────────────────────────────┘
                          ▲
                          │ TUF-like trust
                          ▼
            ┌────────────────────────────────┐
            │   GOPROXY (proxy.golang.org    │
            │     or your internal mirror)   │
            └────────────────────────────────┘
                          ▲
                          │ origin pull (once,
                          │ then cached forever)
                          ▼
            ┌────────────────────────────────┐
            │   Origin VCS (GitHub, GitLab…) │
            └────────────────────────────────┘

   GOSUMDB (sum.golang.org)
   = transparency log of (module@version → hash) tuples
   = append-only, Merkle-tree backed
   = your go.sum is a *partial witness* of this log
```

The mental key: **`go.sum` is a tiny audit log** that says "when *I* downloaded `M@v`, it hashed to X." `sum.golang.org` is the global, append-only audit log. As long as both agree, no one — not your registry mirror, not Google, not your CI — can swap dependency bytes on you without detection.

## The Go Checksum Database

`sum.golang.org` is a Go-team-operated **transparency log** (Trillian-backed) of pairs:

```
M@v  h1:base64-of-sha256-of-tree
```

Every fetch of a new module version verifies inclusion in this log. The log is **append-only**: an entry, once written, cannot be modified or deleted. If Google attempted to silently change `golang.org/x/net@v0.20.0`'s hash, every Go user worldwide would observe the change (because the log root would differ from cached witnesses), and the attack would be globally and publicly recorded.

### Environment variables

| Variable | Purpose |
|----------|---------|
| `GOPROXY` | Where to fetch modules. Default: `proxy.golang.org,direct`. Comma-separated fallback chain. |
| `GOSUMDB` | Which checksum DB to verify against. Default: `sum.golang.org`. Set to `off` to disable. |
| `GOPRIVATE` | Glob list of module paths considered private — bypass both proxy and sumdb. |
| `GONOPROXY` | Modules bypassing the proxy (still hit sumdb if GOSUMDB on). |
| `GONOSUMCHECK` | Removed in Go 1.21+. Use `GOSUMDB=off` or `GONOSUMDB`. |
| `GONOSUMDB` | Modules bypassing the sumdb. |
| `GOFLAGS` | Carry flags like `-mod=readonly` into all `go` invocations. |

### Recommended production settings

```bash
# Public open-source project
export GOPROXY="https://proxy.golang.org,direct"
export GOSUMDB="sum.golang.org"
export GOFLAGS="-mod=readonly"

# Enterprise with internal modules
export GOPRIVATE="*.internal.example.com,github.com/example/private-*"
export GOPROXY="https://athens.internal.example.com,direct"
export GOSUMDB="sum.golang.org"  # still verify public modules
```

`-mod=readonly` is critical in CI: it makes `go build` refuse to touch `go.mod`/`go.sum`, forcing you to run `go mod tidy` explicitly and review the diff.

## Vendoring vs Proxy

Two strategies for guaranteed-reproducible builds:

**Vendoring (`go mod vendor`)**
- All dependencies copied into `./vendor/`.
- `go build` uses `./vendor` automatically (or with `-mod=vendor`).
- Pros: zero network at build time; complete control; trivial audit (`git diff vendor/`).
- Cons: large repo; PRs become massive; can hide upstream changes in noise.

**Proxy + go.sum**
- No vendor directory; rely on GOPROXY + GOSUMDB.
- Reproducibility from `go.mod` + `go.sum`.
- Pros: small repo; clear dependency review.
- Cons: needs reliable proxy; PR review of `go.sum` diff is harder than `vendor/` diff.

A common pattern in regulated industries: **vendor + commit `vendor/` + still keep `go.sum`**. Build is fully offline; reviewer can see the bytes; sumdb still audited at vendor-refresh time.

## SLSA — Supply-chain Levels for Software Artifacts

[SLSA](https://slsa.dev) is a framework from the Open Source Security Foundation defining four levels of build integrity:

| Level | What it gives you |
|-------|-------------------|
| **L1** | Build process is documented; provenance is generated (may be unsigned). |
| **L2** | Provenance is signed; build is run on a hosted service. |
| **L3** | Build is *hermetic* (no network at build time except declared inputs); builder is *isolated* (no admin access; ephemeral). Provenance is verifiable end-to-end. |
| **L4** | Two-party review; hermetic + reproducible builds. |

Go aligns naturally with L3 because `go build -mod=vendor` (or `-mod=readonly` with cached modules) is hermetic, and the toolchain itself is deterministic given the same inputs.

### Generating SLSA provenance with `slsa-github-generator`

```yaml
# .github/workflows/release.yml
name: release
on:
  push: { tags: ["v*"] }

permissions:
  id-token: write   # for OIDC → Sigstore
  contents: write
  attestations: write

jobs:
  build:
    uses: slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@v2.0.0
    with:
      go-version: '1.26'
      config-file: .slsa-goreleaser.yml
```

The reusable workflow:
1. Runs `go build` inside an isolated GitHub-hosted runner.
2. Captures the build inputs (commit SHA, go.mod, builder image).
3. Signs the artifact + provenance with a short-lived OIDC-issued certificate from Fulcio (the Sigstore CA).
4. Logs the signature to Rekor (the Sigstore transparency log).
5. Attaches the signed provenance as a GitHub release asset.

### Verifying SLSA provenance

```bash
# Install slsa-verifier
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@latest

# Verify
slsa-verifier verify-artifact mybinary \
  --provenance-path mybinary.intoto.jsonl \
  --source-uri github.com/example/myrepo \
  --source-tag v1.2.3
```

This confirms: the binary was built by the named GitHub repo, at the named tag, by the SLSA L3 builder workflow — not by a developer's laptop, not on a forked PR, not with patches.

## Sigstore — Sign Everything, Verify Easily

Sigstore is the umbrella name for **`cosign`** (signing), **Fulcio** (short-lived CA), and **Rekor** (transparency log). Key idea: no long-lived signing keys. You authenticate with OIDC (GitHub, Google, your IdP), Fulcio mints a 10-minute certificate bound to your identity, you sign, and the signature is logged to Rekor.

### Sign a Go binary

```bash
# Build deterministically
GOFLAGS="-trimpath -mod=readonly" go build -ldflags="-s -w" -o myapp ./cmd/myapp

# Sign (keyless — uses your OIDC identity)
cosign sign-blob --yes myapp \
  --output-signature myapp.sig \
  --output-certificate myapp.crt
```

### Verify

```bash
cosign verify-blob myapp \
  --signature myapp.sig \
  --certificate myapp.crt \
  --certificate-identity-regexp '^https://github\.com/example/.+' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

The `--certificate-identity-regexp` is the trust anchor: you accept signatures only from a workflow living under `github.com/example/...`. Without it, anyone with a GitHub account could sign and verify.

### Sign a container image

```bash
# Push image first
docker push ghcr.io/example/myapp:v1.2.3

# Sign by digest (recommended)
DIGEST=$(crane digest ghcr.io/example/myapp:v1.2.3)
cosign sign --yes ghcr.io/example/myapp@$DIGEST

# Verify (in admission controller, CI, etc.)
cosign verify ghcr.io/example/myapp@$DIGEST \
  --certificate-identity-regexp '^https://github\.com/example/.+' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

### Attach attestations (SBOM, VEX, provenance)

```bash
# Generate SBOM
syft ghcr.io/example/myapp:v1.2.3 -o cyclonedx-json > sbom.json

# Attach + sign as an attestation
cosign attest --yes --predicate sbom.json \
  --type cyclonedx \
  ghcr.io/example/myapp@$DIGEST
```

`cosign attest` writes an in-toto statement (predicate + subject + signature) and stores it in the OCI registry as a sibling artifact. Verifiers can pull SBOM, VEX, and provenance for an image with a single `cosign download attestation`.

## The Go Toolchain Itself

Go 1.21+ ships with **toolchain directives** in `go.mod`:

```go
module example.com/myapp

go 1.26
toolchain go1.26.1
```

`go build` will automatically download the requested toolchain version (via the proxy + sumdb path) if the locally installed Go is older. The downloaded toolchain binaries are themselves verified against `sum.golang.org`. Go 1.23+ ships the toolchain Sigstore-signed; 1.26 makes verification of toolchain signatures the default for offline-cached installs.

### Toolchain selection rules

- `toolchain` directive sets the *minimum* — Go will upgrade but never downgrade.
- `GOTOOLCHAIN=local` disables automatic toolchain switch (use the installed version even if older).
- `GOTOOLCHAIN=go1.26.2` pins to an exact version regardless of `go.mod`.
- `GOTOOLCHAIN=auto` (default) honours `go.mod`'s `toolchain` directive.

## Module Mirror & Internal Proxy

For enterprise environments, run an internal proxy (Athens, JFrog, Sonatype Nexus, Artifactory) to:

1. Cache public modules (so a GitHub outage doesn't break your build).
2. Enforce policy: vuln-scan modules at ingest; deny by license; deny by retracted version.
3. Audit: log every fetch by who, when, what.

### Minimal Athens deployment

```yaml
# docker-compose.yml
services:
  athens:
    image: gomods/athens:latest
    environment:
      ATHENS_DISK_STORAGE_ROOT: /var/lib/athens
      ATHENS_STORAGE_TYPE: disk
      ATHENS_GOPROXY: "https://proxy.golang.org,direct"
    ports: ["3000:3000"]
    volumes: [athens-data:/var/lib/athens]
volumes: { athens-data: {} }
```

### Developer config

```bash
export GOPROXY="https://athens.example.com,direct"
export GOSUMDB="sum.golang.org"   # still verify against the public log
```

The `direct` fallback is crucial — if Athens is down, builds still succeed by pulling from origin. Drop `direct` if you want hard policy enforcement.

## go.sum Diff Review

Treat every `go.sum` change like a critical config change:

```bash
# After go mod tidy:
git diff -- go.mod go.sum
```

Look for:

- **New modules** you didn't intend to add → indirect dep from a new package; check what brought it in (`go mod why -m <module>`).
- **Major version bumps** → re-read the upstream changelog; assume API + security changes.
- **Replaced or excluded modules** → confirm intentional.
- **Modules without sumdb entries** (`h1:` line missing) → indicates `GOSUMDB=off` was used somewhere; reverting.

## Anti-Patterns & Gotchas

**`GOSUMDB=off` "to fix" a CI failure.** That disables the global tamper-evidence guarantee. The fix is to investigate the mismatch — it usually means a force-push to a tag or a mirror returning bad bytes.

**Editing `go.sum` by hand.** Never. Always re-run `go mod tidy`. Hand edits leave you unable to fetch the modules they describe.

**Letting `go.mod` and `go.sum` drift on PR branches.** Run `go mod tidy` + `git status -s go.mod go.sum` as a CI step; fail if dirty.

**Using `replace` to point at a fork without renaming the module path.** The sumdb still queries the original path; if you replace with a different SHA, you're outside the audit trail. Better: vendor the fork under your own module path.

**Long-lived signing keys.** Sigstore's keyless mode avoids them. Don't put a Cosign private key in your CI secrets unless you have HSM-backed key custody and an emergency revocation plan.

**Trusting `cosign verify` without `--certificate-identity-regexp`.** Without an identity assertion, any Sigstore-signed artifact passes. The whole point is binding to *who* signed it.

**Conflating SBOM, VEX, and provenance.** SBOM = "what's inside." VEX = "this CVE doesn't apply to me." Provenance = "this is how I was built." You need all three.

**Vendoring without verifying.** `go mod vendor` doesn't re-verify the sumdb. Run `go mod verify` first if you've changed dependencies; otherwise you might vendor compromised bytes.

**Allowing arbitrary `GOPROXY` overrides from PRs.** Treat `GOPROXY` like `PATH` — only set it from trusted CI config, never from a PR's workflow file.

**Forgetting to retract bad versions.** When you publish `v1.2.3` and discover it's broken, add a `retract v1.2.3` line in the next release's `go.mod`. Tools surface retractions to users.

**Container images signed but built non-reproducibly.** Provenance attests "I was built here"; reproducibility attests "anyone can rebuild me and get the same bytes." Without the latter, signature only proves *that* builder ran, not *what* it ran.

## Performance Notes

- **Module fetch via proxy:** ~20–200ms per module (vs seconds via direct VCS).
- **`go mod verify`** on a project with ~300 deps: ~1s.
- **Cosign sign-blob:** ~1s (mostly OIDC roundtrip + Rekor log).
- **Cosign verify:** ~500ms (Rekor query + cert chain check).
- **SLSA L3 build overhead** (GitHub-hosted): typically 1–3 minutes added vs unsigned build.

## How Big Companies Use It

- **Google** uses an internal-only Go proxy + sumdb; every internal Go module is verified through a private transparency log analogous to `sum.golang.org`.
- **Kubernetes** publishes signed binaries and container images via `cosign`, with SLSA L3 provenance generated by `slsa-github-generator`. Verifiable via `sigstore.dev`.
- **HashiCorp** (Vault, Terraform, Consul, Nomad) signs all release artifacts with cosign + GPG and publishes SLSA L3 provenance.
- **Chainguard** builds the entire `wolfi-os` and chainguard-images catalog with SLSA L3 + cosign + SBOM by default.
- **Sigstore itself** is largely Go. The `cosign`, `rekor`, and `fulcio` daemons are reference implementations of the model they enforce.
- **GitHub** runs Artifact Attestations (built on Sigstore) generally available for Go releases since 2024.
- **Tailscale** publishes a per-release provenance attestation alongside every binary; reproducible builds are a stated goal of their release process.
- **Cloudflare** mirrors a curated subset of `proxy.golang.org` internally + runs sumdb verification at every artifact boundary.

## Source Code References

- Go module proxy reference impl: https://github.com/golang/go/blob/master/src/cmd/go/internal/modfetch/.
- `sum.golang.org` server source: https://github.com/golang/sumdb.
- Athens proxy: https://github.com/gomods/athens.
- Sigstore cosign: https://github.com/sigstore/cosign.
- Fulcio CA: https://github.com/sigstore/fulcio.
- Rekor transparency log: https://github.com/sigstore/rekor.
- SLSA framework: https://github.com/slsa-framework/slsa.
- SLSA GitHub generator (Go builder): https://github.com/slsa-framework/slsa-github-generator/tree/main/internal/builders/go.

## Further Reading

- "Securing the Software Supply Chain: Recommended Practices for Developers" (NSA/CISA, 2023): https://media.defense.gov/2023/Sep/12/.
- Filippo Valsorda, "Transparent logs for skeptical clients": https://research.swtch.com/tlog.
- Russ Cox, "Securing software by design" (2020): https://research.swtch.com/.
- Sigstore "How does Sigstore work?": https://docs.sigstore.dev/.
- SLSA specification v1.0: https://slsa.dev/spec/v1.0/.
- Filippo Valsorda, "The Go Checksum Database": https://go.dev/blog/checksum-database.
- "in-toto Attestation Framework": https://github.com/in-toto/attestation.
- CNCF "Software Supply Chain Best Practices" (2021): https://github.com/cncf/tag-security/.

## Exercises / Self-Check

1. Inspect your project's `go.sum`. Pick one module and verify its hash matches `sum.golang.org` by fetching `https://sum.golang.org/lookup/<module>@<version>` directly.
2. Set up an Athens proxy in Docker. Configure your Go to use it. Confirm modules fetched once are served offline thereafter.
3. Sign a release binary with `cosign sign-blob` keyless. Verify it with `--certificate-identity-regexp`. Then verify without the regexp — what does the trust model degrade to?
4. Trigger a SLSA L3 build via `slsa-github-generator` for a tagged release. Run `slsa-verifier` against the artifact. Modify the artifact by one byte and re-verify — what happens?
5. Add a `replace` directive that points at a fork. Run `go mod verify`. Run `govulncheck`. Identify what changes about your audit trail.
6. Set `GOSUMDB=off`, run `go mod tidy`, and observe the absence of `h1:` lines in `go.sum`. Re-enable the sumdb and re-tidy — what changes?
7. Write an admission policy (Kyverno or Gatekeeper) that admits only container images verified by `cosign verify` against a specific OIDC identity. Test it by deploying both signed and unsigned images.
