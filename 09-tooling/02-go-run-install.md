# `go run` and `go install` — Running and Installing

## TL;DR

`go run` compiles to a *temporary directory* under `$GOTMPDIR`, executes the binary, and (when the process exits) deletes it. It is meant for scripts and one-off programs; **never** use it as a long-running entry point in production. `go install` compiles and writes the resulting binary into `$GOBIN` (defaulting to `$GOPATH/bin`, defaulting to `$HOME/go/bin`); since Go 1.16 it accepts a version suffix — `go install example.com/cmd/foo@v1.2.3` — that installs from a module *without* touching the current module's `go.mod`. That version-suffixed form is now the standard way to install developer tools (`stringer`, `mockgen`, `delve`, `govulncheck`, …). For libraries (non-`main` packages), `go install` populates the build cache and writes nothing to `$GOBIN`.

## Mental Model

```
   go run                              go install
   ─────────                           ───────────
       ▼                                    ▼
   pkg → compile → /tmp/.../bin       pkg → compile → $GOBIN/<name>
       ▼                                    ▼
   exec & wait for exit                exit; binary stays
       ▼
   rm /tmp/.../bin

   go install example.com/cmd/foo@v1.2.3
       │
       ├─ resolves foo as if it were the only dep
       ├─ ignores the current dir's go.mod entirely
       └─ writes $GOBIN/foo
```

`go install` with a version suffix runs in a **sandboxed module context** — it doesn't pollute your project's `go.mod`. That's the whole reason it was introduced.

## Syntax & Basic Usage

```bash
# go run
$ go run main.go
$ go run .
$ go run ./cmd/seed
$ go run github.com/user/repo/cmd/tool@latest   # since 1.17

# go install
$ go install ./cmd/app                          # build current module's cmd
$ go install ./...                              # install every main pkg in module
$ go install golang.org/x/tools/cmd/stringer@latest
$ go install github.com/go-delve/delve/cmd/dlv@v1.22.0
$ go install example.com/cmd/foo@master         # branch
$ go install example.com/cmd/foo@v1.2.3         # tag
```

Find where binaries land:

```bash
$ go env GOBIN
$ go env GOPATH
$ echo $PATH | tr ':' '\n' | grep go
```

Make sure `$GOBIN` (or `$GOPATH/bin`) is in `$PATH` — otherwise installed tools aren't discoverable.

## Deep Dive

### `go run` mechanics

```bash
$ go run -x main.go 2>&1 | head -20
```

`-x` prints every command. You'll see:

```
WORK=/var/folders/.../go-build1234
mkdir -p $WORK/b001/
cd /Users/me/proj
/usr/local/go/pkg/tool/darwin_arm64/compile -o $WORK/b001/_pkg_.a ... main.go
...link...
$WORK/b001/exe/main
rm -r $WORK/
```

The `$WORK` directory is cleaned up when the process exits. Pass `-work` to keep it for inspection (`go run -work main.go`).

The binary is built **with the standard build cache**. So a second `go run main.go` after a no-op edit still consults `$GOCACHE` — only the final link runs from scratch.

### `go run` with a version suffix (since 1.17)

```bash
$ go run github.com/google/go-jsonnet/cmd/jsonnet@latest -- input.jsonnet
```

The toolchain creates a synthetic module under `$GOMODCACHE`, resolves the dep tree as if `jsonnet` were the only require, builds, and runs. Useful for "run this one tool once" without polluting `$GOPATH/bin`. Beware: the binary is built fresh each run unless cached.

### `go run` doesn't honor `-buildvcs=auto`

The temp build doesn't have `.git` in its workspace, so VCS info is empty. If your code calls `runtime/debug.ReadBuildInfo()`, the `vcs.*` keys will be missing. Build with `go build` (or `go install`) if you need them.

### `go install`'s history

Before Go 1.16, `go get` did three things: download, install, and update `go.mod`. That conflation caused endless confusion. The 1.16 split:

- `go get` — *only* edits `go.mod` (and only inside a module).
- `go install pkg@version` — *only* compiles and installs to `$GOBIN`.

This means: **never use `go get` to install a binary**. Use `go install pkg@version`.

### `go install pkg@version` resolution

`go install example.com/cmd/foo@v1.2.3` is roughly equivalent to:

```bash
$ cd $(mktemp -d)
$ go mod init synthetic
$ go get example.com/cmd/foo@v1.2.3
$ go install example.com/cmd/foo
```

— except it's atomic, hits the proxy directly, and the synthetic module never persists. The version can be a tag (`v1.2.3`), branch (`main`), pseudo-version (`v0.0.0-20240115...-abc`), or `latest` / `upgrade` / `patch`.

Versions are mandatory for binaries installed *outside* a module context. Inside a module (`go install ./cmd/foo`), you're already in a versioned world via the current `go.mod`.

### `$GOBIN` resolution

```bash
$ go env GOBIN          # if set, binaries go here
$ go env GOPATH         # if GOBIN unset, $GOPATH/bin
$ echo $HOME/go/bin     # if GOPATH unset (default)
```

A common setup:

```bash
# ~/.zshrc
export PATH="$(go env GOPATH)/bin:$PATH"
# or, with a custom location
export GOBIN="$HOME/.local/bin"
export PATH="$GOBIN:$PATH"
```

The toolchain refuses to install if `$GOBIN` is unset *and* `$GOPATH` is unset — must be at least one defined.

### Cross-compiling installs

```bash
$ GOOS=linux GOARCH=amd64 go install ./cmd/app
```

Goes to `$GOBIN/linux_amd64/app` (subdirectory inserted when cross-compiling). Useful for building a Linux binary on a Mac dev box and `scp`ing it. Be careful — `which app` won't find it because the subdirectory isn't in `$PATH`.

### `go install` for libraries

If you `go install` a non-`main` package, the toolchain populates the build cache but writes nothing to `$GOBIN`. Historically (pre-modules) this wrote `.a` files under `$GOPATH/pkg/`. That's gone since 1.10 (~).

The result is `go install ./...` in a library module is a no-op for filesystem output — but it's a great way to populate the cache and verify the whole module builds.

### Tool dependencies in `go.mod` (since 1.24)

Before 1.24, the idiomatic way to pin tool versions used by your project was a `tools.go` file with a blank import inside a `// +build tools` constraint:

```go
//go:build tools
// +build tools

package tools

import (
    _ "golang.org/x/tools/cmd/stringer"
    _ "github.com/golang/mock/mockgen"
)
```

Then `go install` would resolve via the module's `go.mod`. Since Go 1.24, `go.mod` has a first-class `tool` directive:

```
tool (
    golang.org/x/tools/cmd/stringer
    github.com/golang/mock/mockgen
)
```

And the new `go tool` command resolves these:

```bash
$ go tool stringer -type=Color
$ go tool mockgen ...
```

See `09-tooling/10-go-fix-and-go-tool.md` for the full story.

### `go run` exit code

`go run`'s exit code matches the program's exit code. So:

```bash
$ go run ./cmd/maybe-fails || echo "failed: $?"
```

works as expected. A compile error returns exit code 1 with the compile output on stderr.

### `go run` and signals

`go run` forwards SIGINT/SIGTERM to the child. Pressing Ctrl-C kills the program, then `go run` itself exits and cleans up the temp dir. If the child traps signals (e.g., for graceful shutdown), `go run` still waits.

There's one subtle pitfall: on Windows, `go run` and the child are in the same console process group, so `Ctrl-C` may behave slightly differently from `os.Interrupt` in production.

### `go install` and version locking

```bash
$ go install golang.org/x/tools/cmd/stringer@latest
$ stringer --version    # whatever was tagged at install time
```

