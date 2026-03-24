# Chapter 53: Dependency Injection in Go

## Table of Contents

1. [What Is Dependency Injection and Why It Matters](#1-what-is-dependency-injection-and-why-it-matters)
2. [Manual DI via Constructor Injection (The Go Way)](#2-manual-di-via-constructor-injection-the-go-way)
3. [Interface-Based Dependency Inversion](#3-interface-based-dependency-inversion)
4. [Struct Embedding vs Composition for DI](#4-struct-embedding-vs-composition-for-di)
5. [Google Wire (Compile-Time DI)](#5-google-wire-compile-time-di)
6. [Uber fx (Runtime DI)](#6-uber-fx-runtime-di)
7. [dig (Uber's Lower-Level DI Container)](#7-dig-ubers-lower-level-di-container)
8. [The Case For and Against DI Frameworks in Go](#8-the-case-for-and-against-di-frameworks-in-go)
9. [Testing with Dependency Injection](#9-testing-with-dependency-injection)
10. [Service Locator Anti-Pattern](#10-service-locator-anti-pattern)
11. [Organizing Large Applications with DI](#11-organizing-large-applications-with-di)
12. [Real-World Example: Building a Web App with Proper DI](#12-real-world-example-building-a-web-app-with-proper-di)
13. [Summary and Decision Guide](#13-summary-and-decision-guide)

---

## 1. What Is Dependency Injection and Why It Matters

### Definition

**Dependency Injection (DI)** is a design technique where an object receives its dependencies
from external code rather than creating them internally. Instead of a struct constructing the
services it needs, those services are "injected" -- typically through constructor parameters,
method arguments, or struct fields.

### The Core Problem

Consider this tightly coupled code:

```go
package order

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq"
)

type OrderService struct{}

func (s *OrderService) PlaceOrder(userID int, items []Item) error {
    // Hard-coded database connection -- impossible to test without a real DB
    db, err := sql.Open("postgres", "host=localhost dbname=shop sslmode=disable")
    if err != nil {
        return err
    }
    defer db.Close()

    // Hard-coded email sending -- sends real emails during tests!
    sendEmail(userID, "Your order has been placed")

    // Hard-coded logging -- no control over output
    log.Printf("Order placed for user %d", userID)

    _, err = db.Exec("INSERT INTO orders (user_id) VALUES ($1)", userID)
    return err
}

func sendEmail(userID int, msg string) {
    fmt.Printf("Sending email to user %d: %s\n", userID, msg)
}
```

Problems with this approach:

| Problem | Consequence |
|---------|-------------|
| Hard-coded DB connection | Cannot run unit tests without a live database |
| Direct email sending | Tests trigger real emails |
| No way to swap implementations | Cannot use in-memory DB for testing |
| Hidden dependencies | Caller has no idea what `PlaceOrder` touches |
| Tight coupling | Changing the DB driver requires modifying business logic |

### The DI Solution

```go
package order

type Repository interface {
    CreateOrder(userID int, items []Item) error
}

type Notifier interface {
    Notify(userID int, message string) error
}

type Logger interface {
    Info(msg string, args ...any)
}

type OrderService struct {
    repo     Repository
    notifier Notifier
    logger   Logger
}

// Constructor injection: all dependencies are explicit parameters.
func NewOrderService(repo Repository, notifier Notifier, logger Logger) *OrderService {
    return &OrderService{
        repo:     repo,
        notifier: notifier,
        logger:   logger,
    }
}

func (s *OrderService) PlaceOrder(userID int, items []Item) error {
    if err := s.repo.CreateOrder(userID, items); err != nil {
        return err
    }
    s.logger.Info("Order placed", "userID", userID)
    return s.notifier.Notify(userID, "Your order has been placed")
}
```

### Why DI Matters in Go

1. **Testability**: Swap real implementations for mocks, fakes, or stubs.
2. **Decoupling**: Business logic does not know about infrastructure details.
3. **Flexibility**: Swap Postgres for MySQL, email for SMS -- without touching core code.
4. **Explicit dependencies**: The constructor signature documents what a type needs.
5. **Single Responsibility**: Each component does one thing; wiring is separate.

> **Tip**: Go's implicit interface satisfaction makes DI especially natural. You never
> need to declare "implements" -- if a type has the right methods, it satisfies the
> interface automatically.

---

## 2. Manual DI via Constructor Injection (The Go Way)

The idiomatic Go approach to DI is **manual constructor injection**. No frameworks, no
containers, no magic -- just functions that accept their dependencies as parameters.

### Basic Pattern

```go
// repository.go
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) FindByID(id int) (*User, error) {
    var u User
    err := r.db.QueryRow("SELECT id, name, email FROM users WHERE id=$1", id).
        Scan(&u.ID, &u.Name, &u.Email)
    if err != nil {
        return nil, fmt.Errorf("find user %d: %w", id, err)
    }
    return &u, nil
}
```

```go
// service.go
type UserService struct {
    repo   *UserRepository
    cache  *RedisCache
    logger *slog.Logger
}

func NewUserService(repo *UserRepository, cache *RedisCache, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        cache:  cache,
        logger: logger,
    }
}

func (s *UserService) GetUser(id int) (*User, error) {
    // Try cache first
    if u, err := s.cache.Get(id); err == nil {
        return u, nil
    }
    u, err := s.repo.FindByID(id)
    if err != nil {
        return nil, err
    }
    s.cache.Set(id, u)
    return u, nil
}
```

```go
// main.go -- the "composition root"
func main() {
    // Create infrastructure
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    redisClient := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Wire everything together manually
    userRepo := NewUserRepository(db)
    cache := NewRedisCache(redisClient)
    userService := NewUserService(userRepo, cache, logger)

    // Build HTTP handlers
    handler := NewUserHandler(userService)

    // Start server
    http.Handle("/users/", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Functional Options Pattern for Optional Dependencies

When a struct has many optional dependencies or configuration, use the functional options
pattern:

```go
type Server struct {
    addr    string
    logger  Logger
    tracer  Tracer
    timeout time.Duration
    tls     *tls.Config
}

type Option func(*Server)

func WithLogger(l Logger) Option {
    return func(s *Server) { s.logger = l }
}

func WithTracer(t Tracer) Option {
    return func(s *Server) { s.tracer = t }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithTLS(cfg *tls.Config) Option {
    return func(s *Server) { s.tls = cfg }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        logger:  defaultLogger{},  // sensible default
        tracer:  noopTracer{},     // sensible default
        timeout: 30 * time.Second, // sensible default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage:
func main() {
    srv := NewServer(":8080",
        WithLogger(zapLogger),
        WithTimeout(60*time.Second),
    )
    srv.ListenAndServe()
}
```

### When Manual DI Becomes Painful

Manual DI works beautifully for small-to-medium projects. It becomes tedious when:

- You have **50+ constructors** to wire together.
- Adding a dependency to one constructor forces changes through a long chain.
- Multiple `main` packages (CLI, server, worker) need slightly different wiring.
- The `main()` function grows to 200+ lines of pure wiring code.

This is where DI frameworks start to make sense -- not because manual DI is wrong,
but because the wiring code itself becomes a maintenance burden.

---

## 3. Interface-Based Dependency Inversion

### The Dependency Inversion Principle

The "D" in SOLID: high-level modules should not depend on low-level modules. Both should
depend on abstractions.

```
WITHOUT DI:                        WITH DI:
+----------------+                 +----------------+
| OrderService   |                 | OrderService   |
|                |                 |                |
| depends on     |                 | depends on     |
|    v           |                 |    v           |
| PostgresRepo   |                 | Repository     | <-- interface
+----------------+                 +-------+--------+
                                           |
                                   implements
                                           |
                                   +-------+--------+
                                   | PostgresRepo   |
                                   +----------------+
```

### Defining Interfaces Where They Are Used

In Go, the convention is to define interfaces in the **consumer** package, not the
**provider** package. This is the opposite of languages like Java.

```go
// WRONG: Defining the interface in the implementation package
package postgres

type UserRepository interface { // Don't do this
    FindByID(id int) (*User, error)
    Save(u *User) error
}

type PGUserRepository struct { ... }
```

```go
// RIGHT: Define the interface where it's consumed
package user

// Only the methods this package actually uses
type Repository interface {
    FindByID(id int) (*User, error)
    Save(u *User) error
}

type Service struct {
    repo Repository
}

func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}
```

```go
// The implementation in its own package, knows nothing about the interface
package postgres

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) FindByID(id int) (*user.User, error) { ... }
func (r *UserRepository) Save(u *user.User) error { ... }
```

### Interface Segregation

Keep interfaces small. Go idiom: **accept interfaces, return structs**.

```go
// BAD: One massive interface
type UserStore interface {
    FindByID(id int) (*User, error)
    FindByEmail(email string) (*User, error)
    FindAll() ([]*User, error)
    Save(u *User) error
    Delete(id int) error
    UpdatePassword(id int, hash string) error
    ListByRole(role string) ([]*User, error)
    Count() (int, error)
}
```

```go
// GOOD: Small, focused interfaces
type UserReader interface {
    FindByID(id int) (*User, error)
}

type UserWriter interface {
    Save(u *User) error
}

type UserDeleter interface {
    Delete(id int) error
}

// Compose when you need multiple capabilities
type UserReadWriter interface {
    UserReader
    UserWriter
}
```

This way, each consumer depends only on the methods it actually calls:

```go
type ProfileService struct {
    users UserReader // only needs to read users
}

type RegistrationService struct {
    users UserWriter // only needs to create users
}

type AdminService struct {
    users UserReadWriter // needs both
}
```

### The Role of `context.Context`

Always propagate `context.Context` in interface methods that perform I/O:

```go
type OrderRepository interface {
    Create(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
    List(ctx context.Context, filter OrderFilter) ([]*Order, error)
}
```

> **Warning**: Never store `context.Context` in a struct field. Always pass it as the
> first parameter to methods.

---

## 4. Struct Embedding vs Composition for DI

### Struct Embedding

Go allows embedding one struct inside another, which promotes all of the embedded type's
methods:

```go
type BaseLogger struct{}

func (l BaseLogger) Info(msg string)  { fmt.Println("[INFO]", msg) }
func (l BaseLogger) Error(msg string) { fmt.Println("[ERROR]", msg) }

type AuditLogger struct {
    BaseLogger // embedded -- promotes Info() and Error()
    auditDB *sql.DB
}

func (l AuditLogger) Info(msg string) {
    l.BaseLogger.Info(msg) // call the embedded method
    // also write to audit log
    l.auditDB.Exec("INSERT INTO audit_log (message) VALUES ($1)", msg)
}
```

### Composition (Field-Based DI)

```go
type AuditLogger struct {
    logger  Logger   // field, not embedded
    auditDB *sql.DB
}

func NewAuditLogger(logger Logger, db *sql.DB) *AuditLogger {
    return &AuditLogger{logger: logger, auditDB: db}
}

func (l *AuditLogger) Info(msg string) {
    l.logger.Info(msg)
    l.auditDB.Exec("INSERT INTO audit_log (message) VALUES ($1)", msg)
}
```

### Comparison

| Aspect | Embedding | Composition |
|--------|-----------|-------------|
| Method promotion | Automatic | Must delegate manually |
| Explicitness | Implicit -- easy to miss | Explicit -- clear delegation |
| Interface satisfaction | Embedded type's methods count | Only explicitly defined methods |
| Testing | Harder to mock the embedded part | Easy to inject a mock |
| Coupling | Tighter -- tied to a concrete type | Looser -- tied to an interface |
| Use case | Reusing behavior (mixins) | Dependency injection |

### When Embedding Breaks DI

```go
// Problematic: embedding a concrete type
type Handler struct {
    *sql.DB // promotes all sql.DB methods directly
}

// Now Handler satisfies any interface that sql.DB satisfies,
// which is almost certainly not what you want.
// Also impossible to mock *sql.DB for testing.
```

```go
// Better: compose with an interface
type Handler struct {
    db Database // interface, easy to mock
}

type Database interface {
    QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
    ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
}
```

### The Embedding-for-Defaults Trick

One legitimate DI use of embedding is providing default interface implementations
in tests:

```go
// In test code:
type mockRepo struct {
    UserRepository // embed the interface itself
}

// Only override what you need:
func (m *mockRepo) FindByID(id int) (*User, error) {
    return &User{ID: id, Name: "Test User"}, nil
}

// All other methods will panic if called (nil interface methods),
// which is actually useful -- it tells you which methods your code
// actually uses.
```

> **Tip**: Prefer composition (fields + interfaces) for production DI. Reserve embedding
> for test helpers and genuine behavior inheritance.

---

## 5. Google Wire (Compile-Time DI)

[Google Wire](https://github.com/google/wire) is a compile-time dependency injection
framework. It generates plain Go code -- no reflection, no runtime overhead.

### Installation

```bash
go install github.com/google/wire/cmd/wire@latest
```

### Core Concepts

Wire has two fundamental concepts:

1. **Providers**: Functions that produce values (your constructors).
2. **Injectors**: Functions that Wire generates to wire providers together.

### Providers

A provider is any function that returns a value:

```go
// These are all valid providers:
func NewDB(cfg *Config) (*sql.DB, error) { ... }
func NewUserRepo(db *sql.DB) *UserRepository { ... }
func NewUserService(repo *UserRepository, logger *slog.Logger) *UserService { ... }
func NewLogger() *slog.Logger { ... }
func LoadConfig() (*Config, error) { ... }
```

Wire inspects the **types** of parameters and return values to build a dependency graph.

### Injectors

An injector is a function you declare; Wire fills in the body. You write it in a file
named `wire.go` (by convention) with a `//go:build wireinject` build tag:

```go
//go:build wireinject
// +build wireinject

package main

import "github.com/google/wire"

func InitializeApp() (*App, error) {
    wire.Build(
        LoadConfig,
        NewDB,
        NewUserRepo,
        NewUserService,
        NewLogger,
        NewApp,
    )
    return nil, nil // Wire will replace this body
}
```

Run `wire` in the package directory, and it generates `wire_gen.go`:

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate go run -mod=mod github.com/google/wire/cmd/wire
//go:build !wireinject
// +build !wireinject

package main

func InitializeApp() (*App, error) {
    config, err := LoadConfig()
    if err != nil {
        return nil, err
    }
    db, err := NewDB(config)
    if err != nil {
        return nil, err
    }
    userRepo := NewUserRepo(db)
    logger := NewLogger()
    userService := NewUserService(userRepo, logger)
    app := NewApp(userService)
    return app, nil
}
```

Notice: the generated code is plain Go. No reflection, no containers. You can read it,
debug it, and step through it.

### Provider Sets

Group related providers together using `wire.NewSet`:

```go
package database

import "github.com/google/wire"

var Set = wire.NewSet(
    NewDB,
    NewUserRepository,
    NewOrderRepository,
    NewProductRepository,
)
```

```go
package service

import "github.com/google/wire"

var Set = wire.NewSet(
    NewUserService,
    NewOrderService,
    NewProductService,
)
```

```go
// wire.go
func InitializeApp() (*App, error) {
    wire.Build(
        config.Set,
        database.Set,
        service.Set,
        handler.Set,
        NewApp,
    )
    return nil, nil
}
```

### Binding Interfaces

When a provider returns a concrete type but a consumer needs an interface, use
`wire.Bind`:

```go
package database

import "github.com/google/wire"

// Tell Wire: when someone needs a user.Repository, use *PGUserRepository
var Set = wire.NewSet(
    NewPGUserRepository,
    wire.Bind(new(user.Repository), new(*PGUserRepository)),
)
```

Full example:

```go
// interfaces
package user

type Repository interface {
    FindByID(ctx context.Context, id int) (*User, error)
    Save(ctx context.Context, u *User) error
}

type Service struct {
    repo Repository
}

func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}
```

```go
// implementation
package postgres

type UserRepo struct {
    db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
    return &UserRepo{db: db}
}

func (r *UserRepo) FindByID(ctx context.Context, id int) (*user.User, error) { ... }
func (r *UserRepo) Save(ctx context.Context, u *user.User) error { ... }
```

```go
// wire.go
var repoSet = wire.NewSet(
    postgres.NewUserRepo,
    wire.Bind(new(user.Repository), new(*postgres.UserRepo)),
)

func InitializeUserService() (*user.Service, error) {
    wire.Build(
        provideDB,
        repoSet,
        user.NewService,
    )
    return nil, nil
}
```

### Struct Providers

For simple structs that just group dependencies, use `wire.Struct`:

```go
type App struct {
    Server      *http.Server
    UserService *user.Service
    Logger      *slog.Logger
}

var AppSet = wire.NewSet(
    wire.Struct(new(App), "*"), // "*" means inject all fields
)

// Or inject only specific fields:
var AppSet = wire.NewSet(
    wire.Struct(new(App), "Server", "Logger"), // only these two
)
```

### Cleanup Functions

Providers can return a cleanup function as the second return value (before the error):

```go
func NewDB(cfg *Config) (*sql.DB, func(), error) {
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        return nil, nil, err
    }

    cleanup := func() {
        db.Close()
    }

    return db, cleanup, nil
}

func NewRedisClient(cfg *Config) (*redis.Client, func(), error) {
    client := redis.NewClient(&redis.Options{Addr: cfg.RedisAddr})
    cleanup := func() {
        client.Close()
    }
    return client, cleanup, nil
}
```

Wire chains cleanup functions automatically in the generated code:

```go
// Generated by Wire:
func InitializeApp() (*App, func(), error) {
    config, err := LoadConfig()
    if err != nil {
        return nil, nil, err
    }
    db, dbCleanup, err := NewDB(config)
    if err != nil {
        return nil, nil, err
    }
    redisClient, redisCleanup, err := NewRedisClient(config)
    if err != nil {
        dbCleanup() // clean up previously created resources
        return nil, nil, err
    }

    // ... more wiring ...

    app := NewApp(db, redisClient)
    return app, func() {
        redisCleanup()
        dbCleanup()
    }, nil
}
```

Usage in main:

```go
func main() {
    app, cleanup, err := InitializeApp()
    if err != nil {
        log.Fatal(err)
    }
    defer cleanup() // all resources cleaned up on exit

    app.Run()
}
```

### Advanced Wire Usage

#### Values and Fields

Provide a specific value directly:

```go
func InitializeApp() (*App, error) {
    wire.Build(
        wire.Value(&Config{Port: 8080, Debug: true}),
        NewApp,
    )
    return nil, nil
}

// Or extract a field from a struct:
func InitializeApp() (*App, error) {
    wire.Build(
        LoadConfig,
        wire.FieldsOf(new(*Config), "DB", "Redis"), // extract specific fields
        NewDB,
        NewRedisClient,
        NewApp,
    )
    return nil, nil
}
```

#### Multiple Injectors

You can have different injectors for different entry points:

```go
//go:build wireinject

package main

import "github.com/google/wire"

// For the API server
func InitializeAPIServer() (*APIServer, func(), error) {
    wire.Build(
        config.Set,
        database.Set,
        service.Set,
        api.Set,
    )
    return nil, nil, nil
}

// For the background worker
func InitializeWorker() (*Worker, func(), error) {
    wire.Build(
        config.Set,
        database.Set,
        service.Set,
        worker.Set,
    )
    return nil, nil, nil
}

// For the CLI tool
func InitializeCLI() (*CLI, func(), error) {
    wire.Build(
        config.Set,
        database.Set,
        cli.Set,
    )
    return nil, nil, nil
}
```

#### Distinguishing Same Types with wire.NewSet and Naming

When you have two values of the same type (e.g., two `*sql.DB` for read and write
replicas), wrap them in distinct types:

```go
type ReadDB struct{ *sql.DB }
type WriteDB struct{ *sql.DB }

func NewReadDB(cfg *Config) (*ReadDB, func(), error) {
    db, err := sql.Open("postgres", cfg.ReadDatabaseURL)
    if err != nil {
        return nil, nil, err
    }
    return &ReadDB{db}, func() { db.Close() }, nil
}

func NewWriteDB(cfg *Config) (*WriteDB, func(), error) {
    db, err := sql.Open("postgres", cfg.WriteDatabaseURL)
    if err != nil {
        return nil, nil, err
    }
    return &WriteDB{db}, func() { db.Close() }, nil
}

func NewUserRepo(readDB *ReadDB, writeDB *WriteDB) *UserRepo {
    return &UserRepo{read: readDB.DB, write: writeDB.DB}
}
```

### Wire Project Structure

```
myapp/
  cmd/
    api/
      main.go          # calls InitializeApp()
      wire.go          # injector declarations (wireinject build tag)
      wire_gen.go      # generated (committed to VCS)
    worker/
      main.go
      wire.go
      wire_gen.go
  internal/
    config/
      config.go
      wire.go          # provider set
    database/
      postgres.go
      wire.go          # provider set with Bind
    service/
      user.go
      order.go
      wire.go          # provider set
    handler/
      user.go
      order.go
      wire.go          # provider set
```

> **Tip**: Always commit `wire_gen.go` to version control. This way, people who don't
> have Wire installed can still build your project.

---

## 6. Uber fx (Runtime DI)

[Uber fx](https://github.com/uber-go/fx) is a runtime dependency injection framework
built on top of `dig`. It manages application lifecycle and is designed for large-scale
services.

### Installation

```bash
go get go.uber.org/fx
```

### Basic Usage

```go
package main

import (
    "context"
    "net/http"

    "go.uber.org/fx"
    "go.uber.org/zap"
)

func main() {
    fx.New(
        fx.Provide(
            NewLogger,
            NewConfig,
            NewDB,
            NewUserRepository,
            NewUserService,
            NewHTTPServer,
        ),
        fx.Invoke(StartServer),
    ).Run()
}

func NewLogger() (*zap.Logger, error) {
    return zap.NewProduction()
}

func NewConfig() *Config {
    return &Config{
        Port:        ":8080",
        DatabaseURL: os.Getenv("DATABASE_URL"),
    }
}

func NewDB(cfg *Config, logger *zap.Logger) (*sql.DB, error) {
    logger.Info("Connecting to database", zap.String("url", cfg.DatabaseURL))
    return sql.Open("postgres", cfg.DatabaseURL)
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func NewUserService(repo *UserRepository, logger *zap.Logger) *UserService {
    return &UserService{repo: repo, logger: logger}
}

func NewHTTPServer(svc *UserService, logger *zap.Logger) *http.Server {
    mux := http.NewServeMux()
    mux.HandleFunc("/users", svc.HandleListUsers)
    return &http.Server{Addr: ":8080", Handler: mux}
}

func StartServer(lc fx.Lifecycle, server *http.Server, logger *zap.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            logger.Info("Starting HTTP server")
            go server.ListenAndServe()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("Stopping HTTP server")
            return server.Shutdown(ctx)
        },
    })
}
```

### Modules

Group providers into reusable modules:

```go
package database

import "go.uber.org/fx"

var Module = fx.Module("database",
    fx.Provide(
        NewDB,
        NewUserRepository,
        NewOrderRepository,
        NewProductRepository,
    ),
)
```

```go
package service

import "go.uber.org/fx"

var Module = fx.Module("service",
    fx.Provide(
        NewUserService,
        NewOrderService,
        NewProductService,
    ),
)
```

```go
package handler

import "go.uber.org/fx"

var Module = fx.Module("handler",
    fx.Provide(
        NewUserHandler,
        NewOrderHandler,
        NewRouter,
    ),
)
```

```go
// main.go
func main() {
    fx.New(
        config.Module,
        database.Module,
        service.Module,
        handler.Module,
        fx.Invoke(StartServer),
    ).Run()
}
```

### Lifecycle Hooks

fx manages the complete application lifecycle:

```go
func RegisterDBLifecycle(lc fx.Lifecycle, db *sql.DB, logger *zap.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            logger.Info("Pinging database...")
            return db.PingContext(ctx)
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("Closing database connection...")
            return db.Close()
        },
    })
}

func RegisterCacheLifecycle(lc fx.Lifecycle, cache *redis.Client, logger *zap.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            return cache.Ping(ctx).Err()
        },
        OnStop: func(ctx context.Context) error {
            return cache.Close()
        },
    })
}

// In main:
fx.New(
    fx.Provide(NewDB, NewRedisClient),
    fx.Invoke(RegisterDBLifecycle, RegisterCacheLifecycle),
)
```

Lifecycle hooks are executed in order:
- `OnStart`: Called in the order they are registered.
- `OnStop`: Called in **reverse** order (LIFO), ensuring proper teardown.

### Named Values

When you have multiple values of the same type, use `fx.Named`:

```go
// Provide named values
fx.Provide(
    fx.Annotate(
        NewReadDB,
        fx.ResultTags(`name:"readDB"`),
    ),
    fx.Annotate(
        NewWriteDB,
        fx.ResultTags(`name:"writeDB"`),
    ),
)

// Consume named values
fx.Provide(
    fx.Annotate(
        NewUserRepository,
        fx.ParamTags(`name:"readDB"`, `name:"writeDB"`),
    ),
)

func NewReadDB(cfg *Config) (*sql.DB, error) {
    return sql.Open("postgres", cfg.ReadDBURL)
}

func NewWriteDB(cfg *Config) (*sql.DB, error) {
    return sql.Open("postgres", cfg.WriteDBURL)
}

func NewUserRepository(readDB *sql.DB, writeDB *sql.DB) *UserRepository {
    return &UserRepository{read: readDB, write: writeDB}
}
```

Alternatively, use struct tags with `fx.In` and `fx.Out`:

```go
type DBParams struct {
    fx.In

    ReadDB  *sql.DB `name:"readDB"`
    WriteDB *sql.DB `name:"writeDB"`
}

func NewUserRepository(params DBParams) *UserRepository {
    return &UserRepository{
        read:  params.ReadDB,
        write: params.WriteDB,
    }
}

type DBResult struct {
    fx.Out

    ReadDB  *sql.DB `name:"readDB"`
    WriteDB *sql.DB `name:"writeDB"`
}

func ProvideDBs(cfg *Config) (DBResult, error) {
    readDB, err := sql.Open("postgres", cfg.ReadDBURL)
    if err != nil {
        return DBResult{}, err
    }
    writeDB, err := sql.Open("postgres", cfg.WriteDBURL)
    if err != nil {
        readDB.Close()
        return DBResult{}, err
    }
    return DBResult{ReadDB: readDB, WriteDB: writeDB}, nil
}
```

### Groups

Groups collect multiple values of the same type into a slice:

```go
// Each handler provides itself to the "routes" group
func NewUserHandler(svc *UserService) RouteResult {
    return RouteResult{Route: Route{Path: "/users", Handler: svc.Handle}}
}

func NewOrderHandler(svc *OrderService) RouteResult {
    return RouteResult{Route: Route{Path: "/orders", Handler: svc.Handle}}
}

type RouteResult struct {
    fx.Out
    Route Route `group:"routes"`
}

// The router collects all routes from the group
type RouterParams struct {
    fx.In
    Routes []Route `group:"routes"`
}

func NewRouter(params RouterParams) *http.ServeMux {
    mux := http.NewServeMux()
    for _, route := range params.Routes {
        mux.HandleFunc(route.Path, route.Handler)
    }
    return mux
}
```

### Supply vs Provide

`fx.Supply` is for providing pre-built values; `fx.Provide` is for constructor functions:

```go
fx.New(
    // Supply an already-built value
    fx.Supply(&Config{Port: 8080, Debug: true}),

    // Provide a constructor
    fx.Provide(NewDB),
)
```

### Interface Binding with fx.Annotate

```go
fx.Provide(
    fx.Annotate(
        NewPostgresUserRepo,
        fx.As(new(UserRepository)), // bind concrete type to interface
    ),
)
```

### Decorators

Wrap an existing provided type:

```go
fx.Decorate(func(logger *zap.Logger) *zap.Logger {
    return logger.With(zap.String("module", "api"))
})
```

### Error Handling and Validation

```go
app := fx.New(
    fx.Provide(NewDB),
    fx.Provide(NewService),
    fx.Invoke(StartServer),
)

// Validate the dependency graph without starting the app
if err := app.Err(); err != nil {
    log.Fatalf("Dependency injection error: %v", err)
}

// Start the app with context
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

if err := app.Start(ctx); err != nil {
    log.Fatalf("Failed to start: %v", err)
}
```

### fx vs Wire Comparison

| Feature | Wire | fx |
|---------|------|-----|
| **Injection time** | Compile-time (code gen) | Runtime (reflection) |
| **Performance** | Zero overhead at runtime | Minimal overhead at startup |
| **Debugging** | Read generated code | Stack traces, fx.Annotate |
| **Error detection** | At `go generate` time | At application startup |
| **Lifecycle management** | None (just wiring) | Full lifecycle (start/stop) |
| **Learning curve** | Moderate (build tags, codegen) | Moderate (annotations, groups) |
| **Binary size** | No additional dependency | Includes fx + dig |
| **IDE support** | Generated code is navigable | Reflection makes jumping harder |
| **Community** | Google, moderate adoption | Uber, wide adoption |
| **Best for** | Libraries, simple services | Large services with lifecycle |

> **Tip**: Wire is often preferred for its transparency (generated code is plain Go).
> fx is often preferred when you need lifecycle management and your team is comfortable
> with runtime DI.

---

## 7. dig (Uber's Lower-Level DI Container)

[dig](https://github.com/uber-go/dig) is the DI container that underlies fx. Use it
when you want container-based DI without fx's lifecycle management.

### Basic Usage

```go
package main

import (
    "fmt"
    "go.uber.org/dig"
)

type Config struct {
    DSN string
}

type DB struct {
    dsn string
}

func NewConfig() *Config {
    return &Config{DSN: "postgres://localhost/mydb"}
}

func NewDB(cfg *Config) *DB {
    return &DB{dsn: cfg.DSN}
}

type UserRepository struct {
    db *DB
}

func NewUserRepository(db *DB) *UserRepository {
    return &UserRepository{db: db}
}

func main() {
    c := dig.New()

    // Register providers
    c.Provide(NewConfig)
    c.Provide(NewDB)
    c.Provide(NewUserRepository)

    // Resolve and use
    err := c.Invoke(func(repo *UserRepository) {
        fmt.Printf("Got repo with DB: %s\n", repo.db.dsn)
    })
    if err != nil {
        panic(err)
    }
}
```

### Named Dependencies

```go
type DBParams struct {
    dig.In

    Primary  *sql.DB `name:"primary"`
    Replica  *sql.DB `name:"replica"`
}

type PrimaryDBResult struct {
    dig.Out

    DB *sql.DB `name:"primary"`
}

type ReplicaDBResult struct {
    dig.Out

    DB *sql.DB `name:"replica"`
}

func ProvidePrimaryDB(cfg *Config) (PrimaryDBResult, error) {
    db, err := sql.Open("postgres", cfg.PrimaryDSN)
    return PrimaryDBResult{DB: db}, err
}

func ProvideReplicaDB(cfg *Config) (ReplicaDBResult, error) {
    db, err := sql.Open("postgres", cfg.ReplicaDSN)
    return ReplicaDBResult{DB: db}, err
}

func NewRepository(params DBParams) *Repository {
    return &Repository{
        write: params.Primary,
        read:  params.Replica,
    }
}
```

### Groups in dig

```go
type HandlerResult struct {
    dig.Out

    Handler Handler `group:"handlers"`
}

type HandlersParam struct {
    dig.In

    Handlers []Handler `group:"handlers"`
}

func ProvideUserHandler() HandlerResult {
    return HandlerResult{Handler: &UserHandler{}}
}

func ProvideOrderHandler() HandlerResult {
    return HandlerResult{Handler: &OrderHandler{}}
}

func NewRouter(params HandlersParam) *Router {
    r := &Router{}
    for _, h := range params.Handlers {
        r.Register(h)
    }
    return r
}
```

### Optional Dependencies

```go
type Params struct {
    dig.In

    Logger *zap.Logger    `optional:"true"` // won't error if not provided
    Tracer *trace.Tracer  `optional:"true"`
}

func NewService(params Params) *Service {
    logger := params.Logger
    if logger == nil {
        logger = zap.NewNop() // fallback
    }
    return &Service{logger: logger, tracer: params.Tracer}
}
```

### Visualizing the Dependency Graph

dig can output a DOT graph:

```go
c := dig.New()
// ... register providers ...

// Visualize
if err := dig.Visualize(c, os.Stdout); err != nil {
    log.Fatal(err)
}
// Pipe output to: dot -Tpng -o deps.png
```

### When to Use dig vs fx

- Use **dig** when you want a standalone DI container without lifecycle management.
- Use **fx** when you want the full application framework (lifecycle, modules, signals).
- Use **neither** when manual DI is sufficient.

---

## 8. The Case For and Against DI Frameworks in Go

### Arguments FOR DI Frameworks

1. **Reduced boilerplate**: In large applications with 100+ constructors, manual wiring
   is tedious and error-prone.

2. **Automatic graph resolution**: The framework figures out the correct initialization
   order.

3. **Lifecycle management** (fx): Automatic startup/shutdown sequencing is hard to get
   right manually.

4. **Consistency**: New team members follow the same pattern; adding a dependency is
   mechanical.

5. **Multiple entry points**: When you have API, worker, CLI, and migration binaries
   sharing services, a framework prevents duplication.

### Arguments AGAINST DI Frameworks

1. **Magic**: Dependencies are resolved through reflection or code generation, making
   it harder to follow the flow.

2. **Go philosophy**: The Go community generally prefers explicit, readable code over
   frameworks.

3. **Debugging difficulty**: Runtime DI errors (fx/dig) appear at startup, not compile
   time. Wire errors appear during generation, not during normal builds.

4. **Learning curve**: New contributors must learn the framework in addition to Go.

5. **Overhead for small projects**: For a service with 10-20 types, manual DI is
   perfectly manageable.

6. **IDE navigation**: Reflection-based DI breaks "Go to Definition" in IDEs.

### Decision Matrix

| Project Size | Team Size | Recommendation |
|-------------|-----------|----------------|
| Small (< 20 types) | 1-3 | Manual DI |
| Medium (20-50 types) | 3-8 | Manual DI or Wire |
| Large (50-200 types) | 5-20 | Wire or fx |
| Very Large (200+ types) | 10+ | fx (with lifecycle) |
| Library/SDK | Any | Manual DI only |

### The Pragmatic Approach

```
Start with manual DI.
When main() gets painful, consider Wire.
When you need lifecycle management, consider fx.
Never use a DI framework in a library.
```

> **Warning**: A DI framework is a tool to manage complexity, not a substitute for good
> design. If your dependency graph is a mess, a framework just makes it a well-managed
> mess. Fix the design first.

---

## 9. Testing with Dependency Injection

DI's greatest benefit is testability. Here's how to use it effectively.

### Mocks, Fakes, and Stubs

| Test Double | Purpose | Example |
|------------|---------|---------|
| **Mock** | Verifies interactions (calls made) | mockgen-generated implementation |
| **Fake** | Working implementation with shortcuts | In-memory database |
| **Stub** | Returns predetermined responses | Hard-coded return values |

### Manual Stubs

The simplest approach -- no tools needed:

```go
// Interface in the service package
type EmailSender interface {
    Send(to, subject, body string) error
}

// Stub for testing
type stubEmailSender struct {
    sentEmails []sentEmail
    shouldFail bool
}

type sentEmail struct {
    to, subject, body string
}

func (s *stubEmailSender) Send(to, subject, body string) error {
    if s.shouldFail {
        return fmt.Errorf("email sending failed")
    }
    s.sentEmails = append(s.sentEmails, sentEmail{to, subject, body})
    return nil
}

// Test
func TestOrderService_PlaceOrder(t *testing.T) {
    emailer := &stubEmailSender{}
    repo := &fakeOrderRepo{}
    svc := NewOrderService(repo, emailer)

    err := svc.PlaceOrder(ctx, Order{UserEmail: "test@example.com", Total: 99.99})
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if len(emailer.sentEmails) != 1 {
        t.Fatalf("expected 1 email, got %d", len(emailer.sentEmails))
    }
    if emailer.sentEmails[0].to != "test@example.com" {
        t.Errorf("email sent to wrong address: %s", emailer.sentEmails[0].to)
    }
}
```

### Fakes (In-Memory Implementations)

```go
// Fake repository that stores data in memory
type fakeUserRepo struct {
    mu    sync.Mutex
    users map[int]*User
    nextID int
}

func newFakeUserRepo() *fakeUserRepo {
    return &fakeUserRepo{users: make(map[int]*User), nextID: 1}
}

func (r *fakeUserRepo) Save(ctx context.Context, u *User) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    if u.ID == 0 {
        u.ID = r.nextID
        r.nextID++
    }
    r.users[u.ID] = u
    return nil
}

func (r *fakeUserRepo) FindByID(ctx context.Context, id int) (*User, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    u, ok := r.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func (r *fakeUserRepo) Delete(ctx context.Context, id int) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    delete(r.users, id)
    return nil
}
```

### mockgen (GoMock)

[mockgen](https://github.com/uber-go/mock) generates mock implementations from interfaces.

```bash
go install go.uber.org/mock/mockgen@latest
```

Usage:

```go
// service.go
package user

//go:generate mockgen -destination=mocks/mock_repository.go -package=mocks . Repository

type Repository interface {
    FindByID(ctx context.Context, id int) (*User, error)
    Save(ctx context.Context, u *User) error
    Delete(ctx context.Context, id int) error
}
```

Run `go generate ./...` to create `mocks/mock_repository.go`.

Test with mockgen:

```go
package user_test

import (
    "context"
    "testing"

    "go.uber.org/mock/gomock"
    "myapp/user"
    "myapp/user/mocks"
)

func TestService_GetUser(t *testing.T) {
    ctrl := gomock.NewController(t)

    mockRepo := mocks.NewMockRepository(ctrl)

    // Set expectations
    expectedUser := &user.User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    mockRepo.EXPECT().
        FindByID(gomock.Any(), 1).
        Return(expectedUser, nil).
        Times(1)

    svc := user.NewService(mockRepo)

    got, err := svc.GetUser(context.Background(), 1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got.Name != "Alice" {
        t.Errorf("got name %q, want %q", got.Name, "Alice")
    }
}

func TestService_GetUser_NotFound(t *testing.T) {
    ctrl := gomock.NewController(t)

    mockRepo := mocks.NewMockRepository(ctrl)
    mockRepo.EXPECT().
        FindByID(gomock.Any(), 999).
        Return(nil, user.ErrNotFound)

    svc := user.NewService(mockRepo)

    _, err := svc.GetUser(context.Background(), 999)
    if err != user.ErrNotFound {
        t.Errorf("got error %v, want ErrNotFound", err)
    }
}
```

### moq

[moq](https://github.com/matryer/moq) generates simple, idiomatic mocks using function
fields:

```bash
go install github.com/matryer/moq@latest
```

```go
//go:generate moq -out mocks/user_repo_mock.go -pkg mocks . Repository
```

Generated mock:

```go
// mocks/user_repo_mock.go (generated)
package mocks

type RepositoryMock struct {
    FindByIDFunc func(ctx context.Context, id int) (*user.User, error)
    SaveFunc     func(ctx context.Context, u *user.User) error
    DeleteFunc   func(ctx context.Context, id int) error
}

func (m *RepositoryMock) FindByID(ctx context.Context, id int) (*user.User, error) {
    return m.FindByIDFunc(ctx, id)
}
// ...
```

Test with moq:

```go
func TestService_GetUser(t *testing.T) {
    repo := &mocks.RepositoryMock{
        FindByIDFunc: func(ctx context.Context, id int) (*user.User, error) {
            return &user.User{ID: id, Name: "Alice"}, nil
        },
    }

    svc := user.NewService(repo)
    got, err := svc.GetUser(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }
    if got.Name != "Alice" {
        t.Errorf("got %q, want %q", got.Name, "Alice")
    }
}
```

### counterfeiter

[counterfeiter](https://github.com/maxbrunsfeld/counterfeiter) generates fakes with
recording capabilities:

```bash
go install github.com/maxbrunsfeld/counterfeiter/v6@latest
```

```go
//go:generate counterfeiter -o fakes/fake_repository.go . Repository
```

Test with counterfeiter:

```go
func TestService_DeleteUser(t *testing.T) {
    fakeRepo := &fakes.FakeRepository{}
    fakeRepo.DeleteReturns(nil)

    svc := user.NewService(fakeRepo)
    err := svc.DeleteUser(context.Background(), 42)
    if err != nil {
        t.Fatal(err)
    }

    // Verify interactions
    if fakeRepo.DeleteCallCount() != 1 {
        t.Errorf("expected 1 call to Delete, got %d", fakeRepo.DeleteCallCount())
    }

    ctx, id := fakeRepo.DeleteArgsForCall(0)
    if id != 42 {
        t.Errorf("deleted wrong user: got %d, want 42", id)
    }
    _ = ctx
}
```

### Mock Tool Comparison

| Feature | mockgen | moq | counterfeiter |
|---------|---------|-----|---------------|
| Style | Expectation-based | Function fields | Recording-based |
| Generated code | Complex, uses gomock | Simple, plain Go | Moderate, feature-rich |
| Call verification | EXPECT().Times(n) | Manual | CallCount(), ArgsForCall() |
| Learning curve | Moderate | Low | Low |
| Extra dependency | go.uber.org/mock | None at runtime | None at runtime |
| Community | Large | Medium | Medium (Cloud Foundry) |

### Table-Driven Tests with Injected Dependencies

The gold standard for Go testing -- combine DI with table-driven tests:

```go
func TestOrderService_CalculateTotal(t *testing.T) {
    tests := []struct {
        name          string
        items         []Item
        taxRate       float64
        discount      float64
        setupRepo     func(*mocks.RepositoryMock)
        setupPricer   func(*mocks.PricerMock)
        wantTotal     float64
        wantErr       bool
        wantErrMsg    string
    }{
        {
            name:     "single item no discount",
            items:    []Item{{ProductID: 1, Quantity: 2}},
            taxRate:  0.1,
            discount: 0,
            setupRepo: func(m *mocks.RepositoryMock) {
                // no setup needed for this test
            },
            setupPricer: func(m *mocks.PricerMock) {
                m.GetPriceFunc = func(ctx context.Context, productID int) (float64, error) {
                    return 25.00, nil
                }
            },
            wantTotal: 55.00, // (25 * 2) * 1.1
        },
        {
            name:     "multiple items with discount",
            items:    []Item{{ProductID: 1, Quantity: 1}, {ProductID: 2, Quantity: 3}},
            taxRate:  0.08,
            discount: 10.00,
            setupRepo: func(m *mocks.RepositoryMock) {},
            setupPricer: func(m *mocks.PricerMock) {
                m.GetPriceFunc = func(ctx context.Context, productID int) (float64, error) {
                    prices := map[int]float64{1: 100.00, 2: 50.00}
                    return prices[productID], nil
                }
            },
            wantTotal: 259.20, // ((100 + 150) - 10) * 1.08
        },
        {
            name:     "pricer error",
            items:    []Item{{ProductID: 999, Quantity: 1}},
            taxRate:  0.1,
            discount: 0,
            setupRepo: func(m *mocks.RepositoryMock) {},
            setupPricer: func(m *mocks.PricerMock) {
                m.GetPriceFunc = func(ctx context.Context, productID int) (float64, error) {
                    return 0, fmt.Errorf("product not found")
                }
            },
            wantErr:    true,
            wantErrMsg: "product not found",
        },
        {
            name:     "empty cart",
            items:    []Item{},
            taxRate:  0.1,
            discount: 0,
            setupRepo:   func(m *mocks.RepositoryMock) {},
            setupPricer: func(m *mocks.PricerMock) {},
            wantTotal:   0,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := &mocks.RepositoryMock{}
            pricer := &mocks.PricerMock{}

            tt.setupRepo(repo)
            tt.setupPricer(pricer)

            svc := NewOrderService(repo, pricer, tt.taxRate)
            total, err := svc.CalculateTotal(context.Background(), tt.items, tt.discount)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error, got nil")
                }
                if tt.wantErrMsg != "" && !strings.Contains(err.Error(), tt.wantErrMsg) {
                    t.Errorf("error %q does not contain %q", err.Error(), tt.wantErrMsg)
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if math.Abs(total-tt.wantTotal) > 0.01 {
                t.Errorf("got total %.2f, want %.2f", total, tt.wantTotal)
            }
        })
    }
}
```

### Testing with fx

fx provides utilities for testing:

```go
func TestApp(t *testing.T) {
    var svc *UserService

    app := fxtest.New(t,
        fx.Provide(
            NewTestConfig,      // test-specific config
            NewInMemoryDB,      // fake DB
            NewUserRepository,
            NewUserService,
        ),
        fx.Populate(&svc), // extract the service for testing
    )
    app.RequireStart()
    defer app.RequireStop()

    // Now test with the fully wired service
    user, err := svc.GetUser(context.Background(), 1)
    // ...
}
```

### Integration Testing Pattern

```go
// testutil/container.go
package testutil

import (
    "database/sql"
    "testing"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func SetupTestDB(t *testing.T) *sql.DB {
    t.Helper()

    ctx := context.Background()
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    if err != nil {
        t.Fatalf("failed to start container: %v", err)
    }

    t.Cleanup(func() {
        pgContainer.Terminate(ctx)
    })

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatal(err)
    }

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        t.Fatal(err)
    }

    t.Cleanup(func() { db.Close() })

    // Run migrations
    runMigrations(t, db)

    return db
}

// Usage in tests:
func TestUserRepository_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    db := testutil.SetupTestDB(t)
    repo := NewUserRepository(db)

    // Test with a real database
    user := &User{Name: "Alice", Email: "alice@test.com"}
    err := repo.Save(context.Background(), user)
    if err != nil {
        t.Fatal(err)
    }

    got, err := repo.FindByID(context.Background(), user.ID)
    if err != nil {
        t.Fatal(err)
    }
    if got.Name != "Alice" {
        t.Errorf("got name %q, want %q", got.Name, "Alice")
    }
}
```

---

## 10. Service Locator Anti-Pattern

### What Is a Service Locator?

A Service Locator is a registry that components query at runtime to find their dependencies.
It is considered an **anti-pattern** in most contexts.

```go
// BAD: Service Locator pattern
package locator

var services = make(map[string]any)

func Register(name string, svc any) {
    services[name] = svc
}

func Get(name string) any {
    return services[name]
}

func GetTyped[T any](name string) T {
    return services[name].(T)
}
```

```go
// BAD: Using the service locator
type OrderService struct{}

func (s *OrderService) PlaceOrder(userID int) error {
    // Dependencies are hidden -- you can't tell from the constructor
    // what this service needs
    repo := locator.GetTyped[*OrderRepo]("orderRepo")
    emailer := locator.GetTyped[EmailSender]("emailer")
    logger := locator.GetTyped[Logger]("logger")

    // ... use them ...
    return nil
}
```

### Why It's an Anti-Pattern

| Problem | Explanation |
|---------|-------------|
| Hidden dependencies | Constructor does not reveal what the type needs |
| Runtime errors | Missing registrations fail at runtime, not compile time |
| Global state | The locator is effectively a global variable |
| Testing difficulty | Must set up the global locator before each test |
| Order dependence | Registration order matters and is fragile |
| No compile-time safety | Typos in service names cause runtime panics |

### Service Locator vs Dependency Injection

```go
// SERVICE LOCATOR (bad):
type Handler struct{}

func NewHandler() *Handler {
    return &Handler{}
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Where do these come from? Who knows!
    svc := container.Get[*UserService]()
    logger := container.Get[*slog.Logger]()
    // ...
}

// DEPENDENCY INJECTION (good):
type Handler struct {
    svc    *UserService
    logger *slog.Logger
}

func NewHandler(svc *UserService, logger *slog.Logger) *Handler {
    return &Handler{svc: svc, logger: logger}
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Dependencies are right here, explicit and testable
    h.logger.Info("handling request")
    user, err := h.svc.GetUser(r.Context(), 42)
    // ...
}
```

### When a Service Locator Might Be Acceptable

There are rare cases where a locator-like pattern is used:

1. **Plugin systems**: Dynamically loaded plugins cannot have compile-time DI.
2. **Middleware registries**: HTTP middleware chains sometimes use a registry.
3. **Legacy migration**: As a stepping stone from global variables to proper DI.

Even in these cases, limit the locator to the composition root and inject dependencies
into business logic normally.

### Context-Based Service Locator (Also an Anti-Pattern)

```go
// BAD: Stuffing dependencies into context.Context
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.WithValue(r.Context(), "db", db)
        ctx = context.WithValue(ctx, "logger", logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    db := r.Context().Value("db").(*sql.DB)       // type-unsafe
    logger := r.Context().Value("logger").(Logger) // may panic
    // ...
}
```

`context.Context` should carry **request-scoped** data (request ID, auth token, deadline),
not application-wide dependencies. Use constructor injection for services.

---

## 11. Organizing Large Applications with DI

### The Composition Root

The **composition root** is the single place where all dependencies are wired together.
In Go, this is typically `main()` or a function called from `main()`.

```
myapp/
  cmd/
    api/main.go          <-- composition root for API
    worker/main.go       <-- composition root for worker
    migrate/main.go      <-- composition root for migrations
  internal/
    domain/              <-- pure business logic, no infrastructure
    service/             <-- application services
    repository/          <-- data access interfaces + implementations
    handler/             <-- HTTP handlers
    config/              <-- configuration loading
```

### Layer Architecture with DI

```
┌──────────────────────────────────┐
│         main() / cmd/            │  Composition Root
│    Wires everything together     │  (knows all packages)
├──────────────────────────────────┤
│         handler/                 │  HTTP/gRPC Layer
│    Depends on: service ifaces    │  (knows domain + service interfaces)
├──────────────────────────────────┤
│         service/                 │  Application Layer
│    Depends on: domain + repo     │  (knows domain + repository interfaces)
│    interfaces                    │
├──────────────────────────────────┤
│         domain/                  │  Domain Layer
│    Depends on: nothing           │  (pure business logic)
├──────────────────────────────────┤
│         repository/              │  Infrastructure Layer
│    Depends on: domain types      │  (implements repository interfaces)
│    (postgres/, redis/, etc.)     │
└──────────────────────────────────┘
```

### Package Structure for DI

```go
// internal/domain/user.go -- pure types, no dependencies
package domain

type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

type UserFilter struct {
    Name  string
    Email string
    Limit int
}
```

```go
// internal/service/user.go -- business logic with interface dependencies
package service

import "myapp/internal/domain"

type UserRepository interface {
    FindByID(ctx context.Context, id int) (*domain.User, error)
    FindAll(ctx context.Context, filter domain.UserFilter) ([]*domain.User, error)
    Save(ctx context.Context, u *domain.User) error
    Delete(ctx context.Context, id int) error
}

type UserNotifier interface {
    NotifyWelcome(ctx context.Context, user *domain.User) error
}

type UserService struct {
    repo     UserRepository
    notifier UserNotifier
    logger   *slog.Logger
}

func NewUserService(repo UserRepository, notifier UserNotifier, logger *slog.Logger) *UserService {
    return &UserService{repo: repo, notifier: notifier, logger: logger}
}

func (s *UserService) Register(ctx context.Context, name, email string) (*domain.User, error) {
    user := &domain.User{
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
    }

    if err := s.repo.Save(ctx, user); err != nil {
        return nil, fmt.Errorf("save user: %w", err)
    }

    if err := s.notifier.NotifyWelcome(ctx, user); err != nil {
        s.logger.Error("failed to send welcome notification",
            "userID", user.ID, "error", err)
        // Don't return error -- notification failure shouldn't block registration
    }

    return user, nil
}
```

```go
// internal/repository/postgres/user.go -- infrastructure implementation
package postgres

import (
    "context"
    "database/sql"

    "myapp/internal/domain"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) FindByID(ctx context.Context, id int) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id).
        Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
    if err == sql.ErrNoRows {
        return nil, domain.ErrNotFound
    }
    return &u, err
}

