# Chapter 12: Working with JSON, HTTP & REST APIs

## Prerequisites

You should be comfortable with Go structs, interfaces, error handling, and goroutines before diving into this chapter. If you have experience building APIs in Node.js/Express, the comparisons throughout will be especially valuable.

---

## Table of Contents

1. [JSON Encoding and Decoding](#1-json-encoding-and-decoding)
2. [JSON Streaming](#2-json-streaming)
3. [Custom JSON Marshaling](#3-custom-json-marshaling)
4. [HTTP Client](#4-http-client)
5. [HTTP Server](#5-http-server)
6. [Building a Complete REST API](#6-building-a-complete-rest-api)
7. [The Middleware Pattern](#7-the-middleware-pattern)
8. [Popular Router Libraries](#8-popular-router-libraries)
9. [Request Validation](#9-request-validation)
10. [Error Responses](#10-error-responses)
11. [Production Considerations](#11-production-considerations)
12. [Go vs Express.js -- Full Comparison](#12-go-vs-expressjs----full-comparison)
13. [Key Takeaways](#13-key-takeaways)
14. [Practice Exercises](#14-practice-exercises)

---

## 1. JSON Encoding and Decoding

### The `encoding/json` Package

Go's standard library ships with `encoding/json`, a fully featured JSON package that requires zero third-party dependencies. This is one area where Go and JavaScript differ fundamentally: JavaScript objects map naturally to JSON (since JSON *is* JavaScript Object Notation), while Go must bridge the gap between its statically typed structs and the loosely typed JSON format.

### Marshal: Go Structs to JSON

`json.Marshal` converts a Go value into a JSON byte slice.

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID        int    `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Email     string `json:"email"`
    Age       int    `json:"age"`
    IsActive  bool   `json:"is_active"`
}

func main() {
    user := User{
        ID:        1,
        FirstName: "Alice",
        LastName:  "Johnson",
        Email:     "alice@example.com",
        Age:       30,
        IsActive:  true,
    }

    // json.Marshal returns []byte and error
    data, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error marshaling:", err)
        return
    }

    fmt.Println(string(data))
    // {"id":1,"first_name":"Alice","last_name":"Johnson","email":"alice@example.com","age":30,"is_active":true}

    // For human-readable output, use MarshalIndent
    pretty, err := json.MarshalIndent(user, "", "  ")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println(string(pretty))
    // {
    //   "id": 1,
    //   "first_name": "Alice",
    //   ...
    // }
}
```

### Unmarshal: JSON to Go Structs

`json.Unmarshal` parses a JSON byte slice and stores the result in a Go value.

```go
func main() {
    jsonData := []byte(`{
        "id": 2,
        "first_name": "Bob",
        "last_name": "Smith",
        "email": "bob@example.com",
        "age": 25,
        "is_active": false
    }`)

    var user User
    err := json.Unmarshal(jsonData, &user)
    if err != nil {
        fmt.Println("Error unmarshaling:", err)
        return
    }

    fmt.Printf("%+v\n", user)
    // {ID:2 FirstName:Bob LastName:Smith Email:bob@example.com Age:25 IsActive:false}
}
```

Notice you pass a *pointer* (`&user`) to `Unmarshal`. This is how Go fills in the struct -- it needs the address to write to.

### WHY Go Uses Struct Tags for Serialization

This is a question every JavaScript developer asks. In JavaScript, object keys *are* the JSON keys:

```javascript
// JavaScript -- keys just work
const user = { first_name: "Alice", age: 30 };
JSON.stringify(user); // '{"first_name":"Alice","age":30}'
```

Go cannot do this because:

1. **Go field names follow PascalCase convention** (`FirstName`, not `first_name`). APIs typically use `snake_case` or `camelCase`. Struct tags bridge this naming gap.

2. **Go is statically typed and compiled.** The compiler needs to know at compile time what type each field is. Struct tags are metadata that the `encoding/json` package reads at runtime via reflection, without affecting the type system.

3. **Struct tags give fine-grained control.** You can rename fields, omit empty values, skip fields entirely, or embed raw JSON -- all with tag annotations.

```go
type Product struct {
    ID          int     `json:"id"`                    // Rename to "id"
    Name        string  `json:"name"`                  // Rename to "name"
    Price       float64 `json:"price"`                 // Rename to "price"
    Description string  `json:"description,omitempty"` // Omit if empty string
    InternalRef string  `json:"-"`                     // Never include in JSON
    SKU         string  `json:"sku,omitempty"`         // Omit if empty string
}

func main() {
    p := Product{
        ID:          1,
        Name:        "Keyboard",
        Price:       79.99,
        Description: "", // empty -- will be omitted
        InternalRef: "secret-ref-123",
        SKU:         "",
    }

    data, _ := json.MarshalIndent(p, "", "  ")
    fmt.Println(string(data))
    // {
    //   "id": 1,
    //   "name": "Keyboard",
    //   "price": 79.99
    // }
    // Notice: description (empty), InternalRef (skipped with "-"), and sku (empty) are all absent.
}
```

### Common Struct Tag Options

| Tag | Meaning |
|-----|---------|
| `` `json:"name"` `` | Use "name" as the JSON key |
| `` `json:"name,omitempty"` `` | Omit if zero value (0, "", nil, false, empty slice/map) |
| `` `json:"-"` `` | Always omit this field from JSON |
| `` `json:",string"` `` | Encode the value as a JSON string (for numbers/bools) |

### Unmarshaling into `map[string]interface{}`

When you do not know the JSON structure ahead of time, you can unmarshal into a generic map. This is the closest equivalent to how JavaScript handles JSON natively.

```go
func main() {
    jsonData := []byte(`{"name":"Widget","price":9.99,"tags":["sale","new"]}`)

    var result map[string]interface{}
    err := json.Unmarshal(jsonData, &result)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    fmt.Println(result["name"])  // Widget (type: string)
    fmt.Println(result["price"]) // 9.99 (type: float64 -- ALL JSON numbers become float64)

    // Type assertion needed to work with values
    tags, ok := result["tags"].([]interface{})
    if ok {
        for _, tag := range tags {
            fmt.Println(tag.(string))
        }
    }
}
```

> **Important:** When unmarshaling into `interface{}`, JSON numbers always become `float64`. If you expect an integer, you need to convert explicitly. This is a common source of bugs.

### Comparison: Go vs JavaScript JSON Handling

```go
// Go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

data, err := json.Marshal(User{Name: "Alice", Email: "a@b.com"})
// data is []byte, err may be non-nil

var u User
err = json.Unmarshal([]byte(`{"name":"Alice","email":"a@b.com"}`), &u)
// Must handle err
```

```javascript
// JavaScript
const user = { name: "Alice", email: "a@b.com" };

const data = JSON.stringify(user);
// data is a string; never fails for plain objects

const u = JSON.parse('{"name":"Alice","email":"a@b.com"}');
// u is a plain object; throws on invalid JSON but no type checking
```

Key differences:

| Aspect | Go | JavaScript |
|--------|-----|------------|
| Type safety | Fields are typed; unknown fields are silently ignored | No type checking; any JSON shape is accepted |
| Error handling | Returns error; you must check it | Throws exception on parse failure |
| Zero values | Missing JSON fields get the type's zero value | Missing fields are `undefined` |
| Output format | `[]byte` | `string` |
| Pretty print | `json.MarshalIndent` | `JSON.stringify(obj, null, 2)` |

---

## 2. JSON Streaming

### The Problem with `Marshal`/`Unmarshal`

`json.Marshal` and `json.Unmarshal` work with entire byte slices in memory. For a 500MB JSON file or a continuous stream of JSON objects from a network connection, loading everything into memory first is wasteful or impossible.

### `json.Decoder` -- Reading JSON Streams

`json.NewDecoder` wraps an `io.Reader` and decodes JSON tokens or objects one at a time.

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

type LogEntry struct {
    Timestamp string `json:"timestamp"`
    Level     string `json:"level"`
    Message   string `json:"message"`
}

func main() {
    // Simulate a stream of newline-delimited JSON (NDJSON)
    stream := strings.NewReader(`
{"timestamp":"2026-01-01T00:00:00Z","level":"info","message":"Server started"}
{"timestamp":"2026-01-01T00:00:01Z","level":"warn","message":"High memory usage"}
{"timestamp":"2026-01-01T00:00:02Z","level":"error","message":"Database connection lost"}
`)

    decoder := json.NewDecoder(stream)

    for decoder.More() {
        var entry LogEntry
        if err := decoder.Decode(&entry); err != nil {
            fmt.Println("Decode error:", err)
            break
        }
        fmt.Printf("[%s] %s: %s\n", entry.Timestamp, entry.Level, entry.Message)
    }
}
```

### `json.Encoder` -- Writing JSON Streams

`json.NewEncoder` wraps an `io.Writer` and encodes values directly to the stream.

```go
package main

import (
    "encoding/json"
    "os"
)

type Event struct {
    Type    string `json:"type"`
    Payload string `json:"payload"`
}

func main() {
    encoder := json.NewEncoder(os.Stdout)
    encoder.SetIndent("", "  ") // Optional: pretty print

    events := []Event{
        {Type: "click", Payload: "button_1"},
        {Type: "scroll", Payload: "page_top"},
    }

    for _, event := range events {
        if err := encoder.Encode(event); err != nil {
            panic(err)
        }
    }
}
```

### WHY Streaming Matters for HTTP Servers

When your HTTP handler receives a request body, `r.Body` is an `io.Reader` -- it is already a stream. Using `json.NewDecoder(r.Body)` reads directly from the network socket without buffering the entire body first.

```go
// BAD: reads entire body into memory, then parses
func handler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body) // Entire body in memory
    var user User
    json.Unmarshal(body, &user)   // Parses the in-memory copy
}

// GOOD: decodes directly from the stream
func handler(w http.ResponseWriter, r *http.Request) {
    var user User
    err := json.NewDecoder(r.Body).Decode(&user)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
}
```

The streaming approach is what you will see in production Go code. It uses less memory and starts processing sooner.

### Token-Level Parsing

For very large JSON structures where you only need certain fields, you can read individual tokens:

```go
func extractNames(r io.Reader) ([]string, error) {
    dec := json.NewDecoder(r)
    var names []string

    for {
        t, err := dec.Token()
        if err != nil {
            break
        }

        // Look for the "name" key
        if key, ok := t.(string); ok && key == "name" {
            // The next token is the value
            t, err = dec.Token()
            if err != nil {
                break
            }
            if name, ok := t.(string); ok {
                names = append(names, name)
            }
        }
    }
    return names, nil
}
```

### Comparison with Node.js Streaming

In Node.js, JSON streaming requires third-party packages like `stream-json`:

```javascript
// Node.js -- requires: npm install stream-json
const { parser } = require('stream-json');
const { streamValues } = require('stream-json/streamers/StreamValues');
const fs = require('fs');

const pipeline = fs.createReadStream('large.json')
  .pipe(parser())
  .pipe(streamValues());

pipeline.on('data', ({ value }) => {
  console.log(value);
});
```

Go's streaming is built into the standard library and works with any `io.Reader`/`io.Writer`, which means it plugs directly into files, network sockets, HTTP bodies, and any other I/O source without extra dependencies.

---

## 3. Custom JSON Marshaling

### The `json.Marshaler` and `json.Unmarshaler` Interfaces

Sometimes the default serialization is not enough. Go lets you take full control by implementing two interfaces:

```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
    UnmarshalJSON([]byte) error
}
```

### Example: Custom Time Format

Go's `time.Time` serializes to RFC 3339 by default. Suppose your API requires `YYYY-MM-DD`:

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

// DateOnly wraps time.Time with a custom JSON format
type DateOnly struct {
    time.Time
}

const dateFormat = "2006-01-02"

func (d DateOnly) MarshalJSON() ([]byte, error) {
    formatted := fmt.Sprintf(`"%s"`, d.Time.Format(dateFormat))
    return []byte(formatted), nil
}

func (d *DateOnly) UnmarshalJSON(data []byte) error {
    // Remove quotes from the JSON string
    str := string(data)
    if str == "null" {
        return nil
    }
    str = str[1 : len(str)-1] // strip the surrounding quotes

    parsed, err := time.Parse(dateFormat, str)
    if err != nil {
        return fmt.Errorf("invalid date format, expected YYYY-MM-DD: %w", err)
    }
    d.Time = parsed
    return nil
}

type Event struct {
    Name string   `json:"name"`
    Date DateOnly `json:"date"`
}

func main() {
    event := Event{
        Name: "Go Conference",
        Date: DateOnly{time.Date(2026, 6, 15, 0, 0, 0, 0, time.UTC)},
    }

    data, _ := json.Marshal(event)
    fmt.Println(string(data))
    // {"name":"Go Conference","date":"2026-06-15"}

    // Unmarshal
    var parsed Event
    json.Unmarshal([]byte(`{"name":"Workshop","date":"2026-12-25"}`), &parsed)
    fmt.Println(parsed.Date.Format("January 2, 2006"))
    // December 25, 2026
}
```

### Example: Enum-Style Marshaling

```go
type Status int

const (
    StatusActive Status = iota
    StatusInactive
    StatusBanned
)

var statusLabels = map[Status]string{
    StatusActive:   "active",
    StatusInactive: "inactive",
    StatusBanned:   "banned",
}

var statusValues = map[string]Status{
    "active":   StatusActive,
    "inactive": StatusInactive,
    "banned":   StatusBanned,
}

func (s Status) MarshalJSON() ([]byte, error) {
    label, ok := statusLabels[s]
    if !ok {
        return nil, fmt.Errorf("unknown status: %d", s)
    }
    return json.Marshal(label)
}

func (s *Status) UnmarshalJSON(data []byte) error {
    var label string
    if err := json.Unmarshal(data, &label); err != nil {
        return err
    }
    val, ok := statusValues[label]
    if !ok {
        return fmt.Errorf("unknown status label: %s", label)
    }
    *s = val
    return nil
}
```

### Comparison with JavaScript

JavaScript's `JSON.stringify` calls `toJSON()` if it exists on an object, and `JSON.parse` accepts a *reviver* function:

```javascript
// JavaScript custom serialization
class DateOnly {
  constructor(date) {
    this.date = date;
  }
  toJSON() {
    return this.date.toISOString().split('T')[0];
  }
}

// Custom deserialization with a reviver
JSON.parse('{"date":"2026-06-15"}', (key, value) => {
  if (key === 'date') return new Date(value);
  return value;
});
```

The Go approach is more structured: each type declares its own serialization behavior through interface implementation, rather than relying on a global reviver function. This makes the behavior predictable and localized to the type.

---

## 4. HTTP Client

### Making Requests with `net/http`

Go's `net/http` package provides a full-featured HTTP client. No `axios`, no `node-fetch` -- it is all in the standard library.

### Simple GET Request

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

type Post struct {
    UserID int    `json:"userId"`
    ID     int    `json:"id"`
    Title  string `json:"title"`
    Body   string `json:"body"`
}

func main() {
    resp, err := http.Get("https://jsonplaceholder.typicode.com/posts/1")
    if err != nil {
        fmt.Println("Request failed:", err)
        return
    }
    defer resp.Body.Close() // ALWAYS close the body

    // Check status code
    if resp.StatusCode != http.StatusOK {
        fmt.Printf("Unexpected status: %d\n", resp.StatusCode)
        return
    }

    // Decode JSON from the response body stream
    var post Post
    if err := json.NewDecoder(resp.Body).Decode(&post); err != nil {
        fmt.Println("Decode error:", err)
        return
    }

    fmt.Printf("Title: %s\n", post.Title)
}
```

> **Critical:** Always call `resp.Body.Close()`. If you do not, the underlying TCP connection will never be returned to the connection pool, and your application will leak file descriptors and eventually crash.

### POST Request with JSON Body

```go
func createPost(title, body string, userID int) (*Post, error) {
    payload := Post{
        Title:  title,
        Body:   body,
        UserID: userID,
    }

    jsonData, err := json.Marshal(payload)
    if err != nil {
        return nil, fmt.Errorf("marshal error: %w", err)
    }

    resp, err := http.Post(
        "https://jsonplaceholder.typicode.com/posts",
        "application/json",
        bytes.NewBuffer(jsonData),
    )
    if err != nil {
        return nil, fmt.Errorf("request error: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("unexpected status %d: %s", resp.StatusCode, body)
    }

    var created Post
    if err := json.NewDecoder(resp.Body).Decode(&created); err != nil {
        return nil, fmt.Errorf("decode error: %w", err)
    }

    return &created, nil
}
```

### Custom Client with Timeouts

The `http.DefaultClient` has no timeout, which means a request can hang forever. **Never use the default client in production.**

```go
func main() {
    client := &http.Client{
        Timeout: 10 * time.Second, // Total request timeout
    }

    resp, err := client.Get("https://jsonplaceholder.typicode.com/posts/1")
    if err != nil {
        // err will mention context deadline exceeded if timeout fires
        fmt.Println("Request failed:", err)
        return
    }
    defer resp.Body.Close()

    // ...process response
}
```

### Full Control with `http.NewRequest`

For PUT, DELETE, or requests needing custom headers:

```go
func updatePost(id int, title string) error {
    payload, _ := json.Marshal(map[string]string{"title": title})

    req, err := http.NewRequest(
        http.MethodPut,
        fmt.Sprintf("https://jsonplaceholder.typicode.com/posts/%d", id),
        bytes.NewBuffer(payload),
    )
    if err != nil {
        return fmt.Errorf("creating request: %w", err)
    }

    // Set headers
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer my-token")
    req.Header.Set("X-Request-ID", "abc-123")

    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("executing request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("status %d: %s", resp.StatusCode, body)
    }

    return nil
}
```

### Connection Pooling

Go's `http.Client` automatically maintains a pool of TCP connections via the default `http.Transport`. You get keep-alive and connection reuse for free.

```go
// You can tune the transport for high-throughput scenarios
transport := &http.Transport{
    MaxIdleConns:        100,              // Max idle connections across all hosts
    MaxIdleConnsPerHost: 10,               // Max idle connections per host
    IdleConnTimeout:     90 * time.Second, // How long idle connections stay open
}

client := &http.Client{
    Transport: transport,
    Timeout:   10 * time.Second,
}

// Reuse this client for all requests -- do NOT create a new client per request
```

### Comparison with Node.js

```javascript
// Node.js (using fetch, available natively since Node 18)
const response = await fetch('https://jsonplaceholder.typicode.com/posts/1');
const post = await response.json();

// POST
const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: 'Hello', body: 'World', userId: 1 }),
});

// Timeout (using AbortController)
const controller = new AbortController();
setTimeout(() => controller.abort(), 10000);
const response = await fetch(url, { signal: controller.signal });
```

| Aspect | Go `net/http` | Node.js `fetch` / `axios` |
|--------|---------------|---------------------------|
| Dependencies | None (standard library) | `fetch` is built-in since Node 18; `axios` is third-party |
| Timeout | `client.Timeout` field | `AbortController` or library option |
| Connection pooling | Built-in via `http.Transport` | Built-in via Node's agent |
| Streaming | Response body is `io.Reader` | Response body is a `ReadableStream` |
| Error on non-2xx | No; you check `resp.StatusCode` manually | `fetch` does not throw; `axios` throws |
| Body closing | Manual: `defer resp.Body.Close()` | Automatic garbage collection |

---

## 5. HTTP Server

### The Simplest Server

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    fmt.Println("Server listening on :8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        fmt.Println("Server error:", err)
    }
}
```

That is a complete, working HTTP server. No frameworks, no imports beyond the standard library.

### How `net/http` Works Internally

Understanding the internals helps you see why Go's HTTP server is production-grade:

1. **`http.ListenAndServe(addr, handler)`** opens a TCP socket and enters an accept loop.

2. **For each incoming connection**, the server spawns a new goroutine. This is the fundamental difference from Node.js: Go uses **one goroutine per connection** rather than an event loop. Goroutines are cheap (a few KB of stack), so handling 10,000 concurrent connections is routine.

3. **The handler** processes the request. If `handler` is `nil`, Go uses `http.DefaultServeMux`, the default request multiplexer (router).

4. **`http.ResponseWriter`** is an interface for writing the response. It buffers headers until you write the body, then flushes everything to the connection.

5. **`*http.Request`** contains the parsed request: method, URL, headers, body (as `io.ReadCloser`), query parameters, and more.

### The `http.Handler` Interface

This is the most important interface in Go's HTTP ecosystem:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

Every HTTP framework, middleware, and router in Go implements this single interface. It is Go's equivalent of Express's `(req, res, next)` signature, but as a formal interface.

Any type with a `ServeHTTP` method is an `http.Handler`:

```go
type HealthHandler struct{}

func (h HealthHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
}

func main() {
    http.Handle("/health", HealthHandler{})
    http.ListenAndServe(":8080", nil)
}
```

### `http.HandlerFunc` -- The Adapter

Writing a struct for every handler would be tedious. `http.HandlerFunc` is an adapter that lets you use ordinary functions:

```go
// HandlerFunc is defined in the standard library as:
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

This is why `http.HandleFunc` works -- it wraps your function into a `HandlerFunc`, which satisfies the `Handler` interface.

### `ServeMux` -- The Built-in Router

`http.ServeMux` is Go's built-in request multiplexer. As of **Go 1.22**, it supports method-based routing and path parameters:

```go
func main() {
    mux := http.NewServeMux()

    // Go 1.22+ pattern syntax
    mux.HandleFunc("GET /api/users", listUsers)
    mux.HandleFunc("POST /api/users", createUser)
    mux.HandleFunc("GET /api/users/{id}", getUser)
    mux.HandleFunc("PUT /api/users/{id}", updateUser)
    mux.HandleFunc("DELETE /api/users/{id}", deleteUser)

    fmt.Println("Server listening on :8080")
    http.ListenAndServe(":8080", mux)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+: extract path parameter
    fmt.Fprintf(w, "User ID: %s", id)
}
```

> **Historical note:** Before Go 1.22, `ServeMux` only matched URL paths without method awareness and without path parameters. This was the primary reason developers reached for third-party routers. Since Go 1.22, the standard `ServeMux` covers the vast majority of routing needs.

### WHY Go's Standard Library HTTP Server Is Production-Ready

This surprises many Node.js developers. In Node.js, you would never deploy the raw `http` module -- you use Express, Fastify, or similar frameworks. In Go, the standard library server is genuinely production-ready:

1. **Concurrency model.** One goroutine per connection with efficient scheduling. No callback hell, no event loop bottlenecks on CPU-bound work.

2. **HTTP/2 support.** Built into `net/http` when using TLS (`http.ListenAndServeTLS`).

3. **Graceful shutdown.** `http.Server` has a `Shutdown` method.

4. **Timeouts.** `http.Server` exposes `ReadTimeout`, `WriteTimeout`, `IdleTimeout`, and `ReadHeaderTimeout`.

5. **TLS.** First-class support via `crypto/tls`.

6. **Battle-tested.** Used in production by Google, Docker, Kubernetes, and countless other projects.

### Comparison: Go vs Express.js

```go
// Go -- complete server, zero dependencies
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{"message": "Hello from Go"})
    })

    fmt.Println("Go server on :8080")
    http.ListenAndServe(":8080", mux)
}
```

```javascript
// Express.js -- requires: npm install express
const express = require('express');
const app = express();

app.get('/api/hello', (req, res) => {
  res.json({ message: 'Hello from Express' });
});

app.listen(8080, () => {
  console.log('Express server on :8080');
});
```

Both are approximately the same amount of code. The difference is that the Go version has zero external dependencies, compiles to a single binary, handles concurrent requests natively with goroutines, and includes production features like timeouts and graceful shutdown.

---

## 6. Building a Complete REST API

Let us build a full CRUD REST API for managing books. We will build it step by step using only the standard library.

### Step 1: Define the Data Model

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "sync"
    "time"
)

// Book represents a book in our collection.
type Book struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Author    string    `json:"author"`
    Year      int       `json:"year"`
    ISBN      string    `json:"isbn,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// CreateBookRequest is the expected JSON body for creating a book.
type CreateBookRequest struct {
    Title  string `json:"title"`
    Author string `json:"author"`
    Year   int    `json:"year"`
    ISBN   string `json:"isbn,omitempty"`
}

// UpdateBookRequest is the expected JSON body for updating a book.
type UpdateBookRequest struct {
    Title  *string `json:"title,omitempty"`  // Pointer: nil means "not provided"
    Author *string `json:"author,omitempty"`
    Year   *int    `json:"year,omitempty"`
    ISBN   *string `json:"isbn,omitempty"`
}
```

> **Design note:** `UpdateBookRequest` uses pointer fields. This lets us distinguish between "the client sent an empty string" vs "the client did not include this field at all." With a plain `string`, both cases would be the zero value `""`.

### Step 2: In-Memory Store with Concurrency Safety

```go
// BookStore is a thread-safe in-memory store.
type BookStore struct {
    mu     sync.RWMutex
    books  map[int]Book
    nextID int
}

func NewBookStore() *BookStore {
    return &BookStore{
        books:  make(map[int]Book),
        nextID: 1,
    }
}

func (s *BookStore) GetAll() []Book {
    s.mu.RLock()
    defer s.mu.RUnlock()

    books := make([]Book, 0, len(s.books))
    for _, b := range s.books {
        books = append(books, b)
    }
    return books
}

func (s *BookStore) GetByID(id int) (Book, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    book, ok := s.books[id]
    return book, ok
}

func (s *BookStore) Create(req CreateBookRequest) Book {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now().UTC()
    book := Book{
        ID:        s.nextID,
        Title:     req.Title,
        Author:    req.Author,
        Year:      req.Year,
        ISBN:      req.ISBN,
        CreatedAt: now,
        UpdatedAt: now,
    }
    s.books[s.nextID] = book
    s.nextID++
    return book
}

func (s *BookStore) Update(id int, req UpdateBookRequest) (Book, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()

    book, ok := s.books[id]
    if !ok {
        return Book{}, false
    }

    if req.Title != nil {
        book.Title = *req.Title
    }
    if req.Author != nil {
        book.Author = *req.Author
    }
    if req.Year != nil {
        book.Year = *req.Year
    }
    if req.ISBN != nil {
        book.ISBN = *req.ISBN
    }
    book.UpdatedAt = time.Now().UTC()
    s.books[id] = book
    return book, true
}

func (s *BookStore) Delete(id int) bool {
    s.mu.Lock()
    defer s.mu.Unlock()

    _, ok := s.books[id]
    if ok {
        delete(s.books, id)
    }
    return ok
}
```

We use `sync.RWMutex` because multiple goroutines (one per HTTP connection) may access the store simultaneously. Reads use `RLock` (multiple readers allowed), writes use `Lock` (exclusive access).

### Step 3: Response Helpers

```go
// respondJSON writes a JSON response with the given status code.
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if data != nil {
        if err := json.NewEncoder(w).Encode(data); err != nil {
            log.Printf("Error encoding response: %v", err)
        }
    }
}

// respondError writes a JSON error response.
func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

### Step 4: Handlers

```go
// handleListBooks handles GET /api/books
func handleListBooks(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        books := store.GetAll()
        respondJSON(w, http.StatusOK, books)
    }
}

// handleGetBook handles GET /api/books/{id}
func handleGetBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }

        book, ok := store.GetByID(id)
        if !ok {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }

        respondJSON(w, http.StatusOK, book)
    }
}

// handleCreateBook handles POST /api/books
func handleCreateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req CreateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
            return
        }

        // Validate required fields
        if req.Title == "" {
            respondError(w, http.StatusBadRequest, "Title is required")
            return
        }
        if req.Author == "" {
            respondError(w, http.StatusBadRequest, "Author is required")
            return
        }
        if req.Year < 1 {
            respondError(w, http.StatusBadRequest, "Year must be a positive number")
            return
        }

        book := store.Create(req)
        respondJSON(w, http.StatusCreated, book)
    }
}

// handleUpdateBook handles PUT /api/books/{id}
func handleUpdateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }

        var req UpdateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
            return
        }

        book, ok := store.Update(id, req)
        if !ok {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }

        respondJSON(w, http.StatusOK, book)
    }
}

// handleDeleteBook handles DELETE /api/books/{id}
func handleDeleteBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }

        if !store.Delete(id) {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }

        respondJSON(w, http.StatusOK, map[string]string{"message": "Book deleted"})
    }
}
```

### Step 5: Wire It All Together

```go
func main() {
    store := NewBookStore()

    // Seed with sample data
    store.Create(CreateBookRequest{
        Title:  "The Go Programming Language",
        Author: "Alan Donovan & Brian Kernighan",
        Year:   2015,
        ISBN:   "978-0134190440",
    })
    store.Create(CreateBookRequest{
        Title:  "Concurrency in Go",
        Author: "Katherine Cox-Buday",
        Year:   2017,
        ISBN:   "978-1491941195",
    })

    mux := http.NewServeMux()

    // Routes (Go 1.22+ syntax)
    mux.HandleFunc("GET /api/books", handleListBooks(store))
    mux.HandleFunc("GET /api/books/{id}", handleGetBook(store))
    mux.HandleFunc("POST /api/books", handleCreateBook(store))
    mux.HandleFunc("PUT /api/books/{id}", handleUpdateBook(store))
    mux.HandleFunc("DELETE /api/books/{id}", handleDeleteBook(store))

    // Configure the server with timeouts
    server := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    fmt.Println("Book API server listening on http://localhost:8080")
    if err := server.ListenAndServe(); err != nil {
        log.Fatal("Server failed:", err)
    }
}
```

### The Same API in Express.js

For comparison, here is the equivalent Express.js implementation:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

let books = [
  { id: 1, title: 'The Go Programming Language', author: 'Alan Donovan & Brian Kernighan',
    year: 2015, isbn: '978-0134190440', created_at: new Date(), updated_at: new Date() },
  { id: 2, title: 'Concurrency in Go', author: 'Katherine Cox-Buday',
    year: 2017, isbn: '978-1491941195', created_at: new Date(), updated_at: new Date() },
];
let nextId = 3;

// GET /api/books
app.get('/api/books', (req, res) => {
  res.json(books);
});

// GET /api/books/:id
app.get('/api/books/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const book = books.find(b => b.id === id);
  if (!book) return res.status(404).json({ error: `Book with ID ${id} not found` });
  res.json(book);
});

// POST /api/books
app.post('/api/books', (req, res) => {
  const { title, author, year, isbn } = req.body;
  if (!title) return res.status(400).json({ error: 'Title is required' });
  if (!author) return res.status(400).json({ error: 'Author is required' });
  if (!year || year < 1) return res.status(400).json({ error: 'Year must be a positive number' });

  const book = {
    id: nextId++, title, author, year, isbn: isbn || '',
    created_at: new Date(), updated_at: new Date()
  };
  books.push(book);
  res.status(201).json(book);
});

// PUT /api/books/:id
app.put('/api/books/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = books.findIndex(b => b.id === id);
  if (index === -1) return res.status(404).json({ error: `Book with ID ${id} not found` });

  const { title, author, year, isbn } = req.body;
  if (title !== undefined) books[index].title = title;
  if (author !== undefined) books[index].author = author;
  if (year !== undefined) books[index].year = year;
  if (isbn !== undefined) books[index].isbn = isbn;
  books[index].updated_at = new Date();
  res.json(books[index]);
});

// DELETE /api/books/:id
app.delete('/api/books/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = books.findIndex(b => b.id === id);
  if (index === -1) return res.status(404).json({ error: `Book with ID ${id} not found` });

  books.splice(index, 1);
  res.json({ message: 'Book deleted' });
});

