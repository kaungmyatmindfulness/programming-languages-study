# Chapter 57: Secure Coding in Go

## Table of Contents

1. [Introduction](#introduction)
2. [Go's Built-in Security Advantages](#gos-built-in-security-advantages)
3. [OWASP Top 10 in the Go Context](#owasp-top-10-in-the-go-context)
4. [Cryptography in Go](#cryptography-in-go)
5. [TLS Configuration](#tls-configuration)
6. [JWT Implementation and Pitfalls](#jwt-implementation-and-pitfalls)
7. [Secret Management](#secret-management)
8. [Input Validation and Sanitization](#input-validation-and-sanitization)
9. [Security Headers Middleware](#security-headers-middleware)
10. [Rate Limiting to Prevent Abuse](#rate-limiting-to-prevent-abuse)
11. [Dependency Vulnerability Scanning](#dependency-vulnerability-scanning)
12. [gosec Static Analysis](#gosec-static-analysis)
13. [Secure Coding Checklist](#secure-coding-checklist)
14. [Common Go-Specific Security Mistakes](#common-go-specific-security-mistakes)
15. [Summary](#summary)

---

## Introduction

Security is not a feature you bolt on after the fact -- it must be woven into every layer of
your application from day one. Go was designed with simplicity, correctness, and safety in
mind, which gives it a strong foundation for building secure systems. However, the language
alone cannot prevent all classes of vulnerabilities. Developers must understand the threat
landscape, follow secure coding practices, and leverage the robust standard library and
tooling that Go provides.

This chapter is a comprehensive guide to writing secure Go code. We will cover everything
from the language-level protections Go offers, through the OWASP Top 10 mapped to Go-specific
mitigations, to advanced topics like cryptography, TLS, JWT handling, and operational
security practices such as secret management and dependency scanning.

---

## Go's Built-in Security Advantages

Before diving into specific vulnerabilities, it is worth understanding why Go is considered
a relatively secure language compared to C/C++ and even some other high-level languages.

### Memory Safety

Go is a memory-safe language. The runtime manages memory allocation and deallocation through
garbage collection, which eliminates entire classes of vulnerabilities:

| Vulnerability Class        | C/C++ | Go     | Why                                      |
|---------------------------|-------|--------|------------------------------------------|
| Buffer overflow            | Yes   | No     | Bounds checking on all slice/array access |
| Use-after-free             | Yes   | No     | Garbage collector manages lifetimes       |
| Double free                | Yes   | No     | No manual memory management               |
| Dangling pointer           | Yes   | No     | GC keeps referenced memory alive          |
| Uninitialized memory read  | Yes   | No     | All variables are zero-initialized        |
| Format string attacks      | Yes   | No     | No printf-style format string execution   |

```go
package main

import "fmt"

func main() {
    // Go prevents buffer overflows at runtime
    s := make([]int, 5)

    // This will panic with "runtime error: index out of range [10]
    // with length 5" -- it will NOT silently corrupt memory
    // s[10] = 42

    // Safe way: always check bounds or use append
    s = append(s, 42)
    fmt.Println(s) // [0 0 0 0 0 42]
}
```

### No Pointer Arithmetic

Unlike C/C++, Go does not allow pointer arithmetic. You cannot increment a pointer to walk
through memory, which prevents an enormous class of memory corruption bugs:

```go
package main

import "fmt"

func main() {
    x := 42
    p := &x

    // This is ILLEGAL in Go -- will not compile:
    // p++
    // p = p + 1
    // *(p + offset) = value

    // You can only dereference or assign pointers
    fmt.Println(*p) // 42
}
```

> **Note:** The `unsafe` package does allow pointer arithmetic, but its use is strongly
> discouraged and is flagged by security tools. Any code using `unsafe` should be carefully
> audited.

### Strong Static Typing

Go's type system catches many bugs at compile time that would be runtime vulnerabilities in
dynamically typed languages:

```go
package main

// The compiler enforces type safety -- you cannot accidentally
// mix up a UserID with a SessionToken

type UserID int64
type SessionToken string

func GetUser(id UserID) {
    // ...
}

func main() {
    var token SessionToken = "abc123"

    // Compile error: cannot use token (variable of type SessionToken)
    // as UserID value in argument to GetUser
    // GetUser(token)

    var id UserID = 42
    GetUser(id) // OK
}
```

### Zero Values

Every type in Go has a zero value, which means variables are never uninitialized:

```go
package main

import "fmt"

func main() {
    var i int       // 0
    var s string    // ""
    var b bool      // false
    var p *int      // nil
    var sl []int    // nil (empty slice)
    var m map[string]int // nil (empty map)

    fmt.Println(i, s, b, p, sl, m)
    // 0  false <nil> [] map[]
}
```

This prevents undefined behavior from reading uninitialized memory -- a common source of
information leaks in C/C++.

### Built-in Race Detector

Go ships with a built-in race detector that can catch data races -- a significant source of
security vulnerabilities in concurrent programs:

```bash
# Run tests with race detection enabled
go test -race ./...

# Run your program with race detection
go run -race main.go
```

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // BAD: data race -- the race detector will catch this
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE
        }()
    }
    wg.Wait()
    fmt.Println(counter) // Unpredictable result

    // GOOD: use synchronization
    counter = 0
    var mu sync.Mutex

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println(counter) // Always 1000
}
```

> **Tip:** Always run `go test -race ./...` in your CI pipeline. The race detector has
> essentially zero false positives -- if it reports a race, you have a real bug.

### Compilation to Static Binaries

Go compiles to statically linked binaries by default. This means:

- No shared library injection attacks (LD_PRELOAD attacks)
- No DLL hijacking
- Smaller attack surface at runtime (no interpreter, no VM)
- Reproducible deployments

```bash
# Build a fully static binary
CGO_ENABLED=0 go build -o myapp .

# Verify it's static
file myapp
# myapp: ELF 64-bit LSB executable, x86-64, statically linked, ...
```

---

## OWASP Top 10 in the Go Context

The OWASP Top 10 is the industry standard awareness document for web application security.
Let's examine each category through the lens of Go development.

### A03:2021 -- Injection (SQL Injection Prevention)

SQL injection remains one of the most dangerous and common vulnerabilities. Go's
`database/sql` package provides parameterized queries as the primary defense.

#### The Vulnerable Way (NEVER DO THIS)

```go
package main

import (
    "database/sql"
    "fmt"
    "net/http"

    _ "github.com/lib/pq"
)

// VULNERABLE: String concatenation in SQL queries
func unsafeHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        username := r.URL.Query().Get("username")

        // DANGER: SQL injection vulnerability!
        // An attacker can send: username=' OR '1'='1' --
        query := fmt.Sprintf("SELECT id, email FROM users WHERE username = '%s'", username)

        rows, err := db.Query(query)
        if err != nil {
            http.Error(w, "Database error", http.StatusInternalServerError)
            return
        }
        defer rows.Close()

        // ... process rows
    }
}
```

An attacker could send `username=' OR '1'='1' --` to retrieve all users, or even
`username='; DROP TABLE users; --` to destroy your data.

#### The Secure Way (Parameterized Queries)

```go
package main

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"

    _ "github.com/lib/pq"
)

// SECURE: Parameterized queries prevent SQL injection
func safeHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        username := r.URL.Query().Get("username")

        // The $1 placeholder is handled by the database driver.
        // The value is sent separately from the query -- the database
        // engine will NEVER interpret it as SQL code.
        var id int
        var email string
        err := db.QueryRow(
            "SELECT id, email FROM users WHERE username = $1",
            username,
        ).Scan(&id, &email)

        if err == sql.ErrNoRows {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        if err != nil {
            log.Printf("Database error: %v", err)
            http.Error(w, "Internal server error", http.StatusInternalServerError)
            return
        }

        json.NewEncoder(w).Encode(map[string]interface{}{
            "id":    id,
            "email": email,
        })
    }
}

// SECURE: Using prepared statements for repeated queries
func setupPreparedStatements(db *sql.DB) (*sql.Stmt, error) {
    // Prepared statements are parsed once and executed many times.
    // They are inherently safe against SQL injection.
    stmt, err := db.Prepare("SELECT id, email FROM users WHERE username = $1 AND active = $2")
    if err != nil {
        return nil, err
    }
    return stmt, nil
}

func safeHandlerWithStmt(stmt *sql.Stmt) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        username := r.URL.Query().Get("username")

        var id int
        var email string
        err := stmt.QueryRow(username, true).Scan(&id, &email)
        if err != nil {
            http.Error(w, "Not found", http.StatusNotFound)
            return
        }

        json.NewEncoder(w).Encode(map[string]interface{}{
            "id":    id,
            "email": email,
        })
    }
}
```

#### Safe Dynamic Queries with Query Builders

Sometimes you need dynamic queries (e.g., optional filters). Use a query builder instead
of string concatenation:

```go
package main

import (
    "database/sql"
    "fmt"
    "strings"
)

// SafeQueryBuilder builds parameterized queries dynamically
type SafeQueryBuilder struct {
    conditions []string
    args       []interface{}
    argIndex   int
}

func NewSafeQueryBuilder() *SafeQueryBuilder {
    return &SafeQueryBuilder{argIndex: 1}
}

func (b *SafeQueryBuilder) AddCondition(column string, value interface{}) {
    // Column names are hardcoded, only values are parameterized
    b.conditions = append(b.conditions, fmt.Sprintf("%s = $%d", column, b.argIndex))
    b.args = append(b.args, value)
    b.argIndex++
}

func (b *SafeQueryBuilder) Build(baseQuery string) (string, []interface{}) {
    if len(b.conditions) == 0 {
        return baseQuery, nil
    }
    return baseQuery + " WHERE " + strings.Join(b.conditions, " AND "), b.args
}

func searchUsers(db *sql.DB, name, email, role string) (*sql.Rows, error) {
    builder := NewSafeQueryBuilder()

    if name != "" {
        builder.AddCondition("name", name)
    }
    if email != "" {
        builder.AddCondition("email", email)
    }
    if role != "" {
        builder.AddCondition("role", role)
    }

    query, args := builder.Build("SELECT id, name, email FROM users")
    return db.Query(query, args...)
}
```

> **Warning:** Never use `fmt.Sprintf` to build SQL queries with user input. Always use
> parameterized queries (`$1`, `?`, or named parameters depending on your driver).

---

### A03:2021 -- Injection (XSS Prevention)

Cross-Site Scripting (XSS) occurs when an attacker injects malicious scripts into web
pages viewed by other users. Go's `html/template` package provides automatic context-aware
escaping.

#### The Vulnerable Way (text/template)

```go
package main

import (
    "net/http"
    "text/template" // DANGER: text/template does NOT escape HTML
)

// VULNERABLE: text/template does not escape HTML entities
func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")

    // If name = <script>alert('XSS')</script>, the script WILL execute
    tmpl := template.Must(template.New("page").Parse(`
        <html><body>
            <h1>Hello, {{.Name}}!</h1>
        </body></html>
    `))

    tmpl.Execute(w, map[string]string{"Name": name})
}
```

#### The Secure Way (html/template)

```go
package main

import (
    "html/template" // SECURE: html/template auto-escapes based on context
    "net/http"
)

// SECURE: html/template automatically escapes HTML entities
func secureHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")

    // If name = <script>alert('XSS')</script>
    // It will be rendered as: &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;
    // The browser will display the text, NOT execute it
    tmpl := template.Must(template.New("page").Parse(`
        <html><body>
            <h1>Hello, {{.Name}}!</h1>
        </body></html>
    `))

    tmpl.Execute(w, map[string]string{"Name": name})
}
```

#### Context-Aware Escaping

`html/template` is context-aware -- it applies different escaping depending on where the
value appears in the HTML:

```go
package main

import (
    "html/template"
    "os"
)

func main() {
    tmpl := template.Must(template.New("context").Parse(`
        <!-- In HTML body: HTML entity encoding -->
        <p>{{.UserInput}}</p>

        <!-- In an attribute: attribute encoding -->
        <div title="{{.UserInput}}">hover me</div>

        <!-- In a URL: URL encoding -->
        <a href="/search?q={{.UserInput}}">search</a>

        <!-- In JavaScript: JS string encoding -->
        <script>
            var name = "{{.UserInput}}";
        </script>

        <!-- In CSS: CSS encoding -->
        <style>
            .user-content { color: {{.Color}}; }
        </style>
    `))

    data := map[string]string{
        "UserInput": `<script>alert("XSS")</script>`,
        "Color":     "red",
    }

    tmpl.Execute(os.Stdout, data)
}
```

#### When You Need Raw HTML (Use With Extreme Caution)

```go
package main

import (
    "html/template"
    "os"
)

func main() {
    tmpl := template.Must(template.New("raw").Parse(`
        <!-- This will be escaped (safe) -->
        <div>{{.UnsafeHTML}}</div>

        <!-- This will NOT be escaped (dangerous -- only use for trusted content) -->
        <div>{{.TrustedHTML}}</div>
    `))

    data := map[string]interface{}{
        "UnsafeHTML": `<b>Bold</b>`,                        // Will be escaped
        "TrustedHTML": template.HTML(`<b>Bold</b>`),        // Will NOT be escaped
    }

    tmpl.Execute(os.Stdout, data)
    // Output:
    // <div>&lt;b&gt;Bold&lt;/b&gt;</div>    (escaped -- safe)
    // <div><b>Bold</b></div>                (raw -- only for trusted content!)
}
```

> **Warning:** Never use `template.HTML()`, `template.JS()`, or `template.URL()` on
> user-supplied input. These types bypass the auto-escaping and should only be used for
> content you fully control and trust.

---

### A05:2021 -- Security Misconfiguration (CSRF Protection)

Cross-Site Request Forgery (CSRF) tricks a user's browser into making unwanted requests
to a site where they are authenticated. The standard defense is a per-session CSRF token.

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "html/template"
    "net/http"
    "sync"
)

// TokenStore stores CSRF tokens per session.
// In production, use a proper session store (Redis, database, etc.)
type TokenStore struct {
    mu     sync.RWMutex
    tokens map[string]string // sessionID -> csrfToken
}

func NewTokenStore() *TokenStore {
    return &TokenStore{tokens: make(map[string]string)}
}

// GenerateToken creates a cryptographically secure random CSRF token
func (ts *TokenStore) GenerateToken(sessionID string) (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", fmt.Errorf("failed to generate CSRF token: %w", err)
    }

    token := base64.URLEncoding.EncodeToString(b)

    ts.mu.Lock()
    ts.tokens[sessionID] = token
    ts.mu.Unlock()

    return token, nil
}

// ValidateToken checks if the submitted token matches the stored one
func (ts *TokenStore) ValidateToken(sessionID, token string) bool {
    ts.mu.RLock()
    defer ts.mu.RUnlock()

    storedToken, exists := ts.tokens[sessionID]
    if !exists {
        return false
    }

    // Use constant-time comparison to prevent timing attacks
    return subtleConstantTimeCompare(storedToken, token)
}

// Constant-time string comparison (prevents timing attacks)
func subtleConstantTimeCompare(a, b string) bool {
    if len(a) != len(b) {
        return false
    }
    var result byte
    for i := 0; i < len(a); i++ {
        result |= a[i] ^ b[i]
    }
    return result == 0
}

// CSRFMiddleware validates the CSRF token on state-changing requests
func CSRFMiddleware(store *TokenStore, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Skip CSRF check for safe methods (GET, HEAD, OPTIONS)
        if r.Method == http.MethodGet || r.Method == http.MethodHead || r.Method == http.MethodOptions {
            next.ServeHTTP(w, r)
            return
        }

        // Extract session ID from cookie
        sessionCookie, err := r.Cookie("session_id")
        if err != nil {
            http.Error(w, "Forbidden: no session", http.StatusForbidden)
            return
        }

        // Extract CSRF token from the request header or form field
        csrfToken := r.Header.Get("X-CSRF-Token")
        if csrfToken == "" {
            csrfToken = r.FormValue("csrf_token")
        }

        if !store.ValidateToken(sessionCookie.Value, csrfToken) {
            http.Error(w, "Forbidden: invalid CSRF token", http.StatusForbidden)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// Form template with CSRF token embedded
var formTemplate = template.Must(template.New("form").Parse(`
<!DOCTYPE html>
<html>
<body>
    <form method="POST" action="/transfer">
        <input type="hidden" name="csrf_token" value="{{.CSRFToken}}">
        <label>Amount: <input type="text" name="amount"></label>
        <label>To: <input type="text" name="recipient"></label>
        <button type="submit">Transfer</button>
    </form>
</body>
</html>
`))

func main() {
    store := NewTokenStore()
    mux := http.NewServeMux()

    mux.HandleFunc("/form", func(w http.ResponseWriter, r *http.Request) {
        // In production, get session ID from cookie/session middleware
        sessionID := "example-session-id"
        token, err := store.GenerateToken(sessionID)
        if err != nil {
            http.Error(w, "Internal error", http.StatusInternalServerError)
            return
        }

        formTemplate.Execute(w, map[string]string{
            "CSRFToken": token,
        })
    })

    mux.HandleFunc("/transfer", func(w http.ResponseWriter, r *http.Request) {
        // CSRF token already validated by middleware
        amount := r.FormValue("amount")
        recipient := r.FormValue("recipient")
        fmt.Fprintf(w, "Transferred %s to %s", amount, recipient)
    })

    // Wrap the mux with CSRF middleware
    handler := CSRFMiddleware(store, mux)

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", handler)
}
```

> **Tip:** In production, consider using the well-tested `github.com/gorilla/csrf` package
> instead of rolling your own CSRF protection. Also, set `SameSite=Strict` or
> `SameSite=Lax` on your session cookies as an additional layer of defense.

---

### A08:2021 -- Software and Data Integrity Failures (Insecure Deserialization)

Go's `encoding/json` is generally safer than deserialization in languages like Java or PHP,
but there are still pitfalls:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

// VULNERABLE: Deserializing into a map allows arbitrary keys
func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
    var data map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }

    // An attacker could send {"role": "admin", "name": "attacker"}
    // and the "role" field would be silently accepted
    fmt.Fprintf(w, "Got: %v", data)
}

// SECURE: Deserialize into a strict struct with explicit fields
type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    // Note: no "role" field -- it will be ignored even if sent
}

func secureHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest

    decoder := json.NewDecoder(r.Body)
    // DisallowUnknownFields rejects payloads with unexpected fields
    decoder.DisallowUnknownFields()

    if err := decoder.Decode(&req); err != nil {
        http.Error(w, "Bad request: "+err.Error(), http.StatusBadRequest)
        return
    }

    // Validate the parsed data
    if req.Name == "" || req.Email == "" {
        http.Error(w, "Name and email are required", http.StatusBadRequest)
        return
    }

    fmt.Fprintf(w, "Creating user: %s (%s)", req.Name, req.Email)
}

// Additional protection: limit request body size
func limitedHandler(w http.ResponseWriter, r *http.Request) {
    // Limit body to 1MB to prevent denial of service
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MB

    var req CreateUserRequest
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()

    if err := decoder.Decode(&req); err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }

    fmt.Fprintf(w, "Creating user: %s", req.Name)
}
```

> **Warning:** Never deserialize directly into `interface{}` or `map[string]interface{}`
> from untrusted input. Always use strongly typed structs. Be especially cautious with
> `encoding/gob` -- it can deserialize arbitrary Go types and should never be used with
> untrusted input.

---

### A07:2021 -- Identification and Authentication Failures (Broken Authentication)

Implementing authentication correctly is critical. Here is a secure session-based
authentication example:

```go
package main

import (
    "crypto/rand"
    "crypto/subtle"
    "database/sql"
    "encoding/hex"
    "fmt"
    "net/http"
    "time"

    "golang.org/x/crypto/bcrypt"
)

// User represents a user in the system
type User struct {
    ID           int64
    Username     string
    PasswordHash string
    CreatedAt    time.Time
}

// Session represents an authenticated session
type Session struct {
    Token     string
    UserID    int64
    ExpiresAt time.Time
    CreatedAt time.Time
}

// AuthService handles authentication operations
type AuthService struct {
    db *sql.DB
}

// Register creates a new user with a securely hashed password
func (s *AuthService) Register(username, password string) error {
    // Validate password strength
    if len(password) < 12 {
        return fmt.Errorf("password must be at least 12 characters")
    }

    // Hash the password with bcrypt (cost factor 12)
    // bcrypt automatically generates a random salt
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    if err != nil {
        return fmt.Errorf("failed to hash password: %w", err)
    }

    _, err = s.db.Exec(
        "INSERT INTO users (username, password_hash, created_at) VALUES ($1, $2, $3)",
        username, string(hash), time.Now(),
    )
    if err != nil {
        return fmt.Errorf("failed to create user: %w", err)
    }

    return nil
}

// Login authenticates a user and creates a session
func (s *AuthService) Login(username, password string) (*Session, error) {
    var user User
    err := s.db.QueryRow(
        "SELECT id, username, password_hash FROM users WHERE username = $1",
        username,
    ).Scan(&user.ID, &user.Username, &user.PasswordHash)

    if err == sql.ErrNoRows {
        // IMPORTANT: Do not reveal whether the username exists
        // Always perform bcrypt comparison to prevent timing attacks
        // that reveal valid usernames
        bcrypt.CompareHashAndPassword(
            []byte("$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"),
            []byte(password),
        )
        return nil, fmt.Errorf("invalid credentials")
    }
    if err != nil {
        return nil, fmt.Errorf("database error: %w", err)
    }

    // Compare the submitted password with the stored hash
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password)); err != nil {
        return nil, fmt.Errorf("invalid credentials")
    }

    // Generate a cryptographically secure session token
    tokenBytes := make([]byte, 32) // 256-bit token
    if _, err := rand.Read(tokenBytes); err != nil {
        return nil, fmt.Errorf("failed to generate session token: %w", err)
    }
    token := hex.EncodeToString(tokenBytes)

    // Store the session in the database
    session := &Session{
        Token:     token,
        UserID:    user.ID,
        ExpiresAt: time.Now().Add(24 * time.Hour), // 24-hour session
        CreatedAt: time.Now(),
    }

    _, err = s.db.Exec(
        "INSERT INTO sessions (token, user_id, expires_at, created_at) VALUES ($1, $2, $3, $4)",
        session.Token, session.UserID, session.ExpiresAt, session.CreatedAt,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create session: %w", err)
    }

    return session, nil
}

// ValidateSession checks if a session token is valid and not expired
func (s *AuthService) ValidateSession(token string) (*Session, error) {
    var session Session
    err := s.db.QueryRow(
        "SELECT token, user_id, expires_at FROM sessions WHERE token = $1 AND expires_at > $2",
        token, time.Now(),
    ).Scan(&session.Token, &session.UserID, &session.ExpiresAt)

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("invalid or expired session")
    }
    if err != nil {
        return nil, fmt.Errorf("database error: %w", err)
    }

    return &session, nil
}

// LoginHandler handles login requests
func LoginHandler(auth *AuthService) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        username := r.FormValue("username")
        password := r.FormValue("password")

        session, err := auth.Login(username, password)
        if err != nil {
            // Generic error message -- do not reveal specifics
            http.Error(w, "Invalid credentials", http.StatusUnauthorized)
            return
        }

        // Set secure session cookie
        http.SetCookie(w, &http.Cookie{
            Name:     "session_token",
            Value:    session.Token,
            Path:     "/",
            HttpOnly: true,                // Not accessible via JavaScript
            Secure:   true,                // Only sent over HTTPS
            SameSite: http.SameSiteStrictMode, // CSRF protection
            MaxAge:   86400,               // 24 hours
        })

        fmt.Fprintln(w, "Login successful")
    }
}

// AuthMiddleware protects routes that require authentication
func AuthMiddleware(auth *AuthService, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        cookie, err := r.Cookie("session_token")
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        session, err := auth.ValidateSession(cookie.Value)
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Use constant-time comparison for token validation
        if subtle.ConstantTimeCompare([]byte(cookie.Value), []byte(session.Token)) != 1 {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Store user info in context for downstream handlers
        // (omitted for brevity -- use context.WithValue)
        next.ServeHTTP(w, r)
    })
}
```

**Key authentication security practices:**

| Practice                          | Why                                              |
|----------------------------------|--------------------------------------------------|
| Hash passwords with bcrypt/argon2 | Resistant to brute force, includes salt           |
| Use constant-time comparison      | Prevents timing attacks                          |
| Generic error messages            | Prevents username enumeration                    |
| Secure cookie flags               | Prevents cookie theft via XSS, MITM             |
| Session expiration                | Limits window of attack for stolen sessions      |
| Rate limiting on login            | Prevents brute force attacks                     |

---

## Cryptography in Go

Go's `crypto` package provides well-reviewed, production-quality cryptographic primitives.
The golden rule: **never implement your own cryptographic algorithms -- use the standard
library.**

### crypto/rand for Secure Random Numbers

Always use `crypto/rand` for security-sensitive random values. Never use `math/rand` for
security purposes.

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "encoding/hex"
    "fmt"
    "math/big"
    // "math/rand" -- NEVER use this for security-sensitive values
)

func main() {
    // Generate a random byte slice (e.g., for encryption keys, tokens)
    key := make([]byte, 32) // 256-bit key
    if _, err := rand.Read(key); err != nil {
        panic(fmt.Sprintf("crypto/rand failed: %v", err))
    }
    fmt.Printf("Random key (hex):    %s\n", hex.EncodeToString(key))
    fmt.Printf("Random key (base64): %s\n", base64.StdEncoding.EncodeToString(key))

    // Generate a random token for URLs / API keys
    token := make([]byte, 32)
    if _, err := rand.Read(token); err != nil {
        panic(fmt.Sprintf("crypto/rand failed: %v", err))
    }
    fmt.Printf("API token: %s\n", base64.URLEncoding.EncodeToString(token))

    // Generate a random integer in a range
    // (e.g., for OTP, verification codes)
    max := big.NewInt(1000000) // 6-digit code
    n, err := rand.Int(rand.Reader, max)
    if err != nil {
        panic(fmt.Sprintf("crypto/rand failed: %v", err))
    }
    fmt.Printf("OTP code: %06d\n", n.Int64())

    // Generate a random UUID v4
    uuid := generateUUIDv4()
    fmt.Printf("UUID: %s\n", uuid)
}

func generateUUIDv4() string {
    b := make([]byte, 16)
    if _, err := rand.Read(b); err != nil {
        panic(fmt.Sprintf("crypto/rand failed: %v", err))
    }

    // Set version 4 and variant bits per RFC 4122
    b[6] = (b[6] & 0x0f) | 0x40 // version 4
    b[8] = (b[8] & 0x3f) | 0x80 // variant 1

    return fmt.Sprintf("%08x-%04x-%04x-%04x-%012x",
        b[0:4], b[4:6], b[6:8], b[8:10], b[10:16])
}
```

> **Warning:** `math/rand` is a pseudo-random number generator (PRNG) that is
> **deterministic** and **predictable**. If an attacker can observe a few outputs, they can
> predict all future outputs. Never use it for tokens, keys, passwords, nonces, or any
> other security-sensitive purpose.

### Hashing

#### SHA-256

SHA-256 is a cryptographic hash function suitable for data integrity verification, but
**not for password hashing** (it is too fast, making brute force feasible).

```go
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "io"
    "os"
)

func main() {
    // Hash a string
    data := "Hello, World!"
    hash := sha256.Sum256([]byte(data))
    fmt.Printf("SHA-256: %s\n", hex.EncodeToString(hash[:]))

    // Hash a large file efficiently (streaming)
    hashFile("example.txt")
}

func hashFile(filename string) {
    f, err := os.Open(filename)
    if err != nil {
        fmt.Printf("Could not open file: %v\n", err)
        return
    }
    defer f.Close()

    h := sha256.New()
    if _, err := io.Copy(h, f); err != nil {
        fmt.Printf("Could not hash file: %v\n", err)
        return
    }

    fmt.Printf("File SHA-256: %s\n", hex.EncodeToString(h.Sum(nil)))
}
```

#### bcrypt for Password Hashing

bcrypt is the minimum standard for password hashing. It is intentionally slow and includes
a built-in salt.

```go
package main

import (
    "fmt"

    "golang.org/x/crypto/bcrypt"
)

func main() {
    password := "my-secure-password-2024!"

    // Hash the password
    // Cost factor 12 means 2^12 = 4096 iterations
    // Higher cost = slower = more secure against brute force
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    if err != nil {
        panic(err)
    }
    fmt.Printf("bcrypt hash: %s\n", string(hash))
    // Example: $2a$12$LJ3m4ys0tXri3rGFm.yXxeFBs9VBg6OYQO1.dfbHuAfu1TZLe8kU2

    // Verify a password against the hash
    err = bcrypt.CompareHashAndPassword(hash, []byte(password))
    if err != nil {
        fmt.Println("Password does NOT match")
    } else {
        fmt.Println("Password matches!")
    }

    // Verify with wrong password
    err = bcrypt.CompareHashAndPassword(hash, []byte("wrong-password"))
    if err != nil {
        fmt.Println("Wrong password correctly rejected")
    }
}
```

#### Argon2 for Password Hashing (Recommended)

Argon2 is the winner of the Password Hashing Competition (2015) and is the current best
practice for password hashing. It is resistant to GPU and ASIC attacks because it is
memory-hard.

```go
package main

import (
    "crypto/rand"
    "crypto/subtle"
    "encoding/base64"
    "fmt"
    "strings"

    "golang.org/x/crypto/argon2"
)

// Argon2Params holds the configuration for Argon2 hashing
type Argon2Params struct {
    Memory      uint32 // Memory in KiB (e.g., 64*1024 = 64 MB)
    Iterations  uint32 // Time cost (number of passes)
    Parallelism uint8  // Number of threads
    SaltLength  uint32 // Salt length in bytes
    KeyLength   uint32 // Derived key length in bytes
}

// DefaultParams returns recommended Argon2 parameters
// These follow OWASP recommendations as of 2024
func DefaultParams() *Argon2Params {
    return &Argon2Params{
        Memory:      64 * 1024, // 64 MB
        Iterations:  3,         // 3 passes
        Parallelism: 4,         // 4 threads
        SaltLength:  16,        // 128-bit salt
        KeyLength:   32,        // 256-bit key
    }
}

// HashPassword hashes a password with Argon2id
func HashPassword(password string, params *Argon2Params) (string, error) {
    // Generate a random salt
    salt := make([]byte, params.SaltLength)
    if _, err := rand.Read(salt); err != nil {
        return "", fmt.Errorf("failed to generate salt: %w", err)
    }

    // Hash the password with Argon2id
    // Argon2id is the recommended variant (hybrid of Argon2i and Argon2d)
    hash := argon2.IDKey(
        []byte(password),
        salt,
        params.Iterations,
        params.Memory,
        params.Parallelism,
        params.KeyLength,
    )

    // Encode as a standard string format:
    // $argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>
    encodedSalt := base64.RawStdEncoding.EncodeToString(salt)
    encodedHash := base64.RawStdEncoding.EncodeToString(hash)

    return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version,
        params.Memory,
        params.Iterations,
        params.Parallelism,
        encodedSalt,
        encodedHash,
    ), nil
}

// VerifyPassword checks a password against an Argon2id hash
func VerifyPassword(password, encodedHash string) (bool, error) {
    // Parse the encoded hash
    parts := strings.Split(encodedHash, "$")
    if len(parts) != 6 {
        return false, fmt.Errorf("invalid hash format")
    }

    var version int
    var memory, iterations uint32
    var parallelism uint8

    _, err := fmt.Sscanf(parts[2], "v=%d", &version)
    if err != nil {
        return false, fmt.Errorf("invalid version: %w", err)
    }

    _, err = fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &memory, &iterations, &parallelism)
    if err != nil {
        return false, fmt.Errorf("invalid parameters: %w", err)
    }

    salt, err := base64.RawStdEncoding.DecodeString(parts[4])
    if err != nil {
        return false, fmt.Errorf("invalid salt: %w", err)
    }

    expectedHash, err := base64.RawStdEncoding.DecodeString(parts[5])
    if err != nil {
        return false, fmt.Errorf("invalid hash: %w", err)
    }

    // Recompute the hash with the same parameters
    computedHash := argon2.IDKey(
        []byte(password),
        salt,
        iterations,
        memory,
        parallelism,
        uint32(len(expectedHash)),
    )

    // Use constant-time comparison
    return subtle.ConstantTimeCompare(computedHash, expectedHash) == 1, nil
}

func main() {
    params := DefaultParams()

    hash, err := HashPassword("my-secure-password!", params)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Argon2id hash: %s\n", hash)

    // Verify correct password
    match, err := VerifyPassword("my-secure-password!", hash)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Correct password: %v\n", match) // true

    // Verify wrong password
    match, err = VerifyPassword("wrong-password", hash)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Wrong password: %v\n", match) // false
}
```

**Password hashing comparison:**

| Algorithm | Memory-Hard | GPU Resistant | Recommended Use         |
|-----------|------------|---------------|------------------------|
| MD5/SHA   | No         | No            | Never for passwords    |
| bcrypt    | No         | Moderate      | Legacy, still acceptable|
| scrypt    | Yes        | Yes           | Good alternative        |
| Argon2id  | Yes        | Yes           | Best practice (current) |

---

### AES Encryption/Decryption

AES-GCM (Galois/Counter Mode) is the recommended mode for symmetric encryption. It provides
both confidentiality and authenticity (authenticated encryption).

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "io"
)

// Encrypt encrypts plaintext using AES-256-GCM
func Encrypt(plaintext, key []byte) ([]byte, error) {
    // key must be 16 (AES-128), 24 (AES-192), or 32 (AES-256) bytes
    if len(key) != 32 {
        return nil, fmt.Errorf("key must be 32 bytes for AES-256")
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, fmt.Errorf("failed to create cipher: %w", err)
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, fmt.Errorf("failed to create GCM: %w", err)
    }

    // Generate a random nonce
    // GCM nonces should NEVER be reused with the same key
    nonce := make([]byte, aesGCM.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, fmt.Errorf("failed to generate nonce: %w", err)
    }

    // Seal encrypts and authenticates the plaintext
    // The nonce is prepended to the ciphertext for later extraction
    ciphertext := aesGCM.Seal(nonce, nonce, plaintext, nil)
    return ciphertext, nil
}

// Decrypt decrypts ciphertext using AES-256-GCM
func Decrypt(ciphertext, key []byte) ([]byte, error) {
    if len(key) != 32 {
        return nil, fmt.Errorf("key must be 32 bytes for AES-256")
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, fmt.Errorf("failed to create cipher: %w", err)
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, fmt.Errorf("failed to create GCM: %w", err)
    }

    nonceSize := aesGCM.NonceSize()
    if len(ciphertext) < nonceSize {
        return nil, fmt.Errorf("ciphertext too short")
    }

    // Extract the nonce from the beginning of the ciphertext
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]

    // Open decrypts and verifies the ciphertext
    // If the data has been tampered with, this will return an error
    plaintext, err := aesGCM.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return nil, fmt.Errorf("decryption failed (data may be tampered): %w", err)
    }

    return plaintext, nil
}

// EncryptWithAAD encrypts with Additional Authenticated Data
// AAD is authenticated but NOT encrypted -- useful for metadata
func EncryptWithAAD(plaintext, key, aad []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, aesGCM.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }

    // The aad parameter is authenticated but not encrypted
    ciphertext := aesGCM.Seal(nonce, nonce, plaintext, aad)
    return ciphertext, nil
}

func main() {
    // Generate a random 256-bit key
    key := make([]byte, 32)
    if _, err := rand.Read(key); err != nil {
        panic(err)
    }
    fmt.Printf("Key: %s\n", hex.EncodeToString(key))

    // Encrypt
    plaintext := []byte("This is a secret message!")
    ciphertext, err := Encrypt(plaintext, key)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Ciphertext: %s\n", hex.EncodeToString(ciphertext))

    // Decrypt
    decrypted, err := Decrypt(ciphertext, key)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Decrypted: %s\n", string(decrypted))

    // Tamper detection: modify a byte in the ciphertext
    tampered := make([]byte, len(ciphertext))
    copy(tampered, ciphertext)
    tampered[len(tampered)-1] ^= 0xFF // Flip bits in the last byte

    _, err = Decrypt(tampered, key)
    if err != nil {
        fmt.Printf("Tamper detected: %v\n", err) // This WILL be triggered
    }
}
```

> **Warning:** Never reuse a nonce with the same key in AES-GCM. Nonce reuse completely
> breaks the security of GCM. Always generate a fresh random nonce for each encryption
> operation.

---

### RSA and Ed25519 Digital Signatures

Digital signatures provide authentication, integrity, and non-repudiation.

#### RSA Signing

```go
package main

import (
    "crypto"
    "crypto/rand"
    "crypto/rsa"
    "crypto/sha256"
    "crypto/x509"
    "encoding/pem"
    "fmt"
    "os"
)

func main() {
    // Generate a 4096-bit RSA key pair
    // Minimum 2048 bits; 4096 bits recommended for long-term security
    privateKey, err := rsa.GenerateKey(rand.Reader, 4096)
    if err != nil {
        panic(err)
    }
    publicKey := &privateKey.PublicKey

    // Export the private key to PEM format
    privateKeyPEM := pem.EncodeToMemory(&pem.Block{
        Type:  "RSA PRIVATE KEY",
        Bytes: x509.MarshalPKCS1PrivateKey(privateKey),
    })
    fmt.Println("Private key (first 100 chars):", string(privateKeyPEM[:100]))

    // Export the public key to PEM format
    publicKeyBytes, err := x509.MarshalPKIXPublicKey(publicKey)
    if err != nil {
        panic(err)
    }
    publicKeyPEM := pem.EncodeToMemory(&pem.Block{
        Type:  "PUBLIC KEY",
        Bytes: publicKeyBytes,
    })
    fmt.Println("Public key:", string(publicKeyPEM))

    // Sign a message
    message := []byte("This message is authentic")
    hashed := sha256.Sum256(message)

    // Use PSS padding (more secure than PKCS1v15)
    signature, err := rsa.SignPSS(
        rand.Reader,
        privateKey,
        crypto.SHA256,
        hashed[:],
        &rsa.PSSOptions{SaltLength: rsa.PSSSaltLengthEqualsHash},
    )
    if err != nil {
        panic(err)
    }
    fmt.Printf("Signature length: %d bytes\n", len(signature))

    // Verify the signature
    err = rsa.VerifyPSS(
        publicKey,
        crypto.SHA256,
        hashed[:],
        signature,
        &rsa.PSSOptions{SaltLength: rsa.PSSSaltLengthEqualsHash},
    )
    if err != nil {
        fmt.Println("Signature verification FAILED:", err)
    } else {
        fmt.Println("Signature verified successfully!")
    }

    // Verify with tampered message
    tampered := []byte("This message has been altered")
    tamperedHash := sha256.Sum256(tampered)
    err = rsa.VerifyPSS(publicKey, crypto.SHA256, tamperedHash[:], signature, nil)
    if err != nil {
        fmt.Println("Tampered message correctly rejected:", err)
    }

    // Save keys to files (for demonstration)
    _ = os.WriteFile("private_key.pem", privateKeyPEM, 0600) // Owner read/write only
    _ = os.WriteFile("public_key.pem", publicKeyPEM, 0644)
}
```

#### Ed25519 Signing (Recommended for New Applications)

Ed25519 is a modern elliptic curve signature scheme that is faster, produces smaller
signatures, and has fewer implementation pitfalls than RSA:

```go
package main

import (
    "crypto/ed25519"
    "crypto/rand"
    "encoding/hex"
    "fmt"
)

func main() {
    // Generate an Ed25519 key pair
    // Ed25519 keys are always 256 bits -- no size decisions needed
    publicKey, privateKey, err := ed25519.GenerateKey(rand.Reader)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Public key  (%d bytes): %s\n", len(publicKey), hex.EncodeToString(publicKey))
    fmt.Printf("Private key (%d bytes): %s\n", len(privateKey), hex.EncodeToString(privateKey))

    // Sign a message
    message := []byte("This is an authentic message")
    signature := ed25519.Sign(privateKey, message)
    fmt.Printf("Signature   (%d bytes): %s\n", len(signature), hex.EncodeToString(signature))

    // Verify the signature
    valid := ed25519.Verify(publicKey, message, signature)
    fmt.Printf("Valid signature: %v\n", valid) // true

    // Verify with tampered message
    tampered := []byte("This is a tampered message")
    valid = ed25519.Verify(publicKey, tampered, signature)
    fmt.Printf("Tampered message: %v\n", valid) // false
}
```

**Comparison of signature algorithms:**

| Property              | RSA-4096          | Ed25519          |
|----------------------|-------------------|------------------|
| Key generation speed  | Slow (~seconds)   | Fast (~ms)       |
| Signing speed         | Moderate          | Fast             |
| Verification speed    | Fast              | Fast             |
| Public key size       | 512 bytes         | 32 bytes         |
| Signature size        | 512 bytes         | 64 bytes         |
| Security level        | ~128 bits         | ~128 bits        |
| Misuse resistance     | Low (many params) | High (no params) |
| Recommended           | Legacy support    | New applications |

---

### HMAC for Message Authentication

HMAC (Hash-based Message Authentication Code) verifies both the integrity and authenticity
of a message using a shared secret key:

```go
package main

import (
    "crypto/hmac"
    "crypto/rand"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
)

// SignMessage creates an HMAC signature for a message
func SignMessage(message, key []byte) []byte {
    mac := hmac.New(sha256.New, key)
    mac.Write(message)
    return mac.Sum(nil)
}

// VerifyMessage checks if the HMAC signature is valid
func VerifyMessage(message, signature, key []byte) bool {
    expectedMAC := SignMessage(message, key)
    // IMPORTANT: Use hmac.Equal for constant-time comparison
    // NEVER use bytes.Equal or == for MAC comparison (timing attack)
    return hmac.Equal(signature, expectedMAC)
}

func main() {
    // Generate a shared secret key
    key := make([]byte, 32)
    if _, err := rand.Read(key); err != nil {
        panic(err)
    }

    message := []byte("Transfer $1000 to account 12345")

    // Create HMAC signature
    signature := SignMessage(message, key)
    fmt.Printf("HMAC: %s\n", hex.EncodeToString(signature))

    // Verify the message
    valid := VerifyMessage(message, signature, key)
    fmt.Printf("Valid: %v\n", valid) // true

    // Try to verify a tampered message
    tampered := []byte("Transfer $9999 to account 99999")
    valid = VerifyMessage(tampered, signature, key)
    fmt.Printf("Tampered: %v\n", valid) // false

    // Practical example: Webhook signature verification
    webhookBody := []byte(`{"event": "payment.completed", "amount": 1000}`)
    webhookSecret := []byte("whsec_your_webhook_secret")

    webhookSig := SignMessage(webhookBody, webhookSecret)
    fmt.Printf("\nWebhook signature: %s\n", hex.EncodeToString(webhookSig))
    fmt.Printf("Webhook verified: %v\n", VerifyMessage(webhookBody, webhookSig, webhookSecret))
}
```

> **Warning:** Always use `hmac.Equal()` to compare MACs. Using `==` or `bytes.Equal()`
> is vulnerable to timing attacks, where an attacker can determine the correct MAC one byte
> at a time by measuring response time differences.

---

## TLS Configuration

Transport Layer Security (TLS) is essential for protecting data in transit. Go's
`crypto/tls` package provides excellent support for configuring TLS with strong defaults.

### Setting Up a TLS Server (Best Practices)

```go
package main

import (
    "crypto/tls"
    "fmt"
    "log"
    "net/http"
    "time"
)

func main() {
    // Production-grade TLS configuration
    tlsConfig := &tls.Config{
        // Minimum TLS version: 1.2
        // TLS 1.0 and 1.1 are deprecated and have known vulnerabilities
        MinVersion: tls.VersionTLS12,

        // Prefer server cipher suites (the server picks the strongest)
        // As of Go 1.22, Go automatically selects secure cipher suites
        // and this field is largely unnecessary, but explicit is better
        CipherSuites: []uint16{
            // TLS 1.3 cipher suites (automatically used when TLS 1.3 is negotiated)
            // These don't need to be listed -- Go uses them automatically for TLS 1.3

            // TLS 1.2 cipher suites (ordered by preference)
            tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
            tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
            tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,
        },

        // Prefer server's cipher suite order
        PreferServerCipherSuites: true,

        // Elliptic curves to use
        CurvePreferences: []tls.CurveID{
            tls.X25519, // Most secure and fastest
            tls.CurveP256,
        },
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, secure world! TLS version: %d", r.TLS.Version)
    })

    server := &http.Server{
        Addr:      ":443",
        Handler:   mux,
        TLSConfig: tlsConfig,

        // Timeouts prevent slowloris attacks and resource exhaustion
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,

        // Limit header size to prevent memory exhaustion
        MaxHeaderBytes: 1 << 20, // 1 MB
    }

    // Redirect HTTP to HTTPS
    go func() {
        redirectServer := &http.Server{
            Addr:         ":80",
            ReadTimeout:  5 * time.Second,
            WriteTimeout: 5 * time.Second,
            Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                target := "https://" + r.Host + r.URL.RequestURI()
                http.Redirect(w, r, target, http.StatusMovedPermanently)
            }),
        }
        log.Fatal(redirectServer.ListenAndServe())
    }()

    log.Println("Starting HTTPS server on :443")
    log.Fatal(server.ListenAndServeTLS("cert.pem", "key.pem"))
}
```

### Mutual TLS (mTLS)

mTLS requires both the server and client to present certificates, providing bidirectional
authentication. This is commonly used for service-to-service communication.

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "io"
    "log"
    "net/http"
    "os"
)

// ============================================================
// SERVER with mTLS
// ============================================================

func startMTLSServer() {
    // Load the CA certificate that signed the client certificates
    caCert, err := os.ReadFile("ca-cert.pem")
    if err != nil {
        log.Fatalf("Failed to read CA cert: %v", err)
    }

    caCertPool := x509.NewCertPool()
    if !caCertPool.AppendCertsFromPEM(caCert) {
        log.Fatal("Failed to parse CA cert")
    }

    tlsConfig := &tls.Config{
        // Require and verify client certificates
        ClientAuth: tls.RequireAndVerifyClientCert,

        // Trust only certificates signed by our CA
        ClientCAs: caCertPool,

        MinVersion: tls.VersionTLS12,
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        // Access client certificate information
        if r.TLS != nil && len(r.TLS.PeerCertificates) > 0 {
            clientCert := r.TLS.PeerCertificates[0]
            fmt.Fprintf(w, "Hello, %s! (verified by mTLS)\n", clientCert.Subject.CommonName)
            fmt.Fprintf(w, "Organization: %v\n", clientCert.Subject.Organization)
            fmt.Fprintf(w, "Serial Number: %s\n", clientCert.SerialNumber.String())
        } else {
            fmt.Fprintln(w, "No client certificate presented")
        }
    })

    server := &http.Server{
        Addr:      ":8443",
        Handler:   mux,
        TLSConfig: tlsConfig,
    }

    log.Println("Starting mTLS server on :8443")
    log.Fatal(server.ListenAndServeTLS("server-cert.pem", "server-key.pem"))
}

// ============================================================
// CLIENT with mTLS
// ============================================================

func createMTLSClient() (*http.Client, error) {
    // Load client certificate and private key
    clientCert, err := tls.LoadX509KeyPair("client-cert.pem", "client-key.pem")
    if err != nil {
        return nil, fmt.Errorf("failed to load client cert: %w", err)
    }

    // Load the CA certificate to verify the server
    caCert, err := os.ReadFile("ca-cert.pem")
    if err != nil {
        return nil, fmt.Errorf("failed to read CA cert: %w", err)
    }

    caCertPool := x509.NewCertPool()
    if !caCertPool.AppendCertsFromPEM(caCert) {
        return nil, fmt.Errorf("failed to parse CA cert")
    }

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{clientCert},
        RootCAs:      caCertPool,
        MinVersion:   tls.VersionTLS12,
    }

    return &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: tlsConfig,
        },
    }, nil
}

func main() {
    // Start the server in a goroutine
    go startMTLSServer()

    // Create the mTLS client
    client, err := createMTLSClient()
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }

    // Make a request
    resp, err := client.Get("https://localhost:8443/")
    if err != nil {
        log.Fatalf("Request failed: %v", err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

### Certificate Management with Let's Encrypt (autocert)

The `golang.org/x/crypto/acme/autocert` package automates certificate management with
Let's Encrypt:

```go
package main

import (
    "crypto/tls"
    "fmt"
    "log"
    "net/http"
    "time"

    "golang.org/x/crypto/acme/autocert"
)

func main() {
    // Create an autocert manager
    certManager := autocert.Manager{
        Prompt: autocert.AcceptTOS,

        // Cache certificates on disk to survive restarts
        Cache: autocert.DirCache("/var/lib/certs"),

        // Only allow certificates for these domains
        HostPolicy: autocert.HostWhitelist(
            "example.com",
            "www.example.com",
        ),
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, auto-TLS world!")
    })

    // HTTPS server with autocert
    server := &http.Server{
        Addr:    ":443",
        Handler: mux,
        TLSConfig: &tls.Config{
            GetCertificate: certManager.GetCertificate,
            MinVersion:     tls.VersionTLS12,
        },
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // HTTP server for ACME challenges and HTTP->HTTPS redirect
    go func() {
        h := certManager.HTTPHandler(nil) // nil = redirect to HTTPS
        log.Fatal(http.ListenAndServe(":80", h))
    }()

    log.Println("Starting auto-TLS server on :443")
    log.Fatal(server.ListenAndServeTLS("", "")) // Empty strings = use autocert
}
```

> **Tip:** In production, always set `HostPolicy` to restrict which domains can obtain
> certificates. Without it, an attacker could point their domain at your server and drain
> your Let's Encrypt rate limit.

---

## JWT Implementation and Pitfalls

JSON Web Tokens (JWTs) are widely used for authentication and authorization. However, they
are frequently misimplemented. This section covers correct usage and common pitfalls.

### Basic JWT Implementation

```go
package main

import (
    "crypto/ed25519"
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "strings"
    "time"
)

// JWTHeader represents the JWT header
type JWTHeader struct {
    Alg string `json:"alg"`
    Typ string `json:"typ"`
}

// JWTClaims represents the JWT payload
type JWTClaims struct {
    // Registered claims (RFC 7519)
    Issuer    string `json:"iss,omitempty"` // Who issued the token
    Subject   string `json:"sub,omitempty"` // Who the token is about
    Audience  string `json:"aud,omitempty"` // Intended recipient
    ExpiresAt int64  `json:"exp,omitempty"` // Expiration time (Unix timestamp)
    NotBefore int64  `json:"nbf,omitempty"` // Not valid before (Unix timestamp)
    IssuedAt  int64  `json:"iat,omitempty"` // When the token was issued

    // Custom claims
    UserID string   `json:"user_id,omitempty"`
    Email  string   `json:"email,omitempty"`
    Roles  []string `json:"roles,omitempty"`
}

// JWTService handles JWT creation and verification
type JWTService struct {
    privateKey ed25519.PrivateKey
    publicKey  ed25519.PublicKey
    issuer     string
}

func NewJWTService(issuer string) (*JWTService, error) {
    pub, priv, err := ed25519.GenerateKey(rand.Reader)
    if err != nil {
        return nil, err
    }
    return &JWTService{
        privateKey: priv,
        publicKey:  pub,
        issuer:     issuer,
    }, nil
}

// CreateToken creates a signed JWT
func (s *JWTService) CreateToken(userID, email string, roles []string, duration time.Duration) (string, error) {
    now := time.Now()

    claims := JWTClaims{
        Issuer:    s.issuer,
        Subject:   userID,
        ExpiresAt: now.Add(duration).Unix(),
        NotBefore: now.Unix(),
        IssuedAt:  now.Unix(),
        UserID:    userID,
        Email:     email,
        Roles:     roles,
    }

    header := JWTHeader{
        Alg: "EdDSA",
        Typ: "JWT",
    }

    // Encode header
    headerJSON, err := json.Marshal(header)
    if err != nil {
        return "", err
    }
    headerEncoded := base64.RawURLEncoding.EncodeToString(headerJSON)

    // Encode payload
    claimsJSON, err := json.Marshal(claims)
    if err != nil {
        return "", err
    }
    claimsEncoded := base64.RawURLEncoding.EncodeToString(claimsJSON)

    // Create the signing input
    signingInput := headerEncoded + "." + claimsEncoded

    // Sign with Ed25519
    signature := ed25519.Sign(s.privateKey, []byte(signingInput))
    signatureEncoded := base64.RawURLEncoding.EncodeToString(signature)

    return signingInput + "." + signatureEncoded, nil
}

// VerifyToken verifies and parses a JWT
func (s *JWTService) VerifyToken(tokenString string) (*JWTClaims, error) {
    parts := strings.Split(tokenString, ".")
    if len(parts) != 3 {
        return nil, fmt.Errorf("invalid token format")
    }

    // Decode and verify header
    headerJSON, err := base64.RawURLEncoding.DecodeString(parts[0])
    if err != nil {
        return nil, fmt.Errorf("invalid header encoding: %w", err)
    }

    var header JWTHeader
    if err := json.Unmarshal(headerJSON, &header); err != nil {
        return nil, fmt.Errorf("invalid header: %w", err)
    }

    // CRITICAL: Verify the algorithm matches what we expect
    // This prevents the "none" algorithm attack and algorithm confusion attacks
    if header.Alg != "EdDSA" {
        return nil, fmt.Errorf("unexpected algorithm: %s (expected EdDSA)", header.Alg)
    }

    // Verify the signature
    signingInput := parts[0] + "." + parts[1]
    signature, err := base64.RawURLEncoding.DecodeString(parts[2])
    if err != nil {
        return nil, fmt.Errorf("invalid signature encoding: %w", err)
    }

    if !ed25519.Verify(s.publicKey, []byte(signingInput), signature) {
        return nil, fmt.Errorf("invalid signature")
    }

    // Decode claims
    claimsJSON, err := base64.RawURLEncoding.DecodeString(parts[1])
    if err != nil {
        return nil, fmt.Errorf("invalid claims encoding: %w", err)
    }

    var claims JWTClaims
    if err := json.Unmarshal(claimsJSON, &claims); err != nil {
        return nil, fmt.Errorf("invalid claims: %w", err)
    }

    // Validate time-based claims
    now := time.Now().Unix()

    if claims.ExpiresAt != 0 && now > claims.ExpiresAt {
        return nil, fmt.Errorf("token has expired")
    }

    if claims.NotBefore != 0 && now < claims.NotBefore {
        return nil, fmt.Errorf("token is not yet valid")
    }

    // Validate issuer
    if claims.Issuer != s.issuer {
        return nil, fmt.Errorf("invalid issuer: %s", claims.Issuer)
    }

    return &claims, nil
}

func main() {
    svc, err := NewJWTService("my-app")
    if err != nil {
        panic(err)
    }

    // Create a token
    token, err := svc.CreateToken("user-123", "user@example.com", []string{"admin", "editor"}, 1*time.Hour)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Token: %s\n\n", token)

    // Verify the token
    claims, err := svc.VerifyToken(token)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Verified claims:\n")
    fmt.Printf("  UserID: %s\n", claims.UserID)
    fmt.Printf("  Email: %s\n", claims.Email)
    fmt.Printf("  Roles: %v\n", claims.Roles)
    fmt.Printf("  Expires: %s\n", time.Unix(claims.ExpiresAt, 0))
}
```

### JWT Pitfalls and Mitigations

Here are the most common JWT security mistakes and how to prevent them:

#### Pitfall 1: The "none" Algorithm Attack

```go
// VULNERABLE: Accepting the "alg" from the token header
func vulnerableVerify(tokenString string) (*JWTClaims, error) {
    parts := strings.Split(tokenString, ".")
    headerJSON, _ := base64.RawURLEncoding.DecodeString(parts[0])

    var header JWTHeader
    json.Unmarshal(headerJSON, &header)

    // DANGER: An attacker can set alg to "none" and skip verification!
    switch header.Alg {
    case "none":
        // No verification needed (CATASTROPHIC)
    case "HS256":
        // Verify with HMAC
    case "RS256":
        // Verify with RSA
    }
    // ...
}

// SECURE: Always enforce the expected algorithm
func secureVerify(tokenString string, expectedAlg string) (*JWTClaims, error) {
    parts := strings.Split(tokenString, ".")
    headerJSON, _ := base64.RawURLEncoding.DecodeString(parts[0])

    var header JWTHeader
    json.Unmarshal(headerJSON, &header)

    // CRITICAL: Reject tokens that don't use the expected algorithm
    if header.Alg != expectedAlg {
        return nil, fmt.Errorf("unexpected algorithm: %s", header.Alg)
    }
    // ...
}
```

#### Pitfall 2: Algorithm Confusion (RS256 vs HS256)

```go
// VULNERABLE: The server uses RSA (RS256) for signing.
// An attacker crafts a token with alg=HS256 and signs it with
// the PUBLIC key (which is public!). If the server uses the same
// key for HMAC verification, the attacker's token will be accepted.

// SECURE: Always verify tokens with the algorithm you EXPECT,
// not the algorithm the TOKEN claims to use.

// Even better: Use a single algorithm (like EdDSA) and reject
// all tokens that claim a different algorithm.
```

#### Pitfall 3: Missing Expiration

```go
// VULNERABLE: Token without expiration
claims := JWTClaims{
    UserID: "user-123",
    Roles:  []string{"admin"},
    // No ExpiresAt set!
}

// SECURE: Always set reasonable expiration times
claims = JWTClaims{
    UserID:    "user-123",
    Roles:     []string{"admin"},
    ExpiresAt: time.Now().Add(15 * time.Minute).Unix(), // Short-lived access token
    IssuedAt:  time.Now().Unix(),
}
```

#### Pitfall 4: Storing Sensitive Data in JWTs

```go
// VULNERABLE: JWTs are NOT encrypted -- they are only Base64-encoded!
// Anyone can decode the payload and read its contents.
claims := JWTClaims{
    UserID: "user-123",
    Email:  "user@example.com",
    // NEVER put these in a JWT:
    // Password: "secret123",
    // SSN: "123-45-6789",
    // CreditCard: "4111111111111111",
}

// The payload is trivially decodable:
// echo "eyJ1c2VyX2lkIjoiMTIzIn0" | base64 -d
// {"user_id":"123"}
```

**JWT security checklist:**

| Check | Description |
|-------|-------------|
| Always validate `alg` | Never trust the algorithm from the token header |
| Set expiration | Use short-lived tokens (15 minutes for access tokens) |
| Validate all claims | Check `exp`, `nbf`, `iss`, `aud` |
| Use asymmetric signing | Prefer Ed25519 or RS256 over HS256 |
| No sensitive data in payload | JWTs are not encrypted by default |
| Use refresh tokens | Short access tokens + longer refresh tokens |
| Revocation strategy | Maintain a blocklist or use short expiry |

---

## Secret Management

Secrets (API keys, database credentials, encryption keys) must never be hardcoded in source
code. Here are strategies for managing secrets securely.

### Environment Variables

The simplest approach, suitable for development and simple deployments:

```go
package main

import (
    "fmt"
    "log"
    "os"
)

// Config holds application configuration
type Config struct {
    DatabaseURL string
    APIKey      string
    JWTSecret   string
    Port        string
}

// LoadConfig loads configuration from environment variables
func LoadConfig() (*Config, error) {
    config := &Config{
        DatabaseURL: os.Getenv("DATABASE_URL"),
        APIKey:      os.Getenv("API_KEY"),
        JWTSecret:   os.Getenv("JWT_SECRET"),
        Port:        os.Getenv("PORT"),
    }

    // Validate required configuration
    if config.DatabaseURL == "" {
        return nil, fmt.Errorf("DATABASE_URL environment variable is required")
    }
    if config.APIKey == "" {
        return nil, fmt.Errorf("API_KEY environment variable is required")
    }
    if config.JWTSecret == "" {
        return nil, fmt.Errorf("JWT_SECRET environment variable is required")
    }

    // Set defaults for optional values
    if config.Port == "" {
        config.Port = "8080"
    }

    return config, nil
}

func main() {
    config, err := LoadConfig()
    if err != nil {
        log.Fatalf("Configuration error: %v", err)
    }

    fmt.Printf("Server starting on port %s\n", config.Port)
    // IMPORTANT: Never log secret values!
    // log.Printf("API Key: %s", config.APIKey) // NEVER DO THIS
}
```

> **Warning:** Never commit `.env` files to version control. Add `.env` to your
> `.gitignore` file.

### HashiCorp Vault Integration

For production systems, use a dedicated secrets manager like HashiCorp Vault:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    vault "github.com/hashicorp/vault/api"
)

// VaultSecretManager manages secrets using HashiCorp Vault
type VaultSecretManager struct {
    client *vault.Client
}

// NewVaultSecretManager creates a new Vault secret manager
func NewVaultSecretManager(address, token string) (*VaultSecretManager, error) {
    config := vault.DefaultConfig()
    config.Address = address

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create vault client: %w", err)
    }

    client.SetToken(token)

    return &VaultSecretManager{client: client}, nil
}

// GetSecret retrieves a secret from Vault
func (v *VaultSecretManager) GetSecret(ctx context.Context, path string) (map[string]interface{}, error) {
    secret, err := v.client.KVv2("secret").Get(ctx, path)
    if err != nil {
        return nil, fmt.Errorf("failed to get secret at %s: %w", path, err)
    }

    return secret.Data, nil
}

// GetDatabaseCredentials retrieves database credentials from Vault
func (v *VaultSecretManager) GetDatabaseCredentials(ctx context.Context) (string, string, error) {
    data, err := v.GetSecret(ctx, "myapp/database")
    if err != nil {
        return "", "", err
    }

    username, ok := data["username"].(string)
    if !ok {
        return "", "", fmt.Errorf("username not found or not a string")
    }

    password, ok := data["password"].(string)
    if !ok {
        return "", "", fmt.Errorf("password not found or not a string")
    }

    return username, password, nil
}

// WatchSecret watches for secret changes and invokes a callback
func (v *VaultSecretManager) WatchSecret(ctx context.Context, path string, callback func(map[string]interface{})) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    var lastVersion int

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            secret, err := v.client.KVv2("secret").Get(ctx, path)
            if err != nil {
                log.Printf("Failed to watch secret: %v", err)
                continue
            }

            version := secret.VersionMetadata.Version
            if version != lastVersion {
                lastVersion = version
                callback(secret.Data)
            }
        }
    }
}