func (r *UserRepository) Save(ctx context.Context, u *domain.User) error {
    return r.db.QueryRowContext(ctx,
        `INSERT INTO users (name, email, created_at)
         VALUES ($1, $2, $3) RETURNING id`,
        u.Name, u.Email, u.CreatedAt).Scan(&u.ID)
}

// ... other methods ...
```

```go
// internal/handler/user.go -- HTTP handler
package handler

import (
    "encoding/json"
    "net/http"

    "myapp/internal/domain"
)

// The handler defines only what it needs from the service layer
type UserRegistrar interface {
    Register(ctx context.Context, name, email string) (*domain.User, error)
}

type UserGetter interface {
    GetUser(ctx context.Context, id int) (*domain.User, error)
}

type UserHandler struct {
    registrar UserRegistrar
    getter    UserGetter
}

func NewUserHandler(registrar UserRegistrar, getter UserGetter) *UserHandler {
    return &UserHandler{registrar: registrar, getter: getter}
}

func (h *UserHandler) HandleRegister(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    user, err := h.registrar.Register(r.Context(), req.Name, req.Email)
    if err != nil {
        http.Error(w, "registration failed", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

### Avoiding Circular Dependencies

DI naturally prevents circular dependencies because each package only depends on
interfaces, not concrete implementations. The wiring happens at the composition root.

```
BAD:                          GOOD:
service -> repository         service -> repository (interface, defined in service)
repository -> service         repository/postgres -> domain (types only)
(circular!)                   main -> service, repository/postgres (wires them)
```

### Configuration Injection

```go
// internal/config/config.go
package config

import (
    "os"
    "strconv"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
    Email    EmailConfig
}

type ServerConfig struct {
    Port         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
}

type DatabaseConfig struct {
    URL             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

type RedisConfig struct {
    Addr     string
    Password string
    DB       int
}

type EmailConfig struct {
    SMTPHost string
    SMTPPort int
    From     string
    Password string
}

func Load() (*Config, error) {
    return &Config{
        Server: ServerConfig{
            Port:         getEnv("SERVER_PORT", ":8080"),
            ReadTimeout:  getDuration("SERVER_READ_TIMEOUT", 10*time.Second),
            WriteTimeout: getDuration("SERVER_WRITE_TIMEOUT", 10*time.Second),
        },
        Database: DatabaseConfig{
            URL:             mustGetEnv("DATABASE_URL"),
            MaxOpenConns:    getInt("DB_MAX_OPEN_CONNS", 25),
            MaxIdleConns:    getInt("DB_MAX_IDLE_CONNS", 5),
            ConnMaxLifetime: getDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
        },
        // ... other config ...
    }, nil
}

func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}

func mustGetEnv(key string) string {
    v := os.Getenv(key)
    if v == "" {
        panic(fmt.Sprintf("required environment variable %s is not set", key))
    }
    return v
}

func getInt(key string, fallback int) int {
    v := os.Getenv(key)
    if v == "" {
        return fallback
    }
    i, err := strconv.Atoi(v)
    if err != nil {
        return fallback
    }
    return i
}

func getDuration(key string, fallback time.Duration) time.Duration {
    v := os.Getenv(key)
    if v == "" {
        return fallback
    }
    d, err := time.ParseDuration(v)
    if err != nil {
        return fallback
    }
    return d
}
```

Inject specific config sections into constructors that need them:

```go
func NewDB(cfg config.DatabaseConfig) (*sql.DB, error) {
    db, err := sql.Open("postgres", cfg.URL)
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    return db, nil
}

func NewHTTPServer(cfg config.ServerConfig, handler http.Handler) *http.Server {
    return &http.Server{
        Addr:         cfg.Port,
        Handler:      handler,
        ReadTimeout:  cfg.ReadTimeout,
        WriteTimeout: cfg.WriteTimeout,
    }
}
```

> **Tip**: Inject narrow config types, not the entire Config struct. `NewDB` should
> receive `DatabaseConfig`, not `*Config`. This makes dependencies explicit and
> testing easier.

---

## 12. Real-World Example: Building a Web App with Proper DI

Let's build a complete bookstore API with proper dependency injection.

### Project Structure

```
bookstore/
  cmd/
    api/
      main.go
  internal/
    config/
      config.go
    domain/
      book.go
      errors.go
    service/
      book_service.go
      book_service_test.go
    repository/
      interfaces.go
      postgres/
        book_repo.go
      inmemory/
        book_repo.go    (for testing)
    handler/
      book_handler.go
      book_handler_test.go
      router.go
    middleware/
      logging.go
      recovery.go
  go.mod
  go.sum
```

### Domain Layer

```go
// internal/domain/book.go
package domain

import "time"

type Book struct {
    ID        string    `json:"id"`
    Title     string    `json:"title"`
    Author    string    `json:"author"`
    ISBN      string    `json:"isbn"`
    Price     float64   `json:"price"`
    Stock     int       `json:"stock"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type BookFilter struct {
    Author string
    MinPrice float64
    MaxPrice float64
    Offset   int
    Limit    int
}

type CreateBookInput struct {
    Title  string  `json:"title"`
    Author string  `json:"author"`
    ISBN   string  `json:"isbn"`
    Price  float64 `json:"price"`
    Stock  int     `json:"stock"`
}

func (i CreateBookInput) Validate() error {
    if i.Title == "" {
        return NewValidationError("title is required")
    }
    if i.Author == "" {
        return NewValidationError("author is required")
    }
    if i.ISBN == "" {
        return NewValidationError("isbn is required")
    }
    if i.Price <= 0 {
        return NewValidationError("price must be positive")
    }
    if i.Stock < 0 {
        return NewValidationError("stock cannot be negative")
    }
    return nil
}
```

```go
// internal/domain/errors.go
package domain

import "errors"

var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrOutOfStock    = errors.New("out of stock")
)

type ValidationError struct {
    Message string
}

func NewValidationError(msg string) *ValidationError {
    return &ValidationError{Message: msg}
}

func (e *ValidationError) Error() string {
    return e.Message
}

func IsValidationError(err error) bool {
    var ve *ValidationError
    return errors.As(err, &ve)
}
```

### Repository Interfaces and Implementations

```go
// internal/repository/interfaces.go
package repository

import (
    "context"

    "bookstore/internal/domain"
)

type BookRepository interface {
    Create(ctx context.Context, book *domain.Book) error
    FindByID(ctx context.Context, id string) (*domain.Book, error)
    FindByISBN(ctx context.Context, isbn string) (*domain.Book, error)
    FindAll(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error)
    Update(ctx context.Context, book *domain.Book) error
    Delete(ctx context.Context, id string) error
    UpdateStock(ctx context.Context, id string, delta int) error
}
```

```go
// internal/repository/postgres/book_repo.go
package postgres

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    "bookstore/internal/domain"

    "github.com/google/uuid"
)

type BookRepository struct {
    db *sql.DB
}

func NewBookRepository(db *sql.DB) *BookRepository {
    return &BookRepository{db: db}
}

func (r *BookRepository) Create(ctx context.Context, book *domain.Book) error {
    book.ID = uuid.New().String()
    book.CreatedAt = time.Now()
    book.UpdatedAt = time.Now()

    _, err := r.db.ExecContext(ctx,
        `INSERT INTO books (id, title, author, isbn, price, stock, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
        book.ID, book.Title, book.Author, book.ISBN,
        book.Price, book.Stock, book.CreatedAt, book.UpdatedAt)
    if err != nil {
        // Check for unique constraint violation on ISBN
        if isUniqueViolation(err) {
            return domain.ErrAlreadyExists
        }
        return fmt.Errorf("insert book: %w", err)
    }
    return nil
}

func (r *BookRepository) FindByID(ctx context.Context, id string) (*domain.Book, error) {
    var b domain.Book
    err := r.db.QueryRowContext(ctx,
        `SELECT id, title, author, isbn, price, stock, created_at, updated_at
         FROM books WHERE id = $1`, id).
        Scan(&b.ID, &b.Title, &b.Author, &b.ISBN,
            &b.Price, &b.Stock, &b.CreatedAt, &b.UpdatedAt)
    if err == sql.ErrNoRows {
        return nil, domain.ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("find book by id: %w", err)
    }
    return &b, nil
}

func (r *BookRepository) FindByISBN(ctx context.Context, isbn string) (*domain.Book, error) {
    var b domain.Book
    err := r.db.QueryRowContext(ctx,
        `SELECT id, title, author, isbn, price, stock, created_at, updated_at
         FROM books WHERE isbn = $1`, isbn).
        Scan(&b.ID, &b.Title, &b.Author, &b.ISBN,
            &b.Price, &b.Stock, &b.CreatedAt, &b.UpdatedAt)
    if err == sql.ErrNoRows {
        return nil, domain.ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("find book by isbn: %w", err)
    }
    return &b, nil
}

func (r *BookRepository) FindAll(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error) {
    query := `SELECT id, title, author, isbn, price, stock, created_at, updated_at FROM books WHERE 1=1`
    args := []any{}
    argIdx := 1

    if filter.Author != "" {
        query += fmt.Sprintf(" AND author ILIKE $%d", argIdx)
        args = append(args, "%"+filter.Author+"%")
        argIdx++
    }
    if filter.MinPrice > 0 {
        query += fmt.Sprintf(" AND price >= $%d", argIdx)
        args = append(args, filter.MinPrice)
        argIdx++
    }
    if filter.MaxPrice > 0 {
        query += fmt.Sprintf(" AND price <= $%d", argIdx)
        args = append(args, filter.MaxPrice)
        argIdx++
    }

    query += " ORDER BY created_at DESC"

    if filter.Limit > 0 {
        query += fmt.Sprintf(" LIMIT $%d", argIdx)
        args = append(args, filter.Limit)
        argIdx++
    }
    if filter.Offset > 0 {
        query += fmt.Sprintf(" OFFSET $%d", argIdx)
        args = append(args, filter.Offset)
    }

    rows, err := r.db.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, fmt.Errorf("list books: %w", err)
    }
    defer rows.Close()

    var books []*domain.Book
    for rows.Next() {
        var b domain.Book
        if err := rows.Scan(&b.ID, &b.Title, &b.Author, &b.ISBN,
            &b.Price, &b.Stock, &b.CreatedAt, &b.UpdatedAt); err != nil {
            return nil, fmt.Errorf("scan book: %w", err)
        }
        books = append(books, &b)
    }
    return books, rows.Err()
}

func (r *BookRepository) Update(ctx context.Context, book *domain.Book) error {
    book.UpdatedAt = time.Now()
    result, err := r.db.ExecContext(ctx,
        `UPDATE books SET title=$1, author=$2, isbn=$3, price=$4, stock=$5, updated_at=$6
         WHERE id=$7`,
        book.Title, book.Author, book.ISBN, book.Price, book.Stock, book.UpdatedAt, book.ID)
    if err != nil {
        return fmt.Errorf("update book: %w", err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return domain.ErrNotFound
    }
    return nil
}

func (r *BookRepository) Delete(ctx context.Context, id string) error {
    result, err := r.db.ExecContext(ctx, "DELETE FROM books WHERE id=$1", id)
    if err != nil {
        return fmt.Errorf("delete book: %w", err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return domain.ErrNotFound
    }
    return nil
}

func (r *BookRepository) UpdateStock(ctx context.Context, id string, delta int) error {
    result, err := r.db.ExecContext(ctx,
        `UPDATE books SET stock = stock + $1, updated_at = $2
         WHERE id = $3 AND stock + $1 >= 0`,
        delta, time.Now(), id)
    if err != nil {
        return fmt.Errorf("update stock: %w", err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        // Could be not found or insufficient stock
        _, findErr := r.FindByID(ctx, id)
        if findErr != nil {
            return domain.ErrNotFound
        }
        return domain.ErrOutOfStock
    }
    return nil
}
```

```go
// internal/repository/inmemory/book_repo.go -- for testing
package inmemory

import (
    "context"
    "sync"
    "time"

    "bookstore/internal/domain"

    "github.com/google/uuid"
)

type BookRepository struct {
    mu    sync.RWMutex
    books map[string]*domain.Book
}

func NewBookRepository() *BookRepository {
    return &BookRepository{books: make(map[string]*domain.Book)}
}

func (r *BookRepository) Create(ctx context.Context, book *domain.Book) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    // Check ISBN uniqueness
    for _, b := range r.books {
        if b.ISBN == book.ISBN {
            return domain.ErrAlreadyExists
        }
    }

    book.ID = uuid.New().String()
    book.CreatedAt = time.Now()
    book.UpdatedAt = time.Now()

    // Store a copy to prevent mutation
    stored := *book
    r.books[book.ID] = &stored
    return nil
}

func (r *BookRepository) FindByID(ctx context.Context, id string) (*domain.Book, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    b, ok := r.books[id]
    if !ok {
        return nil, domain.ErrNotFound
    }
    copy := *b
    return &copy, nil
}

func (r *BookRepository) FindByISBN(ctx context.Context, isbn string) (*domain.Book, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    for _, b := range r.books {
        if b.ISBN == isbn {
            copy := *b
            return &copy, nil
        }
    }
    return nil, domain.ErrNotFound
}

func (r *BookRepository) FindAll(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    var result []*domain.Book
    for _, b := range r.books {
        if filter.Author != "" && b.Author != filter.Author {
            continue
        }
        if filter.MinPrice > 0 && b.Price < filter.MinPrice {
            continue
        }
        if filter.MaxPrice > 0 && b.Price > filter.MaxPrice {
            continue
        }
        copy := *b
        result = append(result, &copy)
    }

    // Apply offset and limit
    if filter.Offset > 0 && filter.Offset < len(result) {
        result = result[filter.Offset:]
    }
    if filter.Limit > 0 && filter.Limit < len(result) {
        result = result[:filter.Limit]
    }

    return result, nil
}

func (r *BookRepository) Update(ctx context.Context, book *domain.Book) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, ok := r.books[book.ID]; !ok {
        return domain.ErrNotFound
    }

    book.UpdatedAt = time.Now()
    stored := *book
    r.books[book.ID] = &stored
    return nil
}

func (r *BookRepository) Delete(ctx context.Context, id string) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, ok := r.books[id]; !ok {
        return domain.ErrNotFound
    }
    delete(r.books, id)
    return nil
}

func (r *BookRepository) UpdateStock(ctx context.Context, id string, delta int) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    b, ok := r.books[id]
    if !ok {
        return domain.ErrNotFound
    }
    if b.Stock+delta < 0 {
        return domain.ErrOutOfStock
    }
    b.Stock += delta
    b.UpdatedAt = time.Now()
    return nil
}
```

### Service Layer

```go
// internal/service/book_service.go
package service

import (
    "context"
    "fmt"
    "log/slog"

    "bookstore/internal/domain"
    "bookstore/internal/repository"
)

type BookService struct {
    repo   repository.BookRepository
    logger *slog.Logger
}

func NewBookService(repo repository.BookRepository, logger *slog.Logger) *BookService {
    return &BookService{repo: repo, logger: logger}
}

func (s *BookService) CreateBook(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error) {
    if err := input.Validate(); err != nil {
        return nil, err
    }

    // Check if ISBN already exists
    existing, err := s.repo.FindByISBN(ctx, input.ISBN)
    if err == nil && existing != nil {
        return nil, fmt.Errorf("book with ISBN %s: %w", input.ISBN, domain.ErrAlreadyExists)
    }

    book := &domain.Book{
        Title:  input.Title,
        Author: input.Author,
        ISBN:   input.ISBN,
        Price:  input.Price,
        Stock:  input.Stock,
    }

    if err := s.repo.Create(ctx, book); err != nil {
        return nil, fmt.Errorf("create book: %w", err)
    }

    s.logger.Info("book created",
        "id", book.ID,
        "title", book.Title,
        "isbn", book.ISBN)

    return book, nil
}

func (s *BookService) GetBook(ctx context.Context, id string) (*domain.Book, error) {
    book, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get book %s: %w", id, err)
    }
    return book, nil
}

func (s *BookService) ListBooks(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error) {
    if filter.Limit <= 0 {
        filter.Limit = 20
    }
    if filter.Limit > 100 {
        filter.Limit = 100
    }
    return s.repo.FindAll(ctx, filter)
}

func (s *BookService) UpdateBook(ctx context.Context, id string, input domain.CreateBookInput) (*domain.Book, error) {
    if err := input.Validate(); err != nil {
        return nil, err
    }

    book, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("find book for update: %w", err)
    }

    book.Title = input.Title
    book.Author = input.Author
    book.ISBN = input.ISBN
    book.Price = input.Price
    book.Stock = input.Stock

    if err := s.repo.Update(ctx, book); err != nil {
        return nil, fmt.Errorf("update book: %w", err)
    }

    s.logger.Info("book updated", "id", book.ID, "title", book.Title)
    return book, nil
}

func (s *BookService) DeleteBook(ctx context.Context, id string) error {
    if err := s.repo.Delete(ctx, id); err != nil {
        return fmt.Errorf("delete book %s: %w", id, err)
    }
    s.logger.Info("book deleted", "id", id)
    return nil
}

func (s *BookService) PurchaseBook(ctx context.Context, id string, quantity int) error {
    if quantity <= 0 {
        return domain.NewValidationError("quantity must be positive")
    }

    if err := s.repo.UpdateStock(ctx, id, -quantity); err != nil {
        return fmt.Errorf("purchase book %s: %w", id, err)
    }

    s.logger.Info("book purchased", "id", id, "quantity", quantity)
    return nil
}
```

### Service Tests

```go
// internal/service/book_service_test.go
package service_test

import (
    "context"
    "errors"
    "log/slog"
    "os"
    "testing"

    "bookstore/internal/domain"
    "bookstore/internal/repository/inmemory"
    "bookstore/internal/service"
)

func newTestService() *service.BookService {
    repo := inmemory.NewBookRepository()
    logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelError}))
    return service.NewBookService(repo, logger)
}

func TestBookService_CreateBook(t *testing.T) {
    tests := []struct {
        name    string
        input   domain.CreateBookInput
        wantErr bool
        errType error
    }{
        {
            name: "valid book",
            input: domain.CreateBookInput{
                Title:  "The Go Programming Language",
                Author: "Donovan & Kernighan",
                ISBN:   "978-0134190440",
                Price:  44.99,
                Stock:  100,
            },
            wantErr: false,
        },
        {
            name: "missing title",
            input: domain.CreateBookInput{
                Author: "Unknown",
                ISBN:   "978-0000000000",
                Price:  10.00,
                Stock:  1,
            },
            wantErr: true,
        },
        {
            name: "negative price",
            input: domain.CreateBookInput{
                Title:  "Bad Book",
                Author: "Nobody",
                ISBN:   "978-1111111111",
                Price:  -5.00,
                Stock:  1,
            },
            wantErr: true,
        },
        {
            name: "zero stock is valid",
            input: domain.CreateBookInput{
                Title:  "Preorder Book",
                Author: "Upcoming Author",
                ISBN:   "978-2222222222",
                Price:  29.99,
                Stock:  0,
            },
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := newTestService()
            ctx := context.Background()

            book, err := svc.CreateBook(ctx, tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if book.ID == "" {
                t.Error("expected book to have an ID")
            }
            if book.Title != tt.input.Title {
                t.Errorf("title = %q, want %q", book.Title, tt.input.Title)
            }
            if book.CreatedAt.IsZero() {
                t.Error("expected CreatedAt to be set")
            }
        })
    }
}

func TestBookService_CreateBook_DuplicateISBN(t *testing.T) {
    svc := newTestService()
    ctx := context.Background()

    input := domain.CreateBookInput{
        Title:  "Original",
        Author: "Author",
        ISBN:   "978-0134190440",
        Price:  44.99,
        Stock:  10,
    }

    _, err := svc.CreateBook(ctx, input)
    if err != nil {
        t.Fatalf("first create failed: %v", err)
    }

    _, err = svc.CreateBook(ctx, input)
    if err == nil {
        t.Fatal("expected error for duplicate ISBN")
    }
    if !errors.Is(err, domain.ErrAlreadyExists) {
        t.Errorf("expected ErrAlreadyExists, got: %v", err)
    }
}

func TestBookService_PurchaseBook(t *testing.T) {
    svc := newTestService()
    ctx := context.Background()

    book, err := svc.CreateBook(ctx, domain.CreateBookInput{
        Title:  "Test Book",
        Author: "Test Author",
        ISBN:   "978-0000000001",
        Price:  19.99,
        Stock:  5,
    })
    if err != nil {
        t.Fatalf("create: %v", err)
    }

    tests := []struct {
        name     string
        quantity int
        wantErr  bool
        errIs    error
    }{
        {name: "buy 2", quantity: 2, wantErr: false},
        {name: "buy 2 more", quantity: 2, wantErr: false},
        {name: "buy 2 but only 1 left", quantity: 2, wantErr: true, errIs: domain.ErrOutOfStock},
        {name: "buy last one", quantity: 1, wantErr: false},
        {name: "buy from empty stock", quantity: 1, wantErr: true, errIs: domain.ErrOutOfStock},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := svc.PurchaseBook(ctx, book.ID, tt.quantity)
            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error")
                }
                if tt.errIs != nil && !errors.Is(err, tt.errIs) {
                    t.Errorf("expected %v, got %v", tt.errIs, err)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
        })
    }
}

func TestBookService_ListBooks_Pagination(t *testing.T) {
    svc := newTestService()
    ctx := context.Background()

    // Create 15 books
    for i := 0; i < 15; i++ {
        _, err := svc.CreateBook(ctx, domain.CreateBookInput{
            Title:  fmt.Sprintf("Book %d", i),
            Author: "Author",
            ISBN:   fmt.Sprintf("978-%010d", i),
            Price:  10.00 + float64(i),
            Stock:  1,
        })
        if err != nil {
            t.Fatalf("create book %d: %v", i, err)
        }
    }

    // Default limit
    books, err := svc.ListBooks(ctx, domain.BookFilter{})
    if err != nil {
        t.Fatalf("list: %v", err)
    }
    if len(books) != 15 {
        t.Errorf("got %d books, want 15", len(books))
    }

    // With limit
    books, err = svc.ListBooks(ctx, domain.BookFilter{Limit: 5})
    if err != nil {
        t.Fatalf("list with limit: %v", err)
    }
    if len(books) != 5 {
        t.Errorf("got %d books, want 5", len(books))
    }
}
```

### HTTP Handler Layer

```go
// internal/handler/book_handler.go
package handler

import (
    "encoding/json"
    "errors"
    "net/http"
    "strconv"

    "bookstore/internal/domain"
)

// Interfaces defined in the consumer package -- handler only depends on what it needs
type BookCreator interface {
    CreateBook(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error)
}

type BookGetter interface {
    GetBook(ctx context.Context, id string) (*domain.Book, error)
}

type BookLister interface {
    ListBooks(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error)
}

type BookUpdater interface {
    UpdateBook(ctx context.Context, id string, input domain.CreateBookInput) (*domain.Book, error)
}

type BookDeleter interface {
    DeleteBook(ctx context.Context, id string) error
}

type BookPurchaser interface {
    PurchaseBook(ctx context.Context, id string, quantity int) error
}

// Combine what this handler needs
type BookService interface {
    BookCreator
    BookGetter
    BookLister
    BookUpdater
    BookDeleter
    BookPurchaser
}

type BookHandler struct {
    svc BookService
}

func NewBookHandler(svc BookService) *BookHandler {
    return &BookHandler{svc: svc}
}

func (h *BookHandler) HandleCreate(w http.ResponseWriter, r *http.Request) {
    var input domain.CreateBookInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    book, err := h.svc.CreateBook(r.Context(), input)
    if err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusCreated, book)
}

func (h *BookHandler) HandleGet(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+

    book, err := h.svc.GetBook(r.Context(), id)
    if err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusOK, book)
}

func (h *BookHandler) HandleList(w http.ResponseWriter, r *http.Request) {
    filter := domain.BookFilter{
        Author: r.URL.Query().Get("author"),
    }
    if v := r.URL.Query().Get("limit"); v != "" {
        filter.Limit, _ = strconv.Atoi(v)
    }
    if v := r.URL.Query().Get("offset"); v != "" {
        filter.Offset, _ = strconv.Atoi(v)
    }
    if v := r.URL.Query().Get("min_price"); v != "" {
        filter.MinPrice, _ = strconv.ParseFloat(v, 64)
    }
    if v := r.URL.Query().Get("max_price"); v != "" {
        filter.MaxPrice, _ = strconv.ParseFloat(v, 64)
    }

    books, err := h.svc.ListBooks(r.Context(), filter)
    if err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusOK, map[string]any{
        "books": books,
        "count": len(books),
    })
}

func (h *BookHandler) HandleUpdate(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    var input domain.CreateBookInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    book, err := h.svc.UpdateBook(r.Context(), id, input)
    if err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusOK, book)
}

func (h *BookHandler) HandleDelete(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    if err := h.svc.DeleteBook(r.Context(), id); err != nil {
        handleServiceError(w, err)
        return
    }

    w.WriteHeader(http.StatusNoContent)
}

func (h *BookHandler) HandlePurchase(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    var req struct {
        Quantity int `json:"quantity"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    if err := h.svc.PurchaseBook(r.Context(), id, req.Quantity); err != nil {
        handleServiceError(w, err)
        return
    }

    writeJSON(w, http.StatusOK, map[string]string{"status": "purchased"})
}

// Helper functions

func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}

func handleServiceError(w http.ResponseWriter, err error) {
    switch {
    case errors.Is(err, domain.ErrNotFound):
        writeError(w, http.StatusNotFound, "resource not found")
    case errors.Is(err, domain.ErrAlreadyExists):
        writeError(w, http.StatusConflict, "resource already exists")
    case errors.Is(err, domain.ErrOutOfStock):
        writeError(w, http.StatusUnprocessableEntity, "out of stock")
    case domain.IsValidationError(err):
        writeError(w, http.StatusBadRequest, err.Error())
    default:
        writeError(w, http.StatusInternalServerError, "internal server error")
    }
}
```

```go
// internal/handler/router.go
package handler

import (
    "log/slog"
    "net/http"

    "bookstore/internal/middleware"
)

func NewRouter(bookHandler *BookHandler, logger *slog.Logger) http.Handler {
    mux := http.NewServeMux()

    // Book routes (Go 1.22+ method-pattern syntax)
    mux.HandleFunc("POST /api/books", bookHandler.HandleCreate)
    mux.HandleFunc("GET /api/books", bookHandler.HandleList)
    mux.HandleFunc("GET /api/books/{id}", bookHandler.HandleGet)
    mux.HandleFunc("PUT /api/books/{id}", bookHandler.HandleUpdate)
    mux.HandleFunc("DELETE /api/books/{id}", bookHandler.HandleDelete)
    mux.HandleFunc("POST /api/books/{id}/purchase", bookHandler.HandlePurchase)

    // Health check
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
    })

    // Apply middleware
    var handler http.Handler = mux
    handler = middleware.Recovery(handler, logger)
    handler = middleware.Logging(handler, logger)

    return handler
}
```

### Middleware

```go
// internal/middleware/logging.go
package middleware

