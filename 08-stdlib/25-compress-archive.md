# `compress/*` and `archive/*` — Compression and Archives

## TL;DR

`compress/gzip`, `compress/zlib`, `compress/flate`, `compress/bzip2`, `compress/lzw` provide `io.Reader`/`io.Writer` wrappers — stack them to compress/decompress streams. `archive/tar` and `archive/zip` read and write archive formats. Combine them: gzip-compressed tar files (`.tar.gz`) are a `tar.NewReader(gzip.NewReader(file))` pipeline.

## Mental Model

```
Stream of bytes
   │
   ▼
gzip.Reader / zlib.Reader / bzip2.NewReader   ← wrap to decompress
   │
   ▼
tar.NewReader  → iterate Header + body         ← unbundle archive

Writing reverses:

bytes → tar.Writer → gzip.Writer → file
```

## Syntax & Basic Usage

```go
package main

import (
	"archive/tar"
	"compress/gzip"
	"fmt"
	"io"
	"os"
)

func main() {
	f, _ := os.Open("backup.tar.gz")
	defer f.Close()
	gz, _ := gzip.NewReader(f); defer gz.Close()
	tr := tar.NewReader(gz)
	for {
		h, err := tr.Next()
		if err == io.EOF { break }
		if err != nil { panic(err) }
		fmt.Println(h.Name, h.Size)
		io.Copy(io.Discard, tr) // skip file contents
	}
}
```

## Deep Dive

### `compress/gzip`

```go
gz, _ := gzip.NewWriter(dst)
gz.ModTime = time.Now()
gz.Name = "data.bin"
gz.Write(payload)
gz.Close() // flushes trailer

// Read:
gz, _ := gzip.NewReader(src); defer gz.Close()
io.Copy(out, gz)
```

`gzip.Writer` supports `Reset(w)` to reuse with a different destination — useful in pools.

Levels: `gzip.NoCompression`, `BestSpeed`, `BestCompression`, `DefaultCompression` (-1, ≈ level 6).

### `compress/zlib` and `compress/flate`

Same shape; different framing. `flate` is raw DEFLATE; `zlib` adds a small header/checksum (used inside PNG, PDF). Most user-facing compression is `gzip`.

### `compress/bzip2`

Decompression only. No encoder in stdlib. For bzip2 compression, use third-party.

### `compress/lzw`

Lempel-Ziv-Welch. Used in GIF and some PDF streams. Rarely needed standalone.

### `archive/tar`

```go
// Write:
tw := tar.NewWriter(dst)
for _, file := range files {
	h := &tar.Header{
		Name: file.Path,
		Mode: 0644,
		Size: int64(len(file.Data)),
	}
	tw.WriteHeader(h)
	tw.Write(file.Data)
}
tw.Close()

// Read:
tr := tar.NewReader(src)
for {
	h, err := tr.Next()
	if err == io.EOF { break }
	if err != nil { return err }
	switch h.Typeflag {
	case tar.TypeReg:
		io.Copy(out, tr)
	case tar.TypeDir:
		os.MkdirAll(h.Name, 0755)
	case tar.TypeSymlink:
		os.Symlink(h.Linkname, h.Name)
	}
}
```

Support for various tar formats (USTAR, PAX, GNU) — stdlib handles all reading; writes use PAX by default.

### `archive/zip`

```go
// Write:
zw := zip.NewWriter(dst)
for _, f := range files {
	w, _ := zw.Create(f.Name)
	w.Write(f.Data)
}
zw.Close()

// Read:
r, _ := zip.OpenReader("archive.zip"); defer r.Close()
for _, f := range r.File {
	rc, _ := f.Open()
	io.Copy(out, rc)
	rc.Close()
}
```

Zip files have a central directory at the end, so reading random files is fast.

### `zip.Reader` as `fs.FS`

```go
r, _ := zip.OpenReader("data.zip")
defer r.Close()
fs.WalkDir(r, ".", func(p string, d fs.DirEntry, err error) error {
	fmt.Println(p)
	return nil
})
```

Use case: read embedded zip as if it were a real filesystem.

### Path safety — zip slip

```go
// VULNERABLE:
out, _ := os.Create(filepath.Join(dest, h.Name)) // h.Name could be "../../etc/passwd"

// SAFE (1.24+):
root, _ := os.OpenRoot(dest)
out, _ := root.Create(h.Name)
```

Always sanitize archive entry names against `..` and absolute paths.

## Standard Library Hooks

- `io.Reader`/`io.Writer` for stream chaining.
- `archive/zip.Reader` implements `fs.FS`.
- `crypto/sha256` etc. via `io.TeeReader` for hashing during compression.

## Real-World Patterns

### 1. Stream-compress a file

```go
func gzipFile(src, dst string) error {
	in, err := os.Open(src); if err != nil { return err }
	defer in.Close()
	out, err := os.Create(dst); if err != nil { return err }
	defer out.Close()
	gz := gzip.NewWriter(out)
	defer gz.Close()
	_, err = io.Copy(gz, in)
	return err
}
```

