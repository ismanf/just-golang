# Secrets Management — Storage, Retrieval, Lifecycle

## TL;DR

A "secret" is any value that gives bearer access — API key, DB password, signing key, OAuth token, encryption key. The lifecycle has five concerns: **provisioning** (how it gets to the process), **storage in memory** (how it stays in the process), **rotation** (replacing it without downtime), **revocation** (turning it off when compromised), and **audit** (who used it when). Go's idiomatic answer in 2026 is: **never** put a secret in source, **never** put a long-lived secret in an environment variable in plaintext, pull secrets from a secrets manager (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, or Doppler/Infisical) at startup, encrypt at rest with a Key Management Service (AWS KMS, GCP KMS, HashiCorp Transit), and store secret material in `[]byte` slices (never `string` — strings are immutable and survive in memory longer than you want). The single biggest gotcha: **Go's runtime makes memory wiping nearly impossible** without the `runtime.AddCleanup`/finalizer dance, manual `crypto/internal/...` use, or third-party libraries like `awnumar/memguard`. For most apps this doesn't matter — the bigger threat is logs and env-var leakage — but for high-value secrets (root signing keys, encryption KEKs), it does.

## Mental Model

```
   ┌────────────────────────────────────────────────────────────┐
   │  Secrets Manager (Vault / AWS SM / GCP SM / Azure KV)       │
   │  • encrypts secrets at rest with KMS                         │
   │  • audit log of every read                                   │
   │  • dynamic secrets: rotates DB creds, AWS IAM creds on demand│
   └────────────────────────────────────────────────────────────┘
            │
            │ workload identity (IAM role, OIDC, SPIFFE)
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Your Go process                                             │
   │  • short-lived secrets in []byte                             │
   │  • TTL-tracked; refresh before expiry                        │
   │  • never logged, never sent in error responses                │
   │  • wipe on rotation (best-effort)                            │
   └────────────────────────────────────────────────────────────┘
```

The mental model rule: **a secret in your process should be the freshest possible copy of a secret stored, audited, and rotated outside your process.** "Stored in env-var, set 14 months ago, shared by 9 services" is the anti-pattern.

## How NOT to Store Secrets

```go
// 1. Hard-coded in source — discoverable via git history forever
const apiKey = "sk-live-abc123..."

// 2. Plaintext env var with no rotation story
key := os.Getenv("API_KEY")    // long-lived, in every `ps` output, in CI logs

// 3. Plaintext config file checked in (yes, this happens)
//    config.yaml committed with "production_db_password: hunter2"

// 4. Read from a public URL or insecure store
resp, _ := http.Get("http://config-server.internal/secrets")  // no auth

// 5. Logged for "debugging"
log.Printf("connecting with key=%s", apiKey)
```

Even private repos are not a secret-storage mechanism — engineers leave, branches get force-pushed, history gets cloned to laptops.

## How TO Provision Secrets

### Pattern 1 — KMS-encrypted env vars (better than plaintext)

```bash
# At deploy time, with a KMS-encrypted blob
ENCRYPTED_DB_PASSWORD=$(aws kms encrypt --key-id alias/app-prod \
  --plaintext "$(cat password.txt)" --output text --query CiphertextBlob)

# Pass to process
ENCRYPTED_DB_PASSWORD=$ENCRYPTED_DB_PASSWORD ./myapp
```

```go
// At startup
import (
    "context"
    "encoding/base64"
    "github.com/aws/aws-sdk-go-v2/service/kms"
)

func loadDBPassword(ctx context.Context, kmsClient *kms.Client) ([]byte, error) {
    ct, _ := base64.StdEncoding.DecodeString(os.Getenv("ENCRYPTED_DB_PASSWORD"))
    out, err := kmsClient.Decrypt(ctx, &kms.DecryptInput{CiphertextBlob: ct})
    if err != nil { return nil, err }
    return out.Plaintext, nil
}
```

The KMS audit log records who decrypted, when. An env var leak now requires *also* compromising IAM.

### Pattern 2 — Secrets manager at startup

```go
import (
    "context"
    "github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

func fetchSecret(ctx context.Context, c *secretsmanager.Client, name string) ([]byte, error) {
    out, err := c.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
        SecretId: &name,
    })
    if err != nil { return nil, err }
    if out.SecretBinary != nil {
        return out.SecretBinary, nil
    }
    return []byte(*out.SecretString), nil
}
```

