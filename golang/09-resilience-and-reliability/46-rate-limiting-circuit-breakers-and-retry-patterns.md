# Chapter 46: Rate Limiting, Circuit Breakers, and Retry Patterns in Go

## Table of Contents

1. [Introduction](#introduction)
2. [Rate Limiting Fundamentals](#rate-limiting-fundamentals)
3. [Token Bucket Algorithm](#token-bucket-algorithm)
4. [Sliding Window Rate Limiting](#sliding-window-rate-limiting)
5. [Per-Client and Per-Endpoint Rate Limiting](#per-client-and-per-endpoint-rate-limiting)
6. [Distributed Rate Limiting with Redis](#distributed-rate-limiting-with-redis)
7. [Circuit Breaker Pattern](#circuit-breaker-pattern)
8. [Building a Circuit Breaker from Scratch](#building-a-circuit-breaker-from-scratch)
9. [Using sony/gobreaker](#using-sonygobreaker)
10. [Using hystrix-go](#using-hystrix-go)
11. [Retry Patterns](#retry-patterns)
12. [Idempotency and Safe Retries](#idempotency-and-safe-retries)
13. [Bulkhead Pattern for Isolation](#bulkhead-pattern-for-isolation)
14. [Timeout Patterns and Cascading Failure Prevention](#timeout-patterns-and-cascading-failure-prevention)
15. [Health-Based Load Shedding](#health-based-load-shedding)
16. [Combining Patterns: Retry + Circuit Breaker + Rate Limit](#combining-patterns-retry--circuit-breaker--rate-limit)
17. [Middleware Implementations for HTTP Servers](#middleware-implementations-for-http-servers)
18. [Testing Resilience Patterns](#testing-resilience-patterns)
19. [Summary and Best Practices](#summary-and-best-practices)

---

## Introduction

In distributed systems, failures are not a matter of *if* but *when*. Networks partition, services crash, databases slow down, and traffic spikes arrive without warning. Building resilient Go applications means anticipating these failures and handling them gracefully.

This chapter covers three foundational resilience patterns:

| Pattern | Purpose | Analogy |
|---------|---------|---------|
| **Rate Limiting** | Control the rate of incoming or outgoing requests | A bouncer at a club door |
| **Circuit Breaker** | Stop calling a failing service to let it recover | An electrical circuit breaker |
| **Retry** | Automatically retry transient failures | Redialing a dropped phone call |

These patterns, combined with supporting techniques like bulkheads, timeouts, and load shedding, form the backbone of production-grade Go services.

---

## Rate Limiting Fundamentals

Rate limiting controls how many operations can occur within a given time window. It serves multiple purposes:

- **Protecting services** from being overwhelmed by traffic
- **Ensuring fair usage** across clients
- **Preventing cascade failures** when downstream services are stressed
- **Complying with API contracts** when calling third-party services

### Common Algorithms

| Algorithm | How It Works | Pros | Cons |
|-----------|-------------|------|------|
| **Token Bucket** | Tokens accumulate at a fixed rate; each request consumes a token | Allows bursts, smooth average rate | Slightly more complex |
| **Leaky Bucket** | Requests queue and drain at a constant rate | Perfectly smooth output rate | No burst tolerance |
| **Fixed Window** | Count requests per fixed time window (e.g., per minute) | Simple to implement | Boundary spike problem |
| **Sliding Window Log** | Track timestamps of all requests in window | Precise counting | Memory-intensive |
| **Sliding Window Counter** | Weighted average of current and previous window | Good balance of precision and efficiency | Approximate |

---

## Token Bucket Algorithm

The token bucket is the most commonly used rate limiting algorithm in Go. Tokens are added to a bucket at a fixed rate. Each request removes a token. If the bucket is empty, the request is denied or delayed.

### Implementation from Scratch

```go
package ratelimit

import (
    "sync"
    "time"
)

// TokenBucket implements the token bucket rate limiting algorithm.
type TokenBucket struct {
    mu         sync.Mutex
    capacity   float64   // maximum number of tokens
    tokens     float64   // current number of tokens
    rate       float64   // tokens added per second
    lastRefill time.Time // last time tokens were added
}

// NewTokenBucket creates a new token bucket.
// rate: tokens per second
// capacity: maximum burst size
func NewTokenBucket(rate float64, capacity int) *TokenBucket {
    return &TokenBucket{
        capacity:   float64(capacity),
        tokens:     float64(capacity), // start full
        rate:       rate,
        lastRefill: time.Now(),
    }
}

// refill adds tokens based on elapsed time since last refill.
// Must be called with mu held.
func (tb *TokenBucket) refill() {
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += elapsed * tb.rate
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastRefill = now
}

// Allow checks if a single request is allowed.
func (tb *TokenBucket) Allow() bool {
    return tb.AllowN(1)
}

// AllowN checks if n tokens are available and consumes them.
func (tb *TokenBucket) AllowN(n int) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    tb.refill()

    if tb.tokens >= float64(n) {
        tb.tokens -= float64(n)
        return true
    }
    return false
}

// Wait blocks until a token is available or the context expires.
func (tb *TokenBucket) Wait(ctx context.Context) error {
    for {
        if tb.Allow() {
            return nil
        }

        // Calculate wait time for next token
        tb.mu.Lock()
        waitDuration := time.Duration(float64(time.Second) / tb.rate)
        tb.mu.Unlock()

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(waitDuration):
            // Try again after waiting
        }
    }
}

// Tokens returns the current number of available tokens.
func (tb *TokenBucket) Tokens() float64 {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    tb.refill()
    return tb.tokens
}
```

### Usage Example

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Allow 10 requests per second, burst of 20
    limiter := NewTokenBucket(10, 20)

    // Simulate burst of requests
    for i := 0; i < 25; i++ {
        if limiter.Allow() {
            fmt.Printf("Request %d: ALLOWED\n", i+1)
        } else {
            fmt.Printf("Request %d: DENIED\n", i+1)
        }
    }

    // Wait a bit for tokens to refill
    fmt.Println("\nWaiting 1 second...")
    time.Sleep(time.Second)

    // Now 10 more tokens should be available
    for i := 0; i < 12; i++ {
        if limiter.Allow() {
            fmt.Printf("Request %d: ALLOWED\n", i+1)
        } else {
            fmt.Printf("Request %d: DENIED\n", i+1)
        }
    }
}
```

### Using golang.org/x/time/rate

The standard extended library provides a production-quality token bucket implementation.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

func main() {
    // rate.Limit is tokens per second. rate.Every converts a duration.
    // Allow 5 events per second, with a burst size of 10.
    limiter := rate.NewLimiter(rate.Every(200*time.Millisecond), 10)

    // --- Method 1: Allow() - non-blocking, returns immediately ---
    fmt.Println("=== Allow() - Non-blocking ===")
    for i := 0; i < 15; i++ {
        if limiter.Allow() {
            fmt.Printf("  Request %d: allowed\n", i+1)
        } else {
            fmt.Printf("  Request %d: rate limited\n", i+1)
        }
    }

    // --- Method 2: Wait() - blocking, waits for a token ---
    fmt.Println("\n=== Wait() - Blocking ===")
    limiter2 := rate.NewLimiter(rate.Every(100*time.Millisecond), 1)

    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()

    for i := 0; i < 10; i++ {
        err := limiter2.Wait(ctx)
        if err != nil {
            fmt.Printf("  Request %d: %v\n", i+1, err)
            break
        }
        fmt.Printf("  Request %d: allowed at %v\n", i+1, time.Now().Format("15:04:05.000"))
    }

    // --- Method 3: Reserve() - get a reservation for future execution ---
    fmt.Println("\n=== Reserve() - Planned Execution ===")
    limiter3 := rate.NewLimiter(2, 1) // 2 per second, burst 1

    for i := 0; i < 5; i++ {
        reservation := limiter3.Reserve()
        if !reservation.OK() {
            fmt.Println("  Cannot make reservation")
            continue
        }
        delay := reservation.Delay()
        fmt.Printf("  Request %d: must wait %v\n", i+1, delay)
        time.Sleep(delay)
        fmt.Printf("  Request %d: executed\n", i+1)
    }

    // --- Concurrent usage ---
    fmt.Println("\n=== Concurrent Usage ===")
    limiter4 := rate.NewLimiter(10, 5) // 10/s, burst 5
    var wg sync.WaitGroup

    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
            defer cancel()

            if err := limiter4.Wait(ctx); err != nil {
                fmt.Printf("  Goroutine %d: timed out\n", id)
                return
            }
            fmt.Printf("  Goroutine %d: processed\n", id)
        }(i)
    }
    wg.Wait()
}
```

> **Tip:** `rate.Inf` disables rate limiting entirely, useful for testing or admin endpoints.

> **Warning:** `Allow()` and `Reserve()` are non-blocking. `Wait()` blocks. Choose the right method based on whether you want to reject, delay, or schedule requests.

---

## Sliding Window Rate Limiting

The fixed-window counter has a well-known boundary problem: a client can send the maximum requests at the end of one window and the beginning of the next, effectively doubling the allowed rate in a short burst. The sliding window approach solves this.

### Sliding Window Log

```go
package ratelimit

import (
    "sync"
    "time"
)

// SlidingWindowLog tracks each request timestamp and counts
// requests within the sliding window.
type SlidingWindowLog struct {
    mu         sync.Mutex
    timestamps []time.Time
    limit      int
    window     time.Duration
}

func NewSlidingWindowLog(limit int, window time.Duration) *SlidingWindowLog {
    return &SlidingWindowLog{
        timestamps: make([]time.Time, 0, limit),
        limit:      limit,
        window:     window,
    }
}

// Allow checks if a request is within the rate limit.
func (sw *SlidingWindowLog) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now()
    windowStart := now.Add(-sw.window)

    // Remove expired timestamps
    validIdx := 0
    for i, ts := range sw.timestamps {
        if ts.After(windowStart) {
            validIdx = i
            break
        }
        if i == len(sw.timestamps)-1 {
            validIdx = len(sw.timestamps) // all expired
        }
    }
    sw.timestamps = sw.timestamps[validIdx:]

    // Check limit
    if len(sw.timestamps) >= sw.limit {
        return false
    }

    // Record this request
    sw.timestamps = append(sw.timestamps, now)
    return true
}

// Count returns the number of requests in the current window.
func (sw *SlidingWindowLog) Count() int {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now()
    windowStart := now.Add(-sw.window)

    count := 0
    for _, ts := range sw.timestamps {
        if ts.After(windowStart) {
            count++
        }
    }
    return count
}
```

### Sliding Window Counter (Memory-Efficient)

This approach uses the weighted sum of the current and previous window counters.

```go
package ratelimit

import (
    "sync"
    "time"
)

// SlidingWindowCounter provides approximate sliding window rate limiting
// with O(1) memory per client.
type SlidingWindowCounter struct {
    mu           sync.Mutex
    prevCount    int
    currCount    int
    limit        int
    window       time.Duration
    currStart    time.Time
}

func NewSlidingWindowCounter(limit int, window time.Duration) *SlidingWindowCounter {
    return &SlidingWindowCounter{
        limit:     limit,
        window:    window,
        currStart: time.Now().Truncate(window),
    }
}

func (sw *SlidingWindowCounter) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now()
    currWindow := now.Truncate(sw.window)

    // Advance windows if needed
    if currWindow != sw.currStart {
        elapsed := currWindow.Sub(sw.currStart)
        if elapsed >= 2*sw.window {
            // Two or more windows have passed, reset everything
            sw.prevCount = 0
            sw.currCount = 0
        } else {
            // One window has passed
            sw.prevCount = sw.currCount
            sw.currCount = 0
        }
        sw.currStart = currWindow
    }

    // Calculate the weighted count
    // Weight of previous window: fraction of previous window still in our sliding window
    elapsed := now.Sub(currWindow)
    weight := 1.0 - (float64(elapsed) / float64(sw.window))

    estimatedCount := float64(sw.prevCount)*weight + float64(sw.currCount)

    if estimatedCount >= float64(sw.limit) {
        return false
    }

    sw.currCount++
    return true
}
```

### Comparison

| Aspect | Sliding Window Log | Sliding Window Counter |
|--------|-------------------|----------------------|
| **Memory** | O(n) per client (stores timestamps) | O(1) per client (two counters) |
| **Accuracy** | Exact | Approximate |
| **Speed** | O(n) cleanup | O(1) check |
| **Best For** | Low-volume, precision-critical | High-volume, memory-sensitive |

---

## Per-Client and Per-Endpoint Rate Limiting

Real applications need different rate limits for different clients, API keys, or endpoints.

### Per-Client Rate Limiter

```go
package ratelimit

import (
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// ClientRateLimiter maintains per-client rate limiters with automatic cleanup.
type ClientRateLimiter struct {
    mu       sync.RWMutex
    clients  map[string]*clientEntry
    rate     rate.Limit
    burst    int
    ttl      time.Duration // how long to keep idle clients
    stopCh   chan struct{}
}

type clientEntry struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

func NewClientRateLimiter(r rate.Limit, burst int, cleanupInterval, ttl time.Duration) *ClientRateLimiter {
    crl := &ClientRateLimiter{
        clients: make(map[string]*clientEntry),
        rate:    r,
        burst:   burst,
        ttl:     ttl,
        stopCh:  make(chan struct{}),
    }

    // Background cleanup of stale entries
    go crl.cleanupLoop(cleanupInterval)

    return crl
}

// GetLimiter returns or creates a rate limiter for the given client ID.
func (crl *ClientRateLimiter) GetLimiter(clientID string) *rate.Limiter {
    // Fast path: check if limiter already exists
    crl.mu.RLock()
    entry, exists := crl.clients[clientID]
    crl.mu.RUnlock()

    if exists {
        crl.mu.Lock()
        entry.lastSeen = time.Now()
        crl.mu.Unlock()
        return entry.limiter
    }

    // Slow path: create new limiter
    crl.mu.Lock()
    defer crl.mu.Unlock()

    // Double-check after acquiring write lock
    if entry, exists := crl.clients[clientID]; exists {
        entry.lastSeen = time.Now()
        return entry.limiter
    }

    limiter := rate.NewLimiter(crl.rate, crl.burst)
    crl.clients[clientID] = &clientEntry{
        limiter:  limiter,
        lastSeen: time.Now(),
    }
    return limiter
}

// Allow checks if a request from the given client is allowed.
func (crl *ClientRateLimiter) Allow(clientID string) bool {
    return crl.GetLimiter(clientID).Allow()
}

func (crl *ClientRateLimiter) cleanupLoop(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            crl.cleanup()
        case <-crl.stopCh:
            return
        }
    }
}

func (crl *ClientRateLimiter) cleanup() {
    crl.mu.Lock()
    defer crl.mu.Unlock()

    cutoff := time.Now().Add(-crl.ttl)
    for id, entry := range crl.clients {
        if entry.lastSeen.Before(cutoff) {
            delete(crl.clients, id)
        }
    }
}

// Stop terminates the cleanup goroutine.
func (crl *ClientRateLimiter) Stop() {
    close(crl.stopCh)
}

// ClientCount returns the number of tracked clients.
func (crl *ClientRateLimiter) ClientCount() int {
    crl.mu.RLock()
    defer crl.mu.RUnlock()
    return len(crl.clients)
}
```

### Per-Endpoint Rate Limiter with Tiered Limits

```go
package ratelimit

import (
    "net/http"
    "strings"

    "golang.org/x/time/rate"
)

// EndpointConfig defines rate limits for a specific endpoint pattern.
type EndpointConfig struct {
    Pattern string
    Rate    rate.Limit
    Burst   int
}

// TieredRateLimiter applies different rate limits based on
// the endpoint, client tier, and HTTP method.
type TieredRateLimiter struct {
    // Per-endpoint per-client limiters
    endpoints map[string]*ClientRateLimiter

    // Tier-based multipliers
    tierMultipliers map[string]float64

    // Default limits
    defaultRate  rate.Limit
    defaultBurst int
}

func NewTieredRateLimiter(
    endpoints []EndpointConfig,
    tierMultipliers map[string]float64,
    defaultRate rate.Limit,
    defaultBurst int,
) *TieredRateLimiter {
    trl := &TieredRateLimiter{
        endpoints:       make(map[string]*ClientRateLimiter),
        tierMultipliers: tierMultipliers,
        defaultRate:     defaultRate,
        defaultBurst:    defaultBurst,
    }

    for _, ep := range endpoints {
        trl.endpoints[ep.Pattern] = NewClientRateLimiter(
            ep.Rate, ep.Burst,
            5*time.Minute, 30*time.Minute,
        )
    }

    return trl
}

// Allow checks if a request is permitted given the client, endpoint, and tier.
func (trl *TieredRateLimiter) Allow(clientID, endpoint, tier string) bool {
    // Find matching endpoint limiter
    limiter := trl.findEndpointLimiter(endpoint)
    if limiter == nil {
        return true // no limit configured
    }

    // Apply tier multiplier by adjusting the client's limiter
    if multiplier, ok := trl.tierMultipliers[tier]; ok {
        clientLimiter := limiter.GetLimiter(clientID)
        adjustedRate := rate.Limit(float64(trl.defaultRate) * multiplier)
        adjustedBurst := int(float64(trl.defaultBurst) * multiplier)
        clientLimiter.SetLimit(adjustedRate)
        clientLimiter.SetBurst(adjustedBurst)
    }

    return limiter.Allow(clientID)
}

func (trl *TieredRateLimiter) findEndpointLimiter(endpoint string) *ClientRateLimiter {
    // Exact match first
    if limiter, ok := trl.endpoints[endpoint]; ok {
        return limiter
    }

    // Prefix match
    for pattern, limiter := range trl.endpoints {
        if strings.HasPrefix(endpoint, pattern) {
            return limiter
        }
    }

    return nil
}
```

### Usage with HTTP Middleware

```go
func RateLimitMiddleware(limiter *ClientRateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract client identifier (IP, API key, user ID, etc.)
            clientID := extractClientID(r)

            if !limiter.Allow(clientID) {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)

                // Include standard rate limit headers
                w.Header().Set("Retry-After", "1")
                w.Header().Set("X-RateLimit-Limit", "100")
                w.Header().Set("X-RateLimit-Remaining", "0")
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

func extractClientID(r *http.Request) string {
    // Priority: API key > authenticated user > IP
    if key := r.Header.Get("X-API-Key"); key != "" {
        return "key:" + key
    }
    if user := r.Header.Get("X-User-ID"); user != "" {
        return "user:" + user
    }
    // Use IP (consider X-Forwarded-For behind proxies)
    ip := r.RemoteAddr
    if forwarded := r.Header.Get("X-Forwarded-For"); forwarded != "" {
        ip = strings.Split(forwarded, ",")[0]
    }
    return "ip:" + strings.TrimSpace(ip)
}
```

---

## Distributed Rate Limiting with Redis

In a multi-instance deployment, each instance must share rate limit state. Redis is the most common choice.

### Token Bucket with Redis and Lua

Using a Lua script ensures atomicity without distributed locks.

```go
package ratelimit

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// RedisRateLimiter implements distributed rate limiting using Redis.
type RedisRateLimiter struct {
    client   *redis.Client
    rate     float64 // tokens per second
    capacity int     // max burst size
    prefix   string
    script   *redis.Script
}

func NewRedisRateLimiter(client *redis.Client, ratePerSec float64, capacity int, prefix string) *RedisRateLimiter {
    // Lua script for atomic token bucket operations.
    // KEYS[1] = bucket key
    // ARGV[1] = rate (tokens/sec)
    // ARGV[2] = capacity
    // ARGV[3] = now (unix timestamp with microseconds)
    // ARGV[4] = requested tokens
    //
    // Returns: {allowed (0/1), remaining tokens, retry_after_seconds}
    luaScript := redis.NewScript(`
        local key = KEYS[1]
        local rate = tonumber(ARGV[1])
        local capacity = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])

        -- Get current state
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1])
        local last_refill = tonumber(bucket[2])

        -- Initialize if first request
        if tokens == nil then
            tokens = capacity
            last_refill = now
        end

        -- Calculate token refill
        local elapsed = now - last_refill
        tokens = math.min(capacity, tokens + (elapsed * rate))
        last_refill = now

        local allowed = 0
        local retry_after = 0

        if tokens >= requested then
            tokens = tokens - requested
            allowed = 1
        else
            -- How long until enough tokens are available?
            retry_after = (requested - tokens) / rate
        end

        -- Save state
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', last_refill)
        -- Auto-expire the key after 2x the fill time to prevent stale keys
        local ttl = math.ceil(capacity / rate) * 2
        redis.call('EXPIRE', key, ttl)

        return {allowed, tostring(tokens), tostring(retry_after)}
    `)

    return &RedisRateLimiter{
        client:   client,
        rate:     ratePerSec,
        capacity: capacity,
        prefix:   prefix,
        script:   luaScript,
    }
}

// Allow checks if a request identified by key is within the rate limit.
func (rl *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    return rl.AllowN(ctx, key, 1)
}

// AllowN checks if n tokens are available for the given key.
func (rl *RedisRateLimiter) AllowN(ctx context.Context, key string, n int) (bool, error) {
    now := float64(time.Now().UnixMicro()) / 1_000_000.0
    redisKey := fmt.Sprintf("%s:%s", rl.prefix, key)

    result, err := rl.script.Run(ctx, rl.client, []string{redisKey},
        rl.rate, rl.capacity, now, n,
    ).StringSlice()
    if err != nil {
        return false, fmt.Errorf("redis rate limit check failed: %w", err)
    }

    allowed := result[0] == "1"
    return allowed, nil
}

// RateLimitInfo returns detailed rate limit information.
type RateLimitInfo struct {
    Allowed    bool
    Remaining  float64
    RetryAfter time.Duration
}

func (rl *RedisRateLimiter) Check(ctx context.Context, key string) (*RateLimitInfo, error) {
    now := float64(time.Now().UnixMicro()) / 1_000_000.0
    redisKey := fmt.Sprintf("%s:%s", rl.prefix, key)

    result, err := rl.script.Run(ctx, rl.client, []string{redisKey},
        rl.rate, rl.capacity, now, 1,
    ).StringSlice()
    if err != nil {
        return nil, fmt.Errorf("redis rate limit check failed: %w", err)
    }

    remaining := 0.0
    retryAfter := 0.0
    fmt.Sscanf(result[1], "%f", &remaining)
    fmt.Sscanf(result[2], "%f", &retryAfter)

    return &RateLimitInfo{
        Allowed:    result[0] == "1",
        Remaining:  remaining,
        RetryAfter: time.Duration(retryAfter * float64(time.Second)),
    }, nil
}
```

### Sliding Window Counter with Redis

```go
package ratelimit

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// RedisSlidingWindow implements a sliding window counter using Redis sorted sets.
type RedisSlidingWindow struct {
    client *redis.Client
    limit  int
    window time.Duration
    prefix string
}

func NewRedisSlidingWindow(client *redis.Client, limit int, window time.Duration, prefix string) *RedisSlidingWindow {
    return &RedisSlidingWindow{
        client: client,
        limit:  limit,
        window: window,
        prefix: prefix,
    }
}

func (sw *RedisSlidingWindow) Allow(ctx context.Context, key string) (bool, error) {
    redisKey := fmt.Sprintf("%s:%s", sw.prefix, key)
    now := time.Now()
    windowStart := now.Add(-sw.window)

    // Use a pipeline for atomicity
    pipe := sw.client.Pipeline()

    // Remove entries outside the window
    pipe.ZRemRangeByScore(ctx, redisKey, "0", fmt.Sprintf("%d", windowStart.UnixNano()))

    // Count entries in the window
    countCmd := pipe.ZCard(ctx, redisKey)

    // Execute pipeline
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, fmt.Errorf("redis sliding window check failed: %w", err)
    }

    count := countCmd.Val()
    if count >= int64(sw.limit) {
        return false, nil
    }

    // Add new entry (use unique member to avoid collisions)
    member := fmt.Sprintf("%d:%d", now.UnixNano(), now.UnixNano())
    sw.client.ZAdd(ctx, redisKey, redis.Z{
        Score:  float64(now.UnixNano()),
        Member: member,
    })

    // Set TTL to auto-expire the whole key
    sw.client.Expire(ctx, redisKey, sw.window+time.Second)

    return true, nil
}
```

> **Warning:** Redis-based rate limiting adds network latency (typically 0.5-2ms). For ultra-high-throughput paths, consider a hybrid approach: use an in-memory limiter for the fast path and sync to Redis periodically.

---

## Circuit Breaker Pattern

The circuit breaker prevents an application from repeatedly trying an operation that is likely to fail. Like an electrical circuit breaker, it "trips" when failures accumulate, allowing the failing service to recover.

### State Machine

```
                    success threshold met
        +----------+                    +------------+
        |          | -----------------> |            |
        |  CLOSED  |                    |  HALF-OPEN |
        |          | <----------------- |            |
        +----------+     failure        +------------+
             |                               |
             | failure threshold met         | success
             v                               v
        +----------+     timeout        +----------+
        |          | -----------------> |          |
        |   OPEN   |                    |  CLOSED  |
        |          |                    |          |
        +----------+                    +----------+
```

| State | Behavior | Transitions |
|-------|----------|-------------|
| **Closed** | Requests pass through. Failures are counted. | -> Open: when failure threshold is reached |
| **Open** | Requests are immediately rejected (fail fast). | -> Half-Open: after a timeout period |
| **Half-Open** | A limited number of probe requests are allowed. | -> Closed: if probes succeed; -> Open: if probes fail |

### Why Use a Circuit Breaker?

1. **Fail fast** - Don't waste time waiting for a service that is down
2. **Protect downstream** - Give the failing service time to recover
3. **Prevent cascade failures** - One slow service doesn't bring everything down
4. **Resource conservation** - Don't exhaust connection pools and goroutines on doomed requests

---

## Building a Circuit Breaker from Scratch

```go
package circuitbreaker

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "time"
)

// State represents the circuit breaker state.
type State int

const (
    StateClosed   State = iota // Normal operation
    StateOpen                  // Failing, reject all requests
    StateHalfOpen              // Testing if service recovered
)

func (s State) String() string {
    switch s {
    case StateClosed:
        return "closed"
    case StateOpen:
        return "open"
    case StateHalfOpen:
        return "half-open"
    default:
        return "unknown"
    }
}

// Predefined errors.
var (
    ErrCircuitOpen    = errors.New("circuit breaker is open")
    ErrTooManyRequests = errors.New("too many requests in half-open state")
)

// Counts holds the number of successes and failures.
type Counts struct {
    Requests             int
    TotalSuccesses       int
    TotalFailures        int
    ConsecutiveSuccesses int
    ConsecutiveFailures  int
}

func (c *Counts) onSuccess() {
    c.Requests++
    c.TotalSuccesses++
    c.ConsecutiveSuccesses++
    c.ConsecutiveFailures = 0
}

func (c *Counts) onFailure() {
    c.Requests++
    c.TotalFailures++
    c.ConsecutiveFailures++
    c.ConsecutiveSuccesses = 0
}

func (c *Counts) clear() {
    *c = Counts{}
}

// Settings configures the circuit breaker behavior.
type Settings struct {
    // Name identifies the circuit breaker for logging.
    Name string

    // MaxRequests is the max number of requests allowed in half-open state.
    MaxRequests int

    // Interval is the cyclic period of the closed state for clearing counts.
    // If 0, counts are never cleared in the closed state.
    Interval time.Duration

    // Timeout is how long the circuit stays open before transitioning to half-open.
    Timeout time.Duration

    // ReadyToTrip is called with a copy of Counts when a request fails in the
    // closed state. If it returns true, the circuit transitions to open.
    ReadyToTrip func(counts Counts) bool

    // OnStateChange is called whenever the state changes.
    OnStateChange func(name string, from State, to State)

    // IsSuccessful classifies the error. If nil, all non-nil errors are failures.
    IsSuccessful func(err error) bool
}

// CircuitBreaker implements the circuit breaker pattern.
type CircuitBreaker struct {
    mu          sync.Mutex
    settings    Settings
    state       State
    counts      Counts
    expiry      time.Time // when the current state expires
    generation  uint64    // incremented on state transitions
}

// New creates a new circuit breaker with the given settings.
func New(settings Settings) *CircuitBreaker {
    if settings.MaxRequests == 0 {
        settings.MaxRequests = 1
    }
    if settings.Timeout == 0 {
        settings.Timeout = 60 * time.Second
    }
    if settings.ReadyToTrip == nil {
        settings.ReadyToTrip = func(counts Counts) bool {
            return counts.ConsecutiveFailures > 5
        }
    }
    if settings.IsSuccessful == nil {
        settings.IsSuccessful = func(err error) bool {
            return err == nil
        }
    }

    cb := &CircuitBreaker{
        settings: settings,
        state:    StateClosed,
    }
    cb.toNewGeneration(time.Now())
    return cb
}

// Execute runs the given function if the circuit breaker allows it.
func (cb *CircuitBreaker) Execute(fn func() error) error {
    generation, err := cb.beforeRequest()
    if err != nil {
        return err
    }

    // Run the actual function
    err = fn()

    cb.afterRequest(generation, cb.settings.IsSuccessful(err))
    return err
}

// ExecuteWithContext runs the function with context support.
func (cb *CircuitBreaker) ExecuteWithContext(ctx context.Context, fn func(ctx context.Context) error) error {
    generation, err := cb.beforeRequest()
    if err != nil {
        return err
    }

    // Check context before executing
    if ctx.Err() != nil {
        return ctx.Err()
    }

    err = fn(ctx)

    cb.afterRequest(generation, cb.settings.IsSuccessful(err))
    return err
}

// State returns the current state of the circuit breaker.
func (cb *CircuitBreaker) CurrentState() State {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    now := time.Now()
    state, _ := cb.currentState(now)
    return state
}

// Counts returns the current counts.
func (cb *CircuitBreaker) CurrentCounts() Counts {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    return cb.counts
}

func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    now := time.Now()
    state, generation := cb.currentState(now)

    switch state {
    case StateClosed:
        // Allow all requests
        return generation, nil

    case StateOpen:
        return generation, ErrCircuitOpen

    case StateHalfOpen:
        if cb.counts.Requests >= cb.settings.MaxRequests {
            return generation, ErrTooManyRequests
        }
        return generation, nil
    }

    return generation, nil
}

func (cb *CircuitBreaker) afterRequest(generation uint64, success bool) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    now := time.Now()
    _, currentGeneration := cb.currentState(now)
    if generation != currentGeneration {
        return // State changed while request was in flight; discard result
    }

    if success {
        cb.onSuccess(now)
    } else {
        cb.onFailure(now)
    }
}

func (cb *CircuitBreaker) onSuccess(now time.Time) {
    switch cb.state {
    case StateClosed:
        cb.counts.onSuccess()

    case StateHalfOpen:
        cb.counts.onSuccess()
        if cb.counts.ConsecutiveSuccesses >= cb.settings.MaxRequests {
            cb.setState(StateClosed, now)
        }
    }
}

func (cb *CircuitBreaker) onFailure(now time.Time) {
    switch cb.state {
    case StateClosed:
        cb.counts.onFailure()
        if cb.settings.ReadyToTrip(cb.counts) {
            cb.setState(StateOpen, now)
        }

    case StateHalfOpen:
        cb.setState(StateOpen, now)
    }
}

func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
    switch cb.state {
    case StateClosed:
        if !cb.expiry.IsZero() && cb.expiry.Before(now) {
            cb.toNewGeneration(now)
        }
    case StateOpen:
        if cb.expiry.Before(now) {
            cb.setState(StateHalfOpen, now)
        }
    }
    return cb.state, cb.generation
}

func (cb *CircuitBreaker) setState(state State, now time.Time) {
    if cb.state == state {
        return
    }

    prev := cb.state
    cb.state = state
    cb.toNewGeneration(now)

    if cb.settings.OnStateChange != nil {
        cb.settings.OnStateChange(cb.settings.Name, prev, state)
    }
}

func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
    cb.generation++
    cb.counts.clear()

    switch cb.state {
    case StateClosed:
        if cb.settings.Interval > 0 {
            cb.expiry = now.Add(cb.settings.Interval)
        } else {
            cb.expiry = time.Time{} // never expires
        }
    case StateOpen:
        cb.expiry = now.Add(cb.settings.Timeout)
    case StateHalfOpen:
        cb.expiry = time.Time{} // no expiry in half-open
    }
}
```

### Using the Custom Circuit Breaker

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "math/rand"
    "net/http"
    "time"
)

func main() {
    cb := circuitbreaker.New(circuitbreaker.Settings{
        Name:        "payment-service",
        MaxRequests: 3,                // allow 3 probe requests in half-open
        Timeout:     10 * time.Second, // stay open for 10 seconds
        Interval:    30 * time.Second, // clear counts every 30 seconds in closed state

        ReadyToTrip: func(counts circuitbreaker.Counts) bool {
            // Trip after 3 consecutive failures or >60% failure rate with at least 10 requests
            if counts.ConsecutiveFailures >= 3 {
                return true
            }
            failureRate := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 10 && failureRate >= 0.6
        },

        OnStateChange: func(name string, from, to circuitbreaker.State) {
            fmt.Printf("[CircuitBreaker] %s: %s -> %s\n", name, from, to)
        },
    })

    // Simulate requests
    for i := 0; i < 20; i++ {
        err := cb.Execute(func() error {
            return callPaymentService()
        })

        if err != nil {
            if errors.Is(err, circuitbreaker.ErrCircuitOpen) {
                fmt.Printf("Request %d: circuit is OPEN (fail fast)\n", i+1)
            } else {
                fmt.Printf("Request %d: failed - %v\n", i+1, err)
            }
        } else {
            fmt.Printf("Request %d: success\n", i+1)
        }

        time.Sleep(500 * time.Millisecond)
    }
}

func callPaymentService() error {
    // Simulate a flaky service
    if rand.Float64() < 0.7 {
        return errors.New("connection refused")
    }
    return nil
}
```

---

## Using sony/gobreaker

The `sony/gobreaker` package is a mature, well-tested circuit breaker implementation.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/sony/gobreaker/v2"
)

func main() {
    // Configure the circuit breaker
    cb := gobreaker.NewCircuitBreaker[*http.Response](gobreaker.Settings{
        Name:        "external-api",
        MaxRequests: 3,                 // half-open probe count
        Interval:    60 * time.Second,  // closed state counter reset interval
        Timeout:     30 * time.Second,  // open state duration

        // ReadyToTrip decides when to trip from closed to open
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 5 && failureRatio >= 0.6
        },

        // OnStateChange logs state transitions
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            fmt.Printf("[%s] State changed: %s -> %s\n", name, from, to)
        },

        // IsSuccessful classifies responses - e.g., 5xx are failures, 4xx are not
        IsSuccessful: func(err error) bool {
            return err == nil
        },
    })

    // Execute a request through the circuit breaker
    resp, err := cb.Execute(func() (*http.Response, error) {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        req, err := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/data", nil)
        if err != nil {
            return nil, err
        }

        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            return nil, err
        }

        // Treat 5xx as failures
        if resp.StatusCode >= 500 {
            body, _ := io.ReadAll(resp.Body)
            resp.Body.Close()
            return nil, fmt.Errorf("server error %d: %s", resp.StatusCode, body)
        }

        return resp, nil
    })

    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("Success: %s\n", body)
}
```

### Circuit Breaker Registry

For managing multiple circuit breakers across your application:

```go
package resilience

import (
    "sync"
    "time"

    "github.com/sony/gobreaker/v2"
)

// BreakerRegistry manages circuit breakers for multiple services.
type BreakerRegistry struct {
    mu       sync.RWMutex
    breakers map[string]*gobreaker.CircuitBreaker[[]byte]
    defaults gobreaker.Settings
}

func NewBreakerRegistry(defaults gobreaker.Settings) *BreakerRegistry {
    return &BreakerRegistry{
        breakers: make(map[string]*gobreaker.CircuitBreaker[[]byte]),
        defaults: defaults,
    }
}

// Get returns the circuit breaker for the given service, creating one if needed.
func (r *BreakerRegistry) Get(service string) *gobreaker.CircuitBreaker[[]byte] {
    r.mu.RLock()
    cb, exists := r.breakers[service]
    r.mu.RUnlock()
    if exists {
        return cb
    }

    r.mu.Lock()
    defer r.mu.Unlock()

    // Double-check
    if cb, exists := r.breakers[service]; exists {
        return cb
    }

    settings := r.defaults
    settings.Name = service
    cb = gobreaker.NewCircuitBreaker[[]byte](settings)
    r.breakers[service] = cb
    return cb
}

// States returns a map of service name to circuit breaker state.
func (r *BreakerRegistry) States() map[string]gobreaker.State {
    r.mu.RLock()
    defer r.mu.RUnlock()

    states := make(map[string]gobreaker.State, len(r.breakers))
    for name, cb := range r.breakers {
        states[name] = cb.State()
    }
    return states
}
```

---

## Using hystrix-go

`hystrix-go` is a Go port of Netflix's Hystrix, providing circuit breaker, timeout, and concurrent request limiting.

```go
package main

import (
    "errors"
    "fmt"
    "net"
    "net/http"
    "time"

    "github.com/afex/hystrix-go/hystrix"
)

func main() {
    // Configure a command
    hystrix.ConfigureCommand("get-user-profile", hystrix.CommandConfig{
        Timeout:                2000, // milliseconds
        MaxConcurrentRequests:  100,
        RequestVolumeThreshold: 10,   // minimum requests before circuit can trip
        SleepWindow:            5000, // milliseconds to wait before half-open
        ErrorPercentThreshold:  50,   // trip when 50% of requests fail
    })

    // Synchronous execution with fallback
    result := make(chan string, 1)

    err := hystrix.Do("get-user-profile", func() error {
        // Primary logic
        resp, err := http.Get("https://api.example.com/users/123")
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        if resp.StatusCode != http.StatusOK {
            return fmt.Errorf("unexpected status: %d", resp.StatusCode)
        }

        result <- "user data from API"
        return nil
    }, func(err error) error {
        // Fallback logic - return cached or default data
        fmt.Printf("Fallback triggered due to: %v\n", err)
        result <- "cached user data"
        return nil
    })

    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Result: %s\n", <-result)
    }

    // Asynchronous execution
    output := hystrix.Go("get-user-profile", func() error {
        // Primary logic
        time.Sleep(100 * time.Millisecond)
        return nil
    }, func(err error) error {
        // Fallback
        return nil
    })

    select {
    case err := <-output:
        if err != nil {
            fmt.Printf("Async error: %v\n", err)
        } else {
            fmt.Println("Async success")
        }
    }

    // Start the Hystrix metrics stream server (for dashboards)
    hystrixStreamHandler := hystrix.NewStreamHandler()
    hystrixStreamHandler.Start()
    go http.ListenAndServe(":8081", hystrixStreamHandler)
}
```

> **Tip:** `hystrix-go` is no longer actively maintained. For new projects, prefer `sony/gobreaker` combined with your own timeout/concurrency controls. However, `hystrix-go` remains useful for its built-in metrics dashboard support.

---

## Retry Patterns

Retries handle transient failures -- temporary network issues, brief service outages, or rate limit responses. The key is knowing *what*, *when*, and *how many times* to retry.

### Retry Strategy Types

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Constant** | Wait the same duration between retries | Simple, predictable |
| **Linear** | Increase wait linearly (1s, 2s, 3s...) | Moderate backpressure |
| **Exponential** | Double wait each time (1s, 2s, 4s, 8s...) | Aggressive backpressure |
| **Exponential + Jitter** | Exponential with random variation | Prevents thundering herd |

### Comprehensive Retry Implementation

```go
package retry

import (
    "context"
    "errors"
    "fmt"
    "math"
    "math/rand"
    "time"
)

// RetryableError wraps an error and indicates if it should be retried.
type RetryableError struct {
    Err       error
    Retryable bool
}

func (e *RetryableError) Error() string { return e.Err.Error() }
func (e *RetryableError) Unwrap() error { return e.Err }

// Retryable marks an error as retryable.
func Retryable(err error) error {
    return &RetryableError{Err: err, Retryable: true}
}

// NotRetryable marks an error as non-retryable.
func NotRetryable(err error) error {
    return &RetryableError{Err: err, Retryable: false}
}

// BackoffStrategy computes the delay for a given attempt.
type BackoffStrategy func(attempt int) time.Duration

// ConstantBackoff returns the same delay every time.
func ConstantBackoff(delay time.Duration) BackoffStrategy {
    return func(attempt int) time.Duration {
        return delay
    }
}

// LinearBackoff increases the delay linearly.
func LinearBackoff(initial time.Duration) BackoffStrategy {
    return func(attempt int) time.Duration {
        return initial * time.Duration(attempt+1)
    }
}

// ExponentialBackoff doubles the delay each attempt.
func ExponentialBackoff(initial time.Duration, multiplier float64) BackoffStrategy {
    return func(attempt int) time.Duration {
        return time.Duration(float64(initial) * math.Pow(multiplier, float64(attempt)))
    }
}

// WithMaxDelay caps the maximum delay.
func WithMaxDelay(strategy BackoffStrategy, maxDelay time.Duration) BackoffStrategy {
    return func(attempt int) time.Duration {
        delay := strategy(attempt)
        if delay > maxDelay {
            return maxDelay
        }
        return delay
    }
}

// WithJitter adds randomness to prevent thundering herd.
func WithJitter(strategy BackoffStrategy, jitterFraction float64) BackoffStrategy {
    return func(attempt int) time.Duration {
        delay := strategy(attempt)
        jitter := time.Duration(float64(delay) * jitterFraction * (rand.Float64()*2 - 1))
        result := delay + jitter
        if result < 0 {
            return 0
        }
        return result
    }
}

// WithFullJitter returns a random delay between 0 and the computed delay.
// This is the recommended jitter strategy (see AWS Architecture Blog).
func WithFullJitter(strategy BackoffStrategy) BackoffStrategy {
    return func(attempt int) time.Duration {
        delay := strategy(attempt)
        return time.Duration(rand.Int63n(int64(delay)))
    }
}

// WithDecorrelatedJitter implements decorrelated jitter (sleep = min(cap, rand(base, sleep*3)).
func WithDecorrelatedJitter(base, cap time.Duration) BackoffStrategy {
    prevSleep := base
    return func(attempt int) time.Duration {
        sleep := base + time.Duration(rand.Int63n(int64(prevSleep*3-base)))
        if sleep > cap {
            sleep = cap
        }
        prevSleep = sleep
        return sleep
    }
}

// Config defines retry behavior.
type Config struct {
    MaxAttempts    int             // maximum number of attempts (including first)
    Backoff        BackoffStrategy // delay strategy between retries
    RetryIf        func(error) bool // determines if an error should be retried
    OnRetry        func(attempt int, err error) // called before each retry
}

// DefaultConfig returns a sensible default configuration.
func DefaultConfig() Config {
    return Config{
        MaxAttempts: 3,
        Backoff:     WithJitter(ExponentialBackoff(100*time.Millisecond, 2.0), 0.3),
        RetryIf:     DefaultRetryIf,
        OnRetry:     nil,
    }
}

// DefaultRetryIf checks if an error is retryable.
func DefaultRetryIf(err error) bool {
    // Check for explicit retryable/non-retryable marking
    var retryableErr *RetryableError
    if errors.As(err, &retryableErr) {
        return retryableErr.Retryable
    }

    // By default, retry all errors
    return true
}

// Do executes the function with retries according to the config.
func Do(ctx context.Context, cfg Config, fn func(ctx context.Context) error) error {
    var lastErr error

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        // Check context before each attempt
        if ctx.Err() != nil {
            return fmt.Errorf("context cancelled after %d attempts: %w", attempt, ctx.Err())
        }

        // Execute the function
        err := fn(ctx)
        if err == nil {
            return nil // success
        }

        lastErr = err

        // Check if we should retry
        if !cfg.RetryIf(err) {
            return fmt.Errorf("non-retryable error on attempt %d: %w", attempt+1, err)
        }

        // Don't delay after the last attempt
        if attempt < cfg.MaxAttempts-1 {
            if cfg.OnRetry != nil {
                cfg.OnRetry(attempt+1, err)
            }

            delay := cfg.Backoff(attempt)
            select {
            case <-ctx.Done():
                return fmt.Errorf("context cancelled during backoff: %w", ctx.Err())
            case <-time.After(delay):
                // Continue to next attempt
            }
        }
    }

    return fmt.Errorf("all %d attempts failed, last error: %w", cfg.MaxAttempts, lastErr)
}
```

### Usage Examples

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Example 1: Simple retry with defaults
    err := retry.Do(ctx, retry.DefaultConfig(), func(ctx context.Context) error {
        resp, err := http.Get("https://api.example.com/data")
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        if resp.StatusCode == http.StatusTooManyRequests {
            return retry.Retryable(fmt.Errorf("rate limited"))
        }
        if resp.StatusCode == http.StatusBadRequest {
            return retry.NotRetryable(fmt.Errorf("bad request"))
        }
        if resp.StatusCode >= 500 {
            return retry.Retryable(fmt.Errorf("server error: %d", resp.StatusCode))
        }

        return nil
    })

    // Example 2: Custom retry with exponential backoff and jitter
    cfg := retry.Config{
        MaxAttempts: 5,
        Backoff: retry.WithMaxDelay(
            retry.WithFullJitter(
                retry.ExponentialBackoff(200*time.Millisecond, 2.0),
            ),
            10*time.Second,
        ),
        RetryIf: func(err error) bool {
            // Only retry on specific errors
            var netErr *net.OpError
            if errors.As(err, &netErr) {
                return true
            }
            return retry.DefaultRetryIf(err)
        },
        OnRetry: func(attempt int, err error) {
            fmt.Printf("Retry attempt %d after error: %v\n", attempt, err)
        },
    }

    err = retry.Do(ctx, cfg, func(ctx context.Context) error {
        return callExternalService(ctx)
    })

    if err != nil {
        fmt.Printf("Final error: %v\n", err)
    }
}
```

### Visualizing Backoff Strategies

```
Constant (1s):
  Attempt 1: [====] 1.0s
  Attempt 2: [====] 1.0s
  Attempt 3: [====] 1.0s
  Attempt 4: [====] 1.0s

Exponential (base=500ms, multiplier=2):
  Attempt 1: [==]   0.5s
  Attempt 2: [====] 1.0s
  Attempt 3: [========] 2.0s
  Attempt 4: [================] 4.0s

Exponential + Full Jitter:
  Attempt 1: [=]    0.3s  (random in [0, 0.5s])
  Attempt 2: [===]  0.7s  (random in [0, 1.0s])
  Attempt 3: [=====] 1.4s (random in [0, 2.0s])
  Attempt 4: [==========] 2.8s (random in [0, 4.0s])
```

> **Warning:** Always set a maximum number of retries and a maximum total timeout. Without these limits, retries can consume resources indefinitely and amplify outages.

---

## Idempotency and Safe Retries

Not all operations are safe to retry. An idempotent operation produces the same result whether executed once or multiple times.

### Idempotent vs Non-Idempotent Operations

| Method | Typically Idempotent? | Notes |
|--------|----------------------|-------|
| GET | Yes | Read-only |
| PUT | Yes | Replaces entire resource |
| DELETE | Yes | Deleting twice = same result |
| POST | **No** | May create duplicate resources |
| PATCH | **Sometimes** | Depends on implementation |

### Implementing Idempotency Keys

```go
package idempotency

import (
    "context"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "errors"
    "fmt"
    "sync"
    "time"
)

// Result stores the result of an idempotent operation.
type Result struct {
    StatusCode int
    Body       []byte
    Headers    map[string]string
    CreatedAt  time.Time
    ExpiresAt  time.Time
}

// Store defines the interface for idempotency key storage.
type Store interface {
    // Get returns the stored result for the key, or nil if not found.
    Get(ctx context.Context, key string) (*Result, error)
    // Set stores a result for the key. Returns false if key already exists.
    Set(ctx context.Context, key string, result *Result) (bool, error)
    // Lock acquires a lock on the key to prevent concurrent processing.
    Lock(ctx context.Context, key string) (bool, error)
    // Unlock releases the lock on the key.
    Unlock(ctx context.Context, key string) error
}

// InMemoryStore is a simple in-memory idempotency store (for single-instance use).
type InMemoryStore struct {
    mu      sync.RWMutex
    results map[string]*Result
    locks   map[string]bool
}

func NewInMemoryStore() *InMemoryStore {
    store := &InMemoryStore{
        results: make(map[string]*Result),
        locks:   make(map[string]bool),
    }
    go store.cleanupLoop()
    return store
}

func (s *InMemoryStore) Get(ctx context.Context, key string) (*Result, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    result, ok := s.results[key]
    if !ok {
        return nil, nil
    }
    if time.Now().After(result.ExpiresAt) {
        return nil, nil
    }
    return result, nil
}

func (s *InMemoryStore) Set(ctx context.Context, key string, result *Result) (bool, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.results[key]; exists {
        return false, nil
    }
    s.results[key] = result
    return true, nil
}

func (s *InMemoryStore) Lock(ctx context.Context, key string) (bool, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if s.locks[key] {
        return false, nil
    }
    s.locks[key] = true
    return true, nil
}

func (s *InMemoryStore) Unlock(ctx context.Context, key string) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.locks, key)
    return nil
}

func (s *InMemoryStore) cleanupLoop() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    for range ticker.C {
        s.mu.Lock()
        now := time.Now()
        for key, result := range s.results {
            if now.After(result.ExpiresAt) {
                delete(s.results, key)
            }
        }
        s.mu.Unlock()
    }
}

// Handler wraps an operation with idempotency guarantees.
type Handler struct {
    store Store
    ttl   time.Duration
}

func NewHandler(store Store, ttl time.Duration) *Handler {
    return &Handler{store: store, ttl: ttl}
}

// Execute runs the operation idempotently. If the idempotency key has been
// seen before, it returns the cached result without executing fn.
func (h *Handler) Execute(ctx context.Context, idempotencyKey string, fn func(ctx context.Context) (*Result, error)) (*Result, error) {
    // Check for existing result
    existing, err := h.store.Get(ctx, idempotencyKey)
    if err != nil {
        return nil, fmt.Errorf("failed to check idempotency store: %w", err)
    }
    if existing != nil {
        return existing, nil // Return cached result
    }

    // Acquire lock to prevent concurrent execution
    locked, err := h.store.Lock(ctx, idempotencyKey)
    if err != nil {
        return nil, fmt.Errorf("failed to acquire lock: %w", err)
    }
    if !locked {
        // Another request is processing this key; wait and return its result
        return h.waitForResult(ctx, idempotencyKey)
    }
    defer h.store.Unlock(ctx, idempotencyKey)

    // Double-check after acquiring lock
    existing, err = h.store.Get(ctx, idempotencyKey)
    if err != nil {
        return nil, fmt.Errorf("failed to recheck idempotency store: %w", err)
    }
    if existing != nil {
        return existing, nil
    }

    // Execute the operation
    result, err := fn(ctx)
    if err != nil {
        return nil, err
    }

    // Store the result
    result.CreatedAt = time.Now()
    result.ExpiresAt = time.Now().Add(h.ttl)
    h.store.Set(ctx, idempotencyKey, result)

    return result, nil
}

func (h *Handler) waitForResult(ctx context.Context, key string) (*Result, error) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-ticker.C:
            result, err := h.store.Get(ctx, key)
            if err != nil {
                return nil, err
            }
            if result != nil {
                return result, nil
            }
        }
    }
}

// GenerateKey creates a deterministic idempotency key from request parameters.
func GenerateKey(method, path string, body interface{}) string {
    data, _ := json.Marshal(body)
    hash := sha256.Sum256(append([]byte(method+path), data...))
    return hex.EncodeToString(hash[:])
}
```

### HTTP Middleware for Idempotency

```go
func IdempotencyMiddleware(handler *idempotency.Handler) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Only apply to non-idempotent methods
            if r.Method == http.MethodGet || r.Method == http.MethodHead {
                next.ServeHTTP(w, r)
                return
            }

            key := r.Header.Get("Idempotency-Key")
            if key == "" {
                // No idempotency key - proceed normally
                next.ServeHTTP(w, r)
                return
            }

            result, err := handler.Execute(r.Context(), key, func(ctx context.Context) (*idempotency.Result, error) {
                // Capture the response
                recorder := &responseRecorder{
                    ResponseWriter: w,
                    statusCode:     http.StatusOK,
                }
                next.ServeHTTP(recorder, r)

                return &idempotency.Result{
                    StatusCode: recorder.statusCode,
                    Body:       recorder.body,
                }, nil
            })

            if err != nil {
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
                return
            }

            // Write cached response
            w.WriteHeader(result.StatusCode)
            w.Write(result.Body)
        })
    }
}

type responseRecorder struct {
    http.ResponseWriter
    statusCode int
    body       []byte
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
}

func (r *responseRecorder) Write(b []byte) (int, error) {
    r.body = append(r.body, b...)
    return len(b), nil
}
```

---

## Bulkhead Pattern for Isolation

The bulkhead pattern isolates components so that a failure in one does not cascade to others. Named after ship bulkheads that prevent a single hull breach from sinking the entire vessel.

### Semaphore-Based Bulkhead

```go
package bulkhead

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "time"
)

var (
    ErrBulkheadFull = errors.New("bulkhead is full")
)

// Bulkhead limits concurrent access to a resource.
type Bulkhead struct {
    name       string
    semaphore  chan struct{}
    maxWait    time.Duration
    mu         sync.RWMutex
    active     int64
    rejected   int64
    completed  int64
}

// New creates a bulkhead with the given concurrency limit.
func New(name string, maxConcurrent int, maxWait time.Duration) *Bulkhead {
    return &Bulkhead{
        name:      name,
        semaphore: make(chan struct{}, maxConcurrent),
        maxWait:   maxWait,
    }
}

// Execute runs the function within the bulkhead's concurrency limits.
func (b *Bulkhead) Execute(ctx context.Context, fn func(ctx context.Context) error) error {
    // Try to acquire a slot
    if err := b.acquire(ctx); err != nil {
        return err
    }
    defer b.release()

    return fn(ctx)
}

func (b *Bulkhead) acquire(ctx context.Context) error {
    // Create a timeout context for waiting
    waitCtx := ctx
    if b.maxWait > 0 {
        var cancel context.CancelFunc
        waitCtx, cancel = context.WithTimeout(ctx, b.maxWait)
        defer cancel()
    }

    select {
    case b.semaphore <- struct{}{}:
        b.mu.Lock()
        b.active++
        b.mu.Unlock()
        return nil
    case <-waitCtx.Done():
        b.mu.Lock()
        b.rejected++
        b.mu.Unlock()
        if ctx.Err() != nil {
            return ctx.Err()
        }
        return ErrBulkheadFull
    }
}

func (b *Bulkhead) release() {
    <-b.semaphore
    b.mu.Lock()
    b.active--
    b.completed++
    b.mu.Unlock()
}

// Stats returns the current bulkhead statistics.
type Stats struct {
    Name      string
    Active    int64
    Rejected  int64
    Completed int64
    Available int
    Capacity  int
}

func (b *Bulkhead) Stats() Stats {
    b.mu.RLock()
    defer b.mu.RUnlock()
    return Stats{
        Name:      b.name,
        Active:    b.active,
        Rejected:  b.rejected,
        Completed: b.completed,
        Available: cap(b.semaphore) - len(b.semaphore),
        Capacity:  cap(b.semaphore),
    }
}
```

### Service-Level Bulkhead Manager

```go
package bulkhead

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Manager manages bulkheads for multiple services/resources.
type Manager struct {
    mu        sync.RWMutex
    bulkheads map[string]*Bulkhead
    defaults  Config
}

type Config struct {
    MaxConcurrent int
    MaxWait       time.Duration
}

func NewManager(defaults Config) *Manager {
    return &Manager{
        bulkheads: make(map[string]*Bulkhead),
        defaults:  defaults,
    }
}

// Get returns or creates a bulkhead for the named service.
func (m *Manager) Get(name string) *Bulkhead {
    m.mu.RLock()
    bh, exists := m.bulkheads[name]
    m.mu.RUnlock()
    if exists {
        return bh
    }

    m.mu.Lock()
    defer m.mu.Unlock()
    if bh, exists = m.bulkheads[name]; exists {
        return bh
    }
    bh = New(name, m.defaults.MaxConcurrent, m.defaults.MaxWait)
    m.bulkheads[name] = bh
    return bh
}

// Configure sets custom limits for a specific service.
func (m *Manager) Configure(name string, maxConcurrent int, maxWait time.Duration) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.bulkheads[name] = New(name, maxConcurrent, maxWait)
}

