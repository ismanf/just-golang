# Repository Pattern

## TL;DR

The **repository pattern** hides persistence behind a domain-shaped interface. A `UserRepository` exposes methods like `Get(ctx, id)`, `Save(ctx, u)`, `Find(ctx, query)` — and *not* SQL strings or DB driver types. Business logic depends on the interface; persistence details live in the implementation. Originated in Eric Evans' *Domain-Driven Design* (2003), it remains useful in Go for **decoupling tests from a real database**, **swapping storage technologies**, and **keeping domain code free of `database/sql` noise**. The single biggest gotcha: **don't overdo it**. A repository that re-implements every SQL operation as a method (`FindByIDOrderByDateLimitTen`) is worse than just writing SQL. Use the pattern when you have *behavior* (transactions, caching, validation) to encapsulate.

## Mental Model

```
   ┌────────────────────────────────────────────────────────────┐
   │  Domain layer (no DB knowledge)                              │
   │                                                              │
   │  type UserService struct { repo UserRepository }             │
   │                                                              │
   │  func (s *UserService) Register(ctx, u User) error {         │
   │      // business rules                                       │
   │      return s.repo.Save(ctx, u)                              │
   │  }                                                           │
   └─────────────────┬──────────────────────────────────────────┘
                     ▼ depends on
   ┌────────────────────────────────────────────────────────────┐
   │  Interface (declared in domain or in a "ports" package)      │
   │                                                              │
   │  type UserRepository interface {                             │
   │      Get(ctx, id) (User, error)                              │
   │      Save(ctx, User) error                                   │
   │      FindByEmail(ctx, email) (User, error)                   │
   │  }                                                           │
   └─────────────────┬──────────────────────────────────────────┘
                     ▲ implemented by
   ┌────────────────────────────────────────────────────────────┐
   │  Adapter (knows about persistence tech)                      │
   │                                                              │
   │  type postgresUserRepo struct { db *sql.DB }                 │
   │  func (r *postgresUserRepo) Get(...) (User, error) {...}     │
   │  func (r *postgresUserRepo) Save(...) error { ... }          │
   └────────────────────────────────────────────────────────────┘
```

Domain knows nothing about `database/sql`; the adapter knows nothing about business rules.

## Syntax & Basic Usage

```go
package user

import "context"

type User struct {
	ID    string
	Name  string
	Email string
}

// Repository — the domain's port.
type Repository interface {
	Get(ctx context.Context, id string) (User, error)
	Save(ctx context.Context, u User) error
	FindByEmail(ctx context.Context, email string) (User, error)
}

// Service — uses Repository; doesn't know how it persists.
type Service struct {
	repo Repository
}

func NewService(repo Repository) *Service { return &Service{repo: repo} }

func (s *Service) Register(ctx context.Context, name, email string) (User, error) {
	if _, err := s.repo.FindByEmail(ctx, email); err == nil {
		return User{}, ErrEmailTaken
	}
	u := User{ID: newID(), Name: name, Email: email}
	if err := s.repo.Save(ctx, u); err != nil {
		return User{}, err
	}
	return u, nil
}

var ErrEmailTaken = errors.New("email already in use")
func newID() string { return "u_" + /* ... */ "" }
```

```go
package userpg

import (
	"context"
	"database/sql"

	"myproj/user"
)

type Repo struct{ db *sql.DB }

func New(db *sql.DB) *Repo { return &Repo{db: db} }

func (r *Repo) Get(ctx context.Context, id string) (user.User, error) {
	var u user.User
	err := r.db.QueryRowContext(ctx,
		"SELECT id, name, email FROM users WHERE id=$1", id,
	).Scan(&u.ID, &u.Name, &u.Email)
	return u, err
}

func (r *Repo) Save(ctx context.Context, u user.User) error {
	_, err := r.db.ExecContext(ctx,
		`INSERT INTO users (id, name, email)
		 VALUES ($1, $2, $3)
		 ON CONFLICT (id) DO UPDATE SET name=$2, email=$3`,
		u.ID, u.Name, u.Email,
	)
	return err
}

func (r *Repo) FindByEmail(ctx context.Context, email string) (user.User, error) {
	var u user.User
	err := r.db.QueryRowContext(ctx,
		"SELECT id, name, email FROM users WHERE email=$1", email,
	).Scan(&u.ID, &u.Name, &u.Email)
	return u, err
}
```

