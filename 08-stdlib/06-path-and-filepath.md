# `path` and `path/filepath` — Path Manipulation

## TL;DR

`path` operates on slash-separated paths (URLs, `io/fs` names, archive entries). `path/filepath` operates on OS-native paths (`/` on Unix, `\` on Windows, with drive letters, UNC paths, etc.). Use `path` for in-memory virtual paths, `filepath` for anything touching the real filesystem. Neither is a security boundary — for that, use `os.Root` (1.24+).

## Mental Model

```
URLs, fs.FS names, zip entries → path
                                  ├─ separator is always '/'
                                  └─ Clean, Join, Split, Match, Ext

OS file paths                  → filepath
                                  ├─ separator is OS-specific
                                  ├─ understands volumes/drives
                                  └─ Walk, Glob, Abs, EvalSymlinks
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"path"
	"path/filepath"
)

func main() {
	fmt.Println(path.Join("a", "b/", "c"))       // a/b/c
	fmt.Println(filepath.Join("a", "b/", "c"))   // a/b/c (Unix); a\b\c (Windows)
	fmt.Println(filepath.Ext("photo.jpg"))       // .jpg
	abs, _ := filepath.Abs("rel/path")
	fmt.Println(abs)
	// Output (Unix):
	// a/b/c
	// a/b/c
	// .jpg
	// /pwd/rel/path
}
```

## Deep Dive

### `path` (virtual, always `/`)

- `path.Join(elem...)` — joins, then Clean.
- `path.Clean(p)` — collapses `..`, removes duplicate `/`.
- `path.Split(p)` — returns `(dir, file)`.
- `path.Base(p)` — last element.
- `path.Dir(p)` — everything but the last element.
- `path.Ext(p)` — `.ext` including the dot.
- `path.Match(pattern, name)` — glob match, returns (matched, error).
- `path.IsAbs(p)` — starts with `/`.

### `path/filepath` (OS-aware)

Everything `path` does plus:

- Uses `filepath.Separator` and `filepath.ListSeparator`.
- `filepath.ToSlash`/`FromSlash` to convert between.
- `filepath.Abs`, `filepath.Rel`.
- `filepath.EvalSymlinks` — resolves symlinks to the real path.
- `filepath.VolumeName(p)` — drive or UNC prefix on Windows.
- `filepath.Walk` / `filepath.WalkDir` (1.16+) — recursive traversal. Prefer `WalkDir`.
- `filepath.Glob(pattern)` — globs against the real filesystem.

### `filepath.WalkDir` vs `filepath.Walk`

`Walk` calls `os.Lstat` on every entry. `WalkDir` (1.16+) uses the `fs.DirEntry` from the parent directory's listing, deferring `Stat` until you ask. Significantly faster on large trees.

```go
filepath.WalkDir("/var/data", func(p string, d fs.DirEntry, err error) error {
	if err != nil { return err }
	if d.IsDir() { return nil }
	if !strings.HasSuffix(d.Name(), ".log") { return nil }
	process(p)
	return nil
})
```

Return `filepath.SkipDir` from the callback to skip a directory, `filepath.SkipAll` (1.20+) to abort the walk early.

### `Clean` semantics

`Clean("a/./b/../c")` → `"a/c"`.  
`Clean("")` → `"."`.  
`Clean("/..")` → `"/"`.

`Clean` does not consult the filesystem. It cannot follow symlinks.

### Globbing

`filepath.Glob("*.go")` matches files via OS calls. Patterns: `*`, `?`, `[abc]`, `[a-z]`, `**` is **not** supported by stdlib — write recursive walks instead, or use `doublestar` (third-party).

### Path joining and security

```go
// NOT SAFE: ".." can escape
fullPath := filepath.Join(base, untrusted)

// Better (1.24+):
root, _ := os.OpenRoot(base)
f, err := root.Open(untrusted)
```

`filepath.Join` collapses `..` but symlinks pointing outside the base directory still escape. See `08-stdlib/04-os-and-exec.md` and security notes below.

### Cross-platform pitfalls

- Windows paths: `\`, drive letters, UNC paths `\\server\share`.
- Reserved Windows names: `CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9`, `LPT1`-`LPT9` — `filepath.Clean` does not reject these.
- Case sensitivity: macOS APFS is usually case-insensitive; ext4 is case-sensitive; NTFS is case-insensitive but case-preserving. Don't compare paths with `==` after user input.
- Max path length: 260 on Windows (without long-path mode), longer on Unix.

## Standard Library Hooks

- `io/fs.WalkDir`, `fs.Glob` — virtual-FS versions for `fs.FS`.
- `os.Root` (1.24+) — sandboxed alternative to `filepath.Join`.
- `net/url` — `url.URL.EscapedPath` for URL components.
- `archive/zip` — file names inside a zip use `/` (path, not filepath).

## Real-World Patterns

### 1. Find all matching files under a tree

```go
import (
	"io/fs"
	"path/filepath"
	"strings"
)

