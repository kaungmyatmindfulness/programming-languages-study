# Chapter 26: Repository Pattern & Database Layer

> **Level:** Real-World Production | **Prerequisites:** Chapters 1-22 (Go fundamentals), Chapter 16 (Context), basic SQL knowledge
> **Goal:** Build a production-grade database layer using PostgreSQL, pgx, the repository pattern, Goose migrations, and battle-tested patterns extracted from real Go API projects.

This chapter is not a toy example. Everything here comes from patterns used in production Go APIs -- connection pooling, upserts, JSONB columns, SQL triggers, structured error handling, transactions, and mock repositories for testing. If you have built backends with Node.js using Prisma, Knex, or TypeORM, you will see how Go achieves the same goals with more explicit control and less magic.

---

## Table of Contents

1. [Why the Repository Pattern?](#1-why-the-repository-pattern)
2. [Database Connection with pgx](#2-database-connection-with-pgx)
3. [Connection Pool Configuration](#3-connection-pool-configuration)
4. [Database Migrations with Goose](#4-database-migrations-with-goose)
5. [Defining Domain Models](#5-defining-domain-models)
6. [Defining Repository Interfaces](#6-defining-repository-interfaces)
7. [Implementing CRUD Operations](#7-implementing-crud-operations)
8. [Upsert Pattern (ON CONFLICT)](#8-upsert-pattern-on-conflict)
9. [Handling PostgreSQL Errors](#9-handling-postgresql-errors)
10. [Working with JSONB Columns](#10-working-with-jsonb-columns)
11. [Context-Aware Queries](#11-context-aware-queries)
12. [Row Scanning Patterns](#12-row-scanning-patterns)
13. [SQL Triggers for Auto-Updating Timestamps](#13-sql-triggers-for-auto-updating-timestamps)
14. [Health Check Queries](#14-health-check-queries)
15. [Transactions](#15-transactions)
16. [Mock Repositories for Testing](#16-mock-repositories-for-testing)
17. [Real-World Example: Complete Repository Layer](#17-real-world-example-complete-repository-layer)
18. [Key Takeaways](#18-key-takeaways)
19. [Practice Exercises](#19-practice-exercises)

---

## 1. Why the Repository Pattern?

### The Problem

Imagine you are building an API. Your HTTP handler needs user data. The simplest approach is to put the SQL query directly in the handler:

```go
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := r.PathValue("id")

    var user User
    err := db.QueryRow(r.Context(),
        "SELECT id, email, name FROM users WHERE id = $1", userID,
    ).Scan(&user.ID, &user.Email, &user.Name)

    if err != nil {
        http.Error(w, "not found", 404)
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

This works. But now you have problems:

1. **Testability** -- To test this handler, you need a running PostgreSQL database. Your CI pipeline needs Docker. Your tests take seconds instead of microseconds.
2. **Duplication** -- Another handler also needs to fetch users. You copy-paste the SQL. Now you have two places to update when the schema changes.
3. **Coupling** -- Your HTTP layer knows about SQL syntax, database drivers, and PostgreSQL-specific features. Changing from PostgreSQL to MySQL means rewriting every handler.
4. **Separation of concerns** -- Your handler is doing two jobs: handling HTTP and querying the database. When it breaks, you do not know which part failed.

### The Solution: Repository Pattern

The repository pattern introduces a layer between your business logic and your data source. Your handler does not know (or care) whether data comes from PostgreSQL, MongoDB, an in-memory map, or a CSV file.

```
┌─────────────────────────────────────────────────────┐
│                   HTTP Handler                       │
│            (knows about HTTP only)                   │
└───────────────────────┬─────────────────────────────┘
                        │ calls method on interface
┌───────────────────────▼─────────────────────────────┐
│              Repository Interface                    │
│         (defines WHAT operations exist)              │
│                                                      │
│   GetUser(ctx, id) (*User, error)                   │
│   CreateUser(ctx, *User) error                      │
│   UpdateUser(ctx, *User) error                      │
│   DeleteUser(ctx, id) error                         │
└───────────────────────┬─────────────────────────────┘
                        │ implemented by
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  PostgreSQL  │ │   In-Memory  │ │    Mock       │
│  Repository  │ │  Repository  │ │  (for tests)  │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Node.js Comparison

In Node.js, you achieve similar separation with different tools:

```
┌──────────────────────────────────────────────────┐
│              Node.js / Express                    │
├──────────────────────────────────────────────────┤
│                                                   │
│  Prisma       →  Schema-driven ORM, auto-gen     │
│                  client. Magic. Hard to debug     │
│                  complex queries.                 │
│                                                   │
│  TypeORM      →  Decorator-based ORM. Active      │
│                  Record or Repository pattern.    │
│                  Heavy, complex configuration.    │
│                                                   │
│  Knex         →  Query builder. More control.     │
│                  You still write SQL-like code.   │
│                                                   │
│  Raw pg       →  Direct SQL. Maximum control.     │
│                  Most similar to Go + pgx.        │
│                                                   │
├──────────────────────────────────────────────────┤
│              Go + pgx                             │
├──────────────────────────────────────────────────┤
│                                                   │
│  You write SQL. You control every query. The      │
│  interface defines your contract. The compiler    │
│  checks your implementation. No magic. No         │
│  runtime reflection for queries. Maximum          │
│  performance, maximum clarity.                    │
│                                                   │
└──────────────────────────────────────────────────┘
```

**Prisma example (Node.js):**

```typescript
// Prisma -- the ORM writes SQL for you
const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { posts: true },
});
```

**Go + pgx equivalent:**

```go
// Go -- you write the SQL, you control everything
func (r *UserRepository) GetUser(ctx context.Context, id string) (*User, error) {
    var user User
    err := r.db.QueryRow(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users WHERE id = $1`, id,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.Picture,
        &user.CreatedAt, &user.UpdatedAt)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return &user, nil
}
```

The Go approach requires more code, but you gain:
- **No hidden queries.** Prisma might generate N+1 queries behind `include`. Your pgx code runs exactly the SQL you wrote.
- **Compile-time safety.** Your interface ensures every repository method is implemented. TypeScript only checks types, not that your Prisma queries are valid (that happens at runtime).
- **Debuggability.** When something is slow, you look at the SQL. There is no ORM layer generating unexpected joins.

### The Three Core Benefits

**1. Testability**

```go
// In production, use the real database
userRepo := repository.NewUserRepository(pgPool)

// In tests, use a mock that returns canned data
userRepo := &MockUserStore{
    GetUserFunc: func(ctx context.Context, id string) (*User, error) {
        return &User{ID: id, Email: "test@example.com"}, nil
    },
}
```

Your handler tests run in microseconds with zero infrastructure.

**2. Swappable Implementations**

Need to add a Redis cache in front of PostgreSQL? Create a `CachedUserRepository` that implements the same interface, checks Redis first, and falls back to the PostgreSQL repository. The handler does not change.

**3. Clear Boundaries**

Your handler code reads like a story: "Get the user. If not found, return 404. Otherwise, return the user as JSON." The SQL details are hidden behind a clean method call.

---

## 2. Database Connection with pgx

### Why pgx Instead of database/sql?

Go's standard library includes `database/sql`, which provides a generic database interface. It works with any database via drivers. However, `pgx` (specifically `jackc/pgx/v5`) is the gold standard for PostgreSQL in Go because:

1. **Native PostgreSQL protocol** -- pgx speaks the PostgreSQL wire protocol directly, bypassing the `database/sql` abstraction layer.
2. **Full PostgreSQL type support** -- JSONB, arrays, hstore, enums, composite types, inet, cidr, macaddr all work natively.
3. **Connection pooling** -- `pgxpool` provides a production-ready connection pool.
4. **COPY protocol** -- Bulk data loading at maximum speed.
5. **Listen/Notify** -- Native PostgreSQL pub/sub support.
6. **Performance** -- Benchmarks consistently show pgx is faster than database/sql + pq driver.

```
┌──────────────────────────────────────────────┐
│          Go Database Driver Landscape         │
├──────────────────────────────────────────────┤
│                                               │
│  database/sql + pq                            │
│    ✓ Standard library interface               │
│    ✗ No native JSONB, arrays, etc.           │
│    ✗ Extra abstraction layer overhead         │
│                                               │
│  database/sql + pgx/stdlib                    │
│    ✓ Standard library interface               │
│    ✓ pgx as the underlying driver             │
│    ✗ Still limited by database/sql types      │
│                                               │
│  pgx native (recommended)                     │
│    ✓ Full PostgreSQL type support             │
│    ✓ Best performance                         │
│    ✓ Connection pool (pgxpool)                │
│    ✓ COPY, Listen/Notify, etc.               │
│    ✗ PostgreSQL only (not swappable to MySQL) │
│                                               │
└──────────────────────────────────────────────┘
```

### Installing pgx

```bash
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
```

### Basic Connection

The simplest way to connect:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    ctx := context.Background()

    // DSN (Data Source Name) format
    dsn := "postgres://username:password@localhost:5432/mydb?sslmode=disable"

    pool, err := pgxpool.New(ctx, dsn)
    if err != nil {
        log.Fatalf("Unable to create connection pool: %v", err)
    }
    defer pool.Close()

    // Verify the connection works
    if err := pool.Ping(ctx); err != nil {
        log.Fatalf("Unable to ping database: %v", err)
    }

    fmt.Println("Connected to PostgreSQL!")
}
```

### DSN Format

The DSN (Data Source Name) is the connection string. PostgreSQL supports two formats:

```
# URL format (most common)
postgres://username:password@host:port/database?param=value

# Keyword/value format
host=localhost port=5432 user=myuser password=secret dbname=mydb sslmode=disable

# Examples:
postgres://admin:secret@localhost:5432/krafty?sslmode=disable
postgres://admin:secret@db.example.com:5432/krafty?sslmode=require
postgres://admin:secret@localhost:5432/krafty?sslmode=disable&timezone=UTC
```

Common query parameters:
- `sslmode=disable` -- No SSL (development only)
- `sslmode=require` -- Require SSL (production)
- `sslmode=verify-full` -- Require SSL + verify server certificate (most secure)
- `timezone=UTC` -- Set session timezone
- `search_path=myschema` -- Set default schema

### Production Connection Factory

In a production application, you want a reusable function that creates configured connection pools. This is the pattern used in real Go APIs:

```go
package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

// PoolConfig holds connection pool configuration.
// These values should come from your application config (env vars, config file, etc.).
type PoolConfig struct {
    MaxOpenConns  int
    MaxIdleConns  int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

// DefaultPoolConfig returns sensible defaults for a production connection pool.
func DefaultPoolConfig() *PoolConfig {
    return &PoolConfig{
        MaxOpenConns:    25,
        MaxIdleConns:    10,
        ConnMaxLifetime: 5 * time.Minute,
        ConnMaxIdleTime: 1 * time.Minute,
    }
}

// New creates a new PostgreSQL connection pool with the given DSN and pool configuration.
// It pings the database to verify connectivity before returning.
func New(dsn string, poolCfg *PoolConfig) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(dsn)
    if err != nil {
        return nil, fmt.Errorf("parsing database DSN: %w", err)
    }

    // Apply pool configuration
    config.MaxConns = int32(poolCfg.MaxOpenConns)
    config.MinConns = int32(poolCfg.MaxIdleConns)
    config.MaxConnLifetime = poolCfg.ConnMaxLifetime
    config.MaxConnIdleTime = poolCfg.ConnMaxIdleTime

    // Create the pool
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("creating connection pool: %w", err)
    }

    // Verify connectivity
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("pinging database: %w", err)
    }

    return pool, nil
}
```

### Using the Connection Factory

```go
package main

import (
    "fmt"
    "log"
    "os"

    "yourproject/internal/database"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        dsn = "postgres://admin:secret@localhost:5432/krafty?sslmode=disable"
    }

    poolCfg := database.DefaultPoolConfig()
    // Override from environment if needed
    // poolCfg.MaxOpenConns = 50

    pool, err := database.New(dsn, poolCfg)
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer pool.Close()

    fmt.Println("Database pool created successfully")

    // Pass pool to your repositories, services, etc.
    // userRepo := repository.NewUserRepository(pool)
    // ...
}
```

### Node.js Comparison

In Node.js with the `pg` package, you do something similar but with less explicit pool management:

```javascript
// Node.js with pg
const { Pool } = require('pg');

const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: 25,                    // MaxOpenConns
    idleTimeoutMillis: 60000,   // ConnMaxIdleTime
    connectionTimeoutMillis: 10000,
});

// With Prisma, you don't manage the pool at all
// const prisma = new PrismaClient()
// Prisma manages its own pool internally -- you have less control
```

The Go approach makes pool configuration explicit and visible. With Prisma, the pool exists but is hidden from you. When something goes wrong (connection exhaustion, slow queries), Go's explicit approach makes debugging far easier.

---

## 3. Connection Pool Configuration

### Why Connection Pooling Matters

A database connection is expensive to create. It involves:
1. TCP handshake (1 round trip)
2. TLS handshake if using SSL (2-3 round trips)
3. PostgreSQL authentication (1-2 round trips)
4. Connection setup (setting timezone, search_path, etc.)

This process takes 20-100ms depending on network distance and TLS. If every HTTP request opened a new connection, you would waste 20-100ms before any work begins.

A connection pool maintains a set of open connections that are reused across requests. When your handler needs a database connection, the pool hands over an idle one in microseconds.

```
┌─────────────────────────────────────────────────────────────┐
│                     Connection Pool                          │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │  Conn 1  │  │  Conn 2  │  │  Conn 3  │  │  Conn 4  │  ...│
│  │  (busy)  │  │  (idle)  │  │  (busy)  │  │  (idle)  │     │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                              │
│  MaxConns: 25        MinConns: 10                            │
│  Active: 12          Idle: 13                                │
│                                                              │
│  Request arrives → grab idle conn → execute query → return  │
│  No idle conn? → wait (up to timeout) or create new one     │
│  MaxConns reached? → block until one is returned             │
└─────────────────────────────────────────────────────────────┘
```

### Configuration Parameters Explained

```go
type PoolConfig struct {
    MaxOpenConns    int           // Maximum number of open connections to the database
    MaxIdleConns    int           // Minimum number of idle connections kept open
    ConnMaxLifetime time.Duration // Maximum time a connection can be reused
    ConnMaxIdleTime time.Duration // Maximum time a connection can sit idle
}
```

**MaxOpenConns (pgxpool: MaxConns)** -- Default: 25

The maximum number of connections the pool can open simultaneously. This is your hard limit. If all 25 connections are busy and a 26th request arrives, it blocks until a connection becomes available.

How to choose:
- PostgreSQL's default `max_connections` is 100.
- If you have 4 instances of your service, each needs at most 25 connections (4 x 25 = 100).
- Formula: `max_connections / number_of_service_instances`
- For most services, 25-50 is appropriate.
- Setting this too high causes PostgreSQL to thrash (context switching between connections).
- Setting this too low causes requests to queue up waiting for connections.

**MaxIdleConns (pgxpool: MinConns)** -- Default: 10

The minimum number of connections the pool keeps open even when idle. These are "warm" connections, ready to serve requests immediately.

How to choose:
- Set to your expected steady-state concurrency.
- If your service normally handles 10 concurrent database queries, set MinConns to 10.
- Setting this too high wastes PostgreSQL resources (each idle connection uses ~10MB of PostgreSQL memory).
- Setting this too low means cold-start latency when traffic arrives after idle periods.

**ConnMaxLifetime (pgxpool: MaxConnLifetime)** -- Default: 5 minutes

The maximum amount of time a connection can be reused before it is closed and replaced. This ensures connections do not grow stale.

Why this matters:
- PostgreSQL connections accumulate memory over time (query plan caches, temp buffers).
- Load balancers (like PgBouncer or AWS RDS Proxy) need connections to rotate so they can rebalance.
- DNS changes (like RDS failover) require new connections to resolve to the new IP.
- Setting this to 5 minutes is a good balance between connection reuse and freshness.

**ConnMaxIdleTime (pgxpool: MaxConnIdleTime)** -- Default: 1 minute

The maximum time a connection can sit idle before being closed. This shrinks the pool during low-traffic periods.

Why this matters:
- During off-peak hours, you do not need 25 open connections.
- Each idle connection holds a PostgreSQL backend process (~10MB).
- Setting this to 1 minute means idle connections are reclaimed quickly, but MinConns are always maintained.

### Complete Example with All Settings

```go
package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

// Config represents the full database configuration.
type Config struct {
    DSN             string        `env:"DATABASE_URL" envDefault:"postgres://localhost:5432/mydb"`
    MaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS" envDefault:"25"`
    MaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS" envDefault:"10"`
    ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" envDefault:"5m"`
    ConnMaxIdleTime time.Duration `env:"DB_CONN_MAX_IDLE_TIME" envDefault:"1m"`
}

// NewPool creates a production-ready pgxpool connection pool.
func NewPool(cfg Config) (*pgxpool.Pool, error) {
    poolConfig, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }

    // Pool size
    poolConfig.MaxConns = int32(cfg.MaxOpenConns)
    poolConfig.MinConns = int32(cfg.MaxIdleConns)

    // Connection lifetime
    poolConfig.MaxConnLifetime = cfg.ConnMaxLifetime
    poolConfig.MaxConnIdleTime = cfg.ConnMaxIdleTime

    // Health check interval -- pgxpool will periodically check idle connections
    poolConfig.HealthCheckPeriod = 30 * time.Second

    // Connect with a timeout
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    pool, err := pgxpool.NewWithConfig(ctx, poolConfig)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }

    // Verify
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping: %w", err)
    }

    return pool, nil
}
```

### Monitoring Pool Statistics

pgxpool exposes pool statistics that you should monitor in production:

```go
package main

import (
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func logPoolStats(pool *pgxpool.Pool) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        stats := pool.Stat()
        fmt.Printf("Pool stats: total=%d, idle=%d, in_use=%d, max=%d\n",
            stats.TotalConns(),
            stats.IdleConns(),
            stats.AcquiredConns(),
            stats.MaxConns(),
        )
        fmt.Printf("  acquired_total=%d, empty_acquire=%d, canceled_acquire=%d\n",
            stats.AcquireCount(),
            stats.EmptyAcquireCount(),  // Times we had to create a new conn
            stats.CanceledAcquireCount(), // Times acquisition was canceled
        )
        fmt.Printf("  acquire_duration=%v\n",
            stats.AcquireDuration(),
        )
    }
}
```

**Key metrics to alert on:**
- `EmptyAcquireCount` growing fast -- Your pool is too small, requests are waiting.
- `CanceledAcquireCount` growing -- Requests are timing out waiting for connections. Your pool is exhausted.
- `AcquireDuration` > 100ms -- Connections are taking too long to acquire.

---

## 4. Database Migrations with Goose

### Why Migrations?

Database migrations are version-controlled changes to your database schema. Instead of manually running SQL scripts, migrations track which changes have been applied and ensure your database schema matches your code.

```
Migration 001: CREATE TABLE users
Migration 002: ADD COLUMN email TO users
Migration 003: CREATE INDEX ON users(email)
Migration 004: CREATE TABLE sessions
```

Every developer and every environment (dev, staging, production) applies the same migrations in the same order, ensuring consistent schemas.

### Why Goose?

There are several migration tools in Go:

```
┌──────────────────────────────────────────────────────────┐
│              Go Migration Tools Comparison                │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  golang-migrate/migrate                                   │
│    ✓ Most popular                                        │
│    ✓ CLI + library                                       │
│    ✗ Up/Down in separate files                           │
│    ✗ No embedded SQL support (needs workaround)          │
│                                                           │
│  pressly/goose (v3)                                       │
│    ✓ Up/Down in single file                              │
│    ✓ Native go:embed support                             │
│    ✓ Run as library (no CLI needed)                      │
│    ✓ SQL and Go migrations                               │
│    ✓ Used in production by many companies                │
│                                                           │
│  atlas (ariga/atlas)                                      │
│    ✓ Declarative migrations                              │
│    ✓ Automatic migration planning                        │
│    ✗ More complex, newer                                 │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

We use Goose because it supports embedding SQL files directly in the Go binary with `go:embed`, allowing migrations to run automatically on application startup with zero external dependencies.

### Installing Goose

```bash
# As a library (recommended -- embed in your app)
go get github.com/pressly/goose/v3

# As a CLI tool (for manual operations)
go install github.com/pressly/goose/v3/cmd/goose@latest
```

### Migration File Format

Goose SQL files use special comments to mark the Up and Down sections:

```sql
-- migrations/001_create_users.sql

-- +goose Up
CREATE TABLE IF NOT EXISTS users (
    id         TEXT PRIMARY KEY,
    email      TEXT NOT NULL UNIQUE,
    name       TEXT,
    picture    TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- +goose Down
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

```sql
-- migrations/002_create_sessions.sql

-- +goose Up
CREATE TABLE IF NOT EXISTS sessions (
    id           TEXT PRIMARY KEY,
    user_id      TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token TEXT NOT NULL,
    user_agent   TEXT,
    ip_address   TEXT,
    expires_at   TIMESTAMPTZ NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);

-- +goose Down
DROP TABLE IF EXISTS sessions;
```

```sql
-- migrations/003_create_projects.sql

-- +goose Up
CREATE TABLE IF NOT EXISTS projects (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    description TEXT,
    settings    JSONB NOT NULL DEFAULT '{}',
    is_archived BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_projects_user_id ON projects(user_id);
CREATE INDEX idx_projects_user_id_archived ON projects(user_id, is_archived);

-- +goose Down
DROP TABLE IF EXISTS projects;
```

### Embedding Migrations with go:embed

The `go:embed` directive (introduced in Go 1.16) lets you embed files directly into your Go binary. This means your migration SQL files are compiled into the executable -- no need to ship them separately.

```go
package database

import (
    "embed"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/jackc/pgx/v5/stdlib"
    "github.com/pressly/goose/v3"
)

//go:embed migrations/*.sql
var migrations embed.FS

// RunMigrations applies all pending database migrations.
// It uses go:embed to include migration files in the binary.
func RunMigrations(pool *pgxpool.Pool) error {
    // Goose needs a *sql.DB, so we convert from pgxpool
    // stdlib.OpenDBFromPool wraps the pgx pool in a database/sql compatible interface
    db := stdlib.OpenDBFromPool(pool)
    defer db.Close()

    // Set the migration file system to our embedded files
    goose.SetBaseFS(migrations)

    // Set the database dialect
    if err := goose.SetDialect("postgres"); err != nil {
        return fmt.Errorf("set dialect: %w", err)
    }

    // Run all pending migrations
    if err := goose.Up(db, "migrations"); err != nil {
        return fmt.Errorf("run migrations: %w", err)
    }

    return nil
}
```

### Running Migrations on Application Startup

In production Go APIs, migrations typically run automatically when the application starts. This ensures the database schema is always in sync with the deployed code.

```go
package main

import (
    "fmt"
    "log"
    "os"

    "yourproject/internal/database"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")

    // Create connection pool
    pool, err := database.NewPool(database.Config{
        DSN:          dsn,
        MaxOpenConns: 25,
        MaxIdleConns: 10,
    })
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer pool.Close()

    // Run migrations before starting the server
    fmt.Println("Running database migrations...")
    if err := database.RunMigrations(pool); err != nil {
        log.Fatalf("Failed to run migrations: %v", err)
    }
    fmt.Println("Migrations complete.")

    // Now start your HTTP server, wire up repositories, etc.
    // ...
}
```

### Using the Goose CLI for Manual Operations

For local development, you can also use the Goose CLI directly:

```bash
# Create a new migration
goose -dir ./internal/database/migrations create add_user_preferences sql

# Apply all pending migrations
goose -dir ./internal/database/migrations postgres "postgres://admin:secret@localhost:5432/krafty?sslmode=disable" up

# Roll back the most recent migration
goose -dir ./internal/database/migrations postgres "postgres://admin:secret@localhost:5432/krafty?sslmode=disable" down

# Check migration status
goose -dir ./internal/database/migrations postgres "postgres://admin:secret@localhost:5432/krafty?sslmode=disable" status

# Roll back all migrations (careful!)
goose -dir ./internal/database/migrations postgres "postgres://admin:secret@localhost:5432/krafty?sslmode=disable" reset
```

### Node.js Comparison

In Node.js, you might use Prisma migrations or Knex migrations:

```javascript
// Prisma: schema-first approach
// You define the schema, Prisma generates the migration SQL
// prisma/schema.prisma:
// model User {
//   id        String   @id @default(uuid())
//   email     String   @unique
//   name      String?
//   createdAt DateTime @default(now())
// }
//
// $ npx prisma migrate dev --name create-users

// Knex: code-first approach (more similar to Goose)
// migrations/001_create_users.js:
exports.up = function(knex) {
    return knex.schema.createTable('users', (table) => {
        table.uuid('id').primary();
        table.string('email').notNullable().unique();
        table.string('name');
        table.timestamps(true, true);
    });
};

exports.down = function(knex) {
    return knex.schema.dropTable('users');
};
```

Goose with embedded SQL is similar to Knex migrations, but uses raw SQL instead of a JavaScript query builder. The Go approach has the advantage of including migration files directly in the binary -- no need to copy migration files to the deployment. The binary is self-contained.

---

## 5. Defining Domain Models

### Struct Design for Database Models

Domain models are Go structs that represent your database tables. They serve double duty as both the database row representation and the JSON response format.

```go
package store

import "time"

// User represents a user in the system.
// Tags control both JSON serialization and database scanning.
type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    Name      *string   `json:"name,omitempty"`       // Nullable: pointer
    Picture   *string   `json:"picture,omitempty"`    // Nullable: pointer
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}

// Session represents an active user session.
type Session struct {
    ID           string    `json:"id"`
    UserID       string    `json:"userId"`
    RefreshToken string    `json:"-"`                  // Never expose in JSON
    UserAgent    *string   `json:"userAgent,omitempty"`
    IPAddress    *string   `json:"ipAddress,omitempty"`
    ExpiresAt    time.Time `json:"expiresAt"`
    CreatedAt    time.Time `json:"createdAt"`
    UpdatedAt    time.Time `json:"updatedAt"`
}

// Project represents a user's project.
type Project struct {
    ID          string          `json:"id"`
    UserID      string          `json:"userId"`
    Name        string          `json:"name"`
    Description *string         `json:"description,omitempty"`
    Settings    ProjectSettings `json:"settings"`      // JSONB column
    IsArchived  bool            `json:"isArchived"`
    CreatedAt   time.Time       `json:"createdAt"`
    UpdatedAt   time.Time       `json:"updatedAt"`
}

// ProjectSettings is stored as JSONB in PostgreSQL.
type ProjectSettings struct {
    Theme       string `json:"theme,omitempty"`
    Language    string `json:"language,omitempty"`
    IsPublic    bool   `json:"isPublic"`
    MaxFileSize int    `json:"maxFileSize,omitempty"`
}
```

### Nullable Fields: Pointers vs sql.Null Types

In PostgreSQL, a column can be `NULL`. In Go, a `string` cannot be `nil` -- its zero value is `""`. You have two strategies for handling nullable columns:

**Strategy 1: Use pointers (recommended with pgx)**

```go
type User struct {
    Name    *string   `json:"name,omitempty"`    // nil = NULL, non-nil = has value
    Picture *string   `json:"picture,omitempty"`
}

// Creating a user with a name
name := "Alice"
user := &User{Name: &name}

// Creating a user without a name
user := &User{Name: nil}  // Will be NULL in the database
```

**Strategy 2: Use sql.Null types (database/sql approach)**

```go
import "database/sql"

type User struct {
    Name    sql.NullString `json:"name"`
    Picture sql.NullString `json:"picture"`
}

// This is verbose and does not serialize well to JSON
// sql.NullString{String: "Alice", Valid: true} → {"String":"Alice","Valid":true}
// You need a custom JSON marshaler to get {"name": "Alice"}
```

**Why pointers win with pgx:**
1. pgx handles pointer scanning natively. `Scan(&user.Name)` works correctly when Name is `*string`.
2. JSON serialization works correctly. `omitempty` with a pointer omits the field when nil.
3. The code is more readable. `*string` is clearly "this might be nil."

### JSON Tags: What They Do

```go
type User struct {
    ID        string    `json:"id"`                   // Go: ID → JSON: "id"
    Email     string    `json:"email"`                // Go: Email → JSON: "email"
    Name      *string   `json:"name,omitempty"`       // Omit from JSON if nil
    Password  string    `json:"-"`                    // NEVER include in JSON
    CreatedAt time.Time `json:"createdAt"`            // camelCase for JS clients
}
```

- `json:"fieldName"` -- Maps Go field name to JSON field name.
- `json:"fieldName,omitempty"` -- Omit this field if it is the zero value (nil for pointers, "" for strings, 0 for numbers, false for bools).
- `json:"-"` -- Never include this field in JSON output. Critical for passwords, tokens, internal IDs.

### Helper Functions for Pointer Types

Working with pointer fields is slightly awkward, so create helpers:

```go
package ptr

// String returns a pointer to the given string.
func String(s string) *string {
    return &s
}

// StringValue returns the value of a string pointer, or "" if nil.
func StringValue(s *string) string {
    if s == nil {
        return ""
    }
    return *s
}

// Int returns a pointer to the given int.
func Int(i int) *int {
    return &i
}

// Bool returns a pointer to the given bool.
func Bool(b bool) *bool {
    return &b
}
```

Usage:

```go
user := &User{
    ID:    "user-123",
    Email: "alice@example.com",
    Name:  ptr.String("Alice"),       // Much cleaner than: name := "Alice"; &name
}
```

### Node.js Comparison

In TypeScript, nullable fields are handled with union types:

```typescript
// TypeScript
interface User {
    id: string;
    email: string;
    name: string | null;       // Nullable
    picture?: string;          // Optional (may be undefined)
    createdAt: Date;
    updatedAt: Date;
}

// With Prisma, nullable fields come from the schema
// model User {
//   name    String?   // ? means nullable → generates string | null
// }
```

Go's `*string` is equivalent to TypeScript's `string | null`. The `omitempty` tag is similar to TypeScript's `?` for optional fields in JSON output.

---

## 6. Defining Repository Interfaces

### Interface-First Design

In Go, you define the interface where it is **used**, not where it is **implemented**. This is a fundamental difference from Java and TypeScript, where interfaces are defined alongside the implementation.

```go
package store

import (
    "context"
)

// UserStore defines all operations for user data access.
// This interface is defined in the "store" package (where models live),
// NOT in the repository package (where the implementation lives).
type UserStore interface {
    // UpsertUser creates or updates a user.
    // If a user with the same ID exists, their email, name, and picture are updated.
    UpsertUser(ctx context.Context, user *User) error

    // GetUser retrieves a single user by ID.
    // Returns an error if the user does not exist.
    GetUser(ctx context.Context, userID string) (*User, error)

    // UpdatePreferences updates a user's display preferences.
    UpdatePreferences(ctx context.Context, userID string, prefs *UserPreferences) error

    // DeleteUser permanently removes a user and all associated data.
    DeleteUser(ctx context.Context, userID string) error
}

// SessionStore defines all operations for session data access.
type SessionStore interface {
    // CreateSession inserts a new session.
    CreateSession(ctx context.Context, session *Session) error

    // GetSession retrieves a session by ID.
    GetSession(ctx context.Context, sessionID string) (*Session, error)

    // GetSessionsByUser retrieves all sessions for a given user.
    GetSessionsByUser(ctx context.Context, userID string) ([]*Session, error)

    // DeleteSession removes a session by ID.
    DeleteSession(ctx context.Context, sessionID string) error

    // DeleteExpiredSessions removes all sessions that have expired.
    DeleteExpiredSessions(ctx context.Context) (int64, error)
}

// ProjectStore defines all operations for project data access.
type ProjectStore interface {
    // CreateProject inserts a new project.
    CreateProject(ctx context.Context, project *Project) error

    // GetProject retrieves a single project by ID.
    GetProject(ctx context.Context, projectID string) (*Project, error)

    // ListProjectsByUser retrieves all projects for a user, with optional filtering.
    ListProjectsByUser(ctx context.Context, userID string, includeArchived bool) ([]*Project, error)

    // UpdateProject updates a project's name, description, and settings.
    UpdateProject(ctx context.Context, project *Project) error

    // DeleteProject permanently removes a project.
    DeleteProject(ctx context.Context, projectID string) error
}

// UserPreferences represents the display preferences for a user.
type UserPreferences struct {
    Theme    string `json:"theme"`
    Language string `json:"language"`
    Timezone string `json:"timezone"`
}
```

### Compile-Time Interface Verification

Go's interfaces are implicit -- a type implements an interface if it has all the required methods. But how do you know if you forgot to implement a method? The compiler will not tell you until someone tries to use the concrete type as the interface.

The solution is a compile-time check:

```go
package repository

import "yourproject/internal/store"

// UserRepository implements store.UserStore using PostgreSQL.
type UserRepository struct {
    db *pgxpool.Pool
}

// Compile-time check: ensure UserRepository implements UserStore.
// If any method is missing, this line will cause a compile error.
var _ store.UserStore = (*UserRepository)(nil)

// This line says:
// "Assign nil (typed as *UserRepository) to a variable of type store.UserStore."
// If *UserRepository does not implement store.UserStore, the compiler rejects it.
// The variable is blank (_) so it is discarded -- this exists only for the check.
```

**WHY this pattern?**

Without it, you might have a typo in a method signature:

```go
// WRONG: parameter is "userId" not "userID" -- does NOT implement the interface
func (r *UserRepository) GetUser(ctx context.Context, userId string) (*store.User, error) {
    // ...
}
```

The code compiles fine until somewhere else in the code you try:

```go
var s store.UserStore = userRepo  // COMPILE ERROR here, far from the bug
```

With `var _ store.UserStore = (*UserRepository)(nil)` in the repository file, the error appears immediately in the repository file, exactly where the missing method should be implemented.

### Combining Interfaces (Storage Aggregation)

For convenience, you can combine all store interfaces into a single `Storage` interface:

```go
package store

// Storage combines all store interfaces into a single interface.
// This makes it easy to pass around a complete data access layer.
type Storage interface {
    UserStore
    SessionStore
    ProjectStore
}
```

And implement it with an aggregate struct:

```go
package repository

import (
    "yourproject/internal/store"
    "github.com/jackc/pgx/v5/pgxpool"
)

// PostgresStorage implements store.Storage by embedding all repositories.
type PostgresStorage struct {
    *UserRepository
    *SessionRepository
    *ProjectRepository
}

var _ store.Storage = (*PostgresStorage)(nil)

// NewPostgresStorage creates a Storage implementation backed by PostgreSQL.
func NewPostgresStorage(pool *pgxpool.Pool) *PostgresStorage {
    return &PostgresStorage{
        UserRepository:    NewUserRepository(pool),
        SessionRepository: NewSessionRepository(pool),
        ProjectRepository: NewProjectRepository(pool),
    }
}
```

### Node.js Comparison

In TypeScript, you define interfaces explicitly:

```typescript
// TypeScript
interface UserStore {
    upsertUser(user: User): Promise<void>;
    getUser(userId: string): Promise<User | null>;
    updatePreferences(userId: string, prefs: UserPreferences): Promise<void>;
    deleteUser(userId: string): Promise<void>;
}

class PostgresUserRepository implements UserStore {
    // TypeScript REQUIRES the "implements" keyword
    // Go does NOT -- the compiler figures it out from the method signatures
}
```

The Go approach (implicit interfaces) is more flexible. You can define a new interface that your existing repository already satisfies, without modifying the repository. This is powerful for testing -- you can define a minimal interface with only the methods your test needs.

---

## 7. Implementing CRUD Operations

### Repository Constructor

Every repository starts with a constructor that takes the database pool:

```go
package repository

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/store"
)

// UserRepository implements store.UserStore using PostgreSQL.
type UserRepository struct {
    db *pgxpool.Pool
}

// NewUserRepository creates a new UserRepository.
func NewUserRepository(db *pgxpool.Pool) *UserRepository {
    return &UserRepository{db: db}
}

// Compile-time interface check.
var _ store.UserStore = (*UserRepository)(nil)
```

### CREATE (Insert)

```go
// CreateUser inserts a new user into the database.
func (r *UserRepository) CreateUser(ctx context.Context, user *store.User) error {
    query := `
        INSERT INTO users (id, email, name, picture, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())`

    _, err := r.db.Exec(ctx, query,
        user.ID,
        user.Email,
        user.Name,    // *string -- pgx handles nil → NULL automatically
        user.Picture, // *string
    )
    if err != nil {
        return fmt.Errorf("create user: %w", err)
    }

    return nil
}
```

**Key points:**
- `$1, $2, $3, $4` -- PostgreSQL uses numbered placeholders (not `?` like MySQL).
- `r.db.Exec` -- Use `Exec` for INSERT/UPDATE/DELETE that do not return rows.
- `NOW()` -- PostgreSQL function for current timestamp with timezone.
- pgx automatically maps Go `nil` pointers to SQL `NULL`.

### READ (Select Single Row)

```go
// GetUser retrieves a user by their ID.
func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    query := `
        SELECT id, email, name, picture, created_at, updated_at
        FROM users
        WHERE id = $1`

    var user store.User
    err := r.db.QueryRow(ctx, query, userID).Scan(
        &user.ID,
        &user.Email,
        &user.Name,      // *string -- pgx scans NULL into nil
        &user.Picture,   // *string
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    if err != nil {
        if err == pgx.ErrNoRows {
            return nil, fmt.Errorf("user %s not found: %w", userID, err)
        }
        return nil, fmt.Errorf("get user %s: %w", userID, err)
    }

    return &user, nil
}
```

**Key points:**
- `QueryRow` -- Use for queries that return at most one row.
- `pgx.ErrNoRows` -- Returned when no row matches. This is the most common error to check.
- `Scan` order must match `SELECT` column order exactly.
- Always wrap errors with `%w` for error chain support (`errors.Is`, `errors.As`).

### READ (Select Multiple Rows)

```go
// ListProjectsByUser retrieves all projects for a user.
func (r *ProjectRepository) ListProjectsByUser(
    ctx context.Context,
    userID string,
    includeArchived bool,
) ([]*store.Project, error) {
    query := `
        SELECT id, user_id, name, description, settings, is_archived, created_at, updated_at
        FROM projects
        WHERE user_id = $1`

    args := []any{userID}

    if !includeArchived {
        query += " AND is_archived = $2"
        args = append(args, false)
    }

    query += " ORDER BY created_at DESC"

    rows, err := r.db.Query(ctx, query, args...)
    if err != nil {
        return nil, fmt.Errorf("list projects for user %s: %w", userID, err)
    }
    defer rows.Close()

    var projects []*store.Project
    for rows.Next() {
        var p store.Project
        err := rows.Scan(
            &p.ID,
            &p.UserID,
            &p.Name,
            &p.Description,
            &p.Settings,    // JSONB -- pgx scans into struct automatically
            &p.IsArchived,
            &p.CreatedAt,
            &p.UpdatedAt,
        )
        if err != nil {
            return nil, fmt.Errorf("scan project: %w", err)
        }
        projects = append(projects, &p)
    }

    // Check for errors that occurred during iteration
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterate projects: %w", err)
    }

    return projects, nil
}
```

**Key points:**
- `Query` -- Use for queries that return multiple rows.
- `defer rows.Close()` -- Always close rows to return the connection to the pool.
- `rows.Next()` -- Iterates through results. Returns false when done or on error.
- `rows.Err()` -- Check for errors after the loop. This catches errors that occurred during iteration (network issues, etc.).
- Dynamic query building with `args` -- Build the query conditionally while using parameterized values.

### UPDATE

```go
// UpdateProject updates a project's mutable fields.
func (r *ProjectRepository) UpdateProject(ctx context.Context, project *store.Project) error {
    query := `
        UPDATE projects
        SET name = $1,
            description = $2,
            settings = $3,
            is_archived = $4,
            updated_at = NOW()
        WHERE id = $5`

    result, err := r.db.Exec(ctx, query,
        project.Name,
        project.Description,
        project.Settings,   // JSONB -- pgx marshals struct to JSON automatically
        project.IsArchived,
        project.ID,
    )
    if err != nil {
        return fmt.Errorf("update project %s: %w", project.ID, err)
    }

    // Check that a row was actually updated
    if result.RowsAffected() == 0 {
        return fmt.Errorf("project %s not found", project.ID)
    }

    return nil
}
```

**Key points:**
- `Exec` returns a `pgconn.CommandTag` with `RowsAffected()`.
- Always check `RowsAffected()` for UPDATE/DELETE to detect "not found" cases.
- `NOW()` updates the timestamp on every modification.

### DELETE

```go
// DeleteProject removes a project by ID.
func (r *ProjectRepository) DeleteProject(ctx context.Context, projectID string) error {
    query := `DELETE FROM projects WHERE id = $1`

    result, err := r.db.Exec(ctx, query, projectID)
    if err != nil {
        return fmt.Errorf("delete project %s: %w", projectID, err)
    }

    if result.RowsAffected() == 0 {
        return fmt.Errorf("project %s not found", projectID)
    }

    return nil
}
```

### DELETE with Count Return

```go
// DeleteExpiredSessions removes all expired sessions and returns how many were deleted.
func (r *SessionRepository) DeleteExpiredSessions(ctx context.Context) (int64, error) {
    query := `DELETE FROM sessions WHERE expires_at < NOW()`

    result, err := r.db.Exec(ctx, query)
    if err != nil {
        return 0, fmt.Errorf("delete expired sessions: %w", err)
    }

    return result.RowsAffected(), nil
}
```

### INSERT with RETURNING

PostgreSQL's `RETURNING` clause lets you get back the inserted or updated row without a separate SELECT:

```go
// CreateProject inserts a new project and populates its server-generated fields.
func (r *ProjectRepository) CreateProject(ctx context.Context, project *store.Project) error {
    query := `
        INSERT INTO projects (id, user_id, name, description, settings, is_archived)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING created_at, updated_at`

    err := r.db.QueryRow(ctx, query,
        project.ID,
        project.UserID,
        project.Name,
        project.Description,
        project.Settings,
        project.IsArchived,
    ).Scan(&project.CreatedAt, &project.UpdatedAt)

    if err != nil {
        return fmt.Errorf("create project: %w", err)
    }

    // project.CreatedAt and project.UpdatedAt are now populated
    // with the values generated by the database (DEFAULT NOW())
    return nil
}
```

### Node.js Comparison

```javascript
// Node.js with raw pg
const { rows } = await pool.query(
    'SELECT id, email, name FROM users WHERE id = $1',
    [userId]
);
const user = rows[0]; // undefined if not found

// Node.js with Prisma
const user = await prisma.user.findUnique({
    where: { id: userId },
});
// Returns null if not found

// Node.js with Knex
const user = await knex('users')
    .where({ id: userId })
    .first();
// Returns undefined if not found
```

In Go, you handle the "not found" case explicitly with `pgx.ErrNoRows`. In Node.js, you check for `null`/`undefined`. Go's approach is more explicit -- you cannot accidentally use a nil user without the compiler warning you.

---

## 8. Upsert Pattern (ON CONFLICT)

### What Is an Upsert?

An upsert is an "INSERT or UPDATE" -- if the row exists, update it; if it does not, insert it. This is extremely common in production APIs:

- **OAuth login** -- User signs in with Google. If the account exists, update their name/picture. If not, create the account.
- **Webhook processing** -- An external service sends events. If you have already processed this event, update it. Otherwise, insert it.
- **Configuration syncing** -- Push settings. If they exist, overwrite. Otherwise, create.

### PostgreSQL ON CONFLICT Syntax

```sql
INSERT INTO table (columns...)
VALUES (values...)
ON CONFLICT (conflict_columns) DO UPDATE SET
    column1 = excluded.column1,
    column2 = excluded.column2;
```

`excluded` is a special PostgreSQL keyword that refers to the row that was proposed for insertion but conflicted. It gives you access to the values you tried to insert.

### Implementing Upsert in Go

```go
// UpsertUser creates a new user or updates an existing one.
// This is typically used during OAuth authentication:
// - First login: creates the user
// - Subsequent logins: updates name, picture, email from the OAuth provider
func (r *UserRepository) UpsertUser(ctx context.Context, user *store.User) error {
    query := `
        INSERT INTO users (id, email, name, picture, updated_at)
        VALUES ($1, $2, $3, $4, NOW())
        ON CONFLICT(id) DO UPDATE SET
            email = excluded.email,
            name = COALESCE(excluded.name, users.name),
            picture = COALESCE(excluded.picture, users.picture),
            updated_at = NOW()
        RETURNING created_at, updated_at`

    err := r.db.QueryRow(ctx, query,
        user.ID,
        user.Email,
        user.Name,
        user.Picture,
    ).Scan(&user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        return fmt.Errorf("upsert user %s: %w", user.ID, err)
    }

    return nil
}
```

### Understanding COALESCE

`COALESCE(a, b)` returns the first non-NULL argument. In the upsert above:

```sql
name = COALESCE(excluded.name, users.name)
```

This means:
- If the new name is not NULL, use the new name.
- If the new name IS NULL, keep the existing name.

This prevents an OAuth login with no name field from wiping out a previously set name.

```
┌──────────────────────────────────────────────────────────────┐
│                    COALESCE Behavior                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Existing name: "Alice"   New name: "Alice Smith"            │
│  → Result: "Alice Smith"  (new value wins)                   │
│                                                               │
│  Existing name: "Alice"   New name: NULL                     │
│  → Result: "Alice"        (keep existing)                    │
│                                                               │
│  Existing name: NULL      New name: "Alice"                  │
│  → Result: "Alice"        (new value fills in)               │
│                                                               │
│  Existing name: NULL      New name: NULL                     │
│  → Result: NULL           (both null, stays null)            │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Upsert with Multiple Conflict Columns

Sometimes the conflict target is a composite key:

```go
// UpsertProjectMember adds a user to a project or updates their role.
func (r *ProjectRepository) UpsertProjectMember(
    ctx context.Context,
    projectID, userID, role string,
) error {
    query := `
        INSERT INTO project_members (project_id, user_id, role, created_at, updated_at)
        VALUES ($1, $2, $3, NOW(), NOW())
        ON CONFLICT (project_id, user_id) DO UPDATE SET
            role = excluded.role,
            updated_at = NOW()`

    _, err := r.db.Exec(ctx, query, projectID, userID, role)
    if err != nil {
        return fmt.Errorf("upsert project member: %w", err)
    }

    return nil
}
```

### Upsert with ON CONFLICT DO NOTHING

Sometimes you want to insert only if the row does not exist, without updating anything:

```go
// EnsureUser creates a user only if they do not already exist.
// Does not update any fields if the user exists.
func (r *UserRepository) EnsureUser(ctx context.Context, user *store.User) error {
    query := `
        INSERT INTO users (id, email, name, picture)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (id) DO NOTHING`

    _, err := r.db.Exec(ctx, query,
        user.ID, user.Email, user.Name, user.Picture,
    )
    if err != nil {
        return fmt.Errorf("ensure user %s: %w", user.ID, err)
    }

    return nil
}
```

### Node.js Comparison

```javascript
// Prisma upsert
const user = await prisma.user.upsert({
    where: { id: userId },
    create: { id: userId, email, name, picture },
    update: { email, name, picture },
});

// Knex -- raw SQL (no built-in upsert)
await knex.raw(`
    INSERT INTO users (id, email, name, picture)
    VALUES (?, ?, ?, ?)
    ON CONFLICT(id) DO UPDATE SET
        email = EXCLUDED.email,
        name = COALESCE(EXCLUDED.name, users.name)
`, [id, email, name, picture]);

// TypeORM -- uses save() which does SELECT then INSERT or UPDATE (2 queries!)
await userRepository.save({ id, email, name, picture });
```

Prisma's `upsert` is clean but limited -- it does not support `COALESCE` logic. Knex requires raw SQL (same as Go). TypeORM's `save()` makes two queries, which is slower and has race conditions. Go's approach is a single atomic SQL statement.

---

## 9. Handling PostgreSQL Errors

### The pgconn.PgError Type

When PostgreSQL returns an error, pgx wraps it in a `pgconn.PgError` struct. This struct contains rich information about what went wrong, including the PostgreSQL error code.

```go
package pgconn

type PgError struct {
    Severity         string // ERROR, FATAL, PANIC, WARNING, NOTICE, DEBUG, INFO, LOG
    Code             string // PostgreSQL error code (e.g., "23505")
    Message          string // Human-readable message
    Detail           string // Additional detail
    Hint             string // Suggestion for fixing the issue
    Position         int32  // Error position in query string
    InternalPosition int32
    InternalQuery    string
    Where            string
    SchemaName       string
    TableName        string
    ColumnName       string
    DataTypeName     string
    ConstraintName   string // Name of the violated constraint
    File             string // PostgreSQL source file
    Line             int32  // PostgreSQL source line
    Routine          string // PostgreSQL source routine
}
```

### Extracting PostgreSQL Errors

```go
package repository

import (
    "errors"
    "fmt"

    "github.com/jackc/pgx/v5/pgconn"
)

// PostgreSQL error codes
// Full list: https://www.postgresql.org/docs/current/errcodes-appendix.html
const (
    PgUniqueViolation     = "23505" // Unique constraint violated
    PgForeignKeyViolation = "23503" // Foreign key constraint violated
    PgCheckViolation      = "23514" // Check constraint violated
    PgNotNullViolation    = "23502" // NOT NULL constraint violated
)

// isPgError checks if an error is a PostgreSQL error with the given code.
func isPgError(err error, code string) bool {
    var pgErr *pgconn.PgError
    return errors.As(err, &pgErr) && pgErr.Code == code
}

// getPgError extracts the PgError from an error, if present.
func getPgError(err error) *pgconn.PgError {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr
    }
    return nil
}
```

### Handling Unique Constraint Violations (23505)

This is the most common error to handle. It occurs when you try to insert a row that violates a UNIQUE constraint.

```go
// CreateUser inserts a new user, returning a friendly error if the email already exists.
func (r *UserRepository) CreateUser(ctx context.Context, user *store.User) error {
    query := `
        INSERT INTO users (id, email, name, picture, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())`

    _, err := r.db.Exec(ctx, query,
        user.ID, user.Email, user.Name, user.Picture,
    )
    if err != nil {
        if isPgError(err, PgUniqueViolation) {
            pgErr := getPgError(err)
            switch pgErr.ConstraintName {
            case "users_email_key":
                return fmt.Errorf("email %s is already registered", user.Email)
            case "users_pkey":
                return fmt.Errorf("user ID %s already exists", user.ID)
            default:
                return fmt.Errorf("duplicate value: %s", pgErr.ConstraintName)
            }
        }
        return fmt.Errorf("create user: %w", err)
    }

    return nil
}
```

### Handling Foreign Key Violations (23503)

Foreign key violations occur when you reference a row that does not exist, or try to delete a row that is referenced by other rows.

```go
// CreateSession inserts a new session, handling the case where the user does not exist.
func (r *SessionRepository) CreateSession(ctx context.Context, session *store.Session) error {
    query := `
        INSERT INTO sessions (id, user_id, refresh_token, user_agent, ip_address, expires_at)
        VALUES ($1, $2, $3, $4, $5, $6)`

    _, err := r.db.Exec(ctx, query,
        session.ID,
        session.UserID,
        session.RefreshToken,
        session.UserAgent,
        session.IPAddress,
        session.ExpiresAt,
    )
    if err != nil {
        if isPgError(err, PgForeignKeyViolation) {
            return fmt.Errorf("user %s does not exist", session.UserID)
        }
        return fmt.Errorf("create session: %w", err)
    }

    return nil
}
```

### Creating Custom Error Types

For production APIs, wrap database errors into domain-specific error types that your HTTP handlers can translate to appropriate HTTP status codes:

```go
package store

import "fmt"

// ErrNotFound indicates that the requested resource does not exist.
type ErrNotFound struct {
    Resource string
    ID       string
}

func (e *ErrNotFound) Error() string {
    return fmt.Sprintf("%s %s not found", e.Resource, e.ID)
}

// ErrDuplicate indicates a unique constraint violation.
type ErrDuplicate struct {
    Resource string
    Field    string
    Value    string
}

func (e *ErrDuplicate) Error() string {
    return fmt.Sprintf("%s with %s '%s' already exists", e.Resource, e.Field, e.Value)
}

// ErrForeignKey indicates a foreign key constraint violation.
type ErrForeignKey struct {
    Resource   string
    Reference  string
    ReferenceID string
}

func (e *ErrForeignKey) Error() string {
    return fmt.Sprintf("%s references non-existent %s %s", e.Resource, e.Reference, e.ReferenceID)
}
```

Using these in the repository:

```go
func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    var user store.User
    err := r.db.QueryRow(ctx, `
        SELECT id, email, name, picture, created_at, updated_at
        FROM users WHERE id = $1`, userID,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.Picture,
        &user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "user", ID: userID}
        }
        return nil, fmt.Errorf("get user: %w", err)
    }

    return &user, nil
}
```

Using these in HTTP handlers:

```go
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    userID := r.PathValue("id")

    user, err := h.store.GetUser(r.Context(), userID)
    if err != nil {
        var notFound *store.ErrNotFound
        var duplicate *store.ErrDuplicate

        switch {
        case errors.As(err, &notFound):
            writeJSON(w, http.StatusNotFound, map[string]string{
                "error": notFound.Error(),
            })
        case errors.As(err, &duplicate):
            writeJSON(w, http.StatusConflict, map[string]string{
                "error": duplicate.Error(),
            })
        default:
            writeJSON(w, http.StatusInternalServerError, map[string]string{
                "error": "internal server error",
            })
        }
        return
    }

    writeJSON(w, http.StatusOK, user)
}
```

### Node.js Comparison

```javascript
// Node.js with pg -- catching unique violations
try {
    await pool.query('INSERT INTO users (id, email) VALUES ($1, $2)', [id, email]);
} catch (err) {
    if (err.code === '23505') {
        // Unique violation
        throw new ConflictError(`Email ${email} already exists`);
    }
    throw err;
}

// Prisma -- catches and wraps errors
try {
    await prisma.user.create({ data: { id, email } });
} catch (err) {
    if (err instanceof Prisma.PrismaClientKnownRequestError) {
        if (err.code === 'P2002') {
            // Unique constraint violation
            throw new ConflictError(`Duplicate value on ${err.meta.target}`);
        }
    }
    throw err;
}
```

Note that Prisma uses its own error codes (`P2002`) instead of PostgreSQL's native codes (`23505`). This abstraction is convenient but means you need to learn Prisma-specific error codes. Go gives you direct access to PostgreSQL error codes -- the same codes documented in the PostgreSQL manual.

---

## 10. Working with JSONB Columns

### Why JSONB?

JSONB (Binary JSON) is one of PostgreSQL's most powerful features. It lets you store semi-structured data in a column while still being able to query, index, and manipulate it with SQL.

Use JSONB when:
- The data shape varies between rows (user preferences, feature flags)
- You need flexible schema within a fixed table structure
- You want to avoid creating many small related tables
- You need to store API responses, webhook payloads, or configuration

Do NOT use JSONB when:
- You need to join on values within the JSON (use a regular column instead)
- The data is always the same shape (use regular columns)
- You need strict type enforcement (regular columns with constraints)

### Defining JSONB Structs

```go
package store

// ProjectSettings is stored as a JSONB column in the projects table.
type ProjectSettings struct {
    Theme       string   `json:"theme,omitempty"`
    Language    string   `json:"language,omitempty"`
    IsPublic    bool     `json:"isPublic"`
    MaxFileSize int      `json:"maxFileSize,omitempty"`
    Tags        []string `json:"tags,omitempty"`
}

// UserPreferences is stored as a JSONB column in the users table.
type UserPreferences struct {
    Theme           string            `json:"theme"`
    Language        string            `json:"language"`
    Timezone        string            `json:"timezone"`
    Notifications   NotificationPrefs `json:"notifications"`
    CustomSettings  map[string]any    `json:"customSettings,omitempty"`
}

type NotificationPrefs struct {
    Email bool `json:"email"`
    Push  bool `json:"push"`
    SMS   bool `json:"sms"`
}
```

### Inserting JSONB Data

pgx automatically marshals Go structs to JSON when inserting into JSONB columns:

```go
func (r *ProjectRepository) CreateProject(ctx context.Context, project *store.Project) error {
    query := `
        INSERT INTO projects (id, user_id, name, description, settings)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING created_at, updated_at`

    // pgx sees that the 'settings' column is JSONB and automatically
    // marshals project.Settings (a Go struct) into JSON bytes
    err := r.db.QueryRow(ctx, query,
        project.ID,
        project.UserID,
        project.Name,
        project.Description,
        project.Settings, // Go struct → JSON → JSONB
    ).Scan(&project.CreatedAt, &project.UpdatedAt)

    if err != nil {
        return fmt.Errorf("create project: %w", err)
    }

    return nil
}
```

### Reading JSONB Data

pgx automatically unmarshals JSONB columns into Go structs:

```go
func (r *ProjectRepository) GetProject(ctx context.Context, projectID string) (*store.Project, error) {
    query := `
        SELECT id, user_id, name, description, settings, is_archived, created_at, updated_at
        FROM projects
        WHERE id = $1`

    var p store.Project
    err := r.db.QueryRow(ctx, query, projectID).Scan(
        &p.ID,
        &p.UserID,
        &p.Name,
        &p.Description,
        &p.Settings,    // JSONB → Go struct automatically
        &p.IsArchived,
        &p.CreatedAt,
        &p.UpdatedAt,
    )
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "project", ID: projectID}
        }
        return nil, fmt.Errorf("get project: %w", err)
    }

    return &p, nil
}
```

### Querying Inside JSONB

PostgreSQL provides operators for querying inside JSONB columns:

```go
// ListPublicProjects retrieves all projects where settings.isPublic is true.
func (r *ProjectRepository) ListPublicProjects(ctx context.Context) ([]*store.Project, error) {
    // The ->> operator extracts a JSON field as text
    // The -> operator extracts a JSON field as JSON
    // The @> operator checks if the JSONB contains the given JSON
    query := `
        SELECT id, user_id, name, description, settings, is_archived, created_at, updated_at
        FROM projects
        WHERE settings @> '{"isPublic": true}'
        AND is_archived = false
        ORDER BY created_at DESC`

    rows, err := r.db.Query(ctx, query)
    if err != nil {
        return nil, fmt.Errorf("list public projects: %w", err)
    }
    defer rows.Close()

    var projects []*store.Project
    for rows.Next() {
        var p store.Project
        if err := rows.Scan(
            &p.ID, &p.UserID, &p.Name, &p.Description,
            &p.Settings, &p.IsArchived, &p.CreatedAt, &p.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan project: %w", err)
        }
        projects = append(projects, &p)
    }

    return projects, rows.Err()
}
```

### Updating Specific JSONB Fields

You can update individual fields within a JSONB column without replacing the entire value:

```go
// UpdateProjectTheme updates only the theme field within the JSONB settings.
func (r *ProjectRepository) UpdateProjectTheme(ctx context.Context, projectID, theme string) error {
    // jsonb_set updates a specific path within the JSONB
    query := `
        UPDATE projects
        SET settings = jsonb_set(settings, '{theme}', to_jsonb($1::text)),
            updated_at = NOW()
        WHERE id = $2`

    result, err := r.db.Exec(ctx, query, theme, projectID)
    if err != nil {
        return fmt.Errorf("update theme: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "project", ID: projectID}
    }

    return nil
}
```

### Merging JSONB Data

```go
// MergeProjectSettings merges new settings into existing settings.
// Existing keys not in the update are preserved.
func (r *ProjectRepository) MergeProjectSettings(
    ctx context.Context,
    projectID string,
    updates store.ProjectSettings,
) error {
    // The || operator merges two JSONB values. Keys in the right operand override the left.
    query := `
        UPDATE projects
        SET settings = settings || $1::jsonb,
            updated_at = NOW()
        WHERE id = $2`

    result, err := r.db.Exec(ctx, query, updates, projectID)
    if err != nil {
        return fmt.Errorf("merge settings: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "project", ID: projectID}
    }

    return nil
}
```

### JSONB PostgreSQL Operators Reference

```
┌────────────────────────────────────────────────────────────┐
│              PostgreSQL JSONB Operators                      │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ->    Extract JSON element (returns jsonb)                 │
│        settings -> 'theme'  →  "dark"  (as jsonb)          │
│                                                             │
│  ->>   Extract JSON element (returns text)                  │
│        settings ->> 'theme'  →  dark  (as text)            │
│                                                             │
│  @>    Contains (left contains right)                       │
│        settings @> '{"isPublic": true}'                    │
│                                                             │
│  <@    Contained by (left is contained by right)            │
│        '{"theme": "dark"}' <@ settings                     │
│                                                             │
│  ?     Key exists                                           │
│        settings ? 'theme'                                  │
│                                                             │
│  ||    Concatenate/merge two jsonb values                   │
│        settings || '{"newKey": "value"}'                   │
│                                                             │
│  #-    Delete key at path                                   │
│        settings #- '{tags,0}'  (delete first tag)          │
│                                                             │
│  jsonb_set(target, path, new_value)                         │
│        Update a specific path                               │
│                                                             │
│  jsonb_array_length(jsonb)                                  │
│        Length of a JSON array                               │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Indexing JSONB Columns

For queries that filter on JSONB fields, add a GIN index:

```sql
-- Index the entire JSONB column (supports @>, ?, ?|, ?& operators)
CREATE INDEX idx_projects_settings ON projects USING GIN (settings);

-- Index a specific key for equality/range queries
CREATE INDEX idx_projects_settings_theme ON projects ((settings ->> 'theme'));

-- Index for checking if a JSONB value contains specific elements
CREATE INDEX idx_projects_settings_contains ON projects USING GIN (settings jsonb_path_ops);
```

---

## 11. Context-Aware Queries

### Why Context in Database Queries?

Every database query should accept a `context.Context` as its first parameter. This enables:

1. **Request cancellation** -- If the HTTP client disconnects, the database query is cancelled.
2. **Timeouts** -- Prevent slow queries from holding connections forever.
3. **Tracing** -- Distributed tracing systems propagate trace IDs through context.

```
Client disconnects
       │
       ▼
HTTP Handler (context cancelled)
       │
       ▼
Service Layer (context cancelled)
       │
       ▼
Repository (context cancelled)
       │
       ▼
pgx query (detects cancelled context, sends cancel to PostgreSQL)
       │
       ▼
PostgreSQL (stops executing the query, frees resources)
```

Without context, the PostgreSQL query keeps running even after the client is gone. With context, the entire chain is cancelled, freeing database connections and CPU.

### pgx and Context

pgx natively supports context in all operations. When a context is cancelled:

1. pgx sends a cancellation signal to PostgreSQL.
2. PostgreSQL stops the running query.
3. pgx returns `context.Canceled` or `context.DeadlineExceeded` error.
4. The connection is returned to the pool.

```go
// Every pgxpool method takes context as the first argument:
pool.Query(ctx, sql, args...)
pool.QueryRow(ctx, sql, args...)
pool.Exec(ctx, sql, args...)
pool.Begin(ctx)
pool.Ping(ctx)
```

### Adding Query Timeouts

For queries that might be slow, add a timeout to the context:

```go
func (r *UserRepository) SearchUsers(ctx context.Context, query string) ([]*store.User, error) {
    // Create a context with a 5-second timeout for this specific query
    queryCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    sqlQuery := `
        SELECT id, email, name, picture, created_at, updated_at
        FROM users
        WHERE email ILIKE $1 OR name ILIKE $1
        ORDER BY created_at DESC
        LIMIT 50`

    rows, err := r.db.Query(queryCtx, sqlQuery, "%"+query+"%")
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("search query timed out after 5s")
        }
        return nil, fmt.Errorf("search users: %w", err)
    }
    defer rows.Close()

    var users []*store.User
    for rows.Next() {
        var u store.User
        if err := rows.Scan(
            &u.ID, &u.Email, &u.Name, &u.Picture, &u.CreatedAt, &u.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan user: %w", err)
        }
        users = append(users, &u)
    }

    return users, rows.Err()
}
```

### Context from HTTP Requests

In HTTP handlers, the request context is automatically cancelled when the client disconnects:

```go
func (h *Handler) ListProjects(w http.ResponseWriter, r *http.Request) {
    // r.Context() is cancelled if:
    // 1. The client disconnects
    // 2. The server's WriteTimeout is exceeded
    // 3. A parent context is cancelled

    userID := r.Context().Value("userID").(string)

    projects, err := h.store.ListProjectsByUser(r.Context(), userID, false)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected -- no point writing a response
            return
        }
        writeJSON(w, http.StatusInternalServerError, map[string]string{
            "error": "failed to list projects",
        })
        return
    }

    writeJSON(w, http.StatusOK, projects)
}
```

### Context Propagation Best Practice

Always pass context down through the entire call chain. Never create a new `context.Background()` in a repository method -- that would break cancellation.

```go
// WRONG: Creates a new context, breaking cancellation propagation
func (r *UserRepository) GetUser(userID string) (*store.User, error) {
    ctx := context.Background() // BAD: if the HTTP request is cancelled, this query keeps running
    return r.db.QueryRow(ctx, "SELECT ...", userID).Scan(...)
}

// CORRECT: Accepts context from the caller
func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    return r.db.QueryRow(ctx, "SELECT ...", userID).Scan(...)
}
```

---

## 12. Row Scanning Patterns

### Single Row Scanning

The simplest pattern -- fetch one row and scan into variables:

```go
func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    var user store.User
    err := r.db.QueryRow(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users WHERE id = $1`, userID,
    ).Scan(
        &user.ID,
        &user.Email,
        &user.Name,      // *string -- nil if NULL
        &user.Picture,   // *string -- nil if NULL
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "user", ID: userID}
        }
        return nil, fmt.Errorf("get user: %w", err)
    }
    return &user, nil
}
```

**Critical rule:** The order and number of `Scan` arguments must exactly match the `SELECT` column list. If you add a column to the SELECT but forget to add it to Scan, you get a runtime error.

### Multiple Row Scanning

```go
func (r *SessionRepository) GetSessionsByUser(
    ctx context.Context,
    userID string,
) ([]*store.Session, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, user_id, refresh_token, user_agent, ip_address,
                expires_at, created_at, updated_at
         FROM sessions
         WHERE user_id = $1
         ORDER BY created_at DESC`, userID,
    )
    if err != nil {
        return nil, fmt.Errorf("query sessions: %w", err)
    }
    defer rows.Close() // ALWAYS close rows

    var sessions []*store.Session
    for rows.Next() {
        var s store.Session
        if err := rows.Scan(
            &s.ID, &s.UserID, &s.RefreshToken, &s.UserAgent,
            &s.IPAddress, &s.ExpiresAt, &s.CreatedAt, &s.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan session: %w", err)
        }
        sessions = append(sessions, &s)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterate sessions: %w", err)
    }

    return sessions, nil
}
```

**Common mistakes to avoid:**
1. Forgetting `defer rows.Close()` -- This leaks database connections. The pool runs out and your app hangs.
2. Forgetting `rows.Err()` -- Errors during iteration (network drops, etc.) are not returned by `rows.Next()`. You must check `rows.Err()` after the loop.
3. Scanning into the same struct variable without re-declaring -- `var s store.Session` must be inside the loop to get a fresh struct each iteration when storing pointers.

### Using pgx.CollectRows (pgx v5 Helper)

pgx v5 provides a helper that simplifies multiple row scanning:

```go
import "github.com/jackc/pgx/v5"

func (r *UserRepository) ListUsers(ctx context.Context) ([]*store.User, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users ORDER BY created_at DESC`)
    if err != nil {
        return nil, fmt.Errorf("query users: %w", err)
    }

    users, err := pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[store.User])
    if err != nil {
        return nil, fmt.Errorf("collect users: %w", err)
    }

    return users, nil
}
```

`pgx.CollectRows` handles the `rows.Next()` loop, closing, and error checking for you. `pgx.RowToAddrOfStructByName` maps column names to struct fields by their `db` tag or by field name (case-insensitive). For this to work, add `db` tags:

```go
type User struct {
    ID        string    `json:"id"        db:"id"`
    Email     string    `json:"email"     db:"email"`
    Name      *string   `json:"name"      db:"name"`
    Picture   *string   `json:"picture"   db:"picture"`
    CreatedAt time.Time `json:"createdAt" db:"created_at"`
    UpdatedAt time.Time `json:"updatedAt" db:"updated_at"`
}
```

### Scanning Aggregate Queries

```go
// CountUserSessions returns the number of active sessions for a user.
func (r *SessionRepository) CountUserSessions(ctx context.Context, userID string) (int64, error) {
    var count int64
    err := r.db.QueryRow(ctx,
        `SELECT COUNT(*) FROM sessions WHERE user_id = $1 AND expires_at > NOW()`,
        userID,
    ).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("count sessions: %w", err)
    }
    return count, nil
}
```

### Scanning with EXISTS

```go
// UserExists checks if a user with the given ID exists.
func (r *UserRepository) UserExists(ctx context.Context, userID string) (bool, error) {
    var exists bool
    err := r.db.QueryRow(ctx,
        `SELECT EXISTS(SELECT 1 FROM users WHERE id = $1)`,
        userID,
    ).Scan(&exists)
    if err != nil {
        return false, fmt.Errorf("check user exists: %w", err)
    }
    return exists, nil
}
```

### Scanning Joins

When joining tables, you scan into a custom struct that represents the combined result:

```go
// ProjectWithOwner represents a project joined with its owner's information.
type ProjectWithOwner struct {
    Project    store.Project
    OwnerEmail string
    OwnerName  *string
}

func (r *ProjectRepository) GetProjectWithOwner(
    ctx context.Context,
    projectID string,
) (*ProjectWithOwner, error) {
    query := `
        SELECT p.id, p.user_id, p.name, p.description, p.settings,
               p.is_archived, p.created_at, p.updated_at,
               u.email, u.name
        FROM projects p
        JOIN users u ON u.id = p.user_id
        WHERE p.id = $1`

    var result ProjectWithOwner
    err := r.db.QueryRow(ctx, query, projectID).Scan(
        &result.Project.ID,
        &result.Project.UserID,
        &result.Project.Name,
        &result.Project.Description,
        &result.Project.Settings,
        &result.Project.IsArchived,
        &result.Project.CreatedAt,
        &result.Project.UpdatedAt,
        &result.OwnerEmail,
        &result.OwnerName,
    )
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "project", ID: projectID}
        }
        return nil, fmt.Errorf("get project with owner: %w", err)
    }

    return &result, nil
}
```

---

## 13. SQL Triggers for Auto-Updating Timestamps

### The Problem

Every time you UPDATE a row, you need to set `updated_at = NOW()`. If you forget it in even one UPDATE query, the timestamp becomes stale and your data is inconsistent.

### The Solution: PostgreSQL Triggers

A trigger automatically fires before or after a database operation. We can create a trigger that sets `updated_at` to `NOW()` on every UPDATE, regardless of which column is modified.

### Migration for the Trigger Function

```sql
-- migrations/004_create_updated_at_trigger.sql

-- +goose Up

-- Create a reusable function that sets updated_at to NOW()
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply the trigger to each table that has an updated_at column
CREATE TRIGGER set_updated_at_users
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER set_updated_at_sessions
    BEFORE UPDATE ON sessions
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER set_updated_at_projects
    BEFORE UPDATE ON projects
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- +goose Down
DROP TRIGGER IF EXISTS set_updated_at_projects ON projects;
DROP TRIGGER IF EXISTS set_updated_at_sessions ON sessions;
DROP TRIGGER IF EXISTS set_updated_at_users ON users;
DROP FUNCTION IF EXISTS update_updated_at_column;
```

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                  Trigger Execution Flow                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Your Go code executes:                               │
│     UPDATE users SET name = 'Alice' WHERE id = 'abc'     │
│                                                          │
│  2. PostgreSQL intercepts the UPDATE (BEFORE trigger)    │
│                                                          │
│  3. Trigger function runs:                               │
│     NEW.updated_at = NOW()                               │
│                                                          │
│  4. PostgreSQL executes the UPDATE with the modified     │
│     NEW row (name='Alice', updated_at=now())             │
│                                                          │
│  Result: updated_at is ALWAYS current, even if your      │
│  Go code does not explicitly set it.                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Simplified Repository Code with Triggers

With triggers in place, your UPDATE queries become simpler:

```go
// WITHOUT trigger -- you must remember to set updated_at every time
func (r *UserRepository) UpdateName(ctx context.Context, userID string, name string) error {
    _, err := r.db.Exec(ctx,
        `UPDATE users SET name = $1, updated_at = NOW() WHERE id = $2`,
        name, userID,
    )
    return err
}

// WITH trigger -- updated_at is handled automatically
func (r *UserRepository) UpdateName(ctx context.Context, userID string, name string) error {
    _, err := r.db.Exec(ctx,
        `UPDATE users SET name = $1 WHERE id = $2`,
        name, userID,
    )
    return err
}
```

Both produce the same result, but the trigger version cannot be forgotten. Even if another developer writes a new UPDATE query months later, the trigger ensures `updated_at` is always correct.

### Node.js Comparison

In Prisma, `@updatedAt` does this automatically:

```prisma
model User {
  id        String   @id
  name      String?
  updatedAt DateTime @updatedAt  // Prisma auto-updates this
}
```

In Knex, you would use the same PostgreSQL trigger approach. The trigger is database-level, so it works regardless of which programming language or ORM is talking to the database.

---

## 14. Health Check Queries

### Why Health Checks?

In production, you need to know if your database connection is alive. Kubernetes, load balancers, and monitoring systems use health check endpoints to determine if your service is ready to handle requests.

```
┌──────────────────────────────────────────────────┐
│            Health Check Architecture              │
├──────────────────────────────────────────────────┤
│                                                   │
│  Kubernetes → GET /healthz → Is the process alive? │
│               (liveness probe)                    │
│                                                   │
│  Kubernetes → GET /readyz → Can it handle traffic? │
│               (readiness probe)                   │
│               Checks: database, cache, etc.       │
│                                                   │
│  Load Balancer → GET /health → Should I route     │
│                  traffic here?                    │
│                                                   │
└──────────────────────────────────────────────────┘
```

### Basic Database Health Check

```go
package handler

import (
    "context"
    "encoding/json"
    "net/http"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

type HealthHandler struct {
    db *pgxpool.Pool
}

func NewHealthHandler(db *pgxpool.Pool) *HealthHandler {
    return &HealthHandler{db: db}
}

// Liveness -- is the process alive?
// Keep this simple. If it can respond, the process is alive.
func (h *HealthHandler) Liveness(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{
        "status": "alive",
    })
}

// Readiness -- can the service handle traffic?
// Check all critical dependencies (database, cache, etc.)
func (h *HealthHandler) Readiness(w http.ResponseWriter, r *http.Request) {
    // Use a short timeout -- health checks should be fast
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    // Check database connectivity
    err := h.db.Ping(ctx)

    w.Header().Set("Content-Type", "application/json")

    if err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]any{
            "status": "not ready",
            "checks": map[string]any{
                "database": map[string]any{
                    "status": "down",
                    "error":  err.Error(),
                },
            },
        })
        return
    }

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]any{
        "status": "ready",
        "checks": map[string]any{
            "database": map[string]any{
                "status": "up",
            },
        },
    })
}
```

### Advanced Health Check with Pool Stats

```go
// DetailedHealth returns detailed health information including pool statistics.
// This endpoint should NOT be exposed publicly -- use for internal monitoring only.
func (h *HealthHandler) DetailedHealth(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    dbErr := h.db.Ping(ctx)
    stats := h.db.Stat()

    status := "healthy"
    httpStatus := http.StatusOK

    if dbErr != nil {
        status = "unhealthy"
        httpStatus = http.StatusServiceUnavailable
    }

    // Warn if pool utilization is high
    utilization := float64(stats.AcquiredConns()) / float64(stats.MaxConns())
    if utilization > 0.8 {
        status = "degraded"
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(httpStatus)
    json.NewEncoder(w).Encode(map[string]any{
        "status": status,
        "database": map[string]any{
            "connected":        dbErr == nil,
            "error":            errorString(dbErr),
            "totalConnections":  stats.TotalConns(),
            "idleConnections":   stats.IdleConns(),
            "activeConnections": stats.AcquiredConns(),
            "maxConnections":    stats.MaxConns(),
            "poolUtilization":   utilization,
        },
    })
}

func errorString(err error) string {
    if err == nil {
        return ""
    }
    return err.Error()
}
```

### Registering Health Check Routes

```go
func main() {
    // ... pool setup ...

    healthHandler := handler.NewHealthHandler(pool)

    mux := http.NewServeMux()
    mux.HandleFunc("GET /healthz", healthHandler.Liveness)
    mux.HandleFunc("GET /readyz", healthHandler.Readiness)
    mux.HandleFunc("GET /health/detail", healthHandler.DetailedHealth)

    // ... start server ...
}
```

---

## 15. Transactions

### Why Transactions?

A transaction ensures that a group of database operations either ALL succeed or ALL fail. Without transactions, a failure halfway through could leave your data in an inconsistent state.

**Example:** Transferring a user from one team to another requires:
1. Remove the user from the old team.
2. Add the user to the new team.
3. Update the user's team_id.

If step 2 fails after step 1 succeeds, the user is in neither team. A transaction ensures all three steps succeed or all three are rolled back.

### Basic Transaction Pattern

```go
func (r *UserRepository) TransferTeam(
    ctx context.Context,
    userID, oldTeamID, newTeamID string,
) error {
    // Begin a transaction
    tx, err := r.db.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    // Defer rollback -- if we reach this line, something went wrong.
    // If tx.Commit() was already called, Rollback is a no-op.
    defer tx.Rollback(ctx)

    // Step 1: Remove from old team
    _, err = tx.Exec(ctx,
        `DELETE FROM team_members WHERE user_id = $1 AND team_id = $2`,
        userID, oldTeamID,
    )
    if err != nil {
        return fmt.Errorf("remove from old team: %w", err)
    }

    // Step 2: Add to new team
    _, err = tx.Exec(ctx,
        `INSERT INTO team_members (user_id, team_id, role) VALUES ($1, $2, 'member')`,
        userID, newTeamID,
    )
    if err != nil {
        return fmt.Errorf("add to new team: %w", err)
    }

    // Step 3: Update user's team_id
    _, err = tx.Exec(ctx,
        `UPDATE users SET team_id = $1 WHERE id = $2`,
        newTeamID, userID,
    )
    if err != nil {
        return fmt.Errorf("update user team: %w", err)
    }

    // All steps succeeded -- commit
    if err := tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }

    return nil
}
```

### Transaction Helper Function

To avoid repeating the begin/rollback/commit boilerplate, create a helper:

```go
package database

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

// WithTx executes a function within a database transaction.
// If the function returns an error, the transaction is rolled back.
// If the function returns nil, the transaction is committed.
func WithTx(ctx context.Context, pool *pgxpool.Pool, fn func(tx pgx.Tx) error) error {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback(ctx) // no-op if already committed

    if err := fn(tx); err != nil {
        return err // rollback happens via defer
    }

    if err := tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }

    return nil
}
```

Usage:

```go
func (r *UserRepository) TransferTeam(
    ctx context.Context,
    userID, oldTeamID, newTeamID string,
) error {
    return database.WithTx(ctx, r.db, func(tx pgx.Tx) error {
        _, err := tx.Exec(ctx,
            `DELETE FROM team_members WHERE user_id = $1 AND team_id = $2`,
            userID, oldTeamID,
        )
        if err != nil {
            return fmt.Errorf("remove from old team: %w", err)
        }

        _, err = tx.Exec(ctx,
            `INSERT INTO team_members (user_id, team_id, role) VALUES ($1, $2, 'member')`,
            userID, newTeamID,
        )
        if err != nil {
            return fmt.Errorf("add to new team: %w", err)
        }

        _, err = tx.Exec(ctx,
            `UPDATE users SET team_id = $1 WHERE id = $2`,
            newTeamID, userID,
        )
        if err != nil {
            return fmt.Errorf("update user team: %w", err)
        }

        return nil
    })
}
```

### Transactions with RETURNING

```go
// CreateUserWithSession creates a user and their initial session in a single transaction.
func (r *Repository) CreateUserWithSession(
    ctx context.Context,
    user *store.User,
    session *store.Session,
) error {
    return database.WithTx(ctx, r.db, func(tx pgx.Tx) error {
        // Insert user and get the generated timestamps
        err := tx.QueryRow(ctx,
            `INSERT INTO users (id, email, name, picture)
             VALUES ($1, $2, $3, $4)
             RETURNING created_at, updated_at`,
            user.ID, user.Email, user.Name, user.Picture,
        ).Scan(&user.CreatedAt, &user.UpdatedAt)
        if err != nil {
            return fmt.Errorf("insert user: %w", err)
        }

        // Insert session (references the user we just created)
        err = tx.QueryRow(ctx,
            `INSERT INTO sessions (id, user_id, refresh_token, user_agent, ip_address, expires_at)
             VALUES ($1, $2, $3, $4, $5, $6)
             RETURNING created_at, updated_at`,
            session.ID, session.UserID, session.RefreshToken,
            session.UserAgent, session.IPAddress, session.ExpiresAt,
        ).Scan(&session.CreatedAt, &session.UpdatedAt)
        if err != nil {
            return fmt.Errorf("insert session: %w", err)
        }

        return nil
    })
}
```

### Nested Operations in Transactions

Sometimes you want a repository method that can work both inside and outside a transaction. Use an interface that both `pgxpool.Pool` and `pgx.Tx` implement:

```go
package database

import (
    "context"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
)

// DBTX is an interface that both *pgxpool.Pool and pgx.Tx implement.
// This allows repository methods to work with or without a transaction.
type DBTX interface {
    Exec(ctx context.Context, sql string, arguments ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}
```

Use DBTX in your repositories:

```go
type UserRepository struct {
    db database.DBTX
}

func NewUserRepository(db database.DBTX) *UserRepository {
    return &UserRepository{db: db}
}

// This method works with both a pool and a transaction
func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    var user store.User
    err := r.db.QueryRow(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users WHERE id = $1`, userID,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.Picture,
        &user.CreatedAt, &user.UpdatedAt)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "user", ID: userID}
        }
        return nil, fmt.Errorf("get user: %w", err)
    }
    return &user, nil
}
```

Now you can use the same repository inside a transaction:

```go
func (s *Service) ProcessSomething(ctx context.Context) error {
    return database.WithTx(ctx, s.pool, func(tx pgx.Tx) error {
        // Create a repository that uses the transaction
        userRepo := repository.NewUserRepository(tx)

        user, err := userRepo.GetUser(ctx, "user-123")
        if err != nil {
            return err
        }

        // ... more operations with the same transaction ...
        return nil
    })
}
```

### Node.js Comparison

```javascript
// Node.js with pg
const client = await pool.connect();
try {
    await client.query('BEGIN');
    await client.query('DELETE FROM team_members WHERE user_id = $1', [userId]);
    await client.query('INSERT INTO team_members (user_id, team_id) VALUES ($1, $2)', [userId, newTeamId]);
    await client.query('COMMIT');
} catch (err) {
    await client.query('ROLLBACK');
    throw err;
} finally {
    client.release();
}

