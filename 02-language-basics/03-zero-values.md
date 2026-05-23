# Zero Values — Why Go Has No `null`-Style Undefined State

## TL;DR

Every type in Go has a **zero value** — a well-defined initial state that the runtime assigns when you declare a variable without initialising it. Numerics are `0`, strings are `""`, booleans are `false`, pointers/interfaces/channels/maps/slices/functions are `nil`. There is no equivalent of JavaScript's `undefined`, no Python's `None`-vs-missing-key distinction, no Java's `NullPointerException`-on-an-uninitialized-primitive. The language guarantees that *every read of a declared variable produces a typed value*, never a runtime trap from "this variable hasn't been assigned." This eliminates an entire class of bugs and is the foundation for many idiomatic Go patterns: `sync.Mutex{}` is usable without `Lock`-time initialisation, `bytes.Buffer{}` is ready to `Write`, `http.Server{}` listens. The single biggest gotcha that newcomers hit: **nil maps cannot be written to** (only read; writes panic), and **nil slices CAN be appended to** (the append returns a new slice). Both are "nil" but they behave differently — by design. The other recurring trap: **`*T` and `T` are different types with different zero values**. `var x int` is `0`; `var x *int` is `nil`, and dereferencing it (`*x`) panics.

## Mental Model

```
   Every Go type T has a single, well-defined zero value:

   bool         false
   int*, uint*, float*, complex*    0
   string       ""  (empty, but addressable)
   pointer (*T) nil
   slice ([]T)  nil  (length 0, cap 0, ptr nil — but usable!)
   map (map[K]V)nil  (writes panic, reads return zero value)
   channel (chan T) nil  (sends/recvs block forever)
   function (func(...)...) nil  (call panics)
   interface    nil  (both the type and the value are nil)
   struct       struct with each field at its own zero value
   array [N]T   array of N zero values
```

Two principles to internalise:

1. **`var x T` is always safe.** It produces a typed, readable value.
2. **`nil` is a typed concept.** `var p *int = nil` and `var m map[string]int = nil` and `var f func() = nil` are all "nil," but they're nil of different types and behave differently.

## Why No `undefined`?

Languages like JavaScript distinguish three states:
- Variable not declared (ReferenceError).
- Variable declared, not assigned (`undefined`).
- Variable assigned but to a "nothing" value (`null`).

Python similarly has uninitialized-attribute-on-an-object, missing-dictionary-key, and `None`-the-value.

Go has one state: declared = has a zero value. There is no "not yet assigned" or "undefined." This means:

- **No `NullPointerException` from forgotten initialization** for value types.
- **Refactoring is safer**: adding a field to a struct doesn't break existing code; the new field just has its zero value.
- **APIs can use the zero value as "default"** without sentinel parameters: `bytes.Buffer{}` is ready; `time.Time{}` is the zero instant; `sync.Mutex{}` is unlocked.

This works because the **Go designers picked types whose zero values are useful** — and explicitly recommend you design your own types with usable zero values.

## Zero Values by Type

### Numeric & Boolean

```go
var i int        // 0
var f float64    // 0
var c complex128 // 0+0i
var b bool       // false
```

### String

```go
var s string     // "" (empty, length 0, but a valid string)

len(s)   // 0
s == ""  // true
s + "x"  // "x" — concatenation works
s[0]     // panic: index out of range
```

The empty string is a usable value — printable, concatenable, length-checkable. Indexing it panics; that's not "nilness," it's just out-of-range.

### Pointer

```go
var p *int       // nil
p == nil         // true
*p               // panic: nil pointer dereference

p = new(int)     // allocates a zero int, returns *int
*p               // 0 — the int's zero value
```

`new(T)` always returns a non-nil `*T` pointing at a zeroed `T`. `&T{}` does the same for composite literals.

### Slice

```go
var s []int      // nil
s == nil         // true
len(s)           // 0
cap(s)           // 0
s = append(s, 1) // OK! returns []int{1}
```

