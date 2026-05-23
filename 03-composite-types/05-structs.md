# Structs

## TL;DR

A struct is a value-typed record of named fields. Fields are laid out in declaration order with **padding** inserted by the compiler to satisfy alignment. Field ordering can change a struct's size by tens of bytes — relevant when you allocate millions of them. Structs are comparable iff every field is comparable. Tags are arbitrary strings parsed by encoders, reflection, and validators. Anonymous fields enable **embedding**: composition with method promotion, not inheritance.

## Mental Model

```
type Bad struct {
    flag bool   // 1 byte
                // 7 bytes padding
    id   int64  // 8 bytes
    n    bool   // 1 byte
                // 7 bytes padding
}                // total: 24 bytes

type Good struct {
    id   int64  // 8 bytes
    flag bool   // 1 byte
    n    bool   // 1 byte
                // 6 bytes padding
}                // total: 16 bytes
```

Order big-to-small. The compiler does **not** reorder fields — what you write is what you get. `structlayout` (`go install honnef.co/go/tools/cmd/structlayout@latest`) prints the layout for you.

## Syntax & Basic Usage

```go
package main

import "fmt"

type Point struct {
	X, Y int
}

type Labeled struct {
	Point         // embedded (anonymous field)
	Label string
}

func main() {
	p := Point{1, 2}                 // positional
	p2 := Point{X: 3, Y: 4}          // named — preferred
	var p3 Point                     // zero value: Point{0,0}
	l := Labeled{Point{5, 6}, "hi"}

	fmt.Println(p, p2, p3, l, l.X)   // l.X promoted from Point
	// Output:
	// {1 2} {3 4} {0 0} {{5 6} hi} 5
}
```

Positional initializers are brittle: adding a field is a breaking change. Use named.

## Deep Dive

### Memory layout, alignment, padding

Each field is placed at the next address satisfying its alignment requirement:

```go
type S struct {
	a int8   // align 1, offset 0
	b int64  // align 8 → offset 8 (7 bytes padding after a)
	c int16  // align 2, offset 16
	d int32  // align 4 → offset 20 (2 bytes padding after c)
}            // size 24 (4 trailing bytes if struct alignment is 8)
```

The trailing pad makes the struct's size a multiple of its largest field's alignment so arrays of it stay aligned. Print actual offsets with:

```go
import (
	"fmt"
	"unsafe"
)

func main() {
	var s S
	fmt.Println(unsafe.Sizeof(s),
		unsafe.Offsetof(s.a),
		unsafe.Offsetof(s.b),
		unsafe.Offsetof(s.c),
		unsafe.Offsetof(s.d))
}
```

### Zero value is usable

```go
var b bytes.Buffer        // ready to write to, no constructor needed
var m sync.Mutex          // ready to Lock
var t time.Time           // valid; IsZero() == true
```

This is a deliberate design principle ("make the zero value useful"). When you design a struct, try to make `var x T{}` valid without a constructor.

### Comparability

```go
type Pt struct{ X, Y int }
Pt{1,2} == Pt{1,2}  // true

type Bad struct{ s []int }
// Bad{} == Bad{}    // compile error: slices are not comparable
```

Comparable structs can be map keys. `==` is field-wise. NaN floats break reflexivity (`NaN != NaN`), as everywhere.

### Tags

```go
type User struct {
	Name  string `json:"name" validate:"required"`
	Email string `json:"email,omitempty"`
}
```

Tags are an opaque string at the language level. The `reflect` package parses them via `StructTag.Get("json")`. Conventions:

- `json:"name,omitempty"` — encoding/json
- `xml:"name,attr"` — encoding/xml
- `yaml:"name"` — gopkg.in/yaml.v3
- `db:"name"` — sqlx
- `validate:"..."` — go-playground/validator
- `protobuf:"..."` — generated

### Embedding (anonymous fields)

```go
type Engine struct{ HP int }
func (e Engine) Start() { /* ... */ }

type Car struct {
	Engine        // embedded — Car.HP and Car.Start() are promoted
	Wheels int
}

var c Car
c.HP = 200    // == c.Engine.HP
c.Start()     // == c.Engine.Start()
```

Promotion has rules:

- Outer-most overrides inner: `Car.Start` (if defined) shadows `Engine.Start`.
- Ambiguity is a **compile error**, not silent precedence — if `Car` embeds two types each with a `Foo` method, you must qualify (`c.Engine.Foo()`).
- Embedding an interface gives you "default zero" behavior — the outer type satisfies the interface but any unimplemented method panics with a nil-pointer dereference. Useful for partial mocks.

