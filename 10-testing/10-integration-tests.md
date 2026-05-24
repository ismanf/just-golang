# Integration Tests — `testcontainers-go`

## TL;DR

**Integration tests** exercise your code against real external dependencies (DBs, message queues, Redis, S3-compatible stores, other services) rather than mocks. **`testcontainers-go`** is the de-facto library for spinning up dependencies as Docker containers per test run: you declare `postgres.RunContainer(ctx, postgres.WithImage("postgres:16"))`, get back a handle with `ConnectionString(ctx)`, run your tests, and the container is torn down on `t.Cleanup`. Containers start in seconds; the API has presets for ~50 popular services (Postgres, MySQL, Mongo, Redis, Kafka, RabbitMQ, MinIO, Elasticsearch, LocalStack for AWS, etc.). Convention: gate integration tests behind a **build tag** (`//go:build integration`) or **`-short`** so unit tests don't pay the cost. Pair with `go test -coverpkg=./...` (1.20+) for cross-binary coverage.

## Mental Model

```
   //go:build integration
   package mypkg_test

   import testcontainers "github.com/testcontainers/testcontainers-go"
   import postgres        "github.com/testcontainers/testcontainers-go/modules/postgres"

   func TestRepo(t *testing.T) {
       ctx := t.Context()
       c, _ := postgres.Run(ctx, "postgres:16-alpine",
           postgres.WithDatabase("test"),
           postgres.WithUsername("postgres"),
           postgres.WithPassword("postgres"),
       )
       t.Cleanup(func() { c.Terminate(ctx) })

       url, _ := c.ConnectionString(ctx, "sslmode=disable")
       db, _ := sql.Open("postgres", url)
       repo := NewRepo(db)
       // ... test against real Postgres
   }
```

The container lifecycle is bound to the test; teardown is automatic.

## Syntax & Basic Usage

```bash
$ go get github.com/testcontainers/testcontainers-go
$ go get github.com/testcontainers/testcontainers-go/modules/postgres
$ go get github.com/testcontainers/testcontainers-go/modules/redis

# Build tag conventionally separates integration from unit:
$ go test ./...                       # unit only
$ go test -tags=integration ./...     # unit + integration
```

```go
// repo_integration_test.go
//go:build integration

package repo_test

import (
    "context"
    "testing"

    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    _ "github.com/lib/pq"
)

func TestInsertSelect(t *testing.T) {
    ctx := t.Context()
    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("postgres"),
        postgres.WithPassword("postgres"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").WithOccurrence(2),
        ),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { pg.Terminate(ctx) })

    dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
    db, _ := sql.Open("postgres", dsn)
    t.Cleanup(func() { db.Close() })

    repo := NewRepo(db)
    if err := repo.Init(ctx); err != nil { t.Fatal(err) }

    if err := repo.Insert(ctx, "alice"); err != nil { t.Fatal(err) }
    got, err := repo.Get(ctx, "alice")
    if err != nil { t.Fatal(err) }
    if got != "alice" { t.Errorf("got %q", got) }
}
```

## Deep Dive

### Why integration tests

- **Catch SQL-level bugs**: dialect differences, index hits, transaction semantics.
- **Catch driver-level bugs**: NULL handling, type coercion.
- **Catch wire-protocol bugs**: serialization edge cases.
- **Provide confidence** the production code path actually works.

Unit tests with mocks can pass while the real DB rejects your migration. Integration tests catch this.

### `testcontainers-go` architecture

`testcontainers-go` is a Go SDK over the Docker API (via `moby/docker` client). It supports:

- **Plain containers**: `testcontainers.GenericContainer` for any image.
- **Modules**: pre-configured presets for popular services.
- **Compose**: orchestrate multi-container setups.
- **Reusable containers**: share across tests via name.

Containers are tracked in a "reaper" container (Ryuk) that ensures cleanup even if tests crash. Set `TESTCONTAINERS_RYUK_DISABLED=true` to disable in CI environments where it conflicts.

