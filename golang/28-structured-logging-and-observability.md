# Chapter 28: Structured Logging & Observability

Production systems do not fail gracefully. They fail at 3 AM, under load, in ways you did not anticipate. When they do, the only thing standing between you and a multi-hour outage is the quality of your logs. If your logs are unstructured `fmt.Println` calls scattered across the codebase, you will spend hours grepping through millions of lines hoping to find the one that matters. If your logs are structured, machine-parseable, and enriched with request context, you will find the problem in minutes.

This chapter covers production-grade structured logging in Go using the standard library's `log/slog` package, introduced in Go 1.21. Every pattern shown here is drawn from real production systems -- the kind of code you find in projects like "krafty-core" where a Go API serves thousands of requests per second and every log line must be useful, safe, and searchable.

This is not academic. We will build a complete request logging middleware that tracks request IDs, measures durations, redacts sensitive data, prevents log injection attacks, and outputs structured JSON that tools like Elasticsearch, Datadog, and Grafana Loki can ingest directly.

---

## Prerequisites

Before diving into this chapter, you should be comfortable with:

- **Chapter 6: Structs and Interfaces** -- We use interfaces heavily (`slog.Handler`, `http.ResponseWriter`)
- **Chapter 8: Error Handling** -- Logging and error handling are tightly coupled
- **Chapter 12: JSON, HTTP, and REST APIs** -- All examples use HTTP handlers and middleware
- **Chapter 16: Context Package** -- Request-scoped values (request IDs, loggers) propagate via `context.Context`
- **Chapter 21: Building Microservices** -- This chapter extends the observability section from Chapter 21

You should also have Go 1.21 or later installed, since `log/slog` was added in that version.

---

## Table of Contents

