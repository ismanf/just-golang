# Third-Party Tools — `air`, `goreleaser`, `mage`, `task`

## TL;DR

The Go ecosystem has a small set of widely-used "ergonomics" tools written in Go that aren't part of the toolchain:

- **`air`** — live-reload for development. Watches your source, rebuilds, restarts your app on every save. The Go equivalent of `nodemon`.
- **`goreleaser`** — packaging and release automation. Builds for many platforms, archives, generates checksums, signs, publishes to GitHub Releases / Docker Hub / Homebrew taps, generates changelogs. A YAML-configured "go build" on steroids.
- **`mage`** — Make-replacement written in Go. Targets are plain Go functions; no Makefile syntax, full IDE support.
- **`task`** — YAML-configured task runner. More opinionated than Make; cross-platform out of the box.

Most Go projects pick *one* of mage/task (or stick with Make), and most projects that produce binaries adopt goreleaser. `air` is universal for service dev.

## Mental Model

```
   air              ── dev-time live reload
        └─ watches .go files; rebuilds; restarts process

   goreleaser       ── release-time multi-arch packaging
        └─ go build × (GOOS × GOARCH) + archive + checksum + sign + publish

   mage             ── Make-replacement, targets in Go
        └─ runs Go functions; deps as Go calls

   task             ── Make-replacement, targets in YAML
        └─ runs shell commands; deps declared in YAML

   Make / Just / Bazel  ── traditional alternatives (out of scope)
```

These tools share a property the Go toolchain lacks: they wrap day-to-day workflows that aren't compile/test/run.

## Syntax & Basic Usage

```bash
# air
$ go install github.com/cosmtrek/air@latest
$ air init                          # create .air.toml
$ air                                # start watching

# goreleaser
$ go install github.com/goreleaser/goreleaser/v2@latest
$ goreleaser init                   # create .goreleaser.yaml
$ goreleaser release --snapshot --clean   # local dry run
$ goreleaser release --clean        # publish

# mage
$ go install github.com/magefile/mage@latest
$ mage -init                        # create magefile.go
$ mage build                        # run the Build target
$ mage -l                           # list targets

# task
$ go install github.com/go-task/task/v3/cmd/task@latest
$ task --init                       # create Taskfile.yml
$ task build
$ task --list                       # list tasks
```

## Deep Dive

### `air` — live reload

**Why**: edit code → see effect immediately, without manually rebuilding/restarting.

**How it works**: scans your source tree, watches files via `fsnotify`, on change runs `go build`, kills the old process, starts the new one.

`.air.toml`:

```toml
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/main ./cmd/app"
  bin = "./tmp/main"
  full_bin = ""
  args_bin = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  exclude_dir = ["tmp", "vendor", "node_modules"]
  include_dir = []
  exclude_file = []
  exclude_regex = ["_test\\.go"]
  exclude_unchanged = true
  follow_symlink = false
  log = "build-errors.log"
  delay = 1000               # ms
  stop_on_error = true
  send_interrupt = false
  kill_delay = "500ms"

[log]
  time = false

[color]
  main = "magenta"
  watcher = "cyan"
  build = "yellow"
  runner = "green"

[misc]
  clean_on_exit = true
```

Run `air` and edit a `.go` file — within seconds you'll see "build" then "main pid 12345" in the terminal.

**Production note**: air is a *dev* tool. Don't ship containers that run `air` in production.

### `goreleaser` — release packaging

**Why**: producing release binaries by hand (cross-compile per OS/arch, archive, checksum, sign, upload, write changelog, push Homebrew tap) is repetitive and error-prone.

**How it works**: reads `.goreleaser.yaml`, runs `go build` for each combination of GOOS/GOARCH, packs into `.tar.gz` / `.zip` / `.deb` / `.rpm` / `.apk`, computes checksums, signs (cosign/gpg), uploads to GitHub Releases, optionally pushes a Docker manifest and updates a Homebrew formula.

Minimal `.goreleaser.yaml`:

```yaml
version: 2
project_name: myapp

before:
  hooks:
    - go mod tidy

builds:
  - main: ./cmd/app
    binary: myapp
    env: [CGO_ENABLED=0]
    flags: [-trimpath]
    ldflags: |
      -s -w
      -X main.Version={{.Version}}
      -X main.Commit={{.Commit}}
      -X main.Date={{.Date}}
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]

archives:
  - format: tar.gz
    name_template: '{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}'
    format_overrides:
      - goos: windows
        format: zip
    files:
      - LICENSE
      - README.md

checksum:
  name_template: 'checksums.txt'

changelog:
  use: github
  groups:
    - title: Features
      regexp: '^.*?feat(\(.+\))??:.+$'
      order: 0
    - title: Bug fixes
      regexp: '^.*?fix(\(.+\))??:.+$'
      order: 1

dockers:
  - image_templates:
      - 'ghcr.io/me/myapp:{{ .Version }}'
      - 'ghcr.io/me/myapp:latest'
    dockerfile: Dockerfile

release:
  github:
    owner: me
    name: myapp
  draft: true
```

CI integration:

```yaml
- uses: goreleaser/goreleaser-action@v6
  with:
    distribution: goreleaser
    version: '~> v2'
    args: release --clean
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The action picks up tags (`git tag v1.2.3 && git push --tags`), runs goreleaser, publishes.

**Snapshot mode**: `goreleaser release --snapshot --clean` builds everything locally without publishing — invaluable for testing the config.

**Pro features** (`goreleaser-pro`): code-signing for macOS notarization, S3 publishing, more output formats. Free version covers most needs.

### `mage` — Make in Go

**Why**: Make's syntax is gnarly; `.PHONY:`, recursive `$(MAKE)`, escaping `$$` for variables. Mage targets are Go functions; refactoring works; types are checked.

`magefile.go`:

```go
//go:build mage

package main

import (
    "fmt"

    "github.com/magefile/mage/mg"
    "github.com/magefile/mage/sh"
)

// Build compiles the binary.
func Build() error {
    return sh.Run("go", "build", "-o", "bin/app", "./cmd/app")
}

// Test runs unit tests.
func Test() error {
    return sh.RunV("go", "test", "-race", "./...")
}

// Lint runs golangci-lint.
func Lint() error {
    return sh.RunV("golangci-lint", "run", "./...")
}

// CI runs lint then test then build.
func CI() {
    mg.SerialDeps(Lint, Test, Build)
}

// Deploy builds and uploads.
func Deploy() error {
    mg.Deps(Build)
    return sh.RunV("scp", "bin/app", "server:/usr/local/bin/")
}

// Default is the target to run when mage is called without args.
var Default = Build
```

```bash
$ mage -l
Targets:
  build*   Build compiles the binary.
  ci       CI runs lint then test then build.
  deploy   Deploy builds and uploads.
  lint     Lint runs golangci-lint.
  test     Test runs unit tests.

  * default target

$ mage              # runs Build
$ mage test
$ mage ci
```

**Deps**: `mg.Deps(A, B, C)` runs A, B, C in parallel (each once); `mg.SerialDeps` runs sequentially.

**Compiled mode**: `mage -compile bin/mage` produces a single binary; ship it for CI environments without mage installed.

### `task` — YAML task runner

**Why**: Make is shell-tied; Mage is Go-tied. Task is a middle ground: YAML, runs shell, cross-platform built in.

`Taskfile.yml`:

```yaml
version: '3'

env:
  CGO_ENABLED: 0

vars:
  BIN_DIR: ./bin
  VERSION:
    sh: git describe --tags --always

tasks:
  default:
    deps: [build]

  build:
    desc: Build the binary
    cmds:
      - go build -trimpath -ldflags="-s -w -X main.Version={{.VERSION}}" -o {{.BIN_DIR}}/app ./cmd/app
    sources:
      - "**/*.go"
      - go.mod
      - go.sum
    generates:
      - "{{.BIN_DIR}}/app"

  test:
    desc: Run tests
    cmds:
      - go test -race -count=1 ./...

  lint:
    desc: Run linters
    cmds:
      - golangci-lint run ./...

  ci:
    desc: Run lint + test + build
    deps: [lint, test, build]

  clean:
    desc: Remove build artifacts
    cmds:
      - rm -rf {{.BIN_DIR}}
