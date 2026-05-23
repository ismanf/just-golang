# `database/sql` — Driver-Agnostic SQL

## TL;DR

`database/sql` is the connection-pool + interface layer over SQL drivers. Drivers (`pgx`, `mysql`, `sqlite`) register via `sql.Register`; your code uses `sql.Open(driverName, dsn)` and gets a `*sql.DB` — which is **not** a connection but a pool. Always use the `Context` variants (`QueryContext`, `ExecContext`, `BeginTx`). The classic bug: forgetting to `Close()` rows leaks connections; the pool exhausts and the next query blocks.

## Mental Model

```
sql.DB
  ├─ connection pool (configurable max/idle)
  ├─ uses driver to dial/talk SQL
  └─ all methods are safe for concurrent use

Each query borrows a conn → executes → returns conn (after rows.Close())

Lifecycle:
  Open → Ping → repeated QueryContext/ExecContext → Close on shutdown
```

## Syntax & Basic Usage

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	_ "github.com/jackc/pgx/v5/stdlib" // imports the driver
)

func main() {
	db, err := sql.Open("pgx", "postgres://user:pw@localhost/mydb")
	if err != nil { panic(err) }
	defer db.Close()

	if err := db.PingContext(context.Background()); err != nil { panic(err) }

	var name string
	err = db.QueryRowContext(context.Background(),
		"SELECT name FROM users WHERE id = $1", 42).Scan(&name)
	if err != nil { panic(err) }
	fmt.Println(name)
}
```

## Deep Dive

### `Open` vs `Connect`

`sql.Open` does NOT establish a connection — it sets up the pool lazily. The first `Query` or explicit `Ping` actually dials.

```go
db, _ := sql.Open(driver, dsn)
if err := db.PingContext(ctx); err != nil { /* real connect error */ }
```

### Pool configuration

```go
db.SetMaxOpenConns(50)
db.SetMaxIdleConns(10)
db.SetConnMaxIdleTime(5 * time.Minute)
db.SetConnMaxLifetime(30 * time.Minute)
```

Defaults: unlimited `MaxOpenConns` (bad on burst), `MaxIdleConns = 2`. For production, set both explicitly.

### Query types

```go
// Single row:
err := db.QueryRowContext(ctx, q, args...).Scan(&dest1, &dest2)
// returns sql.ErrNoRows if no rows

// Multiple rows:
rows, err := db.QueryContext(ctx, q, args...)
if err != nil { return err }
defer rows.Close() // ALWAYS
for rows.Next() {
	if err := rows.Scan(&a, &b); err != nil { return err }
}
if err := rows.Err(); err != nil { return err }

// No result:
res, err := db.ExecContext(ctx, "UPDATE ...", args...)
id, _ := res.LastInsertId()
n, _ := res.RowsAffected()
```

### Transactions

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
if err != nil { return err }
defer tx.Rollback() // no-op if Committed

_, err = tx.ExecContext(ctx, "UPDATE ...")
if err != nil { return err }
return tx.Commit()
```

`Rollback` after `Commit` is a safe no-op. The `defer Rollback()` pattern handles errors automatically.

### Prepared statements

```go
stmt, err := db.PrepareContext(ctx, "SELECT name FROM users WHERE id = $1")
defer stmt.Close()
err = stmt.QueryRowContext(ctx, 42).Scan(&name)
```

The pool may re-prepare per connection — each connection holds its own statement. For one-shot queries, `QueryContext` is simpler; the driver may auto-prepare.

### Parameter placeholders (driver-specific!)

- PostgreSQL: `$1`, `$2`, ...
- MySQL/SQLite: `?`
- SQL Server: `@p1` or `?`

Always parameterize — never concatenate user input.

### Scan and NULL handling

```go
var name sql.NullString
err := row.Scan(&name)
if name.Valid { use(name.String) }
```

Pointers also work: `*string` scans NULL as `nil`. The `Null*` types are explicit.

### `sql.ErrNoRows`

`QueryRow` returns nothing until Scan, at which point a no-row returns `sql.ErrNoRows`. Always check:

```go
err := db.QueryRowContext(ctx, q, id).Scan(&name)
if errors.Is(err, sql.ErrNoRows) { /* 404 */ }
```

### Context cancellation

`QueryContext(ctx, ...)` cancels the in-flight query when ctx is done. For PostgreSQL via `pgx`, this issues a `CancelRequest`. For MySQL it closes the connection.

### Driver registration

```go
import _ "github.com/lib/pq"          // PostgreSQL via lib/pq
import _ "github.com/jackc/pgx/v5/stdlib" // PostgreSQL via pgx
import _ "github.com/go-sql-driver/mysql"
import _ "github.com/mattn/go-sqlite3"
```

The blank import triggers `init()` which calls `sql.Register`.

### Connection vs Conn

`db.Conn(ctx)` returns a `*sql.Conn` — a dedicated connection from the pool. Useful for session-scoped operations (advisory locks, temp tables).

## Standard Library Hooks

