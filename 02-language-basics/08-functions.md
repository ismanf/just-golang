# Functions — Multiple Returns, Named Returns, Variadic, Function Values

## TL;DR

Go functions are first-class values, support **multiple return values** (the canonical way to return `(result, error)`), support **named returns** (useful for documentation and `defer`-based error wrapping), support **variadic parameters** via `...T`, and have **zero-cost local closures** (escape analysis decides heap vs stack). The mental model: a function is just a value with a signature; passing it around, storing it in a struct field, returning it from another function — all idiomatic. Five operational disciplines: (1) **return errors as the last value**, always; (2) **name returns sparingly** — they hurt readability in long functions; (3) **variadic at the call site** spreads a slice with `s...`; (4) **pass by value by default** — pointer parameters are an explicit signal of "I may mutate this or this is too large to copy"; (5) **function-typed parameters are interfaces' lightweight cousin** — `http.HandlerFunc` is the canonical example. Go 1.18 added generics; covered in `04-methods-interfaces-generics/03-generics.md`. The single biggest gotcha: **arguments are passed by value, but slices, maps, channels, interfaces, and pointers all CONTAIN a pointer** — passing them by value passes the header by value but the *underlying data* is shared. Mutations through the header still affect the original.

## Mental Model

```
   func name(params) returnType { body }
   func name(params) (returnTypes) { body }
   func name(params) (named ReturnTypes) { body }

   Parameter passing:
   ─────────────────
   Always BY VALUE.
   But these types CONTAIN pointers (shared underlying):
     slice, map, channel, interface, function, *T

   So:
     func add(s []int, v int) { s = append(s, v) }     // may not see growth
     func addInplace(s *[]int, v int) { *s = append(*s, v) }  // sees growth
     func mutateMap(m map[string]int) { m["k"] = 1 }   // sees mutation
```

## Basic Forms

```go
// No params, no return
func hello() { fmt.Println("hi") }

// One param, one return
func square(x int) int { return x * x }

// Multiple params, multiple returns
func divmod(a, b int) (int, int) {
    return a / b, a % b
}

// Same-type params can share the type annotation
func clamp(lo, hi, v int) int { ... }

// No return: just runs for side effects
func log(msg string) { ... }
```

## Multiple Return Values

The most distinctive Go function feature. The canonical use:

```go
func parseInt(s string) (int, error) {
    n, err := strconv.Atoi(s)
    return n, err
}
```

The convention is **error last**. Tools, linters, and convention all assume it.

Other patterns:

```go
// Value + ok (map lookup, type assertion, channel recv)
v, ok := m["key"]
v, ok := x.(int)
v, ok := <-ch

// Multiple results (math-like)
quotient, remainder := divmod(17, 5)

// Multiple things plus error
user, perms, err := loadUser(id)
```

Call sites can ignore returns with `_`:

```go
_, err := parseInt(s)   // discard the int
n, _ := parseInt("5")   // assume valid, discard error (vet may warn)
```

`go vet` has a `lostcancel` and `errcheck`-style discipline; most teams add the `errcheck` linter to enforce that errors aren't silently dropped.

## Named Returns

```go
func split(s, sep string) (left, right string) {
    i := strings.Index(s, sep)
    if i < 0 {
        left = s
        return
    }
    left, right = s[:i], s[i+len(sep):]
    return
}
```

Named returns:

- Are pre-declared as zero-valued variables.
- Allow `return` without arguments ("naked return") — uses current values.
- Show up in godoc as part of the signature.
- Can be mutated by deferred functions.

Common use: **error wrapping via defer**:

```go
func process(path string) (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("process %q: %w", path, err)
        }
    }()

    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close()
    // ... return any error raw; the defer wraps it
    return nil
}
```

**Anti-pattern**: long functions with naked returns. Readers can't tell what values are being returned. Style guides (Google, Uber) advise: name returns only when (a) the function is short, (b) you have a clear documentation reason, or (c) you use the deferred-wrap pattern.

## Variadic Parameters

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums { total += n }
    return total
}

sum()              // 0
sum(1)             // 1
sum(1, 2, 3)       // 6
```

`...int` makes a function accept any number of `int` arguments. Inside the function, the parameter is a regular `[]int`.

### Spread at the call site

```go
nums := []int{1, 2, 3}
sum(nums...)       // spread; equivalent to sum(1, 2, 3)
```

The `slice...` syntax is mandatory when passing a slice to a variadic. You can't pass `nums` directly without `...`.

### Rules

- The variadic parameter must be **last**.
- Only one variadic per function.
- Inside the function, the parameter is a slice; can be `nil` if no args passed.

```go
func log(prefix string, args ...any) {
    if len(args) == 0 { return }
    fmt.Printf(prefix + ": ")
    fmt.Println(args...)        // spread to Println which is itself variadic
}
```

### `fmt.Printf`-style

```go
func Printf(format string, args ...any) (int, error)
```

`any` (= `interface{}`) lets the variadic accept anything. The trade-off is type safety; usually for logging/formatting only.

## Functions as Values

```go
add := func(a, b int) int { return a + b }
fmt.Println(add(2, 3))

