# Uber Go Style Guide

## TL;DR

The **Uber Go style guide** (https://github.com/uber-go/guide) is the most widely-cited *community* Go style guide. Begun in 2018, maintained on GitHub by Uber engineers, it's dense and prescriptive — over 100 rules covering performance, error handling, naming, and patterns specific to large-scale Go services. Unlike the Google guide (rule-focused with rationales), Uber's reads like **"things we learned the hard way at Uber's scale"** — each rule paired with **bad** vs **good** code examples. The single biggest gotcha: **some Uber-isms reflect Uber's tooling and culture** (heavy use of `zap`, `fx`, `errgroup`); they're sensible defaults but not law. Read for the antipattern examples; adopt rules that match your stack.

## Mental Model

```
   Three big public Go style guides:
   
   Effective Go       — principles, foundational, dated
   Google Go style    — rules + rationale, modern, broad
   Uber Go style      — rules + bad/good examples, practical, dense
   
   Each is the source for a different question:
     - "Why?"             → Effective Go
     - "What's the rule?" → Google
     - "What does bad look like?" → Uber
```

## Syntax & Basic Usage

This page surveys notable rules from the Uber guide. The guide itself is on GitHub: https://github.com/uber-go/guide/blob/master/style.md.

## Deep Dive

### History

- **2017-2018**: Uber's Go team curated internal style notes.
- **2018**: Published on GitHub; received thousands of stars.
- **2019-present**: Quarterly-ish updates from Uber engineers and community PRs.

The guide became the de-facto "second opinion" for Go style outside Google.

### Structure

The guide is one long Markdown file. Sections include:

- Introduction.
- Guidelines (numerical rule list).
- Performance.
- Style.
- Patterns.

Each rule has a "**Bad**" code snippet and a "**Good**" alternative, with terse explanation.

### Notable rules (a curated selection)

#### Guidelines

**Avoid embedding types in public structs.** External users can't avoid the dependency:

```go
// Bad
type Server struct {
    *http.Server  // exposes http.Server publicly
}

// Good
type Server struct {
    srv *http.Server
}
```

**Avoid using built-in names as identifiers.** Shadowing `len`, `cap`, `new`, `delete` causes confusion:

```go
// Bad
func compute(new int) int { /* shadows built-in `new` */ }

// Good
func compute(n int) int { ... }
```

**Avoid `init()`.** Lazy initialization preferred:

```go
// Bad
var conn *sql.DB
func init() { conn, _ = sql.Open(...) }

// Good
func New() (*App, error) { return &App{conn: ...}, nil }
```

**Initialize maps with capacity hints when known**:

```go
// Bad
m := make(map[string]int)

// Good
m := make(map[string]int, len(items))
```

#### Performance

**Prefer `strconv` over `fmt`:**

```go
// Bad (allocates)
s := fmt.Sprintf("%d", n)

// Good (no allocs)
s := strconv.Itoa(n)
```

**Avoid `string(byte_slice)` allocations:**

```go
// Bad
m[string(b)] = v  // allocates string

// Good (since Go optimizes m[string(b)] for lookups)
v := m[string(b)]  // OK if just lookup
```

**Specify slice capacity when known:**

```go
// Bad
s := make([]int, 0)

// Good
s := make([]int, 0, len(items))
```

#### Style

**Group similar declarations:**

```go
// Bad
var a int
var b string
var c bool

// Good
var (
    a int
    b string
    c bool
)
```

**Imports**: stdlib, external, internal — three groups, blank lines.

**Top-level variable declarations**: use `var` not `:=`.

```go
// Bad (at package level)
foo := bar.Foo()

// Good
var foo = bar.Foo()
```

**Use raw string literals for regexes:**

```go
// Bad
re := regexp.MustCompile("\\d+")

// Good
re := regexp.MustCompile(`\d+`)
```

**Initialize struct fields by name:**

```go
// Bad
p := Point{1, 2}

// Good
p := Point{X: 1, Y: 2}
```

#### Patterns

**Use functional options:**

```go
// Good
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{addr: addr}
    for _, opt := range opts { opt(s) }
    return s
}
```

**Use enum constants with iota and a String() method:**

```go
type Status int

const (
    StatusUnknown Status = iota
    StatusActive
    StatusInactive
)

func (s Status) String() string {
    return [...]string{"unknown", "active", "inactive"}[s]
}
```

**Use `errors.Is` and `errors.As`:**

```go
if errors.Is(err, sql.ErrNoRows) {
    // handle missing row
}
```

#### Concurrency

**Always check `defer wg.Done()`** when using `WaitGroup`.

Or use **`WaitGroup.Go`** (1.25+):

```go
var wg sync.WaitGroup
wg.Go(func() { /* ... */ })
wg.Wait()
```

**Use `errgroup` for parallel work with error propagation:**

```go
g, ctx := errgroup.WithContext(ctx)
for _, item := range items {
    item := item
    g.Go(func() error { return process(ctx, item) })
}
if err := g.Wait(); err != nil { return err }
```

**Avoid `goroutine`-per-request without bounds:**

```go
// Bad — unbounded
for _, item := range items {
    go process(item)
}

// Good — bounded via semaphore or errgroup limit
sem := make(chan struct{}, 100)
for _, item := range items {
    sem <- struct{}{}
    go func() { defer func() { <-sem }(); process(item) }()
}
```

#### Errors

**Wrap errors with `%w`** when adding context:

```go
return fmt.Errorf("process %s: %w", id, err)
```

**Sentinel errors** named `ErrX`:

```go
var ErrNotFound = errors.New("not found")
```

**Multiple errors** with `errors.Join`:

```go
return errors.Join(err1, err2)
```

#### Logging

Uber, naturally, prefers **`zap`** (their library):

```go
import "go.uber.org/zap"

log, _ := zap.NewProduction()
defer log.Sync()
log.Info("request", zap.String("path", r.URL.Path), zap.Int("status", code))
```

For new code outside Uber: **`slog`** is the modern equivalent.

#### Testing

**Table-driven tests:**

```go
tests := []struct {
    name string
    in   int
    want int
}{
    {"zero", 0, 0},
    {"positive", 1, 1},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := square(tt.in)
        if got != tt.want { t.Errorf("got %d, want %d", got, tt.want) }
    })
}
```

**`t.Helper()` in helper functions:**

```go
func mustOK(t *testing.T, err error) {
    t.Helper()
    if err != nil { t.Fatalf("unexpected error: %v", err) }
}
```

#### Performance gotchas

**`strings.Builder` over string concatenation:**

```go
// Bad
s := ""
for _, p := range parts { s += p }

// Good
var sb strings.Builder
for _, p := range parts { sb.WriteString(p) }
s := sb.String()
```

**`sync.Pool` for hot allocations:**

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func process(b []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() { buf.Reset(); bufPool.Put(buf) }()
    // use buf
}
```

### Uber-isms (specific to Uber tooling)

#### `zap` everywhere

Uber's logger. Faster than logrus or stdlib log. External code increasingly uses `slog` (1.21+).

#### `fx` for DI

Uber's DI framework. See `19-patterns/03-dependency-injection.md`.

#### `goleak` in tests

Detects goroutine leaks per test:

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

#### `automaxprocs`

Pre-1.25, this set GOMAXPROCS from cgroup CPU quota. Go 1.25+ does this natively.

#### `multierr`

Uber-specific multi-error type (predates `errors.Join`). New code uses `errors.Join`.

### Things the Uber guide forbids

- `init()` for non-trivial work.
- Mutex value-copy via value receivers.
- Goroutine leaks (use `goleak` in tests).
- Returning interfaces as concrete types (`return X` where X is interface — accept that interface).
- Unbounded goroutine fan-out.
- Magic numbers in code (use constants).

### Things the Uber guide endorses

- Functional options.
- Constructor pattern `NewX(...)`.
- Small interfaces.
- "Accept interfaces, return structs."
- Context as first parameter.
- Errors wrapped with %w.
- Table-driven tests.
- Goroutine bounds via semaphore / errgroup.

### Uber-vs-Google differences

- **Logging**: Uber uses zap; Google's klog (internal) / slog (external).
- **DI**: Uber uses fx; Google uses Wire.
- **Error multi-handling**: Uber's multierr → community's errors.Join.
- **Performance section**: Uber's is more aggressive (specific micro-optimizations).
- **Antipattern coverage**: Uber more dense; Google more rationale-focused.

### Adoption

The Uber guide is unusually widely adopted *outside Uber*:

- Many startups.
- Several CNCF projects.
- Numerous Go books cite it.
- golangci-lint's `revive` linter has Uber-style rules.

### Counter-rules

Not everyone agrees with every Uber rule. Common pushback:

- **"Always specify slice capacity"** — sometimes you don't know capacity; the rule shouldn't force a guess.
- **"Avoid init"** — some libraries fundamentally need init for driver registration.
- **"Use named fields in struct literals"** — for 2-field tuples, positional is fine.

Take the guide as advice, not law.

### How to apply

1. Read it once start-to-finish.
2. Configure a linter (`revive` is Uber-compatible).
3. Discuss disagreements as a team.
4. Codify your team's adapted version.
5. Re-read every year; updates come.

## Standard Library Hooks

- `errgroup` (golang.org/x/sync/errgroup): bounded parallel work.
- `goleak` (go.uber.org/goleak): leak detection.
- `automaxprocs` (go.uber.org/automaxprocs): pre-1.25 helper.
- `zap` (go.uber.org/zap): logging.
- `fx`, `dig` (go.uber.org/fx, dig): DI.
- `multierr` (go.uber.org/multierr): legacy; use `errors.Join`.

## Real-World Patterns

### 1. Table-driven test with subtests

```go
tests := []struct {
    name   string
    input  string
    want   string
    err    string
}{
    {"empty", "", "", "empty input"},
    {"basic", "hello", "HELLO", ""},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := upper(tt.input)
        if tt.err != "" {
            if err == nil || !strings.Contains(err.Error(), tt.err) {
                t.Fatalf("err=%v want %q", err, tt.err)
            }
            return
        }
        if err != nil { t.Fatal(err) }
        if got != tt.want { t.Errorf("got %q want %q", got, tt.want) }
    })
}
```

### 2. Bounded errgroup

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(50)
for _, item := range items {
    item := item
    g.Go(func() error { return process(ctx, item) })
}
if err := g.Wait(); err != nil { return err }
```

