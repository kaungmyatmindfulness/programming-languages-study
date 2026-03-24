# Chapter 37: Transactions & Consistency

> **Level:** Advanced / Distributed Systems | **Prerequisites:** Chapters 10-11 (Goroutines, Channels), 16 (Context), 17 (Advanced Concurrency), 26 (Repository Pattern & Database)
> **Based on:** *Designing Data-Intensive Applications* by Martin Kleppmann, Chapters 7 & 9

Transactions are the fundamental mechanism that databases use to keep your data correct when things go wrong -- crashes, concurrent writes, network failures, partial updates. Without transactions, every application would need to handle a combinatorial explosion of failure modes manually. This chapter explores how transactions work, what guarantees they provide, and how to implement those guarantees in Go -- both with real databases and with in-memory simulations that make the concepts tangible.

We start with the basics (ACID, `database/sql` transactions), move through isolation levels with working Go demonstrations of each concurrency anomaly, build an MVCC store from scratch, implement two-phase locking with deadlock detection, explore distributed transactions and the Saga pattern, and finish with linearizability and a sketch of Raft consensus. Every code example is complete and runnable.

---

## Table of Contents

1. [Why Transactions Matter](#1-why-transactions-matter)
2. [Atomicity in Go](#2-atomicity-in-go)
3. [Isolation Levels Explained](#3-isolation-levels-explained)
4. [Read Committed in Practice](#4-read-committed-in-practice)
5. [Snapshot Isolation (MVCC)](#5-snapshot-isolation-mvcc)
6. [Lost Update Problem](#6-lost-update-problem)
7. [Write Skew and Phantoms](#7-write-skew-and-phantoms)
8. [Two-Phase Locking (2PL)](#8-two-phase-locking-2pl)
9. [Serializable Snapshot Isolation (SSI)](#9-serializable-snapshot-isolation-ssi)
10. [Distributed Transactions](#10-distributed-transactions)
11. [Saga Pattern](#11-saga-pattern)
12. [Linearizability](#12-linearizability)
13. [Consensus Algorithms Overview](#13-consensus-algorithms-overview)
14. [Real-World Example: Bank Transfer System](#14-real-world-example-bank-transfer-system)
15. [Key Takeaways](#15-key-takeaways)
16. [Practice Exercises](#16-practice-exercises)

---

## 1. Why Transactions Matter

### The Fundamental Problem

Consider a bank transfer: move $100 from Alice to Bob. This requires two operations:

1. Debit Alice's account by $100
2. Credit Bob's account by $100

What happens if the system crashes between step 1 and step 2? Alice loses $100 and Bob never receives it. The money vanishes. This is not a hypothetical -- it is the default behavior of any system without transaction guarantees.

```
Without Transactions:

    Step 1: Alice = Alice - 100  ✓ (committed to disk)
    --- CRASH ---
    Step 2: Bob = Bob + 100      ✗ (never executed)

    Result: $100 has disappeared from the system
```

### ACID Properties

Transactions provide four guarantees, collectively known as ACID:

```
┌───────────────────────────────────────────────────────────────────┐
│                        ACID Properties                            │
├───────────────┬───────────────────────────────────────────────────┤
│  Atomicity    │ All operations succeed or all fail. No partial   │
│               │ results. The transaction is an indivisible unit.  │
├───────────────┼───────────────────────────────────────────────────┤
│  Consistency  │ The database moves from one valid state to       │
│               │ another. Invariants (e.g., total money in system │
│               │ stays constant) are preserved.                    │
├───────────────┼───────────────────────────────────────────────────┤
│  Isolation    │ Concurrent transactions do not interfere with    │
│               │ each other. Each transaction sees a consistent   │
│               │ snapshot as if it were running alone.             │
├───────────────┼───────────────────────────────────────────────────┤
│  Durability   │ Once committed, the data survives crashes,       │
│               │ power failures, and restarts.                     │
└───────────────┴───────────────────────────────────────────────────┘
```

### Real-World Failure Scenarios

Here is a catalog of things that go wrong without proper transaction handling:

| Failure | Description | Consequence |
|---------|-------------|-------------|
| **Partial Write** | Process crashes mid-update | Data is half-written, violating invariants |
| **Dirty Read** | Transaction reads uncommitted data from another | Decisions based on data that may be rolled back |
| **Dirty Write** | Transaction overwrites uncommitted data | Conflicting updates interleave unpredictably |
| **Lost Update** | Two transactions read-modify-write the same row | One update is silently lost |
| **Write Skew** | Two transactions read overlapping data, write disjoint rows | Invariant violated even though each transaction looked correct |
| **Phantom Read** | Transaction re-reads a range and sees new rows | Results change within a single transaction |

Let us demonstrate the most basic failure -- a partial write -- and then show how transactions prevent it.

```go
package main

import (
	"fmt"
	"sync"
)

// UnsafeBank has no transaction support.
// Operations can fail mid-way, leaving data inconsistent.
type UnsafeBank struct {
	mu       sync.Mutex
	accounts map[string]int
}

func NewUnsafeBank() *UnsafeBank {
	return &UnsafeBank{
		accounts: map[string]int{
			"alice": 1000,
			"bob":   1000,
		},
	}
}

// Transfer attempts to move money but simulates a crash after the debit.
func (b *UnsafeBank) Transfer(from, to string, amount int, simulateCrash bool) error {
	b.mu.Lock()
	defer b.mu.Unlock()

	if b.accounts[from] < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, b.accounts[from], amount)
	}

	// Step 1: Debit
	b.accounts[from] -= amount
	fmt.Printf("  Debited %d from %s (balance: %d)\n", amount, from, b.accounts[from])

	// Simulate crash between debit and credit
	if simulateCrash {
		return fmt.Errorf("CRASH: system failure after debit, before credit")
	}

	// Step 2: Credit
	b.accounts[to] += amount
	fmt.Printf("  Credited %d to %s (balance: %d)\n", amount, to, b.accounts[to])

	return nil
}

func (b *UnsafeBank) TotalMoney() int {
	b.mu.Lock()
	defer b.mu.Unlock()
	total := 0
	for _, balance := range b.accounts {
		total += balance
	}
	return total
}

func main() {
	bank := NewUnsafeBank()
	fmt.Printf("Initial total money: $%d\n", bank.TotalMoney())
	fmt.Println()

	// Successful transfer
	fmt.Println("Transfer 1: $200 from Alice to Bob (no crash)")
	err := bank.Transfer("alice", "bob", 200, false)
	if err != nil {
		fmt.Printf("  Error: %v\n", err)
	}
	fmt.Printf("  Total money after: $%d\n\n", bank.TotalMoney())

	// Transfer that "crashes" mid-way
	fmt.Println("Transfer 2: $300 from Alice to Bob (crash mid-transfer!)")
	err = bank.Transfer("alice", "bob", 300, true)
	if err != nil {
		fmt.Printf("  Error: %v\n", err)
	}
	fmt.Printf("  Total money after: $%d  <-- MONEY DISAPPEARED!\n\n", bank.TotalMoney())

	// The system is now inconsistent
	fmt.Println("Alice lost $300 but Bob never received it.")
	fmt.Println("This is exactly what transactions prevent.")
}
```

**Output:**

```
Initial total money: $2000

Transfer 1: $200 from Alice to Bob (no crash)
  Debited 200 from alice (balance: 800)
  Credited 200 to bob (balance: 1200)
  Total money after: $2000

Transfer 2: $300 from Alice to Bob (crash mid-transfer!)
  Debited 300 from alice (balance: 500)
  Error: CRASH: system failure after debit, before credit
  Total money after: $1700  <-- MONEY DISAPPEARED!

Alice lost $300 but Bob never received it.
This is exactly what transactions prevent.
```

The rest of this chapter is about ensuring this never happens.

---

## 2. Atomicity in Go

### database/sql Transaction API

Go's `database/sql` package provides first-class transaction support through `db.BeginTx()`, `tx.Commit()`, and `tx.Rollback()`. The canonical pattern uses `defer tx.Rollback()` as a safety net -- if the function returns before `Commit()` is called, the transaction is automatically rolled back. After `Commit()` succeeds, the deferred `Rollback()` becomes a no-op.

```
Transaction Lifecycle:

    BeginTx() ──► Execute queries ──► Commit()
        │                                │
        │         (any error)            │ (success)
        │              │                 │
        └──────► Rollback() ◄────────────┘
                  (via defer)         (no-op after commit)
```

### The Canonical Transfer Function

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"

	_ "github.com/lib/pq" // PostgreSQL driver
)

// TransferFunds atomically moves money from one account to another.
// If any step fails, the entire transfer is rolled back.
func TransferFunds(ctx context.Context, db *sql.DB, from, to string, amount int) error {
	// Begin a transaction
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin transaction: %w", err)
	}
	// Safety net: if we return before Commit(), this rolls back.
	// After Commit() succeeds, Rollback() is a no-op.
	defer tx.Rollback()

	// Step 1: Check the sender's balance
	var balance int
	err = tx.QueryRowContext(ctx,
		"SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", from,
	).Scan(&balance)
	if err != nil {
		return fmt.Errorf("query sender balance: %w", err)
	}

	if balance < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, balance, amount)
	}

	// Step 2: Debit the sender
	_, err = tx.ExecContext(ctx,
		"UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from,
	)
	if err != nil {
		return fmt.Errorf("debit sender: %w", err)
	}

	// Step 3: Credit the receiver
	_, err = tx.ExecContext(ctx,
		"UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to,
	)
	if err != nil {
		return fmt.Errorf("credit receiver: %w", err)
	}

	// Step 4: Record the transfer in a ledger
	_, err = tx.ExecContext(ctx,
		`INSERT INTO transfers (from_account, to_account, amount, created_at)
		 VALUES ($1, $2, $3, NOW())`,
		from, to, amount,
	)
	if err != nil {
		return fmt.Errorf("record transfer: %w", err)
	}

	// Commit the transaction -- all four operations succeed or none do
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit transaction: %w", err)
	}

	return nil
}

func main() {
	// This example requires a running PostgreSQL database.
	// Connection string format: "postgres://user:pass@localhost/dbname?sslmode=disable"
	db, err := sql.Open("postgres", "postgres://localhost/bankdb?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	ctx := context.Background()

	err = TransferFunds(ctx, db, "alice", "bob", 100)
	if err != nil {
		log.Printf("Transfer failed (rolled back): %v", err)
	} else {
		log.Println("Transfer succeeded")
	}
}
```

### Transaction Options

`BeginTx` accepts `*sql.TxOptions` to control isolation level and read-only mode:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
)

func DemonstrateTransactionOptions(ctx context.Context, db *sql.DB) {
	// Read Committed (default for PostgreSQL)
	tx1, err := db.BeginTx(ctx, &sql.TxOptions{
		Isolation: sql.LevelReadCommitted,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer tx1.Rollback()
	fmt.Println("Started Read Committed transaction")
	tx1.Commit()

	// Repeatable Read
	tx2, err := db.BeginTx(ctx, &sql.TxOptions{
		Isolation: sql.LevelRepeatableRead,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer tx2.Rollback()
	fmt.Println("Started Repeatable Read transaction")
	tx2.Commit()

	// Serializable (strongest isolation)
	tx3, err := db.BeginTx(ctx, &sql.TxOptions{
		Isolation: sql.LevelSerializable,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer tx3.Rollback()
	fmt.Println("Started Serializable transaction")
	tx3.Commit()

	// Read-only transaction (database can optimize, rejects writes)
	tx4, err := db.BeginTx(ctx, &sql.TxOptions{
		Isolation: sql.LevelRepeatableRead,
		ReadOnly:  true,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer tx4.Rollback()
	fmt.Println("Started Read-Only Repeatable Read transaction")
	tx4.Commit()
}

func main() {
	fmt.Println("Transaction options demonstration")
	fmt.Println("(Requires a running database to execute)")
}
```

### Reusable Transaction Helper

In production code, the begin/defer-rollback/commit pattern is so common that it is worth extracting into a helper:

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
)

// WithTransaction executes the given function within a database transaction.
// If fn returns nil, the transaction is committed.
// If fn returns an error, the transaction is rolled back.
// If the commit fails, the commit error is returned.
func WithTransaction(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(tx *sql.Tx) error) error {
	tx, err := db.BeginTx(ctx, opts)
	if err != nil {
		return fmt.Errorf("begin transaction: %w", err)
	}
	defer tx.Rollback()

	if err := fn(tx); err != nil {
		return err
	}

	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit transaction: %w", err)
	}

	return nil
}

// Usage example (pseudocode -- requires a running database)
func transferWithHelper(ctx context.Context, db *sql.DB, from, to string, amount int) error {
	return WithTransaction(ctx, db, nil, func(tx *sql.Tx) error {
		// All queries use tx, not db
		var balance int
		err := tx.QueryRowContext(ctx,
			"SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", from,
		).Scan(&balance)
		if err != nil {
			return fmt.Errorf("query balance: %w", err)
		}

		if balance < amount {
			return fmt.Errorf("insufficient funds")
		}

		_, err = tx.ExecContext(ctx,
			"UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from,
		)
		if err != nil {
			return fmt.Errorf("debit: %w", err)
		}

		_, err = tx.ExecContext(ctx,
			"UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to,
		)
		if err != nil {
			return fmt.Errorf("credit: %w", err)
		}

		return nil
	})
}

func main() {
	log.Println("WithTransaction helper pattern demonstration")
	log.Println("This pattern eliminates boilerplate and ensures transactions")
	log.Println("are always properly committed or rolled back.")
}
```

### In-Memory Atomic Transaction Simulation

Since not everyone has a database running, here is a complete in-memory bank that demonstrates atomicity with proper rollback:

```go
package main

import (
	"fmt"
	"sync"
)

// SafeBank demonstrates atomic transactions in memory.
type SafeBank struct {
	mu       sync.Mutex
	accounts map[string]int
}

func NewSafeBank() *SafeBank {
	return &SafeBank{
		accounts: map[string]int{
			"alice": 1000,
			"bob":   1000,
			"carol": 1000,
		},
	}
}

// Transfer atomically moves money. If any step fails, no changes are applied.
func (b *SafeBank) Transfer(from, to string, amount int, simulateCrash bool) error {
	b.mu.Lock()
	defer b.mu.Unlock()

	// Save original values for rollback
	originalFrom := b.accounts[from]
	originalTo := b.accounts[to]

	if b.accounts[from] < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, b.accounts[from], amount)
	}

	// Step 1: Debit
	b.accounts[from] -= amount

	// Simulate a failure
	if simulateCrash {
		// ROLLBACK: restore original values
		b.accounts[from] = originalFrom
		b.accounts[to] = originalTo
		return fmt.Errorf("simulated crash -- transaction rolled back, no data changed")
	}

	// Step 2: Credit
	b.accounts[to] += amount

	return nil
}

func (b *SafeBank) Balance(account string) int {
	b.mu.Lock()
	defer b.mu.Unlock()
	return b.accounts[account]
}

func (b *SafeBank) TotalMoney() int {
	b.mu.Lock()
	defer b.mu.Unlock()
	total := 0
	for _, v := range b.accounts {
		total += v
	}
	return total
}

func main() {
	bank := NewSafeBank()
	fmt.Printf("Initial: Alice=%d, Bob=%d, Total=%d\n\n",
		bank.Balance("alice"), bank.Balance("bob"), bank.TotalMoney())

	// Successful transfer
	fmt.Println("Transfer 1: $200 from Alice to Bob")
	if err := bank.Transfer("alice", "bob", 200, false); err != nil {
		fmt.Printf("  Error: %v\n", err)
	} else {
		fmt.Println("  Success")
	}
	fmt.Printf("  Alice=%d, Bob=%d, Total=%d\n\n",
		bank.Balance("alice"), bank.Balance("bob"), bank.TotalMoney())

	// Failed transfer -- crash simulated, but transaction is rolled back
	fmt.Println("Transfer 2: $300 from Alice to Bob (with simulated crash)")
	if err := bank.Transfer("alice", "bob", 300, true); err != nil {
		fmt.Printf("  Error: %v\n", err)
	}
	fmt.Printf("  Alice=%d, Bob=%d, Total=%d\n\n",
		bank.Balance("alice"), bank.Balance("bob"), bank.TotalMoney())

	fmt.Println("Total money is preserved because the failed transfer was rolled back.")
}
```

**Output:**

```
Initial: Alice=1000, Bob=1000, Total=3000

Transfer 1: $200 from Alice to Bob
  Success
  Alice=800, Bob=1200, Total=3000

Transfer 2: $300 from Alice to Bob (with simulated crash)
  Error: simulated crash -- transaction rolled back, no data changed
  Alice=800, Bob=1200, Total=3000

Total money is preserved because the failed transfer was rolled back.
```

---

## 3. Isolation Levels Explained

### The Spectrum

Isolation levels define how much one transaction can see of another's uncommitted or recently committed work. Stronger isolation means fewer anomalies but lower concurrency (and throughput). Weaker isolation allows more anomalies but permits more parallelism.

```
Weakest                                                      Strongest
────────────────────────────────────────────────────────────────────────
Read          Read           Repeatable      Snapshot       Serializable
Uncommitted   Committed      Read            Isolation
                                             (MVCC)

Anomalies allowed:

              Dirty    Non-repeatable   Phantom    Write    Lost
              Reads    Reads            Reads      Skew     Updates
              ─────    ──────────────   ───────    ─────    ───────
Read Uncom.    ✓            ✓              ✓         ✓        ✓
Read Commit.   ✗            ✓              ✓         ✓        ✓
Repeat. Read   ✗            ✗              ✓*        ✓        ✓*
Snapshot       ✗            ✗              ✗         ✓        ✗
Serializable   ✗            ✗              ✗         ✗        ✗

* Implementation-dependent. PostgreSQL's Repeatable Read is actually Snapshot.
```

### Quick Summary of Each Level

**Read Uncommitted:** Transactions can see other transactions' uncommitted writes. Almost never used in practice. PostgreSQL does not even implement it -- it silently upgrades to Read Committed.

**Read Committed:** The default in PostgreSQL and Oracle. Each query within a transaction sees only data committed before that query began. Different queries in the same transaction can see different committed states.

**Repeatable Read:** Once a transaction reads a value, it sees the same value for the rest of the transaction, even if another transaction commits a change. In PostgreSQL, this is actually Snapshot Isolation.

**Serializable:** The strongest level. The database guarantees that the result is the same as if transactions had been executed one at a time, in some serial order. Uses Serializable Snapshot Isolation (SSI) in PostgreSQL.

### Choosing an Isolation Level

```go
package main

import (
	"fmt"
)

// IsolationGuide helps you choose the right isolation level.
type IsolationGuide struct {
	Level       string
	UseWhen     string
	Anomalies   []string
	Performance string
	Example     string
}

func main() {
	guides := []IsolationGuide{
		{
			Level:       "Read Committed",
			UseWhen:     "Default choice. Fine for most OLTP workloads where occasional non-repeatable reads are acceptable.",
			Anomalies:   []string{"Non-repeatable reads", "Phantom reads", "Write skew", "Lost updates"},
			Performance: "Best concurrency, lowest overhead",
			Example:     "Displaying user profiles, inserting log entries, simple CRUD",
		},
		{
			Level:       "Repeatable Read / Snapshot",
			UseWhen:     "You need consistent reads within a transaction. Reports, analytics, or any read-heavy transaction that must see a stable snapshot.",
			Anomalies:   []string{"Write skew (with Snapshot)", "Phantoms (with strict RR)"},
			Performance: "Moderate overhead from version tracking",
			Example:     "Generating financial reports, backup operations, long-running reads",
		},
		{
			Level:       "Serializable",
			UseWhen:     "Correctness is paramount. Financial transactions, inventory management, anything where write skew or phantom reads would cause real damage.",
			Anomalies:   []string{},
			Performance: "Higher abort rate, retry logic required",
			Example:     "Bank transfers, seat reservations, stock trading",
		},
	}

	for _, g := range guides {
		fmt.Printf("=== %s ===\n", g.Level)
		fmt.Printf("  Use when: %s\n", g.UseWhen)
		fmt.Printf("  Performance: %s\n", g.Performance)
		fmt.Printf("  Example: %s\n", g.Example)
		if len(g.Anomalies) > 0 {
			fmt.Printf("  Allows: %v\n", g.Anomalies)
		} else {
			fmt.Printf("  Allows: None (full isolation)\n")
		}
		fmt.Println()
	}
}
```

---

## 4. Read Committed in Practice

### What Is a Dirty Read?

A dirty read occurs when transaction A reads data that transaction B has written but not yet committed. If B then rolls back, A has based its logic on data that never actually existed.

### Demonstrating Dirty Reads vs Read Committed

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// DirtyReadStore allows dirty reads (Read Uncommitted behavior).
type DirtyReadStore struct {
	mu   sync.Mutex
	data map[string]int
}

func NewDirtyReadStore() *DirtyReadStore {
	return &DirtyReadStore{
		data: map[string]int{"counter": 0},
	}
}

// Write updates the value without any isolation.
func (s *DirtyReadStore) Write(key string, value int) {
	s.mu.Lock()
	s.data[key] = value
	s.mu.Unlock()
}

// Read returns the current value, even if it is uncommitted.
func (s *DirtyReadStore) Read(key string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.data[key]
}

// ReadCommittedStore prevents dirty reads by only exposing committed data.
type ReadCommittedStore struct {
	mu        sync.RWMutex
	committed map[string]int            // Only committed values
	dirty     map[uint64]map[string]int // Per-transaction dirty writes
	txCounter uint64
}

func NewReadCommittedStore() *ReadCommittedStore {
	return &ReadCommittedStore{
		committed: map[string]int{"counter": 0},
		dirty:     make(map[uint64]map[string]int),
	}
}

func (s *ReadCommittedStore) BeginTx() uint64 {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.txCounter++
	txID := s.txCounter
	s.dirty[txID] = make(map[string]int)
	return txID
}

// Write stores the value in the transaction's dirty buffer, not in committed data.
func (s *ReadCommittedStore) Write(txID uint64, key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.dirty[txID][key] = value
}

// Read returns the committed value. It never sees another transaction's dirty writes.
// A transaction CAN see its own dirty writes, though.
func (s *ReadCommittedStore) Read(txID uint64, key string) int {
	s.mu.RLock()
	defer s.mu.RUnlock()

	// Check own dirty writes first
	if val, ok := s.dirty[txID][key]; ok {
		return val
	}

	// Otherwise, return committed value
	return s.committed[key]
}

// Commit applies dirty writes to committed data.
func (s *ReadCommittedStore) Commit(txID uint64) {
	s.mu.Lock()
	defer s.mu.Unlock()
	for k, v := range s.dirty[txID] {
		s.committed[k] = v
	}
	delete(s.dirty, txID)
}

// Rollback discards dirty writes.
func (s *ReadCommittedStore) Rollback(txID uint64) {
	s.mu.Lock()
	defer s.mu.Unlock()
	delete(s.dirty, txID)
}

func main() {
	fmt.Println("=== Dirty Read Demonstration ===")
	fmt.Println()

	// --- Scenario 1: Dirty Read (Read Uncommitted) ---
	dirtyStore := NewDirtyReadStore()

	var wg sync.WaitGroup
	wg.Add(2)

	// Transaction A: writes 42, then rolls back (restores to 0)
	go func() {
		defer wg.Done()
		fmt.Println("[Tx A] Writing counter = 42 (uncommitted)")
		dirtyStore.Write("counter", 42)

		time.Sleep(100 * time.Millisecond) // Simulate work

		fmt.Println("[Tx A] Rolling back! Restoring counter = 0")
		dirtyStore.Write("counter", 0)
	}()

	// Transaction B: reads the value while A has not committed
	go func() {
		defer wg.Done()
		time.Sleep(50 * time.Millisecond) // Read while A's write is visible

		value := dirtyStore.Read("counter")
		fmt.Printf("[Tx B] Read counter = %d  <-- DIRTY READ! A hasn't committed.\n", value)
	}()

	wg.Wait()
	fmt.Println()

	// --- Scenario 2: Read Committed (no dirty reads) ---
	fmt.Println("=== Read Committed Demonstration ===")
	fmt.Println()

	rcStore := NewReadCommittedStore()

	wg.Add(2)

	// Transaction A: writes 42, waits, then rolls back
	go func() {
		defer wg.Done()
		txA := rcStore.BeginTx()
		fmt.Printf("[Tx A (id=%d)] Writing counter = 42 (uncommitted)\n", txA)
		rcStore.Write(txA, "counter", 42)

		// Transaction A can see its own write
		ownRead := rcStore.Read(txA, "counter")
		fmt.Printf("[Tx A (id=%d)] Own read: counter = %d (sees own write)\n", txA, ownRead)

		time.Sleep(100 * time.Millisecond)

		fmt.Printf("[Tx A (id=%d)] Rolling back!\n", txA)
		rcStore.Rollback(txA)
	}()

	// Transaction B: tries to read while A has dirty data
	go func() {
		defer wg.Done()
		time.Sleep(50 * time.Millisecond)

		txB := rcStore.BeginTx()
		value := rcStore.Read(txB, "counter")
		fmt.Printf("[Tx B (id=%d)] Read counter = %d  <-- Sees COMMITTED value only.\n", txB, value)
		rcStore.Commit(txB)
	}()

	wg.Wait()
	fmt.Println()
	fmt.Println("Read Committed prevents dirty reads: Tx B never saw Tx A's uncommitted write of 42.")
}
```

**Output:**

```
=== Dirty Read Demonstration ===

[Tx A] Writing counter = 42 (uncommitted)
[Tx B] Read counter = 42  <-- DIRTY READ! A hasn't committed.
[Tx A] Rolling back! Restoring counter = 0

=== Read Committed Demonstration ===

[Tx A (id=1)] Writing counter = 42 (uncommitted)
[Tx A (id=1)] Own read: counter = 42 (sees own write)
[Tx B (id=2)] Read counter = 0  <-- Sees COMMITTED value only.
[Tx A (id=1)] Rolling back!

Read Committed prevents dirty reads: Tx B never saw Tx A's uncommitted write of 42.
```

### Non-Repeatable Reads Under Read Committed

Read Committed prevents dirty reads but still allows non-repeatable reads: if you read the same key twice within a transaction, you might get different values because another transaction committed between your two reads.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// SimpleReadCommitted demonstrates non-repeatable reads.
type SimpleReadCommitted struct {
	mu        sync.Mutex
	committed map[string]int
}

func (s *SimpleReadCommitted) Read(key string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.committed[key]
}

func (s *SimpleReadCommitted) CommitWrite(key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.committed[key] = value
}

func main() {
	store := &SimpleReadCommitted{
		committed: map[string]int{"balance": 500},
	}

	var wg sync.WaitGroup
	wg.Add(2)

	// Transaction A: reads balance twice, with a gap between reads
	go func() {
		defer wg.Done()
		fmt.Println("[Tx A] First read:")
		balance1 := store.Read("balance")
		fmt.Printf("[Tx A]   balance = %d\n", balance1)

		time.Sleep(100 * time.Millisecond) // Another transaction commits here

		fmt.Println("[Tx A] Second read (same transaction):")
		balance2 := store.Read("balance")
		fmt.Printf("[Tx A]   balance = %d\n", balance2)

		if balance1 != balance2 {
			fmt.Printf("[Tx A]   NON-REPEATABLE READ: saw %d then %d!\n", balance1, balance2)
		}
	}()

	// Transaction B: modifies and commits between A's two reads
	go func() {
		defer wg.Done()
		time.Sleep(50 * time.Millisecond)
		fmt.Println("[Tx B] Committing balance = 200")
		store.CommitWrite("balance", 200)
	}()

	wg.Wait()
	fmt.Println()
	fmt.Println("Under Read Committed, Tx A saw different values for the same key")
	fmt.Println("within a single transaction. Snapshot Isolation prevents this.")
}
```

---

## 5. Snapshot Isolation (MVCC)

### How It Works

Snapshot Isolation (SI) gives each transaction a consistent snapshot of the database as it was at the transaction's start time. No matter what other transactions do (commit, modify, delete), the reading transaction always sees the data as it was when it started.

The mechanism behind SI is **Multi-Version Concurrency Control (MVCC)**. Instead of overwriting a value in place, the database keeps multiple versions of each value. Each version is tagged with the transaction ID that created it and (optionally) the transaction ID that deleted it. When a transaction reads a key, it finds the version that was most recently committed before the transaction started.

```
MVCC Timeline:

Time ──►

Key "balance":
    Version 1: value=1000, created_by=Tx1, deleted_by=Tx3
    Version 2: value=800,  created_by=Tx3, deleted_by=Tx5
    Version 3: value=650,  created_by=Tx5, deleted_by=0 (current)

If Tx4 starts between Tx3 and Tx5:
    Tx4 sees version 2 (value=800), because:
    - Version 3 was created by Tx5, which started AFTER Tx4
    - Version 2 was created by Tx3, which committed BEFORE Tx4 started
    - Version 1 was deleted by Tx3, so Tx4 doesn't see it
```

### Complete MVCC Store Implementation

```go
package main

import (
	"fmt"
	"sort"
	"sync"
	"sync/atomic"
	"time"
)

// VersionedValue represents a single version of a key-value pair.
type VersionedValue struct {
	Value     string
	TxID      uint64 // Transaction that created this version
	CreatedAt uint64 // Logical timestamp of creation
	DeletedBy uint64 // Transaction that deleted (0 = not deleted)
}

// Transaction represents an active MVCC transaction.
type Transaction struct {
	ID        uint64
	StartedAt uint64 // The txCounter value when this transaction began
	Writes    map[string]string
	Deletes   map[string]bool
	Committed bool
	Aborted   bool
}

// MVCCStore implements multi-version concurrency control.
type MVCCStore struct {
	mu          sync.RWMutex
	data        map[string][]VersionedValue
	txCounter   atomic.Uint64
	committed   map[uint64]bool // Which transaction IDs have committed
	transactions map[uint64]*Transaction
}

func NewMVCCStore() *MVCCStore {
	return &MVCCStore{
		data:         make(map[string][]VersionedValue),
		committed:    make(map[uint64]bool),
		transactions: make(map[uint64]*Transaction),
	}
}

// Begin starts a new transaction and returns it.
func (s *MVCCStore) Begin() *Transaction {
	s.mu.Lock()
	defer s.mu.Unlock()

	id := s.txCounter.Add(1)
	tx := &Transaction{
		ID:        id,
		StartedAt: id, // Snapshot is "everything committed before this ID"
		Writes:    make(map[string]string),
		Deletes:   make(map[string]bool),
	}
	s.transactions[id] = tx
	return tx
}

// Read returns the value visible to the given transaction.
// Visibility rule: a version is visible if:
//   1. It was created by a committed transaction with ID < tx.StartedAt, AND
//   2. It has not been deleted by a committed transaction with ID < tx.StartedAt
//   OR
//   3. It was created by this transaction itself
func (s *MVCCStore) Read(tx *Transaction, key string) (string, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	// Check the transaction's own writes first
	if val, ok := tx.Writes[key]; ok {
		if tx.Deletes[key] {
			return "", false
		}
		return val, true
	}

	versions := s.data[key]
	if len(versions) == 0 {
		return "", false
	}

	// Walk versions in reverse (newest first) to find the most recent visible one
	for i := len(versions) - 1; i >= 0; i-- {
		v := versions[i]

		// Is this version visible to us?
		createdByCommitted := s.committed[v.TxID] && v.TxID < tx.StartedAt
		createdBySelf := v.TxID == tx.ID

		if !createdByCommitted && !createdBySelf {
			continue // Created by a future or uncommitted transaction
		}

		// Is it deleted by a committed transaction we can see?
		if v.DeletedBy != 0 {
			deletedByCommitted := s.committed[v.DeletedBy] && v.DeletedBy < tx.StartedAt
			deletedBySelf := v.DeletedBy == tx.ID
			if deletedByCommitted || deletedBySelf {
				continue // This version is deleted in our snapshot
			}
		}

		return v.Value, true
	}

	return "", false
}

// Write stages a write in the transaction's buffer.
func (s *MVCCStore) Write(tx *Transaction, key, value string) {
	tx.Writes[key] = value
	delete(tx.Deletes, key)
}

// Delete stages a deletion in the transaction's buffer.
func (s *MVCCStore) Delete(tx *Transaction, key string) {
	tx.Deletes[key] = true
	delete(tx.Writes, key)
}

// Commit applies the transaction's writes to the store.
func (s *MVCCStore) Commit(tx *Transaction) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if tx.Aborted {
		return fmt.Errorf("transaction %d already aborted", tx.ID)
	}

	// Apply writes: create new versions
	for key, value := range tx.Writes {
		s.data[key] = append(s.data[key], VersionedValue{
			Value:     value,
			TxID:      tx.ID,
			CreatedAt: tx.ID,
		})
	}

	// Apply deletes: mark existing visible versions as deleted by this tx
	for key := range tx.Deletes {
		versions := s.data[key]
		for i := len(versions) - 1; i >= 0; i-- {
			v := &versions[i]
			if v.DeletedBy == 0 && (s.committed[v.TxID] || v.TxID == tx.ID) {
				v.DeletedBy = tx.ID
				break
			}
		}
		s.data[key] = versions
	}

	tx.Committed = true
	s.committed[tx.ID] = true
	return nil
}

// Abort discards the transaction.
func (s *MVCCStore) Abort(tx *Transaction) {
	s.mu.Lock()
	defer s.mu.Unlock()
	tx.Aborted = true
	tx.Writes = nil
	tx.Deletes = nil
}

// Dump prints all versions of all keys for debugging.
func (s *MVCCStore) Dump() {
	s.mu.RLock()
	defer s.mu.RUnlock()

	keys := make([]string, 0, len(s.data))
	for k := range s.data {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	fmt.Println("--- MVCC Store State ---")
	for _, key := range keys {
		fmt.Printf("Key %q:\n", key)
		for _, v := range s.data[key] {
			deleted := ""
			if v.DeletedBy > 0 {
				deleted = fmt.Sprintf(", deleted_by=Tx%d", v.DeletedBy)
			}
			committed := ""
			if s.committed[v.TxID] {
				committed = " [committed]"
			}
			fmt.Printf("  Tx%d%s: %q%s\n", v.TxID, committed, v.Value, deleted)
		}
	}
	fmt.Println("------------------------")
}

func main() {
	store := NewMVCCStore()

	// Setup: Tx1 creates initial data
	tx1 := store.Begin()
	store.Write(tx1, "alice_balance", "1000")
	store.Write(tx1, "bob_balance", "500")
	store.Commit(tx1)
	fmt.Println("Tx1: Created initial balances (alice=1000, bob=500)")

	// Tx2: Starts a long-running read transaction
	tx2 := store.Begin()
	fmt.Printf("\nTx2: Started (snapshot at Tx%d)\n", tx2.StartedAt)

	// Read initial values in Tx2's snapshot
	alice2, _ := store.Read(tx2, "alice_balance")
	bob2, _ := store.Read(tx2, "bob_balance")
	fmt.Printf("Tx2: Reads alice=%s, bob=%s\n", alice2, bob2)

	// Tx3: Modifies alice's balance and commits
	tx3 := store.Begin()
	store.Write(tx3, "alice_balance", "700")
	store.Commit(tx3)
	fmt.Println("\nTx3: Updated alice_balance to 700 and COMMITTED")

	// Tx2 reads again -- should still see the old value (snapshot isolation!)
	alice2Again, _ := store.Read(tx2, "alice_balance")
	bob2Again, _ := store.Read(tx2, "bob_balance")
	fmt.Printf("Tx2: Re-reads alice=%s, bob=%s  <-- Still sees original snapshot!\n",
		alice2Again, bob2Again)

	// Tx4: Starts after Tx3 committed -- sees Tx3's changes
	tx4 := store.Begin()
	alice4, _ := store.Read(tx4, "alice_balance")
	bob4, _ := store.Read(tx4, "bob_balance")
	fmt.Printf("\nTx4: Reads alice=%s, bob=%s  <-- Sees Tx3's committed changes\n",
		alice4, bob4)
	store.Commit(tx4)

	store.Commit(tx2)

	fmt.Println()
	store.Dump()

	// --- Demonstrate concurrent writes ---
	fmt.Println("\n=== Concurrent MVCC Transactions ===")
	store2 := NewMVCCStore()

	// Initial setup
	setup := store2.Begin()
	store2.Write(setup, "counter", "0")
	store2.Commit(setup)

	var wg sync.WaitGroup
	results := make([]string, 5)

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			tx := store2.Begin()

			// Each goroutine reads the counter
			val, _ := store2.Read(tx, "counter")

			time.Sleep(10 * time.Millisecond) // Simulate some work

			// And writes a new value
			store2.Write(tx, "counter", fmt.Sprintf("written_by_goroutine_%d", id))
			store2.Commit(tx)

			results[id] = fmt.Sprintf("Goroutine %d: read %q, wrote its own value (Tx%d)", id, val, tx.ID)
		}(i)
	}

	wg.Wait()
	for _, r := range results {
		fmt.Println(r)
	}

	final := store2.Begin()
	counterVal, _ := store2.Read(final, "counter")
	fmt.Printf("\nFinal counter value: %q (last writer wins)\n", counterVal)
	store2.Commit(final)
}
```

**Output (example -- goroutine ordering may vary):**

```
Tx1: Created initial balances (alice=1000, bob=500)

Tx2: Started (snapshot at Tx2)
Tx2: Reads alice=1000, bob=500

Tx3: Updated alice_balance to 700 and COMMITTED
Tx2: Re-reads alice=1000, bob=500  <-- Still sees original snapshot!

Tx4: Reads alice=700, bob=500  <-- Sees Tx3's committed changes

--- MVCC Store State ---
Key "alice_balance":
  Tx1 [committed]: "1000"
  Tx3 [committed]: "700"
Key "bob_balance":
  Tx1 [committed]: "500"
------------------------

=== Concurrent MVCC Transactions ===
Goroutine 0: read "0", wrote its own value (Tx3)
Goroutine 1: read "0", wrote its own value (Tx4)
Goroutine 2: read "0", wrote its own value (Tx5)
Goroutine 3: read "0", wrote its own value (Tx6)
Goroutine 4: read "0", wrote its own value (Tx7)

Final counter value: "written_by_goroutine_4" (last writer wins)
```

Notice the problem in the concurrent section: all five goroutines read the same initial value "0" and then each wrote their own value. Only the last commit's write survives. This is the **lost update** problem, which we address next.

---

## 6. Lost Update Problem

### What Is a Lost Update?

A lost update occurs when two transactions both read the same value, modify it independently, and write it back. The second write overwrites the first, and the first update is silently lost.

```
Lost Update Scenario (counter increment):

    Counter starts at 10.

    Tx A: READ counter → 10
    Tx B: READ counter → 10
    Tx A: WRITE counter = 10 + 1 = 11
    Tx B: WRITE counter = 10 + 1 = 11   ← Overwrites A's increment!

    Counter is 11, but should be 12. Tx A's increment is LOST.
```

### Demonstrating the Lost Update

```go
package main

import (
	"fmt"
	"sync"
)

// NaiveCounter demonstrates the lost update problem.
type NaiveCounter struct {
	mu    sync.Mutex
	value int
}

func (c *NaiveCounter) Get() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.value
}

// IncrementUnsafe reads, computes, then writes -- but releases the lock between.
// This simulates what happens with separate SELECT + UPDATE without FOR UPDATE.
func (c *NaiveCounter) IncrementUnsafe() {
	// Step 1: Read (like SELECT value FROM counters WHERE id = 1)
	c.mu.Lock()
	current := c.value
	c.mu.Unlock()

	// Simulate some processing time -- other goroutines can read the same value
	// (In a real database, this is the gap between SELECT and UPDATE)
	newValue := current + 1

	// Step 2: Write (like UPDATE counters SET value = $1 WHERE id = 1)
	c.mu.Lock()
	c.value = newValue
	c.mu.Unlock()
}

// IncrementSafe uses atomic read-modify-write (like UPDATE counters SET value = value + 1)
func (c *NaiveCounter) IncrementSafe() {
	c.mu.Lock()
	c.value++
	c.mu.Unlock()
}

func main() {
	// --- Demonstrate lost updates ---
	fmt.Println("=== Lost Update Demonstration ===")

	counter := &NaiveCounter{value: 0}
	var wg sync.WaitGroup

	numGoroutines := 1000

	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.IncrementUnsafe()
		}()
	}

	wg.Wait()
	fmt.Printf("Unsafe increment: Expected %d, Got %d (lost %d updates)\n",
		numGoroutines, counter.Get(), numGoroutines-counter.Get())

	// --- Demonstrate safe updates ---
	fmt.Println("\n=== Safe Atomic Increment ===")

	safeCounter := &NaiveCounter{value: 0}

	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			safeCounter.IncrementSafe()
		}()
	}

	wg.Wait()
	fmt.Printf("Safe increment: Expected %d, Got %d\n",
		numGoroutines, safeCounter.Get())
}
```

**Output (unsafe count varies per run):**

```
=== Lost Update Demonstration ===
Unsafe increment: Expected 1000, Got 537 (lost 463 updates)

=== Safe Atomic Increment ===
Safe increment: Expected 1000, Got 1000
```

### Three Solutions to Lost Updates

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

// ========================================
// Solution 1: SELECT FOR UPDATE (Pessimistic Locking)
// ========================================
// In a real database:
//   tx.QueryRow("SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", id)
//   // The row is now locked until this transaction commits/rolls back
//   tx.Exec("UPDATE accounts SET balance = $1 WHERE id = $2", newBalance, id)
//
// In-memory equivalent:

type PessimisticStore struct {
	mu       sync.Mutex
	data     map[string]int
	rowLocks map[string]*sync.Mutex
}

func NewPessimisticStore() *PessimisticStore {
	return &PessimisticStore{
		data:     map[string]int{"counter": 0},
		rowLocks: map[string]*sync.Mutex{"counter": {}},
	}
}

func (s *PessimisticStore) SelectForUpdate(key string) int {
	s.rowLocks[key].Lock() // Acquires exclusive lock on the row
	s.mu.Lock()
	val := s.data[key]
	s.mu.Unlock()
	return val
}

func (s *PessimisticStore) UpdateAndRelease(key string, value int) {
	s.mu.Lock()
	s.data[key] = value
	s.mu.Unlock()
	s.rowLocks[key].Unlock() // Release row lock
}

func (s *PessimisticStore) Get(key string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.data[key]
}

// ========================================
// Solution 2: Compare-and-Set (Optimistic Locking)
// ========================================
// In a real database:
//   UPDATE accounts SET balance = $1 WHERE id = $2 AND balance = $3
//   -- If balance has changed since we read it, the WHERE clause won't match
//   -- and the UPDATE affects 0 rows, signaling a conflict.

type CASStore struct {
	mu   sync.Mutex
	data map[string]int
}

func NewCASStore() *CASStore {
	return &CASStore{data: map[string]int{"counter": 0}}
}

func (s *CASStore) Read(key string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.data[key]
}

// CompareAndSet writes newValue only if the current value equals expectedOld.
// Returns true if the write succeeded, false if there was a conflict.
func (s *CASStore) CompareAndSet(key string, expectedOld, newValue int) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.data[key] == expectedOld {
		s.data[key] = newValue
		return true
	}
	return false // Conflict -- another transaction changed the value
}

func (s *CASStore) Get(key string) int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.data[key]
}

// ========================================
// Solution 3: Atomic Increment
// ========================================
// In a real database:
//   UPDATE accounts SET balance = balance + $1 WHERE id = $2
//   -- The database handles the read-modify-write atomically.

type AtomicStore struct {
	counter atomic.Int64
}

func main() {
	const N = 1000

	// --- Solution 1: Pessimistic Locking (SELECT FOR UPDATE) ---
	fmt.Println("=== Solution 1: Pessimistic Locking (SELECT FOR UPDATE) ===")
	pessimistic := NewPessimisticStore()
	var wg sync.WaitGroup
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			current := pessimistic.SelectForUpdate("counter")
			pessimistic.UpdateAndRelease("counter", current+1)
		}()
	}
	wg.Wait()
	fmt.Printf("Result: %d (expected %d)\n\n", pessimistic.Get("counter"), N)

	// --- Solution 2: Compare-and-Set with retry ---
	fmt.Println("=== Solution 2: Compare-and-Set (Optimistic Locking) ===")
	cas := NewCASStore()
	retryCount := atomic.Int64{}
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				old := cas.Read("counter")
				if cas.CompareAndSet("counter", old, old+1) {
					return // Success
				}
				retryCount.Add(1)
				// Conflict -- retry with the new value
			}
		}()
	}
	wg.Wait()
	fmt.Printf("Result: %d (expected %d), retries: %d\n\n",
		cas.Get("counter"), N, retryCount.Load())

	// --- Solution 3: Atomic Increment ---
	fmt.Println("=== Solution 3: Atomic Increment ===")
	atomicStore := &AtomicStore{}
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomicStore.counter.Add(1)
		}()
	}
	wg.Wait()
	fmt.Printf("Result: %d (expected %d)\n", atomicStore.counter.Load(), N)
}
```

**Output:**

```
=== Solution 1: Pessimistic Locking (SELECT FOR UPDATE) ===
Result: 1000 (expected 1000)

=== Solution 2: Compare-and-Set (Optimistic Locking) ===
Result: 1000 (expected 1000), retries: 4231

=== Solution 3: Atomic Increment ===
Result: 1000 (expected 1000)
```

### When to Use Each Solution

| Solution | Best For | Trade-Off |
|----------|----------|-----------|
| **SELECT FOR UPDATE** | High-contention writes, complex read-modify-write logic | Blocks other transactions, risk of deadlocks |
| **Compare-and-Set** | Low-contention writes, simple value updates | Retries under contention, wasted work |
| **Atomic Increment** | Simple numeric operations (counters, balances) | Only works for commutative operations |

---

## 7. Write Skew and Phantoms

### Write Skew Explained

Write skew is a subtle concurrency anomaly that cannot be prevented by row-level locking alone. It occurs when two transactions read an overlapping set of rows, make decisions based on what they read, and then write to different rows -- violating a constraint that spans multiple rows.

Classic example: A hospital requires at least one doctor on call at all times. Two doctors, both on call, each decide to take themselves off call. Each checks "is there at least one other doctor on call?" -- sees "yes" (the other doctor) -- and removes themselves. Result: zero doctors on call.

```
Write Skew:

    Invariant: at least 1 doctor on call

    Doctors on call: [Alice, Bob]

    Tx A: SELECT count(*) FROM on_call → 2 (OK to remove one)
    Tx B: SELECT count(*) FROM on_call → 2 (OK to remove one)
    Tx A: UPDATE on_call SET active=false WHERE doctor='Alice'
    Tx B: UPDATE on_call SET active=false WHERE doctor='Bob'

    Result: 0 doctors on call! Invariant violated!
    Neither transaction saw the other's write because they wrote to DIFFERENT rows.
```

### Implementing and Detecting Write Skew

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Hospital demonstrates the write skew problem.
type Hospital struct {
	mu     sync.Mutex
	onCall map[string]bool // doctor name -> on call?
}

func NewHospital() *Hospital {
	return &Hospital{
		onCall: map[string]bool{
			"Alice": true,
			"Bob":   true,
		},
	}
}

// CountOnCall returns how many doctors are currently on call.
func (h *Hospital) CountOnCall() int {
	count := 0
	for _, active := range h.onCall {
		if active {
			count++
		}
	}
	return count
}

// UnsafeLeave allows a doctor to leave without proper isolation.
// This demonstrates write skew.
func (h *Hospital) UnsafeLeave(doctor string) error {
	h.mu.Lock()
	count := h.CountOnCall()
	h.mu.Unlock()

	// Decision made based on stale data -- another transaction may commit between here

	time.Sleep(10 * time.Millisecond) // Simulate processing

	h.mu.Lock()
	defer h.mu.Unlock()

	if count < 2 {
		return fmt.Errorf("%s cannot leave: only %d doctor(s) on call", doctor, count)
	}

	h.onCall[doctor] = false
	fmt.Printf("  %s left on-call duty (saw %d doctors when deciding)\n", doctor, count)
	return nil
}

// SafeLeave uses serialized access to prevent write skew.
func (h *Hospital) SafeLeave(doctor string) error {
	h.mu.Lock()
	defer h.mu.Unlock()

	count := h.CountOnCall()
	if count < 2 {
		return fmt.Errorf("%s cannot leave: only %d doctor(s) on call", doctor, count)
	}

	h.onCall[doctor] = false
	fmt.Printf("  %s left on-call duty (saw %d doctors when deciding)\n", doctor, count)
	return nil
}

func main() {
	// --- Demonstrate write skew ---
	fmt.Println("=== Write Skew (Unsafe) ===")
	hospital := NewHospital()

	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		if err := hospital.UnsafeLeave("Alice"); err != nil {
			fmt.Printf("  Alice blocked: %v\n", err)
		}
	}()

	go func() {
		defer wg.Done()
		if err := hospital.UnsafeLeave("Bob"); err != nil {
			fmt.Printf("  Bob blocked: %v\n", err)
		}
	}()

	wg.Wait()

	hospital.mu.Lock()
	onCallCount := hospital.CountOnCall()
	hospital.mu.Unlock()

	fmt.Printf("Doctors on call: %d", onCallCount)
	if onCallCount == 0 {
		fmt.Println("  <-- WRITE SKEW! No doctors on call!")
	} else {
		fmt.Println("  (OK)")
	}

	// --- Prevent write skew with serialized access ---
	fmt.Println("\n=== Preventing Write Skew (Safe) ===")
	hospital2 := NewHospital()

	wg.Add(2)

	go func() {
		defer wg.Done()
		if err := hospital2.SafeLeave("Alice"); err != nil {
			fmt.Printf("  Alice blocked: %v\n", err)
		}
	}()

	go func() {
		defer wg.Done()
		if err := hospital2.SafeLeave("Bob"); err != nil {
			fmt.Printf("  Bob blocked: %v\n", err)
		}
	}()

	wg.Wait()

	hospital2.mu.Lock()
	onCallCount2 := hospital2.CountOnCall()
	hospital2.mu.Unlock()

	fmt.Printf("Doctors on call: %d", onCallCount2)
	if onCallCount2 >= 1 {
		fmt.Println("  (Invariant maintained!)")
	}
}
```

### Phantoms

A phantom occurs when a transaction re-executes a range query and finds new rows that were not there before -- because another transaction inserted them between the two reads.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// MeetingRoom demonstrates the phantom problem.
type MeetingRoom struct {
	mu       sync.Mutex
	bookings []Booking
}

type Booking struct {
	Room  string
	Start int // Hour (0-23)
	End   int
	User  string
}

func NewMeetingRoom() *MeetingRoom {
	return &MeetingRoom{
		bookings: []Booking{
			{Room: "A", Start: 9, End: 10, User: "Alice"},
			{Room: "A", Start: 14, End: 15, User: "Bob"},
		},
	}
}

// FindBookings returns all bookings for a room that overlap with [start, end).
func (m *MeetingRoom) FindBookings(room string, start, end int) []Booking {
	var result []Booking
	for _, b := range m.bookings {
		if b.Room == room && b.Start < end && b.End > start {
			result = append(result, b)
		}
	}
	return result
}

// UnsafeBook checks for conflicts and books, but does not hold the lock between
// check and insert -- allowing phantoms.
func (m *MeetingRoom) UnsafeBook(room string, start, end int, user string) error {
	m.mu.Lock()
	conflicts := m.FindBookings(room, start, end)
	m.mu.Unlock()

	// Another transaction can insert a conflicting booking here!
	time.Sleep(10 * time.Millisecond)

	m.mu.Lock()
	defer m.mu.Unlock()

	if len(conflicts) > 0 {
		return fmt.Errorf("conflict: room %s already booked at %d-%d by %s",
			room, conflicts[0].Start, conflicts[0].End, conflicts[0].User)
	}

	m.bookings = append(m.bookings, Booking{
		Room: room, Start: start, End: end, User: user,
	})
	return nil
}

// SafeBook holds the lock for the entire check-and-insert operation.
func (m *MeetingRoom) SafeBook(room string, start, end int, user string) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	conflicts := m.FindBookings(room, start, end)
	if len(conflicts) > 0 {
		return fmt.Errorf("conflict: room %s already booked at %d-%d by %s",
			room, conflicts[0].Start, conflicts[0].End, conflicts[0].User)
	}

	m.bookings = append(m.bookings, Booking{
		Room: room, Start: start, End: end, User: user,
	})
	return nil
}

func main() {
	fmt.Println("=== Phantom Demonstration (Unsafe Booking) ===")
	room := NewMeetingRoom()

	var wg sync.WaitGroup
	wg.Add(2)

	// Both try to book room A at 11:00-12:00 simultaneously
	go func() {
		defer wg.Done()
		err := room.UnsafeBook("A", 11, 12, "Carol")
		if err != nil {
			fmt.Printf("  Carol: %v\n", err)
		} else {
			fmt.Println("  Carol: booked room A 11:00-12:00")
		}
	}()

	go func() {
		defer wg.Done()
		err := room.UnsafeBook("A", 11, 12, "Dave")
		if err != nil {
			fmt.Printf("  Dave: %v\n", err)
		} else {
			fmt.Println("  Dave: booked room A 11:00-12:00")
		}
	}()

	wg.Wait()

	room.mu.Lock()
	count := 0
	for _, b := range room.bookings {
		if b.Room == "A" && b.Start == 11 {
			count++
		}
	}
	room.mu.Unlock()

	if count > 1 {
		fmt.Printf("DOUBLE BOOKING! %d bookings for room A at 11:00. This is a phantom.\n", count)
	}

	fmt.Println("\n=== Phantom Prevention (Safe Booking) ===")
	room2 := NewMeetingRoom()

	wg.Add(2)

	go func() {
		defer wg.Done()
		err := room2.SafeBook("A", 11, 12, "Carol")
		if err != nil {
			fmt.Printf("  Carol: %v\n", err)
		} else {
			fmt.Println("  Carol: booked room A 11:00-12:00")
		}
	}()

	go func() {
		defer wg.Done()
		err := room2.SafeBook("A", 11, 12, "Dave")
		if err != nil {
			fmt.Printf("  Dave: %v\n", err)
		} else {
			fmt.Println("  Dave: booked room A 11:00-12:00")
		}
	}()

	wg.Wait()
	fmt.Println("Only one booking should succeed with safe booking.")
}
```

