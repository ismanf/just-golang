# `iota` Deep Dive — Bit Flags, Skip Values, Typed Enums

## TL;DR

`iota` is Go's compile-time constant generator. It's a **single keyword** that produces the sequence `0, 1, 2, 3, ...` within a `const ( ... )` block, with one rule that surprises most newcomers: **the right-hand-side expression is repeated for each subsequent line** that omits it, with `iota` incremented per line. This makes `iota` powerful for enum-like declarations, bit flags, and arithmetic-progression constants — three patterns cover ~95% of real uses. The mental model: think of `iota` as "the line index inside the const block, starting at 0," and the expression on each line is reused unless overridden. Four idioms you'll see in every Go codebase: (1) **plain enum** — `iota` for sequential integer constants; (2) **typed enum with String() method** — `type Status int` with `iota`-numbered constants and a `stringer`-generated method; (3) **bit flags** — `1 << iota` for OR-combinable options; (4) **skip-and-resume** — using `_` to consume an iota value without binding it. The single biggest gotcha: **`iota` increments per LINE inside the const block, not per assignment** — `Sunday, Monday = iota, iota` declares both at 0, which is rarely what you want.

## Mental Model

```
   const (
       A = iota    // iota = 0, A = 0
       B           // iota = 1, B = 1 (expression "iota" repeated)
       C           // iota = 2, C = 2
   )

   Three rules:
   1. iota starts at 0 in each const ( ... ) block.
   2. iota increments by 1 per LINE inside the block.
   3. If a line omits its RHS, the previous line's RHS is reused
      (with iota's new value).
```

## The Three Rules in Action

```go
const (
    a = iota          // 0
    b                  // 1 (RHS reused as "iota")
    c                  // 2
    d = 100            // 100 — explicit, breaks the chain
    e                  // 100 — RHS reused as "100", iota still increments to 4
    f = iota           // 5 — iota was 5 even though we didn't use it on d, e
    g                  // 6
)
```

`iota` continues to count regardless of whether each line uses it. Skipping the value (or using a literal) doesn't reset.

## Pattern 1: Plain Enum

```go
const (
    Sunday = iota   // 0
    Monday          // 1
    Tuesday         // 2
    Wednesday       // 3
    Thursday        // 4
    Friday          // 5
    Saturday        // 6
)
```

Simple, zero-cost, untyped constants. Use when you don't need a distinct type — just named integers.

## Pattern 2: Typed Enum

```go
type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)

func (d Weekday) String() string {
    return [...]string{"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"}[d]
}
```

Type `Weekday` makes the compiler reject `var d Weekday = 5` (must be `Sunday`...`Saturday`). The `String()` method makes `fmt.Println(d)` print "Sat" instead of `6`.

Many teams use **`stringer`** to generate `String()`:

```go
//go:generate stringer -type=Weekday
```

Run `go generate ./...` and a `weekday_string.go` file appears with `String()` implemented. Install: `go install golang.org/x/tools/cmd/stringer@latest`.

### Convention: `_UNSPECIFIED = 0`

For protobuf and gRPC interop:

```go
const (
    StatusUnspecified Status = iota   // 0 — required default
    StatusPending                     // 1
    StatusActive                      // 2
    StatusClosed                      // 3
)
```

This avoids the "missing field reads as the first enum value" problem in protobuf and similar formats. Proto3 requires zero-valued enum members.

## Pattern 3: Bit Flags

```go
type Permission uint32

const (
    PermRead    Permission = 1 << iota   // 1
    PermWrite                            // 2
    PermExecute                          // 4
    PermDelete                           // 8
    PermAdmin                            // 16
)
```

`1 << iota` shifts a 1-bit; each value is a distinct bit. Combine with `|`:

```go
perms := PermRead | PermWrite          // 3
hasWrite := perms & PermWrite != 0     // true
perms = perms &^ PermWrite             // 1 — clear write
```

See `05-operators-and-precedence.md` for the `&^` (AND NOT) operator.

### Combined flag constants

```go
const (
    PermRead Permission = 1 << iota
    PermWrite
    PermExecute
    PermDelete

    PermReadWrite = PermRead | PermWrite       // 3
    PermFull      = PermRead | PermWrite | PermExecute | PermDelete   // 15
)
```

You can mix `iota`-generated flags with composed constants in the same block. The composed ones don't use `iota` themselves but `iota` still increments per line.

## Pattern 4: Skip Values

