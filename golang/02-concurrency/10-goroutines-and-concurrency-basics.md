# Chapter 10: Goroutines and Concurrency Fundamentals

> **Level**: Intermediate
> **Prerequisites**: Chapters 1-9 (Go basics, functions, structs, interfaces, error handling)
> **Key Idea**: Go was designed from the ground up for concurrency. This chapter covers the foundational primitives that make Go one of the most productive languages for concurrent programming.

---

## Table of Contents

1. [Concurrency vs Parallelism](#1-concurrency-vs-parallelism)
2. [What Are Goroutines?](#2-what-are-goroutines)
3. [Starting Goroutines](#3-starting-goroutines)
4. [The Go Scheduler](#4-the-go-scheduler)
5. [Goroutine Lifecycle](#5-goroutine-lifecycle)
6. [sync.WaitGroup](#6-syncwaitgroup)
7. [sync.Mutex and sync.RWMutex](#7-syncmutex-and-syncrwmutex)
8. [Race Conditions](#8-race-conditions)
9. [sync.Once and sync.Map](#9-synconce-and-syncmap)
10. [Deep Comparison: Go Concurrency vs Node.js](#10-deep-comparison-go-concurrency-vs-nodejs)
11. [Key Takeaways](#11-key-takeaways)
12. [Practice Exercises](#12-practice-exercises)

---

## 1. Concurrency vs Parallelism

This is the single most important conceptual distinction in this entire chapter. Get this right and everything else falls into place.

### Definitions

**Concurrency** is about *dealing with* multiple things at once. It is a *structural* property of your program -- how you organize and compose independently executing tasks.

**Parallelism** is about *doing* multiple things at once. It is a *runtime execution* property -- whether two computations are literally running at the same instant on different CPU cores.

### Rob Pike's Famous Explanation

Rob Pike (one of Go's creators) gave a talk titled *"Concurrency Is Not Parallelism"* that every Go developer should watch. His key insight:

> **Concurrency is about structure. Parallelism is about execution.**

Think of it this way:

- A single-core machine **can** run a concurrent program. The OS rapidly switches between tasks, giving the illusion of simultaneous execution. The program is concurrent (structured to handle multiple tasks) but NOT parallel (only one thing executes at any instant).
- A parallel program requires multiple cores. Two goroutines literally execute at the same physical instant on different CPUs.

### Visual Diagram

```
CONCURRENCY (Structure - one core)
===================================

    Core 1
    ┌──────────────────────────────────────────────┐
    │ Task A ██░░██░░░░██░░░░░░░░██░░              │
    │ Task B ░░██░░██░░░░██░░░░░░░░██░░            │
    │ Task C ░░░░░░░░██░░░░██████░░░░██            │
    └──────────────────────────────────────────────┘
              ▲        ▲         ▲
              │        │         │
        context switches (tasks interleave on ONE core)

    Tasks take turns. Only one runs at any instant.
    But the PROGRAM is structured to handle all three.


PARALLELISM (Execution - multiple cores)
=========================================

    Core 1  │ Task A ██████████████████████████████
    ─────────────────────────────────────────────────
    Core 2  │ Task B ██████████████████████████████
    ─────────────────────────────────────────────────
    Core 3  │ Task C ██████████████████████████████

    Tasks literally run at the same instant on different cores.


GO WITH GOROUTINES (Concurrent + Potentially Parallel)
=======================================================

    Core 1  │ G1 ██░░G3 ██░░G1 ██░░G5 ██░░G1 ██
    ─────────────────────────────────────────────────
    Core 2  │ G2 ██░░G4 ██░░G2 ██░░G6 ██░░G2 ██
    ─────────────────────────────────────────────────
    Core 3  │ G7 ██░░G8 ██░░G9 ██░░G7 ██░░G10█
    ─────────────────────────────────────────────────
    Core 4  │ G11██░░G12██░░G11██░░G13██░░G14█

    Hundreds of goroutines are MULTIPLEXED onto a few OS threads.
    The Go scheduler decides which goroutine runs where.
```

### Why This Distinction Matters

1. **You write concurrent code; the runtime decides parallelism.** You create goroutines (concurrency). The Go runtime maps them to OS threads across CPU cores (parallelism). You never manually assign goroutines to cores.

2. **Concurrency is useful even without parallelism.** A web server handling 10,000 connections concurrently on a single core is still a massive win -- each connection is an independent task that mostly waits on I/O.

3. **Parallelism without concurrency is useless.** You need a well-structured concurrent program before parallelism can help. Throwing more cores at a poorly structured program accomplishes nothing.

4. **Bugs come from concurrency, not parallelism.** Race conditions, deadlocks, and data corruption are consequences of concurrent structure, not of how many cores you have. Understanding this helps you reason about where bugs hide.

---

## 2. What Are Goroutines?

A goroutine is a lightweight, independently executing function managed by the Go runtime. It is Go's fundamental unit of concurrency.

### Goroutines vs OS Threads

| Property | OS Thread | Goroutine |
|---|---|---|
| **Initial stack size** | ~1-8 MB (fixed) | ~2 KB (dynamically grows) |
| **Creation cost** | Expensive (kernel syscall) | Cheap (runtime bookkeeping) |
| **Context switch** | Slow (kernel mode switch, TLB flush) | Fast (user-space, ~200ns) |
| **Scheduling** | OS kernel scheduler | Go runtime scheduler (user-space) |
| **Practical limit** | Thousands (memory-limited) | Hundreds of thousands to millions |
| **Identity** | Has thread ID | No exposed ID (by design) |
| **Communication** | Shared memory + locks | Channels (preferred) + shared memory |

### Why Goroutines Are So Cheap

**Stack size: 2 KB vs 1+ MB**

An OS thread typically reserves 1-8 MB of virtual memory for its stack. Creating 10,000 OS threads would require 10-80 GB of stack space alone. That is physically impossible on most machines.

A goroutine starts with just ~2 KB of stack. Creating 10,000 goroutines requires only ~20 MB. Creating 1,000,000 goroutines requires ~2 GB -- large but feasible.

The magic: goroutine stacks **grow dynamically**. If a goroutine needs more stack, the runtime allocates a larger one, copies the contents, and updates all pointers. This is called a **segmented stack** (historically) or **contiguous stack** (current implementation). Most goroutines never need more than their initial 2 KB.

**No kernel involvement**

Creating an OS thread requires a system call (like `clone()` on Linux). The CPU must switch to kernel mode, allocate kernel data structures, set up the thread's context, and return to user mode. This takes microseconds.

Creating a goroutine is a function call within the Go runtime. It allocates a small struct, sets up the stack, and puts the goroutine on a run queue. This takes nanoseconds.

**User-space scheduling**

When the OS switches between threads, it must save/restore the full CPU state (registers, program counter, stack pointer, TLB, etc.) and potentially flush caches. This is slow.

When the Go runtime switches between goroutines, it only saves/restores a small set of registers. There is no kernel involvement, no TLB flush, no cache pollution. This is an order of magnitude faster.

### Practical Demonstration

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func main() {
    // How many goroutines can we create?
    numGoroutines := 100_000

    var wg sync.WaitGroup
    wg.Add(numGoroutines)

    start := time.Now()

    for i := 0; i < numGoroutines; i++ {
        go func() {
            defer wg.Done()
            // Each goroutine does minimal work
            time.Sleep(1 * time.Second)
        }()
    }

    fmt.Printf("Launched %d goroutines in %v\n", numGoroutines, time.Since(start))
    fmt.Printf("Active goroutines: %d\n", runtime.NumGoroutine())

    wg.Wait()
    fmt.Printf("All goroutines completed in %v\n", time.Since(start))
}
```

Output (approximate):
```
Launched 100000 goroutines in 45ms
Active goroutines: 100001
All goroutines completed in 1.05s
```

100,000 goroutines launched in under 50ms. Try doing that with OS threads in C or Java.

---

## 3. Starting Goroutines

### The `go` Keyword

Starting a goroutine is the simplest thing in Go. Prefix any function call with the `go` keyword:

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    // Regular function call - blocks until complete
    sayHello("synchronous")

    // Goroutine - returns IMMEDIATELY, runs concurrently
    go sayHello("goroutine")

    // Without this sleep, main() would exit before the goroutine prints
    // (We will learn proper synchronization soon with WaitGroup)
    time.Sleep(100 * time.Millisecond)
}
```

The `go` keyword is what makes Go's concurrency model so accessible. Compare this to other languages:

```
Go:          go doWork()
Java:        new Thread(() -> doWork()).start();
C:           pthread_create(&thread, NULL, doWork, NULL);
Python:      threading.Thread(target=doWork).start()
C#:          Task.Run(() => DoWork());
Rust:        std::thread::spawn(|| do_work());
```

Go's syntax is the simplest, and goroutines are the cheapest. That simplicity is intentional -- the designers wanted concurrency to be a natural, everyday tool rather than a heavyweight construct you avoid until absolutely necessary.

### Anonymous Goroutines

You can launch goroutines with anonymous (inline) functions. This is extremely common:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    // Anonymous goroutine - very common pattern
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("I am an anonymous goroutine")
    }()

    // Anonymous goroutine with arguments
    wg.Add(1)
    go func(msg string) {
        defer wg.Done()
        fmt.Println(msg)
    }("Hello from goroutine")

    wg.Wait()
}
```

### CRITICAL: The Loop Variable Trap

This is the single most common goroutine bug for beginners:

```go
// BUG: All goroutines may print the SAME value
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // CAPTURES the variable i, not its value
    }()
}
// Likely output: 5 5 5 5 5 (all print the final value of i)
```

**Why does this happen?** The anonymous function captures the *variable* `i`, not its *value*. By the time the goroutines execute, the loop has already finished and `i` equals 5.

**Fix 1: Pass as argument (classic approach)**
```go
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n) // n is a copy, unique to each goroutine
    }(i) // i is COPIED into n at launch time
}
```

**Fix 2: Shadow the variable (Go 1.22+ loop variable change)**

Starting with Go 1.22, loop variables are scoped per-iteration by default, which eliminates this class of bugs. But it is still important to understand the underlying issue, both for reading older code and for understanding closures in general.

```go
// Go 1.22+: Each iteration of the loop creates a new 'i'
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // Safe in Go 1.22+ -- each iteration has its own i
    }()
}
```

### Goroutines with Return Values

Goroutines cannot return values directly. The `go` keyword discards any return value:

```go
// This does NOT work -- you cannot capture the return value
// result := go computeSomething()  // COMPILE ERROR

// Instead, use channels (Chapter 11) or shared state with synchronization:
var result int
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    result = computeSomething() // Write to shared variable
}()
wg.Wait()
fmt.Println(result) // Safe to read after Wait()
```

---

## 4. The Go Scheduler

The Go scheduler is one of the language's greatest engineering achievements. Understanding it (at a high level) helps you write better concurrent code and debug performance issues.

### M:N Scheduling

Go uses an **M:N scheduling model**: M goroutines are multiplexed onto N OS threads.

```
M:N SCHEDULING MODEL
======================

    Goroutines (G)         OS Threads (M)           CPU Cores (P)
    ┌────────────┐
    │ G1  G2  G3 │
    │ G4  G5  G6 │──────▶  M1 ──────────────────▶  Core 1
    │ G7  G8  G9 │         M2 ──────────────────▶  Core 2
    │ G10 G11 G12│         M3 ──────────────────▶  Core 3
    │ G13 G14 G15│         M4 ──────────────────▶  Core 4
    │ ... (1000s) │
    └────────────┘
                           (handful of threads)     (physical cores)

    1000+ goroutines  ───▶  ~4-8 OS threads  ───▶  4-8 CPU cores
```

### The GMP Model

The Go scheduler uses three primary entities, often called **GMP**:

**G (Goroutine):** The goroutine itself. Contains the stack, instruction pointer, and other scheduling-related information (e.g., the channel it is blocking on).

**M (Machine):** An OS thread. The "machine" that actually executes Go code. An M must be associated with a P to run goroutines.

**P (Processor):** A logical processor. Represents a resource needed to execute Go code. P holds the local run queue of goroutines. The number of Ps is set by `GOMAXPROCS`.

```
THE GMP MODEL (Detailed)
=========================

    ┌─────────────────────────────────────────────────────┐
    │                   Global Run Queue                   │
    │              [G10] [G11] [G12] [G13]                 │
    └──────────────────────┬──────────────────────────────┘
                           │ (steal when local queue empty)
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │    P1    │    │    P2    │    │    P3    │
    │ Local Q: │    │ Local Q: │    │ Local Q: │
    │ [G1][G4] │    │ [G2][G5] │    │ [G3][G6] │
    │ [G7]     │    │ [G8]     │    │ [G9]     │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
    ┌────▼─────┐    ┌────▼─────┐    ┌────▼─────┐
    │    M1    │    │    M2    │    │    M3    │
    │(OS Thread│    │(OS Thread│    │(OS Thread│
    │ running  │    │ running  │    │ running  │
    │   G1)    │    │   G2)    │    │   G3)    │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
    ┌────▼─────┐    ┌────▼─────┐    ┌────▼─────┐
    │  Core 1  │    │  Core 2  │    │  Core 3  │
    └──────────┘    └──────────┘    └──────────┘
```

### How the Scheduler Works (High Level)

1. **When a new goroutine is created** (`go func()`), it is placed on the local run queue of the current P. If the local queue is full, half the goroutines are moved to the global run queue.

2. **When a goroutine blocks** (on I/O, channel operation, mutex, syscall), the scheduler detaches the goroutine from the M. The M picks up another runnable goroutine from the P's local queue. This means blocking one goroutine does NOT block the OS thread (in most cases).

3. **When a goroutine does a syscall** that cannot be made non-blocking (e.g., file I/O), the M is blocked. The P detaches from the blocked M and attaches to a new M (or creates one) to continue running other goroutines. When the syscall completes, the goroutine is put back on a run queue.

4. **Work stealing:** When a P's local run queue is empty, it tries to steal goroutines from other Ps' local queues (takes half). If that fails, it checks the global run queue. If that is also empty, it checks the network poller for ready I/O goroutines.

```
WORK STEALING IN ACTION
=========================

    Before:
    P1: [G1, G2, G3, G4]     P2: [empty]
         ▲                          │
         │                          │
         └──────── steal! ──────────┘
                 (P2 steals half of P1's queue)

    After:
    P1: [G1, G2]              P2: [G3, G4]

    This keeps all cores busy. No goroutine starves.
```

### GOMAXPROCS

`GOMAXPROCS` controls how many Ps (logical processors) exist, which determines the maximum number of goroutines that can execute *simultaneously*:

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    // Default: number of CPU cores available
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0)) // 0 means "query, don't change"

    // You CAN set it manually (rarely needed)
    runtime.GOMAXPROCS(1) // Only one goroutine runs at a time
    // Concurrency still works, but no parallelism

    runtime.GOMAXPROCS(runtime.NumCPU()) // Restore to all cores (default since Go 1.5)
}
```

**When would you change GOMAXPROCS?**

Almost never. The default (all CPU cores) is correct for the vast majority of workloads. Setting it to 1 can be useful for debugging race conditions (makes execution more deterministic) or for benchmarking single-core performance.

### Why This Design? (Efficiency + Simplicity)

1. **Efficiency:** Thousands of goroutines share a handful of OS threads. Context switches are cheap (user-space). Blocking I/O does not waste threads.

2. **Developer simplicity:** You never think about threads, thread pools, or executor services. You just write `go doSomething()` and the runtime handles the rest.

3. **No callback hell:** Unlike event-loop models (Node.js), each goroutine reads like sequential, blocking code. The scheduler handles the async machinery behind the scenes.

---

## 5. Goroutine Lifecycle

### States of a Goroutine

```
GOROUTINE LIFECYCLE
====================

                     go func()
                         │
                         ▼
                    ┌─────────┐
                    │ CREATED │
                    │(Runnable│
                    │  queue) │
                    └────┬────┘
                         │  scheduler picks it up
                         ▼
                    ┌─────────┐
              ┌────▶│ RUNNING │◀────┐
              │     └────┬────┘     │
              │          │          │
              │    ┌─────┴─────┐   │
              │    ▼           ▼   │
         ┌─────────┐   ┌──────────┐
         │BLOCKED  │   │ WAITING  │
         │(syscall,│   │(channel, │
         │ mutex)  │   │ timer,   │
         └────┬────┘   │ select)  │
              │        └────┬─────┘
              │             │
              │    unblocked│
              └─────────────┘
                         │
                   function returns
                         │
                         ▼
                    ┌─────────┐
                    │  DEAD   │
                    │ (GC'd)  │
                    └─────────┘
```

1. **Created (Runnable):** The goroutine is created and placed on a run queue. It has not started executing yet.

2. **Running:** The goroutine is actively executing on an M (OS thread).

3. **Blocked/Waiting:** The goroutine is suspended because it is waiting for something -- a channel operation, a mutex lock, a syscall, a timer, etc.

4. **Dead:** The goroutine's function returned. Its resources are freed by the garbage collector.

### The Critical Rule: When main() Exits, Everything Dies

This is the most important lifecycle rule to understand:

```go
package main

import "fmt"

func main() {
    go func() {
        fmt.Println("Hello from goroutine") // MAY NEVER PRINT
    }()

    // main() returns immediately.
    // The program exits.
    // The goroutine is killed instantly -- no cleanup, no deferred functions.
    // This is NOT a graceful shutdown.
}
```

**The Go runtime does NOT wait for goroutines to finish.** When `main()` returns, the process terminates, and every goroutine is destroyed without warning. This is fundamentally different from how some other languages handle threads (e.g., Java daemon threads vs user threads).

**This is why synchronization is not optional -- it is mandatory.** You MUST use one of:
- `sync.WaitGroup` (this chapter)
- Channels (Chapter 11)
- `context.Context` (Chapter 12)
- `select` with a done signal (Chapter 11)

### Demonstrating the Problem

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int) {
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second) // Simulate work
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    for i := 1; i <= 5; i++ {
        go worker(i)
    }

    // BAD: Using time.Sleep to wait for goroutines
    // This is fragile -- what if workers take longer than expected?
    // time.Sleep(2 * time.Second)

    fmt.Println("Main exiting")
    // Output: "Main exiting" (workers may or may not print anything)
}
```

---

## 6. sync.WaitGroup

`sync.WaitGroup` is the most fundamental synchronization primitive in Go. It lets you wait for a collection of goroutines to finish.

### How It Works

A WaitGroup is essentially an atomic counter:
- `Add(n)` increments the counter by n
- `Done()` decrements the counter by 1 (equivalent to `Add(-1)`)
- `Wait()` blocks until the counter reaches 0

```
WAITGROUP FLOW
===============

    main goroutine              worker goroutines
    ─────────────               ─────────────────
    wg.Add(3)
    go worker1()  ──────────▶   worker1 starts
    go worker2()  ──────────▶   worker2 starts
    go worker3()  ──────────▶   worker3 starts
    wg.Wait()                   worker1 finishes ──▶ wg.Done() (counter: 2)
       │ (blocked)              worker3 finishes ──▶ wg.Done() (counter: 1)
       │ (blocked)              worker2 finishes ──▶ wg.Done() (counter: 0)
       │
       ▼ (unblocked! counter == 0)
    continue...
```

### Basic Pattern

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // ALWAYS use defer to guarantee Done() is called

    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)           // Increment BEFORE launching goroutine
        go worker(i, &wg)   // Pass pointer -- WaitGroup must NOT be copied
    }

    wg.Wait() // Block until all 5 workers call Done()
    fmt.Println("All workers completed")
}
```

### Common Mistakes

**Mistake 1: Calling Add() inside the goroutine**

```go
// BUG: Race condition between Add() and Wait()
for i := 0; i < 5; i++ {
    go func() {
        wg.Add(1) // TOO LATE -- main might call Wait() before this runs
        defer wg.Done()
        doWork()
    }()
}
wg.Wait() // Might return before all goroutines have called Add()
```

**Fix:** Always call `Add()` in the launching goroutine, *before* the `go` statement.

```go
for i := 0; i < 5; i++ {
    wg.Add(1) // BEFORE the go statement
    go func() {
        defer wg.Done()
        doWork()
    }()
}
wg.Wait()
```

**Mistake 2: Passing WaitGroup by value**

```go
// BUG: WaitGroup is copied -- Done() decrements the copy, not the original
func worker(wg sync.WaitGroup) { // WRONG: passed by value
    defer wg.Done()
    doWork()
}
```

**Fix:** Always pass `*sync.WaitGroup` (a pointer):

```go
func worker(wg *sync.WaitGroup) { // CORRECT: passed by pointer
    defer wg.Done()
    doWork()
}
```

**Mistake 3: Forgetting Done()**

If a goroutine does not call `Done()` (e.g., it panics without a deferred `Done()`), `Wait()` blocks forever (deadlock). Always use `defer wg.Done()` as the first line in your goroutine.

**Mistake 4: Negative counter**

```go
var wg sync.WaitGroup
wg.Add(1)
wg.Done()
wg.Done() // PANIC: sync: negative WaitGroup counter
```

The counter must never go below zero. Ensure every `Add(1)` is paired with exactly one `Done()`.

### Practical Example: Parallel URL Fetcher

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

func fetchURL(url string, wg *sync.WaitGroup) {
    defer wg.Done()

    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        fmt.Printf("ERROR fetching %s: %v\n", url, err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("%-40s %d bytes  %v\n", url, len(body), time.Since(start))
}

func main() {
    urls := []string{
        "https://www.google.com",
        "https://www.github.com",
        "https://www.golang.org",
        "https://www.stackoverflow.com",
        "https://www.reddit.com",
    }

    var wg sync.WaitGroup

    start := time.Now()
    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, &wg)
    }
    wg.Wait()

    fmt.Printf("\nAll %d URLs fetched in %v\n", len(urls), time.Since(start))
    // All 5 fetched in ~200-500ms (parallel) instead of ~2-5s (sequential)
}
```

---

## 7. sync.Mutex and sync.RWMutex

### Why Mutexes Are Needed

Goroutines share the same memory space. When multiple goroutines read and write the same variable without synchronization, the result is a **data race** -- undefined behavior that can produce corrupted data, crashes, or subtly wrong results.

### The Problem: Unsynchronized Access

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DATA RACE: multiple goroutines read-modify-write
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter)
    // Expected: 1000
    // Actual: some number less than 1000 (e.g., 937, 982, 1000 -- unpredictable)
}
```

**Why does this happen?** `counter++` is not atomic. It compiles to three operations:

```
1. READ  counter into a register     (value: 42)
2. ADD   1 to the register            (value: 43)
3. WRITE register back to counter     (counter = 43)
```

When two goroutines do this simultaneously:

```
Goroutine A                     Goroutine B
─────────────                   ─────────────
READ counter (42)
                                READ counter (42)   ← same value!
ADD 1 (43)
                                ADD 1 (43)          ← same result!
WRITE counter (43)
                                WRITE counter (43)  ← overwrites A's write!

Result: counter = 43, but TWO increments happened. One was lost.
```

### sync.Mutex -- Mutual Exclusion

A Mutex ensures only one goroutine can access a critical section at a time:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            mu.Lock()         // Acquire the lock (blocks if another goroutine holds it)
            counter++         // Only ONE goroutine can be here at a time
            mu.Unlock()       // Release the lock
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter) // Always 1000
}
```

### Mutex Best Practices

**Always use defer for Unlock:**

```go
mu.Lock()
defer mu.Unlock()
// ... critical section ...
// Unlock is guaranteed even if a panic occurs
```

**Keep critical sections small:**

```go
// BAD: Holding the lock during expensive operations
mu.Lock()
result := expensiveComputation() // Other goroutines are BLOCKED waiting
data[key] = result
mu.Unlock()