1. [Why Structured Logging?](#1-why-structured-logging)
2. [Go's log/slog Package](#2-gos-logslog-package)
3. [Log Levels](#3-log-levels)
4. [Text vs JSON Handlers](#4-text-vs-json-handlers)
5. [Structured Attributes](#5-structured-attributes)
6. [Request ID Tracking](#6-request-id-tracking)
7. [Log Injection Prevention](#7-log-injection-prevention)
8. [Response Writer Wrapping](#8-response-writer-wrapping)
9. [Query Parameter Redaction](#9-query-parameter-redaction)
10. [Log Level per Status Code](#10-log-level-per-status-code)
11. [Logger Dependency Injection](#11-logger-dependency-injection)
12. [Log Grouping and Child Loggers](#12-log-grouping-and-child-loggers)
13. [Health Check Logging](#13-health-check-logging)
14. [Metrics and Tracing Basics](#14-metrics-and-tracing-basics)
15. [Real-World Example: Complete Logging Stack](#15-real-world-example-complete-logging-stack)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. Why Structured Logging?

### The Problem with Unstructured Logs

Every developer starts here:

```go
package main

import "fmt"

func main() {
	fmt.Println("server starting on port 8080")
	fmt.Println("received request from 192.168.1.100")
	fmt.Printf("request took %dms\n", 42)
	fmt.Println("ERROR: database connection failed")
}
```

Output:
```
server starting on port 8080
received request from 192.168.1.100
request took 42ms
ERROR: database connection failed
```

This looks fine when you are reading it in your terminal during development. Now imagine you have 50 services, each producing 10,000 log lines per minute, and you need to answer: "Which requests to the `/api/orders` endpoint are taking longer than 500ms?"

You cannot answer that question. The logs are human-readable strings with no structure. You cannot filter by endpoint. You cannot filter by duration. You cannot correlate a request across multiple services. You would need to write a regex that maybe works for your particular log format and hope nobody changed the format string.

### What Structured Logging Looks Like

```json
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"request completed","method":"GET","path":"/api/orders","status":200,"duration_ms":42,"request_id":"abc-123","bytes":1542}
{"time":"2025-03-15T14:32:01.456Z","level":"ERROR","msg":"request completed","method":"POST","path":"/api/orders","status":500,"duration_ms":2341,"request_id":"def-456","bytes":89,"error":"database connection refused"}
```

Now you can:
- **Filter**: `jq 'select(.path == "/api/orders" and .duration_ms > 500)'`
- **Aggregate**: Count errors per endpoint per minute
- **Correlate**: Find all logs for request `abc-123` across all services
- **Alert**: Trigger PagerDuty when error rate exceeds 1%
- **Visualize**: Build Grafana dashboards showing P50/P95/P99 latency per endpoint

Every log aggregation system -- Elasticsearch, Datadog, Splunk, Grafana Loki, CloudWatch -- ingests structured JSON natively. Unstructured logs require parsing rules that break whenever someone changes a format string.

### The Go Standard Library Evolution

Go's logging story evolved in three phases:

```
Phase 1: log package (Go 1.0, 2012)
  - Basic: log.Println("something happened")
  - No levels, no structure, no context

Phase 2: Third-party libraries (2014-2023)
  - zerolog, zap, logrus
  - Fast, structured, but incompatible APIs

Phase 3: log/slog (Go 1.21, 2023)
  - Standard library structured logging
  - Handler interface for extensibility
  - Adopted as the standard across the ecosystem
```

Before `slog`, choosing a logging library was a religious war. Uber's `zap` was the fastest. `zerolog` had zero allocations. `logrus` had the nicest API. Each had different interfaces, so changing your logging library meant rewriting every log call in your codebase.

`slog` ended this war. It provides a standard interface that the ecosystem can agree on, with a `Handler` abstraction that lets you plug in any backend. You can use `slog` as your API and route output to `zap`, `zerolog`, or any custom handler.

### Node.js/Express Comparison

In the Node.js world, the same evolution happened:

```
Phase 1: console.log("something happened")
Phase 2: winston, bunyan, pino (third-party)
Phase 3: pino became the de facto standard
```

The closest Node.js equivalent to Go's `slog` is **pino**:

```javascript
// Node.js with pino
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({ method: 'GET', path: '/api/orders', status: 200, duration: 42 }, 'request completed');
// Output: {"level":30,"time":1710512421123,"msg":"request completed","method":"GET","path":"/api/orders","status":200,"duration":42}
```

```go
// Go with slog -- nearly identical semantics
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	logger.Info("request completed",
		slog.String("method", "GET"),
		slog.String("path", "/api/orders"),
		slog.Int("status", 200),
		slog.Int("duration", 42),
	)
	// Output: {"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"request completed","method":"GET","path":"/api/orders","status":200,"duration":42}
}
```

The key difference: Go's `slog` is in the standard library. Node.js requires installing `pino` from npm. Go's approach means every Go library can emit structured logs through the same interface without adding dependencies.

---

## 2. Go's log/slog Package

### Architecture Overview

The `slog` package has a clean, layered architecture:

```
┌─────────────────────────────────────────┐
│            Your Application             │
│   logger.Info("msg", "key", "value")    │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│            *slog.Logger                 │
│   - Convenience methods (Info, Error)   │
│   - Creates slog.Record                 │
│   - Passes Record to Handler            │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│          slog.Handler (interface)        │
│   - Enabled(level) bool                 │
│   - Handle(ctx, Record) error           │
│   - WithAttrs([]Attr) Handler           │
│   - WithGroup(name) Handler             │
└───────────────┬─────────────────────────┘
                │
     ┌──────────┴──────────┐
     │                     │
┌────▼─────┐        ┌─────▼────┐
│TextHandler│        │JSONHandler│
│ (dev)     │        │ (prod)    │
└──────────┘        └──────────┘
```

**Logger** is the thing you call methods on. **Handler** is the thing that decides where and how to write the output. This separation means you can swap handlers without changing any application code.

### slog.Record

Every log call creates a `slog.Record`. This is the internal representation of a log event:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"time"
)

func main() {
	// You rarely create Records manually, but understanding them helps.
	// A Record contains:
	//   - Time:    when the log event occurred
	//   - Level:   DEBUG, INFO, WARN, ERROR
	//   - Message: the human-readable message
	//   - Attrs:   key-value pairs of structured data

	record := slog.NewRecord(time.Now(), slog.LevelInfo, "user logged in", 0)
	record.AddAttrs(
		slog.String("user_id", "usr_abc123"),
		slog.String("ip", "192.168.1.100"),
	)

	// Iterate over attributes
	record.Attrs(func(a slog.Attr) bool {
		fmt.Printf("  %s = %v\n", a.Key, a.Value)
		return true // continue iteration
	})

	// In practice, the Logger creates Records for you:
	logger := slog.New(slog.NewTextHandler(nil, nil))
	logger.InfoContext(context.Background(), "user logged in",
		slog.String("user_id", "usr_abc123"),
		slog.String("ip", "192.168.1.100"),
	)
}
```

### The Handler Interface

The `Handler` interface is where the real power lives:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"time"
)

// The slog.Handler interface:
//
//   type Handler interface {
//       Enabled(context.Context, Level) bool
//       Handle(context.Context, Record) error
//       WithAttrs(attrs []Attr) Handler
//       WithGroup(name string) Handler
//   }

// SimpleHandler is a minimal custom handler to show the interface.
type SimpleHandler struct {
	level slog.Level
}

func (h *SimpleHandler) Enabled(_ context.Context, level slog.Level) bool {
	return level >= h.level
}

func (h *SimpleHandler) Handle(_ context.Context, r slog.Record) error {
	// Custom formatting: LEVEL [TIME] message key=value key=value
	timeStr := r.Time.Format("15:04:05.000")
	fmt.Printf("%-5s [%s] %s", r.Level, timeStr, r.Message)

	r.Attrs(func(a slog.Attr) bool {
		fmt.Printf(" %s=%v", a.Key, a.Value)
		return true
	})
	fmt.Println()
	return nil
}

func (h *SimpleHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	// In a real handler, you would store these attrs and include them
	// in every subsequent Handle call. We keep it simple here.
	return h
}

func (h *SimpleHandler) WithGroup(name string) slog.Handler {
	// In a real handler, you would prefix subsequent keys with this group name.
	return h
}

func main() {
	handler := &SimpleHandler{level: slog.LevelInfo}
	logger := slog.New(handler)

	logger.Debug("this will not appear") // below INFO level
	logger.Info("server started", slog.Int("port", 8080))
	logger.Warn("slow query", slog.Duration("duration", 2*time.Second))
	logger.Error("connection failed", slog.String("host", "db.example.com"))
}
```

Output:
```
INFO  [14:32:01.123] server started port=8080
WARN  [14:32:01.123] slow query duration=2s
ERROR [14:32:01.123] connection failed host=db.example.com
```

### The Default Logger

`slog` provides a package-level default logger, similar to the `log` package:

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// The default logger uses TextHandler writing to stderr.
	slog.Info("using default logger", slog.String("key", "value"))

	// You can replace the default logger:
	jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(jsonLogger)

	// Now package-level calls use the JSON handler:
	slog.Info("now using JSON", slog.String("key", "value"))

	// IMPORTANT: SetDefault also updates the old log package.
	// This means libraries using log.Println will now output structured JSON.
	// This is a crucial bridge for migrating existing codebases.
}
```

**Production recommendation:** Do not rely on the default logger. Create an explicit logger in `main()` and pass it through your application via dependency injection or context. This makes testing easier and eliminates global state. We cover this pattern in Section 11.

---

## 3. Log Levels

### The Four Standard Levels

`slog` defines four levels as integer constants:

```go
package main

import (
	"fmt"
	"log/slog"
)

func main() {
	// slog levels are integers, which allows custom levels between them.
	fmt.Println("DEBUG:", slog.LevelDebug) // -4
	fmt.Println("INFO: ", slog.LevelInfo)  // 0
	fmt.Println("WARN: ", slog.LevelWarn)  // 4
	fmt.Println("ERROR:", slog.LevelError) // 8

	// You can define custom levels:
	const LevelTrace = slog.Level(-8)      // More verbose than DEBUG
	const LevelFatal = slog.Level(12)      // More severe than ERROR
	const LevelNotice = slog.Level(2)      // Between INFO and WARN

	fmt.Println("TRACE:", LevelTrace)
	fmt.Println("FATAL:", LevelFatal)
	fmt.Println("NOTICE:", LevelNotice)
}
```

### When to Use Each Level

This is one of the most important decisions in production logging. Get it wrong and you either drown in noise or miss critical signals.

```
┌──────────┬────────────────────────────────────────────────────────────┐
│ Level    │ When to use                                               │
├──────────┼────────────────────────────────────────────────────────────┤
│ DEBUG    │ Detailed diagnostic info. SQL queries with parameters,    │
│          │ cache hit/miss, full request/response bodies.             │
│          │ OFF in production by default.                             │
├──────────┼────────────────────────────────────────────────────────────┤
│ INFO     │ Normal operations. Request completed, server started,     │
│          │ user logged in, order created. The "heartbeat" of your    │
│          │ application. Should be low enough volume to keep on 24/7. │
├──────────┼────────────────────────────────────────────────────────────┤
│ WARN     │ Something unexpected that is NOT an error. Client sent    │
│          │ invalid input (4xx), deprecated endpoint called, retry    │
│          │ succeeded after first failure, approaching rate limit.    │
│          │ Warrants investigation but not a page.                    │
├──────────┼────────────────────────────────────────────────────────────┤
│ ERROR    │ Something BROKE. Database down, external API unreachable, │
│          │ unhandled panic, 5xx responses. Should trigger alerts.    │
│          │ Every ERROR log should be actionable.                     │
└──────────┴────────────────────────────────────────────────────────────┘
```

**The golden rule:** If you see an ERROR log, someone should investigate. If you see a WARN log, you should check if it becomes a pattern. If ERROR logs happen routinely and nobody investigates, your level assignments are wrong.

### Configurable Log Levels

In production, you want to change the log level without restarting the service. The `slog.LevelVar` type provides an atomic, concurrency-safe level that can be changed at runtime:

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strings"
)

func main() {
	// LevelVar is safe for concurrent use -- you can change it at runtime.
	var programLevel slog.LevelVar

	// Parse level from config/environment
	cfgLevel := os.Getenv("LOG_LEVEL")
	if cfgLevel == "" {
		cfgLevel = "info" // sensible default
	}

	switch strings.ToLower(cfgLevel) {
	case "debug":
		programLevel.Set(slog.LevelDebug)
	case "warn":
		programLevel.Set(slog.LevelWarn)
	case "error":
		programLevel.Set(slog.LevelError)
	default:
		programLevel.Set(slog.LevelInfo)
	}

	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: &programLevel, // Note: pointer to LevelVar
	}))
	slog.SetDefault(logger)

	// Admin endpoint to change level at runtime without restart.
	// This is incredibly useful in production: enable DEBUG for 5 minutes
	// to diagnose an issue, then switch back to INFO.
	http.HandleFunc("/admin/log-level", func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodGet {
			fmt.Fprintf(w, "current level: %s\n", programLevel.Level())
			return
		}

		if r.Method == http.MethodPut {
			newLevel := r.URL.Query().Get("level")
			switch strings.ToLower(newLevel) {
			case "debug":
				programLevel.Set(slog.LevelDebug)
			case "info":
				programLevel.Set(slog.LevelInfo)
			case "warn":
				programLevel.Set(slog.LevelWarn)
			case "error":
				programLevel.Set(slog.LevelError)
			default:
				http.Error(w, "invalid level: use debug, info, warn, or error", http.StatusBadRequest)
				return
			}
			logger.Info("log level changed", slog.String("new_level", newLevel))
			fmt.Fprintf(w, "level set to: %s\n", newLevel)
			return
		}

		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
	})

	logger.Info("server starting",
		slog.String("log_level", programLevel.Level().String()),
		slog.Int("port", 8080),
	)

	// Test: these may or may not appear depending on the configured level
	logger.Debug("debug message -- only visible at DEBUG level")
	logger.Info("info message -- visible at INFO and DEBUG")
	logger.Warn("warn message -- visible at WARN, INFO, and DEBUG")
	logger.Error("error message -- always visible")

	http.ListenAndServe(":8080", nil)
}
```

Usage in production:
```bash
# Check current level
curl http://localhost:8080/admin/log-level

# Enable debug logging temporarily
curl -X PUT "http://localhost:8080/admin/log-level?level=debug"

# Back to normal after investigation
curl -X PUT "http://localhost:8080/admin/log-level?level=info"
```

### The krafty-core Pattern

Here is the exact pattern used in production Go APIs for parsing log levels from configuration:

```go
package main

import (
	"log/slog"
	"os"
)

// Config represents the application configuration.
type Config struct {
	LogLevel string
	Port     int
}

func setupLogger(cfg Config) *slog.Logger {
	var level slog.Level
	switch cfg.LogLevel {
	case "debug":
		level = slog.LevelDebug
	case "warn":
		level = slog.LevelWarn
	case "error":
		level = slog.LevelError
	default:
		level = slog.LevelInfo
	}

	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: level,
	}))

	return logger
}

func main() {
	cfg := Config{
		LogLevel: "debug", // Would come from env/config file in production
		Port:     8080,
	}

	logger := setupLogger(cfg)
	logger.Info("logger configured",
		slog.String("level", cfg.LogLevel),
		slog.Int("port", cfg.Port),
	)

	logger.Debug("this debug message is visible because level is debug")
}
```

**Note the subtle difference:** In the first example, we used `&slog.LevelVar` (a pointer to a mutable level). In this simpler example, we used `slog.Level` directly (an immutable level). Use `LevelVar` when you need runtime level changes. Use `Level` when the level is fixed at startup.

### Node.js Comparison

```javascript
// pino log levels
const pino = require('pino');

// pino levels: trace=10, debug=20, info=30, warn=40, error=50, fatal=60
const logger = pino({
  level: process.env.LOG_LEVEL || 'info'
});

logger.debug('detailed info');  // hidden at info level
logger.info('normal operation');
logger.warn('unexpected');
logger.error('something broke');

// Winston levels (npm's most popular logger)
// error=0, warn=1, info=2, http=3, verbose=4, debug=5, silly=6
// Note: lower number = higher severity (opposite of slog!)
```

The key difference: Go's `slog` uses ascending integers (higher = more severe), while Winston uses descending (lower = more severe). Pino uses ascending like slog. This is a common source of confusion when switching between ecosystems.

---

## 4. Text vs JSON Handlers

### TextHandler -- Development

`TextHandler` produces human-readable key=value output. Use it during local development where you are reading logs in a terminal.

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	logger.Debug("connecting to database",
		slog.String("host", "localhost"),
		slog.Int("port", 5432),
	)

	logger.Info("server started",
		slog.String("addr", ":8080"),
		slog.Duration("startup_time", 42*time.Millisecond),
	)

	logger.Warn("deprecated endpoint called",
		slog.String("path", "/api/v1/users"),
		slog.String("replacement", "/api/v2/users"),
	)

	logger.Error("query failed",
		slog.String("query", "SELECT * FROM users WHERE id = $1"),
		slog.String("error", "connection refused"),
		slog.Duration("duration", 5*time.Second),
	)
}
```

Output:
```
time=2025-03-15T14:32:01.123Z level=DEBUG msg="connecting to database" host=localhost port=5432
time=2025-03-15T14:32:01.123Z level=INFO msg="server started" addr=:8080 startup_time=42ms
time=2025-03-15T14:32:01.123Z level=WARN msg="deprecated endpoint called" path=/api/v1/users replacement=/api/v2/users
time=2025-03-15T14:32:01.123Z level=ERROR msg="query failed" query="SELECT * FROM users WHERE id = $1" error="connection refused" duration=5s
```

### JSONHandler -- Production

`JSONHandler` produces machine-parseable JSON. Use it in production where logs are consumed by aggregation systems.

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	logger.Debug("connecting to database",
		slog.String("host", "localhost"),
		slog.Int("port", 5432),
	)

	logger.Info("server started",
		slog.String("addr", ":8080"),
		slog.Duration("startup_time", 42*time.Millisecond),
	)

	logger.Warn("deprecated endpoint called",
		slog.String("path", "/api/v1/users"),
		slog.String("replacement", "/api/v2/users"),
	)

	logger.Error("query failed",
		slog.String("query", "SELECT * FROM users WHERE id = $1"),
		slog.String("error", "connection refused"),
		slog.Duration("duration", 5*time.Second),
	)
}
```

Output (each line is valid JSON):
```json
{"time":"2025-03-15T14:32:01.123Z","level":"DEBUG","msg":"connecting to database","host":"localhost","port":5432}
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"server started","addr":":8080","startup_time":42000000}
{"time":"2025-03-15T14:32:01.123Z","level":"WARN","msg":"deprecated endpoint called","path":"/api/v1/users","replacement":"/api/v2/users"}
{"time":"2025-03-15T14:32:01.123Z","level":"ERROR","msg":"query failed","query":"SELECT * FROM users WHERE id = $1","error":"connection refused","duration":5000000000}
```

**Notice:** Durations are serialized as nanoseconds in JSON (integers), not as human-readable strings. This is intentional -- it makes aggregation and comparison trivial. Your log visualization tool will format them for human display.

### Switching Based on Environment

The standard production pattern: use TextHandler in development, JSONHandler in production.

```go
package main

import (
	"log/slog"
	"os"
)

func newLogger(env string, level slog.Level) *slog.Logger {
	opts := &slog.HandlerOptions{
		Level: level,
	}

	var handler slog.Handler
	switch env {
	case "production", "staging":
		handler = slog.NewJSONHandler(os.Stdout, opts)
	default:
		// Development: human-readable output
		// Also add source file information for debugging
		opts.AddSource = true
		handler = slog.NewTextHandler(os.Stdout, opts)
	}

	return slog.New(handler)
}

func main() {
	env := os.Getenv("APP_ENV")
	if env == "" {
		env = "development"
	}

	logger := newLogger(env, slog.LevelInfo)
	logger.Info("application starting",
		slog.String("environment", env),
	)
}
```

Development output (with source):
```
time=2025-03-15T14:32:01.123Z level=INFO source=/path/to/main.go:35 msg="application starting" environment=development
```

Production output:
```json
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"application starting","environment":"production"}
```

### AddSource Option

When debugging issues, knowing which file and line produced a log message is invaluable:

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		AddSource: true,
		Level:     slog.LevelDebug,
	}))

	logger.Info("request handled", slog.Int("status", 200))
}
```

Output:
```json
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","source":{"function":"main.main","file":"/path/to/main.go","line":13},"msg":"request handled","status":200}
```

**Production tip:** Enable `AddSource` in development and staging. Disable it in production to save a small amount of CPU (runtime caller lookup is not free) and reduce log volume. Enable it temporarily in production when debugging a specific issue.

### ReplaceAttr -- Customizing Output

Both handlers accept a `ReplaceAttr` function that lets you transform attributes before they are written:

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
			// Rename "time" to "timestamp" (common requirement for ELK stack)
			if a.Key == slog.TimeKey {
				a.Key = "timestamp"
			}

			// Rename "msg" to "message" (Datadog convention)
			if a.Key == slog.MessageKey {
				a.Key = "message"
			}

			// Rename "level" to "severity" (GCP convention)
			if a.Key == slog.LevelKey {
				a.Key = "severity"
			}

			// Format time as Unix epoch milliseconds (some systems prefer this)
			if a.Key == "timestamp" {
				if t, ok := a.Value.Any().(time.Time); ok {
					a.Value = slog.Int64Value(t.UnixMilli())
				}
			}

			return a
		},
	}))

	logger.Info("request completed", slog.Int("status", 200))
}
```

Output:
```json
{"timestamp":1710512421123,"severity":"INFO","message":"request completed","status":200}
```

This is powerful for adapting your logs to whatever format your log aggregation system expects, without changing any application code.

---

## 5. Structured Attributes

### Attribute Types

`slog` provides typed constructors for common Go types. These are not just convenience -- they avoid allocations that would occur with `slog.Any`:

```go
package main

import (
	"log/slog"
	"net"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// String attributes
	logger.Info("user action",
		slog.String("user_id", "usr_abc123"),
		slog.String("action", "login"),
		slog.String("ip", "192.168.1.100"),
	)

	// Numeric attributes
	logger.Info("request metrics",
		slog.Int("status", 200),
		slog.Int64("bytes_written", 15423),
		slog.Float64("cpu_usage", 0.42),
		slog.Uint64("request_number", 1000000),
	)

	// Time and duration
	logger.Info("timing",
		slog.Time("started_at", time.Now()),
		slog.Duration("elapsed", 142*time.Millisecond),
		slog.Duration("timeout", 30*time.Second),
	)

	// Boolean
	logger.Info("feature flags",
		slog.Bool("cache_enabled", true),
		slog.Bool("maintenance_mode", false),
	)

	// Any -- for types without dedicated constructors
	logger.Info("network",
		slog.Any("remote_addr", net.IPv4(192, 168, 1, 100)),
		slog.Any("headers", map[string]string{
			"Content-Type": "application/json",
			"Accept":       "application/json",
		}),
	)
}
```

### LogAttrs -- The High-Performance Path

For performance-critical code paths (like request logging middleware that runs on every single request), use `LogAttrs` instead of `Info`/`Error`. It avoids the interface conversion overhead:

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

	// Simulating request logging
	method := "GET"
	path := "/api/orders"
	statusCode := 200
	duration := 42 * time.Millisecond
	bytesWritten := int64(1542)

	ctx := context.Background()

	// SLOWER: Info with alternating key-value pairs
	// Each value is boxed into an interface{}, causing allocations.
	logger.InfoContext(ctx, "request completed",
		"method", method,
		"path", path,
		"status", statusCode,
		"duration", duration,
		"bytes", bytesWritten,
	)

	// FASTER: LogAttrs with typed Attr values
	// No interface boxing -- values stay as their concrete types.
	logger.LogAttrs(ctx, slog.LevelInfo, "request completed",
		slog.String("method", method),
		slog.String("path", path),
		slog.Int("status", statusCode),
		slog.Duration("duration", duration),
		slog.Int64("bytes", bytesWritten),
	)
}
```

**Why LogAttrs is faster:** When you call `logger.Info("msg", "key", value)`, Go must convert `value` to an `any` (interface), which causes a heap allocation for value types like `int`. `LogAttrs` takes `slog.Attr` values directly, which embed the value in a `slog.Value` struct that avoids allocations for common types.

The performance difference is small per call (~100ns), but in a middleware that runs on every request at 10,000 requests/second, it adds up. Use `LogAttrs` in hot paths. Use the simpler `Info`/`Error` methods everywhere else.

### Grouped Attributes

You can nest attributes into groups for better organization:

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Group creates a nested JSON object
	logger.Info("request completed",
		slog.Group("request",
			slog.String("method", "POST"),
			slog.String("path", "/api/orders"),
			slog.String("remote_addr", "192.168.1.100"),
		),
		slog.Group("response",
			slog.Int("status", 201),
			slog.Int64("bytes", 256),
			slog.Duration("duration", 89*time.Millisecond),
		),
		slog.Group("user",
			slog.String("id", "usr_abc123"),
			slog.String("role", "admin"),
		),
	)
}
```

Output:
```json
{
  "time": "2025-03-15T14:32:01.123Z",
  "level": "INFO",
  "msg": "request completed",
  "request": {
    "method": "POST",
    "path": "/api/orders",
    "remote_addr": "192.168.1.100"
  },
  "response": {
    "status": 201,
    "bytes": 256,
    "duration": 89000000
  },
  "user": {
    "id": "usr_abc123",
    "role": "admin"
  }
}
```

### Implementing the LogValuer Interface

For your own types, implement `slog.LogValuer` to control how they appear in logs:

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
)

// User represents a user in the system.
type User struct {
	ID       string
	Email    string
	Name     string
	Password string // NEVER log this
	APIKey   string // NEVER log this
}

// LogValue implements slog.LogValuer.
// This controls exactly what appears in logs when you pass a User as an attribute.
func (u User) LogValue() slog.Value {
	return slog.GroupValue(
		slog.String("id", u.ID),
		slog.String("email", u.Email),
		slog.String("name", u.Name),
		// Password and APIKey are intentionally omitted
	)
}

// DBConfig is another example -- redact connection credentials.
type DBConfig struct {
	Host     string
	Port     int
	Database string
	User     string
	Password string
}

func (c DBConfig) LogValue() slog.Value {
	return slog.GroupValue(
		slog.String("host", c.Host),
		slog.Int("port", c.Port),
		slog.String("database", c.Database),
		slog.String("user", c.User),
		slog.String("password", "[REDACTED]"),
	)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	user := User{
		ID:       "usr_abc123",
		Email:    "alice@example.com",
		Name:     "Alice",
		Password: "super-secret-password",
		APIKey:   "sk_live_abc123xyz",
	}

	dbConfig := DBConfig{
		Host:     "db.production.internal",
		Port:     5432,
		Database: "krafty_core",
		User:     "app_user",
		Password: "db-password-here",
	}

	// Password and APIKey are automatically excluded from logs
	logger.Info("user authenticated", slog.Any("user", user))

	// Connection password is automatically redacted
	logger.Info("database connected", slog.Any("config", dbConfig))

	// Without LogValuer, you might accidentally log:
	fmt.Println("\n--- Without LogValuer (DANGEROUS) ---")
	fmt.Printf("User logged in: %+v\n", user) // Leaks password and API key!
}
```

Output:
```json
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"user authenticated","user":{"id":"usr_abc123","email":"alice@example.com","name":"Alice"}}
{"time":"2025-03-15T14:32:01.123Z","level":"INFO","msg":"database connected","config":{"host":"db.production.internal","port":5432,"database":"krafty_core","user":"app_user","password":"[REDACTED]"}}
```

**This is a critical security pattern.** By implementing `LogValuer`, you make it structurally impossible to accidentally log sensitive data, no matter where in the codebase someone passes a `User` to a logger.

---

## 6. Request ID Tracking

### Why Request IDs Matter

In a production system, a single user action ("click Buy") can trigger dozens of operations across multiple services: validate the order, check inventory, charge payment, send confirmation email, update analytics. When something goes wrong, you need to trace that entire chain.

A **request ID** is a unique identifier assigned to each incoming request and propagated through every service call, database query, and log line. It is the single most important piece of observability data you can add.

```
Without request IDs:
  ERROR database timeout
  ERROR payment failed
  INFO  order created
  ERROR email send failed
  // Which error goes with which request? No idea.

With request IDs:
  ERROR database timeout     request_id=abc-123
  INFO  order created        request_id=def-456
  ERROR payment failed       request_id=abc-123
  ERROR email send failed    request_id=abc-123
  // Clear: abc-123 had a cascading failure starting from the database.
```

### Generating and Propagating Request IDs

```go
package main

import (
	"context"
	"crypto/rand"
	"fmt"
	"log/slog"
	"net/http"
	"os"
)

// contextKey is an unexported type for context keys to prevent collisions.
type contextKey string

const RequestIDKey contextKey = "request_id"

// generateRequestID creates a cryptographically random ID.
// We use crypto/rand instead of math/rand for unpredictability.
// Format: 16 random hex bytes = 32 character string.
func generateRequestID() string {
	b := make([]byte, 16)
	_, err := rand.Read(b)
	if err != nil {
		// crypto/rand.Read should never fail on modern systems.
		// If it does, something is catastrophically wrong with the OS.
		panic(fmt.Sprintf("failed to generate request ID: %v", err))
	}
	return fmt.Sprintf("%x", b)
}

// requestIDFromContext extracts the request ID from the context.
func requestIDFromContext(ctx context.Context) string {
	if id, ok := ctx.Value(RequestIDKey).(string); ok {
		return id
	}
	return ""
}

// requestIDMiddleware assigns a request ID to every request.
// If the client sends X-Request-ID, we use it (after validation).
// Otherwise, we generate a new one.
func requestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		requestID := r.Header.Get("X-Request-ID")

		if !isValidRequestID(requestID) {
			requestID = generateRequestID()
		}

		// Add to context so downstream handlers and services can access it
		ctx := context.WithValue(r.Context(), RequestIDKey, requestID)

		// Add to response headers so the client can reference it in support tickets
		w.Header().Set("X-Request-ID", requestID)

		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// isValidRequestID checks if a client-provided request ID is safe to use.
// See Section 7 for detailed explanation.
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

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		requestID := requestIDFromContext(r.Context())

		// Every log line includes the request ID
		logger.InfoContext(r.Context(), "fetching orders",
			slog.String("request_id", requestID),
		)

		// Simulate calling another service
		logger.InfoContext(r.Context(), "calling inventory service",
			slog.String("request_id", requestID),
			slog.String("service", "inventory"),
		)

		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, `{"orders": [], "request_id": "%s"}`, requestID)
	})

	// Wrap with middleware
	handler := requestIDMiddleware(mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)
}
```

### Using UUID Libraries

In production, you typically use UUIDs (v4 or v7) for request IDs. Here is the pattern using `google/uuid`:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"

	// In a real project:
	// "github.com/google/uuid"
)

// In production with google/uuid:
//
//   import "github.com/google/uuid"
//
//   requestID := uuid.New().String()
//   // Output: "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
//
// UUID v7 (time-ordered, better for databases):
//   requestID := uuid.Must(uuid.NewV7()).String()
//   // Output: "018e4a2c-1234-7abc-8def-0123456789ab"

// For this example, we use a simple implementation.
func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	ctx := context.Background()

	// Simulating request ID generation
	requestID := "req_a1b2c3d4-e5f6-7890-abcd-ef1234567890"

	logger.InfoContext(ctx, "request received",
		slog.String("request_id", requestID),
		slog.String("method", "GET"),
		slog.String("path", "/api/orders"),
	)

	// The request ID propagates to all downstream calls
	logger.InfoContext(ctx, "database query",
		slog.String("request_id", requestID),
		slog.String("query", "SELECT * FROM orders WHERE user_id = $1"),
		slog.Int("rows", 42),
	)

	logger.InfoContext(ctx, "request completed",
		slog.String("request_id", requestID),
		slog.Int("status", 200),
	)

	fmt.Println("\n--- All three log lines share the same request_id ---")
	fmt.Println("--- You can search: jq 'select(.request_id == \"req_a1b2c3d4-e5f6-7890-abcd-ef1234567890\")' ---")
}
```

### Propagating Request IDs Across Services

When your service calls another service, forward the request ID:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

type contextKey string

const RequestIDKey contextKey = "request_id"

// callDownstreamService demonstrates propagating request IDs across service boundaries.
func callDownstreamService(ctx context.Context, logger *slog.Logger, serviceURL string) error {
	requestID, _ := ctx.Value(RequestIDKey).(string)

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, serviceURL, nil)
	if err != nil {
		return fmt.Errorf("creating request: %w", err)
	}

	// Forward the request ID to the downstream service
	req.Header.Set("X-Request-ID", requestID)

	logger.InfoContext(ctx, "calling downstream service",
		slog.String("request_id", requestID),
		slog.String("service_url", serviceURL),
	)

	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		logger.ErrorContext(ctx, "downstream service call failed",
			slog.String("request_id", requestID),
			slog.String("service_url", serviceURL),
			slog.String("error", err.Error()),
		)
		return fmt.Errorf("calling %s: %w", serviceURL, err)
	}
	defer resp.Body.Close()

	logger.InfoContext(ctx, "downstream service responded",
		slog.String("request_id", requestID),
		slog.String("service_url", serviceURL),
		slog.Int("status", resp.StatusCode),
	)

	return nil
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Simulate a request with a request ID
	ctx := context.WithValue(context.Background(), RequestIDKey, "req-abc-123")

	// This would call a real service in production
	err := callDownstreamService(ctx, logger, "http://inventory-service:8080/api/stock")
	if err != nil {
		logger.Error("operation failed", slog.String("error", err.Error()))
	}
}
```

### Node.js Comparison

```javascript
// Express middleware for request IDs
const { v4: uuidv4 } = require('uuid');
const pino = require('pino');
const pinoHttp = require('pino-http');

