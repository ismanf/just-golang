# Naming Conventions

## TL;DR

Go naming is **opinionated and consistent across the ecosystem**: lowercase short package names, mixedCaps for identifiers (no underscores or kebab-case), exported-by-capitalization, acronyms stay all-caps in identifiers (`URLParser`, `userID`), receivers are 1-2 letters matching the type, single-method interfaces end in `-er` (`Reader`, `Writer`). The single biggest gotcha: **good names compress meaning**. A long descriptive name like `numberOfRetriesBeforeFailure` is often worse than `maxRetries` with a doc comment. The Go style cares more about *contextual brevity* than self-documenting verbosity.

## Mental Model

```
   The hierarchy of names in Go:
   
   Package names:        lowercase, single word, short      bytes, http, user
   Exported identifiers: MixedCaps, capital first letter    NewReader, MaxLines
   Unexported:           mixedCaps, lowercase first         readLine, maxBuf
   Constants:            same as variables (no SCREAMING)   MaxIter, defaultPort
   Receiver names:       1-2 letters, consistent             (s *Server), (b *Buffer)
   Interfaces (1-method): -er suffix                        Reader, Writer, Stringer
   Initialisms:          all caps in identifiers             URL, HTTPS, ID, ASCII
   File names:           snake_case for readability          user_service.go
```

The rules feel restrictive at first; they reward you with code that reads consistently across every Go codebase you'll ever encounter.

## Syntax & Basic Usage

```go
package user                              // package name: lowercase, short

const MaxNameLen = 64                     // exported constant: MixedCaps
const defaultTimeout = 30 * time.Second   // unexported constant: lowercase

type Service struct {                     // exported type: MixedCaps
    repo *Repository
    log  *slog.Logger
}

type repository struct{ /* ... */ }        // unexported type

func NewService(repo *Repository) *Service { // constructor
    return &Service{repo: repo}
}

func (s *Service) Register(ctx context.Context, u User) error { // method
    return s.repo.Save(ctx, u)
}

type Repository interface {                // interface in the consumer
    Save(ctx context.Context, u User) error
    Get(ctx context.Context, id string) (User, error)
}
```

## Deep Dive

### Package names

Rules:
- **Lowercase.**
- **Single word.** No `multi_word` or `multiWord`. (Exception: `httptest`, `tarutils` — these are bad examples some libraries still ship.)
- **Short.** `fmt`, `io`, `os`, `net`, `http`, `time`.
- **Descriptive of what the package provides.**
- **No utility-package names.** `util`, `helpers`, `common`, `misc` — bad.

The Go authors are explicit about this in https://go.dev/blog/package-names.

Examples done well in stdlib: `bytes`, `bufio`, `compress`, `database`, `debug`, `encoding`, `flag`, `image`, `io`, `log`, `math`, `mime`, `net`, `os`, `path`, `reflect`, `regexp`, `runtime`, `sort`, `strconv`, `strings`, `sync`, `syscall`, `text`, `time`, `unicode`.

### Exported vs unexported

**Capital first letter = exported.**

```go
package foo

var Public int     // accessible as foo.Public from outside
var private int    // only accessible within package foo
```

Same rule for type names, function names, field names, method names. No `public/private` keywords; no `protected`. Just capitalization.

### MixedCaps, not snake_case

```go
// Bad
var max_size int
func calculate_average() float64 {}

// Good
var maxSize int
func calculateAverage() float64 {}
```

Always. The compiler accepts `under_scores`, but `go vet` and the community don't.

### Initialisms

Initialisms (acronyms) keep their case:

```go
// Bad
type UrlParser struct{}
var userId int

// Good
type URLParser struct{}
var userID int
```

`go vet` (since several versions ago) and `revive` flag violations. Common offenders: `Url` → `URL`, `Id` → `ID`, `Html` → `HTML`, `Tcp` → `TCP`, `Sql` → `SQL`, `Json` → `JSON`, `Xml` → `XML`, `Ip` → `IP`.