// Prisma transactions
await prisma.$transaction([
    prisma.teamMember.delete({ where: { userId_teamId: { userId, teamId: oldTeamId } } }),
    prisma.teamMember.create({ data: { userId, teamId: newTeamId } }),
    prisma.user.update({ where: { id: userId }, data: { teamId: newTeamId } }),
]);

// Or with interactive transactions
await prisma.$transaction(async (tx) => {
    await tx.teamMember.delete({ where: { userId_teamId: { userId, teamId: oldTeamId } } });
    await tx.teamMember.create({ data: { userId, teamId: newTeamId } });
    await tx.user.update({ where: { id: userId }, data: { teamId: newTeamId } });
});
```

Go's `defer tx.Rollback(ctx)` pattern is cleaner than Node.js's try/catch/finally. The rollback is automatically called on any error path, and is a no-op after a successful commit. In Node.js, forgetting to call `client.release()` in the finally block leads to connection leaks.

---

## 16. Mock Repositories for Testing

### Why Mock?

Unit tests should be fast, deterministic, and independent. If your handler tests require a running PostgreSQL database, they are slow, flaky (network issues), and hard to set up in CI.

Mock repositories implement the same interface as your real repositories but return canned data. Your handler does not know (or care) that it is talking to a mock.

### Strategy 1: Manual Mocks with Function Fields

This is the simplest approach and does not require any third-party libraries:

```go
package mock

