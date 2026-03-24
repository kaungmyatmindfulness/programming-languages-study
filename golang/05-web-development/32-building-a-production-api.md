# Chapter 32: Building a Production API — Putting It All Together

## Prerequisites

You should have completed (or be comfortable with) every chapter in this guide up to this point. This is the capstone chapter -- it assumes mastery of Go fundamentals (Chapters 1-22) and the real-world patterns introduced in Chapters 23-31. Specifically, you will use:

- **Chapter 23** -- Configuration management (environment variables, config structs, validation)
- **Chapter 24** -- HTTP routing (custom routers, path parameters, method-based dispatch)
- **Chapter 25** -- Middleware patterns (chaining, recovery, logging, CORS, rate limiting)
- **Chapter 26** -- Repository pattern and database (connection pools, repositories, migrations, transactions)
- **Chapter 27** -- Authentication and authorization (JWT, bcrypt, token refresh, middleware)
- **Chapter 28** -- Structured logging (log/slog, request IDs, log levels, context-aware logging)
- **Chapter 29** -- Request validation and API responses (validation tags, custom validators, error responses)
- **Chapter 30** -- Graceful shutdown (signal handling, connection draining, cleanup)
- **Chapter 31** -- Concurrency in web applications (background workers, parallel queries, worker pools)

If you are coming from Node.js/Express, this chapter is a complete translation of how you would build a production Express API, but in idiomatic Go.

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

We are building a **Task Management API** -- a simplified project tracker similar to Jira, Linear, or Asana. This is not a toy example. It is a complete, production-grade REST API that demonstrates every pattern from Chapters 23-31 working together in a single codebase.

The API manages three core resources:

- **Users** -- Registration, authentication, profile management
- **Projects** -- Containers for tasks, owned by users
- **Tasks** -- Work items within projects, with status, priority, assignment, and due dates

### Feature Set

| Category | Features |
|----------|----------|
| **Authentication** | Registration, login with JWT, token refresh, password hashing with bcrypt |
| **Projects** | Create, read, update, delete. List with pagination and filtering |
| **Tasks** | Full CRUD. Filter by status, priority, assignee. Pagination |
| **Dashboard** | Aggregated stats computed with parallel queries |
| **Background Jobs** | Periodic cleanup of overdue tasks |
| **Observability** | Structured logging, request IDs, timing |
| **Resilience** | Panic recovery, rate limiting, graceful shutdown, connection draining |

### Architecture

The system follows a layered architecture. Each layer depends only on the layer directly below it, and dependencies flow inward:

```
Request --> Middleware --> Router --> Handler --> Auth/Validation --> Repository --> Database
```

In more detail:

```
+-----------------------------------------------------------+
|                        main.go                            |
|  (wire everything together, start server, handle          |
|   signals, graceful shutdown)                             |
+-----------------------------------------------------------+
|                     Middleware Stack                       |
|  Recovery -> Logging -> CORS -> Rate Limit -> Auth        |
+-----------------------------------------------------------+
|                         Router                            |
|  Maps HTTP method + path -> Handler function              |
+-----------------------------------------------------------+
|                        Handlers                           |
|  Parse request, call repository, write response           |
+-----------------------------------------------------------+
|                   Auth / Validation                       |
|  JWT, bcrypt, input validation                            |
+-----------------------------------------------------------+
|                      Repositories                         |
|  SQL queries, data access                                 |
+-----------------------------------------------------------+
|                       Database                            |
|  Connection pool, migrations                              |
+-----------------------------------------------------------+
|                      Configuration                        |
|  Environment variables, defaults, validation              |
+-----------------------------------------------------------+
```

### Go vs Node.js/Express Architecture Comparison

The layered architecture maps directly to a well-structured Express application:

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

The critical difference: in Express, you install 15-20 npm packages (`express`, `cors`, `helmet`, `morgan`, `jsonwebtoken`, `bcrypt`, `joi`, `pg`, `knex`, `dotenv`, `express-rate-limit`, ...). In Go, the standard library covers most of these needs, and you install only a handful of focused packages (`lib/pq` for PostgreSQL, `golang.org/x/crypto` for bcrypt).

---

## 2. Project Structure

### Idiomatic Go Project Layout

```
taskapi/
├── main.go                          # Entry point: wiring, server start, graceful shutdown
├── go.mod                           # Module definition
├── go.sum                           # Dependency checksums
├── .env                             # Local dev environment variables (not committed)
├── Dockerfile                       # Multi-stage Docker build
├── docker-compose.yml               # Local development with PostgreSQL
├── internal/                        # Private application code
│   ├── config/
│   │   └── config.go                # Configuration loading and validation
│   ├── database/
│   │   ├── database.go              # Connection pool setup, health check, close
│   │   └── migrations/
│   │       ├── 001_create_users.sql
│   │       ├── 002_create_projects.sql
│   │       └── 003_create_tasks.sql
│   ├── models/
│   │   ├── user.go                  # User domain model
│   │   ├── project.go               # Project domain model
│   │   └── task.go                  # Task domain model
│   ├── repository/
│   │   ├── user_repository.go       # User data access (interface + implementation)
│   │   ├── project_repository.go    # Project data access
│   │   └── task_repository.go       # Task data access
│   ├── handlers/
│   │   ├── auth_handler.go          # Login, register, refresh
│   │   ├── project_handler.go       # Project CRUD endpoints
│   │   ├── task_handler.go          # Task CRUD endpoints
│   │   ├── dashboard_handler.go     # Dashboard aggregation endpoint
│   │   └── health_handler.go        # Health check
│   ├── middleware/
│   │   ├── auth.go                  # JWT authentication
│   │   ├── cors.go                  # Cross-Origin Resource Sharing
│   │   ├── logging.go               # Request/response logging
│   │   ├── ratelimit.go             # Per-IP rate limiting
│   │   └── recovery.go              # Panic recovery
│   ├── router/
│   │   └── router.go                # Route registration
│   ├── auth/
│   │   └── jwt.go                   # JWT token creation and validation
│   └── worker/
│       └── cleanup.go               # Background cleanup goroutines
├── pkg/                             # Public reusable packages
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
└── node_modules/        <-- hundreds of directories
```

The Go version has no `node_modules` equivalent. Dependencies are downloaded to `$GOPATH/pkg/mod` globally and shared across projects. The compiled binary is self-contained.

---

## 3. Configuration Layer

Configuration is the foundation of the application. Every other layer reads from it. We load from environment variables with sensible defaults, validate required fields at startup, and make the config immutable after initialization.

This applies patterns from **Chapter 23**.

### `internal/config/config.go`

```go
package config

import (
	"fmt"
	"os"
	"strconv"
	"strings"
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
	JWTSecret        string
	JWTExpiration    time.Duration
	JWTRefreshExpiry time.Duration

	// Rate Limiting
	RateLimitRequests int
	RateLimitWindow   time.Duration

	// CORS
	CORSAllowedOrigins []string

	// Logging
	LogLevel  string
	LogFormat string

	// Workers
	CleanupInterval time.Duration

	// Environment
	Environment string
}

// Load reads configuration from environment variables with sensible defaults.
// It returns an error if any required field is missing or invalid.
func Load() (*Config, error) {
	cfg := &Config{
		// Server defaults
		ServerHost:         envOrDefault("SERVER_HOST", "0.0.0.0"),
		ServerPort:         envOrDefaultInt("SERVER_PORT", 8080),
		ServerReadTimeout:  envOrDefaultDuration("SERVER_READ_TIMEOUT", 10*time.Second),
		ServerWriteTimeout: envOrDefaultDuration("SERVER_WRITE_TIMEOUT", 30*time.Second),
		ServerIdleTimeout:  envOrDefaultDuration("SERVER_IDLE_TIMEOUT", 60*time.Second),

		// Database defaults
		DatabaseHost:     envOrDefault("DB_HOST", "localhost"),
		DatabasePort:     envOrDefaultInt("DB_PORT", 5432),
		DatabaseUser:     envOrDefault("DB_USER", "postgres"),
		DatabasePassword: os.Getenv("DB_PASSWORD"),
		DatabaseName:     envOrDefault("DB_NAME", "taskapi"),
		DatabaseSSLMode:  envOrDefault("DB_SSL_MODE", "disable"),
		DatabaseMaxConns: envOrDefaultInt("DB_MAX_CONNS", 25),
		DatabaseMinConns: envOrDefaultInt("DB_MIN_CONNS", 5),

		// JWT
		JWTSecret:        os.Getenv("JWT_SECRET"),
		JWTExpiration:    envOrDefaultDuration("JWT_EXPIRATION", 15*time.Minute),
		JWTRefreshExpiry: envOrDefaultDuration("JWT_REFRESH_EXPIRY", 7*24*time.Hour),

		// Rate limiting defaults
		RateLimitRequests: envOrDefaultInt("RATE_LIMIT_REQUESTS", 100),
		RateLimitWindow:   envOrDefaultDuration("RATE_LIMIT_WINDOW", time.Minute),

		// CORS defaults
		CORSAllowedOrigins: strings.Split(envOrDefault("CORS_ORIGINS", "http://localhost:3000"), ","),

		// Logging defaults
		LogLevel:  envOrDefault("LOG_LEVEL", "info"),
		LogFormat: envOrDefault("LOG_FORMAT", "json"),

		// Worker defaults
		CleanupInterval: envOrDefaultDuration("CLEANUP_INTERVAL", 5*time.Minute),

		// Environment
		Environment: envOrDefault("ENVIRONMENT", "development"),
	}

	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}

// DatabaseURL returns the PostgreSQL connection string.
func (c *Config) DatabaseURL() string {
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

// IsProduction returns true if the environment is production.
func (c *Config) IsProduction() bool {
	return c.Environment == "production"
}

// validate checks that all required configuration is present and valid.
func (c *Config) validate() error {
	var missing []string

	if c.JWTSecret == "" {
		missing = append(missing, "JWT_SECRET")
	}
	if c.DatabasePassword == "" && c.IsProduction() {
		missing = append(missing, "DB_PASSWORD")
	}

	if len(missing) > 0 {
		return fmt.Errorf("missing required environment variables: %s", strings.Join(missing, ", "))
	}

	if len(c.JWTSecret) < 32 {
		return fmt.Errorf("JWT_SECRET must be at least 32 characters (got %d)", len(c.JWTSecret))
	}

	if c.DatabaseMaxConns < c.DatabaseMinConns {
		return fmt.Errorf("DB_MAX_CONNS (%d) must be >= DB_MIN_CONNS (%d)",
			c.DatabaseMaxConns, c.DatabaseMinConns)
	}

	validLevels := map[string]bool{"debug": true, "info": true, "warn": true, "error": true}
	if !validLevels[c.LogLevel] {
		return fmt.Errorf("LOG_LEVEL must be one of: debug, info, warn, error (got %q)", c.LogLevel)
	}

	return nil
}

// --- helper functions ---

func envOrDefault(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

func envOrDefaultInt(key string, defaultVal int) int {
	val := os.Getenv(key)
	if val == "" {
		return defaultVal
	}
	n, err := strconv.Atoi(val)
	if err != nil {
		return defaultVal
	}
	return n
}

func envOrDefaultDuration(key string, defaultVal time.Duration) time.Duration {
	val := os.Getenv(key)
	if val == "" {
		return defaultVal
	}
	d, err := time.ParseDuration(val)
	if err != nil {
		return defaultVal
	}
	return d
}
```

### Node.js/Express Comparison

In Express, configuration is usually handled with `dotenv` and manual `process.env` reads:

```javascript
// Node.js
require('dotenv').config();

const config = {
  port: parseInt(process.env.PORT) || 3000,
  dbUrl: process.env.DATABASE_URL,          // no validation unless you add it
  jwtSecret: process.env.JWT_SECRET,        // could be empty -- runtime crash later
  corsOrigins: process.env.CORS_ORIGINS?.split(',') || ['*'],
};

module.exports = config;
```

The Go version validates upfront. If `JWT_SECRET` is missing, the application does not start. In the Node.js version, the missing secret is discovered only when the first JWT operation fails -- potentially hours after deployment.

---

## 4. Database Layer

The database layer handles connection pooling, health checks, and migrations. This applies patterns from **Chapter 26**.

### `internal/database/database.go`