// Pass to another function
nums := []int{1, 2, 3}
result := transform(nums, func(x int) int { return x * x })

// Store in a struct
type Server struct {
    handler func(http.ResponseWriter, *http.Request)
}
```

Function values carry their closure environment (the variables they capture from the enclosing scope). See `09-closures.md`.

### Named function types

```go
type Comparator func(a, b int) int
type Middleware func(http.Handler) http.Handler
type EventHandler func(ctx context.Context, e Event) error
```

Named function types are conventional for: middleware chains, sort comparators, callback signatures, event handlers. They're useful for documentation and for attaching methods.

### Methods on function types

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f(w, r)
}
```

This is exactly `http.HandlerFunc` — the stdlib's adapter that lets bare functions implement the `http.Handler` interface. A pattern worth mastering.

## Passing Behaviour by Type

Go passes **all** arguments by value. The value-vs-pointer-semantics distinction is about **what's inside the value**:

| Type | Pass-by-value behaviour |
|------|------------------------|
| `int`, `bool`, `float64`, etc. | Copy of the value |
| `string` | Copy of the header (ptr + len); underlying bytes shared (immutable) |
| Array `[N]T` | Full copy of N elements |
| Struct | Full copy of all fields |
| Slice `[]T` | Copy of header (ptr + len + cap); underlying array shared |
| Map | Copy of map header (one pointer); same map |
| Channel | Copy of channel header (one pointer); same channel |
| Pointer `*T` | Copy of the pointer; same target |
| Interface | Copy of the (type, value) pair; same underlying value |
| Function | Copy of the func value; same underlying code + captured vars |

So `func add(s []int, v int) { s = append(s, v) }` does *not* change the caller's slice if append grows the underlying array — because the new (longer) slice header doesn't propagate back. To mutate the caller's view:

```go
func add(s *[]int, v int) { *s = append(*s, v) }
```

But `func setFirst(s []int, v int) { s[0] = v }` *does* change the caller's data — because `s[0]` writes through the shared underlying array.

This is the most common source of "Go's pass-by-value confused me" misunderstandings. The pass *is* by value; the values *happen to be* pointers.

## Pointer Receivers vs Value Receivers

```go
type Counter struct { n int }

func (c *Counter) Inc()    { c.n++ }     // pointer receiver: mutates
func (c Counter) Get() int { return c.n } // value receiver: copy
```

Covered in `04-methods-interfaces-generics/01-methods.md`. Quick rule:

- Pointer receiver if you mutate or if the struct is large.
- Value receiver if the type is small and immutable-ish.
- Be consistent within a type — don't mix.

## Recursion

```go
func factorial(n int) int {
    if n <= 1 { return 1 }
    return n * factorial(n - 1)
}
```

Goroutines have **growable stacks** (start at 2-8 KB, grow up to 1 GB by default). Deep recursion that would overflow C stacks works fine in Go up to ~100k frames depending on frame size.

No tail-call optimisation in current Go. Recursive code that *needs* TCO must be rewritten as iteration.

## Higher-Order Patterns

### Decorator / wrapping

```go
func withLogging(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println(r.URL.Path)
        h.ServeHTTP(w, r)
    })
}

mux.Handle("/api/", withLogging(apiMux))
```

### Functional options

```go
type Option func(*Server)
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

NewServer(WithTimeout(30*time.Second))
```

Covered in `19-patterns/01-functional-options.md`.

### `map`/`filter`/`reduce`

```go
// With generics (1.18+) and slices package (1.21+)
import "slices"

doubled := slices.Map(nums, func(x int) int { return x * 2 })  // hypothetical
// stdlib slices doesn't have Map yet; use a manual loop or
// github.com/samber/lo for FP-style helpers.
```

Pre-generics, Go culture was "just write the loop." With generics, more functional patterns are emerging, but the idiomatic style remains loop-oriented.

## Anonymous Functions (Closures)

```go
func main() {
    counter := 0
    inc := func() int { counter++; return counter }
    fmt.Println(inc(), inc(), inc())   // 1 2 3
}
```

Closures capture variables by reference. They allocate on the heap if the captured variables outlive the enclosing function. Covered in `09-closures.md`.

## Generics (Brief)

```go
func Max[T cmp.Ordered](a, b T) T {
    if a > b { return a }
    return b
}

Max(1, 2)
Max("apple", "banana")
Max(3.14, 2.71)
```

Covered in `04-methods-interfaces-generics/03-generics.md`. With generics, many "you'd need to write three versions" patterns reduce to one. The `min`/`max` builtins (1.21+) are themselves generic.

## `init()` Functions

