# Operators and Precedence — Including `&^` (AND NOT)

## TL;DR

Go has a deliberately small set of operators and exactly **five precedence levels** (compared to C/C++'s 15 or Python's 17). The compact precedence table is one of the language's quiet wins — almost no expression "looks wrong but parses unexpectedly." Five disciplines: (1) **no operator overloading** — `+` on `time.Time` is meaningless and won't compile; for "operator-like" semantics you use methods (`t.Add(d)`); (2) **integer division truncates toward zero**, `%` follows the sign of the dividend (`-7 % 2 == -1`); (3) **`&&` and `||` short-circuit** (left-to-right); (4) **`&^` is Go-unique** — the "AND NOT" bit-clear operator, `a &^ b` is equivalent to `a & (^b)`; (5) **comparison operators (`==`, `!=`) work on all "comparable" types** — structs are comparable if all their fields are; slices, maps, and functions are NOT comparable (compile error). Go 1.21+ adds `min`, `max`, `clear` builtins; Go has no ternary operator (`x ? a : b`) by design — use `if`. The single biggest gotcha: **integer overflow is silent and wraps**, while **`<<` shifts produce wrap-around** for signed types (`int8(1) << 7 == -128`).

## Mental Model

```
   Precedence (5 levels, highest to lowest):

   5 (highest)   *   /   %   <<   >>   &   &^
   4             +   -   |   ^
   3 (compare)   ==  !=  <   <=  >   >=
   2             &&
   1 (lowest)    ||

   Unary (binds tighter than binary): +x  -x  !x  ^x  *x  &x  <-ch
```

That's it. No assignment-as-expression. No comma operator. No ternary.

## Arithmetic Operators

```go
a + b      // sum
a - b      // difference
a * b      // product
a / b      // quotient
a % b      // remainder

+a         // unary plus (no-op, rarely written)
-a         // unary minus
```

### Integer division

```go
5 / 2      // 2 (truncate toward zero)
-5 / 2     // -2
5 / -2     // -2
-5 / -2    // 2

5 % 2      // 1
-5 % 2     // -1   (sign follows the dividend)
5 % -2     // 1
```

Integer division by zero **panics** with `runtime error: integer divide by zero`. Always validate before dividing on user input.

### Float division

```go
5.0 / 2.0      // 2.5
5.0 / 0        // +Inf
0.0 / 0.0      // NaN
math.Mod(5, 2) // 1.0 (IEEE 754 remainder, sign follows dividend)
```

Float division by zero produces `+Inf` / `-Inf` / `NaN`, never a panic. See `02-primitive-types.md`.

### Overflow

```go
var x int8 = 127
x + 1            // -128 — silent wrap

var u uint8 = 255
u + 1            // 0

math.MaxInt + 1  // wraps to math.MinInt
```

Go does not detect overflow. For overflow-aware code:

```go
import "math/bits"
sum, carry := bits.Add64(a, b, 0)
```

## Bitwise Operators

```go
a & b       // AND
a | b       // OR
a ^ b       // XOR (also "exclusive or")
a &^ b      // AND NOT (clear bits in a that are set in b)
a << n      // left shift (multiply by 2^n)
a >> n      // right shift (divide by 2^n)
^a          // bitwise NOT (unary)
```

### `&^` — the bit-clear operator

`a &^ b` is **equivalent to** `a & (^b)`. The compiler may emit a single instruction (`BIC` on ARM, `ANDN` on x86) where supported.

Used to clear flags:

```go
const (
    FlagRead    = 1 << 0
    FlagWrite   = 1 << 1
    FlagExecute = 1 << 2
)

perms := FlagRead | FlagWrite | FlagExecute   // 0b111
perms = perms &^ FlagWrite                     // 0b101 — Write cleared
```

Without `&^`, you'd write `perms = perms & (^FlagWrite)` or `perms = perms & ^FlagWrite` — the same but two operators.

### Shifts

```go
1 << 32              // = 4294967296 (untyped constant — fits in int64)
var x int32 = 1 << 32  // compile error: shift count too large

uint(1) << 64        // undefined (shift ≥ bit width) — compile error if constant
```

Right shifts:
- For **signed** types: arithmetic shift (sign bit propagates). `int8(-1) >> 1 == -1`.
- For **unsigned** types: logical shift (zero fill). `uint8(255) >> 1 == 127`.

### Bit-twiddling shortcuts (`math/bits`)

```go
import "math/bits"

bits.LeadingZeros64(x)     // # leading zero bits
bits.TrailingZeros32(x)
bits.OnesCount(x)          // population count (popcount)
bits.RotateLeft32(x, n)
bits.Reverse64(x)
```

Most of these compile to single CPU instructions where available (LZCNT, TZCNT, POPCNT, BSWAP, etc.).

## Comparison Operators

```go
a == b      // equal
a != b      // not equal
a <  b      // less than
a <= b      // less than or equal
a >  b      // greater than
a >= b      // greater than or equal
```

### Comparable types

