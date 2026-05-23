# Package Design — Cohesion, Naming, Cyclic Dependencies

## TL;DR

A Go **package** is the unit of compilation, dependency, and API. Good package design is **single-purpose, cohesive, well-named, and free of cyclic dependencies**. The Go community has converged on a small set of design principles: prefer few packages with clear purpose over many shallow ones; put narrow interfaces at consumers; keep packages flat unless splitting earns its keep; use `internal/` to hide implementation. The single biggest gotcha: **packages aren't classes**. Don't model your domain by giving every entity its own package. Group code that *changes together*, not code that *talks about the same noun*.

## Mental Model

```
   Good package boundaries:
   
   ┌──────────────────────────────────────────────────────────┐
   │  myproj/                                                   │
   │    cmd/                                                     │
   │      server/main.go                                         │
   │    internal/                                                │
   │      user/                user.go service.go repository.go │
   │      order/               order.go service.go ...           │
   │      auth/                auth.go middleware.go             │
   │    pkg/                   public APIs only                   │
   │      types/               cross-domain shared types          │
   └──────────────────────────────────────────────────────────┘
   
   Each package:
     - One purpose (single responsibility).
     - Self-contained or depends only on more general packages.
     - Small public API; details under internal/.
     - No cyclic imports.
```

The "right" size is workload-dependent; a flat structure (one or few packages) is fine for small services.

## Syntax & Basic Usage

There's no syntax to demonstrate; this page surveys principles.

## Deep Dive

### One purpose per package

A package should have **one** reason to exist. Examples from stdlib:
- `bytes` — operations on byte slices.
- `bufio` — buffered I/O.
- `crypto/sha256` — SHA-256 hashing.

A package called `utils` or `helpers` has no purpose; it's a graveyard for "stuff I needed to put somewhere".

If you can't summarize a package in one sentence, split it.

### Names

Covered in `06-packages-modules/01-package-fundamentals.md` and `22-idioms/05-naming-conventions.md`. Briefly:

- Lowercase, single word.
- No `util`, `common`, `helpers`, `misc`.
- The package name is part of identifiers: `bytes.Buffer`, not `bytes.BytesBuffer` (no stutter).

### Package as a binary boundary

Two reasons to introduce a package boundary:

1. **Reusability**: another caller wants the code.
2. **Encapsulation**: hide implementation behind a stable API.

If neither applies, the code can stay in the calling package. A package per-file or per-class is over-engineering.

### Don't model by noun

Anti-pattern from OOP languages:

```
user/
  user.go         (User struct)
order/
  order.go        (Order struct)
order_item/
  order_item.go   (OrderItem struct)
order_status/
  order_status.go (OrderStatus type)
```

Better:

```
order/
  order.go          (Order, OrderItem, OrderStatus all in one place)
  service.go        (OrderService)
```

Related types that change together belong in the same package. Splitting "every noun is a package" creates artificial boundaries.

### Cyclic imports

Go forbids cyclic imports at the package level. If `a` imports `b` and `b` imports `a`, the compiler refuses.

Common causes:
- Types from package A referenced in package B's interface, and types from B in A's.
- Helper functions placed in a "central" package that depends on consumers.

Resolutions:
- Move shared types to a third package (e.g., `types`).
- Define interfaces at the consumer: package B defines what it needs from A; A's concrete type satisfies it.
- Restructure: cyclic imports usually indicate a layering bug.

```go
// Bad: A and B mutually depend
package a
import "myproj/b"
type Foo struct{ B *b.Bar }

package b
import "myproj/a"
type Bar struct{ A *a.Foo }

// Better: types in shared package
package types
type Foo struct{}
type Bar struct{}

// a and b both import types; no cycle.
```

### Layering

A common Go layering (variants exist):

```
cmd/...              (binaries; depend on many)
   ↓
internal/service/    (use cases; depend on repos)
   ↓
internal/repository/ (persistence adapters)
   ↓
internal/domain/     (entities, value objects, errors)
```

