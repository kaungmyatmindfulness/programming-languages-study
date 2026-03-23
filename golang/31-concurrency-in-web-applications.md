# Chapter 31: Concurrency in Web Applications

> **Level: Applied / Production** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 16 (Context), 17 (Advanced Concurrency), 21 (Building Microservices)**

This chapter is not theory. Chapters 10, 11, 16, and 17 taught you what goroutines, channels, mutexes, WaitGroups, errgroup, and context are. This chapter teaches you **where they appear in a real production Go API** -- the kind you deploy to a server, point a domain at, and let thousands of users hit concurrently.

Every pattern in this chapter is drawn from real-world Go API projects. We will build actual middleware, actual background workers, actual graceful shutdown sequences. If you have built Express.js APIs in Node.js, the comparisons will be explicit and frequent. If you have not, the Go patterns stand entirely on their own.

By the end of this chapter you will understand how concurrency is woven into every layer of a Go web application -- from the moment a TCP connection arrives to the moment the server shuts down cleanly.

---

## Table of Contents

1. [Concurrency in Real APIs -- Go vs Node.js](#1-concurrency-in-real-apis----go-vs-nodejs)
2. [Per-IP Rate Limiter with sync.Mutex](#2-per-ip-rate-limiter-with-syncmutex)
3. [Background Cleanup with Tickers](#3-background-cleanup-with-tickers)
4. [Context Cancellation in HTTP Handlers](#4-context-cancellation-in-http-handlers)
5. [Database Connection Pool Concurrency](#5-database-connection-pool-concurrency)
6. [Parallel Database Queries with errgroup](#6-parallel-database-queries-with-errgroup)
7. [Background Token Cleanup](#7-background-token-cleanup)
8. [Concurrent Request Processing -- How Go's HTTP Server Works](#8-concurrent-request-processing----how-gos-http-server-works)
9. [Shared State Protection Patterns -- Mutex vs Channel vs Atomic](#9-shared-state-protection-patterns----mutex-vs-channel-vs-atomic)
10. [sync.Once for Lazy Initialization](#10-synconce-for-lazy-initialization)
11. [Worker Pool for Background Jobs](#11-worker-pool-for-background-jobs)
12. [Graceful Goroutine Shutdown](#12-graceful-goroutine-shutdown)
13. [Race Condition Debugging](#13-race-condition-debugging)
14. [Real-World Concurrency Bugs](#14-real-world-concurrency-bugs)
15. [Complete Working System: Rate Limiter + Background Cleanup + Parallel Queries](#15-complete-working-system-rate-limiter--background-cleanup--parallel-queries)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. Concurrency in Real APIs -- Go vs Node.js

### The Fundamental Difference

When a request arrives at a Go HTTP server, the runtime spawns a **new goroutine** to handle it. When a request arrives at an Express.js server, it runs on the **single event loop thread**. This difference is not academic -- it changes how you think about every line of code in your handler.

```
Go HTTP Server                          Node.js / Express Server
─────────────────                       ─────────────────────────

Request 1 ──► goroutine 1              Request 1 ──► event loop
Request 2 ──► goroutine 2              Request 2 ──► event loop (waits)
Request 3 ──► goroutine 3              Request 3 ──► event loop (waits)
   ...           ...                       ...           ...
Request N ──► goroutine N              Request N ──► event loop (waits)

Each goroutine:                         The event loop:
 - Has its own stack (~2KB initial)      - Runs ONE callback at a time
 - Can block on I/O (runtime parks it)   - Must NEVER block (or all requests stall)
 - Scheduled by Go runtime (M:N)         - Delegates I/O to libuv thread pool
 - Can use CPU freely                    - CPU-bound work blocks EVERYTHING
```

### What This Means in Practice

In Node.js, you must be careful never to block the event loop. A CPU-intensive operation -- hashing a password, parsing a large JSON, running a regex on user input -- blocks every other request. You offload to worker threads or external processes.

In Go, blocking is fine. Every handler runs in its own goroutine. If one goroutine blocks on a database query, the runtime parks it and runs other goroutines on the same OS thread. Thousands of goroutines can be blocked on I/O simultaneously, and the server keeps humming.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"net/http"
	"time"
)

// In Go, this handler can do CPU-intensive work without worrying about
// blocking other requests. Each request gets its own goroutine.
func hashHandler(w http.ResponseWriter, r *http.Request) {
	data := r.URL.Query().Get("data")
	if data == "" {
		data = "default"
	}

	// CPU-intensive: hash the data 100,000 times
	// In Node.js, this would block the event loop and freeze ALL requests.
	// In Go, only this goroutine is busy. Other requests proceed normally.
	result := []byte(data)
	for i := 0; i < 100_000; i++ {
		hash := sha256.Sum256(result)
		result = hash[:]
	}

	w.Header().Set("Content-Type", "text/plain")
	fmt.Fprintf(w, "Hash: %s\n", hex.EncodeToString(result))
}

// This handler simulates a slow database query.
// In Go, the goroutine is parked during the sleep. Other goroutines run freely.
// In Node.js, you'd use await -- but the callback scheduling still goes
// through the single event loop.
func slowQueryHandler(w http.ResponseWriter, r *http.Request) {
	// Simulates a 2-second database query
	time.Sleep(2 * time.Second)
	fmt.Fprintln(w, "Query complete")
}

func main() {
	http.HandleFunc("/hash", hashHandler)
	http.HandleFunc("/slow", slowQueryHandler)

	fmt.Println("Server running on :8080")
	http.ListenAndServe(":8080", nil)
}
```

### The Node.js Equivalent (For Comparison)

```javascript
// Node.js / Express -- CPU-bound work blocks the event loop
const express = require('express');
const crypto = require('crypto');
const app = express();

// DANGER: This blocks the event loop. While this runs,
// NO other request can be processed.
app.get('/hash', (req, res) => {
    let result = Buffer.from(req.query.data || 'default');
    for (let i = 0; i < 100000; i++) {
        result = crypto.createHash('sha256').update(result).digest();
    }
    res.send(`Hash: ${result.toString('hex')}`);
});

// This is fine in Node.js -- setTimeout is non-blocking.
// But the callback still runs on the event loop.
app.get('/slow', async (req, res) => {
    await new Promise(resolve => setTimeout(resolve, 2000));
    res.send('Query complete');
});

app.listen(8080);
```

### Where Concurrency Appears in a Go API

Here is every place concurrency plays a role in a typical production Go API. Each of these will be covered in detail in this chapter:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Go API Concurrency Map                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INCOMING REQUEST                                               │
│  ├── net/http spawns a goroutine per request                    │
│  ├── Rate limiter middleware (sync.Mutex on shared map)         │
│  ├── Auth middleware (reads shared config, sync.Once)           │
│  │                                                              │
│  HANDLER LAYER                                                  │
│  ├── Context from request (cancellation if client disconnects)  │
│  ├── Parallel DB queries (errgroup)                             │
│  ├── Database connection pool (pgx pool, semaphore-like)        │
│  │                                                              │
│  BACKGROUND GOROUTINES                                          │
│  ├── Rate limiter cleanup (time.Ticker + goroutine)             │
│  ├── Expired token cleanup (time.Ticker + goroutine)            │
│  ├── Worker pool for async jobs (channels + goroutines)         │
│  │                                                              │
│  SHUTDOWN                                                       │
│  ├── Signal handling (os.Signal channel)                        │
│  ├── HTTP server graceful shutdown (context + timeout)          │
│  ├── Background goroutine shutdown (context cancel + WaitGroup) │
│  └── Database pool close                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Per-IP Rate Limiter with sync.Mutex

### The Problem

Every production API needs rate limiting. Without it, a single client can overwhelm your server, scrape your data, or brute-force your login endpoint. The standard approach: limit each IP address to N requests per second.

In Node.js, you typically use a package like `express-rate-limit` backed by Redis. In Go, you can build a performant in-memory rate limiter with the standard library and `golang.org/x/time/rate`.

### Why This Needs Concurrency Primitives

The rate limiter maintains a **shared map** of IP addresses to rate limiters. Every incoming request (running in its own goroutine) reads and writes this map concurrently. Without synchronization, you get a **race condition** -- specifically, a concurrent map read/write panic that will crash your server.

### The Implementation

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"

	"golang.org/x/time/rate"
)

// visitor holds the rate limiter and last seen time for a single IP.
type visitor struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

// RateLimiter provides per-IP rate limiting for HTTP handlers.
// It is safe for concurrent use -- every field access is protected by mu.
type RateLimiter struct {
	mu       sync.Mutex
	visitors map[string]*visitor
	rate     rate.Limit
	burst    int
}

// NewRateLimiter creates a rate limiter that allows r requests per second
// with a burst capacity of b.
func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
	return &RateLimiter{
		visitors: make(map[string]*visitor),
		rate:     r,
		burst:    b,
	}
}

// Allow checks whether the given IP is allowed to make a request.
// It creates a new rate.Limiter for first-time visitors.
//
// WHY sync.Mutex and not sync.RWMutex:
// Every call to Allow writes to the map (updating lastSeen), so there
// are no read-only calls. RWMutex would provide no benefit.
func (rl *RateLimiter) Allow(ip string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	v, exists := rl.visitors[ip]
	if !exists {
		v = &visitor{
			limiter: rate.NewLimiter(rl.rate, rl.burst),
		}
		rl.visitors[ip] = v
	}
	v.lastSeen = time.Now()
	return v.limiter.Allow()
}

// Middleware returns an HTTP middleware that enforces the rate limit.
func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Extract IP from RemoteAddr. In production, you would also
		// check X-Forwarded-For or X-Real-IP when behind a reverse proxy.
		ip := r.RemoteAddr

		if !rl.Allow(ip) {
			http.Error(w, `{"error": "rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}

		next.ServeHTTP(w, r)
	})
}

func main() {
	// Allow 5 requests per second with a burst of 10.
	limiter := NewRateLimiter(5, 10)

	mux := http.NewServeMux()
	mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, `{"message": "here is your data"}`)
	})

	// Wrap the mux with the rate limiter middleware.
	server := &http.Server{
		Addr:    ":8080",
		Handler: limiter.Middleware(mux),
	}

	log.Println("Server starting on :8080 (5 req/s per IP)")
	log.Fatal(server.ListenAndServe())
}
```

### Understanding rate.Limiter

The `rate.Limiter` from `golang.org/x/time/rate` implements a **token bucket** algorithm:

```
Token Bucket Algorithm
──────────────────────

Bucket capacity (burst): 10 tokens
Refill rate: 5 tokens/second

 Time 0.0s:  [●●●●●●●●●●]  10/10 tokens (full)
 Request  →  [●●●●●●●●● ]   9/10 tokens (allowed)
 Request  →  [●●●●●●●●  ]   8/10 tokens (allowed)
 ...8 more →  [          ]   0/10 tokens (allowed -- used all burst)
 Request  →  DENIED (0 tokens, must wait)

 Time 0.2s:  [●         ]   1/10 tokens (refilled 1 token after 0.2s)
 Request  →  [          ]   0/10 tokens (allowed)
 Request  →  DENIED

 Time 1.0s:  [●●●●●     ]   5/10 tokens (refilled 5 tokens in 1s)
```

The key insight: `burst` controls how many requests can happen in a quick burst, while `rate` controls the sustained throughput. For API rate limiting:

- **rate = 5:** sustained 5 requests per second
- **burst = 10:** allows a burst of 10 rapid requests (like a page load that fires multiple API calls)

### How the Mutex Protects the Map

```
Goroutine 1 (Request from 1.2.3.4)     Goroutine 2 (Request from 5.6.7.8)
─────────────────────────────────────   ─────────────────────────────────────
rl.mu.Lock()    ← acquires lock
                                        rl.mu.Lock()    ← BLOCKS (waits)
v = rl.visitors["1.2.3.4"]
v.lastSeen = time.Now()
v.limiter.Allow()
rl.mu.Unlock()  ← releases lock
                                        ← unblocked, acquires lock
                                        v = rl.visitors["5.6.7.8"]
                                        v.lastSeen = time.Now()
                                        v.limiter.Allow()
                                        rl.mu.Unlock()
```

Without the mutex, both goroutines could try to write to the map simultaneously, triggering Go's built-in concurrent map write detection:

```
fatal error: concurrent map read and map write
```

This is not a race condition you can sometimes get away with -- Go deliberately crashes the program.

### Node.js Comparison

```javascript
// Node.js: No mutex needed because the event loop is single-threaded.
// But you lose the ability to handle requests concurrently.
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 1000,      // 1 second
    max: 5,              // 5 requests per window per IP
    standardHeaders: true,
    legacyHeaders: false,
});

app.use(limiter);

// For production with multiple Node.js processes (cluster mode),
// you need an external store like Redis:
const RedisStore = require('rate-limit-redis');
const limiter = rateLimit({
    store: new RedisStore({ client: redisClient }),
    windowMs: 1000,
    max: 5,
});
```

Go's approach is simpler for single-server deployments: everything is in-process, no Redis dependency. For multi-server deployments, you would also use Redis in Go, but the in-memory approach works well for many production workloads.

---

## 3. Background Cleanup with Tickers

### The Problem

The rate limiter from Section 2 has a memory leak. Every unique IP address that hits your API adds an entry to the map. Over time -- especially if you are behind a mobile network where IPs rotate frequently -- this map grows without bound.

The solution: a background goroutine that periodically removes stale entries.

### The Implementation

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"

	"golang.org/x/time/rate"
)

type visitor struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

type RateLimiter struct {
	mu       sync.Mutex
	visitors map[string]*visitor
	rate     rate.Limit
	burst    int
}

func NewRateLimiter(ctx context.Context, r rate.Limit, b int) *RateLimiter {
	rl := &RateLimiter{
		visitors: make(map[string]*visitor),
		rate:     r,
		burst:    b,
	}

	// Start the background cleanup goroutine.
	// It will stop when ctx is cancelled (typically on server shutdown).
	go rl.startCleanup(ctx, 1*time.Minute)

	return rl
}

func (rl *RateLimiter) Allow(ip string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	v, exists := rl.visitors[ip]
	if !exists {
		v = &visitor{
			limiter: rate.NewLimiter(rl.rate, rl.burst),
		}
		rl.visitors[ip] = v
	}
	v.lastSeen = time.Now()
	return v.limiter.Allow()
}

// startCleanup runs in a background goroutine. It removes visitors that
// have not been seen in the last 3 minutes.
//
// KEY CONCURRENCY PATTERNS:
// 1. time.Ticker for periodic work (not time.Sleep -- Ticker is self-correcting)
// 2. select with ctx.Done() for graceful shutdown
// 3. Same mutex used for cleanup as for Allow() -- no deadlock because
//    we never hold the lock across a blocking operation
func (rl *RateLimiter) startCleanup(ctx context.Context, interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop() // Stop the ticker when the goroutine exits

	for {
		select {
		case <-ctx.Done():
			// Server is shutting down. Exit the goroutine cleanly.
			log.Println("Rate limiter cleanup goroutine stopped")
			return
		case <-ticker.C:
			// Time to clean up stale visitors.
			rl.mu.Lock()
			before := len(rl.visitors)
			for ip, v := range rl.visitors {
				if time.Since(v.lastSeen) > 3*time.Minute {
					delete(rl.visitors, ip)
				}
			}
			after := len(rl.visitors)
			rl.mu.Unlock()

			if before != after {
				log.Printf("Rate limiter cleanup: removed %d stale visitors (%d remaining)",
					before-after, after)
			}
		}
	}
}

// VisitorCount returns the current number of tracked visitors.
// Useful for metrics/monitoring.
func (rl *RateLimiter) VisitorCount() int {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	return len(rl.visitors)
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ip := r.RemoteAddr
		if !rl.Allow(ip) {
			http.Error(w, `{"error": "rate limit exceeded"}`, http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func main() {
	// Create a context that we cancel on shutdown.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	limiter := NewRateLimiter(ctx, 5, 10)

	mux := http.NewServeMux()
	mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, `{"visitors": %d}`, limiter.VisitorCount())
	})

	server := &http.Server{
		Addr:    ":8080",
		Handler: limiter.Middleware(mux),
	}

	log.Println("Server starting on :8080")
	log.Fatal(server.ListenAndServe())
}
```

### Why time.Ticker, Not time.Sleep

```go
// BAD: time.Sleep drifts. If the cleanup takes 500ms and you sleep 1 minute,
// the actual interval is 1 minute + 500ms. Over days, this drift accumulates.
func (rl *RateLimiter) badCleanup(ctx context.Context) {
	for {
		time.Sleep(1 * time.Minute) // Drifts over time
		rl.cleanStaleVisitors()
		// No way to stop this goroutine cleanly!
		// No select, no ctx.Done() check.
	}
}

// GOOD: time.Ticker self-corrects. If cleanup takes 500ms, the next tick
// arrives 500ms earlier to maintain the intended interval.
func (rl *RateLimiter) goodCleanup(ctx context.Context) {
	ticker := time.NewTicker(1 * time.Minute)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return // Clean exit
		case <-ticker.C:
			rl.cleanStaleVisitors()
		}
	}
}
```

### The Pattern

Background cleanup with tickers follows a universal pattern in Go servers:

```
1. Create a time.Ticker
2. Loop with select:
   - ctx.Done() → return (clean shutdown)
   - ticker.C   → do work
3. defer ticker.Stop() to prevent goroutine leak in the runtime
```

You will see this pattern in rate limiter cleanup, token expiry, cache eviction, metrics reporting, health checks, and connection pool maintenance. Every long-running background task in a Go server follows this structure.

---

## 4. Context Cancellation in HTTP Handlers

### How Request Context Works

Every `*http.Request` carries a `context.Context` accessible via `r.Context()`. This context is **automatically cancelled** when:

1. The client disconnects (closes the TCP connection)
2. The client's request times out
3. The HTTP server is shutting down (via `server.Shutdown()`)

This is one of Go's most powerful features for web applications. If a client makes a request, waits 2 seconds, and gives up, the context cancellation propagates to every downstream operation -- database queries, HTTP calls to other services, file I/O -- and they all stop.

### Basic Pattern: Check Context in Handlers

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"
)

// Simulates a repository with a slow database query.
type Repository struct{}

func (repo *Repository) ExpensiveQuery(ctx context.Context) ([]string, error) {
	// Simulate a query that takes 5 seconds.
	// The key: we pass ctx to the operation so it can be cancelled.
	select {
	case <-time.After(5 * time.Second):
		// Query completed successfully
		return []string{"result1", "result2", "result3"}, nil
	case <-ctx.Done():
		// Client disconnected or timeout reached.
		// Return the context error (context.Canceled or context.DeadlineExceeded).
		return nil, ctx.Err()
	}
}

type Handler struct {
	repo *Repository
}

func (h *Handler) LongQuery(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	log.Printf("Starting expensive query for %s", r.RemoteAddr)

	results, err := h.repo.ExpensiveQuery(ctx)
	if err != nil {
		// CRITICAL: Check WHY the error occurred.
		if ctx.Err() == context.Canceled {
			// The client disconnected. There is nobody to send a response to.
			// Writing to w at this point would be pointless (and might panic
			// if the connection is fully closed).
			log.Printf("Client %s disconnected, cancelling query", r.RemoteAddr)
			return
		}
		if ctx.Err() == context.DeadlineExceeded {
			log.Printf("Query timed out for %s", r.RemoteAddr)
			http.Error(w, `{"error": "query timed out"}`, http.StatusGatewayTimeout)
			return
		}
		// Some other error (database down, etc.)
		log.Printf("Query failed: %v", err)
		http.Error(w, `{"error": "internal error"}`, http.StatusInternalServerError)
		return
	}

	log.Printf("Query completed for %s: %d results", r.RemoteAddr, len(results))

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]any{
		"data":  results,
		"count": len(results),
	})
}

func main() {
	handler := &Handler{repo: &Repository{}}

	mux := http.NewServeMux()
	mux.HandleFunc("/api/query", handler.LongQuery)

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
		// Set a global request timeout. Any request taking longer than 30s
		// will have its context cancelled with DeadlineExceeded.
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	fmt.Println("Server running on :8080 -- try: curl localhost:8080/api/query (then Ctrl+C to cancel)")
	log.Fatal(server.ListenAndServe())
}
```

### Adding Per-Handler Timeouts

Sometimes you want different timeouts for different endpoints. A dashboard query should timeout after 10 seconds, but a report generation endpoint might need 60 seconds.

```go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"time"
)

// withTimeout wraps a handler with a per-handler timeout.
// If the handler does not complete within the timeout, the context is cancelled.
func withTimeout(timeout time.Duration, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), timeout)
		defer cancel() // Always cancel to release resources

		// Replace the request's context with the timeout context.
		r = r.WithContext(ctx)
		next(w, r)
	}
}

func dashboardHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// Simulate work
	select {
	case <-time.After(3 * time.Second):
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	case <-ctx.Done():
		log.Printf("Dashboard handler cancelled: %v", ctx.Err())
		if ctx.Err() == context.DeadlineExceeded {
			http.Error(w, `{"error": "timeout"}`, http.StatusGatewayTimeout)
		}
	}
}

func reportHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// Simulate heavy work
	select {
	case <-time.After(15 * time.Second):
		json.NewEncoder(w).Encode(map[string]string{"report": "generated"})
	case <-ctx.Done():
		log.Printf("Report handler cancelled: %v", ctx.Err())
		if ctx.Err() == context.DeadlineExceeded {
			http.Error(w, `{"error": "report generation timed out"}`, http.StatusGatewayTimeout)
		}
	}
}

func main() {
	mux := http.NewServeMux()

	// Dashboard has a 5-second timeout
	mux.HandleFunc("/api/dashboard", withTimeout(5*time.Second, dashboardHandler))

	// Report has a 60-second timeout
	mux.HandleFunc("/api/report", withTimeout(60*time.Second, reportHandler))

	log.Println("Server running on :8080")
	http.ListenAndServe(":8080", mux)
}
```

### Context Propagation: The Full Chain

In a real API, the context flows from the HTTP request through every layer:

```
HTTP Request
  └── r.Context()
       └── middleware (adds values: userID, requestID)
            └── handler
                 └── service layer
                      └── repository layer
                           └── database driver (pgx, sql.DB)
                                └── actual SQL query

If the client disconnects at ANY point, ctx.Done() fires,
and the cancellation propagates to the deepest layer.
```

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"
)

// Each layer accepts and passes context. This is not optional in production Go.

type UserService struct {
	repo *UserRepository
}

func (s *UserService) GetUserDashboard(ctx context.Context, userID string) (map[string]any, error) {
	// Pass ctx to every downstream call.
	user, err := s.repo.FindByID(ctx, userID)
	if err != nil {
		return nil, fmt.Errorf("find user: %w", err)
	}

	stats, err := s.repo.GetStats(ctx, userID)
	if err != nil {
		return nil, fmt.Errorf("get stats: %w", err)
	}

	return map[string]any{
		"user":  user,
		"stats": stats,
	}, nil
}

type UserRepository struct{}

func (r *UserRepository) FindByID(ctx context.Context, id string) (map[string]string, error) {
	// In production: db.QueryRowContext(ctx, "SELECT ... WHERE id = $1", id)
	select {
	case <-time.After(100 * time.Millisecond):
		return map[string]string{"id": id, "name": "Alice"}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

func (r *UserRepository) GetStats(ctx context.Context, userID string) (map[string]int, error) {
	select {
	case <-time.After(200 * time.Millisecond):
		return map[string]int{"sessions": 42, "streak": 7}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

func main() {
	service := &UserService{repo: &UserRepository{}}

	http.HandleFunc("/api/dashboard", func(w http.ResponseWriter, r *http.Request) {
		// The context from the request propagates all the way to the DB calls.
		ctx := r.Context()
		result, err := service.GetUserDashboard(ctx, "user-123")
		if err != nil {
			if ctx.Err() != nil {
				log.Printf("Request cancelled: %v", ctx.Err())
				return
			}
			http.Error(w, `{"error": "internal"}`, http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "%v\n", result)
	})

	log.Println("Server running on :8080")
	http.ListenAndServe(":8080", nil)
}
```

### Node.js Comparison

Node.js added `AbortController` to support request cancellation, but it is opt-in and rarely used consistently:

```javascript
// Node.js: You must manually wire up AbortController
app.get('/api/query', async (req, res) => {
    const controller = new AbortController();

    // If the client disconnects, abort
    req.on('close', () => controller.abort());

    try {
        // Not all libraries support signal
        const result = await db.query('SELECT ...', {
            signal: controller.signal
        });
        res.json(result);
    } catch (err) {
        if (err.name === 'AbortError') {
            return; // Client gone
        }
        res.status(500).json({ error: 'internal' });
    }
});
```

In Go, context cancellation is **built into the request lifecycle**. You do not have to set it up -- it happens automatically. You just have to **pass the context** to downstream operations, which is enforced by convention (and most libraries require it).

---

## 5. Database Connection Pool Concurrency

### How Connection Pools Work

A database connection pool maintains a fixed number of open connections to the database. When a handler needs to run a query, it borrows a connection from the pool, uses it, and returns it. This is inherently a concurrency problem: hundreds of goroutines might need a connection simultaneously, but the pool might only have 25 connections.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Connection Pool (25 connections)                │
│                                                                 │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐   ...   ┌────┐           │
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ │ C5 │         │C25 │           │
│  └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘         └──┬─┘           │
│     │      │      │      │      │               │              │
│  ┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐        ┌──┴──┐          │
│  │ G1  ││ G2  ││ G3  ││ G4  ││ G5  │        │ G25 │          │
│  └─────┘└─────┘└─────┘└─────┘└─────┘        └─────┘          │
│                                                                 │
│  Goroutines G26, G27, ... G500:  BLOCKED (waiting for a free   │
│  connection). They are parked by the runtime -- not spinning.   │
│                                                                 │
│  When G1 finishes its query, it returns C1 to the pool.         │
│  G26 wakes up, acquires C1, and proceeds.                       │
└─────────────────────────────────────────────────────────────────┘
```

### pgx Pool Configuration

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func setupPool(ctx context.Context) (*pgxpool.Pool, error) {
	config, err := pgxpool.ParseConfig("postgres://user:pass@localhost:5432/mydb")
	if err != nil {
		return nil, fmt.Errorf("parse config: %w", err)
	}

	// MaxConns: Maximum number of connections in the pool.
	// This is the upper bound on concurrent database operations.
	// Rule of thumb: NumCPU * 2 + NumDisks (for the DB server).
	// For a small API: 25 is usually fine.
	config.MaxConns = 25

	// MinConns: Connections to keep open even when idle.
	// Avoids connection creation latency on the first request after idle periods.
	config.MinConns = 5

	// MaxConnLifetime: Maximum time a connection stays in the pool.
	// Forces periodic reconnection, which helps with:
	//   - DNS changes (DB failover to a new host)
	//   - Memory leaks in the database driver
	//   - Load balancer connection draining
	config.MaxConnLifetime = 1 * time.Hour

	// MaxConnIdleTime: Close idle connections after this duration.
	// Reduces resource usage during low-traffic periods.
	config.MaxConnIdleTime = 30 * time.Minute

	// HealthCheckPeriod: How often the pool checks if connections are still alive.
	config.HealthCheckPeriod = 1 * time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, config)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}

	// Verify the connection works.
	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("ping: %w", err)
	}

	return pool, nil
}

func main() {
	ctx := context.Background()
	pool, err := setupPool(ctx)
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	// Pool stats are useful for monitoring.
	stat := pool.Stat()
	fmt.Printf("Pool stats:\n")
	fmt.Printf("  Total connections:    %d\n", stat.TotalConns())
	fmt.Printf("  Idle connections:     %d\n", stat.IdleConns())
	fmt.Printf("  Acquired connections: %d\n", stat.AcquiredConns())
	fmt.Printf("  Max connections:      %d\n", stat.MaxConns())
}
```

