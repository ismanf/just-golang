# `hash/maphash`, `net/netip`, `expvar` — Smaller-but-Important Packages

## TL;DR

Three packages that punch above their weight: `hash/maphash` is the fast, randomized hash Go uses internally for maps — exposed so user code can build its own hash tables and structures with the same per-process seed. `net/netip` (covered in `15-net.md` too) is the modern value-type IP address — use it everywhere instead of `net.IP`. `expvar` is the dead-simple JSON metrics endpoint at `/debug/vars` — usable as a poor-man's Prometheus when you don't want the dependency.

## Mental Model

```
hash/maphash:
   Hash struct; SetSeed; Write; Sum64.
   Per-process random seed prevents HashDoS.
   ~1 ns/byte; fast enough for custom data structures.

net/netip:
   Addr (4 or 16 bytes), AddrPort, Prefix; all value types; comparable; immutable.
   Use over net.IP.

expvar:
   Var interface with String() string method.
   /debug/vars endpoint serves all registered vars as JSON.
   Int, Float, Map, String, Func built-in.
```

## Syntax & Basic Usage

```go
package main

import (
	"expvar"
	"fmt"
	"hash/maphash"
	"net/http"
	"net/netip"
)

var reqCount = expvar.NewInt("requests_total")

func main() {
	// maphash
	var h maphash.Hash
	h.WriteString("hello")
	fmt.Println(h.Sum64())

	// netip
	a, _ := netip.ParseAddr("192.0.2.1")
	p, _ := netip.ParsePrefix("192.0.2.0/24")
	fmt.Println(p.Contains(a))

	// expvar — register a handler somewhere; /debug/vars exposes JSON
	go http.ListenAndServe(":8080", nil)
	reqCount.Add(1)

	// curl localhost:8080/debug/vars
}
```

## Deep Dive

### `hash/maphash`

```go
type Hash struct{ /* unexported */ }
func (h *Hash) WriteByte(b byte) error
func (h *Hash) WriteString(s string) (int, error)
func (h *Hash) Write(b []byte) (int, error)
func (h *Hash) Sum64() uint64
func (h *Hash) Reset()
func (h *Hash) Seed() Seed
func (h *Hash) SetSeed(s Seed)
func MakeSeed() Seed
```

Properties:

- Same seed → same hash; different seeds → unpredictable hashes.
- Per-process default seed makes hashing input-independent across processes.
- Throughput: ~10 GB/s on modern x86.
- Not cryptographic (use SHA-256 if you need that).

1.19+ added `Hash.WriteComparable` / `maphash.Comparable` / `maphash.WriteComparable` for hashing arbitrary comparable values:

```go
type Key struct{ A int; B string }
sum := maphash.Comparable(seed, Key{1, "hi"})
```

### `net/netip`

Recap from `15-net.md`:

- `Addr` — IPv4 or IPv6, 4 or 16 bytes inline.
- `AddrPort` — Addr + port + flow info.
- `Prefix` — Addr + prefix length.
- All value types, comparable, immutable, map-key-friendly.

Common ops:

- `ParseAddr`, `ParseAddrPort`, `ParsePrefix`.
- `Is4`, `Is6`, `Is4In6`.
- `IsLoopback`, `IsPrivate`, `IsMulticast`, `IsLinkLocalUnicast`.
- `As4`, `As16`, `AsSlice`.
- `Prefix.Contains(Addr)`.

Use everywhere; `net.IP` is legacy.

### `expvar`

```go
var (
	reqCount = expvar.NewInt("http_requests_total")
	openConn = expvar.NewInt("connections_open")
	cfg      = expvar.NewString("config_version")
	dynLabel = expvar.NewMap("labels")
)

func init() {
	cfg.Set("v1.2.3")
	dynLabel.Add("us-east", 1)
	dynLabel.Add("us-west", 0)
}

// /debug/vars endpoint is registered automatically when you import "expvar".
// Returns JSON: {"http_requests_total": 42, "config_version": "v1.2.3", ...}
```

Custom `Var` via `expvar.Publish(name, var)`:

```go
expvar.Publish("uptime", expvar.Func(func() any {
	return time.Since(start).Seconds()
}))
```

Properties:

