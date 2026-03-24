# Chapter 36: Partitioning & Sharding

> **Level: Distributed Systems / DDIA Series** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 15 (Generics), 16 (Context), 17 (Advanced Concurrency), 21 (Microservices), 35 (Replication)**

This chapter is based on **Chapter 6 of _Designing Data-Intensive Applications_ by Martin Kleppmann**. While the previous chapters in this DDIA series covered how to replicate data (keeping copies on multiple nodes for fault tolerance), this chapter tackles a fundamentally different question: **how do you split your data across multiple nodes so that no single node has to store or process everything?**

Partitioning (also called **sharding**) is the primary mechanism for **horizontal scaling**. When your dataset grows beyond what a single machine can handle -- or when your query load exceeds what a single machine can serve -- you partition the data so that each node is responsible for a subset of it. Every large-scale system you interact with daily (Google, Amazon, Facebook, Netflix) relies on partitioning.

We will implement every major partitioning strategy in Go, building complete, runnable systems that simulate distributed behavior using goroutines, channels, and in-memory data structures. By the end of this chapter, you will have built a fully functional sharded key-value store with consistent hashing, rebalancing, request routing, and scatter-gather queries.

---

## Table of Contents

1. [Why Partition?](#1-why-partition)
2. [Partitioning by Key Range](#2-partitioning-by-key-range)
3. [Partitioning by Hash](#3-partitioning-by-hash)
4. [Consistent Hashing](#4-consistent-hashing)
5. [Handling Hot Spots](#5-handling-hot-spots)
6. [Secondary Indexes with Partitioning](#6-secondary-indexes-with-partitioning)
7. [Rebalancing Partitions](#7-rebalancing-partitions)
8. [Request Routing](#8-request-routing)
9. [Scatter-Gather Queries](#9-scatter-gather-queries)
10. [Partition-Aware Connection Pooling](#10-partition-aware-connection-pooling)
11. [Implementing a Sharded Cache](#11-implementing-a-sharded-cache)
12. [Real-World Example: Building a Sharded Key-Value Store](#12-real-world-example-building-a-sharded-key-value-store)
13. [Key Takeaways](#13-key-takeaways)
14. [Practice Exercises](#14-practice-exercises)

---

## 1. Why Partition?

### The Single-Node Ceiling

Every database starts on a single machine. That machine has finite CPU, RAM, disk I/O, and network bandwidth. When your data grows beyond what one machine can hold, or your query load exceeds what one machine can serve, you have two choices:

```
Vertical Scaling (Scale Up)              Horizontal Scaling (Scale Out)
───────────────────────────              ────────────────────────────────
Buy a bigger machine                     Add more machines

  ┌─────────────────────┐                ┌──────┐ ┌──────┐ ┌──────┐
  │                     │                │ Node │ │ Node │ │ Node │
  │   MEGA SERVER       │                │  A   │ │  B   │ │  C   │
  │   256 GB RAM        │                │ 64GB │ │ 64GB │ │ 64GB │
  │   128 cores         │                │ keys │ │ keys │ │ keys │
  │   All data here     │                │ a-h  │ │ i-q  │ │ r-z  │
  │                     │                └──────┘ └──────┘ └──────┘
  └─────────────────────┘

Pros: Simple                             Pros: No ceiling, cost-effective
Cons: Expensive ceiling, SPOF           Cons: Complexity (this chapter)
```

Vertical scaling has a hard ceiling: the largest machine money can buy. Horizontal scaling has no theoretical limit -- you can keep adding nodes. But splitting data across nodes introduces significant complexity in routing, rebalancing, querying, and consistency. That is what partitioning solves.

### Three Goals of Partitioning

1. **Spread data evenly** -- Each node should hold roughly the same amount of data (avoid skew).
2. **Spread load evenly** -- Each node should handle roughly the same number of queries (avoid hot spots).
3. **Minimize cross-partition queries** -- Queries that touch a single partition are fast. Queries that touch all partitions (scatter-gather) are expensive.

### Partitioning vs Replication

Partitioning and replication are orthogonal concepts that are almost always combined in practice:

```
                    Partition 1           Partition 2           Partition 3
                   (keys a-h)           (keys i-q)           (keys r-z)
                ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   Leader       │   Node A     │     │   Node B     │     │   Node C     │
                └──────────────┘     └──────────────┘     └──────────────┘
                ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   Follower 1   │   Node B     │     │   Node C     │     │   Node A     │
                └──────────────┘     └──────────────┘     └──────────────┘
                ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   Follower 2   │   Node C     │     │   Node A     │     │   Node B     │
                └──────────────┘     └──────────────┘     └──────────────┘
```

Each partition is replicated across multiple nodes for fault tolerance. Each node serves as leader for some partitions and follower for others. This chapter focuses on the partitioning dimension; the replication dimension was covered in the previous chapter.

### A Simple Demonstration: Why One Node Is Not Enough

```go
package main

import (
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// SingleNode represents a database that stores everything on one machine.
type SingleNode struct {
	data map[string]string
	mu   sync.RWMutex
	ops  atomic.Int64
}

func NewSingleNode() *SingleNode {
	return &SingleNode{data: make(map[string]string)}
}

func (n *SingleNode) Put(key, value string) {
	n.mu.Lock()
	n.data[key] = value
	n.mu.Unlock()
	n.ops.Add(1)
}

func (n *SingleNode) Get(key string) (string, bool) {
	n.mu.RLock()
	v, ok := n.data[key]
	n.mu.RUnlock()
	n.ops.Add(1)
	return v, ok
}

// PartitionedCluster splits data across multiple nodes.
type PartitionedCluster struct {
	nodes []*SingleNode
	count int
}

func NewPartitionedCluster(numNodes int) *PartitionedCluster {
	nodes := make([]*SingleNode, numNodes)
	for i := range nodes {
		nodes[i] = NewSingleNode()
	}
	return &PartitionedCluster{nodes: nodes, count: numNodes}
}

func (pc *PartitionedCluster) getNode(key string) *SingleNode {
	// Simple hash partitioning: sum of bytes mod node count
	var hash int
	for _, b := range []byte(key) {
		hash += int(b)
	}
	return pc.nodes[hash%pc.count]
}

func (pc *PartitionedCluster) Put(key, value string) {
	pc.getNode(key).Put(key, value)
}

func (pc *PartitionedCluster) Get(key string) (string, bool) {
	return pc.getNode(key).Get(key)
}

func main() {
	const numKeys = 100_000
	const numReaders = 100

	// Generate keys
	keys := make([]string, numKeys)
	for i := range keys {
		keys[i] = fmt.Sprintf("user:%d:profile", i)
	}

	// --- Single node benchmark ---
	single := NewSingleNode()
	for _, k := range keys {
		single.Put(k, strings.Repeat("x", 100))
	}

	start := time.Now()
	var wg sync.WaitGroup
	for r := 0; r < numReaders; r++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			rng := rand.New(rand.NewSource(time.Now().UnixNano()))
			for i := 0; i < 10_000; i++ {
				single.Get(keys[rng.Intn(numKeys)])
			}
		}()
	}
	wg.Wait()
	singleDuration := time.Since(start)

	// --- Partitioned cluster benchmark (4 nodes) ---
	cluster := NewPartitionedCluster(4)
	for _, k := range keys {
		cluster.Put(k, strings.Repeat("x", 100))
	}

	start = time.Now()
	for r := 0; r < numReaders; r++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			rng := rand.New(rand.NewSource(time.Now().UnixNano()))
			for i := 0; i < 10_000; i++ {
				cluster.Get(keys[rng.Intn(numKeys)])
			}
		}()
	}
	wg.Wait()
	clusterDuration := time.Since(start)

	fmt.Println("=== Single Node vs Partitioned Cluster ===")
	fmt.Printf("Single node:       %v (%d total ops)\n", singleDuration, single.ops.Load())
	fmt.Printf("Partitioned (4x):  %v\n", clusterDuration)
	fmt.Printf("Speedup:           %.2fx\n", float64(singleDuration)/float64(clusterDuration))

	// Show data distribution across partitions
	fmt.Println("\nData distribution across partitions:")
	for i, node := range cluster.nodes {
		node.mu.RLock()
		fmt.Printf("  Node %d: %d keys\n", i, len(node.data))
		node.mu.RUnlock()
	}
}
```

The partitioned cluster is faster because each node's mutex protects a smaller portion of the data, reducing lock contention. In a real distributed system, the benefit is even greater: each node has its own CPU, RAM, and disk.

---

## 2. Partitioning by Key Range

### The Concept

Key-range partitioning assigns a contiguous range of keys to each partition, much like volumes of an encyclopedia: A-D on one shelf, E-H on another, and so on. The keys are sorted within each partition, which makes range queries efficient.

```
Partition 0         Partition 1         Partition 2         Partition 3
[a --- e)           [e --- k)           [k --- r)           [r --- z]
┌──────────┐        ┌──────────┐        ┌──────────┐        ┌──────────┐
│ alice     │        │ eve      │        │ kate     │        │ roger    │
│ bob       │        │ frank    │        │ larry    │        │ sarah    │
│ charlie   │        │ grace    │        │ mike     │        │ tom      │
│ david     │        │ henry    │        │ nancy    │        │ ursula   │
│           │        │ iris     │        │ oscar    │        │ victor   │
│           │        │ john     │        │ pat      │        │ wendy    │
│           │        │          │        │ quinn    │        │ xavier   │
└──────────┘        └──────────┘        └──────────┘        └──────────┘
```

**Advantage**: Range queries (e.g., "all users from `kate` to `oscar`") are efficient because they hit one or two adjacent partitions.

**Disadvantage**: If access patterns are skewed (e.g., all new users have names starting with "S" today), one partition becomes a hot spot.

### Complete Implementation

```go
package main

import (
	"fmt"
	"sort"
	"strings"
	"sync"
)

// Node represents a storage node in the cluster.
type Node struct {
	ID   string
	data map[string]string
	mu   sync.RWMutex
}

func NewNode(id string) *Node {
	return &Node{
		ID:   id,
		data: make(map[string]string),
	}
}

func (n *Node) Put(key, value string) {
	n.mu.Lock()
	n.data[key] = value
	n.mu.Unlock()
}

func (n *Node) Get(key string) (string, bool) {
	n.mu.RLock()
	v, ok := n.data[key]
	n.mu.RUnlock()
	return v, ok
}

func (n *Node) Delete(key string) {
	n.mu.Lock()
	delete(n.data, key)
	n.mu.Unlock()
}

// RangeQuery returns all key-value pairs where lowerBound <= key < upperBound.
func (n *Node) RangeQuery(lowerBound, upperBound string) []KeyValue {
	n.mu.RLock()
	defer n.mu.RUnlock()

	var results []KeyValue
	for k, v := range n.data {
		if k >= lowerBound && k < upperBound {
			results = append(results, KeyValue{Key: k, Value: v})
		}
	}
	// Sort results for consistent ordering
	sort.Slice(results, func(i, j int) bool {
		return results[i].Key < results[j].Key
	})
	return results
}

func (n *Node) Size() int {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return len(n.data)
}

// KeyValue is a simple key-value pair.
type KeyValue struct {
	Key   string
	Value string
}

// RangePartitioner assigns keys to nodes based on sorted key boundaries.
// If boundaries are ["e", "k", "r"], then:
//   - keys < "e"   -> node 0
//   - "e" <= key < "k" -> node 1
//   - "k" <= key < "r" -> node 2
//   - key >= "r"   -> node 3
type RangePartitioner struct {
	boundaries []string // sorted partition boundaries
	nodes      []*Node
	mu         sync.RWMutex
}

// NewRangePartitioner creates a range partitioner.
// boundaries must be sorted. len(nodes) must equal len(boundaries) + 1.
func NewRangePartitioner(boundaries []string, nodes []*Node) *RangePartitioner {
	if len(nodes) != len(boundaries)+1 {
		panic("number of nodes must be len(boundaries) + 1")
	}
	// Verify boundaries are sorted
	for i := 1; i < len(boundaries); i++ {
		if boundaries[i] <= boundaries[i-1] {
			panic("boundaries must be strictly sorted")
		}
	}
	return &RangePartitioner{
		boundaries: boundaries,
		nodes:      nodes,
	}
}

// GetPartition returns the node responsible for the given key.
// Uses binary search for O(log n) lookup.
func (rp *RangePartitioner) GetPartition(key string) *Node {
	rp.mu.RLock()
	defer rp.mu.RUnlock()

	// Binary search: find the first boundary that is > key
	idx := sort.SearchStrings(rp.boundaries, key)

	// If key exactly matches a boundary, it belongs to the next partition.
	// sort.SearchStrings returns the index where key would be inserted,
	// which is the index of the first element >= key.
	// We want: keys < boundaries[0] -> partition 0
	//          boundaries[0] <= keys < boundaries[1] -> partition 1
	//          etc.
	return rp.nodes[idx]
}

// GetPartitionIndex returns the partition index for a key.
func (rp *RangePartitioner) GetPartitionIndex(key string) int {
	rp.mu.RLock()
	defer rp.mu.RUnlock()
	return sort.SearchStrings(rp.boundaries, key)
}

// RangeQuery executes a range query across relevant partitions.
func (rp *RangePartitioner) RangeQuery(lowerBound, upperBound string) []KeyValue {
	rp.mu.RLock()
	defer rp.mu.RUnlock()

	// Find which partitions overlap with [lowerBound, upperBound)
	startPartition := sort.SearchStrings(rp.boundaries, lowerBound)
	endPartition := sort.SearchStrings(rp.boundaries, upperBound)

	// Collect results from all relevant partitions
	var allResults []KeyValue
	for i := startPartition; i <= endPartition && i < len(rp.nodes); i++ {
		results := rp.nodes[i].RangeQuery(lowerBound, upperBound)
		allResults = append(allResults, results...)
	}

	// Final sort across all partitions
	sort.Slice(allResults, func(i, j int) bool {
		return allResults[i].Key < allResults[j].Key
	})
	return allResults
}

// PrintDistribution shows how data is distributed across partitions.
func (rp *RangePartitioner) PrintDistribution() {
	rp.mu.RLock()
	defer rp.mu.RUnlock()

	fmt.Println("Partition distribution:")
	for i, node := range rp.nodes {
		var rangeStr string
		switch {
		case i == 0:
			rangeStr = fmt.Sprintf("[min, %q)", rp.boundaries[0])
		case i == len(rp.nodes)-1:
			rangeStr = fmt.Sprintf("[%q, max]", rp.boundaries[i-1])
		default:
			rangeStr = fmt.Sprintf("[%q, %q)", rp.boundaries[i-1], rp.boundaries[i])
		}
		fmt.Printf("  Partition %d (%s): range %s, %d keys\n",
			i, node.ID, rangeStr, node.Size())
	}
}

func main() {
	// Create 4 nodes
	nodes := []*Node{
		NewNode("node-0"),
		NewNode("node-1"),
		NewNode("node-2"),
		NewNode("node-3"),
	}

	// Partition boundaries: keys < "e" -> node 0, "e"-"k" -> node 1, etc.
	rp := NewRangePartitioner([]string{"e", "k", "r"}, nodes)

	// Insert some data
	users := map[string]string{
		"alice":   "Alice Anderson",
		"bob":     "Bob Brown",
		"charlie": "Charlie Chen",
		"david":   "David Davis",
		"eve":     "Eve Edwards",
		"frank":   "Frank Foster",
		"grace":   "Grace Garcia",
		"henry":   "Henry Hill",
		"iris":    "Iris Ito",
		"john":    "John Jackson",
		"kate":    "Kate Kim",
		"larry":   "Larry Lee",
		"mike":    "Mike Miller",
		"nancy":   "Nancy Nguyen",
		"oscar":   "Oscar Ortiz",
		"pat":     "Pat Patel",
		"quinn":   "Quinn Quill",
		"roger":   "Roger Ross",
		"sarah":   "Sarah Smith",
		"tom":     "Tom Taylor",
		"ursula":  "Ursula Underwood",
		"victor":  "Victor Vasquez",
		"wendy":   "Wendy Wang",
		"xavier":  "Xavier Xu",
	}

	for key, value := range users {
		node := rp.GetPartition(key)
		node.Put(key, value)
		fmt.Printf("  %s -> %s\n", key, node.ID)
	}

	fmt.Println()
	rp.PrintDistribution()

	// Range query: find all users from "frank" to "mike"
	fmt.Println("\nRange query: frank <= key < mike")
	results := rp.RangeQuery("frank", "mike")
	for _, kv := range results {
		fmt.Printf("  %s = %s\n", kv.Key, kv.Value)
	}

	// Point query
	fmt.Println("\nPoint queries:")
	for _, key := range []string{"alice", "kate", "xavier"} {
		node := rp.GetPartition(key)
		value, found := node.Get(key)
		fmt.Printf("  Get(%q) -> node=%s, value=%q, found=%v\n",
			key, node.ID, value, found)
	}

	// Demonstrate the hot spot problem
	fmt.Println("\n--- Hot Spot Demonstration ---")
	fmt.Println("If many keys start with 's' (same partition):")

	hotNodes := []*Node{NewNode("hot-0"), NewNode("hot-1"), NewNode("hot-2"), NewNode("hot-3")}
	hotRP := NewRangePartitioner([]string{"e", "k", "r"}, hotNodes)

	// Simulate a workload skewed toward "s" keys
	for i := 0; i < 1000; i++ {
		key := fmt.Sprintf("s_user_%04d", i)
		hotRP.GetPartition(key).Put(key, "data")
	}
	for i := 0; i < 100; i++ {
		key := fmt.Sprintf("a_user_%04d", i)
		hotRP.GetPartition(key).Put(key, "data")
	}
	for i := 0; i < 100; i++ {
		key := fmt.Sprintf("m_user_%04d", i)
		hotRP.GetPartition(key).Put(key, "data")
	}

	hotRP.PrintDistribution()
	_ = strings.NewReader("") // satisfy import
}
```

### When to Use Key-Range Partitioning

| Use Case | Why It Works |
|----------|-------------|
| Time-series data (partition by date) | Range queries on time windows are common |
| Lexicographic keys (user names, URLs) | Sorted order enables prefix scans |
| Sensor data (partition by sensor ID range) | Each sensor's data is colocated |

| Anti-Pattern | Why It Fails |
|-------------|-------------|
| Monotonically increasing keys (timestamps) | All writes go to the last partition |
| Heavily skewed key distributions | One partition becomes much larger |

---

## 3. Partitioning by Hash

### The Concept

Hash partitioning applies a hash function to the key and uses the hash value to determine the partition. This eliminates the hot-spot problem caused by skewed key distributions because a good hash function distributes keys uniformly, regardless of their lexicographic order.

```
Key "alice" ──hash──► 2847261  mod 4 = 1  ──► Partition 1
Key "bob"   ──hash──► 9183742  mod 4 = 2  ──► Partition 2
Key "carol" ──hash──► 1029384  mod 4 = 0  ──► Partition 0
Key "sarah" ──hash──► 7261940  mod 4 = 0  ──► Partition 0
Key "steve" ──hash──► 4817263  mod 4 = 3  ──► Partition 3

Notice: "sarah" and "steve" (both start with 's') land on DIFFERENT partitions.
This is exactly what we want -- the hash function breaks the lexicographic clustering.
```

**Advantage**: Uniform distribution of keys across partitions, regardless of key patterns.

**Disadvantage**: You lose the ability to do efficient range queries. A range query must be sent to all partitions (scatter-gather).

### Choosing a Hash Function

Not all hash functions are suitable for partitioning. You need one that produces uniform distribution. Cryptographic hash functions (SHA-256) work but are overkill. Go's `hash/fnv` package provides fast, well-distributed hash functions.

**Important**: Go's built-in `map` uses a randomized hash function that changes between program runs. Never use it for partitioning -- you need deterministic hashing.

### Complete Implementation

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
	"sort"
	"sync"
)

// Node represents a storage node.
type Node struct {
	ID   string
	data map[string]string
	mu   sync.RWMutex
}

func NewNode(id string) *Node {
	return &Node{ID: id, data: make(map[string]string)}
}

func (n *Node) Put(key, value string) {
	n.mu.Lock()
	n.data[key] = value
	n.mu.Unlock()
}

func (n *Node) Get(key string) (string, bool) {
	n.mu.RLock()
	v, ok := n.data[key]
	n.mu.RUnlock()
	return v, ok
}

func (n *Node) Size() int {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return len(n.data)
}

func (n *Node) AllKeys() []string {
	n.mu.RLock()
	defer n.mu.RUnlock()
	keys := make([]string, 0, len(n.data))
	for k := range n.data {
		keys = append(keys, k)
	}
	sort.Strings(keys)
	return keys
}

// HashPartitioner distributes keys across nodes using a hash function.
type HashPartitioner struct {
	numPartitions int
	nodes         []*Node
}

// NewHashPartitioner creates a new hash partitioner.
func NewHashPartitioner(nodes []*Node) *HashPartitioner {
	return &HashPartitioner{
		numPartitions: len(nodes),
		nodes:         nodes,
	}
}

// hashKey computes a deterministic hash for a key using FNV-1a.
func hashKey(key string) uint32 {
	h := fnv.New32a()
	h.Write([]byte(key))
	return h.Sum32()
}

// GetPartition returns the node responsible for the given key.
func (hp *HashPartitioner) GetPartition(key string) *Node {
	partition := int(hashKey(key)) % hp.numPartitions
	return hp.nodes[partition]
}

// GetPartitionIndex returns the partition index for a key.
func (hp *HashPartitioner) GetPartitionIndex(key string) int {
	return int(hashKey(key)) % hp.numPartitions
}

// Put stores a key-value pair on the appropriate node.
func (hp *HashPartitioner) Put(key, value string) {
	hp.GetPartition(key).Put(key, value)
}

// Get retrieves a value from the appropriate node.
func (hp *HashPartitioner) Get(key string) (string, bool) {
	return hp.GetPartition(key).Get(key)
}

// PrintDistribution shows how data is distributed.
func (hp *HashPartitioner) PrintDistribution() {
	fmt.Println("Hash partition distribution:")
	totalKeys := 0
	sizes := make([]int, len(hp.nodes))
	for i, node := range hp.nodes {
		sizes[i] = node.Size()
		totalKeys += sizes[i]
	}

	if totalKeys == 0 {
		fmt.Println("  (no data)")
		return
	}

	idealPerNode := float64(totalKeys) / float64(len(hp.nodes))
	for i, node := range hp.nodes {
		bar := ""
		barLen := int(float64(sizes[i]) / float64(totalKeys) * 50)
		for j := 0; j < barLen; j++ {
			bar += "#"
		}
		skew := float64(sizes[i])/idealPerNode - 1.0
		fmt.Printf("  Node %d (%s): %5d keys [%-50s] skew: %+.1f%%\n",
			i, node.ID, sizes[i], bar, skew*100)
	}

	// Calculate standard deviation of distribution
	variance := 0.0
	for _, s := range sizes {
		diff := float64(s) - idealPerNode
		variance += diff * diff
	}
	stddev := math.Sqrt(variance / float64(len(sizes)))
	fmt.Printf("  Ideal per node: %.0f, Std dev: %.1f (%.1f%%)\n",
		idealPerNode, stddev, stddev/idealPerNode*100)
}

func main() {
	// Create 4 nodes
	nodes := make([]*Node, 4)
	for i := range nodes {
		nodes[i] = NewNode(fmt.Sprintf("node-%d", i))
	}

	hp := NewHashPartitioner(nodes)

	// --- Test 1: Uniform key distribution ---
	fmt.Println("=== Test 1: Keys with uniform distribution ===")
	for i := 0; i < 10000; i++ {
		key := fmt.Sprintf("user:%d", i)
		hp.Put(key, fmt.Sprintf("value-%d", i))
	}
	hp.PrintDistribution()

	// --- Test 2: Keys that would be skewed under range partitioning ---
	fmt.Println("\n=== Test 2: Keys with common prefix (would be skewed in range) ===")
	nodes2 := make([]*Node, 4)
	for i := range nodes2 {
		nodes2[i] = NewNode(fmt.Sprintf("node-%d", i))
	}
	hp2 := NewHashPartitioner(nodes2)

	// All keys start with "s" -- this would be terrible for range partitioning
	for i := 0; i < 10000; i++ {
		key := fmt.Sprintf("s_user_%06d", i)
		hp2.Put(key, "data")
	}
	hp2.PrintDistribution()

	// --- Test 3: Point queries work correctly ---
	fmt.Println("\n=== Test 3: Point queries ===")
	for _, key := range []string{"user:0", "user:42", "user:999", "user:9999"} {
		value, found := hp.Get(key)
		nodeIdx := hp.GetPartitionIndex(key)
		fmt.Printf("  Get(%q) -> partition=%d, value=%q, found=%v\n",
			key, nodeIdx, value, found)
	}

	// --- Test 4: The problem with hash partitioning -- range queries ---
	fmt.Println("\n=== Test 4: Range queries require scatter-gather ===")
	fmt.Println("To find all users with IDs 100-199, we must check ALL partitions:")
	for i, node := range nodes {
		count := 0
		for _, k := range node.AllKeys() {
			// Manually check if key matches our range
			var id int
			if _, err := fmt.Sscanf(k, "user:%d", &id); err == nil {
				if id >= 100 && id < 200 {
					count++
				}
			}
		}
		fmt.Printf("  Node %d: found %d matching keys\n", i, count)
	}
	fmt.Println("  (Keys are scattered across all partitions -- no way to skip partitions)")

	// --- Test 5: The redistribution problem ---
	fmt.Println("\n=== Test 5: Adding a node breaks everything ===")
	fmt.Println("With 4 nodes: hash(key) mod 4")
	fmt.Println("With 5 nodes: hash(key) mod 5")
	fmt.Println("Most keys map to a DIFFERENT partition!")

	movedCount := 0
	for i := 0; i < 10000; i++ {
		key := fmt.Sprintf("user:%d", i)
		h := hashKey(key)
		oldPartition := int(h) % 4
		newPartition := int(h) % 5
		if oldPartition != newPartition {
			movedCount++
		}
	}
	fmt.Printf("  Keys that would need to move: %d out of 10000 (%.1f%%)\n",
		movedCount, float64(movedCount)/10000*100)
	fmt.Println("  This is why simple mod-N hashing is problematic for dynamic clusters.")
	fmt.Println("  Solution: Consistent hashing (next section).")
}
```

### Hash Functions Compared

| Hash Function | Speed | Distribution | Deterministic | Use Case |
|--------------|-------|-------------|---------------|----------|
| `fnv.New32a()` | Very fast | Good | Yes | General partitioning |
| `crc32.ChecksumIEEE()` | Fast | Decent | Yes | Checksums, simple partitioning |
| `sha256.Sum256()` | Slow | Excellent | Yes | Security-sensitive partitioning |
| `xxhash` (external) | Fastest | Excellent | Yes | High-throughput partitioning |
| `maphash.Hash` | Fast | Good | **No** (seeded) | NOT for partitioning |

---

## 4. Consistent Hashing

### The Problem with Mod-N Hashing

As we saw in the previous section, simple `hash(key) % N` hashing has a catastrophic flaw: when `N` changes (you add or remove a node), almost every key maps to a different partition. In a system with millions of keys, this means moving almost all data to different nodes -- an extremely expensive operation.

### How Consistent Hashing Works

Consistent hashing maps both keys and nodes onto the same hash ring (a circle of hash values from 0 to 2^32 - 1). To find which node owns a key, you walk clockwise around the ring from the key's position until you hit a node.

```
                        0 / 2^32
                         │
                    ╭────┼────╮
                 ╱  Node A     ╲
               ╱   (hash=100)    ╲
              │                     │
              │     key "alice"     │
              │     (hash=280)      │
  2^32 * 3/4  │       ↓            │  2^32 * 1/4
              │  walks clockwise   │
              │  to → Node B       │
              │     (hash=400)     │
               ╲                  ╱
                 ╲  Node C      ╱
                    ╰────┼────╯
                     (hash=700)
                     2^32 / 2

Key "alice" (hash=280) → walks clockwise → Node B (hash=400)

When Node B is removed:
Key "alice" (hash=280) → walks clockwise → Node C (hash=700)
Only keys between Node A and Node B need to move!
```

### Virtual Nodes (Vnodes)

With only physical nodes on the ring, the distribution can be very uneven. If two nodes happen to hash close together, one gets a tiny arc and the other gets a huge arc. Virtual nodes solve this: each physical node gets many positions on the ring.

```
Physical: 3 nodes (A, B, C)
Virtual:  3 vnodes each = 9 positions on ring

Ring: A0 -- B0 -- C0 -- A1 -- B1 -- C1 -- A2 -- B2 -- C2

With more vnodes, the arcs become more uniform.
150 vnodes per node is a common production setting.
```

### Complete Implementation

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
	"sort"
	"sync"
)

// ConsistentHash implements a consistent hashing ring with virtual nodes.
type ConsistentHash struct {
	ring     map[uint32]string // hash position -> physical node ID
	sorted   []uint32          // sorted hash positions for binary search
	replicas int               // number of virtual nodes per physical node
	nodes    map[string]bool   // set of physical node IDs
	mu       sync.RWMutex
}

// NewConsistentHash creates a new consistent hash ring.
// replicas is the number of virtual nodes per physical node.
// Higher values give better distribution but use more memory.
// Typical production value: 100-200.
func NewConsistentHash(replicas int) *ConsistentHash {
	return &ConsistentHash{
		ring:     make(map[uint32]string),
		sorted:   []uint32{},
		replicas: replicas,
		nodes:    make(map[string]bool),
	}
}

// hashPosition computes the hash for a virtual node position.
func hashPosition(nodeID string, replica int) uint32 {
	h := fnv.New32a()
	h.Write([]byte(fmt.Sprintf("%s#%d", nodeID, replica)))
	return h.Sum32()
}

// AddNode adds a physical node to the ring with `replicas` virtual nodes.
func (ch *ConsistentHash) AddNode(nodeID string) {
	ch.mu.Lock()
	defer ch.mu.Unlock()

	if ch.nodes[nodeID] {
		return // already exists
	}
	ch.nodes[nodeID] = true

	for i := 0; i < ch.replicas; i++ {
		pos := hashPosition(nodeID, i)
		ch.ring[pos] = nodeID
		ch.sorted = append(ch.sorted, pos)
	}

	// Re-sort the ring positions
	sort.Slice(ch.sorted, func(i, j int) bool {
		return ch.sorted[i] < ch.sorted[j]
	})
}

// RemoveNode removes a physical node and all its virtual nodes from the ring.
func (ch *ConsistentHash) RemoveNode(nodeID string) {
	ch.mu.Lock()
	defer ch.mu.Unlock()

	if !ch.nodes[nodeID] {
		return // doesn't exist
	}
	delete(ch.nodes, nodeID)

	// Remove all virtual nodes for this physical node
	for i := 0; i < ch.replicas; i++ {
		pos := hashPosition(nodeID, i)
		delete(ch.ring, pos)
	}

	// Rebuild sorted slice
	ch.sorted = ch.sorted[:0]
	for pos := range ch.ring {
		ch.sorted = append(ch.sorted, pos)
	}
	sort.Slice(ch.sorted, func(i, j int) bool {
		return ch.sorted[i] < ch.sorted[j]
	})
}

// GetNode returns the physical node responsible for the given key.
// It hashes the key and walks clockwise around the ring.
func (ch *ConsistentHash) GetNode(key string) string {
	ch.mu.RLock()
	defer ch.mu.RUnlock()

	if len(ch.sorted) == 0 {
		return ""
	}

	h := fnv.New32a()
	h.Write([]byte(key))
	keyHash := h.Sum32()

	// Binary search for the first ring position >= keyHash
	idx := sort.Search(len(ch.sorted), func(i int) bool {
		return ch.sorted[i] >= keyHash
	})

	// Wrap around if we've gone past the end of the ring
	if idx >= len(ch.sorted) {
		idx = 0
	}

	return ch.ring[ch.sorted[idx]]
}

// GetNodes returns the top N distinct physical nodes for a key.
// Useful for replication: store the key on N different nodes.
func (ch *ConsistentHash) GetNodes(key string, count int) []string {
	ch.mu.RLock()
	defer ch.mu.RUnlock()

	if len(ch.sorted) == 0 {
		return nil
	}

	h := fnv.New32a()
	h.Write([]byte(key))
	keyHash := h.Sum32()

	idx := sort.Search(len(ch.sorted), func(i int) bool {
		return ch.sorted[i] >= keyHash
	})
	if idx >= len(ch.sorted) {
		idx = 0
	}

	seen := make(map[string]bool)
	var result []string

	for len(result) < count && len(seen) < len(ch.nodes) {
		nodeID := ch.ring[ch.sorted[idx]]
		if !seen[nodeID] {
			seen[nodeID] = true
			result = append(result, nodeID)
		}
		idx = (idx + 1) % len(ch.sorted)
	}

	return result
}

// NodeCount returns the number of physical nodes on the ring.
func (ch *ConsistentHash) NodeCount() int {
	ch.mu.RLock()
	defer ch.mu.RUnlock()
	return len(ch.nodes)
}

// ListNodes returns all physical node IDs.
func (ch *ConsistentHash) ListNodes() []string {
	ch.mu.RLock()
	defer ch.mu.RUnlock()
	result := make([]string, 0, len(ch.nodes))
	for id := range ch.nodes {
		result = append(result, id)
	}
	sort.Strings(result)
	return result
}

func main() {
	// --- Demonstration 1: Basic consistent hashing ---
	fmt.Println("=== Basic Consistent Hashing ===")
	ch := NewConsistentHash(150) // 150 virtual nodes per physical node

	ch.AddNode("node-A")
	ch.AddNode("node-B")
	ch.AddNode("node-C")

	// Map some keys
	keys := []string{
		"user:1", "user:2", "user:3", "user:4", "user:5",
		"order:100", "order:200", "order:300",
		"product:a", "product:b", "product:c",
	}

	fmt.Println("Key assignments:")
	for _, key := range keys {
		fmt.Printf("  %s -> %s\n", key, ch.GetNode(key))
	}

	// --- Demonstration 2: Adding a node ---
	fmt.Println("\n=== Adding node-D ===")
	ch.AddNode("node-D")

	movedCount := 0
	fmt.Println("Key assignments after adding node-D:")
	for _, key := range keys {
		newNode := ch.GetNode(key)
		fmt.Printf("  %s -> %s\n", key, newNode)
	}

	// Check what fraction of 10,000 keys would move
	// First record old assignments
	ch2 := NewConsistentHash(150)
	ch2.AddNode("node-A")
	ch2.AddNode("node-B")
	ch2.AddNode("node-C")

	ch3 := NewConsistentHash(150)
	ch3.AddNode("node-A")
	ch3.AddNode("node-B")
	ch3.AddNode("node-C")
	ch3.AddNode("node-D")

	movedCount = 0
	for i := 0; i < 10000; i++ {
		key := fmt.Sprintf("key:%d", i)
		if ch2.GetNode(key) != ch3.GetNode(key) {
			movedCount++
		}
	}
	fmt.Printf("\nKeys moved when adding 1 node to 3-node cluster: %d/10000 (%.1f%%)\n",
		movedCount, float64(movedCount)/100.0)
	fmt.Printf("Ideal (1/4 of keys): 25.0%%\n")

	// --- Demonstration 3: Removing a node ---
	fmt.Println("\n=== Removing node-B ===")
	ch4 := NewConsistentHash(150)
	ch4.AddNode("node-A")
	ch4.AddNode("node-B")
	ch4.AddNode("node-C")

	ch5 := NewConsistentHash(150)
	ch5.AddNode("node-A")
	ch5.AddNode("node-C")

	movedCount = 0
	for i := 0; i < 10000; i++ {
		key := fmt.Sprintf("key:%d", i)
		if ch4.GetNode(key) != ch5.GetNode(key) {
			movedCount++
		}
	}
	fmt.Printf("Keys moved when removing 1 node from 3-node cluster: %d/10000 (%.1f%%)\n",
		movedCount, float64(movedCount)/100.0)
	fmt.Printf("Ideal (1/3 of keys): 33.3%%\n")

	// --- Demonstration 4: Distribution quality vs vnodes ---
	fmt.Println("\n=== Distribution Quality vs Virtual Nodes ===")
	for _, vnodes := range []int{1, 10, 50, 150, 500} {
		ring := NewConsistentHash(vnodes)
		ring.AddNode("A")
		ring.AddNode("B")
		ring.AddNode("C")
		ring.AddNode("D")

		counts := make(map[string]int)
		for i := 0; i < 100000; i++ {
			key := fmt.Sprintf("k:%d", i)
			counts[ring.GetNode(key)]++
		}

		// Calculate standard deviation
		mean := 100000.0 / 4.0
		variance := 0.0
		for _, c := range counts {
			diff := float64(c) - mean
			variance += diff * diff
		}
		stddev := math.Sqrt(variance / 4.0)

		fmt.Printf("  vnodes=%3d: stddev=%.0f (%.1f%% of ideal)\n",
			vnodes, stddev, stddev/mean*100)
	}

	// --- Demonstration 5: Replication with consistent hashing ---
	fmt.Println("\n=== Replication: Getting N nodes for a key ===")
	chRep := NewConsistentHash(150)
	chRep.AddNode("us-east-1a")
	chRep.AddNode("us-east-1b")
	chRep.AddNode("us-west-2a")
	chRep.AddNode("eu-west-1a")

	for _, key := range []string{"user:alice", "user:bob", "order:12345"} {
		nodes := chRep.GetNodes(key, 3) // replicate to 3 nodes
		fmt.Printf("  %s -> primary=%s, replicas=%v\n", key, nodes[0], nodes[1:])
	}
}
```

### Consistent Hashing: Key Properties

| Property | Description |
|----------|-------------|
| **Minimal disruption** | Adding/removing a node moves only ~1/N of keys |
| **Uniform distribution** | With enough virtual nodes, keys are spread evenly |
| **Deterministic** | Same key always maps to the same node (given same ring state) |
| **Incremental scaling** | Can add nodes one at a time without full redistribution |

### Real-World Usage

- **Amazon DynamoDB**: Uses consistent hashing for partition assignment
- **Apache Cassandra**: Consistent hashing with virtual nodes (vnodes)
- **Memcached clients**: Use consistent hashing to distribute cache keys
- **Akamai CDN**: Original paper on consistent hashing was from Akamai researchers

---

## 5. Handling Hot Spots

### The Celebrity Problem

Even with perfect hash distribution, hot spots can occur at the application level. If one key receives disproportionately more reads or writes (e.g., a celebrity's profile on a social network), the node owning that key becomes a bottleneck.

```
Normal distribution:           Celebrity problem:
┌──────┐ ┌──────┐            ┌──────┐ ┌──────┐
│ 25%  │ │ 25%  │            │  5%  │ │  5%  │
│ load │ │ load │            │ load │ │ load │
└──────┘ └──────┘            └──────┘ └──────┘
┌──────┐ ┌──────┐            ┌──────┐ ┌──────┐
│ 25%  │ │ 25%  │            │ 85%  │ │  5%  │
│ load │ │ load │            │ load │ │ load │
└──────┘ └──────┘            └──────┘ └──────┘
  Balanced                    HOT SPOT on one node
```

### Strategies for Handling Hot Spots

1. **Key Salting**: Append a random suffix to hot keys, spreading them across partitions.
2. **Read Replicas**: For read-heavy keys, replicate to multiple nodes.
3. **Local Caching**: Cache hot keys in memory on the routing layer.
4. **Split Hot Partitions**: Detect hot partitions and split them.

### Complete Implementation

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math/rand"
	"sort"
	"sync"
	"sync/atomic"
	"time"
)

// PartitionStats tracks access statistics for a partition.
type PartitionStats struct {
	reads  atomic.Int64
	writes atomic.Int64
}

// HotSpotAwarePartitioner handles hot keys by salting them across partitions.
type HotSpotAwarePartitioner struct {
	numPartitions int
	hotKeys       map[string]int // key -> number of salt variants
	stats         map[int]*PartitionStats
	mu            sync.RWMutex
}

func NewHotSpotAwarePartitioner(numPartitions int) *HotSpotAwarePartitioner {
	stats := make(map[int]*PartitionStats, numPartitions)
	for i := 0; i < numPartitions; i++ {
		stats[i] = &PartitionStats{}
	}
	return &HotSpotAwarePartitioner{
		numPartitions: numPartitions,
		hotKeys:       make(map[string]int),
		stats:         stats,
	}
}

func hashStr(key string) uint32 {
	h := fnv.New32a()
	h.Write([]byte(key))
	return h.Sum32()
}

// MarkHotKey marks a key as "hot" and specifies how many salt variants to use.
// Reads for this key will be distributed across `variants` partitions.
func (p *HotSpotAwarePartitioner) MarkHotKey(key string, variants int) {
	p.mu.Lock()
	p.hotKeys[key] = variants
	p.mu.Unlock()
}

// GetWritePartitions returns ALL partition indices where a key should be written.
// For hot keys, this returns multiple partitions (the salted variants).
// For normal keys, this returns a single partition.
func (p *HotSpotAwarePartitioner) GetWritePartitions(key string) []int {
	p.mu.RLock()
	variants, isHot := p.hotKeys[key]
	p.mu.RUnlock()

	if !isHot {
		partition := int(hashStr(key)) % p.numPartitions
		p.stats[partition].writes.Add(1)
		return []int{partition}
	}

	// Hot key: write to all salted partitions
	partitions := make([]int, 0, variants)
	seen := make(map[int]bool)
	for i := 0; i < variants; i++ {
		saltedKey := fmt.Sprintf("%s#salt_%d", key, i)
		partition := int(hashStr(saltedKey)) % p.numPartitions
		if !seen[partition] {
			seen[partition] = true
			partitions = append(partitions, partition)
			p.stats[partition].writes.Add(1)
		}
	}
	return partitions
}

// GetReadPartition returns a SINGLE partition to read a key from.
// For hot keys, picks a random salted variant to spread read load.
// For normal keys, returns the deterministic partition.
func (p *HotSpotAwarePartitioner) GetReadPartition(key string) int {
	p.mu.RLock()
	variants, isHot := p.hotKeys[key]
	p.mu.RUnlock()

	if !isHot {
		partition := int(hashStr(key)) % p.numPartitions
		p.stats[partition].reads.Add(1)
		return partition
	}

	// Hot key: pick a random salt variant
	salt := rand.Intn(variants)
	saltedKey := fmt.Sprintf("%s#salt_%d", key, salt)
	partition := int(hashStr(saltedKey)) % p.numPartitions
	p.stats[partition].reads.Add(1)
	return partition
}

// PrintStats shows access distribution across partitions.
func (p *HotSpotAwarePartitioner) PrintStats() {
	fmt.Println("Partition access statistics:")
	for i := 0; i < p.numPartitions; i++ {
		reads := p.stats[i].reads.Load()
		writes := p.stats[i].writes.Load()
		readBar := ""
		for j := 0; j < int(reads/100); j++ {
			readBar += "#"
		}
		fmt.Printf("  Partition %d: reads=%5d writes=%5d  [%s]\n",
			i, reads, writes, readBar)
	}
}

// HotSpotDetector monitors partition load and detects hot spots.
type HotSpotDetector struct {
	windowSize    time.Duration
	threshold     float64 // multiplier over average that triggers alert
	partitionOps  map[int]*atomic.Int64
	numPartitions int
	mu            sync.RWMutex
}

func NewHotSpotDetector(numPartitions int, threshold float64) *HotSpotDetector {
	ops := make(map[int]*atomic.Int64, numPartitions)
	for i := 0; i < numPartitions; i++ {
		ops[i] = &atomic.Int64{}
	}
	return &HotSpotDetector{
		windowSize:    time.Second,
		threshold:     threshold,
		partitionOps:  ops,
		numPartitions: numPartitions,
	}
}

func (d *HotSpotDetector) RecordOp(partition int) {
	if counter, ok := d.partitionOps[partition]; ok {
		counter.Add(1)
	}
}

func (d *HotSpotDetector) DetectHotSpots() []int {
	var total int64
	counts := make([]int64, d.numPartitions)
	for i := 0; i < d.numPartitions; i++ {
		counts[i] = d.partitionOps[i].Load()
		total += counts[i]
	}

	if total == 0 {
		return nil
	}

	avg := float64(total) / float64(d.numPartitions)
	var hotSpots []int
	for i, c := range counts {
		if float64(c) > avg*d.threshold {
			hotSpots = append(hotSpots, i)
		}
	}
	return hotSpots
}

func (d *HotSpotDetector) Reset() {
	for i := 0; i < d.numPartitions; i++ {
		d.partitionOps[i].Store(0)
	}
}

// CachingLayer sits in front of the partitioner and caches hot keys.
type CachingLayer struct {
	cache     map[string]cachedEntry
	maxSize   int
	mu        sync.RWMutex
	hits      atomic.Int64
	misses    atomic.Int64
}

type cachedEntry struct {
	value     string
	expiresAt time.Time
}

func NewCachingLayer(maxSize int) *CachingLayer {
	return &CachingLayer{
		cache:   make(map[string]cachedEntry),
		maxSize: maxSize,
	}
}

func (c *CachingLayer) Get(key string) (string, bool) {
	c.mu.RLock()
	entry, ok := c.cache[key]
	c.mu.RUnlock()

	if !ok {
		c.misses.Add(1)
		return "", false
	}
	if time.Now().After(entry.expiresAt) {
		c.misses.Add(1)
		c.mu.Lock()
		delete(c.cache, key)
		c.mu.Unlock()
		return "", false
	}
	c.hits.Add(1)
	return entry.value, true
}

func (c *CachingLayer) Put(key, value string, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()

	// Simple eviction: if cache is full, remove a random entry
	if len(c.cache) >= c.maxSize {
		for k := range c.cache {
			delete(c.cache, k)
			break
		}
	}

	c.cache[key] = cachedEntry{
		value:     value,
		expiresAt: time.Now().Add(ttl),
	}
}

func (c *CachingLayer) Stats() (hits, misses int64) {
	return c.hits.Load(), c.misses.Load()
}

func main() {
	const numPartitions = 8

	// --- Scenario 1: Without hot spot handling ---
	fmt.Println("=== Scenario 1: No hot spot handling ===")
	fmt.Println("Simulating a celebrity user with 90% of all reads")

	naive := NewHotSpotAwarePartitioner(numPartitions)

	// Simulate workload: 90% of reads go to "celebrity:1"
	for i := 0; i < 10000; i++ {
		if rand.Float64() < 0.9 {
			naive.GetReadPartition("celebrity:1")
		} else {
			naive.GetReadPartition(fmt.Sprintf("user:%d", rand.Intn(10000)))
		}
	}
	naive.PrintStats()

	// --- Scenario 2: With hot spot handling (key salting) ---
	fmt.Println("\n=== Scenario 2: With key salting for hot key ===")

	salted := NewHotSpotAwarePartitioner(numPartitions)
	salted.MarkHotKey("celebrity:1", numPartitions) // spread across all partitions

	for i := 0; i < 10000; i++ {
		if rand.Float64() < 0.9 {
			salted.GetReadPartition("celebrity:1")
		} else {
			salted.GetReadPartition(fmt.Sprintf("user:%d", rand.Intn(10000)))
		}
	}
	salted.PrintStats()

	// --- Scenario 3: Automatic hot spot detection ---
	fmt.Println("\n=== Scenario 3: Automatic hot spot detection ===")
	detector := NewHotSpotDetector(numPartitions, 2.0) // trigger if 2x average

	// Simulate uneven load
	for i := 0; i < 5000; i++ {
		detector.RecordOp(3) // partition 3 is hot
	}
	for i := 0; i < numPartitions; i++ {
		if i != 3 {
			for j := 0; j < 500; j++ {
				detector.RecordOp(i)
			}
		}
	}

	hotSpots := detector.DetectHotSpots()
	fmt.Printf("Detected hot partitions: %v\n", hotSpots)

	// --- Scenario 4: Caching layer for hot keys ---
	fmt.Println("\n=== Scenario 4: Caching layer ===")
	cache := NewCachingLayer(1000)

	// Simulate reads with caching
	cache.Put("celebrity:1", `{"name":"Taylor","followers":100000000}`, 5*time.Second)

	for i := 0; i < 10000; i++ {
		key := "celebrity:1"
		if rand.Float64() > 0.9 {
			key = fmt.Sprintf("user:%d", rand.Intn(10000))
		}
		if _, ok := cache.Get(key); ok {
			// cache hit -- no need to go to partition
		} else {
			// cache miss -- would go to partition
		}
	}

	hits, misses := cache.Stats()
	fmt.Printf("Cache hits: %d, misses: %d, hit rate: %.1f%%\n",
		hits, misses, float64(hits)/float64(hits+misses)*100)

	// --- Summary of strategies ---
	fmt.Println("\n=== Hot Spot Mitigation Strategies ===")
	strategies := []struct {
		name    string
		tradeoff string
	}{
		{"Key Salting", "Write amplification (must write to N partitions)"},
		{"Read Replicas", "Stale reads possible, more storage"},
		{"Caching Layer", "Memory overhead, cache invalidation complexity"},
		{"Partition Splitting", "More partitions to manage, rebalancing cost"},
		{"Application-Level Routing", "Application complexity, requires hot key knowledge"},
	}

	for _, s := range strategies {
		fmt.Printf("  %-30s Tradeoff: %s\n", s.name, s.tradeoff)
	}
	_ = sort.Strings // satisfy import if needed
}
```

---

## 6. Secondary Indexes with Partitioning

### The Challenge

Partitioning by primary key is straightforward: each key maps to exactly one partition. But what about secondary indexes? If you partition users by `user_id` but need to query by `email` or `city`, you have a problem: the secondary index entries for a given value (e.g., `city=NYC`) are scattered across all partitions.

Two approaches exist:

```
Approach 1: Local Index (Document-Partitioned)
Each partition maintains its own secondary index covering ONLY its local data.
Reads require scatter-gather across ALL partitions.

Partition 0                 Partition 1                 Partition 2
┌────────────────┐         ┌────────────────┐         ┌────────────────┐
│ Data:          │         │ Data:          │         │ Data:          │
│  user:1 NYC    │         │  user:4 NYC    │         │  user:7 LA     │
│  user:2 LA     │         │  user:5 LA     │         │  user:8 NYC    │
│  user:3 NYC    │         │  user:6 CHI    │         │  user:9 CHI    │
│                │         │                │         │                │
│ Local Index:   │         │ Local Index:   │         │ Local Index:   │
│  NYC → [1,3]   │         │  NYC → [4]     │         │  NYC → [8]     │
│  LA  → [2]     │         │  LA  → [5]     │         │  LA  → [7]     │
│                │         │  CHI → [6]     │         │  CHI → [9]     │
└────────────────┘         └────────────────┘         └────────────────┘

Query "city=NYC" → must ask ALL 3 partitions → gather results → merge


Approach 2: Global Index (Term-Partitioned)
The secondary index itself is partitioned (e.g., cities A-L on partition 0, M-Z on partition 1).
Reads hit only one index partition. Writes must update the remote index partition.

Index Partition 0 (A-L)    Index Partition 1 (M-Z)
┌────────────────┐         ┌────────────────┐
│ CHI → [6, 9]   │         │ NYC → [1,3,4,8]│
│ LA  → [2, 5, 7]│         │                │
└────────────────┘         └────────────────┘

Query "city=NYC" → only Index Partition 1 → direct result
But write user:10 in NYC → must update Index Partition 1 (possibly remote)
```

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"sort"
	"strings"
	"sync"
	"time"
)

// User represents a document stored in our partitioned database.
type User struct {
	ID    string
	Name  string
	City  string
	Email string
	Age   int
}

// Partition stores users and maintains local secondary indexes.
type Partition struct {
	id       int
	users    map[string]*User
	// Local secondary indexes
	cityIndex  map[string][]string // city -> list of user IDs
	emailIndex map[string]string   // email -> user ID (unique index)
	mu         sync.RWMutex
}

func NewPartition(id int) *Partition {
	return &Partition{
		id:         id,
		users:      make(map[string]*User),
		cityIndex:  make(map[string][]string),
		emailIndex: make(map[string]string),
	}
}

func (p *Partition) Insert(user *User) {
	p.mu.Lock()
	defer p.mu.Unlock()

	p.users[user.ID] = user

	// Update local secondary indexes
	p.cityIndex[user.City] = append(p.cityIndex[user.City], user.ID)
	p.emailIndex[user.Email] = user.ID
}

func (p *Partition) GetByID(id string) (*User, bool) {
	p.mu.RLock()
	defer p.mu.RUnlock()
	user, ok := p.users[id]
	return user, ok
}

// QueryByCity returns all users in a given city (from local index).
func (p *Partition) QueryByCity(city string) []*User {
	p.mu.RLock()
	defer p.mu.RUnlock()

	ids, ok := p.cityIndex[city]
	if !ok {
		return nil
	}

	users := make([]*User, 0, len(ids))
	for _, id := range ids {
		if user, exists := p.users[id]; exists {
			users = append(users, user)
		}
	}
	return users
}

// QueryByEmail returns the user with a given email (from local index).
func (p *Partition) QueryByEmail(email string) (*User, bool) {
	p.mu.RLock()
	defer p.mu.RUnlock()

	id, ok := p.emailIndex[email]
	if !ok {
		return nil, false
	}
	user, exists := p.users[id]
	return user, exists
}

func (p *Partition) Size() int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return len(p.users)
}

// ========================================
// Approach 1: Local Index (Scatter-Gather)
// ========================================

// LocalIndexCluster uses document-partitioned (local) secondary indexes.
// Each partition has its own copy of the index, covering only its local data.
// Queries by secondary key require scatter-gather across all partitions.
type LocalIndexCluster struct {
	partitions []*Partition
	numParts   int
}

func NewLocalIndexCluster(numPartitions int) *LocalIndexCluster {
	parts := make([]*Partition, numPartitions)
	for i := range parts {
		parts[i] = NewPartition(i)
	}
	return &LocalIndexCluster{partitions: parts, numParts: numPartitions}
}

func (c *LocalIndexCluster) getPartition(userID string) *Partition {
	h := 0
	for _, b := range []byte(userID) {
		h += int(b)
	}
	return c.partitions[h%c.numParts]
}

func (c *LocalIndexCluster) Insert(user *User) {
	c.getPartition(user.ID).Insert(user)
}

// QueryByCity performs scatter-gather across all partitions.
func (c *LocalIndexCluster) QueryByCity(ctx context.Context, city string) []*User {
	type result struct {
		users []*User
	}

	results := make(chan result, c.numParts)

	for _, p := range c.partitions {
		go func(partition *Partition) {
			users := partition.QueryByCity(city)
			results <- result{users: users}
		}(p)
	}

	var allUsers []*User
	for i := 0; i < c.numParts; i++ {
		select {
		case r := <-results:
			allUsers = append(allUsers, r.users...)
		case <-ctx.Done():
			return allUsers // return partial results on timeout
		}
	}

	sort.Slice(allUsers, func(i, j int) bool {
		return allUsers[i].ID < allUsers[j].ID
	})
	return allUsers
}

// ==========================================
// Approach 2: Global Index (Term-Partitioned)
// ==========================================

// GlobalIndex is a separate partitioned index where the index itself
// is split by term (e.g., cities A-M on index partition 0, N-Z on partition 1).
type GlobalIndex struct {
	// Maps index term -> list of (partitionID, userID) pairs
	shards    []*IndexShard
	numShards int
}

type IndexShard struct {
	// term -> list of document locations
	entries map[string][]DocumentRef
	mu      sync.RWMutex
}

type DocumentRef struct {
	PartitionID int
	DocumentID  string
}

func NewGlobalIndex(numShards int) *GlobalIndex {
	shards := make([]*IndexShard, numShards)
	for i := range shards {
		shards[i] = &IndexShard{entries: make(map[string][]DocumentRef)}
	}
	return &GlobalIndex{shards: shards, numShards: numShards}
}

func (gi *GlobalIndex) getIndexShard(term string) *IndexShard {
	// Partition the index by the first character of the term
	h := 0
	for _, b := range []byte(term) {
		h += int(b)
	}
	return gi.shards[h%gi.numShards]
}

// UpdateIndex adds an entry to the global index.
// In a real system, this would be an async cross-partition write.
func (gi *GlobalIndex) UpdateIndex(term string, ref DocumentRef) {
	shard := gi.getIndexShard(term)
	shard.mu.Lock()
	shard.entries[term] = append(shard.entries[term], ref)
	shard.mu.Unlock()
}

// Lookup queries the global index for a term.
// Only needs to hit ONE index shard.
func (gi *GlobalIndex) Lookup(term string) []DocumentRef {
	shard := gi.getIndexShard(term)
	shard.mu.RLock()
	defer shard.mu.RUnlock()
	return shard.entries[term]
}

// GlobalIndexCluster combines data partitions with a global secondary index.
type GlobalIndexCluster struct {
	partitions []*Partition
	numParts   int
	cityIndex  *GlobalIndex
}

func NewGlobalIndexCluster(numPartitions, numIndexShards int) *GlobalIndexCluster {
	parts := make([]*Partition, numPartitions)
	for i := range parts {
		parts[i] = NewPartition(i)
	}
	return &GlobalIndexCluster{
		partitions: parts,
		numParts:   numPartitions,
		cityIndex:  NewGlobalIndex(numIndexShards),
	}
}

func (c *GlobalIndexCluster) getPartition(userID string) (int, *Partition) {
	h := 0
	for _, b := range []byte(userID) {
		h += int(b)
	}
	idx := h % c.numParts
	return idx, c.partitions[idx]
}

func (c *GlobalIndexCluster) Insert(user *User) {
	partIdx, partition := c.getPartition(user.ID)
	partition.Insert(user)

	// Also update the global index (this would be async in a real system)
	c.cityIndex.UpdateIndex(user.City, DocumentRef{
		PartitionID: partIdx,
		DocumentID:  user.ID,
	})
}

// QueryByCity uses the global index -- hits only one index shard.
func (c *GlobalIndexCluster) QueryByCity(city string) []*User {
	refs := c.cityIndex.Lookup(city)

	var users []*User
	for _, ref := range refs {
		if user, ok := c.partitions[ref.PartitionID].GetByID(ref.DocumentID); ok {
			users = append(users, user)
		}
	}

	sort.Slice(users, func(i, j int) bool {
		return users[i].ID < users[j].ID
	})
	return users
}

func main() {
	// Sample data
	testUsers := []*User{
		{ID: "u001", Name: "Alice", City: "NYC", Email: "alice@example.com", Age: 30},
		{ID: "u002", Name: "Bob", City: "LA", Email: "bob@example.com", Age: 25},
		{ID: "u003", Name: "Charlie", City: "NYC", Email: "charlie@example.com", Age: 35},
		{ID: "u004", Name: "Diana", City: "Chicago", Email: "diana@example.com", Age: 28},
		{ID: "u005", Name: "Eve", City: "NYC", Email: "eve@example.com", Age: 32},
		{ID: "u006", Name: "Frank", City: "LA", Email: "frank@example.com", Age: 40},
		{ID: "u007", Name: "Grace", City: "NYC", Email: "grace@example.com", Age: 27},
		{ID: "u008", Name: "Henry", City: "Chicago", Email: "henry@example.com", Age: 33},
		{ID: "u009", Name: "Iris", City: "LA", Email: "iris@example.com", Age: 29},
		{ID: "u010", Name: "Jack", City: "NYC", Email: "jack@example.com", Age: 31},
	}

	// === Approach 1: Local Index with Scatter-Gather ===
	fmt.Println("=== Approach 1: Local Index (Scatter-Gather) ===")
	localCluster := NewLocalIndexCluster(3)
	for _, u := range testUsers {
		localCluster.Insert(u)
	}

	fmt.Println("Data distribution:")
	for i, p := range localCluster.partitions {
		fmt.Printf("  Partition %d: %d users\n", i, p.Size())
	}

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	start := time.Now()
	nycUsers := localCluster.QueryByCity(ctx, "NYC")
	localDuration := time.Since(start)

	fmt.Printf("\nLocal index query: city=NYC (scatter-gather across %d partitions)\n",
		len(localCluster.partitions))
	fmt.Printf("  Found %d users in %v:\n", len(nycUsers), localDuration)
	for _, u := range nycUsers {
		fmt.Printf("    %s: %s (%s)\n", u.ID, u.Name, u.City)
	}

	// === Approach 2: Global Index ===
	fmt.Println("\n=== Approach 2: Global Index (Term-Partitioned) ===")
	globalCluster := NewGlobalIndexCluster(3, 2) // 3 data partitions, 2 index shards
	for _, u := range testUsers {
		globalCluster.Insert(u)
	}

	start = time.Now()
	nycUsersGlobal := globalCluster.QueryByCity("NYC")
	globalDuration := time.Since(start)

	fmt.Printf("Global index query: city=NYC (single index shard lookup)\n")
	fmt.Printf("  Found %d users in %v:\n", len(nycUsersGlobal), globalDuration)
	for _, u := range nycUsersGlobal {
		fmt.Printf("    %s: %s (%s)\n", u.ID, u.Name, u.City)
	}

	// === Comparison ===
	fmt.Println("\n=== Trade-off Comparison ===")
	fmt.Println("┌─────────────────────┬──────────────────────────┬──────────────────────────┐")
	fmt.Println("│                     │ Local Index               │ Global Index              │")
	fmt.Println("├─────────────────────┼──────────────────────────┼──────────────────────────┤")
	fmt.Println("│ Write performance   │ Fast (local only)         │ Slower (cross-partition)  │")
	fmt.Println("│ Read performance    │ Slower (scatter-gather)   │ Fast (single shard)       │")
	fmt.Println("│ Write consistency   │ Immediate                 │ Often async (eventual)    │")
	fmt.Println("│ Read completeness   │ Always complete            │ May be stale              │")
	fmt.Println("│ Implementation      │ Simpler                   │ More complex              │")
	fmt.Println("│ Used by             │ MongoDB, Elasticsearch    │ DynamoDB, Riak            │")
	fmt.Println("└─────────────────────┴──────────────────────────┴──────────────────────────┘")

	_ = strings.NewReader("")
}
```

---

## 7. Rebalancing Partitions

### Why Rebalancing Is Needed

Over time, the cluster changes:
- **Data grows**: Some partitions get larger than others.
- **Nodes are added**: New nodes should take some partitions to spread the load.
- **Nodes fail**: Failed nodes' partitions must be reassigned to surviving nodes.
- **Load shifts**: Query patterns change, creating new hot spots.

Rebalancing moves partitions between nodes to restore balance. The key challenge: minimize data movement while achieving good balance.

### Rebalancing Strategies

```
Strategy 1: Fixed Number of Partitions
Create many more partitions than nodes at the start (e.g., 1000 partitions for 10 nodes).
When a node is added, transfer some partitions to it.

Before (3 nodes, 12 partitions):
  Node A: [P0, P1, P2, P3]
  Node B: [P4, P5, P6, P7]
  Node C: [P8, P9, P10, P11]

After adding Node D:
  Node A: [P0, P1, P2]
  Node B: [P4, P5, P6]
  Node C: [P8, P9, P10]
  Node D: [P3, P7, P11]      ← took one partition from each node


Strategy 2: Dynamic Partitioning
Start with fewer partitions and split them when they get too large.
Merge small partitions when they shrink.

  [P0: 500MB] → split → [P0a: 250MB] [P0b: 250MB]
  [P1: 10MB] + [P2: 15MB] → merge → [P1+2: 25MB]


Strategy 3: Proportional to Node Count
Each node gets a fixed number of partitions.
When a node joins, it splits existing partitions.
```

### Complete Implementation

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sort"
	"strings"
	"sync"
)

// Partition represents a data partition that can be moved between nodes.
type Partition struct {
	ID       int
	DataSize int64 // bytes
	KeyCount int
	MinKey   string
	MaxKey   string
}

// NodeInfo represents a cluster node.
type NodeInfo struct {
	ID         string
	Partitions map[int]*Partition
	Capacity   int64 // max bytes this node can hold
}

func (n *NodeInfo) TotalSize() int64 {
	var total int64
	for _, p := range n.Partitions {
		total += p.DataSize
	}
	return total
}

func (n *NodeInfo) TotalKeys() int {
	total := 0
	for _, p := range n.Partitions {
		total += p.KeyCount
	}
	return total
}

// RebalanceManager manages partition assignment across nodes.
type RebalanceManager struct {
	partitions map[int]*Partition
	nodes      map[string]*NodeInfo
	assignment map[int]string // partitionID -> nodeID
	mu         sync.Mutex
}

func NewRebalanceManager() *RebalanceManager {
	return &RebalanceManager{
		partitions: make(map[int]*Partition),
		nodes:      make(map[string]*NodeInfo),
		assignment: make(map[int]string),
	}
}

// AddNode adds a new node to the cluster.
func (rm *RebalanceManager) AddNode(nodeID string, capacity int64) {
	rm.mu.Lock()
	defer rm.mu.Unlock()

	rm.nodes[nodeID] = &NodeInfo{
		ID:         nodeID,
		Partitions: make(map[int]*Partition),
		Capacity:   capacity,
	}
}

// RemoveNode removes a node, marking its partitions as unassigned.
func (rm *RebalanceManager) RemoveNode(nodeID string) []int {
	rm.mu.Lock()
	defer rm.mu.Unlock()

	node, ok := rm.nodes[nodeID]
	if !ok {
		return nil
	}

	var orphaned []int
	for pid := range node.Partitions {
		orphaned = append(orphaned, pid)
		delete(rm.assignment, pid)
	}

	delete(rm.nodes, nodeID)
	return orphaned
}

// AddPartition creates a new partition.
func (rm *RebalanceManager) AddPartition(p *Partition) {
	rm.mu.Lock()
	defer rm.mu.Unlock()
	rm.partitions[p.ID] = p
}

// AssignPartition assigns a partition to a specific node.
func (rm *RebalanceManager) AssignPartition(partitionID int, nodeID string) {
	rm.mu.Lock()
	defer rm.mu.Unlock()

	// Remove from old node
	if oldNodeID, exists := rm.assignment[partitionID]; exists {
		if oldNode, ok := rm.nodes[oldNodeID]; ok {
			delete(oldNode.Partitions, partitionID)
		}
	}

	// Assign to new node
	rm.assignment[partitionID] = nodeID
	if node, ok := rm.nodes[nodeID]; ok {
		if p, exists := rm.partitions[partitionID]; exists {
			node.Partitions[partitionID] = p
		}
	}
}

// MoveRecord represents a partition move from one node to another.
type MoveRecord struct {
	PartitionID int
	FromNode    string
	ToNode      string
	DataSize    int64
}

// Rebalance computes and executes a rebalancing plan.
// Strategy: Move partitions from the most loaded nodes to the least loaded
// until the load difference is within a threshold.
func (rm *RebalanceManager) Rebalance() []MoveRecord {
	rm.mu.Lock()
	defer rm.mu.Unlock()

	if len(rm.nodes) == 0 {
		return nil
	}

	// Calculate total data and ideal per-node target
	var totalSize int64
	for _, p := range rm.partitions {
		totalSize += p.DataSize
	}
	idealPerNode := totalSize / int64(len(rm.nodes))
	threshold := idealPerNode / 10 // 10% tolerance

	var moves []MoveRecord

	// Iterative rebalancing: find the most overloaded and most underloaded nodes
	for iterations := 0; iterations < 100; iterations++ {
		// Sort nodes by load
		type nodeLoad struct {
			id   string
			load int64
		}
		var loads []nodeLoad
		for id, node := range rm.nodes {
			loads = append(loads, nodeLoad{id: id, load: node.TotalSize()})
		}
		sort.Slice(loads, func(i, j int) bool {
			return loads[i].load > loads[j].load // descending
		})

		mostLoaded := loads[0]
		leastLoaded := loads[len(loads)-1]

		// Check if rebalancing is needed
		if mostLoaded.load-leastLoaded.load <= threshold {
			break // balanced enough
		}

		// Find a partition to move from most loaded to least loaded
		heaviestNode := rm.nodes[mostLoaded.id]
		var bestPartition *Partition
		bestFit := int64(math.MaxInt64)

		for _, p := range heaviestNode.Partitions {
			// Find the partition whose move would bring loads closest to ideal
			newHeavy := mostLoaded.load - p.DataSize
			newLight := leastLoaded.load + p.DataSize
			imbalance := abs64(newHeavy-idealPerNode) + abs64(newLight-idealPerNode)
			if imbalance < bestFit {
				bestFit = imbalance
				bestPartition = p
			}
		}

		if bestPartition == nil {
			break
		}

		// Execute the move
		move := MoveRecord{
			PartitionID: bestPartition.ID,
			FromNode:    mostLoaded.id,
			ToNode:      leastLoaded.id,
			DataSize:    bestPartition.DataSize,
		}
		moves = append(moves, move)

		// Update internal state
		delete(heaviestNode.Partitions, bestPartition.ID)
		rm.nodes[leastLoaded.id].Partitions[bestPartition.ID] = bestPartition
		rm.assignment[bestPartition.ID] = leastLoaded.id
	}

	return moves
}

// PrintState shows the current partition assignment.
func (rm *RebalanceManager) PrintState() {
	rm.mu.Lock()
	defer rm.mu.Unlock()

	var totalSize int64
	for _, p := range rm.partitions {
		totalSize += p.DataSize
	}

	fmt.Printf("Cluster state (%d partitions, %d nodes, total %.1f MB):\n",
		len(rm.partitions), len(rm.nodes), float64(totalSize)/1024/1024)

	nodeIDs := make([]string, 0, len(rm.nodes))
	for id := range rm.nodes {
		nodeIDs = append(nodeIDs, id)
	}
	sort.Strings(nodeIDs)

	idealPerNode := float64(totalSize) / float64(len(rm.nodes))

	for _, id := range nodeIDs {
		node := rm.nodes[id]
		size := node.TotalSize()
		keys := node.TotalKeys()
		partIDs := make([]int, 0, len(node.Partitions))
		for pid := range node.Partitions {
			partIDs = append(partIDs, pid)
		}
		sort.Ints(partIDs)

		bar := strings.Repeat("#", int(float64(size)/float64(totalSize)*40))
		skew := float64(size)/idealPerNode - 1.0

		fmt.Printf("  %-10s: %3d partitions, %6d keys, %6.1f MB [%-40s] skew: %+.1f%%\n",
			id, len(node.Partitions), keys, float64(size)/1024/1024, bar, skew*100)
	}
}

func abs64(x int64) int64 {
	if x < 0 {
		return -x
	}
	return x
}

func main() {
	rm := NewRebalanceManager()

	// Create a cluster with 3 nodes
	rm.AddNode("node-A", 10*1024*1024*1024) // 10 GB
	rm.AddNode("node-B", 10*1024*1024*1024)
	rm.AddNode("node-C", 10*1024*1024*1024)

	// Create 12 partitions with varying sizes
	for i := 0; i < 12; i++ {
		size := int64(50+rand.Intn(150)) * 1024 * 1024 // 50-200 MB
		keys := 10000 + rand.Intn(50000)
		rm.AddPartition(&Partition{
			ID:       i,
			DataSize: size,
			KeyCount: keys,
			MinKey:   fmt.Sprintf("key_%04d", i*10000),
			MaxKey:   fmt.Sprintf("key_%04d", (i+1)*10000-1),
		})
	}

	// Initial assignment: all partitions on node-A (simulating a migration scenario)
	fmt.Println("=== Initial State: All partitions on node-A ===")
	for i := 0; i < 12; i++ {
		rm.AssignPartition(i, "node-A")
	}
	rm.PrintState()

	// Rebalance
	fmt.Println("\n=== Rebalancing ===")
	moves := rm.Rebalance()
	for _, m := range moves {
		fmt.Printf("  Move partition %d: %s -> %s (%.1f MB)\n",
			m.PartitionID, m.FromNode, m.ToNode, float64(m.DataSize)/1024/1024)
	}
	fmt.Printf("Total moves: %d\n", len(moves))

	fmt.Println("\n=== State After Rebalancing ===")
	rm.PrintState()

	// Now add a 4th node
	fmt.Println("\n=== Adding node-D ===")
	rm.AddNode("node-D", 10*1024*1024*1024)

	moves = rm.Rebalance()
	fmt.Println("Rebalancing after adding node-D:")
	for _, m := range moves {
		fmt.Printf("  Move partition %d: %s -> %s (%.1f MB)\n",
			m.PartitionID, m.FromNode, m.ToNode, float64(m.DataSize)/1024/1024)
	}
	fmt.Printf("Total moves: %d\n", len(moves))

	fmt.Println("\n=== State After Adding node-D ===")
	rm.PrintState()

	// Simulate node failure: remove node-B
	fmt.Println("\n=== Simulating node-B Failure ===")
	orphaned := rm.RemoveNode("node-B")
	fmt.Printf("Orphaned partitions from node-B: %v\n", orphaned)

	// Reassign orphaned partitions
	remainingNodes := []string{"node-A", "node-C", "node-D"}
	for i, pid := range orphaned {
		target := remainingNodes[i%len(remainingNodes)]
		rm.AssignPartition(pid, target)
	}

	fmt.Println("After reassigning orphaned partitions:")
	rm.PrintState()

	// Final rebalance
	moves = rm.Rebalance()
	if len(moves) > 0 {
		fmt.Println("\nFinal rebalancing:")
		for _, m := range moves {
			fmt.Printf("  Move partition %d: %s -> %s (%.1f MB)\n",
				m.PartitionID, m.FromNode, m.ToNode, float64(m.DataSize)/1024/1024)
		}
	}

	fmt.Println("\n=== Final State ===")
	rm.PrintState()
}
```

### Rebalancing Strategies Compared

| Strategy | Pros | Cons | Used By |
|----------|------|------|---------|
| **Fixed partitions** | Simple, predictable moves | Must choose partition count upfront | Riak, Elasticsearch, Couchbase |
| **Dynamic splitting** | Adapts to data growth | More complex, split/merge overhead | HBase, RethinkDB |
| **Proportional** | Adapts to cluster size | Splitting existing partitions is expensive | Cassandra (vnodes) |

---

## 8. Request Routing

### The Routing Problem

When a client wants to read or write a key, how does it know which node to contact? The partition assignment is constantly changing (rebalancing, node failures, splits). Three approaches exist:

```
Approach 1: Client-Side Routing
Client knows the full partition map and contacts nodes directly.
  ┌────────┐
  │ Client │──knows map──► Node A / Node B / Node C
  └────────┘
  Pro: No extra hop. Con: Client must track partition changes.

Approach 2: Routing Tier
Client connects to a routing tier that knows the partition map.
  ┌────────┐    ┌────────┐
  │ Client │───►│ Router │───► Node A / Node B / Node C
  └────────┘    └────────┘
  Pro: Clients are simple. Con: Extra network hop.

Approach 3: Contact Any Node (Gossip)
Client contacts any node; that node forwards if needed.
  ┌────────┐    ┌────────┐
  │ Client │───►│ Node B │──forward──► Node A (owns key)
  └────────┘    └────────┘
  Pro: No special routing. Con: Extra hop if wrong node.
```

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"hash/fnv"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// PartitionMap maps partition IDs to node addresses.
type PartitionMap struct {
	mapping map[int]string // partitionID -> nodeAddr
	version int64
	mu      sync.RWMutex
}

func NewPartitionMap() *PartitionMap {
	return &PartitionMap{
		mapping: make(map[int]string),
	}
}

func (pm *PartitionMap) Set(partitionID int, nodeAddr string) {
	pm.mu.Lock()
	pm.mapping[partitionID] = nodeAddr
	pm.version++
	pm.mu.Unlock()
}

func (pm *PartitionMap) Get(partitionID int) (string, bool) {
	pm.mu.RLock()
	addr, ok := pm.mapping[partitionID]
	pm.mu.RUnlock()
	return addr, ok
}

func (pm *PartitionMap) Version() int64 {
	pm.mu.RLock()
	defer pm.mu.RUnlock()
	return pm.version
}

func (pm *PartitionMap) Clone() map[int]string {
	pm.mu.RLock()
	defer pm.mu.RUnlock()
	clone := make(map[int]string, len(pm.mapping))
	for k, v := range pm.mapping {
		clone[k] = v
	}
	return clone
}

// =====================
// Approach 1: Client-Side Router
// =====================

// ClientRouter maintains a local copy of the partition map.
// Periodically refreshes from a coordination service.
type ClientRouter struct {
	partitionMap map[int]string // local copy of partition map
	numParts     int
	mapVersion   int64
	refreshInterval time.Duration
	mu              sync.RWMutex
	lookups         atomic.Int64
	refreshes       atomic.Int64
}

func NewClientRouter(numPartitions int) *ClientRouter {
	return &ClientRouter{
		partitionMap:    make(map[int]string),
		numParts:        numPartitions,
		refreshInterval: 5 * time.Second,
	}
}

func (cr *ClientRouter) UpdateMap(newMap map[int]string) {
	cr.mu.Lock()
	cr.partitionMap = newMap
	cr.mapVersion++
	cr.mu.Unlock()
	cr.refreshes.Add(1)
}

func (cr *ClientRouter) Route(key string) string {
	h := fnv.New32a()
	h.Write([]byte(key))
	partitionID := int(h.Sum32()) % cr.numParts

	cr.mu.RLock()
	addr := cr.partitionMap[partitionID]
	cr.mu.RUnlock()
	cr.lookups.Add(1)
	return addr
}

func (cr *ClientRouter) Stats() (lookups, refreshes int64) {
	return cr.lookups.Load(), cr.refreshes.Load()
}

// =====================
// Approach 2: Routing Tier
// =====================

// RoutingTier acts as a proxy that routes requests to the correct node.
type RoutingTier struct {
	partitionMap *PartitionMap
	numParts     int
	forwarded    atomic.Int64
}

func NewRoutingTier(numPartitions int, pm *PartitionMap) *RoutingTier {
	return &RoutingTier{
		partitionMap: pm,
		numParts:     numPartitions,
	}
}

func (rt *RoutingTier) Route(key string) (string, int) {
	h := fnv.New32a()
	h.Write([]byte(key))
	partitionID := int(h.Sum32()) % rt.numParts

	addr, _ := rt.partitionMap.Get(partitionID)
	rt.forwarded.Add(1)
	return addr, partitionID
}

// =====================
// Approach 3: Gossip-Based Routing
// =====================

// GossipNode is a node that knows about the partition map and can
// either serve requests locally or forward them to the correct node.
type GossipNode struct {
	ID           string
	Address      string
	localParts   map[int]bool         // partitions this node owns
	partitionMap map[int]string       // full cluster map (learned via gossip)
	data         map[string]string    // local data store
	peers        map[string]*GossipNode // known peers
	forwarded    atomic.Int64
	served       atomic.Int64
	mu           sync.RWMutex
}

func NewGossipNode(id, address string) *GossipNode {
	return &GossipNode{
		ID:           id,
		Address:      address,
		localParts:   make(map[int]bool),
		partitionMap: make(map[int]string),
		data:         make(map[string]string),
		peers:        make(map[string]*GossipNode),
	}
}

func (gn *GossipNode) OwnsPartition(partitionID int) bool {
	gn.mu.RLock()
	defer gn.mu.RUnlock()
	return gn.localParts[partitionID]
}

func (gn *GossipNode) AssignPartition(partitionID int) {
	gn.mu.Lock()
	gn.localParts[partitionID] = true
	gn.partitionMap[partitionID] = gn.Address
	gn.mu.Unlock()
}

func (gn *GossipNode) AddPeer(peer *GossipNode) {
	gn.mu.Lock()
	gn.peers[peer.ID] = peer
	gn.mu.Unlock()
}

// GossipExchange simulates a gossip round where nodes exchange partition maps.
func (gn *GossipNode) GossipExchange() {
	gn.mu.Lock()
	defer gn.mu.Unlock()

	for _, peer := range gn.peers {
		peer.mu.RLock()
		for partID, addr := range peer.partitionMap {
			gn.partitionMap[partID] = addr
		}
		peer.mu.RUnlock()
	}
}

const numGossipPartitions = 12

// HandleRequest processes a request, forwarding if necessary.
func (gn *GossipNode) HandleRequest(key, value string, isWrite bool) (string, string) {
	h := fnv.New32a()
	h.Write([]byte(key))
	partitionID := int(h.Sum32()) % numGossipPartitions

	if gn.OwnsPartition(partitionID) {
		// Serve locally
		gn.mu.Lock()
		if isWrite {
			gn.data[key] = value
		}
		result := gn.data[key]
		gn.mu.Unlock()
		gn.served.Add(1)
		return result, gn.ID
	}

	// Forward to correct node
	gn.mu.RLock()
	targetAddr := gn.partitionMap[partitionID]
	var targetNode *GossipNode
	for _, peer := range gn.peers {
		if peer.Address == targetAddr {
			targetNode = peer
			break
		}
	}
	gn.mu.RUnlock()

	if targetNode == nil {
		return "", "ERROR: no route"
	}

	gn.forwarded.Add(1)
	return targetNode.HandleRequest(key, value, isWrite)
}

func main() {
	const numPartitions = 12

	// === Approach 1: Client-Side Routing ===
	fmt.Println("=== Approach 1: Client-Side Routing ===")
	clientRouter := NewClientRouter(numPartitions)

	// Simulate coordination service providing the map
	initialMap := map[int]string{
		0: "node-a:8080", 1: "node-a:8080", 2: "node-a:8080", 3: "node-a:8080",
		4: "node-b:8080", 5: "node-b:8080", 6: "node-b:8080", 7: "node-b:8080",
		8: "node-c:8080", 9: "node-c:8080", 10: "node-c:8080", 11: "node-c:8080",
	}
	clientRouter.UpdateMap(initialMap)

	// Route some keys
	testKeys := []string{"user:alice", "user:bob", "order:123", "product:xyz"}
	for _, key := range testKeys {
		addr := clientRouter.Route(key)
		fmt.Printf("  %s -> %s\n", key, addr)
	}

	// Simulate a partition map update (node-d joins, takes some partitions)
	fmt.Println("\nPartition map update (node-d takes partitions 3,7,11):")
	updatedMap := make(map[int]string)
	for k, v := range initialMap {
		updatedMap[k] = v
	}
	updatedMap[3] = "node-d:8080"
	updatedMap[7] = "node-d:8080"
	updatedMap[11] = "node-d:8080"
	clientRouter.UpdateMap(updatedMap)

	for _, key := range testKeys {
		addr := clientRouter.Route(key)
		fmt.Printf("  %s -> %s\n", key, addr)
	}

	lookups, refreshes := clientRouter.Stats()
	fmt.Printf("Stats: lookups=%d, map refreshes=%d\n", lookups, refreshes)

	// === Approach 2: Routing Tier ===
	fmt.Println("\n=== Approach 2: Routing Tier ===")
	pm := NewPartitionMap()
	for pid, addr := range updatedMap {
		pm.Set(pid, addr)
	}

	router := NewRoutingTier(numPartitions, pm)
	for _, key := range testKeys {
		addr, partID := router.Route(key)
		fmt.Printf("  %s -> partition %d -> %s\n", key, partID, addr)
	}
	fmt.Printf("Requests forwarded: %d\n", router.forwarded.Load())

	// === Approach 3: Gossip-Based Routing ===
	fmt.Println("\n=== Approach 3: Gossip-Based Routing ===")

	// Create nodes
	nodeA := NewGossipNode("node-A", "10.0.0.1:8080")
	nodeB := NewGossipNode("node-B", "10.0.0.2:8080")
	nodeC := NewGossipNode("node-C", "10.0.0.3:8080")

	// Assign partitions
	for i := 0; i < 4; i++ {
		nodeA.AssignPartition(i)
	}
	for i := 4; i < 8; i++ {
		nodeB.AssignPartition(i)
	}
	for i := 8; i < 12; i++ {
		nodeC.AssignPartition(i)
	}

	// Connect peers
	nodeA.AddPeer(nodeB)
	nodeA.AddPeer(nodeC)
	nodeB.AddPeer(nodeA)
	nodeB.AddPeer(nodeC)
	nodeC.AddPeer(nodeA)
	nodeC.AddPeer(nodeB)

	// Run gossip rounds to propagate partition map
	fmt.Println("Running gossip rounds...")
	for round := 0; round < 3; round++ {
		nodeA.GossipExchange()
		nodeB.GossipExchange()
		nodeC.GossipExchange()
	}

	// Client sends all requests to nodeA (could be any node)
	fmt.Println("Client sends all requests to node-A:")
	for _, key := range []string{"user:1", "user:50", "user:100", "user:150", "user:200"} {
		// Write
		nodeA.HandleRequest(key, fmt.Sprintf("data-for-%s", key), true)
		// Read
		value, servedBy := nodeA.HandleRequest(key, "", false)
		fmt.Printf("  %s -> value=%q, served by %s\n", key, value, servedBy)
	}

	fmt.Printf("\nnode-A: served=%d, forwarded=%d\n", nodeA.served.Load(), nodeA.forwarded.Load())
	fmt.Printf("node-B: served=%d, forwarded=%d\n", nodeB.served.Load(), nodeB.forwarded.Load())
	fmt.Printf("node-C: served=%d, forwarded=%d\n", nodeC.served.Load(), nodeC.forwarded.Load())

	// === Comparison ===
	fmt.Println("\n=== Routing Approaches Compared ===")
	fmt.Println("┌────────────────────┬──────────────────┬──────────────────┬──────────────────┐")
	fmt.Println("│ Approach           │ Latency          │ Complexity       │ Example Systems  │")
	fmt.Println("├────────────────────┼──────────────────┼──────────────────┼──────────────────┤")
	fmt.Println("│ Client-side        │ Lowest (direct)  │ Smart clients    │ Cassandra drivers│")
	fmt.Println("│ Routing tier       │ +1 hop           │ Simple clients   │ MongoDB mongos   │")
	fmt.Println("│ Gossip / any-node  │ +0-1 hop         │ P2P protocol     │ Riak, Cassandra  │")
	fmt.Println("│ ZooKeeper-based    │ +1 hop (cached)  │ External service │ HBase, Kafka     │")
	fmt.Println("└────────────────────┴──────────────────┴──────────────────┴──────────────────┘")

	_ = context.Background()
	_ = rand.Intn
}
```

---

## 9. Scatter-Gather Queries

### The Problem

When you partition by a primary key (e.g., `user_id`), point queries are efficient: you hash the key, find the partition, and contact one node. But what about queries that do not include the partition key?

- "Find all users in NYC" -- must ask every partition.
- "Find the top 10 most expensive products" -- must ask every partition and merge results.
- "Count all orders placed today" -- must ask every partition and sum results.

This pattern is called **scatter-gather**: scatter the query to all partitions in parallel, gather the results, and merge them.

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sort"
	"sync"
	"time"
)

// Record represents a stored document.
type Record struct {
	Key       string
	Value     string
	Timestamp time.Time
	Score     float64
}

// Query represents a query to execute against partitions.
type Query struct {
	Type      string // "filter", "top_n", "count", "aggregate"
	Field     string
	Value     string
	Limit     int
	Predicate func(*Record) bool
}

// QueryResult holds results from a single partition.
type QueryResult struct {
	PartitionID int
	Records     []*Record
	Count       int
	Sum         float64
	Error       error
	Duration    time.Duration
}

// DataPartition simulates a partition holding data.
type DataPartition struct {
	id      int
	records []*Record
	mu      sync.RWMutex
}

func NewDataPartition(id int) *DataPartition {
	return &DataPartition{id: id}
}

func (dp *DataPartition) Insert(r *Record) {
	dp.mu.Lock()
	dp.records = append(dp.records, r)
	dp.mu.Unlock()
}

func (dp *DataPartition) ExecuteQuery(ctx context.Context, q Query) QueryResult {
	start := time.Now()
	dp.mu.RLock()
	defer dp.mu.RUnlock()

	result := QueryResult{PartitionID: dp.id}

	// Simulate some processing time proportional to data size
	processingTime := time.Duration(len(dp.records)) * time.Microsecond
	select {
	case <-time.After(processingTime):
		// processing complete
	case <-ctx.Done():
		result.Error = ctx.Err()
		result.Duration = time.Since(start)
		return result
	}

	switch q.Type {
	case "filter":
		for _, r := range dp.records {
			if q.Predicate != nil && q.Predicate(r) {
				result.Records = append(result.Records, r)
			}
		}
		result.Count = len(result.Records)

	case "top_n":
		// Return top N by score
		matching := make([]*Record, 0)
		for _, r := range dp.records {
			if q.Predicate == nil || q.Predicate(r) {
				matching = append(matching, r)
			}
		}
		sort.Slice(matching, func(i, j int) bool {
			return matching[i].Score > matching[j].Score
		})
		if len(matching) > q.Limit {
			matching = matching[:q.Limit]
		}
		result.Records = matching
		result.Count = len(matching)

	case "count":
		for _, r := range dp.records {
			if q.Predicate == nil || q.Predicate(r) {
				result.Count++
			}
		}

	case "aggregate":
		for _, r := range dp.records {
			if q.Predicate == nil || q.Predicate(r) {
				result.Sum += r.Score
				result.Count++
			}
		}
	}

	result.Duration = time.Since(start)
	return result
}

// ScatterGatherCluster manages partitions and executes scatter-gather queries.
type ScatterGatherCluster struct {
	partitions []*DataPartition
	numParts   int
}

func NewScatterGatherCluster(numPartitions int) *ScatterGatherCluster {
	parts := make([]*DataPartition, numPartitions)
	for i := range parts {
		parts[i] = NewDataPartition(i)
	}
	return &ScatterGatherCluster{partitions: parts, numParts: numPartitions}
}

func (c *ScatterGatherCluster) Insert(r *Record) {
	// Hash-partition by key
	h := 0
	for _, b := range []byte(r.Key) {
		h += int(b)
	}
	c.partitions[h%c.numParts].Insert(r)
}

// ScatterGather sends a query to all partitions in parallel and merges results.
func (c *ScatterGatherCluster) ScatterGather(ctx context.Context, q Query) ([]QueryResult, error) {
	results := make([]QueryResult, c.numParts)
	var wg sync.WaitGroup

	for i, p := range c.partitions {
		wg.Add(1)
		go func(idx int, partition *DataPartition) {
			defer wg.Done()
			results[idx] = partition.ExecuteQuery(ctx, q)
		}(i, p)
	}

	wg.Wait()

	// Check for errors
	for _, r := range results {
		if r.Error != nil {
			return results, fmt.Errorf("partition %d failed: %w", r.PartitionID, r.Error)
		}
	}

	return results, nil
}

// ScatterGatherWithTimeout adds a per-partition timeout.
func (c *ScatterGatherCluster) ScatterGatherWithTimeout(
	ctx context.Context, q Query, timeout time.Duration,
) ([]QueryResult, []int) {
	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	type indexedResult struct {
		idx    int
		result QueryResult
	}

	resultsCh := make(chan indexedResult, c.numParts)
	for i, p := range c.partitions {
		go func(idx int, partition *DataPartition) {
			r := partition.ExecuteQuery(ctx, q)
			resultsCh <- indexedResult{idx: idx, result: r}
		}(i, p)
	}

	results := make([]QueryResult, c.numParts)
	received := make([]bool, c.numParts)
	var failedPartitions []int

	for i := 0; i < c.numParts; i++ {
		select {
		case ir := <-resultsCh:
			results[ir.idx] = ir.result
			received[ir.idx] = true
			if ir.result.Error != nil {
				failedPartitions = append(failedPartitions, ir.idx)
			}
		case <-ctx.Done():
			// Collect what we have, mark the rest as failed
			for j := 0; j < c.numParts; j++ {
				if !received[j] {
					failedPartitions = append(failedPartitions, j)
				}
			}
			return results, failedPartitions
		}
	}

	return results, failedPartitions
}

// MergeFilterResults merges filter results from all partitions.
func MergeFilterResults(results []QueryResult) []*Record {
	var all []*Record
	for _, r := range results {
		all = append(all, r.Records...)
	}
	sort.Slice(all, func(i, j int) bool {
		return all[i].Key < all[j].Key
	})
	return all
}

// MergeTopN merges top-N results from all partitions.
func MergeTopN(results []QueryResult, n int) []*Record {
	var all []*Record
	for _, r := range results {
		all = append(all, r.Records...)
	}
	sort.Slice(all, func(i, j int) bool {
		return all[i].Score > all[j].Score
	})
	if len(all) > n {
		all = all[:n]
	}
	return all
}

// MergeCount sums counts from all partitions.
func MergeCount(results []QueryResult) int {
	total := 0
	for _, r := range results {
		total += r.Count
	}
	return total
}

// MergeAggregate sums aggregates from all partitions.
func MergeAggregate(results []QueryResult) (count int, sum, avg float64) {
	for _, r := range results {
		count += r.Count
		sum += r.Sum
	}
	if count > 0 {
		avg = sum / float64(count)
	}
	return
}

func main() {
	cluster := NewScatterGatherCluster(6)

	// Insert 10,000 records
	cities := []string{"NYC", "LA", "Chicago", "Houston", "Phoenix", "Philadelphia"}
	for i := 0; i < 10000; i++ {
		cluster.Insert(&Record{
			Key:       fmt.Sprintf("user:%05d", i),
			Value:     fmt.Sprintf(`{"name":"User %d","city":"%s"}`, i, cities[i%len(cities)]),
			Timestamp: time.Now().Add(-time.Duration(rand.Intn(365*24)) * time.Hour),
			Score:     rand.Float64() * 100,
		})
	}

	ctx := context.Background()

	// === Query 1: Filter by predicate (scatter-gather) ===
	fmt.Println("=== Query 1: Filter users with score > 95 ===")
	start := time.Now()
	results, err := cluster.ScatterGather(ctx, Query{
		Type: "filter",
		Predicate: func(r *Record) bool {
			return r.Score > 95.0
		},
	})
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	merged := MergeFilterResults(results)
	fmt.Printf("Found %d users with score > 95 in %v\n", len(merged), time.Since(start))
	fmt.Println("Per-partition breakdown:")
	for _, r := range results {
		fmt.Printf("  Partition %d: %d results in %v\n", r.PartitionID, r.Count, r.Duration)
	}
	if len(merged) > 0 {
		fmt.Printf("  First: %s (score=%.2f)\n", merged[0].Key, merged[0].Score)
		fmt.Printf("  Last:  %s (score=%.2f)\n", merged[len(merged)-1].Key, merged[len(merged)-1].Score)
	}

	// === Query 2: Top-N across partitions ===
	fmt.Println("\n=== Query 2: Top 10 users by score ===")
	start = time.Now()
	results, err = cluster.ScatterGather(ctx, Query{
		Type:  "top_n",
		Limit: 10,
	})
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	topN := MergeTopN(results, 10)
	fmt.Printf("Top 10 users (merged from %d partitions) in %v:\n",
		len(cluster.partitions), time.Since(start))
	for i, r := range topN {
		fmt.Printf("  %2d. %s  score=%.4f\n", i+1, r.Key, r.Score)
	}

	// === Query 3: Count across partitions ===
	fmt.Println("\n=== Query 3: Count users created in last 30 days ===")
	cutoff := time.Now().Add(-30 * 24 * time.Hour)
	start = time.Now()
	results, err = cluster.ScatterGather(ctx, Query{
		Type: "count",
		Predicate: func(r *Record) bool {
			return r.Timestamp.After(cutoff)
		},
	})
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	totalCount := MergeCount(results)
	fmt.Printf("Users created in last 30 days: %d (queried in %v)\n", totalCount, time.Since(start))

	// === Query 4: Aggregate (average score) ===
	fmt.Println("\n=== Query 4: Average score across all users ===")
	start = time.Now()
	results, err = cluster.ScatterGather(ctx, Query{
		Type: "aggregate",
	})
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	count, sum, avg := MergeAggregate(results)
	fmt.Printf("Total users: %d, Sum of scores: %.2f, Average: %.4f (in %v)\n",
		count, sum, avg, time.Since(start))

	// === Query 5: With timeout (simulating slow partition) ===
	fmt.Println("\n=== Query 5: Scatter-gather with timeout ===")
	results, failed := cluster.ScatterGatherWithTimeout(ctx, Query{
		Type: "count",
	}, 50*time.Millisecond)
	totalCount = MergeCount(results)
	fmt.Printf("Count (with 50ms timeout): %d\n", totalCount)
	if len(failed) > 0 {
		fmt.Printf("Failed/timed out partitions: %v\n", failed)
	} else {
		fmt.Println("All partitions responded within timeout.")
	}
}
```

### Scatter-Gather Performance Implications

The total latency of a scatter-gather query is determined by the **slowest partition** (the tail latency). If you have 100 partitions and each has a 99th percentile latency of 10ms, the probability that at least one takes >10ms is high.

```
Single partition:   P99 = 10ms
10 partitions:      P99 = ~20ms  (likely to hit one slow partition)
100 partitions:     P99 = ~30ms  (almost certain to hit a slow one)
```

Mitigation strategies:
- **Hedged requests**: Send the request to a replica if the primary does not respond quickly.
- **Partial results**: Return what you have after a timeout, flagging incomplete results.
- **Partition pruning**: If you know the query only touches certain partitions, skip the others.

---

## 10. Partition-Aware Connection Pooling

### The Problem

In a partitioned system, your application needs connections to multiple nodes. Maintaining a single shared pool wastes connections on nodes you rarely contact. Partition-aware pooling maintains separate connection pools per node, sized according to the expected traffic pattern.

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"hash/fnv"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// Connection represents a database/service connection.
type Connection struct {
	ID       int
	NodeAddr string
	Created  time.Time
	inUse    atomic.Bool
}

func (c *Connection) Execute(query string) string {
	// Simulate query execution
	time.Sleep(time.Duration(rand.Intn(5)) * time.Millisecond)
	return fmt.Sprintf("result from %s (conn %d)", c.NodeAddr, c.ID)
}

func (c *Connection) Close() {
	// Simulate closing
}

// NodePool manages connections to a single node.
type NodePool struct {
	nodeAddr    string
	maxSize     int
	minSize     int
	connections chan *Connection
	allConns    []*Connection
	nextID      atomic.Int32
	created     atomic.Int64
	reused      atomic.Int64
	mu          sync.Mutex
}

func NewNodePool(nodeAddr string, minSize, maxSize int) *NodePool {
	pool := &NodePool{
		nodeAddr:    nodeAddr,
		maxSize:     maxSize,
		minSize:     minSize,
		connections: make(chan *Connection, maxSize),
	}

	// Pre-create minimum connections
	for i := 0; i < minSize; i++ {
		conn := pool.createConnection()
		pool.connections <- conn
	}

	return pool
}

func (np *NodePool) createConnection() *Connection {
	id := int(np.nextID.Add(1))
	conn := &Connection{
		ID:       id,
		NodeAddr: np.nodeAddr,
		Created:  time.Now(),
	}
	np.mu.Lock()
	np.allConns = append(np.allConns, conn)
	np.mu.Unlock()
	np.created.Add(1)
	return conn
}

// Acquire gets a connection from the pool.
func (np *NodePool) Acquire(ctx context.Context) (*Connection, error) {
	// Try to get an existing connection
	select {
	case conn := <-np.connections:
		np.reused.Add(1)
		return conn, nil
	default:
		// No available connection, try to create one
	}

	np.mu.Lock()
	currentSize := len(np.allConns)
	np.mu.Unlock()

	if currentSize < np.maxSize {
		return np.createConnection(), nil
	}

	// Pool is full, wait for a connection to be returned
	select {
	case conn := <-np.connections:
		np.reused.Add(1)
		return conn, nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

// Release returns a connection to the pool.
func (np *NodePool) Release(conn *Connection) {
	select {
	case np.connections <- conn:
		// Returned to pool
	default:
		// Pool is full (shouldn't happen with proper sizing)
		conn.Close()
	}
}

// Stats returns pool statistics.
func (np *NodePool) Stats() (size, available int, created, reused int64) {
	np.mu.Lock()
	size = len(np.allConns)
	np.mu.Unlock()
	available = len(np.connections)
	created = np.created.Load()
	reused = np.reused.Load()
	return
}

// Close closes all connections in the pool.
func (np *NodePool) Close() {
	np.mu.Lock()
	defer np.mu.Unlock()
	for _, conn := range np.allConns {
		conn.Close()
	}
}

// PartitionAwarePool manages connection pools for all nodes in the cluster.
type PartitionAwarePool struct {
	pools         map[string]*NodePool // nodeAddr -> pool
	partitionMap  map[int]string       // partitionID -> nodeAddr
	numPartitions int
	defaultMin    int
	defaultMax    int
	mu            sync.RWMutex
}

func NewPartitionAwarePool(numPartitions, defaultMin, defaultMax int) *PartitionAwarePool {
	return &PartitionAwarePool{
		pools:         make(map[string]*NodePool),
		partitionMap:  make(map[int]string),
		numPartitions: numPartitions,
		defaultMin:    defaultMin,
		defaultMax:    defaultMax,
	}
}

// UpdatePartitionMap updates the mapping of partitions to nodes.
// Creates new pools for new nodes, removes pools for departed nodes.
func (pap *PartitionAwarePool) UpdatePartitionMap(newMap map[int]string) {
	pap.mu.Lock()
	defer pap.mu.Unlock()

	// Find new nodes that need pools
	newNodes := make(map[string]bool)
	for _, addr := range newMap {
		newNodes[addr] = true
		if _, exists := pap.pools[addr]; !exists {
			pap.pools[addr] = NewNodePool(addr, pap.defaultMin, pap.defaultMax)
			fmt.Printf("  Created pool for %s (min=%d, max=%d)\n",
				addr, pap.defaultMin, pap.defaultMax)
		}
	}

	// Close pools for nodes no longer in the map
	for addr, pool := range pap.pools {
		if !newNodes[addr] {
			pool.Close()
			delete(pap.pools, addr)
			fmt.Printf("  Closed pool for %s\n", addr)
		}
	}

	pap.partitionMap = newMap
}

// GetConnection returns a connection to the node owning the given key.
func (pap *PartitionAwarePool) GetConnection(ctx context.Context, key string) (*Connection, string, error) {
	h := fnv.New32a()
	h.Write([]byte(key))
	partitionID := int(h.Sum32()) % pap.numPartitions

	pap.mu.RLock()
	nodeAddr, ok := pap.partitionMap[partitionID]
	if !ok {
		pap.mu.RUnlock()
		return nil, "", fmt.Errorf("no node assigned to partition %d", partitionID)
	}
	pool, ok := pap.pools[nodeAddr]
	pap.mu.RUnlock()

	if !ok {
		return nil, "", fmt.Errorf("no pool for node %s", nodeAddr)
	}

	conn, err := pool.Acquire(ctx)
	if err != nil {
		return nil, "", fmt.Errorf("failed to acquire connection to %s: %w", nodeAddr, err)
	}

	return conn, nodeAddr, nil
}

// ReleaseConnection returns a connection to its pool.
func (pap *PartitionAwarePool) ReleaseConnection(conn *Connection) {
	pap.mu.RLock()
	pool, ok := pap.pools[conn.NodeAddr]
	pap.mu.RUnlock()

	if ok {
		pool.Release(conn)
	}
}

// PrintStats shows connection pool statistics for all nodes.
func (pap *PartitionAwarePool) PrintStats() {
	pap.mu.RLock()
	defer pap.mu.RUnlock()

	fmt.Println("Connection pool statistics:")
	addrs := make([]string, 0, len(pap.pools))
	for addr := range pap.pools {
		addrs = append(addrs, addr)
	}
	sort.Strings(addrs)

	for _, addr := range addrs {
		pool := pap.pools[addr]
		size, avail, created, reused := pool.Stats()
		reuseRate := float64(0)
		if created+reused > 0 {
			reuseRate = float64(reused) / float64(created+reused) * 100
		}
		fmt.Printf("  %-20s: size=%d, available=%d, created=%d, reused=%d (%.1f%% reuse)\n",
			addr, size, avail, created, reused, reuseRate)
	}
}

// Close closes all pools.
func (pap *PartitionAwarePool) Close() {
	pap.mu.Lock()
	defer pap.mu.Unlock()
	for _, pool := range pap.pools {
		pool.Close()
	}
}

func main() {
	const numPartitions = 12

	pool := NewPartitionAwarePool(numPartitions, 2, 10)
	defer pool.Close()

	// Initial partition map: 3 nodes
	fmt.Println("=== Setting up initial partition map (3 nodes) ===")
	pool.UpdatePartitionMap(map[int]string{
		0: "node-a:5432", 1: "node-a:5432", 2: "node-a:5432", 3: "node-a:5432",
		4: "node-b:5432", 5: "node-b:5432", 6: "node-b:5432", 7: "node-b:5432",
		8: "node-c:5432", 9: "node-c:5432", 10: "node-c:5432", 11: "node-c:5432",
	})

	// Simulate workload
	fmt.Println("\n=== Running workload (1000 operations) ===")
	ctx := context.Background()
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			key := fmt.Sprintf("user:%d", rand.Intn(10000))
			conn, nodeAddr, err := pool.GetConnection(ctx, key)
			if err != nil {
				return
			}
			_ = conn.Execute(fmt.Sprintf("SELECT * FROM users WHERE id = '%s'", key))
			_ = nodeAddr
			pool.ReleaseConnection(conn)
		}(i)
	}
	wg.Wait()

	pool.PrintStats()

	// Update partition map: add a 4th node
	fmt.Println("\n=== Adding node-d, updating partition map ===")
	pool.UpdatePartitionMap(map[int]string{
		0: "node-a:5432", 1: "node-a:5432", 2: "node-a:5432",
		3: "node-d:5432",
		4: "node-b:5432", 5: "node-b:5432", 6: "node-b:5432",
		7: "node-d:5432",
		8: "node-c:5432", 9: "node-c:5432", 10: "node-c:5432",
		11: "node-d:5432",
	})

	// Run more workload
	fmt.Println("\n=== Running workload after rebalance (1000 more operations) ===")
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			key := fmt.Sprintf("user:%d", rand.Intn(10000))
			conn, _, err := pool.GetConnection(ctx, key)
			if err != nil {
				return
			}
			_ = conn.Execute("SELECT 1")
			pool.ReleaseConnection(conn)
		}(i)
	}
	wg.Wait()

	pool.PrintStats()
}

// sort is used in PrintStats
func sort_unused() { _ = sort.Strings }
```

---

## 11. Implementing a Sharded Cache

### Overview

A sharded cache distributes cached items across multiple internal shards to reduce lock contention. This is the same principle as partitioning but applied within a single process. Go's built-in `sync.Map` uses a similar approach internally.

This implementation combines consistent hashing for shard selection with per-shard LRU eviction and TTL expiration.

### Complete Implementation

```go
package main

import (
	"container/list"
	"fmt"
	"hash/fnv"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// CacheEntry represents a cached item.
type CacheEntry struct {
	Key       string
	Value     interface{}
	ExpiresAt time.Time
	Size      int64
}

// CacheShard is a single shard of the cache with its own lock and LRU list.
type CacheShard struct {
	items    map[string]*list.Element
	lru      *list.List
	maxItems int
	mu       sync.RWMutex
	hits     atomic.Int64
	misses   atomic.Int64
	evictions atomic.Int64
}

func NewCacheShard(maxItems int) *CacheShard {
	return &CacheShard{
		items:    make(map[string]*list.Element),
		lru:      list.New(),
		maxItems: maxItems,
	}
}

func (s *CacheShard) Get(key string) (interface{}, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	elem, ok := s.items[key]
	if !ok {
		s.misses.Add(1)
		return nil, false
	}

	entry := elem.Value.(*CacheEntry)
	if time.Now().After(entry.ExpiresAt) {
		// Expired -- remove it
		s.lru.Remove(elem)
		delete(s.items, key)
		s.misses.Add(1)
		return nil, false
	}

	// Move to front (most recently used)
	s.lru.MoveToFront(elem)
	s.hits.Add(1)
	return entry.Value, true
}

func (s *CacheShard) Set(key string, value interface{}, ttl time.Duration) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// If key exists, update it
	if elem, ok := s.items[key]; ok {
		entry := elem.Value.(*CacheEntry)
		entry.Value = value
		entry.ExpiresAt = time.Now().Add(ttl)
		s.lru.MoveToFront(elem)
		return
	}

	// Evict if at capacity
	for s.lru.Len() >= s.maxItems {
		s.evict()
	}

	// Add new entry
	entry := &CacheEntry{
		Key:       key,
		Value:     value,
		ExpiresAt: time.Now().Add(ttl),
	}
	elem := s.lru.PushFront(entry)
	s.items[key] = elem
}

func (s *CacheShard) Delete(key string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	elem, ok := s.items[key]
	if !ok {
		return false
	}

	s.lru.Remove(elem)
	delete(s.items, key)
	return true
}

// evict removes the least recently used item. Must be called with lock held.
func (s *CacheShard) evict() {
	back := s.lru.Back()
	if back == nil {
		return
	}
	entry := back.Value.(*CacheEntry)
	s.lru.Remove(back)
	delete(s.items, entry.Key)
	s.evictions.Add(1)
}

// CleanExpired removes all expired entries from the shard.
func (s *CacheShard) CleanExpired() int {
	s.mu.Lock()
	defer s.mu.Unlock()

	now := time.Now()
	removed := 0
	var next *list.Element
	for elem := s.lru.Back(); elem != nil; elem = next {
		next = elem.Prev()
		entry := elem.Value.(*CacheEntry)
		if now.After(entry.ExpiresAt) {
			s.lru.Remove(elem)
			delete(s.items, entry.Key)
			removed++
		}
	}
	return removed
}

func (s *CacheShard) Len() int {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return len(s.items)
}

// ShardedCache is a cache distributed across multiple shards.
type ShardedCache struct {
	shards    []*CacheShard
	numShards int
	stopClean chan struct{}
}

// NewShardedCache creates a sharded cache.
// numShards: number of internal shards (use a power of 2 for fast modulo).
// maxItemsPerShard: maximum items per shard before LRU eviction.
func NewShardedCache(numShards, maxItemsPerShard int) *ShardedCache {
	shards := make([]*CacheShard, numShards)
	for i := range shards {
		shards[i] = NewCacheShard(maxItemsPerShard)
	}

	sc := &ShardedCache{
		shards:    shards,
		numShards: numShards,
		stopClean: make(chan struct{}),
	}

	// Start background cleanup goroutine
	go sc.cleanupLoop()

	return sc
}

// getShard returns the shard for a given key.
func (sc *ShardedCache) getShard(key string) *CacheShard {
	h := fnv.New32a()
	h.Write([]byte(key))
	return sc.shards[int(h.Sum32())%sc.numShards]
}

// Get retrieves a value from the cache.
func (sc *ShardedCache) Get(key string) (interface{}, bool) {
	return sc.getShard(key).Get(key)
}

// Set stores a value in the cache with a TTL.
func (sc *ShardedCache) Set(key string, value interface{}, ttl time.Duration) {
	sc.getShard(key).Set(key, value, ttl)
}

// Delete removes a value from the cache.
func (sc *ShardedCache) Delete(key string) bool {
	return sc.getShard(key).Delete(key)
}

// Len returns the total number of items across all shards.
func (sc *ShardedCache) Len() int {
	total := 0
	for _, s := range sc.shards {
		total += s.Len()
	}
	return total
}

// Stats returns cache statistics.
func (sc *ShardedCache) Stats() CacheStats {
	var stats CacheStats
	for i, s := range sc.shards {
		hits := s.hits.Load()
		misses := s.misses.Load()
		evictions := s.evictions.Load()
		size := s.Len()

		stats.TotalHits += hits
		stats.TotalMisses += misses
		stats.TotalEvictions += evictions
		stats.TotalItems += int64(size)
		stats.ShardStats = append(stats.ShardStats, ShardStat{
			ID:        i,
			Items:     size,
			Hits:      hits,
			Misses:    misses,
			Evictions: evictions,
		})
	}
	if stats.TotalHits+stats.TotalMisses > 0 {
		stats.HitRate = float64(stats.TotalHits) /
			float64(stats.TotalHits+stats.TotalMisses) * 100
	}
	return stats
}

// cleanupLoop periodically removes expired entries.
func (sc *ShardedCache) cleanupLoop() {
	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			for _, s := range sc.shards {
				s.CleanExpired()
			}
		case <-sc.stopClean:
			return
		}
	}
}

// Close stops the background cleanup goroutine.
func (sc *ShardedCache) Close() {
	close(sc.stopClean)
}

type CacheStats struct {
	TotalHits      int64
	TotalMisses    int64
	TotalEvictions int64
	TotalItems     int64
	HitRate        float64
	ShardStats     []ShardStat
}

type ShardStat struct {
	ID        int
	Items     int
	Hits      int64
	Misses    int64
	Evictions int64
}

func main() {
	// Create a sharded cache with 16 shards, 1000 items per shard
	cache := NewShardedCache(16, 1000)
	defer cache.Close()

	// === Test 1: Basic operations ===
	fmt.Println("=== Test 1: Basic Operations ===")
	cache.Set("user:1", map[string]string{"name": "Alice", "city": "NYC"}, 5*time.Minute)
	cache.Set("user:2", map[string]string{"name": "Bob", "city": "LA"}, 5*time.Minute)
	cache.Set("user:3", map[string]string{"name": "Charlie", "city": "Chicago"}, 5*time.Minute)

	if val, ok := cache.Get("user:1"); ok {
		fmt.Printf("  user:1 = %v\n", val)
	}
	if _, ok := cache.Get("user:999"); !ok {
		fmt.Println("  user:999 = not found (expected)")
	}

	// === Test 2: TTL expiration ===
	fmt.Println("\n=== Test 2: TTL Expiration ===")
	cache.Set("temp:1", "temporary data", 100*time.Millisecond)
	if val, ok := cache.Get("temp:1"); ok {
		fmt.Printf("  Before expiry: %v\n", val)
	}
	time.Sleep(150 * time.Millisecond)
	if _, ok := cache.Get("temp:1"); !ok {
		fmt.Println("  After expiry: not found (expired)")
	}

	// === Test 3: LRU eviction ===
	fmt.Println("\n=== Test 3: LRU Eviction ===")
	smallCache := NewShardedCache(1, 5) // 1 shard, max 5 items
	defer smallCache.Close()

	for i := 0; i < 10; i++ {
		smallCache.Set(fmt.Sprintf("key:%d", i), i, time.Hour)
	}
	fmt.Printf("  Inserted 10 items into cache with max 5: actual size = %d\n", smallCache.Len())

	// Early keys should be evicted, later ones should remain
	for i := 0; i < 10; i++ {
		_, ok := smallCache.Get(fmt.Sprintf("key:%d", i))
		if ok {
			fmt.Printf("  key:%d = found\n", i)
		}
	}

	// === Test 4: Concurrent access ===
	fmt.Println("\n=== Test 4: Concurrent Access (100k ops across 50 goroutines) ===")
	concCache := NewShardedCache(16, 5000)
	defer concCache.Close()

	// Pre-populate
	for i := 0; i < 10000; i++ {
		concCache.Set(fmt.Sprintf("item:%d", i), i, 5*time.Minute)
	}

	start := time.Now()
	var wg sync.WaitGroup
	opsPerGoroutine := 2000

	for g := 0; g < 50; g++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			rng := rand.New(rand.NewSource(time.Now().UnixNano()))
			for i := 0; i < opsPerGoroutine; i++ {
				key := fmt.Sprintf("item:%d", rng.Intn(15000))
				if rng.Float64() < 0.8 {
					// 80% reads
					concCache.Get(key)
				} else {
					// 20% writes
					concCache.Set(key, rng.Intn(1000), 5*time.Minute)
				}
			}
		}()
	}
	wg.Wait()
	elapsed := time.Since(start)

	totalOps := 50 * opsPerGoroutine
	fmt.Printf("  %d operations in %v (%.0f ops/sec)\n",
		totalOps, elapsed, float64(totalOps)/elapsed.Seconds())

	stats := concCache.Stats()
	fmt.Printf("  Total items: %d\n", stats.TotalItems)
	fmt.Printf("  Hits: %d, Misses: %d, Hit rate: %.1f%%\n",
		stats.TotalHits, stats.TotalMisses, stats.HitRate)
	fmt.Printf("  Evictions: %d\n", stats.TotalEvictions)

	// === Test 5: Shard distribution ===
	fmt.Println("\n=== Test 5: Shard Distribution ===")
	distCache := NewShardedCache(8, 10000)
	defer distCache.Close()

	for i := 0; i < 10000; i++ {
		distCache.Set(fmt.Sprintf("k:%d", i), i, time.Hour)
	}

	distStats := distCache.Stats()
	fmt.Println("Items per shard:")
	for _, ss := range distStats.ShardStats {
		bar := ""
		for j := 0; j < ss.Items/50; j++ {
			bar += "#"
		}
		fmt.Printf("  Shard %2d: %4d items [%s]\n", ss.ID, ss.Items, bar)
	}
}
```

---

## 12. Real-World Example: Building a Sharded Key-Value Store

### Overview

This is the capstone section. We build a complete, functioning sharded key-value store that combines everything from this chapter:

- **Consistent hashing** for partition assignment
- **Virtual nodes** for uniform distribution
- **Request routing** with a routing tier
- **Scatter-gather queries** for secondary index lookups
- **Rebalancing** when nodes join or leave
- **Connection pooling** per partition node
- **Hot spot detection** and mitigation

The system simulates a distributed environment using goroutines and channels. Each "node" runs in its own goroutine and communicates via channels (simulating network calls).

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"hash/fnv"
	"math"
	"math/rand"
	"sort"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// ============================================================
// Data Types
// ============================================================

// KVPair is a key-value pair with metadata.
type KVPair struct {
	Key       string
	Value     string
	Version   int64
	Timestamp time.Time
}

// Request represents an operation to be performed on the store.
type Request struct {
	Type     string // "get", "put", "delete", "scan"
	Key      string
	Value    string
	ScanFrom string
	ScanTo   string
	RespCh   chan Response
}

// Response is the result of a request.
type Response struct {
	Value   string
	Pairs   []KVPair
	Found   bool
	Error   error
	NodeID  string
}

// ============================================================
// Storage Node
// ============================================================

// StorageNode runs as an independent goroutine, processing requests via a channel.
type StorageNode struct {
	ID       string
	data     map[string]*KVPair
	reqCh    chan Request
	stopCh   chan struct{}
	mu       sync.RWMutex
	getOps   atomic.Int64
	putOps   atomic.Int64
	dataSize atomic.Int64
}

func NewStorageNode(id string) *StorageNode {
	node := &StorageNode{
		ID:     id,
		data:   make(map[string]*KVPair),
		reqCh:  make(chan Request, 1000),
		stopCh: make(chan struct{}),
	}
	go node.run()
	return node
}

func (n *StorageNode) run() {
	for {
		select {
		case req := <-n.reqCh:
			n.handleRequest(req)
		case <-n.stopCh:
			return
		}
	}
}

func (n *StorageNode) handleRequest(req Request) {
	switch req.Type {
	case "get":
		n.mu.RLock()
		pair, ok := n.data[req.Key]
		n.mu.RUnlock()
		n.getOps.Add(1)
		if ok {
			req.RespCh <- Response{Value: pair.Value, Found: true, NodeID: n.ID}
		} else {
			req.RespCh <- Response{Found: false, NodeID: n.ID}
		}

	case "put":
		n.mu.Lock()
		existing, exists := n.data[req.Key]
		var version int64 = 1
		if exists {
			version = existing.Version + 1
		}
		n.data[req.Key] = &KVPair{
			Key:       req.Key,
			Value:     req.Value,
			Version:   version,
			Timestamp: time.Now(),
		}
		n.mu.Unlock()
		n.putOps.Add(1)
		n.dataSize.Add(int64(len(req.Key) + len(req.Value)))
		req.RespCh <- Response{Found: true, NodeID: n.ID}

	case "delete":
		n.mu.Lock()
		_, ok := n.data[req.Key]
		if ok {
			delete(n.data, req.Key)
		}
		n.mu.Unlock()
		req.RespCh <- Response{Found: ok, NodeID: n.ID}

	case "scan":
		n.mu.RLock()
		var pairs []KVPair
		for _, pair := range n.data {
			if pair.Key >= req.ScanFrom && pair.Key < req.ScanTo {
				pairs = append(pairs, *pair)
			}
		}
		n.mu.RUnlock()
		sort.Slice(pairs, func(i, j int) bool {
			return pairs[i].Key < pairs[j].Key
		})
		req.RespCh <- Response{Pairs: pairs, Found: true, NodeID: n.ID}

	default:
		req.RespCh <- Response{Error: fmt.Errorf("unknown request type: %s", req.Type)}
	}
}

// GetAllData returns a copy of all data on this node (for rebalancing).
func (n *StorageNode) GetAllData() map[string]*KVPair {
	n.mu.RLock()
	defer n.mu.RUnlock()
	copy := make(map[string]*KVPair, len(n.data))
	for k, v := range n.data {
		copy[k] = v
	}
	return copy
}

// RemoveKeys removes the specified keys from this node (after rebalancing).
func (n *StorageNode) RemoveKeys(keys []string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	for _, k := range keys {
		delete(n.data, k)
	}
}

func (n *StorageNode) KeyCount() int {
	n.mu.RLock()
	defer n.mu.RUnlock()
	return len(n.data)
}

func (n *StorageNode) Stop() {
	close(n.stopCh)
}

// ============================================================
// Consistent Hash Ring
// ============================================================

type HashRing struct {
	ring     map[uint32]string
	sorted   []uint32
	replicas int
	nodes    map[string]bool
	mu       sync.RWMutex
}

func NewHashRing(replicas int) *HashRing {
	return &HashRing{
		ring:     make(map[uint32]string),
		replicas: replicas,
		nodes:    make(map[string]bool),
	}
}

func (hr *HashRing) hashPos(key string, replica int) uint32 {
	h := fnv.New32a()
	h.Write([]byte(fmt.Sprintf("%s#%d", key, replica)))
	return h.Sum32()
}

func (hr *HashRing) AddNode(nodeID string) {
	hr.mu.Lock()
	defer hr.mu.Unlock()

	if hr.nodes[nodeID] {
		return
	}
	hr.nodes[nodeID] = true
	for i := 0; i < hr.replicas; i++ {
		pos := hr.hashPos(nodeID, i)
		hr.ring[pos] = nodeID
		hr.sorted = append(hr.sorted, pos)
	}
	sort.Slice(hr.sorted, func(i, j int) bool {
		return hr.sorted[i] < hr.sorted[j]
	})
}

func (hr *HashRing) RemoveNode(nodeID string) {
	hr.mu.Lock()
	defer hr.mu.Unlock()

	if !hr.nodes[nodeID] {
		return
	}
	delete(hr.nodes, nodeID)
	for i := 0; i < hr.replicas; i++ {
		pos := hr.hashPos(nodeID, i)
		delete(hr.ring, pos)
	}
	hr.sorted = hr.sorted[:0]
	for pos := range hr.ring {
		hr.sorted = append(hr.sorted, pos)
	}
	sort.Slice(hr.sorted, func(i, j int) bool {
		return hr.sorted[i] < hr.sorted[j]
	})
}

func (hr *HashRing) GetNode(key string) string {
	hr.mu.RLock()
	defer hr.mu.RUnlock()

	if len(hr.sorted) == 0 {
		return ""
	}
	h := fnv.New32a()
	h.Write([]byte(key))
	keyHash := h.Sum32()

	idx := sort.Search(len(hr.sorted), func(i int) bool {
		return hr.sorted[i] >= keyHash
	})
	if idx >= len(hr.sorted) {
		idx = 0
	}
	return hr.ring[hr.sorted[idx]]
}

func (hr *HashRing) GetNodeList() []string {
	hr.mu.RLock()
	defer hr.mu.RUnlock()
	list := make([]string, 0, len(hr.nodes))
	for id := range hr.nodes {
		list = append(list, id)
	}
	sort.Strings(list)
	return list
}

// ============================================================
// Sharded Key-Value Store (the main cluster)
// ============================================================

// ShardedKVStore is the main entry point for the sharded key-value store.
type ShardedKVStore struct {
	ring       *HashRing
	nodes      map[string]*StorageNode
	hotKeys    map[string]bool
	detector   *LoadDetector
	mu         sync.RWMutex
	totalGets  atomic.Int64
	totalPuts  atomic.Int64
	totalScans atomic.Int64
}

// LoadDetector tracks per-key access frequency to detect hot spots.
type LoadDetector struct {
	keyCounts map[string]*atomic.Int64
	mu        sync.RWMutex
	threshold int64
}

func NewLoadDetector(threshold int64) *LoadDetector {
	return &LoadDetector{
		keyCounts: make(map[string]*atomic.Int64),
		threshold: threshold,
	}
}

func (ld *LoadDetector) Record(key string) {
	ld.mu.RLock()
	counter, ok := ld.keyCounts[key]
	ld.mu.RUnlock()

	if !ok {
		ld.mu.Lock()
		counter, ok = ld.keyCounts[key]
		if !ok {
			counter = &atomic.Int64{}
			ld.keyCounts[key] = counter
		}
		ld.mu.Unlock()
	}
	counter.Add(1)
}

func (ld *LoadDetector) GetHotKeys() []string {
	ld.mu.RLock()
	defer ld.mu.RUnlock()

	var hot []string
	for key, counter := range ld.keyCounts {
		if counter.Load() >= ld.threshold {
			hot = append(hot, key)
		}
	}
	sort.Strings(hot)
	return hot
}

func (ld *LoadDetector) Reset() {
	ld.mu.Lock()
	defer ld.mu.Unlock()
	ld.keyCounts = make(map[string]*atomic.Int64)
}

// NewShardedKVStore creates a new sharded key-value store.
func NewShardedKVStore(vnodes int, hotKeyThreshold int64) *ShardedKVStore {
	return &ShardedKVStore{
		ring:     NewHashRing(vnodes),
		nodes:    make(map[string]*StorageNode),
		hotKeys:  make(map[string]bool),
		detector: NewLoadDetector(hotKeyThreshold),
	}
}

// AddNode adds a new storage node to the cluster.
func (s *ShardedKVStore) AddNode(nodeID string) {
	s.mu.Lock()
	defer s.mu.Unlock()

	node := NewStorageNode(nodeID)
	s.nodes[nodeID] = node
	s.ring.AddNode(nodeID)
}

// RemoveNode removes a node and redistributes its data.
func (s *ShardedKVStore) RemoveNode(nodeID string) int {
	s.mu.Lock()
	node, ok := s.nodes[nodeID]
	if !ok {
		s.mu.Unlock()
		return 0
	}

	// Get all data from the departing node
	allData := node.GetAllData()
	node.Stop()
	delete(s.nodes, nodeID)
	s.ring.RemoveNode(nodeID)
	s.mu.Unlock()

	// Redistribute data to remaining nodes
	movedCount := 0
	for key, pair := range allData {
		targetNodeID := s.ring.GetNode(key)
		s.mu.RLock()
		targetNode, ok := s.nodes[targetNodeID]
		s.mu.RUnlock()
		if ok {
			resp := make(chan Response, 1)
			targetNode.reqCh <- Request{
				Type:   "put",
				Key:    key,
				Value:  pair.Value,
				RespCh: resp,
			}
			<-resp
			movedCount++
		}
	}
	return movedCount
}

// RebalanceOnJoin redistributes keys that now belong to a new node.
func (s *ShardedKVStore) RebalanceOnJoin(newNodeID string) int {
	s.mu.RLock()
	newNode, ok := s.nodes[newNodeID]
	if !ok {
		s.mu.RUnlock()
		return 0
	}

	// Check all other nodes' data to see if any keys now belong to the new node
	var keysToMove []struct {
		key      string
		value    string
		fromNode string
	}

	for nodeID, node := range s.nodes {
		if nodeID == newNodeID {
			continue
		}
		allData := node.GetAllData()
		for key, pair := range allData {
			if s.ring.GetNode(key) == newNodeID {
				keysToMove = append(keysToMove, struct {
					key      string
					value    string
					fromNode string
				}{key: key, value: pair.Value, fromNode: nodeID})
			}
		}
	}
	s.mu.RUnlock()

	// Move keys
	for _, item := range keysToMove {
		resp := make(chan Response, 1)
		newNode.reqCh <- Request{
			Type:   "put",
			Key:    item.key,
			Value:  item.value,
			RespCh: resp,
		}
		<-resp

		// Remove from old node
		s.mu.RLock()
		if oldNode, ok := s.nodes[item.fromNode]; ok {
			oldNode.RemoveKeys([]string{item.key})
		}
		s.mu.RUnlock()
	}

	return len(keysToMove)
}

// Put stores a key-value pair.
func (s *ShardedKVStore) Put(ctx context.Context, key, value string) error {
	s.detector.Record(key)
	s.totalPuts.Add(1)

	nodeID := s.ring.GetNode(key)
	s.mu.RLock()
	node, ok := s.nodes[nodeID]
	s.mu.RUnlock()

	if !ok {
		return fmt.Errorf("node %s not found", nodeID)
	}

	resp := make(chan Response, 1)
	select {
	case node.reqCh <- Request{Type: "put", Key: key, Value: value, RespCh: resp}:
	case <-ctx.Done():
		return ctx.Err()
	}

	select {
	case <-resp:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

// Get retrieves a value by key.
func (s *ShardedKVStore) Get(ctx context.Context, key string) (string, bool, error) {
	s.detector.Record(key)
	s.totalGets.Add(1)

	nodeID := s.ring.GetNode(key)
	s.mu.RLock()
	node, ok := s.nodes[nodeID]
	s.mu.RUnlock()

	if !ok {
		return "", false, fmt.Errorf("node %s not found", nodeID)
	}

	resp := make(chan Response, 1)
	select {
	case node.reqCh <- Request{Type: "get", Key: key, RespCh: resp}:
	case <-ctx.Done():
		return "", false, ctx.Err()
	}

	select {
	case r := <-resp:
		return r.Value, r.Found, r.Error
	case <-ctx.Done():
		return "", false, ctx.Err()
	}
}

// Delete removes a key.
func (s *ShardedKVStore) Delete(ctx context.Context, key string) (bool, error) {
	nodeID := s.ring.GetNode(key)
	s.mu.RLock()
	node, ok := s.nodes[nodeID]
	s.mu.RUnlock()

	if !ok {
		return false, fmt.Errorf("node %s not found", nodeID)
	}

	resp := make(chan Response, 1)
	select {
	case node.reqCh <- Request{Type: "delete", Key: key, RespCh: resp}:
	case <-ctx.Done():
		return false, ctx.Err()
	}

	select {
	case r := <-resp:
		return r.Found, r.Error
	case <-ctx.Done():
		return false, ctx.Err()
	}
}

// Scan performs a scatter-gather range scan across all nodes.
func (s *ShardedKVStore) Scan(ctx context.Context, fromKey, toKey string) ([]KVPair, error) {
	s.totalScans.Add(1)
	s.mu.RLock()
	nodeList := make([]*StorageNode, 0, len(s.nodes))
	for _, node := range s.nodes {
		nodeList = append(nodeList, node)
	}
	s.mu.RUnlock()

	type scanResult struct {
		pairs []KVPair
		err   error
	}

	results := make(chan scanResult, len(nodeList))
	for _, node := range nodeList {
		go func(n *StorageNode) {
			resp := make(chan Response, 1)
			select {
			case n.reqCh <- Request{
				Type:     "scan",
				ScanFrom: fromKey,
				ScanTo:   toKey,
				RespCh:   resp,
			}:
			case <-ctx.Done():
				results <- scanResult{err: ctx.Err()}
				return
			}

			select {
			case r := <-resp:
				results <- scanResult{pairs: r.Pairs, err: r.Error}
			case <-ctx.Done():
				results <- scanResult{err: ctx.Err()}
			}
		}(node)
	}

	// Gather results
	var allPairs []KVPair
	for i := 0; i < len(nodeList); i++ {
		select {
		case r := <-results:
			if r.err != nil {
				return nil, r.err
			}
			allPairs = append(allPairs, r.pairs...)
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}

	// Merge and sort
	sort.Slice(allPairs, func(i, j int) bool {
		return allPairs[i].Key < allPairs[j].Key
	})
	return allPairs, nil
}

// PrintClusterStatus shows the state of the cluster.
func (s *ShardedKVStore) PrintClusterStatus() {
	s.mu.RLock()
	defer s.mu.RUnlock()

	totalKeys := 0
	var nodeStatuses []struct {
		id    string
		keys  int
		gets  int64
		puts  int64
	}

	nodeIDs := make([]string, 0, len(s.nodes))
	for id := range s.nodes {
		nodeIDs = append(nodeIDs, id)
	}
	sort.Strings(nodeIDs)

	for _, id := range nodeIDs {
		node := s.nodes[id]
		keys := node.KeyCount()
		totalKeys += keys
		nodeStatuses = append(nodeStatuses, struct {
			id    string
			keys  int
			gets  int64
			puts  int64
		}{id: id, keys: keys, gets: node.getOps.Load(), puts: node.putOps.Load()})
	}

	fmt.Printf("\nCluster Status (%d nodes, %d total keys)\n", len(s.nodes), totalKeys)
	fmt.Println(strings.Repeat("-", 65))
	fmt.Printf("%-12s %8s %8s %8s %s\n", "Node", "Keys", "Gets", "Puts", "Distribution")
	fmt.Println(strings.Repeat("-", 65))

	idealKeys := float64(totalKeys) / float64(len(s.nodes))
	for _, ns := range nodeStatuses {
		bar := ""
		if totalKeys > 0 {
			barLen := int(float64(ns.keys) / float64(totalKeys) * 30)
			bar = strings.Repeat("#", barLen)
		}
		skew := ""
		if idealKeys > 0 {
			skewPct := (float64(ns.keys)/idealKeys - 1.0) * 100
			skew = fmt.Sprintf(" (%+.1f%%)", skewPct)
		}
		fmt.Printf("%-12s %8d %8d %8d [%-30s]%s\n",
			ns.id, ns.keys, ns.gets, ns.puts, bar, skew)
	}
	fmt.Println(strings.Repeat("-", 65))
	fmt.Printf("Total ops: gets=%d, puts=%d, scans=%d\n",
		s.totalGets.Load(), s.totalPuts.Load(), s.totalScans.Load())

	// Hot key detection
	hotKeys := s.detector.GetHotKeys()
	if len(hotKeys) > 0 {
		fmt.Printf("Hot keys detected: %v\n", hotKeys)
	}
}

// Stop shuts down all nodes.
func (s *ShardedKVStore) Stop() {
	s.mu.Lock()
	defer s.mu.Unlock()
	for _, node := range s.nodes {
		node.Stop()
	}
}

// ============================================================
// Main: Full demonstration
// ============================================================

func main() {
	fmt.Println("========================================================")
	fmt.Println("  Sharded Key-Value Store - Complete Demonstration")
	fmt.Println("========================================================")

	// Create the store with 150 vnodes and hot key threshold of 100
	store := NewShardedKVStore(150, 100)
	defer store.Stop()
	ctx := context.Background()

	// --- Phase 1: Create initial cluster ---
	fmt.Println("\n--- Phase 1: Creating 3-node cluster ---")
	store.AddNode("node-alpha")
	store.AddNode("node-beta")
	store.AddNode("node-gamma")

	// Insert 5000 keys
	fmt.Println("Inserting 5000 key-value pairs...")
	for i := 0; i < 5000; i++ {
		key := fmt.Sprintf("user:%05d", i)
		value := fmt.Sprintf(`{"id":%d,"name":"User %d","score":%.2f}`, i, i, rand.Float64()*100)
		if err := store.Put(ctx, key, value); err != nil {
			fmt.Printf("Error putting %s: %v\n", key, err)
		}
	}
	store.PrintClusterStatus()

	// --- Phase 2: Point queries ---
	fmt.Println("\n--- Phase 2: Point Queries ---")
	for _, key := range []string{"user:00001", "user:02500", "user:04999", "user:99999"} {
		value, found, err := store.Get(ctx, key)
		if err != nil {
			fmt.Printf("  Get(%s): error=%v\n", key, err)
		} else if found {
			// Truncate long values for display
			display := value
			if len(display) > 50 {
				display = display[:50] + "..."
			}
			fmt.Printf("  Get(%s): %s\n", key, display)
		} else {
			fmt.Printf("  Get(%s): not found\n", key)
		}
	}

	// --- Phase 3: Scatter-gather scan ---
	fmt.Println("\n--- Phase 3: Scatter-Gather Scan ---")
	start := time.Now()
	pairs, err := store.Scan(ctx, "user:01000", "user:01010")
	if err != nil {
		fmt.Printf("Scan error: %v\n", err)
	} else {
		fmt.Printf("Scan [user:01000, user:01010) returned %d results in %v:\n",
			len(pairs), time.Since(start))
		for _, p := range pairs {
			display := p.Value
			if len(display) > 50 {
				display = display[:50] + "..."
			}
			fmt.Printf("  %s = %s\n", p.Key, display)
		}
	}

	// --- Phase 4: Add a new node and rebalance ---
	fmt.Println("\n--- Phase 4: Adding node-delta and Rebalancing ---")
	store.AddNode("node-delta")
	moved := store.RebalanceOnJoin("node-delta")
	fmt.Printf("Rebalanced: moved %d keys to node-delta\n", moved)
	store.PrintClusterStatus()

	// Verify all data is still accessible
	fmt.Println("\nVerifying data integrity after rebalance...")
	missing := 0
	for i := 0; i < 5000; i++ {
		key := fmt.Sprintf("user:%05d", i)
		_, found, err := store.Get(ctx, key)
		if err != nil || !found {
			missing++
		}
	}
	fmt.Printf("Data integrity check: %d missing out of 5000 (%d%% intact)\n",
		missing, (5000-missing)*100/5000)

	// --- Phase 5: Simulate hot key access ---
	fmt.Println("\n--- Phase 5: Hot Key Detection ---")
	store.detector.Reset()
	fmt.Println("Simulating skewed access pattern (celebrity key)...")
	for i := 0; i < 1000; i++ {
		// 80% of reads go to one key
		if rand.Float64() < 0.8 {
			store.Get(ctx, "user:00042") // celebrity user
		} else {
			store.Get(ctx, fmt.Sprintf("user:%05d", rand.Intn(5000)))
		}
	}
	hotKeys := store.detector.GetHotKeys()
	fmt.Printf("Hot keys detected: %v\n", hotKeys)

	// --- Phase 6: Remove a node ---
	fmt.Println("\n--- Phase 6: Removing node-beta (simulating failure) ---")
	moved = store.RemoveNode("node-beta")
	fmt.Printf("Redistributed %d keys from node-beta\n", moved)
	store.PrintClusterStatus()

	// Verify data integrity again
	fmt.Println("\nVerifying data integrity after node removal...")
	missing = 0
	for i := 0; i < 5000; i++ {
		key := fmt.Sprintf("user:%05d", i)
		_, found, err := store.Get(ctx, key)
		if err != nil || !found {
			missing++
		}
	}
	fmt.Printf("Data integrity check: %d missing out of 5000 (%d%% intact)\n",
		missing, (5000-missing)*100/5000)

	// --- Phase 7: Concurrent workload ---
	fmt.Println("\n--- Phase 7: Concurrent Workload (50 goroutines, 500 ops each) ---")
	store.detector.Reset()
	var wg sync.WaitGroup
	startTime := time.Now()

	for g := 0; g < 50; g++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			rng := rand.New(rand.NewSource(time.Now().UnixNano()))
			for i := 0; i < 500; i++ {
				key := fmt.Sprintf("user:%05d", rng.Intn(5000))
				if rng.Float64() < 0.7 {
					store.Get(ctx, key)
				} else {
					store.Put(ctx, key, fmt.Sprintf("updated-%d", i))
				}
			}
		}()
	}
	wg.Wait()
	elapsed := time.Since(startTime)

	totalOps := 50 * 500
	fmt.Printf("Completed %d operations in %v (%.0f ops/sec)\n",
		totalOps, elapsed, float64(totalOps)/elapsed.Seconds())
	store.PrintClusterStatus()

	// --- Phase 8: Delete operations ---
	fmt.Println("\n--- Phase 8: Delete Operations ---")
	deleteCount := 0
	for i := 0; i < 100; i++ {
		key := fmt.Sprintf("user:%05d", i)
		deleted, err := store.Delete(ctx, key)
		if err == nil && deleted {
			deleteCount++
		}
	}
	fmt.Printf("Deleted %d keys\n", deleteCount)

	// Verify they are gone
	for _, key := range []string{"user:00000", "user:00050", "user:00099"} {
		_, found, _ := store.Get(ctx, key)
		fmt.Printf("  After delete, Get(%s): found=%v\n", key, found)
	}

	// Final status
	fmt.Println("\n--- Final Cluster Status ---")
	store.PrintClusterStatus()

	fmt.Println("\n========================================================")
	fmt.Println("  Demonstration Complete")
	fmt.Println("========================================================")
	_ = math.Sqrt(0)
}
```

### Architecture Recap

The sharded key-value store we built demonstrates every concept from this chapter:

```
┌─────────────────────────────────────────────────────────────────┐
│                       ShardedKVStore                            │
│                                                                 │
│  ┌──────────────────────┐    ┌───────────────────────────────┐ │
│  │    Consistent Hash   │    │      Hot Spot Detector        │ │
│  │    Ring (150 vnodes) │    │   (per-key access tracking)   │ │
│  └──────────┬───────────┘    └───────────────────────────────┘ │
│             │                                                   │
│  ┌──────────▼───────────────────────────────────────────┐      │
│  │                   Request Router                      │      │
│  │  key -> hash -> ring position -> node ID -> channel   │      │
│  └───┬──────────┬──────────┬──────────┬─────────────────┘      │
│      │          │          │          │                          │
│  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐                      │
│  │Node A│  │Node B│  │Node C│  │Node D│  ← goroutines         │
│  │reqCh │  │reqCh │  │reqCh │  │reqCh │  ← channel-based I/O  │
│  │data{}│  │data{}│  │data{}│  │data{}│  ← local data map     │
│  └──────┘  └──────┘  └──────┘  └──────┘                       │
│                                                                 │
│  Scatter-Gather: Scan sends to ALL nodes in parallel            │
│  Rebalancing: Move keys when nodes join/leave                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. Key Takeaways

### Fundamental Principles

1. **Partitioning is about splitting data across nodes** to achieve horizontal scaling. It is orthogonal to replication (which copies data for fault tolerance). Real systems combine both.

2. **Key-range partitioning** preserves sort order and enables efficient range queries, but is vulnerable to hot spots when access patterns are skewed toward certain key ranges.

3. **Hash partitioning** distributes keys uniformly regardless of their lexicographic order, eliminating access-pattern skew, but destroys sort order and forces scatter-gather for range queries.

4. **Consistent hashing** solves the redistribution problem of `hash mod N`. When a node joins or leaves, only `~1/N` of keys need to move. Virtual nodes (vnodes) improve distribution uniformity.

5. **Hot spots** can occur even with perfect hash distribution when individual keys receive disproportionate traffic (the celebrity problem). Mitigations include key salting, caching, and partition splitting.

6. **Secondary indexes** in a partitioned system have two approaches:
   - **Local (document-partitioned)**: Each partition indexes its own data. Reads require scatter-gather. Writes are local. Used by MongoDB, Elasticsearch.
   - **Global (term-partitioned)**: The index itself is partitioned. Reads are efficient. Writes require cross-partition updates. Used by DynamoDB.

7. **Rebalancing** must minimize data movement while achieving good balance. The three main strategies are fixed partitions, dynamic splitting, and proportional to node count.

8. **Request routing** can be done at the client (smart clients), at a routing tier (proxy), or by any node (gossip-based forwarding). Each has different latency and complexity trade-offs.

9. **Scatter-gather queries** are the cost of hash partitioning. Total latency is bounded by the slowest partition. Mitigation strategies include hedged requests and partial results with timeouts.

10. **Partition-aware connection pooling** maintains separate pools per node, sized according to the expected traffic to each partition, reducing wasted connections and improving resource utilization.

### Go-Specific Patterns for Partitioning

| Pattern | Go Implementation |
|---------|------------------|
| Node as goroutine | Each storage node runs in a goroutine, receiving requests via a channel |
| Request routing | Hash the key, look up in consistent hash ring, send to the right channel |
| Scatter-gather | Launch goroutines for each partition, collect via buffered channel or `sync.WaitGroup` |
| Rebalancing | Iterate data maps, re-hash keys against new ring, move via channel sends |
| Concurrent access | `sync.RWMutex` for data maps, `atomic` for counters, channels for inter-node communication |
| Timeouts | `context.WithTimeout` for all operations, `select` on `ctx.Done()` |

### DDIA Chapter 6 Concepts Mapped to Go

| DDIA Concept | Section | Go Implementation |
|-------------|---------|------------------|
| Key-range partitioning | Section 2 | `RangePartitioner` with `sort.SearchStrings` |
| Hash partitioning | Section 3 | `HashPartitioner` with `hash/fnv` |
| Consistent hashing with vnodes | Section 4 | `ConsistentHash` with sorted ring and binary search |
| Handling skew and hot spots | Section 5 | `HotSpotAwarePartitioner` with key salting |
| Local vs global secondary indexes | Section 6 | `LocalIndexCluster` vs `GlobalIndexCluster` |
| Rebalancing strategies | Section 7 | `RebalanceManager` with iterative load balancing |
| Request routing | Section 8 | `ClientRouter`, `RoutingTier`, `GossipNode` |
| Parallel query execution | Section 9 | `ScatterGatherCluster` with goroutines and channels |

---

## 14. Practice Exercises

### Exercise 1: Weighted Consistent Hashing

Extend the `ConsistentHash` implementation to support weighted nodes. A node with weight 2 should receive approximately twice as many keys as a node with weight 1. Implement this by assigning more virtual nodes to higher-weight physical nodes.

```go
// Starter code
type WeightedConsistentHash struct {
    // Your implementation here
}

func (wch *WeightedConsistentHash) AddNode(nodeID string, weight int) {
    // weight determines how many virtual nodes this physical node gets
}

// Test: With nodes A(weight=1), B(weight=2), C(weight=3),
// verify that B gets ~2x the keys of A and C gets ~3x the keys of A.
```

### Exercise 2: Dynamic Partition Splitting

Implement a partition manager that automatically splits partitions when they exceed a size threshold and merges partitions when they fall below a minimum threshold.

```go
// Starter code
type DynamicPartitionManager struct {
    partitions []*Partition
    splitThreshold int64 // split when partition exceeds this size
    mergeThreshold int64 // merge when two adjacent partitions are below this
}

func (dpm *DynamicPartitionManager) CheckAndSplit(partitionID int) bool {
    // If the partition is too large, split it into two
}

func (dpm *DynamicPartitionManager) CheckAndMerge(partitionID1, partitionID2 int) bool {
    // If two adjacent partitions are small enough, merge them
}
```

### Exercise 3: Hedged Requests

Implement hedged requests for the scatter-gather pattern. If a partition does not respond within a timeout, send the same request to a replica. Return whichever response arrives first.

```go
// Starter code
func (c *Cluster) HedgedScatterGather(
    ctx context.Context,
    query Query,
    hedgeDelay time.Duration,
) ([]Result, error) {
    // 1. Send query to all primary partitions
    // 2. After hedgeDelay, send to replicas for any that haven't responded
    // 3. For each partition, return whichever response arrives first
}
```

### Exercise 4: Partition-Level Rate Limiting

Implement rate limiting on a per-partition basis. If a partition is receiving too many requests (indicating a hot spot), throttle requests to that partition and return a "retry later" response.

```go
// Starter code
type PartitionRateLimiter struct {
    limits map[int]*rate.Limiter // partitionID -> limiter
}

func (prl *PartitionRateLimiter) Allow(partitionID int) bool {
    // Return true if the request is allowed, false if throttled
}
```

### Exercise 5: Multi-Key Transactions

Implement a simple two-phase commit protocol for operations that span multiple partitions. For example, transferring a balance from one user (on partition A) to another (on partition B).

```go
// Starter code
type MultiPartitionTx struct {
    store *ShardedKVStore
}

func (tx *MultiPartitionTx) Transfer(fromKey, toKey string, amount int) error {
    // Phase 1: Prepare -- lock both keys
    // Phase 2: Commit -- apply changes to both partitions
    // Handle failures: abort and rollback if either partition fails
}
```

### Exercise 6: Partition Health Monitoring

Build a health monitoring system that tracks per-partition latency percentiles (p50, p95, p99), error rates, and data size. Automatically flag unhealthy partitions.

```go
// Starter code
type PartitionHealthMonitor struct {
    metrics map[int]*PartitionMetrics
}

type PartitionMetrics struct {
    latencies   []time.Duration // rolling window
    errorCount  int64
    requestCount int64
    dataSize    int64
}

func (phm *PartitionHealthMonitor) RecordLatency(partitionID int, d time.Duration) {}
func (phm *PartitionHealthMonitor) GetUnhealthyPartitions() []int {}
```

### Exercise 7: Cross-Datacenter Partitioning

Extend the sharded key-value store to support partitions across multiple datacenters. Implement a routing strategy that prefers local datacenter reads but falls back to remote datacenters when the local one is unavailable.

```go
// Starter code
type DatacenterAwareRouter struct {
    localDC    string
    partitions map[int][]DatacenterReplica // partitionID -> replicas in each DC
}

type DatacenterReplica struct {
    Datacenter string
    NodeAddr   string
    IsLeader   bool
}

func (r *DatacenterAwareRouter) Route(key string, readOnly bool) string {
    // For reads: prefer local DC replica
    // For writes: always route to leader (may be in remote DC)
}
```

### Exercise 8: Build a Sharded Queue

Implement a distributed message queue where messages are partitioned by a routing key (similar to Kafka). Include consumer groups where each partition is consumed by exactly one consumer in the group.

```go
// Starter code
type ShardedQueue struct {
    partitions []*QueuePartition
    ring       *ConsistentHash
}

type QueuePartition struct {
    id       int
    messages []*Message
    offset   int64
}

func (sq *ShardedQueue) Produce(routingKey string, message *Message) error {}
func (sq *ShardedQueue) Consume(consumerGroup string, partitionID int) (*Message, error) {}
```

Each exercise builds on the concepts from this chapter and reinforces a different aspect of partitioning in distributed systems. Start with Exercises 1-3 (foundational) and work your way up to Exercises 7-8 (system design challenges).