```go
type Reader interface{ Read([]byte) (int, error) }
type StubReader struct{ Reader }  // panics if Read is called and no inner provided
```

### Anonymous structs

```go
person := struct {
	Name string
	Age  int
}{"Alice", 30}
```

Useful for one-shot test fixtures, table-driven test cases, and JSON request/response shapes that only one function cares about. Don't propagate them across package boundaries.

### Empty struct `struct{}`

Zero size. Common uses:

- `map[K]struct{}` as a set (no per-entry value cost).
- `chan struct{}` as a signal channel (`close(done)` to broadcast).
- Method receivers that need no state but want to satisfy an interface.

```go
done := make(chan struct{})
go func() { /* ... */ close(done) }()
<-done
```

### `structs.HostLayout` (since 1.25)

Marks a struct as having a layout that matches the host C ABI — useful when interoperating with `syscall` or `cgo`. Prevents the compiler from inserting hidden fields or reordering. See `structs` package.

```go
import "structs"

type Iovec struct {
	_   structs.HostLayout
	Base *byte
	Len  uint64
}
```

## Standard Library Hooks

- `reflect`: `Type.Field`, `Value.Field`, `StructTag.Get`/`Lookup`, `StructOf` (construct types at runtime).
- `encoding/json`, `encoding/xml`, `encoding/gob`: read struct tags.
- `unsafe`: `Sizeof`, `Alignof`, `Offsetof` — compile-time constants.
- `structs` (since 1.25): `HostLayout` marker.
- `cmp.Compare`, `slices.SortFunc` work nicely with structs and a custom less func.

## Real-World Patterns

### 1. Functional options on a config struct

```go
type Server struct {
	addr    string
	timeout time.Duration
	logger  *slog.Logger
}

type Option func(*Server)

func WithAddr(a string) Option           { return func(s *Server) { s.addr = a } }
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }
func WithLogger(l *slog.Logger) Option   { return func(s *Server) { s.logger = l } }

func NewServer(opts ...Option) *Server {
	s := &Server{
		addr:    ":8080",
		timeout: 30 * time.Second,
		logger:  slog.Default(),
	}
	for _, o := range opts {
		o(s)
	}
	return s
}
```

### 2. Builder pattern via method chaining

```go
type Query struct {
	table   string
	filters []string
	limit   int
}

func (q *Query) From(t string) *Query    { q.table = t; return q }
func (q *Query) Where(f string) *Query   { q.filters = append(q.filters, f); return q }
func (q *Query) Limit(n int) *Query      { q.limit = n; return q }

q := (&Query{}).From("users").Where("age > 18").Limit(10)
```

Use sparingly — functional options are usually clearer.

### 3. Read-only views via embedded interface

```go
type ReadOnly interface {
	Get(k string) (string, bool)
}

type Store struct{ /* ... */ }
func (s *Store) Get(string) (string, bool) { /* ... */ }
func (s *Store) Set(string, string)        { /* ... */ }

type roStore struct{ *Store } // exposes only ReadOnly methods? No — embedding exposes ALL.

// Correct pattern: a separate wrapper type
type ReadOnlyStore struct{ s *Store }
func (r ReadOnlyStore) Get(k string) (string, bool) { return r.s.Get(k) }
```

Embedding leaks all methods. If you want a strict subset, write a wrapper.

### 4. Tagged unions via interfaces and small structs

```go
type Event interface{ isEvent() }

type LoginEvent struct {
	UserID int64
	At     time.Time
}
func (LoginEvent) isEvent() {}

type LogoutEvent struct {
	UserID int64
}
func (LogoutEvent) isEvent() {}

func handle(e Event) {
	switch v := e.(type) {
	case LoginEvent:
		// ...
	case LogoutEvent:
		// ...
	}
	_ = v
}
```

The unexported `isEvent()` method seals the interface — only types in this package can implement it.

### 5. Pool-friendly value types

```go
type request struct {
	body []byte
	hdr  [16]byte
}

var pool = sync.Pool{New: func() any { return new(request) }}

func handle() {
	r := pool.Get().(*request)
	defer pool.Put(r)
	r.body = r.body[:0]
	// ...
}
```

The struct is reused; the inner slice's backing array is reused via `[:0]`.

## Anti-Patterns & Gotchas

**Field ordering for size**: ignoring it can cost you 30–50% on large structs. Sort `int64`/`uintptr`/pointers/strings/slices first, then `int32`/`float32`, then `int16`, then `bool`/`uint8`.

**Returning a big struct vs pointer**: For small (≤2-3 words) structs, value-return is fine and avoids the heap. For big ones, return `*T`. Profile before changing.