### Modules (selected)

| Module               | Image                                            |
|----------------------|--------------------------------------------------|
| `postgres`           | `postgres:*`                                     |
| `mysql`              | `mysql:*`                                        |
| `mongodb`            | `mongo:*`                                        |
| `redis`              | `redis:*`                                        |
| `kafka`              | `confluentinc/cp-kafka:*`                        |
| `rabbitmq`           | `rabbitmq:*`                                     |
| `localstack`         | `localstack/localstack:*` (AWS emulator)         |
| `minio`              | `minio/minio:*` (S3-compatible)                  |
| `elasticsearch`      | `docker.elastic.co/elasticsearch/elasticsearch:*`|
| `nats`               | `nats:*`                                         |
| `pulsar`             | `apachepulsar/pulsar:*`                          |
| `clickhouse`         | `clickhouse/clickhouse-server:*`                 |
| `cockroachdb`        | `cockroachdb/cockroach:*`                        |
| `dynamodb`           | `amazon/dynamodb-local:*`                        |
| `wiremock`           | `wiremock/wiremock:*`                            |

Each module exposes a `Run(ctx, image, opts...)` constructor and methods to derive connection strings.

### Wait strategies

A container is "started" when Docker says so, but the service inside may not be ready. Wait strategies tell testcontainers when to consider it usable:

```go
import "github.com/testcontainers/testcontainers-go/wait"

wait.ForLog("ready to accept connections")           // grep container logs
wait.ForListeningPort("5432/tcp")
wait.ForHTTP("/health").WithPort("8080")
wait.ForExec([]string{"pg_isready"})
wait.ForSQL("5432/tcp", "postgres", func(host string, port nat.Port) string {
    return fmt.Sprintf("postgres://user:pass@%s:%s/db", host, port.Port())
})
```

Without a wait strategy, your test may try to connect to a half-started service.

### Generic container

For services without a pre-built module:

```go
req := testcontainers.ContainerRequest{
    Image:        "myorg/myservice:1.0",
    ExposedPorts: []string{"8080/tcp"},
    Env: map[string]string{
        "FOO": "bar",
    },
    WaitingFor: wait.ForHTTP("/").WithPort("8080/tcp"),
}
c, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
    ContainerRequest: req,
    Started:          true,
})
```

### Compose

```go
import "github.com/testcontainers/testcontainers-go/modules/compose"

c, err := compose.NewDockerCompose("docker-compose.yml")
if err != nil { t.Fatal(err) }
defer c.Down(ctx)
if err := c.Up(ctx, compose.Wait(true)); err != nil { t.Fatal(err) }

// then connect to services
```

For multi-service setups (e.g., your service + Postgres + Redis + Kafka).

### Build tag convention

```go
//go:build integration
```

Above the package clause in integration test files. Run:

```bash
$ go test ./...                       # unit
$ go test -tags=integration ./...     # all
```

CI typically has two jobs: unit (fast, every PR) and integration (slower, every PR or nightly).

### `-short` alternative

```go
func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    // ...
}
```

Run with `go test -short` to skip. Less clean than build tags (file still compiles, deps still pulled).

### Shared container across tests

A single Postgres container can serve many tests if each uses its own database/transaction:

```go
var sharedDB *sql.DB

func TestMain(m *testing.M) {
    ctx := context.Background()
    pg, _ := postgres.Run(ctx, "postgres:16-alpine", ...)
    defer pg.Terminate(ctx)
    dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
    sharedDB, _ = sql.Open("postgres", dsn)
    os.Exit(m.Run())
}

func TestX(t *testing.T) {
    tx, _ := sharedDB.Begin()
    t.Cleanup(func() { tx.Rollback() })
    // ... test using tx, no commit
}
```

Each test gets its own transaction, rolled back at end. Fast (no container restart) and isolated.

### Container per test (clean slate)

```go
func TestX(t *testing.T) {
    pg := startPG(t)
    db := connect(t, pg)
    // ... test
}
```

