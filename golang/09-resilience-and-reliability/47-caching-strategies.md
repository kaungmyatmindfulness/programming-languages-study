# Chapter 47: Caching Strategies in Go

## Table of Contents

1. [Why Caching Matters](#1-why-caching-matters)
2. [In-Memory Caching with sync.Map](#2-in-memory-caching-with-syncmap)
3. [Building an LRU Cache from Scratch](#3-building-an-lru-cache-from-scratch)
4. [Popular Caching Libraries](#4-popular-caching-libraries)
5. [Cache Eviction Policies](#5-cache-eviction-policies)
6. [Redis as an External Cache](#6-redis-as-an-external-cache)
7. [Caching Patterns](#7-caching-patterns)
8. [Cache Invalidation Strategies](#8-cache-invalidation-strategies)
9. [Thundering Herd / Cache Stampede Prevention](#9-thundering-herd--cache-stampede-prevention)
10. [Multi-Tier Caching](#10-multi-tier-caching)
11. [Cache Warming and Preloading](#11-cache-warming-and-preloading)
12. [Distributed Caching with Consistent Hashing](#12-distributed-caching-with-consistent-hashing)
13. [HTTP Caching Headers](#13-http-caching-headers)
14. [Monitoring Cache Hit Rates and Performance](#14-monitoring-cache-hit-rates-and-performance)
15. [Common Pitfalls](#15-common-pitfalls)
16. [Summary](#16-summary)

---

## 1. Why Caching Matters

Caching is one of the most impactful performance optimizations you can make in any system. At its core, a cache stores the results of expensive computations or data fetches so subsequent requests can be served faster and cheaper.

### 1.1 Latency Reduction

Every network round-trip to a database or external service introduces latency. A typical PostgreSQL query over a local network takes 1-5ms. A Redis lookup takes 0.1-0.5ms. An in-memory cache lookup takes 50-200 nanoseconds. The difference is orders of magnitude.

```
Operation                     | Typical Latency
------------------------------|----------------
In-memory map lookup          | ~50-200ns
Local Redis GET               | ~0.1-0.5ms
PostgreSQL simple query       | ~1-5ms
External API call             | ~50-500ms
Cross-region database query   | ~50-150ms
```

### 1.2 Throughput Improvement

Caching directly increases the number of requests your system can handle. If 90% of requests hit the cache, your database only needs to handle 10% of the traffic. This means a system that can serve 1,000 requests/second from the database can effectively serve 10,000 requests/second with caching.

### 1.3 Cost Reduction

Database instances, API calls, and compute resources all cost money. Caching reduces the load on these expensive resources. Consider:

- A managed database costs $500/month and handles 5,000 QPS
- Adding a Redis cache for $50/month can absorb 90% of reads
- You effectively get 50,000 QPS capacity for $550/month instead of needing $5,000/month in database scaling

### 1.4 When NOT to Cache

Caching is not always the right answer. Avoid caching when:

- Data changes very frequently and staleness is unacceptable
- Each request is unique (cache hit rate would be near zero)
- The data set is so large it would exhaust memory
- The source of truth is already fast enough
- Consistency requirements are strict (financial transactions)

> **Tip:** The two hardest problems in computer science are cache invalidation, naming things, and off-by-one errors. Caching adds complexity. Only cache when the performance benefit justifies it.

---

## 2. In-Memory Caching with sync.Map

Go's `sync.Map` is a concurrent-safe map provided by the standard library. It is optimized for two specific use cases: (1) when the entry for a given key is only ever written once but read many times, and (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys.

### 2.1 Basic Usage

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// SimpleCache wraps sync.Map with expiration support.
type SimpleCache struct {
    data sync.Map
}

type cacheEntry struct {
    value     interface{}
    expiresAt time.Time
}

func NewSimpleCache() *SimpleCache {
    return &SimpleCache{}
}

func (c *SimpleCache) Set(key string, value interface{}, ttl time.Duration) {
    c.data.Store(key, cacheEntry{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    })
}

func (c *SimpleCache) Get(key string) (interface{}, bool) {
    raw, ok := c.data.Load(key)
    if !ok {
        return nil, false
    }

    entry := raw.(cacheEntry)
    if time.Now().After(entry.expiresAt) {
        c.data.Delete(key)
        return nil, false
    }

    return entry.value, true
}

func (c *SimpleCache) Delete(key string) {
    c.data.Delete(key)
}

func main() {
    cache := NewSimpleCache()

    // Store a value with 5-second TTL
    cache.Set("user:123", map[string]string{
        "name":  "Alice",
        "email": "alice@example.com",
    }, 5*time.Second)

    // Retrieve it
    if val, ok := cache.Get("user:123"); ok {
        fmt.Printf("Found: %v\n", val)
    }

    // Wait for expiration
    time.Sleep(6 * time.Second)

    if _, ok := cache.Get("user:123"); !ok {
        fmt.Println("Entry expired")
    }
}
```

### 2.2 Periodic Cleanup of Expired Entries

A key weakness of the above approach is that expired entries linger in memory until they are accessed. You need a background goroutine to clean up stale entries.

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type TTLCache struct {
    data    sync.Map
    stopCh  chan struct{}
}

type ttlEntry struct {
    value     interface{}
    expiresAt time.Time
}

func NewTTLCache(cleanupInterval time.Duration) *TTLCache {
    c := &TTLCache{
        stopCh: make(chan struct{}),
    }
    go c.startCleanup(cleanupInterval)
    return c
}

func (c *TTLCache) startCleanup(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            now := time.Now()
            c.data.Range(func(key, value interface{}) bool {
                entry := value.(ttlEntry)
                if now.After(entry.expiresAt) {
                    c.data.Delete(key)
                }
                return true // continue iteration
            })
        case <-c.stopCh:
            return
        }
    }
}

func (c *TTLCache) Set(key string, value interface{}, ttl time.Duration) {
    c.data.Store(key, ttlEntry{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    })
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
    raw, ok := c.data.Load(key)
    if !ok {
        return nil, false
    }
    entry := raw.(ttlEntry)
    if time.Now().After(entry.expiresAt) {
        c.data.Delete(key)
        return nil, false
    }
    return entry.value, true
}

func (c *TTLCache) Stop() {
    close(c.stopCh)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    cache := NewTTLCache(1 * time.Second) // cleanup every second
    defer cache.Stop()

    cache.Set("session:abc", "user-data-here", 3*time.Second)

    _ = ctx // use ctx for your application logic
    fmt.Println("Cache running with periodic cleanup...")
}
```

### 2.3 Limitations of sync.Map

| Aspect             | sync.Map Behavior                                       |
|--------------------|---------------------------------------------------------|
| Size bounding      | None -- grows unbounded unless you manually evict       |
| Eviction policy    | None built-in                                           |
| TTL support        | Must be implemented manually                            |
| Iteration          | `Range` is O(n) and holds no lock guarantees on snapshot|
| Type safety        | Stores `interface{}` -- no generics support natively     |
| Memory overhead    | Higher per-entry overhead than a plain map + RWMutex    |

> **Warning:** `sync.Map` is not a general-purpose concurrent map. For most caching use cases, a `map` protected by `sync.RWMutex` or a purpose-built library is more appropriate. `sync.Map` shines only when keys are stable or disjoint across goroutines.

### 2.4 Generic Cache with RWMutex (Go 1.18+)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]cacheItem[V]
}

type cacheItem[V any] struct {
    value     V
    expiresAt time.Time
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{
        items: make(map[K]cacheItem[V]),
    }
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = cacheItem[V]{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    item, found := c.items[key]
    if !found {
        var zero V
        return zero, false
    }

    if time.Now().After(item.expiresAt) {
        var zero V
        return zero, false
    }

    return item.value, true
}

func (c *Cache[K, V]) Delete(key K) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *Cache[K, V]) Len() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.items)
}

func main() {
    // Type-safe cache: string keys, int values
    cache := NewCache[string, int]()
    cache.Set("counter", 42, 10*time.Second)

    if val, ok := cache.Get("counter"); ok {
        fmt.Printf("counter = %d\n", val) // counter = 42
    }
}
```

---

## 3. Building an LRU Cache from Scratch

An LRU (Least Recently Used) cache evicts the entry that was accessed least recently when the cache reaches its capacity. This is one of the most commonly used eviction strategies in production systems.

### 3.1 Data Structure Design

An efficient LRU cache requires two data structures working together:

1. **Hash map** for O(1) key lookups
2. **Doubly linked list** for O(1) insertion, deletion, and reordering

```
Hash Map                 Doubly Linked List (most recent -> least recent)
+---------+
| key1  --|----------->  [key1|val1] <-> [key3|val3] <-> [key2|val2]
| key2  --|----------->       ^                               ^
| key3  --|----------->       |                               |
+---------+              most recent                    least recent
                                                        (evict this)
```

### 3.2 Full Implementation

```go
package main

import (
    "container/list"
    "fmt"
    "sync"
)

// LRUCache is a thread-safe LRU cache.
type LRUCache struct {
    mu       sync.Mutex
    capacity int
    items    map[string]*list.Element
    order    *list.List // front = most recent, back = least recent

    // Metrics
    hits   int64
    misses int64
}

type entry struct {
    key   string
    value interface{}
}

// NewLRUCache creates a new LRU cache with the given maximum capacity.
func NewLRUCache(capacity int) *LRUCache {
    if capacity <= 0 {
        panic("LRU cache capacity must be positive")
    }
    return &LRUCache{
        capacity: capacity,
        items:    make(map[string]*list.Element),
        order:    list.New(),
    }
}

// Get retrieves a value from the cache.
// Returns the value and true if found, or nil and false if not.
func (c *LRUCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    elem, found := c.items[key]
    if !found {
        c.misses++
        return nil, false
    }

    c.hits++
    // Move to front (most recently used)
    c.order.MoveToFront(elem)
    return elem.Value.(*entry).value, true
}

// Put adds or updates a value in the cache.
// If the cache is full, the least recently used entry is evicted.
func (c *LRUCache) Put(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    // If key already exists, update it and move to front
    if elem, found := c.items[key]; found {
        c.order.MoveToFront(elem)
        elem.Value.(*entry).value = value
        return
    }

    // If at capacity, evict the least recently used entry
    if c.order.Len() >= c.capacity {
        c.evict()
    }

    // Add new entry at front
    e := &entry{key: key, value: value}
    elem := c.order.PushFront(e)
    c.items[key] = elem
}

// evict removes the least recently used entry. Caller must hold the lock.
func (c *LRUCache) evict() {
    tail := c.order.Back()
    if tail == nil {
        return
    }
    c.order.Remove(tail)
    e := tail.Value.(*entry)
    delete(c.items, e.key)
}

// Delete removes an entry from the cache.
func (c *LRUCache) Delete(key string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()

    elem, found := c.items[key]
    if !found {
        return false
    }
    c.order.Remove(elem)
    delete(c.items, key)
    return true
}

// Len returns the current number of entries in the cache.
func (c *LRUCache) Len() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.order.Len()
}

// HitRate returns the cache hit rate as a percentage.
func (c *LRUCache) HitRate() float64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    total := c.hits + c.misses
    if total == 0 {
        return 0
    }
    return float64(c.hits) / float64(total) * 100
}

// Keys returns all keys in order from most to least recently used.
func (c *LRUCache) Keys() []string {
    c.mu.Lock()
    defer c.mu.Unlock()

    keys := make([]string, 0, c.order.Len())
    for elem := c.order.Front(); elem != nil; elem = elem.Next() {
        keys = append(keys, elem.Value.(*entry).key)
    }
    return keys
}

func main() {
    cache := NewLRUCache(3)

    cache.Put("a", 1)
    cache.Put("b", 2)
    cache.Put("c", 3)

    fmt.Println("Keys after adding a, b, c:", cache.Keys())
    // Output: [c b a]  (c is most recent)

    // Access "a" to make it most recent
    cache.Get("a")
    fmt.Println("Keys after accessing a:", cache.Keys())
    // Output: [a c b]

    // Adding "d" should evict "b" (least recently used)
    cache.Put("d", 4)
    fmt.Println("Keys after adding d:", cache.Keys())
    // Output: [d a c]

    _, found := cache.Get("b")
    fmt.Println("b found:", found) // false -- it was evicted

    fmt.Printf("Hit rate: %.1f%%\n", cache.HitRate())
}
```

### 3.3 LRU Cache with TTL

In practice, you often want both size-based eviction (LRU) and time-based expiration (TTL). Here is an extended version.

```go
package main

import (
    "container/list"
    "fmt"
    "sync"
    "time"
)

type LRUTTLCache struct {
    mu       sync.Mutex
    capacity int
    items    map[string]*list.Element
    order    *list.List
    defaultTTL time.Duration
}

type ttlLRUEntry struct {
    key       string
    value     interface{}
    expiresAt time.Time
}

func NewLRUTTLCache(capacity int, defaultTTL time.Duration) *LRUTTLCache {
    c := &LRUTTLCache{
        capacity:   capacity,
        items:      make(map[string]*list.Element),
        order:      list.New(),
        defaultTTL: defaultTTL,
    }
    return c
}

func (c *LRUTTLCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    elem, found := c.items[key]
    if !found {
        return nil, false
    }

    e := elem.Value.(*ttlLRUEntry)

    // Check TTL
    if time.Now().After(e.expiresAt) {
        c.removeElement(elem)
        return nil, false
    }

    c.order.MoveToFront(elem)
    return e.value, true
}

func (c *LRUTTLCache) Put(key string, value interface{}) {
    c.PutWithTTL(key, value, c.defaultTTL)
}

func (c *LRUTTLCache) PutWithTTL(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if elem, found := c.items[key]; found {
        c.order.MoveToFront(elem)
        e := elem.Value.(*ttlLRUEntry)
        e.value = value
        e.expiresAt = time.Now().Add(ttl)
        return
    }

    // Evict expired entries first, then evict LRU if still at capacity
    c.evictExpired()
    if c.order.Len() >= c.capacity {
        c.evictLRU()
    }

    e := &ttlLRUEntry{
        key:       key,
        value:     value,
        expiresAt: time.Now().Add(ttl),
    }
    elem := c.order.PushFront(e)
    c.items[key] = elem
}

func (c *LRUTTLCache) evictExpired() {
    now := time.Now()
    for elem := c.order.Back(); elem != nil; {
        prev := elem.Prev()
        e := elem.Value.(*ttlLRUEntry)
        if now.After(e.expiresAt) {
            c.removeElement(elem)
        }
        elem = prev
    }
}

func (c *LRUTTLCache) evictLRU() {
    tail := c.order.Back()
    if tail != nil {
        c.removeElement(tail)
    }
}

func (c *LRUTTLCache) removeElement(elem *list.Element) {
    c.order.Remove(elem)
    e := elem.Value.(*ttlLRUEntry)
    delete(c.items, e.key)
}

func main() {
    cache := NewLRUTTLCache(100, 5*time.Minute)

    cache.Put("user:1", "Alice")
    cache.PutWithTTL("session:abc", "session-data", 30*time.Minute)

    if val, ok := cache.Get("user:1"); ok {
        fmt.Println("Found:", val)
    }
}
```

### 3.4 Generic LRU Cache (Go 1.18+)

```go
package main

import (
    "container/list"
    "fmt"
    "sync"
)

type GenericLRU[K comparable, V any] struct {
    mu       sync.Mutex
    capacity int
    items    map[K]*list.Element
    order    *list.List
    onEvict  func(key K, value V) // optional callback
}

type kvPair[K comparable, V any] struct {
    key   K
    value V
}

func NewGenericLRU[K comparable, V any](capacity int, onEvict func(K, V)) *GenericLRU[K, V] {
    return &GenericLRU[K, V]{
        capacity: capacity,
        items:    make(map[K]*list.Element, capacity),
        order:    list.New(),
        onEvict:  onEvict,
    }
}

func (c *GenericLRU[K, V]) Get(key K) (V, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem)
        return elem.Value.(*kvPair[K, V]).value, true
    }
    var zero V
    return zero, false
}

func (c *GenericLRU[K, V]) Put(key K, value V) (evictedKey K, evictedVal V, evicted bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem)
        elem.Value.(*kvPair[K, V]).value = value
        return
    }

    if c.order.Len() >= c.capacity {
        tail := c.order.Back()
        if tail != nil {
            c.order.Remove(tail)
            kv := tail.Value.(*kvPair[K, V])
            delete(c.items, kv.key)
            evictedKey = kv.key
            evictedVal = kv.value
            evicted = true
            if c.onEvict != nil {
                c.onEvict(kv.key, kv.value)
            }
        }
    }

    kv := &kvPair[K, V]{key: key, value: value}
    elem := c.order.PushFront(kv)
    c.items[key] = elem
    return
}

func main() {
    cache := NewGenericLRU[string, int](3, func(key string, val int) {
        fmt.Printf("Evicted: %s -> %d\n", key, val)
    })

    cache.Put("x", 10)
    cache.Put("y", 20)
    cache.Put("z", 30)
    cache.Put("w", 40) // evicts "x"
    // Output: Evicted: x -> 10

    if val, ok := cache.Get("y"); ok {
        fmt.Printf("y = %d\n", val)
    }
}
```

---

## 4. Popular Caching Libraries

Go has a rich ecosystem of caching libraries, each designed for different use cases.

### 4.1 Overview Comparison

| Library     | Type        | Eviction    | TTL  | Concurrency     | GC Friendly | Use Case                      |
|-------------|-------------|-------------|------|-----------------|-------------|-------------------------------|
| groupcache  | Distributed | LRU         | No   | Yes             | No          | Read-heavy distributed cache  |
| bigcache    | In-process  | FIFO/TTL    | Yes  | Sharded          | Yes         | High-throughput, large data   |
| ristretto   | In-process  | TinyLFU     | Yes  | Yes             | No          | Best hit ratio                |
| freecache   | In-process  | LRU approx  | Yes  | Segment locked  | Yes         | Zero GC overhead              |
| go-cache    | In-process  | TTL         | Yes  | RWMutex         | No          | Simple TTL cache              |

### 4.2 groupcache

`groupcache` was created by Brad Fitzpatrick (author of memcached) and is designed for read-heavy workloads where cache fills are expensive. It is a distributed cache that eliminates the "thundering herd" problem by design.

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/golang/groupcache"
)

// Simulated database
var database = map[string]string{
    "user:1": "Alice",
    "user:2": "Bob",
    "user:3": "Charlie",
}

func main() {
    // Create a new group (namespace) with 64MB cache limit
    userCache := groupcache.NewGroup("users", 64<<20, groupcache.GetterFunc(
        func(ctx context.Context, key string, dest groupcache.Sink) error {
            // This function is called on cache miss.
            // It is the "source of truth" loader.
            log.Printf("Cache miss for key: %s, loading from database", key)

            value, ok := database[key]
            if !ok {
                return fmt.Errorf("key %q not found", key)
            }

            // Store the value in the cache
            dest.SetString(value)
            return nil
        },
    ))

    // First access -- triggers the getter (cache miss)
    var result groupcache.ByteView
    if err := userCache.Get(context.Background(), "user:1", groupcache.ByteViewSink(&result)); err != nil {
        log.Fatal(err)
    }
    fmt.Println("First call:", result.String()) // "Alice"

    // Second access -- served from cache (no getter call)
    if err := userCache.Get(context.Background(), "user:1", groupcache.ByteViewSink(&result)); err != nil {
        log.Fatal(err)
    }
    fmt.Println("Second call:", result.String()) // "Alice"
}
```

Key characteristics of `groupcache`:
- **No explicit Set/Delete** -- data is populated only through the getter function
- **Deduplication** -- concurrent requests for the same key are collapsed into one load
- **Peer-to-peer** -- in a cluster, nodes can fetch from each other instead of the source
- **Immutable entries** -- once cached, entries cannot be updated (by design)

### 4.3 bigcache

`bigcache` is designed for storing millions of entries with minimal GC overhead. It achieves this by storing entries as byte slices in a pre-allocated byte buffer, keeping pointers off the heap.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/allegro/bigcache/v3"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func main() {
    // Configure bigcache
    config := bigcache.DefaultConfig(10 * time.Minute) // 10 minute TTL
    config.MaxEntriesInWindow = 1000 * 10 * 60         // expected entries in TTL window
    config.MaxEntrySize = 500                           // max entry size in bytes
    config.Shards = 1024                                // number of shards (power of 2)
    config.HardMaxCacheSize = 512                       // max cache size in MB (0 = unlimited)
    config.Verbose = true                               // log information about cache operations

    cache, err := bigcache.New(context.Background(), config)
    if err != nil {
        log.Fatal(err)
    }
    defer cache.Close()

    // Store a user
    user := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    data, _ := json.Marshal(user)

    if err := cache.Set("user:1", data); err != nil {
        log.Fatal(err)
    }

    // Retrieve the user
    raw, err := cache.Get("user:1")
    if err != nil {
        log.Fatal(err)
    }

    var retrieved User
    json.Unmarshal(raw, &retrieved)
    fmt.Printf("Retrieved: %+v\n", retrieved)

    // Stats
    fmt.Printf("Cache length: %d\n", cache.Len())
    fmt.Printf("Cache capacity: %d bytes\n", cache.Capacity())
}
```

> **Tip:** `bigcache` requires you to serialize values to `[]byte`. This means extra marshaling overhead but results in nearly zero GC pressure, making it ideal for caches with millions of entries.

### 4.4 ristretto

`ristretto` from Dgraph uses a TinyLFU admission policy, which provides close-to-optimal hit ratios. It tracks both frequency and recency of access to make smart eviction decisions.

```go
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/dgraph-io/ristretto"
)

func main() {
    cache, err := ristretto.NewCache(&ristretto.Config{
        NumCounters: 1e7,     // 10M counters for tracking frequency
        MaxCost:     1 << 30, // 1GB maximum cache size
        BufferItems: 64,      // number of keys per Get buffer
    })
    if err != nil {
        log.Fatal(err)
    }
    defer cache.Close()

    // Set with cost (cost = size of value in bytes, or any unit you choose)
    cache.Set("key1", "value1", 10) // cost of 10

    // SetWithTTL
    cache.SetWithTTL("session:abc", "user-session-data", 100, 30*time.Minute)

    // Wait for value to be processed (ristretto uses async admission)
    time.Sleep(10 * time.Millisecond)

    // Get
    value, found := cache.Get("key1")
    if found {
        fmt.Println("Found:", value)
    }

    // Delete
    cache.Del("key1")

    // Metrics
    fmt.Printf("Hits: %d\n", cache.Metrics.Hits())
    fmt.Printf("Misses: %d\n", cache.Metrics.Misses())
    fmt.Printf("Hit ratio: %.2f\n", cache.Metrics.Ratio())
}
```

> **Warning:** `ristretto` uses an asynchronous admission policy. After calling `Set`, the value may not be immediately available via `Get`. In tests, you need a small sleep or use `cache.Wait()`. In production, this is rarely an issue since the time between set and subsequent reads is usually sufficient.

### 4.5 freecache

`freecache` is a concurrent cache with zero GC overhead. Unlike `bigcache`, it supports per-entry TTL and approximate LRU eviction.

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"

    "github.com/coocood/freecache"
)

func main() {
    // Create a 100MB cache
    cacheSize := 100 * 1024 * 1024 // 100MB
    cache := freecache.NewCache(cacheSize)

    // Set with TTL in seconds
    key := []byte("product:42")
    value, _ := json.Marshal(map[string]interface{}{
        "name":  "Widget",
        "price": 9.99,
    })

    err := cache.Set(key, value, 300) // 5 minute TTL
    if err != nil {
        log.Fatal(err)
    }

    // Get
    got, err := cache.Get(key)
    if err != nil {
        log.Fatal(err)
    }

    var product map[string]interface{}
    json.Unmarshal(got, &product)
    fmt.Printf("Product: %v\n", product)

    // TTL remaining
    ttl, err := cache.TTL(key)
    if err == nil {
        fmt.Printf("TTL remaining: %d seconds\n", ttl)
    }

    // Stats
    fmt.Printf("Entry count: %d\n", cache.EntryCount())
    fmt.Printf("Hit count: %d\n", cache.HitCount())
    fmt.Printf("Miss count: %d\n", cache.MissCount())
    fmt.Printf("Hit rate: %.2f%%\n", cache.HitRate()*100)
    fmt.Printf("Average access time: %d ns\n", cache.AverageAccessTime())
}
```

### 4.6 Library Selection Guide

Use the following decision tree:

```
Do you need distributed caching across multiple nodes?
├── Yes --> groupcache (or Redis, see Section 6)
└── No
    ├── Do you have millions of entries?
    │   ├── Yes --> Is GC pressure a concern?
    │   │   ├── Yes --> bigcache or freecache
    │   │   └── No --> ristretto (best hit ratio)
    │   └── No
    │       ├── Need best possible hit ratio? --> ristretto
    │       ├── Need per-entry TTL? --> freecache or ristretto
    │       └── Simple TTL cache? --> go-cache or hand-rolled
    └── Need per-entry TTL with zero GC? --> freecache
```

---

## 5. Cache Eviction Policies

When a cache reaches its capacity, it must decide which entries to remove. The choice of eviction policy significantly impacts cache hit rates.

### 5.1 LRU (Least Recently Used)

Evicts the entry that has not been accessed for the longest time.

**Pros:**
- Simple to implement and understand
- Works well for temporal locality patterns

**Cons:**
- A full scan of the data can flush the entire cache (pollution)
- Does not account for frequency of access

```
Access pattern: A B C D A B E
Cache (size 4): [A B C D] -> access A,B -> [A B C D] -> add E -> evict C -> [E A B D]

Wait, let's trace more carefully:
  Add A: [A]
  Add B: [B, A]
  Add C: [C, B, A]
  Add D: [D, C, B, A]
  Access A: [A, D, C, B]
  Access B: [B, A, D, C]
  Add E: evict C (LRU) -> [E, B, A, D]
```

### 5.2 LFU (Least Frequently Used)

Evicts the entry that has been accessed the fewest times.

**Pros:**
- Resistant to scan pollution
- Keeps frequently accessed items in cache

**Cons:**
- Cold start problem: new entries have low frequency and get evicted quickly
- Items that were popular in the past but are no longer needed stay cached

```go
package main

import (
    "container/heap"
    "fmt"
    "sync"
)

// LFUCache implements a Least Frequently Used cache.
type LFUCache struct {
    mu       sync.Mutex
    capacity int
    items    map[string]*lfuEntry
    freqHeap lfuHeap
    counter  int64 // monotonic counter for tie-breaking
}

type lfuEntry struct {
    key       string
    value     interface{}
    frequency int
    lastUsed  int64 // for tie-breaking among same-frequency entries
    index     int   // index in the heap
}

type lfuHeap []*lfuEntry

func (h lfuHeap) Len() int { return len(h) }
func (h lfuHeap) Less(i, j int) bool {
    if h[i].frequency == h[j].frequency {
        return h[i].lastUsed < h[j].lastUsed // older gets evicted first
    }
    return h[i].frequency < h[j].frequency
}
func (h lfuHeap) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
    h[i].index = i
    h[j].index = j
}
func (h *lfuHeap) Push(x interface{}) {
    entry := x.(*lfuEntry)
    entry.index = len(*h)
    *h = append(*h, entry)
}
func (h *lfuHeap) Pop() interface{} {
    old := *h
    n := len(old)
    entry := old[n-1]
    old[n-1] = nil
    entry.index = -1
    *h = old[:n-1]
    return entry
}

func NewLFUCache(capacity int) *LFUCache {
    return &LFUCache{
        capacity: capacity,
        items:    make(map[string]*lfuEntry),
        freqHeap: make(lfuHeap, 0, capacity),
    }
}

func (c *LFUCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    entry, ok := c.items[key]
    if !ok {
        return nil, false
    }

    c.counter++
    entry.frequency++
    entry.lastUsed = c.counter
    heap.Fix(&c.freqHeap, entry.index)
    return entry.value, true
}

func (c *LFUCache) Put(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.capacity <= 0 {
        return
    }

    c.counter++

    if entry, ok := c.items[key]; ok {
        entry.value = value
        entry.frequency++
        entry.lastUsed = c.counter
        heap.Fix(&c.freqHeap, entry.index)
        return
    }

    if len(c.items) >= c.capacity {
        evicted := heap.Pop(&c.freqHeap).(*lfuEntry)
        delete(c.items, evicted.key)
    }

    entry := &lfuEntry{
        key:       key,
        value:     value,
        frequency: 1,
        lastUsed:  c.counter,
    }
    heap.Push(&c.freqHeap, entry)
    c.items[key] = entry
}

func main() {
    cache := NewLFUCache(3)

    cache.Put("a", 1)
    cache.Put("b", 2)
    cache.Put("c", 3)

    // Access "a" and "b" to increase their frequency
    cache.Get("a")
    cache.Get("a")
    cache.Get("b")

    // Add "d" -- should evict "c" (frequency=1, lowest)
    cache.Put("d", 4)

    _, foundC := cache.Get("c")
    _, foundA := cache.Get("a")
    fmt.Println("c found:", foundC) // false
    fmt.Println("a found:", foundA) // true
}
```

### 5.3 TTL (Time to Live)

Entries expire after a fixed duration regardless of access patterns.

**Pros:**
- Guarantees bounded staleness
- Simple to reason about
- Entries are eventually cleaned up

**Cons:**
- Popular entries may expire unnecessarily
- If many entries expire simultaneously, it causes a "thundering herd"

### 5.4 ARC (Adaptive Replacement Cache)

ARC, developed by IBM, dynamically balances between recency (LRU) and frequency (LFU) by maintaining four lists:

1. **T1** -- recently accessed entries (seen once recently)
2. **T2** -- frequently accessed entries (seen at least twice recently)
3. **B1** -- ghost entries evicted from T1 (tracks recency history)
4. **B2** -- ghost entries evicted from T2 (tracks frequency history)

A parameter `p` dynamically adjusts the partition between T1 and T2 based on workload:
- If a ghost hit occurs in B1, recency is more useful -- increase p (give more space to T1)
- If a ghost hit occurs in B2, frequency is more useful -- decrease p (give more space to T2)

```
                   ARC Cache Structure
    ┌─────────────────────────────────────────┐
    │   B1 (ghost)   │  T1   │  T2  │ B2 (ghost) │
    │   recently      │ recent│ freq │ frequently  │
    │   evicted       │       │      │ evicted     │
    └─────────────────────────────────────────┘
                      ←── p ──→
                    (dynamic boundary)
```

> **Tip:** ARC is patented by IBM. If you need ARC-like behavior in production, consider using `ristretto`, which implements TinyLFU -- a similar adaptive policy without patent concerns.

### 5.5 Eviction Policy Comparison

| Policy | Hit Rate (Zipf) | Hit Rate (Scan) | Complexity | Memory Overhead |
|--------|-----------------|-----------------|------------|-----------------|
| LRU    | Good            | Poor            | O(1)       | Low             |
| LFU    | Great           | Great           | O(log n)   | Medium          |
| FIFO   | Fair            | Fair            | O(1)       | Very Low        |
| ARC    | Excellent       | Excellent       | O(1)       | 2x LRU          |
| TinyLFU| Excellent       | Excellent       | O(1)       | Low             |
| Random | Fair            | Fair            | O(1)       | None            |

---

## 6. Redis as an External Cache

Redis is the most popular external caching solution. It provides persistence, replication, pub/sub, and data structures beyond simple key-value pairs. The `go-redis` library is the standard Go client.

### 6.1 Basic Setup and Operations

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func main() {
    ctx := context.Background()

    // Create Redis client
    rdb := redis.NewClient(&redis.Options{
        Addr:         "localhost:6379",
        Password:     "",               // no password
        DB:           0,                // default DB
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolSize:     10,               // connection pool size
        MinIdleConns: 5,                // minimum idle connections
    })
    defer rdb.Close()

    // Ping to verify connection
    if err := rdb.Ping(ctx).Err(); err != nil {
        log.Fatalf("Failed to connect to Redis: %v", err)
    }

    // --- Basic String Operations ---

    // SET with TTL
    err := rdb.Set(ctx, "greeting", "Hello, Redis!", 5*time.Minute).Err()
    if err != nil {
        log.Fatal(err)
    }

    // GET
    val, err := rdb.Get(ctx, "greeting").Result()
    if err == redis.Nil {
        fmt.Println("Key does not exist")
    } else if err != nil {
        log.Fatal(err)
    } else {
        fmt.Println("greeting:", val)
    }

    // --- Caching Structs ---

    product := Product{ID: 42, Name: "Widget", Price: 9.99}
    data, _ := json.Marshal(product)

    err = rdb.Set(ctx, "product:42", data, 10*time.Minute).Err()
    if err != nil {
        log.Fatal(err)
    }

    raw, err := rdb.Get(ctx, "product:42").Bytes()
    if err != nil {
        log.Fatal(err)
    }

    var retrieved Product
    json.Unmarshal(raw, &retrieved)
    fmt.Printf("Product: %+v\n", retrieved)

    // --- Hash Operations (structured data without serialization) ---

    rdb.HSet(ctx, "user:100", map[string]interface{}{
        "name":  "Alice",
        "email": "alice@example.com",
        "age":   30,
    })
    rdb.Expire(ctx, "user:100", 1*time.Hour)

    name, _ := rdb.HGet(ctx, "user:100", "name").Result()
    fmt.Println("User name:", name)

    allFields, _ := rdb.HGetAll(ctx, "user:100").Result()
    fmt.Printf("User fields: %v\n", allFields)

    // --- Check TTL ---
    ttl, _ := rdb.TTL(ctx, "product:42").Result()
    fmt.Printf("TTL remaining: %v\n", ttl)
}
```

### 6.2 Redis Cache Helper with Generics

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

// RedisCache provides type-safe caching operations.
type RedisCache[T any] struct {
    client *redis.Client
    prefix string
    ttl    time.Duration
}

func NewRedisCache[T any](client *redis.Client, prefix string, ttl time.Duration) *RedisCache[T] {
    return &RedisCache[T]{
        client: client,
        prefix: prefix,
        ttl:    ttl,
    }
}

func (c *RedisCache[T]) key(id string) string {
    return c.prefix + ":" + id
}

func (c *RedisCache[T]) Get(ctx context.Context, id string) (T, error) {
    var result T

    data, err := c.client.Get(ctx, c.key(id)).Bytes()
    if err != nil {
        return result, err
    }

    err = json.Unmarshal(data, &result)
    return result, err
}

func (c *RedisCache[T]) Set(ctx context.Context, id string, value T) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    return c.client.Set(ctx, c.key(id), data, c.ttl).Err()
}

func (c *RedisCache[T]) Delete(ctx context.Context, id string) error {
    return c.client.Del(ctx, c.key(id)).Err()
}

// GetOrLoad attempts to get from cache; on miss, calls loader, caches the result.
func (c *RedisCache[T]) GetOrLoad(ctx context.Context, id string, loader func(ctx context.Context, id string) (T, error)) (T, error) {
    // Try cache first
    result, err := c.Get(ctx, id)
    if err == nil {
        return result, nil
    }
    if err != redis.Nil {
        // Log the error but fall through to the loader
        log.Printf("cache get error for %s: %v", c.key(id), err)
    }

    // Cache miss -- load from source
    result, err = loader(ctx, id)
    if err != nil {
        return result, err
    }

    // Store in cache (non-blocking, best effort)
    if cacheErr := c.Set(ctx, id, result); cacheErr != nil {
        log.Printf("cache set error for %s: %v", c.key(id), cacheErr)
    }

    return result, nil
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()

    ctx := context.Background()

    userCache := NewRedisCache[User](rdb, "user", 15*time.Minute)

    // GetOrLoad: transparently caches database lookups
    user, err := userCache.GetOrLoad(ctx, "1", func(ctx context.Context, id string) (User, error) {
        fmt.Println("Loading user from database...") // only called on cache miss
        return User{ID: 1, Name: "Alice", Email: "alice@example.com"}, nil
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User: %+v\n", user)

    // Second call: served from cache
    user, err = userCache.GetOrLoad(ctx, "1", func(ctx context.Context, id string) (User, error) {
        fmt.Println("This should NOT print on cache hit")
        return User{}, nil
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User (cached): %+v\n", user)
}
```

### 6.3 Redis Pipeline for Bulk Operations

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()

    ctx := context.Background()

    // Pipeline: batch multiple commands into a single round-trip
    pipe := rdb.Pipeline()

    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("item:%d", i)
        pipe.Set(ctx, key, fmt.Sprintf("value-%d", i), 10*time.Minute)
    }

    // Execute all commands in one round-trip
    cmds, err := pipe.Exec(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Executed %d commands in one round-trip\n", len(cmds))

    // Pipeline GET
    pipe = rdb.Pipeline()
    getCmds := make([]*redis.StringCmd, 100)
    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("item:%d", i)
        getCmds[i] = pipe.Get(ctx, key)
    }

    _, err = pipe.Exec(ctx)
    if err != nil {
        log.Fatal(err)
    }

    for i, cmd := range getCmds {
        val, _ := cmd.Result()
        if i < 3 { // just print first few
            fmt.Printf("item:%d = %s\n", i, val)
        }
    }
}
```

### 6.4 Redis Cluster Support

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    rdb := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs: []string{
            "redis-node-1:6379",
            "redis-node-2:6379",
            "redis-node-3:6379",
        },
        Password:     "",
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolSize:     10,

        // Route read-only commands to replicas
        ReadOnly:       true,
        RouteRandomly:  false,
        RouteByLatency: true,
    })
    defer rdb.Close()

    ctx := context.Background()

    err := rdb.Set(ctx, "cluster-key", "cluster-value", 5*time.Minute).Err()
    if err != nil {
        log.Fatal(err)
    }

    val, err := rdb.Get(ctx, "cluster-key").Result()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Cluster value:", val)
}
```

---

## 7. Caching Patterns

There are several established patterns for how a cache interacts with the underlying data store. Choosing the right pattern depends on your read/write ratio, consistency requirements, and tolerance for complexity.

### 7.1 Cache-Aside (Lazy Loading)

The application manages the cache explicitly. On a read, it checks the cache first. On a miss, it fetches from the database, then populates the cache. On a write, it updates the database and invalidates the cache.

```
Read Flow:
  1. App checks cache
  2. Cache HIT  -> return cached value
  3. Cache MISS -> query database -> store in cache -> return value

Write Flow:
  1. App updates database
  2. App invalidates (deletes) cache entry
```

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

type UserRepository struct {
    db    *sql.DB
    cache *redis.Client
    ttl   time.Duration
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func NewUserRepository(db *sql.DB, cache *redis.Client) *UserRepository {
    return &UserRepository{
        db:    db,
        cache: cache,
        ttl:   15 * time.Minute,
    }
}

func (r *UserRepository) cacheKey(id int) string {
    return fmt.Sprintf("user:%d", id)
}

// GetByID implements the cache-aside pattern.
func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    key := r.cacheKey(id)

    // Step 1: Check cache
    data, err := r.cache.Get(ctx, key).Bytes()
    if err == nil {
        var user User
        if json.Unmarshal(data, &user) == nil {
            return &user, nil // Cache HIT
        }
    }
    if err != nil && !errors.Is(err, redis.Nil) {
        log.Printf("cache error (non-fatal): %v", err)
        // Fall through to database on cache errors
    }

    // Step 2: Cache MISS -- load from database
    user, err := r.loadFromDB(ctx, id)
    if err != nil {
        return nil, err
    }

    // Step 3: Populate cache (best effort)
    if encoded, err := json.Marshal(user); err == nil {
        if cacheErr := r.cache.Set(ctx, key, encoded, r.ttl).Err(); cacheErr != nil {
            log.Printf("failed to populate cache: %v", cacheErr)
        }
    }

    return user, nil
}

// Update implements the write path: update DB, then invalidate cache.
func (r *UserRepository) Update(ctx context.Context, user *User) error {
    // Step 1: Update database
    _, err := r.db.ExecContext(ctx,
        "UPDATE users SET name = $1, email = $2 WHERE id = $3",
        user.Name, user.Email, user.ID,
    )
    if err != nil {
        return fmt.Errorf("database update failed: %w", err)
    }

    // Step 2: Invalidate cache
    if err := r.cache.Del(ctx, r.cacheKey(user.ID)).Err(); err != nil {
        log.Printf("cache invalidation failed (non-fatal): %v", err)
        // The entry will expire naturally via TTL
    }

    return nil
}

func (r *UserRepository) loadFromDB(ctx context.Context, id int) (*User, error) {
    user := &User{}
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        return nil, fmt.Errorf("database query failed: %w", err)
    }
    return user, nil
}

func main() {
    // This is a structural example. In production, you'd initialize real
    // *sql.DB and *redis.Client instances here.
    fmt.Println("Cache-aside pattern example")
}
```

> **Warning:** With cache-aside, there is a race condition: if two requests both miss the cache simultaneously, both will query the database and write to the cache. This is generally harmless (both write the same data) but wastes resources. See Section 9 for singleflight to solve this.

### 7.2 Read-Through

The cache itself is responsible for loading data on a miss. The application only interacts with the cache, never directly with the database. This centralizes the loading logic.

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Loader is a function that loads a value from the source of truth.
type Loader[V any] func(ctx context.Context, key string) (V, error)

// ReadThroughCache transparently loads and caches data.
type ReadThroughCache[V any] struct {
    mu     sync.RWMutex
    items  map[string]readThroughEntry[V]
    loader Loader[V]
    ttl    time.Duration
}

type readThroughEntry[V any] struct {
    value     V
    expiresAt time.Time
}

func NewReadThroughCache[V any](loader Loader[V], ttl time.Duration) *ReadThroughCache[V] {
    return &ReadThroughCache[V]{
        items:  make(map[string]readThroughEntry[V]),
        loader: loader,
        ttl:    ttl,
    }
}

// Get retrieves a value from the cache, loading it from the source if necessary.
func (c *ReadThroughCache[V]) Get(ctx context.Context, key string) (V, error) {
    // Check cache under read lock
    c.mu.RLock()
    if entry, ok := c.items[key]; ok && time.Now().Before(entry.expiresAt) {
        c.mu.RUnlock()
        return entry.value, nil
    }
    c.mu.RUnlock()

    // Cache miss -- load from source
    value, err := c.loader(ctx, key)
    if err != nil {
        var zero V
        return zero, err
    }

    // Store in cache
    c.mu.Lock()
    c.items[key] = readThroughEntry[V]{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
    c.mu.Unlock()

    return value, nil
}

// Invalidate removes an entry from the cache, forcing a reload on next access.
func (c *ReadThroughCache[V]) Invalidate(key string) {
    c.mu.Lock()
    delete(c.items, key)
    c.mu.Unlock()
}

func main() {
    // The cache handles all loading -- caller just calls Get.
    cache := NewReadThroughCache[string](
        func(ctx context.Context, key string) (string, error) {
            fmt.Printf("Loading %q from database...\n", key)
            // Simulate DB lookup
            return "loaded-value-for-" + key, nil
        },
        5*time.Minute,
    )

    ctx := context.Background()

    val, _ := cache.Get(ctx, "user:1") // triggers loader
    fmt.Println(val)

    val, _ = cache.Get(ctx, "user:1") // served from cache
    fmt.Println(val)
}
```

### 7.3 Write-Through

On every write, the data is written to both the cache and the database synchronously. This ensures the cache is always consistent with the database but adds latency to writes.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

type WriteThroughCache[V any] struct {
    mu      sync.RWMutex
    items   map[string]writeThroughEntry[V]
    persist func(ctx context.Context, key string, value V) error
    ttl     time.Duration
}

type writeThroughEntry[V any] struct {
    value     V
    expiresAt time.Time
}

func NewWriteThroughCache[V any](
    persist func(ctx context.Context, key string, value V) error,
    ttl time.Duration,
) *WriteThroughCache[V] {
    return &WriteThroughCache[V]{
        items:   make(map[string]writeThroughEntry[V]),
        persist: persist,
        ttl:     ttl,
    }
}

// Set writes to the database first, then updates the cache.
func (c *WriteThroughCache[V]) Set(ctx context.Context, key string, value V) error {
    // Write to database first (source of truth)
    if err := c.persist(ctx, key, value); err != nil {
        return fmt.Errorf("persist failed: %w", err)
    }

    // Update cache
    c.mu.Lock()
    c.items[key] = writeThroughEntry[V]{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
    c.mu.Unlock()

    return nil
}

// Get reads from the cache.
func (c *WriteThroughCache[V]) Get(key string) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, ok := c.items[key]
    if !ok || time.Now().After(entry.expiresAt) {
        var zero V
        return zero, false
    }
    return entry.value, true
}

func main() {
    cache := NewWriteThroughCache[string](
        func(ctx context.Context, key string, value string) error {
            log.Printf("Persisting %s = %s to database", key, value)
            // In production, this would be a real DB write
            return nil
        },
        10*time.Minute,
    )

    ctx := context.Background()

    // Write goes to DB and cache simultaneously
    cache.Set(ctx, "config:theme", "dark")

    // Read served from cache
    if val, ok := cache.Get("config:theme"); ok {
        fmt.Println("Theme:", val)
    }
}
```

### 7.4 Write-Behind (Write-Back)

Writes are applied to the cache immediately and asynchronously flushed to the database in batches. This dramatically improves write throughput but risks data loss if the cache fails before flushing.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

type WriteBehindCache struct {
    mu          sync.RWMutex
    items       map[string]string
    dirty       map[string]string // entries waiting to be flushed
    dirtyMu     sync.Mutex
    flushFunc   func(ctx context.Context, batch map[string]string) error
    flushTicker *time.Ticker
    stopCh      chan struct{}
}

func NewWriteBehindCache(
    flushInterval time.Duration,
    flushFunc func(ctx context.Context, batch map[string]string) error,
) *WriteBehindCache {
    c := &WriteBehindCache{
        items:       make(map[string]string),
        dirty:       make(map[string]string),
        flushFunc:   flushFunc,
        flushTicker: time.NewTicker(flushInterval),
        stopCh:      make(chan struct{}),
    }
    go c.flushLoop()
    return c
}

func (c *WriteBehindCache) Set(key, value string) {
    c.mu.Lock()
    c.items[key] = value
    c.mu.Unlock()

    c.dirtyMu.Lock()
    c.dirty[key] = value
    c.dirtyMu.Unlock()
}

func (c *WriteBehindCache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *WriteBehindCache) flushLoop() {
    for {
        select {
        case <-c.flushTicker.C:
            c.flush()
        case <-c.stopCh:
            c.flush() // Final flush on shutdown
            return
        }
    }
}

func (c *WriteBehindCache) flush() {
    c.dirtyMu.Lock()
    if len(c.dirty) == 0 {
        c.dirtyMu.Unlock()
        return
    }
    // Swap out the dirty map for a new one
    batch := c.dirty
    c.dirty = make(map[string]string)
    c.dirtyMu.Unlock()

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := c.flushFunc(ctx, batch); err != nil {
        log.Printf("Flush failed for %d entries: %v", len(batch), err)
        // Re-queue failed entries
        c.dirtyMu.Lock()
        for k, v := range batch {
            if _, exists := c.dirty[k]; !exists {
                c.dirty[k] = v
            }
        }
        c.dirtyMu.Unlock()
    } else {
        log.Printf("Flushed %d entries to database", len(batch))
    }
}

func (c *WriteBehindCache) Stop() {
    c.flushTicker.Stop()
    close(c.stopCh)
}

func main() {
    cache := NewWriteBehindCache(5*time.Second, func(ctx context.Context, batch map[string]string) error {
        // Simulate batch DB write
        for k, v := range batch {
            fmt.Printf("  DB WRITE: %s = %s\n", k, v)
        }
        return nil
    })
    defer cache.Stop()

    // Writes are instant (cache only)
    cache.Set("counter:page_views", "1042")
    cache.Set("counter:api_calls", "5893")
    cache.Set("counter:errors", "3")

    // Reads are instant too
    if val, ok := cache.Get("counter:page_views"); ok {
        fmt.Println("Page views:", val)
    }

    // Wait for flush
    time.Sleep(6 * time.Second)
}
```

### 7.5 Pattern Comparison

| Pattern       | Read Latency | Write Latency | Consistency | Complexity | Data Loss Risk |
|---------------|-------------|---------------|-------------|------------|----------------|
| Cache-Aside   | Low (hit)   | Normal        | Eventual    | Low        | None           |
| Read-Through  | Low (hit)   | Normal        | Eventual    | Medium     | None           |
| Write-Through | Low (hit)   | Higher        | Strong      | Medium     | None           |
| Write-Behind  | Low (hit)   | Very Low      | Eventual    | High       | Yes (on crash) |

---

## 8. Cache Invalidation Strategies

As Phil Karlton famously said, cache invalidation is one of the hardest problems in computer science. Let's look at practical strategies.

### 8.1 TTL-Based Invalidation

The simplest approach: every cached entry has a time-to-live. After expiration, the next request fetches fresh data.

```go
// Simple TTL setting with Redis
rdb.Set(ctx, "user:1", userData, 15*time.Minute)

// Adjusting TTL based on data volatility
func cacheTTL(dataType string) time.Duration {
    switch dataType {
    case "user_profile":
        return 15 * time.Minute // changes infrequently
    case "product_listing":
        return 5 * time.Minute // changes moderately
    case "stock_price":
        return 5 * time.Second // changes very frequently
    case "site_config":
        return 1 * time.Hour // rarely changes
    default:
        return 10 * time.Minute
    }
}
```

**TTL jittering** prevents many keys from expiring simultaneously (the "thundering herd" at expiry time):

```go
package main

import (
    "math/rand"
    "time"
)

// JitteredTTL returns a TTL with random jitter to prevent synchronized expiration.
// For a baseTTL of 10 minutes with 20% jitter, returns a value between 8-12 minutes.
func JitteredTTL(baseTTL time.Duration, jitterPercent float64) time.Duration {
    jitter := float64(baseTTL) * jitterPercent
    offset := (rand.Float64() * 2 * jitter) - jitter // range: [-jitter, +jitter]
    return baseTTL + time.Duration(offset)
}

func main() {
    for i := 0; i < 5; i++ {
        ttl := JitteredTTL(10*time.Minute, 0.2)
        _ = ttl // Each will be between 8-12 minutes
    }
}
```

### 8.2 Event-Based Invalidation

When data changes, publish an event that triggers cache invalidation. This provides near-real-time consistency.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

// CacheInvalidationEvent represents a cache invalidation message.
type CacheInvalidationEvent struct {
    Entity    string `json:"entity"`     // e.g., "user"
    ID        string `json:"id"`         // e.g., "123"
    Action    string `json:"action"`     // "update", "delete"
    Timestamp int64  `json:"timestamp"`
}

// Publisher side: when data changes, publish an invalidation event
func publishInvalidation(ctx context.Context, rdb *redis.Client, event CacheInvalidationEvent) error {
    event.Timestamp = time.Now().UnixMilli()
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    return rdb.Publish(ctx, "cache:invalidation", data).Err()
}

// Subscriber side: listen for invalidation events and delete cache entries
func startInvalidationListener(ctx context.Context, rdb *redis.Client) {
    pubsub := rdb.Subscribe(ctx, "cache:invalidation")
    defer pubsub.Close()

    ch := pubsub.Channel()
    for msg := range ch {
        var event CacheInvalidationEvent
        if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
            log.Printf("Failed to parse invalidation event: %v", err)
            continue
        }

        cacheKey := fmt.Sprintf("%s:%s", event.Entity, event.ID)
        if err := rdb.Del(ctx, cacheKey).Err(); err != nil {
            log.Printf("Failed to invalidate cache key %s: %v", cacheKey, err)
        } else {
            log.Printf("Invalidated cache key: %s (action: %s)", cacheKey, event.Action)
        }
    }
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()
    ctx := context.Background()

    // Start listener in background
    go startInvalidationListener(ctx, rdb)

    // When a user is updated, publish invalidation
    publishInvalidation(ctx, rdb, CacheInvalidationEvent{
        Entity: "user",
        ID:     "123",
        Action: "update",
    })

    time.Sleep(1 * time.Second) // give listener time to process
}
```

### 8.3 Versioned Keys

Instead of invalidating a cache entry, change the key itself by including a version number. Old entries become orphaned and expire naturally via TTL.

```go
package main

import (
    "context"
    "fmt"
    "sync/atomic"
    "time"

    "github.com/redis/go-redis/v9"
)

type VersionedCache struct {
    rdb     *redis.Client
    version atomic.Int64
    prefix  string
    ttl     time.Duration
}

func NewVersionedCache(rdb *redis.Client, prefix string, ttl time.Duration) *VersionedCache {
    vc := &VersionedCache{
        rdb:    rdb,
        prefix: prefix,
        ttl:    ttl,
    }
    vc.version.Store(1)
    return vc
}

func (c *VersionedCache) key(id string) string {
    return fmt.Sprintf("%s:v%d:%s", c.prefix, c.version.Load(), id)
}

func (c *VersionedCache) Get(ctx context.Context, id string) (string, error) {
    return c.rdb.Get(ctx, c.key(id)).Result()
}

func (c *VersionedCache) Set(ctx context.Context, id string, value string) error {
    return c.rdb.Set(ctx, c.key(id), value, c.ttl).Err()
}

// BumpVersion increments the version, effectively invalidating ALL entries.
// Old entries will be cleaned up when their TTL expires.
func (c *VersionedCache) BumpVersion() int64 {
    return c.version.Add(1)
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()
    ctx := context.Background()

    cache := NewVersionedCache(rdb, "products", 1*time.Hour)

    // Version 1: key is "products:v1:42"
    cache.Set(ctx, "42", "Widget v1 data")

    val, _ := cache.Get(ctx, "42")
    fmt.Println("Before bump:", val)

    // After a major data migration, bump the version
    cache.BumpVersion()

    // Now key is "products:v2:42" -- old entries are effectively invalidated
    _, err := cache.Get(ctx, "42")
    fmt.Println("After bump, error:", err) // redis.Nil -- cache miss
}
```

### 8.4 Tag-Based Invalidation

Group cache entries by tags. When a tag is invalidated, all entries with that tag are deleted. Useful when a single change affects multiple cache entries.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

type TaggedCache struct {
    rdb *redis.Client
    ttl time.Duration
}

func NewTaggedCache(rdb *redis.Client, ttl time.Duration) *TaggedCache {
    return &TaggedCache{rdb: rdb, ttl: ttl}
}

func (c *TaggedCache) tagSetKey(tag string) string {
    return "tag:" + tag
}

// Set stores a value and associates it with one or more tags.
func (c *TaggedCache) Set(ctx context.Context, key string, value string, tags ...string) error {
    pipe := c.rdb.Pipeline()

    // Store the value
    pipe.Set(ctx, key, value, c.ttl)

    // Associate the key with each tag
    for _, tag := range tags {
        pipe.SAdd(ctx, c.tagSetKey(tag), key)
        pipe.Expire(ctx, c.tagSetKey(tag), c.ttl+5*time.Minute) // tag set lives a bit longer
    }

    _, err := pipe.Exec(ctx)
    return err
}

func (c *TaggedCache) Get(ctx context.Context, key string) (string, error) {
    return c.rdb.Get(ctx, key).Result()
}

// InvalidateByTag deletes all cache entries associated with a tag.
func (c *TaggedCache) InvalidateByTag(ctx context.Context, tag string) error {
    tagKey := c.tagSetKey(tag)

    // Get all keys for this tag
    keys, err := c.rdb.SMembers(ctx, tagKey).Result()
    if err != nil {
        return err
    }

    if len(keys) == 0 {
        return nil
    }

    // Delete all associated keys + the tag set itself
    pipe := c.rdb.Pipeline()
    pipe.Del(ctx, keys...)
    pipe.Del(ctx, tagKey)

    _, err = pipe.Exec(ctx)
    log.Printf("Invalidated %d entries for tag %q", len(keys), tag)
    return err
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()
    ctx := context.Background()

    cache := NewTaggedCache(rdb, 15*time.Minute)

    // Cache product listings with tags
    cache.Set(ctx, "product:1", "Widget", "category:electronics", "brand:acme")
    cache.Set(ctx, "product:2", "Gadget", "category:electronics", "brand:beta")
    cache.Set(ctx, "product:3", "Doohickey", "category:tools", "brand:acme")

    // When ACME updates their brand info, invalidate all ACME products
    cache.InvalidateByTag(ctx, "brand:acme")
    // This deletes product:1 and product:3 but leaves product:2

    val, err := cache.Get(ctx, "product:2")
    fmt.Println("product:2 (still cached):", val, err)

    _, err = cache.Get(ctx, "product:1")
    fmt.Println("product:1 (invalidated):", err) // redis.Nil
}
```

---

## 9. Thundering Herd / Cache Stampede Prevention

A **cache stampede** (also called "thundering herd") occurs when a popular cache entry expires and many concurrent requests all attempt to regenerate it simultaneously. This can overwhelm the database.

```
Normal:  Request -> Cache HIT -> Return
         Request -> Cache HIT -> Return

Stampede: (key expires)
         Request 1 -> Cache MISS -> DB Query -> Update Cache
         Request 2 -> Cache MISS -> DB Query -> Update Cache
         Request 3 -> Cache MISS -> DB Query -> Update Cache
         ...
         Request N -> Cache MISS -> DB Query -> Update Cache
         (N simultaneous DB queries for the same data!)
```

### 9.1 singleflight

Go's `golang.org/x/sync/singleflight` package deduplicates concurrent calls for the same key. Only one goroutine actually executes the function; all others wait for and share the result.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    "golang.org/x/sync/singleflight"
)