### Connection Contention in Practice

When the pool is exhausted, new queries block until a connection is returned. With context, you can set a timeout to avoid blocking forever:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"sync/atomic"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

type App struct {
	pool          *pgxpool.Pool
	activeQueries atomic.Int64
}

func (app *App) QueryHandler(w http.ResponseWriter, r *http.Request) {
	// Create a context with a timeout specifically for acquiring a connection.
	// If the pool is exhausted for more than 5 seconds, give up.
	ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
	defer cancel()

	active := app.activeQueries.Add(1)
	defer app.activeQueries.Add(-1)

	log.Printf("Active queries: %d", active)

	// pool.Query internally:
	// 1. Acquires a connection from the pool (may block if pool is full)
	// 2. Executes the query
	// 3. Returns the connection when rows are closed
	//
	// If ctx expires during step 1 (waiting for a connection),
	// the operation returns context.DeadlineExceeded immediately.
	rows, err := app.pool.Query(ctx, "SELECT pg_sleep(2), generate_series(1, 5)")
	if err != nil {
		if ctx.Err() != nil {
			http.Error(w, `{"error": "database timeout - pool exhausted"}`,
				http.StatusServiceUnavailable)
			return
		}
		http.Error(w, `{"error": "query failed"}`, http.StatusInternalServerError)
		return
	}
	defer rows.Close() // CRITICAL: always close rows to return the connection to the pool

	var count int
	for rows.Next() {
		count++
	}

	fmt.Fprintf(w, `{"rows": %d, "active_queries": %d}`, count, app.activeQueries.Load())
}

func main() {
	// This example demonstrates connection contention.
	// With MaxConns=5 and many concurrent requests, some will block.
	ctx := context.Background()

	config, _ := pgxpool.ParseConfig("postgres://user:pass@localhost:5432/mydb")
	config.MaxConns = 5 // Intentionally low to demonstrate contention

	pool, err := pgxpool.NewWithConfig(ctx, config)
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	app := &App{pool: pool}

	http.HandleFunc("/api/query", app.QueryHandler)
	log.Println("Server on :8080 (pool: 5 max connections)")
	http.ListenAndServe(":8080", nil)
}
```

### Why Connection Limits Matter

```
Scenario: 200 concurrent requests, pool size = 25

  Requests 1-25:   Immediately acquire a connection, start querying
  Requests 26-200: BLOCKED, waiting for a connection

  If each query takes 50ms:
    - Request 26 waits ~50ms (until one of 1-25 finishes)
    - Request 200 waits ~350ms (worst case: 8 rounds of 25)

  If each query takes 2 seconds:
    - Request 26 waits ~2s
    - Request 200 waits ~16s (8 rounds of 25, 2s each)

  This is why you must:
  1. Keep queries fast (indexes, EXPLAIN ANALYZE)
  2. Set connection pool size appropriately
  3. Use context timeouts to avoid infinite waits
  4. Monitor pool stats (idle, acquired, waiting)
```

---

## 6. Parallel Database Queries with errgroup

### The Problem

A dashboard endpoint needs data from three different tables: user stats, recent sessions, and progress. Sequentially, this takes the sum of all three query times. In parallel, it takes the time of the slowest query.

```
Sequential:  ──[Stats 80ms]──[Sessions 120ms]──[Progress 60ms]──  = 260ms total
Parallel:    ──[Stats 80ms     ]──
             ──[Sessions 120ms      ]──                             = 120ms total
             ──[Progress 60ms ]──
```

### The errgroup Solution

`errgroup` from `golang.org/x/sync/errgroup` is the standard tool for running multiple goroutines that can fail. It provides:
- Automatic WaitGroup management (no manual `wg.Add` / `wg.Done`)
- First-error collection (returns the first non-nil error)
- Context cancellation on error (if any goroutine fails, all others are cancelled)

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"golang.org/x/sync/errgroup"
)

// Domain types -- in a real app these come from your models package.

type Stats struct {
	TotalSessions int     `json:"totalSessions"`
	TotalMinutes  int     `json:"totalMinutes"`
	AverageScore  float64 `json:"averageScore"`
}

type Session struct {
	ID        string    `json:"id"`
	Topic     string    `json:"topic"`
	Score     int       `json:"score"`
	CreatedAt time.Time `json:"createdAt"`
}

type Progress struct {
	Level          int     `json:"level"`
	XP             int     `json:"xp"`
	CompletionPct  float64 `json:"completionPct"`
}

// Simulated repositories -- replace with actual DB calls.

type StatsRepo struct{}
func (r *StatsRepo) GetStats(ctx context.Context, userID string) (*Stats, error) {
	select {
	case <-time.After(80 * time.Millisecond): // Simulate 80ms query
		return &Stats{TotalSessions: 142, TotalMinutes: 4260, AverageScore: 87.3}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

type SessionRepo struct{}
func (r *SessionRepo) GetRecent(ctx context.Context, userID string, limit int) ([]*Session, error) {
	select {
	case <-time.After(120 * time.Millisecond): // Simulate 120ms query
		return []*Session{
			{ID: "s1", Topic: "Go Concurrency", Score: 92, CreatedAt: time.Now().Add(-1 * time.Hour)},
			{ID: "s2", Topic: "HTTP Handlers", Score: 88, CreatedAt: time.Now().Add(-2 * time.Hour)},
			{ID: "s3", Topic: "Context Package", Score: 95, CreatedAt: time.Now().Add(-3 * time.Hour)},
		}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

type ProgressRepo struct{}
func (r *ProgressRepo) GetProgress(ctx context.Context, userID string) (*Progress, error) {
	select {
	case <-time.After(60 * time.Millisecond): // Simulate 60ms query
		return &Progress{Level: 12, XP: 3400, CompletionPct: 68.5}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

// Handler uses errgroup to fetch all data in parallel.

type DashboardHandler struct {
	statsRepo    *StatsRepo
	sessionRepo  *SessionRepo
	progressRepo *ProgressRepo
}

type DashboardResponse struct {
	Stats    *Stats     `json:"stats"`
	Sessions []*Session `json:"sessions"`
	Progress *Progress  `json:"progress"`
}

func (h *DashboardHandler) GetDashboard(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	userID := "user-123" // In production: extract from auth middleware

	// errgroup.WithContext creates a new group AND a derived context.
	// If any goroutine returns an error, gCtx is cancelled -- which cancels
	// all other goroutines in the group.
	g, gCtx := errgroup.WithContext(ctx)

	// Each variable is written by exactly one goroutine, so no mutex needed.
	var stats *Stats
	var sessions []*Session
	var progress *Progress

	g.Go(func() error {
		var err error
		stats, err = h.statsRepo.GetStats(gCtx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		sessions, err = h.sessionRepo.GetRecent(gCtx, userID, 10)
		return err
	})

	g.Go(func() error {
		var err error
		progress, err = h.progressRepo.GetProgress(gCtx, userID)
		return err
	})

	// Wait blocks until all goroutines complete or one returns an error.
	if err := g.Wait(); err != nil {
		if ctx.Err() != nil {
			// The original request context was cancelled (client disconnected).
			log.Printf("Dashboard request cancelled: %v", ctx.Err())
			return
		}
		log.Printf("Dashboard query failed: %v", err)
		http.Error(w, `{"error": "failed to load dashboard"}`, http.StatusInternalServerError)
		return
	}

	// All three queries succeeded. Write the response.
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(DashboardResponse{
		Stats:    stats,
		Sessions: sessions,
		Progress: progress,
	})
}

func main() {
	handler := &DashboardHandler{
		statsRepo:    &StatsRepo{},
		sessionRepo:  &SessionRepo{},
		progressRepo: &ProgressRepo{},
	}

	http.HandleFunc("/api/dashboard", handler.GetDashboard)

	log.Println("Server running on :8080")
	log.Println("Try: curl localhost:8080/api/dashboard")
	http.ListenAndServe(":8080", nil)
}
```

### Why errgroup, Not Manual WaitGroup + Channels

```go
// VERBOSE: Manual WaitGroup + error channel
func getStatsManual(ctx context.Context, userID string) (*Stats, []*Session, *Progress, error) {
	var wg sync.WaitGroup
	errCh := make(chan error, 3) // Buffered to avoid goroutine leak

	var stats *Stats
	var sessions []*Session
	var progress *Progress

	wg.Add(3)

	go func() {
		defer wg.Done()
		var err error
		stats, err = getStats(ctx, userID)
		if err != nil {
			errCh <- err
		}
	}()

	go func() {
		defer wg.Done()
		var err error
		sessions, err = getSessions(ctx, userID)
		if err != nil {
			errCh <- err
		}
	}()

	go func() {
		defer wg.Done()
		var err error
		progress, err = getProgress(ctx, userID)
		if err != nil {
			errCh <- err
		}
	}()

	wg.Wait()
	close(errCh)

	// Check if any errors occurred
	for err := range errCh {
		return nil, nil, nil, err // Returns first error
	}

	return stats, sessions, progress, nil
}
```

The errgroup version is cleaner, handles cancellation automatically, and is the idiomatic Go approach.

### Cancellation Behavior

```
Scenario: Stats query fails after 30ms. Sessions and Progress are still running.

With errgroup.WithContext:
  1. Stats goroutine returns error at 30ms
  2. errgroup cancels gCtx
  3. Sessions goroutine (at 30ms into its 120ms query) sees gCtx.Done() and returns
  4. Progress goroutine (at 30ms into its 60ms query) sees gCtx.Done() and returns
  5. g.Wait() returns at ~30ms with the Stats error

Without errgroup.WithContext (just errgroup.Group):
  1. Stats goroutine returns error at 30ms
  2. Sessions goroutine keeps running for the full 120ms
  3. Progress goroutine keeps running for the full 60ms
  4. g.Wait() returns at 120ms with the Stats error
  5. 90ms wasted on queries whose results are thrown away
```

---

## 7. Background Token Cleanup

### The Problem

Authentication systems that use refresh tokens must periodically clean up expired tokens from the database. If you don't, the tokens table grows without bound. In a krafty-core style API, this is a background goroutine that runs every hour and deletes tokens that have passed their expiry time.

### The Implementation

```go
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"
)

// TokenCleanupService handles periodic deletion of expired refresh tokens.
type TokenCleanupService struct {
	db       TokenStore
	interval time.Duration
	wg       sync.WaitGroup
}

// TokenStore is the interface for token database operations.
// In production, this would be implemented by a pgx-backed repository.
type TokenStore interface {
	DeleteExpiredTokens(ctx context.Context) (int64, error)
}

// NewTokenCleanupService creates a new service but does not start it.
// Call Start() to begin the background cleanup.
func NewTokenCleanupService(db TokenStore, interval time.Duration) *TokenCleanupService {
	return &TokenCleanupService{
		db:       db,
		interval: interval,
	}
}

// Start begins the background cleanup goroutine.
// It runs until ctx is cancelled. Call Stop() or cancel the context to halt it.
func (s *TokenCleanupService) Start(ctx context.Context) {
	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		s.run(ctx)
	}()
	log.Printf("Token cleanup service started (interval: %s)", s.interval)
}

// Stop waits for the background goroutine to finish.
// You must cancel the context passed to Start() before calling Stop(),
// otherwise Stop() will block forever.
func (s *TokenCleanupService) Stop() {
	s.wg.Wait()
	log.Println("Token cleanup service stopped")
}

func (s *TokenCleanupService) run(ctx context.Context) {
	// Run once immediately on startup to clean any expired tokens
	// that accumulated while the server was down.
	s.cleanup(ctx)

	ticker := time.NewTicker(s.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			s.cleanup(ctx)
		}
	}
}

func (s *TokenCleanupService) cleanup(ctx context.Context) {
	// Use a timeout context for the cleanup operation itself.
	// We don't want a slow DELETE query to run forever.
	cleanupCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	deleted, err := s.db.DeleteExpiredTokens(cleanupCtx)
	if err != nil {
		if ctx.Err() != nil {
			// Parent context cancelled (shutdown). Don't log as error.
			return
		}
		log.Printf("ERROR: token cleanup failed: %v", err)
		return
	}

	if deleted > 0 {
		log.Printf("Token cleanup: deleted %d expired tokens", deleted)
	}
}

// --- Mock implementation for demonstration ---

type MockTokenStore struct {
	expiredCount int
}

func (m *MockTokenStore) DeleteExpiredTokens(ctx context.Context) (int64, error) {
	select {
	case <-time.After(100 * time.Millisecond): // Simulate DB operation
		// Simulate finding some expired tokens
		m.expiredCount++
		if m.expiredCount%3 == 0 {
			return 5, nil // Found expired tokens every third run
		}
		return 0, nil
	case <-ctx.Done():
		return 0, ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	store := &MockTokenStore{}
	cleaner := NewTokenCleanupService(store, 2*time.Second) // Short interval for demo
	cleaner.Start(ctx)

	// Simulate server running for 10 seconds.
	fmt.Println("Server running... (token cleanup runs every 2 seconds)")
	time.Sleep(10 * time.Second)

	// Shutdown sequence.
	fmt.Println("Shutting down...")
	cancel()       // Signal the cleanup goroutine to stop
	cleaner.Stop() // Wait for it to finish
	fmt.Println("Clean shutdown complete")
}
```

