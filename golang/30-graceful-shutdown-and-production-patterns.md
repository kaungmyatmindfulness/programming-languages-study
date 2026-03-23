# Chapter 30: Graceful Shutdown & Production Patterns

Building software that works on your laptop is one thing. Building software that survives in production -- handling traffic spikes, rolling deployments, container restarts, and infrastructure failures without losing a single request -- is an entirely different discipline. This chapter bridges that gap.

We are now in the "real-world" section of this guide. Chapters 1-22 covered Go fundamentals. Starting here, every chapter draws directly from production patterns found in real Go API projects like "krafty-core." This is not theory. Every pattern in this chapter exists because skipping it caused an outage, lost data, or woke someone up at 3 AM.

Graceful shutdown is the single most important operational concern for any long-running Go service. Get it wrong, and every deployment becomes a potential incident. Get it right, and you can deploy a hundred times a day without your users noticing.

---

## Prerequisites

Before reading this chapter, you should be comfortable with:

- **Chapter 10:** Goroutines and Concurrency Basics
- **Chapter 11:** Channels and Select
- **Chapter 12:** JSON, HTTP, and REST APIs
- **Chapter 16:** The Context Package
- **Chapter 17:** Advanced Concurrency

You should also understand basic concepts of:

- HTTP servers and request handling
- Database connection pools
- Environment variables and configuration
- Docker containers (at a conceptual level)

---

## Table of Contents

