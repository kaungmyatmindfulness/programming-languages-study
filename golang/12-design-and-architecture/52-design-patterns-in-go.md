# Chapter 52: Design Patterns in Go

## Table of Contents

1. [Why Patterns Differ in Go](#why-patterns-differ-in-go)
2. [Creational Patterns](#creational-patterns)
   - [Functional Options Pattern](#functional-options-pattern)
   - [Builder Pattern](#builder-pattern)
   - [Factory Functions](#factory-functions)
   - [Singleton (sync.Once)](#singleton-synconce)
   - [Object Pool (sync.Pool)](#object-pool-syncpool)
3. [Structural Patterns](#structural-patterns)
   - [Adapter](#adapter-via-interfaces)
   - [Decorator (Middleware)](#decorator-middleware-pattern)
   - [Composite](#composite)
   - [Facade](#facade)
   - [Proxy](#proxy)
4. [Behavioral Patterns](#behavioral-patterns)
   - [Strategy](#strategy-first-class-functions-and-interfaces)
   - [Observer (Channel-Based)](#observer-channel-based)
   - [Iterator](#iterator)
   - [Command](#command)
   - [Pipeline Pattern](#pipeline-pattern-channel-pipelines)
   - [Fan-Out / Fan-In](#fan-out--fan-in)
   - [Pub/Sub](#pubsub)
5. [Concurrency Patterns](#concurrency-patterns)
   - [Worker Pool](#worker-pool)
   - [Semaphore](#semaphore)
   - [Error Group](#error-group)
   - [Context-Based Cancellation Tree](#context-based-cancellation-tree)
6. [Anti-Patterns to Avoid in Go](#anti-patterns-to-avoid-in-go)
7. [When Patterns Help vs When They Hurt](#when-patterns-help-vs-when-they-hurt)

---

## Why Patterns Differ in Go

The "Gang of Four" (GoF) design patterns were conceived in a world of class-based
object-oriented programming -- C++, Java, Smalltalk. Go is a fundamentally different
language, and trying to transliterate GoF patterns into Go often produces code that
fights the language rather than working with it.

### Key Language Differences

| Feature | Java / C++ | Go |
|---|---|---|
| Inheritance | Class hierarchies, abstract classes | **No inheritance at all** |
| Polymorphism | Virtual methods, class-based | **Implicit interfaces** |
| Composition | Has-a via fields, mixins | **Embedding (struct promotion)** |
| Functions | Must live inside a class | **First-class citizens** |
| Concurrency | Threads + locks (library) | **Goroutines + channels (language)** |
| Generics | Templates / Generics | **Type parameters (Go 1.18+)** |
| Constructors | Special language construct | **Ordinary functions (New...)** |

### Three Principles That Reshape Every Pattern

**1. No classes, no inheritance -- composition over inheritance**

Go structs have no constructors, no destructors, no virtual dispatch tables, and no
class hierarchies. You compose behavior by embedding one struct inside another, which
promotes its methods. This eliminates entire categories of patterns whose sole purpose
is to work around rigid class hierarchies (e.g., the Template Method pattern is often
just an interface + a function in Go).

```go
// Embedding gives you composition with method promotion.
type Logger struct{}

func (l Logger) Log(msg string) { fmt.Println(msg) }

type Server struct {
    Logger // embedded -- Server now has a Log method
    addr string
}

func main() {
    s := Server{addr: ":8080"}
    s.Log("starting server") // promoted from Logger
}
```

**2. Interfaces are satisfied implicitly**

In Java, a class explicitly declares `implements Sortable`. In Go, if your type has
the right method set, it satisfies the interface -- no declaration needed. This makes
the Adapter pattern almost trivial and makes Strategy effortless.

```go
// Any type with a ServeHTTP method is an http.Handler.
// No "implements" keyword required.
type healthCheck struct{}

func (h healthCheck) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}
```

**3. Functions are first-class values**

Many behavioral patterns (Strategy, Command, Observer) that require full class
hierarchies in Java collapse into a single function type in Go.

```go
// The Strategy pattern is often just a function parameter.
func Process(data []byte, compress func([]byte) []byte) []byte {
    return compress(data)
}
```

> **Tip:** Before reaching for a design pattern, ask: "Does Go already have a simpler
> idiom for this?" More often than not, the answer is yes.

---

## Creational Patterns

Creational patterns deal with object construction. In Go there are no constructors, so
we use ordinary functions -- but the *way* we structure those functions matters.

---

### Functional Options Pattern

This is arguably the most idiomatic Go pattern. It solves the problem of constructors
with many optional parameters without resorting to config structs that grow unmanageable
or builder chains that feel unnatural.

#### The Problem

Imagine configuring an HTTP server:

```go
// Approach 1: Positional arguments -- unreadable, rigid.
func NewServer(addr string, port int, timeout time.Duration, maxConn int, tls bool) *Server

// Approach 2: Config struct -- better, but zero values are ambiguous.
// Does maxConn=0 mean "default" or "no connections allowed"?
type Config struct {
    Addr    string
    Port    int
    Timeout time.Duration
    MaxConn int
    TLS     bool
}
func NewServer(cfg Config) *Server
```

#### The Solution: Functional Options

Functional options use variadic function parameters to configure the object. Each
option is a function that modifies the struct being built.

```go
package server

import (
    "crypto/tls"
    "log"
    "time"
)

// Server is the thing we are constructing.
type Server struct {
    addr      string
    port      int
    timeout   time.Duration
    maxConn   int
    tlsConfig *tls.Config
    logger    *log.Logger
}

// Option is a function that configures a Server.
type Option func(*Server)

// WithPort sets the server port.
func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

// WithTimeout sets the request timeout.
func WithTimeout(t time.Duration) Option {
    return func(s *Server) {
        s.timeout = t
    }
}

// WithMaxConnections sets the maximum concurrent connections.
func WithMaxConnections(n int) Option {
    return func(s *Server) {
        s.maxConn = n
    }
}

// WithTLS enables TLS with the given config.
func WithTLS(cfg *tls.Config) Option {
    return func(s *Server) {
        s.tlsConfig = cfg
    }
}

// WithLogger sets a custom logger.
func WithLogger(l *log.Logger) Option {
    return func(s *Server) {
        s.logger = l
    }
}

// New creates a Server with sensible defaults, then applies options.
func New(addr string, opts ...Option) *Server {
    // Start with sensible defaults.
    s := &Server{
        addr:    addr,
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 1000,
    }

    // Apply each option.
    for _, opt := range opts {
        opt(s)
    }

    return s
}
```

Usage is clean and self-documenting:

```go
func main() {
    // Only specify what you need to change.
    srv := server.New("localhost",
        server.WithPort(9090),
        server.WithTimeout(60*time.Second),
        server.WithMaxConnections(5000),
    )
    _ = srv
}
```

#### Functional Options with Validation

Options can return errors for validation:

```go
// OptionErr is an option that can fail.
type OptionErr func(*Server) error

func WithPort(port int) OptionErr {
    return func(s *Server) error {
        if port < 1 || port > 65535 {
            return fmt.Errorf("invalid port: %d", port)
        }
        s.port = port
        return nil
    }
}

func New(addr string, opts ...OptionErr) (*Server, error) {
    s := &Server{addr: addr, port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        if err := opt(s); err != nil {
            return nil, fmt.Errorf("applying option: %w", err)
        }
    }
    return s, nil
}
```

#### Real-World Examples in the Standard Library

- `net/http.Server` -- not functional options, but the *pattern* was popularized by
  Rob Pike and Dave Cheney to address exactly the kind of config explosion that
  `http.Server` has.
- `google.golang.org/grpc.Dial(target, ...DialOption)` -- gRPC uses this pattern
  extensively.
- `go.uber.org/zap.New(core, ...Option)` -- Uber's logger.

> **Tip:** Name your option constructors `WithXxx`. It reads like English:
> `NewServer("addr", WithPort(80), WithTimeout(5*time.Second))`.

---

### Builder Pattern

The Builder pattern constructs complex objects step by step. In Go, it is most useful
when the construction process itself has multiple stages or when you need to produce
different representations of the same construction process.

```go
package query

import (
    "fmt"
    "strings"
)

// Query represents a SQL query.
type Query struct {
    table      string
    conditions []string
    orderBy    string
    limit      int
    columns    []string
}

// Builder constructs Query objects step by step.
type Builder struct {
    query Query
    err   error // accumulate first error
}

// NewBuilder starts building a query for the given table.
func NewBuilder(table string) *Builder {
    return &Builder{
        query: Query{
            table:   table,
            columns: []string{"*"},
        },
    }
}

// Select sets the columns to retrieve.
func (b *Builder) Select(cols ...string) *Builder {
    if b.err != nil {
        return b
    }
    if len(cols) == 0 {
        b.err = fmt.Errorf("select: at least one column required")
        return b
    }
    b.query.columns = cols
    return b
}

// Where adds a condition.
func (b *Builder) Where(condition string) *Builder {
    if b.err != nil {
        return b
    }
    b.query.conditions = append(b.query.conditions, condition)
    return b
}

// OrderBy sets the ordering column.
func (b *Builder) OrderBy(col string) *Builder {
    if b.err != nil {
        return b
    }
    b.query.orderBy = col
    return b
}

// Limit sets the maximum rows.
func (b *Builder) Limit(n int) *Builder {
    if b.err != nil {
        return b
    }
    if n < 0 {
        b.err = fmt.Errorf("limit: must be non-negative, got %d", n)
        return b
    }
    b.query.limit = n
    return b
}

// Build finalizes and returns the Query or the first error encountered.
func (b *Builder) Build() (string, error) {
    if b.err != nil {
        return "", b.err
    }

    var sb strings.Builder
    sb.WriteString("SELECT ")
    sb.WriteString(strings.Join(b.query.columns, ", "))
    sb.WriteString(" FROM ")
    sb.WriteString(b.query.table)

    if len(b.query.conditions) > 0 {
        sb.WriteString(" WHERE ")
        sb.WriteString(strings.Join(b.query.conditions, " AND "))
    }
    if b.query.orderBy != "" {
        sb.WriteString(" ORDER BY ")
        sb.WriteString(b.query.orderBy)
    }
    if b.query.limit > 0 {
        fmt.Fprintf(&sb, " LIMIT %d", b.query.limit)
    }

    return sb.String(), nil
}
```

Usage:

```go
func main() {
    sql, err := query.NewBuilder("users").
        Select("id", "name", "email").
        Where("active = true").
        Where("age > 18").
        OrderBy("name").
        Limit(100).
        Build()

    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(sql)
    // SELECT id, name, email FROM users WHERE active = true AND age > 18 ORDER BY name LIMIT 100
}
```

> **Tip:** The error-accumulation trick (`if b.err != nil { return b }`) prevents
> panics or silent corruption partway through a chain. The caller only checks for
> errors once at the end, in `Build()`.

#### Builder vs Functional Options

| Use Case | Prefer |
|---|---|
| Configuring a struct at creation time | Functional Options |
| Multi-step construction with ordering | Builder |
| Generating output (SQL, HTML, protobuf) | Builder |
| Library APIs consumed by third parties | Functional Options |

---

### Factory Functions

Go does not have constructors. Instead, the convention is to write `NewXxx` functions
that return an initialized value (or pointer). This is the simplest and most common
creational pattern in Go.

```go
package user

import (
    "errors"
    "time"

    "github.com/google/uuid"
)

// User represents a domain entity.
type User struct {
    ID        string
    Name      string
    Email     string
    CreatedAt time.Time
}

// New creates a User with validated inputs.
func New(name, email string) (*User, error) {
    if name == "" {
        return nil, errors.New("user: name is required")
    }
    if email == "" {
        return nil, errors.New("user: email is required")
    }

    return &User{
        ID:        uuid.NewString(),
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
    }, nil
}
```

#### Interface-Based Factories

When you need to return different concrete types behind a common interface, use a
factory function that returns the interface.

```go
package storage

import "fmt"

// Store is the common interface.
type Store interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

// NewStore creates the appropriate Store implementation based on the driver name.
func NewStore(driver string, connStr string) (Store, error) {
    switch driver {
    case "memory":
        return newMemoryStore(), nil
    case "redis":
        return newRedisStore(connStr)
    case "postgres":
        return newPostgresStore(connStr)
    default:
        return nil, fmt.Errorf("unknown storage driver: %s", driver)
    }
}
```

#### Registry-Based Factory

For plugin-like extensibility, use a registry:

```go
package codec

import (
    "fmt"
    "sync"
)

// Encoder is the common interface.
type Encoder interface {
    Encode(v any) ([]byte, error)
}

// Factory is a function that creates an Encoder.
type Factory func() Encoder

var (
    mu       sync.RWMutex
    registry = make(map[string]Factory)
)

// Register adds a new encoder factory under the given name.
func Register(name string, f Factory) {
    mu.Lock()
    defer mu.Unlock()
    if _, dup := registry[name]; dup {
        panic(fmt.Sprintf("codec: duplicate registration for %q", name))
    }
    registry[name] = f
}

// New creates an Encoder by name.
func New(name string) (Encoder, error) {
    mu.RLock()
    defer mu.RUnlock()
    f, ok := registry[name]
    if !ok {
        return nil, fmt.Errorf("codec: unknown encoder %q", name)
    }
    return f(), nil
}
```

Packages register themselves in their `init()`:

```go
package json

import "myapp/codec"

func init() {
    codec.Register("json", func() codec.Encoder {
        return &jsonEncoder{}
    })
}
```

This is exactly the pattern used by `database/sql` for driver registration.

---

### Singleton (sync.Once)

The Singleton pattern ensures only one instance of something exists. In Go, the
canonical way is `sync.Once`.

```go
package db

import (
    "database/sql"
    "sync"

    _ "github.com/lib/pq"
)

var (
    instance *sql.DB
    once     sync.Once
    initErr  error
)

// Connection returns the singleton database connection.
func Connection() (*sql.DB, error) {
    once.Do(func() {
        instance, initErr = sql.Open("postgres", "postgres://localhost/mydb?sslmode=disable")
        if initErr != nil {
            return
        }
        initErr = instance.Ping()
    })
    return instance, initErr
}
```

#### How `sync.Once` Works

- The function passed to `Do` executes **exactly once**, even if multiple goroutines
  call `Do` concurrently.
- Subsequent calls to `Do` return immediately -- they do not re-execute the function.
- If the function panics, `Do` considers it "done" and will not retry. (Since Go 1.21,
  `sync.OnceFunc`, `sync.OnceValue`, and `sync.OnceValues` provide cleaner APIs.)

```go
// Go 1.21+ style using sync.OnceValues
package db

import (
    "database/sql"
    "sync"

    _ "github.com/lib/pq"
)

var getConnection = sync.OnceValues(func() (*sql.DB, error) {
    conn, err := sql.Open("postgres", "postgres://localhost/mydb?sslmode=disable")
    if err != nil {
        return nil, err
    }
    if err := conn.Ping(); err != nil {
        return nil, err
    }
    return conn, nil
})

// Usage:
// conn, err := db.getConnection()
```

> **Warning:** Think carefully before using Singleton. It introduces global state,
> makes testing harder, and hides dependencies. Prefer dependency injection -- pass
> the database connection as a parameter rather than fetching it from a global.

---

### Object Pool (sync.Pool)

`sync.Pool` is a concurrency-safe pool of temporary objects that reduces GC pressure
by reusing allocations.

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

// bufPool is a pool of *bytes.Buffer.
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func formatMessage(name string, age int) string {
    // Get a buffer from the pool.
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset() // Always reset before use!

    // Use the buffer.
    fmt.Fprintf(buf, "Name: %s, Age: %d", name, age)
    result := buf.String()

    // Return the buffer to the pool.
    bufPool.Put(buf)

    return result
}

func main() {
    msg := formatMessage("Alice", 30)
    fmt.Println(msg)
}
```

#### A More Realistic Example: JSON Encoder Pool

```go
package jsonpool

import (
    "bytes"
    "encoding/json"
    "io"
    "sync"
)

var encoderPool = sync.Pool{
    New: func() any {
        buf := new(bytes.Buffer)
        return &pooledEncoder{
            buf: buf,
            enc: json.NewEncoder(buf),
        }
    },
}

type pooledEncoder struct {
    buf *bytes.Buffer
    enc *json.Encoder
}

// WriteJSON encodes v as JSON and writes it to w.
// It reuses buffers from a pool to reduce allocations.
func WriteJSON(w io.Writer, v any) error {
    pe := encoderPool.Get().(*pooledEncoder)
    defer encoderPool.Put(pe)

    pe.buf.Reset()
    pe.enc.SetEscapeHTML(false)

    if err := pe.enc.Encode(v); err != nil {
        return err
    }

    _, err := pe.buf.WriteTo(w)
    return err
}
```

#### sync.Pool Semantics

| Property | Behavior |
|---|---|
| Thread safety | Safe for concurrent Get/Put |
| Lifetime | Objects may be collected by GC at **any** time |
| Guarantee | `New` is called if pool is empty; no guarantee of reuse |
| Use case | Short-lived, frequently allocated objects (buffers, encoders) |

> **Warning:** Never store state in pooled objects that persists across uses. Always
> `Reset()` or zero out the object after `Get()`. Failing to do so leads to data
> leaks between requests -- a serious security bug in web servers.

---

## Structural Patterns

Structural patterns deal with composing types to form larger structures while keeping
them flexible and efficient.

---

### Adapter (via Interfaces)

The Adapter pattern converts one interface into another that a client expects. In Go,
implicit interface satisfaction makes this almost effortless.

#### Example: Adapting a Third-Party Logger

Suppose your application defines its own `Logger` interface, but you want to use a
third-party logger (`zap`, `logrus`, etc.):

```go
package logging

// Logger is your application's logging interface.
type Logger interface {
    Debug(msg string, fields ...Field)
    Info(msg string, fields ...Field)
    Error(msg string, fields ...Field)
}

// Field is a structured logging field.
type Field struct {
    Key   string
    Value any
}
```

Now adapt `log/slog` (standard library, Go 1.21+) to satisfy your interface:

```go
package logging

import "log/slog"

// SlogAdapter adapts *slog.Logger to your Logger interface.
type SlogAdapter struct {
    sl *slog.Logger
}

// NewSlogAdapter wraps a *slog.Logger.
func NewSlogAdapter(sl *slog.Logger) *SlogAdapter {
    return &SlogAdapter{sl: sl}
}

func (a *SlogAdapter) toSlogAttrs(fields []Field) []any {
    attrs := make([]any, 0, len(fields)*2)
    for _, f := range fields {
        attrs = append(attrs, f.Key, f.Value)
    }
    return attrs
}

func (a *SlogAdapter) Debug(msg string, fields ...Field) {
    a.sl.Debug(msg, a.toSlogAttrs(fields)...)
}

func (a *SlogAdapter) Info(msg string, fields ...Field) {
    a.sl.Info(msg, a.toSlogAttrs(fields)...)
}

func (a *SlogAdapter) Error(msg string, fields ...Field) {
    a.sl.Error(msg, a.toSlogAttrs(fields)...)
}
```

#### The `http.HandlerFunc` Adapter

The standard library has a famous adapter: `http.HandlerFunc` adapts a bare function
into the `http.Handler` interface.

```go
// In net/http:
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

This lets you write:

```go
http.Handle("/health", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
}))
```

The pattern -- a named function type that implements an interface by calling itself --
is a uniquely Go idiom. Use it whenever you have a single-method interface.

---

### Decorator (Middleware Pattern)

The Decorator pattern wraps an object to add behavior without modifying its interface.
In Go, this is most commonly seen as HTTP middleware.

#### HTTP Middleware

```go
package middleware

import (
    "log"
    "net/http"
    "time"
)

// Middleware is a function that wraps an http.Handler.
type Middleware func(http.Handler) http.Handler

// Logging logs every request.
func Logging(logger *log.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            // Wrap ResponseWriter to capture status code.
            ww := &responseWriter{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(ww, r)
            logger.Printf("%s %s %d %s",
                r.Method, r.URL.Path, ww.status, time.Since(start))
        })
    }
}

// responseWriter captures the status code.
type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}

// Auth rejects unauthenticated requests.
func Auth(tokenValidator func(string) bool) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("Authorization")
            if !tokenValidator(token) {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// RateLimit limits requests (simplified).
func RateLimit(requestsPerSecond int) Middleware {
    limiter := time.NewTicker(time.Second / time.Duration(requestsPerSecond))
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            <-limiter.C
            next.ServeHTTP(w, r)
        })
    }
}

// Chain applies middleware in order (first middleware is outermost).
func Chain(h http.Handler, mws ...Middleware) http.Handler {
    // Apply in reverse so the first middleware in the list is the outermost.
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
```

Usage:

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/data", handleData)

    logger := log.New(os.Stdout, "", log.LstdFlags)

    handler := middleware.Chain(mux,
        middleware.Logging(logger),
        middleware.Auth(validateToken),
        middleware.RateLimit(100),
    )

    http.ListenAndServe(":8080", handler)
}
```

#### Generic Decorator

The decorator pattern is not limited to HTTP. Here is a generic decorator for any
function:

```go
package main

import (
    "fmt"
    "time"
)

// Measurable is any function we want to time.
type Measurable[T any] func(T) (T, error)

// WithTiming decorates a Measurable with execution time logging.
func WithTiming[T any](name string, fn Measurable[T]) Measurable[T] {
    return func(input T) (T, error) {
        start := time.Now()
        result, err := fn(input)
        fmt.Printf("[%s] took %s\n", name, time.Since(start))
        return result, err
    }
}
```

---

### Composite

The Composite pattern treats individual objects and compositions of objects uniformly
through a common interface. This is ideal for tree structures.

```go
package ui

import (
    "fmt"
    "strings"
)

// Component is the common interface for all UI elements.
type Component interface {
    Render(indent int) string
}

// Text is a leaf component.
type Text struct {
    Content string
}

func (t *Text) Render(indent int) string {
    return strings.Repeat("  ", indent) + t.Content
}

// Container is a composite that holds child components.
type Container struct {
    Name     string
    Children []Component
}

func (c *Container) Add(children ...Component) {
    c.Children = append(c.Children, children...)
}

func (c *Container) Render(indent int) string {
    var sb strings.Builder
    prefix := strings.Repeat("  ", indent)
    sb.WriteString(fmt.Sprintf("%s<%s>\n", prefix, c.Name))
    for _, child := range c.Children {
        sb.WriteString(child.Render(indent+1) + "\n")
    }
    sb.WriteString(fmt.Sprintf("%s</%s>", prefix, c.Name))
    return sb.String()
}

// Button is a leaf component.
type Button struct {
    Label string
}

func (b *Button) Render(indent int) string {
    return fmt.Sprintf("%s[Button: %s]", strings.Repeat("  ", indent), b.Label)
}
```

Usage:

```go
func main() {
    page := &ui.Container{Name: "page"}
    header := &ui.Container{Name: "header"}
    header.Add(&ui.Text{Content: "My Application"})

    body := &ui.Container{Name: "body"}
    form := &ui.Container{Name: "form"}
    form.Add(
        &ui.Text{Content: "Enter your name:"},
        &ui.Button{Label: "Submit"},
    )
    body.Add(form)

    page.Add(header, body)

    fmt.Println(page.Render(0))
}
```

Output:

```
<page>
  <header>
    My Application
  </header>
  <body>
    <form>
      Enter your name:
      [Button: Submit]
    </form>
  </body>
</page>
```

#### Real-World: File System

```go
package fs

import "fmt"

type Entry interface {
    Name() string
    Size() int64
    Print(indent int)
}

type File struct {
    name string
    size int64
}

func (f *File) Name() string  { return f.name }
func (f *File) Size() int64   { return f.size }
func (f *File) Print(indent int) {
    fmt.Printf("%*s%s (%d bytes)\n", indent*2, "", f.name, f.size)
}

type Directory struct {
    name     string
    children []Entry
}

func (d *Directory) Name() string { return d.name }
func (d *Directory) Size() int64 {
    var total int64
    for _, child := range d.children {
        total += child.Size()
    }
    return total
}
func (d *Directory) Add(entries ...Entry) { d.children = append(d.children, entries...) }
func (d *Directory) Print(indent int) {
    fmt.Printf("%*s%s/ (%d bytes total)\n", indent*2, "", d.name, d.Size())
    for _, child := range d.children {
        child.Print(indent + 1)
    }
}
```

---

### Facade

The Facade pattern provides a simplified interface to a complex subsystem. In Go, this
is often just a struct that orchestrates several components.

```go
package order

import (
    "fmt"
    "time"
)

// --- Subsystem components ---

type InventoryService struct{}

func (s *InventoryService) CheckStock(productID string, qty int) error {
    // Imagine a database call here.
    fmt.Printf("  [Inventory] Checking stock for %s (qty: %d)\n", productID, qty)
    return nil
}

func (s *InventoryService) Reserve(productID string, qty int) error {
    fmt.Printf("  [Inventory] Reserved %d of %s\n", qty, productID)
    return nil
}

type PaymentService struct{}

func (s *PaymentService) Charge(userID string, amount float64) (string, error) {
    txID := fmt.Sprintf("tx_%d", time.Now().UnixNano())
    fmt.Printf("  [Payment] Charged $%.2f to user %s (tx: %s)\n", amount, userID, txID)
    return txID, nil
}

type ShippingService struct{}

func (s *ShippingService) CreateShipment(orderID, address string) (string, error) {
    trackingID := fmt.Sprintf("ship_%d", time.Now().UnixNano())
    fmt.Printf("  [Shipping] Created shipment %s for order %s\n", trackingID, orderID)
    return trackingID, nil
}

type NotificationService struct{}

func (s *NotificationService) SendConfirmation(userID, orderID string) error {
    fmt.Printf("  [Notification] Sent confirmation to user %s for order %s\n", userID, orderID)
    return nil
}

// --- Facade ---

// OrderFacade provides a simplified interface for placing orders.
type OrderFacade struct {
    inventory    *InventoryService
    payment      *PaymentService
    shipping     *ShippingService
    notification *NotificationService
}

func NewOrderFacade() *OrderFacade {
    return &OrderFacade{
        inventory:    &InventoryService{},
        payment:      &PaymentService{},
        shipping:     &ShippingService{},
        notification: &NotificationService{},
    }
}

// PlaceOrder orchestrates the entire order flow.
func (f *OrderFacade) PlaceOrder(userID, productID, address string, qty int, pricePerUnit float64) error {
    orderID := fmt.Sprintf("ord_%d", time.Now().UnixNano())
    fmt.Printf("Placing order %s...\n", orderID)

    // Step 1: Check inventory.
    if err := f.inventory.CheckStock(productID, qty); err != nil {
        return fmt.Errorf("stock check failed: %w", err)
    }

    // Step 2: Reserve inventory.
    if err := f.inventory.Reserve(productID, qty); err != nil {
        return fmt.Errorf("reservation failed: %w", err)
    }

    // Step 3: Charge payment.
    amount := float64(qty) * pricePerUnit
    txID, err := f.payment.Charge(userID, amount)
    if err != nil {
        // TODO: release inventory reservation
        return fmt.Errorf("payment failed: %w", err)
    }
    _ = txID

    // Step 4: Create shipment.
    trackingID, err := f.shipping.CreateShipment(orderID, address)
    if err != nil {
        // TODO: refund payment, release inventory
        return fmt.Errorf("shipping failed: %w", err)
    }
    _ = trackingID

    // Step 5: Notify customer.
    if err := f.notification.SendConfirmation(userID, orderID); err != nil {
        // Non-critical -- log but don't fail the order.
        fmt.Printf("  [Warning] notification failed: %v\n", err)
    }

    fmt.Printf("Order %s placed successfully!\n", orderID)
    return nil
}
```

---

### Proxy

The Proxy pattern provides a surrogate or placeholder for another object. Common uses:
lazy initialization, access control, caching, and logging.

#### Caching Proxy

```go
package proxy

import (
    "sync"
    "time"
)

// DataFetcher is the interface for fetching data.
type DataFetcher interface {
    Fetch(key string) ([]byte, error)
}

// RemoteFetcher is the real implementation that calls a remote service.
type RemoteFetcher struct {
    BaseURL string
}

func (f *RemoteFetcher) Fetch(key string) ([]byte, error) {
    // Simulate expensive remote call.
    time.Sleep(100 * time.Millisecond)
    return []byte("data for " + key), nil
}

// CachingProxy wraps a DataFetcher and caches results.
type CachingProxy struct {
    fetcher DataFetcher
    cache   map[string]cacheEntry
    ttl     time.Duration
    mu      sync.RWMutex
}

type cacheEntry struct {
    data      []byte
    expiresAt time.Time
}

func NewCachingProxy(fetcher DataFetcher, ttl time.Duration) *CachingProxy {
    return &CachingProxy{
        fetcher: fetcher,
        cache:   make(map[string]cacheEntry),
        ttl:     ttl,
    }
}

func (p *CachingProxy) Fetch(key string) ([]byte, error) {
    // Check cache first (read lock).
    p.mu.RLock()
    entry, ok := p.cache[key]
    p.mu.RUnlock()

    if ok && time.Now().Before(entry.expiresAt) {
        return entry.data, nil // cache hit
    }

    // Cache miss -- fetch from real source.
    data, err := p.fetcher.Fetch(key)
    if err != nil {
        return nil, err
    }

    // Store in cache (write lock).
    p.mu.Lock()
    p.cache[key] = cacheEntry{
        data:      data,
        expiresAt: time.Now().Add(p.ttl),
    }
    p.mu.Unlock()

    return data, nil
}
```

#### Access Control Proxy

```go
package proxy

import "fmt"

type Document interface {
    Read() string
    Write(content string) error
}

type RealDocument struct {
    content string
}

func (d *RealDocument) Read() string             { return d.content }
func (d *RealDocument) Write(content string) error { d.content = content; return nil }

// ProtectedDocument checks permissions before allowing operations.
type ProtectedDocument struct {
    doc      Document
    userRole string
}

func NewProtectedDocument(doc Document, role string) *ProtectedDocument {
    return &ProtectedDocument{doc: doc, userRole: role}
}

func (p *ProtectedDocument) Read() string {
    return p.doc.Read() // everyone can read
}

func (p *ProtectedDocument) Write(content string) error {
    if p.userRole != "admin" && p.userRole != "editor" {
        return fmt.Errorf("permission denied: role %q cannot write", p.userRole)
    }
    return p.doc.Write(content)
}
```

---

## Behavioral Patterns

Behavioral patterns define how objects communicate and distribute responsibilities.

---

### Strategy (First-Class Functions and Interfaces)

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes
them interchangeable. In Go, this is often just a function parameter.

#### Function-Based Strategy

```go
package compression

import (
    "bytes"
    "compress/gzip"
    "compress/zlib"
    "io"
)

// CompressFunc is a strategy for compression.
type CompressFunc func(data []byte) ([]byte, error)

// Gzip compression strategy.
func Gzip(data []byte) ([]byte, error) {
    var buf bytes.Buffer
    w := gzip.NewWriter(&buf)
    if _, err := w.Write(data); err != nil {
        return nil, err
    }
    if err := w.Close(); err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}

// Zlib compression strategy.
func Zlib(data []byte) ([]byte, error) {
    var buf bytes.Buffer
    w := zlib.NewWriter(&buf)
    if _, err := w.Write(data); err != nil {
        return nil, err
    }
    if err := w.Close(); err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}

// NoCompression strategy (passthrough).
func NoCompression(data []byte) ([]byte, error) {
    return data, nil
}

// Archiver uses a compression strategy to compress files.
type Archiver struct {
    compress CompressFunc
}

func NewArchiver(strategy CompressFunc) *Archiver {
    return &Archiver{compress: strategy}
}

func (a *Archiver) Archive(data []byte) ([]byte, error) {
    return a.compress(data)
}
```

Usage:

```go
func main() {
    data := []byte("some large payload to compress...")

    // Choose strategy at runtime.
    gzipArchiver := compression.NewArchiver(compression.Gzip)
    compressed, _ := gzipArchiver.Archive(data)
    fmt.Printf("Gzip: %d -> %d bytes\n", len(data), len(compressed))

    // Swap strategy without changing Archiver.
    plainArchiver := compression.NewArchiver(compression.NoCompression)
    plain, _ := plainArchiver.Archive(data)
    fmt.Printf("None: %d -> %d bytes\n", len(data), len(plain))
}
```

#### Interface-Based Strategy (When You Need State)

When a strategy needs its own configuration or state, use an interface:

```go
package retry

import (
    "math"
    "math/rand"
    "time"
)

// Backoff defines a strategy for computing retry delays.
type Backoff interface {
    Duration(attempt int) time.Duration
}

// ConstantBackoff always waits the same duration.
type ConstantBackoff struct {
    Delay time.Duration
}

func (b ConstantBackoff) Duration(attempt int) time.Duration {
    return b.Delay
}

// ExponentialBackoff increases delay exponentially with jitter.
type ExponentialBackoff struct {
    Base      time.Duration
    MaxDelay  time.Duration
}

func (b ExponentialBackoff) Duration(attempt int) time.Duration {
    delay := time.Duration(float64(b.Base) * math.Pow(2, float64(attempt)))
    if delay > b.MaxDelay {
        delay = b.MaxDelay
    }
    // Add jitter: +/- 25%.
    jitter := time.Duration(rand.Int63n(int64(delay) / 2)) - delay/4
    return delay + jitter
}

// Retry executes fn up to maxAttempts times using the given backoff strategy.
func Retry(maxAttempts int, backoff Backoff, fn func() error) error {
    var err error
    for i := 0; i < maxAttempts; i++ {
        if err = fn(); err == nil {
            return nil
        }
        if i < maxAttempts-1 {
            time.Sleep(backoff.Duration(i))
        }
    }
    return err
}
```

---

### Observer (Channel-Based)

The Observer pattern defines a one-to-many dependency: when one object changes state,
all dependents are notified. In Go, channels are the natural notification mechanism.

```go
package events

import (
    "fmt"
    "sync"
)

// Event represents something that happened.
type Event struct {
    Type    string
    Payload any
}

// EventBus is a simple publish-subscribe system using channels.
type EventBus struct {
    mu          sync.RWMutex
    subscribers map[string][]chan Event
}

// NewEventBus creates a new EventBus.
func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string][]chan Event),
    }
}

// Subscribe returns a channel that receives events of the given type.
// bufSize controls the channel buffer; 0 means unbuffered.
func (eb *EventBus) Subscribe(eventType string, bufSize int) <-chan Event {
    ch := make(chan Event, bufSize)
    eb.mu.Lock()
    eb.subscribers[eventType] = append(eb.subscribers[eventType], ch)
    eb.mu.Unlock()
    return ch
}

// Publish sends an event to all subscribers of that event type.
// Non-blocking: if a subscriber's channel is full, the event is dropped for that subscriber.
func (eb *EventBus) Publish(evt Event) {
    eb.mu.RLock()
    subs := eb.subscribers[evt.Type]
    eb.mu.RUnlock()

    for _, ch := range subs {
        select {
        case ch <- evt:
        default:
            fmt.Printf("warning: dropped event %s for slow subscriber\n", evt.Type)
        }
    }
}

// Unsubscribe removes a channel from the subscriber list and closes it.
func (eb *EventBus) Unsubscribe(eventType string, ch <-chan Event) {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    subs := eb.subscribers[eventType]
    for i, sub := range subs {
        if sub == ch {
            eb.subscribers[eventType] = append(subs[:i], subs[i+1:]...)
            close(sub)
            return
        }
    }
}

// Close shuts down all subscriber channels.
func (eb *EventBus) Close() {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    for eventType, subs := range eb.subscribers {
        for _, ch := range subs {
            close(ch)
        }
        delete(eb.subscribers, eventType)
    }
}
```

Usage:

```go
func main() {
    bus := events.NewEventBus()
    defer bus.Close()

    // Subscriber 1: log all user events.
    userEvents := bus.Subscribe("user.created", 10)
    go func() {
        for evt := range userEvents {
            fmt.Printf("[Logger] User created: %v\n", evt.Payload)
        }
    }()

    // Subscriber 2: send welcome email.
    welcomeEvents := bus.Subscribe("user.created", 10)
    go func() {
        for evt := range welcomeEvents {
            fmt.Printf("[Email] Sending welcome to: %v\n", evt.Payload)
        }
    }()

    // Publish events.
    bus.Publish(events.Event{Type: "user.created", Payload: "alice@example.com"})
    bus.Publish(events.Event{Type: "user.created", Payload: "bob@example.com"})

    time.Sleep(100 * time.Millisecond) // Give goroutines time to process.
}
```

#### Type-Safe Observer with Generics

```go
package events

import "sync"

// TypedBus is a type-safe event bus for events of type T.
type TypedBus[T any] struct {
    mu   sync.RWMutex
    subs []chan T
}

func NewTypedBus[T any]() *TypedBus[T] {
    return &TypedBus[T]{}
}

func (b *TypedBus[T]) Subscribe(bufSize int) <-chan T {
    ch := make(chan T, bufSize)
    b.mu.Lock()
    b.subs = append(b.subs, ch)
    b.mu.Unlock()
    return ch
}

func (b *TypedBus[T]) Publish(event T) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, ch := range b.subs {
        select {
        case ch <- event:
        default:
        }
    }
}

func (b *TypedBus[T]) Close() {
    b.mu.Lock()
    defer b.mu.Unlock()
    for _, ch := range b.subs {
        close(ch)
    }
    b.subs = nil
}
```

---

### Iterator

Go provides built-in iteration with `for...range`. But there are times when you need
custom iteration, and Go 1.23 introduced the `iter` package for standardized iterator
support.

#### Built-In Range

```go
// Slices
for i, v := range []string{"a", "b", "c"} {
    fmt.Println(i, v)
}

// Maps
for k, v := range map[string]int{"x": 1, "y": 2} {
    fmt.Println(k, v)
}

// Channels
ch := make(chan int)
go func() { ch <- 1; ch <- 2; close(ch) }()
for v := range ch {
    fmt.Println(v)
}
```

#### Closure-Based Iterators (Pre-1.23)

```go
package collection

// IntIterator is a function that returns the next value and whether it exists.
type IntIterator func() (value int, ok bool)

// FilteredRange returns an iterator over values in [min, max] from a slice.
func FilteredRange(data []int, min, max int) IntIterator {
    index := 0
    return func() (int, bool) {
        for index < len(data) {
            v := data[index]
            index++
            if v >= min && v <= max {
                return v, true
            }
        }
        return 0, false
    }
}
```

Usage:

```go
data := []int{1, 5, 3, 8, 2, 9, 4, 7, 6}
next := collection.FilteredRange(data, 3, 7)
for v, ok := next(); ok; v, ok = next() {
    fmt.Println(v) // 5, 3, 4, 7, 6
}
```

#### Go 1.23+ Range-Over-Function Iterators (iter Package)

Go 1.23 introduced range-over-function iterators. A function with the signature
`func(yield func(V) bool)` can be used directly in a `for...range` loop.

```go
package main

import (
    "fmt"
    "iter"
)

// Fibonacci returns an iterator that yields Fibonacci numbers up to max.
func Fibonacci(max int) iter.Seq[int] {
    return func(yield func(int) bool) {
        a, b := 0, 1
        for a <= max {
            if !yield(a) {
                return // caller broke out of the loop
            }
            a, b = b, a+b
        }
    }
}

func main() {
    for v := range Fibonacci(100) {
        fmt.Println(v) // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89
    }
}
```

#### Two-Value Iterators (iter.Seq2)

```go
package main

import (
    "fmt"
    "iter"
)

// Enumerate pairs each element with its index, like Python's enumerate().
func Enumerate[T any](s []T) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        for i, v := range s {
            if !yield(i, v) {
                return
            }
        }
    }
}

func main() {
    names := []string{"Alice", "Bob", "Charlie"}
    for i, name := range Enumerate(names) {
        fmt.Printf("%d: %s\n", i, name)
    }
}
```

#### Composable Iterators

```go
package itertools

import "iter"

// Filter returns an iterator that only yields elements satisfying pred.
func Filter[T any](seq iter.Seq[T], pred func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range seq {
            if pred(v) {
                if !yield(v) {
                    return
                }
            }
        }
    }
}

// Map transforms each element.
func Map[T, U any](seq iter.Seq[T], fn func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range seq {
            if !yield(fn(v)) {
                return
            }
        }
    }
}

// Take yields at most n elements.
func Take[T any](seq iter.Seq[T], n int) iter.Seq[T] {
    return func(yield func(T) bool) {
        count := 0
        for v := range seq {
            if count >= n {
                return
            }
            if !yield(v) {
                return
            }
            count++
        }
    }
}
```

Usage:

```go
func main() {
    // Get first 5 even Fibonacci numbers greater than 0.
    evens := itertools.Filter(Fibonacci(10000), func(n int) bool { return n%2 == 0 && n > 0 })
    firstFive := itertools.Take(evens, 5)

    for v := range firstFive {
        fmt.Println(v) // 2, 8, 34, 144, 610
    }
}
```

---

### Command

The Command pattern encapsulates a request as an object, allowing you to parameterize
clients with different requests, queue operations, and support undo.

```go
package editor

import "fmt"

// Command is the interface for all editor commands.
type Command interface {
    Execute() error
    Undo() error
    String() string
}

// Document is the receiver.
type Document struct {
    Content string
}

// --- Concrete Commands ---

// InsertCommand inserts text at a position.
type InsertCommand struct {
    doc      *Document
    position int
    text     string
}

func NewInsertCommand(doc *Document, pos int, text string) *InsertCommand {
    return &InsertCommand{doc: doc, position: pos, text: text}
}

func (c *InsertCommand) Execute() error {
    if c.position < 0 || c.position > len(c.doc.Content) {
        return fmt.Errorf("position %d out of range [0, %d]", c.position, len(c.doc.Content))
    }
    c.doc.Content = c.doc.Content[:c.position] + c.text + c.doc.Content[c.position:]
    return nil
}

func (c *InsertCommand) Undo() error {
    start := c.position
    end := c.position + len(c.text)
    c.doc.Content = c.doc.Content[:start] + c.doc.Content[end:]
    return nil
}

func (c *InsertCommand) String() string {
    return fmt.Sprintf("Insert(%d, %q)", c.position, c.text)
}

// DeleteCommand deletes a range of text.
type DeleteCommand struct {
    doc     *Document
    start   int
    end     int
    deleted string // saved for undo
}

func NewDeleteCommand(doc *Document, start, end int) *DeleteCommand {
    return &DeleteCommand{doc: doc, start: start, end: end}
}

func (c *DeleteCommand) Execute() error {
    if c.start < 0 || c.end > len(c.doc.Content) || c.start > c.end {
        return fmt.Errorf("invalid range [%d, %d) for content length %d", c.start, c.end, len(c.doc.Content))
    }
    c.deleted = c.doc.Content[c.start:c.end]
    c.doc.Content = c.doc.Content[:c.start] + c.doc.Content[c.end:]
    return nil
}

func (c *DeleteCommand) Undo() error {
    c.doc.Content = c.doc.Content[:c.start] + c.deleted + c.doc.Content[c.start:]
    return nil
}

func (c *DeleteCommand) String() string {
    return fmt.Sprintf("Delete(%d, %d)", c.start, c.end)
}

// --- Invoker ---

// History manages command execution and undo/redo.
type History struct {
    undoStack []Command
    redoStack []Command
}

func (h *History) Execute(cmd Command) error {
    if err := cmd.Execute(); err != nil {
        return err
    }
    h.undoStack = append(h.undoStack, cmd)
    h.redoStack = nil // clear redo stack on new command
    return nil
}

func (h *History) Undo() error {
    if len(h.undoStack) == 0 {
        return fmt.Errorf("nothing to undo")
    }
    cmd := h.undoStack[len(h.undoStack)-1]
    h.undoStack = h.undoStack[:len(h.undoStack)-1]
    if err := cmd.Undo(); err != nil {
        return err
    }
    h.redoStack = append(h.redoStack, cmd)
    return nil
}

func (h *History) Redo() error {
    if len(h.redoStack) == 0 {
        return fmt.Errorf("nothing to redo")
    }
    cmd := h.redoStack[len(h.redoStack)-1]
    h.redoStack = h.redoStack[:len(h.redoStack)-1]
    if err := cmd.Execute(); err != nil {
        return err
    }
    h.undoStack = append(h.undoStack, cmd)
    return nil
}
```

Usage:

```go
func main() {
    doc := &Document{}
    history := &History{}

    history.Execute(NewInsertCommand(doc, 0, "Hello, "))    // "Hello, "
    history.Execute(NewInsertCommand(doc, 7, "World!"))      // "Hello, World!"
    history.Execute(NewDeleteCommand(doc, 5, 7))             // "HelloWorld!"
    fmt.Println(doc.Content) // "HelloWorld!"

    history.Undo()           // undo delete -> "Hello, World!"
    fmt.Println(doc.Content) // "Hello, World!"

    history.Undo()           // undo second insert -> "Hello, "
    fmt.Println(doc.Content) // "Hello, "

    history.Redo()           // redo second insert -> "Hello, World!"
    fmt.Println(doc.Content) // "Hello, World!"
}
```

---

### Pipeline Pattern (Channel Pipelines)

A pipeline is a series of stages connected by channels, where each stage is a
goroutine that receives values from upstream, performs some function, and sends values
downstream. This is one of Go's signature patterns.

```go
package pipeline

import (
    "context"
    "math"
)

// Generator creates a channel that emits values from a slice.
func Generator[T any](ctx context.Context, values ...T) <-chan T {
    out := make(chan T)
    go func() {
        defer close(out)
        for _, v := range values {
            select {
            case out <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Map applies fn to each value from in and sends results to out.
func Map[T, U any](ctx context.Context, in <-chan T, fn func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- fn(v):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Filter passes through only values where pred returns true.
func Filter[T any](ctx context.Context, in <-chan T, pred func(T) bool) <-chan T {
    out := make(chan T)
    go func() {
        defer close(out)
        for v := range in {
            if pred(v) {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

// Reduce consumes all values and returns a single result.
func Reduce[T, U any](ctx context.Context, in <-chan T, initial U, fn func(U, T) U) U {
    acc := initial
    for v := range in {
        select {
        case <-ctx.Done():
            return acc
        default:
            acc = fn(acc, v)
        }
    }
    return acc
}
```

Usage -- compute sum of squares of even numbers:

```go
func main() {
    ctx := context.Background()

    // Pipeline: generate -> filter evens -> square -> sum.
    numbers := pipeline.Generator(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    evens := pipeline.Filter(ctx, numbers, func(n int) bool {
        return n%2 == 0
    })

    squares := pipeline.Map(ctx, evens, func(n int) int {
        return n * n
    })

    sum := pipeline.Reduce(ctx, squares, 0, func(acc, n int) int {
        return acc + n
    })

    fmt.Println("Sum of squares of evens:", sum) // 4 + 16 + 36 + 64 + 100 = 220
}
```

#### Real-World: Log Processing Pipeline

```go
package main

import (
    "bufio"
    "context"
    "fmt"
    "os"
    "strings"
    "time"
)

type LogEntry struct {
    Timestamp time.Time
    Level     string
    Message   string
    Raw       string
}

func readLines(ctx context.Context, filename string) (<-chan string, <-chan error) {
    lines := make(chan string)
    errc := make(chan error, 1)
    go func() {
        defer close(lines)
        defer close(errc)

        f, err := os.Open(filename)
        if err != nil {
            errc <- err
            return
        }
        defer f.Close()

        scanner := bufio.NewScanner(f)
        for scanner.Scan() {
            select {
            case lines <- scanner.Text():
            case <-ctx.Done():
                errc <- ctx.Err()
                return
            }
        }
        errc <- scanner.Err()
    }()
    return lines, errc
}

func parseLines(ctx context.Context, lines <-chan string) <-chan LogEntry {
    entries := make(chan LogEntry)
    go func() {
        defer close(entries)
        for line := range lines {
            // Simplified parsing: "2024-01-15T10:30:00Z ERROR something failed"
            parts := strings.SplitN(line, " ", 3)
            if len(parts) < 3 {
                continue
            }
            ts, err := time.Parse(time.RFC3339, parts[0])
            if err != nil {
                continue
            }
            entry := LogEntry{
                Timestamp: ts,
                Level:     parts[1],
                Message:   parts[2],
                Raw:       line,
            }
            select {
            case entries <- entry:
            case <-ctx.Done():
                return
            }
        }
    }()
    return entries
}

func filterErrors(ctx context.Context, entries <-chan LogEntry) <-chan LogEntry {
    errors := make(chan LogEntry)
    go func() {
        defer close(errors)
        for entry := range entries {
            if entry.Level == "ERROR" || entry.Level == "FATAL" {
                select {
                case errors <- entry:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return errors
}
```

---

### Fan-Out / Fan-In

Fan-out distributes work across multiple goroutines. Fan-in merges results from
multiple channels into one.

```go
package fanout

import (
    "context"
    "crypto/sha256"
    "fmt"
    "sync"
)

// HashResult pairs an input with its hash.
type HashResult struct {
    Input string
    Hash  string
}

// FanOut distributes work from a single input channel to n workers.
// Each worker applies fn to each item it receives.
func FanOut[T, U any](
    ctx context.Context,
    in <-chan T,
    n int,
    fn func(T) U,
) []<-chan U {
    channels := make([]<-chan U, n)
    for i := 0; i < n; i++ {
        channels[i] = worker(ctx, in, fn)
    }
    return channels
}

func worker[T, U any](ctx context.Context, in <-chan T, fn func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- fn(v):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// FanIn merges multiple channels into a single channel.
func FanIn[T any](ctx context.Context, channels ...<-chan T) <-chan T {
    merged := make(chan T)
    var wg sync.WaitGroup

    wg.Add(len(channels))
    for _, ch := range channels {
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                select {
                case merged <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

Usage:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Generate work items.
    input := make(chan string, 100)
    go func() {
        defer close(input)
        for i := 0; i < 1000; i++ {
            input <- fmt.Sprintf("item-%d", i)
        }
    }()

    // Fan out to 5 workers that compute SHA-256 hashes.
    hashFn := func(s string) HashResult {
        h := sha256.Sum256([]byte(s))
        return HashResult{Input: s, Hash: fmt.Sprintf("%x", h[:8])}
    }
    workerChans := fanout.FanOut(ctx, input, 5, hashFn)

    // Fan in results.
    results := fanout.FanIn(ctx, workerChans...)

    // Consume results.
    count := 0
    for result := range results {
        count++
        if count <= 3 {
            fmt.Printf("%s -> %s\n", result.Input, result.Hash)
        }
    }
    fmt.Printf("Processed %d items\n", count)
}
```

#### Bounded Fan-Out with Semaphore

Sometimes you want to fan out but limit concurrency (e.g., making HTTP calls):

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "sync"
)

func fetchURLs(ctx context.Context, urls []string, maxConcurrency int) map[string]int {
    results := make(map[string]int)
    var mu sync.Mutex

    sem := make(chan struct{}, maxConcurrency)
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()

            // Acquire semaphore slot.
            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
            case <-ctx.Done():
                return
            }

            req, err := http.NewRequestWithContext(ctx, "GET", u, nil)
            if err != nil {
                return
            }
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return
            }
            resp.Body.Close()

            mu.Lock()
            results[u] = resp.StatusCode
            mu.Unlock()
        }(url)
    }

    wg.Wait()
    return results
}
```

---

### Pub/Sub

Pub/Sub (Publish-Subscribe) is a messaging pattern where senders (publishers) do not
send messages directly to receivers (subscribers). Instead, messages are published to
topics, and subscribers receive messages for topics they have subscribed to.

This is an extension of the Observer pattern with topic-based routing.

```go
package pubsub

import (
    "context"
    "sync"
)

// Message is a message published to a topic.
type Message struct {
    Topic   string
    Payload any
}

// Subscription represents a subscriber's connection to a topic.
type Subscription struct {
    ch     chan Message
    topic  string
    cancel context.CancelFunc
}

// C returns the channel to receive messages on.
func (s *Subscription) C() <-chan Message { return s.ch }

// Broker manages topics and subscriptions.
type Broker struct {
    mu     sync.RWMutex
    subs   map[string][]*Subscription
    closed bool
}

// NewBroker creates a new Broker.
func NewBroker() *Broker {
    return &Broker{
        subs: make(map[string][]*Subscription),
    }
}

// Subscribe creates a subscription to the given topic.
func (b *Broker) Subscribe(ctx context.Context, topic string, bufSize int) *Subscription {
    ctx, cancel := context.WithCancel(ctx)
    sub := &Subscription{
        ch:     make(chan Message, bufSize),
        topic:  topic,
        cancel: cancel,
    }

    // Automatically unsubscribe when context is cancelled.
    go func() {
        <-ctx.Done()
        b.unsubscribe(sub)
    }()

    b.mu.Lock()
    b.subs[topic] = append(b.subs[topic], sub)
    b.mu.Unlock()

    return sub
}

// Publish sends a message to all subscribers of the given topic.
func (b *Broker) Publish(topic string, payload any) {
    msg := Message{Topic: topic, Payload: payload}

    b.mu.RLock()
    subs := make([]*Subscription, len(b.subs[topic]))
    copy(subs, b.subs[topic])
    b.mu.RUnlock()

    for _, sub := range subs {
        select {
        case sub.ch <- msg:
        default:
            // Slow subscriber; drop message.
        }
    }
}

// unsubscribe removes a subscription and closes its channel.
func (b *Broker) unsubscribe(sub *Subscription) {
    b.mu.Lock()
    defer b.mu.Unlock()

    subs := b.subs[sub.topic]
    for i, s := range subs {
        if s == sub {
            b.subs[sub.topic] = append(subs[:i], subs[i+1:]...)
            close(sub.ch)
            return
        }
    }
}

// Close shuts down all subscriptions.
func (b *Broker) Close() {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.closed {
        return
    }
    b.closed = true
    for topic, subs := range b.subs {
        for _, sub := range subs {
            sub.cancel()
            close(sub.ch)
        }
        delete(b.subs, topic)
    }
}
```

Usage:

```go
func main() {
    broker := pubsub.NewBroker()
    defer broker.Close()

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Subscribe to different topics.
    orderSub := broker.Subscribe(ctx, "orders", 10)
    paymentSub := broker.Subscribe(ctx, "payments", 10)
    allOrderSub := broker.Subscribe(ctx, "orders", 10) // second subscriber

    // Consumers.
    go func() {
        for msg := range orderSub.C() {
            fmt.Printf("[Order Service] %v\n", msg.Payload)
        }
    }()

    go func() {
        for msg := range paymentSub.C() {
            fmt.Printf("[Payment Service] %v\n", msg.Payload)
        }
    }()

    go func() {
        for msg := range allOrderSub.C() {
            fmt.Printf("[Analytics] Order event: %v\n", msg.Payload)
        }
    }()

    // Publish events.
    broker.Publish("orders", map[string]any{"id": 1, "item": "laptop"})
    broker.Publish("payments", map[string]any{"id": 1, "amount": 999.99})
    broker.Publish("orders", map[string]any{"id": 2, "item": "keyboard"})

    time.Sleep(100 * time.Millisecond)
}
```

---

## Concurrency Patterns

These patterns are unique to Go (or at least uniquely natural in Go) because of
goroutines and channels.

---

### Worker Pool

A worker pool limits the number of goroutines processing tasks concurrently. It is
one of the most common and practical patterns in Go.

```go
package workerpool

import (
    "context"
    "fmt"
    "sync"
)

// Task represents a unit of work.
type Task[T any] struct {
    ID      int
    Payload T
}

// Result represents the output of processing a Task.
type Result[T, U any] struct {
    TaskID int
    Input  T
    Output U
    Err    error
}

// Pool processes tasks with a fixed number of workers.
type Pool[T, U any] struct {
    numWorkers int
    tasks      chan Task[T]
    results    chan Result[T, U]
    handler    func(T) (U, error)
    wg         sync.WaitGroup
}

// New creates a worker pool.
func New[T, U any](numWorkers, taskBufSize int, handler func(T) (U, error)) *Pool[T, U] {
    return &Pool[T, U]{
        numWorkers: numWorkers,
        tasks:      make(chan Task[T], taskBufSize),
        results:    make(chan Result[T, U], taskBufSize),
        handler:    handler,
    }
}

// Start launches the worker goroutines.
func (p *Pool[T, U]) Start(ctx context.Context) {
    for i := 0; i < p.numWorkers; i++ {
        p.wg.Add(1)
        go func(workerID int) {
            defer p.wg.Done()
            for task := range p.tasks {
                select {
                case <-ctx.Done():
                    return
                default:
                }

                output, err := p.handler(task.Payload)
                result := Result[T, U]{
                    TaskID: task.ID,
                    Input:  task.Payload,
                    Output: output,
                    Err:    err,
                }

                select {
                case p.results <- result:
                case <-ctx.Done():
                    return
                }
            }
        }(i)
    }

    // Close results channel when all workers are done.
    go func() {
        p.wg.Wait()
        close(p.results)
    }()
}

// Submit adds a task to the pool. Blocks if the task buffer is full.
func (p *Pool[T, U]) Submit(task Task[T]) {
    p.tasks <- task
}

// Done signals that no more tasks will be submitted.
func (p *Pool[T, U]) Done() {
    close(p.tasks)
}

// Results returns the results channel.
func (p *Pool[T, U]) Results() <-chan Result[T, U] {
    return p.results
}
```

Usage:

```go
func main() {
    ctx := context.Background()

    // Create a pool that processes URLs.
    pool := workerpool.New[string, int](10, 100, func(url string) (int, error) {
        resp, err := http.Get(url)
        if err != nil {
            return 0, err
        }
        defer resp.Body.Close()
        return resp.StatusCode, nil
    })

    pool.Start(ctx)

    // Submit tasks.
    urls := []string{
        "https://example.com",
        "https://golang.org",
        "https://httpbin.org/status/200",
        // ... more URLs
    }
    go func() {
        for i, url := range urls {
            pool.Submit(workerpool.Task[string]{ID: i, Payload: url})
        }
        pool.Done()
    }()

    // Collect results.
    for result := range pool.Results() {
        if result.Err != nil {
            fmt.Printf("Task %d (%s): ERROR: %v\n", result.TaskID, result.Input, result.Err)
        } else {
            fmt.Printf("Task %d (%s): status %d\n", result.TaskID, result.Input, result.Output)
        }
    }
}
```

#### Simple Worker Pool (When Generics Are Overkill)

```go
func processItems(ctx context.Context, items []string, concurrency int) error {
    sem := make(chan struct{}, concurrency)
    errCh := make(chan error, len(items))
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(it string) {
            defer wg.Done()

            // Acquire semaphore.
            sem <- struct{}{}
            defer func() { <-sem }()

            if err := processItem(ctx, it); err != nil {
                errCh <- fmt.Errorf("processing %s: %w", it, err)
            }
        }(item)
    }

    wg.Wait()
    close(errCh)

    // Collect errors.
    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }
    return errors.Join(errs...)
}
```

---

### Semaphore

A semaphore limits concurrent access to a resource. In Go, a buffered channel is the
simplest semaphore. For more sophisticated needs, use `golang.org/x/sync/semaphore`.

#### Channel-Based Semaphore

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Semaphore limits concurrency using a buffered channel.
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(maxConcurrency int) *Semaphore {
    return &Semaphore{ch: make(chan struct{}, maxConcurrency)}
}

func (s *Semaphore) Acquire() {
    s.ch <- struct{}{}
}

func (s *Semaphore) Release() {
    <-s.ch
}

func main() {
    sem := NewSemaphore(3) // max 3 concurrent operations
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            sem.Acquire()
            defer sem.Release()

            fmt.Printf("Worker %d: starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d: done\n", id)
        }(i)
    }

    wg.Wait()
}
```

#### Weighted Semaphore (`golang.org/x/sync/semaphore`)

The `x/sync/semaphore` package provides a weighted semaphore with context support:

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/semaphore"
)

func main() {
    // Allow up to 3 concurrent operations, but some cost more than others.
    sem := semaphore.NewWeighted(3)
    ctx := context.Background()
    var wg sync.WaitGroup

    tasks := []struct {
        name   string
        weight int64
    }{
        {"small-1", 1},
        {"small-2", 1},
        {"large-1", 2}, // consumes 2 slots
        {"small-3", 1},
        {"large-2", 3}, // consumes all 3 slots (exclusive)
        {"small-4", 1},
    }

    for _, task := range tasks {
        wg.Add(1)
        go func(name string, weight int64) {
            defer wg.Done()

            // Acquire weighted slots. Blocks until enough capacity is available.
            if err := sem.Acquire(ctx, weight); err != nil {
                fmt.Printf("%s: failed to acquire: %v\n", name, err)
                return
            }
            defer sem.Release(weight)

            fmt.Printf("%s (weight %d): running\n", name, weight)
            time.Sleep(500 * time.Millisecond)
            fmt.Printf("%s: done\n", name)
        }(task.name, task.weight)
    }

    wg.Wait()
}
```

---

### Error Group

`golang.org/x/sync/errgroup` manages a group of goroutines working on subtasks of a
common task. It collects the first error and can cancel remaining work.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "time"

    "golang.org/x/sync/errgroup"
)

// FetchResult holds the response for a URL.
type FetchResult struct {
    URL        string
    StatusCode int
    Size       int64
}

// FetchAll fetches multiple URLs concurrently, returning results or the first error.
func FetchAll(ctx context.Context, urls []string) ([]FetchResult, error) {
    g, ctx := errgroup.WithContext(ctx)

    // Limit concurrency to 5.
    g.SetLimit(5)

    results := make([]FetchResult, len(urls))

    for i, url := range urls {
        i, url := i, url // capture loop variables
        g.Go(func() error {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return fmt.Errorf("creating request for %s: %w", url, err)
            }

            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            defer resp.Body.Close()

            // Read body to get size.
            size, err := io.Copy(io.Discard, resp.Body)
            if err != nil {
                return fmt.Errorf("reading body of %s: %w", url, err)
            }

            results[i] = FetchResult{
                URL:        url,
                StatusCode: resp.StatusCode,
                Size:       size,
            }
            return nil
        })
    }

    // Wait for all goroutines. Returns first non-nil error.
    if err := g.Wait(); err != nil {
        return nil, err
    }

    return results, nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    urls := []string{
        "https://example.com",
        "https://golang.org",
        "https://httpbin.org/get",
    }

    results, err := FetchAll(ctx, urls)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }

    for _, r := range results {
        fmt.Printf("%s: %d (%d bytes)\n", r.URL, r.StatusCode, r.Size)
    }
}
```

#### Errgroup with Multiple Stages

```go
func ProcessPipeline(ctx context.Context, inputFiles []string) error {
    g, ctx := errgroup.WithContext(ctx)

    // Stage 1: Read files.
    fileCh := make(chan FileData, len(inputFiles))
    for _, f := range inputFiles {
        f := f
        g.Go(func() error {
            data, err := readFile(ctx, f)
            if err != nil {
                return err
            }
            select {
            case fileCh <- data:
                return nil
            case <-ctx.Done():
                return ctx.Err()
            }
        })
    }

    // Close fileCh when all readers are done.
    go func() {
        g.Wait() // This is safe -- other goroutines also call g.Go
        close(fileCh)
    }()

    // Stage 2: Process data.
    resultCh := make(chan ProcessedData, len(inputFiles))
    g.Go(func() error {
        defer close(resultCh)
        for data := range fileCh {
            result, err := transform(ctx, data)
            if err != nil {
                return err
            }
            select {
            case resultCh <- result:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    // Stage 3: Write results.
    g.Go(func() error {
        for result := range resultCh {
            if err := writeResult(ctx, result); err != nil {
                return err
            }
        }
        return nil
    })

    return g.Wait()
}
```

---

### Context-Based Cancellation Tree

`context.Context` enables hierarchical cancellation: when a parent context is
cancelled, all derived child contexts are cancelled too. This forms a cancellation
tree.

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// Service represents a microservice component.
type Service struct {
    name string
}

func (s *Service) Run(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            fmt.Printf("[%s] shutting down: %v\n", s.name, ctx.Err())
            return
        case <-ticker.C:
            fmt.Printf("[%s] working...\n", s.name)
        }
    }
}

func main() {
    // Root context with a 5-second deadline.
    rootCtx, rootCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer rootCancel()

    var wg sync.WaitGroup

    // --- Level 1: API server ---
    apiCtx, apiCancel := context.WithCancel(rootCtx)
    defer apiCancel()
    api := &Service{name: "API"}
    wg.Add(1)
    go api.Run(apiCtx, &wg)

    // --- Level 2: Under API, we have a database and a cache ---
    dbCtx, dbCancel := context.WithCancel(apiCtx)
    defer dbCancel()
    db := &Service{name: "Database"}
    wg.Add(1)
    go db.Run(dbCtx, &wg)

    cacheCtx, cacheCancel := context.WithCancel(apiCtx)
    defer cacheCancel()
    cache := &Service{name: "Cache"}
    wg.Add(1)
    go cache.Run(cacheCtx, &wg)

    // --- Level 2: Independent background worker ---
    workerCtx, workerCancel := context.WithCancel(rootCtx)
    defer workerCancel()
    worker := &Service{name: "Worker"}
    wg.Add(1)
    go worker.Run(workerCtx, &wg)

    // Simulate: after 2 seconds, API sub-tree is cancelled (DB + Cache go down too).
    time.AfterFunc(2*time.Second, func() {
        fmt.Println("\n--- Cancelling API sub-tree ---")
        apiCancel()
    })

    // Worker keeps running until the root context times out at 5 seconds.
    wg.Wait()
    fmt.Println("All services stopped.")
}
```

#### Cancellation Tree Diagram

```
rootCtx (timeout: 5s)
├── apiCtx (cancelled at 2s)
│   ├── dbCtx  (cancelled when apiCtx cancels)
│   └── cacheCtx (cancelled when apiCtx cancels)
└── workerCtx (runs until rootCtx timeout at 5s)
```

#### Context Values for Request-Scoped Data

```go
package requestid

import "context"

type contextKey string

const requestIDKey contextKey = "request_id"

// WithRequestID adds a request ID to the context.
func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

// FromContext extracts the request ID from the context.
func FromContext(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```

> **Warning:** Context values are **not** a substitute for function parameters. Use
> them only for request-scoped data that crosses API boundaries (request IDs, auth
> tokens, trace IDs) -- never for optional function parameters.

#### Graceful Shutdown with Context

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Create root context that cancels on SIGINT/SIGTERM.
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello")
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // Start server in background.
    go func() {
        fmt.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            fmt.Fprintf(os.Stderr, "Server error: %v\n", err)
            os.Exit(1)
        }
    }()

    // Wait for cancellation signal.
    <-ctx.Done()
    fmt.Println("\nReceived shutdown signal")

    // Give in-flight requests 10 seconds to complete.
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(shutdownCtx); err != nil {
        fmt.Fprintf(os.Stderr, "Shutdown error: %v\n", err)
    }
    fmt.Println("Server stopped gracefully")
}
```

---

## Anti-Patterns to Avoid in Go

### 1. The "Java in Go" Anti-Pattern

Do not transliterate Java design patterns into Go. The result is verbose, unidiomatic,
and confusing to Go developers.

```go
// BAD: Java-style AbstractFactory in Go.
type AbstractAnimalFactory interface {
    CreateAnimal() Animal
}

type DogFactory struct{}
func (f DogFactory) CreateAnimal() Animal { return &Dog{} }

type CatFactory struct{}
func (f CatFactory) CreateAnimal() Animal { return &Cat{} }

// GOOD: Just use a factory function.
func NewAnimal(species string) (Animal, error) {
    switch species {
    case "dog":
        return &Dog{}, nil
    case "cat":
        return &Cat{}, nil
    default:
        return nil, fmt.Errorf("unknown species: %s", species)
    }
}
```

### 2. Interface Pollution

Defining interfaces before you need them, or with too many methods.

```go
// BAD: Premature, bloated interface.
type UserService interface {
    Create(u User) error
    Get(id string) (User, error)
    Update(u User) error
    Delete(id string) error
    List(filter Filter) ([]User, error)
    Count() (int, error)
    Search(query string) ([]User, error)
    Export(format string) ([]byte, error)
    Import(data []byte) error
    Validate(u User) error
    // ... 20 more methods
}

// GOOD: Small, focused interfaces defined by the consumer.
// The consumer defines the interface it needs.
type UserGetter interface {
    Get(id string) (User, error)
}

type UserCreator interface {
    Create(u User) error
}

// Your handler only depends on what it actually uses.
func GetUserHandler(repo UserGetter) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        user, err := repo.Get(id)
        // ...
    }
}
```

> **Tip:** Go proverb: "The bigger the interface, the weaker the abstraction."
> Interfaces should be defined by the consumer, not the provider.

### 3. Goroutine Leak

Starting goroutines without ensuring they can be stopped.

```go
// BAD: This goroutine leaks if nobody reads from ch.
func produce() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i // blocks forever if nobody reads
        }
    }()
    return ch
}

// GOOD: Accept a context for cancellation.
func produce(ctx context.Context) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; ; i++ {
            select {
            case ch <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return ch
}
```

### 4. Naked Goroutine (No Error Handling)

```go
// BAD: If processItem panics, the whole program crashes with no recovery.
for _, item := range items {
    go processItem(item)
}

// GOOD: Use errgroup or at least recover from panics.
g, ctx := errgroup.WithContext(ctx)
for _, item := range items {
    item := item
    g.Go(func() error {
        return processItem(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    log.Printf("processing failed: %v", err)
}
```

### 5. Channel Overuse

Not everything needs a channel. Sometimes a mutex is simpler and faster.

```go
// BAD: Using a channel as a mutex for a simple counter.
type Counter struct {
    ch chan int
}

func NewCounter() *Counter {
    c := &Counter{ch: make(chan int, 1)}
    c.ch <- 0
    return c
}

func (c *Counter) Increment() {
    val := <-c.ch
    c.ch <- val + 1
}

// GOOD: Just use sync/atomic or a mutex.
type Counter struct {
    val atomic.Int64
}

func (c *Counter) Increment() {
    c.val.Add(1)
}
```

### 6. Empty Interface Abuse

```go
// BAD: Losing all type safety.
func Process(data any) any {
    // What is data? What do we return? Nobody knows.
    return nil
}

// GOOD: Use concrete types or generics.
func Process[T Processable](data T) (Result, error) {
    // Clear contract, compile-time safety.
}
```

### 7. Init() Abuse

```go
// BAD: Complex logic in init() is hard to test and has hidden side effects.
func init() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err) // crashes the program on import!
    }
    globalDB = db
}

// GOOD: Explicit initialization called from main().
func main() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    app := NewApp(db)
    app.Run()
}
```

### 8. Stuttering Names

```go
// BAD: user.UserService, user.UserRepository.
package user

type UserService struct{} // user.UserService -- stutters!
type UserRepository struct{}

// GOOD: Drop the package prefix from type names.
package user

type Service struct{}    // user.Service -- clean.
type Repository struct{} // user.Repository -- clean.
```

### 9. Not Closing Resources in Defer

```go
// BAD: Resource leak if processing panics.
func process(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return err
    }
    // ... lots of code ...
    result, err := doWork(f)
    if err != nil {
        return err // forgot to close f!
    }
    f.Close()
    return nil
}

// GOOD: Defer close immediately after successful open.
func process(name string) error {
    f, err := os.Open(name)
    if err != nil {
        return err
    }
    defer f.Close()

    result, err := doWork(f)
    if err != nil {
        return err
    }
    return nil
}
```

### 10. Overengineering with Patterns

```go
// BAD: AbstractSingletonProxyFactoryBean pattern for a simple task.
type NotifierFactory interface {
    CreateNotifier(config NotifierConfig) (Notifier, error)
}
type NotifierConfig struct { /* ... */ }
type NotifierFactoryImpl struct { /* ... */ }
// ... 200 lines of indirection for sending an email.

// GOOD: Just write the code.
func SendEmail(to, subject, body string) error {
    // Send the email.
    return smtp.SendMail(addr, auth, from, []string{to}, msg)
}
```

---

## When Patterns Help vs When They Hurt

### Decision Framework

Before applying a pattern, ask these questions:

| Question | If Yes | If No |
|---|---|---|
| Does this solve a problem I have *right now*? | Consider the pattern | Do not apply it |
| Would another Go developer understand this on first read? | Pattern is likely appropriate | Simplify |
| Does Go have a simpler built-in idiom? | Use the idiom instead | Consider the pattern |
| Am I adding indirection to handle future changes? | Only if changes are *very likely* | YAGNI -- skip it |
| Does this pattern reduce or increase the line count? | Either is fine if clarity improves | Reconsider |

### Patterns That Almost Always Help in Go

| Pattern | Why It Helps |
|---|---|
| **Functional Options** | Clean API for configurable constructors |
| **Middleware / Decorator** | Standard way to layer cross-cutting concerns in HTTP |
| **Worker Pool** | Controlled concurrency for I/O-bound work |
| **Factory Functions (`NewXxx`)** | Idiomatic construction with validation |
| **Pipeline** | Composable, concurrent data processing |
| **Context Cancellation** | Graceful shutdown and timeout propagation |

### Patterns That Are Situational

| Pattern | When It Helps | When It Hurts |
|---|---|---|
| **Singleton** | Truly global resources (DB connection) | Hides dependencies, harms testability |
| **Builder** | Complex multi-step construction | Simple structs (use literal or options) |
| **Observer/Pub-Sub** | Decoupled event-driven systems | Small apps with 2-3 components |
| **Strategy (interface)** | Multiple algorithms with state | Single algorithm (just write the code) |
| **Command** | Undo/redo, operation queuing | Simple CRUD operations |

### Patterns That Are Rarely Needed in Go

| Pattern | Why It Is Rare in Go |
|---|---|
| **Abstract Factory** | Factory functions + interfaces cover this |
| **Template Method** | Use an interface + a function instead |
| **Prototype** | Structs are value types; just copy them |
| **Chain of Responsibility** | Middleware pattern covers most cases |
| **Memento** | Rarely needed; serialization is simpler |
| **Visitor** | Type switches or interfaces are simpler |

### The Golden Rule

> **Write Go code that a junior developer can read and understand in 30 seconds.**
> If a pattern makes the code harder to follow without a clear, present benefit,
> you are overengineering. Go is a language of simplicity -- respect that.

### Summary Table: Pattern Quick Reference

| Category | Pattern | Go Mechanism | Complexity |
|---|---|---|---|
| Creational | Functional Options | `func(*T)` variadic | Low |
| Creational | Builder | Method chaining | Medium |
| Creational | Factory | `NewXxx()` functions | Low |
| Creational | Singleton | `sync.Once` | Low |
| Creational | Object Pool | `sync.Pool` | Low |
| Structural | Adapter | Implicit interfaces | Low |
| Structural | Decorator/Middleware | `func(Handler) Handler` | Low |
| Structural | Composite | Interface + slice of interface | Medium |
| Structural | Facade | Struct wrapping subsystems | Low |
| Structural | Proxy | Interface wrapping | Medium |
| Behavioral | Strategy | `func` type or interface | Low |
| Behavioral | Observer | Channels | Medium |
| Behavioral | Iterator | `range`, `iter.Seq` (1.23+) | Low |
| Behavioral | Command | Interface with Execute/Undo | Medium |
| Behavioral | Pipeline | Chained channels | Medium |
| Behavioral | Fan-Out/Fan-In | Goroutines + merge | Medium |
| Behavioral | Pub/Sub | Broker + channels | High |
| Concurrency | Worker Pool | Goroutines + task channel | Medium |
| Concurrency | Semaphore | Buffered channel / `x/sync` | Low |
| Concurrency | Error Group | `x/sync/errgroup` | Low |
| Concurrency | Cancellation Tree | `context.Context` | Low |

---

## Key Takeaways

1. **Go is not Java.** Do not force class-based patterns into a language without
   classes. Embrace composition, interfaces, and first-class functions.

2. **Interfaces should be small.** One or two methods. Defined by the consumer, not
   the provider.

3. **Channels are for communication, not synchronization.** Use `sync.Mutex` and
   `sync.WaitGroup` when channels add unnecessary complexity.

4. **Every goroutine must have a clear shutdown path.** Use `context.Context` to
   propagate cancellation.

5. **Functional Options is the most "Go" pattern** -- learn it, use it, love it.

6. **Middleware is the decorator pattern done right** -- it is the standard approach
   for HTTP servers and works beautifully with Go's `http.Handler` interface.

7. **Worker pools and pipelines** are your bread and butter for concurrent programs.
   Combine them with `errgroup` for clean error handling.

8. **The best pattern is no pattern.** Write the simplest code that solves the
   problem. Add abstraction only when you have evidence it is needed.

---

*Next chapter: [53 - Clean Architecture in Go](./53-clean-architecture-in-go.md)*