// AllStats returns stats for all managed bulkheads.
func (m *Manager) AllStats() []Stats {
    m.mu.RLock()
    defer m.mu.RUnlock()

    stats := make([]Stats, 0, len(m.bulkheads))
    for _, bh := range m.bulkheads {
        stats = append(stats, bh.Stats())
    }
    return stats
}
```

### Usage Example: Isolating External Service Calls

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func main() {
    // Create separate bulkheads for different services
    manager := bulkhead.NewManager(bulkhead.Config{
        MaxConcurrent: 10,
        MaxWait:       2 * time.Second,
    })

    // Payment service: critical, higher concurrency limit
    manager.Configure("payment-service", 50, 5*time.Second)

    // Notification service: non-critical, lower limit
    manager.Configure("notification-service", 5, 1*time.Second)

    // Simulate concurrent requests
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ctx := context.Background()

            paymentBH := manager.Get("payment-service")
            notifBH := manager.Get("notification-service")

            // Payment call - isolated
            err := paymentBH.Execute(ctx, func(ctx context.Context) error {
                time.Sleep(100 * time.Millisecond) // simulate work
                return nil
            })
            if err != nil {
                fmt.Printf("Request %d: payment bulkhead rejected: %v\n", id, err)
                return
            }

            // Notification call - isolated, won't affect payment
            err = notifBH.Execute(ctx, func(ctx context.Context) error {
                time.Sleep(500 * time.Millisecond) // slow service
                return nil
            })
            if err != nil {
                // Notification failure is acceptable
                fmt.Printf("Request %d: notification skipped: %v\n", id, err)
            }
        }(i)
    }

    wg.Wait()

    // Print stats
    for _, s := range manager.AllStats() {
        fmt.Printf("%s: active=%d, completed=%d, rejected=%d\n",
            s.Name, s.Active, s.Completed, s.Rejected)
    }
}
```

