# Variables and Constants — `var`, `:=`, `const`, Untyped Constants, `iota`

## TL;DR

Go has two binding keywords (`var` and `const`) and one shorthand (`:=`), each with strict rules about where they live. The mental model is straightforward but the **untyped constants** system is unique to Go and is what makes literals like `1 << 32`, `0.1 + 0.2`, or `math.Pi` work cleanly across `int`, `float64`, `complex128` without overflow or precision loss at compile time. Three rules cover most needs: (1) at package scope you must use `var`, `const`, `type`, or `func` — `:=` is statement-level only; (2) declared-but-unused **variables** are a compile error, declared-but-unused **constants/imports** are not; (3) untyped constants are infinite-precision until the moment they're assigned to a typed location, at which point conversion rules kick in. The single biggest gotcha across all teams: **`:=` re-uses existing variables on the left if any *new* variable is on the left** — `err` shadowing inside `if x, err := foo(); err != nil { ... }` traps a different `err` than the outer one. Go 1.22's loop variable change (each iteration gets its own `i`/`v`) is the most important language behaviour change in a decade; covered in `09-closures.md`.

## Mental Model

```
   Declaration forms
   ─────────────────

   var x int              // explicit type, zero value
   var x = 42             // type inferred
   var x int = 42         // both
   var (                  // grouped
       a int
       b string
   )

   x := 42                // short declaration; ONLY inside functions
   a, b := 1, "x"

   const Pi = 3.14159     // untyped constant
   const N int = 100      // typed constant
   const (                // grouped
       KB = 1 << 10
       MB = 1 << 20
   )

   Scope ladder
   ────────────
   universe  (predeclared: int, true, nil, len, ...)
       │
       └── package  (var/const/type/func at file top level)
              │
              └── function
                     │
                     └── block  ({ })
                            │
                            └── statement (for init, if init, switch init)
```

## `var` — Declaration in Every Scope

```go
// Package scope
var (
    DefaultTimeout = 30 * time.Second
    httpClient     = &http.Client{Timeout: DefaultTimeout}
    once           sync.Once
)

// Function scope
func handler(r *http.Request) {
    var (
        start = time.Now()
        ctx   = r.Context()
    )
    _ = start; _ = ctx
}
```

`var` is the only declaration form that works at **package scope**. Use it for:

- Package-level state (with explicit `var` for visibility).
- Variables you want to declare without initialisation (`var buf []byte`).
- Multiple declarations grouped together (`var ( ... )`).
- When the type matters for documentation even with an initialiser (`var srv *http.Server = newServer()`).

Initialisers at package scope run **in dependency order** (Go computes the DAG), then in source order within each file. Initialisation of multiple files in a package follows alphabetical file order — but rely on this *only* for the `init()` function ordering (covered in `06-packages-modules`).

## `:=` — Short Declaration (Function Scope Only)

```go
func process(r *http.Request) error {
    body, err := io.ReadAll(r.Body)        // both new
    if err != nil { return err }

    // Re-using err — also legal
    parsed, err := json.Marshal(body)      // parsed new, err reassigned
    _ = parsed
    return err
}
```

Rules:

- At least **one** variable on the left must be new.
- All others on the left are **reassigned** (not redeclared), keeping their original type.
- Cannot be used at package scope (compile error).

### The shadowing trap

```go
// BUG: outer err is shadowed
var err error
if x, err := foo(); err != nil {   // this err is NEW inside the if-init
    return err
}
fmt.Println(err)   // always nil
```

The `if`-init introduces a new scope; `err` declared there shadows the outer `err`. Tools like `go vet`'s `-shadow` flag (run via `shadow` analyzer in `golang.org/x/tools/go/analysis/passes/shadow`) catches this — wire it into your CI.

### `_` blank identifier

```go
_, err := io.ReadAll(r.Body)       // ignore the bytes
for _, v := range slice { ... }    // ignore the index
var _ Service = (*MyImpl)(nil)     // compile-time interface assertion
```

`_` is not a variable; it's a syntactic placeholder. Assignments to it cost nothing.

## `const` — Compile-Time Constants

```go
const (
    Pi          = 3.14159265358979323846   // untyped float, infinite precision
    MaxRetries  = 5                         // untyped int
    Greeting    = "hello"                   // untyped string
    DefaultDur  = 30 * time.Second          // typed: time.Duration
)
```

Constants must be **expressible at compile time**: literals, arithmetic on constants, `len`/`cap` of arrays, `unsafe.Sizeof` on types. No function calls (except `len`/`cap`/`unsafe.*`), no slice/map literals, no `time.Now()`.

### Why constants matter

```go
const KB = 1 << 10
const MB = 1 << 20

func cacheSize(mb int) int {
    return mb * MB
}

cacheSize(64)   // computed as 64 * 1048576 = 67108864 — at compile time
```

The compiler folds constant expressions; runtime sees a literal `67108864`. Constants are zero-cost.

