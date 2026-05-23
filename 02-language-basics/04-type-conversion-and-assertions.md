# Type Conversion and Assertions — `T(x)`, `x.(T)`, Comma-Ok, Type Switch

## TL;DR

Go has two superficially-similar but fundamentally-different operations: **type conversion** (`T(x)`) is a *compile-time* check that turns one type into another via the language's conversion rules — works between numeric types, between strings and `[]byte`/`[]rune`, between named types with the same underlying type. **Type assertion** (`x.(T)`) is a *runtime* check that extracts a concrete type from an interface value — works only on interface-typed values and panics if the dynamic type isn't `T` (unless you use the **comma-ok** form `v, ok := x.(T)`). The mental model: conversions are about "I know how to represent this value in another type"; assertions are about "I have an interface, I think it holds a `T`, let me unwrap it." Go 1.18+ added generics, which removed most "interface{} unboxing" use cases — modern code uses type assertions mainly for **`error` chains** (`errors.As`), **HTTP handler wrapping** (`if u, ok := w.(http.Unwrapper)`), and **interface satisfaction checks** (`if c, ok := obj.(io.Closer); ok { c.Close() }`). The single biggest gotcha: **assertions on a nil interface always fail**, regardless of the asserted type — `var i any = nil; _, ok := i.(int)` gives `ok == false`, not "ok=true, value=0."

## Mental Model

```
   Type conversion        Type assertion
   ──────────────         ───────────────
   T(x)                   x.(T)
   compile-time           runtime
   between known types    interface → concrete type
   never panics           panics on mismatch (unless comma-ok)

   Conversion rules:                Assertion rules:
   - numeric → numeric              - x must be an interface
   - string ↔ []byte                - T can be concrete or interface
   - string ↔ []rune                - if T is interface: succeed if x's
   - named ↔ underlying               type satisfies T
     same underlying type           - if T is concrete: succeed if x's
   - explicit only                    dynamic type IS exactly T
```

## Conversions: `T(x)`

### Numeric

```go
var i int = 1000
var f float64 = float64(i)    // 1000.0
var b byte = byte(i)          // 232 (1000 % 256)
var i32 int32 = int32(i)      // 1000
```

Numeric conversions never panic. They truncate, wrap, or round silently:

```go
int8(300)             // 44 (300 - 256)
int(math.Inf(1))      // implementation-defined; possibly MinInt
int(math.NaN())       // implementation-defined; on amd64: MinInt
uint8(-1)             // 255 (wraps via two's complement view)
float32(math.MaxFloat64)  // +Inf (overflow)
```

For "safe" conversions, check bounds first or use `math/big`:

```go
if x > math.MaxInt32 || x < math.MinInt32 { return errOverflow }
y := int32(x)
```

### String ↔ `[]byte` / `[]rune`

```go
s := "héllo"
b := []byte(s)        // bytes — UTF-8 encoded; len(b) > len("hello")
r := []rune(s)        // code points; len(r) is the rune count

s2 := string(b)       // back to string
s3 := string(r)       // back to string
```

Two important conversion subtleties:

```go
// string(int) — converts the int as a CODE POINT, not as digits
s := string(65)       // "A" — not "65" (go vet flags this)
s := string(rune(65)) // "A" — explicit; clearer; doesn't trigger vet

// strconv for "int as decimal text"
s := strconv.Itoa(65) // "65"
```

`string(int)` is a long-standing Go footgun. Use `strconv.Itoa` for text formatting. The compiler emits a vet warning since 1.15.

### Named types ↔ underlying type

```go
type UserID int64
type OrderID int64

var u UserID = 42
var o OrderID = OrderID(u)   // explicit conversion required, even though both are int64

var n int64 = int64(u)        // also explicit
```

Named types **don't auto-convert**, even when their underlying types match. This is a feature: it prevents accidentally passing a `UserID` where an `OrderID` is expected.

```go
func processUser(u UserID) { ... }

var o OrderID = 42
processUser(o)            // compile error
processUser(UserID(o))    // OK if intentional
```

### Pointers — `unsafe.Pointer` only

```go
var p *int
var q *float64 = (*float64)(unsafe.Pointer(p))   // ONLY via unsafe
```

Go forbids pointer-type conversions in safe code. To reinterpret bits, use `unsafe.Pointer` as the intermediate — see `13-unsafe`. Almost always a sign of FFI, serialisation, or premature optimisation.

### Function and channel conversions

```go
type Handler func(http.ResponseWriter, *http.Request)
type AdapterFunc func(http.ResponseWriter, *http.Request)

var h Handler
var a AdapterFunc = AdapterFunc(h)   // OK — same underlying type
```

Channels:

```go
var c chan int          // bidirectional
var r <-chan int = c    // receive-only — assignment, no conversion needed
var s chan<- int = c    // send-only — assignment

var r2 <-chan int = (<-chan int)(c)   // explicit conversion also legal
```

