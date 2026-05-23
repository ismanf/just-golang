# `encoding/base64`, `encoding/hex`, `encoding/base32`, `encoding/ascii85`

## TL;DR

Binary-to-text encodings live under `encoding/*`. The two you'll use weekly: `encoding/base64` (web tokens, basic auth, JWT, S/MIME) and `encoding/hex` (hashes, IDs, debug dumps). All packages share the same shape: `Encoding` type with `Encode`/`Decode` methods, plus streaming `Encoder`/`Decoder` wrapping `io.Writer`/`io.Reader`.

## Mental Model

```
base64: 3 bytes → 4 chars from a 64-char alphabet (+ padding '=')
        StdEncoding = standard alphabet (RFC 4648)
        URLEncoding = URL-safe alphabet (- and _ instead of + and /)
        RawStdEncoding / RawURLEncoding = no padding

hex   : 1 byte → 2 chars; alphabet 0-9a-f; lowercase by default
base32: 5 bytes → 8 chars; RFC 4648 alphabet
ascii85: 4 bytes → 5 chars; PostScript / git-style
```

## Syntax & Basic Usage

```go
package main

import (
	"encoding/base64"
	"encoding/hex"
	"fmt"
)

func main() {
	data := []byte("hello")
	b64 := base64.StdEncoding.EncodeToString(data)
	fmt.Println(b64)
	back, _ := base64.StdEncoding.DecodeString(b64)
	fmt.Println(string(back))

	h := hex.EncodeToString(data)
	fmt.Println(h)
	// Output:
	// aGVsbG8=
	// hello
	// 68656c6c6f
}
```

## Deep Dive

### `encoding/base64`

Four built-in encodings:

| Encoding             | Alphabet                                                | Padding |
|----------------------|---------------------------------------------------------|---------|
| `StdEncoding`        | `A-Z a-z 0-9 + /`                                       | `=`     |
| `URLEncoding`        | `A-Z a-z 0-9 - _`                                       | `=`     |
| `RawStdEncoding`     | same as Std                                             | none    |
| `RawURLEncoding`     | same as URL                                             | none    |

JWT uses `RawURLEncoding`. Basic auth header uses `StdEncoding`.

Custom encoding:

```go
enc := base64.NewEncoding("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_")
enc = enc.WithPadding(base64.NoPadding)
```

Streaming:

```go
w := base64.NewEncoder(base64.StdEncoding, dst)
w.Write(rawBytes)
w.Close() // emits trailing padding

r := base64.NewDecoder(base64.StdEncoding, src)
io.Copy(out, r)
```

`Close` on the encoder is **required** — without it, the last group's padding isn't written.

### `encoding/hex`

```go
hex.EncodeToString(b)     // lowercase
hex.DecodeString(s)        // case-insensitive

n := hex.EncodedLen(len(b)) // 2 * len(b)
buf := make([]byte, n)
hex.Encode(buf, b)
```

`hex.Dumper`/`hex.Dump`:

```go
fmt.Println(hex.Dump(b))
// 00000000  68 65 6c 6c 6f                                    |hello|
```

`hex.Dump` produces the canonical hex+ASCII layout used by `hexdump -C`.

### `encoding/base32`

Less common; used in TOTP secrets, S3 etag URLs, Tor onion v2 addresses (legacy). RFC 4648 alphabet `A-Z 2-7`.

### `encoding/ascii85`

