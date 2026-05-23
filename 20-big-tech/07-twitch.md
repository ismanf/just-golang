# Twitch — Chat at Scale, the Go GC Pioneer Story

## TL;DR

**Twitch** built one of the first widely-cited Go services at multi-million-concurrent-user scale: their **IRC-based chat infrastructure**. Their 2016 blog post "Go's march to low-latency GC" was a landmark — it documented pre-1.5 GC pauses of *seconds* on heap sizes of ~30 GiB, and how iterating with the Go team (and with Rick Hudson's team) reduced p99 pauses to single-digit milliseconds. Twitch's chat service was reported as one of the largest single-process Go deployments of its era, with **millions of simultaneous WebSocket/IRC connections** across geographic regions. The single biggest gotcha: **Twitch's chat scale exposed Go's pre-1.5 GC limits years before everyone else**, which is why their feedback shaped Go's 1.5+ concurrent GC and 1.8's hybrid write barrier.

## Mental Model

```
   Twitch Chat (simplified):
   
   Millions of viewers
        │  WebSocket / IRC over TCP
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │  Chat edge servers (Go)                                  │
   │   - terminate WebSocket/IRC                              │
   │   - 100k+ concurrent connections per process              │
   │   - publish/subscribe to a "fanout" tier                  │
   │   - typed messages in/out                                 │
   └──────────┬──────────────────────────────────────────────┘
              ▼
   ┌─────────────────────────────────────────────────────────┐
   │  Fanout / Pubsub tier (Go)                               │
   │   - one process per channel/shard                        │
   │   - delivers messages to all subscribers                 │
   │   - rate limit, moderation hooks                         │
   └──────────┬──────────────────────────────────────────────┘
              ▼
   ┌─────────────────────────────────────────────────────────┐
   │  Persistence + analytics                                  │
   │   - moderation actions logged                             │
   │   - chat replay (for VODs)                                │
   └─────────────────────────────────────────────────────────┘
```

Chat's defining characteristics: long-lived TCP connections (hours), bursty fanout (one streamer typing → thousands of clients receive), unforgiving tail-latency expectations (chat must feel real-time).

## Syntax & Basic Usage

A skeleton chat server (websocket fanout):

```go
package main

import (
	"context"
	"fmt"
	"sync"

	"github.com/coder/websocket" // modern fork of nhooyr.io/websocket
	"net/http"
)

type Hub struct {
	mu      sync.RWMutex
	clients map[*websocket.Conn]struct{}
}

func NewHub() *Hub { return &Hub{clients: map[*websocket.Conn]struct{}{}} }

func (h *Hub) Add(c *websocket.Conn) {
	h.mu.Lock()
	h.clients[c] = struct{}{}
	h.mu.Unlock()
}

func (h *Hub) Remove(c *websocket.Conn) {
	h.mu.Lock()
	delete(h.clients, c)
	h.mu.Unlock()
}

func (h *Hub) Broadcast(ctx context.Context, msg []byte) {
	h.mu.RLock()
	defer h.mu.RUnlock()
	for c := range h.clients {
		_ = c.Write(ctx, websocket.MessageText, msg)
	}
}

func main() {
	hub := NewHub()
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		c, err := websocket.Accept(w, r, nil)
		if err != nil { return }
		hub.Add(c)
		defer hub.Remove(c)

		for {
			_, msg, err := c.Read(r.Context())
			if err != nil { return }
			hub.Broadcast(r.Context(), msg)
		}
	})
	fmt.Println("listening :8080")
	http.ListenAndServe(":8080", nil)
}
```

Crude but represents the basic fanout model.

## Deep Dive

### Why Go (2014–2015 at Twitch)

Twitch's pre-Go chat was IRC-based, with a mix of C and Ruby. Bottlenecks:
- Ruby couldn't hold millions of connections per node.
- C required intricate concurrency code.
- Node.js was considered; rejected for tail-latency reasons.

Go offered:
- Lightweight goroutines: one per connection at the time.
- Strong stdlib networking.
- Reasonable memory cost per goroutine.
- Fast compile + deploy cycle.

The migration was gradual. The Twitch team published multiple posts as they hit and resolved scaling issues.

### The GC saga

Twitch's published 2016 article "Go's march to low-latency GC" (https://blog.twitch.tv/en/2016/07/05/gos-march-to-low-latency-gc-a6fa96f06eb7/) chronicles the pre-1.5 era and its resolution:

#### Pre-Go-1.5: stop-the-world GC

Heap of ~25 GiB; GC pauses of multiple **seconds**. Chat would freeze; clients timed out.

#### Go 1.5: concurrent mark+sweep

Pauses dropped to ~hundreds of ms. Still painful at p99.

#### Go 1.6: pacer improvements

Pauses ~50–100 ms.

#### Go 1.8: hybrid write barrier (Twitch was a prominent feedback source)

Pauses dropped to <10 ms on Twitch's heap. **This was the watershed**. Chat suddenly behaved.

Twitch's data shaped the case for the 1.8 change. The post:

> "We had a 30+ GiB heap. Pre-1.5, that meant 100ms+ pauses. After 1.8, we routinely see sub-millisecond pauses on the same heap."

#### Go 1.14+: async preemption + low STW

Pauses are routinely <1 ms; the chat tier no longer publishes about GC.

### Heap composition

Their chat connections each held some state: user identity, channel subscriptions, rate-limit counters, send buffers. At 100k connections × ~50 KiB each = 5 GiB just connection state. Plus message buffers, pubsub registrations, moderation maps. The 25 GiB total was realistic.

### Goroutines per connection

Pre-1.14, chat servers ran a few goroutines per connection (read, write, idle timer). At 100k connections × ~3 goroutines = 300k goroutines. Each cost ~8 KiB minimum stack = ~2.4 GiB just stack.

Twitch documented goroutine accounting and the impact of stack-shrink behavior. With 1.14's async preemption, stack growth/shrink rebalance more frequently, easing the total stack footprint.

### The fanout problem

When a streamer types one message in their channel of 100k viewers, the fanout tier copies that message to 100k WebSocket buffers. Naively: 100k * O(message size) writes per second.

Twitch's pattern:
- **Per-channel rooms**: each channel has its own goroutine / shard.
- **Bounded outbound queues**: per-connection.
- **Slow-consumer disconnect**: if a client can't keep up, drop them.
- **Coalescing**: batch small messages within ~10 ms windows.

### IRC bridge

Twitch's chat speaks IRC (yes, IRC) externally — IRC clients can connect directly. Internally the protocol is more compact. The bridge translates.

### Modern Twitch

Twitch was acquired by Amazon in 2014. Their internal architecture has continued evolving:
- Some services moved to AWS-managed (Lambda, MSK Kafka).
- Some services moved to Rust for ultra-low-latency paths.
- Chat itself remained in Go through multiple Go versions.

The Twitch team has not published as much in recent years (post-acquisition norms), but recent GopherCon talks indicate Go remains dominant for chat-tier services.

### Concrete patterns Twitch documented

#### 1. One goroutine per connection, with bounded send buffers

```go
type Conn struct {
    c    net.Conn
    send chan []byte // bounded
}

func (cn *Conn) writer() {
    for msg := range cn.send {
        cn.c.SetWriteDeadline(time.Now().Add(5 * time.Second))
        if _, err := cn.c.Write(msg); err != nil {
            return
        }
    }
}

func (cn *Conn) Send(msg []byte) {
    select {
    case cn.send <- msg:
    default:
        // buffer full; drop the connection
        cn.c.Close()
    }
}
```

The bounded channel + `default` in select implements "slow consumer disconnect".

#### 2. Per-channel hub goroutine

Each chat channel has a goroutine owning its subscriber set. Sends are channel-to-hub-goroutine, fanned out.

#### 3. Adapter pattern for protocols

The same internal chat representation is exposed as IRC for legacy clients, WebSocket-binary for app clients, gRPC for internal services.

#### 4. Heavy use of `sync.Pool` for message buffers

A "chat message" object is born, transmitted to N readers, and dies. With a sync.Pool, these don't allocate.

### Memory model

Twitch documented careful attention to:
- **`noscan` types**: structs without pointers where possible.
- **byte arenas** for messages.
- **Avoid `string` in hot structs** when bytes suffice.
- **Channel send vs direct write**: channels add allocation; direct unbuffered writer goroutines preferred.

## Standard Library Hooks

- `net`, `net/http`: connection management.
- `sync` + `sync/atomic`: hub state.
- `bufio`: buffered I/O on connections.
- `context`: connection lifecycle.
- `golang.org/x/net/websocket` (legacy) / `nhooyr.io/websocket` / `coder/websocket`: WebSocket libs.
- `runtime/pprof`: continuous profiling.
- `runtime/metrics`: GC + scheduler observability.
- `golang.org/x/sync`: errgroup for fanout.

## Real-World Patterns

### 1. Heartbeat / idle timeout

```go
func reader(c net.Conn, fanout chan<- []byte) {
    bufr := bufio.NewReader(c)
    for {
        c.SetReadDeadline(time.Now().Add(60 * time.Second))
        line, err := bufr.ReadBytes('\n')
        if err != nil { return }
        fanout <- line
    }
}
```

Idle clients are dropped after 60s. Prevents zombie connections.

### 2. Fanout via channel-of-subscribers

```go
type Channel struct {
    in   chan []byte
    subs map[*Conn]struct{}
}

func (ch *Channel) run() {
    for msg := range ch.in {
        for sub := range ch.subs {
            sub.Send(msg) // bounded
        }
    }
}
```

The room's goroutine serializes sends; subscribers each get the message via their own bounded channel.

### 3. Coalesce fast bursts

```go
func writer(c net.Conn, src <-chan []byte) {
    var buf bytes.Buffer
    timer := time.NewTimer(10 * time.Millisecond)
    timer.Stop()

    for msg := range src {
        buf.Write(msg)
        timer.Reset(10 * time.Millisecond)
        flush := false
        select {
        case <-timer.C:
            flush = true
        default:
        }
        if buf.Len() > 4096 || flush {
            c.Write(buf.Bytes())
            buf.Reset()
        }
    }
}
```

Pseudocode: collect messages for ~10 ms, then flush. Reduces syscall rate.

### 4. Slow-consumer disconnect

```go
select {
case sub.send <- msg:
default:
    log.Warn("slow consumer; disconnecting")
    sub.Close()
}
```

If the bounded outbound channel is full, the consumer is slow. Drop them rather than back-pressure the entire room.

### 5. Per-room shard supervision

```go
type Hub struct {
    rooms map[string]*Channel
}

func (h *Hub) Subscribe(roomID string, c *Conn) {
    ch, ok := h.rooms[roomID]
    if !ok {
        ch = &Channel{in: make(chan []byte, 1024)}
        go ch.run()
        h.rooms[roomID] = ch
    }
    ch.subs[c] = struct{}{}
}
```

Each room is its own goroutine + map. Resource ownership stays local.

## Anti-Patterns & Gotchas

**Unbounded send buffers per connection.** OOM on slow consumers.

**Storing message bodies as `string`** in long-lived caches. Each adds GC scan cost. Use `[]byte`.

**Goroutine-per-message-per-subscriber.** Drops latency unpredictably under burst.

**Synchronous fan-out write loop** that blocks on the slowest subscriber. Drag the entire room down. Always use bounded send channels with default-case drop.

**Holding a `*http.Request` past the response.** Memory ramps; pre-1.21 was a common gotcha.

**Forgetting `SetReadDeadline`.** Connections held by silent peers leak forever.

**Custom message framing without sentinel / length-prefix.** Hard to debug; use established protocols (WebSocket, length-prefix gRPC, etc.).

**Allocating per message.** Use `sync.Pool` for `*Message` and `[]byte` buffers.

**Treating GC pauses as "improving themselves".** Profile with `GODEBUG=gctrace=1`; characterize behavior under load.

**Trusting WebSocket library defaults for production.** Buffer sizes, compression, ping/pong all need tuning at Twitch-scale.

## Performance Notes

(Estimates from Twitch's published material; varies.)

- Connections per node: 100k+ in production.
- Memory per node: 25–50 GiB total heap (mostly chat state).
- Messages/sec per node: hundreds of thousands during big streams.
- p99 GC pause (post-1.8): <2 ms.
- Goroutines per node: ~300k+ at full load.
- Message coalescing window: ~10 ms.

## How Big Companies Use It

The chat-scale pattern Twitch documented is used by:

- **Discord**: see `20-big-tech/06-discord-state-service.md`.
- **Slack**: Real-time messaging server in mixed languages; chat-fan-out parts informed by Twitch.
- **WhatsApp**: Erlang-based, but operational metrics align.
- **Tinder, Bumble**: WebSocket scaling, partly Go.
- **Twitch's parent Amazon**: Lambda/MSK/Connect borrow some patterns.
- **Streamlabs**: chat overlay services, Go.
- **TikTok Live**: documented similar architecture at GopherCon China.

## Source Code References

Twitch's chat code isn't open source. Related public Go projects:

- `coder/websocket` (modern WebSocket lib, fork of nhooyr.io/websocket): [`github.com/coder/websocket`](https://github.com/coder/websocket).
- `gorilla/websocket` (long-standing alternative): [`github.com/gorilla/websocket`](https://github.com/gorilla/websocket).
- `centrifugal/centrifugo` (open-source Go real-time messaging server inspired by Twitch architecture): [`github.com/centrifugal/centrifugo`](https://github.com/centrifugal/centrifugo).
- `nats-io/nats-server` (cloud pubsub, similar fanout): [`github.com/nats-io/nats-server`](https://github.com/nats-io/nats-server).
- Twitch open-source repos: [`github.com/twitchtv`](https://github.com/twitchtv).
- `twirp` (Twitch's RPC framework): [`twitchtv/twirp`](https://github.com/twitchtv/twirp).
- Their gRPC alternative `twirp` shows the kind of stack they preferred.

## Further Reading

- "Go's march to low-latency GC" (the famous post): https://blog.twitch.tv/en/2016/07/05/.
- "How Twitch handles 1 million concurrent users on Go" — GopherCon 2016 talks.
- Rhys Hiltner, ex-Twitch, "An ode to the garbage collector" — podcasts: https://about.sourcegraph.com/podcast/rhys-hiltner.
- "Go 1.5 concurrent GC pacing" — Austin Clements: https://golang.org/s/go15gcpacer.
- "Rebuilding Twitch chat" — various Twitch tech-blog posts: https://blog.twitch.tv.
- Rick Hudson, "Go GC: Solving the latency problem" (GopherCon 2015): https://www.youtube.com/watch?v=aiv1JOfMjm0.
- centrifugo design docs: https://centrifugal.dev.
- Centrifuge architectural posts.
- Andy Wilkinson, "Building Slack's real-time messaging at scale" — adjacent: https://slack.engineering/.

## Exercises / Self-Check

1. Implement a minimal IRC-style chat hub in Go. Spawn 10k client goroutines and benchmark message fanout latency. Where's the bottleneck?
2. Compare the GC behavior of a 1M-`*Message` cache vs a 1M-flat-Message arena. Measure with `GODEBUG=gctrace=1`.
3. The Twitch post documents pause-time drops at Go 1.5, 1.6, 1.8. What changed in each release that caused this?
4. Implement slow-consumer disconnect using a bounded send channel. Why is "select with default" the right way to detect a full buffer? What's the alternative?
5. Twitch coalesces messages over a ~10 ms window. Prototype this. Quantify the throughput vs latency trade-off as the window grows from 1 ms to 100 ms.