type CachedService struct {
    cache  map[string]cachedValue
    mu     sync.RWMutex
    sf     singleflight.Group
}

type cachedValue struct {
    data      string
    expiresAt time.Time
}

func NewCachedService() *CachedService {
    return &CachedService{
        cache: make(map[string]cachedValue),
    }
}

// Get retrieves data with singleflight protection against cache stampede.
func (s *CachedService) Get(ctx context.Context, key string) (string, error) {
    // Check cache
    s.mu.RLock()
    if cv, ok := s.cache[key]; ok && time.Now().Before(cv.expiresAt) {
        s.mu.RUnlock()
        return cv.data, nil
    }
    s.mu.RUnlock()

    // Cache miss -- use singleflight to deduplicate concurrent loads
    result, err, shared := s.sf.Do(key, func() (interface{}, error) {
        log.Printf("[singleflight] Loading key %q (only one goroutine runs this)", key)

        // Simulate expensive database query
        time.Sleep(200 * time.Millisecond)
        data := fmt.Sprintf("data-for-%s-loaded-at-%d", key, time.Now().UnixMilli())

        // Update cache
        s.mu.Lock()
        s.cache[key] = cachedValue{
            data:      data,
            expiresAt: time.Now().Add(5 * time.Minute),
        }
        s.mu.Unlock()

        return data, nil
    })

    if err != nil {
        return "", err
    }

    log.Printf("[singleflight] Key %q result shared=%v", key, shared)
    return result.(string), nil
}

