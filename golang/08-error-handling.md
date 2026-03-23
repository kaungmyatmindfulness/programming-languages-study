# Chapter 8: Error Handling & Custom Errors

## Table of Contents

1. [Go's Error Philosophy](#1-gos-error-philosophy)
2. [The error Interface](#2-the-error-interface)
3. [Creating Errors](#3-creating-errors)
4. [Custom Error Types](#4-custom-error-types)
5. [Error Wrapping and Unwrapping](#5-error-wrapping-and-unwrapping)
6. [Sentinel Errors](#6-sentinel-errors)
7. [panic and recover](#7-panic-and-recover)
8. [Error Handling Patterns](#8-error-handling-patterns)
9. [Error Handling Best Practices](#9-error-handling-best-practices)
10. [Key Takeaways](#10-key-takeaways)
11. [Practice Exercises](#11-practice-exercises)

---

## 1. Go's Error Philosophy

### Errors Are Values, Not Exceptions

Go made a deliberate, controversial decision: **errors are ordinary values**, not a special control-flow mechanism. There is no `try`, no `catch`, no `finally`, no `throw`. An error in Go is just a value returned from a function, like any other value.

```go
file, err := os.Open("config.json")
if err != nil {
    // handle the error
    log.Fatal(err)
}
// use file
```

This is the single most common pattern in Go code. You will write `if err != nil` hundreds of times. This is intentional.

### WHY Go Chose This Approach

The designers of Go -- Rob Pike, Ken Thompson, and Robert Griesemer -- had decades of experience with C, C++, and languages that use exceptions. They chose explicit error handling for several deeply considered reasons:

**1. No hidden control flow.**

In languages with exceptions, any function call can secretly throw. You cannot tell by reading a line of code whether it might jump to a distant catch block. Consider this JavaScript:

```javascript
// JS -- which of these lines can throw?
const data = parseConfig(raw);       // maybe throws?
const user = await fetchUser(id);    // maybe throws?
const result = transform(data, user); // maybe throws?
```

The answer is: *all of them, or none of them*. You cannot know without reading the implementation of every function. Exceptions create invisible exit paths through your code.

In Go, the control flow is always visible:

```go
data, err := parseConfig(raw)
if err != nil {
    return fmt.Errorf("parsing config: %w", err)
}

user, err := fetchUser(id)
if err != nil {
    return fmt.Errorf("fetching user %d: %w", id, err)
}

result, err := transform(data, user)
if err != nil {
    return fmt.Errorf("transforming data: %w", err)
}
```

Every potential failure point is explicit. Every error gets handled (or deliberately ignored) right where it happens.

**2. Forces you to think about errors at every step.**

Exception-based languages tempt you to wrap large blocks in a single try/catch and handle everything generically. Go forces you to consider each error individually. This produces more robust software because you are making a conscious decision about every failure mode.

**3. Errors are just values -- you can compute with them.**

Because errors are ordinary values, you can store them in variables, put them in slices, pass them to functions, aggregate them, filter them, and do anything else you do with values. This is more powerful than a special exception mechanism.

**4. Simplicity.**

Go's error handling requires no special syntax, no special runtime machinery, no stack unwinding. It is function calls and return values -- the simplest concepts in programming.

### The Node.js Comparison: A Familiar Root

If you come from Node.js, Go's error philosophy should feel partially familiar. The original Node.js callback pattern was essentially the same idea:

```javascript
// Node.js callback pattern (pre-Promise era)
fs.readFile('config.json', (err, data) => {
    if (err) {
        console.error('Failed to read config:', err);
        return;
    }
    // use data
});
```

This `(err, data)` callback convention is strikingly similar to Go's `(result, error)` return convention. Node.js eventually moved toward Promises and async/await with try/catch -- Go deliberately stayed with explicit error returns.

| Aspect | Go | Node.js |
|--------|-----|---------|
| Error mechanism | Return values | Exceptions (throw/catch) |
| Error type | `error` interface | `Error` object |
| Handling | `if err != nil` at each call site | `try/catch` block around code |
| Propagation | Explicit return | Automatic stack unwinding |
| Async errors | Same pattern (errors are values) | Unhandled promise rejections, `.catch()` |
| Hidden failures | Impossible (compiler warns unused returns) | Common (unhandled rejections, swallowed errors) |

---

## 2. The error Interface

### The Definition

The entire `error` interface in Go is:

```go
type error interface {
    Error() string
}
```

That is it. One method. Any type that has an `Error() string` method satisfies the `error` interface.

### WHY It Is So Simple

This simplicity is a feature, not a limitation:

1. **Any type can be an error.** You do not need to inherit from a base class. A struct, a string, an int -- anything can be an error if it implements `Error() string`.

2. **No framework lock-in.** Because the interface is so minimal, there is no "error framework" to learn. Every library in Go uses the same interface.

3. **Composability.** Simple interfaces compose better. You can add behavior by wrapping errors, not by inheriting from richer base types.

Compare with JavaScript's `Error` class:

```javascript
// JS Error has a fixed shape
class Error {
    constructor(message) {
        this.message = message;
        this.stack = '...';  // automatic stack trace
        this.name = 'Error';
    }
}
```

JavaScript's `Error` gives you a stack trace automatically, which is useful for debugging. Go's error does not include a stack trace by default -- this is intentional. Stack traces are expensive to capture, and Go prefers that you add meaningful context (what operation failed, with what arguments) rather than relying on a raw stack dump.

### The nil Error

The zero value of an interface in Go is `nil`. So `error` being an interface means:

```go
func doSomething() error {
    // everything went fine
    return nil  // nil means "no error"
}

err := doSomething()
if err != nil {
    // this block is skipped when err is nil
}
```

This is clean and idiomatic. `nil` means success. A non-nil error means failure.

### A Critical Gotcha: nil Interface vs nil Pointer

This is one of Go's most infamous pitfalls:

```go
type MyError struct {
    Message string
}

func (e *MyError) Error() string {
    return e.Message
}

func doSomething() error {
    var err *MyError  // nil pointer of type *MyError
    // ... some logic that does NOT set err ...
    return err  // DANGER: this is NOT nil!
}

func main() {
    err := doSomething()
    if err != nil {
        // This WILL execute, even though err is "nil"!
        fmt.Println("got error:", err)  // panic: nil pointer dereference
    }
}
```

**Why this happens:** An interface value in Go has two components: a type and a value. When you return a `*MyError(nil)`, the interface holds `(type=*MyError, value=nil)`. This is *not* the same as a nil interface `(type=nil, value=nil)`. The `err != nil` check tests whether the interface itself is nil, and it is not -- it has a type.

**The fix:** Always return the `error` interface type directly:

```go
func doSomething() error {
    var err *MyError
    // ...
    if err != nil {
        return err
    }
    return nil  // return bare nil, not a typed nil pointer
}
```

This gotcha does not exist in JavaScript because JS does not have interfaces or this kind of two-component value.

---

## 3. Creating Errors

Go provides several ways to create errors, each suited to different situations.

### errors.New() -- Simple Static Errors

```go
import "errors"

err := errors.New("something went wrong")
```

Use `errors.New()` when:
- The error message is a fixed string with no dynamic data.
- You are defining a sentinel error (more on this later).
- The error does not need to wrap another error.

```go
// Good uses of errors.New()
var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")

func validate(name string) error {
    if name == "" {
        return errors.New("name cannot be empty")
    }
    return nil
}
```

### fmt.Errorf() -- Errors with Formatted Context

```go
import "fmt"

err := fmt.Errorf("failed to read file %s: permission denied", filename)
```

Use `fmt.Errorf()` when:
- You need to include dynamic values (filenames, IDs, counts) in the error message.
- You want to wrap an existing error with additional context.

```go
func readConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("reading config from %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return Config{}, fmt.Errorf("parsing config from %s: %w", path, err)
    }

    return cfg, nil
}
```

### The %w Verb -- Wrapping Errors

The `%w` verb in `fmt.Errorf()` is special. It wraps the original error so that it can be unwrapped later with `errors.Is()` and `errors.As()`.

```go
originalErr := errors.New("connection refused")

// %v formats the error as a string -- the original error is LOST
lostErr := fmt.Errorf("database call failed: %v", originalErr)

// %w wraps the error -- the original error is PRESERVED
wrappedErr := fmt.Errorf("database call failed: %w", originalErr)

// You can check for the original error through the wrapper
fmt.Println(errors.Is(wrappedErr, originalErr))  // true
fmt.Println(errors.Is(lostErr, originalErr))      // false
```

**When to use %w vs %v:**

| Use `%w` (wrap) | Use `%v` (format) |
|---|---|
| Callers might need to check the underlying error | The underlying error is an implementation detail |
| You want to build an error chain | You want to hide the original error |
| Public API where error types are part of the contract | Internal code where you control all callers |

### Comparison with JavaScript Error Creation

```javascript
// JS -- creating errors
const err1 = new Error("something went wrong");
const err2 = new Error(`failed to read file ${filename}`);

// JS -- error wrapping (ES2022 Error.cause)
try {
    await readFile(path);
} catch (cause) {
    throw new Error(`config read failed: ${path}`, { cause });
}
```

```go
// Go equivalents
err1 := errors.New("something went wrong")
err2 := fmt.Errorf("failed to read file %s", filename)

// Go -- error wrapping
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("config read failed: %s: %w", path, err)
}
```

The key difference: in JavaScript, wrapping via `Error.cause` was added in ES2022 and is still not universally used. In Go, `%w` wrapping has been the standard since Go 1.13 (2019) and is deeply ingrained in the ecosystem.

---

## 4. Custom Error Types

### When errors.New() Is Not Enough

Sometimes you need errors that carry structured data beyond a message string. This is where custom error types shine.

### Implementing the error Interface

Any struct can become an error by implementing `Error() string`:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on field %q: %s", e.Field, e.Message)
}

// Usage
func validateUser(u User) error {
    if u.Email == "" {
        return &ValidationError{
            Field:   "email",
            Message: "cannot be empty",
        }
    }
    if u.Age < 0 {
        return &ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    return nil
}
```

### Real-World Pattern: HTTP Error with Status Code

```go
type HTTPError struct {
    StatusCode int
    Message    string
    Err        error  // the underlying error
}

func (e *HTTPError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("HTTP %d: %s: %v", e.StatusCode, e.Message, e.Err)
    }
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

// Implement Unwrap so errors.Is/As can see the underlying error
func (e *HTTPError) Unwrap() error {
    return e.Err
}

// Usage
func fetchUser(id int) (User, error) {
    resp, err := http.Get(fmt.Sprintf("/api/users/%d", id))
    if err != nil {
        return User{}, &HTTPError{
            StatusCode: 500,
            Message:    "failed to reach user service",
            Err:        err,
        }
    }

    if resp.StatusCode == 404 {
        return User{}, &HTTPError{
            StatusCode: 404,
            Message:    fmt.Sprintf("user %d not found", id),
        }
    }

    // ... parse response
    return user, nil
}
```

### Real-World Pattern: Multi-Field Validation Error

```go
type ValidationErrors struct {
    Errors []ValidationError
}

func (e *ValidationErrors) Error() string {
    msgs := make([]string, len(e.Errors))
    for i, ve := range e.Errors {
        msgs[i] = ve.Error()
    }
    return strings.Join(msgs, "; ")
}

func validateUser(u User) error {
    var errs []ValidationError

    if u.Email == "" {
        errs = append(errs, ValidationError{Field: "email", Message: "required"})
    }
    if u.Age < 0 {
        errs = append(errs, ValidationError{Field: "age", Message: "must be >= 0"})
    }
    if u.Name == "" {
        errs = append(errs, ValidationError{Field: "name", Message: "required"})
    }

    if len(errs) > 0 {
        return &ValidationErrors{Errors: errs}
    }
    return nil
}
```

### Comparison: Custom Errors in Go vs JavaScript

```javascript
// JavaScript -- custom error class
class ValidationError extends Error {
    constructor(field, message) {
        super(`Validation failed on "${field}": ${message}`);
        this.name = 'ValidationError';
        this.field = field;
    }
}

class HTTPError extends Error {
    constructor(statusCode, message, cause) {
        super(`HTTP ${statusCode}: ${message}`, { cause });
        this.name = 'HTTPError';
        this.statusCode = statusCode;
    }
}

// Using them
try {
    const user = await fetchUser(id);
} catch (err) {
    if (err instanceof HTTPError) {
        console.log(err.statusCode);
    } else if (err instanceof ValidationError) {
        console.log(err.field);
    }
}
```

```go
// Go -- same patterns but with explicit checks
user, err := fetchUser(id)
if err != nil {
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        fmt.Println(httpErr.StatusCode)
    }

    var valErr *ValidationError
    if errors.As(err, &valErr) {
        fmt.Println(valErr.Field)
    }
}
```

Key differences:
- JavaScript uses `instanceof` for type checking; Go uses `errors.As()`.
- JavaScript custom errors extend `Error` via class inheritance; Go custom errors implement the `error` interface via a method.
- Go's approach works through wrapped error chains; JavaScript's `instanceof` does not traverse `Error.cause`.

---

## 5. Error Wrapping and Unwrapping

### The Problem: Bare Errors Lose Context

Imagine a user sees this error in a log:

```
open config.json: no such file or directory
```

Which config.json? Loaded by which function? During startup? During a request? During a background job? The raw OS error gives you almost nothing.

Now imagine this:

```
starting server: loading TLS config: reading certificate from /etc/app/cert.pem: open /etc/app/cert.pem: no such file or directory
```

This tells you exactly what happened, why, and where. This is what error wrapping gives you.

### Building Error Chains with %w

Every time you handle an error and return it, you should wrap it with context about what *your* function was doing:

```go
func loadCertificate(path string) (tls.Certificate, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return tls.Certificate{}, fmt.Errorf("reading certificate from %s: %w", path, err)
    }

    cert, err := tls.X509KeyPair(data, keyData)
    if err != nil {
        return tls.Certificate{}, fmt.Errorf("parsing certificate from %s: %w", path, err)
    }

    return cert, nil
}

func loadTLSConfig() (*tls.Config, error) {
    cert, err := loadCertificate("/etc/app/cert.pem")
    if err != nil {
        return nil, fmt.Errorf("loading TLS config: %w", err)
    }
    // ...
    return config, nil
}

func startServer() error {
    tlsCfg, err := loadTLSConfig()
    if err != nil {
        return fmt.Errorf("starting server: %w", err)
    }
    // ...
    return nil
}
```

Each layer adds its own context. The final error reads like a story, from the highest level of abstraction to the lowest.

### errors.Is() -- Checking for Specific Errors in the Chain

`errors.Is()` traverses the entire chain of wrapped errors to find a match:

```go
import (
    "errors"
    "io/fs"
    "os"
)

_, err := os.Open("missing.txt")
// err is: *fs.PathError wrapping syscall.ENOENT

// Direct comparison would fail on wrapped errors:
fmt.Println(err == fs.ErrNotExist)         // false (err is *PathError, not ErrNotExist)

// errors.Is traverses the chain:
fmt.Println(errors.Is(err, fs.ErrNotExist)) // true -- found it inside the chain
```

### errors.As() -- Extracting a Specific Error Type from the Chain

`errors.As()` finds the first error in the chain that matches a target type and extracts it:

```go
_, err := os.Open("missing.txt")

var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Operation:", pathErr.Op)    // "open"
    fmt.Println("Path:", pathErr.Path)        // "missing.txt"
    fmt.Println("Underlying:", pathErr.Err)   // "no such file or directory"
}
```

**The syntax might look odd.** `errors.As()` takes a pointer to the target variable because it needs to *set* that variable when it finds a match. This is a common Go pattern for "out parameters."

### How Unwrap() Works

When you use `%w`, `fmt.Errorf` returns an error that implements:

```go
type interface {
    Unwrap() error
}
```

`errors.Is()` and `errors.As()` call `Unwrap()` repeatedly to walk the chain:

```go
// Conceptually, errors.Is does something like:
func Is(err, target error) bool {
    for {
        if err == target {
            return true
        }
        unwrapper, ok := err.(interface{ Unwrap() error })
        if !ok {
            return false
        }
        err = unwrapper.Unwrap()
    }
}
```

Your custom error types can participate in this chain by implementing `Unwrap()`:

```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
    return e.Err
}
```

### Multiple Wrapping (Go 1.20+)

Go 1.20 added the ability to wrap multiple errors with `%w`:

```go
err := fmt.Errorf("multiple failures: %w and %w", err1, err2)

// Now both are findable
errors.Is(err, err1) // true
errors.Is(err, err2) // true
```

Your custom types can also wrap multiple errors by implementing:

```go
type MultiError struct {
    Errs []error
}

func (e *MultiError) Error() string {
    // ...format the message
}

// Unwrap returns all wrapped errors (Go 1.20+ convention)
func (e *MultiError) Unwrap() []error {
    return e.Errs
}
```

### Comparison: Go Error Wrapping vs JavaScript Error.cause

```javascript
// JavaScript (ES2022)
async function loadConfig(path) {
    try {
        const data = await fs.readFile(path, 'utf8');
        return JSON.parse(data);
    } catch (err) {
        throw new Error(`failed to load config from ${path}`, { cause: err });
    }
}

async function startServer() {
    try {
        const config = await loadConfig('/etc/app/config.json');
    } catch (err) {
        // Manual traversal of cause chain
        console.error(err.message);       // "failed to load config from ..."
        console.error(err.cause?.message); // "ENOENT: no such file ..."

        // No built-in equivalent to errors.Is or errors.As
        // You must manually walk err.cause
    }
}
```

```go
// Go
func loadConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("failed to load config from %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return Config{}, fmt.Errorf("parsing config %s: %w", path, err)
    }
    return cfg, nil
}

func startServer() error {
    cfg, err := loadConfig("/etc/app/config.json")
    if err != nil {
        // Built-in chain traversal
        if errors.Is(err, fs.ErrNotExist) {
            log.Println("Config file missing, using defaults")
            cfg = defaultConfig()
        } else {
            return fmt.Errorf("starting server: %w", err)
        }
    }
    // ...
}
```

Key differences:

| Feature | Go | JavaScript |
|---|---|---|
| Wrapping mechanism | `fmt.Errorf("...: %w", err)` | `new Error("...", { cause: err })` |
| Chain traversal | `errors.Is()`, `errors.As()` -- built-in, recursive | Manual: `err.cause?.cause?.cause` |
| Type checking in chain | `errors.As(err, &target)` | No built-in equivalent |
| Value checking in chain | `errors.Is(err, sentinel)` | No built-in equivalent |
| Ecosystem adoption | Universal since Go 1.13 | Patchy -- `cause` is still underused |

---

## 6. Sentinel Errors

### What Are Sentinel Errors?

Sentinel errors are package-level exported variables that represent specific, well-known error conditions:

```go
// From the standard library
var EOF = errors.New("EOF")                        // io package
var ErrNotExist = errors.New("file does not exist") // fs package
var ErrNoRows = errors.New("sql: no rows in result set") // database/sql
var ErrClosed = errors.New("http: Server closed")  // net/http
```

They are called "sentinels" because they stand guard at the boundary of your API, signaling specific conditions that callers should check for.

### Using Sentinel Errors

```go
import (
    "database/sql"
    "errors"
    "io"
)

// Checking for EOF when reading
func readAll(r io.Reader) ([]byte, error) {
    var result []byte
    buf := make([]byte, 1024)

    for {
        n, err := r.Read(buf)
        result = append(result, buf[:n]...)

        if errors.Is(err, io.EOF) {
            break  // EOF is expected, not a failure
        }
        if err != nil {
            return nil, fmt.Errorf("reading: %w", err)
        }
    }
    return result, nil
}

// Checking for "no rows" in database queries
func getUserByEmail(db *sql.DB, email string) (User, error) {
    var u User
    err := db.QueryRow("SELECT id, name FROM users WHERE email = ?", email).
        Scan(&u.ID, &u.Name)

    if errors.Is(err, sql.ErrNoRows) {
        return User{}, fmt.Errorf("user with email %s not found: %w", email, err)
    }
    if err != nil {
        return User{}, fmt.Errorf("querying user by email %s: %w", email, err)
    }
    return u, nil
}
```

### Creating Your Own Sentinel Errors

```go
package order

import "errors"

var (
    ErrNotFound      = errors.New("order not found")
    ErrAlreadyPaid   = errors.New("order already paid")
    ErrInvalidAmount = errors.New("invalid order amount")
    ErrCancelled     = errors.New("order cancelled")
)

func Pay(orderID string, amount float64) error {
    order, err := findOrder(orderID)
    if err != nil {
        return fmt.Errorf("paying order %s: %w", orderID, err)
    }

    if order.Status == "paid" {
        return fmt.Errorf("paying order %s: %w", orderID, ErrAlreadyPaid)
    }

    if amount <= 0 {
        return fmt.Errorf("paying order %s with amount %.2f: %w", orderID, amount, ErrInvalidAmount)
    }

    // ... process payment
    return nil
}
```

Callers can then check:

```go
err := order.Pay("ord-123", 50.00)
if errors.Is(err, order.ErrAlreadyPaid) {
    // idempotent -- treat as success
    log.Println("Order already paid, skipping")
} else if errors.Is(err, order.ErrInvalidAmount) {
    http.Error(w, "Bad amount", http.StatusBadRequest)
} else if err != nil {
    http.Error(w, "Internal error", http.StatusInternalServerError)
}
```

### When to Use Sentinel Errors vs Custom Types

| Use Sentinel Errors When | Use Custom Error Types When |
|---|---|
| The condition is simple and well-defined | You need to carry structured data |
| Callers only need to know *what* happened | Callers need to know *details* (which field, which ID) |
| There are a small number of known conditions | The error needs context-specific fields |
| Example: `ErrNotFound`, `ErrUnauthorized` | Example: `ValidationError{Field, Message}` |

### Design Pattern: Combining Sentinels and Custom Types

A common pattern is to use sentinel errors as the *wrapped* cause inside a custom type:

```go
var ErrNotFound = errors.New("not found")

type ResourceError struct {
    Resource string
    ID       string
    Err      error
}

func (e *ResourceError) Error() string {
    return fmt.Sprintf("%s %s: %v", e.Resource, e.ID, e.Err)
}

func (e *ResourceError) Unwrap() error {
    return e.Err
}

// Usage
func getUser(id string) (User, error) {
    // ...
    if notInDB {
        return User{}, &ResourceError{
            Resource: "user",
            ID:       id,
            Err:      ErrNotFound,
        }
    }
    // ...
}

// Callers get the best of both worlds
err := getUser("abc-123")

// Check the condition
if errors.Is(err, ErrNotFound) {
    // ...
}

// Extract the details
var resErr *ResourceError
if errors.As(err, &resErr) {
    fmt.Println(resErr.Resource, resErr.ID)
}
```

### Anti-Pattern: Comparing Error Strings

Never do this:

```go
// BAD -- fragile, breaks if the message changes
if err.Error() == "not found" {
    // ...
}

// BAD -- fragile, breaks if wrapping is added
if strings.Contains(err.Error(), "not found") {
    // ...
}
```

Always use `errors.Is()` or `errors.As()`. Error messages are for humans. Sentinel values and types are for programs.

---

## 7. panic and recover

### What Is panic?

`panic` is Go's mechanism for truly unrecoverable errors -- situations where the program cannot continue in any meaningful way:

```go
func mustParseURL(raw string) *url.URL {
    u, err := url.Parse(raw)
    if err != nil {
        panic(fmt.Sprintf("invalid URL %q: %v", raw, err))
    }
    return u
}
```

When a panic occurs:
1. The current function stops executing.
2. Any deferred functions in the current goroutine run.
3. The goroutine's call stack unwinds, running deferred functions at each level.
4. If no `recover` is called, the program crashes with a stack trace.

### WHY panic Should Be Rare

Go's philosophy is that most errors are expected, recoverable, and should be handled with error values. `panic` is reserved for bugs and truly impossible situations:

**Legitimate uses of panic:**
- Programming errors (index out of bounds, nil pointer dereference -- Go does this automatically)
- Initialization that cannot fail in a correctly configured environment
- `Must*` functions that wrap an error-returning function for use in `var` blocks

```go
// Standard library example: regexp.MustCompile
// Panics if the regex is invalid -- because an invalid regex literal is a bug
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

// template.Must -- same pattern
var tmpl = template.Must(template.ParseFiles("index.html"))
```

**Never use panic for:**
- File not found (the file might not exist -- that is expected)
- Network errors (networks are unreliable -- that is expected)
- Invalid user input (users make mistakes -- that is expected)
- Database errors (connections fail -- that is expected)

### recover -- Catching Panics

`recover` is a built-in function that regains control of a panicking goroutine. It **only works inside a deferred function**:

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic in division: %v", r)
        }
    }()

    return a / b, nil  // if b is 0, this panics
}

result, err := safeDiv(10, 0)
// result = 0, err = "panic in division: runtime error: integer divide by zero"
```

### Real-World Use: HTTP Server Panic Recovery

The most common legitimate use of `recover` is in HTTP servers, where a panic in one request handler should not crash the entire server:

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                // Log the panic with stack trace
                log.Printf("PANIC: %v\n%s", rec, debug.Stack())

                // Return 500 to the client
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    // Wrap all handlers with panic recovery
    log.Fatal(http.ListenAndServe(":8080", recoveryMiddleware(mux)))
}
```

Go's standard `net/http` server actually has built-in panic recovery per request, but many frameworks add their own for better logging and response formatting.

### Comparison: Go panic/recover vs JavaScript throw/catch

While they look similar on the surface, they are philosophically different:

```javascript
// JavaScript -- throw/catch is the PRIMARY error mechanism
function divide(a, b) {
    if (b === 0) throw new Error("division by zero");
    return a / b;
}

try {
    const result = divide(10, 0);
} catch (err) {
    console.error(err.message);
}
```

```go
// Go -- panic is the LAST RESORT error mechanism
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")  // return error, don't panic
    }
    return a / b, nil
}

result, err := divide(10, 0)
if err != nil {
    log.Println(err)
}
```

| Aspect | Go panic/recover | JS throw/catch |
|---|---|---|
| Role | Emergency escape hatch | Primary error mechanism |
| Frequency | Rare (bugs, init) | Everywhere |
| Philosophy | "This should never happen" | "This might happen" |
| Performance | Expensive (stack unwinding) | Optimized by engines |
| Cross-goroutine | Cannot recover panics from other goroutines | Promises catch across async boundaries |
| Convention | Use error values instead | Use try/catch for everything |

### A Critical Rule: Panics Do Not Cross Goroutine Boundaries

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)  // This will NOT catch the panic below
        }
    }()

    go func() {
        panic("crash!")  // This goroutine panics
    }()

    time.Sleep(time.Second)
    // The program CRASHES -- the recover in main cannot catch a panic
    // from a different goroutine
}
```

Each goroutine must have its own recovery mechanism:

```go
func main() {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Recovered in goroutine:", r)
            }
        }()
        panic("crash!")
    }()

    time.Sleep(time.Second)
    fmt.Println("Main continues")
}
```

---

## 8. Error Handling Patterns

### Pattern 1: Early Returns (The Guard Clause)

The most fundamental Go error handling pattern. Check for errors immediately and return early:

```go
// BAD -- deeply nested, hard to read
func processFile(path string) error {
    file, err := os.Open(path)
    if err == nil {
        defer file.Close()
        data, err := io.ReadAll(file)
        if err == nil {
            var result Result
            err = json.Unmarshal(data, &result)
            if err == nil {
                err = saveResult(result)
                if err == nil {
                    log.Println("Success")
                } else {
                    return fmt.Errorf("saving: %w", err)
                }
            } else {
                return fmt.Errorf("parsing: %w", err)
            }
        } else {
            return fmt.Errorf("reading: %w", err)
        }
    } else {
        return fmt.Errorf("opening: %w", err)
    }
    return nil
}

// GOOD -- flat, easy to follow
func processFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("opening %s: %w", path, err)
    }
    defer file.Close()

    data, err := io.ReadAll(file)
    if err != nil {
        return fmt.Errorf("reading %s: %w", path, err)
    }

    var result Result
    if err := json.Unmarshal(data, &result); err != nil {
        return fmt.Errorf("parsing %s: %w", path, err)
    }

    if err := saveResult(result); err != nil {
        return fmt.Errorf("saving result from %s: %w", path, err)
    }

    log.Println("Success")
    return nil
}
```

This is the Go equivalent of "fail fast." The happy path flows straight down the left edge of the function.

### Pattern 2: Error Wrapping at Each Level

Each function in the call chain adds its own context. The convention is to describe what the function was trying to do, not what went wrong (the wrapped error says what went wrong):

```go
// Repository layer
func (r *UserRepo) GetByID(id string) (User, error) {
    row := r.db.QueryRow("SELECT name, email FROM users WHERE id = ?", id)
    var u User
    if err := row.Scan(&u.Name, &u.Email); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return User{}, fmt.Errorf("user repo get by id %s: %w", id, ErrNotFound)
        }
        return User{}, fmt.Errorf("user repo get by id %s: %w", id, err)
    }
    u.ID = id
    return u, nil
}

// Service layer
func (s *UserService) GetProfile(userID string) (Profile, error) {
    user, err := s.repo.GetByID(userID)
    if err != nil {
        return Profile{}, fmt.Errorf("getting profile for user %s: %w", userID, err)
    }

    orders, err := s.orderRepo.GetByUserID(userID)
    if err != nil {
        return Profile{}, fmt.Errorf("getting orders for user %s: %w", userID, err)
    }

    return Profile{User: user, Orders: orders}, nil
}

// HTTP handler layer
func (h *Handler) GetProfile(w http.ResponseWriter, r *http.Request) {
    userID := r.PathValue("id")

    profile, err := h.userService.GetProfile(userID)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        log.Printf("ERROR: getting profile: %v", err)
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(profile)
}
```

Notice the flow:
- The **repository** wraps with database-level context.
- The **service** wraps with business-logic context.
- The **handler** makes the final decision: which errors are client-facing (404) vs internal (500).

### Pattern 3: Centralized HTTP Error Handling

Instead of repeating error-to-HTTP-status logic in every handler, use a pattern that returns errors from handlers:

```go
// Define a handler type that returns an error
type appHandler func(w http.ResponseWriter, r *http.Request) error

// Wrapper that converts errors to HTTP responses
func handleErrors(h appHandler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        err := h(w, r)
        if err == nil {
            return
        }

        // Log the full error for debugging
        log.Printf("ERROR: %s %s: %v", r.Method, r.URL.Path, err)

        // Convert to HTTP response
        var httpErr *HTTPError
        if errors.As(err, &httpErr) {
            http.Error(w, httpErr.Message, httpErr.StatusCode)
            return
        }

        if errors.Is(err, ErrNotFound) {
            http.Error(w, "Not found", http.StatusNotFound)
            return
        }

        if errors.Is(err, ErrUnauthorized) {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Default: 500
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
    }
}

// Now handlers are clean -- they just return errors
func getUser(w http.ResponseWriter, r *http.Request) error {
    id := r.PathValue("id")

    user, err := userService.Get(id)
    if err != nil {
        return fmt.Errorf("getting user %s: %w", id, err)
    }

    return json.NewEncoder(w).Encode(user)
}

// Registration
mux.HandleFunc("GET /users/{id}", handleErrors(getUser))
```

### Comparison: Go's Centralized Error Handling vs Express.js Error Middleware

```javascript
// Express.js -- error middleware
app.get('/users/:id', async (req, res, next) => {
    try {
        const user = await userService.get(req.params.id);
        res.json(user);
    } catch (err) {
        next(err);  // pass to error middleware
    }
});

// Centralized error handler
app.use((err, req, res, next) => {
    console.error('ERROR:', err);

    if (err instanceof NotFoundError) {
        return res.status(404).json({ error: 'Not found' });
    }
    if (err instanceof ValidationError) {
        return res.status(400).json({ error: err.message });
    }

    res.status(500).json({ error: 'Internal server error' });
});
```

```go
// Go equivalent -- see handleErrors pattern above
// The key difference: Go handlers return errors explicitly
// Express handlers call next(err) or throw and hope the middleware catches it
```

The patterns are structurally similar, but Go's is more explicit -- the handler signature `func(...) error` makes it obvious that errors are expected.

### Pattern 4: Cleanup on Error with defer

```go
func createUser(db *sql.DB, u User) error {
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("starting transaction: %w", err)
    }

    // Rollback if we return an error; commit if we return nil
    defer func() {
        if err != nil {
            tx.Rollback()
        }
    }()

    _, err = tx.Exec("INSERT INTO users (name, email) VALUES (?, ?)", u.Name, u.Email)
    if err != nil {
        return fmt.Errorf("inserting user: %w", err)
    }

    _, err = tx.Exec("INSERT INTO audit_log (action) VALUES (?)", "user_created")
    if err != nil {
        return fmt.Errorf("inserting audit log: %w", err)
    }

    if err = tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }

    return nil
}
```

Note the use of the named return variable `err`. The deferred function captures it by reference, so it sees the final value of `err` when the function returns. This is a common and idiomatic pattern for transaction rollback.

### Pattern 5: Retry with Backoff

```go
func fetchWithRetry(url string, maxRetries int) ([]byte, error) {
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(attempt*attempt) * 100 * time.Millisecond
            time.Sleep(backoff)
            log.Printf("Retry %d/%d for %s", attempt, maxRetries, url)
        }

        resp, err := http.Get(url)
        if err != nil {
            lastErr = fmt.Errorf("attempt %d: %w", attempt, err)
            continue
        }
        defer resp.Body.Close()

        if resp.StatusCode >= 500 {
            lastErr = fmt.Errorf("attempt %d: server error %d", attempt, resp.StatusCode)
            continue
        }

        body, err := io.ReadAll(resp.Body)
        if err != nil {
            lastErr = fmt.Errorf("attempt %d: reading body: %w", attempt, err)
            continue
        }

        return body, nil
    }

    return nil, fmt.Errorf("all %d retries failed for %s: %w", maxRetries, url, lastErr)
}
```

### Pattern 6: Collecting Multiple Errors

When you need to validate multiple things and report all failures, not just the first:

```go
func validateOrder(o Order) error {
    var errs []error

    if o.CustomerID == "" {
        errs = append(errs, errors.New("customer ID is required"))
    }
    if o.Total <= 0 {
        errs = append(errs, fmt.Errorf("total must be positive, got %.2f", o.Total))
    }
    if len(o.Items) == 0 {
        errs = append(errs, errors.New("order must have at least one item"))
    }

    // errors.Join (Go 1.20+) combines multiple errors into one
    return errors.Join(errs...)
}

// Usage
err := validateOrder(order)
if err != nil {
    // err.Error() returns all messages joined by newlines
    fmt.Println(err)
    // Output:
    // customer ID is required
    // total must be positive, got -5.00
    // order must have at least one item

    // And you can still check for individual errors
    if errors.Is(err, someSpecificErr) {
        // ...
    }
}
```

---

## 9. Error Handling Best Practices

### 1. Never Ignore Errors

```go
// BAD -- silently ignoring the error
data, _ := json.Marshal(user)

// BAD -- calling a function and not checking
os.Remove(tempFile)

// GOOD -- if you truly don't need the error, document WHY
data, err := json.Marshal(user)
if err != nil {
    // json.Marshal only fails on unmarshalable types (channels, funcs).
    // User contains only basic types, so this should never fail.
    panic(fmt.Sprintf("unexpected marshal error: %v", err))
}

// GOOD -- if cleanup failure is acceptable, log it
if err := os.Remove(tempFile); err != nil {
    log.Printf("warning: failed to remove temp file %s: %v", tempFile, err)
}
```

Go's compiler will warn you about unused variables, but `_` suppresses this. Use `_` for errors only when you have a clear reason.

### 2. Wrap Errors with Context, But Wrap Only Once

```go
// BAD -- no context, caller has no idea what failed
func getUser(id string) (User, error) {
    return db.FindUser(id)  // error: "connection refused" -- connection to WHAT?
}

// BAD -- wrapping the same error multiple times at the same level
func getUser(id string) (User, error) {
    u, err := db.FindUser(id)
    if err != nil {
        err = fmt.Errorf("database error: %w", err)
        return User{}, fmt.Errorf("getting user: %w", err)  // double-wrapped
    }
    return u, nil
}

// GOOD -- one wrap with clear context
func getUser(id string) (User, error) {
    u, err := db.FindUser(id)
    if err != nil {
        return User{}, fmt.Errorf("getting user %s from database: %w", id, err)
    }
    return u, nil
}
```

### 3. Handle Errors Only Once

An error should be handled (logged, returned, recovered from) exactly once. If you log it and return it, the caller might log it again.

```go
// BAD -- handling the same error twice
func processOrder(id string) error {
    order, err := getOrder(id)
    if err != nil {
        log.Printf("ERROR: failed to get order: %v", err) // handling #1: logging
        return fmt.Errorf("processing order: %w", err)     // handling #2: returning
        // The caller will probably also log this, resulting in duplicate log entries
    }
    // ...
}

// GOOD -- either log OR return, not both
func processOrder(id string) error {
    order, err := getOrder(id)
    if err != nil {
        return fmt.Errorf("processing order %s: %w", id, err) // let the caller decide
    }
    // ...
}

// The top-level caller (HTTP handler, main, etc.) does the logging
func handleOrder(w http.ResponseWriter, r *http.Request) {
    err := processOrder(r.PathValue("id"))
    if err != nil {
        log.Printf("ERROR: %v", err)  // logged once, at the top
        http.Error(w, "Internal error", 500)
    }
}
```

### 4. Use errors.Is and errors.As, Not Type Assertions

```go
// BAD -- direct type assertion doesn't traverse wrapped errors
if pathErr, ok := err.(*os.PathError); ok {
    // This misses wrapped PathErrors!
}

// GOOD -- errors.As traverses the whole chain
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

### 5. Error Messages: Lowercase, No Punctuation

Go convention is that error strings are lowercase and do not end with punctuation, because they are often wrapped with additional context:

```go
// BAD
return errors.New("Failed to connect to database.")
// When wrapped: "starting server: Failed to connect to database."  -- awkward capitalization

// GOOD
return errors.New("failed to connect to database")
// When wrapped: "starting server: failed to connect to database"  -- reads naturally
```

### 6. Don't Use panic for Expected Errors

```go
// BAD -- panic for an expected condition
func getUser(id string) User {
    u, err := db.Find(id)
    if err != nil {
        panic(err)  // don't do this -- DB errors are expected
    }
    return u
}

// GOOD -- return the error
func getUser(id string) (User, error) {
    u, err := db.Find(id)
    if err != nil {
        return User{}, fmt.Errorf("getting user %s: %w", id, err)
    }
    return u, nil
}
```

### 7. Create Package-Level Sentinel Errors for Your API Contract

```go
package auth

var (
    ErrInvalidCredentials = errors.New("invalid credentials")
    ErrTokenExpired       = errors.New("token expired")
    ErrInsufficientScope  = errors.New("insufficient scope")
)
```

These become part of your package's public API. Document them and keep them stable.

### 8. Use Custom Error Types When Callers Need Structured Data

```go
// When the caller needs more than just "what went wrong"
type RateLimitError struct {
    Limit      int
    RetryAfter time.Duration
}

func (e *RateLimitError) Error() string {
    return fmt.Sprintf("rate limited (limit: %d, retry after: %s)", e.Limit, e.RetryAfter)
}

// Callers can extract the retry duration
var rlErr *RateLimitError
if errors.As(err, &rlErr) {
    time.Sleep(rlErr.RetryAfter)
    // retry...
}
```

### Summary Table: Choosing Your Error Strategy

| Scenario | Strategy | Example |
|---|---|---|
| Simple failure, fixed message | `errors.New()` | `errors.New("connection closed")` |
| Failure with dynamic data | `fmt.Errorf()` | `fmt.Errorf("user %s not found", id)` |
| Wrapping a lower-level error | `fmt.Errorf() + %w` | `fmt.Errorf("loading config: %w", err)` |
| Known conditions callers should check | Sentinel errors | `var ErrNotFound = errors.New(...)` |
| Structured error data for callers | Custom error types | `&ValidationError{Field: "email", ...}` |
| Multiple validation failures | `errors.Join()` | `errors.Join(err1, err2, err3)` |
| Truly unrecoverable bug | `panic()` | `panic("unreachable code")` |
| Protecting a goroutine from panics | `defer + recover` | HTTP panic recovery middleware |

---

## 10. Key Takeaways

1. **Errors are values.** Go's most fundamental design choice for error handling. No exceptions, no hidden control flow. Every error is visible in the code.

2. **The `error` interface is intentionally minimal.** One method: `Error() string`. This simplicity enables maximum composability. Any type can be an error.

3. **Always wrap errors with context using `%w`.** Build error chains that tell a complete story: "starting server: loading TLS config: reading cert from /path: file not found."

4. **Use `errors.Is()` and `errors.As()` to inspect error chains.** Never compare error strings or use direct type assertions on potentially wrapped errors.

5. **Handle errors exactly once.** Either log it or return it, but not both. Let the top-level caller (HTTP handler, main function) do the logging.

6. **Sentinel errors define your API contract.** Export them as package-level variables when callers need to check for specific conditions.

7. **Custom error types carry structured data.** Use them when callers need more than a string message (status codes, retry durations, field names).

8. **`panic` is for bugs, not expected failures.** Reserve it for "this should be impossible" situations. Use `recover` only at boundaries (HTTP handlers, goroutine launchers).

9. **The `if err != nil` pattern is verbose by design.** It forces you to handle every error explicitly. This is a feature -- it makes Go programs more reliable.

10. **Coming from Node.js, the biggest shift is mindset.** You are moving from "handle errors at a distance with try/catch" to "handle errors right here, right now, every time." It is more typing. It is also fewer production surprises.

---

## 11. Practice Exercises

### Exercise 1: Basic Error Handling

Write a function `readJSON(path string, v any) error` that:
- Opens a file at the given path.
- Reads and decodes JSON into `v`.
- Returns wrapped errors with context at each step.
- Returns a sentinel `ErrNotFound` if the file does not exist (hint: use `errors.Is(err, fs.ErrNotExist)`).

### Exercise 2: Custom Error Type

Create a `ValidationError` type with fields `Field`, `Value`, and `Message`. Write a `validateEmail(email string)` function that returns a `*ValidationError` when the input is invalid. Write a caller that uses `errors.As` to extract the structured error.

### Exercise 3: Error Wrapping Chain

Build a three-layer application:
- **Repository**: `func GetUser(id string) (User, error)` -- returns `ErrNotFound` or database errors.
- **Service**: `func GetUserProfile(id string) (Profile, error)` -- calls the repository, wraps the error.
- **Handler**: `func HandleGetProfile(w http.ResponseWriter, r *http.Request)` -- calls the service, converts `ErrNotFound` to 404 and everything else to 500.

Verify that `errors.Is(err, ErrNotFound)` works through all layers of wrapping.

### Exercise 4: Panic Recovery Middleware

Write an HTTP middleware that:
- Recovers from panics in handlers.
- Logs the panic value and stack trace.
- Returns a 500 response to the client.
- Test it with a handler that deliberately panics.

### Exercise 5: Retry with Error Classification

Write a `retry(maxAttempts int, fn func() error) error` function that:
- Calls `fn` up to `maxAttempts` times.
- Only retries if the error is a `*TemporaryError` (create this custom type with a `Retry bool` field).
- Returns immediately for non-retryable errors.
- Collects all errors using `errors.Join` and returns them if all attempts fail.

### Exercise 6: Port from Node.js

Take this Node.js code and rewrite it in idiomatic Go:

```javascript
async function processOrder(orderId) {
    try {
        const order = await db.getOrder(orderId);
        if (!order) throw new NotFoundError(`Order ${orderId}`);

        const payment = await paymentService.charge(order.total);
        if (!payment.success) {
            throw new PaymentError(payment.declineReason, {
                cause: new Error(`charge failed for order ${orderId}`)
            });
        }

        await db.updateOrderStatus(orderId, 'paid');
        return { order, payment };
    } catch (err) {
        if (err instanceof NotFoundError) {
            throw err; // re-throw
        }
        if (err instanceof PaymentError) {
            console.error('Payment failed:', err.message, err.cause);
            throw err;
        }
        throw new Error(`processing order ${orderId} failed`, { cause: err });
    }
}
```

Your Go version should use sentinel errors, custom error types, `errors.Is`, `errors.As`, and proper wrapping.

---

**Next Chapter:** [Chapter 9: Concurrency with Goroutines & Channels](./09-concurrency.md) -- where Go's true superpower begins.