import (
    "log/slog"
    "net/http"
    "time"
)

type responseWriter struct {
    http.ResponseWriter
    statusCode int
    bytes      int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bytes += n
    return n, err
}

func Logging(next http.Handler, logger *slog.Logger) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(rw, r)

        logger.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", rw.statusCode,
            "bytes", rw.bytes,
            "duration", time.Since(start),
        )
    })
}
```

```go
// internal/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"
    "runtime/debug"
)

func Recovery(next http.Handler, logger *slog.Logger) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.Error("panic recovered",
                    "error", rec,
                    "stack", string(debug.Stack()),
                )
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Configuration

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
}

type ServerConfig struct {
    Port         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
}

type DatabaseConfig struct {
    URL             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

func Load() (*Config, error) {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return nil, fmt.Errorf("DATABASE_URL is required")
    }

    return &Config{
        Server: ServerConfig{
            Port:         getEnv("PORT", ":8080"),
            ReadTimeout:  getDuration("READ_TIMEOUT", 10*time.Second),
            WriteTimeout: getDuration("WRITE_TIMEOUT", 10*time.Second),
        },
        Database: DatabaseConfig{
            URL:             dbURL,
            MaxOpenConns:    getInt("DB_MAX_OPEN_CONNS", 25),
            MaxIdleConns:    getInt("DB_MAX_IDLE_CONNS", 5),
            ConnMaxLifetime: getDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
        },
    }, nil
}

