# `time` — Times, Durations, Clocks, Timers

## TL;DR

`time.Time` carries an instant + a `*Location` + (since 1.9) a monotonic reading. `time.Duration` is `int64` nanoseconds. Format/parse using the reference time `Mon Jan 2 15:04:05 MST 2006`. Use `time.Since`/`time.Until` for elapsed measurement (they use the monotonic clock); never subtract wall-clock times from external sources. The `time.Timer` and `time.Ticker` APIs leak goroutines if not stopped — since 1.23, `Timer` no longer requires draining its channel before reset.

## Mental Model

```
time.Time:
  wall (seconds since 1885 + nanoseconds) + monotonic (since process start) + *Location
                                              ↑ for accurate elapsed measurements

time.Duration = int64 nanoseconds
                e.g., 250 * time.Millisecond is just 250_000_000

Reference layout: "Mon Jan 2 15:04:05 MST 2006"  ← 01/02 03:04:05 PM '06 -0700
                  this single canonical date is the format string template
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()
	fmt.Println(now.Format(time.RFC3339))

	start := time.Now()
	time.Sleep(10 * time.Millisecond)
	fmt.Println("elapsed:", time.Since(start))

	t, err := time.Parse("2006-01-02", "2026-01-15")
	if err != nil { panic(err) }
	fmt.Println(t)
	// Output (varies):
	// 2026-05-21T10:30:00Z
	// elapsed: 10.123ms
	// 2026-01-15 00:00:00 +0000 UTC
}
```

## Deep Dive

### The reference time

`Mon Jan 2 15:04:05 MST 2006` = `01/02 03:04:05 PM '06 -0700`. The numeric memory aid: `1 2 3 4 5 6 7`. Every layout uses pieces of this exact date.

Common layouts (constants):

```go
time.RFC3339         // "2006-01-02T15:04:05Z07:00"
time.RFC3339Nano
time.RFC1123         // HTTP date format
time.DateTime        // "2006-01-02 15:04:05"  (since 1.20)
time.DateOnly        // "2006-01-02"           (since 1.20)
time.TimeOnly        // "15:04:05"             (since 1.20)
```

### Time zones

```go
loc, _ := time.LoadLocation("America/New_York")
t := time.Date(2026, time.July, 4, 12, 0, 0, 0, loc)
fmt.Println(t.UTC()) // converts to UTC
```

`time.UTC` and `time.Local` are predefined. `time.LoadLocation` reads the system tz database. In Docker images, you may need to install `tzdata` — or use the `time/tzdata` package which embeds the database.

```go
import _ "time/tzdata" // adds ~500 KiB to binary; LoadLocation works without OS tz files
```

### Monotonic clock

Since 1.9, `time.Now()` reads both wall and monotonic clocks. Subtraction (`t2.Sub(t1)`) uses the monotonic delta — robust against NTP corrections, leap seconds, system sleep.

The monotonic reading is *stripped* by:

- `t.Round`, `t.Truncate` (which "wall-only" the time).
- Marshal/unmarshal (JSON, gob, text).
- `t.UTC`, `t.Local`, `t.In(loc)` (since 1.9 these keep monotonic, but `In(loc)` results from network sources won't have one).

Always measure elapsed with `time.Since(start)`, never `wallNow.Sub(wallStart)` from different machines.

### `time.Duration`

```go
d := 250 * time.Millisecond   // 250_000_000 (int64)
d.Seconds()                   // 0.25 (float64)
d.Milliseconds()              // 250 (int64)
d.Truncate(time.Second)       // round down

// Parse user input:
d, err := time.ParseDuration("2h45m")
```

`ParseDuration` accepts `ns`, `us`/`µs`, `ms`, `s`, `m`, `h`. Larger units must be expressed as combinations (no `d` for day).

### `time.Sleep`, `Timer`, `Ticker`

```go
time.Sleep(100 * time.Millisecond) // park the goroutine

t := time.NewTimer(2 * time.Second)
<-t.C // fires once

tk := time.NewTicker(1 * time.Second)
defer tk.Stop()
for {
	<-tk.C
	work()
}
```

**1.23 changes:**

- `Timer.Reset(d)` no longer requires you to drain the channel first. Pre-1.23 you needed: `if !t.Stop() { <-t.C }; t.Reset(d)`. Now: `t.Reset(d)` is enough.
- `Timer` and `Ticker` channels are now garbage-collected when unreferenced (no leak even if you forget to `Stop`). Still call `Stop` for promptness, but the GC won't pin them.

### `time.AfterFunc` and `time.After`

```go
time.AfterFunc(5*time.Second, func() { log.Println("late") })

select {
case <-someCh:
case <-time.After(1 * time.Second): // timeout
}
```

`time.After` allocates a timer per call; in a hot select, prefer `time.NewTimer` + `Stop`.

### Formatting and parsing

```go
t.Format("2006-01-02 15:04:05.000")  // ".000" preserves trailing zeros
t.Format("2006-01-02 15:04:05.999")  // ".999" trims trailing zeros

time.Parse(layout, value)            // strict
time.ParseInLocation(layout, value, loc) // assume value is in loc if no tz in layout
```

### Comparison

```go
t1.Equal(t2)   // value-equal ignoring monotonic
t1 == t2        // also compares monotonic + Location — usually NOT what you want
t1.Before(t2)
t1.After(t2)
```

Use `Equal`/`Before`/`After`, not `==`.

### Truncation and rounding

```go
t.Truncate(time.Hour)              // round down
t.Round(time.Minute)               // banker's rounding to nearest
d.Truncate(100 * time.Millisecond)
```

## Standard Library Hooks

- `context.WithTimeout` / `context.WithDeadline` for cancellation by time.
- `time.Time` is `JSON`-encoded as RFC3339 by `encoding/json`.
- `time.Duration` is encoded as a number (nanoseconds) by JSON — typically you marshal it manually.
- `time.NewTimer` / `Ticker` work with `select`.
- `time/tzdata` embeds the IANA tz database.
- `testing/synctest` (stable in 1.25) lets you write deterministic time-based tests by replacing the clock.

## Real-World Patterns

### 1. Timeout via context

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
res, err := db.QueryContext(ctx, "SELECT ...")
```

Use case: every external call.

### 2. Periodic background work

```go
func worker(ctx context.Context) {
	tk := time.NewTicker(30 * time.Second)
	defer tk.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-tk.C:
			doWork()
		}
	}
}
```

Use case: metric flushers, cache refreshers.

### 3. Retry with exponential backoff

```go
backoff := 100 * time.Millisecond
for attempt := 0; attempt < 5; attempt++ {
	if err := op(); err == nil { return nil }
	time.Sleep(backoff)
	backoff = min(backoff*2, 10*time.Second)
}
```

Use case: outbound API calls, queue consumers.

### 4. Rate limit with `time.Ticker`

```go
limiter := time.NewTicker(time.Second / 100) // 100/sec
defer limiter.Stop()
for req := range queue {
	<-limiter.C
	send(req)
}
```

For more sophisticated limiting, use `golang.org/x/time/rate`.

### 5. Deterministic time in tests (1.25+)

```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
	synctest.Run(func() {
		ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
		defer cancel()
		time.Sleep(2 * time.Second) // synctest advances virtual time
		// ctx.Err() is DeadlineExceeded
	})
}
```

Use case: testing timeouts without real waits.

## Anti-Patterns & Gotchas

**Subtracting wall-clock times across machines.** Use a single source.

**Storing `time.Time` in serialized form and expecting monotonic to survive.** It doesn't — serialize stores only wall.

**`time.Now().Sub(start)` instead of `time.Since(start)`.** Same result but Since is clearer.

**Comparing times with `==`.** Use `Equal`. `==` compares monotonic + Location too.

**`time.After` inside a select in a loop.** Allocates a timer per iteration. Use `time.NewTimer` + manual reset.

**Forgetting `defer ticker.Stop()`.** Pre-1.23 leaked. 1.23+ is GC-safe but still wasteful.

**Parsing dates with the wrong layout.** `2006-01-02 15:04:05` vs `2006/01/02`. Test with edge dates (Jan, Dec, leap years).

**Assuming `time.LoadLocation` works on stripped Docker images.** Either install `tzdata` or import `time/tzdata`.

**Using `time.Duration` for clock offsets that need negative values.** Duration *is* signed; this works, but make sure you mean signed.

**Sleep inside HTTP handler.** Blocks the goroutine and the response. Use context with timeout.

**`time.Parse("2006-01-02", input)` for user input.** Returns UTC; user may have meant local. Use `ParseInLocation`.

## Performance Notes

- `time.Now()` is one VDSO call on Linux (~20 ns).
- Parsing date strings: ~100-500 ns depending on layout complexity.
- `time.Tick` (without `Stop`-able handle) leaks until 1.23; even now, prefer `NewTicker`.
- `time.Duration` arithmetic is integer ops; very fast.
- `LoadLocation("UTC")` is special-cased; non-UTC zones load and cache tz data.

## How Big Companies Use It

- **Kubernetes** uses `time.AfterFunc` for resource leases and `context.WithDeadline` for API call timeouts.
- **etcd** uses monotonic time exclusively for leader election timeouts (NTP corrections would otherwise break consensus).
- **Cockroach** has its own `hlc` (hybrid logical clock) on top of `time.Now`; for distributed-SQL ordering they cannot trust pure wall time.
- **Grafana** uses `time/tzdata` to ship a self-contained binary with full tz support.
- **Discord** documented their `time.Tick`-everywhere bug that leaked goroutines — leading to a careful rewrite using `NewTicker`.

## Source Code References

Pinned to `go1.26`.

- `time` package: [`src/time/time.go`](https://github.com/golang/go/blob/master/src/time/time.go).
- Format/parse: [`src/time/format.go`](https://github.com/golang/go/blob/master/src/time/format.go).
- Tick/Timer: [`src/time/tick.go`](https://github.com/golang/go/blob/master/src/time/tick.go), `sleep.go`.
- 1.23 timer changes: https://go.dev/doc/go1.23#timer-changes.
- `time/tzdata`: [`src/time/tzdata/`](https://github.com/golang/go/tree/master/src/time/tzdata).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/time.
- Go blog, "Time, Time Zones, and Monotonic Clocks": https://go.dev/blog/monotonic.
- Go 1.23 release notes (timer GC): https://go.dev/doc/go1.23.
- Russ Cox, on the time layout choice: https://research.swtch.com/godata.

## Exercises / Self-Check

1. Implement a backoff helper that retries up to N times with exponential delay capped at 30s.
2. Why does `t.Format("Mon Jan 2 15:04:05 2006")` give a real date but `t.Format("Mon Jan 1 15:04:05 2006")` gives the literal "1"? Read the layout rules.
3. Show that `time.Since(t)` returns the right elapsed even after `t.Round(0)` (which strips monotonic).
4. Build a `Ticker`-based worker that exits cleanly on context cancel.
5. Use `testing/synctest` (1.25+) to test a 1-hour deadline without waiting an hour.
