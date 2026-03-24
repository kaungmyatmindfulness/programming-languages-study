# Chapter 48: SQL in Go (pgx, sqlx, Migrations)

## Table of Contents

1. [Introduction](#introduction)
2. [database/sql Standard Library](#databasesql-standard-library)
3. [Connection Pool Tuning](#connection-pool-tuning)
4. [pgx for PostgreSQL](#pgx-for-postgresql)
5. [sqlx for Enhanced database/sql](#sqlx-for-enhanced-databasesql)
6. [Prepared Statements and SQL Injection Prevention](#prepared-statements-and-sql-injection-prevention)
7. [Transactions](#transactions)
8. [Context-Aware Queries](#context-aware-queries)
9. [Scanning Complex Types](#scanning-complex-types)
10. [Database Migrations](#database-migrations)
11. [Migration Best Practices](#migration-best-practices)
12. [Query Builders](#query-builders)
13. [Testing with Databases](#testing-with-databases)
14. [Connection Error Handling and Retries](#connection-error-handling-and-retries)
15. [Monitoring and Logging Queries](#monitoring-and-logging-queries)
16. [N+1 Query Problem and Solutions](#n1-query-problem-and-solutions)
17. [Summary](#summary)

---

## Introduction

Working with SQL databases is one of the most common tasks in backend Go development. Go's
ecosystem provides several layers of abstraction for database access:

| Layer | Package | Purpose |
|-------|---------|---------|
| Standard library | `database/sql` | Generic SQL interface with connection pooling |
| Driver-specific | `pgx`, `lib/pq`, `go-sql-driver/mysql` | Database-specific drivers |
| Enhanced stdlib | `sqlx` | Struct scanning, named parameters on top of `database/sql` |
| Query builders | `squirrel`, `goqu` | Programmatic SQL construction |
| ORMs | `GORM`, `ent`, `sqlboiler` | Full object-relational mapping |

This chapter focuses on the lower and middle layers -- the ones that give you control over
your SQL while reducing boilerplate. We will use PostgreSQL as the primary database in
examples, but the patterns apply broadly.

---

## database/sql Standard Library

The `database/sql` package is Go's built-in abstraction for relational databases. It provides:

- A **connection pool** managed transparently
- A **driver interface** so any database can plug in
- **Rows**, **Exec**, **Query**, and **QueryRow** for executing SQL
- **Prepared statements** and **transactions**

### Opening a Connection

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq" // PostgreSQL driver (side-effect import)
)

func main() {
    // Format: "postgres://user:password@host:port/dbname?sslmode=disable"
    dsn := "postgres://app_user:secret@localhost:5432/myapp?sslmode=disable"

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        log.Fatalf("failed to open database: %v", err)
    }
    defer db.Close()

    // sql.Open does NOT establish a connection -- it only validates the DSN.
    // Use Ping to verify the connection is reachable.
    if err := db.Ping(); err != nil {
        log.Fatalf("failed to ping database: %v", err)
    }

    fmt.Println("Connected to database successfully")
}
```

> **Warning:** `sql.Open` does **not** open a connection. It validates arguments and
> initializes the pool. Always call `db.Ping()` or `db.PingContext(ctx)` to verify
> connectivity at startup.

### The sql.DB Object

`sql.DB` is **not** a single connection. It is a **connection pool**. You should create one
`sql.DB` per database and share it across your application (it is safe for concurrent use).

```go
// Typical application structure
type App struct {
    db *sql.DB
}

func NewApp(dsn string) (*App, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening database: %w", err)
    }
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("pinging database: %w", err)
    }
    return &App{db: db}, nil
}
```

### Exec -- Statements That Do Not Return Rows

Use `Exec` for INSERT, UPDATE, DELETE, and DDL statements.

```go
func (a *App) CreateUser(ctx context.Context, name, email string) (int64, error) {
    result, err := a.db.ExecContext(ctx,
        `INSERT INTO users (name, email, created_at)
         VALUES ($1, $2, NOW())`,
        name, email,
    )
    if err != nil {
        return 0, fmt.Errorf("inserting user: %w", err)
    }

    // RowsAffected tells you how many rows the statement touched.
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return 0, fmt.Errorf("getting rows affected: %w", err)
    }

    // LastInsertId is NOT supported by PostgreSQL with lib/pq.
    // Use RETURNING clause instead (shown later with pgx/sqlx).
    return rowsAffected, nil
}
```

> **Tip:** PostgreSQL uses `$1, $2, ...` for placeholders. MySQL uses `?`. The `database/sql`
> package does not normalize them -- you must use the driver's placeholder style.

### QueryRow -- Single Row

Use `QueryRow` when you expect exactly one result.

```go
type User struct {
    ID        int64
    Name      string
    Email     string
    CreatedAt time.Time
}

func (a *App) GetUser(ctx context.Context, id int64) (User, error) {
    var u User
    err := a.db.QueryRowContext(ctx,
        `SELECT id, name, email, created_at FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)

    if err == sql.ErrNoRows {
        return User{}, fmt.Errorf("user %d not found", id)
    }
    if err != nil {
        return User{}, fmt.Errorf("querying user %d: %w", id, err)
    }
    return u, nil
}
```

> **Warning:** Always check for `sql.ErrNoRows` when using `QueryRow`. It is returned by
> `Scan`, not by `QueryRow` itself.

### Query -- Multiple Rows

Use `Query` when you expect zero or more results.

```go
func (a *App) ListUsers(ctx context.Context, limit int) ([]User, error) {
    rows, err := a.db.QueryContext(ctx,
        `SELECT id, name, email, created_at
         FROM users
         ORDER BY created_at DESC
         LIMIT $1`, limit,
    )
    if err != nil {
        return nil, fmt.Errorf("querying users: %w", err)
    }
    defer rows.Close() // CRITICAL: always close rows

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt); err != nil {
            return nil, fmt.Errorf("scanning user row: %w", err)
        }
        users = append(users, u)
    }

    // Check for errors from iteration
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterating user rows: %w", err)
    }

    return users, nil
}
```

> **Warning:** Forgetting `defer rows.Close()` is one of the most common bugs. It leaks
> database connections back to the pool. Similarly, always check `rows.Err()` after
> the loop to catch errors that occurred during iteration (network failures, etc.).

### The Rows Lifecycle

```
db.QueryContext(ctx, sql, args...)
         |
         v
    rows (holds a connection from the pool)
         |
    rows.Next()  -->  returns true/false
         |
    rows.Scan(&col1, &col2, ...)
         |
    (repeat until Next() returns false)
         |
    rows.Err()   -->  check for iteration errors
         |
    rows.Close() -->  returns connection to pool
```

The connection is held for the entire duration of the iteration. If you forget to close
Rows, or if your iteration is slow, you starve the pool.

### Nullable Columns

Standard `database/sql` provides nullable types:

```go
func (a *App) GetUserProfile(ctx context.Context, id int64) error {
    var (
        name     string
        bio      sql.NullString   // nullable text
        age      sql.NullInt64    // nullable integer
        rating   sql.NullFloat64  // nullable float
        verified sql.NullBool     // nullable boolean
        birthday sql.NullTime     // nullable timestamp (Go 1.13+)
    )

    err := a.db.QueryRowContext(ctx,
        `SELECT name, bio, age, rating, verified, birthday
         FROM user_profiles WHERE user_id = $1`, id,
    ).Scan(&name, &bio, &age, &rating, &verified, &birthday)
    if err != nil {
        return err
    }

    // Check validity before using
    if bio.Valid {
        fmt.Println("Bio:", bio.String)
    } else {
        fmt.Println("Bio: not set")
    }

    if birthday.Valid {
        fmt.Println("Birthday:", birthday.Time.Format("2006-01-02"))
    }

    return nil
}
```

> **Tip:** Using `*string`, `*int64` etc. with pgx or sqlx is often cleaner than the
> `sql.Null*` types. We cover this in the pgx section.

---

## Connection Pool Tuning

`sql.DB` manages a pool of connections. The defaults are conservative and may not fit
production workloads. Understanding pool tuning is critical for performance and stability.

### Pool Configuration Methods

```go
func configurePool(db *sql.DB) {
    // Maximum number of open (in-use + idle) connections.
    // Default: 0 (unlimited) -- DANGEROUS in production.
    db.SetMaxOpenConns(25)

    // Maximum number of idle connections retained in the pool.
    // Default: 2 -- often too low.
    db.SetMaxIdleConns(10)

    // Maximum time a connection can be reused.
    // Default: 0 (no limit) -- stale connections can cause issues.
    db.SetConnMaxLifetime(5 * time.Minute)

    // Maximum time a connection can sit idle before being closed.
    // Default: 0 (no limit). Added in Go 1.15.
    db.SetConnMaxIdleTime(1 * time.Minute)
}
```

### What Each Setting Does

| Setting | Default | Effect |
|---------|---------|--------|
| `MaxOpenConns` | 0 (unlimited) | Upper limit on total connections (active + idle). When reached, new queries block until a connection is available. |
| `MaxIdleConns` | 2 | How many connections to keep idle. When a connection is released and the pool already has this many idle, it is closed instead of returned. |
| `ConnMaxLifetime` | 0 (forever) | Maximum total time a connection lives. After this, the connection is closed and a new one is created. Helps with DNS changes, load balancer rotation, and PostgreSQL memory leaks. |
| `ConnMaxIdleTime` | 0 (forever) | Maximum time a connection sits idle. Useful for reducing idle connections during low-traffic periods. |

### Tuning Guidelines

```
 +-------------------------------------------------+
 |  Production Pool Tuning Checklist                |
 +-------------------------------------------------+
 |                                                  |
 |  1. ALWAYS set MaxOpenConns.                     |
 |     - Match your database's max_connections      |
 |       minus some headroom.                       |
 |     - If 5 app instances share a DB with         |
 |       max_connections = 100, use ~15 per app.    |
 |                                                  |
 |  2. Set MaxIdleConns = MaxOpenConns              |
 |     - Avoids thrashing (creating/closing         |
 |       connections repeatedly).                   |
 |                                                  |
 |  3. Set ConnMaxLifetime = 5-15 minutes           |
 |     - Prevents stale connections.                |
 |     - Stagger across instances to avoid          |
 |       "thundering herd" reconnections.           |
 |                                                  |
 |  4. Set ConnMaxIdleTime = 1-5 minutes            |
 |     - Scale down idle connections during          |
 |       low traffic.                               |
 +-------------------------------------------------+
```

### Monitoring Pool Stats

```go
func logPoolStats(db *sql.DB) {
    stats := db.Stats()

    log.Printf("Pool stats: open=%d inUse=%d idle=%d waitCount=%d waitDuration=%s maxIdleClosed=%d maxIdleTimeClosed=%d maxLifetimeClosed=%d",
        stats.OpenConnections,
        stats.InUse,
        stats.Idle,
        stats.WaitCount,        // total number of waits for a connection
        stats.WaitDuration,     // total time spent waiting
        stats.MaxIdleClosed,    // closed because MaxIdleConns was exceeded
        stats.MaxIdleTimeClosed,// closed because ConnMaxIdleTime expired
        stats.MaxLifetimeClosed,// closed because ConnMaxLifetime expired
    )
}

// Expose these as Prometheus metrics in production
func registerPoolMetrics(db *sql.DB) {
    go func() {
        ticker := time.NewTicker(15 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            stats := db.Stats()
            dbOpenConns.Set(float64(stats.OpenConnections))
            dbInUseConns.Set(float64(stats.InUse))
            dbIdleConns.Set(float64(stats.Idle))
            dbWaitCount.Add(float64(stats.WaitCount))
            dbWaitDuration.Observe(stats.WaitDuration.Seconds())
        }
    }()
}
```

### Common Pool Problems

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `context deadline exceeded` on queries | MaxOpenConns too low or connection leak | Increase MaxOpenConns; audit `rows.Close()` calls |
| Connections spike then drop | MaxIdleConns too low | Set MaxIdleConns = MaxOpenConns |
| Sporadic "connection reset" | ConnMaxLifetime not set; stale connections | Set ConnMaxLifetime to 5-10 min |
| Too many connections error from DB | MaxOpenConns not set (unlimited) | Always set MaxOpenConns |

---

## pgx for PostgreSQL

[pgx](https://github.com/jackc/pgx) is the most popular PostgreSQL driver for Go. It can
be used in two modes:

1. **As a `database/sql` driver** (`pgx/v5/stdlib`) -- drop-in replacement for `lib/pq`
2. **As a standalone library** (`pgx/v5`) -- direct access to PostgreSQL features

### Why pgx Over lib/pq

| Feature | lib/pq | pgx |
|---------|--------|-----|
| Actively maintained | Maintenance mode | Active development |
| Binary protocol | No (text only) | Yes (faster encoding) |
| COPY support | Basic | Full streaming COPY |
| Batch queries | No | Yes (pipeline multiple queries) |
| LISTEN/NOTIFY | Basic | First-class support |
| Composite types | No | Yes |
| Logical replication | No | Yes |
| Performance | Good | Better (binary protocol) |
| Connection pooling | Via database/sql | Built-in pgxpool |
| Prepared statement cache | No | Automatic |

> **Tip:** For new projects, always choose pgx. lib/pq is in maintenance mode and its
> maintainers recommend pgx.

### Basic pgx Usage (Standalone Mode)

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()

    // Single connection (not pooled) -- useful for scripts, migrations
    conn, err := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatalf("unable to connect: %v", err)
    }
    defer conn.Close(ctx)

    var greeting string
    err = conn.QueryRow(ctx, "SELECT 'Hello, pgx!'").Scan(&greeting)
    if err != nil {
        log.Fatalf("query failed: %v", err)
    }
    fmt.Println(greeting)
}
```

### pgx Connection Pool (pgxpool)

For applications, always use `pgxpool.Pool` instead of a single connection.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    ctx := context.Background()

    // Parse configuration from DSN
    config, err := pgxpool.ParseConfig(os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatalf("unable to parse config: %v", err)
    }

    // Pool tuning
    config.MaxConns = 25                          // max open connections
    config.MinConns = 5                           // minimum idle connections
    config.MaxConnLifetime = 5 * time.Minute      // connection max age
    config.MaxConnIdleTime = 1 * time.Minute      // idle timeout
    config.HealthCheckPeriod = 30 * time.Second   // background health check

    // Connection-level configuration
    config.ConnConfig.ConnectTimeout = 5 * time.Second

    // Optional: run function after connecting (set session params, etc.)
    config.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
        // Example: set timezone per connection
        _, err := conn.Exec(ctx, "SET timezone = 'UTC'")
        return err
    }

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        log.Fatalf("unable to create pool: %v", err)
    }
    defer pool.Close()

    // Use the pool
    var now time.Time
    err = pool.QueryRow(ctx, "SELECT NOW()").Scan(&now)
    if err != nil {
        log.Fatalf("query failed: %v", err)
    }
    fmt.Println("Database time:", now)
}
```

### Querying with pgx

pgx provides `Query`, `QueryRow`, and `Exec` like `database/sql`, but with improvements:

```go
// pgx supports scanning into Go pointers directly for nullable columns
func getUser(ctx context.Context, pool *pgxpool.Pool, id int64) error {
    var (
        name  string
        bio   *string  // nil when SQL NULL -- much cleaner than sql.NullString
        age   *int32
    )

    err := pool.QueryRow(ctx,
        `SELECT name, bio, age FROM users WHERE id = $1`, id,
    ).Scan(&name, &bio, &age)
    if err != nil {
        return fmt.Errorf("querying user: %w", err)
    }

    fmt.Printf("Name: %s\n", name)
    if bio != nil {
        fmt.Printf("Bio: %s\n", *bio)
    }
    return nil
}
```

### pgx CollectRows -- The Modern Way

pgx v5 introduced `pgx.CollectRows` to eliminate the boilerplate scanning loop:

```go
import "github.com/jackc/pgx/v5"

type User struct {
    ID    int64  `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
}

func listUsers(ctx context.Context, pool *pgxpool.Pool) ([]User, error) {
    rows, err := pool.Query(ctx,
        `SELECT id, name, email FROM users ORDER BY id`)
    if err != nil {
        return nil, err
    }

    // CollectRows handles iteration, scanning, closing, and error checking
    users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
    if err != nil {
        return nil, fmt.Errorf("collecting users: %w", err)
    }
    return users, nil
}

// Other collectors:
// pgx.RowTo[T]          -- scan single column into T
// pgx.RowToMap           -- scan into map[string]any
// pgx.RowToStructByPos   -- scan by column position (faster, order-dependent)
// pgx.RowToStructByName  -- scan by column name (safer, order-independent)
// pgx.RowToAddrOfStructByName[T] -- returns []*T instead of []T
```

### RETURNING Clause with pgx

PostgreSQL's `RETURNING` clause lets you get back data from INSERT/UPDATE/DELETE:

```go
func createUser(ctx context.Context, pool *pgxpool.Pool, name, email string) (int64, error) {
    var id int64
    err := pool.QueryRow(ctx,
        `INSERT INTO users (name, email, created_at)
         VALUES ($1, $2, NOW())
         RETURNING id`,
        name, email,
    ).Scan(&id)
    if err != nil {
        return 0, fmt.Errorf("inserting user: %w", err)
    }
    return id, nil
}
```

### Batch Queries

Batching sends multiple queries in a single network round-trip. This is a major performance
win when you need to execute several independent queries.

```go
func batchExample(ctx context.Context, pool *pgxpool.Pool) error {
    batch := &pgx.Batch{}

    // Queue multiple queries
    batch.Queue("SELECT name FROM users WHERE id = $1", 1)
    batch.Queue("SELECT name FROM users WHERE id = $1", 2)
    batch.Queue("SELECT count(*) FROM users")
    batch.Queue(
        `INSERT INTO audit_log (action, timestamp)
         VALUES ($1, NOW())`, "batch_query",
    )

    // Send the entire batch in one round-trip
    br := pool.SendBatch(ctx, batch)
    defer br.Close() // MUST close the batch result

    // Read results in the same order they were queued
    var name1 string
    if err := br.QueryRow().Scan(&name1); err != nil {
        return fmt.Errorf("query 1: %w", err)
    }

    var name2 string
    if err := br.QueryRow().Scan(&name2); err != nil {
        return fmt.Errorf("query 2: %w", err)
    }

    var count int64
    if err := br.QueryRow().Scan(&count); err != nil {
        return fmt.Errorf("query 3: %w", err)
    }

    // For Exec results (no rows returned)
    _, err := br.Exec()
    if err != nil {
        return fmt.Errorf("query 4: %w", err)
    }

    fmt.Printf("User1: %s, User2: %s, Total: %d\n", name1, name2, count)
    return nil
}
```

### COPY Protocol

PostgreSQL's COPY protocol is the fastest way to bulk-insert data. pgx supports it natively.

```go
func bulkInsertUsers(ctx context.Context, pool *pgxpool.Pool, users []User) (int64, error) {
    // CopyFrom uses the PostgreSQL COPY protocol for maximum throughput
    count, err := pool.CopyFrom(
        ctx,
        pgx.Identifier{"users"},           // table name
        []string{"name", "email"},          // column names
        pgx.CopyFromSlice(len(users), func(i int) ([]any, error) {
            return []any{users[i].Name, users[i].Email}, nil
        }),
    )
    if err != nil {
        return 0, fmt.Errorf("copy from: %w", err)
    }

    fmt.Printf("Inserted %d rows via COPY\n", count)
    return count, nil
}

// CopyFrom also supports pgx.CopyFromRows for a pre-built [][]any slice:
func bulkInsertFromRows(ctx context.Context, pool *pgxpool.Pool) (int64, error) {
    rows := [][]any{
        {"Alice", "alice@example.com"},
        {"Bob", "bob@example.com"},
        {"Charlie", "charlie@example.com"},
    }

    return pool.CopyFrom(
        ctx,
        pgx.Identifier{"users"},
        []string{"name", "email"},
        pgx.CopyFromRows(rows),
    )
}
```

Performance comparison for inserting 100,000 rows:

| Method | Time | Relative |
|--------|------|----------|
| Individual INSERT | ~45s | 1x |
| Batch INSERT (1000/batch) | ~4s | 11x |
| COPY protocol | ~0.8s | 56x |

### LISTEN/NOTIFY

pgx has first-class support for PostgreSQL's pub/sub mechanism:

```go
func listenForEvents(ctx context.Context, pool *pgxpool.Pool) error {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return err
    }
    defer conn.Release()

    // Subscribe to channel
    _, err = conn.Exec(ctx, "LISTEN order_events")
    if err != nil {
        return err
    }

    fmt.Println("Listening for order_events...")

    for {
        notification, err := conn.Conn().WaitForNotification(ctx)
        if err != nil {
            return fmt.Errorf("waiting for notification: %w", err)
        }

        fmt.Printf("Channel: %s, Payload: %s\n",
            notification.Channel, notification.Payload)
    }
}

// In another function or service:
func publishEvent(ctx context.Context, pool *pgxpool.Pool, payload string) error {
    _, err := pool.Exec(ctx,
        "SELECT pg_notify('order_events', $1)", payload)
    return err
}
```

### Using pgx as a database/sql Driver

If you need `database/sql` compatibility (e.g., for sqlx or existing code), use the stdlib
adapter:

```go
import (
    "database/sql"
    _ "github.com/jackc/pgx/v5/stdlib" // register pgx as database/sql driver
)

func openWithStdlib() (*sql.DB, error) {
    db, err := sql.Open("pgx", "postgres://user:pass@localhost/mydb")
    if err != nil {
        return nil, err
    }
    return db, nil
}
```

---

## sqlx for Enhanced database/sql

[sqlx](https://github.com/jmoiron/sqlx) wraps `database/sql` with convenience methods that
reduce boilerplate. It is driver-agnostic and works with PostgreSQL, MySQL, SQLite, etc.

### Getting Started with sqlx

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/jmoiron/sqlx"
    _ "github.com/jackc/pgx/v5/stdlib" // use pgx as the underlying driver
)

type User struct {
    ID        int64     `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
}

func main() {
    // sqlx.Connect = sql.Open + Ping
    db, err := sqlx.Connect("pgx", "postgres://user:pass@localhost/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Pool tuning works the same way
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(5 * time.Minute)
}
```

### StructScan -- Scan Rows into Structs

The killer feature of sqlx: automatic scanning based on `db` struct tags.

```go
// With database/sql (verbose):
func listUsersStdlib(db *sql.DB) ([]User, error) {
    rows, err := db.Query("SELECT id, name, email, created_at FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}

// With sqlx (concise):
func listUsersSqlx(db *sqlx.DB) ([]User, error) {
    var users []User
    err := db.Select(&users,
        "SELECT id, name, email, created_at FROM users")
    return users, err
}

// Single row with sqlx:
func getUserSqlx(db *sqlx.DB, id int64) (User, error) {
    var u User
    err := db.Get(&u,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id)
    return u, err
}
```

> **Warning:** `db.Select` loads all rows into memory at once. For large result sets
> (thousands of rows), prefer `db.Queryx` with manual iteration to avoid OOM.

### Queryx -- Iterating with StructScan

```go
func listUsersStream(db *sqlx.DB) error {
    rows, err := db.Queryx("SELECT id, name, email, created_at FROM users")
    if err != nil {
        return err
    }
    defer rows.Close()

    for rows.Next() {
        var u User
        if err := rows.StructScan(&u); err != nil {
            return err
        }
        fmt.Printf("User: %+v\n", u)
    }
    return rows.Err()
}

// Scanning into maps (useful for dynamic queries):
func queryToMaps(db *sqlx.DB, query string) ([]map[string]interface{}, error) {
    rows, err := db.Queryx(query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []map[string]interface{}
    for rows.Next() {
        row := make(map[string]interface{})
        if err := rows.MapScan(row); err != nil {
            return nil, err
        }
        results = append(results, row)
    }
    return results, rows.Err()
}
```

### NamedExec and NamedQuery

Use struct fields or maps as query parameters:

```go
// Named parameters with structs
func createUserNamed(db *sqlx.DB, u User) error {
    _, err := db.NamedExec(
        `INSERT INTO users (name, email, created_at)
         VALUES (:name, :email, :created_at)`, u)
    return err
}

// Named parameters with maps
func createUserFromMap(db *sqlx.DB, data map[string]interface{}) error {
    _, err := db.NamedExec(
        `INSERT INTO users (name, email)
         VALUES (:name, :email)`, data)
    return err
}

// Named query (returns rows)
func findUsersByName(db *sqlx.DB, name string) ([]User, error) {
    var users []User
    rows, err := db.NamedQuery(
        `SELECT id, name, email, created_at
         FROM users WHERE name = :name`,
        map[string]interface{}{"name": name},
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    for rows.Next() {
        var u User
        if err := rows.StructScan(&u); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}
```

### IN Clause Expansion

One of sqlx's handiest features. Expanding slices into `IN (...)` clauses is notoriously
awkward with `database/sql`.

```go
func getUsersByIDs(db *sqlx.DB, ids []int64) ([]User, error) {
    // Step 1: Build the query with sqlx.In
    query, args, err := sqlx.In(
        "SELECT id, name, email FROM users WHERE id IN (?)", ids)
    if err != nil {
        return nil, fmt.Errorf("building IN query: %w", err)
    }

    // Step 2: Rebind placeholders from ? to $1, $2, ... for PostgreSQL
    query = db.Rebind(query)

    // Step 3: Execute
    var users []User
    err = db.Select(&users, query, args...)
    if err != nil {
        return nil, fmt.Errorf("querying users: %w", err)
    }
    return users, nil
}

// Combined IN with other parameters:
func searchUsers(db *sqlx.DB, name string, statuses []string) ([]User, error) {
    query, args, err := sqlx.In(
        `SELECT id, name, email FROM users
         WHERE name LIKE ? AND status IN (?)`,
        "%"+name+"%", statuses)
    if err != nil {
        return nil, err
    }

    query = db.Rebind(query)

    var users []User
    err = db.Select(&users, query, args...)
    return users, err
}
```

### Embedded Structs

sqlx supports embedded structs for joining tables:

```go
type Address struct {
    Street string `db:"street"`
    City   string `db:"city"`
    State  string `db:"state"`
}

type UserWithAddress struct {
    User               // embedded struct
    Address `db:"addr" // prefix for address columns
}

// You can also use explicit prefixes:
type OrderWithUser struct {
    OrderID   int64     `db:"order_id"`
    Total     float64   `db:"total"`
    UserName  string    `db:"user_name"`
    UserEmail string    `db:"user_email"`
}

func getOrdersWithUsers(db *sqlx.DB) ([]OrderWithUser, error) {
    var results []OrderWithUser
    err := db.Select(&results, `
        SELECT
            o.id AS order_id,
            o.total,
            u.name AS user_name,
            u.email AS user_email
        FROM orders o
        JOIN users u ON o.user_id = u.id
    `)
    return results, err
}
```

---

## Prepared Statements and SQL Injection Prevention

### SQL Injection: The Threat

```go
// NEVER DO THIS -- vulnerable to SQL injection
func unsafeQuery(db *sql.DB, userInput string) error {
    query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userInput)
    // If userInput = "'; DROP TABLE users; --" ... disaster.
    _, err := db.Query(query)
    return err
}
```

### Parameterized Queries (The Solution)

```go
// ALWAYS use parameterized queries
func safeQuery(db *sql.DB, ctx context.Context, name string) (*sql.Rows, error) {
    return db.QueryContext(ctx,
        "SELECT * FROM users WHERE name = $1", name)
}
```

The database driver sends the query and parameters separately. The database engine treats
parameters as **data**, never as **SQL code**. There is no way to "break out" of a parameter.

### Prepared Statements

A prepared statement is parsed, analyzed, and planned by the database once, then executed
many times with different parameters.

```go
func bulkCheckUsers(ctx context.Context, db *sql.DB, names []string) error {
    // Prepare once
    stmt, err := db.PrepareContext(ctx,
        "SELECT id, email FROM users WHERE name = $1")
    if err != nil {
        return fmt.Errorf("preparing statement: %w", err)
    }
    defer stmt.Close()

    // Execute many times
    for _, name := range names {
        var id int64
        var email string
        err := stmt.QueryRowContext(ctx, name).Scan(&id, &email)
        if err == sql.ErrNoRows {
            fmt.Printf("User %s not found\n", name)
            continue
        }
        if err != nil {
            return fmt.Errorf("querying user %s: %w", name, err)
        }
        fmt.Printf("Found user %s: id=%d, email=%s\n", name, id, email)
    }
    return nil
}
```

> **Tip:** pgx automatically caches prepared statements at the connection level, so
> you often don't need to explicitly prepare them. Each unique query string is prepared
> once per connection and reused.

### Dynamic Column/Table Names

Parameters only work for **values**, not for identifiers (table names, column names).
For dynamic identifiers, use a whitelist:

```go
var allowedSortColumns = map[string]bool{
    "name":       true,
    "email":      true,
    "created_at": true,
}

func listUsersSorted(ctx context.Context, db *sql.DB, sortCol, sortDir string) ([]User, error) {
    // Whitelist the column name
    if !allowedSortColumns[sortCol] {
        return nil, fmt.Errorf("invalid sort column: %s", sortCol)
    }

    // Whitelist the direction
    if sortDir != "ASC" && sortDir != "DESC" {
        sortDir = "ASC"
    }

    // Safe to interpolate because values are whitelisted
    query := fmt.Sprintf(
        "SELECT id, name, email, created_at FROM users ORDER BY %s %s",
        sortCol, sortDir)

    rows, err := db.QueryContext(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    // ... scan rows
    return nil, nil
}
```

### Using pgx QuoteIdentifier

```go
import "github.com/jackc/pgx/v5"

// pgx provides safe identifier quoting
func dynamicTable(ctx context.Context, pool *pgxpool.Pool, tableName string) error {
    // QuoteIdentifier wraps in double quotes and escapes internal quotes
    safeTable := pgx.Identifier{tableName}.Sanitize()
    query := fmt.Sprintf("SELECT count(*) FROM %s", safeTable)

    var count int64
    return pool.QueryRow(ctx, query).Scan(&count)
}
```

---

## Transactions

Transactions group multiple SQL statements into an atomic unit: either all succeed
(COMMIT) or none take effect (ROLLBACK).

### Basic Transaction Pattern

```go
func transferFunds(ctx context.Context, db *sql.DB, fromID, toID int64, amount float64) error {
    // Begin transaction
    tx, err := db.BeginTx(ctx, nil) // nil = default isolation level
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }

    // Defer rollback. If tx.Commit() is called first, Rollback is a no-op.
    defer tx.Rollback()

    // Debit the source account
    result, err := tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1`,
        amount, fromID)
    if err != nil {
        return fmt.Errorf("debiting account %d: %w", fromID, err)
    }

    rows, _ := result.RowsAffected()
    if rows == 0 {
        return fmt.Errorf("insufficient funds in account %d", fromID)
    }

    // Credit the destination account
    _, err = tx.ExecContext(ctx,
        `UPDATE accounts SET balance = balance + $1 WHERE id = $2`,
        amount, toID)
    if err != nil {
        return fmt.Errorf("crediting account %d: %w", toID, err)
    }

    // Insert audit record
    _, err = tx.ExecContext(ctx,
        `INSERT INTO transfers (from_id, to_id, amount, created_at)
         VALUES ($1, $2, $3, NOW())`,
        fromID, toID, amount)
    if err != nil {
        return fmt.Errorf("recording transfer: %w", err)
    }

    // Commit the transaction
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }

    return nil
}
```

> **Tip:** The `defer tx.Rollback()` pattern is safe because calling Rollback on an
> already-committed transaction returns `sql.ErrTxDone`, which we ignore.

### Transaction Helper Function

To avoid repeating the begin/commit/rollback boilerplate:

```go
// WithTx runs fn within a transaction. If fn returns an error, the transaction
// is rolled back. Otherwise, it is committed.
func WithTx(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, opts)
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }

    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            panic(p) // re-throw after rollback
        }
    }()

    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("rolling back: %v (original error: %w)", rbErr, err)
        }
        return err
    }

    return tx.Commit()
}

// Usage:
func transferFundsV2(ctx context.Context, db *sql.DB, fromID, toID int64, amount float64) error {
    return WithTx(ctx, db, nil, func(tx *sql.Tx) error {
        _, err := tx.ExecContext(ctx,
            `UPDATE accounts SET balance = balance - $1
             WHERE id = $2 AND balance >= $1`, amount, fromID)
        if err != nil {
            return err
        }

        _, err = tx.ExecContext(ctx,
            `UPDATE accounts SET balance = balance + $1
             WHERE id = $2`, amount, toID)
        return err
    })
}
```

### Isolation Levels

```go
func readCommitted(ctx context.Context, db *sql.DB) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelReadCommitted, // default for PostgreSQL
        ReadOnly:  false,
    })
    if err != nil {
        return err
    }
    defer tx.Rollback()
    // ...
    return tx.Commit()
}

func serializable(ctx context.Context, db *sql.DB) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable, // strongest isolation
    })
    if err != nil {
        return err
    }
    defer tx.Rollback()
    // ...
    return tx.Commit()
}
```

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Use Case |
|----------------|-------------|---------------------|---------------|----------|
| `ReadUncommitted` | Possible | Possible | Possible | Rarely used |
| `ReadCommitted` | No | Possible | Possible | Default for PostgreSQL |
| `RepeatableRead` | No | No | Possible | Consistent reads within tx |
| `Serializable` | No | No | No | Financial, inventory |

### Nested Transactions with Savepoints

SQL does not support true nested transactions, but PostgreSQL savepoints provide similar
functionality.

```go
func processOrderWithSavepoints(ctx context.Context, db *sql.DB, orderID int64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Main operation: update order status
    _, err = tx.ExecContext(ctx,
        `UPDATE orders SET status = 'processing' WHERE id = $1`, orderID)
    if err != nil {
        return err
    }

    // Savepoint: try to send notification (non-critical)
    _, err = tx.ExecContext(ctx, "SAVEPOINT send_notification")
    if err != nil {
        return err
    }

    _, err = tx.ExecContext(ctx,
        `INSERT INTO notifications (order_id, type, sent_at)
         VALUES ($1, 'processing', NOW())`, orderID)
    if err != nil {
        // Notification failed -- rollback to savepoint, but continue
        log.Printf("notification insert failed: %v, rolling back savepoint", err)
        _, rbErr := tx.ExecContext(ctx, "ROLLBACK TO SAVEPOINT send_notification")
        if rbErr != nil {
            return fmt.Errorf("rollback to savepoint: %w", rbErr)
        }
    } else {
        _, err = tx.ExecContext(ctx, "RELEASE SAVEPOINT send_notification")
        if err != nil {
            return err
        }
    }

    // Continue with the main transaction regardless of notification outcome
    _, err = tx.ExecContext(ctx,
        `UPDATE orders SET status = 'processed' WHERE id = $1`, orderID)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

### Transactions with pgx

```go
func pgxTransaction(ctx context.Context, pool *pgxpool.Pool) error {
    // pgx provides a cleaner transaction API
    tx, err := pool.Begin(ctx)
    if err != nil {
        return err
    }
    defer tx.Rollback(ctx)

    _, err = tx.Exec(ctx, `UPDATE accounts SET balance = balance - 100 WHERE id = 1`)
    if err != nil {
        return err
    }

    _, err = tx.Exec(ctx, `UPDATE accounts SET balance = balance + 100 WHERE id = 2`)
    if err != nil {
        return err
    }

    return tx.Commit(ctx)
}

// pgx also has a helper similar to our WithTx:
func pgxTransactionHelper(ctx context.Context, pool *pgxpool.Pool) error {
    return pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
        _, err := tx.Exec(ctx,
            `UPDATE accounts SET balance = balance - 100 WHERE id = 1`)
        if err != nil {
            return err
        }
        _, err = tx.Exec(ctx,
            `UPDATE accounts SET balance = balance + 100 WHERE id = 2`)
        return err
    })
    // Automatically commits on nil error, rolls back on error
}
```

### Transactions with sqlx

```go
func sqlxTransaction(db *sqlx.DB) error {
    tx, err := db.Beginx()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // NamedExec works within transactions
    _, err = tx.NamedExec(
        `INSERT INTO users (name, email) VALUES (:name, :email)`,
        User{Name: "Alice", Email: "alice@example.com"})
    if err != nil {
        return err
    }

    // Get/Select also work
    var u User
    err = tx.Get(&u, "SELECT * FROM users WHERE email = $1", "alice@example.com")
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

---

## Context-Aware Queries

Every database operation should accept a `context.Context` to support timeouts and
cancellation. This prevents runaway queries from consuming resources indefinitely.

### Query Timeouts

```go
func queryWithTimeout(db *sql.DB) ([]User, error) {
    // 3-second deadline for this specific query
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx,
        `SELECT id, name, email FROM users WHERE status = $1`, "active")
    if err != nil {
        // Check if the error is a timeout
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("query timed out after 3 seconds")
        }
        return nil, err
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()
}
```

### Request-Scoped Context in HTTP Handlers

```go
func (a *App) handleGetUsers(w http.ResponseWriter, r *http.Request) {
    // r.Context() is cancelled if the client disconnects.
    // Add a timeout on top of that.
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    users, err := a.listUsers(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected -- do not bother writing response
            return
        }
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "request timed out", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(users)
}
```

### Statement-Level Timeouts via PostgreSQL

You can also set timeouts at the database level:

```go
func queryWithStatementTimeout(ctx context.Context, pool *pgxpool.Pool) error {
    // Per-query statement timeout using PostgreSQL's statement_timeout
    _, err := pool.Exec(ctx, "SET LOCAL statement_timeout = '3s'")
    if err != nil {
        return err
    }

    // This query will be cancelled by PostgreSQL if it exceeds 3 seconds
    rows, err := pool.Query(ctx, "SELECT * FROM large_table WHERE complex_condition")
    if err != nil {
        return err
    }
    defer rows.Close()
    // ...
    return nil
}
```

### Context with pgx Pool Acquire

```go
func acquireWithTimeout(pool *pgxpool.Pool) error {
    // If the pool is exhausted, Acquire blocks until a connection is available
    // or the context is cancelled.
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    conn, err := pool.Acquire(ctx)
    if err != nil {
        return fmt.Errorf("acquiring connection (pool exhausted?): %w", err)
    }
    defer conn.Release()

    // Use conn...
    return nil
}
```

---

## Scanning Complex Types

Real-world databases store more than strings and integers. PostgreSQL in particular supports
JSON, arrays, composite types, enums, and more.

### JSON Columns

```go
import "encoding/json"

// Method 1: Implement sql.Scanner and driver.Valuer
type Metadata map[string]interface{}

func (m *Metadata) Scan(value interface{}) error {
    if value == nil {
        *m = nil
        return nil
    }
    bytes, ok := value.([]byte)
    if !ok {
        return fmt.Errorf("expected []byte, got %T", value)
    }
    return json.Unmarshal(bytes, m)
}

func (m Metadata) Value() (driver.Value, error) {
    if m == nil {
        return nil, nil
    }
    return json.Marshal(m)
}

// Usage:
type Product struct {
    ID       int64    `db:"id"`
    Name     string   `db:"name"`
    Metadata Metadata `db:"metadata"`
}

func getProduct(ctx context.Context, db *sqlx.DB, id int64) (Product, error) {
    var p Product
    err := db.GetContext(ctx, &p,
        `SELECT id, name, metadata FROM products WHERE id = $1`, id)
    return p, err
}

func createProduct(ctx context.Context, db *sqlx.DB, p Product) error {
    _, err := db.NamedExecContext(ctx,
        `INSERT INTO products (name, metadata)
         VALUES (:name, :metadata)`, p)
    return err
}
```

### Typed JSON Columns

```go
// For strongly-typed JSON, embed the JSON struct
type UserPreferences struct {
    Theme       string   `json:"theme"`
    Language    string   `json:"language"`
    Timezone    string   `json:"timezone"`
    Tags        []string `json:"tags"`
}

// JSONColumn is a generic wrapper for JSON scanning
type JSONColumn[T any] struct {
    Val   T
    Valid bool
}

func (j *JSONColumn[T]) Scan(value interface{}) error {
    if value == nil {
        j.Valid = false
        return nil
    }
    j.Valid = true
    bytes, ok := value.([]byte)
    if !ok {
        return fmt.Errorf("expected []byte, got %T", value)
    }
    return json.Unmarshal(bytes, &j.Val)
}

func (j JSONColumn[T]) Value() (driver.Value, error) {
    if !j.Valid {
        return nil, nil
    }
    return json.Marshal(j.Val)
}

// Usage:
type UserProfile struct {
    ID          int64                        `db:"id"`
    Name        string                       `db:"name"`
    Preferences JSONColumn[UserPreferences]  `db:"preferences"`
}
```

### PostgreSQL Arrays with pgx

```go
func arrayExamples(ctx context.Context, pool *pgxpool.Pool) error {
    // Inserting arrays
    tags := []string{"golang", "postgres", "tutorial"}
    _, err := pool.Exec(ctx,
        `INSERT INTO articles (title, tags)
         VALUES ($1, $2)`,
        "SQL in Go", tags) // pgx handles []string -> text[] automatically
    if err != nil {
        return err
    }

    // Reading arrays
    var (
        title    string
        readTags []string
    )
    err = pool.QueryRow(ctx,
        `SELECT title, tags FROM articles WHERE id = $1`, 1,
    ).Scan(&title, &readTags)
    if err != nil {
        return err
    }

    fmt.Printf("Title: %s, Tags: %v\n", title, readTags)

    // Querying with array operators
    rows, err := pool.Query(ctx,
        `SELECT title FROM articles WHERE tags @> $1`, // contains
        []string{"golang"})
    if err != nil {
        return err
    }
    defer rows.Close()

    // Using ANY with arrays
    rows2, err := pool.Query(ctx,
        `SELECT title FROM articles WHERE $1 = ANY(tags)`, "golang")
    if err != nil {
        return err
    }
    defer rows2.Close()

    return nil
}
```

### PostgreSQL Arrays with database/sql (pq.Array)

```go
import "github.com/lib/pq"

func arrayWithPq(ctx context.Context, db *sql.DB) error {
    // Writing arrays
    tags := []string{"go", "sql"}
    _, err := db.ExecContext(ctx,
        `INSERT INTO articles (title, tags) VALUES ($1, $2)`,
        "Article Title", pq.Array(tags))
    if err != nil {
        return err
    }

    // Reading arrays
    var readTags []string
    err = db.QueryRowContext(ctx,
        `SELECT tags FROM articles WHERE id = $1`, 1,
    ).Scan(pq.Array(&readTags))
    return err
}
```

### Custom Enum Types

```go
// PostgreSQL: CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'delivered');

type OrderStatus string

const (
    OrderPending   OrderStatus = "pending"
    OrderShipped   OrderStatus = "shipped"
    OrderDelivered OrderStatus = "delivered"
)

func (s *OrderStatus) Scan(value interface{}) error {
    str, ok := value.(string)
    if !ok {
        bytes, ok := value.([]byte)
        if !ok {
            return fmt.Errorf("expected string or []byte, got %T", value)
        }
        str = string(bytes)
    }

    switch OrderStatus(str) {
    case OrderPending, OrderShipped, OrderDelivered:
        *s = OrderStatus(str)
        return nil
    default:
        return fmt.Errorf("invalid order status: %s", str)
    }
}

func (s OrderStatus) Value() (driver.Value, error) {
    return string(s), nil
}

type Order struct {
    ID     int64       `db:"id"`
    Status OrderStatus `db:"status"`
}
```

### UUID Columns

```go
import "github.com/google/uuid"

// uuid.UUID already implements sql.Scanner and driver.Valuer
type Account struct {
    ID    uuid.UUID `db:"id"`
    Name  string    `db:"name"`
}

func createAccount(ctx context.Context, db *sqlx.DB) (Account, error) {
    a := Account{
        ID:   uuid.New(),
        Name: "Test Account",
    }
    _, err := db.NamedExecContext(ctx,
        `INSERT INTO accounts (id, name) VALUES (:id, :name)`, a)
    return a, err
}

// With pgx, uuid support is built in:
func pgxUUID(ctx context.Context, pool *pgxpool.Pool) error {
    id := uuid.New()
    _, err := pool.Exec(ctx,
        `INSERT INTO accounts (id, name) VALUES ($1, $2)`,
        id, "Test Account")
    return err
}
```

### Composite Types and Row Scanning

```go
// Scanning joined results into nested structs
type OrderDetail struct {
    OrderID     int64       `db:"order_id"`
    OrderStatus string      `db:"order_status"`
    ProductName string      `db:"product_name"`
    Quantity    int         `db:"quantity"`
    Price       float64     `db:"price"`
}

func getOrderDetails(ctx context.Context, db *sqlx.DB, orderID int64) ([]OrderDetail, error) {
    var details []OrderDetail
    err := db.SelectContext(ctx, &details, `
        SELECT
            o.id       AS order_id,
            o.status   AS order_status,
            p.name     AS product_name,
            oi.quantity,
            oi.price
        FROM orders o
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE o.id = $1
    `, orderID)
    return details, err
}
```

---

## Database Migrations

Migrations are versioned SQL scripts that evolve your database schema over time. They are
essential for reproducible deployments and team collaboration.

### golang-migrate

[golang-migrate](https://github.com/golang-migrate/migrate) is the most popular migration
tool in Go.

**Installation:**

```bash
# CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# As a library
go get github.com/golang-migrate/migrate/v4
```

**Creating Migrations:**

```bash
# Creates two files: 000001_create_users.up.sql and 000001_create_users.down.sql
migrate create -ext sql -dir migrations -seq create_users
```

**Migration Files:**

```sql
-- migrations/000001_create_users.up.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/000001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

```sql
-- migrations/000002_create_orders.up.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

-- migrations/000002_create_orders.down.sql
DROP TABLE IF EXISTS orders;
```

**Running Migrations from CLI:**

```bash
# Apply all pending migrations
migrate -database "postgres://user:pass@localhost/mydb?sslmode=disable" \
        -path migrations up

# Roll back the last migration
migrate -database "postgres://user:pass@localhost/mydb?sslmode=disable" \
        -path migrations down 1

# Go to a specific version
migrate -database "postgres://user:pass@localhost/mydb?sslmode=disable" \
        -path migrations goto 3

# Check current version
migrate -database "postgres://user:pass@localhost/mydb?sslmode=disable" \
        -path migrations version
```

**Running Migrations Programmatically:**

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func runMigrations(databaseURL string) error {
    m, err := migrate.New(
        "file://migrations",
        databaseURL,
    )
    if err != nil {
        return fmt.Errorf("creating migrator: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("running migrations: %w", err)
    }

    version, dirty, err := m.Version()
    if err != nil {
        return fmt.Errorf("getting version: %w", err)
    }

    log.Printf("Migration version: %d, dirty: %v", version, dirty)
    return nil
}

// Embedding migrations in the binary (Go 1.16+):
import "embed"

//go:embed migrations/*.sql
var migrationsFS embed.FS

func runEmbeddedMigrations(databaseURL string) error {
    source, err := iofs.New(migrationsFS, "migrations")
    if err != nil {
        return err
    }

    m, err := migrate.NewWithSourceInstance("iofs", source, databaseURL)
    if err != nil {
        return err
    }
    defer m.Close()

    return m.Up()
}
```

### goose

[goose](https://github.com/pressly/goose) is another popular migration tool that supports
both SQL and Go-based migrations.

**Installation:**

```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
```

**Creating Migrations:**

```bash
goose -dir migrations create add_user_roles sql
```

**Migration File (goose format):**

```sql
-- migrations/20240101120000_add_user_roles.sql

-- +goose Up
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE user_roles (
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    role_id INT REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

INSERT INTO roles (name) VALUES ('admin'), ('editor'), ('viewer');

-- +goose Down
DROP TABLE IF EXISTS user_roles;
DROP TABLE IF EXISTS roles;
```

**Go-based Migrations (for complex data transformations):**

```go
package migrations

import (
    "context"
    "database/sql"

    "github.com/pressly/goose/v3"
)

func init() {
    goose.AddMigrationContext(upBackfillUserNames, downBackfillUserNames)
}

func upBackfillUserNames(ctx context.Context, tx *sql.Tx) error {
    // Complex migration logic that requires Go code
    rows, err := tx.QueryContext(ctx,
        `SELECT id, first_name, last_name FROM users WHERE display_name IS NULL`)
    if err != nil {
        return err
    }
    defer rows.Close()

    stmt, err := tx.PrepareContext(ctx,
        `UPDATE users SET display_name = $1 WHERE id = $2`)
    if err != nil {
        return err
    }
    defer stmt.Close()

    for rows.Next() {
        var id int64
        var first, last string
        if err := rows.Scan(&id, &first, &last); err != nil {
            return err
        }
        displayName := fmt.Sprintf("%s %s", first, last)
        if _, err := stmt.ExecContext(ctx, displayName, id); err != nil {
            return err
        }
    }
    return rows.Err()
}

func downBackfillUserNames(ctx context.Context, tx *sql.Tx) error {
    _, err := tx.ExecContext(ctx,
        `UPDATE users SET display_name = NULL WHERE display_name IS NOT NULL`)
    return err
}
```

**Running goose Programmatically:**

```go
import "github.com/pressly/goose/v3"

func runGooseMigrations(db *sql.DB) error {
    goose.SetDialect("postgres")

    if err := goose.Up(db, "migrations"); err != nil {
        return fmt.Errorf("running goose migrations: %w", err)
    }

    current, err := goose.GetDBVersion(db)
    if err != nil {
        return err
    }
    log.Printf("Current migration version: %d", current)
    return nil
}
```

### Atlas

[Atlas](https://atlasgo.io/) takes a declarative approach -- you describe the desired state,
and Atlas computes the diff.

```hcl
// schema.hcl -- desired state
schema "public" {
}

table "users" {
  schema = schema.public
  column "id" {
    type = bigserial
  }
  column "name" {
    type = varchar(255)
    null = false
  }
  column "email" {
    type = varchar(255)
    null = false
  }
  primary_key {
    columns = [column.id]
  }
  index "idx_users_email" {
    columns = [column.email]
    unique  = true
  }
}
```

```bash
# Generate migration from desired state vs current DB
atlas migrate diff create_users \
  --to file://schema.hcl \
  --dev-url "docker://postgres/15"

# Apply migrations
atlas migrate apply --url "postgres://user:pass@localhost/mydb?sslmode=disable"

# Inspect current schema
atlas schema inspect --url "postgres://user:pass@localhost/mydb?sslmode=disable"
```

### Comparison of Migration Tools

| Feature | golang-migrate | goose | Atlas |
|---------|---------------|-------|-------|
| Approach | Imperative (up/down) | Imperative (up/down) | Declarative + imperative |
| Go migrations | No | Yes | Yes |
| Embedded FS | Yes | Yes | Yes |
| Auto-rollback | No | No | Yes (with --dev-url) |
| Schema diff | No | No | Yes |
| CI/CD linting | No | No | Yes |
| Complexity | Low | Low | Medium |

---

## Migration Best Practices

### Idempotent Migrations

Migrations should be safe to run more than once (in case of partial failures):

```sql
-- GOOD: Idempotent
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_users_name ON users(name);

ALTER TABLE users ADD COLUMN IF NOT EXISTS email VARCHAR(255);

-- BAD: Will fail on re-run
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

### Reversible Migrations

Always write down migrations. Every `up` should have a corresponding `down`:

```sql
-- Up: Add column with default
ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL DEFAULT 'active';

-- Down: Remove column
ALTER TABLE users DROP COLUMN IF EXISTS status;
```

> **Warning:** Some operations are inherently irreversible (dropping a table with data,
> removing a column). In those cases, the down migration should recreate the structure
> but acknowledge that data is lost.

### Zero-Downtime Migrations

Certain DDL operations lock tables and can cause downtime. Here are patterns to avoid that:

**Adding a Column (Safe):**

```sql
-- Step 1: Add column as nullable (no lock, instant in PostgreSQL 11+)
ALTER TABLE users ADD COLUMN phone VARCHAR(50);

-- Step 2: Backfill in batches (application-level, not in migration)
-- Step 3: Add NOT NULL constraint (in a later migration, after backfill)
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

**Adding an Index (Safe with CONCURRENTLY):**

```sql
-- BAD: Locks the table for the duration of index creation
CREATE INDEX idx_orders_status ON orders(status);

-- GOOD: Non-blocking index creation
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
```

> **Warning:** `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. With
> golang-migrate, use the `x-no-transaction` annotation. With goose, use
> `-- +goose NO TRANSACTION`.

```sql
-- goose format:
-- +goose Up
-- +goose NO TRANSACTION
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_status ON orders(status);

-- +goose Down
-- +goose NO TRANSACTION
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_status;
```

**Renaming a Column (Multi-Step):**

```
Step 1: Add new column
Step 2: Dual-write to both columns (application change)
Step 3: Backfill new column from old column
Step 4: Read from new column (application change)
Step 5: Stop writing to old column (application change)
Step 6: Drop old column
```

**Changing Column Type (Multi-Step):**

```sql
-- Step 1: Add new column with desired type
ALTER TABLE users ADD COLUMN age_int INTEGER;

-- Step 2: Backfill (application or batch job)
UPDATE users SET age_int = CAST(age_text AS INTEGER) WHERE age_int IS NULL;

-- Step 3: Swap reads to new column (application change)
-- Step 4: Drop old column
ALTER TABLE users DROP COLUMN age_text;
ALTER TABLE users RENAME COLUMN age_int TO age;
```

### Migration Naming Conventions

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_orders.up.sql
├── 000002_create_orders.down.sql
├── 000003_add_user_phone.up.sql
├── 000003_add_user_phone.down.sql
├── 000004_add_orders_status_index.up.sql
└── 000004_add_orders_status_index.down.sql
```

Guidelines:
- Use sequential numbering (`000001`, `000002`) or timestamps (`20240101120000`)
- Descriptive names: `create_X`, `add_X_to_Y`, `remove_X_from_Y`, `alter_X`
- One logical change per migration
- Keep migrations small and focused

### Locking and Advisory Locks

In production, multiple instances may try to run migrations simultaneously. Use advisory locks:

```go
func runMigrationsWithLock(ctx context.Context, db *sql.DB) error {
    // Acquire PostgreSQL advisory lock
    const lockID = 123456789 // arbitrary unique ID for your app
    var acquired bool
    err := db.QueryRowContext(ctx,
        "SELECT pg_try_advisory_lock($1)", lockID).Scan(&acquired)
    if err != nil {
        return err
    }
    if !acquired {
        log.Println("Another instance is running migrations, skipping")
        return nil
    }
    defer db.ExecContext(ctx, "SELECT pg_advisory_unlock($1)", lockID)

    // Run migrations
    return runMigrations(db)
}
```

---

## Query Builders

Query builders construct SQL programmatically. They are useful when queries have many
optional filters or dynamic conditions.

### Squirrel

[squirrel](https://github.com/Masterminds/squirrel) is the most popular query builder
for Go.

```go
import sq "github.com/Masterminds/squirrel"

// Use PostgreSQL dollar-sign placeholders
var psql = sq.StatementBuilder.PlaceholderFormat(sq.Dollar)

// Simple SELECT
func listActiveUsers(ctx context.Context, db *sql.DB) ([]User, error) {
    query, args, err := psql.
        Select("id", "name", "email", "created_at").
        From("users").
        Where(sq.Eq{"status": "active"}).
        OrderBy("created_at DESC").
        Limit(100).
        ToSql()
    if err != nil {
        return nil, err
    }

    // query: SELECT id, name, email, created_at FROM users
    //        WHERE status = $1 ORDER BY created_at DESC LIMIT 100
    // args:  ["active"]

    rows, err := db.QueryContext(ctx, query, args...)
    // ... scan rows
    return nil, err
}

// Dynamic filters
type UserFilter struct {
    Name     *string
    Email    *string
    Status   *string
    MinAge   *int
    MaxAge   *int
    Tags     []string
    SortBy   string
    SortDir  string
    Page     int
    PageSize int
}

func searchUsers(ctx context.Context, db *sql.DB, f UserFilter) ([]User, error) {
    q := psql.
        Select("id", "name", "email", "age", "status").
        From("users")

    // Conditionally add WHERE clauses
    if f.Name != nil {
        q = q.Where(sq.Like{"name": "%" + *f.Name + "%"})
    }
    if f.Email != nil {
        q = q.Where(sq.Eq{"email": *f.Email})
    }
    if f.Status != nil {
        q = q.Where(sq.Eq{"status": *f.Status})
    }
    if f.MinAge != nil {
        q = q.Where(sq.GtOrEq{"age": *f.MinAge})
    }
    if f.MaxAge != nil {
        q = q.Where(sq.LtOrEq{"age": *f.MaxAge})
    }
    if len(f.Tags) > 0 {
        q = q.Where(sq.Eq{"status": f.Tags}) // IN clause
    }

    // Sorting with whitelist
    validSorts := map[string]bool{
        "name": true, "email": true, "created_at": true,
    }
    if validSorts[f.SortBy] {
        dir := "ASC"
        if f.SortDir == "DESC" {
            dir = "DESC"
        }
        q = q.OrderBy(f.SortBy + " " + dir)
    }

    // Pagination
    if f.PageSize <= 0 {
        f.PageSize = 20
    }
    if f.Page <= 0 {
        f.Page = 1
    }
    q = q.Limit(uint64(f.PageSize)).
        Offset(uint64((f.Page - 1) * f.PageSize))

    query, args, err := q.ToSql()
    if err != nil {
        return nil, fmt.Errorf("building query: %w", err)
    }

    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    // ... scan
    return nil, nil
}

// INSERT with squirrel
func createUserSq(ctx context.Context, db *sql.DB, name, email string) (int64, error) {
    query, args, err := psql.
        Insert("users").
        Columns("name", "email").
        Values(name, email).
        Suffix("RETURNING id").
        ToSql()
    if err != nil {
        return 0, err
    }

    var id int64
    err = db.QueryRowContext(ctx, query, args...).Scan(&id)
    return id, err
}

// UPDATE with squirrel
func updateUserSq(ctx context.Context, db *sql.DB, id int64, name, email string) error {
    query, args, err := psql.
        Update("users").
        Set("name", name).
        Set("email", email).
        Set("updated_at", sq.Expr("NOW()")).
        Where(sq.Eq{"id": id}).
        ToSql()
    if err != nil {
        return err
    }

    _, err = db.ExecContext(ctx, query, args...)
    return err
}

// DELETE with squirrel
func deleteUserSq(ctx context.Context, db *sql.DB, id int64) error {
    query, args, err := psql.
        Delete("users").
        Where(sq.Eq{"id": id}).
        ToSql()
    if err != nil {
        return err
    }

    _, err = db.ExecContext(ctx, query, args...)
    return err
}

// Complex conditions
func complexQuery(ctx context.Context, db *sql.DB) error {
    query, args, err := psql.
        Select("*").
        From("orders").
        Where(sq.And{
            sq.Eq{"status": "shipped"},
            sq.Or{
                sq.GtOrEq{"total": 100},
                sq.Eq{"priority": "high"},
            },
        }).
        ToSql()
    // WHERE (status = $1 AND (total >= $2 OR priority = $3))

    _, err = db.QueryContext(ctx, query, args...)
    return err
}
```

### goqu

[goqu](https://github.com/doug-martin/goqu) is a more feature-rich query builder with
dialect support.

```go
import (
    "github.com/doug-martin/goqu/v9"
    _ "github.com/doug-martin/goqu/v9/dialect/postgres"
)

var dialect = goqu.Dialect("postgres")

func goquExamples(ctx context.Context, db *sql.DB) error {
    // SELECT
    query, args, err := dialect.
        From("users").
        Select("id", "name", "email").
        Where(
            goqu.C("status").Eq("active"),
            goqu.C("age").Gte(18),
        ).
        Order(goqu.C("name").Asc()).
        Limit(10).
        ToSQL()
    if err != nil {
        return err
    }
    fmt.Println(query, args)

    // INSERT with RETURNING
    query, args, err = dialect.
        Insert("users").
        Rows(goqu.Record{
            "name":  "Alice",
            "email": "alice@example.com",
        }).
        Returning("id").
        ToSQL()
    if err != nil {
        return err
    }
    fmt.Println(query, args)

    // Subqueries
    subquery := dialect.
        From("orders").
        Select(goqu.L("DISTINCT user_id")).
        Where(goqu.C("status").Eq("completed"))

    query, args, err = dialect.
        From("users").
        Where(goqu.C("id").In(subquery)).
        ToSQL()
    if err != nil {
        return err
    }
    fmt.Println(query, args)
    // SELECT * FROM "users" WHERE "id" IN (
    //   SELECT DISTINCT user_id FROM "orders" WHERE "status" = 'completed'
    // )

    return nil
}
```

### When to Use Query Builders

| Scenario | Recommendation |
|----------|---------------|
| Fixed queries | Write raw SQL (simpler, faster, easier to review) |
| 2-3 optional filters | Raw SQL with conditional string building |
| Many optional filters | Query builder (squirrel, goqu) |
| Report/dashboard queries | Query builder or raw SQL with templates |
| Need DB-specific features | Raw SQL |
| Multi-database support | Query builder with dialect support |

> **Tip:** Do not reach for a query builder for simple CRUD. Raw SQL with parameterized
> queries is perfectly fine and easier to understand. Query builders shine when the SQL
> structure itself is dynamic.

---

## Testing with Databases

### Strategy Overview

| Approach | Speed | Fidelity | Complexity |
|----------|-------|----------|------------|
| Mock interfaces | Fast | Low | Medium |
| SQLite in-memory | Fast | Medium | Medium |
| testcontainers | Slow (first run) | High | Medium |
| Shared test database | Medium | High | Low |
| Transaction rollback | Fast | High | Low |

### testcontainers-go

[testcontainers-go](https://github.com/testcontainers/testcontainers-go) spins up real
Docker containers for tests. This gives you the highest fidelity.

```go
package repository_test

import (
    "context"
    "database/sql"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    _ "github.com/jackc/pgx/v5/stdlib"
)

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    ctx := context.Background()

    // Start a real PostgreSQL container
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("starting postgres container: %v", err)
    }

    // Clean up container when test finishes
    t.Cleanup(func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("terminating container: %v", err)
        }
    })

    // Get connection string
    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("getting connection string: %v", err)
    }

    db, err := sql.Open("pgx", connStr)
    if err != nil {
        t.Fatalf("opening database: %v", err)
    }
    t.Cleanup(func() { db.Close() })

    // Run migrations
    runMigrations(db) // your migration function

    return db
}