There's no manifest tracking what you installed. To stay reproducible, pin versions:

```bash
$ go install golang.org/x/tools/cmd/stringer@v0.20.0
```

Or use the `go.mod` `tool` directive (1.24+).

## Standard Library Hooks

- `os.Executable()` — path to the running binary; useful inside a `go install`-installed tool.
- `runtime/debug.ReadBuildInfo` — read embedded info.
- `os/exec` — invoke installed tools from Go code (`exec.Command("stringer", ...)`).
- `flag` — almost every `go install`-able tool uses `flag` for CLI parsing.

## Real-World Patterns

### 1. Quick prototype with `go run`

```bash
$ cat > /tmp/probe.go <<'EOF'
package main
import "fmt"
func main() { fmt.Println(1+1) }
EOF
$ go run /tmp/probe.go
2
```

### 2. Install a tool without polluting `go.mod`

```bash
$ go install honnef.co/go/tools/cmd/staticcheck@2024.1.1
$ staticcheck ./...
```

`go.mod` of the current project is untouched.

### 3. Tool versions pinned via `go.mod` (1.24+)

```
// go.mod
module github.com/me/proj

go 1.26

tool (
    golang.org/x/tools/cmd/stringer
    github.com/golang/mock/mockgen
    honnef.co/go/tools/cmd/staticcheck
)
```

Then:

```bash
$ go tool stringer -type=Color ./types
$ go tool staticcheck ./...
```

### 4. CI installer step

```yaml
- name: Install tools
  run: |
    go install golang.org/x/tools/cmd/stringer@v0.20.0
    go install github.com/golang/mock/mockgen@v1.6.0
    echo "$(go env GOPATH)/bin" >> "$GITHUB_PATH"
```

### 5. `go run` as `make` substitute

```makefile
generate:
    go run github.com/golang/mock/mockgen@v1.6.0 -source=foo.go -destination=mock_foo.go
test:
    go test ./...
```

No `go install` step needed — `go run` builds-and-runs in one go. Slower per invocation but simpler.

### 6. Cross-compiled tool for a remote server

```bash
$ GOOS=linux GOARCH=amd64 go install ./cmd/probe
$ scp "$(go env GOPATH)/bin/linux_amd64/probe" server:/usr/local/bin/
```

## Anti-Patterns & Gotchas

**Using `go run` in production.** It compiles every time; container image bloats because the toolchain has to be present; cleanup is fragile if SIGKILL hits. Always `go build` + run the binary.

**Using `go get` to install a binary.** Since 1.16, `go get pkg` only edits `go.mod` (and only inside a module). Outside a module, it errors. Always use `go install pkg@version`.

**`go install` of a remote package inside a module.** Without `@version`, it's an error since 1.16. Always include the version suffix when installing tools.

**Forgetting `$GOBIN` (or `$GOPATH/bin`) in `$PATH`.** You install `stringer`, run `stringer`, get "command not found". Add the bin dir.

**Pinning `@latest` for CI tools.** `latest` resolves at install time; different CI runs may install different versions. Pin a tag for reproducibility.

**Assuming `go install` is hermetic.** It still consults `$GOPROXY`, `$GOSUMDB`, the module cache. Air-gapped CI needs `GOPROXY=off` and a pre-warmed cache.

**`go run ./...`** — runs *every* `main` package in the module sequentially. Almost never what you want; `go install ./...` populates them all without running.

**Relying on `runtime/debug.ReadBuildInfo()` working under `go run`.** It does, but `vcs.*` keys are typically empty. Test with `go build` first if you depend on them.

**Confusing `go run pkg@latest` with `go install pkg@latest`.** Both work; `go install` keeps the binary, `go run` discards it. If you'll need it again next minute, install once.

**Tracking tool versions in a `tools.go` *and* a `go.mod` `tool` block (1.24+).** Pick one. Mixing them confuses future maintainers.