func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}

func getInt(key string, fallback int) int {
    if v := os.Getenv(key); v != "" {
        if i, err := strconv.Atoi(v); err == nil {
            return i
        }
    }
    return fallback
}

func getDuration(key string, fallback time.Duration) time.Duration {
    if v := os.Getenv(key); v != "" {
        if d, err := time.ParseDuration(v); err == nil {
            return d
        }
    }
    return fallback
}
```

### The Composition Root (main.go) -- Manual DI

```go
// cmd/api/main.go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    _ "github.com/lib/pq"

    "bookstore/internal/config"
    "bookstore/internal/handler"
    "bookstore/internal/repository/postgres"
    "bookstore/internal/service"
)

func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}

func run() error {
    // ---- Configuration ----
    cfg, err := config.Load()
    if err != nil {
        return fmt.Errorf("load config: %w", err)
    }

    // ---- Logger ----
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    // ---- Database ----
    db, err := sql.Open("postgres", cfg.Database.URL)
    if err != nil {
        return fmt.Errorf("open database: %w", err)
    }
    defer db.Close()

    db.SetMaxOpenConns(cfg.Database.MaxOpenConns)
    db.SetMaxIdleConns(cfg.Database.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.Database.ConnMaxLifetime)

    if err := db.Ping(); err != nil {
        return fmt.Errorf("ping database: %w", err)
    }
    logger.Info("connected to database")

    // ---- Repository Layer ----
    bookRepo := postgres.NewBookRepository(db)

    // ---- Service Layer ----
    bookService := service.NewBookService(bookRepo, logger)

    // ---- Handler Layer ----
    bookHandler := handler.NewBookHandler(bookService)
    router := handler.NewRouter(bookHandler, logger)

    // ---- HTTP Server ----
    srv := &http.Server{
        Addr:         cfg.Server.Port,
        Handler:      router,
        ReadTimeout:  cfg.Server.ReadTimeout,
        WriteTimeout: cfg.Server.WriteTimeout,
    }

    // ---- Graceful Shutdown ----
    errCh := make(chan error, 1)
    go func() {
        logger.Info("server starting", "addr", cfg.Server.Port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case sig := <-quit:
        logger.Info("shutdown signal received", "signal", sig)
    case err := <-errCh:
        return fmt.Errorf("server error: %w", err)
    }

    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        return fmt.Errorf("server shutdown: %w", err)
    }

    logger.Info("server stopped gracefully")
    return nil
}
```

### Alternative: The Same App with fx

```go
// cmd/api/main.go (fx version)
package main