1. [Why Graceful Shutdown Matters](#1-why-graceful-shutdown-matters)
2. [OS Signals in Go](#2-os-signals-in-go)
3. [http.Server Shutdown](#3-httpserver-shutdown)
4. [Background Task Lifecycle](#4-background-task-lifecycle)
5. [Health Check Endpoints](#5-health-check-endpoints)
6. [Server Timeouts](#6-server-timeouts)
7. [Connection Draining](#7-connection-draining)
8. [Database Connection Cleanup](#8-database-connection-cleanup)
9. [Startup Sequence](#9-startup-sequence)
10. [Structured Main Function](#10-structured-main-function)
11. [Docker Considerations](#11-docker-considerations)
12. [Kubernetes Readiness](#12-kubernetes-readiness)
13. [Error Channels Pattern](#13-error-channels-pattern)
14. [Restart Resilience](#14-restart-resilience)
15. [Real-World Example: Complete main.go](#15-real-world-example-complete-maingo)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. Why Graceful Shutdown Matters

### The Problem

Your Go API server is handling 500 requests per second. You deploy a new version. The orchestrator sends a kill signal to the old process. What happens to the 50 requests that are currently in-flight?

Without graceful shutdown: they die. The client gets a connection reset. If any of those requests were writing to a database, you might have half-written records. If any were processing payments, you might have charged a customer without recording the transaction.

With graceful shutdown: the server stops accepting new connections, waits for all in-flight requests to complete (up to a timeout), cleanly closes database connections, flushes logs, and then exits. The client never knows a deployment happened.

### What Can Go Wrong Without Graceful Shutdown

```
+---------------------------------------------------------------------+
| Scenario                    | Without Graceful Shutdown              |
+---------------------------------------------------------------------+
| HTTP request mid-response   | Connection reset, client error         |
| Database transaction open   | Transaction rolled back by DB timeout  |
| Background job processing   | Job lost, needs retry mechanism        |
| File being written          | Corrupt/partial file on disk           |
| Message being consumed      | Message lost or double-processed       |
| WebSocket connection open   | Client disconnects without close frame |
| Cache warming in progress   | Cache starts cold on next boot         |
| Metrics being flushed       | Last batch of metrics lost             |
+---------------------------------------------------------------------+
```

### In Container Orchestration

This problem is magnified in Kubernetes, Docker Swarm, ECS, and similar platforms. These orchestrators routinely stop and start containers:

- **Rolling deployments:** Old pods are terminated as new pods come up
- **Horizontal autoscaling:** Pods are added and removed based on load
- **Node maintenance:** All pods on a node are evicted
- **Spot instances:** Cloud provider reclaims the instance with 2 minutes notice
- **Health check failures:** Unhealthy pods are killed and replaced

In Kubernetes specifically, the shutdown sequence is:

```
1. Pod marked for termination
2. Pod removed from Service endpoints (no new traffic)
3. PreStop hook runs (if configured)
4. SIGTERM sent to PID 1 in the container
5. Grace period countdown starts (default: 30 seconds)
6. If process still running after grace period: SIGKILL
```

Your application must handle step 4 correctly. Everything between SIGTERM and process exit is your responsibility.

### Node.js Comparison

In Node.js/Express, graceful shutdown is not built into the framework. You have to wire it up manually, and there are footguns everywhere:

```javascript
// Node.js -- manual graceful shutdown (common but incomplete)
const server = app.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  server.close(() => {
    console.log('HTTP server closed');
    // Close database connections, etc.
    process.exit(0);
  });

  // Force shutdown after timeout
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 10000);
});
```

The Node.js approach has problems: `server.close()` only stops accepting new connections -- it does not actively close idle keep-alive connections. You need additional libraries like `http-terminator` or `stoppable` to handle this correctly. Go's `http.Server.Shutdown()` handles all of this out of the box.

### The Go Advantage

Go's standard library provides `http.Server.Shutdown()` which:

1. Closes all listeners (stops accepting new connections)
2. Closes all idle connections immediately
3. Waits for all active connections to become idle (request completes)
4. Returns when all connections are closed or the context deadline is reached

This is production-ready behavior built into the standard library. No third-party packages needed.

---

## 2. OS Signals in Go

### What Are Signals?

Signals are asynchronous notifications sent to a process by the operating system or by other processes. They are the fundamental mechanism by which a container orchestrator tells your application "please shut down."

The signals you care about for graceful shutdown:

```
+----------+--------+---------------------------------------------------+
| Signal   | Number | Purpose                                           |
+----------+--------+---------------------------------------------------+
| SIGINT   | 2      | Interrupt -- sent when you press Ctrl+C            |
| SIGTERM  | 15     | Terminate -- sent by Kubernetes, Docker, systemd   |
| SIGHUP   | 1      | Hang up -- traditionally "reload configuration"    |
| SIGKILL  | 9      | Kill -- cannot be caught, immediate termination    |
| SIGUSR1  | 10     | User-defined -- used for custom operations         |
| SIGUSR2  | 12     | User-defined -- used for custom operations         |
+----------+--------+---------------------------------------------------+
```

**SIGKILL cannot be caught or handled.** This is why you must respond to SIGTERM quickly -- if you do not exit within the grace period, the orchestrator sends SIGKILL and your process dies immediately.

### The signal.Notify Pattern

Go provides the `os/signal` package for catching signals. The pattern is always the same: create a channel, register it for the signals you care about, and read from it.

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	// Create a buffered channel (buffer size 1 is important!)
	quit := make(chan os.Signal, 1)

	// Register to receive SIGINT and SIGTERM
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	fmt.Println("Server is running. Press Ctrl+C to stop.")

	// Block until a signal is received
	sig := <-quit
	fmt.Printf("\nReceived signal: %s\n", sig)
	fmt.Println("Performing cleanup...")
	fmt.Println("Server stopped.")
}
```

### WHY the Channel Must Be Buffered

This is a subtle but critical detail. The `signal.Notify` function does **not** block when sending to the channel. If the channel has no buffer and nobody is reading from it at the exact moment the signal arrives, the signal is dropped. A buffer of 1 ensures the signal is stored even if your goroutine is not currently blocking on the receive.

```go
// WRONG: unbuffered channel -- signal might be dropped
quit := make(chan os.Signal)
signal.Notify(quit, syscall.SIGTERM)

// CORRECT: buffered channel -- signal is stored until consumed
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM)
```

The Go documentation explicitly states: "Package signal will not block sending to c: the caller must ensure that c has sufficient buffer space to keep up with the expected signal rate."

### Handling Multiple Signals

You can register different channels for different signals, or handle them all in one place:

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)

	fmt.Println("Waiting for signals...")

	for {
		sig := <-sigs
		switch sig {
		case syscall.SIGHUP:
			fmt.Println("Received SIGHUP -- reloading configuration")
			// reloadConfig()
		case syscall.SIGINT, syscall.SIGTERM:
			fmt.Printf("Received %s -- shutting down\n", sig)
			// performGracefulShutdown()
			return
		}
	}
}
```

### signal.NotifyContext (Go 1.16+)

Go 1.16 introduced `signal.NotifyContext`, which creates a context that is cancelled when a signal is received. This is often cleaner than managing a raw channel:

```go
package main

import (
	"context"
	"fmt"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// Create a context that is cancelled on SIGINT or SIGTERM
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	fmt.Println("Server running. Press Ctrl+C to stop.")

	// Block until the context is cancelled
	<-ctx.Done()

	fmt.Printf("Received signal: %s\n", ctx.Err())
	fmt.Println("Starting graceful shutdown...")

	// Create a new context for shutdown operations
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Use shutdownCtx for cleanup operations
	_ = shutdownCtx // placeholder
	fmt.Println("Shutdown complete.")
}
```

### Double-Signal Pattern (Force Quit)

A common production pattern: the first SIGTERM triggers graceful shutdown, but if the operator sends a second signal, force-quit immediately. This is useful when debugging a stuck shutdown.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	fmt.Println("Server running...")

	// Wait for first signal
	<-ctx.Done()
	stop() // Deregister so the next signal uses default behavior (terminate)
	fmt.Println("Shutting down gracefully... (press Ctrl+C again to force)")

	// Simulate slow shutdown
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	done := make(chan struct{})
	go func() {
		// Simulate cleanup work
		time.Sleep(5 * time.Second)
		close(done)
	}()

	select {
	case <-done:
		fmt.Println("Graceful shutdown complete.")
	case <-shutdownCtx.Done():
		fmt.Println("Shutdown timed out.")
		os.Exit(1)
	}
}
```

After calling `stop()`, subsequent SIGINT/SIGTERM signals are handled by Go's default behavior, which terminates the process. This gives the operator an escape hatch.

### Node.js Comparison

```javascript
// Node.js signal handling
process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  gracefulShutdown();
});

process.on('SIGINT', () => {
  console.log('SIGINT received');
  gracefulShutdown();
});

// Node.js has no concept of buffered channels.
// If your handler throws an exception, the process crashes.
// If you register the handler after the signal arrives, you miss it.
```

Go's channel-based approach is safer: the signal is stored in the channel buffer regardless of what your code is doing at the moment. You cannot "miss" a signal.

---

## 3. http.Server Shutdown

### The Shutdown Method

Go's `http.Server` has a `Shutdown(ctx)` method that gracefully stops the server. This is the cornerstone of production Go services.

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
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		// Simulate a request that takes 3 seconds
		time.Sleep(3 * time.Second)
		fmt.Fprintln(w, "Hello, World!")
	})

	srv := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start server in a goroutine
	errChan := make(chan error, 1)
	go func() {
		logger.Info("server starting", slog.String("addr", srv.Addr))
		errChan <- srv.ListenAndServe()
	}()

	// Setup signal handling
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	// Wait for signal or server error
	select {
	case <-quit:
		logger.Info("received shutdown signal")
	case err := <-errChan:
		// ListenAndServe returned an error that is NOT ErrServerClosed
		if err != http.ErrServerClosed {
			logger.Error("server failed to start", slog.String("error", err.Error()))
			os.Exit(1)
		}
	}

	// Start graceful shutdown
	logger.Info("shutting down server...")

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("server forced to shutdown", slog.String("error", err.Error()))
		os.Exit(1)
	}

	logger.Info("server stopped gracefully")
}
```

### What Shutdown Actually Does (Step by Step)

When you call `srv.Shutdown(ctx)`:

```
1. Closes all open listeners (the TCP socket stops accepting new connections)
2. Closes all idle connections (keep-alive connections with no active request)
3. Waits indefinitely for active connections to become idle
   (i.e., waits for in-flight requests to complete)
4. Returns nil when all connections are closed
5. Returns ctx.Err() if the context deadline/cancellation fires first
```

This means:

- New clients connecting get "connection refused"
- Clients with in-flight requests get to finish (up to the timeout)
- Idle keep-alive connections are closed immediately
- The Shutdown method blocks until everything is done or the context expires

### The ErrServerClosed Detail

After `Shutdown` is called, `ListenAndServe` returns `http.ErrServerClosed`. This is **not an error** -- it is the expected behavior. You must check for it:

```go
go func() {
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		logger.Error("server error", slog.String("error", err.Error()))
		os.Exit(1)
	}
}()
```

If you do not check for `ErrServerClosed`, your error logging will fire on every clean shutdown, which creates noise in your logs and can confuse alerting systems.

### Production Pattern: The select Statement

The `select` statement waiting on both the signal channel and the error channel is the standard production pattern. Here is why both branches are necessary:

```go
select {
case <-quit:
    // Operator or orchestrator asked us to stop.
    // This is the normal shutdown path.
    logger.Info("shutting down server...")
case err := <-errChan:
    // The server itself failed (e.g., port already in use).
    // We cannot continue -- log and exit.
    logger.Error("server failed", slog.String("error", err.Error()))
    os.Exit(1)
}
```

Without the error channel branch, if the server fails to bind to the port, your main goroutine blocks forever on `<-quit`, and you never know the server is not running.

### Shutdown with Multiple Servers

In production, you might run multiple servers (e.g., an HTTP API server and a metrics server):

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// API server
	apiMux := http.NewServeMux()
	apiMux.HandleFunc("GET /api/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, `{"status":"healthy"}`)
	})

	apiServer := &http.Server{
		Addr:         ":8080",
		Handler:      apiMux,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Metrics server (separate port for Prometheus scraping)
	metricsMux := http.NewServeMux()
	metricsMux.HandleFunc("GET /metrics", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "# HELP requests_total Total requests")
	})

	metricsServer := &http.Server{
		Addr:         ":9090",
		Handler:      metricsMux,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 5 * time.Second,
	}

	// Start both servers
	go func() {
		logger.Info("API server starting", slog.String("addr", apiServer.Addr))
		if err := apiServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("API server failed", slog.String("error", err.Error()))
			os.Exit(1)
		}
	}()

	go func() {
		logger.Info("metrics server starting", slog.String("addr", metricsServer.Addr))
		if err := metricsServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("metrics server failed", slog.String("error", err.Error()))
			os.Exit(1)
		}
	}()

	// Wait for shutdown signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("shutting down all servers...")

	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	// Shutdown both servers concurrently
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		if err := apiServer.Shutdown(ctx); err != nil {
			logger.Error("API server shutdown error", slog.String("error", err.Error()))
		}
		logger.Info("API server stopped")
	}()

	go func() {
		defer wg.Done()
		if err := metricsServer.Shutdown(ctx); err != nil {
			logger.Error("metrics server shutdown error", slog.String("error", err.Error()))
		}
		logger.Info("metrics server stopped")
	}()

	wg.Wait()
	logger.Info("all servers stopped gracefully")
}
```

### Node.js Comparison

```javascript
// Node.js Express -- no built-in graceful shutdown
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  setTimeout(() => res.send('Hello'), 3000);
});

const server = app.listen(3000);

process.on('SIGTERM', () => {
  // server.close() stops new connections but does NOT
  // close keep-alive connections. You need a library.
  server.close(() => {
    process.exit(0);
  });

  // Must forcibly close idle connections yourself
  // or use a library like 'http-terminator'
});
```

Go's `Shutdown` handles keep-alive connections correctly. In Node.js, `server.close()` will hang forever if any client has an open keep-alive connection, because `close()` only stops the listener -- it does not close idle connections. This is a notorious footgun in Node.js production deployments.

---

## 4. Background Task Lifecycle

### The Problem

Production services do not just handle HTTP requests. They run background tasks: cleaning up expired tokens, sending batch emails, syncing data, refreshing caches. These tasks run in goroutines, and if you do not manage their lifecycle, they will either:

1. Keep running after the server shuts down (leaking goroutines)
2. Panic with "database connection closed" because the database was cleaned up before the goroutine stopped
3. Leave work half-done

### Context-Based Goroutine Lifecycle

The pattern is: create a background context, pass it to all background goroutines, cancel the context during shutdown.

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Create a cancellable context for all background tasks
	bgCtx, bgCancel := context.WithCancel(context.Background())
	defer bgCancel()

	// WaitGroup to track background goroutines
	var bgWg sync.WaitGroup

	// Background task 1: Token cleanup (runs every hour)
	bgWg.Add(1)
	go func() {
		defer bgWg.Done()
		ticker := time.NewTicker(1 * time.Hour)
		defer ticker.Stop()

		logger.Info("token cleanup task started")
		for {
			select {
			case <-bgCtx.Done():
				logger.Info("token cleanup task stopping")
				return
			case <-ticker.C:
				logger.Info("running token cleanup")
				// deleted, err := tokenRepo.DeleteExpired(bgCtx)
				// if err != nil {
				//     logger.Error("token cleanup failed", slog.String("error", err.Error()))
				//     continue
				// }
				// logger.Info("cleaned up tokens", slog.Int("deleted", deleted))
			}
		}
	}()

	// Background task 2: Metrics reporting (runs every 30 seconds)
	bgWg.Add(1)
	go func() {
		defer bgWg.Done()
		ticker := time.NewTicker(30 * time.Second)
		defer ticker.Stop()

		logger.Info("metrics reporter started")
		for {
			select {
			case <-bgCtx.Done():
				logger.Info("metrics reporter stopping")
				return
			case <-ticker.C:
				logger.Info("reporting metrics")
				// reportMetrics(bgCtx)
			}
		}
	}()

	// Wait for shutdown signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("received shutdown signal")

	// Cancel all background tasks
	bgCancel()

	// Wait for all background tasks to finish
	done := make(chan struct{})
	go func() {
		bgWg.Wait()
		close(done)
	}()

	select {
	case <-done:
		logger.Info("all background tasks stopped")
	case <-time.After(5 * time.Second):
		logger.Error("background tasks did not stop in time")
	}

	fmt.Println("Clean exit.")
}
```

### The Shutdown Order Matters

This is critical: you must stop things in the right order.

```
Shutdown sequence:
1. Stop accepting new HTTP connections (srv.Shutdown)
2. Wait for in-flight requests to complete
3. Cancel background task context (bgCancel)
4. Wait for background tasks to finish (bgWg.Wait)
5. Close database connections (db.Close)
6. Flush logs
7. Exit
```

If you close the database before cancelling background tasks, those tasks will panic or log errors when they try to query the database. If you cancel background tasks before the HTTP server finishes shutdown, you might have handlers that depend on background services that are no longer running.

### Real-World Background Task: Token Cleanup from krafty-core

```go
package main

import (
	"context"
	"database/sql"
	"log/slog"
	"os"
	"sync"
	"time"
)

// TokenRepository handles database operations for auth tokens
type TokenRepository struct {
	db     *sql.DB
	logger *slog.Logger
}

func NewTokenRepository(db *sql.DB, logger *slog.Logger) *TokenRepository {
	return &TokenRepository{db: db, logger: logger}
}

// DeleteExpired removes tokens that have passed their expiry time.
// Returns the number of deleted tokens.
func (r *TokenRepository) DeleteExpired(ctx context.Context) (int, error) {
	result, err := r.db.ExecContext(ctx,
		"DELETE FROM tokens WHERE expires_at < NOW()",
	)
	if err != nil {
		return 0, err
	}
	deleted, err := result.RowsAffected()
	if err != nil {
		return 0, err
	}
	return int(deleted), nil
}

// StartTokenCleanup runs periodic token cleanup as a background task.
// It respects context cancellation for graceful shutdown.
func StartTokenCleanup(
	ctx context.Context,
	wg *sync.WaitGroup,
	repo *TokenRepository,
	interval time.Duration,
	logger *slog.Logger,
) {
	wg.Add(1)
	go func() {
		defer wg.Done()

		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		logger.Info("token cleanup started",
			slog.String("interval", interval.String()),
		)

		// Run immediately on startup, then on every tick
		cleanup := func() {
			// Use a timeout context derived from the background context
			cleanupCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
			defer cancel()

			deleted, err := repo.DeleteExpired(cleanupCtx)
			if err != nil {
				// Check if the error is because our parent context was cancelled
				if ctx.Err() != nil {
					return
				}
				logger.Error("token cleanup failed",
					slog.String("error", err.Error()),
				)
				return
			}

			if deleted > 0 {
				logger.Info("expired tokens cleaned up",
					slog.Int("deleted", deleted),
				)
			}
		}

		cleanup() // Run once at startup

		for {
			select {
			case <-ctx.Done():
				logger.Info("token cleanup stopping")
				return
			case <-ticker.C:
				cleanup()
			}
		}
	}()
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// In production, this would be a real database connection
	// db, _ := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	// repo := NewTokenRepository(db, logger)

	bgCtx, bgCancel := context.WithCancel(context.Background())
	defer bgCancel()

	var bgWg sync.WaitGroup

	// Start background tasks
	// StartTokenCleanup(bgCtx, &bgWg, repo, 1*time.Hour, logger)

	_ = bgCtx
	_ = &bgWg

	logger.Info("application started")

	// ... server lifecycle ...

	// On shutdown:
	bgCancel()
	bgWg.Wait()
	logger.Info("all background tasks stopped")
}
```

### Node.js Comparison

```javascript
// Node.js -- background tasks with setInterval
const cleanupInterval = setInterval(async () => {
  try {
    const result = await db.query('DELETE FROM tokens WHERE expires_at < NOW()');
    console.log(`Cleaned up ${result.rowCount} tokens`);
  } catch (err) {
    console.error('Token cleanup failed:', err);
  }
}, 60 * 60 * 1000); // 1 hour

// On shutdown, you must remember to clear the interval
process.on('SIGTERM', () => {
  clearInterval(cleanupInterval);
  // But what if the cleanup query is currently running?
  // There is no built-in way to "cancel" it.
});
```

Go's context-based cancellation is fundamentally more powerful: when you cancel the context, any in-progress database query using that context will be interrupted. In Node.js, there is no standard way to cancel an in-progress database query.

---

## 5. Health Check Endpoints

### Liveness vs Readiness Probes

In Kubernetes (and other orchestrators), there are two types of health checks:

**Liveness probe:** "Is the process alive and not deadlocked?" If this fails, Kubernetes kills the pod and restarts it. This should be a simple check that the process can respond to HTTP requests.

**Readiness probe:** "Is the process ready to receive traffic?" If this fails, the pod is removed from the Service's endpoints (no traffic is sent to it), but it is NOT killed. This should check that all dependencies (database, cache, etc.) are reachable.

```
+-----------+-----------------------------+--------------------------------+
| Probe     | What it checks              | What happens on failure        |
+-----------+-----------------------------+--------------------------------+
| Liveness  | Process is alive            | Pod is killed and restarted    |
| Readiness | Dependencies are reachable  | Pod removed from load balancer |
+-----------+-----------------------------+--------------------------------+
```

**Common mistake:** Making your liveness probe check the database. If the database goes down, Kubernetes will restart all your pods in a crash loop, making the situation worse. The liveness probe should only check that the process itself is functioning.

### Complete Health Check Implementation

```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"runtime"
	"sync/atomic"
	"time"
)

// HealthHandler manages health check endpoints
type HealthHandler struct {
	db     *sql.DB
	logger *slog.Logger
	ready  atomic.Bool // Atomic flag for readiness state
}

func NewHealthHandler(db *sql.DB, logger *slog.Logger) *HealthHandler {
	return &HealthHandler{
		db:     db,
		logger: logger,
	}
}

// SetReady marks the service as ready to receive traffic
func (h *HealthHandler) SetReady(ready bool) {
	h.ready.Store(ready)
}

// LivenessResponse represents the liveness probe response
type LivenessResponse struct {
	Status     string `json:"status"`
	Goroutines int    `json:"goroutines"`
}

// ReadinessResponse represents the readiness probe response
type ReadinessResponse struct {
	Status   string            `json:"status"`
	Checks   map[string]string `json:"checks"`
	Duration string            `json:"duration"`
}

// Liveness is a simple check that the process is alive.
// It should NEVER check external dependencies.
func (h *HealthHandler) Liveness(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(LivenessResponse{
		Status:     "alive",
		Goroutines: runtime.NumGoroutine(),
	})
}

// Readiness checks all dependencies and reports whether the service
// can handle traffic.
func (h *HealthHandler) Readiness(w http.ResponseWriter, r *http.Request) {
	start := time.Now()

	checks := make(map[string]string)
	allHealthy := true

	// Check: Is the application in ready state?
	if !h.ready.Load() {
		checks["app"] = "not ready (starting up or shutting down)"
		allHealthy = false
	} else {
		checks["app"] = "ready"
	}

	// Check: Database connectivity
	dbCtx, dbCancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer dbCancel()

	if err := h.db.PingContext(dbCtx); err != nil {
		checks["database"] = fmt.Sprintf("unreachable: %s", err.Error())
		allHealthy = false
		h.logger.Warn("readiness check: database unreachable",
			slog.String("error", err.Error()),
		)
	} else {
		checks["database"] = "connected"
	}

	// Check: Database pool stats
	stats := h.db.Stats()
	if stats.OpenConnections >= stats.MaxOpenConnections {
		checks["db_pool"] = fmt.Sprintf("exhausted (%d/%d)",
			stats.OpenConnections, stats.MaxOpenConnections)
		allHealthy = false
	} else {
		checks["db_pool"] = fmt.Sprintf("healthy (%d/%d)",
			stats.OpenConnections, stats.MaxOpenConnections)
	}

	duration := time.Since(start)

	resp := ReadinessResponse{
		Checks:   checks,
		Duration: duration.String(),
	}

	w.Header().Set("Content-Type", "application/json")

	if allHealthy {
		resp.Status = "ready"
		w.WriteHeader(http.StatusOK)
	} else {
		resp.Status = "not ready"
		w.WriteHeader(http.StatusServiceUnavailable)
	}

	json.NewEncoder(w).Encode(resp)
}

// Health is a combined health check for simpler deployments.
// This is the pattern used in krafty-core.
func (h *HealthHandler) Health(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer cancel()

	if err := h.db.PingContext(ctx); err != nil {
		h.logger.Error("health check failed",
			slog.String("error", err.Error()),
		)
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusServiceUnavailable)
		json.NewEncoder(w).Encode(map[string]string{
			"status": "unhealthy",
			"error":  "database unreachable",
		})
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{
		"status": "healthy",
	})
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// In production, this would be a real database connection
	db, err := sql.Open("postgres", "postgres://localhost/mydb?sslmode=disable")
	if err != nil {
		logger.Error("failed to open database", slog.String("error", err.Error()))
		os.Exit(1)
	}
	defer db.Close()

	health := NewHealthHandler(db, logger)

	mux := http.NewServeMux()

	// Liveness: simple process check (for Kubernetes liveness probe)
	mux.HandleFunc("GET /healthz", health.Liveness)

	// Readiness: full dependency check (for Kubernetes readiness probe)
	mux.HandleFunc("GET /readyz", health.Readiness)

	// Combined health (for simpler deployments)
	mux.HandleFunc("GET /health", health.Health)

	// Mark as ready after all initialization is complete
	health.SetReady(true)
	logger.Info("service marked as ready")

	// During shutdown, mark as not ready BEFORE stopping the server:
	// health.SetReady(false)
	// time.Sleep(5 * time.Second) // Give LB time to stop sending traffic
	// srv.Shutdown(ctx)

	srv := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	logger.Info("starting server", slog.String("addr", srv.Addr))
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		logger.Error("server failed", slog.String("error", err.Error()))
		os.Exit(1)
	}
}
```

### The Ready Flag Pattern

Notice the `atomic.Bool` field called `ready`. This pattern is important:

1. When the server starts, `ready` is false
2. After all initialization is complete (database connected, migrations run, caches warmed), set `ready` to true
3. When shutdown begins, set `ready` to false BEFORE calling `srv.Shutdown()`

Step 3 is subtle but important. During shutdown, there is a window where the pod is still receiving traffic (Kubernetes has not yet removed it from the endpoint list) but you are about to shut down. By failing the readiness probe during this window, you accelerate the removal from the load balancer.

```go
// During shutdown:
logger.Info("starting graceful shutdown")

// 1. Mark as not ready -- readiness probe will fail
health.SetReady(false)

// 2. Wait a moment for the load balancer to notice
// This gives Kubernetes time to update endpoints
time.Sleep(5 * time.Second)

// 3. Now shut down the HTTP server
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

### Node.js Comparison

```javascript
// Node.js health check
const express = require('express');
const app = express();

let isReady = false;

// Liveness
app.get('/healthz', (req, res) => {
  res.json({ status: 'alive' });
});

// Readiness
app.get('/readyz', async (req, res) => {
  if (!isReady) {
    return res.status(503).json({ status: 'not ready' });
  }

  try {
    await pool.query('SELECT 1');
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message });
  }
});