// GOOD: Only lock when accessing shared state
result := expensiveComputation() // No lock needed -- local variable
mu.Lock()
data[key] = result               // Only lock for the shared write
mu.Unlock()
```

**Protect the data, not the code:**

Think of a mutex as protecting a *piece of data*, not a *section of code*. Every access to the shared data (reads AND writes) must be protected by the same mutex.

### Encapsulating Mutexes in Structs

The idiomatic Go pattern is to embed the mutex alongside the data it protects:

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func NewSafeCounter() *SafeCounter {
    return &SafeCounter{
        v: make(map[string]int),
    }
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.v[key]
}
```

This pattern ensures the mutex and the data it protects are always together. External code never touches the mutex directly -- it uses the safe methods.

### sync.RWMutex -- Read/Write Mutex

A regular Mutex is exclusive: only one goroutine can hold it. But many workloads are read-heavy. If 100 goroutines want to read and only 1 wants to write, a regular Mutex forces all 100 readers to wait for each other, even though concurrent reads are safe.

`sync.RWMutex` solves this:

```
RWMutex Rules:
- Multiple readers can hold RLock() simultaneously
- A writer (Lock()) has exclusive access -- no readers or other writers
- A writer waiting for Lock() blocks new RLock() calls (prevents writer starvation)
```