A nil slice is **functionally equivalent to an empty slice** for almost everything: `len`, `range`, `append`. The only difference: `s == nil` distinguishes them.

```go
empty := []int{}    // not nil — points at a zero-length array
nilSlice := []int(nil)

len(empty) == len(nilSlice)   // both 0
empty == nil      // false
nilSlice == nil   // true
```

In practice: prefer `var s []int` (nil) over `s := []int{}` unless a downstream API distinguishes (rare — `encoding/json` does for some empty-vs-null cases).

### Map

```go
var m map[string]int    // nil
m["x"]                  // 0 (the int zero value) — READ is fine
m["x"] = 1              // PANIC: assignment to entry in nil map
```

**Reads from a nil map are valid; writes are not.** This is asymmetric and surprising:

```go
m := map[string][]int(nil)   // nil map
v := m["x"]                  // OK — v is nil slice
m["x"] = append(v, 1)        // panic!
```

The fix is to initialise:

```go
m := map[string][]int{}      // or make(map[string][]int)
m["x"] = append(m["x"], 1)   // OK
```

Initialisation is one of three forms:

```go
m := make(map[string]int)        // empty map
m := make(map[string]int, 100)   // empty map, pre-sized for ~100 entries
m := map[string]int{}            // composite literal, same as make
m := map[string]int{"a": 1}      // initialised with values
```

### Channel

```go
var c chan int    // nil
c <- 1            // blocks FOREVER (deadlock if it's the only goroutine)
<-c               // blocks FOREVER
close(c)          // panic
```

A nil channel never proceeds. Useful as a "disable this case" idiom in a `select`:

```go
var done chan struct{}   // nil — never selectable
if shouldEnableDone {
    done = make(chan struct{})
}
select {
case <-done: ...   // never picked if done is nil
case msg := <-other: ...
}
```

### Interface

```go
var i interface{}     // nil
i == nil              // true

var p *int            // nil pointer
i = p                 // i is NOT nil! — see below
i == nil              // FALSE
```

This is **the most famous Go gotcha**. An interface is "nil" only if **both the dynamic type and the dynamic value are nil**. A nil pointer of a concrete type wrapped in an interface is "not nil" because the type is set.

```go
func myFunc() error {
    var e *MyError = nil
    return e          // returns a non-nil error!
}

err := myFunc()
if err != nil {
    // entered — even though e was nil
    err.Error()       // PANIC — calling method on nil pointer
}
```

The fix: return `nil` explicitly when you mean nil:

```go
func myFunc() error {
    var e *MyError = nil
    if some_condition {
        e = &MyError{...}
    }
    if e == nil {
        return nil
    }
    return e
}
```

### Function

```go
var f func(int) int   // nil
f(5)                  // panic: nil function call

f = func(x int) int { return x * 2 }
f(5)                  // 10
```

### Struct

```go
type User struct {
    ID    int
    Name  string
    Tags  []string
    Roles map[string]bool
}

var u User
// u.ID = 0, u.Name = "", u.Tags = nil, u.Roles = nil

len(u.Tags)   // 0 — OK
u.Tags = append(u.Tags, "admin")  // OK
u.Roles["x"] = true   // PANIC — nil map
```

Each field gets its own zero value, recursively. The struct value itself is *not nil* — only pointers, slices, maps, channels, interfaces, and functions can be nil. A struct is always "there"; it just may have nil fields.

### Array

```go
var arr [5]int    // [0, 0, 0, 0, 0]
var arr2 [3]string  // ["", "", ""]
var arr3 [2]User    // two zero-Users
```

Arrays are value types; the array itself is fully initialized to zero values. Length is part of the type.

## "Useful Zero Value" — A Design Discipline

Standard library types are designed so their zero value is immediately useful:

