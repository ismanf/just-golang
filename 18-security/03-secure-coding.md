# Secure Coding — Input Validation, SSRF, Path Traversal, `os.Root`

## TL;DR

The Go standard library cannot save you from secure-coding mistakes — but it does give you sharp tools if you know which ones to reach for. Five families of bugs cause the bulk of Go security incidents: **path traversal** (now largely solved by `os.Root` in 1.24+, and tightened further in 1.26), **SSRF** (server-side request forgery — your service makes outbound HTTP to attacker-chosen URLs), **command injection** (`exec.Command` misuse), **SQL injection** (raw string-concatenated queries), and **HTML/template injection** (rendering untrusted strings without `html/template`). The defensive postures are: validate *intent*, not just shape; resolve all paths against a known root *before* opening them; use parametric APIs (`database/sql` placeholders, `html/template` escaping, `exec.Command` with arg arrays); and treat any input that becomes a *destination* (URL, hostname, IP) as hostile — never trust DNS resolution. Go 1.26's `os.Root.OpenInRoot`, `net/url.Parse` strict mode, and `net/http`'s default `Server.WriteTimeout` all bend the runtime toward safer defaults, but the language won't catch shape-vs-intent confusion. That's still your job.

## Mental Model

```
   Untrusted input
        │
        ▼
   ┌─────────────────────────────┐
   │ 1. Validate the SHAPE        │  → reject non-conforming early
   │    (length, charset, encoding)│
   └─────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────┐
   │ 2. Validate the INTENT       │  → reject "/etc/passwd" even
   │    (semantic constraints)    │    if it's a "valid path"
   └─────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────┐
   │ 3. Use a SAFE API            │  → parameterised SQL,
   │    (no string-concat)        │    os.Root.Open, exec args
   └─────────────────────────────┘
        │
        ▼
       Action
```

Validation is *not* sanitisation. Sanitising means "I'll fix the bad input"; validation means "I'll reject it." Prefer validation. The output side (templates, SQL placeholders, exec arg lists) handles the rest.

## Path Traversal and `os.Root` (Go 1.24+)

The classic Go path-traversal bug:

```go
// VULNERABLE
func serveFile(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    f, err := os.Open(filepath.Join("/var/uploads", name))
    // attacker passes ?name=../../etc/passwd
    // ...
}
```

`filepath.Join("/var/uploads", "../../etc/passwd")` cleans to `/etc/passwd`. The "join with prefix" defence is **not enough** — `filepath.Clean` removes the `..` after the join.

### The pre-1.24 defence (clunky, error-prone)

```go
func openInRoot_old(root, name string) (*os.File, error) {
    cleaned := filepath.Clean(filepath.Join(root, name))
    if !strings.HasPrefix(cleaned, filepath.Clean(root)+string(os.PathSeparator)) {
        return nil, fmt.Errorf("escape attempt")
    }
    return os.Open(cleaned)
}
```

Still vulnerable to TOCTOU — the path may be a symlink at `os.Open` time even after the prefix check. Symlink races have been a Go-team CVE source.

### The 1.24+ defence — `os.Root`

```go
import "os"

func serveFile(w http.ResponseWriter, r *http.Request) error {
    root, err := os.OpenRoot("/var/uploads")
    if err != nil {
        return err
    }
    defer root.Close()
    f, err := root.Open(r.URL.Query().Get("name"))   // *cannot* escape the root
    // ...
}
```

`os.Root` (and its `*os.File`-returning methods `Open`, `OpenFile`, `Stat`, `Mkdir`, `Create`, `Remove`, `Rename`, `Lstat`, `Readlink`, `Symlink`) refuses any operation that would escape the root directory, including via:

- `..` segments (`/var/uploads/../etc/passwd`)
- Absolute paths (`/etc/passwd`)
- Symlinks pointing outside the root (the symlink is resolved within the root only)
- Mount points (best-effort; Linux `openat2(RESOLVE_BENEATH | RESOLVE_NO_SYMLINKS)` does the heavy lifting)
- Reverse-path attacks via Windows `..\` and case folding

Under the hood on Linux this uses `openat2` with `RESOLVE_BENEATH`. On Windows and other Unixes it uses `O_NOFOLLOW` walks and explicit ancestor checking.

### `os.Root` API (1.24 + 1.26 additions)

```go
type Root struct { /* ... */ }

func OpenRoot(name string) (*Root, error)
func (r *Root) Close() error
func (r *Root) Name() string