### 2. Tar.gz a directory

```go
func tarGz(root, dst string) error {
	out, _ := os.Create(dst); defer out.Close()
	gz := gzip.NewWriter(out); defer gz.Close()
	tw := tar.NewWriter(gz); defer tw.Close()
	return filepath.WalkDir(root, func(p string, d fs.DirEntry, err error) error {
		if err != nil { return err }
		fi, _ := d.Info()
		hdr, err := tar.FileInfoHeader(fi, "")
		if err != nil { return err }
		rel, _ := filepath.Rel(root, p)
		hdr.Name = rel
		if err := tw.WriteHeader(hdr); err != nil { return err }
		if d.IsDir() { return nil }
		f, _ := os.Open(p); defer f.Close()
		_, err = io.Copy(tw, f)
		return err
	})
}
```

Use case: backup tools, container image building.

### 3. HTTP response decompression (manual)

```go
resp, _ := client.Get(url)
defer resp.Body.Close()
var r io.Reader = resp.Body
if resp.Header.Get("Content-Encoding") == "gzip" {
	gz, _ := gzip.NewReader(resp.Body)
	defer gz.Close()
	r = gz
}
// process r
```

`net/http` already does this when not set to `DisableCompression`.

### 4. Zip extraction with path safety

```go
func unzip(src, dest string) error {
	r, err := zip.OpenReader(src); if err != nil { return err }
	defer r.Close()
	root, err := os.OpenRoot(dest); if err != nil { return err }
	defer root.Close()
	for _, f := range r.File {
		if f.FileInfo().IsDir() {
			root.Mkdir(f.Name, 0755); continue
		}
		out, err := root.Create(f.Name)
		if err != nil { return err }
		in, err := f.Open()
		if err != nil { out.Close(); return err }
		_, err = io.Copy(out, in)
		in.Close(); out.Close()
		if err != nil { return err }
	}
	return nil
}
```

### 5. Pooled gzip writers for high throughput

```go
var gzPool = sync.Pool{New: func() any { return gzip.NewWriter(nil) }}

func compress(dst io.Writer, src io.Reader) error {
	gz := gzPool.Get().(*gzip.Writer)
	defer gzPool.Put(gz)
	gz.Reset(dst)
	defer gz.Close()
	_, err := io.Copy(gz, src)
	return err
}
```

Use case: per-request response compression in HTTP servers.

## Anti-Patterns & Gotchas

**Forgetting `gzip.Writer.Close()`.** Trailer not written; reader chokes.

**Tar extraction without path sanitization.** Zip-slip vulnerability.

**`zip.OpenReader` (not `NewReader`) on an http.Response.Body.** OpenReader needs `*os.File`. Use `zip.NewReader(rdr, size)` after reading into a buffer or a seeker.

**Reusing `gzip.Writer` across goroutines.** Not safe; use sync.Pool with one per goroutine.

**Decompressing untrusted data without size limits.** Zip bombs.

**Forgetting `defer r.Close()` on `gzip.NewReader`.** Buffers retained.

**Using `compress/lzw` for new code.** Almost always wrong; use gzip.

## Performance Notes

- gzip throughput: ~150 MB/s decompress, ~50 MB/s compress (level 6). Hardware-accelerated alternatives like `klauspost/compress` are 3-5× faster.
- tar overhead is negligible — it's mostly headers.
- zip uses DEFLATE per entry; opening a single file from a large zip is fast thanks to the central directory.
- Reuse `gzip.Writer` via `Reset` to avoid allocating per use.

## How Big Companies Use It

- **containerd / Docker** uses `archive/tar` for image layers.
- **Go module proxy** serves `.zip` archives of module versions.
- **HashiCorp `gomplate`** uses gzip for compressed templates.
- **Hugo** uses `archive/zip` for `hugo new theme` bundles.

## Source Code References

Pinned to `go1.26`.

- `compress/gzip`: [`src/compress/gzip/`](https://github.com/golang/go/tree/master/src/compress/gzip).
- `compress/flate`: [`src/compress/flate/`](https://github.com/golang/go/tree/master/src/compress/flate).
- `archive/tar`: [`src/archive/tar/`](https://github.com/golang/go/tree/master/src/archive/tar).
- `archive/zip`: [`src/archive/zip/`](https://github.com/golang/go/tree/master/src/archive/zip).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/compress/gzip, /archive/tar, /archive/zip.
- klauspost/compress: https://github.com/klauspost/compress — faster gzip/zstd/snappy.
- "Zip Slip Vulnerability": https://snyk.io/research/zip-slip-vulnerability.

## Exercises / Self-Check

1. Compress a 1 GB file with gzip levels 1, 6, 9. Compare time vs ratio.
2. Build a tar.gz of a directory; extract elsewhere; verify checksum.
3. Implement safe zip extraction using `os.Root`.
4. Set up a sync.Pool of gzip writers; benchmark HTTP gzip vs naive.
5. Write a function to inspect a `.tar.gz` and list entries without extracting.
