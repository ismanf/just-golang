# Primitive Types — Int Sizes, Float, Complex, Bool, Byte, Rune

## TL;DR

Go's primitive types are deliberately small and explicit: 11 integer types (signed and unsigned at 8/16/32/64-bit, plus the platform-sized `int`/`uint`/`uintptr`), 2 floating-point (`float32`, `float64`), 2 complex (`complex64`, `complex128`), and `bool`, `string`, `byte` (alias for `uint8`), `rune` (alias for `int32`). The mental model: **`int` is for "an integer count of things on this machine"** (its size depends on GOARCH — 32-bit on 32-bit targets, 64-bit on 64-bit targets), while **`int32` / `int64` are for "exactly N bits regardless of platform"** (file formats, network protocols, hashing). Three rules cover most needs: (1) **no implicit numeric conversions** — `int + int32` is a compile error, you must convert; (2) **integer overflow wraps silently** (two's complement) — `int8(127) + 1 == -128`; (3) **integer division truncates toward zero** for positive results, and **division/modulo by zero panics for integers but produces `+Inf`/`NaN` for floats**. Go 1.21+ added `min`, `max`, `clear` builtins that work generically on ordered types. The single biggest gotcha across teams: **`byte` is `uint8` and `rune` is `int32`** — they look like first-class types in code but are pure aliases; `[]byte("hello")` and `[]uint8("hello")` are identical, and `'A'` is an `int32`, not a `byte`.

## Mental Model

```
                  Integer        Unsigned       Float       Complex
                  ─────────     ──────────     ───────     ────────
   8-bit          int8           uint8 (byte)
   16-bit         int16          uint16
   32-bit         int32 (rune)   uint32         float32     complex64
   64-bit         int64          uint64         float64     complex128

   Platform       int (32/64)    uint (32/64)
   Pointer-sized                                uintptr

   Other          bool, string
```

Two invariants:

1. **All numeric types are distinct.** `int(5) + int32(3)` is a compile error. You must `int(x) + int(y)` or `int32(x) + int32(y)`.
2. **Bit width is fixed at the source level.** `int32` is always 32 bits. `int` is "natural word size" — 64 on amd64/arm64, 32 on i386/arm.

## Integer Types

| Type | Bits | Range |
|------|------|-------|
| `int8`   | 8  | -128 .. 127 |
| `int16`  | 16 | -32,768 .. 32,767 |
| `int32`  | 32 | -2,147,483,648 .. 2,147,483,647 |
| `int64`  | 64 | -9.2e18 .. 9.2e18 |
| `int`    | 32 or 64 | platform-dependent |
| `uint8` (`byte`) | 8 | 0 .. 255 |
| `uint16` | 16 | 0 .. 65,535 |
| `uint32` | 32 | 0 .. 4,294,967,295 |
| `uint64` | 64 | 0 .. 1.8e19 |
| `uint`   | 32 or 64 | platform-dependent |
| `uintptr`| 32 or 64 | enough to hold a pointer |

### When to use which

- **`int`** — counts, lengths, indices. The default for "I just need an integer."
- **`int64`** — Unix timestamps in nanos, monotonic counters, anything ≥ ~2.1 billion.
- **`int32`** — wire protocols requiring exactly 32 bits; otherwise `int`.
- **`uint8` (`byte`)** — raw bytes (file contents, network buffers, hashes).
- **`uint32` / `uint64`** — bit fields, hashes, IDs that conceptually have no negative meaning, atomic counters.
- **`uintptr`** — only `unsafe` code that needs to hold a pointer-as-integer. Not garbage-collector-safe; see `13-unsafe`.

### `int` vs `int32` / `int64` in real code

```go
// Idiomatic Go:
n := len(items)              // returns int
for i := 0; i < n; i++ { ... }

// Bad:
var n int32 = int32(len(items))   // unnecessary; introduces conversions everywhere

// When int32 is required:
var crc uint32 = 0xCBF29CE484222325   // FNV-1a hash; the algorithm requires 32 bits
```

### Overflow

```go
var x int8 = 127
x++                        // -128 (silent wrap, two's complement)

var y uint8 = 255
y++                        // 0
```

Go does **not** check for integer overflow. This is a deliberate performance choice. For overflow-safe arithmetic:

```go
import "math"

if a > 0 && b > math.MaxInt-a {
    return errOverflow
}
sum := a + b
```

Or use `math/big.Int` for arbitrary precision.

Go 1.23+ ships `math/bits` helpers for overflow detection at low cost:

```go
sum, carry := bits.Add64(a, b, 0)
if carry != 0 { return errOverflow }
```

### Bit operations

```go
a & b      // AND
a | b      // OR
a ^ b      // XOR
a &^ b     // AND NOT (clear bits) — Go-unique
a << n     // shift left
a >> n     // shift right (arithmetic for signed, logical for unsigned)
^a         // bitwise NOT
```

The `&^` "AND NOT" is Go's syntactic sugar for `a & (^b)` — clear the bits in `a` that are set in `b`. Used in flag manipulation. Covered in `05-operators-and-precedence.md`.

## `byte` and `rune` — Aliases You Should Know

