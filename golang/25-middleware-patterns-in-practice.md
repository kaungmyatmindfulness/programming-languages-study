# Chapter 25: Middleware Patterns in Practice

Every production HTTP API is built on layers. Before a request reaches your business logic, it passes through authentication, logging, rate limiting, panic recovery, CORS handling, and security headers. After the response is formed, it passes back through those same layers in reverse. This pipeline of cross-cutting concerns is called middleware, and getting it right is the difference between an API that survives production traffic and one that leaks errors, drops requests, and exposes security holes.

This chapter builds a complete, production-grade middleware stack in Go -- the kind you would find in a real API project like krafty-core. Every middleware is explained from first principles, shown with complete runnable code, compared with the Node.js/Express equivalent, and tested. By the end, you will have a middleware toolkit that you can drop directly into any Go HTTP service.

This is not theory. This is the code that stands between the internet and your business logic.

---

## Prerequisites

You should be comfortable with:
- Go HTTP servers and the `http.Handler` interface (Chapter 12)
- Context package and value propagation (Chapter 16)
- Goroutines and concurrency primitives like `sync.Mutex` (Chapters 10, 17)
- Interfaces and function types (Chapters 4, 6)
- Error handling patterns (Chapter 8)
- Testing with `httptest` (Chapter 14)
- Structured logging with `log/slog` (Chapter 21)

If you have built Express.js middleware before, the conceptual mapping will be immediate -- but the implementation details differ in important ways.

---

## Table of Contents

1. [Middleware Fundamentals](#1-middleware-fundamentals)
2. [Recovery Middleware](#2-recovery-middleware)
3. [Request Logging Middleware](#3-request-logging-middleware)
4. [CORS Middleware](#4-cors-middleware)
5. [Rate Limiting Middleware](#5-rate-limiting-middleware)
6. [Security Headers Middleware](#6-security-headers-middleware)
7. [Body Size Limiting Middleware](#7-body-size-limiting-middleware)
8. [Request ID Middleware](#8-request-id-middleware)
9. [Authentication Middleware](#9-authentication-middleware)
10. [Middleware Ordering](#10-middleware-ordering)
11. [Per-Route vs Global Middleware](#11-per-route-vs-global-middleware)
12. [Composing Middleware](#12-composing-middleware)
13. [Testing Middleware](#13-testing-middleware)
14. [Real-World Example: Complete Middleware Stack](#14-real-world-example-complete-middleware-stack)
15. [Key Takeaways](#15-key-takeaways)
16. [Practice Exercises](#16-practice-exercises)

---

## 1. Middleware Fundamentals

### What Middleware Is

Middleware is a function that wraps an HTTP handler to add behavior before and/or after the handler executes. It intercepts the request, optionally modifies it, calls the next handler in the chain, and optionally modifies the response. Every production API uses middleware for cross-cutting concerns that apply to many or all routes: logging, authentication, rate limiting, panic recovery, CORS, and more.

The concept is identical across languages. In Express.js, middleware is a function `(req, res, next) => { ... }`. In Go, middleware is a function `func(http.Handler) http.Handler`. The shape differs but the purpose is the same: separate cross-cutting concerns from business logic.

### The `func(http.Handler) http.Handler` Pattern

Go's middleware pattern is built on a single type signature:

```go
func(http.Handler) http.Handler
```

This is a function that takes a handler, wraps it, and returns a new handler. The wrapping handler can run code before calling the inner handler, after calling it, or both. This pattern is sometimes called the "decorator pattern" or the "onion model" -- each middleware is a layer of the onion, and the request peels through each layer on the way in and passes back through on the way out.

Here is the simplest possible middleware:

```go
package main

import (
	"fmt"
	"net/http"
)

// TimingMiddleware adds a Server-Timing header showing how long the request took.
// This is a middleware: it takes a handler and returns a new handler.
func TimingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// Call the next handler in the chain (pre-processing is done, now delegate)
		next.ServeHTTP(w, r)

		// Post-processing: this runs AFTER the inner handler has finished
		duration := time.Since(start)
		fmt.Printf("Request %s %s took %v\n", r.Method, r.URL.Path, duration)
	})
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, World!"))
}

func main() {
	// Wrap the handler with the middleware
	handler := TimingMiddleware(http.HandlerFunc(helloHandler))

	http.ListenAndServe(":8080", handler)
}
```

Let's break down exactly what happens when a request arrives:

```
1. Request arrives at the server
2. TimingMiddleware runs: records start time
3. TimingMiddleware calls next.ServeHTTP(w, r)
4. helloHandler runs: writes "Hello, World!" to the response
5. Control returns to TimingMiddleware
6. TimingMiddleware runs: prints the duration
7. Response is sent to the client
```

This is the **pre-processing / delegate / post-processing** pattern. Every middleware in this chapter follows it.

### Execution Order: The Onion Model

When you stack multiple middleware, they execute in a specific order. The outermost middleware runs first on the way in and last on the way out:

```
Request → [Middleware A (pre)] → [Middleware B (pre)] → [Handler] → [Middleware B (post)] → [Middleware A (post)] → Response
```

```go
package main

import (
	"fmt"
	"net/http"
)

func middlewareA(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("A: before")
		next.ServeHTTP(w, r)
		fmt.Println("A: after")
	})
}

func middlewareB(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("B: before")
		next.ServeHTTP(w, r)
		fmt.Println("B: after")
	})
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("Handler")
	w.Write([]byte("OK"))
}

func main() {
	// Wrapping order: A wraps B wraps handler
	// Execution order: A(pre) → B(pre) → handler → B(post) → A(post)
	h := middlewareA(middlewareB(http.HandlerFunc(handler)))

	http.ListenAndServe(":8080", h)
}
```

Output for a single request:

```
A: before
B: before
Handler
B: after
A: after
```

This is identical to how Express middleware works -- `app.use(A); app.use(B);` executes A first, then B, then the route handler. The difference is that in Express, `next()` is a callback passed as an argument, while in Go, `next` is the handler itself and you call `next.ServeHTTP(w, r)`.

### Middleware That Short-Circuits

A middleware can choose not to call the next handler. This is how authentication middleware rejects unauthorized requests:

```go
package main

import (
	"net/http"
)

func requireAPIKey(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		key := r.Header.Get("X-API-Key")
		if key != "secret-key-123" {
			// Short-circuit: do NOT call next.ServeHTTP
			http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
			return
		}
		// Key is valid, proceed to the next handler
		next.ServeHTTP(w, r)
	})
}

func protectedHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(`{"message":"you have access"}`))
}

func main() {
	h := requireAPIKey(http.HandlerFunc(protectedHandler))
	http.ListenAndServe(":8080", h)
}
```

When the API key is missing or wrong, the middleware writes a 401 response and returns. The inner handler never executes. This is the Go equivalent of calling `return res.status(401).json({error: "unauthorized"})` before `next()` in Express.

### Middleware Factories

Most real middleware needs configuration. Instead of a plain `func(http.Handler) http.Handler`, you write a factory function that takes configuration and returns a middleware:

```go
package main

import (
	"log/slog"
	"net/http"
)

// LoggingConfig configures the logging middleware.
type LoggingConfig struct {
	Logger      *slog.Logger
	LogBody     bool
	SkipPaths   []string
}

// Logging is a middleware factory. It takes configuration and returns a middleware.
func Logging(cfg LoggingConfig) func(http.Handler) http.Handler {
	skipSet := make(map[string]bool, len(cfg.SkipPaths))
	for _, p := range cfg.SkipPaths {
		skipSet[p] = true
	}

	// This is the actual middleware
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if skipSet[r.URL.Path] {
				next.ServeHTTP(w, r)
				return
			}
			cfg.Logger.Info("request received",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
			)
			next.ServeHTTP(w, r)
		})
	}
}
```

The pattern is always the same: **outer function takes config, returns `func(http.Handler) http.Handler`**. Every middleware in this chapter uses this factory pattern because real middleware always needs configuration.

### Node.js/Express Comparison

```javascript
// Express middleware
const timing = (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} took ${duration}ms`);
  });
  next();
};

app.use(timing);

// Express middleware factory
const rateLimit = (options) => {
  return (req, res, next) => {
    // use options.maxRequests, options.windowMs, etc.
    next();
  };
};

app.use(rateLimit({ maxRequests: 100, windowMs: 60000 }));
```

```go
// Go middleware
func Timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		duration := time.Since(start)
		fmt.Printf("%s %s took %v\n", r.Method, r.URL.Path, duration)
	})
}

// Go middleware factory
func RateLimit(cfg RateLimitConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// use cfg.MaxRequests, cfg.Window, etc.
			next.ServeHTTP(w, r)
		})
	}
}
```

The key differences:

| Aspect | Express | Go |
|--------|---------|-----|
| Signature | `(req, res, next) => {}` | `func(http.Handler) http.Handler` |
| Delegation | Call `next()` | Call `next.ServeHTTP(w, r)` |
| Post-processing | `res.on('finish', ...)` event listener | Code after `next.ServeHTTP(w, r)` |
| Short-circuit | Return without calling `next()` | Return without calling `next.ServeHTTP` |
| Config | Factory function returning middleware | Factory function returning middleware |
| Type safety | None (any function accepted) | Enforced by `http.Handler` interface |

Go's approach is more explicit. You see exactly when the next handler is called. There is no hidden event system. Post-processing code is just code that runs after `next.ServeHTTP(w, r)` returns.

---

## 2. Recovery Middleware

### Why You Need Panic Recovery

In Go, a panic in an HTTP handler will crash the goroutine handling that request. By default, Go's `net/http` server recovers from panics per-goroutine, logs the panic, and keeps the server running. However, the default recovery behavior writes a raw stack trace to stderr and closes the connection without sending a proper HTTP response. In production, you want:

1. **Structured logging** of the panic with a full stack trace
2. **A proper 500 JSON response** to the client
3. **Request context** in the log (which URL, which method, which request ID)
4. **No server crash** -- other requests continue unaffected

### The Problem: What Happens Without Recovery

```go
package main

import (
	"net/http"
)

func dangerousHandler(w http.ResponseWriter, r *http.Request) {
	// This will panic
	var s []string
	_ = s[10] // index out of range
}

func main() {
	http.HandleFunc("/danger", dangerousHandler)
	http.ListenAndServe(":8080", nil)
}
```

Without recovery middleware, the client sees a connection reset. The server logs an ugly unstructured stack trace to stderr. If you are running behind a load balancer, the health check might still pass because the server is technically still running -- but the error goes unnoticed in your monitoring system because it bypasses your structured logging pipeline.

### Production Recovery Middleware

```go
package main

import (
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"runtime/debug"
)

// Recovery returns middleware that recovers from panics, logs the error
// with a full stack trace, and returns a 500 JSON response.
func Recovery(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if err := recover(); err != nil {
					// Capture the full stack trace
					stack := debug.Stack()

					// Log the panic with structured fields
					logger.Error("panic recovered",
						slog.String("error", fmt.Sprintf("%v", err)),
						slog.String("method", r.Method),
						slog.String("path", r.URL.Path),
						slog.String("remote_addr", r.RemoteAddr),
						slog.String("stack", string(stack)),
					)

					// Only send a response if headers haven't been written yet.
					// If the handler already called w.WriteHeader() or w.Write(),
					// we cannot change the status code -- the response is already
					// partially sent. The best we can do is log the panic.
					//
					// In practice, most panics happen before any response is written,
					// so this check succeeds almost every time.
					w.Header().Set("Content-Type", "application/json")
					w.WriteHeader(http.StatusInternalServerError)
					json.NewEncoder(w).Encode(map[string]string{
						"error": "internal server error",
					})
				}
			}()

			next.ServeHTTP(w, r)
		})
	}
}

// Simulate a handler that panics
func panicHandler(w http.ResponseWriter, r *http.Request) {
	panic("something went terribly wrong")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /panic", panicHandler)
	mux.HandleFunc("GET /health", healthHandler)

	// Wrap the entire mux with recovery middleware
	handler := Recovery(logger)(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	http.ListenAndServe(":8080", handler)
}
```

### Handling the "Headers Already Sent" Problem

There is a subtle issue with recovery middleware: if the handler has already called `w.WriteHeader()` or `w.Write()` before panicking, the response headers are already on the wire. You cannot change the status code. The `responseWriter` wrapper we build in Section 3 helps detect this:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"runtime/debug"
)

// responseWriter wraps http.ResponseWriter to track whether headers have been sent.
type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK,
	}
}

func (rw *responseWriter) WriteHeader(code int) {
	if !rw.headerWritten {
		rw.statusCode = code
		rw.headerWritten = true
		rw.ResponseWriter.WriteHeader(code)
	}
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.headerWritten = true
	}
	return rw.ResponseWriter.Write(b)
}

// RecoveryWithWriterCheck uses the wrapper to detect if headers were already sent.
func RecoveryWithWriterCheck(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			rw := newResponseWriter(w)

			defer func() {
				if err := recover(); err != nil {
					stack := debug.Stack()

					logger.Error("panic recovered",
						slog.String("error", fmt.Sprintf("%v", err)),
						slog.String("method", r.Method),
						slog.String("path", r.URL.Path),
						slog.String("stack", string(stack)),
					)

					if !rw.headerWritten {
						rw.Header().Set("Content-Type", "application/json")
						rw.WriteHeader(http.StatusInternalServerError)
						json.NewEncoder(rw).Encode(map[string]string{
							"error": "internal server error",
						})
					} else {
						// Headers already sent. We cannot change the status code.
						// Log that we couldn't send a proper error response.
						logger.Warn("panic after headers written, cannot send 500 response",
							slog.String("path", r.URL.Path),
						)
					}
				}
			}()

			next.ServeHTTP(rw, r)
		})
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /partial-panic", func(w http.ResponseWriter, r *http.Request) {
		// Write some data, then panic -- headers are already sent
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("partial response..."))
		panic("mid-response panic")
	})

	handler := RecoveryWithWriterCheck(logger)(mux)
	http.ListenAndServe(":8080", handler)
}
```

### Node.js/Express Comparison

```javascript
// Express error handling middleware (must have 4 parameters)
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err.stack);
  if (!res.headersSent) {
    res.status(500).json({ error: 'internal server error' });
  }
});

// For uncaught exceptions (panics equivalent)
process.on('uncaughtException', (err) => {
  console.error('Uncaught exception:', err);
  process.exit(1); // Node.js MUST exit -- the process is in an unknown state
});
```

A critical difference: in Go, `recover()` catches panics in the current goroutine and the goroutine continues safely. In Node.js, an uncaught exception puts the process in an undefined state and you should exit. Go's goroutine-per-request model means a panic in one request does not affect other requests. Node.js's single-threaded model means an uncaught exception in one request can corrupt shared state.

---

## 3. Request Logging Middleware

### Why Structured Request Logging Matters

Every request to your API should be logged with:
- HTTP method and path
- Response status code
- Duration
- Bytes written
- Client IP
- Request ID (if present)

This data feeds your dashboards, alerting, and debugging. Without it, you are flying blind in production.

### The ResponseWriter Wrapper

Go's `http.ResponseWriter` interface does not expose the status code after `WriteHeader` is called. To capture it, you must wrap the `ResponseWriter`:

```go
package main

import (
	"net/http"
)

// responseWriter wraps http.ResponseWriter to capture the status code
// and number of bytes written. This is essential for logging middleware.
type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK, // Default to 200 if WriteHeader is never called
	}
}

// WriteHeader captures the status code before delegating to the underlying writer.
func (rw *responseWriter) WriteHeader(code int) {
	if !rw.headerWritten {
		rw.statusCode = code
		rw.headerWritten = true
		rw.ResponseWriter.WriteHeader(code)
	}
}

// Write captures the number of bytes written. It also implicitly calls
// WriteHeader(200) if WriteHeader hasn't been called yet (matching the
// standard library's behavior).
func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.headerWritten = true
		// The standard library's ResponseWriter calls WriteHeader(200) on first Write
		rw.statusCode = http.StatusOK
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

// Unwrap returns the underlying ResponseWriter. This is needed for
// http.ResponseController and other standard library features that
// type-assert the ResponseWriter to access Flusher, Hijacker, etc.
func (rw *responseWriter) Unwrap() http.ResponseWriter {
	return rw.ResponseWriter
}
```

**WHY this wrapper is necessary:** Go's `http.ResponseWriter` is an interface with three methods: `Header()`, `Write()`, and `WriteHeader()`. None of them return the current status code. The status code is internal to the concrete type. The only way to observe what status code was sent is to intercept the `WriteHeader` call. This is a deliberate design choice in Go -- the `ResponseWriter` is intentionally minimal. The cost is that every logging middleware in existence must wrap it.

### The Unwrap Method

The `Unwrap()` method is important. Go 1.20 introduced `http.ResponseController`, which uses interface type assertions to check if the underlying `ResponseWriter` supports features like `Flush()` (for streaming) or `Hijack()` (for WebSockets). Without `Unwrap()`, wrapping the `ResponseWriter` would break these features. The standard library's `http.ResponseController` calls `Unwrap()` to peel through wrapper layers and reach the real writer.

### Production Logging Middleware

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"net/url"
	"os"
	"strings"
	"time"
)

// responseWriter wraps http.ResponseWriter to capture status and bytes written.
type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK,
	}
}