### Pattern 3 — File-mounted secret (Kubernetes Secret, Vault Agent, Doppler)

```go
func loadFromFile(path string) ([]byte, error) {
    b, err := os.ReadFile(path)
    if err != nil { return nil, err }
    return bytes.TrimRight(b, "\r\n"), nil   // strip trailing newline
}
```

Common pattern in Kubernetes:

```yaml
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: app-prod-secrets
```

The CSI driver mounts secrets from Vault/AWS/GCP/Azure into the pod filesystem. Your process reads files; rotation happens by the driver re-mounting.

### Pattern 4 — Vault dynamic secrets

```go
import "github.com/hashicorp/vault-client-go"

vc, _ := vault.New(vault.WithAddress("https://vault.example.com:8200"))
vc.SetToken(os.Getenv("VAULT_TOKEN"))

// Vault issues a NEW database credential, valid for the lease duration
out, err := vc.Read(ctx, "database/creds/myapp-prod")
// out.Data["username"], out.Data["password"], out.LeaseID, out.LeaseDuration
```

Dynamic secrets are the gold standard: a unique credential per process instance, automatically expiring. Compromise of one process's creds doesn't compromise others. Vault revokes the lease on shutdown.

## Workload Identity — the Underlying Story

A process needs *some* identity to authenticate to the secrets manager. The chain:

```
  Cloud instance metadata service (IMDS) / OIDC token from k8s
                          │
                          ▼
              "I am workload X in account Y"
                          │
                          ▼
      Cloud IAM grants permission to read secret Z
                          │
                          ▼
            Secret is delivered over TLS
```

Workload identity is **the** key trust anchor. Without it, you fall back to a long-lived bootstrap secret to fetch other secrets — the chicken-and-egg problem. With it, *nothing* about your process needs to be secret in advance.

Common implementations:

- **AWS IRSA** (IAM Roles for Service Accounts): Kubernetes pod assumes an IAM role via OIDC.
- **GKE Workload Identity**: GCP service account bound to Kubernetes SA.
- **Azure Workload Identity**: similar.
- **SPIFFE/SPIRE**: vendor-neutral; issues X.509-SVIDs or JWT-SVIDs.

## In-Memory Storage

```go
// Use []byte, not string
type Credentials struct {
    Username string  // not secret
    Password []byte  // secret — mutable, can be wiped
}

func (c *Credentials) Wipe() {
    for i := range c.Password {
        c.Password[i] = 0
    }
}

defer creds.Wipe()
```

Why `[]byte` over `string`? `string` is immutable; the underlying bytes may be referenced by other string headers, deduplicated by the linker, or kept alive by string interning. `[]byte` you can zero. The runtime still may copy it (GC, escape analysis, growing slices), so this is best-effort, not bulletproof.

### `memguard` — defence in depth

```go
import "github.com/awnumar/memguard"

// Locks pages with mlock; encrypts at rest in memory; zeroes on destroy
key := memguard.NewBufferRandom(32)
defer key.Destroy()

// Open() temporarily makes readable; Freeze() makes read-only; Melt() makes writable
plain := key.Bytes()
// use plain ...
```

Use for: root signing keys, master KEKs, anything you would hate to see in a core dump. Overkill for short-lived tokens.

### `crypto/subtle.ConstantTimeCompare` for verification

```go
// Compare a presented secret to the expected one
if subtle.ConstantTimeCompare(presented, expected) != 1 {
    return errInvalid
}
```

Never `==` or `bytes.Equal` for secret comparison — timing leaks bits.

## Rotation

Three rotation models:

### Lease-based (best — Vault, AWS dynamic)

```go
type DBPool struct {
    mu       sync.RWMutex
    pool     *pgxpool.Pool
    lease    string
    expires  time.Time
}

func (p *DBPool) refreshBefore(d time.Duration) {
    for {
        sleep := time.Until(p.expires) - d
        if sleep < 0 { sleep = 0 }
        time.Sleep(sleep)
        if err := p.renew(); err != nil {
            log.Printf("renew failed: %v", err)
            // fall back to full re-fetch from Vault
        }
    }
}
```

Renew before expiry; fall back to full re-fetch on failure.

### Scheduled (good — cron rotates secret manager value)

```
00:00 daily: terraform/secretsmanager rotates production-db-password
       │
       ▼
00:01: app reloads secret (SIGHUP, file watch, or polling)
```

