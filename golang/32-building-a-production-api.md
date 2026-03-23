# Chapter 32: Building a Production API — Putting It All Together

## Prerequisites

You should have completed (or be comfortable with) the concepts from every previous chapter in this guide. This is the capstone chapter -- it assumes mastery of Go fundamentals (Chapters 1-22) and familiarity with the real-world patterns introduced in Chapters 23-31. Specifically, you will use:

- **Chapter 23** -- Configuration management (environment variables, config structs, validation)
- **Chapter 24** -- HTTP routing (custom routers, path parameters, method-based dispatch)
- **Chapter 25** -- Middleware patterns (chaining, recovery, logging, CORS, rate limiting)
- **Chapter 26** -- Database access (connection pools, repositories, migrations, transactions)
- **Chapter 27** -- Authentication and authorization (JWT, bcrypt, token refresh, middleware)
- **Chapter 28** -- Structured logging (log/slog, request IDs, log levels, context-aware logging)
- **Chapter 29** -- Input validation and error handling (validation tags, custom validators, error responses)
- **Chapter 30** -- Graceful shutdown (signal handling, connection draining, cleanup)
- **Chapter 31** -- Concurrency patterns in services (background workers, parallel queries, worker pools)

If you are coming from Node.js/Express, this chapter will serve as a complete translation of how you would build a production Express API, but in idiomatic Go.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Project Structure](#2-project-structure)
3. [Configuration Layer](#3-configuration-layer)
4. [Database Layer](#4-database-layer)
5. [Domain Models](#5-domain-models)
6. [Repository Layer](#6-repository-layer)
7. [Authentication](#7-authentication)
8. [Validation and Responses](#8-validation-and-responses)
9. [Middleware Stack](#9-middleware-stack)
10. [Router](#10-router)
11. [Handlers](#11-handlers)
12. [Structured Logging](#12-structured-logging)
13. [Concurrency Patterns](#13-concurrency-patterns)
14. [Graceful Shutdown and Main](#14-graceful-shutdown-and-main)
15. [Testing the API](#15-testing-the-api)
16. [Docker Deployment](#16-docker-deployment)
17. [API Documentation](#17-api-documentation)
18. [What We Have Built](#18-what-we-have-built)
19. [Key Takeaways](#19-key-takeaways)
20. [Practice Exercises](#20-practice-exercises)

---

## 1. Project Overview

### What We Are Building

We are building a **Task Management API** -- a simplified project tracker similar to Jira, Linear, or Asana. This is not a toy example. It is a complete, production-grade REST API that demonstrates every pattern you have learned across this entire study guide.

The API manages four core resources:

- **Users** -- Registration, authentication, profile management
- **Projects** -- Containers for tasks, owned by users, with team membership
- **Tasks** -- Work items within projects, with status, priority, assignment, and due dates
- **Comments** -- Discussion threads attached to tasks

### Feature Set

| Category | Features |
|----------|----------|
| **Authentication** | User registration, login with JWT, token refresh, password hashing with bcrypt |
| **Projects** | Create, read, update, delete projects. List with pagination and filtering |
| **Tasks** | Full CRUD. Filter by status, priority, assignee. Search by title. Pagination |
| **Comments** | Add comments to tasks. List comments with pagination |
| **Dashboard** | Aggregated stats computed with parallel queries |
| **Background Jobs** | Periodic cleanup of expired tokens, overdue task notifications |
| **Observability** | Structured logging, request IDs, timing |
| **Resilience** | Panic recovery, rate limiting, graceful shutdown, connection draining |

### Architecture

The system follows a layered architecture. Each layer depends only on the layer directly below it, and dependencies flow inward:

```
Request → Middleware → Router → Handler → Service/Auth → Repository → Database
                                              ↑
                                          Validation
```

In more detail:

```
┌─────────────────────────────────────────────────────────┐
│                        main.go                          │
│  (wire everything together, start server, handle        │
│   signals, graceful shutdown)                           │
├─────────────────────────────────────────────────────────┤
│                     Middleware Stack                     │
│  Recovery → Logging → CORS → Rate Limit → Auth          │
├─────────────────────────────────────────────────────────┤
│                         Router                          │
│  Maps HTTP method + path → Handler function             │
├─────────────────────────────────────────────────────────┤
│                        Handlers                         │
│  Parse request, call service, write response            │
├─────────────────────────────────────────────────────────┤
│              Services / Auth / Validation                │
│  Business logic, JWT, input validation                  │
├─────────────────────────────────────────────────────────┤
│                      Repositories                       │
│  SQL queries, data access, transactions                 │
├─────────────────────────────────────────────────────────┤
│                       Database                          │
│  Connection pool, migrations                            │
├─────────────────────────────────────────────────────────┤
│                      Configuration                      │
│  Environment variables, defaults, validation            │
└─────────────────────────────────────────────────────────┘
```

### Go vs Node.js/Express Architecture Comparison

The layered architecture above maps almost directly to a well-structured Express application:

| Go Layer | Express Equivalent |
|----------|-------------------|
| `config/config.go` | `config/index.js` with `dotenv` |
| `database/database.go` | `db/index.js` with `knex` or `prisma` |
| `models/*.go` | `models/*.js` (Mongoose schemas, Sequelize models) |
| `repository/*.go` | `repositories/*.js` or `services/*.js` (data access) |
| `handlers/*.go` | `controllers/*.js` or `routes/*.js` |
| `middleware/*.go` | `middleware/*.js` |
| `router/router.go` | `routes/index.js` with `express.Router()` |
| `auth/jwt.go` | `utils/jwt.js` with `jsonwebtoken` |
| `main.go` | `app.js` / `server.js` |

The critical difference: in Express, you install 15-20 npm packages (`express`, `cors`, `helmet`, `morgan`, `jsonwebtoken`, `bcrypt`, `joi`, `pg`, `knex`, `dotenv`, `express-rate-limit`, ...). In Go, the standard library covers most of these needs, and you install only a handful of focused packages.

---

## 2. Project Structure

### Idiomatic Go Project Layout

```
taskapi/
├── main.go                          # Entry point: wiring, server start, graceful shutdown
├── go.mod                           # Module definition
├── go.sum                           # Dependency checksums
├── Dockerfile                       # Multi-stage Docker build
├── docker-compose.yml               # Local development with PostgreSQL
├── internal/                        # Private application code (cannot be imported by others)
│   ├── config/
│   │   └── config.go                # Configuration loading and validation
│   ├── database/
│   │   ├── database.go              # Connection pool setup, health check, close
│   │   └── migrations/
│   │       ├── 001_create_users.sql
│   │       ├── 002_create_projects.sql
│   │       ├── 003_create_tasks.sql
│   │       └── 004_create_comments.sql
│   ├── models/
│   │   ├── user.go                  # User domain model
│   │   ├── project.go               # Project domain model
│   │   ├── task.go                  # Task domain model
│   │   └── comment.go               # Comment domain model
│   ├── repository/
│   │   ├── user_repository.go       # User data access
│   │   ├── project_repository.go    # Project data access
│   │   ├── task_repository.go       # Task data access
│   │   └── comment_repository.go    # Comment data access
│   ├── service/
│   │   ├── user_service.go          # User business logic
│   │   ├── project_service.go       # Project business logic
│   │   ├── task_service.go          # Task business logic
│   │   └── dashboard_service.go     # Dashboard with parallel queries
│   ├── handlers/
│   │   ├── auth_handler.go          # Login, register, refresh
│   │   ├── project_handler.go       # Project CRUD endpoints
│   │   ├── task_handler.go          # Task CRUD endpoints
│   │   ├── comment_handler.go       # Comment endpoints
│   │   ├── dashboard_handler.go     # Dashboard aggregation endpoint
│   │   └── health_handler.go        # Health check
│   ├── middleware/
│   │   ├── auth.go                  # JWT authentication
│   │   ├── cors.go                  # Cross-Origin Resource Sharing
│   │   ├── logging.go               # Request/response logging
│   │   ├── ratelimit.go             # Token bucket rate limiting
│   │   └── recovery.go              # Panic recovery
│   ├── router/
│   │   └── router.go                # Route registration, method dispatch
│   ├── auth/
│   │   └── jwt.go                   # JWT token creation and validation
│   └── worker/
│       └── cleanup.go               # Background cleanup goroutines
├── pkg/                             # Public packages (reusable across projects)
│   ├── response/
│   │   └── response.go              # Standardized API response helpers
│   └── validator/
│       └── validator.go             # Input validation utilities
└── tests/
    └── integration/
        └── api_test.go              # Integration tests
```

### Why `internal/` and `pkg/`?

This is the standard Go project layout convention:

- **`internal/`** -- The Go compiler enforces that code inside `internal/` can only be imported by code in the parent directory tree. This prevents other projects from depending on your implementation details. Your handlers, repositories, and middleware are internal to this application.

- **`pkg/`** -- Code that is safe and intended to be imported by other projects. Our `response` and `validator` packages are generic utilities that any Go API could use.

- **`cmd/`** -- Some projects use `cmd/appname/main.go` when there are multiple binaries. Since we have a single binary, `main.go` at the root is simpler.

### Node.js/Express Comparison

A typical Express project:

```
task-api/
├── src/
│   ├── config/
│   │   └── index.js
│   ├── models/
│   │   └── User.js
│   ├── routes/
│   │   └── users.js
│   ├── middleware/
│   │   └── auth.js
│   ├── controllers/
│   │   └── userController.js
│   └── app.js
├── package.json
├── package-lock.json
└── node_modules/        ← hundreds of directories
```

The Go version has no `node_modules` equivalent. Dependencies are downloaded to `$GOPATH/pkg/mod` globally and shared across projects, and the compiled binary contains everything.

---

## 3. Configuration Layer

Configuration is the foundation of the application. Every other layer reads from it. We load from environment variables with sensible defaults, validate required fields at startup, and make the config immutable after initialization.

### `internal/config/config.go`

```go
package config

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

// Config holds all application configuration.
// Fields are exported for reading but the struct is created only via Load().
type Config struct {
	// Server
	ServerHost         string
	ServerPort         int
	ServerReadTimeout  time.Duration
	ServerWriteTimeout time.Duration
	ServerIdleTimeout  time.Duration

	// Database
	DatabaseHost     string
	DatabasePort     int
	DatabaseUser     string
	DatabasePassword string
	DatabaseName     string
	DatabaseSSLMode  string
	DatabaseMaxConns int
	DatabaseMinConns int

	// JWT
	JWTSecret          string
	JWTExpiration      time.Duration
	JWTRefreshExpiry   time.Duration

	// Rate Limiting
	RateLimitRequests int
	RateLimitWindow   time.Duration

	// CORS
	CORSAllowedOrigins []string

	// Logging
	LogLevel  string
	LogFormat string // "json" or "text"

	// Environment
	Environment string // "development", "staging", "production"
}

// Load reads configuration from environment variables with defaults.
func Load() (*Config, error) {
	cfg := &Config{
		// Server defaults
		ServerHost:         getEnv("SERVER_HOST", "0.0.0.0"),
		ServerPort:         getEnvInt("SERVER_PORT", 8080),
		ServerReadTimeout:  getEnvDuration("SERVER_READ_TIMEOUT", 10*time.Second),
		ServerWriteTimeout: getEnvDuration("SERVER_WRITE_TIMEOUT", 30*time.Second),
		ServerIdleTimeout:  getEnvDuration("SERVER_IDLE_TIMEOUT", 60*time.Second),

		// Database defaults
		DatabaseHost:     getEnv("DB_HOST", "localhost"),
		DatabasePort:     getEnvInt("DB_PORT", 5432),
		DatabaseUser:     getEnv("DB_USER", "taskapi"),
		DatabasePassword: getEnv("DB_PASSWORD", ""),
		DatabaseName:     getEnv("DB_NAME", "taskapi"),
		DatabaseSSLMode:  getEnv("DB_SSL_MODE", "disable"),
		DatabaseMaxConns: getEnvInt("DB_MAX_CONNS", 25),
		DatabaseMinConns: getEnvInt("DB_MIN_CONNS", 5),

		// JWT defaults
		JWTSecret:        getEnv("JWT_SECRET", ""),
		JWTExpiration:    getEnvDuration("JWT_EXPIRATION", 15*time.Minute),
		JWTRefreshExpiry: getEnvDuration("JWT_REFRESH_EXPIRY", 7*24*time.Hour),

		// Rate limiting defaults
		RateLimitRequests: getEnvInt("RATE_LIMIT_REQUESTS", 100),
		RateLimitWindow:   getEnvDuration("RATE_LIMIT_WINDOW", 1*time.Minute),

		// CORS defaults
		CORSAllowedOrigins: getEnvSlice("CORS_ALLOWED_ORIGINS", []string{"http://localhost:3000"}),

		// Logging defaults
		LogLevel:  getEnv("LOG_LEVEL", "info"),
		LogFormat: getEnv("LOG_FORMAT", "json"),

		// Environment
		Environment: getEnv("ENVIRONMENT", "development"),
	}

	if err := cfg.validate(); err != nil {
		return nil, fmt.Errorf("config validation: %w", err)
	}

	return cfg, nil
}

// validate ensures all required configuration is present and sane.
func (c *Config) validate() error {
	if c.JWTSecret == "" {
		return fmt.Errorf("JWT_SECRET is required")
	}
	if len(c.JWTSecret) < 32 {
		return fmt.Errorf("JWT_SECRET must be at least 32 characters")
	}
	if c.DatabasePassword == "" && c.Environment == "production" {
		return fmt.Errorf("DB_PASSWORD is required in production")
	}
	if c.ServerPort < 1 || c.ServerPort > 65535 {
		return fmt.Errorf("SERVER_PORT must be between 1 and 65535")
	}
	if c.DatabaseMaxConns < 1 {
		return fmt.Errorf("DB_MAX_CONNS must be at least 1")
	}
	validEnvs := map[string]bool{"development": true, "staging": true, "production": true}
	if !validEnvs[c.Environment] {
		return fmt.Errorf("ENVIRONMENT must be development, staging, or production")
	}
	return nil
}

// DatabaseDSN returns the PostgreSQL connection string.
func (c *Config) DatabaseDSN() string {
	return fmt.Sprintf(
		"host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
		c.DatabaseHost, c.DatabasePort, c.DatabaseUser,
		c.DatabasePassword, c.DatabaseName, c.DatabaseSSLMode,
	)
}

// ServerAddr returns the host:port string for the HTTP server.
func (c *Config) ServerAddr() string {
	return fmt.Sprintf("%s:%d", c.ServerHost, c.ServerPort)
}

// IsDevelopment returns true if running in development mode.
func (c *Config) IsDevelopment() bool {
	return c.Environment == "development"
}

// IsProduction returns true if running in production mode.
func (c *Config) IsProduction() bool {
	return c.Environment == "production"
}

// --- Helper functions ---

func getEnv(key, defaultVal string) string {
	if val, ok := os.LookupEnv(key); ok {
		return val
	}
	return defaultVal
}

func getEnvInt(key string, defaultVal int) int {
	val, ok := os.LookupEnv(key)
	if !ok {
		return defaultVal
	}
	intVal, err := strconv.Atoi(val)
	if err != nil {
		return defaultVal
	}
	return intVal
}

func getEnvDuration(key string, defaultVal time.Duration) time.Duration {
	val, ok := os.LookupEnv(key)
	if !ok {
		return defaultVal
	}
	dur, err := time.ParseDuration(val)
	if err != nil {
		return defaultVal
	}
	return dur
}

func getEnvSlice(key string, defaultVal []string) []string {
	val, ok := os.LookupEnv(key)
	if !ok {
		return defaultVal
	}
	// Simple comma-separated parsing
	var result []string
	start := 0
	for i := 0; i <= len(val); i++ {
		if i == len(val) || val[i] == ',' {
			item := val[start:i]
			// Trim spaces
			for len(item) > 0 && item[0] == ' ' {
				item = item[1:]
			}
			for len(item) > 0 && item[len(item)-1] == ' ' {
				item = item[:len(item)-1]
			}
			if item != "" {
				result = append(result, item)
			}
			start = i + 1
		}
	}
	if len(result) == 0 {
		return defaultVal
	}
	return result
}
```

### Key Design Decisions

**1. Fail fast on invalid config.** The `validate()` method runs at startup. If the JWT secret is missing or the port is out of range, the application refuses to start. This catches deployment mistakes immediately rather than surfacing them as runtime errors hours later.

**2. Immutable after creation.** `Load()` returns a `*Config` pointer, and no setter methods are provided. The config is effectively read-only after initialization. This prevents middleware or handlers from accidentally mutating global settings.

**3. Typed defaults.** Instead of passing all configuration as strings and parsing everywhere, each field is its proper Go type (`int`, `time.Duration`, `[]string`). The parsing happens once in the helper functions.

### Node.js/Express Comparison

In Express, you typically use `dotenv` plus manual parsing:

```javascript
// Node.js equivalent
require('dotenv').config();

const config = {
  port: parseInt(process.env.PORT) || 3000,
  jwtSecret: process.env.JWT_SECRET,  // Might be undefined!
  dbUrl: process.env.DATABASE_URL,
  // No type safety, no validation at startup
};

// You only discover missing config when a request hits that code path
```

The Go version validates everything upfront and ensures type safety at compile time. You will never have a "config is undefined" error in production.

---

## 4. Database Layer

The database layer manages PostgreSQL connections, runs migrations, and provides health checks. We use `database/sql` from the standard library with the `lib/pq` driver.

### `internal/database/database.go`

```go
package database

import (
	"context"
	"database/sql"
	"embed"
	"fmt"
	"log/slog"
	"sort"
	"strings"
	"time"

	_ "github.com/lib/pq" // PostgreSQL driver
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

// DB wraps *sql.DB and provides application-specific methods.
type DB struct {
	*sql.DB
	logger *slog.Logger
}

// New creates a new database connection pool with the given DSN.
func New(dsn string, maxConns, minConns int, logger *slog.Logger) (*DB, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("opening database: %w", err)
	}

	// Configure connection pool
	db.SetMaxOpenConns(maxConns)
	db.SetMaxIdleConns(minConns)
	db.SetConnMaxLifetime(30 * time.Minute)
	db.SetConnMaxIdleTime(5 * time.Minute)

	// Verify connection
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, fmt.Errorf("pinging database: %w", err)
	}

	logger.Info("database connection established",
		"max_open_conns", maxConns,
		"max_idle_conns", minConns,
	)

	return &DB{DB: db, logger: logger}, nil
}

// Health checks the database connection.
func (db *DB) Health(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	return db.PingContext(ctx)
}

// RunMigrations executes all embedded SQL migration files in order.
func (db *DB) RunMigrations(ctx context.Context) error {
	// Create migrations tracking table
	_, err := db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS schema_migrations (
			version    TEXT PRIMARY KEY,
			applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
		)
	`)
	if err != nil {
		return fmt.Errorf("creating migrations table: %w", err)
	}

	// Read embedded migration files
	entries, err := migrationsFS.ReadDir("migrations")
	if err != nil {
		return fmt.Errorf("reading migrations directory: %w", err)
	}

	// Sort by filename to ensure order
	sort.Slice(entries, func(i, j int) bool {
		return entries[i].Name() < entries[j].Name()
	})

	for _, entry := range entries {
		if entry.IsDir() || !strings.HasSuffix(entry.Name(), ".sql") {
			continue
		}

		version := entry.Name()

		// Check if already applied
		var exists bool
		err := db.QueryRowContext(ctx,
			"SELECT EXISTS(SELECT 1 FROM schema_migrations WHERE version = $1)",
			version,
		).Scan(&exists)
		if err != nil {
			return fmt.Errorf("checking migration %s: %w", version, err)
		}
		if exists {
			db.logger.Debug("migration already applied", "version", version)
			continue
		}

		// Read and execute migration
		content, err := migrationsFS.ReadFile("migrations/" + entry.Name())
		if err != nil {
			return fmt.Errorf("reading migration %s: %w", version, err)
		}

		tx, err := db.BeginTx(ctx, nil)
		if err != nil {
			return fmt.Errorf("beginning transaction for %s: %w", version, err)
		}

		if _, err := tx.ExecContext(ctx, string(content)); err != nil {
			tx.Rollback()
			return fmt.Errorf("executing migration %s: %w", version, err)
		}

		if _, err := tx.ExecContext(ctx,
			"INSERT INTO schema_migrations (version) VALUES ($1)", version,
		); err != nil {
			tx.Rollback()
			return fmt.Errorf("recording migration %s: %w", version, err)
		}

		if err := tx.Commit(); err != nil {
			return fmt.Errorf("committing migration %s: %w", version, err)
		}

		db.logger.Info("migration applied", "version", version)
	}

	return nil
}

// Close shuts down the connection pool gracefully.
func (db *DB) Close() error {
	db.logger.Info("closing database connection pool")
	return db.DB.Close()
}
```

### Migration Files

Each migration is a standalone SQL file. Migrations run in lexicographic order of filenames.

### `internal/database/migrations/001_create_users.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE IF NOT EXISTS users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    full_name     TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'member')),
    is_active     BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
CREATE INDEX IF NOT EXISTS idx_users_active ON users(is_active) WHERE is_active = true;
```

### `internal/database/migrations/002_create_projects.sql`

```sql
CREATE TABLE IF NOT EXISTS projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    description TEXT,
    owner_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_archived BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_projects_owner ON projects(owner_id);
CREATE INDEX IF NOT EXISTS idx_projects_archived ON projects(is_archived);

CREATE TABLE IF NOT EXISTS project_members (
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role       TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member')),
    joined_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (project_id, user_id)
);
```

### `internal/database/migrations/003_create_tasks.sql`

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id   UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title        TEXT NOT NULL,
    description  TEXT,
    status       TEXT NOT NULL DEFAULT 'todo' CHECK (status IN ('todo', 'in_progress', 'done')),
    priority     TEXT NOT NULL DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
    assignee_id  UUID REFERENCES users(id) ON DELETE SET NULL,
    creator_id   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    due_date     TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_tasks_project ON tasks(project_id);
CREATE INDEX IF NOT EXISTS idx_tasks_assignee ON tasks(assignee_id);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status);
CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority);
CREATE INDEX IF NOT EXISTS idx_tasks_due_date ON tasks(due_date) WHERE due_date IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_tasks_title_search ON tasks USING gin(to_tsvector('english', title));
```

### `internal/database/migrations/004_create_comments.sql`

```sql
CREATE TABLE IF NOT EXISTS comments (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id    UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    author_id  UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_comments_task ON comments(task_id);
CREATE INDEX IF NOT EXISTS idx_comments_author ON comments(author_id);
CREATE INDEX IF NOT EXISTS idx_comments_created ON comments(task_id, created_at DESC);
```

### Connection Pool Tuning

The connection pool settings deserve explanation:

| Setting | Value | Why |
|---------|-------|-----|
| `MaxOpenConns` | 25 | Limits total connections to PostgreSQL. A typical PG server handles ~100 connections. Leave room for other services and admin tools. |
| `MaxIdleConns` | 5 | Keeps a few warm connections ready. Too many idle connections waste server memory. |
| `ConnMaxLifetime` | 30 min | Recycles connections periodically. Prevents issues with long-lived connections (network equipment timeouts, PG config changes, connection-level memory leaks). |
| `ConnMaxIdleTime` | 5 min | Closes idle connections faster than the lifetime. Reduces resource usage during low-traffic periods. |

### Embedded Migrations with `go:embed`

We use Go's `embed` directive (from Go 1.16+) to bundle the SQL files into the compiled binary. This means the migration files travel with the binary -- no need to deploy them separately. This is a significant advantage over Node.js, where you must ensure migration files are deployed alongside the application.

---

## 5. Domain Models

Models define the shape of our data. They are plain Go structs with JSON tags for serialization and validation tags for input checking.

### `internal/models/user.go`

```go
package models

import "time"

// User represents a registered user in the system.
type User struct {
	ID           string    `json:"id"`
	Email        string    `json:"email" validate:"required,email"`
	PasswordHash string    `json:"-"` // Never serialize
	FullName     string    `json:"fullName" validate:"required,min=1,max=100"`
	Role         UserRole  `json:"role" validate:"required,oneof=admin member"`
	IsActive     bool      `json:"isActive"`
	CreatedAt    time.Time `json:"createdAt"`
	UpdatedAt    time.Time `json:"updatedAt"`
}

// UserRole represents the role of a user in the system.
type UserRole string

const (
	RoleAdmin  UserRole = "admin"
	RoleMember UserRole = "member"
)

// CreateUserRequest is the input for user registration.
type CreateUserRequest struct {
	Email    string `json:"email" validate:"required,email,max=255"`
	Password string `json:"password" validate:"required,min=8,max=128"`
	FullName string `json:"fullName" validate:"required,min=1,max=100"`
}

// UpdateUserRequest is the input for updating user profile.
type UpdateUserRequest struct {
	FullName *string `json:"fullName,omitempty" validate:"omitempty,min=1,max=100"`
	Email    *string `json:"email,omitempty" validate:"omitempty,email,max=255"`
}

// LoginRequest is the input for authentication.
type LoginRequest struct {
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required"`
}

// AuthResponse is returned after successful authentication.
type AuthResponse struct {
	AccessToken  string `json:"accessToken"`
	RefreshToken string `json:"refreshToken"`
	ExpiresIn    int    `json:"expiresIn"` // seconds
	User         *User  `json:"user"`
}

// RefreshRequest is the input for token refresh.
type RefreshRequest struct {
	RefreshToken string `json:"refreshToken" validate:"required"`
}
```

### `internal/models/project.go`

```go
package models

import "time"

// Project represents a container for tasks.
type Project struct {
	ID          string    `json:"id"`
	Name        string    `json:"name" validate:"required,min=1,max=200"`
	Description *string   `json:"description,omitempty" validate:"omitempty,max=2000"`
	OwnerID     string    `json:"ownerId"`
	IsArchived  bool      `json:"isArchived"`
	CreatedAt   time.Time `json:"createdAt"`
	UpdatedAt   time.Time `json:"updatedAt"`
}

// CreateProjectRequest is the input for creating a project.
type CreateProjectRequest struct {
	Name        string  `json:"name" validate:"required,min=1,max=200"`
	Description *string `json:"description,omitempty" validate:"omitempty,max=2000"`
}

// UpdateProjectRequest is the input for updating a project.
type UpdateProjectRequest struct {
	Name        *string `json:"name,omitempty" validate:"omitempty,min=1,max=200"`
	Description *string `json:"description,omitempty" validate:"omitempty,max=2000"`
	IsArchived  *bool   `json:"isArchived,omitempty"`
}

// ProjectMember represents a user's membership in a project.
type ProjectMember struct {
	ProjectID string    `json:"projectId"`
	UserID    string    `json:"userId"`
	Role      string    `json:"role"`
	JoinedAt  time.Time `json:"joinedAt"`
}
```

### `internal/models/task.go`

```go
package models

import "time"

// Task represents a work item within a project.
type Task struct {
	ID          string     `json:"id"`
	ProjectID   string     `json:"projectId" validate:"required,uuid"`
	Title       string     `json:"title" validate:"required,min=1,max=200"`
	Description *string    `json:"description,omitempty" validate:"omitempty,max=5000"`
	Status      TaskStatus `json:"status" validate:"required,oneof=todo in_progress done"`
	Priority    Priority   `json:"priority" validate:"required,oneof=low medium high urgent"`
	AssigneeID  *string    `json:"assigneeId,omitempty" validate:"omitempty,uuid"`
	CreatorID   string     `json:"creatorId"`
	DueDate     *time.Time `json:"dueDate,omitempty"`
	CompletedAt *time.Time `json:"completedAt,omitempty"`
	CreatedAt   time.Time  `json:"createdAt"`
	UpdatedAt   time.Time  `json:"updatedAt"`
}

// TaskStatus represents the current state of a task.
type TaskStatus string

const (
	StatusTodo       TaskStatus = "todo"
	StatusInProgress TaskStatus = "in_progress"
	StatusDone       TaskStatus = "done"
)

// Priority represents the urgency of a task.
type Priority string

const (
	PriorityLow    Priority = "low"
	PriorityMedium Priority = "medium"
	PriorityHigh   Priority = "high"
	PriorityUrgent Priority = "urgent"
)

// CreateTaskRequest is the input for creating a task.
type CreateTaskRequest struct {
	Title       string     `json:"title" validate:"required,min=1,max=200"`
	Description *string    `json:"description,omitempty" validate:"omitempty,max=5000"`
	Status      TaskStatus `json:"status" validate:"omitempty,oneof=todo in_progress done"`
	Priority    Priority   `json:"priority" validate:"omitempty,oneof=low medium high urgent"`
	AssigneeID  *string    `json:"assigneeId,omitempty" validate:"omitempty,uuid"`
	DueDate     *time.Time `json:"dueDate,omitempty"`
}

// UpdateTaskRequest is the input for updating a task.
type UpdateTaskRequest struct {
	Title       *string    `json:"title,omitempty" validate:"omitempty,min=1,max=200"`
	Description *string    `json:"description,omitempty" validate:"omitempty,max=5000"`
	Status      *TaskStatus `json:"status,omitempty" validate:"omitempty,oneof=todo in_progress done"`
	Priority    *Priority  `json:"priority,omitempty" validate:"omitempty,oneof=low medium high urgent"`
	AssigneeID  *string    `json:"assigneeId,omitempty" validate:"omitempty,uuid"`
	DueDate     *time.Time `json:"dueDate,omitempty"`
}

// TaskFilter holds query parameters for listing tasks.
type TaskFilter struct {
	Status     *TaskStatus `json:"status,omitempty"`
	Priority   *Priority   `json:"priority,omitempty"`
	AssigneeID *string     `json:"assigneeId,omitempty"`
	Search     *string     `json:"search,omitempty"`
}
```

### `internal/models/comment.go`

```go
package models

import "time"

// Comment represents a discussion entry on a task.
type Comment struct {
	ID        string    `json:"id"`
	TaskID    string    `json:"taskId"`
	AuthorID  string    `json:"authorId"`
	Body      string    `json:"body" validate:"required,min=1,max=5000"`
	CreatedAt time.Time `json:"createdAt"`
	UpdatedAt time.Time `json:"updatedAt"`
}

// CreateCommentRequest is the input for adding a comment to a task.
type CreateCommentRequest struct {
	Body string `json:"body" validate:"required,min=1,max=5000"`
}

// UpdateCommentRequest is the input for editing a comment.
type UpdateCommentRequest struct {
	Body string `json:"body" validate:"required,min=1,max=5000"`
}
```

### Common Types: Pagination

```go
package models

// PaginationParams holds pagination query parameters.
type PaginationParams struct {
	Page    int `json:"page" validate:"min=1"`
	PerPage int `json:"perPage" validate:"min=1,max=100"`
}

// DefaultPagination returns pagination with sensible defaults.
func DefaultPagination() PaginationParams {
	return PaginationParams{
		Page:    1,
		PerPage: 20,
	}
}

// Offset calculates the SQL OFFSET from page and perPage.
func (p PaginationParams) Offset() int {
	return (p.Page - 1) * p.PerPage
}

// Limit returns the SQL LIMIT value.
func (p PaginationParams) Limit() int {
	return p.PerPage
}

// PaginatedResponse wraps a list response with pagination metadata.
type PaginatedResponse[T any] struct {
	Data       []T            `json:"data"`
	Pagination PaginationMeta `json:"pagination"`
}

// PaginationMeta provides pagination info in API responses.
type PaginationMeta struct {
	Page       int `json:"page"`
	PerPage    int `json:"perPage"`
	Total      int `json:"total"`
	TotalPages int `json:"totalPages"`
}

// NewPaginationMeta creates pagination metadata from counts.
func NewPaginationMeta(page, perPage, total int) PaginationMeta {
	totalPages := total / perPage
	if total%perPage > 0 {
		totalPages++
	}
	return PaginationMeta{
		Page:       page,
		PerPage:    perPage,
		Total:      total,
		TotalPages: totalPages,
	}
}
```

### Design Decisions in the Models

**1. Pointer fields for optional values.** Fields like `Description *string` and `AssigneeID *string` use pointers so they can be `nil`. This maps cleanly to SQL `NULL` and JSON `null`. Without pointers, you cannot distinguish between "user did not send this field" and "user sent an empty string."

**2. Separate request and model structs.** `CreateTaskRequest` is different from `Task`. The request struct contains only the fields the client can set. The model struct includes server-generated fields like `ID`, `CreatedAt`, and `CreatorID`. This prevents clients from setting their own IDs or timestamps.

**3. String-based enums with const blocks.** Go does not have built-in enums like TypeScript. The `type TaskStatus string` + `const` pattern provides type safety while keeping JSON serialization straightforward. The `validate:"oneof=..."` tag enforces valid values.

**4. Generic PaginatedResponse.** Using Go generics (from Chapter 15), `PaginatedResponse[T any]` works with any model type -- tasks, projects, comments -- without code duplication.

### Node.js/Express Comparison

In Express with TypeScript, you might define similar models:

```typescript
// TypeScript equivalent
interface Task {
  id: string;
  projectId: string;
  title: string;
  description?: string;   // Optional via ?
  status: 'todo' | 'in_progress' | 'done';  // Union types
  priority: 'low' | 'medium' | 'high' | 'urgent';
  assigneeId?: string;
  dueDate?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

TypeScript's `?` syntax and union types are arguably more ergonomic for optional fields and enums. But Go's approach with pointers and const blocks is explicit and leaves no ambiguity about nil handling.

---

## 6. Repository Layer

Repositories encapsulate all database access. Each repository is defined as an **interface** (for testing with mocks) and an **implementation** that uses `*sql.DB`.

### `internal/repository/user_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"taskapi/internal/models"
)

// UserRepository defines the data access interface for users.
type UserRepository interface {
	Create(ctx context.Context, user *models.User) error
	GetByID(ctx context.Context, id string) (*models.User, error)
	GetByEmail(ctx context.Context, email string) (*models.User, error)
	Update(ctx context.Context, user *models.User) error
	Delete(ctx context.Context, id string) error
	List(ctx context.Context, params models.PaginationParams) ([]models.User, int, error)
	Count(ctx context.Context) (int, error)
}

// userRepo is the PostgreSQL implementation of UserRepository.
type userRepo struct {
	db *sql.DB
}

// NewUserRepository creates a new PostgreSQL-backed user repository.
func NewUserRepository(db *sql.DB) UserRepository {
	return &userRepo{db: db}
}

func (r *userRepo) Create(ctx context.Context, user *models.User) error {
	query := `
		INSERT INTO users (email, password_hash, full_name, role)
		VALUES ($1, $2, $3, $4)
		RETURNING id, is_active, created_at, updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		user.Email, user.PasswordHash, user.FullName, user.Role,
	).Scan(&user.ID, &user.IsActive, &user.CreatedAt, &user.UpdatedAt)
}

func (r *userRepo) GetByID(ctx context.Context, id string) (*models.User, error) {
	query := `
		SELECT id, email, password_hash, full_name, role, is_active, created_at, updated_at
		FROM users
		WHERE id = $1
	`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&user.ID, &user.Email, &user.PasswordHash, &user.FullName,
		&user.Role, &user.IsActive, &user.CreatedAt, &user.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil // Not found
	}
	if err != nil {
		return nil, fmt.Errorf("getting user by id: %w", err)
	}
	return user, nil
}

func (r *userRepo) GetByEmail(ctx context.Context, email string) (*models.User, error) {
	query := `
		SELECT id, email, password_hash, full_name, role, is_active, created_at, updated_at
		FROM users
		WHERE email = $1
	`
	user := &models.User{}
	err := r.db.QueryRowContext(ctx, query, email).Scan(
		&user.ID, &user.Email, &user.PasswordHash, &user.FullName,
		&user.Role, &user.IsActive, &user.CreatedAt, &user.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting user by email: %w", err)
	}
	return user, nil
}

func (r *userRepo) Update(ctx context.Context, user *models.User) error {
	query := `
		UPDATE users
		SET email = $1, full_name = $2, role = $3, is_active = $4, updated_at = NOW()
		WHERE id = $5
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		user.Email, user.FullName, user.Role, user.IsActive, user.ID,
	).Scan(&user.UpdatedAt)
}

func (r *userRepo) Delete(ctx context.Context, id string) error {
	query := `DELETE FROM users WHERE id = $1`
	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return fmt.Errorf("deleting user: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("checking affected rows: %w", err)
	}
	if rows == 0 {
		return fmt.Errorf("user not found")
	}
	return nil
}

func (r *userRepo) List(ctx context.Context, params models.PaginationParams) ([]models.User, int, error) {
	// Count total
	var total int
	countQuery := `SELECT COUNT(*) FROM users WHERE is_active = true`
	if err := r.db.QueryRowContext(ctx, countQuery).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("counting users: %w", err)
	}

	// Fetch page
	query := `
		SELECT id, email, password_hash, full_name, role, is_active, created_at, updated_at
		FROM users
		WHERE is_active = true
		ORDER BY created_at DESC
		LIMIT $1 OFFSET $2
	`
	rows, err := r.db.QueryContext(ctx, query, params.Limit(), params.Offset())
	if err != nil {
		return nil, 0, fmt.Errorf("listing users: %w", err)
	}
	defer rows.Close()

	var users []models.User
	for rows.Next() {
		var u models.User
		if err := rows.Scan(
			&u.ID, &u.Email, &u.PasswordHash, &u.FullName,
			&u.Role, &u.IsActive, &u.CreatedAt, &u.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning user: %w", err)
		}
		users = append(users, u)
	}
	if err := rows.Err(); err != nil {
		return nil, 0, fmt.Errorf("iterating users: %w", err)
	}

	return users, total, nil
}

func (r *userRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users").Scan(&count)
	return count, err
}

// scanUser is not used externally; keeping row scanning DRY.
func scanUser(row interface{ Scan(dest ...any) error }) (*models.User, error) {
	user := &models.User{}
	err := row.Scan(
		&user.ID, &user.Email, &user.PasswordHash, &user.FullName,
		&user.Role, &user.IsActive, &user.CreatedAt, &user.UpdatedAt,
	)
	if err != nil {
		return nil, err
	}
	return user, nil
}

// ensure userRepo implements UserRepository at compile time
var _ UserRepository = (*userRepo)(nil)
```

### `internal/repository/project_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"taskapi/internal/models"
)

// ProjectRepository defines the data access interface for projects.
type ProjectRepository interface {
	Create(ctx context.Context, project *models.Project) error
	GetByID(ctx context.Context, id string) (*models.Project, error)
	Update(ctx context.Context, project *models.Project) error
	Delete(ctx context.Context, id string) error
	ListByUser(ctx context.Context, userID string, params models.PaginationParams) ([]models.Project, int, error)
	AddMember(ctx context.Context, projectID, userID, role string) error
	IsMember(ctx context.Context, projectID, userID string) (bool, error)
	Count(ctx context.Context) (int, error)
}

type projectRepo struct {
	db *sql.DB
}

func NewProjectRepository(db *sql.DB) ProjectRepository {
	return &projectRepo{db: db}
}

func (r *projectRepo) Create(ctx context.Context, project *models.Project) error {
	tx, err := r.db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("beginning transaction: %w", err)
	}
	defer tx.Rollback() // No-op if committed

	// Create the project
	query := `
		INSERT INTO projects (name, description, owner_id)
		VALUES ($1, $2, $3)
		RETURNING id, is_archived, created_at, updated_at
	`
	err = tx.QueryRowContext(ctx, query,
		project.Name, project.Description, project.OwnerID,
	).Scan(&project.ID, &project.IsArchived, &project.CreatedAt, &project.UpdatedAt)
	if err != nil {
		return fmt.Errorf("inserting project: %w", err)
	}

	// Add the creator as an owner member
	memberQuery := `
		INSERT INTO project_members (project_id, user_id, role)
		VALUES ($1, $2, 'owner')
	`
	if _, err := tx.ExecContext(ctx, memberQuery, project.ID, project.OwnerID); err != nil {
		return fmt.Errorf("adding owner as member: %w", err)
	}

	return tx.Commit()
}

func (r *projectRepo) GetByID(ctx context.Context, id string) (*models.Project, error) {
	query := `
		SELECT id, name, description, owner_id, is_archived, created_at, updated_at
		FROM projects
		WHERE id = $1
	`
	project := &models.Project{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&project.ID, &project.Name, &project.Description, &project.OwnerID,
		&project.IsArchived, &project.CreatedAt, &project.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting project: %w", err)
	}
	return project, nil
}

func (r *projectRepo) Update(ctx context.Context, project *models.Project) error {
	query := `
		UPDATE projects
		SET name = $1, description = $2, is_archived = $3, updated_at = NOW()
		WHERE id = $4
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		project.Name, project.Description, project.IsArchived, project.ID,
	).Scan(&project.UpdatedAt)
}

func (r *projectRepo) Delete(ctx context.Context, id string) error {
	result, err := r.db.ExecContext(ctx, "DELETE FROM projects WHERE id = $1", id)
	if err != nil {
		return fmt.Errorf("deleting project: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("project not found")
	}
	return nil
}

func (r *projectRepo) ListByUser(ctx context.Context, userID string, params models.PaginationParams) ([]models.Project, int, error) {
	// Count total projects the user is a member of
	var total int
	countQuery := `
		SELECT COUNT(*)
		FROM projects p
		INNER JOIN project_members pm ON p.id = pm.project_id
		WHERE pm.user_id = $1 AND p.is_archived = false
	`
	if err := r.db.QueryRowContext(ctx, countQuery, userID).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("counting projects: %w", err)
	}

	// Fetch page
	query := `
		SELECT p.id, p.name, p.description, p.owner_id, p.is_archived, p.created_at, p.updated_at
		FROM projects p
		INNER JOIN project_members pm ON p.id = pm.project_id
		WHERE pm.user_id = $1 AND p.is_archived = false
		ORDER BY p.created_at DESC
		LIMIT $2 OFFSET $3
	`
	rows, err := r.db.QueryContext(ctx, query, userID, params.Limit(), params.Offset())
	if err != nil {
		return nil, 0, fmt.Errorf("listing projects: %w", err)
	}
	defer rows.Close()

	var projects []models.Project
	for rows.Next() {
		var p models.Project
		if err := rows.Scan(
			&p.ID, &p.Name, &p.Description, &p.OwnerID,
			&p.IsArchived, &p.CreatedAt, &p.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning project: %w", err)
		}
		projects = append(projects, p)
	}
	return projects, total, rows.Err()
}

func (r *projectRepo) AddMember(ctx context.Context, projectID, userID, role string) error {
	query := `
		INSERT INTO project_members (project_id, user_id, role)
		VALUES ($1, $2, $3)
		ON CONFLICT (project_id, user_id) DO UPDATE SET role = $3
	`
	_, err := r.db.ExecContext(ctx, query, projectID, userID, role)
	return err
}

func (r *projectRepo) IsMember(ctx context.Context, projectID, userID string) (bool, error) {
	var exists bool
	query := `
		SELECT EXISTS(
			SELECT 1 FROM project_members
			WHERE project_id = $1 AND user_id = $2
		)
	`
	err := r.db.QueryRowContext(ctx, query, projectID, userID).Scan(&exists)
	return exists, err
}

func (r *projectRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM projects").Scan(&count)
	return count, err
}

var _ ProjectRepository = (*projectRepo)(nil)
```

### `internal/repository/task_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"strings"
	"time"

	"taskapi/internal/models"
)

// TaskRepository defines the data access interface for tasks.
type TaskRepository interface {
	Create(ctx context.Context, task *models.Task) error
	GetByID(ctx context.Context, id string) (*models.Task, error)
	Update(ctx context.Context, task *models.Task) error
	Delete(ctx context.Context, id string) error
	ListByProject(ctx context.Context, projectID string, filter models.TaskFilter, params models.PaginationParams) ([]models.Task, int, error)
	CountByStatus(ctx context.Context, projectID string) (map[models.TaskStatus]int, error)
	CountOverdue(ctx context.Context) (int, error)
	Count(ctx context.Context) (int, error)
	MarkOverdueTasks(ctx context.Context) (int64, error)
}

type taskRepo struct {
	db *sql.DB
}

func NewTaskRepository(db *sql.DB) TaskRepository {
	return &taskRepo{db: db}
}

func (r *taskRepo) Create(ctx context.Context, task *models.Task) error {
	// Default status and priority if not set
	if task.Status == "" {
		task.Status = models.StatusTodo
	}
	if task.Priority == "" {
		task.Priority = models.PriorityMedium
	}

	query := `
		INSERT INTO tasks (project_id, title, description, status, priority, assignee_id, creator_id, due_date)
		VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
		RETURNING id, completed_at, created_at, updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		task.ProjectID, task.Title, task.Description, task.Status,
		task.Priority, task.AssigneeID, task.CreatorID, task.DueDate,
	).Scan(&task.ID, &task.CompletedAt, &task.CreatedAt, &task.UpdatedAt)
}

func (r *taskRepo) GetByID(ctx context.Context, id string) (*models.Task, error) {
	query := `
		SELECT id, project_id, title, description, status, priority,
		       assignee_id, creator_id, due_date, completed_at, created_at, updated_at
		FROM tasks
		WHERE id = $1
	`
	task := &models.Task{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&task.ID, &task.ProjectID, &task.Title, &task.Description,
		&task.Status, &task.Priority, &task.AssigneeID, &task.CreatorID,
		&task.DueDate, &task.CompletedAt, &task.CreatedAt, &task.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting task: %w", err)
	}
	return task, nil
}

func (r *taskRepo) Update(ctx context.Context, task *models.Task) error {
	// If status changed to done, set completed_at
	var completedAt *time.Time
	if task.Status == models.StatusDone {
		now := time.Now()
		completedAt = &now
	}

	query := `
		UPDATE tasks
		SET title = $1, description = $2, status = $3, priority = $4,
		    assignee_id = $5, due_date = $6, completed_at = $7, updated_at = NOW()
		WHERE id = $8
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		task.Title, task.Description, task.Status, task.Priority,
		task.AssigneeID, task.DueDate, completedAt, task.ID,
	).Scan(&task.UpdatedAt)
}

func (r *taskRepo) Delete(ctx context.Context, id string) error {
	result, err := r.db.ExecContext(ctx, "DELETE FROM tasks WHERE id = $1", id)
	if err != nil {
		return fmt.Errorf("deleting task: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("task not found")
	}
	return nil
}

func (r *taskRepo) ListByProject(ctx context.Context, projectID string, filter models.TaskFilter, params models.PaginationParams) ([]models.Task, int, error) {
	// Build dynamic WHERE clause
	conditions := []string{"project_id = $1"}
	args := []any{projectID}
	argIdx := 2

	if filter.Status != nil {
		conditions = append(conditions, fmt.Sprintf("status = $%d", argIdx))
		args = append(args, *filter.Status)
		argIdx++
	}
	if filter.Priority != nil {
		conditions = append(conditions, fmt.Sprintf("priority = $%d", argIdx))
		args = append(args, *filter.Priority)
		argIdx++
	}
	if filter.AssigneeID != nil {
		conditions = append(conditions, fmt.Sprintf("assignee_id = $%d", argIdx))
		args = append(args, *filter.AssigneeID)
		argIdx++
	}
	if filter.Search != nil && *filter.Search != "" {
		conditions = append(conditions, fmt.Sprintf(
			"to_tsvector('english', title) @@ plainto_tsquery('english', $%d)", argIdx,
		))
		args = append(args, *filter.Search)
		argIdx++
	}

	where := strings.Join(conditions, " AND ")

	// Count total matching
	var total int
	countQuery := fmt.Sprintf("SELECT COUNT(*) FROM tasks WHERE %s", where)
	if err := r.db.QueryRowContext(ctx, countQuery, args...).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("counting tasks: %w", err)
	}

	// Fetch page
	query := fmt.Sprintf(`
		SELECT id, project_id, title, description, status, priority,
		       assignee_id, creator_id, due_date, completed_at, created_at, updated_at
		FROM tasks
		WHERE %s
		ORDER BY
			CASE priority
				WHEN 'urgent' THEN 1
				WHEN 'high' THEN 2
				WHEN 'medium' THEN 3
				WHEN 'low' THEN 4
			END,
			created_at DESC
		LIMIT $%d OFFSET $%d
	`, where, argIdx, argIdx+1)

	args = append(args, params.Limit(), params.Offset())

	rows, err := r.db.QueryContext(ctx, query, args...)
	if err != nil {
		return nil, 0, fmt.Errorf("listing tasks: %w", err)
	}
	defer rows.Close()

	var tasks []models.Task
	for rows.Next() {
		var t models.Task
		if err := rows.Scan(
			&t.ID, &t.ProjectID, &t.Title, &t.Description,
			&t.Status, &t.Priority, &t.AssigneeID, &t.CreatorID,
			&t.DueDate, &t.CompletedAt, &t.CreatedAt, &t.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning task: %w", err)
		}
		tasks = append(tasks, t)
	}

	return tasks, total, rows.Err()
}

func (r *taskRepo) CountByStatus(ctx context.Context, projectID string) (map[models.TaskStatus]int, error) {
	query := `
		SELECT status, COUNT(*)
		FROM tasks
		WHERE project_id = $1
		GROUP BY status
	`
	rows, err := r.db.QueryContext(ctx, query, projectID)
	if err != nil {
		return nil, fmt.Errorf("counting by status: %w", err)
	}
	defer rows.Close()

	counts := make(map[models.TaskStatus]int)
	for rows.Next() {
		var status models.TaskStatus
		var count int
		if err := rows.Scan(&status, &count); err != nil {
			return nil, err
		}
		counts[status] = count
	}
	return counts, rows.Err()
}

func (r *taskRepo) CountOverdue(ctx context.Context) (int, error) {
	var count int
	query := `
		SELECT COUNT(*)
		FROM tasks
		WHERE due_date < NOW()
		  AND status != 'done'
		  AND due_date IS NOT NULL
	`
	err := r.db.QueryRowContext(ctx, query).Scan(&count)
	return count, err
}

func (r *taskRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM tasks").Scan(&count)
	return count, err
}

func (r *taskRepo) MarkOverdueTasks(ctx context.Context) (int64, error) {
	// This is used by the background cleanup worker
	query := `
		UPDATE tasks
		SET updated_at = NOW()
		WHERE due_date < NOW()
		  AND status != 'done'
		  AND due_date IS NOT NULL
	`
	result, err := r.db.ExecContext(ctx, query)
	if err != nil {
		return 0, err
	}
	return result.RowsAffected()
}

var _ TaskRepository = (*taskRepo)(nil)
```

### `internal/repository/comment_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"taskapi/internal/models"
)

// CommentRepository defines the data access interface for comments.
type CommentRepository interface {
	Create(ctx context.Context, comment *models.Comment) error
	GetByID(ctx context.Context, id string) (*models.Comment, error)
	Update(ctx context.Context, comment *models.Comment) error
	Delete(ctx context.Context, id string) error
	ListByTask(ctx context.Context, taskID string, params models.PaginationParams) ([]models.Comment, int, error)
}

type commentRepo struct {
	db *sql.DB
}

func NewCommentRepository(db *sql.DB) CommentRepository {
	return &commentRepo{db: db}
}

func (r *commentRepo) Create(ctx context.Context, comment *models.Comment) error {
	query := `
		INSERT INTO comments (task_id, author_id, body)
		VALUES ($1, $2, $3)
		RETURNING id, created_at, updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		comment.TaskID, comment.AuthorID, comment.Body,
	).Scan(&comment.ID, &comment.CreatedAt, &comment.UpdatedAt)
}

func (r *commentRepo) GetByID(ctx context.Context, id string) (*models.Comment, error) {
	query := `
		SELECT id, task_id, author_id, body, created_at, updated_at
		FROM comments
		WHERE id = $1
	`
	c := &models.Comment{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&c.ID, &c.TaskID, &c.AuthorID, &c.Body, &c.CreatedAt, &c.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting comment: %w", err)
	}
	return c, nil
}

func (r *commentRepo) Update(ctx context.Context, comment *models.Comment) error {
	query := `
		UPDATE comments
		SET body = $1, updated_at = NOW()
		WHERE id = $2
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query, comment.Body, comment.ID).Scan(&comment.UpdatedAt)
}

func (r *commentRepo) Delete(ctx context.Context, id string) error {
	result, err := r.db.ExecContext(ctx, "DELETE FROM comments WHERE id = $1", id)
	if err != nil {
		return fmt.Errorf("deleting comment: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return fmt.Errorf("comment not found")
	}
	return nil
}

func (r *commentRepo) ListByTask(ctx context.Context, taskID string, params models.PaginationParams) ([]models.Comment, int, error) {
	var total int
	countQuery := `SELECT COUNT(*) FROM comments WHERE task_id = $1`
	if err := r.db.QueryRowContext(ctx, countQuery, taskID).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("counting comments: %w", err)
	}

	query := `
		SELECT id, task_id, author_id, body, created_at, updated_at
		FROM comments
		WHERE task_id = $1
		ORDER BY created_at ASC
		LIMIT $2 OFFSET $3
	`
	rows, err := r.db.QueryContext(ctx, query, taskID, params.Limit(), params.Offset())
	if err != nil {
		return nil, 0, fmt.Errorf("listing comments: %w", err)
	}
	defer rows.Close()

	var comments []models.Comment
	for rows.Next() {
		var c models.Comment
		if err := rows.Scan(
			&c.ID, &c.TaskID, &c.AuthorID, &c.Body, &c.CreatedAt, &c.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning comment: %w", err)
		}
		comments = append(comments, c)
	}
	return comments, total, rows.Err()
}

var _ CommentRepository = (*commentRepo)(nil)
```

### The Interface Pattern

Every repository is an interface. This is the single most important pattern in this entire codebase. Here is why:

**1. Testability.** Handlers and services depend on the interface, not the PostgreSQL implementation. In tests, you inject a mock that returns canned data. No database required.

**2. Swappability.** Want to switch from PostgreSQL to MySQL? Implement the interface with a new struct. The rest of the application does not change.

**3. Compile-time verification.** The `var _ UserRepository = (*userRepo)(nil)` line ensures that `userRepo` implements every method in `UserRepository`. If you add a method to the interface but forget the implementation, the compiler catches it immediately.

### Node.js/Express Comparison

In Express, the repository pattern is optional and often skipped:

```javascript
// Express: controllers often query the database directly
app.get('/api/users/:id', async (req, res) => {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
  res.json(user.rows[0]);
});
```

This works but makes testing impossible without a real database. The Go pattern of interface-based repositories forces separation that pays dividends in testing and maintenance.

---

## 7. Authentication

Authentication is handled with JWT (JSON Web Tokens). We use `bcrypt` for password hashing and issue access/refresh token pairs.

### `internal/auth/jwt.go`

```go
package auth

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"strings"
	"time"
)

// Claims represents the JWT payload.
type Claims struct {
	UserID string `json:"sub"`
	Email  string `json:"email"`
	Role   string `json:"role"`
	Type   string `json:"type"` // "access" or "refresh"
	IssuedAt  int64 `json:"iat"`
	ExpiresAt int64 `json:"exp"`
}

// IsExpired checks if the token has expired.
func (c *Claims) IsExpired() bool {
	return time.Now().Unix() > c.ExpiresAt
}

// JWTService handles JWT creation and validation.
type JWTService struct {
	secret         []byte
	accessExpiry   time.Duration
	refreshExpiry  time.Duration
}

// NewJWTService creates a new JWT service.
func NewJWTService(secret string, accessExpiry, refreshExpiry time.Duration) *JWTService {
	return &JWTService{
		secret:        []byte(secret),
		accessExpiry:  accessExpiry,
		refreshExpiry: refreshExpiry,
	}
}

// GenerateAccessToken creates a new access token for the given user.
func (s *JWTService) GenerateAccessToken(userID, email, role string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID:    userID,
		Email:     email,
		Role:      role,
		Type:      "access",
		IssuedAt:  now.Unix(),
		ExpiresAt: now.Add(s.accessExpiry).Unix(),
	}
	return s.encode(claims)
}

// GenerateRefreshToken creates a new refresh token for the given user.
func (s *JWTService) GenerateRefreshToken(userID, email, role string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID:    userID,
		Email:     email,
		Role:      role,
		Type:      "refresh",
		IssuedAt:  now.Unix(),
		ExpiresAt: now.Add(s.refreshExpiry).Unix(),
	}
	return s.encode(claims)
}

// ValidateToken parses and validates a JWT string.
func (s *JWTService) ValidateToken(tokenStr string) (*Claims, error) {
	parts := strings.Split(tokenStr, ".")
	if len(parts) != 3 {
		return nil, fmt.Errorf("invalid token format")
	}

	// Verify signature
	signingInput := parts[0] + "." + parts[1]
	expectedSig := s.sign([]byte(signingInput))
	actualSig, err := base64.RawURLEncoding.DecodeString(parts[2])
	if err != nil {
		return nil, fmt.Errorf("invalid signature encoding")
	}
	if !hmac.Equal(expectedSig, actualSig) {
		return nil, fmt.Errorf("invalid signature")
	}

	// Decode claims
	claimsJSON, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		return nil, fmt.Errorf("invalid claims encoding")
	}

	var claims Claims
	if err := json.Unmarshal(claimsJSON, &claims); err != nil {
		return nil, fmt.Errorf("invalid claims: %w", err)
	}

	if claims.IsExpired() {
		return nil, fmt.Errorf("token expired")
	}

	return &claims, nil
}

// AccessExpiry returns the access token expiration duration (for API responses).
func (s *JWTService) AccessExpiry() time.Duration {
	return s.accessExpiry
}

// encode creates a signed JWT string from claims.
func (s *JWTService) encode(claims Claims) (string, error) {
	// Header
	header := map[string]string{"alg": "HS256", "typ": "JWT"}
	headerJSON, err := json.Marshal(header)
	if err != nil {
		return "", fmt.Errorf("encoding header: %w", err)
	}
	headerB64 := base64.RawURLEncoding.EncodeToString(headerJSON)

	// Payload
	claimsJSON, err := json.Marshal(claims)
	if err != nil {
		return "", fmt.Errorf("encoding claims: %w", err)
	}
	claimsB64 := base64.RawURLEncoding.EncodeToString(claimsJSON)

	// Signature
	signingInput := headerB64 + "." + claimsB64
	signature := s.sign([]byte(signingInput))
	signatureB64 := base64.RawURLEncoding.EncodeToString(signature)

	return signingInput + "." + signatureB64, nil
}

// sign creates an HMAC-SHA256 signature.
func (s *JWTService) sign(data []byte) []byte {
	mac := hmac.New(sha256.New, s.secret)
	mac.Write(data)
	return mac.Sum(nil)
}
```

### Password Hashing Utility

We also need bcrypt for password hashing. This is one of the few places we use a third-party package:

```go
package auth

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

// HashPassword generates a bcrypt hash of the password.
func HashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hashing password: %w", err)
	}
	return string(hash), nil
}

// CheckPassword compares a plaintext password against a bcrypt hash.
func CheckPassword(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
	return err == nil
}
```

### Why Build JWT from Scratch?

In a production codebase, you would typically use a well-tested library like `github.com/golang-jwt/jwt/v5`. We implement it here from scratch for educational purposes -- to demystify what JWT actually is:

1. **Header** -- JSON with algorithm and type, base64url-encoded
2. **Payload** -- JSON with claims (user ID, expiry, etc.), base64url-encoded
3. **Signature** -- HMAC-SHA256 of `header.payload` using a secret key

That is the entire JWT specification for HMAC signing. It is three base64url strings separated by dots. No magic.

### Node.js/Express Comparison

The Express equivalent uses `jsonwebtoken`:

```javascript
// Express
const jwt = require('jsonwebtoken');

// Create token
const token = jwt.sign({ userId: user.id, role: user.role }, process.env.JWT_SECRET, {
  expiresIn: '15m',
});

// Verify token
const decoded = jwt.verify(token, process.env.JWT_SECRET);
```

In Go, we built the equivalent functionality in about 100 lines. The logic is identical; the implementation is more explicit.

---

## 8. Validation and Responses

### `pkg/validator/validator.go`

A lightweight validation utility that checks struct tags and returns structured error messages.

```go
package validator

import (
	"fmt"
	"net/mail"
	"reflect"
	"strconv"
	"strings"
)

// ValidationError represents a single field validation failure.
type ValidationError struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// ValidationErrors is a collection of validation errors.
type ValidationErrors []ValidationError

// Error implements the error interface.
func (ve ValidationErrors) Error() string {
	if len(ve) == 0 {
		return "validation passed"
	}
	msgs := make([]string, len(ve))
	for i, e := range ve {
		msgs[i] = fmt.Sprintf("%s: %s", e.Field, e.Message)
	}
	return strings.Join(msgs, "; ")
}

// HasErrors returns true if there are validation errors.
func (ve ValidationErrors) HasErrors() bool {
	return len(ve) > 0
}

// Validate checks a struct against its `validate` tags.
// Supported tags: required, min, max, email, oneof, uuid, omitempty
func Validate(v interface{}) ValidationErrors {
	var errors ValidationErrors
	val := reflect.ValueOf(v)

	// Handle pointers
	if val.Kind() == reflect.Ptr {
		if val.IsNil() {
			errors = append(errors, ValidationError{
				Field:   "body",
				Message: "request body is required",
			})
			return errors
		}
		val = val.Elem()
	}

	if val.Kind() != reflect.Struct {
		return errors
	}

	typ := val.Type()
	for i := 0; i < val.NumField(); i++ {
		field := typ.Field(i)
		fieldVal := val.Field(i)
		tag := field.Tag.Get("validate")
		if tag == "" || tag == "-" {
			continue
		}

		// Get JSON field name for error messages
		jsonTag := field.Tag.Get("json")
		fieldName := field.Name
		if jsonTag != "" {
			parts := strings.Split(jsonTag, ",")
			if parts[0] != "" && parts[0] != "-" {
				fieldName = parts[0]
			}
		}

		rules := strings.Split(tag, ",")
		isOmitempty := false
		for _, rule := range rules {
			if rule == "omitempty" {
				isOmitempty = true
				break
			}
		}

		// For pointer fields, check nil
		isNil := false
		if fieldVal.Kind() == reflect.Ptr {
			if fieldVal.IsNil() {
				isNil = true
				// If omitempty and nil, skip all validations
				if isOmitempty {
					continue
				}
			} else {
				fieldVal = fieldVal.Elem()
			}
		}

		for _, rule := range rules {
			if rule == "omitempty" {
				continue
			}

			switch {
			case rule == "required":
				if isNil || isZero(fieldVal) {
					errors = append(errors, ValidationError{
						Field:   fieldName,
						Message: "is required",
					})
				}

			case rule == "email":
				if !isNil && fieldVal.Kind() == reflect.String {
					str := fieldVal.String()
					if str != "" {
						if _, err := mail.ParseAddress(str); err != nil {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: "must be a valid email address",
							})
						}
					}
				}

			case rule == "uuid":
				if !isNil && fieldVal.Kind() == reflect.String {
					str := fieldVal.String()
					if str != "" && !isValidUUID(str) {
						errors = append(errors, ValidationError{
							Field:   fieldName,
							Message: "must be a valid UUID",
						})
					}
				}

			case strings.HasPrefix(rule, "min="):
				minStr := strings.TrimPrefix(rule, "min=")
				minVal, _ := strconv.Atoi(minStr)
				if !isNil {
					switch fieldVal.Kind() {
					case reflect.String:
						if len(fieldVal.String()) < minVal {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: fmt.Sprintf("must be at least %d characters", minVal),
							})
						}
					case reflect.Int, reflect.Int64:
						if fieldVal.Int() < int64(minVal) {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: fmt.Sprintf("must be at least %d", minVal),
							})
						}
					}
				}

			case strings.HasPrefix(rule, "max="):
				maxStr := strings.TrimPrefix(rule, "max=")
				maxVal, _ := strconv.Atoi(maxStr)
				if !isNil {
					switch fieldVal.Kind() {
					case reflect.String:
						if len(fieldVal.String()) > maxVal {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: fmt.Sprintf("must be at most %d characters", maxVal),
							})
						}
					case reflect.Int, reflect.Int64:
						if fieldVal.Int() > int64(maxVal) {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: fmt.Sprintf("must be at most %d", maxVal),
							})
						}
					}
				}

			case strings.HasPrefix(rule, "oneof="):
				optionsStr := strings.TrimPrefix(rule, "oneof=")
				options := strings.Fields(optionsStr)
				if !isNil && fieldVal.Kind() == reflect.String {
					val := fieldVal.String()
					if val != "" {
						found := false
						for _, opt := range options {
							if val == opt {
								found = true
								break
							}
						}
						if !found {
							errors = append(errors, ValidationError{
								Field:   fieldName,
								Message: fmt.Sprintf("must be one of: %s", strings.Join(options, ", ")),
							})
						}
					}
				}
			}
		}
	}

	return errors
}

