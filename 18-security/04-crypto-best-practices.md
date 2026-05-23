# Crypto Best Practices — `crypto/*` in Go 1.26

## TL;DR

Go's `crypto/*` packages are conservative, audited, and good defaults — but only if you pick the right primitive. Five rules cover ~90% of real-world cases: (1) **never roll your own crypto** — even composing AES + HMAC by hand is dangerous; use an AEAD (`crypto/cipher.AEAD` with `crypto/cipher.NewGCM` or `golang.org/x/crypto/chacha20poly1305`); (2) **source randomness only from `crypto/rand.Reader`** — `math/rand` is predictable; (3) **derive keys with HKDF (`crypto/hkdf` in 1.24+)**, not raw SHA-256 of a password; (4) **compare secrets with `crypto/subtle.ConstantTimeCompare`**; (5) **prefer Ed25519 for signatures and X25519 for key exchange** — they have safer defaults than ECDSA/ECDH P-256, no curve-parameter footguns, and constant-time implementations. Go 1.24+ added native `crypto/hkdf`, `crypto/pbkdf2`, `crypto/sha3`, and `crypto/mlkem` (ML-KEM-768/1024, post-quantum key encapsulation). Go 1.26 ships `crypto/mldsa` (ML-DSA signatures, draft FIPS 204), promotes `golang.org/x/crypto/cryptobyte` patterns into `crypto/internal`, and stabilises the FIPS 140-3 native module — the same `crypto/aes`, `crypto/sha256`, etc., but in a validated boundary when `GOFIPS140=on`.

## Mental Model

```
   "I need to..."                  "Use..."
   ─────────────                  ──────────────────────────────
   encrypt with shared key   ──►  crypto/cipher.NewGCM(aes.New…)
   encrypt for a recipient   ──►  X25519 ECDH → HKDF → AEAD
                                  (or age, or HPKE)
   sign a message            ──►  ed25519
   verify identity (login)   ──►  ed25519 + challenge
   hash a password           ──►  argon2id (golang.org/x/crypto)
                                  or pbkdf2-sha256 (crypto/pbkdf2, 1.24+)
   derive a key from a key   ──►  crypto/hkdf (1.24+)
   integrity-check a file    ──►  sha256 (sha512 if you're paranoid)
   MAC a message             ──►  crypto/hmac with sha256
   random bytes              ──►  crypto/rand.Read
   random int in range       ──►  rand.Int(crypto/rand.Reader, max)
   post-quantum key exchange ──►  crypto/mlkem (1.24+)
   post-quantum signature    ──►  crypto/mldsa (1.26+)
   TLS                       ──►  crypto/tls with sane defaults
```

Defaults are usually right. If you're reaching for `crypto/aes` directly without an AEAD wrapper, something is probably wrong.

## Randomness

```go
import "crypto/rand"

// Bytes
key := make([]byte, 32)
if _, err := rand.Read(key); err != nil {
    panic(err)
}

// Int in [0, max)
n, err := rand.Int(rand.Reader, big.NewInt(1_000_000))

// 1.22+ math/rand/v2 is deterministic+seeded — DO NOT use for security
```

Rules:

- `crypto/rand.Reader` is *always* the right source for keys, nonces, tokens, IDs that need to be unguessable.
- `crypto/rand.Read` never returns short reads — if `err == nil`, you have N bytes.
- It blocks only on first call on Linux while the entropy pool initialises (rare; very fast in practice).
- `math/rand` and `math/rand/v2` are **deterministic PRNGs**. Useful for tests and load generators; never for tokens, keys, or nonces.

## Symmetric Encryption — AEAD

AEAD = Authenticated Encryption with Associated Data. Encryption and integrity check in one primitive. Never use raw `aes.Encrypt` / `cipher.NewCBCEncrypter` without an explicit MAC — that path has produced more security incidents than any other.

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
)

func encrypt(key, plaintext, aad []byte) (ciphertext, nonce []byte, err error) {
    block, err := aes.NewCipher(key) // key must be 16, 24, or 32 bytes
    if err != nil { return nil, nil, err }
    aead, err := cipher.NewGCM(block)
    if err != nil { return nil, nil, err }
    nonce = make([]byte, aead.NonceSize())
    if _, err := rand.Read(nonce); err != nil { return nil, nil, err }
    ciphertext = aead.Seal(nil, nonce, plaintext, aad)
    return ciphertext, nonce, nil
}

