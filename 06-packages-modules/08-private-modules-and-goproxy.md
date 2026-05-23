# Private Modules and GOPROXY — GOPRIVATE, GONOSUMDB, Athens, JFrog

## TL;DR

Public modules flow through `proxy.golang.org` (mirror) and `sum.golang.org` (checksum DB). For **private code**, both are bypassed via the **`GOPRIVATE`** env var — a comma-separated list of glob patterns matching private module paths. Direct VCS access requires authentication (SSH key, HTTPS netrc/PAT). For organizations with significant private code and supply-chain requirements, run an **internal proxy**: **Athens** (open source), **JFrog Artifactory**, **Sonatype Nexus**, or **GitLab's package registry**. The single biggest gotcha: **`GOPRIVATE` does NOT make modules private** — it only tells the Go tool to skip the public proxy/sumdb. The actual access control comes from your VCS (private repos + auth). Misconfigured, you can leak module paths to `proxy.golang.org`, which logs them.

## Mental Model

```
   Public module fetch:
       go get github.com/x/y@v1.2.3
           ↓
       GOPROXY=https://proxy.golang.org,direct
           ↓
       proxy returns module zip + .info + .mod
           ↓
       GOSUMDB=sum.golang.org verifies hash
           ↓
       cache + build

   Private module fetch:
       go get my.corp/internal/foo@v1.0.0
           ↓
       GOPRIVATE=*.my.corp,my.corp/*
       (matches → skip proxy + sumdb)
           ↓
       direct VCS: git clone https://my.corp/internal/foo.git
       (auth via netrc, SSH key, or GIT_ASKPASS)
           ↓
       cache + build

   Internal proxy:
       go get my.corp/internal/foo@v1.0.0
           ↓
       GOPROXY=https://athens.my.corp,direct
       GOPRIVATE='' (let athens handle everything)
           ↓
       athens forwards to VCS, caches result
           ↓
       cache + build
```

The right configuration depends on scale: small teams can use `GOPRIVATE` + direct VCS; large orgs run an internal proxy.

## Syntax & Basic Usage

```bash
# Tell Go that anything under *.corp.com is private
$ export GOPRIVATE='*.corp.com,corp.com/*'
$ go env -w GOPRIVATE='*.corp.com,corp.com/*'  # persistent

# Verify
$ go env GOPRIVATE
*.corp.com,corp.com/*

# Fetch a private module (auth must be configured for VCS)
$ go get corp.com/internal/lib@v1.0.0
```

Configure HTTPS auth for private GitHub:

```bash
$ cat ~/.netrc
machine github.com login me password ghp_xxxxxxxxxxxxxxx
```

Or via git config:

```bash
$ git config --global url."https://me:ghp_xxx@github.com/".insteadOf "https://github.com/"
```

## Deep Dive

### What `GOPRIVATE` does

```
GOPRIVATE=*.corp.example.com,internal.lan/*,github.com/myorg/*
```

For any module path matching a glob in `GOPRIVATE`:
1. **GOPROXY is skipped** — the tool goes direct to VCS (or to a proxy that explicitly handles privates).
2. **GOSUMDB is skipped** — no public checksum verification.
3. **`go.sum`** is still maintained locally and on disk.

Globs use shell-style matching (`*`, `?`, `[...]`); `*.corp.com` matches any subdomain of `corp.com`.

`GOPRIVATE` is shorthand for setting:

```
GONOSUMCHECK=  (deprecated)
GONOSUMDB=*.corp.example.com,...     # same paths, skip sumdb
GONOPROXY=*.corp.example.com,...     # same paths, skip proxy
```

You can set the more-specific knobs individually if you want, e.g., go through a proxy but skip the sumdb. Most setups set `GOPRIVATE` and call it a day.

### `GONOSUMDB` vs `GONOSUMCHECK`

Both predate `GOPRIVATE`. `GONOSUMDB` skips the public sumdb for matching paths; `GONOSUMCHECK=*` disables checksum DB entirely (legacy). Use `GOPRIVATE` instead.

### `GOPROXY` syntax