```go
type byte = uint8     // alias (not a distinct type)
type rune = int32     // alias

var b byte = 'A'      // OK — 'A' is 65, fits in uint8
var r rune = '世'      // OK — 'A' is int32; CJK ideographs need int32
```

`byte` is conventional for "raw byte data." `rune` is conventional for "Unicode code point." You'll see them in stdlib signatures (`[]byte` for I/O, `[]rune` for character iteration of strings). Internally, both are just `uint8` and `int32`.

A `string` ranged over with `for i, r := range s` gives `r rune`, decoded from UTF-8. A `string` indexed `s[i]` gives `byte`. This distinction matters; see `03-composite-types/04-strings-bytes-runes.md`.

## Floating-Point: `float32`, `float64`

| Type | Bits | Precision (decimal digits) |
|------|------|----------------------------|
| `float32` | 32 | ~7 |
| `float64` | 64 | ~15-17 |

Use `float64` unless you have a specific reason (memory in huge arrays, GPU buffers, audio formats). Default literal `3.14` is `float64`.

### IEEE 754 — same gotchas as everywhere

```go
0.1 + 0.2 == 0.3       // false (true literally everywhere with binary floats)
math.NaN() == math.NaN() // false (NaN != anything, including itself)
1.0 / 0.0              // +Inf, no panic
math.Inf(1) + math.Inf(-1) // NaN
```

Compare floats with tolerance:

```go
import "math"
if math.Abs(a - b) < 1e-9 { ... }      // absolute tolerance
if math.Abs(a-b)/math.Abs(b) < 1e-9 { ... }  // relative
```

### Special values

```go
math.NaN()              // not a number
math.Inf(1)             // +infinity
math.Inf(-1)            // -infinity
math.IsNaN(x)
math.IsInf(x, 0)        // either +/- inf
math.MaxFloat64
math.SmallestNonzeroFloat64
```

### Don't use floats for money

Floats *cannot* exactly represent `0.10`. Use integers (cents) or `shopspring/decimal` / `cockroachdb/apd`.

## Complex Numbers: `complex64`, `complex128`

```go
c := complex(2.0, 3.0)     // 2 + 3i
r := real(c)               // 2.0
i := imag(c)               // 3.0
c2 := 1 + 2i               // literal — i suffix
```

Most Go programs never use complex numbers. They exist for scientific computing (signal processing, FFTs) and a few specialized libraries. If you don't need them, ignore.

## `bool`

```go
var b bool = true
b = !b
b = a < c
```

Zero value: `false`. No implicit conversion to/from integers (`if 1 { ... }` is invalid). Boolean operators short-circuit (`&&`, `||`).