func TestCreateUser(t *testing.T) {
    db := setupTestDB(t)
    repo := NewUserRepository(db)

    user, err := repo.Create(context.Background(), "Alice", "alice@test.com")
    if err != nil {
        t.Fatalf("creating user: %v", err)
    }

    if user.ID == 0 {
        t.Error("expected non-zero user ID")
    }
    if user.Name != "Alice" {
        t.Errorf("expected name Alice, got %s", user.Name)
    }
}
```

### Transaction Rollback Pattern

For fast tests that share a database, wrap each test in a transaction that is rolled back:

```go
package repository_test

import (
    "context"
    "database/sql"
    "testing"
)

// testTx creates a transaction that is rolled back after the test.
// This keeps each test isolated without the cost of recreating the database.
func testTx(t *testing.T, db *sql.DB) *sql.Tx {
    t.Helper()

    tx, err := db.BeginTx(context.Background(), nil)
    if err != nil {
        t.Fatalf("beginning transaction: %v", err)
    }

    t.Cleanup(func() {
        tx.Rollback() // always rollback -- test data disappears
    })

    return tx
}

// Interface that both *sql.DB and *sql.Tx satisfy
type DBTX interface {
    ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}

type UserRepository struct {
    db DBTX
}

func NewUserRepository(db DBTX) *UserRepository {
    return &UserRepository{db: db}
}

