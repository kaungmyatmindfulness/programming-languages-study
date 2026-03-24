# Chapter 39: Batch & Stream Processing

> **Level: Data-Intensive Systems** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 15 (Generics), 16 (Context), 17 (Advanced Concurrency), 13 (File I/O & CLI Tools)**

This chapter implements the core ideas from Chapters 10 and 11 of *Designing Data-Intensive Applications* by Martin Kleppmann -- batch processing and stream processing -- entirely in Go. These two processing paradigms form the backbone of every data pipeline, analytics system, log aggregator, and event-driven architecture in production today.

Batch processing operates on a bounded, finite dataset. You know how much data there is before you start. MapReduce, ETL jobs, nightly report generation -- these are batch. Stream processing operates on an unbounded, continuously arriving dataset. You never know when (or if) the data will stop. Kafka consumers, real-time dashboards, CDC pipelines -- these are stream.

Go is exceptionally well suited to both paradigms. Its goroutines model parallel batch workers naturally. Its channels model event streams with built-in backpressure. Its `io.Reader` and `io.Writer` interfaces embody the Unix pipeline philosophy that underpins all of this.

By the end of this chapter, you will have built:
- A complete MapReduce framework from scratch
- An event sourcing system with CQRS
- A generic stream processing library with windowing
- A real-world log processing pipeline

---

## Table of Contents