import (
    "context"
    "database/sql"
    "log/slog"
    "net/http"
    "os"

    _ "github.com/lib/pq"
    "go.uber.org/fx"

    "bookstore/internal/config"
    "bookstore/internal/handler"
    "bookstore/internal/repository"
    "bookstore/internal/repository/postgres"
    "bookstore/internal/service"
)

func main() {
    fx.New(
        // Configuration
        fx.Provide(config.Load),

        // Logger
        fx.Provide(func() *slog.Logger {
            return slog.New(slog.NewJSONHandler(os.Stdout, nil))
        }),

        // Database
        fx.Provide(NewDB),

        // Repository (bind interface)
        fx.Provide(
            fx.Annotate(
                postgres.NewBookRepository,
                fx.As(new(repository.BookRepository)),
            ),
        ),

        // Service (bind interface)
        fx.Provide(
            fx.Annotate(
                service.NewBookService,
                fx.As(new(handler.BookService)),
            ),
        ),

        // Handler
        fx.Provide(
            handler.NewBookHandler,
            handler.NewRouter,
        ),

        // HTTP Server
        fx.Provide(NewHTTPServer),

        // Start the server
        fx.Invoke(RegisterLifecycle),
    ).Run()
}

func NewDB(cfg *config.Config, logger *slog.Logger) (*sql.DB, error) {
    db, err := sql.Open("postgres", cfg.Database.URL)
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(cfg.Database.MaxOpenConns)
    db.SetMaxIdleConns(cfg.Database.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.Database.ConnMaxLifetime)
    return db, nil
}