```
GOPROXY=https://proxy.golang.org,direct
GOPROXY=https://proxy.example.com|https://proxy.golang.org,direct
GOPROXY=off
```

Separators:
- `,` (comma): try next on **404 or 410** only. Other errors (network, 5xx) abort.
- `|` (pipe): try next on **any** error. Useful for "this proxy is unreliable; fall back fast".

Special values:
- `direct`: fetch from VCS directly. Always the right "last resort".
- `off`: refuse all module downloads. Useful for vendor-only builds.

### Direct VCS protocols

The Go tool understands several VCS schemes:

```
git+ssh://github.com/me/repo.git
git+https://github.com/me/repo.git
git://github.com/me/repo.git
hg::https://hg.example.com/repo
svn::https://svn.example.com/repo
bzr::https://bzr.example.com/repo
```

For GitHub/GitLab/Bitbucket, recognition is automatic. For other hosts, the Go tool consults `<host>?go-get=1` (vanity discovery).

### Auth methods for `direct`

**SSH** (most common for private repos):

```bash
$ cat ~/.ssh/config
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519

$ git config --global url.git@github.com:.insteadOf https://github.com/
$ go get github.com/myorg/private-repo
```

The `url.<base>.insteadOf` rewrite is the key — Go's "direct" fetcher uses HTTPS, but git rewrites to SSH before connecting.

**HTTPS with token**:

```bash
$ cat ~/.netrc
machine github.com login me password ghp_xxx

$ chmod 600 ~/.netrc
```

Or environment-injection:

```bash
$ export GIT_ASKPASS=/tmp/give-token.sh
$ cat /tmp/give-token.sh
#!/bin/sh
echo "$GIT_TOKEN"
```

**HTTPS with insteadOf**:

```bash
$ git config --global url."https://user:${GIT_TOKEN}@github.com/".insteadOf "https://github.com/"
```

Avoid baking tokens into config files — use CI secrets or environment variables.

### `GONOSUMCHECK` and `go.sum` for private modules

Private modules still generate `go.sum` entries — the Go tool computes the hash locally on first fetch. Subsequent builds verify against this local hash. So `go.sum` is your own integrity record for private code; `GOSUMDB` is the public attestation.

For private code, the *first* successful fetch establishes the truth. Don't blindly trust `go.sum` updates in PRs — review them as you would code.

### Internal proxies — why and when

Internal proxies (Athens, JFrog, Nexus, GitLab Package Registry) solve:

1. **Reliability**: `proxy.golang.org` is highly available but external; some orgs want zero external dependencies during builds.
2. **Caching**: build hosts share a single warm proxy instead of each maintaining its own cache.
3. **Air-gapping**: build hosts can't reach the public internet; proxy fronts a controlled mirror.
4. **Auditing**: every dep fetch is logged centrally.
5. **Filtering**: block specific modules/versions at the proxy (CVE blocklist).
6. **Performance**: faster than the public proxy when both are remote; LAN latency vs. internet.
7. **Private code**: the proxy authenticates against private VCS once; clients don't need individual VCS credentials.

### Athens