func findLogs(root string) ([]string, error) {
	var out []string
	err := filepath.WalkDir(root, func(p string, d fs.DirEntry, err error) error {
		if err != nil { return err }
		if d.IsDir() { return nil }
		if strings.HasSuffix(d.Name(), ".log") {
			out = append(out, p)
		}
		return nil
	})
	return out, err
}
```

Use case: log aggregators, build tools.

### 2. Normalize user-supplied output path

```go
import "path/filepath"

func resolveOut(input string) (string, error) {
	abs, err := filepath.Abs(input)
	if err != nil { return "", err }
	return filepath.Clean(abs), nil
}
```

Use case: CLI flags like `-o`.

### 3. Build URLs with `path`, not `filepath`

```go
import "path"

func apiURL(base, resource, id string) string {
	return base + path.Join("/v1", resource, id)
}
```

`path` works regardless of OS; `filepath` would corrupt URLs on Windows.

### 4. Cross-tree relative paths

```go
rel, _ := filepath.Rel("/a/b", "/a/b/c/d.txt") // c/d.txt
```

Use case: producing portable manifest entries from absolute scan paths.

### 5. Glob a directory then sort by mtime

```go
matches, _ := filepath.Glob("/var/log/app.*.log")
sort.Slice(matches, func(i, j int) bool {
	fi, _ := os.Stat(matches[i])
	fj, _ := os.Stat(matches[j])
	return fi.ModTime().After(fj.ModTime())
})
```

Use case: rotating log selection.

## Anti-Patterns & Gotchas

**Using `filepath.Join(base, untrusted)` as security.** Not enough. Use `os.Root`.

**Using `path` for OS paths on Windows.** Produces `/` mixed with `\`; breaks.

**Using `filepath` for URL paths.** Breaks on Windows.

**`filepath.Walk` on huge trees** when `WalkDir` would do.

**Returning `filepath.SkipAll` from `filepath.Walk` (not `WalkDir`).** Old `Walk` doesn't know it; use `WalkDir`.

**Comparing paths with `==`.** Case differences, trailing slashes, symlinks. Normalize first.

**Resolving symlinks for security decisions then opening the result.** TOCTOU race: the link can change between check and open. Use `os.Root` instead.

**Forgetting that `filepath.Glob` is OS-native** — `\\` patterns on Windows are weird.

## Performance Notes

- `filepath.WalkDir` is much faster than `filepath.Walk` (no per-entry `Lstat`).
- `filepath.Clean` is cheap (string manipulation only).
- `filepath.EvalSymlinks` walks every component and stats each — slow on deep trees.
- `filepath.Glob` runs in user space after listing directories with `Readdirnames` — no shell involved.

## How Big Companies Use It

- **`go` toolchain** uses `path` for module paths, `filepath` for on-disk build cache.
- **Hugo / static site generators** walk source trees with `WalkDir`, then use `path` (not `filepath`) to build URLs.
- **Docker** uses `filepath.EvalSymlinks` for container rootfs handling — historically a source of CVEs (CVE-2019-5736).
- **Kubernetes** uses `filepath.Walk` for cert directories; recent code migrated to `WalkDir`.

## Source Code References

Pinned to `go1.26`.

- `path`: [`src/path/path.go`](https://github.com/golang/go/blob/master/src/path/path.go), `match.go`.
- `filepath`: [`src/path/filepath/path.go`](https://github.com/golang/go/blob/master/src/path/filepath/path.go), `walk.go`, `match.go`.
- OS-specific separators: [`src/path/filepath/path_unix.go`](https://github.com/golang/go/blob/master/src/path/filepath/path_unix.go), [`path_windows.go`](https://github.com/golang/go/blob/master/src/path/filepath/path_windows.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/path, https://pkg.go.dev/path/filepath.
- "Go's path/filepath is not for security" (community write-ups around 2022 CVEs).
- Russ Cox on `os.Root` design: https://github.com/golang/go/issues/49580.

## Exercises / Self-Check

1. Walk a tree with `filepath.WalkDir` and skip any directory named `.git`. Use `filepath.SkipDir`.
2. Why does `path.Clean("//a/b//c")` return `/a/b/c`? Trace through.
3. Demonstrate the symlink-escape problem with `filepath.Join`. Then redo with `os.Root` to show the difference.
4. Write a small glob that matches `**/*.go` (recursive). Compare your code with the `doublestar` third-party library.
5. On Windows, what does `filepath.Join("C:", "a", "b")` return? On Unix?