1. [Batch vs Stream Processing](#1-batch-vs-stream-processing)
2. [Unix Philosophy in Go](#2-unix-philosophy-in-go)
3. [MapReduce in Go](#3-mapreduce-in-go)
4. [Word Count Example](#4-word-count-example)
5. [Distributed MapReduce](#5-distributed-mapreduce)
6. [Beyond MapReduce: Dataflow Engines](#6-beyond-mapreduce-dataflow-engines)
7. [Event Sourcing](#7-event-sourcing)
8. [CQRS (Command Query Responsibility Segregation)](#8-cqrs-command-query-responsibility-segregation)
9. [Stream Processing Fundamentals](#9-stream-processing-fundamentals)
10. [Building a Stream Processor in Go](#10-building-a-stream-processor-in-go)
11. [Windowing Strategies](#11-windowing-strategies)
12. [Message Queues and Go](#12-message-queues-and-go)
13. [Change Data Capture (CDC)](#13-change-data-capture-cdc)
14. [Backpressure](#14-backpressure)
15. [Real-World Example: Log Processing Pipeline](#15-real-world-example-log-processing-pipeline)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. Batch vs Stream Processing

### The Two Paradigms

Every data processing system falls somewhere on a spectrum between two extremes:

| Property | Batch Processing | Stream Processing |
|----------|-----------------|-------------------|
| **Input** | Bounded (finite dataset) | Unbounded (continuous flow) |
| **Timing** | Run periodically (hourly, daily) | Run continuously |
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Throughput** | Very high (process everything at once) | Moderate (process as it arrives) |
| **Fault tolerance** | Restart the whole job | Checkpoint and resume |
| **State** | Read input, compute, write output | Maintain running state |
| **Examples** | MapReduce, Spark, nightly ETL | Kafka Streams, Flink, Go channels |

### When to Use Which

**Use batch processing when:**
- Data is already stored and you need to process all of it
- Latency of minutes or hours is acceptable
- You need maximum throughput (process terabytes)
- Jobs run on a schedule (daily reports, weekly aggregations)
- You can afford to re-process the entire input on failure

**Use stream processing when:**
- Data arrives continuously and you need near-real-time results
- Low latency is critical (fraud detection, monitoring alerts)
- You want incremental updates rather than full recomputation
- The input is conceptually infinite (user clicks, sensor readings, logs)

### The Unified View

Kleppmann's key insight: batch processing is really just stream processing over a bounded stream. If you build a good stream processor, you can run it over a file (batch) or over a live feed (stream) with the same code. Go's `io.Reader` interface captures this perfectly -- it does not care whether the bytes come from a file, a network socket, or a channel.

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"strings"
	"time"
)

// ProcessLines demonstrates the unified view: the same processing logic
// works on both bounded (file) and unbounded (channel-fed reader) input.
func ProcessLines(r io.Reader, process func(string)) error {
	scanner := bufio.NewScanner(r)
	for scanner.Scan() {
		process(scanner.Text())
	}
	return scanner.Err()
}

func main() {
	// === Batch mode: process a fixed string (simulating a file) ===
	fmt.Println("=== Batch Mode ===")
	batchInput := "apple\nbanana\napple\ncherry\nbanana\napple\n"
	counts := make(map[string]int)

	err := ProcessLines(strings.NewReader(batchInput), func(line string) {
		counts[line]++
	})
	if err != nil {
		fmt.Fprintf(os.Stderr, "batch error: %v\n", err)
		return
	}
	for word, count := range counts {
		fmt.Printf("  %s: %d\n", word, count)
	}

	// === Stream mode: process data arriving over time ===
	fmt.Println("\n=== Stream Mode ===")
	pr, pw := io.Pipe()

	// Simulate a producer that sends data over time
	go func() {
		defer pw.Close()
		events := []string{"login:user1", "click:user2", "login:user3", "purchase:user1", "click:user1"}
		for _, event := range events {
			time.Sleep(100 * time.Millisecond) // simulate real-time arrival
			fmt.Fprintln(pw, event)
		}
	}()

	// Same ProcessLines function, now on an unbounded (pipe) source
	streamCounts := make(map[string]int)
	err = ProcessLines(pr, func(line string) {
		parts := strings.SplitN(line, ":", 2)
		if len(parts) == 2 {
			eventType := parts[0]
			streamCounts[eventType]++
			fmt.Printf("  Received %s (total %s events: %d)\n", line, eventType, streamCounts[eventType])
		}
	})
	if err != nil {
		fmt.Fprintf(os.Stderr, "stream error: %v\n", err)
	}

	fmt.Println("\nFinal stream counts:")
	for eventType, count := range streamCounts {
		fmt.Printf("  %s: %d\n", eventType, count)
	}
}
```

---

## 2. Unix Philosophy in Go

### Composable Tools

The Unix philosophy -- small programs that do one thing well, connected by pipes -- is the original data processing framework. Doug McIlroy's summary: *"Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."*

Go's `io.Reader` and `io.Writer` interfaces are the programmatic equivalent of Unix pipes. Every Go function that accepts a Reader and returns results to a Writer is a composable pipeline stage.

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"regexp"
	"sort"
	"strings"
)

// Grep filters lines matching a pattern.
// This is equivalent to the Unix grep command.
func Grep(pattern string, r io.Reader, w io.Writer) error {
	re, err := regexp.Compile(pattern)
	if err != nil {
		return fmt.Errorf("invalid pattern %q: %w", pattern, err)
	}

	scanner := bufio.NewScanner(r)
	for scanner.Scan() {
		line := scanner.Text()
		if re.MatchString(line) {
			fmt.Fprintln(w, line)
		}
	}
	return scanner.Err()
}

// Sort reads all lines, sorts them, and writes them out.
// This is equivalent to the Unix sort command.
func Sort(r io.Reader, w io.Writer) error {
	scanner := bufio.NewScanner(r)
	var lines []string
	for scanner.Scan() {
		lines = append(lines, scanner.Text())
	}
	if err := scanner.Err(); err != nil {
		return err
	}

	sort.Strings(lines)
	for _, line := range lines {
		fmt.Fprintln(w, line)
	}
	return nil
}

// Uniq removes consecutive duplicate lines.
// This is equivalent to the Unix uniq command.
func Uniq(r io.Reader, w io.Writer) error {
	scanner := bufio.NewScanner(r)
	var prev string
	first := true
	for scanner.Scan() {
		line := scanner.Text()
		if first || line != prev {
			fmt.Fprintln(w, line)
			prev = line
			first = false
		}
	}
	return scanner.Err()
}

// UniqCount removes consecutive duplicates and counts occurrences.
// This is equivalent to uniq -c.
func UniqCount(r io.Reader, w io.Writer) error {
	scanner := bufio.NewScanner(r)
	var prev string
	count := 0
	first := true

	flush := func() {
		if !first {
			fmt.Fprintf(w, "%7d %s\n", count, prev)
		}
	}

	for scanner.Scan() {
		line := scanner.Text()
		if first || line != prev {
			flush()
			prev = line
			count = 1
			first = false
		} else {
			count++
		}
	}
	flush()
	return scanner.Err()
}

// Pipeline connects stages together using io.Pipe.
// Each stage runs in its own goroutine, just like Unix pipeline stages
// run as separate processes.
type PipelineStage func(r io.Reader, w io.Writer) error

func Pipeline(input io.Reader, output io.Writer, stages ...PipelineStage) error {
	if len(stages) == 0 {
		_, err := io.Copy(output, input)
		return err
	}

	// For a single stage, run directly
	if len(stages) == 1 {
		return stages[0](input, output)
	}

	// For multiple stages, chain them with pipes
	errs := make(chan error, len(stages))
	reader := input

	// Create pipes between stages
	var pipes []*io.PipeWriter
	for i := 0; i < len(stages)-1; i++ {
		pr, pw := io.Pipe()
		pipes = append(pipes, pw)

		currentReader := reader
		currentWriter := pw
		stage := stages[i]

		go func() {
			err := stage(currentReader, currentWriter)
			currentWriter.CloseWithError(err)
			errs <- err
		}()

		reader = pr
	}

	// Run the last stage in the current goroutine, writing to output
	lastErr := stages[len(stages)-1](reader, output)

	// Collect errors from goroutines
	for i := 0; i < len(stages)-1; i++ {
		if err := <-errs; err != nil && lastErr == nil {
			lastErr = err
		}
	}

	return lastErr
}

func main() {
	// Input data: web server access log lines
	input := `192.168.1.1 GET /api/users 200
192.168.1.2 GET /api/products 200
192.168.1.1 POST /api/users 201
10.0.0.5 GET /api/users 200
192.168.1.2 GET /api/users 404
10.0.0.5 DELETE /api/users/5 403
192.168.1.1 GET /api/users 200
10.0.0.5 GET /api/products 200
192.168.1.1 GET /api/users 200
192.168.1.2 POST /api/orders 201
`

	fmt.Println("=== Pipeline: grep '/api/users' | sort | uniq -c ===")
	fmt.Println()

	// Build the pipeline: grep for /api/users, sort, then count unique lines.
	// This is exactly: grep '/api/users' access.log | sort | uniq -c
	err := Pipeline(
		strings.NewReader(input),
		os.Stdout,
		func(r io.Reader, w io.Writer) error {
			return Grep("/api/users", r, w)
		},
		Sort,
		UniqCount,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "pipeline error: %v\n", err)
		os.Exit(1)
	}

	fmt.Println()
	fmt.Println("=== Pipeline: grep 'GET' | sort | uniq ===")
	fmt.Println()

	err = Pipeline(
		strings.NewReader(input),
		os.Stdout,
		func(r io.Reader, w io.Writer) error {
			return Grep("GET", r, w)
		},
		Sort,
		Uniq,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "pipeline error: %v\n", err)
		os.Exit(1)
	}
}
```

### Why This Matters

The Unix pipeline model is the conceptual ancestor of every modern data processing framework:
- **MapReduce** is `map | shuffle | reduce`
- **Kafka Streams** is an unbounded Unix pipe
- **Spark** is a lazily evaluated pipeline DAG

Go's interfaces make this composition natural. Any function that reads from `io.Reader` and writes to `io.Writer` can be a pipeline stage. Goroutines provide the concurrency that Unix processes provide in the shell.

---

## 3. MapReduce in Go

### The MapReduce Model

MapReduce, popularized by Google's 2004 paper, is a batch processing framework built on two user-defined functions:

1. **Map**: Takes one input record and emits zero or more key-value pairs
2. **Reduce**: Takes a key and all values associated with that key, and combines them into a result

Between Map and Reduce sits a **Shuffle** phase (handled by the framework) that groups all values by key.

```
Input → [Map] → [(key, value), ...] → [Shuffle/Group by Key] → [Reduce] → Output
```

### Complete Implementation

```go
package main

import (
	"fmt"
	"hash/fnv"
	"sort"
	"sync"
)

// KeyValue represents a single key-value pair emitted by Map or produced by Reduce.
type KeyValue struct {
	Key   string
	Value string
}

// MapFunc takes an input key-value pair and returns zero or more intermediate key-value pairs.
type MapFunc func(key, value string) []KeyValue

// ReduceFunc takes a key and all values associated with that key, and produces a single result.
type ReduceFunc func(key string, values []string) string

// MapReduceJob holds the configuration for a MapReduce computation.
type MapReduceJob struct {
	MapFn      MapFunc
	ReduceFn   ReduceFunc
	NumWorkers int // Number of parallel workers for each phase
}

// NewMapReduceJob creates a new MapReduce job.
func NewMapReduceJob(mapFn MapFunc, reduceFn ReduceFunc, workers int) *MapReduceJob {
	if workers < 1 {
		workers = 1
	}
	return &MapReduceJob{
		MapFn:      mapFn,
		ReduceFn:   reduceFn,
		NumWorkers: workers,
	}
}

// Run executes the full MapReduce pipeline on the given input.
func (j *MapReduceJob) Run(input []KeyValue) []KeyValue {
	// === Phase 1: Map (parallel) ===
	intermediate := j.mapPhase(input)

	// === Phase 2: Shuffle (group by key) ===
	grouped := j.shufflePhase(intermediate)

	// === Phase 3: Reduce (parallel) ===
	results := j.reducePhase(grouped)

	return results
}

// mapPhase distributes input records across workers and runs the map function.
func (j *MapReduceJob) mapPhase(input []KeyValue) []KeyValue {
	var (
		mu     sync.Mutex
		result []KeyValue
		wg     sync.WaitGroup
	)

	// Split input into chunks for parallel processing
	chunks := splitInput(input, j.NumWorkers)

	for _, chunk := range chunks {
		wg.Add(1)
		go func(records []KeyValue) {
			defer wg.Done()
			var local []KeyValue
			for _, record := range records {
				kvs := j.MapFn(record.Key, record.Value)
				local = append(local, kvs...)
			}
			mu.Lock()
			result = append(result, local...)
			mu.Unlock()
		}(chunk)
	}

	wg.Wait()
	return result
}

// shufflePhase groups intermediate key-value pairs by key.
func (j *MapReduceJob) shufflePhase(intermediate []KeyValue) map[string][]string {
	grouped := make(map[string][]string)
	for _, kv := range intermediate {
		grouped[kv.Key] = append(grouped[kv.Key], kv.Value)
	}
	return grouped
}

// reducePhase runs the reduce function on each group in parallel.
func (j *MapReduceJob) reducePhase(grouped map[string][]string) []KeyValue {
	// Convert map to a slice of work items so we can distribute them
	type reduceWork struct {
		key    string
		values []string
	}

	var work []reduceWork
	for k, v := range grouped {
		work = append(work, reduceWork{key: k, values: v})
	}

	var (
		mu     sync.Mutex
		result []KeyValue
		wg     sync.WaitGroup
	)

	// Use a worker pool for reduce
	workCh := make(chan reduceWork, len(work))
	for _, w := range work {
		workCh <- w
	}
	close(workCh)

	numReducers := j.NumWorkers
	if numReducers > len(work) {
		numReducers = len(work)
	}

	for i := 0; i < numReducers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			var local []KeyValue
			for w := range workCh {
				reduced := j.ReduceFn(w.key, w.values)
				local = append(local, KeyValue{Key: w.key, Value: reduced})
			}
			mu.Lock()
			result = append(result, local...)
			mu.Unlock()
		}()
	}

	wg.Wait()

	// Sort by key for deterministic output
	sort.Slice(result, func(i, j int) bool {
		return result[i].Key < result[j].Key
	})

	return result
}

// splitInput divides input records into roughly equal chunks.
func splitInput(input []KeyValue, n int) [][]KeyValue {
	if n <= 0 {
		n = 1
	}
	chunks := make([][]KeyValue, n)
	for i, kv := range input {
		chunks[i%n] = append(chunks[i%n], kv)
	}
	// Remove empty chunks
	var result [][]KeyValue
	for _, chunk := range chunks {
		if len(chunk) > 0 {
			result = append(result, chunk)
		}
	}
	return result
}

// hashKey returns a consistent hash for partitioning keys across reducers.
func hashKey(key string, numPartitions int) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32()) % numPartitions
}

func main() {
	// === Example: URL access counting ===
	fmt.Println("=== URL Access Count ===")

	// Simulate web server log entries
	logEntries := []KeyValue{
		{Key: "log1", Value: "GET /api/users"},
		{Key: "log2", Value: "GET /api/products"},
		{Key: "log3", Value: "GET /api/users"},
		{Key: "log4", Value: "POST /api/orders"},
		{Key: "log5", Value: "GET /api/users"},
		{Key: "log6", Value: "GET /api/products"},
		{Key: "log7", Value: "DELETE /api/users/5"},
		{Key: "log8", Value: "GET /api/users"},
		{Key: "log9", Value: "POST /api/orders"},
		{Key: "log10", Value: "GET /api/products"},
	}

	job := NewMapReduceJob(
		// Map: emit (url, "1") for each log entry
		func(key, value string) []KeyValue {
			return []KeyValue{{Key: value, Value: "1"}}
		},
		// Reduce: count the number of "1"s for each URL
		func(key string, values []string) string {
			return fmt.Sprintf("%d", len(values))
		},
		4, // 4 workers
	)

	results := job.Run(logEntries)
	for _, kv := range results {
		fmt.Printf("  %s => %s requests\n", kv.Key, kv.Value)
	}

	// === Example: Inverted index ===
	fmt.Println("\n=== Inverted Index ===")

	documents := []KeyValue{
		{Key: "doc1", Value: "the cat sat on the mat"},
		{Key: "doc2", Value: "the dog sat on the log"},
		{Key: "doc3", Value: "the cat chased the dog"},
	}

	invertedIndexJob := NewMapReduceJob(
		// Map: for each word in the document, emit (word, docID)
		func(docID, content string) []KeyValue {
			seen := make(map[string]bool) // deduplicate within same doc
			var result []KeyValue
			for _, word := range splitWords(content) {
				if !seen[word] {
					result = append(result, KeyValue{Key: word, Value: docID})
					seen[word] = true
				}
			}
			return result
		},
		// Reduce: collect all document IDs for each word
		func(word string, docIDs []string) string {
			sort.Strings(docIDs)
			return fmt.Sprintf("[%s]", joinStrings(docIDs, ", "))
		},
		2,
	)

	indexResults := invertedIndexJob.Run(documents)
	for _, kv := range indexResults {
		fmt.Printf("  %-10s => %s\n", kv.Key, kv.Value)
	}
}

func splitWords(s string) []string {
	var words []string
	word := ""
	for _, ch := range s {
		if ch == ' ' || ch == '\t' || ch == '\n' {
			if word != "" {
				words = append(words, word)
				word = ""
			}
		} else {
			word += string(ch)
		}
	}
	if word != "" {
		words = append(words, word)
	}
	return words
}

func joinStrings(parts []string, sep string) string {
	result := ""
	for i, p := range parts {
		if i > 0 {
			result += sep
		}
		result += p
	}
	return result
}
```

---

## 4. Word Count Example

### The "Hello World" of MapReduce

Word count is the canonical MapReduce example. Let us implement it three ways: naive, MapReduce, and optimized Go.

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"runtime"
	"sort"
	"strings"
	"sync"
	"time"
)

// === Approach 1: Naive single-threaded word count ===

func naiveWordCount(text string) map[string]int {
	counts := make(map[string]int)
	scanner := bufio.NewScanner(strings.NewReader(text))
	scanner.Split(bufio.ScanWords)
	for scanner.Scan() {
		word := strings.ToLower(scanner.Text())
		counts[word]++
	}
	return counts
}

// === Approach 2: MapReduce word count ===

type KeyValue struct {
	Key   string
	Value string
}

func mapReduceWordCount(text string, numWorkers int) map[string]int {
	// Split text into chunks (one per line for simplicity)
	lines := strings.Split(text, "\n")

	// Map phase: parallel
	type mapResult struct {
		kvs []KeyValue
	}

	chunks := make([][]string, numWorkers)
	for i, line := range lines {
		chunks[i%numWorkers] = append(chunks[i%numWorkers], line)
	}

	var wg sync.WaitGroup
	mapResults := make([][]KeyValue, numWorkers)

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			var kvs []KeyValue
			for _, line := range chunks[workerID] {
				words := strings.Fields(line)
				for _, word := range words {
					word = strings.ToLower(word)
					kvs = append(kvs, KeyValue{Key: word, Value: "1"})
				}
			}
			mapResults[workerID] = kvs
		}(i)
	}
	wg.Wait()

	// Shuffle phase: group by key
	grouped := make(map[string]int)
	for _, kvs := range mapResults {
		for _, kv := range kvs {
			grouped[kv.Key]++
		}
	}

	return grouped
}

// === Approach 3: Optimized concurrent word count ===
// Uses per-worker local maps to minimize contention, then merges.

func optimizedWordCount(r io.Reader, numWorkers int) map[string]int {
	scanner := bufio.NewScanner(r)
	lines := make(chan string, 1000)

	// Producer: read lines into channel
	go func() {
		defer close(lines)
		for scanner.Scan() {
			lines <- scanner.Text()
		}
	}()

	// Workers: each maintains a local count map (zero contention)
	localCounts := make([]map[string]int, numWorkers)
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		localCounts[i] = make(map[string]int)
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			local := localCounts[workerID]
			for line := range lines {
				for _, word := range strings.Fields(line) {
					local[strings.ToLower(word)]++
				}
			}
		}(i)
	}
	wg.Wait()

	// Merge phase: combine all local maps
	merged := make(map[string]int)
	for _, local := range localCounts {
		for word, count := range local {
			merged[word] += count
		}
	}

	return merged
}

func main() {
	// Generate a large-ish text for benchmarking
	var sb strings.Builder
	paragraph := "the quick brown fox jumps over the lazy dog and the fox ran away from the dog while the brown cat watched from the fence and then the dog chased the cat across the green field"
	for i := 0; i < 10000; i++ {
		sb.WriteString(paragraph)
		sb.WriteString("\n")
	}
	text := sb.String()

	// === Approach 1: Naive ===
	start := time.Now()
	result1 := naiveWordCount(text)
	d1 := time.Since(start)
	fmt.Printf("Naive:      %d unique words, took %v\n", len(result1), d1)

	// === Approach 2: MapReduce ===
	numCPU := runtime.NumCPU()
	start = time.Now()
	result2 := mapReduceWordCount(text, numCPU)
	d2 := time.Since(start)
	fmt.Printf("MapReduce:  %d unique words, took %v (%d workers)\n", len(result2), d2, numCPU)

	// === Approach 3: Optimized ===
	start = time.Now()
	result3 := optimizedWordCount(strings.NewReader(text), numCPU)
	d3 := time.Since(start)
	fmt.Printf("Optimized:  %d unique words, took %v (%d workers)\n", len(result3), d3, numCPU)

	// Print top 10 words from the optimized result
	type wordCount struct {
		word  string
		count int
	}
	var wcs []wordCount
	for w, c := range result3 {
		wcs = append(wcs, wordCount{w, c})
	}
	sort.Slice(wcs, func(i, j int) bool {
		return wcs[i].count > wcs[j].count
	})

	fmt.Println("\nTop 10 words:")
	for i := 0; i < 10 && i < len(wcs); i++ {
		fmt.Printf("  %-10s %d\n", wcs[i].word, wcs[i].count)
	}
}
```

### Key Observations

1. **Naive** is simple but single-threaded. For small data, this is fastest due to zero overhead.
2. **MapReduce** adds overhead from goroutine creation and intermediate KeyValue allocation. The overhead is justified only for large data.
3. **Optimized** avoids the intermediate KeyValue pairs entirely. Each worker maintains a local `map[string]int`, eliminating all contention. The merge phase is O(unique_words * num_workers), which is fast.

The lesson: MapReduce is a programming model, not necessarily an optimization. For in-process work, direct concurrent approaches often outperform strict MapReduce. MapReduce shines when data is distributed across machines and you need a framework to coordinate.

---

## 5. Distributed MapReduce

### Simulating Distribution

In a real distributed MapReduce (like Hadoop), input is split across machines, map tasks run on data-local nodes, intermediate data is shuffled over the network, and reduce tasks run on assigned nodes. We simulate this with goroutines as "nodes" and channels as "network."

```go
package main

import (
	"fmt"
	"hash/fnv"
	"sort"
	"strings"
	"sync"
	"time"
)

// KeyValue represents a key-value pair.
type KeyValue struct {
	Key   string
	Value string
}

// InputSplit represents a chunk of input assigned to one mapper.
type InputSplit struct {
	ID      int
	Records []KeyValue
}

// TaskStatus tracks the status of a map or reduce task.
type TaskStatus int

const (
	TaskPending TaskStatus = iota
	TaskRunning
	TaskCompleted
	TaskFailed
)

// WorkerInfo tracks a worker's status.
type WorkerInfo struct {
	ID     int
	Status string
	TaskID int
}

// Coordinator manages the MapReduce job.
type Coordinator struct {
	mu sync.Mutex

	mapFunc    func(key, value string) []KeyValue
	reduceFunc func(key string, values []string) string

	// Input splits for mappers
	inputSplits []InputSplit

	// Number of reduce partitions
	numReducers int

	// Intermediate data: partitioned by reducer ID
	// intermediate[reducerID] = []KeyValue
	intermediate [][]KeyValue
	intermediateMu sync.Mutex

	// Final results
	results []KeyValue

	// Worker tracking
	workers map[int]*WorkerInfo
}

// NewCoordinator creates a new coordinator.
func NewCoordinator(
	mapFn func(key, value string) []KeyValue,
	reduceFn func(key string, values []string) string,
	input []KeyValue,
	numMappers, numReducers int,
) *Coordinator {
	// Split input into chunks
	splits := make([]InputSplit, numMappers)
	for i, kv := range input {
		splitID := i % numMappers
		splits[splitID].ID = splitID
		splits[splitID].Records = append(splits[splitID].Records, kv)
	}

	return &Coordinator{
		mapFunc:      mapFn,
		reduceFunc:   reduceFn,
		inputSplits:  splits,
		numReducers:  numReducers,
		intermediate: make([][]KeyValue, numReducers),
		workers:      make(map[int]*WorkerInfo),
	}
}

// Run executes the distributed MapReduce job.
func (c *Coordinator) Run() []KeyValue {
	fmt.Println("[Coordinator] Starting MapReduce job")
	fmt.Printf("[Coordinator] %d input splits, %d reducers\n", len(c.inputSplits), c.numReducers)

	// === Map Phase ===
	fmt.Println("\n[Coordinator] === MAP PHASE ===")
	start := time.Now()
	c.runMapPhase()
	fmt.Printf("[Coordinator] Map phase complete in %v\n", time.Since(start))

	// === Shuffle Phase (already done by partitioning in map) ===
	fmt.Println("\n[Coordinator] === SHUFFLE PHASE ===")
	start = time.Now()
	grouped := c.shufflePhase()
	fmt.Printf("[Coordinator] Shuffle complete: %d unique keys distributed to %d reducers\n",
		countKeys(grouped), c.numReducers)
	fmt.Printf("[Coordinator] Shuffle phase complete in %v\n", time.Since(start))

	// === Reduce Phase ===
	fmt.Println("\n[Coordinator] === REDUCE PHASE ===")
	start = time.Now()
	c.results = c.runReducePhase(grouped)
	fmt.Printf("[Coordinator] Reduce phase complete in %v\n", time.Since(start))

	return c.results
}

// runMapPhase dispatches map tasks to workers.
func (c *Coordinator) runMapPhase() {
	var wg sync.WaitGroup

	for _, split := range c.inputSplits {
		wg.Add(1)
		go func(s InputSplit) {
			defer wg.Done()

			workerID := s.ID
			fmt.Printf("  [Mapper %d] Processing %d records\n", workerID, len(s.Records))

			// Simulate network delay and processing time
			time.Sleep(10 * time.Millisecond)

			// Run map function on each record
			var mapped []KeyValue
			for _, record := range s.Records {
				kvs := c.mapFunc(record.Key, record.Value)
				mapped = append(mapped, kvs...)
			}

			// Partition output by key hash (this is the shuffle write)
			partitions := make([][]KeyValue, c.numReducers)
			for _, kv := range mapped {
				partition := hashKey(kv.Key, c.numReducers)
				partitions[partition] = append(partitions[partition], kv)
			}

			// "Send" partitions to reducers (simulate network transfer)
			c.intermediateMu.Lock()
			for i, part := range partitions {
				c.intermediate[i] = append(c.intermediate[i], part...)
			}
			c.intermediateMu.Unlock()

			fmt.Printf("  [Mapper %d] Emitted %d intermediate pairs\n", workerID, len(mapped))
		}(split)
	}

	wg.Wait()
}

// shufflePhase groups intermediate results by key within each partition.
func (c *Coordinator) shufflePhase() []map[string][]string {
	grouped := make([]map[string][]string, c.numReducers)

	for i := 0; i < c.numReducers; i++ {
		grouped[i] = make(map[string][]string)
		for _, kv := range c.intermediate[i] {
			grouped[i][kv.Key] = append(grouped[i][kv.Key], kv.Value)
		}
		fmt.Printf("  [Shuffle] Reducer %d has %d unique keys\n", i, len(grouped[i]))
	}

	return grouped
}

// runReducePhase dispatches reduce tasks to workers.
func (c *Coordinator) runReducePhase(grouped []map[string][]string) []KeyValue {
	var (
		mu      sync.Mutex
		results []KeyValue
		wg      sync.WaitGroup
	)

	for i, partition := range grouped {
		wg.Add(1)
		go func(reducerID int, data map[string][]string) {
			defer wg.Done()

			fmt.Printf("  [Reducer %d] Processing %d keys\n", reducerID, len(data))

			// Simulate processing time
			time.Sleep(10 * time.Millisecond)

			var local []KeyValue
			for key, values := range data {
				result := c.reduceFunc(key, values)
				local = append(local, KeyValue{Key: key, Value: result})
			}

			mu.Lock()
			results = append(results, local...)
			mu.Unlock()

			fmt.Printf("  [Reducer %d] Produced %d results\n", reducerID, len(local))
		}(i, partition)
	}

	wg.Wait()

	sort.Slice(results, func(i, j int) bool {
		return results[i].Key < results[j].Key
	})

	return results
}

func hashKey(key string, numPartitions int) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32()) % numPartitions
}

func countKeys(grouped []map[string][]string) int {
	seen := make(map[string]bool)
	for _, g := range grouped {
		for k := range g {
			seen[k] = true
		}
	}
	return len(seen)
}

func main() {
	// Simulated log data
	documents := []KeyValue{
		{Key: "server1.log", Value: "error timeout connecting to database"},
		{Key: "server2.log", Value: "error null pointer exception in handler"},
		{Key: "server3.log", Value: "warning high memory usage detected"},
		{Key: "server1.log", Value: "error timeout connecting to cache"},
		{Key: "server2.log", Value: "error database connection pool exhausted"},
		{Key: "server3.log", Value: "error timeout connecting to database"},
		{Key: "server1.log", Value: "warning disk space low on volume"},
		{Key: "server2.log", Value: "error null pointer exception in handler"},
		{Key: "server3.log", Value: "error timeout connecting to database"},
		{Key: "server1.log", Value: "info deployment completed successfully"},
	}

	coordinator := NewCoordinator(
		// Map: extract each word
		func(filename, line string) []KeyValue {
			var result []KeyValue
			for _, word := range strings.Fields(line) {
				result = append(result, KeyValue{Key: word, Value: "1"})
			}
			return result
		},
		// Reduce: count occurrences
		func(word string, counts []string) string {
			return fmt.Sprintf("%d", len(counts))
		},
		documents,
		3, // 3 mappers
		2, // 2 reducers
	)

	results := coordinator.Run()

	fmt.Println("\n=== Final Results (Word Counts) ===")
	for _, kv := range results {
		fmt.Printf("  %-20s %s\n", kv.Key, kv.Value)
	}
}
```

### Fault Tolerance

In a real distributed MapReduce:
- If a mapper fails, the coordinator re-assigns the input split to another worker.
- If a reducer fails, it can be re-run because the intermediate data is stored on the mappers' local disks.
- The coordinator tracks heartbeats from workers and reassigns tasks on timeout.

The key insight: **map tasks are idempotent** (they read input and produce output, no side effects), so re-running a failed map task is safe. Reduce tasks are also idempotent if the output is written atomically (write to a temp file, then rename).

---

## 6. Beyond MapReduce: Dataflow Engines

### The Problem with MapReduce

MapReduce materializes intermediate state to disk between every stage. If you chain multiple MapReduce jobs (which is common), each one writes its output, and the next one reads it. This is slow.

Modern dataflow engines (Spark, Flink, Dask) model computation as a **Directed Acyclic Graph (DAG)** of operators. Intermediate data flows through memory, and only spills to disk when necessary. Operators can be fused (combined) to avoid unnecessary materialization.

### DAG Pipeline Pattern in Go

```go
package main

import (
	"fmt"
	"strings"
	"sync"
)

// Record represents a data record flowing through the pipeline.
type Record map[string]interface{}

// Operator is a single stage in the dataflow DAG.
type Operator interface {
	Process(in <-chan Record) <-chan Record
	Name() string
}

// MapOperator applies a transformation to each record.
type MapOperator struct {
	name string
	fn   func(Record) Record
}

func (m *MapOperator) Name() string { return m.name }

func (m *MapOperator) Process(in <-chan Record) <-chan Record {
	out := make(chan Record, 100)
	go func() {
		defer close(out)
		for record := range in {
			if result := m.fn(record); result != nil {
				out <- result
			}
		}
	}()
	return out
}

// FilterOperator removes records that don't match a predicate.
type FilterOperator struct {
	name      string
	predicate func(Record) bool
}

func (f *FilterOperator) Name() string { return f.name }

func (f *FilterOperator) Process(in <-chan Record) <-chan Record {
	out := make(chan Record, 100)
	go func() {
		defer close(out)
		for record := range in {
			if f.predicate(record) {
				out <- record
			}
		}
	}()
	return out
}

// FlatMapOperator maps one record to zero or more records.
type FlatMapOperator struct {
	name string
	fn   func(Record) []Record
}

func (fm *FlatMapOperator) Name() string { return fm.name }

func (fm *FlatMapOperator) Process(in <-chan Record) <-chan Record {
	out := make(chan Record, 100)
	go func() {
		defer close(out)
		for record := range in {
			for _, r := range fm.fn(record) {
				out <- r
			}
		}
	}()
	return out
}

// AggregateOperator groups records by a key and applies an aggregation.
type AggregateOperator struct {
	name    string
	keyFn   func(Record) string
	aggFn   func(accumulated, incoming Record) Record
	initFn  func() Record
}

func (a *AggregateOperator) Name() string { return a.name }

func (a *AggregateOperator) Process(in <-chan Record) <-chan Record {
	out := make(chan Record, 100)
	go func() {
		defer close(out)
		groups := make(map[string]Record)
		for record := range in {
			key := a.keyFn(record)
			if _, ok := groups[key]; !ok {
				groups[key] = a.initFn()
			}
			groups[key] = a.aggFn(groups[key], record)
		}
		for _, agg := range groups {
			out <- agg
		}
	}()
	return out
}

// ParallelMapOperator runs a map function across multiple goroutines.
// This is operator fusion -- combining parallelism with transformation.
type ParallelMapOperator struct {
	name       string
	fn         func(Record) Record
	numWorkers int
}

func (p *ParallelMapOperator) Name() string { return p.name }

func (p *ParallelMapOperator) Process(in <-chan Record) <-chan Record {
	out := make(chan Record, 100)
	var wg sync.WaitGroup
	for i := 0; i < p.numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for record := range in {
				if result := p.fn(record); result != nil {
					out <- result
				}
			}
		}()
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

// DAG represents a dataflow computation as a chain of operators.
type DAG struct {
	operators []Operator
}

func NewDAG() *DAG {
	return &DAG{}
}

func (d *DAG) AddOperator(op Operator) *DAG {
	d.operators = append(d.operators, op)
	return d
}

func (d *DAG) Execute(source <-chan Record) <-chan Record {
	fmt.Printf("Executing DAG with %d operators:\n", len(d.operators))
	current := source
	for i, op := range d.operators {
		fmt.Printf("  Stage %d: %s\n", i+1, op.Name())
		current = op.Process(current)
	}
	return current
}

func main() {
	// === Build a data processing DAG ===
	// Pipeline: Parse log lines -> Extract fields -> Filter errors -> Count by category

	dag := NewDAG()

	// Stage 1: Parse raw log lines into records
	dag.AddOperator(&MapOperator{
		name: "ParseLogLine",
		fn: func(r Record) Record {
			line, ok := r["raw"].(string)
			if !ok {
				return nil
			}
			parts := strings.Fields(line)
			if len(parts) < 3 {
				return nil
			}
			return Record{
				"level":   parts[0],
				"service": parts[1],
				"message": strings.Join(parts[2:], " "),
			}
		},
	})

	// Stage 2: Filter to only error-level logs
	dag.AddOperator(&FilterOperator{
		name: "FilterErrors",
		predicate: func(r Record) bool {
			return r["level"] == "ERROR"
		},
	})

	// Stage 3: Categorize errors
	dag.AddOperator(&MapOperator{
		name: "CategorizeError",
		fn: func(r Record) Record {
			msg, _ := r["message"].(string)
			category := "other"
			switch {
			case strings.Contains(msg, "timeout"):
				category = "timeout"
			case strings.Contains(msg, "connection"):
				category = "connection"
			case strings.Contains(msg, "auth"):
				category = "authentication"
			case strings.Contains(msg, "null"):
				category = "null_pointer"
			}
			r["category"] = category
			return r
		},
	})

	// Stage 4: Aggregate by category
	dag.AddOperator(&AggregateOperator{
		name: "CountByCategory",
		keyFn: func(r Record) string {
			cat, _ := r["category"].(string)
			return cat
		},
		aggFn: func(acc, incoming Record) Record {
			count, _ := acc["count"].(int)
			acc["count"] = count + 1
			acc["category"] = incoming["category"]
			return acc
		},
		initFn: func() Record {
			return Record{"count": 0}
		},
	})

	// === Feed data ===
	source := make(chan Record, 100)
	go func() {
		defer close(source)
		logs := []string{
			"ERROR api-gateway connection refused to downstream service",
			"INFO api-gateway request completed in 45ms",
			"ERROR auth-service timeout waiting for database response",
			"ERROR api-gateway connection reset by peer",
			"WARN payment-service high latency detected",
			"ERROR auth-service auth token validation failed",
			"ERROR api-gateway timeout connecting to auth-service",
			"INFO user-service user created successfully",
			"ERROR payment-service null pointer in payment handler",
			"ERROR api-gateway connection refused to payment-service",
			"ERROR auth-service auth session expired unexpectedly",
			"ERROR api-gateway timeout on upstream health check",
		}
		for _, line := range logs {
			source <- Record{"raw": line}
		}
	}()

	// === Execute ===
	fmt.Println("=== Dataflow DAG Execution ===\n")
	output := dag.Execute(source)

	fmt.Println("\n=== Results: Error counts by category ===")
	for record := range output {
		fmt.Printf("  Category: %-20s Count: %d\n",
			record["category"], record["count"])
	}
}
```

### Advantages Over MapReduce

| Feature | MapReduce | Dataflow DAG |
|---------|-----------|--------------|
| **Intermediate storage** | Always disk | Memory (spill to disk if needed) |
| **Multi-stage jobs** | Separate jobs, each reads/writes disk | Single DAG, data flows in memory |
| **Operator fusion** | Not possible | Adjacent operators can be merged |
| **Latency** | High (disk I/O between stages) | Low (in-memory) |
| **Flexibility** | Map then Reduce | Any operator graph |

---

## 7. Event Sourcing

### Storing Events, Not State

Traditional systems store the current state of each entity. Event sourcing stores the **sequence of events** that led to the current state. The current state is derived by replaying all events.

This is the same principle as:
- A bank ledger (you store transactions, not just the balance)
- Version control (you store commits, not just the latest files)
- Database write-ahead logs (you store the writes, then apply them)

### Complete Event Sourcing Implementation

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"sync"
	"time"
)

// ============================================================
// Core Event Sourcing Types
// ============================================================

// Event is the fundamental unit of state change.
type Event struct {
	ID            string          `json:"id"`
	Type          string          `json:"type"`
	AggregateID   string          `json:"aggregate_id"`
	AggregateType string          `json:"aggregate_type"`
	Data          json.RawMessage `json:"data"`
	Timestamp     time.Time       `json:"timestamp"`
	Version       int             `json:"version"`
}

// Aggregate is an entity whose state is derived from events.
type Aggregate interface {
	Apply(event Event) error
	AggregateID() string
	AggregateType() string
}

// EventStore persists and retrieves events.
type EventStore struct {
	mu     sync.RWMutex
	events []Event
	// Index: aggregateID -> list of event indices
	byAggregate map[string][]int
	// Subscribers
	subscribers []chan Event
	subMu       sync.RWMutex
}

// NewEventStore creates a new in-memory event store.
func NewEventStore() *EventStore {
	return &EventStore{
		byAggregate: make(map[string][]int),
	}
}

// Append stores new events and notifies subscribers.
func (s *EventStore) Append(events ...Event) error {
	s.mu.Lock()

	for i := range events {
		events[i].ID = fmt.Sprintf("evt-%d", len(s.events)+1)
		events[i].Timestamp = time.Now()

		// Optimistic concurrency: check version
		existing := s.byAggregate[events[i].AggregateID]
		expectedVersion := len(existing)
		if events[i].Version != expectedVersion {
			s.mu.Unlock()
			return fmt.Errorf("concurrency conflict: expected version %d, got %d",
				expectedVersion, events[i].Version)
		}

		idx := len(s.events)
		s.events = append(s.events, events[i])
		s.byAggregate[events[i].AggregateID] = append(existing, idx)
	}

	s.mu.Unlock()

	// Notify subscribers (outside the lock)
	s.subMu.RLock()
	for _, sub := range s.subscribers {
		for _, event := range events {
			// Non-blocking send
			select {
			case sub <- event:
			default:
				// Subscriber is slow, skip (in production, you'd handle this better)
			}
		}
	}
	s.subMu.RUnlock()

	return nil
}

// GetEvents retrieves all events for an aggregate.
func (s *EventStore) GetEvents(aggregateID string) []Event {
	s.mu.RLock()
	defer s.mu.RUnlock()

	indices := s.byAggregate[aggregateID]
	events := make([]Event, len(indices))
	for i, idx := range indices {
		events[i] = s.events[idx]
	}
	return events
}

// GetAllEvents returns all events in order.
func (s *EventStore) GetAllEvents() []Event {
	s.mu.RLock()
	defer s.mu.RUnlock()

	result := make([]Event, len(s.events))
	copy(result, s.events)
	return result
}

// Subscribe returns a channel that receives new events.
func (s *EventStore) Subscribe() chan Event {
	ch := make(chan Event, 100)
	s.subMu.Lock()
	s.subscribers = append(s.subscribers, ch)
	s.subMu.Unlock()
	return ch
}

// Load reconstructs an aggregate's state by replaying its events.
func (s *EventStore) Load(agg Aggregate) error {
	events := s.GetEvents(agg.AggregateID())
	for _, event := range events {
		if err := agg.Apply(event); err != nil {
			return fmt.Errorf("applying event %s: %w", event.ID, err)
		}
	}
	return nil
}

// ============================================================
// Domain: Bank Account (the classic event sourcing example)
// ============================================================

// Event types
const (
	AccountOpened    = "AccountOpened"
	MoneyDeposited   = "MoneyDeposited"
	MoneyWithdrawn   = "MoneyWithdrawn"
	AccountClosed    = "AccountClosed"
)

// Event data structures
type AccountOpenedData struct {
	Owner          string `json:"owner"`
	InitialBalance int    `json:"initial_balance"` // in cents
}

type MoneyDepositedData struct {
	Amount      int    `json:"amount"`
	Description string `json:"description"`
}

type MoneyWithdrawnData struct {
	Amount      int    `json:"amount"`
	Description string `json:"description"`
}

type AccountClosedData struct {
	Reason string `json:"reason"`
}

// BankAccount is our aggregate.
type BankAccount struct {
	ID      string
	Owner   string
	Balance int  // in cents
	Closed  bool
	Version int
}

func NewBankAccount(id string) *BankAccount {
	return &BankAccount{ID: id}
}

func (a *BankAccount) AggregateID() string   { return a.ID }
func (a *BankAccount) AggregateType() string { return "BankAccount" }

// Apply updates the account state based on an event.
func (a *BankAccount) Apply(event Event) error {
	switch event.Type {
	case AccountOpened:
		var data AccountOpenedData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		a.Owner = data.Owner
		a.Balance = data.InitialBalance
		a.Closed = false

	case MoneyDeposited:
		var data MoneyDepositedData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		a.Balance += data.Amount

	case MoneyWithdrawn:
		var data MoneyWithdrawnData
		if err := json.Unmarshal(event.Data, &data); err != nil {
			return err
		}
		a.Balance -= data.Amount

	case AccountClosed:
		a.Closed = true

	default:
		return fmt.Errorf("unknown event type: %s", event.Type)
	}

	a.Version = event.Version + 1
	return nil
}

// ============================================================
// Command handlers (business logic that produces events)
// ============================================================

type BankService struct {
	store *EventStore
}

func NewBankService(store *EventStore) *BankService {
	return &BankService{store: store}
}

func (s *BankService) OpenAccount(accountID, owner string, initialBalance int) error {
	// Check if account already exists
	existing := s.store.GetEvents(accountID)
	if len(existing) > 0 {
		return errors.New("account already exists")
	}

	data, _ := json.Marshal(AccountOpenedData{
		Owner:          owner,
		InitialBalance: initialBalance,
	})

	return s.store.Append(Event{
		Type:          AccountOpened,
		AggregateID:   accountID,
		AggregateType: "BankAccount",
		Data:          data,
		Version:       0,
	})
}

func (s *BankService) Deposit(accountID string, amount int, description string) error {
	// Load current state
	account := NewBankAccount(accountID)
	if err := s.store.Load(account); err != nil {
		return err
	}
	if account.Closed {
		return errors.New("account is closed")
	}
	if amount <= 0 {
		return errors.New("deposit amount must be positive")
	}

	data, _ := json.Marshal(MoneyDepositedData{
		Amount:      amount,
		Description: description,
	})

	return s.store.Append(Event{
		Type:          MoneyDeposited,
		AggregateID:   accountID,
		AggregateType: "BankAccount",
		Data:          data,
		Version:       account.Version,
	})
}

func (s *BankService) Withdraw(accountID string, amount int, description string) error {
	account := NewBankAccount(accountID)
	if err := s.store.Load(account); err != nil {
		return err
	}
	if account.Closed {
		return errors.New("account is closed")
	}
	if amount <= 0 {
		return errors.New("withdrawal amount must be positive")
	}
	if account.Balance < amount {
		return fmt.Errorf("insufficient funds: balance=%d, requested=%d", account.Balance, amount)
	}

	data, _ := json.Marshal(MoneyWithdrawnData{
		Amount:      amount,
		Description: description,
	})

	return s.store.Append(Event{
		Type:          MoneyWithdrawn,
		AggregateID:   accountID,
		AggregateType: "BankAccount",
		Data:          data,
		Version:       account.Version,
	})
}

func (s *BankService) GetBalance(accountID string) (int, error) {
	account := NewBankAccount(accountID)
	if err := s.store.Load(account); err != nil {
		return 0, err
	}
	return account.Balance, nil
}

func main() {
	store := NewEventStore()
	bank := NewBankService(store)

	// === Perform operations ===
	fmt.Println("=== Bank Account Event Sourcing ===\n")

	if err := bank.OpenAccount("acc-001", "Alice", 100_00); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Println("Opened account for Alice with $100.00")

	if err := bank.Deposit("acc-001", 50_00, "Birthday gift"); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Println("Deposited $50.00 (Birthday gift)")

	if err := bank.Withdraw("acc-001", 30_00, "Groceries"); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Println("Withdrew $30.00 (Groceries)")

	if err := bank.Deposit("acc-001", 200_00, "Salary"); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Println("Deposited $200.00 (Salary)")

	if err := bank.Withdraw("acc-001", 75_00, "Electric bill"); err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Println("Withdrew $75.00 (Electric bill)")

	// === Check balance ===
	balance, _ := bank.GetBalance("acc-001")
	fmt.Printf("\nCurrent balance: $%.2f\n", float64(balance)/100)

	// === Show full event history ===
	fmt.Println("\n=== Full Event History ===")
	events := store.GetEvents("acc-001")
	for _, event := range events {
		fmt.Printf("  [v%d] %-18s %s\n", event.Version, event.Type, string(event.Data))
	}

	// === Demonstrate rebuild: recreate state from events ===
	fmt.Println("\n=== Rebuilding State From Events ===")
	rebuilt := NewBankAccount("acc-001")
	for _, event := range events {
		rebuilt.Apply(event)
		fmt.Printf("  After %-18s -> Balance: $%.2f\n", event.Type, float64(rebuilt.Balance)/100)
	}

	// === Try an invalid operation ===
	err := bank.Withdraw("acc-001", 999_00, "Too much")
	if err != nil {
		fmt.Printf("\nExpected error: %v\n", err)
	}
}
```

### Why Event Sourcing?

1. **Complete audit trail**: You never lose information. You can answer "what happened" at any point in time.
2. **Temporal queries**: You can rebuild the state as it was at any past moment by replaying events up to that timestamp.
3. **Debugging**: When something goes wrong, you can replay events to reproduce the exact sequence that caused the bug.
4. **Decoupling**: Other systems can subscribe to the event stream and build their own views of the data.
5. **Event replay**: You can add a new projection later and populate it by replaying historical events.

---

## 8. CQRS (Command Query Responsibility Segregation)

### Separating Reads and Writes

CQRS is the pattern of using different models for reading and writing data. The write model (command side) processes commands and produces events. The read model (query side) maintains materialized views optimized for queries.

This pairs naturally with event sourcing: commands produce events, and projectors consume those events to update read models.

```
Commands → [Command Handler] → [Event Store] → Events
                                                  ↓
                                            [Projector]
                                                  ↓
                                           [Read Model] ← Queries
```

### Complete CQRS Implementation

```go
package main

import (
	"encoding/json"
	"fmt"
	"sort"
	"sync"
	"time"
)

// ============================================================
// Event Infrastructure (shared with Event Sourcing)
// ============================================================

type Event struct {
	ID            string          `json:"id"`
	Type          string          `json:"type"`
	AggregateID   string          `json:"aggregate_id"`
	AggregateType string          `json:"aggregate_type"`
	Data          json.RawMessage `json:"data"`
	Timestamp     time.Time       `json:"timestamp"`
	Version       int             `json:"version"`
}

type EventStore struct {
	mu          sync.RWMutex
	events      []Event
	byAggregate map[string][]int
	subscribers []chan Event
	subMu       sync.RWMutex
}

func NewEventStore() *EventStore {
	return &EventStore{byAggregate: make(map[string][]int)}
}

func (s *EventStore) Append(events ...Event) error {
	s.mu.Lock()
	for i := range events {
		events[i].ID = fmt.Sprintf("evt-%d", len(s.events)+1)
		events[i].Timestamp = time.Now()
		idx := len(s.events)
		s.events = append(s.events, events[i])
		s.byAggregate[events[i].AggregateID] = append(
			s.byAggregate[events[i].AggregateID], idx)
	}
	s.mu.Unlock()

	s.subMu.RLock()
	for _, sub := range s.subscribers {
		for _, event := range events {
			select {
			case sub <- event:
			default:
			}
		}
	}
	s.subMu.RUnlock()
	return nil
}

func (s *EventStore) GetEvents(aggregateID string) []Event {
	s.mu.RLock()
	defer s.mu.RUnlock()
	indices := s.byAggregate[aggregateID]
	events := make([]Event, len(indices))
	for i, idx := range indices {
		events[i] = s.events[idx]
	}
	return events
}

func (s *EventStore) Subscribe() chan Event {
	ch := make(chan Event, 256)
	s.subMu.Lock()
	s.subscribers = append(s.subscribers, ch)
	s.subMu.Unlock()
	return ch
}

func (s *EventStore) AllEvents() []Event {
	s.mu.RLock()
	defer s.mu.RUnlock()
	result := make([]Event, len(s.events))
	copy(result, s.events)
	return result
}

// ============================================================
// Domain Events
// ============================================================

const (
	TaskCreated    = "TaskCreated"
	TaskAssigned   = "TaskAssigned"
	TaskCompleted  = "TaskCompleted"
	CommentAdded   = "CommentAdded"
)

type TaskCreatedData struct {
	Title       string `json:"title"`
	Description string `json:"description"`
	ProjectID   string `json:"project_id"`
	CreatedBy   string `json:"created_by"`
}

type TaskAssignedData struct {
	AssigneeID string `json:"assignee_id"`
}

type TaskCompletedData struct {
	CompletedBy string `json:"completed_by"`
}

type CommentAddedData struct {
	CommentID string `json:"comment_id"`
	Author    string `json:"author"`
	Body      string `json:"body"`
}

// ============================================================
// COMMAND SIDE: Handles writes
// ============================================================

type CommandHandler struct {
	store *EventStore
}

func NewCommandHandler(store *EventStore) *CommandHandler {
	return &CommandHandler{store: store}
}

func (h *CommandHandler) CreateTask(taskID, title, description, projectID, createdBy string) error {
	data, _ := json.Marshal(TaskCreatedData{
		Title:       title,
		Description: description,
		ProjectID:   projectID,
		CreatedBy:   createdBy,
	})
	return h.store.Append(Event{
		Type:          TaskCreated,
		AggregateID:   taskID,
		AggregateType: "Task",
		Data:          data,
		Version:       0,
	})
}

func (h *CommandHandler) AssignTask(taskID, assigneeID string) error {
	events := h.store.GetEvents(taskID)
	if len(events) == 0 {
		return fmt.Errorf("task %s not found", taskID)
	}
	data, _ := json.Marshal(TaskAssignedData{AssigneeID: assigneeID})
	return h.store.Append(Event{
		Type:          TaskAssigned,
		AggregateID:   taskID,
		AggregateType: "Task",
		Data:          data,
		Version:       len(events),
	})
}

func (h *CommandHandler) CompleteTask(taskID, completedBy string) error {
	events := h.store.GetEvents(taskID)
	if len(events) == 0 {
		return fmt.Errorf("task %s not found", taskID)
	}
	data, _ := json.Marshal(TaskCompletedData{CompletedBy: completedBy})
	return h.store.Append(Event{
		Type:          TaskCompleted,
		AggregateID:   taskID,
		AggregateType: "Task",
		Data:          data,
		Version:       len(events),
	})
}

func (h *CommandHandler) AddComment(taskID, commentID, author, body string) error {
	events := h.store.GetEvents(taskID)
	if len(events) == 0 {
		return fmt.Errorf("task %s not found", taskID)
	}
	data, _ := json.Marshal(CommentAddedData{
		CommentID: commentID,
		Author:    author,
		Body:      body,
	})
	return h.store.Append(Event{
		Type:          CommentAdded,
		AggregateID:   taskID,
		AggregateType: "Task",
		Data:          data,
		Version:       len(events),
	})
}

// ============================================================
// QUERY SIDE: Read models (materialized views)
// ============================================================

// TaskView is the read model for a single task.
type TaskView struct {
	ID           string    `json:"id"`
	Title        string    `json:"title"`
	Description  string    `json:"description"`
	ProjectID    string    `json:"project_id"`
	CreatedBy    string    `json:"created_by"`
	AssignedTo   string    `json:"assigned_to,omitempty"`
	Status       string    `json:"status"`
	CommentCount int       `json:"comment_count"`
	CreatedAt    time.Time `json:"created_at"`
	CompletedAt  *time.Time `json:"completed_at,omitempty"`
}

// DashboardView is a summary read model.
type DashboardView struct {
	TotalTasks     int            `json:"total_tasks"`
	CompletedTasks int            `json:"completed_tasks"`
	OpenTasks      int            `json:"open_tasks"`
	TasksByProject map[string]int `json:"tasks_by_project"`
	TasksByUser    map[string]int `json:"tasks_by_user"`
}

// ReadModel stores materialized views.
type ReadModel struct {
	mu        sync.RWMutex
	tasks     map[string]*TaskView
	dashboard *DashboardView
}

func NewReadModel() *ReadModel {
	return &ReadModel{
		tasks: make(map[string]*TaskView),
		dashboard: &DashboardView{
			TasksByProject: make(map[string]int),
			TasksByUser:    make(map[string]int),
		},
	}
}

// Query methods
func (rm *ReadModel) GetTask(id string) (*TaskView, bool) {
	rm.mu.RLock()
	defer rm.mu.RUnlock()
	t, ok := rm.tasks[id]
	if !ok {
		return nil, false
	}
	copy := *t
	return &copy, true
}

func (rm *ReadModel) ListTasks(projectID string) []*TaskView {
	rm.mu.RLock()
	defer rm.mu.RUnlock()
	var result []*TaskView
	for _, t := range rm.tasks {
		if projectID == "" || t.ProjectID == projectID {
			copy := *t
			result = append(result, &copy)
		}
	}
	sort.Slice(result, func(i, j int) bool {
		return result[i].CreatedAt.Before(result[j].CreatedAt)
	})
	return result
}

func (rm *ReadModel) GetDashboard() DashboardView {
	rm.mu.RLock()
	defer rm.mu.RUnlock()
	return *rm.dashboard
}

// ============================================================
// PROJECTOR: Listens to events and updates read models
// ============================================================

type Projector struct {
	readModel *ReadModel
	store     *EventStore
}

func NewProjector(store *EventStore, readModel *ReadModel) *Projector {
	return &Projector{
		readModel: readModel,
		store:     store,
	}
}

// Start begins listening for events and projecting them.
func (p *Projector) Start(done <-chan struct{}) {
	ch := p.store.Subscribe()
	go func() {
		for {
			select {
			case event := <-ch:
				p.project(event)
			case <-done:
				return
			}
		}
	}()
}

// Rebuild replays all historical events to rebuild the read model.
func (p *Projector) Rebuild() {
	events := p.store.AllEvents()
	for _, event := range events {
		p.project(event)
	}
}

func (p *Projector) project(event Event) {
	p.readModel.mu.Lock()
	defer p.readModel.mu.Unlock()

	switch event.Type {
	case TaskCreated:
		var data TaskCreatedData
		json.Unmarshal(event.Data, &data)
		p.readModel.tasks[event.AggregateID] = &TaskView{
			ID:          event.AggregateID,
			Title:       data.Title,
			Description: data.Description,
			ProjectID:   data.ProjectID,
			CreatedBy:   data.CreatedBy,
			Status:      "open",
			CreatedAt:   event.Timestamp,
		}
		p.readModel.dashboard.TotalTasks++
		p.readModel.dashboard.OpenTasks++
		p.readModel.dashboard.TasksByProject[data.ProjectID]++

	case TaskAssigned:
		var data TaskAssignedData
		json.Unmarshal(event.Data, &data)
		if task, ok := p.readModel.tasks[event.AggregateID]; ok {
			task.AssignedTo = data.AssigneeID
			p.readModel.dashboard.TasksByUser[data.AssigneeID]++
		}

	case TaskCompleted:
		if task, ok := p.readModel.tasks[event.AggregateID]; ok {
			task.Status = "completed"
			now := event.Timestamp
			task.CompletedAt = &now
			p.readModel.dashboard.CompletedTasks++
			p.readModel.dashboard.OpenTasks--
		}

	case CommentAdded:
		if task, ok := p.readModel.tasks[event.AggregateID]; ok {
			task.CommentCount++
		}
	}
}

// ============================================================
// QUERY HANDLER: Serves read requests from the read model
// ============================================================

type QueryHandler struct {
	readModel *ReadModel
}

func NewQueryHandler(readModel *ReadModel) *QueryHandler {
	return &QueryHandler{readModel: readModel}
}

func (q *QueryHandler) GetTask(id string) (*TaskView, error) {
	task, ok := q.readModel.GetTask(id)
	if !ok {
		return nil, fmt.Errorf("task %s not found", id)
	}
	return task, nil
}

func (q *QueryHandler) ListProjectTasks(projectID string) []*TaskView {
	return q.readModel.ListTasks(projectID)
}

func (q *QueryHandler) GetDashboard() DashboardView {
	return q.readModel.GetDashboard()
}

// ============================================================
// Main: Wire everything together
// ============================================================

func main() {
	// Create infrastructure
	store := NewEventStore()
	readModel := NewReadModel()
	projector := NewProjector(store, readModel)

	// Start the projector (listens for real-time events)
	done := make(chan struct{})
	projector.Start(done)

	// Create command and query handlers
	commands := NewCommandHandler(store)
	queries := NewQueryHandler(readModel)

	// === Execute commands (writes) ===
	fmt.Println("=== Executing Commands ===\n")

	commands.CreateTask("task-1", "Implement user auth", "Add JWT-based authentication", "proj-backend", "alice")
	fmt.Println("Created: Implement user auth")

	commands.CreateTask("task-2", "Design landing page", "Create responsive landing page", "proj-frontend", "bob")
	fmt.Println("Created: Design landing page")

	commands.CreateTask("task-3", "Set up CI/CD", "Configure GitHub Actions", "proj-infra", "charlie")
	fmt.Println("Created: Set up CI/CD")

	commands.AssignTask("task-1", "alice")
	fmt.Println("Assigned task-1 to alice")

	commands.AssignTask("task-2", "bob")
	fmt.Println("Assigned task-2 to bob")

	commands.AddComment("task-1", "comment-1", "bob", "Should we use OAuth2 instead?")
	fmt.Println("Added comment on task-1")

	commands.AddComment("task-1", "comment-2", "alice", "Good idea, let me look into it")
	fmt.Println("Added comment on task-1")

	commands.CompleteTask("task-3", "charlie")
	fmt.Println("Completed task-3")

	// Small delay to let projector process all events
	time.Sleep(50 * time.Millisecond)

	// === Execute queries (reads from materialized view) ===
	fmt.Println("\n=== Query Results ===\n")

	// Get single task
	task, err := queries.GetTask("task-1")
	if err == nil {
		fmt.Printf("Task: %s\n", task.Title)
		fmt.Printf("  Status:     %s\n", task.Status)
		fmt.Printf("  Assigned:   %s\n", task.AssignedTo)
		fmt.Printf("  Comments:   %d\n", task.CommentCount)
	}

	// List all tasks
	fmt.Println("\nAll Tasks:")
	allTasks := queries.ListProjectTasks("")
	for _, t := range allTasks {
		status := t.Status
		if t.AssignedTo != "" {
			fmt.Printf("  [%s] %s (assigned to %s)\n", status, t.Title, t.AssignedTo)
		} else {
			fmt.Printf("  [%s] %s\n", status, t.Title)
		}
	}

	// Dashboard
	fmt.Println("\nDashboard:")
	dash := queries.GetDashboard()
	fmt.Printf("  Total tasks:     %d\n", dash.TotalTasks)
	fmt.Printf("  Open tasks:      %d\n", dash.OpenTasks)
	fmt.Printf("  Completed tasks: %d\n", dash.CompletedTasks)
	fmt.Println("  Tasks by project:")
	for project, count := range dash.TasksByProject {
		fmt.Printf("    %s: %d\n", project, count)
	}

	// === Show the event log ===
	fmt.Println("\n=== Complete Event Log ===")
	for _, evt := range store.AllEvents() {
		fmt.Printf("  [%s] %-18s aggregate=%s data=%s\n",
			evt.ID, evt.Type, evt.AggregateID, string(evt.Data))
	}

	close(done)
}
```

### CQRS Tradeoffs

**Benefits:**
- Read models are optimized for specific queries (denormalized, pre-joined, pre-aggregated)
- Write model captures business rules and invariants
- Read and write sides can scale independently
- Multiple read models for different query patterns

**Costs:**
- Eventual consistency between write and read models (there is a delay)
- More complex infrastructure (event store + read database + projector)
- Read model can become stale if the projector falls behind

---

## 9. Stream Processing Fundamentals

### Unbounded Data

Stream processing operates on data that has no defined end. Unlike batch processing, where you process all data then produce a result, stream processing must produce results incrementally as data arrives.

### Key Concepts

**Event Time vs Processing Time**: Event time is when the event actually occurred. Processing time is when the system processes it. These are different because of network delays, buffering, and reprocessing. A stream processor must handle events arriving out of order.

**Windows**: Since streams are unbounded, you need to group events into finite chunks for aggregation. A window defines the boundaries of these chunks.

**Watermarks**: A watermark is a declaration that "I believe all events with timestamps up to T have arrived." Events arriving after the watermark are considered late.

**Triggers**: Triggers determine when to emit results for a window. You might emit when the window closes, when a watermark passes, or periodically.

**State**: Stream processors maintain state between events (running counts, session data, etc.). This state must survive restarts, which requires checkpointing.

```go
package main

import (
	"fmt"
	"time"
)

// StreamEvent represents a single event in a stream.
type StreamEvent struct {
	Key       string
	Value     interface{}
	EventTime time.Time
	// ProcessTime is set when the processor receives the event
	ProcessTime time.Time
}

// Watermark represents "all events up to this time have arrived."
type Watermark struct {
	Timestamp time.Time
}

// WindowType represents the strategy for grouping events.
type WindowType int

const (
	TumblingWindow WindowType = iota
	SlidingWindow
	SessionWindow
)

// Window represents a time range that groups events.
type Window struct {
	Start time.Time
	End   time.Time
}

func (w Window) Contains(t time.Time) bool {
	return !t.Before(w.Start) && t.Before(w.End)
}

func (w Window) String() string {
	return fmt.Sprintf("[%s, %s)",
		w.Start.Format("15:04:05"),
		w.End.Format("15:04:05"))
}

// WindowAssigner assigns events to windows based on their event time.
type WindowAssigner struct {
	Type     WindowType
	Size     time.Duration
	Slide    time.Duration // only for sliding windows
	GapLimit time.Duration // only for session windows
}

// AssignTumbling assigns an event to its tumbling window.
func (wa *WindowAssigner) AssignTumbling(eventTime time.Time) Window {
	// Floor the event time to the nearest window boundary
	windowStart := eventTime.Truncate(wa.Size)
	return Window{
		Start: windowStart,
		End:   windowStart.Add(wa.Size),
	}
}

// AssignSliding assigns an event to all overlapping sliding windows.
func (wa *WindowAssigner) AssignSliding(eventTime time.Time) []Window {
	var windows []Window
	// Find the earliest window that could contain this event
	earliestStart := eventTime.Truncate(wa.Slide)
	for start := earliestStart.Add(-wa.Size + wa.Slide); !start.After(eventTime); start = start.Add(wa.Slide) {
		w := Window{Start: start, End: start.Add(wa.Size)}
		if w.Contains(eventTime) {
			windows = append(windows, w)
		}
	}
	return windows
}

func main() {
	baseTime := time.Date(2025, 1, 15, 10, 0, 0, 0, time.UTC)

	events := []StreamEvent{
		{Key: "user1", Value: "click", EventTime: baseTime.Add(0 * time.Second)},
		{Key: "user2", Value: "click", EventTime: baseTime.Add(3 * time.Second)},
		{Key: "user1", Value: "click", EventTime: baseTime.Add(7 * time.Second)},
		{Key: "user3", Value: "click", EventTime: baseTime.Add(12 * time.Second)},
		{Key: "user1", Value: "click", EventTime: baseTime.Add(15 * time.Second)},
		{Key: "user2", Value: "click", EventTime: baseTime.Add(18 * time.Second)},
		{Key: "user1", Value: "click", EventTime: baseTime.Add(25 * time.Second)},
		// Late event -- arrived after its window should have closed
		{Key: "user3", Value: "click", EventTime: baseTime.Add(2 * time.Second)},
	}

	// === Tumbling Windows (10 second windows) ===
	fmt.Println("=== Tumbling Windows (10s) ===")
	tumblingAssigner := &WindowAssigner{Type: TumblingWindow, Size: 10 * time.Second}

	tumblingCounts := make(map[Window]int)
	for _, event := range events {
		w := tumblingAssigner.AssignTumbling(event.EventTime)
		tumblingCounts[w]++
	}

	for w, count := range tumblingCounts {
		fmt.Printf("  Window %s: %d events\n", w, count)
	}

	// === Sliding Windows (10s window, 5s slide) ===
	fmt.Println("\n=== Sliding Windows (10s window, 5s slide) ===")
	slidingAssigner := &WindowAssigner{
		Type:  SlidingWindow,
		Size:  10 * time.Second,
		Slide: 5 * time.Second,
	}

	slidingCounts := make(map[Window]int)
	for _, event := range events {
		windows := slidingAssigner.AssignSliding(event.EventTime)
		for _, w := range windows {
			slidingCounts[w]++
		}
	}

	for w, count := range slidingCounts {
		fmt.Printf("  Window %s: %d events\n", w, count)
	}

	// === Demonstrate Event Time vs Processing Time ===
	fmt.Println("\n=== Event Time vs Processing Time ===")
	fmt.Println("Notice: the last event has EventTime=10:00:02 but arrives last.")
	fmt.Println("In a real system, its ProcessingTime would be much later.")
	for i, e := range events {
		fmt.Printf("  Arrival order %d: key=%s event_time=%s\n",
			i+1, e.Key, e.EventTime.Format("15:04:05"))
	}
	fmt.Println("\nA watermark at 10:00:20 would mark the last event as LATE")
	fmt.Println("because its event time (10:00:02) is before the watermark.")
}
```

---

## 10. Building a Stream Processor in Go

### Generic Stream Operations

Using Go generics, we can build a composable stream processing library where channels serve as streams.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Stream is a read-only channel representing an event stream.
type Stream[T any] <-chan T

// SourceFrom creates a stream from a slice (useful for testing).
func SourceFrom[T any](items ...T) Stream[T] {
	ch := make(chan T, len(items))
	go func() {
		defer close(ch)
		for _, item := range items {
			ch <- item
		}
	}()
	return ch
}

// Map transforms each element in the stream.
func Map[T, U any](in Stream[T], fn func(T) U) Stream[U] {
	out := make(chan U, cap(in))
	go func() {
		defer close(out)
		for item := range in {
			out <- fn(item)
		}
	}()
	return out
}

// Filter keeps only elements that satisfy the predicate.
func Filter[T any](in Stream[T], fn func(T) bool) Stream[T] {
	out := make(chan T, cap(in))
	go func() {
		defer close(out)
		for item := range in {
			if fn(item) {
				out <- item
			}
		}
	}()
	return out
}

// FlatMap transforms each element into zero or more elements.
func FlatMap[T, U any](in Stream[T], fn func(T) []U) Stream[U] {
	out := make(chan U, cap(in))
	go func() {
		defer close(out)
		for item := range in {
			for _, result := range fn(item) {
				out <- result
			}
		}
	}()
	return out
}

// Reduce consumes the entire stream and produces a single value.
func Reduce[T, U any](in Stream[T], initial U, fn func(U, T) U) U {
	acc := initial
	for item := range in {
		acc = fn(acc, item)
	}
	return acc
}

// Window collects elements into time-based windows.
func Window[T any](in Stream[T], size time.Duration) Stream[[]T] {
	out := make(chan []T, 10)
	go func() {
		defer close(out)
		ticker := time.NewTicker(size)
		defer ticker.Stop()

		var buffer []T
		for {
			select {
			case item, ok := <-in:
				if !ok {
					// Stream closed; emit remaining buffer
					if len(buffer) > 0 {
						out <- buffer
					}
					return
				}
				buffer = append(buffer, item)
			case <-ticker.C:
				if len(buffer) > 0 {
					out <- buffer
					buffer = nil
				}
			}
		}
	}()
	return out
}

// CountWindow collects a fixed number of elements per window.
func CountWindow[T any](in Stream[T], count int) Stream[[]T] {
	out := make(chan []T, 10)
	go func() {
		defer close(out)
		buffer := make([]T, 0, count)
		for item := range in {
			buffer = append(buffer, item)
			if len(buffer) >= count {
				out <- buffer
				buffer = make([]T, 0, count)
			}
		}
		// Emit remaining
		if len(buffer) > 0 {
			out <- buffer
		}
	}()
	return out
}

// Merge combines multiple streams into one.
func Merge[T any](streams ...Stream[T]) Stream[T] {
	out := make(chan T, 100)
	var wg sync.WaitGroup
	for _, s := range streams {
		wg.Add(1)
		go func(stream Stream[T]) {
			defer wg.Done()
			for item := range stream {
				out <- item
			}
		}(s)
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

// ParallelMap applies a function across multiple goroutines.
func ParallelMap[T, U any](in Stream[T], numWorkers int, fn func(T) U) Stream[U] {
	out := make(chan U, 100)
	var wg sync.WaitGroup
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for item := range in {
				out <- fn(item)
			}
		}()
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

// Tee duplicates a stream into two.
func Tee[T any](in Stream[T]) (Stream[T], Stream[T]) {
	out1 := make(chan T, 100)
	out2 := make(chan T, 100)
	go func() {
		defer close(out1)
		defer close(out2)
		for item := range in {
			out1 <- item
			out2 <- item
		}
	}()
	return out1, out2
}

// Collect drains a stream into a slice.
func Collect[T any](in Stream[T]) []T {
	var result []T
	for item := range in {
		result = append(result, item)
	}
	return result
}

// ============================================================
// Example: Real-time event processing
// ============================================================

type ClickEvent struct {
	UserID    string
	Page      string
	Timestamp time.Time
}

type PageCount struct {
	Page  string
	Count int
}

func main() {
	fmt.Println("=== Stream Processing with Generics ===\n")

	// === Example 1: Basic stream operations ===
	fmt.Println("--- Map, Filter, Reduce ---")

	numbers := SourceFrom(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

	// Filter even numbers, then square them
	evens := Filter(numbers, func(n int) bool { return n%2 == 0 })
	squared := Map(evens, func(n int) int { return n * n })
	sum := Reduce(squared, 0, func(acc, n int) int { return acc + n })

	fmt.Printf("Sum of squares of even numbers 1-10: %d\n", sum)
	// 4 + 16 + 36 + 64 + 100 = 220

	// === Example 2: FlatMap ===
	fmt.Println("\n--- FlatMap ---")

	sentences := SourceFrom("hello world", "foo bar baz", "go is great")
	words := FlatMap(sentences, func(s string) []string {
		result := []string{}
		word := ""
		for _, ch := range s {
			if ch == ' ' {
				if word != "" {
					result = append(result, word)
					word = ""
				}
			} else {
				word += string(ch)
			}
		}
		if word != "" {
			result = append(result, word)
		}
		return result
	})

	allWords := Collect(words)
	fmt.Printf("Words: %v\n", allWords)

	// === Example 3: Count windows ===
	fmt.Println("\n--- Count Windows (batches of 3) ---")

	items := SourceFrom("a", "b", "c", "d", "e", "f", "g", "h")
	batches := CountWindow(items, 3)

	for batch := range batches {
		fmt.Printf("  Batch: %v\n", batch)
	}

	// === Example 4: Merge streams ===
	fmt.Println("\n--- Merge Streams ---")

	stream1 := SourceFrom("A1", "A2", "A3")
	stream2 := SourceFrom("B1", "B2", "B3")
	stream3 := SourceFrom("C1", "C2", "C3")

	merged := Merge(stream1, stream2, stream3)
	mergedItems := Collect(merged)
	fmt.Printf("Merged (%d items): %v\n", len(mergedItems), mergedItems)

	// === Example 5: Tee and parallel processing ===
	fmt.Println("\n--- Tee (duplicate stream) ---")

	source := SourceFrom(10, 20, 30, 40, 50)
	branch1, branch2 := Tee(source)

	// Process each branch differently
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		sum := Reduce(branch1, 0, func(acc, n int) int { return acc + n })
		fmt.Printf("  Branch 1 (sum): %d\n", sum)
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		max := Reduce(branch2, 0, func(acc, n int) int {
			if n > acc {
				return n
			}
			return acc
		})
		fmt.Printf("  Branch 2 (max): %d\n", max)
	}()

	wg.Wait()
}
```

---

## 11. Windowing Strategies

### Three Types of Windows

In stream processing, windows are essential for turning an unbounded stream into bounded chunks that can be aggregated. There are three fundamental windowing strategies:

1. **Tumbling windows**: Fixed-size, non-overlapping windows. Every event belongs to exactly one window.
2. **Sliding windows**: Fixed-size windows that advance by a smaller "slide" interval. Events may belong to multiple windows.
3. **Session windows**: Dynamic windows defined by gaps in activity. A new window starts after a period of inactivity.

```go
package main

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

// Event represents a timestamped event.
type Event struct {
	Key       string
	Value     float64
	Timestamp time.Time
}

// WindowResult holds the aggregated result for a window.
type WindowResult struct {
	WindowStart time.Time
	WindowEnd   time.Time
	Key         string
	Count       int
	Sum         float64
	Avg         float64
}

func (wr WindowResult) String() string {
	return fmt.Sprintf("[%s - %s] key=%s count=%d sum=%.1f avg=%.1f",
		wr.WindowStart.Format("15:04:05"),
		wr.WindowEnd.Format("15:04:05"),
		wr.Key, wr.Count, wr.Sum, wr.Avg)
}

// ============================================================
// Tumbling Window Processor
// ============================================================

type TumblingWindowProcessor struct {
	windowSize time.Duration
	mu         sync.Mutex
	windows    map[time.Time]map[string]*accumulator
	results    chan WindowResult
}

type accumulator struct {
	count int
	sum   float64
}

func NewTumblingWindowProcessor(size time.Duration) *TumblingWindowProcessor {
	return &TumblingWindowProcessor{
		windowSize: size,
		windows:    make(map[time.Time]map[string]*accumulator),
		results:    make(chan WindowResult, 100),
	}
}

func (p *TumblingWindowProcessor) Process(event Event) {
	windowStart := event.Timestamp.Truncate(p.windowSize)

	p.mu.Lock()
	if p.windows[windowStart] == nil {
		p.windows[windowStart] = make(map[string]*accumulator)
	}
	acc, ok := p.windows[windowStart][event.Key]
	if !ok {
		acc = &accumulator{}
		p.windows[windowStart][event.Key] = acc
	}
	acc.count++
	acc.sum += event.Value
	p.mu.Unlock()
}

func (p *TumblingWindowProcessor) Flush() []WindowResult {
	p.mu.Lock()
	defer p.mu.Unlock()

	var results []WindowResult
	for windowStart, keys := range p.windows {
		for key, acc := range keys {
			results = append(results, WindowResult{
				WindowStart: windowStart,
				WindowEnd:   windowStart.Add(p.windowSize),
				Key:         key,
				Count:       acc.count,
				Sum:         acc.sum,
				Avg:         acc.sum / float64(acc.count),
			})
		}
	}
	sort.Slice(results, func(i, j int) bool {
		if results[i].WindowStart.Equal(results[j].WindowStart) {
			return results[i].Key < results[j].Key
		}
		return results[i].WindowStart.Before(results[j].WindowStart)
	})
	return results
}

// ============================================================
// Sliding Window Processor
// ============================================================

type SlidingWindowProcessor struct {
	windowSize time.Duration
	slideSize  time.Duration
	mu         sync.Mutex
	events     []Event // store all events; in production, evict old ones
}

func NewSlidingWindowProcessor(windowSize, slideSize time.Duration) *SlidingWindowProcessor {
	return &SlidingWindowProcessor{
		windowSize: windowSize,
		slideSize:  slideSize,
	}
}

func (p *SlidingWindowProcessor) Process(event Event) {
	p.mu.Lock()
	p.events = append(p.events, event)
	p.mu.Unlock()
}

func (p *SlidingWindowProcessor) Flush() []WindowResult {
	p.mu.Lock()
	defer p.mu.Unlock()

	if len(p.events) == 0 {
		return nil
	}

	// Find the time range
	minTime := p.events[0].Timestamp
	maxTime := p.events[0].Timestamp
	for _, e := range p.events[1:] {
		if e.Timestamp.Before(minTime) {
			minTime = e.Timestamp
		}
		if e.Timestamp.After(maxTime) {
			maxTime = e.Timestamp
		}
	}

	// Generate all window starts
	var results []WindowResult
	firstWindowStart := minTime.Truncate(p.slideSize)

	for ws := firstWindowStart; ws.Before(maxTime); ws = ws.Add(p.slideSize) {
		we := ws.Add(p.windowSize)
		keyAcc := make(map[string]*accumulator)

		for _, event := range p.events {
			if !event.Timestamp.Before(ws) && event.Timestamp.Before(we) {
				acc, ok := keyAcc[event.Key]
				if !ok {
					acc = &accumulator{}
					keyAcc[event.Key] = acc
				}
				acc.count++
				acc.sum += event.Value
			}
		}

		for key, acc := range keyAcc {
			results = append(results, WindowResult{
				WindowStart: ws,
				WindowEnd:   we,
				Key:         key,
				Count:       acc.count,
				Sum:         acc.sum,
				Avg:         acc.sum / float64(acc.count),
			})
		}
	}

	sort.Slice(results, func(i, j int) bool {
		if results[i].WindowStart.Equal(results[j].WindowStart) {
			return results[i].Key < results[j].Key
		}
		return results[i].WindowStart.Before(results[j].WindowStart)
	})
	return results
}

// ============================================================
// Session Window Processor
// ============================================================

type SessionWindowProcessor struct {
	gapTimeout time.Duration
	mu         sync.Mutex
	// sessions[key] = list of events in current session
	sessions map[string][]Event
}

func NewSessionWindowProcessor(gapTimeout time.Duration) *SessionWindowProcessor {
	return &SessionWindowProcessor{
		gapTimeout: gapTimeout,
		sessions:   make(map[string][]Event),
	}
}

func (p *SessionWindowProcessor) Process(event Event) {
	p.mu.Lock()
	p.sessions[event.Key] = append(p.sessions[event.Key], event)
	p.mu.Unlock()
}

func (p *SessionWindowProcessor) Flush() []WindowResult {
	p.mu.Lock()
	defer p.mu.Unlock()

	var results []WindowResult

	for key, events := range p.sessions {
		if len(events) == 0 {
			continue
		}

		// Sort events by timestamp
		sort.Slice(events, func(i, j int) bool {
			return events[i].Timestamp.Before(events[j].Timestamp)
		})

		// Split into sessions based on gap timeout
		sessionStart := events[0].Timestamp
		var sessionEvents []Event
		sessionEvents = append(sessionEvents, events[0])

		emitSession := func() {
			sum := 0.0
			for _, e := range sessionEvents {
				sum += e.Value
			}
			results = append(results, WindowResult{
				WindowStart: sessionStart,
				WindowEnd:   sessionEvents[len(sessionEvents)-1].Timestamp.Add(1 * time.Millisecond),
				Key:         key,
				Count:       len(sessionEvents),
				Sum:         sum,
				Avg:         sum / float64(len(sessionEvents)),
			})
		}

		for i := 1; i < len(events); i++ {
			gap := events[i].Timestamp.Sub(events[i-1].Timestamp)
			if gap > p.gapTimeout {
				// Gap exceeded: close current session, start new one
				emitSession()
				sessionStart = events[i].Timestamp
				sessionEvents = nil
			}
			sessionEvents = append(sessionEvents, events[i])
		}

		// Emit the last session
		if len(sessionEvents) > 0 {
			emitSession()
		}
	}

	sort.Slice(results, func(i, j int) bool {
		if results[i].Key == results[j].Key {
			return results[i].WindowStart.Before(results[j].WindowStart)
		}
		return results[i].Key < results[j].Key
	})
	return results
}

func main() {
	baseTime := time.Date(2025, 6, 15, 14, 0, 0, 0, time.UTC)

	// Simulated user activity events (value = page load time in ms)
	events := []Event{
		{Key: "user-1", Value: 120, Timestamp: baseTime.Add(0 * time.Second)},
		{Key: "user-2", Value: 85, Timestamp: baseTime.Add(2 * time.Second)},
		{Key: "user-1", Value: 200, Timestamp: baseTime.Add(5 * time.Second)},
		{Key: "user-1", Value: 95, Timestamp: baseTime.Add(8 * time.Second)},
		{Key: "user-2", Value: 150, Timestamp: baseTime.Add(12 * time.Second)},
		{Key: "user-1", Value: 300, Timestamp: baseTime.Add(15 * time.Second)},
		{Key: "user-2", Value: 110, Timestamp: baseTime.Add(18 * time.Second)},
		{Key: "user-1", Value: 75, Timestamp: baseTime.Add(22 * time.Second)},
		// Gap of 15 seconds for user-1 (session break)
		{Key: "user-1", Value: 180, Timestamp: baseTime.Add(40 * time.Second)},
		{Key: "user-1", Value: 90, Timestamp: baseTime.Add(43 * time.Second)},
		{Key: "user-2", Value: 200, Timestamp: baseTime.Add(25 * time.Second)},
	}

	// === Tumbling Windows (10 second windows) ===
	fmt.Println("=== Tumbling Windows (10s) ===")
	fmt.Println("Each event belongs to exactly one window.\n")

	tumbling := NewTumblingWindowProcessor(10 * time.Second)
	for _, event := range events {
		tumbling.Process(event)
	}
	for _, result := range tumbling.Flush() {
		fmt.Printf("  %s\n", result)
	}

	// === Sliding Windows (15s window, 5s slide) ===
	fmt.Println("\n=== Sliding Windows (15s window, 5s slide) ===")
	fmt.Println("Events can appear in multiple overlapping windows.\n")

	sliding := NewSlidingWindowProcessor(15*time.Second, 5*time.Second)
	for _, event := range events {
		sliding.Process(event)
	}
	for _, result := range sliding.Flush() {
		fmt.Printf("  %s\n", result)
	}

	// === Session Windows (10s gap timeout) ===
	fmt.Println("\n=== Session Windows (10s gap) ===")
	fmt.Println("A new session starts after 10s of inactivity per key.\n")

	session := NewSessionWindowProcessor(10 * time.Second)
	for _, event := range events {
		session.Process(event)
	}
	for _, result := range session.Flush() {
		fmt.Printf("  %s\n", result)
	}
}
```

### Choosing a Window Strategy

| Strategy | Use Case | Example |
|----------|----------|---------|
| **Tumbling** | Fixed reporting intervals | "Count errors per minute" |
| **Sliding** | Moving averages, overlapping analysis | "Average latency over last 5 minutes, computed every minute" |
| **Session** | User activity analysis | "Track page views per user session" |

---

## 12. Message Queues and Go

### Kafka-Style Consumer/Producer Patterns

Apache Kafka is the dominant distributed message queue for stream processing. Its key ideas:
- **Topics** are ordered, append-only logs partitioned across brokers
- **Producers** append messages to topics
- **Consumers** read messages at their own pace, tracking their position (offset)
- **Consumer groups** distribute partitions across consumers for parallel processing
- Messages are durable and replayable

We simulate these patterns in Go to understand the mechanics without external dependencies.

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// ============================================================
// Message and Topic
// ============================================================

// Message represents a single message in a topic.
type Message struct {
	Key       string
	Value     []byte
	Offset    int64
	Partition int
	Timestamp time.Time
}

// Partition is an ordered, append-only log of messages.
type Partition struct {
	mu       sync.RWMutex
	messages []Message
	id       int
}

func (p *Partition) Append(msg Message) int64 {
	p.mu.Lock()
	defer p.mu.Unlock()
	msg.Offset = int64(len(p.messages))
	msg.Partition = p.id
	msg.Timestamp = time.Now()
	p.messages = append(p.messages, msg)
	return msg.Offset
}

func (p *Partition) Read(offset int64, maxMessages int) []Message {
	p.mu.RLock()
	defer p.mu.RUnlock()
	if offset >= int64(len(p.messages)) {
		return nil
	}
	end := int64(len(p.messages))
	if offset+int64(maxMessages) < end {
		end = offset + int64(maxMessages)
	}
	result := make([]Message, end-offset)
	copy(result, p.messages[offset:end])
	return result
}

func (p *Partition) Size() int64 {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return int64(len(p.messages))
}

// Topic is a collection of partitions.
type Topic struct {
	Name       string
	Partitions []*Partition

	// Subscriber notification
	notifyMu sync.Mutex
	notify   []chan struct{}
}

func NewTopic(name string, numPartitions int) *Topic {
	partitions := make([]*Partition, numPartitions)
	for i := range partitions {
		partitions[i] = &Partition{id: i}
	}
	return &Topic{
		Name:       name,
		Partitions: partitions,
	}
}

// selectPartition determines which partition a message goes to.
func (t *Topic) selectPartition(key string) int {
	if key == "" {
		return rand.Intn(len(t.Partitions))
	}
	// Simple hash-based partitioning
	h := 0
	for _, ch := range key {
		h = h*31 + int(ch)
	}
	if h < 0 {
		h = -h
	}
	return h % len(t.Partitions)
}

func (t *Topic) notifyConsumers() {
	t.notifyMu.Lock()
	defer t.notifyMu.Unlock()
	for _, ch := range t.notify {
		select {
		case ch <- struct{}{}:
		default:
		}
	}
}

func (t *Topic) subscribe() chan struct{} {
	t.notifyMu.Lock()
	defer t.notifyMu.Unlock()
	ch := make(chan struct{}, 1)
	t.notify = append(t.notify, ch)
	return ch
}

// ============================================================
// Producer
// ============================================================

type Producer struct {
	topic *Topic
}

func NewProducer(topic *Topic) *Producer {
	return &Producer{topic: topic}
}

// Send writes a message to the topic. The partition is chosen by the key.
func (p *Producer) Send(key string, value []byte) (partition int, offset int64) {
	partIdx := p.topic.selectPartition(key)
	msg := Message{Key: key, Value: value}
	off := p.topic.Partitions[partIdx].Append(msg)
	p.topic.notifyConsumers()
	return partIdx, off
}

// ============================================================
// Consumer
// ============================================================

type Consumer struct {
	topic    *Topic
	groupID  string
	offsets  map[int]int64 // partition -> offset
	mu       sync.Mutex
	assigned []int // assigned partition IDs
}

func NewConsumer(topic *Topic, groupID string) *Consumer {
	return &Consumer{
		topic:   topic,
		groupID: groupID,
		offsets: make(map[int]int64),
	}
}

// Assign tells this consumer which partitions to read from.
func (c *Consumer) Assign(partitions ...int) {
	c.mu.Lock()
	c.assigned = partitions
	for _, p := range partitions {
		if _, ok := c.offsets[p]; !ok {
			c.offsets[p] = 0
		}
	}
	c.mu.Unlock()
}

// Poll reads available messages from assigned partitions.
func (c *Consumer) Poll(maxPerPartition int) []Message {
	c.mu.Lock()
	defer c.mu.Unlock()

	var all []Message
	for _, partID := range c.assigned {
		offset := c.offsets[partID]
		messages := c.topic.Partitions[partID].Read(offset, maxPerPartition)
		all = append(all, messages...)
		c.offsets[partID] = offset + int64(len(messages))
	}
	return all
}

// Commit saves the current offsets (in a real system, to a persistent store).
func (c *Consumer) Commit() map[int]int64 {
	c.mu.Lock()
	defer c.mu.Unlock()
	committed := make(map[int]int64)
	for k, v := range c.offsets {
		committed[k] = v
	}
	return committed
}

// ============================================================
// Consumer Group: distributes partitions across consumers
// ============================================================

type ConsumerGroup struct {
	GroupID   string
	topic     *Topic
	consumers []*Consumer
}

func NewConsumerGroup(groupID string, topic *Topic, numConsumers int) *ConsumerGroup {
	cg := &ConsumerGroup{
		GroupID: groupID,
		topic:   topic,
	}

	for i := 0; i < numConsumers; i++ {
		cg.consumers = append(cg.consumers, NewConsumer(topic, groupID))
	}

	// Assign partitions round-robin to consumers
	for i, partition := range topic.Partitions {
		consumerIdx := i % numConsumers
		cg.consumers[consumerIdx].Assign(partition.id)
	}

	return cg
}

func main() {
	fmt.Println("=== Kafka-Style Message Queue Simulation ===\n")

	// Create a topic with 4 partitions
	topic := NewTopic("user-events", 4)

	// === Produce messages ===
	producer := NewProducer(topic)

	events := []struct {
		key   string
		value string
	}{
		{"user-1", `{"action":"login"}`},
		{"user-2", `{"action":"login"}`},
		{"user-1", `{"action":"view_page","page":"/home"}`},
		{"user-3", `{"action":"login"}`},
		{"user-2", `{"action":"view_page","page":"/products"}`},
		{"user-1", `{"action":"add_to_cart","item":"widget"}`},
		{"user-3", `{"action":"view_page","page":"/about"}`},
		{"user-1", `{"action":"purchase","amount":29.99}`},
		{"user-2", `{"action":"logout"}`},
		{"user-3", `{"action":"view_page","page":"/contact"}`},
		{"user-1", `{"action":"logout"}`},
		{"user-3", `{"action":"logout"}`},
	}

	fmt.Println("--- Producing Messages ---")
	for _, e := range events {
		part, offset := producer.Send(e.key, []byte(e.value))
		fmt.Printf("  Sent key=%s to partition=%d offset=%d\n", e.key, part, offset)
	}

	// === Consumer Group with 2 consumers ===
	fmt.Println("\n--- Consumer Group (2 consumers for 4 partitions) ---")
	group := NewConsumerGroup("analytics-group", topic, 2)

	for i, consumer := range group.consumers {
		consumer.mu.Lock()
		assigned := consumer.assigned
		consumer.mu.Unlock()

		fmt.Printf("  Consumer %d assigned partitions: %v\n", i, assigned)

		messages := consumer.Poll(100)
		fmt.Printf("  Consumer %d received %d messages:\n", i, len(messages))
		for _, msg := range messages {
			fmt.Printf("    [p%d:o%d] key=%s value=%s\n",
				msg.Partition, msg.Offset, msg.Key, string(msg.Value))
		}
		committed := consumer.Commit()
		fmt.Printf("  Consumer %d committed offsets: %v\n\n", i, committed)
	}

	// === Demonstrate at-least-once processing ===
	fmt.Println("--- At-Least-Once Processing ---")
	fmt.Println("If a consumer crashes before committing, it re-reads from the last committed offset.")
	fmt.Println()

	// Create a consumer that reads some messages but does not commit
	consumer := NewConsumer(topic, "manual-group")
	consumer.Assign(0) // Only partition 0

	firstBatch := consumer.Poll(2)
	fmt.Printf("First read: %d messages from partition 0\n", len(firstBatch))
	for _, msg := range firstBatch {
		fmt.Printf("  [o%d] %s: %s\n", msg.Offset, msg.Key, string(msg.Value))
	}

	// Simulate crash: reset offset to 0 (did not commit)
	consumer.mu.Lock()
	consumer.offsets[0] = 0
	consumer.mu.Unlock()

	// Re-read: we get the same messages again (at-least-once)
	secondBatch := consumer.Poll(2)
	fmt.Printf("\nAfter simulated crash, re-read: %d messages (same ones!)\n", len(secondBatch))
	for _, msg := range secondBatch {
		fmt.Printf("  [o%d] %s: %s\n", msg.Offset, msg.Key, string(msg.Value))
	}

	fmt.Println("\nThis is why consumers must be idempotent with at-least-once delivery.")

	// === Show partition distribution ===
	fmt.Println("\n--- Partition Sizes ---")
	for i, p := range topic.Partitions {
		fmt.Printf("  Partition %d: %d messages\n", i, p.Size())
	}
}
```

### Delivery Guarantees

| Guarantee | Description | How |
|-----------|-------------|-----|
| **At-most-once** | Messages may be lost, never duplicated | Commit offset before processing |
| **At-least-once** | Messages never lost, may be duplicated | Commit offset after processing |
| **Exactly-once** | Messages processed exactly once | Transactional writes + idempotent consumers |

Go pattern for at-least-once processing:

```go
// Process then commit (at-least-once):
messages := consumer.Poll(100)
for _, msg := range messages {
    if err := processMessage(msg); err != nil {
        // Do NOT commit. Will re-read on next poll.
        log.Printf("error processing message: %v", err)
        break
    }
}
consumer.Commit() // Only commit if all succeeded
```

---

## 13. Change Data Capture (CDC)

### Tailing the Database Log

Change Data Capture captures changes (inserts, updates, deletes) from a database and streams them as events. This lets you build derived views, keep caches in sync, feed search indexes, and trigger downstream processing -- all without coupling the downstream systems to the source database.

In DDIA, Kleppmann argues that CDC makes the database's internal write-ahead log available as a public API, turning the database into an event source.

```go
package main

import (
	"encoding/json"
	"fmt"
	"sync"
	"time"
)

// ============================================================
// Simulated Database with CDC
// ============================================================

// ChangeType represents the type of database change.
type ChangeType string

const (
	Insert ChangeType = "INSERT"
	Update ChangeType = "UPDATE"
	Delete ChangeType = "DELETE"
)

// ChangeEvent represents a captured database change.
type ChangeEvent struct {
	ID        int64           `json:"id"`
	Table     string          `json:"table"`
	Type      ChangeType      `json:"type"`
	Key       string          `json:"key"`
	Before    json.RawMessage `json:"before,omitempty"` // nil for INSERT
	After     json.RawMessage `json:"after,omitempty"`  // nil for DELETE
	Timestamp time.Time       `json:"timestamp"`
}

// Database simulates a simple key-value store with CDC.
type Database struct {
	mu   sync.RWMutex
	data map[string]map[string]json.RawMessage // table -> key -> value

	// Write-ahead log (the CDC source)
	walMu sync.Mutex
	wal   []ChangeEvent
	seqID int64

	// CDC subscribers
	subMu       sync.RWMutex
	subscribers []chan ChangeEvent
}

func NewDatabase() *Database {
	return &Database{
		data: make(map[string]map[string]json.RawMessage),
	}
}

func (db *Database) emit(event ChangeEvent) {
	db.walMu.Lock()
	db.seqID++
	event.ID = db.seqID
	event.Timestamp = time.Now()
	db.wal = append(db.wal, event)
	db.walMu.Unlock()

	db.subMu.RLock()
	for _, sub := range db.subscribers {
		select {
		case sub <- event:
		default:
			// Slow consumer
		}
	}
	db.subMu.RUnlock()
}

func (db *Database) Subscribe() chan ChangeEvent {
	ch := make(chan ChangeEvent, 256)
	db.subMu.Lock()
	db.subscribers = append(db.subscribers, ch)
	db.subMu.Unlock()
	return ch
}

func (db *Database) Insert(table, key string, value interface{}) error {
	data, err := json.Marshal(value)
	if err != nil {
		return err
	}

	db.mu.Lock()
	if db.data[table] == nil {
		db.data[table] = make(map[string]json.RawMessage)
	}
	if _, exists := db.data[table][key]; exists {
		db.mu.Unlock()
		return fmt.Errorf("key %s already exists in table %s", key, table)
	}
	db.data[table][key] = data
	db.mu.Unlock()

	db.emit(ChangeEvent{
		Table: table,
		Type:  Insert,
		Key:   key,
		After: data,
	})
	return nil
}

func (db *Database) Update(table, key string, value interface{}) error {
	data, err := json.Marshal(value)
	if err != nil {
		return err
	}

	db.mu.Lock()
	if db.data[table] == nil || db.data[table][key] == nil {
		db.mu.Unlock()
		return fmt.Errorf("key %s not found in table %s", key, table)
	}
	before := db.data[table][key]
	db.data[table][key] = data
	db.mu.Unlock()

	db.emit(ChangeEvent{
		Table:  table,
		Type:   Update,
		Key:    key,
		Before: before,
		After:  data,
	})
	return nil
}

func (db *Database) Delete(table, key string) error {
	db.mu.Lock()
	if db.data[table] == nil || db.data[table][key] == nil {
		db.mu.Unlock()
		return fmt.Errorf("key %s not found in table %s", key, table)
	}
	before := db.data[table][key]
	delete(db.data[table], key)
	db.mu.Unlock()

	db.emit(ChangeEvent{
		Table:  table,
		Type:   Delete,
		Key:    key,
		Before: before,
	})
	return nil
}

func (db *Database) Get(table, key string) (json.RawMessage, bool) {
	db.mu.RLock()
	defer db.mu.RUnlock()
	if db.data[table] == nil {
		return nil, false
	}
	data, ok := db.data[table][key]
	return data, ok
}

// GetWAL returns the write-ahead log from a given position.
func (db *Database) GetWAL(fromID int64) []ChangeEvent {
	db.walMu.Lock()
	defer db.walMu.Unlock()

	var result []ChangeEvent
	for _, event := range db.wal {
		if event.ID > fromID {
			result = append(result, event)
		}
	}
	return result
}

// ============================================================
// CDC Consumers: Search Index and Cache
// ============================================================

// SearchIndex is a derived view built from CDC events.
type SearchIndex struct {
	mu      sync.RWMutex
	index   map[string]map[string]string // table -> key -> searchable_text
	lastID  int64
}

func NewSearchIndex() *SearchIndex {
	return &SearchIndex{
		index: make(map[string]map[string]string),
	}
}

func (si *SearchIndex) HandleChange(event ChangeEvent) {
	si.mu.Lock()
	defer si.mu.Unlock()

	if si.index[event.Table] == nil {
		si.index[event.Table] = make(map[string]string)
	}

	switch event.Type {
	case Insert, Update:
		// Extract searchable text from the JSON
		var data map[string]interface{}
		json.Unmarshal(event.After, &data)
		text := ""
		for _, v := range data {
			text += fmt.Sprintf("%v ", v)
		}
		si.index[event.Table][event.Key] = text
	case Delete:
		delete(si.index[event.Table], event.Key)
	}

	si.lastID = event.ID
}

func (si *SearchIndex) Search(table, query string) []string {
	si.mu.RLock()
	defer si.mu.RUnlock()

	var results []string
	if si.index[table] == nil {
		return results
	}
	for key, text := range si.index[table] {
		// Simple substring search
		if containsSubstring(text, query) {
			results = append(results, key)
		}
	}
	return results
}

func containsSubstring(s, substr string) bool {
	if len(substr) == 0 {
		return true
	}
	for i := 0; i <= len(s)-len(substr); i++ {
		if s[i:i+len(substr)] == substr {
			return true
		}
	}
	return false
}

// Cache is another derived view: an in-memory cache kept in sync via CDC.
type Cache struct {
	mu    sync.RWMutex
	data  map[string]json.RawMessage // "table:key" -> value
	hits  int
	misses int
}

func NewCache() *Cache {
	return &Cache{
		data: make(map[string]json.RawMessage),
	}
}

func (c *Cache) HandleChange(event ChangeEvent) {
	c.mu.Lock()
	defer c.mu.Unlock()

	cacheKey := event.Table + ":" + event.Key
	switch event.Type {
	case Insert, Update:
		c.data[cacheKey] = event.After
	case Delete:
		delete(c.data, cacheKey)
	}
}

func (c *Cache) Get(table, key string) (json.RawMessage, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	cacheKey := table + ":" + key
	data, ok := c.data[cacheKey]
	if ok {
		c.hits++
	} else {
		c.misses++
	}
	return data, ok
}

// ============================================================
// CDC Pipeline: connects the database to consumers
// ============================================================

type CDCPipeline struct {
	db        *Database
	handlers  []func(ChangeEvent)
}

func NewCDCPipeline(db *Database) *CDCPipeline {
	return &CDCPipeline{db: db}
}

func (p *CDCPipeline) AddHandler(handler func(ChangeEvent)) {
	p.handlers = append(p.handlers, handler)
}

func (p *CDCPipeline) Start(done <-chan struct{}) {
	ch := p.db.Subscribe()
	go func() {
		for {
			select {
			case event := <-ch:
				for _, handler := range p.handlers {
					handler(event)
				}
			case <-done:
				return
			}
		}
	}()
}

// ============================================================
// Domain types
// ============================================================

type Product struct {
	Name     string  `json:"name"`
	Category string  `json:"category"`
	Price    float64 `json:"price"`
	InStock  bool    `json:"in_stock"`
}

func main() {
	fmt.Println("=== Change Data Capture (CDC) ===\n")

	// Set up infrastructure
	db := NewDatabase()
	searchIndex := NewSearchIndex()
	cache := NewCache()

	// Wire up CDC pipeline
	pipeline := NewCDCPipeline(db)
	pipeline.AddHandler(searchIndex.HandleChange)
	pipeline.AddHandler(cache.HandleChange)
	pipeline.AddHandler(func(event ChangeEvent) {
		// Audit log handler
		fmt.Printf("  [CDC] %s on %s.%s\n", event.Type, event.Table, event.Key)
	})

	done := make(chan struct{})
	pipeline.Start(done)

	// === Make database changes ===
	fmt.Println("--- Making Database Changes ---")

	db.Insert("products", "prod-1", Product{
		Name: "Mechanical Keyboard", Category: "electronics", Price: 149.99, InStock: true,
	})
	db.Insert("products", "prod-2", Product{
		Name: "Ergonomic Mouse", Category: "electronics", Price: 79.99, InStock: true,
	})
	db.Insert("products", "prod-3", Product{
		Name: "Standing Desk", Category: "furniture", Price: 599.99, InStock: false,
	})
	db.Insert("products", "prod-4", Product{
		Name: "Monitor Arm", Category: "furniture", Price: 129.99, InStock: true,
	})

	// Wait for CDC to process
	time.Sleep(50 * time.Millisecond)

	db.Update("products", "prod-3", Product{
		Name: "Standing Desk", Category: "furniture", Price: 549.99, InStock: true,
	})
	db.Delete("products", "prod-2")

	time.Sleep(50 * time.Millisecond)

	// === Query derived views ===
	fmt.Println("\n--- Search Index (derived from CDC) ---")

	results := searchIndex.Search("products", "Keyboard")
	fmt.Printf("Search 'Keyboard': %v\n", results)

	results = searchIndex.Search("products", "furniture")
	fmt.Printf("Search 'furniture': %v\n", results)

	results = searchIndex.Search("products", "Mouse")
	fmt.Printf("Search 'Mouse' (deleted): %v\n", results)

	// === Check cache ===
	fmt.Println("\n--- Cache (derived from CDC) ---")

	if data, ok := cache.Get("products", "prod-1"); ok {
		fmt.Printf("Cache hit for prod-1: %s\n", string(data))
	}
	if data, ok := cache.Get("products", "prod-3"); ok {
		fmt.Printf("Cache hit for prod-3 (updated): %s\n", string(data))
	}
	if _, ok := cache.Get("products", "prod-2"); !ok {
		fmt.Println("Cache miss for prod-2 (deleted) -- correct!")
	}

	// === Show WAL ===
	fmt.Println("\n--- Write-Ahead Log ---")
	wal := db.GetWAL(0)
	for _, entry := range wal {
		fmt.Printf("  #%d [%s] %s.%s\n", entry.ID, entry.Type, entry.Table, entry.Key)
	}

	close(done)
}
```

### Why CDC Matters

CDC is the bridge between batch and stream worlds. Your database is optimized for OLTP (small, fast queries). But you also need:
- A search index (Elasticsearch) for full-text search
- A cache (Redis) for fast lookups
- A data warehouse for analytics
- Real-time dashboards

Without CDC, you either poll the database (wasteful, delayed) or couple your application to all these systems (brittle). With CDC, changes flow automatically to all derived systems.

---

## 14. Backpressure

### The Slow Consumer Problem

When a producer generates data faster than a consumer can process it, you have three choices:

1. **Drop data**: Simple, but you lose information (acceptable for metrics, not for billing)
2. **Buffer data**: Works until the buffer fills up, then you must choose again
3. **Slow down the producer**: This is backpressure -- the consumer pushes back on the producer

Go channels provide natural backpressure. A send to a full channel blocks the sender. This is exactly backpressure.

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
// Strategy 1: Bounded Buffer (natural channel backpressure)
// ============================================================

func WithBackpressure[T any](in <-chan T, bufSize int) <-chan T {
	out := make(chan T, bufSize)
	go func() {
		defer close(out)
		for v := range in {
			out <- v // blocks when buffer full -- this IS backpressure
		}
	}()
	return out
}

// ============================================================
// Strategy 2: Drop on overflow
// ============================================================

func WithDropOnOverflow[T any](in <-chan T, bufSize int) (<-chan T, *int64) {
	out := make(chan T, bufSize)
	var dropped int64

	go func() {
		defer close(out)
		for v := range in {
			select {
			case out <- v:
				// sent successfully
			default:
				// buffer full, drop the message
				atomic.AddInt64(&dropped, 1)
			}
		}
	}()

	return out, &dropped
}

// ============================================================
// Strategy 3: Rate Limiting
// ============================================================

func WithRateLimit[T any](in <-chan T, rate time.Duration) <-chan T {
	out := make(chan T, 1)
	go func() {
		defer close(out)
		ticker := time.NewTicker(rate)
		defer ticker.Stop()

		for v := range in {
			<-ticker.C // wait for the next tick
			out <- v
		}
	}()
	return out
}

// ============================================================
// Strategy 4: Adaptive Batching
// Collects items until the buffer fills or a timeout expires.
// ============================================================

func WithAdaptiveBatch[T any](in <-chan T, maxBatch int, maxWait time.Duration) <-chan []T {
	out := make(chan []T, 10)
	go func() {
		defer close(out)
		for {
			batch := make([]T, 0, maxBatch)
			timer := time.NewTimer(maxWait)

			// Wait for the first item
			item, ok := <-in
			if !ok {
				timer.Stop()
				return
			}
			batch = append(batch, item)

			// Collect more items until batch is full or timeout
		collect:
			for len(batch) < maxBatch {
				select {
				case item, ok := <-in:
					if !ok {
						break collect
					}
					batch = append(batch, item)
				case <-timer.C:
					break collect
				}
			}

			timer.Stop()
			if len(batch) > 0 {
				out <- batch
			}
		}
	}()
	return out
}

// ============================================================
// Strategy 5: Token Bucket Rate Limiter
// ============================================================

type TokenBucket struct {
	tokens     chan struct{}
	refillRate time.Duration
}

func NewTokenBucket(capacity int, refillRate time.Duration) *TokenBucket {
	tb := &TokenBucket{
		tokens:     make(chan struct{}, capacity),
		refillRate: refillRate,
	}
	// Fill the bucket initially
	for i := 0; i < capacity; i++ {
		tb.tokens <- struct{}{}
	}
	// Start refilling
	go func() {
		ticker := time.NewTicker(refillRate)
		defer ticker.Stop()
		for range ticker.C {
			select {
			case tb.tokens <- struct{}{}:
			default: // bucket full
			}
		}
	}()
	return tb
}

func (tb *TokenBucket) Wait(ctx context.Context) error {
	select {
	case <-tb.tokens:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func WithTokenBucket[T any](ctx context.Context, in <-chan T, bucket *TokenBucket) <-chan T {
	out := make(chan T, 1)
	go func() {
		defer close(out)
		for v := range in {
			if err := bucket.Wait(ctx); err != nil {
				return
			}
			select {
			case out <- v:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

// ============================================================
// Demo: Fast producer, slow consumer
// ============================================================

func main() {
	fmt.Println("=== Backpressure Strategies ===\n")

	// === Strategy 1: Bounded buffer backpressure ===
	fmt.Println("--- Strategy 1: Bounded Buffer Backpressure ---")
	{
		source := make(chan int, 0) // unbuffered -- maximum backpressure
		var produced, consumed int64

		// Fast producer
		go func() {
			defer close(source)
			for i := 0; i < 20; i++ {
				source <- i
				atomic.AddInt64(&produced, 1)
			}
		}()

		// Slow consumer (through backpressure buffer)
		buffered := WithBackpressure(source, 5)
		for v := range buffered {
			time.Sleep(50 * time.Millisecond) // slow processing
			atomic.AddInt64(&consumed, 1)
			_ = v
		}

		fmt.Printf("  Produced: %d, Consumed: %d (no data loss)\n",
			atomic.LoadInt64(&produced), atomic.LoadInt64(&consumed))
	}

	// === Strategy 2: Drop on overflow ===
	fmt.Println("\n--- Strategy 2: Drop on Overflow ---")
	{
		source := make(chan int, 100)
		go func() {
			defer close(source)
			for i := 0; i < 100; i++ {
				source <- i
			}
		}()

		// Very small buffer -- will drop messages
		limited, dropped := WithDropOnOverflow(source, 3)

		var consumed int64
		for v := range limited {
			time.Sleep(10 * time.Millisecond) // slow consumer
			atomic.AddInt64(&consumed, 1)
			_ = v
		}

		fmt.Printf("  Produced: 100, Consumed: %d, Dropped: %d\n",
			atomic.LoadInt64(&consumed), atomic.LoadInt64(dropped))
	}

	// === Strategy 3: Rate limiting ===
	fmt.Println("\n--- Strategy 3: Rate Limiting ---")
	{
		source := make(chan int, 100)
		go func() {
			defer close(source)
			for i := 0; i < 10; i++ {
				source <- i
			}
		}()

		limited := WithRateLimit(source, 50*time.Millisecond)

		start := time.Now()
		count := 0
		for range limited {
			count++
		}
		elapsed := time.Since(start)
		fmt.Printf("  Processed %d items in %v (rate-limited to ~20/sec)\n",
			count, elapsed.Round(time.Millisecond))
	}

	// === Strategy 4: Adaptive batching ===
	fmt.Println("\n--- Strategy 4: Adaptive Batching ---")
	{
		source := make(chan string, 100)
		go func() {
			defer close(source)
			// Burst of messages
			for i := 0; i < 15; i++ {
				source <- fmt.Sprintf("msg-%d", i)
				if i == 7 {
					time.Sleep(200 * time.Millisecond) // pause mid-stream
				}
			}
		}()

		batches := WithAdaptiveBatch(source, 5, 100*time.Millisecond)

		batchNum := 0
		for batch := range batches {
			batchNum++
			fmt.Printf("  Batch %d (%d items): %v\n", batchNum, len(batch), batch)
		}
	}

	// === Strategy 5: Token bucket ===
	fmt.Println("\n--- Strategy 5: Token Bucket ---")
	{
		ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()

		bucket := NewTokenBucket(3, 100*time.Millisecond)

		source := make(chan int, 100)
		go func() {
			defer close(source)
			for i := 0; i < 20; i++ {
				source <- i
			}
		}()

		limited := WithTokenBucket(ctx, source, bucket)

		start := time.Now()
		count := 0
		for range limited {
			count++
		}
		fmt.Printf("  Processed %d items in %v with token bucket\n",
			count, time.Since(start).Round(time.Millisecond))
	}

	// === Demonstrate why backpressure matters ===
	fmt.Println("\n--- Why Backpressure Matters ---")
	{
		// Without backpressure: unbounded memory growth
		fmt.Println("  Without backpressure:")
		fmt.Println("    Producer rate: 10,000/sec")
		fmt.Println("    Consumer rate: 1,000/sec")
		fmt.Println("    After 1 minute: 540,000 items buffered in memory")
		fmt.Println("    Result: OOM kill")
		fmt.Println()
		fmt.Println("  With backpressure (bounded channel):")
		fmt.Println("    Buffer size: 1000")
		fmt.Println("    Producer blocks when buffer full")
		fmt.Println("    Effective rate: 1,000/sec (matches consumer)")
		fmt.Println("    Memory: constant (1000 items max)")
	}

	_ = rand.Int
	_ = sync.Mutex{}
}
```

---

## 15. Real-World Example: Log Processing Pipeline

### Complete Pipeline: Read, Parse, Filter, Aggregate, Output

This section brings everything together. We build a complete log processing pipeline that reads raw log lines, parses them into structured events, filters and enriches them, aggregates metrics over time windows, and outputs results -- all using Go's concurrency primitives.

```go
package main

import (
	"bufio"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"math/rand"
	"sort"
	"strings"
	"sync"
	"time"
)

// ============================================================
// Types
// ============================================================

// RawLogLine is an unparsed log line.
type RawLogLine struct {
	Line      string
	Source    string
	ReadTime time.Time
}

// LogEntry is a parsed, structured log entry.
type LogEntry struct {
	Timestamp  time.Time `json:"timestamp"`
	Level      string    `json:"level"`
	Service    string    `json:"service"`
	Method     string    `json:"method"`
	Path       string    `json:"path"`
	StatusCode int       `json:"status_code"`
	Duration   time.Duration `json:"duration_ms"`
	UserID     string    `json:"user_id,omitempty"`
	RequestID  string    `json:"request_id"`
	Message    string    `json:"message,omitempty"`
}

// Metric is an aggregated metric.
type Metric struct {
	WindowStart time.Time `json:"window_start"`
	WindowEnd   time.Time `json:"window_end"`
	Service     string    `json:"service"`
	Endpoint    string    `json:"endpoint"`
	Count       int       `json:"count"`
	ErrorCount  int       `json:"error_count"`
	AvgDuration float64   `json:"avg_duration_ms"`
	P99Duration float64   `json:"p99_duration_ms"`
	ErrorRate   float64   `json:"error_rate"`
}

// Alert is triggered when metrics exceed thresholds.
type Alert struct {
	Time     time.Time `json:"time"`
	Severity string    `json:"severity"`
	Service  string    `json:"service"`
	Message  string    `json:"message"`
}

// ============================================================
// Stage 1: Log Source (Reader)
// ============================================================

func ReadLogs(ctx context.Context, r io.Reader, source string) <-chan RawLogLine {
	out := make(chan RawLogLine, 100)
	go func() {
		defer close(out)
		scanner := bufio.NewScanner(r)
		for scanner.Scan() {
			select {
			case <-ctx.Done():
				return
			case out <- RawLogLine{
				Line:     scanner.Text(),
				Source:   source,
				ReadTime: time.Now(),
			}:
			}
		}
	}()
	return out
}

// ============================================================
// Stage 2: Parser
// ============================================================

func ParseLogs(ctx context.Context, in <-chan RawLogLine, numWorkers int) <-chan LogEntry {
	out := make(chan LogEntry, 100)
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for raw := range in {
				entry, err := parseLine(raw.Line)
				if err != nil {
					continue // skip unparseable lines
				}
				select {
				case <-ctx.Done():
					return
				case out <- entry:
				}
			}
		}()
	}

	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

func parseLine(line string) (LogEntry, error) {
	// Expected format:
	// 2025-01-15T10:00:05Z INFO api-gateway GET /api/users 200 45ms req-123 user-1
	parts := strings.Fields(line)
	if len(parts) < 8 {
		return LogEntry{}, fmt.Errorf("invalid log format: %s", line)
	}

	ts, err := time.Parse(time.RFC3339, parts[0])
	if err != nil {
		return LogEntry{}, err
	}

	var statusCode int
	fmt.Sscanf(parts[5], "%d", &statusCode)

	durationStr := parts[6]
	durationStr = strings.TrimSuffix(durationStr, "ms")
	var durationMs int
	fmt.Sscanf(durationStr, "%d", &durationMs)

	entry := LogEntry{
		Timestamp:  ts,
		Level:      parts[1],
		Service:    parts[2],
		Method:     parts[3],
		Path:       parts[4],
		StatusCode: statusCode,
		Duration:   time.Duration(durationMs) * time.Millisecond,
		RequestID:  parts[7],
	}

	if len(parts) > 8 {
		entry.UserID = parts[8]
	}

	return entry, nil
}

// ============================================================
// Stage 3: Filter and Enrich
// ============================================================

func FilterAndEnrich(ctx context.Context, in <-chan LogEntry) <-chan LogEntry {
	out := make(chan LogEntry, 100)
	go func() {
		defer close(out)
		for entry := range in {
			// Filter out health check endpoints
			if entry.Path == "/health" || entry.Path == "/ready" {
				continue
			}

			// Enrich: normalize the path (strip IDs)
			entry.Path = normalizePath(entry.Path)

			// Enrich: add a message for errors
			if entry.StatusCode >= 500 {
				entry.Message = "server error"
				entry.Level = "ERROR"
			} else if entry.StatusCode >= 400 {
				entry.Message = "client error"
				entry.Level = "WARN"
			}

			select {
			case <-ctx.Done():
				return
			case out <- entry:
			}
		}
	}()
	return out
}

func normalizePath(path string) string {
	// /api/users/123 -> /api/users/:id
	parts := strings.Split(path, "/")
	for i, part := range parts {
		if isNumeric(part) {
			parts[i] = ":id"
		}
	}
	return strings.Join(parts, "/")
}

func isNumeric(s string) bool {
	if s == "" {
		return false
	}
	for _, ch := range s {
		if ch < '0' || ch > '9' {
			return false
		}
	}
	return true
}

// ============================================================
// Stage 4: Aggregate (Tumbling Window)
// ============================================================

func Aggregate(ctx context.Context, in <-chan LogEntry, windowSize time.Duration) <-chan Metric {
	out := make(chan Metric, 100)
	go func() {
		defer close(out)

		type windowKey struct {
			windowStart time.Time
			service     string
			endpoint    string
		}

		type windowAcc struct {
			count      int
			errorCount int
			durations  []float64
		}

		windows := make(map[windowKey]*windowAcc)

		for entry := range in {
			ws := entry.Timestamp.Truncate(windowSize)
			key := windowKey{
				windowStart: ws,
				service:     entry.Service,
				endpoint:    fmt.Sprintf("%s %s", entry.Method, entry.Path),
			}

			acc, ok := windows[key]
			if !ok {
				acc = &windowAcc{}
				windows[key] = acc
			}

			acc.count++
			if entry.StatusCode >= 500 {
				acc.errorCount++
			}
			acc.durations = append(acc.durations, float64(entry.Duration.Milliseconds()))
		}

		// Emit all window results
		for key, acc := range windows {
			sort.Float64s(acc.durations)

			avgDuration := 0.0
			for _, d := range acc.durations {
				avgDuration += d
			}
			avgDuration /= float64(len(acc.durations))

			p99Idx := int(float64(len(acc.durations)) * 0.99)
			if p99Idx >= len(acc.durations) {
				p99Idx = len(acc.durations) - 1
			}

			metric := Metric{
				WindowStart: key.windowStart,
				WindowEnd:   key.windowStart.Add(windowSize),
				Service:     key.service,
				Endpoint:    key.endpoint,
				Count:       acc.count,
				ErrorCount:  acc.errorCount,
				AvgDuration: avgDuration,
				P99Duration: acc.durations[p99Idx],
				ErrorRate:   float64(acc.errorCount) / float64(acc.count) * 100,
			}

			select {
			case <-ctx.Done():
				return
			case out <- metric:
			}
		}
	}()
	return out
}

// ============================================================
// Stage 5: Alert Detection
// ============================================================

func DetectAlerts(ctx context.Context, in <-chan Metric) (<-chan Metric, <-chan Alert) {
	metrics := make(chan Metric, 100)
	alerts := make(chan Alert, 100)

	go func() {
		defer close(metrics)
		defer close(alerts)

		for metric := range in {
			// Pass through the metric
			select {
			case <-ctx.Done():
				return
			case metrics <- metric:
			}

			// Check for alert conditions
			if metric.ErrorRate > 10 {
				select {
				case alerts <- Alert{
					Time:     metric.WindowEnd,
					Severity: "CRITICAL",
					Service:  metric.Service,
					Message: fmt.Sprintf("High error rate: %.1f%% on %s (window %s-%s)",
						metric.ErrorRate, metric.Endpoint,
						metric.WindowStart.Format("15:04:05"),
						metric.WindowEnd.Format("15:04:05")),
				}:
				case <-ctx.Done():
					return
				}
			}

			if metric.P99Duration > 1000 { // > 1 second
				select {
				case alerts <- Alert{
					Time:     metric.WindowEnd,
					Severity: "WARNING",
					Service:  metric.Service,
					Message: fmt.Sprintf("High latency: p99=%.0fms on %s",
						metric.P99Duration, metric.Endpoint),
				}:
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	return metrics, alerts
}

// ============================================================
// Stage 6: Output (Sink)
// ============================================================

func OutputMetrics(ctx context.Context, metrics <-chan Metric, alerts <-chan Alert) {
	var wg sync.WaitGroup

	// Collect metrics
	var allMetrics []Metric
	wg.Add(1)
	go func() {
		defer wg.Done()
		for m := range metrics {
			allMetrics = append(allMetrics, m)
		}
	}()

	// Collect alerts
	var allAlerts []Alert
	wg.Add(1)
	go func() {
		defer wg.Done()
		for a := range alerts {
			allAlerts = append(allAlerts, a)
		}
	}()

	wg.Wait()

	// Sort metrics by window start, then service, then endpoint
	sort.Slice(allMetrics, func(i, j int) bool {
		if allMetrics[i].WindowStart.Equal(allMetrics[j].WindowStart) {
			if allMetrics[i].Service == allMetrics[j].Service {
				return allMetrics[i].Endpoint < allMetrics[j].Endpoint
			}
			return allMetrics[i].Service < allMetrics[j].Service
		}
		return allMetrics[i].WindowStart.Before(allMetrics[j].WindowStart)
	})

	// Print metrics
	fmt.Println("=== Aggregated Metrics ===\n")
	fmt.Printf("%-23s %-15s %-25s %6s %6s %8s %8s %6s\n",
		"Window", "Service", "Endpoint", "Count", "Errors", "Avg(ms)", "P99(ms)", "Err%")
	fmt.Println(strings.Repeat("-", 110))

	for _, m := range allMetrics {
		fmt.Printf("%-23s %-15s %-25s %6d %6d %8.1f %8.1f %5.1f%%\n",
			m.WindowStart.Format("15:04:05")+"-"+m.WindowEnd.Format("15:04:05"),
			m.Service, m.Endpoint,
			m.Count, m.ErrorCount, m.AvgDuration, m.P99Duration, m.ErrorRate)
	}

	// Print alerts
	if len(allAlerts) > 0 {
		fmt.Println("\n=== Alerts ===\n")
		for _, alert := range allAlerts {
			fmt.Printf("  [%s] %s: %s\n", alert.Severity, alert.Service, alert.Message)
		}
	}

	// Print summary
	fmt.Printf("\n=== Summary ===\n")
	fmt.Printf("  Total metrics: %d\n", len(allMetrics))
	fmt.Printf("  Alerts: %d\n", len(allAlerts))

	totalRequests := 0
	totalErrors := 0
	for _, m := range allMetrics {
		totalRequests += m.Count
		totalErrors += m.ErrorCount
	}
	fmt.Printf("  Total requests: %d\n", totalRequests)
	fmt.Printf("  Total errors: %d\n", totalErrors)
	if totalRequests > 0 {
		fmt.Printf("  Overall error rate: %.2f%%\n", float64(totalErrors)/float64(totalRequests)*100)
	}
}

// ============================================================
// Log Generator (simulates production logs)
// ============================================================

func GenerateLogs(numLines int) string {
	rng := rand.New(rand.NewSource(42))
	baseTime := time.Date(2025, 1, 15, 10, 0, 0, 0, time.UTC)

	services := []string{"api-gateway", "user-service", "payment-service", "auth-service"}
	methods := []string{"GET", "POST", "PUT", "DELETE"}
	paths := []string{
		"/api/users", "/api/users/123", "/api/products", "/api/products/456",
		"/api/orders", "/api/orders/789", "/health", "/api/auth/login",
		"/api/payments", "/api/payments/101",
	}
	levels := []string{"INFO", "INFO", "INFO", "INFO", "WARN", "ERROR"} // weighted toward INFO

	var sb strings.Builder
	for i := 0; i < numLines; i++ {
		ts := baseTime.Add(time.Duration(i) * 500 * time.Millisecond)
		level := levels[rng.Intn(len(levels))]
		service := services[rng.Intn(len(services))]
		method := methods[rng.Intn(len(methods))]
		path := paths[rng.Intn(len(paths))]

		// Status code: mostly 200, some errors
		statusCode := 200
		switch level {
		case "ERROR":
			codes := []int{500, 502, 503}
			statusCode = codes[rng.Intn(len(codes))]
		case "WARN":
			statusCode = 400 + rng.Intn(22) // 400-421
		}

		// Duration: normally fast, occasionally slow
		duration := 20 + rng.Intn(80) // 20-100ms
		if rng.Float64() < 0.05 {
			duration = 500 + rng.Intn(2000) // slow: 500-2500ms
		}

		requestID := fmt.Sprintf("req-%05d", i)
		userID := fmt.Sprintf("user-%d", rng.Intn(100))

		fmt.Fprintf(&sb, "%s %s %s %s %s %d %dms %s %s\n",
			ts.Format(time.RFC3339),
			level, service, method, path,
			statusCode, duration, requestID, userID)
	}

	return sb.String()
}

// ============================================================
// Main: Wire the pipeline
// ============================================================

func main() {
	fmt.Println("=== Log Processing Pipeline ===\n")
	fmt.Println("Pipeline: Read -> Parse -> Filter/Enrich -> Aggregate -> Alert -> Output\n")

	ctx := context.Background()

	// Generate simulated logs
	logData := GenerateLogs(200)
	fmt.Printf("Generated %d log lines\n\n", 200)

	// === Build the pipeline ===

	// Stage 1: Read logs
	rawLogs := ReadLogs(ctx, strings.NewReader(logData), "server-1")

	// Stage 2: Parse (4 parallel workers)
	parsed := ParseLogs(ctx, rawLogs, 4)

	// Stage 3: Filter and enrich
	enriched := FilterAndEnrich(ctx, parsed)

	// Stage 4: Aggregate into 30-second tumbling windows
	aggregated := Aggregate(ctx, enriched, 30*time.Second)

	// Stage 5: Detect alerts
	metrics, alerts := DetectAlerts(ctx, aggregated)

	// Stage 6: Output
	OutputMetrics(ctx, metrics, alerts)

	// === Show the pipeline as JSON ===
	fmt.Println("\n=== Sample Parsed Log Entry (JSON) ===")
	sampleLogs := GenerateLogs(1)
	entry, err := parseLine(strings.TrimSpace(sampleLogs))
	if err == nil {
		jsonBytes, _ := json.MarshalIndent(entry, "  ", "  ")
		fmt.Printf("  %s\n", string(jsonBytes))
	}
}
```

### Pipeline Architecture Recap

```
                                    +----> [Metrics Display]
                                    |
[Log Files] --> [Parser] --> [Filter/Enrich] --> [Aggregator] --> [Alert Detector] -+
                  (4x)          (1x)               (1x)              (1x)           |
                                                                                    +----> [Alert Handler]
```

Each arrow is a Go channel. Each box is a goroutine (or pool of goroutines). Backpressure flows backwards through the channels -- if the aggregator is slow, the filter blocks on send, which causes the parser to block on send, which causes the reader to block on send.

This is the same architecture that production systems like Fluentd, Logstash, and Vector use. The only difference is that production systems add checkpointing (save progress to disk), partitioning (distribute across machines), and durability (write to Kafka between stages).

---

## 16. Key Takeaways

### Batch Processing

1. **Batch processing operates on bounded data.** You know the input size before you start. MapReduce, ETL jobs, and nightly aggregations are batch.

2. **MapReduce is a programming model, not necessarily an optimization.** It provides a framework for distributing work across machines. For in-process work, direct concurrent approaches are often faster.

3. **The Unix pipeline philosophy lives in Go's `io.Reader`/`io.Writer`.** Functions that accept these interfaces compose naturally into data pipelines.

4. **Dataflow DAGs supersede MapReduce** by keeping intermediate data in memory and allowing arbitrary operator graphs instead of just map-then-reduce.

5. **Fault tolerance in batch comes from idempotency.** If a task fails, re-run it. MapReduce tasks read immutable input and write output atomically.

### Stream Processing

6. **Stream processing operates on unbounded data.** Events arrive continuously and you must produce results incrementally.

7. **Event time and processing time are different.** Events can arrive late or out of order. Windows, watermarks, and triggers handle this.

8. **Go channels are natural stream primitives.** A channel is a stream with built-in backpressure. Goroutines are stream operators.

9. **Windowing turns infinite streams into finite chunks.** Tumbling windows for fixed intervals, sliding windows for moving averages, session windows for activity-based grouping.

10. **Backpressure prevents unbounded memory growth.** Bounded channels, rate limiting, and dropping are the three strategies. Go channels provide backpressure by default when bounded.

### Event Sourcing and CQRS

11. **Event sourcing stores what happened, not what is.** Current state is derived by replaying events. You get a complete audit trail and the ability to rebuild state.

12. **CQRS separates the read and write models.** The write model enforces business rules. The read model is optimized for queries. Projectors keep them in sync.

13. **CDC bridges databases and event streams.** It captures changes from the database's write-ahead log and publishes them as events, enabling derived views without polling.

### Architecture

14. **Batch and stream are points on a spectrum.** The same processing logic can often run on both bounded and unbounded input. Design for stream; batch is just a finite stream.

15. **Composition is the key design principle.** Small, focused processing stages connected by channels (or io.Reader/Writer) create systems that are easy to understand, test, and modify.

---

## 17. Practice Exercises

### Exercise 1: Multi-Stage MapReduce (Difficulty: Medium)

Implement a two-stage MapReduce pipeline that first counts word frequencies across documents, then finds the top-K most frequent words. The first MapReduce job produces word counts. The second job takes those counts and emits only the top K.

**Requirements:**
- Stage 1: Standard word count MapReduce
- Stage 2: Take word count output, emit top 10 words
- Both stages run with parallel workers
- Handle at least 100 documents

### Exercise 2: Exactly-Once Stream Processing (Difficulty: Hard)

Extend the Kafka simulation from Section 12 to support exactly-once processing. Implement:
- Idempotent message processing using a deduplication table
- Transactional offset commits (commit offset and processing result atomically)
- A consumer that counts events per user and guarantees each event is counted exactly once, even if the consumer crashes and restarts

**Hints:**
- Store `(offset, result)` pairs atomically
- On restart, resume from the last committed offset
- Use a map as the deduplication table

### Exercise 3: Session Window with Late Events (Difficulty: Hard)

Build a session window processor that:
- Tracks user sessions (gap timeout: 30 seconds)
- Handles late-arriving events by merging them into existing sessions
- Emits updated session results when late events arrive
- Provides a watermark mechanism to finalize sessions

**Test with:** Events arriving 10-60 seconds after the session they belong to.

### Exercise 4: Event Sourcing Snapshots (Difficulty: Medium)

The event sourcing implementation in Section 7 replays all events to rebuild state. For aggregates with thousands of events, this is slow. Implement a snapshot mechanism:
- After every N events, save a snapshot of the aggregate state
- When loading, start from the latest snapshot and replay only subsequent events
- Verify that loading from a snapshot produces the same state as full replay

### Exercise 5: CDC to Full-Text Search Index (Difficulty: Medium)

Extend the CDC example from Section 13 to build a proper full-text search index:
- Tokenize text fields into individual words
- Build an inverted index (word -> set of document keys)
- Support multi-word queries (AND logic: all words must match)
- Handle updates (re-index changed documents) and deletes (remove from index)
- Benchmark search performance with 10,000 documents

### Exercise 6: Backpressure-Aware Pipeline (Difficulty: Medium)

Build a pipeline that monitors its own backpressure and adapts:
- Each stage reports its queue depth (channel fill level)
- When a stage's input buffer is more than 80% full, log a warning
- Implement an "elastic" worker pool for the bottleneck stage that spins up additional goroutines when backpressure is detected and spins them down when pressure subsides
- Visualize the queue depths over time (print a simple ASCII chart)

### Exercise 7: Complete Event-Driven Microservice (Difficulty: Expert)

Combine event sourcing, CQRS, and stream processing to build a mini order-processing system:
- **Commands**: CreateOrder, AddItem, RemoveItem, SubmitOrder, CancelOrder
- **Events**: OrderCreated, ItemAdded, ItemRemoved, OrderSubmitted, OrderCancelled
- **Read models**: (1) Order details view, (2) User order history, (3) Product popularity dashboard
- **Stream processing**: Real-time revenue calculation with 1-minute tumbling windows
- **Alerts**: Trigger an alert if order cancellation rate exceeds 20% in any window

The system should handle concurrent commands on the same order safely (use optimistic concurrency from the event store's version checking).

---

**Next Chapter:** Chapter 40 will explore distributed consensus and coordination in Go -- implementing Raft, distributed locks, leader election, and service discovery from the principles in DDIA Chapters 8 and 9.