// After initialization:
isReady = true;

// In Go, the atomic.Bool provides thread-safe access without locks.
// In Node.js (single-threaded), a plain boolean works, but
// you cannot set a timeout on the database ping without additional code.
```

---

## 6. Server Timeouts

### WHY Timeouts Are Non-Negotiable

Without timeouts, a single slow or malicious client can consume a connection and a goroutine forever. Multiply this by thousands of connections, and you have a denial-of-service vulnerability built into your own server.

Go's `http.Server` has four timeout fields. Each controls a different phase of the request lifecycle:

```
     Client        Server
       |              |
       |---connect--->|
       |              |---- ReadHeaderTimeout starts
       |---headers--->|
       |              |---- ReadHeaderTimeout ends
       |              |---- ReadTimeout starts (from connection accept)
       |----body----->|
       |              |---- ReadTimeout ends
       |              |---- WriteTimeout starts (from end of header read)
       |              |---- Handler executes
       |<--response---|
       |              |---- WriteTimeout ends
       |              |
       |   (idle)     |---- IdleTimeout starts
       |              |
       |---request--->|---- IdleTimeout ends (new request on same connection)
```

### The Four Timeouts Explained

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           router,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       15 * time.Second,
	WriteTimeout:      15 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

**ReadHeaderTimeout** (Go 1.8+): How long to wait for the client to send request headers after connecting. This is your primary defense against slowloris attacks, where an attacker opens a connection and sends headers extremely slowly to tie up your server's goroutines.

**ReadTimeout**: Maximum duration from when the connection is accepted to when the entire request body is read. This includes `ReadHeaderTimeout`. If you set both, `ReadTimeout` must be >= `ReadHeaderTimeout`.

**WriteTimeout**: Maximum duration from when the request headers are read to when the response is fully written. This covers your handler execution time. If your handler takes longer than `WriteTimeout`, the connection is closed and the client gets nothing.

**IdleTimeout**: How long to keep an idle keep-alive connection open between requests. After this time, the connection is closed. This reclaims resources from clients that opened a connection but are not actively making requests.

### Choosing Timeout Values

```
+-------------------+------------------+--------------------------------------+
| Timeout           | Recommended      | Reasoning                            |
+-------------------+------------------+--------------------------------------+
| ReadHeaderTimeout | 5s               | Headers should arrive quickly        |
| ReadTimeout       | 15s              | Generous for file uploads            |
| WriteTimeout      | 15s              | Must exceed your slowest handler     |
| IdleTimeout       | 60-120s          | Matches typical LB idle timeouts     |
+-------------------+------------------+--------------------------------------+
```

**Critical rule:** Your `WriteTimeout` must be longer than your slowest handler. If you have a handler that takes 10 seconds to process (e.g., a complex report), set `WriteTimeout` to at least 15 seconds.

**Critical rule:** Your `IdleTimeout` must be shorter than any upstream load balancer or reverse proxy timeout. If your load balancer has a 60-second idle timeout, set yours to 60 seconds or less. If yours is longer, the LB will close the connection and the server will try to write to a dead connection.

### Complete Example with All Timeouts

```go
package main

import (
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()

	// Fast endpoint
	mux.HandleFunc("GET /api/ping", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"pong": "ok"})
	})

	// Slow endpoint (report generation)
	mux.HandleFunc("GET /api/reports/generate", func(w http.ResponseWriter, r *http.Request) {
		// This takes up to 10 seconds
		time.Sleep(10 * time.Second)
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"report": "done"})
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: mux,

		// Defense against slowloris attacks
		ReadHeaderTimeout: 5 * time.Second,

		// Maximum time to read the full request (headers + body)
		ReadTimeout: 15 * time.Second,

		// Maximum time from end of request header read to end of response write
		// Must be > your slowest handler (10s report) + some buffer
		WriteTimeout: 30 * time.Second,

		// How long to keep idle keep-alive connections open
		// Set this <= your load balancer's idle timeout
		IdleTimeout: 60 * time.Second,

		// Maximum header size (defense against excessively large headers)
		MaxHeaderBytes: 1 << 20, // 1 MB
	}

	logger.Info("starting server",
		slog.String("addr", srv.Addr),
		slog.String("readHeaderTimeout", "5s"),
		slog.String("readTimeout", "15s"),
		slog.String("writeTimeout", "30s"),
		slog.String("idleTimeout", "60s"),
	)

	if err := srv.ListenAndServe(); err != nil {
		logger.Error("server failed", slog.String("error", err.Error()))
		os.Exit(1)
	}
}
```

### Per-Handler Timeouts with http.TimeoutHandler

If most of your endpoints are fast but one is slow, you can use `http.TimeoutHandler` instead of increasing `WriteTimeout` for the entire server:

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()

	// Fast handler -- no special timeout
	mux.HandleFunc("GET /api/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "pong")
	})

	// Slow handler -- wrapped with its own timeout
	slowHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(10 * time.Second)
		fmt.Fprintln(w, "done")
	})
	mux.Handle("GET /api/reports",
		http.TimeoutHandler(slowHandler, 30*time.Second, "request timed out\n"),
	)

	srv := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		WriteTimeout: 15 * time.Second, // Short default for most endpoints
		IdleTimeout:  60 * time.Second,
	}

	srv.ListenAndServe()
}
```

**Important:** `http.TimeoutHandler` sends a 503 Service Unavailable response when the timeout fires. This is different from `WriteTimeout`, which silently closes the connection. `TimeoutHandler` is generally a better user experience.

### Node.js Comparison

```javascript
// Node.js -- server timeouts
const server = app.listen(3000);

// Equivalent of ReadTimeout + WriteTimeout combined
server.timeout = 15000; // 15 seconds total

// Equivalent of ReadHeaderTimeout
server.headersTimeout = 5000; // 5 seconds

// Equivalent of IdleTimeout
server.keepAliveTimeout = 60000; // 60 seconds

// Node.js does not have per-handler timeouts built in.
// You must implement them yourself:
app.get('/slow', (req, res) => {
  req.setTimeout(30000); // Per-request timeout
  // ...
});
```

Go's approach is more granular: you have separate control over reading headers, reading the body, writing the response, and idle connections. Node.js lumps some of these together.

---

## 7. Connection Draining

### How Shutdown Drains Connections

Connection draining is the process of allowing in-flight requests to complete while refusing new connections. Go's `http.Server.Shutdown()` handles this automatically, but understanding the mechanics is important for debugging and tuning.

### The Shutdown Dance

```
Timeline:

  srv.Shutdown(ctx) called
  |
  |-- Listener closed (stop accepting new TCP connections)
  |
  |-- Idle keep-alive connections closed immediately
  |
  |-- Active connections tracked:
  |     - Request A: 2 seconds remaining
  |     - Request B: 5 seconds remaining
  |     - Request C: 8 seconds remaining
  |
  |-- Request A completes, connection closed
  |-- Request B completes, connection closed
  |-- Request C completes, connection closed
  |
  |-- Shutdown returns nil (all connections drained)

  OR

  |-- Context deadline fires before all requests complete
  |-- Shutdown returns context.DeadlineExceeded
  |-- Remaining connections are abandoned (NOT closed by Shutdown)
```

### SetKeepAlivesEnabled

Before calling `Shutdown`, you can disable keep-alives to signal clients to close their connections after the current request:

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
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	mux := http.NewServeMux()
	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(2 * time.Second)
		fmt.Fprintln(w, "OK")
	})

	srv := &http.Server{
		Addr:        ":8080",
		Handler:     mux,
		IdleTimeout: 60 * time.Second,
	}

	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("server error", slog.String("error", err.Error()))
			os.Exit(1)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("shutdown initiated")

	// Step 1: Disable keep-alives
	// This sets the "Connection: close" header on all subsequent responses,
	// telling clients not to reuse this connection.
	srv.SetKeepAlivesEnabled(false)
	logger.Info("keep-alives disabled")

	// Step 2: Give in-flight requests a moment to complete
	// (Shutdown does this anyway, but being explicit aids understanding)
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	// Step 3: Shutdown
	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("shutdown error", slog.String("error", err.Error()))
		os.Exit(1)
	}

	logger.Info("server drained and stopped")
}
```

### What Happens When Shutdown Times Out

If the context deadline fires before all connections are drained, `Shutdown` returns `context.DeadlineExceeded`, but it does **not** forcefully close the remaining connections. Those connections are left open, and the goroutines serving them continue to run.

In practice, this means you should call `srv.Close()` as a fallback:

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
	logger.Error("graceful shutdown failed, forcing close",
		slog.String("error", err.Error()),
	)
	// Close forcefully terminates all active connections
	srv.Close()
}
```