// 1.24
func (r *Root) Open(name string) (*os.File, error)
func (r *Root) OpenFile(name string, flag int, perm os.FileMode) (*os.File, error)
func (r *Root) Create(name string) (*os.File, error)
func (r *Root) Stat(name string) (os.FileInfo, error)
func (r *Root) Lstat(name string) (os.FileInfo, error)
func (r *Root) Mkdir(name string, perm os.FileMode) error
func (r *Root) Remove(name string) error

// 1.25
func (r *Root) Chmod(name string, mode os.FileMode) error
func (r *Root) Chown(name string, uid, gid int) error
func (r *Root) ReadFile(name string) ([]byte, error)
func (r *Root) WriteFile(name string, data []byte, perm os.FileMode) error

// 1.26
func (r *Root) Rename(oldname, newname string) error
func (r *Root) Symlink(oldname, newname string) error
func (r *Root) Readlink(name string) (string, error)
func (r *Root) Walk(fn fs.WalkDirFunc) error             // 1.26
func (r *Root) FS() fs.FS                                // 1.26 — pass to template engines
```

### Pattern: serving a sandboxed file tree

```go
// On startup
root, err := os.OpenRoot("/var/srv/static")
// store root in a struct, NOT re-opened per request

// Per request
func (h *handler) serve(w http.ResponseWriter, r *http.Request) {
    name := strings.TrimPrefix(r.URL.Path, "/files/")
    f, err := h.root.Open(name)
    if err != nil {
        http.Error(w, "not found", 404)
        return
    }
    defer f.Close()
    io.Copy(w, f)
}
```

Open the root once; reuse for the process lifetime. Closing it doesn't close already-open file handles — it just prevents future opens.

### Pattern: writing under a quota

```go
root, _ := os.OpenRoot("/var/uploads/tenant-42")
f, err := root.OpenFile("incoming.bin", os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0o600)
```

Combined with disk-quota or `f.Truncate` limits, you bound a tenant's filesystem footprint without juggling absolute paths.

## SSRF — Server-Side Request Forgery

Your backend service calls a URL the client controls. Without protection, the client points your service at:

- `http://169.254.169.254/...` — cloud-instance metadata service (AWS IMDS)
- `http://localhost:6379/...` — your local Redis with no auth
- `http://10.0.0.1/...` — internal admin panel
- `file:///etc/passwd` (if you use a URL library that supports file:)

### Wrong: validate string then fetch

```go
// VULNERABLE — TOCTOU between Parse and Get
u, _ := url.Parse(input)
if !strings.HasSuffix(u.Hostname(), ".trusted.com") { return errRefused }
resp, _ := http.Get(input)   // DNS resolves NOW — attacker can flip
```

DNS rebinding: the hostname `evil.example.com` resolves to a trusted IP at parse time and to `169.254.169.254` at fetch time.

### Right: pin the resolved IP, then dial directly

```go
import (
    "context"
    "errors"
    "net"
    "net/http"
    "net/netip"
    "time"
)

var blockedNets = []netip.Prefix{
    netip.MustParsePrefix("127.0.0.0/8"),
    netip.MustParsePrefix("10.0.0.0/8"),
    netip.MustParsePrefix("172.16.0.0/12"),
    netip.MustParsePrefix("192.168.0.0/16"),
    netip.MustParsePrefix("169.254.0.0/16"),  // link-local + IMDS
    netip.MustParsePrefix("::1/128"),
    netip.MustParsePrefix("fc00::/7"),
    netip.MustParsePrefix("fe80::/10"),
}

func safeDialContext(ctx context.Context, network, addr string) (net.Conn, error) {
    host, port, err := net.SplitHostPort(addr)
    if err != nil { return nil, err }
    ips, err := net.DefaultResolver.LookupNetIP(ctx, "ip", host)
    if err != nil { return nil, err }
    if len(ips) == 0 { return nil, errors.New("no addresses") }
    ip := ips[0]
    for _, p := range blockedNets {
        if p.Contains(ip) {
            return nil, errors.New("ssrf: blocked range")
        }
    }
    // Dial the IP directly — eliminates DNS-rebinding TOCTOU
    var d net.Dialer
    return d.DialContext(ctx, network, net.JoinHostPort(ip.String(), port))
}

func ssrfSafeClient() *http.Client {
    return &http.Client{
        Timeout: 10 * time.Second,
        Transport: &http.Transport{
            DialContext: safeDialContext,
            // Disable redirects to attacker-chosen URLs:
            // (or set CheckRedirect on the client)
        },
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            if len(via) >= 5 {
                return errors.New("too many redirects")
            }
            // Redirected URL is re-fetched via DialContext, so its IP is re-checked.
            return nil
        },
    }
}
```