---

## Timeout Patterns and Cascading Failure Prevention

Timeouts are the simplest yet most critical resilience mechanism. Without proper timeouts, a single slow dependency can consume all goroutines and crash your service.

### The Cascading Failure Problem

```
Client -> Service A -> Service B -> Service C (slow/down)
                                      |
                       Service B blocks waiting for C
                       |
          Service A blocks waiting for B
          |
Client times out, retries, makes it worse
```

### Layered Timeout Strategy

```go
package timeout

import (
    "context"
    "fmt"
    "net"
    "net/http"
    "time"
)

// TimeoutConfig defines timeouts at different layers.
type TimeoutConfig struct {
    // Connection-level: how long to establish TCP connection
    ConnectTimeout time.Duration

    // TLS handshake timeout
    TLSTimeout time.Duration

    // Request-level: total time for the entire request
    RequestTimeout time.Duration

    // Response header timeout: time to wait for response headers
    ResponseHeaderTimeout time.Duration

    // Idle connection timeout
    IdleConnTimeout time.Duration
}

// DefaultTimeoutConfig returns production-ready timeout defaults.
func DefaultTimeoutConfig() TimeoutConfig {
    return TimeoutConfig{
        ConnectTimeout:        3 * time.Second,
        TLSTimeout:            3 * time.Second,
        RequestTimeout:        10 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
        IdleConnTimeout:       90 * time.Second,
    }
}

// NewHTTPClient creates an HTTP client with proper timeouts at every layer.
func NewHTTPClient(cfg TimeoutConfig) *http.Client {
    transport := &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   cfg.ConnectTimeout,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   cfg.TLSTimeout,
        ResponseHeaderTimeout:  cfg.ResponseHeaderTimeout,
        IdleConnTimeout:        cfg.IdleConnTimeout,
        MaxIdleConns:           100,
        MaxIdleConnsPerHost:    10,
        MaxConnsPerHost:        100,
    }

    return &http.Client{
        Transport: transport,
        Timeout:   cfg.RequestTimeout, // overall request timeout
    }
}
```