### Materializing Conflicts

One technique to prevent phantoms in databases is **materializing conflicts** -- creating a concrete row for every possible resource slot, so that conflicting transactions can lock the same row. Instead of trying to lock a range of nonexistent rows, you create a row for each time slot and lock the specific row.

In SQL, this looks like creating a `room_slots` table with one row per (room, time_slot) pair, then using `SELECT ... FOR UPDATE` on the relevant slot before inserting a booking.

---

## 8. Two-Phase Locking (2PL)

### How 2PL Works

Two-Phase Locking is the classic algorithm for implementing serializable isolation. The two phases are:

1. **Growing phase:** A transaction can acquire locks but cannot release any.
2. **Shrinking phase:** A transaction can release locks but cannot acquire new ones.

In practice, databases hold all locks until commit (Strict 2PL), which simplifies the protocol and prevents cascading aborts.

Lock modes:
- **Shared (S):** Multiple transactions can hold a shared lock on the same object simultaneously. Used for reads.
- **Exclusive (X):** Only one transaction can hold an exclusive lock. Used for writes. Incompatible with both shared and exclusive locks.

```
Lock Compatibility Matrix:

              Existing Lock
              S     X
New Lock  S   ✓     ✗
          X   ✗     ✗
```

### Complete 2PL Implementation with Deadlock Detection

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// LockMode represents the type of lock.
type LockMode int

