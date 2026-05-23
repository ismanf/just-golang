# Interfaces

## TL;DR

An interface in Go is a **set of method signatures**. A type satisfies an interface automatically — no `implements` keyword — by having all the required methods. At runtime, an interface value is a **2-word fat pointer**: a type descriptor + a data pointer. This boxing is the source of most interface-related performance discussions. Keep interfaces small. Accept interfaces, return concrete types.

## Mental Model

```
interface value (16 bytes on 64-bit):
+----------+---------+
|   *type  |  *data  |
+----------+---------+
     │           │
     │           └─► the actual value (or a pointer to it if it doesn't fit a word)
     │
     └─► itab: methods + type info; resolved once and cached
```

Two flavors of interface header in the runtime:

- `iface` — for any interface with at least one method (most). Holds an `itab`.
- `eface` — for the empty interface (`any`). Holds a raw `*_type`.

Conceptually identical from your code's perspective; the runtime treats them separately for speed.

## Syntax & Basic Usage

```go
package main

import "fmt"

type Greeter interface {
	Greet() string
}

type Spanish struct{ Name string }
func (s Spanish) Greet() string { return "Hola, " + s.Name }

type Klingon struct{ Name string }
func (k Klingon) Greet() string { return "nuqneH, " + k.Name }

func main() {
	var g Greeter
	g = Spanish{"Maria"}
	fmt.Println(g.Greet())
	g = Klingon{"Worf"}
	fmt.Println(g.Greet())
	// Output:
	// Hola, Maria
	// nuqneH, Worf
}
```

There is no `implements` keyword. `Spanish` satisfies `Greeter` because it has a method with the right signature.

## Deep Dive

### Structural (duck) typing

```go
type Stringer interface{ String() string }

// time.Time, net.IP, big.Int, errors.errorString, your custom Money type — all
// satisfy Stringer without "knowing" about it.
```

Concrete types are decoupled from the interfaces they satisfy. This is the single most important design property of Go's type system: you can write an interface *after* the types that satisfy it exist.

### The two-word header (`iface`)

```go
// Simplified runtime layout from src/runtime/runtime2.go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32   // copy of _type.hash, for type switches
	_     [4]byte
	fun   [1]uintptr // first method pointer; variable-sized in practice
}
```

When you write `var g Greeter = someValue`, the compiler:

1. Looks up or creates the `itab` for `(Greeter, typeof(someValue))`.
2. Boxes the value: if it fits in a word and isn't a pointer, copies it into `data` directly; otherwise allocates and stores a pointer.
3. Stores the `itab` pointer and the data pointer in the interface header.

Calls through `g.Greet()` look up the function pointer in `itab.fun`. The lookup itself is a fixed offset — no dynamic search per call.

### `nil` interface vs typed nil

```go
var p *MyErr        // nil pointer of concrete type
var e error = p     // interface value: tab != nil, data == nil
fmt.Println(e == nil) // false — the interface is NOT nil
```

`e == nil` only when **both** the type and data pointers are nil. This is the most-stumbled-on trap in Go. Cure: never return a typed nil where the return type is an interface. Return `nil` explicitly.

```go
func find() error {
	var p *MyErr
	if conditionFailed() {
		return p   // BUG: caller's err != nil
	}
	return nil    // correct
}
```

### Interface satisfaction: value vs pointer receivers

If your interface methods are declared with **pointer** receivers, the method set of `*T` satisfies the interface but the method set of `T` does not.

```go
type Stamper interface{ Stamp() }
type S struct{}
func (s *S) Stamp() {}

var _ Stamper = &S{}   // OK
var _ Stamper = S{}    // compile error
```

Practical impact: when passing values to functions taking interfaces, prefer pointers if any methods need pointer receivers.

### Small interfaces

Idiomatic Go interfaces tend to be small: `io.Reader`, `io.Writer`, `io.Closer`, `fmt.Stringer`, `sort.Interface`, `error`. The standard library has more 1-method interfaces than all of Java's standard library. Small interfaces compose:

```go
type ReadWriter interface {
	io.Reader
	io.Writer
}
```

Embed; don't enumerate.

### Accept interfaces, return concrete types

```go
// Good
func Process(r io.Reader) (*Report, error) { /* ... */ }

// Less good
func Process(r *os.File) (Reporter, error) { /* ... */ }
```

Accepting interfaces makes your function testable and composable. Returning concrete types makes their API discoverable and avoids leaky abstractions. Exceptions exist (e.g., `os.Open` returns `*File` which is correct; some factories must return interfaces). The principle is a default, not a law.

