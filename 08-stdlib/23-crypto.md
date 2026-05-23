# `crypto/*` — Cryptography in the Standard Library

## TL;DR

The `crypto` family covers hashing (`crypto/sha256`, `sha512`, `md5`), HMAC (`crypto/hmac`), symmetric (`aes`, `cipher`, `chacha20poly1305`), public-key (`rsa`, `ed25519`, `ecdsa`, `ecdh`), TLS (`crypto/tls`), random (`crypto/rand`), and certificates (`crypto/x509`). Since 1.24, the stdlib supports a FIPS-140 native mode. Always use `crypto/rand` for security-relevant randomness. Prefer modern primitives: ed25519 over ECDSA, chacha20poly1305 over AES-CTR, AES-GCM for AEAD, and let `crypto/tls` choose cipher suites — don't override unless you know exactly why.

## Mental Model

```
crypto/rand        : CSPRNG from OS (getrandom/urandom)
crypto/sha256/etc. : Hash → 32 (or N) bytes
crypto/hmac        : keyed MAC over any hash
crypto/aes + crypto/cipher : block cipher + modes (GCM, CBC, CTR)
crypto/chacha20poly1305    : AEAD; constant-time, no AES hardware needed
crypto/rsa, /ed25519, /ecdsa, /ecdh : public-key
crypto/x509        : certificate parsing/verification
crypto/tls         : TLS 1.2/1.3 (server + client)
```

## Syntax & Basic Usage

```go
package main

import (
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	sum := sha256.Sum256([]byte("hello"))
	fmt.Println(hex.EncodeToString(sum[:]))

	key := make([]byte, 32)
	rand.Read(key)
	mac := hmac.New(sha256.New, key)
	mac.Write([]byte("message"))
	tag := mac.Sum(nil)
	fmt.Println(hex.EncodeToString(tag))
	// Output (random key produces random tag):
	// 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
	// (hex of HMAC)
}
```

## Deep Dive

### Hashing

```go
sha256.Sum256(data)         // returns [32]byte
h := sha256.New()           // streaming
h.Write(chunk1); h.Write(chunk2)
digest := h.Sum(nil)
```

Available: `crypto/md5`, `sha1`, `sha256`, `sha512`, `crypto/sha3` (since 1.24).

**Use SHA-256 by default.** MD5 and SHA-1 are broken for security; use them only for non-security checksums (and even then, prefer better).

`hash.Hash` is `io.Writer` — you can write any stream.

### HMAC

```go
mac := hmac.New(sha256.New, key)
mac.Write(data)
tag := mac.Sum(nil)

ok := hmac.Equal(tag, expected) // constant-time
```

**Always use `hmac.Equal`**, never `bytes.Equal`, when comparing MACs — prevents timing attacks.

### Symmetric encryption

AES-GCM is the default AEAD:

```go
block, _ := aes.NewCipher(key) // 16, 24, or 32 byte key
gcm, _ := cipher.NewGCM(block)
nonce := make([]byte, gcm.NonceSize())
rand.Read(nonce)
ct := gcm.Seal(nonce, nonce, plaintext, additionalData)

// Decrypt:
nonce, ct = ct[:gcm.NonceSize()], ct[gcm.NonceSize():]
pt, err := gcm.Open(nil, nonce, ct, additionalData)
```

ChaCha20-Poly1305 alternative (`golang.org/x/crypto/chacha20poly1305`, also in stdlib at `crypto/chacha20poly1305` since 1.20):

```go
import "crypto/chacha20poly1305"
aead, _ := chacha20poly1305.New(key)
ct := aead.Seal(nonce, nonce, pt, ad)
```

**Never reuse a nonce with the same key.** AES-GCM's security collapses; nonce-reuse with ChaCha20-Poly1305 leaks both plaintexts.

### Public-key signatures

```go
import "crypto/ed25519"

pub, priv, _ := ed25519.GenerateKey(rand.Reader)
sig := ed25519.Sign(priv, message)
ok := ed25519.Verify(pub, message, sig)
```