```go
// Polling pattern
go func() {
    t := time.NewTicker(5 * time.Minute)
    for range t.C {
        new, _ := fetchSecret(ctx, c, "myapp/db-password")
        if !bytes.Equal(new, current.Load().([]byte)) {
            current.Store(new)
            log.Println("rotated db password")
            // ... reconnect pool
        }
    }
}()
```

### Push-based (best UX — secret manager notifies)

AWS Secrets Manager rotation Lambda invokes a webhook; your service receives the new secret and acknowledges. Implementation is more involved but eliminates the polling lag.

## Revocation

The forgotten cousin of rotation. If a secret leaks, you must:

1. **Revoke the leaked value**. With lease-based, revoke the lease. With static secrets, rotate immediately.
2. **Audit retroactively**: who used the leaked value, when, from where?
3. **Force re-auth** on dependent services if applicable.
4. **Update detection rules**: scan logs for the leaked value's prefix in case it appears again.

Pre-incident: ensure your secrets manager supports immediate revocation. Vault leases can be revoked in O(1) via `vault lease revoke`. AWS Secrets Manager rotation can be triggered manually.

## Anti-Patterns & Gotchas

**Hard-coded secrets.** Even in tests. Use `t.Setenv` or test fixtures with throwaway values.

**Secrets in `os.Args`.** Anyone with `ps` can read them. Use env vars (only marginally better) or files.

**Logging request bodies, headers, query strings.** All common secret carriers. Redact at the logger layer (slog has `slog.Attr` redaction patterns; logrus has hooks).

**Including secrets in error messages.** `fmt.Errorf("auth failed: bad password %q", pw)` will end up in CloudWatch/Datadog.

**Round-tripping secrets through JSON marshalling.** A struct field with a secret should have `json:"-"` tag — or be a separate type entirely.

**Storing secrets in `string`.** Mostly cosmetic, but matters for the high-stakes case. Use `[]byte`.

**Forgetting to wipe `[]byte` after use.** Best-effort but worth it for high-value keys.

**Caching secrets longer than their lease.** Tie cache TTL to lease TTL.

**Fetching a secret on every request.** Defeats secrets manager rate limits and adds latency. Fetch + cache + refresh-before-expiry.

**`if err := vault.Login(...); err != nil { panic(err) }` at init.** A secrets-manager outage shouldn't crash all your services. Retry with backoff; allow graceful degradation where possible.

**Single shared secret across environments.** Dev secrets ≠ prod secrets. Even better: dev secrets are *generated* per developer/branch, not shared.

**Using long-lived AWS access keys.** Use IAM roles, IRSA, or workload identity.

**Putting secrets in container image layers.** Layers are cached and shipped. Use multi-stage builds + runtime mounts.

**Trusting `git secrets`/`detect-secrets` alone.** Pre-commit hooks miss; rotation/revocation must be the safety net.

**Forgetting backup encryption.** Database backups need encrypted at rest with a *different* KEK from the running DB — limit blast radius.

## Operational Patterns

### Slog with redaction

```go
type Secret string

func (s Secret) LogValue() slog.Value {
    return slog.StringValue("[REDACTED]")
}

cfg := struct {
    User string
    Pass Secret
}{"alice", "p@ss"}

slog.Info("config", "cfg", cfg)
// → "cfg.Pass": "[REDACTED]"
```

Newer Go versions (1.21+) make `LogValuer` the right hook. Wrap secret strings in a typed wrapper that implements it; the logger never sees the plaintext.

### Health endpoint should not expose secret presence

```go
// BAD
{ "ok": true, "db": { "host": "...", "password_prefix": "p@ss" } }

// GOOD
{ "ok": true }
```

### Secret hash in logs (audit, not value)

```go
// Log the first 4 chars of a SHA-256 of the secret — enough to identify rotations
import "crypto/sha256"
sum := sha256.Sum256(secret)
log.Printf("loaded secret fingerprint=%x", sum[:4])
```

Useful for "did rotation succeed?" without leaking value.

## Performance Notes

- **AWS SecretsManager `GetSecretValue`**: ~30–100ms per call. Cache.
- **GCP SM `AccessSecretVersion`**: ~30–80ms. Cache.
- **Vault `Read`**: ~5–20ms within the same VPC. Cache.
- **KMS `Decrypt`**: ~10–30ms. Cache the *decrypted plaintext* (not the ciphertext) if you'll need it again.
- **In-memory `[]byte` access**: ns. Free.
- **`memguard` access**: µs per Open/Close due to mprotect.