import (
    "context"
    "yourproject/internal/store"
)

// UserStore is a mock implementation of store.UserStore.
// Each method is backed by a function field that tests can set.
type UserStore struct {
    UpsertUserFunc        func(ctx context.Context, user *store.User) error
    GetUserFunc           func(ctx context.Context, userID string) (*store.User, error)
    UpdatePreferencesFunc func(ctx context.Context, userID string, prefs *store.UserPreferences) error
    DeleteUserFunc        func(ctx context.Context, userID string) error
}

// Compile-time check: ensure MockUserStore implements store.UserStore.
var _ store.UserStore = (*UserStore)(nil)

func (m *UserStore) UpsertUser(ctx context.Context, user *store.User) error {
    if m.UpsertUserFunc != nil {
        return m.UpsertUserFunc(ctx, user)
    }
    return nil
}

func (m *UserStore) GetUser(ctx context.Context, userID string) (*store.User, error) {
    if m.GetUserFunc != nil {
        return m.GetUserFunc(ctx, userID)
    }
    return nil, &store.ErrNotFound{Resource: "user", ID: userID}
}

func (m *UserStore) UpdatePreferences(ctx context.Context, userID string, prefs *store.UserPreferences) error {
    if m.UpdatePreferencesFunc != nil {
        return m.UpdatePreferencesFunc(ctx, userID, prefs)
    }
    return nil
}

