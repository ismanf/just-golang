# Dependency Injection — Manual, Wire, Fx

## TL;DR

Go has **no DI framework in the stdlib**, and the idiomatic style is **manual constructor injection** — pass dependencies as constructor arguments, hold them as struct fields, use them in methods. For larger projects two frameworks dominate: **Wire** (Google, *compile-time* code-generation; no reflection at runtime) and **Fx** (Uber, *runtime* graph resolution via reflection). Both produce a wired object graph; both let you compose modules. The single biggest gotcha: **manual DI scales further than people expect**. Most Go services that adopt a framework do so for *application lifecycle* (startup ordering, shutdown sequencing) rather than for wiring per se. Reach for Fx when your `main` package becomes a thousand-line constructor; reach for Wire when you want zero runtime cost.

## Mental Model

```
   Manual DI (always works):
   
   func main() {
       cfg := loadConfig()
       log := slog.New(...)
       db := mustOpen(cfg.DB)
       repo := store.New(db)
       svc := service.New(repo, log)
       srv := http.NewServer(svc)
       srv.ListenAndServe()
   }

   Wire (compile-time):
   
   //+build wireinject
   func InitializeApp() (*App, error) {
       wire.Build(loadConfig, slog.New, mustOpen, store.New, service.New, http.NewServer, NewApp)
       return nil, nil  // wire generates wire_gen.go that fills this in
   }

   Fx (runtime):
   
   fx.New(
       fx.Provide(loadConfig, slog.New, mustOpen, store.New, service.New, http.NewServer),
       fx.Invoke(func(*http.Server) {}),
   ).Run()
```

All three express the same dependency graph; the difference is *where* the wiring happens and *what infrastructure* helps you.

## Syntax & Basic Usage

Manual DI:

```go
package main

import (
	"context"
	"database/sql"
	"log/slog"
	"net/http"
	"os"
)

type Config struct{ DSN string }

type UserRepo struct{ db *sql.DB }
func NewUserRepo(db *sql.DB) *UserRepo { return &UserRepo{db: db} }

type UserService struct {
	repo *UserRepo
	log  *slog.Logger
}
func NewUserService(repo *UserRepo, log *slog.Logger) *UserService {
	return &UserService{repo: repo, log: log}
}

func main() {
	ctx := context.Background()
	cfg := Config{DSN: os.Getenv("DSN")}
	log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	db, err := sql.Open("postgres", cfg.DSN)
	if err != nil { panic(err) }
	defer db.Close()

	repo := NewUserRepo(db)
	svc := NewUserService(repo, log)
	srv := &http.Server{Addr: ":8080", Handler: handler(svc)}

	_ = srv.ListenAndServe()
	_ = ctx
}

func handler(*UserService) http.Handler { return http.NotFoundHandler() }
```

Each constructor takes its dependencies; `main` assembles. Zero runtime magic.

## Deep Dive

### Why Go discourages frameworks

Rob Pike, Russ Cox, and Bryan Mills have all said variations of: "constructors that take their dependencies as arguments are DI; you don't need a framework for that". Idiomatic Go embraces:

- Explicit imports.
- Constructors visible at call sites.
- Type-safe wiring (compile errors when types don't match).
- No runtime registration / reflection magic.

A `main.go` that manually wires 50 dependencies isn't a code smell — it's *the* place to see the whole shape of the application.

### When manual stops scaling

Three pain points show up in large services:

1. **Hundreds of constructors**. `main.go` grows to thousands of lines.
2. **Lifecycle**: each dependency may need `Start(ctx)` / `Stop(ctx)` in a specific order.
3. **Conditional graphs**: production vs staging vs test wires different concrete types.

These are where Wire and Fx help.

### Wire (compile-time code generation)

Wire (https://github.com/google/wire) generates Go code at *build time* that does the wiring. You declare "providers" and "sets"; Wire generates the function body.

```go
// wire.go (build tag wireinject keeps it out of normal builds)
//go:build wireinject

package main

import "github.com/google/wire"

func InitializeApp(cfg Config) (*App, func(), error) {
	wire.Build(
		NewLogger,
		NewDB,
		NewUserRepo,
		NewUserService,
		NewServer,
		NewApp,
	)
	return nil, nil, nil  // never executed; wire_gen.go has the real impl
}
```

Run `wire` (the CLI). It produces `wire_gen.go`:

```go
// generated
func InitializeApp(cfg Config) (*App, func(), error) {
	log := NewLogger()
	db, err := NewDB(cfg)
	if err != nil { return nil, nil, err }
	cleanup := func() { db.Close() }
	repo := NewUserRepo(db)
	svc := NewUserService(repo, log)
	srv := NewServer(svc)
	app := NewApp(srv, log)
	return app, cleanup, nil
}
```

Properties:
- **No runtime cost**: the generated code is plain Go.
- **Compile-time errors**: wire fails the build if a type can't be satisfied.
- **No reflection**.
- **Bindings**: tell wire "this interface is satisfied by this struct" via `wire.Bind`.
- **Provider sets**: bundle related providers (`wire.NewSet(NewLogger, NewDB, ...)`).

Limitations:
- Generated code; needs `go generate` step.
- Wire's grammar of providers is its own DSL inside Go.
- Conditional wiring requires variants of the injector function.

### Fx (runtime, reflection)

Fx (https://github.com/uber-go/fx) by Uber wires the graph at runtime using reflection. Pattern:

```go
package main

import (
	"context"
	"net/http"

	"go.uber.org/fx"
	"go.uber.org/zap"
)

func NewMux(log *zap.Logger) *http.ServeMux {
	m := http.NewServeMux()
	m.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		log.Info("hit")
		w.Write([]byte("hi"))
	})
	return m
}

func NewServer(lc fx.Lifecycle, mux *http.ServeMux) *http.Server {
	srv := &http.Server{Addr: ":8080", Handler: mux}
	lc.Append(fx.Hook{
		OnStart: func(_ context.Context) error {
			go srv.ListenAndServe()
			return nil
		},
		OnStop: func(ctx context.Context) error { return srv.Shutdown(ctx) },
	})
	return srv
}

func main() {
	fx.New(
		fx.Provide(zap.NewProduction, NewMux, NewServer),
		fx.Invoke(func(*http.Server) {}),
	).Run()
}
```

Properties:
- **Reflection-based**: Fx inspects function signatures, matches providers to consumers.
- **Lifecycle hooks**: `fx.Lifecycle` is special; constructors can `lc.Append(fx.Hook{...})` to register start/stop.
- **Modules**: `fx.Module("name", fx.Provide(...))` groups providers.
- **Multi-return providers**: a constructor can return multiple types.

Trade-offs vs Wire:
- Errors surface at runtime (during `fx.New`), not compile.
- Reflection cost on startup (negligible after).
- More flexible for cross-cutting modules.
- The framework owns `main`; you `Invoke` to bootstrap.

### dig (foundational lib under Fx)

[Uber's `dig`](https://github.com/uber-go/dig) is Fx's underlying container. Lower-level; sometimes used directly when Fx's lifecycle isn't needed.

### Other DI frameworks

- **GoogleCloudPlatform's `wire`** (above): Google's recommended.
- **Uber's `fx`** + `dig`: Uber's standard.
- **`facebookgo/inject`**: older, tag-based field injection. Largely abandoned.
- **`samber/do`**: lightweight runtime container with generics; newer.
- **`golobby/container`**: simple container.

### Functional / lambda-based DI

The Go community sometimes uses higher-order functions as a lightweight DI alternative:

```go
type GetUserFunc func(ctx context.Context, id string) (User, error)

func NewGetUser(db *sql.DB) GetUserFunc {
    return func(ctx context.Context, id string) (User, error) {
        // query db
        return User{}, nil
    }
}

func handler(getUser GetUserFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        u, _ := getUser(r.Context(), r.URL.Query().Get("id"))
        _ = u
    }
}
```

Each "service" is a function value; dependencies are closure-captured. Useful for test doubles (just pass a fake function). Doesn't scale to many cross-cutting deps.

### Interfaces and DI

Defining interfaces at the *consumer* side (not the producer side) is the Go idiom — sometimes called "**accept interfaces, return structs**":

```go
package service

type Repo interface {
    GetUser(ctx context.Context, id string) (User, error)
}

type Service struct{ repo Repo }

func New(repo Repo) *Service { return &Service{repo: repo} }
```

`Service` only sees `Repo` (a narrow interface). Concrete implementations can be passed; tests use mocks.

Don't create interfaces for everything. Use them where a *seam* matters (testing, swapping implementations).

### Lifecycle ordering

Manual:

```go
db := openDB(cfg)
defer db.Close()         // closes first

cache := openCache(cfg)
defer cache.Close()      // closes before db (LIFO)

svc := NewService(db, cache)
```

`defer` runs LIFO — last-opened-first-closed. Often the right order naturally.

Fx makes lifecycle explicit via `lc.Append(fx.Hook{OnStart, OnStop})`. Stops happen in reverse order of starts.

Wire doesn't manage lifecycle; you return a `cleanup func()` from the injector and call it manually.

### Conditional wiring

Manual: an `if` in `main`.

Wire: separate injector functions (`InitProd`, `InitDev`, `InitTest`) each with its own `wire.Build`.

Fx: `fx.Options(...)` plus conditional includes.

### Mocking in tests

Most Go services don't use the DI framework in unit tests; they instantiate the SUT (System Under Test) directly with fakes:

```go
func TestUserService(t *testing.T) {
    repo := &mockRepo{}
    svc := NewUserService(repo, slog.Default())
    // ...
}
```

Frameworks add value at the *integration test* level (spinning up the whole graph with test stand-ins).

### `init()` and DI

`init()` is a tempting place for "global wiring" — registering a logger, opening a connection. **Don't**. `init()` runs before `main` can set context, makes errors fatal silently, and prevents testing. Use explicit constructors.

### DI and `context.Context`

`context.Context` is NOT injected; it's *passed*. The first parameter to every method that does I/O or blocking work. DI provides the singleton dependencies (logger, db, service); context flows per request.

### Common patterns at scale

#### Module pattern (Fx)

```go
package store

var Module = fx.Module("store",
    fx.Provide(NewDB, NewUserRepo, NewSessionRepo),
)

// elsewhere:
fx.New(store.Module, service.Module, http.Module).Run()
```

Each package exposes a `Module`; main composes.

#### Provider set (Wire)

```go
var ProductionSet = wire.NewSet(NewDB, NewUserRepo, NewService, NewServer)
```

Bundles providers for reuse across injectors.

#### Explicit interface registration

Wire needs you to register interface bindings:

```go
wire.Bind(new(Repo), new(*PostgresRepo))
```

Fx infers bindings from constructor return types (just return `Repo`, not `*PostgresRepo`).

## Standard Library Hooks

No DI tools in stdlib. Related:

- `context.Context`: per-request data passing.
- `runtime.SetFinalizer` / `runtime.AddCleanup`: not for DI; for resource cleanup.
- `sync.Once`: lazy singleton initialization.
- `flag` / `os` / `encoding/...`: config loading.

## Real-World Patterns

### 1. Manual DI with explicit lifecycle

```go
type App struct {
    db    *sql.DB
    cache Cache
    svc   *Service
    srv   *http.Server
}

func (a *App) Start(ctx context.Context) error {
    return a.srv.ListenAndServe()
}

func (a *App) Stop(ctx context.Context) error {
    a.srv.Shutdown(ctx)
    a.cache.Close()
    a.db.Close()
    return nil
}

func main() {
    cfg := loadConfig()
    db := openDB(cfg)
    cache := openCache(cfg)
    svc := NewService(db, cache)
    srv := &http.Server{Handler: handler(svc)}
    app := &App{db: db, cache: cache, svc: svc, srv: srv}

    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    go app.Start(ctx)
    <-ctx.Done()
    sctx, scancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer scancel()
    app.Stop(sctx)
}
```

Idiomatic Go; scales further than people expect.

### 2. Wire injector + cleanup

```go
//go:build wireinject

func InitializeApp(cfg Config) (*App, func(), error) {
    wire.Build(NewLogger, NewDB, NewCache, NewService, NewServer, NewApp)
    return nil, nil, nil
}

// Generated code returns a cleanup that closes resources.
```

Caller:

```go
app, cleanup, err := InitializeApp(cfg)
if err != nil { return err }
defer cleanup()
app.Run()
```

### 3. Fx with modules

```go
package main

import (
    "go.uber.org/fx"
    "myproj/internal/db"
    "myproj/internal/svc"
    "myproj/internal/http"
)

func main() {
    fx.New(
        db.Module,
        svc.Module,
        http.Module,
        fx.Invoke(func(*http.Server) {}),
    ).Run()
}
```

`Run()` blocks until SIGINT/SIGTERM; lifecycle stops in reverse.

### 4. Test-friendly constructors

```go
func TestUserService(t *testing.T) {
    repo := mockRepo{users: map[string]User{"u1": {Name: "Alice"}}}
    svc := NewUserService(repo, slog.Default())

    u, err := svc.Get(context.Background(), "u1")
    require.NoError(t, err)
    require.Equal(t, "Alice", u.Name)
}
```

`mockRepo` satisfies the `Repo` interface. No DI framework involved.

### 5. Functional-style "DI"

```go
type Deps struct {
    DB  *sql.DB
    Log *slog.Logger
}

func GetUser(d Deps) func(ctx context.Context, id string) (User, error) {
    return func(ctx context.Context, id string) (User, error) {
        d.Log.Info("get_user", "id", id)
        return queryDB(ctx, d.DB, id)
    }
}
```

Suitable for small projects or per-handler wiring.

## Anti-Patterns & Gotchas

**Service locator pattern.** A global registry that anyone can pull from defeats type safety. Pass dependencies explicitly.

**God constructor.** A `NewApp(everything ...)` with 20 parameters means too much coupling. Decompose into sub-packages.

**Interface bloat.** Every implementation gets a 30-method interface; tests now write 30 mocked methods. Define narrow interfaces at consumers.

**Hidden globals.** Logger or DB as a package-level variable. Hard to test, hard to reason about.

**Tag-based "auto-injection"** (the `facebookgo/inject` style). Reflection-driven; surprising errors; not idiomatic.

**`init()` for resource opening.** Pre-`main` failures are uninspectable.

**Fx for tiny projects.** Overhead exceeds value. Manual DI suffices below ~30 constructors.

**Mixing DI styles.** Some constructors via Fx, some via globals, some manual. Pick one for the codebase.

**Wire bindings that lie about contracts.** Binding `Repo` to `MockRepo` in tests when production uses `PostgresRepo` — the test isn't testing real behavior.

**Re-injecting `context.Context`.** Context flows per call; never as a constructor argument unless you specifically need a "lifetime context".

## Performance Notes

- Manual DI: zero overhead.
- Wire-generated code: zero overhead at runtime.
- Fx startup: ~ms (reflection on small graphs); negligible after startup.
- Singleton pattern + lazy init: `sync.Once` is ~ns.
- Mock vs real implementation at runtime: dispatch through interface adds ~1-2 ns; rarely matters.

## How Big Companies Use It

- **Google**: Wire on most new Go services; manual on legacy.
- **Uber**: Fx on virtually every Go service.
- **Cloudflare**: mostly manual; small projects use bespoke containers.
- **CockroachDB**: manual (the team values explicit imports).
- **Kubernetes**: manual; informer / controller-runtime is a framework layer over manual DI.
- **HashiCorp**: manual.
- **GitHub Actions runner (Go)**: manual.
- **Tailscale**: manual.

The split: Uber-influenced companies tend Fx; Google-influenced tend Wire; everyone else tends manual.

## Source Code References

- Wire: [`google/wire`](https://github.com/google/wire).
- Fx: [`uber-go/fx`](https://github.com/uber-go/fx).
- Dig (under Fx): [`uber-go/dig`](https://github.com/uber-go/dig).
- samber/do: [`samber/do`](https://github.com/samber/do).
- golobby/container: [`golobby/container`](https://github.com/golobby/container).
- Kubernetes-style controller wiring (manual): [`kubernetes/sample-controller`](https://github.com/kubernetes/sample-controller).

## Further Reading

- "Dependency Injection in Go" — Mat Ryer: https://medium.com/@matryer.
- "Wire: Compile-Time Dependency Injection for Go" — Google blog: https://blog.golang.org/wire.
- "Fx documentation": https://uber-go.github.io/fx.
- "Accept interfaces, return structs" — Jack Lindamood blog.
- Dave Cheney, "SOLID Go design": https://dave.cheney.net/2016/08/20/solid-go-design.
- Mat Ryer, "Idiomatic dependency injection in Go" (GopherCon talks).
- Bryan Mills, "Functional dependencies and Go": golang-nuts archives.
- Effective Go on composition: https://go.dev/doc/effective_go.

## Exercises / Self-Check

1. Manually wire a service with 5 dependencies (logger, config, DB, cache, HTTP server). Then convert to Wire. Compare.
2. Implement the same in Fx. What lifecycle hooks do you add?
3. Replace one concrete dependency with an interface (`Repo`) and write a mock implementation. Run unit tests.
4. In Wire, add a `wire.Bind(new(Repo), new(*PostgresRepo))`. Why is this needed for interface satisfaction?
5. At what scale (rough number of constructors) does manual DI feel painful in your projects? Identify the exact reason — wiring, lifecycle, or conditional logic.
