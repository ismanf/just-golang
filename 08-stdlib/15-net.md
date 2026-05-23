# `net` — TCP, UDP, Unix sockets, DNS, `netip.Addr`

## TL;DR

`net` is the low-level networking package: `Listen`/`Dial` for TCP/UDP/Unix; `Conn`/`Listener`/`PacketConn` interfaces; DNS resolution via `LookupHost`/`Resolver`; address types `IPAddr`, `TCPAddr`, `UDPAddr`. Since 1.18, `net/netip` provides the modern value-type `netip.Addr` (no pointer, no slice; comparable; mappable) that should replace `net.IP` in new code where possible. Higher-level protocols (`net/http`, `crypto/tls`, gRPC) build on these primitives.

## Mental Model

```
Listen("tcp", ":8080")  → net.Listener
   .Accept()            → net.Conn   (io.Reader + io.Writer + Close + LocalAddr/RemoteAddr)

Dial("tcp", "1.2.3.4:80") → net.Conn

UDP:
PacketConn from ListenUDP — message-oriented; ReadFrom/WriteTo

netip.Addr (1.18+): value type, comparable, supports IPv4/v6
net.IP     (legacy): []byte; not comparable; mutable
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"io"
	"net"
)

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil { panic(err) }
	defer ln.Close()
	fmt.Println("listening on", ln.Addr())

	go func() {
		conn, _ := net.Dial("tcp", ln.Addr().String())
		conn.Write([]byte("hi\n"))
		conn.Close()
	}()

	c, _ := ln.Accept()
	b, _ := io.ReadAll(c)
	fmt.Println(string(b))
	c.Close()
	// Output (port varies):
	// listening on 127.0.0.1:54321
	// hi
}
```

## Deep Dive

### `net.Listen` and `net.Dial`

```go
ln, _ := net.Listen("tcp", ":8080")
ln, _ := net.Listen("tcp4", ":8080")  // IPv4 only
ln, _ := net.Listen("tcp6", ":8080")  // IPv6 only
ln, _ := net.Listen("unix", "/tmp/socket")

conn, _ := net.Dial("tcp", "example.com:80")
conn, _ := net.DialTimeout("tcp", addr, 5*time.Second)
```

Use `net.Dialer` with a `Timeout`, `KeepAlive`, `LocalAddr`, `Control` for full control. Use `DialContext(ctx, network, addr)` to support cancellation.

### `net.Conn` interface

```go
type Conn interface {
	Read([]byte) (int, error)
	Write([]byte) (int, error)
	Close() error
	LocalAddr() Addr
	RemoteAddr() Addr
	SetDeadline(t time.Time) error
	SetReadDeadline(t time.Time) error
	SetWriteDeadline(t time.Time) error
}
```

Implementations: `*TCPConn`, `*UDPConn`, `*UnixConn`, `*tls.Conn`, plus any wrapper your code defines.

### Deadlines vs context

`net.Conn` predates `context`; deadlines are absolute times. For new code, prefer `*Dialer.DialContext` and convert ctx to deadline:

```go
if d, ok := ctx.Deadline(); ok {
	conn.SetDeadline(d)
}
```

### `netip.Addr` — modern address type (since 1.18)

```go
import "net/netip"

addr, _ := netip.ParseAddr("192.0.2.1")
addr.Is4()
addr.Is6()
addr.IsPrivate()
addr.IsLoopback()
addr.As4()        // [4]byte
addr.AsSlice()    // []byte

addrPort := netip.AddrPortFrom(addr, 80) // host:port
prefix, _ := netip.ParsePrefix("10.0.0.0/8")
prefix.Contains(addr)
```

Properties:

- Value type (16 bytes for IPv6, 4 for IPv4 — stored compactly).
- Comparable with `==`.
- Usable as map key.
- Immutable.

Conversions:

```go
ip := net.IP(addr.AsSlice())
addr2, _ := netip.AddrFromSlice(ip)
```

