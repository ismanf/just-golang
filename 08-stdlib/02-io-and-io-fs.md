# `io` and `io/fs` — Streams and Filesystem Abstractions

## TL;DR

`io` defines the small interfaces that every byte stream in Go implements: `Reader`, `Writer`, `Closer`, `Seeker`, `ReaderAt`, `WriterAt`, and combinations. `io/fs` (since 1.16) generalizes file access into `fs.FS`, `fs.File`, `fs.DirEntry` — so the same code reads from disk, from `//go:embed` bundles, from zip archives, or from in-memory test fakes. Build your APIs around these interfaces, not around `*os.File`, and your code becomes testable, composable, and source-agnostic.

## Mental Model

```
io.Reader: Read(p []byte) (n int, err error)
                          │
              fills p with up to len(p) bytes;
              returns n>0 with err possibly non-nil (incl. io.EOF on last chunk)

io.Writer: Write(p []byte) (n int, err error)
                          │
              writes all of p; returns n<len(p) ONLY with err != nil

Stream composition:

    file → bufio.Reader → gzip.Reader → tar.Reader → your parser
    └───────────────  io.Reader  ──────────────────┘

fs.FS abstracts the root:

    os.DirFS("/var/data") → real filesystem rooted at /var/data
    embed.FS              → compiled-in files
    fstest.MapFS          → in-memory fake for tests
    zip.Reader (as FS)    → contents of a zip
```

The contract is small; the ecosystem is huge because of it.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("hello, world")
	b, err := io.ReadAll(r)
	if err != nil { panic(err) }
	fmt.Println(string(b))
	// Output:
	// hello, world
}
```

```go
// Copying any Reader to any Writer
io.Copy(dst, src)         // typical disk-to-disk, http-to-disk
io.CopyN(dst, src, 1024)  // first N bytes only
```

## Deep Dive

### `Read` contract

A `Read` call:

- May return `0 <= n <= len(p)` AND a nil error → caller should call again.
- May return `n > 0` AND `io.EOF` → process the n bytes; stream is done.
- May return `n == 0` AND `io.EOF` → done, no more data.
- Should never return `n == 0` AND `nil` (would cause infinite loop in some helpers).

Helpers like `io.ReadFull` and `io.ReadAll` handle the loop for you. Roll your own only when you understand the contract.

### `Write` contract

`Write(p)` must write all of `p` or return an error. `n < len(p)` always implies `err != nil`. Implementing a `Writer` that short-writes without an error is a bug. `io.ErrShortWrite` is the sentinel for that case (returned by `io.Copy` when it detects a buggy writer).

### Common combinator interfaces

```go
io.ReadCloser   = Reader + Closer        // http.Response.Body
io.WriteCloser  = Writer + Closer        // gzip.NewWriter result
io.ReadWriter   = Reader + Writer        // net.Conn (basically)
io.ReadSeeker   = Reader + Seeker        // *os.File for read-only files
io.ReaderAt     = pread() — random read at offset
io.WriterAt     = pwrite() — random write at offset
```

### `io` helpers worth memorizing

- `io.Copy(dst, src)` — pump until EOF; returns bytes copied.
- `io.CopyBuffer(dst, src, buf)` — bring-your-own-buffer variant; avoids allocation.
- `io.ReadAll(r)` — into a `[]byte`; allocates as it goes.
- `io.ReadFull(r, buf)` — fill the buffer fully or error.
- `io.LimitReader(r, n)` — wraps `r` to stop after n bytes (protects against unbounded reads).
- `io.MultiReader(r1, r2, ...)` — concatenates streams.
- `io.TeeReader(r, w)` — copy reads into `w` as a side effect (useful for hashing/auditing).
- `io.Pipe()` — in-memory synchronous Reader/Writer pair.
- `io.NopCloser(r)` — wrap a `Reader` to look like `ReadCloser`.
- `io.Discard` — `/dev/null` as a `Writer`.

### `fs.FS` (since 1.16)

```go
type FS interface { Open(name string) (File, error) }
```

Implementations:

- `os.DirFS(root)` — files under `root` on the real filesystem.
- `embed.FS` — files embedded at compile time via `//go:embed`.
- `fstest.MapFS` — in-memory map of names to file content (test fake).
- `archive/zip.Reader` — files within a zip archive.

Code that uses `fs.FS` works against all of them unchanged.

### `io/fs` helpers

