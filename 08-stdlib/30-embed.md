# `embed` — Embed Files in the Binary

## TL;DR

`//go:embed` (since 1.16) inlines files or directories from your module into the compiled binary as `string`, `[]byte`, or `embed.FS`. The `embed.FS` form implements `fs.FS`, so embedded assets can be served by `http.FS`, walked by `fs.WalkDir`, and tested swappable with `fstest.MapFS`. Ship a single self-contained binary with templates, migrations, static assets, and config defaults baked in.

## Mental Model

```
//go:embed pattern
var name TYPE     // TYPE = string | []byte | embed.FS

string / []byte → single file
embed.FS        → one or more files/dirs as fs.FS

The compiler resolves patterns at build time; runtime is just a sorted blob.
```

## Syntax & Basic Usage

```go
package main

import (
	"embed"
	"fmt"
	"io/fs"
)

//go:embed VERSION
var version string

//go:embed banner.txt
var banner []byte

//go:embed assets/*
var assets embed.FS

func main() {
	fmt.Print(version)
	fmt.Print(string(banner))
	fs.WalkDir(assets, "assets", func(p string, d fs.DirEntry, err error) error {
		fmt.Println(p)
		return nil
	})
}
```

## Deep Dive

### Pattern syntax

```go
//go:embed file.txt              // single file
//go:embed dir                   // a directory (recursive)
//go:embed dir/*                 // files in dir (non-recursive)
//go:embed dir/*.html dir/*.css  // multiple patterns
//go:embed all:dir               // include dot-prefixed files (since 1.18)
```

By default, files starting with `.` or `_` are NOT included. `all:` prefix overrides.

### Patterns are relative to the source file

```
mypkg/
  main.go            // //go:embed templates/*.html — relative to main.go
  templates/
    index.html
```

You cannot embed files outside the module.

### `embed.FS` is `fs.FS`

```go
var assets embed.FS

// Treat as fs.FS:
http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(assets))))

// Read a file:
data, err := assets.ReadFile("assets/logo.png")

// Walk:
fs.WalkDir(assets, ".", visit)

// Sub-FS:
sub, _ := fs.Sub(assets, "templates")
template.ParseFS(sub, "*.html")
```

### Comment placement matters

The `//go:embed` directive must directly precede a `var` declaration:

```go
//go:embed file.txt
var s string  // OK

// BAD: separated by other comments or code
//go:embed file.txt

// some comment
var s string  // compile error
```

### File size in the binary

Embedded files inflate binary size 1:1 with the file content. For very large blobs (binaries, big datasets), consider an external file or a content-addressable store.

### Mutability

`embed.FS` is read-only. The underlying data lives in the binary's read-only data section.

### Combining with `html/template`

```go
//go:embed templates/*.html
var tmplFS embed.FS

var tpl = template.Must(template.ParseFS(tmplFS, "templates/*.html"))
```

Use case: ship templates with the binary; no deploy step beyond `scp`.

### Combining with `database/sql` migrations

```go
//go:embed migrations/*.sql
var migrations embed.FS

// Pass to a migration library that accepts fs.FS.
```

## Standard Library Hooks

- `io/fs.FS`, `fs.File`, `fs.DirEntry`, `fs.WalkDir`.
- `net/http.FS(fsys)` wraps `fs.FS` as `http.FileSystem`.
- `text/template.ParseFS`, `html/template.ParseFS`.
- `testing/fstest.MapFS` for fakes.

## Real-World Patterns

### 1. Static asset server

```go
//go:embed assets/*
var assets embed.FS

func main() {
	mux := http.NewServeMux()
	mux.Handle("/static/", http.StripPrefix("/", http.FileServer(http.FS(assets))))
	http.ListenAndServe(":8080", mux)
}
```

### 2. HTML templates + static + favicon