Use ed25519 unless you have a reason for ECDSA (e.g., regulatory). RSA is still common for legacy interop.

### Diffie-Hellman / ECDH

```go
import "crypto/ecdh"
curve := ecdh.X25519()
priv, _ := curve.GenerateKey(rand.Reader)
shared, _ := priv.ECDH(theirPub) // derive shared secret
```

`crypto/ecdh` (since 1.20) is the modern API; older code uses `crypto/elliptic` directly.

### `crypto/tls`

Server:

```go
cert, _ := tls.LoadX509KeyPair("cert.pem", "key.pem")
cfg := &tls.Config{Certificates: []tls.Certificate{cert}}
ln, _ := tls.Listen("tcp", ":443", cfg)
```

Client:

```go
cfg := &tls.Config{
	MinVersion: tls.VersionTLS12,
	NextProtos: []string{"h2", "http/1.1"},
}
conn, _ := tls.Dial("tcp", "example.com:443", cfg)
```

Defaults are sensible; do **not** disable verification (`InsecureSkipVerify: true`) in production.

### `crypto/x509`

```go
pem, _ := os.ReadFile("ca.pem")
pool := x509.NewCertPool()
pool.AppendCertsFromPEM(pem)
cfg := &tls.Config{RootCAs: pool}
```

Parse a single cert:

```go
block, _ := pem.Decode(pemBytes)
cert, err := x509.ParseCertificate(block.Bytes)
```

Verify chain:

```go
opts := x509.VerifyOptions{Roots: pool, Intermediates: intPool}
_, err := cert.Verify(opts)
```

### `crypto/rand`

```go
buf := make([]byte, 32)
rand.Read(buf)
```

On Linux uses `getrandom(2)` (or `/dev/urandom`); on macOS uses `getentropy`; on Windows uses `BCryptGenRandom`. Never use `math/rand` for keys, tokens, nonces.

### FIPS-140 mode (since 1.24)

```bash
GODEBUG=fips140=on go run main.go
```

The Go runtime ships a self-contained FIPS-validated crypto module since 1.24. Enables compliance with FedRAMP/government contracts.

## Standard Library Hooks

- `encoding/pem`, `encoding/asn1` — for PEM/DER encoding of keys and certs.
- `crypto/subtle` — `ConstantTimeCompare`, `ConstantTimeSelect` for timing-safe ops.
- `crypto/ed25519` for signatures.
- `golang.org/x/crypto` — extensions: `bcrypt`, `argon2`, `nacl/box`, `ssh`.

## Real-World Patterns

### 1. Password hashing with bcrypt

```go
import "golang.org/x/crypto/bcrypt"

hash, _ := bcrypt.GenerateFromPassword([]byte(pw), bcrypt.DefaultCost)
err := bcrypt.CompareHashAndPassword(hash, []byte(pw))
```

**Never** store passwords with SHA-256 alone. Use bcrypt, scrypt, or argon2id.

### 2. JWT-like signed token

```go
import "crypto/hmac"
import "crypto/sha256"
import "encoding/base64"

func sign(payload []byte, key []byte) string {
	mac := hmac.New(sha256.New, key); mac.Write(payload)
	tag := mac.Sum(nil)
	return base64.RawURLEncoding.EncodeToString(payload) + "." +
		base64.RawURLEncoding.EncodeToString(tag)
}
```

(For real JWT, use `golang-jwt/jwt`.)

### 3. AEAD-encrypted blob

```go
func encrypt(key, plaintext []byte) ([]byte, error) {
	block, err := aes.NewCipher(key)
	if err != nil { return nil, err }
	gcm, err := cipher.NewGCM(block)
	if err != nil { return nil, err }
	nonce := make([]byte, gcm.NonceSize())
	if _, err := rand.Read(nonce); err != nil { return nil, err }
	return gcm.Seal(nonce, nonce, plaintext, nil), nil
}
```

