# `bufio` — Buffered I/O and Scanning

## TL;DR

`bufio` wraps any `io.Reader`/`Writer` with a memory buffer so reads/writes happen in bulk rather than one syscall per byte. `bufio.Scanner` is the convenience for line-by-line / token-by-token input; its single biggest gotcha is `bufio.MaxScanTokenSize` (64 KiB by default), which silently fails on long lines. Always pair scanners with `Buffer()` when input lines can be large, and prefer `bufio.Reader.ReadString('\n')` if you need to control max line length explicitly.

## Mental Model

```
unbuffered Reader: 1M tiny Read calls = 1M syscalls = slow
                            │
                            ▼
bufio.Reader:       1 syscall fills 4 KiB; ReadByte/ReadRune/ReadString served from RAM

bufio.Scanner:      Reader → split function → tokens
                                  │
                                  ├─ ScanLines (default)
                                  ├─ ScanWords
                                  ├─ ScanRunes
                                  ├─ ScanBytes
                                  └─ your custom SplitFunc
```

## Syntax & Basic Usage

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func main() {
	r := strings.NewReader("alpha\nbeta\ngamma\n")
	sc := bufio.NewScanner(r)
	for sc.Scan() {
		fmt.Println(sc.Text())
	}
	if err := sc.Err(); err != nil {
		fmt.Println("error:", err)
	}
	// Output:
	// alpha
	// beta
	// gamma
}
```

## Deep Dive

### `bufio.Reader`

```go
br := bufio.NewReader(f)             // default 4096-byte buffer
br := bufio.NewReaderSize(f, 64<<10) // 64 KiB
```

API surface:

- `ReadByte() (byte, error)` — single byte.
- `ReadRune() (rune, size int, error)` — UTF-8-aware.
- `ReadString(delim byte) (string, error)` — read up to and including delim.
- `ReadBytes(delim)` — same, returns `[]byte`.
- `ReadLine()` — low-level, returns line without delim plus an `isPrefix` flag if the line exceeded the buffer (deprecated; prefer `Scanner` or `ReadString`).
- `Peek(n)` — look ahead without advancing.
- `UnreadByte()` / `UnreadRune()` — push back one unit.

### `bufio.Writer`

```go
bw := bufio.NewWriterSize(f, 1<<16)
fmt.Fprintln(bw, "line 1")
fmt.Fprintln(bw, "line 2")
bw.Flush() // ALWAYS flush before close
```

Forgetting `Flush` is the most common bug. `defer bw.Flush()` after creation is the safe pattern.

### `bufio.Scanner` — the killer convenience

Splits input into tokens. Default split: lines. Override with:

- `bufio.ScanLines` — strips trailing `\r\n` or `\n`.
- `bufio.ScanWords` — whitespace-separated tokens.
- `bufio.ScanRunes` — one rune per Scan.
- `bufio.ScanBytes` — one byte per Scan.
- `func(data []byte, atEOF bool) (advance int, token []byte, err error)` — custom.

```go
sc := bufio.NewScanner(r)
sc.Split(bufio.ScanWords)
for sc.Scan() {
	fmt.Println(sc.Text())
}
```

### `MaxScanTokenSize` — the famous trap

```go
const MaxScanTokenSize = 64 * 1024 // bufio package constant
```

If a single token exceeds 64 KiB, `Scan` returns `false` and `Err()` returns `bufio.ErrTooLong`. Files with long log lines (JSON-stringified payloads, base64 blobs) hit this *silently* unless you check `Err()`.

Fix:

```go
sc := bufio.NewScanner(f)
buf := make([]byte, 1<<20)
sc.Buffer(buf, 16<<20) // 1 MiB initial, up to 16 MiB
```

`Buffer(initial, max)` controls the scanner's internal buffer growth.

### Custom SplitFunc

```go
// Split on null bytes.
sc.Split(func(data []byte, atEOF bool) (int, []byte, error) {
	if i := bytes.IndexByte(data, 0); i >= 0 {
		return i + 1, data[:i], nil
	}
	if atEOF && len(data) > 0 {
		return len(data), data, nil
	}
	return 0, nil, nil
})
```

The signature is unusual; read `bufio.SplitFunc` docs carefully.

### Returned slices are reused

`sc.Bytes()` returns a slice into the scanner's internal buffer. The next `Scan()` overwrites it. If you need to keep the value, copy:

```go
keep := append([]byte(nil), sc.Bytes()...)
```

Same for `sc.Text()` — but `Text()` returns a `string` (a copy), so it's safe to retain.

### `bufio.Reader` vs `bufio.Scanner` — when to use which

| Use case | Pick |
|----------|------|
| Read all lines of a small file | Scanner |
| Read large/unbounded lines | Reader.ReadString / ReadBytes |
| Read structured binary stream | Reader.Peek + custom logic |
| Need backpressure / random access | Reader |
| Multi-line records (paragraphs) | Scanner with custom SplitFunc |
| Need to know whether line had trailing newline | Reader.ReadString (Scanner strips it) |

## Standard Library Hooks

- `io.Reader` / `io.Writer` — the underlying interfaces; `bufio` wraps them.
- `os.File` — the most common wrapped type.
- `net.Conn` — wrap with `bufio.Reader` for line-oriented protocols (HTTP, SMTP).
- `encoding/csv` uses `bufio.Reader` internally.
- `net/http` uses `bufio.Reader` to parse request lines and headers.
- `text/scanner` is a different package — for Go-syntax tokenization, not byte streams.

## Real-World Patterns

### 1. Reading a large log file by line, with long-line tolerance

```go
import (
	"bufio"
	"fmt"
	"os"
)

