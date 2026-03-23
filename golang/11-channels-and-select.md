# Chapter 11: Channels, Select & Concurrency Patterns

## Table of Contents

1. [What Are Channels?](#1-what-are-channels)
2. [Channel Basics](#2-channel-basics)
3. [Buffered vs Unbuffered Channels](#3-buffered-vs-unbuffered-channels)
4. [Channel Patterns](#4-channel-patterns)
5. [The select Statement](#5-the-select-statement)
6. [Closing Channels](#6-closing-channels)
7. [nil Channels](#7-nil-channels)
8. [Channel Idioms](#8-channel-idioms)
9. [Deadlocks](#9-deadlocks)
10. [Go vs Node.js Concurrency Comparison](#10-go-vs-nodejs-concurrency-comparison)
11. [Key Takeaways](#11-key-takeaways)
12. [Practice Exercises](#12-practice-exercises)

---

## 1. What Are Channels?

### The Philosophy: CSP (Communicating Sequential Processes)

Go's concurrency model is rooted in **CSP**, a formal language for describing interaction patterns in concurrent systems, published by Tony Hoare in 1978. The core idea is simple but profound:

> **"Don't communicate by sharing memory; share memory by communicating."**
> -- Go Proverb

Most languages (C, C++, Java, even Node.js with worker threads) default to **shared memory** concurrency: multiple threads/goroutines access the same memory location, protected by locks (mutexes). This is error-prone, hard to reason about, and the source of entire categories of bugs: race conditions, deadlocks, priority inversion, lock contention.

Go flips this around. Instead of multiple goroutines reaching into the same memory and coordinating with locks, goroutines send **messages** (values) to each other through **channels**. The channel itself handles the synchronization.

### The Pipe Analogy

Think of a channel as a **physical pipe** connecting two rooms (goroutines):

```
  +-----------+                         +-----------+
  | Goroutine |    ==================   | Goroutine |
  |     A     | -->|  channel (pipe) |-->|     B     |
  |  (sender) |    ==================   | (receiver)|
  +-----------+                         +-----------+

  A puts a value INTO one end of the pipe.
  B takes the value OUT of the other end.
  Neither A nor B needs to know about the other's internal state.
```

Key properties of this pipe:
- **Type-safe**: Only one type of value can flow through a given pipe.
- **Blocking**: If the pipe is full, the sender waits. If empty, the receiver waits.
- **Thread-safe**: No mutex needed; the channel handles synchronization internally.
- **First-class**: Channels are values. You can pass them to functions, store them in structs, send them through other channels.

### Why Not Just Use Mutexes?

Mutexes are available in Go (`sync.Mutex`), and they have their place. But channels are preferred when:

| Use Case | Channels | Mutexes |
|---|---|---|
| Passing data between goroutines | Ideal | Awkward |
| Signaling events (done, cancel) | Ideal | Poor fit |
| Orchestrating pipeline stages | Ideal | Very complex |
| Protecting a shared counter | Overkill | Ideal |
| Guarding a shared map/cache | Possible but verbose | Ideal |

**Rule of thumb**: Use channels to **transfer ownership** of data or to **coordinate** goroutines. Use mutexes to **protect** shared state accessed by multiple goroutines.

---

## 2. Channel Basics

### Creating Channels

Channels are created with `make()`. They are reference types (like maps and slices); a zero-value channel is `nil`.

```go
// Create an unbuffered channel of integers
ch := make(chan int)

// Create a buffered channel of strings with capacity 5
ch2 := make(chan string, 5)

// Zero-value channel is nil
var ch3 chan int  // ch3 == nil
```

### Sending and Receiving

The `<-` operator is used for both sending and receiving. The arrow shows the **direction of data flow**.

```go
ch <- 42        // Send the value 42 INTO the channel
value := <-ch   // Receive a value FROM the channel
<-ch            // Receive and discard the value
```

Visual mnemonic:

```
  ch <- 42      "channel receives 42"     (data flows INTO channel)
       ^^
       arrow points LEFT, into ch

  v := <-ch     "v receives from channel"  (data flows OUT of channel)
       ^^
       arrow points LEFT, out of ch into v
```

### Complete Example: Ping-Pong

```go
package main

import "fmt"

func main() {
    ch := make(chan string)

    // Goroutine sends a message
    go func() {
        ch <- "ping"  // Send "ping" into the channel
    }()

    // Main goroutine receives the message
    msg := <-ch  // Blocks until the goroutine sends
    fmt.Println(msg)  // Output: ping
}
```

Execution flow:

```
  main goroutine              anonymous goroutine
  ─────────────              ────────────────────
  1. creates channel
  2. launches goroutine  -->  3. starts running
  4. blocks on <-ch           5. sends "ping" into ch
  6. receives "ping"    <--   7. goroutine exits
  8. prints "ping"
```

### Channel Direction: Restricting Access

You can declare channels as **send-only** or **receive-only** to enforce compile-time safety:

```go
chan T     // bidirectional: can send and receive
chan<- T   // send-only: can only send into this channel
<-chan T   // receive-only: can only receive from this channel
```

Visual mnemonic:

```
  chan<- T     Arrow points INTO chan   = "you can only put things in"  = SEND-ONLY
  <-chan T     Arrow points OUT of chan = "you can only take things out" = RECEIVE-ONLY
```

**Why do directional channels exist?** They serve as **API contracts**. When a function accepts a `chan<- int`, it is promising the caller: "I will only send on this channel, never receive from it." The compiler enforces this, preventing entire classes of bugs.

```go
package main

import "fmt"

// producer can ONLY send into the channel
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // Sender closes the channel
}

// consumer can ONLY receive from the channel
func consumer(ch <-chan int) {
    for val := range ch {
        fmt.Println("Received:", val)
    }
}

func main() {
    ch := make(chan int) // bidirectional

    go producer(ch) // implicitly converts to chan<- int
    consumer(ch)    // implicitly converts to <-chan int
}
```

A bidirectional channel is **implicitly convertible** to a directional one (the conversion narrows access). The reverse is NOT allowed:

```go
var bidir chan int = make(chan int)
var sendOnly chan<- int = bidir    // OK: narrowing
var recvOnly <-chan int = bidir    // OK: narrowing
// bidir = sendOnly                // COMPILE ERROR: cannot widen
```

---

## 3. Buffered vs Unbuffered Channels

This is one of the most important distinctions in Go concurrency.

### Unbuffered Channels (Default)

An unbuffered channel has **zero capacity**. Every send blocks until another goroutine receives, and every receive blocks until another goroutine sends. This creates a **synchronization point** -- a "handshake" between sender and receiver.

```go
ch := make(chan int)  // unbuffered (capacity 0)
```

```
  UNBUFFERED CHANNEL: Sender and receiver must meet at the same time.

  Goroutine A             Channel (capacity=0)          Goroutine B
  ───────────             ──────────────────            ───────────
  ch <- 42  ─── BLOCKS ──────────────────── waits...
                          ┌──────┐
              ── SEND ──> │  42  │ ── RECV ──>          v := <-ch
                          └──────┘
              (A unblocks)                              (B unblocks)

  Both goroutines synchronize: the send and receive happen SIMULTANEOUSLY.
```

**Why is unbuffered the default?** Because synchronization is the safer primitive. An unbuffered channel guarantees that when the sender's `ch <- val` statement completes, the receiver has already received the value. This means:

- The sender knows the value was delivered.
- Operations before the send in goroutine A **happen-before** operations after the receive in goroutine B.
- It is a synchronization primitive, not just a data transfer mechanism.

### Buffered Channels

A buffered channel has a **fixed capacity**. Sends only block when the buffer is full. Receives only block when the buffer is empty.

```go
ch := make(chan int, 3)  // buffered with capacity 3
```

```
  BUFFERED CHANNEL (capacity=3):

  State after ch <- 1; ch <- 2; ch <- 3:

  ┌───────────────────────┐
  │  [ 1 ]  [ 2 ]  [ 3 ] │   <- buffer is FULL
  └───────────────────────┘
  Next send will BLOCK until a receive empties a slot.

  State after <-ch (receives 1):

  ┌───────────────────────┐
  │  [ 2 ]  [ 3 ]  [   ] │   <- one slot free
  └───────────────────────┘
  Next send will succeed immediately.
```

### Blocking Behavior Comparison

```
  ┌─────────────────────────────────────────────────────────────────┐
  │              UNBUFFERED (cap=0)    │   BUFFERED (cap=N)         │
  ├─────────────────────────────────────────────────────────────────┤
  │ Send blocks until...              │                             │
  │   A receiver is ready             │   Buffer has space          │
  │                                   │   (blocks when full)        │
  ├─────────────────────────────────────────────────────────────────┤
  │ Receive blocks until...           │                             │
  │   A sender is ready               │   Buffer has data           │
  │                                   │   (blocks when empty)       │
  ├─────────────────────────────────────────────────────────────────┤
  │ Guarantees synchronization?       │                             │
  │   YES -- sender knows receiver    │   NO -- send may succeed    │
  │   got the value                   │   before any receive        │
  ├─────────────────────────────────────────────────────────────────┤
  │ Use when...                       │                             │
  │   You need a handshake/sync       │   You want to decouple      │
  │   point between goroutines        │   sender/receiver timing    │
  └─────────────────────────────────────────────────────────────────┘
```

### When to Use Each

**Unbuffered channels:**
- Signaling between goroutines (done channels, quit signals)
- When the sender must know the receiver processed the value
- Request-response patterns
- Any time you need a synchronization point

**Buffered channels:**
- When the sender and receiver operate at different speeds (burst absorption)
- Worker pools (job queues)
- Semaphore patterns (limiting concurrency)
- When you want to avoid blocking the sender unnecessarily
- Collecting results from multiple goroutines

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Unbuffered: sender blocks until receiver is ready
    unbuf := make(chan string)
    go func() {
        fmt.Println("Sending on unbuffered...")
        unbuf <- "hello"
        fmt.Println("Sent on unbuffered!") // Only prints after main receives
    }()
    time.Sleep(time.Second)          // Simulate slow receiver
    fmt.Println("Received:", <-unbuf)

    fmt.Println("---")

    // Buffered: sender does NOT block (if buffer has space)
    buf := make(chan string, 1)
    go func() {
        fmt.Println("Sending on buffered...")
        buf <- "world"
        fmt.Println("Sent on buffered!") // Prints IMMEDIATELY (buffer not full)
    }()
    time.Sleep(time.Second)
    fmt.Println("Received:", <-buf)
}
```

Output:

```
Sending on unbuffered...
Received: hello
Sent on unbuffered!
---
Sending on buffered...
Sent on buffered!
Received: world
```

### Checking Channel State

```go
ch := make(chan int, 5)
ch <- 1
ch <- 2

fmt.Println(len(ch))  // 2 -- number of elements currently in the buffer
fmt.Println(cap(ch))  // 5 -- total buffer capacity
```

**Warning**: `len(ch)` on a channel is rarely useful in production code. By the time you act on the result, another goroutine may have changed the channel's state. It is useful only for debugging and testing.

---

## 4. Channel Patterns

These are the bread-and-butter patterns for building concurrent Go programs.

### Pattern 1: Pipeline

A **pipeline** is a series of stages connected by channels, where each stage is a group of goroutines that:
1. Receives values from an **upstream** channel
2. Performs some computation
3. Sends results to a **downstream** channel

```
  ┌──────────┐    ch1    ┌──────────┐    ch2    ┌──────────┐
  │ generate │ ────────> │ square   │ ────────> │ print    │
  │ (stage1) │           │ (stage2) │           │ (stage3) │
  └──────────┘           └──────────┘           └──────────┘
```

```go
package main

import "fmt"

// Stage 1: Generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: Square each number
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: Add 1 to each number
func addOne(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n + 1
        }
        close(out)
    }()
    return out
}

func main() {
    // Compose the pipeline:
    // generate -> square -> addOne -> print
    ch := addOne(square(generate(2, 3, 4)))

    for result := range ch {
        fmt.Println(result)
    }
    // Output:
    // 5   (2*2 + 1)
    // 10  (3*3 + 1)
    // 17  (4*4 + 1)
}
```

**Why this pattern matters**: Each stage runs in its own goroutine. They execute **concurrently** -- stage 2 can process item 1 while stage 1 is generating item 2. This is real parallelism, not just concurrency.

### Pattern 2: Fan-Out

**Fan-out** means starting multiple goroutines to read from the same channel. This distributes work across multiple concurrent processors.

```
                          ┌──────────┐
                     ┌──> │ worker 1 │
                     │    └──────────┘
  ┌──────────┐  jobs │    ┌──────────┐
  │ producer │ ──────┼──> │ worker 2 │
  └──────────┘       │    └──────────┘
                     │    ┌──────────┐
                     └──> │ worker 3 │
                          └──────────┘
  Multiple goroutines read from the SAME channel.
  Go's channel semantics guarantee each value is
  received by EXACTLY ONE worker (no duplicates).
```

### Pattern 3: Fan-In

**Fan-in** means reading from multiple channels into a single channel. This merges results from multiple sources.

```
  ┌──────────┐
  │ worker 1 │ ──┐
  └──────────┘   │
  ┌──────────┐   │  results   ┌──────────┐
  │ worker 2 │ ──┼──────────> │ consumer │
  └──────────┘   │            └──────────┘
  ┌──────────┐   │
  │ worker 3 │ ──┘
  └──────────┘
  Multiple channels merged into ONE.
```

```go
package main

import (
    "fmt"
    "sync"
)

// fanIn merges multiple channels into one
func fanIn(channels ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup

    // Start one goroutine per input channel
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                merged <- val
            }
        }(ch)
    }

    // Close merged channel when all inputs are done
    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

func producer(id int, count int) <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < count; i++ {
            ch <- id*100 + i
        }
        close(ch)
    }()
    return ch
}

func main() {
    // Create three producers
    ch1 := producer(1, 3)  // sends: 100, 101, 102
    ch2 := producer(2, 3)  // sends: 200, 201, 202
    ch3 := producer(3, 3)  // sends: 300, 301, 302

    // Fan-in: merge all into one channel
    for val := range fanIn(ch1, ch2, ch3) {
        fmt.Println(val)
    }
    // Output (order may vary due to concurrency):
    // 100, 200, 300, 101, 201, 301, 102, 202, 302
}
```

### Pattern 4: Worker Pool

The worker pool combines fan-out and fan-in. A fixed number of workers process jobs from a shared channel and send results to a shared results channel.

```
                     ┌──────────────────────────────────┐
                     │          WORKER POOL              │
                     │                                   │
  ┌──────────┐ jobs  │  ┌────────┐           results    │    ┌──────────┐
  │ producer │──────>│  │worker 1│──────────────────────>│───>│ consumer │
  └──────────┘       │  └────────┘                      │    └──────────┘
                     │  ┌────────┐                      │
                     │  │worker 2│──────────────────────>│
                     │  └────────┘                      │
                     │  ┌────────┐                      │
                     │  │worker 3│──────────────────────>│
                     │  └────────┘                      │
                     └──────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID    int
    Input int
}

type Result struct {
    Job    Job
    Output int
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        // Simulate CPU-intensive work
        time.Sleep(100 * time.Millisecond)
        results <- Result{
            Job:    job,
            Output: job.Input * job.Input,
        }
        fmt.Printf("Worker %d processed job %d\n", id, job.ID)
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    // Start workers
    var wg sync.WaitGroup
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Input: j}
    }
    close(jobs) // Signal no more jobs

    // Wait for all workers to finish, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for result := range results {
        fmt.Printf("Job %d: %d^2 = %d\n", result.Job.ID, result.Job.Input, result.Output)
    }
}
```

### Pattern 5: Semaphore (Limiting Concurrency)

A buffered channel can act as a **counting semaphore** to limit how many goroutines can run concurrently.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    const maxConcurrency = 3
    sem := make(chan struct{}, maxConcurrency) // semaphore

    var wg sync.WaitGroup

    for i := 1; i <= 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            sem <- struct{}{}        // Acquire: blocks if 3 goroutines already running
            defer func() { <-sem }() // Release: free a slot when done

            fmt.Printf("Task %d started (concurrent slots used: %d/%d)\n",
                id, len(sem), maxConcurrency)
            time.Sleep(500 * time.Millisecond) // Simulate work
            fmt.Printf("Task %d done\n", id)
        }(i)
    }

    wg.Wait()
}
```

**Why `struct{}`?** An empty struct occupies zero bytes of memory. We only care about the channel's blocking behavior (the "slot"), not the data. This is an idiomatic Go pattern.

---

## 5. The select Statement

### What Is select?

`select` lets a goroutine wait on **multiple channel operations simultaneously**. It is like a `switch` statement, but each case is a channel operation (send or receive).

```go
select {
case val := <-ch1:
    fmt.Println("Received from ch1:", val)
case ch2 <- 42:
    fmt.Println("Sent 42 to ch2")
case val, ok := <-ch3:
    if !ok {
        fmt.Println("ch3 is closed")
    }
}
```

**Why does select exist?** Without it, a goroutine can only wait on ONE channel at a time. In real programs, you often need to:
- Read from a data channel OR a quit/cancel channel
- Write to output OR handle a timeout
- Listen on multiple input sources and react to whichever is ready first

### select Rules

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  SELECT RULES:                                                  │
  │                                                                 │
  │  1. All cases are evaluated simultaneously.                     │
  │  2. If MULTIPLE cases are ready, one is chosen at RANDOM.       │
  │     (not the first one -- this prevents starvation)             │
  │  3. If NO case is ready and there is no default, select BLOCKS. │
  │  4. If NO case is ready and there IS a default, default runs.   │
  │  5. An empty select{} blocks FOREVER.                           │
  └─────────────────────────────────────────────────────────────────┘
```

### Basic Multiplexing

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch1 <- "from channel 1"
    }()

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch2 <- "from channel 2"
    }()

    // Wait for BOTH values, one at a time
    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println(msg)
        case msg := <-ch2:
            fmt.Println(msg)
        }
    }
    // Output:
    // from channel 2   (arrives first -- 100ms)
    // from channel 1   (arrives second -- 200ms)
}
```

### The default Case: Non-Blocking Operations

The `default` case runs immediately if no other case is ready. This makes the select **non-blocking**.

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 1)

    // Non-blocking send
    select {
    case ch <- 42:
        fmt.Println("Sent 42")
    default:
        fmt.Println("Channel is full, skipping")
    }

    // Non-blocking receive
    select {
    case val := <-ch:
        fmt.Println("Received:", val)
    default:
        fmt.Println("No data available")
    }

    // Second receive -- channel is now empty
    select {
    case val := <-ch:
        fmt.Println("Received:", val)
    default:
        fmt.Println("No data available")
    }
    // Output:
    // Sent 42
    // Received: 42
    // No data available
}
```

### Timeout Pattern

One of the most common uses of `select`: timing out a slow operation.

```go
package main