Wiring in `main`:

```go
db, _ := sql.Open("postgres", dsn)
repo := userpg.New(db)
svc := user.NewService(repo)
```

The domain (`user.Service`) sees only `Repository`. Tests can pass a `mockRepo`.

## Deep Dive

### Where to put the interface

Two conventions:

**Domain owns the interface** (recommended):

```
user/
  user.go            // domain types
  service.go         // Service
  repository.go      // type Repository interface { ... }
userpg/
  repo.go            // implements user.Repository
```

The domain declares what it needs; adapters satisfy.

**"Ports" package** (hexagonal style):

```
ports/
  user_repository.go  // type UserRepository interface
```

Convenient when many adapters import a shared interfaces package. See `19-patterns/06-clean-hexagonal.md`.

The Go idiom **"accept interfaces, return structs"** says: define the interface where it's *consumed* (the domain), implement (return a concrete struct) where it's *provided* (the adapter).

### Don't expose persistence types

```go
// BAD: leaks sql.Rows
type Repository interface {
    Query(ctx context.Context, where string) (*sql.Rows, error)
}

// GOOD: returns domain values
type Repository interface {
    FindActive(ctx context.Context) ([]User, error)
}
```

If `*sql.Rows` appears in your interface, you've leaked persistence into domain. Same for ORM types, transaction handles, etc.

### Transactions

Naive: pass a `*sql.Tx` through methods. Couples domain to `database/sql`.

**Better**: define a `Tx` interface or use the **Unit of Work** pattern.

```go
type Tx interface {
    Commit() error
    Rollback() error
}

type Repository interface {
    BeginTx(ctx context.Context) (Tx, error)
    SaveTx(tx Tx, u User) error
    GetTx(tx Tx, id string) (User, error)
}
```

Or, **pass the transaction via context**:

```go
type txKey struct{}

func WithTx(ctx context.Context, tx *sql.Tx) context.Context {
    return context.WithValue(ctx, txKey{}, tx)
}

func (r *Repo) Save(ctx context.Context, u User) error {
    if tx, ok := ctx.Value(txKey{}).(*sql.Tx); ok {
        // use tx
    }
    // use r.db
    return nil
}
```

The service can wrap calls in a transaction without each repository method having a `tx` parameter.

### Domain errors

Return *domain-shaped* errors, not driver-specific ones:

```go
var (
    ErrNotFound      = errors.New("not found")
    ErrEmailTaken    = errors.New("email already in use")
)

func (r *Repo) Get(ctx, id) (User, error) {
    var u User
    err := r.db.QueryRowContext(...).Scan(&u.ID, &u.Name)
    if err == sql.ErrNoRows {
        return User{}, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    return u, err
}
```

Callers test via `errors.Is(err, user.ErrNotFound)`. They never see `sql.ErrNoRows`.

### Query objects

Instead of dozens of `FindBy*` methods, expose a query struct:

```go
type Query struct {
    Name     string
    MinAge   int
    OrderBy  string
    Limit    int
}

type Repository interface {
    Search(ctx context.Context, q Query) ([]User, error)
}
```

Adapter translates the Query into SQL. Domain stays small.

### Caching

Wrap the repository:

```go
type cachedRepo struct {
    next  Repository
    cache *lru.Cache
}

func (c *cachedRepo) Get(ctx context.Context, id string) (User, error) {
    if u, ok := c.cache.Get(id); ok { return u.(User), nil }
    u, err := c.next.Get(ctx, id)
    if err == nil { c.cache.Add(id, u) }
    return u, err
}
```

Decorate any `Repository` with caching. Service doesn't care.

### Testing

```go
type mockRepo struct {
    users map[string]User
}

func (m *mockRepo) Get(_ context.Context, id string) (User, error) {
    u, ok := m.users[id]
    if !ok { return User{}, ErrNotFound }
    return u, nil
}

func TestRegister(t *testing.T) {
    repo := &mockRepo{users: map[string]User{}}
    svc := NewService(repo)
    u, err := svc.Register(context.Background(), "Alice", "a@example.com")
    require.NoError(t, err)
    require.NotEmpty(t, u.ID)
}
```