const logger = pino();

// Middleware
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.id);

  // Create a child logger with request ID bound
  req.log = logger.child({ request_id: req.id });
  next();
});

// In handlers:
app.get('/api/orders', (req, res) => {
  req.log.info({ method: req.method, path: req.path }, 'fetching orders');
  // ...
});

// pino-http does this automatically:
app.use(pinoHttp({
  genReqId: (req) => req.headers['x-request-id'] || uuidv4()
}));
```

The pattern is identical. The difference is that Go requires explicit context propagation (`context.Context`), while Node.js can attach the logger to the request object. Go's approach is more verbose but also more explicit -- you always know where the request ID comes from.

---

## 7. Log Injection Prevention

### The Attack

Log injection is a real security vulnerability. If you blindly log user-supplied values, an attacker can inject fake log entries:

```
// Attacker sends this as X-Request-ID:
// "legit-id\n{\"level\":\"INFO\",\"msg\":\"admin logged in\",\"user\":\"admin\"}"

// Without validation, your logs now contain:
{"level":"INFO","msg":"request","request_id":"legit-id
{"level":"INFO","msg":"admin logged in","user":"admin"}"}

// A log aggregation system might parse the injected line as a real log entry.
// An attacker could inject fake admin login events, hide their tracks, or
// trigger false alerts.
```

The `slog` JSON handler properly escapes special characters in string values, which prevents the most basic injection. But defense in depth means you should also validate inputs before they reach the logger.

### Validating Request IDs

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
)

// isValidRequestID ensures a client-supplied request ID is safe.
// Rules:
//   - Not empty
//   - Not longer than 128 characters (prevent memory abuse)
//   - Only alphanumeric, hyphens, and underscores (prevent injection)
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

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	testIDs := []struct {
		name string
		id   string
	}{
		{"valid UUID", "550e8400-e29b-41d4-a716-446655440000"},
		{"valid custom", "req_abc123_def456"},
		{"empty", ""},
		{"too long", string(make([]byte, 200))},
		{"injection attempt", "legit\n{\"fake\":\"log\"}"},
		{"special chars", "id<script>alert('xss')</script>"},
		{"null bytes", "id\x00injected"},
		{"valid short", "abc"},
		{"unicode", "id-\u00e9\u00e8\u00ea"},
	}

	for _, test := range testIDs {
		valid := isValidRequestID(test.id)
		status := "ACCEPTED"
		if !valid {
			status = "REJECTED"
		}
		logger.Info("request ID validation",
			slog.String("test", test.name),
			slog.Bool("valid", valid),
			slog.String("status", status),
		)
	}

	fmt.Println("\n--- Demonstration ---")

	// In the middleware, you would use it like this:
	clientID := "legit\n{\"fake\":\"log\"}"
	if !isValidRequestID(clientID) {
		// Generate a safe ID instead
		safeID := "generated-safe-id-12345"
		logger.Warn("invalid request ID from client, generated new one",
			slog.String("generated_id", safeID),
			// Do NOT log the original invalid ID without sanitization.
			// The slog JSON handler escapes it, but better safe than sorry.
			slog.Int("original_length", len(clientID)),
		)
	}
}
```

### Sanitizing Other User Input in Logs

Beyond request IDs, be careful with any user-supplied data that ends up in logs:

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
	"strings"
	"unicode"
)

// sanitizeForLog removes or replaces characters that could cause
// log parsing issues. Use this for any user-supplied string that
// you want to include in logs.
func sanitizeForLog(s string, maxLen int) string {
	if len(s) > maxLen {
		s = s[:maxLen] + "...[truncated]"
	}

	// Replace control characters (newlines, tabs, null bytes)
	var builder strings.Builder
	builder.Grow(len(s))

	for _, r := range s {
		if unicode.IsControl(r) {
			builder.WriteString(fmt.Sprintf("\\u%04x", r))
		} else {
			builder.WriteRune(r)
		}
	}

	return builder.String()
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Simulate user input that might be logged
	userAgent := "Mozilla/5.0\nX-Injected-Header: malicious"
	username := "alice\x00bob"                       // null byte injection
	searchQuery := strings.Repeat("x", 10000)       // extremely long input

	logger.Info("request received",
		slog.String("user_agent", sanitizeForLog(userAgent, 256)),
		slog.String("username", sanitizeForLog(username, 64)),
		slog.String("search_query", sanitizeForLog(searchQuery, 200)),
	)
}
```

**Important nuance:** Go's `slog.NewJSONHandler` already escapes special characters in JSON output (newlines become `\n`, quotes become `\"`). So log injection via JSON output is already prevented by the handler. However:

1. `TextHandler` does NOT escape the same way -- newlines in text output can still cause issues.
2. Even with proper JSON escaping, a 10MB user-agent string can fill your log storage.
3. Defense in depth: validate and truncate user input before it reaches the logger.

### Rate Limiting Log Output

An attacker can also abuse logging by triggering thousands of error logs per second, filling your log storage and potentially causing a denial of service:

```go
package main

import (
	"log/slog"
	"os"
	"sync"
	"time"
)

// rateLimitedLogger wraps a logger and limits how often a specific message can be logged.
type rateLimitedLogger struct {
	logger    *slog.Logger
	seen      map[string]time.Time
	mu        sync.Mutex
	interval  time.Duration
}

func newRateLimitedLogger(logger *slog.Logger, interval time.Duration) *rateLimitedLogger {
	return &rateLimitedLogger{
		logger:   logger,
		seen:     make(map[string]time.Time),
		interval: interval,
	}
}

// shouldLog returns true if the message has not been logged within the rate limit interval.
func (rl *rateLimitedLogger) shouldLog(key string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	if lastSeen, ok := rl.seen[key]; ok {
		if time.Since(lastSeen) < rl.interval {
			return false
		}
	}
	rl.seen[key] = time.Now()
	return true
}

func main() {
	baseLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	rl := newRateLimitedLogger(baseLogger, 5*time.Second)

	// Simulate an attack: 1000 rapid invalid requests
	for i := 0; i < 1000; i++ {
		if rl.shouldLog("invalid_auth_token") {
			rl.logger.Warn("invalid auth token",
				slog.String("ip", "192.168.1.100"),
				slog.String("note", "rate limited to once per 5 seconds"),
			)
		}
	}

	// Only ONE log line was produced instead of 1000
	baseLogger.Info("done", slog.String("note", "only 1 warn log was emitted from 1000 attempts"))
}
```

