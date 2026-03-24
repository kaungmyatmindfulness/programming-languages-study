# Chapter 35: Data Replication Patterns

> **Level: Distributed Systems / Applied** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 16 (Context), 17 (Advanced Concurrency), 31 (Concurrency in Web Applications)**

This chapter explores how data is replicated across multiple nodes for availability, durability, and performance. The concepts come directly from Chapter 5 of Martin Kleppmann's *Designing Data-Intensive Applications* (DDIA), but every pattern is implemented in Go using goroutines, channels, and mutexes to simulate distributed nodes within a single process.

Replication is not optional in production systems. Every serious database -- PostgreSQL, MySQL, MongoDB, Cassandra, DynamoDB -- uses replication. Understanding *how* replication works, what guarantees it provides, and where it breaks down is essential for building and operating data-intensive applications.

By the end of this chapter you will understand single-leader, multi-leader, and leaderless replication architectures. You will implement write-ahead logs, failover managers, quorum reads and writes, vector clocks, CRDTs, and Merkle trees -- all in runnable Go code.

---

## Table of Contents

1. [Why Replicate Data?](#1-why-replicate-data)
2. [Leader-Follower Replication](#2-leader-follower-replication)
3. [Replication Log Implementation](#3-replication-log-implementation)
4. [Handling Follower Failures](#4-handling-follower-failures)
5. [Handling Leader Failures (Failover)](#5-handling-leader-failures-failover)
6. [Replication Lag Problems](#6-replication-lag-problems)
7. [Read-After-Write Consistency](#7-read-after-write-consistency)
8. [Multi-Leader Replication](#8-multi-leader-replication)
9. [Conflict Resolution Strategies](#9-conflict-resolution-strategies)
10. [Leaderless Replication](#10-leaderless-replication)
11. [Quorum Reads and Writes](#11-quorum-reads-and-writes)
12. [Vector Clocks](#12-vector-clocks)
13. [Anti-Entropy and Merkle Trees](#13-anti-entropy-and-merkle-trees)
14. [Real-World Example: Building a Replicated Key-Value Store](#14-real-world-example-building-a-replicated-key-value-store)
15. [Key Takeaways](#15-key-takeaways)
16. [Practice Exercises](#16-practice-exercises)

---

## 1. Why Replicate Data?

### The Problem

A single database server is a single point of failure. If it goes down, your entire application goes down with it. Even if it stays up, a single server can only handle so many read queries per second, and users in distant geographic regions experience high latency.

Replication solves these problems by keeping copies of the same data on multiple machines. Each machine that stores a copy is called a **replica** or **node**.

### Four Reasons to Replicate

| Reason | Description |
|--------|-------------|
| **High availability** | If one node fails, others can continue serving requests. The system stays up. |
| **Fault tolerance** | Data survives hardware failures. If a disk dies, other replicas still have the data. |
| **Latency reduction** | Place replicas geographically close to users. A user in Tokyo reads from a Tokyo replica instead of a server in Virginia. |
| **Read scaling** | Distribute read queries across multiple replicas. One leader handles writes; many followers handle reads. |

### The Fundamental Difficulty

If data never changed, replication would be trivial -- just copy the data once to every node. The difficulty lies in handling **changes** to the data. When a user writes new data or updates existing data, that change must propagate to all replicas. The three major approaches differ in how they handle this:

1. **Single-leader replication** -- one node accepts writes, propagates to followers
2. **Multi-leader replication** -- multiple nodes accept writes, synchronize with each other
3. **Leaderless replication** -- any node accepts writes, quorum determines success

Let us start with a simple demonstration of why replication matters.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// SingleServer represents a database with no replication.
// If it goes down, all data access stops.
type SingleServer struct {
	data    map[string]string
	mu      sync.RWMutex
	healthy bool
}

func NewSingleServer() *SingleServer {
	return &SingleServer{
		data:    make(map[string]string),
		healthy: true,
	}
}

func (s *SingleServer) Write(key, value string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if !s.healthy {
		return fmt.Errorf("server is down -- data unavailable")
	}
	s.data[key] = value
	return nil
}

func (s *SingleServer) Read(key string) (string, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if !s.healthy {
		return "", fmt.Errorf("server is down -- data unavailable")
	}
	val, ok := s.data[key]
	if !ok {
		return "", fmt.Errorf("key %q not found", key)
	}
	return val, nil
}

func (s *SingleServer) Crash() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.healthy = false
}

// ReplicatedServers represents three replicas of the same data.
// If one goes down, the others can still serve reads.
type ReplicatedServers struct {
	replicas []*SingleServer
}

func NewReplicatedServers(n int) *ReplicatedServers {
	rs := &ReplicatedServers{}
	for i := 0; i < n; i++ {
		rs.replicas = append(rs.replicas, NewSingleServer())
	}
	return rs
}

func (rs *ReplicatedServers) Write(key, value string) error {
	// Write to all replicas (simplified -- real systems are more nuanced)
	var lastErr error
	successCount := 0
	for _, r := range rs.replicas {
		if err := r.Write(key, value); err != nil {
			lastErr = err
		} else {
			successCount++
		}
	}
	if successCount == 0 {
		return fmt.Errorf("all replicas down: %w", lastErr)
	}
	return nil
}

func (rs *ReplicatedServers) Read(key string) (string, error) {
	// Read from any healthy replica
	for _, r := range rs.replicas {
		val, err := r.Read(key)
		if err == nil {
			return val, nil
		}
	}
	return "", fmt.Errorf("no healthy replica has key %q", key)
}

func main() {
	fmt.Println("=== Single Server (No Replication) ===")
	single := NewSingleServer()
	_ = single.Write("user:1", "Alice")
	val, _ := single.Read("user:1")
	fmt.Printf("Before crash: user:1 = %s\n", val)

	single.Crash()
	_, err := single.Read("user:1")
	fmt.Printf("After crash:  %v\n\n", err)

	fmt.Println("=== Replicated Servers (3 Replicas) ===")
	replicated := NewReplicatedServers(3)
	_ = replicated.Write("user:1", "Alice")
	val, _ = replicated.Read("user:1")
	fmt.Printf("Before crash: user:1 = %s\n", val)

	// Crash two out of three replicas
	replicated.replicas[0].Crash()
	replicated.replicas[1].Crash()
	time.Sleep(10 * time.Millisecond)

	val, err = replicated.Read("user:1")
	if err != nil {
		fmt.Printf("After 2 crashes: %v\n", err)
	} else {
		fmt.Printf("After 2 crashes: user:1 = %s (still available!)\n", val)
	}

	// Crash the last replica
	replicated.replicas[2].Crash()
	_, err = replicated.Read("user:1")
	fmt.Printf("After all crash: %v\n", err)
}
```

**Output:**

```
=== Single Server (No Replication) ===
Before crash: user:1 = Alice
After crash:  server is down -- data unavailable

=== Replicated Servers (3 Replicas) ===
Before crash: user:1 = Alice
After 2 crashes: user:1 = Alice (still available!)
After all crash: no healthy replica has key "user:1"
```

The replicated system survived two out of three servers crashing. This is the power of replication. Now let us build it properly.

---

## 2. Leader-Follower Replication

### How It Works

Leader-follower (also called master-slave or primary-secondary) replication is the most common replication strategy. It works as follows:

1. One replica is designated the **leader** (primary). All write requests go to the leader.
2. The other replicas are **followers** (secondaries). When the leader writes new data, it sends the change to all followers via a **replication log**.
3. Clients can read from the leader or any follower. Reads from followers may be slightly stale.

This is the default replication mode in PostgreSQL, MySQL, MongoDB, and many others.

### Synchronous vs Asynchronous Replication

- **Synchronous**: The leader waits for at least one follower to confirm it received the write before reporting success to the client. Guarantees the follower has an up-to-date copy, but adds latency.
- **Asynchronous**: The leader reports success immediately after writing locally. Followers receive changes eventually. Faster, but a leader crash can lose recent writes.
- **Semi-synchronous**: One follower is synchronous (guaranteeing durability on at least two nodes), the rest are asynchronous. This is what PostgreSQL calls "synchronous replication" in practice.

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// VersionedValue stores a value with metadata for replication tracking.
type VersionedValue struct {
	Value     string
	Version   int64
	Timestamp time.Time
}

// LogEntry represents a single write in the replication log.
type LogEntry struct {
	Offset    int64
	Operation string // "SET" or "DELETE"
	Key       string
	Value     string
	Timestamp time.Time
}

// Node represents a single database replica.
type Node struct {
	id       string
	data     map[string]VersionedValue
	log      []LogEntry
	offset   int64
	isLeader bool
	healthy  bool
	mu       sync.RWMutex

	// Simulated network latency for this node
	networkDelay time.Duration
}

func NewNode(id string, networkDelay time.Duration) *Node {
	return &Node{
		id:           id,
		data:         make(map[string]VersionedValue),
		log:          make([]LogEntry, 0),
		healthy:      true,
		networkDelay: networkDelay,
	}
}

func (n *Node) ApplyEntry(entry LogEntry) {
	n.mu.Lock()
	defer n.mu.Unlock()

	// Simulate network delay
	time.Sleep(n.networkDelay)

	n.log = append(n.log, entry)
	n.offset = entry.Offset

	switch entry.Operation {
	case "SET":
		n.data[entry.Key] = VersionedValue{
			Value:     entry.Value,
			Version:   entry.Offset,
			Timestamp: entry.Timestamp,
		}
	case "DELETE":
		delete(n.data, entry.Key)
	}
}

func (n *Node) Read(key string) (VersionedValue, bool) {
	n.mu.RLock()
	defer n.mu.RUnlock()
	val, ok := n.data[key]
	return val, ok
}

func (n *Node) GetOffset() int64 {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.offset
}

// ReplicationMode determines how the leader waits for followers.
type ReplicationMode int

const (
	Async ReplicationMode = iota
	SemiSync
	FullSync
)

func (m ReplicationMode) String() string {
	switch m {
	case Async:
		return "async"
	case SemiSync:
		return "semi-sync"
	case FullSync:
		return "full-sync"
	default:
		return "unknown"
	}
}

// ReplicaSet manages a leader and its followers.
type ReplicaSet struct {
	leader    *Node
	followers []*Node
	mode      ReplicationMode
	nextOff   int64
	mu        sync.Mutex
}

func NewReplicaSet(leader *Node, followers []*Node, mode ReplicationMode) *ReplicaSet {
	leader.isLeader = true
	return &ReplicaSet{
		leader:    leader,
		followers: followers,
		mode:      mode,
	}
}

// Write processes a write request. Behavior depends on the replication mode.
func (rs *ReplicaSet) Write(key, value string) error {
	rs.mu.Lock()
	rs.nextOff++
	entry := LogEntry{
		Offset:    rs.nextOff,
		Operation: "SET",
		Key:       key,
		Value:     value,
		Timestamp: time.Now(),
	}
	rs.mu.Unlock()

	// Step 1: Apply to leader first
	rs.leader.ApplyEntry(entry)

	// Step 2: Replicate to followers based on mode
	switch rs.mode {
	case Async:
		// Fire and forget -- replicate in background
		for _, f := range rs.followers {
			go func(follower *Node) {
				follower.ApplyEntry(entry)
			}(f)
		}

	case SemiSync:
		// Wait for at least one follower to confirm
		done := make(chan struct{}, len(rs.followers))
		for _, f := range rs.followers {
			go func(follower *Node) {
				follower.ApplyEntry(entry)
				done <- struct{}{}
			}(f)
		}
		// Wait for first follower to finish
		<-done
		// Let remaining followers replicate in background (they will finish eventually)

	case FullSync:
		// Wait for ALL followers to confirm
		var wg sync.WaitGroup
		for _, f := range rs.followers {
			wg.Add(1)
			go func(follower *Node) {
				defer wg.Done()
				follower.ApplyEntry(entry)
			}(f)
		}
		wg.Wait()
	}

	return nil
}

// ReadFromLeader always reads from the leader (strongly consistent).
func (rs *ReplicaSet) ReadFromLeader(key string) (VersionedValue, bool) {
	return rs.leader.Read(key)
}

// ReadFromFollower reads from a random follower (may be stale).
func (rs *ReplicaSet) ReadFromFollower(key string) (VersionedValue, bool, string) {
	idx := rand.Intn(len(rs.followers))
	follower := rs.followers[idx]
	val, ok := follower.Read(key)
	return val, ok, follower.id
}

func main() {
	fmt.Println("=== Leader-Follower Replication Demo ===\n")

	// Create nodes with varying simulated network delays
	leader := NewNode("leader", 1*time.Millisecond)
	follower1 := NewNode("follower-1", 5*time.Millisecond)  // Fast follower
	follower2 := NewNode("follower-2", 50*time.Millisecond)  // Slow follower
	follower3 := NewNode("follower-3", 100*time.Millisecond) // Very slow follower

	followers := []*Node{follower1, follower2, follower3}

	// --- Test Async Replication ---
	fmt.Println("--- Async Replication ---")
	rs := NewReplicaSet(leader, followers, Async)

	start := time.Now()
	_ = rs.Write("user:1", "Alice")
	writeTime := time.Since(start)
	fmt.Printf("Write completed in %v (leader only, did not wait for followers)\n", writeTime)

	// Read immediately -- followers may not have the data yet
	val, ok := rs.ReadFromLeader("user:1")
	fmt.Printf("Leader read:   ok=%v value=%q\n", ok, val.Value)

	// Followers may or may not have it depending on timing
	for _, f := range followers {
		v, exists := f.Read("user:1")
		fmt.Printf("  %s: ok=%v value=%q offset=%d\n", f.id, exists, v.Value, f.GetOffset())
	}

	// Wait for all followers to catch up
	time.Sleep(200 * time.Millisecond)
	fmt.Println("\nAfter waiting 200ms:")
	for _, f := range followers {
		v, exists := f.Read("user:1")
		fmt.Printf("  %s: ok=%v value=%q offset=%d\n", f.id, exists, v.Value, f.GetOffset())
	}

	// --- Test Sync Replication ---
	fmt.Println("\n--- Full-Sync Replication ---")
	leader2 := NewNode("leader-2", 1*time.Millisecond)
	syncFollower1 := NewNode("sync-f1", 5*time.Millisecond)
	syncFollower2 := NewNode("sync-f2", 50*time.Millisecond)
	syncFollower3 := NewNode("sync-f3", 100*time.Millisecond)

	syncFollowers := []*Node{syncFollower1, syncFollower2, syncFollower3}
	rsSync := NewReplicaSet(leader2, syncFollowers, FullSync)

	start = time.Now()
	_ = rsSync.Write("user:2", "Bob")
	writeTime = time.Since(start)
	fmt.Printf("Write completed in %v (waited for ALL followers)\n", writeTime)

	// All followers guaranteed to have the data
	for _, f := range syncFollowers {
		v, exists := f.Read("user:2")
		fmt.Printf("  %s: ok=%v value=%q offset=%d\n", f.id, exists, v.Value, f.GetOffset())
	}

	// --- Test Semi-Sync Replication ---
	fmt.Println("\n--- Semi-Sync Replication ---")
	leader3 := NewNode("leader-3", 1*time.Millisecond)
	semiF1 := NewNode("semi-f1", 5*time.Millisecond)
	semiF2 := NewNode("semi-f2", 200*time.Millisecond)

	semiFollowers := []*Node{semiF1, semiF2}
	rsSemi := NewReplicaSet(leader3, semiFollowers, SemiSync)

	start = time.Now()
	_ = rsSemi.Write("user:3", "Charlie")
	writeTime = time.Since(start)
	fmt.Printf("Write completed in %v (waited for fastest follower only)\n", writeTime)
	fmt.Printf("  Leader offset:  %d\n", leader3.GetOffset())
	fmt.Printf("  semi-f1 offset: %d\n", semiF1.GetOffset())
	fmt.Printf("  semi-f2 offset: %d (may still be replicating)\n", semiF2.GetOffset())
}
```

**Key observations:**

- Async writes return almost instantly (leader latency only) but followers may be stale.
- Full-sync writes are as slow as the slowest follower, but all replicas are guaranteed up-to-date.
- Semi-sync is the practical middle ground -- one follower is guaranteed to have the data, giving you durability on at least two nodes without waiting for every follower.

---

## 3. Replication Log Implementation

### How Changes Propagate

The leader must send data changes to followers. There are several strategies:

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Statement-based** | Send the SQL statement (e.g., `UPDATE users SET name='Bob' WHERE id=1`) | Simple | Non-deterministic functions (`NOW()`, `RAND()`) produce different results on each replica |
| **Write-ahead log (WAL) shipping** | Send the physical storage engine's write-ahead log | Exact byte-level copy | Coupled to storage engine internals; cannot run different versions |
| **Row-based (logical) log** | Send a logical description of the change (which row, old/new column values) | Decoupled from storage engine, supports change data capture | More verbose |

In practice, most modern systems use logical (row-based) replication. PostgreSQL's logical replication, MySQL's row-based binlog, and MongoDB's oplog all follow this pattern.

```go
package main

import (
	"encoding/json"
	"fmt"
	"sync"
	"time"
)

// --- Statement-Based Replication ---

// Statement represents a raw SQL-like statement.
type Statement struct {
	SQL       string
	Timestamp time.Time
}

// StatementLog ships raw statements to followers.
type StatementLog struct {
	entries []Statement
	mu      sync.Mutex
}

func (sl *StatementLog) Append(sql string) {
	sl.mu.Lock()
	defer sl.mu.Unlock()
	sl.entries = append(sl.entries, Statement{
		SQL:       sql,
		Timestamp: time.Now(),
	})
}

func (sl *StatementLog) GetEntries(fromIndex int) []Statement {
	sl.mu.Lock()
	defer sl.mu.Unlock()
	if fromIndex >= len(sl.entries) {
		return nil
	}
	result := make([]Statement, len(sl.entries)-fromIndex)
	copy(result, sl.entries[fromIndex:])
	return result
}

// --- Write-Ahead Log (WAL) ---

// WALEntry represents a physical storage change.
type WALEntry struct {
	LSN       int64  // Log Sequence Number
	TableID   uint32
	PageNum   uint32
	Offset    uint16
	OldBytes  []byte
	NewBytes  []byte
	Timestamp time.Time
}

// WriteAheadLog maintains a sequence of physical changes.
type WriteAheadLog struct {
	entries []WALEntry
	nextLSN int64
	mu      sync.Mutex
}

func NewWriteAheadLog() *WriteAheadLog {
	return &WriteAheadLog{nextLSN: 1}
}

func (wal *WriteAheadLog) Append(tableID uint32, pageNum uint32, offset uint16, oldBytes, newBytes []byte) int64 {
	wal.mu.Lock()
	defer wal.mu.Unlock()
	lsn := wal.nextLSN
	wal.nextLSN++
	wal.entries = append(wal.entries, WALEntry{
		LSN:       lsn,
		TableID:   tableID,
		PageNum:   pageNum,
		Offset:    offset,
		OldBytes:  oldBytes,
		NewBytes:  newBytes,
		Timestamp: time.Now(),
	})
	return lsn
}

func (wal *WriteAheadLog) GetEntriesSince(lsn int64) []WALEntry {
	wal.mu.Lock()
	defer wal.mu.Unlock()
	var result []WALEntry
	for _, e := range wal.entries {
		if e.LSN >= lsn {
			result = append(result, e)
		}
	}
	return result
}

// --- Row-Based (Logical) Replication ---

// ChangeType represents the kind of data change.
type ChangeType string

const (
	Insert ChangeType = "INSERT"
	Update ChangeType = "UPDATE"
	Delete ChangeType = "DELETE"
)

// RowChange represents a logical change to a single row.
type RowChange struct {
	LSN       int64                  `json:"lsn"`
	Type      ChangeType             `json:"type"`
	Table     string                 `json:"table"`
	Key       string                 `json:"key"`
	OldValues map[string]interface{} `json:"old_values,omitempty"` // For UPDATE and DELETE
	NewValues map[string]interface{} `json:"new_values,omitempty"` // For INSERT and UPDATE
	Timestamp time.Time              `json:"timestamp"`
}

// LogicalLog maintains a sequence of logical row changes.
type LogicalLog struct {
	entries []RowChange
	nextLSN int64
	mu      sync.Mutex
}

func NewLogicalLog() *LogicalLog {
	return &LogicalLog{nextLSN: 1}
}

func (ll *LogicalLog) AppendInsert(table, key string, values map[string]interface{}) int64 {
	ll.mu.Lock()
	defer ll.mu.Unlock()
	lsn := ll.nextLSN
	ll.nextLSN++
	ll.entries = append(ll.entries, RowChange{
		LSN:       lsn,
		Type:      Insert,
		Table:     table,
		Key:       key,
		NewValues: values,
		Timestamp: time.Now(),
	})
	return lsn
}

func (ll *LogicalLog) AppendUpdate(table, key string, oldValues, newValues map[string]interface{}) int64 {
	ll.mu.Lock()
	defer ll.mu.Unlock()
	lsn := ll.nextLSN
	ll.nextLSN++
	ll.entries = append(ll.entries, RowChange{
		LSN:       lsn,
		Type:      Update,
		Table:     table,
		Key:       key,
		OldValues: oldValues,
		NewValues: newValues,
		Timestamp: time.Now(),
	})
	return lsn
}

func (ll *LogicalLog) AppendDelete(table, key string, oldValues map[string]interface{}) int64 {
	ll.mu.Lock()
	defer ll.mu.Unlock()
	lsn := ll.nextLSN
	ll.nextLSN++
	ll.entries = append(ll.entries, RowChange{
		LSN:       lsn,
		Type:      Delete,
		Table:     table,
		Key:       key,
		OldValues: oldValues,
		Timestamp: time.Now(),
	})
	return lsn
}

func (ll *LogicalLog) GetEntriesSince(lsn int64) []RowChange {
	ll.mu.Lock()
	defer ll.mu.Unlock()
	var result []RowChange
	for _, e := range ll.entries {
		if e.LSN >= lsn {
			result = append(result, e)
		}
	}
	return result
}

// --- Follower that applies logical log entries ---

type LogicalFollower struct {
	id        string
	data      map[string]map[string]interface{} // table -> key -> values
	lastLSN   int64
	mu        sync.RWMutex
}

func NewLogicalFollower(id string) *LogicalFollower {
	return &LogicalFollower{
		id:   id,
		data: make(map[string]map[string]interface{}),
	}
}

func (lf *LogicalFollower) Apply(changes []RowChange) {
	lf.mu.Lock()
	defer lf.mu.Unlock()
	for _, change := range changes {
		if _, ok := lf.data[change.Table]; !ok {
			lf.data[change.Table] = make(map[string]interface{})
		}
		switch change.Type {
		case Insert, Update:
			lf.data[change.Table][change.Key] = change.NewValues
		case Delete:
			delete(lf.data[change.Table], change.Key)
		}
		lf.lastLSN = change.LSN
	}
}

func (lf *LogicalFollower) Get(table, key string) (interface{}, bool) {
	lf.mu.RLock()
	defer lf.mu.RUnlock()
	t, ok := lf.data[table]
	if !ok {
		return nil, false
	}
	v, ok := t[key]
	return v, ok
}

func main() {
	fmt.Println("=== Replication Log Strategies ===\n")

	// --- Statement-Based ---
	fmt.Println("--- Statement-Based Replication ---")
	stmtLog := &StatementLog{}
	stmtLog.Append("INSERT INTO users (id, name) VALUES (1, 'Alice')")
	stmtLog.Append("UPDATE users SET name = 'Alicia' WHERE id = 1")
	stmtLog.Append("INSERT INTO users (id, name) VALUES (2, 'Bob')")

	entries := stmtLog.GetEntries(0)
	for _, e := range entries {
		fmt.Printf("  Statement: %s\n", e.SQL)
	}
	fmt.Println("  Problem: What if a statement uses NOW()? Each replica gets a different time.\n")

	// --- WAL ---
	fmt.Println("--- Write-Ahead Log (WAL) Shipping ---")
	wal := NewWriteAheadLog()
	lsn1 := wal.Append(1, 42, 0, nil, []byte{0x01, 0x02, 0x03})
	lsn2 := wal.Append(1, 42, 3, []byte{0x01}, []byte{0x04})
	fmt.Printf("  WAL Entry LSN=%d: tableID=1 page=42 offset=0 new=[01 02 03]\n", lsn1)
	fmt.Printf("  WAL Entry LSN=%d: tableID=1 page=42 offset=3 old=[01] new=[04]\n", lsn2)
	fmt.Println("  Problem: Coupled to storage engine internals. Cannot replicate to different version.\n")

	// --- Logical (Row-Based) ---
	fmt.Println("--- Row-Based (Logical) Replication ---")
	logLog := NewLogicalLog()

	logLog.AppendInsert("users", "1", map[string]interface{}{
		"name": "Alice", "email": "alice@example.com",
	})
	logLog.AppendUpdate("users", "1",
		map[string]interface{}{"name": "Alice", "email": "alice@example.com"},
		map[string]interface{}{"name": "Alicia", "email": "alice@example.com"},
	)
	logLog.AppendInsert("users", "2", map[string]interface{}{
		"name": "Bob", "email": "bob@example.com",
	})
	logLog.AppendDelete("users", "2", map[string]interface{}{
		"name": "Bob", "email": "bob@example.com",
	})

	// Show the log entries as JSON
	changes := logLog.GetEntriesSince(1)
	for _, c := range changes {
		jsonBytes, _ := json.Marshal(c)
		fmt.Printf("  %s\n", jsonBytes)
	}

	// Apply to a follower
	fmt.Println("\nApplying log to follower:")
	follower := NewLogicalFollower("follower-1")
	follower.Apply(changes)

	val, ok := follower.Get("users", "1")
	fmt.Printf("  users/1: ok=%v data=%v\n", ok, val)

	val, ok = follower.Get("users", "2")
	fmt.Printf("  users/2: ok=%v (was deleted)\n", ok)
	fmt.Println("  Advantage: Decoupled from storage engine. Supports change data capture (CDC).")
}
```

The logical (row-based) approach is the most flexible. It cleanly describes *what changed* without being tied to *how* the storage engine represents it internally. This is why most modern systems use it.

---

## 4. Handling Follower Failures

### Catch-Up Recovery

When a follower crashes and restarts, it knows the last log offset it successfully processed. It asks the leader for all log entries since that offset and replays them. This is called **catch-up recovery**.

If the follower has been down for a long time and the leader has discarded old log entries, the follower must start from a **snapshot** -- a consistent point-in-time copy of the leader's data -- and then replay log entries from the snapshot's offset forward.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// LogEntry represents a write in the replication log.
type LogEntry struct {
	Offset    int64
	Key       string
	Value     string
	Timestamp time.Time
}

// Snapshot represents a point-in-time copy of all data.
type Snapshot struct {
	Data   map[string]string
	Offset int64 // The log offset at the time of the snapshot
}

// LeaderNode is the primary that accepts writes and maintains the full log.
type LeaderNode struct {
	data          map[string]string
	log           []LogEntry
	nextOffset    int64
	maxLogRetain  int // Maximum number of log entries to keep
	mu            sync.RWMutex
}

func NewLeaderNode(maxLogRetain int) *LeaderNode {
	return &LeaderNode{
		data:         make(map[string]string),
		maxLogRetain: maxLogRetain,
	}
}

func (l *LeaderNode) Write(key, value string) LogEntry {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.nextOffset++
	entry := LogEntry{
		Offset:    l.nextOffset,
		Key:       key,
		Value:     value,
		Timestamp: time.Now(),
	}
	l.data[key] = value
	l.log = append(l.log, entry)

	// Trim old log entries if we exceed retention
	if len(l.log) > l.maxLogRetain {
		l.log = l.log[len(l.log)-l.maxLogRetain:]
	}
	return entry
}

// GetLogSince returns all log entries with offset > sinceOffset.
// Returns nil if the requested offset is no longer in the log.
func (l *LeaderNode) GetLogSince(sinceOffset int64) ([]LogEntry, bool) {
	l.mu.RLock()
	defer l.mu.RUnlock()

	if len(l.log) == 0 {
		return nil, true
	}

	// Check if the requested offset is still available
	oldestAvailable := l.log[0].Offset
	if sinceOffset < oldestAvailable-1 {
		// The follower is too far behind -- needs a snapshot
		return nil, false
	}

	var result []LogEntry
	for _, e := range l.log {
		if e.Offset > sinceOffset {
			result = append(result, e)
		}
	}
	return result, true
}

// TakeSnapshot creates a consistent copy of the leader's data.
func (l *LeaderNode) TakeSnapshot() Snapshot {
	l.mu.RLock()
	defer l.mu.RUnlock()
	dataCopy := make(map[string]string, len(l.data))
	for k, v := range l.data {
		dataCopy[k] = v
	}
	return Snapshot{
		Data:   dataCopy,
		Offset: l.nextOffset,
	}
}

// FollowerNode replicates data from the leader.
type FollowerNode struct {
	id       string
	data     map[string]string
	offset   int64
	healthy  bool
	mu       sync.RWMutex
}

func NewFollowerNode(id string) *FollowerNode {
	return &FollowerNode{
		id:      id,
		data:    make(map[string]string),
		healthy: true,
	}
}

func (f *FollowerNode) ApplyEntries(entries []LogEntry) {
	f.mu.Lock()
	defer f.mu.Unlock()
	for _, e := range entries {
		if e.Offset <= f.offset {
			continue // Already applied
		}
		f.data[e.Key] = e.Value
		f.offset = e.Offset
	}
}

func (f *FollowerNode) RestoreFromSnapshot(snap Snapshot) {
	f.mu.Lock()
	defer f.mu.Unlock()
	f.data = make(map[string]string, len(snap.Data))
	for k, v := range snap.Data {
		f.data[k] = v
	}
	f.offset = snap.Offset
	fmt.Printf("  [%s] Restored from snapshot at offset %d (%d keys)\n", f.id, snap.Offset, len(snap.Data))
}

func (f *FollowerNode) Crash() {
	f.mu.Lock()
	defer f.mu.Unlock()
	f.healthy = false
	fmt.Printf("  [%s] CRASHED at offset %d\n", f.id, f.offset)
}

func (f *FollowerNode) Restart() {
	f.mu.Lock()
	defer f.mu.Unlock()
	f.healthy = true
	fmt.Printf("  [%s] RESTARTED, last known offset: %d\n", f.id, f.offset)
}

func (f *FollowerNode) GetOffset() int64 {
	f.mu.RLock()
	defer f.mu.RUnlock()
	return f.offset
}

func (f *FollowerNode) GetDataCount() int {
	f.mu.RLock()
	defer f.mu.RUnlock()
	return len(f.data)
}

// CatchUp attempts to bring a follower up to date with the leader.
func CatchUp(leader *LeaderNode, follower *FollowerNode) error {
	followerOffset := follower.GetOffset()
	entries, available := leader.GetLogSince(followerOffset)

	if !available {
		// Follower is too far behind. Need snapshot + replay.
		fmt.Printf("  [%s] Log entries no longer available (offset %d too old). Taking snapshot.\n",
			follower.id, followerOffset)

		snapshot := leader.TakeSnapshot()
		follower.RestoreFromSnapshot(snapshot)

		// Now get any entries since the snapshot
		entries, _ = leader.GetLogSince(snapshot.Offset)
		if len(entries) > 0 {
			follower.ApplyEntries(entries)
			fmt.Printf("  [%s] Applied %d additional entries after snapshot\n", follower.id, len(entries))
		}
		return nil
	}

	if len(entries) > 0 {
		follower.ApplyEntries(entries)
		fmt.Printf("  [%s] Caught up by applying %d log entries (offset %d -> %d)\n",
			follower.id, len(entries), followerOffset, follower.GetOffset())
	} else {
		fmt.Printf("  [%s] Already up to date at offset %d\n", follower.id, followerOffset)
	}
	return nil
}

func main() {
	fmt.Println("=== Handling Follower Failures ===\n")

	// Leader retains only the last 5 log entries (simulating limited log retention)
	leader := NewLeaderNode(5)
	follower := NewFollowerNode("follower-1")

	// Write some initial data and replicate
	fmt.Println("--- Initial Writes ---")
	for i := 1; i <= 3; i++ {
		entry := leader.Write(fmt.Sprintf("key:%d", i), fmt.Sprintf("value-%d", i))
		follower.ApplyEntries([]LogEntry{entry})
	}
	fmt.Printf("  Leader offset: %d, Follower offset: %d, Follower keys: %d\n",
		leader.nextOffset, follower.GetOffset(), follower.GetDataCount())

	// Scenario 1: Follower crashes briefly, catches up from log
	fmt.Println("\n--- Scenario 1: Brief Crash (Catch-Up from Log) ---")
	follower.Crash()

	// Leader continues writing while follower is down
	for i := 4; i <= 6; i++ {
		leader.Write(fmt.Sprintf("key:%d", i), fmt.Sprintf("value-%d", i))
	}
	fmt.Printf("  Leader advanced to offset %d while follower was down\n", leader.nextOffset)

	follower.Restart()
	CatchUp(leader, follower)
	fmt.Printf("  Follower data count: %d\n", follower.GetDataCount())

	// Scenario 2: Follower has been down too long, log entries are gone
	fmt.Println("\n--- Scenario 2: Extended Outage (Snapshot + Replay) ---")
	follower2 := NewFollowerNode("follower-2")
	// Follower-2 only has data up to offset 2
	follower2.ApplyEntries([]LogEntry{
		{Offset: 1, Key: "key:1", Value: "value-1"},
		{Offset: 2, Key: "key:2", Value: "value-2"},
	})
	fmt.Printf("  follower-2 offset: %d\n", follower2.GetOffset())

	// Leader writes many more entries, pushing old ones out of retention
	for i := 7; i <= 15; i++ {
		leader.Write(fmt.Sprintf("key:%d", i), fmt.Sprintf("value-%d", i))
	}
	fmt.Printf("  Leader now at offset %d (only retains last 5 entries)\n", leader.nextOffset)

	// follower-2 tries to catch up from offset 2, but those entries are gone
	CatchUp(leader, follower2)
	fmt.Printf("  follower-2 offset: %d, data count: %d\n", follower2.GetOffset(), follower2.GetDataCount())
}
```

**Key insight:** The two-phase recovery approach (try log first, fall back to snapshot) is exactly what PostgreSQL, MySQL, and MongoDB do. In PostgreSQL, if a standby's requested WAL position is no longer in `pg_wal`, it must be re-created from a base backup.

---

## 5. Handling Leader Failures (Failover)

### The Failover Problem

When the leader crashes, one of the followers must be promoted to become the new leader. This is called **failover**. It is one of the hardest problems in distributed systems because of several challenges:

1. **Failure detection**: How do you know the leader is actually down, not just slow?
2. **Leader election**: Which follower should become the new leader?
3. **Split-brain**: What if the old leader comes back and thinks it is still the leader?
4. **Fencing tokens**: How do you prevent the old leader from corrupting data?

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// NodeStatus represents the current state of a node.
type NodeStatus int

const (
	StatusHealthy NodeStatus = iota
	StatusSuspect
	StatusDead
)

func (s NodeStatus) String() string {
	switch s {
	case StatusHealthy:
		return "HEALTHY"
	case StatusSuspect:
		return "SUSPECT"
	case StatusDead:
		return "DEAD"
	default:
		return "UNKNOWN"
	}
}

// NodeRole represents whether a node is a leader or follower.
type NodeRole int

const (
	RoleFollower NodeRole = iota
	RoleLeader
)

func (r NodeRole) String() string {
	if r == RoleLeader {
		return "LEADER"
	}
	return "FOLLOWER"
}

// ReplicaNode represents a node in the cluster.
type ReplicaNode struct {
	ID            string
	Role          NodeRole
	Status        NodeStatus
	Offset        int64 // Latest log offset this node has applied
	Healthy       bool
	LastHeartbeat time.Time
	FencingToken  uint64 // Must present this token to perform leader operations
	mu            sync.RWMutex
}

func NewReplicaNode(id string) *ReplicaNode {
	return &ReplicaNode{
		ID:            id,
		Role:          RoleFollower,
		Status:        StatusHealthy,
		Healthy:       true,
		LastHeartbeat: time.Now(),
	}
}

func (n *ReplicaNode) SendHeartbeat() bool {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.Healthy
}

func (n *ReplicaNode) GetOffset() int64 {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.Offset
}

func (n *ReplicaNode) SetOffset(offset int64) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.Offset = offset
}

func (n *ReplicaNode) PromoteToLeader(token uint64) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.Role = RoleLeader
	n.FencingToken = token
	fmt.Printf("    [%s] Promoted to LEADER with fencing token %d\n", n.ID, token)
}

func (n *ReplicaNode) DemoteToFollower() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.Role = RoleFollower
	n.FencingToken = 0
}

func (n *ReplicaNode) Crash() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.Healthy = false
	fmt.Printf("    [%s] CRASHED\n", n.ID)
}

func (n *ReplicaNode) Recover() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.Healthy = true
	n.Role = RoleFollower // Always come back as follower
	n.FencingToken = 0
	fmt.Printf("    [%s] RECOVERED (as follower)\n", n.ID)
}

// FailoverManager monitors the cluster and handles leader failures.
type FailoverManager struct {
	nodes            []*ReplicaNode
	currentLeader    *ReplicaNode
	fencingToken     uint64
	heartbeatTimeout time.Duration
	mu               sync.Mutex
	eventLog         []string
}

func NewFailoverManager(nodes []*ReplicaNode, heartbeatTimeout time.Duration) *FailoverManager {
	return &FailoverManager{
		nodes:            nodes,
		heartbeatTimeout: heartbeatTimeout,
	}
}

func (fm *FailoverManager) logEvent(format string, args ...interface{}) {
	msg := fmt.Sprintf(format, args...)
	fm.eventLog = append(fm.eventLog, msg)
	fmt.Printf("  [FailoverManager] %s\n", msg)
}

// SetLeader designates a node as the initial leader.
func (fm *FailoverManager) SetLeader(node *ReplicaNode) {
	fm.mu.Lock()
	defer fm.mu.Unlock()
	fm.fencingToken++
	node.PromoteToLeader(fm.fencingToken)
	fm.currentLeader = node
	fm.logEvent("Initial leader set to %s (token=%d)", node.ID, fm.fencingToken)
}

// DetectFailure checks if the current leader is responsive.
func (fm *FailoverManager) DetectFailure() bool {
	fm.mu.Lock()
	defer fm.mu.Unlock()

	if fm.currentLeader == nil {
		fm.logEvent("No leader configured")
		return true
	}

	// Send heartbeat to current leader
	responsive := fm.currentLeader.SendHeartbeat()

	if !responsive {
		fm.currentLeader.mu.Lock()
		fm.currentLeader.Status = StatusDead
		fm.currentLeader.mu.Unlock()
		fm.logEvent("Leader %s failed heartbeat check -- marked DEAD", fm.currentLeader.ID)
		return true
	}

	fm.currentLeader.mu.Lock()
	fm.currentLeader.LastHeartbeat = time.Now()
	fm.currentLeader.Status = StatusHealthy
	fm.currentLeader.mu.Unlock()
	return false
}

// ElectNewLeader selects the follower with the highest log offset.
func (fm *FailoverManager) ElectNewLeader() (*ReplicaNode, error) {
	fm.mu.Lock()
	defer fm.mu.Unlock()

	var candidates []*ReplicaNode
	for _, n := range fm.nodes {
		n.mu.RLock()
		isCandidate := n.Healthy && n.Role == RoleFollower && n.Status != StatusDead
		n.mu.RUnlock()
		if isCandidate {
			candidates = append(candidates, n)
		}
	}

	if len(candidates) == 0 {
		fm.logEvent("CRITICAL: No healthy followers available for failover!")
		return nil, fmt.Errorf("no healthy followers available")
	}

	fm.logEvent("Candidates for leader election: %d", len(candidates))
	for _, c := range candidates {
		fm.logEvent("  Candidate %s: offset=%d", c.ID, c.GetOffset())
	}

	// Select the follower with the highest offset (most up-to-date data)
	best := candidates[0]
	for _, c := range candidates[1:] {
		if c.GetOffset() > best.GetOffset() {
			best = c
		}
	}

	// Demote old leader
	if fm.currentLeader != nil {
		fm.currentLeader.DemoteToFollower()
		fm.logEvent("Demoted old leader %s to follower", fm.currentLeader.ID)
	}

	// Issue new fencing token and promote
	fm.fencingToken++
	best.PromoteToLeader(fm.fencingToken)
	fm.currentLeader = best
	fm.logEvent("New leader elected: %s (offset=%d, token=%d)", best.ID, best.GetOffset(), fm.fencingToken)

	return best, nil
}

// DetectSplitBrain checks if multiple nodes think they are the leader.
func (fm *FailoverManager) DetectSplitBrain() []string {
	fm.mu.Lock()
	defer fm.mu.Unlock()

	var leaders []string
	for _, n := range fm.nodes {
		n.mu.RLock()
		if n.Role == RoleLeader {
			leaders = append(leaders, n.ID)
		}
		n.mu.RUnlock()
	}

	if len(leaders) > 1 {
		fm.logEvent("SPLIT BRAIN DETECTED! Multiple leaders: %v", leaders)
	}
	return leaders
}

// ValidateFencingToken checks if a write operation has a valid (current) fencing token.
func (fm *FailoverManager) ValidateFencingToken(token uint64) bool {
	fm.mu.Lock()
	defer fm.mu.Unlock()
	valid := token == fm.fencingToken
	if !valid {
		fm.logEvent("REJECTED stale fencing token %d (current is %d)", token, fm.fencingToken)
	}
	return valid
}

func main() {
	fmt.Println("=== Leader Failover Demo ===\n")

	// Create a cluster of 5 nodes
	nodes := make([]*ReplicaNode, 5)
	for i := 0; i < 5; i++ {
		nodes[i] = NewReplicaNode(fmt.Sprintf("node-%d", i))
	}

	// Simulate different replication progress
	nodes[0].SetOffset(100)
	nodes[1].SetOffset(98)
	nodes[2].SetOffset(100)
	nodes[3].SetOffset(95)
	nodes[4].SetOffset(99)

	fm := NewFailoverManager(nodes, 5*time.Second)

	// Step 1: Set initial leader
	fmt.Println("--- Step 1: Set Initial Leader ---")
	fm.SetLeader(nodes[0])

	// Step 2: Leader operates normally
	fmt.Println("\n--- Step 2: Normal Operation ---")
	failed := fm.DetectFailure()
	fmt.Printf("  Failure detected: %v\n", failed)

	// Step 3: Leader crashes
	fmt.Println("\n--- Step 3: Leader Crashes ---")
	nodes[0].Crash()

	// Step 4: Detect failure
	fmt.Println("\n--- Step 4: Detect Failure ---")
	failed = fm.DetectFailure()
	fmt.Printf("  Failure detected: %v\n", failed)

	// Step 5: Elect new leader
	fmt.Println("\n--- Step 5: Elect New Leader ---")
	newLeader, err := fm.ElectNewLeader()
	if err != nil {
		fmt.Printf("  Election failed: %v\n", err)
	} else {
		fmt.Printf("  New leader: %s\n", newLeader.ID)
	}

	// Step 6: Demonstrate split-brain scenario
	fmt.Println("\n--- Step 6: Split Brain Detection ---")
	// Simulate old leader coming back and thinking it is still leader
	nodes[0].Recover()
	nodes[0].mu.Lock()
	nodes[0].Role = RoleLeader // Bug: old leader did not get the memo
	nodes[0].FencingToken = 1  // Stale token
	nodes[0].mu.Unlock()

	leaders := fm.DetectSplitBrain()
	fmt.Printf("  Current leaders: %v\n", leaders)

	// Step 7: Fencing token prevents stale leader from writing
	fmt.Println("\n--- Step 7: Fencing Token Validation ---")
	// Old leader tries to write with its stale token
	oldToken := uint64(1)
	fmt.Printf("  Old leader (node-0) tries token=%d: valid=%v\n",
		oldToken, fm.ValidateFencingToken(oldToken))

	// New leader writes with current token
	nodes[0].mu.RLock()
	// We need the new leader's token
	nodes[0].mu.RUnlock()
	newLeader.mu.RLock()
	currentToken := newLeader.FencingToken
	newLeader.mu.RUnlock()
	fmt.Printf("  New leader (%s) uses token=%d: valid=%v\n",
		newLeader.ID, currentToken, fm.ValidateFencingToken(currentToken))

	// Demonstrate automatic failover loop
	fmt.Println("\n--- Step 8: Automated Failover Loop ---")
	// Reset the cluster cleanly
	for _, n := range nodes {
		n.mu.Lock()
		n.Role = RoleFollower
		n.Healthy = true
		n.Status = StatusHealthy
		n.FencingToken = 0
		n.mu.Unlock()
	}
	fm2 := NewFailoverManager(nodes, 1*time.Second)
	fm2.SetLeader(nodes[0])

	// Simulate a sequence of crashes
	crashOrder := []int{0, 2, 4}
	for _, idx := range crashOrder {
		fmt.Printf("\n  Crashing node-%d...\n", idx)
		nodes[idx].Crash()

		if fm2.DetectFailure() {
			_, err := fm2.ElectNewLeader()
			if err != nil {
				fmt.Printf("  Cannot elect new leader: %v\n", err)
				break
			}
		}
	}

	_ = rand.Int() // suppress unused import
}
```

**Important lessons from this code:**

- **Highest offset wins**: The follower with the most up-to-date data should become the new leader to minimize data loss.
- **Fencing tokens are monotonically increasing**: Even if an old leader comes back, its stale token will be rejected.
- **Split-brain is dangerous**: Two nodes both accepting writes as "leader" can corrupt data. Fencing tokens and consensus protocols prevent this.

---

## 6. Replication Lag Problems

### The Inevitable Lag

With asynchronous replication, followers lag behind the leader. This lag is usually small (milliseconds to seconds), but under heavy load, network issues, or follower recovery, it can grow to minutes or more. This creates several consistency anomalies.

| Problem | Description | Example |
|---------|-------------|---------|
| **Read-after-write** | User writes data, then reads from a follower that has not received it yet | User updates profile, refreshes page, sees old profile |
| **Monotonic reads** | User reads from two different followers, sees time go "backward" | User sees a new comment, refreshes, comment disappears |
| **Consistent prefix** | User sees an effect before its cause | User sees a reply before the original message |

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// ReplicaStore simulates a replica with configurable delay.
type ReplicaStore struct {
	name     string
	data     map[string]string
	versions map[string]int64 // key -> version (timestamp-like)
	delay    time.Duration
	mu       sync.RWMutex
}

func NewReplicaStore(name string, delay time.Duration) *ReplicaStore {
	return &ReplicaStore{
		name:     name,
		data:     make(map[string]string),
		versions: make(map[string]int64),
		delay:    delay,
	}
}

func (r *ReplicaStore) Write(key, value string, version int64) {
	// Simulate replication delay
	go func() {
		time.Sleep(r.delay)
		r.mu.Lock()
		defer r.mu.Unlock()
		r.data[key] = value
		r.versions[key] = version
	}()
}

// WriteSync writes immediately (for the leader).
func (r *ReplicaStore) WriteSync(key, value string, version int64) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.data[key] = value
	r.versions[key] = version
}

func (r *ReplicaStore) Read(key string) (string, int64, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	val, ok := r.data[key]
	ver := r.versions[key]
	return val, ver, ok
}

func main() {
	fmt.Println("=== Replication Lag Problems ===\n")

	leader := NewReplicaStore("leader", 0)
	follower1 := NewReplicaStore("follower-1", 50*time.Millisecond)  // 50ms lag
	follower2 := NewReplicaStore("follower-2", 200*time.Millisecond) // 200ms lag

	// ---------------------------------------------------
	// Problem 1: Read-After-Write Inconsistency
	// ---------------------------------------------------
	fmt.Println("--- Problem 1: Read-After-Write Inconsistency ---")
	fmt.Println("User updates their profile name...")

	version := int64(1)
	// User writes through the leader
	leader.WriteSync("user:1:name", "Alice (updated)", version)
	follower1.Write("user:1:name", "Alice (updated)", version)
	follower2.Write("user:1:name", "Alice (updated)", version)

	// User immediately reads -- routed to a follower
	val, _, ok := follower1.Read("user:1:name")
	fmt.Printf("  Immediately read from follower-1: ok=%v value=%q\n", ok, val)
	fmt.Println("  The follower has not received the update yet!")
	fmt.Println("  User sees their old profile -- confused and frustrated.")

	time.Sleep(100 * time.Millisecond)
	val, _, ok = follower1.Read("user:1:name")
	fmt.Printf("  After 100ms from follower-1:     ok=%v value=%q\n", ok, val)

	// ---------------------------------------------------
	// Problem 2: Non-Monotonic Reads
	// ---------------------------------------------------
	fmt.Println("\n--- Problem 2: Non-Monotonic Reads ---")
	fmt.Println("User makes two reads from different followers...")

	// Seed some data in follower-1 (already replicated)
	follower1.WriteSync("post:1:comments", "3 comments", 10)
	// follower-2 is behind, has old data
	follower2.WriteSync("post:1:comments", "2 comments", 8)

	// First read: hits follower-1 (more up to date)
	val1, ver1, _ := follower1.Read("post:1:comments")
	fmt.Printf("  First read  (follower-1): %q (version %d)\n", val1, ver1)

	// Second read: hits follower-2 (stale)
	val2, ver2, _ := follower2.Read("post:1:comments")
	fmt.Printf("  Second read (follower-2): %q (version %d)\n", val2, ver2)
	fmt.Println("  Time went backward! User saw 3 comments, then only 2.")

	// ---------------------------------------------------
	// Problem 3: Consistent Prefix Reads
	// ---------------------------------------------------
	fmt.Println("\n--- Problem 3: Consistent Prefix Reads ---")
	fmt.Println("A conversation between two users:")

	// The conversation happens in order on the leader
	leader.WriteSync("chat:1", "Alice: How is the weather?", 100)
	leader.WriteSync("chat:2", "Bob: It is sunny!", 101)

	// But follower-2 is so slow it receives message 2 before message 1
	// (This can happen with partitioned replication or independent log streams)
	follower2.WriteSync("chat:2", "Bob: It is sunny!", 101)
	// chat:1 has not arrived at follower-2 yet

	_, _, hasChat1 := follower2.Read("chat:1")
	val, _, _ = follower2.Read("chat:2")
	fmt.Printf("  follower-2 has chat:1: %v\n", hasChat1)
	fmt.Printf("  follower-2 has chat:2: %q\n", val)
	fmt.Println("  User sees Bob's reply before Alice's question!")
	fmt.Println("  This violates causal ordering.")

	fmt.Println("\n--- Summary ---")
	fmt.Println("All three problems are caused by replication lag.")
	fmt.Println("The next sections show how to solve each one.")
}
```

---

## 7. Read-After-Write Consistency

### Strategies

Read-after-write consistency (also called read-your-writes) guarantees that if a user writes data and then reads it back, they see their own write. Other users may still see stale data.

Here are three implementation strategies:

1. **Read from leader for user's own data**: After a user writes, route their subsequent reads to the leader (for a short window).
2. **Track write timestamps**: The client remembers the timestamp of its last write, and only reads from replicas that are at least that up-to-date.
3. **Sticky sessions**: Route a user's requests to the same replica consistently.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Replica represents a data node with a known replication position.
type Replica struct {
	name     string
	data     map[string]string
	position int64 // The latest log position this replica has applied
	mu       sync.RWMutex
}

func NewReplica(name string) *Replica {
	return &Replica{
		name: name,
		data: make(map[string]string),
	}
}

func (r *Replica) Set(key, value string, pos int64) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.data[key] = value
	if pos > r.position {
		r.position = pos
	}
}

func (r *Replica) Get(key string) (string, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	v, ok := r.data[key]
	return v, ok
}

func (r *Replica) Position() int64 {
	r.mu.RLock()
	defer r.mu.RUnlock()
	return r.position
}

// ConsistentRouter routes reads to ensure read-after-write consistency.
type ConsistentRouter struct {
	leader    *Replica
	followers []*Replica

	// Strategy 1: Track which users wrote recently
	recentWriters map[string]time.Time // userID -> last write time
	writeWindow   time.Duration        // How long to route reads to leader

	// Strategy 2: Track write positions per user
	userPositions map[string]int64 // userID -> last write position

	mu sync.RWMutex
}

func NewConsistentRouter(leader *Replica, followers []*Replica) *ConsistentRouter {
	return &ConsistentRouter{
		leader:        leader,
		followers:     followers,
		recentWriters: make(map[string]time.Time),
		userPositions: make(map[string]int64),
		writeWindow:   500 * time.Millisecond,
	}
}

// Write always goes to the leader.
func (cr *ConsistentRouter) Write(userID, key, value string, pos int64) {
	cr.leader.Set(key, value, pos)

	cr.mu.Lock()
	cr.recentWriters[userID] = time.Now()
	cr.userPositions[userID] = pos
	cr.mu.Unlock()

	// Simulate async replication to followers (with delay)
	for _, f := range cr.followers {
		go func(follower *Replica) {
			time.Sleep(100 * time.Millisecond) // Simulated replication lag
			follower.Set(key, value, pos)
		}(f)
	}
}

// Strategy 1: Read from leader if user wrote recently.
func (cr *ConsistentRouter) ReadWithLeaderFallback(userID, key string) (string, string) {
	cr.mu.RLock()
	lastWrite, wrote := cr.recentWriters[userID]
	cr.mu.RUnlock()

	if wrote && time.Since(lastWrite) < cr.writeWindow {
		// User wrote recently -- read from leader to guarantee consistency
		val, _ := cr.leader.Get(key)
		return val, fmt.Sprintf("leader (user wrote %v ago)", time.Since(lastWrite).Round(time.Millisecond))
	}

	// Safe to read from a follower
	for _, f := range cr.followers {
		if val, ok := f.Get(key); ok {
			return val, f.name
		}
	}
	val, _ := cr.leader.Get(key)
	return val, "leader (fallback)"
}

// Strategy 2: Read from a follower that is at least at the user's write position.
func (cr *ConsistentRouter) ReadWithPositionCheck(userID, key string) (string, string) {
	cr.mu.RLock()
	requiredPos := cr.userPositions[userID]
	cr.mu.RUnlock()

	// Find a follower that is at least at the required position
	for _, f := range cr.followers {
		if f.Position() >= requiredPos {
			val, _ := f.Get(key)
			return val, fmt.Sprintf("%s (pos=%d >= required=%d)", f.name, f.Position(), requiredPos)
		}
	}

	// No follower is caught up -- read from leader
	val, _ := cr.leader.Get(key)
	return val, fmt.Sprintf("leader (no follower at pos>=%d)", requiredPos)
}

// Strategy 3: Sticky session -- always route a user to the same follower.
type StickyRouter struct {
	leader    *Replica
	followers []*Replica
	sessions  map[string]*Replica // userID -> assigned follower
	mu        sync.RWMutex
}

func NewStickyRouter(leader *Replica, followers []*Replica) *StickyRouter {
	return &StickyRouter{
		leader:    leader,
		followers: followers,
		sessions:  make(map[string]*Replica),
	}
}

func (sr *StickyRouter) AssignFollower(userID string) *Replica {
	sr.mu.Lock()
	defer sr.mu.Unlock()
	if f, ok := sr.sessions[userID]; ok {
		return f
	}
	// Simple hash-based assignment
	idx := len(userID) % len(sr.followers)
	sr.sessions[userID] = sr.followers[idx]
	return sr.followers[idx]
}

func (sr *StickyRouter) Read(userID, key string) (string, string) {
	follower := sr.AssignFollower(userID)
	val, _ := follower.Get(key)
	return val, follower.name + " (sticky)"
}

func main() {
	fmt.Println("=== Read-After-Write Consistency Strategies ===\n")

	leader := NewReplica("leader")
	f1 := NewReplica("follower-1")
	f2 := NewReplica("follower-2")
	followers := []*Replica{f1, f2}

	router := NewConsistentRouter(leader, followers)

	// User writes their profile
	fmt.Println("--- User 'alice' updates their profile ---")
	router.Write("alice", "profile:alice", "Alice - Software Engineer (UPDATED)", 42)

	// Strategy 1: Leader fallback for recent writers
	fmt.Println("\n--- Strategy 1: Leader Fallback ---")
	val, source := router.ReadWithLeaderFallback("alice", "profile:alice")
	fmt.Printf("  alice reads: %q from %s\n", val, source)

	val, source = router.ReadWithLeaderFallback("bob", "profile:alice")
	fmt.Printf("  bob reads:   %q from %s (bob did not write, reads from follower)\n", val, source)

	// Wait for write window to expire
	time.Sleep(600 * time.Millisecond)
	val, source = router.ReadWithLeaderFallback("alice", "profile:alice")
	fmt.Printf("  alice (after window): %q from %s\n", val, source)

	// Strategy 2: Position-based routing
	fmt.Println("\n--- Strategy 2: Position Check ---")
	// Reset: write again
	router.Write("alice", "profile:alice", "Alice - Senior Engineer", 43)

	// Immediately read -- followers at position 42, need 43
	val, source = router.ReadWithPositionCheck("alice", "profile:alice")
	fmt.Printf("  alice reads: %q from %s\n", val, source)

	// Wait for replication
	time.Sleep(150 * time.Millisecond)
	val, source = router.ReadWithPositionCheck("alice", "profile:alice")
	fmt.Printf("  alice (after replication): %q from %s\n", val, source)

	// Strategy 3: Sticky sessions
	fmt.Println("\n--- Strategy 3: Sticky Sessions ---")
	sticky := NewStickyRouter(leader, followers)

	// Pre-populate followers with some data
	f1.Set("profile:alice", "Alice - from f1", 40)
	f2.Set("profile:alice", "Alice - from f2", 40)

	val, source = sticky.Read("alice", "profile:alice")
	fmt.Printf("  alice reads: %q from %s\n", val, source)
	val, source = sticky.Read("alice", "profile:alice")
	fmt.Printf("  alice again: %q from %s (same follower!)\n", val, source)
	val, source = sticky.Read("bob", "profile:alice")
	fmt.Printf("  bob reads:   %q from %s (may be different follower)\n", val, source)
}
```

In production, the most common approach is a combination: use **position-based routing** for correctness and **sticky sessions** as an optimization to avoid checking positions on every request.

---

## 8. Multi-Leader Replication

### When Single-Leader Is Not Enough

Single-leader replication has a limitation: all writes must go through one node. This creates problems when:

- **Multi-datacenter deployment**: If the leader is in Virginia and a user in Tokyo writes, the write must cross the Pacific (high latency). With multi-leader, each datacenter has its own leader.
- **Clients with offline operation**: Apps like Google Docs, Notion, or calendar apps accept writes when offline. Each device acts as a "leader" for the user's data.
- **Collaborative editing**: Multiple users editing the same document simultaneously.

In multi-leader replication, each leader independently accepts writes, and leaders asynchronously replicate to each other. The fundamental challenge is **conflict resolution** -- what happens when two leaders modify the same data concurrently?

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// WriteRecord represents a write that happened at a specific leader.
type WriteRecord struct {
	Key       string
	Value     string
	Timestamp time.Time
	LeaderID  string
	Version   int64
}

// LeaderNode is a node that accepts writes independently.
type LeaderNode struct {
	ID         string
	Datacenter string
	data       map[string]WriteRecord
	writeLog   []WriteRecord
	version    int64
	peers      []*LeaderNode
	mu         sync.RWMutex
}

func NewLeaderNode(id, datacenter string) *LeaderNode {
	return &LeaderNode{
		ID:         id,
		Datacenter: datacenter,
		data:       make(map[string]WriteRecord),
	}
}

func (ln *LeaderNode) AddPeer(peer *LeaderNode) {
	ln.mu.Lock()
	defer ln.mu.Unlock()
	ln.peers = append(ln.peers, peer)
}

// Write accepts a write locally, then asynchronously replicates to peers.
func (ln *LeaderNode) Write(key, value string) WriteRecord {
	ln.mu.Lock()
	ln.version++
	record := WriteRecord{
		Key:       key,
		Value:     value,
		Timestamp: time.Now(),
		LeaderID:  ln.ID,
		Version:   ln.version,
	}
	ln.data[key] = record
	ln.writeLog = append(ln.writeLog, record)
	peers := make([]*LeaderNode, len(ln.peers))
	copy(peers, ln.peers)
	ln.mu.Unlock()

	// Async replication to peer leaders
	for _, peer := range peers {
		go func(p *LeaderNode) {
			p.ReceiveReplication(record)
		}(peer)
	}

	return record
}

// ReceiveReplication applies a write from a peer leader.
// This is where conflicts can occur.
func (ln *LeaderNode) ReceiveReplication(record WriteRecord) ConflictResult {
	ln.mu.Lock()
	defer ln.mu.Unlock()

	existing, exists := ln.data[record.Key]
	if exists && existing.LeaderID != record.LeaderID {
		// CONFLICT: Two leaders wrote to the same key.
		// For now, use Last-Write-Wins (LWW) based on timestamp.
		if record.Timestamp.After(existing.Timestamp) {
			ln.data[record.Key] = record
			return ConflictResult{
				Detected: true,
				Winner:   record.LeaderID,
				Loser:    existing.LeaderID,
				Key:      record.Key,
			}
		}
		return ConflictResult{
			Detected: true,
			Winner:   existing.LeaderID,
			Loser:    record.LeaderID,
			Key:      record.Key,
		}
	}

	ln.data[record.Key] = record
	return ConflictResult{Detected: false}
}

func (ln *LeaderNode) Read(key string) (WriteRecord, bool) {
	ln.mu.RLock()
	defer ln.mu.RUnlock()
	val, ok := ln.data[key]
	return val, ok
}

// ConflictResult describes the outcome of conflict detection.
type ConflictResult struct {
	Detected bool
	Winner   string
	Loser    string
	Key      string
}

// MultiLeaderCluster manages multiple leader nodes.
type MultiLeaderCluster struct {
	leaders []*LeaderNode
}

func NewMultiLeaderCluster() *MultiLeaderCluster {
	return &MultiLeaderCluster{}
}

func (mlc *MultiLeaderCluster) AddLeader(leader *LeaderNode) {
	// Connect to all existing leaders
	for _, existing := range mlc.leaders {
		existing.AddPeer(leader)
		leader.AddPeer(existing)
	}
	mlc.leaders = append(mlc.leaders, leader)
}

func main() {
	fmt.Println("=== Multi-Leader Replication ===\n")

	// --- Use Case 1: Multi-Datacenter Deployment ---
	fmt.Println("--- Use Case 1: Multi-Datacenter ---")

	cluster := NewMultiLeaderCluster()
	usEast := NewLeaderNode("us-east-leader", "us-east")
	euWest := NewLeaderNode("eu-west-leader", "eu-west")
	apNorth := NewLeaderNode("ap-north-leader", "ap-northeast")

	cluster.AddLeader(usEast)
	cluster.AddLeader(euWest)
	cluster.AddLeader(apNorth)

	// Users in different regions write to their local leader
	fmt.Println("User in Virginia writes to us-east leader:")
	rec := usEast.Write("user:alice:status", "Online")
	fmt.Printf("  Written at %s: key=%s value=%q\n", rec.LeaderID, rec.Key, rec.Value)

	fmt.Println("User in Frankfurt writes to eu-west leader:")
	rec = euWest.Write("user:bob:status", "Away")
	fmt.Printf("  Written at %s: key=%s value=%q\n", rec.LeaderID, rec.Key, rec.Value)

	fmt.Println("User in Tokyo writes to ap-north leader:")
	rec = apNorth.Write("user:charlie:status", "Do Not Disturb")
	fmt.Printf("  Written at %s: key=%s value=%q\n", rec.LeaderID, rec.Key, rec.Value)

	// Wait for async replication
	time.Sleep(50 * time.Millisecond)

	// All leaders should have all data (eventually consistent)
	fmt.Println("\nAfter replication, all leaders have all data:")
	for _, leader := range cluster.leaders {
		for _, key := range []string{"user:alice:status", "user:bob:status", "user:charlie:status"} {
			val, ok := leader.Read(key)
			if ok {
				fmt.Printf("  %s: %s = %q\n", leader.ID, key, val.Value)
			}
		}
	}

	// --- Use Case 2: Conflict Detection ---
	fmt.Println("\n--- Use Case 2: Concurrent Writes (Conflict) ---")

	cluster2 := NewMultiLeaderCluster()
	leader1 := NewLeaderNode("dc1", "datacenter-1")
	leader2 := NewLeaderNode("dc2", "datacenter-2")
	cluster2.AddLeader(leader1)
	cluster2.AddLeader(leader2)

	fmt.Println("Both leaders write to the same key simultaneously:")

	// Simulate concurrent writes by writing to both leaders before replication
	leader1.mu.Lock()
	leader1.version++
	r1 := WriteRecord{
		Key: "doc:1:title", Value: "Version A (from dc1)",
		Timestamp: time.Now(), LeaderID: "dc1", Version: leader1.version,
	}
	leader1.data[r1.Key] = r1
	leader1.writeLog = append(leader1.writeLog, r1)
	leader1.mu.Unlock()
	fmt.Printf("  dc1 wrote: %q\n", r1.Value)

	time.Sleep(1 * time.Millisecond) // Tiny time difference for LWW

	leader2.mu.Lock()
	leader2.version++
	r2 := WriteRecord{
		Key: "doc:1:title", Value: "Version B (from dc2)",
		Timestamp: time.Now(), LeaderID: "dc2", Version: leader2.version,
	}
	leader2.data[r2.Key] = r2
	leader2.writeLog = append(leader2.writeLog, r2)
	leader2.mu.Unlock()
	fmt.Printf("  dc2 wrote: %q\n", r2.Value)

	// Now replicate to each other
	conflict1 := leader2.ReceiveReplication(r1)
	conflict2 := leader1.ReceiveReplication(r2)

	fmt.Printf("\n  Conflict at dc2: detected=%v winner=%s\n", conflict1.Detected, conflict1.Winner)
	fmt.Printf("  Conflict at dc1: detected=%v winner=%s\n", conflict2.Detected, conflict2.Winner)

	// Check final state
	val1, _ := leader1.Read("doc:1:title")
	val2, _ := leader2.Read("doc:1:title")
	fmt.Printf("\n  dc1 final value: %q\n", val1.Value)
	fmt.Printf("  dc2 final value: %q\n", val2.Value)

	if val1.Value == val2.Value {
		fmt.Println("  Leaders converged to the same value.")
	} else {
		fmt.Println("  WARNING: Leaders have DIFFERENT values (inconsistency).")
	}
}
```

**Key insight:** Multi-leader replication provides lower write latency in geographically distributed deployments, but conflict handling is complex. The next section explores different strategies for resolving conflicts.

---

## 9. Conflict Resolution Strategies

When two leaders accept concurrent writes to the same key, you need a strategy to decide the final value. Here are the most common approaches:

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Last-Write-Wins (LWW)** | Compare timestamps; latest wins | Simple, always converges | Data loss -- earlier writes are silently discarded. Clock skew can cause surprises. |
| **Merge function** | Application-specific logic combines both values | No data loss | Complex; application must define merge semantics |
| **CRDT** (Conflict-free Replicated Data Type) | Mathematical structures that can always be merged | Provably convergent, no coordination needed | Limited to specific data structures |
| **Custom resolution** | Present conflicts to the user or application | Maximum flexibility | Complexity pushed to application layer |

```go
package main

import (
	"fmt"
	"sort"
	"strings"
	"sync"
	"time"
)

// ========================================
// Strategy 1: Last-Write-Wins (LWW) Register
// ========================================

// LWWRegister is a register where the latest timestamp wins.
type LWWRegister struct {
	Value     string
	Timestamp time.Time
	NodeID    string
	mu        sync.RWMutex
}

func NewLWWRegister() *LWWRegister {
	return &LWWRegister{}
}

func (r *LWWRegister) Set(value, nodeID string, ts time.Time) bool {
	r.mu.Lock()
	defer r.mu.Unlock()
	if ts.After(r.Timestamp) || (ts.Equal(r.Timestamp) && nodeID > r.NodeID) {
		r.Value = value
		r.Timestamp = ts
		r.NodeID = nodeID
		return true // This write won
	}
	return false // This write was older, discarded
}

func (r *LWWRegister) Get() (string, time.Time, string) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	return r.Value, r.Timestamp, r.NodeID
}

// Merge combines two LWW registers. The newer timestamp wins.
func (r *LWWRegister) Merge(other *LWWRegister) {
	other.mu.RLock()
	otherVal, otherTS, otherNode := other.Value, other.Timestamp, other.NodeID
	other.mu.RUnlock()

	r.Set(otherVal, otherNode, otherTS)
}

// ========================================
// Strategy 2: Multi-Value Register (Siblings)
// ========================================

// Sibling represents one version of a conflicting value.
type Sibling struct {
	Value  string
	NodeID string
	Time   time.Time
}

// MVRegister keeps all conflicting values until the application resolves them.
type MVRegister struct {
	siblings []Sibling
	mu       sync.RWMutex
}

func NewMVRegister() *MVRegister {
	return &MVRegister{}
}

func (mv *MVRegister) Write(value, nodeID string) {
	mv.mu.Lock()
	defer mv.mu.Unlock()
	mv.siblings = append(mv.siblings, Sibling{
		Value: value, NodeID: nodeID, Time: time.Now(),
	})
}

func (mv *MVRegister) Read() []Sibling {
	mv.mu.RLock()
	defer mv.mu.RUnlock()
	result := make([]Sibling, len(mv.siblings))
	copy(result, mv.siblings)
	return result
}

// Resolve applies a merge function to collapse siblings into a single value.
func (mv *MVRegister) Resolve(mergeFn func(siblings []Sibling) string) string {
	mv.mu.Lock()
	defer mv.mu.Unlock()
	if len(mv.siblings) == 0 {
		return ""
	}
	merged := mergeFn(mv.siblings)
	mv.siblings = []Sibling{{
		Value: merged, NodeID: "merged", Time: time.Now(),
	}}
	return merged
}

// ========================================
// Strategy 3: G-Counter CRDT
// ========================================

// GCounter is a grow-only counter CRDT.
// Each node maintains its own count. The total is the sum of all counts.
// Merging takes the max of each node's count.
type GCounter struct {
	counts map[string]int64
	mu     sync.RWMutex
}

func NewGCounter() *GCounter {
	return &GCounter{counts: make(map[string]int64)}
}

func (gc *GCounter) Increment(nodeID string) {
	gc.mu.Lock()
	defer gc.mu.Unlock()
	gc.counts[nodeID]++
}

func (gc *GCounter) IncrementBy(nodeID string, amount int64) {
	gc.mu.Lock()
	defer gc.mu.Unlock()
	gc.counts[nodeID] += amount
}

func (gc *GCounter) Value() int64 {
	gc.mu.RLock()
	defer gc.mu.RUnlock()
	var total int64
	for _, c := range gc.counts {
		total += c
	}
	return total
}

// Merge combines this counter with another. Takes the max of each node's count.
// This is the key CRDT property: merge is commutative, associative, and idempotent.
func (gc *GCounter) Merge(other *GCounter) {
	gc.mu.Lock()
	defer gc.mu.Unlock()
	other.mu.RLock()
	defer other.mu.RUnlock()

	for nodeID, count := range other.counts {
		if count > gc.counts[nodeID] {
			gc.counts[nodeID] = count
		}
	}
}

func (gc *GCounter) String() string {
	gc.mu.RLock()
	defer gc.mu.RUnlock()
	parts := make([]string, 0, len(gc.counts))
	for k, v := range gc.counts {
		parts = append(parts, fmt.Sprintf("%s:%d", k, v))
	}
	sort.Strings(parts)
	return fmt.Sprintf("{%s} = %d", strings.Join(parts, ", "), gc.Value())
}

// ========================================
// Strategy 4: PN-Counter CRDT (supports decrement)
// ========================================

// PNCounter supports both increment and decrement using two G-Counters.
type PNCounter struct {
	positive *GCounter
	negative *GCounter
}

func NewPNCounter() *PNCounter {
	return &PNCounter{
		positive: NewGCounter(),
		negative: NewGCounter(),
	}
}

func (pn *PNCounter) Increment(nodeID string) {
	pn.positive.Increment(nodeID)
}

func (pn *PNCounter) Decrement(nodeID string) {
	pn.negative.Increment(nodeID)
}

func (pn *PNCounter) Value() int64 {
	return pn.positive.Value() - pn.negative.Value()
}

func (pn *PNCounter) Merge(other *PNCounter) {
	pn.positive.Merge(other.positive)
	pn.negative.Merge(other.negative)
}

// ========================================
// Strategy 5: OR-Set CRDT (Observed-Remove Set)
// ========================================

// UniqueTag pairs a value with a unique identifier so that adds and removes
// can be tracked independently.
type UniqueTag struct {
	Value string
	Tag   string // Globally unique tag (e.g., nodeID + counter)
}

// ORSet is an add/remove set CRDT. Adding an element creates a unique tag.
// Removing an element removes all known tags for that value.
type ORSet struct {
	elements map[string]map[string]bool // value -> set of tags
	tagCount int64
	nodeID   string
	mu       sync.RWMutex
}

func NewORSet(nodeID string) *ORSet {
	return &ORSet{
		elements: make(map[string]map[string]bool),
		nodeID:   nodeID,
	}
}

func (s *ORSet) Add(value string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.tagCount++
	tag := fmt.Sprintf("%s:%d", s.nodeID, s.tagCount)
	if _, ok := s.elements[value]; !ok {
		s.elements[value] = make(map[string]bool)
	}
	s.elements[value][tag] = true
}

func (s *ORSet) Remove(value string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	delete(s.elements, value)
}

func (s *ORSet) Contains(value string) bool {
	s.mu.RLock()
	defer s.mu.RUnlock()
	tags, ok := s.elements[value]
	return ok && len(tags) > 0
}

func (s *ORSet) Values() []string {
	s.mu.RLock()
	defer s.mu.RUnlock()
	var result []string
	for val, tags := range s.elements {
		if len(tags) > 0 {
			result = append(result, val)
		}
	}
	sort.Strings(result)
	return result
}

// Merge combines two OR-Sets. The union of all unique tags.
func (s *ORSet) Merge(other *ORSet) {
	s.mu.Lock()
	defer s.mu.Unlock()
	other.mu.RLock()
	defer other.mu.RUnlock()

	for value, otherTags := range other.elements {
		if _, ok := s.elements[value]; !ok {
			s.elements[value] = make(map[string]bool)
		}
		for tag := range otherTags {
			s.elements[value][tag] = true
		}
	}
}

func main() {
	fmt.Println("=== Conflict Resolution Strategies ===\n")

	// --- Strategy 1: Last-Write-Wins ---
	fmt.Println("--- Strategy 1: Last-Write-Wins (LWW) ---")
	reg := NewLWWRegister()

	t1 := time.Now()
	t2 := t1.Add(1 * time.Millisecond)

	won1 := reg.Set("Alice's version", "node-1", t1)
	fmt.Printf("  node-1 writes at t1: won=%v\n", won1)

	won2 := reg.Set("Bob's version", "node-2", t2)
	fmt.Printf("  node-2 writes at t2: won=%v\n", won2)

	val, ts, node := reg.Get()
	fmt.Printf("  Final: value=%q winner=%s time=%v\n", val, node, ts.Format(time.RFC3339Nano))
	fmt.Println("  Problem: Alice's write is silently lost.\n")

	// --- Strategy 2: Multi-Value (keep all siblings) ---
	fmt.Println("--- Strategy 2: Multi-Value Register (Siblings) ---")
	mv := NewMVRegister()
	mv.Write("Alice's edit", "node-1")
	mv.Write("Bob's edit", "node-2")

	siblings := mv.Read()
	fmt.Printf("  Siblings found: %d\n", len(siblings))
	for _, s := range siblings {
		fmt.Printf("    value=%q from=%s\n", s.Value, s.NodeID)
	}

	// Resolve by concatenating (application-specific merge)
	merged := mv.Resolve(func(siblings []Sibling) string {
		var parts []string
		for _, s := range siblings {
			parts = append(parts, s.Value)
		}
		return strings.Join(parts, " + ")
	})
	fmt.Printf("  Merged result: %q\n\n", merged)

	// --- Strategy 3: G-Counter CRDT ---
	fmt.Println("--- Strategy 3: G-Counter CRDT ---")
	// Three nodes independently counting page views
	counterA := NewGCounter()
	counterB := NewGCounter()
	counterC := NewGCounter()

	// Each node increments independently (no coordination needed)
	counterA.IncrementBy("node-A", 10)
	counterB.IncrementBy("node-B", 7)
	counterC.IncrementBy("node-C", 15)
	counterA.IncrementBy("node-A", 5) // node-A gets more views

	fmt.Printf("  Node A view: %d (local increments only)\n", counterA.Value())
	fmt.Printf("  Node B view: %d\n", counterB.Value())
	fmt.Printf("  Node C view: %d\n", counterC.Value())

	// Merge all counters -- order does not matter! (commutative + associative)
	counterA.Merge(counterB)
	counterA.Merge(counterC)
	fmt.Printf("  After merge at A: %d (10+5+7+15 = 37)\n", counterA.Value())

	// Merge again (idempotent -- no change)
	counterA.Merge(counterB)
	fmt.Printf("  After duplicate merge: %d (still 37, merge is idempotent)\n\n", counterA.Value())

	// --- Strategy 4: PN-Counter ---
	fmt.Println("--- Strategy 4: PN-Counter CRDT (Increment + Decrement) ---")
	likesA := NewPNCounter()
	likesB := NewPNCounter()

	likesA.Increment("node-A") // +1
	likesA.Increment("node-A") // +1
	likesB.Increment("node-B") // +1
	likesB.Decrement("node-B") // -1 (user unliked)

	fmt.Printf("  Node A sees: %d likes\n", likesA.Value())
	fmt.Printf("  Node B sees: %d likes\n", likesB.Value())

	likesA.Merge(likesB)
	fmt.Printf("  After merge: %d likes (2+1-1 = 2)\n\n", likesA.Value())

	// --- Strategy 5: OR-Set CRDT ---
	fmt.Println("--- Strategy 5: OR-Set CRDT ---")
	setA := NewORSet("node-A")
	setB := NewORSet("node-B")

	// Concurrent operations
	setA.Add("milk")
	setA.Add("eggs")
	setA.Add("bread")

	setB.Add("milk")
	setB.Add("butter")
	setB.Remove("milk") // node-B removes milk

	fmt.Printf("  Node A: %v\n", setA.Values())
	fmt.Printf("  Node B: %v\n", setB.Values())

	// After merge: milk survives because node-A's add has a tag that node-B never saw
	setA.Merge(setB)
	fmt.Printf("  After merge at A: %v\n", setA.Values())
	fmt.Println("  Note: 'milk' survives because node-A added it with a unique tag")
	fmt.Println("  that node-B's remove did not know about. Add wins over concurrent remove.")
}
```

**CRDT Key Properties:**

- **Commutative**: `merge(A, B) = merge(B, A)` -- order does not matter.
- **Associative**: `merge(merge(A, B), C) = merge(A, merge(B, C))` -- grouping does not matter.
- **Idempotent**: `merge(A, A) = A` -- merging the same data twice has no effect.

These three properties guarantee that replicas always converge to the same state, regardless of the order or number of times they synchronize.

---

## 10. Leaderless Replication

### Dynamo-Style Architecture

In leaderless replication, there is no leader. Any node can accept reads and writes. The client sends writes to multiple replicas and reads from multiple replicas. Correctness depends on **quorum** rules.

This approach was popularized by Amazon's Dynamo paper and is used by Cassandra, Riak, and Voldemort.

The key parameters are:
- **N** = total number of replicas
- **W** = number of replicas that must acknowledge a write (write quorum)
- **R** = number of replicas that must respond to a read (read quorum)

As long as **W + R > N**, the read set and write set must overlap, guaranteeing that at least one read replica has the latest value.

```go
package main

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

// DynamoValue represents a versioned value stored in a node.
type DynamoValue struct {
	Value     string
	Version   int64
	Timestamp time.Time
	Writer    string // Which client wrote this
}

// DynamoNode is a single node in a leaderless cluster.
type DynamoNode struct {
	ID      string
	data    map[string]DynamoValue
	healthy bool
	delay   time.Duration // Simulated response delay
	mu      sync.RWMutex
}

func NewDynamoNode(id string, delay time.Duration) *DynamoNode {
	return &DynamoNode{
		ID:      id,
		data:    make(map[string]DynamoValue),
		healthy: true,
		delay:   delay,
	}
}

func (dn *DynamoNode) Write(key string, val DynamoValue) error {
	time.Sleep(dn.delay)
	dn.mu.Lock()
	defer dn.mu.Unlock()
	if !dn.healthy {
		return fmt.Errorf("node %s is down", dn.ID)
	}
	existing, exists := dn.data[key]
	if !exists || val.Version > existing.Version {
		dn.data[key] = val
	}
	return nil
}

func (dn *DynamoNode) Read(key string) (DynamoValue, error) {
	time.Sleep(dn.delay)
	dn.mu.RLock()
	defer dn.mu.RUnlock()
	if !dn.healthy {
		return DynamoValue{}, fmt.Errorf("node %s is down", dn.ID)
	}
	val, ok := dn.data[key]
	if !ok {
		return DynamoValue{}, fmt.Errorf("key %q not found on %s", key, dn.ID)
	}
	return val, nil
}

func (dn *DynamoNode) Crash() {
	dn.mu.Lock()
	defer dn.mu.Unlock()
	dn.healthy = false
}

func (dn *DynamoNode) Recover() {
	dn.mu.Lock()
	defer dn.mu.Unlock()
	dn.healthy = true
}

// DynamoCluster implements leaderless replication with quorum reads/writes.
type DynamoCluster struct {
	nodes   []*DynamoNode
	n       int // Total replicas
	w       int // Write quorum
	r       int // Read quorum
	version int64
	mu      sync.Mutex
}

func NewDynamoCluster(nodes []*DynamoNode, w, r int) *DynamoCluster {
	return &DynamoCluster{
		nodes: nodes,
		n:     len(nodes),
		w:     w,
		r:     r,
	}
}

// Write sends the value to all nodes and waits for W acknowledgements.
func (dc *DynamoCluster) Write(key, value, writer string) (int, error) {
	dc.mu.Lock()
	dc.version++
	ver := dc.version
	dc.mu.Unlock()

	val := DynamoValue{
		Value:     value,
		Version:   ver,
		Timestamp: time.Now(),
		Writer:    writer,
	}

	type writeResult struct {
		nodeID string
		err    error
	}

	results := make(chan writeResult, dc.n)

	for _, node := range dc.nodes {
		go func(n *DynamoNode) {
			err := n.Write(key, val)
			results <- writeResult{nodeID: n.ID, err: err}
		}(node)
	}

	acks := 0
	failures := 0
	var failedNodes []string

	for i := 0; i < dc.n; i++ {
		res := <-results
		if res.err == nil {
			acks++
		} else {
			failures++
			failedNodes = append(failedNodes, res.nodeID)
		}
	}

	if acks < dc.w {
		return acks, fmt.Errorf("write quorum not met: got %d acks, need %d (failed: %v)",
			acks, dc.w, failedNodes)
	}

	return acks, nil
}

// Read queries all nodes and returns the latest value from R responses.
func (dc *DynamoCluster) Read(key string) (DynamoValue, int, error) {
	type readResult struct {
		nodeID string
		val    DynamoValue
		err    error
	}

	results := make(chan readResult, dc.n)

	for _, node := range dc.nodes {
		go func(n *DynamoNode) {
			val, err := n.Read(key)
			results <- readResult{nodeID: n.ID, val: val, err: err}
		}(node)
	}

	var successes []readResult
	var failures []string

	for i := 0; i < dc.n; i++ {
		res := <-results
		if res.err == nil {
			successes = append(successes, res)
		} else {
			failures = append(failures, res.nodeID)
		}
	}

	if len(successes) < dc.r {
		return DynamoValue{}, len(successes),
			fmt.Errorf("read quorum not met: got %d responses, need %d (failed: %v)",
				len(successes), dc.r, failures)
	}

	// Find the latest version among responses
	sort.Slice(successes, func(i, j int) bool {
		return successes[i].val.Version > successes[j].val.Version
	})

	latest := successes[0].val

	// Read repair: if any responding node had a stale version, update it
	for _, res := range successes[1:] {
		if res.val.Version < latest.Version {
			// This node is stale -- repair it in the background
			for _, node := range dc.nodes {
				if node.ID == res.nodeID {
					go func(n *DynamoNode) {
						_ = n.Write(key, latest)
					}(node)
					break
				}
			}
		}
	}

	return latest, len(successes), nil
}

func main() {
	fmt.Println("=== Leaderless (Dynamo-Style) Replication ===\n")

	// Create 5 nodes with varying response times
	nodes := []*DynamoNode{
		NewDynamoNode("node-0", 5*time.Millisecond),
		NewDynamoNode("node-1", 10*time.Millisecond),
		NewDynamoNode("node-2", 15*time.Millisecond),
		NewDynamoNode("node-3", 20*time.Millisecond),
		NewDynamoNode("node-4", 25*time.Millisecond),
	}

	// N=5, W=3, R=3 (W+R=6 > N=5, guarantees overlap)
	cluster := NewDynamoCluster(nodes, 3, 3)
	fmt.Printf("Cluster: N=%d, W=%d, R=%d (W+R=%d > N)\n\n", 5, 3, 3, 6)

	// --- Normal operation ---
	fmt.Println("--- Normal Write and Read ---")
	acks, err := cluster.Write("user:1", "Alice", "client-A")
	fmt.Printf("Write user:1='Alice': acks=%d err=%v\n", acks, err)

	val, responses, err := cluster.Read("user:1")
	fmt.Printf("Read user:1: value=%q version=%d responses=%d err=%v\n\n",
		val.Value, val.Version, responses, err)

	// --- One node fails ---
	fmt.Println("--- One Node Fails ---")
	nodes[4].Crash()
	fmt.Println("node-4 crashed")

	acks, err = cluster.Write("user:2", "Bob", "client-B")
	fmt.Printf("Write user:2='Bob': acks=%d err=%v\n", acks, err)

	val, responses, err = cluster.Read("user:2")
	fmt.Printf("Read user:2: value=%q version=%d responses=%d err=%v\n\n",
		val.Value, val.Version, responses, err)

	// --- Two nodes fail (still meets quorum with 3 out of 5) ---
	fmt.Println("--- Two Nodes Fail ---")
	nodes[3].Crash()
	fmt.Println("node-3 also crashed")

	acks, err = cluster.Write("user:3", "Charlie", "client-C")
	fmt.Printf("Write user:3='Charlie': acks=%d err=%v\n", acks, err)

	val, responses, err = cluster.Read("user:3")
	fmt.Printf("Read user:3: value=%q responses=%d err=%v\n\n",
		val.Value, responses, err)

	// --- Three nodes fail (quorum lost!) ---
	fmt.Println("--- Three Nodes Fail (Quorum Lost) ---")
	nodes[2].Crash()
	fmt.Println("node-2 also crashed")

	acks, err = cluster.Write("user:4", "Dave", "client-D")
	fmt.Printf("Write user:4: acks=%d err=%v\n", acks, err)

	// --- Demonstrate read repair ---
	fmt.Println("\n--- Read Repair ---")
	nodes[2].Recover()
	nodes[3].Recover()
	nodes[4].Recover()
	fmt.Println("All nodes recovered")

	// node-4 missed the write for user:2 (it was down)
	// A quorum read will detect the stale data and repair it
	val, responses, _ = cluster.Read("user:2")
	fmt.Printf("Read user:2: value=%q (triggers read repair on stale nodes)\n", val.Value)

	time.Sleep(50 * time.Millisecond) // Wait for read repair to propagate
	// Now all nodes should have user:2
	for _, n := range nodes {
		v, err := n.Read("user:2")
		status := "OK"
		if err != nil {
			status = err.Error()
		} else {
			status = fmt.Sprintf("value=%q version=%d", v.Value, v.Version)
		}
		fmt.Printf("  %s: %s\n", n.ID, status)
	}
}
```

---

## 11. Quorum Reads and Writes

### The Math Behind Quorums

The quorum condition **W + R > N** ensures that any read will overlap with at least one node that participated in the latest write. This provides a probabilistic guarantee of reading fresh data.

Different W/R configurations offer different trade-offs:

| Configuration | Write Performance | Read Performance | Fault Tolerance |
|--------------|-------------------|------------------|-----------------|
| W=N, R=1 | Slow (all must ack) | Fast (one node enough) | Any failure blocks writes |
| W=1, R=N | Fast (one node enough) | Slow (all must respond) | Any failure blocks reads |
| W=2, R=2 (N=3) | Moderate | Moderate | Tolerates 1 failure for both |
| W=3, R=3 (N=5) | Moderate | Moderate | Tolerates 2 failures for both |

**Sloppy quorum**: When not enough nodes in the designated set are reachable, writes can be accepted by nodes outside the designated set (hinted handoff). This improves availability at the cost of potential stale reads even when W + R > N.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// QuorumConfig holds the N, W, R parameters.
type QuorumConfig struct {
	N int
	W int
	R int
}

func (qc QuorumConfig) Valid() bool {
	return qc.W+qc.R > qc.N && qc.W <= qc.N && qc.R <= qc.N
}

func (qc QuorumConfig) String() string {
	overlap := qc.W + qc.R - qc.N
	return fmt.Sprintf("N=%d W=%d R=%d (overlap=%d, tolerates %d failures for writes, %d for reads)",
		qc.N, qc.W, qc.R, overlap, qc.N-qc.W, qc.N-qc.R)
}

// QNode is a simple key-value node.
type QNode struct {
	id      string
	data    map[string]int64 // key -> version
	values  map[string]string
	healthy bool
	mu      sync.RWMutex
}

func NewQNode(id string) *QNode {
	return &QNode{
		id:      id,
		data:    make(map[string]int64),
		values:  make(map[string]string),
		healthy: true,
	}
}

func (n *QNode) Put(key, value string, version int64) bool {
	n.mu.Lock()
	defer n.mu.Unlock()
	if !n.healthy {
		return false
	}
	if version >= n.data[key] {
		n.data[key] = version
		n.values[key] = value
	}
	return true
}

func (n *QNode) Get(key string) (string, int64, bool) {
	n.mu.RLock()
	defer n.mu.RUnlock()
	if !n.healthy {
		return "", 0, false
	}
	v, ok := n.values[key]
	ver := n.data[key]
	return v, ver, ok
}

// HintedHandoff stores writes destined for unavailable nodes.
type HintedHandoff struct {
	hints map[string][]HintEntry // nodeID -> pending entries
	mu    sync.Mutex
}

type HintEntry struct {
	Key     string
	Value   string
	Version int64
}

func NewHintedHandoff() *HintedHandoff {
	return &HintedHandoff{hints: make(map[string][]HintEntry)}
}

func (hh *HintedHandoff) Store(nodeID, key, value string, version int64) {
	hh.mu.Lock()
	defer hh.mu.Unlock()
	hh.hints[nodeID] = append(hh.hints[nodeID], HintEntry{
		Key: key, Value: value, Version: version,
	})
}

func (hh *HintedHandoff) Deliver(nodeID string, node *QNode) int {
	hh.mu.Lock()
	defer hh.mu.Unlock()
	hints, ok := hh.hints[nodeID]
	if !ok {
		return 0
	}
	delivered := 0
	for _, h := range hints {
		if node.Put(h.Key, h.Value, h.Version) {
			delivered++
		}
	}
	delete(hh.hints, nodeID)
	return delivered
}

// SloppyQuorumCluster supports hinted handoff for improved availability.
type SloppyQuorumCluster struct {
	nodes    []*QNode
	config   QuorumConfig
	hints    *HintedHandoff
	version  int64
	mu       sync.Mutex
}

func NewSloppyQuorumCluster(config QuorumConfig) *SloppyQuorumCluster {
	nodes := make([]*QNode, config.N)
	for i := 0; i < config.N; i++ {
		nodes[i] = NewQNode(fmt.Sprintf("node-%d", i))
	}
	return &SloppyQuorumCluster{
		nodes:  nodes,
		config: config,
		hints:  NewHintedHandoff(),
	}
}

func (sc *SloppyQuorumCluster) Write(key, value string) (int, int, error) {
	sc.mu.Lock()
	sc.version++
	ver := sc.version
	sc.mu.Unlock()

	acks := 0
	hinted := 0
	for _, n := range sc.nodes {
		if n.Put(key, value, ver) {
			acks++
		} else {
			// Node is down -- store hint for later delivery
			sc.hints.Store(n.id, key, value, ver)
			hinted++
		}
	}

	if acks < sc.config.W {
		return acks, hinted, fmt.Errorf("write quorum not met: %d/%d (hinted: %d)",
			acks, sc.config.W, hinted)
	}
	return acks, hinted, nil
}

func (sc *SloppyQuorumCluster) Read(key string) (string, int64, int, error) {
	type result struct {
		value   string
		version int64
		ok      bool
	}

	var results []result
	for _, n := range sc.nodes {
		v, ver, ok := n.Get(key)
		if ok {
			results = append(results, result{v, ver, ok})
		}
	}

	if len(results) < sc.config.R {
		return "", 0, len(results), fmt.Errorf("read quorum not met: %d/%d",
			len(results), sc.config.R)
	}

	// Find latest version
	best := results[0]
	for _, r := range results[1:] {
		if r.version > best.version {
			best = r
		}
	}

	return best.value, best.version, len(results), nil
}

// DeliverHints sends stored hints to recovered nodes.
func (sc *SloppyQuorumCluster) DeliverHints() {
	for _, n := range sc.nodes {
		n.mu.RLock()
		isHealthy := n.healthy
		n.mu.RUnlock()
		if isHealthy {
			delivered := sc.hints.Deliver(n.id, n)
			if delivered > 0 {
				fmt.Printf("  Delivered %d hinted writes to %s\n", delivered, n.id)
			}
		}
	}
}

func main() {
	fmt.Println("=== Quorum Reads and Writes ===\n")

	// Show different quorum configurations
	fmt.Println("--- Quorum Configurations ---")
	configs := []QuorumConfig{
		{N: 3, W: 2, R: 2},
		{N: 3, W: 3, R: 1},
		{N: 3, W: 1, R: 3},
		{N: 5, W: 3, R: 3},
		{N: 5, W: 4, R: 2},
	}
	for _, c := range configs {
		fmt.Printf("  %s\n", c)
	}

	// Demonstrate sloppy quorum with hinted handoff
	fmt.Println("\n--- Sloppy Quorum with Hinted Handoff ---")
	cluster := NewSloppyQuorumCluster(QuorumConfig{N: 5, W: 3, R: 3})
	fmt.Printf("Cluster: N=5, W=3, R=3\n\n")

	// Normal write
	acks, hints, err := cluster.Write("key:1", "value-1")
	fmt.Printf("Write key:1: acks=%d hints=%d err=%v\n", acks, hints, err)

	// Take down 2 nodes
	cluster.nodes[3].mu.Lock()
	cluster.nodes[3].healthy = false
	cluster.nodes[3].mu.Unlock()
	cluster.nodes[4].mu.Lock()
	cluster.nodes[4].healthy = false
	cluster.nodes[4].mu.Unlock()
	fmt.Println("\nnode-3 and node-4 are DOWN")

	// Write still succeeds (3 out of 5 still healthy)
	acks, hints, err = cluster.Write("key:2", "value-2")
	fmt.Printf("Write key:2: acks=%d hints=%d err=%v\n", acks, hints, err)

	// The hint store has entries for the down nodes
	fmt.Printf("Hints stored for delivery when nodes recover\n")

	// Recover nodes and deliver hints
	fmt.Println("\nRecovering node-3 and node-4...")
	cluster.nodes[3].mu.Lock()
	cluster.nodes[3].healthy = true
	cluster.nodes[3].mu.Unlock()
	cluster.nodes[4].mu.Lock()
	cluster.nodes[4].healthy = true
	cluster.nodes[4].mu.Unlock()

	cluster.DeliverHints()

	// Verify all nodes now have the data
	fmt.Println("\nAll nodes after hint delivery:")
	for _, n := range cluster.nodes {
		v, ver, ok := n.Get("key:2")
		if ok {
			fmt.Printf("  %s: value=%q version=%d\n", n.id, v, ver)
		} else {
			fmt.Printf("  %s: no data\n", n.id)
		}
	}

	// Demonstrate quorum failure
	fmt.Println("\n--- Quorum Failure (too many nodes down) ---")
	cluster.nodes[2].mu.Lock()
	cluster.nodes[2].healthy = false
	cluster.nodes[2].mu.Unlock()
	cluster.nodes[3].mu.Lock()
	cluster.nodes[3].healthy = false
	cluster.nodes[3].mu.Unlock()
	cluster.nodes[4].mu.Lock()
	cluster.nodes[4].healthy = false
	cluster.nodes[4].mu.Unlock()
	fmt.Println("3 out of 5 nodes are DOWN")

	acks, hints, err = cluster.Write("key:3", "value-3")
	fmt.Printf("Write key:3: acks=%d hints=%d err=%v\n", acks, hints, err)

	_ = time.Now() // suppress unused import warning
}
```

---

## 12. Vector Clocks

### Detecting Concurrent Writes

In leaderless and multi-leader systems, we need to determine whether two writes are:
- **Causally related**: One happened before the other (we can pick the later one)
- **Concurrent**: Neither happened before the other (we have a conflict to resolve)

A **vector clock** (or **version vector**) is a map from node IDs to counters. Each node increments its own counter when it performs a write. By comparing vector clocks, we can determine the causal relationship between any two events.

**Rules:**
- Clock A **happens-before** clock B if every entry in A is less than or equal to the corresponding entry in B, and at least one is strictly less.
- If neither A happens-before B nor B happens-before A, the events are **concurrent**.

```go
package main

import (
	"fmt"
	"sort"
	"strings"
	"sync"
)

// VectorClock tracks causal relationships between events in a distributed system.
type VectorClock map[string]uint64

// NewVectorClock creates an empty vector clock.
func NewVectorClock() VectorClock {
	return make(VectorClock)
}

// Copy returns a deep copy of the vector clock.
func (vc VectorClock) Copy() VectorClock {
	copy := make(VectorClock, len(vc))
	for k, v := range vc {
		copy[k] = v
	}
	return copy
}

// Increment advances the clock for a specific node.
func (vc VectorClock) Increment(nodeID string) {
	vc[nodeID]++
}

// Merge combines two vector clocks by taking the maximum of each component.
func (vc VectorClock) Merge(other VectorClock) {
	for nodeID, count := range other {
		if count > vc[nodeID] {
			vc[nodeID] = count
		}
	}
}

// HappensBefore returns true if this clock causally precedes the other.
// vc < other iff all(vc[i] <= other[i]) and exists(vc[i] < other[i])
func (vc VectorClock) HappensBefore(other VectorClock) bool {
	atLeastOneLess := false

	// Check all entries in vc
	for nodeID, count := range vc {
		otherCount := other[nodeID]
		if count > otherCount {
			return false // vc has a higher value -- cannot happen before
		}
		if count < otherCount {
			atLeastOneLess = true
		}
	}

	// Check entries in other that are not in vc
	for nodeID := range other {
		if _, exists := vc[nodeID]; !exists {
			if other[nodeID] > 0 {
				atLeastOneLess = true
			}
		}
	}

	return atLeastOneLess
}

// IsConcurrentWith returns true if neither clock happens-before the other.
func (vc VectorClock) IsConcurrentWith(other VectorClock) bool {
	return !vc.HappensBefore(other) && !other.HappensBefore(vc) && !vc.Equals(other)
}

// Equals returns true if two vector clocks are identical.
func (vc VectorClock) Equals(other VectorClock) bool {
	if len(vc) != len(other) {
		// Account for zero values
		allKeys := make(map[string]bool)
		for k := range vc {
			allKeys[k] = true
		}
		for k := range other {
			allKeys[k] = true
		}
		for k := range allKeys {
			if vc[k] != other[k] {
				return false
			}
		}
		return true
	}
	for k, v := range vc {
		if other[k] != v {
			return false
		}
	}
	return true
}

func (vc VectorClock) String() string {
	parts := make([]string, 0, len(vc))
	for k, v := range vc {
		parts = append(parts, fmt.Sprintf("%s:%d", k, v))
	}
	sort.Strings(parts)
	return fmt.Sprintf("{%s}", strings.Join(parts, ", "))
}

// Relationship describes the causal relationship between two events.
type Relationship string

const (
	Before     Relationship = "HAPPENS-BEFORE"
	After      Relationship = "HAPPENS-AFTER"
	Concurrent Relationship = "CONCURRENT"
	Equal      Relationship = "EQUAL"
)

func Compare(a, b VectorClock) Relationship {
	if a.Equals(b) {
		return Equal
	}
	if a.HappensBefore(b) {
		return Before
	}
	if b.HappensBefore(a) {
		return After
	}
	return Concurrent
}

// VersionedItem stores a value with its vector clock.
type VersionedItem struct {
	Value string
	Clock VectorClock
}

// VectorClockStore is a key-value store that uses vector clocks for conflict detection.
type VectorClockStore struct {
	items map[string][]VersionedItem // key -> list of concurrent versions
	mu    sync.RWMutex
}

func NewVectorClockStore() *VectorClockStore {
	return &VectorClockStore{
		items: make(map[string][]VersionedItem),
	}
}

// Put writes a value with a vector clock context.
// If the new clock supersedes all existing versions, they are replaced.
// If it is concurrent with any existing version, both are kept (siblings).
func (store *VectorClockStore) Put(key, value string, clientClock VectorClock, nodeID string) VectorClock {
	store.mu.Lock()
	defer store.mu.Unlock()

	// Create new clock: merge client's context and increment for this node
	newClock := clientClock.Copy()
	newClock.Increment(nodeID)

	existing := store.items[key]
	var surviving []VersionedItem

	for _, item := range existing {
		if newClock.HappensBefore(item.Clock) || newClock.Equals(item.Clock) {
			// New write is older -- keep existing
			surviving = append(surviving, item)
		} else if item.Clock.HappensBefore(newClock) {
			// Existing is older -- discard it (new write supersedes)
			continue
		} else {
			// Concurrent -- keep both
			surviving = append(surviving, item)
		}
	}

	surviving = append(surviving, VersionedItem{Value: value, Clock: newClock})
	store.items[key] = surviving

	return newClock
}

// Get retrieves all current versions of a key (there may be siblings if concurrent).
func (store *VectorClockStore) Get(key string) []VersionedItem {
	store.mu.RLock()
	defer store.mu.RUnlock()
	items := store.items[key]
	result := make([]VersionedItem, len(items))
	copy(result, items)
	return result
}

func main() {
	fmt.Println("=== Vector Clocks ===\n")

	// --- Basic vector clock operations ---
	fmt.Println("--- Basic Operations ---")
	vc1 := NewVectorClock()
	vc1.Increment("A")
	vc1.Increment("A")
	vc1.Increment("B")
	fmt.Printf("vc1 = %s\n", vc1)

	vc2 := NewVectorClock()
	vc2.Increment("A")
	vc2.Increment("B")
	vc2.Increment("B")
	vc2.Increment("C")
	fmt.Printf("vc2 = %s\n", vc2)

	fmt.Printf("vc1 happens-before vc2? %v\n", vc1.HappensBefore(vc2))
	fmt.Printf("vc2 happens-before vc1? %v\n", vc2.HappensBefore(vc1))
	fmt.Printf("Concurrent? %v\n", vc1.IsConcurrentWith(vc2))
	fmt.Printf("Relationship: %s\n\n", Compare(vc1, vc2))

	// --- Causal ordering example ---
	fmt.Println("--- Causal Ordering Example ---")
	fmt.Println("Alice writes, then Bob reads Alice's write and writes based on it:")

	aliceClock := NewVectorClock()
	aliceClock.Increment("alice") // {alice:1}
	fmt.Printf("  Alice writes:    clock=%s\n", aliceClock)

	// Bob reads Alice's data, gets her clock, then writes
	bobClock := aliceClock.Copy()
	bobClock.Increment("bob") // {alice:1, bob:1}
	fmt.Printf("  Bob writes:      clock=%s\n", bobClock)

	fmt.Printf("  Alice -> Bob?    %v (Alice's write happened before Bob's)\n",
		aliceClock.HappensBefore(bobClock))

	// Charlie writes independently (did not see Alice's or Bob's writes)
	charlieClock := NewVectorClock()
	charlieClock.Increment("charlie") // {charlie:1}
	fmt.Printf("  Charlie writes:  clock=%s\n", charlieClock)
	fmt.Printf("  Bob || Charlie?  %v (concurrent -- neither saw the other)\n\n",
		bobClock.IsConcurrentWith(charlieClock))

	// --- Version vector store with conflict detection ---
	fmt.Println("--- Vector Clock Store with Conflict Detection ---")
	store := NewVectorClockStore()

	// Client 1 writes shopping cart
	fmt.Println("\nStep 1: Client 1 adds 'milk' to cart")
	clock1 := store.Put("cart", "milk", NewVectorClock(), "server-A")
	fmt.Printf("  Stored with clock: %s\n", clock1)

	// Client 2 reads cart (gets clock {server-A:1}), adds eggs
	fmt.Println("\nStep 2: Client 2 reads cart, adds 'eggs'")
	items := store.Get("cart")
	context := items[0].Clock.Copy()
	clock2 := store.Put("cart", "milk,eggs", context, "server-B")
	fmt.Printf("  Stored with clock: %s\n", clock2)

	items = store.Get("cart")
	fmt.Printf("  Cart versions: %d (previous was superseded)\n", len(items))
	for _, item := range items {
		fmt.Printf("    value=%q clock=%s\n", item.Value, item.Clock)
	}

	// Client 3 reads original cart (stale context!) and writes concurrently
	fmt.Println("\nStep 3: Client 3 (with stale context) adds 'bread'")
	staleContext := NewVectorClock()
	staleContext["server-A"] = 1 // Client 3 only saw the first version
	clock3 := store.Put("cart", "milk,bread", staleContext, "server-C")
	fmt.Printf("  Stored with clock: %s\n", clock3)

	// Now there are siblings (concurrent versions)
	items = store.Get("cart")
	fmt.Printf("\n  Cart now has %d concurrent versions (siblings):\n", len(items))
	for i, item := range items {
		fmt.Printf("    Version %d: value=%q clock=%s\n", i+1, item.Value, item.Clock)
	}

	fmt.Println("\n  The application must resolve these siblings!")
	fmt.Println("  For a shopping cart, the resolution is to take the union: milk, eggs, bread")
}
```

**Vector clocks enable precise conflict detection.** Unlike timestamps (which suffer from clock skew), vector clocks can definitively distinguish "one happened before the other" from "they happened concurrently." The trade-off is that vector clocks grow with the number of participating nodes.

---

## 13. Anti-Entropy and Merkle Trees

### Background Synchronization

Even with quorum reads and read repair, some replicas may remain out of sync. **Anti-entropy** is a background process that continually compares data across replicas and repairs any inconsistencies.

The challenge is efficiency: comparing every key-value pair between two replicas with millions of entries is expensive. **Merkle trees** (hash trees) solve this by enabling efficient detection of differences.

A Merkle tree works by:
1. Dividing the key space into ranges.
2. Computing a hash of each range.
3. Building a tree of hashes bottom-up.
4. Two replicas compare their tree roots. If roots match, all data is identical. If not, they walk down the tree to find exactly which ranges differ.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"sort"
	"strings"
)

// MerkleNode represents a node in a Merkle tree.
type MerkleNode struct {
	Hash     string
	Left     *MerkleNode
	Right    *MerkleNode
	KeyRange string // The key range this node covers (leaf nodes only)
	IsLeaf   bool
}

// MerkleTree is used for efficient comparison of data between replicas.
type MerkleTree struct {
	Root       *MerkleNode
	LeafCount  int
	bucketSize int
}

// hashData computes SHA-256 of a string.
func hashData(data string) string {
	h := sha256.Sum256([]byte(data))
	return hex.EncodeToString(h[:8]) // Use first 8 bytes for readability
}

// BuildMerkleTree constructs a Merkle tree from a sorted key-value dataset.
func BuildMerkleTree(data map[string]string, buckets int) *MerkleTree {
	// Sort keys for deterministic ordering
	keys := make([]string, 0, len(data))
	for k := range data {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	if buckets < 1 {
		buckets = 1
	}

	// Distribute keys into buckets
	bucketData := make([][]string, buckets)
	for i := 0; i < buckets; i++ {
		bucketData[i] = []string{}
	}

	for _, key := range keys {
		bucketIdx := hashToBucket(key, buckets)
		bucketData[bucketIdx] = append(bucketData[bucketIdx], key+"="+data[key])
	}

	// Create leaf nodes
	leaves := make([]*MerkleNode, buckets)
	for i := 0; i < buckets; i++ {
		sort.Strings(bucketData[i])
		content := strings.Join(bucketData[i], "|")
		leaves[i] = &MerkleNode{
			Hash:     hashData(content),
			IsLeaf:   true,
			KeyRange: fmt.Sprintf("bucket-%d (%d keys)", i, len(bucketData[i])),
		}
	}

	// Build tree bottom-up
	root := buildTree(leaves)

	return &MerkleTree{
		Root:       root,
		LeafCount:  buckets,
		bucketSize: buckets,
	}
}

func hashToBucket(key string, buckets int) int {
	h := sha256.Sum256([]byte(key))
	return int(h[0]) % buckets
}

func buildTree(nodes []*MerkleNode) *MerkleNode {
	if len(nodes) == 1 {
		return nodes[0]
	}

	// Pad to even number
	if len(nodes)%2 != 0 {
		nodes = append(nodes, nodes[len(nodes)-1])
	}

	var parents []*MerkleNode
	for i := 0; i < len(nodes); i += 2 {
		parent := &MerkleNode{
			Hash:  hashData(nodes[i].Hash + nodes[i+1].Hash),
			Left:  nodes[i],
			Right: nodes[i+1],
		}
		parents = append(parents, parent)
	}

	return buildTree(parents)
}

// Difference holds information about a detected inconsistency.
type Difference struct {
	Bucket      string
	LocalHash   string
	RemoteHash  string
}

// CompareRoots quickly checks if two trees are identical.
func CompareRoots(local, remote *MerkleTree) bool {
	if local.Root == nil || remote.Root == nil {
		return false
	}
	return local.Root.Hash == remote.Root.Hash
}

// FindDifferences walks two Merkle trees to find which buckets differ.
func FindDifferences(local, remote *MerkleNode, depth int) []Difference {
	if local == nil || remote == nil {
		return nil
	}

	if local.Hash == remote.Hash {
		return nil // This entire subtree is identical
	}

	if local.IsLeaf && remote.IsLeaf {
		return []Difference{{
			Bucket:     local.KeyRange,
			LocalHash:  local.Hash,
			RemoteHash: remote.Hash,
		}}
	}

	var diffs []Difference
	if local.Left != nil && remote.Left != nil {
		diffs = append(diffs, FindDifferences(local.Left, remote.Left, depth+1)...)
	}
	if local.Right != nil && remote.Right != nil {
		diffs = append(diffs, FindDifferences(local.Right, remote.Right, depth+1)...)
	}

	return diffs
}

// PrintTree displays the Merkle tree structure.
func PrintTree(node *MerkleNode, prefix string, isLast bool) {
	if node == nil {
		return
	}

	connector := "|-- "
	if isLast {
		connector = "`-- "
	}

	if node.IsLeaf {
		fmt.Printf("%s%s[LEAF %s] hash=%s\n", prefix, connector, node.KeyRange, node.Hash[:12])
	} else {
		fmt.Printf("%s%s[NODE] hash=%s\n", prefix, connector, node.Hash[:12])
	}

	childPrefix := prefix + "    "
	if !isLast {
		childPrefix = prefix + "|   "
	}

	if node.Left != nil {
		PrintTree(node.Left, childPrefix, false)
	}
	if node.Right != nil {
		PrintTree(node.Right, childPrefix, true)
	}
}

// AntiEntropySync represents a background synchronization process.
type AntiEntropySync struct {
	localData  map[string]string
	remoteData map[string]string
	buckets    int
}

func NewAntiEntropySync(local, remote map[string]string, buckets int) *AntiEntropySync {
	return &AntiEntropySync{
		localData:  local,
		remoteData: remote,
		buckets:    buckets,
	}
}

func (ae *AntiEntropySync) Sync() (int, int) {
	localTree := BuildMerkleTree(ae.localData, ae.buckets)
	remoteTree := BuildMerkleTree(ae.remoteData, ae.buckets)

	if CompareRoots(localTree, remoteTree) {
		fmt.Println("  Trees are identical -- no sync needed")
		return 0, 0
	}

	diffs := FindDifferences(localTree.Root, remoteTree.Root, 0)
	fmt.Printf("  Found %d differing bucket(s)\n", len(diffs))

	// In a real system, we would only exchange the keys in the differing buckets.
	// Here we simulate by comparing all keys.
	added := 0
	updated := 0

	for key, remoteVal := range ae.remoteData {
		localVal, exists := ae.localData[key]
		if !exists {
			ae.localData[key] = remoteVal
			added++
		} else if localVal != remoteVal {
			// In practice, use vector clocks or timestamps to decide which wins
			ae.localData[key] = remoteVal
			updated++
		}
	}

	return added, updated
}

func main() {
	fmt.Println("=== Anti-Entropy and Merkle Trees ===\n")

	// --- Identical replicas ---
	fmt.Println("--- Scenario 1: Identical Replicas ---")
	data1 := map[string]string{
		"user:1": "Alice", "user:2": "Bob", "user:3": "Charlie",
		"user:4": "Diana", "user:5": "Eve", "user:6": "Frank",
	}
	data2 := map[string]string{
		"user:1": "Alice", "user:2": "Bob", "user:3": "Charlie",
		"user:4": "Diana", "user:5": "Eve", "user:6": "Frank",
	}

	tree1 := BuildMerkleTree(data1, 4)
	tree2 := BuildMerkleTree(data2, 4)

	fmt.Println("Replica 1 Merkle Tree:")
	PrintTree(tree1.Root, "  ", true)
	fmt.Printf("\nRoots match: %v\n\n", CompareRoots(tree1, tree2))

	// --- Diverged replicas ---
	fmt.Println("--- Scenario 2: Diverged Replicas ---")
	data3 := map[string]string{
		"user:1": "Alice",   "user:2": "Bob",     "user:3": "Charlie",
		"user:4": "Diana",   "user:5": "Eve",     "user:6": "Frank",
		"user:7": "Grace", // Extra key
	}
	data4 := map[string]string{
		"user:1": "Alice",    "user:2": "Robert", // Different value!
		"user:3": "Charlie",  "user:4": "Diana",
		"user:5": "Eve",      "user:6": "Frank",
	}

	tree3 := BuildMerkleTree(data3, 4)
	tree4 := BuildMerkleTree(data4, 4)

	fmt.Println("Replica A (has user:7, original Bob):")
	PrintTree(tree3.Root, "  ", true)
	fmt.Println("\nReplica B (no user:7, Bob -> Robert):")
	PrintTree(tree4.Root, "  ", true)

	fmt.Printf("\nRoots match: %v\n", CompareRoots(tree3, tree4))

	diffs := FindDifferences(tree3.Root, tree4.Root, 0)
	fmt.Printf("Differences found: %d\n", len(diffs))
	for _, d := range diffs {
		fmt.Printf("  Bucket: %s\n", d.Bucket)
		fmt.Printf("    Local hash:  %s\n", d.LocalHash)
		fmt.Printf("    Remote hash: %s\n", d.RemoteHash)
	}

	// --- Anti-entropy synchronization ---
	fmt.Println("\n--- Scenario 3: Anti-Entropy Sync ---")
	localData := map[string]string{
		"key:1": "v1", "key:2": "v2", "key:3": "v3",
	}
	remoteData := map[string]string{
		"key:1": "v1", "key:2": "v2-updated", "key:3": "v3",
		"key:4": "v4", "key:5": "v5",
	}

	fmt.Printf("Before sync: local has %d keys, remote has %d keys\n", len(localData), len(remoteData))

	sync := NewAntiEntropySync(localData, remoteData, 4)
	added, updated := sync.Sync()
	fmt.Printf("After sync: added=%d updated=%d, local now has %d keys\n",
		added, updated, len(localData))

	fmt.Println("\nLocal data after sync:")
	keys := make([]string, 0, len(localData))
	for k := range localData {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	for _, k := range keys {
		fmt.Printf("  %s = %s\n", k, localData[k])
	}

	fmt.Println("\n--- Why Merkle Trees Matter ---")
	fmt.Println("Without Merkle trees: must compare ALL key-value pairs (O(n))")
	fmt.Println("With Merkle trees: compare O(log n) hashes to find differing ranges")
	fmt.Println("For a billion keys with 1000 differing, you compare ~30 hashes instead of 1 billion keys")
}
```

Merkle trees are used by Cassandra, DynamoDB, and many blockchain systems for exactly this purpose -- efficiently detecting which portions of a dataset differ between two replicas without comparing every single record.

---

## 14. Real-World Example: Building a Replicated Key-Value Store

This section brings together everything from the chapter into a complete, working replicated key-value store. It supports:

- Single-leader replication with write-ahead log
- Automatic failover with leader election and fencing tokens
- Configurable sync/async replication
- Read-after-write consistency for the writing client
- Background anti-entropy synchronization

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// ============================================================
// Core Types
// ============================================================

// WALEntry is a single entry in the write-ahead log.
type WALEntry struct {
	LSN       int64
	Operation string // "SET" or "DEL"
	Key       string
	Value     string
	Timestamp time.Time
}

// NodeState represents the operational state of a node.
type NodeState int

const (
	StateFollower NodeState = iota
	StateLeader
	StateCandidate
	StateDead
)

func (s NodeState) String() string {
	switch s {
	case StateFollower:
		return "FOLLOWER"
	case StateLeader:
		return "LEADER"
	case StateCandidate:
		return "CANDIDATE"
	case StateDead:
		return "DEAD"
	default:
		return "UNKNOWN"
	}
}

// KVNode is a single node in the replicated key-value store.
type KVNode struct {
	ID    string
	State NodeState

	// Data store
	data map[string]string

	// Write-ahead log
	wal    []WALEntry
	lastLSN int64

	// Health
	alive         bool
	lastHeartbeat time.Time

	// Fencing
	fencingToken uint64

	// Metrics
	writeCount int64
	readCount  int64

	mu sync.RWMutex
}

func NewKVNode(id string) *KVNode {
	return &KVNode{
		ID:            id,
		State:         StateFollower,
		data:          make(map[string]string),
		wal:           make([]WALEntry, 0, 1000),
		alive:         true,
		lastHeartbeat: time.Now(),
	}
}

func (n *KVNode) applyEntry(entry WALEntry) {
	switch entry.Operation {
	case "SET":
		n.data[entry.Key] = entry.Value
	case "DEL":
		delete(n.data, entry.Key)
	}
	if entry.LSN > n.lastLSN {
		n.lastLSN = entry.LSN
	}
}

func (n *KVNode) IsAlive() bool {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.alive
}

func (n *KVNode) GetLSN() int64 {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.lastLSN
}

func (n *KVNode) Kill() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.alive = false
	n.State = StateDead
}

func (n *KVNode) Revive() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.alive = true
	n.State = StateFollower
	n.fencingToken = 0
}

func (n *KVNode) GetSnapshot() map[string]string {
	n.mu.RLock()
	defer n.mu.RUnlock()
	snap := make(map[string]string, len(n.data))
	for k, v := range n.data {
		snap[k] = v
	}
	return snap
}

func (n *KVNode) RestoreSnapshot(snap map[string]string, lsn int64) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.data = make(map[string]string, len(snap))
	for k, v := range snap {
		n.data[k] = v
	}
	n.lastLSN = lsn
}

// ============================================================
// Replicated Key-Value Store
// ============================================================

// ReplicationMode controls durability guarantees.
type ReplicationMode int

const (
	AsyncReplication ReplicationMode = iota
	SemiSyncReplication
	SyncReplication
)

// ReplicatedKVStore is the main cluster managing replication.
type ReplicatedKVStore struct {
	nodes         []*KVNode
	leader        *KVNode
	fencingToken  uint64
	replicaMode   ReplicationMode
	globalLSN     int64

	// Client write tracking for read-after-write consistency
	clientWriteLSN map[string]int64 // clientID -> last write LSN

	// Anti-entropy
	antiEntropyInterval time.Duration
	stopAntiEntropy     chan struct{}

	mu sync.RWMutex
}

func NewReplicatedKVStore(nodeCount int, mode ReplicationMode) *ReplicatedKVStore {
	nodes := make([]*KVNode, nodeCount)
	for i := 0; i < nodeCount; i++ {
		nodes[i] = NewKVNode(fmt.Sprintf("node-%d", i))
	}

	store := &ReplicatedKVStore{
		nodes:          nodes,
		replicaMode:    mode,
		clientWriteLSN: make(map[string]int64),
		stopAntiEntropy: make(chan struct{}),
	}

	// Elect initial leader (node-0)
	store.electLeader(nodes[0])
	return store
}

func (s *ReplicatedKVStore) electLeader(node *KVNode) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// Demote current leader if any
	if s.leader != nil {
		s.leader.mu.Lock()
		s.leader.State = StateFollower
		s.leader.fencingToken = 0
		s.leader.mu.Unlock()
	}

	s.fencingToken++
	node.mu.Lock()
	node.State = StateLeader
	node.fencingToken = s.fencingToken
	node.mu.Unlock()

	s.leader = node
}

// Set writes a key-value pair through the leader.
func (s *ReplicatedKVStore) Set(clientID, key, value string) error {
	s.mu.RLock()
	leader := s.leader
	s.mu.RUnlock()

	if leader == nil || !leader.IsAlive() {
		// Attempt failover
		if err := s.performFailover(); err != nil {
			return fmt.Errorf("no leader available: %w", err)
		}
		s.mu.RLock()
		leader = s.leader
		s.mu.RUnlock()
	}

	// Create WAL entry
	lsn := atomic.AddInt64(&s.globalLSN, 1)
	entry := WALEntry{
		LSN:       lsn,
		Operation: "SET",
		Key:       key,
		Value:     value,
		Timestamp: time.Now(),
	}

	// Apply to leader
	leader.mu.Lock()
	leader.wal = append(leader.wal, entry)
	leader.applyEntry(entry)
	atomic.AddInt64(&leader.writeCount, 1)
	leader.mu.Unlock()

	// Track client's latest write LSN
	s.mu.Lock()
	s.clientWriteLSN[clientID] = lsn
	s.mu.Unlock()

	// Replicate to followers
	s.replicateToFollowers(entry)

	return nil
}

func (s *ReplicatedKVStore) replicateToFollowers(entry WALEntry) {
	s.mu.RLock()
	leader := s.leader
	mode := s.replicaMode
	s.mu.RUnlock()

	var followers []*KVNode
	for _, n := range s.nodes {
		if n != leader && n.IsAlive() {
			followers = append(followers, n)
		}
	}

	applyToFollower := func(f *KVNode) {
		f.mu.Lock()
		defer f.mu.Unlock()
		f.wal = append(f.wal, entry)
		f.applyEntry(entry)
	}

	switch mode {
	case AsyncReplication:
		for _, f := range followers {
			go applyToFollower(f)
		}

	case SemiSyncReplication:
		if len(followers) == 0 {
			return
		}
		done := make(chan struct{}, len(followers))
		for _, f := range followers {
			go func(follower *KVNode) {
				applyToFollower(follower)
				done <- struct{}{}
			}(f)
		}
		<-done // Wait for at least one

	case SyncReplication:
		var wg sync.WaitGroup
		for _, f := range followers {
			wg.Add(1)
			go func(follower *KVNode) {
				defer wg.Done()
				applyToFollower(follower)
			}(f)
		}
		wg.Wait()
	}
}

// Get reads a key, respecting read-after-write consistency for the given client.
func (s *ReplicatedKVStore) Get(clientID, key string) (string, string, error) {
	s.mu.RLock()
	requiredLSN := s.clientWriteLSN[clientID]
	leader := s.leader
	s.mu.RUnlock()

	// If the client wrote recently, we need a node that has caught up
	if requiredLSN > 0 {
		// Try followers first
		for _, n := range s.nodes {
			if n != leader && n.IsAlive() && n.GetLSN() >= requiredLSN {
				n.mu.RLock()
				val, ok := n.data[key]
				atomic.AddInt64(&n.readCount, 1)
				n.mu.RUnlock()
				if ok {
					return val, n.ID, nil
				}
			}
		}
		// Fall back to leader for consistency
		if leader != nil && leader.IsAlive() {
			leader.mu.RLock()
			val, ok := leader.data[key]
			atomic.AddInt64(&leader.readCount, 1)
			leader.mu.RUnlock()
			if ok {
				return val, leader.ID + " (consistency fallback)", nil
			}
			return "", leader.ID, fmt.Errorf("key %q not found", key)
		}
	}

	// No consistency requirement -- read from any healthy node
	healthyNodes := make([]*KVNode, 0)
	for _, n := range s.nodes {
		if n.IsAlive() {
			healthyNodes = append(healthyNodes, n)
		}
	}
	if len(healthyNodes) == 0 {
		return "", "", fmt.Errorf("no healthy nodes available")
	}

	node := healthyNodes[rand.Intn(len(healthyNodes))]
	node.mu.RLock()
	val, ok := node.data[key]
	atomic.AddInt64(&node.readCount, 1)
	node.mu.RUnlock()

	if !ok {
		return "", node.ID, fmt.Errorf("key %q not found", key)
	}
	return val, node.ID, nil
}

// Delete removes a key through the leader.
func (s *ReplicatedKVStore) Delete(clientID, key string) error {
	s.mu.RLock()
	leader := s.leader
	s.mu.RUnlock()

	if leader == nil || !leader.IsAlive() {
		if err := s.performFailover(); err != nil {
			return err
		}
		s.mu.RLock()
		leader = s.leader
		s.mu.RUnlock()
	}

	lsn := atomic.AddInt64(&s.globalLSN, 1)
	entry := WALEntry{
		LSN:       lsn,
		Operation: "DEL",
		Key:       key,
		Timestamp: time.Now(),
	}

	leader.mu.Lock()
	leader.wal = append(leader.wal, entry)
	leader.applyEntry(entry)
	leader.mu.Unlock()

	s.mu.Lock()
	s.clientWriteLSN[clientID] = lsn
	s.mu.Unlock()

	s.replicateToFollowers(entry)
	return nil
}

// performFailover promotes the most up-to-date follower to leader.
func (s *ReplicatedKVStore) performFailover() error {
	s.mu.Lock()
	defer s.mu.Unlock()

	var best *KVNode
	var bestLSN int64 = -1

	for _, n := range s.nodes {
		if n.IsAlive() && n.State != StateLeader {
			lsn := n.GetLSN()
			if lsn > bestLSN {
				bestLSN = lsn
				best = n
			}
		}
	}

	if best == nil {
		return fmt.Errorf("no healthy followers available for failover")
	}

	// Demote old leader
	if s.leader != nil {
		s.leader.mu.Lock()
		s.leader.State = StateFollower
		s.leader.fencingToken = 0
		s.leader.mu.Unlock()
	}

	// Promote new leader
	s.fencingToken++
	best.mu.Lock()
	best.State = StateLeader
	best.fencingToken = s.fencingToken
	best.mu.Unlock()
	s.leader = best

	return nil
}

// RecoverNode brings a dead node back online and catches it up.
func (s *ReplicatedKVStore) RecoverNode(nodeID string) error {
	var target *KVNode
	for _, n := range s.nodes {
		if n.ID == nodeID {
			target = n
			break
		}
	}
	if target == nil {
		return fmt.Errorf("node %s not found", nodeID)
	}

	target.Revive()

	// Find the leader and catch up from its WAL
	s.mu.RLock()
	leader := s.leader
	s.mu.RUnlock()

	if leader == nil || !leader.IsAlive() {
		return fmt.Errorf("no leader to catch up from")
	}

	targetLSN := target.GetLSN()

	leader.mu.RLock()
	var entriesToApply []WALEntry
	for _, entry := range leader.wal {
		if entry.LSN > targetLSN {
			entriesToApply = append(entriesToApply, entry)
		}
	}
	leader.mu.RUnlock()

	if len(entriesToApply) == 0 {
		// Need full snapshot
		snap := leader.GetSnapshot()
		target.RestoreSnapshot(snap, leader.GetLSN())
		return nil
	}

	target.mu.Lock()
	for _, entry := range entriesToApply {
		target.wal = append(target.wal, entry)
		target.applyEntry(entry)
	}
	target.mu.Unlock()

	return nil
}

// Status returns a summary of the cluster state.
func (s *ReplicatedKVStore) Status() string {
	s.mu.RLock()
	defer s.mu.RUnlock()

	var sb strings.Builder
	sb.WriteString("Cluster Status:\n")
	sb.WriteString(fmt.Sprintf("  Fencing Token: %d\n", s.fencingToken))

	leaderID := "(none)"
	if s.leader != nil {
		leaderID = s.leader.ID
	}
	sb.WriteString(fmt.Sprintf("  Leader: %s\n", leaderID))
	sb.WriteString("  Nodes:\n")

	for _, n := range s.nodes {
		n.mu.RLock()
		sb.WriteString(fmt.Sprintf("    %s: state=%-9s alive=%-5v lsn=%-4d keys=%-4d writes=%-4d reads=%d\n",
			n.ID, n.State, n.alive, n.lastLSN, len(n.data),
			atomic.LoadInt64(&n.writeCount), atomic.LoadInt64(&n.readCount)))
		n.mu.RUnlock()
	}

	return sb.String()
}

// AllKeys returns all keys in the store (read from leader).
func (s *ReplicatedKVStore) AllKeys() []string {
	s.mu.RLock()
	leader := s.leader
	s.mu.RUnlock()

	if leader == nil {
		return nil
	}

	leader.mu.RLock()
	defer leader.mu.RUnlock()
	keys := make([]string, 0, len(leader.data))
	for k := range leader.data {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	return keys
}

func main() {
	fmt.Println("=== Replicated Key-Value Store ===\n")

	// Create a 5-node cluster with semi-synchronous replication
	store := NewReplicatedKVStore(5, SemiSyncReplication)
	fmt.Println(store.Status())

	// --- Phase 1: Normal Operations ---
	fmt.Println("--- Phase 1: Normal Operations ---")
	clients := []string{"client-A", "client-B", "client-C"}
	testData := map[string]string{
		"user:1:name":  "Alice",
		"user:2:name":  "Bob",
		"user:3:name":  "Charlie",
		"user:1:email": "alice@example.com",
		"user:2:email": "bob@example.com",
		"config:theme": "dark",
		"config:lang":  "en",
	}

	for key, value := range testData {
		client := clients[rand.Intn(len(clients))]
		if err := store.Set(client, key, value); err != nil {
			fmt.Printf("  Write error: %v\n", err)
		}
	}
	time.Sleep(20 * time.Millisecond) // Let async replication finish

	fmt.Printf("Wrote %d key-value pairs\n", len(testData))
	fmt.Println(store.Status())

	// Read some data
	fmt.Println("--- Reads ---")
	for _, key := range []string{"user:1:name", "user:2:email", "config:theme"} {
		val, source, err := store.Get("client-A", key)
		if err != nil {
			fmt.Printf("  GET %s: error=%v\n", key, err)
		} else {
			fmt.Printf("  GET %s = %q (from %s)\n", key, val, source)
		}
	}

	// --- Phase 2: Leader Crash and Failover ---
	fmt.Println("\n--- Phase 2: Leader Crash ---")
	store.nodes[0].Kill()
	fmt.Println("node-0 (leader) crashed!")

	// Next write triggers automatic failover
	err := store.Set("client-A", "user:4:name", "Diana")
	if err != nil {
		fmt.Printf("  Write after crash: %v\n", err)
	} else {
		fmt.Println("  Write succeeded after automatic failover")
	}
	fmt.Println(store.Status())

	// Read-after-write consistency check
	val, source, err := store.Get("client-A", "user:4:name")
	if err != nil {
		fmt.Printf("  Read-after-write: error=%v\n", err)
	} else {
		fmt.Printf("  Read-after-write: user:4:name = %q (from %s)\n", val, source)
	}

	// --- Phase 3: Node Recovery ---
	fmt.Println("\n--- Phase 3: Node Recovery ---")
	fmt.Println("Recovering node-0...")
	if err := store.RecoverNode("node-0"); err != nil {
		fmt.Printf("  Recovery error: %v\n", err)
	} else {
		fmt.Println("  node-0 recovered and caught up")
	}
	fmt.Println(store.Status())

	// Verify node-0 has all the data
	node0 := store.nodes[0]
	node0.mu.RLock()
	fmt.Printf("node-0 data count: %d, LSN: %d\n", len(node0.data), node0.lastLSN)
	node0.mu.RUnlock()

	// --- Phase 4: Multiple Failures ---
	fmt.Println("\n--- Phase 4: Cascading Failures ---")
	store.nodes[1].Kill()
	fmt.Println("node-1 crashed!")
	store.nodes[2].Kill()
	fmt.Println("node-2 crashed!")

	// Writes should still work (failover to remaining nodes)
	err = store.Set("client-B", "user:5:name", "Eve")
	if err != nil {
		fmt.Printf("  Write with 2 failures: %v\n", err)
	} else {
		fmt.Println("  Write succeeded with 2 nodes down")
	}
	fmt.Println(store.Status())

	// --- Phase 5: Delete Operation ---
	fmt.Println("--- Phase 5: Delete ---")
	err = store.Delete("client-C", "config:theme")
	if err != nil {
		fmt.Printf("  Delete error: %v\n", err)
	} else {
		fmt.Println("  Deleted config:theme")
	}

	_, _, err = store.Get("client-C", "config:theme")
	fmt.Printf("  Read config:theme after delete: %v\n", err)

	// --- Phase 6: Full Recovery ---
	fmt.Println("\n--- Phase 6: Full Recovery ---")
	_ = store.RecoverNode("node-1")
	_ = store.RecoverNode("node-2")
	time.Sleep(20 * time.Millisecond)
	fmt.Println("All nodes recovered.")
	fmt.Println(store.Status())

	// Verify consistency -- all nodes should have the same data
	fmt.Println("--- Data Consistency Check ---")
	allKeys := store.AllKeys()
	fmt.Printf("Total keys: %d\n", len(allKeys))

	s.mu.RLock() // This would be store.mu, but for this demo we read directly
	for _, n := range store.nodes {
		n.mu.RLock()
		fmt.Printf("  %s: %d keys, LSN=%d\n", n.ID, len(n.data), n.lastLSN)
		n.mu.RUnlock()
	}

	fmt.Println("\n--- Final Key-Value Dump (from leader) ---")
	for _, key := range allKeys {
		val, source, _ := store.Get("admin", key)
		fmt.Printf("  %s = %q (from %s)\n", key, val, source)
	}
}
```

Wait -- the above code has a bug at the end where I reference `s.mu.RLock()` instead of accessing nodes directly. Let me fix the complete program. Here is the corrected version:

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// ============================================================
// Core Types
// ============================================================

type WALEntry struct {
	LSN       int64
	Operation string
	Key       string
	Value     string
	Timestamp time.Time
}

type NodeState int

const (
	StateFollower NodeState = iota
	StateLeader
	StateDead
)

func (s NodeState) String() string {
	switch s {
	case StateFollower:
		return "FOLLOWER"
	case StateLeader:
		return "LEADER"
	case StateDead:
		return "DEAD"
	default:
		return "UNKNOWN"
	}
}

type KVNode struct {
	ID         string
	State      NodeState
	data       map[string]string
	wal        []WALEntry
	lastLSN    int64
	alive      bool
	fencingTok uint64
	writeCount int64
	readCount  int64
	mu         sync.RWMutex
}

func NewKVNode(id string) *KVNode {
	return &KVNode{
		ID:    id,
		State: StateFollower,
		data:  make(map[string]string),
		wal:   make([]WALEntry, 0, 1000),
		alive: true,
	}
}

func (n *KVNode) applyEntry(entry WALEntry) {
	switch entry.Operation {
	case "SET":
		n.data[entry.Key] = entry.Value
	case "DEL":
		delete(n.data, entry.Key)
	}
	if entry.LSN > n.lastLSN {
		n.lastLSN = entry.LSN
	}
}

func (n *KVNode) IsAlive() bool {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.alive
}

func (n *KVNode) GetLSN() int64 {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return n.lastLSN
}

func (n *KVNode) Kill() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.alive = false
	n.State = StateDead
}

func (n *KVNode) Revive() {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.alive = true
	n.State = StateFollower
	n.fencingTok = 0
}

type ReplicatedKV struct {
	nodes          []*KVNode
	leader         *KVNode
	fencingToken   uint64
	globalLSN      int64
	clientWriteLSN map[string]int64
	mu             sync.RWMutex
}

func NewReplicatedKV(nodeCount int) *ReplicatedKV {
	nodes := make([]*KVNode, nodeCount)
	for i := 0; i < nodeCount; i++ {
		nodes[i] = NewKVNode(fmt.Sprintf("node-%d", i))
	}
	store := &ReplicatedKV{
		nodes:          nodes,
		clientWriteLSN: make(map[string]int64),
	}
	store.promoteLeader(nodes[0])
	return store
}

func (kv *ReplicatedKV) promoteLeader(node *KVNode) {
	if kv.leader != nil {
		kv.leader.mu.Lock()
		kv.leader.State = StateFollower
		kv.leader.fencingTok = 0
		kv.leader.mu.Unlock()
	}
	kv.fencingToken++
	node.mu.Lock()
	node.State = StateLeader
	node.fencingTok = kv.fencingToken
	node.mu.Unlock()
	kv.leader = node
}

func (kv *ReplicatedKV) failover() error {
	var best *KVNode
	bestLSN := int64(-1)
	for _, n := range kv.nodes {
		if n.IsAlive() {
			n.mu.RLock()
			isLeader := n.State == StateLeader
			n.mu.RUnlock()
			if !isLeader {
				lsn := n.GetLSN()
				if lsn > bestLSN {
					bestLSN = lsn
					best = n
				}
			}
		}
	}
	if best == nil {
		return fmt.Errorf("no healthy followers for failover")
	}
	kv.promoteLeader(best)
	return nil
}

func (kv *ReplicatedKV) Set(clientID, key, value string) error {
	kv.mu.Lock()
	leader := kv.leader
	if leader == nil || !leader.IsAlive() {
		if err := kv.failover(); err != nil {
			kv.mu.Unlock()
			return err
		}
		leader = kv.leader
	}
	kv.mu.Unlock()

	lsn := atomic.AddInt64(&kv.globalLSN, 1)
	entry := WALEntry{LSN: lsn, Operation: "SET", Key: key, Value: value, Timestamp: time.Now()}

	leader.mu.Lock()
	leader.wal = append(leader.wal, entry)
	leader.applyEntry(entry)
	atomic.AddInt64(&leader.writeCount, 1)
	leader.mu.Unlock()

	kv.mu.Lock()
	kv.clientWriteLSN[clientID] = lsn
	kv.mu.Unlock()

	// Semi-sync: wait for one follower
	done := make(chan struct{}, len(kv.nodes))
	for _, n := range kv.nodes {
		if n != leader && n.IsAlive() {
			go func(f *KVNode) {
				f.mu.Lock()
				f.wal = append(f.wal, entry)
				f.applyEntry(entry)
				f.mu.Unlock()
				done <- struct{}{}
			}(n)
		}
	}
	// Wait for at least one ack (with timeout)
	select {
	case <-done:
	case <-time.After(100 * time.Millisecond):
	}
	return nil
}

func (kv *ReplicatedKV) Get(clientID, key string) (string, string, error) {
	kv.mu.RLock()
	requiredLSN := kv.clientWriteLSN[clientID]
	leader := kv.leader
	kv.mu.RUnlock()

	// Try a follower that meets the consistency requirement
	for _, n := range kv.nodes {
		if n != leader && n.IsAlive() && n.GetLSN() >= requiredLSN {
			n.mu.RLock()
			val, ok := n.data[key]
			atomic.AddInt64(&n.readCount, 1)
			n.mu.RUnlock()
			if ok {
				return val, n.ID, nil
			}
		}
	}

	// Fall back to leader
	if leader != nil && leader.IsAlive() {
		leader.mu.RLock()
		val, ok := leader.data[key]
		atomic.AddInt64(&leader.readCount, 1)
		leader.mu.RUnlock()
		if ok {
			return val, leader.ID + " (leader)", nil
		}
		return "", "", fmt.Errorf("key %q not found", key)
	}
	return "", "", fmt.Errorf("no healthy nodes")
}

func (kv *ReplicatedKV) Delete(clientID, key string) error {
	kv.mu.Lock()
	leader := kv.leader
	if leader == nil || !leader.IsAlive() {
		if err := kv.failover(); err != nil {
			kv.mu.Unlock()
			return err
		}
		leader = kv.leader
	}
	kv.mu.Unlock()

	lsn := atomic.AddInt64(&kv.globalLSN, 1)
	entry := WALEntry{LSN: lsn, Operation: "DEL", Key: key, Timestamp: time.Now()}

	leader.mu.Lock()
	leader.wal = append(leader.wal, entry)
	leader.applyEntry(entry)
	leader.mu.Unlock()

	kv.mu.Lock()
	kv.clientWriteLSN[clientID] = lsn
	kv.mu.Unlock()

	for _, n := range kv.nodes {
		if n != leader && n.IsAlive() {
			go func(f *KVNode) {
				f.mu.Lock()
				f.wal = append(f.wal, entry)
				f.applyEntry(entry)
				f.mu.Unlock()
			}(n)
		}
	}
	return nil
}

func (kv *ReplicatedKV) RecoverNode(nodeID string) error {
	var target *KVNode
	for _, n := range kv.nodes {
		if n.ID == nodeID {
			target = n
			break
		}
	}
	if target == nil {
		return fmt.Errorf("node %s not found", nodeID)
	}
	target.Revive()

	kv.mu.RLock()
	leader := kv.leader
	kv.mu.RUnlock()

	if leader == nil || !leader.IsAlive() {
		return fmt.Errorf("no leader for catch-up")
	}

	targetLSN := target.GetLSN()
	leader.mu.RLock()
	var entries []WALEntry
	for _, e := range leader.wal {
		if e.LSN > targetLSN {
			entries = append(entries, e)
		}
	}
	leader.mu.RUnlock()

	if len(entries) == 0 && targetLSN < leader.GetLSN() {
		// Need snapshot
		leader.mu.RLock()
		snap := make(map[string]string, len(leader.data))
		for k, v := range leader.data {
			snap[k] = v
		}
		leaderLSN := leader.lastLSN
		leader.mu.RUnlock()

		target.mu.Lock()
		target.data = snap
		target.lastLSN = leaderLSN
		target.mu.Unlock()
	} else {
		target.mu.Lock()
		for _, e := range entries {
			target.wal = append(target.wal, e)
			target.applyEntry(e)
		}
		target.mu.Unlock()
	}
	return nil
}

func (kv *ReplicatedKV) Status() string {
	kv.mu.RLock()
	defer kv.mu.RUnlock()

	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("Cluster [fencing=%d", kv.fencingToken))
	if kv.leader != nil {
		sb.WriteString(fmt.Sprintf(" leader=%s", kv.leader.ID))
	}
	sb.WriteString("]\n")

	for _, n := range kv.nodes {
		n.mu.RLock()
		sb.WriteString(fmt.Sprintf("  %-8s state=%-9s alive=%-5v lsn=%-4d keys=%-3d writes=%-3d reads=%d\n",
			n.ID, n.State, n.alive, n.lastLSN, len(n.data),
			atomic.LoadInt64(&n.writeCount), atomic.LoadInt64(&n.readCount)))
		n.mu.RUnlock()
	}
	return sb.String()
}

func main() {
	fmt.Println("======================================================")
	fmt.Println("  Replicated Key-Value Store -- Complete Demo")
	fmt.Println("======================================================\n")

	store := NewReplicatedKV(5)
	fmt.Println(store.Status())

	// Phase 1: Normal writes
	fmt.Println("--- Phase 1: Writing Data ---")
	data := []struct{ key, value string }{
		{"user:1:name", "Alice"},
		{"user:2:name", "Bob"},
		{"user:3:name", "Charlie"},
		{"user:1:email", "alice@example.com"},
		{"user:2:email", "bob@example.com"},
		{"config:theme", "dark"},
		{"config:lang", "en"},
		{"session:abc", "active"},
	}
	for _, d := range data {
		client := fmt.Sprintf("client-%d", rand.Intn(3))
		_ = store.Set(client, d.key, d.value)
	}
	time.Sleep(30 * time.Millisecond)
	fmt.Printf("Wrote %d entries.\n", len(data))
	fmt.Println(store.Status())

	// Phase 2: Reads
	fmt.Println("--- Phase 2: Reading Data ---")
	for _, key := range []string{"user:1:name", "user:2:email", "config:theme"} {
		val, source, err := store.Get("client-0", key)
		if err != nil {
			fmt.Printf("  GET %-20s ERROR: %v\n", key, err)
		} else {
			fmt.Printf("  GET %-20s = %-25q from %s\n", key, val, source)
		}
	}

	// Phase 3: Leader crash + failover
	fmt.Println("\n--- Phase 3: Leader Crash + Automatic Failover ---")
	store.nodes[0].Kill()
	fmt.Println("node-0 (leader) KILLED")

	err := store.Set("client-0", "user:4:name", "Diana")
	if err != nil {
		fmt.Printf("Write after crash: %v\n", err)
	} else {
		fmt.Println("Write succeeded -- failover happened transparently")
	}
	fmt.Println(store.Status())

	// Read-after-write check
	val, src, _ := store.Get("client-0", "user:4:name")
	fmt.Printf("Read-after-write: user:4:name = %q from %s\n", val, src)

	// Phase 4: Recovery
	fmt.Println("\n--- Phase 4: Node Recovery ---")
	err = store.RecoverNode("node-0")
	if err != nil {
		fmt.Printf("Recovery failed: %v\n", err)
	} else {
		fmt.Println("node-0 recovered and caught up")
	}
	time.Sleep(10 * time.Millisecond)
	fmt.Println(store.Status())

	// Phase 5: More failures
	fmt.Println("--- Phase 5: Multiple Failures ---")
	store.nodes[1].Kill()
	store.nodes[3].Kill()
	fmt.Println("node-1 and node-3 KILLED")

	_ = store.Set("client-1", "resilience", "tested")
	val, src, _ = store.Get("client-1", "resilience")
	fmt.Printf("Write + read with 2 nodes down: %q from %s\n", val, src)
	fmt.Println(store.Status())

	// Phase 6: Delete
	fmt.Println("--- Phase 6: Delete ---")
	_ = store.Delete("client-2", "session:abc")
	_, _, err = store.Get("client-2", "session:abc")
	fmt.Printf("After delete: %v\n", err)

	// Phase 7: Full recovery
	fmt.Println("\n--- Phase 7: Full Recovery ---")
	_ = store.RecoverNode("node-1")
	_ = store.RecoverNode("node-3")
	time.Sleep(20 * time.Millisecond)
	fmt.Println("All nodes recovered.")
	fmt.Println(store.Status())

	// Verify consistency
	fmt.Println("--- Consistency Verification ---")
	leaderKeys := make(map[string]string)
	store.mu.RLock()
	leader := store.leader
	store.mu.RUnlock()

	leader.mu.RLock()
	for k, v := range leader.data {
		leaderKeys[k] = v
	}
	leader.mu.RUnlock()

	keys := make([]string, 0, len(leaderKeys))
	for k := range leaderKeys {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	fmt.Printf("Leader has %d keys:\n", len(keys))
	for _, k := range keys {
		fmt.Printf("  %s = %q\n", k, leaderKeys[k])
	}

	// Check each node matches the leader
	allMatch := true
	for _, n := range store.nodes {
		if !n.IsAlive() {
			continue
		}
		n.mu.RLock()
		nodeKeyCount := len(n.data)
		match := nodeKeyCount == len(leaderKeys)
		if match {
			for k, v := range leaderKeys {
				if n.data[k] != v {
					match = false
					break
				}
			}
		}
		n.mu.RUnlock()
		if !match {
			fmt.Printf("  %s: INCONSISTENT (has %d keys)\n", n.ID, nodeKeyCount)
			allMatch = false
		}
	}
	if allMatch {
		fmt.Println("\nAll healthy nodes are consistent with the leader.")
	}
}
```

This complete example demonstrates:
- **Write-ahead logging**: Every write is first logged, then applied.
- **Semi-synchronous replication**: Waits for one follower before acknowledging.
- **Automatic failover**: Promotes the most up-to-date follower on leader failure.
- **Fencing tokens**: Monotonically increasing tokens prevent stale leaders.
- **Read-after-write consistency**: Clients see their own writes by tracking LSN positions.
- **Node recovery**: Dead nodes catch up via WAL replay or snapshot restoration.
- **Consistency verification**: All replicas converge to the same state after recovery.

---

## 15. Key Takeaways

### Replication Architectures

1. **Single-leader replication** is the simplest and most common. All writes go through one node. Use it when write throughput from a single node is sufficient (which it is for most applications).

2. **Multi-leader replication** is needed when you have multiple datacenters or offline-capable clients. The hard part is conflict resolution -- there is no single correct answer, and your choice depends on your application semantics.

3. **Leaderless replication** (Dynamo-style) provides the highest availability because any node can serve reads and writes. The trade-off is weaker consistency guarantees and the need for quorum management.

### Consistency and Lag

4. **Replication lag is inevitable** with asynchronous replication. The three main problems it causes are: read-after-write violations, non-monotonic reads, and causal ordering violations.

5. **Read-after-write consistency** can be achieved by routing reads to the leader after writes, tracking write positions per client, or using sticky sessions.

6. **Monotonic reads** are guaranteed by always reading from the same replica (sticky sessions) or by tracking the minimum required version.

### Conflict Resolution

7. **Last-write-wins (LWW)** is simple but loses data. Use it only when data loss is acceptable (e.g., caching, session storage).

8. **CRDTs** (Conflict-free Replicated Data Types) are mathematically guaranteed to converge. G-Counters, PN-Counters, and OR-Sets cover many common use cases without coordination.

9. **Vector clocks** are the correct way to detect concurrent writes. Timestamps are unreliable because of clock skew.

### Operational Concerns

10. **Failover is hard**. The three critical problems are: detecting the failure (without false positives), electing a new leader (without split-brain), and preventing the old leader from corrupting data (fencing tokens).

11. **Anti-entropy with Merkle trees** enables efficient background synchronization between replicas by comparing O(log n) hashes instead of O(n) key-value pairs.

12. **Quorum math matters**: W + R > N guarantees overlap between read and write sets. Choose W and R based on your read/write ratio and fault tolerance requirements.

### Go-Specific Lessons

13. **Goroutines model distributed nodes well**. Each goroutine can represent a node, with channels or shared memory simulating network communication.

14. **sync.RWMutex is essential** for node-level data protection. Reads are far more frequent than writes, so RWMutex significantly outperforms Mutex.

15. **Channels with timeouts** naturally model network communication with failure detection. `select` with `time.After` is the Go idiom for heartbeat timeouts.

---

## 16. Practice Exercises

### Exercise 1: Tunable Consistency

Extend the `DynamoCluster` from Section 10 to support client-specified consistency levels per request:

```go
type ConsistencyLevel int

const (
	One     ConsistencyLevel = iota // Read/write from one node
	Quorum                          // Read/write from W or R nodes
	All                             // Read/write from all nodes
)

func (dc *DynamoCluster) WriteWithLevel(key, value string, level ConsistencyLevel) error {
	// Implement: adjust the effective W based on the level
}

func (dc *DynamoCluster) ReadWithLevel(key string, level ConsistencyLevel) (string, error) {
	// Implement: adjust the effective R based on the level
}
```

This is how Cassandra works -- each query specifies its own consistency level.

### Exercise 2: Conflict-Free Shopping Cart

Build a shopping cart using CRDTs that supports concurrent adds and removes from multiple devices:

```go
type ShoppingCart struct {
	// Use an OR-Set for items and a PN-Counter per item for quantities
	items      *ORSet
	quantities map[string]*PNCounter // item -> quantity
}

func (sc *ShoppingCart) AddItem(item string, qty int) { /* ... */ }
func (sc *ShoppingCart) RemoveItem(item string)       { /* ... */ }
func (sc *ShoppingCart) Merge(other *ShoppingCart)     { /* ... */ }
func (sc *ShoppingCart) Contents() map[string]int64    { /* ... */ }
```

Test it by:
1. Creating two carts (simulating phone and laptop).
2. Adding items on both devices concurrently.
3. Removing an item on one device.
4. Merging and verifying the result preserves all additions and respects the removal.

### Exercise 3: Implement Gossip Protocol

Build a gossip-based failure detection system where nodes periodically exchange state:

```go
type GossipNode struct {
	id        string
	heartbeat map[string]int64    // nodeID -> heartbeat counter
	alive     map[string]bool
	timeout   time.Duration
}

func (g *GossipNode) Tick()                           { /* increment own heartbeat */ }
func (g *GossipNode) Gossip(peer *GossipNode)         { /* exchange heartbeats */ }
func (g *GossipNode) DetectFailures() []string         { /* nodes with stale heartbeats */ }
```

Properties to verify:
- A crashed node is detected within a bounded time.
- A recovered node is re-detected as alive.
- Gossip propagates information in O(log N) rounds.

### Exercise 4: Read Repair with Conflict Resolution

Extend the `DynamoCluster` to perform automatic read repair with configurable conflict resolution:

```go
type ConflictResolver func(values []DynamoValue) DynamoValue

// Built-in resolvers
func LWWResolver(values []DynamoValue) DynamoValue { /* latest timestamp wins */ }
func HighestVersionResolver(values []DynamoValue) DynamoValue { /* highest version wins */ }

func (dc *DynamoCluster) ReadAndRepair(key string, resolver ConflictResolver) (DynamoValue, error) {
	// 1. Read from all nodes
	// 2. Detect stale values
	// 3. Apply resolver to pick winner
	// 4. Write winner back to stale nodes
}
```

### Exercise 5: Multi-Leader with Operation Transforms

Implement a collaborative text editor using multi-leader replication with operation transforms (OT):

```go
type Operation struct {
	Type     string // "insert" or "delete"
	Position int
	Char     rune
	Origin   string // which node generated this
}

type OTDocument struct {
	content []rune
	history []Operation
}

func Transform(op1, op2 Operation) (Operation, Operation) {
	// Transform op1 and op2 so they can be applied in either order
	// and produce the same result
}
```

This is the foundation of how Google Docs works. Two users can type simultaneously, and their operations are transformed so that the final document converges regardless of the order operations are received.

### Exercise 6: Merkle Tree Synchronization with Bandwidth Tracking

Extend the Merkle tree implementation to track how much data is transferred during synchronization:

```go
type SyncStats struct {
	HashesCompared   int
	KeysTransferred  int
	BytesTransferred int64
	Duration         time.Duration
}

func EfficientSync(local, remote map[string]string, buckets int) SyncStats {
	// 1. Build Merkle trees on both sides
	// 2. Compare trees top-down, counting hash comparisons
	// 3. Only transfer keys in differing buckets
	// 4. Report stats
}
```

Compare the bandwidth used by Merkle tree sync vs naive full-data sync for datasets of 100, 10,000, and 1,000,000 keys where only 0.1% of keys differ.

---

**Next Chapter:** [Chapter 36: Partitioning and Sharding](./36-partitioning-and-sharding.md) -- how to split data across multiple nodes when a single replica cannot hold the entire dataset, consistent hashing, partition rebalancing, and secondary indexes in a partitioned database.