**Putting `go install ./...` in a tight CI loop.** It rebuilds against the current module's `go.mod`. If your tool deps don't match what the tool needs internally, you can get weird "this require is at the wrong version" errors. Install with `@version` outside the module.

## Performance Notes

- `go run` first compile: ~0.5–3 s for a small program (mostly link time).
- `go run` after edits (cache warm): ~50–200 ms.
- `go install` of a tool (first time, deps not cached): 2–10 s.
- `go install` of a tool (warm cache): 100–500 ms.
- `go install` of a remote tool via proxy: dominated by download (~1–3 s).
- `go install ./...` on a 200-pkg module: ~2–5 s warm.

`go run` is convenient but not free: every iteration pays the link cost. For tight inner loops (TDD with a custom binary), build once and re-run.

## How Big Companies Use It

- **Google** uses `go install`-style version pinning in internal tooling, then mirrors approved versions through their internal proxy: https://go.dev/blog/v2-go-modules.
- **Kubernetes** uses `go install` plus a `vendor/` dir for tools required by the build (`vendor/k8s.io/code-generator/...`): https://github.com/kubernetes/kubernetes.
- **The Go team** dogfoods the `tool` directive (1.24+) inside the `golang.org/x/tools` repo: https://github.com/golang/tools.
- **Discord** runs every dev workflow through `go run` (sqlc, mockgen, asm-generators) to avoid global tool installs on developer machines.
- **HashiCorp** pins tool versions in a `tools/tools.go` file across most of their repos: https://github.com/hashicorp/terraform.
- **Tailscale** publishes a `tool-versions` file and uses `go install` in their `Makefile`: https://github.com/tailscale/tailscale.
- **Cockroach Labs** uses `bazel`-driven builds for production but `go install` for ad-hoc developer tools.

## Source Code References

Pinned to `go1.26`.

- `go run` command: [`src/cmd/go/internal/run/run.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/run/run.go).
- `go install` command: [`src/cmd/go/internal/work/build.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/work/build.go) (search for `runInstall`).
- Version-suffix install: [`src/cmd/go/internal/modcmd/install.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modcmd/install.go) (or follow `install pkg@version` logic in `modload`).
- `tool` directive support: [`src/cmd/go/internal/modload/init.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/modload/init.go).
- `go tool`: [`src/cmd/go/internal/tool/tool.go`](https://github.com/golang/go/blob/release-branch.go1.26/src/cmd/go/internal/tool/tool.go).

(BSD-3-Clause © The Go Authors.)

## Further Reading

- "Command go — Run Go program": https://pkg.go.dev/cmd/go#hdr-Compile_and_run_Go_program.
- "Command go — Install": https://pkg.go.dev/cmd/go#hdr-Compile_and_install_packages_and_dependencies.
- "Go 1.16: `go install` and `go get`" (Jay Conrod): https://go.dev/blog/go116-module-changes.
- "Go 1.24 tool directive" (release notes): https://tip.golang.org/doc/go1.24.
- Russ Cox, "go install for binaries" (golang-dev thread): https://go.googlesource.com/proposal/+/refs/heads/master/design/24250-go-install-version.md.
- Dave Cheney, "Installing tools with Go modules": https://dave.cheney.net/2019/10/29/go-modules-and-the-tools-tooling.

## Exercises / Self-Check

1. Compare the binary produced by `go build ./cmd/foo` and the one `go run` produces under `$WORK`. Are they identical?
2. Use `go install` to pin two versions of `stringer` (`v0.18.0` and `v0.20.0`). Where does each one live? How do you switch?
3. Convert a `tools/tools.go` file to a 1.24 `tool` directive. Run a tool via `go tool` and confirm it picks the pinned version.
4. Try `go install ./...` on a library module (no `main` packages). What happens in `$GOBIN`?
5. Use `go run -work main.go` and inspect the temp directory. Find the link command line; identify the inputs.