func (m *UserStore) DeleteUser(ctx context.Context, userID string) error {
    if m.DeleteUserFunc != nil {
        return m.DeleteUserFunc(ctx, userID)
    }
    return nil
}
```

### Using Manual Mocks in Tests

```go
package handler_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "yourproject/internal/handler"
    "yourproject/internal/mock"
    "yourproject/internal/store"
)

func TestGetUser_Success(t *testing.T) {
    // Arrange: set up the mock to return a specific user
    mockStore := &mock.UserStore{
        GetUserFunc: func(ctx context.Context, userID string) (*store.User, error) {
            name := "Alice"
            return &store.User{
                ID:        userID,
                Email:     "alice@example.com",
                Name:      &name,
                CreatedAt: time.Now(),
                UpdatedAt: time.Now(),
            }, nil
        },
    }

    h := handler.NewUserHandler(mockStore)

    // Act: make the HTTP request
    req := httptest.NewRequest("GET", "/users/user-123", nil)
    req.SetPathValue("id", "user-123")
    rec := httptest.NewRecorder()

    h.GetUser(rec, req)

    // Assert: check the response
    if rec.Code != http.StatusOK {
        t.Errorf("expected status 200, got %d", rec.Code)
    }

    var user store.User
    if err := json.NewDecoder(rec.Body).Decode(&user); err != nil {
        t.Fatalf("failed to decode response: %v", err)
    }

    if user.Email != "alice@example.com" {
        t.Errorf("expected email alice@example.com, got %s", user.Email)
    }
}