func (rw *responseWriter) WriteHeader(code int) {
	if !rw.headerWritten {
		rw.statusCode = code
		rw.headerWritten = true
		rw.ResponseWriter.WriteHeader(code)
	}
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.headerWritten = true
		rw.statusCode = http.StatusOK
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

func (rw *responseWriter) Unwrap() http.ResponseWriter {
	return rw.ResponseWriter
}

// sensitiveParams is the set of query parameter names whose values should be
// redacted in logs. These commonly contain secrets.
var sensitiveParams = map[string]bool{
	"token":    true,
	"api_key":  true,
	"apikey":   true,
	"password": true,
	"secret":   true,
	"key":      true,
	"access_token": true,
}

// redactQuery replaces the values of sensitive query parameters with "[REDACTED]".
func redactQuery(rawQuery string) string {
	if rawQuery == "" {
		return ""
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil {
		return "[invalid query]"
	}
	for param := range values {
		if sensitiveParams[strings.ToLower(param)] {
			values.Set(param, "[REDACTED]")
		}
	}
	return values.Encode()
}

// LoggingConfig configures the request logging middleware.
type LoggingConfig struct {
	Logger    *slog.Logger
	SkipPaths map[string]bool // Paths to skip logging (e.g., health checks)
}

// Logging returns middleware that logs every request with method, path,
// status code, duration, and bytes written.
func Logging(cfg LoggingConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Skip logging for health check endpoints to avoid log noise
			if cfg.SkipPaths[r.URL.Path] {
				next.ServeHTTP(w, r)
				return
			}

			start := time.Now()
			rw := newResponseWriter(w)

			// Call the next handler
			next.ServeHTTP(rw, r)

			// Post-processing: log the completed request
			duration := time.Since(start)

			// Build the log attributes
			attrs := []slog.Attr{
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", rw.statusCode),
				slog.String("duration", duration.String()),
				slog.Int64("duration_ms", duration.Milliseconds()),
				slog.Int64("bytes", rw.written),
				slog.String("remote_addr", r.RemoteAddr),
				slog.String("user_agent", r.UserAgent()),
			}

			// Include redacted query string if present
			if r.URL.RawQuery != "" {
				attrs = append(attrs,
					slog.String("query", redactQuery(r.URL.RawQuery)),
				)
			}

			// Include request ID if present (set by RequestID middleware)
			if reqID := r.Header.Get("X-Request-ID"); reqID != "" {
				attrs = append(attrs,
					slog.String("request_id", reqID),
				)
			}

			// Choose log level based on status code
			msg := fmt.Sprintf("%s %s %d", r.Method, r.URL.Path, rw.statusCode)
			switch {
			case rw.statusCode >= 500:
				cfg.Logger.LogAttrs(r.Context(), slog.LevelError, msg, attrs...)
			case rw.statusCode >= 400:
				cfg.Logger.LogAttrs(r.Context(), slog.LevelWarn, msg, attrs...)
			default:
				cfg.Logger.LogAttrs(r.Context(), slog.LevelInfo, msg, attrs...)
			}
		})
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"status":"ok"}`))
	})
	mux.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`[{"id":1,"name":"Alice"}]`))
	})
	mux.HandleFunc("GET /error", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, `{"error":"not found"}`, http.StatusNotFound)
	})

	// Apply logging middleware, skipping health checks
	handler := Logging(LoggingConfig{
		Logger:    logger,
		SkipPaths: map[string]bool{"/health": true},
	})(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	http.ListenAndServe(":8080", handler)
}
```

### Log Output

When you hit `GET /users?token=abc123&page=1`, the log output looks like:

```json
{
  "time": "2026-03-24T10:15:30.000Z",
  "level": "INFO",
  "msg": "GET /users 200",
  "method": "GET",
  "path": "/users",
  "status": 200,
  "duration": "1.234ms",
  "duration_ms": 1,
  "bytes": 27,
  "remote_addr": "127.0.0.1:54321",
  "user_agent": "curl/8.1.0",
  "query": "page=1&token=%5BREDACTED%5D"
}
```

Notice that `token` was redacted. This prevents API keys and passwords from leaking into your log aggregation system.

### Node.js/Express Comparison

```javascript
// morgan (common Express logging middleware)
const morgan = require('morgan');
app.use(morgan(':method :url :status :response-time ms'));

// Custom middleware with duration and status capture
app.use((req, res, next) => {
  const start = Date.now();
  const originalEnd = res.end;

  res.end = function(...args) {
    const duration = Date.now() - start;
    console.log(JSON.stringify({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration_ms: duration,
    }));
    originalEnd.apply(res, args);
  };

  next();
});
```

In Express, capturing the status code requires monkey-patching `res.end` or using the `on-finished` package. In Go, you wrap the `ResponseWriter` with a struct that intercepts `WriteHeader`. Both are hacks around the fact that the response interface does not expose the status code after it is written -- but Go's struct embedding makes the wrapper cleaner and more type-safe than JavaScript's prototype patching.

---

## 4. CORS Middleware

### What CORS Is and Why It Exists

Cross-Origin Resource Sharing (CORS) is a security mechanism enforced by web browsers. When JavaScript running on `https://app.example.com` makes a fetch request to `https://api.example.com`, the browser blocks the response unless the API server explicitly allows it with CORS headers.

CORS is **not a server security feature**. It is a **browser security feature**. curl, Postman, and server-to-server requests ignore CORS entirely. CORS only matters when your API is called from a web browser running JavaScript on a different origin.

### The Preflight Request

For non-simple requests (anything with custom headers, JSON content type, or methods other than GET/POST), the browser sends a **preflight request** first: an OPTIONS request to the same URL. The server must respond with the appropriate CORS headers. Only then does the browser send the actual request.

```
Browser                          Server
   |                                |
   |  OPTIONS /api/users            |   ← Preflight request
   |  Origin: https://app.com       |
   |  Access-Control-Request-Method: POST
   |  Access-Control-Request-Headers: Content-Type
   |                                |
   |  204 No Content                |   ← Preflight response
   |  Access-Control-Allow-Origin:  |
   |    https://app.com             |
   |  Access-Control-Allow-Methods: |
   |    GET, POST, PUT, DELETE      |
   |  Access-Control-Allow-Headers: |
   |    Content-Type, Authorization |
   |  Access-Control-Max-Age: 86400 |
   |                                |
   |  POST /api/users               |   ← Actual request
   |  Origin: https://app.com       |
   |  Content-Type: application/json|
   |                                |
   |  200 OK                        |   ← Actual response
   |  Access-Control-Allow-Origin:  |
   |    https://app.com             |
```

### Production CORS Middleware

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"os"
	"strconv"
	"strings"
)

// CORSConfig configures CORS behavior.
type CORSConfig struct {
	// AllowedOrigins is the list of origins that are allowed to make requests.
	// Use specific origins like "https://app.example.com" -- never "*" with credentials.
	AllowedOrigins []string

	// AllowedMethods is the list of HTTP methods allowed.
	AllowedMethods []string

	// AllowedHeaders is the list of headers the client is allowed to send.
	AllowedHeaders []string

	// ExposedHeaders is the list of headers the client is allowed to read from the response.
	ExposedHeaders []string

	// AllowCredentials indicates whether the request can include credentials
	// (cookies, HTTP authentication, client-side SSL certificates).
	// WARNING: If true, AllowedOrigins MUST NOT contain "*".
	AllowCredentials bool

	// MaxAge is how long (in seconds) the browser should cache the preflight response.
	MaxAge int
}

