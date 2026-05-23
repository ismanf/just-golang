# `math`, `math/rand`/`math/rand/v2`, `math/big` — Numeric Primitives

## TL;DR

`math` is floating-point math (`Sqrt`, `Sin`, `Log`, `IsNaN`, `Inf`). `math/rand/v2` (since 1.22) is the modern PRNG — non-deterministic by default, fast PCG generator, no global mutex. Use `crypto/rand` for anything security-sensitive. `math/big` provides arbitrary-precision `Int`, `Rat`, `Float` for crypto, financial calculations, and anything that overflows `int64`/`float64`.

## Mental Model

```
math:        f64-precision functions and constants. Hardware-backed.
math/rand/v2: deterministic PCG generator, per-Source, no global lock.
math/rand:   legacy (1.x); slower, globally locked. Migrate to /v2.
crypto/rand: CSPRNG from OS entropy. Use for keys, tokens, nonces.
math/big:    Int, Rat, Float types with arbitrary precision.
```

## Syntax & Basic Usage

```go
package main

import (
	"fmt"
	"math"
	"math/big"
	"math/rand/v2"
)

func main() {
	fmt.Println(math.Pi)
	fmt.Println(math.Sqrt(2))

	fmt.Println(rand.IntN(100))           // 0..99
	fmt.Println(rand.Float64())            // 0.0..<1.0

	x := big.NewInt(1)
	for i := int64(1); i <= 20; i++ {
		x.Mul(x, big.NewInt(i))
	}
	fmt.Println("20! =", x.String())
	// Output (varies for rand):
	// 3.141592653589793
	// 1.4142135623730951
	// 42
	// 0.7137...
	// 20! = 2432902008176640000
}
```

## Deep Dive

### `math`

Constants:

- `math.Pi`, `math.E`, `math.Sqrt2`, `math.MaxFloat64`, `math.MaxInt64`, `math.MinInt64`, etc.

Useful functions:

- `Sqrt`, `Cbrt`, `Pow`, `Exp`, `Log`, `Log2`, `Log10`.
- `Sin`, `Cos`, `Tan` and inverses; hyperbolic versions.
- `Abs`, `Ceil`, `Floor`, `Round`, `Trunc`, `Mod`.
- `Min`, `Max` — for float64 (Go 1.21 also added builtins `min`/`max` that work on any ordered type).
- `IsNaN`, `IsInf`, `NaN`, `Inf(sign)`.
- `Nextafter`, `Float64bits`/`Float64frombits` for IEEE bit-twiddling.

`math.NaN() != math.NaN()` — always. Use `math.IsNaN` to test.

### `math/rand/v2` (since 1.22)

```go
import "math/rand/v2"

rand.IntN(10)      // 0..9
rand.Int64N(100)   // typed variants
rand.Float64()
rand.Shuffle(len(s), func(i, j int) { s[i], s[j] = s[j], s[i] })

// Seeded:
r := rand.New(rand.NewPCG(42, 1024))
r.IntN(10)
```

Properties:

- Non-deterministic seed by default (process-unique).
- No global mutex; functions on top-level call a per-goroutine source.
- PCG generator: high-quality, fast, small state.
- Generic helpers like `rand.IntN[T Integer]`.

### `math/rand` (legacy, since 1.0)

Pre-1.22 the only option. Globally seeded (1.20+: auto-seeded, before that you had to call `Seed` yourself). Slower due to global mutex. **Use `/v2` for new code.**

### `crypto/rand` — for security

```go
import "crypto/rand"

buf := make([]byte, 32)
_, err := rand.Read(buf) // pulls from /dev/urandom or getrandom(2)
```

For random integers in a range:

```go
import "crypto/rand"
import "math/big"

n, err := rand.Int(rand.Reader, big.NewInt(1000)) // uniform 0..999
```

**Never** use `math/rand` for tokens, session IDs, passwords, salt, nonces. Use `crypto/rand`.

### `math/big`

- `big.Int` — arbitrary-precision integer.
- `big.Rat` — arbitrary-precision rational (numerator/denominator).
- `big.Float` — arbitrary-precision float.

Mutating API: methods *modify the receiver* and return it for chaining:

```go
z := new(big.Int)
z.Add(a, b) // z = a + b
z.Mul(z, c) // z = z * c
```

This shocks newcomers. The reason: avoid allocating a new `big.Int` per operation.

Conversions:

```go
n := big.NewInt(0)
n.SetString("123456789012345678901234567890", 10)
n.String()
n.Text(16)
n.Bytes()
n.Sign()  // -1, 0, +1
```

### Floating-point pitfalls

- `0.1 + 0.2 == 0.3` is false. Use `math.Abs(a-b) < epsilon` for "close enough".
- For money, do NOT use `float64`. Use `int64` cents, or `math/big.Rat`, or a third-party decimal lib like `shopspring/decimal`.
- `math.Inf(1) - math.Inf(1)` is NaN.

## Standard Library Hooks