func main() {
    svc := NewCachedService()
    ctx := context.Background()

    // Simulate 100 concurrent requests for the same key
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            result, err := svc.Get(ctx, "popular-item")
            if err != nil {
                log.Printf("goroutine %d error: %v", id, err)
                return
            }
            _ = result
        }(i)
    }
    wg.Wait()

    // You'll see only ONE "Loading key" message in the logs,
    // even though 100 goroutines requested it simultaneously.
    fmt.Println("All 100 requests served, but only 1 database query executed")
}
```

### 9.2 singleflight with Redis

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
    "golang.org/x/sync/singleflight"
)

type ProductService struct {
    rdb    *redis.Client
    sf     singleflight.Group
}

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func (s *ProductService) GetProduct(ctx context.Context, id int) (*Product, error) {
    cacheKey := fmt.Sprintf("product:%d", id)

    // Try cache
    data, err := s.rdb.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var p Product
        json.Unmarshal(data, &p)
        return &p, nil
    }

    // singleflight: only one goroutine loads from DB per product ID
    sfKey := fmt.Sprintf("product:%d", id)
    result, err, _ := s.sf.Do(sfKey, func() (interface{}, error) {
        // Double-check cache (another goroutine may have populated it)
        data, err := s.rdb.Get(ctx, cacheKey).Bytes()
        if err == nil {
            var p Product
            json.Unmarshal(data, &p)
            return &p, nil
        }

        // Load from database
        product, err := loadProductFromDB(ctx, id)
        if err != nil {
            return nil, err
        }

        // Populate cache
        encoded, _ := json.Marshal(product)
        s.rdb.Set(ctx, cacheKey, encoded, 10*time.Minute)

        return product, nil
    })

    if err != nil {
        return nil, err
    }
    return result.(*Product), nil
}

func loadProductFromDB(ctx context.Context, id int) (*Product, error) {
    log.Printf("DB QUERY: loading product %d", id)
    return &Product{ID: id, Name: "Widget", Price: 9.99}, nil
}

func main() {
    fmt.Println("singleflight + Redis example")
}
```