Lower layers depend on nothing or only on more abstract things. This avoids cycles automatically.

### `internal/` directories

Packages under `internal/` are importable only by packages in the same module (or sub-modules of the same module). Use them aggressively for non-public code.

```
myorg/myproj/
  internal/
    store/     ← can be imported by anything in myorg/myproj/...
    auth/     ← same
  pkg/
    api/      ← public; other modules can import
```

Default to `internal/`; promote to `pkg/` only when you intend a public API.

### `pkg/` and the layout debate

The community is split on `pkg/`:

- **Pro `pkg/`**: clearly separates "intended for export" from internal. Pattern in `golang-standards/project-layout`.
- **Anti `pkg/`**: redundant; the module root or top-level packages serve as public API.

The Go team doesn't endorse `pkg/`; it isn't in stdlib's layout. For most projects, either is fine. Be consistent within a project.

### File layout within a package

Multiple files in one package are fine and common. Group by feature, not by type:

```
order/
  order.go          (Order struct, basic methods)
  service.go        (OrderService)
  repository.go     (Repository interface + adapters)
  pricing.go        (pricing logic)
  validation.go     (input validation)
  order_test.go
```

Anti-pattern: one file per struct. Splitting `Order` and `OrderItem` and `OrderStatus` into separate files when they're inter-dependent.

### Public API surface

Every exported identifier is part of your API. Each adds a maintenance commitment.

Practice:
- Default to unexported.
- Export only what callers genuinely need.
- Remove unused exports proactively.

### Interfaces — where to define them

**Where they're consumed, not where they're implemented.**

```go
// Bad — interface in producer
package userpg
type UserRepository interface { Get(...) }    // declared with the implementation
type Repo struct { db *sql.DB }              // implementation

// Good — interface in consumer
package service
type UserRepository interface { Get(...) }    // declared in the consumer

// In userpg:
type Repo struct { db *sql.DB }              // implementation; satisfies service.UserRepository implicitly
```

The producer doesn't import the consumer's interface; Go's structural typing means it just has to match.

Exception: when the interface is **the contract you're publishing** (`io.Reader`, `http.Handler`), define it in the package that owns the abstraction.

### Doc-driven design

A package's doc comment is the cover letter:

```go
// Package retry implements exponential-backoff retry helpers.
//
// # Quick start
//
// Use [Do] ...
package retry
```

If you can't write a clean package doc, the package's purpose isn't clear. Refactor.

### The "package per service" mistake

Microservice teams sometimes mirror their service architecture in packages:

```
user-service/
  user/
    handler/
      handler.go
    service/
      service.go
    repository/
      repository.go
```

Three packages where one would do. The `internal/user/` flat layout is usually better unless these layers are *individually* reusable (rare).

### Test packages: internal vs external

A test file in `package foo` (internal test) can access unexported names. A test file in `package foo_test` (external test) accesses only the public API.

```go
// foo.go        — package foo
// foo_test.go   — package foo or package foo_test
// foo_external_test.go — package foo_test
```

Both can coexist in the same directory. External tests guarantee you can build a working consumer using only the public API; internal tests verify private helpers.

### Avoiding "common" packages

Examples of bad common packages:
- `common` — what's common?
- `util` — utility for what?
- `helpers` — helping whom?
- `models` — models of what?
- `types` — vague.

Bad in that they collect unrelated code. Better names express purpose:
- `httpx` — HTTP extensions.
- `iox` — I/O extensions.
- `loggers` — logger configurations (or `slogutil`).
- `domain` — domain models, but specifically your domain.

### Aggregating sub-packages

When a package gets large, split:

```
order/
  order.go           (Order, OrderItem)
  service.go         (OrderService)
  ...

becomes

order/
  order.go           (Order, OrderItem)
  ...
order/pricing/       (pricing logic split out)
order/discount/      (discounts)
```

Sub-package names extend the parent. `order/pricing` is `pricing` package with logical relationship to `order`.

### Versioning packages