There is no `||=` or `&&=` (Go doesn't have compound logical assignments). Write the full form.

## `string` — Brief Mention

Covered fully in `03-composite-types/04-strings-bytes-runes.md`. Here:

- A `string` is an **immutable** sequence of bytes (typically UTF-8).
- The empty string `""` is the zero value.
- Indexing `s[i]` returns a `byte`, not a rune.
- Length `len(s)` is bytes, not runes.
- Strings are comparable (`==`, `<`, etc.).

## `uintptr` — Pointer-Sized Integer

```go
var p *int = &x
addr := uintptr(unsafe.Pointer(p))   // numeric address
```

Used only with `unsafe`. **GC ignores `uintptr`** — a `uintptr` holding an address does *not* keep the referent alive. This is why you cannot store a pointer "for later" as a `uintptr`.

For documentation: never use `uintptr` to hold an actual pointer in normal Go code. Use `*T` or `unsafe.Pointer`.

## Built-in Numeric Functions

```go
// 1.21+: generic builtins
min(1, 2, 3)         // 1
max(1.5, 2.5)        // 2.5
min[T cmp.Ordered]   // works on any ordered type, including strings

// math package — float-focused
math.Abs(x)
math.Sqrt(x)
math.Pow(x, y)
math.Floor(x)
math.Ceil(x)
math.Round(x)
math.Trunc(x)

// math/bits — integer bit twiddling
bits.LeadingZeros64(x)
bits.OnesCount32(x)        // popcount
bits.RotateLeft32(x, n)
bits.Mul64(a, b)            // 128-bit multiply
```

## Conversion Rules

```go
var i int = 100
var f float64 = float64(i)    // explicit; no implicit
var u uint8 = uint8(i)        // OK; 100 fits in uint8

i = 300
u = uint8(i)                  // 300 % 256 = 44 — silent truncation
```

Conversions never panic for numeric types except float→int with NaN/Inf:

```go
math.Floor(math.NaN())          // NaN
n := int(math.NaN())            // implementation-defined, may panic on some arches
```

Use `math.IsNaN(x) || math.IsInf(x, 0)` before converting.

## Sizes (`unsafe.Sizeof`)

```go
import "unsafe"

unsafe.Sizeof(int8(0))      // 1
unsafe.Sizeof(int32(0))     // 4
unsafe.Sizeof(int(0))       // 8 on 64-bit
unsafe.Sizeof("")           // 16 on 64-bit (pointer + length)
unsafe.Sizeof([]int{})      // 24 (pointer + length + cap)
unsafe.Sizeof(map[int]int{})// 8 (one pointer to the hmap)
unsafe.Sizeof(true)         // 1
unsafe.Sizeof(true)         // 1
unsafe.Sizeof(struct{ a bool; b int64 }{})  // 16 — alignment padding
```

Critical for understanding cache lines, struct layout, and `sync/atomic` operations. See `03-composite-types/05-structs.md` and `13-unsafe`.

## Anti-Patterns & Gotchas

**Using `int32` everywhere "because Java uses int."** In Go, `int` is the natural choice. Reserve `int32`/`int64` for fixed-width needs.

**Using `uint` to "ensure positive."** Underflow on subtraction wraps to huge positive numbers. Use `int` with validation.

**Float for currency.** Always integer cents or fixed-point decimal.

**`if 1 { ... }`.** Not valid Go. Booleans only.

**Forgetting that `byte` is `uint8`.** `byte + byte` can overflow silently.

**`var x int = math.MaxInt64`.** On 32-bit GOARCH, `MaxInt64` doesn't fit in `int`. Compile error; use `int64`.

**Comparing `NaN` directly.** Always false. Use `math.IsNaN`.

**Iterating a `string` with `for i := 0; i < len(s); i++` and using `s[i]` for multi-byte chars.** You see bytes, not runes. Use `for i, r := range s` instead.

**Mixing `int` arithmetic with `int32` constants.** Constants are untyped until assigned; usually fine, but explicit conversion makes intent clear.

**Comparing `float64` values for equality.** Use tolerance.

**Truncating `float64` to `int` without bounds checking.** A NaN-to-int conversion is undefined-behaviour-ish; on amd64 it returns `MinInt`. Validate first.

**Using `uintptr` to "save memory" instead of `*T`.** GC unsafety bug waiting to happen.

**Bit-shifting a negative `int`.** `-1 << 32` is implementation-defined; avoid.

**Forgetting that integer division truncates.** `5 / 2 == 2`, not `2.5`. Use `float64(5) / float64(2)` for the latter.

**Using `math.Floor` to round.** `math.Floor(-1.5) == -2`. Use `math.Round` for nearest-even.

## Performance Notes

- All integer arithmetic is single-instruction; sub-nanosecond.
- Float ops likewise on modern CPUs (FMA, AVX-512 on x86; NEON, SVE on arm).
- `int8`/`int16` operations are *not* faster than `int32`/`int64` — they often take the same instruction width. Use small types only for memory savings in large arrays.
- `math/bits` intrinsics map to single CPU instructions where available (POPCNT, LZCNT, TZCNT).
- Generic `min`/`max` (1.21+) compile to type-specialized inline code.

## How Big Companies Use It

- **Google's internal style guide** discourages `int32`/`int64` for general use; reserve for wire formats. Same for `uint*` — explicit signed/unsigned only when the *type* models that.
- **Uber's Go style guide** says "prefer specifying integer constants of a specific type" only at module boundaries; internal code uses `int`.
- **Kubernetes** uses `int32` extensively in API types (because the API is also Protobuf — fixed widths).
- **CockroachDB** uses `int64` for transaction IDs and timestamps.
- **Tailscale** uses `uint64` for monotonic node identifiers.
- **Cloudflare**'s low-level packet processing code uses `uint8` / `uint16` / `uint32` to match wire formats; everywhere else, `int`.

## Source Code References

- Go spec — Numeric types: https://go.dev/ref/spec#Numeric_types.
- `math` package: https://pkg.go.dev/math.
- `math/bits`: https://pkg.go.dev/math/bits.
- `math/big`: https://pkg.go.dev/math/big.
- `math/cmplx`: https://pkg.go.dev/math/cmplx.
- Conversion rules: https://go.dev/ref/spec#Conversions.

## Further Reading

- "Numeric Constants" (Rob Pike): https://go.dev/blog/constants.
- "Floating Point Arithmetic" (Goldberg): https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html.
- "Counting Bits Quickly" — uses `math/bits` examples.
- Go FAQ — "Why is `int` size implementation-dependent?": https://go.dev/doc/faq#int_size.
- "What Every Computer Scientist Should Know About Floating-Point Arithmetic".

## Exercises / Self-Check

1. Print `unsafe.Sizeof(int(0))` on both a 32-bit and 64-bit build. Note the difference.
2. Cause `int8` overflow with `x := int8(127); x++`. Confirm the value wraps to `-128`.
3. Compute `0.1 + 0.2 == 0.3`. Note `false`. Implement an `equalish(a, b, tol)` helper.
4. Convert `math.NaN()` to `int`. Observe the result (implementation-defined).
5. Use `math/bits.OnesCount32` to count bits in `0xFF00FF00`. Confirm 16.
6. Use `bits.Add64` to detect overflow when adding two large `uint64` values.
7. Range over `"héllo"` two ways: `for i, r := range s` vs `for i := 0; i < len(s); i++ { _ = s[i] }`. Note the difference in count.
8. Write a `clampInt` function using `min` and `max` (1.21+). Compare to the pre-1.21 if-else version.