### 9.3 Early Expiration (Probabilistic Refresh)

Instead of waiting for a key to actually expire, proactively refresh it when it is close to expiration. This avoids the stampede entirely because the key never truly expires under load.

```go
package main

import (
    "context"
    "fmt"
    "math"
    "math/rand"
    "sync"
    "time"
)

type EarlyExpirationCache struct {
    mu    sync.RWMutex
    items map[string]*earlyEntry
    sf    sync.Map // singleflight per key
}

type earlyEntry struct {
    value     interface{}
    createdAt time.Time
    ttl       time.Duration
}

func NewEarlyExpirationCache() *EarlyExpirationCache {
    return &EarlyExpirationCache{
        items: make(map[string]*earlyEntry),
    }
}

// ShouldRefresh uses an exponential probability function to decide
// if we should proactively refresh the cache entry before it expires.
// As the entry gets closer to expiry, the probability of refresh increases.
func (e *earlyEntry) ShouldRefresh() bool {
    elapsed := time.Since(e.createdAt)
    remaining := e.ttl - elapsed

    if remaining <= 0 {
        return true // actually expired
    }

    // Probability increases exponentially as we approach expiry
    // At 80% of TTL elapsed: ~18% chance
    // At 90% of TTL elapsed: ~63% chance
    // At 95% of TTL elapsed: ~92% chance
    ratio := float64(elapsed) / float64(e.ttl)
    if ratio < 0.7 {
        return false // too early to consider refresh
    }

    beta := 1.0
    probability := math.Exp(-beta * float64(remaining) / float64(e.ttl) * 10)
    return rand.Float64() < probability
}

func (c *EarlyExpirationCache) Get(ctx context.Context, key string, loader func() (interface{}, error)) (interface{}, error) {
    c.mu.RLock()
    entry, exists := c.items[key]
    c.mu.RUnlock()

    if exists && !entry.ShouldRefresh() {
        return entry.value, nil
    }

    // Refresh in background if entry still exists (stale-while-revalidate)
    if exists {
        go c.refresh(key, loader) // non-blocking background refresh
        return entry.value, nil   // return stale data immediately
    }

    // No entry at all -- must block and load
    return c.loadAndStore(key, loader)
}

func (c *EarlyExpirationCache) refresh(key string, loader func() (interface{}, error)) {
    c.loadAndStore(key, loader) // errors are silently ignored for background refresh
}

func (c *EarlyExpirationCache) loadAndStore(key string, loader func() (interface{}, error)) (interface{}, error) {
    value, err := loader()
    if err != nil {
        return nil, err
    }

    c.mu.Lock()
    c.items[key] = &earlyEntry{
        value:     value,
        createdAt: time.Now(),
        ttl:       5 * time.Minute,
    }
    c.mu.Unlock()

    return value, nil
}

func main() {
    cache := NewEarlyExpirationCache()
    ctx := context.Background()

    val, _ := cache.Get(ctx, "data", func() (interface{}, error) {
        fmt.Println("Loading from source...")
        return "fresh-data", nil
    })
    fmt.Println("Value:", val)
}
```