Each test fully isolated but spends ~5s on container startup. Acceptable for slow CI; painful for local iteration.

### Connection reuse

```go
func TestSuite(t *testing.T) {
    pg := startPG(t)
    db := connect(t, pg)

    t.Run("insert", func(t *testing.T) { testInsert(t, db) })
    t.Run("update", func(t *testing.T) { testUpdate(t, db) })
    t.Run("delete", func(t *testing.T) { testDelete(t, db) })
}
```

One container, many subtests. Run subtests sequentially or use isolated schemas per subtest if parallel.

### Init scripts

```go
pg, _ := postgres.Run(ctx, "postgres:16-alpine",
    postgres.WithInitScripts("testdata/schema.sql"),
    postgres.WithDatabase("test"),
    postgres.WithUsername("postgres"),
    postgres.WithPassword("postgres"),
)
```

`postgres-data/init-scripts` runs `.sql` and `.sh` files at container init. Your schema/seed lives in `testdata/`.

### Performance tips

- **Cache images**: `docker pull` ahead of test runs in CI.
- **Use Alpine variants**: smaller, faster start.
- **Reuse containers via TestMain**: amortize ~5s startup over many tests.
- **Parallelize via `t.Parallel`** but limit (`-parallel=4`) to avoid Docker daemon overload.

### Coverage of integration tests (1.20+)

```bash
$ go test -tags=integration -coverpkg=./... -cover -coverprofile=cov.out ./...
$ go tool cover -func=cov.out | tail
```

Pre-1.20, integration tests couldn't easily contribute to coverage of code in other packages. Go 1.20 added `-coverpkg=./...` plus binary-mode coverage; integration tests now count.

For coverage from `-cover`-built binaries running inside containers:

```go
// in main.go
import _ "runtime/coverage"     // optional explicit import

// at process exit
coverage.WriteMetaDir("/cov")
coverage.WriteCountersDir("/cov")
```

Then collect via `go tool covdata`. See `10-testing/11-coverage.md`.

### CI patterns

GitHub Actions:

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-go@v5
- run: docker pull postgres:16-alpine   # warm cache
- run: go test -tags=integration ./...
```

The runner needs Docker installed (GitHub-hosted runners include it).

For Mac/Windows GitHub-hosted runners, Docker isn't available by default; use Linux for integration tests.

### Alternatives

- **In-memory SQLite**: `modernc.org/sqlite` (pure Go, no CGO). Fast but dialect differs from Postgres/MySQL.
- **In-process Postgres**: [`embedded-postgres-go`](https://github.com/fergusstrange/embedded-postgres) downloads a Postgres binary; lightweight but binary footprint.
- **Real shared DB**: a Postgres instance per developer; brittle, hard to parallelize.
- **TestContainers in cloud**: TestContainers Cloud (paid) runs containers in their cloud; offloads from CI.

For most teams: testcontainers + local Docker = pragmatic default.

### Docker-less CI

If Docker isn't available (some shared CI), testcontainers won't work. Fall back to:

- SQLite for simple cases.
- A shared remote DB (managed Postgres on a dev cluster).
- A real cloud DB via Terraform per test run (expensive).

### LocalStack for AWS

```go
import "github.com/testcontainers/testcontainers-go/modules/localstack"

l, _ := localstack.Run(ctx, "localstack/localstack:3.0",
    testcontainers.WithEnv(map[string]string{"SERVICES": "s3,sqs"}),
)
endpoint, _ := l.PortEndpoint(ctx, "4566/tcp", "http")
// configure AWS SDK to use endpoint
```

Tests S3/SQS/Lambda/DynamoDB integration without an AWS account.

## Standard Library Hooks

- `t.Cleanup` — auto-terminate containers.
- `t.Context` (1.24+) — pass to containerized operations.
- `database/sql` — DB driver wrapper.
- `os/exec` — invoke external tools.
- `testing.Short` — skip-decision helper.

Third-party essentials:

- `testcontainers-go` — the SDK.
- `lib/pq`, `jackc/pgx`, `go-sql-driver/mysql` — DB drivers.
- `redis/go-redis` — Redis client.
- `aws/aws-sdk-go-v2` — AWS SDK (works with LocalStack).

## Real-World Patterns

### 1. Postgres per test

```go
func TestRepo(t *testing.T) {
    db := setupPostgres(t)
    repo := NewRepo(db)
    // ...
}