```go
type SafeCache struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()         // Multiple goroutines can read simultaneously
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()          // Exclusive access for writing
    defer c.mu.Unlock()
    c.items[key] = value
}
```

### When to Use Mutex vs RWMutex

| Use Case | Choice |
|---|---|
| Roughly equal reads and writes | `sync.Mutex` (simpler, lower overhead) |
| Many more reads than writes (10:1+) | `sync.RWMutex` |
| Only writes | `sync.Mutex` |
| Performance-critical, measured bottleneck | Benchmark both and choose |

`RWMutex` has slightly more overhead than `Mutex` (it tracks reader counts). Do not use it unless you have measured that the read contention is actually a bottleneck.

---

## 8. Race Conditions

### What Is a Race Condition?

A **race condition** occurs when:
1. Two or more goroutines access shared memory
2. At least one access is a write
3. The accesses are not synchronized

The result depends on the *timing* of goroutine execution, which is non-deterministic. Your program may work correctly 99% of the time and fail catastrophically the other 1% -- in production, at 3 AM, on the busiest day of the year.

### The `-race` Flag

Go has a built-in race detector. It is one of the most valuable tools in the Go ecosystem:

```bash
# Compile and run with race detection enabled
go run -race main.go

# Test with race detection
go test -race ./...

# Build with race detection (for CI/CD)
go build -race -o myapp
```

