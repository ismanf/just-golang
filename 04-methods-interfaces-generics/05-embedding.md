# Embedding

## TL;DR

Embedding is Go's mechanism for **composition with method promotion**. Declaring an anonymous field of type `T` in a struct (or an interface inside another interface) makes `T`'s exported fields and methods accessible on the outer type as if they were declared there. It is **not** inheritance — there is no `super`, no polymorphism through the outer type, and embedded methods know only about the inner value. The single ambiguity rule: shallower wins; same depth = compile error.

## Mental Model

```
type Engine struct { HP int }
func (e *Engine) Start() {}

type Car struct {
    Engine            // embedded — anonymous field
    Wheels int
}

c := Car{Engine: Engine{HP: 200}, Wheels: 4}
c.HP     == c.Engine.HP    // promotion: shorthand for c.Engine.HP
c.Start()                  // calls (&c.Engine).Start(); receiver is &c.Engine, NOT &c
```

Promotion is *syntactic sugar* over `c.Engine.X`. The runtime sees the inner field directly; the outer struct is invisible to embedded methods.

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"sync"
)

type Mutexed struct {
	sync.Mutex            // embedded — Mutexed.Lock(), Mutexed.Unlock() are promoted
	Counter int
}

func (m *Mutexed) Inc() {
	m.Lock()
	defer m.Unlock()
	m.Counter++
}

func main() {
	var m Mutexed
	m.Inc()
	m.Lock()
	m.Counter += 10
	m.Unlock()
	fmt.Println(m.Counter)
	// Output: 11
}
```

`sync.Mutex` is the canonical embedding target: zero-value-usable, no constructor needed, promotes `Lock`/`Unlock`.

## Deep Dive

### Anonymous fields

Embedding requires the field's name to be the type name. You can't rename:

```go
type Car struct {
	Engine     // OK
	// e Engine   // this is a named field, NOT embedding
}
```

The field name is `Engine`; `c.Engine` and `c.HP` both work.

### Promotion rules

- Exported and unexported fields/methods are both promoted.
- Method promotion respects receiver type: a `*T` method is promoted onto `*OuterType` only if the embedded field is `T` or `*T`. Specifically:
  - Embedding `T` makes value-receiver methods part of `OuterType`'s method set, and pointer-receiver methods part of `*OuterType`'s method set (provided the outer is addressable).
  - Embedding `*T` puts all of `T`'s methods (value and pointer) into both `OuterType` and `*OuterType`'s method sets.

### Ambiguity

```go
type A struct{ X int }
type B struct{ X int }

type C struct{ A; B }

var c C
// c.X  // compile error: ambiguous selector c.X
c.A.X  // OK
c.B.X  // OK
```

When two embedded fields offer the same name at the **same shallowest depth**, accessing without qualification is a compile error. If one is shallower than the other, the shallower wins silently — which is a feature, not a bug.

### Shadowing

You can deliberately shadow a promoted method:

```go
type Base struct{}
func (Base) Hello() string { return "base" }

type Derived struct{ Base }
func (Derived) Hello() string { return "derived" }

var d Derived
fmt.Println(d.Hello())      // derived
fmt.Println(d.Base.Hello()) // base
```

But there's no "virtual" dispatch — calls from `Base` methods do **not** reach `Derived.Hello`. There is no `super` mechanism. Embedded methods are bound to the inner value at compile time.

### Interface satisfaction via embedding

```go
type Reader interface{ Read([]byte) (int, error) }

type CountingReader struct {
	io.Reader  // embedded interface
	N int64
}

func (c *CountingReader) Read(p []byte) (int, error) {
	n, err := c.Reader.Read(p)
	c.N += int64(n)
	return n, err
}
```

Embedding `io.Reader` gives `CountingReader` a `Read` method "for free." The outer type overrides it to count bytes, then delegates to the inner. This is the canonical decorator pattern in Go.

### Embedded interface with nil inner

```go
type RO struct{ io.Reader }
var r RO        // RO.Reader is nil
// r.Read(buf)  // panic: nil pointer dereference
```

Useful for partial mocks (assign `io.Reader` only when you need that path) but a footgun in production code. Always check `nil` or always assign.

### Embedding `*T` vs `T`

```go
type Foo struct {
	Engine     // by value; copying Foo copies the Engine
}