func TestUserRepository_Create(t *testing.T) {
    db := getSharedTestDB(t) // setup once, reuse across tests
    tx := testTx(t, db)     // isolated transaction per test

    repo := NewUserRepository(tx)

    user, err := repo.Create(context.Background(), "Bob", "bob@test.com")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    // Verify
    got, err := repo.GetByID(context.Background(), user.ID)
    if err != nil {
        t.Fatalf("getting user: %v", err)
    }
    if got.Name != "Bob" {
        t.Errorf("got name %q, want %q", got.Name, "Bob")
    }
}

// After the test, tx.Rollback() is called -- the user "Bob" does not persist.
```

### Test Fixtures

```go
package testutil

import (
    "context"
    "database/sql"
    "testing"
    "time"
)

type Fixtures struct {
    db    DBTX
    Users []User
    Orders []Order
}

func SeedFixtures(t *testing.T, db DBTX) *Fixtures {
    t.Helper()
    ctx := context.Background()

    f := &Fixtures{db: db}

    // Create users
    for _, u := range []struct{ name, email string }{
        {"Alice", "alice@test.com"},
        {"Bob", "bob@test.com"},
        {"Charlie", "charlie@test.com"},
    } {
        var user User
        err := db.QueryRowContext(ctx,
            `INSERT INTO users (name, email, created_at)
             VALUES ($1, $2, $3) RETURNING id, name, email, created_at`,
            u.name, u.email, time.Now(),
        ).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
        if err != nil {
            t.Fatalf("seeding user %s: %v", u.name, err)
        }
        f.Users = append(f.Users, user)
    }

    // Create orders for Alice
    for _, total := range []float64{99.99, 149.50, 299.00} {
        var order Order
        err := db.QueryRowContext(ctx,
            `INSERT INTO orders (user_id, total, status, created_at)
             VALUES ($1, $2, 'pending', $3) RETURNING id`,
            f.Users[0].ID, total, time.Now(),
        ).Scan(&order.ID)
        if err != nil {
            t.Fatalf("seeding order: %v", err)
        }
        f.Orders = append(f.Orders, order)
    }

    return f
}