Key invariants:

1. **Resolve hostname → IP once**, then dial by IP. Anything else is racy.
2. **Reject private/loopback/link-local ranges** explicitly.
3. **Reject IPv4-in-IPv6** (`::ffff:127.0.0.1`) — your IPv4 block list must also catch the IPv6 mapped form. `netip.Addr.Is4In6()` + `.Unmap()` handles this.
4. **Cap redirects** and re-validate each hop.
5. **Cap response body size** (`io.LimitReader`) — attacker can otherwise feed you 50GB.
6. **Cap response time** (client `Timeout` AND per-request `context.WithTimeout`).

### URL scheme allow-listing

```go
u, err := url.Parse(input)
if err != nil { return err }
switch u.Scheme {
case "http", "https":
    // ok
default:
    return errors.New("scheme not allowed")
}
```

Without this, `file:///etc/passwd` (with libraries that follow it), `gopher://`, `ftp://`, `data:` and friends become attack surface.

## Command Injection

```go
// VULNERABLE
out, _ := exec.Command("sh", "-c", "ls " + userInput).Output()
// userInput = "; cat /etc/passwd"
```

The fix is to **never pass user input through a shell**. `exec.Command` already takes an argv list — use it:

```go
// SAFE
out, _ := exec.Command("ls", userInput).Output()
```

Each element of argv is delivered to the child verbatim; the shell never parses them. Even `userInput = "; rm -rf /"` is passed as a single literal filename to `ls`.

If you *must* construct a shell pipeline (rare — almost always a sign of bad design), use `golang.org/x/sys/execabs` (validates `PATH`) and quote with `shellescape` or pass commands as separate `exec.Command`s connected via `cmd.Stdout = otherCmd.StdinPipe()`.

### `LookPath` and PATH hijacking (CVE-2022-29804, CVE-2022-30580)

`exec.LookPath("./foo")` historically returned `./foo` if the binary existed *in the current directory* — even when `.` wasn't in PATH. Go 1.19 fixed this. Don't pin to older toolchains.

### `os/exec` and environment isolation

```go
cmd := exec.CommandContext(ctx, "git", "rev-parse", "HEAD")
cmd.Env = []string{
    "PATH=/usr/bin:/bin",
    "HOME=/tmp/sandbox",
    "GIT_TERMINAL_PROMPT=0",
}
cmd.Dir = workdir
```

Default `cmd.Env = nil` inherits the parent process env — including `LD_PRELOAD`, `GIT_SSH_COMMAND`, `BASH_ENV`, and many other lever points. For untrusted code execution always set `cmd.Env` explicitly.

## SQL Injection

```go
// VULNERABLE
rows, _ := db.Query("SELECT * FROM users WHERE name = '" + name + "'")
```

The fix is parameterised queries. The `database/sql` API has placeholders for exactly this:

```go
rows, err := db.QueryContext(ctx,
    `SELECT id, email FROM users WHERE name = $1 AND active = $2`,
    name, true)
```

The driver sends the SQL and parameters separately to the database. Even if `name = "'; DROP TABLE users; --"` the database treats it as a single string literal — never as SQL.

### Identifiers can't be parameterised

Table names, column names, `ORDER BY` directions — these are *not* values and cannot use placeholders. Allow-list:

```go
allowedSort := map[string]string{
    "name": "name",
    "created": "created_at",
}
col, ok := allowedSort[input]
if !ok { return errBadSort }
q := fmt.Sprintf(`SELECT * FROM users ORDER BY %s`, col)
```

The map enforces a finite, code-controlled vocabulary; user input only selects a key.

### `sqlx`, `sqlc`, `pgx`

- `sqlx.NamedExec`: named placeholders — safer than positional for long queries.
- `sqlc` generates type-safe Go from `.sql` files at compile time — eliminates string concatenation entirely.
- `pgx` is the most common modern Postgres driver; supports `pgx.Identifier` for safe-quoted identifiers.

## HTML/Template Injection (XSS)

`text/template` does **no** escaping. `html/template` is context-aware HTML escaping. Always:

```go
import "html/template"

tmpl := template.Must(template.New("page").Parse(`<p>Hello, {{.Name}}</p>`))
tmpl.Execute(w, map[string]any{"Name": userInput})  // safely escaped
```

`html/template` knows whether `{{.X}}` is inside attribute context, JS context, URL context, CSS context, and escapes accordingly. Bypassing it via `template.HTML(input)` or `template.URL(input)` is an opt-out — only use with values you've validated as safe.