Major versions (`/v2`+) require the suffix in the module path AND in import statements (see `06-packages-modules/06-versioning-and-semver.md`). Use multiple major versions only when truly breaking the API.

For minor changes, expand the API. Don't introduce `myproj/v2` for an additive change.

### Re-exports

You can re-export types from internal packages via the public package:

```go
// In pkg/api/
package api
import "myproj/internal/types"
type User = types.User       // alias re-export (1.24+ generic alias)
var ErrNotFound = types.ErrNotFound
```

Useful for stabilizing a public API while keeping implementation in internal. Use sparingly.

### `init` functions

`init()` runs once per package at startup. Use for:
- Driver registration (`database/sql`).
- Compile regex into a package var (alternative: `var re = regexp.MustCompile(...)`).

Avoid for:
- I/O.
- Configuration loading.
- Anything fallible without good recovery.

Heavy init makes startup slow and surprising.

### Public types vs internal types

```go
// Public — exported through the package
type Result struct {
    ID   string
    Data []byte
}

// Internal — only within package
type cache struct{ /* ... */ }
```

Public types are part of your API forever. Once shipped, you can't remove fields without a major version bump.

### Subpackage hierarchy

Avoid deeply nested package paths:

```
myorg/myproj/internal/foo/bar/baz/qux/  ← too deep
```

If you find yourself nesting beyond 3 levels under `internal/`, consider flattening. Each path segment must justify its existence.

### Documentation as API

Doc comments are part of the API:

```go
// Process processes the input and returns a result.
//
// On error, the returned [Result] is the zero value.
//
// Process is safe for concurrent use.
func Process(input string) (Result, error)
```

Callers depend on documented behavior. Changing behavior is a breaking change even if the signature is unchanged.

### Avoid premature abstraction

Don't introduce an interface for "future flexibility" if there's currently one implementation. Add it when a second arises.

```go
// Premature
type UserRepository interface { ... }
type postgresUserRepository struct { ... }  // only implementation

// Better at first
type UserRepository struct { db *sql.DB }  // concrete

// When a memcached cache implementation appears:
type UserRepository interface { ... }
type PostgresUserRepository struct { ... }
type MemcachedUserRepository struct { ... }
```

Refactoring is cheap; over-engineering is expensive.

### Package layout examples — small project

```
myapp/
  go.mod
  main.go              ← single binary; everything in one package
```

Two-file project? Fine. Don't impose hexagonal layout.

### Package layout examples — medium project

```
myapp/
  go.mod
  cmd/
    server/main.go
    cli/main.go
  internal/
    user/
      user.go
      service.go
      repository.go
    auth/
      auth.go
  go.sum
```

Two binaries, two internal packages. Clear ownership.

### Package layout examples — large project

```
myapp/
  go.mod
  cmd/
    server/, cli/, migrate/, ... (each its own main)
  internal/
    user/, order/, payment/, shipping/, auth/, ... (each cohesive)
  pkg/
    api/, client/, sdk/, ... (public packages)
  go.sum
```

Many internal packages organized by domain. `pkg/` for public APIs.

### When to split a package

Signs a package is too big:
- More than ~30 .go files (excluding tests).
- More than ~5 unrelated concerns inside.
- Doc comment doesn't capture all features.
- Import graph shows it imported by everyone (low cohesion).

Splitting usually involves identifying a sub-domain that has its own cohesion.

### When to merge packages

Signs of over-splitting:
- Many packages with 1-2 files each.
- Excessive cross-package types passed.
- Layering ceremony without rationale.
- Single-developer maintains the whole thing.

Merge when sub-packages are tightly coupled.

## Standard Library Hooks

- `go list -deps ./...` — show package dependency graph.
- `go vet`, `staticcheck`: catch many design issues.
- `go-callvis` (third-party): visualize.
- `internal/` directories: visibility.

## Real-World Patterns

### 1. Cmd + internal layout

```
myapp/
  cmd/
    server/main.go
  internal/
    httpapi/handlers.go
    store/repo.go
```