func TestListOrdersForUser(t *testing.T) {
    db := getSharedTestDB(t)
    tx := testTx(t, db)
    fixtures := SeedFixtures(t, tx)

    repo := NewOrderRepository(tx)
    orders, err := repo.ListByUserID(context.Background(), fixtures.Users[0].ID)
    if err != nil {
        t.Fatalf("listing orders: %v", err)
    }

    if len(orders) != 3 {
        t.Errorf("expected 3 orders, got %d", len(orders))
    }
}
```

### Mocking with Interfaces

For unit tests that should not touch the database at all:

```go
// Define the interface your code depends on
type UserStore interface {
    GetByID(ctx context.Context, id int64) (User, error)
    Create(ctx context.Context, name, email string) (User, error)
    List(ctx context.Context, limit int) ([]User, error)
}

// Real implementation
type PostgresUserStore struct {
    db *sql.DB
}

func (s *PostgresUserStore) GetByID(ctx context.Context, id int64) (User, error) {
    // real implementation
}

// Mock for testing
type MockUserStore struct {
    GetByIDFunc func(ctx context.Context, id int64) (User, error)
    CreateFunc  func(ctx context.Context, name, email string) (User, error)
    ListFunc    func(ctx context.Context, limit int) ([]User, error)
}

func (m *MockUserStore) GetByID(ctx context.Context, id int64) (User, error) {
    return m.GetByIDFunc(ctx, id)
}