func main() {
    ctx := context.Background()

    manager, err := NewVaultSecretManager("https://vault.example.com:8200", "s.mytoken")
    if err != nil {
        log.Fatalf("Failed to connect to Vault: %v", err)
    }

    // Get database credentials
    username, password, err := manager.GetDatabaseCredentials(ctx)
    if err != nil {
        log.Fatalf("Failed to get credentials: %v", err)
    }

    fmt.Printf("Connected as user: %s\n", username)
    _ = password // Use in database connection string

    // Watch for secret rotation
    go manager.WatchSecret(ctx, "myapp/database", func(data map[string]interface{}) {
        log.Println("Database credentials rotated -- reconnecting...")
        // Reconnect to database with new credentials
    })
}
```

### Kubernetes Sealed Secrets Pattern

For Kubernetes deployments, use Sealed Secrets or external secrets operators:

```go
package main

import (
    "fmt"
    "os"
    "strings"
)

// KubernetesSecretLoader loads secrets mounted as files by Kubernetes
type KubernetesSecretLoader struct {
    mountPath string
}

func NewKubernetesSecretLoader(mountPath string) *KubernetesSecretLoader {
    return &KubernetesSecretLoader{mountPath: mountPath}
}

// GetSecret reads a secret from the mounted volume
func (k *KubernetesSecretLoader) GetSecret(name string) (string, error) {
    // Kubernetes mounts secrets as files at the specified path
    // e.g., /var/run/secrets/myapp/database-password
    path := k.mountPath + "/" + name

    data, err := os.ReadFile(path)
    if err != nil {
        return "", fmt.Errorf("failed to read secret %s: %w", name, err)
    }

    return strings.TrimSpace(string(data)), nil
}

