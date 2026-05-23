# WebSockets — `nhooyr/websocket` and `gorilla/websocket`

## TL;DR

WebSocket is a long-lived bidirectional message-oriented connection over a single TCP socket, upgraded from HTTP. Two Go libraries dominate: **`nhooyr.io/websocket`** (now `coder/websocket` after the maintainer joined Coder — clean, context-aware API, supports compression, ~1k LoC) and **`github.com/gorilla/websocket`** (the original, more mature, more APIs to choose between, more legacy code to find on Stack Overflow). For new code in 2026, prefer `coder/websocket`. Five operational disciplines distinguish working WebSocket apps from broken ones: **ping/pong keepalive** (browsers and proxies kill idle WebSockets after 30-60s; the lib doesn't ping for you by default), **read/write deadlines on every operation** (a slow client can hold a goroutine indefinitely), **backpressure** (slow consumer + unbuffered fan-out = server OOM), **message size limits** (default `nhooyr/websocket` is 32 KiB; protect from huge inbound frames), and **graceful close** (status code + reason; surprisingly often skipped). The single biggest gotcha across both libs: **concurrent writes to a single WebSocket connection are not safe** — you need a writer goroutine that owns the conn, with a channel inbox. WebSockets are bidirectional but the *write side* is single-producer in practice.

## Mental Model

```
   Client                                         Server
   ──────                                         ──────
       │── HTTP/1.1 GET + Upgrade: websocket ──►│
       │◄─── HTTP/1.1 101 Switching Protocols ──│
       │                                         │
       │═══════════ WebSocket frames ═══════════│
       │   text/binary, control (ping/pong/close)│
       │═══════════════════════════════════════│
       │── close frame ──────────────────────► │
       │◄────────────── close frame ─────────── │
       │── TCP FIN ──────────────────────────► │
```

Frame types:

- **Text** — UTF-8 strings.
- **Binary** — arbitrary bytes.
- **Ping/Pong** — keepalive (control frames).
- **Close** — initiates a graceful shutdown (control frame).

A "message" can span multiple frames (`continuation` frames) — fragmentation. Libraries usually hide this from you.

## Library: `coder/websocket` (formerly `nhooyr/websocket`)

```bash
go get github.com/coder/websocket
```

### Server

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "time"

    "github.com/coder/websocket"
    "github.com/coder/websocket/wsjson"
)

func wsHandler(w http.ResponseWriter, r *http.Request) {
    c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns:    []string{"app.example.com"},
        CompressionMode:   websocket.CompressionContextTakeover,
        InsecureSkipVerify: false,
    })
    if err != nil {
        slog.Error("ws accept", "err", err)
        return
    }
    defer c.CloseNow()

    c.SetReadLimit(64 * 1024)

    ctx, cancel := context.WithCancel(r.Context())
    defer cancel()

    for {
        ctx, cancelRead := context.WithTimeout(ctx, 60*time.Second)
        var msg map[string]any
        err := wsjson.Read(ctx, c, &msg)
        cancelRead()
        if err != nil {
            // includes normal close
            return
        }

        // echo
        writeCtx, cancelWrite := context.WithTimeout(ctx, 10*time.Second)
        if err := wsjson.Write(writeCtx, c, msg); err != nil {
            cancelWrite()
            return
        }
        cancelWrite()
    }
}

func main() {
    http.HandleFunc("/ws", wsHandler)
    http.ListenAndServe(":8080", nil)
}
```

Notes:

- `websocket.Accept` handles the handshake. `OriginPatterns` enforces CSRF protection — without it, any web page can open a WebSocket to your server with the user's cookies.
- `SetReadLimit` bounds the largest single message. Default 32 KiB.
- Every `Read`/`Write` takes a `context.Context` — cancellation is first-class. `gorilla/websocket` requires `SetReadDeadline` / `SetWriteDeadline` instead.
- `c.CloseNow()` is the "hard close" variant. `c.Close(code, reason)` is graceful.

### Client

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

c, _, err := websocket.Dial(ctx, "wss://api.example.com/ws", nil)
if err != nil { return err }
defer c.CloseNow()

err = wsjson.Write(ctx, c, map[string]string{"hello": "world"})
```