### The Pattern: Start/Stop Lifecycle

This Start/Stop pattern is used for every background service in a Go API:

```go
type BackgroundService struct {
    wg sync.WaitGroup
}

func (s *BackgroundService) Start(ctx context.Context) {
    s.wg.Add(1)
    go func() {
        defer s.wg.Done()
        // ... ticker loop with ctx.Done() ...
    }()
}

func (s *BackgroundService) Stop() {
    s.wg.Wait()
}
```

The WaitGroup guarantees that when `Stop()` returns, the goroutine is fully finished. This is critical during server shutdown -- you must not close the database pool until all background goroutines that use it have stopped.

---

## 8. Concurrent Request Processing -- How Go's HTTP Server Works

### Under the Hood

When you call `http.ListenAndServe(":8080", handler)`, here is what happens at the concurrency level:

```go
// Simplified version of what net/http does internally.
// The real implementation is in net/http/server.go.

func (srv *Server) Serve(l net.Listener) error {
    for {
        // Accept blocks until a new TCP connection arrives.
        conn, err := l.Accept()
        if err != nil {
            return err
        }

        // CONCURRENCY: Every connection gets its own goroutine.
        // This is the fundamental concurrency model of Go's HTTP server.
        go srv.handleConnection(conn)
    }
}

func (srv *Server) handleConnection(conn net.Conn) {
    // For HTTP/1.1, each connection may have multiple requests (keep-alive).
    // For HTTP/2, the connection is multiplexed.
    for {
        req, err := readRequest(conn)
        if err != nil {
            return
        }

        // Create a context for this request.
        // It will be cancelled when the connection closes.
        ctx, cancel := context.WithCancel(srv.BaseContext)
        req = req.WithContext(ctx)

        // Call the handler (your code).
        srv.Handler.ServeHTTP(responseWriter, req)

        cancel() // Clean up the context
    }
}
```

### Visualizing Concurrent Requests

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"sync/atomic"
	"time"
)

var (
	activeRequests  atomic.Int64
	totalRequests   atomic.Int64
)

func handler(w http.ResponseWriter, r *http.Request) {
	// Track active concurrent requests.
	active := activeRequests.Add(1)
	total := totalRequests.Add(1)
	defer activeRequests.Add(-1)

	// Simulate some work (database query, etc.)
	time.Sleep(100 * time.Millisecond)

	fmt.Fprintf(w, "Request #%d | Active: %d | Goroutines: %d\n",
		total, active, runtime.NumGoroutine())
}

func statsHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Active requests: %d\nTotal requests: %d\nGoroutines: %d\n",
		activeRequests.Load(),
		totalRequests.Load(),
		runtime.NumGoroutine(),
	)
}

func main() {
	http.HandleFunc("/api/work", handler)
	http.HandleFunc("/stats", statsHandler)

	log.Println("Server on :8080 -- use 'ab -n 1000 -c 100 http://localhost:8080/api/work' to test")
	http.ListenAndServe(":8080", nil)
}
```

### Testing with Apache Bench

```bash
# Send 1000 requests, 100 at a time concurrently:
ab -n 1000 -c 100 http://localhost:8080/api/work

# Expected results:
# - 100 goroutines active simultaneously
# - Each request takes ~100ms
# - Total time: ~1 second (1000 requests / 100 concurrent * 100ms)
# - No requests failed
```

### Configuring the HTTP Server for Production

```go
package main

import (
	"context"
	"log"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("OK"))
	})

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,

		// ReadTimeout: Maximum time to read the entire request (headers + body).
		// Protects against slowloris attacks.
		ReadTimeout: 15 * time.Second,

		// ReadHeaderTimeout: Maximum time to read request headers only.
		// More targeted than ReadTimeout for protecting against slowloris.
		ReadHeaderTimeout: 5 * time.Second,

		// WriteTimeout: Maximum time to write the response.
		// Prevents slow clients from tying up connections indefinitely.
		WriteTimeout: 30 * time.Second,

		// IdleTimeout: Maximum time a keep-alive connection can be idle
		// before the server closes it.
		IdleTimeout: 60 * time.Second,

		// MaxHeaderBytes: Maximum size of request headers.
		// Protects against excessively large headers.
		MaxHeaderBytes: 1 << 20, // 1 MB
	}

	log.Printf("Server starting on %s", server.Addr)

	if err := server.ListenAndServe(); err != http.ErrServerClosed {
		log.Fatal(err)
	}
}
```

### Go vs Node.js: Handling 10,000 Concurrent Connections

```
Go:
  - Spawns 10,000 goroutines (~2KB stack each = ~20MB total)
  - Each goroutine can block independently
  - The Go scheduler multiplexes onto NumCPU OS threads
  - Memory: ~50MB for 10K connections

Node.js:
  - Single event loop handles all 10K connections
  - Non-blocking I/O via libuv
  - If any callback does CPU work > ~1ms, all connections stall
  - Memory: ~100MB+ due to V8 overhead per connection state

Go with CPU-bound handlers:
  - Each goroutine uses its own CPU time slice
  - NumCPU goroutines run truly in parallel
  - Others are scheduled fairly by the Go runtime
  - No starvation -- all requests make progress

Node.js with CPU-bound handlers:
  - Event loop is BLOCKED
  - All 10K connections stall
  - Must use worker_threads or child_process
  - Complex code to coordinate
```

---

## 9. Shared State Protection Patterns -- Mutex vs Channel vs Atomic

### The Decision Framework

In a web application, you will encounter shared state constantly. The question is always: **which synchronization primitive should I use?**

```
┌─────────────────────────────────────────────────────────────────┐
│              SHARED STATE DECISION TREE                          │
│                                                                 │
│  Is it a simple counter or flag?                                │
│  ├── YES → Use sync/atomic (fastest, lock-free)                 │
│  │         Examples: request counter, active connections,       │
│  │                   feature flag (bool), health status         │
│  │                                                              │
│  └── NO → Is it protecting a data structure (map, slice, etc.)? │
│       ├── YES → Is it mostly reads, rare writes?                │
│       │    ├── YES → Use sync.RWMutex                           │
│       │    │         Examples: config cache, feature flags map,  │
│       │    │                   route table                       │
│       │    │                                                     │
│       │    └── NO → Use sync.Mutex                              │
│       │             Examples: rate limiter map (every call       │
│       │                       writes), session store            │
│       │                                                         │
│       └── NO → Is it coordinating between goroutines?           │
│            ├── YES → Use channels                               │
│            │         Examples: job queue, results collection,    │
│            │                   shutdown signaling                │
│            │                                                     │
│            └── Need one-time init? → Use sync.Once              │
│                Examples: DB connection, config loading           │
└─────────────────────────────────────────────────────────────────┘
```

### Pattern 1: Atomic for Simple Counters

```go
package main

import (
	"fmt"
	"net/http"
	"sync/atomic"
)

// RequestMetrics tracks API metrics without locks.
// Atomics are the fastest synchronization primitive --
// they compile to single CPU instructions (LOCK XADD, etc.).
type RequestMetrics struct {
	totalRequests  atomic.Int64
	activeRequests atomic.Int64
	totalErrors    atomic.Int64
	totalBytes     atomic.Int64
}

func (m *RequestMetrics) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		m.totalRequests.Add(1)
		m.activeRequests.Add(1)
		defer m.activeRequests.Add(-1)

		// Wrap the ResponseWriter to capture status and bytes.
		wrapped := &statusWriter{ResponseWriter: w, status: 200}
		next.ServeHTTP(wrapped, r)

		m.totalBytes.Add(int64(wrapped.bytes))
		if wrapped.status >= 500 {
			m.totalErrors.Add(1)
		}
	})
}

func (m *RequestMetrics) Stats() map[string]int64 {
	return map[string]int64{
		"total_requests":  m.totalRequests.Load(),
		"active_requests": m.activeRequests.Load(),
		"total_errors":    m.totalErrors.Load(),
		"total_bytes":     m.totalBytes.Load(),
	}
}

type statusWriter struct {
	http.ResponseWriter
	status int
	bytes  int
}

func (w *statusWriter) WriteHeader(status int) {
	w.status = status
	w.ResponseWriter.WriteHeader(status)
}

func (w *statusWriter) Write(b []byte) (int, error) {
	n, err := w.ResponseWriter.Write(b)
	w.bytes += n
	return n, err
}

func main() {
	metrics := &RequestMetrics{}

	mux := http.NewServeMux()
	mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, `{"data": "hello"}`)
	})
	mux.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		stats := metrics.Stats()
		fmt.Fprintf(w, "total=%d active=%d errors=%d bytes=%d\n",
			stats["total_requests"], stats["active_requests"],
			stats["total_errors"], stats["total_bytes"])
	})

	http.ListenAndServe(":8080", metrics.Middleware(mux))
}
```

### Pattern 2: RWMutex for Read-Heavy Config

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
)

// ConfigStore holds application configuration that is read by every request
// but updated rarely (e.g., by an admin endpoint or a config reload).
//
// sync.RWMutex is ideal here:
//   - RLock allows unlimited concurrent readers (every request)
//   - Lock blocks all readers and writers (only during config update)
type ConfigStore struct {
	mu     sync.RWMutex
	config map[string]string
}

func NewConfigStore() *ConfigStore {
	return &ConfigStore{
		config: map[string]string{
			"max_upload_size":  "10MB",
			"maintenance_mode": "false",
			"api_version":      "v2",
		},
	}
}

// Get is called on every request -- uses RLock (concurrent reads allowed).
func (cs *ConfigStore) Get(key string) (string, bool) {
	cs.mu.RLock()
	defer cs.mu.RUnlock()
	val, ok := cs.config[key]
	return val, ok
}

// GetAll returns a copy of the entire config.
func (cs *ConfigStore) GetAll() map[string]string {
	cs.mu.RLock()
	defer cs.mu.RUnlock()
	// Return a copy so the caller cannot mutate the internal map.
	copy := make(map[string]string, len(cs.config))
	for k, v := range cs.config {
		copy[k] = v
	}
	return copy
}

// Set updates a config value -- uses Lock (exclusive, blocks all readers).
func (cs *ConfigStore) Set(key, value string) {
	cs.mu.Lock()
	defer cs.mu.Unlock()
	cs.config[key] = value
}

func main() {
	store := NewConfigStore()

	http.HandleFunc("/config", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodGet:
			// Many concurrent requests can read simultaneously.
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(store.GetAll())

		case http.MethodPut:
			// Updating config blocks readers momentarily.
			key := r.URL.Query().Get("key")
			value := r.URL.Query().Get("value")
			if key == "" || value == "" {
				http.Error(w, "key and value required", http.StatusBadRequest)
				return
			}
			store.Set(key, value)
			fmt.Fprintf(w, "Set %s = %s\n", key, value)

		default:
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		}
	})

	fmt.Println("Server on :8080")
	http.ListenAndServe(":8080", nil)
}
```

### Pattern 3: Channel for Job Queue

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
)

// Job represents a background task to be processed.
type Job struct {
	ID      string
	Payload string
}

// JobQueue uses a channel to coordinate between HTTP handlers
// (producers) and background workers (consumers).
type JobQueue struct {
	jobs chan Job
	wg   sync.WaitGroup
}

func NewJobQueue(bufferSize int, workerCount int, ctx context.Context) *JobQueue {
	jq := &JobQueue{
		jobs: make(chan Job, bufferSize),
	}

	// Start worker goroutines.
	for i := 0; i < workerCount; i++ {
		jq.wg.Add(1)
		go jq.worker(ctx, i)
	}

	return jq
}

func (jq *JobQueue) Submit(job Job) bool {
	select {
	case jq.jobs <- job:
		return true // Job accepted
	default:
		return false // Queue full, reject
	}
}

func (jq *JobQueue) worker(ctx context.Context, id int) {
	defer jq.wg.Done()
	log.Printf("Worker %d started", id)

	for {
		select {
		case <-ctx.Done():
			log.Printf("Worker %d stopping", id)
			return
		case job, ok := <-jq.jobs:
			if !ok {
				log.Printf("Worker %d: channel closed", id)
				return
			}
			log.Printf("Worker %d processing job %s: %s", id, job.ID, job.Payload)
			time.Sleep(500 * time.Millisecond) // Simulate work
			log.Printf("Worker %d completed job %s", id, job.ID)
		}
	}
}