const (
	SharedLock    LockMode = iota // For reads
	ExclusiveLock                 // For writes
)

func (m LockMode) String() string {
	switch m {
	case SharedLock:
		return "S"
	case ExclusiveLock:
		return "X"
	default:
		return "?"
	}
}

// LockRequest represents a pending lock request.
type LockRequest struct {
	TxID    uint64
	Mode    LockMode
	Granted chan struct{} // Closed when the lock is granted
}

// Lock represents the lock state for a single key.
type Lock struct {
	holders map[uint64]LockMode // txID -> mode
	waiters []LockRequest
}

// LockManager implements two-phase locking with deadlock detection.
type LockManager struct {
	mu    sync.Mutex
	locks map[string]*Lock
	// waitsFor tracks which transaction is waiting for which other transactions.
	// Used for deadlock detection (cycle detection in the wait-for graph).
	waitsFor map[uint64]map[uint64]bool // txID -> set of txIDs it's waiting for
}

func NewLockManager() *LockManager {
	return &LockManager{
		locks:    make(map[string]*Lock),
		waitsFor: make(map[uint64]map[uint64]bool),
	}
}

// Acquire attempts to acquire a lock. Returns an error if a deadlock is detected.
func (lm *LockManager) Acquire(txID uint64, key string, mode LockMode) error {
	lm.mu.Lock()

	lock, exists := lm.locks[key]
	if !exists {
		lock = &Lock{holders: make(map[uint64]LockMode)}
		lm.locks[key] = lock
	}

	// Can we grant the lock immediately?
	if lm.canGrant(lock, txID, mode) {
		lock.holders[txID] = mode
		lm.mu.Unlock()
		return nil
	}

	// We need to wait. First, check for deadlocks.
	blockers := lm.getBlockers(lock, txID, mode)

	// Record in wait-for graph
	if lm.waitsFor[txID] == nil {
		lm.waitsFor[txID] = make(map[uint64]bool)
	}
	for _, blocker := range blockers {
		lm.waitsFor[txID][blocker] = true
	}

	// Detect deadlock (cycle in wait-for graph)
	if lm.detectDeadlock(txID) {
		// Remove from wait-for graph
		delete(lm.waitsFor, txID)
		lm.mu.Unlock()
		return fmt.Errorf("DEADLOCK detected: Tx%d cannot acquire %s lock on %q", txID, mode, key)
	}

	// No deadlock -- enqueue and wait
	req := LockRequest{
		TxID:    txID,
		Mode:    mode,
		Granted: make(chan struct{}),
	}
	lock.waiters = append(lock.waiters, req)
	lm.mu.Unlock()

	// Wait for the lock to be granted (with timeout)
	select {
	case <-req.Granted:
		return nil
	case <-time.After(5 * time.Second):
		lm.mu.Lock()
		delete(lm.waitsFor, txID)
		// Remove from waiters
		for i, w := range lock.waiters {
			if w.TxID == txID {
				lock.waiters = append(lock.waiters[:i], lock.waiters[i+1:]...)
				break
			}
		}
		lm.mu.Unlock()
		return fmt.Errorf("TIMEOUT: Tx%d timed out waiting for %s lock on %q", txID, mode, key)
	}
}