app.listen(8080, () => console.log('Express server on :8080'));
```

### Side-by-Side Observations

| Aspect | Go | Express.js |
|--------|-----|------------|
| Lines of code | ~200 (with store logic) | ~60 |
| Dependencies | 0 | 1 (express) |
| Type safety | Full -- compiler catches mismatches | None -- runtime errors |
| Concurrency | One goroutine per connection; mutex for shared state | Single-threaded event loop; no concurrency issues with state |
| Deployment | Single static binary | Requires Node.js runtime + `node_modules` |
| JSON parsing | Manual decode + validate | `express.json()` middleware auto-parses |
| Partial updates | Pointer fields to detect "not sent" vs "sent empty" | `undefined` check |

---

## 7. The Middleware Pattern

Middleware wraps handlers to add cross-cutting behavior like logging, authentication, CORS, rate limiting, or panic recovery.

### How Middleware Works in Go

A middleware is simply a function that takes an `http.Handler` and returns a new `http.Handler`:

```go
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Do something BEFORE the handler
        next.ServeHTTP(w, r) // Call the next handler
        // Do something AFTER the handler
    })
}
```

### Logging Middleware

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrap ResponseWriter to capture the status code
        wrapped := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        log.Printf(
            "%s %s %d %s",
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            time.Since(start),
        )
    })
}

// statusRecorder wraps http.ResponseWriter to capture the status code.
type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
    sr.statusCode = code
    sr.ResponseWriter.WriteHeader(code)
}
```

