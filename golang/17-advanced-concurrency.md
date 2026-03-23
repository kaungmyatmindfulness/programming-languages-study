# Chapter 17: Advanced Concurrency Patterns

> **Level: Advanced** | **Prerequisites: Chapters on goroutines, channels, sync.Mutex, sync.WaitGroup, context**

Go's basic concurrency primitives -- goroutines, channels, mutexes, and WaitGroups -- cover most use cases. But production systems demand more: object pooling to tame the garbage collector, atomic operations for lock-free hot paths, deduplication of thundering herds, rate limiting, pipeline orchestration with proper cancellation, and the disciplined avoidance of the concurrency bugs that bring down servers at 3 AM.

This chapter covers the advanced toolkit.

---

## Table of Contents

1. [sync.Pool -- Object Pooling for GC Pressure Reduction](#1-syncpool--object-pooling-for-gc-pressure-reduction)
2. [sync/atomic -- Lock-Free Atomic Operations](#2-syncatomic--lock-free-atomic-operations)
3. [sync.Cond -- Condition Variables](#3-synccond--condition-variables)
4. [errgroup -- Concurrent Operations with Error Collection](#4-errgroup--concurrent-operations-with-error-collection)
5. [semaphore -- Weighted Concurrency Limiting](#5-semaphore--weighted-concurrency-limiting)
6. [singleflight -- Deduplicating Concurrent Requests](#6-singleflight--deduplicating-concurrent-requests)
7. [Pipeline Pattern Deep Dive](#7-pipeline-pattern-deep-dive)
8. [Fan-Out, Fan-In Deep Dive](#8-fan-out-fan-in-deep-dive)
9. [Rate Limiting](#9-rate-limiting)
10. [Concurrent Data Structures](#10-concurrent-data-structures)
11. [Common Concurrency Bugs](#11-common-concurrency-bugs)
12. [Decision Tree: Which Concurrency Primitive?](#12-decision-tree-which-concurrency-primitive)
13. [Go vs Node.js Concurrency Comparison](#13-go-vs-nodejs-concurrency-comparison)
14. [Key Takeaways](#14-key-takeaways)
15. [Practice Exercises](#15-practice-exercises)

---

## 1. sync.Pool -- Object Pooling for GC Pressure Reduction

### WHY It Exists

Every time you allocate an object (e.g., `bytes.Buffer`, a slice, a struct), Go's garbage collector must eventually track and reclaim it. In high-throughput servers handling thousands of requests per second, the constant churn of short-lived allocations creates **GC pressure** -- the garbage collector runs more frequently, causing latency spikes.

`sync.Pool` provides a **cache of allocated-but-unused objects** that can be reused instead of creating new ones. Objects are borrowed from the pool, used, and returned. The GC is free to evict pool entries between GC cycles (this is a feature, not a bug -- the pool is a hint, not a guarantee).

```
┌─────────────────────────────────────────────────────────┐
│                    sync.Pool Lifecycle                    │
│                                                           │
│   Goroutine A              Pool               GC         │
│   ─────────────           ──────           ──────         │
│                                                           │
│   pool.Get() ──────────► [buf1, buf2, buf3]               │
│   ◄──── buf1                                              │
│   (use buf1)                                              │
│   pool.Put(buf1) ──────► [buf1, buf2, buf3]               │
│                                                           │
│                           [buf1, buf2, buf3]              │
│                                  │                        │
│                              GC cycle ──► evicts some     │
│                                  │        or all entries  │
│                           [buf1] or []                    │
│                                                           │
│   pool.Get() ──────────► [] (empty)                       │
│   ◄──── New() creates                                     │
│         fresh object                                      │
└─────────────────────────────────────────────────────────┘
```

### How It Works

```go
var bufferPool = sync.Pool{
    New: func() any {
        // Called when the pool is empty and Get() is called.
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) string {
    // Borrow a buffer from the pool
    buf := bufferPool.Get().(*bytes.Buffer)

    // CRITICAL: always reset before use -- the buffer may contain
    // leftover data from a previous borrower
    buf.Reset()

    // Use the buffer
    buf.Write(data)
    buf.WriteString(" processed")
    result := buf.String()

    // Return the buffer to the pool
    bufferPool.Put(buf)

    return result
}
```

### Internals (Simplified)

Each `sync.Pool` maintains a **per-P (per-processor) local pool** plus a **shared victim cache**:

```
┌──────────────────────────────────────────────────┐
│                  sync.Pool Internals              │
│                                                    │
│   P0 local        P1 local        P2 local        │
│   ┌────────┐      ┌────────┐      ┌────────┐      │
│   │ buf, buf│      │ buf    │      │        │      │
│   └────────┘      └────────┘      └────────┘      │
│        │               │               │           │
│        └───────────────┼───────────────┘           │
│                        ▼                           │
│              ┌──────────────────┐                  │
│              │  Shared / Victim │                  │
│              │  [buf, buf, buf] │                  │
│              └──────────────────┘                  │
│                        │                           │
│                    GC cycle                        │
│                   can clear                        │
│                  victim cache                      │
└──────────────────────────────────────────────────┘
```

- `Get()` first checks the current P's local pool (lock-free, fast). If empty, it steals from other P's local pools or the shared victim cache. If all empty, it calls `New()`.
- `Put()` places the object into the current P's local pool.
- On every GC cycle, the current locals become the victim cache, and the old victim cache is discarded.

### Real-World Example: HTTP Server with bytes.Buffer Pooling

```go
package main

import (
    "bytes"
    "encoding/json"
    "net/http"
    "sync"
)

var bufPool = sync.Pool{
    New: func() any {
        return bytes.NewBuffer(make([]byte, 0, 1024)) // pre-allocate 1KB
    },
}

type Response struct {
    Status  string `json:"status"`
    Message string `json:"message"`
    Data    any    `json:"data,omitempty"`
}

func jsonResponse(w http.ResponseWriter, resp Response) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)

    // Encode JSON into the pooled buffer (not directly into w,
    // so we can check for encoding errors before writing headers)
    enc := json.NewEncoder(buf)
    if err := enc.Encode(resp); err != nil {
        http.Error(w, "encoding error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    w.Write(buf.Bytes())
}

func handler(w http.ResponseWriter, r *http.Request) {
    jsonResponse(w, Response{
        Status:  "ok",
        Message: "processed",
        Data:    map[string]int{"items": 42},
    })
}

func main() {
    http.HandleFunc("/api", handler)
    http.ListenAndServe(":8080", nil)
}
```

### When to Use sync.Pool

**Use it when:**
- You allocate and discard many identical objects in a hot path (HTTP handlers, parsers, encoders).
- Profiling (with `pprof`) shows significant allocation overhead or GC pause times.
- The objects are expensive to create or large enough that allocation overhead matters.

**Do NOT use it when:**
- You need objects to persist. Pool objects can be collected by the GC at any time.
- You're pooling very small objects -- the pool overhead may exceed the allocation cost.
- You haven't profiled first. Premature pooling adds complexity for no measurable gain.

### Common Pitfalls

```go
// BUG: Forgetting to reset before use
buf := bufPool.Get().(*bytes.Buffer)
// buf might contain "hello from previous request"
buf.Write(newData) // now contains "hello from previous requestnewData"

// BUG: Putting an object that grew too large -- pollutes pool
// with oversized buffers that waste memory
bigBuf := bufPool.Get().(*bytes.Buffer)
bigBuf.Reset()
bigBuf.Grow(10 * 1024 * 1024) // 10MB
// ... use it ...
if bigBuf.Cap() > 64*1024 {
    // Don't return oversized buffers to pool; let GC reclaim them
    return
}
bufPool.Put(bigBuf)
```

### Benchmark: Pool vs No Pool

```go
func BenchmarkWithPool(b *testing.B) {
    pool := sync.Pool{New: func() any { return new(bytes.Buffer) }}
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            buf := pool.Get().(*bytes.Buffer)
            buf.Reset()
            buf.WriteString("benchmark data")
            _ = buf.String()
            pool.Put(buf)
        }
    })
}

func BenchmarkWithoutPool(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            buf := new(bytes.Buffer)
            buf.WriteString("benchmark data")
            _ = buf.String()
        }
    })
}
```

Typical results on a modern machine:

```
BenchmarkWithPool-8       50000000       24.3 ns/op       0 B/op    0 allocs/op
BenchmarkWithoutPool-8    20000000       68.1 ns/op      64 B/op    1 allocs/op
```

The pooled version is ~2.8x faster with **zero allocations per operation**.

---

## 2. sync/atomic -- Lock-Free Atomic Operations

### WHY Atomics Over Mutexes

A `sync.Mutex` is a general-purpose lock: it can protect complex critical sections spanning many lines. But for **simple operations** on a single variable -- increment a counter, load a pointer, swap a flag -- a mutex is overkill. The lock/unlock overhead (goroutine scheduling, memory barriers) dominates the actual work.

Atomic operations use **CPU hardware instructions** (e.g., `LOCK CMPXCHG` on x86) that guarantee a read-modify-write on a single memory location is indivisible. No goroutine blocking, no scheduler involvement.

```
┌──────────────────────────────────────────────────┐
│          Mutex vs Atomic: Cost Comparison          │
│                                                    │
│   Mutex Lock/Unlock:                               │
│   ┌──────────┐                                     │
│   │ Lock()   │ ← may block goroutine               │
│   │ counter++│                                     │
│   │ Unlock() │ ← wake other goroutines             │
│   └──────────┘                                     │
│   ~25-50ns (uncontended), much worse under         │
│   contention with goroutine scheduling overhead    │
│                                                    │
│   Atomic Add:                                      │
│   ┌──────────────────┐                             │
│   │ atomic.AddInt64() │ ← single CPU instruction   │
│   └──────────────────┘                             │
│   ~5-10ns, no blocking ever                        │
└──────────────────────────────────────────────────┘
```

### Core Atomic Operations

```go
import "sync/atomic"

// --- Integer operations ---

var counter int64

// Add: atomically adds delta and returns the new value
newVal := atomic.AddInt64(&counter, 1)  // increment
newVal = atomic.AddInt64(&counter, -1)  // decrement

// Load: atomically reads the value
val := atomic.LoadInt64(&counter)

// Store: atomically writes the value
atomic.StoreInt64(&counter, 42)

// CompareAndSwap (CAS): atomically sets to new if current == old
// Returns true if the swap happened.
swapped := atomic.CompareAndSwapInt64(&counter, 42, 100)
// "If counter is currently 42, set it to 100"
```

### The CAS Loop Pattern

`CompareAndSwap` is the building block of lock-free algorithms. The pattern:

```go
// Lock-free max update: set counter to the max of its current value and newVal
func atomicMax(addr *int64, newVal int64) {
    for {
        old := atomic.LoadInt64(addr)
        if newVal <= old {
            return // nothing to do
        }
        if atomic.CompareAndSwapInt64(addr, old, newVal) {
            return // successfully updated
        }
        // CAS failed: another goroutine changed the value.
        // Loop and retry.
    }
}
```

```
┌─────────────────────────────────────────────────────┐
│              CAS Loop Execution Flow                 │
│                                                       │
│  Goroutine A                    Goroutine B           │
│  ──────────                    ──────────             │
│  Load counter → 5                                     │
│  Want to set 10                                       │
│                                 Load counter → 5      │
│                                 Want to set 7         │
│                                 CAS(5→7) ✓ success    │
│  CAS(5→10) ✗ fail                                    │
│  (counter is now 7, not 5)                            │
│  Retry: Load → 7                                      │
│  CAS(7→10) ✓ success                                 │
│                                                       │
│  Final value: 10 (correct!)                           │
└─────────────────────────────────────────────────────┘
```

### atomic.Value -- Lock-Free Configuration Updates

`atomic.Value` stores and loads an arbitrary value atomically. Perfect for **read-heavy, write-rare** scenarios like configuration hot-reloading.

```go
package main

import (
    "fmt"
    "sync/atomic"
    "time"
)

type Config struct {
    MaxConns    int
    Timeout     time.Duration
    DebugMode   bool
    AllowedIPs  []string // safe: we never mutate the slice, only replace it
}

var globalConfig atomic.Value

func init() {
    // Must store the initial value before any Load
    globalConfig.Store(&Config{
        MaxConns:   100,
        Timeout:    5 * time.Second,
        DebugMode:  false,
        AllowedIPs: []string{"10.0.0.0/8"},
    })
}

// Called by thousands of goroutines per second -- lock-free!
func getConfig() *Config {
    return globalConfig.Load().(*Config)
}

// Called rarely (e.g., SIGHUP handler, admin API)
func updateConfig(newCfg *Config) {
    globalConfig.Store(newCfg)
    // After this, all future Load() calls see the new config.
    // In-flight requests using the old *Config pointer are fine --
    // they hold a reference to the old struct, which won't be GC'd
    // until they're done.
}

func handler() {
    cfg := getConfig()
    fmt.Printf("MaxConns: %d, Debug: %v\n", cfg.MaxConns, cfg.DebugMode)
}
```

### Go 1.19+: Typed Atomics

Go 1.19 introduced typed wrappers that are cleaner and safer:

```go
import "sync/atomic"

// Old style (still works, but verbose and error-prone)
var counter int64
atomic.AddInt64(&counter, 1)
val := atomic.LoadInt64(&counter)

// New style (Go 1.19+): typed, method-based
var counter atomic.Int64
counter.Add(1)
val := counter.Load()

// Also available:
var flag atomic.Bool
flag.Store(true)
if flag.Load() { /* ... */ }

var ptr atomic.Pointer[Config]
ptr.Store(&Config{MaxConns: 100})
cfg := ptr.Load()

var val atomic.Value // for any type (stores interface{})
```

### When to Use Atomics vs Mutexes

| Scenario | Use Atomics | Use Mutex |
|----------|-------------|-----------|
| Simple counter increment | Yes | Overkill |
| Read a single flag/pointer | Yes (`atomic.Load`) | Overkill |
| Update two related fields together | No (not atomic across fields) | Yes |
| Complex critical section (map access, slice modification) | No | Yes |
| Read-heavy config (thousands of reads, rare writes) | Yes (`atomic.Value`) | Too slow |
| Guard a resource (file handle, connection) | No | Yes |

**Rule of thumb:** If the operation touches a single word-sized value and doesn't need to be grouped with other operations, use atomics. Otherwise, use a mutex.

---

## 3. sync.Cond -- Condition Variables

### WHY They Exist

Sometimes goroutines need to **wait until a specific condition becomes true**, then proceed. You might think "just use a channel," and often that works. But `sync.Cond` solves a specific class of problems:

- **Multiple goroutines waiting for the same condition** (broadcast wake-up).
- **The condition is a complex expression** on shared state, not just "a value was sent."
- **You need Broadcast** -- wake ALL waiters, not just one.

```
┌─────────────────────────────────────────────────────┐
│        Channel vs sync.Cond: When to Pick Which      │
│                                                       │
│  Channel (preferred for most cases):                  │
│  - One goroutine sends, one goroutine receives        │
│  - "Event happened" signaling                         │
│  - Passing data between goroutines                    │
│                                                       │
│  sync.Cond (specific niche):                          │
│  - Multiple goroutines waiting on shared state        │
│  - Need Broadcast to wake ALL waiters                 │
│  - Condition is complex (e.g., "queue.Len() > 0       │
│    AND !paused AND connectionCount < maxConns")       │
│  - Integrates with existing mutex-protected state     │
└─────────────────────────────────────────────────────┘
```

### How It Works

```go
import "sync"

var (
    mu    sync.Mutex
    cond  = sync.NewCond(&mu) // Cond is always paired with a Locker
    ready bool
)

// Waiter goroutine
func waiter(id int) {
    mu.Lock()
    for !ready {    // MUST be a loop -- spurious wakeups are possible
        cond.Wait() // atomically: unlocks mu, suspends goroutine, re-locks mu on wake
    }
    fmt.Printf("Waiter %d: proceeding\n", id)
    mu.Unlock()
}

// Signaler goroutine
func signaler() {
    time.Sleep(time.Second)
    mu.Lock()
    ready = true
    mu.Unlock()
    cond.Broadcast() // wake ALL waiters
}
```

**Critical rule:** Always check the condition in a `for` loop, never an `if`. The goroutine can be woken spuriously or another goroutine may have consumed the condition.

### Detailed Example: Bounded Blocking Queue

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type BoundedQueue struct {
    mu       sync.Mutex
    notEmpty *sync.Cond
    notFull  *sync.Cond
    items    []int
    capacity int
}

func NewBoundedQueue(capacity int) *BoundedQueue {
    q := &BoundedQueue{
        items:    make([]int, 0, capacity),
        capacity: capacity,
    }
    q.notEmpty = sync.NewCond(&q.mu)
    q.notFull = sync.NewCond(&q.mu)
    return q
}

// Enqueue blocks if the queue is full.
func (q *BoundedQueue) Enqueue(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.items) == q.capacity {
        q.notFull.Wait() // wait until not full
    }

    q.items = append(q.items, item)
    q.notEmpty.Signal() // wake ONE consumer
}

// Dequeue blocks if the queue is empty.
func (q *BoundedQueue) Dequeue() int {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.items) == 0 {
        q.notEmpty.Wait() // wait until not empty
    }

    item := q.items[0]
    q.items = q.items[1:]
    q.notFull.Signal() // wake ONE producer
    return item
}

func main() {
    q := NewBoundedQueue(3)

    // Producer
    go func() {
        for i := 0; i < 10; i++ {
            q.Enqueue(i)
            fmt.Printf("Produced: %d\n", i)
        }
    }()

    // Consumer (slow)
    for i := 0; i < 10; i++ {
        time.Sleep(200 * time.Millisecond)
        val := q.Dequeue()
        fmt.Printf("Consumed: %d\n", val)
    }
}
```

```
┌──────────────────────────────────────────────────────────┐
│           Bounded Queue with sync.Cond                    │
│                                                            │
│  Producer                    Queue [cap=3]     Consumer    │
│  ────────                    ────────────      ────────    │
│  Enqueue(0) ─────────────► [0]                             │
│  Enqueue(1) ─────────────► [0, 1]                          │
│  Enqueue(2) ─────────────► [0, 1, 2]                       │
│  Enqueue(3) ─── BLOCKS ──  (full, Wait on notFull)         │
│                                              Dequeue() → 0 │
│                             [1, 2]                         │
│             ◄── Signal(notFull)                             │
│  Enqueue(3) ─────────────► [1, 2, 3]                       │
│  ...                                                       │
└──────────────────────────────────────────────────────────┘
```

### When to Use sync.Cond vs Alternatives

| Need | Best Tool |
|------|-----------|
| Wait for one event, pass a value | Channel |
| Wait for one event, no value | `chan struct{}` or `context.Done()` |
| Multiple waiters, wake all | `sync.Cond` + `Broadcast` |
| Complex condition on shared state | `sync.Cond` |
| Timeout on wait | Channel + `select` (sync.Cond has no built-in timeout) |

> **Practical note:** `sync.Cond` is rarely used in application code. Channels and context cover most patterns. You'll see `sync.Cond` in low-level infrastructure: schedulers, custom queues, connection pools.

---

## 4. errgroup -- Concurrent Operations with Error Collection

### WHY It's Better Than Manual WaitGroup + Error Handling

With `sync.WaitGroup`, handling errors from concurrent goroutines is tedious and error-prone:

```go
// Manual approach: verbose, easy to mess up
var wg sync.WaitGroup
var mu sync.Mutex
var firstErr error

for _, url := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        if err := fetch(u); err != nil {
            mu.Lock()
            if firstErr == nil {
                firstErr = err
            }
            mu.Unlock()
        }
    }(url)
}
wg.Wait()
// But wait: even if one fails early, ALL goroutines keep running.
// No cancellation, no way to short-circuit.
```

`errgroup` solves this cleanly:

```go
import "golang.org/x/sync/errgroup"

// errgroup approach: clean, with automatic cancellation
g, ctx := errgroup.WithContext(context.Background())

for _, url := range urls {
    url := url // capture loop variable (pre-Go 1.22)
    g.Go(func() error {
        return fetchWithContext(ctx, url) // ctx is cancelled on first error
    })
}

if err := g.Wait(); err != nil {
    // err is the first non-nil error returned by any goroutine
    log.Fatalf("fetch failed: %v", err)
}
```

### What errgroup Gives You

1. **Wait + first error** -- `g.Wait()` blocks until all goroutines finish, returns the first error.
2. **Automatic cancellation** -- `WithContext` creates a context that's cancelled when any goroutine returns an error. Other goroutines can check `ctx.Done()` and bail out early.
3. **Concurrency limiting** -- `g.SetLimit(n)` caps the number of concurrent goroutines (Go 1.20+).
4. **TryGo** -- `g.TryGo(f)` starts the goroutine only if under the concurrency limit (non-blocking).

### Full Production Example: Parallel API Fetcher

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"

    "golang.org/x/sync/errgroup"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func fetchUser(ctx context.Context, client *http.Client, id int) (*User, error) {
    url := fmt.Sprintf("https://api.example.com/users/%d", id)
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request for user %d: %w", id, err)
    }

    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("fetch user %d: %w", id, err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("user %d: status %d", id, resp.StatusCode)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, fmt.Errorf("decode user %d: %w", id, err)
    }
    return &user, nil
}

func fetchAllUsers(ctx context.Context, userIDs []int) ([]*User, error) {
    client := &http.Client{Timeout: 10 * time.Second}
    users := make([]*User, len(userIDs))

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(5) // at most 5 concurrent HTTP requests

    for i, id := range userIDs {
        i, id := i, id
        g.Go(func() error {
            user, err := fetchUser(ctx, client, id)
            if err != nil {
                return err // cancels ctx, other goroutines will see ctx.Done()
            }
            users[i] = user // safe: each goroutine writes to a unique index
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return users, nil
}

func main() {
    ctx := context.Background()
    users, err := fetchAllUsers(ctx, []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    for _, u := range users {
        fmt.Printf("User: %+v\n", u)
    }
}
```

```
┌──────────────────────────────────────────────────────────┐
│              errgroup with SetLimit(3)                     │
│                                                            │
│  g.Go(task1) ──► [goroutine 1] ─── running                │
│  g.Go(task2) ──► [goroutine 2] ─── running                │
│  g.Go(task3) ──► [goroutine 3] ─── running                │
│  g.Go(task4) ──► (waits... limit=3)                       │
│                                                            │
│  [goroutine 2] returns error ──► ctx cancelled             │
│                                                            │
│  [goroutine 1] sees ctx.Done() → returns early             │
│  [goroutine 3] sees ctx.Done() → returns early             │
│  (task4 starts but immediately sees ctx.Done())            │
│                                                            │
│  g.Wait() returns first error from goroutine 2             │
└──────────────────────────────────────────────────────────┘
```

---

## 5. semaphore -- Weighted Concurrency Limiting

### WHY It Exists

`errgroup.SetLimit` limits the number of goroutines. But what if different operations have different "weights"? For example, a large database query should count as "3 slots" while a small one counts as "1 slot." This is what `golang.org/x/sync/semaphore` provides: a **weighted semaphore**.

### Basic Usage

```go
import "golang.org/x/sync/semaphore"

// Create a semaphore with capacity 10
sem := semaphore.NewWeighted(10)

// Acquire 1 unit (blocks if fewer than 1 available)
if err := sem.Acquire(ctx, 1); err != nil {
    // ctx was cancelled while waiting
    return err
}
// ... do work ...
sem.Release(1)

// Acquire 3 units (for a "heavy" operation)
if err := sem.Acquire(ctx, 3); err != nil {
    return err
}
// ... heavy work ...
sem.Release(3)

// TryAcquire: non-blocking, returns false if not available
if sem.TryAcquire(1) {
    // got it
    defer sem.Release(1)
    // ... work ...
} else {
    // couldn't acquire, do something else
}
```

### Real-World Example: Limiting Concurrent Database Connections

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    "golang.org/x/sync/errgroup"
    "golang.org/x/sync/semaphore"
)

// DBPool wraps a sql.DB with weighted semaphore for query limiting
type DBPool struct {
    db  *sql.DB
    sem *semaphore.Weighted
}

func NewDBPool(db *sql.DB, maxConcurrentQueries int64) *DBPool {
    return &DBPool{
        db:  db,
        sem: semaphore.NewWeighted(maxConcurrentQueries),
    }
}

// LightQuery acquires 1 slot
func (p *DBPool) LightQuery(ctx context.Context, query string) (*sql.Rows, error) {
    if err := p.sem.Acquire(ctx, 1); err != nil {
        return nil, fmt.Errorf("acquire semaphore: %w", err)
    }
    // NOTE: don't release here -- caller must call rows.Close(),
    // which we wrap to release the semaphore
    rows, err := p.db.QueryContext(ctx, query)
    if err != nil {
        p.sem.Release(1) // release on error
        return nil, err
    }
    return rows, nil
}

// HeavyQuery acquires 3 slots (prevents too many expensive queries running)
func (p *DBPool) HeavyQuery(ctx context.Context, query string) (*sql.Rows, error) {
    if err := p.sem.Acquire(ctx, 3); err != nil {
        return nil, fmt.Errorf("acquire semaphore: %w", err)
    }
    rows, err := p.db.QueryContext(ctx, query)
    if err != nil {
        p.sem.Release(3)
        return nil, err
    }
    return rows, nil
}

// Example: process 1000 items with limited DB concurrency
func processItems(ctx context.Context, pool *DBPool, items []string) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, item := range items {
        item := item
        g.Go(func() error {
            // Semaphore inside pool.LightQuery limits actual DB concurrency,
            // even though we might have hundreds of goroutines
            rows, err := pool.LightQuery(ctx,
                fmt.Sprintf("SELECT * FROM items WHERE name = '%s'", item))
            if err != nil {
                return err
            }
            defer rows.Close()
            // process rows...
            return nil
        })
    }

    return g.Wait()
}
```

```
┌──────────────────────────────────────────────────────┐
│      Weighted Semaphore (capacity=10)                 │
│                                                        │
│  Available: [██████████] 10/10                         │
│                                                        │
│  LightQuery (weight=1): Acquire(1)                     │
│  Available: [█████████░] 9/10                          │
│                                                        │
│  LightQuery (weight=1): Acquire(1)                     │
│  Available: [████████░░] 8/10                          │
│                                                        │
│  HeavyQuery (weight=3): Acquire(3)                     │
│  Available: [█████░░░░░] 5/10                          │
│                                                        │
│  HeavyQuery (weight=3): Acquire(3)                     │
│  Available: [██░░░░░░░░] 2/10                          │
│                                                        │
│  HeavyQuery (weight=3): Acquire(3) ← BLOCKS           │
│  (only 2 available, needs 3)                           │
│                                                        │
│  First LightQuery finishes: Release(1)                 │
│  Available: [███░░░░░░░] 3/10  ← unblocks HeavyQuery  │
└──────────────────────────────────────────────────────┘
```

---

## 6. singleflight -- Deduplicating Concurrent Requests

### WHY It Matters: The Thundering Herd Problem

Imagine your cache key expires. Suddenly, 1000 concurrent requests all miss the cache and hit the database with the same query:

```
┌──────────────────────────────────────────────────────┐
│            The Thundering Herd Problem                 │
│                                                        │
│  Request 1 ──► Cache MISS ──► DB Query "SELECT ..."    │
│  Request 2 ──► Cache MISS ──► DB Query "SELECT ..."    │
│  Request 3 ──► Cache MISS ──► DB Query "SELECT ..."    │
│  ...                                                   │
│  Request 1000 ► Cache MISS ──► DB Query "SELECT ..."   │
│                                                        │
│  1000 IDENTICAL queries hit the database!              │
│  Database overloaded. Cascading failure.               │
│                                                        │
│  ═══════════════════════════════════════════════        │
│                                                        │
│  With singleflight:                                    │
│                                                        │
│  Request 1 ──► Cache MISS ──► DB Query "SELECT ..."    │
│  Request 2 ──► Cache MISS ──► (waits for Request 1)    │
│  Request 3 ──► Cache MISS ──► (waits for Request 1)    │
│  ...                                                   │
│  Request 1000 ► Cache MISS ──► (waits for Request 1)   │
│                                                        │
│  1 query hits the database. All 1000 get the result.   │
└──────────────────────────────────────────────────────┘
```

### How singleflight Works

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

// Do: execute fn only once per key. Concurrent callers with the same key
// block and receive the same result.
result, err, shared := group.Do("cache-key", func() (interface{}, error) {
    // This function runs ONCE, even if 1000 goroutines called Do concurrently.
    return expensiveDatabaseQuery()
})
// shared == true means this result was shared with other callers
```

### Production Example: Cache with Singleflight Protection

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/singleflight"
)

type Product struct {
    ID    string
    Name  string
    Price float64
}

type ProductCache struct {
    mu    sync.RWMutex
    items map[string]*cacheEntry
    group singleflight.Group
    ttl   time.Duration
}

type cacheEntry struct {
    product   *Product
    expiresAt time.Time
}

func NewProductCache(ttl time.Duration) *ProductCache {
    return &ProductCache{
        items: make(map[string]*cacheEntry),
        ttl:   ttl,
    }
}

func (c *ProductCache) Get(ctx context.Context, productID string) (*Product, error) {
    // 1. Check cache (read lock -- fast, concurrent reads)
    c.mu.RLock()
    if entry, ok := c.items[productID]; ok && time.Now().Before(entry.expiresAt) {
        c.mu.RUnlock()
        return entry.product, nil
    }
    c.mu.RUnlock()

    // 2. Cache miss -- use singleflight to deduplicate concurrent fetches
    result, err, shared := c.group.Do(productID, func() (interface{}, error) {
        // Double-check: another goroutine may have populated the cache
        // while we were waiting for the singleflight lock
        c.mu.RLock()
        if entry, ok := c.items[productID]; ok && time.Now().Before(entry.expiresAt) {
            c.mu.RUnlock()
            return entry.product, nil
        }
        c.mu.RUnlock()

        // Actually fetch from database
        product, err := fetchProductFromDB(ctx, productID)
        if err != nil {
            return nil, err
        }

        // Populate cache
        c.mu.Lock()
        c.items[productID] = &cacheEntry{
            product:   product,
            expiresAt: time.Now().Add(c.ttl),
        }
        c.mu.Unlock()

        return product, nil
    })

    if err != nil {
        return nil, err
    }

    if shared {
        fmt.Printf("Product %s: result shared with other callers\n", productID)
    }

    return result.(*Product), nil
}

func fetchProductFromDB(ctx context.Context, id string) (*Product, error) {
    // Simulate database query
    time.Sleep(100 * time.Millisecond)
    return &Product{ID: id, Name: "Widget", Price: 9.99}, nil
}
```

### singleflight.Group Methods

```go
var group singleflight.Group

// Do: blocks until the function returns. All callers get the same result.
v, err, shared := group.Do(key, fn)

// DoChan: returns a channel. Useful with select for timeouts.
ch := group.DoChan(key, fn)
select {
case result := <-ch:
    v, err = result.Val, result.Err
case <-ctx.Done():
    return ctx.Err()
}

// Forget: removes the key from the group, allowing the next Do to start fresh.
// Useful when you know the in-flight result is stale.
group.Forget(key)
```

### When to Use singleflight

| Scenario | Use singleflight? |
|----------|-------------------|
| Cache stampede / thundering herd | Yes -- primary use case |
| Deduplicating identical API calls | Yes |
| Database query deduplication | Yes |
| Operations that should always run (e.g., writes) | No -- you want each write to execute |
| Long-running operations with updates | Maybe -- consider Forget() |

---

## 7. Pipeline Pattern Deep Dive

### The Pipeline

A pipeline is a series of stages connected by channels, where each stage is a group of goroutines running the same function. Each stage:
1. Receives values from an upstream channel
2. Performs some processing
3. Sends values to a downstream channel

```
┌──────────────────────────────────────────────────────────┐
│                    Pipeline Architecture                   │
│                                                            │
│  ┌─────────┐    chan    ┌──────────┐    chan    ┌────────┐  │
│  │ Generate │ ────────► │ Process  │ ────────► │ Output │  │
│  │ (stage1) │           │ (stage2) │           │(stage3)│  │
│  └─────────┘           └──────────┘           └────────┘  │
│       │                      │                     │       │
│   produces              transforms             consumes    │
│   values                 values                 values     │
│                                                            │
│  Each stage:                                               │
│  - Runs in its own goroutine(s)                            │
│  - Reads from input channel                                │
│  - Writes to output channel                                │
│  - Closes output channel when done                         │
└──────────────────────────────────────────────────────────┘
```

### Production Pipeline with Cancellation and Error Handling

```go
package main

import (
    "context"
    "fmt"
    "math"
    "runtime"
    "sync"
)

// Stage 1: Generate numbers
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return // pipeline cancelled
            }
        }
    }()
    return out
}

// Stage 2: Square each number (CPU-bound work)
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Stage 3: Filter -- only pass through values matching predicate
func filter(ctx context.Context, in <-chan int, pred func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if pred(n) {
                select {
                case out <- n:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

// Sink: collect results
func collect(ctx context.Context, in <-chan int) ([]int, error) {
    var results []int
    for n := range in {
        select {
        case <-ctx.Done():
            return results, ctx.Err()
        default:
            results = append(results, n)
        }
    }
    return results, nil
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Build the pipeline: generate -> square -> filter(isPrime)
    nums := generate(ctx, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13)
    squared := square(ctx, nums)
    primeSquares := filter(ctx, squared, func(n int) bool {
        if n < 2 {
            return false
        }
        for i := 2; i <= int(math.Sqrt(float64(n))); i++ {
            if n%i == 0 {
                return false
            }
        }
        return true
    })

    results, err := collect(ctx, primeSquares)
    if err != nil {
        fmt.Printf("Pipeline error: %v\n", err)
        return
    }
    fmt.Println("Prime squares:", results) // [4, 9, 25, 49, 121, 169]
}
```

### Pipeline with Error Propagation

The tricky part of pipelines is handling errors without leaking goroutines. Here's a robust pattern:

```go
// Result carries a value or an error through the pipeline
type Result[T any] struct {
    Value T
    Err   error
}

// Stage that can fail
func transform[In, Out any](
    ctx context.Context,
    in <-chan Result[In],
    fn func(In) (Out, error),
) <-chan Result[Out] {
    out := make(chan Result[Out])
    go func() {
        defer close(out)
        for r := range in {
            // Propagate upstream errors without processing
            if r.Err != nil {
                select {
                case out <- Result[Out]{Err: r.Err}:
                case <-ctx.Done():
                    return
                }
                continue
            }

            // Process the value
            val, err := fn(r.Value)
            select {
            case out <- Result[Out]{Value: val, Err: err}:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### Pipeline Teardown

```
┌──────────────────────────────────────────────────────────┐
│          Pipeline Teardown (No Goroutine Leaks)           │
│                                                            │
│  Normal completion:                                        │
│  generate closes out → square sees range end → closes out  │
│  → filter sees range end → closes out → collect returns    │
│                                                            │
│  Cancellation (ctx cancelled):                             │
│  All stages check ctx.Done() on every send/receive.        │
│  When cancelled:                                           │
│  1. Stages stop sending (select catches ctx.Done)          │
│  2. Downstream ranges complete (channel closed or drained) │
│  3. All goroutines exit                                    │
│                                                            │
│  KEY RULE: Every goroutine must have a guaranteed exit     │
│  path. Every channel send must be wrapped in select        │
│  with ctx.Done() to prevent blocking forever.              │
└──────────────────────────────────────────────────────────┘
```

---

## 8. Fan-Out, Fan-In Deep Dive

### The Pattern

**Fan-out:** Start multiple goroutines to read from the same channel, distributing work.
**Fan-in:** Merge multiple channels into a single channel.

```
┌──────────────────────────────────────────────────────────┐
│              Fan-Out, Fan-In Architecture                  │
│                                                            │
│                        Fan-Out                             │
│                    ┌──► Worker 1 ──┐                       │
│                    │                │                       │
│  Source ──► jobs ──┼──► Worker 2 ──┼──► results ──► Sink   │
│                    │                │                       │
│                    └──► Worker 3 ──┘                       │
│                                                            │
│              Fan-In (merge)                                │
│                                                            │
│  Multiple goroutines read from `jobs` (fan-out).           │
│  All write to shared `results` channel (fan-in).           │
│                                                            │
│  OR: each worker has its own output channel,               │
│  and a merge function combines them.                       │
└──────────────────────────────────────────────────────────┘
```

### Dynamic Worker Pool with Backpressure

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "time"
)

type Job struct {
    ID      int
    Payload string
}

type Result struct {
    JobID    int
    Output   string
    Duration time.Duration
}

// WorkerPool processes jobs with a fixed number of workers.
// The jobs channel provides natural backpressure: if workers are busy,
// sends to jobs will block, slowing the producer.
func WorkerPool(ctx context.Context, numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return
                default:
                }

                start := time.Now()
                // Simulate processing
                output := fmt.Sprintf("worker-%d processed job-%d: %s",
                    workerID, job.ID, job.Payload)
                time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)

                select {
                case results <- Result{
                    JobID:    job.ID,
                    Output:   output,
                    Duration: time.Since(start),
                }:
                case <-ctx.Done():
                    return
                }
            }
        }(i)
    }

    // Close results when all workers are done
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Buffered channel provides backpressure:
    // producer blocks when 10 jobs are queued and workers are busy
    jobs := make(chan Job, 10)

    // Start worker pool (fan-out)
    numWorkers := runtime.NumCPU()
    results := WorkerPool(ctx, numWorkers, jobs)

    // Producer: generate jobs
    go func() {
        defer close(jobs)
        for i := 0; i < 50; i++ {
            select {
            case jobs <- Job{ID: i, Payload: fmt.Sprintf("task-%d", i)}:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Consumer: collect results (fan-in already done by shared results channel)
    for result := range results {
        fmt.Printf("Job %d completed in %v: %s\n",
            result.JobID, result.Duration, result.Output)
    }
}
```

### Merge Function (Fan-In from Multiple Channels)

When each worker has its own output channel, use a merge function:

```go
// merge combines multiple channels into one.
// The output channel is closed when ALL input channels are closed.
func merge[T any](ctx context.Context, channels ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan T) {
            defer wg.Done()
            for val := range c {
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### Backpressure Explained

```
┌──────────────────────────────────────────────────────────┐
│                   Backpressure Flow                        │
│                                                            │
│  Scenario: jobs channel buffer = 3, 2 workers              │
│                                                            │
│  Producer (fast)         Buffer [cap=3]      Workers       │
│  ──────────────         ──────────────      ────────       │
│                                                            │
│  send job1 ──────────► [job1, _, _]                        │
│  send job2 ──────────► [job1, job2, _]     worker1 ← job1 │
│  send job3 ──────────► [job2, job3, _]     worker2 ← job2 │
│  send job4 ──────────► [job3, job4, _]                     │
│  send job5 ──────────► [job3, job4, job5]                  │
│  send job6 ──── BLOCKS (buffer full!)                      │
│                                                            │
│  This is backpressure: the fast producer is slowed down    │
│  to match the pace of the slow consumers. No unbounded     │
│  memory growth, no dropped work.                           │
│                                                            │
│  When worker1 finishes:                                    │
│  [job4, job5, _]       worker1 ← job3                     │
│  send job6 unblocks → [job4, job5, job6]                   │
└──────────────────────────────────────────────────────────┘
```

---

## 9. Rate Limiting

### Simple Rate Limiting with time.Ticker

```go
// Process at most 10 items per second
func rateLimitedProcess(items []string) {
    ticker := time.NewTicker(100 * time.Millisecond) // 10 per second
    defer ticker.Stop()

    for _, item := range items {
        <-ticker.C // wait for the next tick
        process(item)
    }
}
```

This is simple but inflexible -- no bursting, no dynamic adjustment.

### Token Bucket with golang.org/x/time/rate

The `rate.Limiter` implements the **token bucket algorithm**:
- A bucket holds up to `b` tokens (burst size).
- Tokens are added at a rate of `r` per second.
- Each operation consumes one (or more) tokens.
- If no tokens are available, the operation waits.

```
┌──────────────────────────────────────────────────────────┐
│              Token Bucket Algorithm                        │
│                                                            │
│   Bucket capacity (burst) = 5                              │
│   Refill rate = 2 tokens/second                            │
│                                                            │
│   Time 0.0s: [●●●●●] 5 tokens (full)                      │
│   Request:   [●●●●○] 4 tokens (consumed 1)                │
│   Request:   [●●●○○] 3 tokens                             │
│   Request:   [●●○○○] 2 tokens                             │
│   Request:   [●○○○○] 1 token                              │
│   Request:   [○○○○○] 0 tokens                             │
│   Request:   WAIT... (no tokens)                           │
│                                                            │
│   Time 0.5s: [●○○○○] 1 token (refilled)                   │
│   Request:   [○○○○○] 0 tokens (consumed)                  │
│   Request:   WAIT...                                       │
│                                                            │
│   Time 1.0s: [●●○○○] 2 tokens (refilled 2 in 1s)         │
│   2 requests can proceed immediately                       │
└──────────────────────────────────────────────────────────┘
```

### rate.Limiter in Practice

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// --- Per-client rate limiting for an HTTP API ---

type ClientLimiter struct {
    mu       sync.Mutex
    clients  map[string]*rate.Limiter
    rate     rate.Limit
    burst    int
}

func NewClientLimiter(r rate.Limit, burst int) *ClientLimiter {
    return &ClientLimiter{
        clients: make(map[string]*rate.Limiter),
        rate:    r,
        burst:   burst,
    }
}

func (cl *ClientLimiter) GetLimiter(clientIP string) *rate.Limiter {
    cl.mu.Lock()
    defer cl.mu.Unlock()

    limiter, exists := cl.clients[clientIP]
    if !exists {
        limiter = rate.NewLimiter(cl.rate, cl.burst)
        cl.clients[clientIP] = limiter
    }
    return limiter
}

// RateLimitMiddleware limits each client to 10 requests/second with burst of 20
func RateLimitMiddleware(cl *ClientLimiter, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        limiter := cl.GetLimiter(r.RemoteAddr)

        if !limiter.Allow() {
            // Allow() consumes 1 token. Returns false if no tokens available.
            http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func main() {
    cl := NewClientLimiter(10, 20) // 10 req/s, burst 20

    mux := http.NewServeMux()
    mux.HandleFunc("/api", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "OK")
    })

    server := &http.Server{
        Addr:    ":8080",
        Handler: RateLimitMiddleware(cl, mux),
    }
    server.ListenAndServe()
}
```

### rate.Limiter Methods

```go
limiter := rate.NewLimiter(10, 20) // 10 events/sec, burst 20

// Allow: non-blocking. Returns true if a token is available (consumes it).
if limiter.Allow() {
    // proceed
}

// Wait: blocks until a token is available or ctx is cancelled.
if err := limiter.Wait(ctx); err != nil {
    // context cancelled or deadline exceeded
}

// Reserve: returns a Reservation that tells you how long to wait.
// Useful for advanced scenarios (e.g., logging wait times).
reservation := limiter.Reserve()
if !reservation.OK() {
    // Cannot satisfy (burst too small)
}
delay := reservation.Delay()
time.Sleep(delay) // or use a timer

// WaitN: wait for N tokens (for batch operations)
if err := limiter.WaitN(ctx, 5); err != nil {
    // ...
}

// Dynamic adjustment:
limiter.SetLimit(20)  // change to 20 events/sec
limiter.SetBurst(50)  // change burst to 50
```

### Choosing a Rate Limiting Strategy

| Strategy | Use Case | Go Implementation |
|----------|----------|-------------------|
| Fixed window | Simple, approximate | Custom (counter + timer) |
| Sliding window | More accurate | Custom |
| Token bucket | Smooth, allows bursts | `golang.org/x/time/rate` |
| Leaky bucket | Strictly smooth output | Custom (channel-based) |
| Per-client limiting | API rate limiting | Map of `rate.Limiter` |

---

## 10. Concurrent Data Structures

### Building a Thread-Safe Map (Beyond sync.Map)

`sync.Map` is optimized for two specific patterns: (1) write-once, read-many, and (2) disjoint key sets per goroutine. For general-purpose concurrent maps, a sharded map with fine-grained locking often performs better.

```go
package main

import (
    "fmt"
    "hash/fnv"
    "sync"
)

const numShards = 32

// ShardedMap splits a map into multiple shards, each with its own lock.
// This reduces lock contention: goroutines accessing different shards
// never block each other.
type ShardedMap[K comparable, V any] struct {
    shards [numShards]shard[K, V]
    hash   func(K) uint64
}

type shard[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]V
}

func NewShardedMap[V any]() *ShardedMap[string, V] {
    sm := &ShardedMap[string, V]{
        hash: func(key string) uint64 {
            h := fnv.New64a()
            h.Write([]byte(key))
            return h.Sum64()
        },
    }
    for i := range sm.shards {
        sm.shards[i].items = make(map[string]V)
    }
    return sm
}

func (sm *ShardedMap[K, V]) getShard(key K) *shard[K, V] {
    h := sm.hash(key)
    return &sm.shards[h%numShards]
}

func (sm *ShardedMap[K, V]) Set(key K, value V) {
    s := sm.getShard(key)
    s.mu.Lock()
    s.items[key] = value
    s.mu.Unlock()
}

func (sm *ShardedMap[K, V]) Get(key K) (V, bool) {
    s := sm.getShard(key)
    s.mu.RLock()
    val, ok := s.items[key]
    s.mu.RUnlock()
    return val, ok
}

func (sm *ShardedMap[K, V]) Delete(key K) {
    s := sm.getShard(key)
    s.mu.Lock()
    delete(s.items, key)
    s.mu.Unlock()
}

// Range iterates over all items. Acquires locks shard by shard.
func (sm *ShardedMap[K, V]) Range(fn func(key K, value V) bool) {
    for i := range sm.shards {
        s := &sm.shards[i]
        s.mu.RLock()
        for k, v := range s.items {
            if !fn(k, v) {
                s.mu.RUnlock()
                return
            }
        }
        s.mu.RUnlock()
    }
}
```

```
┌──────────────────────────────────────────────────────────┐
│             ShardedMap Lock Contention                     │
│                                                            │
│  Regular sync.Mutex map:                                   │
│  All goroutines contend for ONE lock                       │
│  ┌──────────────┐                                          │
│  │   LOCK (1)   │ ← bottleneck                            │
│  │  map[k]v     │                                          │
│  └──────────────┘                                          │
│                                                            │
│  ShardedMap (32 shards):                                   │
│  Goroutines spread across 32 independent locks             │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐      ┌────┐                  │
│  │ S0 │ │ S1 │ │ S2 │ │ S3 │ ···  │S31 │                  │
│  └────┘ └────┘ └────┘ └────┘      └────┘                  │
│    ↑      ↑      ↑                                         │
│   G1     G2     G3    (minimal contention)                 │
└──────────────────────────────────────────────────────────┘
```

### Lock-Free Stack (CAS-based)

For educational purposes -- demonstrating lock-free programming with `CompareAndSwap`:

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "unsafe"
)

type node[T any] struct {
    value T
    next  *node[T]
}

// LockFreeStack uses CAS for thread-safe push/pop without locks.
type LockFreeStack[T any] struct {
    top atomic.Pointer[node[T]]
    len atomic.Int64
}

func (s *LockFreeStack[T]) Push(value T) {
    newNode := &node[T]{value: value}
    for {
        oldTop := s.top.Load()
        newNode.next = oldTop
        if s.top.CompareAndSwap(oldTop, newNode) {
            s.len.Add(1)
            return
        }
        // CAS failed: another goroutine modified top. Retry.
    }
}

func (s *LockFreeStack[T]) Pop() (T, bool) {
    for {
        oldTop := s.top.Load()
        if oldTop == nil {
            var zero T
            return zero, false
        }
        if s.top.CompareAndSwap(oldTop, oldTop.next) {
            s.len.Add(-1)
            return oldTop.value, true
        }
        // CAS failed: retry.
    }
}

func (s *LockFreeStack[T]) Len() int64 {
    return s.len.Load()
}

func main() {
    var stack LockFreeStack[int]

    var wg sync.WaitGroup
    // 10 goroutines each push 1000 values
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(base int) {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                stack.Push(base*1000 + j)
            }
        }(i)
    }
    wg.Wait()
    fmt.Printf("Stack length: %d\n", stack.Len()) // 10000
}
```

> **Note on lock-free programming:** Lock-free data structures are tricky to get right and should only be used when profiling proves mutex contention is a bottleneck. In most Go programs, a simple mutex or `sync.Map` is sufficient and far easier to reason about. The lock-free stack above is educational -- in production, prefer proven implementations.

### sync.Map: When to Use

```go
var cache sync.Map

// Store
cache.Store("key1", "value1")

// Load
val, ok := cache.Load("key1")

// LoadOrStore: atomic "get or set"
actual, loaded := cache.LoadOrStore("key2", "default")
// loaded==false means we stored "default"
// loaded==true  means "key2" already existed, actual is the existing value

// LoadAndDelete: atomic "get and remove"
val, loaded := cache.LoadAndDelete("key1")

// Range (not snapshot -- concurrent modifications may or may not be seen)
cache.Range(func(key, value any) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true // return false to stop iteration
})
```

**sync.Map is optimized for:**
- Keys written once, read many times (e.g., caches that are populated at startup)
- Many goroutines read/write disjoint sets of keys

**sync.Map is NOT good for:**
- General-purpose map with mixed read/write patterns -- use `RWMutex` + map or ShardedMap

---

## 11. Common Concurrency Bugs

### Bug 1: Data Races

A data race occurs when two goroutines access the same variable concurrently and at least one access is a write, without synchronization.

```go
// BUG: Data race on 'count'
var count int

func increment() {
    for i := 0; i < 1000; i++ {
        count++ // READ count, ADD 1, WRITE count -- not atomic!
    }
}

func main() {
    go increment()
    go increment()
    time.Sleep(time.Second)
    fmt.Println(count) // Unpredictable! Could be 1000, 1500, 2000, anything.
}
```

**Fix: Use atomic or mutex**

```go
// Fix 1: atomic
var count atomic.Int64

func increment() {
    for i := 0; i < 1000; i++ {
        count.Add(1)
    }
}

// Fix 2: mutex
var (
    mu    sync.Mutex
    count int
)

func increment() {
    for i := 0; i < 1000; i++ {
        mu.Lock()
        count++
        mu.Unlock()
    }
}
```

**Detection: Go's race detector**

```bash
go run -race main.go
go test -race ./...
```

```
==================
WARNING: DATA RACE
Read at 0x00c0000b4010 by goroutine 7:
  main.increment()
      /main.go:10 +0x3e

Previous write at 0x00c0000b4010 by goroutine 6:
  main.increment()
      /main.go:10 +0x56

Goroutine 7 (running) created at:
  main.main()
      /main.go:15 +0x7a
==================
```

> **Best practice:** Always run `go test -race` in CI. The race detector has near-zero false positives. If it fires, you have a real bug.

### Bug 2: Deadlocks

A deadlock occurs when two or more goroutines are each waiting for the other to release a resource.

```go
// BUG: Classic ABBA deadlock
var mu1, mu2 sync.Mutex

func goroutine1() {
    mu1.Lock()
    time.Sleep(time.Millisecond)
    mu2.Lock() // waits for goroutine2 to release mu2
    // ... critical section ...
    mu2.Unlock()
    mu1.Unlock()
}

func goroutine2() {
    mu2.Lock()
    time.Sleep(time.Millisecond)
    mu1.Lock() // waits for goroutine1 to release mu1
    // ... critical section ...
    mu1.Unlock()
    mu2.Unlock()
}

// Both goroutines wait forever.
```

```
┌──────────────────────────────────────────────────────────┐
│                   ABBA Deadlock                           │
│                                                            │
│  Goroutine 1               Goroutine 2                    │
│  ──────────               ──────────                      │
│  Lock(mu1) ✓              Lock(mu2) ✓                     │
│       │                        │                           │
│  Lock(mu2) ← WAIT         Lock(mu1) ← WAIT               │
│       │         ↑              │         ↑                 │
│       └─────────┼──────────────┘         │                 │
│                 └────────────────────────┘                 │
│                 CIRCULAR WAIT = DEADLOCK                   │
└──────────────────────────────────────────────────────────┘
```

**Fix: Always acquire locks in the same order**

```go
func goroutine1() {
    mu1.Lock() // always mu1 first
    mu2.Lock()
    // ...
    mu2.Unlock()
    mu1.Unlock()
}

func goroutine2() {
    mu1.Lock() // always mu1 first (same order!)
    mu2.Lock()
    // ...
    mu2.Unlock()
    mu1.Unlock()
}
```

### Bug 3: Goroutine Leaks

A goroutine leak occurs when a goroutine blocks forever and is never cleaned up. Each leaked goroutine consumes ~2-8 KB of stack memory, and the leak compounds over time.

```go
// BUG: Goroutine leak -- channel never read
func leakyFunction() {
    ch := make(chan int)
    go func() {
        result := expensiveComputation()
        ch <- result // BLOCKS FOREVER if nobody reads from ch
    }()
    // ... caller returns without reading ch
    // The goroutine is now leaked!
}

// BUG: Goroutine leak -- context not used
func leakyServer(ctx context.Context) {
    ch := make(chan string)
    go func() {
        for {
            data := fetchData() // what if ctx is cancelled?
            ch <- data          // blocks forever if consumer stops
        }
    }()
    // ...
}
```

**Fix: Always provide an exit path via context or done channel**

```go
func properFunction(ctx context.Context) (int, error) {
    ch := make(chan int, 1) // buffered! goroutine won't block even if we don't read
    go func() {
        result := expensiveComputation()
        select {
        case ch <- result:
        case <-ctx.Done():
            // caller cancelled -- exit cleanly
        }
    }()

    select {
    case result := <-ch:
        return result, nil
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

### Bug 4: Channel Misuse

```go
// BUG: Sending on closed channel (PANIC!)
ch := make(chan int)
close(ch)
ch <- 1 // panic: send on closed channel

// BUG: Closing a channel twice (PANIC!)
ch := make(chan int)
close(ch)
close(ch) // panic: close of closed channel

// BUG: Reading from nil channel blocks forever
var ch chan int // nil
<-ch // blocks forever, no error, no panic, just... frozen

// FIX: Always initialize channels
ch := make(chan int)
```

**Channel closing rules:**
1. Only the **sender** (producer) closes a channel, never the receiver.
2. If multiple goroutines send on a channel, use a `sync.WaitGroup` to close it after all senders are done.
3. Never close a channel from the receiver side.

```go
// Safe pattern: multiple senders, one close
func multipleSenders() {
    ch := make(chan int)
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ch <- id
        }(i)
    }

    // Dedicated goroutine waits for all senders, then closes
    go func() {
        wg.Wait()
        close(ch)
    }()

    for val := range ch {
        fmt.Println(val)
    }
}
```

### Bug 5: Mutex Copy

```go
// BUG: Copying a mutex
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c SafeCounter) Increment() { // VALUE receiver -- copies the struct including the mutex!
    c.mu.Lock()   // locks the COPY, not the original
    c.count++
    c.mu.Unlock()
}

// FIX: Use pointer receiver
func (c *SafeCounter) Increment() { // POINTER receiver -- no copy
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}
```

> **Detection tip:** `go vet` catches mutex copies. Always run `go vet ./...` in CI.

### Bug 6: Premature WaitGroup Done

```go
// BUG: wg.Done() may not execute if goroutine panics or returns early
var wg sync.WaitGroup
wg.Add(1)
go func() {
    if condition {
        return // OOPS: wg.Done() never called -- wg.Wait() blocks forever
    }
    // ... work ...
    wg.Done()
}()
wg.Wait()

// FIX: Always use defer
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done() // guaranteed to run
    if condition {
        return
    }
    // ... work ...
}()
wg.Wait()
```

### Summary of Detection Tools

| Bug Type | Detection Tool | Command |
|----------|---------------|---------|
| Data races | Race detector | `go test -race ./...` |
| Mutex copy | go vet | `go vet ./...` |
| Goroutine leaks | Runtime monitoring | `runtime.NumGoroutine()` |
| Goroutine leaks | goleak (Uber) | `goleak.VerifyNone(t)` in tests |
| Deadlocks | go-deadlock (sasha-s) | Drop-in `sync.Mutex` replacement |
| All of the above | Profile | `pprof` goroutine profile |

---

## 12. Decision Tree: Which Concurrency Primitive?

```
┌──────────────────────────────────────────────────────────────────┐
│          DECISION TREE: Choosing the Right Primitive              │
│                                                                    │
│  Do you need to pass data between goroutines?                      │
│  ├─ YES → Use a channel                                           │
│  │        ├─ One sender, one receiver? → unbuffered or small buf  │
│  │        ├─ Multiple senders? → channel + WaitGroup to close     │
│  │        ├─ Broadcast to all? → close(ch) as signal              │
│  │        └─ Need backpressure? → buffered channel                │
│  │                                                                 │
│  └─ NO → Do you need to protect shared state?                     │
│          ├─ YES                                                    │
│          │   ├─ Single variable, simple op? → sync/atomic          │
│          │   ├─ Read-heavy config? → atomic.Value or atomic.Ptr    │
│          │   ├─ Multiple fields together? → sync.Mutex             │
│          │   ├─ Many readers, few writers? → sync.RWMutex          │
│          │   └─ Map with specific patterns? → sync.Map             │
│          │                                                         │
│          └─ NO → Do you need to coordinate goroutines?             │
│                  ├─ Wait for N goroutines? → sync.WaitGroup        │
│                  ├─ Wait with errors? → errgroup                   │
│                  ├─ Limit concurrency? → errgroup.SetLimit or      │
│                  │                       semaphore                  │
│                  ├─ Deduplicate work? → singleflight               │
│                  ├─ Rate limit? → x/time/rate                      │
│                  ├─ Run once? → sync.Once                          │
│                  ├─ Complex wait condition? → sync.Cond            │
│                  ├─ Reuse objects? → sync.Pool                     │
│                  └─ Cancel propagation? → context.Context          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 13. Go vs Node.js Concurrency Comparison

### sync.Pool vs Node.js generic-pool

```
┌─────────────────────────────────────────────────────────┐
│  Go: sync.Pool                  │  Node.js: generic-pool │
├─────────────────────────────────┼────────────────────────┤
│  Built-in, zero dependencies    │  npm package           │
│  GC can evict entries anytime   │  Manual lifecycle      │
│  Per-goroutine (lock-free)      │  Single-threaded       │
│  For reducing GC pressure       │  For managing scarce   │
│                                 │  resources (DB conns)  │
│  pool := sync.Pool{             │  const pool =          │
│    New: func() any {            │    genericPool.        │
│      return new(bytes.Buffer)   │    createPool({        │
│    },                           │      create: () =>     │
│  }                              │        createConn(),   │
│  buf := pool.Get()              │      destroy: (c) =>   │
│  pool.Put(buf)                  │        c.close(),      │
│                                 │      max: 10           │
│                                 │    });                 │
│                                 │  const c = await       │
│                                 │    pool.acquire();     │
│                                 │  pool.release(c);      │
├─────────────────────────────────┴────────────────────────┤
│  KEY DIFFERENCE: Go's sync.Pool is a soft cache for GC   │
│  optimization. Node.js pools manage hard resource limits  │
│  (connections, workers). Different problems, similar API. │
└──────────────────────────────────────────────────────────┘
```

### Atomic Operations

```
┌─────────────────────────────────────────────────────────┐
│  Go: sync/atomic                │  Node.js: Atomics      │
├─────────────────────────────────┼────────────────────────┤
│  Works across goroutines        │  Works across Workers   │
│  (shared memory by default)     │  (SharedArrayBuffer)    │
│                                 │                         │
│  var counter atomic.Int64       │  const sab = new        │
│  counter.Add(1)                 │    SharedArrayBuffer(8);│
│  val := counter.Load()          │  const arr = new        │
│                                 │    Int32Array(sab);     │
│                                 │  Atomics.add(arr, 0, 1);│
│                                 │  Atomics.load(arr, 0);  │
├─────────────────────────────────┴────────────────────────┤
│  KEY DIFFERENCE: Go goroutines share memory naturally,    │
│  so atomics are common. Node.js workers are isolated by   │
│  default; Atomics are only needed with SharedArrayBuffer, │
│  which is uncommon in typical Node.js applications.       │
└──────────────────────────────────────────────────────────┘
```

### errgroup vs Promise.allSettled

```go
// Go: errgroup -- fails fast, cancels remaining on first error
g, ctx := errgroup.WithContext(ctx)
for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetch(ctx, url)
    })
}
err := g.Wait() // first error, or nil
```

```javascript
// Node.js: Promise.allSettled -- runs ALL to completion
const results = await Promise.allSettled(
  urls.map(url => fetch(url))
);
// results: [{status: 'fulfilled', value: ...}, {status: 'rejected', reason: ...}]
// No early cancellation! All promises run to completion.

// Node.js equivalent of errgroup (fail-fast):
const controller = new AbortController();
try {
  await Promise.all(
    urls.map(url =>
      fetch(url, { signal: controller.signal })
    )
  );
} catch (err) {
  controller.abort(); // cancel remaining requests
  throw err;
}
```

```
┌─────────────────────────────────────────────────────────┐
│  Go errgroup                    │  Node.js equivalents    │
├─────────────────────────────────┼─────────────────────────┤
│  g.Wait() → first error        │  Promise.all → first    │
│  ctx auto-cancelled             │  rejection (no cancel)  │
│  g.SetLimit(n) for concurrency  │  p-limit package        │
│                                 │                         │
│  No Promise.allSettled equiv    │  Promise.allSettled     │
│  in stdlib (collect all errors  │  collects all results   │
│  requires manual implementation)│  regardless of errors   │
├─────────────────────────────────┴─────────────────────────┤
│  KEY DIFFERENCE: Go errgroup propagates cancellation via   │
│  context. Node.js Promise.all rejects on first error but   │
│  does NOT cancel other promises (no built-in mechanism).   │
│  AbortController is opt-in and manual.                     │
└──────────────────────────────────────────────────────────┘
```

### singleflight vs Node.js Request Deduplication

```go
// Go: built-in singleflight package
var group singleflight.Group

func getUser(id string) (*User, error) {
    val, err, _ := group.Do(id, func() (interface{}, error) {
        return db.QueryUser(id)
    })
    return val.(*User), err
}
```

```javascript
// Node.js: no built-in equivalent -- must implement manually
const inFlight = new Map();

async function getUser(id) {
  if (inFlight.has(id)) {
    return inFlight.get(id); // return existing promise
  }
  const promise = db.queryUser(id).finally(() => {
    inFlight.delete(id);
  });
  inFlight.set(id, promise);
  return promise;
}
```

```
┌─────────────────────────────────────────────────────────┐
│  KEY DIFFERENCE: Go's singleflight is a well-tested,     │
│  community-standard package. Node.js requires custom      │
│  implementation using a Map of Promises -- easy to get    │
│  wrong (error handling, cleanup, memory leaks).           │
│                                                           │
│  Both leverage the same insight: if multiple callers      │
│  want the same result, only compute it once.              │
└──────────────────────────────────────────────────────────┘
```

### Rate Limiting

```
┌─────────────────────────────────────────────────────────┐
│  Go: x/time/rate               │  Node.js:               │
│                                │  rate-limiter-flexible   │
├────────────────────────────────┼─────────────────────────┤
│  Token bucket, stdlib-adjacent │  Multiple strategies     │
│  In-process only               │  Redis-backed option     │
│                                │                          │
│  limiter := rate.NewLimiter(   │  const limiter = new     │
│    10, 20)                     │    RateLimiterMemory({   │
│  limiter.Wait(ctx)             │      points: 10,         │
│                                │      duration: 1         │
│                                │    });                   │
│                                │  await limiter.consume(  │
│                                │    key);                 │
├────────────────────────────────┴─────────────────────────┤
│  KEY DIFFERENCE: Go's rate limiter is per-process. For    │
│  distributed systems, both languages need external stores │
│  (Redis). Node.js rate-limiter-flexible has built-in      │
│  Redis/Cluster support. Go requires a separate library    │
│  or custom implementation for distributed rate limiting.  │
└──────────────────────────────────────────────────────────┘
```

### Goroutine Leaks vs Node.js Memory Leaks

```
┌─────────────────────────────────────────────────────────┐
│  Go Goroutine Leaks             │  Node.js Memory Leaks   │
├─────────────────────────────────┼─────────────────────────┤
│  Blocked goroutines (~2-8KB     │  Unreleased event       │
│  each, accumulate over time)    │  listeners, unclosed    │
│                                 │  streams, closures      │
│                                 │  holding references     │
│                                 │                         │
│  Detection:                     │  Detection:             │
│  - runtime.NumGoroutine()       │  - process.memoryUsage()│
│  - pprof goroutine profile      │  - --inspect + Chrome   │
│  - uber-go/goleak in tests      │    DevTools heap snap   │
│                                 │  - clinic.js            │
│                                 │                         │
│  Common causes:                 │  Common causes:         │
│  - Blocked channel send/recv    │  - emitter.on() without │
│  - Missing ctx.Done() check     │    emitter.off()        │
│  - Infinite loop without exit   │  - Unclosed streams     │
│  - Forgotten timer/ticker       │  - Large closures in    │
│                                 │    callbacks            │
│                                 │  - Global caches with   │
│                                 │    no eviction          │
├─────────────────────────────────┴─────────────────────────┤
│  KEY DIFFERENCE: Go leaks goroutines (lightweight but     │
│  accumulate). Node.js leaks heap memory (event listeners, │
│  closures). Both are insidious; both need monitoring.     │
│  Go has the race detector; Node.js has nothing comparable.│
└──────────────────────────────────────────────────────────┘
```

### Race Detector -- Go's Killer Feature

```
┌─────────────────────────────────────────────────────────┐
│                  Race Detector Comparison                 │
│                                                           │
│  Go:                                                      │
│  $ go test -race ./...                                    │
│  - Built into the toolchain                               │
│  - Instruments every memory access at compile time         │
│  - Near-zero false positives                               │
│  - ~2-10x slowdown (acceptable for tests and staging)     │
│  - Detects actual data races at runtime                    │
│                                                           │
│  Node.js:                                                 │
│  - No equivalent                                          │
│  - Single-threaded event loop avoids most races            │
│  - Worker threads + SharedArrayBuffer can race, but       │
│    no tooling detects it                                  │
│  - Must rely on code review and careful design             │
│                                                           │
│  This is one of Go's strongest advantages for              │
│  concurrent programming.                                   │
└──────────────────────────────────────────────────────────┘
```

---

## 14. Key Takeaways

1. **sync.Pool** is a GC optimization tool, not a resource manager. Objects can be evicted at any GC cycle. Always reset borrowed objects before use, and don't return oversized objects.

2. **sync/atomic** is for simple, single-variable operations where mutex overhead dominates. Use the typed wrappers (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) from Go 1.19+. Use CAS loops for lock-free algorithms.

3. **sync.Cond** fills a narrow niche: multiple goroutines waiting for a complex condition with Broadcast capability. For most cases, channels and context are simpler and safer.

4. **errgroup** replaces the "WaitGroup + mutex + error variable" boilerplate. Use `WithContext` for automatic cancellation on first error. Use `SetLimit` to bound concurrency.

5. **semaphore** provides weighted concurrency control. Different operations can consume different amounts of capacity from a shared pool.

6. **singleflight** prevents thundering herds by deduplicating concurrent calls for the same key. Essential for caches and API gateways.

7. **Pipelines** compose concurrent stages via channels. Every stage must check `ctx.Done()` on every send to prevent goroutine leaks on cancellation.

8. **Fan-out/Fan-in** distributes work across multiple goroutines and merges results. Buffered channels provide natural backpressure.

9. **Rate limiting** with `x/time/rate` implements a token bucket. Use per-client limiters for API rate limiting. For distributed systems, you need an external store.

10. **Concurrent data structures** like ShardedMap reduce contention by splitting state across independent locks. Lock-free structures use CAS but are hard to get right -- prefer mutexes unless profiling proves otherwise.

11. **The race detector (`go test -race`)** is non-negotiable. Run it in CI on every commit. It has near-zero false positives. If it fires, fix the bug.

12. **Goroutine leak prevention:** every goroutine must have a guaranteed exit path. Every channel operation in a goroutine should be wrapped in `select` with `ctx.Done()` or a done channel.

---

## 15. Practice Exercises

### Exercise 1: Build a Caching Layer with singleflight

Build an in-memory cache with:
- TTL-based expiration
- singleflight protection against thundering herds
- Concurrent-safe access (sync.RWMutex or sync.Map)
- A `GetOrFetch(key string, fetcher func() (any, error)) (any, error)` method

Test it by simulating 100 concurrent requests for the same key and verifying the fetcher runs only once.

### Exercise 2: Rate-Limited Worker Pool

Build a worker pool that:
- Accepts jobs via a channel
- Processes at most N jobs concurrently (semaphore)
- Rate-limits to R jobs per second (rate.Limiter)
- Supports graceful shutdown via context cancellation
- Collects results and errors via errgroup

### Exercise 3: Concurrent Pipeline with Error Handling

Build a 4-stage pipeline:
1. **Read:** Read lines from a file (or simulate with a slice)
2. **Parse:** Parse each line into a struct (some lines are malformed -- return errors)
3. **Validate:** Validate the struct (some fail validation)
4. **Store:** Simulate storing valid structs

Requirements:
- Use the `Result[T]` pattern to propagate errors through the pipeline
- Support context cancellation at every stage
- Count successes, parse errors, and validation errors
- No goroutine leaks (verify with `runtime.NumGoroutine()` before and after)

### Exercise 4: Detect and Fix Concurrency Bugs

The following code has multiple concurrency bugs. Find and fix them all:

```go
package main

import (
    "fmt"
    "sync"
)

type BankAccount struct {
    balance int
    mu      sync.Mutex
}

func (a BankAccount) Deposit(amount int) {
    a.mu.Lock()
    a.balance += amount
    a.mu.Unlock()
}

func (a BankAccount) Balance() int {
    return a.balance
}

func transfer(from, to *BankAccount, amount int) {
    from.mu.Lock()
    to.mu.Lock()
    from.balance -= amount
    to.balance += amount
    to.mu.Unlock()
    from.mu.Unlock()
}

func main() {
    a := BankAccount{balance: 1000}
    b := BankAccount{balance: 1000}

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(2)
        go func() {
            transfer(&a, &b, 10)
            wg.Done()
        }()
        go func() {
            transfer(&b, &a, 10)
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Printf("A: %d, B: %d, Total: %d\n", a.Balance(), b.Balance(), a.Balance()+b.Balance())
}
```

**Bugs to find:**
1. Value receiver on `Deposit` -- locks a copy of the mutex
2. `Balance()` reads without lock -- data race
3. `transfer()` locks in inconsistent order -- deadlock risk (A->B vs B->A)
4. `wg.Done()` not deferred -- risk of missing Done on panic

### Exercise 5: sync.Pool Benchmark

Write a benchmark comparing:
1. Allocating a `[]byte` slice of 4KB on every iteration
2. Using `sync.Pool` to reuse `[]byte` slices

Run with `go test -bench=. -benchmem` and analyze:
- ns/op
- B/op
- allocs/op

Then run with `GOGC=10` (aggressive GC) and observe how the pool advantage changes under GC pressure.

---

> **Next Chapter:** Chapter 18 will cover Testing and Benchmarking in Go -- table-driven tests, mocks, test fixtures, `testing.B` for benchmarks, `pprof` for profiling, and fuzz testing.