func TestGetUser_NotFound(t *testing.T) {
    mockStore := &mock.UserStore{
        GetUserFunc: func(ctx context.Context, userID string) (*store.User, error) {
            return nil, &store.ErrNotFound{Resource: "user", ID: userID}
        },
    }

    h := handler.NewUserHandler(mockStore)

    req := httptest.NewRequest("GET", "/users/nonexistent", nil)
    req.SetPathValue("id", "nonexistent")
    rec := httptest.NewRecorder()

    h.GetUser(rec, req)

    if rec.Code != http.StatusNotFound {
        t.Errorf("expected status 404, got %d", rec.Code)
    }
}

func TestGetUser_InternalError(t *testing.T) {
    mockStore := &mock.UserStore{
        GetUserFunc: func(ctx context.Context, userID string) (*store.User, error) {
            return nil, fmt.Errorf("database connection lost")
        },
    }

    h := handler.NewUserHandler(mockStore)

    req := httptest.NewRequest("GET", "/users/user-123", nil)
    req.SetPathValue("id", "user-123")
    rec := httptest.NewRecorder()

    h.GetUser(rec, req)

    if rec.Code != http.StatusInternalServerError {
        t.Errorf("expected status 500, got %d", rec.Code)
    }
}
```

### Strategy 2: In-Memory Repository

For integration-style tests that need stateful behavior (insert, then read back), create an in-memory implementation:

```go
package mock

