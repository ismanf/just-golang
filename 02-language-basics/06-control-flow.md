# Control Flow — `if`, `for` (All Forms), `switch`, `goto`, Labels

## TL;DR

Go's control flow is **deliberately minimalist**: one loop keyword (`for`), no `while`/`do-while`, no `try`/`catch`/`finally`, no ternary, and `switch` cases that don't fall through by default. Eight constructs cover everything: `if`/`else`, `for` (in four forms — C-style, while-style, infinite, range), `switch` (expression and type), `goto`, labels, `break`, `continue`. The mental model: **Go has no loop fallthrough but has explicit cases**; **`if` and `for` accept an init statement** (`if v, err := f(); err != nil`); **`switch` defaults to "true"** so `switch { case x > 0: ...; case x < 0: ...; }` is the idiomatic "long if-else." Go 1.22 made the most important loop change in the language's history: **each iteration gets its own loop variable** (covered in `09-closures.md`). Go 1.23+ added **range over function** (`for x := range myIter`) — covered in `24-frontier/04-range-over-func-advanced.md`. The single biggest gotcha for newcomers: **forgetting that `if` and `for` don't take parentheses around the condition but DO require braces around the body** — `if x > 0 { ... }` not `if (x > 0) ...`.

## Mental Model

```
   if              if cond { ... } else { ... }
                   if init; cond { ... }

   for             for init; cond; post { ... }    // C-style
                   for cond { ... }                  // while
                   for { ... }                       // infinite
                   for i, v := range x { ... }       // range
                   for i := range n { ... }          // 1.22: range over int
                   for v := range myIter { ... }     // 1.23: range over func

   switch          switch [init;] [tag] { case ... }
                   switch v := x.(type) { case T: ... }   // type switch

   break / continue [optional label]
   goto label
   return / defer / panic   (see 02/07 and 05)
```

## `if`

```go
if x > 0 {
    fmt.Println("positive")
} else if x < 0 {
    fmt.Println("negative")
} else {
    fmt.Println("zero")
}
```

### Init statement

```go
if err := doSomething(); err != nil {
    return err
}

if v, ok := m["key"]; ok {
    use(v)
}
```

`init; cond`. The init runs once; its variables are scoped to the entire `if`/`else` chain.

```go
if v, ok := m["key"]; ok {
    // v in scope
} else {
    // v ALSO in scope here (== zero value)
}
// v NOT in scope here
```

This scope rule is why the `if`-init pattern is so prevalent — it limits `err`/`ok`/etc. to the block that uses them.

### No parentheses; braces required

```go
if (x > 0) { ... }   // legal but un-Go-like; gofmt keeps the parens
if x > 0  ...        // ILLEGAL — braces required
```

Single-line if is not allowed. Always braces.

## `for` — One Loop, Four Forms

### Form 1: C-style

```go
for i := 0; i < n; i++ {
    use(i)
}
```

Standard counted loop. Init runs once; cond checked before each iteration; post runs after each iteration.

### Form 2: while-style

```go
for cond {
    // ...
}
```

Same as form 1 with init and post omitted.

### Form 3: infinite

```go
for {
    // ...
    if shouldStop { break }
}
```

The idiomatic "loop forever; break when done." Common for event loops, retry pumps, signal handlers.

### Form 4a: range over slice / array / string

```go
for i, v := range slice {
    // i = index, v = COPY of element
}

for _, v := range slice { ... }  // discard index
for i := range slice { ... }     // discard value
for range slice { ... }          // discard both (count iterations)
```

For strings:

```go
for i, r := range "héllo" {
    // i = byte index, r = rune (UTF-8 decoded)
}
```

For maps (iteration order is **randomised** each time):

```go
for k, v := range m {
    // ...
}
```

For channels (until closed):

```go
for v := range ch {
    // received until ch is closed AND drained
}
```

### Form 4b: range over int (Go 1.22+)

```go
for i := range 10 {
    fmt.Println(i)   // 0, 1, ..., 9
}
```

Equivalent to `for i := 0; i < 10; i++`. Useful when N is the entire loop logic.

### Form 4c: range over function (Go 1.23+)

```go
for v := range myIter {
    // myIter is a func(yield func(V) bool)
    // Pull-based iterator; see 24-frontier/04
}
```

Covered in detail in `24-frontier/04-range-over-func-advanced.md`.

### Loop variable semantics (Go 1.22 change)

**Pre-1.22**:

```go
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs { f() }
// PRE-1.22: prints 3, 3, 3 — all closures share one i
// POST-1.22: prints 0, 1, 2 — each iteration has its own i
```

The change is automatic when your `go.mod` says `go 1.22` or later. See `09-closures.md`.

## `break` and `continue`

```go
for i := 0; i < 10; i++ {
    if i == 5 { break }      // exit the loop
    if i % 2 == 0 { continue } // skip to next iteration
    use(i)
}
```

### Labels

For breaking out of nested loops or specific switch:

```go
outer:
for i := 0; i < 10; i++ {
    for j := 0; j < 10; j++ {
        if condition(i, j) {
            break outer        // breaks the OUTER loop
        }
        if otherCondition {
            continue outer     // jumps to next outer iteration
        }
    }
}
```