```

```bash
$ task              # runs default → build
$ task test
$ task ci           # runs lint, test, build in parallel
$ task --list
```

**Smart re-runs**: the `sources:`/`generates:` block lets Task skip running `build` if no source changed and the output exists. Like Make's timestamp check, but better with globs.

**Includes**: split into multiple files via `includes:`.

**Cross-platform**: Task has a shell built in (`mvdan/sh`); your scripts run on Linux/macOS/Windows without porting.

### Compare: Make vs. Mage vs. Task

| Feature             | Make           | Mage            | Task            |
|---------------------|----------------|-----------------|-----------------|
| Syntax              | Makefile DSL   | Go              | YAML            |
| Dependencies        | Implicit       | Explicit (`mg.Deps`) | Explicit (`deps:`) |
| Parallelism         | `-j`           | `mg.Deps` parallel | `deps:` parallel |
| Cross-platform      | Hard (POSIX vs Windows) | Yes        | Yes             |
| IDE support         | Minimal         | Full (Go)        | YAML schema     |
| Boot time           | Instant         | Compiles first time, then cached | Instant |
| Targets visible from CLI | `make -p` | `mage -l`        | `task --list`   |
| Ecosystem           | Universal       | Go-only          | Growing         |

Pick by what your team is comfortable with. Mage suits Go-only teams; Task fits polyglot orgs; Make is what your existing infra probably uses.

### Lesser-known but used

- **`docker buildx bake`** — multi-platform Docker builds; sometimes pairs with goreleaser.
- **`ko`** — build OCI images from Go without a Dockerfile (https://ko.build). Bypasses `docker` entirely.
- **`just`** — non-Go Make alternative; some Go projects pick it (https://github.com/casey/just).
- **`reflex`** — older alternative to `air`.
- **`watchexec`** — generic file-watch runner; works for any language.
- **`modd`** — YAML-driven dev process orchestrator.
- **`overmind` / `foreman`** — Procfile-based dev orchestrators (multi-process).
- **`nilaway`** — Uber's nil-safety static analyzer.
- **`go-acc`** — coverage accumulator across `go test ./...`.
- **`gomarkdoc`** — generates Markdown docs from Go doc comments.

## Real-World Patterns

### 1. air + goreleaser for a service

```toml
# .air.toml — dev
[build]
  cmd = "go build -o tmp/server ./cmd/server"
  bin = "tmp/server"
```

```yaml
# .goreleaser.yaml — release
builds:
  - main: ./cmd/server
    binary: server
```

Dev loop: `air`. Release: `git tag v1.0.0 && git push --tags` → CI runs goreleaser.

### 2. mage targets for typical CI

```go
func Lint() error { return sh.RunV("golangci-lint", "run", "./...") }
func Test() error { return sh.RunV("go", "test", "-race", "./...") }
func Build() error { return sh.RunV("go", "build", "-o", "bin/app", "./cmd/app") }
func CI() { mg.SerialDeps(Lint, Test, Build) }
```

CI just runs `mage ci`.

### 3. task with watch

```yaml
tasks:
  dev:
    cmds:
      - task -w build
```

`-w` watches `sources:` and reruns.

### 4. goreleaser to Homebrew tap

```yaml
brews:
  - tap:
      owner: me
      name: homebrew-tools
    folder: Formula
    homepage: https://example.com/myapp
    description: My application
    license: MIT
```

After release: `brew tap me/tools && brew install myapp`.

### 5. ko for image without Dockerfile

```bash
$ ko build ./cmd/app
ko.local/app-abc123:latest
```

ko reads `go build`; produces a multi-arch OCI image; pushes to registry. Skips Dockerfile entirely.

### 6. mage with goreleaser inside

```go
func Release() error {
    return sh.RunV("goreleaser", "release", "--clean")
}
```

`mage release` becomes the team's one entry point.

### 7. task with versioned tools

```yaml
tasks:
  install-tools:
    cmds:
      - go install golang.org/x/tools/cmd/stringer@v0.20.0
      - go install github.com/golang/mock/mockgen@v1.6.0
```

Or use Go 1.24's `tool` directive (`09-tooling/10-go-fix-and-go-tool.md`).

## Anti-Patterns & Gotchas

**Running `air` in production.** It rebuilds on every change; rebuilds need the Go toolchain present and bloat the image. Dev-only.

**Configuring `air` to watch `vendor/`.** Vendor directories should be excluded; default config does this — verify.

**`goreleaser` with unpinned tool version.** Each release uses whatever `latest` is at the time. Pin in CI: `version: 'v2.5.0'`.

**`goreleaser release --clean` without `--snapshot` first.** Publishes immediately. Always `--snapshot` to test, *then* `--clean` to ship.

**Mage targets that call `sh.Run` for everything.** You could just write Go. Mix: use Go for control flow, `sh.Run` only for actual subprocess work.

**Mage's compiled mode without `-keep`.** The compiled binary may be one-shot; subsequent edits require recompile. Set `-keep` if persisting.

**Task `sources:` without `generates:`.** Task can't tell if it needs to re-run; rebuilds every time. Always include outputs.

**Mixing Make + Mage + Task in one repo.** Confusing; team doesn't know which entry point is canonical. Pick one.

**`goreleaser` with `dockers:` and forgetting Docker auth in CI.** Build succeeds; push fails. Add `docker/login-action` before goreleaser.

**Per-developer `.air.toml` deviations.** Drift across team. Commit a baseline; let individuals override locally.

**Mage targets without doc comments.** `mage -l` shows blank descriptions. Always add `// FuncName does X.` doc comment.