- **All numeric types**: comparable.
- **Booleans**: comparable.
- **Strings**: comparable (lexicographic).
- **Pointers**: comparable (address equality).
- **Channels**: comparable (same channel).
- **Interfaces**: comparable (panics if the dynamic type is non-comparable).
- **Arrays**: comparable IF the element type is comparable.
- **Structs**: comparable IF all fields are comparable.
- **Slices, maps, functions**: **NOT comparable** (compile error except against `nil`).

```go
var s []int
s == nil     // OK — slice can be compared to nil
s1 == s2     // compile error
```

To compare slices: `slices.Equal(s1, s2)` (Go 1.21+) or write a loop.

### Float comparison

```go
math.NaN() == math.NaN()     // false (always)
0.0 == -0.0                  // true
1.0 / 0.0 == 2.0 / 0.0       // true (both +Inf)
```

Don't compare floats for equality unless you really mean bit-exact (e.g., constants known to be representable).

### Interface comparison panic

```go
var a, b any = []int{1}, []int{1}
_ = a == b    // PANIC at runtime: comparing uncomparable type []int
```

If two interfaces hold slice/map/func values, `==` compiles but panics. Safer:

```go
import "reflect"
reflect.DeepEqual(a, b)
```

Or, ideally, don't put non-comparable types behind an interface you'll compare.

## Logical Operators

```go
a && b      // AND, short-circuits (b not evaluated if a is false)
a || b      // OR, short-circuits (b not evaluated if a is true)
!a          // NOT
```

Short-circuit semantics matter for guards:

```go
if p != nil && p.Field == 42 { ... }   // safe — p.Field not accessed if p is nil
```

There are **no `&&=` or `||=`** compound assignments. Write the full form.

There's also **no logical operator on numerics**: `5 && 3` is a compile error. C/JS habits don't transfer.

## Assignment Operators

```go
x = y         // simple assignment
x += y        // x = x + y
x -= y
x *= y
x /= y
x %= y
x &= y
x |= y
x ^= y
x <<= y
x >>= y
x &^= y       // bit-clear assignment

x++           // STATEMENT, not expression — same as x += 1
x--           // STATEMENT, same as x -= 1
```

Two things to remember:

1. **`x++` is a statement, not an expression.** You can't `y := x++` or `if x++ > 5`. Go made this choice to remove pre/post increment ambiguity.
2. **`=` and `:=` are different.** `=` reassigns; `:=` declares at least one new variable.

## Channel Operators

```go
ch <- v       // send
v := <-ch     // receive
v, ok := <-ch // receive with closed-detection
```

`<-ch` is a unary expression in receive form; `ch <-` is a statement (no value).

## Address-of and Dereference

```go
p := &x       // & = address-of
*p = 10       // * = dereference (write through pointer)
v := *p       // dereference (read)
```

Combined with `new`/`&T{}`:

```go
p := new(int)       // *int, points to a zero int
q := &Point{1, 2}   // *Point — common
```

## `++` and `--` — Statements

```go
x++           // OK
x--           // OK

y := x++      // COMPILE ERROR — ++ is not an expression
if x++ > 5 {} // COMPILE ERROR

++x           // COMPILE ERROR — no pre-increment
```

Move to a separate statement:

```go
x++
y := x
if y > 5 { ... }
```

## The Precedence Table — Full

| Precedence | Operators |
|------------|-----------|
| 5 (highest) | `*  /  %  <<  >>  &  &^` |
| 4 | `+  -  |  ^` |
| 3 | `==  !=  <  <=  >  >=` |
| 2 | `&&` |
| 1 (lowest) | `\|\|` |

Unary operators (`!`, `^`, `-`, `+`, `*`, `&`, `<-`) bind tighter than any binary operator.

Examples:

```go
a + b * c                   // a + (b*c)         — multiplication first
a == b && c > d             // (a==b) && (c>d)   — compare, then logical
a | b == c                  // a | (b==c)        — surprise! compare is lower than OR
a & b | c                   // (a&b) | c         — AND tighter than OR
a << 1 + b                  // (a<<1) + b        — shift tighter than +
```

Two cases where parentheses are *strongly* recommended:

- `a | b == c` parses as `a | (b == c)`. Usually NOT what you want. Write `(a | b) == c`.
- `a + b << c` parses as `a + (b << c)`. Usually NOT what you want. Write `(a + b) << c`.

When in doubt, parenthesize.

## No Ternary

```go
// Other languages:
//   y = x > 0 ? x : -x;

// Go:
y := x
if x < 0 { y = -x }

// Or for trivial cases — a helper:
func ifThenElse[T any](cond bool, a, b T) T {
    if cond { return a }
    return b
}
```

Go intentionally omitted the ternary operator to push toward `if` (or a small helper, with generics 1.18+). The argument: ternaries nested become unreadable; `if` doesn't.

## `min`, `max`, `clear` (1.21+ Builtins)

```go
min(1, 2, 3)         // 1 — works on any cmp.Ordered (int, float, string, ...)
max("a", "b", "c")   // "c"

clear(m)             // empties a map; preserves capacity
clear(s)             // zeroes a slice in place; preserves length
```