### Typed vs untyped constants — the killer feature

```go
const x = 1 << 32          // untyped — fits in int64; not an int (which might be 32-bit)
var a int64 = x            // OK
var b int = x              // compile error on 32-bit GOARCH

const y int = 1 << 32      // typed int — REJECTED if int is 32-bit
```

Untyped constants have **infinite precision** at compile time. They're "implicitly converted" to whatever typed location they're assigned to, with overflow checked then.

```go
const Pi = 3.14159265358979323846264338327950288   // ~30 digits — Go keeps them all
var f float32 = Pi          // rounds to float32 precision
var d float64 = Pi          // keeps float64 precision
```

Compare to languages where literals carry a fixed type — `3.14` would lose digits *as you write it*.

### Default types of untyped constants

When an untyped constant is forced into a typed context with no explicit type:

| Constant kind | Default type |
|---------------|--------------|
| Integer       | `int` |
| Floating-point| `float64` |
| Rune          | `rune` (i.e. `int32`) |
| String        | `string` |
| Boolean       | `bool` |
| Complex       | `complex128` |

```go
x := 42         // x is int
y := 3.14       // y is float64
s := "hi"       // s is string
r := 'A'        // r is rune (int32)
```

`:=` triggers the default type promotion. Explicit `var x int32 = 42` keeps the smaller type.

### Constant overflow at conversion

```go
const N = 1 << 32
var x int8 = N      // compile error: constant 4294967296 overflows int8

var n int = N
var y int8 = int8(n)   // OK at compile; truncates at runtime to 0
```

Compile-time constant overflow is *always* an error. Runtime conversions can silently truncate (or panic for float→int with NaN/inf — see `04-type-conversion-and-assertions.md`).

## `iota` — Sequential Constant Generator

```go
const (
    Sunday    = iota   // 0
    Monday              // 1 (iota = 1)
    Tuesday             // 2
    Wednesday           // 3
    Thursday            // 4
    Friday              // 5
    Saturday            // 6
)
```

`iota` resets to 0 at each `const` block and increments by 1 per *line* (not per name). When the right-hand side is omitted, it repeats the previous line's expression.

Common patterns are in `10-iota-deep-dive.md`. The takeaway: iota's elegance comes from "the expression repeats."

## Multiple Assignment

```go
a, b := 1, 2
a, b = b, a               // swap — atomic at the statement level (Go evaluates RHS first)

x, y, z := f(), g(), h()   // function calls happen left to right
```

The RHS is evaluated in full before any LHS is assigned. So `a, b = b, a` is the canonical swap — no temporary required.

For multi-value returns:

```go
func divmod(a, b int) (int, int) { return a / b, a % b }

q, r := divmod(17, 5)
q, _ := divmod(17, 5)              // discard remainder
```

## Scoping Rules

```go
const X = 10           // package scope; visible everywhere in package

func f() {
    x := 20            // function scope
    {
        x := 30        // block scope; shadows the outer x
        fmt.Println(x) // 30
    }
    fmt.Println(x)     // 20
}
```

`if`-init, `for`-init/post, `switch`-init introduce **their own scope** for their block:

```go
if v, err := f(); err == nil {
    use(v)             // v in scope
}
// v NOT in scope here
```

This is *why* the shadowing trap is so common. Hoist with `var` if you need to use a value after the conditional:

```go
v, err := f()
if err != nil { return err }
use(v)
```

## Package-Scope Initialization Order

```go
// file_a.go
var a = b + 1       // depends on b

// file_b.go
var b = 42
```

Go computes the dependency DAG and initialises in topological order. If a file has no inter-file dependencies, files are processed alphabetically — but write code that doesn't depend on this.

If you need *imperative* initialisation:

```go
func init() {
    // runs after var initialisers in this file
    if err := loadConfig(); err != nil {
        log.Fatal(err)
    }
}
```

Multiple `init()` functions per file are allowed; they run in source order. Across files in a package, alphabetical file order. Across packages, dependency order (imported packages init first).

## Visibility (Exported vs Unexported)

```go
var PublicVar int    // exported (starts with capital letter)
var privateVar int   // unexported

const InternalPi = 3.14   // exported
const internalE = 2.718   // unexported
```

The single language rule: identifiers starting with an uppercase letter are exported (visible outside the package); lowercase are package-private. There's no `public`/`private` keyword. This rule applies to all package-scope identifiers: vars, consts, types, functions, methods, struct fields.

## Anti-Patterns & Gotchas

**Using `:=` at package scope.** Compile error. Use `var`.

**Shadowing in `if`-init / `for`-init.** Outer variable is captured but the inner assignment doesn't propagate. Use `go vet -shadow` or `shadow` linter.

**Declared and unused.** `x := 5` with no use is a compile error. Use `_` if intentional or remove. Imports are also subject to the rule.