### 3. goleak in tests

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

### 4. Functional options + zap

```go
type Server struct {
    log *zap.Logger
    /* ... */
}

type Option func(*Server)

func WithLogger(l *zap.Logger) Option { return func(s *Server) { s.log = l } }

func New(opts ...Option) *Server {
    s := &Server{log: zap.NewNop()}
    for _, o := range opts { o(s) }
    return s
}
```

### 5. errors.Join (replaces multierr)

```go
var errs []error
for _, item := range items {
    if err := process(item); err != nil {
        errs = append(errs, err)
    }
}
return errors.Join(errs...)
```

## Anti-Patterns & Gotchas

**Copying every Uber-ism into a small project.** The guide is dense; selective adoption is wise.

**Skipping the Bad/Good examples.** They're often the most instructive part.

**Adopting `zap` without checking `slog`.** `slog` is stdlib since 1.21; new code should consider it.

**Multi-error via `multierr` after 1.20.** Use `errors.Join`.

**Unbounded goroutines.** Uber's specific scale showed this fails; everyone since agrees.

**Naming receivers `this`/`self`.** Non-Go; rejected by every guide.

**Mutex copying via value receivers** on types embedding `sync.Mutex`. `go vet -copylocks` flags this.

**Skipping struct literal field names** for non-trivial types.