func decrypt(key, ciphertext, nonce, aad []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil { return nil, err }
    aead, err := cipher.NewGCM(block)
    if err != nil { return nil, err }
    return aead.Open(nil, nonce, ciphertext, aad)  // verifies tag
}
```

Key rules:

- AES-256-GCM (32-byte key) is the default.
- Nonces are **public** but must be **unique per key**. 12 bytes is GCM's standard. Random 12-byte nonces are safe for ≤2³² messages per key (birthday bound). For higher volumes, use counter-based nonces, or rotate keys.
- AAD (additional authenticated data) is authenticated but not encrypted. Use it for the message header (user ID, timestamp, version byte).
- The tag is appended to the ciphertext by `Seal`. Never split them.

### ChaCha20-Poly1305 (preferred on hardware without AES-NI)

```go
import "golang.org/x/crypto/chacha20poly1305"

aead, err := chacha20poly1305.NewX(key)   // XChaCha20: 24-byte nonce, large nonce space
nonce := make([]byte, aead.NonceSize())
rand.Read(nonce)
ct := aead.Seal(nil, nonce, plaintext, aad)
```

XChaCha20-Poly1305's 24-byte nonce makes random nonces safe for billions of messages per key.

## Key Derivation

Three common needs:

### From another key (KDF)

```go
import "crypto/hkdf"  // 1.24+

// 32-byte output, with optional salt and info
key, err := hkdf.Key(sha256.New, masterKey, salt, []byte("encryption-v1"), 32)
```

`info` lets you derive distinct keys from one master: `"encryption-v1"`, `"mac-v1"`, `"token-v1"`. No salt: pass `nil`. The `info` parameter is the most under-used and most useful — it cryptographically separates purposes.

### From a password (PBKDF)

```go
// Built-in (Go 1.24+)
import "crypto/pbkdf2"
key, err := pbkdf2.Key(sha256.New, password, salt, 600_000, 32)

// Preferred: Argon2id from x/crypto
import "golang.org/x/crypto/argon2"
key := argon2.IDKey(password, salt, 1, 64*1024, 4, 32)
//                    time, memory(KiB), threads, keyLen
```

Argon2id is the modern choice (Password Hashing Competition winner, 2015). Tune `time`/`memory` to ~250–500ms on your target hardware. Store the parameters alongside the hash so you can verify even after re-tuning.

### From elliptic-curve key exchange

```go
import "crypto/ecdh"

curve := ecdh.X25519()
priv, _ := curve.GenerateKey(rand.Reader)
pub := priv.PublicKey()

// On the other side:
peerPriv, _ := curve.GenerateKey(rand.Reader)
peerPub := peerPriv.PublicKey()

shared, _ := priv.ECDH(peerPub)
// Run shared through HKDF before using as a key
key, _ := hkdf.Key(sha256.New, shared, nil, []byte("session"), 32)
```

**Never use the raw ECDH output as a key.** Pass it through HKDF — that's what HKDF was designed for.

## Signatures

### Ed25519 (default)

```go
import "crypto/ed25519"

pub, priv, _ := ed25519.GenerateKey(rand.Reader)   // pub:32, priv:64
sig := ed25519.Sign(priv, message)                 // sig:64
ok := ed25519.Verify(pub, message, sig)
```

Properties:

- Deterministic (no nonce reuse foot-guns).
- Constant-time signing and verification.
- Fast — millions of ops/sec on modern CPUs.
- No parameter choice — there's only one curve.

Use this unless you have a hard interop constraint requiring ECDSA or RSA.

### ECDSA P-256 (when forced to interop)

```go
import "crypto/ecdsa"
import "crypto/elliptic"

priv, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
hash := sha256.Sum256(message)
sig, _ := ecdsa.SignASN1(rand.Reader, priv, hash[:])
ok := ecdsa.VerifyASN1(&priv.PublicKey, hash[:], sig)
```

Always use `SignASN1`/`VerifyASN1` — they handle the encoding correctly.

### RSA (only for legacy interop)

If you must, use `rsa.SignPSS` (PSS padding), 3072-bit minimum, SHA-256 or SHA-512. Avoid PKCS#1 v1.5 except for legacy verification.

### Post-quantum: ML-DSA (Go 1.26+)

```go
import "crypto/mldsa"   // 1.26+

