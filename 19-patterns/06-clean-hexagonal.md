# Clean / Hexagonal Architecture — Ports and Adapters in Go

## TL;DR

**Hexagonal architecture** (Alistair Cockburn, 2005), also known as **ports and adapters**, structures an application around a central **domain core** that depends on nothing external; **ports** (interfaces) define what the domain needs; **adapters** (implementations) provide it for specific technologies (HTTP, SQL, Kafka, etc.). **Clean Architecture** (Uncle Bob, 2012) is essentially the same idea with concentric circles instead of a hexagon. In Go, the pattern translates to: **domain package imports nothing except stdlib**, **adapter packages import the domain and the tech**, **`main` wires them**. The single biggest gotcha: **dogmatic implementations are expensive in Go** — you end up with dozens of one-method interfaces, manually-written DTOs, and translation layers between layers. Go's "accept interfaces, return structs" idiom *naturally* produces hexagonal-flavored code; explicit "use cases" and "entity" packages often add ceremony without adding clarity.

## Mental Model

```
                    ┌─────────────────────────────┐
                    │       Domain (core)          │
                    │   - business rules           │
                    │   - entities + value objects │
                    │   - use cases / services     │
                    │   - port interfaces          │
                    │   imports: stdlib only       │
                    └──────┬──────────────────────┘
                           │ depends on (via interfaces)
   ┌─────────────────┬─────┴──────┬────────────────────┐
   ▼                 ▼            ▼                    ▼
   HTTP adapter   gRPC adapter   SQL adapter         Email adapter
   (driving)      (driving)      (driven)            (driven)
   
   - "Driving" adapters initiate work in the domain (UI, RPC).
   - "Driven" adapters are tools the domain uses (DB, email, S3).
   - All depend on the domain; the domain depends on none of them.
```

Dependencies always point **inward**. The domain is unaware of the outside world.

## Syntax & Basic Usage

A small project structured hexagonally:

```
myproj/
├── cmd/server/main.go        ← wiring
├── internal/
│   ├── user/                 ← domain core
│   │   ├── user.go           ← entity
│   │   ├── service.go        ← use cases
│   │   └── ports.go          ← interfaces (UserRepository, EmailSender)
│   ├── adapters/
│   │   ├── userpg/repo.go    ← SQL adapter (driven)
│   │   ├── userhttp/api.go   ← HTTP adapter (driving)
│   │   └── usersmtp/email.go ← email adapter (driven)
└── go.mod
```

```go
// internal/user/user.go
package user

type User struct {
	ID    string
	Email string
	Name  string
}
```

```go
// internal/user/ports.go
package user

import "context"

// Driven port: the domain needs these capabilities.
type Repository interface {
	Get(ctx context.Context, id string) (User, error)
	Save(ctx context.Context, u User) error
}

type EmailSender interface {
	SendWelcome(ctx context.Context, u User) error
}
```

```go
// internal/user/service.go
package user

import (
	"context"
	"errors"
)

type Service struct {
	repo  Repository
	email EmailSender
}

func NewService(repo Repository, email EmailSender) *Service {
	return &Service{repo: repo, email: email}
}

func (s *Service) Register(ctx context.Context, email, name string) (User, error) {
	if email == "" { return User{}, errors.New("email required") }
	u := User{ID: newID(), Email: email, Name: name}
	if err := s.repo.Save(ctx, u); err != nil { return User{}, err }
	if err := s.email.SendWelcome(ctx, u); err != nil { /* log but don't fail */ }
	return u, nil
}

func newID() string { return "u_" + /* ... */ "" }
```

```go
// internal/adapters/userhttp/api.go (driving adapter)
package userhttp

import (
	"encoding/json"
	"net/http"

	"myproj/internal/user"
)

type API struct{ svc *user.Service }

func New(svc *user.Service) *API { return &API{svc: svc} }

func (a *API) Register(w http.ResponseWriter, r *http.Request) {
	var req struct{ Email, Name string }
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), 400)
		return
	}
	u, err := a.svc.Register(r.Context(), req.Email, req.Name)
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}
	json.NewEncoder(w).Encode(u)
}
```

```go
// cmd/server/main.go
package main

import (
	"database/sql"
	"net/http"

	"myproj/internal/adapters/userhttp"
	"myproj/internal/adapters/userpg"
	"myproj/internal/adapters/usersmtp"
	"myproj/internal/user"
)

func main() {
	db, _ := sql.Open("postgres", "...")
	repo := userpg.New(db)
	email := usersmtp.New("smtp.example.com")
	svc := user.NewService(repo, email)
	api := userhttp.New(svc)

	http.HandleFunc("/users", api.Register)
	http.ListenAndServe(":8080", nil)
}
```