Postscript and git use it. 4 bytes → 5 chars; denser than base64 (but characters include some that need escaping in URLs/JSON, so it's rarely used outside niche).

### Size math

| Encoding | Bytes in | Chars out |
|----------|----------|-----------|
| base64   | n        | ⌈n/3⌉ × 4 |
| hex      | n        | 2n        |
| base32   | n        | ⌈n/5⌉ × 8 |

`EncodedLen` / `DecodedLen` methods give exact sizes.

### Errors

- `base64.CorruptInputError(offset)` — malformed character at offset.
- `hex.ErrLength` — odd-length input.
- `hex.InvalidByteError(c)` — non-hex character.

## Standard Library Hooks

- `io.Reader`/`io.Writer` for streaming.
- `crypto/rand` paired with `base64.RawURLEncoding` for tokens.
- `net/url.QueryEscape` not the same — that's percent-encoding, not base64.
- `encoding/pem` for PEM-wrapped base64 (TLS certs, SSH keys).

## Real-World Patterns

### 1. Cryptographically random URL-safe token

```go
import (
	"crypto/rand"
	"encoding/base64"
)

func token(n int) (string, error) {
	b := make([]byte, n)
	if _, err := rand.Read(b); err != nil { return "", err }
	return base64.RawURLEncoding.EncodeToString(b), nil
}
```

Use case: session IDs, password reset tokens, OAuth state.

### 2. Basic auth header

```go
import "encoding/base64"

cred := base64.StdEncoding.EncodeToString([]byte(user + ":" + pass))
req.Header.Set("Authorization", "Basic "+cred)
```

### 3. Hex hash output

```go
import (
	"crypto/sha256"
	"encoding/hex"
)

sum := sha256.Sum256(data)
fmt.Println(hex.EncodeToString(sum[:]))
```

Use case: file checksums, object storage etags, content-addressed caches.

### 4. Stream-decode a base64-encoded HTTP body

```go
dec := base64.NewDecoder(base64.StdEncoding, resp.Body)
out, err := os.Create("file.bin")
if err != nil { return err }
defer out.Close()
_, err = io.Copy(out, dec)
return err
```

Use case: APIs that return base64-encoded binary payloads (Gmail, GitHub blob API).

### 5. Debug dump of binary data

```go
fmt.Println(hex.Dump(buf))
```

Use case: protocol debugging, packet dumps.

## Anti-Patterns & Gotchas

**Forgetting `Close()` on `base64.NewEncoder`.** Truncates the last group.

**Mixing standard and URL alphabets in decode.** Decoding `Std`-encoded data with `URLEncoding` fails on `+`/`/`.

**Using `hex.Encode` and forgetting the buffer must be `2*len(src)`.** Use `hex.EncodedLen`.

**Padding mismatches.** `RawStdEncoding` vs `StdEncoding`; pick one consistently per protocol.

**Base64 in URL paths.** Use `URLEncoding` or you'll need percent-encoding too.

**Treating base64 as encryption.** It's encoding, not encryption.

**`fmt.Sprintf("%x", b)` vs `hex.EncodeToString(b)`.** Both work; `hex.EncodeToString` is faster and clearer.

**Decoding huge base64 into memory.** Stream-decode.

## Performance Notes

- `base64`/`hex` are simple table-driven loops; ~1-2 GB/s on modern CPUs.
- `fmt.Sprintf("%x", b)` allocates and routes through reflection; `hex.EncodeToString` is faster.
- Streaming versions reuse small internal buffers; suitable for arbitrary input sizes.
- `crypto/rand.Read` + `base64.RawURLEncoding.EncodeToString` together: ~hundreds of ns for a 32-byte token.

## How Big Companies Use It

- **Every OAuth implementation** uses `base64.RawURLEncoding` for state and JWT segments.
- **Docker** uses hex for image IDs.
- **etcd / Kubernetes** use hex for content hashes; base64 for client cert encoding in kubeconfig.
- **HashiCorp Vault** uses `base64.RawURLEncoding` for token bodies, hex for transit-engine keys.

## Source Code References

Pinned to `go1.26`.

- `encoding/base64`: [`src/encoding/base64/base64.go`](https://github.com/golang/go/blob/master/src/encoding/base64/base64.go).
- `encoding/hex`: [`src/encoding/hex/hex.go`](https://github.com/golang/go/blob/master/src/encoding/hex/hex.go).
- `encoding/base32`: [`src/encoding/base32/base32.go`](https://github.com/golang/go/blob/master/src/encoding/base32/base32.go).
- `encoding/ascii85`: [`src/encoding/ascii85/ascii85.go`](https://github.com/golang/go/blob/master/src/encoding/ascii85/ascii85.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/encoding/base64, /hex, /base32, /ascii85.
- RFC 4648: https://datatracker.ietf.org/doc/html/rfc4648.

## Exercises / Self-Check

1. Write a function that generates a 256-bit URL-safe token.
2. Why does decoding `"aGVsbG8"` (no padding) with `StdEncoding` fail? Try `RawStdEncoding`.
3. Compute a SHA-256 of a file streaming through `hex.NewEncoder` into the output.
4. Implement a basic-auth client wrapper around `http.Client`.
5. Compare allocations of `fmt.Sprintf("%x", b)` vs `hex.EncodeToString(b)`.