import (
    "context"
    "sync"
    "time"

    "yourproject/internal/store"
)

// InMemoryUserStore is a thread-safe in-memory implementation of store.UserStore.
type InMemoryUserStore struct {
    mu    sync.RWMutex
    users map[string]*store.User
}

var _ store.UserStore = (*InMemoryUserStore)(nil)

func NewInMemoryUserStore() *InMemoryUserStore {
    return &InMemoryUserStore{
        users: make(map[string]*store.User),
    }
}

func (s *InMemoryUserStore) UpsertUser(ctx context.Context, user *store.User) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    existing, exists := s.users[user.ID]
    if exists {
        // Update
        existing.Email = user.Email
        if user.Name != nil {
            existing.Name = user.Name
        }
        if user.Picture != nil {
            existing.Picture = user.Picture
        }
        existing.UpdatedAt = time.Now()
        user.CreatedAt = existing.CreatedAt
        user.UpdatedAt = existing.UpdatedAt
    } else {
        // Insert
        now := time.Now()
        user.CreatedAt = now
        user.UpdatedAt = now
        // Store a copy to avoid external mutation
        stored := *user
        s.users[user.ID] = &stored
    }

    return nil
}

func (s *InMemoryUserStore) GetUser(ctx context.Context, userID string) (*store.User, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    user, exists := s.users[userID]
    if !exists {
        return nil, &store.ErrNotFound{Resource: "user", ID: userID}
    }

    // Return a copy to avoid external mutation
    result := *user
    return &result, nil
}

func (s *InMemoryUserStore) UpdatePreferences(ctx context.Context, userID string, prefs *store.UserPreferences) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    _, exists := s.users[userID]
    if !exists {
        return &store.ErrNotFound{Resource: "user", ID: userID}
    }

    // In a real implementation, you would store preferences
    // For this mock, we just verify the user exists
    return nil
}

func (s *InMemoryUserStore) DeleteUser(ctx context.Context, userID string) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.users[userID]; !exists {
        return &store.ErrNotFound{Resource: "user", ID: userID}
    }

    delete(s.users, userID)
    return nil
}
```

### Using In-Memory Repository in Tests

```go
func TestUpsertAndGetUser(t *testing.T) {
    store := mock.NewInMemoryUserStore()
    ctx := context.Background()

    name := "Alice"
    user := &store.User{
        ID:    "user-123",
        Email: "alice@example.com",
        Name:  &name,
    }

    // Insert
    if err := store.UpsertUser(ctx, user); err != nil {
        t.Fatalf("upsert user: %v", err)
    }

    // Read back
    got, err := store.GetUser(ctx, "user-123")
    if err != nil {
        t.Fatalf("get user: %v", err)
    }

    if got.Email != "alice@example.com" {
        t.Errorf("expected email alice@example.com, got %s", got.Email)
    }
    if *got.Name != "Alice" {
        t.Errorf("expected name Alice, got %s", *got.Name)
    }

    // Update via upsert
    newName := "Alice Smith"
    user.Name = &newName
    if err := store.UpsertUser(ctx, user); err != nil {
        t.Fatalf("upsert user (update): %v", err)
    }

    got, err = store.GetUser(ctx, "user-123")
    if err != nil {
        t.Fatalf("get user after update: %v", err)
    }
    if *got.Name != "Alice Smith" {
        t.Errorf("expected name Alice Smith, got %s", *got.Name)
    }
}
```

### Strategy 3: Integration Tests with a Real Database

For critical database operations (complex queries, JSONB, triggers), you should also test against a real PostgreSQL instance. Use `testcontainers` to spin up a PostgreSQL container for the test:

```go
package repository_test

import (
    "context"
    "testing"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"

    "yourproject/internal/database"
    "yourproject/internal/repository"
    "yourproject/internal/store"
)

func setupTestDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := context.Background()

    // Start a PostgreSQL container
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(5*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("start postgres container: %v", err)
    }

    t.Cleanup(func() {
        pgContainer.Terminate(ctx)
    })

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("get connection string: %v", err)
    }

    pool, err := pgxpool.New(ctx, connStr)
    if err != nil {
        t.Fatalf("create pool: %v", err)
    }

    t.Cleanup(func() {
        pool.Close()
    })

    // Run migrations
    if err := database.RunMigrations(pool); err != nil {
        t.Fatalf("run migrations: %v", err)
    }

    return pool
}

func TestUserRepository_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    pool := setupTestDB(t)
    repo := repository.NewUserRepository(pool)
    ctx := context.Background()

    // Test CreateUser
    name := "Alice"
    user := &store.User{
        ID:    "user-integration-test",
        Email: "alice@integration.test",
        Name:  &name,
    }

    err := repo.UpsertUser(ctx, user)
    if err != nil {
        t.Fatalf("upsert user: %v", err)
    }

    // Test GetUser
    got, err := repo.GetUser(ctx, user.ID)
    if err != nil {
        t.Fatalf("get user: %v", err)
    }

    if got.Email != user.Email {
        t.Errorf("email: got %s, want %s", got.Email, user.Email)
    }
    if *got.Name != *user.Name {
        t.Errorf("name: got %s, want %s", *got.Name, *user.Name)
    }

    // Verify timestamps were set by the database
    if got.CreatedAt.IsZero() {
        t.Error("created_at should be set by database")
    }
}
```

### Node.js Comparison

```javascript
// Node.js mock with Jest
jest.mock('../repositories/userRepository');
const userRepository = require('../repositories/userRepository');

