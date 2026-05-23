# Channels

## TL;DR

A channel is a typed, thread-safe FIFO queue with built-in blocking semantics. `make(chan T)` is unbuffered (send blocks until a receiver is ready, and vice versa); `make(chan T, n)` is buffered (sends block only when the buffer is full). The single biggest gotcha: **closing a channel is a signal, not a teardown**. A closed channel can still be received from (yielding the zero value); sending to a closed channel **panics**; closing a nil or already-closed channel **panics**. The producer closes; the consumer reads.

## Mental Model

```
                hchan (runtime.hchan)
        +----------------------------------+
        | buf:    ring buffer (cap slots)  |
        | qcount: items currently in buf   |
        | sendx, recvx: ring indexes       |
        | sendq:  parked senders           |
        | recvq:  parked receivers         |
        | lock:   runtime.mutex            |
        | closed: 0 or 1                   |
        +----------------------------------+

unbuffered: cap==0, buf==nil — a send must hand the value directly
            to a parked receiver (or park itself).

buffered:   send copies into buf if room, else parks.
            recv copies out of buf if any, else parks.
```

A channel is a pointer to a heap-allocated `hchan`. Copying a channel value is cheap and shares state. The runtime's mutex on `hchan` is short-lived; channels are O(1) lock-and-unlock per op.

## Syntax & Basic Usage

```go
package main

import "fmt"

func main() {
	unbuf := make(chan int)        // unbuffered
	buf := make(chan int, 3)       // capacity 3

	go func() {
		for v := range unbuf {     // ranges until close
			fmt.Println("got", v)
		}
		fmt.Println("done")
	}()

	unbuf <- 10
	unbuf <- 20
	close(unbuf)

	buf <- 1; buf <- 2; buf <- 3   // none block
	close(buf)
	for v := range buf {           // drains the 3 values, then exits
		fmt.Println("buf", v)
	}
	// Output (deterministic up to one interleaving):
	// got 10
	// got 20
	// done
	// buf 1
	// buf 2
	// buf 3
}
```

A `range` over a channel receives until the channel is closed and drained. The `for v := range ch` form *cannot* observe whether the channel was closed vs simply still open — it just stops when both. Use the comma-ok form (`v, ok := <-ch`) for that.

## Deep Dive

### Send and receive semantics

| State                | Send                           | Receive                         |
|----------------------|--------------------------------|---------------------------------|
| nil                  | blocks forever                 | blocks forever                  |
| open, buffer not full| copies into buf, returns       | copies out of buf, returns      |
| open, buffer full    | parks until reader takes one   | parks until sender provides one |
| open, unbuffered     | parks until matched receiver   | parks until matched sender      |
| closed               | **panic**                      | returns zero value, ok=false    |

A `nil` channel is incredibly useful in `select` for *disabling* a branch — see `03-select.md`.

### The handoff for unbuffered channels

When sender and receiver are both ready (one is parked, the other just arrived), the runtime copies the value **directly** from the sender's stack to the receiver's stack — no buffer involved. That's the "synchronous handoff" Pike talks about. Cost: one mutex lock, one memmove, one goroutine wakeup.

### The ring buffer for buffered channels

`hchan.buf` is a circular array of `cap` slots of element size `elemsize`. `sendx` and `recvx` are the head/tail indexes. The buffer is allocated inline with the `hchan` (one allocation per channel) when `elemsize` isn't huge.

### Closing semantics

```go
ch := make(chan int, 2)
ch <- 1
close(ch)
v, ok := <-ch          // v=1, ok=true
v, ok = <-ch           // v=0, ok=false (drained + closed)
```

Closing wakes every parked receiver with the zero value and `ok=false`, and every parked sender with a `send on closed channel` panic. This makes `close` an excellent **broadcast** mechanism — see Pattern #2 below.

Rules of close:
1. **Only the sender closes.** Two senders, one close — coordinate so exactly one closes.
2. Receivers never close — they have no way of knowing the sender is done.
3. If you have multiple senders and need to signal completion, use a separate "done" channel and don't close the data channel until all senders have stopped.
4. Closing nil → panic. Closing already-closed → panic. There is no "safe-close" stdlib helper; use `sync.Once`:

```go
var once sync.Once
safeClose := func() { once.Do(func() { close(ch) }) }
```

### Direction-typed channels