Exception: at the start of an unexported identifier, lowercase: `urlParser` (not `uRLParser`).

### Receiver names

- **Short**: 1-3 letters.
- **Consistent** across all methods on a type.
- **Lowercase**.
- Often the first letter(s) of the type name.

```go
type Server struct{ /* ... */ }
func (s *Server) Start() error { /* ... */ }
func (s *Server) Stop() error  { /* ... */ }

type Cache struct{ /* ... */ }
func (c *Cache) Get(k string) (V, bool) { /* ... */ }
```

Never `this`, `self`, or `me`. Never the type name fully written out (`func (server *Server) Start()` is too long).

### Interface names

Single-method interfaces: name ends in `-er`:

```go
type Reader interface { Read(p []byte) (int, error) }
type Writer interface { Write(p []byte) (int, error) }
type Stringer interface { String() string }
type Closer interface { Close() error }
```

For interfaces with several methods, the name describes the abstraction:

```go
type Handler interface { ServeHTTP(ResponseWriter, *Request) }
type Repository interface { Save(...); Get(...); Delete(...) }
```

Avoid pseudo-Java patterns like `IRepository` or `RepositoryInterface`.

### Variable names

- **Short for short scope**: `i` for loop indices, `r` for the file you just opened a few lines up.
- **Longer for longer scope**: `connectionPool`, `userRepository`.
- **Acronyms**: `httpClient`, not `httpcli` or `HTTPCli`.

```go
func main() {
    // Short scope
    for i := 0; i < 10; i++ { /* ... */ }

    // Longer scope: package-level variable
    var defaultRetryPolicy = Policy{Attempts: 3, Backoff: time.Second}
}
```

### Constant names

Constants follow variable rules. **No SCREAMING_SNAKE_CASE**:

```go
// Bad
const MAX_RETRIES = 3
const DEFAULT_TIMEOUT = 30

// Good
const MaxRetries = 3
const defaultTimeout = 30 * time.Second
```

### Enum-style constants

`iota`-based enums conventionally use a prefix or grouping name:

```go
type Status int

const (
    StatusUnknown Status = iota
    StatusActive
    StatusInactive
    StatusBanned
)
```

The prefix `Status` makes them globally distinguishable (`StatusActive`, not just `Active`, in case `Active` collides).

For unexported enums in a package, sometimes no prefix:

```go
const (
    statusUnknown = iota
    statusActive
    statusInactive
)
```

### Type names

Exported types are `MixedCaps`:

```go
type Server struct{ /* ... */ }
type User struct{ /* ... */ }
type Repository interface{ /* ... */ }
```

For types that wrap primitives (defined types):

```go
type Currency string         // exported
type backoff time.Duration   // unexported
```

### Function and method names

```go
// Exported function: noun or verb depending on what it does
func NewServer(addr string) *Server { /* ... */ }   // returns a Server — noun
func ParseRequest(b []byte) (*Request, error)       // verbs are also fine

// Method: usually a verb
func (s *Server) Start() error { /* ... */ }
func (s *Server) Listen(addr string) error { /* ... */ }
```

Constructors: `NewX` or `MakeX`. `NewX` returns `*X`; `MakeX` returns `X` (like `make([]int, 10)` returns the value, not a pointer).

In practice: `NewX` is the dominant convention; `MakeX` is rare outside stdlib.

### Field names

Struct fields follow MixedCaps:

```go
type User struct {
    ID        string
    Name      string
    Email     string
    CreatedAt time.Time
    metadata  map[string]any  // unexported
}
```

Embedded fields use the type name:

```go
type Server struct {
    *http.Server                // embedded; access as s.Server
    log *slog.Logger
}
```

### Filenames

Files use `snake_case`:

```
user_service.go
user_service_test.go
user_repository.go
```

Underscores OK in filenames (the only place); not in identifiers. Test files MUST end in `_test.go`.