Use case: encrypted config at rest.

### 4. Stream-hash a file

```go
f, _ := os.Open(path); defer f.Close()
h := sha256.New()
io.Copy(h, f)
fmt.Println(hex.EncodeToString(h.Sum(nil)))
```

Use case: content addressing.

### 5. TLS server with strong defaults

```go
cfg := &tls.Config{
	MinVersion:               tls.VersionTLS13,
	CurvePreferences:         []tls.CurveID{tls.X25519, tls.CurveP256},
	PreferServerCipherSuites: true,
}
srv := &http.Server{Addr: ":443", Handler: mux, TLSConfig: cfg}
srv.ListenAndServeTLS("cert.pem", "key.pem")
```

## Anti-Patterns & Gotchas

**`math/rand` for crypto.** Use `crypto/rand`.

**Comparing MACs with `bytes.Equal`.** Use `hmac.Equal` (constant-time).

**Reusing GCM nonces.** Catastrophic.

**Implementing your own crypto.** Use stdlib primitives or `x/crypto`.

**`InsecureSkipVerify: true` in production.** Disables TLS validation.

**Storing passwords as SHA-256/512.** Use bcrypt/argon2id.

**Using ECB mode.** Don't.

**Hardcoded keys in source.** Use a KMS or env vars (with care).

**Generating RSA keys < 2048 bits.** 2048 minimum; 3072+ preferred for long-term.

**Trusting MD5/SHA-1 for security.** Broken.

## Performance Notes

- AES-GCM is hardware-accelerated on x86 (AES-NI) and ARM (cryptography extensions) — gigabytes/sec.
- ChaCha20-Poly1305 is constant-time everywhere, ~1 GB/s on modern CPUs even without hardware.
- SHA-256 is hardware-accelerated on ARMv8 and Intel SHA-NI; otherwise ~500 MB/s.
- ed25519 signing: ~tens of microseconds. Verification: ~hundred microseconds.
- TLS handshakes are the slow part of HTTPS; reuse connections.

## How Big Companies Use It

- **Tailscale** uses `crypto/ed25519` for node keys, `crypto/curve25519` for ECDH, all stdlib.
- **HashiCorp Vault** uses `crypto/rsa`, `crypto/ed25519`, `crypto/x509` plus its own KMS abstractions.
- **Caddy** uses `crypto/tls` with automatic certificate management.
- **etcd** uses `crypto/tls` with mTLS between cluster nodes.
- **Cloudflare's tlsproxy** is built on `crypto/tls` with custom session resumption.

## Source Code References

Pinned to `go1.26`.

- `crypto`: [`src/crypto/`](https://github.com/golang/go/tree/master/src/crypto).
- TLS: [`src/crypto/tls/`](https://github.com/golang/go/tree/master/src/crypto/tls).
- FIPS module (1.24+): [`src/crypto/internal/fips140/`](https://github.com/golang/go/tree/master/src/crypto/internal/fips140).
- `crypto/rand`: [`src/crypto/rand/rand.go`](https://github.com/golang/go/blob/master/src/crypto/rand/rand.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/crypto, /crypto/tls, /crypto/rand.
- Filippo Valsorda blog: https://filippo.io (the Go cryptography lead).
- Go 1.24 FIPS notes: https://go.dev/doc/go1.24.
- "Cryptography in Go" (gopher academy talks).

## Exercises / Self-Check

1. Encrypt a file with AES-256-GCM. Decrypt it. Show the ciphertext changes each time due to nonce.
2. Implement HMAC-SHA256 signed cookies. Use `hmac.Equal` for verification.
3. Generate an ed25519 keypair. Sign a message. Verify with the public key.
4. Start a TLS 1.3 server with a self-signed cert. Connect with a client that pins the cert.
5. Why is reusing a GCM nonce catastrophic? Read the AES-GCM security analysis.