---

## 8. Response Writer Wrapping

### The Problem

Go's `http.ResponseWriter` does not expose the status code or number of bytes written after a handler completes. But for request logging, you need both. The solution is to wrap `ResponseWriter` with a type that intercepts `WriteHeader` and `Write` calls.

### Basic Response Writer Wrapper

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// responseWriter wraps http.ResponseWriter to capture response metadata.
type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

// newResponseWriter creates a wrapped ResponseWriter with sensible defaults.
func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK, // Default if WriteHeader is never called
	}
}

// WriteHeader captures the status code. It ensures WriteHeader is only
// called once on the underlying writer, matching Go's http.ResponseWriter contract.
func (rw *responseWriter) WriteHeader(code int) {
	if rw.headerWritten {
		return // Prevent double WriteHeader, which causes a warning
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

// Write captures the number of bytes written and ensures headers are sent.
func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK) // Implicit 200, matching net/http behavior
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

// loggingMiddleware logs every request with status code, duration, and bytes.
func loggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := newResponseWriter(w)

		// Call the actual handler
		next.ServeHTTP(wrapped, r)

		// Log after the handler completes
		duration := time.Since(start)
		logger.LogAttrs(r.Context(), slog.LevelInfo, "request completed",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", wrapped.statusCode),
			slog.Duration("duration", duration),
			slog.Int64("bytes", wrapped.written),
			slog.String("remote_addr", r.RemoteAddr),
		)
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		fmt.Fprint(w, `{"orders": [{"id": 1, "item": "Widget"}]}`)
	})

	mux.HandleFunc("GET /api/not-found", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "not found", http.StatusNotFound)
	})

	mux.HandleFunc("GET /api/error", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "internal server error", http.StatusInternalServerError)
	})

	handler := loggingMiddleware(logger, mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)
}
```

### Supporting http.Flusher and http.Hijacker

In production, your response writer wrapper must also implement optional interfaces that middleware and handlers might need. If you do not, features like Server-Sent Events (SSE), WebSockets, and streaming responses break silently.

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"net/http"
)

// responseWriter wraps http.ResponseWriter to capture response metadata.
// It also conditionally implements http.Flusher and http.Hijacker.
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
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

// Flush implements http.Flusher. Required for streaming responses and SSE.
func (rw *responseWriter) Flush() {
	if f, ok := rw.ResponseWriter.(http.Flusher); ok {
		f.Flush()
	}
}

// Hijack implements http.Hijacker. Required for WebSocket upgrades.
func (rw *responseWriter) Hijack() (net.Conn, *bufio.ReadWriter, error) {
	if h, ok := rw.ResponseWriter.(http.Hijacker); ok {
		return h.Hijack()
	}
	return nil, nil, fmt.Errorf("underlying ResponseWriter does not implement http.Hijacker")
}

// Unwrap returns the underlying ResponseWriter.
// This is important for Go 1.20+ which uses Unwrap to check for optional interfaces.
func (rw *responseWriter) Unwrap() http.ResponseWriter {
	return rw.ResponseWriter
}

func main() {
	// The key insight: always implement Unwrap() so Go's http package
	// can discover the original ResponseWriter's capabilities.
	fmt.Println("Response writer wrapper with Flusher, Hijacker, and Unwrap support")
	fmt.Println("This ensures SSE, WebSockets, and streaming work through the middleware")
}
```

**Why this matters:** If your logging middleware wraps `ResponseWriter` but does not implement `http.Flusher`, then any handler that tries to flush (for SSE or streaming) will fail. The handler calls `w.(http.Flusher)`, the type assertion fails, and the handler panics. This is one of the most common bugs in Go HTTP middleware.

### Node.js Comparison

Node.js/Express does not have this problem because the response object is always the same class with the same methods. You can monkey-patch it:

```javascript
// Express middleware to capture status and bytes
app.use((req, res, next) => {
  const start = Date.now();
  const originalEnd = res.end;

  res.end = function(chunk, encoding) {
    const duration = Date.now() - start;
    logger.info({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration,
      bytes: res.get('content-length') || 0,
    }, 'request completed');

    originalEnd.call(this, chunk, encoding);
  };

  next();
});
```

Go's approach is more explicit and type-safe, but requires understanding the interface delegation pattern. Node.js is simpler here but relies on monkey-patching, which can have unexpected side effects.

---

## 9. Query Parameter Redaction

### The Problem

URLs often contain sensitive data: API keys, session tokens, passwords in form submissions. If you log full URLs, you are logging credentials.

```
// DANGEROUS: logging full URLs
GET /api/data?token=sk_live_abc123xyz&user=alice
GET /auth/callback?code=oauth_secret_code_here
GET /reset-password?token=password_reset_token
```

These logs are stored in your log aggregation system, backed up, and accessible to anyone with log access. You have effectively turned your log storage into a credential database.

### Redacting Sensitive Query Parameters

```go
package main

import (
	"fmt"
	"log/slog"
	"net/url"
	"os"
	"strings"
)

// sensitiveParams is the list of query parameter name patterns to redact.
// We use contains-matching so "api_key", "apiKey", "x-api-key" are all caught.
var sensitiveParams = []string{
	"token",
	"api_key",
	"apikey",
	"password",
	"passwd",
	"secret",
	"auth",
	"credential",
	"key",
	"session",
	"jwt",
	"bearer",
	"access_token",
	"refresh_token",
}

// redactQuery takes a raw query string and returns it with sensitive values replaced.
func redactQuery(rawQuery string) string {
	if rawQuery == "" {
		return ""
	}

	values, err := url.ParseQuery(rawQuery)
	if err != nil {
		// If we cannot parse the query, redact the entire thing.
		// Better to lose the data than to leak credentials.
		return "[REDACTED_UNPARSEABLE]"
	}

	for key := range values {
		lowerKey := strings.ToLower(key)
		for _, sensitive := range sensitiveParams {
			if strings.Contains(lowerKey, sensitive) {
				values.Set(key, "[REDACTED]")
				break
			}
		}
	}

	return values.Encode()
}

// redactURL takes a full URL and returns it with sensitive query parameters redacted.
func redactURL(rawURL string) string {
	u, err := url.Parse(rawURL)
	if err != nil {
		return "[REDACTED_UNPARSEABLE_URL]"
	}

	u.RawQuery = redactQuery(u.RawQuery)
	return u.String()
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	testURLs := []string{
		"/api/data?token=sk_live_abc123&user=alice&page=1",
		"/auth/callback?code=normal_param&api_key=secret123&format=json",
		"/api/users?password=hunter2&username=bob",
		"/api/search?q=hello+world&page=1&limit=10",
		"/webhook?secret_token=abc123&event=push",
		"/api/data?access_token=jwt_here&refresh_token=refresh_here",
		"/api/orders?page=1&sort=date", // No sensitive params
		"",                              // Empty query
	}

	for _, rawURL := range testURLs {
		redacted := redactURL(rawURL)
		logger.Info("request",
			slog.String("original", rawURL),
			slog.String("redacted", redacted),
		)
		fmt.Printf("  %s\n  -> %s\n\n", rawURL, redacted)
	}
}
```

Output:
```
  /api/data?token=sk_live_abc123&user=alice&page=1
  -> /api/data?page=1&token=%5BREDACTED%5D&user=alice

  /auth/callback?code=normal_param&api_key=secret123&format=json
  -> /auth/callback?api_key=%5BREDACTED%5D&code=normal_param&format=json

  /api/users?password=hunter2&username=bob
  -> /api/users?password=%5BREDACTED%5D&username=bob

  /api/search?q=hello+world&page=1&limit=10
  -> /api/search?limit=10&page=1&q=hello+world

  /api/orders?page=1&sort=date
  -> /api/orders?page=1&sort=date
```

### Integration with Request Logging

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

var sensitiveParams = []string{"token", "api_key", "password", "secret", "auth", "key"}

func redactQuery(rawQuery string) string {
	if rawQuery == "" {
		return ""
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil {
		return "[REDACTED]"
	}
	for key := range values {
		lowerKey := strings.ToLower(key)
		for _, sensitive := range sensitiveParams {
			if strings.Contains(lowerKey, sensitive) {
				values.Set(key, "[REDACTED]")
				break
			}
		}
	}
	return values.Encode()
}

type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

func loggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := newResponseWriter(w)

		next.ServeHTTP(wrapped, r)

		duration := time.Since(start)

		// Build the path with redacted query
		logPath := r.URL.Path
		if r.URL.RawQuery != "" {
			logPath = r.URL.Path + "?" + redactQuery(r.URL.RawQuery)
		}

		logger.LogAttrs(r.Context(), slog.LevelInfo, "request completed",
			slog.String("method", r.Method),
			slog.String("path", logPath),
			slog.Int("status", wrapped.statusCode),
			slog.Duration("duration", duration),
			slog.Int64("bytes", wrapped.written),
		)
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/data", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, `{"data": "safe response"}`)
	})

	handler := loggingMiddleware(logger, mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)

	// Test with: curl "http://localhost:8080/api/data?token=secret123&page=1"
	// Log output will show: path="/api/data?page=1&token=[REDACTED]"
}
```

### Redacting Request Headers

Similarly, redact sensitive headers:

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"strings"
)

// sensitiveHeaders lists headers that should be redacted in logs.
var sensitiveHeaders = map[string]bool{
	"authorization": true,
	"cookie":        true,
	"set-cookie":    true,
	"x-api-key":    true,
	"x-auth-token": true,
}

// redactHeaders returns a map of headers safe for logging.
func redactHeaders(headers http.Header) map[string]string {
	safe := make(map[string]string, len(headers))
	for key, values := range headers {
		if sensitiveHeaders[strings.ToLower(key)] {
			safe[key] = "[REDACTED]"
		} else {
			safe[key] = strings.Join(values, ", ")
		}
	}
	return safe
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Simulate incoming request headers
	headers := http.Header{
		"Content-Type":  {"application/json"},
		"Accept":        {"application/json"},
		"Authorization": {"Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.secret"},
		"Cookie":        {"session=abc123; preferences=dark"},
		"X-Request-ID":  {"req-12345"},
		"X-Api-Key":     {"sk_live_very_secret_key"},
		"User-Agent":    {"Mozilla/5.0"},
	}

	safe := redactHeaders(headers)

	logger.Info("request headers", slog.Any("headers", safe))

	for key, value := range safe {
		fmt.Printf("  %s: %s\n", key, value)
	}
}
```

---

## 10. Log Level per Status Code

### Mapping HTTP Status Codes to Log Levels

Not all requests deserve the same log level. A successful request is informational. A client error (404, 400) is a warning. A server error (500) is an error that needs investigation.

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// logLevelForStatus returns the appropriate slog level for an HTTP status code.
//
//	2xx -> INFO  (success, normal operation)
//	3xx -> INFO  (redirects are normal)
//	4xx -> WARN  (client error, not our bug, but worth monitoring)
//	5xx -> ERROR (server error, our bug, needs investigation)
func logLevelForStatus(status int) slog.Level {
	switch {
	case status >= 500:
		return slog.LevelError
	case status >= 400:
		return slog.LevelWarn
	default:
		return slog.LevelInfo
	}
}

type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

func loggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := newResponseWriter(w)

		next.ServeHTTP(wrapped, r)

		duration := time.Since(start)
		level := logLevelForStatus(wrapped.statusCode)

		// Use the appropriate level based on the response status
		logger.LogAttrs(r.Context(), level, "request completed",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", wrapped.statusCode),
			slog.Duration("duration", duration),
			slog.Int64("bytes", wrapped.written),
			slog.String("remote_addr", r.RemoteAddr),
		)
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()

	// 200 OK -> logged at INFO
	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, `{"orders": []}`)
	})

	// 400 Bad Request -> logged at WARN
	mux.HandleFunc("POST /api/orders", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, `{"error": "missing required field: name"}`, http.StatusBadRequest)
	})

	// 404 Not Found -> logged at WARN
	mux.HandleFunc("GET /api/legacy", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, `{"error": "not found"}`, http.StatusNotFound)
	})

	// 500 Internal Server Error -> logged at ERROR
	mux.HandleFunc("GET /api/broken", func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, `{"error": "internal server error"}`, http.StatusInternalServerError)
	})

	handler := loggingMiddleware(logger, mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)
}
```

### Why Separate 4xx and 5xx?

This distinction is critical for alerting:

```
Alerting Rules:
  - 5xx rate > 1% of total requests  -> PAGE on-call engineer (our bug)
  - 4xx rate > 10% of total requests -> Notify team in Slack (possibly a client bug, API misuse, or attack)
  - 4xx on a single endpoint spikes  -> Investigate (possibly a breaking API change)
```

If you log 4xx at ERROR level, your error rate metric includes client errors, and you get paged for things that are not your fault (like a bot sending malformed requests). If you log 5xx at WARN level, you miss actual server failures.

### Fine-Grained Status Code Mapping

For more nuanced logging, you might want to treat certain 4xx codes differently:

```go
package main

import (
	"fmt"
	"log/slog"
)

// detailedLogLevel provides more nuanced level mapping.
func detailedLogLevel(status int) slog.Level {
	switch {
	case status >= 500:
		return slog.LevelError

	// 429 Too Many Requests -- might indicate an attack or misconfigured client
	case status == 429:
		return slog.LevelWarn

	// 401/403 -- authentication/authorization failures, worth monitoring
	case status == 401 || status == 403:
		return slog.LevelWarn

	// 404 -- usually noise from bots scanning for vulnerabilities
	// In high-traffic systems, you might even log these at DEBUG
	case status == 404:
		return slog.LevelInfo

	// 400 -- client sent bad data, warn so we can check API docs
	case status == 400:
		return slog.LevelWarn

	// Other 4xx -- warn
	case status >= 400:
		return slog.LevelWarn

	// 2xx, 3xx -- normal operation
	default:
		return slog.LevelInfo
	}
}