### DNS

```go
ips, _ := net.LookupIP("example.com")
hosts, _ := net.LookupHost("example.com")
cname, _ := net.LookupCNAME("www.example.com")
mxs, _ := net.LookupMX("example.com")
srvs, _ := net.LookupSRV("xmpp-server", "tcp", "example.com")
txts, _ := net.LookupTXT("example.com")

// Custom resolver:
r := &net.Resolver{
	PreferGo: true, // pure-Go resolver, doesn't use cgo
	Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
		return net.Dial(network, "8.8.8.8:53")
	},
}
ips, _ := r.LookupIPAddr(ctx, "example.com")
```

### UDP

```go
conn, _ := net.ListenPacket("udp", ":8125")
buf := make([]byte, 1500)
for {
	n, addr, err := conn.ReadFrom(buf)
	if err != nil { return }
	process(buf[:n], addr)
}
```

Or `net.DialUDP` for a connected UDP (peer fixed at dial-time).

### Unix domain sockets

```go
ln, _ := net.Listen("unix", "/tmp/app.sock")
// or "unixpacket" for SOCK_SEQPACKET
conn, _ := net.Dial("unix", "/tmp/app.sock")
```

Faster than TCP for same-host IPC; supports FD passing via `*UnixConn.WriteMsgUnix`.

### TLS

Layer with `crypto/tls`:

```go
import "crypto/tls"

cfg := &tls.Config{InsecureSkipVerify: false /* etc */}
conn, _ := tls.Dial("tcp", "example.com:443", cfg)
ln, _ := tls.Listen("tcp", ":443", tlsServerCfg)
```

### Connection lifecycle and FIN/RST

- Closing a `net.Conn` sends FIN (graceful).
- `*TCPConn.SetLinger(0)` then `Close` sends RST (abortive).
- After Close, subsequent reads/writes return `net.ErrClosed`.

### `net.Error` interface

```go
var nerr net.Error
if errors.As(err, &nerr) && nerr.Timeout() {
	// timeout-specific handling
}
```

`Temporary()` is deprecated (always rely on `Timeout()` and explicit context).

## Standard Library Hooks

- `net/netip` — modern address types.
- `net/http`, `net/rpc`, `crypto/tls`, `net/smtp`, `net/mail` all built on `net.Conn`.
- `golang.org/x/net` — extensions (HTTP/2 internals, ICMP, websocket helpers, ipv4/ipv6 control).
- `context.Context` for cancellation in `DialContext`, `Resolver.LookupX`.

## Real-World Patterns

### 1. Simple TCP echo server

```go
func echo(addr string) error {
	ln, err := net.Listen("tcp", addr)
	if err != nil { return err }
	defer ln.Close()
	for {
		conn, err := ln.Accept()
		if err != nil { return err }
		go func(c net.Conn) {
			defer c.Close()
			io.Copy(c, c)
		}(conn)
	}
}
```

### 2. UDP statsd-style packet receiver

```go
func recvStatsd(addr string, h func(line string)) error {
	conn, err := net.ListenPacket("udp", addr)
	if err != nil { return err }
	defer conn.Close()
	buf := make([]byte, 65535)
	for {
		n, _, err := conn.ReadFrom(buf)
		if err != nil { return err }
		for _, line := range strings.Split(string(buf[:n]), "\n") {
			if line != "" { h(line) }
		}
	}
}
```

### 3. CIDR membership check with `netip`

```go
import "net/netip"

var allowed = []netip.Prefix{
	netip.MustParsePrefix("10.0.0.0/8"),
	netip.MustParsePrefix("192.168.0.0/16"),
}

func isAllowed(addrStr string) bool {
	a, err := netip.ParseAddr(addrStr)
	if err != nil { return false }
	for _, p := range allowed {
		if p.Contains(a) { return true }
	}
	return false
}
```