### Context-Based Timeout Propagation

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// ServiceA calls ServiceB with a deadline.
func ServiceA(ctx context.Context) (string, error) {
    // Set the overall deadline for this operation
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // Call Service B with remaining time budget
    result, err := ServiceB(ctx)
    if err != nil {
        return "", fmt.Errorf("serviceA: %w", err)
    }

    return "A:" + result, nil
}

// ServiceB calls ServiceC, shrinking the timeout to leave room for its own processing.
func ServiceB(ctx context.Context) (string, error) {
    // Reserve 500ms for our own processing
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        adjusted := remaining - 500*time.Millisecond
        if adjusted <= 0 {
            return "", context.DeadlineExceeded
        }
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, adjusted)
        defer cancel()
    }

    result, err := ServiceC(ctx)
    if err != nil {
        return "", fmt.Errorf("serviceB: %w", err)
    }

    // Our own processing (within the reserved 500ms)
    time.Sleep(100 * time.Millisecond)
    return "B:" + result, nil
}

// ServiceC is the leaf service.
func ServiceC(ctx context.Context) (string, error) {
    // Simulate work that respects context cancellation
    select {
    case <-time.After(200 * time.Millisecond):
        return "C:data", nil
    case <-ctx.Done():
        return "", fmt.Errorf("serviceC: %w", ctx.Err())
    }
}