func main() {
	testCases := []int{200, 201, 301, 400, 401, 403, 404, 429, 500, 502, 503}

	for _, status := range testCases {
		level := detailedLogLevel(status)
		fmt.Printf("  HTTP %d -> %s\n", status, level)
	}
}
```

Output:
```
  HTTP 200 -> INFO
  HTTP 201 -> INFO
  HTTP 301 -> INFO
  HTTP 400 -> WARN
  HTTP 401 -> WARN
  HTTP 403 -> WARN
  HTTP 404 -> INFO
  HTTP 429 -> WARN
  HTTP 500 -> ERROR
  HTTP 502 -> ERROR
  HTTP 503 -> ERROR
```

---

## 11. Logger Dependency Injection

### The Anti-Pattern: Global Logger

```go
// DO NOT DO THIS in production code
package main

import "log/slog"

// Global mutable state -- impossible to test, impossible to configure per-component
var logger = slog.Default()

func handleOrder() {
	logger.Info("processing order") // Which logger? What configuration? Unknown.
}
```

Global loggers are convenient but create several problems:
1. **Testing:** You cannot capture log output in tests without modifying global state.
2. **Configuration:** You cannot have different log levels for different components.
3. **Concurrency:** Replacing the global logger while requests are in flight is a race condition.

### The Pattern: Constructor Injection

Pass the logger through constructors, just like you pass a database connection:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// OrderService handles order-related business logic.
type OrderService struct {
	logger *slog.Logger
	// db     *sql.DB  // Would have other dependencies too
}

// NewOrderService creates an OrderService with its dependencies.
func NewOrderService(logger *slog.Logger) *OrderService {
	return &OrderService{
		logger: logger.With(slog.String("component", "order_service")),
	}
}

// CreateOrder processes a new order.
func (s *OrderService) CreateOrder(ctx context.Context, userID string, amount float64) (string, error) {
	s.logger.InfoContext(ctx, "creating order",
		slog.String("user_id", userID),
		slog.Float64("amount", amount),
	)

	// Simulate order creation
	orderID := fmt.Sprintf("ord_%d", time.Now().UnixNano())

	s.logger.InfoContext(ctx, "order created",
		slog.String("order_id", orderID),
		slog.String("user_id", userID),
		slog.Float64("amount", amount),
	)

	return orderID, nil
}

// OrderHandler handles HTTP requests for orders.
type OrderHandler struct {
	logger  *slog.Logger
	service *OrderService
}

// NewOrderHandler creates an OrderHandler with its dependencies.
func NewOrderHandler(logger *slog.Logger, service *OrderService) *OrderHandler {
	return &OrderHandler{
		logger:  logger.With(slog.String("component", "order_handler")),
		service: service,
	}
}

func (h *OrderHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		UserID string  `json:"user_id"`
		Amount float64 `json:"amount"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		h.logger.WarnContext(r.Context(), "invalid request body",
			slog.String("error", err.Error()),
		)
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	orderID, err := h.service.CreateOrder(r.Context(), req.UserID, req.Amount)
	if err != nil {
		h.logger.ErrorContext(r.Context(), "failed to create order",
			slog.String("error", err.Error()),
		)
		http.Error(w, "internal server error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]string{"order_id": orderID})
}

func main() {
	// Create the root logger
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	// Wire up dependencies
	orderService := NewOrderService(logger)
	orderHandler := NewOrderHandler(logger, orderService)

	mux := http.NewServeMux()
	mux.Handle("POST /api/orders", orderHandler)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", mux)
}
```

Each log line now includes the component name, making it easy to filter:
```json
{"level":"INFO","msg":"creating order","component":"order_service","user_id":"usr_123","amount":42.99}
{"level":"INFO","msg":"order created","component":"order_service","order_id":"ord_1710512421","user_id":"usr_123","amount":42.99}
```

### Logger in Context

An alternative pattern puts the logger itself into the context. This works well for request-scoped loggers that carry request IDs:

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
)

type contextKey string

const loggerKey contextKey = "logger"

// withLogger stores a logger in the context.
func withLogger(ctx context.Context, logger *slog.Logger) context.Context {
	return context.WithValue(ctx, loggerKey, logger)
}

// fromContext retrieves the logger from the context.
// Falls back to the default logger if none is set.
func fromContext(ctx context.Context) *slog.Logger {
	if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
		return logger
	}
	return slog.Default()
}

// requestLoggerMiddleware creates a request-scoped logger with the request ID
// and stores it in the context.
func requestLoggerMiddleware(baseLogger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		requestID := r.Header.Get("X-Request-ID")
		if requestID == "" {
			requestID = "gen-12345" // Would use uuid.New() in production
		}

		// Create a child logger with the request ID baked in.
		// Every log call through this logger automatically includes request_id.
		reqLogger := baseLogger.With(
			slog.String("request_id", requestID),
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
		)

		// Store in context
		ctx := withLogger(r.Context(), reqLogger)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
	logger := fromContext(r.Context())

	// This log line automatically includes request_id, method, and path
	// because they were added to the logger in the middleware.
	logger.Info("fetching orders from database")
	logger.Info("orders fetched", slog.Int("count", 42))

	w.WriteHeader(http.StatusOK)
}

func main() {
	baseLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/orders", handleOrders)

	handler := requestLoggerMiddleware(baseLogger, mux)

	baseLogger.Info("server starting")
	http.ListenAndServe(":8080", handler)
}
```

Output for a request:
```json
{"level":"INFO","msg":"fetching orders from database","request_id":"gen-12345","method":"GET","path":"/api/orders"}
{"level":"INFO","msg":"orders fetched","request_id":"gen-12345","method":"GET","path":"/api/orders","count":42}
```

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| Constructor injection | Explicit, testable, IDE-friendly | More boilerplate |
| Context injection | Request-scoped loggers with automatic attributes | Hidden dependency, type assertion at retrieval |

**Recommendation:** Use constructor injection for services and repositories (long-lived objects). Use context injection for request-scoped loggers in HTTP handlers.

### Testing with Injected Loggers

The whole point of dependency injection is testability:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log/slog"
	"strings"
)

// OrderService with an injected logger (same as above, simplified).
type OrderService struct {
	logger *slog.Logger
}

func NewOrderService(logger *slog.Logger) *OrderService {
	return &OrderService{logger: logger}
}

func (s *OrderService) ProcessOrder(orderID string) error {
	s.logger.Info("processing order", slog.String("order_id", orderID))
	// ... business logic ...
	s.logger.Info("order processed", slog.String("order_id", orderID))
	return nil
}

func main() {
	// In tests, capture log output with a buffer:
	var buf bytes.Buffer
	testLogger := slog.New(slog.NewJSONHandler(&buf, nil))

	service := NewOrderService(testLogger)
	service.ProcessOrder("ord_123")

	// Now you can assert on the log output
	output := buf.String()
	fmt.Println("--- Captured log output ---")
	fmt.Println(output)

	// Parse and verify
	lines := strings.Split(strings.TrimSpace(output), "\n")
	for _, line := range lines {
		var entry map[string]interface{}
		json.Unmarshal([]byte(line), &entry)
		fmt.Printf("Message: %s, Order: %s\n", entry["msg"], entry["order_id"])
	}

	// Assertions you would make in a test:
	// assert.Contains(t, output, "processing order")
	// assert.Contains(t, output, "order processed")
	// assert.Contains(t, output, "ord_123")
}
```

---

## 12. Log Grouping and Child Loggers

### slog.With -- Adding Persistent Attributes

`slog.Logger.With()` creates a new logger with additional attributes that are included in every log call. This is the single most useful feature for production logging.

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	baseLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Create child loggers for different components.
	// Each one includes a "component" attribute in every log line.
	dbLogger := baseLogger.With(
		slog.String("component", "database"),
		slog.String("db_host", "db.production.internal"),
	)

	cacheLogger := baseLogger.With(
		slog.String("component", "cache"),
		slog.String("cache_host", "redis.production.internal"),
	)

	httpLogger := baseLogger.With(
		slog.String("component", "http_server"),
		slog.Int("port", 8080),
	)

	// Each logger automatically includes its persistent attributes
	dbLogger.Info("connection pool initialized",
		slog.Int("max_conns", 25),
		slog.Int("idle_conns", 5),
	)

	cacheLogger.Info("connected",
		slog.Duration("latency", 2*time.Millisecond),
	)

	httpLogger.Info("server started",
		slog.String("addr", ":8080"),
	)

	// You can chain With calls to build up context
	requestLogger := httpLogger.With(
		slog.String("request_id", "req-abc-123"),
		slog.String("user_id", "usr-456"),
	)

	requestLogger.Info("handling request") // Includes component, port, request_id, user_id
}
```

Output:
```json
{"time":"...","level":"INFO","msg":"connection pool initialized","component":"database","db_host":"db.production.internal","max_conns":25,"idle_conns":5}
{"time":"...","level":"INFO","msg":"connected","component":"cache","cache_host":"redis.production.internal","latency":2000000}
{"time":"...","level":"INFO","msg":"server started","component":"http_server","port":8080,"addr":":8080"}
{"time":"...","level":"INFO","msg":"handling request","component":"http_server","port":8080,"request_id":"req-abc-123","user_id":"usr-456"}
```

### WithGroup -- Namespacing Attributes

`WithGroup` prefixes all subsequent attributes under a named group, creating nested JSON objects:

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// WithGroup creates a namespace for attributes
	reqLogger := logger.WithGroup("request").With(
		slog.String("method", "POST"),
		slog.String("path", "/api/orders"),
		slog.String("remote_addr", "192.168.1.100"),
	)

	// These attributes go under the "request" group
	reqLogger.Info("request received")

	// Combine groups for deeply nested structure
	serviceLogger := logger.
		WithGroup("service").With(slog.String("name", "order-service")).
		WithGroup("instance").With(
			slog.String("id", "i-abc123"),
			slog.String("region", "us-east-1"),
		)

	serviceLogger.Info("health check passed",
		slog.Duration("uptime", 48*time.Hour),
	)
}
```

Output:
```json
{"time":"...","level":"INFO","msg":"request received","request":{"method":"POST","path":"/api/orders","remote_addr":"192.168.1.100"}}
{"time":"...","level":"INFO","msg":"health check passed","service":{"name":"order-service","instance":{"id":"i-abc123","region":"us-east-1","uptime":172800000000000}}}
```

### Real-World Child Logger Hierarchy

```go
package main

import (
	"context"
	"log/slog"
	"os"
	"time"
)

// Application demonstrates a realistic logger hierarchy.
type Application struct {
	logger *slog.Logger
}

type DatabasePool struct {
	logger *slog.Logger
}

type UserRepository struct {
	logger *slog.Logger
	db     *DatabasePool
}

type UserService struct {
	logger *slog.Logger
	repo   *UserRepository
}

func main() {
	// Root logger: application-wide settings
	rootLogger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	})).With(
		slog.String("app", "krafty-core"),
		slog.String("version", "1.2.3"),
		slog.String("env", "production"),
	)

	// Each layer gets its own child logger
	app := &Application{
		logger: rootLogger.With(slog.String("component", "app")),
	}

	dbPool := &DatabasePool{
		logger: rootLogger.With(
			slog.String("component", "database"),
			slog.String("driver", "postgres"),
		),
	}

	userRepo := &UserRepository{
		logger: rootLogger.With(slog.String("component", "user_repository")),
		db:     dbPool,
	}

	userService := &UserService{
		logger: rootLogger.With(slog.String("component", "user_service")),
		repo:   userRepo,
	}

	// Simulate application lifecycle
	ctx := context.Background()

	app.logger.InfoContext(ctx, "application starting")

	dbPool.logger.InfoContext(ctx, "connecting to database",
		slog.String("host", "db.production.internal"),
		slog.Int("port", 5432),
	)

	dbPool.logger.InfoContext(ctx, "connection pool ready",
		slog.Int("pool_size", 25),
		slog.Duration("connect_time", 150*time.Millisecond),
	)

	userService.logger.InfoContext(ctx, "looking up user",
		slog.String("user_id", "usr_abc123"),
	)

	userRepo.logger.DebugContext(ctx, "executing query",
		slog.String("query", "SELECT * FROM users WHERE id = $1"),
	)

	userService.logger.InfoContext(ctx, "user found",
		slog.String("user_id", "usr_abc123"),
		slog.String("email", "alice@example.com"),
	)

	_ = app
	_ = userRepo
}
```

Every log line includes the `app`, `version`, `env`, and `component` fields. In your log aggregation system, you can filter by:
- `app=krafty-core` -- all logs from this application
- `component=database` -- all database-related logs
- `component=user_service AND level=ERROR` -- errors in the user service
- `env=production AND version=1.2.3` -- logs from a specific deployment

---

## 13. Health Check Logging

### The Problem: Noisy Health Checks

In Kubernetes, the kubelet sends health check requests every 10 seconds to every pod. With a typical setup:

```
- 10 pods
- 2 probes per pod (liveness + readiness)
- Every 10 seconds
= 120 health check log lines per minute
= 172,800 per day
= Pure noise
```

These health check logs drown out meaningful request logs. You need to either suppress them entirely or log them at DEBUG level.

### Pattern 1: Skip Health Check Paths

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// healthCheckPaths lists paths that should not be logged at INFO level.
var healthCheckPaths = map[string]bool{
	"/healthz":  true,
	"/readyz":   true,
	"/livez":    true,
	"/health":   true,
	"/ready":    true,
	"/ping":     true,
	"/metrics":  true, // Prometheus scrapes every 15s
}

type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

func loggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := newResponseWriter(w)

		next.ServeHTTP(wrapped, r)

		duration := time.Since(start)

		// Skip logging for health check paths (or log at DEBUG)
		if healthCheckPaths[r.URL.Path] {
			// Option A: Skip entirely
			// return

			// Option B: Log at DEBUG level (visible only when DEBUG is enabled)
			logger.LogAttrs(r.Context(), slog.LevelDebug, "health check",
				slog.String("path", r.URL.Path),
				slog.Int("status", wrapped.statusCode),
				slog.Duration("duration", duration),
			)
			return
		}

		// Normal request logging at INFO/WARN/ERROR
		level := slog.LevelInfo
		if wrapped.statusCode >= 500 {
			level = slog.LevelError
		} else if wrapped.statusCode >= 400 {
			level = slog.LevelWarn
		}

		logger.LogAttrs(r.Context(), level, "request completed",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", wrapped.statusCode),
			slog.Duration("duration", duration),
			slog.Int64("bytes", wrapped.written),
		)
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo, // Health checks at DEBUG will be suppressed
	}))

	mux := http.NewServeMux()

	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprint(w, "ok")
	})

	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		// In production, check database, cache, etc.
		w.WriteHeader(http.StatusOK)
		fmt.Fprint(w, "ok")
	})

	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, `{"orders": []}`)
	})

	handler := loggingMiddleware(logger, mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)
}
```