userRepository.getUser.mockResolvedValue({
    id: 'user-123',
    email: 'alice@example.com',
    name: 'Alice',
});

// Or with a mock class
class MockUserRepository {
    constructor() {
        this.users = new Map();
    }
    async getUser(userId) {
        return this.users.get(userId) || null;
    }
}
```

Go's approach is more explicit. There is no runtime mocking framework that intercepts function calls. You define a mock struct that implements the interface, and you can set its behavior per-test. This is less magical but more debuggable -- when something fails, you know exactly which mock function was called.

---

## 17. Real-World Example: Complete Repository Layer

This section brings everything together into a complete, production-ready repository layer. This is the kind of code you would find in a real Go API project.

### Project Structure

```
krafty-core/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── database/
│   │   ├── database.go          # Connection pool factory
│   │   ├── migrate.go           # Migration runner
│   │   ├── tx.go                # Transaction helper
│   │   └── migrations/
│   │       ├── 001_create_users.sql
│   │       ├── 002_create_sessions.sql
│   │       ├── 003_create_projects.sql
│   │       └── 004_create_triggers.sql
│   ├── store/
│   │   ├── models.go            # Domain models (structs)
│   │   ├── interfaces.go        # Repository interfaces
│   │   └── errors.go            # Domain error types
│   ├── repository/
│   │   ├── user.go              # PostgreSQL user repository
│   │   ├── session.go           # PostgreSQL session repository
│   │   ├── project.go           # PostgreSQL project repository
│   │   └── storage.go           # Aggregate storage
│   ├── handler/
│   │   ├── user.go              # HTTP handlers
│   │   └── health.go            # Health check handlers
│   └── mock/
│       ├── user_store.go        # Mock user store
│       └── session_store.go     # Mock session store
├── go.mod
└── go.sum
```

### File: internal/store/errors.go

```go
package store

import "fmt"

// ErrNotFound indicates that the requested resource does not exist.
type ErrNotFound struct {
    Resource string
    ID       string
}

func (e *ErrNotFound) Error() string {
    return fmt.Sprintf("%s with id '%s' not found", e.Resource, e.ID)
}

// ErrDuplicate indicates a unique constraint violation.
type ErrDuplicate struct {
    Resource string
    Field    string
    Value    string
}

func (e *ErrDuplicate) Error() string {
    return fmt.Sprintf("%s with %s '%s' already exists", e.Resource, e.Field, e.Value)
}
```

### File: internal/store/models.go

```go
package store

import "time"

// User represents a user account.
type User struct {
    ID        string    `json:"id"        db:"id"`
    Email     string    `json:"email"     db:"email"`
    Name      *string   `json:"name,omitempty"      db:"name"`
    Picture   *string   `json:"picture,omitempty"   db:"picture"`
    CreatedAt time.Time `json:"createdAt" db:"created_at"`
    UpdatedAt time.Time `json:"updatedAt" db:"updated_at"`
}

// Session represents an authenticated session.
type Session struct {
    ID           string    `json:"id"           db:"id"`
    UserID       string    `json:"userId"       db:"user_id"`
    RefreshToken string    `json:"-"            db:"refresh_token"`
    UserAgent    *string   `json:"userAgent,omitempty"   db:"user_agent"`
    IPAddress    *string   `json:"ipAddress,omitempty"   db:"ip_address"`
    ExpiresAt    time.Time `json:"expiresAt"    db:"expires_at"`
    CreatedAt    time.Time `json:"createdAt"    db:"created_at"`
    UpdatedAt    time.Time `json:"updatedAt"    db:"updated_at"`
}

// Project represents a user-owned project.
type Project struct {
    ID          string          `json:"id"          db:"id"`
    UserID      string          `json:"userId"      db:"user_id"`
    Name        string          `json:"name"        db:"name"`
    Description *string         `json:"description,omitempty" db:"description"`
    Settings    ProjectSettings `json:"settings"    db:"settings"`
    IsArchived  bool            `json:"isArchived"  db:"is_archived"`
    CreatedAt   time.Time       `json:"createdAt"   db:"created_at"`
    UpdatedAt   time.Time       `json:"updatedAt"   db:"updated_at"`
}

// ProjectSettings is stored as JSONB in PostgreSQL.
type ProjectSettings struct {
    Theme       string   `json:"theme,omitempty"`
    Language    string   `json:"language,omitempty"`
    IsPublic    bool     `json:"isPublic"`
    MaxFileSize int      `json:"maxFileSize,omitempty"`
    Tags        []string `json:"tags,omitempty"`
}

// UserPreferences represents configurable user preferences.
type UserPreferences struct {
    Theme    string `json:"theme"`
    Language string `json:"language"`
    Timezone string `json:"timezone"`
}
```

### File: internal/store/interfaces.go

```go
package store

import "context"

// UserStore defines the data access operations for users.
type UserStore interface {
    UpsertUser(ctx context.Context, user *User) error
    GetUser(ctx context.Context, userID string) (*User, error)
    GetUserByEmail(ctx context.Context, email string) (*User, error)
    UpdatePreferences(ctx context.Context, userID string, prefs *UserPreferences) error
    DeleteUser(ctx context.Context, userID string) error
}

// SessionStore defines the data access operations for sessions.
type SessionStore interface {
    CreateSession(ctx context.Context, session *Session) error
    GetSession(ctx context.Context, sessionID string) (*Session, error)
    GetSessionsByUser(ctx context.Context, userID string) ([]*Session, error)
    DeleteSession(ctx context.Context, sessionID string) error
    DeleteUserSessions(ctx context.Context, userID string) (int64, error)
    DeleteExpiredSessions(ctx context.Context) (int64, error)
}

// ProjectStore defines the data access operations for projects.
type ProjectStore interface {
    CreateProject(ctx context.Context, project *Project) error
    GetProject(ctx context.Context, projectID string) (*Project, error)
    ListProjectsByUser(ctx context.Context, userID string, includeArchived bool) ([]*Project, error)
    UpdateProject(ctx context.Context, project *Project) error
    ArchiveProject(ctx context.Context, projectID string) error
    DeleteProject(ctx context.Context, projectID string) error
}

// Storage combines all store interfaces.
type Storage interface {
    UserStore
    SessionStore
    ProjectStore
}
```

### File: internal/database/database.go

```go
package database

import (
    "context"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

// Config holds database configuration.
type Config struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

// DefaultConfig returns sensible defaults.
func DefaultConfig(dsn string) Config {
    return Config{
        DSN:             dsn,
        MaxOpenConns:    25,
        MaxIdleConns:    10,
        ConnMaxLifetime: 5 * time.Minute,
        ConnMaxIdleTime: 1 * time.Minute,
    }
}

// New creates a new connection pool.
func New(cfg Config) (*pgxpool.Pool, error) {
    poolCfg, err := pgxpool.ParseConfig(cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("parse dsn: %w", err)
    }

    poolCfg.MaxConns = int32(cfg.MaxOpenConns)
    poolCfg.MinConns = int32(cfg.MaxIdleConns)
    poolCfg.MaxConnLifetime = cfg.ConnMaxLifetime
    poolCfg.MaxConnIdleTime = cfg.ConnMaxIdleTime
    poolCfg.HealthCheckPeriod = 30 * time.Second

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }

    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping: %w", err)
    }

    return pool, nil
}
```

### File: internal/database/tx.go

```go
package database

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    "github.com/jackc/pgx/v5/pgxpool"
)

// DBTX is an interface satisfied by both *pgxpool.Pool and pgx.Tx.
type DBTX interface {
    Exec(ctx context.Context, sql string, arguments ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

// WithTx executes fn within a database transaction.
func WithTx(ctx context.Context, pool *pgxpool.Pool, fn func(tx pgx.Tx) error) error {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback(ctx)

    if err := fn(tx); err != nil {
        return err
    }

    if err := tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit tx: %w", err)
    }

    return nil
}
```

### File: internal/database/migrate.go

```go
package database

import (
    "embed"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/jackc/pgx/v5/stdlib"
    "github.com/pressly/goose/v3"
)

//go:embed migrations/*.sql
var migrations embed.FS

// RunMigrations applies all pending database migrations.
func RunMigrations(pool *pgxpool.Pool) error {
    db := stdlib.OpenDBFromPool(pool)
    defer db.Close()

    goose.SetBaseFS(migrations)

    if err := goose.SetDialect("postgres"); err != nil {
        return fmt.Errorf("set dialect: %w", err)
    }

    if err := goose.Up(db, "migrations"); err != nil {
        return fmt.Errorf("run migrations: %w", err)
    }

    return nil
}
```

### File: internal/repository/user.go

```go
package repository

import (
    "context"
    "errors"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/store"
)

// UserRepository implements store.UserStore with PostgreSQL.
type UserRepository struct {
    db *pgxpool.Pool
}

var _ store.UserStore = (*UserRepository)(nil)

// NewUserRepository creates a new UserRepository.
func NewUserRepository(db *pgxpool.Pool) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) UpsertUser(ctx context.Context, user *store.User) error {
    query := `
        INSERT INTO users (id, email, name, picture, updated_at)
        VALUES ($1, $2, $3, $4, NOW())
        ON CONFLICT(id) DO UPDATE SET
            email = excluded.email,
            name = COALESCE(excluded.name, users.name),
            picture = COALESCE(excluded.picture, users.picture),
            updated_at = NOW()
        RETURNING created_at, updated_at`

    err := r.db.QueryRow(ctx, query,
        user.ID, user.Email, user.Name, user.Picture,
    ).Scan(&user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23505" {
            return &store.ErrDuplicate{
                Resource: "user",
                Field:    "email",
                Value:    user.Email,
            }
        }
        return fmt.Errorf("upsert user: %w", err)
    }

    return nil
}

func (r *UserRepository) GetUser(ctx context.Context, userID string) (*store.User, error) {
    var user store.User
    err := r.db.QueryRow(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users WHERE id = $1`, userID,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.Picture,
        &user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "user", ID: userID}
        }
        return nil, fmt.Errorf("get user: %w", err)
    }

    return &user, nil
}

func (r *UserRepository) GetUserByEmail(ctx context.Context, email string) (*store.User, error) {
    var user store.User
    err := r.db.QueryRow(ctx,
        `SELECT id, email, name, picture, created_at, updated_at
         FROM users WHERE email = $1`, email,
    ).Scan(&user.ID, &user.Email, &user.Name, &user.Picture,
        &user.CreatedAt, &user.UpdatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "user", ID: email}
        }
        return nil, fmt.Errorf("get user by email: %w", err)
    }

    return &user, nil
}

func (r *UserRepository) UpdatePreferences(
    ctx context.Context,
    userID string,
    prefs *store.UserPreferences,
) error {
    query := `
        UPDATE users
        SET preferences = $1,
            updated_at = NOW()
        WHERE id = $2`

    result, err := r.db.Exec(ctx, query, prefs, userID)
    if err != nil {
        return fmt.Errorf("update preferences: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "user", ID: userID}
    }

    return nil
}

func (r *UserRepository) DeleteUser(ctx context.Context, userID string) error {
    result, err := r.db.Exec(ctx,
        `DELETE FROM users WHERE id = $1`, userID,
    )
    if err != nil {
        return fmt.Errorf("delete user: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "user", ID: userID}
    }

    return nil
}
```

### File: internal/repository/session.go

```go
package repository

import (
    "context"
    "errors"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/store"
)

// SessionRepository implements store.SessionStore with PostgreSQL.
type SessionRepository struct {
    db *pgxpool.Pool
}

var _ store.SessionStore = (*SessionRepository)(nil)

func NewSessionRepository(db *pgxpool.Pool) *SessionRepository {
    return &SessionRepository{db: db}
}

func (r *SessionRepository) CreateSession(ctx context.Context, session *store.Session) error {
    query := `
        INSERT INTO sessions (id, user_id, refresh_token, user_agent, ip_address, expires_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING created_at, updated_at`

    err := r.db.QueryRow(ctx, query,
        session.ID, session.UserID, session.RefreshToken,
        session.UserAgent, session.IPAddress, session.ExpiresAt,
    ).Scan(&session.CreatedAt, &session.UpdatedAt)

    if err != nil {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23503" {
            return &store.ErrNotFound{Resource: "user", ID: session.UserID}
        }
        return fmt.Errorf("create session: %w", err)
    }

    return nil
}

func (r *SessionRepository) GetSession(ctx context.Context, sessionID string) (*store.Session, error) {
    var s store.Session
    err := r.db.QueryRow(ctx,
        `SELECT id, user_id, refresh_token, user_agent, ip_address,
                expires_at, created_at, updated_at
         FROM sessions WHERE id = $1`, sessionID,
    ).Scan(&s.ID, &s.UserID, &s.RefreshToken, &s.UserAgent,
        &s.IPAddress, &s.ExpiresAt, &s.CreatedAt, &s.UpdatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "session", ID: sessionID}
        }
        return nil, fmt.Errorf("get session: %w", err)
    }

    return &s, nil
}

func (r *SessionRepository) GetSessionsByUser(
    ctx context.Context,
    userID string,
) ([]*store.Session, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, user_id, refresh_token, user_agent, ip_address,
                expires_at, created_at, updated_at
         FROM sessions
         WHERE user_id = $1
         ORDER BY created_at DESC`, userID,
    )
    if err != nil {
        return nil, fmt.Errorf("query sessions: %w", err)
    }
    defer rows.Close()

    var sessions []*store.Session
    for rows.Next() {
        var s store.Session
        if err := rows.Scan(
            &s.ID, &s.UserID, &s.RefreshToken, &s.UserAgent,
            &s.IPAddress, &s.ExpiresAt, &s.CreatedAt, &s.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan session: %w", err)
        }
        sessions = append(sessions, &s)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterate sessions: %w", err)
    }

    return sessions, nil
}

func (r *SessionRepository) DeleteSession(ctx context.Context, sessionID string) error {
    result, err := r.db.Exec(ctx,
        `DELETE FROM sessions WHERE id = $1`, sessionID,
    )
    if err != nil {
        return fmt.Errorf("delete session: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "session", ID: sessionID}
    }

    return nil
}

func (r *SessionRepository) DeleteUserSessions(ctx context.Context, userID string) (int64, error) {
    result, err := r.db.Exec(ctx,
        `DELETE FROM sessions WHERE user_id = $1`, userID,
    )
    if err != nil {
        return 0, fmt.Errorf("delete user sessions: %w", err)
    }

    return result.RowsAffected(), nil
}

func (r *SessionRepository) DeleteExpiredSessions(ctx context.Context) (int64, error) {
    result, err := r.db.Exec(ctx,
        `DELETE FROM sessions WHERE expires_at < NOW()`,
    )
    if err != nil {
        return 0, fmt.Errorf("delete expired sessions: %w", err)
    }

    return result.RowsAffected(), nil
}
```

### File: internal/repository/project.go

```go
package repository

import (
    "context"
    "errors"
    "fmt"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/store"
)

// ProjectRepository implements store.ProjectStore with PostgreSQL.
type ProjectRepository struct {
    db *pgxpool.Pool
}

var _ store.ProjectStore = (*ProjectRepository)(nil)

func NewProjectRepository(db *pgxpool.Pool) *ProjectRepository {
    return &ProjectRepository{db: db}
}

func (r *ProjectRepository) CreateProject(ctx context.Context, project *store.Project) error {
    query := `
        INSERT INTO projects (id, user_id, name, description, settings, is_archived)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING created_at, updated_at`

    err := r.db.QueryRow(ctx, query,
        project.ID, project.UserID, project.Name,
        project.Description, project.Settings, project.IsArchived,
    ).Scan(&project.CreatedAt, &project.UpdatedAt)

    if err != nil {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) {
            switch pgErr.Code {
            case "23503":
                return &store.ErrNotFound{Resource: "user", ID: project.UserID}
            case "23505":
                return &store.ErrDuplicate{Resource: "project", Field: "id", Value: project.ID}
            }
        }
        return fmt.Errorf("create project: %w", err)
    }

    return nil
}

func (r *ProjectRepository) GetProject(ctx context.Context, projectID string) (*store.Project, error) {
    var p store.Project
    err := r.db.QueryRow(ctx,
        `SELECT id, user_id, name, description, settings, is_archived, created_at, updated_at
         FROM projects WHERE id = $1`, projectID,
    ).Scan(&p.ID, &p.UserID, &p.Name, &p.Description,
        &p.Settings, &p.IsArchived, &p.CreatedAt, &p.UpdatedAt)

    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, &store.ErrNotFound{Resource: "project", ID: projectID}
        }
        return nil, fmt.Errorf("get project: %w", err)
    }

    return &p, nil
}