func main() {
    loader := NewKubernetesSecretLoader("/var/run/secrets/myapp")

    dbPassword, err := loader.GetSecret("database-password")
    if err != nil {
        // Fall back to environment variable
        dbPassword = os.Getenv("DATABASE_PASSWORD")
        if dbPassword == "" {
            panic("No database password configured")
        }
    }

    fmt.Println("Secret loaded successfully")
    _ = dbPassword
}
```

**Secret management comparison:**

| Method            | Pros                          | Cons                          | Best For             |
|-------------------|-------------------------------|-------------------------------|----------------------|
| Env vars          | Simple, universal             | No rotation, no audit log     | Development, simple apps |
| Vault             | Rotation, audit, dynamic      | Complex setup, dependency     | Production systems   |
| K8s Secrets       | Native K8s, easy to use       | Base64 not encrypted at rest  | Kubernetes workloads |
| Sealed Secrets    | GitOps-friendly, encrypted    | K8s-specific                  | GitOps workflows     |
| Cloud KMS/SM      | Managed, integrated           | Cloud vendor lock-in          | Cloud-native apps    |

---

## Input Validation and Sanitization

Never trust user input. Validate and sanitize all input at the boundary of your application.

```go
package main

import (
    "encoding/json"
    "fmt"
    "net"
    "net/http"
    "net/mail"
    "regexp"
    "strings"
    "unicode/utf8"
)