In-memory fakes are usually preferred to mocking frameworks; they're faster and more readable.

### Anti-pattern: anemic interface

```go
type Repository interface {
    DB() *sql.DB
}
```

You've just renamed `*sql.DB`. Use the actual type.

### Generic repository (1.18+)

Sometimes useful for CRUD:

```go
type CRUD[T any, ID comparable] interface {
    Get(ctx context.Context, id ID) (T, error)
    Save(ctx context.Context, item T) error
    Delete(ctx context.Context, id ID) error
}
```

A pure CRUD type benefits from generics. Domain-specific methods (`FindByEmail`) need concrete types.

Russ Cox's "when in doubt, don't" applies: introduce generics only when there's at least two real implementations.

### Repository vs ORM

ORMs (GORM, Ent) provide automatic CRUD generation. Pros: less boilerplate. Cons: hide SQL, awkward joins, complex queries fight the model.

Many Go teams use a hybrid: ORM for trivial CRUD, repository pattern for anything non-trivial.

Or **sqlc**: generates type-safe Go from SQL. The generated code *is* a kind of repository. See `16-data/04-sqlx-and-sqlc.md`.

### Aggregate roots

Domain-Driven Design's strict version: a repository is per *aggregate root* (a clustered entity that's edited together). E.g., `OrderRepository` handles the Order and its OrderItems atomically.

In Go this maps to: one repository per top-level entity that has its own consistency boundary.

### Pagination

```go
type Page struct {
    Limit  int
    Offset int
    Cursor string  // alternative to offset
}

type Repository interface {
    List(ctx context.Context, p Page) ([]User, NextPage, error)
}
```

Cursor-based pagination scales better; offset-based has known issues at scale.

### Bulk operations

```go
type Repository interface {
    SaveMany(ctx context.Context, users []User) error
    GetMany(ctx context.Context, ids []string) ([]User, error)
}
```

For high-volume work, batched APIs beat per-item calls.

## Standard Library Hooks

- `database/sql`: `*sql.DB`, `*sql.Tx`, `Rows.Scan`.
- `context.Context`: pervasive.
- `errors.Is`, `errors.As`: domain error testing.

## Real-World Patterns

### 1. Decorate with caching

```go
func NewCachedRepo(next Repository, size int) Repository {
    c, _ := lru.New[string, User](size)
    return &cachedRepo{next: next, cache: c}
}

type cachedRepo struct {
    next  Repository
    cache *lru.Cache[string, User]
}

func (c *cachedRepo) Get(ctx context.Context, id string) (User, error) {
    if u, ok := c.cache.Get(id); ok { return u, nil }
    u, err := c.next.Get(ctx, id)
    if err == nil { c.cache.Add(id, u) }
    return u, err
}

func (c *cachedRepo) Save(ctx context.Context, u User) error {
    if err := c.next.Save(ctx, u); err != nil { return err }
    c.cache.Add(u.ID, u)
    return nil
}
```

Decorator composes; service is unchanged.

### 2. Repository with Unit of Work

```go
type UnitOfWork struct {
    db *sql.DB
}

func (uow *UnitOfWork) Do(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := uow.db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()

    ctx = WithTx(ctx, tx)
    if err := fn(ctx); err != nil { return err }
    return tx.Commit()
}

// Service uses it:
err := uow.Do(ctx, func(ctx context.Context) error {
    if err := orders.Save(ctx, o); err != nil { return err }
    return payments.Charge(ctx, o.ID, o.Total)
})
```

Multiple repo calls in a single transaction; rollback on error.

### 3. Multiple adapters

```go
// userpg/repo.go: postgres impl
// userredis/repo.go: redis impl (for read-only cache scenario)
// usermem/repo.go: in-memory for tests
// usermock/repo.go: testify-style mock for unit tests
```

The interface lives once in `user/repository.go`; multiple implementations.

### 4. Soft delete via repository

```go
type Repository interface {
    Get(ctx context.Context, id string) (User, error)  // skips soft-deleted
    GetIncludingDeleted(ctx context.Context, id string) (User, error)
    Delete(ctx context.Context, id string) error      // soft-delete
    Purge(ctx context.Context, id string) error      // hard-delete
}
```

Domain decides the policy; adapter implements (e.g., `WHERE deleted_at IS NULL`).

### 5. sqlc-generated repository