```go
func produce(out chan<- int) { out <- 1 }   // send-only
func consume(in <-chan int)  { <-in }       // receive-only
```

The compiler enforces direction at the function boundary. A `chan T` converts implicitly to either `chan<- T` or `<-chan T`, but not vice versa. Use this aggressively in APIs — it documents intent and prevents the wrong side from closing.

### Channel zero value

`var ch chan int` is `nil`. Sends and receives on a nil channel block **forever**. Closing a nil channel panics. The block-forever behavior is the trick `select` uses to disable cases.

### Memory model

A send on a channel happens-before the corresponding receive completes. A receive from a closed channel (returning zero) happens-after the close. This is the basis for using channels as synchronization primitives — see `09-go-memory-model.md`.

### Channel of struct{} as a signal

`make(chan struct{})` is the idiom for a *signal-only* channel. `struct{}` occupies zero bytes, so the send/receive carry no data; the synchronization is the entire point.

```go
done := make(chan struct{})
go func() {
	defer close(done)
	work()
}()
<-done // wait
```

## Standard Library Hooks

- `time.After(d) <-chan Time` — fires once. Cheap to use sparingly; leaks until the duration elapses if you abandon it. For long-running selects use `time.NewTimer` and `Stop`.
- `time.NewTicker(d)` — periodic. Must `Stop()` or the underlying goroutine leaks.
- `context.Context.Done() <-chan struct{}` — the universal cancellation signal.
- `signal.Notify(c chan<- os.Signal, ...)` — OS signals piped into a channel.
- `runtime.Gosched()` — yield the goroutine; rarely necessary near channel code because send/recv already yield.
- The reflect package: `reflect.Select` and `reflect.ChanDir`. Slow but flexible.

## Real-World Patterns

### 1. Generator with cancellation

```go
package main

import "context"

func count(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 0; ; i++ {
			select {
			case <-ctx.Done():
				return
			case out <- i:
			}
		}
	}()
	return out
}
```

The generator owns the channel and closes it on exit. The `select` with `ctx.Done()` is mandatory or the goroutine leaks if the consumer abandons it.

### 2. Done-channel broadcast

```go
type Server struct{ stop chan struct{} }

func (s *Server) Stop() { close(s.stop) }      // safe single-broadcast
func (s *Server) loop() {
	for {
		select {
		case <-s.stop:
			return
		case req := <-s.requests:
			s.handle(req)
		}
	}
}
```

Closing `s.stop` wakes every goroutine selecting on `<-s.stop`. This is the canonical Go shutdown pattern. Guard `Stop` with `sync.Once` if it can be called more than once.

### 3. Fan-in with explicit close coordination

```go
package main

import "sync"

func merge[T any](cs ...<-chan T) <-chan T {
	out := make(chan T)
	var wg sync.WaitGroup
	for _, c := range cs {
		wg.Add(1)
		go func(c <-chan T) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(c)
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

Notice the second goroutine: only **after** all sources are exhausted does `out` close. The closer holds no input channels; it only owns the output.

### 4. Pipeline stage

```go
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			out <- v * v
		}
	}()
	return out
}
```

Stages compose by chaining: `square(square(count(ctx)))`. Closing propagates downstream automatically because each stage `range`s its input and closes its output on exit. See `12-concurrency-patterns.md`.

### 5. Bounded request queue

```go
type Service struct{ queue chan Request }

func New(buf int) *Service {
	return &Service{queue: make(chan Request, buf)}
}