Labels are scoped to the function. The label must precede a `for`, `switch`, or `select`.

## `switch`

```go
switch x {
case 1:
    fmt.Println("one")
case 2, 3:
    fmt.Println("two or three")
case 4:
    fmt.Println("four")
default:
    fmt.Println("other")
}
```

Key differences from C/Java:

- **No fall-through by default.** Each case ends implicitly.
- **`case` accepts a list** (`case 2, 3:`) — no need to chain.
- **`break` is implicit** — written `break` is legal but a no-op.
- **`fallthrough` is explicit** — must be the last statement in a case.

```go
switch x {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two (or one)")
}
```

`fallthrough` falls into the next case *regardless of its condition* (it doesn't re-check).

### Tagless switch (expression cases)

```go
switch {
case x > 0:
    fmt.Println("positive")
case x < 0:
    fmt.Println("negative")
default:
    fmt.Println("zero")
}
```

A `switch` without a tag is equivalent to `switch true`. The cases are full boolean expressions. This is Go's "long if-else" idiom — cleaner than chained `if`/`else if`.

### Init statement

```go
switch v := compute(); {
case v > 100:
    // ...
case v > 10:
    // ...
default:
    // ...
}
```

### Type switch

```go
switch v := x.(type) {
case int:
    use(v + 1)         // v is int
case string:
    use(v + "!")       // v is string
case nil:
    // ...
case io.Reader, io.Writer:
    // v is the original interface (since multiple types)
default:
    fmt.Printf("unknown: %T\n", v)
}
```

Covered in `04-type-conversion-and-assertions.md`.

### `select` for channels

```go
select {
case msg := <-ch1:
    use(msg)
case ch2 <- value:
    // sent successfully
case <-ctx.Done():
    return
default:
    // none ready — non-blocking
}
```

Like `switch` but for channel operations. If multiple cases are ready, one is chosen pseudo-randomly. `default` makes the select non-blocking. Covered in `07-concurrency/04-select-statement.md`.

## `goto` and Labels

```go
func find(items []int, target int) int {
    for i, v := range items {
        if v == target {
            goto found
        }
    }
    return -1
found:
    return i   // compile error — i not in scope here
}
```

`goto` exists. Rules:

- Cannot jump into a block from outside.
- Cannot skip variable declarations.
- Target must be in the same function.

Idiomatic uses are rare. Two legitimate ones:

1. **Cleanup chains** that pre-date `defer` (you rarely see this in modern Go).
2. **Generated code** (parser generators, state machines).

For all human-written code, `for`/`break`/`continue`/labels covers what `goto` would do, more readably.

## `return`

```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("divide by zero")
    }
    return a / b, nil
}

// Named returns + naked return
func split(s string, sep byte) (left, right string) {
    for i := 0; i < len(s); i++ {
        if s[i] == sep {
            left = s[:i]
            right = s[i+1:]
            return            // naked: returns left, right
        }
    }
    left = s
    return
}
```

Named returns + naked `return` is allowed but considered a code smell in long functions (the returned values aren't visible at the return site). Use sparingly. See `08-functions.md`.

## `defer` (brief mention)

```go
func process() error {
    f, err := os.Open("x")
    if err != nil { return err }
    defer f.Close()    // runs when process() returns
    // ...
}
```

`defer` schedules a function call to run when the surrounding function returns (or panics). Covered fully in `07-defer.md`.

## `panic` and `recover` (brief mention)

```go
func mayPanic() {
    defer func() {
        if r := recover(); r != nil {
            log.Println("recovered:", r)
        }
    }()
    panic("oh no")
}
```

Covered in `05-errors`. Use sparingly — Go errors are usually returned values, not exceptions.

## Patterns

### Guard clauses (early return)

```go
func process(r *http.Request) error {
    if r == nil { return errors.New("nil request") }
    if r.URL == nil { return errors.New("nil URL") }
    if r.Method != "POST" { return errors.New("must be POST") }
    // main logic
}
```

Idiomatic Go style: handle the negative cases first, leaving the main path unindented.

### Loop-then-finish pattern

```go
done := false
for !done {
    select {
    case msg := <-ch:
        if msg == "stop" {
            done = true
            break    // breaks the SELECT, not the loop
        }
        process(msg)
    case <-ctx.Done():
        done = true
    }
}
```

Note: `break` inside a `select` breaks the select, not the surrounding loop. To break the loop from inside a select, label the loop:

```go
loop:
for {
    select {
    case <-ch: ...
    case <-stop:
        break loop
    }
}
```

### Pump loop

```go
for {
    job, ok := <-queue
    if !ok { return }
    process(job)
}

// Or with range:
for job := range queue {
    process(job)
}
```

## Anti-Patterns & Gotchas

**`for cond {}` infinite without break.** Spin loop — burns 100% CPU. Add a `time.Sleep` or a channel wait.

**`break` inside a select expecting it to break the loop.** Use a label.

**Forgetting `continue` after a `case` in a `switch` inside a loop.** `switch` doesn't continue the outer loop; you need explicit `continue`.

**`if x = compute(); x != 0` not `if x := compute(); x != 0`.** `=` requires the variable already declared; `:=` declares it. Wrong form → compile error.

**Shadowing variables in `if`-init or `for`-init.** See `01-variables-and-constants.md`.

**Using `goto` for control flow.** Almost always replaceable with `for`/`break`/`continue`.

**Iterating a map and assuming order.** Map iteration is randomised by design (since Go 1.0). For ordered iteration, collect keys, sort, then loop.

**Modifying a map during `range`.** Adding entries: behaviour is unspecified — they may or may not be visited. Deleting entries: safe.

**Modifying a slice during `range`** by appending. The range captures `len(s)` at the start; the new entries are not visited. Use a `for i := 0; i < len(s); i++` if you need to see grown-during-iteration entries.

**Reading the value `v` in `for _, v := range slice` and modifying it expecting the slice to change.** `v` is a copy. Use `s[i]` directly to mutate.

**`fallthrough` in a case for some reason.** Almost never the right call. Use shared logic via `case 1, 2:` or extract to a function.

**Calling `break` in a non-loop context.** Compile error.

**`for range ch` when the channel isn't closed.** Deadlocks once nothing more is sent.

**Using `goto` to skip past variable declarations.** Compile error: "jumps over declaration."

**`switch x.(type)` outside an interface.** Compile error — only on interface-typed values.

**Naked `return` in a long function with named returns.** Hard to read; return values invisible.

**`if x := f(); x > 0 { ... } else if x := g(); x > 0 { ... }`** — two different `x`s! The second `:=` creates a new scope.

**Forgetting `default:` in a switch.** Sometimes intentional; sometimes a bug. The compiler doesn't require it. `exhaustive` linter checks for enum-like switches.

## Performance Notes

- **`if`**: zero overhead vs straight-line code.
- **`for`**: zero overhead vs straight-line code (init runs once, condition is a branch each iteration).
- **`switch`**: compiler may generate jump tables for dense integer cases; otherwise sequential comparisons. Type switches use an itab lookup per case.
- **`for range` over slice / array**: identical performance to `for i := 0; i < len(s); i++`.
- **`for range` over map**: each iteration is a hash-table walk step — slower than indexing. Avoid when iteration order matters; collect keys.
- **`for range` over channel**: per-iteration channel recv (~50-200 ns + scheduler overhead).
- **`select`**: per-case readiness check. Scales linearly with number of cases.
- **`goto`**: a JMP instruction; zero overhead.

## How Big Companies Use It

- **Google's internal style guide** mandates braces always and discourages naked `return`s.
- **Uber's Go style guide** prefers guard clauses + early return over deep nesting.
- **HashiCorp** projects use tagless `switch { case ... }` heavily for state machines.
- **Kubernetes** uses labeled loops in informer/controller code where breaking out of nested loops is unavoidable.
- **Cloudflare** uses `select { default: }` non-blocking patterns extensively in proxy/pump code.
- **CockroachDB** uses extensive type switches in SQL planner; migrating to generics where practical.

`goto` is virtually unused in production Go codebases. Searching the stdlib shows a few in `runtime` (low-level VM) and the compiler — that's about it.

## Source Code References

- Go spec — Statements: https://go.dev/ref/spec#Statements.
- Go spec — For statements: https://go.dev/ref/spec#For_statements.
- Go spec — Switch statements: https://go.dev/ref/spec#Switch_statements.
- Go spec — Select statements: https://go.dev/ref/spec#Select_statements.
- Go spec — Goto: https://go.dev/ref/spec#Goto_statements.
- Loop variable change (1.22): https://go.dev/blog/loopvar-preview.

## Further Reading

- Effective Go — Control structures: https://go.dev/doc/effective_go#control-structures.
- Russ Cox, "Go 1.22 loop variable change": https://go.dev/blog/loopvar-preview.
- "Go control flow" — chapter in *The Go Programming Language*.
- "exhaustive" linter for switch coverage: https://github.com/nishanths/exhaustive.
- Rob Pike, "Less is exponentially more" (philosophy of Go's minimal control flow): various talks.

## Exercises / Self-Check

1. Convert a chain of `if/else if/else` into a tagless `switch`. Compare readability.
2. Use a labeled `break` to exit a nested loop. Verify the inner `break` alone doesn't suffice.
3. Range over a map twice. Confirm the iteration order differs.
4. Use a `for range 10` (1.22+). Verify the generated code is identical to `for i := 0; i < 10; i++`.
5. Capture loop variable in a goroutine: `for i := 0; i < 3; i++ { go func() { fmt.Println(i) }() }`. With `go 1.22`, observe correct output; revert to `go 1.21` and observe the bug.
6. Build a select-based event loop. Add a labeled break to exit on a shutdown signal.
7. Try `goto` inside a function. Then refactor to use `for`/`break`. Which reads better?
8. Add a `case 2, 3:` in a switch. Compare to two separate cases with `fallthrough`.