`main` wires everything; nothing in `internal/user` imports `database/sql` or `net/smtp`.

## Deep Dive

### Why hexagonal works in Go

Two language properties make hex-style natural:

1. **No inheritance**: composition forces explicit dependencies.
2. **Interfaces are satisfied implicitly**: adapter packages don't need to declare "implements X" — the compiler figures it out.

Consequence: the domain declares interfaces in its own package; adapters just have to match shape. No coupling-by-import in the wrong direction.

### Ports

A **port** is a domain interface describing what the domain needs.

- **Driven ports** (used by domain): repositories, senders, clocks, randomness.
- **Driving ports** (call domain): use case interfaces consumed by handlers.

In small Go projects, the driving "port" is often just `Service`'s public methods — no separate interface needed. Driven ports are interfaces in the domain package.

### Where to put port interfaces

Convention varies. Two common styles:

**Domain package owns it**: `internal/user/ports.go` defines `Repository`. `internal/adapters/userpg` imports `internal/user` and implements.

**Separate `ports` package**: `internal/ports/user.go`. Both domain and adapter import `ports`.

The first is more idiomatic Go ("accept interfaces, return structs" — define the interface where you *consume*, not where you *implement*).

### Adapters

An **adapter** implements a port (driven) or invokes domain use cases (driving). Adapters can depend on:
- The domain package (to know types and interfaces).
- The tech package (`database/sql`, `net/http`, ...).

They must **not** depend on each other. The HTTP adapter doesn't know about the SQL adapter.

### Entities and value objects

**Entity**: has identity (`User`, `Order`). Two users with the same name are different.

**Value object**: defined by attributes (`Money{Amount, Currency}`). Two `Money{100, USD}` are equal.

In Go, entities are usually `struct` with an `ID`; value objects are `struct` (compared with `==`).

### Use cases / interactors

Uncle Bob's Clean Architecture introduces explicit "use case" types:

```go
type RegisterUseCase struct {
    repo  user.Repository
    email user.EmailSender
}

func (uc *RegisterUseCase) Execute(ctx context.Context, req RegisterRequest) (RegisterResponse, error) {
    // ...
}
```

In Go, a `Service` with several methods often serves the same purpose. The Service-with-methods style is more idiomatic; explicit use-case types appear in larger codebases.

### Driving adapters: HTTP, gRPC, CLI, message consumer

```
HTTP adapter:   internal/adapters/userhttp/api.go
gRPC adapter:   internal/adapters/usergrpc/server.go
CLI adapter:    internal/adapters/usercli/cmd.go
Kafka consumer: internal/adapters/userkafka/consumer.go
```

All call into the same `user.Service`. The same domain logic powers HTTP, gRPC, CLI, async consumers — write once, expose many ways.

### Driven adapters: SQL, Redis, S3, email, SMS, telemetry

```
internal/adapters/userpg/      → PostgreSQL repo
internal/adapters/userredis/   → Redis cache
internal/adapters/usermem/     → in-memory (for tests)
internal/adapters/usersmtp/    → SMTP email
internal/adapters/usersms/     → SMS via Twilio
internal/adapters/usermetric/  → Prometheus metrics
```

Each adapter implements the relevant port from `internal/user`.

### Test doubles

Hexagonal architecture makes tests trivial: instantiate the domain with fake adapters.

```go
type fakeRepo struct{ users map[string]user.User }
func (f *fakeRepo) Get(_ context.Context, id string) (user.User, error) { /* ... */ }
func (f *fakeRepo) Save(_ context.Context, u user.User) error { f.users[u.ID] = u; return nil }

type fakeEmail struct{ sent []user.User }
func (f *fakeEmail) SendWelcome(_ context.Context, u user.User) error {
    f.sent = append(f.sent, u); return nil
}

func TestRegister(t *testing.T) {
    repo := &fakeRepo{users: map[string]user.User{}}
    email := &fakeEmail{}
    svc := user.NewService(repo, email)
    u, err := svc.Register(context.Background(), "a@example.com", "Alice")
    require.NoError(t, err)
    require.Len(t, email.sent, 1)
    require.Equal(t, u.ID, repo.users[u.ID].ID)
}
```

Fast, deterministic, no I/O.

