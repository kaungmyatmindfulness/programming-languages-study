# Chapter 19: Go Runtime Internals (EXPERT)

> **Difficulty**: Expert
> **Prerequisites**: Chapters 1-18 (especially concurrency, memory, and interfaces)
> **Goal**: Understand what happens beneath your Go code -- the scheduler, garbage collector, memory allocator, network poller, and compiler -- and how these internals compare to Node.js/V8.

---

## Table of Contents

1. [The Go Runtime](#1-the-go-runtime)
2. [Goroutine Scheduler Deep Dive (G-M-P Model)](#2-goroutine-scheduler-deep-dive-g-m-p-model)
3. [Goroutine States and Transitions](#3-goroutine-states-and-transitions)
4. [Stack Management](#4-stack-management)
5. [Garbage Collector](#5-garbage-collector)
6. [Memory Allocator](#6-memory-allocator)
7. [Escape Analysis](#7-escape-analysis)
8. [Network Poller (netpoller)](#8-network-poller-netpoller)
9. [Assembly and Compiler](#9-assembly-and-compiler)
10. [Runtime Metrics and Debugging](#10-runtime-metrics-and-debugging)
11. [Go vs Node.js/V8 Architecture Comparison](#11-go-vs-nodejs-v8-architecture-comparison)
12. [Key Takeaways](#12-key-takeaways)
13. [Further Reading](#13-further-reading)

---

## 1. The Go Runtime

### What Is the Go Runtime?

Every Go binary ships with a **runtime** -- a block of Go and assembly code that is compiled directly into your executable. This is fundamentally different from a virtual machine (like the JVM or V8). There is no interpreter, no bytecode, no JIT compiler. Your Go code compiles to native machine code, and the runtime is simply more native code linked alongside it.

The Go runtime provides four critical subsystems:

```
+------------------------------------------------------------------+
|                        YOUR GO PROGRAM                           |
+------------------------------------------------------------------+
|                         GO RUNTIME                               |
|  +-------------+  +----------+  +-----------+  +--------------+  |
|  |  Goroutine  |  |  Garbage |  |  Memory   |  |   Network    |  |
|  |  Scheduler  |  | Collector|  | Allocator |  |   Poller     |  |
|  |  (G-M-P)    |  | (GC)     |  | (mcache/  |  |  (epoll/     |  |
|  |             |  |          |  |  mheap)   |  |   kqueue)    |  |
|  +-------------+  +----------+  +-----------+  +--------------+  |
+------------------------------------------------------------------+
|                    OPERATING SYSTEM (Linux/macOS/Windows)         |
+------------------------------------------------------------------+
```

**The Goroutine Scheduler** multiplexes potentially millions of goroutines onto a small pool of OS threads. This is why goroutines are cheap and OS threads are expensive.

**The Garbage Collector** reclaims memory that is no longer referenced. Go's GC is a concurrent, tri-color, mark-and-sweep collector optimized for low latency (sub-millisecond pauses).

**The Memory Allocator** is a TCMalloc-inspired system that minimizes contention across threads by using per-thread caches and size-class-based allocation.

**The Network Poller** abstracts OS-level async I/O (epoll on Linux, kqueue on macOS) so that goroutines can use blocking-style code while the runtime handles multiplexing underneath.

### Why Go Has a Runtime but Is NOT a VM

| Aspect | Go Runtime | VM (e.g., JVM, V8) |
|--------|-----------|---------------------|
| Compilation | Ahead-of-time to native code | Bytecode + JIT (or interpreted) |
| Binary | Single static binary | Requires runtime environment |
| GC | Embedded in binary | Part of the VM |
| Scheduling | Goroutine scheduler (userspace) | OS threads or green threads |
| Startup | Microseconds | Milliseconds to seconds |
| Memory overhead | ~few MB for runtime | Tens to hundreds of MB |

The key distinction: a VM *interprets or JIT-compiles* code at execution time. Go's runtime is just a library of supporting functions compiled alongside your code. When you call `go func()`, you are calling a function (`runtime.newproc`) that is native machine code, not a VM instruction.

### Go Runtime vs V8 Runtime

```
Go Binary:                          Node.js Process:
+------------------+                +------------------+
| Your Go Code     |  native code   | Your JS Code     |  source text
| (compiled)       |                | (interpreted/    |
+------------------+                |  JIT compiled)   |
| Go Runtime       |  native code   +------------------+
| - scheduler      |                | V8 Engine        |
| - GC             |                | - Ignition       |  (interpreter)
| - allocator      |                | - TurboFan       |  (JIT compiler)
| - netpoller      |                | - Orinoco GC     |
+------------------+                | - Hidden Classes |
| OS Kernel        |                +------------------+
+------------------+                | libuv            |  (event loop)
                                    +------------------+
                                    | OS Kernel        |
                                    +------------------+
```

V8 must parse JavaScript, build an AST, interpret it (Ignition), profile hot paths, then JIT-compile them (TurboFan). Go does all compilation ahead of time. V8's GC uses generational collection (young/old generations) optimized for short-lived allocations common in dynamic languages. Go's GC uses a non-generational tri-color algorithm optimized for consistent low latency.

---

## 2. Goroutine Scheduler Deep Dive (G-M-P Model)

### The Problem the Scheduler Solves

OS threads are expensive: each one consumes ~1MB of stack memory and requires a kernel context switch (~1-10 microseconds) to schedule. If you create 100,000 OS threads, you need 100GB of stack space alone.

Goroutines solve this with **userspace scheduling**: you can have millions of goroutines multiplexed onto a handful of OS threads. The Go scheduler is the machinery that makes this possible.

### The G-M-P Model

Go's scheduler uses three fundamental entities:

- **G (Goroutine)**: A unit of work. Contains the goroutine's stack, instruction pointer, and other scheduling state. Defined as `runtime.g` in the runtime source.
- **M (Machine)**: An OS thread. Executes Go code. Defined as `runtime.m`. There is no limit by default (can be set with `runtime.SetMaxThreads`).
- **P (Processor)**: A logical processor / execution context. Holds a local run queue of goroutines. The number of Ps equals `GOMAXPROCS` (default: number of CPU cores).

**Critical insight**: A G must be attached to an M to run, and an M must acquire a P to execute Go code. The P is the resource that limits parallelism.

```
          G-M-P Model Overview
          ====================

  GOMAXPROCS = 4 (four logical processors)

  +--------+    +--------+    +--------+    +--------+
  |   P0   |    |   P1   |    |   P2   |    |   P3   |
  | Local  |    | Local  |    | Local  |    | Local  |
  | RunQ:  |    | RunQ:  |    | RunQ:  |    | RunQ:  |
  | [G5]   |    | [G8]   |    | [G11]  |    | (empty)|
  | [G6]   |    | [G9]   |    | [G12]  |    |        |
  | [G7]   |    | [G10]  |    |        |    |        |
  +---+----+    +---+----+    +---+----+    +---+----+
      |             |             |             |
      v             v             v             v
  +--------+    +--------+    +--------+    +--------+
  |   M0   |    |   M1   |    |   M2   |    |   M3   |
  | running|    | running|    | running|    |  idle   |
  |  G1    |    |  G2    |    |  G3    |    |  (no G) |
  +--------+    +--------+    +--------+    +--------+
      |             |             |             |
      v             v             v             v
  [OS Thread]  [OS Thread]  [OS Thread]  [OS Thread]


  Global Run Queue: [G13, G14, G15, ...]
  (overflow / unassigned goroutines)
```

### How Scheduling Works Step by Step

When a goroutine is created with `go func()`:

1. The runtime calls `runtime.newproc`, which creates a new G struct.
2. The new G is placed on the **local run queue** of the current P.
3. If the local queue is full (capacity: 256), half the queue is moved to the **global run queue**.

When the scheduler needs to pick the next goroutine to run (called `schedule()`):

```
func schedule() {
    // 1. Check local run queue of current P
    //    (fast path, no locking required)

    // 2. If empty, check global run queue
    //    (requires lock, grab a batch)

    // 3. If empty, try to STEAL from other P's queues
    //    (work stealing -- pick a random P, steal half its queue)

    // 4. If nothing found, check the network poller
    //    (goroutines waiting on I/O that are now ready)

    // 5. If still nothing, park the M (put OS thread to sleep)
}
```

### Work Stealing

Work stealing is what keeps all processors busy. When a P runs out of goroutines:

```
  P0 (busy)         P1 (busy)         P2 (IDLE!)
  Local RunQ:       Local RunQ:       Local RunQ:
  [G5, G6, G7,      [G12]             (empty)
   G8, G9, G10]                            |
                                           |  "I have nothing to do..."
                                           |
                                           |  1. Check global queue -> empty
                                           |  2. Pick random P... P0!
                                           |  3. Steal HALF of P0's queue
                                           v
  AFTER STEALING:
  P0               P1                P2
  [G5, G6, G7]    [G12]             [G8, G9, G10]  <-- stolen!
```

The stealing algorithm randomizes which P to steal from, preventing pathological behavior where all idle Ps contend on the same victim.

### Scheduling Points (When Does a Goroutine Yield?)

Go uses **cooperative preemption** (with async preemption since Go 1.14). A goroutine yields control at:

| Scheduling Point | How It Works |
|-----------------|--------------|
| Channel operations | Goroutine blocks, moved to channel's wait queue |
| `time.Sleep` | Goroutine parked, timer set |
| System calls | M detaches from P; P handed to another M |
| `runtime.Gosched()` | Explicit yield (rare in practice) |
| Function prologues | Compiler inserts stack-growth checks (cooperative) |
| Async preemption (1.14+) | OS signal sent to long-running goroutine |
| I/O operations | Goroutine parked on netpoller |
| Mutex/sync ops | Goroutine blocks on lock |
| GC | Stop-the-world phases briefly stop all Gs |

**Before Go 1.14**, a tight loop like `for {}` with no function calls would never yield, starving other goroutines. Since Go 1.14, the runtime sends a SIGURG signal to preempt long-running goroutines asynchronously.

### GOMAXPROCS

`GOMAXPROCS` sets the number of Ps (logical processors). It defaults to the number of CPU cores.

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    fmt.Println("CPUs:", runtime.NumCPU())          // e.g., 8
    fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0)) // 0 means "query, don't set"

    // Override: limit to 2 OS threads executing Go code simultaneously
    runtime.GOMAXPROCS(2)

    // Or set via environment variable:
    // GOMAXPROCS=2 go run main.go
}
```

**When to change GOMAXPROCS:**
- Almost never. The default (NumCPU) is correct for most workloads.
- In containers, set it to the number of allocated CPU cores (libraries like `go.uber.org/automaxprocs` do this automatically).
- For latency-sensitive applications, sometimes GOMAXPROCS > NumCPU helps reduce scheduling delays.

### System Call Handling: The "Hand-Off" Mechanism

When a goroutine makes a blocking system call (e.g., file I/O, CGO call), the runtime cannot let it hold the P hostage:

```
  BEFORE syscall:              DURING syscall:

  P0 ---- M0 (running G1)     P0 ---- M2 (new/idle M)
                                        running G5 (from P0's queue)
           G1 calls read()
           (blocking syscall)  M0 (blocked in kernel)
                                \-- G1 (waiting for I/O)

  The P is "handed off" to     M0 is now detached from P0.
  another M so work continues. When the syscall completes,
                               M0 tries to reacquire a P.
```

This hand-off mechanism is why Go can handle thousands of concurrent file operations without thousands of idle Ps.

---

## 3. Goroutine States and Transitions

### Goroutine States

Every goroutine is in one of these states (defined in `runtime/runtime2.go`):

```
  +--------------------------------------------------+
  |              Goroutine State Machine              |
  +--------------------------------------------------+

                    runtime.newproc()
                          |
                          v
                    +-----------+
                    | _Grunnable|  (on a run queue, waiting to be scheduled)
                    +-----+-----+
                          |
                 scheduler picks G
                          |
                          v
                    +-----------+
              +---->| _Grunning |  (actively executing on an M/P)
              |     +-----+-----+
              |           |
              |     +-----+-----+-----+-----+
              |     |           |           |
              |     v           v           v
              | +--------+ +--------+ +---------+
              | |_Gwaiting| |_Gsyscall| |  _Gdead |
              | |(blocked)| |(in      | | (finished|
              | |         | | syscall)| |  or new) |
              | +----+----+ +----+---+ +---------+
              |      |           |
              |      | event     | syscall
              |      | arrives   | returns
              |      v           v
              |  _Grunnable  _Grunnable
              |      |           |
              +------+-----------+
```

### State Details

| State | Description | Example |
|-------|------------|---------|
| `_Gidle` | Just allocated, not yet initialized | Freshly `runtime.newproc`'d before setup |
| `_Grunnable` | Ready to run, on a run queue | Waiting for scheduler to pick it |
| `_Grunning` | Currently executing on an M with a P | Your code is running |
| `_Gsyscall` | Executing a system call, M is blocked | `os.Open()`, CGO call |
| `_Gwaiting` | Blocked, NOT on a run queue | Channel receive, mutex lock, I/O wait |
| `_Gdead` | Finished execution (or unused) | `return` from goroutine function |
| `_Gcopystack` | Stack is being copied (growing) | Stack growth triggered |
| `_Gpreempted` | Stopped due to async preemption | Long-running loop interrupted (Go 1.14+) |

### How the Scheduler Picks Which Goroutine to Run

The scheduling algorithm in `runtime.schedule()` follows this priority:

```
1. Every 61st schedule tick:
   -> Check GLOBAL run queue (prevents starvation)

2. Check LOCAL run queue of current P:
   -> O(1) dequeue from head (fast, no lock)

3. If local queue empty:
   -> Try findrunnable():
      a. Check global queue (grab up to len(globalQ)/GOMAXPROCS + 1)
      b. Check netpoller for ready network I/O
      c. Try work stealing from random other P (steal half)
      d. If all else fails, park the M (go to sleep)
```

The magic number 61 is a prime chosen to reduce lock contention on the global queue while still preventing indefinite starvation.

### Inspecting Goroutine States at Runtime

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // Create some goroutines in various states
    for i := 0; i < 10; i++ {
        go func() {
            time.Sleep(time.Hour) // _Gwaiting
        }()
    }

    for i := 0; i < 5; i++ {
        go func() {
            for { // _Grunning (will be preempted)
                runtime.Gosched()
            }
        }()
    }

    time.Sleep(100 * time.Millisecond)

    fmt.Println("Number of goroutines:", runtime.NumGoroutine())
    // Output: Number of goroutines: 16
    // (10 sleeping + 5 looping + 1 main)

    // For detailed goroutine dumps, use GOTRACEBACK or pprof:
    // GOTRACEBACK=all go run main.go
    // Or: curl http://localhost:6060/debug/pprof/goroutine?debug=2
}
```

---

## 4. Stack Management

### The Problem: Fixed vs Dynamic Stacks

Every goroutine needs a stack for local variables, function call frames, and return addresses. OS threads typically get a fixed 1-8MB stack. With 100,000 goroutines, that would require 100GB-800GB of memory.

Go solves this with **dynamic, growable stacks** that start at just **2KB** (since Go 1.4; it was 4KB before, and 8KB before that).

### History: Segmented Stacks vs Contiguous Stacks

**Segmented Stacks (Go before 1.3)**

```
  Goroutine Stack (segmented):

  +----------+     +----------+     +----------+
  | Segment 1| --> | Segment 2| --> | Segment 3|
  | 4KB      |     | 8KB      |     | 8KB      |
  +----------+     +----------+     +----------+

  Problem: "Hot split" -- if a function call at a segment
  boundary triggers grow/shrink repeatedly:

  call foo() --> need new segment (allocate)
  return     --> don't need it   (free)
  call foo() --> need new segment (allocate)
  return     --> don't need it   (free)
  ... (thousands of times per second = terrible performance)
```

**Contiguous Stacks (Go 1.3+, current)**

```
  Goroutine Stack (contiguous):

  BEFORE GROWTH:               AFTER GROWTH:
  +----------+                 +--------------------+
  | Stack    |   copy all      | Stack              |
  | 8KB      |  --------->     | 16KB               |
  | [data]   |   contents      | [data ... copied]  |
  +----------+   & update      +--------------------+
                 all pointers
```

With contiguous stacks, when the stack needs to grow, the runtime:
1. Allocates a new stack that is **2x the current size**.
2. Copies all data from the old stack to the new stack.
3. Adjusts all pointers (including pointers within stack frames) to point to the new locations.
4. Frees the old stack.

### How Stack Growth Works

The compiler inserts a **stack check** (called a "preamble" or "prologue") at the beginning of most functions:

```
  Function Entry:
  +-----------------------------------------+
  | Compare SP (stack pointer) against       |
  | the goroutine's stack guard (stackguard0)|
  |                                          |
  | if SP < stackguard0 {                   |
  |     call runtime.morestack()            |
  |     // This grows the stack, then       |
  |     // re-executes the function         |
  | }                                       |
  |                                          |
  | // ... rest of the function ...          |
  +-----------------------------------------+
```

This check is extremely cheap (a single comparison) and happens at every function call, which is why it also serves as a **cooperative preemption point** (the scheduler can set `stackguard0` to a special value to force a yield).

### Stack Shrinking

Stacks don't just grow -- they also shrink. During garbage collection, the runtime scans goroutine stacks and **halves** any stack that is less than 1/4 utilized:

```
  Stack utilization check (during GC):

  +--------------------+
  | Stack: 64KB        |
  |                    |
  |   (unused space)   |    < 25% used? Shrink to 32KB
  |                    |
  |                    |
  |   [actual data]    |    ~12KB used
  +--------------------+
          |
          v (copy to smaller stack)
  +----------+
  | Stack:   |
  | 32KB     |
  | [data]   |
  +----------+
```

### Why 2KB Starting Size?

The 2KB starting size is a careful engineering tradeoff:

- **Too small** (e.g., 256 bytes): Almost every goroutine would immediately need to grow, wasting time on copying.
- **Too large** (e.g., 64KB): You could only fit ~15,000 goroutines per GB of RAM.
- **2KB**: Most simple goroutines (channel receive, HTTP handler doing basic work) fit without growing. You can fit ~500,000 goroutines per GB just for stacks.

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    var m runtime.MemStats

    runtime.ReadMemStats(&m)
    before := m.Sys

    n := 100_000
    wg.Add(n)
    for i := 0; i < n; i++ {
        go func() {
            defer wg.Done()
            select {} // block forever (minimal stack usage)
        }()
    }

    runtime.ReadMemStats(&m)
    after := m.Sys

    fmt.Printf("Created %d goroutines\n", n)
    fmt.Printf("Memory increase: ~%d MB\n", (after-before)/(1024*1024))
    // Roughly: 100,000 * 2KB = ~200MB (plus overhead)
}
```

### The Stack Copying Mechanism in Detail

When a stack needs to grow, `runtime.copystack()` performs these steps:

```
  Step 1: Allocate new stack (2x size)

  Old Stack (8KB)         New Stack (16KB)
  +----------+            +--------------------+
  | frame 3  | SP ->      |                    |
  | frame 2  |            |                    |
  | frame 1  |            |                    |
  +----------+            +--------------------+

  Step 2: Copy bytes

  Old Stack (8KB)         New Stack (16KB)
  +----------+            +--------------------+
  | frame 3  |  ------->  | frame 3  (copied)  |
  | frame 2  |  ------->  | frame 2  (copied)  |
  | frame 1  |  ------->  | frame 1  (copied)  |
  +----------+            +--------------------+

  Step 3: Adjust pointers
  - Walk all stack frames
  - For each pointer that points INTO the old stack:
    adjust it by (new_base - old_base) offset
  - Update goroutine's stack bounds
  - This is why you CANNOT take the address of a
    stack variable and store it in a way the runtime
    can't track -- the pointer would become invalid!
    (This is one reason Go has escape analysis.)

  Step 4: Free old stack
```

**Critical implication**: Because stacks can move, you cannot safely pass a pointer to a stack-allocated variable to C code (via CGO) unless you pin it. The runtime must know about all pointers to stack memory.

---

## 5. Garbage Collector

### Overview

Go's garbage collector is a **concurrent, tri-color, mark-and-sweep** collector. Its design prioritizes **low latency** (short pauses) over **throughput** (total time spent collecting). This is the opposite of most JVM collectors, which prioritize throughput.

| Property | Go GC | Typical JVM GC (G1) |
|----------|-------|---------------------|
| Algorithm | Tri-color mark-sweep | Generational, region-based |
| Generational? | No (non-generational) | Yes (young + old gen) |
| Compacting? | No | Yes |
| Concurrent? | Yes (most work) | Mostly (some STW) |
| Typical pause | < 1ms | 5-50ms (G1 target: 200ms) |
| Throughput cost | ~25% CPU during GC | ~5-10% overall |

### Why Non-Generational?

Most GCs use generations based on the "generational hypothesis": most objects die young. Go deliberately chose not to because:

1. **Escape analysis** already keeps most short-lived objects on the stack (free to collect).
2. A generational GC requires **write barriers** on every pointer write (to track old-to-young references). Go's compiler-assisted approach reduces this overhead.
3. Go's allocator uses size classes and per-P caches, making allocation fast without needing a young generation bump-pointer.

The Go team has discussed adding generational aspects in the future, but the current design works well because escape analysis is so effective at keeping temporary objects on the stack.

### Tri-Color Mark-and-Sweep

The GC categorizes every object into one of three colors:

- **White**: Not yet visited. Presumed garbage. (If still white at end of marking, it IS garbage.)
- **Grey**: Visited, but its references have not been fully scanned yet.
- **Black**: Visited, and all its references have been scanned. Definitely reachable.

```
  TRI-COLOR MARKING PHASES
  =========================

  Phase 1: Initial State (all objects WHITE)
  +---+  +---+  +---+  +---+  +---+  +---+
  | A |  | B |  | C |  | D |  | E |  | F |
  | W |  | W |  | W |  | W |  | W |  | W |
  +---+  +---+  +---+  +---+  +---+  +---+
    ^
    |
   root (stack variable, global, etc.)

  Phase 2: Mark roots GREY
  +---+  +---+  +---+  +---+  +---+  +---+
  | A |  | B |  | C |  | D |  | E |  | F |
  | G |  | W |  | W |  | W |  | W |  | W |
  +---+  +---+  +---+  +---+  +---+  +---+

  Phase 3: Scan grey objects, mark their references grey,
           turn scanned object black

  A -> B, A -> C  (A is scanned, becomes BLACK)

  +---+  +---+  +---+  +---+  +---+  +---+
  | A |  | B |  | C |  | D |  | E |  | F |
  | B |  | G |  | G |  | W |  | W |  | W |
  +---+  +---+  +---+  +---+  +---+  +---+

  Phase 4: Continue scanning grey objects

  B -> D  (B becomes BLACK)
  C -> (nothing new)  (C becomes BLACK)

  +---+  +---+  +---+  +---+  +---+  +---+
  | A |  | B |  | C |  | D |  | E |  | F |
  | B |  | B |  | B |  | G |  | W |  | W |
  +---+  +---+  +---+  +---+  +---+  +---+

  Phase 5: D -> (nothing), D becomes BLACK

  +---+  +---+  +---+  +---+  +---+  +---+
  | A |  | B |  | C |  | D |  | E |  | F |
  | B |  | B |  | B |  | B |  | W |  | W |
  +---+  +---+  +---+  +---+  +---+  +---+

  Phase 6: No more grey objects. SWEEP white objects (E, F).
  E and F are garbage!
```

### The GC Cycle in Detail

```
  GC CYCLE TIMELINE
  =================

  Mutator         |===== running =====|   |==== running ====|   |=== running ===|
  (your code)                          STW                   STW

  GC Work                              |== Mark Phase ==|
                                       (concurrent with mutators)

  Time ------>    [...program runs...]  [STW1]  [MARK]  [STW2]  [SWEEP]  [...]

  STW1: "Mark Setup" (~10-30 microseconds)
    - Enable write barrier
    - Scan goroutine stacks for root pointers
    - Very brief pause (all goroutines stopped)

  MARK: Concurrent marking
    - GC workers (dedicated goroutines) trace the object graph
    - Your code runs simultaneously
    - Write barrier tracks pointer updates made by your code
    - Uses ~25% of available CPU (GOMAXPROCS/4 dedicated workers)

  STW2: "Mark Termination" (~10-30 microseconds)
    - Ensure all marking is complete
    - Disable write barrier
    - Very brief pause

  SWEEP: Concurrent sweeping
    - Reclaim white (unreachable) objects
    - Happens lazily as allocations occur
    - No pause required
```

### The Write Barrier

Because the GC runs concurrently with your program, your code might modify pointers while the GC is scanning. The **write barrier** ensures correctness:

```
  Without write barrier (BUG):

  1. GC scans object A (BLACK), sees pointer to B
  2. Your code runs: A.ptr = C  (C is WHITE, not yet scanned)
  3. Your code runs: B.ptr = nil (B no longer points to C)
  4. GC finishes. C is still WHITE. C is collected.
  5. A.ptr now points to freed memory! CRASH!

  With write barrier:
  When your code writes A.ptr = C, the write barrier
  marks C as GREY, ensuring the GC will scan it.
```

Go uses a **hybrid write barrier** (since Go 1.8) that combines Dijkstra-style (shade the new pointer target) and Yuasa-style (shade the old pointer target) barriers. This eliminates the need to re-scan stacks during GC, reducing pause times.

### GOGC: Controlling GC Frequency

`GOGC` controls how often the GC runs. It sets the **target ratio of new heap allocations to live heap**.

- `GOGC=100` (default): GC triggers when heap has grown 100% since last GC.
- `GOGC=200`: GC triggers at 200% growth (less frequent GC, more memory usage).
- `GOGC=50`: GC triggers at 50% growth (more frequent GC, less memory).
- `GOGC=off`: Disables GC entirely.

```
  GOGC=100 (default):
  Live heap after last GC: 100MB
  Next GC triggers at: 200MB (100MB + 100% of 100MB)

  GOGC=200:
  Live heap after last GC: 100MB
  Next GC triggers at: 300MB (100MB + 200% of 100MB)

  GOGC=50:
  Live heap after last GC: 100MB
  Next GC triggers at: 150MB (100MB + 50% of 100MB)
```

```go
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
)

func main() {
    // Set GOGC programmatically
    old := debug.SetGCPercent(200) // less frequent GC
    fmt.Println("Previous GOGC:", old)

    // Or use the newer SetMemoryLimit (Go 1.19+)
    // This sets a soft memory limit for the entire Go process
    debug.SetMemoryLimit(1 << 30) // 1 GB limit

    // Force a GC cycle
    runtime.GC()

    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("HeapAlloc: %d MB\n", m.HeapAlloc/(1024*1024))
    fmt.Printf("NumGC: %d\n", m.NumGC)
    fmt.Printf("GC Pause Total: %v\n", m.PauseTotalNs)
}
```

### SetMemoryLimit (Go 1.19+)

`debug.SetMemoryLimit` provides a **soft memory limit**. When the Go process approaches this limit, the GC runs more aggressively regardless of GOGC:

```go
// Common pattern: set a high GOGC with a memory limit
// This means: "don't GC often, but definitely GC if we approach 4GB"
debug.SetGCPercent(200)
debug.SetMemoryLimit(4 << 30) // 4 GB

// Or via environment variables:
// GOGC=200 GOMEMLIMIT=4GiB go run main.go
```

This is the recommended approach for production systems where you know your memory budget.

### GC Pacing

The GC pacer is the algorithm that decides **when** to start a GC cycle. It aims to:
1. Complete the GC cycle before the heap grows past the GOGC target.
2. Use approximately 25% of CPU for GC work.
3. Minimize stop-the-world pauses.

The pacer uses a **feedback control loop**: if the last GC finished too late (heap overshot the target), the next cycle starts earlier. If it finished with room to spare, the next cycle starts later.

---

## 6. Memory Allocator

### Overview: TCMalloc-Inspired Design

Go's memory allocator is inspired by TCMalloc (Thread-Caching Malloc) from Google. The key insight is: **most allocations can be served from thread-local caches without any locking**.

### Three-Tier Architecture

```
  MEMORY ALLOCATOR HIERARCHY
  ==========================

  Per-P (no locking):
  +--------------------------------------------------+
  |  P0: mcache          P1: mcache       P2: mcache |
  |  +----------+        +----------+     +--------+ |
  |  | size 8B  |->span  | size 8B  |     |  ...   | |
  |  | size 16B |->span  | size 16B |     |        | |
  |  | size 32B |->span  | size 32B |     |        | |
  |  | ...      |        | ...      |     |        | |
  |  | size 32KB|->span  | size 32KB|     |        | |
  |  +----------+        +----------+     +--------+ |
  +--------------------------------------------------+
           |  cache miss        |
           v                    v
  Per-size-class (locking per class):
  +--------------------------------------------------+
  |                    mcentral                       |
  |  +------------+ +------------+ +------------+    |
  |  | size 8B    | | size 16B   | | size 32B   |    |
  |  | free spans | | free spans | | free spans |    |
  |  | list       | | list       | | list       |    |
  |  +------------+ +------------+ +------------+    |
  +--------------------------------------------------+
           |  central miss      |
           v                    v
  Global (locking):
  +--------------------------------------------------+
  |                      mheap                        |
  |  +----------------------------------------------+|
  |  | Free page spans (treap / page allocator)     ||
  |  | Arena management                              ||
  |  | OS memory mapping (mmap/VirtualAlloc)        ||
  |  +----------------------------------------------+|
  +--------------------------------------------------+
           |
           v
  +--------------------------------------------------+
  |              Operating System (mmap)              |
  +--------------------------------------------------+
```

### mcache (Per-P Cache)

Each P has its own `mcache`. Since only one goroutine runs on a P at a time, **no locking is needed** to allocate from the mcache. This is the fast path.

The mcache contains an `mspan` for each **size class**. An mspan is a contiguous run of pages divided into fixed-size slots:

```
  mspan for size class 16B (object size = 16 bytes):

  +----+----+----+----+----+----+----+----+
  | 16B| 16B| 16B| 16B| 16B| 16B| 16B| 16B|
  |used|FREE|used|FREE|used|FREE|FREE|used|
  +----+----+----+----+----+----+----+----+
    ^    ^         ^
    |    |         |
  alloc  next     next   (free list / bitmap tracking)
         free     free
```

### Size Classes

Go defines ~67 size classes from 8 bytes to 32,768 bytes (32KB). Objects larger than 32KB are allocated directly from the mheap.

| Size Class | Object Size | Span Pages | Objects per Span |
|-----------|-------------|------------|-----------------|
| 1 | 8 B | 1 (8KB) | 1024 |
| 2 | 16 B | 1 | 512 |
| 3 | 24 B | 1 | 341 |
| 4 | 32 B | 1 | 256 |
| ... | ... | ... | ... |
| 27 | 512 B | 1 | 16 |
| ... | ... | ... | ... |
| 67 | 32,768 B | 4 | 1 |

When you allocate a 20-byte object, it goes into size class 3 (24 bytes). There is ~4 bytes of internal fragmentation, but this is a worthwhile tradeoff for fast allocation.

### The Tiny Allocator

For objects smaller than 16 bytes that contain **no pointers** (e.g., small strings, small integers used as values), Go uses a special **tiny allocator** that packs multiple tiny objects into a single 16-byte slot:

```
  Tiny allocator combines multiple small allocations:

  Standard allocation:
  allocate(3 bytes) -> 8B slot  (5 bytes wasted)
  allocate(5 bytes) -> 8B slot  (3 bytes wasted)
  allocate(2 bytes) -> 8B slot  (6 bytes wasted)
  Total: 24 bytes used for 10 bytes of data

  Tiny allocator:
  +----------------+
  | 16-byte slot   |
  | [3B][5B][2B]   |  <-- all packed into one slot
  | [6B free]      |
  +----------------+
  Total: 16 bytes used for 10 bytes of data
```

This is particularly effective for allocating small temporary values and reduces GC pressure significantly.

### Large Object Allocation

Objects > 32KB bypass the mcache/mcentral entirely and go straight to the mheap:

```go
// This 64KB allocation goes directly to mheap
bigSlice := make([]byte, 65536)
```

The mheap manages free pages using a **page allocator** (radix tree based since Go 1.14) that can efficiently find contiguous runs of free pages.

### Allocation Flow Summary

```
  new(MyStruct) or make([]byte, n)
           |
           v
  Is size <= 32KB?
   /              \
  YES              NO
   |                |
   v                v
  mcache           mheap (directly)
  (no lock)        (lock required)
   |
   v
  Has free slot in mspan for this size class?
   /              \
  YES              NO
   |                |
   v                v
  Return slot     mcentral
  (FAST!)         (per-class lock)
                    |
                    v
                  Has free mspan?
                   /          \
                  YES          NO
                   |            |
                   v            v
                  Give span   mheap
                  to mcache   (global lock)
                               |
                               v
                             mmap() from OS
```

---

## 7. Escape Analysis

### What Is Escape Analysis?

Escape analysis is a **compile-time** optimization where the Go compiler determines whether a variable can safely live on the stack or must be allocated on the heap.

- **Stack allocation**: Free (deallocated when the function returns). No GC involvement.
- **Heap allocation**: Requires GC to eventually collect it. More expensive.

The compiler's goal: **keep as much as possible on the stack**.

### When Does a Variable "Escape" to the Heap?

A variable escapes when the compiler cannot prove it will die when the function returns:

```go
// Case 1: Returning a pointer (ESCAPES)
func newUser() *User {
    u := User{Name: "Alice"} // u escapes to heap
    return &u                // pointer outlives the function
}

// Case 2: Local variable (STAYS ON STACK)
func processUser() {
    u := User{Name: "Bob"} // u stays on stack
    fmt.Println(u.Name)    // u does not escape... but wait!
    // Actually, fmt.Println takes interface{}, so u DOES escape!
}

// Case 3: Closure capturing a variable (MAY ESCAPE)
func counter() func() int {
    n := 0           // n escapes to heap
    return func() int {
        n++          // closure captures n, outlives counter()
        return n
    }
}

// Case 4: Slice too large for stack (ESCAPES)
func bigSlice() {
    s := make([]byte, 64*1024) // escapes (too large for stack)
    _ = s
}

// Case 5: Interface method call (OFTEN ESCAPES)
func writeData(w io.Writer, data []byte) {
    w.Write(data) // data may escape (compiler can't prove w won't store it)
}
```

### Viewing Escape Analysis Decisions

Use `-gcflags='-m'` to see what the compiler decides:

```bash
$ go build -gcflags='-m' main.go

# Example output:
./main.go:10:6: can inline newUser
./main.go:11:2: moved to heap: u
./main.go:17:6: can inline processUser
./main.go:18:13: u escapes to heap
./main.go:24:2: moved to heap: n
./main.go:32:15: make([]byte, 65536) escapes to heap
```

Use `-gcflags='-m -m'` (double -m) for more verbose output explaining WHY:

```bash
$ go build -gcflags='-m -m' main.go

./main.go:11:2: u escapes to heap:
./main.go:11:2:   flow: ~r0 = &u:
./main.go:11:2:     from &u (address-of) at ./main.go:12:9
./main.go:11:2:     from return &u (return) at ./main.go:12:2
```

### Common Escape Scenarios

| Scenario | Escapes? | Why |
|----------|----------|-----|
| Return pointer to local | Yes | Pointer outlives function |
| Pass to `interface{}` parameter | Yes | Compiler can't track through interface |
| Store in package-level variable | Yes | Outlives any function |
| Closure captures variable | Yes | Variable must live as long as closure |
| `make([]byte, n)` where n is variable | Yes | Compiler can't prove size at compile time |
| `make([]byte, 1024)` (constant, small) | No | Compiler knows it fits on stack |
| Local struct used only in function | No | Dies with the function |
| Passed by value to another function | No | Copied, original stays local |
| Slice append that might grow | Depends | New backing array may escape |

### Performance Implications

```go
package main

import "testing"

type Point struct{ X, Y float64 }

// Stack allocation (fast)
func BenchmarkStack(b *testing.B) {
    for i := 0; i < b.N; i++ {
        p := Point{X: 1.0, Y: 2.0} // stays on stack
        _ = p.X + p.Y
    }
}

// Heap allocation (slower -- GC pressure)
func BenchmarkHeap(b *testing.B) {
    for i := 0; i < b.N; i++ {
        p := &Point{X: 1.0, Y: 2.0} // escapes to heap
        sink = p                       // global var forces escape
    }
}

var sink *Point
```

Typical results:
```
BenchmarkStack-8    1000000000    0.25 ns/op    0 B/op    0 allocs/op
BenchmarkHeap-8      50000000   24.00 ns/op   16 B/op    1 allocs/op
```

Stack allocation is roughly **100x faster** because it requires zero GC involvement.

### Tips to Reduce Heap Escapes

1. **Return values, not pointers**, when the struct is small.
2. **Avoid unnecessary interfaces** in hot paths.
3. **Pre-allocate slices** with known capacity: `make([]T, 0, expectedCap)`.
4. **Use `sync.Pool`** for frequently allocated/freed objects.
5. **Pass structs by value** when they are small (up to ~128 bytes is often faster than a pointer due to cache locality).
6. **Be careful with `fmt.Println`** and similar functions that take `interface{}` -- they cause arguments to escape.

---

## 8. Network Poller (netpoller)

### The Magic Behind Goroutine I/O

One of Go's most powerful illusions: goroutines appear to do **blocking I/O**, but under the hood, the runtime uses **non-blocking I/O with OS-level event notification** (epoll on Linux, kqueue on macOS, IOCP on Windows).

### The Problem netpoller Solves

```
  WITHOUT netpoller (naive approach):
  ====================================

  10,000 goroutines each reading from a TCP connection.
  Each goroutine blocks an OS thread on read().

  Result: 10,000 OS threads blocked in the kernel.
  Memory: 10,000 * 1MB = 10GB just for thread stacks!
  Performance: Kernel scheduling 10,000 threads = terrible.

  WITH netpoller:
  ================

  10,000 goroutines each reading from a TCP connection.
  The runtime uses NON-BLOCKING sockets + epoll/kqueue.

  Result: ~GOMAXPROCS OS threads (e.g., 8).
  Each blocked goroutine is "parked" (2KB stack, no OS thread).
  epoll/kqueue monitors ALL 10,000 file descriptors.
  When data arrives, the goroutine is woken up.
```

### How It Works

```
  NETPOLLER ARCHITECTURE
  ======================

  Your Code:                    Runtime:
  +-----------+
  | conn.Read |  (looks like    1. Set socket to non-blocking
  | (blocking)|   a blocking    2. Try read() -> EAGAIN (no data yet)
  +-----------+  call)          3. Register FD with epoll/kqueue
       |                        4. Park goroutine (G -> _Gwaiting)
       |                        5. M is free to run other goroutines!
       |
       |  ... time passes, data arrives on the socket ...
       |
       |                        6. epoll_wait() returns this FD as ready
       |                        7. Unpark goroutine (G -> _Grunnable)
       |                        8. Goroutine is put back on a run queue
       v
  +-----------+
  | conn.Read |                 9. When scheduled, read() now succeeds
  | returns!  |                 10. Goroutine continues execution
  +-----------+
```

### Integration with the Scheduler

The netpoller is deeply integrated with the scheduler. The `findrunnable()` function (called when a P has no goroutines to run) checks the netpoller:

```
  Scheduler's findrunnable():

  1. Check local run queue     -> empty
  2. Check global run queue    -> empty
  3. Check netpoller:
     -> Call netpoll(non-blocking)
     -> Returns list of goroutines whose I/O is ready
     -> Put them on run queues
     -> Run one of them!
  4. Try work stealing         -> nothing to steal
  5. Block and wait            -> netpoll(blocking) wakes us up
```

### The System Calls Under the Hood

**Linux (epoll)**:
```
epoll_create1()           // Create epoll instance (once at startup)
epoll_ctl(ADD, fd, ...)   // Register a socket FD for monitoring
epoll_wait(...)           // Block until one or more FDs are ready
```

**macOS (kqueue)**:
```
kqueue()                  // Create kqueue instance (once at startup)
kevent(REGISTER, fd, ...) // Register a socket FD for monitoring
kevent(WAIT, ...)         // Block until one or more FDs are ready
```

### Code Example: What Happens Under the Hood

```go
package main

import (
    "fmt"
    "net"
    "runtime"
)

func handleConn(conn net.Conn) {
    buf := make([]byte, 1024)

    // This LOOKS blocking, but internally:
    // 1. runtime sets conn's FD to non-blocking
    // 2. calls read() -> EAGAIN
    // 3. registers FD with netpoller (epoll/kqueue)
    // 4. parks this goroutine
    // 5. when data arrives, goroutine is unparked
    n, err := conn.Read(buf)
    if err != nil {
        return
    }
    fmt.Printf("Read %d bytes\n", n)
}

func main() {
    ln, _ := net.Listen("tcp", ":8080")

    for {
        conn, _ := ln.Accept() // Also uses netpoller internally!

        // Each connection gets its own goroutine.
        // 100,000 connections = 100,000 goroutines
        // but only ~GOMAXPROCS OS threads.
        go handleConn(conn)
    }

    // Check how many OS threads are actually being used:
    fmt.Println("Goroutines:", runtime.NumGoroutine())
    // Could be 100,000+
    // But OS threads: still just a handful
}
```

### Comparison: Go netpoller vs Node.js libuv

```
  Go netpoller:                    Node.js libuv:
  ==============                   ================

  +----------+                     +----------+
  | Goroutine| (per connection)    | Callback | (per event)
  | blocking |                     | function |
  | Read()   |                     +----------+
  +----------+                          |
       |                                v
       v                          +----------+
  +----------+                    | Event     |
  | netpoller|                    | Loop      | (single-threaded)
  | (runtime)|                    | (libuv)   |
  +----------+                    +----------+
       |                                |
       v                                v
  +----------+                    +----------+
  | epoll/   |                    | epoll/   |
  | kqueue   |                    | kqueue   |
  +----------+                    +----------+

  Key difference:
  - Go: Each connection has its own goroutine with
    its own stack. Sequential, readable code.
    Parallelism via GOMAXPROCS.
  - Node: All connections share one thread.
    Callback/async-await style. Single thread
    for JS execution (worker threads for CPU).
```

---

## 9. Assembly and Compiler

### Go's Compiler Pipeline

Go's compiler (`cmd/compile`) transforms source code to machine code through several phases:

```
  GO COMPILER PIPELINE
  ====================

  Source Code (.go files)
       |
       v
  +------------------+
  | 1. PARSING       |  Lexer + Parser -> AST
  |    (syntax/noder)|  (Abstract Syntax Tree)
  +------------------+
       |
       v
  +------------------+
  | 2. TYPE CHECKING |  Types, inference, method resolution
  |    (types2)      |  Uses the new types2 package (Go 1.18+)
  +------------------+
       |
       v
  +------------------+
  | 3. IR BUILD      |  AST -> Intermediate Representation
  |    (noder/irgen) |  Unified IR (Go 1.18+)
  +------------------+
       |
       v
  +------------------+
  | 4. MIDDLE END    |  Escape analysis, inlining, devirtualization
  |    (ir passes)   |  Dead code elimination, closure analysis
  +------------------+
       |
       v
  +------------------+
  | 5. SSA           |  Convert to Static Single Assignment form
  |    (ssa package) |  ~50 optimization passes
  +------------------+
       |
       v
  +------------------+
  | 6. MACHINE CODE  |  Register allocation, instruction selection
  |    (obj writer)  |  Write object files (.o)
  +------------------+
       |
       v
  +------------------+
  | 7. LINKING       |  Combine object files + runtime
  |    (cmd/link)    |  Produce final executable
  +------------------+
       |
       v
  Executable Binary
```

### SSA (Static Single Assignment)

SSA is the heart of Go's optimization engine. In SSA form, every variable is assigned exactly once, which makes optimizations much easier:

```go
// Original Go code:
x := 1
x = x + 2
y := x * 3

// SSA form (each assignment is unique):
x1 = 1
x2 = x1 + 2
y1 = x2 * 3
```

You can visualize the SSA passes with:

```bash
# Generate SSA HTML visualization for a specific function
GOSSAFUNC=main go build main.go
# Opens ssa.html in your browser showing all SSA optimization passes
```

### Key Compiler Optimizations

| Optimization | What It Does |
|-------------|--------------|
| **Inlining** | Replaces function calls with function body (small functions) |
| **Escape analysis** | Determines stack vs heap allocation |
| **Dead code elimination** | Removes unreachable code |
| **Bounds check elimination** | Removes array bounds checks when provably safe |
| **Devirtualization** | Converts interface method calls to direct calls when type is known |
| **Prove pass** | Uses mathematical proofs to eliminate checks |
| **Nil check elimination** | Removes nil checks when pointer is provably non-nil |

### Viewing Compiler Output

```bash
# See all compiler optimizations applied
go build -gcflags='-m -m' main.go

# See inlining decisions
go build -gcflags='-m' main.go | grep 'inlining'

# See bounds check eliminations
go build -gcflags='-d=ssa/check_bce/debug=1' main.go

# Generate assembly output
go build -gcflags='-S' main.go

# Or use go tool compile directly
go tool compile -S main.go

# Disassemble an existing binary
go tool objdump -s 'main\.main' ./mybinary

# See the symbol table
go tool nm ./mybinary
```

### Go Assembly Basics

Go uses a **Plan 9-style assembly** syntax that is architecture-independent (sort of). The assembler translates it to actual machine code. This is NOT AT&T or Intel syntax:

```asm
// Plan 9 style (Go assembly):
// func add(a, b int) int
TEXT ·add(SB), NOSPLIT, $0-24
    MOVQ    a+0(FP), AX      // load first argument
    ADDQ    b+8(FP), AX      // add second argument
    MOVQ    AX, ret+16(FP)   // store return value
    RET
```

Key registers in Go's pseudo-assembly:
- `SB` (Static Base): Global symbols
- `FP` (Frame Pointer): Function arguments
- `SP` (Stack Pointer): Top of current stack frame
- `PC` (Program Counter): Instruction pointer

### Compiler Directives (Pragmas)

```go
//go:noinline     -- prevent inlining of this function
//go:nosplit      -- don't insert stack split check (dangerous!)
//go:noescape     -- compiler hint that args don't escape (used in runtime)
//go:linkname     -- link to a symbol in another package (bypass visibility)
//go:generate     -- run code generation tool (not a compiler directive per se)
```

---

## 10. Runtime Metrics and Debugging

### runtime.NumGoroutine()

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    fmt.Println("Goroutines at start:", runtime.NumGoroutine()) // 1

    for i := 0; i < 1000; i++ {
        go func() {
            time.Sleep(time.Second)
        }()
    }

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Goroutines now:", runtime.NumGoroutine()) // ~1001
}
```

### runtime.MemStats

```go
package main

import (
    "fmt"
    "runtime"
)

func printMemStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("=== Memory Stats ===\n")
    fmt.Printf("Alloc (heap in use):    %d MB\n", m.Alloc/(1024*1024))
    fmt.Printf("TotalAlloc (cumulative):%d MB\n", m.TotalAlloc/(1024*1024))
    fmt.Printf("Sys (OS memory):        %d MB\n", m.Sys/(1024*1024))
    fmt.Printf("HeapObjects:            %d\n", m.HeapObjects)
    fmt.Printf("HeapAlloc:              %d MB\n", m.HeapAlloc/(1024*1024))
    fmt.Printf("HeapIdle:               %d MB\n", m.HeapIdle/(1024*1024))
    fmt.Printf("HeapReleased:           %d MB\n", m.HeapReleased/(1024*1024))
    fmt.Printf("StackInuse:             %d KB\n", m.StackInuse/1024)
    fmt.Printf("NumGC:                  %d\n", m.NumGC)
    fmt.Printf("GCCPUFraction:          %.4f%%\n", m.GCCPUFraction*100)
    fmt.Printf("LastGC (ns since epoch):%d\n", m.LastGC)

    // Pause durations for the most recent GC cycles
    fmt.Printf("Last GC pause:          %d us\n", m.PauseNs[(m.NumGC+255)%256]/1000)
}

func main() {
    // Allocate some memory
    data := make([][]byte, 1000)
    for i := range data {
        data[i] = make([]byte, 10000)
    }

    printMemStats()
    _ = data
}
```

### GODEBUG Environment Variable

`GODEBUG` is your window into the runtime. Set it to get detailed traces:

```bash
# GC tracing: see every GC cycle
GODEBUG=gctrace=1 go run main.go
# Output: gc 1 @0.012s 2%: 0.009+0.23+0.004 ms clock, ...
#         gc <N> @<time>s <cpu%>: <STW1>+<concurrent>+<STW2> ms clock, ...

# Scheduler tracing: see scheduler decisions
GODEBUG=schedtrace=1000 go run main.go
# Output every 1000ms:
# SCHED 1000ms: gomaxprocs=8 idleprocs=6 threads=10 idlethreads=4
#   runqueue=0 [0 0 0 0 0 0 2 0]

# Detailed scheduler trace (very verbose)
GODEBUG=schedtrace=1000,scheddetail=1 go run main.go

# Memory allocator trace
GODEBUG=allocfreetrace=1 go run main.go

# Invalidate cached memory mappings
GODEBUG=madvdontneed=1 go run main.go
```

### Understanding gctrace Output

```
gc 5 @0.234s 3%: 0.011+1.2+0.003 ms clock, 0.089+0.45/1.1/0.001+0.024 ms cpu, 4->5->2 MB, 5 MB goal, 0 MB stacks, 0 MB globals, 8 P

|    |        |    |           |          |          |              |
|    |        |    |           |          |          |              Number of Ps
|    |        |    |           |          |          |
|    |        |    |           |          |          Heap sizes:
|    |        |    |           |          |          4 MB before, 5 MB after, 2 MB live
|    |        |    |           |          |
|    |        |    |           |          CPU times:
|    |        |    |           |          assist + background/idle + dedicated
|    |        |    |           |
|    |        |    |           Wall clock times:
|    |        |    |           STW1 + concurrent mark + STW2
|    |        |    |
|    |        |    Percentage of CPU used by GC
|    |        |
|    |        Time since program start
|    |
|    GC cycle number
|
gc = regular GC, scvg = scavenger (returning memory to OS)
```

### runtime/metrics Package (Go 1.16+)

The newer `runtime/metrics` package provides structured, stable metrics:

```go
package main

import (
    "fmt"
    "runtime/metrics"
)

func main() {
    // List all available metrics
    descs := metrics.All()
    for _, desc := range descs {
        fmt.Printf("%-60s %s\n", desc.Name, desc.Description)
    }

    // Read specific metrics
    samples := []metrics.Sample{
        {Name: "/gc/cycles/total:gc-cycles"},
        {Name: "/gc/heap/allocs:bytes"},
        {Name: "/gc/pauses:seconds"},
        {Name: "/memory/classes/heap/objects:bytes"},
        {Name: "/sched/goroutines:goroutines"},
    }

    metrics.Read(samples)

    for _, s := range samples {
        switch s.Value.Kind() {
        case metrics.KindUint64:
            fmt.Printf("%s: %d\n", s.Name, s.Value.Uint64())
        case metrics.KindFloat64:
            fmt.Printf("%s: %.2f\n", s.Name, s.Value.Float64())
        case metrics.KindFloat64Histogram:
            h := s.Value.Float64Histogram()
            fmt.Printf("%s: %d buckets\n", s.Name, len(h.Buckets))
        }
    }
}
```

### pprof: Production Profiling

```go
package main

import (
    "net/http"
    _ "net/http/pprof" // Register pprof handlers
)

func main() {
    // Start pprof server
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Your application code here...

    select {} // block forever
}
```

```bash
# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Block profile (goroutines blocked on sync primitives)
go tool pprof http://localhost:6060/debug/pprof/block

# Mutex contention profile
go tool pprof http://localhost:6060/debug/pprof/mutex

# Trace (execution tracer, not profiler)
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5
go tool trace trace.out
```

### Execution Tracer

The execution tracer gives you a microsecond-level view of goroutine scheduling:

```go
package main

import (
    "os"
    "runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()

    trace.Start(f)
    defer trace.Stop()

    // Your code here -- all goroutine scheduling,
    // GC events, syscalls, etc. will be recorded
    ch := make(chan int)
    go func() { ch <- 42 }()
    <-ch
}
```

```bash
go run main.go
go tool trace trace.out
# Opens a browser with an interactive timeline showing:
# - Goroutine creation/destruction
# - Goroutine blocking/unblocking
# - GC phases
# - System calls
# - Processor utilization
```

---

## 11. Go vs Node.js/V8 Architecture Comparison

### Scheduler vs Event Loop

```
  GO SCHEDULER                         NODE.JS EVENT LOOP
  ============                         ==================

  Multiple OS threads (M)              Single main thread
  Multiple logical processors (P)      + libuv thread pool (4 default)
  Millions of goroutines (G)           Thousands of callbacks

  +---------+  +---------+            +------------------+
  | P0 + M0 |  | P1 + M1 |           |  Event Loop      |
  | run G1  |  | run G2  |           |  (single thread) |
  +---------+  +---------+           +--------+---------+
  | G3,G4.. |  | G5,G6.. |                    |
  +---------+  +---------+           +--------+---------+
                                     | libuv threadpool |
  True parallelism!                  | (fs, dns, crypto)|
  Each P/M runs goroutines           | 4 threads        |
  simultaneously on CPU cores.       +------------------+

  Concurrency Model:                 Concurrency Model:
  - Preemptive (Go 1.14+)           - Cooperative (run-to-completion)
  - Goroutines can run truly         - JS never runs in parallel*
    in parallel                      - Worker threads are separate
  - Shared memory + channels         - Message passing between workers
  - Race detector available          - No shared memory (mostly)

  * Unless using Worker Threads, which have separate V8 isolates
```

| Aspect | Go | Node.js |
|--------|-----|---------|
| Parallelism | True (multi-core) | Single-threaded JS (+ worker threads) |
| Context switch | ~tens of nanoseconds (userspace) | ~microseconds (event loop tick) |
| Max concurrent units | Millions of goroutines | Thousands of connections (limited by callbacks + memory) |
| Blocking code | Each goroutine can block freely | Blocking main thread freezes everything |
| CPU-bound work | Spread across cores naturally | Blocks event loop (use worker_threads) |
| Memory per unit | ~2KB initial stack | ~few hundred bytes per callback (but shared stack) |

### Garbage Collectors

```
  GO GC                                V8 GC (ORINOCO)
  =====                                ================

  Non-generational                     Generational
  Tri-color mark-and-sweep             Mark-sweep + Mark-compact

  +------------------+                 +----------+----------+
  | Heap             |                 | Young Gen| Old Gen  |
  | (one generation) |                 | (Scavenge| (Mark-   |
  | mark-and-sweep   |                 |  /copy)  |  Compact)|
  +------------------+                 +----------+----------+

  Optimized for:                       Optimized for:
  - Low latency (<1ms pauses)          - Throughput (short-lived objects)
  - Consistent pause times             - Dynamic language allocation patterns
  - Server workloads                   - Generational hypothesis
                                       - Incremental + concurrent marking

  Tuning:                              Tuning:
  - GOGC (heap growth ratio)           - --max-old-space-size
  - GOMEMLIMIT (memory limit)          - --max-semi-space-size
  - runtime/debug.SetGCPercent         - --expose-gc + global.gc()
```

| Feature | Go GC | V8 GC (Orinoco) |
|---------|-------|-----------------|
| Generational | No | Yes (young + old) |
| Compacting | No | Yes (old gen mark-compact) |
| Concurrent marking | Yes | Yes (incremental + concurrent) |
| Typical pause | < 1ms | 1-10ms (improved dramatically with Orinoco) |
| Write barrier | Hybrid (Dijkstra + Yuasa) | Remembered set for old-to-young |
| Fragmentation | Can fragment over time | Compaction eliminates fragmentation |
| Stack objects | Escape analysis (many objects on stack) | V8 decides (hidden classes, inline caching) |

### Stack Management

```
  GO STACKS                            V8/NODE STACKS
  =========                            ===============

  Per goroutine: 2KB initial           Per thread: ~1MB fixed
  Grows dynamically (up to 1GB)        (configurable with --stack-size)

  Goroutine 1: [2KB -> 4KB -> 8KB]    Main thread: [1MB fixed]
  Goroutine 2: [2KB]                   Worker 1:    [1MB fixed]
  Goroutine 3: [2KB -> 4KB]            Worker 2:    [1MB fixed]

  100,000 goroutines:                  100,000 "units of work":
  ~200MB minimum                       Callbacks share one 1MB stack
  (but only for stack memory)          (can't have 100K threads)

  Stack overflow: detected and         Stack overflow: "Maximum call
  causes clean crash                   stack size exceeded" error
```

### Network I/O

```
  GO NETPOLLER                         NODE.JS LIBUV
  ==============                       =============

  Built into runtime                   Separate C library
  Uses epoll/kqueue/IOCP               Uses epoll/kqueue/IOCP
  Transparent to programmer            Transparent to programmer

  Code style:                          Code style:
  +-----------------------+            +--------------------------+
  | data, err := conn.Read|            | socket.on('data', (d) => |
  | // goroutine blocks   |            |   // callback fires     |
  | // (parked internally)|            |   // in event loop      |
  | process(data)         |            |   process(d)            |
  +-----------------------+            | })                       |
                                       +--------------------------+

  OR with async/await:
                                       +---------------------------+
                                       | const data = await        |
                                       |   readAsync(socket)       |
                                       | process(data)             |
                                       +---------------------------+

  Both use the same OS primitives (epoll/kqueue).
  The difference is the programming model:
  - Go: Goroutine per connection (sequential code)
  - Node: Event-driven (callbacks/promises/async-await)
```

### Escape Analysis vs V8 Optimizations

Go and V8 solve the performance problem differently because they face different challenges:

| Optimization | Go | V8 |
|-------------|-----|-----|
| **Goal** | Decide stack vs heap at compile time | Optimize dynamic dispatch at runtime |
| **Escape analysis** | Yes (compile-time, static) | Limited (TurboFan can do some) |
| **Hidden classes** | N/A (types known at compile time) | Maps object shapes to memory layouts |
| **Inline caching** | N/A (direct calls at compile time) | Caches method lookups for hot paths |
| **Inlining** | Yes (small functions, compile-time) | Yes (hot functions, JIT runtime) |
| **Devirtualization** | Yes (interface -> concrete call) | Yes (megamorphic -> monomorphic) |
| **When optimized** | Compile time (once) | Runtime (profile-guided, can deoptimize) |

Go's advantage: All optimizations happen once at compile time, with zero runtime overhead. V8's advantage: Can optimize based on actual runtime behavior (speculative optimization), which can beat static analysis for dynamic code patterns.

### GC Tuning Comparison

```go
// Go: Tune GC frequency and memory limit
import "runtime/debug"

debug.SetGCPercent(100)           // GOGC: trigger GC at 100% heap growth
debug.SetMemoryLimit(2 << 30)    // Soft limit: 2 GB
runtime.GC()                     // Force GC cycle
```

```javascript
// Node.js: Tune V8 heap size
// Start with: node --max-old-space-size=4096 app.js  (4GB old gen)
//             node --max-semi-space-size=64 app.js    (64MB young gen)

// Force GC (must start with --expose-gc):
if (global.gc) {
    global.gc();
}

// Check heap stats:
const v8 = require('v8');
console.log(v8.getHeapStatistics());
// {
//   total_heap_size: 9682944,
//   used_heap_size: 5765696,
//   heap_size_limit: 4294967296,
//   ...
// }
```

---

## 12. Key Takeaways

### The Go Runtime Is Not a VM
Go's runtime is compiled native code linked into your binary. It provides goroutine scheduling, GC, memory allocation, and network polling. There is no interpreter, no JIT, no bytecode. This is why Go programs start instantly and have small binaries compared to JVM or Node.js applications.

### The G-M-P Scheduler Is Elegant
Three simple primitives (Goroutine, Machine, Processor) with work stealing create a scheduler that handles millions of goroutines on a handful of OS threads. The key constraint: `GOMAXPROCS` Ps control parallelism, and work stealing ensures load balance.

### Goroutine Stacks Are Dynamic
Starting at 2KB and growing/shrinking as needed via contiguous stack copying, goroutine stacks are what make it feasible to have millions of goroutines. The stack check in every function prologue is the mechanism for both growth and cooperative preemption.

### The GC Trades Throughput for Latency
Go's GC aims for sub-millisecond pauses, which is critical for servers. It achieves this with concurrent tri-color marking and a hybrid write barrier. You pay for this with ~25% CPU during GC and no compaction (potential fragmentation). Tune with GOGC and GOMEMLIMIT.

### Escape Analysis Is Your Best Friend
Understanding escape analysis is the single most impactful runtime knowledge for writing fast Go. Use `go build -gcflags='-m'` to see what escapes. Return values (not pointers) for small types. Avoid unnecessary interfaces in hot paths. Every heap allocation that becomes a stack allocation is a win.

### The Netpoller Is the Magic Behind Goroutines
Goroutines look like they do blocking I/O, but the runtime uses epoll/kqueue underneath. This is what makes "goroutine per connection" viable for 100K+ concurrent connections. The same OS primitive that Node.js uses (epoll/kqueue via libuv), just with a different programming model.

### Use the Tools
- `GODEBUG=gctrace=1` for GC behavior
- `GODEBUG=schedtrace=1000` for scheduler behavior
- `go build -gcflags='-m'` for escape analysis
- `go tool pprof` for CPU/memory profiling
- `go tool trace` for execution tracing
- `runtime/metrics` for production monitoring

---

## 13. Further Reading

### Official Resources
- [Go Runtime Source Code (runtime package)](https://github.com/golang/go/tree/master/src/runtime) -- The actual implementation of everything discussed in this chapter.
- [Go GC Guide](https://tip.golang.org/doc/gc-guide) -- Official guide to understanding and tuning the GC.
- [Go Memory Model](https://go.dev/ref/mem) -- Formal specification of when reads observe writes across goroutines.

### Deep Dive Articles
- [Go Scheduler: Implementing Language with Lightweight Concurrency](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html) -- William Kennedy's three-part series on the Go scheduler (Ardan Labs).
- [Dmitry Vyukov's Scalable Go Scheduler Design](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit) -- Original design document for the G-M-P scheduler.
- [Getting to Go: The Journey of Go's Garbage Collector](https://go.dev/blog/ismmkeynote) -- Rick Hudson's talk on the evolution of Go's GC.
- [Go Memory Allocator Visual Guide](https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed) -- Step-by-step visual explanation of mcache/mcentral/mheap.
- [Contiguous Stacks Design Document](https://docs.google.com/document/d/1wAaf1rYoM4nAvA_reML1rSuliN3Q9YZE6e4mBVDLOPg/edit) -- Why Go switched from segmented to contiguous stacks.

### Tools and Utilities
- [go.uber.org/automaxprocs](https://github.com/uber-go/automaxprocs) -- Automatically set GOMAXPROCS to match container CPU quota.
- [github.com/google/pprof](https://github.com/google/pprof) -- Enhanced pprof visualization tool.
- [github.com/felixge/fgprof](https://github.com/felixge/fgprof) -- Full goroutine profiler (includes off-CPU time).
- [github.com/DataDog/go-profiler-notes](https://github.com/DataDog/go-profiler-notes) -- Comprehensive notes on Go's profiling capabilities.

### Talks
- "Understanding the Go Runtime" by Jesper Louis Andersen -- Deep dive into scheduler internals.
- "Go GC: Latency Problem Solved" by Rick Hudson (GopherCon 2015) -- The concurrent GC design.
- "Understanding Channels" by Kavya Joshi (GopherCon 2017) -- How channels interact with the scheduler.

---

> **Next Chapter**: Chapter 20 will cover **Performance Optimization and Benchmarking** -- putting this runtime knowledge into practice with profiling, benchmarking, and real-world optimization techniques.