`srv.Close()` immediately closes all listeners and all connections. It does not wait for in-flight requests. Use it only as a last resort.

### Connection Draining in Load Balanced Environments

In production, your service sits behind a load balancer (Kubernetes Service, AWS ALB, etc.). The draining sequence becomes:

```
1. Orchestrator sends SIGTERM
2. Your app fails the readiness probe
3. Load balancer removes your pod from its target list
   (This takes a few seconds -- there is always a gap!)
4. Your app calls srv.Shutdown()
5. In-flight requests complete
6. Process exits

The GAP in step 3 is critical.

During the gap, the load balancer might still send new requests
to your pod. If you have already closed the listener, those
requests get "connection refused."

Solution: Sleep briefly before shutting down the server.
```

```go
// On SIGTERM:
logger.Info("SIGTERM received, starting drain sequence")

// Fail readiness probe
health.SetReady(false)

// Wait for load balancer to stop sending traffic
// This should be at least 2x your readiness probe interval
time.Sleep(5 * time.Second)

// Now shut down the server
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

---

## 8. Database Connection Cleanup

### The Defer Pattern

Database connections are expensive resources. A connection pool that is not cleaned up on shutdown can leave orphaned connections on the database server, consuming resources and potentially hitting connection limits.

The cleanup is simple: `defer db.Close()`.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log/slog"
	"os"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Open the database connection pool
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		logger.Error("failed to open database", slog.String("error", err.Error()))
		os.Exit(1)
	}
	// This is the critical line. defer ensures Close is called
	// when main() returns, regardless of how it returns.
	defer db.Close()

	// Configure the pool
	db.SetMaxOpenConns(25)              // Maximum connections in the pool
	db.SetMaxIdleConns(5)               // Connections kept idle for reuse
	db.SetConnMaxLifetime(5 * time.Minute) // Max time a connection can be reused
	db.SetConnMaxIdleTime(1 * time.Minute) // Max time a connection can be idle

	// Verify connectivity
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		logger.Error("database unreachable", slog.String("error", err.Error()))
		os.Exit(1)
	}

	logger.Info("database connected",
		slog.Int("maxOpenConns", 25),
		slog.Int("maxIdleConns", 5),
	)

	// ... rest of application ...

	fmt.Println("Application running")
}
```

### Connection Pool Settings Explained

```
+-------------------+-----------------------------------------------------------+
| Setting           | What it does                                              |
+-------------------+-----------------------------------------------------------+
| MaxOpenConns      | Hard limit on total connections (active + idle).          |
|                   | Set this to match your database's max_connections         |
|                   | divided by the number of application instances.           |
+-------------------+-----------------------------------------------------------+
| MaxIdleConns      | How many idle connections to keep open for reuse.         |
|                   | Higher = faster response to traffic bursts.               |
|                   | Lower = fewer resources consumed when idle.               |
+-------------------+-----------------------------------------------------------+
| ConnMaxLifetime   | How long a connection can be reused. After this time,     |
|                   | the connection is closed and a new one is created.        |
|                   | Prevents stale connections (e.g., after DB failover).     |
+-------------------+-----------------------------------------------------------+
| ConnMaxIdleTime   | How long an idle connection can sit unused. After this    |
|                   | time, the idle connection is closed. Reclaims resources   |
|                   | after traffic spikes.                                     |
+-------------------+-----------------------------------------------------------+
```

### Monitoring Pool Health

```go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"net/http"
)

// PoolStats returns current connection pool statistics
func PoolStatsHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		stats := db.Stats()

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"max_open_connections": stats.MaxOpenConnections,
			"open_connections":     stats.OpenConnections,
			"in_use":               stats.InUse,
			"idle":                 stats.Idle,
			"wait_count":           stats.WaitCount,
			"wait_duration_ms":     stats.WaitDuration.Milliseconds(),
			"max_idle_closed":      stats.MaxIdleClosed,
			"max_idle_time_closed": stats.MaxIdleTimeClosed,
			"max_lifetime_closed":  stats.MaxLifetimeClosed,
		})
	}
}

func main() {
	// db, _ := sql.Open(...)
	// mux.HandleFunc("GET /debug/db-pool", PoolStatsHandler(db))
	fmt.Println("Pool stats handler registered")
}
```

**Key metrics to alert on:**

- `WaitCount` increasing: Your pool is too small, requests are waiting for connections
- `InUse` == `MaxOpenConnections`: Pool is exhausted, new queries will block
- `MaxLifetimeClosed` high: Connections are being recycled frequently (ConnMaxLifetime too short)

### Shutdown Order for Database Connections

```go
// CORRECT order in main():
func main() {
    db := connectDatabase()
    defer db.Close() // Will execute LAST (LIFO order of defers)

    bgCtx, bgCancel := context.WithCancel(context.Background())
    defer bgCancel() // Will execute THIRD

    srv := startServer(db)

    // Wait for shutdown signal
    <-quit

    // 1. Stop accepting traffic
    srv.Shutdown(ctx)       // In-flight requests may still use db

    // 2. Cancel background tasks
    bgCancel()              // Background tasks may still use db
    bgWg.Wait()             // Wait for them to finish

    // 3. db.Close() runs via defer -- now safe because
    //    nothing is using the database anymore
}
```

If you close the database before the server or background tasks finish, those operations will fail with "sql: database is closed" errors.

---

## 9. Startup Sequence

### The Correct Order

Application initialization is not random. Each step depends on the previous one. Here is the correct startup sequence for a production Go API:

```
Step 1: Load Configuration
    ↓ (config needed for everything else)
Step 2: Setup Logger
    ↓ (logger needed to report errors in subsequent steps)
Step 3: Connect to Database
    ↓ (database needed for migrations and repositories)
Step 4: Run Database Migrations
    ↓ (schema must exist before querying)
Step 5: Create Repositories
    ↓ (repositories needed by handlers)
Step 6: Create Service/Handler Layer
    ↓ (handlers needed by router)
Step 7: Setup Router with Middleware
    ↓ (router is the HTTP handler)
Step 8: Create and Configure HTTP Server
    ↓ (server ready to start)
Step 9: Start Background Tasks
    ↓ (background work begins)
Step 10: Start HTTP Server (in goroutine)
    ↓ (accepting traffic)
Step 11: Mark as Ready
    ↓ (readiness probe passes)
Step 12: Wait for Shutdown Signal
```

### Each Step in Detail

```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"time"
)

// Step 1: Configuration
type Config struct {
	Port        string
	DatabaseURL string
	LogLevel    string
	Environment string
}

func LoadConfig() (*Config, error) {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}

	logLevel := os.Getenv("LOG_LEVEL")
	if logLevel == "" {
		logLevel = "info"
	}

	env := os.Getenv("ENVIRONMENT")
	if env == "" {
		env = "development"
	}

	return &Config{
		Port:        port,
		DatabaseURL: dbURL,
		LogLevel:    logLevel,
		Environment: env,
	}, nil
}

// Step 2: Logger Setup
func SetupLogger(level string) *slog.Logger {
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

	handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: logLevel,
	})

	return slog.New(handler)
}

// Step 3: Database Connection
func ConnectDatabase(dbURL string, logger *slog.Logger) (*sql.DB, error) {
	db, err := sql.Open("pgx", dbURL)
	if err != nil {
		return nil, fmt.Errorf("failed to open database: %w", err)
	}

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, fmt.Errorf("database unreachable: %w", err)
	}

	logger.Info("database connected")
	return db, nil
}

// Step 4: Migrations (simplified example)
func RunMigrations(db *sql.DB, logger *slog.Logger) error {
	// In production, use a migration tool like golang-migrate, goose, or atlas
	migrations := []string{
		`CREATE TABLE IF NOT EXISTS users (
			id SERIAL PRIMARY KEY,
			email TEXT UNIQUE NOT NULL,
			created_at TIMESTAMPTZ DEFAULT NOW()
		)`,
		`CREATE TABLE IF NOT EXISTS tokens (
			id SERIAL PRIMARY KEY,
			user_id INTEGER REFERENCES users(id),
			token TEXT UNIQUE NOT NULL,
			expires_at TIMESTAMPTZ NOT NULL
		)`,
	}

	for i, migration := range migrations {
		if _, err := db.Exec(migration); err != nil {
			return fmt.Errorf("migration %d failed: %w", i, err)
		}
	}

	logger.Info("migrations completed", slog.Int("count", len(migrations)))
	return nil
}

// Steps 5-6: Repository and Handler
type UserRepository struct {
	db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

type Handler struct {
	userRepo *UserRepository
	logger   *slog.Logger
}

func NewHandler(userRepo *UserRepository, logger *slog.Logger) *Handler {
	return &Handler{userRepo: userRepo, logger: logger}
}

func (h *Handler) GetUsers(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"users": "list"})
}

// Step 7: Router Setup
func SetupRouter(handler *Handler, logger *slog.Logger) http.Handler {
	mux := http.NewServeMux()

	// API routes
	mux.HandleFunc("GET /api/users", handler.GetUsers)

	// Health check
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
	})

	// Add middleware (logging, recovery, etc.)
	var h http.Handler = mux
	h = LoggingMiddleware(logger, h)
	h = RecoveryMiddleware(logger, h)

	return h
}

// Middleware examples
func LoggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		logger.Info("request",
			slog.String("method", r.Method),
			slog.String("path", r.URL.Path),
			slog.Duration("duration", time.Since(start)),
		)
	})
}

func RecoveryMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				logger.Error("panic recovered",
					slog.Any("panic", rec),
					slog.String("path", r.URL.Path),
				)
				http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

func main() {
	// This function demonstrates the startup sequence.
	// Each step depends on the previous one.

	// Step 1: Load configuration
	cfg, err := LoadConfig()
	if err != nil {
		fmt.Fprintf(os.Stderr, "configuration error: %s\n", err)
		os.Exit(1)
	}

	// Step 2: Setup logger (needs config for log level)
	logger := SetupLogger(cfg.LogLevel)
	logger.Info("configuration loaded",
		slog.String("environment", cfg.Environment),
		slog.String("port", cfg.Port),
	)

	// Step 3: Connect to database (needs config for URL, logger for errors)
	db, err := ConnectDatabase(cfg.DatabaseURL, logger)
	if err != nil {
		logger.Error("database connection failed", slog.String("error", err.Error()))
		os.Exit(1)
	}
	defer db.Close()

	// Step 4: Run migrations (needs database)
	if err := RunMigrations(db, logger); err != nil {
		logger.Error("migration failed", slog.String("error", err.Error()))
		os.Exit(1)
	}

	// Step 5: Create repositories (needs database)
	userRepo := NewUserRepository(db)

	// Step 6: Create handlers (needs repositories)
	handler := NewHandler(userRepo, logger)

	// Step 7: Setup router (needs handlers)
	router := SetupRouter(handler, logger)

	// Step 8: Create server (needs router)
	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      router,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	logger.Info("startup sequence complete, server starting",
		slog.String("addr", srv.Addr),
	)

	// Steps 9-12 would follow (background tasks, start server, wait for shutdown)
}
```

### WHY the Order Matters

Every step has a dependency chain:

```
Config has no dependencies      → Load first
Logger depends on Config        → Load second
Database depends on Config      → Load after Config (logger for error reporting)
Migrations depend on Database   → Run after DB connects
Repositories depend on Database → Create after migrations
Handlers depend on Repositories → Create after repos
Router depends on Handlers      → Create after handlers
Server depends on Router        → Create last
```

If you try to create a repository before the database is connected, you get a nil pointer dereference. If you try to query a table before migrations run, you get "relation does not exist." The startup sequence is a dependency graph, and you must resolve it in topological order.

---

## 10. Structured Main Function

### The Problem with Long Main Functions

A common mistake is putting everything in `main()`. The function grows to hundreds of lines, mixing initialization, business logic, and error handling. It becomes unreadable and untestable.

### The run() Pattern

A production pattern is to move the real logic into a `run()` function that returns an error. This makes the entry point testable and the error handling consistent:

```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
}

func run() error {
	// -------------------------------------------------------
	// Configuration
	// -------------------------------------------------------
	port := envOrDefault("PORT", "8080")
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		return fmt.Errorf("DATABASE_URL environment variable is required")
	}
	logLevel := envOrDefault("LOG_LEVEL", "info")

	// -------------------------------------------------------
	// Logger
	// -------------------------------------------------------
	logger := setupLogger(logLevel)
	logger.Info("starting application", slog.String("port", port))

	// -------------------------------------------------------
	// Database
	// -------------------------------------------------------
	db, err := setupDatabase(dbURL)
	if err != nil {
		return fmt.Errorf("database setup: %w", err)
	}
	defer db.Close()
	logger.Info("database connected")

	// -------------------------------------------------------
	// Background context (for long-running goroutines)
	// -------------------------------------------------------
	bgCtx, bgCancel := context.WithCancel(context.Background())
	defer bgCancel()

	var bgWg sync.WaitGroup

	// Start background tasks
	startTokenCleanup(bgCtx, &bgWg, db, logger)

	// -------------------------------------------------------
	// Router
	// -------------------------------------------------------
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", healthHandler(db))
	mux.HandleFunc("GET /api/users", listUsersHandler(db, logger))

	// -------------------------------------------------------
	// HTTP Server
	// -------------------------------------------------------
	srv := &http.Server{
		Addr:              ":" + port,
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// Start server in goroutine
	errChan := make(chan error, 1)
	go func() {
		logger.Info("HTTP server starting", slog.String("addr", srv.Addr))
		errChan <- srv.ListenAndServe()
	}()

	// -------------------------------------------------------
	// Wait for shutdown signal or server error
	// -------------------------------------------------------
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case sig := <-quit:
		logger.Info("received signal", slog.String("signal", sig.String()))
	case err := <-errChan:
		if err != nil && err != http.ErrServerClosed {
			return fmt.Errorf("server error: %w", err)
		}
	}

	// -------------------------------------------------------
	// Graceful shutdown
	// -------------------------------------------------------
	logger.Info("initiating graceful shutdown")

	// 1. Shut down HTTP server (drain in-flight requests)
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer shutdownCancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("server shutdown: %w", err)
	}
	logger.Info("HTTP server stopped")

	// 2. Cancel background tasks
	bgCancel()

	// 3. Wait for background tasks with timeout
	bgDone := make(chan struct{})
	go func() {
		bgWg.Wait()
		close(bgDone)
	}()

	select {
	case <-bgDone:
		logger.Info("background tasks stopped")
	case <-time.After(5 * time.Second):
		logger.Warn("background tasks did not stop in time")
	}

	// 4. db.Close() runs via defer

	logger.Info("shutdown complete")
	return nil
}

// -------------------------------------------------------
// Helper functions
// -------------------------------------------------------

func envOrDefault(key, fallback string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return fallback
}

func setupLogger(level string) *slog.Logger {
	var lvl slog.Level
	switch level {
	case "debug":
		lvl = slog.LevelDebug
	case "warn":
		lvl = slog.LevelWarn
	case "error":
		lvl = slog.LevelError
	default:
		lvl = slog.LevelInfo
	}
	return slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: lvl}))
}

func setupDatabase(url string) (*sql.DB, error) {
	db, err := sql.Open("pgx", url)
	if err != nil {
		return nil, err
	}

	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, err
	}

	return db, nil
}

func startTokenCleanup(ctx context.Context, wg *sync.WaitGroup, db *sql.DB, logger *slog.Logger) {
	wg.Add(1)
	go func() {
		defer wg.Done()
		ticker := time.NewTicker(1 * time.Hour)
		defer ticker.Stop()

		for {
			select {
			case <-ctx.Done():
				logger.Info("token cleanup stopped")
				return
			case <-ticker.C:
				cleanupCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
				result, err := db.ExecContext(cleanupCtx, "DELETE FROM tokens WHERE expires_at < NOW()")
				cancel()
				if err != nil {
					if ctx.Err() != nil {
						return
					}
					logger.Error("token cleanup failed", slog.String("error", err.Error()))
					continue
				}
				deleted, _ := result.RowsAffected()
				if deleted > 0 {
					logger.Info("tokens cleaned up", slog.Int64("deleted", deleted))
				}
			}
		}
	}()
}

func healthHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
		defer cancel()

		w.Header().Set("Content-Type", "application/json")
		if err := db.PingContext(ctx); err != nil {
			w.WriteHeader(http.StatusServiceUnavailable)
			json.NewEncoder(w).Encode(map[string]string{
				"status": "unhealthy",
				"error":  "database unreachable",
			})
			return
		}

		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]string{
			"status": "healthy",
		})
	}
}

func listUsersHandler(db *sql.DB, logger *slog.Logger) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"users": "[]",
		})
	}
}
```

### WHY the run() Pattern

1. **Testability:** You can call `run()` from a test and check the returned error
2. **Single exit point:** All errors flow through the return value instead of scattered `os.Exit()` calls
3. **Defer works correctly:** In `main()`, `os.Exit()` does not run deferred functions. In `run()`, returning an error allows `main()` to call `os.Exit()` after all defers in `run()` have executed
4. **Clean separation:** `main()` is one line of real code

### The os.Exit Gotcha

This is a subtle but important detail:

```go
// PROBLEM: os.Exit does NOT run deferred functions
func main() {
    db := connectDB()
    defer db.Close() // This will NOT run if os.Exit is called below!

    if err := doSomething(); err != nil {
        log.Fatal(err) // log.Fatal calls os.Exit(1)
        // db.Close() is NEVER called
    }
}

// SOLUTION: The run() pattern
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "error: %s\n", err)
        os.Exit(1) // All defers in run() have already executed
    }
}

func run() error {
    db := connectDB()
    defer db.Close() // This WILL run when run() returns

    if err := doSomething(); err != nil {
        return err // db.Close() runs as we unwind
    }
    return nil
}
```

---

## 11. Docker Considerations

### Signal Forwarding (The #1 Docker Gotcha)

When Docker sends SIGTERM to stop a container, it sends it to PID 1 inside the container. If PID 1 is a shell (like `/bin/sh`), the shell does **not** forward the signal to your Go process. Your process never receives SIGTERM, Docker waits for the grace period (default 10 seconds), then sends SIGKILL. Result: no graceful shutdown, ever.

```dockerfile
# WRONG: Shell form -- /bin/sh is PID 1, Go process is a child
CMD go run main.go

# WRONG: Shell form -- /bin/sh is PID 1
CMD ./myserver

# CORRECT: Exec form -- Go process IS PID 1
CMD ["./myserver"]
```

### Production Dockerfile (Multi-Stage Build)

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy dependency files first (better layer caching)
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build with optimizations
# CGO_ENABLED=0 produces a fully static binary
# -ldflags="-s -w" strips debug info for smaller binary
# -trimpath removes filesystem paths from the binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -trimpath -o /server ./cmd/server

# Stage 2: Run
FROM alpine:3.19

# Install CA certificates (needed for HTTPS calls)
RUN apk --no-cache add ca-certificates tzdata

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy the binary from the builder stage
COPY --from=builder /server .

# Copy migrations if needed
COPY --from=builder /app/migrations ./migrations

# Use non-root user
USER appuser

# Expose the port (documentation only, does not publish)
EXPOSE 8080

# Health check (Docker's built-in health check)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# CRITICAL: Use exec form so the Go process is PID 1
# and receives SIGTERM directly from Docker
CMD ["./server"]
```

### WHY Multi-Stage Builds

```
+---------------------------+---------------+------------------+
| Image Type                | Image Size    | Attack Surface   |
+---------------------------+---------------+------------------+
| golang:1.22               | ~800 MB       | Huge (build tools)|
| golang:1.22-alpine        | ~250 MB       | Large            |
| alpine:3.19 (multi-stage) | ~15 MB        | Small            |
| scratch (multi-stage)     | ~6 MB         | Minimal          |
+---------------------------+---------------+------------------+
```

The build stage uses the full Go SDK. The run stage only contains your binary and minimal dependencies. This reduces image size, pull time, and attack surface.

### Scratch Image (Maximum Minimalism)

For applications that do not need shell access or system certificates (or bundle their own):

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -trimpath -o /server ./cmd/server

# Scratch has NOTHING -- no shell, no ls, no cat, nothing.
FROM scratch

# Copy CA certificates from builder
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the binary
COPY --from=builder /server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Tradeoffs of scratch:**

- Pro: Smallest possible image, minimal attack surface
- Con: Cannot exec into the container for debugging (`kubectl exec` fails)
- Con: No shell for health check commands (must use HTTP probe instead)

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DATABASE_URL=postgres://postgres:secret@db:5432/myapp?sslmode=disable
      - LOG_LEVEL=debug
      - ENVIRONMENT=development
    depends_on:
      db:
        condition: service_healthy
    # Docker sends SIGTERM, waits stop_grace_period, then SIGKILL
    stop_grace_period: 15s

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### Node.js Docker Comparison

```dockerfile
# Node.js Dockerfile -- compare the complexity
FROM node:20-alpine

WORKDIR /app

# Node.js requires TWO package files
COPY package.json package-lock.json ./

# Install dependencies (can be 200+ MB of node_modules)
RUN npm ci --only=production

# Copy source
COPY . .

EXPOSE 3000

# Tini ensures signals are forwarded correctly in Node.js
# because Node.js does not always handle PID 1 correctly
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]

CMD ["node", "server.js"]
```

Go does not need `tini` because a Go binary handles signals correctly when running as PID 1. Node.js needs `tini` or `dumb-init` because the Node.js runtime does not always handle signals properly when running as PID 1 in a container.

---

## 12. Kubernetes Readiness

### Pod Lifecycle and Shutdown

Understanding the Kubernetes pod termination sequence is essential for zero-downtime deployments:

```
1. Deployment rollout begins (new pods created)
2. Old pod marked for termination
3. Two things happen IN PARALLEL:
   a. Pod removed from Service endpoints (async, takes seconds)
   b. SIGTERM sent to PID 1 in the container
4. PreStop hook runs (if configured)
5. terminationGracePeriodSeconds countdown starts
6. SIGKILL if process still running after grace period
```

**The critical insight:** Steps 3a and 3b happen in parallel. When your process receives SIGTERM, traffic from the load balancer might still arrive for a few seconds. This is the "endpoint propagation delay."

### Complete Kubernetes Manifests

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: krafty-api
  labels:
    app: krafty-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # One extra pod during rollout
      maxUnavailable: 0   # Never reduce below desired count
  selector:
    matchLabels:
      app: krafty-api
  template:
    metadata:
      labels:
        app: krafty-api
    spec:
      # Total time from SIGTERM to SIGKILL
      terminationGracePeriodSeconds: 30

      containers:
      - name: api
        image: registry.example.com/krafty-api:latest
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP

        env:
        - name: PORT
          value: "8080"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: krafty-db-credentials
              key: url
        - name: LOG_LEVEL
          value: "info"

        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

        # Liveness: Is the process alive?
        # Failure -> pod is killed and restarted
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        # Readiness: Can the process handle traffic?
        # Failure -> pod removed from Service endpoints
        readinessProbe:
          httpGet:
            path: /readyz
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
          successThreshold: 1

        # Startup: Is the process still starting up?
        # While this probe fails, liveness and readiness are not checked.
        # Prevents slow-starting apps from being killed.
        startupProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 2
          periodSeconds: 3
          failureThreshold: 10  # 10 * 3s = 30s to start

        # PreStop hook: runs BEFORE SIGTERM is sent
        # This sleep gives the endpoint controller time to
        # remove this pod from the Service before we start shutdown
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "5"]
```

### WHY the PreStop Sleep

The PreStop hook is one of the most misunderstood Kubernetes features. Here is why it exists:

```
WITHOUT PreStop:
  t=0: SIGTERM sent AND endpoint removal begins (parallel)
  t=0: Your app starts shutting down immediately
  t=0-3s: Endpoint still exists in some kube-proxy instances
  t=0-3s: New requests arrive at your shutting-down app
  t=0-3s: Some requests get "connection refused" ← BAD

WITH PreStop sleep 5:
  t=0: PreStop begins (sleep 5)
  t=0: Endpoint removal begins (parallel)
  t=3s: Endpoint fully removed everywhere
  t=5s: PreStop finishes, SIGTERM sent
  t=5s: Your app starts shutting down
  t=5s: No new traffic arrives ← GOOD
```

The `sleep 5` in the PreStop hook gives the Kubernetes networking layer time to propagate the endpoint removal before your application starts refusing connections. This is the standard pattern for zero-downtime deployments.

