# Chapter 38: Distributed Systems Challenges

> **Level: Advanced / Distributed Systems** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 16 (Context), 17 (Advanced Concurrency), 21 (Microservices)**

This chapter is about the fundamental truths that make distributed computing hard. Not the tools, not the frameworks -- the physics. Networks lose packets. Clocks drift. Processes crash without warning. Every distributed system you will ever build sits on top of these harsh realities, and pretending otherwise is how outages happen.

The material here is based on Chapter 8 of Martin Kleppmann's *Designing Data-Intensive Applications* -- "The Trouble with Distributed Systems." We implement every concept in Go, using goroutines as nodes and channels as networks. You will build failure detectors, logical clocks, vector clocks, leader election, Raft consensus, distributed locks, gossip protocols, and more -- all as runnable Go programs.

By the end of this chapter, you will have a visceral understanding of why distributed systems fail, and you will have built the core algorithms that real systems use to cope.

---

## Table of Contents

1. [Why Distributed Systems Are Hard](#1-why-distributed-systems-are-hard)
2. [Network Faults](#2-network-faults)
3. [Timeouts and Failure Detection](#3-timeouts-and-failure-detection)
4. [Clock Problems](#4-clock-problems)
5. [Logical Clocks](#5-logical-clocks)
6. [Vector Clocks](#6-vector-clocks)
7. [The Byzantine Generals Problem](#7-the-byzantine-generals-problem)
8. [Leader Election](#8-leader-election)
9. [Consensus with Raft](#9-consensus-with-raft)
10. [Distributed Mutual Exclusion](#10-distributed-mutual-exclusion)
11. [CAP Theorem](#11-cap-theorem)
12. [Exactly-Once Semantics](#12-exactly-once-semantics)
13. [Gossip Protocols](#13-gossip-protocols)
14. [Real-World Example: Building a Distributed Lock Service](#14-real-world-example-building-a-distributed-lock-service)
15. [Key Takeaways](#15-key-takeaways)
16. [Practice Exercises](#16-practice-exercises)

---

## 1. Why Distributed Systems Are Hard

### The Three Fundamental Problems

A single-machine program lives in a kind universe. Memory is reliable. The CPU executes instructions in order. If you call a function, it either returns a result or your process crashes -- there is no ambiguity. Distributed systems offer no such guarantees:

```
Single Machine                        Distributed System
──────────────                        ──────────────────

Function call:                        Network call:
  - Returns a result                    - Returns a result
  - OR throws an exception              - OR returns an error
  - Nothing else can happen             - OR times out (did it succeed?)
                                        - OR the response is lost
                                        - OR the remote crashed after processing
                                        - OR the remote is slow, not dead
                                        - OR a network partition isolates you

Shared memory:                        No shared memory:
  - Read what was written               - Message could arrive out of order
  - Consistent view of state            - Each node has its own state
  - One clock                           - Each node has its own clock
                                        - Clocks drift apart

Failure:                              Partial failure:
  - Everything works                    - Node A is up, node B is down
  - OR everything crashes               - Node A cannot tell if B is down or slow
                                        - Different nodes have different views
```

These are not edge cases. They are the normal operating conditions of any system that communicates over a network.

### The Eight Fallacies of Distributed Computing

Peter Deutsch and others at Sun Microsystems identified assumptions that developers incorrectly make about networks:

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology does not change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every fallacy, when violated, causes a different class of bug. This chapter systematically addresses fallacies 1-3 (network reliability, latency, clocks) and the consequences of partial failure.

### Demonstrating the Problem

Let us start with a simple demonstration of why distributed systems are hard. We simulate three nodes trying to agree on a value when messages can be lost:

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Node represents a participant in our distributed system.
type Node struct {
	id       string
	value    int
	received []Message
	mu       sync.Mutex
}

// Message represents a network message between nodes.
type Message struct {
	From  string
	Value int
}

// UnreliableChannel simulates a network that loses 30% of messages.
type UnreliableChannel struct {
	lossRate float64
	target   chan Message
}

func (uc *UnreliableChannel) Send(msg Message) bool {
	if rand.Float64() < uc.lossRate {
		// Message lost in transit.
		return false
	}
	uc.target <- msg
	return true
}

func main() {
	rand.Seed(time.Now().UnixNano())

	// Create three nodes, each with a different initial value.
	nodes := map[string]*Node{
		"A": {id: "A", value: 10},
		"B": {id: "B", value: 20},
		"C": {id: "C", value: 30},
	}

	// Create unreliable channels between nodes (30% loss rate).
	channels := make(map[string]chan Message)
	for id := range nodes {
		channels[id] = make(chan Message, 10)
	}

	unreliable := make(map[string]*UnreliableChannel)
	for id, ch := range channels {
		unreliable[id] = &UnreliableChannel{lossRate: 0.3, target: ch}
	}

	// Each node tries to broadcast its value to every other node.
	var wg sync.WaitGroup

	// Start receivers.
	for id, node := range nodes {
		wg.Add(1)
		go func(id string, node *Node) {
			defer wg.Done()
			timeout := time.After(500 * time.Millisecond)
			for {
				select {
				case msg := <-channels[id]:
					node.mu.Lock()
					node.received = append(node.received, msg)
					node.mu.Unlock()
				case <-timeout:
					return
				}
			}
		}(id, node)
	}

	// Send messages.
	for fromID, fromNode := range nodes {
		for toID := range nodes {
			if fromID == toID {
				continue
			}
			msg := Message{From: fromID, Value: fromNode.value}
			delivered := unreliable[toID].Send(msg)
			if !delivered {
				fmt.Printf("  LOST: %s -> %s (value=%d)\n", fromID, toID, fromNode.value)
			} else {
				fmt.Printf("  OK:   %s -> %s (value=%d)\n", fromID, toID, fromNode.value)
			}
		}
	}

	wg.Wait()

	// Check what each node knows.
	fmt.Println("\n--- Node Knowledge ---")
	for id, node := range nodes {
		node.mu.Lock()
		fmt.Printf("Node %s: own value=%d, received %d messages: ", id, node.value, len(node.received))
		for _, msg := range node.received {
			fmt.Printf("[from %s: %d] ", msg.From, msg.Value)
		}
		fmt.Println()
		node.mu.Unlock()
	}

	// The point: nodes have DIFFERENT views of the world.
	fmt.Println("\nEach node has a potentially different view of the system.")
	fmt.Println("This is the fundamental challenge of distributed systems.")
}
```

Run this several times. Each run produces different results because messages are randomly lost. Node A might know about B's value but not C's. Node C might know about both A and B. No node has a complete, consistent view of the system. This is the reality we must design for.

---

## 2. Network Faults

### Types of Network Failures

Networks fail in many ways, and the failure mode matters:

```
Failure Type       Description                        Example
─────────────      ───────────────────────────────     ──────────────────────
Packet loss        Message never arrives               Router drops packet
Packet delay       Message arrives much later          Congestion, queuing
Packet reorder     Messages arrive out of order        Multi-path routing
Packet duplicate   Same message arrives twice          Retransmission
Partition          Two groups of nodes cannot talk      Switch failure, cable cut
Asymmetric fault   A can reach B, but B cannot reach A  One-way link failure
```

### Simulating an Unreliable Network

We build a comprehensive network simulator that models all of these failure modes:

```go
package main

import (
	"errors"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Network errors.
var (
	ErrPacketLost    = errors.New("packet lost in transit")
	ErrPartitioned   = errors.New("network partition: destination unreachable")
	ErrNodeNotFound  = errors.New("destination node not found")
)

// Packet represents a message in transit.
type Packet struct {
	From    string
	To      string
	Payload []byte
	SeqNum  uint64
}

// UnreliableNetwork simulates a network with configurable failure modes.
type UnreliableNetwork struct {
	lossRate      float64           // Probability of dropping a packet (0.0 to 1.0).
	minDelay      time.Duration     // Minimum delivery delay.
	maxDelay      time.Duration     // Maximum delivery delay.
	duplicateRate float64           // Probability of duplicating a packet.
	reorderRate   float64           // Probability of reordering (extra delay).
	partitioned   map[string]bool   // Nodes that are isolated.
	inboxes       map[string]chan Packet
	seqCounter    uint64
	mu            sync.RWMutex
	stats         NetworkStats
}

// NetworkStats tracks what happened on the network.
type NetworkStats struct {
	Sent       int
	Delivered  int
	Lost       int
	Duplicated int
	Partitioned int
	mu         sync.Mutex
}

// NewUnreliableNetwork creates a network with the given parameters.
func NewUnreliableNetwork(lossRate, duplicateRate, reorderRate float64,
	minDelay, maxDelay time.Duration) *UnreliableNetwork {
	return &UnreliableNetwork{
		lossRate:      lossRate,
		minDelay:      minDelay,
		maxDelay:      maxDelay,
		duplicateRate: duplicateRate,
		reorderRate:   reorderRate,
		partitioned:   make(map[string]bool),
		inboxes:       make(map[string]chan Packet),
	}
}

// RegisterNode adds a node to the network.
func (n *UnreliableNetwork) RegisterNode(id string) chan Packet {
	n.mu.Lock()
	defer n.mu.Unlock()
	ch := make(chan Packet, 100)
	n.inboxes[id] = ch
	return ch
}

// Partition isolates a node from the rest of the network.
func (n *UnreliableNetwork) Partition(nodeID string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.partitioned[nodeID] = true
	fmt.Printf("  [NET] Node %s is now PARTITIONED\n", nodeID)
}

// Heal removes a node from partition.
func (n *UnreliableNetwork) Heal(nodeID string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	delete(n.partitioned, nodeID)
	fmt.Printf("  [NET] Node %s partition HEALED\n", nodeID)
}

// Send transmits a packet with simulated failures.
func (n *UnreliableNetwork) Send(from, to string, payload []byte) error {
	n.mu.RLock()
	inbox, exists := n.inboxes[to]
	fromPartitioned := n.partitioned[from]
	toPartitioned := n.partitioned[to]
	n.mu.RUnlock()

	if !exists {
		return ErrNodeNotFound
	}

	n.stats.mu.Lock()
	n.stats.Sent++
	n.stats.mu.Unlock()

	// Check for network partition.
	if fromPartitioned || toPartitioned {
		n.stats.mu.Lock()
		n.stats.Partitioned++
		n.stats.mu.Unlock()
		return ErrPartitioned
	}

	// Simulate packet loss.
	if rand.Float64() < n.lossRate {
		n.stats.mu.Lock()
		n.stats.Lost++
		n.stats.mu.Unlock()
		return ErrPacketLost
	}

	n.mu.Lock()
	n.seqCounter++
	seq := n.seqCounter
	n.mu.Unlock()

	pkt := Packet{From: from, To: to, Payload: payload, SeqNum: seq}

	// Calculate delivery delay.
	delay := n.minDelay
	if n.maxDelay > n.minDelay {
		delay += time.Duration(rand.Int63n(int64(n.maxDelay - n.minDelay)))
	}

	// Simulate reorder by adding extra delay.
	if rand.Float64() < n.reorderRate {
		delay += n.maxDelay
	}

	// Schedule delivery.
	go func() {
		time.Sleep(delay)
		select {
		case inbox <- pkt:
			n.stats.mu.Lock()
			n.stats.Delivered++
			n.stats.mu.Unlock()
		default:
			// Inbox full, drop.
			n.stats.mu.Lock()
			n.stats.Lost++
			n.stats.mu.Unlock()
		}
	}()

	// Simulate duplication: send the packet again with extra delay.
	if rand.Float64() < n.duplicateRate {
		dupDelay := delay + time.Duration(rand.Int63n(int64(n.maxDelay)))
		go func() {
			time.Sleep(dupDelay)
			select {
			case inbox <- pkt:
				n.stats.mu.Lock()
				n.stats.Duplicated++
				n.stats.mu.Unlock()
			default:
			}
		}()
	}

	return nil
}

// PrintStats displays network statistics.
func (n *UnreliableNetwork) PrintStats() {
	n.stats.mu.Lock()
	defer n.stats.mu.Unlock()
	fmt.Printf("\n--- Network Statistics ---\n")
	fmt.Printf("Sent:        %d\n", n.stats.Sent)
	fmt.Printf("Delivered:   %d\n", n.stats.Delivered)
	fmt.Printf("Lost:        %d\n", n.stats.Lost)
	fmt.Printf("Duplicated:  %d\n", n.stats.Duplicated)
	fmt.Printf("Partitioned: %d\n", n.stats.Partitioned)
}

func main() {
	rand.Seed(time.Now().UnixNano())

	// Create a network: 10% loss, 5% duplicate, 10% reorder,
	// 5-50ms delay.
	net := NewUnreliableNetwork(
		0.10, 0.05, 0.10,
		5*time.Millisecond, 50*time.Millisecond,
	)

	// Register three nodes.
	inboxA := net.RegisterNode("A")
	inboxB := net.RegisterNode("B")
	inboxC := net.RegisterNode("C")

	// Receiver goroutines.
	var wg sync.WaitGroup
	received := make(map[string][]Packet)
	var recvMu sync.Mutex

	startReceiver := func(id string, inbox chan Packet) {
		wg.Add(1)
		go func() {
			defer wg.Done()
			timeout := time.After(1 * time.Second)
			for {
				select {
				case pkt := <-inbox:
					recvMu.Lock()
					received[id] = append(received[id], pkt)
					recvMu.Unlock()
				case <-timeout:
					return
				}
			}
		}()
	}

	startReceiver("A", inboxA)
	startReceiver("B", inboxB)
	startReceiver("C", inboxC)

	// Node A sends 20 messages to B and C.
	fmt.Println("--- Sending 20 messages from A to B and C ---")
	for i := 0; i < 20; i++ {
		payload := []byte(fmt.Sprintf("msg-%d", i))
		if err := net.Send("A", "B", payload); err != nil {
			fmt.Printf("  A->B msg-%d: %v\n", i, err)
		}
		if err := net.Send("A", "C", payload); err != nil {
			fmt.Printf("  A->C msg-%d: %v\n", i, err)
		}
	}

	// After 10 messages, partition node C.
	time.Sleep(100 * time.Millisecond)
	net.Partition("C")

	for i := 20; i < 30; i++ {
		payload := []byte(fmt.Sprintf("msg-%d", i))
		if err := net.Send("A", "B", payload); err != nil {
			fmt.Printf("  A->B msg-%d: %v\n", i, err)
		}
		if err := net.Send("A", "C", payload); err != nil {
			fmt.Printf("  A->C msg-%d: %v\n", i, err)
		}
	}

	// Heal partition.
	time.Sleep(100 * time.Millisecond)
	net.Heal("C")

	for i := 30; i < 40; i++ {
		payload := []byte(fmt.Sprintf("msg-%d", i))
		if err := net.Send("A", "B", payload); err != nil {
			fmt.Printf("  A->B msg-%d: %v\n", i, err)
		}
		if err := net.Send("A", "C", payload); err != nil {
			fmt.Printf("  A->C msg-%d: %v\n", i, err)
		}
	}

	wg.Wait()

	// Print results.
	recvMu.Lock()
	for id, pkts := range received {
		fmt.Printf("\nNode %s received %d packets:\n", id, len(pkts))
		seqNums := make([]uint64, len(pkts))
		for i, pkt := range pkts {
			seqNums[i] = pkt.SeqNum
		}
		fmt.Printf("  Sequence numbers: %v\n", seqNums)

		// Check for duplicates.
		seen := make(map[uint64]int)
		for _, s := range seqNums {
			seen[s]++
		}
		for s, count := range seen {
			if count > 1 {
				fmt.Printf("  DUPLICATE: seq %d received %d times\n", s, count)
			}
		}

		// Check for out-of-order delivery.
		outOfOrder := 0
		for i := 1; i < len(seqNums); i++ {
			if seqNums[i] < seqNums[i-1] {
				outOfOrder++
			}
		}
		if outOfOrder > 0 {
			fmt.Printf("  OUT-OF-ORDER: %d times\n", outOfOrder)
		}
	}
	recvMu.Unlock()

	net.PrintStats()
}
```

### What to Observe

When you run this program multiple times, you will see:

- **Lost messages** -- Some messages never arrive. Node B and C receive different subsets.
- **Duplicates** -- The same sequence number appears more than once at a receiver.
- **Reordering** -- Sequence numbers arrive out of their original order.
- **Partition effects** -- During the partition, all messages to C fail. After healing, messages resume but C missed everything during the partition.

This is not theory. This is what TCP connections experience under load, during datacenter maintenance, during cloud provider incidents, and across wide-area networks.

---

## 3. Timeouts and Failure Detection

### The Impossibility of Perfect Failure Detection

In an asynchronous network (one where messages can be delayed arbitrarily), it is mathematically impossible to distinguish a crashed node from a slow one. This is not a limitation of our tools -- it is a proven impossibility result (Fischer, Lynch, Paterson, 1985).

The best we can do is use **unreliable failure detectors** that give probabilistic answers. Two properties matter:

- **Completeness**: If a node crashes, it is eventually suspected by every correct node.
- **Accuracy**: A correct (alive) node is not falsely suspected.

You cannot have both perfectly. Aggressive timeouts improve completeness but hurt accuracy (false positives). Conservative timeouts improve accuracy but hurt completeness (slow detection).

### Simple Heartbeat Failure Detector

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// FailureDetector uses heartbeats to detect node failures.
type FailureDetector struct {
	heartbeats map[string]time.Time // Last heartbeat time per node.
	timeout    time.Duration        // How long before we suspect a node.
	mu         sync.RWMutex
	suspects   map[string]bool      // Currently suspected nodes.
	callbacks  []func(nodeID string, suspected bool)
}

// NewFailureDetector creates a failure detector with the given timeout.
func NewFailureDetector(timeout time.Duration) *FailureDetector {
	return &FailureDetector{
		heartbeats: make(map[string]time.Time),
		timeout:    timeout,
		suspects:   make(map[string]bool),
	}
}

// OnSuspicionChange registers a callback for when suspicion status changes.
func (fd *FailureDetector) OnSuspicionChange(cb func(nodeID string, suspected bool)) {
	fd.mu.Lock()
	defer fd.mu.Unlock()
	fd.callbacks = append(fd.callbacks, cb)
}

// Heartbeat records that a node is alive.
func (fd *FailureDetector) Heartbeat(nodeID string) {
	fd.mu.Lock()
	defer fd.mu.Unlock()
	fd.heartbeats[nodeID] = time.Now()

	// If this node was suspected, clear the suspicion.
	if fd.suspects[nodeID] {
		delete(fd.suspects, nodeID)
		for _, cb := range fd.callbacks {
			cb(nodeID, false) // No longer suspected.
		}
	}
}

// IsAlive returns true if we have received a recent heartbeat.
func (fd *FailureDetector) IsAlive(nodeID string) bool {
	fd.mu.RLock()
	defer fd.mu.RUnlock()
	last, ok := fd.heartbeats[nodeID]
	return ok && time.Since(last) < fd.timeout
}

// Check evaluates all known nodes and updates suspicion status.
func (fd *FailureDetector) Check() map[string]bool {
	fd.mu.Lock()
	defer fd.mu.Unlock()

	status := make(map[string]bool)
	now := time.Now()

	for nodeID, lastBeat := range fd.heartbeats {
		alive := now.Sub(lastBeat) < fd.timeout
		status[nodeID] = alive

		wasSuspected := fd.suspects[nodeID]
		if !alive && !wasSuspected {
			fd.suspects[nodeID] = true
			for _, cb := range fd.callbacks {
				cb(nodeID, true) // Now suspected.
			}
		} else if alive && wasSuspected {
			delete(fd.suspects, nodeID)
			for _, cb := range fd.callbacks {
				cb(nodeID, false) // No longer suspected.
			}
		}
	}

	return status
}

// GetSuspects returns all currently suspected nodes.
func (fd *FailureDetector) GetSuspects() []string {
	fd.mu.RLock()
	defer fd.mu.RUnlock()
	var result []string
	for id := range fd.suspects {
		result = append(result, id)
	}
	return result
}

func main() {
	fd := NewFailureDetector(500 * time.Millisecond)

	// Register suspicion change callback.
	fd.OnSuspicionChange(func(nodeID string, suspected bool) {
		if suspected {
			fmt.Printf("  [ALERT] Node %s is SUSPECTED DEAD\n", nodeID)
		} else {
			fmt.Printf("  [CLEAR] Node %s is ALIVE again\n", nodeID)
		}
	})

	// Simulate three nodes sending heartbeats.
	var wg sync.WaitGroup

	// Node A: healthy, sends heartbeats every 100ms.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 20; i++ {
			fd.Heartbeat("node-A")
			time.Sleep(100 * time.Millisecond)
		}
	}()

	// Node B: crashes after 500ms.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			fd.Heartbeat("node-B")
			time.Sleep(100 * time.Millisecond)
		}
		fmt.Println("  [SIM] Node B has CRASHED")
		// No more heartbeats.
	}()

	// Node C: slow -- heartbeats every 400ms (near the timeout).
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			fd.Heartbeat("node-C")
			time.Sleep(400 * time.Millisecond)
		}
	}()

	// Periodic checker.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 25; i++ {
			time.Sleep(200 * time.Millisecond)
			status := fd.Check()
			alive := 0
			dead := 0
			for _, isAlive := range status {
				if isAlive {
					alive++
				} else {
					dead++
				}
			}
			fmt.Printf("  [CHECK t=%dms] alive=%d suspected=%d\n",
				(i+1)*200, alive, dead)
		}
	}()

	wg.Wait()
	fmt.Println("\nFinal suspects:", fd.GetSuspects())
}
```

### Phi Accrual Failure Detector

The simple heartbeat detector uses a fixed timeout, which is brittle. The **phi accrual failure detector** (used in Akka and Cassandra) dynamically adapts based on observed heartbeat intervals. Instead of a binary alive/dead decision, it computes a **suspicion level** (phi) that represents the likelihood a node has failed:

```go
package main

import (
	"fmt"
	"math"
	"sync"
	"time"
)

// PhiAccrualDetector implements an adaptive failure detector.
// It tracks the distribution of heartbeat intervals and computes
// a suspicion value (phi) based on how long it has been since
// the last heartbeat.
type PhiAccrualDetector struct {
	intervals    map[string][]float64  // Heartbeat interval samples per node.
	lastBeat     map[string]time.Time  // Last heartbeat timestamp per node.
	maxSamples   int                   // Maximum samples to keep per node.
	mu           sync.RWMutex
}

// NewPhiAccrualDetector creates a new phi accrual failure detector.
func NewPhiAccrualDetector(maxSamples int) *PhiAccrualDetector {
	return &PhiAccrualDetector{
		intervals:  make(map[string][]float64),
		lastBeat:   make(map[string]time.Time),
		maxSamples: maxSamples,
	}
}

// Heartbeat records a heartbeat from a node.
func (d *PhiAccrualDetector) Heartbeat(nodeID string) {
	d.mu.Lock()
	defer d.mu.Unlock()

	now := time.Now()
	if last, ok := d.lastBeat[nodeID]; ok {
		interval := now.Sub(last).Seconds() * 1000 // Convert to milliseconds.
		intervals := d.intervals[nodeID]
		intervals = append(intervals, interval)
		if len(intervals) > d.maxSamples {
			intervals = intervals[1:]
		}
		d.intervals[nodeID] = intervals
	}
	d.lastBeat[nodeID] = now
}

// mean computes the mean of a slice of float64.
func mean(vals []float64) float64 {
	if len(vals) == 0 {
		return 0
	}
	sum := 0.0
	for _, v := range vals {
		sum += v
	}
	return sum / float64(len(vals))
}

// stddev computes the standard deviation.
func stddev(vals []float64) float64 {
	if len(vals) < 2 {
		return 0
	}
	m := mean(vals)
	sumSqDiff := 0.0
	for _, v := range vals {
		diff := v - m
		sumSqDiff += diff * diff
	}
	return math.Sqrt(sumSqDiff / float64(len(vals)-1))
}

// Phi computes the suspicion level for a node.
// Higher phi means more likely to be dead.
// Typical threshold: phi > 8 means suspected dead.
func (d *PhiAccrualDetector) Phi(nodeID string) float64 {
	d.mu.RLock()
	defer d.mu.RUnlock()

	last, ok := d.lastBeat[nodeID]
	if !ok {
		return 0 // Never seen this node.
	}

	intervals := d.intervals[nodeID]
	if len(intervals) < 2 {
		// Not enough data; use a default.
		elapsed := time.Since(last).Seconds() * 1000
		if elapsed > 1000 {
			return 16.0 // Very suspicious.
		}
		return 0.1
	}

	elapsed := time.Since(last).Seconds() * 1000 // ms since last heartbeat.
	m := mean(intervals)
	s := stddev(intervals)
	if s < 1 {
		s = 1 // Avoid division by zero.
	}

	// Phi is based on the cumulative distribution function of the normal
	// distribution. If elapsed is much larger than the mean interval,
	// phi will be high.
	// phi = -log10(1 - CDF(elapsed))
	// We use an approximation of the normal CDF.
	y := (elapsed - m) / s
	// CDF approximation using the error function.
	cdf := 0.5 * (1.0 + math.Erf(y/math.Sqrt2))
	if cdf >= 1.0 {
		cdf = 0.9999999
	}
	phi := -math.Log10(1.0 - cdf)
	if phi < 0 {
		phi = 0
	}
	return phi
}

// IsSuspected returns true if phi exceeds the threshold.
func (d *PhiAccrualDetector) IsSuspected(nodeID string, threshold float64) bool {
	return d.Phi(nodeID) > threshold
}

func main() {
	detector := NewPhiAccrualDetector(100)

	var wg sync.WaitGroup

	// Node A: consistent heartbeats every 100ms.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 30; i++ {
			detector.Heartbeat("stable-node")
			time.Sleep(100 * time.Millisecond)
		}
	}()

	// Node B: erratic heartbeats (50-300ms intervals).
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 15; i++ {
			detector.Heartbeat("erratic-node")
			delay := 50 + time.Duration(rand.Intn(250))*time.Millisecond
			time.Sleep(delay)
		}
		fmt.Println("  [SIM] erratic-node stopped sending heartbeats")
	}()

	// Node C: crashes after 1 second.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			detector.Heartbeat("crash-node")
			time.Sleep(100 * time.Millisecond)
		}
		fmt.Println("  [SIM] crash-node has CRASHED")
	}()

	// Monitor phi values.
	wg.Add(1)
	go func() {
		defer wg.Done()
		threshold := 8.0
		for i := 0; i < 40; i++ {
			time.Sleep(150 * time.Millisecond)
			phiStable := detector.Phi("stable-node")
			phiErratic := detector.Phi("erratic-node")
			phiCrash := detector.Phi("crash-node")

			fmt.Printf("  [t=%4dms] phi: stable=%.1f erratic=%.1f crash=%.1f",
				(i+1)*150, phiStable, phiErratic, phiCrash)

			if detector.IsSuspected("crash-node", threshold) {
				fmt.Print(" ** crash-node SUSPECTED **")
			}
			fmt.Println()
		}
	}()

	wg.Wait()
}

// Need this import for the erratic node simulation.
import "math/rand"
```

Note: This example has the `import "math/rand"` at the bottom for readability. In real Go code, put all imports at the top. Here is the corrected version with proper imports:

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sync"
	"time"
)

// PhiAccrualDetector implements an adaptive failure detector.
type PhiAccrualDetector struct {
	intervals  map[string][]float64
	lastBeat   map[string]time.Time
	maxSamples int
	mu         sync.RWMutex
}

func NewPhiAccrualDetector(maxSamples int) *PhiAccrualDetector {
	return &PhiAccrualDetector{
		intervals:  make(map[string][]float64),
		lastBeat:   make(map[string]time.Time),
		maxSamples: maxSamples,
	}
}

func (d *PhiAccrualDetector) Heartbeat(nodeID string) {
	d.mu.Lock()
	defer d.mu.Unlock()
	now := time.Now()
	if last, ok := d.lastBeat[nodeID]; ok {
		interval := now.Sub(last).Seconds() * 1000
		intervals := d.intervals[nodeID]
		intervals = append(intervals, interval)
		if len(intervals) > d.maxSamples {
			intervals = intervals[1:]
		}
		d.intervals[nodeID] = intervals
	}
	d.lastBeat[nodeID] = now
}

func mean(vals []float64) float64 {
	if len(vals) == 0 {
		return 0
	}
	sum := 0.0
	for _, v := range vals {
		sum += v
	}
	return sum / float64(len(vals))
}

func stddev(vals []float64) float64 {
	if len(vals) < 2 {
		return 0
	}
	m := mean(vals)
	sumSq := 0.0
	for _, v := range vals {
		d := v - m
		sumSq += d * d
	}
	return math.Sqrt(sumSq / float64(len(vals)-1))
}

func (d *PhiAccrualDetector) Phi(nodeID string) float64 {
	d.mu.RLock()
	defer d.mu.RUnlock()
	last, ok := d.lastBeat[nodeID]
	if !ok {
		return 0
	}
	intervals := d.intervals[nodeID]
	if len(intervals) < 2 {
		elapsed := time.Since(last).Seconds() * 1000
		if elapsed > 1000 {
			return 16.0
		}
		return 0.1
	}
	elapsed := time.Since(last).Seconds() * 1000
	m := mean(intervals)
	s := stddev(intervals)
	if s < 1 {
		s = 1
	}
	y := (elapsed - m) / s
	cdf := 0.5 * (1.0 + math.Erf(y/math.Sqrt2))
	if cdf >= 1.0 {
		cdf = 0.9999999
	}
	phi := -math.Log10(1.0 - cdf)
	if phi < 0 {
		phi = 0
	}
	return phi
}

func (d *PhiAccrualDetector) IsSuspected(nodeID string, threshold float64) bool {
	return d.Phi(nodeID) > threshold
}

func main() {
	rand.Seed(time.Now().UnixNano())
	detector := NewPhiAccrualDetector(100)

	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 30; i++ {
			detector.Heartbeat("stable")
			time.Sleep(100 * time.Millisecond)
		}
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			detector.Heartbeat("crash")
			time.Sleep(100 * time.Millisecond)
		}
		fmt.Println("  [SIM] crash node stopped")
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 40; i++ {
			time.Sleep(100 * time.Millisecond)
			fmt.Printf("  [t=%4dms] phi(stable)=%.2f phi(crash)=%.2f suspected(crash)=%v\n",
				(i+1)*100,
				detector.Phi("stable"),
				detector.Phi("crash"),
				detector.IsSuspected("crash", 8.0))
		}
	}()

	wg.Wait()
}
```

The advantage of the phi accrual detector is that it adapts automatically. A node with erratic but normal heartbeat intervals will have a higher standard deviation, so it takes longer before the detector suspects it. A node with steady heartbeats will be suspected quickly if even one heartbeat is late.

---

## 4. Clock Problems

### Wall Clocks vs Monotonic Clocks

There are two kinds of clocks, and confusing them causes distributed systems bugs:

```
Wall Clock (time-of-day)              Monotonic Clock
────────────────────────              ────────────────
Purpose: What time is it?             Purpose: How much time has elapsed?
Source:  OS clock, synced via NTP      Source:  CPU counter, never adjusted
Can jump: YES (NTP step, leap second)  Can jump: NO (only moves forward)
Comparable across machines: Kinda      Comparable across machines: NO
Use for: Timestamps in logs            Use for: Measuring durations
         Display to users                       Timeouts
         Scheduling events                      Benchmarks
```

### Go's Clock Behavior

Go's `time.Now()` returns a struct that contains BOTH a wall clock reading and a monotonic clock reading. The `time.Since()` and `time.Until()` functions use the monotonic component, which makes them safe for measuring durations even if the wall clock jumps:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// time.Now() captures both wall clock and monotonic clock.
	start := time.Now()
	fmt.Printf("start = %v\n", start)
	// Output includes monotonic reading: "m=+0.000123456"

	// Simulate some work.
	time.Sleep(100 * time.Millisecond)

	// time.Since uses the monotonic component -- safe even if
	// NTP adjusts the wall clock during this interval.
	elapsed := time.Since(start)
	fmt.Printf("elapsed (monotonic) = %v\n", elapsed)

	// If you strip the monotonic reading (by using Round(0) or
	// calling .UTC() etc.), time.Since falls back to wall clock
	// comparison, which is UNSAFE for duration measurement.
	wallOnly := start.Round(0) // Strips monotonic reading.
	fmt.Printf("wallOnly = %v\n", wallOnly)
	// No "m=" in the output anymore.

	// Demonstration: computing duration with wall clock vs monotonic.
	end := time.Now()
	monoElapsed := end.Sub(start)       // Uses monotonic: SAFE.
	wallElapsed := end.Round(0).Sub(wallOnly) // Uses wall clock: UNSAFE.
	fmt.Printf("monotonic elapsed: %v\n", monoElapsed)
	fmt.Printf("wall clock elapsed: %v\n", wallElapsed)
	fmt.Println("These are usually the same, but can differ if NTP adjusts the clock.")
}
```

### NTP Drift Simulation

Clocks on different machines drift apart. NTP tries to keep them synchronized, but there is always some error. Here is a simulation showing why you cannot rely on timestamps from different machines to determine event ordering:

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"time"
)

// SimulatedClock represents a clock on a node that drifts from real time.
type SimulatedClock struct {
	nodeID    string
	offset    time.Duration // Current offset from "true" time.
	driftRate float64       // Drift in nanoseconds per nanosecond.
	lastSync  time.Time     // When NTP last corrected this clock.
}

// NewSimulatedClock creates a clock with the given drift rate.
// driftRate is parts-per-million (ppm). A typical quartz crystal
// drifts at 10-100 ppm.
func NewSimulatedClock(nodeID string, driftPPM float64) *SimulatedClock {
	return &SimulatedClock{
		nodeID:    nodeID,
		offset:    time.Duration(rand.Intn(100)-50) * time.Millisecond,
		driftRate: driftPPM / 1e6,
		lastSync:  time.Now(),
	}
}

// Now returns this node's view of the current time.
func (c *SimulatedClock) Now() time.Time {
	realNow := time.Now()
	elapsed := realNow.Sub(c.lastSync)
	drift := time.Duration(float64(elapsed) * c.driftRate)
	return realNow.Add(c.offset + drift)
}

// NTPSync simulates an NTP correction (imperfect -- leaves residual error).
func (c *SimulatedClock) NTPSync() {
	residualError := time.Duration(rand.Intn(10)-5) * time.Millisecond
	c.offset = residualError
	c.lastSync = time.Now()
}

// Event records something that happened on a node.
type Event struct {
	NodeID    string
	Timestamp time.Time
	RealTime  time.Time // The "true" time (for comparison).
	Action    string
}

func main() {
	rand.Seed(time.Now().UnixNano())

	// Three servers with different clock drift rates.
	clocks := map[string]*SimulatedClock{
		"server-1": NewSimulatedClock("server-1", 50),  // 50 ppm drift.
		"server-2": NewSimulatedClock("server-2", -30), // -30 ppm drift.
		"server-3": NewSimulatedClock("server-3", 80),  // 80 ppm drift.
	}

	// Simulate events happening in a known order.
	var events []Event

	fmt.Println("--- Events in TRUE order ---")
	actions := []struct {
		node   string
		action string
		delay  time.Duration
	}{
		{"server-1", "user creates order #1001", 0},
		{"server-2", "payment processed for order #1001", 10 * time.Millisecond},
		{"server-3", "inventory reserved for order #1001", 20 * time.Millisecond},
		{"server-1", "confirmation email sent", 30 * time.Millisecond},
		{"server-2", "order status updated to CONFIRMED", 40 * time.Millisecond},
	}

	for i, a := range actions {
		time.Sleep(a.delay)
		realTime := time.Now()
		nodeTime := clocks[a.node].Now()

		event := Event{
			NodeID:    a.node,
			Timestamp: nodeTime,
			RealTime:  realTime,
			Action:    a.action,
		}
		events = append(events, event)
		fmt.Printf("  %d. [%s] %s\n", i+1, a.node, a.action)
		fmt.Printf("     real: %s  node-clock: %s  offset: %v\n",
			realTime.Format("15:04:05.000"),
			nodeTime.Format("15:04:05.000"),
			nodeTime.Sub(realTime))
	}

	// Now sort events by their node timestamps (as a distributed system would).
	sorted := make([]Event, len(events))
	copy(sorted, events)
	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].Timestamp.Before(sorted[j].Timestamp)
	})

	fmt.Println("\n--- Events sorted by NODE TIMESTAMPS ---")
	for i, e := range sorted {
		fmt.Printf("  %d. [%s] %s (node time: %s)\n",
			i+1, e.NodeID, e.Action, e.Timestamp.Format("15:04:05.000"))
	}

	fmt.Println("\nNotice: The node-timestamp order may DIFFER from the true order.")
	fmt.Println("This is why you cannot use wall clocks to determine causality")
	fmt.Println("in a distributed system. You need logical clocks (Section 5).")
}
```

### Key Rules for Clocks in Distributed Systems

1. **Never use `time.Now()` to determine the order of events across machines.** Clock skew means timestamps from different machines are not comparable.
2. **Use `time.Since()` for duration measurement.** It uses the monotonic clock and is immune to NTP adjustments.
3. **Use logical clocks (Lamport or vector) for causal ordering.** See the next two sections.
4. **If you must use wall clocks, add a confidence interval.** Google's TrueTime API returns `[earliest, latest]` bounds -- it tells you "the real time is somewhere in this interval."

---

## 5. Logical Clocks

### The Problem with Physical Clocks

As we just demonstrated, physical clocks on different machines drift and cannot be trusted for ordering events. Lamport (1978) showed that you do not need physical clocks to establish a meaningful order of events. You only need a logical notion of "happened before."

### The Happens-Before Relationship

Event A **happens before** event B (written A -> B) if:

1. A and B are in the same process and A occurs before B, OR
2. A is the sending of a message and B is the receipt of that message, OR
3. There exists an event C such that A -> C and C -> B (transitivity).

If neither A -> B nor B -> A, then A and B are **concurrent** -- neither caused the other, and a distributed system cannot determine which happened first.

### Lamport Timestamps

A Lamport clock is a single counter that preserves the happens-before relationship:

- **Rule 1**: Before every local event, increment the counter.
- **Rule 2**: When sending a message, include the counter value.
- **Rule 3**: When receiving a message with timestamp T, set counter = max(local, T) + 1.

**Guarantee**: If A -> B, then L(A) < L(B). But the converse is NOT true: L(A) < L(B) does NOT mean A -> B. Lamport timestamps give a **total order** consistent with causality but cannot distinguish causal ordering from coincidence.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// LamportClock implements Lamport logical timestamps.
type LamportClock struct {
	counter uint64
	nodeID  string
	mu      sync.Mutex
}

// NewLamportClock creates a new Lamport clock for the given node.
func NewLamportClock(nodeID string) *LamportClock {
	return &LamportClock{nodeID: nodeID}
}

// Tick increments the clock for a local event and returns the new timestamp.
func (lc *LamportClock) Tick() uint64 {
	lc.mu.Lock()
	defer lc.mu.Unlock()
	lc.counter++
	return lc.counter
}

// Time returns the current clock value without incrementing.
func (lc *LamportClock) Time() uint64 {
	lc.mu.Lock()
	defer lc.mu.Unlock()
	return lc.counter
}

// Witness processes a timestamp received from another node.
// Sets counter = max(local, other) + 1.
func (lc *LamportClock) Witness(other uint64) uint64 {
	lc.mu.Lock()
	defer lc.mu.Unlock()
	if other > lc.counter {
		lc.counter = other
	}
	lc.counter++
	return lc.counter
}

// LamportMessage carries a payload and a Lamport timestamp.
type LamportMessage struct {
	From      string
	To        string
	Timestamp uint64
	Payload   string
}

// LamportNode is a node in a distributed system using Lamport clocks.
type LamportNode struct {
	id    string
	clock *LamportClock
	inbox chan LamportMessage
	log   []string
	mu    sync.Mutex
}

// NewLamportNode creates a new node.
func NewLamportNode(id string) *LamportNode {
	return &LamportNode{
		id:    id,
		clock: NewLamportClock(id),
		inbox: make(chan LamportMessage, 100),
	}
}

// LocalEvent records a local event.
func (n *LamportNode) LocalEvent(description string) {
	ts := n.clock.Tick()
	entry := fmt.Sprintf("[%s t=%d] LOCAL: %s", n.id, ts, description)
	n.mu.Lock()
	n.log = append(n.log, entry)
	n.mu.Unlock()
	fmt.Println(entry)
}

// Send sends a message to another node.
func (n *LamportNode) Send(to *LamportNode, payload string) {
	ts := n.clock.Tick()
	msg := LamportMessage{
		From:      n.id,
		To:        to.id,
		Timestamp: ts,
		Payload:   payload,
	}
	entry := fmt.Sprintf("[%s t=%d] SEND -> %s: %s", n.id, ts, to.id, payload)
	n.mu.Lock()
	n.log = append(n.log, entry)
	n.mu.Unlock()
	fmt.Println(entry)

	// Simulate network delay.
	go func() {
		time.Sleep(10 * time.Millisecond)
		to.inbox <- msg
	}()
}

// Receive processes one incoming message.
func (n *LamportNode) Receive() *LamportMessage {
	select {
	case msg := <-n.inbox:
		ts := n.clock.Witness(msg.Timestamp)
		entry := fmt.Sprintf("[%s t=%d] RECV <- %s: %s (msg_ts=%d)",
			n.id, ts, msg.From, msg.Payload, msg.Timestamp)
		n.mu.Lock()
		n.log = append(n.log, entry)
		n.mu.Unlock()
		fmt.Println(entry)
		return &msg
	case <-time.After(100 * time.Millisecond):
		return nil
	}
}

func main() {
	alice := NewLamportNode("Alice")
	bob := NewLamportNode("Bob")
	carol := NewLamportNode("Carol")

	fmt.Println("=== Lamport Clock Demonstration ===\n")

	// Scenario: Alice sends to Bob, Bob processes and sends to Carol,
	// Carol sends back to Alice. Meanwhile, Bob does some local work.

	alice.LocalEvent("create order #42")
	alice.Send(bob, "process order #42")

	time.Sleep(50 * time.Millisecond)
	bob.Receive() // Receives from Alice.
	bob.LocalEvent("validate payment")
	bob.LocalEvent("reserve inventory")
	bob.Send(carol, "ship order #42")

	time.Sleep(50 * time.Millisecond)
	carol.Receive() // Receives from Bob.
	carol.LocalEvent("package item")
	carol.Send(alice, "order #42 shipped")

	time.Sleep(50 * time.Millisecond)
	alice.Receive() // Receives from Carol.
	alice.LocalEvent("send confirmation email")

	// Show final clock values.
	fmt.Printf("\n=== Final Clock Values ===\n")
	fmt.Printf("Alice: %d\n", alice.clock.Time())
	fmt.Printf("Bob:   %d\n", bob.clock.Time())
	fmt.Printf("Carol: %d\n", carol.clock.Time())

	fmt.Println("\nNotice how timestamps increase along causal chains:")
	fmt.Println("  Alice(create) -> Alice(send) -> Bob(recv) -> Bob(validate)")
	fmt.Println("  -> Bob(reserve) -> Bob(send) -> Carol(recv) -> Carol(package)")
	fmt.Println("  -> Carol(send) -> Alice(recv) -> Alice(confirm)")
	fmt.Println("Every step has a strictly increasing Lamport timestamp.")
}
```

### Limitations of Lamport Timestamps

Lamport timestamps tell you: "If A caused B, then L(A) < L(B)." But they do NOT tell you: "If L(A) < L(B), then A caused B." Two concurrent events can have any relative Lamport timestamp order. To track true causality, you need vector clocks.

---

## 6. Vector Clocks

### Why Vector Clocks Exist

A Lamport timestamp is a single number. It provides a total order but loses information about which events are truly concurrent versus causally related. A **vector clock** maintains a separate counter for each node, preserving the full causal history:

```
Lamport:  event has timestamp 7
          -- You know 7 > 5, but you do not know if the event at 5 caused this one.

Vector:   event has timestamp {A:3, B:2, C:1}
          -- You know this event saw A's 3rd event, B's 2nd, and C's 1st.
          -- You can tell exactly what this event depends on.
```

### Vector Clock Rules

Each node N maintains a vector V where V[i] is the count of events from node i that N knows about.

- **Local event**: V[self]++
- **Send message**: V[self]++, include V in the message.
- **Receive message (from node with vector V')**: V[i] = max(V[i], V'[i]) for all i, then V[self]++.

Two events are **concurrent** if neither vector dominates the other.

```go
package main

import (
	"fmt"
	"sort"
	"strings"
	"sync"
	"time"
)

// VectorClock represents a vector of logical timestamps.
type VectorClock struct {
	clocks map[string]uint64
	mu     sync.Mutex
}

// NewVectorClock creates a new vector clock.
func NewVectorClock() *VectorClock {
	return &VectorClock{clocks: make(map[string]uint64)}
}

// Copy creates a deep copy of the vector clock.
func (vc *VectorClock) Copy() *VectorClock {
	vc.mu.Lock()
	defer vc.mu.Unlock()
	newVC := NewVectorClock()
	for k, v := range vc.clocks {
		newVC.clocks[k] = v
	}
	return newVC
}

// Tick increments the clock for the given node.
func (vc *VectorClock) Tick(nodeID string) {
	vc.mu.Lock()
	defer vc.mu.Unlock()
	vc.clocks[nodeID]++
}

// Merge takes the element-wise maximum with another vector clock.
func (vc *VectorClock) Merge(other *VectorClock) {
	vc.mu.Lock()
	defer vc.mu.Unlock()
	other.mu.Lock()
	defer other.mu.Unlock()
	for k, v := range other.clocks {
		if v > vc.clocks[k] {
			vc.clocks[k] = v
		}
	}
}

// Ordering represents the causal relationship between two events.
type Ordering int

const (
	Before     Ordering = iota // A happened before B.
	After                      // A happened after B.
	Concurrent                 // A and B are concurrent.
	Equal                      // A and B are the same event.
)

func (o Ordering) String() string {
	switch o {
	case Before:
		return "BEFORE"
	case After:
		return "AFTER"
	case Concurrent:
		return "CONCURRENT"
	case Equal:
		return "EQUAL"
	default:
		return "UNKNOWN"
	}
}

// Compare determines the causal relationship between two vector clocks.
func Compare(a, b *VectorClock) Ordering {
	a.mu.Lock()
	defer a.mu.Unlock()
	b.mu.Lock()
	defer b.mu.Unlock()

	// Collect all keys.
	keys := make(map[string]bool)
	for k := range a.clocks {
		keys[k] = true
	}
	for k := range b.clocks {
		keys[k] = true
	}

	aBeforeB := false // At least one component of A is strictly less than B.
	bBeforeA := false // At least one component of B is strictly less than A.

	for k := range keys {
		av := a.clocks[k]
		bv := b.clocks[k]
		if av < bv {
			aBeforeB = true
		}
		if bv < av {
			bBeforeA = true
		}
	}

	switch {
	case aBeforeB && !bBeforeA:
		return Before // A happened before B.
	case bBeforeA && !aBeforeB:
		return After // A happened after B.
	case !aBeforeB && !bBeforeA:
		return Equal
	default:
		return Concurrent // Some components are greater, some lesser.
	}
}

// String returns a human-readable representation.
func (vc *VectorClock) String() string {
	vc.mu.Lock()
	defer vc.mu.Unlock()

	// Sort keys for deterministic output.
	keys := make([]string, 0, len(vc.clocks))
	for k := range vc.clocks {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	parts := make([]string, len(keys))
	for i, k := range keys {
		parts[i] = fmt.Sprintf("%s:%d", k, vc.clocks[k])
	}
	return "{" + strings.Join(parts, ", ") + "}"
}

// VCMessage carries a vector clock with a message.
type VCMessage struct {
	From    string
	Clock   *VectorClock
	Payload string
}

// VCNode is a node using vector clocks.
type VCNode struct {
	id    string
	clock *VectorClock
	inbox chan VCMessage
	events []struct {
		description string
		clock       *VectorClock
	}
	mu sync.Mutex
}

func NewVCNode(id string) *VCNode {
	vc := NewVectorClock()
	return &VCNode{
		id:    id,
		clock: vc,
		inbox: make(chan VCMessage, 100),
	}
}

func (n *VCNode) recordEvent(desc string) {
	snapshot := n.clock.Copy()
	n.mu.Lock()
	n.events = append(n.events, struct {
		description string
		clock       *VectorClock
	}{desc, snapshot})
	n.mu.Unlock()
	fmt.Printf("  [%s] %s  clock=%s\n", n.id, desc, snapshot)
}

func (n *VCNode) LocalEvent(desc string) {
	n.clock.Tick(n.id)
	n.recordEvent(desc)
}

func (n *VCNode) Send(to *VCNode, payload string) {
	n.clock.Tick(n.id)
	msg := VCMessage{
		From:    n.id,
		Clock:   n.clock.Copy(),
		Payload: payload,
	}
	n.recordEvent(fmt.Sprintf("send to %s: %s", to.id, payload))
	go func() {
		time.Sleep(5 * time.Millisecond)
		to.inbox <- msg
	}()
}

func (n *VCNode) Receive() *VCMessage {
	select {
	case msg := <-n.inbox:
		n.clock.Merge(msg.Clock)
		n.clock.Tick(n.id)
		n.recordEvent(fmt.Sprintf("recv from %s: %s", msg.From, msg.Payload))
		return &msg
	case <-time.After(200 * time.Millisecond):
		return nil
	}
}

func main() {
	fmt.Println("=== Vector Clock Demonstration ===\n")

	alice := NewVCNode("A")
	bob := NewVCNode("B")
	carol := NewVCNode("C")

	// Scenario: Alice and Carol both independently send to Bob.
	// These are concurrent events -- vector clocks can detect this.

	alice.LocalEvent("write x=1")      // A:{A:1}
	carol.LocalEvent("write y=2")      // C:{C:1}

	alice.Send(bob, "x=1")             // A:{A:2}
	carol.Send(bob, "y=2")             // C:{C:2}

	time.Sleep(50 * time.Millisecond)

	bob.Receive() // From Alice or Carol (order may vary).
	bob.Receive() // The other one.

	bob.LocalEvent("process both updates")
	bob.Send(alice, "ack")

	time.Sleep(50 * time.Millisecond)
	alice.Receive()

	// Now compare events to detect concurrency.
	fmt.Println("\n=== Causality Analysis ===")

	alice.mu.Lock()
	bob.mu.Lock()
	carol.mu.Lock()

	// Alice's first event vs Carol's first event.
	if len(alice.events) > 0 && len(carol.events) > 0 {
		aliceFirst := alice.events[0]
		carolFirst := carol.events[0]
		order := Compare(aliceFirst.clock, carolFirst.clock)
		fmt.Printf("\n  Alice(%s) vs Carol(%s) = %s\n",
			aliceFirst.description, carolFirst.description, order)
		fmt.Println("  -> These events are CONCURRENT. Neither caused the other.")
	}

	// Alice's first event vs Bob's last event.
	if len(alice.events) > 0 && len(bob.events) > 0 {
		aliceFirst := alice.events[0]
		bobLast := bob.events[len(bob.events)-1]
		order := Compare(aliceFirst.clock, bobLast.clock)
		fmt.Printf("\n  Alice(%s) vs Bob(%s) = %s\n",
			aliceFirst.description, bobLast.description, order)
		fmt.Println("  -> Alice's write happened BEFORE Bob's ack (causal chain).")
	}

	carol.mu.Unlock()
	bob.mu.Unlock()
	alice.mu.Unlock()

	fmt.Println("\nVector clocks can detect concurrency -- Lamport timestamps cannot.")
}
```

### When to Use Which Clock

```
Clock Type          Use Case                                    Cost
──────────          ────────                                    ────
Lamport timestamp   Total ordering, tiebreaking                 O(1) per event
Vector clock        Detecting concurrent updates (conflicts)    O(N) per event, N = nodes
Physical clock      User-facing timestamps, TTLs                Free but imprecise
Hybrid (HLC)        Best of both: physical + logical            O(1) per event
```

In practice, systems like DynamoDB and Riak use vector clocks (or their variants) to detect conflicting writes. CockroachDB uses hybrid logical clocks. Lamport timestamps are used internally in Raft and Paxos.

---

## 7. The Byzantine Generals Problem

### The Problem

Imagine several army divisions surrounding a city. They must all agree on a plan: attack or retreat. They communicate by messenger. The problem:

1. Messages can be lost (unreliable network).
2. Some generals may be **traitors** who send contradictory messages to different allies.

This maps directly to distributed systems. Nodes might not just crash -- they might behave maliciously (sending wrong data, corrupting messages, lying about their state). This is a **Byzantine fault**.

### Trust Assumptions

Most distributed systems operate under one of three failure models:

```
Failure Model        Assumption                         Example Systems
─────────────        ──────────                         ───────────────
Crash-stop           Nodes either work or crash.         Raft, ZooKeeper, etcd
                     No lying, no corruption.

Crash-recovery       Nodes can crash and restart.         Databases with WAL
                     May lose in-memory state.

Byzantine            Nodes can behave arbitrarily.        Blockchain, BFT protocols
                     May lie, send wrong data,            (PBFT, Tendermint)
                     collude with other faulty nodes.
```

Most internal systems (microservices within a company) use crash-stop or crash-recovery models. Byzantine fault tolerance is needed when nodes are controlled by different parties who may not trust each other.

### Byzantine Fault Tolerance: The Math

- **Crash failures**: Need 2f + 1 nodes to tolerate f failures (majority quorum).
- **Byzantine failures**: Need 3f + 1 nodes to tolerate f Byzantine nodes.

Why 3f + 1? A Byzantine node can lie. To outvote f liars, you need 2f + 1 honest nodes that agree. Total = f (Byzantine) + 2f + 1 (honest) = 3f + 1.

### Simulating Byzantine Behavior

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// General represents a node in the Byzantine generals problem.
type General struct {
	id         int
	isByzantine bool
	decision   string      // "attack" or "retreat"
	received   map[int]string // Messages received from other generals.
	mu         sync.Mutex
}

// ByzantineSystem simulates the Byzantine generals problem.
type ByzantineSystem struct {
	generals []*General
	n        int
}

func NewByzantineSystem(n int, byzantineIDs []int) *ByzantineSystem {
	byzantineSet := make(map[int]bool)
	for _, id := range byzantineIDs {
		byzantineSet[id] = true
	}

	generals := make([]*General, n)
	for i := 0; i < n; i++ {
		generals[i] = &General{
			id:          i,
			isByzantine: byzantineSet[i],
			received:    make(map[int]string),
		}
	}
	return &ByzantineSystem{generals: generals, n: n}
}

// SimulateRound runs one round of message exchange.
// The commander (generals[0]) proposes a value.
// Honest generals relay truthfully; Byzantine generals may lie.
func (bs *ByzantineSystem) SimulateRound(proposal string) {
	fmt.Printf("Commander (General 0) proposes: %s", proposal)
	if bs.generals[0].isByzantine {
		fmt.Print(" [BYZANTINE - will send conflicting messages]")
	}
	fmt.Println()

	var wg sync.WaitGroup

	// Phase 1: Commander sends to all lieutenants.
	for i := 1; i < bs.n; i++ {
		wg.Add(1)
		go func(lieutenantID int) {
			defer wg.Done()

			var msg string
			if bs.generals[0].isByzantine {
				// Byzantine commander sends different values to different generals.
				if rand.Float64() < 0.5 {
					msg = "attack"
				} else {
					msg = "retreat"
				}
			} else {
				msg = proposal
			}

			bs.generals[lieutenantID].mu.Lock()
			bs.generals[lieutenantID].received[0] = msg
			bs.generals[lieutenantID].mu.Unlock()

			fmt.Printf("  General 0 -> General %d: %s\n", lieutenantID, msg)
		}(i)
	}
	wg.Wait()

	// Phase 2: Each lieutenant relays what they received to all others.
	for i := 1; i < bs.n; i++ {
		for j := 1; j < bs.n; j++ {
			if i == j {
				continue
			}
			wg.Add(1)
			go func(from, to int) {
				defer wg.Done()

				bs.generals[from].mu.Lock()
				originalMsg := bs.generals[from].received[0]
				bs.generals[from].mu.Unlock()

				var relayMsg string
				if bs.generals[from].isByzantine {
					// Byzantine lieutenant may lie about what they received.
					if rand.Float64() < 0.5 {
						relayMsg = "attack"
					} else {
						relayMsg = "retreat"
					}
				} else {
					relayMsg = originalMsg
				}

				bs.generals[to].mu.Lock()
				bs.generals[to].received[from] = relayMsg
				bs.generals[to].mu.Unlock()
			}(i, j)
		}
	}
	wg.Wait()

	// Phase 3: Each honest general decides by majority vote.
	fmt.Println("\n--- Decisions ---")
	for i := 0; i < bs.n; i++ {
		g := bs.generals[i]
		if g.isByzantine {
			fmt.Printf("  General %d: BYZANTINE (no honest decision)\n", i)
			continue
		}

		g.mu.Lock()
		attackCount := 0
		retreatCount := 0
		for _, msg := range g.received {
			if msg == "attack" {
				attackCount++
			} else {
				retreatCount++
			}
		}
		g.mu.Unlock()

		if attackCount >= retreatCount {
			g.decision = "attack"
		} else {
			g.decision = "retreat"
		}

		fmt.Printf("  General %d: decides %s (attack=%d, retreat=%d)\n",
			i, g.decision, attackCount, retreatCount)
	}

	// Check agreement among honest generals.
	fmt.Println("\n--- Agreement Check ---")
	decisions := make(map[string][]int)
	for i := 0; i < bs.n; i++ {
		if !bs.generals[i].isByzantine {
			decisions[bs.generals[i].decision] = append(
				decisions[bs.generals[i].decision], i)
		}
	}

	if len(decisions) == 1 {
		fmt.Println("  All honest generals AGREE.")
	} else {
		fmt.Println("  Honest generals DISAGREE -- Byzantine fault caused inconsistency!")
		for decision, generals := range decisions {
			fmt.Printf("    %s: generals %v\n", decision, generals)
		}
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())

	fmt.Println("=== Scenario 1: 4 generals, 1 Byzantine (should work: 3f+1 = 4, f=1) ===\n")
	sys1 := NewByzantineSystem(4, []int{2}) // General 2 is Byzantine.
	sys1.SimulateRound("attack")

	fmt.Println("\n=== Scenario 2: 3 generals, 1 Byzantine (may fail: need 4 for f=1) ===\n")
	sys2 := NewByzantineSystem(3, []int{1})
	sys2.SimulateRound("attack")

	fmt.Println("\n=== Scenario 3: 7 generals, 2 Byzantine (should work: 3f+1 = 7, f=2) ===\n")
	sys3 := NewByzantineSystem(7, []int{1, 4})
	sys3.SimulateRound("retreat")
}
```

The key takeaway: with 3f+1 nodes, you can tolerate f Byzantine faults. With fewer nodes, Byzantine traitors can cause honest nodes to disagree.

---

## 8. Leader Election

### Why Leaders Are Needed

Many distributed algorithms require one node to act as a **leader** (also called coordinator or master). The leader serializes decisions, breaking symmetry and avoiding conflicts. But leaders can crash, so you need an algorithm to elect a new one.

### The Bully Algorithm

The simplest leader election algorithm. When a node suspects the leader is dead:

1. It sends an ELECTION message to all nodes with higher IDs.
2. If no one responds, it declares itself the leader.
3. If a higher-ID node responds, it backs off and waits.
4. The highest-ID node that responds wins and broadcasts COORDINATOR.

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type BullyMessage struct {
	Type   string // "ELECTION", "OK", "COORDINATOR"
	FromID int
}

type BullyNode struct {
	id       int
	alive    bool
	leader   int
	inbox    chan BullyMessage
	peers    map[int]*BullyNode
	mu       sync.Mutex
	electing bool
}

func NewBullyNode(id int) *BullyNode {
	return &BullyNode{
		id:    id,
		alive: true,
		leader: -1,
		inbox: make(chan BullyMessage, 50),
		peers: make(map[int]*BullyNode),
	}
}

func (n *BullyNode) sendTo(targetID int, msg BullyMessage) bool {
	n.mu.Lock()
	peer, exists := n.peers[targetID]
	n.mu.Unlock()

	if !exists {
		return false
	}

	peer.mu.Lock()
	isAlive := peer.alive
	peer.mu.Unlock()

	if !isAlive {
		return false
	}

	select {
	case peer.inbox <- msg:
		return true
	case <-time.After(100 * time.Millisecond):
		return false
	}
}

func (n *BullyNode) StartElection() {
	n.mu.Lock()
	if n.electing {
		n.mu.Unlock()
		return
	}
	n.electing = true
	n.mu.Unlock()

	fmt.Printf("  Node %d: Starting election\n", n.id)

	// Send ELECTION to all nodes with higher IDs.
	gotResponse := false
	var responseMu sync.Mutex
	var wg sync.WaitGroup

	n.mu.Lock()
	for peerID := range n.peers {
		if peerID > n.id {
			wg.Add(1)
			go func(pid int) {
				defer wg.Done()
				ok := n.sendTo(pid, BullyMessage{Type: "ELECTION", FromID: n.id})
				if ok {
					responseMu.Lock()
					gotResponse = true
					responseMu.Unlock()
				}
			}(peerID)
		}
	}
	n.mu.Unlock()

	wg.Wait()

	// Wait a bit for OK responses.
	time.Sleep(200 * time.Millisecond)

	// Check if anyone responded with OK.
	gotOK := false
	drainLoop:
	for {
		select {
		case msg := <-n.inbox:
			if msg.Type == "OK" {
				gotOK = true
				fmt.Printf("  Node %d: Got OK from %d, backing off\n", n.id, msg.FromID)
			} else if msg.Type == "COORDINATOR" {
				n.mu.Lock()
				n.leader = msg.FromID
				n.electing = false
				n.mu.Unlock()
				fmt.Printf("  Node %d: Accepted %d as leader\n", n.id, msg.FromID)
				return
			}
		default:
			break drainLoop
		}
	}

	if !gotResponse || !gotOK {
		// No higher node responded -- I am the leader.
		n.mu.Lock()
		n.leader = n.id
		n.electing = false
		n.mu.Unlock()
		fmt.Printf("  Node %d: I am the new LEADER\n", n.id)

		// Broadcast COORDINATOR to all.
		n.mu.Lock()
		for peerID := range n.peers {
			go n.sendTo(peerID, BullyMessage{Type: "COORDINATOR", FromID: n.id})
		}
		n.mu.Unlock()
	} else {
		n.mu.Lock()
		n.electing = false
		n.mu.Unlock()
	}
}

func (n *BullyNode) Run(done chan struct{}) {
	for {
		select {
		case <-done:
			return
		case msg := <-n.inbox:
			n.mu.Lock()
			isAlive := n.alive
			n.mu.Unlock()

			if !isAlive {
				continue
			}

			switch msg.Type {
			case "ELECTION":
				fmt.Printf("  Node %d: Got ELECTION from %d, sending OK\n", n.id, msg.FromID)
				n.sendTo(msg.FromID, BullyMessage{Type: "OK", FromID: n.id})
				// Start my own election since I have a higher ID.
				go n.StartElection()

			case "COORDINATOR":
				n.mu.Lock()
				n.leader = msg.FromID
				n.mu.Unlock()
				fmt.Printf("  Node %d: Accepted %d as leader\n", n.id, msg.FromID)
			}
		}
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())

	// Create 5 nodes.
	nodes := make(map[int]*BullyNode)
	for i := 1; i <= 5; i++ {
		nodes[i] = NewBullyNode(i)
	}

	// Connect all nodes to each other.
	for i, ni := range nodes {
		for j, nj := range nodes {
			if i != j {
				ni.peers[j] = nj
			}
		}
	}

	done := make(chan struct{})

	// Start all nodes.
	for _, node := range nodes {
		go node.Run(done)
	}

	fmt.Println("=== Initial Election (node 1 triggers) ===")
	nodes[1].StartElection()
	time.Sleep(1 * time.Second)

	// Print current leader.
	for id, node := range nodes {
		node.mu.Lock()
		fmt.Printf("  Node %d thinks leader is: %d\n", id, node.leader)
		node.mu.Unlock()
	}

	// Crash the leader (node 5).
	fmt.Println("\n=== Node 5 crashes ===")
	nodes[5].mu.Lock()
	nodes[5].alive = false
	nodes[5].mu.Unlock()

	// Node 2 detects the crash and starts election.
	time.Sleep(200 * time.Millisecond)
	fmt.Println("\n=== Node 2 detects crash, starts election ===")
	nodes[2].StartElection()
	time.Sleep(1 * time.Second)

	for id, node := range nodes {
		node.mu.Lock()
		if node.alive {
			fmt.Printf("  Node %d thinks leader is: %d\n", id, node.leader)
		} else {
			fmt.Printf("  Node %d is DEAD\n", id)
		}
		node.mu.Unlock()
	}

	close(done)
}
```

---

## 9. Consensus with Raft

### Why Consensus Is Hard

Consensus means getting a group of nodes to agree on a value, even if some nodes crash. The FLP impossibility result (Fischer, Lynch, Paterson, 1985) proves that no deterministic algorithm can guarantee consensus in an asynchronous system where even one node can crash. Practical algorithms like Raft work around this by using timeouts (partially synchronous model) and randomization.

### Raft Overview

Raft decomposes consensus into three sub-problems:

1. **Leader election**: One node is elected leader. Only the leader handles client requests.
2. **Log replication**: The leader replicates log entries to followers.
3. **Safety**: Once a log entry is committed (replicated to a majority), it stays committed.

### Raft Node States

```
                     timeout,                  receives votes
                     start election            from majority
    ┌──────────┐  ──────────────►  ┌───────────┐  ──────────►  ┌────────┐
    │ Follower │                   │ Candidate │               │ Leader │
    └──────────┘  ◄──────────────  └───────────┘               └────────┘
         ▲          discovers                                      │
         │          leader or                                      │
         │          higher term                                    │
         └─────────────────────────────────────────────────────────┘
                        discovers node with higher term
```

### Simplified Raft Implementation

This is a simplified but functional Raft implementation that demonstrates leader election and log replication:

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// NodeState represents the role of a Raft node.
type NodeState int

const (
	Follower  NodeState = iota
	Candidate
	Leader
)

func (s NodeState) String() string {
	switch s {
	case Follower:
		return "Follower"
	case Candidate:
		return "Candidate"
	case Leader:
		return "Leader"
	default:
		return "Unknown"
	}
}

// LogEntry is one entry in the replicated log.
type LogEntry struct {
	Term    uint64
	Index   uint64
	Command string
}

// VoteRequest is sent by candidates to gather votes.
type VoteRequest struct {
	Term         uint64
	CandidateID  string
	LastLogIndex uint64
	LastLogTerm  uint64
	Reply        chan VoteResponse
}

// VoteResponse is the reply to a vote request.
type VoteResponse struct {
	Term        uint64
	VoteGranted bool
	VoterID     string
}

// AppendRequest is sent by the leader to replicate log entries.
type AppendRequest struct {
	Term         uint64
	LeaderID     string
	PrevLogIndex uint64
	PrevLogTerm  uint64
	Entries      []LogEntry
	LeaderCommit uint64
	Reply        chan AppendResponse
}

// AppendResponse is the reply to an append request.
type AppendResponse struct {
	Term    uint64
	Success bool
	NodeID  string
}

// RaftNode is a single node in a Raft cluster.
type RaftNode struct {
	id          string
	state       NodeState
	currentTerm uint64
	votedFor    string
	log         []LogEntry
	commitIndex uint64
	lastApplied uint64

	peers    map[string]*RaftNode
	voteCh   chan VoteRequest
	appendCh chan AppendRequest

	mu   sync.Mutex
	done chan struct{}
}

// NewRaftNode creates a new Raft node.
func NewRaftNode(id string) *RaftNode {
	return &RaftNode{
		id:       id,
		state:    Follower,
		peers:    make(map[string]*RaftNode),
		voteCh:   make(chan VoteRequest, 20),
		appendCh: make(chan AppendRequest, 20),
		done:     make(chan struct{}),
	}
}

// lastLogInfo returns the index and term of the last log entry.
func (rn *RaftNode) lastLogInfo() (uint64, uint64) {
	if len(rn.log) == 0 {
		return 0, 0
	}
	last := rn.log[len(rn.log)-1]
	return last.Index, last.Term
}

// Run starts the Raft node's main loop.
func (rn *RaftNode) Run() {
	for {
		rn.mu.Lock()
		state := rn.state
		rn.mu.Unlock()

		select {
		case <-rn.done:
			return
		default:
		}

		switch state {
		case Follower:
			rn.runFollower()
		case Candidate:
			rn.runCandidate()
		case Leader:
			rn.runLeader()
		}
	}
}

func (rn *RaftNode) runFollower() {
	timeout := time.Duration(300+rand.Intn(300)) * time.Millisecond
	timer := time.NewTimer(timeout)
	defer timer.Stop()

	for {
		select {
		case <-rn.done:
			return

		case vr := <-rn.voteCh:
			rn.handleVoteRequest(vr)
			// Reset timer on receiving a vote request.
			if !timer.Stop() {
				select {
				case <-timer.C:
				default:
				}
			}
			timer.Reset(timeout)

		case ar := <-rn.appendCh:
			rn.handleAppendEntries(ar)
			if !timer.Stop() {
				select {
				case <-timer.C:
				default:
				}
			}
			timer.Reset(timeout)

		case <-timer.C:
			// Election timeout -- become candidate.
			rn.mu.Lock()
			fmt.Printf("  [%s] Election timeout, becoming candidate (term %d)\n",
				rn.id, rn.currentTerm+1)
			rn.state = Candidate
			rn.mu.Unlock()
			return
		}
	}
}

func (rn *RaftNode) runCandidate() {
	rn.mu.Lock()
	rn.currentTerm++
	rn.votedFor = rn.id
	term := rn.currentTerm
	lastIdx, lastTerm := rn.lastLogInfo()
	rn.mu.Unlock()

	fmt.Printf("  [%s] Starting election for term %d\n", rn.id, term)

	votes := 1 // Vote for self.
	needed := (len(rn.peers)+1)/2 + 1
	replyCh := make(chan VoteResponse, len(rn.peers))

	// Request votes from all peers.
	for _, peer := range rn.peers {
		go func(p *RaftNode) {
			vr := VoteRequest{
				Term:         term,
				CandidateID:  rn.id,
				LastLogIndex: lastIdx,
				LastLogTerm:  lastTerm,
				Reply:        replyCh,
			}
			select {
			case p.voteCh <- vr:
			case <-time.After(100 * time.Millisecond):
				// Peer unreachable.
			}
		}(peer)
	}

	timeout := time.After(time.Duration(300+rand.Intn(300)) * time.Millisecond)

	for {
		select {
		case <-rn.done:
			return

		case resp := <-replyCh:
			rn.mu.Lock()
			if resp.Term > rn.currentTerm {
				rn.currentTerm = resp.Term
				rn.state = Follower
				rn.votedFor = ""
				rn.mu.Unlock()
				fmt.Printf("  [%s] Discovered higher term %d, stepping down\n",
					rn.id, resp.Term)
				return
			}
			rn.mu.Unlock()

			if resp.VoteGranted {
				votes++
				fmt.Printf("  [%s] Got vote from %s (%d/%d)\n",
					rn.id, resp.VoterID, votes, needed)
				if votes >= needed {
					rn.mu.Lock()
					rn.state = Leader
					rn.mu.Unlock()
					fmt.Printf("  [%s] WON election for term %d!\n", rn.id, term)
					return
				}
			}

		case ar := <-rn.appendCh:
			// Got AppendEntries from a leader -- step down.
			rn.mu.Lock()
			if ar.Term >= rn.currentTerm {
				rn.state = Follower
				rn.currentTerm = ar.Term
				rn.mu.Unlock()
				rn.handleAppendEntries(ar)
				return
			}
			rn.mu.Unlock()

		case vr := <-rn.voteCh:
			rn.handleVoteRequest(vr)

		case <-timeout:
			// Election timeout -- start new election.
			fmt.Printf("  [%s] Election timed out, retrying\n", rn.id)
			return
		}
	}
}

func (rn *RaftNode) runLeader() {
	fmt.Printf("  [%s] Acting as LEADER for term %d\n", rn.id, rn.currentTerm)

	heartbeatInterval := 100 * time.Millisecond
	ticker := time.NewTicker(heartbeatInterval)
	defer ticker.Stop()

	// Send initial heartbeat.
	rn.sendHeartbeats()

	for {
		select {
		case <-rn.done:
			return

		case <-ticker.C:
			rn.sendHeartbeats()

		case vr := <-rn.voteCh:
			rn.mu.Lock()
			if vr.Term > rn.currentTerm {
				rn.currentTerm = vr.Term
				rn.state = Follower
				rn.votedFor = ""
				rn.mu.Unlock()
				rn.handleVoteRequest(vr)
				return
			}
			rn.mu.Unlock()
			// Deny vote -- I am the leader.
			vr.Reply <- VoteResponse{Term: rn.currentTerm, VoteGranted: false, VoterID: rn.id}

		case ar := <-rn.appendCh:
			rn.mu.Lock()
			if ar.Term > rn.currentTerm {
				rn.currentTerm = ar.Term
				rn.state = Follower
				rn.mu.Unlock()
				rn.handleAppendEntries(ar)
				return
			}
			rn.mu.Unlock()
		}
	}
}

func (rn *RaftNode) sendHeartbeats() {
	rn.mu.Lock()
	term := rn.currentTerm
	commitIdx := rn.commitIndex
	rn.mu.Unlock()

	for _, peer := range rn.peers {
		go func(p *RaftNode) {
			replyCh := make(chan AppendResponse, 1)
			ar := AppendRequest{
				Term:         term,
				LeaderID:     rn.id,
				Entries:      nil, // Heartbeat has no entries.
				LeaderCommit: commitIdx,
				Reply:        replyCh,
			}
			select {
			case p.appendCh <- ar:
			case <-time.After(50 * time.Millisecond):
			}
		}(peer)
	}
}

func (rn *RaftNode) handleVoteRequest(vr VoteRequest) {
	rn.mu.Lock()
	defer rn.mu.Unlock()

	if vr.Term < rn.currentTerm {
		vr.Reply <- VoteResponse{Term: rn.currentTerm, VoteGranted: false, VoterID: rn.id}
		return
	}

	if vr.Term > rn.currentTerm {
		rn.currentTerm = vr.Term
		rn.state = Follower
		rn.votedFor = ""
	}

	// Grant vote if we have not voted yet (or voted for this candidate)
	// and their log is at least as up-to-date as ours.
	lastIdx, lastTerm := rn.lastLogInfo()
	logOK := vr.LastLogTerm > lastTerm ||
		(vr.LastLogTerm == lastTerm && vr.LastLogIndex >= lastIdx)

	if (rn.votedFor == "" || rn.votedFor == vr.CandidateID) && logOK {
		rn.votedFor = vr.CandidateID
		vr.Reply <- VoteResponse{Term: rn.currentTerm, VoteGranted: true, VoterID: rn.id}
		fmt.Printf("  [%s] Voted for %s in term %d\n", rn.id, vr.CandidateID, vr.Term)
	} else {
		vr.Reply <- VoteResponse{Term: rn.currentTerm, VoteGranted: false, VoterID: rn.id}
	}
}

func (rn *RaftNode) handleAppendEntries(ar AppendRequest) {
	rn.mu.Lock()
	defer rn.mu.Unlock()

	if ar.Term < rn.currentTerm {
		ar.Reply <- AppendResponse{Term: rn.currentTerm, Success: false, NodeID: rn.id}
		return
	}

	if ar.Term > rn.currentTerm {
		rn.currentTerm = ar.Term
		rn.state = Follower
		rn.votedFor = ""
	}

	// Append new entries.
	for _, entry := range ar.Entries {
		rn.log = append(rn.log, entry)
	}

	if ar.LeaderCommit > rn.commitIndex {
		rn.commitIndex = ar.LeaderCommit
	}

	ar.Reply <- AppendResponse{Term: rn.currentTerm, Success: true, NodeID: rn.id}
}

// AppendCommand is called by clients to add a command to the replicated log.
func (rn *RaftNode) AppendCommand(command string) bool {
	rn.mu.Lock()
	if rn.state != Leader {
		rn.mu.Unlock()
		return false
	}

	entry := LogEntry{
		Term:    rn.currentTerm,
		Index:   uint64(len(rn.log) + 1),
		Command: command,
	}
	rn.log = append(rn.log, entry)
	term := rn.currentTerm
	rn.mu.Unlock()

	fmt.Printf("  [%s] Leader appending: %s (index=%d, term=%d)\n",
		rn.id, command, entry.Index, term)

	// Replicate to peers.
	successCount := 1 // Self.
	needed := (len(rn.peers)+1)/2 + 1
	var countMu sync.Mutex
	var wg sync.WaitGroup

	for _, peer := range rn.peers {
		wg.Add(1)
		go func(p *RaftNode) {
			defer wg.Done()
			replyCh := make(chan AppendResponse, 1)
			ar := AppendRequest{
				Term:     term,
				LeaderID: rn.id,
				Entries:  []LogEntry{entry},
				Reply:    replyCh,
			}
			select {
			case p.appendCh <- ar:
				select {
				case resp := <-replyCh:
					if resp.Success {
						countMu.Lock()
						successCount++
						countMu.Unlock()
					}
				case <-time.After(200 * time.Millisecond):
				}
			case <-time.After(200 * time.Millisecond):
			}
		}(peer)
	}

	wg.Wait()

	if successCount >= needed {
		rn.mu.Lock()
		rn.commitIndex = entry.Index
		rn.mu.Unlock()
		fmt.Printf("  [%s] Command committed: %s (replicated to %d/%d)\n",
			rn.id, command, successCount, len(rn.peers)+1)
		return true
	}

	fmt.Printf("  [%s] Command NOT committed: %s (only %d/%d)\n",
		rn.id, command, successCount, len(rn.peers)+1)
	return false
}

func main() {
	rand.Seed(time.Now().UnixNano())

	// Create a 5-node Raft cluster.
	nodeIDs := []string{"N1", "N2", "N3", "N4", "N5"}
	nodes := make(map[string]*RaftNode)

	for _, id := range nodeIDs {
		nodes[id] = NewRaftNode(id)
	}

	// Connect all peers.
	for _, node := range nodes {
		for id, peer := range nodes {
			if id != node.id {
				node.peers[id] = peer
			}
		}
	}

	// Start all nodes.
	for _, node := range nodes {
		go node.Run()
	}

	fmt.Println("=== Raft Cluster Started ===")
	fmt.Println("Waiting for leader election...\n")
	time.Sleep(2 * time.Second)

	// Find the leader.
	var leader *RaftNode
	for _, node := range nodes {
		node.mu.Lock()
		if node.state == Leader {
			leader = node
		}
		fmt.Printf("  %s: state=%s term=%d log_len=%d\n",
			node.id, node.state, node.currentTerm, len(node.log))
		node.mu.Unlock()
	}

	if leader != nil {
		fmt.Printf("\n=== Replicating Commands via Leader %s ===\n\n", leader.id)
		leader.AppendCommand("SET x = 1")
		time.Sleep(500 * time.Millisecond)
		leader.AppendCommand("SET y = 2")
		time.Sleep(500 * time.Millisecond)
		leader.AppendCommand("SET z = 3")
		time.Sleep(500 * time.Millisecond)

		fmt.Println("\n=== Cluster State After Replication ===")
		for _, node := range nodes {
			node.mu.Lock()
			fmt.Printf("  %s: state=%s term=%d commit=%d log=%v\n",
				node.id, node.state, node.currentTerm, node.commitIndex,
				func() []string {
					cmds := make([]string, len(node.log))
					for i, e := range node.log {
						cmds[i] = e.Command
					}
					return cmds
				}())
			node.mu.Unlock()
		}
	} else {
		fmt.Println("No leader elected (try running again -- randomized timeouts)")
	}

	// Shut down.
	for _, node := range nodes {
		close(node.done)
	}
}
```

### What This Demonstrates

- **Randomized election timeouts** prevent split votes (most of the time).
- **Term numbers** prevent stale leaders from causing confusion.
- **Log replication** ensures all nodes eventually have the same commands.
- **Majority quorum** (3 out of 5) ensures committed entries survive failures.

In production Raft implementations (etcd, CockroachDB), there are many additional details: log compaction, snapshotting, membership changes, read leases, and pre-vote extensions. This implementation captures the core protocol.

---

## 10. Distributed Mutual Exclusion

### Why Distributed Locks Are Hard

On a single machine, `sync.Mutex` is simple and reliable. In a distributed system, a lock must handle:

- The lock holder crashes -- who releases the lock?
- The lock holder is slow, not dead -- the lock expires but the holder thinks it still has it.
- Network partition -- two nodes both think they hold the lock.

### Fencing Tokens

The solution is **fencing tokens**. Every time a lock is acquired, a monotonically increasing token is issued. The storage system checks the token and rejects requests with stale tokens:

```
Time ──────────────────────────────────────────────────►

Client A acquires lock   token=33
Client A starts writing to storage
                         Client A's lock EXPIRES (slow GC pause)
                         Client B acquires lock   token=34
                         Client B writes to storage with token=34
Client A tries to write with token=33
Storage sees 33 < 34 and REJECTS the write
```

Without fencing tokens, Client A's stale write would overwrite Client B's data.

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

var (
	ErrLockHeld       = errors.New("lock is held by another client")
	ErrLockExpired    = errors.New("lock has expired")
	ErrStaleFencing   = errors.New("stale fencing token")
	ErrNotLockHolder  = errors.New("not the lock holder")
)

// DistributedLock represents a lock with fencing tokens and expiry.
type DistributedLock struct {
	lockHolder   string
	fencingToken uint64
	expiry       time.Time
	tokenCounter uint64
	mu           sync.Mutex
}

// NewDistributedLock creates a new distributed lock.
func NewDistributedLock() *DistributedLock {
	return &DistributedLock{}
}

// Acquire attempts to acquire the lock. Returns a fencing token on success.
func (dl *DistributedLock) Acquire(clientID string, ttl time.Duration) (uint64, error) {
	dl.mu.Lock()
	defer dl.mu.Unlock()

	// Check if lock is currently held and not expired.
	if dl.lockHolder != "" && time.Now().Before(dl.expiry) {
		if dl.lockHolder == clientID {
			// Re-entrant: refresh the TTL.
			dl.expiry = time.Now().Add(ttl)
			return dl.fencingToken, nil
		}
		return 0, ErrLockHeld
	}

	// Lock is available (either unheld or expired).
	dl.tokenCounter++
	dl.lockHolder = clientID
	dl.fencingToken = dl.tokenCounter
	dl.expiry = time.Now().Add(ttl)

	fmt.Printf("  [LOCK] %s acquired lock (token=%d, expires in %v)\n",
		clientID, dl.fencingToken, ttl)
	return dl.fencingToken, nil
}

// Release releases the lock.
func (dl *DistributedLock) Release(clientID string) error {
	dl.mu.Lock()
	defer dl.mu.Unlock()

	if dl.lockHolder != clientID {
		return ErrNotLockHolder
	}

	fmt.Printf("  [LOCK] %s released lock (token=%d)\n", clientID, dl.fencingToken)
	dl.lockHolder = ""
	return nil
}

// FencedStorage is a storage system that checks fencing tokens.
type FencedStorage struct {
	data         map[string]string
	lastToken    map[string]uint64 // Highest fencing token seen per key.
	writeLog     []WriteOp
	mu           sync.Mutex
}

// WriteOp records a write operation for auditing.
type WriteOp struct {
	ClientID string
	Key      string
	Value    string
	Token    uint64
	Accepted bool
}

// NewFencedStorage creates a new fenced storage system.
func NewFencedStorage() *FencedStorage {
	return &FencedStorage{
		data:      make(map[string]string),
		lastToken: make(map[string]uint64),
	}
}

// Write attempts to write a value, checking the fencing token.
func (fs *FencedStorage) Write(clientID, key, value string, fencingToken uint64) error {
	fs.mu.Lock()
	defer fs.mu.Unlock()

	op := WriteOp{
		ClientID: clientID,
		Key:      key,
		Value:    value,
		Token:    fencingToken,
	}

	// Check fencing token.
	if lastToken, exists := fs.lastToken[key]; exists && fencingToken < lastToken {
		op.Accepted = false
		fs.writeLog = append(fs.writeLog, op)
		fmt.Printf("  [STORAGE] REJECTED write from %s (token=%d < last=%d)\n",
			clientID, fencingToken, lastToken)
		return ErrStaleFencing
	}

	fs.data[key] = value
	fs.lastToken[key] = fencingToken
	op.Accepted = true
	fs.writeLog = append(fs.writeLog, op)
	fmt.Printf("  [STORAGE] ACCEPTED write from %s: %s=%s (token=%d)\n",
		clientID, key, value, fencingToken)
	return nil
}

// PrintLog shows all write operations.
func (fs *FencedStorage) PrintLog() {
	fs.mu.Lock()
	defer fs.mu.Unlock()
	fmt.Println("\n--- Write Operation Log ---")
	for i, op := range fs.writeLog {
		status := "ACCEPTED"
		if !op.Accepted {
			status = "REJECTED"
		}
		fmt.Printf("  %d. [%s] %s wrote %s=%s (token=%d)\n",
			i+1, status, op.ClientID, op.Key, op.Value, op.Token)
	}
	fmt.Printf("\nFinal data: %v\n", fs.data)
}

func main() {
	lock := NewDistributedLock()
	storage := NewFencedStorage()

	fmt.Println("=== Scenario: Fencing Token Prevents Stale Writes ===\n")

	// Client A acquires the lock.
	tokenA, err := lock.Acquire("client-A", 500*time.Millisecond)
	if err != nil {
		fmt.Printf("Client A failed to acquire lock: %v\n", err)
		return
	}

	// Client A writes with its token.
	storage.Write("client-A", "balance", "100", tokenA)

	// Client A's lock expires (simulate slow GC pause).
	fmt.Println("\n  [SIM] Client A enters long GC pause...")
	time.Sleep(600 * time.Millisecond) // Lock expires after 500ms.

	// Client B acquires the now-expired lock.
	tokenB, err := lock.Acquire("client-B", 500*time.Millisecond)
	if err != nil {
		fmt.Printf("Client B failed to acquire lock: %v\n", err)
		return
	}

	// Client B writes with its (higher) token.
	storage.Write("client-B", "balance", "200", tokenB)

	// Client A wakes up from GC pause and tries to write with its OLD token.
	fmt.Println("\n  [SIM] Client A wakes up from GC pause, tries to write...")
	err = storage.Write("client-A", "balance", "150", tokenA)
	if err != nil {
		fmt.Printf("  Client A's stale write was correctly rejected: %v\n", err)
	}

	storage.PrintLog()

	// Show that without fencing tokens, Client A would have overwritten
	// Client B's correct value.
	fmt.Println("\nWithout fencing tokens, the final balance would be 150 (WRONG).")
	fmt.Println("With fencing tokens, the final balance is 200 (CORRECT).")
}
```

---

## 11. CAP Theorem

### What CAP Actually Says

The CAP theorem (Brewer, 2000; formally proven by Gilbert and Lynch, 2002) states that a distributed data store cannot simultaneously provide all three of these guarantees:

- **Consistency (C)**: Every read receives the most recent write or an error.
- **Availability (A)**: Every request receives a non-error response (but not necessarily the most recent write).
- **Partition tolerance (P)**: The system continues to operate despite network partitions.

### The Real Choice: CP vs AP

Since network partitions **will** happen (you do not get to choose P -- the network decides), the real choice is between CP and AP during a partition:

```
During a network partition:

CP system (e.g., etcd, ZooKeeper):
  ┌─────────────────┐         ┌─────────────────┐
  │  Partition A     │   ╳╳╳   │  Partition B     │
  │  (has majority)  │         │  (minority)      │
  │  Continues to    │         │  Returns ERRORS   │
  │  serve reads     │         │  (unavailable)   │
  │  and writes      │         │                  │
  └─────────────────┘         └─────────────────┘

AP system (e.g., DynamoDB, Cassandra):
  ┌─────────────────┐         ┌─────────────────┐
  │  Partition A     │   ╳╳╳   │  Partition B     │
  │  Serves reads    │         │  Serves reads    │
  │  and writes      │         │  and writes      │
  │  (may be stale)  │         │  (may diverge)   │
  └─────────────────┘         └─────────────────┘
  After partition heals: must RECONCILE divergent data
```

### CAP Simulation

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Consistency model.
type ConsistencyModel int

const (
	CPModel ConsistencyModel = iota // Consistency + Partition tolerance.
	APModel                         // Availability + Partition tolerance.
)

func (cm ConsistencyModel) String() string {
	if cm == CPModel {
		return "CP"
	}
	return "AP"
}

// KVNode is a key-value store node.
type KVNode struct {
	id          string
	data        map[string]string
	model       ConsistencyModel
	partitioned bool
	leader      bool
	peers       []*KVNode
	mu          sync.RWMutex
}

func NewKVNode(id string, model ConsistencyModel) *KVNode {
	return &KVNode{
		id:    id,
		data:  make(map[string]string),
		model: model,
	}
}

// Write attempts to write a key-value pair.
func (n *KVNode) Write(key, value string) (bool, string) {
	n.mu.Lock()
	defer n.mu.Unlock()

	if n.partitioned {
		switch n.model {
		case CPModel:
			if !n.leader {
				// CP: Minority partition refuses writes.
				return false, fmt.Sprintf("[%s CP] REFUSED: partitioned, not leader", n.id)
			}
			// Leader in majority partition can still serve.
			n.data[key] = value
			return true, fmt.Sprintf("[%s CP] ACCEPTED: leader in majority", n.id)

		case APModel:
			// AP: Accept the write anyway (may diverge).
			n.data[key] = value
			return true, fmt.Sprintf("[%s AP] ACCEPTED: serving despite partition (may diverge)", n.id)
		}
	}

	// No partition -- replicate normally.
	n.data[key] = value

	// Replicate to peers (simplified).
	for _, peer := range n.peers {
		if !peer.partitioned || n.model == APModel {
			peer.mu.Lock()
			peer.data[key] = value
			peer.mu.Unlock()
		}
	}

	return true, fmt.Sprintf("[%s] ACCEPTED: replicated to %d peers", n.id, len(n.peers))
}

// Read attempts to read a key.
func (n *KVNode) Read(key string) (string, bool, string) {
	n.mu.RLock()
	defer n.mu.RUnlock()

	if n.partitioned {
		switch n.model {
		case CPModel:
			if !n.leader {
				return "", false, fmt.Sprintf("[%s CP] REFUSED: partitioned, not leader", n.id)
			}
		case APModel:
			// AP: Serve the read (may be stale).
			val, ok := n.data[key]
			return val, ok, fmt.Sprintf("[%s AP] SERVED: may be stale", n.id)
		}
	}

	val, ok := n.data[key]
	return val, ok, fmt.Sprintf("[%s] SERVED: consistent", n.id)
}

func main() {
	fmt.Println("=== CAP Theorem Demonstration ===\n")

	// --- CP System ---
	fmt.Println("--- CP System (like etcd/ZooKeeper) ---\n")

	cpNodes := make([]*KVNode, 3)
	for i := 0; i < 3; i++ {
		cpNodes[i] = NewKVNode(fmt.Sprintf("CP-%d", i), CPModel)
	}
	// Node 0 is the leader.
	cpNodes[0].leader = true
	// Connect peers.
	for i := range cpNodes {
		for j := range cpNodes {
			if i != j {
				cpNodes[i].peers = append(cpNodes[i].peers, cpNodes[j])
			}
		}
	}

	// Write before partition.
	ok, msg := cpNodes[0].Write("user:1", "Alice")
	fmt.Printf("  Before partition: %s (ok=%v)\n", msg, ok)

	// Create partition: node 2 is isolated.
	fmt.Println("\n  --- Network Partition ---")
	cpNodes[2].partitioned = true

	// Leader (node 0) can still write (has majority: 0 and 1).
	ok, msg = cpNodes[0].Write("user:2", "Bob")
	fmt.Printf("  Leader write: %s (ok=%v)\n", msg, ok)

	// Partitioned node cannot read.
	_, _, msg = cpNodes[2].Read("user:1")
	fmt.Printf("  Partitioned node read: %s\n", msg)

	// Partitioned node cannot write.
	ok, msg = cpNodes[2].Write("user:3", "Carol")
	fmt.Printf("  Partitioned node write: %s (ok=%v)\n", msg, ok)

	// --- AP System ---
	fmt.Println("\n--- AP System (like DynamoDB/Cassandra) ---\n")

	apNodes := make([]*KVNode, 3)
	for i := 0; i < 3; i++ {
		apNodes[i] = NewKVNode(fmt.Sprintf("AP-%d", i), APModel)
	}
	for i := range apNodes {
		for j := range apNodes {
			if i != j {
				apNodes[i].peers = append(apNodes[i].peers, apNodes[j])
			}
		}
	}

	// Write before partition.
	ok, msg = apNodes[0].Write("config", "v1")
	fmt.Printf("  Before partition: %s (ok=%v)\n", msg, ok)

	// Create partition: node 2 is isolated.
	fmt.Println("\n  --- Network Partition ---")
	apNodes[2].partitioned = true

	// Both sides can write -- but they diverge!
	ok, msg = apNodes[0].Write("config", "v2-from-partition-A")
	fmt.Printf("  Partition A write: %s (ok=%v)\n", msg, ok)

	ok, msg = apNodes[2].Write("config", "v2-from-partition-B")
	fmt.Printf("  Partition B write: %s (ok=%v)\n", msg, ok)

	// Both sides can read -- but they see different values!
	val, _, msg := apNodes[0].Read("config")
	fmt.Printf("  Partition A read: %s -> value=%q\n", msg, val)

	val, _, msg = apNodes[2].Read("config")
	fmt.Printf("  Partition B read: %s -> value=%q\n", msg, val)

	fmt.Println("\n  After partition heals, AP system must reconcile:")
	fmt.Printf("  Node 0 has: %q\n", apNodes[0].data["config"])
	fmt.Printf("  Node 2 has: %q\n", apNodes[2].data["config"])
	fmt.Println("  Which value wins? Last-writer-wins? Merge? Application-specific!")

	fmt.Println("\n=== Summary ===")
	fmt.Println("CP: Refuses some requests during partition (maintains consistency)")
	fmt.Println("AP: Serves all requests during partition (may return stale/divergent data)")
	fmt.Println("Neither is better -- it depends on your application's requirements.")

	_ = time.Now() // Avoid unused import.
}
```

### Beyond CAP: PACELC

CAP only describes behavior during a partition. The PACELC theorem extends it:

- **P**artition: Choose **A**vailability or **C**onsistency
- **E**lse (no partition): Choose **L**atency or **C**onsistency

```
System          During Partition (PAC)    Normal Operation (ELC)
──────          ─────────────────────     ─────────────────────
DynamoDB        PA (available)            EL (low latency, eventual consistency)
Cassandra       PA (available)            EL (tunable consistency)
MongoDB         PC (consistent)           EC (consistent reads from primary)
etcd/ZooKeeper  PC (consistent)           EC (linearizable)
CockroachDB     PC (consistent)           EC (serializable)
```

---

## 12. Exactly-Once Semantics

### The Delivery Guarantee Spectrum

```
At-most-once:   Send and forget. Message may be lost.
                Fire and forget. Simple, but unreliable.

At-least-once:  Retry until acknowledged. Message may be delivered multiple times.
                Safe if operations are idempotent.

Exactly-once:   Each message is processed exactly once.
                Actually means: at-least-once delivery + idempotent processing.
```

True exactly-once delivery is impossible in the general case (it would require solving the Two Generals Problem). What systems actually implement is **effectively exactly-once**: at-least-once delivery combined with deduplication.

### Idempotency Keys

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// Response represents the result of processing a request.
type Response struct {
	Status  string
	Data    string
	Created time.Time
}

// IdempotencyStore ensures each request is processed at most once.
type IdempotencyStore struct {
	processed map[string]Response
	mu        sync.RWMutex
	ttl       time.Duration
}

// NewIdempotencyStore creates a new store with the given TTL for entries.
func NewIdempotencyStore(ttl time.Duration) *IdempotencyStore {
	return &IdempotencyStore{
		processed: make(map[string]Response),
		ttl:       ttl,
	}
}

// ProcessOnce executes fn only if the key has not been seen before.
// If the key was already processed, returns the cached response.
func (is *IdempotencyStore) ProcessOnce(key string, fn func() Response) Response {
	// Fast path: check if already processed (read lock).
	is.mu.RLock()
	if resp, ok := is.processed[key]; ok {
		is.mu.RUnlock()
		fmt.Printf("  [IDEMP] Key %s: returning CACHED response\n", key[:12])
		return resp
	}
	is.mu.RUnlock()

	// Slow path: acquire write lock and double-check.
	is.mu.Lock()
	defer is.mu.Unlock()

	// Double-check after acquiring write lock.
	if resp, ok := is.processed[key]; ok {
		fmt.Printf("  [IDEMP] Key %s: returning CACHED response (race avoided)\n", key[:12])
		return resp
	}

	// Process the request.
	fmt.Printf("  [IDEMP] Key %s: PROCESSING new request\n", key[:12])
	resp := fn()
	resp.Created = time.Now()
	is.processed[key] = resp
	return resp
}

// Cleanup removes expired entries.
func (is *IdempotencyStore) Cleanup() int {
	is.mu.Lock()
	defer is.mu.Unlock()

	cutoff := time.Now().Add(-is.ttl)
	removed := 0
	for key, resp := range is.processed {
		if resp.Created.Before(cutoff) {
			delete(is.processed, key)
			removed++
		}
	}
	return removed
}

// Size returns the number of stored responses.
func (is *IdempotencyStore) Size() int {
	is.mu.RLock()
	defer is.mu.RUnlock()
	return len(is.processed)
}

// generateIdempotencyKey creates a unique key for a request.
func generateIdempotencyKey() string {
	b := make([]byte, 16)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// PaymentService simulates a payment processing service.
type PaymentService struct {
	store    *IdempotencyStore
	balance  map[string]float64
	mu       sync.Mutex
}

// NewPaymentService creates a new payment service.
func NewPaymentService() *PaymentService {
	return &PaymentService{
		store:   NewIdempotencyStore(24 * time.Hour),
		balance: map[string]float64{"alice": 1000, "bob": 500},
	}
}

// Transfer processes a money transfer with idempotency protection.
func (ps *PaymentService) Transfer(idempotencyKey, from, to string, amount float64) Response {
	return ps.store.ProcessOnce(idempotencyKey, func() Response {
		ps.mu.Lock()
		defer ps.mu.Unlock()

		if ps.balance[from] < amount {
			return Response{
				Status: "error",
				Data:   fmt.Sprintf("insufficient funds: %s has %.2f", from, ps.balance[from]),
			}
		}

		ps.balance[from] -= amount
		ps.balance[to] += amount

		return Response{
			Status: "success",
			Data: fmt.Sprintf("transferred %.2f from %s to %s (balances: %s=%.2f, %s=%.2f)",
				amount, from, to, from, ps.balance[from], to, ps.balance[to]),
		}
	})
}

func main() {
	fmt.Println("=== Idempotency Key Demonstration ===\n")

	svc := NewPaymentService()

	// Generate one idempotency key for this transfer.
	key := generateIdempotencyKey()
	fmt.Printf("Idempotency key: %s\n\n", key)

	// First attempt: processes the transfer.
	fmt.Println("--- Attempt 1 (first try) ---")
	resp1 := svc.Transfer(key, "alice", "bob", 100)
	fmt.Printf("  Result: %s - %s\n\n", resp1.Status, resp1.Data)

	// Second attempt: same key, returns cached response.
	// This simulates a network retry after timeout.
	fmt.Println("--- Attempt 2 (retry after timeout) ---")
	resp2 := svc.Transfer(key, "alice", "bob", 100)
	fmt.Printf("  Result: %s - %s\n\n", resp2.Status, resp2.Data)

	// Third attempt: same key again.
	fmt.Println("--- Attempt 3 (another retry) ---")
	resp3 := svc.Transfer(key, "alice", "bob", 100)
	fmt.Printf("  Result: %s - %s\n\n", resp3.Status, resp3.Data)

	// The transfer was only processed ONCE, despite three attempts.
	// Alice has 900, Bob has 600.

	// New transfer with a different key.
	key2 := generateIdempotencyKey()
	fmt.Println("--- New transfer (different key) ---")
	resp4 := svc.Transfer(key2, "alice", "bob", 50)
	fmt.Printf("  Result: %s - %s\n\n", resp4.Status, resp4.Data)

	fmt.Printf("Idempotency store size: %d entries\n", svc.store.Size())
	fmt.Println("\nThe first transfer was executed ONCE despite 3 attempts.")
	fmt.Println("This is 'effectively exactly-once' semantics.")

	// Demonstrate concurrent retries.
	fmt.Println("\n=== Concurrent Retries ===\n")
	key3 := generateIdempotencyKey()
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(attempt int) {
			defer wg.Done()
			resp := svc.Transfer(key3, "alice", "bob", 25)
			fmt.Printf("  Goroutine %d: %s\n", attempt, resp.Status)
		}(i)
	}

	wg.Wait()
	fmt.Printf("\nFinal balances: alice=%.2f, bob=%.2f\n",
		svc.balance["alice"], svc.balance["bob"])
	fmt.Println("Even with 10 concurrent retries, transfer happened exactly ONCE.")
}
```

---

## 13. Gossip Protocols

### What Is Gossip?

Gossip protocols (also called epidemic protocols) spread information through a cluster the way rumors spread in a social network: each node periodically tells a random peer what it knows. Information eventually reaches every node, even with failures.

Properties:
- **Scalable**: Each node only communicates with a few peers per round.
- **Fault-tolerant**: Handles node failures and network partitions gracefully.
- **Eventually consistent**: All nodes converge to the same state, but not immediately.

Used in: Cassandra (cluster membership), Consul (service discovery), SWIM (failure detection).

### Gossip Protocol Implementation

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// NodeInfo stores information about a node in the cluster.
type NodeInfo struct {
	ID        string
	Address   string
	State     string // "alive", "suspect", "dead"
	Heartbeat uint64 // Monotonically increasing counter.
	Timestamp time.Time
}

// GossipNode is a node that participates in gossip.
type GossipNode struct {
	id         string
	members    map[string]*NodeInfo // Known cluster members.
	inbox      chan GossipMessage
	peers      map[string]*GossipNode
	heartbeat  uint64
	fanout     int           // Number of peers to gossip with per round.
	interval   time.Duration // How often to gossip.
	mu         sync.RWMutex
	done       chan struct{}
	eventLog   []string
	logMu      sync.Mutex
}

// GossipMessage is exchanged between nodes.
type GossipMessage struct {
	From    string
	Members map[string]*NodeInfo
}

// NewGossipNode creates a new gossip node.
func NewGossipNode(id string, fanout int, interval time.Duration) *GossipNode {
	node := &GossipNode{
		id:       id,
		members:  make(map[string]*NodeInfo),
		inbox:    make(chan GossipMessage, 100),
		peers:    make(map[string]*GossipNode),
		fanout:   fanout,
		interval: interval,
		done:     make(chan struct{}),
	}

	// Add self to member list.
	node.members[id] = &NodeInfo{
		ID:        id,
		Address:   fmt.Sprintf("127.0.0.1:%d", 8000+rand.Intn(1000)),
		State:     "alive",
		Heartbeat: 0,
		Timestamp: time.Now(),
	}

	return node
}

func (n *GossipNode) logEvent(format string, args ...interface{}) {
	entry := fmt.Sprintf("[%s] %s", n.id, fmt.Sprintf(format, args...))
	n.logMu.Lock()
	n.eventLog = append(n.eventLog, entry)
	n.logMu.Unlock()
}

// Start begins the gossip protocol.
func (n *GossipNode) Start() {
	// Receiver goroutine.
	go func() {
		for {
			select {
			case <-n.done:
				return
			case msg := <-n.inbox:
				n.handleGossip(msg)
			}
		}
	}()

	// Gossip sender goroutine.
	go func() {
		ticker := time.NewTicker(n.interval)
		defer ticker.Stop()
		for {
			select {
			case <-n.done:
				return
			case <-ticker.C:
				n.gossipRound()
			}
		}
	}()
}

// Stop shuts down the gossip node.
func (n *GossipNode) Stop() {
	close(n.done)
}

// gossipRound picks random peers and sends them our member list.
func (n *GossipNode) gossipRound() {
	// Increment own heartbeat.
	n.mu.Lock()
	n.heartbeat++
	if info, ok := n.members[n.id]; ok {
		info.Heartbeat = n.heartbeat
		info.Timestamp = time.Now()
	}

	// Copy member list for sending.
	membersCopy := make(map[string]*NodeInfo)
	for k, v := range n.members {
		copied := *v
		membersCopy[k] = &copied
	}

	// Get peer list.
	peerIDs := make([]string, 0, len(n.peers))
	for id := range n.peers {
		peerIDs = append(peerIDs, id)
	}
	n.mu.Unlock()

	if len(peerIDs) == 0 {
		return
	}

	// Shuffle and pick fanout peers.
	rand.Shuffle(len(peerIDs), func(i, j int) {
		peerIDs[i], peerIDs[j] = peerIDs[j], peerIDs[i]
	})

	count := n.fanout
	if count > len(peerIDs) {
		count = len(peerIDs)
	}

	for _, peerID := range peerIDs[:count] {
		n.mu.RLock()
		peer := n.peers[peerID]
		n.mu.RUnlock()

		if peer != nil {
			msg := GossipMessage{From: n.id, Members: membersCopy}
			select {
			case peer.inbox <- msg:
			default:
				// Peer's inbox is full; skip.
			}
		}
	}
}

// handleGossip merges received member information with local state.
func (n *GossipNode) handleGossip(msg GossipMessage) {
	n.mu.Lock()
	defer n.mu.Unlock()

	for id, remoteInfo := range msg.Members {
		localInfo, exists := n.members[id]

		if !exists {
			// New member discovered!
			copied := *remoteInfo
			n.members[id] = &copied
			n.logEvent("discovered new member: %s (via %s)", id, msg.From)
			continue
		}

		// Update if remote has newer information.
		if remoteInfo.Heartbeat > localInfo.Heartbeat {
			localInfo.Heartbeat = remoteInfo.Heartbeat
			localInfo.State = remoteInfo.State
			localInfo.Timestamp = time.Now()
		}
	}
}

// InjectData adds custom data to this node's member list (simulating
// a new node joining).
func (n *GossipNode) InjectMember(info *NodeInfo) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.members[info.ID] = info
	n.logEvent("injected member: %s", info.ID)
}

// GetMembers returns a snapshot of known members.
func (n *GossipNode) GetMembers() map[string]NodeInfo {
	n.mu.RLock()
	defer n.mu.RUnlock()
	result := make(map[string]NodeInfo)
	for k, v := range n.members {
		result[k] = *v
	}
	return result
}

func main() {
	rand.Seed(time.Now().UnixNano())

	fmt.Println("=== Gossip Protocol Demonstration ===\n")

	// Create a cluster of 6 nodes.
	nodeIDs := []string{"N1", "N2", "N3", "N4", "N5", "N6"}
	nodes := make(map[string]*GossipNode)

	for _, id := range nodeIDs {
		nodes[id] = NewGossipNode(id, 2, 100*time.Millisecond) // Fanout=2, gossip every 100ms.
	}

	// Connect nodes in a ring + some random extra connections.
	// Each node only knows a few peers (not all).
	for i, id := range nodeIDs {
		nextIdx := (i + 1) % len(nodeIDs)
		nextID := nodeIDs[nextIdx]
		nodes[id].peers[nextID] = nodes[nextID]
		nodes[nextID].peers[id] = nodes[id]
	}

	// At the start, each node only knows about itself.
	fmt.Println("Initial state: Each node knows only about itself.")
	for _, id := range nodeIDs {
		members := nodes[id].GetMembers()
		fmt.Printf("  %s knows about: ", id)
		for mid := range members {
			fmt.Printf("%s ", mid)
		}
		fmt.Println()
	}

	// Start gossip on all nodes.
	fmt.Println("\nStarting gossip protocol...")
	for _, node := range nodes {
		node.Start()
	}

	// Wait for information to spread.
	rounds := []time.Duration{
		200 * time.Millisecond,
		400 * time.Millisecond,
		800 * time.Millisecond,
		1500 * time.Millisecond,
	}

	for _, wait := range rounds {
		time.Sleep(wait)
		fmt.Printf("\n--- After %v ---\n", wait)
		allComplete := true
		for _, id := range nodeIDs {
			members := nodes[id].GetMembers()
			fmt.Printf("  %s knows %d/%d members: ", id, len(members), len(nodeIDs))
			for mid := range members {
				fmt.Printf("%s ", mid)
			}
			fmt.Println()
			if len(members) < len(nodeIDs) {
				allComplete = false
			}
		}
		if allComplete {
			fmt.Println("\n  All nodes have FULL membership knowledge!")
			break
		}
	}

	// Now inject a new member via N1 and watch it spread.
	fmt.Println("\n=== New node N7 joins via N1 ===")
	nodes["N1"].InjectMember(&NodeInfo{
		ID:        "N7",
		Address:   "127.0.0.1:9007",
		State:     "alive",
		Heartbeat: 1,
		Timestamp: time.Now(),
	})

	time.Sleep(500 * time.Millisecond)

	fmt.Println("\nAfter 500ms, who knows about N7?")
	for _, id := range nodeIDs {
		members := nodes[id].GetMembers()
		_, knowsN7 := members["N7"]
		fmt.Printf("  %s: knows N7 = %v (total members: %d)\n", id, knowsN7, len(members))
	}

	time.Sleep(1 * time.Second)

	fmt.Println("\nAfter 1.5s total:")
	for _, id := range nodeIDs {
		members := nodes[id].GetMembers()
		_, knowsN7 := members["N7"]
		fmt.Printf("  %s: knows N7 = %v\n", id, knowsN7)
	}

	// Stop all nodes.
	for _, node := range nodes {
		node.Stop()
	}

	// Print discovery logs.
	fmt.Println("\n=== Discovery Event Log ===")
	for _, id := range nodeIDs {
		nodes[id].logMu.Lock()
		for _, entry := range nodes[id].eventLog {
			fmt.Printf("  %s\n", entry)
		}
		nodes[id].logMu.Unlock()
	}

	fmt.Println("\nGossip spreads information exponentially -- like a rumor.")
	fmt.Println("Each round, every node tells 2 random peers.")
	fmt.Println("In O(log N) rounds, the entire cluster is informed.")
}
```

### Gossip Protocol Properties

```
Property              Value
────────              ─────
Convergence time      O(log N) rounds for N nodes
Message complexity    O(N * fanout) per round
Fault tolerance       Tolerates up to N-1 failures
Consistency           Eventual (all nodes converge)
Scalability           Excellent (each node has fixed cost per round)
```

---

## 14. Real-World Example: Building a Distributed Lock Service

This section brings together many of the concepts from this chapter into a complete, working distributed lock service with:

- Raft-based consensus for lock state
- Fencing tokens for safety
- TTL-based automatic expiry
- Heartbeat-based failure detection
- Client retry with idempotency

```go
package main

import (
	"errors"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// --- Errors ---

var (
	ErrNotLeader      = errors.New("not the leader")
	ErrLockNotHeld    = errors.New("lock not held")
	ErrLockBusy       = errors.New("lock held by another client")
	ErrStaleToken     = errors.New("stale fencing token")
	ErrLockExpired    = errors.New("lock expired")
)

// --- Lock State ---

// LockState represents the current state of a distributed lock.
type LockState struct {
	Name         string
	Holder       string
	FencingToken uint64
	ExpiresAt    time.Time
	AcquiredAt   time.Time
}

// IsHeld returns true if the lock is currently held and not expired.
func (ls *LockState) IsHeld() bool {
	return ls.Holder != "" && time.Now().Before(ls.ExpiresAt)
}

// --- Lock Command ---

// LockCommand represents an operation on the lock service.
type LockCommand struct {
	Type      string // "acquire", "release", "renew"
	LockName  string
	ClientID  string
	TTL       time.Duration
	RequestID string // For idempotency.
}

// LockResult is the result of a lock command.
type LockResult struct {
	Success      bool
	FencingToken uint64
	Error        error
	ExpiresAt    time.Time
}

// --- Lock Service Node ---

// LockServiceNode is one node in the distributed lock service.
type LockServiceNode struct {
	id           string
	isLeader     bool
	term         uint64
	locks        map[string]*LockState
	tokenCounter uint64
	processedReq map[string]LockResult // Idempotency store.
	peers        []*LockServiceNode
	inbox        chan replicationMsg
	mu           sync.Mutex

	// Failure detection.
	heartbeats map[string]time.Time
	failureTimeout time.Duration
}

type replicationMsg struct {
	cmd   LockCommand
	reply chan LockResult
}

// NewLockServiceNode creates a new lock service node.
func NewLockServiceNode(id string) *LockServiceNode {
	return &LockServiceNode{
		id:             id,
		locks:          make(map[string]*LockState),
		processedReq:   make(map[string]LockResult),
		inbox:          make(chan replicationMsg, 100),
		heartbeats:     make(map[string]time.Time),
		failureTimeout: 5 * time.Second,
	}
}

// ProcessCommand handles a lock command (only on the leader).
func (n *LockServiceNode) ProcessCommand(cmd LockCommand) LockResult {
	n.mu.Lock()
	defer n.mu.Unlock()

	if !n.isLeader {
		return LockResult{Success: false, Error: ErrNotLeader}
	}

	// Idempotency check.
	if result, ok := n.processedReq[cmd.RequestID]; ok {
		return result
	}

	var result LockResult

	switch cmd.Type {
	case "acquire":
		result = n.acquireLock(cmd)
	case "release":
		result = n.releaseLock(cmd)
	case "renew":
		result = n.renewLock(cmd)
	default:
		result = LockResult{Success: false, Error: fmt.Errorf("unknown command: %s", cmd.Type)}
	}

	// Store for idempotency.
	n.processedReq[cmd.RequestID] = result

	// Replicate to peers (simplified: synchronous replication).
	n.replicateToPeers(cmd)

	return result
}

func (n *LockServiceNode) acquireLock(cmd LockCommand) LockResult {
	lock, exists := n.locks[cmd.LockName]

	if exists && lock.IsHeld() {
		if lock.Holder == cmd.ClientID {
			// Re-entrant acquire: refresh TTL.
			lock.ExpiresAt = time.Now().Add(cmd.TTL)
			return LockResult{
				Success:      true,
				FencingToken: lock.FencingToken,
				ExpiresAt:    lock.ExpiresAt,
			}
		}
		return LockResult{Success: false, Error: ErrLockBusy}
	}

	// Lock is available.
	n.tokenCounter++
	newLock := &LockState{
		Name:         cmd.LockName,
		Holder:       cmd.ClientID,
		FencingToken: n.tokenCounter,
		ExpiresAt:    time.Now().Add(cmd.TTL),
		AcquiredAt:   time.Now(),
	}
	n.locks[cmd.LockName] = newLock

	fmt.Printf("  [%s] ACQUIRED lock %q by %s (token=%d, ttl=%v)\n",
		n.id, cmd.LockName, cmd.ClientID, newLock.FencingToken, cmd.TTL)

	return LockResult{
		Success:      true,
		FencingToken: newLock.FencingToken,
		ExpiresAt:    newLock.ExpiresAt,
	}
}

func (n *LockServiceNode) releaseLock(cmd LockCommand) LockResult {
	lock, exists := n.locks[cmd.LockName]
	if !exists || !lock.IsHeld() {
		return LockResult{Success: false, Error: ErrLockNotHeld}
	}

	if lock.Holder != cmd.ClientID {
		return LockResult{Success: false, Error: ErrLockBusy}
	}

	fmt.Printf("  [%s] RELEASED lock %q by %s (token=%d)\n",
		n.id, cmd.LockName, cmd.ClientID, lock.FencingToken)

	lock.Holder = ""
	return LockResult{Success: true, FencingToken: lock.FencingToken}
}

func (n *LockServiceNode) renewLock(cmd LockCommand) LockResult {
	lock, exists := n.locks[cmd.LockName]
	if !exists || !lock.IsHeld() {
		return LockResult{Success: false, Error: ErrLockExpired}
	}

	if lock.Holder != cmd.ClientID {
		return LockResult{Success: false, Error: ErrLockBusy}
	}

	lock.ExpiresAt = time.Now().Add(cmd.TTL)
	fmt.Printf("  [%s] RENEWED lock %q by %s (token=%d, new ttl=%v)\n",
		n.id, cmd.LockName, cmd.ClientID, lock.FencingToken, cmd.TTL)

	return LockResult{
		Success:      true,
		FencingToken: lock.FencingToken,
		ExpiresAt:    lock.ExpiresAt,
	}
}

func (n *LockServiceNode) replicateToPeers(cmd LockCommand) {
	for _, peer := range n.peers {
		go func(p *LockServiceNode) {
			p.mu.Lock()
			defer p.mu.Unlock()
			// Apply the same command on the follower.
			switch cmd.Type {
			case "acquire":
				p.acquireLock(cmd)
			case "release":
				p.releaseLock(cmd)
			case "renew":
				p.renewLock(cmd)
			}
		}(peer)
	}
}

// RecordHeartbeat records that a client is alive.
func (n *LockServiceNode) RecordHeartbeat(clientID string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	n.heartbeats[clientID] = time.Now()
}

// CheckExpiredLocks releases locks held by clients that have stopped
// sending heartbeats.
func (n *LockServiceNode) CheckExpiredLocks() {
	n.mu.Lock()
	defer n.mu.Unlock()

	now := time.Now()
	for name, lock := range n.locks {
		if lock.IsHeld() && now.After(lock.ExpiresAt) {
			fmt.Printf("  [%s] Lock %q EXPIRED (was held by %s, token=%d)\n",
				n.id, name, lock.Holder, lock.FencingToken)
			lock.Holder = ""
		}
	}
}

// GetLockInfo returns the current state of a lock.
func (n *LockServiceNode) GetLockInfo(name string) (*LockState, bool) {
	n.mu.Lock()
	defer n.mu.Unlock()
	lock, ok := n.locks[name]
	if ok {
		copied := *lock
		return &copied, true
	}
	return nil, false
}

// --- Client ---

// LockClient is a client that interacts with the lock service.
type LockClient struct {
	clientID string
	leader   *LockServiceNode
	reqSeq   uint64
	mu       sync.Mutex
}

// NewLockClient creates a new lock client.
func NewLockClient(clientID string, leader *LockServiceNode) *LockClient {
	return &LockClient{
		clientID: clientID,
		leader:   leader,
	}
}

func (c *LockClient) nextRequestID() string {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.reqSeq++
	return fmt.Sprintf("%s-req-%d", c.clientID, c.reqSeq)
}

// Acquire tries to acquire a lock with retries.
func (c *LockClient) Acquire(lockName string, ttl time.Duration, maxRetries int) (uint64, error) {
	reqID := c.nextRequestID()

	for attempt := 0; attempt <= maxRetries; attempt++ {
		result := c.leader.ProcessCommand(LockCommand{
			Type:      "acquire",
			LockName:  lockName,
			ClientID:  c.clientID,
			TTL:       ttl,
			RequestID: reqID,
		})

		if result.Success {
			return result.FencingToken, nil
		}

		if result.Error == ErrLockBusy && attempt < maxRetries {
			backoff := time.Duration(50*(attempt+1)) * time.Millisecond
			fmt.Printf("  [CLIENT %s] Lock busy, retrying in %v (attempt %d/%d)\n",
				c.clientID, backoff, attempt+1, maxRetries)
			time.Sleep(backoff)
			continue
		}

		return 0, result.Error
	}

	return 0, ErrLockBusy
}

// Release releases a lock.
func (c *LockClient) Release(lockName string) error {
	result := c.leader.ProcessCommand(LockCommand{
		Type:      "release",
		LockName:  lockName,
		ClientID:  c.clientID,
		RequestID: c.nextRequestID(),
	})
	return result.Error
}

// Renew renews a lock's TTL.
func (c *LockClient) Renew(lockName string, ttl time.Duration) error {
	result := c.leader.ProcessCommand(LockCommand{
		Type:      "renew",
		LockName:  lockName,
		ClientID:  c.clientID,
		TTL:       ttl,
		RequestID: c.nextRequestID(),
	})
	if !result.Success {
		return result.Error
	}
	return nil
}

// --- Protected Resource ---

// ProtectedResource uses fencing tokens to ensure correctness.
type ProtectedResource struct {
	lastToken uint64
	data      string
	writeLog  []string
	mu        sync.Mutex
}

// Write writes to the resource, checking the fencing token.
func (pr *ProtectedResource) Write(clientID string, token uint64, data string) error {
	pr.mu.Lock()
	defer pr.mu.Unlock()

	if token < pr.lastToken {
		entry := fmt.Sprintf("REJECTED: %s (token=%d < last=%d)", clientID, token, pr.lastToken)
		pr.writeLog = append(pr.writeLog, entry)
		fmt.Printf("  [RESOURCE] %s\n", entry)
		return ErrStaleToken
	}

	pr.lastToken = token
	pr.data = data
	entry := fmt.Sprintf("ACCEPTED: %s wrote %q (token=%d)", clientID, data, token)
	pr.writeLog = append(pr.writeLog, entry)
	fmt.Printf("  [RESOURCE] %s\n", entry)
	return nil
}

// --- Main ---

func main() {
	rand.Seed(time.Now().UnixNano())

	fmt.Println("=== Distributed Lock Service ===\n")

	// Create a 3-node lock service cluster.
	leader := NewLockServiceNode("leader")
	follower1 := NewLockServiceNode("follower-1")
	follower2 := NewLockServiceNode("follower-2")

	leader.isLeader = true
	leader.term = 1
	leader.peers = []*LockServiceNode{follower1, follower2}

	// Start lock expiry checker.
	go func() {
		ticker := time.NewTicker(100 * time.Millisecond)
		defer ticker.Stop()
		for range ticker.C {
			leader.CheckExpiredLocks()
		}
	}()

	// Create a protected resource (e.g., a shared file or database).
	resource := &ProtectedResource{}

	// --- Scenario 1: Normal lock acquire and release ---
	fmt.Println("--- Scenario 1: Normal Lock Usage ---\n")

	clientA := NewLockClient("client-A", leader)
	tokenA, err := clientA.Acquire("my-lock", 2*time.Second, 3)
	if err != nil {
		fmt.Printf("Client A failed: %v\n", err)
		return
	}
	fmt.Printf("  Client A acquired lock with token=%d\n\n", tokenA)

	resource.Write("client-A", tokenA, "data from A")
	clientA.Release("my-lock")

	// --- Scenario 2: Lock contention ---
	fmt.Println("\n--- Scenario 2: Lock Contention ---\n")

	clientB := NewLockClient("client-B", leader)
	clientC := NewLockClient("client-C", leader)

	// B acquires first.
	tokenB, _ := clientB.Acquire("shared-lock", 500*time.Millisecond, 0)
	fmt.Printf("  Client B acquired lock with token=%d\n", tokenB)

	// C tries to acquire -- will fail (lock is held).
	_, err = clientC.Acquire("shared-lock", 500*time.Millisecond, 2)
	if err != nil {
		fmt.Printf("  Client C: %v (expected -- B holds the lock)\n", err)
	}

	// B releases.
	clientB.Release("shared-lock")

	// Now C can acquire.
	tokenC, err := clientC.Acquire("shared-lock", 500*time.Millisecond, 0)
	if err == nil {
		fmt.Printf("  Client C acquired lock with token=%d\n", tokenC)
	}

	// --- Scenario 3: Lock expiry + fencing token protection ---
	fmt.Println("\n--- Scenario 3: Lock Expiry + Fencing Token ---\n")

	clientD := NewLockClient("client-D", leader)
	clientE := NewLockClient("client-E", leader)

	// D acquires with short TTL.
	tokenD, _ := clientD.Acquire("critical-lock", 300*time.Millisecond, 0)
	fmt.Printf("  Client D acquired lock with token=%d (300ms TTL)\n\n", tokenD)

	// D starts working but is slow.
	fmt.Println("  Client D is working (slowly)...")
	time.Sleep(500 * time.Millisecond) // Lock expires!

	// E acquires the now-expired lock.
	tokenE, _ := clientE.Acquire("critical-lock", 1*time.Second, 0)
	fmt.Printf("  Client E acquired lock with token=%d\n\n", tokenE)

	// E writes to resource.
	resource.Write("client-E", tokenE, "data from E")

	// D tries to write with its old token -- REJECTED by fencing.
	fmt.Println()
	err = resource.Write("client-D", tokenD, "stale data from D")
	if err != nil {
		fmt.Printf("  Client D's stale write correctly rejected: %v\n", err)
	}

	// --- Scenario 4: Idempotent retries ---
	fmt.Println("\n--- Scenario 4: Idempotent Retries ---\n")

	clientF := NewLockClient("client-F", leader)
	// Simulate sending the same request 3 times (network retries).
	reqID := "client-F-req-manual"
	for i := 0; i < 3; i++ {
		result := leader.ProcessCommand(LockCommand{
			Type:      "acquire",
			LockName:  "idempotent-lock",
			ClientID:  "client-F",
			TTL:       1 * time.Second,
			RequestID: reqID, // Same request ID every time.
		})
		fmt.Printf("  Attempt %d: success=%v token=%d\n", i+1, result.Success, result.FencingToken)
	}
	fmt.Println("  All 3 attempts returned the same token (idempotent).")

	_ = clientF // Keep the compiler happy.

	// Final state.
	fmt.Println("\n--- Final Lock States ---")
	for _, name := range []string{"my-lock", "shared-lock", "critical-lock", "idempotent-lock"} {
		if lock, ok := leader.GetLockInfo(name); ok {
			held := "free"
			if lock.IsHeld() {
				held = fmt.Sprintf("held by %s (token=%d, expires in %v)",
					lock.Holder, lock.FencingToken, time.Until(lock.ExpiresAt).Round(time.Millisecond))
			}
			fmt.Printf("  %s: %s\n", name, held)
		}
	}

	fmt.Println("\n--- Resource Write Log ---")
	resource.mu.Lock()
	for i, entry := range resource.writeLog {
		fmt.Printf("  %d. %s\n", i+1, entry)
	}
	fmt.Printf("  Final data: %q\n", resource.data)
	resource.mu.Unlock()
}
```

### What This System Demonstrates

1. **Consensus** -- The leader replicates lock state to followers (simplified Raft).
2. **Fencing tokens** -- Monotonically increasing tokens prevent stale writes.
3. **TTL-based expiry** -- Locks automatically expire if the holder crashes or is slow.
4. **Idempotency** -- Retried requests return the same result without double-processing.
5. **Failure detection** -- Periodic checks release expired locks.

In production, you would use etcd or ZooKeeper for the consensus layer rather than building your own, but the principles are identical.

---

## 15. Key Takeaways

### The Fundamental Truths

1. **Networks are unreliable.** Packets are lost, delayed, duplicated, and reordered. Design every protocol to handle all of these.

2. **Clocks cannot be trusted across machines.** Wall clocks drift. NTP helps but leaves millisecond-level uncertainty. Use logical clocks (Lamport or vector) for causal ordering.

3. **Partial failure is the norm.** In a distributed system, some nodes are up while others are down. Your system must make progress despite partial failure.

4. **Failure detection is probabilistic.** You cannot distinguish a crashed node from a slow one. Use timeouts and heartbeats, but accept that you will sometimes be wrong.

5. **Consensus is expensive but necessary.** Getting nodes to agree requires multiple rounds of communication and is blocked by partitions. Use it only where you must (leader election, metadata, coordination).

### Which Tool for Which Problem

```
Problem                          Solution
───────                          ────────
Ordering events across nodes     Lamport timestamps, vector clocks
Detecting concurrent updates     Vector clocks
Electing a leader                Raft, Bully algorithm
Replicating state consistently   Raft, Paxos
Detecting failures               Heartbeats, phi accrual detector
Preventing stale writes          Fencing tokens
Handling retries safely          Idempotency keys
Spreading information            Gossip protocols
Distributed locking              Fencing tokens + TTL + consensus
Choosing CP vs AP                Depends on your application requirements
```

### Go-Specific Lessons

- **Goroutines as nodes** -- Go's goroutines and channels naturally model distributed systems. Each goroutine is a "node" with its own state, communicating via message passing.
- **`sync.Mutex` vs channels** -- Use mutexes for protecting shared state within a node. Use channels for communication between nodes.
- **`time.Since()` is safe** -- It uses the monotonic clock. Use it for timeouts and duration measurement.
- **`time.Now()` for timestamps** -- Fine for logging, but not for cross-machine ordering.
- **Context for cancellation** -- Use `context.Context` to propagate deadlines and cancellation across goroutine boundaries, just as you would across network boundaries.

### Design Principles

1. **Design for failure.** Assume every network call can fail. Implement retries with backoff and idempotency.
2. **Use fencing tokens.** Any time a lease or lock can expire, use fencing tokens to prevent stale operations.
3. **Prefer idempotent operations.** Make every operation safe to retry. Store results keyed by request ID.
4. **Use consensus sparingly.** Consensus is slow and expensive. Use it for metadata and coordination, not for every data operation.
5. **Embrace eventual consistency where possible.** Most user-facing operations do not require strong consistency. Use AP where you can, CP where you must.

---

## 16. Practice Exercises

### Exercise 1: Implement a Network Partition Simulator
Build a network simulator where you can:
- Create a cluster of N nodes.
- Send messages between any two nodes.
- Create arbitrary partitions (e.g., {A,B} vs {C,D,E}).
- Measure message delivery rates during and after partition healing.
- Verify that messages sent during a partition are either delivered after healing or lost (depending on your buffering strategy).

### Exercise 2: Hybrid Logical Clock (HLC)
Implement a Hybrid Logical Clock that combines physical and logical timestamps:
- HLC.Now() returns (physical_time, logical_counter).
- When sending a message, include the HLC timestamp.
- When receiving a message, merge: physical = max(local_physical, msg_physical, wall_clock), and adjust the logical counter.
- Demonstrate that HLC preserves causality while staying close to wall clock time.

### Exercise 3: SWIM Failure Detector
Implement the SWIM (Scalable Weakly-consistent Infection-style process group Membership) protocol:
- Each node periodically pings a random peer.
- If the ping fails, ask K other nodes to ping the target (indirect probe).
- If all indirect probes fail, mark the node as suspect.
- After a timeout, move suspect nodes to dead.
- Use gossip to disseminate membership changes.

### Exercise 4: Multi-Decree Paxos
Implement a simplified Multi-Decree Paxos:
- A proposer sends Prepare(n) to all acceptors.
- Acceptors respond with Promise(n, previously_accepted_value).
- Proposer sends Accept(n, value) to all acceptors.
- Acceptors respond with Accepted(n, value).
- Implement multiple rounds and handle competing proposers.

### Exercise 5: Conflict-Free Replicated Data Type (CRDT)
Build a G-Counter (grow-only counter) CRDT:
- Each node maintains its own counter.
- The value of the counter is the sum of all node counters.
- Merge is element-wise max.
- Demonstrate that concurrent increments on different nodes converge to the correct total without coordination.

Then extend it to a PN-Counter (positive-negative counter) that supports both increment and decrement.

### Exercise 6: Raft Log Compaction
Extend the Raft implementation from Section 9:
- Implement log compaction (snapshotting).
- When the log grows beyond a threshold, take a snapshot of the state machine.
- Discard log entries that are covered by the snapshot.
- Implement InstallSnapshot RPC for slow followers that fall behind.

### Exercise 7: Distributed Rate Limiter
Build a distributed rate limiter that works across multiple nodes:
- Each node tracks local request counts.
- Nodes periodically share their counts via gossip.
- The rate limit is enforced globally (total across all nodes).
- Handle the case where a node's gossip is delayed -- should you allow slightly over the limit or risk being too conservative?

### Exercise 8: Split-Brain Detection
Build a system that detects split-brain scenarios:
- A cluster of N nodes using heartbeats for failure detection.
- When a partition occurs, both sides might elect a leader.
- Implement a mechanism (e.g., disk-based witness, external arbiter) that prevents both sides from operating simultaneously.
- Test with various partition topologies.

### Exercise 9: Exactly-Once Message Queue
Build a simple message queue with exactly-once delivery semantics:
- Producers can send messages with an idempotency key.
- Consumers acknowledge messages after processing.
- If a consumer crashes before acknowledging, the message is redelivered.
- Use idempotency keys to ensure consumers process each message at most once.
- Handle the case where the queue itself is replicated across nodes.

### Exercise 10: End-to-End Distributed System
Combine everything into a distributed key-value store:
- Raft for consensus on metadata (which node owns which key range).
- Consistent hashing for data distribution.
- Vector clocks for conflict detection.
- Gossip for membership management.
- Fencing tokens for distributed locking.
- Idempotency keys for client retries.
- Test under network partitions, node failures, and concurrent writes.

---

**Next Chapter**: Chapter 39 will cover **Consistency and Consensus** -- diving deeper into linearizability, serializability, two-phase commit, and the theory behind Raft and Paxos, all implemented in Go.