The race detector instruments memory accesses at compile time and detects data races at runtime. It has ~2-10x performance overhead, so you typically use it during development and testing, not in production.

### Race Condition Examples

**Example 1: The lost update (counter)**

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // RACE: read-modify-write without synchronization
        }()
    }

    wg.Wait()
    fmt.Println(counter)
}
```

Running with `-race`:
```
==================
WARNING: DATA RACE
Read at 0x00c000014098 by goroutine 7:
  main.main.func1()
      /tmp/main.go:14 +0x6a

Previous write at 0x00c000014098 by goroutine 6:
  main.main.func1()
      /tmp/main.go:14 +0x80

Goroutine 7 (running) created at:
  main.main()
      /tmp/main.go:12 +0x6c

Goroutine 6 (finished) created at:
  main.main()
      /tmp/main.go:12 +0x6c
==================
```

**Fix:** Use a mutex or atomic operations:

```go
// Fix with Mutex
var mu sync.Mutex
counter := 0

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

// Fix with atomic (better for simple counters)
import "sync/atomic"

var counter int64

go func() {
    atomic.AddInt64(&counter, 1)
}()
```

**Example 2: The race on a map**

Go maps are NOT safe for concurrent use. Concurrent reads are fine, but a concurrent read and write (or two concurrent writes) will crash your program:

```go
// CRASH: concurrent map writes
m := make(map[string]int)
var wg sync.WaitGroup