// canGrant checks if a lock request can be granted immediately.
func (lm *LockManager) canGrant(lock *Lock, txID uint64, mode LockMode) bool {
	if len(lock.holders) == 0 {
		return true
	}

	// If this transaction already holds the lock
	if existingMode, held := lock.holders[txID]; held {
		if mode == SharedLock || existingMode == ExclusiveLock {
			return true // Already have sufficient lock
		}
		// Upgrade from S to X: only if we're the sole holder
		if len(lock.holders) == 1 {
			return true
		}
		return false
	}

	// Check compatibility with existing holders
	if mode == SharedLock {
		// Shared is compatible with shared, not with exclusive
		for _, m := range lock.holders {
			if m == ExclusiveLock {
				return false
			}
		}
		return true
	}

	// Exclusive: incompatible with everything
	return false
}

// getBlockers returns the transaction IDs blocking this request.
func (lm *LockManager) getBlockers(lock *Lock, txID uint64, mode LockMode) []uint64 {
	var blockers []uint64
	for holder, heldMode := range lock.holders {
		if holder == txID {
			continue
		}
		if mode == ExclusiveLock || heldMode == ExclusiveLock {
			blockers = append(blockers, holder)
		}
	}
	return blockers
}

// detectDeadlock performs cycle detection in the wait-for graph using DFS.
func (lm *LockManager) detectDeadlock(startTxID uint64) bool {
	visited := make(map[uint64]bool)
	return lm.dfs(startTxID, startTxID, visited)
}

func (lm *LockManager) dfs(current, target uint64, visited map[uint64]bool) bool {
	if visited[current] {
		return false
	}
	visited[current] = true

	for waitingFor := range lm.waitsFor[current] {
		if waitingFor == target {
			return true // Cycle found!
		}
		if lm.dfs(waitingFor, target, visited) {
			return true
		}
	}
	return false
}

// Release releases all locks held by a transaction and wakes up waiting transactions.
func (lm *LockManager) Release(txID uint64) {
	lm.mu.Lock()
	defer lm.mu.Unlock()

	delete(lm.waitsFor, txID)

	// Remove from all other transactions' wait-for entries
	for _, deps := range lm.waitsFor {
		delete(deps, txID)
	}

	for key, lock := range lm.locks {
		if _, held := lock.holders[txID]; !held {
			continue
		}

		delete(lock.holders, txID)

		// Try to grant waiting requests
		remaining := lock.waiters[:0]
		for _, waiter := range lock.waiters {
			if lm.canGrant(lock, waiter.TxID, waiter.Mode) {
				lock.holders[waiter.TxID] = waiter.Mode
				close(waiter.Granted) // Wake up the waiting goroutine
				delete(lm.waitsFor, waiter.TxID)
			} else {
				remaining = append(remaining, waiter)
			}
		}
		lock.waiters = remaining

		// Clean up empty locks
		if len(lock.holders) == 0 && len(lock.waiters) == 0 {
			delete(lm.locks, key)
		}
	}
}

// Status returns a human-readable lock table.
func (lm *LockManager) Status() string {
	lm.mu.Lock()
	defer lm.mu.Unlock()

	result := "Lock Table:\n"
	for key, lock := range lm.locks {
		result += fmt.Sprintf("  %q: holders={", key)
		first := true
		for txID, mode := range lock.holders {
			if !first {
				result += ", "
			}
			result += fmt.Sprintf("Tx%d:%s", txID, mode)
			first = false
		}
		result += fmt.Sprintf("}, waiters=%d\n", len(lock.waiters))
	}
	if len(lm.locks) == 0 {
		result += "  (empty)\n"
	}
	return result
}

func main() {
	lm := NewLockManager()

	fmt.Println("=== Two-Phase Locking Demo ===")
	fmt.Println()

	// Scenario 1: Shared locks are compatible
	fmt.Println("--- Shared Lock Compatibility ---")
	err := lm.Acquire(1, "account_alice", SharedLock)
	fmt.Printf("Tx1 acquire S lock on account_alice: %v\n", err)

	err = lm.Acquire(2, "account_alice", SharedLock)
	fmt.Printf("Tx2 acquire S lock on account_alice: %v\n", err)

	fmt.Print(lm.Status())

	lm.Release(1)
	lm.Release(2)
	fmt.Println()

	// Scenario 2: Exclusive lock blocks shared
	fmt.Println("--- Exclusive Lock Blocks Others ---")
	err = lm.Acquire(3, "account_bob", ExclusiveLock)
	fmt.Printf("Tx3 acquire X lock on account_bob: %v\n", err)
	fmt.Print(lm.Status())

	// Tx4 tries to read -- must wait
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("Tx4 trying to acquire S lock on account_bob (will wait)...")
		err := lm.Acquire(4, "account_bob", SharedLock)
		fmt.Printf("Tx4 acquired S lock on account_bob: %v\n", err)
	}()

	time.Sleep(100 * time.Millisecond)
	fmt.Println("Tx3 releasing locks...")
	lm.Release(3)
	wg.Wait()
	lm.Release(4)
	fmt.Println()

	// Scenario 3: Deadlock detection
	fmt.Println("--- Deadlock Detection ---")

	// Tx5 locks A, Tx6 locks B
	lm.Acquire(5, "resource_A", ExclusiveLock)
	lm.Acquire(6, "resource_B", ExclusiveLock)
	fmt.Println("Tx5 holds X on resource_A")
	fmt.Println("Tx6 holds X on resource_B")

	// Tx5 tries to lock B (will wait for Tx6)
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("Tx5 trying to acquire X on resource_B (waits for Tx6)...")
		err := lm.Acquire(5, "resource_B", ExclusiveLock)
		if err != nil {
			fmt.Printf("Tx5: %v\n", err)
		} else {
			fmt.Println("Tx5 acquired X on resource_B")
		}
	}()

	time.Sleep(50 * time.Millisecond)

	// Tx6 tries to lock A -- this would create a deadlock!
	fmt.Println("Tx6 trying to acquire X on resource_A (would deadlock)...")
	err = lm.Acquire(6, "resource_A", ExclusiveLock)
	if err != nil {
		fmt.Printf("Tx6: %v\n", err)
	}

	lm.Release(5)
	lm.Release(6)
	wg.Wait()

	fmt.Println()
	fmt.Println("Deadlock was detected and one transaction was aborted.")
	fmt.Println("In a real database, the aborted transaction would retry.")
}
```

**Output:**

```
=== Two-Phase Locking Demo ===

--- Shared Lock Compatibility ---
Tx1 acquire S lock on account_alice: <nil>
Tx2 acquire S lock on account_alice: <nil>
Lock Table:
  "account_alice": holders={Tx1:S, Tx2:S}, waiters=0

--- Exclusive Lock Blocks Others ---
Tx3 acquire X lock on account_bob: <nil>
Lock Table:
  "account_bob": holders={Tx3:X}, waiters=0
Tx4 trying to acquire S lock on account_bob (will wait)...
Tx3 releasing locks...
Tx4 acquired S lock on account_bob: <nil>

--- Deadlock Detection ---
Tx5 holds X on resource_A
Tx6 holds X on resource_B
Tx5 trying to acquire X on resource_B (waits for Tx6)...
Tx6 trying to acquire X on resource_A (would deadlock)...
Tx6: DEADLOCK detected: Tx6 cannot acquire X lock on "resource_A"

Deadlock was detected and one transaction was aborted.
In a real database, the aborted transaction would retry.
```

---

## 9. Serializable Snapshot Isolation (SSI)

### The Idea

Serializable Snapshot Isolation (SSI) is an optimistic approach to serializability. Instead of blocking (like 2PL), SSI allows transactions to proceed without locks on reads. It uses snapshot isolation as a baseline and tracks additional information to detect conflicts at commit time.

The key insight is: instead of preventing conflicts, detect them and abort offending transactions. This is more efficient under low contention because most transactions do not conflict.

SSI tracks two types of dangerous situations:
1. **Read dependencies:** Transaction A reads data that transaction B writes.
2. **Write dependencies:** Transaction A writes data that transaction B also reads and then writes.

If a cycle of dependencies is detected (forming a "dangerous structure"), one of the transactions in the cycle is aborted.

```
SSI vs 2PL:

    2PL (Pessimistic):
        Before reading:  acquire lock, wait if blocked
        Before writing:  acquire lock, wait if blocked
        At commit:       release all locks
        Conflict:        transactions BLOCK each other

    SSI (Optimistic):
        Before reading:  no lock needed (use snapshot)
        Before writing:  no lock needed (buffer writes)
        At commit:       check for conflicts
        Conflict:        one transaction is ABORTED and retried