### Pattern 2: User-Agent Based Filtering

Some health checks come from load balancers with specific user agents:

```go
package main

import (
	"fmt"
	"strings"
)

// isHealthCheckRequest detects health check requests from various sources.
func isHealthCheckRequest(path string, userAgent string) bool {
	// Path-based detection
	healthPaths := map[string]bool{
		"/healthz": true,
		"/readyz":  true,
		"/livez":   true,
		"/health":  true,
		"/ping":    true,
	}
	if healthPaths[path] {
		return true
	}

	// User-Agent based detection
	ua := strings.ToLower(userAgent)
	healthCheckAgents := []string{
		"kube-probe",                    // Kubernetes
		"elb-healthchecker",             // AWS ALB/NLB
		"googlehc",                      // Google Cloud load balancer
		"azure traffic manager",         // Azure
		"health check",                  // Generic
		"datadog",                       // Datadog synthetic monitoring
		"uptime",                        // Various uptime monitors
	}
	for _, agent := range healthCheckAgents {
		if strings.Contains(ua, agent) {
			return true
		}
	}

	return false
}

func main() {
	testCases := []struct {
		path      string
		userAgent string
	}{
		{"/healthz", "kube-probe/1.28"},
		{"/api/orders", "Mozilla/5.0"},
		{"/readyz", ""},
		{"/api/users", "ELB-HealthChecker/2.0"},
		{"/api/data", "curl/7.88.0"},
	}

	for _, tc := range testCases {
		isHealth := isHealthCheckRequest(tc.path, tc.userAgent)
		fmt.Printf("  path=%-15s ua=%-30s health_check=%v\n", tc.path, tc.userAgent, isHealth)
	}
}
```

### Node.js Comparison

```javascript
// Express with pino-http
const pinoHttp = require('pino-http');

app.use(pinoHttp({
  // Suppress health check logging
  autoLogging: {
    ignore: (req) => {
      return ['/healthz', '/readyz', '/metrics'].includes(req.url);
    }
  }
}));

// Or with morgan:
app.use(morgan('combined', {
  skip: (req, res) => req.url === '/healthz'
}));
```

The concept is identical. Both ecosystems need to handle the noise problem.

---

## 14. Metrics and Tracing Basics

### The Three Pillars of Observability

Logging is one of three observability pillars. Understanding how they fit together is essential:

```
┌─────────────────────────────────────────────────────────────┐
│                  Three Pillars of Observability              │
├─────────────┬──────────────────┬────────────────────────────┤
│   Logging   │    Metrics       │    Tracing                 │
├─────────────┼──────────────────┼────────────────────────────┤
│ What        │ What             │ What                       │
│ happened?   │ is happening?    │ path did it take?          │
├─────────────┼──────────────────┼────────────────────────────┤
│ Discrete    │ Aggregated       │ Request-scoped             │
│ events      │ measurements     │ journey                    │
├─────────────┼──────────────────┼────────────────────────────┤
│ Text/JSON   │ Numeric time     │ Spans with                 │
│ lines       │ series           │ parent-child               │
├─────────────┼──────────────────┼────────────────────────────┤
│ slog,       │ Prometheus,      │ OpenTelemetry,             │
│ zap,        │ StatsD,          │ Jaeger, Zipkin,            │
│ zerolog     │ Datadog          │ Tempo                      │
├─────────────┼──────────────────┼────────────────────────────┤
│ "Order      │ request_count:   │ Span: HTTP GET /orders     │
│  ABC123     │ 1500/min         │  └─ Span: DB query         │
│  failed:    │ p99_latency:     │      └─ Span: Cache lookup │
│  db timeout"│ 450ms            │                            │
└─────────────┴──────────────────┴────────────────────────────┘
```

**Logs** tell you what happened to a specific request. **Metrics** tell you how the system is performing overall. **Traces** show you the path a request took through multiple services.

### Prometheus Metrics in Go

Prometheus is the standard for metrics in Go applications. Here is how to add basic request metrics:

```go
package main

import (
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"

	// In a real project:
	// "github.com/prometheus/client_golang/prometheus"
	// "github.com/prometheus/client_golang/prometheus/promhttp"
)

// This example shows the structure. In production, use the actual prometheus client.

// MetricsCollector holds Prometheus metrics for HTTP requests.
type MetricsCollector struct {
	// In production with real Prometheus:
	//
	// requestsTotal *prometheus.CounterVec
	// requestDuration *prometheus.HistogramVec
	// requestsInFlight prometheus.Gauge
	// responseSize *prometheus.HistogramVec

	// For this example, we use simple counters
	totalRequests int64
	totalErrors   int64
}

func NewMetricsCollector() *MetricsCollector {
	// In production:
	//
	// m := &MetricsCollector{
	//     requestsTotal: prometheus.NewCounterVec(
	//         prometheus.CounterOpts{
	//             Name: "http_requests_total",
	//             Help: "Total number of HTTP requests",
	//         },
	//         []string{"method", "path", "status"},
	//     ),
	//     requestDuration: prometheus.NewHistogramVec(
	//         prometheus.HistogramOpts{
	//             Name:    "http_request_duration_seconds",
	//             Help:    "HTTP request duration in seconds",
	//             Buckets: prometheus.DefBuckets,
	//         },
	//         []string{"method", "path"},
	//     ),
	//     requestsInFlight: prometheus.NewGauge(
	//         prometheus.GaugeOpts{
	//             Name: "http_requests_in_flight",
	//             Help: "Number of HTTP requests currently being processed",
	//         },
	//     ),
	// }
	// prometheus.MustRegister(m.requestsTotal, m.requestDuration, m.requestsInFlight)
	// return m

	return &MetricsCollector{}
}

// RecordRequest records metrics for a completed HTTP request.
func (m *MetricsCollector) RecordRequest(method, path string, status int, duration time.Duration) {
	m.totalRequests++
	if status >= 500 {
		m.totalErrors++
	}

	// In production:
	// statusStr := strconv.Itoa(status)
	// m.requestsTotal.WithLabelValues(method, path, statusStr).Inc()
	// m.requestDuration.WithLabelValues(method, path).Observe(duration.Seconds())
}

type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	written       int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

// observabilityMiddleware combines logging and metrics collection.
func observabilityMiddleware(logger *slog.Logger, metrics *MetricsCollector, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		wrapped := newResponseWriter(w)

		// In production with Prometheus:
		// metrics.requestsInFlight.Inc()
		// defer metrics.requestsInFlight.Dec()

		next.ServeHTTP(wrapped, r)

		duration := time.Since(start)

		// Record metrics
		metrics.RecordRequest(r.Method, r.URL.Path, wrapped.statusCode, duration)

		// Log the request
		level := slog.LevelInfo
		if wrapped.statusCode >= 500 {
			level = slog.LevelError
		} else if wrapped.statusCode >= 400 {
			level = slog.LevelWarn
		}

		logger.LogAttrs(r.Context(), level, "request completed",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Int("status", wrapped.statusCode),
			slog.Duration("duration", duration),
			slog.Int64("bytes", wrapped.written),
		)
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	metrics := NewMetricsCollector()

	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprint(w, `{"orders": []}`)
	})

	// In production, expose Prometheus metrics:
	// mux.Handle("GET /metrics", promhttp.Handler())

	handler := observabilityMiddleware(logger, metrics, mux)

	logger.Info("server starting", slog.Int("port", 8080))
	http.ListenAndServe(":8080", handler)
}
```

### Production Prometheus Setup

Here is what a production Prometheus integration looks like with the actual client library:

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
	"time"

	// These would be real imports in a go.mod project:
	// "github.com/prometheus/client_golang/prometheus"
	// "github.com/prometheus/client_golang/prometheus/promauto"
	// "github.com/prometheus/client_golang/prometheus/promhttp"
)

// In production, define your metrics:
//
// var (
//     httpRequestsTotal = promauto.NewCounterVec(
//         prometheus.CounterOpts{
//             Namespace: "krafty",
//             Subsystem: "http",
//             Name:      "requests_total",
//             Help:      "Total number of HTTP requests by method, path, and status code.",
//         },
//         []string{"method", "path", "status_code"},
//     )
//
//     httpRequestDuration = promauto.NewHistogramVec(
//         prometheus.HistogramOpts{
//             Namespace: "krafty",
//             Subsystem: "http",
//             Name:      "request_duration_seconds",
//             Help:      "HTTP request duration in seconds.",
//             Buckets:   []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
//         },
//         []string{"method", "path"},
//     )
//
//     httpResponseSize = promauto.NewHistogramVec(
//         prometheus.HistogramOpts{
//             Namespace: "krafty",
//             Subsystem: "http",
//             Name:      "response_size_bytes",
//             Help:      "HTTP response size in bytes.",
//             Buckets:   prometheus.ExponentialBuckets(100, 10, 7), // 100, 1K, 10K, 100K, 1M, 10M, 100M
//         },
//         []string{"method", "path"},
//     )
//
//     httpRequestsInFlight = promauto.NewGauge(
//         prometheus.GaugeOpts{
//             Namespace: "krafty",
//             Subsystem: "http",
//             Name:      "requests_in_flight",
//             Help:      "Number of HTTP requests currently being processed.",
//         },
//     )
// )

// The Prometheus metrics endpoint produces output like:
//
// # HELP krafty_http_requests_total Total number of HTTP requests.
// # TYPE krafty_http_requests_total counter
// krafty_http_requests_total{method="GET",path="/api/orders",status_code="200"} 1542
// krafty_http_requests_total{method="POST",path="/api/orders",status_code="201"} 89
// krafty_http_requests_total{method="GET",path="/api/orders",status_code="500"} 3
//
// # HELP krafty_http_request_duration_seconds HTTP request duration in seconds.
// # TYPE krafty_http_request_duration_seconds histogram
// krafty_http_request_duration_seconds_bucket{method="GET",path="/api/orders",le="0.01"} 1200
// krafty_http_request_duration_seconds_bucket{method="GET",path="/api/orders",le="0.05"} 1480
// krafty_http_request_duration_seconds_bucket{method="GET",path="/api/orders",le="0.1"} 1520
// krafty_http_request_duration_seconds_bucket{method="GET",path="/api/orders",le="+Inf"} 1542

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	logger.Info("prometheus metrics example",
		slog.String("note", "use 'go get github.com/prometheus/client_golang' to add prometheus"),
		slog.Duration("scrape_interval", 15*time.Second),
	)

	fmt.Println("Key Prometheus metric types:")
	fmt.Println("  Counter  - monotonically increasing (total requests, total errors)")
	fmt.Println("  Gauge    - goes up and down (requests in flight, temperature)")
	fmt.Println("  Histogram - distribution (request duration, response size)")
	fmt.Println("  Summary  - like histogram but calculates quantiles client-side")
}
```

### OpenTelemetry Distributed Tracing

OpenTelemetry (OTel) is the standard for distributed tracing. A trace follows a request across multiple services:

```go
package main

import (
	"fmt"
	"log/slog"
	"os"
	"time"
)

// This is a conceptual overview. Real OTel setup requires:
//   go get go.opentelemetry.io/otel
//   go get go.opentelemetry.io/otel/sdk
//   go get go.opentelemetry.io/otel/exporters/otlp/otlptrace

// A distributed trace looks like this:
//
// Trace ID: abc123 (shared across ALL services for this request)
//
// ┌──────────────────────────────────────────────────────────────────┐
// │ Span: API Gateway (50ms)                                        │
// │  trace_id=abc123 span_id=001                                    │
// │ ┌────────────────────────────────────────────────────────────┐   │
// │ │ Span: Order Service (40ms)                                │   │
// │ │  trace_id=abc123 span_id=002 parent_span_id=001           │   │
// │ │ ┌──────────────────────┐ ┌─────────────────────────────┐  │   │
// │ │ │ Span: DB Query (5ms) │ │ Span: Inventory Check (20ms)│  │   │
// │ │ │  span_id=003         │ │  span_id=004                │  │   │
// │ │ │  parent=002          │ │  parent=002                 │  │   │
// │ │ └──────────────────────┘ └─────────────────────────────┘  │   │
// │ └────────────────────────────────────────────────────────────┘   │
// └──────────────────────────────────────────────────────────────────┘

