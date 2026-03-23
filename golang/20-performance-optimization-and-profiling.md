# Chapter 20: Performance Optimization & Profiling

## Difficulty: EXPERT

Performance optimization is the art of making programs faster, leaner, and more efficient -- but only after you have proven, with data, that they need to be. This chapter teaches you to profile Go programs, interpret the results, and apply targeted optimizations. Every claim is backed by measurement. Every trick is accompanied by a benchmark. If you come from Node.js, you will see how Go gives you surgical control that V8 simply cannot offer.

---

## Table of Contents

1. [Performance Mindset](#1-performance-mindset)
2. [pprof -- The Profiler](#2-pprof----the-profiler)
3. [Benchmarking](#3-benchmarking)
4. [Memory Optimization](#4-memory-optimization)
5. [CPU Optimization](#5-cpu-optimization)
6. [Concurrency Optimization](#6-concurrency-optimization)
7. [Compiler Optimizations](#7-compiler-optimizations)
8. [I/O Optimization](#8-io-optimization)
9. [GC Tuning](#9-gc-tuning)
10. [Tracing](#10-tracing)
11. [Real-World Optimization Case Study](#11-real-world-optimization-case-study)
12. [Go vs Node.js Performance Tooling Comparison](#12-go-vs-nodejs-performance-tooling-comparison)
13. [Key Takeaways](#13-key-takeaways)
14. [Practice Exercises](#14-practice-exercises)

---

## 1. Performance Mindset

### Measure First, Optimize Second

The single most important rule in performance engineering:

> "Premature optimization is the root of all evil." -- Donald Knuth

This is not a suggestion. It is a law. Here is why:

1. **You do not know where the bottleneck is.** Humans are terrible at guessing. The function you think is slow is usually not the one consuming the most time.
2. **Optimized code is harder to read.** Every optimization trades clarity for speed. If the optimization does not matter, you paid the maintenance cost for nothing.
3. **Optimizations can backfire.** Clever tricks that helped on one architecture may hurt on another, or may confuse the compiler's own optimizer.

The correct workflow is always:

```
Write correct, clear code
        |
        v
  Measure (benchmark / profile)
        |
        v
  Identify the bottleneck
        |
        v
  Optimize the bottleneck ONLY
        |
        v
  Measure again to confirm improvement
        |
        v
  Repeat if necessary
```

### Go's Performance by Default

Go is already fast. Before you optimize anything, appreciate what the language gives you for free:

- **Compiled to native machine code** -- no interpreter, no JIT warm-up.
- **Static typing** -- the compiler knows exact memory layouts at compile time.
- **Value semantics** -- structs live on the stack by default, avoiding heap allocations.
- **Efficient goroutine scheduler** -- context switches cost ~200ns, not ~1-2us like OS threads.
- **Garbage collector** -- sub-millisecond pause times in Go 1.19+ for most workloads.
- **Zero-cost abstractions** -- interfaces and closures are inlined when the compiler can prove the concrete type.

**Node.js comparison:** V8 is a remarkable JIT compiler, but it cannot match ahead-of-time compiled Go for raw throughput. V8 must speculate on types at runtime, deoptimize when guesses are wrong, and cannot control memory layout. Go starts fast and stays fast.

### When to Optimize

Optimize when:
- A benchmark shows the code is too slow for the requirement.
- Profiling data points to a specific function or allocation pattern.
- The system is hitting resource limits (CPU, memory, network) under production load.

Do NOT optimize when:
- "It feels slow" without measurement.
- "This allocation looks wasteful" without proving it matters.
- "I read that maps are slow" -- they are O(1) amortized and usually fine.

---

## 2. pprof -- The Profiler

`pprof` is Go's built-in profiling tool. It is one of the most powerful profiling systems in any language ecosystem. It can profile CPU usage, memory allocations, goroutine activity, lock contention, and more.

### Enabling pprof in a Running Server

The easiest way to enable profiling is via the `net/http/pprof` package:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    _ "net/http/pprof" // Side-effect import: registers /debug/pprof/ handlers
)

func main() {
    // Your application logic here
    go func() {
        // Heavy computation to demonstrate profiling
        for {
            computeExpensiveStuff()
        }
    }()

    fmt.Println("Server running on :6060")
    fmt.Println("Profile at: http://localhost:6060/debug/pprof/")
    log.Fatal(http.ListenAndServe(":6060", nil))
}

func computeExpensiveStuff() {
    data := make([]byte, 1024*1024) // 1MB allocation
    for i := range data {
        data[i] = byte(i % 256)
    }
}
```

This blank import registers HTTP handlers at `/debug/pprof/`. Available endpoints:

| Endpoint | What It Profiles |
|---|---|
| `/debug/pprof/profile` | CPU profile (30s default) |
| `/debug/pprof/heap` | Heap memory allocations |
| `/debug/pprof/goroutine` | All goroutine stack traces |
| `/debug/pprof/block` | Goroutine blocking events |
| `/debug/pprof/mutex` | Mutex contention |
| `/debug/pprof/threadcreate` | OS thread creation |
| `/debug/pprof/allocs` | Past memory allocations (even freed) |
| `/debug/pprof/trace` | Execution trace (for `go tool trace`) |

> **Security warning:** Never expose `/debug/pprof/` in production without authentication. It reveals internal details and can slow your service.

### Programmatic Profiling (No HTTP Server)

For CLI tools or tests, write profiles to files:

```go
package main

import (
    "os"
    "runtime"
    "runtime/pprof"
    "log"
)

func main() {
    // ---- CPU Profile ----
    cpuFile, err := os.Create("cpu.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer cpuFile.Close()

    if err := pprof.StartCPUProfile(cpuFile); err != nil {
        log.Fatal(err)
    }
    defer pprof.StopCPUProfile()

    // ---- Run your code ----
    doWork()

    // ---- Memory Profile ----
    memFile, err := os.Create("mem.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer memFile.Close()

    runtime.GC() // Force GC to get accurate heap stats
    if err := pprof.WriteHeapProfile(memFile); err != nil {
        log.Fatal(err)
    }
}

func doWork() {
    // ... your application logic ...
    result := make([]string, 0)
    for i := 0; i < 1_000_000; i++ {
        result = append(result, fmt.Sprintf("item-%d", i))
    }
    _ = result
}
```

### Using go tool pprof

#### CPU Profiling

```bash
# Capture a 30-second CPU profile from a running server
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Or analyze a saved profile file
go tool pprof cpu.prof
```

Once inside the interactive `pprof` shell:

```
(pprof) top 10
Showing nodes accounting for 4.82s, 96.40% of 5s total
Dropped 12 nodes (cum <= 0.03s)
Showing top 10 nodes out of 23
      flat  flat%   sum%        cum   cum%
     2.10s 42.00% 42.00%      2.10s 42.00%  main.computeExpensiveStuff
     1.35s 27.00% 69.00%      1.35s 27.00%  runtime.memclrNoHeapPointers
     0.55s 11.00% 80.00%      0.55s 11.00%  runtime.mallocgc
     0.32s  6.40% 86.40%      0.32s  6.40%  runtime.gcDrainMarkWorkerDedicated
     0.20s  4.00% 90.40%      4.00s 80.00%  main.main.func1
     0.15s  3.00% 93.40%      0.15s  3.00%  runtime.scanobject
     0.08s  1.60% 95.00%      0.08s  1.60%  runtime.markBits.isMarked
     0.04s  0.80% 95.80%      0.04s  0.80%  runtime.pageIndexOf
     0.02s  0.40% 96.20%      0.02s  0.40%  runtime.memmove
     0.01s  0.20% 96.40%      0.01s  0.20%  runtime.nextFreeFast
```

Key columns:
- **flat**: Time spent IN this function (not counting calls to other functions).
- **cum**: Time spent in this function AND everything it calls (cumulative).
- **flat%**: Percentage of total profile time.

#### Common pprof Commands

```
(pprof) top 20          # Show top 20 functions by flat time
(pprof) top -cum 20     # Show top 20 functions by cumulative time
(pprof) list doWork     # Show source code annotated with profile data
(pprof) web             # Open an SVG graph in your browser
(pprof) png > out.png   # Save a graph as PNG
(pprof) peek doWork     # Show callers and callees of doWork
(pprof) tree            # Show call tree
```

#### Memory Profiling

```bash
# From a running server
go tool pprof http://localhost:6060/debug/pprof/heap

# Analyze a saved profile
go tool pprof mem.prof
```

Memory profiles have four views. Switch between them:

```
(pprof) sample_index = alloc_objects   # Number of allocations (regardless of size)
(pprof) sample_index = alloc_space     # Bytes allocated (total, including freed)
(pprof) sample_index = inuse_objects   # Number of live objects on heap now
(pprof) sample_index = inuse_space     # Bytes currently on heap (live memory)
```

- Use `alloc_objects` to find code that allocates frequently (GC pressure).
- Use `inuse_space` to find code that holds the most live memory.

Example output:

```
(pprof) top 10 -sample_index=alloc_space
Showing nodes accounting for 512.17MB, 98.50% of 520MB total
      flat  flat%   sum%        cum   cum%
  256.08MB 49.25% 49.25%   256.08MB 49.25%  main.doWork
  128.04MB 24.62% 73.87%   128.04MB 24.62%  fmt.Sprintf
   64.03MB 12.31% 86.18%    64.03MB 12.31%  bytes.makeSlice
   32.01MB  6.16% 92.34%    32.01MB  6.16%  strings.(*Builder).grow
   16.01MB  3.08% 95.42%   512.17MB 98.50%  main.main
    8.00MB  1.54% 96.96%     8.00MB  1.54%  runtime.allocm
    4.00MB  0.77% 97.73%     4.00MB  0.77%  runtime.malg
    2.00MB  0.38% 98.11%     2.00MB  0.38%  os.newFile
    1.00MB  0.19% 98.31%     1.00MB  0.19%  bufio.NewReaderSize
    1.00MB  0.19% 98.50%     1.00MB  0.19%  net.newFD
```

#### Goroutine Profiling

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

This is invaluable for diagnosing goroutine leaks. If the number of goroutines grows over time, this profile shows you where they are stuck.

```
(pprof) top
Showing nodes accounting for 10042, 100% of 10042 total
      flat  flat%   sum%        cum   cum%
     10000 99.58% 99.58%      10000 99.58%  runtime.gopark
        20  0.20% 99.78%         20  0.20%  runtime.netpollblock
        12  0.12% 99.90%         12  0.12%  runtime/internal/syscall.Syscall6
        10  0.10% 100.0%         10  0.10%  runtime.goroutineReady
```

Ten thousand goroutines parked? You have a leak. Use `list` or `tree` to find who created them.

#### Block Profiling

Block profiling records when goroutines block on synchronization primitives (channels, mutexes, select, etc.). You must enable it first:

```go
runtime.SetBlockProfileRate(1) // 1 = record every block event; higher = sample
```

```bash
go tool pprof http://localhost:6060/debug/pprof/block
```

### Reading Flame Graphs

Flame graphs are the most intuitive way to visualize profiles. Use the `-http` flag:

```bash
go tool pprof -http=:8080 cpu.prof
```

This opens a web UI with multiple views, including a flame graph. How to read it:

```
                    +--------------------------+
                    |       main.main          |   <- entry point at bottom
                    +--------------------------+
                    |    main.processRequests  |
                    +--------+---------+-------+
                    | doAuth | doQuery | doLog |   <- callees stacked above
                    +--------+---------+-------+
                    |        | doScan  |       |
                    |        +---------+       |
```

- **Width** = time spent in that function (wider = more time).
- **Height** = call depth (bottom is the entry point, top is the leaf).
- **Color** = usually random; some tools color by package.

What to look for:
- **Wide bars at the top** (leaves) = functions consuming CPU time directly.
- **Wide bars in the middle** = functions that delegate to many children.
- **Narrow tall towers** = deep but infrequent call chains (usually not a problem).

**Node.js comparison:** In Node.js, you get flame graphs via `0x`, `clinic flame`, or Chrome DevTools. The concept is identical. The difference is that Go's pprof is built in and does not require any third-party tool.

---

## 3. Benchmarking

### Writing Effective Benchmarks

Go has first-class benchmark support in the `testing` package:

```go
// string_concat_test.go
package stringops

import (
    "fmt"
    "strings"
    "testing"
)

// Benchmark string concatenation with + operator
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "hello"
        }
    }
}

// Benchmark string concatenation with strings.Builder
func BenchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("hello")
        }
        _ = sb.String()
    }
}

// Benchmark string concatenation with pre-sized strings.Builder
func BenchmarkStringBuilderPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        sb.Grow(500) // 100 * len("hello")
        for j := 0; j < 100; j++ {
            sb.WriteString("hello")
        }
        _ = sb.String()
    }
}

// Benchmark fmt.Sprintf
func BenchmarkSprintf(b *testing.B) {
    for i := 0; i < b.N; i++ {
        parts := make([]any, 100)
        for j := 0; j < 100; j++ {
            parts[j] = "hello"
        }
        _ = fmt.Sprintf(strings.Repeat("%s", 100), parts...)
    }
}
```

Run benchmarks:

```bash
# Run all benchmarks in the current package
go test -bench=. -benchmem

# Run a specific benchmark
go test -bench=BenchmarkStringBuilder -benchmem

# Run with a minimum duration (more stable results)
go test -bench=. -benchmem -benchtime=5s

# Run with count for statistical significance
go test -bench=. -benchmem -count=10
```

Example output:

```
BenchmarkStringConcat-8            18316     65420 ns/op    53240 B/op     99 allocs/op
BenchmarkStringBuilder-8          245678      4872 ns/op      864 B/op      6 allocs/op
BenchmarkStringBuilderPrealloc-8  398204      3012 ns/op      512 B/op      1 allocs/op
BenchmarkSprintf-8                  8234    145321 ns/op    87432 B/op    203 allocs/op
```

Reading the output:
- `-8` = GOMAXPROCS (8 CPU cores).
- `245678` = number of iterations `b.N` settled on.
- `4872 ns/op` = nanoseconds per operation.
- `864 B/op` = bytes allocated per operation.
- `6 allocs/op` = number of heap allocations per operation.

The `strings.Builder` with preallocation is **21x faster** than `+` concatenation and uses **99% fewer allocations**.

### b.ReportAllocs()

You can also call `b.ReportAllocs()` inside the benchmark instead of passing `-benchmem`:

```go
func BenchmarkSomething(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        // ...
    }
}
```

### Sub-Benchmarks and Table-Driven Benchmarks

```go
func BenchmarkSliceAppend(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000, 100000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                s := make([]int, 0)
                for j := 0; j < size; j++ {
                    s = append(s, j)
                }
            }
        })

        b.Run(fmt.Sprintf("prealloc-size=%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                s := make([]int, 0, size)
                for j := 0; j < size; j++ {
                    s = append(s, j)
                }
            }
        })
    }
}
```

Output:

```
BenchmarkSliceAppend/size=10-8             12456789      96.3 ns/op
BenchmarkSliceAppend/prealloc-size=10-8    20345678      58.2 ns/op
BenchmarkSliceAppend/size=100-8             1834567     654.0 ns/op
BenchmarkSliceAppend/prealloc-size=100-8    3456789     345.0 ns/op
BenchmarkSliceAppend/size=1000-8             234567    5120.0 ns/op
BenchmarkSliceAppend/prealloc-size=1000-8    567890    2098.0 ns/op
BenchmarkSliceAppend/size=10000-8             23456   51234.0 ns/op
BenchmarkSliceAppend/prealloc-size=10000-8    56789   19876.0 ns/op
BenchmarkSliceAppend/size=100000-8             2345  512340.0 ns/op
BenchmarkSliceAppend/prealloc-size=100000-8    5678  198760.0 ns/op
```

Preallocation consistently wins by ~2-3x due to avoided reallocation and copying.

### benchstat for Comparison

`benchstat` is the standard tool for comparing benchmark results with statistical rigor:

```bash
# Install benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# Run benchmarks before optimization
go test -bench=. -count=10 -benchmem > old.txt

# Make your optimization...

# Run benchmarks after optimization
go test -bench=. -count=10 -benchmem > new.txt

# Compare
benchstat old.txt new.txt
```

Output:

```
goos: linux
goarch: amd64
pkg: example.com/myapp
cpu: AMD Ryzen 9 5900X

                  │   old.txt   │              new.txt               │
                  │   sec/op    │   sec/op     vs base               │
ProcessRequest-8   45.23u ± 2%   12.45u ± 1%  -72.47% (p=0.000 n=10)
ParsePayload-8     8.34u ± 3%    7.98u ± 2%   -4.32% (p=0.023 n=10)
ValidateInput-8    1.23u ± 1%    1.21u ± 1%        ~ (p=0.089 n=10)

                  │   old.txt    │              new.txt               │
                  │    B/op      │    B/op      vs base               │
ProcessRequest-8   4.125Ki ± 0%   0.512Ki ± 0%  -87.58% (p=0.000 n=10)
ParsePayload-8     2.048Ki ± 0%   2.048Ki ± 0%        ~ (p=1.000 n=10)
ValidateInput--8     128.0 ± 0%     128.0 ± 0%        ~ (p=1.000 n=10)
```

The `~` symbol means "no statistically significant change" (p-value > 0.05). The `-count=10` is critical -- a single run is meaningless for comparison.

### Avoiding Benchmark Pitfalls

#### Pitfall 1: Compiler Eliminating Dead Code

```go
// BAD: Compiler may optimize away the entire loop
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        computeSomething(42) // Return value unused -- compiler may remove this
    }
}