pub, priv, _ := mldsa.GenerateKey65(rand.Reader)  // ML-DSA-65, ~NIST level 3
sig := mldsa.Sign(priv, message, nil)
ok := mldsa.Verify(pub, message, sig)
```

ML-DSA (formerly Dilithium) is the NIST-selected post-quantum signature standard (FIPS 204). Use for *new* applications where you anticipate a long-lived signing capability that must outlast classical-quantum transition (~2030+). For now, **hybrid** (ed25519 + ml-dsa) is the recommended deployment.

## Key Encapsulation (Post-Quantum)

```go
import "crypto/mlkem"   // 1.24+

priv, _ := mlkem.GenerateKey768()
pub := priv.EncapsulationKey()

// Sender encapsulates a shared secret
ct, ss := pub.Encapsulate()       // ss is the shared 32-byte key

// Receiver decapsulates
ss2, _ := priv.Decapsulate(ct)
// ss == ss2 → use through HKDF
```

ML-KEM-768 (formerly Kyber-768) is FIPS 203. Combine with X25519 for hybrid key exchange — that's what Chrome, TLS 1.3, and modern SSH already do.

## MACs

```go
import "crypto/hmac"
import "crypto/sha256"

mac := hmac.New(sha256.New, key)
mac.Write(data)
sum := mac.Sum(nil)

// Verify with constant time
if !hmac.Equal(received, sum) {
    return errInvalid
}
```

`hmac.Equal` is constant-time. Never use `bytes.Equal` for MAC comparison.

## Hashing

| Purpose | Use |
|---------|-----|
| File integrity, content addressing | SHA-256 |
| HMAC | HMAC-SHA-256 |
| Password (interactive) | argon2id |
| Password (high-throughput legacy) | bcrypt or pbkdf2-sha256 |
| Short fingerprint (non-security) | xxh64 / fnv |
| Merkle trees, content-defined | BLAKE3 (`zeebo/blake3`) or BLAKE2b (`x/crypto/blake2b`) |
| Random-oracle in protocol | SHAKE128/256 (`crypto/sha3`, 1.24+) |

Avoid SHA-1 and MD5 for any new security purpose.

## Constant-Time Operations

```go
import "crypto/subtle"

// Compare two []byte of equal length, constant time
ok := subtle.ConstantTimeCompare(a, b) == 1

// Constant-time select
out := subtle.ConstantTimeSelect(cond, vIfTrue, vIfFalse)

// Constant-time byte copy if cond is 1
subtle.ConstantTimeCopy(cond, dst, src)
```

Use for: token comparison, signature byte comparison, any path where attacker-observable timing leaks bits of a secret.

## TLS Configuration

```go
import "crypto/tls"

cfg := &tls.Config{
    MinVersion: tls.VersionTLS13,             // require 1.3
    CipherSuites: nil,                         // let the runtime choose
    PreferServerCipherSuites: false,           // ignored in 1.3
    CurvePreferences: []tls.CurveID{
        tls.X25519MLKEM768,                    // post-quantum hybrid (1.24+)
        tls.X25519,
        tls.CurveP256,
    },
    Certificates: []tls.Certificate{cert},
    // For mTLS:
    ClientCAs: caPool,
    ClientAuth: tls.RequireAndVerifyClientCert,
}
```

Critical defaults Go 1.26 already enforces:

- TLS 1.3 by default; 1.0/1.1 disabled at compile time.
- AES-GCM and ChaCha20-Poly1305 only.
- X25519 + post-quantum hybrid in negotiation.
- ECDSA P-256, P-384, Ed25519 for certificates.

Do not set `InsecureSkipVerify: true` in production. If you need a self-signed peer, put its cert in `RootCAs`.

### Session resumption

```go
cfg.SessionTicketsDisabled = false              // resumption on (default)
cfg.SessionTicketKey = ... // optional, key rotation responsibility moves to you
```

Default behaviour rotates tickets safely. Setting your own key implies you must rotate on a schedule.

## Common Patterns

### Token generation

```go
func newToken() string {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil { panic(err) }
    return base64.RawURLEncoding.EncodeToString(b)
}
```

32 bytes = 256 bits. Never use a UUID for a security token — UUIDv4 only has ~122 bits and depends on randomness quality of the library.

### Encrypted cookie (signed + AEAD)

```go
func sealCookie(key []byte, payload []byte) (string, error) {
    block, _ := aes.NewCipher(key)
    aead, _ := cipher.NewGCM(block)
    nonce := make([]byte, aead.NonceSize())
    if _, err := rand.Read(nonce); err != nil { return "", err }
    ct := aead.Seal(nonce, nonce, payload, nil)   // prepend nonce
    return base64.RawURLEncoding.EncodeToString(ct), nil
}
func openCookie(key []byte, s string) ([]byte, error) {
    raw, err := base64.RawURLEncoding.DecodeString(s)
    if err != nil { return nil, err }
    block, _ := aes.NewCipher(key)
    aead, _ := cipher.NewGCM(block)
    n := aead.NonceSize()
    if len(raw) < n { return nil, errors.New("short") }
    return aead.Open(nil, raw[:n], raw[n:], nil)
}
```

For long-lived cookies, version the payload (`{"v":1,...}`) and put the version in the AAD so old payloads can't be replayed under a new format.

### File-level encryption — use `age`

```go
// github.com/FiloSottile/age
identity, _ := age.GenerateX25519Identity()
recipient := identity.Recipient()