import (
    "fmt"
    "time"
)

func slowOperation() <-chan string {
    ch := make(chan string)
    go func() {
        time.Sleep(3 * time.Second) // Simulate slow work
        ch <- "operation complete"
    }()
    return ch
}

func main() {
    result := slowOperation()

    select {
    case val := <-result:
        fmt.Println("Success:", val)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout: operation took too long")
    }
    // Output: Timeout: operation took too long
}
```

**How `time.After` works**: It returns a `<-chan time.Time` that receives a single value after the specified duration. When the timeout fires, that channel becomes ready, and `select` picks it.

```
  Timeline:
  ─────────────────────────────────────────────────>
  0s          1s              2s              3s
  │           │               │               │
  start       timeout fires   .               slowOperation finishes
              select picks                    (but we already moved on)
              the timeout case
```

### Ticker Pattern: Periodic Work

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ticker := time.NewTicker(500 * time.Millisecond)
    done := make(chan bool)

    go func() {
        time.Sleep(2200 * time.Millisecond)
        done <- true
    }()

    for {
        select {
        case <-done:
            ticker.Stop()
            fmt.Println("Done!")
            return
        case t := <-ticker.C:
            fmt.Println("Tick at", t.Format("15:04:05.000"))
        }
    }
    // Output:
    // Tick at 10:30:00.500
    // Tick at 10:30:01.000
    // Tick at 10:30:01.500
    // Tick at 10:30:02.000
    // Done!
}
```