// GOOD: Use the result to prevent dead code elimination
var benchResult int // Package-level variable; compiler cannot optimize it away

func BenchmarkGood(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = computeSomething(42)
    }
    benchResult = r // Assign to package-level var at the end
}
```

#### Pitfall 2: Measuring Setup Time

```go
// BAD: setup is included in the measurement
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        data := generateLargeDataset() // This is setup, not what we are benchmarking
        processData(data)
    }
}

// GOOD: reset the timer after setup
func BenchmarkGood(b *testing.B) {
    data := generateLargeDataset()
    b.ResetTimer() // Exclude setup from measurement

    for i := 0; i < b.N; i++ {
        processData(data)
    }
}
```

#### Pitfall 3: Cache Effects / Input Variation

```go
// BAD: Same input every iteration -- CPU branch predictor learns the pattern
func BenchmarkBad(b *testing.B) {
    input := "the same string every time"
    for i := 0; i < b.N; i++ {
        processString(input)
    }
}

// BETTER: Vary input to simulate real workload
func BenchmarkBetter(b *testing.B) {
    inputs := make([]string, 1000)
    for i := range inputs {
        inputs[i] = fmt.Sprintf("input-%d-with-varying-length-%s", i, strings.Repeat("x", i%50))
    }
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        processString(inputs[i%len(inputs)])
    }
}
```

#### Pitfall 4: Not Using -count

A single benchmark run is noisy. Always use `-count=6` or higher and feed the results to `benchstat`.

**Node.js comparison:** In Node.js, you would use `benchmark.js` or `vitest bench`. Go's built-in benchmark framework is tightly integrated with the test runner and the profiler. `benchmark.js` is more of a standalone library. Go benchmarks also report allocation counts natively, which `benchmark.js` cannot do.

---

## 4. Memory Optimization

### WHY Allocations Matter

Every heap allocation in Go:
1. Calls `runtime.mallocgc` -- this is not free (~50-100ns per allocation).
2. Creates work for the garbage collector -- it must scan, mark, and sweep every live object.
3. Can trigger a GC cycle -- if the heap has grown past the GC threshold, your goroutine may be forced to assist with GC work.

Reducing allocations does NOT just save memory. It saves CPU time by reducing GC pressure.

### Reducing Allocations

#### Pre-allocate Slices

```go
// BAD: 20+ allocations as the slice grows
func collectIDs(users []User) []int {
    var ids []int // nil slice, cap=0
    for _, u := range users {
        ids = append(ids, u.ID)
    }
    return ids
}