// ========================================
// Validation Framework
// ========================================

// ValidationError holds a list of field validation errors
type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// Validator validates a struct
type Validator struct {
    errors []ValidationError
}

func NewValidator() *Validator {
    return &Validator{}
}

func (v *Validator) AddError(field, message string) {
    v.errors = append(v.errors, ValidationError{Field: field, Message: message})
}

func (v *Validator) HasErrors() bool {
    return len(v.errors) > 0
}

func (v *Validator) Errors() []ValidationError {
    return v.errors
}

// Required checks that a string is not empty
func (v *Validator) Required(field, value string) {
    if strings.TrimSpace(value) == "" {
        v.AddError(field, "is required")
    }
}

// MinLength checks minimum string length (in runes, not bytes)
func (v *Validator) MinLength(field, value string, min int) {
    if utf8.RuneCountInString(value) < min {
        v.AddError(field, fmt.Sprintf("must be at least %d characters", min))
    }
}

// MaxLength checks maximum string length
func (v *Validator) MaxLength(field, value string, max int) {
    if utf8.RuneCountInString(value) > max {
        v.AddError(field, fmt.Sprintf("must be at most %d characters", max))
    }
}

// Email validates an email address
func (v *Validator) Email(field, value string) {
    _, err := mail.ParseAddress(value)
    if err != nil {
        v.AddError(field, "must be a valid email address")
    }
}