```go
package database

import (
	"context"
	"database/sql"
	"fmt"
	"log/slog"
	"time"

	_ "github.com/lib/pq" // PostgreSQL driver
)

// DB wraps the standard sql.DB with additional capabilities.
type DB struct {
	*sql.DB
	logger *slog.Logger
}

// New creates a new database connection pool.
func New(dsn string, maxConns, minConns int, logger *slog.Logger) (*DB, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("opening database: %w", err)
	}

	// Configure the connection pool
	db.SetMaxOpenConns(maxConns)
	db.SetMaxIdleConns(minConns)
	db.SetConnMaxLifetime(30 * time.Minute)
	db.SetConnMaxIdleTime(5 * time.Minute)

	// Verify connectivity
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, fmt.Errorf("pinging database: %w", err)
	}

	logger.Info("database connected",
		"max_open_conns", maxConns,
		"max_idle_conns", minConns,
	)

	return &DB{DB: db, logger: logger}, nil
}

// Health checks if the database is reachable.
func (d *DB) Health(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	return d.PingContext(ctx)
}

// Close closes the database connection pool.
func (d *DB) Close() error {
	d.logger.Info("closing database connection pool")
	return d.DB.Close()
}
```

### Migration SQL Files

We use plain SQL files applied in order. In production, you might use a tool like `golang-migrate/migrate` or `pressly/goose`. For this project, we apply them at startup.

### `internal/database/migrations/001_create_users.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name   VARCHAR(255) NOT NULL,
    role        VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
CREATE INDEX IF NOT EXISTS idx_users_is_active ON users(is_active);
```

### `internal/database/migrations/002_create_projects.sql`

```sql
CREATE TABLE IF NOT EXISTS projects (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status      VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_projects_owner ON projects(owner_id);
CREATE INDEX IF NOT EXISTS idx_projects_status ON projects(status);
```

### `internal/database/migrations/003_create_tasks.sql`

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title       VARCHAR(500) NOT NULL,
    description TEXT,
    status      VARCHAR(50) NOT NULL DEFAULT 'todo',
    priority    VARCHAR(50) NOT NULL DEFAULT 'medium',
    assignee_id UUID REFERENCES users(id) ON DELETE SET NULL,
    due_date    DATE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_tasks_project ON tasks(project_id);
CREATE INDEX IF NOT EXISTS idx_tasks_assignee ON tasks(assignee_id);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status);
CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority);
CREATE INDEX IF NOT EXISTS idx_tasks_due_date ON tasks(due_date);
```

### Migration Runner

```go
package database

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"path/filepath"
	"sort"
	"strings"
)

// RunMigrations applies all SQL migration files in order.
// In production, use a proper migration tool like golang-migrate/migrate.
func (d *DB) RunMigrations(ctx context.Context, migrationsDir string) error {
	// Read migration files
	entries, err := os.ReadDir(migrationsDir)
	if err != nil {
		return fmt.Errorf("reading migrations directory: %w", err)
	}

	// Sort by filename (001_, 002_, etc.)
	var files []string
	for _, entry := range entries {
		if !entry.IsDir() && strings.HasSuffix(entry.Name(), ".sql") {
			files = append(files, entry.Name())
		}
	}
	sort.Strings(files)

	for _, file := range files {
		path := filepath.Join(migrationsDir, file)
		content, err := os.ReadFile(path)
		if err != nil {
			return fmt.Errorf("reading migration %s: %w", file, err)
		}

		if _, err := d.ExecContext(ctx, string(content)); err != nil {
			return fmt.Errorf("executing migration %s: %w", file, err)
		}

		d.logger.Info("migration applied", "file", file)
	}

	return nil
}
```

### Node.js/Express Comparison

In Express with Knex:

```javascript
// Node.js with Knex
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  pool: { min: 2, max: 10 },
});

// Knex manages migrations with a CLI tool:
// npx knex migrate:latest
```

In Go, `database/sql` is the standard library. The driver (`lib/pq`) registers itself via its `init()` function -- that is what `_ "github.com/lib/pq"` does. Pool configuration is built in. No ORM, no query builder -- you write SQL directly and scan results into structs.

---

## 5. Domain Models

Domain models define the data structures your application works with. We define custom types for enums, attach validation logic, and separate database models from API request/response types.

This applies patterns from **Chapter 29**.

### `internal/models/user.go`

```go
package models

import "time"

// Role represents a user role in the system.
type Role string

const (
	RoleAdmin  Role = "admin"
	RoleMember Role = "member"
)

// User represents a user in the system.
type User struct {
	ID           string    `json:"id"`
	Email        string    `json:"email"`
	PasswordHash string    `json:"-"` // Never expose in JSON
	FullName     string    `json:"full_name"`
	Role         Role      `json:"role"`
	IsActive     bool      `json:"is_active"`
	CreatedAt    time.Time `json:"created_at"`
	UpdatedAt    time.Time `json:"updated_at"`
}

// CreateUserRequest is the payload for user registration.
type CreateUserRequest struct {
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required,min=8,max=128"`
	FullName string `json:"full_name" validate:"required,min=1,max=255"`
}

// LoginRequest is the payload for user login.
type LoginRequest struct {
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required"`
}

// AuthResponse is returned after successful authentication.
type AuthResponse struct {
	AccessToken  string `json:"access_token"`
	RefreshToken string `json:"refresh_token"`
	ExpiresIn    int    `json:"expires_in"`
	User         *User  `json:"user"`
}

// RefreshRequest is the payload for token refresh.
type RefreshRequest struct {
	RefreshToken string `json:"refresh_token" validate:"required"`
}
```

### `internal/models/project.go`

```go
package models

import "time"

// ProjectStatus represents the status of a project.
type ProjectStatus string

const (
	ProjectStatusActive   ProjectStatus = "active"
	ProjectStatusArchived ProjectStatus = "archived"
)

// Project represents a project that contains tasks.
type Project struct {
	ID          string        `json:"id"`
	Name        string        `json:"name"`
	Description string        `json:"description"`
	OwnerID     string        `json:"owner_id"`
	Status      ProjectStatus `json:"status"`
	CreatedAt   time.Time     `json:"created_at"`
	UpdatedAt   time.Time     `json:"updated_at"`
}

// CreateProjectRequest is the payload for creating a project.
type CreateProjectRequest struct {
	Name        string `json:"name" validate:"required,min=1,max=255"`
	Description string `json:"description" validate:"max=2000"`
}

// UpdateProjectRequest is the payload for updating a project.
type UpdateProjectRequest struct {
	Name        *string `json:"name" validate:"omitempty,min=1,max=255"`
	Description *string `json:"description" validate:"omitempty,max=2000"`
	Status      *string `json:"status" validate:"omitempty,oneof=active archived"`
}
```

### `internal/models/task.go`

```go
package models

import "time"

// TaskStatus represents the status of a task.
type TaskStatus string

const (
	TaskStatusTodo       TaskStatus = "todo"
	TaskStatusInProgress TaskStatus = "in_progress"
	TaskStatusDone       TaskStatus = "done"
)

// Priority represents the priority of a task.
type Priority string

const (
	PriorityLow    Priority = "low"
	PriorityMedium Priority = "medium"
	PriorityHigh   Priority = "high"
	PriorityUrgent Priority = "urgent"
)

// Task represents a work item within a project.
type Task struct {
	ID          string     `json:"id"`
	ProjectID   string     `json:"project_id"`
	Title       string     `json:"title"`
	Description string     `json:"description"`
	Status      TaskStatus `json:"status"`
	Priority    Priority   `json:"priority"`
	AssigneeID  *string    `json:"assignee_id"` // nullable
	DueDate     *time.Time `json:"due_date"`    // nullable
	CreatedAt   time.Time  `json:"created_at"`
	UpdatedAt   time.Time  `json:"updated_at"`
}

// CreateTaskRequest is the payload for creating a task.
type CreateTaskRequest struct {
	Title       string  `json:"title" validate:"required,min=1,max=500"`
	Description string  `json:"description" validate:"max=5000"`
	Priority    string  `json:"priority" validate:"omitempty,oneof=low medium high urgent"`
	AssigneeID  *string `json:"assignee_id" validate:"omitempty,uuid"`
	DueDate     *string `json:"due_date" validate:"omitempty"`
}

// UpdateTaskRequest is the payload for updating a task.
type UpdateTaskRequest struct {
	Title       *string `json:"title" validate:"omitempty,min=1,max=500"`
	Description *string `json:"description" validate:"omitempty,max=5000"`
	Status      *string `json:"status" validate:"omitempty,oneof=todo in_progress done"`
	Priority    *string `json:"priority" validate:"omitempty,oneof=low medium high urgent"`
	AssigneeID  *string `json:"assignee_id" validate:"omitempty"`
	DueDate     *string `json:"due_date" validate:"omitempty"`
}

// TaskFilter holds query parameters for listing tasks.
type TaskFilter struct {
	Status     string
	Priority   string
	AssigneeID string
	Page       int
	PerPage    int
}
```

### Shared Pagination Types

```go
package models

// PaginationParams provides standardized pagination across all list endpoints.
type PaginationParams struct {
	Page    int `json:"page"`
	PerPage int `json:"per_page"`
}

// Limit returns the SQL LIMIT value.
func (p PaginationParams) Limit() int {
	if p.PerPage <= 0 || p.PerPage > 100 {
		return 20 // default
	}
	return p.PerPage
}

// Offset returns the SQL OFFSET value.
func (p PaginationParams) Offset() int {
	if p.Page <= 0 {
		return 0
	}
	return (p.Page - 1) * p.Limit()
}

// PaginationMeta is included in paginated API responses.
type PaginationMeta struct {
	Page       int `json:"page"`
	PerPage    int `json:"per_page"`
	Total      int `json:"total"`
	TotalPages int `json:"total_pages"`
}

// NewPaginationMeta computes pagination metadata.
func NewPaginationMeta(page, perPage, total int) PaginationMeta {
	if perPage <= 0 {
		perPage = 20
	}
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

### Custom Types vs Raw Strings

Notice that `TaskStatus`, `Priority`, `Role`, and `ProjectStatus` are all `type X string` rather than plain `string`. This is a Go idiom that gives you:

1. **Type safety** -- You cannot accidentally assign a `Priority` to a `TaskStatus`
2. **Documentation** -- The type name communicates intent
3. **Constant grouping** -- All valid values live next to the type definition
4. **Method receivers** -- You can attach methods like `IsValid()` to the type

In Node.js, you would use TypeScript enums or string literal union types:

```typescript
// TypeScript
type TaskStatus = 'todo' | 'in_progress' | 'done';
type Priority = 'low' | 'medium' | 'high' | 'urgent';
```

The Go version enforces these at the database level (via CHECK constraints if you add them) and at the validation level (via `oneof` tags).

---

## 6. Repository Layer

Repositories encapsulate all database access. Each repository is defined as an **interface** (for testing with mocks) and an **implementation** that uses `*sql.DB`.

This applies patterns from **Chapter 26**.

### `internal/repository/user_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"taskapi/internal/models"
)

// UserRepository defines the data access interface for users.
type UserRepository interface {
	Create(ctx context.Context, user *models.User) error
	GetByID(ctx context.Context, id string) (*models.User, error)
	GetByEmail(ctx context.Context, email string) (*models.User, error)
	Update(ctx context.Context, user *models.User) error
	Delete(ctx context.Context, id string) error
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

// Compile-time check: *userRepo must implement UserRepository.
var _ UserRepository = (*userRepo)(nil)

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
		return nil, nil // Not found -- caller checks for nil
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

func (r *userRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM users").Scan(&count)
	return count, err
}
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
	ListByOwner(ctx context.Context, ownerID string, params models.PaginationParams) ([]models.Project, int, error)
	Count(ctx context.Context) (int, error)
}

// projectRepo is the PostgreSQL implementation of ProjectRepository.
type projectRepo struct {
	db *sql.DB
}

// NewProjectRepository creates a new PostgreSQL-backed project repository.
func NewProjectRepository(db *sql.DB) ProjectRepository {
	return &projectRepo{db: db}
}

// Compile-time check
var _ ProjectRepository = (*projectRepo)(nil)

func (r *projectRepo) Create(ctx context.Context, project *models.Project) error {
	query := `
		INSERT INTO projects (name, description, owner_id, status)
		VALUES ($1, $2, $3, $4)
		RETURNING id, created_at, updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		project.Name, project.Description, project.OwnerID, project.Status,
	).Scan(&project.ID, &project.CreatedAt, &project.UpdatedAt)
}

func (r *projectRepo) GetByID(ctx context.Context, id string) (*models.Project, error) {
	query := `
		SELECT id, name, description, owner_id, status, created_at, updated_at
		FROM projects
		WHERE id = $1
	`
	p := &models.Project{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&p.ID, &p.Name, &p.Description, &p.OwnerID,
		&p.Status, &p.CreatedAt, &p.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting project by id: %w", err)
	}
	return p, nil
}

func (r *projectRepo) Update(ctx context.Context, project *models.Project) error {
	query := `
		UPDATE projects
		SET name = $1, description = $2, status = $3, updated_at = NOW()
		WHERE id = $4
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		project.Name, project.Description, project.Status, project.ID,
	).Scan(&project.UpdatedAt)
}