### CORS Middleware

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Handle preflight
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### Recovery Middleware (Panic Recovery)

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC: %v", err)
                respondError(w, http.StatusInternalServerError, "Internal server error")
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

### Authentication Middleware

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            respondError(w, http.StatusUnauthorized, "Authorization header required")
            return
        }

        // In production, validate the token (JWT, API key, etc.)
        if !strings.HasPrefix(token, "Bearer ") {
            respondError(w, http.StatusUnauthorized, "Invalid authorization format")
            return
        }

        // You could add user info to the request context here
        // ctx := context.WithValue(r.Context(), "userID", extractedUserID)
        // next.ServeHTTP(w, r.WithContext(ctx))

        next.ServeHTTP(w, r)
    })
}
```

### Chaining Middleware

```go
// Chain applies middleware in order: first middleware is outermost.
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    // Apply in reverse so the first middleware listed is the first to execute
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func main() {
    store := NewBookStore()
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/books", handleListBooks(store))
    mux.HandleFunc("POST /api/books", handleCreateBook(store))
    // ... other routes

    // Wrap the mux with middleware
    handler := Chain(
        mux,
        recoveryMiddleware,
        loggingMiddleware,
        corsMiddleware,
    )

    // Request flow: recoveryMiddleware -> loggingMiddleware -> corsMiddleware -> mux -> handler

    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    log.Fatal(server.ListenAndServe())
}
```

### Per-Route Middleware

Sometimes you want middleware on specific routes only:

```go
func main() {
    store := NewBookStore()
    mux := http.NewServeMux()

    // Public routes
    mux.HandleFunc("GET /api/books", handleListBooks(store))
    mux.HandleFunc("GET /api/books/{id}", handleGetBook(store))

    // Protected routes -- wrap individual handlers
    mux.Handle("POST /api/books",
        authMiddleware(http.HandlerFunc(handleCreateBook(store))))
    mux.Handle("PUT /api/books/{id}",
        authMiddleware(http.HandlerFunc(handleUpdateBook(store))))
    mux.Handle("DELETE /api/books/{id}",
        authMiddleware(http.HandlerFunc(handleDeleteBook(store))))

    // Global middleware still applies to all
    handler := Chain(mux, recoveryMiddleware, loggingMiddleware, corsMiddleware)
    http.ListenAndServe(":8080", handler)
}
```

### Comparison with Express.js Middleware

```javascript
// Express.js middleware
const logger = (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    console.log(`${req.method} ${req.url} ${res.statusCode} ${Date.now() - start}ms`);
  });
  next();
};