### for-select Loop (The Workhorse Pattern)

The combination of `for` + `select` is the most common pattern in Go concurrent code. It creates a goroutine that continuously listens on multiple channels:

```go
func worker(done <-chan struct{}, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-done:
            return // Exit goroutine when done signal received
        case job, ok := <-jobs:
            if !ok {
                return // Jobs channel closed
            }
            results <- process(job)
        }
    }
}
```

---

## 6. Closing Channels

### The close() Function

Closing a channel signals that **no more values will ever be sent** on it. Receivers can detect this.

```go
ch := make(chan int)
close(ch)
```

### What Happens After close()

```
  ┌────────────────────────────────────────────────────────────────┐
  │  AFTER close(ch):                                              │
  │                                                                │
  │  Sending:    ch <- val    --> PANIC! (send on closed channel)  │
  │  Receiving:  val := <-ch  --> Returns zero value immediately   │
  │  Closing:    close(ch)    --> PANIC! (close of closed channel) │
  │  Ranging:    for v := range ch --> Loop terminates              │
  └────────────────────────────────────────────────────────────────┘
```

### Detecting a Closed Channel

The two-value receive form tells you whether the channel is still open:

```go
val, ok := <-ch
// ok == true:  val is a real value sent by a goroutine
// ok == false: channel is closed (and val is the zero value of the type)
```