func (r *projectRepo) Delete(ctx context.Context, id string) error {
	result, err := r.db.ExecContext(ctx, "DELETE FROM projects WHERE id = $1", id)
	if err != nil {
		return fmt.Errorf("deleting project: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("checking affected rows: %w", err)
	}
	if rows == 0 {
		return fmt.Errorf("project not found")
	}
	return nil
}

func (r *projectRepo) ListByOwner(ctx context.Context, ownerID string, params models.PaginationParams) ([]models.Project, int, error) {
	// Count total
	var total int
	err := r.db.QueryRowContext(ctx,
		"SELECT COUNT(*) FROM projects WHERE owner_id = $1", ownerID,
	).Scan(&total)
	if err != nil {
		return nil, 0, fmt.Errorf("counting projects: %w", err)
	}

	// Fetch page
	query := `
		SELECT id, name, description, owner_id, status, created_at, updated_at
		FROM projects
		WHERE owner_id = $1
		ORDER BY created_at DESC
		LIMIT $2 OFFSET $3
	`
	rows, err := r.db.QueryContext(ctx, query, ownerID, params.Limit(), params.Offset())
	if err != nil {
		return nil, 0, fmt.Errorf("listing projects: %w", err)
	}
	defer rows.Close()

	var projects []models.Project
	for rows.Next() {
		var p models.Project
		if err := rows.Scan(
			&p.ID, &p.Name, &p.Description, &p.OwnerID,
			&p.Status, &p.CreatedAt, &p.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning project: %w", err)
		}
		projects = append(projects, p)
	}
	if err := rows.Err(); err != nil {
		return nil, 0, fmt.Errorf("iterating projects: %w", err)
	}

	return projects, total, nil
}

func (r *projectRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM projects").Scan(&count)
	return count, err
}
```

### `internal/repository/task_repository.go`

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"

	"taskapi/internal/models"
)

// TaskRepository defines the data access interface for tasks.
type TaskRepository interface {
	Create(ctx context.Context, task *models.Task) error
	GetByID(ctx context.Context, id string) (*models.Task, error)
	Update(ctx context.Context, task *models.Task) error
	Delete(ctx context.Context, id string) error
	ListByProject(ctx context.Context, projectID string, filter models.TaskFilter) ([]models.Task, int, error)
	Count(ctx context.Context) (int, error)
	CountOverdue(ctx context.Context) (int, error)
	MarkOverdueTasks(ctx context.Context) (int64, error)
}

// taskRepo is the PostgreSQL implementation of TaskRepository.
type taskRepo struct {
	db *sql.DB
}

// NewTaskRepository creates a new PostgreSQL-backed task repository.
func NewTaskRepository(db *sql.DB) TaskRepository {
	return &taskRepo{db: db}
}

// Compile-time check
var _ TaskRepository = (*taskRepo)(nil)

func (r *taskRepo) Create(ctx context.Context, task *models.Task) error {
	query := `
		INSERT INTO tasks (project_id, title, description, status, priority, assignee_id, due_date)
		VALUES ($1, $2, $3, $4, $5, $6, $7)
		RETURNING id, created_at, updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		task.ProjectID, task.Title, task.Description,
		task.Status, task.Priority, task.AssigneeID, task.DueDate,
	).Scan(&task.ID, &task.CreatedAt, &task.UpdatedAt)
}

func (r *taskRepo) GetByID(ctx context.Context, id string) (*models.Task, error) {
	query := `
		SELECT id, project_id, title, description, status, priority,
		       assignee_id, due_date, created_at, updated_at
		FROM tasks
		WHERE id = $1
	`
	t := &models.Task{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&t.ID, &t.ProjectID, &t.Title, &t.Description,
		&t.Status, &t.Priority, &t.AssigneeID, &t.DueDate,
		&t.CreatedAt, &t.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("getting task by id: %w", err)
	}
	return t, nil
}

func (r *taskRepo) Update(ctx context.Context, task *models.Task) error {
	query := `
		UPDATE tasks
		SET title = $1, description = $2, status = $3, priority = $4,
		    assignee_id = $5, due_date = $6, updated_at = NOW()
		WHERE id = $7
		RETURNING updated_at
	`
	return r.db.QueryRowContext(ctx, query,
		task.Title, task.Description, task.Status, task.Priority,
		task.AssigneeID, task.DueDate, task.ID,
	).Scan(&task.UpdatedAt)
}

func (r *taskRepo) Delete(ctx context.Context, id string) error {
	result, err := r.db.ExecContext(ctx, "DELETE FROM tasks WHERE id = $1", id)
	if err != nil {
		return fmt.Errorf("deleting task: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("checking affected rows: %w", err)
	}
	if rows == 0 {
		return fmt.Errorf("task not found")
	}
	return nil
}

func (r *taskRepo) ListByProject(ctx context.Context, projectID string, filter models.TaskFilter) ([]models.Task, int, error) {
	// Build dynamic WHERE clause
	where := "WHERE project_id = $1"
	args := []any{projectID}
	argIdx := 2

	if filter.Status != "" {
		where += fmt.Sprintf(" AND status = $%d", argIdx)
		args = append(args, filter.Status)
		argIdx++
	}
	if filter.Priority != "" {
		where += fmt.Sprintf(" AND priority = $%d", argIdx)
		args = append(args, filter.Priority)
		argIdx++
	}
	if filter.AssigneeID != "" {
		where += fmt.Sprintf(" AND assignee_id = $%d", argIdx)
		args = append(args, filter.AssigneeID)
		argIdx++
	}

	// Count total matching
	var total int
	countQuery := "SELECT COUNT(*) FROM tasks " + where
	if err := r.db.QueryRowContext(ctx, countQuery, args...).Scan(&total); err != nil {
		return nil, 0, fmt.Errorf("counting tasks: %w", err)
	}

	// Pagination defaults
	limit := filter.PerPage
	if limit <= 0 || limit > 100 {
		limit = 20
	}
	offset := 0
	if filter.Page > 1 {
		offset = (filter.Page - 1) * limit
	}

	// Fetch page
	query := fmt.Sprintf(`
		SELECT id, project_id, title, description, status, priority,
		       assignee_id, due_date, created_at, updated_at
		FROM tasks
		%s
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
	args = append(args, limit, offset)

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
			&t.Status, &t.Priority, &t.AssigneeID, &t.DueDate,
			&t.CreatedAt, &t.UpdatedAt,
		); err != nil {
			return nil, 0, fmt.Errorf("scanning task: %w", err)
		}
		tasks = append(tasks, t)
	}
	if err := rows.Err(); err != nil {
		return nil, 0, fmt.Errorf("iterating tasks: %w", err)
	}

	return tasks, total, nil
}

func (r *taskRepo) Count(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM tasks").Scan(&count)
	return count, err
}

func (r *taskRepo) CountOverdue(ctx context.Context) (int, error) {
	var count int
	err := r.db.QueryRowContext(ctx,
		"SELECT COUNT(*) FROM tasks WHERE due_date < CURRENT_DATE AND status != 'done'",
	).Scan(&count)
	return count, err
}

func (r *taskRepo) MarkOverdueTasks(ctx context.Context) (int64, error) {
	result, err := r.db.ExecContext(ctx, `
		UPDATE tasks
		SET updated_at = NOW()
		WHERE due_date < CURRENT_DATE
		  AND status != 'done'
		  AND status != 'overdue'
	`)
	if err != nil {
		return 0, fmt.Errorf("marking overdue tasks: %w", err)
	}
	return result.RowsAffected()
}
```

### The Compile-Time Interface Check Pattern

Notice this line in every repository file:

```go
var _ UserRepository = (*userRepo)(nil)
```

This is a zero-cost compile-time assertion. It tells the compiler: "verify that `*userRepo` satisfies the `UserRepository` interface." If you forget to implement a method, or change the interface, the build fails immediately -- not at runtime when a test happens to exercise that code path.

In Node.js/TypeScript, you get this from the type system:

```typescript
class UserRepo implements UserRepository {
  // TypeScript compiler forces you to implement all methods
}
```

The Go version achieves the same safety without generics or class keywords.

---

## 7. Authentication

Authentication handles user registration (password hashing), login (password verification + JWT issuance), and token refresh. This applies patterns from **Chapter 27**.

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

	"golang.org/x/crypto/bcrypt"
)

// JWTService handles token creation and validation.
type JWTService struct {
	secret        []byte
	accessExpiry  time.Duration
	refreshExpiry time.Duration
}

// NewJWTService creates a new JWT service.
func NewJWTService(secret string, accessExpiry, refreshExpiry time.Duration) *JWTService {
	return &JWTService{
		secret:        []byte(secret),
		accessExpiry:  accessExpiry,
		refreshExpiry: refreshExpiry,
	}
}

// AccessExpiry returns the access token expiration duration.
func (s *JWTService) AccessExpiry() time.Duration {
	return s.accessExpiry
}

// Claims represents the JWT payload.
type Claims struct {
	UserID string `json:"user_id"`
	Email  string `json:"email"`
	Role   string `json:"role"`
	Type   string `json:"type"` // "access" or "refresh"
	Exp    int64  `json:"exp"`
	Iat    int64  `json:"iat"`
}

// IsExpired returns true if the token has expired.
func (c *Claims) IsExpired() bool {
	return time.Now().Unix() > c.Exp
}

// jwtHeader is the static header for all our tokens.
var jwtHeader = base64URLEncode([]byte(`{"alg":"HS256","typ":"JWT"}`))

// GenerateAccessToken creates a short-lived access token.
func (s *JWTService) GenerateAccessToken(userID, email, role string) (string, error) {
	return s.generateToken(userID, email, role, "access", s.accessExpiry)
}

// GenerateRefreshToken creates a long-lived refresh token.
func (s *JWTService) GenerateRefreshToken(userID, email, role string) (string, error) {
	return s.generateToken(userID, email, role, "refresh", s.refreshExpiry)
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
	actualSig := parts[2]

	if expectedSig != actualSig {
		return nil, fmt.Errorf("invalid token signature")
	}

	// Decode payload
	payload, err := base64URLDecode(parts[1])
	if err != nil {
		return nil, fmt.Errorf("decoding payload: %w", err)
	}

	var claims Claims
	if err := json.Unmarshal(payload, &claims); err != nil {
		return nil, fmt.Errorf("parsing claims: %w", err)
	}

	if claims.IsExpired() {
		return nil, fmt.Errorf("token expired")
	}

	return &claims, nil
}

func (s *JWTService) generateToken(userID, email, role, tokenType string, expiry time.Duration) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID: userID,
		Email:  email,
		Role:   role,
		Type:   tokenType,
		Iat:    now.Unix(),
		Exp:    now.Add(expiry).Unix(),
	}

	payloadJSON, err := json.Marshal(claims)
	if err != nil {
		return "", fmt.Errorf("marshaling claims: %w", err)
	}

	payload := base64URLEncode(payloadJSON)
	signingInput := jwtHeader + "." + payload
	signature := s.sign([]byte(signingInput))

	return signingInput + "." + signature, nil
}

func (s *JWTService) sign(data []byte) string {
	mac := hmac.New(sha256.New, s.secret)
	mac.Write(data)
	return base64URLEncode(mac.Sum(nil))
}

func base64URLEncode(data []byte) string {
	return strings.TrimRight(base64.URLEncoding.EncodeToString(data), "=")
}

func base64URLDecode(s string) ([]byte, error) {
	// Add padding back
	switch len(s) % 4 {
	case 2:
		s += "=="
	case 3:
		s += "="
	}
	return base64.URLEncoding.DecodeString(s)
}

// --- Password Hashing ---

// HashPassword hashes a plaintext password using bcrypt.
func HashPassword(password string) (string, error) {
	bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hashing password: %w", err)
	}
	return string(bytes), nil
}

// CheckPassword verifies a password against a bcrypt hash.
func CheckPassword(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
	return err == nil
}
```

### Node.js/Express Comparison

In Express, you would use `jsonwebtoken` and `bcrypt`:

```javascript
// Node.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

// Generate token
const token = jwt.sign(
  { userId: user.id, email: user.email, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '15m' }
);

// Verify token
const decoded = jwt.verify(token, process.env.JWT_SECRET);

// Hash password
const hash = await bcrypt.hash(password, 10);

// Compare password
const match = await bcrypt.compare(password, hash);
```

The Go version builds JWT from scratch using HMAC-SHA256 from the standard library's `crypto/hmac` and `crypto/sha256`. In production, you could also use `github.com/golang-jwt/jwt/v5`, but the standard library provides everything needed. Bcrypt comes from `golang.org/x/crypto/bcrypt` -- the official extended library.

### Why Build JWT Manually?

We build JWT from scratch in this chapter for educational purposes -- so you understand exactly what a JWT is: a base64-encoded header, a base64-encoded payload, and an HMAC signature. In a production codebase, using `github.com/golang-jwt/jwt/v5` is perfectly acceptable and adds features like key rotation and multiple algorithms. The point is that Go's standard library makes it straightforward to implement common cryptographic protocols without third-party code.

---

## 8. Validation and Responses

Consistent input validation and response formatting are essential for a production API. Clients should always know what to expect.

This applies patterns from **Chapter 29**.

### `pkg/validator/validator.go`

```go
package validator