for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        key := fmt.Sprintf("key%d", n%10)
        m[key] = n // FATAL: concurrent map writes
    }(i)
}
wg.Wait()
```

This does not just produce wrong results -- it **panics**:
```
fatal error: concurrent map writes
```

**Fix:** Use a mutex or `sync.Map`:

```go
// Fix with Mutex
var mu sync.Mutex
m := make(map[string]int)

go func(n int) {
    mu.Lock()
    m[key] = n
    mu.Unlock()
}(i)

// Fix with sync.Map (see section 9)
var m sync.Map
go func(n int) {
    m.Store(key, n)
}(i)
```

**Example 3: The race on a slice**

```go
// BUG: concurrent append is not safe
var results []int
var wg sync.WaitGroup

for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        results = append(results, n) // RACE: append reads and writes the slice header
    }(i)
}
wg.Wait()
// results may have fewer than 100 elements, may have corrupted data
```

**Fix:** Use a mutex:

```go
var mu sync.Mutex
var results []int
var wg sync.WaitGroup

for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        mu.Lock()
        results = append(results, n)
        mu.Unlock()
    }(i)
}
wg.Wait()
// results has exactly 100 elements (order is non-deterministic)
```

### Race Condition Checklist

When writing concurrent code, ask yourself:

1. **Is any variable accessed by multiple goroutines?** If yes, proceed to 2.
2. **Is at least one access a write?** If yes, you need synchronization.
3. **What synchronization mechanism?**
   - Simple counter: `sync/atomic`
   - Protecting a data structure: `sync.Mutex` or `sync.RWMutex`
   - Coordinating goroutines: channels (Chapter 11)
   - One-time initialization: `sync.Once`
   - Concurrent map access: `sync.Map` or mutex-protected map

---

## 9. sync.Once and sync.Map

### sync.Once

`sync.Once` guarantees that a function is executed exactly once, regardless of how many goroutines call it. This is perfect for lazy initialization of expensive resources.

```go
package main

import (
    "fmt"
    "sync"
)

type Database struct {
    connection string
}

var (
    db   *Database
    once sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        fmt.Println("Initializing database connection...") // Prints ONCE
        db = &Database{connection: "postgres://localhost/mydb"}
    })
    return db
}

func main() {
    var wg sync.WaitGroup

    // 100 goroutines all try to get the database connection
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            conn := GetDB()
            _ = conn
        }()
    }

    wg.Wait()
    // "Initializing database connection..." is printed exactly ONCE
}
```

**Why not just use a mutex?**

You could, but `sync.Once` is:
1. **Cleaner** -- expresses intent directly ("do this once")
2. **Faster after first call** -- subsequent calls are essentially free (just an atomic read)
3. **Correct** -- handles the double-checked locking pattern correctly (which is notoriously tricky to implement yourself)

**Important behavior:** If the function passed to `Do()` panics, the `Once` is still considered "done." The function will not be retried. In Go 1.21+, `sync.OnceFunc`, `sync.OnceValue`, and `sync.OnceValues` provide more convenient wrappers:

```go
// Go 1.21+
getDB := sync.OnceValue(func() *Database {
    fmt.Println("Initializing...")
    return &Database{connection: "postgres://localhost/mydb"}
})

db1 := getDB() // Initializes
db2 := getDB() // Returns cached value
// db1 == db2 (same pointer)
```

### sync.Map

`sync.Map` is a concurrent-safe map provided by the standard library. Unlike a regular map, it does not require external locking.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var m sync.Map

    // Store key-value pairs (no type parameters -- uses interface{}/any)
    m.Store("name", "Alice")
    m.Store("age", 30)
    m.Store("city", "Portland")

    // Load a value
    if val, ok := m.Load("name"); ok {
        fmt.Println("Name:", val) // Name: Alice
    }

    // LoadOrStore: Load if exists, otherwise store and return the new value
    actual, loaded := m.LoadOrStore("name", "Bob")
    fmt.Println(actual, loaded) // Alice true (already existed)

    actual, loaded = m.LoadOrStore("email", "alice@example.com")
    fmt.Println(actual, loaded) // alice@example.com false (newly stored)

    // Delete a key
    m.Delete("city")

    // Range: iterate over all key-value pairs
    m.Range(func(key, value any) bool {
        fmt.Printf("%s: %v\n", key, value)
        return true // return false to stop iteration
    })

    // LoadAndDelete: atomically load and delete
    val, loaded := m.LoadAndDelete("age")
    fmt.Println("Deleted age:", val, loaded) // Deleted age: 30 true
}
```

### When to Use sync.Map vs Mutex + Regular Map

`sync.Map` is NOT a drop-in replacement for all map + mutex combinations. It is optimized for specific patterns:

| Pattern | Best Choice |
|---|---|
| Keys are mostly stable (written once, read many times) | `sync.Map` |
| High-contention reads, rare writes | `sync.Map` |
| Disjoint key sets (different goroutines access different keys) | `sync.Map` |
| Keys change frequently (many writes) | `Mutex + map` |
| Need type safety | `Mutex + map` (with generics) |
| Known key set at startup | `Mutex + map` |
| Simple logic, few goroutines | `Mutex + map` (simpler to reason about) |

The standard library documentation states:
> The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys.

### Practical Example: Thread-Safe Cache with sync.Map

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Cache struct {
    data sync.Map
}