### Common XSS pitfalls

- Concatenating user data into JSON embedded in `<script>` blocks → use `template.JS` only with strictly numeric/encoded data; better, render the data as `application/json` and fetch via JS.
- `href="javascript:..."` allowed through → `html/template` warns on this since 1.5, but custom escapers might miss it. Allow-list schemes.
- `dangerouslySetInnerHTML`-like patterns in Go template helpers — avoid them.

## Input Validation Recipes

```go
import (
    "errors"
    "net/mail"
    "regexp"
    "unicode/utf8"
)

// Email — use net/mail, not regex
func ValidateEmail(s string) error {
    addr, err := mail.ParseAddress(s)
    if err != nil { return err }
    if addr.Address != s { return errors.New("address has extra components") }
    return nil
}

// UUID v4 — strict
var uuidRe = regexp.MustCompile(`^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`)

// Username
func ValidateUsername(s string) error {
    if len(s) < 3 || len(s) > 32 { return errors.New("length") }
    if !utf8.ValidString(s) { return errors.New("not utf8") }
    for _, r := range s {
        if !(r >= 'a' && r <= 'z') && !(r >= '0' && r <= '9') && r != '_' {
            return errors.New("charset")
        }
    }
    return nil
}
```

Rules of thumb:

- Length cap **before** parsing — protects against DoS via huge inputs.
- UTF-8 validity (`utf8.ValidString`) **before** processing strings — invalid UTF-8 has caused real CVEs (CVE-2022-23806, CVE-2022-27664).
- Reject, don't sanitise.
- Validate at the *trust boundary* (request handler) — not after data has propagated.

## Race Conditions and TOCTOU

```go
// VULNERABLE — symlink swap between check and use
info, _ := os.Stat(path)
if info.Mode().IsRegular() {
    f, _ := os.Open(path)   // attacker swapped to symlink → /etc/passwd
}
```