import (
	"fmt"
	"net/mail"
	"reflect"
	"strconv"
	"strings"
)

// ValidationErrors holds field-level validation errors.
type ValidationErrors struct {
	Errors []FieldError `json:"errors"`
}

// FieldError represents a single validation error on a field.
type FieldError struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// HasErrors returns true if there are validation errors.
func (ve *ValidationErrors) HasErrors() bool {
	return len(ve.Errors) > 0
}

// Add appends a validation error.
func (ve *ValidationErrors) Add(field, message string) {
	ve.Errors = append(ve.Errors, FieldError{Field: field, Message: message})
}

// Validate checks a struct against its `validate` tags.
// Supported tags: required, email, min=N, max=N, oneof=a b c, uuid
func Validate(v any) *ValidationErrors {
	errs := &ValidationErrors{}

	val := reflect.ValueOf(v)
	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}
	if val.Kind() != reflect.Struct {
		return errs
	}

	typ := val.Type()
	for i := 0; i < val.NumField(); i++ {
		field := typ.Field(i)
		value := val.Field(i)
		tag := field.Tag.Get("validate")
		if tag == "" {
			continue
		}

		// Get the JSON field name for error messages
		jsonName := field.Tag.Get("json")
		if jsonName == "" || jsonName == "-" {
			jsonName = field.Name
		}
		jsonName = strings.Split(jsonName, ",")[0]

		rules := strings.Split(tag, ",")
		for _, rule := range rules {
			if rule == "omitempty" {
				// If the value is zero/nil, skip remaining rules
				if value.IsZero() {
					break
				}
				continue
			}

			if err := applyRule(value, rule, jsonName); err != "" {
				errs.Add(jsonName, err)
				break // Only report first error per field
			}
		}
	}

	return errs
}

func applyRule(value reflect.Value, rule, fieldName string) string {
	// Handle pointer types
	if value.Kind() == reflect.Ptr {
		if value.IsNil() {
			if rule == "required" {
				return fmt.Sprintf("%s is required", fieldName)
			}
			return ""
		}
		value = value.Elem()
	}

	str := fmt.Sprintf("%v", value.Interface())

	switch {
	case rule == "required":
		if str == "" || str == "0" || str == "<nil>" {
			return fmt.Sprintf("%s is required", fieldName)
		}

	case rule == "email":
		if _, err := mail.ParseAddress(str); err != nil {
			return fmt.Sprintf("%s must be a valid email address", fieldName)
		}

	case strings.HasPrefix(rule, "min="):
		minStr := strings.TrimPrefix(rule, "min=")
		min, _ := strconv.Atoi(minStr)
		if value.Kind() == reflect.String && len(str) < min {
			return fmt.Sprintf("%s must be at least %d characters", fieldName, min)
		}

	case strings.HasPrefix(rule, "max="):
		maxStr := strings.TrimPrefix(rule, "max=")
		max, _ := strconv.Atoi(maxStr)
		if value.Kind() == reflect.String && len(str) > max {
			return fmt.Sprintf("%s must be at most %d characters", fieldName, max)
		}

	case strings.HasPrefix(rule, "oneof="):
		options := strings.Fields(strings.TrimPrefix(rule, "oneof="))
		found := false
		for _, opt := range options {
			if str == opt {
				found = true
				break
			}
		}
		if !found {
			return fmt.Sprintf("%s must be one of: %s", fieldName, strings.Join(options, ", "))
		}

	case rule == "uuid":
		if len(str) != 36 || strings.Count(str, "-") != 4 {
			return fmt.Sprintf("%s must be a valid UUID", fieldName)
		}
	}

	return ""
}
```

### `pkg/response/response.go`

```go
package response

import (
	"encoding/json"
	"net/http"
)

// APIResponse is the standard response envelope for all endpoints.
type APIResponse struct {
	Success bool   `json:"success"`
	Data    any    `json:"data,omitempty"`
	Error   *Error `json:"error,omitempty"`
	Meta    any    `json:"meta,omitempty"`
}

// Error represents an error in the API response.
type Error struct {
	Code    string `json:"code"`
	Message string `json:"message"`
	Details any    `json:"details,omitempty"`
}

// JSON writes a successful response with the given status code and data.
func JSON(w http.ResponseWriter, status int, data any) {
	writeJSON(w, status, APIResponse{
		Success: true,
		Data:    data,
	})
}

// JSONWithMeta writes a successful response with data and pagination metadata.
func JSONWithMeta(w http.ResponseWriter, status int, data any, meta any) {
	writeJSON(w, status, APIResponse{
		Success: true,
		Data:    data,
		Meta:    meta,
	})
}

// Created writes a 201 response.
func Created(w http.ResponseWriter, data any) {
	JSON(w, http.StatusCreated, data)
}

// NoContent writes a 204 response with no body.
func NoContent(w http.ResponseWriter) {
	w.WriteHeader(http.StatusNoContent)
}

// ErrorResponse writes an error response.
func ErrorResponse(w http.ResponseWriter, status int, code, message string) {
	writeJSON(w, status, APIResponse{
		Success: false,
		Error: &Error{
			Code:    code,
			Message: message,
		},
	})
}

// ErrorWithDetails writes an error response with validation details.
func ErrorWithDetails(w http.ResponseWriter, status int, code, message string, details any) {
	writeJSON(w, status, APIResponse{
		Success: false,
		Error: &Error{
			Code:    code,
			Message: message,
			Details: details,
		},
	})
}

// Convenience error responses

func BadRequest(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusBadRequest, "BAD_REQUEST", message)
}

func Unauthorized(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusUnauthorized, "UNAUTHORIZED", message)
}

func Forbidden(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusForbidden, "FORBIDDEN", message)
}

func NotFound(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusNotFound, "NOT_FOUND", message)
}

func Conflict(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusConflict, "CONFLICT", message)
}

func InternalError(w http.ResponseWriter, message string) {
	ErrorResponse(w, http.StatusInternalServerError, "INTERNAL_ERROR", message)
}

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}
```

### Response Format

Every response from the API follows this shape:

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 47,
    "total_pages": 3
  }
}
```

Error responses:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": {
      "errors": [
        { "field": "email", "message": "email must be a valid email address" },
        { "field": "password", "message": "password must be at least 8 characters" }
      ]
    }
  }
}
```

### Node.js/Express Comparison

In Express, you either build this yourself or use a library:

```javascript
// Node.js -- manual response helpers
const success = (res, data, meta) => {
  res.json({ success: true, data, meta });
};

const error = (res, status, code, message) => {
  res.status(status).json({ success: false, error: { code, message } });
};
```

The pattern is identical. The Go version uses `http.ResponseWriter` directly; Express wraps it in `res.json()`.

---

## 9. Middleware Stack

Middleware in Go is a function that takes an `http.Handler` and returns an `http.Handler`. Middleware wraps handlers to add cross-cutting behavior: logging, panic recovery, CORS, rate limiting, authentication.

This applies patterns from **Chapter 25**.

### `internal/middleware/recovery.go`

```go
package middleware

import (
	"log/slog"
	"net/http"
	"runtime/debug"

	"taskapi/pkg/response"
)