// GOOD: 1 allocation
func collectIDs(users []User) []int {
    ids := make([]int, 0, len(users)) // Pre-allocate exact capacity
    for _, u := range users {
        ids = append(ids, u.ID)
    }
    return ids
}

// ALSO GOOD: Direct indexing when length is known
func collectIDs(users []User) []int {
    ids := make([]int, len(users))
    for i, u := range users {
        ids[i] = u.ID
    }
    return ids
}
```

#### Pre-allocate Maps

```go
// BAD: Multiple resizes as map grows
func buildIndex(items []Item) map[string]Item {
    m := make(map[string]Item) // Default small capacity
    for _, item := range items {
        m[item.Key] = item
    }
    return m
}

// GOOD: Allocate with expected size
func buildIndex(items []Item) map[string]Item {
    m := make(map[string]Item, len(items))
    for _, item := range items {
        m[item.Key] = item
    }
    return m
}
```

#### strings.Builder Instead of Concatenation

```go
// BAD: Each += allocates a new string
func buildCSV(rows [][]string) string {
    result := ""
    for _, row := range rows {
        for i, col := range row {
            if i > 0 {
                result += ","
            }
            result += col
        }
        result += "\n"
    }
    return result
}

// GOOD: Single allocation at the end
func buildCSV(rows [][]string) string {
    var sb strings.Builder
    // Estimate total size to avoid re-growing
    sb.Grow(len(rows) * 100) // rough estimate

    for _, row := range rows {
        for i, col := range row {
            if i > 0 {
                sb.WriteByte(',')
            }
            sb.WriteString(col)
        }
        sb.WriteByte('\n')
    }
    return sb.String()
}
```

#### Avoid String-to-Byte-Slice Conversions in Hot Paths

```go
// BAD: Allocates a new []byte every call
func hashPassword(password string) []byte {
    return sha256.Sum256([]byte(password)) // []byte(password) allocates
}

// If called in a hot loop, accept []byte directly:
func hashPassword(password []byte) [32]byte {
    return sha256.Sum256(password) // No allocation
}
```

> Note: The compiler can sometimes optimize `[]byte(string)` conversions away, but do not rely on it in hot paths.

### sync.Pool

`sync.Pool` is a cache of temporary objects that can be reused to avoid allocations. It is particularly useful for objects that are:
- Expensive to create.
- Used frequently.
- Short-lived (needed for one request, then returned).

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) string {
    // Get a buffer from the pool (or create a new one)
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset() // CRITICAL: always reset before use

    // Use the buffer
    buf.Write(data)
    buf.WriteString(" processed")
    result := buf.String()

    // Return the buffer to the pool
    bufPool.Put(buf)

    return result
}
```

Benchmark comparison:

```go
func BenchmarkWithoutPool(b *testing.B) {
    data := []byte("hello world")
    for i := 0; i < b.N; i++ {
        buf := new(bytes.Buffer)
        buf.Write(data)
        buf.WriteString(" processed")
        _ = buf.String()
    }
}

func BenchmarkWithPool(b *testing.B) {
    data := []byte("hello world")
    for i := 0; i < b.N; i++ {
        buf := bufPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.Write(data)
        buf.WriteString(" processed")
        _ = buf.String()
        bufPool.Put(buf)
    }
}
```

Typical results:

```
BenchmarkWithoutPool-8    5678901    210 ns/op    256 B/op    2 allocs/op
BenchmarkWithPool-8      12345678     87 ns/op     32 B/op    1 allocs/op
```

**sync.Pool rules:**
- Objects in the pool may be collected by the GC at any time -- do not rely on them being there.
- Always `Reset()` objects before reuse.
- Do not store pointers to pool objects long-term.
- Best for high-frequency, short-lived allocations (HTTP handlers, serializers, etc.).

### Struct Field Ordering for Alignment

The Go compiler aligns struct fields to their natural boundaries. This can cause padding (wasted bytes) if fields are ordered poorly.

```go
// BAD: 32 bytes due to padding
type BadStruct struct {
    a bool    // 1 byte  + 7 bytes padding (to align b)
    b float64 // 8 bytes
    c bool    // 1 byte  + 3 bytes padding (to align d)
    d int32   // 4 bytes
    e bool    // 1 byte  + 7 bytes padding (to align to struct alignment)
}
// sizeof = 1+7 + 8 + 1+3 + 4 + 1+7 = 32 bytes

// GOOD: 16 bytes, same fields
type GoodStruct struct {
    b float64 // 8 bytes
    d int32   // 4 bytes
    a bool    // 1 byte
    c bool    // 1 byte
    e bool    // 1 byte + 1 byte padding (to align to 8-byte boundary)
}
// sizeof = 8 + 4 + 1 + 1 + 1 + 1 = 16 bytes
```

The rule: **order fields from largest to smallest alignment requirement**.

```go
// Verify with unsafe.Sizeof
fmt.Println(unsafe.Sizeof(BadStruct{}))  // 32
fmt.Println(unsafe.Sizeof(GoodStruct{})) // 16
```