**Manual flag parsing where stdlib `flag` works.** `flag` is fine; reach for cobra only when you need subcommands.

**Cargo-culting "always specify capacity".** Sometimes you don't know.

## Performance Notes

Uber's guide emphasizes performance heavily. Notable wins documented:

- `strconv.Itoa(n)` vs `fmt.Sprintf("%d", n)`: ~10× faster, 0 allocs vs 2.
- `strings.Builder` vs `+=`: dramatically less allocation.
- `sync.Pool` for hot buffers: significant GC reduction.
- Slice pre-sizing: removes growslice calls.

Most micro-optimizations: invisible until profiled. Don't preemptively apply; profile first.

## How Big Companies Use It

- **Uber**: source of truth internally.
- **Many startups**: adopt wholesale.
- **CNCF projects**: cite as a reference.
- **Most Go shops**: use as a "+1" to Effective Go.

## Source Code References

- Uber Go style guide: [`uber-go/guide`](https://github.com/uber-go/guide).
- zap: [`uber-go/zap`](https://github.com/uber-go/zap).
- fx: [`uber-go/fx`](https://github.com/uber-go/fx).
- goleak: [`uber-go/goleak`](https://github.com/uber-go/goleak).
- automaxprocs: [`uber-go/automaxprocs`](https://github.com/uber-go/automaxprocs).
- multierr: [`uber-go/multierr`](https://github.com/uber-go/multierr).
- errgroup: [`golang.org/x/sync/errgroup`](https://pkg.go.dev/golang.org/x/sync/errgroup).

## Further Reading

- Uber Go style guide: https://github.com/uber-go/guide.
- Uber's logger zap: https://github.com/uber-go/zap.
- "Why we wrote zap" — Uber blog.
- "Idiomatic Go" talks by Uber engineers at GopherCon.
- "Performance Go" — Damian Gryski.
- Comparison: Uber vs Google style — community blog posts.
- "Go At Scale" — Tyler Treat talks.

## Exercises / Self-Check

1. Browse the Uber guide top to bottom. List five rules you didn't already follow.
2. Configure `golangci-lint` with `revive` enabled. Run on your codebase. Audit findings.
3. Adopt `errgroup.WithContext` + `SetLimit` for one place currently using unbounded goroutines. Verify correctness.
4. Add `goleak.VerifyTestMain` to one package. Did it catch anything?
5. Compare Uber's logging advice (zap) vs the stdlib's `slog`. Which would you choose for a new project?
