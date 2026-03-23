# Chapter 16: The Context Package & Request Scoping

## Table of Contents

1. [What is Context?](#1-what-is-context)
2. [context.Background() and context.TODO()](#2-contextbackground-and-contexttodo)
3. [Cancellation](#3-cancellation)
4. [Timeouts and Deadlines](#4-timeouts-and-deadlines)
5. [Context Values](#5-context-values)
6. [Context in HTTP Servers](#6-context-in-http-servers)
7. [Context in Database Operations](#7-context-in-database-operations)
8. [Context Best Practices](#8-context-best-practices)
9. [Context Propagation](#9-context-propagation)
10. [Common Mistakes](#10-common-mistakes)
11. [Go vs Node.js Comparison](#11-go-vs-nodejs-comparison)
12. [Key Takeaways](#12-key-takeaways)
13. [Practice Exercises](#13-practice-exercises)

---

## 1. What is Context?

### The Problem

Imagine an HTTP server handling a request. The handler calls a service layer, which calls a database, which calls an external API. Now the client disconnects. What happens?

Without context, **nothing**. Every goroutine in that chain keeps running. The database query continues executing. The external API call waits for its full timeout. You are burning CPU, memory, and network connections for a response nobody will ever read.

Now multiply this by thousands of concurrent requests. You have a resource leak that will bring your server to its knees.

This is the problem `context` solves.

### The context.Context Interface

```go
type Context interface {
    // Deadline returns the time when this context will be cancelled.
    // ok is false if no deadline is set.
    Deadline() (deadline time.Time, ok bool)

    // Done returns a channel that is closed when this context is cancelled.
    // Returns nil if the context can never be cancelled.
    Done() <-chan struct{}

    // Err returns nil if Done is not yet closed.
    // After Done is closed, Err returns why:
    //   - context.Canceled if the context was cancelled
    //   - context.DeadlineExceeded if the deadline passed
    Err() error

    // Value returns the value associated with this context for key,
    // or nil if no value is associated with key.
    Value(key any) any
}
```

The interface is deliberately small. Four methods. That is it. But these four methods give you three critical capabilities:

1. **Cancellation propagation** -- Tell an entire tree of goroutines "stop what you are doing"
2. **Deadline enforcement** -- Automatically cancel work that takes too long
3. **Request-scoped values** -- Carry data across API boundaries without threading it through every function signature

### WHY Context Exists

Before Go 1.7, every team at Google invented their own mechanism for cancellation. Some used channels, some used custom types, some used global variables. There was no standard way to say "this operation should be cancelled" that worked across package boundaries.

The `context` package was extracted from Google's internal codebase and added to the standard library to solve this coordination problem. It provides a **single, standard interface** that the entire Go ecosystem can agree on.

### The Mental Model

Think of context as a **tree**. You start with a root context and derive children from it. Each child can be cancelled independently, and cancelling a parent automatically cancels all of its children. But cancelling a child never affects the parent.

```
                    Background (root)
                         |
                  WithCancel (server)
                   /            \
        WithTimeout            WithTimeout
        (request A)            (request B)
          /    \                    |
   WithValue  WithCancel      WithDeadline
   (user ID)  (db query)      (external API)
```

When request A's timeout expires, both its children (the value context and the db query context) are cancelled. But request B is completely unaffected.

---

## 2. context.Background() and context.TODO()

### context.Background()

```go
ctx := context.Background()
```

`Background()` returns a non-nil, empty context. It is never cancelled, has no deadline, and has no values. It is the **root of all context trees**.

Use `Background()` when you are creating the top-level context for:
- The `main` function
- Initialization code
- Tests
- The top-level context for an incoming request (though `net/http` does this for you)

```go
func main() {
    ctx := context.Background()
    server := NewServer()
    server.Run(ctx)
}
```

### context.TODO()

```go
ctx := context.TODO()
```

`TODO()` is functionally identical to `Background()`. It returns the exact same kind of empty, never-cancelled context. So **why does it exist?**

It exists as a **signal to code reviewers and static analysis tools**. It says: "I know I need a context here, but I have not figured out which context to use yet."

Use `TODO()` when:
- You are refactoring and have not yet plumbed a context through
- You are writing a prototype and will add proper context handling later
- A function requires a context but you are not sure where to get one

```go
// BAD: Using Background when you should have a real context
func (s *Service) Process() error {
    return s.repo.Save(context.Background(), data)  // Where is the request context?
}

// BETTER: Signals that this needs to be fixed
func (s *Service) Process() error {
    return s.repo.Save(context.TODO(), data)  // TODO: plumb context from handler
}

// BEST: Proper context propagation
func (s *Service) Process(ctx context.Context) error {
    return s.repo.Save(ctx, data)
}
```

### WHY Both Exist

A static analysis tool (like `go vet` or custom linters) can scan for `context.TODO()` calls and flag them as tech debt. If everything used `Background()`, you could never distinguish "this is genuinely the root context" from "I was too lazy to figure out context propagation here."

---

## 3. Cancellation

### context.WithCancel

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

`WithCancel` creates a new context derived from `parent`. It returns:
1. A new `ctx` that will be cancelled when `cancel()` is called
2. A `cancel` function -- calling it cancels `ctx` and all contexts derived from `ctx`

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()  // ALWAYS defer cancel to avoid leaks

    go worker(ctx, "worker-1")
    go worker(ctx, "worker-2")

    time.Sleep(3 * time.Second)
    fmt.Println("Main: telling workers to stop")
    cancel()  // Signal all workers to stop

    time.Sleep(1 * time.Second)  // Give workers time to clean up
}

func worker(ctx context.Context, name string) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s: stopped (%v)\n", name, ctx.Err())
            return
        default:
            fmt.Printf("%s: working...\n", name)
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

Output:
```
worker-1: working...
worker-2: working...
worker-1: working...
worker-2: working...
...
Main: telling workers to stop
worker-1: stopped (context canceled)
worker-2: stopped (context canceled)
```

### The ctx.Done() Channel

`ctx.Done()` returns a `<-chan struct{}`. This channel is **closed** when the context is cancelled. Why `struct{}` and not `bool` or some value? Because `struct{}` is zero-size. You never need to send data through it -- you only need the signal that it is closed.

The key insight: **a closed channel can be read from by an unlimited number of goroutines simultaneously, and they all receive the zero value immediately.** This is how one `cancel()` call can notify thousands of goroutines at once.

```go
// This is how you check for cancellation in a select
select {
case <-ctx.Done():
    // Context was cancelled. Clean up and return.
    return ctx.Err()
case result := <-resultCh:
    // Got a result before cancellation. Process it.
    return process(result)
}
```

### How Cancellation Propagates

This is crucial to understand. Cancellation flows **downward** through the context tree, never upward.

```
            WithCancel (A)          <-- cancel() called here
               /       \
        WithCancel (B)  WithTimeout (C)
            |               |
        WithValue (D)   WithCancel (E)

Cancellation order:
1. A is cancelled
2. B and C are cancelled (children of A)
3. D and E are cancelled (children of B and C)

All of Done() channels close. All of Err() return context.Canceled.
```

### Cancellation Flow Diagram

```
    [HTTP Handler]
         |
         | ctx, cancel := context.WithCancel(r.Context())
         | defer cancel()
         |
         +---> [Service Layer]
         |          |
         |          | Uses same ctx
         |          |
         |          +---> [Database Query]
         |          |         |
         |          |         | select {
         |          |         | case <-ctx.Done(): return ctx.Err()
         |          |         | case row := <-rows: ...
         |          |         | }
         |          |
         |          +---> [External API Call]
         |                    |
         |                    | req, _ := http.NewRequestWithContext(ctx, ...)
         |                    | // Automatically cancelled if ctx is done
         |
         | CLIENT DISCONNECTS
         | r.Context() is cancelled by net/http
         |     |
         |     v
         | ctx.Done() closes
         |     |
         |     +---> Database Query sees ctx.Done(), stops
         |     +---> External API Call sees ctx.Done(), stops
         |
         | Resources freed. No wasted work.
```

### context.WithCancelCause (Go 1.20+)

Go 1.20 added `WithCancelCause`, letting you provide a reason for cancellation:

```go
ctx, cancel := context.WithCancelCause(parent)

// Later, when you need to cancel with a reason:
cancel(fmt.Errorf("user rate limited: %d requests/sec", rate))

// The cause is retrievable:
err := context.Cause(ctx)  // "user rate limited: 5 requests/sec"
```

This is extremely useful for debugging. Instead of just seeing "context canceled" in your logs, you see **why** it was cancelled.

### The Golden Rule: Always Call Cancel

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()  // <-- NEVER forget this
```

WHY? When you call `WithCancel`, Go internally registers the child context with its parent so that parent cancellation propagates to the child. If you never call `cancel()`, this registration is never cleaned up until the parent is cancelled. In a long-running server where `Background()` is the parent, that means **never** -- it is a memory leak.

Even if the context will be cancelled by the parent, calling `cancel()` yourself is safe (it is idempotent) and ensures cleanup happens as early as possible.

---

## 4. Timeouts and Deadlines

### context.WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithTimeout` creates a context that will be automatically cancelled after `timeout` duration elapses.

```go
func fetchUser(ctx context.Context, userID string) (*User, error) {
    // This entire operation must complete within 5 seconds
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET",
        "https://api.example.com/users/"+userID, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("user service timed out after 5s: %w", err)
        }
        return nil, fmt.Errorf("user service error: %w", err)
    }
    defer resp.Body.Close()

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    return &user, nil
}
```

### context.WithDeadline

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

`WithDeadline` creates a context that will be cancelled at a specific point in time, rather than after a duration.

```go
// "This must finish by 5pm"
deadline := time.Date(2026, 3, 24, 17, 0, 0, 0, time.UTC)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
```

In practice, `WithTimeout` is syntactic sugar for `WithDeadline`:

```go
// These are equivalent:
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
ctx, cancel := context.WithDeadline(parent, time.Now().Add(5*time.Second))
```

### The Shortest Deadline Wins

If a parent already has a deadline, the child's deadline can only be **shorter**, never longer. This is a critical safety feature.

```go
// Parent: 10 second timeout
parentCtx, parentCancel := context.WithTimeout(context.Background(), 10*time.Second)
defer parentCancel()

// Child: trying to set 30 second timeout
// The child will STILL be cancelled after 10 seconds (parent's deadline)
childCtx, childCancel := context.WithTimeout(parentCtx, 30*time.Second)
defer childCancel()

deadline, ok := childCtx.Deadline()
// deadline is ~10 seconds from now, NOT 30 seconds
// The parent's tighter deadline wins
```

This prevents a common bug: a downstream function trying to give itself more time than the overall operation allows.

### WHY Timeouts Matter for Production Systems

Without timeouts, a single slow dependency can cascade into a full system outage:

```
WITHOUT TIMEOUTS:
=================

1. External API slows from 100ms to 30s response time
2. Your goroutines pile up, each waiting 30s
3. Each goroutine holds a database connection
4. Connection pool exhausted
5. New requests cannot get database connections
6. Entire server becomes unresponsive
7. Upstream services start timing out against YOU
8. Cascading failure across the entire system

WITH TIMEOUTS:
==============

1. External API slows from 100ms to 30s response time
2. After 2s timeout, your context cancels the call
3. Goroutine returns an error, releases its resources
4. Connection pool stays healthy
5. Requests that do not need the external API continue working
6. You return a degraded response (partial data) instead of no response
7. System remains available
```

### Checking Remaining Time

```go
func processWithDeadlineAwareness(ctx context.Context) error {
    deadline, ok := ctx.Deadline()
    if !ok {
        // No deadline set -- proceed normally
        return doFullProcessing(ctx)
    }

    remaining := time.Until(deadline)
    if remaining < 100*time.Millisecond {
        // Not enough time left -- skip expensive operations
        return doQuickProcessing(ctx)
    }

    return doFullProcessing(ctx)
}
```

### Nested Timeout Pattern

A common real-world pattern is a "timeout budget" where the outer handler sets an overall timeout, and inner functions carve out smaller pieces:

```go
func handleOrder(w http.ResponseWriter, r *http.Request) {
    // Overall request: 10 seconds
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    // Validate user: 2 seconds
    user, err := validateUser(ctx, 2*time.Second)
    if err != nil {
        http.Error(w, "auth failed", http.StatusUnauthorized)
        return
    }

    // Check inventory: 3 seconds
    available, err := checkInventory(ctx, 3*time.Second)
    if err != nil {
        http.Error(w, "inventory check failed", http.StatusServiceUnavailable)
        return
    }

    // Charge payment: 4 seconds
    receipt, err := chargePayment(ctx, 4*time.Second)
    if err != nil {
        http.Error(w, "payment failed", http.StatusPaymentRequired)
        return
    }

    // Remaining time is used for the response
    json.NewEncoder(w).Encode(OrderResponse{
        User:    user,
        Receipt: receipt,
    })
}

func validateUser(ctx context.Context, budget time.Duration) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, budget)
    defer cancel()
    // ... validation logic using ctx ...
}
```

---

## 5. Context Values

### context.WithValue

```go
func WithValue(parent Context, key, val any) Context
```

`WithValue` creates a new context that carries a key-value pair. This is the most controversial part of the context package.

```go
type contextKey string

const userIDKey contextKey = "userID"

func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := extractUserID(r)
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    userID, ok := r.Context().Value(userIDKey).(string)
    if !ok {
        http.Error(w, "no user ID", http.StatusUnauthorized)
        return
    }
    fmt.Fprintf(w, "Hello, user %s", userID)
}
```

### How Value Lookup Works Internally

Context values form a **linked list**. Each `WithValue` call wraps the parent context with a new node. When you call `Value(key)`, Go walks up the chain from child to parent until it finds a match or reaches the root.

```
WithValue(key3, val3)    <-- ctx.Value(key3) = val3 (found immediately)
    |
WithValue(key2, val2)    <-- ctx.Value(key2) = val2 (found after 1 hop)
    |
WithValue(key1, val1)    <-- ctx.Value(key1) = val1 (found after 2 hops)
    |
Background()             <-- ctx.Value(anything_else) = nil
```

This means value lookup is **O(n)** where n is the number of value contexts in the chain. For a handful of values this is fine. For dozens, it is not.

### WHY Custom Key Types Matter

Never use strings or other built-in types as context keys:

```go
// BAD: String keys can collide across packages
ctx = context.WithValue(ctx, "userID", "abc123")
// Another package might also use "userID" as a key!

// GOOD: Unexported custom type prevents collisions
type contextKey string
const userIDKey contextKey = "userID"
ctx = context.WithValue(ctx, userIDKey, "abc123")

// BEST: Unexported empty struct type -- zero-size, impossible to collide
type userIDKeyType struct{}
var userIDKey userIDKeyType
ctx = context.WithValue(ctx, userIDKey, "abc123")
```

WHY? Two different packages might both want to store a "userID" in the context. If they both use the string `"userID"` as the key, they will overwrite each other. An unexported type is unique to your package -- no other package can create a value of that type, so collisions are impossible.

### When to Use Context Values

**Use context values for:**
- Request ID / trace ID (for logging and tracing)
- Authenticated user / claims (set by auth middleware)
- Locale / language preferences
- Values that cross API boundaries and are request-scoped

**Do NOT use context values for:**
- Function parameters (pass them explicitly)
- Optional configuration (use functional options)
- Dependencies (use dependency injection)
- Anything that affects correctness (context values are invisible)

### WHY Context Values Are Controversial

```go
// This function's behavior depends on a hidden input
func processOrder(ctx context.Context, order Order) error {
    // Where does discountRate come from? Who sets it?
    // What happens if it is not set?
    discount, _ := ctx.Value(discountKey).(float64)
    order.Total *= (1 - discount)
    // ...
}

// This function's dependencies are clear
func processOrder(ctx context.Context, order Order, discountRate float64) error {
    order.Total *= (1 - discountRate)
    // ...
}
```

Context values are:
- **Type-unsafe**: `Value()` returns `any`, requiring type assertions
- **Hidden dependencies**: You cannot see from a function signature what values it expects
- **Hard to test**: You need to construct a context with the right values
- **Easy to misuse**: Developers start putting everything in context instead of designing proper APIs

The rule of thumb: **if removing a context value would break your program's correctness, it probably should not be a context value.** Context values should be for cross-cutting concerns (tracing, auth) not business logic inputs.

### Type-Safe Context Value Pattern

The recommended approach is to wrap context value operations in typed accessor functions:

```go
package auth

// unexported key type -- impossible to collide
type claimsKeyType struct{}

var claimsKey claimsKeyType

// Claims represents authenticated user information.
type Claims struct {
    UserID string
    Email  string
    Roles  []string
}

// WithClaims returns a new context carrying the given claims.
func WithClaims(ctx context.Context, claims Claims) context.Context {
    return context.WithValue(ctx, claimsKey, claims)
}

// GetClaims extracts claims from the context.
// Returns the claims and true if found, zero value and false if not.
func GetClaims(ctx context.Context) (Claims, bool) {
    claims, ok := ctx.Value(claimsKey).(Claims)
    return claims, ok
}
```

Now consumers have a clean, typed API:

```go
// In middleware:
ctx = auth.WithClaims(ctx, claims)

// In handler:
claims, ok := auth.GetClaims(ctx)
if !ok {
    http.Error(w, "unauthorized", http.StatusUnauthorized)
    return
}
```

---

## 6. Context in HTTP Servers

### How net/http Creates Contexts

When `net/http` accepts a connection and starts serving a request, it creates a context for that request. This context is cancelled when:

1. The request handler returns (the `ServeHTTP` method completes)
2. The client closes the connection
3. The server shuts down

You access it via `r.Context()`:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // This context is already "live" -- it is tied to the connection.
    // If the client disconnects, ctx.Done() will close.
}
```

### Request Lifecycle with Context

```
Client sends HTTP request
         |
         v
net/http accepts connection
         |
         v
net/http creates context (derived from Server.BaseContext or Background)
         |
         v
ConnContext hook runs (if configured) -- can add connection-scoped values
         |
         v
Request parsed, context attached to *http.Request
         |
         v
Handler runs: r.Context() is available
         |
    +---------+
    |         |
    v         v
Handler     Client
returns   disconnects
    |         |
    v         v
Context is cancelled
         |
         v
All downstream operations see ctx.Done() closed
```

### Detecting Client Disconnection

```go
func longRunningHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    resultCh := make(chan Result, 1)
    go func() {
        resultCh <- expensiveComputation()
    }()

    select {
    case <-ctx.Done():
        // Client disconnected or server shutting down.
        // No point writing a response.
        log.Printf("client gone: %v", ctx.Err())
        return
    case result := <-resultCh:
        json.NewEncoder(w).Encode(result)
    }
}
```

### Adding Context via Middleware

A common pattern is middleware that enriches the request context:

```go
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Create a new context with the request ID
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)

        // Set the response header
        w.Header().Set("X-Request-ID", requestID)

        // Pass the enriched context downstream
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func timeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### Server Shutdown with Context

```go
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }

    // Channel to listen for OS signals
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

    // Start server in a goroutine
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Block until signal received
    <-stop
    log.Println("Shutdown signal received")

    // Give active requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Forced shutdown: %v", err)
    }

    log.Println("Server stopped gracefully")
}
```

When `srv.Shutdown(ctx)` is called:
1. The server stops accepting new connections
2. It waits for all active requests to complete
3. If the context deadline is hit, it forcefully closes remaining connections
4. Each active request's context is cancelled, so well-behaved handlers will notice and return promptly

---

## 7. Context in Database Operations

### Why Context Matters for Database Operations

Database queries can hang. Networks can partition. Connection pools can be exhausted. Without context, a single slow query can tie up a goroutine and a database connection indefinitely.

The `database/sql` package provides context-aware versions of all operations:

| Without Context       | With Context               |
|-----------------------|----------------------------|
| `db.Query(...)`       | `db.QueryContext(ctx, ...)` |
| `db.Exec(...)`        | `db.ExecContext(ctx, ...)`  |
| `db.QueryRow(...)`    | `db.QueryRowContext(ctx, ...)`|
| `db.Begin()`          | `db.BeginTx(ctx, ...)`     |
| `db.Prepare(...)`     | `db.PrepareContext(ctx, ...)`|
| `tx.Commit()`         | (no context variant needed) |

### Basic Usage

```go
func (r *UserRepository) GetByID(ctx context.Context, id string) (*User, error) {
    query := `SELECT id, name, email, created_at FROM users WHERE id = $1`

    var user User
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.CreatedAt,
    )
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, ErrUserNotFound
        }
        return nil, fmt.Errorf("query user %s: %w", id, err)
    }
    return &user, nil
}
```

If the context is cancelled while the query is running, the database driver will attempt to cancel the query on the server side (sending a `KILL QUERY` or equivalent). This prevents orphaned queries from consuming database resources.

### Transaction with Context

```go
func (r *OrderRepository) CreateOrder(ctx context.Context, order Order) error {
    // Start transaction with context -- if ctx is cancelled, the
    // transaction setup is aborted
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    // Always clean up: rollback if we have not committed
    defer tx.Rollback()

    // Insert order
    _, err = tx.ExecContext(ctx,
        `INSERT INTO orders (id, user_id, total) VALUES ($1, $2, $3)`,
        order.ID, order.UserID, order.Total,
    )
    if err != nil {
        return fmt.Errorf("insert order: %w", err)
    }

    // Insert order items
    for _, item := range order.Items {
        _, err = tx.ExecContext(ctx,
            `INSERT INTO order_items (order_id, product_id, quantity, price)
             VALUES ($1, $2, $3, $4)`,
            order.ID, item.ProductID, item.Quantity, item.Price,
        )
        if err != nil {
            return fmt.Errorf("insert item %s: %w", item.ProductID, err)
        }
    }

    // Commit -- if ctx is cancelled between ExecContext and here,
    // the deferred Rollback will clean up
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }

    return nil
}
```

### Real-World Example: Repository with Timeouts

```go
type PostgresUserRepo struct {
    db           *sql.DB
    queryTimeout time.Duration  // e.g., 5 seconds
}

func NewPostgresUserRepo(db *sql.DB) *PostgresUserRepo {
    return &PostgresUserRepo{
        db:           db,
        queryTimeout: 5 * time.Second,
    }
}

func (r *PostgresUserRepo) Search(ctx context.Context, query string) ([]User, error) {
    // Apply a per-query timeout, but respect the parent context's deadline too
    ctx, cancel := context.WithTimeout(ctx, r.queryTimeout)
    defer cancel()

    rows, err := r.db.QueryContext(ctx,
        `SELECT id, name, email FROM users
         WHERE name ILIKE $1 OR email ILIKE $1
         ORDER BY name LIMIT 50`,
        "%"+query+"%",
    )
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("search timed out after %v: %w", r.queryTimeout, err)
        }
        return nil, fmt.Errorf("search query: %w", err)
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, fmt.Errorf("scan row: %w", err)
        }
        users = append(users, u)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("rows iteration: %w", err)
    }

    return users, nil
}
```

---

## 8. Context Best Practices

### Rule 1: Always Pass Context as the First Parameter

```go
// GOOD
func GetUser(ctx context.Context, id string) (*User, error)
func ProcessOrder(ctx context.Context, order Order) error
func (s *Service) FetchData(ctx context.Context) ([]byte, error)

// BAD
func GetUser(id string, ctx context.Context) (*User, error)
func ProcessOrder(order Order, ctx context.Context) error
```

**WHY:** This is a convention established by the standard library and followed by the entire Go ecosystem. `database/sql`, `net/http`, `google.golang.org/grpc`, and virtually every well-maintained library follows this pattern. Consistency makes code readable across teams and organizations. When you see `ctx context.Context` as the first parameter, you immediately know the function supports cancellation.

### Rule 2: Never Store Context in a Struct

```go
// BAD
type Server struct {
    ctx context.Context  // Do not do this
    db  *sql.DB
}

func (s *Server) GetUser(id string) (*User, error) {
    return s.repo.GetByID(s.ctx, id)  // Stale context!
}

// GOOD
type Server struct {
    db *sql.DB
}

func (s *Server) GetUser(ctx context.Context, id string) (*User, error) {
    return s.repo.GetByID(ctx, id)  // Fresh, per-request context
}
```

**WHY:** A context represents a single operation's lifetime. Storing it in a struct means all operations on that struct share the same context. If the context is cancelled (because the original request is done), all subsequent operations will fail immediately, even if they are for different requests.

The one exception: you may store a context in a struct if the struct itself represents a single operation (like an `http.Request`). But even then, you should be cautious.

### Rule 3: Use Custom Key Types for Context Values

```go
// BAD: Exported string key -- collision-prone
const UserIDKey = "user_id"

// BAD: Built-in type key
ctx = context.WithValue(ctx, 42, "some value")

// GOOD: Unexported custom type
type userIDKeyType struct{}
var userIDKey userIDKeyType
```

**WHY:** Covered in detail in Section 5. Unexported types make key collisions across packages impossible, not just unlikely.

### Rule 4: Do Not Use Context Values for Function Parameters

```go
// BAD: Hidden dependency on context value
func CalculateDiscount(ctx context.Context, price float64) float64 {
    tier, _ := ctx.Value(tierKey).(string)
    switch tier {
    case "gold":
        return price * 0.20
    case "silver":
        return price * 0.10
    default:
        return 0
    }
}

// GOOD: Explicit parameter
func CalculateDiscount(price float64, tier string) float64 {
    switch tier {
    case "gold":
        return price * 0.20
    case "silver":
        return price * 0.10
    default:
        return 0
    }
}
```

**WHY:** Function signatures are documentation. When you look at `CalculateDiscount(ctx, price)`, you have no idea it needs a tier value. When you look at `CalculateDiscount(price, tier)`, the contract is clear. Context values should be used for cross-cutting concerns (trace IDs, auth tokens) not for business logic inputs.

### Rule 5: Always Defer the Cancel Function

```go
// GOOD
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

// BAD: cancel never called on the happy path
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
result, err := doWork(ctx)
if err != nil {
    cancel()
    return err
}
return result  // cancel() is never called -- MEMORY LEAK
```

**WHY:** Covered in Section 3. Failing to call `cancel()` leaks the internal goroutine that the runtime uses to track the timeout. `defer cancel()` ensures it is always called regardless of which code path is taken.

### Rule 6: Do Not Pass nil Context

```go
// BAD
doSomething(nil, data)

// GOOD: Use context.TODO() if you truly do not have a context
doSomething(context.TODO(), data)
```

**WHY:** Many functions do `ctx.Done()` or `ctx.Deadline()` without nil-checking. Passing `nil` will cause a panic. `context.TODO()` is a safe placeholder that also signals intent.

---

## 9. Context Propagation

### How Context Flows Through Application Layers

In a well-designed Go application, context flows from the entry point (HTTP handler, gRPC handler, CLI command) through every layer:

```
[Entry Point: HTTP Handler]
         |
         | r.Context()  -- created by net/http, cancelled on disconnect
         |
         v
[Middleware Layer]
         |
         | Adds values: request ID, auth claims, trace span
         | May add timeout: context.WithTimeout(ctx, 30s)
         |
         v
[Service Layer]
         |
         | Business logic. Passes ctx to all I/O operations.
         | May add tighter timeouts for specific operations.
         |
         v
[Repository Layer]
         |
         | db.QueryContext(ctx, ...)
         | Passes ctx to database driver
         |
         v
[External Service Client]
         |
         | http.NewRequestWithContext(ctx, ...)
         | Passes ctx to outgoing HTTP request
         |
         v
[Third-Party API]
```

### Complete Example: Request Flow

```go
// === Entry Point ===
func main() {
    userRepo := NewUserRepo(db)
    userService := NewUserService(userRepo, emailClient)
    handler := NewUserHandler(userService)

    mux := http.NewServeMux()
    mux.Handle("/users/", requestIDMiddleware(authMiddleware(handler)))

    srv := &http.Server{Addr: ":8080", Handler: mux}
    srv.ListenAndServe()
}

// === Middleware ===
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.WithValue(r.Context(), requestIDKey, uuid.New().String())
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        claims, err := validateJWT(r.Header.Get("Authorization"))
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), claimsKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// === Handler Layer ===
type UserHandler struct {
    service *UserService
}

func (h *UserHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Has: request ID, auth claims, cancellation from net/http

    // Extract route parameter
    userID := strings.TrimPrefix(r.URL.Path, "/users/")

    // Set overall timeout for this handler
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    user, err := h.service.GetUser(ctx, userID)
    if err != nil {
        handleError(w, err)
        return
    }

    json.NewEncoder(w).Encode(user)
}

// === Service Layer ===
type UserService struct {
    repo        *UserRepo
    emailClient *EmailClient
}

func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    // Context carries: request ID (for logging), claims (for authorization),
    // timeout (from handler), cancellation (from net/http)

    // Authorization check using context value
    claims, ok := ctx.Value(claimsKey).(Claims)
    if !ok {
        return nil, ErrUnauthorized
    }

    // Check permission
    if claims.UserID != id && !claims.HasRole("admin") {
        return nil, ErrForbidden
    }

    // Log with request ID from context
    reqID, _ := ctx.Value(requestIDKey).(string)
    log.Printf("[%s] fetching user %s", reqID, id)

    // Pass context to repository -- propagates timeout and cancellation
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("find user: %w", err)
    }

    return user, nil
}

// === Repository Layer ===
type UserRepo struct {
    db *sql.DB
}

func (r *UserRepo) FindByID(ctx context.Context, id string) (*User, error) {
    // Context still carries everything from above
    // The database driver will respect cancellation
    var user User
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, email FROM users WHERE id = $1`, id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

### Propagation to External HTTP Calls

```go
func (c *PaymentClient) Charge(ctx context.Context, amount Money) (*Receipt, error) {
    body, _ := json.Marshal(ChargeRequest{Amount: amount})

    // NewRequestWithContext ties the outgoing request to our context.
    // If our context is cancelled, the HTTP client will abort the request.
    req, err := http.NewRequestWithContext(ctx, "POST",
        c.baseURL+"/charge", bytes.NewReader(body))
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+c.apiKey)

    // Propagate trace/request ID for distributed tracing
    if reqID, ok := ctx.Value(requestIDKey).(string); ok {
        req.Header.Set("X-Request-ID", reqID)
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("charge request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("charge failed: status %d", resp.StatusCode)
    }

    var receipt Receipt
    if err := json.NewDecoder(resp.Body).Decode(&receipt); err != nil {
        return nil, fmt.Errorf("decode receipt: %w", err)
    }
    return &receipt, nil
}
```

---

## 10. Common Mistakes

### Mistake 1: Ignoring Context Cancellation

```go
// BAD: Ignores context entirely
func fetchAllPages(ctx context.Context, url string) ([]Page, error) {
    var pages []Page
    for pageNum := 1; ; pageNum++ {
        page, err := fetchPage(url, pageNum)  // No ctx passed!
        if err != nil {
            return pages, err
        }
        if len(page.Items) == 0 {
            break
        }
        pages = append(pages, page)
    }
    return pages, nil
}

// GOOD: Checks context on each iteration
func fetchAllPages(ctx context.Context, url string) ([]Page, error) {
    var pages []Page
    for pageNum := 1; ; pageNum++ {
        // Check if we should stop before making another request
        select {
        case <-ctx.Done():
            return pages, ctx.Err()
        default:
        }

        page, err := fetchPage(ctx, url, pageNum)  // ctx passed!
        if err != nil {
            return pages, err
        }
        if len(page.Items) == 0 {
            break
        }
        pages = append(pages, page)
    }
    return pages, nil
}
```

### Mistake 2: Using String Keys for Context Values

```go
// BAD: Two packages can easily collide
// package auth
ctx = context.WithValue(ctx, "user", authUser)

// package profile
ctx = context.WithValue(ctx, "user", profileUser)  // Overwrites auth's value!

// GOOD: Unexported types cannot collide
// package auth
type authUserKey struct{}
ctx = context.WithValue(ctx, authUserKey{}, authUser)

// package profile
type profileUserKey struct{}
ctx = context.WithValue(ctx, profileUserKey{}, profileUser)  // Different key!
```

### Mistake 3: Creating Context in the Wrong Place

```go
// BAD: Creating a new context deep in the call chain
// This disconnects from the request's lifecycle
func (r *Repo) Save(data Data) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    return r.db.ExecContext(ctx, "INSERT ...", data)
}
// If the HTTP request is cancelled, this operation continues running!

// GOOD: Accept context from caller
func (r *Repo) Save(ctx context.Context, data Data) error {
    // Optionally add a tighter timeout
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    return r.db.ExecContext(ctx, "INSERT ...", data)
}
```

### Mistake 4: Using context.WithValue for Dependency Injection

```go
// BAD: Hiding dependencies in context
func handleRequest(ctx context.Context) error {
    db := ctx.Value(dbKey).(*sql.DB)           // Hidden dependency
    logger := ctx.Value(loggerKey).(*Logger)    // Hidden dependency
    cache := ctx.Value(cacheKey).(Cache)        // Hidden dependency
    // ...
}

// GOOD: Explicit dependencies via struct
type Handler struct {
    db     *sql.DB
    logger *Logger
    cache  Cache
}

func (h *Handler) handleRequest(ctx context.Context) error {
    // Dependencies are visible and testable
    // ...
}
```

### Mistake 5: Not Calling Cancel (Memory Leak)

```go
// BAD: cancel function is discarded
func quickCheck(parentCtx context.Context) bool {
    ctx, _ := context.WithTimeout(parentCtx, 100*time.Millisecond)
    // The _ discards the cancel function!
    // The timer goroutine will live until the timeout expires,
    // even if this function returns immediately.
    result, err := check(ctx)
    return err == nil && result
}

// GOOD: Always capture and defer cancel
func quickCheck(parentCtx context.Context) bool {
    ctx, cancel := context.WithTimeout(parentCtx, 100*time.Millisecond)
    defer cancel()  // Immediately cleans up the timer
    result, err := check(ctx)
    return err == nil && result
}
```

### Mistake 6: Passing Context Across Goroutine Boundaries Without Thought

```go
// POTENTIALLY BAD: Background work using request context
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Fire-and-forget background work
    go sendAnalytics(ctx, event)  // ctx will be cancelled when handler returns!

    w.WriteHeader(http.StatusOK)
}
// The handler returns, net/http cancels ctx, sendAnalytics is cancelled.

// GOOD: Use a separate context for background work
func handler(w http.ResponseWriter, r *http.Request) {
    // Copy values we need, but use a new context for the background task
    bgCtx, bgCancel := context.WithTimeout(context.Background(), 10*time.Second)

    // Copy request-scoped values we need
    if reqID, ok := r.Context().Value(requestIDKey).(string); ok {
        bgCtx = context.WithValue(bgCtx, requestIDKey, reqID)
    }

    go func() {
        defer bgCancel()
        sendAnalytics(bgCtx, event)
    }()

    w.WriteHeader(http.StatusOK)
}
```

### Mistake 7: Overly Tight Timeouts in a Chain

```go
// BAD: Each layer adds its own timeout without considering the parent
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    // Service sets its own timeout...
    svc.Process(ctx)  // Inside: WithTimeout(ctx, 10*time.Second) -- redundant
}

// GOOD: Derive from parent context, use smaller budgets
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    svc.Process(ctx)  // Inside: WithTimeout(ctx, 3*time.Second) -- tighter budget
}
```

---

## 11. Go vs Node.js Comparison

### Cancellation: Go Context vs Node.js AbortController

Go provides cancellation as a first-class concept baked into the standard library. Node.js adopted `AbortController` (from the browser's Fetch API) starting in Node.js 15.

**Go:**
```go
func fetchWithCancel(ctx context.Context) ([]byte, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/data", nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err  // Returns context.Canceled if ctx was cancelled
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// Usage: cancel propagates automatically through the call chain
func handler(w http.ResponseWriter, r *http.Request) {
    data, err := fetchWithCancel(r.Context())  // Cancelled if client disconnects
    // ...
}
```

**Node.js:**
```javascript
async function fetchWithCancel(signal) {
    const response = await fetch('https://api.example.com/data', { signal });
    return response.json();  // Throws AbortError if signal is aborted
}

// Usage: you must manually thread the signal
app.get('/data', async (req, res) => {
    const controller = new AbortController();
    req.on('close', () => controller.abort());  // Manual wiring!

    try {
        const data = await fetchWithCancel(controller.signal);
        res.json(data);
    } catch (err) {
        if (err.name === 'AbortError') {
            // Client disconnected
            return;
        }
        res.status(500).json({ error: err.message });
    }
});
```

**Key Differences:**

| Aspect                 | Go Context                          | Node.js AbortController            |
|------------------------|--------------------------------------|--------------------------------------|
| Standard library       | Yes, since Go 1.7                    | Yes, since Node.js 15               |
| Automatic propagation  | Yes, through context tree            | No, must pass signal manually        |
| HTTP server integration| Built-in (r.Context())               | Manual wiring required               |
| DB driver support      | Universal (QueryContext, etc.)        | Limited, driver-dependent            |
| Carries values         | Yes (WithValue)                      | No (signal is cancellation only)     |
| Carries deadlines      | Yes (WithTimeout, WithDeadline)      | No (must combine with setTimeout)    |

### Timeouts: Go Context vs Node.js setTimeout + AbortController

**Go:**
```go
func callExternalAPI(ctx context.Context) (*Result, error) {
    // One line: timeout + cancellation + propagation
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("API call timed out")
        }
        return nil, err
    }
    defer resp.Body.Close()
    // ...
}
```

**Node.js:**
```javascript
async function callExternalAPI(signal) {
    // Must compose timeout + parent signal manually
    const controller = new AbortController();

    // Abort after 5 seconds
    const timeout = setTimeout(() => controller.abort(), 5000);

    // Also abort if parent signal fires
    if (signal) {
        signal.addEventListener('abort', () => controller.abort());
    }

    try {
        const response = await fetch(url, { signal: controller.signal });
        return await response.json();
    } finally {
        clearTimeout(timeout);  // Clean up timer
    }
}

// Node.js 18.14+ has AbortSignal.timeout() as a shortcut:
async function callExternalAPIModern() {
    const response = await fetch(url, {
        signal: AbortSignal.timeout(5000),  // Simpler, but no parent signal
    });
    return response.json();
}
```

Go's approach is more elegant because timeout, cancellation, and propagation are a single unified concept. In Node.js, you must compose multiple primitives.

### Request-Scoped Values: Go Context vs Node.js AsyncLocalStorage

Go's `context.WithValue` and Node.js's `AsyncLocalStorage` solve the same problem: carrying request-scoped data through the call chain without passing it explicitly through every function.

**Go:**
```go
// Middleware sets value
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := validateToken(r.Header.Get("Authorization"))
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Deep in the call chain, retrieve value
func (r *AuditRepo) LogAction(ctx context.Context, action string) {
    userID, _ := ctx.Value(userIDKey).(string)
    r.db.ExecContext(ctx, "INSERT INTO audit_log ...", userID, action)
}
```

**Node.js:**
```javascript
const { AsyncLocalStorage } = require('async_hooks');
const asyncLocalStorage = new AsyncLocalStorage();

// Middleware sets value
app.use((req, res, next) => {
    const userID = validateToken(req.headers.authorization);
    asyncLocalStorage.run({ userID }, () => {
        next();
    });
});

// Deep in the call chain, retrieve value -- no explicit passing!
class AuditRepo {
    async logAction(action) {
        const { userID } = asyncLocalStorage.getStore();
        await db.query('INSERT INTO audit_log ...', [userID, action]);
    }
}
```

**Key Differences:**

| Aspect                    | Go context.WithValue             | Node.js AsyncLocalStorage       |
|---------------------------|-----------------------------------|---------------------------------|
| Explicit passing          | Yes, ctx is passed through       | No, implicit via async chain    |
| Visible in signatures     | Yes                               | No (invisible)                  |
| Type safety               | No (any -> type assertion)        | No (getStore() returns any)     |
| Performance               | O(n) linked list lookup           | O(1) lookup per async context   |
| Ecosystem adoption        | Universal in Go                   | Growing, but not universal      |
| Available since           | Go 1.7 (2016)                     | Node.js 13.10 (2020), stable 16 |

Go's approach is more **explicit** -- you always see `ctx` being passed. Node.js's approach is more **implicit** -- values flow through the async chain invisibly. The Go community generally prefers the explicitness, arguing it makes data flow easier to trace and debug.

### HTTP Request Context: Go vs Express

**Go:**
```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Context is part of the request, created by the server
    ctx := r.Context()

    // Automatically cancelled when:
    // - Handler returns
    // - Client disconnects
    // - Server shuts down

    // Add values via middleware chain
    userID, _ := ctx.Value(userIDKey).(string)
    reqID, _ := ctx.Value(requestIDKey).(string)

    // Pass to downstream operations
    user, err := userService.Get(ctx, userID)
}
```

**Express (Node.js):**
```javascript
app.get('/users/:id', async (req, res) => {
    // The req object IS the context carrier
    // Values are typically attached directly:
    const userID = req.user.id;       // Set by auth middleware
    const reqID = req.requestId;      // Set by request-id middleware

    // No built-in cancellation!
    // req.on('close') is available but rarely used
    const user = await userService.get(req.params.id);
    // If client disconnects, this keeps running...
});
```

In Go, the request context provides cancellation, deadlines, AND values in a single mechanism. In Express, the `req` object carries values but has no built-in cancellation propagation. You can listen for `req.on('close')` but it requires manual wiring at every layer.

### Graceful Shutdown: Go vs Node.js

**Go:**
```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Wait for interrupt
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    // Graceful shutdown with 30 second deadline
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Shutdown stops accepting new connections and waits for active ones.
    // Active request contexts are cancelled when the deadline hits.
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("Forced shutdown: %v", err)
    }
    log.Println("Server exited")
}
```

**Node.js:**
```javascript
const server = app.listen(8080, () => {
    console.log('Server started on :8080');
});

// Track active connections for forceful shutdown
const connections = new Set();
server.on('connection', (conn) => {
    connections.add(conn);
    conn.on('close', () => connections.delete(conn));
});

process.on('SIGTERM', () => {
    console.log('Shutdown signal received');

    // Stop accepting new connections
    server.close(() => {
        console.log('Server exited');
        process.exit(0);
    });

    // Force-close after 30 seconds
    setTimeout(() => {
        console.log('Forcing shutdown');
        connections.forEach((conn) => conn.destroy());
        process.exit(1);
    }, 30000);
});
```

Go's `Shutdown(ctx)` elegantly combines "wait for active requests" and "force stop after deadline" in a single call, using the same context mechanism you use everywhere else. Node.js requires manual connection tracking and timer management.

### Summary Table: Go Context vs Node.js Equivalents

| Capability                | Go                          | Node.js                                    |
|---------------------------|-----------------------------|---------------------------------------------|
| Cancellation signal       | `ctx.Done()` channel        | `AbortSignal`                               |
| Create cancellable        | `context.WithCancel()`      | `new AbortController()`                     |
| Timeout                   | `context.WithTimeout()`     | `AbortSignal.timeout()` + `setTimeout`      |
| Request-scoped values     | `context.WithValue()`       | `AsyncLocalStorage` / `req` object          |
| HTTP cancel on disconnect | Automatic (`r.Context()`)   | Manual (`req.on('close')`)                  |
| DB query cancellation     | `db.QueryContext(ctx)`      | Driver-specific, inconsistent               |
| Graceful shutdown         | `srv.Shutdown(ctx)`         | `server.close()` + manual timers            |
| Propagation mechanism     | Pass `ctx` as first arg     | Pass `signal` / use `AsyncLocalStorage`     |
| Ecosystem adoption        | Universal, standard         | Fragmented, improving                       |

The fundamental difference is that Go designed `context` as a **single, unified mechanism** for cancellation, deadlines, and values -- and the entire ecosystem adopted it. Node.js has separate mechanisms for each concern (`AbortController`, `setTimeout`, `AsyncLocalStorage`, `req`) that must be manually composed.

---

## 12. Key Takeaways

1. **Context solves coordination across goroutines and API boundaries.** Without it, cancellation, timeouts, and request scoping require ad-hoc, inconsistent solutions.

2. **Context forms a tree.** Cancelling a parent cancels all children. Children cannot affect parents. This gives you precise, hierarchical control over operation lifetimes.

3. **Always pass context as the first parameter.** This is not just convention -- it makes cancellation support visible in every function signature.

4. **Always `defer cancel()`.** Failing to call cancel leaks resources. It is safe to call cancel multiple times.

5. **The shortest deadline wins.** A child context cannot extend a parent's deadline. This prevents downstream code from overriding safety limits.

6. **Use context values sparingly.** They are for cross-cutting concerns like trace IDs and auth claims, not for function parameters or dependencies. Wrap them in typed accessor functions.

7. **Never store context in a struct.** Context represents a single operation's lifetime. Store it in a struct and you lose that per-operation scoping.

8. **Check for cancellation in long-running loops.** If your function iterates or polls, check `ctx.Done()` on each iteration.

9. **Use separate contexts for background work.** If you spawn a goroutine that should outlive the request, derive from `context.Background()`, not the request context.

10. **Context is Go's biggest advantage over Node.js for production systems.** The unified mechanism for cancellation, timeouts, and values -- adopted universally across the ecosystem -- makes it dramatically easier to build robust, resource-efficient services.

---

## 13. Practice Exercises

### Exercise 1: Basic Cancellation

Write a program that starts 5 worker goroutines. Each worker prints its ID every second. After 5 seconds, the main function cancels all workers using context. Verify that all workers stop.

**Bonus:** Use `context.WithCancelCause` and print the cancellation reason in each worker.

### Exercise 2: Timeout Budget

Write an HTTP handler that:
1. Sets an overall 10-second timeout
2. Calls `validateInput(ctx)` with a 2-second budget
3. Calls `fetchFromDB(ctx)` with a 3-second budget
4. Calls `callExternalAPI(ctx)` with a 4-second budget

Simulate each function with `time.Sleep` of varying durations. Test what happens when:
- All functions complete quickly
- The external API takes 6 seconds (exceeds its budget)
- The total time exceeds 10 seconds

### Exercise 3: Context Value Middleware Chain

Build a minimal HTTP server with three middleware layers:
1. `requestIDMiddleware` -- generates and attaches a UUID
2. `authMiddleware` -- extracts a user ID from a header and attaches it
3. `loggingMiddleware` -- logs the request ID and user ID from context

Use proper unexported key types and typed accessor functions. Write a handler that reads both values from context and returns them in the response.

### Exercise 4: Graceful Shutdown

Write a server that:
1. Handles requests that simulate slow work (sleep 5-15 seconds)
2. Listens for SIGTERM
3. On SIGTERM, stops accepting new connections
4. Gives active requests 10 seconds to complete
5. Forcefully closes after 10 seconds

Test by sending multiple requests, then sending SIGTERM. Verify that in-progress requests either complete or are cancelled.

### Exercise 5: Context-Aware Pipeline

Build a data processing pipeline with three stages, each running in its own goroutine and connected by channels:

1. **Producer**: Generates integers 1 through 1000, one per millisecond
2. **Transformer**: Squares each number
3. **Consumer**: Prints results

Use context to implement:
- Cancel the entire pipeline after 500ms
- Each stage should check `ctx.Done()` and exit cleanly
- No goroutine leaks -- verify with `runtime.NumGoroutine()`

### Exercise 6: Comparing Go and Node.js (Thought Exercise)

Rewrite Exercise 2 (Timeout Budget) conceptually in Node.js using `AbortController` and `AbortSignal.timeout()`. Consider:
- How do you propagate the overall timeout?
- How do you check remaining time before starting the next step?
- How do you handle partial completion if a middle step times out?
- What is easier in Go? What is easier in Node.js?

---

**Next Chapter:** [Chapter 17: Testing in Go -- Unit Tests, Benchmarks, and Table-Driven Tests](./17-testing.md)