**Comparing structs containing time.Time** with `==`: works if both monotonic clocks are equal, often doesn't. Use `time.Equal`.

**Embedding for "code reuse" when it's actually inheritance.** Embedding is composition. If you find yourself overriding promoted methods and casting, you wanted a regular field, not embedding.

**Public fields with `omitempty` and an `int` zero value.** `Count int \`json:",omitempty"\`` will hide `Count: 0` — possibly not what you want. Use `*int` or a custom marshaler.

**Mutating a struct field obtained from a `map[K]Struct`**: doesn't compile. Use `map[K]*Struct` or read-modify-write.

**Defining `MarshalJSON` on a struct that embeds another struct with its own `MarshalJSON`**: the outer's promoted `MarshalJSON` may surprise you. Test what `json.Marshal` actually outputs.

**Using struct tags as documentation.** They're parsed by reflection; typos in tag keys silently break encoding. Use a vet pass or a linter (`govet` has a `structtag` check).

**Reflective tag parsing in hot paths.** `reflect` is slow. Generate code (`stringer`, `easyjson`, `sqlc`) or cache the reflect results.

## Performance Notes

- Sort fields big-to-small to shrink size. Use `structlayout` to verify.
- A struct passed by value is `memmove`d; large ones are slower than pointer-passing. Cross-over usually around 4-8 words.
- Struct fields are accessed via constant offsets — no extra indirection. This is faster than a `map[string]any` for record-like data.
- `unsafe.Sizeof(s)` is a compile-time constant; use it in tests to lock in struct size.
- A struct with all-pointer fields is GC-scanned in O(field count). A struct with all-scalar fields is not scanned at all.
- `sync.Mutex` is 8 bytes; embedding it inline is cheap. Don't allocate it separately.

## How Big Companies Use It

- **Kubernetes API types**: heavy use of embedding (`metav1.ObjectMeta` is embedded into every resource). See [`staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go`](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go).
- **Go runtime itself**: `runtime.g` (goroutine) struct is hand-tuned for cache-line layout. [`src/runtime/runtime2.go`](https://github.com/golang/go/blob/master/src/runtime/runtime2.go).
- **CockroachDB's `roachpb.BatchRequest`**: hand-ordered fields for size.
- **Dropbox's `gocyclo` / wire format structs**: code-generated to avoid reflection cost in hot paths.
- **Discord's GC-heavy state service** struggled with `map[Snowflake]*Member` where each Member was a fat struct full of pointers — see the famous blog post on GC pause.

## Source Code References

Pinned to `go1.26`.

- Struct type descriptor: [`src/internal/abi/type.go`](https://github.com/golang/go/blob/master/src/internal/abi/type.go) — `StructType` / `StructField`.
- Alignment and offset calculation: [`src/go/types/struct.go`](https://github.com/golang/go/blob/master/src/go/types/struct.go) plus [`src/cmd/compile/internal/types/size.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/types/size.go).
- `unsafe.Sizeof`/`Offsetof` lowering: [`src/cmd/compile/internal/typecheck/const.go`](https://github.com/golang/go/blob/master/src/cmd/compile/internal/typecheck/const.go).
- `structs.HostLayout` (since 1.25): [`src/structs/structs.go`](https://github.com/golang/go/blob/master/src/structs/structs.go).
- Reflection on struct tags: [`src/reflect/type.go`](https://github.com/golang/go/blob/master/src/reflect/type.go), search `StructTag`.

## Further Reading

- Spec, "Struct types": https://go.dev/ref/spec#Struct_types
- Go blog, "Go data structures": https://research.swtch.com/godata
- Dave Cheney, "Should methods be declared on T or *T?": https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t
- `structlayout` tool: https://pkg.go.dev/honnef.co/go/tools/cmd/structlayout
- "Padding is hard": https://dave.cheney.net/2015/10/09/padding-is-hard
- Effective Go on composition: https://go.dev/doc/effective_go#embedding

## Exercises / Self-Check

1. Compute the size of `struct{a bool; b int64; c bool; d int64}` vs `struct{b, d int64; a, c bool}`. Confirm with `unsafe.Sizeof`.
2. Why is `var b bytes.Buffer; b.WriteString("x")` valid without a constructor? Trace through `bytes.Buffer`'s fields.
3. Embed an `io.ReadCloser` interface in a struct. What happens if you call `Read` before assigning the inner interface?
4. Build a tagged-union (`Event`) hierarchy with three concrete types and a `switch`-based handler. Why is the sealing-method trick useful?
5. Write a struct intended for `sync.Pool` reuse. What do you reset between `Put` and `Get` to avoid leaking references?