// CORS returns middleware that handles Cross-Origin Resource Sharing.
func CORS(cfg CORSConfig) func(http.Handler) http.Handler {
	// Build a set for O(1) origin lookups instead of scanning a slice on every request.
	// In a production API receiving thousands of requests per second, this matters.
	allowedOriginSet := make(map[string]bool, len(cfg.AllowedOrigins))
	for _, o := range cfg.AllowedOrigins {
		allowedOriginSet[o] = true
	}

	// Pre-compute header values that don't change per request.
	methodsHeader := strings.Join(cfg.AllowedMethods, ", ")
	headersHeader := strings.Join(cfg.AllowedHeaders, ", ")
	exposedHeader := strings.Join(cfg.ExposedHeaders, ", ")
	maxAgeHeader := strconv.Itoa(cfg.MaxAge)

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")

			// No Origin header means this is not a CORS request (e.g., same-origin,
			// curl, or server-to-server). Skip CORS handling entirely.
			if origin == "" {
				next.ServeHTTP(w, r)
				return
			}

			// Check if the origin is allowed
			if !allowedOriginSet[origin] {
				// Origin not allowed. Do NOT set any CORS headers.
				// The browser will block the response on the client side.
				if r.Method == http.MethodOptions {
					w.WriteHeader(http.StatusForbidden)
					return
				}
				next.ServeHTTP(w, r)
				return
			}

			// Set CORS headers for allowed origins.
			// IMPORTANT: We echo back the specific origin, not "*".
			// Using "*" with credentials is forbidden by the CORS spec.
			w.Header().Set("Access-Control-Allow-Origin", origin)

			if cfg.AllowCredentials {
				w.Header().Set("Access-Control-Allow-Credentials", "true")
			}

			if exposedHeader != "" {
				w.Header().Set("Access-Control-Expose-Headers", exposedHeader)
			}

			// Vary header tells caches that the response varies based on Origin.
			// Without this, a CDN might cache a response for origin A and serve it
			// to origin B, which would have the wrong CORS headers.
			w.Header().Add("Vary", "Origin")

			// Handle preflight requests
			if r.Method == http.MethodOptions {
				w.Header().Set("Access-Control-Allow-Methods", methodsHeader)
				w.Header().Set("Access-Control-Allow-Headers", headersHeader)
				if cfg.MaxAge > 0 {
					w.Header().Set("Access-Control-Max-Age", maxAgeHeader)
				}
				// Respond with 204 No Content for preflight requests.
				// Some older clients expect 200, but 204 is correct per the spec.
				w.WriteHeader(http.StatusNoContent)
				return
			}

			// Not a preflight -- pass through to the next handler
			next.ServeHTTP(w, r)
		})
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/users", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode([]map[string]string{
			{"id": "1", "name": "Alice"},
		})
	})

	handler := CORS(CORSConfig{
		AllowedOrigins:   []string{"https://app.example.com", "http://localhost:3000"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
		AllowedHeaders:   []string{"Content-Type", "Authorization", "X-Request-ID"},
		ExposedHeaders:   []string{"X-Request-ID", "X-RateLimit-Remaining"},
		AllowCredentials: true,
		MaxAge:           86400, // 24 hours
	})(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	http.ListenAndServe(":8080", handler)
}
```

### Common CORS Mistakes

**Mistake 1: Using `*` with credentials.**

```go
// WRONG -- the browser will reject this
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Credentials", "true")
```

The CORS specification explicitly forbids `Access-Control-Allow-Origin: *` when `Access-Control-Allow-Credentials: true`. The browser will block the response. You must echo the specific origin.

**Mistake 2: Forgetting the Vary header.**

Without `Vary: Origin`, a CDN or browser cache might cache the response for one origin and serve the cached response (with the wrong `Access-Control-Allow-Origin` value) to a request from a different origin.

**Mistake 3: Not handling preflight (OPTIONS) requests.**

If your router does not have an OPTIONS handler for the endpoint, the preflight request returns 405 Method Not Allowed, and the browser blocks the actual request. CORS middleware must intercept OPTIONS before routing.

### Node.js/Express Comparison

```javascript
const cors = require('cors');

app.use(cors({
  origin: ['https://app.example.com', 'http://localhost:3000'],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-Request-ID', 'X-RateLimit-Remaining'],
  credentials: true,
  maxAge: 86400,
}));
```

The Express `cors` package handles all of this in a few lines. Go requires you to implement it yourself (or use a library like `rs/cors`). The upside is that you understand exactly what headers are being set and can customize behavior for edge cases that the Express package's options do not cover.

---

## 5. Rate Limiting Middleware

### Why Rate Limiting at the Application Level

Even if you have rate limiting at the infrastructure level (API gateway, Nginx, Cloudflare), application-level rate limiting gives you fine-grained control per-endpoint, per-user, or per-API-key. It is your last line of defense against abuse.

### Token Bucket Algorithm

The token bucket algorithm is simple and effective:
- Each client has a bucket that holds up to `N` tokens
- The bucket starts full
- Each request removes one token
- Tokens are added at a fixed rate (e.g., 10 per second)
- If the bucket is empty, the request is rejected

```
Bucket capacity: 5 tokens
Refill rate: 1 token per second

Time 0s: [*][*][*][*][*]  ← Full (5 tokens)
Request → [*][*][*][*][ ]  ← 4 tokens remaining
Request → [*][*][*][ ][ ]  ← 3 tokens remaining
Time 1s:  [*][*][*][*][ ]  ← Refilled 1 token (4 tokens)
Request → [*][*][*][ ][ ]  ← 3 tokens remaining
Request → [*][*][ ][ ][ ]  ← 2 tokens remaining
Request → [*][ ][ ][ ][ ]  ← 1 token remaining
Request → [ ][ ][ ][ ][ ]  ← 0 tokens -- next request REJECTED
Time 1s:  [*][ ][ ][ ][ ]  ← Refilled 1 token
```

### Production Rate Limiting Middleware

```go
package main

import (
	"context"
	"encoding/json"
	"log/slog"
	"net"
	"net/http"
	"os"
	"strconv"
	"sync"
	"time"
)

// RateLimitConfig configures the rate limiter.
type RateLimitConfig struct {
	// RequestsPerSecond is the steady-state rate of allowed requests per IP.
	RequestsPerSecond float64

	// BurstSize is the maximum number of requests allowed in a burst.
	BurstSize int

	// CleanupInterval is how often stale entries are removed from the visitor map.
	CleanupInterval time.Duration

	// StaleAfter is how long after the last request before a visitor is considered stale.
	StaleAfter time.Duration

	Logger *slog.Logger
}

// visitor tracks the rate limit state for a single IP address.
type visitor struct {
	tokens    float64
	lastCheck time.Time
	mu        sync.Mutex
}

// allow checks whether the visitor has tokens available.
// It refills tokens based on elapsed time since the last check.
func (v *visitor) allow(rate float64, burst int) bool {
	v.mu.Lock()
	defer v.mu.Unlock()

	now := time.Now()
	elapsed := now.Sub(v.lastCheck).Seconds()
	v.lastCheck = now

	// Add tokens based on elapsed time
	v.tokens += elapsed * rate
	if v.tokens > float64(burst) {
		v.tokens = float64(burst)
	}

	// Check if there is at least one token available
	if v.tokens < 1.0 {
		return false
	}

	v.tokens -= 1.0
	return true
}

// rateLimiter manages per-IP rate limiting.
type rateLimiter struct {
	visitors map[string]*visitor
	mu       sync.Mutex
	rate     float64
	burst    int
}

func newRateLimiter(rate float64, burst int) *rateLimiter {
	return &rateLimiter{
		visitors: make(map[string]*visitor),
		rate:     rate,
		burst:    burst,
	}
}

// getVisitor returns the visitor for the given IP, creating one if it doesn't exist.
func (rl *rateLimiter) getVisitor(ip string) *visitor {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	v, exists := rl.visitors[ip]
	if !exists {
		v = &visitor{
			tokens:    float64(rl.burst), // Start with a full bucket
			lastCheck: time.Now(),
		}
		rl.visitors[ip] = v
	}
	return v
}

// cleanup removes visitors that haven't been seen since the given cutoff time.
func (rl *rateLimiter) cleanup(staleAfter time.Duration) {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	cutoff := time.Now().Add(-staleAfter)
	for ip, v := range rl.visitors {
		v.mu.Lock()
		if v.lastCheck.Before(cutoff) {
			delete(rl.visitors, ip)
		}
		v.mu.Unlock()
	}
}

// startCleanup starts a goroutine that periodically removes stale visitors.
// The goroutine stops when the context is cancelled.
func (rl *rateLimiter) startCleanup(ctx context.Context, interval, staleAfter time.Duration) {
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				rl.cleanup(staleAfter)
			}
		}
	}()
}

// extractIP extracts the client IP from the request.
// It checks X-Forwarded-For first (for use behind a reverse proxy),
// then falls back to RemoteAddr.
func extractIP(r *http.Request) string {
	// In production, you should only trust X-Forwarded-For if your reverse
	// proxy sets it. An attacker can spoof this header to bypass rate limiting.
	// For a proxy you control, use the leftmost IP in X-Forwarded-For.
	if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
		// X-Forwarded-For: client, proxy1, proxy2
		// The first IP is the client
		parts := splitFirst(xff, ',')
		return trimSpace(parts)
	}

	if xri := r.Header.Get("X-Real-IP"); xri != "" {
		return trimSpace(xri)
	}

	// Fall back to RemoteAddr, stripping the port
	ip, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return ip
}

// splitFirst returns the substring before the first occurrence of sep.
func splitFirst(s string, sep byte) string {
	for i := 0; i < len(s); i++ {
		if s[i] == sep {
			return s[:i]
		}
	}
	return s
}

// trimSpace trims leading and trailing whitespace from a string.
func trimSpace(s string) string {
	start := 0
	end := len(s)
	for start < end && (s[start] == ' ' || s[start] == '\t') {
		start++
	}
	for end > start && (s[end-1] == ' ' || s[end-1] == '\t') {
		end--
	}
	return s[start:end]
}

// RateLimit returns middleware that limits requests per IP address.
func RateLimit(ctx context.Context, cfg RateLimitConfig) func(http.Handler) http.Handler {
	limiter := newRateLimiter(cfg.RequestsPerSecond, cfg.BurstSize)

	// Start the cleanup goroutine, bound to the context's lifecycle.
	// When the server shuts down and the context is cancelled, the cleanup
	// goroutine exits cleanly -- no goroutine leak.
	limiter.startCleanup(ctx, cfg.CleanupInterval, cfg.StaleAfter)

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ip := extractIP(r)
			v := limiter.getVisitor(ip)

			if !v.allow(cfg.RequestsPerSecond, cfg.BurstSize) {
				cfg.Logger.Warn("rate limit exceeded",
					slog.String("ip", ip),
					slog.String("path", r.URL.Path),
				)

				w.Header().Set("Content-Type", "application/json")
				w.Header().Set("Retry-After", strconv.Itoa(1)) // Retry after 1 second
				w.Header().Set("X-RateLimit-Limit", strconv.Itoa(cfg.BurstSize))
				w.Header().Set("X-RateLimit-Remaining", "0")
				w.WriteHeader(http.StatusTooManyRequests)
				json.NewEncoder(w).Encode(map[string]string{
					"error": "rate limit exceeded",
				})
				return
			}

			// Add rate limit headers to the response so clients can
			// see how many requests they have remaining.
			w.Header().Set("X-RateLimit-Limit", strconv.Itoa(cfg.BurstSize))

			next.ServeHTTP(w, r)
		})
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Create a context that will be cancelled on shutdown.
	// This ensures the cleanup goroutine exits cleanly.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"data": "hello"})
	})

	handler := RateLimit(ctx, RateLimitConfig{
		RequestsPerSecond: 10,  // 10 requests per second steady state
		BurstSize:         20,  // Allow bursts up to 20 requests
		CleanupInterval:   time.Minute,
		StaleAfter:        5 * time.Minute,
		Logger:            logger,
	})(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	http.ListenAndServe(":8080", handler)
}
```

### Context-Based Goroutine Lifecycle

Notice how the cleanup goroutine's lifecycle is tied to a `context.Context`:

```go
limiter.startCleanup(ctx, cfg.CleanupInterval, cfg.StaleAfter)
```

When the server shuts down and `cancel()` is called, the cleanup goroutine detects `ctx.Done()` and returns. Without this, the goroutine would leak -- it would run forever, even after the server stops. This is a fundamental Go pattern: **always give long-running goroutines a way to stop**.

### The Visitor Map and sync.Mutex

The rate limiter uses two levels of mutexing:

1. **`rateLimiter.mu`** protects the `visitors` map (reads and writes to the map itself)
2. **`visitor.mu`** protects each visitor's token state (the token bucket calculation)

This two-level approach prevents a global lock from serializing all requests. Different IPs can be rate-checked concurrently -- only requests from the same IP contend on the visitor-level lock.

### Node.js/Express Comparison

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 1000,       // 1 second window
  max: 10,              // 10 requests per window
  standardHeaders: true, // Return rate limit info in RateLimit-* headers
  legacyHeaders: false,
  message: { error: 'rate limit exceeded' },
  keyGenerator: (req) => req.ip, // Per-IP limiting
});

app.use(limiter);
```

The Express `express-rate-limit` package handles all of this in a few lines but uses a simpler fixed-window algorithm by default. The Go implementation above uses token bucket, which is smoother -- it doesn't have the edge case where a client can make `2 * max` requests at the boundary between two windows.

In production Node.js, you would use `rate-limit-redis` for distributed rate limiting across multiple instances. In Go, you would use a similar approach with Redis, or use a distributed rate limiter like the one in `golang.org/x/time/rate` combined with a shared store.

---

## 6. Security Headers Middleware

### Why Security Headers Matter

Every response from your API should include security headers that tell the browser how to handle the content. These headers prevent a class of attacks including XSS, clickjacking, MIME sniffing, and protocol downgrade attacks. They cost nothing to implement and provide significant defense in depth.

### Production Security Headers Middleware

```go
package main

import (
	"net/http"
)

// SecurityHeadersConfig configures which security headers to set.
type SecurityHeadersConfig struct {
	// ContentSecurityPolicy controls which resources the browser is allowed to load.
	// Example: "default-src 'self'; script-src 'self'"
	ContentSecurityPolicy string

	// HSTSMaxAge enables HTTP Strict Transport Security.
	// The value is the number of seconds the browser should remember to only use HTTPS.
	// Set to 0 to disable. Typical production value: 31536000 (1 year).
	HSTSMaxAge int

	// HSTSIncludeSubdomains includes all subdomains in the HSTS policy.
	HSTSIncludeSubdomains bool

	// FrameOptions controls whether the page can be embedded in an iframe.
	// Values: "DENY", "SAMEORIGIN". Default: "DENY".
	FrameOptions string

	// ReferrerPolicy controls how much referrer information is sent.
	// Values: "no-referrer", "strict-origin-when-cross-origin", etc.
	ReferrerPolicy string
}

// DefaultSecurityHeadersConfig returns a sensible default configuration.
func DefaultSecurityHeadersConfig() SecurityHeadersConfig {
	return SecurityHeadersConfig{
		ContentSecurityPolicy: "default-src 'self'",
		HSTSMaxAge:            31536000, // 1 year
		HSTSIncludeSubdomains: true,
		FrameOptions:          "DENY",
		ReferrerPolicy:        "strict-origin-when-cross-origin",
	}
}

// SecurityHeaders returns middleware that sets standard security headers on every response.
func SecurityHeaders(cfg SecurityHeadersConfig) func(http.Handler) http.Handler {
	// Pre-compute the HSTS header value since it doesn't change per request.
	var hstsValue string
	if cfg.HSTSMaxAge > 0 {
		hstsValue = "max-age=" + itoa(cfg.HSTSMaxAge)
		if cfg.HSTSIncludeSubdomains {
			hstsValue += "; includeSubDomains"
		}
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// X-Content-Type-Options: nosniff
			// Prevents the browser from MIME-sniffing the content type.
			// Without this, a browser might interpret a JSON response as HTML
			// and execute embedded scripts.
			w.Header().Set("X-Content-Type-Options", "nosniff")

			// X-Frame-Options
			// Prevents the page from being embedded in an iframe.
			// This blocks clickjacking attacks.
			if cfg.FrameOptions != "" {
				w.Header().Set("X-Frame-Options", cfg.FrameOptions)
			}

			// Content-Security-Policy
			// Controls which resources the browser can load.
			// For an API that only serves JSON, "default-src 'self'" is appropriate.
			if cfg.ContentSecurityPolicy != "" {
				w.Header().Set("Content-Security-Policy", cfg.ContentSecurityPolicy)
			}

			// Strict-Transport-Security (HSTS)
			// Tells the browser to only use HTTPS for this domain.
			// Only set this over HTTPS -- setting it over HTTP is meaningless and
			// some security scanners will flag it as a misconfiguration.
			if hstsValue != "" && r.TLS != nil {
				w.Header().Set("Strict-Transport-Security", hstsValue)
			}

			// X-XSS-Protection
			// Note: Modern browsers have deprecated this in favor of CSP.
			// Setting it to "0" disables the XSS auditor, which can actually
			// introduce vulnerabilities in older browsers. The recommended
			// approach is to rely on CSP and set this to "0".
			w.Header().Set("X-XSS-Protection", "0")

			// Referrer-Policy
			// Controls how much referrer information is sent with requests.
			if cfg.ReferrerPolicy != "" {
				w.Header().Set("Referrer-Policy", cfg.ReferrerPolicy)
			}

			// Permissions-Policy (formerly Feature-Policy)
			// Disables browser features you don't need.
			// For an API, disable everything.
			w.Header().Set("Permissions-Policy", "camera=(), microphone=(), geolocation=()")

			next.ServeHTTP(w, r)
		})
	}
}

// itoa converts an int to a string without importing strconv.
func itoa(n int) string {
	if n == 0 {
		return "0"
	}
	buf := make([]byte, 0, 20)
	for n > 0 {
		buf = append(buf, byte('0'+n%10))
		n /= 10
	}
	// Reverse
	for i, j := 0, len(buf)-1; i < j; i, j = i+1, j-1 {
		buf[i], buf[j] = buf[j], buf[i]
	}
	return string(buf)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`{"status":"ok"}`))
	})

	handler := SecurityHeaders(DefaultSecurityHeadersConfig())(mux)

	http.ListenAndServe(":8080", handler)
}
```