Use the `fieldalignment` linter to detect this automatically:

```bash
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
fieldalignment ./...
```

Or use `structlayout` for a visual representation:

```bash
go install honnef.co/go/tools/cmd/structlayout@latest
structlayout -json ./pkg MyStruct | structlayout-pretty
```

**When does alignment matter?** When you have millions of instances of a struct (e.g., in a large slice). The difference between 32 bytes and 16 bytes per struct means 50% less memory and better cache utilization. For a struct you create once, it does not matter at all.

**Node.js comparison:** In Node.js, you have zero control over object memory layout. V8 decides everything. In Go, you control the exact byte layout of every struct. This is one of Go's greatest advantages for performance-critical code.

---

## 5. CPU Optimization

### Identifying Hot Paths

A "hot path" is code that executes many times -- the inner loops of your program. Optimizing a function called once is useless. Optimizing a function called 10 million times per second is everything.

Use pprof to identify hot paths:

```bash
go tool pprof -http=:8080 cpu.prof
# Navigate to the flame graph
# The widest bars at the top are your hot paths
```

### Inlining

The Go compiler inlines small functions -- it replaces the function call with the function body. This eliminates call overhead and enables further optimizations.

```go
// This function is small enough to be inlined
func add(a, b int) int {
    return a + b
}

// This function calls add() in a hot loop
// The compiler will inline add(), making it as fast as "sum += i + i*2"
func sumPairs(n int) int {
    sum := 0
    for i := 0; i < n; i++ {
        sum = add(i, i*2) // Inlined by the compiler
    }
    return sum
}
```

Check what the compiler inlines:

```bash
go build -gcflags='-m' ./...
```

Output:

```
./math.go:4:6: can inline add
./math.go:9:6: can inline sumPairs
./math.go:12:13: inlining call to add
```

Use `//go:noinline` to prevent inlining (useful for benchmarking to measure real call overhead):

```go
//go:noinline
func addNoInline(a, b int) int {
    return a + b
}
```

Benchmark the difference:

```go
func BenchmarkInlined(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = add(i, i)
    }
}

func BenchmarkNotInlined(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = addNoInline(i, i)
    }
}
```

```
BenchmarkInlined-8       1000000000    0.28 ns/op    0 B/op    0 allocs/op
BenchmarkNotInlined-8     598234567    2.01 ns/op    0 B/op    0 allocs/op
```

Inlining made the function 7x faster by eliminating call overhead entirely.

### Bounds Check Elimination (BCE)

Go inserts bounds checks on slice/array access to prevent out-of-bounds panics. These checks have a cost in tight loops.

```go
// BAD: Bounds check on every iteration
func sumSlice(s []int) int {
    sum := 0
    for i := 0; i < len(s); i++ {
        sum += s[i] // Bounds check here
    }
    return sum
}

// GOOD: range eliminates bounds checks (compiler proves safety)
func sumSliceRange(s []int) int {
    sum := 0
    for _, v := range s {
        sum += v // No bounds check needed
    }
    return sum
}

// GOOD: Prove bounds once, eliminate subsequent checks
func processSlice(s []int) {
    if len(s) < 4 {
        return
    }
    // After the length check, the compiler knows s[0] through s[3] are safe
    _ = s[3] // Explicit hint to the compiler: "I need at least 4 elements"
    a := s[0] // No bounds check
    b := s[1] // No bounds check
    c := s[2] // No bounds check
    d := s[3] // No bounds check
    _ = a + b + c + d
}
```

Check bounds check elimination with:

```bash
go build -gcflags='-d=ssa/check_bce/debug=1' ./...
```

Output shows where bounds checks remain:

```
./process.go:15:12: Found IsInBounds
```

### Avoiding Interface Dispatch in Hot Loops

Interface method calls require an indirect call through the interface's method table (itab). This is slower than a direct function call and prevents inlining.

```go
type Processor interface {
    Process(data []byte) []byte
}

type FastProcessor struct{}

func (p FastProcessor) Process(data []byte) []byte {
    // ... process data ...
    return data
}

// BAD: Interface dispatch in hot loop -- prevents inlining, adds indirection
func processAll(p Processor, items [][]byte) {
    for _, item := range items {
        p.Process(item) // Indirect call through interface
    }
}

// GOOD: Use concrete type in hot loop
func processAllFast(p FastProcessor, items [][]byte) {
    for _, item := range items {
        p.Process(item) // Direct call, can be inlined
    }
}

// PATTERN: Accept interface at API boundary, use concrete type internally
func ProcessItems(p Processor, items [][]byte) {
    // Type-assert once at the boundary
    if fp, ok := p.(FastProcessor); ok {
        processAllFast(fp, items) // Fast path with concrete type
        return
    }
    processAll(p, items) // Slow path for other implementations
}
```

### Avoiding Unnecessary Conversions and Allocations in Hot Paths

```go
// BAD: fmt.Sprintf allocates every time
func formatKey(prefix string, id int) string {
    return fmt.Sprintf("%s:%d", prefix, id)
}

// GOOD: strconv + string concatenation for simple cases
func formatKey(prefix string, id int) string {
    return prefix + ":" + strconv.Itoa(id)
}

// BETTER: For extreme hot paths, use a byte buffer
func formatKeyFast(buf []byte, prefix string, id int) []byte {
    buf = buf[:0]
    buf = append(buf, prefix...)
    buf = append(buf, ':')
    buf = strconv.AppendInt(buf, int64(id), 10)
    return buf
}
```

Benchmark:

```
BenchmarkFmtSprintf-8      5000000    240 ns/op    48 B/op    3 allocs/op
BenchmarkStrconcat-8       20000000     62 ns/op    24 B/op    1 allocs/op
BenchmarkByteBuffer-8      50000000     31 ns/op     0 B/op    0 allocs/op
```

---

## 6. Concurrency Optimization

### Proper Goroutine Pool Sizing

Do not create unlimited goroutines. Use a worker pool pattern:

```go
func processItems(items []Item, workerCount int) []Result {
    // Use a semaphore pattern with a buffered channel
    sem := make(chan struct{}, workerCount)
    results := make([]Result, len(items))
    var wg sync.WaitGroup

    for i, item := range items {
        wg.Add(1)
        sem <- struct{}{} // Acquire semaphore (blocks if pool is full)

        go func(idx int, it Item) {
            defer wg.Done()
            defer func() { <-sem }() // Release semaphore

            results[idx] = processItem(it)
        }(i, item)
    }

    wg.Wait()
    return results
}
```

How to size the pool:
- **CPU-bound work:** `runtime.NumCPU()` workers. More does not help and adds scheduling overhead.
- **I/O-bound work:** Higher count (10x-100x NumCPU). Goroutines spend most time waiting.
- **Mixed:** Benchmark with different values. There is no formula.

```go
// Good default for CPU-bound work
workerCount := runtime.GOMAXPROCS(0) // Returns current GOMAXPROCS (usually NumCPU)

// For I/O-bound work
workerCount := runtime.GOMAXPROCS(0) * 10
```

### Reducing Lock Contention

Locks are a major source of performance problems in concurrent Go programs.

#### Strategy 1: Reduce Critical Section Size

```go
// BAD: Lock held during expensive computation
func (s *Store) Process(key string) Result {
    s.mu.Lock()
    defer s.mu.Unlock()

    data := s.data[key]
    result := expensiveComputation(data) // Other goroutines blocked here!
    s.results[key] = result
    return result
}

// GOOD: Lock only for data access, not computation
func (s *Store) Process(key string) Result {
    s.mu.RLock()
    data := s.data[key]
    s.mu.RUnlock()

    result := expensiveComputation(data) // No lock held during computation

    s.mu.Lock()
    s.results[key] = result
    s.mu.Unlock()

    return result
}
```

#### Strategy 2: Use RWMutex for Read-Heavy Workloads

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

// Multiple goroutines can read simultaneously
func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

// Only one goroutine can write at a time
func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

#### Strategy 3: Sharded Data Structures