- `fs.ReadFile(fsys, name)` — full file contents.
- `fs.ReadDir(fsys, dir)` — directory listing as `[]fs.DirEntry`.
- `fs.WalkDir(fsys, root, fn)` — recursive walk; preferred over `filepath.Walk` since 1.16 (returns `DirEntry`, avoids `os.Lstat` per file).
- `fs.Glob(fsys, pattern)` — `*.go` style matching.
- `fs.Stat(fsys, name)` — metadata.
- `fs.Sub(fsys, dir)` — subtree as its own `fs.FS`.

### `os.Root` (since 1.24) — sandboxed FS access

```go
root, err := os.OpenRoot("/var/data")
defer root.Close()
f, err := root.Open("user/cache.bin") // refuses paths that escape /var/data
```

A safer alternative to `filepath.Join` + `os.Open` when handling untrusted paths. See `08-stdlib/04-os-and-exec.md`.

### Why streaming matters

In-memory:

```go
b, _ := os.ReadFile("huge.json")
process(b) // OOM for huge files
```

Streaming:

```go
f, _ := os.Open("huge.json")
defer f.Close()
dec := json.NewDecoder(f) // reads in chunks
for dec.More() { dec.Decode(&item); handle(item) }
```

The interface lets you decode a 50 GB file with constant memory.

## Standard Library Hooks

- `os.File` implements `io.Reader`, `Writer`, `Closer`, `Seeker`, `ReaderAt`, `WriterAt`, and `fs.File`.
- `net.Conn` is `io.ReadWriteCloser`.
- `http.Request.Body` is `io.ReadCloser`.
- `bytes.Buffer`, `bytes.Reader`, `strings.Builder`, `strings.Reader` — in-memory io.
- `bufio.Reader` / `bufio.Writer` — buffering wrappers (see `08-stdlib/03-bufio.md`).
- `compress/gzip`, `compress/flate`, `compress/zlib` — `Reader`/`Writer` wrappers.
- `crypto/sha256`, `crypto/hmac` — Hash implements `io.Writer`; feed bytes by writing.
- `encoding/json`, `encoding/xml` — `NewEncoder(w)` / `NewDecoder(r)`.
- `archive/tar`, `archive/zip` — stream-oriented archive readers.
- `text/template`, `html/template` — `Execute(w, data)` writes to `io.Writer`.

## Real-World Patterns

### 1. Pipe-and-process for compression on the fly

```go
import (
	"compress/gzip"
	"io"
	"os"
)

func compressTo(dst io.Writer, src io.Reader) error {
	gz := gzip.NewWriter(dst)
	defer gz.Close()
	_, err := io.Copy(gz, src)
	return err
}

// Usage:
in, _ := os.Open("data.json")
defer in.Close()
out, _ := os.Create("data.json.gz")
defer out.Close()
compressTo(out, in)
```

Use case: log shippers that compress before uploading.

### 2. Tee for hashing while transferring

```go
import (
	"crypto/sha256"
	"io"
	"os"
)

func saveAndHash(path string, src io.Reader) (string, error) {
	f, _ := os.Create(path)
	defer f.Close()
	h := sha256.New()
	if _, err := io.Copy(f, io.TeeReader(src, h)); err != nil {
		return "", err
	}
	return fmt.Sprintf("%x", h.Sum(nil)), nil
}
```

Use case: object-storage uploads that need a checksum without re-reading the file.

### 3. `embed.FS` for assets, tested with `fstest.MapFS`

```go
//go:embed templates/*.html
var tmplFS embed.FS

func render(w io.Writer, name string, data any) error {
	tmpl, err := template.ParseFS(tmplFS, "templates/"+name)
	if err != nil { return err }
	return tmpl.Execute(w, data)
}

// In tests, swap the FS:
testFS := fstest.MapFS{"templates/page.html": {Data: []byte("Hi {{.}}")}}
```

Use case: HTTP servers shipping a single self-contained binary that nonetheless can be tested with synthetic templates.

### 4. `LimitReader` to protect against malicious uploads

```go
const maxUpload = 10 << 20 // 10 MiB
body := io.LimitReader(r.Body, maxUpload+1)
b, err := io.ReadAll(body)
if err != nil { return err }
if len(b) > maxUpload {
	http.Error(w, "too large", http.StatusRequestEntityTooLarge)
	return
}
```

Use case: HTTP upload endpoints.

### 5. `io.Pipe` to bridge a producer and consumer

```go
pr, pw := io.Pipe()
go func() {
	defer pw.Close()
	_ = json.NewEncoder(pw).Encode(largeObject)
}()

req, _ := http.NewRequest("POST", url, pr)
http.DefaultClient.Do(req) // streams the JSON without buffering it all
```