type CacheEntry struct {
    Value     string
    ExpiresAt time.Time
}

func (c *Cache) Set(key, value string, ttl time.Duration) {
    c.data.Store(key, CacheEntry{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    })
}

func (c *Cache) Get(key string) (string, bool) {
    val, ok := c.data.Load(key)
    if !ok {
        return "", false
    }

    entry := val.(CacheEntry)
    if time.Now().After(entry.ExpiresAt) {
        c.data.Delete(key)
        return "", false
    }

    return entry.Value, true
}

func main() {
    cache := &Cache{}

    var wg sync.WaitGroup

    // Writer goroutines
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", id)
            cache.Set(key, fmt.Sprintf("value-%d", id), 5*time.Second)
        }(i)
    }

    // Reader goroutines
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", id)
            if val, ok := cache.Get(key); ok {
                fmt.Printf("Got %s = %s\n", key, val)
            }
        }(i)
    }

    wg.Wait()
}
```

---

## 10. Deep Comparison: Go Concurrency vs Node.js

This is where things get really interesting. Go and Node.js have **fundamentally different** concurrency models. Neither is universally better -- they excel in different scenarios.

### The Mental Models

```
GO'S CONCURRENCY MODEL
========================

    ┌─────────────────────────────────────────────────┐
    │                 Your Go Program                   │
    │                                                   │
    │   Goroutine 1    Goroutine 2    Goroutine 3      │
    │   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
    │   │ req = ..  │   │ result = │   │ for i := │    │
    │   │ resp, err│   │   db.Quer│   │   0; i < │    │
    │   │  = http. │   │   y(ctx, │   │   n; i++ │    │
    │   │  Get(url)│   │   sql)   │   │   { ... }│    │
    │   │ // blocks│   │ // blocks│   │ // CPU   │    │
    │   │ // THIS  │   │ // THIS  │   │ // bound │    │
    │   │ // gorout│   │ // gorout│   │          │    │
    │   │ // ine   │   │ // ine   │   │          │    │
    │   └──────────┘   └──────────┘   └──────────┘    │
    │        │               │               │          │
    │        ▼               ▼               ▼          │
    │   ┌────────────────────────────────────────┐     │
    │   │         Go Runtime Scheduler            │     │
    │   │  (multiplexes onto OS threads)          │     │
    │   └──┬─────────────┬─────────────┬─────────┘     │
    │      ▼             ▼             ▼                │
    │   OS Thread 1   OS Thread 2   OS Thread 3        │
    └──────┬─────────────┬─────────────┬───────────────┘
           ▼             ▼             ▼
        CPU Core 1    CPU Core 2    CPU Core 3

    EACH goroutine writes SEQUENTIAL, BLOCKING code.
    The runtime handles the async complexity.
    TRUE parallelism: multiple goroutines execute simultaneously.


NODE.JS CONCURRENCY MODEL
===========================

    ┌──────────────────────────────────────────────────┐
    │                Your Node.js Program                │
    │                                                    │
    │   ┌────────────────────────────────────────────┐  │
    │   │              Event Loop                     │  │
    │   │         (single-threaded)                   │  │
    │   │                                             │  │
    │   │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐   │  │
    │   │  │ tick │─▶│ tick │─▶│ tick │─▶│ tick │──▶ │  │
    │   │  └──────┘  └──────┘  └──────┘  └──────┘   │  │
    │   │                                             │  │
    │   │  Each tick: pick a callback, run it to      │  │
    │   │  completion, then pick the next one.        │  │
    │   │  NEVER runs two callbacks simultaneously.   │  │
    │   └────────────────────────────────────────────┘  │
    │              │                                      │
    │              ▼                                      │
    │   ┌──────────────────┐    ┌────────────────────┐  │
    │   │   libuv Thread   │    │  Worker Threads    │  │
    │   │     Pool (4)     │    │  (if using them)   │  │
    │   │  - File I/O      │    │  - CPU-bound work  │  │
    │   │  - DNS lookup    │    │  - Separate V8     │  │
    │   │  - Compression   │    │    isolates        │  │
    │   └──────────────────┘    └────────────────────┘  │
    └──────────────────────────────────────────────────┘
                       │
                       ▼
                  CPU Core 1 (main thread)
                  (other cores mostly idle unless using worker_threads)

    ALL JavaScript runs on ONE thread.
    Async I/O is handled by the OS/libuv thread pool.
    Callbacks/promises resume on the main thread.
```

### Side-by-Side: Same Task, Different Models

**Task: Fetch 5 URLs concurrently**

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

func fetchURL(url string, wg *sync.WaitGroup) {
    defer wg.Done()

    resp, err := http.Get(url) // Blocks THIS goroutine (not the whole program)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("%s: %d bytes\n", url, len(body))
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://github.com",
        "https://google.com",
        "https://stackoverflow.com",
        "https://reddit.com",
    }

    var wg sync.WaitGroup
    start := time.Now()

    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, &wg) // Each URL in its own goroutine
    }

    wg.Wait()
    fmt.Printf("Total: %v\n", time.Since(start))
}
```

**Node.js:**
```javascript
// Node.js equivalent
async function fetchURL(url) {
    const response = await fetch(url);  // Does NOT block the event loop
    const body = await response.text();
    console.log(`${url}: ${body.length} bytes`);
}

async function main() {
    const urls = [
        "https://golang.org",
        "https://github.com",
        "https://google.com",
        "https://stackoverflow.com",
        "https://reddit.com",
    ];

    const start = Date.now();

    // Promise.all runs all fetches concurrently
    await Promise.all(urls.map(url => fetchURL(url)));

    console.log(`Total: ${Date.now() - start}ms`);
}

main();
```

**Key differences:**
- Go: Each goroutine writes *blocking* code (`http.Get` blocks until response arrives). The scheduler handles concurrency.
- Node.js: You write *async* code (`await fetch` yields control back to the event loop). You explicitly manage concurrency with `Promise.all`.
- Both achieve the same result. For I/O-bound work like this, performance is similar.

### Side-by-Side: CPU-Bound Work

This is where the models diverge dramatically.

**Task: Compute SHA-256 hash of a large buffer, 8 times concurrently**

**Go:**
```go
package main

import (
    "crypto/sha256"
    "fmt"
    "sync"
    "time"
)

func hashWork(id int, data []byte, wg *sync.WaitGroup) {
    defer wg.Done()
    // Truly CPU-bound: hash 100MB of data
    for i := 0; i < 100; i++ {
        sha256.Sum256(data)
    }
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    data := make([]byte, 1_000_000) // 1MB buffer
    var wg sync.WaitGroup

    start := time.Now()

    for i := 0; i < 8; i++ {
        wg.Add(1)
        go hashWork(i, data, &wg) // Each goroutine runs on a different core
    }

    wg.Wait()
    fmt.Printf("Total: %v\n", time.Since(start))
    // On an 8-core machine: ~same time as ONE worker (true parallelism!)
}
```