const auth = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  next();
};

// Global middleware
app.use(logger);

// Route-specific middleware
app.post('/api/books', auth, (req, res) => { /* ... */ });
```

| Aspect | Go Middleware | Express Middleware |
|--------|-------------|-------------------|
| Signature | `func(http.Handler) http.Handler` | `(req, res, next) => {}` |
| Flow control | Call `next.ServeHTTP(w, r)` or return | Call `next()` or send response |
| Error handling | Return error response directly | Call `next(err)` for error middleware |
| Composition | Function wrapping / `Chain` helper | `app.use()` |
| Type safety | Handlers are typed interfaces | Dynamic function signatures |

The key conceptual difference: Express middleware uses `next()` to continue, while Go middleware calls `next.ServeHTTP()`. Both achieve the same result, but Go's approach is based on wrapping (decorator pattern) rather than a callback chain.

---

## 8. Popular Router Libraries

While Go 1.22's `ServeMux` covers most needs, several popular routers offer additional features. Here is a brief overview of when you might reach for one.

### `chi` (github.com/go-chi/chi)

Chi is the most popular standard-library-compatible router. It implements `http.Handler`, so it works with all standard middleware and tools.

```go
import "github.com/go-chi/chi/v5"
import "github.com/go-chi/chi/v5/middleware"