func isZero(v reflect.Value) bool {
	switch v.Kind() {
	case reflect.String:
		return v.String() == ""
	case reflect.Int, reflect.Int64:
		return v.Int() == 0
	case reflect.Bool:
		return !v.Bool()
	case reflect.Slice, reflect.Map:
		return v.Len() == 0
	default:
		return false
	}
}

func isValidUUID(s string) bool {
	// UUID format: 8-4-4-4-12 hex digits
	if len(s) != 36 {
		return false
	}
	for i, c := range s {
		if i == 8 || i == 13 || i == 18 || i == 23 {
			if c != '-' {
				return false
			}
		} else {
			if !((c >= '0' && c <= '9') || (c >= 'a' && c <= 'f') || (c >= 'A' && c <= 'F')) {
				return false
			}
		}
	}
	return true
}
```

### `pkg/response/response.go`

Standardized API response helpers ensure every endpoint returns consistent JSON.

```go
package response

import (
	"encoding/json"
	"net/http"
)

// APIResponse is the standard wrapper for all API responses.
type APIResponse struct {
	Success bool        `json:"success"`
	Data    interface{} `json:"data,omitempty"`
	Error   *APIError   `json:"error,omitempty"`
	Meta    interface{} `json:"meta,omitempty"`
}

// APIError represents an error in the response.
type APIError struct {
	Code    string      `json:"code"`
	Message string      `json:"message"`
	Details interface{} `json:"details,omitempty"`
}