### Dogma and pragmatism

Strict hexagonal/Clean implementations sometimes go overboard:

- **DTOs at every boundary**: domain entity → HTTP DTO → JSON. Triple representation of the same data. In Go, returning the entity from a service and JSON-encoding it directly is usually fine.
- **"Mappers" between layers**: `func mapHTTPToDomain(...)` per type. Often more code than the actual logic.
- **"Interactors" for every use case**: a function would do.
- **Each entity gets its own folder**: 100 files for 30 entities. Verbose.

Use the parts that help; skip the parts that don't.

### The Go-idiomatic version

A common Go interpretation:

```
internal/
  user/                ← all domain code for user (entity + service + port interfaces)
  order/               ← all domain code for order
  adapters/
    postgres/          ← adapters for all entities
    httpapi/           ← HTTP API for all entities
    smtp/
cmd/server/main.go     ← wire
```

Or even simpler — flat packages with `_test.go` separating concerns. Don't impose hexagonal where a simpler structure works.

### When hexagonal pays off

- Multi-year codebases with rotating team members.
- Domains that get re-skinned: same logic, new transport (REST → gRPC → GraphQL).
- Heavy testing requirements (fast unit tests covering many scenarios).
- Multiple persistence backends.

When it doesn't pay off:

- Small services (<5k LOC).
- Throwaway internal tools.
- Read-mostly APIs over a single database.

### Clean Architecture circles

Uncle Bob's diagram:

```
   ┌─────────────────────────────────────────────────────┐
   │ Frameworks & Drivers (web, DB, devices)              │
   │   ┌──────────────────────────────────────────┐      │
   │   │ Interface Adapters (controllers, gateways)│      │
   │   │   ┌─────────────────────────────────┐    │      │
   │   │   │ Application Business Rules       │    │      │
   │   │   │ (use cases)                      │    │      │
   │   │   │   ┌────────────────────┐         │    │      │
   │   │   │   │ Enterprise Business│         │    │      │
   │   │   │   │ Rules (entities)   │         │    │      │
   │   │   │   └────────────────────┘         │    │      │
   │   │   └─────────────────────────────────┘    │      │
   │   └──────────────────────────────────────────┘      │
   └─────────────────────────────────────────────────────┘
```

Same idea as hexagon: inner depends on nothing outer.

### Onion architecture, Clean Architecture, hexagonal

All express the same principle: "dependency inversion: outer layers depend on inner layers, not vice versa". The naming differs; the engineering essentially doesn't.

## Standard Library Hooks

- `context.Context`: pervasive across layers.
- `errors.Is`/`errors.As`: cross-layer error inspection without leaking types.
- `database/sql`, `net/http`, `net/smtp`: in adapter packages only.
- `internal/`: prevents external imports of domain or adapters.

## Real-World Patterns

### 1. Multi-transport same domain

```go
// HTTP adapter
http.HandleFunc("/users", userHTTP.Register)

// gRPC adapter
userpb.RegisterUserServiceServer(srv, userGRPC.New(svc))

// CLI adapter
cmd := &cobra.Command{Use: "register", RunE: userCLI.New(svc).Register}
```

Same `user.Service`; multiple driving adapters.

### 2. Caching adapter wraps real adapter

```go
type CachingRepo struct {
    next user.Repository
    cache map[string]user.User
    mu    sync.RWMutex
}

func (c *CachingRepo) Get(ctx context.Context, id string) (user.User, error) {
    c.mu.RLock()
    if u, ok := c.cache[id]; ok { c.mu.RUnlock(); return u, nil }
    c.mu.RUnlock()
    u, err := c.next.Get(ctx, id)
    if err == nil {
        c.mu.Lock()
        c.cache[id] = u
        c.mu.Unlock()
    }
    return u, err
}
```

Composition of adapters; `CachingRepo` is a Repository.

### 3. Test with all in-memory

```go
func setup(t *testing.T) *user.Service {
    return user.NewService(&fakeRepo{users: map[string]user.User{}}, &fakeEmail{})
}

func TestRegister_Happy(t *testing.T) { /* ... */ }
func TestRegister_EmptyEmail(t *testing.T) { /* ... */ }
// 100s of cases run in milliseconds.
```

### 4. Switch backend in main

```go
var repo user.Repository
switch cfg.Backend {
case "postgres":
    repo = userpg.New(db)
case "redis":
    repo = userredis.New(rdb)
case "memory":
    repo = usermem.New()
}
svc := user.NewService(repo, emailer)
```