// Production OTel setup in Go:
//
// func initTracer() (*trace.TracerProvider, error) {
//     exporter, err := otlptrace.New(
//         context.Background(),
//         otlptracegrpc.NewClient(
//             otlptracegrpc.WithEndpoint("otel-collector:4317"),
//             otlptracegrpc.WithInsecure(),
//         ),
//     )
//     if err != nil {
//         return nil, err
//     }
//
//     tp := trace.NewTracerProvider(
//         trace.WithBatcher(exporter),
//         trace.WithResource(resource.NewWithAttributes(
//             semconv.SchemaURL,
//             semconv.ServiceNameKey.String("krafty-core"),
//             semconv.ServiceVersionKey.String("1.2.3"),
//             attribute.String("environment", "production"),
//         )),
//     )
//     otel.SetTracerProvider(tp)
//     return tp, nil
// }
//
// // In a handler:
// func handleOrder(w http.ResponseWriter, r *http.Request) {
//     ctx, span := otel.Tracer("order-handler").Start(r.Context(), "HandleOrder")
//     defer span.End()
//
//     span.SetAttributes(
//         attribute.String("order.id", orderID),
//         attribute.String("user.id", userID),
//     )
//
//     // The span is automatically propagated through context
//     order, err := orderService.CreateOrder(ctx, orderID)
//     if err != nil {
//         span.RecordError(err)
//         span.SetStatus(codes.Error, err.Error())
//         // ...
//     }
// }

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	logger.Info("OpenTelemetry tracing overview",
		slog.String("note", "use 'go get go.opentelemetry.io/otel' for real tracing"),
	)

	fmt.Println("\nKey OpenTelemetry concepts:")
	fmt.Println("  Trace    - the full journey of a request across services")
	fmt.Println("  Span     - a single operation within a trace")
	fmt.Println("  Context  - carries trace/span IDs through function calls")
	fmt.Println("  Exporter - sends traces to a backend (Jaeger, Tempo, Datadog)")
	fmt.Println("  Sampler  - decides which traces to record (not all, in production)")

	fmt.Println("\nCommon exporters:")
	fmt.Println("  Jaeger  - open source, self-hosted, good for getting started")
	fmt.Println("  Tempo   - Grafana's trace backend, pairs with Loki and Prometheus")
	fmt.Println("  Datadog - commercial, full observability platform")
	fmt.Println("  X-Ray   - AWS native tracing")

	_ = time.Second // suppress unused import
	_ = logger
}
```

### Connecting Logs and Traces

The ultimate goal is to correlate logs with traces. Include the trace ID in your log output:

```go
package main

import (
	"context"
	"log/slog"
	"os"

	// In production:
	// "go.opentelemetry.io/otel/trace"
)

// traceHandler wraps an slog.Handler to add trace context to log records.
type traceHandler struct {
	slog.Handler
}

func (h *traceHandler) Handle(ctx context.Context, r slog.Record) error {
	// In production with OpenTelemetry:
	//
	// spanCtx := trace.SpanContextFromContext(ctx)
	// if spanCtx.HasTraceID() {
	//     r.AddAttrs(
	//         slog.String("trace_id", spanCtx.TraceID().String()),
	//         slog.String("span_id", spanCtx.SpanID().String()),
	//     )
	// }

	// For demonstration, simulate trace context
	if traceID, ok := ctx.Value("trace_id").(string); ok {
		r.AddAttrs(slog.String("trace_id", traceID))
	}
	if spanID, ok := ctx.Value("span_id").(string); ok {
		r.AddAttrs(slog.String("span_id", spanID))
	}

	return h.Handler.Handle(ctx, r)
}

func (h *traceHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &traceHandler{Handler: h.Handler.WithAttrs(attrs)}
}

func (h *traceHandler) WithGroup(name string) slog.Handler {
	return &traceHandler{Handler: h.Handler.WithGroup(name)}
}

func main() {
	baseHandler := slog.NewJSONHandler(os.Stdout, nil)
	logger := slog.New(&traceHandler{Handler: baseHandler})

	// Simulate a traced request
	ctx := context.Background()
	ctx = context.WithValue(ctx, "trace_id", "abc123def456789")
	ctx = context.WithValue(ctx, "span_id", "span_001")

	logger.InfoContext(ctx, "processing order",
		slog.String("order_id", "ord_789"),
	)

	// Output includes trace_id and span_id automatically:
	// {"time":"...","level":"INFO","msg":"processing order","order_id":"ord_789","trace_id":"abc123def456789","span_id":"span_001"}

	// Now in Grafana/Jaeger, you can click a trace ID in a log line
	// and jump directly to the distributed trace visualization.
}
```

### Node.js Comparison

```javascript
// Node.js OpenTelemetry setup
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4317',
  }),
  serviceName: 'order-service',
});
sdk.start();

// In Express with pino:
const { trace } = require('@opentelemetry/api');

app.use((req, res, next) => {
  const span = trace.getActiveSpan();
  if (span) {
    const ctx = span.spanContext();
    req.log = logger.child({
      trace_id: ctx.traceId,
      span_id: ctx.spanId,
    });
  }
  next();
});
```

The concepts are identical in both ecosystems. OpenTelemetry provides SDKs for both Go and Node.js with nearly the same configuration patterns.

---

## 15. Real-World Example: Complete Logging Stack

This section brings together every pattern from the chapter into a single, production-ready logging middleware. This is the kind of code you would find in a project like krafty-core.

### Complete Production Middleware

```go
package main

import (
	"bufio"
	"context"
	"crypto/rand"
	"encoding/json"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"net/url"
	"os"
	"strings"
	"time"
)

// ============================================================
// Context Keys
// ============================================================

type contextKey string

const (
	RequestIDKey contextKey = "request_id"
	LoggerKey    contextKey = "logger"
)

// ============================================================
// Request ID Generation & Validation
// ============================================================

// generateRequestID creates a cryptographically random request ID.
func generateRequestID() string {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		panic(fmt.Sprintf("crypto/rand failed: %v", err))
	}
	return fmt.Sprintf("%x", b)
}

// isValidRequestID ensures a client-provided request ID is safe to use.
// Only allows alphanumeric, hyphens, and underscores.
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

// ============================================================
// Query Parameter Redaction
// ============================================================

var sensitiveParams = []string{
	"token", "api_key", "apikey", "password", "secret",
	"auth", "credential", "key", "session", "jwt",
	"access_token", "refresh_token",
}

// redactQuery masks sensitive values in a query string.
func redactQuery(rawQuery string) string {
	if rawQuery == "" {
		return ""
	}
	values, err := url.ParseQuery(rawQuery)
	if err != nil {
		return "[REDACTED_UNPARSEABLE]"
	}
	for key := range values {
		lowerKey := strings.ToLower(key)
		for _, sensitive := range sensitiveParams {
			if strings.Contains(lowerKey, sensitive) {
				values.Set(key, "[REDACTED]")
				break
			}
		}
	}
	return values.Encode()
}

// ============================================================
// Response Writer Wrapper
// ============================================================

// responseWriter captures status code and bytes written.
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
	if rw.headerWritten {
		return
	}
	rw.headerWritten = true
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.headerWritten {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.written += int64(n)
	return n, err
}

func (rw *responseWriter) Flush() {
	if f, ok := rw.ResponseWriter.(http.Flusher); ok {
		f.Flush()
	}
}

func (rw *responseWriter) Hijack() (net.Conn, *bufio.ReadWriter, error) {
	if h, ok := rw.ResponseWriter.(http.Hijacker); ok {
		return h.Hijack()
	}
	return nil, nil, fmt.Errorf("hijack not supported")
}

func (rw *responseWriter) Unwrap() http.ResponseWriter {
	return rw.ResponseWriter
}

// ============================================================
// Log Level Mapping
// ============================================================

// logLevelForStatus returns the appropriate log level for an HTTP status code.
func logLevelForStatus(status int) slog.Level {
	switch {
	case status >= 500:
		return slog.LevelError
	case status >= 400:
		return slog.LevelWarn
	default:
		return slog.LevelInfo
	}
}

// ============================================================
// Health Check Detection
// ============================================================

var healthCheckPaths = map[string]bool{
	"/healthz": true,
	"/readyz":  true,
	"/livez":   true,
	"/health":  true,
	"/ping":    true,
	"/metrics": true,
}

func isHealthCheck(path string) bool {
	return healthCheckPaths[path]
}

// ============================================================
// Context Helpers
// ============================================================

func requestIDFromContext(ctx context.Context) string {
	if id, ok := ctx.Value(RequestIDKey).(string); ok {
		return id
	}
	return ""
}

func loggerFromContext(ctx context.Context) *slog.Logger {
	if l, ok := ctx.Value(LoggerKey).(*slog.Logger); ok {
		return l
	}
	return slog.Default()
}

// ============================================================
// The Complete Middleware
// ============================================================

// RequestLoggingMiddleware is the production-grade request logging middleware.
// It combines:
//   - Request ID generation/validation
//   - Log injection prevention
//   - Response writer wrapping for status/bytes capture
//   - Query parameter redaction
//   - Status-code-based log levels
//   - Health check suppression
//   - Request-scoped logger in context
func RequestLoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// 1. Request ID: validate or generate
			requestID := r.Header.Get("X-Request-ID")
			if !isValidRequestID(requestID) {
				requestID = generateRequestID()
			}

			// 2. Add request ID to response headers
			w.Header().Set("X-Request-ID", requestID)

			// 3. Create request-scoped logger with persistent attributes
			reqLogger := logger.With(
				slog.String("request_id", requestID),
			)

			// 4. Store request ID and logger in context
			ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
			ctx = context.WithValue(ctx, LoggerKey, reqLogger)

			// 5. Wrap response writer to capture status and bytes
			wrapped := newResponseWriter(w)

			// 6. Call the next handler
			next.ServeHTTP(wrapped, r.WithContext(ctx))

			// 7. Calculate duration
			duration := time.Since(start)

			// 8. Health check: log at DEBUG (suppressed in production)
			if isHealthCheck(r.URL.Path) {
				reqLogger.LogAttrs(ctx, slog.LevelDebug, "health check",
					slog.String("path", r.URL.Path),
					slog.Int("status", wrapped.statusCode),
					slog.Duration("duration", duration),
				)
				return
			}

			// 9. Build redacted path
			logPath := r.URL.Path
			if r.URL.RawQuery != "" {
				logPath = r.URL.Path + "?" + redactQuery(r.URL.RawQuery)
			}

			// 10. Determine log level from status code
			level := logLevelForStatus(wrapped.statusCode)

			// 11. Log the request with all attributes
			reqLogger.LogAttrs(ctx, level, "request completed",
				slog.String("method", r.Method),
				slog.String("path", logPath),
				slog.Int("status", wrapped.statusCode),
				slog.Duration("duration", duration),
				slog.Int64("bytes", wrapped.written),
				slog.String("remote_addr", r.RemoteAddr),
				slog.String("user_agent", r.UserAgent()),
				slog.String("referer", r.Referer()),
				slog.String("protocol", r.Proto),
			)
		})
	}
}

// ============================================================
// Application Setup
// ============================================================

// Config holds application configuration.
type Config struct {
	Port     int
	LogLevel string
	Env      string
}

// setupLogger creates the application logger based on configuration.
func setupLogger(cfg Config) *slog.Logger {
	var level slog.Level
	switch cfg.LogLevel {
	case "debug":
		level = slog.LevelDebug
	case "warn":
		level = slog.LevelWarn
	case "error":
		level = slog.LevelError
	default:
		level = slog.LevelInfo
	}

	opts := &slog.HandlerOptions{
		Level: level,
	}

	var handler slog.Handler
	switch cfg.Env {
	case "production", "staging":
		handler = slog.NewJSONHandler(os.Stdout, opts)
	default:
		opts.AddSource = true
		handler = slog.NewTextHandler(os.Stdout, opts)
	}

	return slog.New(handler)
}

// ============================================================
// Handlers
// ============================================================

func handleListOrders(w http.ResponseWriter, r *http.Request) {
	logger := loggerFromContext(r.Context())
	logger.InfoContext(r.Context(), "fetching orders from database")

	// Simulate database work
	time.Sleep(10 * time.Millisecond)

	orders := []map[string]interface{}{
		{"id": "ord_001", "item": "Widget", "amount": 29.99},
		{"id": "ord_002", "item": "Gadget", "amount": 49.99},
	}

	logger.InfoContext(r.Context(), "orders fetched",
		slog.Int("count", len(orders)),
	)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{"orders": orders})
}