func main() {
    r := chi.NewRouter()

    // Built-in middleware
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.RequestID)

    r.Route("/api/books", func(r chi.Router) {
        r.Get("/", listBooks)
        r.Post("/", createBook)

        r.Route("/{id}", func(r chi.Router) {
            r.Get("/", getBook)
            r.Put("/", updateBook)
            r.Delete("/", deleteBook)
        })
    })

    http.ListenAndServe(":8080", r)
}
```

**When to use chi:** When you want route grouping, subrouters, or its large collection of built-in middleware -- all while staying compatible with the standard library.

### `gorilla/mux` (github.com/gorilla/mux)

Gorilla/mux was the go-to router for years. It supports regex patterns in routes, host matching, and other advanced features.

```go
import "github.com/gorilla/mux"

func main() {
    r := mux.NewRouter()

    r.HandleFunc("/api/books", listBooks).Methods("GET")
    r.HandleFunc("/api/books/{id:[0-9]+}", getBook).Methods("GET") // Regex constraint
    r.HandleFunc("/api/books", createBook).Methods("POST")

    http.ListenAndServe(":8080", r)
}
```

> **Note:** The Gorilla toolkit was briefly archived in late 2022 but was revived by community maintainers. It remains widely used but is in maintenance mode. For new projects, chi or the standard library are generally preferred.

### `gin` (github.com/gin-gonic/gin)

Gin is a full web framework (not just a router) with built-in JSON binding, validation, and its own context type. It is the fastest in benchmarks but deviates from the standard library's `http.Handler` interface.

```go
import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default() // Includes logger and recovery middleware

    r.GET("/api/books", func(c *gin.Context) {
        c.JSON(200, books)
    })

    r.POST("/api/books", func(c *gin.Context) {
        var req CreateBookRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(400, gin.H{"error": err.Error()})
            return
        }
        // ...
    })

    r.Run(":8080")
}
```

**When to use gin:** When you want a full framework with built-in validation, binding, and rendering -- and you do not mind locking into its API.

### WHY the Standard Library Is Often Enough

Since Go 1.22, the standard `ServeMux` supports:
- Method-based routing (`"GET /api/books"`)
- Path parameters (`"/api/books/{id}"`)
- Wildcard matching (`"/files/{path...}"`)

This covers the needs of most REST APIs. Third-party routers add value when you need:
- Route grouping and subrouters (chi)
- Regex path constraints (gorilla/mux)
- Built-in binding/validation (gin)
- A large ecosystem of pre-built middleware (chi, gin)

For learning and for many production services, the standard library is the right choice. You can always introduce a router later without rewriting your handlers, since chi and gorilla/mux both implement `http.Handler`.

---

## 9. Request Validation

### Manual Validation

The simplest approach is manual validation inside your handler, as shown in the book API:

```go
func handleCreateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req CreateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
            return
        }

        // Manual validation
        var errors []string
        if req.Title == "" {
            errors = append(errors, "title is required")
        }
        if req.Author == "" {
            errors = append(errors, "author is required")
        }
        if req.Year < 1 || req.Year > time.Now().Year()+1 {
            errors = append(errors, "year must be between 1 and next year")
        }
        if req.ISBN != "" && len(req.ISBN) != 13 && len(req.ISBN) != 17 {
            errors = append(errors, "isbn must be 13 or 17 characters (with hyphens)")
        }

        if len(errors) > 0 {
            respondJSON(w, http.StatusBadRequest, map[string]interface{}{
                "error":   "Validation failed",
                "details": errors,
            })
            return
        }

        book := store.Create(req)
        respondJSON(w, http.StatusCreated, book)
    }
}
```

### A Reusable Validation Helper

You can build a simple validator for common patterns:

```go
type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type Validator struct {
    Errors []ValidationError
}

func NewValidator() *Validator {
    return &Validator{}
}

func (v *Validator) Required(field, value string) {
    if strings.TrimSpace(value) == "" {
        v.Errors = append(v.Errors, ValidationError{
            Field:   field,
            Message: fmt.Sprintf("%s is required", field),
        })
    }
}