These are not operators but builtins. The compiler may emit type-specialised code for common cases.

## Anti-Patterns & Gotchas

**Mixing types in arithmetic.** `var i int = 1; var i32 int32 = 1; i + i32` → compile error. Convert.

**Forgetting `&^` exists.** Writing `a & (^b)` works but is two operators where one would do.

**Comparing slices with `==`.** Compile error (except vs nil). Use `slices.Equal`.

**Comparing interface-wrapped slices with `==`.** Panics. Use `reflect.DeepEqual` or type-assert first.

**Float equality.** `0.1 + 0.2 == 0.3` is false. Use tolerance.

**Integer division surprise.** `5/2 == 2`. If you want 2.5, convert one operand to float.

**Modulo with negative operands.** `-7 % 3 == -1`, not `2`. For "Python-style" non-negative modulo: `((x % m) + m) % m`.

**`<<` overflow.** Shifting past the bit width is allowed at runtime (yields 0) but constant shifts past width are compile errors.

**Using `++` as an expression.** It's not. Statement only.

**Confusing `&` (AND) with `&&` (logical).** `5 & 3 == 1`, `true & false` is a compile error.

**Forgetting `&^` precedence is the same as `&`.** `a + b &^ c` is `a + (b &^ c)`. Parenthesize if uncertain.

**Comparing `time.Time` with `==`.** It works (monotonic clock + wall clock — two fields), but two times that "should be equal" may not be. Use `t1.Equal(t2)` for wall-time-only comparison.

**Treating `nil == nil` for interfaces.** Two nil interfaces compare equal; a nil-pointer-wrapped-in-interface ≠ nil.

**Shift by non-constant that may exceed width.** `x >> n` where `n` could be 64 on a 64-bit int produces 0; sometimes a real bug.

**`a & b == 0` to test a flag — parses correctly here (`&` higher than `==`), but `a | b == c` doesn't.** Be consistent: parenthesize the bit expression.

## Performance Notes

- **All operators**: 1-cycle CPU ops (integer / float / bit).
- **`%` for integer**: typically a single DIV instruction; slower than `&` for power-of-two divisors. `x % 8` vs `x & 7` — the compiler often does this rewrite if the divisor is a constant power of two.
- **`/` by constant power-of-two on signed types**: slightly more expensive than unsigned, because of rounding-toward-zero semantics. The compiler emits a small sequence.
- **Comparison of large structs**: O(size); fields compared in order with short-circuit on mismatch.
- **String comparison**: O(min(len1, len2)) for `==`; O(len) for `<` lexicographic.
- **`reflect.DeepEqual`**: much slower; allocates. Avoid in hot loops.

## How Big Companies Use It

- **Google's internal style guide** discourages bit manipulation outside hardware/protocol code.
- **Cloudflare**'s wire protocol code uses `&^` extensively for flag masking.
- **HashiCorp** code uses bit-flag enums via `iota` and clears with `&^`.
- **Kubernetes** uses `|` and `&` for `metav1.Status` and condition combinations.
- **CockroachDB** uses `math/bits` intrinsics for high-performance roaring-bitmap and hash code.
- **Tailscale**'s wire format parsers rely on shifts and masks for compact encoding.

## Source Code References

- Go spec — Operators: https://go.dev/ref/spec#Operators.
- Go spec — Comparison operators: https://go.dev/ref/spec#Comparison_operators.
- Go spec — IncDec statements: https://go.dev/ref/spec#IncDec_statements.
- `math/bits` (intrinsics): https://github.com/golang/go/tree/master/src/math/bits.
- Compiler bit-fiddling (rewrite): https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/rewrite.go.

## Further Reading

- Effective Go — control structures: https://go.dev/doc/effective_go.
- Rob Pike on operator overloading and ternary: various FAQ and talks.
- "Bit Twiddling Hacks" (Sean Eron Anderson): http://graphics.stanford.edu/~seander/bithacks.html.
- "Hacker's Delight" (Henry S. Warren) — definitive bit manipulation reference.
- Go FAQ — "Why doesn't Go have ternary?": https://go.dev/doc/faq#Does_Go_have_a_ternary_form.

## Exercises / Self-Check

1. Compute `-7 % 2` in Go and Python. Note the sign difference.
2. Set `var x int8 = 127`. Compute `x++`. Confirm the wrap.
3. Use `&^` to clear a flag in a permission bitmask. Compare to the `a & (^b)` form.
4. Try `var a, b any = []int{1}, []int{1}; _ = a == b`. Observe the runtime panic. Replace with `reflect.DeepEqual` or a slices.Equal-typed comparison.
5. Parse `a | b == c` mentally. Confirm precedence by experimenting. Parenthesize for clarity.
6. Write a benchmark for `x % 8` vs `x & 7`. Confirm equivalent generated code (via `go tool compile -S`).
7. Use `min` / `max` builtins (1.21+) on integers, floats, and strings. Confirm they accept any ordered type.
8. Try `var ch chan int; ch <- 1`. Observe the deadlock (nil channel send blocks forever).