### 9.4 Distributed Locking

For distributed systems where `singleflight` only protects within a single process, use a distributed lock (e.g., Redis-based) to ensure only one node regenerates a cache entry.

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// TryLock attempts to acquire a distributed lock using Redis SET NX.
func TryLock(ctx context.Context, rdb *redis.Client, lockKey string, ttl time.Duration) (bool, error) {
    ok, err := rdb.SetNX(ctx, lockKey, "locked", ttl).Result()
    return ok, err
}

func Unlock(ctx context.Context, rdb *redis.Client, lockKey string) error {
    return rdb.Del(ctx, lockKey).Err()
}

// GetWithDistributedLock prevents cache stampede across multiple nodes.
func GetWithDistributedLock(
    ctx context.Context,
    rdb *redis.Client,
    cacheKey string,
    loader func() (string, error),
) (string, error) {
    // Try cache
    val, err := rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        return val, nil
    }

    // Cache miss -- try to acquire lock
    lockKey := "lock:" + cacheKey
    locked, err := TryLock(ctx, rdb, lockKey, 10*time.Second)
    if err != nil {
        return "", fmt.Errorf("lock error: %w", err)
    }

    if locked {
        defer Unlock(ctx, rdb, lockKey)

        // Double-check cache
        val, err = rdb.Get(ctx, cacheKey).Result()
        if err == nil {
            return val, nil
        }

        // Load from source
        val, err = loader()
        if err != nil {
            return "", err
        }

        // Populate cache
        rdb.Set(ctx, cacheKey, val, 10*time.Minute)
        return val, nil
    }

    // Another process holds the lock -- wait and retry
    for i := 0; i < 50; i++ { // max 5 seconds wait
        time.Sleep(100 * time.Millisecond)
        val, err = rdb.Get(ctx, cacheKey).Result()
        if err == nil {
            return val, nil
        }
    }

    // Timeout -- load directly as fallback
    return loader()
}

func main() {
    fmt.Println("Distributed lock cache stampede prevention example")
}
```

---

## 10. Multi-Tier Caching

Production systems often use multiple cache layers: a fast in-process L1 cache and a shared L2 cache (like Redis). This combines the sub-microsecond latency of local memory with the shared state of a distributed cache.

### 10.1 Architecture

```
Request
   │
   ▼
┌────────────────┐
│   L1 Cache     │  In-process (bigcache, ristretto, etc.)
│   ~100ns       │  Per-instance, not shared
└───────┬────────┘
        │ MISS
        ▼
┌────────────────┐
│   L2 Cache     │  Redis / Memcached
│   ~0.5ms       │  Shared across instances
└───────┬────────┘
        │ MISS
        ▼
┌────────────────┐
│   Database     │  PostgreSQL, MySQL, etc.
│   ~5ms         │  Source of truth
└────────────────┘
```

### 10.2 Full Implementation

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
    "golang.org/x/sync/singleflight"
)

// ---- L1 Cache (in-process) ----

type L1Cache struct {
    mu    sync.RWMutex
    items map[string]l1Entry
    maxSize int
}

type l1Entry struct {
    data      []byte
    expiresAt time.Time
}

func NewL1Cache(maxSize int) *L1Cache {
    return &L1Cache{
        items:   make(map[string]l1Entry, maxSize),
        maxSize: maxSize,
    }
}

func (c *L1Cache) Get(key string) ([]byte, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, ok := c.items[key]
    if !ok || time.Now().After(entry.expiresAt) {
        return nil, false
    }
    return entry.data, true
}

func (c *L1Cache) Set(key string, data []byte, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    // Simple eviction: if at capacity, clear half the cache
    if len(c.items) >= c.maxSize {
        count := 0
        for k := range c.items {
            delete(c.items, k)
            count++
            if count >= c.maxSize/2 {
                break
            }
        }
    }

    c.items[key] = l1Entry{
        data:      data,
        expiresAt: time.Now().Add(ttl),
    }
}

func (c *L1Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

// ---- Multi-Tier Cache ----

type MultiTierCache struct {
    l1     *L1Cache
    l2     *redis.Client
    l1TTL  time.Duration
    l2TTL  time.Duration
    sf     singleflight.Group

    // Metrics
    mu       sync.Mutex
    l1Hits   int64
    l2Hits   int64
    dbLoads  int64
}

func NewMultiTierCache(l2 *redis.Client) *MultiTierCache {
    return &MultiTierCache{
        l1:    NewL1Cache(10000),
        l2:    l2,
        l1TTL: 1 * time.Minute,   // L1: short TTL (local, fast refresh)
        l2TTL: 15 * time.Minute,  // L2: longer TTL (shared, slower refresh)
    }
}

// Get retrieves data checking L1, then L2, then loading from the source.
func (c *MultiTierCache) Get(
    ctx context.Context,
    key string,
    loader func(ctx context.Context) (interface{}, error),
) (interface{}, error) {

    // ---- L1: In-process cache ----
    if data, ok := c.l1.Get(key); ok {
        var result interface{}
        if err := json.Unmarshal(data, &result); err == nil {
            c.recordHit("l1")
            return result, nil
        }
    }

    // ---- L2: Redis ----
    data, err := c.l2.Get(ctx, key).Bytes()
    if err == nil {
        // Populate L1 from L2
        c.l1.Set(key, data, c.l1TTL)

        var result interface{}
        if err := json.Unmarshal(data, &result); err == nil {
            c.recordHit("l2")
            return result, nil
        }
    }
    if err != nil && !errors.Is(err, redis.Nil) {
        log.Printf("L2 cache error for %s: %v", key, err)
    }

    // ---- Database (with singleflight) ----
    result, err, _ := c.sf.Do(key, func() (interface{}, error) {
        // Double-check L2 (another goroutine may have populated it)
        if data, err := c.l2.Get(ctx, key).Bytes(); err == nil {
            c.l1.Set(key, data, c.l1TTL)
            var result interface{}
            if err := json.Unmarshal(data, &result); err == nil {
                return result, nil
            }
        }

        // Load from source
        value, err := loader(ctx)
        if err != nil {
            return nil, err
        }

        c.recordHit("db")

        // Populate both cache tiers
        encoded, _ := json.Marshal(value)
        c.l2.Set(ctx, key, encoded, c.l2TTL)
        c.l1.Set(key, encoded, c.l1TTL)

        return value, nil
    })

    return result, err
}

// Invalidate removes an entry from both cache tiers.
func (c *MultiTierCache) Invalidate(ctx context.Context, key string) {
    c.l1.Delete(key)
    c.l2.Del(ctx, key)
}

func (c *MultiTierCache) recordHit(tier string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    switch tier {
    case "l1":
        c.l1Hits++
    case "l2":
        c.l2Hits++
    case "db":
        c.dbLoads++
    }
}

func (c *MultiTierCache) Stats() (l1Hits, l2Hits, dbLoads int64) {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.l1Hits, c.l2Hits, c.dbLoads
}

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    defer rdb.Close()

    cache := NewMultiTierCache(rdb)
    ctx := context.Background()

    loader := func(ctx context.Context) (interface{}, error) {
        log.Println("Loading from database...")
        return map[string]interface{}{
            "id":   1,
            "name": "Alice",
        }, nil
    }

    // First call: L1 miss -> L2 miss -> DB load
    val, _ := cache.Get(ctx, "user:1", loader)
    fmt.Printf("Call 1: %v\n", val)

    // Second call: L1 hit
    val, _ = cache.Get(ctx, "user:1", loader)
    fmt.Printf("Call 2: %v\n", val)

    l1, l2, db := cache.Stats()
    fmt.Printf("L1 hits: %d, L2 hits: %d, DB loads: %d\n", l1, l2, db)
}
```

### 10.3 L1 Invalidation Across Instances

When you have multiple application instances, each with its own L1 cache, you need a mechanism to invalidate L1 across all instances when data changes. Redis Pub/Sub works well for this.

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/redis/go-redis/v9"
)