### range Over Channels

`range` on a channel keeps receiving until the channel is closed:

```go
package main

import "fmt"

func main() {
    ch := make(chan int)

    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch) // MUST close, or the range will block forever
    }()

    // range automatically terminates when ch is closed
    for val := range ch {
        fmt.Println(val)
    }
    // Output: 0 1 2 3 4
}
```

### Why Only Senders Should Close

This is a critical rule:

> **Only the sender (producer) should close a channel. Never the receiver.**

Why? Because:

1. **Sending to a closed channel panics.** If a receiver closes the channel while a sender is still active, the sender will panic.
2. **The receiver doesn't know if the sender is done.** Only the sender knows when there are no more values to send.
3. **Multiple receivers can't coordinate closing.** If two receivers both try to close, the second one panics.

```go
// WRONG: Receiver closing the channel
func consumer(ch chan int) {
    val := <-ch
    close(ch) // DANGEROUS: what if producer tries to send more?
}

// RIGHT: Sender closing the channel
func producer(ch chan int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch) // Safe: we know we're done sending
}
```

**What about multiple senders?** If multiple goroutines send on the same channel, none of them should close it individually. Instead, use a `sync.WaitGroup`:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    ch := make(chan int)
    var wg sync.WaitGroup

    // Launch 3 senders
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 3; j++ {
                ch <- id*10 + j
            }
        }(i)
    }

    // Close channel when ALL senders are done
    go func() {
        wg.Wait()
        close(ch)
    }()

    for val := range ch {
        fmt.Println(val)
    }
}
```

### Common Mistakes with close()

```go
// Mistake 1: Closing a channel twice
ch := make(chan int)
close(ch)
close(ch) // PANIC: close of closed channel

// Mistake 2: Sending on a closed channel
ch := make(chan int)
close(ch)
ch <- 42  // PANIC: send on closed channel

// Mistake 3: Forgetting to close, causing receiver to block forever
ch := make(chan int)
go func() {
    ch <- 1
    ch <- 2
    // forgot close(ch)
}()
for val := range ch {  // BLOCKS FOREVER after receiving 1 and 2
    fmt.Println(val)   // This causes a deadlock!
}

// Mistake 4: Not necessary to close channels that will be garbage collected
// Channels don't NEED to be closed. Only close them if a receiver
// needs to know that no more values are coming (e.g., for range).
```

---

## 7. nil Channels

A nil channel is a channel variable that has not been initialized:

```go
var ch chan int  // ch == nil
```

### Behavior of nil Channels

```
  ┌──────────────────────────────────────────────────────────────┐
  │  nil CHANNEL BEHAVIOR:                                       │
  │                                                              │
  │  Sending:    ch <- val    --> BLOCKS FOREVER                 │
  │  Receiving:  val := <-ch  --> BLOCKS FOREVER                 │
  │  Closing:    close(ch)    --> PANIC                          │
  │  In select:  case <-ch:   --> This case is NEVER selected    │
  └──────────────────────────────────────────────────────────────┘