### What Each Header Does

```
+----------------------------+-----------------------------------------------+----------------------------+
| Header                     | Purpose                                       | Attack Prevented           |
+----------------------------+-----------------------------------------------+----------------------------+
| X-Content-Type-Options     | Prevents MIME sniffing                        | XSS via content sniffing   |
| X-Frame-Options            | Prevents iframe embedding                     | Clickjacking               |
| Content-Security-Policy    | Controls resource loading                     | XSS, code injection        |
| Strict-Transport-Security  | Forces HTTPS                                  | Protocol downgrade, MITM   |
| X-XSS-Protection: 0       | Disables buggy browser XSS auditor            | XSS auditor bypass         |
| Referrer-Policy            | Controls referrer leakage                     | Information disclosure     |
| Permissions-Policy         | Disables browser APIs                         | Feature abuse              |
+----------------------------+-----------------------------------------------+----------------------------+
```

### Node.js/Express Comparison

```javascript
const helmet = require('helmet');

// helmet sets all these headers automatically
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
  },
  frameguard: { action: 'deny' },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));
```

Express has the `helmet` package which wraps all of this up nicely. In Go, you write it yourself -- but it is only ~50 lines of code, and you have full control over every header. No magic, no hidden defaults.

---

## 7. Body Size Limiting Middleware

### Why Limit Request Body Size

Without a body size limit, an attacker can send a 10GB POST request to your API. Your server will allocate memory to read and parse it, potentially crashing the process or triggering the OOM killer. Body size limiting is a simple but critical defense.

### Using http.MaxBytesReader

Go's standard library provides `http.MaxBytesReader`, which wraps a reader and returns an error if the body exceeds the limit. This is the idiomatic way to limit body size in Go:

```go
package main

import (
	"encoding/json"
	"errors"
	"net/http"
)

// BodyLimitConfig configures the body size limiter.
type BodyLimitConfig struct {
	// MaxBytes is the maximum allowed request body size in bytes.
	MaxBytes int64
}

// BodyLimit returns middleware that limits the size of request bodies.
func BodyLimit(cfg BodyLimitConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Only limit body size for methods that have a body.
			// GET, HEAD, OPTIONS, and TRACE typically don't have bodies.
			if r.Method == http.MethodPost ||
				r.Method == http.MethodPut ||
				r.Method == http.MethodPatch {

				// http.MaxBytesReader wraps the body and returns an error
				// if reading exceeds maxBytes. It also sets a flag on the
				// response that causes the server to close the connection,
				// preventing the client from continuing to send data.
				r.Body = http.MaxBytesReader(w, r.Body, cfg.MaxBytes)
			}

			next.ServeHTTP(w, r)
		})
	}
}

// handleUpload demonstrates how MaxBytesReader works with JSON decoding.
func handleUpload(w http.ResponseWriter, r *http.Request) {
	var payload map[string]interface{}
	err := json.NewDecoder(r.Body).Decode(&payload)
	if err != nil {
		// Check if the error is due to the body being too large.
		// http.MaxBytesError is available since Go 1.19.
		var maxBytesErr *http.MaxBytesError
		if errors.As(err, &maxBytesErr) {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusRequestEntityTooLarge)
			json.NewEncoder(w).Encode(map[string]string{
				"error": "request body too large",
			})
			return
		}

		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{
			"error": "invalid request body",
		})
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "received"})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /api/upload", handleUpload)

	// Limit request bodies to 1 MB
	handler := BodyLimit(BodyLimitConfig{
		MaxBytes: 1 << 20, // 1 MB = 1,048,576 bytes
	})(mux)

	http.ListenAndServe(":8080", handler)
}
```

### How http.MaxBytesReader Works Internally

`http.MaxBytesReader` returns a custom `io.ReadCloser` that:
1. Tracks how many bytes have been read
2. Returns an error once the limit is exceeded
3. Sets a flag on the underlying connection to close it after the response

This third point is important: it prevents a slow-loris style attack where the client sends data forever. Once the limit is hit, the server will close the connection after sending the response.

### Per-Route Limits

Different endpoints might need different limits. A file upload endpoint might allow 50MB while a JSON API endpoint allows 1MB:

```go
package main

import (
	"encoding/json"
	"net/http"
)

// BodyLimitForRoute wraps a single handler with a body size limit.
// Use this when different routes need different limits.
func BodyLimitForRoute(maxBytes int64, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
		next(w, r)
	}
}

func main() {
	mux := http.NewServeMux()

	// JSON endpoints: 1 MB limit
	mux.HandleFunc("POST /api/users", BodyLimitForRoute(1<<20, func(w http.ResponseWriter, r *http.Request) {
		var user map[string]string
		json.NewDecoder(r.Body).Decode(&user)
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(user)
	}))

	// File upload endpoint: 50 MB limit
	mux.HandleFunc("POST /api/upload", BodyLimitForRoute(50<<20, func(w http.ResponseWriter, r *http.Request) {
		// Handle file upload
		w.Write([]byte(`{"status":"uploaded"}`))
	}))

	http.ListenAndServe(":8080", mux)
}
```

### Node.js/Express Comparison

```javascript
const express = require('express');
const app = express();

// Global body limit
app.use(express.json({ limit: '1mb' }));

// Per-route limit
app.post('/api/upload', express.raw({ limit: '50mb' }), (req, res) => {
  res.json({ status: 'uploaded' });
});
```

Express handles this with the `limit` option in body parser middleware. Go's `http.MaxBytesReader` is lower-level but more explicit -- you see exactly where the limit is applied and what happens when it is exceeded.

---

## 8. Request ID Middleware

### Why Every Request Needs an ID

In a distributed system, a single user action triggers multiple HTTP requests across multiple services. When something fails, you need to trace the request through every service it touched. A request ID is a unique identifier attached to every request that flows through the entire system.

- **Debugging**: Search logs by request ID to find every log line related to a single request
- **Correlation**: When Service A calls Service B, it passes the request ID so both services' logs can be correlated
- **Support**: Users can report their request ID so support can trace exactly what happened

### Log Injection Prevention

A critical security concern: if the request ID comes from the client (via a header), an attacker can inject malicious content. Consider a request ID like `abc123\n{"level":"INFO","msg":"admin logged in"}`. If this is logged verbatim, it injects a fake log line. You must validate request IDs:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"log/slog"
	"net/http"
	"os"
)

// contextKey is an unexported type for context keys to prevent collisions.
type contextKey string

const requestIDKey contextKey = "request_id"

// isValidRequestID checks if a request ID is safe to use in logs.
// Only alphanumeric characters, hyphens, and underscores are allowed.
// This prevents log injection attacks.
func isValidRequestID(id string) bool {
	if len(id) == 0 || len(id) > 128 {
		return false
	}
	for _, c := range id {
		if !((c >= 'a' && c <= 'z') ||
			(c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') ||
			c == '-' || c == '_') {
			return false
		}
	}
	return true
}

// generateRequestID creates a new random request ID.
// Using crypto/rand ensures the ID is unpredictable.
func generateRequestID() string {
	b := make([]byte, 16) // 128-bit ID
	_, err := rand.Read(b)
	if err != nil {
		// crypto/rand.Read should never fail on a modern OS.
		// If it does, something is seriously wrong.
		panic(fmt.Sprintf("crypto/rand.Read failed: %v", err))
	}
	return hex.EncodeToString(b)
}

// RequestIDConfig configures the request ID middleware.
type RequestIDConfig struct {
	// HeaderName is the header used to propagate the request ID.
	// Default: "X-Request-ID"
	HeaderName string

	// TrustIncoming determines whether to trust the request ID from the client.
	// Set to true only if your API is behind a trusted reverse proxy that sets
	// the header. Set to false if the API is directly exposed to the internet.
	TrustIncoming bool
}

// RequestID returns middleware that ensures every request has a unique ID.
func RequestID(cfg RequestIDConfig) func(http.Handler) http.Handler {
	if cfg.HeaderName == "" {
		cfg.HeaderName = "X-Request-ID"
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			var id string

			// If we trust incoming headers, check for an existing request ID.
			if cfg.TrustIncoming {
				incoming := r.Header.Get(cfg.HeaderName)
				if incoming != "" && isValidRequestID(incoming) {
					id = incoming
				}
			}

			// Generate a new ID if none was provided or the incoming one was invalid.
			if id == "" {
				id = generateRequestID()
			}

			// Set the request ID on the response header so the client can see it.
			// This is useful for debugging -- users can report the request ID.
			w.Header().Set(cfg.HeaderName, id)

			// Store the request ID in the request header so downstream handlers
			// and middleware can access it via r.Header.Get("X-Request-ID").
			r.Header.Set(cfg.HeaderName, id)

			// Also store it in the request context for programmatic access.
			ctx := context.WithValue(r.Context(), requestIDKey, id)
			r = r.WithContext(ctx)

			next.ServeHTTP(w, r)
		})
	}
}

// GetRequestID extracts the request ID from the context.
func GetRequestID(ctx context.Context) string {
	id, ok := ctx.Value(requestIDKey).(string)
	if !ok {
		return ""
	}
	return id
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/users", func(w http.ResponseWriter, r *http.Request) {
		reqID := GetRequestID(r.Context())
		logger.Info("handling users request",
			slog.String("request_id", reqID),
		)
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`[{"id":1,"name":"Alice"}]`))
	})

	handler := RequestID(RequestIDConfig{
		HeaderName:    "X-Request-ID",
		TrustIncoming: false, // Don't trust client-provided IDs
	})(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	http.ListenAndServe(":8080", handler)
}
```

### Using the Request ID in Logging Middleware

The request ID middleware should run before the logging middleware so the logger can include the request ID in every log line. Here is how they work together:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

type ctxKey string

const reqIDCtxKey ctxKey = "request_id"

func genID() string {
	b := make([]byte, 16)
	rand.Read(b)
	return hex.EncodeToString(b)
}

func requestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := genID()
		w.Header().Set("X-Request-ID", id)
		ctx := context.WithValue(r.Context(), reqIDCtxKey, id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

type statusWriter struct {
	http.ResponseWriter
	code int
}

func (sw *statusWriter) WriteHeader(code int) {
	sw.code = code
	sw.ResponseWriter.WriteHeader(code)
}

func loggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			sw := &statusWriter{ResponseWriter: w, code: 200}

			next.ServeHTTP(sw, r)

			// Extract request ID from context (set by requestIDMiddleware)
			reqID, _ := r.Context().Value(reqIDCtxKey).(string)

			logger.Info("request completed",
				slog.String("request_id", reqID),
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", sw.code),
				slog.String("duration", time.Since(start).String()),
			)
		})
	}
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"data":"hello"}`))
	})

	// Order matters: RequestID runs first (outermost), then Logging
	handler := requestIDMiddleware(loggingMiddleware(logger)(mux))

	http.ListenAndServe(":8080", handler)
}
```

Output:

```json
{
  "time": "2026-03-24T10:30:00.000Z",
  "level": "INFO",
  "msg": "request completed",
  "request_id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
  "method": "GET",
  "path": "/api/data",
  "status": 200,
  "duration": "0.5ms"
}
```

### Node.js/Express Comparison

```javascript
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  const id = req.headers['x-request-id'] || uuidv4();
  req.requestId = id;
  res.setHeader('X-Request-ID', id);
  next();
});

// In subsequent middleware/handlers:
app.use((req, res, next) => {
  console.log(`[${req.requestId}] ${req.method} ${req.path}`);
  next();
});
```

The pattern is identical. The Go version uses `context.WithValue` to propagate the ID through the request, while Express attaches it directly to the `req` object. Go's approach is more explicit and type-safe -- you cannot accidentally access a property that does not exist, and the context key type prevents collisions between different middleware.

---

## 9. Authentication Middleware

### JWT Authentication for APIs

Most production Go APIs use JWT (JSON Web Tokens) for authentication. The middleware extracts the token from the request, validates it, and injects the user's claims into the request context. Downstream handlers can then access the authenticated user without knowing anything about JWT.

### Token Extraction: Cookie vs Header

A well-designed auth middleware should check multiple sources for the token:
1. **httpOnly cookie** (preferred for web clients -- the browser manages token storage)
2. **Authorization header** (for mobile clients, CLI tools, and service-to-service calls)

```go
package main

import (
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strings"
	"time"
)