func (jq *JobQueue) Stop() {
	close(jq.jobs) // Signal workers no more jobs are coming
	jq.wg.Wait()   // Wait for all workers to finish current jobs
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	queue := NewJobQueue(100, 3, ctx) // Buffer 100 jobs, 3 workers

	http.HandleFunc("/api/submit", func(w http.ResponseWriter, r *http.Request) {
		job := Job{
			ID:      fmt.Sprintf("job-%d", time.Now().UnixNano()),
			Payload: r.URL.Query().Get("task"),
		}

		if queue.Submit(job) {
			fmt.Fprintf(w, `{"status": "accepted", "job_id": "%s"}`, job.ID)
		} else {
			http.Error(w, `{"error": "queue full"}`, http.StatusServiceUnavailable)
		}
	})

	log.Println("Server on :8080 -- POST /api/submit?task=something")
	http.ListenAndServe(":8080", nil)
}
```

### Summary Table

```
┌─────────────────────┬─────────────────────────┬──────────────────────────┐
│ Primitive           │ Best For                │ Web App Example          │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ sync/atomic         │ Simple counters, flags  │ Request metrics,         │
│                     │ (int64, bool, pointer)  │ active connection count  │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ sync.Mutex          │ Protecting shared data  │ Rate limiter map,        │
│                     │ with frequent writes    │ session store            │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ sync.RWMutex        │ Protecting shared data  │ Config store, route      │
│                     │ with many reads, few    │ table, feature flags     │
│                     │ writes                  │                          │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ Channels            │ Communicating between   │ Job queues, fan-out,     │
│                     │ goroutines, signaling   │ shutdown signals         │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ sync.Once           │ One-time initialization │ DB pool, logger, config  │
├─────────────────────┼─────────────────────────┼──────────────────────────┤
│ errgroup            │ Parallel operations     │ Parallel DB queries,     │
│                     │ with error handling     │ fetching multiple APIs   │
└─────────────────────┴─────────────────────────┴──────────────────────────┘
```

---

## 10. sync.Once for Lazy Initialization

### The Problem

Some resources are expensive to initialize: database connection pools, HTTP clients with custom transports, compiled templates, loaded configuration files. You want to initialize them **once**, **lazily** (not at startup if they might not be needed), and **safely** (even if 100 goroutines try to use them simultaneously).

### sync.Once Guarantees

`sync.Once` provides these guarantees:
1. The function passed to `Do()` runs **exactly once**, even if called from 1000 goroutines.
2. All goroutines that call `Do()` **block** until the first call completes.
3. After the first call completes, subsequent calls to `Do()` return immediately.

### Database Pool Initialization

```go
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"
)

// In a real application, you would import:
// "github.com/jackc/pgx/v5/pgxpool"

// Simulated pool for demonstration.
type Pool struct {
	DSN string
}

func (p *Pool) Close() {
	log.Printf("Pool closed (DSN: %s)", p.DSN)
}

func NewPool(ctx context.Context, dsn string) (*Pool, error) {
	log.Println("Creating database pool... (expensive operation)")
	time.Sleep(500 * time.Millisecond) // Simulate slow connection setup
	return &Pool{DSN: dsn}, nil
}

// --- Global singleton pattern with sync.Once ---

var (
	dbOnce sync.Once
	dbPool *Pool
	dbErr  error
)

// GetDB returns the database pool, creating it on first call.
// All subsequent calls return the same pool instance.
//
// WARNING: If initialization fails, dbErr is set and every subsequent
// call returns the same error. sync.Once does NOT retry on failure.
// For retry-on-failure, see the pattern below.
func GetDB() (*Pool, error) {
	dbOnce.Do(func() {
		log.Println("First call to GetDB -- initializing pool")
		dbPool, dbErr = NewPool(context.Background(), "postgres://localhost:5432/mydb")
	})
	return dbPool, dbErr
}

func main() {
	// Simulate 10 goroutines all calling GetDB concurrently at startup.
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			pool, err := GetDB()
			if err != nil {
				log.Printf("Goroutine %d: error: %v", id, err)
				return
			}
			fmt.Printf("Goroutine %d: got pool (DSN: %s)\n", id, pool.DSN)
		}(i)
	}
	wg.Wait()

	// Output:
	// First call to GetDB -- initializing pool
	// Creating database pool... (expensive operation)
	// Goroutine 0: got pool (DSN: postgres://localhost:5432/mydb)
	// Goroutine 1: got pool (DSN: postgres://localhost:5432/mydb)
	// ... (all 10 goroutines get the same pool)
	// "Creating database pool" appears EXACTLY ONCE.
}
```

### sync.Once with Retry on Failure

The standard `sync.Once` has a limitation: if the function fails, it is still considered "done" and will never run again. For initialization that might fail (network issues, database down), you need a retry pattern:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"sync"
	"sync/atomic"
	"time"
)

// Pool is a simulated database pool.
type Pool struct{ DSN string }

// LazyPool provides lazy initialization with retry-on-failure semantics.
// Unlike sync.Once, if initialization fails, the next call will try again.
type LazyPool struct {
	mu      sync.Mutex
	pool    *Pool
	dsn     string
	initErr error
	inited  atomic.Bool // Fast path: skip the mutex after successful init
}

func NewLazyPool(dsn string) *LazyPool {
	return &LazyPool{dsn: dsn}
}

func (lp *LazyPool) Get(ctx context.Context) (*Pool, error) {
	// Fast path: already initialized successfully.
	// atomic.Bool.Load is a single CPU instruction -- no contention.
	if lp.inited.Load() {
		return lp.pool, nil
	}

	// Slow path: need to initialize (or retry after failure).
	lp.mu.Lock()
	defer lp.mu.Unlock()

	// Double-check after acquiring the lock (another goroutine may have
	// initialized while we were waiting).
	if lp.inited.Load() {
		return lp.pool, nil
	}

	log.Println("Attempting to initialize pool...")
	pool, err := connectToDatabase(ctx, lp.dsn)
	if err != nil {
		return nil, fmt.Errorf("init pool: %w", err)
	}

	lp.pool = pool
	lp.inited.Store(true)
	log.Println("Pool initialized successfully")
	return lp.pool, nil
}

// Simulated connection that fails 50% of the time.
var attempt atomic.Int32

func connectToDatabase(ctx context.Context, dsn string) (*Pool, error) {
	n := attempt.Add(1)
	if n <= 2 {
		return nil, errors.New("connection refused")
	}
	return &Pool{DSN: dsn}, nil
}

func main() {
	lp := NewLazyPool("postgres://localhost:5432/mydb")
	ctx := context.Background()

	for i := 0; i < 5; i++ {
		pool, err := lp.Get(ctx)
		if err != nil {
			log.Printf("Attempt %d: %v", i+1, err)
			time.Sleep(100 * time.Millisecond)
			continue
		}
		fmt.Printf("Attempt %d: success (DSN: %s)\n", i+1, pool.DSN)
		break
	}
}
```

### Common Uses of sync.Once in Web Applications

```go
// 1. Template compilation
var (
	tmplOnce sync.Once
	tmpl     *template.Template
)

func getTemplates() *template.Template {
	tmplOnce.Do(func() {
		tmpl = template.Must(template.ParseGlob("templates/*.html"))
	})
	return tmpl
}

// 2. Logger initialization
var (
	loggerOnce sync.Once
	logger     *slog.Logger
)

func getLogger() *slog.Logger {
	loggerOnce.Do(func() {
		logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
			Level: slog.LevelInfo,
		}))
	})
	return logger
}

// 3. HTTP client with custom settings
var (
	clientOnce sync.Once
	httpClient *http.Client
)

func getHTTPClient() *http.Client {
	clientOnce.Do(func() {
		httpClient = &http.Client{
			Timeout: 30 * time.Second,
			Transport: &http.Transport{
				MaxIdleConns:        100,
				MaxIdleConnsPerHost: 10,
				IdleConnTimeout:     90 * time.Second,
			},
		}
	})
	return httpClient
}
```

### Node.js Comparison

Node.js does not need `sync.Once` because module loading is synchronous and single-threaded. A module's top-level code runs once, and all `require()` calls return the cached export:

```javascript
// Node.js: The module system IS sync.Once.
// db.js
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
module.exports = pool;

// Any file that requires db.js gets the same pool instance.
// handler.js
const pool = require('./db'); // Same instance every time
```

In Go, package-level `init()` functions serve a similar purpose, but `sync.Once` gives you **lazy** initialization (only when first used) and **error handling** (init can fail, and you handle it at the call site).

---

## 11. Worker Pool for Background Jobs

### The Problem

Many API operations trigger work that does not need to happen before the response is sent: sending welcome emails, generating thumbnails, updating analytics, syncing to a third-party service. A worker pool processes these jobs in the background without blocking the HTTP response.

### Complete Worker Pool Implementation

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"sync"
	"sync/atomic"
	"time"
)

// Job represents a unit of background work.
type Job struct {
	ID        string    `json:"id"`
	Type      string    `json:"type"`
	Payload   string    `json:"payload"`
	CreatedAt time.Time `json:"createdAt"`
}

// JobResult holds the outcome of processing a job.
type JobResult struct {
	JobID    string        `json:"jobId"`
	Success  bool          `json:"success"`
	Error    string        `json:"error,omitempty"`
	Duration time.Duration `json:"duration"`
}

// WorkerPool manages a fixed number of worker goroutines that process
// jobs from a shared channel.
type WorkerPool struct {
	jobs       chan Job
	results    chan JobResult
	numWorkers int
	wg         sync.WaitGroup
	processor  JobProcessor

	// Metrics (atomic for lock-free concurrent access)
	processed atomic.Int64
	failed    atomic.Int64
	pending   atomic.Int64
}

// JobProcessor defines how to process a job. Different job types
// can be dispatched by implementing this interface.
type JobProcessor interface {
	Process(ctx context.Context, job Job) error
}

// ProcessorFunc adapts a function to the JobProcessor interface.
type ProcessorFunc func(ctx context.Context, job Job) error

func (f ProcessorFunc) Process(ctx context.Context, job Job) error {
	return f(ctx, job)
}

func NewWorkerPool(numWorkers, queueSize int, processor JobProcessor) *WorkerPool {
	return &WorkerPool{
		jobs:       make(chan Job, queueSize),
		results:    make(chan JobResult, queueSize),
		numWorkers: numWorkers,
		processor:  processor,
	}
}

// Start launches all worker goroutines. They run until ctx is cancelled.
func (wp *WorkerPool) Start(ctx context.Context) {
	// Start workers
	for i := 0; i < wp.numWorkers; i++ {
		wp.wg.Add(1)
		go wp.worker(ctx, i)
	}

	// Start result collector (optional: logs results, updates DB, etc.)
	wp.wg.Add(1)
	go wp.resultCollector(ctx)

	log.Printf("Worker pool started: %d workers, queue size %d", wp.numWorkers, cap(wp.jobs))
}

// Submit adds a job to the queue. Returns false if the queue is full.
func (wp *WorkerPool) Submit(job Job) bool {
	select {
	case wp.jobs <- job:
		wp.pending.Add(1)
		return true
	default:
		return false // Queue full
	}
}

// Stats returns current pool metrics.
func (wp *WorkerPool) Stats() map[string]int64 {
	return map[string]int64{
		"processed": wp.processed.Load(),
		"failed":    wp.failed.Load(),
		"pending":   wp.pending.Load(),
		"queued":    int64(len(wp.jobs)),
	}
}

// Stop signals workers to stop and waits for them to finish.
func (wp *WorkerPool) Stop() {
	close(wp.jobs)  // Signal no more jobs
	wp.wg.Wait()    // Wait for all workers and collector to finish
	close(wp.results)
	log.Println("Worker pool stopped")
}

func (wp *WorkerPool) worker(ctx context.Context, id int) {
	defer wp.wg.Done()
	log.Printf("Worker %d started", id)

	for {
		select {
		case <-ctx.Done():
			// Drain remaining jobs before exiting (optional: depends on requirements)
			log.Printf("Worker %d received shutdown signal", id)
			wp.drainJobs(id)
			return

		case job, ok := <-wp.jobs:
			if !ok {
				// Channel closed -- no more jobs.
				log.Printf("Worker %d: no more jobs, exiting", id)
				return
			}

			start := time.Now()
			err := wp.processor.Process(ctx, job)
			duration := time.Since(start)

			result := JobResult{
				JobID:    job.ID,
				Success:  err == nil,
				Duration: duration,
			}

			wp.pending.Add(-1)

			if err != nil {
				result.Error = err.Error()
				wp.failed.Add(1)
			} else {
				wp.processed.Add(1)
			}

			// Non-blocking send to results channel.
			select {
			case wp.results <- result:
			default:
				// Results channel full -- log but don't block the worker.
				log.Printf("Worker %d: results channel full, dropping result for job %s", id, job.ID)
			}
		}
	}
}

func (wp *WorkerPool) drainJobs(workerID int) {
	for {
		select {
		case job, ok := <-wp.jobs:
			if !ok {
				return
			}
			log.Printf("Worker %d: draining job %s", workerID, job.ID)
			ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
			wp.processor.Process(ctx, job)
			cancel()
			wp.pending.Add(-1)
		default:
			return
		}
	}
}

func (wp *WorkerPool) resultCollector(ctx context.Context) {
	defer wp.wg.Done()
	for {
		select {
		case <-ctx.Done():
			return
		case result, ok := <-wp.results:
			if !ok {
				return
			}
			if result.Success {
				log.Printf("Job %s completed in %s", result.JobID, result.Duration)
			} else {
				log.Printf("Job %s FAILED in %s: %s", result.JobID, result.Duration, result.Error)
			}
		}
	}
}