func (m *MockUserStore) Create(ctx context.Context, name, email string) (User, error) {
    return m.CreateFunc(ctx, name, email)
}

func (m *MockUserStore) List(ctx context.Context, limit int) ([]User, error) {
    return m.ListFunc(ctx, limit)
}

// Test the handler, not the database
func TestHandleGetUser(t *testing.T) {
    store := &MockUserStore{
        GetByIDFunc: func(ctx context.Context, id int64) (User, error) {
            return User{ID: 1, Name: "Alice", Email: "alice@test.com"}, nil
        },
    }

    handler := NewUserHandler(store)
    // test the HTTP handler with the mock store
}
```

---

## Connection Error Handling and Retries

Database connections can fail due to network issues, server restarts, or failovers.
Robust applications handle these gracefully.

### Identifying Retryable Errors

```go
import (
    "github.com/jackc/pgx/v5/pgconn"
    "errors"
    "net"
)

// isRetryableError determines if a database error is transient and worth retrying.
func isRetryableError(err error) bool {
    if err == nil {
        return false
    }

    // Connection was reset or closed
    if errors.Is(err, sql.ErrConnDone) {
        return true
    }

    // Network errors
    var netErr net.Error
    if errors.As(err, &netErr) {
        return true
    }

    // PostgreSQL-specific error codes
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        switch pgErr.Code {
        case "40001": // serialization_failure
            return true
        case "40P01": // deadlock_detected
            return true
        case "08006": // connection_failure
            return true
        case "08001": // sqlclient_unable_to_establish_sqlconnection
            return true
        case "08004": // sqlserver_rejected_establishment_of_sqlconnection
            return true
        case "57P01": // admin_shutdown
            return true
        case "57P02": // crash_shutdown
            return true
        case "57P03": // cannot_connect_now
            return true
        }
    }

    // Check error message for common patterns
    msg := err.Error()
    retryableMessages := []string{
        "connection refused",
        "connection reset by peer",
        "broken pipe",
        "no connection to the server",
        "unexpected EOF",
        "i/o timeout",
    }
    for _, m := range retryableMessages {
        if strings.Contains(msg, m) {
            return true
        }
    }

    return false
}
```

### Retry with Exponential Backoff

```go
import (
    "math"
    "math/rand"
    "time"
)