### Interface comparison

Two interface values are equal if both their dynamic types and dynamic values are equal:

```go
var a, b error = io.EOF, io.EOF
a == b // true — same type, same value
```

If the dynamic type isn't comparable (e.g., contains a slice), comparing panics at runtime. `reflect.DeepEqual` doesn't panic but is slow and frequently wrong for what you actually wanted. Prefer `errors.Is`/`errors.As` for errors.

### Interface assertion under the hood

```go
v, ok := iface.(Concrete)
```

The runtime checks if `iface.tab._type == typeof(Concrete)` (one pointer compare) and unboxes. Failure with the comma-ok form yields the zero value and `ok=false`. Failure without comma-ok panics.

### Empty interface (`any`)

`any` is an alias for `interface{}` introduced in 1.18. It uses the `eface` runtime layout (no methods → no itab needed; just the type descriptor and the data pointer). Treat it as "any type, dynamically." See the dedicated page for cost analysis.

### Interfaces and type sets (since 1.18)

Generics generalized interfaces to also describe **type sets**:

```go
type Signed interface{ ~int | ~int8 | ~int16 | ~int32 | ~int64 }
```

Used as **constraints** in generics, not as values. Distinguish between traditional method-set interfaces (usable as values) and type-set-only interfaces (constraint-only). The compiler rejects using a type-set-only interface as a value type.

## Standard Library Hooks

- `io.Reader`, `io.Writer`, `io.Closer`, `io.ReaderAt`, `io.Seeker` — the foundational stream interfaces.
- `error`, `fmt.Stringer`, `fmt.GoStringer`, `fmt.Formatter` — formatting.
- `sort.Interface` — historical.
- `http.Handler`, `http.RoundTripper` — server/client extension points.
- `json.Marshaler`, `json.Unmarshaler`, `encoding.TextMarshaler` — encoding hooks.
- `flag.Value`, `flag.Getter` — CLI flags.
- `context.Context` — the most copied interface in modern Go.
- `slog.Handler`, `slog.LogValuer` — structured logging.
- `reflect.Type`, `reflect.Value` — runtime type introspection.

## Real-World Patterns

### 1. Dependency injection via interface parameter

```go
type Repository interface {
	Get(ctx context.Context, id string) (*Item, error)
}

type Service struct{ repo Repository }
func (s *Service) Handle(ctx context.Context, id string) error {
	item, err := s.repo.Get(ctx, id)
	if err != nil { return err }
	_ = item
	return nil
}
```

Tests substitute a mock `Repository` without touching the production database.

### 2. Sealing an interface to a package

```go
type Event interface {
	event() // unexported method — only this package can implement
}

type Login struct{}  ; func (Login) event()  {}
type Logout struct{} ; func (Logout) event() {}
```

Callers can hold `Event` values but cannot define new variants. Useful for tagged unions and exhaustive `switch`es.

### 3. Optional capability detection

```go
type Closer interface{ Close() error }

func tryClose(v any) {
	if c, ok := v.(Closer); ok {
		_ = c.Close()
	}
}
```

`net/http`'s `ResponseWriter` famously uses this for `Flusher`, `Hijacker`, `Pusher`. Avoid for new APIs — it's brittle — but it's everywhere in the stdlib.

### 4. Decorator / middleware

```go
type Handler interface {
	ServeHTTP(http.ResponseWriter, *http.Request)
}

func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s %s", r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

`http.HandlerFunc` is itself a one-line trick: a function type that implements `Handler` via a method on the function type. Read it to internalize the idiom.

### 5. Replaceable global via interface

```go
type Clock interface{ Now() time.Time }

type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }

var DefaultClock Clock = RealClock{}