func NewHTTPServer(cfg *config.Config, router http.Handler) *http.Server {
    return &http.Server{
        Addr:         cfg.Server.Port,
        Handler:      router,
        ReadTimeout:  cfg.Server.ReadTimeout,
        WriteTimeout: cfg.Server.WriteTimeout,
    }
}

func RegisterLifecycle(lc fx.Lifecycle, srv *http.Server, db *sql.DB, logger *slog.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            if err := db.PingContext(ctx); err != nil {
                return err
            }
            logger.Info("database connected")

            go func() {
                logger.Info("server starting", "addr", srv.Addr)
                srv.ListenAndServe()
            }()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("shutting down")
            if err := srv.Shutdown(ctx); err != nil {
                return err
            }
            return db.Close()
        },
    })
}
```

### Alternative: The Same App with Wire

```go
//go:build wireinject

// cmd/api/wire.go
package main

import (
    "log/slog"
    "os"

    "github.com/google/wire"

    "bookstore/internal/config"
    "bookstore/internal/handler"
    "bookstore/internal/repository"
    "bookstore/internal/repository/postgres"
    "bookstore/internal/service"
)

var repoSet = wire.NewSet(
    postgres.NewBookRepository,
    wire.Bind(new(repository.BookRepository), new(*postgres.BookRepository)),
)