```go
const numShards = 256

type ShardedMap struct {
    shards [numShards]struct {
        mu   sync.RWMutex
        data map[string]string
    }
}

func NewShardedMap() *ShardedMap {
    sm := &ShardedMap{}
    for i := range sm.shards {
        sm.shards[i].data = make(map[string]string)
    }
    return sm
}

func (sm *ShardedMap) getShard(key string) *struct {
    mu   sync.RWMutex
    data map[string]string
} {
    h := fnv32(key)
    return &sm.shards[h%numShards]
}

func fnv32(key string) uint32 {
    hash := uint32(2166136261)
    for i := 0; i < len(key); i++ {
        hash *= 16777619
        hash ^= uint32(key[i])
    }
    return hash
}

func (sm *ShardedMap) Get(key string) (string, bool) {
    shard := sm.getShard(key)
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    val, ok := shard.data[key]
    return val, ok
}

func (sm *ShardedMap) Set(key, value string) {
    shard := sm.getShard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    shard.data[key] = value
}
```

With 256 shards, contention is reduced by ~256x compared to a single lock.

#### Strategy 4: Use sync/atomic for Simple Counters

```go
// BAD: Mutex for a simple counter
type CounterMutex struct {
    mu    sync.Mutex
    count int64
}

func (c *CounterMutex) Increment() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// GOOD: Atomic operation -- lock-free
type CounterAtomic struct {
    count atomic.Int64
}

func (c *CounterAtomic) Increment() {
    c.count.Add(1)
}
```

Benchmark:

```
BenchmarkCounterMutex-8     20000000    65.3 ns/op
BenchmarkCounterAtomic-8    80000000    15.2 ns/op
```

Atomics are ~4x faster because they avoid OS-level locking.

### False Sharing and Cache-Line Alignment

False sharing occurs when two goroutines write to different variables that happen to share the same CPU cache line (typically 64 bytes). The CPU must invalidate the entire cache line, causing both goroutines to slow down.

```go
// BAD: Counters likely share a cache line
type Counters struct {
    a atomic.Int64 // 8 bytes
    b atomic.Int64 // 8 bytes -- on the same cache line as 'a'!
}

// GOOD: Pad to separate cache lines
type Counters struct {
    a atomic.Int64
    _ [56]byte // Padding to fill 64-byte cache line (64 - 8 = 56)
    b atomic.Int64
    _ [56]byte
}
```

When does false sharing matter? Only when two goroutines are writing to adjacent fields at very high frequency (millions of times per second). For most programs, this is irrelevant. Profile first.

---

## 7. Compiler Optimizations

### Inspecting Compiler Decisions

The Go compiler performs many optimizations automatically. Use `-gcflags` to see what it does:

```bash
# Show optimization decisions (inlining, escape analysis)
go build -gcflags='-m' ./...

# More verbose (two -m flags)
go build -gcflags='-m -m' ./...

# Show inlining decisions only
go build -gcflags='-m' ./... 2>&1 | grep 'inlining'

# Show escape analysis only
go build -gcflags='-m' ./... 2>&1 | grep 'escapes\|does not escape'

# Disable inlining (for testing)
go build -gcflags='-l' ./...

# Disable optimizations entirely
go build -gcflags='-N -l' ./...
```

### Escape Analysis

Escape analysis determines whether a variable can live on the stack (fast, no GC) or must go on the heap (slower, GC must track it).

```go
// This does NOT escape -- allocated on the stack
func sum(a, b int) int {
    result := a + b // 'result' lives on the stack
    return result   // Returned by value (copied)
}

// This DOES escape -- must be allocated on the heap
func newUser(name string) *User {
    u := User{Name: name} // 'u' escapes because we return a pointer to it
    return &u              // Pointer to a local variable -> must live beyond this frame
}
```

Check with:

```bash
go build -gcflags='-m' ./...
```

```
./user.go:10:2: moved to heap: u   # This variable escapes to the heap
./math.go:4:2: result does not escape
```

Common causes of escape:
1. **Returning a pointer** to a local variable.
2. **Storing in an interface** (the compiler cannot determine the concrete type at compile time).
3. **Sending to a channel** (the receiving goroutine may outlive the sender).
4. **Closures capturing variables** (the closure may outlive the enclosing function).
5. **Slices that grow** beyond their initial capacity (the backing array may be reallocated).

#### Reducing Escapes

```go
// BAD: Returns pointer, causes escape
func createBuffer() *bytes.Buffer {
    buf := bytes.Buffer{} // escapes to heap
    return &buf
}

// GOOD: Accept pointer as parameter (caller controls allocation)
func initBuffer(buf *bytes.Buffer) {
    buf.Reset()
    buf.Grow(1024)
}

// Usage: buf can be stack-allocated if it does not escape the caller
var buf bytes.Buffer
initBuffer(&buf)
```

### Dead Code Elimination

The compiler removes code that can never execute:

```go
const debug = false

func process(data []byte) {
    if debug {
        log.Printf("processing %d bytes", len(data)) // Compiler removes this entirely
    }
    // ... actual processing ...
}
```

This is free at runtime -- the `if debug` block generates zero machine code.

### SSA Optimizations

The Go compiler converts code to Static Single Assignment (SSA) form and applies many optimizations:

- Constant folding (`3 + 4` becomes `7`)
- Copy propagation (eliminate unnecessary copies)
- Common subexpression elimination
- Loop-invariant code motion
- Nil check elimination
- Strength reduction (replace expensive operations with cheaper ones)

View the SSA output:

```bash
GOSSAFUNC=processData go build ./...
# This creates an ssa.html file you can open in a browser
```

The HTML file shows each SSA optimization pass and what it changed -- fascinating for understanding the compiler but rarely needed in practice.

---

## 8. I/O Optimization

### Buffered I/O

Unbuffered I/O makes a syscall for every read/write. Syscalls are expensive (~100-1000ns each). Buffered I/O batches operations.

```go
// BAD: Unbuffered -- one syscall per line
func writeLines(filename string, lines []string) error {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    for _, line := range lines {
        f.WriteString(line + "\n") // Syscall for every line!
    }
    return nil
}

// GOOD: Buffered -- syscalls only when buffer is full (default 4KB)
func writeLines(filename string, lines []string) error {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    w := bufio.NewWriter(f)
    defer w.Flush() // CRITICAL: flush remaining buffer at the end

    for _, line := range lines {
        w.WriteString(line)
        w.WriteByte('\n')
    }
    return nil
}

// BETTER: Larger buffer for large writes
func writeLines(filename string, lines []string) error {
    f, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    w := bufio.NewWriterSize(f, 64*1024) // 64KB buffer
    defer w.Flush()

    for _, line := range lines {
        w.WriteString(line)
        w.WriteByte('\n')
    }
    return nil
}
```

Same for reading:

```go
// BAD: One syscall per read
reader := os.Open("data.txt")

// GOOD: Buffered
reader := bufio.NewReader(os.Open("data.txt"))

// BETTER: Larger buffer
reader := bufio.NewReaderSize(os.Open("data.txt"), 64*1024)
```

Benchmark for writing 1 million lines:

```
BenchmarkUnbuffered-8       1    4523456789 ns/op    (4.5 seconds)
BenchmarkBuffered4KB-8      3     412345678 ns/op    (0.4 seconds)
BenchmarkBuffered64KB-8     5     234567890 ns/op    (0.2 seconds)
```

Buffering gives a **10-20x speedup** for sequential I/O.

### Connection Pooling

`net/http` has built-in connection pooling. Tune it for your workload:

```go
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,              // Total idle connections across all hosts
        MaxIdleConnsPerHost: 20,               // Idle connections per host
        MaxConnsPerHost:     100,              // Max total connections per host
        IdleConnTimeout:     90 * time.Second, // How long to keep idle connections
        DisableCompression:  false,
        ForceAttemptHTTP2:   true,             // Enable HTTP/2
    },
    Timeout: 30 * time.Second,
}
```

**Critical mistake:** Not reading the response body completely. This prevents connection reuse:

```go
// BAD: Connection cannot be reused
resp, err := client.Get("https://api.example.com/data")
if err != nil {
    return err
}
defer resp.Body.Close()
// Oops, we did not read the body -- connection is wasted

// GOOD: Always drain the body
resp, err := client.Get("https://api.example.com/data")
if err != nil {
    return err
}
defer resp.Body.Close()
defer io.Copy(io.Discard, resp.Body) // Drain remaining body to enable connection reuse

body, err := io.ReadAll(resp.Body)
```

### HTTP/2 Multiplexing

HTTP/2 sends multiple requests over a single TCP connection. Go supports it by default with TLS:

```go
// HTTP/2 is automatically used for HTTPS with the default transport
client := &http.Client{
    Transport: &http.Transport{
        ForceAttemptHTTP2: true,
    },
}

// Multiple concurrent requests share one connection to the same host
var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        resp, _ := client.Get("https://api.example.com/data")
        if resp != nil {
            io.Copy(io.Discard, resp.Body)
            resp.Body.Close()
        }
    }()
}
wg.Wait()
```