type RetryConfig struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    Multiplier  float64
}

var DefaultRetryConfig = RetryConfig{
    MaxAttempts: 3,
    BaseDelay:   100 * time.Millisecond,
    MaxDelay:    5 * time.Second,
    Multiplier:  2.0,
}

func withRetry[T any](ctx context.Context, cfg RetryConfig, operation func() (T, error)) (T, error) {
    var lastErr error
    var zero T

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        result, err := operation()
        if err == nil {
            return result, nil
        }

        if !isRetryableError(err) {
            return zero, err // non-retryable -- fail immediately
        }

        lastErr = err

        if attempt < cfg.MaxAttempts-1 {
            delay := time.Duration(float64(cfg.BaseDelay) * math.Pow(cfg.Multiplier, float64(attempt)))
            if delay > cfg.MaxDelay {
                delay = cfg.MaxDelay
            }

            // Add jitter (0.5x to 1.5x)
            jitter := 0.5 + rand.Float64()
            delay = time.Duration(float64(delay) * jitter)

            log.Printf("Retry attempt %d/%d after %v: %v",
                attempt+1, cfg.MaxAttempts, delay, err)

            select {
            case <-time.After(delay):
                // continue to next attempt
            case <-ctx.Done():
                return zero, ctx.Err()
            }
        }
    }

    return zero, fmt.Errorf("max retries (%d) exceeded: %w", cfg.MaxAttempts, lastErr)
}