**`const x = float64(time.Hour)`** — `time.Hour` is a typed const; you can't convert it in a const expression in a way that creates a new typed const without `float64()` itself being a conversion, which produces a non-constant value. Compile error. Use `const x = float64(60 * 60 * 1e9)` if you need a constant.

**Confusing default int (`int`) with explicit int32/int64.** A constant `1 << 31` is fine; but `var n int32 = 1 << 31` overflows.

**Forgetting that constants of type `time.Duration` are integers (nanoseconds).** `const T = time.Hour * 24` is fine; `const T = time.Hour * 24 * 30` is fine; multiplying durations is also fine because durations are typed ints under the hood.

**Top-level `var foo = compute()` with a slow `compute`.** `var` initialisers run at program start. A 5s init means a 5s startup. Use lazy initialisation (`sync.Once`) if needed.

**Mutating package-scope vars from many goroutines.** Data race. Either make immutable (`const`), guard with a mutex, or use `atomic`.

**Assuming `iota` increments mid-line.** It increments per line. See `10-iota-deep-dive.md` for the rules.

**Using `_ = x` to "use" a variable to silence the unused error.** Sometimes legit (placeholder), often a smell — usually the right fix is to delete `x`.

**Re-using `err` with `:=` across many lines and forgetting it stays the same variable.** Subtle when scopes are nested.

**Initialising package-scope `var` in init order that the linker happens to process favourably today.** Don't depend on file order across files. Wire dependencies explicitly.

**Treating `const Now = time.Now()` as legal.** It's not — `time.Now()` is a function call, not a compile-time expression.

**Confusing `const x int = 5` with `var x int = 5`.** The const can be used where a constant expression is required (array sizes, case labels); the var cannot.

## Performance Notes

- **Constants**: compile-time only. Zero runtime cost. The compiler folds them into instructions.
- **Package-level vars**: initialised once at startup; zero cost thereafter except for memory.
- **`:=`**: no runtime difference vs `var`. Identical machine code.
- **Untyped constant arithmetic**: happens at compile time. `1 << 32` compiles to a literal.

## How Big Companies Use It

- **Google** internally prefers grouped `var ( ... )` and `const ( ... )` blocks for package state — easier to scan.
- **Uber's Go style guide** advises `const` blocks for all magic numbers and forbids string literals in production code (except in tests).
- **Kubernetes** uses extensive `iota`-based enum constants for resource states, condition types, and event reasons.
- **Cloudflare** uses package-level `var` for global pools (`sync.Pool`, `http.Client`) initialised lazily via `sync.Once`.
- **Tailscale**'s style guide minimises package-level state to reduce coupling — prefer passing dependencies through struct fields rather than reaching for package vars.
- **HashiCorp** projects often group all package-level constants in a `consts.go` file.

## Source Code References

- Go spec — Declarations and scope: https://go.dev/ref/spec#Declarations_and_scope.
- Go spec — Constants: https://go.dev/ref/spec#Constants.
- Go spec — Short variable declarations: https://go.dev/ref/spec#Short_variable_declarations.
- `cmd/compile` constant folding: https://github.com/golang/go/blob/master/src/cmd/compile/internal/ir/const.go.
- shadow analyzer: https://github.com/golang/tools/blob/master/go/analysis/passes/shadow/.
- Effective Go — Declarations: https://go.dev/doc/effective_go#declarations.

## Further Reading

- Rob Pike, "Constants in Go": https://go.dev/blog/constants.
- Dave Cheney, "Constants and the const keyword": https://dave.cheney.net/2014/03/25/the-empty-struct (related discussion).
- Go FAQ — "Why does Go not have a ternary operator?": addresses the philosophy of declarations.
- Go FAQ — Visibility and naming: https://go.dev/doc/faq#case.
- Effective Go: https://go.dev/doc/effective_go.

## Exercises / Self-Check

1. Try `x := 5` at package scope. Note the compile error. Convert to `var x = 5` or `const x = 5`.
2. Declare a typed `const N int32 = 1 << 32`. Note the overflow. Change to untyped `const N = 1 << 32` and assign to `int64`.
3. Write code that triggers the `if-init` shadowing bug. Run `go vet -vettool=$(which shadow) ./...` and observe the warning.
4. Use `iota` to define a `Permission` flag enum: `Read = 1 << iota`, `Write`, `Execute`. Combine flags with `|`; check with `&`.
5. Demonstrate that `var a = b + 1` followed (in another file) by `var b = 42` initialises in the right order. Then remove the dependency; observe the alphabetical fallback.
6. Run a benchmark of `for i := 0; i < N; i++` with N as a `const` vs N as a `var`. Confirm the compiler inlines / constant-folds the const case.
7. Try `const T = time.Hour * 24 * 365`. Confirm it's a valid constant. Try `const T = time.Now()`. Note the error.
8. Use multiple assignment to swap two variables; confirm no temporary is needed.