Configurable backends; domain unchanged.

### 5. Outgoing webhook as an adapter

```go
type WebhookSender interface {
    SendUserRegistered(ctx context.Context, u user.User) error
}

// Adapters: HTTP webhook, Kafka publisher, no-op.
```

External integrations are adapters too; domain triggers, adapter delivers.

## Anti-Patterns & Gotchas

**Interface per type "for testability"**. Most types don't need interfaces. Use them only at *seams* — where a fake helps.

**DTO mappers for every layer.** Domain → service → HTTP: three structs for the same thing. In Go, often the domain struct can be the API response directly.

**Use case structs for trivial actions.** `RegisterUserUseCase{}` with one method called by one place is overkill.

**Folders-per-entity for tiny entities.** 50 folders for an app with 5 features.

**Domain types with persistence tags.** `User struct { ID string \`db:"id" json:"id"\` }` mixes layers. Some teams accept this; purists don't.

**Adapter packages importing each other.** They should compose only at `main`. Adapter A doesn't know B exists.

**Domain packages importing `net/http`.** A red flag.

**Trying to abstract `time.Now` via a port "for tests"**. Use [`testing/synctest`](https://pkg.go.dev/testing/synctest) (1.25+) or a small `Clock` interface only where it matters.

**Layering for layering's sake.** Each abstraction should pay for itself.

**Re-introducing globals in main.** `main.go` should explicitly wire, not lazy-init globals.

## Performance Notes

Hexagonal is a structural pattern; per-call overhead is interface dispatch (~1-2 ns) and one extra allocation per port-crossing struct (sometimes). Negligible for almost all workloads.

Domain code (the pure logic) is usually the easiest to optimize because it has no I/O. Adapters can be tuned independently (caching, batching, sync.Pool) without affecting domain logic.

## How Big Companies Use It

- **Uber**: many Go services follow hexagonal-style with `pkg/`, `cmd/`, `internal/`.
- **DDDD-adopting companies** (Mercari, GoCardless, Monzo): explicit ports/adapters in Go.
- **CockroachDB**: layered but not strictly hexagonal; the storage and SQL layers have clear boundaries.
- **HashiCorp Vault**: storage backends are adapters behind a Storage interface.
- **Kubernetes**: not strictly hexagonal but apiserver has clear ports for storage, admission, etc.
- **Stripe, Square**: business logic decoupled from web/RPC layers.
- **Many YC fintech startups**: hexagonal-style is common in financial domains.

## Source Code References

The pattern is structural; references illustrate the layout:

- Mat Ryer's example: [`matryer/moq`](https://github.com/matryer/moq) — interface mocks (the "ports" your domain needs).
- A canonical hexagonal Go example: [`bxcodec/go-clean-arch`](https://github.com/bxcodec/go-clean-arch).
- Wild Workouts (DDD example): [`ThreeDotsLabs/wild-workouts-go-ddd-example`](https://github.com/ThreeDotsLabs/wild-workouts-go-ddd-example).
- Bookhouse hexagonal example: [`grpc-ecosystem/go-grpc-middleware`](https://github.com/grpc-ecosystem/go-grpc-middleware) (the middleware portion is "adapters").

## Further Reading

- Alistair Cockburn, "Hexagonal Architecture" (2005): https://alistair.cockburn.us/hexagonal-architecture/.
- Robert C. Martin, *Clean Architecture* (book, 2017).
- Eric Evans, *Domain-Driven Design* (book, 2003).
- Vaughn Vernon, *Implementing Domain-Driven Design* (book).
- Three Dots Labs, "Combining DDD, CQRS, and Clean Architecture": https://threedots.tech/post/.
- "Hexagonal Architecture, There Are Always Two Sides To Every Story" — Pablo Picouto Garcia.
- "Go and Domain-Driven Design": many community write-ups.
- Mat Ryer, "Structuring applications in Go" — GopherCon talks.

## Exercises / Self-Check

1. Restructure a small Go service into clear `domain/`, `adapters/`, `cmd/` layers. Identify each adapter's port.
2. Argue whether mapping domain entities to HTTP DTOs is worth the code. Pick a real example in your codebase.
3. Replace one production adapter (e.g., SQL repository) with a test fake in a unit test. How fast does the test run?
4. Could you swap PostgreSQL for SQLite in your hexagonal service without changing domain code? What would have to change?
5. When does hexagonal architecture cost more than it saves? Identify a project where you'd not use it.