```go
func init() {
    // runs before main(), once per package, after var initialisers
    if err := loadConfig(); err != nil { log.Fatal(err) }
}
```

Multiple `init` per file allowed. Cannot be called from user code. Order: var initialisers, then `init`s in source order within each file, then by alphabetical file order within a package, then by dependency order across packages.

Use sparingly. `init` is hard to test and creates implicit ordering. Prefer explicit initialisation when possible.

## Anti-Patterns & Gotchas

**Returning many values where a struct would do.** `func User() (id int, name string, email string, age int)` — make it `func User() UserData`.

**Naked returns in long functions.** Hides what's being returned.

**Variadic `...any` when you could be specific.** Loses type safety. Use specifically-typed variadic when possible.

**Calling a variadic with a slice without `...`.** Compile error.

**Assuming `func add(s []int)` modifies caller's slice header.** It doesn't if `append` grows. Pass `*[]int` or return the new slice.

**Discarding errors with `_`.** Use `errcheck` linter.

**`func() error { ... }()` immediately invoked for "scope".** Sometimes legit; often a defer-in-loop workaround that could be refactored.

**Returning `(value, error)` and checking `err == nil` to use `value`** — but `value` is non-nil even on error in some libraries. Always check err first; then use value.

**Mixing pointer and value receivers on the same type.** Subtle method-set issues. Pick one.

**Variadic in a hot path.** Each call allocates the backing slice. Pre-build a slice and use it explicitly.

**Long parameter lists.** Refactor to a config struct or use the functional-options pattern.

**Returning slices and modifying them in the caller without realizing they share the backing array.** Document if shared; copy if you want isolation.

**Function values stored long-term that capture huge structs in their closure.** Memory leak.

**`func()` with side effects in a `defer` argument** — argument captured at defer; the function only runs at return. Don't be surprised when "logging" doesn't fire when you expect.

**`func main()` doing real work.** It should set up + call into application logic. Easier to test the "real" function.

## Performance Notes

- **Function call overhead**: ~1-2 ns (call + return; modern CPUs).
- **Inlining**: small functions are inlined by the compiler; check with `go build -gcflags="-m"`.
- **Variadic call**: allocates a slice (~16 bytes header + N*sizeof(T) bytes); usually fast.
- **Closure**: allocates ~16-32 bytes per captured var if it escapes to heap.
- **`recover()` in deferred function**: ~5-10 ns; cheap enough to use defensively.
- **Pointer vs value receiver**: pointer is a copy of one word (8 bytes on 64-bit). Value receiver copies the entire struct. For large structs, pointer is much faster.

Use `go test -bench` + `-cpuprofile` to measure actual impact.

## How Big Companies Use It

- **Google's internal style guide** recommends multi-return rather than tuple types; reserves named returns for documentation or defer-wrap.
- **Uber's Go style guide** strongly discourages naked returns. Strongly recommends `errcheck`.
- **Kubernetes** uses functional options extensively in their builder pattern (`fake.NewSimpleClientset(WithCustomMetricsAPIServer(...))`).
- **Tailscale** uses small, single-purpose functions; their style guide emphasises "many small functions" over "few large ones."
- **CockroachDB** uses methods-on-function-types extensively for SQL row processing pipelines.
- **HashiCorp** uses variadic options for resource construction.

## Source Code References

- Go spec — Functions: https://go.dev/ref/spec#Function_types.
- Go spec — Function literals: https://go.dev/ref/spec#Function_literals.
- Go spec — Calls: https://go.dev/ref/spec#Calls.
- `errcheck`: https://github.com/kisielk/errcheck.
- `http.HandlerFunc`: https://github.com/golang/go/blob/master/src/net/http/server.go.
- Functional options pattern: https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html (Rob Pike).

## Further Reading

- Effective Go — Functions: https://go.dev/doc/effective_go#functions.
- Dave Cheney, "Don't return interfaces, accept interfaces" (related to function signatures).
- Rob Pike, "Self-referential functions and the design of options".
- "Practical Go: Real world advice for writing maintainable Go programs" (Dave Cheney).
- Uber Go style guide on functions: https://github.com/uber-go/guide.

## Exercises / Self-Check

1. Write a function with three return values. Use `_` to ignore the middle one.
2. Use named returns + defer to wrap an error with a context message.
3. Pass a slice to a function and append in the function. Observe that the caller's slice may not see the growth. Refactor with `*[]int` or by returning the new slice.
4. Write a variadic function that accepts `...int`. Call with no args, with three args, and with a spread slice.
5. Define a `type Comparator func(a, b int) int`. Use it to sort a slice via `sort.Slice`.
6. Implement a `withLogging` HTTP middleware. Wrap a handler; verify logs appear.
7. Use functional options to construct a `Server` with optional `WithTimeout`, `WithLogger`.
8. Profile a hot loop calling a variadic vs the same loop with explicit args. Quantify allocation difference.
