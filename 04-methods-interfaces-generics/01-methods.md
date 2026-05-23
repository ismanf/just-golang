# Methods

## TL;DR

A method is a function with a **receiver** — a special parameter declared between `func` and the method name. Receivers can be value (`T`) or pointer (`*T`); they determine the **method set** of the type, which determines which interfaces the type satisfies. Use a pointer receiver when the method mutates, when the type is large, when the type contains a `sync.*` field, or when consistency across the type's methods demands it.

## Mental Model

```
type Counter struct{ n int }

func (c  Counter) Read()  int   { return c.n }   // receiver is a COPY
func (c *Counter) Inc()         { c.n++ }        // receiver is a POINTER

Method set of Counter:   { Read }                // value-receiver methods only
Method set of *Counter:  { Read, Inc }           // both

So:
  var v Counter           v.Inc() — OK; Go auto-takes &v (addressable)
  var p *Counter = &v     p.Inc() — OK
  Counter{}.Inc()         — compile error: Counter literal is NOT addressable
```

The receiver is just the first parameter dressed up with a position. The trick is that Go also uses the receiver shape to compute method sets.

## Syntax & Basic Usage

```go
package main

import "fmt"

type Rect struct{ W, H float64 }

// Value receiver — pure function over a copy.
func (r Rect) Area() float64 { return r.W * r.H }

// Pointer receiver — mutates the original.
func (r *Rect) Scale(k float64) {
	r.W *= k
	r.H *= k
}

func main() {
	r := Rect{W: 3, H: 4}
	fmt.Println(r.Area()) // 12

	r.Scale(2) // auto: (&r).Scale(2)
	fmt.Println(r.Area()) // 48
	// Output:
	// 12
	// 48
}
```

You can define a method on any **named type** declared in the same package as the method — not just structs.

```go
type Celsius float64
func (c Celsius) Fahrenheit() float64 { return float64(c)*9/5 + 32 }

type IDs []int
func (ids IDs) Contains(x int) bool {
	for _, v := range ids { if v == x { return true } }
	return false
}
```