Operational rule: at startup, fetch all known secrets in parallel (`errgroup.Group`), cache for the lease duration, refresh in a background goroutine. Total startup cost: O(slowest fetch), not O(sum).

## How Big Companies Use It

- **HashiCorp Vault** is the de-facto industry standard for on-prem and hybrid. Most large fintechs and regulated industries run it.
- **AWS** customers use AWS Secrets Manager + IRSA for Kubernetes, Lambda env decryption with KMS Key Policies, and Parameter Store for non-secret config.
- **Google** internally uses an internal secrets manager (similar to but predating GCP Secret Manager). External GCP customers use Secret Manager + Workload Identity.
- **Microsoft Azure** uses Key Vault + Workload Identity (AKS) or Managed Identity (VMs).
- **Kubernetes Secrets** as-shipped are base64-encoded plaintext; production deployments wrap them with KMS encryption-at-rest (`--encryption-provider-config`) or use external-secrets-operator to pull from Vault/AWS/GCP.
- **Stripe** built and open-sourced **veneur** + uses bespoke per-environment secret stores with extensive audit.
- **Netflix** open-sourced **Bless** (SSH cert authority) and uses **Spinnaker** for deploy-time secret injection.
- **GitHub** uses **Hubot/Lita-style** signed grants for human access to production secrets; programmatic access via short-lived tokens from internal IdP.
- **Tailscale** distributes per-node keys generated at first boot; no shared static secrets across the fleet.
- **Cloudflare** uses an internal KMS + SPIFFE-based identity for service-to-service auth.

## Source Code References

- HashiCorp Vault Go client: https://github.com/hashicorp/vault-client-go.
- AWS SDK for Go (Secrets Manager, KMS): https://github.com/aws/aws-sdk-go-v2.
- GCP Secret Manager client: https://github.com/googleapis/google-cloud-go/tree/main/secretmanager.
- Azure Key Vault: https://github.com/Azure/azure-sdk-for-go.
- `memguard`: https://github.com/awnumar/memguard.
- SPIFFE/SPIRE Go libraries: https://github.com/spiffe/go-spiffe.
- External Secrets Operator: https://github.com/external-secrets/external-secrets.
- Doppler Go SDK: https://github.com/DopplerHQ/cli.
- Sigstore cosign (signing keys): https://github.com/sigstore/cosign.

## Further Reading

- HashiCorp, "Vault Architecture": https://developer.hashicorp.com/vault/docs/internals/architecture.
- AWS, "Best practices for using IAM roles": https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html.
- NIST SP 800-57 — Key Management Recommendations.
- Google SRE Book, ch. on credential management: https://sre.google/sre-book/.
- "The 12-Factor App — Config" — https://12factor.net/config — the original "config in env" argument, with subsequent industry rebuttals.
- SPIFFE specification: https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/.
- "Trust on First Use" patterns (TOFU vs PKI vs SPIFFE): various.
- OWASP "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html.

## Exercises / Self-Check

1. Build a Go service that fetches a secret from AWS Secrets Manager at startup, caches it in a `[]byte`, and wipes it on shutdown. Verify with a debugger that the bytes are zeroed before process exit.
2. Implement a typed `Secret` wrapper that implements `LogValuer` to redact in slog and `MarshalJSON` to redact in JSON. Use it across your codebase.
3. Set up Vault dynamic secrets for a PostgreSQL database. Confirm each process gets a unique DB user; revoke the lease and verify the user is dropped.
4. Write a benchmark comparing `bytes.Equal` vs `subtle.ConstantTimeCompare` on a 32-byte token. Argue why timing variance matters.
5. Use `memguard` to hold a 4096-bit RSA private key. Verify that `cat /proc/<pid>/maps` shows locked pages; verify that `ptrace`/`gcore` can't extract the bytes trivially.
6. Configure External Secrets Operator to sync a Vault secret into a Kubernetes Secret, mounted into a pod via CSI. Trigger rotation; verify the pod sees the new value within the polling interval.
7. Build a leakage detector: scan recent log lines for the first 4 chars of `sha256(secret)`. Alert if any secret-shaped match appears.
8. Document your secret lifecycle: who can read each secret, how it's provisioned, how it's rotated, who is paged on leak. Put this in your runbook.