Read-only / write-only channel restriction is a one-way operation. You can narrow a bidirectional channel to a unidirectional one (and pass it that way to a function), but not widen.

## Type Assertions: `x.(T)`

```go
var i any = "hello"

s := i.(string)         // OK — i's dynamic type IS string
n := i.(int)            // PANIC — i's dynamic type is string, not int

s, ok := i.(string)     // OK — ok = true
n, ok := i.(int)        // n = 0, ok = false
```

The **comma-ok form** is non-panicking. Use it whenever you're not 100% certain of the dynamic type.

### Asserting to an interface

```go
type Closer interface { Close() error }

var i any = someFile         // *os.File implements Close
c, ok := i.(Closer)
if ok { c.Close() }
```

When `T` is an interface, the assertion succeeds if `x`'s dynamic type satisfies `T`. This is how you do **capability detection** at runtime:

```go
func tryClose(x any) {
    if c, ok := x.(io.Closer); ok {
        c.Close()
    }
}
```

### Nil interface

```go
var i any                 // i is nil
_, ok := i.(int)          // ok = false

var p *int                // nil typed pointer
i = p                     // i is NOT nil (has type *int, value nil)
_, ok = i.(*int)          // ok = true!  value is nil
```

Nil interface (both type and value are nil) fails any assertion. A nil pointer wrapped in an interface succeeds the pointer-type assertion, but the resulting pointer is still nil — dereference at your peril.

### Performance

Type assertions are cheap (~1-2 ns on modern CPUs) — they're a single pointer comparison in the runtime's interface table. Don't avoid them on performance grounds in normal code.

## Type Switch: `switch x := v.(type)`

Multi-case form of assertion:

```go
func describe(i any) string {
    switch v := i.(type) {
    case nil:
        return "nil"
    case int:
        return fmt.Sprintf("int: %d", v)        // v is int
    case string:
        return fmt.Sprintf("string: %q", v)     // v is string
    case []byte:
        return fmt.Sprintf("bytes: %x", v)      // v is []byte
    case io.Reader:
        return "reader"                         // v is io.Reader
    case interface{ Close() error }:
        return "closeable"                      // ad-hoc interface
    default:
        return fmt.Sprintf("unknown: %T", v)
    }
}
```

`v` inside each case has the asserted type. Multiple types in one case:

```go
case int, int32, int64:
    // v is type any (because the types don't agree)
    return fmt.Sprintf("integer of some kind: %v", v)
```