| Type | Zero-value behaviour |
|------|----------------------|
| `bytes.Buffer{}` | Empty buffer, ready to `Write`/`Read` |
| `strings.Builder{}` | Empty builder, ready to `WriteString` |
| `sync.Mutex{}` | Unlocked, ready to `Lock` |
| `sync.WaitGroup{}` | Counter at 0 |
| `sync.Once{}` | Action hasn't run |
| `sync.Map{}` | Empty map |
| `http.Server{}` | Ready to `ListenAndServe` (with defaults) |
| `time.Time{}` | The zero time (year 1, month 1, day 1) |
| `log.Logger{}` | (less useful — usually you want `log.New`) |

When designing your own types, ask: **is the zero value usable?** If yes, no constructor needed. If no, write a `NewT(...)`.

```go
// Good — zero value usable
type Counter struct {
    n int64
}
func (c *Counter) Inc() { atomic.AddInt64(&c.n, 1) }
func (c *Counter) Get() int64 { return atomic.LoadInt64(&c.n) }

var ctr Counter   // usable immediately

// Less good — requires constructor
type Pool struct {
    workers int
    queue   chan job
    started bool
}
// must call NewPool(workers) — zero value would deadlock
```

## Detecting "Not Set"

Sometimes you genuinely need to distinguish "explicitly zero" from "not provided." Three idioms:

### 1. Pointer wrapper

```go
type Config struct {
    Timeout *time.Duration  // nil = not set; non-nil = set
}

if cfg.Timeout != nil {
    use(*cfg.Timeout)
} else {
    use(defaultTimeout)
}
```

Verbose but explicit.

### 2. Sentinel value

```go
type Config struct {
    Timeout time.Duration  // 0 = default
}

dur := cfg.Timeout
if dur == 0 {
    dur = defaultTimeout
}
```

Cleanest if zero is genuinely "not a valid value" for the domain (a 0-second timeout is meaningless).

### 3. "Optional" type

```go
type Optional[T any] struct {
    Value T
    Set   bool
}
```

Or use `database/sql.NullString`-style:

```go
type NullString struct {
    String string
    Valid  bool
}
```

Useful for SQL fields where NULL and empty string differ.

### 4. Map presence (`comma-ok`)

```go
m := map[string]int{"a": 0, "b": 5}

v, ok := m["a"]    // v=0, ok=true — present, value happens to be 0
v, ok = m["c"]     // v=0, ok=false — not present
```

The comma-ok idiom distinguishes "zero value because not set" from "zero value because that's what was stored."

## Memory Allocation

When you write `var x T`, the compiler allocates space for one `T` on the stack (or heap, if escape analysis decides) and zeroes it. The zeroing is done by:

- **Small types** (≤ 32 bytes): inlined stores.
- **Larger types**: `runtime.memclr` (uses `REP STOSB` or SIMD on x86).
- **Heap allocations** via `new(T)` or composite literals that escape: malloc + memclr.

The zero-initialization cost is built into allocation. There's no separate "skip zeroing for performance" mode in safe Go code (some `unsafe`+`reflect` tricks bypass it; see `13-unsafe`).

## Anti-Patterns & Gotchas

**Writing to a nil map.** Panics. Always `make` or `{}` before writing.

**Returning a `nil` typed pointer as `error`.** The interface is non-nil even when the underlying pointer is nil. Always `return nil` explicitly.

**Treating `nil` slice and `[]T{}` differently** in JSON. They serialise differently:
```go
var nilSlice []int      // marshals to "null"
emptySlice := []int{}   // marshals to "[]"
```
Pick one consistently per API.

**Forgetting that struct fields zero-init.** A `time.Time{}` is a valid time (year 1, January 1) — `.IsZero()` is the safe check, not `== time.Time{}`.

**`var x *Mutex; x.Lock()`** — nil pointer dereference. You meant `var x sync.Mutex` (value type, not pointer).

**Initializing every field "for clarity"** — defeats the readability of zero-value design. `User{}` is fine; `User{ID: 0, Name: "", Tags: nil}` is noise.