func (v *Validator) MinLength(field, value string, min int) {
    if len(value) < min {
        v.Errors = append(v.Errors, ValidationError{
            Field:   field,
            Message: fmt.Sprintf("%s must be at least %d characters", field, min),
        })
    }
}

func (v *Validator) Range(field string, value, min, max int) {
    if value < min || value > max {
        v.Errors = append(v.Errors, ValidationError{
            Field:   field,
            Message: fmt.Sprintf("%s must be between %d and %d", field, min, max),
        })
    }
}

func (v *Validator) MatchesPattern(field, value, pattern, message string) {
    matched, _ := regexp.MatchString(pattern, value)
    if !matched {
        v.Errors = append(v.Errors, ValidationError{
            Field:   field,
            Message: message,
        })
    }
}

func (v *Validator) IsValid() bool {
    return len(v.Errors) == 0
}

// Usage
func validateCreateBook(req CreateBookRequest) *Validator {
    v := NewValidator()
    v.Required("title", req.Title)
    v.Required("author", req.Author)
    v.Range("year", req.Year, 1, time.Now().Year()+1)
    if req.ISBN != "" {
        v.MatchesPattern("isbn", req.ISBN, `^\d{3}-\d{10}$`, "isbn must match format XXX-XXXXXXXXXX")
    }
    return v
}
```

### Using the Validator in a Handler

```go
func handleCreateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req CreateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON")
            return
        }

        if v := validateCreateBook(req); !v.IsValid() {
            respondJSON(w, http.StatusBadRequest, map[string]interface{}{
                "error":   "Validation failed",
                "details": v.Errors,
            })
            return
        }

        book := store.Create(req)
        respondJSON(w, http.StatusCreated, book)
    }
}
```

### A Generic Decode-and-Validate Helper

To avoid repeating the decode/validate pattern in every handler:

```go
func decodeAndValidate[T any](r *http.Request, validate func(T) *Validator) (T, *Validator, error) {
    var req T
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        return req, nil, err
    }
    if validate != nil {
        v := validate(req)
        if !v.IsValid() {
            return req, v, nil
        }
    }
    return req, nil, nil
}

// Usage in handler
func handleCreateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        req, v, err := decodeAndValidate(r, validateCreateBook)
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON")
            return
        }
        if v != nil {
            respondJSON(w, http.StatusBadRequest, map[string]interface{}{
                "error":   "Validation failed",
                "details": v.Errors,
            })
            return
        }

        book := store.Create(req)
        respondJSON(w, http.StatusCreated, book)
    }
}
```

### Comparison with Express.js Validation

```javascript
// Express.js with express-validator
const { body, validationResult } = require('express-validator');

app.post('/api/books',
  body('title').notEmpty().withMessage('Title is required'),
  body('author').notEmpty().withMessage('Author is required'),
  body('year').isInt({ min: 1, max: 2027 }).withMessage('Invalid year'),
  body('isbn').optional().matches(/^\d{3}-\d{10}$/).withMessage('Invalid ISBN'),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ error: 'Validation failed', details: errors.array() });
    }
    // ... create book
  }
);
```

Express-validator is declarative and reads nicely. Go's approach is more imperative but equally clear, and requires no additional dependencies. For projects that want a similar declarative experience in Go, the `go-playground/validator` package provides struct tag-based validation:

```go
import "github.com/go-playground/validator/v10"

type CreateBookRequest struct {
    Title  string `json:"title" validate:"required,min=1,max=200"`
    Author string `json:"author" validate:"required,min=1,max=100"`
    Year   int    `json:"year" validate:"required,min=1,max=2027"`
    ISBN   string `json:"isbn" validate:"omitempty,isbn13"`
}

var validate = validator.New()

func validateStruct(s interface{}) error {
    return validate.Struct(s)
}
```

---

## 10. Error Responses

### Consistent Error Format

Every API should use a consistent error response structure. Here is a pattern that works well:

```go
// APIError represents a structured error response.
type APIError struct {
    Status  int               `json:"-"`                   // HTTP status code (not in JSON body)
    Error   string            `json:"error"`               // Human-readable message
    Code    string            `json:"code,omitempty"`      // Machine-readable error code
    Details []ValidationError `json:"details,omitempty"`   // Field-level errors
}

// Common error constructors
func ErrBadRequest(msg string) APIError {
    return APIError{Status: http.StatusBadRequest, Error: msg, Code: "BAD_REQUEST"}
}

func ErrNotFound(resource string, id interface{}) APIError {
    return APIError{
        Status: http.StatusNotFound,
        Error:  fmt.Sprintf("%s with ID %v not found", resource, id),
        Code:   "NOT_FOUND",
    }
}

func ErrUnauthorized(msg string) APIError {
    return APIError{Status: http.StatusUnauthorized, Error: msg, Code: "UNAUTHORIZED"}
}

func ErrForbidden(msg string) APIError {
    return APIError{Status: http.StatusForbidden, Error: msg, Code: "FORBIDDEN"}
}

func ErrInternal(msg string) APIError {
    return APIError{Status: http.StatusInternalServerError, Error: msg, Code: "INTERNAL_ERROR"}
}

func ErrValidation(errors []ValidationError) APIError {
    return APIError{
        Status:  http.StatusBadRequest,
        Error:   "Validation failed",
        Code:    "VALIDATION_ERROR",
        Details: errors,
    }
}

// writeError sends the APIError as a JSON response.
func writeError(w http.ResponseWriter, apiErr APIError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(apiErr.Status)
    json.NewEncoder(w).Encode(apiErr)
}
```

### Usage in Handlers

```go
func handleGetBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            writeError(w, ErrBadRequest("Invalid book ID: must be a number"))
            return
        }

        book, ok := store.GetByID(id)
        if !ok {
            writeError(w, ErrNotFound("Book", id))
            return
        }

        respondJSON(w, http.StatusOK, book)
    }
}
```

### Error Responses Across Status Codes

| Status Code | When to Use | Example |
|-------------|-------------|---------|
| `400 Bad Request` | Malformed JSON, missing required fields, invalid types | `{"error":"Invalid JSON","code":"BAD_REQUEST"}` |
| `401 Unauthorized` | Missing or invalid authentication | `{"error":"Token expired","code":"UNAUTHORIZED"}` |
| `403 Forbidden` | Authenticated but lacks permission | `{"error":"Admin access required","code":"FORBIDDEN"}` |
| `404 Not Found` | Resource does not exist | `{"error":"Book with ID 99 not found","code":"NOT_FOUND"}` |
| `409 Conflict` | Duplicate resource, version conflict | `{"error":"ISBN already exists","code":"CONFLICT"}` |
| `422 Unprocessable Entity` | Valid JSON but semantically invalid | `{"error":"Year cannot be in the future","code":"VALIDATION_ERROR"}` |
| `429 Too Many Requests` | Rate limit exceeded | `{"error":"Rate limit exceeded","code":"RATE_LIMITED"}` |
| `500 Internal Server Error` | Unexpected server failure | `{"error":"Internal server error","code":"INTERNAL_ERROR"}` |

> **Important:** Never expose internal error details (stack traces, SQL queries, file paths) in 5xx responses. Log them server-side and return a generic message to the client.

---

## 11. Production Considerations

### Timeouts

An HTTP server without timeouts is vulnerable to slowloris attacks and resource exhaustion:

```go
server := &http.Server{
    Addr:    ":8080",
    Handler: handler,

    // ReadTimeout covers the time from connection acceptance to
    // the end of reading the request body.
    ReadTimeout: 5 * time.Second,

    // ReadHeaderTimeout limits how long the server waits for the
    // request headers. Set this to prevent slowloris attacks.
    ReadHeaderTimeout: 2 * time.Second,

    // WriteTimeout covers the time from reading the request body
    // to the end of writing the response.
    WriteTimeout: 10 * time.Second,

    // IdleTimeout limits how long a keep-alive connection stays
    // open when idle.
    IdleTimeout: 120 * time.Second,

    // MaxHeaderBytes limits the size of request headers.
    MaxHeaderBytes: 1 << 20, // 1 MB
}
```

### Graceful Shutdown

When your server receives a shutdown signal (SIGINT, SIGTERM), you want to stop accepting new connections while letting in-flight requests finish:

```go
func main() {
    store := NewBookStore()
    mux := http.NewServeMux()
    // ... register routes ...

    handler := Chain(mux, recoveryMiddleware, loggingMiddleware, corsMiddleware)

    server := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    // Run server in a goroutine so we can listen for shutdown signals
    go func() {
        log.Printf("Server listening on %s", server.Addr)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Printf("Received signal %s, shutting down gracefully...", sig)

    // Create a deadline context for shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Forced shutdown: %v", err)
    }

    log.Println("Server stopped cleanly")
}
```

This pattern is the standard for production Go servers. The `server.Shutdown` method:
1. Stops accepting new connections.
2. Waits for all active requests to complete (or until the context deadline).
3. Returns once everything is done.

### Request Body Size Limiting

Protect against oversized payloads:

```go
func limitBody(next http.Handler, maxBytes int64) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
        next.ServeHTTP(w, r)
    })
}