OS/arch-specific files use suffix conventions:

```
file_linux.go      // builds on linux
file_linux_amd64.go // builds on linux/amd64
```

### Avoid stuttering

Package + identifier should read smoothly:

```go
// Bad (stutters: "user.UserService")
package user
type UserService struct{}

// Good
package user
type Service struct{}

// Caller: user.Service — non-stuttering.
```

Apply to type, function, method, field names within a package.

### Generic type parameter names

Usually one capital letter:

```go
func Map[T, U any](in []T, f func(T) U) []U
type Result[T any] struct { Value T }
```

For two-parameter helpers, `K` and `V` are conventional (key, value):

```go
func Keys[K comparable, V any](m map[K]V) []K
```

For ordered types: `T`. For comparable: `T`. Stick with single letters; multi-letter type parameter names are rare and usually noise.

### Error variables

Sentinel errors: `ErrX`:

```go
var ErrNotFound = errors.New("not found")
var ErrInvalidArg = errors.New("invalid argument")
```

Error types: `XError`:

```go
type ValidationError struct{ Field, Msg string }
func (e ValidationError) Error() string { return e.Field + ": " + e.Msg }
```

### Test names

`TestX`, `BenchmarkX`, `ExampleX`, `FuzzX`:

```go
func TestParse(t *testing.T) { /* ... */ }
func BenchmarkParse(b *testing.B) { /* ... */ }
func ExampleParse() { /* ... */ }
func FuzzParse(f *testing.F) { /* ... */ }
```

For sub-tests, use lower-case descriptors:

```go
t.Run("empty input", func(t *testing.T) { /* ... */ })
t.Run("special chars", func(t *testing.T) { /* ... */ })
```

### Avoid using built-in names

Don't name variables `len`, `cap`, `new`, `delete`, `error`, `nil`. Compiler allows it; readers suffer.

```go
// Bad
func process(new int) int { /* shadows built-in `new` */ }

// Good
func process(value int) int
```

### Names for getter / setter methods

**Getter**: just the field name as a method. **No** `Get` prefix:

```go
// Bad
func (u *User) GetName() string

// Good
func (u *User) Name() string
```

**Setter**: `SetX`:

```go
func (u *User) SetName(name string)
```

### Boolean naming

Use positive forms:

```go
// Bad
func (s *Server) IsNotReady() bool

// Good
func (s *Server) IsReady() bool
```

`HasX`, `IsX`, `CanX` are common prefixes.

### Mutable vs immutable hints

By convention:
- `New X` — constructor of a new fresh value.
- `Make X` — same, less common.
- `WithX` — returns a new value with a field changed (immutable update).
- `SetX` — mutates the receiver.

```go
ctx := context.Background()
ctx = context.WithValue(ctx, key, value)    // immutable derived context
```

### Avoid Hungarian notation

Don't encode type in the name:

```go
// Bad (Hungarian)
var nUsers int
var sName string

// Good
var users int
var name string
```

Go's type system makes type prefixes redundant.

### Internal package boundary

Identifiers inside `internal/` packages can be exported but won't escape. Naming is the same; visibility is filesystem-based, not naming-based.

### Common mistakes from non-Go backgrounds

#### From Java/C#

- `IRepository`, `IUser` — drop the `I`.
- `getOwner()`, `setOwner(x)` — drop `get`, keep `set`.
- `UserService`, `UserManager`, `UserHelper` — too "patterns-y". Just `Service` if it lives in package `user`.

#### From Python

- `snake_case` — switch to `mixedCaps`.
- `__init__` — Go has constructors as regular functions.
- Module = package; one file per class — Go is one directory per package.

#### From C++

- `m_field`, `s_static` — drop prefixes.
- `template<typename T>` → `[T any]`.
- Header files / declarations — Go has no headers.

### When in doubt

Look at the standard library. Standardize on its style. Read what `stdlib` modules call their types and methods; mirror the patterns.