func (r *ProjectRepository) ListProjectsByUser(
    ctx context.Context,
    userID string,
    includeArchived bool,
) ([]*store.Project, error) {
    query := `
        SELECT id, user_id, name, description, settings, is_archived, created_at, updated_at
        FROM projects
        WHERE user_id = $1`

    args := []any{userID}

    if !includeArchived {
        query += " AND is_archived = false"
    }

    query += " ORDER BY created_at DESC"

    rows, err := r.db.Query(ctx, query, args...)
    if err != nil {
        return nil, fmt.Errorf("list projects: %w", err)
    }
    defer rows.Close()

    var projects []*store.Project
    for rows.Next() {
        var p store.Project
        if err := rows.Scan(
            &p.ID, &p.UserID, &p.Name, &p.Description,
            &p.Settings, &p.IsArchived, &p.CreatedAt, &p.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan project: %w", err)
        }
        projects = append(projects, &p)
    }

    return projects, rows.Err()
}

func (r *ProjectRepository) UpdateProject(ctx context.Context, project *store.Project) error {
    query := `
        UPDATE projects
        SET name = $1, description = $2, settings = $3, is_archived = $4, updated_at = NOW()
        WHERE id = $5`

    result, err := r.db.Exec(ctx, query,
        project.Name, project.Description, project.Settings,
        project.IsArchived, project.ID,
    )
    if err != nil {
        return fmt.Errorf("update project: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "project", ID: project.ID}
    }

    return nil
}

func (r *ProjectRepository) ArchiveProject(ctx context.Context, projectID string) error {
    result, err := r.db.Exec(ctx,
        `UPDATE projects SET is_archived = true, updated_at = NOW() WHERE id = $1`,
        projectID,
    )
    if err != nil {
        return fmt.Errorf("archive project: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "project", ID: projectID}
    }

    return nil
}

func (r *ProjectRepository) DeleteProject(ctx context.Context, projectID string) error {
    result, err := r.db.Exec(ctx,
        `DELETE FROM projects WHERE id = $1`, projectID,
    )
    if err != nil {
        return fmt.Errorf("delete project: %w", err)
    }

    if result.RowsAffected() == 0 {
        return &store.ErrNotFound{Resource: "project", ID: projectID}
    }

    return nil
}
```

### File: internal/repository/storage.go

```go
package repository

import (
    "github.com/jackc/pgx/v5/pgxpool"
    "yourproject/internal/store"
)

// PostgresStorage implements store.Storage by embedding all repositories.
type PostgresStorage struct {
    *UserRepository
    *SessionRepository
    *ProjectRepository
}

var _ store.Storage = (*PostgresStorage)(nil)

// NewPostgresStorage creates a complete storage layer backed by PostgreSQL.
func NewPostgresStorage(pool *pgxpool.Pool) *PostgresStorage {
    return &PostgresStorage{
        UserRepository:    NewUserRepository(pool),
        SessionRepository: NewSessionRepository(pool),
        ProjectRepository: NewProjectRepository(pool),
    }
}
```

### File: cmd/api/main.go

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "yourproject/internal/database"
    "yourproject/internal/handler"
    "yourproject/internal/repository"
    "yourproject/internal/store"
)

func main() {
    // Logger
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    // Database
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        dsn = "postgres://admin:secret@localhost:5432/krafty?sslmode=disable"
    }

    dbCfg := database.DefaultConfig(dsn)
    pool, err := database.New(dbCfg)
    if err != nil {
        logger.Error("failed to connect to database", "error", err)
        os.Exit(1)
    }
    defer pool.Close()
    logger.Info("connected to database")

    // Migrations
    if err := database.RunMigrations(pool); err != nil {
        logger.Error("failed to run migrations", "error", err)
        os.Exit(1)
    }
    logger.Info("migrations complete")

    // Storage layer
    storage := repository.NewPostgresStorage(pool)

    // Health handler
    healthHandler := handler.NewHealthHandler(pool)

    // Routes
    mux := http.NewServeMux()
    mux.HandleFunc("GET /healthz", healthHandler.Liveness)
    mux.HandleFunc("GET /readyz", healthHandler.Readiness)

    // User routes (simplified -- in production, add auth middleware)
    mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        userID := r.PathValue("id")
        user, err := storage.GetUser(r.Context(), userID)
        if err != nil {
            handleError(w, err)
            return
        }
        writeJSON(w, http.StatusOK, user)
    })

    mux.HandleFunc("GET /api/users/{userId}/projects", func(w http.ResponseWriter, r *http.Request) {
        userID := r.PathValue("userId")
        projects, err := storage.ListProjectsByUser(r.Context(), userID, false)
        if err != nil {
            handleError(w, err)
            return
        }
        writeJSON(w, http.StatusOK, projects)
    })

    // Server
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Start
    go func() {
        logger.Info("starting server", "addr", server.Addr)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    logger.Info("shutting down server")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        logger.Error("server shutdown error", "error", err)
    }

    pool.Close()
    logger.Info("server stopped")
}

func handleError(w http.ResponseWriter, err error) {
    var notFound *store.ErrNotFound
    var duplicate *store.ErrDuplicate

    switch {
    case errors.As(err, &notFound):
        writeJSON(w, http.StatusNotFound, map[string]string{"error": notFound.Error()})
    case errors.As(err, &duplicate):
        writeJSON(w, http.StatusConflict, map[string]string{"error": duplicate.Error()})
    default:
        writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal server error"})
    }
}

func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

### Putting It All Together: The Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                     Complete Request Flow                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Client sends:  GET /api/users/abc123                         │
│                                                                   │
│  2. HTTP Router matches the route, extracts {id} = "abc123"     │
│                                                                   │
│  3. Handler calls:                                                │
│     storage.GetUser(r.Context(), "abc123")                       │
│                                                                   │
│  4. storage.GetUser is actually UserRepository.GetUser because   │
│     PostgresStorage embeds *UserRepository                       │
│                                                                   │
│  5. UserRepository.GetUser executes:                             │
│     SELECT id, email, name, ... FROM users WHERE id = $1        │
│     with context from the HTTP request                           │
│                                                                   │
│  6. pgx sends the query over a pooled connection to PostgreSQL   │
│                                                                   │
│  7. PostgreSQL returns the row (or no row)                       │
│                                                                   │
│  8. pgx scans the row into a store.User struct                   │
│     - If no row: returns ErrNotFound                             │
│     - If error: returns wrapped error                            │
│                                                                   │
│  9. Handler checks the error:                                     │
│     - ErrNotFound → 404 JSON response                            │
│     - Other error → 500 JSON response                            │
│     - No error → 200 JSON response with user data               │
│                                                                   │
│  10. If client disconnects at any point, r.Context() is          │
│      cancelled, pgx cancels the PostgreSQL query, and the        │
│      connection is returned to the pool.                         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 18. Key Takeaways

1. **The repository pattern separates data access from business logic.** Define interfaces where they are used (in the store/domain package), implement them in the repository package. Your handlers and services never import `pgx` directly.

2. **pgx is the best PostgreSQL driver for Go.** It speaks the PostgreSQL wire protocol natively, supports all PostgreSQL types (JSONB, arrays, etc.), and provides a production-ready connection pool via `pgxpool`.

3. **Connection pool configuration matters in production.** Set `MaxConns` based on `max_connections / instances`. Set `MinConns` to your steady-state concurrency. Set `MaxConnLifetime` to 5 minutes for connection rotation. Monitor `EmptyAcquireCount` and `AcquireDuration` to detect pool exhaustion.

4. **Embed migrations in your binary with `go:embed`.** Goose + embedded SQL files means your application binary is self-contained. Migrations run automatically on startup, ensuring the database schema always matches the deployed code.

5. **Use pointers for nullable database columns.** `*string` maps naturally to SQL NULL, works with pgx scanning, and serializes correctly with `json:"field,omitempty"`. Avoid `sql.NullString` with pgx.

6. **The compile-time interface check `var _ Store = (*Repo)(nil)` catches missing methods immediately.** Without it, you discover missing methods only when someone tries to use the concrete type as the interface, potentially far from the repository file.

7. **Always use `ON CONFLICT` for upserts instead of SELECT-then-INSERT.** The upsert is atomic (no race conditions), single query (faster), and handles concurrent requests correctly. Use `COALESCE` to preserve existing values when the new value is NULL.

8. **Handle PostgreSQL errors with `errors.As(err, &pgconn.PgError{})`** and check the `Code` field. The most important codes are `23505` (unique violation), `23503` (foreign key violation), and `23502` (not null violation). Translate these into domain error types that your HTTP handlers can map to HTTP status codes.

9. **Every database method must accept `context.Context` as its first parameter.** This enables request cancellation (client disconnects), query timeouts, and distributed tracing. Never create a new `context.Background()` inside a repository method.

10. **Always close rows with `defer rows.Close()` and check `rows.Err()` after the loop.** Forgetting `rows.Close()` leaks database connections. Forgetting `rows.Err()` misses errors that occurred during iteration.

11. **Use PostgreSQL triggers for `updated_at` timestamps.** A `BEFORE UPDATE` trigger that sets `updated_at = NOW()` cannot be forgotten. Even if a developer writes a new UPDATE query months later, the timestamp is always correct.

12. **Use the `WithTx` helper for transactions.** The pattern `defer tx.Rollback(ctx)` ensures cleanup on any error path, and is a no-op after a successful commit. The `DBTX` interface lets repository methods work with both pools and transactions.

13. **Use PostgreSQL JSONB for semi-structured data.** pgx automatically marshals Go structs to/from JSONB. Use `@>` for containment queries, `||` for merging, and `jsonb_set` for updating specific paths. Add GIN indexes for query performance.

14. **Mock repositories enable fast, deterministic unit tests.** Use function-field mocks for simple tests, in-memory repositories for stateful tests, and `testcontainers` for integration tests against real PostgreSQL. Each has its place.

15. **Health checks should verify database connectivity.** The liveness probe should be trivial (process is alive). The readiness probe should ping the database with a 3-second timeout. Expose pool statistics for monitoring but do not expose them publicly.

---

## 19. Practice Exercises

### Exercise 1: Blog Repository

Build a complete repository layer for a blog application with the following tables:

- `authors` -- id, name, email, bio, created_at, updated_at
- `posts` -- id, author_id (FK), title, slug (unique), content, status (draft/published), published_at, created_at, updated_at
- `comments` -- id, post_id (FK), author_name, content, created_at

Requirements:
- Write Goose migration files for all tables, including the `updated_at` trigger
- Define domain models with proper JSON tags and nullable fields
- Define repository interfaces for each entity
- Implement the PostgreSQL repositories with full CRUD operations
- Implement `GetPostBySlug` that returns the post with its author's name (JOIN)
- Implement `ListPostsByStatus` that filters by status
- Handle all PostgreSQL errors (unique slug violation, foreign key violations)
- Write mock implementations for all interfaces
- Write table-driven unit tests for at least one handler using the mocks

### Exercise 2: JSONB Preferences

Extend the user model with a `preferences` JSONB column that stores:

```go
type Preferences struct {
    Theme         string            `json:"theme"`
    Language      string            `json:"language"`
    Timezone      string            `json:"timezone"`
    Notifications NotificationPrefs `json:"notifications"`
    Features      map[string]bool   `json:"features"`
}
```

Requirements:
- Write a migration that adds the `preferences` column with a default value of `'{}'`
- Implement `UpdatePreferences` that replaces the entire preferences object
- Implement `UpdateNotificationPrefs` that updates only the notifications sub-object using `jsonb_set`
- Implement `ToggleFeature` that sets a single key in the features map using `jsonb_set`
- Implement `GetUsersWithFeature` that queries for users where a specific feature flag is true using the `@>` operator
- Write unit tests for all of these operations

### Exercise 3: Transaction Challenge

Build an "order processing" system where placing an order requires multiple atomic operations:

- `products` table with id, name, price, stock_count
- `orders` table with id, user_id, total_amount, status, created_at
- `order_items` table with id, order_id, product_id, quantity, unit_price

Requirements:
- Implement `PlaceOrder(ctx, userID, items []OrderItem)` as a single transaction that:
  1. Checks stock for all items (SELECT ... FOR UPDATE to prevent races)
  2. Decrements stock for each item
  3. Creates the order record
  4. Creates order_item records for each item
  5. Returns the complete order with all items
- If any item is out of stock, the entire transaction is rolled back
- Use the `DBTX` interface so repository methods work both inside and outside transactions
- Write tests that verify: successful order, out-of-stock rollback, and concurrent order handling

### Exercise 4: Pagination and Cursor-Based Queries

Implement cursor-based pagination (used by production APIs for large result sets):

```go
type PageRequest struct {
    Cursor string // Opaque cursor (base64 encoded "created_at,id")
    Limit  int    // Page size (default 20, max 100)
}

type PageResponse[T any] struct {
    Items      []T    `json:"items"`
    NextCursor string `json:"nextCursor,omitempty"` // Empty if no more results
    HasMore    bool   `json:"hasMore"`
}
```

Requirements:
- Implement `ListProjectsPaginated(ctx, userID, page PageRequest) (*PageResponse[Project], error)`
- The cursor encodes the `created_at` timestamp and `id` of the last item (for stable ordering)
- Use `WHERE (created_at, id) < ($cursor_time, $cursor_id)` for cursor-based pagination
- Validate and sanitize the limit (minimum 1, maximum 100, default 20)
- Write tests that verify: first page, middle page, last page (hasMore=false), empty results, invalid cursor

### Exercise 5: Background Cleanup Worker

Build a background worker that periodically cleans up expired data:

Requirements:
- Create a `Cleaner` struct that runs on a configurable interval (e.g., every 5 minutes)
- It should delete expired sessions using `DeleteExpiredSessions`
- It should archive projects that have not been updated in 90 days
- It should use context for graceful shutdown (when the app shuts down, the cleaner stops)
- Log how many records were cleaned up each run
- Write the cleaner so it can be tested with the mock repositories and a fast tick interval

### Exercise 6: Full Integration Test Suite

Write a complete integration test suite using `testcontainers`:

Requirements:
- Start a PostgreSQL container before all tests
- Run migrations against the test database
- Test the full lifecycle: create user, create session, create project, update project, list projects, archive project, delete project, verify cascading deletes
- Test error cases: duplicate email, non-existent foreign key, deleting a user cascades to sessions and projects
- Test JSONB operations: store and retrieve complex settings, update specific fields, query by contained values
- Test transactions: create user + session atomically, verify rollback on failure
- Clean up between tests (truncate tables or use subtests with unique IDs)
- All tests should pass with `go test -v ./internal/repository/...`