// Usage:
func (r *UserRepo) GetUserWithRetry(ctx context.Context, id int64) (User, error) {
    return withRetry(ctx, DefaultRetryConfig, func() (User, error) {
        return r.GetUser(ctx, id)
    })
}
```

### Connection Health Checks

```go
// Startup health check
func waitForDatabase(ctx context.Context, db *sql.DB, maxWait time.Duration) error {
    ctx, cancel := context.WithTimeout(ctx, maxWait)
    defer cancel()

    for {
        err := db.PingContext(ctx)
        if err == nil {
            return nil
        }

        log.Printf("Waiting for database: %v", err)

        select {
        case <-ctx.Done():
            return fmt.Errorf("database not ready after %v: %w", maxWait, ctx.Err())
        case <-time.After(1 * time.Second):
            // try again
        }
    }
}

// Liveness probe for Kubernetes
func (a *App) healthCheckHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    if err := a.db.PingContext(ctx); err != nil {
        http.Error(w, "database unreachable", http.StatusServiceUnavailable)
        return
    }

    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}
```

### Handling PostgreSQL Failover

When using a primary/replica setup or managed services with failover:

```go
func connectWithFailoverAwareness(primaryDSN, replicaDSN string) (*pgxpool.Pool, *pgxpool.Pool, error) {
    ctx := context.Background()

    primaryPool, err := pgxpool.New(ctx, primaryDSN)
    if err != nil {
        return nil, nil, fmt.Errorf("connecting to primary: %w", err)
    }

    replicaPool, err := pgxpool.New(ctx, replicaDSN)
    if err != nil {
        primaryPool.Close()
        return nil, nil, fmt.Errorf("connecting to replica: %w", err)
    }

    return primaryPool, replicaPool, nil
}

type DB struct {
    primary *pgxpool.Pool
    replica *pgxpool.Pool
}

// Route reads to replica, writes to primary
func (db *DB) QueryReadOnly(ctx context.Context, sql string, args ...any) (pgx.Rows, error) {
    return db.replica.Query(ctx, sql, args...)
}

func (db *DB) Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error) {
    return db.primary.Exec(ctx, sql, args...)
}
```

---

## Monitoring and Logging Queries

### Query Logging with pgx

pgx has a built-in tracer interface for comprehensive query logging:

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/tracelog"
)

// Custom logger adapter
type PgxLogger struct {
    logger *slog.Logger
}

func (l *PgxLogger) Log(ctx context.Context, level tracelog.LogLevel, msg string, data map[string]interface{}) {
    attrs := make([]slog.Attr, 0, len(data))
    for k, v := range data {
        attrs = append(attrs, slog.Any(k, v))
    }

    switch level {
    case tracelog.LogLevelTrace, tracelog.LogLevelDebug:
        l.logger.LogAttrs(ctx, slog.LevelDebug, msg, attrs...)
    case tracelog.LogLevelInfo:
        l.logger.LogAttrs(ctx, slog.LevelInfo, msg, attrs...)
    case tracelog.LogLevelWarn:
        l.logger.LogAttrs(ctx, slog.LevelWarn, msg, attrs...)
    case tracelog.LogLevelError:
        l.logger.LogAttrs(ctx, slog.LevelError, msg, attrs...)
    }
}

func setupPoolWithLogging() (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(os.Getenv("DATABASE_URL"))
    if err != nil {
        return nil, err
    }

    logger := slog.Default()
    config.ConnConfig.Tracer = &tracelog.TraceLog{
        Logger:   &PgxLogger{logger: logger},
        LogLevel: tracelog.LogLevelInfo,
    }

    return pgxpool.NewWithConfig(context.Background(), config)
}
```

### Custom Query Tracer with Timing

```go
import "github.com/jackc/pgx/v5"

type QueryTracer struct {
    logger *slog.Logger
}

// TraceQueryStart is called before query execution
func (t *QueryTracer) TraceQueryStart(ctx context.Context, conn *pgx.Conn, data pgx.TraceQueryStartData) context.Context {
    return context.WithValue(ctx, "queryStart", time.Now())
}

// TraceQueryEnd is called after query execution
func (t *QueryTracer) TraceQueryEnd(ctx context.Context, conn *pgx.Conn, data pgx.TraceQueryEndData) {
    start, ok := ctx.Value("queryStart").(time.Time)
    if !ok {
        return
    }

    duration := time.Since(start)

    // Log slow queries
    level := slog.LevelDebug
    if duration > 100*time.Millisecond {
        level = slog.LevelWarn
    }
    if duration > 1*time.Second {
        level = slog.LevelError
    }

    t.logger.LogAttrs(ctx, level, "query executed",
        slog.String("sql", data.SQL),
        slog.Duration("duration", duration),
        slog.String("commandTag", data.CommandTag.String()),
    )

    if data.Err != nil {
        t.logger.Error("query error",
            slog.String("sql", data.SQL),
            slog.String("error", data.Err.Error()),
            slog.Duration("duration", duration),
        )
    }

    // Update Prometheus metrics
    queryDuration.WithLabelValues(extractQueryType(data.SQL)).Observe(duration.Seconds())
}

func extractQueryType(sql string) string {
    sql = strings.TrimSpace(strings.ToUpper(sql))
    switch {
    case strings.HasPrefix(sql, "SELECT"):
        return "SELECT"
    case strings.HasPrefix(sql, "INSERT"):
        return "INSERT"
    case strings.HasPrefix(sql, "UPDATE"):
        return "UPDATE"
    case strings.HasPrefix(sql, "DELETE"):
        return "DELETE"
    default:
        return "OTHER"
    }
}
```

### Prometheus Metrics for Database Operations

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    queryDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "db_query_duration_seconds",
            Help:    "Duration of database queries",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        },
        []string{"query_type"},
    )

    queryErrors = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "db_query_errors_total",
            Help: "Total number of database query errors",
        },
        []string{"query_type", "error_code"},
    )

    poolStats = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "db_pool_connections",
            Help: "Database connection pool statistics",
        },
        []string{"state"}, // "open", "in_use", "idle"
    )
)

// Periodically export pool stats
func exportPoolStats(db *sql.DB) {
    go func() {
        ticker := time.NewTicker(10 * time.Second)
        defer ticker.Stop()
        for range ticker.C {
            stats := db.Stats()
            poolStats.WithLabelValues("open").Set(float64(stats.OpenConnections))
            poolStats.WithLabelValues("in_use").Set(float64(stats.InUse))
            poolStats.WithLabelValues("idle").Set(float64(stats.Idle))
        }
    }()
}
```

### Middleware for Query Instrumentation

```go
// A wrapper that instruments all database calls
type InstrumentedDB struct {
    db     *sql.DB
    logger *slog.Logger
}

func (idb *InstrumentedDB) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    start := time.Now()
    rows, err := idb.db.QueryContext(ctx, query, args...)
    duration := time.Since(start)

    idb.logger.LogAttrs(ctx, slog.LevelDebug, "SQL query",
        slog.String("query", query),
        slog.Duration("duration", duration),
        slog.Bool("error", err != nil),
    )

    queryDuration.WithLabelValues("SELECT").Observe(duration.Seconds())
    if err != nil {
        queryErrors.WithLabelValues("SELECT", errorCode(err)).Inc()
    }

    return rows, err
}