// Usage: limit to 1MB
mux.Handle("POST /api/books",
    limitBody(http.HandlerFunc(handleCreateBook(store)), 1<<20))
```

If the body exceeds the limit, `json.NewDecoder(r.Body).Decode()` will return an error containing "http: request body too large".

### Request ID Tracking

Assign a unique ID to every request for tracing through logs:

```go
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = fmt.Sprintf("%d", time.Now().UnixNano()) // Use UUID in production
        }

        // Add to response header
        w.Header().Set("X-Request-ID", id)

        // Add to request context for use in handlers
        ctx := context.WithValue(r.Context(), "requestID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 12. Go vs Express.js -- Full Comparison

### Architecture: Goroutines vs Event Loop

This is the single most important difference between Go and Node.js HTTP servers.

**Go (goroutine-per-connection):**
```
Client A ---> [Goroutine A] ---> Handler ---> DB Query (blocks goroutine, not OS thread)
Client B ---> [Goroutine B] ---> Handler ---> DB Query (runs concurrently)
Client C ---> [Goroutine C] ---> Handler ---> DB Query (runs concurrently)
```

Each connection gets its own goroutine. If one goroutine blocks on I/O (database query, file read), the Go runtime scheduler transparently switches to another goroutine. Your code reads top-to-bottom, sequentially, yet handles thousands of concurrent connections.

**Node.js (event loop):**
```
Client A ---|
Client B ---|---> [Event Loop] ---> Callback Queue ---> DB Query (async, non-blocking)
Client C ---|
```

All connections share a single thread. I/O operations are asynchronous and use callbacks (or Promises/async-await). CPU-bound work blocks the event loop and starves all other connections.

**Practical implications:**

| Scenario | Go | Node.js |
|----------|-----|---------|
| CPU-heavy request (image processing, crypto) | Runs on its goroutine; other goroutines are unaffected | Blocks the event loop; all requests stall |
| 10,000 concurrent connections | 10,000 goroutines (~40MB total) | Single thread, non-blocking callbacks |
| Code style | Synchronous (blocking calls look normal) | Async/await everywhere |
| Race conditions | Possible -- you need mutex/channels | Rare in single-threaded code; possible with shared state across async ops |

### Dependency Count

Building a production REST API:

| Feature | Go | Express.js |
|---------|-----|------------|
| HTTP server | `net/http` (stdlib) | `express` |
| JSON parsing | `encoding/json` (stdlib) | `express` (built-in body parser) |
| Routing | `net/http` (stdlib, Go 1.22+) | `express` |
| Middleware | Manual or `chi` | `express` |
| Validation | Manual or `go-playground/validator` | `express-validator` or `joi` |
| CORS | Manual or `rs/cors` | `cors` |
| Logging | `log` or `slog` (stdlib) | `morgan` or `winston` |
| Graceful shutdown | `http.Server.Shutdown` (stdlib) | `http-terminator` or manual |
| **Total dependencies** | **0-3** | **5-10+** |

### Performance Comparison

Based on typical benchmarks (numbers vary by hardware and workload):

| Metric | Go `net/http` | Express.js |
|--------|--------------|------------|
| Requests/sec (simple JSON) | 50,000-100,000+ | 10,000-30,000 |
| Latency (p99) | 1-5ms | 5-20ms |
| Memory per connection | ~4KB (goroutine stack) | Shared event loop (lower per-connection overhead) |
| Memory baseline | ~10MB | ~30-50MB |
| Startup time | ~10ms (compiled binary) | ~200-500ms (runtime + module loading) |
| CPU utilization (multi-core) | Automatic (GOMAXPROCS) | Single core by default; cluster mode for multi-core |

Go's advantage grows with CPU-bound work and high concurrency. Node.js is competitive for I/O-bound workloads at moderate scale.

### Deployment

| Aspect | Go | Node.js |
|--------|-----|---------|
| Build artifact | Single static binary | Source code + `node_modules` |
| Docker image size | ~10-20MB (FROM scratch) | ~100-300MB (node:alpine + deps) |
| Runtime required | None | Node.js runtime |
| Cross-compilation | `GOOS=linux GOARCH=amd64 go build` | N/A (same JS runs everywhere Node runs) |

### When to Choose Which

**Choose Go when:**
- You need high throughput and low latency
- You want minimal dependencies and small deployments
- Your service does CPU-intensive work
- You value type safety and compile-time error checking
- You want built-in concurrency primitives

**Choose Express.js/Node.js when:**
- You want rapid prototyping
- Your team is JavaScript-first
- You are building a BFF (backend-for-frontend) alongside a JavaScript frontend
- Your workload is purely I/O-bound at moderate scale
- You want a massive npm ecosystem of packages

---

## 13. Key Takeaways

1. **`encoding/json` is powerful and sufficient.** Struct tags give you precise control over serialization. You rarely need a third-party JSON library.

2. **Use `json.NewDecoder`/`json.NewEncoder` for HTTP bodies.** They stream directly from/to `io.Reader`/`io.Writer`, avoiding unnecessary memory allocation.

3. **Custom marshaling via `MarshalJSON`/`UnmarshalJSON` is the Go way** to handle special formats, enums, or computed fields. It keeps serialization logic co-located with the type.

4. **Always set timeouts on `http.Client` and `http.Server`.** The defaults have no timeout, which is dangerous in production.

5. **Always close `resp.Body`.** Leaking response bodies leaks TCP connections.

6. **Go's `net/http` is production-ready.** You do not need a framework to build a real-world REST API. The standard library includes routing (Go 1.22+), HTTP/2, TLS, graceful shutdown, and timeout support.

7. **The `http.Handler` interface is the foundation.** Every router, middleware, and handler revolves around this single method. Understanding it is essential.

8. **Middleware in Go is function wrapping**, not a callback chain. A middleware takes a handler and returns a new handler. It is the decorator pattern applied to HTTP.

9. **Goroutine-per-connection means true concurrency.** Unlike the Node.js event loop, CPU-bound work in one goroutine does not block other connections. But you must protect shared state with mutexes or channels.

10. **Use pointer fields in update requests** to distinguish "not sent" from "sent as zero value." This is a common pattern for PATCH/PUT endpoints.

11. **Consistent error responses build trust with API consumers.** Define a standard error structure early and use it everywhere.

12. **Graceful shutdown is not optional.** Use `signal.Notify` + `server.Shutdown` to ensure in-flight requests complete on deployment.

---

## 14. Practice Exercises

### Exercise 1: Todo API
Build a complete CRUD REST API for a Todo application with:
- Fields: `id`, `title`, `description`, `completed`, `due_date`, `created_at`, `updated_at`
- Filtering: `GET /api/todos?completed=true` and `GET /api/todos?due_before=2026-12-31`
- Sorting: `GET /api/todos?sort=due_date&order=asc`
- Pagination: `GET /api/todos?page=1&limit=10` (return total count in response)

### Exercise 2: Custom JSON Types
Create a `Money` type that:
- Stores amounts as integers (cents) internally to avoid floating-point issues
- Marshals to JSON as `"$12.99"` (string with dollar sign)
- Unmarshals from both `12.99` (number) and `"$12.99"` (string)
- Test with edge cases: `0`, negative amounts, very large amounts

### Exercise 3: HTTP Client Wrapper
Build a reusable HTTP client package that:
- Supports GET, POST, PUT, DELETE with JSON bodies
- Handles timeouts (configurable per-request)
- Retries failed requests with exponential backoff (max 3 retries)
- Adds default headers (Content-Type, User-Agent, Authorization)
- Returns typed responses using generics: `func Get[T any](url string) (T, error)`

### Exercise 4: Middleware Suite
Implement the following middleware:
- **Rate limiter:** Limit each IP to 100 requests per minute (use a `sync.Map` to track)
- **Request logger:** Log method, path, status code, duration, and request ID
- **JWT auth:** Parse a JWT from the Authorization header, verify the signature, and inject the claims into the request context
- **Response compression:** Gzip the response body if the client sends `Accept-Encoding: gzip`

### Exercise 5: Full API with Tests
Take the Book API from this chapter and:
- Add unit tests for all handlers using `httptest.NewRequest` and `httptest.NewRecorder`
- Add integration tests that start the server and make real HTTP requests
- Add search: `GET /api/books?q=golang` (search title and author)
- Add pagination: return `{ "data": [...], "total": 42, "page": 1, "per_page": 10 }`
- Deploy it: create a Dockerfile that builds a minimal image (`FROM scratch`)

### Exercise 6: Express.js Port
Take the Express.js book API from Section 6 and extend it to match all the features of the Go version. Then benchmark both:
- Use `hey` or `wrk` to send 10,000 requests with 100 concurrent connections
- Compare: requests/second, p99 latency, memory usage
- Try a CPU-intensive endpoint (compute SHA-256 of a large string 1000 times) and compare again

---

## Complete Runnable Example

Below is the entire Book API as a single file you can copy, build, and run.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "strconv"
    "strings"
    "sync"
    "syscall"
    "time"
)