Single binary; clear internal boundary.

### 2. Domain-driven structure

```
myapp/
  internal/
    user/        (User domain: entity, service, repo)
    order/       (Order domain)
    payment/     (Payment domain)
```

Each subdir is a cohesive bounded context.

### 3. Library project layout

```
mylib/
  go.mod
  lib.go         (public top-level)
  internal/
    impl/      (implementation details)
```

Users import `mylib`; internal details are hidden.

### 4. Multi-binary project

```
myorg/
  go.mod
  cmd/
    api-server/
    background-worker/
    cli/
  internal/
    shared types and helpers
```

Each `cmd/X/main.go` is its own binary. Shared code under `internal/`.

### 5. Re-export for public API stability

```go
// pkg/api/user.go
package api
import "myorg/myproj/internal/user"
type User = user.User
type Service = user.Service
```

API consumers see stable types; implementation can move freely under `internal/`.

## Anti-Patterns & Gotchas

**`util`, `common`, `helpers`** packages. Use descriptive names.

**One package per noun.** Group related domain code.

**Cyclic imports.** Indicates layering bug.

**Mutual dependencies.** Move shared types out.

**Premature interfaces.** Wait for a second implementation.

**Heavy `init()`.** Fragile.

**`internal/` ignored.** Use it for everything not in your public API.

**Deep nesting** (`internal/foo/bar/baz/qux/`). Flatten.

**Package per file.** Group related code.

**Stuttering** (`bytes.BytesBuffer`). Drop the prefix.

**Exporting everything.** Default to unexported.

**Letting docs go stale.** Doc comments are part of the API.

## Performance Notes

Package design doesn't typically affect runtime performance. Some compile-time effects:
- More packages → more parallel compilation (faster for big projects).
- Fewer packages → smaller compile cache footprint.
- Tightly-coupled packages with cross-package inlining benefit from co-location.

Practically: package design is a readability and maintainability concern.

## How Big Companies Use It

- **Kubernetes**: massive `internal/`, `cmd/`, `pkg/`, `staging/`, structured by domain.
- **CockroachDB**: hundreds of packages, organized by SQL/KV/storage layers.
- **HashiCorp**: `command/`, `vault/`, etc.; cmd-vs-internal pattern.
- **Tailscale**: small `cmd/`, large `tsnet/`, `wgengine/`, `magicsock/`.
- **Caddy**: modular plugin packages; central `caddy/` core.

The pattern: structure follows the domain; `internal/` is universal.

## Source Code References

- Effective Go § Package Names: https://go.dev/doc/effective_go#package-names.
- Andrew Gerrand, "Package names": https://go.dev/blog/package-names.
- `golang-standards/project-layout`: https://github.com/golang-standards/project-layout (community; not Go-team endorsed).
- Bill Kennedy, "Package-oriented design": https://www.ardanlabs.com/blog/2017/02/package-oriented-design.html.

## Further Reading

- Andrew Gerrand, "Package names" Go blog.
- Bill Kennedy, "Package-Oriented Design" — Ardan Labs.
- Mat Ryer, "Structuring applications in Go" — GopherCon talks.
- "Standard Go Project Layout" — github.com/golang-standards/project-layout (debate it).
- Dave Cheney, "Practical Go" lecture (package design section).
- "Go Programming Language" — Donovan & Kernighan (chapter on packages).
- Discussion threads on golang-nuts about pkg/ vs internal/.

## Exercises / Self-Check

1. Audit your project's package boundaries. Identify a `util`, `common`, or `helpers` package and propose a better split.
2. Find an exported identifier that nobody uses outside the package. Should it be exported?
3. Identify a cyclic-import attempt in your code (or imagine one). Show how to resolve via a third package or consumer-side interface.
4. Argue both sides of `pkg/` vs no-`pkg/`. Which would you adopt for a new public library?
5. Take a package that has grown to 50 files. Propose a split into two cohesive packages. What's the criterion?