## Library: `gorilla/websocket`

```bash
go get github.com/gorilla/websocket
```

### Server

```go
package main

import (
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  4096,
    WriteBufferSize: 4096,
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return origin == "https://app.example.com"
    },
    EnableCompression: true,
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    c, err := upgrader.Upgrade(w, r, nil)
    if err != nil { return }
    defer c.Close()

    c.SetReadLimit(64 * 1024)
    c.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.SetPongHandler(func(string) error {
        c.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    go pinger(c)

    for {
        _, msg, err := c.ReadMessage()
        if err != nil { return }
        c.SetWriteDeadline(time.Now().Add(10 * time.Second))
        if err := c.WriteMessage(websocket.TextMessage, msg); err != nil { return }
    }
}

func pinger(c *websocket.Conn) {
    t := time.NewTicker(30 * time.Second)
    defer t.Stop()
    for range t.C {
        c.SetWriteDeadline(time.Now().Add(10 * time.Second))
        if err := c.WriteMessage(websocket.PingMessage, nil); err != nil {
            return
        }
    }
}
```

`gorilla/websocket`'s API is more imperative: you set deadlines manually, you wire the pong handler to refresh the read deadline, you run your own pinger goroutine. More boilerplate; more control.

## Ping/Pong Keepalive

Without periodic pings, intermediaries (NLBs, ALBs, Cloudflare, corporate firewalls) close idle WebSockets after 30-120s. The fix: a pinger.

```go
// nhooyr/coder
ticker := time.NewTicker(20 * time.Second)
for {
    select {
    case <-ticker.C:
        ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
        err := c.Ping(ctx)
        cancel()
        if err != nil { return }
    case <-ctx.Done():
        return
    }
}
```

`gorilla/websocket`: write a `PingMessage` and register a `SetPongHandler` to refresh the read deadline.

Browsers' WebSocket API doesn't expose ping/pong to JS — the browser handles control frames automatically. So pings from server are received-and-acked invisibly.

## Backpressure

The single most-skipped piece of WebSocket production code.

```go
type Conn struct {
    c    *websocket.Conn
    send chan []byte
}

func (cn *Conn) writePump(ctx context.Context) {
    for {
        select {
        case msg := <-cn.send:
            writeCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
            err := cn.c.Write(writeCtx, websocket.MessageText, msg)
            cancel()
            if err != nil { return }
        case <-ctx.Done():
            return
        }
    }
}

func (cn *Conn) Send(msg []byte) error {
    select {
    case cn.send <- msg:
        return nil
    default:
        return errSlowClient
    }
}
```

The bounded `send chan` is your backpressure boundary. If a client can't keep up, you drop messages (or terminate the connection). Without it, an unbounded queue grows; one slow client OOMs the server.

Strategies under backpressure:

- **Drop oldest** — for "current state" topics (`presence`, market prices).
- **Drop newest** — when sequence matters (chat); just don't lose state.
- **Disconnect slow client** — for fairness across a fleet of clients.
- **Coalesce** — for "any update wins" topics (latest cursor position).

## Concurrency Rules

- **Reads**: one goroutine. `Read` calls are serialised.
- **Writes**: one goroutine. `Write` calls are NOT safe to concurrent-call.
- **Ping/Pong from background goroutine**: write side; must coordinate with main writer via channel.
- The standard pattern: **one read goroutine, one write goroutine, channel-based fan-in for writes.**

## Subprotocols

```go
c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
    Subprotocols: []string{"graphql-transport-ws", "chat-v1"},
})
chosen := c.Subprotocol()   // "" or the negotiated one
```

Subprotocols (negotiated via `Sec-WebSocket-Protocol`) let one endpoint serve multiple protocols. The graphql-ws / graphql-transport-ws protocols are the canonical example (GraphQL subscriptions over WebSocket).