// Recovery catches panics in handlers and converts them to 500 responses.
// Without this, a panic in any handler crashes the entire server.
func Recovery(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if err := recover(); err != nil {
					stack := string(debug.Stack())
					logger.Error("panic recovered",
						"error", err,
						"method", r.Method,
						"path", r.URL.Path,
						"stack", stack,
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

```go
package middleware

import (
	"log/slog"
	"net/http"
	"time"
)

// statusWriter wraps http.ResponseWriter to capture the status code.
type statusWriter struct {
	http.ResponseWriter
	status int
}

func (sw *statusWriter) WriteHeader(code int) {
	sw.status = code
	sw.ResponseWriter.WriteHeader(code)
}

// Logging records method, path, status, and duration for every request.
func Logging(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
			next.ServeHTTP(sw, r)

			logger.Info("http request",
				"method", r.Method,
				"path", r.URL.Path,
				"status", sw.status,
				"duration", time.Since(start).String(),
				"remote_addr", r.RemoteAddr,
			)
		})
	}
}
```

### `internal/middleware/cors.go`

```go
package middleware

import (
	"net/http"
	"strings"
)

// CORS adds Cross-Origin Resource Sharing headers.
func CORS(allowedOrigins []string) func(http.Handler) http.Handler {
	originSet := make(map[string]bool, len(allowedOrigins))
	for _, o := range allowedOrigins {
		originSet[strings.TrimSpace(o)] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")

			if originSet["*"] || originSet[origin] {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
				w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
				w.Header().Set("Access-Control-Max-Age", "86400")
			}

			// Handle preflight
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

```go
package middleware

import (
	"log/slog"
	"net/http"
	"sync"
	"time"

	"taskapi/pkg/response"
)

// visitor tracks request counts per IP.
type visitor struct {
	count    int
	lastSeen time.Time
}

// RateLimit limits requests per IP within a time window.
func RateLimit(maxRequests int, window time.Duration, logger *slog.Logger) func(http.Handler) http.Handler {
	var mu sync.Mutex
	visitors := make(map[string]*visitor)

	// Background cleanup of stale entries
	go func() {
		for {
			time.Sleep(window)
			mu.Lock()
			for ip, v := range visitors {
				if time.Since(v.lastSeen) > window {
					delete(visitors, ip)
				}
			}
			mu.Unlock()
		}
	}()

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ip := r.RemoteAddr

			mu.Lock()
			v, exists := visitors[ip]
			if !exists {
				visitors[ip] = &visitor{count: 1, lastSeen: time.Now()}
				mu.Unlock()
				next.ServeHTTP(w, r)
				return
			}

			// Reset if window has passed
			if time.Since(v.lastSeen) > window {
				v.count = 1
				v.lastSeen = time.Now()
				mu.Unlock()
				next.ServeHTTP(w, r)
				return
			}

			v.count++
			v.lastSeen = time.Now()
			count := v.count
			mu.Unlock()

			if count > maxRequests {
				logger.Warn("rate limit exceeded",
					"ip", ip,
					"count", count,
					"max", maxRequests,
				)
				response.ErrorResponse(w, http.StatusTooManyRequests,
					"RATE_LIMIT_EXCEEDED",
					"Too many requests. Please try again later.",
				)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

### `internal/middleware/auth.go`

```go
package middleware

import (
	"context"
	"net/http"
	"strings"

	"taskapi/internal/auth"
	"taskapi/pkg/response"
)

// contextKey is an unexported type for context keys, preventing collisions.
type contextKey string

const (
	// UserIDKey is the context key for the authenticated user's ID.
	UserIDKey contextKey = "user_id"
	// UserEmailKey is the context key for the authenticated user's email.
	UserEmailKey contextKey = "user_email"
	// UserRoleKey is the context key for the authenticated user's role.
	UserRoleKey contextKey = "user_role"
)

// Auth validates the JWT from the Authorization header and injects
// user claims into the request context.
func Auth(jwtService *auth.JWTService) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			header := r.Header.Get("Authorization")
			if header == "" {
				response.Unauthorized(w, "Missing authorization header")
				return
			}

			parts := strings.SplitN(header, " ", 2)
			if len(parts) != 2 || parts[0] != "Bearer" {
				response.Unauthorized(w, "Invalid authorization format. Use: Bearer <token>")
				return
			}

			claims, err := jwtService.ValidateToken(parts[1])
			if err != nil {
				response.Unauthorized(w, "Invalid or expired token")
				return
			}

			if claims.Type != "access" {
				response.Unauthorized(w, "Invalid token type. Use an access token.")
				return
			}

			// Inject user info into context
			ctx := r.Context()
			ctx = context.WithValue(ctx, UserIDKey, claims.UserID)
			ctx = context.WithValue(ctx, UserEmailKey, claims.Email)
			ctx = context.WithValue(ctx, UserRoleKey, claims.Role)

			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// GetUserID extracts the user ID from the request context.
func GetUserID(ctx context.Context) string {
	if id, ok := ctx.Value(UserIDKey).(string); ok {
		return id
	}
	return ""
}

// GetUserRole extracts the user role from the request context.
func GetUserRole(ctx context.Context) string {
	if role, ok := ctx.Value(UserRoleKey).(string); ok {
		return role
	}
	return ""
}
```

### Middleware Chaining

All middleware follows the same signature: `func(http.Handler) http.Handler`. They compose by wrapping:

```go
// This is how middleware stacks work -- each wraps the next.
// The request flows: Recovery -> Logging -> CORS -> RateLimit -> Auth -> Handler
handler := Recovery(logger)(
    Logging(logger)(
        CORS(origins)(
            RateLimit(100, time.Minute, logger)(
                Auth(jwtService)(
                    actualHandler,
                ),
            ),
        ),
    ),
)
```

We will use a helper to make this cleaner in the router (Section 10).

### Node.js/Express Comparison

In Express, middleware is `app.use()`:

```javascript
// Node.js / Express
app.use(helmet());          // Security headers
app.use(cors(corsOptions)); // CORS
app.use(morgan('combined')); // Logging
app.use(rateLimit({ windowMs: 60000, max: 100 })); // Rate limit
app.use('/api', authMiddleware); // Auth on API routes
```

The Go pattern is functionally identical. Each middleware receives the request, does its work, and either calls the next handler or short-circuits (e.g., returning 401 for auth failures).

---

## 10. Router

The router maps HTTP methods and paths to handler functions. Go 1.22 enhanced the standard `http.ServeMux` with method-based routing and path parameters, eliminating the need for third-party routers in most cases.

This applies patterns from **Chapter 24**.

### `internal/router/router.go`

```go
package router

import (
	"log/slog"
	"net/http"
	"time"

	"taskapi/internal/auth"
	"taskapi/internal/database"
	"taskapi/internal/handlers"
	"taskapi/internal/middleware"
	"taskapi/internal/repository"
)

// New creates and configures the HTTP router with all routes and middleware.
func New(
	db *database.DB,
	jwtService *auth.JWTService,
	logger *slog.Logger,
	allowedOrigins []string,
	rateLimitMax int,
	rateLimitWindow time.Duration,
) http.Handler {
	mux := http.NewServeMux()

	// --- Create repositories ---
	userRepo := repository.NewUserRepository(db.DB)
	projectRepo := repository.NewProjectRepository(db.DB)
	taskRepo := repository.NewTaskRepository(db.DB)

	// --- Create handlers ---
	healthHandler := handlers.NewHealthHandler(db)
	authHandler := handlers.NewAuthHandler(userRepo, jwtService, logger)
	projectHandler := handlers.NewProjectHandler(projectRepo, logger)
	taskHandler := handlers.NewTaskHandler(taskRepo, projectRepo, logger)
	dashboardHandler := handlers.NewDashboardHandler(userRepo, projectRepo, taskRepo, logger)

	// --- Auth middleware ---
	requireAuth := middleware.Auth(jwtService)

	// --- Public routes (no auth required) ---
	mux.HandleFunc("GET /health", healthHandler.Health)
	mux.HandleFunc("GET /ready", healthHandler.Ready)
	mux.HandleFunc("POST /api/v1/auth/register", authHandler.Register)
	mux.HandleFunc("POST /api/v1/auth/login", authHandler.Login)
	mux.HandleFunc("POST /api/v1/auth/refresh", authHandler.Refresh)

	// --- Protected routes (auth required) ---

	// Projects
	mux.Handle("POST /api/v1/projects", requireAuth(http.HandlerFunc(projectHandler.Create)))
	mux.Handle("GET /api/v1/projects", requireAuth(http.HandlerFunc(projectHandler.List)))
	mux.Handle("GET /api/v1/projects/{id}", requireAuth(http.HandlerFunc(projectHandler.GetByID)))
	mux.Handle("PUT /api/v1/projects/{id}", requireAuth(http.HandlerFunc(projectHandler.Update)))
	mux.Handle("DELETE /api/v1/projects/{id}", requireAuth(http.HandlerFunc(projectHandler.Delete)))

	// Tasks
	mux.Handle("POST /api/v1/projects/{projectId}/tasks", requireAuth(http.HandlerFunc(taskHandler.Create)))
	mux.Handle("GET /api/v1/projects/{projectId}/tasks", requireAuth(http.HandlerFunc(taskHandler.List)))
	mux.Handle("GET /api/v1/tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.GetByID)))
	mux.Handle("PUT /api/v1/tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.Update)))
	mux.Handle("DELETE /api/v1/tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.Delete)))

	// Dashboard
	mux.Handle("GET /api/v1/dashboard", requireAuth(http.HandlerFunc(dashboardHandler.Overview)))

	// --- Apply global middleware stack ---
	// Order matters: outermost middleware runs first.
	var handler http.Handler = mux
	handler = middleware.RateLimit(rateLimitMax, rateLimitWindow, logger)(handler)
	handler = middleware.CORS(allowedOrigins)(handler)
	handler = middleware.Logging(logger)(handler)
	handler = middleware.Recovery(logger)(handler)

	return handler
}
```

### How Go 1.22 Routing Works

Go 1.22 introduced enhanced pattern matching in `http.ServeMux`:

```go
// Method-based dispatch -- no need for manual method checking
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("POST /users", createUser)

// Path parameters -- extracted with r.PathValue()
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
}
```

Before Go 1.22, you needed `gorilla/mux` or `chi` for this. Now the standard library handles it natively.

### Node.js/Express Comparison

In Express:

```javascript
// Express router
const router = express.Router();

// Public routes
router.post('/auth/register', authController.register);
router.post('/auth/login', authController.login);

// Protected routes
router.use(authMiddleware); // All routes below require auth
router.get('/projects', projectController.list);
router.post('/projects', projectController.create);
router.get('/projects/:id', projectController.getById);

app.use('/api/v1', router);
```

The structure is nearly identical. Express uses `:id` for path params; Go 1.22 uses `{id}`. Express uses `router.use()` for middleware; Go wraps handlers with middleware functions.

---

## 11. Handlers

Handlers are the HTTP-facing layer. They parse requests, validate input, call repositories, and write responses. They should be thin -- coordinate, do not compute.

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

// Health is a simple liveness check.
func (h *HealthHandler) Health(w http.ResponseWriter, r *http.Request) {
	response.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

// Ready checks if the application can serve traffic (database reachable).
func (h *HealthHandler) Ready(w http.ResponseWriter, r *http.Request) {
	if err := h.db.Health(r.Context()); err != nil {
		response.ErrorResponse(w, http.StatusServiceUnavailable, "NOT_READY", "Database is not available")
		return
	}
	response.JSON(w, http.StatusOK, map[string]string{"status": "ready"})
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

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
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

	h.logger.Info("user registered", "user_id", user.ID, "email", user.Email)

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

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

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

	if !auth.CheckPassword(req.Password, user.PasswordHash) {
		response.Unauthorized(w, "Invalid email or password")
		return
	}

	if !user.IsActive {
		response.Forbidden(w, "Account is deactivated")
		return
	}

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

	h.logger.Info("user logged in", "user_id", user.ID)

	response.JSON(w, http.StatusOK, models.AuthResponse{
		AccessToken:  accessToken,
		RefreshToken: refreshToken,
		ExpiresIn:    int(h.jwtService.AccessExpiry().Seconds()),
		User:         user,
	})
}

// Refresh exchanges a valid refresh token for a new access token.
func (h *AuthHandler) Refresh(w http.ResponseWriter, r *http.Request) {
	var req models.RefreshRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

	claims, err := h.jwtService.ValidateToken(req.RefreshToken)
	if err != nil {
		response.Unauthorized(w, "Invalid or expired refresh token")
		return
	}

	if claims.Type != "refresh" {
		response.Unauthorized(w, "Invalid token type. Provide a refresh token.")
		return
	}

	// Issue new access token
	accessToken, err := h.jwtService.GenerateAccessToken(claims.UserID, claims.Email, claims.Role)
	if err != nil {
		h.logger.Error("generating access token", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	response.JSON(w, http.StatusOK, map[string]any{
		"access_token": accessToken,
		"expires_in":   int(h.jwtService.AccessExpiry().Seconds()),
	})
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
	repo   repository.ProjectRepository
	logger *slog.Logger
}

// NewProjectHandler creates a new project handler.
func NewProjectHandler(repo repository.ProjectRepository, logger *slog.Logger) *ProjectHandler {
	return &ProjectHandler{repo: repo, logger: logger}
}

// Create creates a new project owned by the authenticated user.
func (h *ProjectHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req models.CreateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

	userID := middleware.GetUserID(r.Context())

	project := &models.Project{
		Name:        req.Name,
		Description: req.Description,
		OwnerID:     userID,
		Status:      models.ProjectStatusActive,
	}

	if err := h.repo.Create(r.Context(), project); err != nil {
		h.logger.Error("creating project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project created", "project_id", project.ID, "owner_id", userID)
	response.Created(w, project)
}

// List returns all projects owned by the authenticated user.
func (h *ProjectHandler) List(w http.ResponseWriter, r *http.Request) {
	userID := middleware.GetUserID(r.Context())

	page, _ := strconv.Atoi(r.URL.Query().Get("page"))
	perPage, _ := strconv.Atoi(r.URL.Query().Get("per_page"))
	if page < 1 {
		page = 1
	}
	if perPage < 1 || perPage > 100 {
		perPage = 20
	}

	params := models.PaginationParams{Page: page, PerPage: perPage}
	projects, total, err := h.repo.ListByOwner(r.Context(), userID, params)
	if err != nil {
		h.logger.Error("listing projects", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	meta := models.NewPaginationMeta(page, perPage, total)
	response.JSONWithMeta(w, http.StatusOK, projects, meta)
}

// GetByID returns a single project by ID.
func (h *ProjectHandler) GetByID(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	project, err := h.repo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	// Authorization: only the owner can view
	userID := middleware.GetUserID(r.Context())
	if project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this project")
		return
	}

	response.JSON(w, http.StatusOK, project)
}

// Update modifies an existing project.
func (h *ProjectHandler) Update(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	project, err := h.repo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	userID := middleware.GetUserID(r.Context())
	if project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this project")
		return
	}

	var req models.UpdateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

	// Apply partial updates
	if req.Name != nil {
		project.Name = *req.Name
	}
	if req.Description != nil {
		project.Description = *req.Description
	}
	if req.Status != nil {
		project.Status = models.ProjectStatus(*req.Status)
	}

	if err := h.repo.Update(r.Context(), project); err != nil {
		h.logger.Error("updating project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project updated", "project_id", project.ID)
	response.JSON(w, http.StatusOK, project)
}

// Delete removes a project and all its tasks (via CASCADE).
func (h *ProjectHandler) Delete(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	project, err := h.repo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}

	userID := middleware.GetUserID(r.Context())
	if project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this project")
		return
	}

	if err := h.repo.Delete(r.Context(), id); err != nil {
		h.logger.Error("deleting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("project deleted", "project_id", id)
	response.NoContent(w)
}
```

### `internal/handlers/task_handler.go`

```go
package handlers

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"strconv"
	"time"

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

// Create creates a new task within a project.
func (h *TaskHandler) Create(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("projectId")
	userID := middleware.GetUserID(r.Context())

	// Verify the project exists and belongs to the user
	project, err := h.projectRepo.GetByID(r.Context(), projectID)
	if err != nil {
		h.logger.Error("getting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}
	if project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this project")
		return
	}

	var req models.CreateTaskRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

	// Set defaults
	priority := models.PriorityMedium
	if req.Priority != "" {
		priority = models.Priority(req.Priority)
	}

	task := &models.Task{
		ProjectID:   projectID,
		Title:       req.Title,
		Description: req.Description,
		Status:      models.TaskStatusTodo,
		Priority:    priority,
		AssigneeID:  req.AssigneeID,
	}

	// Parse due date if provided
	if req.DueDate != nil && *req.DueDate != "" {
		t, err := time.Parse("2006-01-02", *req.DueDate)
		if err != nil {
			response.BadRequest(w, "Invalid due_date format. Use YYYY-MM-DD.")
			return
		}
		task.DueDate = &t
	}

	if err := h.taskRepo.Create(r.Context(), task); err != nil {
		h.logger.Error("creating task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task created", "task_id", task.ID, "project_id", projectID)
	response.Created(w, task)
}

// List returns all tasks for a project with filtering and pagination.
func (h *TaskHandler) List(w http.ResponseWriter, r *http.Request) {
	projectID := r.PathValue("projectId")
	userID := middleware.GetUserID(r.Context())

	// Verify project access
	project, err := h.projectRepo.GetByID(r.Context(), projectID)
	if err != nil {
		h.logger.Error("getting project", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil {
		response.NotFound(w, "Project not found")
		return
	}
	if project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this project")
		return
	}

	page, _ := strconv.Atoi(r.URL.Query().Get("page"))
	perPage, _ := strconv.Atoi(r.URL.Query().Get("per_page"))
	if page < 1 {
		page = 1
	}
	if perPage < 1 || perPage > 100 {
		perPage = 20
	}

	filter := models.TaskFilter{
		Status:     r.URL.Query().Get("status"),
		Priority:   r.URL.Query().Get("priority"),
		AssigneeID: r.URL.Query().Get("assignee_id"),
		Page:       page,
		PerPage:    perPage,
	}

	tasks, total, err := h.taskRepo.ListByProject(r.Context(), projectID, filter)
	if err != nil {
		h.logger.Error("listing tasks", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	meta := models.NewPaginationMeta(page, perPage, total)
	response.JSONWithMeta(w, http.StatusOK, tasks, meta)
}

// GetByID returns a single task.
func (h *TaskHandler) GetByID(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	task, err := h.taskRepo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	// Verify the user owns the project this task belongs to
	userID := middleware.GetUserID(r.Context())
	project, err := h.projectRepo.GetByID(r.Context(), task.ProjectID)
	if err != nil {
		h.logger.Error("getting project for task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil || project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	response.JSON(w, http.StatusOK, task)
}

// Update modifies an existing task.
func (h *TaskHandler) Update(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	task, err := h.taskRepo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	// Authorization check
	userID := middleware.GetUserID(r.Context())
	project, err := h.projectRepo.GetByID(r.Context(), task.ProjectID)
	if err != nil {
		h.logger.Error("getting project for task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil || project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	var req models.UpdateTaskRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		response.BadRequest(w, "Invalid JSON body")
		return
	}

	if errs := validator.Validate(&req); errs.HasErrors() {
		response.ErrorWithDetails(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid input", errs)
		return
	}

	// Apply partial updates
	if req.Title != nil {
		task.Title = *req.Title
	}
	if req.Description != nil {
		task.Description = *req.Description
	}
	if req.Status != nil {
		task.Status = models.TaskStatus(*req.Status)
	}
	if req.Priority != nil {
		task.Priority = models.Priority(*req.Priority)
	}
	if req.AssigneeID != nil {
		if *req.AssigneeID == "" {
			task.AssigneeID = nil // Unassign
		} else {
			task.AssigneeID = req.AssigneeID
		}
	}
	if req.DueDate != nil {
		if *req.DueDate == "" {
			task.DueDate = nil // Clear due date
		} else {
			t, err := time.Parse("2006-01-02", *req.DueDate)
			if err != nil {
				response.BadRequest(w, "Invalid due_date format. Use YYYY-MM-DD.")
				return
			}
			task.DueDate = &t
		}
	}

	if err := h.taskRepo.Update(r.Context(), task); err != nil {
		h.logger.Error("updating task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task updated", "task_id", task.ID)
	response.JSON(w, http.StatusOK, task)
}

// Delete removes a task.
func (h *TaskHandler) Delete(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	task, err := h.taskRepo.GetByID(r.Context(), id)
	if err != nil {
		h.logger.Error("getting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if task == nil {
		response.NotFound(w, "Task not found")
		return
	}

	userID := middleware.GetUserID(r.Context())
	project, err := h.projectRepo.GetByID(r.Context(), task.ProjectID)
	if err != nil {
		h.logger.Error("getting project for task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}
	if project == nil || project.OwnerID != userID {
		response.Forbidden(w, "You do not have access to this task")
		return
	}

	if err := h.taskRepo.Delete(r.Context(), id); err != nil {
		h.logger.Error("deleting task", "error", err)
		response.InternalError(w, "Internal server error")
		return
	}

	h.logger.Info("task deleted", "task_id", id)
	response.NoContent(w)
}
```

### `internal/handlers/dashboard_handler.go`

```go
package handlers

import (
	"log/slog"
	"net/http"
	"sync"

	"taskapi/internal/repository"
	"taskapi/pkg/response"
)

// DashboardHandler provides aggregated statistics.
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

// DashboardData holds the aggregated statistics.
type DashboardData struct {
	TotalUsers    int `json:"total_users"`
	TotalProjects int `json:"total_projects"`
	TotalTasks    int `json:"total_tasks"`
	OverdueTasks  int `json:"overdue_tasks"`
}

// Overview runs four independent database queries in parallel to build
// dashboard statistics. This demonstrates the parallel query pattern
// from Chapter 31.
//
// Sequential:  |--users--||--projects--||--tasks--||--overdue--|  = ~40ms
// Parallel:    |--users--|                                        = ~10ms
//              |--projects--|
//              |--tasks--|
//              |--overdue--|
func (h *DashboardHandler) Overview(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	var data DashboardData
	var mu sync.Mutex
	var wg sync.WaitGroup
	var firstErr error

	// Helper to record the first error
	setErr := func(err error) {
		mu.Lock()
		if firstErr == nil {
			firstErr = err
		}
		mu.Unlock()
	}

	wg.Add(4)

	go func() {
		defer wg.Done()
		count, err := h.userRepo.Count(ctx)
		if err != nil {
			setErr(err)
			return
		}
		mu.Lock()
		data.TotalUsers = count
		mu.Unlock()
	}()

	go func() {
		defer wg.Done()
		count, err := h.projectRepo.Count(ctx)
		if err != nil {
			setErr(err)
			return
		}
		mu.Lock()
		data.TotalProjects = count
		mu.Unlock()
	}()

	go func() {
		defer wg.Done()
		count, err := h.taskRepo.Count(ctx)
		if err != nil {
			setErr(err)
			return
		}
		mu.Lock()
		data.TotalTasks = count
		mu.Unlock()
	}()

	go func() {
		defer wg.Done()
		count, err := h.taskRepo.CountOverdue(ctx)
		if err != nil {
			setErr(err)
			return
		}
		mu.Lock()
		data.OverdueTasks = count
		mu.Unlock()
	}()

	wg.Wait()

	if firstErr != nil {
		h.logger.Error("dashboard query failed", "error", firstErr)
		response.InternalError(w, "Internal server error")
		return
	}

	response.JSON(w, http.StatusOK, data)
}
```

### Handler Pattern Summary

Every handler follows the same pattern:

1. **Parse** the request (path params, query params, JSON body)
2. **Validate** the input using the validator
3. **Authorize** by checking that the authenticated user has access
4. **Execute** by calling the repository
5. **Respond** with a consistent JSON envelope

This is exactly the same flow as an Express controller:

```javascript
// Node.js / Express equivalent
const createProject = async (req, res) => {
  // 1. Parse -- Express does this automatically
  const { name, description } = req.body;

  // 2. Validate
  const { error } = projectSchema.validate(req.body);
  if (error) return res.status(400).json({ ... });

  // 3. Authorize
  const userId = req.user.id; // From auth middleware

  // 4. Execute
  const project = await db('projects').insert({ name, description, owner_id: userId });

  // 5. Respond
  res.status(201).json({ success: true, data: project });
};
```

---

## 12. Structured Logging

We use Go's `log/slog` package (standard library since Go 1.21) for structured, leveled logging throughout the application.

This applies patterns from **Chapter 28**.

### Logger Setup

The logger is created in `main.go` and passed to every layer via constructor injection:

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
{"time":"2025-03-15T10:30:00.000Z","level":"INFO","msg":"http request","method":"POST","path":"/api/v1/projects","status":201,"duration":"3.2ms","remote_addr":"192.168.1.1:54321"}
{"time":"2025-03-15T10:30:00.003Z","level":"INFO","msg":"project created","project_id":"550e8400-e29b-41d4-a716-446655440000","owner_id":"user-123"}
```

In text format (development):

```
time=2025-03-15T10:30:00.000Z level=INFO msg="http request" method=POST path=/api/v1/projects status=201 duration=3.2ms
```

### Why `log/slog` Instead of Third-Party Loggers?

Before Go 1.21, the standard `log` package only supported unstructured text. Teams used `zap`, `zerolog`, or `logrus`. Now `log/slog` provides:

- **Structured key-value pairs** -- `slog.Info("msg", "key", value)`
- **Log levels** -- Debug, Info, Warn, Error
- **Pluggable handlers** -- JSON, text, or custom
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

Our API uses concurrency in two places: a background cleanup worker and the parallel dashboard queries (shown in Section 11). Both come from **Chapter 31**.

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

// Start begins the background cleanup loop. It runs until the context is cancelled.
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

### How the Worker Fits In

The cleanup worker runs as a goroutine started from `main.go`. The key concurrency patterns:

1. **Ticker-based loop** -- `time.NewTicker` fires at regular intervals
2. **Context cancellation** -- When main cancels the context (on SIGTERM), the worker exits
3. **Done channel** -- Main waits for the worker to finish before closing the database

```
main goroutine                    cleanup worker goroutine
      |                                   |
      |--- go worker.Start(ctx) --------->|
      |                                   |--- ticker fires
      |                                   |--- mark overdue tasks
      |                                   |--- ticker fires
      |                                   |--- mark overdue tasks
      |                                   |
      |--- cancel(ctx) ----------------->|
      |                                   |--- ctx.Done() received
      |                                   |--- close(done)
      |<-- <-worker.Done() --------------|
      |--- db.Close()
```

### Parallel Dashboard Queries

The `DashboardHandler.Overview` method (Section 11) demonstrates the parallel query pattern. Four independent COUNT queries run concurrently using goroutines with `sync.WaitGroup`, reducing total response time from the sum of all queries to the duration of the slowest single query.

```
Sequential:  |--users--||--projects--||--tasks--||--overdue--|  = ~40ms total
Parallel:    |--users--|                                        = ~10ms total
             |--projects--|
             |--tasks--|
             |--overdue--|
```

### Node.js/Express Comparison

In Node.js, you use `Promise.all` for parallel queries:

```javascript
// Node.js
const [users, projects, tasks, overdue] = await Promise.all([
  db('users').count('* as count').first(),
  db('projects').count('* as count').first(),
  db('tasks').count('* as count').first(),
  db('tasks').where('due_date', '<', 'now()').where('status', '!=', 'done').count('* as count').first(),
]);
```

Both approaches achieve the same result: run independent I/O operations in parallel. Go uses goroutines and WaitGroup; Node.js uses Promises and `Promise.all`. The Go version gives you explicit control over error handling and shared state.

---

## 14. Graceful Shutdown and Main

The `main.go` file wires everything together and manages the application lifecycle. It follows the pattern from **Chapter 30**: initialize, start, wait for signal, shutdown gracefully.

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

// run contains all initialization and lifecycle logic.
// Separating it from main() makes the application testable.
func run() error {
	// --- 1. Load configuration ---
	cfg, err := config.Load()
	if err != nil {
		return fmt.Errorf("loading config: %w", err)
	}

	// --- 2. Setup logger ---
	logger := setupLogger(cfg.LogLevel, cfg.LogFormat)
	logger.Info("starting application",
		"environment", cfg.Environment,
		"port", cfg.ServerPort,
	)

	// --- 3. Connect to database ---
	db, err := database.New(
		cfg.DatabaseURL(),
		cfg.DatabaseMaxConns,
		cfg.DatabaseMinConns,
		logger,
	)
	if err != nil {
		return fmt.Errorf("connecting to database: %w", err)
	}
	defer db.Close()

	// --- 4. Run migrations ---
	if err := db.RunMigrations(context.Background(), "internal/database/migrations"); err != nil {
		return fmt.Errorf("running migrations: %w", err)
	}

	// --- 5. Create JWT service ---
	jwtService := auth.NewJWTService(cfg.JWTSecret, cfg.JWTExpiration, cfg.JWTRefreshExpiry)

	// --- 6. Start background workers ---
	workerCtx, workerCancel := context.WithCancel(context.Background())
	defer workerCancel()

	taskRepo := repository.NewTaskRepository(db.DB)
	cleanupWorker := worker.NewCleanupWorker(taskRepo, logger, cfg.CleanupInterval)
	go cleanupWorker.Start(workerCtx)

	// --- 7. Create router with all routes and middleware ---
	handler := router.New(
		db,
		jwtService,
		logger,
		cfg.CORSAllowedOrigins,
		cfg.RateLimitRequests,
		cfg.RateLimitWindow,
	)

	// --- 8. Configure HTTP server ---
	server := &http.Server{
		Addr:         cfg.ServerAddr(),
		Handler:      handler,
		ReadTimeout:  cfg.ServerReadTimeout,
		WriteTimeout: cfg.ServerWriteTimeout,
		IdleTimeout:  cfg.ServerIdleTimeout,
	}

	// --- 9. Start server in a goroutine ---
	serverErr := make(chan error, 1)
	go func() {
		logger.Info("server listening", "addr", server.Addr)
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			serverErr <- err
		}
	}()

	// --- 10. Wait for shutdown signal ---
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-serverErr:
		return fmt.Errorf("server error: %w", err)
	case sig := <-quit:
		logger.Info("shutdown signal received", "signal", sig.String())
	}

	// --- 11. Graceful shutdown ---
	logger.Info("shutting down server...")

	// Give active requests time to finish
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer shutdownCancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("server shutdown: %w", err)
	}
	logger.Info("server stopped accepting new connections")

	// Stop background workers
	workerCancel()
	select {
	case <-cleanupWorker.Done():
		logger.Info("cleanup worker stopped")
	case <-time.After(10 * time.Second):
		logger.Warn("cleanup worker did not stop in time")
	}

	// Close database
	if err := db.Close(); err != nil {
		logger.Error("closing database", "error", err)
	}

	logger.Info("shutdown complete")
	return nil
}
```

### The `run()` Pattern

Notice that `main()` is a one-liner that calls `run()`. This is a common Go pattern for two reasons:

1. **Error handling** -- `main()` cannot return an error, but `run()` can. This lets you use normal Go error handling (`if err != nil { return err }`) instead of `log.Fatal()` scattered throughout initialization.

2. **Testability** -- You can test `run()` by setting environment variables and calling it directly, which is impossible with `main()`.

### Shutdown Order

The shutdown sequence is deliberate:

1. **Stop accepting new HTTP connections** (`server.Shutdown`)
2. **Drain active requests** (Shutdown waits for in-flight requests)
3. **Stop background workers** (`workerCancel`)
4. **Wait for workers to finish** (`<-cleanupWorker.Done()`)
5. **Close database pool** (`db.Close()`)

This order prevents a common bug: if you close the database before draining HTTP requests, in-flight requests that try to query the database will fail.

### Node.js/Express Comparison

In Express:

```javascript
// Node.js
const server = app.listen(PORT, () => {
  console.log(`Listening on port ${PORT}`);
});

process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  server.close(() => {
    console.log('Server closed');
    pool.end(() => {
      console.log('Database pool closed');
      process.exit(0);
    });
  });
});
```

The Go version is more explicit about timeout handling (30 seconds for shutdown, 10 seconds for worker cleanup) and about the order of operations. The `context.WithTimeout` pattern ensures that a stuck request or worker does not prevent shutdown forever.

---

## 15. Testing the API

Testing in Go uses the standard `testing` package. No test runner installation required -- `go test ./...` runs everything. We write two kinds of tests: unit tests with mocks and integration tests.

### Handler Unit Tests with Mocks

Since our handlers depend on repository interfaces, we can test them without a database by providing mock implementations.

### `tests/mocks_test.go`

```go
package tests

import (
	"context"

	"taskapi/internal/models"
)

// mockUserRepo is a test double for repository.UserRepository.
type mockUserRepo struct {
	users    map[string]*models.User
	byEmail  map[string]*models.User
	createFn func(ctx context.Context, user *models.User) error
}

func newMockUserRepo() *mockUserRepo {
	return &mockUserRepo{
		users:   make(map[string]*models.User),
		byEmail: make(map[string]*models.User),
	}
}

func (m *mockUserRepo) Create(ctx context.Context, user *models.User) error {
	if m.createFn != nil {
		return m.createFn(ctx, user)
	}
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

func (m *mockUserRepo) Count(ctx context.Context) (int, error) {
	return len(m.users), nil
}
```

### `tests/auth_handler_test.go`

```go
package tests

import (
	"bytes"
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"
	"time"

	"taskapi/internal/auth"
	"taskapi/internal/handlers"
	"taskapi/pkg/response"
)

func TestRegister_Success(t *testing.T) {
	// Setup
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)
	logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	// Create request
	body := map[string]string{
		"email":     "test@example.com",
		"password":  "securepass123",
		"full_name": "Test User",
	}
	bodyJSON, _ := json.Marshal(body)
	req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	// Execute
	handler.Register(rec, req)

	// Assert
	if rec.Code != http.StatusCreated {
		t.Errorf("expected status 201, got %d", rec.Code)
	}

	var resp response.APIResponse
	if err := json.NewDecoder(rec.Body).Decode(&resp); err != nil {
		t.Fatalf("decoding response: %v", err)
	}

	if !resp.Success {
		t.Error("expected success=true")
	}
}

func TestRegister_DuplicateEmail(t *testing.T) {
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)
	logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	// Pre-populate user
	userRepo.byEmail["existing@example.com"] = &models.User{
		ID:    "existing-id",
		Email: "existing@example.com",
	}

	body := map[string]string{
		"email":     "existing@example.com",
		"password":  "securepass123",
		"full_name": "Test User",
	}
	bodyJSON, _ := json.Marshal(body)
	req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	handler.Register(rec, req)

	if rec.Code != http.StatusConflict {
		t.Errorf("expected status 409, got %d", rec.Code)
	}
}

func TestRegister_InvalidInput(t *testing.T) {
	userRepo := newMockUserRepo()
	jwtService := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)
	logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
	handler := handlers.NewAuthHandler(userRepo, jwtService, logger)

	tests := []struct {
		name string
		body map[string]string
	}{
		{
			name: "missing email",
			body: map[string]string{"password": "securepass123", "full_name": "Test"},
		},
		{
			name: "invalid email",
			body: map[string]string{"email": "not-an-email", "password": "securepass123", "full_name": "Test"},
		},
		{
			name: "short password",
			body: map[string]string{"email": "test@example.com", "password": "short", "full_name": "Test"},
		},
		{
			name: "missing name",
			body: map[string]string{"email": "test@example.com", "password": "securepass123"},
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			bodyJSON, _ := json.Marshal(tc.body)
			req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register", bytes.NewReader(bodyJSON))
			req.Header.Set("Content-Type", "application/json")
			rec := httptest.NewRecorder()

			handler.Register(rec, req)

			if rec.Code != http.StatusBadRequest {
				t.Errorf("expected status 400, got %d", rec.Code)
			}
		})
	}
}
```

### JWT Unit Tests

```go
package tests

import (
	"testing"
	"time"

	"taskapi/internal/auth"
)

func TestJWT_GenerateAndValidate(t *testing.T) {
	svc := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)

	token, err := svc.GenerateAccessToken("user-123", "test@example.com", "member")
	if err != nil {
		t.Fatalf("generating token: %v", err)
	}

	claims, err := svc.ValidateToken(token)
	if err != nil {
		t.Fatalf("validating token: %v", err)
	}

	if claims.UserID != "user-123" {
		t.Errorf("expected user_id user-123, got %s", claims.UserID)
	}
	if claims.Email != "test@example.com" {
		t.Errorf("expected email test@example.com, got %s", claims.Email)
	}
	if claims.Type != "access" {
		t.Errorf("expected type access, got %s", claims.Type)
	}
}

func TestJWT_ExpiredToken(t *testing.T) {
	svc := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		-1*time.Second, // Already expired
		7*24*time.Hour,
	)

	token, err := svc.GenerateAccessToken("user-123", "test@example.com", "member")
	if err != nil {
		t.Fatalf("generating token: %v", err)
	}

	_, err = svc.ValidateToken(token)
	if err == nil {
		t.Error("expected error for expired token, got nil")
	}
}

func TestJWT_InvalidSignature(t *testing.T) {
	svc1 := auth.NewJWTService(
		"secret-key-one-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)
	svc2 := auth.NewJWTService(
		"secret-key-two-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)

	token, _ := svc1.GenerateAccessToken("user-123", "test@example.com", "member")
	_, err := svc2.ValidateToken(token)
	if err == nil {
		t.Error("expected error for wrong secret, got nil")
	}
}

func TestJWT_RefreshTokenType(t *testing.T) {
	svc := auth.NewJWTService(
		"test-secret-key-that-is-at-least-32-characters",
		15*time.Minute,
		7*24*time.Hour,
	)

	token, _ := svc.GenerateRefreshToken("user-123", "test@example.com", "member")
	claims, err := svc.ValidateToken(token)
	if err != nil {
		t.Fatalf("validating refresh token: %v", err)
	}
	if claims.Type != "refresh" {
		t.Errorf("expected type refresh, got %s", claims.Type)
	}
}
```

### Validator Unit Tests

```go
package tests

import (
	"testing"

	"taskapi/pkg/validator"
)

func TestValidate_Required(t *testing.T) {
	type Input struct {
		Name string `json:"name" validate:"required"`
	}

	errs := validator.Validate(&Input{Name: ""})
	if !errs.HasErrors() {
		t.Error("expected validation error for empty required field")
	}

	errs = validator.Validate(&Input{Name: "Alice"})
	if errs.HasErrors() {
		t.Error("expected no errors for valid input")
	}
}

func TestValidate_Email(t *testing.T) {
	type Input struct {
		Email string `json:"email" validate:"required,email"`
	}

	errs := validator.Validate(&Input{Email: "not-an-email"})
	if !errs.HasErrors() {
		t.Error("expected error for invalid email")
	}

	errs = validator.Validate(&Input{Email: "valid@example.com"})
	if errs.HasErrors() {
		t.Error("expected no errors for valid email")
	}
}

func TestValidate_MinMax(t *testing.T) {
	type Input struct {
		Password string `json:"password" validate:"required,min=8,max=128"`
	}

	errs := validator.Validate(&Input{Password: "short"})
	if !errs.HasErrors() {
		t.Error("expected error for too-short password")
	}

	errs = validator.Validate(&Input{Password: "longenoughpassword"})
	if errs.HasErrors() {
		t.Error("expected no errors for valid password")
	}
}

func TestValidate_OneOf(t *testing.T) {
	type Input struct {
		Status string `json:"status" validate:"required,oneof=todo in_progress done"`
	}

	errs := validator.Validate(&Input{Status: "invalid"})
	if !errs.HasErrors() {
		t.Error("expected error for invalid status")
	}

	errs = validator.Validate(&Input{Status: "todo"})
	if errs.HasErrors() {
		t.Error("expected no errors for valid status")
	}
}
```

### Running Tests

```bash
# Run all tests
go test ./...

# Run with verbose output
go test -v ./...

# Run a specific test
go test -v -run TestRegister_Success ./tests/

# Run with race detector
go test -race ./...

# Run with coverage
go test -cover ./...
```

### Node.js/Express Comparison

In Express, you would use Jest or Mocha with `supertest`:

```javascript
// Node.js with Jest + supertest
const request = require('supertest');
const app = require('../src/app');

describe('POST /api/v1/auth/register', () => {
  it('should register a new user', async () => {
    const res = await request(app)
      .post('/api/v1/auth/register')
      .send({ email: 'test@example.com', password: 'secure123', full_name: 'Test' });

    expect(res.status).toBe(201);
    expect(res.body.success).toBe(true);
  });
});
```

The Go version uses `httptest.NewRecorder()` and `httptest.NewRequest()` from the standard library -- no third-party test utilities needed. The test doubles (mocks) are plain structs that implement the interface. No mocking framework required.

---

## 16. Docker Deployment

Docker packages the application into a container that runs identically everywhere. We use a multi-stage build to produce a minimal image.

### `Dockerfile`

```dockerfile
# Stage 1: Build the Go binary
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy dependency files first (layer caching)
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build a static binary
# CGO_ENABLED=0 produces a binary with no C dependencies
# -ldflags="-s -w" strips debug info to reduce binary size
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/taskapi .

# Stage 2: Create minimal runtime image
FROM alpine:3.19

# Install CA certificates (needed for HTTPS calls, e.g., external APIs)
RUN apk --no-cache add ca-certificates

WORKDIR /app

# Copy the binary from the builder stage
COPY --from=builder /app/taskapi .

# Copy migration files
COPY --from=builder /app/internal/database/migrations ./internal/database/migrations

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Expose the application port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Run the binary
ENTRYPOINT ["./taskapi"]
```

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ENVIRONMENT=development
      - SERVER_PORT=8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=taskapi
      - DB_PASSWORD=taskapi_secret
      - DB_NAME=taskapi
      - DB_SSL_MODE=disable
      - DB_MAX_CONNS=10
      - DB_MIN_CONNS=2
      - JWT_SECRET=your-development-secret-key-at-least-32-characters-long
      - JWT_EXPIRATION=15m
      - JWT_REFRESH_EXPIRY=168h
      - LOG_LEVEL=debug
      - LOG_FORMAT=text
      - CORS_ORIGINS=http://localhost:3000
      - RATE_LIMIT_REQUESTS=100
      - RATE_LIMIT_WINDOW=1m
      - CLEANUP_INTERVAL=5m
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: taskapi
      POSTGRES_PASSWORD: taskapi_secret
      POSTGRES_DB: taskapi
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapi"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Build and Run

```bash
# Build and start everything
docker compose up --build

# Run in background
docker compose up --build -d

# View logs
docker compose logs -f api

# Stop everything
docker compose down

# Stop and remove data
docker compose down -v
```

### Image Size Comparison

```bash
# Check the final image size
docker images taskapi
# REPOSITORY   TAG       SIZE
# taskapi      latest    ~18MB
```

Compare this to a typical Node.js Docker image:

```dockerfile
# Node.js Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "src/server.js"]
# Final image: ~200-350MB
```

The Go image is about 18MB because the binary is statically compiled and self-contained. The Node.js image includes the entire Node.js runtime (100MB+) and `node_modules`.

### Multi-Stage Build Explanation

The Dockerfile uses two stages:

1. **Builder stage** (`golang:1.22-alpine`, ~350MB) -- Contains the Go toolchain, downloads dependencies, compiles the binary
2. **Runtime stage** (`alpine:3.19`, ~7MB) -- Contains only the compiled binary and CA certificates

The final image only includes the runtime stage. The builder stage is discarded. This is why the image is small.

### `go.mod`

```
module taskapi

go 1.22

require (
	github.com/lib/pq v1.10.9
	golang.org/x/crypto v0.21.0
)
```

Two dependencies. That is it.

---

## 17. API Documentation

### Endpoint Reference

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/health` | No | Liveness check |
| `GET` | `/ready` | No | Readiness check (verifies database) |
| `POST` | `/api/v1/auth/register` | No | Register a new user |
| `POST` | `/api/v1/auth/login` | No | Login and receive JWT tokens |
| `POST` | `/api/v1/auth/refresh` | No | Exchange refresh token for new access token |
| `POST` | `/api/v1/projects` | Yes | Create a project |
| `GET` | `/api/v1/projects` | Yes | List your projects (paginated) |
| `GET` | `/api/v1/projects/{id}` | Yes | Get a project by ID |
| `PUT` | `/api/v1/projects/{id}` | Yes | Update a project |
| `DELETE` | `/api/v1/projects/{id}` | Yes | Delete a project and its tasks |
| `POST` | `/api/v1/projects/{projectId}/tasks` | Yes | Create a task in a project |
| `GET` | `/api/v1/projects/{projectId}/tasks` | Yes | List tasks in a project (filtered, paginated) |
| `GET` | `/api/v1/tasks/{id}` | Yes | Get a task by ID |
| `PUT` | `/api/v1/tasks/{id}` | Yes | Update a task |
| `DELETE` | `/api/v1/tasks/{id}` | Yes | Delete a task |
| `GET` | `/api/v1/dashboard` | Yes | Get aggregated statistics |

### Request/Response Examples

**Register:**

```bash
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@example.com",
    "password": "securepass123",
    "full_name": "Alice Johnson"
  }'
```

Response (201):

```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 900,
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "alice@example.com",
      "full_name": "Alice Johnson",
      "role": "member",
      "is_active": true,
      "created_at": "2025-03-15T10:30:00Z",
      "updated_at": "2025-03-15T10:30:00Z"
    }
  }
}
```

**Create Project:**

```bash
curl -X POST http://localhost:8080/api/v1/projects \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -d '{
    "name": "Q1 Launch",
    "description": "Product launch tasks for Q1"
  }'
```

Response (201):

```json
{
  "success": true,
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "name": "Q1 Launch",
    "description": "Product launch tasks for Q1",
    "owner_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "active",
    "created_at": "2025-03-15T10:31:00Z",
    "updated_at": "2025-03-15T10:31:00Z"
  }
}
```

**Create Task:**

```bash
curl -X POST http://localhost:8080/api/v1/projects/660e8400.../tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -d '{
    "title": "Set up CI/CD pipeline",
    "description": "Configure GitHub Actions for automated deployments",
    "priority": "high",
    "due_date": "2025-03-31"
  }'
```

**List Tasks with Filters:**

```bash
curl "http://localhost:8080/api/v1/projects/660e8400.../tasks?status=todo&priority=high&page=1&per_page=10" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Response (200):

```json
{
  "success": true,
  "data": [
    {
      "id": "770e8400-e29b-41d4-a716-446655440002",
      "project_id": "660e8400-e29b-41d4-a716-446655440001",
      "title": "Set up CI/CD pipeline",
      "description": "Configure GitHub Actions for automated deployments",
      "status": "todo",
      "priority": "high",
      "assignee_id": null,
      "due_date": "2025-03-31",
      "created_at": "2025-03-15T10:32:00Z",
      "updated_at": "2025-03-15T10:32:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 10,
    "total": 1,
    "total_pages": 1
  }
}
```

**Dashboard:**

```bash
curl http://localhost:8080/api/v1/dashboard \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Response (200):

```json
{
  "success": true,
  "data": {
    "total_users": 5,
    "total_projects": 12,
    "total_tasks": 87,
    "overdue_tasks": 3
  }
}
```

---

## 18. What We Have Built

This chapter brought together every pattern from the real-world section of this guide. Here is how each chapter contributed:

| Chapter | What It Taught | Where It Appears in This API |
|---------|---------------|------------------------------|
| **23 -- Configuration** | Environment variables, config structs, validation at startup | `internal/config/config.go` -- Load(), validate(), defaults |
| **24 -- Custom Router** | Method-based dispatch, path parameters | `internal/router/router.go` -- Go 1.22 `http.ServeMux` with `GET /path/{id}` |
| **25 -- Middleware** | func(http.Handler) http.Handler, chaining | `internal/middleware/` -- Recovery, Logging, CORS, RateLimit, Auth |
| **26 -- Repository Pattern** | Interfaces, SQL, connection pools, compile-time checks | `internal/repository/` -- UserRepository, ProjectRepository, TaskRepository |
| **27 -- Authentication** | JWT, bcrypt, token refresh, auth middleware | `internal/auth/jwt.go` + `internal/middleware/auth.go` |
| **28 -- Structured Logging** | log/slog, JSON/text output, log levels | Logger passed to every layer, request logging middleware |
| **29 -- Validation & Responses** | Validate tags, standard response envelope | `pkg/validator/` + `pkg/response/` |
| **30 -- Graceful Shutdown** | Signal handling, connection draining, cleanup order | `main.go` -- run(), SIGTERM handler, ordered shutdown |
| **31 -- Concurrency** | Background workers, parallel queries, mutex | `internal/worker/cleanup.go` + `DashboardHandler.Overview` |

### Architecture Principles

The API follows these principles consistently:

1. **Dependency injection** -- Every layer receives its dependencies through constructors. Nothing reaches for global state.
2. **Interface-driven design** -- Repository interfaces decouple handlers from the database. Swap PostgreSQL for SQLite or a mock without changing handler code.
3. **Fail fast** -- Configuration validation, startup health checks, and nil checks catch problems before they cause runtime errors.
4. **Standard library first** -- Two external dependencies (`lib/pq`, `golang.org/x/crypto`). Everything else is `net/http`, `database/sql`, `log/slog`, `encoding/json`, `crypto/hmac`.

---

## 19. Key Takeaways

1. **Layered architecture keeps code organized.** Config, Database, Repository, Handler, Router, Middleware, Main. Each layer has a single responsibility and depends only on the layer below it.

2. **Interfaces enable testing.** Define repository interfaces and inject them into handlers. In tests, provide mock implementations. No database needed for unit tests.

3. **Fail fast on configuration errors.** Validate all configuration at startup. A missing JWT secret should crash the application immediately, not cause mysterious 500 errors hours later.

4. **Middleware is the Go pattern for cross-cutting concerns.** Recovery, logging, CORS, rate limiting, and authentication are all `func(http.Handler) http.Handler`. They compose cleanly and are reusable.

5. **Go 1.22's enhanced ServeMux eliminates the need for third-party routers** in most applications. `mux.HandleFunc("GET /users/{id}", handler)` covers the vast majority of routing needs.

6. **Graceful shutdown is non-negotiable for production.** Handle SIGINT/SIGTERM, drain active connections, stop background workers, close the database pool -- in that order.

7. **Structured logging with `log/slog` is production-ready** in the standard library. JSON format for machines, text format for humans.

8. **Concurrency should be used where it provides real benefit.** The dashboard handler runs 4 parallel queries because they are independent I/O operations. Do not add goroutines to sequential work.

9. **Consistent API responses build trust.** Every response has the same shape: `{ success, data, error, meta }`. Clients always know what to expect.

10. **Two dependencies is enough.** Go's standard library provides HTTP servers, JSON, cryptography, SQL, testing, and structured logging. You need the PostgreSQL driver and bcrypt. That is it.

11. **Docker multi-stage builds produce minimal images.** An 18MB container with a static Go binary versus a 200-350MB container with Node.js and node_modules.

12. **The `run()` pattern makes `main()` testable.** Move initialization logic into a function that returns an error. Keep `main()` as a thin wrapper.

---

## 20. Practice Exercises

### Exercise 1: Add WebSocket Notifications

Extend the API with real-time notifications:

- When a task is created, updated, or assigned, notify relevant users
- Use `golang.org/x/net/websocket` or `github.com/gorilla/websocket`
- Add a `/api/v1/ws` endpoint that upgrades to WebSocket after JWT auth
- Maintain a hub of active connections per user
- When a task event occurs, broadcast to connected project members

**Hint:** Create a `hub` struct with a map of `userID -> []*websocket.Conn`, protected by a `sync.RWMutex`. Handlers publish events to a channel, and a goroutine reads from the channel and broadcasts.

### Exercise 2: File Uploads for Task Attachments

Add file attachment support:

- `POST /api/v1/tasks/{id}/attachments` -- upload a file (multipart/form-data)
- `GET /api/v1/tasks/{id}/attachments` -- list attachments
- `GET /api/v1/attachments/{id}/download` -- download a file
- `DELETE /api/v1/attachments/{id}` -- delete an attachment
- Limit file size to 10MB. Restrict allowed MIME types.
- Add a new `attachments` migration and repository

**Hint:** Use `r.ParseMultipartForm(10 << 20)` and `r.FormFile("file")` to handle uploads.

### Exercise 3: Full-Text Search

Enhance task search with PostgreSQL full-text search:

- Add a `to_tsvector` GIN index on task `title` and `description`
- Create `GET /api/v1/search?q=keyword` that searches across all the user's tasks
- Return results ranked by relevance using `ts_rank`
- Highlight matching terms using `ts_headline`

### Exercise 4: Redis Caching Layer

Add a caching layer:

- Cache dashboard stats for 30 seconds
- Cache user profiles for 5 minutes
- Invalidate caches when underlying data changes
- Create a `CacheRepository` interface with `Get`, `Set`, `Delete`
- Add Redis to `docker-compose.yml`

**Hint:** Create a generic `Get[T any](ctx, key) (*T, error)` function that deserializes cached JSON.

### Exercise 5: API Versioning

Add version management:

- Support `v1` and `v2` simultaneously
- `v2` returns a different task format (e.g., `status` becomes `state`)
- Add `Deprecation: true` and `Sunset` headers to `v1` responses
- Create a version negotiation middleware

### Exercise 6: Comprehensive Test Suite

Build a thorough test suite:

- Unit tests for every handler (positive and negative cases)
- Unit tests for JWT (expired tokens, wrong signatures, wrong token type)
- Unit tests for the validator (every rule)
- Integration tests using `testcontainers-go` to spin up a real PostgreSQL
- Achieve greater than 80% coverage

**Hint:** For testcontainers:

```go
package integration

import (
	"context"
	"testing"

	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestAPI_Integration(t *testing.T) {
	ctx := context.Background()

	container, err := postgres.RunContainer(ctx,
		testcontainers.WithImage("postgres:16-alpine"),
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
	)
	if err != nil {
		t.Fatalf("starting postgres container: %v", err)
	}
	defer container.Terminate(ctx)

	connStr, err := container.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		t.Fatalf("getting connection string: %v", err)
	}

	// Use connStr to initialize your database and run tests
	_ = connStr
}
```

### Exercise 7: Metrics and Observability

Add Prometheus metrics:

- Request count by method, path, and status code
- Request duration histogram
- Active connections gauge
- Database connection pool stats
- Expose at `/metrics`

**Hint:** Use `github.com/prometheus/client_golang/prometheus` and `promhttp.Handler()`.

### Exercise 8: Role-Based Access Control (RBAC)

Implement fine-grained permissions:

- Define permissions: `project:create`, `project:read`, `task:create`, etc.
- Roles map to permission sets: admin gets all, member gets read/create
- Create `RequirePermission("task:create")` middleware
- Add project-level roles (owner, admin, member)

---

## Congratulations

You have built a complete production API. Over this chapter, you took every pattern from Chapters 23-31 and combined them into a single, working application with authentication, CRUD endpoints, pagination, filtering, parallel queries, background workers, graceful shutdown, Docker deployment, and comprehensive tests.

The fundamentals from Chapters 1-22 gave you the language. The patterns from Chapters 23-31 gave you the tools. This chapter showed you how they all fit together.

Go was designed for building exactly this kind of software: reliable, efficient, maintainable services that a team can understand, test, and deploy with confidence. Every pattern here exists because experienced Go developers learned -- often the hard way -- that these patterns prevent bugs, simplify debugging, and make systems resilient.

Build something real. Deploy it. Watch the logs. Fix the bugs. That is how the knowledge solidifies.

---

*This is Chapter 32, the capstone of the real-world section of the Go Complete Study Guide.*