out, _ := os.Create("file.age")
w, _ := age.Encrypt(out, recipient)
io.Copy(w, plaintextReader)
w.Close()
out.Close()
```

`age` was designed by Filippo Valsorda (formerly the Go security lead) as an opinionated, simple file-encryption tool. Use it instead of inventing your own envelope format.

## Anti-Patterns & Gotchas

**Using `math/rand` for tokens, IDs, salts, or nonces.** Always `crypto/rand`.

**Reusing nonces under the same key.** Catastrophic for AES-GCM (key recovery in some cases). Random 12-byte nonce with strict per-key counters, or XChaCha20-Poly1305.

**MAC-then-encrypt** (vs encrypt-then-MAC, which AEAD enforces). Don't roll your own.

**Comparing MACs/tokens with `bytes.Equal`.** Use `hmac.Equal` / `subtle.ConstantTimeCompare`.

**Using SHA-256 of a password as a "hash."** Use argon2id or pbkdf2.

**Hard-coding keys.** Pull from KMS/Vault or a secrets-manager pattern; encrypt at rest with a separate KEK.

**`InsecureSkipVerify: true`.** Even in dev. Pin a cert.

**Mixing TLS versions and curves "for compatibility."** Either require 1.3 or document the legacy reason. Don't enable 1.2 silently.

**`crypto/rsa.SignPKCS1v15` for new code.** Use PSS or, better, Ed25519.

**Storing IV/nonce in a global variable.** It must be generated fresh per encryption.

**Truncating MAC tags.** Don't. The full tag length is what makes the security claim.

**Confusing AAD with plaintext.** AAD is authenticated, not encrypted. If you put a secret there, it leaks.

**Using `ecdsa.Sign` (the non-ASN1 form) and writing your own R/S marshalling.** Use `SignASN1`.

**`net/http`'s `DefaultTransport` without `MinVersion: tls.VersionTLS12`.** Go 1.24+ defaults are sane; older toolchains aren't. Pin.

**Forgetting to wipe key material from memory.** Go GC + immutable strings make this hard; for high-stakes use, `[]byte` with explicit zeroing in `defer`. For paranoia-grade requirements, `memguard` or `mlock`.

**Treating PEM as binary-safe.** PEM is base64-armoured text; `bytes.Equal` on two semantically-equal PEMs may fail due to line-ending differences. Compare DER bytes instead.

## Performance Notes

(Modern x86 with AES-NI; numbers vary widely.)

| Operation | Throughput |
|-----------|-----------|
| AES-256-GCM seal/open | ~3–5 GB/s |
| ChaCha20-Poly1305 (no AES-NI / ARM) | ~1–2 GB/s |
| SHA-256 | ~1–2 GB/s |
| SHA-512 | ~1.5–3 GB/s (faster than SHA-256 on 64-bit) |
| BLAKE3 | ~2–6 GB/s |
| Ed25519 sign | ~30,000 ops/sec |
| Ed25519 verify | ~12,000 ops/sec |
| X25519 ECDH | ~30,000 ops/sec |
| ECDSA P-256 sign | ~50,000 ops/sec |
| ECDSA P-256 verify | ~15,000 ops/sec |
| ML-KEM-768 encap | ~50,000 ops/sec |
| ML-DSA-65 sign | ~5,000 ops/sec |
| argon2id (`m=64MiB t=1`) | ~10–50 ms/op |
| pbkdf2-sha256 (600k iters) | ~150 ms/op |

Use a benchmark on your real hardware; ARM Graviton, Apple Silicon, and Intel/AMD differ noticeably.

## How Big Companies Use It

- **Google** internally uses Tink (https://github.com/google/tink) — a higher-level wrapper over `crypto/*` that exposes primitives like "AEAD," "DeterministicAEAD," "MAC" with safe defaults and prebuilt key-management. Worth using if you want guardrails.
- **Cloudflare** uses `crypto/tls` with full TLS 1.3 + post-quantum hybrid (X25519MLKEM768) since 2024 across their edge.
- **Tailscale** uses WireGuard's crypto primitives (X25519 + ChaCha20-Poly1305 + BLAKE2s) implemented in Go.
- **HashiCorp Vault**'s transit secrets engine uses AES-256-GCM with a versioned key ring and AAD-bound contexts — the model worth copying.
- **Signal Foundation**'s libsignal Go bindings use Ed25519 + X25519 + ChaCha20-Poly1305 throughout.
- **age** (Filippo Valsorda) is the file-encryption gold standard for "I have a file and a recipient public key."
- **Cosign** (Sigstore) uses Ed25519 + ECDSA P-256 + Fulcio-issued certificates.
- **Caddy** auto-renews TLS via ACME with Ed25519 account keys.

## Source Code References

- `crypto/cipher`: https://github.com/golang/go/tree/master/src/crypto/cipher.
- `crypto/aes`: https://github.com/golang/go/tree/master/src/crypto/aes.
- `crypto/ecdh`: https://github.com/golang/go/tree/master/src/crypto/ecdh.
- `crypto/ed25519`: https://github.com/golang/go/tree/master/src/crypto/ed25519.
- `crypto/mlkem`: https://github.com/golang/go/tree/master/src/crypto/mlkem.
- `crypto/mldsa` (1.26+): https://github.com/golang/go/tree/master/src/crypto/mldsa.
- `crypto/hkdf`: https://github.com/golang/go/tree/master/src/crypto/hkdf.
- `crypto/tls`: https://github.com/golang/go/tree/master/src/crypto/tls.
- `golang.org/x/crypto`: https://github.com/golang/crypto.
- `age`: https://github.com/FiloSottile/age.
- Tink: https://github.com/google/tink.

## Further Reading

- Filippo Valsorda's blog: https://blog.filippo.io/ — most readable modern source on Go crypto.
- "Cryptography Engineering" (Ferguson, Schneier, Kohno) — the gold reference.
- "Real-World Cryptography" (David Wong) — modern, practical.
- "Serious Cryptography" (Jean-Philippe Aumasson).
- NIST SP 800-38D — AES-GCM specification.
- NIST FIPS 203/204/205 — ML-KEM, ML-DSA, SLH-DSA standards.
- RFC 8439 — ChaCha20-Poly1305.
- RFC 8032 — Ed25519.
- RFC 5869 — HKDF.

## Exercises / Self-Check

1. Write a function that encrypts and decrypts with AES-256-GCM, using HKDF to derive separate "encrypt" and "MAC" keys from a master key (purpose-binding via `info`).
2. Benchmark `crypto/rand.Read(32)` per call. Confirm it's >1M ops/sec.
3. Write an Ed25519 sign/verify round trip. Modify a single byte of the signature; verify fails.
4. Implement password-storage with argon2id. Tune parameters to ~300ms on your machine. Store and verify a hash.
5. Replace `bytes.Equal` MAC comparison with `hmac.Equal`. Use a timing benchmark to argue why.
6. Generate an ML-KEM-768 keypair. Encapsulate; decapsulate; assert the shared secrets match. Now corrupt one byte of the ciphertext — what happens?
7. Configure a TLS 1.3-only server with `tls.X25519MLKEM768` first in `CurvePreferences`. Connect with `curl --curves x25519_kyber768`. Verify the negotiated curve in keylog.
8. Implement an encrypted cookie that includes a version byte in AAD. Roll the version; show old cookies are rejected.