func setupPostgres(t *testing.T) *sql.DB {
    t.Helper()
    ctx := t.Context()
    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("postgres"),
        postgres.WithPassword("postgres"),
        postgres.WithInitScripts("testdata/schema.sql"),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { pg.Terminate(ctx) })

    dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
    db, err := sql.Open("postgres", dsn)
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { db.Close() })
    return db
}
```

### 2. Shared via TestMain

```go
var pgURL string

func TestMain(m *testing.M) {
    ctx := context.Background()
    pg, err := postgres.Run(ctx, "postgres:16-alpine", ...)
    if err != nil { panic(err) }
    defer pg.Terminate(ctx)
    pgURL, _ = pg.ConnectionString(ctx, "sslmode=disable")
    os.Exit(m.Run())
}

func TestX(t *testing.T) {
    db, _ := sql.Open("postgres", pgURL)
    defer db.Close()
    // ...
}
```

### 3. Transaction-per-test

```go
func freshTx(t *testing.T, db *sql.DB) *sql.Tx {
    t.Helper()
    tx, err := db.Begin()
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { tx.Rollback() })
    return tx
}

func TestUpdate(t *testing.T) {
    tx := freshTx(t, sharedDB)
    repo := NewRepoFromTx(tx)
    // ... commits never happen; rollback at end
}
```

### 4. Redis integration

```go
func TestCache(t *testing.T) {
    ctx := t.Context()
    r, _ := redis.Run(ctx, "redis:7-alpine")
    t.Cleanup(func() { r.Terminate(ctx) })

    url, _ := r.ConnectionString(ctx)
    client := goredis.NewClient(&goredis.Options{Addr: url})
    t.Cleanup(func() { client.Close() })

    // ... test
}
```

### 5. Multi-container

```go
ctx := t.Context()
network, _ := testcontainers.NewNetwork(ctx)
t.Cleanup(func() { network.Remove(ctx) })

pg, _ := postgres.Run(ctx, "postgres:16-alpine",
    network.WithNetwork([]string{"db"}, network),
)
app, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
    ContainerRequest: testcontainers.ContainerRequest{
        Image: "myorg/myapp:test",
        Env: map[string]string{"DB_URL": "postgres://...@db:5432/..."},
        Networks: []string{network.Name},
    },
    Started: true,
})
```

### 6. Kafka

```go
import "github.com/testcontainers/testcontainers-go/modules/kafka"

k, _ := kafka.Run(ctx, "confluentinc/cp-kafka:7.6.0")
brokers, _ := k.Brokers(ctx)
// connect with sarama / segmentio/kafka-go
```

### 7. CI integration job

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: go test ./...
  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - run: docker pull postgres:16-alpine redis:7-alpine
      - run: go test -tags=integration ./...
```

## Anti-Patterns & Gotchas

**Running integration tests on every push without parallelism.** A 30-test suite at 5s startup each = 2.5 min per CI run. Use shared containers or `t.Parallel` + isolated transactions.

**Forgetting `t.Cleanup` to terminate containers.** Containers leak; eventually Docker host exhausts resources.

**Not specifying a wait strategy.** Tests connect before the service is ready; flaky.

**Hardcoded ports in test config.** `testcontainers` exposes random host ports; use `ConnectionString` / `MappedPort`.

**Tests assuming Docker is available.** Mac/Windows GitHub runners don't have Docker. Build tag your integration tests; run them only on Linux runners.

**Container images without version tags.** `postgres:latest` changes; tests flake. Pin: `postgres:16-alpine`.

**One huge container shared by all tests with no isolation.** Tests pollute each other's state. Use transactions or per-test schemas.