```go
//go:embed templates/*.html
//go:embed static
//go:embed favicon.ico
var content embed.FS

tpl := template.Must(template.ParseFS(content, "templates/*.html"))
mux.Handle("/static/", http.StripPrefix("/", http.FileServer(http.FS(content))))
mux.HandleFunc("/favicon.ico", func(w http.ResponseWriter, r *http.Request) {
	data, _ := content.ReadFile("favicon.ico")
	w.Write(data)
})
```

### 3. SQL migrations bundled with binary

```go
//go:embed db/migrations/*.sql
var migrationsFS embed.FS

func migrate(db *sql.DB) error {
	entries, _ := fs.ReadDir(migrationsFS, "db/migrations")
	for _, e := range entries {
		data, _ := fs.ReadFile(migrationsFS, "db/migrations/"+e.Name())
		if _, err := db.Exec(string(data)); err != nil { return err }
	}
	return nil
}
```

### 4. Default config

```go
//go:embed defaults.yaml
var defaultConfig []byte

func loadConfig(path string) (*Config, error) {
	var c Config
	if err := yaml.Unmarshal(defaultConfig, &c); err != nil { return nil, err }
	if path != "" {
		userData, err := os.ReadFile(path)
		if err == nil { yaml.Unmarshal(userData, &c) }
	}
	return &c, nil
}
```

### 5. Testable embedded FS

```go
type App struct { Assets fs.FS }

// Production:
//go:embed assets/*
var prodAssets embed.FS
app := &App{Assets: prodAssets}

// Tests:
testFS := fstest.MapFS{"assets/x": {Data: []byte("hi")}}
app := &App{Assets: testFS}
```

## Anti-Patterns & Gotchas

**Embedding gigabytes of data.** Slow link, slow startup load into memory.

**Pattern outside the module.** Not allowed.

**Pattern that doesn't match anything.** Compile error.

**Using `//go:embed` comment with blank lines before var.** Must be directly attached.

**Modifying `embed.FS`.** Read-only; will panic on attempted writes (or fail to compile).

**Forgetting `all:` for dotfiles.** `.gitignore`-style files won't embed by default.

**Hot-reload expectations.** Embedded files are baked at build time — no hot-reload.

**Adding `//go:embed` to test files.** Works but tests live alongside source; consider whether you want test fixtures embedded vs read live.

## Performance Notes

- Lookup in `embed.FS` is binary-search over a sorted file table: O(log N).
- File reads are zero-copy slices into the read-only data section.
- Binary size grows by the size of embedded content (plus tiny per-file metadata).
- Startup overhead is negligible; `embed.FS` is essentially a pre-built sorted table.

## How Big Companies Use It

- **Caddy** embeds the default `Caddyfile` and HTML help pages.
- **Hugo** ships theme templates as a tree of files; many bundles use `embed`.
- **`go` toolchain** uses `embed` for some helper assets (the `vet` analyzer registry, etc.).
- **Tailscale `tailscale` CLI** embeds its admin UI as `embed.FS` served at `localhost:8088`.

## Source Code References

Pinned to `go1.26`.

- `embed`: [`src/embed/embed.go`](https://github.com/golang/go/blob/master/src/embed/embed.go).
- `//go:embed` directive handling in compiler: [`src/cmd/compile/internal/noder/`](https://github.com/golang/go/tree/master/src/cmd/compile/internal/noder).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/embed.
- Go 1.16 release notes — embed: https://go.dev/doc/go1.16#embed.
- Carlos Becker, "Embedding files in Go": various blog posts.

## Exercises / Self-Check

1. Embed an HTML template tree and parse it with `html/template.ParseFS`.
2. Build a CLI that serves embedded static files via `http.FS`. Show that no external files are needed at runtime.
3. Run SQL migrations from `embed.FS`. Order by filename.
4. Test code that takes `fs.FS`: use `fstest.MapFS` to supply fixtures.
5. Why does `//go:embed .config` fail by default? Use `all:.config` and see it succeed.