var serviceSet = wire.NewSet(
    service.NewBookService,
    wire.Bind(new(handler.BookService), new(*service.BookService)),
)

var handlerSet = wire.NewSet(
    handler.NewBookHandler,
    handler.NewRouter,
)

func provideLogger() *slog.Logger {
    return slog.New(slog.NewJSONHandler(os.Stdout, nil))
}

func InitializeApp() (*App, func(), error) {
    wire.Build(
        config.Load,
        provideLogger,
        provideDB,       // returns (*sql.DB, func(), error)
        repoSet,
        serviceSet,
        handlerSet,
        provideHTTPServer,
        wire.Struct(new(App), "*"),
    )
    return nil, nil, nil
}
```

```go
// cmd/api/main.go (Wire version)
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
)

type App struct {
    Server *http.Server
    Logger *slog.Logger
}

func main() {
    app, cleanup, err := InitializeApp()
    if err != nil {
        fmt.Fprintf(os.Stderr, "initialization error: %v\n", err)
        os.Exit(1)
    }
    defer cleanup()

    errCh := make(chan error, 1)
    go func() {
        app.Logger.Info("server starting", "addr", app.Server.Addr)
        if err := app.Server.ListenAndServe(); err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case <-quit:
    case err := <-errCh:
        app.Logger.Error("server failed", "error", err)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    app.Server.Shutdown(ctx)
}
```

### Handler-Level Testing

```go
// internal/handler/book_handler_test.go
package handler_test

import (
    "bytes"
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "bookstore/internal/domain"
    "bookstore/internal/handler"
)

// Stub service for handler tests
type stubBookService struct {
    createFunc   func(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error)
    getFunc      func(ctx context.Context, id string) (*domain.Book, error)
    listFunc     func(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error)
    updateFunc   func(ctx context.Context, id string, input domain.CreateBookInput) (*domain.Book, error)
    deleteFunc   func(ctx context.Context, id string) error
    purchaseFunc func(ctx context.Context, id string, quantity int) error
}

func (s *stubBookService) CreateBook(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error) {
    return s.createFunc(ctx, input)
}
func (s *stubBookService) GetBook(ctx context.Context, id string) (*domain.Book, error) {
    return s.getFunc(ctx, id)
}
func (s *stubBookService) ListBooks(ctx context.Context, filter domain.BookFilter) ([]*domain.Book, error) {
    return s.listFunc(ctx, filter)
}
func (s *stubBookService) UpdateBook(ctx context.Context, id string, input domain.CreateBookInput) (*domain.Book, error) {
    return s.updateFunc(ctx, id, input)
}
func (s *stubBookService) DeleteBook(ctx context.Context, id string) error {
    return s.deleteFunc(ctx, id)
}
func (s *stubBookService) PurchaseBook(ctx context.Context, id string, quantity int) error {
    return s.purchaseFunc(ctx, id, quantity)
}

func TestBookHandler_HandleCreate(t *testing.T) {
    tests := []struct {
        name       string
        body       string
        setupSvc   func(*stubBookService)
        wantStatus int
    }{
        {
            name: "success",
            body: `{"title":"Go Book","author":"Author","isbn":"978-0000000001","price":29.99,"stock":10}`,
            setupSvc: func(s *stubBookService) {
                s.createFunc = func(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error) {
                    return &domain.Book{
                        ID:     "abc-123",
                        Title:  input.Title,
                        Author: input.Author,
                        ISBN:   input.ISBN,
                        Price:  input.Price,
                        Stock:  input.Stock,
                    }, nil
                }
            },
            wantStatus: http.StatusCreated,
        },
        {
            name: "validation error",
            body: `{"title":"","author":"Author","isbn":"978-0000000001","price":29.99,"stock":10}`,
            setupSvc: func(s *stubBookService) {
                s.createFunc = func(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error) {
                    return nil, domain.NewValidationError("title is required")
                }
            },
            wantStatus: http.StatusBadRequest,
        },
        {
            name: "duplicate isbn",
            body: `{"title":"Go Book","author":"Author","isbn":"978-0000000001","price":29.99,"stock":10}`,
            setupSvc: func(s *stubBookService) {
                s.createFunc = func(ctx context.Context, input domain.CreateBookInput) (*domain.Book, error) {
                    return nil, domain.ErrAlreadyExists
                }
            },
            wantStatus: http.StatusConflict,
        },
        {
            name:       "invalid json",
            body:       `{invalid}`,
            setupSvc:   func(s *stubBookService) {},
            wantStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            svc := &stubBookService{}
            tt.setupSvc(svc)

            h := handler.NewBookHandler(svc)

            req := httptest.NewRequest(http.MethodPost, "/api/books", bytes.NewBufferString(tt.body))
            req.Header.Set("Content-Type", "application/json")
            rec := httptest.NewRecorder()

            h.HandleCreate(rec, req)

            if rec.Code != tt.wantStatus {
                t.Errorf("status = %d, want %d. Body: %s", rec.Code, tt.wantStatus, rec.Body.String())
            }

            if tt.wantStatus == http.StatusCreated {
                var book domain.Book
                if err := json.NewDecoder(rec.Body).Decode(&book); err != nil {
                    t.Fatalf("failed to decode response: %v", err)
                }
                if book.ID == "" {
                    t.Error("expected book ID in response")
                }
            }
        })
    }
}