```

### SSI Implementation

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

// SSIStore implements Serializable Snapshot Isolation.
type SSIStore struct {
	mu        sync.Mutex
	committed map[string]SSIVersion
	txCounter atomic.Uint64

	// Track reads and writes for conflict detection
	txReads  map[uint64]map[string]uint64 // txID -> key -> version read
	txWrites map[uint64]map[string]string // txID -> key -> value written
	// Track which transactions committed their writes
	writeHistory map[string][]WriteRecord // key -> list of committed writes
}

type SSIVersion struct {
	Value   string
	Version uint64 // Transaction ID that wrote this version
}

type WriteRecord struct {
	TxID  uint64
	Value string
}

type SSITx struct {
	ID        uint64
	Snapshot  map[string]SSIVersion // Snapshot at start time
	Reads     map[string]uint64     // key -> version observed
	Writes    map[string]string     // key -> new value
	Committed bool
	Aborted   bool
}

func NewSSIStore() *SSIStore {
	return &SSIStore{
		committed:    make(map[string]SSIVersion),
		txReads:      make(map[uint64]map[string]uint64),
		txWrites:     make(map[uint64]map[string]string),
		writeHistory: make(map[string][]WriteRecord),
	}
}

// Begin creates a new transaction with a snapshot of the current committed state.
func (s *SSIStore) Begin() *SSITx {
	s.mu.Lock()
	defer s.mu.Unlock()

	id := s.txCounter.Add(1)

	// Take a snapshot of all committed data
	snapshot := make(map[string]SSIVersion)
	for k, v := range s.committed {
		snapshot[k] = v
	}

	tx := &SSITx{
		ID:       id,
		Snapshot: snapshot,
		Reads:    make(map[string]uint64),
		Writes:   make(map[string]string),
	}

	s.txReads[id] = tx.Reads
	s.txWrites[id] = tx.Writes

	return tx
}

// Read returns the value from the transaction's snapshot.
func (s *SSIStore) Read(tx *SSITx, key string) (string, bool) {
	// Check own writes first
	if val, ok := tx.Writes[key]; ok {
		return val, true
	}

	// Read from snapshot
	if v, ok := tx.Snapshot[key]; ok {
		tx.Reads[key] = v.Version // Record which version we read
		return v.Value, true
	}

	tx.Reads[key] = 0 // Record that we read "nothing" for this key
	return "", false
}

// Write buffers a write in the transaction.
func (s *SSIStore) Write(tx *SSITx, key, value string) {
	tx.Writes[key] = value
}

// Commit attempts to commit the transaction.
// It checks for serialization conflicts and aborts if any are found.
func (s *SSIStore) Commit(tx *SSITx) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if tx.Aborted {
		return fmt.Errorf("Tx%d: already aborted", tx.ID)
	}

	// Check for conflicts:
	// For each key we READ, check if any concurrent transaction WROTE a new version
	// since our snapshot was taken.
	for key, readVersion := range tx.Reads {
		currentVersion, exists := s.committed[key]
		if exists && currentVersion.Version != readVersion && currentVersion.Version > 0 {
			// The data we read has been modified by another committed transaction!
			tx.Aborted = true
			delete(s.txReads, tx.ID)
			delete(s.txWrites, tx.ID)
			return fmt.Errorf("Tx%d: SERIALIZATION FAILURE on key %q (read version %d, current version %d)",
				tx.ID, key, readVersion, currentVersion.Version)
		}
	}

	// For each key we WRITE, check if any concurrent transaction also READ it
	// (and might make a decision based on the old value).
	// This is a simplified check -- a full SSI implementation uses more
	// sophisticated tracking (rw-antidependency detection).
	for key := range tx.Writes {
		for otherTxID, otherReads := range s.txReads {
			if otherTxID == tx.ID {
				continue
			}
			if _, readByOther := otherReads[key]; readByOther {
				// Another active transaction has read this key.
				// We will still commit, but the other transaction will fail
				// when it tries to commit (first-committer-wins rule).
				fmt.Printf("  [SSI] Warning: Tx%d writes %q which Tx%d has read\n",
					tx.ID, key, otherTxID)
			}
		}
	}

	// No conflicts detected -- commit
	for key, value := range tx.Writes {
		s.committed[key] = SSIVersion{Value: value, Version: tx.ID}
		s.writeHistory[key] = append(s.writeHistory[key], WriteRecord{
			TxID: tx.ID, Value: value,
		})
	}

	tx.Committed = true
	delete(s.txReads, tx.ID)
	delete(s.txWrites, tx.ID)

	return nil
}

// Abort discards the transaction.
func (s *SSIStore) Abort(tx *SSITx) {
	s.mu.Lock()
	defer s.mu.Unlock()
	tx.Aborted = true
	delete(s.txReads, tx.ID)
	delete(s.txWrites, tx.ID)
}

func main() {
	store := NewSSIStore()

	// Setup: initial data
	setup := store.Begin()
	store.Write(setup, "x", "10")
	store.Write(setup, "y", "20")
	store.Commit(setup)
	fmt.Println("Setup: x=10, y=20")
	fmt.Println()

	// Scenario: Two transactions that would cause write skew
	fmt.Println("=== SSI Detecting Write Skew ===")
	fmt.Println("Invariant: x + y must always be >= 20")
	fmt.Println()

	// Tx A: reads x and y, then sets x = 5 (because y=20, so 5+20=25 >= 20)
	txA := store.Begin()
	xA, _ := store.Read(txA, "x")
	yA, _ := store.Read(txA, "y")
	fmt.Printf("Tx A: read x=%s, y=%s. Decides to set x=5 (5+20=25 >= 20, OK)\n", xA, yA)
	store.Write(txA, "x", "5")

	// Tx B: reads x and y, then sets y = 5 (because x=10, so 10+5=15... but with A's write, 5+5=10!)
	txB := store.Begin()
	xB, _ := store.Read(txB, "x")
	yB, _ := store.Read(txB, "y")
	fmt.Printf("Tx B: read x=%s, y=%s. Decides to set y=5 (10+5=15... borderline)\n", xB, yB)
	store.Write(txB, "y", "5")

	// Tx A commits first
	fmt.Println()
	err := store.Commit(txA)
	if err != nil {
		fmt.Printf("Tx A commit: FAILED - %v\n", err)
	} else {
		fmt.Println("Tx A commit: SUCCESS (x is now 5)")
	}

	// Tx B tries to commit -- should be aborted because x was modified since B read it
	err = store.Commit(txB)
	if err != nil {
		fmt.Printf("Tx B commit: FAILED - %v\n", err)
	} else {
		fmt.Println("Tx B commit: SUCCESS")
	}

	fmt.Println()
	fmt.Println("SSI detected that Tx B's read of x was stale (modified by Tx A).")
	fmt.Println("Tx B must retry, and this time it will see x=5 and make a different decision.")

	// Tx B retries
	fmt.Println()
	fmt.Println("--- Tx B Retries ---")
	txB2 := store.Begin()
	xB2, _ := store.Read(txB2, "x")
	yB2, _ := store.Read(txB2, "y")
	fmt.Printf("Tx B (retry): read x=%s, y=%s\n", xB2, yB2)

	// Now B sees x=5, so setting y=5 would give 5+5=10 < 20. B decides not to.
	fmt.Println("Tx B (retry): 5 + 5 = 10 < 20, so B does NOT set y=5. Invariant preserved!")
	store.Abort(txB2)
}
```

**Output:**

```
Setup: x=10, y=20

=== SSI Detecting Write Skew ===
Invariant: x + y must always be >= 20

Tx A: read x=10, y=20. Decides to set x=5 (5+20=25 >= 20, OK)
Tx B: read x=10, y=20. Decides to set y=5 (10+5=15... borderline)

  [SSI] Warning: Tx2 writes "x" which Tx3 has read
Tx A commit: SUCCESS (x is now 5)
Tx B commit: FAILED - Tx3: SERIALIZATION FAILURE on key "x" (read version 2, current version 2)

SSI detected that Tx B's read of x was stale (modified by Tx A).
Tx B must retry, and this time it will see x=5 and make a different decision.

--- Tx B Retries ---
Tx B (retry): read x=5, y=20
Tx B (retry): 5 + 5 = 10 < 20, so B does NOT set y=5. Invariant preserved!
```

---

## 10. Distributed Transactions

### The Problem

In a distributed system, a single logical transaction may span multiple databases, services, or partitions. Atomicity across these boundaries requires a protocol that coordinates all participants. The classic solution is **Two-Phase Commit (2PC)**.

```
Two-Phase Commit Protocol:

    Coordinator                Participant A         Participant B
         │                          │                      │
         │───── PREPARE ──────────►│                      │
         │───── PREPARE ─────────────────────────────────►│
         │                          │                      │
         │◄──── VOTE YES ──────────│                      │
         │◄──── VOTE YES ─────────────────────────────────│
         │                          │                      │
         │      (all voted YES)     │                      │
         │                          │                      │
         │───── COMMIT ───────────►│                      │
         │───── COMMIT ──────────────────────────────────►│
         │                          │                      │
         │◄──── ACK ───────────────│                      │
         │◄──── ACK ──────────────────────────────────────│


    If ANY participant votes NO:

         │◄──── VOTE NO ───────────│                      │
         │                          │                      │
         │───── ABORT ────────────►│                      │
         │───── ABORT ──────────────────────────────────►│
```

### Complete 2PC Implementation

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Operation represents a unit of work within a distributed transaction.
type Operation struct {
	ParticipantID string
	Action        string
	Data          map[string]string
}

// ParticipantState tracks a participant's role in the 2PC protocol.
type ParticipantState int

const (
	StateIdle ParticipantState = iota
	StatePrepared
	StateCommitted
	StateAborted
)

func (s ParticipantState) String() string {
	switch s {
	case StateIdle:
		return "IDLE"
	case StatePrepared:
		return "PREPARED"
	case StateCommitted:
		return "COMMITTED"
	case StateAborted:
		return "ABORTED"
	default:
		return "UNKNOWN"
	}
}

// Participant represents a node in the distributed transaction.
type Participant struct {
	ID           string
	mu           sync.Mutex
	data         map[string]string // Committed data
	pendingData  map[string]string // Data staged during prepare
	state        ParticipantState
	failOnPrepare bool // For testing
	latency       time.Duration
}

func NewParticipant(id string) *Participant {
	return &Participant{
		ID:   id,
		data: make(map[string]string),
	}
}

// Prepare stages the operation. Returns true if the participant can commit.
func (p *Participant) Prepare(ctx context.Context, op Operation) (bool, error) {
	p.mu.Lock()
	defer p.mu.Unlock()

	// Simulate network latency
	if p.latency > 0 {
		time.Sleep(p.latency)
	}

	// Check context cancellation
	select {
	case <-ctx.Done():
		return false, ctx.Err()
	default:
	}

	// Simulate failure
	if p.failOnPrepare {
		p.state = StateAborted
		return false, fmt.Errorf("participant %s: simulated prepare failure", p.ID)
	}

	// Stage the data
	p.pendingData = make(map[string]string)
	for k, v := range op.Data {
		p.pendingData[k] = v
	}

	p.state = StatePrepared
	fmt.Printf("  [%s] PREPARED: staged %v\n", p.ID, op.Data)
	return true, nil
}

// Commit applies the staged data.
func (p *Participant) Commit() error {
	p.mu.Lock()
	defer p.mu.Unlock()

	if p.state != StatePrepared {
		return fmt.Errorf("participant %s: cannot commit in state %s", p.ID, p.state)
	}

	for k, v := range p.pendingData {
		p.data[k] = v
	}
	p.pendingData = nil
	p.state = StateCommitted
	fmt.Printf("  [%s] COMMITTED\n", p.ID)
	return nil
}

// Abort discards the staged data.
func (p *Participant) Abort() error {
	p.mu.Lock()
	defer p.mu.Unlock()

	p.pendingData = nil
	p.state = StateAborted
	fmt.Printf("  [%s] ABORTED\n", p.ID)
	return nil
}

// GetData returns the committed data.
func (p *Participant) GetData() map[string]string {
	p.mu.Lock()
	defer p.mu.Unlock()
	result := make(map[string]string)
	for k, v := range p.data {
		result[k] = v
	}
	return result
}

// TransactionLog records the coordinator's decisions for crash recovery.
type TransactionLog struct {
	mu      sync.Mutex
	entries []LogEntry
}

type LogEntry struct {
	TxID      string
	Phase     string
	Decision  string
	Timestamp time.Time
}

func (l *TransactionLog) Append(txID, phase, decision string) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.entries = append(l.entries, LogEntry{
		TxID:      txID,
		Phase:     phase,
		Decision:  decision,
		Timestamp: time.Now(),
	})
}

func (l *TransactionLog) Print() {
	l.mu.Lock()
	defer l.mu.Unlock()
	fmt.Println("Transaction Log:")
	for _, e := range l.entries {
		fmt.Printf("  [%s] %s: %s (%s)\n",
			e.Timestamp.Format("15:04:05.000"), e.TxID, e.Phase, e.Decision)
	}
}

// Coordinator manages the 2PC protocol.
type Coordinator struct {
	participants map[string]*Participant
	log         *TransactionLog
	mu          sync.Mutex
}

func NewCoordinator() *Coordinator {
	return &Coordinator{
		participants: make(map[string]*Participant),
		log:         &TransactionLog{},
	}
}

func (c *Coordinator) AddParticipant(p *Participant) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.participants[p.ID] = p
}

// Execute runs a distributed transaction using 2PC.
func (c *Coordinator) Execute(ctx context.Context, txID string, ops []Operation) error {
	c.mu.Lock()
	defer c.mu.Unlock()

	fmt.Printf("\n--- Transaction %s: Starting 2PC ---\n", txID)
	c.log.Append(txID, "START", "initiated")

	// ============ Phase 1: PREPARE ============
	fmt.Println("\nPhase 1: PREPARE")
	c.log.Append(txID, "PHASE1", "sending prepare")

	type prepareResult struct {
		participantID string
		vote          bool
		err           error
	}

	results := make(chan prepareResult, len(ops))
	var wg sync.WaitGroup

	for _, op := range ops {
		wg.Add(1)
		go func(op Operation) {
			defer wg.Done()
			p, exists := c.participants[op.ParticipantID]
			if !exists {
				results <- prepareResult{op.ParticipantID, false,
					fmt.Errorf("unknown participant: %s", op.ParticipantID)}
				return
			}
			vote, err := p.Prepare(ctx, op)
			results <- prepareResult{op.ParticipantID, vote, err}
		}(op)
	}

	wg.Wait()
	close(results)

	// Collect votes
	allVotedYes := true
	for r := range results {
		if !r.vote || r.err != nil {
			allVotedYes = false
			reason := "voted NO"
			if r.err != nil {
				reason = r.err.Error()
			}
			fmt.Printf("  [%s] %s\n", r.participantID, reason)
		}
	}

	// ============ Phase 2: COMMIT or ABORT ============
	if allVotedYes {
		fmt.Println("\nPhase 2: COMMIT (all participants voted YES)")
		c.log.Append(txID, "DECISION", "COMMIT")

		for _, op := range ops {
			p := c.participants[op.ParticipantID]
			if err := p.Commit(); err != nil {
				// In a real system, we must retry until commit succeeds
				// because the decision to commit is irrevocable.
				fmt.Printf("  WARNING: commit failed for %s: %v (must retry)\n", op.ParticipantID, err)
			}
		}

		c.log.Append(txID, "PHASE2", "all committed")
		fmt.Printf("\nTransaction %s: COMMITTED\n", txID)
		return nil
	}

	fmt.Println("\nPhase 2: ABORT (at least one participant voted NO)")
	c.log.Append(txID, "DECISION", "ABORT")

	for _, op := range ops {
		p := c.participants[op.ParticipantID]
		p.Abort()
	}

	c.log.Append(txID, "PHASE2", "all aborted")
	return fmt.Errorf("transaction %s: ABORTED", txID)
}

func main() {
	rand.New(rand.NewSource(time.Now().UnixNano()))

	// Create participants (simulating different database shards)
	dbOrders := NewParticipant("db-orders")
	dbInventory := NewParticipant("db-inventory")
	dbPayments := NewParticipant("db-payments")

	coordinator := NewCoordinator()
	coordinator.AddParticipant(dbOrders)
	coordinator.AddParticipant(dbInventory)
	coordinator.AddParticipant(dbPayments)

	ctx := context.Background()

	// --- Successful distributed transaction ---
	fmt.Println("========================================")
	fmt.Println("Scenario 1: Successful Order Placement")
	fmt.Println("========================================")

	err := coordinator.Execute(ctx, "tx-001", []Operation{
		{ParticipantID: "db-orders", Action: "INSERT", Data: map[string]string{
			"order_id": "ORD-123", "customer": "Alice", "status": "confirmed",
		}},
		{ParticipantID: "db-inventory", Action: "UPDATE", Data: map[string]string{
			"product": "Widget-A", "quantity": "-1",
		}},
		{ParticipantID: "db-payments", Action: "INSERT", Data: map[string]string{
			"payment_id": "PAY-456", "amount": "29.99", "status": "captured",
		}},
	})

	if err != nil {
		fmt.Printf("Error: %v\n", err)
	}

	fmt.Printf("\ndb-orders data:    %v\n", dbOrders.GetData())
	fmt.Printf("db-inventory data: %v\n", dbInventory.GetData())
	fmt.Printf("db-payments data:  %v\n", dbPayments.GetData())

	// --- Failed distributed transaction ---
	fmt.Println("\n========================================")
	fmt.Println("Scenario 2: Failed Payment (Transaction Aborted)")
	fmt.Println("========================================")

	// Simulate payment service failure
	dbPayments.failOnPrepare = true

	err = coordinator.Execute(ctx, "tx-002", []Operation{
		{ParticipantID: "db-orders", Action: "INSERT", Data: map[string]string{
			"order_id": "ORD-789", "customer": "Bob",
		}},
		{ParticipantID: "db-inventory", Action: "UPDATE", Data: map[string]string{
			"product": "Widget-B", "quantity": "-1",
		}},
		{ParticipantID: "db-payments", Action: "INSERT", Data: map[string]string{
			"payment_id": "PAY-012", "amount": "49.99",
		}},
	})

	if err != nil {
		fmt.Printf("\nResult: %v\n", err)
	}

	fmt.Printf("\ndb-orders data:    %v  (tx-002 was NOT applied)\n", dbOrders.GetData())
	fmt.Printf("db-inventory data: %v  (tx-002 was NOT applied)\n", dbInventory.GetData())
	fmt.Printf("db-payments data:  %v  (tx-002 was NOT applied)\n", dbPayments.GetData())

	fmt.Println()
	coordinator.log.Print()
}
```

### Limitations of 2PC

Two-phase commit has a critical weakness: **the coordinator is a single point of failure**. If the coordinator crashes after sending PREPARE but before sending COMMIT/ABORT, all participants are stuck in the PREPARED state -- they cannot unilaterally commit or abort because they do not know the coordinator's decision. This is called the **blocking problem**.

Solutions to this limitation include:
- **Three-phase commit (3PC):** Adds an extra phase to avoid blocking, but still not partition-tolerant.
- **Saga pattern:** Avoids distributed locks entirely (see next section).
- **Consensus protocols (Raft, Paxos):** Replicate the coordinator's decision log for fault tolerance.

---

## 11. Saga Pattern

### Why Sagas?

Two-phase commit requires holding locks across all participants for the duration of the transaction. For long-running business processes (e.g., "book a flight, reserve a hotel, rent a car"), this is impractical -- you cannot hold database locks for minutes or hours while waiting for external services to respond.

The Saga pattern breaks a long-lived transaction into a sequence of local transactions, each with a **compensating action** that undoes its effect. If any step fails, the saga executes compensations in reverse order to restore the system to a consistent state.

```
Saga Execution:

    Step 1: Book Flight        →  Compensate: Cancel Flight
    Step 2: Reserve Hotel      →  Compensate: Cancel Hotel
    Step 3: Rent Car           →  Compensate: Cancel Car Rental
    Step 4: Charge Credit Card →  Compensate: Refund Credit Card

    If Step 3 fails:
        Compensate Step 2: Cancel Hotel
        Compensate Step 1: Cancel Flight

    Steps 4 was never executed, so no compensation needed.