```go
const (
    _ = iota          // 0 — discarded
    KB = 1 << (10 * iota)  // 1 << 10 = 1024
    MB                      // 1 << 20 = 1048576
    GB                      // 1 << 30
    TB                      // 1 << 40
    PB                      // 1 << 50
    EB                      // 1 << 60
)
```

The first line uses `_` to "burn" iota = 0 (which would make `KB = 1 << 0 = 1`, wrong). Then each subsequent line gets `iota = 1, 2, 3, ...`. The expression `1 << (10 * iota)` is repeated implicitly.

This is the canonical "kilobytes-style power-of-1024" definition. The same pattern works for any progression.

## Pattern 5: Reset Per Block

```go
const (
    a = iota   // 0
    b          // 1
)

const (
    c = iota   // 0 — reset
    d          // 1
)
```

Each `const ( ... )` block has its own iota counter. Use grouped blocks deliberately.

## Pattern 6: Multi-Value Per Line

```go
const (
    A, B = iota, iota * 2    // A=0, B=0
    C, D                      // C=1, D=2  (iota=1; expression is "iota, iota * 2")
    E, F                      // E=2, F=4
)
```

`iota` is the same value across both elements on a line. The expression is repeated with the new iota value.

Useful when paired constants need related values.

## Pattern 7: Empty Implicit Lines

```go
const (
    a = iota    // 0
                // iota = 1, but no declaration on this line — but THIS LINE IS A COMMENT, not a const declaration!
    b           // iota = 1, b = 1
)
```

Comments and blank lines **don't count** as const declarations and don't increment iota. iota only increments on declaration lines.

```go
const (
    a = iota    // 0
    // comment — does not affect iota
    b           // 1

    c           // 2
)
```

## When `iota` Is the Wrong Tool

```go
// BAD — fragile; adding/removing a value renumbers everything downstream
const (
    StateA = iota
    StateB
    StateC
    StateD
)

// In a database that already stores StateC = 2 — adding StateAB between
// A and B silently shifts every downstream value, including persisted ones.
```

For values that get **persisted, logged, or sent over the wire**, prefer explicit values:

```go
const (
    StateA State = 1
    StateB State = 2
    StateC State = 3
    StateD State = 4
    // adding StateE: explicit, no risk of renumbering
    StateE State = 10
)
```

Rule of thumb: `iota` for in-memory enums; explicit values for serialised state.

## `stringer` and Code Generation

```go
//go:generate stringer -type=Status -trimprefix=Status

type Status int

const (
    StatusUnspecified Status = iota
    StatusPending
    StatusActive
    StatusClosed
)
```

Generated `status_string.go`:

```go
func (s Status) String() string {
    switch s {
    case StatusUnspecified: return "Unspecified"
    case StatusPending:     return "Pending"
    case StatusActive:      return "Active"
    case StatusClosed:      return "Closed"
    }
    return fmt.Sprintf("Status(%d)", s)
}
```

The `-trimprefix=Status` flag strips the type prefix from each string. The generated code uses a jump table for performance.