// --- Example usage with an HTTP API ---

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Create a processor that simulates email sending.
	processor := ProcessorFunc(func(ctx context.Context, job Job) error {
		// Simulate work that takes 200-800ms
		duration := time.Duration(200+rand.Intn(600)) * time.Millisecond

		select {
		case <-time.After(duration):
			// Simulate occasional failures
			if rand.Intn(10) == 0 {
				return fmt.Errorf("SMTP connection failed")
			}
			return nil
		case <-ctx.Done():
			return ctx.Err()
		}
	})

	pool := NewWorkerPool(5, 100, processor)
	pool.Start(ctx)

	mux := http.NewServeMux()

	// Submit a background job
	mux.HandleFunc("/api/jobs", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "POST only", http.StatusMethodNotAllowed)
			return
		}

		job := Job{
			ID:        fmt.Sprintf("job-%d", time.Now().UnixNano()),
			Type:      "send_email",
			Payload:   r.URL.Query().Get("to"),
			CreatedAt: time.Now(),
		}

		if pool.Submit(job) {
			w.WriteHeader(http.StatusAccepted)
			json.NewEncoder(w).Encode(map[string]string{
				"status": "accepted",
				"job_id": job.ID,
			})
		} else {
			http.Error(w, `{"error": "queue full, try again later"}`, http.StatusServiceUnavailable)
		}
	})

	// Check pool stats
	mux.HandleFunc("/api/jobs/stats", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(pool.Stats())
	})

	log.Println("Server on :8080")
	log.Println("  POST /api/jobs?to=user@example.com  -- submit a job")
	log.Println("  GET  /api/jobs/stats                -- view pool stats")
	http.ListenAndServe(":8080", mux)
}
```

### Worker Pool Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        Worker Pool                                  │
│                                                                     │
│   HTTP Handler                      Workers                         │
│   ┌─────────┐    ┌──────────┐    ┌──────────┐                     │
│   │ POST    │───►│          │    │ Worker 0 │──► Process ──►       │
│   │ /api/job│    │  jobs    │    ├──────────┤          │           │
│   └─────────┘    │ channel  │───►│ Worker 1 │──────────┤           │
│                  │ (buffer  │    ├──────────┤          │           │
│   ┌─────────┐    │  = 100)  │    │ Worker 2 │──────────┤   results │
│   │ POST    │───►│          │    ├──────────┤          ▼   channel │
│   │ /api/job│    └──────────┘    │ Worker 3 │     ┌────────┐      │
│   └─────────┘                    ├──────────┤     │Collector│      │
│                                  │ Worker 4 │     │(logs,  │      │
│   Response: 202 Accepted         └──────────┘     │metrics)│      │
│   (returned immediately,                          └────────┘      │
│    before job is processed)                                        │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 12. Graceful Goroutine Shutdown

### The Problem

When your server receives SIGTERM (Kubernetes pod termination, `docker stop`, `systemctl stop`), you must:

1. Stop accepting new HTTP requests
2. Wait for in-flight requests to complete
3. Stop all background goroutines (cleanup tasks, workers)
4. Wait for background goroutines to finish
5. Close the database pool
6. Exit

If you skip any step, you get data corruption (mid-query DB writes), goroutine leaks (in tests), or lost jobs.

### Complete Graceful Shutdown

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// BackgroundService represents any goroutine that runs for the lifetime
// of the application (cleanup tasks, workers, etc.).
type BackgroundService struct {
	name string
	wg   sync.WaitGroup
}

func (s *BackgroundService) Start(ctx context.Context, work func(context.Context)) {
	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		log.Printf("[%s] started", s.name)
		work(ctx)
		log.Printf("[%s] stopped", s.name)
	}()
}

func (s *BackgroundService) Stop() {
	s.wg.Wait()
}

// simulateTokenCleanup mimics the token cleanup goroutine.
func simulateTokenCleanup(ctx context.Context) {
	ticker := time.NewTicker(2 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			log.Println("[token-cleanup] cleaning expired tokens...")
		}
	}
}

// simulateMetricsReporter mimics a metrics flushing goroutine.
func simulateMetricsReporter(ctx context.Context) {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			// Flush final metrics before exiting.
			log.Println("[metrics] flushing final metrics...")
			time.Sleep(200 * time.Millisecond) // Simulate flush
			return
		case <-ticker.C:
			log.Println("[metrics] reporting metrics...")
		}
	}
}

func main() {
	// STEP 1: Create a context for background goroutines.
	// Cancelling this context will signal ALL background goroutines to stop.
	bgCtx, bgCancel := context.WithCancel(context.Background())

	// STEP 2: Start background services.
	tokenCleaner := &BackgroundService{name: "token-cleanup"}
	tokenCleaner.Start(bgCtx, simulateTokenCleanup)

	metricsReporter := &BackgroundService{name: "metrics"}
	metricsReporter.Start(bgCtx, simulateMetricsReporter)

	// STEP 3: Set up the HTTP server.
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// Simulate a slow request to demonstrate in-flight request handling.
		time.Sleep(1 * time.Second)
		fmt.Fprintln(w, "OK")
	})

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	// STEP 4: Start the HTTP server in a goroutine.
	// ListenAndServe blocks, so we run it in a goroutine to keep main() flowing.
	serverErr := make(chan error, 1)
	go func() {
		log.Println("HTTP server starting on :8080")
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			serverErr <- err
		}
		close(serverErr)
	}()

	// STEP 5: Wait for shutdown signal.
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case sig := <-quit:
		log.Printf("Received signal: %s", sig)
	case err := <-serverErr:
		log.Printf("Server error: %v", err)
	}

	// STEP 6: Graceful shutdown sequence.
	log.Println("Starting graceful shutdown...")

	// 6a: Stop accepting new HTTP requests and wait for in-flight requests.
	// Give in-flight requests up to 30 seconds to complete.
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer shutdownCancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Printf("HTTP server shutdown error: %v", err)
	} else {
		log.Println("HTTP server stopped (all in-flight requests completed)")
	}

	// 6b: Cancel the background context to signal all background goroutines.
	bgCancel()

	// 6c: Wait for all background goroutines to finish.
	log.Println("Waiting for background services to stop...")
	tokenCleaner.Stop()
	metricsReporter.Stop()
	log.Println("All background services stopped")

	// 6d: Close the database pool (after all goroutines that use it have stopped).
	// pool.Close()
	log.Println("Database pool closed")

	log.Println("Shutdown complete")
}
```

### The Shutdown Timeline

```
Time    Event
────    ─────────────────────────────────────────────────
0.0s    SIGTERM received
0.0s    server.Shutdown() called -- stop accepting new connections
0.0s    In-flight requests continue processing
0.5s    In-flight request 1 completes
1.0s    In-flight request 2 completes
1.0s    server.Shutdown() returns -- all requests done
1.0s    bgCancel() -- signals background goroutines
1.0s    token-cleanup goroutine sees ctx.Done(), returns
1.0s    metrics goroutine sees ctx.Done(), flushes final batch
1.2s    metrics goroutine returns
1.2s    All wg.Wait() calls return
1.2s    pool.Close() -- database connections closed
1.2s    main() returns, process exits with code 0
```

### Why the Order Matters

```
WRONG ORDER:
  pool.Close()       ← Database connections closed
  bgCancel()         ← Background goroutines try to use pool... PANIC

WRONG ORDER:
  bgCancel()         ← Background goroutines stopping
  server.Shutdown()  ← In-flight requests might still need background services

CORRECT ORDER:
  server.Shutdown()  ← No new requests, wait for in-flight
  bgCancel()         ← Stop background goroutines
  wg.Wait()          ← Wait for them to finish
  pool.Close()       ← Safe: nobody is using the pool anymore
```

### Node.js Comparison

```javascript
// Node.js graceful shutdown -- conceptually similar but fewer moving parts
// because there are no background goroutines to manage.
const server = app.listen(8080);

process.on('SIGTERM', async () => {
    console.log('SIGTERM received');

    // Stop accepting new connections
    server.close(() => {
        console.log('HTTP server closed');
    });

    // Stop background intervals
    clearInterval(tokenCleanupInterval);
    clearInterval(metricsInterval);

    // Close database pool
    await pool.end();

    process.exit(0);
});

// Problem: server.close() does not wait for in-flight requests by default.
// You need additional libraries (like 'stoppable') or manual tracking.
// Go's server.Shutdown() handles this natively.
```

---

## 13. Race Condition Debugging

### The Race Detector

Go has a built-in race detector that instruments your code at compile time to detect concurrent access to shared memory without proper synchronization. It is one of the most valuable tools in Go's toolchain.

```bash
# Run tests with race detection
go test -race ./...

# Run a program with race detection
go run -race main.go

# Build with race detection (for staging environments)
go build -race -o myapp
```

### Example: Finding a Race Condition

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync"
	"testing"
)

// BUG: This handler has a race condition.
// The map is shared across all goroutines (requests) and modified
// without synchronization.
type BuggyHandler struct {
	counts map[string]int // RACE: concurrent map writes
}

func (h *BuggyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	path := r.URL.Path
	h.counts[path]++ // RACE: unsynchronized write
	fmt.Fprintf(w, "Count: %d", h.counts[path]) // RACE: unsynchronized read
}

func TestBuggyHandler_Race(t *testing.T) {
	handler := &BuggyHandler{counts: make(map[string]int)}
	server := httptest.NewServer(handler)
	defer server.Close()

	// Send 100 concurrent requests.
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			http.Get(server.URL + "/api/test")
		}()
	}
	wg.Wait()
}

// Run with: go test -race -run TestBuggyHandler_Race
//
// Output:
// ==================
// WARNING: DATA RACE
// Write at 0x00c0000a2060 by goroutine 23:
//   runtime.mapassign_faststr()
//       /usr/local/go/src/runtime/map_faststr.go:203 +0x0
//   main.(*BuggyHandler).ServeHTTP()
//       /path/to/main.go:18 +0x78
// ...
// Previous write at 0x00c0000a2060 by goroutine 22:
//   runtime.mapassign_faststr()
// ...
// ==================
// FAIL
```

### The Fix

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
)

// FIXED: Protect the map with a mutex.
type FixedHandler struct {
	mu     sync.Mutex
	counts map[string]int
}

func (h *FixedHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	path := r.URL.Path

	h.mu.Lock()
	h.counts[path]++
	count := h.counts[path]
	h.mu.Unlock()

	// Write the response AFTER releasing the lock.
	// Never hold a lock while doing I/O -- it creates unnecessary contention.
	fmt.Fprintf(w, "Count: %d", count)
}
```

### Common Race Conditions in Web Applications

```go
// RACE 1: Shared map without mutex
type Cache struct {
    data map[string]any // BUG: no mutex
}

func (c *Cache) Set(key string, val any) {
    c.data[key] = val // RACE: concurrent writes from different handler goroutines
}

// RACE 2: Closing over loop variable (fixed in Go 1.22+, but still common in older code)
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // Before Go 1.22: RACE -- all goroutines see the same i
    }()
}

// RACE 3: Reading a shared variable while another goroutine writes it
var config *Config // Shared

func reloadConfig() {
    config = loadFromFile() // Write
}

func handler(w http.ResponseWriter, r *http.Request) {
    cfg := config // Read -- RACE with reloadConfig
    // ...
}

// RACE 4: Accessing http.ResponseWriter after handler returns
func handler(w http.ResponseWriter, r *http.Request) {
    go func() {
        time.Sleep(1 * time.Second)
        w.Write([]byte("too late")) // RACE: handler already returned
    }()
}
```

### Testing for Race Conditions

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"sync"
	"testing"
)

// RaceTest is a standard pattern for testing HTTP handlers for races.
// Key: use httptest.NewServer (not httptest.NewRecorder) so that
// requests are actually handled in separate goroutines.
func TestHandler_NoRace(t *testing.T) {
	handler := NewMyHandler() // Your handler
	server := httptest.NewServer(handler)
	defer server.Close()

	var wg sync.WaitGroup
	for i := 0; i < 200; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()

			// Mix of different operations to maximize race surface.
			switch n % 3 {
			case 0:
				http.Get(server.URL + "/api/read")
			case 1:
				http.Post(server.URL+"/api/write", "application/json", nil)
			case 2:
				http.Get(server.URL + "/api/stats")
			}
		}(i)
	}
	wg.Wait()
}

// Always run with: go test -race -count=10 ./...
// -count=10 runs the test 10 times -- races are non-deterministic,
// so more runs = higher chance of catching them.

type MyHandler struct{}

func NewMyHandler() *MyHandler {
	return &MyHandler{}
}

func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("OK"))
}
```

### Race Detector Performance Impact

```
Without -race:  Normal performance
With -race:     ~2-10x slower, ~5-10x more memory

Use -race for:
  ✓ All tests in CI (go test -race ./...)
  ✓ Integration tests
  ✓ Staging environment (build with -race)
  ✓ Development/debugging

Do NOT use -race for:
  ✗ Production builds (performance overhead)
  ✗ Benchmarks (skews results)
```

---

## 14. Real-World Concurrency Bugs

### Bug 1: Concurrent Map Write (The Most Common Go Web App Crash)

```go
package main

// BUG: This will crash with "fatal error: concurrent map read and map write"
// in production, usually at 3 AM under peak traffic.

type SessionStore struct {
	sessions map[string]*Session
}

type Session struct {
	UserID    string
	ExpiresAt time.Time
}

// Every HTTP request (goroutine) calls Get and Set concurrently.
func (s *SessionStore) Get(id string) *Session {
	return s.sessions[id] // CONCURRENT READ
}

func (s *SessionStore) Set(id string, sess *Session) {
	s.sessions[id] = sess // CONCURRENT WRITE → CRASH
}
```

**Fix:**

```go
package main

import (
	"sync"
	"time"
)

type Session struct {
	UserID    string
	ExpiresAt time.Time
}

type SessionStore struct {
	mu       sync.RWMutex
	sessions map[string]*Session
}

func NewSessionStore() *SessionStore {
	return &SessionStore{
		sessions: make(map[string]*Session),
	}
}

func (s *SessionStore) Get(id string) *Session {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.sessions[id]
}

func (s *SessionStore) Set(id string, sess *Session) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.sessions[id] = sess
}