### Reducing Syscalls

Every syscall is a context switch between user space and kernel space. Reduce them:

```go
// BAD: Many small writes
for _, b := range data {
    conn.Write([]byte{b}) // Syscall per byte!
}

// GOOD: One large write
conn.Write(data) // Single syscall

// GOOD: Use writev (scatter/gather I/O) via net.Buffers
bufs := net.Buffers{header, body, trailer}
bufs.WriteTo(conn) // Single writev syscall for multiple buffers
```

**Node.js comparison:** Node.js handles I/O buffering via its event loop and libuv. Streams in Node.js buffer by default. Go requires you to explicitly use `bufio` for buffering. However, Go's `net/http` server is already well-buffered internally.

---

## 9. GC Tuning

### Understanding Go's Garbage Collector

Go uses a **concurrent, tri-color mark-and-sweep** garbage collector. Key properties:
- **Concurrent:** GC runs alongside your program (mostly).
- **Non-generational:** Unlike V8 (Node.js), Go's GC does not distinguish young/old objects.
- **Low latency:** Sub-millisecond pause times for most workloads.
- **Tunable:** You control when GC runs via `GOGC` and `GOMEMLIMIT`.

### GOGC

`GOGC` controls how often GC runs. It sets the target percentage of heap growth before the next GC cycle:

```
GOGC=100 (default): GC triggers when heap has grown 100% since last GC
GOGC=200:           GC triggers when heap has grown 200% (less frequent GC, more memory)
GOGC=50:            GC triggers when heap has grown 50% (more frequent GC, less memory)
GOGC=off:           GC is disabled entirely (use with extreme caution)
```

```bash
# Set via environment variable
GOGC=200 ./myapp

# Or in code
import "runtime/debug"

func init() {
    debug.SetGCPercent(200) // Same as GOGC=200
}
```

**When to increase GOGC:**
- Your program has plenty of memory but CPU is the bottleneck.
- GC is consuming too much CPU time (visible in pprof).

**When to decrease GOGC:**
- Memory is constrained and you want tighter heap usage.

### GOMEMLIMIT (Go 1.19+)

`GOMEMLIMIT` sets a soft memory limit for the Go runtime. This is the modern, preferred way to tune GC behavior:

```bash
# Set a 1GB memory limit
GOMEMLIMIT=1GiB ./myapp

# Or in code
import "runtime/debug"

func init() {
    debug.SetMemoryLimit(1 << 30) // 1 GiB
}
```

The GC will work harder (run more frequently) to stay under the limit. It does NOT crash if the limit is exceeded -- it is a soft limit.

**The recommended approach (Go 1.19+):**

```bash
# Set GOGC=off and rely on GOMEMLIMIT only
GOGC=off GOMEMLIMIT=1GiB ./myapp
```

This tells Go: "Use up to 1GB, and do not GC until you need to." This is optimal for services running in containers with known memory limits.

```go
// Typical container configuration
func configureRuntime() {
    // Container has 2GB memory limit
    // Leave 20% headroom for non-heap memory (goroutine stacks, etc.)
    debug.SetMemoryLimit(int64(float64(2<<30) * 0.8)) // 1.6 GiB
    debug.SetGCPercent(-1) // Equivalent to GOGC=off -- only GC when near the limit
}
```

### The Ballast Technique (Pre-1.19)

Before `GOMEMLIMIT`, a common trick was to allocate a large, unused byte slice to raise the GC threshold:

```go
func main() {
    // Allocate a 1GB "ballast" that is never used
    // This raises the baseline heap size, so GOGC=100 triggers at 2GB instead of ~100MB
    ballast := make([]byte, 1<<30) // 1 GiB
    _ = ballast // Keep it alive

    // ... rest of your program ...
}
```

Why this works: With `GOGC=100`, GC triggers when the heap doubles. If baseline heap is 10MB, GC triggers at 20MB. If baseline is 1GB (due to ballast), GC triggers at 2GB. Fewer GC cycles means less CPU spent on GC.

**This technique is obsolete with Go 1.19+.** Use `GOMEMLIMIT` instead.

### Understanding GC Pauses

GC pauses are the brief stop-the-world (STW) phases of garbage collection. In modern Go:
- **Mark setup:** ~10-30 microseconds STW.
- **Mark termination:** ~10-30 microseconds STW.
- **Marking:** Concurrent (no pause).
- **Sweeping:** Concurrent (no pause).

Total pause time is typically under 100 microseconds. If you see longer pauses, something is wrong (usually too many goroutines or an extremely large heap).

Monitor GC behavior:

```bash
# Print GC trace
GODEBUG=gctrace=1 ./myapp
```

Output:

```
gc 1 @0.012s 2%: 0.011+0.45+0.004 ms clock, 0.089+0.36/0.82/0.15+0.032 ms cpu,
4->4->1 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P
```

Reading the output:
- `gc 1` = GC cycle number.
- `@0.012s` = time since program start.
- `2%` = percentage of CPU time spent in GC.
- `0.011+0.45+0.004 ms clock` = STW sweep termination + concurrent mark + STW mark termination.
- `4->4->1 MB` = heap before GC -> heap at GC trigger -> live heap after GC.
- `8 P` = number of processors used.

### runtime.ReadMemStats()

For programmatic monitoring:

```go
func printMemStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("Alloc = %v MiB\n", m.Alloc/1024/1024)           // Current heap in use
    fmt.Printf("TotalAlloc = %v MiB\n", m.TotalAlloc/1024/1024)  // Total ever allocated
    fmt.Printf("Sys = %v MiB\n", m.Sys/1024/1024)                // Total memory from OS
    fmt.Printf("NumGC = %v\n", m.NumGC)                          // Number of GC cycles
    fmt.Printf("GCPauseTotal = %v\n", time.Duration(m.PauseTotalNs))
    fmt.Printf("LastGCPause = %v\n", time.Duration(m.PauseNs[(m.NumGC+255)%256]))
}
```

**Node.js comparison:**

| Go | Node.js |
|---|---|
| `GOGC=200` | `--max-old-space-size=4096` (not equivalent, but closest) |
| `GOMEMLIMIT=1GiB` | No direct equivalent |
| `debug.SetGCPercent()` | `--gc-interval=100` (V8 flag, rarely used) |
| `runtime.ReadMemStats()` | `process.memoryUsage()` |
| `GODEBUG=gctrace=1` | `--trace-gc` |
| Sub-ms pauses | 10-100ms pauses for major GC (V8 has generational GC) |

Go gives you much finer control over GC behavior. Node.js/V8 is more "set it and forget it."

---

## 10. Tracing

### go tool trace

While pprof shows WHERE time is spent, the trace tool shows WHEN things happen. It captures a timeline of events: goroutine scheduling, GC activity, syscalls, network I/O, and more.

### Capturing a Trace

#### From a Running Server

```bash
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5
go tool trace trace.out
```

#### Programmatically

```go
package main

import (
    "os"
    "runtime/trace"
    "log"
)

func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    if err := trace.Start(f); err != nil {
        log.Fatal(err)
    }
    defer trace.Stop()

    // Your program logic here
    doWork()
}
```

#### From a Test

```bash
go test -trace=trace.out ./...
go tool trace trace.out
```

### Understanding Trace Output

```bash
go tool trace trace.out
# Opens a web UI at http://127.0.0.1:PORT
```

The trace viewer shows:

1. **Goroutine analysis:** How many goroutines are running vs. waiting at each point in time.
2. **Timeline view:** A Gantt chart of goroutine execution across all P's (processors).
3. **Network blocking profile:** Where goroutines are blocked on network I/O.
4. **Synchronization blocking profile:** Where goroutines are blocked on locks/channels.
5. **Syscall blocking profile:** Where goroutines are blocked in syscalls.
6. **Scheduler latency profile:** How long goroutines wait before being scheduled.

### Identifying Latency Sources

#### Goroutine Starvation

In the timeline view, look for goroutines that are "Runnable" (ready to run but not running) for long periods. This means they are waiting for a P (processor) to become available.

Causes:
- Too many goroutines competing for too few P's.
- Some goroutines running long CPU-bound operations without preemption points.
- `GOMAXPROCS` set too low.

#### GC Interference