## Compression

Permessage-deflate (RFC 7692) compresses each message. `coder/websocket` supports `CompressionContextTakeover` (per-connection LZ77 window — better ratio) and `CompressionNoContextTakeover` (per-message — less memory).

```go
// nhooyr/coder
websocket.AcceptOptions{ CompressionMode: websocket.CompressionContextTakeover }

// gorilla
websocket.Upgrader{ EnableCompression: true }
```

Trade-off: compression saves bandwidth (especially for repetitive JSON) but costs CPU and memory per connection. For chat/JSON-heavy workloads it's usually a win; for already-compressed payloads (binary protobuf, images) it's overhead.

## Origin / CSRF

Browsers honour Same-Origin Policy for fetch, but **WebSockets bypass it**. Without `CheckOrigin`/`OriginPatterns`, any malicious website can open a WS to your server with the user's cookies. Always allow-list:

```go
// coder/websocket
OriginPatterns: []string{"app.example.com", "*.example.com"}

// gorilla
CheckOrigin: func(r *http.Request) bool {
    return r.Header.Get("Origin") == "https://app.example.com"
}
```

## Close Semantics

```go
// Initiate graceful close
c.Close(websocket.StatusNormalClosure, "bye")

// Coerce a close on the way out
c.CloseNow()
```

Close codes (RFC 6455):

- `1000 NormalClosure` — graceful
- `1001 GoingAway` — server shutting down, client navigating
- `1002 ProtocolError` — peer violated protocol
- `1003 UnsupportedData` — got binary in a text channel
- `1008 PolicyViolation` — generic "you broke a rule"
- `1009 MessageTooBig` — payload exceeded limit
- `1011 InternalError` — server bug
- `4000-4999` — application-specific

Always close with a code; clients can dispatch on it.

## Scaling Out

A single Go server can handle 50k-100k concurrent WebSockets per instance (each is a goroutine + small buffers; ~10-20 KB each). To scale beyond:

- **Fan-out via Redis pub/sub or NATS**: each instance subscribes to topics; publishes propagate to all subscribers' WS.
- **Sticky sessions**: load balancer routes by client ID, so reconnects land on the same backend.
- **Centralised connection registry**: not recommended; doesn't scale; just publish events.

Common architecture:

```
   Browser ───► LB ───► WS pod (one of N)
                            │
                            └── subscribes Redis topic
                                       ▲
                                       │
                                Producer ───► PUBLISH topic
                            (any service that emits events)
```

## SSE — Often Simpler Than WebSocket

If you only need **server → client** push (notifications, dashboards, live progress), **Server-Sent Events** is one-way, plain HTTP, no upgrade dance:

```go
func sse(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    rc := http.NewResponseController(w)
    for ev := range events {
        fmt.Fprintf(w, "data: %s\n\n", ev)
        rc.Flush()
    }
}
```

Browser-side:

```js
const es = new EventSource('/events');
es.onmessage = (e) => { console.log(e.data); };
```

Auto-reconnect, last-event-id resumption, simpler ops. Pick SSE for one-way; WebSocket for bidirectional.

## Anti-Patterns & Gotchas

**No `CheckOrigin`/`OriginPatterns`.** CSRF via WebSocket.

**Concurrent writes to one conn.** Garbled frames; lib panics or errors.

**No backpressure.** One slow client OOMs the server.

**No pings.** Idle conns die behind LBs/firewalls; clients can't tell when to reconnect.

**`SetReadLimit` left at defaults.** 32 KiB is small for some apps; configure based on real max message size.

**Read deadline left infinite.** Slow client holds goroutine indefinitely.

**Massive single messages.** Fragmentation isn't free; either break into multiple application-level messages or use compression.

**Stateful sessions in WS handler with no shutdown plumbing.** When the server SIGTERMs, all your sessions hang up abruptly — write a graceful close path.