func (s *Service) Submit(ctx context.Context, r Request) error {
	select {
	case s.queue <- r:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

A buffered channel doubles as a backpressure mechanism: when full, callers either wait or get an explicit timeout.

## Anti-Patterns & Gotchas

**Closing from the receiver.** Don't. The sender owns the channel.

**Closing twice or closing nil.** Panic. Either prove uniqueness statically or wrap in `sync.Once`.

**Forgetting `default` in a "non-blocking try-send".**

```go
select {
case ch <- v:
default:
	// dropped
}
```

Without `default`, the send blocks until a receiver appears. With `default`, it's a one-shot best-effort.

**Using `time.After` in a hot select.** Each call allocates a new `Timer` and the underlying goroutine doesn't free until the duration elapses. In a tight loop, the channel garbage compounds. Use `time.NewTimer` + `Reset`/`Stop`.

**Goroutine leaks via unbuffered sends.** Sender parked on `ch <- v`, consumer disappears, sender lives forever. The fix is either to give the sender a way to cancel (`select { case ch <- v: case <-ctx.Done(): }`) or to buffer the channel adequately.

**Range over closed-but-still-buffered channel.** It drains the buffer first, then exits. Often misread as "stops on close."

**Using `len(ch)` for synchronization.** Racy. Between `len` and the next op, anything can happen. Use the channel itself.

**Assuming channel sends are FIFO across senders.** They are FIFO **per channel**, but if multiple senders are parked, the scheduler picks one in implementation-defined order.

**Treating channels as cheap.** Each `make(chan T)` allocates an `hchan` plus the buffer. Don't allocate one per packet in a hot loop. Reuse, or use a `sync.Pool`.

## Performance Notes

- Send/receive on an unbuffered channel: ~50–100 ns when matched, plus a scheduler context switch (~200–500 ns) if a goroutine has to park or wake.
- Buffered send/receive when buffer has room: ~30 ns, no scheduler involvement.
- A `select` with N cases is O(N) per evaluation (each case must be checked for readiness).
- `close` is O(N) in the number of parked goroutines (all of them are woken).
- Channels touch the heap. For very tight inner loops, `atomic` or a `sync.Mutex` over a slice may be 10× faster — see `15-channels-vs-mutexes.md`.
- The `hchan` struct is ~96 bytes plus `cap * elemsize` for the buffer.

## How Big Companies Use It

- **Kubernetes** uses channels extensively in `informers` and `workqueues` for event delivery: [`client-go/tools/cache`](https://github.com/kubernetes/client-go/tree/master/tools/cache).
- **etcd's raft library** uses channels for proposal queues and ready notifications. The `Node` interface is a channel-driven state machine: [`go.etcd.io/raft`](https://github.com/etcd-io/raft).
- **Prometheus** uses channels in its scrape pool and TSDB write path.
- **NATS server** (`nats-io/nats-server`) uses channels for subscription delivery — and famously moved some hot paths *off* channels to lock-free structures for throughput.
- **Bryan C. Mills (Go team) "Rethinking Classical Concurrency Patterns"** explicitly demonstrates how the "naive" generator pattern leaks and how to fix it: https://www.youtube.com/watch?v=5zXAHh5tJqQ.

## Source Code References

Pinned to `go1.26`.

- The `hchan` struct and all send/recv code: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go). Read `chansend`, `chanrecv`, `closechan`, `selectgo`.
- Compiler lowering of `<-ch` / `ch<-v`: [`src/cmd/compile/internal/ssagen/ssa.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssagen/ssa.go), search `OSEND`, `ORECV`, `OCLOSE`.
- Channel memory model wording: [`src/runtime/chan.go`](https://github.com/golang/go/blob/master/src/runtime/chan.go) header comment.
- `select` implementation: same file, function `selectgo`.

## Further Reading

- Go spec, "Send statements" and "Receive operator": https://go.dev/ref/spec#Send_statements
- Go memory model — channel communication: https://go.dev/ref/mem#chan
- Sameer Ajmani, "Go Concurrency Patterns: Pipelines and cancellation": https://go.dev/blog/pipelines
- Rob Pike, "Go Concurrency Patterns" (Google I/O 2012): https://www.youtube.com/watch?v=f6kdp27TYZs
- Bryan C. Mills, "Rethinking Classical Concurrency Patterns": https://github.com/bcmills/go-concurrency-patterns
- Dmitry Vyukov on lock-free channels (rejected proposal, instructive): https://docs.google.com/document/d/1yIAYmbvL3JxOKOjuCyon7JhW4cSv1wy5hC0ApeGMV9s/
- Russ Cox, "Go's `select` statement": https://research.swtch.com/godoc

## Exercises / Self-Check

1. What happens when you `close(nil)`? When you send to a closed channel? Receive from one?
2. Implement `SafeClose(ch chan T)` that closes only if not already closed, using `sync.Once`.
3. Why does an unbuffered channel send have to park even if a receiver is "about to" call recv?
4. Write a `merge[T any](cs ...<-chan T) <-chan T` that closes the output only after every input is drained.
5. Sketch the `hchan` state (qcount, sendx, recvx, sendq, recvq) after `make(chan int, 2); ch <- 1; ch <- 2; go func(){ <-ch }(); ch <- 3`.