```

### Why Is This Useful? Dynamically Disabling select Cases

This is an advanced but powerful technique. Setting a channel to `nil` inside a `select` loop effectively **disables** that case, because a nil channel never becomes ready.

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int, 2)
    ch2 := make(chan int, 2)

    ch1 <- 1
    ch1 <- 2
    close(ch1)

    ch2 <- 10
    ch2 <- 20
    close(ch2)

    for ch1 != nil || ch2 != nil {
        select {
        case val, ok := <-ch1:
            if !ok {
                fmt.Println("ch1 is closed, disabling it")
                ch1 = nil // Disable this select case!
                continue
            }
            fmt.Println("ch1:", val)
        case val, ok := <-ch2:
            if !ok {
                fmt.Println("ch2 is closed, disabling it")
                ch2 = nil // Disable this select case!
                continue
            }
            fmt.Println("ch2:", val)
        }
    }
    fmt.Println("Both channels drained")
}
```

Output:
```
ch1: 1
ch2: 10
ch1: 2
ch2: 20
ch1 is closed, disabling it
ch2 is closed, disabling it
Both channels drained
```

**Why this matters**: Without nil channels, once a channel is closed, `<-ch` would return the zero value endlessly (in the `select` case), causing a busy loop. By setting the variable to `nil`, you remove it from contention entirely.

```
  Timeline of select behavior:

  Iteration 1:  select watches [ch1, ch2]   --> picks ch1 or ch2
  Iteration 2:  select watches [ch1, ch2]   --> picks ch1 or ch2
  ...
  ch1 closed:   ch1 = nil
                 select watches [nil, ch2]   --> ONLY ch2 can fire
  ch2 closed:   ch2 = nil
                 select watches [nil, nil]   --> for loop exits
```

---

## 8. Channel Idioms

### Idiom 1: Done Channel

A `done` channel is used to signal goroutines to stop. It is typically an unbuffered `chan struct{}` that gets closed.

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, done <-chan struct{}) {
    for {
        select {
        case <-done:
            fmt.Printf("Worker %d: shutting down\n", id)
            return
        default:
            fmt.Printf("Worker %d: working...\n", id)
            time.Sleep(200 * time.Millisecond)
        }
    }
}

func main() {
    done := make(chan struct{})

    for i := 1; i <= 3; i++ {
        go worker(i, done)
    }

    time.Sleep(1 * time.Second)
    fmt.Println("Main: signaling all workers to stop")
    close(done) // Closing broadcasts to ALL receivers

    time.Sleep(500 * time.Millisecond) // Give workers time to shut down
    fmt.Println("Main: exiting")
}
```

**Why close() and not sending?** `close(done)` broadcasts to ALL goroutines waiting on `done`. Sending `done <- struct{}{}` would only unblock ONE goroutine. This is the Go pattern for "cancellation".

```
  close(done):

  ┌──────────┐
  │ Worker 1 │ <── receives zero value (unblocks)
  └──────────┘
  ┌──────────┐
  │ Worker 2 │ <── receives zero value (unblocks)
  └──────────┘     close() is like a BROADCAST signal
  ┌──────────┐
  │ Worker 3 │ <── receives zero value (unblocks)
  └──────────┘

  vs.

  done <- struct{}{}:

  ┌──────────┐
  │ Worker 1 │ <── receives the value (only ONE worker unblocks)
  └──────────┘
  ┌──────────┐
  │ Worker 2 │     (still blocking)
  └──────────┘
  ┌──────────┐
  │ Worker 3 │     (still blocking)
  └──────────┘
```

### Idiom 2: Or-Channel (First One Wins)

Merge multiple "done" channels into one that fires when ANY of them fires. Useful when you have multiple cancellation sources.

```go
package main

import (
    "fmt"
    "time"
)

// or merges multiple done channels into one.
// The returned channel closes when ANY input channel closes.
func or(channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan struct{})
    go func() {
        defer close(orDone)

        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            // Recursively split channels into halves
            mid := len(channels) / 2
            select {
            case <-or(channels[:mid]...):
            case <-or(channels[mid:]...):
            }
        }
    }()
    return orDone
}

func after(d time.Duration) <-chan struct{} {
    ch := make(chan struct{})
    go func() {
        time.Sleep(d)
        close(ch)
    }()
    return ch
}

func main() {
    start := time.Now()

    // Whichever duration fires first will trigger the or-channel
    <-or(
        after(2*time.Hour),
        after(5*time.Minute),
        after(1*time.Second),   // <-- this fires first
        after(1*time.Hour),
        after(1*time.Minute),
    )

    fmt.Printf("Done after %v\n", time.Since(start))
    // Output: Done after ~1s
}
```

### Idiom 3: Tee-Channel

Like the Unix `tee` command: split one channel into two channels that both receive every value.

```go
package main

import "fmt"

func tee(done <-chan struct{}, in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)

    go func() {
        defer close(out1)
        defer close(out2)

        for val := range in {
            // Use local variables to avoid blocking issues
            o1, o2 := out1, out2
            for i := 0; i < 2; i++ {
                select {
                case <-done:
                    return
                case o1 <- val:
                    o1 = nil // Disable after sending once
                case o2 <- val:
                    o2 = nil // Disable after sending once
                }
            }
        }
    }()

    return out1, out2
}

func main() {
    done := make(chan struct{})
    defer close(done)

    // Source channel
    source := make(chan int)
    go func() {
        defer close(source)
        for i := 1; i <= 5; i++ {
            source <- i
        }
    }()

    // Split into two
    ch1, ch2 := tee(done, source)

    // Read from both (must read in lockstep)
    for val1 := range ch1 {
        val2 := <-ch2
        fmt.Printf("ch1: %d, ch2: %d\n", val1, val2)
    }
    // Output:
    // ch1: 1, ch2: 1
    // ch1: 2, ch2: 2
    // ...
}
```

### Idiom 4: Bridge-Channel

Flatten a channel of channels into a single channel. Useful when a pipeline stage produces channels instead of values.

```go
package main

import "fmt"

func bridge(done <-chan struct{}, chanStream <-chan <-chan int) <-chan int {
    valStream := make(chan int)

    go func() {
        defer close(valStream)

        for {
            var stream <-chan int
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }

            for val := range stream {
                select {
                case valStream <- val:
                case <-done:
                    return
                }
            }
        }
    }()

    return valStream
}