func TestBookHandler_HandleGet_NotFound(t *testing.T) {
    svc := &stubBookService{
        getFunc: func(ctx context.Context, id string) (*domain.Book, error) {
            return nil, domain.ErrNotFound
        },
    }

    h := handler.NewBookHandler(svc)

    req := httptest.NewRequest(http.MethodGet, "/api/books/nonexistent", nil)
    req.SetPathValue("id", "nonexistent") // Go 1.22+
    rec := httptest.NewRecorder()

    h.HandleGet(rec, req)

    if rec.Code != http.StatusNotFound {
        t.Errorf("status = %d, want %d", rec.Code, http.StatusNotFound)
    }
}
```

---

## 13. Summary and Decision Guide

### Key Takeaways

1. **DI is a technique, not a framework**. The core idea -- passing dependencies through
   constructors -- requires no libraries.

2. **Go's implicit interfaces** make DI natural. Define small interfaces where they are
   consumed, and let implementations satisfy them implicitly.

3. **Start with manual DI**. It's explicit, debuggable, and sufficient for most projects.

4. **Use Wire** when wiring code becomes tedious but you want compile-time safety and
   zero runtime overhead.

5. **Use fx** when you need lifecycle management (start/stop hooks) and your team is
   comfortable with runtime DI.

6. **Never use a DI framework in a library**. Libraries should accept interfaces via
   constructors, letting callers decide how to provide them.

7. **Test with the simplest double that works**: stubs for most tests, fakes for
   integration-like tests, mocks (mockgen/moq) only when you need to verify interactions.

8. **Avoid the Service Locator pattern**. It hides dependencies and defeats the purpose
   of DI.

### Quick Decision Flowchart

```
Is your main() under 50 lines of wiring?
  YES -> Manual DI. You're done.
  NO  -> Do you need lifecycle management (start/stop)?
           YES -> Use fx (or manual with your own lifecycle)
           NO  -> Do you want compile-time safety?
                    YES -> Use Wire
                    NO  -> Use fx or dig

Is this a library?
  YES -> Manual DI only. Accept interfaces, return structs.
```

### DI Principles Checklist

- [ ] Dependencies are passed in, never created internally
- [ ] Interfaces are defined in the consumer package
- [ ] Interfaces are small (1-3 methods when possible)
- [ ] `context.Context` is passed to methods, not stored in structs
- [ ] Configuration is injected as narrow types, not a global config
- [ ] The composition root is the only place that knows about all packages
- [ ] No circular dependencies between packages
- [ ] Test doubles (mocks/fakes/stubs) are trivial to create

### Further Reading

- [Google Wire Documentation](https://github.com/google/wire/blob/main/docs/guide.md)
- [Uber fx Documentation](https://uber-go.github.io/fx/)
- [Uber dig Documentation](https://pkg.go.dev/go.uber.org/dig)
- [Effective Go - Interfaces](https://go.dev/doc/effective_go#interfaces)
- [Go Wiki - Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [SOLID Principles in Go](https://dave.cheney.net/2016/08/20/solid-go-design)
- [Rob Pike: "A little copying is better than a little dependency"](https://go-proverbs.github.io/)
- [Ben Johnson: Standard Package Layout](https://www.gobeyond.dev/standard-package-layout/)