// ---------- JWT Utilities ----------

// Claims represents the payload of a JWT.
type Claims struct {
	UserID    string `json:"sub"`
	Email     string `json:"email"`
	Role      string `json:"role"`
	IssuedAt  int64  `json:"iat"`
	ExpiresAt int64  `json:"exp"`
}

// jwtHeader is the standard JWT header for HMAC-SHA256.
var jwtHeader = base64.RawURLEncoding.EncodeToString([]byte(`{"alg":"HS256","typ":"JWT"}`))

// SignJWT creates a signed JWT token from claims.
// In production, use a well-tested library like golang-jwt/jwt.
// This implementation is for educational purposes.
func SignJWT(claims Claims, secret []byte) (string, error) {
	payload, err := json.Marshal(claims)
	if err != nil {
		return "", fmt.Errorf("marshal claims: %w", err)
	}
	encodedPayload := base64.RawURLEncoding.EncodeToString(payload)

	signingInput := jwtHeader + "." + encodedPayload
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(signingInput))
	signature := base64.RawURLEncoding.EncodeToString(mac.Sum(nil))

	return signingInput + "." + signature, nil
}

// VerifyJWT validates a JWT and returns the claims.
func VerifyJWT(tokenStr string, secret []byte) (*Claims, error) {
	parts := strings.Split(tokenStr, ".")
	if len(parts) != 3 {
		return nil, errors.New("invalid token format")
	}

	// Verify signature
	signingInput := parts[0] + "." + parts[1]
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(signingInput))
	expectedSig := base64.RawURLEncoding.EncodeToString(mac.Sum(nil))

	if !hmac.Equal([]byte(parts[2]), []byte(expectedSig)) {
		return nil, errors.New("invalid signature")
	}

	// Decode claims
	payload, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		return nil, fmt.Errorf("decode payload: %w", err)
	}

	var claims Claims
	if err := json.Unmarshal(payload, &claims); err != nil {
		return nil, fmt.Errorf("unmarshal claims: %w", err)
	}

	// Check expiration
	if claims.ExpiresAt < time.Now().Unix() {
		return nil, errors.New("token expired")
	}

	return &claims, nil
}

// ---------- Auth Middleware ----------

type authContextKey string

const claimsKey authContextKey = "claims"

// AuthConfig configures the authentication middleware.
type AuthConfig struct {
	// JWTSecret is the HMAC secret used to verify JWT signatures.
	JWTSecret []byte

	// CookieName is the name of the httpOnly cookie containing the JWT.
	CookieName string

	// Logger for authentication events.
	Logger *slog.Logger

	// SkipPaths are paths that do not require authentication.
	SkipPaths map[string]bool
}

// Auth returns middleware that validates JWT tokens from cookies or headers.
func Auth(cfg AuthConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Skip authentication for public paths
			if cfg.SkipPaths[r.URL.Path] {
				next.ServeHTTP(w, r)
				return
			}

			// Extract the token -- try cookie first, then Authorization header.
			token := extractToken(r, cfg.CookieName)
			if token == "" {
				cfg.Logger.Warn("no auth token found",
					slog.String("path", r.URL.Path),
					slog.String("remote_addr", r.RemoteAddr),
				)
				writeAuthError(w, "authentication required", http.StatusUnauthorized)
				return
			}

			// Validate the token
			claims, err := VerifyJWT(token, cfg.JWTSecret)
			if err != nil {
				cfg.Logger.Warn("invalid auth token",
					slog.String("error", err.Error()),
					slog.String("path", r.URL.Path),
					slog.String("remote_addr", r.RemoteAddr),
				)
				writeAuthError(w, "invalid or expired token", http.StatusUnauthorized)
				return
			}

			// Inject claims into the request context.
			// Downstream handlers access claims via GetClaims(r.Context()).
			ctx := context.WithValue(r.Context(), claimsKey, claims)

			// Also add the user ID to the logger context for structured logging.
			cfg.Logger.Debug("authenticated request",
				slog.String("user_id", claims.UserID),
				slog.String("email", claims.Email),
				slog.String("path", r.URL.Path),
			)

			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// extractToken tries to find a JWT in the request.
// Priority: 1) httpOnly cookie, 2) Authorization: Bearer header
func extractToken(r *http.Request, cookieName string) string {
	// Try the httpOnly cookie first.
	// Web browsers send this automatically -- no JavaScript needed to manage the token.
	if cookieName != "" {
		cookie, err := r.Cookie(cookieName)
		if err == nil && cookie.Value != "" {
			return cookie.Value
		}
	}

	// Fall back to the Authorization header.
	// Mobile apps and CLI tools typically use this.
	authHeader := r.Header.Get("Authorization")
	if authHeader != "" {
		// Expected format: "Bearer <token>"
		const prefix = "Bearer "
		if len(authHeader) > len(prefix) &&
			strings.EqualFold(authHeader[:len(prefix)], prefix) {
			return authHeader[len(prefix):]
		}
	}

	return ""
}

// GetClaims extracts the authenticated user's claims from the context.
func GetClaims(ctx context.Context) *Claims {
	claims, ok := ctx.Value(claimsKey).(*Claims)
	if !ok {
		return nil
	}
	return claims
}

// writeAuthError sends a standardized authentication error response.
func writeAuthError(w http.ResponseWriter, msg string, status int) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(map[string]string{"error": msg})
}

// ---------- Role-Based Authorization ----------

// RequireRole returns middleware that checks if the authenticated user has the required role.
// This MUST be applied after the Auth middleware.
func RequireRole(roles ...string) func(http.Handler) http.Handler {
	roleSet := make(map[string]bool, len(roles))
	for _, r := range roles {
		roleSet[r] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetClaims(r.Context())
			if claims == nil {
				writeAuthError(w, "authentication required", http.StatusUnauthorized)
				return
			}

			if !roleSet[claims.Role] {
				writeAuthError(w, "insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// ---------- Example Server ----------

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	jwtSecret := []byte("my-secret-key-change-in-production")

	// Generate a test token
	token, err := SignJWT(Claims{
		UserID:    "user-123",
		Email:     "alice@example.com",
		Role:      "admin",
		IssuedAt:  time.Now().Unix(),
		ExpiresAt: time.Now().Add(24 * time.Hour).Unix(),
	}, jwtSecret)
	if err != nil {
		panic(err)
	}
	logger.Info("test token generated", slog.String("token", token))

	mux := http.NewServeMux()

	// Public endpoint (no auth)
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"status":"ok"}`))
	})

	// Authenticated endpoint
	mux.HandleFunc("GET /api/profile", func(w http.ResponseWriter, r *http.Request) {
		claims := GetClaims(r.Context())
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"user_id": claims.UserID,
			"email":   claims.Email,
			"role":    claims.Role,
		})
	})

	// Admin-only endpoint: wrap with both Auth and RequireRole
	adminHandler := RequireRole("admin")(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`{"users":["alice","bob","charlie"]}`))
	}))
	mux.Handle("GET /api/admin/users", adminHandler)

	// Apply auth middleware globally, skipping public paths
	handler := Auth(AuthConfig{
		JWTSecret:  jwtSecret,
		CookieName: "auth_token",
		Logger:     logger,
		SkipPaths:  map[string]bool{"/health": true},
	})(mux)

	logger.Info("server starting", slog.String("addr", ":8080"))
	logger.Info("test with: curl -H 'Authorization: Bearer <token>' http://localhost:8080/api/profile")
	http.ListenAndServe(":8080", handler)
}
```

### Cookie-Based Auth: Why It Is Preferred for Web Clients

```
Authorization Header (Bearer token):
  - Client must store the token in localStorage or sessionStorage
  - Vulnerable to XSS: any JavaScript on the page can read the token
  - Must manually attach the header to every request

httpOnly Cookie:
  - Browser stores the token automatically
  - JavaScript CANNOT access it (httpOnly flag)
  - Browser sends it automatically with every request
  - Vulnerable to CSRF (mitigated with SameSite=Strict and CSRF tokens)
```

For web clients, httpOnly cookies are more secure because they are immune to XSS attacks. For mobile clients and CLI tools, the Authorization header is the standard approach. A well-designed middleware checks both.

### Node.js/Express Comparison

```javascript
const jwt = require('jsonwebtoken');

const auth = (req, res, next) => {
  // Try cookie first
  let token = req.cookies?.auth_token;

  // Fall back to Authorization header
  if (!token) {
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith('Bearer ')) {
      token = authHeader.slice(7);
    }
  }

  if (!token) {
    return res.status(401).json({ error: 'authentication required' });
  }

  try {
    const claims = jwt.verify(token, process.env.JWT_SECRET);
    req.user = claims;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'invalid or expired token' });
  }
};

// Apply to specific routes
app.get('/api/profile', auth, (req, res) => {
  res.json({ user_id: req.user.sub, email: req.user.email });
});

// Apply to all routes under /api
app.use('/api', auth);
```

The pattern is the same. Express attaches claims to `req.user`. Go uses `context.WithValue`. Express applies middleware per-route with the middleware argument. Go can do the same by wrapping specific handlers (Section 11).

---

## 10. Middleware Ordering

### Why Order Matters

Middleware order is not arbitrary. Each middleware depends on what the middleware before it has set up, and each middleware's failure mode determines what the middleware after it sees. Here is the recommended order for a production API and the reasoning behind it:

```
Request from client
         │
         ▼
┌─────────────────────┐
│  1. Recovery         │  ← MUST be first: catches panics from ALL subsequent middleware
├─────────────────────┤
│  2. Request ID       │  ← Early: all subsequent logs should include the request ID
├─────────────────────┤
│  3. Security Headers │  ← Early: headers should be set even if later middleware rejects
├─────────────────────┤
│  4. CORS             │  ← Before auth: preflight requests don't carry auth tokens
├─────────────────────┤
│  5. Rate Limiting    │  ← Before auth: prevents brute-force attacks on auth
├─────────────────────┤
│  6. Body Size Limit  │  ← Before parsing: prevents memory exhaustion from large bodies
├─────────────────────┤
│  7. Request Logging  │  ← After rate limit: don't log every rate-limited request at INFO
├─────────────────────┤
│  8. Authentication   │  ← Last in the chain: only authenticated requests reach handlers
├─────────────────────┤
│  9. Route Handler    │  ← Your business logic
└─────────────────────┘
```

### Detailed Reasoning

**Recovery first:** If any middleware panics, recovery catches it. If recovery is not the outermost layer, a panic in an outer middleware crashes the goroutine.

**Request ID early:** The logging middleware, auth middleware, and your handlers all want to include the request ID. It must be set before any of them run.

**Security Headers early:** Even if CORS rejects the request or rate limiting blocks it, the response should still have security headers. An attacker analyzing error responses should not see missing security headers.

**CORS before Auth:** Browser preflight (OPTIONS) requests do not carry cookies or authorization headers. If auth middleware runs before CORS, it will reject all preflight requests, and the browser will never send the actual request.

**Rate Limiting before Auth:** Rate limiting should apply to unauthenticated requests too. An attacker brute-forcing login should be rate-limited before the server spends CPU time verifying JWT signatures.

**Body Size Limit before Request Logging:** If the body is too large, we want to reject it before we start timing the request and logging it.

**Request Logging before Auth:** We want to log both authenticated and unauthenticated requests. If auth rejects a request, the logging middleware still captures the 401 response.

**Auth last:** By the time the request reaches auth, it has a request ID, security headers, has passed rate limiting and body size checks, and is being logged. The handler receives a fully validated, tracked request.

### Implementing the Order

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// Chain applies middleware in order. The first middleware in the list
// is the outermost (runs first on the way in, last on the way out).
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
	// Apply in reverse order so the first middleware in the list is outermost.
	for i := len(middlewares) - 1; i >= 0; i-- {
		handler = middlewares[i](handler)
	}
	return handler
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	ctx := context.Background()

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"data":"hello"}`))
	})

	// Apply middleware in the correct order.
	// Chain reads left-to-right as outermost-to-innermost.
	handler := Chain(mux,
		Recovery(logger),                          // 1. Catch panics
		RequestID(RequestIDConfig{}),               // 2. Assign request IDs
		SecurityHeaders(DefaultSecurityHeaders()),  // 3. Set security headers
		CORS(CORSConfig{                            // 4. Handle CORS
			AllowedOrigins: []string{"https://app.example.com"},
			AllowedMethods: []string{"GET", "POST", "PUT", "DELETE"},
			AllowedHeaders: []string{"Content-Type", "Authorization"},
		}),
		RateLimit(ctx, RateLimitConfig{             // 5. Rate limiting
			RequestsPerSecond: 10,
			BurstSize:         20,
			CleanupInterval:   time.Minute,
			StaleAfter:        5 * time.Minute,
			Logger:            logger,
		}),
		BodyLimit(BodyLimitConfig{MaxBytes: 1 << 20}), // 6. Body size limit
		Logging(LoggingConfig{Logger: logger}),         // 7. Request logging
		Auth(AuthConfig{                                // 8. Authentication
			JWTSecret:  []byte("secret"),
			CookieName: "auth_token",
			Logger:     logger,
			SkipPaths:  map[string]bool{"/health": true},
		}),
	)

	http.ListenAndServe(":8080", handler)
}
```

### Common Ordering Mistakes

| Mistake | Consequence |
|---------|------------|
| Auth before CORS | Browser preflight requests rejected with 401 |
| Logging before Recovery | Panic in logging middleware crashes goroutine |
| Auth before Rate Limit | Brute-force attacks exhaust JWT verification CPU |
| Body Limit after Handler | Large body already read into memory before limit check |
| Request ID after Logging | Log entries missing request ID |

---

## 11. Per-Route vs Global Middleware

### Global Middleware

Global middleware applies to every request. Recovery, logging, request ID, security headers, and CORS are typically global:

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"status":"ok"}`))
	})
	mux.HandleFunc("GET /api/public", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"data":"public"}`))
	})
	mux.HandleFunc("GET /api/private", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"data":"private"}`))
	})

	// Global middleware: applies to ALL routes including /health
	handler := Recovery(logger)(
		Logging(LoggingConfig{Logger: logger})(mux),
	)

	http.ListenAndServe(":8080", handler)
}
```