func handleCreateOrder(w http.ResponseWriter, r *http.Request) {
	logger := loggerFromContext(r.Context())

	var req struct {
		Item   string  `json:"item"`
		Amount float64 `json:"amount"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		logger.WarnContext(r.Context(), "invalid request body",
			slog.String("error", err.Error()),
		)
		http.Error(w, `{"error":"invalid request body"}`, http.StatusBadRequest)
		return
	}

	if req.Item == "" {
		logger.WarnContext(r.Context(), "missing required field",
			slog.String("field", "item"),
		)
		http.Error(w, `{"error":"item is required"}`, http.StatusBadRequest)
		return
	}

	orderID := fmt.Sprintf("ord_%d", time.Now().UnixNano())
	logger.InfoContext(r.Context(), "order created",
		slog.String("order_id", orderID),
		slog.String("item", req.Item),
		slog.Float64("amount", req.Amount),
	)

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]string{"order_id": orderID})
}

func handleHealthz(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "ok")
}

func handleReadyz(w http.ResponseWriter, r *http.Request) {
	// In production, check database connectivity, cache, etc.
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "ok")
}

func handleServerError(w http.ResponseWriter, r *http.Request) {
	logger := loggerFromContext(r.Context())
	logger.ErrorContext(r.Context(), "simulated database failure",
		slog.String("error", "connection refused"),
		slog.String("db_host", "db.production.internal"),
	)
	http.Error(w, `{"error":"internal server error"}`, http.StatusInternalServerError)
}

// ============================================================
// Main
// ============================================================

func main() {
	cfg := Config{
		Port:     8080,
		LogLevel: "info",
		Env:      "production",
	}

	// Override from environment
	if env := os.Getenv("APP_ENV"); env != "" {
		cfg.Env = env
	}
	if level := os.Getenv("LOG_LEVEL"); level != "" {
		cfg.LogLevel = level
	}

	logger := setupLogger(cfg)

	// Register routes
	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/orders", handleListOrders)
	mux.HandleFunc("POST /api/orders", handleCreateOrder)
	mux.HandleFunc("GET /healthz", handleHealthz)
	mux.HandleFunc("GET /readyz", handleReadyz)
	mux.HandleFunc("GET /api/error", handleServerError)

	// Apply middleware
	handler := RequestLoggingMiddleware(logger)(mux)

	// Start server
	logger.Info("server starting",
		slog.Int("port", cfg.Port),
		slog.String("environment", cfg.Env),
		slog.String("log_level", cfg.LogLevel),
	)

	addr := fmt.Sprintf(":%d", cfg.Port)
	if err := http.ListenAndServe(addr, handler); err != nil {
		logger.Error("server failed to start", slog.String("error", err.Error()))
		os.Exit(1)
	}
}
```

### Testing the Complete Stack

Run the server and test with curl:

```bash
# Start the server
APP_ENV=production LOG_LEVEL=info go run main.go

# Test successful request
curl http://localhost:8080/api/orders
# Log: {"level":"INFO","msg":"request completed","request_id":"a1b2c3...","method":"GET","path":"/api/orders","status":200,...}

# Test with custom request ID
curl -H "X-Request-ID: my-trace-123" http://localhost:8080/api/orders
# Log: {"level":"INFO","msg":"request completed","request_id":"my-trace-123",...}

# Test with sensitive query params (redacted in logs)
curl "http://localhost:8080/api/orders?token=secret123&page=1"
# Log: {"level":"INFO","msg":"request completed",...,"path":"/api/orders?page=1&token=%5BREDACTED%5D",...}

# Test 400 Bad Request (logged at WARN)
curl -X POST http://localhost:8080/api/orders -d '{}'
# Log: {"level":"WARN","msg":"request completed",...,"status":400,...}

# Test 500 Internal Server Error (logged at ERROR)
curl http://localhost:8080/api/error
# Log: {"level":"ERROR","msg":"request completed",...,"status":500,...}

# Test health check (suppressed at INFO level)
curl http://localhost:8080/healthz
# No log at INFO level (logged at DEBUG, which is suppressed)

# Test invalid request ID (rejected, new one generated)
curl -H "X-Request-ID: evil<script>alert(1)</script>" http://localhost:8080/api/orders
# Log: request_id is a newly generated safe ID, not the malicious one

# Check response headers for request ID
curl -v http://localhost:8080/api/orders 2>&1 | grep X-Request-ID
# < X-Request-ID: a1b2c3d4e5f6...
```

### Production Log Output

Here is what the logs look like in production, piped through `jq` for readability:

```json
{
  "time": "2025-03-15T14:32:01.123Z",
  "level": "INFO",
  "msg": "server starting",
  "port": 8080,
  "environment": "production",
  "log_level": "info"
}
{
  "time": "2025-03-15T14:32:01.456Z",
  "level": "INFO",
  "msg": "request completed",
  "request_id": "a1b2c3d4e5f6789012345678abcdef01",
  "method": "GET",
  "path": "/api/orders",
  "status": 200,
  "duration": 12543000,
  "bytes": 89,
  "remote_addr": "192.168.1.100:54321",
  "user_agent": "curl/7.88.0",
  "referer": "",
  "protocol": "HTTP/1.1"
}
{
  "time": "2025-03-15T14:32:02.789Z",
  "level": "WARN",
  "msg": "request completed",
  "request_id": "fedcba98765432100123456789abcdef",
  "method": "POST",
  "path": "/api/orders",
  "status": 400,
  "duration": 1234000,
  "bytes": 42,
  "remote_addr": "192.168.1.100:54322",
  "user_agent": "curl/7.88.0",
  "referer": "",
  "protocol": "HTTP/1.1"
}
{
  "time": "2025-03-15T14:32:03.012Z",
  "level": "ERROR",
  "msg": "request completed",
  "request_id": "1234567890abcdef1234567890abcdef",
  "method": "GET",
  "path": "/api/error",
  "status": 500,
  "duration": 5678000,
  "bytes": 56,
  "remote_addr": "192.168.1.100:54323",
  "user_agent": "curl/7.88.0",
  "referer": "",
  "protocol": "HTTP/1.1"
}
```

### Full Middleware Comparison: Go vs Node.js/Express

For reference, here is the equivalent middleware in Express with pino:

```javascript
// Node.js/Express equivalent of the complete logging middleware

const pino = require('pino');
const pinoHttp = require('pino-http');
const { v4: uuidv4 } = require('uuid');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // JSON output by default in pino (unlike winston)
});

const sensitiveParams = ['token', 'api_key', 'password', 'secret', 'auth', 'key'];
const healthPaths = new Set(['/healthz', '/readyz', '/livez', '/health', '/metrics']);

function isValidRequestId(id) {
  if (!id || id.length > 128) return false;
  return /^[a-zA-Z0-9_-]+$/.test(id);
}

function redactQuery(url) {
  const u = new URL(url, 'http://localhost');
  for (const key of u.searchParams.keys()) {
    const lower = key.toLowerCase();
    if (sensitiveParams.some(s => lower.includes(s))) {
      u.searchParams.set(key, '[REDACTED]');
    }
  }
  return u.pathname + (u.search ? u.search : '');
}

// The middleware
app.use((req, res, next) => {
  const start = Date.now();

  // Request ID
  let requestId = req.headers['x-request-id'];
  if (!isValidRequestId(requestId)) {
    requestId = uuidv4();
  }
  res.setHeader('X-Request-ID', requestId);

  // Child logger with request ID
  req.log = logger.child({ request_id: requestId });

  // Capture response finish
  res.on('finish', () => {
    if (healthPaths.has(req.path)) return; // Suppress health checks

    const duration = Date.now() - start;
    const logData = {
      method: req.method,
      path: redactQuery(req.originalUrl),
      status: res.statusCode,
      duration_ms: duration,
      bytes: res.get('content-length') || 0,
      remote_addr: req.ip,
      user_agent: req.get('user-agent'),
    };

    if (res.statusCode >= 500) {
      req.log.error(logData, 'request completed');
    } else if (res.statusCode >= 400) {
      req.log.warn(logData, 'request completed');
    } else {
      req.log.info(logData, 'request completed');
    }
  });

  next();
});
```

**Key differences:**
- Go requires explicit response writer wrapping; Node.js uses the `finish` event
- Go propagates via `context.Context`; Node.js attaches to `req`
- Go's slog is in the standard library; Node.js requires pino/winston from npm
- Both produce the same structured JSON output
- Both need the same security patterns (request ID validation, query redaction)

---

## 16. Key Takeaways

1. **Use structured logging from day one.** `fmt.Println` and `log.Println` are fine for learning. In production, every log line must be structured JSON. The cost of adding structure is low. The cost of not having it during an incident is enormous.

2. **Go's `log/slog` package is the standard.** Introduced in Go 1.21, it provides a common interface that the entire ecosystem can agree on. Stop debating zap vs zerolog vs logrus. Use `slog`. If you need the raw performance of zap, use a zap handler behind the slog interface.

3. **Use JSON in production, text in development.** Switch handlers based on the `APP_ENV` environment variable. JSON for machines (production). Text for humans (development). Same application code, different output format.

4. **Log levels are a contract, not a suggestion.** ERROR means someone needs to investigate now. WARN means something unexpected that might become a problem. INFO is the heartbeat. DEBUG is for temporary diagnostic detail. If you have ERROR logs that nobody investigates, your levels are wrong.

5. **Use `slog.LevelVar` for runtime level changes.** Being able to enable DEBUG logging in production without restarting the service is one of the most useful debugging tools you can have. Expose it behind an admin endpoint with proper authentication.

6. **Request IDs are non-negotiable.** Every request gets a unique ID. It propagates through context to every log line, every database query, every downstream service call. This is the single most important observability pattern.

7. **Validate request IDs to prevent log injection.** Never blindly trust client-supplied values in log output. Validate format, enforce length limits, restrict to safe characters. The `slog` JSON handler escapes special characters, but defense in depth is the right approach.

8. **Wrap ResponseWriter to capture metrics.** Go's `http.ResponseWriter` does not expose status code or bytes written after the handler completes. Wrapping it is the standard pattern. Always implement `Flush()`, `Hijack()`, and `Unwrap()` to avoid breaking SSE, WebSocket, and streaming handlers.

9. **Redact sensitive data in logs.** Query parameters (`token`, `api_key`, `password`), headers (`Authorization`, `Cookie`), and struct fields (`Password`, `APIKey`) must be redacted before logging. Implement `slog.LogValuer` on your types to make this automatic and impossible to forget.

10. **Map status codes to log levels.** 2xx is INFO, 4xx is WARN, 5xx is ERROR. This separation is essential for meaningful alerting. Alert on 5xx rate (your bug). Monitor 4xx trends (possible API misuse). Never wake someone up for a 404.

11. **Inject loggers as dependencies.** Pass `*slog.Logger` through constructors for long-lived services. Use context-based loggers for request-scoped data. Never rely on a package-level global logger in production code. Dependency injection makes testing trivial.

12. **Use `slog.With` to build child loggers.** Add `component`, `service`, `version`, and `request_id` as persistent attributes. Every log line from that logger automatically includes this context. This is how you make logs filterable and searchable at scale.

13. **Suppress health check logs.** Kubernetes probes, load balancer health checks, and Prometheus scrapes generate enormous log volume with zero diagnostic value. Log them at DEBUG or skip them entirely.

14. **Logging is one of three observability pillars.** Logs (what happened), metrics (what is happening), and traces (what path did it take) work together. Connect them with shared request IDs and trace IDs so you can jump from a log line to a trace to a dashboard.

15. **Use `LogAttrs` in hot paths.** For middleware that runs on every request, `LogAttrs` with typed `slog.Attr` values avoids the allocation overhead of the generic `Info`/`Error` methods. The difference is small per call but adds up at scale.

---

## 17. Practice Exercises

### Exercise 1: Basic Structured Logger

Build a program that:
- Creates a JSON logger with configurable level from `LOG_LEVEL` environment variable
- Logs startup information (version, port, environment)
- Logs 10 simulated requests with method, path, status, and duration
- Uses different log levels based on status code (200=INFO, 404=WARN, 500=ERROR)

Test by running with different `LOG_LEVEL` values and verify that filtering works correctly.

### Exercise 2: Custom Handler

Implement a custom `slog.Handler` that:
- Outputs logs in a custom format: `[LEVEL] TIMESTAMP | message | key1=val1 key2=val2`
- Colors the level based on severity (DEBUG=gray, INFO=green, WARN=yellow, ERROR=red) using ANSI codes
- Respects the `Enabled` method for level filtering
- Supports `WithAttrs` and `WithGroup`
- Write tests that capture output and verify the format

### Exercise 3: Request ID Middleware

Build a complete HTTP server with request ID middleware that:
- Accepts `X-Request-ID` from clients (with validation)
- Generates a new UUID if none provided or if the provided one is invalid
- Returns the request ID in response headers
- Includes the request ID in all log lines for that request
- Propagates the request ID when making HTTP calls to downstream services (simulate with a second handler)

Write a test that sends 100 concurrent requests and verifies each response has a unique request ID.

### Exercise 4: Sensitive Data Redaction

Build a logging system that:
- Implements `slog.LogValuer` for `User`, `CreditCard`, and `APICredentials` types
- Redacts passwords, card numbers (show only last 4 digits), and API secrets
- Redacts sensitive query parameters in URLs
- Redacts sensitive HTTP headers
- Write tests that log these types and verify the output contains `[REDACTED]` and never contains the actual sensitive values

### Exercise 5: Complete Logging Middleware

Build a production-grade HTTP logging middleware that combines all patterns from this chapter:
- Request ID generation and validation
- Response writer wrapping with Flusher/Hijacker support
- Query parameter redaction
- Status-code-based log levels
- Health check suppression
- Duration and bytes tracking
- Request-scoped child logger in context

Test it with a small API (3-4 endpoints) and verify:
- Health checks do not appear in INFO logs
- Sensitive query params are redacted
- 4xx responses are logged at WARN
- 5xx responses are logged at ERROR
- Invalid request IDs are rejected and replaced

### Exercise 6: Dynamic Log Level API

Build an HTTP server with an admin endpoint that:
- Shows the current log level (`GET /admin/log-level`)
- Changes the log level at runtime (`PUT /admin/log-level?level=debug`)
- Requires a simple Bearer token for authentication
- Logs the level change event
- Verify that changing to DEBUG makes debug-level logs appear immediately without restart

### Exercise 7: Log Aggregation Simulation

Build a system that simulates a multi-service architecture:
- Service A (API Gateway) receives requests and forwards to Service B
- Service B (Order Service) processes orders and calls Service C
- Service C (Inventory Service) checks stock
- Each service logs with its own child logger (`component` attribute)
- All services share the same request ID
- Write all logs to a single file in JSON format
- Write a separate tool that reads the log file and:
  - Groups all log lines by request ID
  - Calculates total duration per request
  - Identifies failed requests
  - Produces a summary report

### Exercise 8: Metrics Integration

Extend Exercise 5 to include Prometheus-style metrics:
- Count total requests by method, path, and status code
- Track request duration as a histogram
- Track response size as a histogram
- Track in-flight requests as a gauge
- Expose a `/metrics` endpoint that outputs the metrics in Prometheus text format
- (Bonus) Use the actual `prometheus/client_golang` library instead of a custom implementation

### Exercise 9: Benchmarking Log Performance

Write benchmarks comparing:
- `fmt.Printf` vs `slog.Info` vs `slog.LogAttrs`
- `TextHandler` vs `JSONHandler`
- Logger with 0 attributes vs 5 attributes vs 10 attributes
- Logging to `os.Stdout` vs `io.Discard` (to isolate serialization cost)
- `slog.String("key", value)` vs `slog.Any("key", value)`

Use `testing.B` and report allocations. Explain the results.

### Exercise 10: OpenTelemetry Integration

Build a traced HTTP server that:
- Initializes an OpenTelemetry tracer (export to console or OTLP)
- Creates a span for each HTTP request
- Adds request attributes to spans (method, path, status)
- Creates child spans for simulated database queries
- Writes a custom `slog.Handler` that adds `trace_id` and `span_id` to every log line
- Correlates logs and traces through shared IDs

This requires installing OpenTelemetry packages:
```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/stdout/stdouttrace
```