`pkg.go.dev/std` lets you browse every stdlib package's symbols.

## Standard Library Hooks

- `gofmt`: doesn't enforce naming.
- `go vet`: catches some (`composites`, `printf`).
- `staticcheck`: more (e.g., `ST1003` for naming).
- `revive`: rule-based; covers most naming guidelines.
- `golangci-lint`: aggregates.

## Real-World Patterns

### 1. Consistent receiver names

```go
type DB struct{}
func (d *DB) Query(...) {...}
func (d *DB) Exec(...) {...}
// All `d`. Never switch to `db` or `database`.
```

### 2. Constructor pattern

```go
func NewServer(addr string, opts ...Option) *Server { /* ... */ }
```

### 3. Error sentinel

```go
package store

var ErrNotFound = errors.New("not found")

// usage
if errors.Is(err, store.ErrNotFound) { /* ... */ }
```

### 4. Option pattern names

```go
type Option func(*Server)
func WithTimeout(d time.Duration) Option { /* ... */ }
func WithLogger(l *slog.Logger) Option   { /* ... */ }
```

### 5. Test naming

```go
func TestParse(t *testing.T) {
    t.Run("empty input", func(t *testing.T) { /* ... */ })
    t.Run("valid", func(t *testing.T) { /* ... */ })
    t.Run("malformed", func(t *testing.T) { /* ... */ })
}
```

## Anti-Patterns & Gotchas

**Stuttering**: `user.UserService` reads as redundant. Drop the prefix.

**Hungarian notation**: `iCount`, `sName`. The type system handles this.

**Long descriptive variable names** in short scopes: `forEachConnectionInThePool := range conns`. `for _, c := range conns`.

**`Get`-prefixed getters**: non-Go.

**SCREAMING_SNAKE_CASE constants**: non-Go.

**snake_case identifiers** (anywhere except filenames).

**Lowercase initialisms**: `userId` → `userID`.

**`I` prefix on interfaces**: `IRepository` → `Repository`.

**`Impl` suffix on implementations**: `RepositoryImpl` → `repositoryPostgres` or similar concrete name.

**Naming a method after its return type**: `func (u *User) IsString() string`. Method name should describe action, not return.

**`init` for non-trivial work**: see other guides.

**Field names that match method names** on the same struct: callers can't distinguish.

## Performance Notes

Naming is purely stylistic; zero runtime impact. Linters and `gopls` work harder on long names, but it's microseconds.

## How Big Companies Use It

Every published Go style guide agrees on naming. Universal across:
- Standard library.
- Google.
- Uber.
- Kubernetes.
- CockroachDB.
- HashiCorp.
- Tailscale.
- Every YC company doing Go.

## Source Code References

- Effective Go § Naming: https://go.dev/doc/effective_go#names.
- Code Review Comments wiki: https://go.dev/wiki/CodeReviewComments.
- Google Go style guide: https://google.github.io/styleguide/go/decisions#naming.
- Uber Go style guide: https://github.com/uber-go/guide#style.
- `revive`'s naming rules: https://github.com/mgechev/revive#available-rules.
- `staticcheck` ST1003 (naming): https://staticcheck.dev/docs/checks#ST1003.

## Further Reading

- "What's in a name?" — Rob Pike, JFKL Conf 2014: talks.
- Andrew Gerrand, "Package names": https://go.dev/blog/package-names.
- Dave Cheney, "Practical Go" lecture: https://dave.cheney.net/practical-go.
- "Effective Go" naming section.
- Mat Ryer talks on Go API design.

## Exercises / Self-Check

1. Audit a Go file in your codebase for naming violations. List them.
2. Why does Go use `MixedCaps` instead of `snake_case`? What's the historical rationale?
3. Rename a stuttering type in your project. Verify the result reads cleaner at call sites.
4. Why are getter prefixes `Get` discouraged but setter prefixes `Set` allowed?
5. Take a 30-character variable name in your code. Can you cut it without losing clarity?