// JSON writes a JSON response with the given status code.
func JSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(APIResponse{
		Success: status >= 200 && status < 300,
		Data:    data,
	})
}

// JSONWithMeta writes a JSON response with additional metadata (e.g., pagination).
func JSONWithMeta(w http.ResponseWriter, status int, data interface{}, meta interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(APIResponse{
		Success: true,
		Data:    data,
		Meta:    meta,
	})
}

// Error writes an error response.
func Error(w http.ResponseWriter, status int, code, message string) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(APIResponse{
		Success: false,
		Error: &APIError{
			Code:    code,
			Message: message,
		},
	})
}

// ErrorWithDetails writes an error response with additional details (e.g., validation errors).
func ErrorWithDetails(w http.ResponseWriter, status int, code, message string, details interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(APIResponse{
		Success: false,
		Error: &APIError{
			Code:    code,
			Message: message,
			Details: details,
		},
	})
}

// Standard error responses (convenience functions)

func BadRequest(w http.ResponseWriter, message string) {
	Error(w, http.StatusBadRequest, "BAD_REQUEST", message)
}

func Unauthorized(w http.ResponseWriter, message string) {
	Error(w, http.StatusUnauthorized, "UNAUTHORIZED", message)
}

func Forbidden(w http.ResponseWriter, message string) {
	Error(w, http.StatusForbidden, "FORBIDDEN", message)
}