In the timeline, GC phases are highlighted. Look for:
- Mark assist (your goroutines are forced to help with GC marking).
- STW (stop-the-world) pauses.

If GC is frequent and mark assist is high, reduce allocations or tune `GOGC` / `GOMEMLIMIT`.

#### Lock Contention

The synchronization blocking profile shows which locks are causing the most blocking. If one lock dominates, consider:
- Sharding (see Section 6).
- Using `sync.RWMutex` instead of `sync.Mutex`.
- Redesigning to avoid shared state.

### Custom Trace Regions and Tasks

Add custom annotations to the trace to mark your application's logical operations:

```go
import "runtime/trace"

func handleRequest(ctx context.Context, req *Request) {
    // Create a trace task (high-level operation)
    ctx, task := trace.NewTask(ctx, "handleRequest")
    defer task.End()

    // Create trace regions (sub-operations within the task)
    trace.WithRegion(ctx, "authenticate", func() {
        authenticate(ctx, req)
    })

    trace.WithRegion(ctx, "queryDatabase", func() {
        queryDatabase(ctx, req)
    })

    trace.WithRegion(ctx, "renderResponse", func() {
        renderResponse(ctx, req)
    })
}
```

These custom regions appear in the trace viewer, letting you see exactly where time is spent within your application's logical flow.

### Trace Log Messages

```go
trace.Log(ctx, "cache", "miss") // Appears in the trace timeline
trace.Log(ctx, "userID", "12345")
```

**Node.js comparison:**

| Go | Node.js |
|---|---|
| `go tool trace` | `async_hooks` + custom tracing |
| `trace.NewTask()` / `trace.WithRegion()` | `async_hooks` / `perf_hooks` |
| Built-in timeline viewer | Chrome DevTools / clinic.js |
| Goroutine scheduling visibility | Event loop delay monitoring (`monitorEventLoopDelay`) |

Go's trace tool provides far more detail about concurrent execution than anything available in Node.js out of the box.

---

## 11. Real-World Optimization Case Study

Let us take a realistic function and optimize it step by step, measuring at each stage.

### The Problem

We have a function that reads a CSV file of user records and builds a lookup map by email. It is called frequently and has been identified as a bottleneck via pprof.

### Version 1: Naive Implementation

```go
// v1: Naive -- allocates heavily, uses fmt.Sprintf unnecessarily
func buildUserIndex(csvData string) map[string]User {
    lines := strings.Split(csvData, "\n")
    index := make(map[string]User)

    for _, line := range lines {
        if line == "" {
            continue
        }
        fields := strings.Split(line, ",")
        if len(fields) < 4 {
            continue
        }

        age, _ := strconv.Atoi(fields[2])
        user := User{
            Name:  fmt.Sprintf("%s", fields[0]), // Unnecessary Sprintf!
            Email: fmt.Sprintf("%s", fields[1]), // Unnecessary Sprintf!
            Age:   age,
            City:  fmt.Sprintf("%s", fields[3]), // Unnecessary Sprintf!
        }
        index[strings.ToLower(user.Email)] = user
    }
    return index
}
```

Benchmark baseline:

```
BenchmarkBuildUserIndex_V1-8    234    5102345 ns/op    3245678 B/op    98765 allocs/op
```

Profile with pprof shows:
- 40% of time in `fmt.Sprintf` (completely unnecessary).
- 25% in `strings.Split` (allocates a new slice every call).
- 15% in `map` operations (map resizing due to no pre-allocation).
- 20% in `runtime.mallocgc` (GC pressure).

### Version 2: Remove Unnecessary Allocations

```go
// v2: Remove fmt.Sprintf, pre-allocate map
func buildUserIndex(csvData string) map[string]User {
    lines := strings.Split(csvData, "\n")
    index := make(map[string]User, len(lines)) // Pre-allocate

    for _, line := range lines {
        if line == "" {
            continue
        }
        fields := strings.Split(line, ",")
        if len(fields) < 4 {
            continue
        }

        age, _ := strconv.Atoi(fields[2])
        user := User{
            Name:  fields[0], // Direct assignment, no Sprintf
            Email: fields[1],
            Age:   age,
            City:  fields[3],
        }
        index[strings.ToLower(user.Email)] = user
    }
    return index
}
```

```
BenchmarkBuildUserIndex_V2-8    567    2012345 ns/op    1845678 B/op    45678 allocs/op
```

**Result: 2.5x faster, 43% fewer allocations.** Just by removing `fmt.Sprintf` and pre-allocating the map.

### Version 3: Avoid strings.Split for Line Parsing

`strings.Split` allocates a new slice. We can parse fields in place using `strings.Index`:

```go
// v3: Parse without allocating slices for each line
func buildUserIndex(csvData string) map[string]User {
    index := make(map[string]User, strings.Count(csvData, "\n"))

    for len(csvData) > 0 {
        // Find end of line
        lineEnd := strings.IndexByte(csvData, '\n')
        var line string
        if lineEnd == -1 {
            line = csvData
            csvData = ""
        } else {
            line = csvData[:lineEnd]
            csvData = csvData[lineEnd+1:]
        }

        if len(line) == 0 {
            continue
        }

        // Parse fields without allocating a []string
        f1 := strings.IndexByte(line, ',')
        if f1 == -1 {
            continue
        }
        rest := line[f1+1:]
        f2 := strings.IndexByte(rest, ',')
        if f2 == -1 {
            continue
        }
        rest2 := rest[f2+1:]
        f3 := strings.IndexByte(rest2, ',')
        if f3 == -1 {
            continue
        }

        name := line[:f1]
        email := rest[:f2]
        ageStr := rest2[:f3]
        city := rest2[f3+1:]

        age, _ := strconv.Atoi(ageStr)
        user := User{
            Name:  name,
            Email: email,
            Age:   age,
            City:  city,
        }
        index[strings.ToLower(email)] = user
    }
    return index
}
```

```
BenchmarkBuildUserIndex_V3-8    1234    912345 ns/op    645678 B/op    12345 allocs/op
```

**Result: 5.6x faster than V1, 87% fewer allocations.** Eliminating `strings.Split` per line was a major win because it avoided creating a `[]string` slice for every row.

### Version 4: Avoid strings.ToLower Allocation

`strings.ToLower` allocates a new string. If the email is already lowercase (common case), this is wasteful:

```go
// v4: Avoid ToLower allocation when not needed
func isLower(s string) bool {
    for i := 0; i < len(s); i++ {
        if s[i] >= 'A' && s[i] <= 'Z' {
            return false
        }
    }
    return true
}

func toLowerEmail(email string) string {
    if isLower(email) {
        return email // No allocation -- already lowercase
    }
    return strings.ToLower(email) // Allocate only when needed
}

// In buildUserIndex, replace:
//   index[strings.ToLower(email)] = user
// with:
//   index[toLowerEmail(email)] = user
```

```
BenchmarkBuildUserIndex_V4-8    1456    812345 ns/op    545678 B/op    10234 allocs/op
```

**Result: 6.3x faster than V1.** Small win, but every allocation saved matters under high load.

### Version 5: Reuse Buffers with sync.Pool (for Concurrent Use)

If this function is called per-request in an HTTP handler, use `sync.Pool` for the index map:

```go
var indexPool = sync.Pool{
    New: func() any {
        return make(map[string]User, 1024)
    },
}

func buildUserIndexPooled(csvData string) map[string]User {
    index := indexPool.Get().(map[string]User)
    // Clear the map (reuse memory, avoid re-allocation)
    for k := range index {
        delete(index, k)
    }
    // ... same parsing logic as V4 ...
    // Caller is responsible for calling indexPool.Put(index) when done
    return index
}
```

### Optimization Summary

| Version | ns/op | B/op | allocs/op | Speedup |
|---|---|---|---|---|
| V1 (Naive) | 5,102,345 | 3,245,678 | 98,765 | 1.0x |
| V2 (No Sprintf, prealloc map) | 2,012,345 | 1,845,678 | 45,678 | 2.5x |
| V3 (No Split per line) | 912,345 | 645,678 | 12,345 | 5.6x |
| V4 (Smart ToLower) | 812,345 | 545,678 | 10,234 | 6.3x |

**Key observation:** The biggest wins came from removing unnecessary allocations, not from clever algorithms. This is true for most Go optimization work.

---

## 12. Go vs Node.js Performance Tooling Comparison

### Profiling Tools