### Per-Route Middleware

Auth middleware should not apply to public routes (health checks, login, signup). You have several strategies:

**Strategy 1: SkipPaths configuration (shown in the Auth middleware above)**

```go
Auth(AuthConfig{
    SkipPaths: map[string]bool{
        "/health":       true,
        "/api/login":    true,
        "/api/register": true,
    },
})
```

**Strategy 2: Separate muxes for public and authenticated routes**

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Public routes (no auth required)
	publicMux := http.NewServeMux()
	publicMux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"status":"ok"}`))
	})
	publicMux.HandleFunc("POST /api/login", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"token":"abc123"}`))
	})

	// Authenticated routes
	privateMux := http.NewServeMux()
	privateMux.HandleFunc("GET /api/profile", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"user":"alice"}`))
	})
	privateMux.HandleFunc("GET /api/settings", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"theme":"dark"}`))
	})

	// Wrap private routes with auth middleware
	authedPrivate := Auth(AuthConfig{
		JWTSecret:  []byte("secret"),
		CookieName: "auth_token",
		Logger:     logger,
	})(privateMux)

	// Combine both into a single mux
	mainMux := http.NewServeMux()
	mainMux.Handle("/health", publicMux)
	mainMux.Handle("/api/login", publicMux)
	mainMux.Handle("/api/", authedPrivate) // Everything under /api/ except login

	// Apply global middleware
	handler := Recovery(logger)(
		Logging(LoggingConfig{Logger: logger})(mainMux),
	)

	http.ListenAndServe(":8080", handler)
}
```

**Strategy 3: Wrap individual handlers**

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
)

// wrapMiddleware applies middleware to a single handler function.
func wrapMiddleware(h http.HandlerFunc, mws ...func(http.Handler) http.Handler) http.Handler {
	var handler http.Handler = h
	for i := len(mws) - 1; i >= 0; i-- {
		handler = mws[i](handler)
	}
	return handler
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	authMW := Auth(AuthConfig{
		JWTSecret:  []byte("secret"),
		CookieName: "auth_token",
		Logger:     logger,
	})
	adminMW := RequireRole("admin")

	mux := http.NewServeMux()

	// Public routes
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"status":"ok"}`))
	})

	// Auth-required routes
	mux.Handle("GET /api/profile", wrapMiddleware(
		func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte(`{"user":"alice"}`))
		},
		authMW,
	))

	// Admin-required routes (auth + role check)
	mux.Handle("GET /api/admin/users", wrapMiddleware(
		func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte(`{"users":["alice","bob"]}`))
		},
		authMW,
		adminMW,
	))

	http.ListenAndServe(":8080", mux)
}
```

### Node.js/Express Comparison

Express makes per-route middleware particularly ergonomic:

```javascript
// Global middleware
app.use(cors());
app.use(helmet());
app.use(morgan('combined'));

// Per-route middleware
app.get('/api/profile', auth, (req, res) => { ... });
app.get('/api/admin/users', auth, requireRole('admin'), (req, res) => { ... });

// Per-group middleware
const apiRouter = express.Router();
apiRouter.use(auth);
apiRouter.get('/profile', (req, res) => { ... });
app.use('/api', apiRouter);
```

Go's standard library `http.ServeMux` does not have a built-in per-route middleware syntax. You wrap handlers manually. Third-party routers like `chi` and `gorilla/mux` provide group-level middleware:

```go
// Using chi router (not standard library)
r := chi.NewRouter()
r.Use(Recovery, Logging)          // Global middleware

r.Group(func(r chi.Router) {
    r.Use(Auth)                    // Group middleware
    r.Get("/api/profile", profileHandler)
    r.Get("/api/settings", settingsHandler)
})

r.Get("/health", healthHandler)   // No auth middleware
```

---

## 12. Composing Middleware

### The Chain Function

We saw `Chain` in Section 10. Here is the full implementation with detailed explanation:

```go
package main

import (
	"net/http"
)

// Chain applies a sequence of middleware to a handler.
// The first middleware in the list is the outermost layer (executes first).
//
// Chain(handler, A, B, C) is equivalent to A(B(C(handler))).
//
// Without Chain, you write:
//   handler := A(B(C(mux)))
// With Chain, you write:
//   handler := Chain(mux, A, B, C)
//
// Chain reads left-to-right in execution order, which is much easier to reason about.
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		handler = middlewares[i](handler)
	}
	return handler
}

// Middleware is a named type for middleware functions.
// This makes it slightly easier to work with middleware in some contexts.
type Middleware = func(http.Handler) http.Handler

// MiddlewareStack holds a reusable collection of middleware.
type MiddlewareStack struct {
	middlewares []Middleware
}

// NewMiddlewareStack creates a new middleware stack.
func NewMiddlewareStack(mws ...Middleware) *MiddlewareStack {
	return &MiddlewareStack{middlewares: mws}
}

// Use adds middleware to the stack.
func (s *MiddlewareStack) Use(mw Middleware) {
	s.middlewares = append(s.middlewares, mw)
}

// Then applies all middleware in the stack to the given handler.
func (s *MiddlewareStack) Then(handler http.Handler) http.Handler {
	return Chain(handler, s.middlewares...)
}

// ThenFunc applies all middleware in the stack to the given handler function.
func (s *MiddlewareStack) ThenFunc(fn http.HandlerFunc) http.Handler {
	return s.Then(fn)
}
```

### Reusable Middleware Stacks

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	ctx := context.Background()

	// Base stack: middleware that every endpoint needs
	base := NewMiddlewareStack(
		Recovery(logger),
		RequestID(RequestIDConfig{}),
		SecurityHeaders(DefaultSecurityHeadersConfig()),
		CORS(CORSConfig{
			AllowedOrigins:   []string{"https://app.example.com"},
			AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE"},
			AllowedHeaders:   []string{"Content-Type", "Authorization"},
			AllowCredentials: true,
		}),
		RateLimit(ctx, RateLimitConfig{
			RequestsPerSecond: 10,
			BurstSize:         20,
			CleanupInterval:   time.Minute,
			StaleAfter:        5 * time.Minute,
			Logger:            logger,
		}),
		BodyLimit(BodyLimitConfig{MaxBytes: 1 << 20}),
		Logging(LoggingConfig{Logger: logger}),
	)

	// Auth stack: base + authentication
	authStack := NewMiddlewareStack(
		Recovery(logger),
		RequestID(RequestIDConfig{}),
		SecurityHeaders(DefaultSecurityHeadersConfig()),
		CORS(CORSConfig{
			AllowedOrigins:   []string{"https://app.example.com"},
			AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE"},
			AllowedHeaders:   []string{"Content-Type", "Authorization"},
			AllowCredentials: true,
		}),
		RateLimit(ctx, RateLimitConfig{
			RequestsPerSecond: 10,
			BurstSize:         20,
			CleanupInterval:   time.Minute,
			StaleAfter:        5 * time.Minute,
			Logger:            logger,
		}),
		BodyLimit(BodyLimitConfig{MaxBytes: 1 << 20}),
		Logging(LoggingConfig{Logger: logger}),
		Auth(AuthConfig{
			JWTSecret:  []byte("secret"),
			CookieName: "auth_token",
			Logger:     logger,
		}),
	)

	// Use the stacks
	publicMux := http.NewServeMux()
	publicMux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"status":"ok"}`))
	})

	privateMux := http.NewServeMux()
	privateMux.HandleFunc("GET /api/profile", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`{"user":"alice"}`))
	})

	mainMux := http.NewServeMux()
	mainMux.Handle("/health", base.Then(publicMux))
	mainMux.Handle("/api/", authStack.Then(privateMux))

	http.ListenAndServe(":8080", mainMux)
}
```

### Conditional Middleware

Sometimes you want middleware that only runs under certain conditions:

```go
package main

import (
	"net/http"
	"os"
)

// ConditionalMiddleware wraps middleware that only runs when the condition is true.
func ConditionalMiddleware(condition bool, mw func(http.Handler) http.Handler) func(http.Handler) http.Handler {
	if condition {
		return mw
	}
	// Return a no-op middleware that just passes through
	return func(next http.Handler) http.Handler {
		return next
	}
}

func main() {
	isDev := os.Getenv("ENV") == "development"

	mux := http.NewServeMux()
	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("hello"))
	})

	// Only enable verbose logging in development
	handler := Chain(mux,
		Recovery(nil),
		ConditionalMiddleware(isDev, VerboseLogging()),
		ConditionalMiddleware(!isDev, ProductionLogging()),
	)

	http.ListenAndServe(":8080", handler)
}
```

---

## 13. Testing Middleware

### The httptest Package

Go's `net/http/httptest` package provides `httptest.NewRecorder()` and `httptest.NewRequest()` for testing HTTP handlers and middleware without starting a real server:

```go
package middleware_test

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

// Simple middleware for testing purposes
func addHeader(key, value string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Header().Set(key, value)
			next.ServeHTTP(w, r)
		})
	}
}

func TestAddHeader(t *testing.T) {
	// Create a simple inner handler
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	})

	// Wrap with middleware
	handler := addHeader("X-Custom", "test-value")(inner)

	// Create a test request
	req := httptest.NewRequest(http.MethodGet, "/test", nil)
	rec := httptest.NewRecorder()

	// Execute
	handler.ServeHTTP(rec, req)

	// Assert
	if got := rec.Header().Get("X-Custom"); got != "test-value" {
		t.Errorf("X-Custom header = %q, want %q", got, "test-value")
	}
	if rec.Code != http.StatusOK {
		t.Errorf("status code = %d, want %d", rec.Code, http.StatusOK)
	}
}
```

### Testing Recovery Middleware

```go
package middleware_test

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"
)

func TestRecoveryMiddleware(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))

	// Handler that panics
	panicHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		panic("test panic")
	})

	handler := Recovery(logger)(panicHandler)

	req := httptest.NewRequest(http.MethodGet, "/panic", nil)
	rec := httptest.NewRecorder()

	// This should NOT panic -- the middleware catches it
	handler.ServeHTTP(rec, req)

	// Should return 500
	if rec.Code != http.StatusInternalServerError {
		t.Errorf("status code = %d, want %d", rec.Code, http.StatusInternalServerError)
	}

	// Should return JSON error
	var body map[string]string
	if err := json.Unmarshal(rec.Body.Bytes(), &body); err != nil {
		t.Fatalf("failed to unmarshal response body: %v", err)
	}
	if body["error"] != "internal server error" {
		t.Errorf("error message = %q, want %q", body["error"], "internal server error")
	}
}

func TestRecoveryMiddleware_NoPanic(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))

	normalHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"status":"ok"}`))
	})

	handler := Recovery(logger)(normalHandler)

	req := httptest.NewRequest(http.MethodGet, "/normal", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("status code = %d, want %d", rec.Code, http.StatusOK)
	}
}
```

### Testing CORS Middleware

```go
package middleware_test

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestCORSMiddleware_PreflightRequest(t *testing.T) {
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t.Error("inner handler should not be called for preflight")
	})

	handler := CORS(CORSConfig{
		AllowedOrigins:   []string{"https://app.example.com"},
		AllowedMethods:   []string{"GET", "POST"},
		AllowedHeaders:   []string{"Content-Type", "Authorization"},
		AllowCredentials: true,
		MaxAge:           3600,
	})(inner)

	req := httptest.NewRequest(http.MethodOptions, "/api/data", nil)
	req.Header.Set("Origin", "https://app.example.com")
	req.Header.Set("Access-Control-Request-Method", "POST")
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	// Preflight should return 204
	if rec.Code != http.StatusNoContent {
		t.Errorf("status code = %d, want %d", rec.Code, http.StatusNoContent)
	}

	// Should echo the specific origin
	if got := rec.Header().Get("Access-Control-Allow-Origin"); got != "https://app.example.com" {
		t.Errorf("Access-Control-Allow-Origin = %q, want %q", got, "https://app.example.com")
	}

	// Should include credentials header
	if got := rec.Header().Get("Access-Control-Allow-Credentials"); got != "true" {
		t.Errorf("Access-Control-Allow-Credentials = %q, want %q", got, "true")
	}
}

func TestCORSMiddleware_DisallowedOrigin(t *testing.T) {
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	handler := CORS(CORSConfig{
		AllowedOrigins: []string{"https://app.example.com"},
	})(inner)

	req := httptest.NewRequest(http.MethodGet, "/api/data", nil)
	req.Header.Set("Origin", "https://evil.com")
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	// Should NOT have CORS headers
	if got := rec.Header().Get("Access-Control-Allow-Origin"); got != "" {
		t.Errorf("Access-Control-Allow-Origin should be empty for disallowed origin, got %q", got)
	}
}

func TestCORSMiddleware_NoOriginHeader(t *testing.T) {
	called := false
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		called = true
		w.WriteHeader(http.StatusOK)
	})

	handler := CORS(CORSConfig{
		AllowedOrigins: []string{"https://app.example.com"},
	})(inner)

	req := httptest.NewRequest(http.MethodGet, "/api/data", nil)
	// No Origin header -- not a CORS request
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if !called {
		t.Error("inner handler should be called for non-CORS requests")
	}
}
```

### Testing Rate Limiting