// ============================================================
// Models
// ============================================================

type Book struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Author    string    `json:"author"`
    Year      int       `json:"year"`
    ISBN      string    `json:"isbn,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type CreateBookRequest struct {
    Title  string `json:"title"`
    Author string `json:"author"`
    Year   int    `json:"year"`
    ISBN   string `json:"isbn,omitempty"`
}

type UpdateBookRequest struct {
    Title  *string `json:"title,omitempty"`
    Author *string `json:"author,omitempty"`
    Year   *int    `json:"year,omitempty"`
    ISBN   *string `json:"isbn,omitempty"`
}

// ============================================================
// Store
// ============================================================

type BookStore struct {
    mu     sync.RWMutex
    books  map[int]Book
    nextID int
}

func NewBookStore() *BookStore {
    return &BookStore{books: make(map[int]Book), nextID: 1}
}

func (s *BookStore) GetAll() []Book {
    s.mu.RLock()
    defer s.mu.RUnlock()
    books := make([]Book, 0, len(s.books))
    for _, b := range s.books {
        books = append(books, b)
    }
    return books
}

func (s *BookStore) GetByID(id int) (Book, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    b, ok := s.books[id]
    return b, ok
}

func (s *BookStore) Create(req CreateBookRequest) Book {
    s.mu.Lock()
    defer s.mu.Unlock()
    now := time.Now().UTC()
    book := Book{
        ID: s.nextID, Title: req.Title, Author: req.Author,
        Year: req.Year, ISBN: req.ISBN, CreatedAt: now, UpdatedAt: now,
    }
    s.books[s.nextID] = book
    s.nextID++
    return book
}

func (s *BookStore) Update(id int, req UpdateBookRequest) (Book, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    book, ok := s.books[id]
    if !ok {
        return Book{}, false
    }
    if req.Title != nil {
        book.Title = *req.Title
    }
    if req.Author != nil {
        book.Author = *req.Author
    }
    if req.Year != nil {
        book.Year = *req.Year
    }
    if req.ISBN != nil {
        book.ISBN = *req.ISBN
    }
    book.UpdatedAt = time.Now().UTC()
    s.books[id] = book
    return book, true
}

func (s *BookStore) Delete(id int) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    _, ok := s.books[id]
    if ok {
        delete(s.books, id)
    }
    return ok
}

// ============================================================
// Response Helpers
// ============================================================

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if data != nil {
        json.NewEncoder(w).Encode(data)
    }
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// ============================================================
// Middleware
// ============================================================

type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
    sr.statusCode = code
    sr.ResponseWriter.WriteHeader(code)
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &statusRecorder{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)
        log.Printf("%s %s %d %s", r.Method, r.URL.Path, wrapped.statusCode, time.Since(start))
    })
}

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC: %v", err)
                respondError(w, http.StatusInternalServerError, "Internal server error")
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func Chain(handler http.Handler, mws ...func(http.Handler) http.Handler) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        handler = mws[i](handler)
    }
    return handler
}

// ============================================================
// Handlers
// ============================================================

func handleListBooks(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        books := store.GetAll()

        // Simple search by query parameter
        if q := r.URL.Query().Get("q"); q != "" {
            q = strings.ToLower(q)
            filtered := make([]Book, 0)
            for _, b := range books {
                if strings.Contains(strings.ToLower(b.Title), q) ||
                    strings.Contains(strings.ToLower(b.Author), q) {
                    filtered = append(filtered, b)
                }
            }
            books = filtered
        }

        respondJSON(w, http.StatusOK, books)
    }
}

func handleGetBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }
        book, ok := store.GetByID(id)
        if !ok {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }
        respondJSON(w, http.StatusOK, book)
    }
}

func handleCreateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req CreateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
            return
        }
        if req.Title == "" {
            respondError(w, http.StatusBadRequest, "Title is required")
            return
        }
        if req.Author == "" {
            respondError(w, http.StatusBadRequest, "Author is required")
            return
        }
        if req.Year < 1 {
            respondError(w, http.StatusBadRequest, "Year must be a positive number")
            return
        }
        book := store.Create(req)
        respondJSON(w, http.StatusCreated, book)
    }
}

func handleUpdateBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }
        var req UpdateBookRequest
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            respondError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
            return
        }
        book, ok := store.Update(id, req)
        if !ok {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }
        respondJSON(w, http.StatusOK, book)
    }
}

func handleDeleteBook(store *BookStore) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id, err := strconv.Atoi(r.PathValue("id"))
        if err != nil {
            respondError(w, http.StatusBadRequest, "Invalid book ID")
            return
        }
        if !store.Delete(id) {
            respondError(w, http.StatusNotFound, fmt.Sprintf("Book with ID %d not found", id))
            return
        }
        respondJSON(w, http.StatusOK, map[string]string{"message": "Book deleted"})
    }
}

// ============================================================
// Main
// ============================================================

func main() {
    store := NewBookStore()

    // Seed data
    store.Create(CreateBookRequest{
        Title: "The Go Programming Language", Author: "Alan Donovan & Brian Kernighan",
        Year: 2015, ISBN: "978-0134190440",
    })
    store.Create(CreateBookRequest{
        Title: "Concurrency in Go", Author: "Katherine Cox-Buday",
        Year: 2017, ISBN: "978-1491941195",
    })

    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/books", handleListBooks(store))
    mux.HandleFunc("GET /api/books/{id}", handleGetBook(store))
    mux.HandleFunc("POST /api/books", handleCreateBook(store))
    mux.HandleFunc("PUT /api/books/{id}", handleUpdateBook(store))
    mux.HandleFunc("DELETE /api/books/{id}", handleDeleteBook(store))

    handler := Chain(mux, recoveryMiddleware, loggingMiddleware, corsMiddleware)

    server := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    go func() {
        log.Printf("Book API server listening on http://localhost%s", server.Addr)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Printf("Received %s, shutting down...", sig)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Forced shutdown: %v", err)
    }
    log.Println("Server stopped")
}
```

**To run:**
```bash
# Requires Go 1.22 or later
go run main.go

# Test with curl:
curl http://localhost:8080/api/books
curl http://localhost:8080/api/books/1
curl -X POST http://localhost:8080/api/books -H "Content-Type: application/json" -d '{"title":"New Book","author":"Author","year":2026}'
curl -X PUT http://localhost:8080/api/books/1 -H "Content-Type: application/json" -d '{"title":"Updated Title"}'
curl -X DELETE http://localhost:8080/api/books/1
curl "http://localhost:8080/api/books?q=concurrency"
```

---

*Next chapter: Chapter 13 -- Testing in Go*