func main() {
    done := make(chan struct{})
    defer close(done)

    // Create a channel of channels
    chanStream := make(chan (<-chan int))
    go func() {
        defer close(chanStream)

        for i := 0; i < 3; i++ {
            stream := make(chan int, 3)
            for j := 1; j <= 3; j++ {
                stream <- i*10 + j
            }
            close(stream)
            chanStream <- stream
        }
    }()

    // Bridge flattens it into a single stream
    for val := range bridge(done, chanStream) {
        fmt.Print(val, " ")
    }
    fmt.Println()
    // Output: 1 2 3 11 12 13 21 22 23
}
```

---

## 9. Deadlocks

A deadlock occurs when goroutines are all waiting for something that will never happen. The Go runtime can detect when **ALL** goroutines are blocked and will crash with:

```
fatal error: all goroutines are asleep - deadlock!
```

**Important**: The runtime can only detect deadlocks where ALL goroutines are stuck. If even one goroutine is running (e.g., a timer, HTTP server), the runtime will NOT detect the deadlock -- the blocked goroutines will simply hang silently.

### Deadlock Scenario 1: Single Goroutine, Unbuffered Channel

```go
func main() {
    ch := make(chan int)
    ch <- 42       // DEADLOCK! No other goroutine to receive
    fmt.Println(<-ch)
}
```

```
  main goroutine
  ──────────────
  1. creates unbuffered channel
  2. tries to send 42 into ch
  3. BLOCKS (waiting for a receiver)
  4. ...but there IS no receiver (main is blocked)
  5. DEADLOCK! All goroutines asleep.
```

**Fix**: Use a goroutine or a buffered channel.

```go
// Fix 1: Use a goroutine
func main() {
    ch := make(chan int)
    go func() { ch <- 42 }()
    fmt.Println(<-ch)
}

// Fix 2: Use a buffered channel
func main() {
    ch := make(chan int, 1)
    ch <- 42
    fmt.Println(<-ch)
}
```

### Deadlock Scenario 2: Goroutines Waiting on Each Other

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        val := <-ch1  // Waits for ch1
        ch2 <- val    // Then sends to ch2
    }()

    val := <-ch2  // Waits for ch2 (but ch2 needs ch1 first!)
    ch1 <- val    // Would send to ch1, but we're stuck above
}
```

```
  main goroutine              goroutine 2
  ──────────────              ───────────
  blocked on <-ch2            blocked on <-ch1
       │                          │
       └── needs goroutine 2 ─────┘
           to send to ch2         needs main to
                                  send to ch1

  CIRCULAR DEPENDENCY = DEADLOCK
```

**Fix**: Reorder the operations.

```go
func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        val := <-ch1
        ch2 <- val
    }()

    ch1 <- 42           // Send first (unblocks goroutine 2)
    val := <-ch2         // Then receive
    fmt.Println(val)     // 42
}
```

### Deadlock Scenario 3: Forgetting to Close a Channel

```go
func main() {
    ch := make(chan int)

    go func() {
        ch <- 1
        ch <- 2
        ch <- 3
        // Forgot close(ch)!
    }()

    for val := range ch { // range waits for close
        fmt.Println(val)  // Prints 1, 2, 3... then DEADLOCK
    }
}
```

### Deadlock Scenario 4: Sending to a Full Buffered Channel in the Same Goroutine

```go
func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    ch <- 3 // DEADLOCK! Buffer is full, no goroutine to receive
}
```

### How to Prevent Deadlocks

```
  ┌────────────────────────────────────────────────────────────────────┐
  │  DEADLOCK PREVENTION CHECKLIST:                                    │
  │                                                                    │
  │  1. Every send must have a corresponding receive                   │
  │     (and vice versa) in a DIFFERENT goroutine.                     │
  │                                                                    │
  │  2. Always close channels when done sending                        │
  │     (especially if receivers use range).                           │
  │                                                                    │
  │  3. Use buffered channels when sender and receiver                 │
  │     might not be ready at the same time.                           │
  │                                                                    │
  │  4. Use select with a timeout or default case to                   │
  │     avoid indefinite blocking.                                     │
  │                                                                    │
  │  5. Use the race detector: go run -race yourprogram.go             │
  │                                                                    │
  │  6. Avoid circular dependencies between channel operations.        │
  │                                                                    │
  │  7. Design goroutine lifetimes explicitly: know how each           │
  │     goroutine starts and how it will end.                          │
  └────────────────────────────────────────────────────────────────────┘
```

### Debugging Tool: go run -race

The Go race detector catches data races (not deadlocks per se, but related concurrent bugs):

```bash
go run -race main.go
```

This instruments the binary to detect concurrent access to shared memory without synchronization. Use it during development and in CI.

---

## 10. Go vs Node.js Concurrency Comparison

### Fundamental Architecture Difference

```
  GO:                                     NODE.JS:
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │     Go Runtime           │            │     V8 Engine            │
  │  ┌─────┐ ┌─────┐        │            │  ┌──────────────────┐    │
  │  │ G1  │ │ G2  │  ...   │            │  │  Event Loop      │    │
  │  └──┬──┘ └──┬──┘        │            │  │  (single thread) │    │
  │     │       │            │            │  └────────┬─────────┘    │
  │  ┌──┴───────┴──┐        │            │           │              │
  │  │  Scheduler   │        │            │  ┌────────┴─────────┐    │
  │  │  (M:N maps   │        │            │  │  libuv thread    │    │
  │  │   goroutines  │        │            │  │  pool (4 threads)│    │
  │  │   to OS       │        │            │  │  (I/O only)      │    │
  │  │   threads)    │        │            │  └──────────────────┘    │
  │  └──────────────┘        │            │                          │
  │  True parallelism        │            │  Concurrency, not        │
  │  across CPU cores        │            │  parallelism (unless     │
  │                          │            │  using worker_threads)   │
  └──────────────────────────┘            └──────────────────────────┘
```

### Go Channels vs Node.js Streams

Both are "data pipelines," but they serve different purposes:

**Go Channels** -- Synchronization primitives for goroutine communication:

```go
// Go: Channel pipeline
func double(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            out <- v * 2
        }
    }()
    return out
}
```

**Node.js Streams** -- I/O abstractions for processing data in chunks:

```javascript
// Node.js: Stream pipeline
const { Transform } = require('stream');

const doubleStream = new Transform({
    objectMode: true,
    transform(chunk, encoding, callback) {
        this.push(chunk * 2);
        callback();
    }
});

// Usage: readable.pipe(doubleStream).pipe(writable);
```