**Node.js (main thread -- BAD for CPU-bound work):**
```javascript
const crypto = require('crypto');

function hashWork(id, data) {
    for (let i = 0; i < 100; i++) {
        crypto.createHash('sha256').update(data).digest();
    }
    console.log(`Worker ${id} done`);
}

async function main() {
    const data = Buffer.alloc(1_000_000);
    const start = Date.now();

    // These run SEQUENTIALLY -- all on the single main thread!
    // Promise.all does NOT help because the work is synchronous/CPU-bound
    const promises = [];
    for (let i = 0; i < 8; i++) {
        promises.push(new Promise(resolve => {
            hashWork(i, data);  // Blocks the event loop!
            resolve();
        }));
    }
    await Promise.all(promises);

    console.log(`Total: ${Date.now() - start}ms`);
    // Takes ~8x longer than Go -- single threaded!
}

main();
```

**Node.js (worker_threads -- correct approach for CPU-bound):**
```javascript
// main.js
const { Worker } = require('worker_threads');

function runWorker(id, data) {
    return new Promise((resolve, reject) => {
        const worker = new Worker('./hash-worker.js', {
            workerData: { id, data }
        });
        worker.on('message', resolve);
        worker.on('error', reject);
    });
}

async function main() {
    const data = Buffer.alloc(1_000_000);
    const start = Date.now();

    // Now truly parallel -- each worker is a separate thread
    const workers = [];
    for (let i = 0; i < 8; i++) {
        workers.push(runWorker(i, data));
    }
    await Promise.all(workers);

    console.log(`Total: ${Date.now() - start}ms`);
}

main();

// hash-worker.js
const { workerData, parentPort } = require('worker_threads');
const crypto = require('crypto');

for (let i = 0; i < 100; i++) {
    crypto.createHash('sha256').update(workerData.data).digest();
}
console.log(`Worker ${workerData.id} done`);
parentPort.postMessage('done');
```

**Analysis:**

| Aspect | Go | Node.js (main thread) | Node.js (worker_threads) |
|---|---|---|---|
| Lines of code | 25 | 20 | 35+ (2 files) |
| True parallelism | Yes (automatic) | No | Yes (manual setup) |
| Communication | Shared memory | N/A | Message passing (serialized) |
| Startup cost per worker | ~nanoseconds | N/A | ~milliseconds (new V8 isolate) |
| Memory per worker | ~2 KB | N/A | ~4-8 MB (V8 isolate) |

### Go Goroutines vs Node.js Event Loop: Comprehensive Comparison

```
FUNDAMENTAL ARCHITECTURE DIFFERENCE
=====================================

GO:
  "Write blocking code. The runtime makes it concurrent."

  fetch(url)         ← Looks blocking, but the goroutine is parked
  process(result)    ← Resumes when data arrives
  save(processed)    ← Looks sequential, reads top-to-bottom

NODE.JS:
  "Write async code. The event loop interleaves execution."

  const result = await fetch(url);     ← Yields to event loop
  const processed = process(result);   ← Resumes when microtask runs
  await save(processed);               ← Yields again
```

| Dimension | Go Goroutines | Node.js Event Loop |
|---|---|---|
| **Threading model** | M:N (many goroutines on few threads) | Single-threaded + thread pool for blocking I/O |
| **CPU utilization** | All cores, automatically | Single core (main); multi-core requires worker_threads |
| **Code style** | Synchronous-looking (blocking) | Async/await (explicitly non-blocking) |
| **Error handling** | `if err != nil` (at each step) | try/catch or `.catch()` |
| **Shared memory** | Yes (requires synchronization) | No (within single thread) / Via SharedArrayBuffer (workers) |
| **Race conditions** | Common (need mutex/channels) | Rare in main thread; possible with async state |
| **Deadlocks** | Possible (mutex + channel misuse) | Impossible in main thread (no blocking locks) |
| **I/O-bound perf** | Excellent | Excellent |
| **CPU-bound perf** | Excellent (true parallelism) | Poor without worker_threads |
| **Memory per task** | ~2 KB | ~closure size (very small for promises) |
| **Max concurrent tasks** | Millions of goroutines | Millions of promises (but single-threaded) |
| **Debugging** | `-race` flag, goroutine dumps | Async stack traces (improved but still tricky) |
| **Ecosystem** | Standard library has strong concurrency support | Third-party libraries handle most concurrency patterns |

### "Don't Communicate by Sharing Memory; Share Memory by Communicating"

This is Go's central concurrency philosophy (covered deeply in Chapter 11 with channels), but let us understand why it matters now.

**Sharing memory (traditional approach -- C, Java, Go with mutexes):**
```go
// Go: Sharing memory (works but error-prone)
var (
    mu      sync.Mutex
    balance int
)

func deposit(amount int) {
    mu.Lock()
    balance += amount  // Shared memory, protected by mutex
    mu.Unlock()
}

func withdraw(amount int) bool {
    mu.Lock()
    defer mu.Unlock()
    if balance >= amount {
        balance -= amount
        return true
    }
    return false
}
```