type L1Coordinator struct {
    rdb     *redis.Client
    l1      *L1Cache // from previous example
    channel string
}

func NewL1Coordinator(rdb *redis.Client, l1 *L1Cache) *L1Coordinator {
    c := &L1Coordinator{
        rdb:     rdb,
        l1:      l1,
        channel: "l1:invalidate",
    }
    go c.subscribe()
    return c
}

// InvalidateAcrossInstances publishes an invalidation message.
func (c *L1Coordinator) InvalidateAcrossInstances(ctx context.Context, key string) error {
    // Invalidate local L1 immediately
    c.l1.Delete(key)
    // Notify other instances
    return c.rdb.Publish(ctx, c.channel, key).Err()
}

func (c *L1Coordinator) subscribe() {
    ctx := context.Background()
    pubsub := c.rdb.Subscribe(ctx, c.channel)
    defer pubsub.Close()

    for msg := range pubsub.Channel() {
        key := msg.Payload
        c.l1.Delete(key)
        log.Printf("L1 invalidated via pub/sub: %s", key)
    }
}

func main() {
    fmt.Println("L1 invalidation coordinator via Redis Pub/Sub")
}
```

---

## 11. Cache Warming and Preloading

Cache warming is the process of populating the cache before traffic hits it. This is critical after deployments, restarts, or when spinning up new instances.

### 11.1 Startup Warming

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

type CacheWarmer struct {
    rdb *redis.Client
    db  *sql.DB
}

func NewCacheWarmer(rdb *redis.Client, db *sql.DB) *CacheWarmer {
    return &CacheWarmer{rdb: rdb, db: db}
}

// WarmPopularProducts loads the top N products into cache.
func (w *CacheWarmer) WarmPopularProducts(ctx context.Context, limit int) error {
    log.Printf("Warming cache with top %d products...", limit)
    start := time.Now()

    rows, err := w.db.QueryContext(ctx,
        `SELECT id, name, price FROM products
         ORDER BY view_count DESC LIMIT $1`, limit)
    if err != nil {
        return fmt.Errorf("query failed: %w", err)
    }
    defer rows.Close()

    pipe := w.rdb.Pipeline()
    count := 0

    for rows.Next() {
        var id int
        var name string
        var price float64

        if err := rows.Scan(&id, &name, &price); err != nil {
            log.Printf("scan error: %v", err)
            continue
        }

        data, _ := json.Marshal(map[string]interface{}{
            "id": id, "name": name, "price": price,
        })

        key := fmt.Sprintf("product:%d", id)
        pipe.Set(ctx, key, data, 30*time.Minute)
        count++

        // Execute in batches of 100
        if count%100 == 0 {
            if _, err := pipe.Exec(ctx); err != nil {
                log.Printf("pipeline exec error: %v", err)
            }
            pipe = w.rdb.Pipeline()
        }
    }

    // Flush remaining
    if _, err := pipe.Exec(ctx); err != nil {
        log.Printf("final pipeline exec error: %v", err)
    }

    log.Printf("Warmed %d products in %v", count, time.Since(start))
    return nil
}

// WarmAll runs all warming tasks concurrently.
func (w *CacheWarmer) WarmAll(ctx context.Context) error {
    var wg sync.WaitGroup
    errCh := make(chan error, 3)

    warmTasks := []struct {
        name string
        fn   func(context.Context) error
    }{
        {"popular_products", func(ctx context.Context) error {
            return w.WarmPopularProducts(ctx, 1000)
        }},
        {"site_config", func(ctx context.Context) error {
            return w.warmSiteConfig(ctx)
        }},
        {"categories", func(ctx context.Context) error {
            return w.warmCategories(ctx)
        }},
    }

    for _, task := range warmTasks {
        wg.Add(1)
        go func(name string, fn func(context.Context) error) {
            defer wg.Done()
            if err := fn(ctx); err != nil {
                errCh <- fmt.Errorf("%s warming failed: %w", name, err)
            }
        }(task.name, task.fn)
    }

    wg.Wait()
    close(errCh)

    for err := range errCh {
        log.Printf("WARNING: %v", err)
    }

    return nil
}

func (w *CacheWarmer) warmSiteConfig(ctx context.Context) error {
    log.Println("Warming site configuration...")
    // Load and cache site configuration
    return nil
}

func (w *CacheWarmer) warmCategories(ctx context.Context) error {
    log.Println("Warming categories...")
    // Load and cache category tree
    return nil
}

func main() {
    fmt.Println("Cache warming example")
    // In production:
    // warmer := NewCacheWarmer(rdb, db)
    // warmer.WarmAll(context.Background())
    // Then start accepting traffic
}
```

### 11.2 Lazy Warming with Background Refresh

Instead of blocking startup, lazily warm the cache by refreshing entries in the background before they expire.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

type BackgroundRefreshCache struct {
    mu       sync.RWMutex
    items    map[string]*refreshEntry
    stopCh   chan struct{}
}

type refreshEntry struct {
    value     interface{}
    expiresAt time.Time
    ttl       time.Duration
    loader    func(ctx context.Context) (interface{}, error)
}

func NewBackgroundRefreshCache() *BackgroundRefreshCache {
    c := &BackgroundRefreshCache{
        items:  make(map[string]*refreshEntry),
        stopCh: make(chan struct{}),
    }
    go c.refreshLoop()
    return c
}

func (c *BackgroundRefreshCache) Register(key string, ttl time.Duration, loader func(ctx context.Context) (interface{}, error)) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[key] = &refreshEntry{
        ttl:    ttl,
        loader: loader,
    }
}

func (c *BackgroundRefreshCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, ok := c.items[key]
    if !ok || entry.value == nil {
        return nil, false
    }
    return entry.value, true
}

func (c *BackgroundRefreshCache) refreshLoop() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            c.refreshExpiring()
        case <-c.stopCh:
            return
        }
    }
}

func (c *BackgroundRefreshCache) refreshExpiring() {
    c.mu.RLock()
    var toRefresh []string
    now := time.Now()

    for key, entry := range c.items {
        // Refresh at 80% of TTL or if never loaded
        if entry.value == nil || now.After(entry.expiresAt.Add(-entry.ttl/5)) {
            toRefresh = append(toRefresh, key)
        }
    }
    c.mu.RUnlock()

    for _, key := range toRefresh {
        c.mu.RLock()
        entry, ok := c.items[key]
        c.mu.RUnlock()
        if !ok {
            continue
        }

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        value, err := entry.loader(ctx)
        cancel()

        if err != nil {
            log.Printf("Background refresh failed for %s: %v", key, err)
            continue
        }

        c.mu.Lock()
        entry.value = value
        entry.expiresAt = time.Now().Add(entry.ttl)
        c.mu.Unlock()

        log.Printf("Background refresh completed for %s", key)
    }
}

func (c *BackgroundRefreshCache) Stop() {
    close(c.stopCh)
}

func main() {
    cache := NewBackgroundRefreshCache()
    defer cache.Stop()

    // Register data sources with their loaders
    cache.Register("exchange_rates", 30*time.Second, func(ctx context.Context) (interface{}, error) {
        fmt.Println("Fetching exchange rates...")
        return map[string]float64{"USD/EUR": 0.85, "USD/GBP": 0.73}, nil
    })

    cache.Register("feature_flags", 1*time.Minute, func(ctx context.Context) (interface{}, error) {
        fmt.Println("Fetching feature flags...")
        return map[string]bool{"new_checkout": true, "dark_mode": false}, nil
    })

    // Wait for initial load
    time.Sleep(2 * time.Second)

    if val, ok := cache.Get("exchange_rates"); ok {
        fmt.Printf("Exchange rates: %v\n", val)
    }
}
```

---

## 12. Distributed Caching with Consistent Hashing

When you shard your cache across multiple nodes, consistent hashing ensures that adding or removing a node only remaps a small fraction of keys, not all of them.

### 12.1 How Consistent Hashing Works

Traditional hashing: `node = hash(key) % num_nodes` -- adding a node remaps nearly all keys.

Consistent hashing: nodes are placed on a ring. A key is assigned to the first node clockwise from its hash position. Adding a node only remaps keys between the new node and its predecessor.

```
       Traditional Modular Hashing         Consistent Hashing Ring

  3 nodes: key -> node 0,1,2               Node A ----+---- Node B
  4 nodes: key -> node 0,1,2,3              /                  \
  Almost ALL keys remap!                   /     hash ring      \
                                        Node D ----+---- Node C

                                        Adding Node E only remaps
                                        keys between D and E.
```

### 12.2 Implementation

```go
package main

import (
    "fmt"
    "hash/crc32"
    "sort"
    "strconv"
    "sync"
)

// ConsistentHash implements a consistent hashing ring.
type ConsistentHash struct {
    mu       sync.RWMutex
    ring     []uint32          // sorted hash values
    nodes    map[uint32]string // hash -> node name
    replicas int               // virtual nodes per physical node
}

func NewConsistentHash(replicas int) *ConsistentHash {
    return &ConsistentHash{
        nodes:    make(map[uint32]string),
        replicas: replicas,
    }
}

func (ch *ConsistentHash) hash(key string) uint32 {
    return crc32.ChecksumIEEE([]byte(key))
}

// AddNode adds a node to the ring with virtual replicas.
func (ch *ConsistentHash) AddNode(node string) {
    ch.mu.Lock()
    defer ch.mu.Unlock()

    for i := 0; i < ch.replicas; i++ {
        virtualKey := fmt.Sprintf("%s#%d", node, i)
        h := ch.hash(virtualKey)
        ch.ring = append(ch.ring, h)
        ch.nodes[h] = node
    }

    sort.Slice(ch.ring, func(i, j int) bool {
        return ch.ring[i] < ch.ring[j]
    })
}

// RemoveNode removes a node and all its virtual replicas from the ring.
func (ch *ConsistentHash) RemoveNode(node string) {
    ch.mu.Lock()
    defer ch.mu.Unlock()

    newRing := make([]uint32, 0, len(ch.ring))
    for _, h := range ch.ring {
        if ch.nodes[h] == node {
            delete(ch.nodes, h)
        } else {
            newRing = append(newRing, h)
        }
    }
    ch.ring = newRing
}

// GetNode returns the node responsible for the given key.
func (ch *ConsistentHash) GetNode(key string) string {
    ch.mu.RLock()
    defer ch.mu.RUnlock()

    if len(ch.ring) == 0 {
        return ""
    }

    h := ch.hash(key)

    // Binary search for the first node with hash >= key hash
    idx := sort.Search(len(ch.ring), func(i int) bool {
        return ch.ring[i] >= h
    })

    // Wrap around to the first node if we passed the end
    if idx >= len(ch.ring) {
        idx = 0
    }

    return ch.nodes[ch.ring[idx]]
}

// GetNodes returns N distinct nodes for replication.
func (ch *ConsistentHash) GetNodes(key string, count int) []string {
    ch.mu.RLock()
    defer ch.mu.RUnlock()

    if len(ch.ring) == 0 {
        return nil
    }

    h := ch.hash(key)
    idx := sort.Search(len(ch.ring), func(i int) bool {
        return ch.ring[i] >= h
    })

    seen := make(map[string]bool)
    var result []string

    for i := 0; i < len(ch.ring) && len(result) < count; i++ {
        nodeIdx := (idx + i) % len(ch.ring)
        node := ch.nodes[ch.ring[nodeIdx]]
        if !seen[node] {
            seen[node] = true
            result = append(result, node)
        }
    }

    return result
}

func main() {
    ch := NewConsistentHash(150) // 150 virtual nodes per physical node

    // Add cache nodes
    ch.AddNode("cache-1:6379")
    ch.AddNode("cache-2:6379")
    ch.AddNode("cache-3:6379")

    // Route keys to nodes
    testKeys := []string{"user:1", "user:2", "user:3", "product:42", "session:abc"}
    for _, key := range testKeys {
        node := ch.GetNode(key)
        fmt.Printf("Key %-15s -> Node %s\n", key, node)
    }

    fmt.Println("\n--- Adding cache-4 ---")
    ch.AddNode("cache-4:6379")

    // Check how many keys moved
    moved := 0
    for i := 0; i < 10000; i++ {
        key := "key:" + strconv.Itoa(i)
        // In practice you'd compare before/after
        _ = ch.GetNode(key)
    }

    for _, key := range testKeys {
        node := ch.GetNode(key)
        fmt.Printf("Key %-15s -> Node %s\n", key, node)
    }

    // Replication: get 2 nodes for each key
    fmt.Println("\n--- Replication (2 copies) ---")
    for _, key := range testKeys {
        nodes := ch.GetNodes(key, 2)
        fmt.Printf("Key %-15s -> Nodes %v\n", key, nodes)
    }

    _ = moved
}
```

### 12.3 Distributed Cache Client

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/redis/go-redis/v9"
)

type DistributedCache struct {
    ring    *ConsistentHash
    clients map[string]*redis.Client
}

func NewDistributedCache(nodes []string) *DistributedCache {
    dc := &DistributedCache{
        ring:    NewConsistentHash(150),
        clients: make(map[string]*redis.Client),
    }

    for _, node := range nodes {
        dc.ring.AddNode(node)
        dc.clients[node] = redis.NewClient(&redis.Options{
            Addr:         node,
            DialTimeout:  2 * time.Second,
            ReadTimeout:  1 * time.Second,
            WriteTimeout: 1 * time.Second,
            PoolSize:     10,
        })
    }

    return dc
}

func (dc *DistributedCache) getClient(key string) *redis.Client {
    node := dc.ring.GetNode(key)
    return dc.clients[node]
}

func (dc *DistributedCache) Get(ctx context.Context, key string) (string, error) {
    client := dc.getClient(key)
    return client.Get(ctx, key).Result()
}

func (dc *DistributedCache) Set(ctx context.Context, key, value string, ttl time.Duration) error {
    client := dc.getClient(key)
    return client.Set(ctx, key, value, ttl).Err()
}

func (dc *DistributedCache) Delete(ctx context.Context, key string) error {
    client := dc.getClient(key)
    return client.Del(ctx, key).Err()
}

// SetWithReplication writes to multiple nodes for redundancy.
func (dc *DistributedCache) SetWithReplication(ctx context.Context, key, value string, ttl time.Duration, replicas int) error {
    nodes := dc.ring.GetNodes(key, replicas)
    var lastErr error
    for _, node := range nodes {
        if err := dc.clients[node].Set(ctx, key, value, ttl).Err(); err != nil {
            lastErr = err
            log.Printf("Replication to %s failed: %v", node, err)
        }
    }
    return lastErr
}

func (dc *DistributedCache) Close() {
    for _, client := range dc.clients {
        client.Close()
    }
}

func main() {
    dc := NewDistributedCache([]string{
        "redis-1:6379",
        "redis-2:6379",
        "redis-3:6379",
    })
    defer dc.Close()

    ctx := context.Background()

    // Single-node write
    dc.Set(ctx, "user:1", "Alice", 10*time.Minute)

    // Replicated write (2 copies)
    dc.SetWithReplication(ctx, "critical:config", "value", 1*time.Hour, 2)

    fmt.Println("Distributed cache with consistent hashing")
}
```