func tailLogs(path string) error {
	f, err := os.Open(path)
	if err != nil { return err }
	defer f.Close()

	sc := bufio.NewScanner(f)
	sc.Buffer(make([]byte, 1<<16), 1<<20) // 1 MiB max line
	for sc.Scan() {
		fmt.Println(sc.Text())
	}
	return sc.Err()
}
```

Use case: log analysis tools, CLI parsers.

### 2. Buffered writes for batch CSV output

```go
import (
	"bufio"
	"encoding/csv"
	"os"
)

func writeReport(rows [][]string) error {
	f, err := os.Create("report.csv")
	if err != nil { return err }
	defer f.Close()

	bw := bufio.NewWriter(f)
	defer bw.Flush()

	w := csv.NewWriter(bw)
	w.WriteAll(rows)
	w.Flush()
	return w.Error()
}
```

Use case: report generators that write millions of rows; without buffering, each `Write` becomes a syscall.

### 3. Parsing a line-prefix protocol (Redis-like)

```go
br := bufio.NewReader(conn)
line, err := br.ReadString('\n')
if err != nil { return err }
// Inspect prefix byte:
switch line[0] {
case '+': // simple string
case '-': // error
case ':': // integer
case '$': // bulk string with length
case '*': // array
}
```

Use case: implementing a Redis-protocol parser.

### 4. Custom SplitFunc for fixed-size record stream

```go
const recordSize = 128
sc := bufio.NewScanner(r)
sc.Buffer(make([]byte, recordSize*64), 1<<20)
sc.Split(func(data []byte, atEOF bool) (int, []byte, error) {
	if len(data) < recordSize {
		if atEOF { return len(data), data, nil }
		return 0, nil, nil
	}
	return recordSize, data[:recordSize], nil
})
```

Use case: binary log formats, fixed-width legacy file imports.

## Anti-Patterns & Gotchas

**Forgetting `Flush` on a `bufio.Writer`.** Data sits in RAM forever.

**Not checking `Scanner.Err()` after the loop.** Silent failure mode for `ErrTooLong`.

**Retaining `sc.Bytes()` past the next `Scan()`.** Use `sc.Text()` or copy.

**Using default `bufio.Scanner` for log files with long lines.** 64 KiB ceiling.

**Wrapping an `io.Reader` you've already read from with `bufio.Reader` mid-stream.** The buffered reader sees only future bytes; you can't recover what's already consumed.

**Wrapping a `bufio.Reader` in another `bufio.Reader`.** Pointless and wastes RAM.

**Calling `Reader.Read` directly when wrapped — bypassing the buffer.** The buffer only fills on the bufio methods. Mixing `br.Read(p)` and `br.ReadString` mostly works but is rarely what you intended.

**Closing the buffer instead of the underlying file.** `bufio.Writer` has no `Close` — it's not a Closer. Close the underlying `*os.File` after flushing.

**`bufio.NewReader(strings.NewReader(...))`.** Strings are already in memory — buffering adds nothing.

## Performance Notes

- Default buffer: 4 KiB. For high-throughput files, 64 KiB or 1 MiB pays off.
- A buffered write of small chunks → one syscall instead of N. On Linux, syscall overhead is ~hundreds of ns; saving 1M of them is meaningful.
- `bufio.Scanner` allocates one buffer per Scanner. Reuse the Scanner across calls if possible.
- `ReadString`/`ReadBytes` allocate the returned slice; for very high throughput, `Peek` + manual indexing avoids the alloc.
- Wrapping `os.Stdin` with `bufio.NewReader` once at program start beats interactive-style unbuffered reads.

## How Big Companies Use It

- **`grep`-like tools** (`ripgrep` is Rust, but Go equivalents like `ag` ports and `pt` use `bufio.Scanner` with raised buffer ceilings).
- **HashiCorp Vault** uses `bufio.Reader` for token-stream parsing in CLI commands.
- **Kubernetes `kubectl logs`** uses `bufio.Scanner` with explicit `Buffer` calls for very long container log lines.
- **etcd** uses `bufio.Writer` around its WAL log writer for batch durability writes (then `Flush` before fsync).

## Source Code References

Pinned to `go1.26`.

- `bufio.Reader`, `bufio.Writer`: [`src/bufio/bufio.go`](https://github.com/golang/go/blob/master/src/bufio/bufio.go).
- `bufio.Scanner`: [`src/bufio/scan.go`](https://github.com/golang/go/blob/master/src/bufio/scan.go).
- Default `MaxScanTokenSize`: same file, search the constant.
- `net/http` request parser using bufio.Reader: [`src/net/http/request.go`](https://github.com/golang/go/blob/master/src/net/http/request.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/bufio.
- Dave Cheney, "bufio.Scanner versus bufio.Reader": https://dave.cheney.net/2013/10/15.
- Go blog mentions on `Scanner`: https://go.dev/blog (search "Scanner").

## Exercises / Self-Check

1. Write a program that reads a 500 MB log file line by line. Time it with and without `bufio.Scanner`. Then add a 1 MB line — what happens by default? Add `Buffer()`.
2. Implement a `SplitFunc` that splits on `"---\n"` separators (YAML-document style).
3. Compare `bufio.Writer` vs unbuffered `*os.File` for writing 10M small lines. Plot the difference.
4. Why does `bufio.Reader.Peek(n)` sometimes return fewer than n bytes? When?
5. Build a Redis RESP parser using `bufio.Reader.ReadString('\n')`. Handle bulk strings and arrays.