Also useful: `enumer` (https://github.com/dmarkham/enumer) — extends `stringer` with `MarshalJSON`, `UnmarshalJSON`, `Values()`, `Validate()`.

## `iota` in Type-Switch Constants

```go
type Op int

const (
    OpAdd Op = iota
    OpSub
    OpMul
    OpDiv
)

func (op Op) Apply(a, b int) int {
    switch op {
    case OpAdd: return a + b
    case OpSub: return a - b
    case OpMul: return a * b
    case OpDiv: return a / b
    }
    panic("unknown op")
}
```

Typed iota enums + switch is a complete sum type. Cover all cases or use `default`.

The **`exhaustive`** linter (https://github.com/nishanths/exhaustive) checks that every iota-typed enum has all values covered in its switch statements.

## `iota` with Complex Expressions

```go
type SignalQuality int

const (
    BarsNone SignalQuality = iota   // 0
    BarsLow                         // 1
    BarsMedium                      // 2
    BarsHigh                        // 3
    BarsExcellent                   // 4
)

// Constants derived from iota
const (
    SignalDBmCutoff = -90 + iota*10   // -90, -80, -70, -60, ...
    SignalCutoffMedium
    SignalCutoffHigh
    SignalCutoffExcellent
)
```

Any compile-time expression involving `iota` works.

```go
const (
    KiB = 1 << ((iota + 1) * 10)   // 1024
    MiB                              // 1048576
    GiB                              // ...
)
```

## Anti-Patterns & Gotchas

**Renumbering persisted iota enums.** Adding a value in the middle shifts everything downstream. For wire/storage formats, use explicit values.

**`Sunday, Monday = iota, iota`.** Declares both at the same iota value — usually a bug. Use separate lines.

**`iota` outside a `const` block.** Compile error — `iota` only works inside `const ( ... )`.

**Forgetting `_UNSPECIFIED = 0` for protobuf-bound enums.** Will surprise you later.

**Bit flags without `1 <<`.** `const ( A = iota; B; C )` gives 0, 1, 2 — `A | B` is `0 | 1 = 1`, but `A | C` is `0 | 2 = 2`. Doesn't work as flags. Use `1 << iota`.

**Mixing constant types unintentionally.**
```go
const (
    a int = iota       // a is int
    b                  // b is also int (type inherited)
    c = iota           // c becomes untyped! type is NOT carried across "explicit RHS" lines
    d                  // d is untyped, same expression
)
```
When a line has its own RHS, the type isn't carried — it's untyped unless explicit.

**Iota in maps**:
```go
var weekdayNames = map[Weekday]string{
    Sunday:    "Sun",
    Monday:    "Mon",
    // ...
}
```
Fine — but adding `Sunday Weekday = iota` after the fact and forgetting to update the map silently produces missing entries. Generate or test.

**Long iota chains.** Past ~10 values, switch to explicit. The chain becomes hard to reason about.

**`const ( a, b = iota, iota + 1 )` and expecting `a = 0, b = 1`.** Works for line 1, but the second line repeats the expression — `c, d = iota, iota + 1` gives `c = 1, d = 2`. The "iota + 1" is part of the repeated expression.

**Naming convention: lowercase first member.** Don't shadow the type's zero value with a private member; downstream code can't access it via the constant name.

**Adding a new enum value at the end and forgetting to update `String()`, validators, etc.** Use `stringer` + `enumer` + `exhaustive` linter as a safety net.

## Performance Notes

- Constants are **resolved at compile time** — zero runtime cost.
- `iota` expressions are folded into literals; the generated assembly contains the literal value.
- `stringer`-generated `String()` is jump-tabled (constant time).
- Bit-flag operations are single CPU instructions.

## How Big Companies Use It

- **Google's internal style guide** uses `iota` for in-memory enums; reserves explicit values for cross-process / persisted state.
- **Kubernetes** uses `iota` for `metav1` condition statuses, event reasons, resource phases.
- **etcd** uses `iota` for leader election states and Raft message types.
- **CockroachDB** uses `iota` for SQL expression operation codes.
- **Cloudflare** uses bit-flag `iota` patterns for packet flags.
- **HashiCorp Vault** uses `iota` for capability bitmasks and lease states.
- **Docker / Moby** uses `iota` for container state machines.

`stringer` is in widespread use; you'll see `//go:generate stringer -type=X` in nearly every mature Go codebase.

## Source Code References

- Go spec — `iota`: https://go.dev/ref/spec#Iota.
- Go spec — Constant expressions: https://go.dev/ref/spec#Constant_expressions.
- `stringer`: https://pkg.go.dev/golang.org/x/tools/cmd/stringer.
- `enumer`: https://github.com/dmarkham/enumer.
- `exhaustive` linter: https://github.com/nishanths/exhaustive.
- `gen-string` and similar tools: various.

## Further Reading

- "Constants in Go" (Rob Pike): https://go.dev/blog/constants.
- "Enumerations in Go" — various community blog posts.
- "Effective Go" — Constants: https://go.dev/doc/effective_go#constants.
- "Implementing enums in Go" (Dave Cheney): https://dave.cheney.net/.
- "Go: Anatomy of an enum" — Medium posts of various detail.

## Exercises / Self-Check

1. Define `type Weekday int` with iota-numbered values. Add a `String()` method. Verify `fmt.Println(Tuesday)` prints "Tue".
2. Define bit flags `PermRead = 1 << iota`, `PermWrite`, `PermExecute`. Combine; clear individual flags with `&^`.
3. Define KB/MB/GB/TB constants using `_ = iota`. Verify `KB * 1024 == MB`.
4. Run `//go:generate stringer -type=Status` on a Status enum. Inspect the generated file.
5. Add a new enum value in the middle of a `const ( iota ... )` block. Observe that downstream values shift. Migrate to explicit values for persistence safety.
6. Use `exhaustive` linter on a `switch` over an iota enum. Omit a case; observe the warning.
7. Try `Sunday, Monday = iota, iota`. Confirm both are 0; switch to separate lines.
8. Write a `Format(state State) string` using an explicit `switch state {}`; compare line count and clarity to a generated `stringer` version.