---

## 13. HTTP Caching Headers

Proper HTTP caching reduces server load and improves client-side performance. Go's `net/http` package provides the building blocks, but you need to set the right headers.

### 13.1 Cache-Control

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// CacheControl sets appropriate Cache-Control headers based on content type.
func CacheControl(maxAge time.Duration, directives ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            cc := fmt.Sprintf("max-age=%d", int(maxAge.Seconds()))
            for _, d := range directives {
                cc += ", " + d
            }
            w.Header().Set("Cache-Control", cc)
            next.ServeHTTP(w, r)
        })
    }
}

func main() {
    mux := http.NewServeMux()

    // Static assets: cache for 1 year, immutable
    staticHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("static content"))
    })
    mux.Handle("/static/",
        CacheControl(365*24*time.Hour, "public", "immutable")(staticHandler))

    // API responses: cache for 5 minutes, must revalidate
    apiHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"data": "response"}`))
    })
    mux.Handle("/api/products",
        CacheControl(5*time.Minute, "public", "must-revalidate")(apiHandler))

    // User-specific content: private cache only
    privateHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("user-specific content"))
    })
    mux.Handle("/api/me",
        CacheControl(1*time.Minute, "private", "no-store")(privateHandler))

    http.ListenAndServe(":8080", mux)
}
```

### 13.2 ETag-Based Conditional Requests

ETags allow clients to validate their cached copy with the server. If the content has not changed, the server returns `304 Not Modified` instead of the full response body.

```go
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "net/http"
    "time"
)

type Product struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Price     float64   `json:"price"`
    UpdatedAt time.Time `json:"updated_at"`
}

// ETagMiddleware adds ETag support to responses.
func ETagMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Wrap the response writer to capture the body
        rec := &responseRecorder{
            ResponseWriter: w,
            statusCode:     http.StatusOK,
        }

        next.ServeHTTP(rec, r)

        // Generate ETag from response body
        hash := sha256.Sum256(rec.body)
        etag := `"` + hex.EncodeToString(hash[:16]) + `"`

        // Check If-None-Match header
        if match := r.Header.Get("If-None-Match"); match == etag {
            w.WriteHeader(http.StatusNotModified)
            return
        }

        // Set ETag header and write the response
        w.Header().Set("ETag", etag)
        w.Header().Set("Cache-Control", "public, max-age=0, must-revalidate")
        w.WriteHeader(rec.statusCode)
        w.Write(rec.body)
    })
}

type responseRecorder struct {
    http.ResponseWriter
    statusCode int
    body       []byte
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
    // Don't write header yet -- we need to compute ETag first
}

func (r *responseRecorder) Write(b []byte) (int, error) {
    r.body = append(r.body, b...)
    return len(b), nil
}

func main() {
    mux := http.NewServeMux()

    mux.Handle("/api/product/1", ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        product := Product{
            ID:        1,
            Name:      "Widget",
            Price:     9.99,
            UpdatedAt: time.Date(2025, 1, 15, 0, 0, 0, 0, time.UTC),
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(product)
    })))

    // Client flow:
    // 1. GET /api/product/1
    //    Response: 200 OK + ETag: "abc123..."
    //
    // 2. GET /api/product/1 with If-None-Match: "abc123..."
    //    Response: 304 Not Modified (no body -- saves bandwidth)

    http.ListenAndServe(":8080", mux)
}
```

### 13.3 Last-Modified / If-Modified-Since

```go
package main

import (
    "encoding/json"
    "net/http"
    "time"
)

func LastModifiedHandler(w http.ResponseWriter, r *http.Request) {
    // In production, this would come from the database
    lastModified := time.Date(2025, 6, 15, 10, 30, 0, 0, time.UTC)

    // Check If-Modified-Since
    if ims := r.Header.Get("If-Modified-Since"); ims != "" {
        if t, err := http.ParseTime(ims); err == nil {
            if !lastModified.After(t) {
                w.WriteHeader(http.StatusNotModified)
                return
            }
        }
    }

    w.Header().Set("Last-Modified", lastModified.UTC().Format(http.TimeFormat))
    w.Header().Set("Cache-Control", "public, max-age=60")
    w.Header().Set("Content-Type", "application/json")

    json.NewEncoder(w).Encode(map[string]string{
        "data": "this is the response body",
    })
}

func main() {
    http.HandleFunc("/api/data", LastModifiedHandler)
    http.ListenAndServe(":8080", nil)
}
```

### 13.4 Cache-Control Directive Reference

| Directive          | Meaning                                                            |
|--------------------|--------------------------------------------------------------------|
| `public`           | Response can be cached by any cache (CDN, proxy, browser)         |
| `private`          | Response is for a single user; only browser cache allowed          |
| `no-cache`         | Cache must revalidate with origin before using cached copy         |
| `no-store`         | Do not cache the response at all                                   |
| `max-age=N`        | Response is fresh for N seconds                                    |
| `s-maxage=N`       | Like max-age but for shared caches (CDNs)                         |
| `must-revalidate`  | Once stale, cache must not use without revalidation               |
| `immutable`        | Content will never change; skip revalidation even on refresh      |
| `stale-while-revalidate=N` | Serve stale for N seconds while revalidating in background |

---

## 14. Monitoring Cache Hit Rates and Performance

You cannot optimize what you do not measure. Cache monitoring is essential for understanding whether your caching strategy is effective.

### 14.1 Cache Metrics Collector

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

// CacheMetrics tracks cache performance statistics.
type CacheMetrics struct {
    hits       atomic.Int64
    misses     atomic.Int64
    sets       atomic.Int64
    deletes    atomic.Int64
    evictions  atomic.Int64
    errors     atomic.Int64
    totalGetNs atomic.Int64 // total nanoseconds spent in Get
    totalSetNs atomic.Int64 // total nanoseconds spent in Set

    // Size tracking
    mu         sync.RWMutex
    currentSize int64
    maxSize     int64
}

func NewCacheMetrics() *CacheMetrics {
    return &CacheMetrics{}
}

func (m *CacheMetrics) RecordHit(latency time.Duration) {
    m.hits.Add(1)
    m.totalGetNs.Add(int64(latency))
}

func (m *CacheMetrics) RecordMiss(latency time.Duration) {
    m.misses.Add(1)
    m.totalGetNs.Add(int64(latency))
}

func (m *CacheMetrics) RecordSet(latency time.Duration) {
    m.sets.Add(1)
    m.totalSetNs.Add(int64(latency))
}

func (m *CacheMetrics) RecordEviction() {
    m.evictions.Add(1)
}

func (m *CacheMetrics) RecordError() {
    m.errors.Add(1)
}

func (m *CacheMetrics) UpdateSize(current, max int64) {
    m.mu.Lock()
    m.currentSize = current
    m.maxSize = max
    m.mu.Unlock()
}

// HitRate returns the cache hit rate as a ratio between 0 and 1.
func (m *CacheMetrics) HitRate() float64 {
    hits := m.hits.Load()
    misses := m.misses.Load()
    total := hits + misses
    if total == 0 {
        return 0
    }
    return float64(hits) / float64(total)
}

// AvgGetLatency returns the average GET latency.
func (m *CacheMetrics) AvgGetLatency() time.Duration {
    total := m.hits.Load() + m.misses.Load()
    if total == 0 {
        return 0
    }
    return time.Duration(m.totalGetNs.Load() / total)
}

// AvgSetLatency returns the average SET latency.
func (m *CacheMetrics) AvgSetLatency() time.Duration {
    sets := m.sets.Load()
    if sets == 0 {
        return 0
    }
    return time.Duration(m.totalSetNs.Load() / sets)
}

// Snapshot returns a point-in-time snapshot of all metrics.
func (m *CacheMetrics) Snapshot() map[string]interface{} {
    m.mu.RLock()
    currentSize := m.currentSize
    maxSize := m.maxSize
    m.mu.RUnlock()

    return map[string]interface{}{
        "hits":            m.hits.Load(),
        "misses":          m.misses.Load(),
        "hit_rate":        m.HitRate(),
        "sets":            m.sets.Load(),
        "deletes":         m.deletes.Load(),
        "evictions":       m.evictions.Load(),
        "errors":          m.errors.Load(),
        "avg_get_latency": m.AvgGetLatency().String(),
        "avg_set_latency": m.AvgSetLatency().String(),
        "current_size":    currentSize,
        "max_size":        maxSize,
        "utilization":     float64(currentSize) / float64(maxSize),
    }
}

func (m *CacheMetrics) String() string {
    snap := m.Snapshot()
    return fmt.Sprintf(
        "Hits: %d | Misses: %d | Hit Rate: %.2f%% | Evictions: %d | Avg Get: %s | Size: %d/%d (%.1f%%)",
        snap["hits"], snap["misses"],
        snap["hit_rate"].(float64)*100,
        snap["evictions"],
        snap["avg_get_latency"],
        snap["current_size"], snap["max_size"],
        snap["utilization"].(float64)*100,
    )
}

func main() {
    metrics := NewCacheMetrics()

    // Simulate some cache operations
    for i := 0; i < 1000; i++ {
        start := time.Now()
        time.Sleep(10 * time.Microsecond) // simulate work
        latency := time.Since(start)

        if i%5 == 0 {
            metrics.RecordMiss(latency)
        } else {
            metrics.RecordHit(latency)
        }
    }

    metrics.UpdateSize(800, 1000)

    fmt.Println(metrics.String())
    fmt.Printf("\nFull snapshot: %v\n", metrics.Snapshot())
}
```

### 14.2 Prometheus Integration

```go
package main

import (
    "net/http"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    cacheHits = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "cache_hits_total",
        Help: "Total number of cache hits",
    }, []string{"cache_name"})

    cacheMisses = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "cache_misses_total",
        Help: "Total number of cache misses",
    }, []string{"cache_name"})

    cacheLatency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "cache_operation_duration_seconds",
        Help:    "Time spent on cache operations",
        Buckets: []float64{.0001, .0005, .001, .005, .01, .05, .1, .5, 1},
    }, []string{"cache_name", "operation"})

    cacheSize = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "cache_entries_current",
        Help: "Current number of entries in the cache",
    }, []string{"cache_name"})

    cacheEvictions = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "cache_evictions_total",
        Help: "Total number of cache evictions",
    }, []string{"cache_name"})
)

// InstrumentedCache wraps any cache with Prometheus metrics.
type InstrumentedCache struct {
    name string
    // In a real implementation, this would wrap an actual cache
}

func (c *InstrumentedCache) RecordHit() {
    cacheHits.WithLabelValues(c.name).Inc()
}

func (c *InstrumentedCache) RecordMiss() {
    cacheMisses.WithLabelValues(c.name).Inc()
}

func (c *InstrumentedCache) ObserveGetLatency(seconds float64) {
    cacheLatency.WithLabelValues(c.name, "get").Observe(seconds)
}

func (c *InstrumentedCache) ObserveSetLatency(seconds float64) {
    cacheLatency.WithLabelValues(c.name, "set").Observe(seconds)
}

func (c *InstrumentedCache) SetSize(size float64) {
    cacheSize.WithLabelValues(c.name).Set(size)
}

func (c *InstrumentedCache) RecordEviction() {
    cacheEvictions.WithLabelValues(c.name).Inc()
}

func main() {
    // Example: expose metrics endpoint
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":2112", nil)
}
```

### 14.3 Useful Prometheus Queries

```promql
# Cache hit rate over 5 minutes
rate(cache_hits_total[5m]) / (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))

# Eviction rate per second
rate(cache_evictions_total[5m])

# 99th percentile GET latency
histogram_quantile(0.99, rate(cache_operation_duration_seconds_bucket{operation="get"}[5m]))

# Cache size trend
cache_entries_current
```

### 14.4 Health Check Endpoint

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"

    "github.com/redis/go-redis/v9"
)

type CacheHealthChecker struct {
    rdb     *redis.Client
    metrics *CacheMetrics
}

func (h *CacheHealthChecker) HealthHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    status := "healthy"
    checks := make(map[string]interface{})

    // Check Redis connectivity
    start := time.Now()
    err := h.rdb.Ping(ctx).Err()
    redisLatency := time.Since(start)

    checks["redis"] = map[string]interface{}{
        "status":  boolToStatus(err == nil),
        "latency": redisLatency.String(),
    }
    if err != nil {
        status = "degraded"
        checks["redis"].(map[string]interface{})["error"] = err.Error()
    }

    // Check cache hit rate
    hitRate := h.metrics.HitRate()
    checks["hit_rate"] = map[string]interface{}{
        "value":  fmt.Sprintf("%.2f%%", hitRate*100),
        "status": boolToStatus(hitRate >= 0.5),
    }
    if hitRate < 0.5 {
        status = "warning"
    }

    // Check cache metrics
    checks["metrics"] = h.metrics.Snapshot()

    response := map[string]interface{}{
        "status":    status,
        "timestamp": time.Now().UTC(),
        "checks":    checks,
    }

    w.Header().Set("Content-Type", "application/json")
    if status != "healthy" {
        w.WriteHeader(http.StatusServiceUnavailable)
    }
    json.NewEncoder(w).Encode(response)
}

func boolToStatus(ok bool) string {
    if ok {
        return "ok"
    }
    return "error"
}

func main() {
    fmt.Println("Cache health check endpoint example")
}
```

---

## 15. Common Pitfalls

### 15.1 Stale Data

**Problem:** Cached data does not reflect the latest state of the source of truth.

**Scenarios:**
- Application updates the database but fails to invalidate the cache
- Multiple services update the same data but only one invalidates the cache
- Race condition: read-old-value occurs between a write and invalidation