| Aspect | Go Channels | Node.js Streams |
|--------|-------------|-----------------|
| Primary purpose | Goroutine synchronization | I/O data processing |
| Carries | Any type (type-safe) | Buffers or objects |
| Backpressure | Automatic (blocking) | Manual (highWaterMark, pause/resume) |
| Concurrency | True parallel goroutines | Single-threaded event loop |
| Error handling | Return errors as values | 'error' event or pipeline callback |
| Cancellation | Done channel / context | AbortController / stream.destroy() |

### Go Fan-Out/Fan-In vs Node.js Promise.all/Promise.race

**Go -- Fan-Out/Fan-In:**

```go
package main

import (
    "fmt"
    "sync"
)

func fetchURL(url string) <-chan string {
    ch := make(chan string)
    go func() {
        // Simulate HTTP request
        ch <- fmt.Sprintf("Response from %s", url)
    }()
    return ch
}

func main() {
    urls := []string{"/api/users", "/api/posts", "/api/comments"}

    // Fan-out: launch concurrent fetches
    channels := make([]<-chan string, len(urls))
    for i, url := range urls {
        channels[i] = fetchURL(url)
    }

    // Fan-in: collect all results
    var wg sync.WaitGroup
    results := make(chan string, len(urls))
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan string) {
            defer wg.Done()
            results <- <-c
        }(ch)
    }
    go func() {
        wg.Wait()
        close(results)
    }()

    for result := range results {
        fmt.Println(result)
    }
}
```

**Node.js -- Promise.all (equivalent pattern):**

```javascript
// Node.js: Promise.all (fan-out + fan-in in one call)
async function fetchURL(url) {
    const res = await fetch(url);
    return res.json();
}

async function main() {
    const urls = ['/api/users', '/api/posts', '/api/comments'];

    // Fan-out + fan-in: launch all, wait for all
    const results = await Promise.all(urls.map(fetchURL));

    results.forEach(r => console.log(r));
}
```

| Aspect | Go Fan-Out/Fan-In | Node.js Promise.all |
|--------|-------------------|---------------------|
| Parallelism | True OS-level parallelism | Concurrent I/O, single-threaded JS |
| Cancellation | Done channel or context.Context | AbortController signal |
| Error handling | Explicit (check each result) | Rejects on FIRST error |
| Backpressure | Natural (channel blocking) | None (all promises start immediately) |
| Code verbosity | More code, more control | Less code, less control |

### Go select vs Node.js Promise.race

**Go -- select (wait for first ready channel):**

```go
select {
case result := <-primaryDB:
    fmt.Println("Primary responded:", result)
case result := <-replicaDB:
    fmt.Println("Replica responded:", result)
case <-time.After(5 * time.Second):
    fmt.Println("Both databases timed out")
}
```

**Node.js -- Promise.race (wait for first resolved promise):**

```javascript
const result = await Promise.race([
    queryPrimaryDB(),
    queryReplicaDB(),
    new Promise((_, reject) =>
        setTimeout(() => reject(new Error('timeout')), 5000)
    ),
]);
```

| Aspect | Go select | Node.js Promise.race |
|--------|-----------|----------------------|
| What it waits on | Channel operations | Promises |
| Reusable | Yes (runs in a loop) | One-shot (promise settles once) |
| Random fairness | Yes (random when multiple ready) | No (first settled wins) |
| Non-blocking mode | default case | Not built-in |
| Can send | Yes (case ch <- val) | No (promises are receive-only) |

### Go Worker Pool vs Node.js Worker Threads

**Go -- Worker Pool:**

```go
// (See Pattern 4: Worker Pool above for full implementation)
// Go worker pools are lightweight -- you can spin up thousands of goroutines.
// Each goroutine is ~2-8 KB of stack space.
```

**Node.js -- Worker Thread Pool (using piscina):**

```javascript
// Node.js: worker thread pool with piscina
const Piscina = require('piscina');

const pool = new Piscina({
    filename: './worker.js',  // Each worker is a full V8 isolate
    maxThreads: 4,            // OS threads are expensive (~1MB each)
});

async function main() {
    const results = await Promise.all(
        Array.from({ length: 10 }, (_, i) => pool.run(i))
    );
    console.log(results);
}
```

| Aspect | Go Worker Pool | Node.js Worker Threads |
|--------|----------------|------------------------|
| Cost per worker | ~2-8 KB (goroutine) | ~1 MB (OS thread + V8 isolate) |
| Typical pool size | Hundreds to thousands | 4-16 (CPU cores) |
| Communication | Channels (zero-copy within process) | postMessage (serialized/transferred) |
| Shared state | Shared memory (with sync) | SharedArrayBuffer (limited) |
| Setup complexity | ~10 lines of Go | Separate worker file + library |

### Go Channel Timeout vs Node.js AbortController