func (s *SessionStore) Delete(id string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	delete(s.sessions, id)
}
```

### Bug 2: Writing to ResponseWriter After Handler Returns

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

// BUG: The goroutine outlives the handler. When it tries to write to w,
// the HTTP response has already been sent and the connection may be reused.
func buggyAsyncHandler(w http.ResponseWriter, r *http.Request) {
	// Start a background task
	go func() {
		result := expensiveComputation()
		// BUG: By this time, the handler has returned and the ResponseWriter
		// is invalid. This may silently fail, corrupt another response,
		// or panic.
		fmt.Fprintf(w, "Result: %s", result) // WRONG
	}()

	// Handler returns immediately. The goroutine above is still running.
	w.Write([]byte("Processing..."))
}

func expensiveComputation() string {
	time.Sleep(2 * time.Second)
	return "done"
}

// FIX: Never pass ResponseWriter to a goroutine that outlives the handler.
// Instead, use one of these patterns:

// Pattern 1: Wait for the goroutine to finish before returning.
func fixedSyncHandler(w http.ResponseWriter, r *http.Request) {
	resultCh := make(chan string, 1)
	go func() {
		resultCh <- expensiveComputation()
	}()

	select {
	case result := <-resultCh:
		fmt.Fprintf(w, "Result: %s", result)
	case <-r.Context().Done():
		log.Println("Client disconnected")
	}
}

// Pattern 2: Use a job queue and return immediately with a job ID.
func fixedAsyncHandler(w http.ResponseWriter, r *http.Request) {
	jobID := submitToJobQueue("expensive-computation")
	w.WriteHeader(http.StatusAccepted)
	fmt.Fprintf(w, `{"job_id": "%s", "status": "processing"}`, jobID)
}

func submitToJobQueue(taskName string) string {
	// Submit to worker pool (see Section 11).
	return "job-12345"
}

func main() {
	http.HandleFunc("/sync", fixedSyncHandler)
	http.HandleFunc("/async", fixedAsyncHandler)
	http.ListenAndServe(":8080", nil)
}
```

### Bug 3: Goroutine Leak (The Silent Killer)

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

// BUG: This function creates a goroutine that never terminates.
// Every call to buggyFetch leaks one goroutine.
// After 10,000 requests: 10,000 leaked goroutines consuming memory.
func buggyFetch(url string) (string, error) {
	ch := make(chan string) // UNBUFFERED

	go func() {
		// Simulate HTTP call
		time.Sleep(5 * time.Second)
		ch <- "response data" // BLOCKS FOREVER if nobody reads from ch
	}()

	// Give up after 1 second.
	select {
	case result := <-ch:
		return result, nil
	case <-time.After(1 * time.Second):
		return "", fmt.Errorf("timeout")
		// The goroutine is still running!
		// It will block on ch <- "response data" forever
		// because nobody will ever read from ch.
	}
}

// FIX: Use a buffered channel or context cancellation.
func fixedFetch(ctx context.Context, url string) (string, error) {
	ch := make(chan string, 1) // BUFFERED: goroutine can send even if nobody reads

	go func() {
		// Simulate HTTP call that respects context
		select {
		case <-time.After(5 * time.Second):
			ch <- "response data"
		case <-ctx.Done():
			// Context cancelled -- exit cleanly
			return
		}
	}()

	select {
	case result := <-ch:
		return result, nil
	case <-ctx.Done():
		return "", ctx.Err()
	}
}

func main() {
	fmt.Printf("Goroutines before: %d\n", runtime.NumGoroutine())

	// Demonstrate the leak
	for i := 0; i < 10; i++ {
		buggyFetch("http://example.com")
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("Goroutines after buggyFetch x10: %d (leaked!)\n", runtime.NumGoroutine())

	// Demonstrate the fix
	for i := 0; i < 10; i++ {
		ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
		fixedFetch(ctx, "http://example.com")
		cancel() // Always cancel to clean up
	}
	time.Sleep(100 * time.Millisecond)
	fmt.Printf("Goroutines after fixedFetch x10: %d (no leak)\n", runtime.NumGoroutine())
}
```

### Bug 4: Forgetting to Close Rows (Connection Pool Exhaustion)

```go
package main

import (
	"context"
	"fmt"
	"log"
)

// BUG: Forgetting to close rows leaks a database connection.
// After MaxConns calls, all connections are leaked and the pool is exhausted.
// New queries block forever (or until timeout).
func buggyQuery(ctx context.Context, pool interface{}) {
	// rows, err := pool.Query(ctx, "SELECT id, name FROM users")
	// if err != nil {
	//     log.Fatal(err)
	// }
	//
	// for rows.Next() {
	//     var id int
	//     var name string
	//     rows.Scan(&id, &name)
	//     fmt.Printf("User: %d %s\n", id, name)
	// }
	//
	// BUG: rows.Close() is never called!
	// The connection is NEVER returned to the pool.

	fmt.Println("Query function returned, but connection is still checked out")
}

// FIX: Always defer rows.Close().
func fixedQuery(ctx context.Context, pool interface{}) {
	// rows, err := pool.Query(ctx, "SELECT id, name FROM users")
	// if err != nil {
	//     log.Fatal(err)
	// }
	// defer rows.Close() // ALWAYS close rows
	//
	// for rows.Next() {
	//     var id int
	//     var name string
	//     rows.Scan(&id, &name)
	//     fmt.Printf("User: %d %s\n", id, name)
	// }
	//
	// // Also check rows.Err() -- iteration might have failed.
	// if err := rows.Err(); err != nil {
	//     log.Printf("Error iterating rows: %v", err)
	// }

	log.Println("Query function returned, connection returned to pool")
}
```

### Bug 5: Mutex Copied by Value

```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu    sync.Mutex
	count int
}

// BUG: Value receiver copies the entire struct, INCLUDING the mutex.
// Each call gets its own copy of the mutex -- the lock protects nothing.
func (c SafeCounter) BuggyIncrement() {
	c.mu.Lock()         // Locks a COPY of the mutex
	defer c.mu.Unlock()
	c.count++           // Increments a COPY of the count
	// Both the lock and the increment are lost when the function returns
}

// FIX: Always use pointer receivers on types that contain sync primitives.
func (c *SafeCounter) FixedIncrement() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}

func (c *SafeCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}

func main() {
	c := &SafeCounter{}

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.FixedIncrement()
		}()
	}
	wg.Wait()

	fmt.Printf("Count: %d (expected: 1000)\n", c.Value())
}
```

### Summary of Real-World Bugs

```
┌──────────────────────────────┬─────────────────────────────────────────┐
│ Bug                          │ Prevention                              │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Concurrent map write         │ Always protect maps with sync.Mutex    │
│                              │ or sync.RWMutex                        │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Write to ResponseWriter      │ Never pass w to goroutines that        │
│ after handler returns        │ outlive the handler                    │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Goroutine leak               │ Every goroutine must have a guaranteed │
│                              │ exit path (context, buffered channel)  │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Connection pool exhaustion   │ Always defer rows.Close() / conn.      │
│                              │ Release()                              │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Mutex copied by value        │ Use pointer receivers on types with    │
│                              │ sync primitives. Use go vet.           │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Deadlock from lock ordering  │ Always acquire multiple locks in       │
│                              │ consistent order (e.g., by address)    │
├──────────────────────────────┼─────────────────────────────────────────┤
│ Race on error handling path  │ Run go test -race -count=10. Test      │
│                              │ error paths as thoroughly as happy      │
│                              │ paths.                                  │
└──────────────────────────────┴─────────────────────────────────────────┘
```

---

## 15. Complete Working System: Rate Limiter + Background Cleanup + Parallel Queries

This section brings together every pattern from the chapter into a single, complete, runnable program. It implements a realistic API server with:

- Per-IP rate limiting (Section 2)
- Background cleanup of stale visitors (Section 3)
- Context cancellation (Section 4)
- Parallel database queries (Section 6)
- Background token cleanup (Section 7)
- Request metrics (Section 9)
- Graceful shutdown (Section 12)

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"sync/atomic"
	"syscall"
	"time"

	"golang.org/x/sync/errgroup"
	"golang.org/x/time/rate"
)

// ============================================================
// Domain Types
// ============================================================

type Stats struct {
	TotalSessions int     `json:"totalSessions"`
	TotalMinutes  int     `json:"totalMinutes"`
	AverageScore  float64 `json:"averageScore"`
}

type Session struct {
	ID        string    `json:"id"`
	Topic     string    `json:"topic"`
	Score     int       `json:"score"`
	CreatedAt time.Time `json:"createdAt"`
}

type Progress struct {
	Level         int     `json:"level"`
	XP            int     `json:"xp"`
	CompletionPct float64 `json:"completionPct"`
}

type DashboardResponse struct {
	Stats    *Stats     `json:"stats"`
	Sessions []*Session `json:"sessions"`
	Progress *Progress  `json:"progress"`
}

// ============================================================
// Simulated Repositories (replace with real DB calls)
// ============================================================

type StatsRepo struct{}

func (r *StatsRepo) GetStats(ctx context.Context, userID string) (*Stats, error) {
	delay := time.Duration(50+rand.Intn(100)) * time.Millisecond
	select {
	case <-time.After(delay):
		return &Stats{
			TotalSessions: 142,
			TotalMinutes:  4260,
			AverageScore:  87.3,
		}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

type SessionRepo struct{}

func (r *SessionRepo) GetRecent(ctx context.Context, userID string) ([]*Session, error) {
	delay := time.Duration(80+rand.Intn(120)) * time.Millisecond
	select {
	case <-time.After(delay):
		return []*Session{
			{ID: "s1", Topic: "Goroutines", Score: 92, CreatedAt: time.Now().Add(-1 * time.Hour)},
			{ID: "s2", Topic: "Channels", Score: 88, CreatedAt: time.Now().Add(-2 * time.Hour)},
			{ID: "s3", Topic: "Context", Score: 95, CreatedAt: time.Now().Add(-3 * time.Hour)},
		}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

type ProgressRepo struct{}

func (r *ProgressRepo) GetProgress(ctx context.Context, userID string) (*Progress, error) {
	delay := time.Duration(30+rand.Intn(70)) * time.Millisecond
	select {
	case <-time.After(delay):
		return &Progress{
			Level:         12,
			XP:            3400,
			CompletionPct: 68.5,
		}, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

type TokenRepo struct{}

func (r *TokenRepo) DeleteExpiredTokens(ctx context.Context) (int64, error) {
	select {
	case <-time.After(50 * time.Millisecond):
		// Simulate finding 0-3 expired tokens
		return int64(rand.Intn(4)), nil
	case <-ctx.Done():
		return 0, ctx.Err()
	}
}

// ============================================================
// Rate Limiter (Sections 2 & 3)
// ============================================================

type visitor struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

type RateLimiter struct {
	mu       sync.Mutex
	visitors map[string]*visitor
	rate     rate.Limit
	burst    int
	wg       sync.WaitGroup
}

func NewRateLimiter(ctx context.Context, r rate.Limit, b int) *RateLimiter {
	rl := &RateLimiter{
		visitors: make(map[string]*visitor),
		rate:     r,
		burst:    b,
	}
	rl.wg.Add(1)
	go func() {
		defer rl.wg.Done()
		rl.cleanup(ctx)
	}()
	return rl
}

func (rl *RateLimiter) Allow(ip string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	v, exists := rl.visitors[ip]
	if !exists {
		v = &visitor{limiter: rate.NewLimiter(rl.rate, rl.burst)}
		rl.visitors[ip] = v
	}
	v.lastSeen = time.Now()
	return v.limiter.Allow()
}

func (rl *RateLimiter) cleanup(ctx context.Context) {
	ticker := time.NewTicker(1 * time.Minute)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			log.Println("[rate-limiter] cleanup stopped")
			return
		case <-ticker.C:
			rl.mu.Lock()
			before := len(rl.visitors)
			for ip, v := range rl.visitors {
				if time.Since(v.lastSeen) > 3*time.Minute {
					delete(rl.visitors, ip)
				}
			}
			after := len(rl.visitors)
			rl.mu.Unlock()
			if before != after {
				log.Printf("[rate-limiter] cleaned %d stale entries (%d remaining)",
					before-after, after)
			}
		}
	}
}

func (rl *RateLimiter) VisitorCount() int {
	rl.mu.Lock()
	defer rl.mu.Unlock()
	return len(rl.visitors)
}

func (rl *RateLimiter) Stop() {
	rl.wg.Wait()
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if !rl.Allow(r.RemoteAddr) {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusTooManyRequests)
			json.NewEncoder(w).Encode(map[string]string{
				"error": "rate limit exceeded",
			})
			return
		}
		next.ServeHTTP(w, r)
	})
}

// ============================================================
// Token Cleanup Service (Section 7)
// ============================================================

type TokenCleanupService struct {
	repo     *TokenRepo
	interval time.Duration
	wg       sync.WaitGroup
}

func NewTokenCleanupService(repo *TokenRepo, interval time.Duration) *TokenCleanupService {
	return &TokenCleanupService{repo: repo, interval: interval}
}

func (s *TokenCleanupService) Start(ctx context.Context) {
	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		ticker := time.NewTicker(s.interval)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				log.Println("[token-cleanup] stopped")
				return
			case <-ticker.C:
				cleanCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
				deleted, err := s.repo.DeleteExpiredTokens(cleanCtx)
				cancel()
				if err != nil {
					if ctx.Err() == nil {
						log.Printf("[token-cleanup] error: %v", err)
					}
					continue
				}
				if deleted > 0 {
					log.Printf("[token-cleanup] deleted %d expired tokens", deleted)
				}
			}
		}
	}()
}

func (s *TokenCleanupService) Stop() {
	s.wg.Wait()
}

// ============================================================
// Request Metrics (Section 9)
// ============================================================

type Metrics struct {
	totalRequests  atomic.Int64
	activeRequests atomic.Int64
	totalErrors    atomic.Int64
}

func (m *Metrics) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		m.totalRequests.Add(1)
		m.activeRequests.Add(1)
		defer m.activeRequests.Add(-1)

		sw := &statusCapture{ResponseWriter: w, status: 200}
		next.ServeHTTP(sw, r)

		if sw.status >= 500 {
			m.totalErrors.Add(1)
		}
	})
}

func (m *Metrics) Stats() map[string]int64 {
	return map[string]int64{
		"total_requests":  m.totalRequests.Load(),
		"active_requests": m.activeRequests.Load(),
		"total_errors":    m.totalErrors.Load(),
	}
}

type statusCapture struct {
	http.ResponseWriter
	status int
}

func (w *statusCapture) WriteHeader(code int) {
	w.status = code
	w.ResponseWriter.WriteHeader(code)
}

// ============================================================
// Dashboard Handler (Section 6)
// ============================================================

type DashboardHandler struct {
	statsRepo    *StatsRepo
	sessionRepo  *SessionRepo
	progressRepo *ProgressRepo
}

func (h *DashboardHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	userID := "user-123" // In production: extract from auth middleware

	g, gCtx := errgroup.WithContext(ctx)

	var stats *Stats
	var sessions []*Session
	var progress *Progress

	g.Go(func() error {
		var err error
		stats, err = h.statsRepo.GetStats(gCtx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		sessions, err = h.sessionRepo.GetRecent(gCtx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		progress, err = h.progressRepo.GetProgress(gCtx, userID)
		return err
	})

	if err := g.Wait(); err != nil {
		if ctx.Err() != nil {
			log.Printf("[dashboard] request cancelled: %v", ctx.Err())
			return
		}
		log.Printf("[dashboard] error: %v", err)
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusInternalServerError)
		json.NewEncoder(w).Encode(map[string]string{"error": "failed to load dashboard"})
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(DashboardResponse{
		Stats:    stats,
		Sessions: sessions,
		Progress: progress,
	})
}

// ============================================================
// Health Check Handler
// ============================================================

type HealthHandler struct {
	metrics  *Metrics
	limiter  *RateLimiter
}

func (h *HealthHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]any{
		"status":   "ok",
		"metrics":  h.metrics.Stats(),
		"visitors": h.limiter.VisitorCount(),
	})
}

// ============================================================
// Main: Wire everything together with graceful shutdown
// ============================================================

func main() {
	log.SetFlags(log.Ltime | log.Lshortfile)

	// Background context for all background goroutines.
	bgCtx, bgCancel := context.WithCancel(context.Background())

	// Initialize services.
	limiter := NewRateLimiter(bgCtx, 10, 20) // 10 req/s, burst 20
	metrics := &Metrics{}

	tokenCleaner := NewTokenCleanupService(&TokenRepo{}, 1*time.Hour)
	tokenCleaner.Start(bgCtx)

	dashboardHandler := &DashboardHandler{
		statsRepo:    &StatsRepo{},
		sessionRepo:  &SessionRepo{},
		progressRepo: &ProgressRepo{},
	}

	healthHandler := &HealthHandler{
		metrics: metrics,
		limiter: limiter,
	}

	// Routes.
	mux := http.NewServeMux()
	mux.Handle("/api/dashboard", dashboardHandler)
	mux.Handle("/health", healthHandler)
	mux.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, `{"pong": true}`)
	})

	// Middleware chain: metrics → rate limiter → router.
	handler := metrics.Middleware(limiter.Middleware(mux))

	server := &http.Server{
		Addr:              ":8080",
		Handler:           handler,
		ReadTimeout:       15 * time.Second,
		ReadHeaderTimeout: 5 * time.Second,
		WriteTimeout:      30 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// Start HTTP server.
	serverErr := make(chan error, 1)
	go func() {
		log.Println("Server starting on :8080")
		log.Println("  GET /api/dashboard  -- parallel queries demo")
		log.Println("  GET /api/ping       -- simple ping")
		log.Println("  GET /health         -- health check + metrics")
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			serverErr <- err
		}
		close(serverErr)
	}()

	// Wait for shutdown signal.
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case sig := <-quit:
		log.Printf("Received %s signal", sig)
	case err := <-serverErr:
		log.Printf("Server error: %v", err)
	}

	// Graceful shutdown sequence.
	log.Println("=== GRACEFUL SHUTDOWN STARTING ===")

	// 1. Stop accepting new requests, wait for in-flight.
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer shutdownCancel()
	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Printf("HTTP shutdown error: %v", err)
	}
	log.Println("HTTP server stopped")

	// 2. Cancel all background goroutines.
	bgCancel()

	// 3. Wait for background goroutines to finish.
	limiter.Stop()
	tokenCleaner.Stop()
	log.Println("Background services stopped")

	// 4. Close database pool (if we had one).
	// pool.Close()

	log.Println("=== SHUTDOWN COMPLETE ===")
}
```