// Pattern validates against a regex pattern
func (v *Validator) Pattern(field, value, pattern, message string) {
    re := regexp.MustCompile(pattern)
    if !re.MatchString(value) {
        v.AddError(field, message)
    }
}

// InList checks that a value is in a list of allowed values
func (v *Validator) InList(field, value string, allowed []string) {
    for _, a := range allowed {
        if value == a {
            return
        }
    }
    v.AddError(field, fmt.Sprintf("must be one of: %s", strings.Join(allowed, ", ")))
}

// IPAddress validates an IP address
func (v *Validator) IPAddress(field, value string) {
    if net.ParseIP(value) == nil {
        v.AddError(field, "must be a valid IP address")
    }
}

// NoHTMLTags checks that a string contains no HTML tags
func (v *Validator) NoHTMLTags(field, value string) {
    re := regexp.MustCompile(`<[^>]*>`)
    if re.MatchString(value) {
        v.AddError(field, "must not contain HTML tags")
    }
}

// IntRange validates an integer is within a range
func (v *Validator) IntRange(field string, value, min, max int) {
    if value < min || value > max {
        v.AddError(field, fmt.Sprintf("must be between %d and %d", min, max))
    }
}

// ========================================
// Usage Example
// ========================================

// CreateUserRequest is the API request for creating a user
type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"password"`
    Role     string `json:"role"`
    Age      int    `json:"age"`
    Bio      string `json:"bio"`
}

// Validate validates the CreateUserRequest
func (r *CreateUserRequest) Validate() *Validator {
    v := NewValidator()

    // Username: required, 3-30 chars, alphanumeric + underscore only
    v.Required("username", r.Username)
    v.MinLength("username", r.Username, 3)
    v.MaxLength("username", r.Username, 30)
    v.Pattern("username", r.Username, `^[a-zA-Z0-9_]+$`, "must contain only letters, numbers, and underscores")

    // Email: required, valid format
    v.Required("email", r.Email)
    v.Email("email", r.Email)

    // Password: required, minimum 12 chars
    v.Required("password", r.Password)
    v.MinLength("password", r.Password, 12)

    // Role: must be one of the allowed values
    v.Required("role", r.Role)
    v.InList("role", r.Role, []string{"user", "editor", "admin"})

    // Age: must be between 13 and 150
    v.IntRange("age", r.Age, 13, 150)

    // Bio: optional but no HTML tags and max 500 chars
    if r.Bio != "" {
        v.MaxLength("bio", r.Bio, 500)
        v.NoHTMLTags("bio", r.Bio)
    }

    return v
}

// SanitizeInput sanitizes a string by removing control characters
// and normalizing whitespace
func SanitizeInput(s string) string {
    // Remove null bytes
    s = strings.ReplaceAll(s, "\x00", "")

    // Remove other control characters (except newline, tab)
    var cleaned strings.Builder
    for _, r := range s {
        if r == '\n' || r == '\t' || r >= 32 {
            cleaned.WriteRune(r)
        }
    }

    // Normalize whitespace
    s = strings.TrimSpace(cleaned.String())

    return s
}

func CreateUserHandler(w http.ResponseWriter, r *http.Request) {
    // Limit body size
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MB

    var req CreateUserRequest
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()

    if err := decoder.Decode(&req); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": "Invalid JSON"})
        return
    }

    // Sanitize inputs
    req.Username = SanitizeInput(req.Username)
    req.Email = SanitizeInput(req.Email)
    req.Bio = SanitizeInput(req.Bio)
    // Note: Do NOT sanitize the password (hashing handles that)

    // Validate
    validator := req.Validate()
    if validator.HasErrors() {
        w.WriteHeader(http.StatusUnprocessableEntity)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "errors": validator.Errors(),
        })
        return
    }

    // Process the valid request...
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{
        "message": "User created successfully",
    })
}

func main() {
    http.HandleFunc("/users", CreateUserHandler)
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

### Path Traversal Prevention

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "path/filepath"
    "strings"
)

// SafeFileServer serves files while preventing path traversal attacks
func SafeFileServer(baseDir string) http.HandlerFunc {
    // Resolve the absolute path of the base directory
    absBase, err := filepath.Abs(baseDir)
    if err != nil {
        panic(fmt.Sprintf("invalid base directory: %v", err))
    }

    return func(w http.ResponseWriter, r *http.Request) {
        // Clean the requested path
        requestedPath := filepath.Clean(r.URL.Path)

        // Join with base directory
        fullPath := filepath.Join(absBase, requestedPath)

        // CRITICAL: Verify the resolved path is still within the base directory
        // This prevents path traversal attacks like ../../etc/passwd
        if !strings.HasPrefix(fullPath, absBase+string(os.PathSeparator)) && fullPath != absBase {
            http.Error(w, "Forbidden", http.StatusForbidden)
            return
        }

        // Check that the file exists and is not a directory
        info, err := os.Stat(fullPath)
        if err != nil || info.IsDir() {
            http.Error(w, "Not found", http.StatusNotFound)
            return
        }

        http.ServeFile(w, r, fullPath)
    }
}

func main() {
    http.HandleFunc("/files/", SafeFileServer("/var/www/public"))
    http.ListenAndServe(":8080", nil)
}
```