**Go -- Channel-based timeout with context:**

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func slowOperation(ctx context.Context) (string, error) {
    select {
    case <-time.After(5 * time.Second):
        return "done", nil
    case <-ctx.Done():
        return "", ctx.Err() // context.DeadlineExceeded
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    result, err := slowOperation(ctx)
    if err != nil {
        fmt.Println("Error:", err) // Error: context deadline exceeded
        return
    }
    fmt.Println("Result:", result)
}
```

**Node.js -- AbortController timeout:**

```javascript
async function slowOperation(signal) {
    return new Promise((resolve, reject) => {
        const timer = setTimeout(() => resolve('done'), 5000);
        signal.addEventListener('abort', () => {
            clearTimeout(timer);
            reject(new DOMException('Aborted', 'AbortError'));
        });
    });
}

async function main() {
    const controller = new AbortController();
    setTimeout(() => controller.abort(), 2000); // 2s timeout

    try {
        const result = await slowOperation(controller.signal);
        console.log('Result:', result);
    } catch (err) {
        console.log('Error:', err.message); // Error: Aborted
    }
}
```

| Aspect | Go context.WithTimeout | Node.js AbortController |
|--------|------------------------|-------------------------|
| Propagation | Passed through function call chain | Signal passed through function chain |
| Hierarchy | Child contexts inherit parent deadlines | Manual (no built-in hierarchy) |
| Cleanup | defer cancel() | clearTimeout, removeEventListener |
| Integration | Standard library (net/http, database/sql) | Fetch API, some stream APIs |
| Cancellation reason | ctx.Err() returns typed error | signal.reason (newer API) |

### Go Pipeline vs Node.js stream.pipe()

**Go -- Channel pipeline:**

```go
// Generate -> Filter -> Transform -> Collect
source := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
evens := filter(source, func(n int) bool { return n%2 == 0 })
doubled := transform(evens, func(n int) int { return n * 2 })

for val := range doubled {
    fmt.Println(val) // 4, 8, 12, 16, 20
}
```

**Node.js -- stream.pipe():**

```javascript
const { pipeline } = require('stream/promises');

await pipeline(
    createReadStream('input.txt'),
    new Transform({ transform(chunk, enc, cb) { /* filter */ } }),
    new Transform({ transform(chunk, enc, cb) { /* transform */ } }),
    createWriteStream('output.txt')
);
```

| Aspect | Go Channel Pipeline | Node.js stream.pipe() |
|--------|--------------------|-----------------------|
| Stages run in | Separate goroutines (parallel) | Same event loop (concurrent I/O) |
| Backpressure | Automatic (channel blocking) | Built-in but error-prone |
| Error propagation | Must handle per-stage | pipeline() handles it |
| Data type | Any Go type | Buffer or object |
| Memory | Bounded by channel buffer | Bounded by highWaterMark |

---

## 11. Key Takeaways

```
  ┌────────────────────────────────────────────────────────────────────┐
  │  KEY TAKEAWAYS                                                     │
  │                                                                    │
  │  1. CHANNELS are the primary way goroutines communicate.           │
  │     They transfer OWNERSHIP of data, not just access.              │
  │                                                                    │
  │  2. UNBUFFERED channels synchronize. BUFFERED channels decouple.   │
  │     Default to unbuffered unless you have a reason for buffered.   │
  │                                                                    │
  │  3. DIRECTIONAL channels (chan<-, <-chan) are API contracts.        │
  │     Use them in function signatures to prevent misuse.             │
  │                                                                    │
  │  4. SELECT multiplexes channel operations. Combined with for,      │
  │     it is the workhorse of concurrent Go code.                     │
  │                                                                    │
  │  5. Only SENDERS close channels. Receivers detect closure with     │
  │     the two-value receive (val, ok := <-ch) or range.             │
  │                                                                    │
  │  6. NIL channels block forever. Use this to disable select cases.  │
  │                                                                    │
  │  7. DEADLOCKS happen when goroutines wait on each other in a       │
  │     cycle. The runtime detects total deadlocks, but partial        │
  │     deadlocks (goroutine leaks) are silent killers.                │
  │                                                                    │
  │  8. PATTERNS: pipeline, fan-out, fan-in, worker pool, semaphore,   │
  │     done channel, or-channel, tee, bridge. Know them all.          │
  │                                                                    │
  │  9. Use CONTEXT for cancellation and timeouts in production code,  │
  │     not raw done channels (context is the standard library's       │
  │     done channel with superpowers).                                │
  │                                                                    │
  │  10. Run "go run -race" always. The race detector finds bugs       │
  │      that no amount of code review will catch.                     │
  └────────────────────────────────────────────────────────────────────┘
```

---

## 12. Practice Exercises

### Exercise 1: Basic Pipeline (Beginner)

Build a three-stage pipeline:
1. `generate(done, nums ...int)` -- sends each number to a channel
2. `multiply(done, in, factor)` -- multiplies each received number by `factor`
3. `toString(done, in)` -- converts each int to a string like `"Result: 42"`

Connect them and print all output. Use a `done` channel for cancellation. Ensure no goroutine leaks.

### Exercise 2: Parallel Web Scraper (Intermediate)

Simulate fetching 20 URLs with a worker pool of 5 workers:
- Each "fetch" should take a random duration (100-500ms)
- Workers should read from a jobs channel and send results to a results channel
- Implement a timeout: if ANY individual fetch takes longer than 300ms, skip it
- Print a summary: how many succeeded, how many timed out

### Exercise 3: Rate Limiter (Intermediate)

Implement a token bucket rate limiter using channels:
- A buffered channel of capacity N acts as the bucket
- A ticker goroutine refills one token every X milliseconds
- A `request()` function blocks until a token is available
- Test it by firing 10 requests simultaneously with a limit of 3 per second

### Exercise 4: Fan-In with Ordering (Advanced)

Given 5 producer goroutines, each producing values tagged with a sequence number, merge them into a single channel while maintaining **global order** (by sequence number). Hint: use a heap/priority queue on the receiving side.

### Exercise 5: Graceful Shutdown (Advanced)

Build a mini-server with the following components:
- A "listener" goroutine that generates incoming "requests" (simulated)
- A pool of 3 worker goroutines that process requests
- A "logger" goroutine that receives log messages from workers
- On SIGINT (Ctrl+C), perform graceful shutdown:
  1. Stop accepting new requests
  2. Wait for in-flight requests to complete (with a 5-second deadline)
  3. Drain the log channel
  4. Print a shutdown summary

Use `os/signal`, `context.WithTimeout`, `sync.WaitGroup`, and channel patterns from this chapter.

### Exercise 6: Concurrent Merge Sort (Advanced)

Implement merge sort where each recursive split spawns a goroutine and uses channels to return sorted sub-slices. Add a depth limit: beyond depth N, use sequential sort (to avoid spawning millions of goroutines for large inputs). Benchmark the concurrent version vs the sequential version.

---

## Quick Reference Card

```
  OPERATION                     SYNTAX                   NOTES
  ─────────────────────────────────────────────────────────────────
  Create channel                ch := make(chan T)       unbuffered
  Create buffered channel       ch := make(chan T, N)    capacity N
  Send                          ch <- val               blocks if full/unbuffered
  Receive                       val := <-ch             blocks if empty
  Receive + check closed        val, ok := <-ch         ok==false means closed
  Close                         close(ch)               only sender should close
  Range                         for v := range ch       loops until closed
  Length                        len(ch)                 items in buffer
  Capacity                      cap(ch)                 buffer size
  Send-only type                chan<- T                 for function params
  Receive-only type             <-chan T                 for function params
  Select                        select { case ... }     multiplex channels
  Non-blocking select           select { default: }     returns immediately
  Timeout                       case <-time.After(d):   in select
  nil channel in select         ch = nil                disables that case
```