```go
// BAD: Race condition between update and invalidation
func (r *Repo) UpdateUser(ctx context.Context, user *User) error {
    // Window of inconsistency starts here
    err := r.db.Update(ctx, user)   // 1. Update DB
    // If another request reads here, it gets stale cache
    r.cache.Delete(ctx, user.CacheKey()) // 2. Invalidate cache
    return err
}

// BETTER: Delete cache FIRST (but risks a cache miss that loads old data)
func (r *Repo) UpdateUserBetter(ctx context.Context, user *User) error {
    r.cache.Delete(ctx, user.CacheKey()) // 1. Invalidate cache
    return r.db.Update(ctx, user)        // 2. Update DB
}

// BEST: Delete cache BEFORE and AFTER the update (double invalidation)
func (r *Repo) UpdateUserBest(ctx context.Context, user *User) error {
    key := user.CacheKey()
    r.cache.Delete(ctx, key)       // 1. Pre-invalidate
    err := r.db.Update(ctx, user)  // 2. Update DB
    if err != nil {
        return err
    }
    // Small delay to let any concurrent reads that loaded old data to finish
    time.AfterFunc(500*time.Millisecond, func() {
        r.cache.Delete(context.Background(), key) // 3. Post-invalidate
    })
    return nil
}
```

### 15.2 Memory Leaks

**Problem:** Cache grows without bounds, eventually consuming all available memory.

```go
// BAD: No size limit, no TTL, no eviction
var cache = make(map[string]interface{})

func Get(key string) interface{} {
    return cache[key] // Grows forever!
}

// GOOD: Bounded cache with TTL
// Use one of: LRU cache with max size, bigcache with HardMaxCacheSize,
// ristretto with MaxCost, or freecache with fixed memory allocation.
```

**Prevention checklist:**
- Always set a maximum size or memory limit
- Always set a TTL (even a long one like 24 hours)
- Monitor memory usage in production
- Test with realistic data volumes

### 15.3 Serialization Overhead

**Problem:** JSON marshaling/unmarshaling adds significant overhead for high-throughput caches.

```go
package main

import (
    "encoding/gob"
    "encoding/json"
    "bytes"
    "fmt"
    "time"
)

type LargeStruct struct {
    ID       int
    Name     string
    Tags     []string
    Metadata map[string]string
}

func benchmarkSerialization() {
    data := LargeStruct{
        ID:   1,
        Name: "test",
        Tags: []string{"a", "b", "c", "d", "e"},
        Metadata: map[string]string{
            "key1": "value1",
            "key2": "value2",
            "key3": "value3",
        },
    }

    // JSON
    start := time.Now()
    for i := 0; i < 100000; i++ {
        encoded, _ := json.Marshal(data)
        var decoded LargeStruct
        json.Unmarshal(encoded, &decoded)
    }
    jsonDuration := time.Since(start)

    // Gob (binary, Go-native, much faster)
    start = time.Now()
    for i := 0; i < 100000; i++ {
        var buf bytes.Buffer
        gob.NewEncoder(&buf).Encode(data)
        var decoded LargeStruct
        gob.NewDecoder(&buf).Decode(&decoded)
    }
    gobDuration := time.Since(start)

    fmt.Printf("JSON:  %v\n", jsonDuration)
    fmt.Printf("Gob:   %v\n", gobDuration)
    fmt.Printf("Speedup: %.1fx\n", float64(jsonDuration)/float64(gobDuration))
}

func main() {
    benchmarkSerialization()
}
```

**Alternatives to JSON for cache serialization:**

| Format     | Speed     | Size     | Cross-Language | Readability |
|------------|-----------|----------|----------------|-------------|
| JSON       | Slow      | Large    | Yes            | Yes         |
| Gob        | Fast      | Medium   | No (Go only)   | No          |
| MessagePack| Fast      | Small    | Yes            | No          |
| Protocol Buffers | Very Fast | Very Small | Yes      | No          |
| FlatBuffers| Fastest   | Smallest | Yes            | No          |

> **Tip:** For in-process caches, store values as-is (no serialization needed). Serialization is only required when crossing process boundaries (Redis, memcached) or when using libraries like `bigcache` that store `[]byte`.

### 15.4 Cache Penetration

**Problem:** Repeated requests for keys that do not exist in either the cache or the database. Every request goes to the database.

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// Solution 1: Cache negative results (null/empty markers)
func GetWithNegativeCache(ctx context.Context, rdb *redis.Client, key string, loader func() (string, error)) (string, bool, error) {
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        if val == "__NULL__" {
            return "", false, nil // Cached negative result
        }
        return val, true, nil
    }

    result, err := loader()
    if err != nil {
        return "", false, err
    }

    if result == "" {
        // Cache the "not found" result with a shorter TTL
        rdb.Set(ctx, key, "__NULL__", 2*time.Minute)
        return "", false, nil
    }

    rdb.Set(ctx, key, result, 15*time.Minute)
    return result, true, nil
}

// Solution 2: Bloom filter to reject impossible keys
// (conceptual -- use a library like github.com/bits-and-blooms/bloom)
type BloomFilter struct {
    // In practice, use a proper implementation
    knownKeys map[string]bool
}

func (bf *BloomFilter) MightExist(key string) bool {
    // A real bloom filter would use hash functions and a bit array.
    // False positives are possible, but false negatives are not.
    _, ok := bf.knownKeys[key]
    return ok
}

func GetWithBloomFilter(ctx context.Context, rdb *redis.Client, bf *BloomFilter, key string, loader func() (string, error)) (string, error) {
    // Fast rejection of impossible keys
    if !bf.MightExist(key) {
        return "", fmt.Errorf("key not found")
    }

    // Proceed with normal cache-aside pattern
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return val, nil
    }

    return loader()
}

func main() {
    fmt.Println("Cache penetration prevention examples")
}
```

### 15.5 Dog-Piling on Cold Start

**Problem:** After a deployment or restart, the cache is empty and all requests hit the database simultaneously.

**Solutions:**
- **Cache warming** (see Section 11)
- **Staggered deployments** -- roll out one instance at a time so the cache is warm on most instances
- **Persistent cache** -- use Redis (survives restarts) instead of purely in-process cache
- **Rate limiting on cache miss** -- use a semaphore to limit concurrent database queries

```go
package main

import (
    "context"
    "fmt"
    "time"
)

type RateLimitedLoader struct {
    sem chan struct{}
}

func NewRateLimitedLoader(maxConcurrent int) *RateLimitedLoader {
    return &RateLimitedLoader{
        sem: make(chan struct{}, maxConcurrent),
    }
}

func (l *RateLimitedLoader) Load(ctx context.Context, loader func(ctx context.Context) (interface{}, error)) (interface{}, error) {
    select {
    case l.sem <- struct{}{}: // acquire semaphore
        defer func() { <-l.sem }() // release semaphore
        return loader(ctx)
    case <-ctx.Done():
        return nil, ctx.Err()
    case <-time.After(5 * time.Second):
        return nil, fmt.Errorf("timeout waiting for database access slot")
    }
}

func main() {
    // Allow at most 10 concurrent database queries during cold start
    limiter := NewRateLimitedLoader(10)

    ctx := context.Background()
    result, err := limiter.Load(ctx, func(ctx context.Context) (interface{}, error) {
        return "loaded from DB", nil
    })
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
}
```

### 15.6 Incorrect TTL Values

| TTL Too Short                          | TTL Too Long                           |
|----------------------------------------|----------------------------------------|
| Low hit rate                           | High staleness                         |
| Increased database load                | Wasted memory on stale entries         |
| More cache misses                      | Delayed propagation of changes         |
| Higher latency (more rebuilds)         | Users see outdated data                |

**Guidelines for choosing TTL:**

```
Data Type               | Recommended TTL    | Reasoning
------------------------|--------------------|---------------------------------
Configuration/settings  | 1-24 hours         | Changes very infrequently
User profiles           | 5-30 minutes       | Changes occasionally
Product listings        | 1-10 minutes       | Changes moderately
Search results          | 30s-5 minutes      | Freshness matters
Real-time feeds         | 5-30 seconds       | Must be near-real-time
Session data            | Match session TTL   | Must match session lifetime
Static assets           | 1 year + immutable  | Content-addressed URLs
```

### 15.7 Thundering Herd at Expiration

Already covered in Section 9, but worth reiterating: when a popular key expires, hundreds of concurrent requests may all try to rebuild it simultaneously.

**Defense in depth:**
1. Use `singleflight` to deduplicate within a process
2. Use distributed locks to deduplicate across processes
3. Use TTL jitter to stagger expirations
4. Use early/probabilistic refresh to avoid actual expiration
5. Use stale-while-revalidate to serve stale data while refreshing

### 15.8 Key Design Anti-Patterns

```go
// BAD: Key is too generic -- collision risk
cache.Set("user", userData, ttl)

// BAD: Key includes mutable data
cache.Set(fmt.Sprintf("user:%s", user.Email), userData, ttl) // email can change

// BAD: Key is too long (wastes memory, slower lookups)
cache.Set("application:service:v2:users:active:by-id:12345:full-profile:with-preferences", data, ttl)

// GOOD: Namespace + entity + immutable ID
cache.Set("myapp:user:12345", userData, ttl)

// GOOD: Include version for schema changes
cache.Set("myapp:v2:user:12345", userData, ttl)

// GOOD: Include query parameters for different views
cache.Set("myapp:products:category=electronics:page=1:limit=20", data, ttl)
```

### 15.9 Not Handling Cache Failures Gracefully

```go
// BAD: Cache error kills the request
func GetUser(ctx context.Context, id int) (*User, error) {
    data, err := cache.Get(ctx, fmt.Sprintf("user:%d", id))
    if err != nil {
        return nil, err // Redis down? All requests fail!
    }
    // ...
}

// GOOD: Cache is optional -- fall back to database on any cache error
func GetUser(ctx context.Context, id int) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    // Try cache (non-fatal errors)
    data, err := cache.Get(ctx, key)
    if err == nil {
        var user User
        if json.Unmarshal(data, &user) == nil {
            return &user, nil
        }
    }
    if err != nil && err != redis.Nil {
        log.Printf("cache error (falling back to DB): %v", err)
    }

    // Always fall back to database
    user, err := db.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }

    // Best-effort cache population
    if encoded, err := json.Marshal(user); err == nil {
        if cacheErr := cache.Set(ctx, key, encoded, 15*time.Minute); cacheErr != nil {
            log.Printf("cache set error (non-fatal): %v", cacheErr)
        }
    }

    return user, nil
}
```

### 15.10 Race Condition: Read-Your-Write Inconsistency

After a write, the same user may read stale data from cache. This is especially confusing for the user who just made the change.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Solution: Per-request cache bypass for recent writers

type RequestContext struct {
    UserID          int
    BypassCacheFor  map[string]time.Time // keys that were recently written
}

func ShouldBypassCache(ctx context.Context, key string) bool {
    rc, ok := ctx.Value("request_context").(*RequestContext)
    if !ok {
        return false
    }

    if writeTime, exists := rc.BypassCacheFor[key]; exists {
        // Bypass cache for 5 seconds after a write
        if time.Since(writeTime) < 5*time.Second {
            return true
        }
        delete(rc.BypassCacheFor, key)
    }
    return false
}

func main() {
    fmt.Println("Read-your-write consistency example")
}
```

---

## 16. Summary

### Decision Matrix

| Scenario                        | Recommended Approach                                |
|---------------------------------|----------------------------------------------------|
| Simple key-value, few entries   | `sync.Map` or `map` + `sync.RWMutex`              |
| Millions of entries, low GC     | `bigcache` or `freecache`                          |
| Best hit ratio                  | `ristretto` (TinyLFU)                              |
| Distributed cache               | Redis or `groupcache`                               |
| High-frequency stampede risk    | `singleflight` + early expiration                  |
| Multi-instance consistency      | Redis + Pub/Sub invalidation                        |
| Read-heavy, rare writes         | Read-through with long TTL                          |
| Write-heavy                     | Write-behind with batch flushes                     |
| Cross-service shared state      | Redis Cluster with consistent hashing               |

### Caching Checklist

Before deploying caching to production, verify:

- [ ] **Bounded size** -- cache has a maximum size or memory limit
- [ ] **TTL set** -- every entry has a time-to-live
- [ ] **Eviction policy** -- appropriate for your access pattern (LRU, LFU, etc.)
- [ ] **Stampede protection** -- `singleflight` or distributed locks in place
- [ ] **Graceful degradation** -- application works (slower) if cache is unavailable
- [ ] **Invalidation strategy** -- clear plan for when and how to invalidate
- [ ] **Monitoring** -- hit rate, miss rate, latency, evictions tracked
- [ ] **Alerting** -- alerts on low hit rate, high eviction rate, cache errors
- [ ] **Key design** -- namespaced, versioned, using immutable identifiers
- [ ] **Serialization** -- efficient format (avoid JSON for high-throughput)
- [ ] **Cache warming** -- plan for cold starts after deployments
- [ ] **Memory monitoring** -- cache memory usage tracked and bounded
- [ ] **Error handling** -- cache failures do not cascade to application failures

### Key Takeaways

1. **Start simple.** A `map` with `sync.RWMutex` and TTL is often sufficient. Reach for libraries only when you hit limitations.

2. **Measure first.** Do not cache speculatively. Profile your application, identify hot paths, and cache only where it provides measurable benefit.

3. **TTL is your safety net.** Even with event-based invalidation, always set a TTL. It is your last line of defense against stale data.

4. **singleflight is almost always worth it.** If you have a cache, you should probably have `singleflight` protecting your cache-miss path. The cost is minimal and the benefit is significant.

5. **Cache is not the source of truth.** Always be prepared for the cache to be empty, stale, or unavailable. The application should always be able to fall back to the source of truth.

6. **Monitor aggressively.** A cache with a 20% hit rate is worse than no cache at all (added complexity with minimal benefit). Target 80%+ hit rates.

7. **Think in tiers.** L1 (in-process) for the hottest data, L2 (Redis) for shared state. This gives you both speed and consistency.

---

### Further Reading

- [groupcache](https://github.com/golang/groupcache) -- Brad Fitzpatrick's distributed cache
- [bigcache](https://github.com/allegro/bigcache) -- Fast, concurrent, evicting cache
- [ristretto](https://github.com/dgraph-io/ristretto) -- High-performance cache with TinyLFU
- [freecache](https://github.com/coocood/freecache) -- Zero GC cache
- [go-redis](https://github.com/redis/go-redis) -- Redis client for Go
- [singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight) -- Duplicate function call suppression
- [TinyLFU paper](https://arxiv.org/abs/1512.00727) -- The admission policy used by ristretto
- [ARC paper](https://www.usenix.org/conference/fast-03/arc-self-tuning-low-overhead-replacement-cache) -- Adaptive Replacement Cache