| Task | Go | Node.js |
|---|---|---|
| **CPU profiling** | `pprof` (built-in) | `--inspect` + Chrome DevTools, `v8-profiler` |
| **Heap profiling** | `pprof` heap profile | Chrome DevTools heap snapshot |
| **Flame graphs** | `go tool pprof -http=:8080` | `0x`, `clinic flame`, Chrome DevTools |
| **Benchmarking** | `testing.B` (built-in) | `benchmark.js`, `vitest bench`, `tinybench` |
| **GC monitoring** | `GODEBUG=gctrace=1` | `--trace-gc` |
| **Tracing** | `go tool trace` | `async_hooks`, `perf_hooks`, `clinic bubbleprof` |
| **Memory allocation tracking** | `-benchmem`, `pprof allocs` | No native equivalent; use heap snapshots |
| **Struct/memory layout** | `unsafe.Sizeof`, `fieldalignment` | Not possible (V8 controls layouts) |

### What Go Gives You That Node.js Cannot

1. **Allocation counting in benchmarks.** `-benchmem` tells you exactly how many heap allocations each operation makes. In Node.js, you have no direct equivalent.

2. **Struct memory layout control.** You can order fields, pad cache lines, and guarantee memory layout. V8 objects are opaque.

3. **Goroutine profiling.** You can see the stack trace of every goroutine. In Node.js, you can see async stack traces but not "where is every pending operation."

4. **Escape analysis output.** `go build -gcflags='-m'` tells you exactly which variables escape to the heap. V8 does not expose this.

5. **Deterministic benchmarks.** Go benchmarks run until results stabilize (`b.N` auto-adjusts). Node.js benchmarks require manual iteration count tuning.

6. **Integrated toolchain.** `go test -bench`, `go tool pprof`, `go tool trace` are all part of the standard distribution. Node.js requires installing multiple third-party packages.

### Where Node.js Competes

1. **I/O-bound workloads.** Node.js's event loop is highly optimized for I/O-heavy servers. Go is also excellent here, but the gap narrows for pure I/O workloads.

2. **Developer iteration speed.** No compilation step means faster development cycles (though `go run` is fast enough for most workflows).

3. **Chrome DevTools integration.** The Chrome DevTools profiling UI is arguably more polished than `pprof`'s web UI, with better heap snapshot exploration.

### Performance Characteristics

| Workload | Go | Node.js |
|---|---|---|
| **CPU-bound computation** | 5-50x faster | Slower (JIT overhead, no true parallelism without workers) |
| **JSON serialization** | 2-5x faster (`encoding/json`); 10-20x with `sonic`/`easyjson` | V8's JSON.parse/stringify is heavily optimized |
| **HTTP request handling** | 3-10x more req/s at lower latency | Competitive with frameworks like `fastify` |
| **I/O-bound (database queries)** | 1-2x faster | Competitive; bottleneck is the database |
| **Concurrent connections** | Handles 100K+ goroutines easily | Handles 10K+ connections via event loop |
| **Memory usage** | 10-100MB typical for web servers | 50-500MB typical (V8 heap overhead) |
| **Startup time** | 1-10ms (compiled binary) | 100-500ms (module loading, JIT warmup) |
| **GC pause times** | <1ms typically | 10-100ms for major GC |

---

## 13. Key Takeaways

1. **Measure before you optimize.** Use `pprof` and benchmarks. Never guess.

2. **Reduce allocations first.** The number one optimization in Go is reducing heap allocations. This reduces GC pressure AND directly reduces CPU time spent in `mallocgc`.

3. **Pre-allocate slices and maps** when you know the size. This is the single easiest optimization with the highest return.

4. **Use `strings.Builder` instead of `+` concatenation** for building strings in loops.

5. **Use `sync.Pool` for frequently allocated, short-lived objects** like buffers and temporary structs.

6. **Order struct fields from largest to smallest** to minimize padding. Use `fieldalignment` to check.

7. **Buffer your I/O.** Use `bufio.Reader` and `bufio.Writer` for file and network I/O.

8. **Use `GOMEMLIMIT` (Go 1.19+) instead of the ballast technique** for GC tuning.

9. **Use `benchstat` with `-count=10`** for statistically meaningful benchmark comparisons.

10. **Use `go tool trace` for concurrency issues** (goroutine scheduling, lock contention, GC interference). Use `pprof` for CPU and memory profiling.

11. **Avoid interface dispatch in hot loops.** Use concrete types where performance matters.

12. **Avoid `fmt.Sprintf` in hot paths.** Use `strconv` and direct string concatenation instead.

13. **Profile in production-like conditions.** Benchmarks on your laptop may not reflect production behavior due to different hardware, load patterns, and data sizes.

14. **The compiler is your friend.** Check `go build -gcflags='-m'` to understand inlining and escape analysis decisions. Work WITH the compiler, not against it.

---

## 14. Practice Exercises

### Exercise 1: Profile and Optimize a Slow Function

Write a function that processes a list of 1 million strings (convert to uppercase, trim whitespace, deduplicate). Profile it with pprof and optimize it to be at least 3x faster.

```go
// Start with this naive version:
func processStrings(input []string) []string {
    seen := make(map[string]bool)
    var result []string

    for _, s := range input {
        processed := strings.ToUpper(strings.TrimSpace(s))
        if !seen[processed] {
            seen[processed] = true
            result = append(result, processed)
        }
    }
    return result
}
```

Tasks:
1. Write a benchmark for this function.
2. Profile with `go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof`.
3. Analyze with `go tool pprof cpu.prof` and `go tool pprof mem.prof`.
4. Optimize and measure improvement with `benchstat`.

### Exercise 2: Benchmark and Compare Serialization Approaches

Write benchmarks comparing three ways to serialize a struct:
1. `encoding/json`
2. `encoding/gob`
3. Manual byte serialization with `binary.Write`

Use `b.ReportAllocs()` and `-count=10` with `benchstat`.

### Exercise 3: Find and Fix a Goroutine Leak

Write a program that intentionally leaks goroutines (e.g., goroutines blocked on channels that are never closed). Use `pprof`'s goroutine profile to detect the leak and fix it.

### Exercise 4: Escape Analysis Investigation

Write three versions of a function that returns data:
1. Returns a pointer to a struct (escapes).
2. Returns a struct by value (may not escape).
3. Accepts a pointer parameter and fills it in (caller controls allocation).

Use `go build -gcflags='-m'` to verify which variables escape. Benchmark all three.

### Exercise 5: GC Tuning Experiment

Write a program that allocates many short-lived objects (simulating an HTTP server processing requests). Measure with `GODEBUG=gctrace=1`. Then:
1. Try `GOGC=200` and `GOGC=50` -- observe CPU vs memory tradeoff.
2. Try `GOMEMLIMIT=100MiB` with `GOGC=off`.
3. Measure throughput and latency under each configuration.

### Exercise 6: Trace a Concurrent Pipeline

Build a three-stage concurrent pipeline (producer -> processor -> writer). Add `trace.WithRegion()` annotations. Capture a trace and analyze it with `go tool trace` to identify:
- Goroutine scheduling delays.
- Channel blocking patterns.
- How work is distributed across P's.

### Exercise 7: Optimize Struct Layout

Given this struct, reorder the fields for minimal size. Verify with `unsafe.Sizeof`:

```go
type Record struct {
    Active    bool
    Score     float64
    ID        int32
    Name      string
    Verified  bool
    Timestamp int64
    Category  uint8
    Tags      []string
    Count     int16
}
```

Calculate the size before and after reordering. What is the memory savings for 10 million records?

### Exercise 8: HTTP Client Performance

Write a program that makes 10,000 HTTP GET requests to a local test server. Measure performance with:
1. Default `http.Client` (no tuning).
2. Tuned `http.Transport` (connection pooling, keep-alive).
3. HTTP/2 enabled.

Use `go tool trace` to visualize the network activity patterns.

---

## Further Reading

- [Go Blog: Profiling Go Programs](https://go.dev/blog/pprof)
- [Go Blog: A Guide to the Go Garbage Collector](https://tip.golang.org/doc/gc-guide)
- [runtime/pprof package documentation](https://pkg.go.dev/runtime/pprof)
- [net/http/pprof package documentation](https://pkg.go.dev/net/http/pprof)
- [Go Blog: Diagnostics](https://go.dev/doc/diagnostics)
- [Dave Cheney: High Performance Go Workshop](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html)
- [Effective Go: Concurrency](https://go.dev/doc/effective_go#concurrency)
- [benchstat documentation](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat)