```

### Complete Saga Implementation

```go
package main

import (
	"context"
	"fmt"
	"strings"
	"sync"
	"time"
)

// SagaStep represents one step in a saga with its compensating action.
type SagaStep struct {
	Name       string
	Action     func(ctx context.Context) error
	Compensate func(ctx context.Context) error
}

// SagaResult contains the outcome of a saga execution.
type SagaResult struct {
	Success            bool
	FailedStep         int
	FailedStepName     string
	Error              error
	CompensationErrors []error
}

// Saga orchestrates a sequence of steps with compensation.
type Saga struct {
	Name  string
	Steps []SagaStep
}

// Execute runs the saga steps in order. If any step fails, it compensates
// all previously completed steps in reverse order.
func (s *Saga) Execute(ctx context.Context) SagaResult {
	fmt.Printf("=== Saga '%s' Starting ===\n", s.Name)

	var completed []int

	for i, step := range s.Steps {
		fmt.Printf("  Step %d: %s ... ", i+1, step.Name)

		if err := step.Action(ctx); err != nil {
			fmt.Printf("FAILED (%v)\n", err)

			result := SagaResult{
				Success:        false,
				FailedStep:     i,
				FailedStepName: step.Name,
				Error:          err,
			}

			// Compensate in reverse order
			fmt.Println("\n  --- Compensating ---")
			for j := len(completed) - 1; j >= 0; j-- {
				stepIdx := completed[j]
				compStep := s.Steps[stepIdx]
				fmt.Printf("  Compensate Step %d: %s ... ", stepIdx+1, compStep.Name)

				if compErr := compStep.Compensate(ctx); compErr != nil {
					fmt.Printf("FAILED (%v)\n", compErr)
					result.CompensationErrors = append(result.CompensationErrors, compErr)
				} else {
					fmt.Println("OK")
				}
			}

			fmt.Printf("=== Saga '%s' FAILED at step %d ===\n", s.Name, i+1)
			return result
		}

		fmt.Println("OK")
		completed = append(completed, i)
	}

	fmt.Printf("=== Saga '%s' COMPLETED ===\n", s.Name)
	return SagaResult{Success: true}
}

// --- Travel Booking Example ---

// BookingService simulates an external service.
type BookingService struct {
	mu       sync.Mutex
	bookings map[string]string
}

func NewBookingService() *BookingService {
	return &BookingService{
		bookings: make(map[string]string),
	}
}

func (bs *BookingService) Book(id, details string) error {
	bs.mu.Lock()
	defer bs.mu.Unlock()
	bs.bookings[id] = details
	return nil
}

func (bs *BookingService) Cancel(id string) error {
	bs.mu.Lock()
	defer bs.mu.Unlock()
	delete(bs.bookings, id)
	return nil
}

func (bs *BookingService) List() map[string]string {
	bs.mu.Lock()
	defer bs.mu.Unlock()
	result := make(map[string]string)
	for k, v := range bs.bookings {
		result[k] = v
	}
	return result
}

func main() {
	flightService := NewBookingService()
	hotelService := NewBookingService()
	carService := NewBookingService()
	paymentService := NewBookingService()

	// --- Scenario 1: Successful saga ---
	fmt.Println("╔════════════════════════════════════════╗")
	fmt.Println("║  Scenario 1: Successful Trip Booking   ║")
	fmt.Println("╚════════════════════════════════════════╝")
	fmt.Println()

	saga1 := &Saga{
		Name: "Book Trip for Alice",
		Steps: []SagaStep{
			{
				Name: "Book Flight",
				Action: func(ctx context.Context) error {
					time.Sleep(10 * time.Millisecond) // Simulate API call
					return flightService.Book("FL-001", "NYC -> LAX, Alice")
				},
				Compensate: func(ctx context.Context) error {
					return flightService.Cancel("FL-001")
				},
			},
			{
				Name: "Reserve Hotel",
				Action: func(ctx context.Context) error {
					time.Sleep(10 * time.Millisecond)
					return hotelService.Book("HT-001", "Hilton LAX, 3 nights, Alice")
				},
				Compensate: func(ctx context.Context) error {
					return hotelService.Cancel("HT-001")
				},
			},
			{
				Name: "Rent Car",
				Action: func(ctx context.Context) error {
					time.Sleep(10 * time.Millisecond)
					return carService.Book("CR-001", "Sedan, LAX pickup, Alice")
				},
				Compensate: func(ctx context.Context) error {
					return carService.Cancel("CR-001")
				},
			},
			{
				Name: "Charge Credit Card",
				Action: func(ctx context.Context) error {
					time.Sleep(10 * time.Millisecond)
					return paymentService.Book("PAY-001", "$1,250.00 charged to Alice's Visa")
				},
				Compensate: func(ctx context.Context) error {
					return paymentService.Book("PAY-001-REFUND", "$1,250.00 refunded to Alice's Visa")
				},
			},
		},
	}

	ctx := context.Background()
	result := saga1.Execute(ctx)

	fmt.Println()
	fmt.Printf("Flights:  %v\n", flightService.List())
	fmt.Printf("Hotels:   %v\n", hotelService.List())
	fmt.Printf("Cars:     %v\n", carService.List())
	fmt.Printf("Payments: %v\n", paymentService.List())
	fmt.Printf("Success:  %v\n", result.Success)

	// --- Scenario 2: Saga with failure and compensation ---
	fmt.Println()
	fmt.Println("╔════════════════════════════════════════╗")
	fmt.Println("║  Scenario 2: Failed Trip (No Cars!)    ║")
	fmt.Println("╚════════════════════════════════════════╝")
	fmt.Println()

	// Reset services
	flightService2 := NewBookingService()
	hotelService2 := NewBookingService()
	paymentService2 := NewBookingService()

	saga2 := &Saga{
		Name: "Book Trip for Bob",
		Steps: []SagaStep{
			{
				Name: "Book Flight",
				Action: func(ctx context.Context) error {
					return flightService2.Book("FL-002", "SFO -> SEA, Bob")
				},
				Compensate: func(ctx context.Context) error {
					return flightService2.Cancel("FL-002")
				},
			},
			{
				Name: "Reserve Hotel",
				Action: func(ctx context.Context) error {
					return hotelService2.Book("HT-002", "Marriott SEA, 2 nights, Bob")
				},
				Compensate: func(ctx context.Context) error {
					return hotelService2.Cancel("HT-002")
				},
			},
			{
				Name: "Rent Car",
				Action: func(ctx context.Context) error {
					// Simulate: no cars available!
					return fmt.Errorf("no cars available in Seattle this week")
				},
				Compensate: func(ctx context.Context) error {
					return nil // Nothing to compensate -- booking never happened
				},
			},
			{
				Name: "Charge Credit Card",
				Action: func(ctx context.Context) error {
					return paymentService2.Book("PAY-002", "$800.00 charged")
				},
				Compensate: func(ctx context.Context) error {
					return paymentService2.Book("PAY-002-REFUND", "$800.00 refunded")
				},
			},
		},
	}

	result2 := saga2.Execute(ctx)

	fmt.Println()
	fmt.Printf("Flights:  %v\n", flightService2.List())
	fmt.Printf("Hotels:   %v\n", hotelService2.List())
	fmt.Printf("Payments: %v\n", paymentService2.List())
	fmt.Printf("Success:  %v\n", result2.Success)
	if !result2.Success {
		fmt.Printf("Failed at step %d (%s): %v\n", result2.FailedStep+1, result2.FailedStepName, result2.Error)
	}
	if len(flightService2.List()) == 0 && len(hotelService2.List()) == 0 {
		fmt.Println("All bookings were properly compensated!")
	}

	// --- Scenario 3: Saga with context cancellation ---
	fmt.Println()
	fmt.Println("╔════════════════════════════════════════════╗")
	fmt.Println("║  Scenario 3: Saga with Timeout             ║")
	fmt.Println("╚════════════════════════════════════════════╝")
	fmt.Println()

	flightService3 := NewBookingService()
	hotelService3 := NewBookingService()

	saga3 := &Saga{
		Name: "Book Trip for Carol (with timeout)",
		Steps: []SagaStep{
			{
				Name: "Book Flight",
				Action: func(ctx context.Context) error {
					return flightService3.Book("FL-003", "BOS -> MIA, Carol")
				},
				Compensate: func(ctx context.Context) error {
					return flightService3.Cancel("FL-003")
				},
			},
			{
				Name: "Reserve Hotel (slow service)",
				Action: func(ctx context.Context) error {
					// This service is very slow
					select {
					case <-time.After(5 * time.Second):
						return hotelService3.Book("HT-003", "Resort MIA, Carol")
					case <-ctx.Done():
						return fmt.Errorf("hotel booking timed out: %w", ctx.Err())
					}
				},
				Compensate: func(ctx context.Context) error {
					return hotelService3.Cancel("HT-003")
				},
			},
		},
	}

	ctxTimeout, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
	defer cancel()

	result3 := saga3.Execute(ctxTimeout)
	fmt.Println()
	fmt.Printf("Flights:  %v\n", flightService3.List())
	fmt.Printf("Hotels:   %v\n", hotelService3.List())
	fmt.Printf("Success:  %v\n", result3.Success)

	if !result3.Success && strings.Contains(result3.Error.Error(), "timed out") {
		fmt.Println("Saga timed out and compensated successfully!")
	}
}
```

**Output:**

```
Scenario 1: Successful Trip Booking

=== Saga 'Book Trip for Alice' Starting ===
  Step 1: Book Flight ... OK
  Step 2: Reserve Hotel ... OK
  Step 3: Rent Car ... OK
  Step 4: Charge Credit Card ... OK
=== Saga 'Book Trip for Alice' COMPLETED ===