**Comparing structs with nil-able fields directly.** `User{} == User{}` is true; `User{Tags: nil} == User{Tags: []string{}}` is a compile error (slices aren't comparable).

**Using `make(map[string]int, 0)`.** Same as `make(map[string]int)` — the 0 is meaningless. Omit it.

**Allocating large slices with `make([]T, N)` when you don't need the zero values** (you'll immediately overwrite them). The zeroing is wasted work; use `slices.Grow` + index assignment if profiling reveals it's hot. Or use `unsafe` (last resort).

**Forgetting that `time.Time{}` is comparable but `time.Time{}.IsZero()` is the idiomatic check.** `t == time.Time{}` works but doesn't read as clearly.

**`var ch chan int; ch <- 1`** — sends to nil channel block forever. Initialize with `make(chan int)`.

**Calling a method on a nil-pointer receiver expecting it to work** — sometimes it does (the method handles nil), sometimes it panics. Document explicitly.

**Confusing `nil` interface with `(*T)(nil)` interface.** They print and compare differently.

## Performance Notes

- **Zero-initialisation cost** for small types is essentially free (a single store).
- **`make([]T, N)`** zeros N entries — O(N).
- **`new(T)`** zeros one T — O(sizeof(T)).
- **Composite literals** (`T{}`) that escape to heap zero on allocation; on stack they zero-initialize.
- **`clear(map)`** (1.21+) is faster than re-allocating an empty map if you want to retain capacity.

## How Big Companies Use It

- **Google's internal style guide** emphasises designing types with useful zero values — explicit in their public Go style guide.
- **Kubernetes** uses `metav1.ObjectMeta{}` zero-value extensively; many APIs accept partially-populated objects with defaults applied.
- **HashiCorp** Vault and Consul rely on `sync.Mutex{}` and `sync.Once{}` zero values throughout.
- **CockroachDB** uses zero-value `roachpb.Timestamp` to mean "now or unspecified."
- **Tailscale**'s code style emphasises avoiding pointers where possible; struct zero values are preferred to nil-checked pointers.
- **Cloudflare**'s wire-format packet parsers rely on zero-valued buffers being safe to write into.

The recurring pattern: **constructors are a smell unless zero value is unusable.** New Go codebases tend to have fewer `New*` functions than other languages' equivalents.

## Source Code References

- Go spec — The zero value: https://go.dev/ref/spec#The_zero_value.
- `runtime.memclr`: https://github.com/golang/go/blob/master/src/runtime/memclr_amd64.s.
- `sync.Mutex` zero-value design: https://github.com/golang/go/blob/master/src/sync/mutex.go.
- `bytes.Buffer` zero-value design: https://github.com/golang/go/blob/master/src/bytes/buffer.go.
- Effective Go — The zero value: https://go.dev/doc/effective_go#composite_literals.

## Further Reading

- "Constructors and composite literals" in Effective Go: https://go.dev/doc/effective_go#composite_literals.
- "Make the zero value useful" — Go proverb (Rob Pike).
- Dave Cheney, "Avoid package level state in Go": https://dave.cheney.net/2017/06/11/go-without-package-scoped-variables.
- Go FAQ — "When are function parameters passed by value?": https://go.dev/doc/faq#pass_by_value.
- "Why nil and empty maps are different": various blog posts.
- "The Go Programming Language" (Donovan & Kernighan), chapter on types.

## Exercises / Self-Check

1. Write to a nil map; observe the panic. Initialise with `make` and re-run.
2. Read from a nil map; observe you get the zero value (not a panic).
3. Append to a nil slice; confirm the result is a valid slice of length 1.
4. Send to a nil channel; confirm it blocks (the program will deadlock).
5. Return a `(*MyError)(nil)` as `error`. Test `if err != nil` — observe it's true. Implement the safe pattern.
6. Marshal a nil slice vs empty slice to JSON. Observe `null` vs `[]`.
7. Implement a `Counter` type with a usable zero value. Use it without a constructor.
8. Use `clear(m)` on a map you wish to reuse. Compare to `m = map[string]int{}`. Note the difference in capacity retention.