func main() {
    ctx := context.Background()
    result, err := ServiceA(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Result: %s\n", result)
}
```

### Timeout with Fallback

```go
package timeout

import (
    "context"
    "time"
)

// WithFallback attempts the primary function, falling back if it times out.
func WithFallback[T any](
    ctx context.Context,
    timeout time.Duration,
    primary func(ctx context.Context) (T, error),
    fallback func(ctx context.Context) (T, error),
) (T, error) {
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    type result struct {
        value T
        err   error
    }

    ch := make(chan result, 1)
    go func() {
        v, err := primary(ctx)
        ch <- result{v, err}
    }()

    select {
    case r := <-ch:
        if r.err == nil {
            return r.value, nil
        }
        // Primary failed - try fallback
        return fallback(context.Background()) // fresh context for fallback
    case <-ctx.Done():
        // Primary timed out - try fallback
        return fallback(context.Background())
    }
}
```

### Deadline Propagation in gRPC

```go
package main

import (
    "context"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
)

// PropagateDeadlineInterceptor ensures deadlines are forwarded to downstream calls.
func PropagateDeadlineInterceptor() grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        // If no deadline set, add a default one
        if _, ok := ctx.Deadline(); !ok {
            var cancel context.CancelFunc
            ctx, cancel = context.WithTimeout(ctx, 10*time.Second)
            defer cancel()
        }

        // gRPC automatically propagates deadlines via the "grpc-timeout" header
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

---

## Health-Based Load Shedding

Load shedding is the practice of rejecting requests *before* the system becomes overloaded. Rather than degrading performance for everyone, it maintains quality of service for accepted requests.

### Adaptive Load Shedder

```go
package loadshed

import (
    "math"
    "net/http"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// LoadShedder monitors system health and rejects requests when overloaded.
type LoadShedder struct {
    mu              sync.RWMutex
    cpuThreshold    float64    // reject when CPU usage exceeds this (0.0-1.0)
    memThreshold    float64    // reject when memory usage exceeds this
    latencyTarget   time.Duration // target p99 latency
    activeRequests  int64
    maxInFlight     int64
    totalRequests   int64
    shedRequests    int64

    // EMA (Exponential Moving Average) of recent latencies
    avgLatency      float64
    latencyAlpha    float64 // smoothing factor

    // Health signals
    healthScore     float64 // 0.0 (unhealthy) to 1.0 (healthy)
}

func NewLoadShedder(cpuThreshold, memThreshold float64, latencyTarget time.Duration, maxInFlight int64) *LoadShedder {
    ls := &LoadShedder{
        cpuThreshold:  cpuThreshold,
        memThreshold:  memThreshold,
        latencyTarget: latencyTarget,
        maxInFlight:   maxInFlight,
        latencyAlpha:  0.1, // smoothing factor for EMA
        healthScore:   1.0,
    }
    go ls.monitorLoop()
    return ls
}

// ShouldShed determines whether to reject the incoming request.
func (ls *LoadShedder) ShouldShed() bool {
    active := atomic.LoadInt64(&ls.activeRequests)

    // Hard limit: never exceed max in-flight
    if active >= ls.maxInFlight {
        atomic.AddInt64(&ls.shedRequests, 1)
        return true
    }

    // Probabilistic shedding based on health score
    ls.mu.RLock()
    health := ls.healthScore
    ls.mu.RUnlock()

    if health < 0.5 {
        // Below 50% health: progressively shed more traffic
        // At 0% health, shed ~90% of traffic
        // At 50% health, shed ~0% of traffic
        shedProbability := (0.5 - health) * 1.8 // maps 0.5->0.0, 0.0->0.9
        if randFloat() < shedProbability {
            atomic.AddInt64(&ls.shedRequests, 1)
            return true
        }
    }

    return false
}

// TrackRequest marks a request as started and returns a function to mark it done.
func (ls *LoadShedder) TrackRequest() func(latency time.Duration) {
    atomic.AddInt64(&ls.activeRequests, 1)
    atomic.AddInt64(&ls.totalRequests, 1)

    return func(latency time.Duration) {
        atomic.AddInt64(&ls.activeRequests, -1)

        // Update latency EMA
        ls.mu.Lock()
        ls.avgLatency = ls.latencyAlpha*float64(latency) + (1-ls.latencyAlpha)*ls.avgLatency
        ls.mu.Unlock()
    }
}

func (ls *LoadShedder) monitorLoop() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for range ticker.C {
        ls.updateHealthScore()
    }
}

func (ls *LoadShedder) updateHealthScore() {
    ls.mu.Lock()
    defer ls.mu.Unlock()

    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)

    // Factor 1: Memory pressure
    memUsage := float64(memStats.Alloc) / float64(memStats.Sys)
    memHealth := 1.0 - math.Min(1.0, memUsage/ls.memThreshold)

    // Factor 2: Latency health
    latencyHealth := 1.0
    if ls.avgLatency > 0 {
        ratio := ls.avgLatency / float64(ls.latencyTarget)
        if ratio > 1.0 {
            latencyHealth = 1.0 / ratio
        }
    }

    // Factor 3: Concurrency pressure
    active := float64(atomic.LoadInt64(&ls.activeRequests))
    maxInFlight := float64(ls.maxInFlight)
    concurrencyHealth := 1.0 - math.Min(1.0, active/maxInFlight)

    // Combined health score (weighted average)
    ls.healthScore = memHealth*0.3 + latencyHealth*0.4 + concurrencyHealth*0.3
}

// Stats returns current load shedder statistics.
type LoadShedStats struct {
    ActiveRequests int64
    TotalRequests  int64
    ShedRequests   int64
    HealthScore    float64
    AvgLatency     time.Duration
}

func (ls *LoadShedder) Stats() LoadShedStats {
    ls.mu.RLock()
    defer ls.mu.RUnlock()
    return LoadShedStats{
        ActiveRequests: atomic.LoadInt64(&ls.activeRequests),
        TotalRequests:  atomic.LoadInt64(&ls.totalRequests),
        ShedRequests:   atomic.LoadInt64(&ls.shedRequests),
        HealthScore:    ls.healthScore,
        AvgLatency:     time.Duration(ls.avgLatency),
    }
}

// Middleware integration
func (ls *LoadShedder) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if ls.ShouldShed() {
            w.Header().Set("Retry-After", "5")
            http.Error(w, "Service Overloaded", http.StatusServiceUnavailable)
            return
        }

        start := time.Now()
        done := ls.TrackRequest()
        defer func() {
            done(time.Since(start))
        }()

        next.ServeHTTP(w, r)
    })
}

// Helper: simple random float (not cryptographically secure)
func randFloat() float64 {
    // Use a fast, non-crypto random source
    return float64(time.Now().UnixNano()%1000) / 1000.0
}
```

### Priority-Based Load Shedding

```go
package loadshed

import (
    "net/http"
)

// Priority levels for requests.
type Priority int

const (
    PriorityCritical Priority = iota // Never shed (health checks, auth)
    PriorityHigh                     // Shed last (payments, core business)
    PriorityNormal                   // Shed early (non-essential features)
    PriorityLow                      // Shed first (analytics, recommendations)
)

// PriorityShedder sheds low-priority traffic first.
type PriorityShedder struct {
    base       *LoadShedder
    thresholds map[Priority]float64 // health score below which to shed
}

func NewPriorityShedder(base *LoadShedder) *PriorityShedder {
    return &PriorityShedder{
        base: base,
        thresholds: map[Priority]float64{
            PriorityCritical: 0.0,  // never shed
            PriorityHigh:     0.2,  // shed below 20% health
            PriorityNormal:   0.5,  // shed below 50% health
            PriorityLow:      0.8,  // shed below 80% health
        },
    }
}

func (ps *PriorityShedder) ShouldShed(priority Priority) bool {
    stats := ps.base.Stats()

    threshold, ok := ps.thresholds[priority]
    if !ok {
        threshold = 0.5
    }

    return stats.HealthScore < threshold
}

// Middleware extracts priority from request headers or path and applies shedding.
func (ps *PriorityShedder) Middleware(getPriority func(*http.Request) Priority) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            priority := getPriority(r)

            if ps.ShouldShed(priority) {
                w.Header().Set("Retry-After", "5")
                http.Error(w, "Service Overloaded", http.StatusServiceUnavailable)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

---

## Combining Patterns: Retry + Circuit Breaker + Rate Limit

The true power of resilience patterns emerges when they are composed. The correct ordering matters.

### Layering Order

```
Request -> Rate Limiter -> Circuit Breaker -> Retry -> Timeout -> Actual Call
              |                 |                |         |
              |                 |                |         +-- Per-attempt timeout
              |                 |                +-- Retry with backoff
              |                 +-- Fail fast if service is down
              +-- Don't overwhelm the system
```

### Resilient Client

```go
package resilience

import (
    "context"
    "errors"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/sony/gobreaker/v2"
    "golang.org/x/time/rate"
)

// Client wraps an HTTP client with rate limiting, circuit breaking, and retries.
type Client struct {
    httpClient     *http.Client
    rateLimiter    *rate.Limiter
    circuitBreaker *gobreaker.CircuitBreaker[*http.Response]
    retryConfig    RetryConfig
}

type RetryConfig struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
}