Use case: streaming a large encoded object as an HTTP request body.

## Anti-Patterns & Gotchas

**Calling `Read` and assuming `n == len(p)`.** Short reads are legal. Use `io.ReadFull`.

**Forgetting `defer Body.Close()`.** Leaks the connection. (See `08-stdlib/17-net-http-client.md`.)

**`io.ReadAll` on an unknown-size stream.** Can OOM. Wrap with `io.LimitReader`.

**Writing to a `bytes.Buffer` and *then* writing the buffer to disk** when streaming would do. Don't intermediate-buffer when you can pipe.

**Calling `Close` twice.** Often (not always) safe, but contract is undefined. Use `sync.Once` if necessary.

**Treating `io.EOF` as an error to log.** It's the normal end-of-stream signal.

**Returning `*os.File` from public APIs.** Breaks substitutability. Return `io.ReadCloser` or accept `io.Reader`.

**`io.Copy(dst, src)` to a `bytes.Buffer` to convert to string** when `strings.Builder` or `io.ReadAll` would do.

**Walking `filepath.Walk` for a tree under a virtual FS.** Use `fs.WalkDir`.

## Performance Notes

- `io.Copy` uses `WriterTo`/`ReaderFrom` shortcuts if either side implements them — e.g., `*os.File.ReadFrom` triggers `splice(2)` / `sendfile(2)` on Linux for zero-copy file-to-socket.
- Default buffer size in `io.Copy` is 32 KiB when neither shortcut applies. Override with `io.CopyBuffer`.
- `io.ReadAll` doubles its buffer; final size = next power of 2 above the actual size.
- `io.Pipe` is synchronous: a write blocks until a corresponding read consumes. Useful as a backpressure mechanism.
- `embed.FS` reads are zero-allocation lookups in a sorted file table.
- `fs.WalkDir` is faster than the old `filepath.Walk` because it uses `DirEntry` (no `os.Lstat` per file).

## How Big Companies Use It

- **Caddy** uses `fs.FS` so the same handler serves from disk, from an embedded distribution, or from a custom backend.
- **`go` toolchain** uses `embed.FS` for stdlib documentation and resources shipped with the compiler.
- **Tailscale** wraps connections in `io.ReadWriter` chains so the same wire-level code works over TCP, WireGuard, and DERP relays.
- **MinIO** uses `io.Reader` chains throughout its object pipeline; checksums, encryption, erasure-coding are all readers stacked on each other.
- **HashiCorp Consul** uses `fstest.MapFS` heavily in unit tests for configuration-loader code.

## Source Code References

Pinned to `go1.26`.

- `io` interfaces & helpers: [`src/io/io.go`](https://github.com/golang/go/blob/master/src/io/io.go).
- `io/fs` types: [`src/io/fs/fs.go`](https://github.com/golang/go/blob/master/src/io/fs/fs.go).
- `os.DirFS` and `os.Root`: [`src/os/file.go`](https://github.com/golang/go/blob/master/src/os/file.go).
- `embed`: [`src/embed/embed.go`](https://github.com/golang/go/blob/master/src/embed/embed.go).
- `io.Copy` fast-path detection: [`src/io/io.go`](https://github.com/golang/go/blob/master/src/io/io.go) — function `copyBuffer`.
- `splice(2)` / `sendfile(2)` shortcut: [`src/internal/poll/splice_linux.go`](https://github.com/golang/go/blob/master/src/internal/poll/splice_linux.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/io, https://pkg.go.dev/io/fs.
- Go blog, "Package io: introducing fs.FS": https://go.dev/blog/io-fs.
- Effective Go, "Interfaces and methods" (the `io.Writer`/`Reader` example).
- "The 1.16 io/fs design" proposal: https://github.com/golang/go/issues/41190.

## Exercises / Self-Check

1. Implement an `io.Reader` that yields the first N bytes of an underlying reader and then stops with `io.EOF`. Compare with `io.LimitReader`.
2. Write a `Writer` that counts bytes written but discards them (like `io.Discard` + counter). Use it to measure the size of any `WriterTo`.
3. Build a small `fs.FS` over an in-memory map. Run `fs.WalkDir` over it.
4. Pipe a JSON encoder into an HTTP request body via `io.Pipe`. Show that memory usage stays low when the encoded object is large.
5. Trace what happens under the hood when `io.Copy(file, file)` triggers the `splice` path on Linux.