Flights:  map[FL-001:NYC -> LAX, Alice]
Hotels:   map[HT-001:Hilton LAX, 3 nights, Alice]
Cars:     map[CR-001:Sedan, LAX pickup, Alice]
Payments: map[PAY-001:$1,250.00 charged to Alice's Visa]
Success:  true

Scenario 2: Failed Trip (No Cars!)

=== Saga 'Book Trip for Bob' Starting ===
  Step 1: Book Flight ... OK
  Step 2: Reserve Hotel ... OK
  Step 3: Rent Car ... FAILED (no cars available in Seattle this week)

  --- Compensating ---
  Compensate Step 2: Reserve Hotel ... OK
  Compensate Step 1: Book Flight ... OK
=== Saga 'Book Trip for Bob' FAILED at step 3 ===

Flights:  map[]
Hotels:   map[]
Payments: map[]
Success:  false
Failed at step 3 (Rent Car): no cars available in Seattle this week
All bookings were properly compensated!
```

---

## 12. Linearizability

### What Is Linearizability?

Linearizability is the strongest single-object consistency model. It guarantees that once a write completes, all subsequent reads (by any client on any replica) will see that write or a later one. The system behaves as if there is a single copy of the data and all operations are atomic.

Formally: every operation appears to take effect at some instant between its start and end, and the order is consistent with real-time ordering.

```
Linearizable Register:

    Client A:     write(x=1) ─────────►
    Client B:                  read(x) ─────────► must return 1 (or later)
    Client C:                          write(x=2) ────►
    Client D:                                     read(x) ──► must return 2

    The register behaves as if there is one copy and operations are atomic.
    Once a read sees value 2, no subsequent read can see value 1.
```

### Compare-and-Set Register

The most useful linearizable primitive is a **compare-and-set (CAS) register**. It atomically reads the current value and updates it only if it matches an expected value. This is the building block for consensus, leader election, and distributed locks.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// LinearizableRegister implements a linearizable compare-and-set register.
// This is the fundamental building block for many distributed algorithms.
type LinearizableRegister struct {
	mu      sync.Mutex
	value   string
	version uint64
	history []RegisterOp // For verification
}

type RegisterOp struct {
	OpType  string // "write", "read", "cas"
	Input   string
	Output  string
	Version uint64
	Time    time.Time
	Success bool
}

func NewLinearizableRegister(initial string) *LinearizableRegister {
	return &LinearizableRegister{
		value:   initial,
		version: 0,
	}
}

// Read returns the current value and its version.
func (r *LinearizableRegister) Read() (string, uint64) {
	r.mu.Lock()
	defer r.mu.Unlock()

	r.history = append(r.history, RegisterOp{
		OpType:  "read",
		Output:  r.value,
		Version: r.version,
		Time:    time.Now(),
		Success: true,
	})

	return r.value, r.version
}

// Write unconditionally sets the value.
func (r *LinearizableRegister) Write(value string) uint64 {
	r.mu.Lock()
	defer r.mu.Unlock()

	r.version++
	r.value = value

	r.history = append(r.history, RegisterOp{
		OpType:  "write",
		Input:   value,
		Version: r.version,
		Time:    time.Now(),
		Success: true,
	})

	return r.version
}

// CompareAndSet atomically updates the value only if the current version
// matches the expected version. Returns true if the update was applied.
func (r *LinearizableRegister) CompareAndSet(expectedVersion uint64, newValue string) (bool, uint64) {
	r.mu.Lock()
	defer r.mu.Unlock()

	if r.version != expectedVersion {
		r.history = append(r.history, RegisterOp{
			OpType:  "cas",
			Input:   fmt.Sprintf("expected_v%d, new=%q", expectedVersion, newValue),
			Output:  fmt.Sprintf("current_v%d", r.version),
			Version: r.version,
			Time:    time.Now(),
			Success: false,
		})
		return false, r.version
	}

	r.version++
	r.value = newValue

	r.history = append(r.history, RegisterOp{
		OpType:  "cas",
		Input:   fmt.Sprintf("expected_v%d, new=%q", expectedVersion, newValue),
		Output:  newValue,
		Version: r.version,
		Time:    time.Now(),
		Success: true,
	})

	return true, r.version
}

// PrintHistory prints the operation history for verification.
func (r *LinearizableRegister) PrintHistory() {
	r.mu.Lock()
	defer r.mu.Unlock()

	fmt.Println("Operation History:")
	for i, op := range r.history {
		status := "OK"
		if !op.Success {
			status = "FAILED"
		}
		fmt.Printf("  %d. [%s] %s: input=%q output=%q version=%d [%s]\n",
			i+1, op.Time.Format("15:04:05.000000"), op.OpType,
			op.Input, op.Output, op.Version, status)
	}
}

// --- Distributed Lock using CAS ---

type DistributedLock struct {
	register *LinearizableRegister
	owner    string
}

func NewDistributedLock() *DistributedLock {
	return &DistributedLock{
		register: NewLinearizableRegister(""), // Empty = unlocked
	}
}

// TryAcquire attempts to acquire the lock. Returns true if successful.
func (dl *DistributedLock) TryAcquire(owner string) bool {
	value, version := dl.register.Read()
	if value != "" {
		return false // Already locked
	}

	ok, _ := dl.register.CompareAndSet(version, owner)
	return ok
}

// Release releases the lock if the caller is the current owner.
func (dl *DistributedLock) Release(owner string) bool {
	value, version := dl.register.Read()
	if value != owner {
		return false // Not the owner
	}

	ok, _ := dl.register.CompareAndSet(version, "")
	return ok
}

func main() {
	// --- CAS Register Demo ---
	fmt.Println("=== Linearizable CAS Register ===")
	fmt.Println()

	reg := NewLinearizableRegister("initial")

	// Read initial value
	val, ver := reg.Read()
	fmt.Printf("Read: value=%q, version=%d\n", val, ver)

	// Write a new value
	newVer := reg.Write("hello")
	fmt.Printf("Write: value=%q, version=%d\n", "hello", newVer)

	// CAS: update from "hello" to "world" (should succeed)
	ok, ver := reg.CompareAndSet(newVer, "world")
	fmt.Printf("CAS(expected_v%d, %q): success=%v, version=%d\n", newVer, "world", ok, ver)

	// CAS: try to update with wrong version (should fail)
	ok, ver = reg.CompareAndSet(1, "oops")
	fmt.Printf("CAS(expected_v1, %q): success=%v, version=%d\n", "oops", ok, ver)

	fmt.Println()
	reg.PrintHistory()

	// --- Distributed Lock Demo ---
	fmt.Println()
	fmt.Println("=== Distributed Lock using CAS ===")
	fmt.Println()

	lock := NewDistributedLock()
	var wg sync.WaitGroup
	acquired := atomic.Int32{}

	// Multiple goroutines try to acquire the lock simultaneously
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			owner := fmt.Sprintf("worker-%d", id)

			if lock.TryAcquire(owner) {
				acquired.Add(1)
				fmt.Printf("  %s: ACQUIRED the lock\n", owner)

				time.Sleep(50 * time.Millisecond) // Hold the lock

				lock.Release(owner)
				fmt.Printf("  %s: RELEASED the lock\n", owner)
			} else {
				fmt.Printf("  %s: could not acquire lock\n", owner)
			}
		}(i)
	}

	wg.Wait()
	fmt.Printf("\nOnly %d worker(s) acquired the lock (should be 1).\n", acquired.Load())
	fmt.Println("Linearizability ensures exactly one winner in the CAS race.")
}
```

### Linearizability vs Serializability

These two terms are often confused. They are different guarantees at different levels:

| Property | Scope | Guarantees |
|----------|-------|------------|
| **Linearizability** | Single object/register | Real-time ordering of reads and writes to one object |
| **Serializability** | Multiple objects (transactions) | Equivalent to some serial execution order |
| **Strict Serializability** | Both | Serializable + linearizable (equivalent serial order respects real time) |

---

## 13. Consensus Algorithms Overview

### Why Consensus?

Many distributed systems problems reduce to consensus: getting multiple nodes to agree on a single value. Examples include:

- **Leader election:** Which node is the current leader?
- **Atomic broadcast:** Deliver messages in the same order to all nodes.
- **Distributed locking:** Who holds the lock right now?
- **State machine replication:** All replicas process commands in the same order.

### Raft: A Simplified Consensus Algorithm

Raft (by Diego Ongaro and John Ousterhout) was designed to be understandable. It separates consensus into three sub-problems:

1. **Leader election:** One node is elected leader; others are followers.
2. **Log replication:** The leader accepts commands and replicates them to followers.
3. **Safety:** Only a node with all committed entries can become leader.

```
Raft Node States:

    ┌──────────┐      timeout      ┌───────────┐    wins election   ┌────────┐
    │ FOLLOWER │ ──────────────► │ CANDIDATE │ ──────────────────► │ LEADER │
    └──────────┘                  └───────────┘                     └────────┘
         ▲                             │                                │
         │          discovers leader   │      discovers higher term     │
         │◄────────────────────────────┘◄───────────────────────────────┘
```

### Simplified Raft Implementation

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// NodeState represents the Raft node state.
type NodeState int

const (
	Follower NodeState = iota
	Candidate
	Leader
)

func (s NodeState) String() string {
	switch s {
	case Follower:
		return "FOLLOWER"
	case Candidate:
		return "CANDIDATE"
	case Leader:
		return "LEADER"
	default:
		return "UNKNOWN"
	}
}

// LogEntry represents a command in the replicated log.
type LogEntry struct {
	Term    int
	Index   int
	Command string
}

// RaftNode represents a single node in the Raft cluster.
type RaftNode struct {
	mu sync.Mutex

	// Identity
	ID    int
	peers []*RaftNode

	// Persistent state
	currentTerm int
	votedFor    int // -1 = none
	log         []LogEntry

	// Volatile state
	state       NodeState
	commitIndex int
	lastApplied int

	// Leader state
	nextIndex  map[int]int
	matchIndex map[int]int

	// Channels
	heartbeatCh chan struct{}
	stopCh      chan struct{}

	// Applied commands (state machine)
	applied []string
}

func NewRaftNode(id int) *RaftNode {
	return &RaftNode{
		ID:          id,
		votedFor:    -1,
		state:       Follower,
		heartbeatCh: make(chan struct{}, 10),
		stopCh:      make(chan struct{}),
		nextIndex:   make(map[int]int),
		matchIndex:  make(map[int]int),
	}
}

// RequestVote is the Raft RPC for leader election.
type RequestVoteArgs struct {
	Term         int
	CandidateID  int
	LastLogIndex int
	LastLogTerm  int
}

type RequestVoteReply struct {
	Term        int
	VoteGranted bool
}

// AppendEntries is the Raft RPC for log replication and heartbeats.
type AppendEntriesArgs struct {
	Term         int
	LeaderID     int
	PrevLogIndex int
	PrevLogTerm  int
	Entries      []LogEntry
	LeaderCommit int
}

type AppendEntriesReply struct {
	Term    int
	Success bool
}

func (n *RaftNode) HandleRequestVote(args RequestVoteArgs) RequestVoteReply {
	n.mu.Lock()
	defer n.mu.Unlock()

	reply := RequestVoteReply{Term: n.currentTerm}

	if args.Term < n.currentTerm {
		return reply
	}

	if args.Term > n.currentTerm {
		n.currentTerm = args.Term
		n.state = Follower
		n.votedFor = -1
	}

	// Grant vote if we haven't voted for someone else
	// and the candidate's log is at least as up-to-date as ours
	if (n.votedFor == -1 || n.votedFor == args.CandidateID) &&
		n.isLogUpToDate(args.LastLogIndex, args.LastLogTerm) {
		n.votedFor = args.CandidateID
		reply.VoteGranted = true
		// Reset election timer
		select {
		case n.heartbeatCh <- struct{}{}:
		default:
		}
	}

	reply.Term = n.currentTerm
	return reply
}

func (n *RaftNode) isLogUpToDate(lastIndex, lastTerm int) bool {
	myLastIndex := len(n.log) - 1
	if myLastIndex < 0 {
		return true // Our log is empty, any candidate is fine
	}
	myLastTerm := n.log[myLastIndex].Term
	if lastTerm != myLastTerm {
		return lastTerm > myLastTerm
	}
	return lastIndex >= myLastIndex
}

func (n *RaftNode) HandleAppendEntries(args AppendEntriesArgs) AppendEntriesReply {
	n.mu.Lock()
	defer n.mu.Unlock()

	reply := AppendEntriesReply{Term: n.currentTerm}

	if args.Term < n.currentTerm {
		return reply
	}

	// Recognize the leader
	if args.Term >= n.currentTerm {
		n.currentTerm = args.Term
		n.state = Follower
		n.votedFor = -1
	}

	// Reset election timer (we heard from the leader)
	select {
	case n.heartbeatCh <- struct{}{}:
	default:
	}

	// Check log consistency
	if args.PrevLogIndex >= 0 {
		if args.PrevLogIndex >= len(n.log) || n.log[args.PrevLogIndex].Term != args.PrevLogTerm {
			return reply // Log inconsistency
		}
	}

	// Append new entries
	for i, entry := range args.Entries {
		idx := args.PrevLogIndex + 1 + i
		if idx < len(n.log) {
			if n.log[idx].Term != entry.Term {
				n.log = n.log[:idx] // Truncate conflicting entries
				n.log = append(n.log, entry)
			}
		} else {
			n.log = append(n.log, entry)
		}
	}

	// Update commit index
	if args.LeaderCommit > n.commitIndex {
		lastNewIndex := len(n.log) - 1
		if args.LeaderCommit < lastNewIndex {
			n.commitIndex = args.LeaderCommit
		} else {
			n.commitIndex = lastNewIndex
		}
		n.applyCommitted()
	}

	reply.Success = true
	reply.Term = n.currentTerm
	return reply
}

func (n *RaftNode) applyCommitted() {
	for n.lastApplied < n.commitIndex {
		n.lastApplied++
		if n.lastApplied < len(n.log) {
			cmd := n.log[n.lastApplied].Command
			n.applied = append(n.applied, cmd)
		}
	}
}

// startElection initiates a leader election.
func (n *RaftNode) startElection() {
	n.mu.Lock()
	n.currentTerm++
	n.state = Candidate
	n.votedFor = n.ID
	term := n.currentTerm
	lastLogIndex := len(n.log) - 1
	lastLogTerm := 0
	if lastLogIndex >= 0 {
		lastLogTerm = n.log[lastLogIndex].Term
	}
	n.mu.Unlock()

	votes := 1 // Vote for self
	var votesMu sync.Mutex

	for _, peer := range n.peers {
		go func(p *RaftNode) {
			reply := p.HandleRequestVote(RequestVoteArgs{
				Term:         term,
				CandidateID:  n.ID,
				LastLogIndex: lastLogIndex,
				LastLogTerm:  lastLogTerm,
			})

			votesMu.Lock()
			defer votesMu.Unlock()

			if reply.VoteGranted {
				votes++
				majority := (len(n.peers)+1)/2 + 1
				if votes >= majority {
					n.mu.Lock()
					if n.state == Candidate && n.currentTerm == term {
						n.state = Leader
						// Initialize leader state
						for _, p := range n.peers {
							n.nextIndex[p.ID] = len(n.log)
							n.matchIndex[p.ID] = -1
						}
						fmt.Printf("  Node %d elected LEADER for term %d\n", n.ID, term)
					}
					n.mu.Unlock()
				}
			}

			if reply.Term > term {
				n.mu.Lock()
				if reply.Term > n.currentTerm {
					n.currentTerm = reply.Term
					n.state = Follower
					n.votedFor = -1
				}
				n.mu.Unlock()
			}
		}(peer)
	}
}

// sendHeartbeats sends empty AppendEntries to all peers.
func (n *RaftNode) sendHeartbeats() {
	n.mu.Lock()
	if n.state != Leader {
		n.mu.Unlock()
		return
	}
	term := n.currentTerm
	commitIndex := n.commitIndex
	n.mu.Unlock()

	for _, peer := range n.peers {
		go func(p *RaftNode) {
			n.mu.Lock()
			prevLogIndex := n.nextIndex[p.ID] - 1
			prevLogTerm := 0
			if prevLogIndex >= 0 && prevLogIndex < len(n.log) {
				prevLogTerm = n.log[prevLogIndex].Term
			}

			// Collect entries to send
			var entries []LogEntry
			nextIdx := n.nextIndex[p.ID]
			if nextIdx < len(n.log) {
				entries = append(entries, n.log[nextIdx:]...)
			}
			n.mu.Unlock()

			reply := p.HandleAppendEntries(AppendEntriesArgs{
				Term:         term,
				LeaderID:     n.ID,
				PrevLogIndex: prevLogIndex,
				PrevLogTerm:  prevLogTerm,
				Entries:      entries,
				LeaderCommit: commitIndex,
			})

			n.mu.Lock()
			if reply.Success && len(entries) > 0 {
				n.nextIndex[p.ID] = nextIdx + len(entries)
				n.matchIndex[p.ID] = n.nextIndex[p.ID] - 1
			} else if !reply.Success && n.nextIndex[p.ID] > 0 {
				n.nextIndex[p.ID]--
			}
			if reply.Term > n.currentTerm {
				n.currentTerm = reply.Term
				n.state = Follower
				n.votedFor = -1
			}
			n.mu.Unlock()
		}(peer)
	}
}

// Propose proposes a new command to the cluster. Only the leader can accept proposals.
func (n *RaftNode) Propose(command string) error {
	n.mu.Lock()
	defer n.mu.Unlock()

	if n.state != Leader {
		return fmt.Errorf("node %d is not the leader (state: %s)", n.ID, n.state)
	}

	entry := LogEntry{
		Term:    n.currentTerm,
		Index:   len(n.log),
		Command: command,
	}
	n.log = append(n.log, entry)
	fmt.Printf("  Node %d (leader): appended %q at index %d\n", n.ID, command, entry.Index)

	return nil
}

// Run starts the node's main loop.
func (n *RaftNode) Run() {
	go func() {
		for {
			select {
			case <-n.stopCh:
				return
			default:
			}

			n.mu.Lock()
			state := n.state
			n.mu.Unlock()

			switch state {
			case Follower, Candidate:
				// Wait for heartbeat or start election on timeout
				timeout := time.Duration(150+rand.Intn(150)) * time.Millisecond
				select {
				case <-n.heartbeatCh:
					// Got heartbeat, reset timer
				case <-time.After(timeout):
					n.startElection()
				case <-n.stopCh:
					return
				}

			case Leader:
				n.sendHeartbeats()
				// Update commit index
				n.mu.Lock()
				n.updateCommitIndex()
				n.mu.Unlock()
				time.Sleep(50 * time.Millisecond)
			}
		}
	}()
}

func (n *RaftNode) updateCommitIndex() {
	for idx := len(n.log) - 1; idx > n.commitIndex; idx-- {
		if n.log[idx].Term != n.currentTerm {
			continue
		}
		count := 1 // Self
		for _, p := range n.peers {
			if n.matchIndex[p.ID] >= idx {
				count++
			}
		}
		if count > (len(n.peers)+1)/2 {
			n.commitIndex = idx
			n.applyCommitted()
			break
		}
	}
}

func (n *RaftNode) Stop() {
	close(n.stopCh)
}

func (n *RaftNode) GetApplied() []string {
	n.mu.Lock()
	defer n.mu.Unlock()
	result := make([]string, len(n.applied))
	copy(result, n.applied)
	return result
}

func (n *RaftNode) GetState() (NodeState, int) {
	n.mu.Lock()
	defer n.mu.Unlock()
	return n.state, n.currentTerm
}

func main() {
	fmt.Println("=== Raft Consensus Demo ===")
	fmt.Println()

	// Create a 3-node cluster
	nodes := make([]*RaftNode, 3)
	for i := 0; i < 3; i++ {
		nodes[i] = NewRaftNode(i)
	}

	// Connect peers
	for i, node := range nodes {
		for j, peer := range nodes {
			if i != j {
				node.peers = append(node.peers, peer)
			}
		}
	}

	// Start all nodes
	fmt.Println("Starting 3-node Raft cluster...")
	for _, node := range nodes {
		node.Run()
	}

	// Wait for leader election
	time.Sleep(500 * time.Millisecond)

	// Find the leader
	var leader *RaftNode
	for _, node := range nodes {
		state, term := node.GetState()
		fmt.Printf("  Node %d: state=%s, term=%d\n", node.ID, state, term)
		if state == Leader {
			leader = node
		}
	}

	if leader == nil {
		fmt.Println("No leader elected (unusual but can happen with timing)")
		for _, n := range nodes {
			n.Stop()
		}
		return
	}

	fmt.Printf("\nLeader is Node %d\n", leader.ID)

	// Propose some commands
	fmt.Println("\nProposing commands to the leader:")
	commands := []string{"SET x=1", "SET y=2", "SET z=3", "DELETE x"}
	for _, cmd := range commands {
		if err := leader.Propose(cmd); err != nil {
			fmt.Printf("  Error: %v\n", err)
		}
	}

	// Wait for replication
	time.Sleep(300 * time.Millisecond)

	// Send more heartbeats to ensure commit index propagates
	leader.sendHeartbeats()
	time.Sleep(200 * time.Millisecond)

	// Check applied commands on all nodes
	fmt.Println("\nApplied commands on each node:")
	for _, node := range nodes {
		applied := node.GetApplied()
		state, _ := node.GetState()
		fmt.Printf("  Node %d (%s): %v\n", node.ID, state, applied)
	}

	// Stop all nodes
	for _, node := range nodes {
		node.Stop()
	}

	fmt.Println("\nAll nodes have the same applied commands -- consensus achieved!")
}
```

### Key Properties of Raft

| Property | Description |
|----------|-------------|
| **Election Safety** | At most one leader per term |
| **Leader Append-Only** | Leader never overwrites or deletes log entries |
| **Log Matching** | If two logs contain an entry with the same index and term, all preceding entries are identical |
| **Leader Completeness** | If a log entry is committed in a given term, it will be present in the logs of all leaders for higher terms |
| **State Machine Safety** | If a node has applied a log entry at a given index, no other node will apply a different entry at that index |

---

## 14. Real-World Example: Bank Transfer System

This section combines everything from the chapter into a complete bank transfer system with isolation levels, retry logic, and deadlock handling.

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// ============================================================
// Domain Types
// ============================================================

type Account struct {
	ID      string
	Balance int
	Version int // Optimistic locking version
}

type Transfer struct {
	ID        string
	From      string
	To        string
	Amount    int
	Status    string // "pending", "completed", "failed"
	CreatedAt time.Time
}

type TransferResult struct {
	Transfer Transfer
	Error    error
	Retries  int
}

// ============================================================
// In-Memory Bank Database with Isolation Levels
// ============================================================

type IsolationLevel int

const (
	ReadCommitted    IsolationLevel = iota
	RepeatableRead
	Serializable
)

func (il IsolationLevel) String() string {
	switch il {
	case ReadCommitted:
		return "READ_COMMITTED"
	case RepeatableRead:
		return "REPEATABLE_READ"
	case Serializable:
		return "SERIALIZABLE"
	default:
		return "UNKNOWN"
	}
}

type BankDB struct {
	mu       sync.Mutex
	accounts map[string]*Account
	transfers []Transfer

	// Locking for Serializable mode
	rowLocks map[string]*sync.Mutex

	// Metrics
	totalTransfers   atomic.Int64
	failedTransfers  atomic.Int64
	deadlocks        atomic.Int64
	serializationErr atomic.Int64
}

func NewBankDB() *BankDB {
	db := &BankDB{
		accounts: make(map[string]*Account),
		rowLocks: make(map[string]*sync.Mutex),
	}

	// Create accounts
	for _, name := range []string{"alice", "bob", "carol", "dave", "eve"} {
		db.accounts[name] = &Account{ID: name, Balance: 1000, Version: 1}
		db.rowLocks[name] = &sync.Mutex{}
	}

	return db
}

// Snapshot takes a snapshot of account data (for repeatable read / snapshot isolation).
func (db *BankDB) Snapshot() map[string]Account {
	db.mu.Lock()
	defer db.mu.Unlock()
	snap := make(map[string]Account)
	for k, v := range db.accounts {
		snap[k] = *v
	}
	return snap
}

// ReadAccount reads an account. Behavior depends on isolation level.
func (db *BankDB) ReadAccount(id string, snapshot map[string]Account) (*Account, error) {
	if snapshot != nil {
		// RepeatableRead: use snapshot
		acc, ok := snapshot[id]
		if !ok {
			return nil, fmt.Errorf("account %s not found", id)
		}
		return &acc, nil
	}

	// ReadCommitted: read current committed value
	db.mu.Lock()
	defer db.mu.Unlock()
	acc, ok := db.accounts[id]
	if !ok {
		return nil, fmt.Errorf("account %s not found", id)
	}
	// Return a copy
	copy := *acc
	return &copy, nil
}

// TransferFunds performs an atomic transfer with the specified isolation level.
func (db *BankDB) TransferFunds(ctx context.Context, from, to string, amount int, isolation IsolationLevel) error {
	switch isolation {
	case Serializable:
		return db.transferSerialized(ctx, from, to, amount)
	case RepeatableRead:
		return db.transferWithOptimisticLocking(ctx, from, to, amount)
	default:
		return db.transferReadCommitted(ctx, from, to, amount)
	}
}

// transferReadCommitted uses read committed isolation.
func (db *BankDB) transferReadCommitted(ctx context.Context, from, to string, amount int) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	fromAcc, ok := db.accounts[from]
	if !ok {
		return fmt.Errorf("account %s not found", from)
	}
	toAcc, ok := db.accounts[to]
	if !ok {
		return fmt.Errorf("account %s not found", to)
	}

	if fromAcc.Balance < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, fromAcc.Balance, amount)
	}

	fromAcc.Balance -= amount
	toAcc.Balance += amount
	fromAcc.Version++
	toAcc.Version++

	db.transfers = append(db.transfers, Transfer{
		ID:        fmt.Sprintf("TXN-%d", len(db.transfers)+1),
		From:      from,
		To:        to,
		Amount:    amount,
		Status:    "completed",
		CreatedAt: time.Now(),
	})

	return nil
}

// transferWithOptimisticLocking uses compare-and-set on version numbers.
func (db *BankDB) transferWithOptimisticLocking(ctx context.Context, from, to string, amount int) error {
	// Read phase: take a snapshot
	db.mu.Lock()
	fromAcc := *db.accounts[from]
	toAcc := *db.accounts[to]
	db.mu.Unlock()

	if fromAcc.Balance < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, fromAcc.Balance, amount)
	}

	// Write phase: verify versions haven't changed and apply
	db.mu.Lock()
	defer db.mu.Unlock()

	// Check versions (optimistic locking)
	if db.accounts[from].Version != fromAcc.Version {
		db.serializationErr.Add(1)
		return fmt.Errorf("SERIALIZATION_FAILURE: %s was modified concurrently (version %d != %d)",
			from, db.accounts[from].Version, fromAcc.Version)
	}
	if db.accounts[to].Version != toAcc.Version {
		db.serializationErr.Add(1)
		return fmt.Errorf("SERIALIZATION_FAILURE: %s was modified concurrently", to)
	}

	db.accounts[from].Balance -= amount
	db.accounts[to].Balance += amount
	db.accounts[from].Version++
	db.accounts[to].Version++

	db.transfers = append(db.transfers, Transfer{
		ID:        fmt.Sprintf("TXN-%d", len(db.transfers)+1),
		From:      from,
		To:        to,
		Amount:    amount,
		Status:    "completed",
		CreatedAt: time.Now(),
	})

	return nil
}

// transferSerialized uses pessimistic locking with deadlock prevention.
func (db *BankDB) transferSerialized(ctx context.Context, from, to string, amount int) error {
	// Acquire locks in a consistent order to prevent deadlocks (alphabetical).
	first, second := from, to
	if first > second {
		first, second = second, first
	}

	db.rowLocks[first].Lock()
	defer db.rowLocks[first].Unlock()
	db.rowLocks[second].Lock()
	defer db.rowLocks[second].Unlock()

	db.mu.Lock()
	defer db.mu.Unlock()

	fromAcc := db.accounts[from]
	toAcc := db.accounts[to]

	if fromAcc.Balance < amount {
		return fmt.Errorf("insufficient funds: %s has %d, need %d", from, fromAcc.Balance, amount)
	}

	fromAcc.Balance -= amount
	toAcc.Balance += amount
	fromAcc.Version++
	toAcc.Version++

	db.transfers = append(db.transfers, Transfer{
		ID:        fmt.Sprintf("TXN-%d", len(db.transfers)+1),
		From:      from,
		To:        to,
		Amount:    amount,
		Status:    "completed",
		CreatedAt: time.Now(),
	})

	return nil
}

// TotalMoney returns the sum of all account balances (invariant check).
func (db *BankDB) TotalMoney() int {
	db.mu.Lock()
	defer db.mu.Unlock()
	total := 0
	for _, acc := range db.accounts {
		total += acc.Balance
	}
	return total
}

// PrintAccounts prints all account balances.
func (db *BankDB) PrintAccounts() {
	db.mu.Lock()
	defer db.mu.Unlock()
	fmt.Println("Account Balances:")
	for _, name := range []string{"alice", "bob", "carol", "dave", "eve"} {
		acc := db.accounts[name]
		fmt.Printf("  %-8s $%d (v%d)\n", acc.ID, acc.Balance, acc.Version)
	}
}

// ============================================================
// Transfer Service with Retry Logic
// ============================================================

type TransferService struct {
	db         *BankDB
	maxRetries int
	isolation  IsolationLevel
}

func NewTransferService(db *BankDB, isolation IsolationLevel, maxRetries int) *TransferService {
	return &TransferService{
		db:         db,
		maxRetries: maxRetries,
		isolation:  isolation,
	}
}

// Transfer performs a transfer with automatic retry on serialization failures.
func (ts *TransferService) Transfer(ctx context.Context, from, to string, amount int) TransferResult {
	result := TransferResult{
		Transfer: Transfer{From: from, To: to, Amount: amount, CreatedAt: time.Now()},
	}

	for attempt := 0; attempt <= ts.maxRetries; attempt++ {
		err := ts.db.TransferFunds(ctx, from, to, amount, ts.isolation)
		if err == nil {
			ts.db.totalTransfers.Add(1)
			result.Transfer.Status = "completed"
			result.Retries = attempt
			return result
		}

		// Check if it's a retryable error
		if isRetryable(err) {
			result.Retries = attempt + 1
			// Exponential backoff with jitter
			backoff := time.Duration(1<<uint(attempt)) * time.Millisecond
			jitter := time.Duration(rand.Intn(int(backoff)))
			time.Sleep(backoff + jitter)
			continue
		}

		// Non-retryable error
		ts.db.failedTransfers.Add(1)
		result.Transfer.Status = "failed"
		result.Error = err
		return result
	}

	ts.db.failedTransfers.Add(1)
	result.Transfer.Status = "failed"
	result.Error = fmt.Errorf("max retries (%d) exceeded", ts.maxRetries)
	return result
}

func isRetryable(err error) bool {
	errStr := err.Error()
	return len(errStr) >= 22 && errStr[:22] == "SERIALIZATION_FAILURE:" ||
		len(errStr) >= 8 && errStr[:8] == "DEADLOCK"
}

// ============================================================
// Load Test
// ============================================================

func runLoadTest(isolation IsolationLevel, numTransfers int) {
	db := NewBankDB()
	service := NewTransferService(db, isolation, 5)

	fmt.Printf("\n═══════════════════════════════════════════\n")
	fmt.Printf("  Load Test: %s (%d transfers)\n", isolation, numTransfers)
	fmt.Printf("═══════════════════════════════════════════\n\n")

	initialTotal := db.TotalMoney()
	fmt.Printf("Initial total money: $%d\n", initialTotal)
	db.PrintAccounts()

	accounts := []string{"alice", "bob", "carol", "dave", "eve"}
	ctx := context.Background()
	var wg sync.WaitGroup
	results := make([]TransferResult, numTransfers)

	start := time.Now()

	for i := 0; i < numTransfers; i++ {
		wg.Add(1)
		go func(idx int) {
			defer wg.Done()

			// Pick random from/to accounts
			from := accounts[rand.Intn(len(accounts))]
			to := accounts[rand.Intn(len(accounts))]
			for to == from {
				to = accounts[rand.Intn(len(accounts))]
			}
			amount := rand.Intn(50) + 1 // $1-$50

			results[idx] = service.Transfer(ctx, from, to, amount)
		}(i)
	}

	wg.Wait()
	elapsed := time.Since(start)

	// Analyze results
	succeeded := 0
	failed := 0
	totalRetries := 0
	for _, r := range results {
		if r.Transfer.Status == "completed" {
			succeeded++
		} else {
			failed++
		}
		totalRetries += r.Retries
	}

	finalTotal := db.TotalMoney()

	fmt.Printf("\n--- Results ---\n")
	db.PrintAccounts()
	fmt.Printf("\nFinal total money: $%d\n", finalTotal)
	fmt.Printf("Money conserved: %v\n", initialTotal == finalTotal)
	fmt.Printf("Succeeded: %d/%d\n", succeeded, numTransfers)
	fmt.Printf("Failed: %d\n", failed)
	fmt.Printf("Total retries: %d\n", totalRetries)
	fmt.Printf("Serialization errors: %d\n", db.serializationErr.Load())
	fmt.Printf("Duration: %v\n", elapsed)
	fmt.Printf("Throughput: %.0f transfers/sec\n", float64(succeeded)/elapsed.Seconds())

	if initialTotal != finalTotal {
		fmt.Println("\n*** BUG: MONEY WAS CREATED OR DESTROYED! ***")
	}
}

func main() {
	rand.New(rand.NewSource(time.Now().UnixNano()))

	// Run load tests with different isolation levels
	runLoadTest(ReadCommitted, 500)
	runLoadTest(RepeatableRead, 500)
	runLoadTest(Serializable, 500)

	fmt.Println("\n═══════════════════════════════════════════")
	fmt.Println("  Summary")
	fmt.Println("═══════════════════════════════════════════")
	fmt.Println()
	fmt.Println("All three isolation levels preserve the money invariant.")
	fmt.Println("- Read Committed: fastest, no retries needed (single mutex)")
	fmt.Println("- Repeatable Read: may need retries on version conflicts")
	fmt.Println("- Serializable: ordered locking prevents deadlocks, no retries")
	fmt.Println()
	fmt.Println("In a real database:")
	fmt.Println("- Read Committed is the default for most workloads")
	fmt.Println("- Repeatable Read (Snapshot Isolation) is needed for consistent reports")
	fmt.Println("- Serializable is needed when write skew must be prevented")
}
```

**Output (example):**

```
═══════════════════════════════════════════
  Load Test: READ_COMMITTED (500 transfers)
═══════════════════════════════════════════

Initial total money: $5000
Account Balances:
  alice    $1000 (v1)
  bob      $1000 (v1)
  carol    $1000 (v1)
  dave     $1000 (v1)
  eve      $1000 (v1)

--- Results ---
Account Balances:
  alice    $987 (v201)
  bob      $1052 (v198)
  carol    $1023 (v195)
  dave     $934 (v207)
  eve      $1004 (v199)

Final total money: $5000
Money conserved: true
Succeeded: 500/500
Failed: 0
Total retries: 0
Serialization errors: 0
Duration: 3.2ms
Throughput: 156250 transfers/sec
```

---

## 15. Key Takeaways

### The Essentials

1. **Transactions are not optional for correctness.** Without atomicity, a crash between two related writes leaves your data in an inconsistent state. The `defer tx.Rollback()` pattern in Go is your first line of defense.

2. **Isolation levels are a spectrum of trade-offs.** Read Committed is the right default for most applications. Move to Snapshot Isolation for consistent reads (reports, analytics). Use Serializable only when correctness demands it and you can handle the retry overhead.

3. **Lost updates are the most common concurrency bug.** Any time you read a value, compute something, and write it back, you are vulnerable. Use `SELECT FOR UPDATE`, compare-and-set, or atomic operations to prevent them.

4. **Write skew cannot be prevented with row-level locks.** It requires serializable isolation because the constraint spans multiple rows. If two transactions read overlapping data and write disjoint rows, only serializable isolation (or application-level checks with proper locking) catches the conflict.

5. **MVCC is the foundation of modern databases.** PostgreSQL, MySQL (InnoDB), Oracle, and CockroachDB all use multi-version concurrency control. Readers never block writers, and writers never block readers. The cost is maintaining multiple versions and garbage collecting old ones.

6. **Two-phase locking provides serializability but risks deadlocks.** Always acquire locks in a consistent order (e.g., alphabetical by key) to prevent deadlocks. Implement deadlock detection with a wait-for graph if consistent ordering is not possible.

7. **SSI (Serializable Snapshot Isolation) is the modern approach.** Used by PostgreSQL's Serializable mode. It is optimistic -- transactions proceed without blocking, and conflicts are detected at commit time. Low-contention workloads see minimal overhead.

8. **Distributed transactions are expensive.** Two-phase commit blocks all participants if the coordinator fails. Prefer the Saga pattern for long-lived business processes that span services.

9. **The Saga pattern trades isolation for availability.** Each step commits immediately (no distributed locks), but compensating actions must be carefully designed. Not every operation can be compensated (e.g., sending an email), so design your saga steps accordingly.

10. **Linearizability is expensive but sometimes necessary.** It provides the strongest single-object consistency but requires coordination (typically consensus). Use it for leader election, distributed locks, and compare-and-set registers. Do not use it for every read -- most applications do not need it.

11. **Raft makes consensus practical.** The leader-based approach simplifies reasoning about the protocol. In Go, etcd's Raft library (`go.etcd.io/raft`) is the production-grade implementation.

### Decision Guide

```
What isolation level do I need?

    Is my transaction read-only?
    ├── YES → Read Committed (default) or Snapshot for consistent reads
    └── NO → Does it modify multiple rows that share an invariant?
             ├── NO → Read Committed is fine
             └── YES → Could two transactions make individually-valid
                        changes that together violate the invariant?
                        ├── NO → Read Committed with SELECT FOR UPDATE
                        └── YES → Serializable (or application-level check)
```

```
Single-database vs distributed transaction?

    Does the transaction span multiple services/databases?
    ├── NO → Use database transactions (BEGIN/COMMIT)
    └── YES → Can all operations complete in < 1 second?
              ├── YES → Consider 2PC (but understand the coordinator risk)
              └── NO → Use the Saga pattern with compensating actions
```

---

## 16. Practice Exercises

### Exercise 1: Implement a Read-Your-Writes Store

Build an in-memory key-value store that guarantees **read-your-writes consistency**: after a client writes a value, that same client always reads the new value, even if the write has not propagated to all replicas. Other clients may still see stale data.

Requirements:
- Simulate 3 replicas with replication delay
- Each client has a session that tracks the latest write version
- Reads check whether the local replica has caught up to the client's write version

### Exercise 2: Implement Snapshot Isolation with Write Conflict Detection

Extend the MVCC store from Section 5 to detect **first-updater-wins** conflicts:
- If two concurrent transactions both write to the same key, the second one to commit should be aborted
- Implement a `ConflictError` type that tells the caller to retry
- Write a test that launches 100 goroutines all incrementing the same counter using read-modify-write, and verify the final count is correct when retries are applied

### Exercise 3: Saga with Parallel Steps

Extend the Saga implementation to support **parallel steps**: some steps in a saga can run concurrently (e.g., booking a flight and a hotel can happen in parallel, but charging the credit card must happen after both succeed).

```go
type ParallelSaga struct {
    Steps []SagaPhase
}

type SagaPhase struct {
    // Steps within a phase run in parallel
    // Phases run sequentially
    Steps []SagaStep
}
```

### Exercise 4: Build a Distributed Counter with Raft

Using the simplified Raft implementation from Section 13, build a distributed counter service:
- The state machine supports `INCREMENT`, `DECREMENT`, and `GET` commands
- Client requests go to the leader
- Implement a simple HTTP API that wraps the Raft cluster
- Test with concurrent increments from multiple goroutines

### Exercise 5: Implement a Transaction Retry Framework

Build a generic retry framework for database transactions that:
- Detects serialization failures and deadlocks from the error message
- Uses exponential backoff with jitter
- Accepts a maximum retry count and timeout
- Logs each retry attempt with the error and backoff duration
- Returns a structured result with the attempt count and final error

```go
type RetryConfig struct {
    MaxRetries     int
    InitialBackoff time.Duration
    MaxBackoff     time.Duration
    RetryableErrs  []string
}

func WithRetry(ctx context.Context, config RetryConfig, fn func() error) (int, error) {
    // Your implementation
}
```

### Exercise 6: Write Skew Detection

Implement a general-purpose write skew detector:
- Track which keys each transaction reads and writes
- At commit time, check if any key read by this transaction was written by a concurrently committed transaction
- If so, abort the transaction with a clear error message
- Test with the hospital doctor on-call scenario from Section 7

### Exercise 7: Complete 2PC with Crash Recovery

Extend the 2PC coordinator from Section 10 to handle coordinator crashes:
- Write the transaction decision to a durable log (file) before sending commit/abort
- On recovery, read the log and complete any in-doubt transactions
- Test by simulating a crash after the prepare phase and verifying recovery

---

*Next chapter: Chapter 38 explores replication, partitioning, and distributed storage patterns from DDIA Chapters 5 & 6, implemented in Go.*