type ClientConfig struct {
    // HTTP
    RequestTimeout time.Duration
    ConnTimeout    time.Duration

    // Rate limiting
    RateLimit rate.Limit
    BurstSize int

    // Circuit breaker
    CBMaxRequests   uint32
    CBInterval      time.Duration
    CBTimeout       time.Duration
    CBFailureRatio  float64

    // Retry
    RetryMaxAttempts int
    RetryBaseDelay   time.Duration
    RetryMaxDelay    time.Duration
}

func DefaultClientConfig() ClientConfig {
    return ClientConfig{
        RequestTimeout:   10 * time.Second,
        ConnTimeout:      3 * time.Second,
        RateLimit:        rate.Limit(100),   // 100 requests/sec
        BurstSize:        20,
        CBMaxRequests:    3,
        CBInterval:       60 * time.Second,
        CBTimeout:        30 * time.Second,
        CBFailureRatio:   0.6,
        RetryMaxAttempts: 3,
        RetryBaseDelay:   200 * time.Millisecond,
        RetryMaxDelay:    5 * time.Second,
    }
}

func NewClient(cfg ClientConfig) *Client {
    // HTTP client with timeouts
    httpClient := &http.Client{
        Timeout: cfg.RequestTimeout,
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            IdleConnTimeout:     90 * time.Second,
        },
    }

    // Rate limiter
    limiter := rate.NewLimiter(cfg.RateLimit, cfg.BurstSize)

    // Circuit breaker
    cb := gobreaker.NewCircuitBreaker[*http.Response](gobreaker.Settings{
        Name:        "resilient-client",
        MaxRequests: cfg.CBMaxRequests,
        Interval:    cfg.CBInterval,
        Timeout:     cfg.CBTimeout,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            if counts.Requests < 5 {
                return false
            }
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return failureRatio >= cfg.CBFailureRatio
        },
    })

    return &Client{
        httpClient:     httpClient,
        rateLimiter:    limiter,
        circuitBreaker: cb,
        retryConfig: RetryConfig{
            MaxAttempts: cfg.RetryMaxAttempts,
            BaseDelay:   cfg.RetryBaseDelay,
            MaxDelay:    cfg.RetryMaxDelay,
        },
    }
}

// Do executes an HTTP request with all resilience patterns applied.
func (c *Client) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt < c.retryConfig.MaxAttempts; attempt++ {
        // Step 1: Rate limiting
        if err := c.rateLimiter.Wait(ctx); err != nil {
            return nil, fmt.Errorf("rate limiter: %w", err)
        }

        // Step 2: Circuit breaker
        resp, err := c.circuitBreaker.Execute(func() (*http.Response, error) {
            // Step 3: Execute with per-attempt timeout
            attemptCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
            defer cancel()

            reqCopy := req.Clone(attemptCtx)
            resp, err := c.httpClient.Do(reqCopy)
            if err != nil {
                return nil, err
            }

            // Classify response
            if resp.StatusCode >= 500 {
                body, _ := io.ReadAll(resp.Body)
                resp.Body.Close()
                return nil, fmt.Errorf("server error %d: %s", resp.StatusCode, string(body))
            }

            return resp, nil
        })

        if err == nil {
            return resp, nil
        }

        lastErr = err

        // Don't retry certain errors
        if !c.isRetryable(err) {
            return nil, err
        }

        // Don't wait after the last attempt
        if attempt < c.retryConfig.MaxAttempts-1 {
            delay := c.calculateBackoff(attempt)
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-time.After(delay):
            }
        }
    }

    return nil, fmt.Errorf("all %d attempts exhausted: %w", c.retryConfig.MaxAttempts, lastErr)
}

func (c *Client) isRetryable(err error) bool {
    // Circuit breaker open - don't retry, it will just fail fast again
    if errors.Is(err, gobreaker.ErrOpenState) {
        return false
    }
    if errors.Is(err, gobreaker.ErrTooManyRequests) {
        return true // half-open state, worth retrying later
    }
    // Context cancelled - don't retry
    if errors.Is(err, context.Canceled) {
        return false
    }
    return true
}

func (c *Client) calculateBackoff(attempt int) time.Duration {
    delay := c.retryConfig.BaseDelay * time.Duration(1<<uint(attempt))
    if delay > c.retryConfig.MaxDelay {
        delay = c.retryConfig.MaxDelay
    }
    // Add jitter: +/- 25%
    jitter := time.Duration(float64(delay) * 0.25 * (2*randFloat() - 1))
    return delay + jitter
}
```

### Using the Resilient Client

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func main() {
    client := resilience.NewClient(resilience.DefaultClientConfig())

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    req, err := http.NewRequest("GET", "https://api.example.com/orders/123", nil)
    if err != nil {
        panic(err)
    }

    resp, err := client.Do(ctx, req)
    if err != nil {
        fmt.Printf("Request failed after all retries: %v\n", err)
        return
    }
    defer resp.Body.Close()

    fmt.Printf("Success: status %d\n", resp.StatusCode)
}
```

---

## Middleware Implementations for HTTP Servers

### Complete Resilience Middleware Stack

```go
package middleware

import (
    "context"
    "fmt"
    "log/slog"
    "net/http"
    "runtime"
    "strconv"
    "strings"
    "sync/atomic"
    "time"

    "golang.org/x/time/rate"
)

// ---- Rate Limit Middleware ----

// RateLimitConfig configures the rate limiter middleware.
type RateLimitConfig struct {
    // Global rate limit
    GlobalRate  rate.Limit
    GlobalBurst int

    // Per-client rate limit
    ClientRate  rate.Limit
    ClientBurst int

    // How to identify clients
    KeyFunc     func(r *http.Request) string

    // Custom response when rate limited
    DeniedHandler http.Handler
}

func DefaultRateLimitConfig() RateLimitConfig {
    return RateLimitConfig{
        GlobalRate:  1000,
        GlobalBurst: 200,
        ClientRate:  50,
        ClientBurst: 10,
        KeyFunc: func(r *http.Request) string {
            return strings.Split(r.RemoteAddr, ":")[0]
        },
        DeniedHandler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Content-Type", "application/json")
            w.Header().Set("Retry-After", "1")
            w.WriteHeader(http.StatusTooManyRequests)
            fmt.Fprintf(w, `{"error":"rate limit exceeded","retry_after":1}`)
        }),
    }
}

func RateLimit(cfg RateLimitConfig) func(http.Handler) http.Handler {
    globalLimiter := rate.NewLimiter(cfg.GlobalRate, cfg.GlobalBurst)
    clientLimiters := NewClientRateLimiter(cfg.ClientRate, cfg.ClientBurst, time.Minute, 10*time.Minute)

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Check global rate limit
            if !globalLimiter.Allow() {
                slog.Warn("global rate limit exceeded")
                cfg.DeniedHandler.ServeHTTP(w, r)
                return
            }

            // Check per-client rate limit
            clientKey := cfg.KeyFunc(r)
            clientLimiter := clientLimiters.GetLimiter(clientKey)

            if !clientLimiter.Allow() {
                slog.Warn("client rate limit exceeded", "client", clientKey)

                // Add rate limit headers
                w.Header().Set("X-RateLimit-Limit", strconv.Itoa(cfg.ClientBurst))
                w.Header().Set("X-RateLimit-Remaining", "0")
                w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(time.Now().Add(time.Second).Unix(), 10))

                cfg.DeniedHandler.ServeHTTP(w, r)
                return
            }

            // Add rate limit headers for successful requests
            w.Header().Set("X-RateLimit-Limit", strconv.Itoa(cfg.ClientBurst))
            tokens := int(clientLimiter.Tokens())
            if tokens < 0 {
                tokens = 0
            }
            w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(tokens))

            next.ServeHTTP(w, r)
        })
    }
}

// ---- Timeout Middleware ----

func Timeout(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()

            r = r.WithContext(ctx)

            done := make(chan struct{})
            tw := &timeoutWriter{ResponseWriter: w}

            go func() {
                next.ServeHTTP(tw, r)
                close(done)
            }()

            select {
            case <-done:
                // Handler completed normally
            case <-ctx.Done():
                tw.mu.Lock()
                defer tw.mu.Unlock()
                if !tw.written {
                    w.WriteHeader(http.StatusGatewayTimeout)
                    fmt.Fprintf(w, `{"error":"request timeout"}`)
                    tw.written = true
                    tw.timedOut = true
                }
            }
        })
    }
}

type timeoutWriter struct {
    http.ResponseWriter
    mu       sync.Mutex
    written  bool
    timedOut bool
}

func (tw *timeoutWriter) WriteHeader(code int) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.timedOut {
        return
    }
    tw.written = true
    tw.ResponseWriter.WriteHeader(code)
}

func (tw *timeoutWriter) Write(b []byte) (int, error) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.timedOut {
        return 0, context.DeadlineExceeded
    }
    tw.written = true
    return tw.ResponseWriter.Write(b)
}

// ---- Circuit Breaker Middleware ----

func CircuitBreaker(cb *gobreaker.CircuitBreaker[struct{}]) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            _, err := cb.Execute(func() (struct{}, error) {
                recorder := &statusRecorder{ResponseWriter: w, statusCode: 200}
                next.ServeHTTP(recorder, r)

                if recorder.statusCode >= 500 {
                    return struct{}{}, fmt.Errorf("server error: %d", recorder.statusCode)
                }
                return struct{}{}, nil
            })

            if err != nil {
                if errors.Is(err, gobreaker.ErrOpenState) || errors.Is(err, gobreaker.ErrTooManyRequests) {
                    w.Header().Set("Retry-After", "30")
                    http.Error(w, `{"error":"service temporarily unavailable"}`, http.StatusServiceUnavailable)
                }
                // Other errors were already written by the handler
            }
        })
    }
}

type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *statusRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}

// ---- Max In-Flight Middleware ----

func MaxInFlight(limit int64) func(http.Handler) http.Handler {
    var current int64

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            active := atomic.AddInt64(&current, 1)
            defer atomic.AddInt64(&current, -1)

            if active > limit {
                w.Header().Set("Retry-After", "5")
                http.Error(w, `{"error":"too many concurrent requests"}`, http.StatusServiceUnavailable)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// ---- Recovery Middleware ----

func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                buf := make([]byte, 4096)
                n := runtime.Stack(buf, false)

                slog.Error("panic recovered",
                    "error", rec,
                    "stack", string(buf[:n]),
                    "path", r.URL.Path,
                    "method", r.Method,
                )

                http.Error(w, `{"error":"internal server error"}`, http.StatusInternalServerError)
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

### Assembling the Middleware Stack

```go
package main