**Communicating via channels (Go's preferred approach):**
```go
// Go: Sharing memory by communicating (idiomatic)
type BankOp struct {
    Amount int
    OpType string // "deposit" or "withdraw"
    Result chan bool
}

func bankManager(ops chan BankOp) {
    balance := 0 // Only ONE goroutine owns this variable
    for op := range ops {
        switch op.OpType {
        case "deposit":
            balance += op.Amount
            op.Result <- true
        case "withdraw":
            if balance >= op.Amount {
                balance -= op.Amount
                op.Result <- true
            } else {
                op.Result <- false
            }
        }
    }
}
```

In the channel-based approach, only one goroutine (the bank manager) ever touches the `balance` variable. Other goroutines send *messages* requesting operations. No mutex needed. No race conditions possible.

**How Node.js handles this:**

Node.js does not need mutexes for most code because JavaScript runs on a single thread. However, async code can still have logical race conditions:

```javascript
// Node.js: Logical race condition with async code
let balance = 100;

async function withdraw(amount) {
    if (balance >= amount) {       // Check
        await simulateDBWrite();    // Async gap -- other code can run here!
        balance -= amount;          // Act -- balance may have changed!
        return true;
    }
    return false;
}

// Two concurrent withdrawals of 80 from a 100 balance:
// Both check: balance (100) >= 80? Yes!
// Both subtract: balance = 100 - 80 - 80 = -60  BUG!
await Promise.all([withdraw(80), withdraw(80)]);
```

This is a **TOCTOU (Time of Check to Time of Use)** bug. Even without threads, async/await creates interleaving points where other code can run and invalidate your assumptions.

### When Node.js Concurrency Model Is Better

1. **I/O-heavy, low CPU web servers**: Node.js excels at handling thousands of concurrent connections where each request mostly waits on database queries, API calls, or file reads. The event loop is extremely efficient for this pattern with minimal memory overhead.

2. **Rapid prototyping**: The single-threaded model means you rarely think about race conditions, mutexes, or deadlocks. For most web applications, this is a massive productivity boost.

3. **Frontend + Backend code sharing**: JavaScript/TypeScript on both sides with the same async patterns is a significant advantage for full-stack teams.

4. **Rich ecosystem for web**: npm has an enormous collection of async-first libraries optimized for the event-loop model.

### When Go's Concurrency Model Is Better

1. **CPU-bound workloads**: Image processing, cryptography, data transformation, compression, machine learning inference. Go uses all cores automatically; Node.js requires worker_threads with significant boilerplate.

2. **Low-latency systems**: Go's goroutine scheduler is highly optimized with predictable, low-latency context switches. Node.js event loop can be blocked by a single slow synchronous operation.

3. **Complex concurrent systems**: When you need fine-grained control over concurrent workflows -- fan-out/fan-in, pipelines, cancellation, timeouts. Go's goroutines + channels + context are purpose-built for this.

4. **Systems with shared state**: When multiple concurrent operations need to coordinate on shared data structures, Go's explicit synchronization primitives (mutexes, channels) give you precise control. Node.js requires careful async discipline.

5. **High-throughput data processing**: ETL pipelines, log processing, real-time analytics. Go's ability to parallelize across all cores without serialization overhead is a decisive advantage.

### Summary Table

| Workload | Winner | Why |
|---|---|---|
| REST API (I/O-bound) | Tie | Both handle async I/O well |
| WebSocket server (many connections) | Tie | Both handle thousands of connections |
| Image processing pipeline | **Go** | True parallelism on all cores |
| Real-time data pipeline | **Go** | Goroutines + channels = natural pipeline model |
| Simple CRUD web app | **Node.js** | Faster development, no concurrency bugs |
| Microservice with heavy computation | **Go** | Parallel execution without worker_threads |
| CLI tool with concurrent I/O | **Go** | Simpler than managing async in Node.js |
| Full-stack web app (SSR) | **Node.js** | Share code between frontend and backend |

---

## 11. Key Takeaways

1. **Concurrency is structure; parallelism is execution.** You write concurrent programs with goroutines. The Go runtime decides if and how they run in parallel across cores.

2. **Goroutines are cheap.** With ~2 KB stack and user-space scheduling, you can create hundreds of thousands. Do not hesitate to use them.

3. **`main()` does not wait.** When `main()` returns, all goroutines are killed immediately. Always use proper synchronization (`sync.WaitGroup`, channels).

4. **Always call `wg.Add()` before `go func()`.** And always `defer wg.Done()` as the first statement in the goroutine.

5. **Shared state requires synchronization.** Use `sync.Mutex` for general shared data, `sync.RWMutex` when reads dominate writes, `sync/atomic` for simple counters.

6. **Use the `-race` flag.** Run `go test -race ./...` in CI. It catches data races that would be nearly impossible to find through manual review.

7. **Maps are not goroutine-safe.** Concurrent writes to a map will crash your program. Use `sync.Map` or a mutex-protected map.

8. **Go vs Node.js is not about which is better.** It is about which concurrency model fits your workload. I/O-bound web servers: both are great. CPU-bound parallel work: Go wins decisively.

9. **Prefer channels over mutexes** (once you learn them in Chapter 11). Go's philosophy is "share memory by communicating" -- channels make concurrent code easier to reason about and less error-prone.

10. **The Go scheduler is your friend.** You write simple, blocking code. The scheduler turns it into efficient concurrent execution. Trust it.

---

## 12. Practice Exercises

### Exercise 1: Parallel File Processor (Beginner)

Write a program that:
1. Takes a list of 10 filenames
2. Launches a goroutine for each file that reads it and counts the number of lines
3. Uses `sync.WaitGroup` to wait for all goroutines
4. Prints the total line count across all files

**Hints:**
- Use `bufio.Scanner` to count lines
- Remember to handle errors (file not found)
- Use `defer wg.Done()` in each goroutine

### Exercise 2: Safe Counter (Beginner)

Implement a `SafeCounter` struct with methods:
- `Inc()` -- increments by 1
- `Dec()` -- decrements by 1
- `Value() int` -- returns current value

Test it by launching 1000 goroutines that increment and 1000 that decrement. Verify the final value is 0.

**Requirements:**
- Use `sync.Mutex` for synchronization
- Run with `-race` to verify no data races

### Exercise 3: Parallel Map Processing (Intermediate)

You have a `map[string]int` with 1000 entries. Write a program that:
1. Splits the map into N chunks (one per CPU core)
2. Processes each chunk in a goroutine (multiply each value by 2)
3. Collects results into a new map
4. Verifies the result is correct

**Hints:**
- Use `runtime.NumCPU()` to determine chunk count
- Use a mutex to protect the result map, or use separate result maps per goroutine and merge after

### Exercise 4: Rate-Limited URL Fetcher (Intermediate)

Write a program that:
1. Has a list of 50 URLs to fetch
2. Fetches them concurrently, but with a maximum of 5 concurrent requests (rate limiting)
3. Prints the status code and response time for each

**Hints:**
- Use a buffered channel of capacity 5 as a semaphore (Chapter 11 preview)
- Or use a simple approach: launch 5 worker goroutines that pull from a shared work queue

### Exercise 5: The Dining Philosophers (Advanced)

Implement the classic Dining Philosophers problem:
- 5 philosophers sit around a table
- Between each pair is a fork (5 forks total)
- A philosopher needs both adjacent forks to eat
- Implement a deadlock-free solution using mutexes

**Goals:**
- First implement a version that CAN deadlock (all philosophers pick up left fork simultaneously)
- Then fix it (e.g., odd/even ordering, or a waiter goroutine)
- Run with `-race` to verify correctness

### Exercise 6: Go vs Node.js Benchmark (Comparative)

Implement the same task in both Go and Node.js:
- Generate 1,000,000 random integers
- Sort them using a parallel merge sort (split into chunks, sort chunks concurrently, merge)
- Measure and compare execution times

In Go, use goroutines. In Node.js, try both the main thread and worker_threads approaches.

---

> **Next Chapter**: [Chapter 11 - Channels and Select](./11-channels-and-select.md) -- where you will learn Go's primary communication mechanism between goroutines, the `select` statement for multiplexing, and the full power of "share memory by communicating."