type Bar struct {
	*Engine    // by pointer; copying Bar copies only the pointer; shared state
}
```

Both promote methods, but the semantics for the outer type differ. If `Engine` is large or stateful, use `*Engine`. If it's small and immutable, embed by value.

### Interface embedding interfaces

```go
type ReadWriter interface {
	io.Reader
	io.Writer
}
```

Define one interface as the union of others. The resulting type set is the intersection of capabilities (a type must satisfy *all* embedded interfaces).

### Embedding a generic type

```go
type Stack[T any] struct{ items []T }

type AuditedStack[T any] struct {
	Stack[T]   // embedded generic type
	OnPush func(T)
}
```

Works just like non-generic embedding. The outer methods may shadow inner ones; promotion follows the standard rules.

### `_` (blank) field embedding (since 1.25 `structs.HostLayout`)

```go
import "structs"

type Header struct {
	_ structs.HostLayout
	A, B uint32
}
```

Embedding `structs.HostLayout` is a marker for C-ABI layout. It doesn't promote anything; it's a compiler directive.

## Standard Library Hooks

- `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup` — frequently embedded for zero-value-usable locking.
- `io.Reader`, `io.Writer` and friends — embedded to build decorators (counting, hashing, limiting).
- `http.Handler` — embedded to extend handlers.
- `context.Context` — embedding it in your own context wrappers preserves `Deadline`, `Done`, `Err`, `Value` for free.
- `bytes.Buffer` — embed it to inherit `Read`/`Write` while adding fields.
- `time.Time` — embedded in custom types to preserve all formatting and arithmetic methods (rare; usually you store, not embed).

## Real-World Patterns

### 1. Lockable types via embedded mutex

```go
type Counter struct {
	sync.Mutex
	n int
}

func (c *Counter) Inc()    { c.Lock(); c.n++; c.Unlock() }
func (c *Counter) Value() int {
	c.Lock()
	defer c.Unlock()
	return c.n
}
```

Tip: do **not** export the embedded mutex if you don't want callers to lock from outside. Use a named, unexported field instead:

```go
type Counter struct {
	mu sync.Mutex  // unexported; lock is internal
	n  int
}
```

Embedding for `sync.Mutex` is convenient but leaks `Lock`/`Unlock` to API consumers. The Go team's own style guide is split on this; modern code increasingly prefers the named-field form.

### 2. Decorator pattern

```go
type CountingReader struct {
	io.Reader
	N int64
}

func (c *CountingReader) Read(p []byte) (int, error) {
	n, err := c.Reader.Read(p)
	c.N += int64(n)
	return n, err
}

func main() {
	r := &CountingReader{Reader: strings.NewReader("hello world")}
	io.Copy(io.Discard, r)
	fmt.Println(r.N) // 11
}
```

This is the pattern used by `gzip.Reader`, `bufio.Reader`, `tls.Conn`, and most stream wrappers in the standard library.

### 3. Context wrapping

```go
type tracedContext struct {
	context.Context
	traceID string
}

func WithTraceID(ctx context.Context, id string) context.Context {
	return &tracedContext{Context: ctx, traceID: id}
}

func TraceID(ctx context.Context) string {
	if t, ok := ctx.(*tracedContext); ok {
		return t.traceID
	}
	return ""
}
```

Embedding `context.Context` preserves cancellation, deadlines, and values from the parent.

### 4. Sealing variants in a tagged union

```go
type expr interface{ isExpr() }

type baseExpr struct{}
func (baseExpr) isExpr() {}