**Skipping integration tests in CI.** If they never run, they bit-rot.

**Running unit tests with `-tags=integration`.** Integration tests run on every push; slows the loop. Keep them separate.

**Pulling images during the test rather than CI prep.** Each pull is ~10s; tests appear slower than they should. `docker pull` ahead of time.

**Logging container output to test stdout always.** Verbose; clutters CI logs. Capture with `c.Logs(ctx)` only on failure.

**Forgetting CGO for SQLite.** `mattn/go-sqlite3` needs CGO; if `CGO_ENABLED=0`, build fails. Use `modernc.org/sqlite` (pure Go) instead.

## Performance Notes

- Container startup: 2–15 s depending on image and wait strategy.
- Postgres ready: ~3 s.
- Redis ready: ~1 s.
- Kafka ready: ~10 s (Zookeeper sequence).
- Per-test wait + connect: ~ms after the container is up.
- Total integration suite (50 tests, shared container): 1–3 min typical.
- Without shared container: 5–15 min.

Docker daemon overhead: each `docker run` is ~100ms on the API side; bulk operations are faster.

## How Big Companies Use It

- **CockroachDB** uses testcontainers for client SDK tests: https://github.com/cockroachdb/cockroach-go.
- **Kubernetes** uses `envtest` (a k8s-specific equivalent) for API server integration: https://github.com/kubernetes-sigs/controller-runtime/tree/master/pkg/envtest.
- **HashiCorp** uses testcontainers for Vault's storage backend tests: https://github.com/hashicorp/vault.
- **Uber** uses testcontainers in `cadence`/`temporal` for DB and Cassandra tests: https://github.com/temporalio/temporal.
- **Cloudflare** uses testcontainers + LocalStack for AWS integration: https://blog.cloudflare.com.
- **Tailscale** uses testcontainers for DERP relay tests: https://github.com/tailscale/tailscale.
- **Discord** uses testcontainers for ScyllaDB and Redis cluster integration: https://discord.com/engineering.

## Source Code References

- testcontainers-go: [`github.com/testcontainers/testcontainers-go`](https://github.com/testcontainers/testcontainers-go).
- Modules: [`github.com/testcontainers/testcontainers-go/modules`](https://github.com/testcontainers/testcontainers-go/tree/main/modules).
- Wait strategies: [`github.com/testcontainers/testcontainers-go/wait`](https://github.com/testcontainers/testcontainers-go/tree/main/wait).
- Ryuk reaper: [`github.com/testcontainers/moby-ryuk`](https://github.com/testcontainers/moby-ryuk).
- LocalStack: [`github.com/localstack/localstack`](https://github.com/localstack/localstack).
- Compose: [`github.com/testcontainers/testcontainers-go/modules/compose`](https://github.com/testcontainers/testcontainers-go/tree/main/modules/compose).

(Apache-2.0 license.)

## Further Reading

- "testcontainers-go documentation": https://golang.testcontainers.org.
- "Modules catalog": https://golang.testcontainers.org/modules.
- "Testing strategies in Go" (Mat Ryer): https://medium.com/@matryer.
- "Integration testing in Go" (Mickael Maison): https://www.cncf.io.
- "Coverage of integration tests" (Go blog): https://go.dev/blog/integration-test-coverage.
- "Test containers everywhere" (Sergei Egorov, original Java author): https://www.testcontainers.org.
- "Using transactions for test isolation" (Mat Ryer): https://pace.dev/blog.

## Exercises / Self-Check

1. Write a single integration test that spins up Postgres, applies a schema, inserts a row, queries it. Use `t.Cleanup`.
2. Convert your integration suite to shared-container + transaction-per-test. Measure the speedup.
3. Add a wait strategy that polls a health endpoint. What if you omit it?
4. Use LocalStack to test S3 upload. Compare to mocking `s3.Client`.
5. Set up a CI job that runs only integration tests (`-tags=integration`). Make it skip on non-Linux runners.