[Athens](https://github.com/gomods/athens) is the canonical open-source Go module proxy. Run it as a Docker container:

```yaml
# docker-compose.yml
services:
  athens:
    image: gomods/athens:v0.15.0
    environment:
      ATHENS_STORAGE_TYPE: disk
      ATHENS_DISK_STORAGE_ROOT: /var/lib/athens
      ATHENS_NETRC_PATH: /etc/secrets/netrc  # private VCS auth
    ports:
      - "3000:3000"
    volumes:
      - athens-data:/var/lib/athens
      - ./netrc:/etc/secrets/netrc:ro
```

Client setup:

```bash
$ export GOPROXY=https://athens.my-corp/,direct
$ go get my-corp/private-lib
```

Athens fetches from your private GitHub Enterprise (using `netrc`), caches the result, and serves to all clients. Private code is now proxied — no need for `GOPRIVATE` on the client side.

Storage backends: disk, S3, GCS, Mongo, Azure Blob.

### JFrog Artifactory

JFrog Artifactory has first-class Go module proxy support. Self-hosted or SaaS. Pros: enterprise UI, RBAC, multi-language artifact management. Cons: licensed (paid).

```bash
$ export GOPROXY=https://my-org.jfrog.io/artifactory/go/,direct
$ export GONOSUMDB='*'    # or use GOPRIVATE
```

### Sonatype Nexus

Similar to Artifactory; supports Go via its repository manager. Open-source community edition + paid pro.

### GitLab Package Registry

GitLab can act as a Go proxy for code hosted on the same GitLab instance:

```bash
$ export GOPROXY=https://gitlab.example.com/api/v4/projects/123/packages/go,direct
```

Simpler if you're already on GitLab; not a generic proxy.

### The `GOPROXY=https://...|direct` chain

Common production setup:

```
GOPROXY=https://athens.my-corp.com,https://proxy.golang.org,direct
```

Tries athens first. On 404 (athens doesn't know about this module yet), tries the public proxy. On 404 from public proxy, goes direct to VCS.

Pipe (`|`) is useful when athens flaps; then:

```
GOPROXY=https://athens.my-corp.com|https://proxy.golang.org,direct
```

Athens 5xx falls through to public proxy quickly.

### `GOPROXY=off`

Refuses any module fetch. Useful in CI when you've vendored all deps:

```yaml
- env:
    GOFLAGS: -mod=vendor
    GOPROXY: off
  run: go test ./...
```

If the vendor is incomplete, the build fails immediately (no silent network fetch).

### Replacing the public sumdb

`GOSUMDB=sum.example.com` lets you run your own checksum DB. Almost no one does — the protocol is complex, and `sum.golang.org` is highly trusted. For private modules, simply skip the sumdb via `GOPRIVATE`.

You can also set `GOSUMDB=off` globally, but that disables integrity checking for *all* modules — never do this in production.

### Module hash collision risk

Hash collisions are computationally infeasible. The Go sumdb's value is in preventing *retroactive tampering* — even if an upstream repo's history is rewritten, the sumdb's append-only log catches the divergence.

For private code, your `go.sum` is the local record. If the file is committed and reviewed, you have the same guarantee.

### Auditing dep fetches

Log every fetch via proxy:

```bash
$ GODEBUG=installgoroot=1,fetchretries=0 go get -v github.com/x/y
```

`-v` enables verbose. The proxy logs help reconstruct what was fetched and when.

### Replacing private deps in `go.mod`

```
replace internal.corp.com/old => internal.corp.com/new v1.0.0
```

`replace` works for private modules identically to public. Useful for forks and migrations.

### Vendor for paranoid private builds

For "the world ends if we lose internet" scenarios: vendor and commit. Combined with private VCS as backup. See `06-packages-modules/07-vendoring.md`.

### `GOPRIVATE` examples

```
GOPRIVATE=github.com/myorg/*                     # one GitHub org
GOPRIVATE=*.corp.example.com                     # any subdomain
GOPRIVATE=corp.example.com/internal/*            # specific subtree
GOPRIVATE=github.com/myorg/*,gitlab.example.com/*  # multiple
GOPRIVATE='go.uber.org/myinternal,*.tailscale.io' # mixed
```

Tilde and ${var} expansion are not supported; use absolute literals.

## Standard Library Hooks

- `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB` env vars.
- `GOPROXY`, `GOSUMDB` env vars.
- `GOFLAGS` env var.
- `go env -w KEY=VALUE` — persistent env in `go env`'s config file.
- `go env -u KEY` — unset.
- `go list -m -json all` — see resolved versions.
- `runtime/debug.ReadBuildInfo` — see `GOFLAGS`, `GOPROXY` settings.
- `~/.netrc`, `~/.ssh/config`, `git config url.X.insteadOf` — VCS auth.

## Real-World Patterns

### 1. Small-team config: GOPRIVATE + SSH

```bash
$ go env -w GOPRIVATE='github.com/myorg/*'

$ cat ~/.ssh/config
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519

$ git config --global url.git@github.com:.insteadOf https://github.com/

$ go get github.com/myorg/private-lib@v1.0.0
```

The git config makes Go's "direct" fetcher use SSH instead of HTTPS. SSH key auth is reliable, well-understood, doesn't expire.

### 2. Mid-size org: Athens + GOPRIVATE on the proxy

Run Athens in the corporate network with VCS auth pre-configured. Clients:

```bash
$ go env -w GOPROXY='https://athens.corp.com,https://proxy.golang.org,direct'
$ go env -w GONOSUMDB='*.corp.com'        # if athens doesn't pretend to be sumdb
$ go get corp.com/lib                      # athens fetches and caches
```

Athens does the auth once; clients are simple.

### 3. CI with auth via injected token

```yaml
- name: Configure Go auth
  env:
    GH_TOKEN: ${{ secrets.GH_TOKEN }}
  run: |
    git config --global url."https://x-access-token:${GH_TOKEN}@github.com/".insteadOf "https://github.com/"
    go env -w GOPRIVATE='github.com/myorg/*'
- run: go build ./...
```

`x-access-token` is GitHub's username for token-based HTTPS auth.

### 4. Air-gapped: vendor everything

```bash
# On internet-connected build:
$ go mod tidy
$ go mod vendor
$ git add vendor go.mod go.sum
$ git commit -m "vendor"

# On air-gapped:
$ GOPROXY=off go build -mod=vendor ./...
```

No network. No proxy. No private VCS. Source of truth: committed files.

### 5. CVE blocklist via proxy

Athens (and JFrog) support module filters. Block `github.com/buggy/lib v1.2.3`:

```yaml
# athens-config.yml
filter:
  - !github.com/buggy/lib@v1.2.3
  - +/*
```

Any client trying to fetch the blocked version gets a 410 from athens; falls through to direct (which the corp policy presumably blocks too). Forces an upgrade.

### 6. Inspect what's in your build

```bash
$ go list -m all
github.com/me/proj
github.com/google/uuid v1.6.0
golang.org/x/sync v0.7.0
...

$ govulncheck ./...
```

`govulncheck` cross-references the build list with known CVEs from `vuln.go.dev`. See `09-tooling/20-govulncheck.md`.

## Anti-Patterns & Gotchas

**Setting `GOSUMDB=off` globally to fix a private-module checksum error.** Disables verification everywhere — supply-chain risk. Use `GOPRIVATE` instead.

**Leaking private module paths to the public proxy.** Without `GOPRIVATE`, the Go tool asks `proxy.golang.org` for `my.corp/internal/foo`; the proxy logs the request. Module names can be sensitive.

**Hard-coding HTTPS tokens in `~/.netrc` checked into a repo.** Even private repos. Use CI secrets and inject at build time.

**Using `replace` to point a private dep at a public path "to bypass auth".** It just changes the lookup target; still needs to fetch from somewhere. Set up auth properly.

**Athens with insufficient storage.** Module zips can be GBs across an org's full dep tree. Plan for 10+ GB minimum; monitor.

**Mixing `GOPRIVATE` and a proxy that already handles private modules.** Athens with private VCS auth doesn't need `GOPRIVATE` set by clients (the proxy is the auth boundary). Set `GOPRIVATE` only if your proxy doesn't handle privates.

**Forgetting to set `GOFLAGS` in CI.** Developer machines have `go env -w` settings; CI starts fresh. Use `GOFLAGS` env or run `go env -w` in CI prologue.

**Trusting the public proxy with private code.** `proxy.golang.org` doesn't host private repos, but if your `GOPRIVATE` is missing, the proxy logs the lookup attempt. Subtle leak.

**Updating `~/.netrc` across team members manually.** Use a secrets manager or scripted setup.

**Proxy that doesn't verify checksums.** Some homemade proxies skip hash verification. Verify with `go mod verify` after fetch.

**Athens permanent failure mode.** When athens is misconfigured or down, `GOPROXY=https://athens|direct` falls through to direct (which may fail too). Monitor athens uptime separately.

**Disabling the sumdb because "private modules don't have entries".** Private modules never had public sumdb entries; the sumdb only verifies modules it knows. `GOSUMDB=off` is overkill; `GOPRIVATE` is the targeted fix.

## Performance Notes

- Public proxy fetch: ~100–500 ms per module (latency + transfer).
- Internal proxy (athens, JFrog): ~10–100 ms LAN.
- Direct VCS (git): ~500 ms – 5 s (clone overhead).
- `go.sum` verification: <1 ms per dep.
- Athens with disk storage: ~10 ms per cached hit.
- Cold dep tree (200 deps): 30 s – 2 min over public proxy; 5–15 s over LAN athens.
- Athens memory footprint: tens of MB.
- Sumdb verification: one HTTP roundtrip on first fetch per module-version.

For build performance, a warm internal proxy is the single biggest win after the build cache itself.

## How Big Companies Use It

- **Google internal**: a custom internal proxy mirror with comprehensive VCS auth, replaces external proxy for first-party code.
- **Cloudflare**: documented their Athens deployment plus `GOPRIVATE` policy: https://blog.cloudflare.com/secure-and-fast-with-athens/.
- **Uber**: documents private module workflows in their style guide: https://github.com/uber-go/guide.
- **HashiCorp**: uses JFrog Artifactory across products for caching and SBOM auditing.
- **GitLab**: their own platform's Go projects use the built-in GitLab Package Registry: https://docs.gitlab.com/ee/user/packages/go_proxy/.
- **Bytedance/TikTok**: documented Athens deployment at GopherCon China 2023.
- **CNCF projects** generally use the public proxy + `GOPRIVATE` for any internal forks during development.
- **Discord**: uses Athens internally for build determinism across staging/prod.

## Source Code References

Pinned to `go1.26`.

- `GOPRIVATE` / `GONOPROXY` / `GONOSUMDB` resolution: [`src/cmd/go/internal/modfetch/proxy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/proxy.go).
- Proxy fetch chain: same file, function `proxyURLs`.
- Direct VCS fetch: [`src/cmd/go/internal/vcs/vcs.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/vcs/vcs.go).
- Sumdb client: [`src/cmd/go/internal/modfetch/fetch.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/fetch.go).
- Module path globbing: [`src/cmd/go/internal/modfetch/proxy.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modfetch/proxy.go) — search `MatchPrefixPatterns`.
- `go env` config: [`src/cmd/go/internal/envcmd/env.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/envcmd/env.go).
- Athens source: [`github.com/gomods/athens`](https://github.com/gomods/athens).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Module proxy protocol": https://go.dev/ref/mod#module-proxy.
- "Private modules": https://go.dev/ref/mod#private-modules.
- "Go module proxy concepts": https://go.dev/blog/module-mirror-launch.
- "Configuring GOPRIVATE": https://go.dev/ref/mod#environment-variables.
- Athens documentation: https://docs.gomods.io.
- JFrog Go module proxy guide: https://jfrog.com/help/r/jfrog-artifactory-documentation/go-registry.
- GitLab Go proxy docs: https://docs.gitlab.com/ee/user/packages/go_proxy/.
- "Go module checksum database design" (Russ Cox): https://research.swtch.com/tlog.
- "Supply-chain security in Go" (sigstore + sumdb): https://blog.sigstore.dev.

## Exercises / Self-Check

1. Set `GOPRIVATE='github.com/myorg/*'` and pull a private repo using SSH. Verify the public proxy is never contacted via packet capture or `GODEBUG=installgoroot=1`.
2. Stand up Athens locally with disk storage. Configure it to proxy `golang.org/x/...` modules. Point a project at it and confirm caching works.
3. A CI build fails: `verifying module: checksum mismatch`. What are three potential causes, and how would you diagnose each?
4. Compare `GOPROXY=...|direct` (pipe) vs `GOPROXY=...,direct` (comma) behavior when the proxy returns 500. Which falls through?
5. Write a `GOPRIVATE` value that matches `github.com/myorg/anything` and `gitlab.corp.com/team-*`. Verify with `go env GOPRIVATE`.