When the case lists multiple types, `v` remains of the original interface type (you can't use `v` as a specific numeric type) — useful for grouping into "treat all of these as X."

## `errors.Is` / `errors.As` — The Modern Assertion

Type assertion on errors is the most common modern use:

```go
import "errors"

var ErrNotFound = errors.New("not found")

func getUser(id string) (*User, error) {
    return nil, fmt.Errorf("repo: %w", ErrNotFound)   // wraps
}

err := getUser("42")
if errors.Is(err, ErrNotFound) {       // checks the chain
    // handle
}

var pe *fs.PathError
if errors.As(err, &pe) {               // unwraps to a specific concrete type
    log.Println("path:", pe.Path)
}
```

`errors.As` is **assertion + unwrap chain walk + assignment**. Prefer it over raw `err.(*fs.PathError)` because it handles wrapped errors (via `Unwrap` chain).

The classic gotcha:

```go
// WRONG — doesn't unwrap
if pe, ok := err.(*fs.PathError); ok { ... }

// RIGHT
var pe *fs.PathError
if errors.As(err, &pe) { ... }
```

Covered in `05-errors`.

## Interface-Satisfaction Checks

```go
type Service struct{}

// Compile-time assertion that *Service satisfies an interface
var _ http.Handler = (*Service)(nil)
```

The `var _ T = ...` idiom is used to fail at compile time if `Service` doesn't satisfy `http.Handler`. Zero runtime cost; lives in package scope.

For runtime detection:

```go
func wrap(h http.Handler) http.Handler {
    if _, ok := h.(http.Hijacker); ok {
        // log that the wrapper might break hijacking
    }
    return ...
}
```

## Converting Between Interfaces

If `A` and `B` are interfaces and `A`'s method set ⊆ `B`'s method set, then `A` is assignable to `B` directly (no assertion):

```go
type Reader interface { Read([]byte) (int, error) }
type ReadCloser interface { Read([]byte) (int, error); Close() error }

var rc ReadCloser = someFile
var r Reader = rc        // OK — direct assignment, no conversion needed
```

Going the other direction needs assertion (the runtime check that `r`'s concrete type also satisfies `Close`):

```go
rc, ok := r.(ReadCloser)
```

## Anti-Patterns & Gotchas

**`i.(T)` without comma-ok when unsure.** Panic.

**Type-switch on a generic-typed parameter.** With Go generics, you often don't need this. Use type constraints instead.

**`string(int)` to convert a number to its text form.** Use `strconv.Itoa`. The `string(int)` form treats the int as a Unicode code point.

**Forgetting nil-pointer-in-interface gotcha when asserting.** `i.(*T)` succeeds but result is nil; calling methods panics.

**`errors.As(err, pe)` instead of `errors.As(err, &pe)`.** `errors.As` needs a pointer to the target.

**`if e, ok := err.(*MyError); ok` not unwrapping.** Use `errors.As`.

**Converting `*X` to `*Y` "to reinterpret" via clever code.** Use `unsafe.Pointer`; don't try to fool the compiler.

**Heavy type switching where generics would be cleaner.** Modernise.

**Asserting to `any` ("for type erasure") — pointless.** Every concrete type satisfies `any`; the assertion is a no-op (you already have the interface).

**Confusing assertion `v.(T)` with conversion `T(v)`.** They look similar; do entirely different things.

**Using `reflect.TypeOf(i).String()` for runtime type info when a type switch would suffice.** Reflect is much slower and harder to read.

**Numeric conversions that silently truncate large values.** Always bounds-check.

**Repeatedly asserting the same value.** Cache the result:
```go
// BAD
if v, ok := x.(Foo); ok { use(v.A) }
if v, ok := x.(Foo); ok { use(v.B) }
// GOOD
v, ok := x.(Foo)
if ok { use(v.A); use(v.B) }
```

**Assuming `string([]byte{...})` is zero-copy.** It always copies (string immutability requires it). For zero-copy reads, use `unsafe.String` (1.20+); see `13-unsafe`.

## Performance Notes

- **Numeric conversions**: 0-1 CPU instructions; effectively free.
- **`string ↔ []byte`**: O(N) — copies. Don't do in hot loops; use `bytes.Buffer` / `strings.Builder` if mutating.
- **`string ↔ []rune`**: O(N) — UTF-8 decode pass.
- **Type assertion (concrete)**: ~1-2 ns; pointer comparison in itab.
- **Type assertion (interface)**: ~5-10 ns; itab lookup or build.
- **Type switch**: ~ same as one assertion per non-trivial case; jump-tabled.
- **`errors.As`**: walks the unwrap chain — O(chain depth); each step is a `Unwrap()` call.

## How Big Companies Use It

- **Google's internal style** strongly prefers `errors.As` / `errors.Is` over raw assertions on errors.
- **Kubernetes** uses extensive type switches in `runtime.Object` handling; with generics adoption now decreasing.
- **HashiCorp** uses interface-satisfaction `var _ T = (*X)(nil)` idiomatically in every package.
- **Cloudflare** uses type assertions sparingly in hot paths; preferred pattern is generic code with `any` constraint.
- **CockroachDB** uses type switches in SQL expression evaluation but is migrating to generic interpreters.
- **etcd** uses assertions for HTTP handler wrappers and gRPC metadata interpretation.

The Go community as a whole has moved away from "untyped interface + type switch" toward generics where possible. Type assertion remains essential for error handling, HTTP/middleware capability detection, and reflection-adjacent code.

## Source Code References

- Go spec — Conversions: https://go.dev/ref/spec#Conversions.
- Go spec — Type assertions: https://go.dev/ref/spec#Type_assertions.
- Go spec — Type switches: https://go.dev/ref/spec#Type_switches.
- `errors.As` / `errors.Is`: https://github.com/golang/go/blob/master/src/errors/wrap.go.
- Type-assertion runtime (itab): https://github.com/golang/go/blob/master/src/runtime/iface.go.
- vet's `stringintconv` analyser: https://github.com/golang/tools/tree/master/go/analysis/passes/stringintconv.

## Further Reading

- "Working with Errors in Go 1.13" (Damien Neil): https://go.dev/blog/go1.13-errors.
- Go FAQ — "How do I get a pointer to a value in an interface?": https://go.dev/doc/faq.
- "The Laws of Reflection" (Rob Pike): https://go.dev/blog/laws-of-reflection.
- Dave Cheney, "Avoid empty interface": https://dave.cheney.net/2014/03/25/the-empty-struct (adjacent topic).
- "Why does Go not have implicit numeric conversion?" — Go FAQ.

## Exercises / Self-Check

1. Convert `int(3.7)`. Confirm truncation toward zero. Try `int(-3.7)`.
2. Run `string(65)` and compare to `strconv.Itoa(65)`. Note the difference and the `go vet` warning.
3. Define `type UserID int64`. Try `var u UserID = 5; var n int = u`. Note the compile error; add an explicit conversion.
4. Assert a nil interface to a type with `i.(int)`. Observe panic. Use the comma-ok form.
5. Wrap an error with `fmt.Errorf("...: %w", innerErr)`. Use `errors.As` to extract the inner type; confirm it works through the wrap.
6. Use a type switch over `any` with cases for `int`, `string`, `[]byte`, and `default`. Verify each branch.
7. Add a `var _ http.Handler = (*MyHandler)(nil)` assertion. Break `MyHandler`'s method set; observe the compile error.
8. Convert `[]byte("hello")` to `string` and back; profile the allocations with `go test -benchmem`.