// In tests:
DefaultClock = fakeClock{at: time.Date(...)}
```

Avoid the global if you can — pass `Clock` as a dependency — but when a singleton is required, this swap-in pattern is clean.

## Anti-Patterns & Gotchas

**Typed nil interface.** Already covered. Single biggest interface footgun.

**Defining interfaces before you need them.** Don't write an interface for every concrete type "just in case." Wait until you have at least two real implementations or a real testing need. This is the "accept interfaces, return concrete types" guidance flipped: don't *export* interfaces speculatively.

**Big god interfaces.** `Service` with 30 methods is hard to mock and hard to satisfy. Split.

**Interface "method bag" for unrelated capabilities.** A single interface holding `Read`, `Save`, `Validate`, `Render`, `Notify` ties all callers to all implementations. Many small interfaces compose better.

**Calling unimplemented embedded interface methods.** `type Foo struct{ io.Reader }` — calling `Foo.Read` when the inner `Reader` is nil panics. Fine for partial mocks, bad in production.

**`switch v.(type)` cascades** that grow without bound. If you're adding a new type and editing 14 switch statements, your interface should have grown a method instead.

**Using `any` as the primary parameter type for "flexibility".** You've thrown away the type system. Use generics or a sealed interface.

**Calling a pointer-receiver method through an interface holding a value.** Won't satisfy. Compile-time error if you try `var _ Iface = T{}`.

## Performance Notes

- Interface call cost ≈ one indirect call (function pointer lookup in `itab` + jump). Typically 2-3× slower than a direct call when not inlined, but invisible if the work inside the method is more than a few instructions.
- Boxing a value into an interface may allocate. The compiler avoids the alloc if the value fits in a word and contains no pointers (e.g., `int`, `bool`, small enum). For larger values, it allocates on the heap.
- `itab`s are cached globally; lookup for `(InterfaceType, ConcreteType)` happens once.
- Type assertions are fast: pointer compare on `itab` or `_type`. Type switches compile to a series of compares, often jump-tabled.
- `reflect`-based interface inspection is **much** slower than direct calls. Don't use `reflect` in hot loops.
- Embedding an interface in a struct adds a 2-word field; copying the struct copies the header (cheap).

## How Big Companies Use It

- **Kubernetes `runtime.Object`** and the `metav1.Object` interfaces let one set of generic controllers and reconcilers operate over every API resource. Read the `controller-runtime` library for the canonical patterns: https://github.com/kubernetes-sigs/controller-runtime.
- **HashiCorp Vault** uses `Backend` and `Plugin` interfaces to let third-party storage and auth providers slot in without recompiling Vault.
- **Caddy** uses module interfaces (`caddy.Module`) so every plugin (TLS provider, HTTP handler, log writer) registers via the same one-method registration interface.
- **Cockroach `kv.Sender`** is a single-method interface — `Send(ctx, BatchRequest) (BatchResponse, *Error)` — and the entire stack of distributed-SQL request handling is composed of these.
- **Standard library `io.Reader`/`Writer`** drive composability for compression, encryption, hashing, buffering, network. The 1-method `io.Reader` is arguably Go's most successful API.

## Source Code References

Pinned to `go1.26`.

- Interface runtime layout: [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/master/src/runtime/runtime2.go) — search `type iface`, `type eface`, `type itab`.
- `itab` lookup and caching: [`src/runtime/iface.go`](https://github.com/golang/go/blob/master/src/runtime/iface.go) — functions `getitab`, `convT2I*`, `assertI2I*`.
- Type assertion fast paths (compiler): [`src/cmd/compile/internal/walk/assign.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/assign.go) and `convert.go`.
- `http.HandlerFunc` adapter: [`src/net/http/server.go`](https://github.com/golang/go/blob/master/src/net/http/server.go).
- `io.Reader` and friends: [`src/io/io.go`](https://github.com/golang/go/blob/master/src/io/io.go).
- Famous typed-nil example in the Go FAQ: https://go.dev/doc/faq#nil_error.

## Further Reading

- Spec, "Interface types": https://go.dev/ref/spec#Interface_types
- Go blog, "Russ Cox: Go Data Structures: Interfaces": https://research.swtch.com/interfaces
- Effective Go on interfaces: https://go.dev/doc/effective_go#interfaces
- Go FAQ, "Why does my nil error value not equal nil?": https://go.dev/doc/faq#nil_error
- Dave Cheney, "The Zen of Go": https://dave.cheney.net/2020/02/23/the-zen-of-go (interface design principles)
- "Accept interfaces, return structs": https://go.dev/wiki/CodeReviewComments#interfaces

## Exercises / Self-Check

1. Implement `io.Reader` for an iterator that yields the bytes of a `*strings.Reader` but truncated to the first `N` bytes. Test with `io.ReadAll`.
2. Write a function `IsCloser(v any) bool` that returns whether `v` satisfies `io.Closer`. Why must you use a type assertion and not a type comparison?
3. Why does `var x error = (*MyErr)(nil); x != nil` print `true`? Trace through the `iface` representation.
4. Build a sealed-interface tagged union with three variants. Show that another package cannot add a new variant.
5. Benchmark a call through `io.Writer` interface vs a direct call to `*bytes.Buffer.Write`. Where does the cost come from?