---

## Security Headers Middleware

HTTP security headers provide defense-in-depth against various attacks. Here is a
comprehensive security headers middleware:

```go
package main

import (
    "fmt"
    "net/http"
    "strings"
)

// SecurityHeadersConfig configures security headers
type SecurityHeadersConfig struct {
    // Content-Security-Policy directives
    CSPDefaultSrc []string
    CSPScriptSrc  []string
    CSPStyleSrc   []string
    CSPImgSrc     []string
    CSPFontSrc    []string
    CSPConnectSrc []string
    CSPFrameSrc   []string

    // HSTS settings
    HSTSMaxAge            int  // in seconds
    HSTSIncludeSubdomains bool
    HSTSPreload           bool

    // Other headers
    FrameOptions          string // DENY, SAMEORIGIN
    ContentTypeNoSniff    bool
    XSSProtection         bool
    ReferrerPolicy        string
    PermissionsPolicy     string
}

// DefaultSecurityConfig returns a secure default configuration
func DefaultSecurityConfig() *SecurityHeadersConfig {
    return &SecurityHeadersConfig{
        CSPDefaultSrc: []string{"'self'"},
        CSPScriptSrc:  []string{"'self'"},
        CSPStyleSrc:   []string{"'self'"},
        CSPImgSrc:     []string{"'self'", "data:"},
        CSPFontSrc:    []string{"'self'"},
        CSPConnectSrc: []string{"'self'"},
        CSPFrameSrc:   []string{"'none'"},

        HSTSMaxAge:            31536000, // 1 year
        HSTSIncludeSubdomains: true,
        HSTSPreload:           true,

        FrameOptions:       "DENY",
        ContentTypeNoSniff: true,
        XSSProtection:      true,
        ReferrerPolicy:     "strict-origin-when-cross-origin",
        PermissionsPolicy:  "camera=(), microphone=(), geolocation=(), payment=()",
    }
}

// buildCSP builds the Content-Security-Policy header value
func (c *SecurityHeadersConfig) buildCSP() string {
    var directives []string

    if len(c.CSPDefaultSrc) > 0 {
        directives = append(directives, "default-src "+strings.Join(c.CSPDefaultSrc, " "))
    }
    if len(c.CSPScriptSrc) > 0 {
        directives = append(directives, "script-src "+strings.Join(c.CSPScriptSrc, " "))
    }
    if len(c.CSPStyleSrc) > 0 {
        directives = append(directives, "style-src "+strings.Join(c.CSPStyleSrc, " "))
    }
    if len(c.CSPImgSrc) > 0 {
        directives = append(directives, "img-src "+strings.Join(c.CSPImgSrc, " "))
    }
    if len(c.CSPFontSrc) > 0 {
        directives = append(directives, "font-src "+strings.Join(c.CSPFontSrc, " "))
    }
    if len(c.CSPConnectSrc) > 0 {
        directives = append(directives, "connect-src "+strings.Join(c.CSPConnectSrc, " "))
    }
    if len(c.CSPFrameSrc) > 0 {
        directives = append(directives, "frame-src "+strings.Join(c.CSPFrameSrc, " "))
    }

    return strings.Join(directives, "; ")
}

// buildHSTS builds the Strict-Transport-Security header value
func (c *SecurityHeadersConfig) buildHSTS() string {
    hsts := fmt.Sprintf("max-age=%d", c.HSTSMaxAge)
    if c.HSTSIncludeSubdomains {
        hsts += "; includeSubDomains"
    }
    if c.HSTSPreload {
        hsts += "; preload"
    }
    return hsts
}

// SecurityHeaders middleware adds security headers to all responses
func SecurityHeaders(config *SecurityHeadersConfig) func(http.Handler) http.Handler {
    // Pre-compute header values
    csp := config.buildCSP()
    hsts := config.buildHSTS()

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Content-Security-Policy
            // Prevents XSS, clickjacking, code injection
            w.Header().Set("Content-Security-Policy", csp)

            // Strict-Transport-Security (HSTS)
            // Forces HTTPS for the specified duration
            w.Header().Set("Strict-Transport-Security", hsts)

            // X-Frame-Options
            // Prevents clickjacking (legacy; CSP frame-ancestors is preferred)
            w.Header().Set("X-Frame-Options", config.FrameOptions)

            // X-Content-Type-Options
            // Prevents MIME type sniffing
            if config.ContentTypeNoSniff {
                w.Header().Set("X-Content-Type-Options", "nosniff")
            }

            // X-XSS-Protection
            // Enables browser XSS filter (legacy, but harmless to include)
            if config.XSSProtection {
                w.Header().Set("X-XSS-Protection", "1; mode=block")
            }

            // Referrer-Policy
            // Controls how much referrer information is sent
            w.Header().Set("Referrer-Policy", config.ReferrerPolicy)

            // Permissions-Policy (formerly Feature-Policy)
            // Controls browser features (camera, microphone, etc.)
            w.Header().Set("Permissions-Policy", config.PermissionsPolicy)

            // Cache-Control for authenticated responses
            // Prevents sensitive data from being cached
            if r.Header.Get("Authorization") != "" {
                w.Header().Set("Cache-Control", "no-store, no-cache, must-revalidate, private")
                w.Header().Set("Pragma", "no-cache")
            }

            next.ServeHTTP(w, r)
        })
    }
}

func main() {
    config := DefaultSecurityConfig()

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprintln(w, "<html><body><h1>Secured!</h1></body></html>")
    })

    mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprintln(w, `{"message": "secure API response"}`)
    })

    // Wrap with security headers middleware
    handler := SecurityHeaders(config)(mux)

    fmt.Println("Server starting on :8080 with security headers")
    http.ListenAndServe(":8080", handler)
}
```

**Security headers reference:**

| Header | Purpose | Recommended Value |
|--------|---------|-------------------|
| Content-Security-Policy | Prevents XSS and data injection | `default-src 'self'` (customize per app) |
| Strict-Transport-Security | Forces HTTPS | `max-age=31536000; includeSubDomains; preload` |
| X-Frame-Options | Prevents clickjacking | `DENY` or `SAMEORIGIN` |
| X-Content-Type-Options | Prevents MIME sniffing | `nosniff` |
| Referrer-Policy | Controls referrer info | `strict-origin-when-cross-origin` |
| Permissions-Policy | Restricts browser APIs | Disable unused features |
| Cache-Control | Controls caching | `no-store` for sensitive responses |

---

## Rate Limiting to Prevent Abuse

Rate limiting is essential for preventing brute force attacks, denial of service, and API
abuse. Here are several approaches.

### Token Bucket Rate Limiter