### Service and Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: krafty-api
spec:
  selector:
    app: krafty-api
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: krafty-api
  annotations:
    # Important: keep-alive timeout must be > your IdleTimeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: krafty-api
            port:
              number: 80
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: krafty-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: krafty-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60  # Remove at most 1 pod per minute
```

The slow scale-down policy is important: when pods are removed, they go through the graceful shutdown sequence. If you scale down too fast, many pods are shutting down simultaneously, and the remaining pods might be overwhelmed.

### Pod Disruption Budget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: krafty-api
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: krafty-api
```

This tells Kubernetes: "Always keep at least 2 pods running." During node maintenance, Kubernetes will respect this and drain nodes one at a time, waiting for your pods to be rescheduled before draining the next node.

---

## 13. Error Channels Pattern

### Goroutine Error Propagation

Goroutines cannot return errors. If a goroutine encounters a fatal error, you need a way to communicate that back to the main goroutine. Error channels are the standard pattern.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Error channel collects errors from all goroutines
	errChan := make(chan error, 1)

	// Start HTTP server
	srv := &http.Server{
		Addr:    ":8080",
		Handler: http.DefaultServeMux,
	}

	go func() {
		logger.Info("HTTP server starting")
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			errChan <- fmt.Errorf("HTTP server: %w", err)
		}
	}()

	// Start background worker
	bgCtx, bgCancel := context.WithCancel(context.Background())
	defer bgCancel()

	go func() {
		if err := runWorker(bgCtx, logger); err != nil {
			errChan <- fmt.Errorf("worker: %w", err)
		}
	}()

	// Wait for shutdown signal OR fatal error from any goroutine
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case sig := <-quit:
		logger.Info("received signal", slog.String("signal", sig.String()))
	case err := <-errChan:
		logger.Error("fatal error from goroutine", slog.String("error", err.Error()))
	}

	// Shutdown everything regardless of why we are stopping
	logger.Info("shutting down")
	bgCancel()

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	srv.Shutdown(ctx)

	logger.Info("shutdown complete")
}

func runWorker(ctx context.Context, logger *slog.Logger) error {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return nil
		case <-ticker.C:
			if err := doWork(ctx); err != nil {
				// Decide: is this error fatal or recoverable?
				if isFatal(err) {
					return fmt.Errorf("fatal worker error: %w", err)
				}
				logger.Error("worker error (recoverable)",
					slog.String("error", err.Error()),
				)
			}
		}
	}
}

func doWork(ctx context.Context) error {
	// Simulate work that might fail
	return nil
}

func isFatal(err error) bool {
	// Define what constitutes a fatal error
	// Example: database connection permanently lost
	return errors.Is(err, context.Canceled)
}
```

### Multiple Error Sources with errgroup

The `golang.org/x/sync/errgroup` package provides a more structured approach when you have multiple goroutines that should all stop if any one fails:

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

	"golang.org/x/sync/errgroup"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// Create a context that is cancelled on SIGTERM or SIGINT
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	// errgroup creates a derived context that is cancelled when
	// any goroutine returns an error
	g, gCtx := errgroup.WithContext(ctx)

	// HTTP Server
	srv := &http.Server{
		Addr:    ":8080",
		Handler: http.DefaultServeMux,
	}

	// Goroutine 1: Run the HTTP server
	g.Go(func() error {
		logger.Info("HTTP server starting")
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			return fmt.Errorf("HTTP server: %w", err)
		}
		return nil
	})

	// Goroutine 2: Shut down the HTTP server when the group context is done
	g.Go(func() error {
		<-gCtx.Done()
		logger.Info("shutting down HTTP server")
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		return srv.Shutdown(shutdownCtx)
	})

	// Goroutine 3: Background worker
	g.Go(func() error {
		ticker := time.NewTicker(30 * time.Second)
		defer ticker.Stop()

		for {
			select {
			case <-gCtx.Done():
				logger.Info("worker stopping")
				return nil
			case <-ticker.C:
				logger.Info("worker tick")
			}
		}
	})

	// Wait for all goroutines to finish
	if err := g.Wait(); err != nil {
		logger.Error("application error", slog.String("error", err.Error()))
		os.Exit(1)
	}

	logger.Info("clean shutdown")
}
```

### The errgroup Advantage

With raw channels, you have to manage the channel buffer size, decide which error to surface, and coordinate cancellation manually. `errgroup` handles all of this:

1. If any goroutine returns a non-nil error, the group context is cancelled (all other goroutines see `ctx.Done()`)
2. `g.Wait()` returns the first non-nil error
3. All goroutines are guaranteed to have returned before `Wait()` returns

```
+-----------------------+-----------------------------+-----------------------------+
| Feature               | Raw Channels                | errgroup                    |
+-----------------------+-----------------------------+-----------------------------+
| Error propagation     | Manual channel send/receive | Automatic via return value  |
| Context cancellation  | Manual bgCancel()           | Automatic on first error    |
| Wait for completion   | Manual WaitGroup            | Built-in g.Wait()           |
| Multiple errors       | Must handle buffer/select   | First error wins            |
| Panic handling        | Must recover manually       | Must recover manually       |
+-----------------------+-----------------------------+-----------------------------+
```

---

## 14. Restart Resilience

### Idempotent Operations

Your application will be restarted. Containers crash, nodes fail, deployments roll out. Every operation your application performs on startup must be **idempotent** -- safe to run multiple times with the same result.

### Idempotent Migrations

```go
package main

import (
	"database/sql"
	"fmt"
	"log/slog"
)

// Migration represents a single database migration
type Migration struct {
	ID  string
	SQL string
}

// RunMigrations executes all pending migrations idempotently.
// It uses a migrations table to track which migrations have been applied.
func RunMigrations(db *sql.DB, logger *slog.Logger) error {
	// Create the migrations tracking table if it does not exist
	// This is itself idempotent (IF NOT EXISTS)
	_, err := db.Exec(`
		CREATE TABLE IF NOT EXISTS schema_migrations (
			id TEXT PRIMARY KEY,
			applied_at TIMESTAMPTZ DEFAULT NOW()
		)
	`)
	if err != nil {
		return fmt.Errorf("create migrations table: %w", err)
	}

	migrations := []Migration{
		{
			ID: "001_create_users",
			SQL: `CREATE TABLE IF NOT EXISTS users (
				id SERIAL PRIMARY KEY,
				email TEXT UNIQUE NOT NULL,
				password_hash TEXT NOT NULL,
				created_at TIMESTAMPTZ DEFAULT NOW(),
				updated_at TIMESTAMPTZ DEFAULT NOW()
			)`,
		},
		{
			ID: "002_create_tokens",
			SQL: `CREATE TABLE IF NOT EXISTS tokens (
				id SERIAL PRIMARY KEY,
				user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
				token_hash TEXT UNIQUE NOT NULL,
				expires_at TIMESTAMPTZ NOT NULL,
				created_at TIMESTAMPTZ DEFAULT NOW()
			)`,
		},
		{
			ID: "003_add_users_name",
			SQL: `ALTER TABLE users ADD COLUMN IF NOT EXISTS name TEXT DEFAULT ''`,
		},
	}

	for _, m := range migrations {
		// Check if this migration has already been applied
		var exists bool
		err := db.QueryRow(
			"SELECT EXISTS(SELECT 1 FROM schema_migrations WHERE id = $1)", m.ID,
		).Scan(&exists)
		if err != nil {
			return fmt.Errorf("check migration %s: %w", m.ID, err)
		}

		if exists {
			logger.Debug("migration already applied", slog.String("id", m.ID))
			continue
		}

		// Apply the migration in a transaction
		tx, err := db.Begin()
		if err != nil {
			return fmt.Errorf("begin transaction for %s: %w", m.ID, err)
		}

		if _, err := tx.Exec(m.SQL); err != nil {
			tx.Rollback()
			return fmt.Errorf("execute migration %s: %w", m.ID, err)
		}

		if _, err := tx.Exec(
			"INSERT INTO schema_migrations (id) VALUES ($1)", m.ID,
		); err != nil {
			tx.Rollback()
			return fmt.Errorf("record migration %s: %w", m.ID, err)
		}

		if err := tx.Commit(); err != nil {
			return fmt.Errorf("commit migration %s: %w", m.ID, err)
		}

		logger.Info("migration applied", slog.String("id", m.ID))
	}

	return nil
}

func main() {
	fmt.Println("Migration runner example")
}
```

### Database Reconnection

Database connections can drop for many reasons: network blip, database restart, connection pool recycling. Your application must handle reconnection gracefully.

Go's `database/sql` package handles reconnection automatically through its connection pool. When a connection fails, the pool marks it as dead and creates a new one on the next request. However, you should configure the pool to detect dead connections quickly:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log/slog"
	"os"
	"time"
)

func setupResilientDB(dsn string, logger *slog.Logger) (*sql.DB, error) {
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return nil, fmt.Errorf("open: %w", err)
	}

	// Connection pool configuration for resilience
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)

	// ConnMaxLifetime: Close connections after this duration.
	// This ensures that after a database failover, old connections
	// to the old primary are eventually replaced with connections
	// to the new primary.
	db.SetConnMaxLifetime(5 * time.Minute)

	// ConnMaxIdleTime: Close idle connections after this duration.
	// This reclaims resources after traffic spikes and ensures
	// stale connections are replaced.
	db.SetConnMaxIdleTime(1 * time.Minute)

	// Verify initial connectivity with retries
	var lastErr error
	for attempt := 1; attempt <= 5; attempt++ {
		ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		lastErr = db.PingContext(ctx)
		cancel()

		if lastErr == nil {
			logger.Info("database connected",
				slog.Int("attempt", attempt),
			)
			return db, nil
		}

		logger.Warn("database connection attempt failed",
			slog.Int("attempt", attempt),
			slog.String("error", lastErr.Error()),
		)

		// Exponential backoff: 1s, 2s, 4s, 8s, 16s
		backoff := time.Duration(1<<uint(attempt-1)) * time.Second
		time.Sleep(backoff)
	}

	db.Close()
	return nil, fmt.Errorf("database unreachable after 5 attempts: %w", lastErr)
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		dsn = "postgres://localhost/myapp?sslmode=disable"
	}

	db, err := setupResilientDB(dsn, logger)
	if err != nil {
		logger.Error("failed to connect to database",
			slog.String("error", err.Error()),
		)
		os.Exit(1)
	}
	defer db.Close()

	logger.Info("application ready")
}
```

### Startup Health Check Backoff

When your application starts, its dependencies might not be ready yet (especially in Kubernetes where pods start in parallel). Use exponential backoff for startup checks:

```go
package main

import (
	"context"
	"fmt"
	"math"
	"time"
)

// WaitForReady repeatedly calls check until it succeeds or maxAttempts is reached.
// Uses exponential backoff between attempts.
func WaitForReady(
	ctx context.Context,
	name string,
	check func(ctx context.Context) error,
	maxAttempts int,
	baseDelay time.Duration,
) error {
	for attempt := 1; attempt <= maxAttempts; attempt++ {
		checkCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
		err := check(checkCtx)
		cancel()

		if err == nil {
			return nil
		}

		if attempt == maxAttempts {
			return fmt.Errorf("%s not ready after %d attempts: %w", name, maxAttempts, err)
		}

		// Exponential backoff with a cap
		delay := time.Duration(math.Min(
			float64(baseDelay)*math.Pow(2, float64(attempt-1)),
			float64(30*time.Second),
		))

		fmt.Printf("%s not ready (attempt %d/%d), retrying in %s: %s\n",
			name, attempt, maxAttempts, delay, err)

		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(delay):
		}
	}

	return fmt.Errorf("%s: max attempts reached", name)
}

func main() {
	ctx := context.Background()

	// Example: Wait for database
	err := WaitForReady(ctx, "database", func(ctx context.Context) error {
		// db.PingContext(ctx)
		return nil // Simulated success
	}, 10, 1*time.Second)

	if err != nil {
		fmt.Printf("Startup failed: %s\n", err)
		return
	}

	fmt.Println("All dependencies ready")
}
```

### Idempotent Background Tasks

Background tasks must also be idempotent. If the process restarts mid-task, the task will run again:

```go
// BAD: Non-idempotent -- running twice sends duplicate emails
func sendWelcomeEmails(ctx context.Context, db *sql.DB) error {
    rows, _ := db.QueryContext(ctx, "SELECT id, email FROM users WHERE welcomed = false")
    for rows.Next() {
        // Send email
        // Mark as welcomed
        // If process crashes between these two steps, email is sent twice on restart
    }
    return nil
}