import (
    "log"
    "net/http"
    "time"

    "golang.org/x/time/rate"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/api/orders", handleOrders)
    mux.HandleFunc("/api/users", handleUsers)
    mux.HandleFunc("/health", handleHealth)

    // Build middleware stack (applied bottom-to-top)
    var handler http.Handler = mux

    // 1. Recovery (outermost - catches panics from everything)
    handler = middleware.Recovery(handler)

    // 2. Timeout (prevent requests from running forever)
    handler = middleware.Timeout(15 * time.Second)(handler)

    // 3. Rate limiting (reject excessive traffic)
    handler = middleware.RateLimit(middleware.RateLimitConfig{
        GlobalRate:  500,
        GlobalBurst: 100,
        ClientRate:  20,
        ClientBurst: 5,
        KeyFunc: func(r *http.Request) string {
            // Use API key if present, otherwise IP
            if key := r.Header.Get("X-API-Key"); key != "" {
                return "key:" + key
            }
            return "ip:" + r.RemoteAddr
        },
        DeniedHandler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusTooManyRequests)
            w.Write([]byte(`{"error":"rate limit exceeded"}`))
        }),
    })(handler)

    // 4. Max in-flight (prevent overload)
    handler = middleware.MaxInFlight(1000)(handler)

    // 5. Load shedding (health-based rejection)
    shedder := loadshed.NewLoadShedder(0.8, 0.9, 500*time.Millisecond, 2000)
    handler = shedder.Middleware(handler)

    server := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      20 * time.Second,
        IdleTimeout:       120 * time.Second,
        MaxHeaderBytes:    1 << 20, // 1 MB
    }

    log.Printf("Server starting on :8080")
    log.Fatal(server.ListenAndServe())
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte(`{"orders":[]}`))
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte(`{"users":[]}`))
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"healthy"}`))
}
```

---

## Testing Resilience Patterns

Testing resilience requires simulating failures, latency, and load. This section covers strategies and tools.

### Testing Token Bucket Rate Limiter

```go
package ratelimit_test

import (
    "sync"
    "testing"
    "time"
)

func TestTokenBucket_BasicAllowDeny(t *testing.T) {
    // 10 tokens/sec, burst of 5
    tb := NewTokenBucket(10, 5)

    // Should allow first 5 (burst)
    for i := 0; i < 5; i++ {
        if !tb.Allow() {
            t.Fatalf("expected request %d to be allowed", i+1)
        }
    }

    // 6th should be denied (burst exhausted)
    if tb.Allow() {
        t.Fatal("expected request 6 to be denied")
    }
}

func TestTokenBucket_RefillsOverTime(t *testing.T) {
    tb := NewTokenBucket(10, 5)

    // Exhaust all tokens
    for i := 0; i < 5; i++ {
        tb.Allow()
    }

    // Wait for 1 token to refill (100ms at 10 tokens/sec)
    time.Sleep(150 * time.Millisecond)

    if !tb.Allow() {
        t.Fatal("expected request to be allowed after refill")
    }
}

func TestTokenBucket_ConcurrentAccess(t *testing.T) {
    tb := NewTokenBucket(100, 50)

    var allowed int64
    var denied int64
    var wg sync.WaitGroup

    for i := 0; i < 200; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            if tb.Allow() {
                atomic.AddInt64(&allowed, 1)
            } else {
                atomic.AddInt64(&denied, 1)
            }
        }()
    }

    wg.Wait()

    // At most 50 should be allowed (burst size)
    if allowed > 50 {
        t.Fatalf("expected at most 50 allowed, got %d", allowed)
    }
    if allowed+denied != 200 {
        t.Fatalf("expected 200 total, got %d", allowed+denied)
    }
}

func TestTokenBucket_RateOverTime(t *testing.T) {
    tokensPerSec := 10.0
    tb := NewTokenBucket(tokensPerSec, 1) // burst 1 to measure steady rate

    // Measure how many requests are allowed over 1 second
    start := time.Now()
    allowed := 0

    for time.Since(start) < time.Second {
        if tb.Allow() {
            allowed++
        }
        time.Sleep(10 * time.Millisecond)
    }

    // Should be approximately 10 (+/- some tolerance for timing)
    if allowed < 8 || allowed > 12 {
        t.Fatalf("expected ~10 requests/sec, got %d", allowed)
    }
}
```

### Testing Circuit Breaker

```go
package circuitbreaker_test

import (
    "errors"
    "testing"
    "time"
)

var errService = errors.New("service unavailable")

func TestCircuitBreaker_ClosedState(t *testing.T) {
    cb := New(Settings{
        Name:        "test",
        MaxRequests: 1,
        Timeout:     5 * time.Second,
        ReadyToTrip: func(counts Counts) bool {
            return counts.ConsecutiveFailures >= 3
        },
    })

    // Should start closed
    if cb.CurrentState() != StateClosed {
        t.Fatalf("expected closed, got %s", cb.CurrentState())
    }

    // Successful requests should keep it closed
    for i := 0; i < 10; i++ {
        err := cb.Execute(func() error { return nil })
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
    }

    if cb.CurrentState() != StateClosed {
        t.Fatalf("expected closed after successes, got %s", cb.CurrentState())
    }
}

func TestCircuitBreaker_TripsAfterFailures(t *testing.T) {
    cb := New(Settings{
        Name:        "test",
        MaxRequests: 1,
        Timeout:     1 * time.Second,
        ReadyToTrip: func(counts Counts) bool {
            return counts.ConsecutiveFailures >= 3
        },
    })

    // 3 consecutive failures should trip the breaker
    for i := 0; i < 3; i++ {
        cb.Execute(func() error { return errService })
    }

    if cb.CurrentState() != StateOpen {
        t.Fatalf("expected open after 3 failures, got %s", cb.CurrentState())
    }

    // Requests should now fail fast
    err := cb.Execute(func() error { return nil })
    if !errors.Is(err, ErrCircuitOpen) {
        t.Fatalf("expected ErrCircuitOpen, got %v", err)
    }
}

func TestCircuitBreaker_HalfOpenRecovery(t *testing.T) {
    cb := New(Settings{
        Name:        "test",
        MaxRequests: 2,
        Timeout:     100 * time.Millisecond, // short timeout for testing
        ReadyToTrip: func(counts Counts) bool {
            return counts.ConsecutiveFailures >= 2
        },
    })

    // Trip the breaker
    cb.Execute(func() error { return errService })
    cb.Execute(func() error { return errService })

    if cb.CurrentState() != StateOpen {
        t.Fatalf("expected open, got %s", cb.CurrentState())
    }

    // Wait for timeout to transition to half-open
    time.Sleep(150 * time.Millisecond)

    if cb.CurrentState() != StateHalfOpen {
        t.Fatalf("expected half-open, got %s", cb.CurrentState())
    }

    // Successful probe should close the breaker
    err := cb.Execute(func() error { return nil })
    if err != nil {
        t.Fatalf("unexpected error in half-open: %v", err)
    }

    err = cb.Execute(func() error { return nil })
    if err != nil {
        t.Fatalf("unexpected error in half-open: %v", err)
    }

    if cb.CurrentState() != StateClosed {
        t.Fatalf("expected closed after successful probes, got %s", cb.CurrentState())
    }
}

func TestCircuitBreaker_HalfOpenFailure(t *testing.T) {
    cb := New(Settings{
        Name:        "test",
        MaxRequests: 2,
        Timeout:     100 * time.Millisecond,
        ReadyToTrip: func(counts Counts) bool {
            return counts.ConsecutiveFailures >= 2
        },
    })

    // Trip the breaker
    cb.Execute(func() error { return errService })
    cb.Execute(func() error { return errService })

    // Wait for half-open
    time.Sleep(150 * time.Millisecond)

    // Failure in half-open should re-open
    cb.Execute(func() error { return errService })

    if cb.CurrentState() != StateOpen {
        t.Fatalf("expected open after half-open failure, got %s", cb.CurrentState())
    }
}
```

### Testing Retry Logic

```go
package retry_test

import (
    "context"
    "errors"
    "testing"
    "time"
)

func TestRetry_SucceedsOnFirstAttempt(t *testing.T) {
    attempts := 0

    err := Do(context.Background(), DefaultConfig(), func(ctx context.Context) error {
        attempts++
        return nil
    })

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if attempts != 1 {
        t.Fatalf("expected 1 attempt, got %d", attempts)
    }
}

func TestRetry_SucceedsAfterTransientFailure(t *testing.T) {
    attempts := 0
    failUntil := 2

    cfg := Config{
        MaxAttempts: 5,
        Backoff:     ConstantBackoff(10 * time.Millisecond), // fast for testing
        RetryIf:     DefaultRetryIf,
    }

    err := Do(context.Background(), cfg, func(ctx context.Context) error {
        attempts++
        if attempts <= failUntil {
            return Retryable(errors.New("transient error"))
        }
        return nil
    })

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if attempts != 3 {
        t.Fatalf("expected 3 attempts, got %d", attempts)
    }
}

func TestRetry_StopsOnNonRetryableError(t *testing.T) {
    attempts := 0

    cfg := Config{
        MaxAttempts: 5,
        Backoff:     ConstantBackoff(10 * time.Millisecond),
        RetryIf:     DefaultRetryIf,
    }

    err := Do(context.Background(), cfg, func(ctx context.Context) error {
        attempts++
        return NotRetryable(errors.New("permanent error"))
    })

    if err == nil {
        t.Fatal("expected error")
    }
    if attempts != 1 {
        t.Fatalf("expected 1 attempt for non-retryable error, got %d", attempts)
    }
}

func TestRetry_RespectsContextCancellation(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    cfg := Config{
        MaxAttempts: 100,
        Backoff:     ConstantBackoff(100 * time.Millisecond), // longer than context timeout
        RetryIf:     DefaultRetryIf,
    }

    start := time.Now()
    err := Do(ctx, cfg, func(ctx context.Context) error {
        return errors.New("always fails")
    })

    elapsed := time.Since(start)
    if err == nil {
        t.Fatal("expected error")
    }
    if elapsed > 200*time.Millisecond {
        t.Fatalf("retry did not respect context timeout, took %v", elapsed)
    }
}

func TestRetry_ExhaustsAllAttempts(t *testing.T) {
    attempts := 0
    maxAttempts := 4

    cfg := Config{
        MaxAttempts: maxAttempts,
        Backoff:     ConstantBackoff(1 * time.Millisecond),
        RetryIf:     DefaultRetryIf,
    }

    err := Do(context.Background(), cfg, func(ctx context.Context) error {
        attempts++
        return errors.New("always fails")
    })

    if err == nil {
        t.Fatal("expected error after exhausting attempts")
    }
    if attempts != maxAttempts {
        t.Fatalf("expected %d attempts, got %d", maxAttempts, attempts)
    }
}

func TestBackoff_Exponential(t *testing.T) {
    strategy := ExponentialBackoff(100*time.Millisecond, 2.0)

    expected := []time.Duration{
        100 * time.Millisecond,  // attempt 0
        200 * time.Millisecond,  // attempt 1
        400 * time.Millisecond,  // attempt 2
        800 * time.Millisecond,  // attempt 3
    }

    for i, want := range expected {
        got := strategy(i)
        if got != want {
            t.Errorf("attempt %d: expected %v, got %v", i, want, got)
        }
    }
}

func TestBackoff_WithMaxDelay(t *testing.T) {
    strategy := WithMaxDelay(
        ExponentialBackoff(100*time.Millisecond, 2.0),
        500*time.Millisecond,
    )

    got := strategy(10) // Would be 100ms * 2^10 = 102400ms
    if got != 500*time.Millisecond {
        t.Errorf("expected max delay 500ms, got %v", got)
    }
}
```