```go
// db/queries.sql:
// -- name: GetUser :one
// SELECT * FROM users WHERE id = $1;

// sqlc generates:
func (q *Queries) GetUser(ctx context.Context, id string) (User, error) {
    row := q.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", id)
    var u User
    err := row.Scan(&u.ID, &u.Name, &u.Email)
    return u, err
}

// You wrap *Queries in your repository.
type Repo struct{ q *Queries }
func (r *Repo) Get(ctx context.Context, id string) (User, error) { return r.q.GetUser(ctx, id) }
```

Type-safe SQL; manual wrapping into the Repository interface keeps the seam.

## Anti-Patterns & Gotchas

**Generic CRUD interface for everything.** A 30-method interface is a code smell; usually means the abstraction is wrong.

**`FindByXAndYAndZ` method explosion.** Use a Query struct; let the adapter compose SQL.

**Leaking persistence types** (`*sql.Tx`, ORM models). Domain shouldn't know.

**Passing `*sql.DB` into the service.** Defeats the purpose.

**Interface with a single implementation that you'll never replace.** Just use the concrete struct. The seam only matters where you need it.

**Implementing the same query in many adapters with subtle drift.** Use sqlc or shared SQL.

**Mocking instead of using in-memory fakes.** Fakes are usually faster and more honest. Mock only what's truly hard to fake.

**Repository methods that take huge filter structs.** Probably indicates the domain wants a *query DSL*, not a repository.

**Domain code that calls `errors.Is(err, sql.ErrNoRows)`.** Domain should test domain errors only.

**Repository per query.** That's not a repository; it's a function. Group related queries.

## Performance Notes

- Interface method dispatch: ~1-2 ns. Negligible.
- Reflection-based ORM: variable; some queries 10-100× slower than raw SQL.
- sqlc: zero overhead; codegen produces direct DB calls.
- Decorator chain (cache → real): adds ~ns per call.

Repository is rarely the performance bottleneck. Premature optimization here is wasted.

## How Big Companies Use It

- **Uber**: heavily uses repository pattern + interfaces.
- **Stripe Go SDK**: kind-of-repository per resource (`stripe.Customer`, etc.).
- **CockroachDB internal**: less explicit; their tables-as-Go-structs predates the broader Go community's adoption.
- **Microsoft / Azure SDKs**: every resource has a "client" similar to a repository.
- **Most enterprise Go shops**: repository per aggregate root.
- **Many YC startups**: hybrid: sqlc + thin repository wrapper.

## Source Code References

The pattern is conceptual. References:

- `database/sql.DB` design as the lowest-level adapter: [`src/database/sql/`](https://github.com/golang/go/tree/release-branch.go1.26/src/database/sql).
- sqlc (generates repository-like code): [`sqlc-dev/sqlc`](https://github.com/sqlc-dev/sqlc).
- GORM (full ORM): [`go-gorm/gorm`](https://github.com/go-gorm/gorm).
- Ent (graph ORM): [`ent/ent`](https://github.com/ent/ent).
- Kubernetes' "lister" pattern is repository-like: [`kubernetes/client-go/listers/`](https://github.com/kubernetes/client-go/tree/master/listers).

## Further Reading

- Eric Evans, *Domain-Driven Design* (book) — original definition.
- Vaughn Vernon, *Implementing Domain-Driven Design* — practical patterns.
- Martin Fowler, "Repository" (P of EAA): https://martinfowler.com/eaaCatalog/repository.html.
- "Repository pattern in Go" — multiple community posts.
- "Beyond CRUD: building good Go data layer" — talks at GopherCon.
- "Design patterns: repository" — Refactoring Guru (language-agnostic).
- Mat Ryer, "Go programming with patterns" — talks.

## Exercises / Self-Check

1. Define a `Repository` interface for an `Order` entity. Include `Get`, `Save`, `FindByCustomer`, and `Cancel`. Why doesn't `Cancel` belong in the repository?
2. Implement the same repository for SQL and for in-memory. Confirm a test passes against both.
3. Add a caching decorator. What invalidation strategy do you use?
4. Implement Unit of Work that propagates a `*sql.Tx` via context. Show two repository calls inside one transaction.
5. Critique: a 50-method `UserRepository`. What does this likely indicate about the domain design?