You cannot define methods on types from other packages (e.g., you can't put a method on `time.Time`). The workaround is a wrapper type.

## Deep Dive

### Method sets

Spec ([Method sets](https://go.dev/ref/spec#Method_sets)):

- The method set of type `T` consists of all methods declared with receiver type `T`.
- The method set of pointer type `*T` consists of all methods declared with receiver `T` **and** `*T`.

So a `*T` has *more* methods than a `T`. This is the rule that decides interface satisfaction.

```go
type Mutator interface{ Mutate() }
type V struct{}
func (v *V) Mutate() {} // pointer receiver only

var _ Mutator = &V{}    // OK
var _ Mutator = V{}     // compile error: V does not satisfy Mutator
                        // (Mutate has pointer receiver)
```

### Auto-addressing

When you call `v.PointerMethod()` on a value `v`, Go silently rewrites it as `(&v).PointerMethod()` — **but only if `v` is addressable**. Map values and function returns are not addressable:

```go
type M struct{ n int }
func (m *M) Inc() { m.n++ }

m := M{}
m.Inc()              // OK

mm := map[string]M{"a": {}}
// mm["a"].Inc()      // compile error: cannot take address of mm["a"]
```

Fix: use `map[string]*M`, or read-modify-write.

### Pointer vs value: how to choose

Authoritative checklist (Code Review Comments):

1. If the method needs to mutate the receiver, use `*T`.
2. If the receiver contains a `sync.Mutex` or other field that **must not be copied**, use `*T` and forbid copying via `go vet`.
3. If the receiver is a large struct or array, `*T` avoids the per-call copy.
4. If the receiver is a small value type whose copy is cheap (e.g., `time.Time`, `Point{X, Y int}`), value receivers are fine — and they let callers use the value form too.
5. **Be consistent.** If some methods on `T` use pointer receivers, give the rest pointer receivers too. Mixing leads to confusion about method sets.
6. If you'd otherwise pass `*T` everywhere, just use pointer receivers.

A common rule of thumb: when in doubt, pointer receivers. The exception is small immutable types where value semantics make the API friendlier (`big.Int` — actually uses pointer receivers; `netip.Addr` — uses value receivers because the struct is small and immutable by design).

### Methods on type aliases

A type alias (`type B = A`) shares its method set with the original — no new methods may be declared on the alias. A **defined type** (`type B A`) starts with an empty method set and you can add methods to it.

```go
type Bytes = []byte           // alias; can't add methods
type ByteSlice []byte         // new type; can add methods
func (b ByteSlice) Reverse() { /* ... */ }
```

### Methods can be values and expressions

```go
r := Rect{W: 3, H: 4}

areaMethod := r.Area     // method value: bound to r
fmt.Println(areaMethod()) // 12

areaExpr := Rect.Area    // method expression: takes receiver explicitly
fmt.Println(areaExpr(r)) // 12
```

Method values close over the receiver (so they may cause it to escape). Method expressions are explicit functions of `(receiver, args...) → result`.

### `MarshalJSON`, `String`, and friends

The standard library uses methods as "magic": if your type implements `String() string`, `fmt.Println` calls it; if it implements `MarshalJSON() ([]byte, error)`, `json.Marshal` calls it; same for `error`, `Stringer`, `time.Marshaler`, `flag.Value`. Pay attention to receiver type — `String()` on `*T` means a `T` won't be String-formatted when printed by value.

### Methods on generic types

Type parameters on methods? **No** — you cannot add extra type parameters on methods. Methods inherit the type's parameters but cannot introduce their own.

```go
type Stack[T any] struct{ s []T }
func (st *Stack[T]) Push(v T) { st.s = append(st.s, v) }
// func (st *Stack[T]) Map[U any](f func(T) U) []U { ... } // INVALID
```

Workaround: package-level generic functions, or interface-level abstraction.

## Standard Library Hooks

- `fmt.Stringer` — any type with `String() string` formats via `%v`/`%s`/`Println`.
- `error` — any type with `Error() string` satisfies `error`.
- `json.Marshaler` / `json.Unmarshaler` — custom JSON.
- `encoding.TextMarshaler` / `TextUnmarshaler` — text encoding hook (used by JSON for map keys, `time.Time`, etc.).
- `sort.Interface` — three methods (`Len`, `Less`, `Swap`); largely superseded by `slices.SortFunc`.
- `flag.Value` — for custom command-line flag types.
- `io.Reader` / `io.Writer` — interfaces defined entirely by one method.
- `http.Handler` — `ServeHTTP(w, r)` is the canonical "method as plugin" interface.

## Real-World Patterns

### 1. Builder methods returning the receiver

```go
type Query struct {
	table   string
	filters []string
}

func (q *Query) From(t string) *Query  { q.table = t; return q }
func (q *Query) Where(f string) *Query { q.filters = append(q.filters, f); return q }

q := (&Query{}).From("users").Where("age > 18")
```

### 2. Satisfying `error` with a typed error

```go
type NotFoundError struct{ ID string }
func (e *NotFoundError) Error() string { return "not found: " + e.ID }

func lookup(id string) error {
	return &NotFoundError{ID: id}
}

var nfe *NotFoundError
if errors.As(err, &nfe) {
	// nfe.ID available
}
```

Pointer receiver here so that two `&NotFoundError{ID:"x"}` values are not equal by `==` — typically what you want for errors.

### 3. Implementing `http.Handler`

```go
type echoHandler struct{ logger *slog.Logger }

func (h *echoHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	h.logger.Info("hit", "path", r.URL.Path)
	io.Copy(w, r.Body)
}

http.Handle("/", &echoHandler{logger: slog.Default()})
```

### 4. Method values for callbacks

```go
type Worker struct{ id int }
func (w *Worker) Process(item string) { /* ... */ }

w := &Worker{id: 1}
go pump(w.Process)   // pass the bound method value
```

`w.Process` is a closure over `w`; the goroutine gets exactly what it needs.

### 5. `fmt.Stringer` for safe logging

```go
type APIKey string
func (k APIKey) String() string { return "***redacted***" }

var k APIKey = "sk_live_supersecret"
fmt.Println(k)              // ***redacted***
fmt.Println(string(k))      // sk_live_supersecret (explicit conversion bypasses)
```

Useful for redacting secrets in default-formatted log lines.

## Anti-Patterns & Gotchas

**Mixing value and pointer receivers on the same type.** Decide once. Mixing surprises readers and creates inconsistent method-set behavior.

**Calling a pointer method on an unaddressable value.** `mm[k].Mutate()` where `mm` is `map[K]V` doesn't compile. Use `*V` in the map, or rewrite the value.

**Defining `String()` that calls `fmt.Sprintf("%v", s)` recursively.** Infinite recursion — `%v` sees the `Stringer` interface and calls `String()` again.

**Putting expensive work in `String()`.** Every `fmt.Println(x)` triggers it. Same goes for `Error()`. Keep them cheap.

**Value receiver on a type with `sync.Mutex` field.** `go vet` flags. Each call copies the mutex, losing the lock semantics.

**Defining methods on collection aliases for types from other packages.** You can't add methods to `time.Duration`; you'd need a defined wrapper.

**Forgetting that the method set of `T` doesn't include `*T` methods.** `var _ MyIface = T{}` fails if `MyIface` needs a method with pointer receiver.

## Performance Notes

- Method calls compile to a direct function call when the static type is concrete — no v-table lookup. Indirect calls happen only through interfaces.
- Value receivers copy the receiver. For multi-word structs in hot paths, this adds up.
- Pointer receivers don't allocate by themselves; the receiver pointer is already passed via the caller. But if the call site has to take `&someLocal` and that escape-analyzes to the heap, you've allocated.
- Method values (`x.Method` stored in a variable) close over the receiver, often causing it to escape.
- Inlining works for both value and pointer methods. Use `-gcflags='-m=2'` to see what's inlined.

## How Big Companies Use It

- **Kubernetes' `runtime.Object`** is satisfied by every API type via a small pair of methods (`GetObjectKind`, `DeepCopyObject`). The runtime uses reflection over the method set to dispatch.
- **Docker's interfaces (`Driver`, `Plugin`)** all rely on method sets across pointer receivers — making mock objects trivially substitutable in tests.
- **Cockroach's `Replica`** has hundreds of methods, all on `*Replica`, all consistent — a real-world example of the "be consistent" rule.
- **`time.Time`** is the canonical example of a value-receiver-heavy small type with one notable pointer-receiver method (`*time.Time.UnmarshalJSON`).
- **The standard library's `big.Int`** uses pointer receivers everywhere — values are mutable, large, and reused. The result type is always the receiver.

## Source Code References

Pinned to `go1.26`.

- Method type info: [`src/internal/abi/type.go`](https://github.com/golang/go/blob/master/src/internal/abi/type.go) — `Method`, `Imethod`.
- Method set computation: [`src/go/types/methodset.go`](https://github.com/golang/go/blob/master/src/go/types/methodset.go).
- Auto-addressing rules: spec at [Method calls](https://go.dev/ref/spec#Method_expressions).
- `time.Time` methods (value-receiver showcase): [`src/time/time.go`](https://github.com/golang/go/blob/master/src/time/time.go).
- `big.Int` methods (pointer-receiver showcase): [`src/math/big/int.go`](https://github.com/golang/go/blob/master/src/math/big/int.go).
- `bytes.Buffer` methods (pointer receiver, mutable state): [`src/bytes/buffer.go`](https://github.com/golang/go/blob/master/src/bytes/buffer.go).

## Further Reading

- Spec, "Method declarations": https://go.dev/ref/spec#Method_declarations
- Spec, "Method sets": https://go.dev/ref/spec#Method_sets
- Code Review Comments, "Receiver Type": https://go.dev/wiki/CodeReviewComments#receiver-type
- Dave Cheney, "Should methods be declared on T or *T?": https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t
- Effective Go, "Pointers vs. Values": https://go.dev/doc/effective_go#pointers_vs_values
- "The Go Programming Language" by Donovan & Kernighan, Chapter 6 (Methods)

## Exercises / Self-Check

1. Write `Counter` with `Inc()` and `Value()`. Make `*Counter` satisfy a `Reader` interface (with `Value() int`). Why does `var r Reader = Counter{}` fail?
2. Implement `fmt.Stringer` for a `Money` type that prints "$1,234.56" with thousands separators.
3. Why is `mm["x"].PointerMethod()` a compile error when `mm` is `map[string]MyStruct`? Show two ways to fix it.
4. Define a method on a type alias of `[]int`. Why does it fail with `type IDs = []int` but work with `type IDs []int`?
5. Profile method-value invocation (`f := x.M; for { f() }`) vs direct call (`for { x.M() }`). Where does the difference come from?