- `math/cmplx` for complex math.
- `math/bits` for bit-twiddling (`OnesCount`, `TrailingZeros`, `LeadingZeros`).
- `sort.Float64s`, `slices.Sort` for sorting.
- `encoding/json` — `math.NaN()` and `math.Inf()` cannot be JSON-encoded by default.

## Real-World Patterns

### 1. Cryptographically random hex token

```go
import (
	"crypto/rand"
	"encoding/hex"
)

func token(n int) (string, error) {
	b := make([]byte, n)
	if _, err := rand.Read(b); err != nil { return "", err }
	return hex.EncodeToString(b), nil
}
```

Use case: session IDs, API keys.

### 2. Weighted random choice with `math/rand/v2`

```go
type Weighted[T any] struct {
	items []T
	weights []float64
	total   float64
}

func (w *Weighted[T]) Pick() T {
	r := rand.Float64() * w.total
	acc := 0.0
	for i, weight := range w.weights {
		acc += weight
		if r < acc { return w.items[i] }
	}
	return w.items[len(w.items)-1]
}
```

Use case: A/B test variant assignment, load-balanced backend picking.

### 3. Big-integer factorial

```go
func factorial(n int64) *big.Int {
	result := big.NewInt(1)
	for i := int64(2); i <= n; i++ {
		result.Mul(result, big.NewInt(i))
	}
	return result
}
```

Use case: combinatorics, cryptographic key generation primes.

### 4. Decimal money via int64

```go
type Cents int64

func (c Cents) Format() string {
	sign := ""
	v := int64(c)
	if v < 0 { sign = "-"; v = -v }
	return fmt.Sprintf("%s$%d.%02d", sign, v/100, v%100)
}
```

Use case: any financial system. Floats are forbidden by every accounting standard.

### 5. Histogram bucket with `math.Log2`

```go
func bucket(v float64) int {
	if v <= 1 { return 0 }
	return int(math.Floor(math.Log2(v)))
}
```

Use case: latency histograms, log-scale aggregations.

## Anti-Patterns & Gotchas

**`math/rand` (no `/v2`) for new code.** Use `/v2`.

**`math/rand` for security.** `crypto/rand` mandatory.

**Floats for money.** Always wrong.

**Comparing floats with `==`.** Use epsilon.

**`big.Int` value receivers.** Methods take pointers; pass `*big.Int`.

**Forgetting `big.Int` methods modify the receiver.** `z.Add(a, b)` writes to `z`. Sometimes you want `new(big.Int).Add(a, b)`.

**`rand.Seed(time.Now().UnixNano())`** — pre-1.20 ritual. 1.20+ auto-seeds. 1.22 `/v2` ignores it.

**Using `math.Min`/`Max` for ints.** Use the builtins `min`/`max` (1.21+) which work on ordered types.

**Comparing `*big.Int` with `==`.** That compares pointers. Use `a.Cmp(b)`.

## Performance Notes

- `math/rand/v2` is faster than `math/rand` (no mutex; PCG is cheap).
- `math/big` is order-of-magnitude slower than `int64`. Use only when range demands it.
- `crypto/rand.Read` may block on entropy starvation at boot; rare on modern systems.
- `math.Sqrt`, `math.Sin` etc. compile to hardware instructions on x86/ARM where possible.
- `math/bits.OnesCount64` becomes `POPCNT` on x86 with the right GOAMD64.

## How Big Companies Use It

- **Cockroach** uses `math/rand` (pre-`/v2`) seeded per-transaction for chaos testing.
- **Ethereum clients (geth)** use `math/big` extensively for 256-bit arithmetic.
- **HashiCorp Vault** uses `crypto/rand` for token generation; `math/big` for RSA/DH primes.
- **Discord** uses `math/rand/v2`'s shuffle for random feature flag bucketing.

## Source Code References

Pinned to `go1.26`.

- `math`: [`src/math/`](https://github.com/golang/go/tree/master/src/math).
- `math/rand/v2`: [`src/math/rand/v2/`](https://github.com/golang/go/tree/master/src/math/rand/v2).
- `math/big`: [`src/math/big/`](https://github.com/golang/go/tree/master/src/math/big) — `int.go`, `rat.go`, `float.go`.
- `crypto/rand`: [`src/crypto/rand/rand.go`](https://github.com/golang/go/blob/master/src/crypto/rand/rand.go).

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/math, /math/rand/v2, /math/big, /crypto/rand.
- PCG random paper: https://www.pcg-random.org/.
- "What Every Computer Scientist Should Know About Floating-Point Arithmetic" (Goldberg).

## Exercises / Self-Check

1. Implement `RandString(n)` returning a base64 token from `crypto/rand`.
2. Show that `0.1 + 0.2 != 0.3` in `float64`. Now write a near-equal comparison.
3. Compute the 1000th Fibonacci number using `math/big`.
4. Build a weighted picker and test that proportions match weights over 1M draws.
5. Why is `math/rand/v2` faster than `math/rand` even for a single goroutine? Hint: lock elision.