```go
package main

import (
    "encoding/json"
    "fmt"
    "net"
    "net/http"
    "sync"
    "time"
)

// TokenBucket implements the token bucket rate limiting algorithm
type TokenBucket struct {
    mu         sync.Mutex
    tokens     float64
    maxTokens  float64
    refillRate float64   // tokens per second
    lastRefill time.Time
}

// NewTokenBucket creates a new token bucket
func NewTokenBucket(maxTokens, refillRate float64) *TokenBucket {
    return &TokenBucket{
        tokens:     maxTokens,
        maxTokens:  maxTokens,
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

// Allow checks if a request is allowed and consumes a token
func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    // Refill tokens based on elapsed time
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += elapsed * tb.refillRate
    if tb.tokens > tb.maxTokens {
        tb.tokens = tb.maxTokens
    }
    tb.lastRefill = now

    // Check if we have a token available
    if tb.tokens < 1 {
        return false
    }

    tb.tokens--
    return true
}

// RateLimiter manages per-IP rate limiting
type RateLimiter struct {
    mu        sync.RWMutex
    limiters  map[string]*TokenBucket
    maxTokens float64
    refillRate float64

    // Cleanup configuration
    cleanupInterval time.Duration
    maxAge          time.Duration
}

// NewRateLimiter creates a new rate limiter
func NewRateLimiter(maxTokens, refillRate float64) *RateLimiter {
    rl := &RateLimiter{
        limiters:        make(map[string]*TokenBucket),
        maxTokens:       maxTokens,
        refillRate:      refillRate,
        cleanupInterval: 1 * time.Minute,
        maxAge:          10 * time.Minute,
    }

    // Start background cleanup to prevent memory leaks
    go rl.cleanup()

    return rl
}

// GetLimiter returns the rate limiter for a given key (e.g., IP address)
func (rl *RateLimiter) GetLimiter(key string) *TokenBucket {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    if limiter, exists := rl.limiters[key]; exists {
        return limiter
    }

    limiter := NewTokenBucket(rl.maxTokens, rl.refillRate)
    rl.limiters[key] = limiter
    return limiter
}

// cleanup removes stale rate limiter entries
func (rl *RateLimiter) cleanup() {
    ticker := time.NewTicker(rl.cleanupInterval)
    defer ticker.Stop()

    for range ticker.C {
        rl.mu.Lock()
        now := time.Now()
        for key, limiter := range rl.limiters {
            limiter.mu.Lock()
            if now.Sub(limiter.lastRefill) > rl.maxAge {
                delete(rl.limiters, key)
            }
            limiter.mu.Unlock()
        }
        rl.mu.Unlock()
    }
}

// RateLimitMiddleware applies rate limiting per client IP
func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract client IP
            ip := getClientIP(r)

            // Check rate limit
            bucket := limiter.GetLimiter(ip)
            if !bucket.Allow() {
                w.Header().Set("Retry-After", "1")
                w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%.0f", limiter.maxTokens))
                w.Header().Set("X-RateLimit-Remaining", "0")
                w.WriteHeader(http.StatusTooManyRequests)
                json.NewEncoder(w).Encode(map[string]string{
                    "error": "Rate limit exceeded. Please try again later.",
                })
                return
            }

            // Add rate limit headers to successful responses
            w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%.0f", limiter.maxTokens))

            next.ServeHTTP(w, r)
        })
    }
}

// getClientIP extracts the real client IP, accounting for proxies
func getClientIP(r *http.Request) string {
    // Check X-Forwarded-For header (set by reverse proxies)
    // WARNING: Only trust this header if your server is behind a trusted proxy
    xff := r.Header.Get("X-Forwarded-For")
    if xff != "" {
        // Take the first IP (the original client)
        parts := splitAndTrim(xff, ",")
        if len(parts) > 0 {
            return parts[0]
        }
    }

    // Check X-Real-IP header
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

func splitAndTrim(s, sep string) []string {
    var result []string
    for _, part := range splitString(s, sep) {
        trimmed := trimSpace(part)
        if trimmed != "" {
            result = append(result, trimmed)
        }
    }
    return result
}

func splitString(s, sep string) []string {
    var result []string
    for {
        i := indexOf(s, sep)
        if i < 0 {
            result = append(result, s)
            break
        }
        result = append(result, s[:i])
        s = s[i+len(sep):]
    }
    return result
}

func indexOf(s, sub string) int {
    for i := 0; i <= len(s)-len(sub); i++ {
        if s[i:i+len(sub)] == sub {
            return i
        }
    }
    return -1
}

func trimSpace(s string) string {
    start := 0
    for start < len(s) && (s[start] == ' ' || s[start] == '\t') {
        start++
    }
    end := len(s)
    for end > start && (s[end-1] == ' ' || s[end-1] == '\t') {
        end--
    }
    return s[start:end]
}

// ========================================
// Login-specific rate limiting
// ========================================

// LoginRateLimiter provides stricter rate limiting for login attempts
type LoginRateLimiter struct {
    perIP       *RateLimiter // Rate limit per IP
    perUsername *RateLimiter // Rate limit per username
}

func NewLoginRateLimiter() *LoginRateLimiter {
    return &LoginRateLimiter{
        perIP:       NewRateLimiter(10, 0.1),   // 10 attempts, refill 1 per 10s
        perUsername: NewRateLimiter(5, 0.05),    // 5 attempts, refill 1 per 20s
    }
}

func (l *LoginRateLimiter) Allow(ip, username string) bool {
    // Both checks must pass
    return l.perIP.GetLimiter(ip).Allow() && l.perUsername.GetLimiter(username).Allow()
}

func main() {
    // General API rate limiter: 100 requests, refill 10 per second
    apiLimiter := NewRateLimiter(100, 10)

    // Login rate limiter: much stricter
    loginLimiter := NewLoginRateLimiter()

    mux := http.NewServeMux()

    mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"data": "Hello, World!"})
    })

    mux.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        ip := getClientIP(r)
        username := r.FormValue("username")

        if !loginLimiter.Allow(ip, username) {
            http.Error(w, "Too many login attempts", http.StatusTooManyRequests)
            return
        }

        // Process login...
        fmt.Fprintln(w, "Login endpoint")
    })

    // Apply general rate limiting to all routes
    handler := RateLimitMiddleware(apiLimiter)(mux)

    fmt.Println("Server starting on :8080 with rate limiting")
    http.ListenAndServe(":8080", handler)
}
```

### Using golang.org/x/time/rate (Standard Library Extension)

For simpler cases, use the official rate limiter from the Go team:

```go
package main

import (
    "fmt"
    "net/http"
    "sync"

    "golang.org/x/time/rate"
)

// IPRateLimiter uses golang.org/x/time/rate for per-IP limiting
type IPRateLimiter struct {
    mu       sync.RWMutex
    limiters map[string]*rate.Limiter
    rate     rate.Limit
    burst    int
}

func NewIPRateLimiter(r rate.Limit, burst int) *IPRateLimiter {
    return &IPRateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    burst,
    }
}

func (l *IPRateLimiter) GetLimiter(ip string) *rate.Limiter {
    l.mu.Lock()
    defer l.mu.Unlock()

    limiter, exists := l.limiters[ip]
    if !exists {
        limiter = rate.NewLimiter(l.rate, l.burst)
        l.limiters[ip] = limiter
    }

    return limiter
}

func main() {
    // Allow 5 requests per second with a burst of 10
    limiter := NewIPRateLimiter(5, 10)

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        if !limiter.GetLimiter(ip).Allow() {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        fmt.Fprintln(w, "OK")
    })

    http.ListenAndServe(":8080", mux)
}
```

---

## Dependency Vulnerability Scanning

### govulncheck

`govulncheck` is the official Go vulnerability scanner. It analyzes your code and
dependencies against the Go Vulnerability Database.

```bash
# Install govulncheck
go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan your project
govulncheck ./...

# Scan a specific binary
govulncheck -mode=binary ./myapp

# Output in JSON format (for CI integration)
govulncheck -json ./...
```

Example output:

```
Scanning your code and 245 packages across 38 dependent modules for known
vulnerabilities...

Vulnerability #1: GO-2024-2687
    HTTP/2 CONTINUATION frames can be exploited for DoS attacks
  More info: https://pkg.go.dev/vuln/GO-2024-2687
  Module: golang.org/x/net
    Found in: golang.org/x/net@v0.17.0
    Fixed in: golang.org/x/net@v0.23.0
  Example traces found:
    #1: cmd/server/main.go:45:12: server.Start calls http2.ConfigureServer

=== Informational ===

Found 1 vulnerability in packages that you import.
Use '-show verbose' for more details.
```

### Integrating govulncheck in CI/CD

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # Run daily at midnight UTC
    - cron: '0 0 * * *'

jobs:
  govulncheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Run govulncheck
        run: govulncheck ./...

  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Verify go.sum
        run: |
          go mod verify
          go mod tidy
          git diff --exit-code go.mod go.sum
```

### go.sum Verification

Go's module system provides built-in integrity verification through `go.sum`:

```bash
# Verify all downloaded modules match their checksums
go mod verify

# Tidy dependencies (remove unused, add missing)
go mod tidy

# Check for any inconsistencies
go mod verify
# all modules verified
```

> **Tip:** Always commit `go.sum` to your repository. It ensures that everyone building
> your code uses the exact same dependency versions with verified checksums.

---

## gosec Static Analysis

`gosec` (Go Security Checker) performs static analysis of Go code to find security issues.

### Installation and Usage

```bash
# Install gosec
go install github.com/securego/gosec/v2/cmd/gosec@latest

# Scan your project
gosec ./...

# Scan with specific rules
gosec -include=G101,G201,G301 ./...

# Exclude rules
gosec -exclude=G104 ./...

# Output in different formats
gosec -fmt=json -out=results.json ./...
gosec -fmt=sarif -out=results.sarif ./...

# Scan with severity filter
gosec -severity=medium ./...
```

### Key gosec Rules

| Rule ID | Description | Example |
|---------|-------------|---------|
| G101 | Hardcoded credentials | `password := "secret123"` |
| G102 | Bind to all interfaces | `http.ListenAndServe(":8080", nil)` |
| G103 | Use of unsafe package | `import "unsafe"` |
| G104 | Unhandled errors | `f.Close()` without checking error |
| G107 | URL provided to HTTP request | `http.Get(userInput)` |
| G108 | Profiling endpoint exposed | `import _ "net/http/pprof"` |
| G109 | Integer overflow conversion | `int32(bigInt)` |
| G110 | Potential DoS via decompression bomb | `io.Copy(w, gzipReader)` |
| G201 | SQL string concatenation | `db.Query("SELECT * WHERE id=" + id)` |
| G202 | SQL string formatting | `db.Query(fmt.Sprintf("...%s...", id))` |
| G301 | Poor file permissions on mkdir | `os.Mkdir(dir, 0777)` |
| G302 | Poor file permissions on create | `os.Create` with 0666 |
| G304 | File path from tainted input | `os.Open(r.URL.Query().Get("file"))` |
| G401 | Use of weak crypto (MD5, SHA1) | `md5.Sum(data)` |
| G402 | TLS with InsecureSkipVerify | `tls.Config{InsecureSkipVerify: true}` |
| G501 | Import of crypto blacklist | `import "crypto/des"` |
| G601 | Implicit memory aliasing in for loop | `for _, v := range s { use(&v) }` |

### Suppressing False Positives

```go
package main

import (
    "net/http"
)

func main() {
    // Suppress gosec warning with a comment
    // This is acceptable because we're behind a reverse proxy
    http.ListenAndServe(":8080", nil) // #nosec G102 -- behind reverse proxy
}
```

### Integrating gosec in CI

```yaml
# .github/workflows/gosec.yml
name: gosec

on:
  push:
    branches: [main]
  pull_request:

jobs:
  gosec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run gosec
        uses: securego/gosec@master
        with:
          args: '-fmt sarif -out results.sarif ./...'

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

---

## Secure Coding Checklist

Use this checklist for every Go application you build:

### Authentication and Session Management

- [ ] Hash passwords with Argon2id or bcrypt (minimum cost factor 12)
- [ ] Use `crypto/rand` for all tokens, session IDs, and random values
- [ ] Set secure cookie attributes: `HttpOnly`, `Secure`, `SameSite=Strict`
- [ ] Implement session expiration and rotation
- [ ] Use constant-time comparison for tokens and MACs
- [ ] Rate limit login endpoints (per IP and per username)
- [ ] Return generic error messages ("Invalid credentials", not "Wrong password")
- [ ] Implement account lockout after repeated failures
- [ ] Support multi-factor authentication for sensitive operations

### Input Validation

- [ ] Validate all input at the boundary (type, length, format, range)
- [ ] Use `json.Decoder.DisallowUnknownFields()` for JSON parsing
- [ ] Limit request body size with `http.MaxBytesReader`
- [ ] Sanitize input (remove control characters, normalize whitespace)
- [ ] Validate file uploads (type, size, content -- do not trust file extensions)
- [ ] Prevent path traversal in file operations
- [ ] Use parameterized queries for all SQL operations
- [ ] Use `html/template` (not `text/template`) for HTML output

### Cryptography

- [ ] Use `crypto/rand` -- never `math/rand` -- for security-sensitive values
- [ ] Use AES-256-GCM for symmetric encryption
- [ ] Use Ed25519 or RSA-PSS for digital signatures
- [ ] Never reuse nonces in AES-GCM
- [ ] Store encryption keys securely (Vault, KMS, not in code)
- [ ] Use HMAC with constant-time comparison for message authentication
- [ ] Do not implement your own cryptographic algorithms

### TLS and Network

- [ ] Enforce TLS 1.2 minimum (`MinVersion: tls.VersionTLS12`)
- [ ] Redirect HTTP to HTTPS
- [ ] Set HSTS headers with long max-age
- [ ] Use strong cipher suites (AES-GCM, ChaCha20-Poly1305)
- [ ] Verify TLS certificates (never use `InsecureSkipVerify: true` in production)
- [ ] Use mTLS for service-to-service communication
- [ ] Set appropriate timeouts on HTTP servers and clients

### HTTP Security

- [ ] Set Content-Security-Policy header
- [ ] Set X-Frame-Options header
- [ ] Set X-Content-Type-Options: nosniff
- [ ] Set Referrer-Policy header
- [ ] Set Permissions-Policy header
- [ ] Implement CSRF protection for state-changing endpoints
- [ ] Set Cache-Control: no-store for sensitive responses
- [ ] Validate Content-Type on incoming requests

### Error Handling and Logging

- [ ] Never expose stack traces or internal errors to users
- [ ] Log security events (failed logins, permission denials, input validation failures)
- [ ] Do not log sensitive data (passwords, tokens, credit cards, PII)
- [ ] Use structured logging for security events
- [ ] Handle all errors explicitly (check every `error` return)
- [ ] Set up alerting for security-related log patterns

### Dependencies and Build

- [ ] Run `govulncheck ./...` in CI
- [ ] Run `gosec ./...` in CI
- [ ] Run `go vet ./...` in CI
- [ ] Run `go test -race ./...` in CI
- [ ] Commit `go.sum` and verify with `go mod verify`
- [ ] Keep dependencies up to date
- [ ] Review dependency licenses and security posture
- [ ] Build with `CGO_ENABLED=0` when possible (smaller attack surface)
- [ ] Use minimal base images for containers (distroless, scratch)

### Secrets Management

- [ ] Never hardcode secrets in source code
- [ ] Use environment variables or a secrets manager (Vault, AWS Secrets Manager)
- [ ] Add `.env` to `.gitignore`
- [ ] Rotate secrets regularly
- [ ] Use different secrets for each environment (dev, staging, prod)
- [ ] Set restrictive file permissions on secret files (0600)