func (idb *InstrumentedDB) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
    start := time.Now()
    result, err := idb.db.ExecContext(ctx, query, args...)
    duration := time.Since(start)

    queryType := extractQueryType(query)
    idb.logger.LogAttrs(ctx, slog.LevelDebug, "SQL exec",
        slog.String("query", query),
        slog.String("type", queryType),
        slog.Duration("duration", duration),
        slog.Bool("error", err != nil),
    )

    queryDuration.WithLabelValues(queryType).Observe(duration.Seconds())
    if err != nil {
        queryErrors.WithLabelValues(queryType, errorCode(err)).Inc()
    }

    return result, err
}
```

---

## N+1 Query Problem and Solutions

The N+1 problem occurs when code executes 1 query to fetch a list of N items, then N
additional queries to fetch related data for each item. For 100 users with orders,
that is 101 queries instead of 2.

### The Problem

```go
// BAD: N+1 queries
func listUsersWithOrders_BAD(ctx context.Context, db *sql.DB) ([]UserWithOrders, error) {
    // Query 1: Get all users
    rows, err := db.QueryContext(ctx, "SELECT id, name FROM users LIMIT 100")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []UserWithOrders
    for rows.Next() {
        var u UserWithOrders
        if err := rows.Scan(&u.ID, &u.Name); err != nil {
            return nil, err
        }

        // Queries 2..N+1: Get orders for EACH user (this is the N+1!)
        orderRows, err := db.QueryContext(ctx,
            "SELECT id, total, status FROM orders WHERE user_id = $1", u.ID)
        if err != nil {
            return nil, err
        }
        for orderRows.Next() {
            var o Order
            if err := orderRows.Scan(&o.ID, &o.Total, &o.Status); err != nil {
                orderRows.Close()
                return nil, err
            }
            u.Orders = append(u.Orders, o)
        }
        orderRows.Close()

        results = append(results, u)
    }
    return results, rows.Err()
}
```

### Solution 1: JOIN

Fetch everything in a single query:

```go
type UserOrderRow struct {
    UserID      int64          `db:"user_id"`
    UserName    string         `db:"user_name"`
    OrderID     sql.NullInt64  `db:"order_id"`
    OrderTotal  sql.NullFloat64 `db:"order_total"`
    OrderStatus sql.NullString `db:"order_status"`
}

func listUsersWithOrders_JOIN(ctx context.Context, db *sqlx.DB) ([]UserWithOrders, error) {
    var rows []UserOrderRow
    err := db.SelectContext(ctx, &rows, `
        SELECT
            u.id   AS user_id,
            u.name AS user_name,
            o.id   AS order_id,
            o.total AS order_total,
            o.status AS order_status
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        ORDER BY u.id, o.id
        LIMIT 1000
    `)
    if err != nil {
        return nil, err
    }

    // Group rows by user
    userMap := make(map[int64]*UserWithOrders)
    var userOrder []int64 // preserve order

    for _, row := range rows {
        u, exists := userMap[row.UserID]
        if !exists {
            u = &UserWithOrders{
                ID:   row.UserID,
                Name: row.UserName,
            }
            userMap[row.UserID] = u
            userOrder = append(userOrder, row.UserID)
        }
        if row.OrderID.Valid {
            u.Orders = append(u.Orders, Order{
                ID:     row.OrderID.Int64,
                Total:  row.OrderTotal.Float64,
                Status: row.OrderStatus.String,
            })
        }
    }

    results := make([]UserWithOrders, 0, len(userOrder))
    for _, id := range userOrder {
        results = append(results, *userMap[id])
    }
    return results, nil
}
```

> **Warning:** JOINs can produce large result sets with duplicated user data (each user
> row is repeated for every order). For users with many related rows, this wastes bandwidth.

### Solution 2: Two Queries with IN Clause

Fetch users first, then batch-fetch all orders in a second query:

```go
func listUsersWithOrders_TwoQueries(ctx context.Context, db *sqlx.DB) ([]UserWithOrders, error) {
    // Query 1: Get users
    var users []UserWithOrders
    err := db.SelectContext(ctx, &users,
        "SELECT id, name FROM users ORDER BY id LIMIT 100")
    if err != nil {
        return nil, err
    }

    if len(users) == 0 {
        return users, nil
    }

    // Collect user IDs
    userIDs := make([]int64, len(users))
    userMap := make(map[int64]*UserWithOrders, len(users))
    for i := range users {
        userIDs[i] = users[i].ID
        userMap[users[i].ID] = &users[i]
    }

    // Query 2: Get ALL orders for these users in one query
    query, args, err := sqlx.In(
        "SELECT id, user_id, total, status FROM orders WHERE user_id IN (?)",
        userIDs)
    if err != nil {
        return nil, err
    }
    query = db.Rebind(query)

    var orders []struct {
        Order
        UserID int64 `db:"user_id"`
    }
    err = db.SelectContext(ctx, &orders, query, args...)
    if err != nil {
        return nil, err
    }

    // Assign orders to users
    for _, o := range orders {
        if u, ok := userMap[o.UserID]; ok {
            u.Orders = append(u.Orders, o.Order)
        }
    }

    return users, nil
}
```

This approach is 2 queries regardless of N. It is simple, efficient, and avoids the
data duplication problem of JOINs.

### Solution 3: pgx Batch Queries

Use pgx's batch feature to send multiple queries in parallel:

```go
func listUsersWithOrders_Batch(ctx context.Context, pool *pgxpool.Pool) ([]UserWithOrders, error) {
    // First, get users
    userRows, err := pool.Query(ctx,
        "SELECT id, name FROM users ORDER BY id LIMIT 100")
    if err != nil {
        return nil, err
    }

    users, err := pgx.CollectRows(userRows, pgx.RowToStructByName[UserWithOrders])
    if err != nil {
        return nil, err
    }

    if len(users) == 0 {
        return users, nil
    }

    // Batch all order queries into one round-trip
    batch := &pgx.Batch{}
    for _, u := range users {
        batch.Queue(
            "SELECT id, total, status FROM orders WHERE user_id = $1 ORDER BY id",
            u.ID,
        )
    }

    br := pool.SendBatch(ctx, batch)
    defer br.Close()

    for i := range users {
        rows, err := br.Query()
        if err != nil {
            return nil, fmt.Errorf("batch query %d: %w", i, err)
        }

        orders, err := pgx.CollectRows(rows, pgx.RowToStructByName[Order])
        if err != nil {
            return nil, fmt.Errorf("collecting orders for user %d: %w", users[i].ID, err)
        }
        users[i].Orders = orders
    }

    return users, nil
}
```

### Solution 4: PostgreSQL Array Aggregation

Use PostgreSQL's `array_agg` or `json_agg` to collapse related rows:

```go
type UserWithOrderJSON struct {
    ID     int64           `db:"id"`
    Name   string          `db:"name"`
    Orders json.RawMessage `db:"orders_json"`
}

func listUsersWithOrders_JsonAgg(ctx context.Context, db *sqlx.DB) ([]UserWithOrders, error) {
    var rawUsers []UserWithOrderJSON
    err := db.SelectContext(ctx, &rawUsers, `
        SELECT
            u.id,
            u.name,
            COALESCE(
                json_agg(
                    json_build_object(
                        'id', o.id,
                        'total', o.total,
                        'status', o.status
                    )
                ) FILTER (WHERE o.id IS NOT NULL),
                '[]'
            ) AS orders_json
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        GROUP BY u.id, u.name
        ORDER BY u.id
        LIMIT 100
    `)
    if err != nil {
        return nil, err
    }

    // Parse JSON arrays
    results := make([]UserWithOrders, len(rawUsers))
    for i, ru := range rawUsers {
        results[i].ID = ru.ID
        results[i].Name = ru.Name
        if err := json.Unmarshal(ru.Orders, &results[i].Orders); err != nil {
            return nil, fmt.Errorf("unmarshaling orders for user %d: %w", ru.ID, err)
        }
    }
    return results, nil
}
```

### N+1 Solutions Comparison

| Solution | Queries | Network Round-Trips | Complexity | Best For |
|----------|---------|-------------------|------------|----------|
| N+1 (bad) | N+1 | N+1 | Low | Never |
| JOIN | 1 | 1 | Medium | Few relations, small data |
| Two queries + IN | 2 | 2 | Low | Most cases |
| pgx Batch | N+1 | 1 | Medium | pgx users, many relations |
| json_agg | 1 | 1 | Medium | PostgreSQL, complex nesting |

### Detecting N+1 in Tests

```go
// QueryCounter wraps a DB and counts queries -- useful for test assertions
type QueryCounter struct {
    db    *sql.DB
    count int64
    mu    sync.Mutex
}

func (qc *QueryCounter) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    qc.mu.Lock()
    qc.count++
    qc.mu.Unlock()
    return qc.db.QueryContext(ctx, query, args...)
}

func (qc *QueryCounter) Count() int64 {
    qc.mu.Lock()
    defer qc.mu.Unlock()
    return qc.count
}

func (qc *QueryCounter) Reset() {
    qc.mu.Lock()
    qc.count = 0
    qc.mu.Unlock()
}

// In tests:
func TestNoNPlusOne(t *testing.T) {
    counter := &QueryCounter{db: testDB}
    repo := NewUserRepository(counter)

    counter.Reset()
    users, err := repo.ListWithOrders(context.Background(), 50)
    if err != nil {
        t.Fatal(err)
    }

    queryCount := counter.Count()
    if queryCount > 3 {
        t.Errorf("expected at most 3 queries, got %d (possible N+1)", queryCount)
    }
    t.Logf("Fetched %d users with %d queries", len(users), queryCount)
}
```

---

## Summary

### Quick Reference: Which Package to Use

| Need | Package | Notes |
|------|---------|-------|
| Simple PostgreSQL access | pgx (standalone) | Best performance, PostgreSQL-specific |
| Driver-agnostic access | database/sql + driver | Standard, works with any DB |
| Reduced boilerplate | sqlx | StructScan, NamedExec, IN clause |
| Bulk inserts | pgx COPY | 10-50x faster than individual INSERTs |
| Multiple related queries | pgx Batch | Single round-trip |
| Dynamic filters | squirrel or goqu | Clean query construction |
| Migrations | golang-migrate or goose | Version-controlled schema changes |
| Integration tests | testcontainers-go | Real database, high fidelity |

### Key Takeaways

1. **Always set connection pool limits.** Unbounded pools cause "too many connections" errors in production. Set `MaxOpenConns` based on your database's capacity divided by the number of application instances.

2. **Always close Rows.** Use `defer rows.Close()` immediately after `db.Query`. Forgetting this leaks connections and eventually deadlocks your pool.

3. **Always check `rows.Err()`.** The `for rows.Next()` loop can exit early due to errors that are only surfaced by `rows.Err()`.

4. **Use parameterized queries.** Never interpolate user input into SQL strings. Use `$1, $2` placeholders with PostgreSQL, `?` with MySQL.

5. **Use context for timeouts.** Every database call should accept a `context.Context`. Set per-query timeouts to prevent runaway queries from consuming connections.

6. **Prefer pgx for PostgreSQL.** It is faster (binary protocol), actively maintained, and supports COPY, batching, LISTEN/NOTIFY, and more.

7. **Use transactions for multi-statement operations.** The `defer tx.Rollback()` + explicit `Commit()` pattern is safe and clean.

8. **Write reversible, idempotent migrations.** Use `IF NOT EXISTS`, `IF EXISTS`, and `CONCURRENTLY` for zero-downtime deploys.

9. **Watch for N+1 queries.** Use JOINs, batch-fetching with IN clauses, or pgx batches instead of querying in a loop.

10. **Monitor your connection pool.** Export `db.Stats()` as metrics. Alert on high `WaitCount` and `WaitDuration`.

### Further Reading

- [pgx documentation](https://github.com/jackc/pgx)
- [sqlx documentation](https://github.com/jmoiron/sqlx)
- [golang-migrate](https://github.com/golang-migrate/migrate)
- [goose](https://github.com/pressly/goose)
- [Atlas](https://atlasgo.io/)
- [squirrel](https://github.com/Masterminds/squirrel)
- [testcontainers-go](https://github.com/testcontainers/testcontainers-go)
- [PostgreSQL Wire Protocol](https://www.postgresql.org/docs/current/protocol.html)
- [database/sql tutorial](https://go.dev/doc/database/)