Use case: firewalls, allowlists.

### 4. Custom resolver via DoH (DNS over HTTPS)

```go
r := &net.Resolver{
	PreferGo: true,
	Dial: dohDial, // custom transport
}
ips, _ := r.LookupHost(ctx, "example.com")
```

Use case: privacy-preserving DNS, ad blockers.

### 5. Unix-socket admin endpoint

```go
sock := "/var/run/myapp.sock"
os.Remove(sock)
ln, _ := net.Listen("unix", sock)
os.Chmod(sock, 0o600)
defer ln.Close()
http.Serve(ln, adminMux)
```

Use case: privileged management interface on the same host.

## Anti-Patterns & Gotchas

**Not setting deadlines or context.** Connections hang forever on flaky peers.

**Using `net.IP` (`[]byte`) as map key.** Slices are not comparable. Use `netip.Addr`.

**Comparing `net.IP` with `==`.** Compares slice headers. Use `.Equal()`.

**`net.LookupHost` blocking in a goroutine you can't cancel.** Use `Resolver.LookupHost(ctx, ...)`.

**Default Dialer with no Timeout.** Connection attempts can hang minutes (kernel SYN retry).

**Forgetting `conn.Close()`.** FD leak.

**Reading after Close.** Returns `net.ErrClosed` (1.16+); handle it.

**Mixing `net.Dial` (blocking) with `context.Context` expectations.** Use `Dialer.DialContext`.

**Assuming `LookupIP` returns IPv4 only.** Returns both families; sort/select if you care.

**Setting `SO_REUSEPORT` etc. via raw syscall.** Use `net.ListenConfig.Control` (since 1.11) cleanly.

## Performance Notes

- The Go runtime's netpoller wraps epoll/kqueue/IOCP; goroutines blocked on `Read`/`Write` are parked until the FD is ready.
- `io.Copy(dst, src)` between `*TCPConn`s uses `splice(2)` on Linux for zero-copy — useful for proxies.
- DNS resolution caches at the OS layer (nscd, systemd-resolved); Go itself does not cache.
- UDP `WriteTo` is one syscall per packet; batch with `golang.org/x/net/ipv4` for `sendmmsg(2)`.

## How Big Companies Use It

- **Tailscale** writes its WireGuard data plane on top of `net.PacketConn` and `netip`.
- **Caddy** uses `crypto/tls` + `net.Listen` directly for its HTTPS handling.
- **Cloudflare** internal proxies (Workers, custom edge) use `net` with custom dialers for connection pooling.
- **etcd** uses Unix sockets for some local admin endpoints; gRPC over TCP for cluster communication.
- **Discord** uses `netip` for IP-range matching in their custom rate limiter.

## Source Code References

Pinned to `go1.26`.

- `net`: [`src/net/`](https://github.com/golang/go/tree/master/src/net).
- `net/netip`: [`src/net/netip/`](https://github.com/golang/go/tree/master/src/net/netip).
- Netpoller integration: [`src/runtime/netpoll.go`](https://github.com/golang/go/blob/master/src/runtime/netpoll.go).
- TLS: [`src/crypto/tls/`](https://github.com/golang/go/tree/master/src/crypto/tls).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/net, /net/netip.
- Go blog, "netaddr.IP: a new IP address type": https://tailscale.com/blog/netaddr-new-ip-type-for-go (origin of `netip`).
- Filippo Valsorda, networking deep dives.

## Exercises / Self-Check

1. Build a TCP proxy: listen on one port, dial another, copy bytes both ways.
2. Use `netip.ParsePrefix` to check whether a client IP is in `10.0.0.0/8`.
3. Demonstrate the difference between `net.IP{0x01,0x02,0x03,0x04}` comparison via `==` (compile error) vs `Equal`.
4. Write a UDP echo server. Run with `nc -u`.
5. Customize `net.Resolver` to point at `1.1.1.1`. Look up an A record.