type IntLit struct{ baseExpr; N int }
type Add    struct{ baseExpr; L, R expr }
type Neg    struct{ baseExpr; E expr }
```

Embedding `baseExpr` gives every variant the unexported sealing method for free.

### 5. Interface composition

```go
type ReadWriteCloser interface {
	io.Reader
	io.Writer
	io.Closer
}
```

Compose interfaces by embedding rather than re-listing methods. The standard library defines `io.ReadWriter`, `io.ReadCloser`, `io.WriteCloser`, `io.ReadWriteCloser` all by embedding.

## Anti-Patterns & Gotchas

**Treating embedding as inheritance.** No virtual dispatch. Methods on the inner type don't see the outer type. If you want polymorphism, define an interface.

**Exporting an embedded `sync.Mutex` you don't want callers to lock.** Embed an unexported named mutex field instead.

**Embedded nil interface fields.** Calling promoted methods panics. Default-initialize or check.

**Ambiguous promotion across two embedded fields with same-named methods.** Compile error on unqualified access. Either qualify or rename.

**Embedded methods calling back to the outer type.** They can't. If you need that, accept the outer type as a parameter or use an interface.

**Embedding `error` (the interface).** Compiles, but `var e MyError; e.Error()` panics because the inner `error` is nil. Instead: store an `error` in a named field, or define `Error()` directly.

**Multiple embeddings of "shape-shifting" types (`time.Time`, `*url.URL`).** Promoted methods include zero-value-unsafe ones; users can hit them by accident.

**Storage size from embedding by value.** Embedding a fat struct copies it on every outer-struct copy. Use `*T` if the inner is large.

## Performance Notes

- Promoted method calls compile to direct calls on the inner field. There is no v-table; no extra indirection beyond the field offset.
- Embedding by value vs pointer changes copy cost: `Outer{Inner T}` copies the whole `T` on assignment; `Outer{*T}` copies one word.
- Embedded `sync.Mutex` adds 8 bytes to the struct. Embedded `sync.RWMutex` adds 24 bytes. Plan struct layout accordingly.
- Embedding an interface puts a 16-byte (`iface`) header in your struct, plus the dynamic value's storage.
- Multiple levels of embedding (`A` embeds `B` embeds `C`) still resolve to a single offset at compile time — promotion is flattened.

## How Big Companies Use It

- **Kubernetes API types** all embed `metav1.TypeMeta` and `metav1.ObjectMeta`. Every resource gets `GetName()`, `GetNamespace()`, `GetUID()`, etc. for free. See [`staging/src/k8s.io/api/core/v1/types.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/core/v1/types.go).
- **`tls.Conn` embeds `net.Conn`** so TLS connections satisfy `net.Conn` automatically, falling through to the underlying transport for unimplemented methods. [`src/crypto/tls/conn.go`](https://github.com/golang/go/blob/master/src/crypto/tls/conn.go).
- **Cockroach `kvserver.Replica`** embeds several locked-state types via unexported fields; the outer type carefully refuses to expose them publicly.
- **HashiCorp Terraform's `helper/schema.Resource`** uses embedded interfaces extensively to allow custom CRUD overrides while keeping defaults.
- **Caddy's module configs** embed `caddy.Module` so every plugin gets `CaddyModule()` registration. [github.com/caddyserver/caddy](https://github.com/caddyserver/caddy).

## Source Code References

Pinned to `go1.26`.

- Method promotion / selector resolution: [`src/go/types/lookup.go`](https://github.com/golang/go/blob/master/src/go/types/lookup.go) — function `LookupFieldOrMethod`.
- Embedded interface semantics: [`src/go/types/interface.go`](https://github.com/golang/go/blob/master/src/go/types/interface.go).
- `metav1.ObjectMeta` (Kubernetes): https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
- `tls.Conn` embedding `net.Conn`: https://github.com/golang/go/blob/master/src/crypto/tls/conn.go
- `bufio.Reader` (does NOT embed; chose composition with named field — useful comparison): https://github.com/golang/go/blob/master/src/bufio/bufio.go

## Further Reading

- Spec, "Struct types — Promoted fields and methods": https://go.dev/ref/spec#Struct_types
- Effective Go, "Embedding": https://go.dev/doc/effective_go#embedding
- Go FAQ, "What does Go have instead of inheritance?": https://go.dev/doc/faq#inheritance
- Dave Cheney, "On declaring variables": https://dave.cheney.net/2014/05/24/on-declaring-variables (covers embedding ergonomics)
- "Composition with Go's interfaces and embedded types": https://research.swtch.com (Russ Cox)

## Exercises / Self-Check

1. Build a `LoggingReader` that decorates `io.Reader`, logging every successful `Read`. Why does the embedded interface make this trivial?
2. Demonstrate the ambiguity error: two embedded types with the same-named field, accessed without qualification. Then fix it by qualifying.
3. Why doesn't an embedded method see the outer type? Show the difference between "method called from outside" and "method called from within an inner method on the embedded type."
4. Embed `sync.Mutex` exported vs as an unexported named field. Show how the API surface differs for callers.
5. Use embedding to satisfy a 5-method interface with a single `*X` field plus one override. How would generics improve or fail to improve this?