func NotFound(w http.ResponseWriter, message string) {
	Error(w, http.StatusNotFound, "NOT_FOUND", message)
}

func Conflict(w http.ResponseWriter, message string) {
	Error(w, http.StatusConflict, "CONFLICT", message)
}

func TooManyRequests(w http.ResponseWriter, message string) {
	Error(w, http.StatusTooManyRequests, "TOO_MANY_REQUESTS", message)
}

func InternalError(w http.ResponseWriter, message string) {
	Error(w, http.StatusInternalServerError, "INTERNAL_ERROR", message)
}

// Created writes a 201 Created response with the given data.
func Created(w http.ResponseWriter, data interface{}) {
	JSON(w, http.StatusCreated, data)
}

// NoContent writes a 204 No Content response.
func NoContent(w http.ResponseWriter) {
	w.WriteHeader(http.StatusNoContent)
}
```

### Response Format

Every response from our API follows this shape:

```json
// Success
{
  "success": true,
  "data": { ... },
  "meta": {
    "pagination": { "page": 1, "perPage": 20, "total": 100, "totalPages": 5 }
  }
}

// Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": [
      { "field": "title", "message": "is required" },
      { "field": "priority", "message": "must be one of: low, medium, high, urgent" }
    ]
  }
}
```

This consistency makes the API predictable for frontend developers. They always check `success`, and error handling always follows the same pattern.

### Node.js/Express Comparison

In Express, developers often use inconsistent response shapes:

```javascript
// Inconsistent Express responses (common anti-pattern)
res.json({ user: data });           // Sometimes nested under entity name
res.json({ data: users });          // Sometimes under "data"
res.json({ error: 'Not found' });   // Sometimes error is a string
res.json({ errors: [{ ... }] });    // Sometimes an array
```

Our Go response package enforces a single, consistent format across the entire API.

---

## 9. Middleware Stack

Middleware wraps handlers to add cross-cutting concerns: panic recovery, request logging, CORS, rate limiting, and authentication. Each middleware is a function that takes an `http.Handler` and returns an `http.Handler`.

### `internal/middleware/recovery.go`

This is the outermost middleware. It catches panics from any downstream handler or middleware, logs the stack trace, and returns a 500 error instead of crashing the server.

```go
package middleware

import (
	"log/slog"
	"net/http"
	"runtime/debug"

	"taskapi/pkg/response"
)

// Recovery catches panics and returns a 500 error.
func Recovery(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if err := recover(); err != nil {
					stack := debug.Stack()
					logger.Error("panic recovered",
						"error", err,
						"stack", string(stack),
						"method", r.Method,
						"path", r.URL.Path,
					)
					response.InternalError(w, "Internal server error")
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}
```

### `internal/middleware/logging.go`

Logs every request with method, path, status code, duration, and request ID.

```go
package middleware

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"log/slog"
	"net/http"
	"time"
)

// contextKey is an unexported type for context keys to prevent collisions.
type contextKey string

const (
	// RequestIDKey is the context key for the request ID.
	RequestIDKey contextKey = "requestID"
	// UserIDKey is the context key for the authenticated user's ID.
	UserIDKey contextKey = "userID"
	// UserRoleKey is the context key for the authenticated user's role.
	UserRoleKey contextKey = "userRole"
)

// GetRequestID extracts the request ID from the context.
func GetRequestID(ctx context.Context) string {
	if id, ok := ctx.Value(RequestIDKey).(string); ok {
		return id
	}
	return ""
}

// GetUserID extracts the authenticated user ID from the context.
func GetUserID(ctx context.Context) string {
	if id, ok := ctx.Value(UserIDKey).(string); ok {
		return id
	}
	return ""
}

// GetUserRole extracts the authenticated user role from the context.
func GetUserRole(ctx context.Context) string {
	if role, ok := ctx.Value(UserRoleKey).(string); ok {
		return role
	}
	return ""
}

// statusRecorder wraps http.ResponseWriter to capture the status code.
type statusRecorder struct {
	http.ResponseWriter
	statusCode int
	written    bool
}

func (r *statusRecorder) WriteHeader(code int) {
	if !r.written {
		r.statusCode = code
		r.written = true
	}
	r.ResponseWriter.WriteHeader(code)
}

func (r *statusRecorder) Write(b []byte) (int, error) {
	if !r.written {
		r.statusCode = http.StatusOK
		r.written = true
	}
	return r.ResponseWriter.Write(b)
}

// Logging logs each request with its duration, status, and request ID.
func Logging(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// Generate request ID
			requestID := generateRequestID()

			// Add request ID to context
			ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
			r = r.WithContext(ctx)

			// Set request ID header
			w.Header().Set("X-Request-ID", requestID)

			// Wrap response writer to capture status code
			rec := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}

			// Call next handler
			next.ServeHTTP(rec, r)

			// Log the request
			duration := time.Since(start)
			level := slog.LevelInfo
			if rec.statusCode >= 500 {
				level = slog.LevelError
			} else if rec.statusCode >= 400 {
				level = slog.LevelWarn
			}

			logger.LogAttrs(r.Context(), level, "http request",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.String("query", r.URL.RawQuery),
				slog.Int("status", rec.statusCode),
				slog.Duration("duration", duration),
				slog.String("request_id", requestID),
				slog.String("remote_addr", r.RemoteAddr),
				slog.String("user_agent", r.UserAgent()),
			)
		})
	}
}

func generateRequestID() string {
	b := make([]byte, 8)
	rand.Read(b)
	return hex.EncodeToString(b)
}
```

### `internal/middleware/cors.go`

CORS (Cross-Origin Resource Sharing) allows the frontend application (running on a different domain) to call our API.

```go
package middleware

import (
	"net/http"
	"strings"
)

// CORSConfig holds CORS configuration.
type CORSConfig struct {
	AllowedOrigins []string
	AllowedMethods []string
	AllowedHeaders []string
	MaxAge         int // seconds
}

// DefaultCORSConfig returns sensible CORS defaults.
func DefaultCORSConfig() CORSConfig {
	return CORSConfig{
		AllowedOrigins: []string{"*"},
		AllowedMethods: []string{"GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"},
		AllowedHeaders: []string{"Content-Type", "Authorization", "X-Request-ID"},
		MaxAge:         86400, // 24 hours
	}
}