**Routing WebSockets through L7 LBs without `Upgrade` support.** Some proxies strip `Upgrade`; clients fail at handshake.

**Trying to do JSON over WebSocket without a framing convention.** Either send one JSON object per message (use the lib's `wsjson` helper) or define an envelope format with type discriminator.

**Treating WS as request-response.** It's a long-lived channel; don't model it as RPC unless you build a request/response layer on top.

**Forgetting message ordering during reconnect.** Either resume from a sequence number or accept gaps.

**Sending sensitive data without TLS.** Always `wss://` in production.

## Performance Notes

- **Connection memory**: ~20-40 KB per idle conn (goroutine stack + buffers).
- **Throughput per conn**: ~100 MB/s easily over LAN; gated by frame parsing.
- **Concurrent conns per instance**: 50k-100k typical; 500k+ with tuning (file descriptors, ulimits, GOMAXPROCS).
- **Compression cost**: ~10-30 µs per message for small payloads.
- **Handshake latency**: ~1 RTT + TLS handshake; subsequent messages just one network round trip.

For >1M concurrent connections, look at:

- `epoll`-based event loops (Go's runtime already does this).
- Reduce goroutine count via merged read/write goroutines.
- Tune `SO_REUSEPORT` + multiple processes.
- Profile with `pprof goroutine` to find slow consumers.

## How Big Companies Use It

- **Slack** uses WebSockets for the real-time messaging layer; Go on backend.
- **Discord** uses WebSocket on the API gateway (originally Elixir, much Go in newer services).
- **Coder** maintains the `coder/websocket` fork (since the maintainer joined them).
- **Cloudflare** uses WebSocket extensively in Workers + Durable Objects; their Go services use stdlib + nhooyr.
- **Twitch** uses WebSocket for chat (IRC bridge + custom).
- **Plaid** uses WebSocket for streaming financial data.
- **Tailscale** uses long-poll + WS for control plane events.
- **Grafana** uses WebSocket for live dashboards (Live feature).
- **GitHub** uses WebSockets for live updates in the UI (PR refresh, notifications).

## Source Code References

- `coder/websocket`: https://github.com/coder/websocket.
- `gorilla/websocket`: https://github.com/gorilla/websocket.
- RFC 6455 (WebSocket protocol): https://www.rfc-editor.org/rfc/rfc6455.
- RFC 7692 (permessage-deflate): https://www.rfc-editor.org/rfc/rfc7692.
- `golang.org/x/net/websocket` (deprecated stdlib): https://pkg.go.dev/golang.org/x/net/websocket.
- graphql-transport-ws spec: https://github.com/enisdenjo/graphql-ws.

## Further Reading

- "WebSockets in Go" (Anmol Sethi's nhooyr blog series, archived).
- "Building a chat with WebSockets" (DigitalOcean tutorial, various languages).
- "Scaling WebSockets" (Phoenix Channels / Elixir literature — Go shop equivalents).
- Cloudflare blog on WebSockets at scale.
- "Server-Sent Events vs WebSockets" — Smashing Magazine.
- "Why we don't use WebSockets at Slack" (older post — historical context).

## Exercises / Self-Check

1. Build a WebSocket echo server using `coder/websocket`. Verify with `wscat` or `websocat`.
2. Implement a pinger that pings every 20s and disconnects the client if no pong arrives within 30s.
3. Build a chat broadcast: N clients all see each other's messages. Add a bounded send channel per client; drop messages with a "slow client" warning instead of blocking the broadcaster.
4. Enforce `OriginPatterns` for `app.example.com`. Test from a different origin; confirm rejection.
5. Add compression. Compare bandwidth for a sample 10-message conversation.
6. Implement graceful shutdown: on SIGTERM, send `StatusGoingAway` to every client and wait up to 5s for them to disconnect.
7. Build an SSE alternative for one-way notifications. Compare implementation complexity.
8. Stress-test with 10k concurrent WebSockets. Measure memory and CPU; tune accordingly.