```go
package middleware_test

import (
	"context"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"
	"time"
)

func TestRateLimiting(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	handler := RateLimit(ctx, RateLimitConfig{
		RequestsPerSecond: 2,
		BurstSize:         3,
		CleanupInterval:   time.Minute,
		StaleAfter:        time.Minute,
		Logger:            logger,
	})(inner)

	// First 3 requests should succeed (burst size = 3)
	for i := 0; i < 3; i++ {
		req := httptest.NewRequest(http.MethodGet, "/test", nil)
		req.RemoteAddr = "192.168.1.1:12345"
		rec := httptest.NewRecorder()

		handler.ServeHTTP(rec, req)

		if rec.Code != http.StatusOK {
			t.Errorf("request %d: status = %d, want %d", i+1, rec.Code, http.StatusOK)
		}
	}

	// 4th request should be rate limited
	req := httptest.NewRequest(http.MethodGet, "/test", nil)
	req.RemoteAddr = "192.168.1.1:12345"
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusTooManyRequests {
		t.Errorf("4th request: status = %d, want %d", rec.Code, http.StatusTooManyRequests)
	}

	// Different IP should not be rate limited
	req = httptest.NewRequest(http.MethodGet, "/test", nil)
	req.RemoteAddr = "192.168.1.2:12345"
	rec = httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("different IP: status = %d, want %d", rec.Code, http.StatusOK)
	}
}
```

### Testing Auth Middleware

```go
package middleware_test

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"
	"time"
)

func TestAuthMiddleware_ValidToken(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
	secret := []byte("test-secret")

	token, err := SignJWT(Claims{
		UserID:    "user-1",
		Email:     "alice@example.com",
		Role:      "admin",
		IssuedAt:  time.Now().Unix(),
		ExpiresAt: time.Now().Add(time.Hour).Unix(),
	}, secret)
	if err != nil {
		t.Fatalf("failed to sign JWT: %v", err)
	}

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := GetClaims(r.Context())
		if claims == nil {
			t.Error("claims should be in context")
			return
		}
		if claims.UserID != "user-1" {
			t.Errorf("user ID = %q, want %q", claims.UserID, "user-1")
		}
		w.WriteHeader(http.StatusOK)
	})

	handler := Auth(AuthConfig{
		JWTSecret:  secret,
		CookieName: "auth_token",
		Logger:     logger,
	})(inner)

	// Test with Authorization header
	req := httptest.NewRequest(http.MethodGet, "/api/profile", nil)
	req.Header.Set("Authorization", "Bearer "+token)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
	}
}

func TestAuthMiddleware_ExpiredToken(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
	secret := []byte("test-secret")

	token, _ := SignJWT(Claims{
		UserID:    "user-1",
		ExpiresAt: time.Now().Add(-time.Hour).Unix(), // Expired 1 hour ago
	}, secret)

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t.Error("handler should not be called for expired token")
	})

	handler := Auth(AuthConfig{
		JWTSecret: secret,
		Logger:    logger,
	})(inner)

	req := httptest.NewRequest(http.MethodGet, "/api/profile", nil)
	req.Header.Set("Authorization", "Bearer "+token)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusUnauthorized {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusUnauthorized)
	}
}

func TestAuthMiddleware_NoToken(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t.Error("handler should not be called without token")
	})

	handler := Auth(AuthConfig{
		JWTSecret: []byte("secret"),
		Logger:    logger,
	})(inner)

	req := httptest.NewRequest(http.MethodGet, "/api/profile", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusUnauthorized {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusUnauthorized)
	}

	var body map[string]string
	json.Unmarshal(rec.Body.Bytes(), &body)
	if body["error"] != "authentication required" {
		t.Errorf("error = %q, want %q", body["error"], "authentication required")
	}
}

func TestAuthMiddleware_CookieToken(t *testing.T) {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
	secret := []byte("test-secret")

	token, _ := SignJWT(Claims{
		UserID:    "user-cookie",
		Email:     "cookie@example.com",
		ExpiresAt: time.Now().Add(time.Hour).Unix(),
	}, secret)

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := GetClaims(r.Context())
		if claims.UserID != "user-cookie" {
			t.Errorf("user ID = %q, want %q", claims.UserID, "user-cookie")
		}
		w.WriteHeader(http.StatusOK)
	})

	handler := Auth(AuthConfig{
		JWTSecret:  secret,
		CookieName: "auth_token",
		Logger:     logger,
	})(inner)

	req := httptest.NewRequest(http.MethodGet, "/api/profile", nil)
	req.AddCookie(&http.Cookie{
		Name:  "auth_token",
		Value: token,
	})
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)
	}
}
```

### Testing Tips

1. **Test each middleware in isolation.** Do not test the full middleware stack in unit tests -- that is an integration test.
2. **Use table-driven tests** for middleware with multiple code paths (valid/invalid/missing tokens, allowed/blocked origins).
3. **Test the inner handler's expectations.** Assert that the handler receives the correct context values, modified headers, etc.
4. **Test that short-circuiting works.** Verify that the inner handler is NOT called when middleware rejects the request.
5. **Use `httptest.NewRecorder()`** for response assertions and `httptest.NewRequest()` for request construction.

---

## 14. Real-World Example: Complete Middleware Stack

This section brings everything together into a single, self-contained, runnable program. This is the kind of middleware stack you would find in a production API.

```go
package main

import (
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"runtime/debug"
	"strconv"
	"strings"
	"sync"
	"syscall"
	"time"
)

// =============================================================================
// Context Keys
// =============================================================================

type ctxKeyType string

const (
	ctxKeyRequestID ctxKeyType = "request_id"
	ctxKeyClaims    ctxKeyType = "claims"
)

// =============================================================================
// Response Writer Wrapper
// =============================================================================

type wrappedResponseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func wrapResponseWriter(w http.ResponseWriter) *wrappedResponseWriter {
	return &wrappedResponseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (w *wrappedResponseWriter) WriteHeader(code int) {
	if !w.headerWritten {
		w.statusCode = code
		w.headerWritten = true
		w.ResponseWriter.WriteHeader(code)
	}
}

func (w *wrappedResponseWriter) Write(b []byte) (int, error) {
	if !w.headerWritten {
		w.headerWritten = true
		w.statusCode = http.StatusOK
	}
	n, err := w.ResponseWriter.Write(b)
	w.written += int64(n)
	return n, err
}

func (w *wrappedResponseWriter) Unwrap() http.ResponseWriter {
	return w.ResponseWriter
}

// =============================================================================
// 1. Recovery Middleware
// =============================================================================

func recoveryMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if err := recover(); err != nil {
					stack := debug.Stack()
					logger.Error("panic recovered",
						slog.String("error", fmt.Sprintf("%v", err)),
						slog.String("method", r.Method),
						slog.String("path", r.URL.Path),
						slog.String("stack", string(stack)),
					)
					w.Header().Set("Content-Type", "application/json")
					w.WriteHeader(http.StatusInternalServerError)
					json.NewEncoder(w).Encode(map[string]string{
						"error": "internal server error",
					})
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================================
// 2. Request ID Middleware
// =============================================================================

func generateID() string {
	b := make([]byte, 16)
	rand.Read(b)
	return hex.EncodeToString(b)
}

func isValidRequestID(id string) bool {
	if len(id) == 0 || len(id) > 128 {
		return false
	}
	for _, c := range id {
		if !((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') ||
			(c >= '0' && c <= '9') || c == '-' || c == '_') {
			return false
		}
	}
	return true
}

func requestIDMiddleware(trustIncoming bool) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			var id string
			if trustIncoming {
				if incoming := r.Header.Get("X-Request-ID"); incoming != "" && isValidRequestID(incoming) {
					id = incoming
				}
			}
			if id == "" {
				id = generateID()
			}
			w.Header().Set("X-Request-ID", id)
			r.Header.Set("X-Request-ID", id)
			ctx := context.WithValue(r.Context(), ctxKeyRequestID, id)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

func getRequestID(ctx context.Context) string {
	id, _ := ctx.Value(ctxKeyRequestID).(string)
	return id
}

// =============================================================================
// 3. Security Headers Middleware
// =============================================================================

func securityHeadersMiddleware() func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Header().Set("X-Content-Type-Options", "nosniff")
			w.Header().Set("X-Frame-Options", "DENY")
			w.Header().Set("Content-Security-Policy", "default-src 'self'")
			w.Header().Set("X-XSS-Protection", "0")
			w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
			w.Header().Set("Permissions-Policy", "camera=(), microphone=(), geolocation=()")
			if r.TLS != nil {
				w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
			}
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================================
// 4. CORS Middleware
// =============================================================================

type corsConfig struct {
	allowedOrigins   map[string]bool
	allowedMethods   string
	allowedHeaders   string
	exposedHeaders   string
	allowCredentials bool
	maxAge           string
}

func corsMiddleware(cfg corsConfig) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			origin := r.Header.Get("Origin")
			if origin == "" {
				next.ServeHTTP(w, r)
				return
			}
			if !cfg.allowedOrigins[origin] {
				if r.Method == http.MethodOptions {
					w.WriteHeader(http.StatusForbidden)
					return
				}
				next.ServeHTTP(w, r)
				return
			}
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Add("Vary", "Origin")
			if cfg.allowCredentials {
				w.Header().Set("Access-Control-Allow-Credentials", "true")
			}
			if cfg.exposedHeaders != "" {
				w.Header().Set("Access-Control-Expose-Headers", cfg.exposedHeaders)
			}
			if r.Method == http.MethodOptions {
				w.Header().Set("Access-Control-Allow-Methods", cfg.allowedMethods)
				w.Header().Set("Access-Control-Allow-Headers", cfg.allowedHeaders)
				if cfg.maxAge != "" {
					w.Header().Set("Access-Control-Max-Age", cfg.maxAge)
				}
				w.WriteHeader(http.StatusNoContent)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================================
// 5. Rate Limiting Middleware
// =============================================================================

type tokenBucket struct {
	tokens    float64
	lastCheck time.Time
	mu        sync.Mutex
}

func (b *tokenBucket) allow(rate float64, burst int) bool {
	b.mu.Lock()
	defer b.mu.Unlock()
	now := time.Now()
	b.tokens += now.Sub(b.lastCheck).Seconds() * rate
	b.lastCheck = now
	if b.tokens > float64(burst) {
		b.tokens = float64(burst)
	}
	if b.tokens < 1.0 {
		return false
	}
	b.tokens--
	return true
}

type ipRateLimiter struct {
	visitors map[string]*tokenBucket
	mu       sync.Mutex
	rate     float64
	burst    int
}

func newIPRateLimiter(rate float64, burst int) *ipRateLimiter {
	return &ipRateLimiter{
		visitors: make(map[string]*tokenBucket),
		rate:     rate,
		burst:    burst,
	}
}

func (rl *ipRateLimiter) get(ip string) *tokenBucket {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	b, ok := rl.visitors[ip]
	if !ok {
		b = &tokenBucket{tokens: float64(rl.burst), lastCheck: time.Now()}
		rl.visitors[ip] = b
	}
	return b
}

func (rl *ipRateLimiter) cleanup(staleAfter time.Duration) {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	cutoff := time.Now().Add(-staleAfter)
	for ip, b := range rl.visitors {
		b.mu.Lock()
		stale := b.lastCheck.Before(cutoff)
		b.mu.Unlock()
		if stale {
			delete(rl.visitors, ip)
		}
	}
}

func rateLimitMiddleware(ctx context.Context, rate float64, burst int, logger *slog.Logger) func(http.Handler) http.Handler {
	limiter := newIPRateLimiter(rate, burst)

	// Cleanup goroutine
	go func() {
		ticker := time.NewTicker(time.Minute)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				limiter.cleanup(5 * time.Minute)
			}
		}
	}()

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ip := clientIP(r)
			if !limiter.get(ip).allow(rate, burst) {
				logger.Warn("rate limit exceeded", slog.String("ip", ip), slog.String("path", r.URL.Path))
				w.Header().Set("Content-Type", "application/json")
				w.Header().Set("Retry-After", "1")
				w.WriteHeader(http.StatusTooManyRequests)
				json.NewEncoder(w).Encode(map[string]string{"error": "rate limit exceeded"})
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

func clientIP(r *http.Request) string {
	if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
		if idx := strings.IndexByte(xff, ','); idx != -1 {
			return strings.TrimSpace(xff[:idx])
		}
		return strings.TrimSpace(xff)
	}
	if xri := r.Header.Get("X-Real-IP"); xri != "" {
		return strings.TrimSpace(xri)
	}
	ip, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return ip
}

// =============================================================================
// 6. Body Size Limit Middleware
// =============================================================================

func bodyLimitMiddleware(maxBytes int64) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
				r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
			}
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================================
// 7. Request Logging Middleware
// =============================================================================

var sensitiveQueryParams = map[string]bool{
	"token": true, "api_key": true, "apikey": true,
	"password": true, "secret": true, "key": true, "access_token": true,
}

func redactQueryString(raw string) string {
	if raw == "" {
		return ""
	}
	vals, err := url.ParseQuery(raw)
	if err != nil {
		return "[invalid]"
	}
	for k := range vals {
		if sensitiveQueryParams[strings.ToLower(k)] {
			vals.Set(k, "[REDACTED]")
		}
	}
	return vals.Encode()
}

func loggingMiddleware(logger *slog.Logger, skipPaths map[string]bool) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if skipPaths[r.URL.Path] {
				next.ServeHTTP(w, r)
				return
			}

			start := time.Now()
			rw := wrapResponseWriter(w)
			next.ServeHTTP(rw, r)
			duration := time.Since(start)

			attrs := []slog.Attr{
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", rw.statusCode),
				slog.String("duration", duration.String()),
				slog.Int64("duration_ms", duration.Milliseconds()),
				slog.Int64("bytes", rw.written),
				slog.String("ip", clientIP(r)),
			}

			if reqID := getRequestID(r.Context()); reqID != "" {
				attrs = append(attrs, slog.String("request_id", reqID))
			}
			if r.URL.RawQuery != "" {
				attrs = append(attrs, slog.String("query", redactQueryString(r.URL.RawQuery)))
			}

			msg := fmt.Sprintf("%s %s %d", r.Method, r.URL.Path, rw.statusCode)
			switch {
			case rw.statusCode >= 500:
				logger.LogAttrs(r.Context(), slog.LevelError, msg, attrs...)
			case rw.statusCode >= 400:
				logger.LogAttrs(r.Context(), slog.LevelWarn, msg, attrs...)
			default:
				logger.LogAttrs(r.Context(), slog.LevelInfo, msg, attrs...)
			}
		})
	}
}

// =============================================================================
// 8. Authentication Middleware
// =============================================================================

type jwtClaims struct {
	UserID    string `json:"sub"`
	Email     string `json:"email"`
	Role      string `json:"role"`
	IssuedAt  int64  `json:"iat"`
	ExpiresAt int64  `json:"exp"`
}

var jwtHeaderB64 = base64.RawURLEncoding.EncodeToString([]byte(`{"alg":"HS256","typ":"JWT"}`))

func signJWT(claims jwtClaims, secret []byte) (string, error) {
	payload, err := json.Marshal(claims)
	if err != nil {
		return "", err
	}
	encoded := base64.RawURLEncoding.EncodeToString(payload)
	input := jwtHeaderB64 + "." + encoded
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(input))
	sig := base64.RawURLEncoding.EncodeToString(mac.Sum(nil))
	return input + "." + sig, nil
}

func verifyJWT(token string, secret []byte) (*jwtClaims, error) {
	parts := strings.Split(token, ".")
	if len(parts) != 3 {
		return nil, errors.New("invalid token format")
	}
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(parts[0] + "." + parts[1]))
	expected := base64.RawURLEncoding.EncodeToString(mac.Sum(nil))
	if !hmac.Equal([]byte(parts[2]), []byte(expected)) {
		return nil, errors.New("invalid signature")
	}
	payload, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		return nil, err
	}
	var claims jwtClaims
	if err := json.Unmarshal(payload, &claims); err != nil {
		return nil, err
	}
	if claims.ExpiresAt < time.Now().Unix() {
		return nil, errors.New("token expired")
	}
	return &claims, nil
}

func getClaims(ctx context.Context) *jwtClaims {
	c, _ := ctx.Value(ctxKeyClaims).(*jwtClaims)
	return c
}

func authMiddleware(jwtSecret []byte, cookieName string, logger *slog.Logger, skipPaths map[string]bool) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if skipPaths[r.URL.Path] {
				next.ServeHTTP(w, r)
				return
			}

			var token string
			if cookieName != "" {
				if c, err := r.Cookie(cookieName); err == nil && c.Value != "" {
					token = c.Value
				}
			}
			if token == "" {
				if auth := r.Header.Get("Authorization"); strings.HasPrefix(auth, "Bearer ") {
					token = auth[7:]
				}
			}
			if token == "" {
				writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "authentication required"})
				return
			}

			claims, err := verifyJWT(token, jwtSecret)
			if err != nil {
				logger.Warn("auth failed",
					slog.String("error", err.Error()),
					slog.String("path", r.URL.Path),
					slog.String("request_id", getRequestID(r.Context())),
				)
				writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid or expired token"})
				return
			}

			ctx := context.WithValue(r.Context(), ctxKeyClaims, claims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// =============================================================================
// Utilities
// =============================================================================

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func chain(handler http.Handler, mws ...func(http.Handler) http.Handler) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		handler = mws[i](handler)
	}
	return handler
}

// =============================================================================
// Route Handlers
// =============================================================================

func healthHandler(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func publicInfoHandler(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{
		"app":     "krafty-core",
		"version": "1.0.0",
	})
}

func profileHandler(w http.ResponseWriter, r *http.Request) {
	claims := getClaims(r.Context())
	if claims == nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "not authenticated"})
		return
	}
	writeJSON(w, http.StatusOK, map[string]interface{}{
		"user_id":    claims.UserID,
		"email":      claims.Email,
		"role":       claims.Role,
		"request_id": getRequestID(r.Context()),
	})
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
	claims := getClaims(r.Context())
	if claims == nil || claims.Role != "admin" {
		writeJSON(w, http.StatusForbidden, map[string]string{"error": "admin access required"})
		return
	}
	writeJSON(w, http.StatusOK, []map[string]string{
		{"id": "1", "name": "Alice", "email": "alice@example.com"},
		{"id": "2", "name": "Bob", "email": "bob@example.com"},
		{"id": "3", "name": "Charlie", "email": "charlie@example.com"},
	})
}

func panicHandler(w http.ResponseWriter, r *http.Request) {
	panic("intentional panic for testing recovery middleware")
}

// =============================================================================
// Main
// =============================================================================

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	jwtSecret := []byte("change-this-in-production-use-env-var")

	// Generate a test token for demonstration
	testToken, err := signJWT(jwtClaims{
		UserID:    "user-001",
		Email:     "alice@krafty.io",
		Role:      "admin",
		IssuedAt:  time.Now().Unix(),
		ExpiresAt: time.Now().Add(24 * time.Hour).Unix(),
	}, jwtSecret)
	if err != nil {
		logger.Error("failed to generate test token", slog.String("error", err.Error()))
		os.Exit(1)
	}

	// Create a cancellable context for goroutine lifecycle management
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Public paths that skip authentication
	publicPaths := map[string]bool{
		"/health":    true,
		"/api/info":  true,
		"/api/panic": true, // For testing; remove in production!
	}

	// Register routes
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", healthHandler)
	mux.HandleFunc("GET /api/info", publicInfoHandler)
	mux.HandleFunc("GET /api/profile", profileHandler)
	mux.HandleFunc("GET /api/admin/users", usersHandler)
	mux.HandleFunc("GET /api/panic", panicHandler)

	// Build the middleware stack in execution order
	handler := chain(mux,
		// 1. Recovery -- catches panics from everything below
		recoveryMiddleware(logger),

		// 2. Request ID -- assigns IDs for tracing
		requestIDMiddleware(false),

		// 3. Security Headers -- set on every response
		securityHeadersMiddleware(),

		// 4. CORS -- must be before auth for preflight requests
		corsMiddleware(corsConfig{
			allowedOrigins:   map[string]bool{"http://localhost:3000": true, "https://krafty.io": true},
			allowedMethods:   "GET, POST, PUT, PATCH, DELETE, OPTIONS",
			allowedHeaders:   "Content-Type, Authorization, X-Request-ID",
			exposedHeaders:   "X-Request-ID, X-RateLimit-Remaining",
			allowCredentials: true,
			maxAge:           "86400",
		}),

		// 5. Rate Limiting -- before auth to prevent brute force
		rateLimitMiddleware(ctx, 10, 20, logger),

		// 6. Body Size Limit -- before parsing
		bodyLimitMiddleware(1<<20), // 1 MB

		// 7. Request Logging -- logs all requests including rejected ones
		loggingMiddleware(logger, map[string]bool{"/health": true}),

		// 8. Authentication -- last in chain, only for protected routes
		authMiddleware(jwtSecret, "auth_token", logger, publicPaths),
	)

	server := &http.Server{
		Addr:         ":8080",
		Handler:      handler,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	// Start server in a goroutine
	go func() {
		logger.Info("server starting",
			slog.String("addr", server.Addr),
		)
		logger.Info("test endpoints",
			slog.String("health", "curl http://localhost:8080/health"),
			slog.String("public", "curl http://localhost:8080/api/info"),
			slog.String("auth", fmt.Sprintf("curl -H 'Authorization: Bearer %s' http://localhost:8080/api/profile", testToken)),
			slog.String("admin", fmt.Sprintf("curl -H 'Authorization: Bearer %s' http://localhost:8080/api/admin/users", testToken)),
			slog.String("panic", "curl http://localhost:8080/api/panic"),
		)

		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("server error", slog.String("error", err.Error()))
			os.Exit(1)
		}
	}()

	// Wait for interrupt signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("shutting down server...")
	cancel() // Stop the rate limiter cleanup goroutine

	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer shutdownCancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		logger.Error("shutdown error", slog.String("error", err.Error()))
	}
	logger.Info("server stopped")
}
```