// CORS adds Cross-Origin Resource Sharing headers.
func CORS(config CORSConfig) func(http.Handler) http.Handler {
	allowedOrigins := make(map[string]bool)
	allowAll := false
	for _, origin := range config.AllowedOrigins {
		if origin == "*" {
			allowAll = true
			break
		}
		allowedOrigins[origin] = true
	}

	methods := strings.Join(config.AllowedMethods, ", ")
	headers := strings.Join(config.AllowedHeaders, ", ")

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")

			if allowAll {
				w.Header().Set("Access-Control-Allow-Origin", "*")
			} else if allowedOrigins[origin] {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Vary", "Origin")
			}

			w.Header().Set("Access-Control-Allow-Methods", methods)
			w.Header().Set("Access-Control-Allow-Headers", headers)
			w.Header().Set("Access-Control-Max-Age", strings.Itoa(config.MaxAge))

			// Handle preflight requests
			if r.Method == http.MethodOptions {
				w.WriteHeader(http.StatusNoContent)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

### `internal/middleware/ratelimit.go`

A token bucket rate limiter that tracks requests per IP address.

```go
package middleware

import (
	"log/slog"
	"net"
	"net/http"
	"sync"
	"time"

	"taskapi/pkg/response"
)

// visitor tracks rate limiting state for a single client.
type visitor struct {
	tokens    float64
	lastSeen  time.Time
}

// RateLimiter implements a per-IP token bucket rate limiter.
type RateLimiter struct {
	mu          sync.Mutex
	visitors    map[string]*visitor
	maxTokens   float64
	refillRate  float64 // tokens per second
	logger      *slog.Logger
}

// NewRateLimiter creates a rate limiter that allows maxRequests per window.
func NewRateLimiter(maxRequests int, window time.Duration, logger *slog.Logger) *RateLimiter {
	rl := &RateLimiter{
		visitors:   make(map[string]*visitor),
		maxTokens:  float64(maxRequests),
		refillRate: float64(maxRequests) / window.Seconds(),
		logger:     logger,
	}

	// Background cleanup of stale entries every minute
	go rl.cleanup()

	return rl
}

// Middleware returns the rate limiting middleware.
func (rl *RateLimiter) Middleware() func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ip := extractIP(r)

			if !rl.allow(ip) {
				rl.logger.Warn("rate limit exceeded",
					"ip", ip,
					"path", r.URL.Path,
				)
				w.Header().Set("Retry-After", "60")
				response.TooManyRequests(w, "Rate limit exceeded. Please try again later.")
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// allow checks if the given IP has tokens available.
func (rl *RateLimiter) allow(ip string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	v, exists := rl.visitors[ip]
	now := time.Now()

	if !exists {
		rl.visitors[ip] = &visitor{
			tokens:   rl.maxTokens - 1, // consume one token
			lastSeen: now,
		}
		return true
	}

	// Refill tokens based on elapsed time
	elapsed := now.Sub(v.lastSeen).Seconds()
	v.tokens += elapsed * rl.refillRate
	if v.tokens > rl.maxTokens {
		v.tokens = rl.maxTokens
	}
	v.lastSeen = now

	if v.tokens < 1 {
		return false
	}

	v.tokens--
	return true
}

// cleanup removes visitors not seen in the last 3 minutes.
func (rl *RateLimiter) cleanup() {
	ticker := time.NewTicker(1 * time.Minute)
	defer ticker.Stop()

	for range ticker.C {
		rl.mu.Lock()
		threshold := time.Now().Add(-3 * time.Minute)
		for ip, v := range rl.visitors {
			if v.lastSeen.Before(threshold) {
				delete(rl.visitors, ip)
			}
		}
		rl.mu.Unlock()
	}
}

// extractIP gets the client IP from the request, handling proxies.
func extractIP(r *http.Request) string {
	// Check X-Forwarded-For first (behind reverse proxy)
	xff := r.Header.Get("X-Forwarded-For")
	if xff != "" {
		// Take the first IP (client IP)
		parts := strings.SplitN(xff, ",", 2)
		ip := strings.TrimSpace(parts[0])
		if ip != "" {
			return ip
		}
	}

	// Check X-Real-IP
	xri := r.Header.Get("X-Real-IP")
	if xri != "" {
		return xri
	}

	// Fall back to RemoteAddr
	ip, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return ip
}
```

### `internal/middleware/auth.go`

The authentication middleware validates the JWT token and injects the user information into the request context.

```go
package middleware

import (
	"context"
	"net/http"
	"strings"

	"taskapi/internal/auth"
	"taskapi/pkg/response"
)

// Auth creates an authentication middleware using the given JWT service.
func Auth(jwtService *auth.JWTService) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Extract token from Authorization header
			authHeader := r.Header.Get("Authorization")
			if authHeader == "" {
				response.Unauthorized(w, "Missing authorization header")
				return
			}

			// Expect "Bearer <token>"
			parts := strings.SplitN(authHeader, " ", 2)
			if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
				response.Unauthorized(w, "Invalid authorization header format")
				return
			}

			tokenStr := parts[1]

			// Validate token
			claims, err := jwtService.ValidateToken(tokenStr)
			if err != nil {
				response.Unauthorized(w, "Invalid or expired token")
				return
			}

			// Ensure it's an access token, not a refresh token
			if claims.Type != "access" {
				response.Unauthorized(w, "Invalid token type")
				return
			}

			// Add user info to context
			ctx := r.Context()
			ctx = context.WithValue(ctx, UserIDKey, claims.UserID)
			ctx = context.WithValue(ctx, UserRoleKey, claims.Role)

			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// RequireRole creates middleware that checks for a specific user role.
func RequireRole(roles ...string) func(http.Handler) http.Handler {
	roleSet := make(map[string]bool, len(roles))
	for _, r := range roles {
		roleSet[r] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userRole := GetUserRole(r.Context())
			if !roleSet[userRole] {
				response.Forbidden(w, "Insufficient permissions")
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

### Middleware Chaining

All these middlewares need to be composed in order. The standard Go pattern is to nest function calls:

```go
// The middleware chain executes from outermost to innermost.
// Recovery runs first (catches panics from everything inside),
// then Logging, then CORS, then RateLimit, then the router.
handler := Recovery(logger)(
    Logging(logger)(
        CORS(corsConfig)(
            rateLimiter.Middleware()(
                router,
            ),
        ),
    ),
)
```

This nesting can be hard to read. We can write a `Chain` helper:

```go
package middleware

import "net/http"

// Chain composes multiple middleware into a single middleware.
// Middlewares are applied in order: first in the list wraps outermost.
func Chain(middlewares ...func(http.Handler) http.Handler) func(http.Handler) http.Handler {
	return func(final http.Handler) http.Handler {
		for i := len(middlewares) - 1; i >= 0; i-- {
			final = middlewares[i](final)
		}
		return final
	}
}
```

Usage becomes much cleaner:

```go
stack := middleware.Chain(
    middleware.Recovery(logger),
    middleware.Logging(logger),
    middleware.CORS(corsConfig),
    rateLimiter.Middleware(),
)
handler := stack(router)
```

### Node.js/Express Comparison

Express middleware works the same conceptual way but with a different syntax:

```javascript
// Express middleware chain
app.use(morgan('combined'));     // Logging
app.use(cors());                 // CORS
app.use(helmet());               // Security headers
app.use(rateLimit({ ... }));     // Rate limiting
app.use('/api', authMiddleware); // Auth (on specific routes)
```

Express uses `app.use()` which adds middleware in order. In Go, we explicitly compose functions. The Go approach makes the execution order and dependencies more visible.

---

## 10. Router

The router maps incoming HTTP requests to handler functions based on method and path. We use Go 1.22's enhanced `net/http.ServeMux` which supports method-based routing and path parameters natively.

### `internal/router/router.go`

```go
package router

import (
	"net/http"

	"taskapi/internal/auth"
	"taskapi/internal/handlers"
	"taskapi/internal/middleware"
)

// Config holds the dependencies needed to build the router.
type Config struct {
	AuthHandler      *handlers.AuthHandler
	ProjectHandler   *handlers.ProjectHandler
	TaskHandler      *handlers.TaskHandler
	CommentHandler   *handlers.CommentHandler
	DashboardHandler *handlers.DashboardHandler
	HealthHandler    *handlers.HealthHandler
	JWTService       *auth.JWTService
}

// New creates and configures the application router with all routes.
func New(cfg Config) http.Handler {
	mux := http.NewServeMux()

	// Auth middleware wrapper -- wraps a handler function with JWT authentication.
	authMW := middleware.Auth(cfg.JWTService)
	requireAuth := func(h http.HandlerFunc) http.Handler {
		return authMW(h)
	}

	// --- Public routes (no authentication required) ---

	// Health check
	mux.HandleFunc("GET /health", cfg.HealthHandler.Health)
	mux.HandleFunc("GET /ready", cfg.HealthHandler.Ready)

	// Authentication
	mux.HandleFunc("POST /api/v1/auth/register", cfg.AuthHandler.Register)
	mux.HandleFunc("POST /api/v1/auth/login", cfg.AuthHandler.Login)
	mux.HandleFunc("POST /api/v1/auth/refresh", cfg.AuthHandler.Refresh)

	// --- Protected routes (authentication required) ---

	// User profile
	mux.Handle("GET /api/v1/auth/me", requireAuth(cfg.AuthHandler.Me))

	// Projects
	mux.Handle("GET /api/v1/projects", requireAuth(cfg.ProjectHandler.List))
	mux.Handle("POST /api/v1/projects", requireAuth(cfg.ProjectHandler.Create))
	mux.Handle("GET /api/v1/projects/{id}", requireAuth(cfg.ProjectHandler.Get))
	mux.Handle("PUT /api/v1/projects/{id}", requireAuth(cfg.ProjectHandler.Update))
	mux.Handle("DELETE /api/v1/projects/{id}", requireAuth(cfg.ProjectHandler.Delete))

	// Tasks (nested under projects for creation and listing)
	mux.Handle("GET /api/v1/projects/{projectId}/tasks", requireAuth(cfg.TaskHandler.ListByProject))
	mux.Handle("POST /api/v1/projects/{projectId}/tasks", requireAuth(cfg.TaskHandler.Create))

	// Tasks (direct access by task ID)
	mux.Handle("GET /api/v1/tasks/{id}", requireAuth(cfg.TaskHandler.Get))
	mux.Handle("PUT /api/v1/tasks/{id}", requireAuth(cfg.TaskHandler.Update))
	mux.Handle("DELETE /api/v1/tasks/{id}", requireAuth(cfg.TaskHandler.Delete))

	// Comments (nested under tasks)
	mux.Handle("GET /api/v1/tasks/{taskId}/comments", requireAuth(cfg.CommentHandler.ListByTask))
	mux.Handle("POST /api/v1/tasks/{taskId}/comments", requireAuth(cfg.CommentHandler.Create))
	mux.Handle("PUT /api/v1/comments/{id}", requireAuth(cfg.CommentHandler.Update))
	mux.Handle("DELETE /api/v1/comments/{id}", requireAuth(cfg.CommentHandler.Delete))

	// Dashboard
	mux.Handle("GET /api/v1/dashboard", requireAuth(cfg.DashboardHandler.Overview))

	return mux
}
```

### Route Table Summary

| Method | Path | Auth | Handler | Description |
|--------|------|------|---------|-------------|
| GET | `/health` | No | HealthHandler.Health | Liveness check |
| GET | `/ready` | No | HealthHandler.Ready | Readiness check (includes DB) |
| POST | `/api/v1/auth/register` | No | AuthHandler.Register | Create new user |
| POST | `/api/v1/auth/login` | No | AuthHandler.Login | Get JWT tokens |
| POST | `/api/v1/auth/refresh` | No | AuthHandler.Refresh | Refresh access token |
| GET | `/api/v1/auth/me` | Yes | AuthHandler.Me | Current user profile |
| GET | `/api/v1/projects` | Yes | ProjectHandler.List | List user's projects |
| POST | `/api/v1/projects` | Yes | ProjectHandler.Create | Create project |
| GET | `/api/v1/projects/{id}` | Yes | ProjectHandler.Get | Get project by ID |
| PUT | `/api/v1/projects/{id}` | Yes | ProjectHandler.Update | Update project |
| DELETE | `/api/v1/projects/{id}` | Yes | ProjectHandler.Delete | Delete project |
| GET | `/api/v1/projects/{projectId}/tasks` | Yes | TaskHandler.ListByProject | List tasks in project |
| POST | `/api/v1/projects/{projectId}/tasks` | Yes | TaskHandler.Create | Create task in project |
| GET | `/api/v1/tasks/{id}` | Yes | TaskHandler.Get | Get task by ID |
| PUT | `/api/v1/tasks/{id}` | Yes | TaskHandler.Update | Update task |
| DELETE | `/api/v1/tasks/{id}` | Yes | TaskHandler.Delete | Delete task |
| GET | `/api/v1/tasks/{taskId}/comments` | Yes | CommentHandler.ListByTask | List comments on task |
| POST | `/api/v1/tasks/{taskId}/comments` | Yes | CommentHandler.Create | Add comment to task |
| PUT | `/api/v1/comments/{id}` | Yes | CommentHandler.Update | Edit comment |
| DELETE | `/api/v1/comments/{id}` | Yes | CommentHandler.Delete | Delete comment |
| GET | `/api/v1/dashboard` | Yes | DashboardHandler.Overview | Dashboard stats |

### Go 1.22 ServeMux vs Express Router

Go 1.22's enhanced `ServeMux` brings it much closer to Express:

```go
// Go 1.22+
mux.HandleFunc("GET /api/v1/projects/{id}", handler)
id := r.PathValue("id")  // Extract path parameter

// Express equivalent
router.get('/api/v1/projects/:id', handler);
req.params.id  // Extract path parameter
```

Before Go 1.22, you needed a third-party router like `gorilla/mux` or `chi` for path parameters and method-based routing. Now the standard library handles it natively.

---

## 11. Handlers

Handlers are the HTTP-facing layer. They parse requests, call services or repositories, and write responses. They should be thin -- business logic belongs in the service layer.

### `internal/handlers/health_handler.go`

```go
package handlers

import (
	"net/http"

	"taskapi/internal/database"
	"taskapi/pkg/response"
)

// HealthHandler provides health check endpoints.
type HealthHandler struct {
	db *database.DB
}

// NewHealthHandler creates a new health handler.
func NewHealthHandler(db *database.DB) *HealthHandler {
	return &HealthHandler{db: db}
}

// Health is a simple liveness check. Returns 200 if the server is running.
func (h *HealthHandler) Health(w http.ResponseWriter, r *http.Request) {
	response.JSON(w, http.StatusOK, map[string]string{
		"status": "ok",
	})
}

// Ready checks if the application is ready to serve traffic (DB connected, etc.).
func (h *HealthHandler) Ready(w http.ResponseWriter, r *http.Request) {
	if err := h.db.Health(r.Context()); err != nil {
		response.Error(w, http.StatusServiceUnavailable, "NOT_READY", "Database is not available")
		return
	}
	response.JSON(w, http.StatusOK, map[string]string{
		"status": "ready",
	})
}
```

### `internal/handlers/auth_handler.go`

```go
package handlers

import (
	"encoding/json"
	"log/slog"
	"net/http"

	"taskapi/internal/auth"
	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/repository"
	"taskapi/pkg/response"
	"taskapi/pkg/validator"
)

// AuthHandler handles authentication endpoints.
type AuthHandler struct {
	userRepo   repository.UserRepository
	jwtService *auth.JWTService
	logger     *slog.Logger
}

// NewAuthHandler creates a new auth handler.
func NewAuthHandler(userRepo repository.UserRepository, jwtService *auth.JWTService, logger *slog.Logger) *AuthHandler {
	return &AuthHandler{
		userRepo:   userRepo,
		jwtService: jwtService,
		logger:     logger,
	}
}

// Register creates a new user account.
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	var req models.CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	// Validate input
	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	// Check if user already exists
	existing, err := h.userRepo.GetByEmail(r.Context(), req.Email)
	if err != nil {
		h.logger.Error("checking existing user", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if existing != nil {
		response.Conflict(w, "A user with this email already exists")
		return
	}

	// Hash password
	hashedPassword, err := auth.HashPassword(req.Password)
	if err != nil {
		h.logger.Error("hashing password", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	// Create user
	user := &models.User{
		Email:        req.Email,
		PasswordHash: hashedPassword,
		FullName:     req.FullName,
		Role:         models.RoleMember,
	}

	if err := h.userRepo.Create(r.Context(), user); err != nil {
		h.logger.Error("creating user", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("user registered",
		"user_id", user.ID,
		"email", user.Email,
	)

	// Generate tokens
	accessToken, err := h.jwtService.GenerateAccessToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating access token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	refreshToken, err := h.jwtService.GenerateRefreshToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating refresh token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	response.Created(w, models.AuthResponse{
		AccessToken:  accessToken,
		RefreshToken: refreshToken,
		ExpiresIn:    int(h.jwtService.AccessExpiry().Seconds()),
		User:         user,
	})
}

// Login authenticates a user and returns JWT tokens.
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req models.LoginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	// Look up user
	user, err := h.userRepo.GetByEmail(r.Context(), req.Email)
	if err != nil {
		h.logger.Error("looking up user", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if user == nil {
		response.Unauthorized(w, "Invalid email or password")
		return
	}

	// Check password
	if !auth.CheckPassword(req.Password, user.PasswordHash) {
		response.Unauthorized(w, "Invalid email or password")
		return
	}

	if !user.IsActive {
		response.Forbidden(w, "Account is deactivated")
		return
	}

	// Generate tokens
	accessToken, err := h.jwtService.GenerateAccessToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating access token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	refreshToken, err := h.jwtService.GenerateRefreshToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating refresh token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("user logged in",
		"user_id", user.ID,
		"email", user.Email,
	)

	response.JSON(w, http.StatusOK, models.AuthResponse{
		AccessToken:  accessToken,
		RefreshToken: refreshToken,
		ExpiresIn:    int(h.jwtService.AccessExpiry().Seconds()),
		User:         user,
	})
}

// Refresh exchanges a refresh token for a new access token.
func (h *AuthHandler) Refresh(w http.ResponseWriter, r *http.Request) {
	var req models.RefreshRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	// Validate the refresh token
	claims, err := h.jwtService.ValidateToken(req.RefreshToken)
	if err != nil {
		response.Unauthorized(w, "Invalid or expired refresh token")
		return
	}

	if claims.Type != "refresh" {
		response.Unauthorized(w, "Invalid token type")
		return
	}

	// Verify user still exists and is active
	user, err := h.userRepo.GetByID(r.Context(), claims.UserID)
	if err != nil {
		h.logger.Error("looking up user for refresh", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if user == nil || !user.IsActive {
		response.Unauthorized(w, "User not found or deactivated")
		return
	}

	// Generate new access token
	accessToken, err := h.jwtService.GenerateAccessToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating access token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	// Generate new refresh token (rotation)
	newRefreshToken, err := h.jwtService.GenerateRefreshToken(user.ID, user.Email, string(user.Role))
	if err != nil {
		h.logger.Error("generating refresh token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	response.JSON(w, http.StatusOK, models.AuthResponse{
		AccessToken:  accessToken,
		RefreshToken: newRefreshToken,
		ExpiresIn:    int(h.jwtService.AccessExpiry().Seconds()),
		User:         user,
	})
}

// Me returns the currently authenticated user's profile.
func (h *AuthHandler) Me(w http.ResponseWriter, r *http.Request) {
	userID := middleware.GetUserID(r.Context())
	if userID == "" {
		response.Unauthorized(w, "Not authenticated")
		return
	}

	user, err := h.userRepo.GetByID(r.Context(), userID)
	if err != nil {
		h.logger.Error("getting user profile", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if user == nil {
		response.NotFound(w, "User not found")
		return
	}

	response.JSON(w, http.StatusOK, user)
}
```

### `internal/handlers/project_handler.go`

```go
package handlers

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"strconv"

	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/repository"
	"taskapi/pkg/response"
	"taskapi/pkg/validator"
)

// ProjectHandler handles project CRUD endpoints.
type ProjectHandler struct {
	projectRepo repository.ProjectRepository
	logger      *slog.Logger
}

// NewProjectHandler creates a new project handler.
func NewProjectHandler(projectRepo repository.ProjectRepository, logger *slog.Logger) *ProjectHandler {
	return &ProjectHandler{
		projectRepo: projectRepo,
		logger:      logger,
	}
}

// List returns the authenticated user's projects with pagination.
func (h *ProjectHandler) List(w http.ResponseWriter, r *http.Request) {
	userID := middleware.GetUserID(r.Context())

	params := parsePagination(r)

	projects, total, err := h.projectRepo.ListByUser(r.Context(), userID, params)
	if err != nil {
		h.logger.Error("listing projects", "error", err, "user_id", userID)
		response.InternalError(w, "Internal server error")
		return
	}

	if projects == nil {
		projects = []models.Project{}
	}

	meta := models.NewPaginationMeta(params.Page, params.PerPage, total)
	response.JSONWithMeta(w, http.StatusOK, projects, map[string]interface{}{
		"pagination": meta,
	})
}

// Create creates a new project owned by the authenticated user.
func (h *ProjectHandler) Create(w http.ResponseWriter, r *http.Request) {
	userID := middleware.GetUserID(r.Context())

	var req models.CreateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	project := &models.Project{
		Name:        req.Name,
		Description: req.Description,
		OwnerID:     userID,
	}

	if err := h.projectRepo.Create(r.Context(), project); err != nil {
		h.logger.Error("creating project", "error", err, "user_id", userID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project created",
		"project_id", project.ID,
		"owner_id", userID,
		"name", project.Name,
	)

	response.Created(w, project)
}

// Get returns a single project by ID.
func (h *ProjectHandler) Get(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	project, err := h.projectRepo.GetByID(r.Context(), projectID)
	if err != nil {
		h.logger.Error("getting project", "error", err, "project_id", projectID)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	// Check membership
	isMember, err := h.projectRepo.IsMember(r.Context(), projectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You are not a member of this project")
		return
	}

	response.JSON(w, http.StatusOK, project)
}

// Update modifies an existing project.
func (h *ProjectHandler) Update(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	project, err := h.projectRepo.GetByID(r.Context(), projectID)
	if err != nil {
		h.logger.Error("getting project for update", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	// Only the owner can update the project
	if project.OwnerID != userID {
		response.Forbidden(w, "Only the project owner can update it")
		return
	}

	var req models.UpdateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	// Apply partial updates
	if req.Name != nil {
		project.Name = *req.Name
	}
	if req.Description != nil {
		project.Description = req.Description
	}
	if req.IsArchived != nil {
		project.IsArchived = *req.IsArchived
	}

	if err := h.projectRepo.Update(r.Context(), project); err != nil {
		h.logger.Error("updating project", "error", err, "project_id", projectID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project updated", "project_id", projectID, "user_id", userID)
	response.JSON(w, http.StatusOK, project)
}

// Delete removes a project and all its tasks.
func (h *ProjectHandler) Delete(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	project, err := h.projectRepo.GetByID(r.Context(), projectID)
	if err != nil {
		h.logger.Error("getting project for delete", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	if project.OwnerID != userID {
		response.Forbidden(w, "Only the project owner can delete it")
		return
	}

	if err := h.projectRepo.Delete(r.Context(), projectID); err != nil {
		h.logger.Error("deleting project", "error", err, "project_id", projectID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project deleted", "project_id", projectID, "user_id", userID)
	response.NoContent(w)
}

// parsePagination extracts pagination parameters from query string.
func parsePagination(r *http.Request) models.PaginationParams {
	params := models.DefaultPagination()

	if p := r.URL.Query().Get("page"); p != "" {
		if page, err := strconv.Atoi(p); err == nil && page > 0 {
			params.Page = page
		}
	}

	if pp := r.URL.Query().Get("perPage"); pp != "" {
		if perPage, err := strconv.Atoi(pp); err == nil && perPage > 0 && perPage <= 100 {
			params.PerPage = perPage
		}
	}

	return params
}
```

### `internal/handlers/task_handler.go`

```go
package handlers

import (
	"encoding/json"
	"log/slog"
	"net/http"

	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/repository"
	"taskapi/pkg/response"
	"taskapi/pkg/validator"
)

// TaskHandler handles task CRUD endpoints.
type TaskHandler struct {
	taskRepo    repository.TaskRepository
	projectRepo repository.ProjectRepository
	logger      *slog.Logger
}

// NewTaskHandler creates a new task handler.
func NewTaskHandler(taskRepo repository.TaskRepository, projectRepo repository.ProjectRepository, logger *slog.Logger) *TaskHandler {
	return &TaskHandler{
		taskRepo:    taskRepo,
		projectRepo: projectRepo,
		logger:      logger,
	}
}

// ListByProject returns tasks for a project with filtering and pagination.
func (h *TaskHandler) ListByProject(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("projectId")
	userID := middleware.GetUserID(r.Context())

	// Verify project membership
	isMember, err := h.projectRepo.IsMember(r.Context(), projectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You are not a member of this project")
		return
	}

	// Parse filters from query parameters
	filter := models.TaskFilter{}
	if status := r.URL.Query().Get("status"); status != "" {
		s := models.TaskStatus(status)
		filter.Status = &s
	}
	if priority := r.URL.Query().Get("priority"); priority != "" {
		p := models.Priority(priority)
		filter.Priority = &p
	}
	if assignee := r.URL.Query().Get("assigneeId"); assignee != "" {
		filter.AssigneeID = &assignee
	}
	if search := r.URL.Query().Get("search"); search != "" {
		filter.Search = &search
	}

	params := parsePagination(r)

	tasks, total, err := h.taskRepo.ListByProject(r.Context(), projectID, filter, params)
	if err != nil {
		h.logger.Error("listing tasks", "error", err, "project_id", projectID)
		response.InternalError(w, "Internal server error")
		return
	}

	if tasks == nil {
		tasks = []models.Task{}
	}

	meta := models.NewPaginationMeta(params.Page, params.PerPage, total)
	response.JSONWithMeta(w, http.StatusOK, tasks, map[string]interface{}{
		"pagination": meta,
	})
}

// Create adds a new task to a project.
func (h *TaskHandler) Create(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("projectId")
	userID := middleware.GetUserID(r.Context())

	// Verify project membership
	isMember, err := h.projectRepo.IsMember(r.Context(), projectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You are not a member of this project")
		return
	}

	var req models.CreateTaskRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	task := &models.Task{
		ProjectID:   projectID,
		Title:       req.Title,
		Description: req.Description,
		Status:      req.Status,
		Priority:    req.Priority,
		AssigneeID:  req.AssigneeID,
		CreatorID:   userID,
		DueDate:     req.DueDate,
	}

	if err := h.taskRepo.Create(r.Context(), task); err != nil {
		h.logger.Error("creating task", "error", err, "project_id", projectID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task created",
		"task_id", task.ID,
		"project_id", projectID,
		"creator_id", userID,
	)

	response.Created(w, task)
}

// Get returns a single task by ID.
func (h *TaskHandler) Get(w http.ResponseWriter, r *http.Request) {
	taskID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	task, err := h.taskRepo.GetByID(r.Context(), taskID)
	if err != nil {
		h.logger.Error("getting task", "error", err, "task_id", taskID)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	// Verify the user has access to this task's project
	isMember, err := h.projectRepo.IsMember(r.Context(), task.ProjectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	response.JSON(w, http.StatusOK, task)
}

// Update modifies an existing task.
func (h *TaskHandler) Update(w http.ResponseWriter, r *http.Request) {
	taskID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	task, err := h.taskRepo.GetByID(r.Context(), taskID)
	if err != nil {
		h.logger.Error("getting task for update", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	// Verify project membership
	isMember, err := h.projectRepo.IsMember(r.Context(), task.ProjectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	var req models.UpdateTaskRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	// Apply partial updates
	if req.Title != nil {
		task.Title = *req.Title
	}
	if req.Description != nil {
		task.Description = req.Description
	}
	if req.Status != nil {
		task.Status = *req.Status
	}
	if req.Priority != nil {
		task.Priority = *req.Priority
	}
	if req.AssigneeID != nil {
		task.AssigneeID = req.AssigneeID
	}
	if req.DueDate != nil {
		task.DueDate = req.DueDate
	}

	if err := h.taskRepo.Update(r.Context(), task); err != nil {
		h.logger.Error("updating task", "error", err, "task_id", taskID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task updated", "task_id", taskID, "user_id", userID)
	response.JSON(w, http.StatusOK, task)
}

// Delete removes a task.
func (h *TaskHandler) Delete(w http.ResponseWriter, r *http.Request) {
	taskID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	task, err := h.taskRepo.GetByID(r.Context(), taskID)
	if err != nil {
		h.logger.Error("getting task for delete", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	// Verify project membership
	isMember, err := h.projectRepo.IsMember(r.Context(), task.ProjectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	if err := h.taskRepo.Delete(r.Context(), taskID); err != nil {
		h.logger.Error("deleting task", "error", err, "task_id", taskID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task deleted", "task_id", taskID, "user_id", userID)
	response.NoContent(w)
}
```

### `internal/handlers/comment_handler.go`

```go
package handlers

import (
	"encoding/json"
	"log/slog"
	"net/http"

	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/repository"
	"taskapi/pkg/response"
	"taskapi/pkg/validator"
)

// CommentHandler handles comment endpoints.
type CommentHandler struct {
	commentRepo repository.CommentRepository
	taskRepo    repository.TaskRepository
	projectRepo repository.ProjectRepository
	logger      *slog.Logger
}

// NewCommentHandler creates a new comment handler.
func NewCommentHandler(
	commentRepo repository.CommentRepository,
	taskRepo repository.TaskRepository,
	projectRepo repository.ProjectRepository,
	logger *slog.Logger,
) *CommentHandler {
	return &CommentHandler{
		commentRepo: commentRepo,
		taskRepo:    taskRepo,
		projectRepo: projectRepo,
		logger:      logger,
	}
}

// ListByTask returns comments for a task with pagination.
func (h *CommentHandler) ListByTask(w http.ResponseWriter, r *http.Request) {
	taskID := r.PathValue("taskId")
	userID := middleware.GetUserID(r.Context())

	// Verify the user has access to this task's project
	task, err := h.taskRepo.GetByID(r.Context(), taskID)
	if err != nil {
		h.logger.Error("getting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	isMember, err := h.projectRepo.IsMember(r.Context(), task.ProjectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	params := parsePagination(r)

	comments, total, err := h.commentRepo.ListByTask(r.Context(), taskID, params)
	if err != nil {
		h.logger.Error("listing comments", "error", err, "task_id", taskID)
		response.InternalError(w, "Internal server error")
		return
	}

	if comments == nil {
		comments = []models.Comment{}
	}

	meta := models.NewPaginationMeta(params.Page, params.PerPage, total)
	response.JSONWithMeta(w, http.StatusOK, comments, map[string]interface{}{
		"pagination": meta,
	})
}

// Create adds a comment to a task.
func (h *CommentHandler) Create(w http.ResponseWriter, r *http.Request) {
	taskID := r.PathValue("taskId")
	userID := middleware.GetUserID(r.Context())

	// Verify access
	task, err := h.taskRepo.GetByID(r.Context(), taskID)
	if err != nil {
		h.logger.Error("getting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	isMember, err := h.projectRepo.IsMember(r.Context(), task.ProjectID, userID)
	if err != nil {
		h.logger.Error("checking membership", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if !isMember {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	var req models.CreateCommentRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	comment := &models.Comment{
		TaskID:   taskID,
		AuthorID: userID,
		Body:     req.Body,
	}

	if err := h.commentRepo.Create(r.Context(), comment); err != nil {
		h.logger.Error("creating comment", "error", err, "task_id", taskID)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("comment created",
		"comment_id", comment.ID,
		"task_id", taskID,
		"author_id", userID,
	)

	response.Created(w, comment)
}

// Update modifies a comment (only the author can edit).
func (h *CommentHandler) Update(w http.ResponseWriter, r *http.Request) {
	commentID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	comment, err := h.commentRepo.GetByID(r.Context(), commentID)
	if err != nil {
		h.logger.Error("getting comment", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if comment == nil {
		response.NotFound(w, "Comment not found")
		return
	}

	if comment.AuthorID != userID {
		response.Forbidden(w, "You can only edit your own comments")
		return
	}

	var req models.UpdateCommentRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errors := validator.Validate(&req); errors.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errors)
		return
	}

	comment.Body = req.Body
	if err := h.commentRepo.Update(r.Context(), comment); err != nil {
		h.logger.Error("updating comment", "error", err, "comment_id", commentID)
		response.InternalError(w, "Internal server error")
		return
	}

	response.JSON(w, http.StatusOK, comment)
}

// Delete removes a comment (only the author can delete).
func (h *CommentHandler) Delete(w http.ResponseWriter, r *http.Request) {
	commentID := r.PathValue("id")
	userID := middleware.GetUserID(r.Context())

	comment, err := h.commentRepo.GetByID(r.Context(), commentID)
	if err != nil {
		h.logger.Error("getting comment", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if comment == nil {
		response.NotFound(w, "Comment not found")
		return
	}

	if comment.AuthorID != userID {
		response.Forbidden(w, "You can only delete your own comments")
		return
	}

	if err := h.commentRepo.Delete(r.Context(), commentID); err != nil {
		h.logger.Error("deleting comment", "error", err, "comment_id", commentID)
		response.InternalError(w, "Internal server error")
		return
	}

	response.NoContent(w)
}
```

### `internal/handlers/dashboard_handler.go`

The dashboard handler demonstrates the concurrency patterns from Chapter 31 -- we fetch multiple statistics in parallel.

```go
package handlers

import (
	"context"
	"log/slog"
	"net/http"
	"sync"

	"taskapi/internal/repository"
	"taskapi/pkg/response"
)

// DashboardHandler provides aggregated dashboard data.
type DashboardHandler struct {
	userRepo    repository.UserRepository
	projectRepo repository.ProjectRepository
	taskRepo    repository.TaskRepository
	logger      *slog.Logger
}

// NewDashboardHandler creates a new dashboard handler.
func NewDashboardHandler(
	userRepo repository.UserRepository,
	projectRepo repository.ProjectRepository,
	taskRepo repository.TaskRepository,
	logger *slog.Logger,
) *DashboardHandler {
	return &DashboardHandler{
		userRepo:    userRepo,
		projectRepo: projectRepo,
		taskRepo:    taskRepo,
		logger:      logger,
	}
}

// DashboardStats holds the aggregated dashboard data.
type DashboardStats struct {
	TotalUsers    int `json:"totalUsers"`
	TotalProjects int `json:"totalProjects"`
	TotalTasks    int `json:"totalTasks"`
	OverdueTasks  int `json:"overdueTasks"`
}

// Overview returns aggregated statistics using parallel queries.
func (h *DashboardHandler) Overview(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	var (
		stats DashboardStats
		mu    sync.Mutex
		wg    sync.WaitGroup
		errs  []error
	)

	// Helper to run a query concurrently and collect results
	type queryFunc func(ctx context.Context) (int, error)
	type queryTarget struct {
		fn     queryFunc
		target *int
		name   string
	}

	queries := []queryTarget{
		{fn: h.userRepo.Count, target: &stats.TotalUsers, name: "users"},
		{fn: h.projectRepo.Count, target: &stats.TotalProjects, name: "projects"},
		{fn: h.taskRepo.Count, target: &stats.TotalTasks, name: "tasks"},
		{fn: h.taskRepo.CountOverdue, target: &stats.OverdueTasks, name: "overdue"},
	}

	wg.Add(len(queries))
	for _, q := range queries {
		go func(q queryTarget) {
			defer wg.Done()
			count, err := q.fn(ctx)
			mu.Lock()
			defer mu.Unlock()
			if err != nil {
				errs = append(errs, err)
				h.logger.Error("dashboard query failed", "query", q.name, "error", err)
			} else {
				*q.target = count
			}
		}(q)
	}

	wg.Wait()

	if len(errs) > 0 {
		response.InternalError(w, "Failed to load some dashboard data")
		return
	}

	response.JSON(w, http.StatusOK, stats)
}
```

### Handler Pattern Summary

Every handler follows the same pattern:

1. **Extract path parameters** -- `r.PathValue("id")`
2. **Extract user from context** -- `middleware.GetUserID(r.Context())`
3. **Parse request body** -- `json.NewDecoder(r.Body).Decode(&req)`
4. **Validate input** -- `validator.Validate(&req)`
5. **Check authorization** -- verify the user has access to the resource
6. **Call repository** -- execute the database operation
7. **Log the action** -- structured logging with relevant IDs
8. **Write response** -- `response.JSON()`, `response.Created()`, `response.NoContent()`

This predictable structure makes handlers easy to read, review, and test.

---

## 12. Structured Logging

We use Go's `log/slog` package (standard library since Go 1.21) for structured, leveled logging throughout the application.

### Logger Setup

```go
package main

import (
	"log/slog"
	"os"
)

// setupLogger creates a structured logger based on configuration.
func setupLogger(level, format string) *slog.Logger {
	var logLevel slog.Level
	switch level {
	case "debug":
		logLevel = slog.LevelDebug
	case "info":
		logLevel = slog.LevelInfo
	case "warn":
		logLevel = slog.LevelWarn
	case "error":
		logLevel = slog.LevelError
	default:
		logLevel = slog.LevelInfo
	}

	opts := &slog.HandlerOptions{
		Level: logLevel,
	}

	var handler slog.Handler
	switch format {
	case "text":
		handler = slog.NewTextHandler(os.Stdout, opts)
	default:
		handler = slog.NewJSONHandler(os.Stdout, opts)
	}

	return slog.New(handler)
}
```

### Log Output Examples

In JSON format (production):

```json
{"time":"2025-03-15T10:30:00.000Z","level":"INFO","msg":"http request","method":"POST","path":"/api/v1/projects","status":201,"duration":"3.2ms","request_id":"a1b2c3d4e5f6"}
{"time":"2025-03-15T10:30:00.003Z","level":"INFO","msg":"project created","project_id":"550e8400-e29b-41d4-a716-446655440000","owner_id":"user-123","name":"My Project"}
```

In text format (development):

```
time=2025-03-15T10:30:00.000Z level=INFO msg="http request" method=POST path=/api/v1/projects status=201 duration=3.2ms request_id=a1b2c3d4e5f6
```

### Why `log/slog` Instead of Third-Party Loggers?

Before Go 1.21, the standard `log` package only supported unstructured text logging. Teams used `zap`, `zerolog`, or `logrus` for structured logging. Now `log/slog` provides:

- **Structured key-value pairs** -- `slog.Info("msg", "key", value)`
- **Log levels** -- Debug, Info, Warn, Error
- **Pluggable handlers** -- JSON, text, or custom (write your own `slog.Handler`)
- **Context integration** -- `logger.LogAttrs(ctx, level, msg, attrs...)`
- **High performance** -- designed to minimize allocations

### Node.js/Express Comparison

In Express, you typically use `winston` or `pino`:

```javascript
// Node.js with pino
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({ userId: '123', projectId: '456' }, 'project created');
// {"level":30,"time":1710500000000,"userId":"123","projectId":"456","msg":"project created"}
```

The Go `slog` API is remarkably similar to `pino`'s API. The main difference is that Go's is built into the standard library.

---

## 13. Concurrency Patterns

### Background Workers

Our API needs background tasks that run independently of HTTP requests. These come from Chapter 31's patterns.

### `internal/worker/cleanup.go`

```go
package worker

import (
	"context"
	"log/slog"
	"time"

	"taskapi/internal/repository"
)

// CleanupWorker runs periodic maintenance tasks in the background.
type CleanupWorker struct {
	taskRepo repository.TaskRepository
	logger   *slog.Logger
	interval time.Duration
	done     chan struct{}
}

// NewCleanupWorker creates a new background cleanup worker.
func NewCleanupWorker(taskRepo repository.TaskRepository, logger *slog.Logger, interval time.Duration) *CleanupWorker {
	return &CleanupWorker{
		taskRepo: taskRepo,
		logger:   logger,
		interval: interval,
		done:     make(chan struct{}),
	}
}

// Start begins the background cleanup loop. It runs until Stop() is called
// or the context is cancelled.
func (w *CleanupWorker) Start(ctx context.Context) {
	w.logger.Info("cleanup worker started", "interval", w.interval)

	ticker := time.NewTicker(w.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			w.logger.Info("cleanup worker stopping (context cancelled)")
			close(w.done)
			return
		case <-ticker.C:
			w.runCleanup(ctx)
		}
	}
}

// Done returns a channel that is closed when the worker has finished.
func (w *CleanupWorker) Done() <-chan struct{} {
	return w.done
}

// runCleanup performs a single cleanup iteration.
func (w *CleanupWorker) runCleanup(ctx context.Context) {
	w.logger.Debug("running cleanup cycle")

	// Mark overdue tasks
	affected, err := w.taskRepo.MarkOverdueTasks(ctx)
	if err != nil {
		w.logger.Error("marking overdue tasks", "error", err)
		return
	}

	if affected > 0 {
		w.logger.Info("marked overdue tasks", "count", affected)
	}
}
```

### Parallel Dashboard Queries (Already Shown in Section 11)

The `DashboardHandler.Overview` method demonstrates the parallel query pattern. Four independent database queries run concurrently using goroutines with a `sync.WaitGroup`, collecting results into a shared struct protected by a `sync.Mutex`.

This pattern is directly from Chapter 31. The key insight: because each query is independent and hits the database over the network, running them in parallel can reduce the total response time from `sum(all query times)` to `max(single query time)`.

```
Sequential:  |--users--||--projects--||--tasks--||--overdue--|  = ~40ms total
Parallel:    |--users--|                                        = ~10ms total
             |--projects--|
             |--tasks--|
             |--overdue--|
```

### Graceful Worker Shutdown

The cleanup worker respects context cancellation. When the application shuts down (Section 14), the main goroutine cancels the context, and the worker's `select` statement picks up `ctx.Done()` and exits cleanly. The `Done()` channel lets the main goroutine wait for the worker to finish its current iteration before proceeding with shutdown.

---

## 14. Graceful Shutdown and Main

The `main.go` file is the entry point that wires everything together. It follows the lifecycle pattern from Chapter 30: initialize, start, wait for signal, shutdown gracefully.

### `main.go`

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"taskapi/internal/auth"
	"taskapi/internal/config"
	"taskapi/internal/database"
	"taskapi/internal/handlers"
	"taskapi/internal/middleware"
	"taskapi/internal/repository"
	"taskapi/internal/router"
	"taskapi/internal/worker"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}

func run() error {
	// -------------------------------------------------------
	// 1. Load configuration
	// -------------------------------------------------------
	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("loading config: %w", err)
	}

	// -------------------------------------------------------
	// 2. Set up structured logger
	// -------------------------------------------------------
	logger := setupLogger(cfg.LogLevel, cfg.LogFormat)
	logger.Info("starting application",
		"environment", cfg.Environment,
		"address", cfg.ServerAddr(),
	)

	// -------------------------------------------------------
	// 3. Connect to database
	// -------------------------------------------------------
	db, err := database.New(cfg.DatabaseDSN(), cfg.DatabaseMaxConns, cfg.DatabaseMinConns, logger)
	if err != nil {
		return fmt.Errorf("connecting to database: %w", err)
	}
	defer db.Close()

	// -------------------------------------------------------
	// 4. Run migrations
	// -------------------------------------------------------
	ctx := context.Background()
	if err := db.RunMigrations(ctx); err != nil {
		return fmt.Errorf("running migrations: %w", err)
	}

	// -------------------------------------------------------
	// 5. Create repositories
	// -------------------------------------------------------
	userRepo := repository.NewUserRepository(db.DB)
	projectRepo := repository.NewProjectRepository(db.DB)
	taskRepo := repository.NewTaskRepository(db.DB)
	commentRepo := repository.NewCommentRepository(db.DB)

	// -------------------------------------------------------
	// 6. Create services
	// -------------------------------------------------------
	jwtService := auth.NewJWTService(cfg.JWTSecret, cfg.JWTExpiration, cfg.JWTRefreshExpiry)

	// -------------------------------------------------------
	// 7. Create handlers
	// -------------------------------------------------------
	healthHandler := handlers.NewHealthHandler(db)
	authHandler := handlers.NewAuthHandler(userRepo, jwtService, logger)
	projectHandler := handlers.NewProjectHandler(projectRepo, logger)
	taskHandler := handlers.NewTaskHandler(taskRepo, projectRepo, logger)
	commentHandler := handlers.NewCommentHandler(commentRepo, taskRepo, projectRepo, logger)
	dashboardHandler := handlers.NewDashboardHandler(userRepo, projectRepo, taskRepo, logger)

	// -------------------------------------------------------
	// 8. Build router
	// -------------------------------------------------------
	appRouter := router.New(router.Config{
		AuthHandler:      authHandler,
		ProjectHandler:   projectHandler,
		TaskHandler:      taskHandler,
		CommentHandler:   commentHandler,
		DashboardHandler: dashboardHandler,
		HealthHandler:    healthHandler,
		JWTService:       jwtService,
	})

	// -------------------------------------------------------
	// 9. Build middleware stack
	// -------------------------------------------------------
	rateLimiter := middleware.NewRateLimiter(cfg.RateLimitRequests, cfg.RateLimitWindow, logger)

	corsConfig := middleware.CORSConfig{
		AllowedOrigins: cfg.CORSAllowedOrigins,
		AllowedMethods: []string{"GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"},
		AllowedHeaders: []string{"Content-Type", "Authorization", "X-Request-ID"},
		MaxAge:         86400,
	}

	stack := middleware.Chain(
		middleware.Recovery(logger),
		middleware.Logging(logger),
		middleware.CORS(corsConfig),
		rateLimiter.Middleware(),
	)

	handler := stack(appRouter)

	// -------------------------------------------------------
	// 10. Create HTTP server
	// -------------------------------------------------------
	server := &http.Server{
		Addr:         cfg.ServerAddr(),
		Handler:      handler,
		ReadTimeout:  cfg.ServerReadTimeout,
		WriteTimeout: cfg.ServerWriteTimeout,
		IdleTimeout:  cfg.ServerIdleTimeout,
	}

	// -------------------------------------------------------
	// 11. Start background workers
	// -------------------------------------------------------
	workerCtx, workerCancel := context.WithCancel(context.Background())
	defer workerCancel()

	cleanupWorker := worker.NewCleanupWorker(taskRepo, logger, 5*time.Minute)
	go cleanupWorker.Start(workerCtx)

	// -------------------------------------------------------
	// 12. Start server in a goroutine
	// -------------------------------------------------------
	serverErrors := make(chan error, 1)
	go func() {
		logger.Info("server listening", "address", cfg.ServerAddr())
		serverErrors <- server.ListenAndServe()
	}()

	// -------------------------------------------------------
	// 13. Wait for shutdown signal
	// -------------------------------------------------------
	shutdown := make(chan os.Signal, 1)
	signal.Notify(shutdown, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-serverErrors:
		return fmt.Errorf("server error: %w", err)

	case sig := <-shutdown:
		logger.Info("shutdown signal received", "signal", sig.String())

		// Give active requests time to complete
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 15*time.Second)
		defer shutdownCancel()

		// Stop accepting new requests, wait for active ones to finish
		if err := server.Shutdown(shutdownCtx); err != nil {
			// Force close if graceful shutdown times out
			server.Close()
			return fmt.Errorf("graceful shutdown failed: %w", err)
		}

		// Stop background workers
		workerCancel()

		// Wait for cleanup worker to finish its current cycle
		select {
		case <-cleanupWorker.Done():
			logger.Info("cleanup worker stopped")
		case <-time.After(5 * time.Second):
			logger.Warn("cleanup worker did not stop in time")
		}

		logger.Info("server stopped gracefully")
	}

	return nil
}

// setupLogger creates a structured logger based on configuration.
func setupLogger(level, format string) *slog.Logger {
	var logLevel slog.Level
	switch level {
	case "debug":
		logLevel = slog.LevelDebug
	case "info":
		logLevel = slog.LevelInfo
	case "warn":
		logLevel = slog.LevelWarn
	case "error":
		logLevel = slog.LevelError
	default:
		logLevel = slog.LevelInfo
	}

	opts := &slog.HandlerOptions{
		Level: logLevel,
	}

	var handler slog.Handler
	switch format {
	case "text":
		handler = slog.NewTextHandler(os.Stdout, opts)
	default:
		handler = slog.NewJSONHandler(os.Stdout, opts)
	}

	return slog.New(handler)
}
```

### `go.mod`

```
module taskapi

go 1.22

require (
	github.com/lib/pq v1.10.9
	golang.org/x/crypto v0.21.0
)
```

### The `run()` Pattern

Notice that `main()` calls `run()` which returns an error. This is an idiomatic Go pattern:

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```

Why not just put everything in `main()`? Because `main()` cannot return an error. The `run()` function lets us use Go's normal error handling (`return fmt.Errorf(...)`) instead of scattering `log.Fatal()` calls throughout initialization. It also makes the startup sequence testable -- you can call `run()` in a test.

### Shutdown Sequence

The shutdown sequence is critical for a production service:

```
1. Receive SIGINT/SIGTERM signal
2. Log the signal
3. Call server.Shutdown(ctx) with a 15-second timeout
   - Server stops accepting NEW connections
   - Server waits for IN-FLIGHT requests to complete
   - If they don't complete in 15 seconds, force-close
4. Cancel the worker context
   - Background workers detect ctx.Done() and exit
5. Wait for workers to finish (with 5-second timeout)
6. Close database connections (deferred)
7. Exit cleanly
```

This ensures:
- No in-flight requests are dropped (they complete within the timeout)
- No background workers are in the middle of a database operation when the pool closes
- The database connection pool is the last thing closed (since everything else depends on it)

### Node.js/Express Comparison

Express does not have built-in graceful shutdown. You typically write:

```javascript
// Express graceful shutdown (manual)
process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  server.close(() => {       // Stop accepting new connections
    db.end();                 // Close database pool
    process.exit(0);
  });
  setTimeout(() => {          // Force-kill after timeout
    process.exit(1);
  }, 15000);
});
```

Go's `server.Shutdown()` is more robust because it tracks active connections internally and waits for them to complete, whereas Express's `server.close()` just stops accepting new connections.

---

## 15. Testing the API

Testing is crucial. We demonstrate three types: unit tests for handlers (with mock repositories), integration tests that test the full HTTP stack, and repository tests (if a database is available).

### Unit Testing Handlers with Mock Repositories

Since our repositories are interfaces, we can create mock implementations for testing without a database.

```go
package handlers_test

import (
	"bytes"
	"context"
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"

	"taskapi/internal/auth"
	"taskapi/internal/handlers"
	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/repository"
)

// --- Mock UserRepository ---

type mockUserRepo struct {
	users  map[string]*models.User
	byEmail map[string]*models.User
}

func newMockUserRepo() *mockUserRepo {
	return &mockUserRepo{
		users:   make(map[string]*models.User),
		byEmail: make(map[string]*models.User),
	}
}

func (m *mockUserRepo) Create(ctx context.Context, user *models.User) error {
	user.ID = "test-user-id"
	user.IsActive = true
	m.users[user.ID] = user
	m.byEmail[user.Email] = user
	return nil
}

func (m *mockUserRepo) GetByID(ctx context.Context, id string) (*models.User, error) {
	return m.users[id], nil
}

func (m *mockUserRepo) GetByEmail(ctx context.Context, email string) (*models.User, error) {
	return m.byEmail[email], nil
}

func (m *mockUserRepo) Update(ctx context.Context, user *models.User) error {
	m.users[user.ID] = user
	return nil
}

func (m *mockUserRepo) Delete(ctx context.Context, id string) error {
	delete(m.users, id)
	return nil
}

func (m *mockUserRepo) List(ctx context.Context, params models.PaginationParams) ([]models.User, int, error) {
	var users []models.User
	for _, u := range m.users {
		users = append(users, *u)
	}
	return users, len(users), nil
}

func (m *mockUserRepo) Count(ctx context.Context) (int, error) {
	return len(m.users), nil
}

// Compile-time check
var _ repository.UserRepository = (*mockUserRepo)(nil)

// --- Mock ProjectRepository ---

type mockProjectRepo struct {
	projects map[string]*models.Project
	members  map[string]map[string]bool // projectID -> userID -> exists
}

func newMockProjectRepo() *mockProjectRepo {
	return &mockProjectRepo{
		projects: make(map[string]*models.Project),
		members:  make(map[string]map[string]bool),
	}
}

func (m *mockProjectRepo) Create(ctx context.Context, project *models.Project) error {
	project.ID = "test-project-id"
	m.projects[project.ID] = project
	if m.members[project.ID] == nil {
		m.members[project.ID] = make(map[string]bool)
	}
	m.members[project.ID][project.OwnerID] = true
	return nil
}

func (m *mockProjectRepo) GetByID(ctx context.Context, id string) (*models.Project, error) {
	return m.projects[id], nil
}

func (m *mockProjectRepo) Update(ctx context.Context, project *models.Project) error {
	m.projects[project.ID] = project
	return nil
}

func (m *mockProjectRepo) Delete(ctx context.Context, id string) error {
	delete(m.projects, id)
	return nil
}

func (m *mockProjectRepo) ListByUser(ctx context.Context, userID string, params models.PaginationParams) ([]models.Project, int, error) {
	var projects []models.Project
	for id, members := range m.members {
		if members[userID] {
			if p, ok := m.projects[id]; ok {
				projects = append(projects, *p)
			}
		}
	}
	return projects, len(projects), nil
}

func (m *mockProjectRepo) AddMember(ctx context.Context, projectID, userID, role string) error {
	if m.members[projectID] == nil {
		m.members[projectID] = make(map[string]bool)
	}
	m.members[projectID][userID] = true
	return nil
}

func (m *mockProjectRepo) IsMember(ctx context.Context, projectID, userID string) (bool, error) {
	if m.members[projectID] == nil {
		return false, nil
	}
	return m.members[projectID][userID], nil
}

func (m *mockProjectRepo) Count(ctx context.Context) (int, error) {
	return len(m.projects), nil
}

var _ repository.ProjectRepository = (*mockProjectRepo)(nil)

// --- Tests ---

func TestRegister(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-that-is-at-least-32-chars-long",
		15*60*1e9, // 15 minutes
		7*24*60*60*1e9, // 7 days
	)

	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	body := models.CreateUserRequest{
		Email:    "test@example.com",
		Password: "securepassword123",
		FullName: "Test User",
	}
	bodyJSON, _ := json.Marshal(body)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	handler.Register(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("expected status 201, got %d: %s", rec.Code, rec.Body.String())
	}

	var resp map[string]interface{}
	json.NewDecoder(rec.Body).Decode(&resp)

	data, ok := resp["data"].(map[string]interface{})
	if !ok {
		t.Fatal("response missing data field")
	}

	if data["accessToken"] == nil {
		t.Error("response missing accessToken")
	}
	if data["refreshToken"] == nil {
		t.Error("response missing refreshToken")
	}
}

func TestRegister_DuplicateEmail(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-that-is-at-least-32-chars-long",
		15*60*1e9,
		7*24*60*60*1e9,
	)

	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	// Register first user
	body := models.CreateUserRequest{
		Email:    "test@example.com",
		Password: "securepassword123",
		FullName: "Test User",
	}
	bodyJSON, _ := json.Marshal(body)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()
	handler.Register(rec, req)

	// Try to register with same email
	req2 := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req2.Header.Set("Content-Type", "application/json")
	rec2 := httptest.NewRecorder()
	handler.Register(rec2, req2)

	if rec2.Code != http.StatusConflict {
		t.Fatalf("expected status 409, got %d", rec2.Code)
	}
}

func TestRegister_ValidationError(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-that-is-at-least-32-chars-long",
		15*60*1e9,
		7*24*60*60*1e9,
	)

	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	// Missing required fields
	body := models.CreateUserRequest{
		Email: "invalid-email", // invalid email format
	}
	bodyJSON, _ := json.Marshal(body)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()
	handler.Register(rec, req)

	if rec.Code != http.StatusBadRequest {
		t.Fatalf("expected status 400, got %d: %s", rec.Code, rec.Body.String())
	}
}

func TestCreateProject(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	projectRepo := newMockProjectRepo()

	handler := handlers.NewProjectHandler(projectRepo, logger)

	body := models.CreateProjectRequest{
		Name: "My Test Project",
	}
	bodyJSON, _ := json.Marshal(body)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/projects", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")

	// Simulate authenticated user by setting context values
	ctx := context.WithValue(req.Context(), middleware.UserIDKey, "test-user-id")
	ctx = context.WithValue(ctx, middleware.UserRoleKey, "member")
	req = req.WithContext(ctx)

	rec := httptest.NewRecorder()
	handler.Create(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("expected status 201, got %d: %s", rec.Code, rec.Body.String())
	}

	var resp map[string]interface{}
	json.NewDecoder(rec.Body).Decode(&resp)

	data, ok := resp["data"].(map[string]interface{})
	if !ok {
		t.Fatal("response missing data field")
	}

	if data["name"] != "My Test Project" {
		t.Errorf("expected name 'My Test Project', got %v", data["name"])
	}
}

func TestCreateProject_MissingName(t *testing.T) {
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
	projectRepo := newMockProjectRepo()

	handler := handlers.NewProjectHandler(projectRepo, logger)

	body := models.CreateProjectRequest{
		// Name is missing
	}
	bodyJSON, _ := json.Marshal(body)

	req := httptest.NewRequest(http.MethodPost, "/api/v1/projects", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")

	ctx := context.WithValue(req.Context(), middleware.UserIDKey, "test-user-id")
	req = req.WithContext(ctx)

	rec := httptest.NewRecorder()
	handler.Create(rec, req)

	if rec.Code != http.StatusBadRequest {
		t.Fatalf("expected status 400, got %d: %s", rec.Code, rec.Body.String())
	}
}
```

### Integration Test (Full HTTP Stack)

```go
package integration_test

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"taskapi/internal/auth"
	"taskapi/internal/handlers"
	"taskapi/internal/middleware"
	"taskapi/internal/models"
	"taskapi/internal/router"
)

// This test exercises the full HTTP stack: router → middleware → handler
func TestFullStack_RegisterAndLogin(t *testing.T) {
	// Skip if no test database
	// In a real project, you'd use a test database or docker-compose
	t.Skip("requires database - run with: go test -tags integration ./tests/integration/")

	// This demonstrates the pattern even if we can't run it here:
	//
	// 1. Set up a test database (use testcontainers or docker-compose)
	// 2. Run migrations
	// 3. Create real repositories pointing to test DB
	// 4. Build the full router
	// 5. Make HTTP requests against httptest.NewServer
	// 6. Assert responses
	// 7. Clean up test data
}

// TestRouterIntegration tests the router with mock handlers.
func TestRouterIntegration(t *testing.T) {
	// Create mock dependencies
	userRepo := newMockUserRepo()
	projectRepo := newMockProjectRepo()
	taskRepo := newMockTaskRepo()
	commentRepo := newMockCommentRepo()

	jwtService := auth.NewJWTService(
		"integration-test-secret-that-is-long-enough",
		15*60*1e9,
		7*24*60*60*1e9,
	)

	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	// Create real handlers with mock repos
	authHandler := handlers.NewAuthHandler(userRepo, jwtService, logger)
	projectHandler := handlers.NewProjectHandler(projectRepo, logger)
	taskHandler := handlers.NewTaskHandler(taskRepo, projectRepo, logger)
	commentHandler := handlers.NewCommentHandler(commentRepo, taskRepo, projectRepo, logger)
	dashboardHandler := handlers.NewDashboardHandler(userRepo, projectRepo, taskRepo, logger)
	healthHandler := handlers.NewHealthHandler(nil) // nil DB for unit test

	// Build router
	appRouter := router.New(router.Config{
		AuthHandler:      authHandler,
		ProjectHandler:   projectHandler,
		TaskHandler:      taskHandler,
		CommentHandler:   commentHandler,
		DashboardHandler: dashboardHandler,
		HealthHandler:    healthHandler,
		JWTService:       jwtService,
	})

	// Apply middleware
	stack := middleware.Chain(
		middleware.Recovery(logger),
		middleware.Logging(logger),
	)
	handler := stack(appRouter)

	// Create test server
	server := httptest.NewServer(handler)
	defer server.Close()

	// Test health endpoint
	t.Run("health check", func(t *testing.T) {
		resp, err := http.Get(server.URL + "/health")
		if err != nil {
			t.Fatal(err)
		}
		defer resp.Body.Close()

		if resp.StatusCode != http.StatusOK {
			t.Errorf("expected 200, got %d", resp.StatusCode)
		}
	})

	// Test registration
	t.Run("register user", func(t *testing.T) {
		body, _ := json.Marshal(models.CreateUserRequest{
			Email:    "integration@test.com",
			Password: "password123",
			FullName: "Integration Test",
		})

		resp, err := http.Post(server.URL+"/api/v1/auth/register", "application/json", bytes.NewReader(body))
		if err != nil {
			t.Fatal(err)
		}
		defer resp.Body.Close()

		if resp.StatusCode != http.StatusCreated {
			t.Errorf("expected 201, got %d", resp.StatusCode)
		}
	})

	// Test unauthenticated access
	t.Run("unauthenticated access denied", func(t *testing.T) {
		resp, err := http.Get(server.URL + "/api/v1/projects")
		if err != nil {
			t.Fatal(err)
		}
		defer resp.Body.Close()

		if resp.StatusCode != http.StatusUnauthorized {
			t.Errorf("expected 401, got %d", resp.StatusCode)
		}
	})
}
```

### Testing Strategy Summary

| Test Type | What It Tests | Speed | Database Required |
|-----------|--------------|-------|-------------------|
| **Unit tests** | Single handler with mock repos | Fast (ms) | No |
| **Integration tests** | Full HTTP stack with router + middleware | Fast (ms) | Optional |
| **Database tests** | Repositories against real PostgreSQL | Slow (s) | Yes |
| **End-to-end tests** | Full system including Docker | Slowest (min) | Yes |

Run the fast tests in CI on every push. Run the slow tests before releases.

```bash
# Run all unit tests
go test ./internal/...

# Run with race detector
go test -race ./internal/...

# Run with coverage
go test -cover ./internal/...

# Run integration tests (requires DB)
go test -tags integration ./tests/integration/

# Run benchmarks
go test -bench=. ./internal/...
```

---

## 16. Docker Deployment

### `Dockerfile`

A multi-stage Docker build that produces a minimal container.

```dockerfile
# ==========================================
# Stage 1: Build the Go binary
# ==========================================
FROM golang:1.22-alpine AS builder

# Install git (needed for go mod download) and ca-certificates
RUN apk add --no-cache git ca-certificates

# Set working directory
WORKDIR /app

# Copy module files first for layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build the binary
# CGO_ENABLED=0 produces a static binary (no C dependencies)
# -ldflags="-s -w" strips debug info, reducing binary size by ~30%
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w" \
    -o /app/taskapi \
    .

# ==========================================
# Stage 2: Create minimal runtime image
# ==========================================
FROM alpine:3.19

# Install ca-certificates for HTTPS and timezone data
RUN apk add --no-cache ca-certificates tzdata

# Create a non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/taskapi .

# Use non-root user
USER appuser

# Expose the default port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -q --spider http://localhost:8080/health || exit 1

# Run the binary
ENTRYPOINT ["./taskapi"]
```

### `docker-compose.yml`

For local development with PostgreSQL:

```yaml
version: "3.9"

services:
  # PostgreSQL database
  postgres:
    image: postgres:16-alpine
    container_name: taskapi-db
    environment:
      POSTGRES_USER: taskapi
      POSTGRES_PASSWORD: taskapi_secret
      POSTGRES_DB: taskapi
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapi"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Task API application
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: taskapi
    ports:
      - "8080:8080"
    environment:
      ENVIRONMENT: development
      SERVER_HOST: "0.0.0.0"
      SERVER_PORT: "8080"
      DB_HOST: postgres
      DB_PORT: "5432"
      DB_USER: taskapi
      DB_PASSWORD: taskapi_secret
      DB_NAME: taskapi
      DB_SSL_MODE: disable
      JWT_SECRET: "your-development-secret-that-is-at-least-32-characters-long"
      LOG_LEVEL: debug
      LOG_FORMAT: text
      RATE_LIMIT_REQUESTS: "200"
      CORS_ALLOWED_ORIGINS: "http://localhost:3000,http://localhost:5173"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

### Production Docker Compose with Traefik

For production, add a reverse proxy with automatic TLS:

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    command:
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@yourdomain.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"
    restart: always

  postgres:
    image: postgres:16-alpine
    container_name: taskapi-db-prod
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata-prod:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always

  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: taskapi-prod
    environment:
      ENVIRONMENT: production
      SERVER_HOST: "0.0.0.0"
      SERVER_PORT: "8080"
      DB_HOST: postgres
      DB_PORT: "5432"
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
      DB_SSL_MODE: disable
      DB_MAX_CONNS: "50"
      JWT_SECRET: ${JWT_SECRET}
      LOG_LEVEL: info
      LOG_FORMAT: json
      RATE_LIMIT_REQUESTS: "100"
      CORS_ALLOWED_ORIGINS: "https://yourdomain.com"
    depends_on:
      postgres:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.yourdomain.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.services.api.loadbalancer.server.port=8080"
    restart: always

volumes:
  pgdata-prod:
  letsencrypt:
```

### Docker Image Size

```
$ docker images taskapi
REPOSITORY   TAG      SIZE
taskapi      latest   18MB
```

The final image is ~18MB. Compare this with a Node.js application:

```
$ docker images node-api
REPOSITORY   TAG      SIZE
node-api     latest   350MB
```

Go produces a single static binary (~12MB after stripping). The Alpine base adds ~6MB for ca-certificates and timezone data. Node.js requires the entire Node.js runtime, npm, and node_modules.

### Development Workflow

```bash
# Start everything
docker-compose up -d

# View logs
docker-compose logs -f api

# Run migrations (happens automatically on startup)
# But you can also do it manually:
docker-compose exec api ./taskapi migrate

# Stop everything
docker-compose down

# Stop and remove data
docker-compose down -v
```

---

## 17. API Documentation

### Endpoint Reference

#### Authentication

| Endpoint | Method | Auth | Request Body | Response |
|----------|--------|------|-------------|----------|
| `/api/v1/auth/register` | POST | No | `{ email, password, fullName }` | `{ accessToken, refreshToken, expiresIn, user }` |
| `/api/v1/auth/login` | POST | No | `{ email, password }` | `{ accessToken, refreshToken, expiresIn, user }` |
| `/api/v1/auth/refresh` | POST | No | `{ refreshToken }` | `{ accessToken, refreshToken, expiresIn, user }` |
| `/api/v1/auth/me` | GET | Yes | - | `User` object |

#### Projects

| Endpoint | Method | Auth | Query Params | Request Body | Response |
|----------|--------|------|-------------|-------------|----------|
| `/api/v1/projects` | GET | Yes | `page`, `perPage` | - | `[Project]` + pagination |
| `/api/v1/projects` | POST | Yes | - | `{ name, description? }` | `Project` |
| `/api/v1/projects/{id}` | GET | Yes | - | - | `Project` |
| `/api/v1/projects/{id}` | PUT | Yes | - | `{ name?, description?, isArchived? }` | `Project` |
| `/api/v1/projects/{id}` | DELETE | Yes | - | - | 204 No Content |

#### Tasks

| Endpoint | Method | Auth | Query Params | Request Body | Response |
|----------|--------|------|-------------|-------------|----------|
| `/api/v1/projects/{projectId}/tasks` | GET | Yes | `page`, `perPage`, `status`, `priority`, `assigneeId`, `search` | - | `[Task]` + pagination |
| `/api/v1/projects/{projectId}/tasks` | POST | Yes | - | `{ title, description?, status?, priority?, assigneeId?, dueDate? }` | `Task` |
| `/api/v1/tasks/{id}` | GET | Yes | - | - | `Task` |
| `/api/v1/tasks/{id}` | PUT | Yes | - | `{ title?, description?, status?, priority?, assigneeId?, dueDate? }` | `Task` |
| `/api/v1/tasks/{id}` | DELETE | Yes | - | - | 204 No Content |

#### Comments

| Endpoint | Method | Auth | Query Params | Request Body | Response |
|----------|--------|------|-------------|-------------|----------|
| `/api/v1/tasks/{taskId}/comments` | GET | Yes | `page`, `perPage` | - | `[Comment]` + pagination |
| `/api/v1/tasks/{taskId}/comments` | POST | Yes | - | `{ body }` | `Comment` |
| `/api/v1/comments/{id}` | PUT | Yes | - | `{ body }` | `Comment` |
| `/api/v1/comments/{id}` | DELETE | Yes | - | - | 204 No Content |

#### Dashboard

| Endpoint | Method | Auth | Response |
|----------|--------|------|----------|
| `/api/v1/dashboard` | GET | Yes | `{ totalUsers, totalProjects, totalTasks, overdueTasks }` |

#### Health

| Endpoint | Method | Auth | Response |
|----------|--------|------|----------|
| `/health` | GET | No | `{ status: "ok" }` |
| `/ready` | GET | No | `{ status: "ready" }` or 503 |

### Example API Calls

```bash
# Register a new user
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "securepassword123",
    "fullName": "Alice Johnson"
  }'

# Login
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "securepassword123"
  }'

# Save the access token
TOKEN="eyJhbGci..."

# Create a project
curl -X POST http://localhost:8080/api/v1/projects \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Website Redesign",
    "description": "Complete redesign of the company website"
  }'

# List projects (with pagination)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/v1/projects?page=1&perPage=10"

# Create a task
curl -X POST http://localhost:8080/api/v1/projects/PROJECT_ID/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Design homepage mockup",
    "description": "Create wireframes and high-fidelity mockups for the new homepage",
    "priority": "high",
    "status": "todo",
    "dueDate": "2025-04-01T00:00:00Z"
  }'

# List tasks with filters
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/v1/projects/PROJECT_ID/tasks?status=todo&priority=high&page=1"

# Search tasks
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/api/v1/projects/PROJECT_ID/tasks?search=homepage"

# Update task status
curl -X PUT http://localhost:8080/api/v1/tasks/TASK_ID \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"status": "in_progress"}'

# Add a comment
curl -X POST http://localhost:8080/api/v1/tasks/TASK_ID/comments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"body": "Started working on this. Will have mockups ready by Friday."}'

# Get dashboard stats
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/dashboard

# Refresh token
curl -X POST http://localhost:8080/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "eyJhbGci..."}'
```

---

## 18. What We Have Built

Let us step back and look at what we have accomplished. This single chapter ties together every pattern from Chapters 23-31:

### Chapter-by-Chapter Integration

| Chapter | Pattern | Where We Used It |
|---------|---------|-----------------|
| **Ch.23 -- Configuration** | Environment-based config with validation | `internal/config/config.go` -- All settings loaded from env vars, validated at startup, typed correctly |
| **Ch.24 -- Routing** | Method + path routing with parameters | `internal/router/router.go` -- Go 1.22 ServeMux with `{id}` path parameters, method prefixes |
| **Ch.25 -- Middleware** | Chain of cross-cutting concerns | `internal/middleware/` -- Recovery, Logging, CORS, Rate Limiting, Auth -- all composable `func(http.Handler) http.Handler` |
| **Ch.26 -- Database** | Connection pool, migrations, repositories | `internal/database/` -- Pool tuning, embedded SQL migrations, interface-based repos with parameterized queries |
| **Ch.27 -- Authentication** | JWT + bcrypt, auth middleware | `internal/auth/jwt.go` -- Token generation, validation, refresh rotation. `internal/middleware/auth.go` -- context injection |
| **Ch.28 -- Logging** | Structured, leveled, context-aware | `log/slog` throughout -- JSON/text format, request IDs, log levels, key-value pairs |
| **Ch.29 -- Validation** | Struct tag validation, error responses | `pkg/validator/` -- Custom validation engine. `pkg/response/` -- Consistent API response format |
| **Ch.30 -- Graceful Shutdown** | Signal handling, connection draining | `main.go` -- SIGINT/SIGTERM → Shutdown(ctx) → worker cancel → DB close |
| **Ch.31 -- Concurrency** | Background workers, parallel queries | `internal/worker/cleanup.go` -- Periodic cleanup. `DashboardHandler` -- 4 parallel DB queries |

### Architecture Principles Applied

**1. Dependency injection via interfaces.** Every handler receives its dependencies through constructor functions. Repositories are interfaces. This makes the entire system testable without a database.

**2. Separation of concerns.** Config knows nothing about HTTP. Repositories know nothing about JSON. Handlers know nothing about SQL. Each layer has a single responsibility.

**3. Fail fast.** Invalid configuration is caught at startup, not at request time. Missing database connections surface immediately, not when the first user tries to log in.

**4. Consistent error handling.** Every error follows the same path: log it, return it, respond to the client with a structured error. No silent failures, no uncaught exceptions.

**5. Defense in depth.** Recovery middleware catches panics. Rate limiting prevents abuse. Auth middleware protects routes. Input validation prevents bad data. SQL parameterization prevents injection. CORS restricts origins.

### What This Would Look Like in Node.js/Express

To build the equivalent in Express, you would need these npm packages (at minimum):

```json
{
  "dependencies": {
    "express": "^4.18.0",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "morgan": "^1.10.0",
    "express-rate-limit": "^7.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3",
    "joi": "^17.9.0",
    "pg": "^8.11.0",
    "knex": "^3.0.0",
    "dotenv": "^16.3.0",
    "uuid": "^9.0.0",
    "pino": "^8.15.0",
    "pino-http": "^9.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0",
    "nodemon": "^3.0.0"
  }
}
```

That is 15 runtime dependencies and 3 dev dependencies. Each has its own transitive dependencies, totaling hundreds of packages in `node_modules`. Each needs to be audited for security, updated for compatibility, and trusted not to inject malicious code.

Our Go project has **2 dependencies**: `lib/pq` (PostgreSQL driver) and `golang.org/x/crypto` (bcrypt). Everything else is the standard library.

This is not a knock on Node.js -- Express's ecosystem is rich and productive. But Go's approach of shipping comprehensive standard libraries means fewer dependencies, smaller attack surface, and simpler auditing.

---

## 19. Key Takeaways

1. **Layered architecture keeps code organized.** Config → Database → Repository → Service → Handler → Router → Middleware → Main. Each layer has a single responsibility and depends only on the layer below it.

2. **Interfaces enable testing.** Define repository interfaces and inject them into handlers. In tests, provide mock implementations. This eliminates the need for a database in unit tests.

3. **Fail fast on configuration errors.** Validate all configuration at startup. A missing JWT secret should crash the application immediately, not cause mysterious 500 errors hours later.

4. **Middleware is the Go pattern for cross-cutting concerns.** Recovery, logging, CORS, rate limiting, and authentication are all `func(http.Handler) http.Handler`. They compose cleanly and are reusable across projects.

5. **Go 1.22's enhanced ServeMux eliminates the need for third-party routers** in most applications. `mux.HandleFunc("GET /users/{id}", handler)` with `r.PathValue("id")` covers the vast majority of routing needs.

6. **Graceful shutdown is non-negotiable for production.** Handle SIGINT/SIGTERM, drain active connections, stop background workers, close the database pool -- in that order.

7. **Structured logging with `log/slog` is production-ready** in the standard library. JSON format for machines, text format for humans. Add request IDs to every log line for traceability.

8. **Concurrency should be used where it provides real benefit.** The dashboard handler runs 4 parallel queries because they are independent I/O operations. Do not add goroutines to CPU-bound sequential work.

9. **Consistent API responses build trust.** Every response has the same shape: `{ success, data, error, meta }`. Clients always know what to expect.

10. **Two dependencies is enough.** The Go standard library provides HTTP servers, JSON handling, cryptography, SQL access, testing, and structured logging. You need the PostgreSQL driver and bcrypt. That is it.

11. **Docker multi-stage builds produce minimal images.** An 18MB container with a static Go binary versus a 350MB container with Node.js runtime and node_modules. Faster deployments, less memory, smaller attack surface.

12. **The `run()` pattern makes `main()` testable.** Move all initialization logic into a function that returns an error, and keep `main()` as a thin wrapper that calls it.

---

## 20. Practice Exercises

### Exercise 1: Add WebSocket Notifications

Extend the API with real-time notifications using WebSocket:

- When a task is created, updated, or assigned, notify all members of the project
- Use `golang.org/x/net/websocket` or `github.com/gorilla/websocket`
- Add a `/api/v1/ws` endpoint that upgrades to WebSocket after JWT auth
- Maintain a hub of active connections per user
- When a task event occurs, broadcast to connected project members

**Hint:** Create a `hub` struct with a map of `userID → []*websocket.Conn`, protected by a `sync.RWMutex`. Handlers publish events to a channel, and a goroutine reads from the channel and broadcasts.

### Exercise 2: File Uploads for Task Attachments

Add file attachment support:

- `POST /api/v1/tasks/{id}/attachments` -- upload a file (multipart/form-data)
- `GET /api/v1/tasks/{id}/attachments` -- list attachments
- `GET /api/v1/attachments/{id}/download` -- download a file
- `DELETE /api/v1/attachments/{id}` -- delete an attachment
- Store files in a local directory or S3-compatible storage
- Add a new `attachments` migration and repository
- Limit file size to 10MB, restrict allowed MIME types

**Hint:** Use `r.ParseMultipartForm(10 << 20)` and `r.FormFile("file")` to handle uploads. Store the file metadata in PostgreSQL and the file content on disk or S3.

### Exercise 3: Full-Text Search

Enhance the task search with PostgreSQL full-text search:

- Add `to_tsvector` index on task `title` and `description`
- Create a `GET /api/v1/search?q=keyword` endpoint that searches across all the user's tasks
- Return results ranked by relevance using `ts_rank`
- Highlight matching terms in the results using `ts_headline`

**Hint:** The migration in section 4 already creates a GIN index on `to_tsvector('english', title)`. Extend it to include description and use `plainto_tsquery` for the search.

### Exercise 4: Redis Caching Layer

Add a caching layer using Redis:

- Cache dashboard stats for 30 seconds (they rarely change and are expensive to compute)
- Cache user profiles for 5 minutes
- Invalidate caches when underlying data changes
- Create a `CacheRepository` interface with `Get`, `Set`, `Delete` methods
- Implement it with `github.com/redis/go-redis/v9`
- Add Redis to `docker-compose.yml`

**Hint:** Create a `cache` package with a generic `Get[T any](ctx, key) (*T, error)` function that deserializes cached JSON.

### Exercise 5: API Versioning and Deprecation

Add API version management:

- Support `v1` and `v2` simultaneously
- `v2` returns a different task format (e.g., `status` becomes `state` with different enum values)
- Add a `Deprecation` header to `v1` responses: `Deprecation: true`
- Add a `Sunset` header: `Sunset: Sat, 01 Jan 2026 00:00:00 GMT`
- Create a version negotiation middleware that reads the `Accept-Version` header
- Write an adapter that converts v1 requests/responses to v2 format

### Exercise 6: Comprehensive Test Suite

Build a thorough test suite:

- Unit tests for every handler (positive and negative cases)
- Unit tests for the JWT service (expired tokens, invalid signatures, wrong type)
- Unit tests for the validator (every rule: required, email, min, max, oneof)
- Integration tests using `testcontainers-go` to spin up a real PostgreSQL in Docker
- Load tests using `go test -bench` that measure handler throughput
- Add test coverage reporting and ensure >80% coverage

**Hint:** For testcontainers, use:
```go
import "github.com/testcontainers/testcontainers-go/modules/postgres"

container, err := postgres.RunContainer(ctx,
    testcontainers.WithImage("postgres:16-alpine"),
    postgres.WithDatabase("testdb"),
    postgres.WithUsername("test"),
    postgres.WithPassword("test"),
)
```

### Exercise 7: Metrics and Observability

Add Prometheus metrics:

- Request count by method, path, and status code
- Request duration histogram
- Active connections gauge
- Database connection pool stats
- Background worker iteration count
- Expose metrics at `/metrics` endpoint
- Create a Grafana dashboard JSON that visualizes the metrics

**Hint:** Use `github.com/prometheus/client_golang/prometheus` to create metrics and `promhttp.Handler()` for the endpoint.

### Exercise 8: Role-Based Access Control (RBAC)

Implement fine-grained permissions:

- Define permissions: `project:create`, `project:read`, `project:update`, `project:delete`, `task:create`, etc.
- Roles map to permission sets: admin gets all, member gets create/read/update
- Create a `RequirePermission("task:create")` middleware
- Add project-level roles (owner, admin, member) with different permissions per project
- Ensure the owner cannot be removed from their own project

---

## Congratulations

You have reached the end of this Go study guide. Over 32 chapters, you have gone from "Hello, World" to building a complete production API with authentication, pagination, background workers, graceful shutdown, Docker deployment, and structured logging.

The fundamentals you learned in Chapters 1-22 gave you the language. The patterns in Chapters 23-31 gave you the tools. This chapter showed you how they all fit together.

Go was designed for building exactly this kind of software: reliable, efficient, maintainable services that a team can understand, test, and deploy with confidence. Every pattern in this chapter exists because experienced Go developers learned (often the hard way) that these patterns prevent bugs, simplify debugging, and make systems resilient.

Build something real. Deploy it. Watch the logs. Fix the bugs. That is how the knowledge solidifies.

---

*This is Chapter 32, the final chapter of the Go Complete Study Guide.*