// GOOD: Idempotent -- uses a transaction to atomically mark and process
func sendWelcomeEmails(ctx context.Context, db *sql.DB) error {
    // Claim a batch of users atomically
    rows, _ := db.QueryContext(ctx, `
        UPDATE users
        SET welcome_status = 'processing'
        WHERE welcome_status = 'pending'
        AND id IN (SELECT id FROM users WHERE welcome_status = 'pending' LIMIT 100)
        RETURNING id, email
    `)
    // Process claimed users
    // If we crash, 'processing' users are retried on next run
    // (add a "claimed_at" timestamp and reclaim after a timeout)
    return nil
}
```

---

## 15. Real-World Example: Complete main.go

This is a complete, production-ready `main.go` that incorporates every pattern from this chapter. It is modeled after the krafty-core API structure.

```go
// cmd/server/main.go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"runtime"
	"sync"
	"sync/atomic"
	"syscall"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// =============================================================================
// Configuration
// =============================================================================

type Config struct {
	Port                string
	DatabaseURL         string
	LogLevel            string
	Environment         string
	ReadTimeout         time.Duration
	WriteTimeout        time.Duration
	IdleTimeout         time.Duration
	ShutdownTimeout     time.Duration
	DBMaxOpenConns      int
	DBMaxIdleConns      int
	DBConnMaxLifetime   time.Duration
	TokenCleanupInterval time.Duration
}

func LoadConfig() (*Config, error) {
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}

	return &Config{
		Port:                envOrDefault("PORT", "8080"),
		DatabaseURL:         dbURL,
		LogLevel:            envOrDefault("LOG_LEVEL", "info"),
		Environment:         envOrDefault("ENVIRONMENT", "development"),
		ReadTimeout:         15 * time.Second,
		WriteTimeout:        15 * time.Second,
		IdleTimeout:         60 * time.Second,
		ShutdownTimeout:     10 * time.Second,
		DBMaxOpenConns:      25,
		DBMaxIdleConns:      5,
		DBConnMaxLifetime:   5 * time.Minute,
		TokenCleanupInterval: 1 * time.Hour,
	}, nil
}

func envOrDefault(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

// =============================================================================
// Logger
// =============================================================================

func SetupLogger(level, environment string) *slog.Logger {
	var lvl slog.Level
	switch level {
	case "debug":
		lvl = slog.LevelDebug
	case "warn":
		lvl = slog.LevelWarn
	case "error":
		lvl = slog.LevelError
	default:
		lvl = slog.LevelInfo
	}

	opts := &slog.HandlerOptions{Level: lvl}

	var handler slog.Handler
	if environment == "development" {
		handler = slog.NewTextHandler(os.Stdout, opts)
	} else {
		handler = slog.NewJSONHandler(os.Stdout, opts)
	}

	return slog.New(handler)
}

// =============================================================================
// Database
// =============================================================================

func ConnectDatabase(cfg *Config, logger *slog.Logger) (*sql.DB, error) {
	db, err := sql.Open("pgx", cfg.DatabaseURL)
	if err != nil {
		return nil, fmt.Errorf("sql.Open: %w", err)
	}

	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	db.SetConnMaxIdleTime(1 * time.Minute)

	// Retry connection with exponential backoff
	var lastErr error
	for attempt := 1; attempt <= 5; attempt++ {
		ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		lastErr = db.PingContext(ctx)
		cancel()

		if lastErr == nil {
			logger.Info("database connected",
				slog.Int("maxOpenConns", cfg.DBMaxOpenConns),
				slog.Int("maxIdleConns", cfg.DBMaxIdleConns),
			)
			return db, nil
		}

		logger.Warn("database connection failed, retrying",
			slog.Int("attempt", attempt),
			slog.String("error", lastErr.Error()),
		)

		backoff := time.Duration(1<<uint(attempt-1)) * time.Second
		time.Sleep(backoff)
	}

	db.Close()
	return nil, fmt.Errorf("database unreachable after retries: %w", lastErr)
}

// =============================================================================
// Migrations
// =============================================================================

func RunMigrations(db *sql.DB, logger *slog.Logger) error {
	_, err := db.Exec(`
		CREATE TABLE IF NOT EXISTS schema_migrations (
			id TEXT PRIMARY KEY,
			applied_at TIMESTAMPTZ DEFAULT NOW()
		)
	`)
	if err != nil {
		return fmt.Errorf("create migrations table: %w", err)
	}

	migrations := []struct {
		id  string
		sql string
	}{
		{
			id: "001_create_users",
			sql: `CREATE TABLE IF NOT EXISTS users (
				id SERIAL PRIMARY KEY,
				email TEXT UNIQUE NOT NULL,
				password_hash TEXT NOT NULL,
				name TEXT DEFAULT '',
				created_at TIMESTAMPTZ DEFAULT NOW(),
				updated_at TIMESTAMPTZ DEFAULT NOW()
			)`,
		},
		{
			id: "002_create_tokens",
			sql: `CREATE TABLE IF NOT EXISTS tokens (
				id SERIAL PRIMARY KEY,
				user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
				token_hash TEXT UNIQUE NOT NULL,
				expires_at TIMESTAMPTZ NOT NULL,
				created_at TIMESTAMPTZ DEFAULT NOW()
			)`,
		},
	}

	for _, m := range migrations {
		var exists bool
		if err := db.QueryRow(
			"SELECT EXISTS(SELECT 1 FROM schema_migrations WHERE id = $1)", m.id,
		).Scan(&exists); err != nil {
			return fmt.Errorf("check migration %s: %w", m.id, err)
		}

		if exists {
			continue
		}

		tx, err := db.Begin()
		if err != nil {
			return fmt.Errorf("begin tx for %s: %w", m.id, err)
		}

		if _, err := tx.Exec(m.sql); err != nil {
			tx.Rollback()
			return fmt.Errorf("migration %s: %w", m.id, err)
		}

		if _, err := tx.Exec(
			"INSERT INTO schema_migrations (id) VALUES ($1)", m.id,
		); err != nil {
			tx.Rollback()
			return fmt.Errorf("record migration %s: %w", m.id, err)
		}

		if err := tx.Commit(); err != nil {
			return fmt.Errorf("commit migration %s: %w", m.id, err)
		}

		logger.Info("migration applied", slog.String("id", m.id))
	}

	return nil
}

// =============================================================================
// Repository Layer
// =============================================================================

type UserRepository struct {
	db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

type User struct {
	ID        int       `json:"id"`
	Email     string    `json:"email"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"created_at"`
}

func (r *UserRepository) List(ctx context.Context, limit, offset int) ([]User, error) {
	rows, err := r.db.QueryContext(ctx,
		"SELECT id, email, name, created_at FROM users ORDER BY id LIMIT $1 OFFSET $2",
		limit, offset,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var users []User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Email, &u.Name, &u.CreatedAt); err != nil {
			return nil, err
		}
		users = append(users, u)
	}

	return users, rows.Err()
}

type TokenRepository struct {
	db *sql.DB
}

func NewTokenRepository(db *sql.DB) *TokenRepository {
	return &TokenRepository{db: db}
}

func (r *TokenRepository) DeleteExpired(ctx context.Context) (int, error) {
	result, err := r.db.ExecContext(ctx,
		"DELETE FROM tokens WHERE expires_at < NOW()",
	)
	if err != nil {
		return 0, err
	}
	n, err := result.RowsAffected()
	return int(n), err
}

// =============================================================================
// Handler Layer
// =============================================================================

type Handler struct {
	userRepo  *UserRepository
	tokenRepo *TokenRepository
	db        *sql.DB
	logger    *slog.Logger
	ready     atomic.Bool
}

func NewHandler(
	userRepo *UserRepository,
	tokenRepo *TokenRepository,
	db *sql.DB,
	logger *slog.Logger,
) *Handler {
	return &Handler{
		userRepo:  userRepo,
		tokenRepo: tokenRepo,
		db:        db,
		logger:    logger,
	}
}

func (h *Handler) SetReady(ready bool) {
	h.ready.Store(ready)
}

func (h *Handler) Liveness(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]interface{}{
		"status":     "alive",
		"goroutines": runtime.NumGoroutine(),
	})
}

func (h *Handler) Readiness(w http.ResponseWriter, r *http.Request) {
	if !h.ready.Load() {
		writeJSON(w, http.StatusServiceUnavailable, map[string]string{
			"status": "not ready",
		})
		return
	}

	ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer cancel()

	if err := h.db.PingContext(ctx); err != nil {
		h.logger.Warn("readiness check failed", slog.String("error", err.Error()))
		writeJSON(w, http.StatusServiceUnavailable, map[string]string{
			"status": "not ready",
			"error":  "database unreachable",
		})
		return
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "ready"})
}

func (h *Handler) Health(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer cancel()

	if err := h.db.PingContext(ctx); err != nil {
		writeJSON(w, http.StatusServiceUnavailable, map[string]string{
			"status": "unhealthy",
			"error":  "database unreachable",
		})
		return
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "healthy"})
}

func (h *Handler) ListUsers(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	users, err := h.userRepo.List(ctx, 50, 0)
	if err != nil {
		h.logger.Error("failed to list users", slog.String("error", err.Error()))
		writeJSON(w, http.StatusInternalServerError, map[string]string{
			"error": "internal server error",
		})
		return
	}

	writeJSON(w, http.StatusOK, map[string]interface{}{
		"users": users,
		"count": len(users),
	})
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// =============================================================================
// Middleware
// =============================================================================

func RequestLogging(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// Wrap ResponseWriter to capture status code
			wrapped := &statusWriter{ResponseWriter: w, status: http.StatusOK}
			next.ServeHTTP(wrapped, r)

			logger.Info("request",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", wrapped.status),
				slog.Duration("duration", time.Since(start)),
				slog.String("remote", r.RemoteAddr),
			)
		})
	}
}

type statusWriter struct {
	http.ResponseWriter
	status int
}

func (w *statusWriter) WriteHeader(status int) {
	w.status = status
	w.ResponseWriter.WriteHeader(status)
}

func Recovery(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if rec := recover(); rec != nil {
					logger.Error("panic recovered",
						slog.Any("panic", rec),
						slog.String("method", r.Method),
						slog.String("path", r.URL.Path),
					)
					writeJSON(w, http.StatusInternalServerError, map[string]string{
						"error": "internal server error",
					})
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================================
// Router
// =============================================================================

func SetupRouter(h *Handler, logger *slog.Logger) http.Handler {
	mux := http.NewServeMux()

	// Health endpoints (no middleware -- must be fast and reliable)
	mux.HandleFunc("GET /healthz", h.Liveness)
	mux.HandleFunc("GET /readyz", h.Readiness)
	mux.HandleFunc("GET /health", h.Health)

	// API endpoints
	mux.HandleFunc("GET /api/v1/users", h.ListUsers)

	// Apply middleware stack (outermost first)
	var handler http.Handler = mux
	handler = RequestLogging(logger)(handler)
	handler = Recovery(logger)(handler)

	return handler
}

// =============================================================================
// Background Tasks
// =============================================================================

func StartTokenCleanup(
	ctx context.Context,
	wg *sync.WaitGroup,
	repo *TokenRepository,
	interval time.Duration,
	logger *slog.Logger,
) {
	wg.Add(1)
	go func() {
		defer wg.Done()
		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		logger.Info("background: token cleanup started",
			slog.String("interval", interval.String()),
		)

		for {
			select {
			case <-ctx.Done():
				logger.Info("background: token cleanup stopped")
				return
			case <-ticker.C:
				cleanupCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
				deleted, err := repo.DeleteExpired(cleanupCtx)
				cancel()

				if err != nil {
					if ctx.Err() != nil {
						return // Parent context cancelled, exit cleanly
					}
					logger.Error("background: token cleanup failed",
						slog.String("error", err.Error()),
					)
					continue
				}

				if deleted > 0 {
					logger.Info("background: expired tokens removed",
						slog.Int("deleted", deleted),
					)
				}
			}
		}
	}()
}

// =============================================================================
// Application Entry Point
// =============================================================================

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "fatal: %s\n", err)
		os.Exit(1)
	}
}