### Testing the Complete Stack

Save the file and run it:

```bash
go run main.go
```

Test each layer:

```bash
# Health check (public, skips auth and logging)
curl http://localhost:8080/health
# {"status":"ok"}

# Public endpoint (no auth required)
curl -v http://localhost:8080/api/info
# Check response headers for security headers, request ID, etc.

# Authenticated endpoint (requires token)
curl http://localhost:8080/api/profile
# {"error":"authentication required"}

curl -H "Authorization: Bearer <token>" http://localhost:8080/api/profile
# {"user_id":"user-001","email":"alice@krafty.io","role":"admin","request_id":"..."}

# Panic recovery
curl http://localhost:8080/api/panic
# {"error":"internal server error"}
# Server continues running -- check logs for stack trace

# CORS preflight
curl -X OPTIONS -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: POST" \
     http://localhost:8080/api/profile
# 204 No Content with CORS headers

# Rate limiting (hit it rapidly)
for i in $(seq 1 25); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/api/info; done
# First 20 return 200, then 429
```

---

## 15. Key Takeaways

1. **The fundamental pattern is `func(http.Handler) http.Handler`.** Every middleware in Go follows this signature. It takes a handler, wraps it, and returns a new handler. Master this pattern and you can write any middleware.

2. **Middleware factories take configuration.** Real middleware needs configuration (logger, secrets, allowed origins). The factory pattern -- a function that takes config and returns `func(http.Handler) http.Handler` -- is how you parameterize middleware.

3. **Order matters critically.** Recovery must be outermost. CORS must precede auth. Rate limiting must precede auth. Request ID must precede logging. Get the order wrong and you get security holes, missing logs, or broken browser requests.

4. **Wrap ResponseWriter to capture status codes.** Go's `http.ResponseWriter` does not expose the status code after it is written. Wrapping it is the only way to log response status. Always implement `Unwrap()` to preserve standard library features.

5. **Use `defer` with `recover()` for panic recovery.** This is the Go equivalent of try/catch for panics. It runs even if the handler panics, ensuring the server stays up and a proper error response is sent.

6. **Redact sensitive query parameters in logs.** Tokens, API keys, and passwords in query strings must never appear in log output. Build a redaction list and apply it in your logging middleware.

7. **Validate incoming request IDs.** Never trust client-provided data. Request IDs can be used for log injection if not validated. Only allow alphanumeric characters, hyphens, and underscores.

8. **Tie goroutine lifecycles to context.** The rate limiter's cleanup goroutine uses `context.Context` to know when to stop. Without this, you leak goroutines when the server shuts down.

9. **Use `http.MaxBytesReader` for body size limits.** It integrates with the HTTP server to close connections from abusive clients. Do not roll your own body size limiting.

10. **Test middleware in isolation with `httptest`.** Each middleware should be testable independently. Use `httptest.NewRecorder()` and `httptest.NewRequest()` to simulate requests without a running server.

11. **CORS is a browser security feature, not a server security feature.** It only matters for browser-based JavaScript clients. Never use `*` with credentials. Always set the `Vary: Origin` header. Handle preflight (OPTIONS) before routing.

12. **Check both cookies and headers for auth tokens.** Web clients use httpOnly cookies (more secure against XSS). Mobile and CLI clients use Authorization headers. Good auth middleware supports both.

---

## 16. Practice Exercises

### Exercise 1: Timeout Middleware

Write middleware that cancels the request context after a configurable duration. If the handler does not complete within the timeout, the middleware should return a 504 Gateway Timeout response.

Hints:
- Use `context.WithTimeout` to create a deadline
- Use a `select` statement with a channel to detect timeout
- Be careful about writing to the response after timeout -- the handler might still be running

```go
// Skeleton
func TimeoutMiddleware(duration time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Your implementation here
        })
    }
}
```

### Exercise 2: Request Deduplication Middleware

Write middleware that prevents duplicate requests. If the same request ID is seen within a 5-second window, return the cached response instead of processing the request again. This is useful for idempotent POST requests.

Requirements:
- Use a `sync.Map` or `map` with `sync.Mutex` to store recent request IDs
- Include a cleanup mechanism to prevent unbounded growth
- Only apply to POST/PUT/PATCH methods

### Exercise 3: Maintenance Mode Middleware

Write middleware that returns a 503 Service Unavailable response when a maintenance flag is set. The flag should be controlled by an atomic variable that can be toggled at runtime (e.g., via a signal or an admin endpoint).

Requirements:
- Use `sync/atomic` for the maintenance flag
- Include a `Retry-After` header
- Return a JSON response with a human-readable message

### Exercise 4: API Versioning Middleware

Write middleware that extracts the API version from the URL path (e.g., `/v1/users`, `/v2/users`) or from a custom header (`X-API-Version`), and injects it into the request context.

Requirements:
- Support both URL path and header-based versioning
- Default to the latest version if none is specified
- Reject unsupported versions with a 400 Bad Request

### Exercise 5: Full Middleware Stack Test

Write a test that starts a complete middleware stack (recovery, request ID, logging, auth) and verifies:
- Public routes return 200 without auth
- Protected routes return 401 without auth
- Protected routes return 200 with a valid token
- Panic routes return 500 (not crash)
- All responses include `X-Request-ID`
- All responses include security headers

Use `httptest.NewServer` to start a real HTTP server for integration testing:

```go
func TestFullStack(t *testing.T) {
    // Build the middleware stack
    // Create an httptest.NewServer
    // Make real HTTP requests
    // Assert status codes, headers, and response bodies
}
```

### Exercise 6: Middleware Metrics

Write middleware that collects Prometheus-style metrics:
- Request count by method, path, and status code
- Request duration histogram
- In-flight request gauge

Use `sync/atomic` and maps to collect metrics, and expose them on a `/metrics` endpoint in a Prometheus-compatible format.

### Exercise 7: Graceful Degradation Middleware

Write middleware that implements circuit-breaker behavior. If an upstream dependency (simulated by a function call) fails more than N times in a row, the middleware starts returning cached responses or a degraded response for a cooldown period before trying the upstream again.

This is an advanced exercise that combines rate limiting concepts with error tracking and state machines.