Use `os.Root` (handles by file descriptor; symlink swap can't break out) or `os.OpenFile(path, os.O_NOFOLLOW, 0)` to refuse following symlinks at open time.

## HTTP Server Defaults (Go 1.26)

Go 1.26 hardens `net/http.Server` defaults:

- `ReadHeaderTimeout` defaults to a non-zero value (was unset).
- `WriteTimeout` defaults to a finite value when `http.ListenAndServe` is used.
- `MaxHeaderBytes` defaults to 1 MiB and is enforced more strictly.

Older code that relied on zero-valued (infinite) timeouts may need to set them explicitly to `0` if you really want no timeout — but you should not want that.

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
    MaxHeaderBytes:    1 << 20,
}
```

Without timeouts, an attacker holding a slow-loris connection consumes a goroutine per stuck request — classic Go-server DoS.

## Anti-Patterns & Gotchas

**`filepath.Join(root, userInput)` as the only defence.** `..` segments survive the join unless you also do a strict `HasPrefix` check after `filepath.Clean`. Use `os.Root` instead.

**Hostname allow-list without IP pinning.** DNS rebinding defeats it.

**`http.Get(userURL)` from a backend.** Always SSRF-protect.

**Trusting `r.Host`, `X-Forwarded-For`, `X-Forwarded-Host` without explicit proxy config.** Behind a proxy, these are spoofable. Use `http.Server.TrustForwardedHeaders` (1.25+) or a vetted reverse-proxy library.

**Logging user input verbatim.** Log injection: `userInput = "\n[ADMIN] deleted=true"` corrupts log parsing. Either escape (`%+q`) or use structured logging (`slog`).

**`crypto/md5`, `crypto/sha1` for authentication.** They're broken for collision-resistance. Use SHA-256 / SHA-512 / BLAKE2 for integrity, HMAC-SHA-256 for MACs.

**Comparing tokens with `==`.** Use `crypto/subtle.ConstantTimeCompare` for any secret comparison.

**Reading entire request body into memory.** `io.ReadAll(r.Body)` with no `io.LimitReader` is a DoS gift. Always cap.

**Trusting `Content-Length` from a client.** It's a lie. Cap with `MaxBytesReader`.

**Storing JWTs in URL query strings.** They leak via referer, logs, history. Use cookies (Secure, HttpOnly, SameSite=Lax) or Authorization headers.

**Skipping TLS cert verification.** `tls.Config{InsecureSkipVerify: true}` — even in tests, mark it loud, and never in production. If you must talk to a self-signed peer, pin the cert in `RootCAs`.

**Forgetting `defer f.Close()`** on files opened from user input. File descriptor exhaustion is a denial-of-service vector and a security one (random handles dangling open can keep deleted-but-sensitive data alive).

**Mixing `os` and `os.Root` operations on the same logical path.** If half your code uses `os.Open` with concatenation and the other half uses `os.Root`, the weak half is the security boundary.

## Performance Notes

- **`os.Root.Open` overhead**: <1µs on Linux 5.6+ (single `openat2` syscall). On older kernels or non-Linux, the userland fallback walks ancestors and adds 1–5µs.
- **Strict IP allow-listing** in SSRF: a single `netip.Addr.Compare` per CIDR — sub-microsecond for typical block lists.
- **`html/template` escaping**: ~2–5x slower than `text/template` for the same output. Worth it.
- **Parameterised SQL**: identical or faster than concatenated (prepared statements cache parse trees).
- **`crypto/subtle.ConstantTimeCompare`**: ~10x slower than `bytes.Equal` on short inputs; negligible absolute cost.

## How Big Companies Use It

- **Google's internal Go style guide** mandates `os.Root` for any code that opens user-named files and forbids raw `os.Open(filepath.Join(root, name))`.
- **Cloudflare's Tunnel and `cloudflared`** use a strict SSRF guard inspired by the netip-based dialer above.
- **Tailscale**'s `tsnet` library uses a synthetic resolver to enforce that user-named hostnames resolve only within the Tailnet — a strong-form SSRF guard built into the dialer.
- **GitHub** uses `sqlc` + a custom Go linter that forbids `db.Query` with a non-literal first argument; PRs touching DB code require type-safe query files.
- **Shopify** runs `gosec` + custom Semgrep rules on every Go PR to flag `exec.Command("sh", "-c", ...)`, `os.Open(filepath.Join(...))`, and direct URL fetch without dialer wrap.
- **Kubernetes** uses a `runtime/file` wrapper (precursor to `os.Root`) that has eliminated several historic path-traversal CVEs in `kubelet`.
- **Caddy** server applies the SSRF dialer pattern for its `reverse_proxy` directive when proxying to user-named upstreams.

## Source Code References

- `os.Root` implementation: https://github.com/golang/go/blob/master/src/os/root_unix.go and `root_windows.go`.
- `openat2` resolve flags: https://man7.org/linux/man-pages/man2/openat2.2.html.
- `net/url`: https://github.com/golang/go/blob/master/src/net/url/url.go.
- `html/template` escapers: https://github.com/golang/go/blob/master/src/html/template/escape.go.
- `crypto/subtle`: https://github.com/golang/go/blob/master/src/crypto/subtle/.
- gosec scanner: https://github.com/securego/gosec.
- semgrep rules for Go: https://github.com/dgryski/semgrep-go.

## Further Reading

- Go security policy: https://go.dev/security.
- "Securing Go applications" (Filippo Valsorda): https://blog.filippo.io/.
- OWASP Top 10 (2021): https://owasp.org/Top10/.
- "The Tangled Web" (Michał Zalewski) — still the best book on web-app security primitives.
- Russ Cox, "Anatomy of a path-traversal" (CVE write-ups across years): https://research.swtch.com/.
- "On the Insecurity of Whitelists" (rebinding-class attacks): various.
- `os.Root` proposal: https://github.com/golang/go/issues/67002.

## Exercises / Self-Check

1. Write a file-server handler that uses `os.Root` to serve `/var/srv/public`. Attempt `../etc/passwd`, `../../etc/passwd`, symlink-to-`/etc/passwd`, and absolute paths. All should fail.
2. Write an SSRF-safe HTTP client. Verify it refuses `http://169.254.169.254/`, `http://[::ffff:127.0.0.1]/`, and a hostname that DNS-rebinds at fetch time.
3. Convert a string-concat SQL query to a parameterised one using `database/sql.QueryContext`. Use `EXPLAIN` to confirm the prepared statement is cached.
4. Replace a `text/template` HTML output with `html/template`. Inject `<script>alert(1)</script>` as a username and observe the difference.
5. Write a benchmark comparing `bytes.Equal` vs `subtle.ConstantTimeCompare` on a 32-byte token. Confirm the timing variance is removed.
6. Build a middleware that enforces `MaxBytesReader` and a 10s body-read deadline on every POST. Validate with `curl --limit-rate 10`.
7. Set `cmd.Env` explicitly to a minimal allow-list when invoking `git`. Confirm `GIT_TERMINAL_PROMPT=0` prevents an interactive prompt from blocking the process.