### Testing the Complete System

```bash
# Terminal 1: Run the server
go run main.go

# Terminal 2: Test the dashboard (parallel queries)
curl -s localhost:8080/api/dashboard | jq .

# Terminal 3: Test rate limiting (send 25 rapid requests)
for i in $(seq 1 25); do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" localhost:8080/api/ping
done
# First ~20 should be 200, then 429 (rate limited)

# Terminal 4: Check health/metrics
curl -s localhost:8080/health | jq .

# Terminal 5: Graceful shutdown
kill -SIGTERM $(pgrep -f "go run main.go")
# Watch Terminal 1 for the shutdown sequence
```

### What This System Demonstrates

```
┌─────────────────────────────────────────────────────────────────┐
│  CONCURRENCY PRIMITIVES USED IN THIS SYSTEM                    │
├──────────────────────┬──────────────────────────────────────────┤
│ Primitive            │ Where Used                               │
├──────────────────────┼──────────────────────────────────────────┤
│ goroutine per request│ net/http server (automatic)              │
│ sync.Mutex           │ RateLimiter.visitors map                 │
│ sync/atomic          │ Metrics counters                         │
│ time.Ticker          │ Rate limiter cleanup, token cleanup      │
│ context.Context      │ Request cancellation, shutdown signal    │
│ errgroup             │ Parallel dashboard queries               │
│ sync.WaitGroup       │ Background service lifecycle             │
│ channels             │ Server error, shutdown signal            │
│ select               │ Ticker loops, timeout handling           │
│ context.WithTimeout  │ Per-operation timeouts                   │
│ context.WithCancel   │ Background goroutine lifecycle           │
└──────────────────────┴──────────────────────────────────────────┘
```

---

## 16. Key Takeaways

1. **Every HTTP request is a goroutine.** Go's HTTP server spawns a goroutine per incoming connection. This is fundamentally different from Node.js's event loop. In Go, blocking is normal and expected. In Node.js, blocking is fatal to all requests.

2. **Shared maps must be protected.** The most common crash in Go web applications is a concurrent map write. Every map accessed from multiple goroutines needs a `sync.Mutex` or `sync.RWMutex`. No exceptions.

3. **Context is the cancellation backbone.** Every `*http.Request` carries a context that is cancelled when the client disconnects. Pass this context to every downstream call (database, HTTP client, file I/O). This gives you free cancellation propagation through your entire call stack.

4. **Background goroutines need lifecycle management.** Every background goroutine must have: (a) a context for cancellation, (b) a `sync.WaitGroup` for shutdown coordination, (c) `time.Ticker` for periodic work (not `time.Sleep`), and (d) a `select` loop watching `ctx.Done()`.

5. **errgroup is the standard tool for parallel operations.** When you need to run multiple operations concurrently and collect errors, use `errgroup.WithContext`. It handles WaitGroup management, error collection, and context cancellation automatically.

6. **Connection pools are concurrency primitives.** A database connection pool is essentially a semaphore. Understand your pool size, set context timeouts for connection acquisition, always close/release resources with `defer`, and monitor pool stats.

7. **Use the right primitive for the job.** `sync/atomic` for counters and flags. `sync.Mutex` for protecting data with frequent writes. `sync.RWMutex` for read-heavy data. Channels for communication between goroutines. `sync.Once` for one-time initialization. `errgroup` for parallel operations.

8. **Graceful shutdown has a specific order.** Stop accepting new requests, wait for in-flight requests, cancel background goroutines, wait for them to finish, close the database pool. Getting the order wrong causes crashes or data loss.

9. **The race detector is not optional.** Run `go test -race ./...` in every CI pipeline. Race conditions are non-deterministic -- they work 99% of the time and crash at 3 AM under peak load. The race detector catches them deterministically at test time.

10. **Never write to ResponseWriter from a goroutine that outlives the handler.** The ResponseWriter is only valid during the handler's execution. If you need async processing, use a job queue and return a job ID.

11. **Every goroutine must have a guaranteed exit path.** If you spawn a goroutine, you must be able to answer: "What makes this goroutine stop?" If the answer is "nothing," you have a goroutine leak. Context cancellation and buffered channels are your primary tools.

12. **Concurrency in Go web apps is not optional -- it is the default.** Unlike Node.js where you opt into concurrency with worker threads, Go makes concurrency the default by handling each request in its own goroutine. Your job is not to "add concurrency" but to "correctly synchronize the concurrency that already exists."

---

## 17. Practice Exercises

### Exercise 1: Rate Limiter with Different Limits per Route

Extend the rate limiter from Section 2 to support different rate limits for different routes:

- `/api/auth/login`: 3 requests per minute (brute-force protection)
- `/api/data/*`: 30 requests per second (normal API usage)
- `/api/health`: No rate limit

Requirements:
- Use a map of route patterns to rate configurations
- Each route-IP combination should have its own `rate.Limiter`
- Include background cleanup for all route-IP combinations
- Write a test that verifies different routes have different limits

### Exercise 2: Parallel API Aggregator

Build an HTTP handler that fetches data from three external APIs in parallel and combines the results. Simulate the external APIs as separate goroutines with random delays and occasional failures.

Requirements:
- Use `errgroup.WithContext` for parallel fetching
- If any API call fails, cancel the others and return an error
- Add a per-request timeout of 5 seconds
- If the client disconnects, all API calls should be cancelled
- Track metrics: total requests, average response time, error count per API
- Write a test that verifies cancellation works correctly

### Exercise 3: In-Memory Cache with TTL and Background Eviction

Build a concurrent-safe in-memory cache with:

```go
type Cache[K comparable, V any] struct {
    // Your implementation
}

func (c *Cache[K, V]) Get(key K) (V, bool)
func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration)
func (c *Cache[K, V]) Delete(key K)
func (c *Cache[K, V]) Len() int
```

Requirements:
- Thread-safe using `sync.RWMutex` (reads with `RLock`, writes with `Lock`)
- TTL-based expiration: items expire after their TTL
- Background eviction goroutine that runs every 30 seconds and removes expired items
- Context-based lifecycle (background goroutine stops when context is cancelled)
- Run `go test -race -count=100` to verify no race conditions
- Benchmark `Get` performance with 100 concurrent goroutines

### Exercise 4: Graceful Shutdown with Drain

Build a complete HTTP server with:

1. A handler that simulates a 2-second database operation
2. A background worker pool processing jobs from a channel
3. On SIGTERM:
   - Stop accepting new HTTP requests
   - Wait for in-flight requests to finish (up to 30 seconds)
   - Stop the worker pool (let current jobs finish, discard queued jobs)
   - Log the complete shutdown timeline with timestamps

Test by:
1. Starting the server
2. Sending 10 concurrent requests (each takes 2 seconds)
3. Immediately sending SIGTERM
4. Verifying all 10 requests complete successfully
5. Verifying the worker pool stopped cleanly

### Exercise 5: Find and Fix the Concurrency Bugs

The following server has five concurrency bugs. Find and fix all of them.

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

var requestLog = map[string][]time.Time{}
var userSessions = map[string]string{}

func logRequest(ip string) {
	requestLog[ip] = append(requestLog[ip], time.Now())
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
	username := r.FormValue("username")
	sessionID := fmt.Sprintf("sess-%d", time.Now().UnixNano())
	userSessions[username] = sessionID

	go func() {
		time.Sleep(1 * time.Second)
		fmt.Fprintf(w, "Welcome, %s! Session: %s", username, sessionID)
	}()
}

func statsHandler(w http.ResponseWriter, r *http.Request) {
	logRequest(r.RemoteAddr)
	total := 0
	for _, times := range requestLog {
		total += len(times)
	}
	fmt.Fprintf(w, "Total requests logged: %d", total)
}

type Counter struct {
	mu    sync.Mutex
	count int
}

func (c Counter) Increment() {
	c.mu.Lock()
	c.count++
	c.mu.Unlock()
}

func main() {
	http.HandleFunc("/login", loginHandler)
	http.HandleFunc("/stats", statsHandler)
	http.ListenAndServe(":8080", nil)
}
```

**Bugs to find:**
1. `requestLog` -- concurrent map writes without mutex
2. `userSessions` -- concurrent map writes without mutex
3. `loginHandler` -- writes to `ResponseWriter` after handler returns (goroutine outlives handler)
4. `Counter.Increment` -- value receiver copies the mutex
5. `statsHandler` -- reads and writes `requestLog` without synchronization (both `logRequest` and the iteration are unprotected)

Fix all five bugs, then run `go test -race` to verify your fixes.

### Exercise 6: Monitor Goroutine Count

Build a middleware that tracks the goroutine count over time and exposes it as an endpoint:

```
GET /debug/goroutines
{
    "current": 42,
    "peak": 156,
    "samples": [
        {"time": "2024-01-15T10:00:00Z", "count": 42},
        {"time": "2024-01-15T10:00:01Z", "count": 87},
        ...
    ]
}
```

Requirements:
- Sample `runtime.NumGoroutine()` every second in a background goroutine
- Keep the last 300 samples (5 minutes) in a ring buffer
- Track peak goroutine count
- Thread-safe access to the samples slice
- Context-based lifecycle for the sampling goroutine

---

> **Next Chapter:** Chapter 32 will cover Authentication and Security Patterns in Go APIs -- JWT handling, bcrypt password hashing, secure middleware chains, CORS, CSRF protection, and secrets management.