**`task --watch` with too-broad sources.** Rebuilds on touch of `.git/index`. Scope `sources:` precisely.

**`goreleaser` for "build a single binary in CI".** Overkill. Use plain `go build` for non-release builds.

## Performance Notes

- `air` rebuild: dominated by `go build` time (~1–5 s for small services).
- `goreleaser` full release: 30 s – 5 min depending on platform/arch matrix.
- `mage -compile`: 1–5 s first time; subsequent runs are instant.
- `task` startup: <100 ms.
- `task` rebuild check (`sources:`/`generates:`): <50 ms.

All four are negligible compared to the work they orchestrate.

## How Big Companies Use It

- **Cloudflare** uses `air` for dev of their Workers Go runtime: https://blog.cloudflare.com.
- **HashiCorp** uses `goreleaser` for Terraform provider releases: https://github.com/hashicorp/terraform-provider-aws/blob/main/.goreleaser.yml.
- **Grafana Labs** uses `mage` across grafana, loki, mimir, tempo: https://github.com/grafana/loki.
- **Kubernetes** uses Make + bash; they have an internal build system (Bazel) for production: https://github.com/kubernetes/kubernetes.
- **Tailscale** uses `goreleaser` for `tailscale` binary releases: https://github.com/tailscale/tailscale.
- **Discord** uses `task` for cross-service dev workflows: https://github.com/discord.
- **CockroachDB** uses Bazel + custom scripts; no goreleaser: https://github.com/cockroachdb/cockroach.
- **Charm** (the bubbletea/lipgloss org) uses goreleaser across their CLI tools: https://github.com/charmbracelet.

## Source Code References

- `air`: [`github.com/cosmtrek/air`](https://github.com/cosmtrek/air).
- `goreleaser`: [`github.com/goreleaser/goreleaser`](https://github.com/goreleaser/goreleaser).
- `mage`: [`github.com/magefile/mage`](https://github.com/magefile/mage).
- `task`: [`github.com/go-task/task`](https://github.com/go-task/task).
- `ko`: [`github.com/ko-build/ko`](https://github.com/ko-build/ko).
- `just`: [`github.com/casey/just`](https://github.com/casey/just).
- `watchexec`: [`github.com/watchexec/watchexec`](https://github.com/watchexec/watchexec).

(MIT licenses for `air`, `mage`, `task`, `ko`; Apache-2.0 for `goreleaser`.)

## Further Reading

- "air documentation": https://github.com/cosmtrek/air#readme.
- "goreleaser docs": https://goreleaser.com/intro.
- "mage book": https://magefile.org.
- "Task documentation": https://taskfile.dev.
- "Releasing Go binaries with goreleaser" (Goreleaser blog): https://carlosbecker.com/posts/goreleaser-best-practices.
- "Mage vs Make vs Task" (various comparisons): https://blog.kowalczyk.info/article/d4HwT/build-tools-comparison-make-vs-mage-vs-task.html.
- "ko: container images without Dockerfiles" (Jason Hall): https://medium.com/google-cloud/ko-fast-kubernetes-microservice-development-in-go-f93bf0a3deeb.

## Exercises / Self-Check

1. Set up `air` for a small HTTP service. Edit a handler; confirm the process restarts within a second.
2. Configure `.goreleaser.yaml` for a single-binary CLI; run `--snapshot` to verify before tagging.
3. Write a Magefile with `Lint`, `Test`, `Build`, and `CI` targets. `mage CI` should run all three.
4. Convert a Makefile to a Taskfile. Which Make features were tedious to replicate?
5. Use `ko build ./cmd/app` to produce a container image without writing a Dockerfile. Compare to `docker build`.