- `context.Context` everywhere — use the `*Context` methods.
- `io.Closer` for `*sql.DB` and `*sql.Rows`.
- Driver interfaces: `database/sql/driver` for implementing drivers.

## Real-World Patterns

### 1. Repository pattern

```go
type Store struct { db *sql.DB }

func (s *Store) GetUser(ctx context.Context, id string) (*User, error) {
	var u User
	err := s.db.QueryRowContext(ctx, `SELECT id, name FROM users WHERE id = $1`, id).Scan(&u.ID, &u.Name)
	if errors.Is(err, sql.ErrNoRows) { return nil, ErrNotFound }
	if err != nil { return nil, fmt.Errorf("get user %q: %w", id, err) }
	return &u, nil
}
```

### 2. Transaction helper

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil { return err }
	defer tx.Rollback()
	if err := fn(tx); err != nil { return err }
	return tx.Commit()
}
```

### 3. Streaming large result set

```go
rows, err := db.QueryContext(ctx, "SELECT * FROM huge_table")
if err != nil { return err }
defer rows.Close()
for rows.Next() {
	var r Record
	if err := rows.Scan(&r.A, &r.B); err != nil { return err }
	if err := process(r); err != nil { return err }
}
return rows.Err()
```

Use case: backfills, exports.

### 4. Bulk insert with `pgx.CopyFrom` (driver-specific)

`database/sql` has no batch API; for high-throughput inserts, drop to the driver:

```go
import "github.com/jackc/pgx/v5"
conn, _ := pgx.Connect(ctx, dsn)
defer conn.Close(ctx)
_, _ = conn.CopyFrom(ctx, pgx.Identifier{"users"}, []string{"id", "name"}, pgx.CopyFromRows(rows))
```

### 5. Per-request connection with session settings

```go
conn, err := db.Conn(ctx)
if err != nil { return err }
defer conn.Close()
_, _ = conn.ExecContext(ctx, "SET LOCAL search_path TO myschema")
return conn.QueryRowContext(ctx, ...).Scan(...)
```

## Anti-Patterns & Gotchas

**Forgetting `rows.Close()`.** Connection stays checked out → pool exhaustion.

**Not checking `rows.Err()` after the loop.** Network error during iteration is silent otherwise.

**Concatenating user input into SQL.** SQL injection.

**`db.Query` without `Context`.** Hung queries can't be cancelled.

**Treating `sql.Open` as connect.** Use `Ping` to verify.

**Default `MaxOpenConns = 0` (unlimited).** Burst floods the DB.

**Using `*sql.DB.Begin()` then `db.QueryContext` (not on `tx`).** The query runs on a different connection — not in the transaction.

**Forgetting `defer tx.Rollback()`.** Leaked transactions hold locks.

**Returning `*sql.Rows` from a function without docs that the caller must Close.** Pin in the same function.

**Comparing `err == sql.ErrNoRows`.** Use `errors.Is` (some drivers wrap).

**Driver-specific behavior across DBs.** What works on Postgres breaks on MySQL.

## Performance Notes

- Pool overhead per query: ~microseconds; dominated by driver and network.
- `Prepare` followed by repeated `Exec`: faster for the same query against the same connection.
- `pgx` (native) bypasses `database/sql` for max speed; use `pgxpool` for the pool.
- `SetConnMaxIdleTime` and `SetConnMaxLifetime` recycle connections — important for cloud DBs that idle-kill.
- Don't `Ping` per request; do it on health checks only.

## How Big Companies Use It

- **Cockroach** is Postgres-wire-compatible; clients typically use `pgx`.
- **GitHub Actions runner** uses `database/sql` with MySQL for some persistence.
- **Vault** uses `database/sql` against multiple backends via storage plugins.
- **Many Go shops** use `sqlc` (codegen) on top of `database/sql` for typed queries.

## Source Code References

Pinned to `go1.26`.

- `database/sql`: [`src/database/sql/sql.go`](https://github.com/golang/go/blob/master/src/database/sql/sql.go).
- `database/sql/driver`: [`src/database/sql/driver/`](https://github.com/golang/go/tree/master/src/database/sql/driver).
- Connection pool internals: same `sql.go`, search `openNewConnection`, `connectionRequest`.

Go source is BSD-3 licensed.

## Further Reading

- pkg.go.dev: https://pkg.go.dev/database/sql.
- "Common pitfalls when using database/sql": https://go-database-sql.org/.
- pgx README: https://github.com/jackc/pgx.
- sqlc: https://sqlc.dev/.

## Exercises / Self-Check

1. Build a `WithTx` helper that uses Postgres `SERIALIZABLE`. Test that a conflicting concurrent transaction retries.
2. Tune `SetMaxOpenConns`/`SetMaxIdleConns` and observe behavior under load (use `db.Stats()` for metrics).
3. Stream-process a 10M-row table using `QueryContext` + `rows.Next()` without OOM.
4. Reproduce a connection leak by forgetting `rows.Close()`. See `db.Stats()` show "open connections" climb.
5. Compare `database/sql` + `lib/pq` vs `pgx` native for 10k inserts.