- One-line registration.
- JSON-only output; no Prometheus or OTel format.
- Mostly useful for ops dashboards reading the JSON, or for the `net/http/pprof`-style debug endpoints.
- No labels/tags (it's not Prometheus).

## Standard Library Hooks

- `hash` interface — `maphash.Hash` implements it.
- `net/http` — `expvar` registers `/debug/vars` on `http.DefaultServeMux`.
- `net.IP` interop with `netip` via `netip.AddrFromSlice` / `Addr.AsSlice`.

## Real-World Patterns

### 1. Custom hash table with `maphash`

```go
type CustomMap[K comparable, V any] struct {
	seed  maphash.Seed
	bucks [][]entry[K, V]
}

func (m *CustomMap[K, V]) hash(k K) uint64 {
	return maphash.Comparable(m.seed, k)
}
```

Use case: specialized concurrent maps, perfect-hash tables.

### 2. IP allow-list

```go
import "net/netip"

var allow = []netip.Prefix{
	netip.MustParsePrefix("10.0.0.0/8"),
	netip.MustParsePrefix("192.168.0.0/16"),
}

func allowed(ipStr string) bool {
	a, err := netip.ParseAddr(ipStr)
	if err != nil { return false }
	for _, p := range allow { if p.Contains(a) { return true } }
	return false
}
```

### 3. Quick `/debug/vars` metrics

```go
import _ "net/http/pprof" // also adds /debug/pprof
import "expvar"

var queueDepth = expvar.NewInt("queue_depth")

func enqueue(j Job) { queueDepth.Add(1); /* ... */ }
func dequeue() Job  { queueDepth.Add(-1); return jobs[0] }
```

Endpoint: `curl localhost:6060/debug/vars`.

### 4. Bloom filter using `maphash`

```go
type Bloom struct {
	seed [4]maphash.Seed
	bits []uint64
}
func (b *Bloom) Add(s string) {
	for _, s2 := range b.seed {
		var h maphash.Hash; h.SetSeed(s2); h.WriteString(s)
		i := h.Sum64() % uint64(len(b.bits)*64)
		b.bits[i/64] |= 1 << (i % 64)
	}
}
```

### 5. `netip.AddrPort` map for connection state

```go
var conns = map[netip.AddrPort]*Conn{}
```

Map-keyable thanks to value type — `net.UDPAddr` would not work.

## Anti-Patterns & Gotchas

**Using `maphash` for cryptography.** Use `crypto/sha256` etc.

**Sharing `maphash.Hash` across goroutines.** Not safe.

**Using `net.IP` in new code.** Migrate to `netip`.

**Using `expvar` as a metrics replacement.** It's for ops debug; for proper metrics use Prometheus client_golang.

**Registering many `expvar` variables and forgetting they appear in JSON output.** Be selective; the endpoint can balloon.

**Reading `/debug/vars` over the public internet.** Leaks internal state.

## Performance Notes

- `maphash`: ~10 GB/s on x86 with hardware accel; fastest non-cryptographic stdlib hash.
- `netip.Addr` is 16 bytes (compact); operations are pure value moves.
- `expvar` writes are atomic where possible (`Int.Add` uses `atomic.AddInt64`).
- JSON serialization of `/debug/vars` is reflection-based; not for high QPS scraping.

## How Big Companies Use It

- **Tailscale** uses `netip` everywhere; their `netaddr` package was the upstream proposal that became stdlib `netip`.
- **etcd** uses `expvar` for some internal counters (in addition to Prometheus).
- **Cockroach** uses `maphash` internally for some hash-shaped indexes.
- **Stdlib `net/http/pprof`** registers under `/debug/`, often alongside `/debug/vars`.

## Source Code References

Pinned to `go1.26`.

- `hash/maphash`: [`src/hash/maphash/maphash.go`](https://github.com/golang/go/blob/master/src/hash/maphash/maphash.go).
- `net/netip`: [`src/net/netip/`](https://github.com/golang/go/tree/master/src/net/netip).
- `expvar`: [`src/expvar/expvar.go`](https://github.com/golang/go/blob/master/src/expvar/expvar.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/hash/maphash, /net/netip, /expvar.
- Tailscale blog, origin of netip: https://tailscale.com/blog/netaddr-new-ip-type-for-go.
- Go blog, "expvar revisited" (community posts).

## Exercises / Self-Check

1. Build a Bloom filter using `maphash` with four seeds. Test false positive rate.
2. Replace `net.IP` usage in a small package with `netip.Addr`. Note where APIs changed.
3. Register two `expvar.Int` counters and one `expvar.Func` that returns uptime. Hit `/debug/vars`.
4. Hash a struct key with `maphash.Comparable` and use the result to bucket entries.
5. Why is `net.IP` not usable as a map key but `netip.Addr` is?