### Deployment

- [ ] Run as non-root user in containers
- [ ] Use read-only filesystem where possible
- [ ] Limit container capabilities
- [ ] Enable resource limits (CPU, memory)
- [ ] Disable debug endpoints in production (`net/http/pprof`)
- [ ] Implement health checks and readiness probes
- [ ] Use network policies to restrict service communication

---

## Common Go-Specific Security Mistakes

### Mistake 1: Using math/rand for Security

```go
package main

import (
    "crypto/rand"
    "encoding/hex"
    "fmt"
    mathrand "math/rand"
    "time"
)

func main() {
    // BAD: math/rand is deterministic and predictable
    mathrand.Seed(time.Now().UnixNano())
    badToken := fmt.Sprintf("%x", mathrand.Int63())
    fmt.Println("BAD token (predictable):", badToken)

    // An attacker who knows the approximate time your server started
    // can predict all tokens generated by math/rand.

    // GOOD: crypto/rand is cryptographically secure
    goodToken := make([]byte, 32)
    rand.Read(goodToken)
    fmt.Println("GOOD token (unpredictable):", hex.EncodeToString(goodToken))
}
```

### Mistake 2: Ignoring Errors

```go
package main

import (
    "encoding/json"
    "net/http"
    "os"
)

func main() {
    // BAD: Ignoring the error from json.Unmarshal
    var config map[string]string
    data, _ := os.ReadFile("config.json") // Ignoring error!
    json.Unmarshal(data, &config)          // Ignoring error!
    // config could be nil, leading to a panic

    // GOOD: Handle every error
    data, err := os.ReadFile("config.json")
    if err != nil {
        // Handle the error appropriately
        panic("failed to read config: " + err.Error())
    }

    if err := json.Unmarshal(data, &config); err != nil {
        panic("failed to parse config: " + err.Error())
    }

    // BAD: Ignoring http.ListenAndServe error
    http.ListenAndServe(":8080", nil)

    // GOOD: Log and exit on server startup failure
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic("server failed: " + err.Error())
    }
}
```

### Mistake 3: Leaking Goroutines

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

// BAD: This goroutine will leak if the context is never cancelled
func leakyHandler(w http.ResponseWriter, r *http.Request) {
    ch := make(chan string)

    go func() {
        // This goroutine will block forever if nobody reads from ch
        result := expensiveOperation()
        ch <- result // blocks forever if handler already returned
    }()

    select {
    case result := <-ch:
        fmt.Fprintln(w, result)
    case <-time.After(2 * time.Second):
        http.Error(w, "Timeout", http.StatusGatewayTimeout)
        // The goroutine is still running and will leak!
    }
}

// GOOD: Use context cancellation to clean up goroutines
func safeHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel() // Ensures cleanup

    ch := make(chan string, 1) // Buffered channel -- goroutine won't block

    go func() {
        result := expensiveOperationWithContext(ctx)
        select {
        case ch <- result:
        case <-ctx.Done():
            // Context was cancelled; discard the result
        }
    }()

    select {
    case result := <-ch:
        fmt.Fprintln(w, result)
    case <-ctx.Done():
        http.Error(w, "Timeout", http.StatusGatewayTimeout)
    }
}

func expensiveOperation() string {
    time.Sleep(5 * time.Second)
    return "result"
}

func expensiveOperationWithContext(ctx context.Context) string {
    select {
    case <-time.After(5 * time.Second):
        return "result"
    case <-ctx.Done():
        return ""
    }
}
```

### Mistake 4: Insecure TLS Configuration

```go
package main

import (
    "crypto/tls"
    "fmt"
    "net/http"
)

func main() {
    // BAD: Disabling TLS verification
    badClient := &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                InsecureSkipVerify: true, // NEVER do this in production!
            },
        },
    }
    _ = badClient

    // BAD: Allowing old TLS versions
    badTLS := &tls.Config{
        MinVersion: tls.VersionTLS10, // TLS 1.0 is broken!
    }
    _ = badTLS

    // GOOD: Proper TLS configuration
    goodTLS := &tls.Config{
        MinVersion: tls.VersionTLS12,
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP256,
        },
    }

    client := &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: goodTLS,
        },
    }

    resp, err := client.Get("https://example.com")
    if err != nil {
        fmt.Printf("Request failed: %v\n", err)
        return
    }
    defer resp.Body.Close()
    fmt.Printf("Status: %s, TLS: %d\n", resp.Status, resp.TLS.Version)
}
```

### Mistake 5: SQL Injection via String Formatting

```go
package main

import (
    "database/sql"
    "fmt"
    "net/http"
)

// BAD: Every one of these is vulnerable to SQL injection
func badQueries(db *sql.DB, userInput string) {
    // String concatenation
    db.Query("SELECT * FROM users WHERE name = '" + userInput + "'")

    // fmt.Sprintf
    db.Query(fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userInput))

    // String interpolation in ORDER BY (common mistake!)
    sortCol := userInput
    db.Query(fmt.Sprintf("SELECT * FROM users ORDER BY %s", sortCol))
}

// GOOD: Parameterized queries
func goodQueries(db *sql.DB, userInput string) {
    // Use placeholders for values
    db.Query("SELECT * FROM users WHERE name = $1", userInput)

    // For ORDER BY, use a whitelist
    allowedSortColumns := map[string]bool{
        "name":       true,
        "created_at": true,
        "email":      true,
    }

    sortCol := userInput
    if !allowedSortColumns[sortCol] {
        sortCol = "created_at" // default
    }
    // Column name is from whitelist, not user input -- safe
    db.Query(fmt.Sprintf("SELECT * FROM users ORDER BY %s", sortCol))
}
```

### Mistake 6: Exposing Debug Endpoints in Production

```go
package main

import (
    "net/http"
    // BAD: Importing pprof in production code exposes profiling endpoints
    // _ "net/http/pprof"
)

// GOOD: Only expose pprof behind authentication and in non-production
func setupDebugEndpoints(mux *http.ServeMux, enableDebug bool) {
    if !enableDebug {
        return
    }

    // Protect debug endpoints with authentication
    debugMux := http.NewServeMux()
    // Register pprof handlers on the debug mux...

    mux.Handle("/debug/", requireAuth(debugMux))
}

func requireAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Verify admin authentication before allowing access
        token := r.Header.Get("Authorization")
        if token != "Bearer admin-debug-token" {
            http.Error(w, "Forbidden", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

### Mistake 7: Race Conditions in Shared State

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

// BAD: Unsynchronized access to shared map
type UnsafeCache struct {
    data map[string]string
}

func (c *UnsafeCache) Get(key string) string {
    return c.data[key] // DATA RACE
}

func (c *UnsafeCache) Set(key, value string) {
    c.data[key] = value // DATA RACE -- concurrent map writes will panic
}

// GOOD: Option 1 -- sync.RWMutex for read-heavy workloads
type SafeCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewSafeCache() *SafeCache {
    return &SafeCache{data: make(map[string]string)}
}

func (c *SafeCache) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

// GOOD: Option 2 -- sync.Map for high-concurrency scenarios
type ConcurrentCache struct {
    data sync.Map
}

func (c *ConcurrentCache) Get(key string) (string, bool) {
    value, ok := c.data.Load(key)
    if !ok {
        return "", false
    }
    return value.(string), true
}

func (c *ConcurrentCache) Set(key, value string) {
    c.data.Store(key, value)
}

// GOOD: Option 3 -- atomic for simple counters
type SafeCounter struct {
    count atomic.Int64
}

func (c *SafeCounter) Increment() {
    c.count.Add(1)
}

func (c *SafeCounter) Value() int64 {
    return c.count.Load()
}

func main() {
    cache := NewSafeCache()
    cache.Set("key", "value")
    fmt.Println(cache.Get("key"))

    counter := &SafeCounter{}
    counter.Increment()
    fmt.Println(counter.Value())
}
```

### Mistake 8: Timing Attacks in String Comparison

```go
package main

import (
    "crypto/hmac"
    "crypto/subtle"
)

// BAD: Standard string comparison leaks timing information
func vulnerableCompare(provided, expected string) bool {
    return provided == expected
    // An attacker can determine the correct value one byte at a time:
    // "a..." is faster to reject than "correct_prefix_wrong_suffix"
}

// GOOD: Constant-time comparison for secrets
func safeCompareStrings(a, b string) bool {
    // subtle.ConstantTimeCompare only returns 1 if the slices are equal
    return subtle.ConstantTimeCompare([]byte(a), []byte(b)) == 1
}

// GOOD: For HMAC comparison, use hmac.Equal
func safeCompareMAC(mac1, mac2 []byte) bool {
    return hmac.Equal(mac1, mac2)
}
```

### Mistake 9: Logging Sensitive Information

```go
package main

import (
    "log/slog"
    "net/http"
    "strings"
)

// BAD: Logging sensitive data
func badLogging(r *http.Request) {
    slog.Info("Login attempt",
        "username", r.FormValue("username"),
        "password", r.FormValue("password"), // NEVER log passwords!
        "token", r.Header.Get("Authorization"), // NEVER log tokens!
    )
}

// GOOD: Redact sensitive fields
func goodLogging(r *http.Request) {
    slog.Info("Login attempt",
        "username", r.FormValue("username"),
        "password", "[REDACTED]",
        "has_token", r.Header.Get("Authorization") != "",
    )
}

// GOOD: Use a redacting logger
func redactHeader(header string) string {
    if len(header) <= 8 {
        return "[REDACTED]"
    }
    return header[:4] + "..." + header[len(header)-4:]
}

// SensitiveString is a type that redacts itself when logged
type SensitiveString string

func (s SensitiveString) String() string {
    return "[REDACTED]"
}

func (s SensitiveString) LogValue() slog.Value {
    return slog.StringValue("[REDACTED]")
}

// MaskedString partially reveals a string for debugging
type MaskedString string

func (m MaskedString) LogValue() slog.Value {
    s := string(m)
    if len(s) <= 4 {
        return slog.StringValue(strings.Repeat("*", len(s)))
    }
    return slog.StringValue(s[:2] + strings.Repeat("*", len(s)-4) + s[len(s)-2:])
}
```

### Mistake 10: Unbounded Resource Consumption

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

// BAD: No limits on request body size
func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body) // Could read gigabytes into memory!
    var data map[string]interface{}
    json.Unmarshal(body, &data)
    fmt.Fprintln(w, "OK")
}

// BAD: No timeout on HTTP client
func vulnerableClient() {
    // Default http.Client has no timeout!
    // A malicious server could keep the connection open forever
    resp, _ := http.Get("https://evil.example.com/slow")
    defer resp.Body.Close()
    io.ReadAll(resp.Body) // Could block forever
}

// GOOD: Limit everything
func safeHandler(w http.ResponseWriter, r *http.Request) {
    // Limit request body size
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MB max

    // Use a decoder with limits
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()

    var data struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }

    if err := decoder.Decode(&data); err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }

    fmt.Fprintln(w, "OK")
}

// GOOD: Always set timeouts on HTTP clients
func safeClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second, // Total request timeout
        Transport: &http.Transport{
            TLSHandshakeTimeout:   10 * time.Second,
            ResponseHeaderTimeout: 10 * time.Second,
            IdleConnTimeout:       90 * time.Second,
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   10,
        },
    }
}
```

> **Note:** The above `safeClient()` function references `time.Second` but has the
> `"time"` import omitted for brevity. In real code, always include the import.

---

## Summary

Secure coding in Go is about leveraging the language's built-in safety features while being
vigilant about the areas where Go cannot protect you automatically. Here are the key
takeaways:

**Go gives you a head start:**
- Memory safety eliminates buffer overflows, use-after-free, and uninitialized reads
- Strong typing catches type-confusion bugs at compile time
- Zero values prevent uninitialized variable vulnerabilities
- The race detector catches concurrency bugs
- Static binaries reduce runtime attack surface

**But you still need to:**
- Use parameterized SQL queries (never string concatenation)
- Use `html/template` for HTML output (never `text/template`)
- Use `crypto/rand` for security (never `math/rand`)
- Hash passwords with Argon2id or bcrypt (never SHA-256 or MD5)
- Configure TLS properly (TLS 1.2+, strong ciphers, verify certificates)
- Validate and sanitize all input at the boundary
- Set security headers on all HTTP responses
- Rate limit endpoints to prevent abuse
- Manage secrets securely (never hardcode them)
- Scan dependencies with govulncheck and code with gosec
- Handle every error explicitly
- Use constant-time comparison for all secret comparisons
- Set timeouts and size limits on all I/O operations

**Tools to run in every CI pipeline:**

```bash
# Static analysis
go vet ./...

# Race detection
go test -race ./...

# Vulnerability scanning
govulncheck ./...

# Security linting
gosec ./...

# Dependency verification
go mod verify
```

Security is not a one-time task but an ongoing practice. Stay up to date with the Go
security announcements, keep your dependencies current, and regularly review your code
against the checklist in this chapter.

---

**Further Reading:**

- [Go Security Policy](https://go.dev/security/policy)
- [Go Vulnerability Database](https://vuln.go.dev)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Go Secure Coding Practices](https://owasp.org/www-project-go-secure-coding-practices-guide/)
- [CWE/SANS Top 25](https://cwe.mitre.org/top25/archive/2023/2023_top25_list.html)
- [gosec Rules Reference](https://github.com/securego/gosec#available-rules)
- [Go Cryptography Documentation](https://pkg.go.dev/crypto)