func run() error {
	// -----------------------------------------------------------------
	// Step 1: Load configuration
	// -----------------------------------------------------------------
	cfg, err := LoadConfig()
	if err != nil {
		return fmt.Errorf("config: %w", err)
	}

	// -----------------------------------------------------------------
	// Step 2: Setup structured logger
	// -----------------------------------------------------------------
	logger := SetupLogger(cfg.LogLevel, cfg.Environment)
	logger.Info("starting krafty-core API",
		slog.String("environment", cfg.Environment),
		slog.String("port", cfg.Port),
		slog.String("go_version", runtime.Version()),
	)

	// -----------------------------------------------------------------
	// Step 3: Connect to database
	// -----------------------------------------------------------------
	db, err := ConnectDatabase(cfg, logger)
	if err != nil {
		return fmt.Errorf("database: %w", err)
	}
	defer func() {
		logger.Info("closing database connection pool")
		db.Close()
	}()

	// -----------------------------------------------------------------
	// Step 4: Run database migrations
	// -----------------------------------------------------------------
	if err := RunMigrations(db, logger); err != nil {
		return fmt.Errorf("migrations: %w", err)
	}

	// -----------------------------------------------------------------
	// Step 5: Create repositories
	// -----------------------------------------------------------------
	userRepo := NewUserRepository(db)
	tokenRepo := NewTokenRepository(db)

	// -----------------------------------------------------------------
	// Step 6: Create handler
	// -----------------------------------------------------------------
	handler := NewHandler(userRepo, tokenRepo, db, logger)

	// -----------------------------------------------------------------
	// Step 7: Setup router with middleware
	// -----------------------------------------------------------------
	router := SetupRouter(handler, logger)

	// -----------------------------------------------------------------
	// Step 8: Create and configure HTTP server
	// -----------------------------------------------------------------
	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       cfg.ReadTimeout,
		WriteTimeout:      cfg.WriteTimeout,
		IdleTimeout:       cfg.IdleTimeout,
	}

	// -----------------------------------------------------------------
	// Step 9: Start background tasks
	// -----------------------------------------------------------------
	bgCtx, bgCancel := context.WithCancel(context.Background())
	defer bgCancel()

	var bgWg sync.WaitGroup
	StartTokenCleanup(bgCtx, &bgWg, tokenRepo, cfg.TokenCleanupInterval, logger)

	// -----------------------------------------------------------------
	// Step 10: Start HTTP server in a goroutine
	// -----------------------------------------------------------------
	errChan := make(chan error, 1)
	go func() {
		logger.Info("HTTP server listening",
			slog.String("addr", srv.Addr),
		)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			errChan <- err
		}
	}()

	// -----------------------------------------------------------------
	// Step 11: Mark as ready (readiness probe will pass)
	// -----------------------------------------------------------------
	handler.SetReady(true)
	logger.Info("application ready -- accepting traffic")

	// -----------------------------------------------------------------
	// Step 12: Wait for shutdown signal or fatal error
	// -----------------------------------------------------------------
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case sig := <-quit:
		logger.Info("shutdown signal received",
			slog.String("signal", sig.String()),
		)
	case err := <-errChan:
		return fmt.Errorf("server error: %w", err)
	}

	// =================================================================
	// GRACEFUL SHUTDOWN SEQUENCE
	// =================================================================
	logger.Info("=== graceful shutdown initiated ===")

	// Step A: Fail readiness probe immediately
	handler.SetReady(false)
	logger.Info("readiness probe: failing (not ready)")

	// Step B: Wait for load balancer to drain traffic
	// (Give Kubernetes endpoint controller time to remove us)
	logger.Info("waiting for load balancer drain (5s)")
	time.Sleep(5 * time.Second)

	// Step C: Stop accepting new connections, drain in-flight requests
	shutdownCtx, shutdownCancel := context.WithTimeout(
		context.Background(), cfg.ShutdownTimeout,
	)
	defer shutdownCancel()

	srv.SetKeepAlivesEnabled(false)
	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("HTTP server shutdown error",
			slog.String("error", err.Error()),
		)
		// Force close as fallback
		srv.Close()
	}
	logger.Info("HTTP server stopped")

	// Step D: Cancel and wait for background tasks
	bgCancel()

	bgDone := make(chan struct{})
	go func() {
		bgWg.Wait()
		close(bgDone)
	}()

	select {
	case <-bgDone:
		logger.Info("all background tasks stopped")
	case <-time.After(5 * time.Second):
		logger.Warn("background tasks did not stop within timeout")
	}

	// Step E: Database connection pool closes via defer db.Close()

	logger.Info("=== shutdown complete ===")
	return nil
}
```

### Running This Example

```bash
# Set up environment
export DATABASE_URL="postgres://postgres:secret@localhost:5432/krafty?sslmode=disable"
export PORT=8080
export LOG_LEVEL=info
export ENVIRONMENT=production

# Build and run
go build -o krafty-api ./cmd/server
./krafty-api

# In another terminal, test it:
curl http://localhost:8080/healthz     # Liveness
curl http://localhost:8080/readyz      # Readiness
curl http://localhost:8080/health      # Combined health
curl http://localhost:8080/api/v1/users # API endpoint

# Graceful shutdown: send SIGTERM
kill -TERM $(pgrep krafty-api)

# Or press Ctrl+C for SIGINT
```

### Expected Shutdown Logs

```json
{"time":"2026-03-24T10:00:00Z","level":"INFO","msg":"shutdown signal received","signal":"interrupt"}
{"time":"2026-03-24T10:00:00Z","level":"INFO","msg":"=== graceful shutdown initiated ==="}
{"time":"2026-03-24T10:00:00Z","level":"INFO","msg":"readiness probe: failing (not ready)"}
{"time":"2026-03-24T10:00:00Z","level":"INFO","msg":"waiting for load balancer drain (5s)"}
{"time":"2026-03-24T10:00:05Z","level":"INFO","msg":"HTTP server stopped"}
{"time":"2026-03-24T10:00:05Z","level":"INFO","msg":"background: token cleanup stopped"}
{"time":"2026-03-24T10:00:05Z","level":"INFO","msg":"all background tasks stopped"}
{"time":"2026-03-24T10:00:05Z","level":"INFO","msg":"closing database connection pool"}
{"time":"2026-03-24T10:00:05Z","level":"INFO","msg":"=== shutdown complete ==="}
```

### Production Dockerfile for This Example

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -trimpath -o /krafty-api ./cmd/server

# Run stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /krafty-api .
USER app
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/healthz || exit 1
CMD ["./krafty-api"]
```

---

## 16. Key Takeaways

### The Non-Negotiables

1. **Always handle SIGTERM and SIGINT.** In production, your process will receive these signals. If you ignore them, you get SIGKILL after the grace period -- no cleanup, no connection draining, no log flushing.

2. **Always use `srv.Shutdown(ctx)`, never `srv.Close()`.** `Shutdown` drains connections gracefully. `Close` kills them immediately. Use `Close` only as a fallback when `Shutdown` times out.

3. **Always set server timeouts.** Without `ReadHeaderTimeout`, you are vulnerable to slowloris. Without `WriteTimeout`, a slow client can hold a goroutine forever. Without `IdleTimeout`, keep-alive connections accumulate.

4. **Always use buffered signal channels.** `make(chan os.Signal, 1)` -- not `make(chan os.Signal)`. An unbuffered channel can miss the signal.

5. **Always manage background goroutine lifecycle.** Use `context.WithCancel` and `sync.WaitGroup` to ensure all goroutines stop cleanly before the process exits.

### The Shutdown Order

```
Stop accepting HTTP traffic
    → Drain in-flight HTTP requests
        → Cancel background tasks
            → Wait for background tasks to finish
                → Close database connections
                    → Flush logs
                        → Exit
```

Never close a resource that something else is still using. The dependency graph determines the teardown order -- it is the reverse of the startup order.

### The Startup Pattern

```go
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "fatal: %s\n", err)
        os.Exit(1)
    }
}
```

The `run()` pattern gives you proper defer execution, testability, and a single exit path for errors.

### Health Checks

- **Liveness** = "am I alive?" (never check dependencies)
- **Readiness** = "can I handle traffic?" (check all dependencies)
- **Set readiness to false before shutdown** to stop receiving traffic

### Docker

- **Always use exec form** for CMD: `CMD ["./server"]`
- **Multi-stage builds** for minimal images
- **CGO_ENABLED=0** for static binaries

### Kubernetes

- **PreStop hook** with `sleep 5` to handle endpoint propagation delay
- **terminationGracePeriodSeconds** > your shutdown timeout
- **Pod Disruption Budget** to prevent simultaneous pod terminations

### Comparison Summary: Go vs Node.js

```
+---------------------------+----------------------------------+------------------------------+
| Concern                   | Go                               | Node.js                      |
+---------------------------+----------------------------------+------------------------------+
| Graceful HTTP shutdown    | srv.Shutdown() -- built in       | server.close() -- incomplete |
| Keep-alive draining       | Handled by Shutdown              | Needs http-terminator lib    |
| Signal handling           | Buffered channel, never missed   | process.on(), can throw      |
| Background task cancel    | Context cancellation propagates  | No built-in mechanism        |
| DB query cancellation     | Context-aware queries            | No standard approach         |
| PID 1 in Docker           | Works correctly as PID 1         | Needs tini or dumb-init      |
| Server timeouts           | 4 independent timeout settings   | 2 settings, less granular    |
| Health check              | Standard library HTTP handler    | Express middleware           |
| Connection pool stats     | db.Stats() built in              | Depends on driver            |
| Binary deployment         | Single static binary             | Runtime + node_modules       |
+---------------------------+----------------------------------+------------------------------+
```

---

## 17. Practice Exercises

### Exercise 1: Basic Graceful Shutdown

Build an HTTP server that:
- Serves `GET /` with a 3-second delay (simulating slow work)
- Handles SIGINT and SIGTERM
- Shuts down gracefully with a 10-second timeout
- Logs each phase of shutdown

**Test it:** Start the server, make a request with `curl http://localhost:8080/`, and immediately press Ctrl+C. Verify that the request completes successfully before the server exits.

### Exercise 2: Multiple Background Tasks

Create a program that runs three background tasks:
1. A "heartbeat" task that prints every 5 seconds
2. A "cleanup" task that prints every 15 seconds
3. A "sync" task that prints every 30 seconds

All tasks must:
- Respect context cancellation
- Be tracked with a WaitGroup
- Stop gracefully on SIGTERM
- Log when they start and stop

**Test it:** Run the program for 1 minute, then send SIGTERM. Verify all three tasks log their stop message.

### Exercise 3: Health Check Server

Build a service with three endpoints:
- `GET /healthz` -- always returns 200 (liveness)
- `GET /readyz` -- returns 200 only when ready, 503 otherwise (readiness)
- `GET /api/data` -- returns some JSON data

Implement the ready flag pattern:
1. Start with ready=false
2. Simulate initialization (sleep 5 seconds)
3. Set ready=true
4. On shutdown, set ready=false, wait 3 seconds, then shut down

**Test it:** Immediately after starting, curl `/readyz` and verify it returns 503. Wait 5 seconds and verify it returns 200. Send SIGTERM and verify `/readyz` returns 503 before the server shuts down.

### Exercise 4: Error Channel Coordination

Build a program that starts:
1. An HTTP server on port 8080
2. A metrics server on port 9090
3. A background worker goroutine that periodically does work

Use error channels so that:
- If the HTTP server fails to start (try using port 8080 when it is already in use), the entire program shuts down
- If the worker encounters a "fatal" error (simulate this), the entire program shuts down
- If SIGTERM is received, everything shuts down gracefully

### Exercise 5: Production main.go

Using the patterns from this chapter, build a complete `main.go` for a TODO API with:
- Configuration from environment variables
- Structured JSON logging with slog
- SQLite database (for simplicity) with proper pool configuration
- Idempotent migrations for a `todos` table
- A repository layer with `List`, `Create`, `Complete`, and `Delete` methods
- Health check endpoints (liveness and readiness)
- Request logging and panic recovery middleware
- Background task: delete completed TODOs older than 30 days (every hour)
- Full graceful shutdown with proper ordering
- The `run()` pattern

This exercise combines everything from this chapter into a single, deployable application.

### Exercise 6: Dockerfile

Write a production Dockerfile for Exercise 5 that:
- Uses multi-stage build
- Produces an image under 20 MB
- Uses exec form for CMD
- Runs as a non-root user
- Includes a Docker HEALTHCHECK

### Exercise 7: Kubernetes Manifests

Write Kubernetes manifests for Exercise 5 that include:
- Deployment with rolling update strategy
- Liveness, readiness, and startup probes
- PreStop hook
- Service
- Pod Disruption Budget
- Resource requests and limits

---

**Next Chapter:** [Chapter 31: Project Structure & Clean Architecture](/golang/31-project-structure-and-clean-architecture.md) -- How to organize a Go project with clear boundaries between layers, dependency injection, and testable architecture.