### Testing Bulkhead

```go
package bulkhead_test

import (
    "context"
    "sync"
    "sync/atomic"
    "testing"
    "time"
)

func TestBulkhead_LimitsConcurrency(t *testing.T) {
    bh := New("test", 5, time.Second)

    var maxConcurrent int64
    var current int64
    var wg sync.WaitGroup

    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            err := bh.Execute(context.Background(), func(ctx context.Context) error {
                c := atomic.AddInt64(&current, 1)
                defer atomic.AddInt64(&current, -1)

                // Track maximum observed concurrency
                for {
                    old := atomic.LoadInt64(&maxConcurrent)
                    if c <= old || atomic.CompareAndSwapInt64(&maxConcurrent, old, c) {
                        break
                    }
                }

                time.Sleep(50 * time.Millisecond)
                return nil
            })

            if err != nil && err != ErrBulkheadFull {
                t.Errorf("unexpected error: %v", err)
            }
        }()
    }

    wg.Wait()

    if maxConcurrent > 5 {
        t.Fatalf("expected max concurrency of 5, observed %d", maxConcurrent)
    }
}

func TestBulkhead_RejectsWhenFull(t *testing.T) {
    bh := New("test", 2, 10*time.Millisecond)

    var rejected int64
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            err := bh.Execute(context.Background(), func(ctx context.Context) error {
                time.Sleep(100 * time.Millisecond)
                return nil
            })

            if err == ErrBulkheadFull {
                atomic.AddInt64(&rejected, 1)
            }
        }()
    }

    wg.Wait()

    if rejected == 0 {
        t.Fatal("expected some requests to be rejected")
    }
}
```

### Integration Test: Full Resilience Stack

```go
package integration_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "sync"
    "sync/atomic"
    "testing"
    "time"
)

func TestFullResilienceStack(t *testing.T) {
    // Create a flaky upstream server
    var upstreamCalls int64
    upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        call := atomic.AddInt64(&upstreamCalls, 1)

        // First 5 calls fail, then recover
        if call <= 5 {
            w.WriteHeader(http.StatusInternalServerError)
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"ok"}`))
    }))
    defer upstream.Close()

    // Create resilient client
    client := resilience.NewClient(resilience.ClientConfig{
        RequestTimeout:   5 * time.Second,
        ConnTimeout:      1 * time.Second,
        RateLimit:        100,
        BurstSize:        20,
        CBMaxRequests:    2,
        CBInterval:       30 * time.Second,
        CBTimeout:        500 * time.Millisecond, // short for testing
        CBFailureRatio:   0.5,
        RetryMaxAttempts: 3,
        RetryBaseDelay:   50 * time.Millisecond,
        RetryMaxDelay:    500 * time.Millisecond,
    })

    ctx := context.Background()

    // Phase 1: Requests during failure period
    var failedRequests int
    for i := 0; i < 5; i++ {
        req, _ := http.NewRequest("GET", upstream.URL+"/api/data", nil)
        _, err := client.Do(ctx, req)
        if err != nil {
            failedRequests++
        }
    }

    t.Logf("Phase 1 - Failed requests: %d/5", failedRequests)

    // Phase 2: Wait for circuit breaker timeout, then retry
    time.Sleep(600 * time.Millisecond) // wait for half-open

    req, _ := http.NewRequest("GET", upstream.URL+"/api/data", nil)
    resp, err := client.Do(ctx, req)
    if err != nil {
        t.Logf("Phase 2 - Still failing (expected if CB needs more probes): %v", err)
    } else {
        t.Logf("Phase 2 - Recovered! Status: %d", resp.StatusCode)
        resp.Body.Close()
    }
}

func TestRateLimitMiddleware_Integration(t *testing.T) {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    cfg := middleware.RateLimitConfig{
        GlobalRate:  10,
        GlobalBurst: 5,
        ClientRate:  5,
        ClientBurst: 2,
        KeyFunc: func(r *http.Request) string {
            return r.RemoteAddr
        },
        DeniedHandler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.WriteHeader(http.StatusTooManyRequests)
        }),
    }

    server := httptest.NewServer(middleware.RateLimit(cfg)(handler))
    defer server.Close()

    // Send burst of requests
    var allowed, denied int64
    var wg sync.WaitGroup

    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            resp, err := http.Get(server.URL + "/api/test")
            if err != nil {
                return
            }
            defer resp.Body.Close()

            if resp.StatusCode == http.StatusOK {
                atomic.AddInt64(&allowed, 1)
            } else if resp.StatusCode == http.StatusTooManyRequests {
                atomic.AddInt64(&denied, 1)
            }
        }()
    }

    wg.Wait()

    t.Logf("Allowed: %d, Denied: %d", allowed, denied)

    if denied == 0 {
        t.Fatal("expected some requests to be rate limited")
    }
}
```

### Chaos Testing Helper

```go
package chaos

import (
    "math/rand"
    "net/http"
    "time"
)

// FaultInjector adds configurable faults to HTTP handlers for testing.
type FaultInjector struct {
    // Probability of injecting a fault (0.0 - 1.0)
    ErrorRate float64

    // Range of artificial latency to add
    MinLatency time.Duration
    MaxLatency time.Duration

    // Probability of adding latency (0.0 - 1.0)
    LatencyRate float64

    // Error status code to return
    ErrorCode int
}

func (fi *FaultInjector) Wrap(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Inject latency
        if fi.LatencyRate > 0 && rand.Float64() < fi.LatencyRate {
            delay := fi.MinLatency + time.Duration(rand.Int63n(int64(fi.MaxLatency-fi.MinLatency)))
            time.Sleep(delay)
        }

        // Inject errors
        if fi.ErrorRate > 0 && rand.Float64() < fi.ErrorRate {
            code := fi.ErrorCode
            if code == 0 {
                code = http.StatusInternalServerError
            }
            w.WriteHeader(code)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// NewFlakyServer creates a test server that randomly fails.
func NewFlakyServer(errorRate float64) *httptest.Server {
    injector := &FaultInjector{
        ErrorRate:   errorRate,
        LatencyRate: 0.3,
        MinLatency:  10 * time.Millisecond,
        MaxLatency:  200 * time.Millisecond,
        ErrorCode:   http.StatusInternalServerError,
    }

    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"data":"ok"}`))
    })

    return httptest.NewServer(injector.Wrap(handler))
}
```

### Benchmarking Resilience Overhead

```go
package ratelimit_test

import (
    "testing"

    "golang.org/x/time/rate"
)

func BenchmarkTokenBucket_Allow(b *testing.B) {
    tb := NewTokenBucket(1000000, 1000000) // high limits so we measure overhead, not blocking
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        tb.Allow()
    }
}

func BenchmarkStdRateLimiter_Allow(b *testing.B) {
    limiter := rate.NewLimiter(rate.Limit(1000000), 1000000)
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        limiter.Allow()
    }
}

func BenchmarkTokenBucket_Allow_Parallel(b *testing.B) {
    tb := NewTokenBucket(1000000, 1000000)
    b.ResetTimer()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            tb.Allow()
        }
    })
}

func BenchmarkSlidingWindowCounter_Allow(b *testing.B) {
    sw := NewSlidingWindowCounter(1000000, time.Minute)
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        sw.Allow()
    }
}

func BenchmarkCircuitBreaker_Execute_Closed(b *testing.B) {
    cb := New(Settings{
        Name: "bench",
        ReadyToTrip: func(counts Counts) bool {
            return counts.ConsecutiveFailures > 1000
        },
    })
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        cb.Execute(func() error { return nil })
    }
}
```

---

## Summary and Best Practices

### Pattern Selection Guide

| Scenario | Recommended Pattern(s) |
|----------|----------------------|
| API receiving too many requests | Rate Limiter (per-client, token bucket) |
| Calling an unstable external service | Circuit Breaker + Retry with backoff |
| Brief network glitches | Retry with exponential backoff + jitter |
| One slow service blocking others | Bulkhead (semaphore isolation) |
| Gradual system overload | Load shedding (health-based) |
| Multi-service request chain | Timeout propagation via context |
| POST/payment requests with retries | Idempotency keys |
| Distributed multi-instance system | Redis-based rate limiting |

### Key Design Principles

1. **Fail fast** -- Detect failures quickly and stop wasting resources on doomed requests.
2. **Degrade gracefully** -- Provide reduced functionality instead of total failure.
3. **Recover automatically** -- Systems should self-heal without manual intervention.
4. **Isolate failures** -- One failing component should not bring down unrelated ones.
5. **Add observability** -- Log state transitions, emit metrics, set up alerts.

### Common Mistakes to Avoid

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| No timeout on HTTP clients | Goroutine/connection leak | Always set connect + request timeouts |
| Retrying non-idempotent operations | Duplicate side effects | Use idempotency keys for writes |
| Retry without backoff | Hammers failing service | Use exponential backoff + jitter |
| Retry without limit | Infinite retry storm | Set max attempts and total timeout |
| Same retry delay across clients | Thundering herd | Add jitter (full jitter preferred) |
| Circuit breaker per instance instead of per service | Inaccurate failure detection | Use per-service or shared state |
| Rate limiting without cleanup | Memory leak | Periodically evict stale entries |
| Not propagating context deadlines | Downstream timeouts ignored | Pass and shrink context through chain |

### Configuration Recommendations

```go
// Production-ready defaults for a typical microservice

// Rate Limiting
const (
    GlobalRateLimit  = 10000 // requests/second across all clients
    PerClientRate    = 100   // requests/second per client
    PerClientBurst   = 20    // burst allowance per client
)

// Circuit Breaker
const (
    CBFailureThreshold = 5       // consecutive failures to trip
    CBFailureRatio     = 0.6     // or 60% failure rate
    CBTimeout          = 30 * time.Second // time in open state
    CBHalfOpenProbes   = 3       // probes in half-open
    CBCountInterval    = 60 * time.Second // counter reset interval
)

// Retry
const (
    RetryMaxAttempts  = 3
    RetryBaseDelay    = 200 * time.Millisecond
    RetryMaxDelay     = 10 * time.Second
    RetryMultiplier   = 2.0
)

// Timeouts
const (
    ConnectTimeout         = 3 * time.Second
    RequestTimeout         = 10 * time.Second
    ResponseHeaderTimeout  = 5 * time.Second
    IdleConnTimeout        = 90 * time.Second
)

// Bulkhead
const (
    DefaultMaxConcurrent = 50
    CriticalMaxConcurrent = 200
    MaxWaitTime          = 2 * time.Second
)
```

### Monitoring Checklist

Every resilience pattern should expose these signals:

- **Rate Limiter**: requests allowed, requests denied, current token count per client
- **Circuit Breaker**: current state, state transitions (with timestamps), success/failure counts
- **Retry**: attempts per request (histogram), final success/failure rate, total retry overhead
- **Bulkhead**: active requests, rejected requests, queue depth, wait time
- **Load Shedder**: health score, shed rate, system metrics (CPU, memory, latency)

```go
// Example: Prometheus metrics for circuit breaker
var (
    cbStateGauge = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "circuit_breaker_state",
            Help: "Current state of circuit breaker (0=closed, 1=half-open, 2=open)",
        },
        []string{"service"},
    )

    cbTransitionsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "circuit_breaker_transitions_total",
            Help: "Total number of state transitions",
        },
        []string{"service", "from", "to"},
    )

    retryAttemptsHistogram = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "retry_attempts",
            Help:    "Number of retry attempts per operation",
            Buckets: []float64{1, 2, 3, 4, 5},
        },
        []string{"service", "operation"},
    )

    rateLimitRejections = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "rate_limit_rejections_total",
            Help: "Total number of rate-limited requests",
        },
        []string{"client", "endpoint"},
    )
)
```

---

### Further Reading

- **"Release It!" by Michael Nygard** -- The seminal book on stability patterns
- **AWS Architecture Blog: Exponential Backoff and Jitter** -- Definitive analysis of jitter strategies
- **Google SRE Book, Chapter 22: Addressing Cascading Failures** -- Production war stories
- **Netflix Hystrix Wiki** -- Original circuit breaker pattern documentation
- **golang.org/x/time/rate source code** -- Clean, idiomatic Go rate limiter implementation

---

*Next Chapter: [Chapter 47 - Graceful Shutdown and Health Checks](./47-graceful-shutdown-and-health-checks.md)*
