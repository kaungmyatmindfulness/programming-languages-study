# Chapter 40: Observability Fundamentals

Your distributed system is broken. Not obviously broken -- it is not returning 500 errors or crashing. But some users in Europe are seeing 8-second response times, and only when they hit the checkout flow, and only when their cart contains items from two different warehouses. You have dashboards. They are all green. You have alerts. None fired. You have logs. Millions of them. None contain the word "slow" or "timeout." Your monitoring says everything is fine. Your users say otherwise.

This is the gap that observability fills. Monitoring tells you _when_ something is wrong. Observability lets you understand _why_ -- even when the "why" is something you never anticipated, never wrote an alert for, and never imagined could happen.

This chapter is grounded in the principles from *Observability Engineering* by Charity Majors, Liz Fong-Jones, and George Miranda (O'Reilly, 2022), Chapters 1-7. We take those principles and implement them in Go using OpenTelemetry, the industry-standard instrumentation framework. Every code example is complete, runnable, and drawn from patterns used in real production Go services.

Chapter 28 introduced structured logging with `slog`. This chapter goes far beyond logging. We build a complete observability foundation: structured events with high cardinality and dimensionality, distributed traces that propagate across service boundaries, metrics that measure what matters, and the correlation glue that ties all three together. By the end, you will have a reusable observability library and a fully instrumented HTTP API.

---

## Prerequisites

Before reading this chapter, you should be comfortable with:

- **Chapter 12: JSON, HTTP, and REST APIs** -- All examples use HTTP servers and clients
- **Chapter 16: Context Package** -- Trace context propagates via `context.Context`
- **Chapter 21: Building Microservices** -- Service-to-service communication patterns
- **Chapter 25: Middleware Patterns** -- We build tracing and metrics middleware
- **Chapter 28: Structured Logging & Observability** -- `slog` basics, request IDs, structured attributes
- **Chapter 30: Graceful Shutdown** -- Proper shutdown of telemetry providers

You should also have Go 1.22 or later installed. You will need to install OpenTelemetry packages:

```bash
go get go.opentelemetry.io/otel@latest
go get go.opentelemetry.io/otel/sdk@latest
go get go.opentelemetry.io/otel/sdk/metric@latest
go get go.opentelemetry.io/otel/exporters/stdout/stdouttrace@latest
go get go.opentelemetry.io/otel/exporters/stdout/stdoutmetric@latest
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp@latest
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp@latest
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp@latest
```

---

## Table of Contents

1. [What Is Observability?](#1-what-is-observability)
2. [The Three Pillars (and Beyond)](#2-the-three-pillars-and-beyond)
3. [Instrumentation Philosophy](#3-instrumentation-philosophy)
4. [Structured Events](#4-structured-events)
5. [OpenTelemetry in Go](#5-opentelemetry-in-go)
6. [Instrumenting HTTP Servers](#6-instrumenting-http-servers)
7. [Instrumenting HTTP Clients](#7-instrumenting-http-clients)
8. [Instrumenting Database Calls](#8-instrumenting-database-calls)
9. [Custom Spans and Events](#9-custom-spans-and-events)
10. [Log Correlation](#10-log-correlation)
11. [Context Propagation](#11-context-propagation)
12. [Sampling Strategies](#12-sampling-strategies)
13. [Instrumentation Best Practices](#13-instrumentation-best-practices)
14. [Building an Observability Library](#14-building-an-observability-library)
15. [Real-World Example: Fully Instrumented HTTP API](#15-real-world-example-fully-instrumented-http-api)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. What Is Observability?

### Observability vs Monitoring

Monitoring and observability are not synonyms. They are fundamentally different approaches to understanding system behavior.

**Monitoring** is reactive. You decide in advance what questions you might need to answer, and you build dashboards and alerts for those specific questions. "Is CPU above 80%?" "Is error rate above 1%?" "Is P99 latency above 500ms?" Monitoring works well for _known_ failure modes. If you have seen the problem before, you can write a check for it.

**Observability** is exploratory. You instrument your system to emit rich, structured data -- and then you ask arbitrary questions _after_ something goes wrong. "Which users are experiencing slow checkouts?" "What do all the slow requests have in common?" "Is the problem correlated with a particular database shard, a specific feature flag, or a deployment that happened two hours ago?" You do not need to anticipate these questions ahead of time.

The formal definition, borrowed from control theory: **a system is observable if you can determine its internal state by examining its external outputs.** In software terms, that means your telemetry must be rich enough to let you reconstruct what happened inside your system without deploying new code, adding new logging, or SSHing into a production box.

```
+------------------------------------------------------------------+
|                    Monitoring vs Observability                     |
+------------------------------------------------------------------+
| Monitoring                     | Observability                   |
+--------------------------------+---------------------------------+
| Pre-defined questions          | Arbitrary ad-hoc questions      |
| Dashboards and alerts          | Exploration and discovery       |
| "Is X broken?"                 | "WHY is X broken?"              |
| Known failure modes            | Unknown failure modes           |
| Low-cardinality metrics        | High-cardinality events         |
| Aggregated data                | Individual request data         |
| "What is the error rate?"      | "Which user_ids see errors?"    |
| Detect symptoms                | Diagnose root causes            |
+--------------------------------+---------------------------------+
```

### Cardinality and Dimensionality

Two concepts from *Observability Engineering* are critical to understanding why traditional monitoring breaks down.

**Cardinality** is the number of unique values a field can take. The `http_method` field has low cardinality -- maybe 5-10 unique values (GET, POST, PUT, DELETE, PATCH, etc.). The `user_id` field has _high_ cardinality -- millions of unique values. The `trace_id` field has effectively infinite cardinality -- every request gets a unique one.

**Dimensionality** is the number of fields (dimensions) attached to an event. A log line with `timestamp`, `level`, and `message` has low dimensionality (3 fields). A structured event with 50 fields -- user_id, endpoint, status, duration, database queries, cache hits, feature flags, deployment version, region, availability zone -- has high dimensionality.

Traditional metrics systems (Prometheus, StatsD) struggle with high cardinality. If you try to create a metric with a `user_id` label, you create a time series _per user_. A million users means a million time series. Your metrics backend explodes.

Observability requires _both_ high cardinality and high dimensionality. You need to ask "show me all requests from user `abc123` that hit endpoint `/checkout` and took more than 2 seconds, grouped by the database shard they queried." That query involves high-cardinality fields (user_id, trace_id) across many dimensions (endpoint, duration, db_shard).

```go
package main

import (
	"fmt"
)

func main() {
	// Low-cardinality field: few unique values
	// Good for metrics labels, bad for finding specific requests
	lowCardinality := map[string][]string{
		"http_method": {"GET", "POST", "PUT", "DELETE"},
		"status_code": {"200", "201", "400", "404", "500"},
		"environment": {"dev", "staging", "production"},
	}

	// High-cardinality field: many unique values
	// Bad for metrics labels, essential for observability
	highCardinality := map[string]string{
		"user_id":       "usr_a8f3k29x",
		"trace_id":      "4bf92f3577b34da6a3ce929d0e0e4736",
		"session_id":    "sess_9xk2m4p1",
		"request_id":    "req_7h3n5k2m",
		"order_id":      "ord_3j8k2l5n",
		"cart_item_ids": "[item_1, item_2, item_3]",
	}

	// Low dimensionality: few fields per event (traditional logging)
	fmt.Println("=== Low Dimensionality (Traditional Log) ===")
	fmt.Printf("timestamp=2025-03-15T14:32:01Z level=INFO msg=\"request completed\"\n")

	// High dimensionality: many fields per event (observability)
	fmt.Println("\n=== High Dimensionality (Structured Event) ===")
	fmt.Println("An event with 15+ fields allows arbitrary queries:")
	fmt.Println("  trace_id, span_id, parent_id, service, endpoint,")
	fmt.Println("  user_id, method, status, duration_ms, db_queries,")
	fmt.Println("  cache_hits, feature_flags, deploy_version, region, az")

	// The key insight: you need BOTH high cardinality AND
	// high dimensionality to answer arbitrary questions about
	// your system's behavior.

	fmt.Println("\n=== Cardinality Examples ===")
	for field, values := range lowCardinality {
		fmt.Printf("  %s: %d unique values (low cardinality)\n", field, len(values))
	}
	fmt.Println()
	for field, example := range highCardinality {
		fmt.Printf("  %s: millions of unique values, e.g. %s (high cardinality)\n", field, example)
	}
}
```

### The Debugging Loop

Majors, Fong-Jones, and Miranda describe the core debugging workflow that observability enables:

1. **Start broad.** Look at the overall system. Something seems wrong -- latency is up, error rate is elevated.
2. **Narrow down.** Group by dimensions. Is it all endpoints or just `/checkout`? All regions or just `eu-west-1`? All users or just users with carts > 10 items?
3. **Find an exemplar.** Pick one specific request that exhibits the problem. Look at its trace -- every span, every database query, every downstream call.
4. **Compare.** Find a similar request that _did not_ exhibit the problem. What is different? Different database shard? Different cache state? Different feature flag?
5. **Hypothesize and verify.** Form a theory and test it against the data. "Requests hitting shard 3 are slow because that shard is compacting." Check: are all slow requests on shard 3? Are all shard 3 requests slow?

This workflow is only possible if your telemetry has enough cardinality and dimensionality to support it. If all you have is `request_count{status="500"}`, you cannot perform step 2.

### Node.js Comparison

The observability landscape in Node.js is nearly identical to Go in terms of concepts and tooling:

```javascript
// Node.js: OpenTelemetry instrumentation
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { ConsoleSpanExporter } = require('@opentelemetry/sdk-trace-base');
const { Resource } = require('@opentelemetry/resources');

const provider = new NodeTracerProvider({
  resource: new Resource({
    'service.name': 'checkout-service',
    'deployment.environment': 'production',
  }),
});

provider.addSpanProcessor(new SimpleSpanProcessor(new ConsoleSpanExporter()));
provider.register();

// Node has auto-instrumentation for Express, HTTP, pg, etc.
// Go requires explicit instrumentation (or contrib packages).
```

The key difference: Node.js has **auto-instrumentation** that hooks into module loading to instrument Express, `http`, `pg`, `mysql2`, and others automatically. Go does not have this. Go requires you to either use instrumented wrappers (from `go.opentelemetry.io/contrib`) or manually add spans to your code. This is more work, but it gives you more control and avoids the "magic" that can make Node.js auto-instrumentation hard to debug.

---

## 2. The Three Pillars (and Beyond)

### Logs

Logs are discrete events. Each log line records something that happened at a point in time.

```go
package main

import (
	"log/slog"
	"os"
	"time"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	// A log is a discrete event
	logger.Info("order placed",
		slog.String("order_id", "ord_3j8k2l5n"),
		slog.String("user_id", "usr_a8f3k29x"),
		slog.Float64("total", 149.99),
		slog.Int("item_count", 3),
		slog.Duration("processing_time", 245*time.Millisecond),
	)

	// Logs are great for:
	// - Recording discrete events (user signed up, order placed)
	// - Error details with stack context
	// - Audit trails

	// Logs are bad for:
	// - Understanding request flow across services
	// - Aggregating over time (what's the average latency?)
	// - Correlating events without a shared ID
}
```

**Strengths:** Rich context, human-readable, great for discrete events.
**Weaknesses:** Hard to aggregate, expensive to store at high volume, no inherent correlation.

### Metrics

Metrics are numerical measurements aggregated over time. They answer "how much" and "how fast" questions.

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sort"
	"time"
)

// SimpleHistogram is a basic histogram for demonstration.
// In production, use prometheus/client_golang or OpenTelemetry metrics.
type SimpleHistogram struct {
	values []float64
}

func (h *SimpleHistogram) Observe(value float64) {
	h.values = append(h.values, value)
}

func (h *SimpleHistogram) Percentile(p float64) float64 {
	if len(h.values) == 0 {
		return 0
	}
	sorted := make([]float64, len(h.values))
	copy(sorted, h.values)
	sort.Float64s(sorted)
	idx := int(math.Ceil(p/100*float64(len(sorted)))) - 1
	if idx < 0 {
		idx = 0
	}
	return sorted[idx]
}

func (h *SimpleHistogram) Count() int {
	return len(h.values)
}

func (h *SimpleHistogram) Sum() float64 {
	total := 0.0
	for _, v := range h.values {
		total += v
	}
	return total
}

func main() {
	hist := &SimpleHistogram{}

	// Simulate 1000 request durations
	rng := rand.New(rand.NewSource(time.Now().UnixNano()))
	for i := 0; i < 1000; i++ {
		// Most requests are fast (20-100ms)
		// Some are slow (200-500ms)
		// A few are very slow (1000-3000ms)
		duration := 20.0 + rng.Float64()*80.0
		if rng.Float64() < 0.1 {
			duration = 200.0 + rng.Float64()*300.0
		}
		if rng.Float64() < 0.02 {
			duration = 1000.0 + rng.Float64()*2000.0
		}
		hist.Observe(duration)
	}

	// Metrics tell you the shape of your traffic
	fmt.Printf("Request Duration Histogram:\n")
	fmt.Printf("  Count:  %d\n", hist.Count())
	fmt.Printf("  P50:    %.1fms\n", hist.Percentile(50))
	fmt.Printf("  P90:    %.1fms\n", hist.Percentile(90))
	fmt.Printf("  P95:    %.1fms\n", hist.Percentile(95))
	fmt.Printf("  P99:    %.1fms\n", hist.Percentile(99))
	fmt.Printf("  Avg:    %.1fms\n", hist.Sum()/float64(hist.Count()))

	// Metrics are great for:
	// - Alerting (error rate > 1%, P99 > 500ms)
	// - Dashboards (traffic over time)
	// - Capacity planning (requests per second trend)

	// Metrics are bad for:
	// - Debugging individual requests
	// - High-cardinality dimensions (user_id, trace_id)
	// - Understanding WHY something is slow
	fmt.Println("\nMetrics tell you WHAT is happening.")
	fmt.Println("They cannot tell you WHY.")
}
```

**Strengths:** Cheap to store, fast to query, great for alerting and dashboards.
**Weaknesses:** Low cardinality only, aggregated (lose individual request details), cannot answer "why."

### Traces

Traces follow a single request through a distributed system. A trace is a tree of spans, where each span represents a unit of work.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"time"
)

// Span represents a unit of work in a trace.
type Span struct {
	TraceID   string        `json:"trace_id"`
	SpanID    string        `json:"span_id"`
	ParentID  string        `json:"parent_id,omitempty"`
	Service   string        `json:"service"`
	Operation string        `json:"operation"`
	Start     time.Time     `json:"start"`
	Duration  time.Duration `json:"duration"`
	Status    string        `json:"status"`
	Tags      map[string]string `json:"tags,omitempty"`
}

func main() {
	// A trace is a tree of spans sharing the same trace_id
	traceID := "4bf92f3577b34da6a3ce929d0e0e4736"
	now := time.Now()

	spans := []Span{
		{
			TraceID:   traceID,
			SpanID:    "span_001",
			Service:   "api-gateway",
			Operation: "POST /api/checkout",
			Start:     now,
			Duration:  850 * time.Millisecond,
			Status:    "ok",
			Tags:      map[string]string{"http.method": "POST", "user_id": "usr_a8f3k29x"},
		},
		{
			TraceID:   traceID,
			SpanID:    "span_002",
			ParentID:  "span_001",
			Service:   "order-service",
			Operation: "CreateOrder",
			Start:     now.Add(5 * time.Millisecond),
			Duration:  800 * time.Millisecond,
			Status:    "ok",
			Tags:      map[string]string{"order_id": "ord_3j8k2l5n"},
		},
		{
			TraceID:   traceID,
			SpanID:    "span_003",
			ParentID:  "span_002",
			Service:   "order-service",
			Operation: "db.orders.insert",
			Start:     now.Add(10 * time.Millisecond),
			Duration:  45 * time.Millisecond,
			Status:    "ok",
			Tags:      map[string]string{"db.system": "postgresql", "db.statement": "INSERT INTO orders..."},
		},
		{
			TraceID:   traceID,
			SpanID:    "span_004",
			ParentID:  "span_002",
			Service:   "inventory-service",
			Operation: "ReserveItems",
			Start:     now.Add(60 * time.Millisecond),
			Duration:  720 * time.Millisecond, // <-- This is the slow span!
			Status:    "ok",
			Tags:      map[string]string{"item_count": "3", "warehouse": "eu-west-1"},
		},
		{
			TraceID:   traceID,
			SpanID:    "span_005",
			ParentID:  "span_004",
			Service:   "inventory-service",
			Operation: "db.inventory.update",
			Start:     now.Add(65 * time.Millisecond),
			Duration:  700 * time.Millisecond, // <-- Root cause: slow DB query
			Status:    "ok",
			Tags:      map[string]string{"db.system": "postgresql", "db.statement": "UPDATE inventory SET reserved=true..."},
		},
		{
			TraceID:   traceID,
			SpanID:    "span_006",
			ParentID:  "span_002",
			Service:   "payment-service",
			Operation: "ChargeCard",
			Start:     now.Add(785 * time.Millisecond),
			Duration:  10 * time.Millisecond,
			Status:    "ok",
			Tags:      map[string]string{"payment_method": "card", "amount": "149.99"},
		},
	}

	// Visualize the trace as a timeline
	fmt.Println("=== Distributed Trace ===")
	fmt.Printf("Trace ID: %s\n\n", traceID)

	for _, s := range spans {
		indent := ""
		if s.ParentID != "" {
			indent = "  "
			if s.ParentID != "span_001" {
				indent = "    "
			}
		}
		flag := ""
		if s.Duration >= 500*time.Millisecond {
			flag = " *** SLOW ***"
		}
		fmt.Printf("%s[%s] %s/%s  %v%s\n", indent, s.SpanID, s.Service, s.Operation, s.Duration, flag)
	}

	// Print as JSON for reference
	fmt.Println("\n=== Trace as JSON ===")
	enc := json.NewEncoder(os.Stdout)
	enc.SetIndent("", "  ")
	enc.Encode(spans)

	fmt.Println("\nTraces tell you WHERE time was spent.")
	fmt.Println("The slow span (db.inventory.update at 700ms) is the root cause.")
}
```

**Strengths:** Show causality, reveal where time is spent, work across service boundaries.
**Weaknesses:** Expensive to store every trace, no aggregation, hard to search without additional tooling.

### Why the Three Pillars Are Insufficient Alone

*Observability Engineering* makes a compelling argument: logs, metrics, and traces are necessary but not sufficient. The real unit of observability is the **structured event** -- a single, wide, arbitrarily rich blob of data that represents one unit of work (typically one request or one operation).

The three pillars fail when they are treated as separate systems:
- Logs in Elasticsearch, metrics in Prometheus, traces in Jaeger -- three UIs, three query languages, no correlation.
- A metric tells you latency is high. You switch to traces to find a slow trace. You switch to logs to find the error message. Each switch costs time and context.

The ideal is a single system where each event has all the context you need: trace IDs, metric values, log messages, and arbitrary high-cardinality fields. In practice, OpenTelemetry is converging the three pillars into a unified signal model, and we will use it throughout this chapter.

---

## 3. Instrumentation Philosophy

### Instrument at the Boundaries

The highest-value instrumentation points are at the boundaries of your system -- where it interacts with the outside world. Every boundary crossing is a place where latency, errors, and unexpected behavior can appear.

```
+-------------------------------------------------------------------+
|                    Instrumentation Boundaries                      |
+-------------------------------------------------------------------+
| Boundary                        | What to capture                 |
+---------------------------------+---------------------------------+
| Incoming HTTP request           | Method, path, status, duration, |
|                                 | request/response size, user_id  |
+---------------------------------+---------------------------------+
| Outgoing HTTP request           | URL, method, status, duration,  |
|                                 | retry count                     |
+---------------------------------+---------------------------------+
| Database query                  | Query type, table, duration,    |
|                                 | rows affected, error            |
+---------------------------------+---------------------------------+
| Cache operation                 | Operation (get/set), key prefix,|
|                                 | hit/miss, duration              |
+---------------------------------+---------------------------------+
| Message queue publish/consume   | Queue name, message type,       |
|                                 | processing duration             |
+---------------------------------+---------------------------------+
| gRPC call (client or server)    | Service, method, status, dur.   |
+---------------------------------+---------------------------------+
| File system I/O                 | Operation, path, size, duration |
+---------------------------------+---------------------------------+
| External API call               | Service, endpoint, status, dur. |
+---------------------------------+---------------------------------+
```

### What to Instrument

The cost of instrumentation is not zero. Every span you create, every metric you record, every log you emit adds CPU cycles, memory allocation, and network bandwidth. But the cost of _not_ instrumenting is an outage you cannot diagnose.

**Always instrument:**
- Every incoming request (HTTP, gRPC, message)
- Every outgoing request (HTTP calls, database queries, cache operations)
- Business-critical operations (payment processing, order creation)
- Error paths (with full context)

**Instrument selectively:**
- Internal function calls (only when they represent significant work)
- Loop iterations (only if the loop is the bottleneck)
- Utility functions (usually not worth it)

**Never instrument:**
- Tight inner loops (the overhead will dominate)
- Trivial getters/setters
- Pure computation that takes microseconds

### The Cost of Instrumentation

```go
package main

import (
	"context"
	"fmt"
	"testing"
	"time"
)

// Simulated span creation overhead
func createSpan(ctx context.Context, name string) (context.Context, func()) {
	start := time.Now()
	// In a real tracer, this allocates a span struct, generates span ID,
	// records parent, and stores in context.
	ctx = context.WithValue(ctx, name, start)
	return ctx, func() {
		_ = time.Since(start)
	}
}

// BenchmarkNoInstrumentation measures raw function call overhead.
func BenchmarkNoInstrumentation(b *testing.B) {
	for i := 0; i < b.N; i++ {
		result := doWork()
		_ = result
	}
}

// BenchmarkWithSpan measures function call + span creation overhead.
func BenchmarkWithSpan(b *testing.B) {
	ctx := context.Background()
	for i := 0; i < b.N; i++ {
		_, end := createSpan(ctx, "doWork")
		result := doWork()
		_ = result
		end()
	}
}

func doWork() int {
	sum := 0
	for i := 0; i < 100; i++ {
		sum += i
	}
	return sum
}

func main() {
	// Demonstrate the relative cost of instrumentation
	const iterations = 1_000_000

	// Without instrumentation
	start := time.Now()
	for i := 0; i < iterations; i++ {
		_ = doWork()
	}
	withoutInstr := time.Since(start)

	// With instrumentation
	ctx := context.Background()
	start = time.Now()
	for i := 0; i < iterations; i++ {
		_, end := createSpan(ctx, "doWork")
		_ = doWork()
		end()
	}
	withInstr := time.Since(start)

	overhead := float64(withInstr-withoutInstr) / float64(withoutInstr) * 100

	fmt.Printf("Without instrumentation: %v (%d iterations)\n", withoutInstr, iterations)
	fmt.Printf("With instrumentation:    %v (%d iterations)\n", withInstr, iterations)
	fmt.Printf("Overhead:                %.1f%%\n", overhead)
	fmt.Println()
	fmt.Println("Key insight: instrumentation overhead is negligible compared")
	fmt.Println("to real I/O (database queries take 1-100ms, HTTP calls take")
	fmt.Println("10-1000ms). Instrument at boundaries where I/O happens.")
}
```

### Node.js Comparison

In Node.js, auto-instrumentation handles boundary instrumentation automatically:

```javascript
// Node.js: auto-instrumentation wraps require() to instrument libraries
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { PgInstrumentation } = require('@opentelemetry/instrumentation-pg');

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),     // Auto-instruments http/https
    new ExpressInstrumentation(),  // Auto-instruments Express routes
    new PgInstrumentation(),       // Auto-instruments pg queries
  ],
});
// After this, every HTTP request, Express route, and pg query
// automatically creates spans. No code changes needed.
```

In Go, you must explicitly wrap or instrument each boundary:

```go
// Go: explicit instrumentation at each boundary
// HTTP server: use otelhttp middleware
// HTTP client: use otelhttp.NewTransport()
// Database: wrap with spans manually or use contrib packages
// gRPC: use otelgrpc interceptors

// This is more work but has two advantages:
// 1. No hidden magic -- you see exactly what is instrumented
// 2. Full control over what attributes are added to spans
```

---

## 4. Structured Events

### The Core Unit of Observability

*Observability Engineering* argues that the fundamental unit of observability is not a log line, a metric point, or a trace span -- it is a **structured event**. An event is a wide, arbitrarily rich record of one unit of work.

A structured event has:
- **High cardinality**: Fields like `user_id`, `order_id`, `trace_id` with millions of unique values
- **High dimensionality**: Many fields (20-50+) providing rich context
- **A timestamp**: When the event occurred
- **A duration**: How long the work took

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"time"
)

// Event represents a structured observability event.
// This is the core data structure for observability.
// It corresponds to what Charity Majors calls a "wide event" --
// a single blob of data with enough context to answer arbitrary questions.
type Event struct {
	// Trace context -- connects this event to a distributed trace
	TraceID  string `json:"trace_id"`
	SpanID   string `json:"span_id"`
	ParentID string `json:"parent_id,omitempty"`

	// Identity -- what service and operation produced this event
	ServiceName string `json:"service_name"`
	Name        string `json:"name"`

	// Timing -- when it happened and how long it took
	Timestamp time.Time     `json:"timestamp"`
	Duration  time.Duration `json:"duration_ms"`
	Status    string        `json:"status"` // "ok", "error"

	// Arbitrary fields -- the high-cardinality, high-dimensionality data
	// that makes observability possible
	Fields map[string]any `json:"fields"`
}

// NewEvent creates a new Event with initialized fields.
func NewEvent(service, name string) *Event {
	return &Event{
		ServiceName: service,
		Name:        name,
		Timestamp:   time.Now(),
		Status:      "ok",
		Fields:      make(map[string]any),
	}
}

// Set adds a field to the event.
func (e *Event) Set(key string, value any) {
	e.Fields[key] = value
}

// SetError marks the event as errored and records the error.
func (e *Event) SetError(err error) {
	e.Status = "error"
	e.Fields["error"] = err.Error()
}

// Finish records the duration.
func (e *Event) Finish() {
	e.Duration = time.Since(e.Timestamp)
}

func main() {
	// Build a rich structured event for an HTTP request
	event := NewEvent("checkout-service", "POST /api/checkout")
	event.TraceID = "4bf92f3577b34da6a3ce929d0e0e4736"
	event.SpanID = "00f067aa0ba902b7"

	// Request context (high cardinality)
	event.Set("http.method", "POST")
	event.Set("http.url", "/api/checkout")
	event.Set("http.status_code", 200)
	event.Set("http.request_content_length", 1024)
	event.Set("http.response_content_length", 256)

	// User context (high cardinality)
	event.Set("user.id", "usr_a8f3k29x")
	event.Set("user.email_domain", "example.com")
	event.Set("user.plan", "premium")
	event.Set("user.country", "DE")

	// Business context
	event.Set("order.id", "ord_3j8k2l5n")
	event.Set("order.total", 149.99)
	event.Set("order.item_count", 3)
	event.Set("order.currency", "EUR")
	event.Set("order.warehouse_count", 2)

	// Infrastructure context
	event.Set("infra.region", "eu-west-1")
	event.Set("infra.az", "eu-west-1a")
	event.Set("infra.instance_id", "i-0abc123def456")
	event.Set("infra.deploy_version", "v2.34.1")
	event.Set("infra.deploy_sha", "a1b2c3d")

	// Performance context
	event.Set("db.query_count", 4)
	event.Set("db.total_duration_ms", 89)
	event.Set("cache.hits", 2)
	event.Set("cache.misses", 1)
	event.Set("downstream.calls", 2)
	event.Set("downstream.total_duration_ms", 720)

	// Feature flags
	event.Set("feature.new_checkout_flow", true)
	event.Set("feature.warehouse_optimization", false)

	// Finish the event
	time.Sleep(10 * time.Millisecond) // simulate work
	event.Finish()

	// Output as JSON
	enc := json.NewEncoder(os.Stdout)
	enc.SetIndent("", "  ")
	enc.Encode(event)

	fmt.Println("\n=== Why This Matters ===")
	fmt.Println("With this single event, you can answer:")
	fmt.Println("  - 'Show me all slow checkouts from German users'")
	fmt.Println("  - 'Do multi-warehouse orders take longer?'")
	fmt.Println("  - 'Is the new checkout flow faster than the old one?'")
	fmt.Println("  - 'Which deploy version introduced the regression?'")
	fmt.Println("  - 'Are cache misses correlated with slow requests?'")
	fmt.Println()
	fmt.Println("You do NOT need to anticipate these questions ahead of time.")
	fmt.Println("You just need rich, high-cardinality, high-dimensionality events.")
}
```

### Building an Event Emitter

In practice, you need a way to build events throughout the lifetime of a request and emit them when the request completes.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"sync"
	"time"
)

// EventBuilder accumulates fields throughout a request's lifetime.
// It is safe for concurrent use because a request might spawn
// goroutines that all want to add fields.
type EventBuilder struct {
	mu     sync.Mutex
	event  map[string]any
	start  time.Time
	onSend func(map[string]any)
}

// NewEventBuilder creates a new builder.
func NewEventBuilder(onSend func(map[string]any)) *EventBuilder {
	return &EventBuilder{
		event:  make(map[string]any),
		start:  time.Now(),
		onSend: onSend,
	}
}

// Set adds or overwrites a field. Safe for concurrent use.
func (b *EventBuilder) Set(key string, value any) {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.event[key] = value
}

// SetIfNotExists adds a field only if it does not already exist.
func (b *EventBuilder) SetIfNotExists(key string, value any) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if _, exists := b.event[key]; !exists {
		b.event[key] = value
	}
}

// Incr increments a numeric field. Useful for counters
// like "db.query_count" or "cache.hits".
func (b *EventBuilder) Incr(key string) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if v, ok := b.event[key]; ok {
		if count, ok := v.(int); ok {
			b.event[key] = count + 1
			return
		}
	}
	b.event[key] = 1
}

// AddDuration adds a duration to a cumulative duration field.
func (b *EventBuilder) AddDuration(key string, d time.Duration) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if v, ok := b.event[key]; ok {
		if existing, ok := v.(time.Duration); ok {
			b.event[key] = existing + d
			return
		}
	}
	b.event[key] = d
}

// Send emits the event. Call this once, when the request completes.
func (b *EventBuilder) Send() {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.event["duration_ms"] = time.Since(b.start).Milliseconds()
	b.event["timestamp"] = b.start.Format(time.RFC3339Nano)
	if b.onSend != nil {
		b.onSend(b.event)
	}
}

// contextKey is an unexported type for context keys.
type contextKey struct{}

// WithEventBuilder stores an EventBuilder in the context.
func WithEventBuilder(ctx context.Context, eb *EventBuilder) context.Context {
	return context.WithValue(ctx, contextKey{}, eb)
}

// EventBuilderFromContext retrieves the EventBuilder from the context.
func EventBuilderFromContext(ctx context.Context) *EventBuilder {
	if eb, ok := ctx.Value(contextKey{}).(*EventBuilder); ok {
		return eb
	}
	// Return a no-op builder to avoid nil checks everywhere
	return NewEventBuilder(nil)
}

func main() {
	// The emitter sends events to stdout as JSON.
	emitter := func(event map[string]any) {
		enc := json.NewEncoder(os.Stdout)
		enc.SetIndent("", "  ")
		enc.Encode(event)
	}

	// Simulate a request lifecycle
	eb := NewEventBuilder(emitter)
	ctx := WithEventBuilder(context.Background(), eb)

	// HTTP middleware adds request fields
	eb.Set("service", "checkout-service")
	eb.Set("http.method", "POST")
	eb.Set("http.path", "/api/checkout")

	// Auth middleware adds user fields
	eb.Set("user.id", "usr_a8f3k29x")
	eb.Set("user.plan", "premium")

	// Handler adds business fields
	eb.Set("order.item_count", 3)
	eb.Set("order.total", 149.99)

	// Database layer adds performance fields
	simulateDBQuery(ctx)
	simulateDBQuery(ctx)

	// Cache layer adds its own fields
	eb.Incr("cache.hit")
	eb.Incr("cache.hit")
	eb.Incr("cache.miss")

	// Response is written
	eb.Set("http.status_code", 200)

	// Emit the event when the request completes
	eb.Send()
}

func simulateDBQuery(ctx context.Context) {
	eb := EventBuilderFromContext(ctx)
	start := time.Now()
	time.Sleep(5 * time.Millisecond) // simulate query
	eb.Incr("db.query_count")
	eb.AddDuration("db.total_duration", time.Since(start))
}
```

---

## 5. OpenTelemetry in Go

### What Is OpenTelemetry?

OpenTelemetry (OTel) is a vendor-neutral, open-source observability framework. It provides APIs, SDKs, and tools for generating, collecting, and exporting telemetry data (traces, metrics, and logs). It is the merger of OpenTracing and OpenCensus and has become the industry standard.

The architecture is layered:

```
+---------------------------------------------------------------+
|                    Your Application Code                       |
|  otel.Tracer("myapp").Start(ctx, "operation")                 |
+-----------------------------+---------------------------------+
                              |
+-----------------------------v---------------------------------+
|                        OTel API                                |
|  Stable interfaces: Tracer, Meter, Logger                      |
|  go.opentelemetry.io/otel                                      |
+-----------------------------+---------------------------------+
                              |
+-----------------------------v---------------------------------+
|                        OTel SDK                                |
|  Implementation: TracerProvider, SpanProcessor, Sampler         |
|  go.opentelemetry.io/otel/sdk                                  |
+-----------------------------+---------------------------------+
                              |
+-----------------------------v---------------------------------+
|                       Exporters                                |
|  OTLP, Jaeger, Zipkin, Console, Prometheus                     |
|  go.opentelemetry.io/otel/exporters/*                          |
+-----------------------------+---------------------------------+
                              |
+-----------------------------v---------------------------------+
|                      Collector / Backend                       |
|  OTel Collector, Jaeger, Honeycomb, Datadog, Grafana Tempo     |
+---------------------------------------------------------------+
```

### SDK Setup: TracerProvider

The `TracerProvider` is the central configuration point for tracing. It manages the lifecycle of spans, sampling, and exporting.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initTracer() (*sdktrace.TracerProvider, error) {
	// 1. Create an exporter.
	// For development, export to stdout.
	// For production, use OTLP exporter to send to a collector.
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, fmt.Errorf("creating stdout exporter: %w", err)
	}

	// 2. Create a resource.
	// A resource describes the entity producing telemetry.
	// This is attached to every span emitted by this provider.
	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("checkout-service"),
			semconv.ServiceVersionKey.String("v2.34.1"),
			semconv.DeploymentEnvironmentKey.String("production"),
			attribute.String("team", "platform"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	// 3. Create the TracerProvider.
	tp := sdktrace.NewTracerProvider(
		// Use a BatchSpanProcessor for production.
		// It batches spans and sends them in bulk, reducing overhead.
		sdktrace.WithBatcher(exporter,
			sdktrace.WithMaxQueueSize(2048),
			sdktrace.WithMaxExportBatchSize(512),
			sdktrace.WithBatchTimeout(5*time.Second),
		),
		sdktrace.WithResource(res),
		// Sample 100% of traces in this example.
		// In production, use a more selective sampler.
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
	)

	// 4. Register as the global TracerProvider.
	// This lets any code call otel.Tracer("name") to get a tracer.
	otel.SetTracerProvider(tp)

	return tp, nil
}

func main() {
	// Initialize the tracer
	tp, err := initTracer()
	if err != nil {
		log.Fatalf("Failed to initialize tracer: %v", err)
	}
	// Shutdown flushes any remaining spans and releases resources.
	// This is critical -- without it, you lose the last batch of spans.
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		if err := tp.Shutdown(ctx); err != nil {
			log.Printf("Error shutting down tracer provider: %v", err)
		}
	}()

	// Create a tracer. The name identifies the instrumentation library.
	tracer := otel.Tracer("github.com/myorg/checkout-service")

	// Create a span
	ctx, span := tracer.Start(context.Background(), "ProcessCheckout")
	defer span.End()

	// Add attributes to the span
	span.SetAttributes(
		attribute.String("user.id", "usr_a8f3k29x"),
		attribute.Float64("order.total", 149.99),
		attribute.Int("order.item_count", 3),
	)

	// Do some work
	processItems(ctx, tracer)

	fmt.Println("Trace emitted to stdout (see JSON output above)")
}

func processItems(ctx context.Context, tracer otel.Tracer) {
	// Create a child span. It automatically becomes a child of the
	// span in the context.
	_, span := tracer.Start(ctx, "ProcessItems")
	defer span.End()

	span.SetAttributes(attribute.Int("item.count", 3))

	// Simulate work
	time.Sleep(10 * time.Millisecond)
}
```

### SDK Setup: MeterProvider

Metrics in OpenTelemetry use a `MeterProvider` analogous to the `TracerProvider`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutmetric"
	"go.opentelemetry.io/otel/metric"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initMeter() (*sdkmetric.MeterProvider, error) {
	exporter, err := stdoutmetric.New(stdoutmetric.WithPrettyPrint())
	if err != nil {
		return nil, fmt.Errorf("creating stdout metric exporter: %w", err)
	}

	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("checkout-service"),
			semconv.ServiceVersionKey.String("v2.34.1"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	mp := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(
			sdkmetric.NewPeriodicReader(exporter,
				sdkmetric.WithInterval(10*time.Second),
			),
		),
		sdkmetric.WithResource(res),
	)

	otel.SetMeterProvider(mp)
	return mp, nil
}

func main() {
	mp, err := initMeter()
	if err != nil {
		log.Fatalf("Failed to initialize meter: %v", err)
	}
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		if err := mp.Shutdown(ctx); err != nil {
			log.Printf("Error shutting down meter provider: %v", err)
		}
	}()

	meter := otel.Meter("github.com/myorg/checkout-service")

	// Counter: counts things that only go up
	requestCount, err := meter.Int64Counter("http.request.count",
		metric.WithDescription("Total number of HTTP requests"),
		metric.WithUnit("{request}"),
	)
	if err != nil {
		log.Fatal(err)
	}

	// Histogram: records distribution of values
	requestDuration, err := meter.Float64Histogram("http.request.duration",
		metric.WithDescription("HTTP request duration in milliseconds"),
		metric.WithUnit("ms"),
		metric.WithExplicitBucketBoundaries(5, 10, 25, 50, 100, 250, 500, 1000),
	)
	if err != nil {
		log.Fatal(err)
	}

	// UpDownCounter: gauge-like counter (can go up or down)
	activeRequests, err := meter.Int64UpDownCounter("http.request.active",
		metric.WithDescription("Number of active HTTP requests"),
		metric.WithUnit("{request}"),
	)
	if err != nil {
		log.Fatal(err)
	}

	// Simulate some HTTP traffic
	ctx := context.Background()
	rng := rand.New(rand.NewSource(time.Now().UnixNano()))

	methods := []string{"GET", "POST", "PUT", "DELETE"}
	paths := []string{"/api/orders", "/api/users", "/api/products"}
	statuses := []int{200, 200, 200, 200, 201, 400, 404, 500}

	for i := 0; i < 100; i++ {
		method := methods[rng.Intn(len(methods))]
		path := paths[rng.Intn(len(paths))]
		status := statuses[rng.Intn(len(statuses))]
		duration := 10.0 + rng.Float64()*200.0

		attrs := []attribute.KeyValue{
			attribute.String("http.method", method),
			attribute.String("http.route", path),
			attribute.Int("http.status_code", status),
		}

		// Record the request
		activeRequests.Add(ctx, 1, metric.WithAttributes(attrs...))
		requestCount.Add(ctx, 1, metric.WithAttributes(attrs...))
		requestDuration.Record(ctx, duration, metric.WithAttributes(attrs...))
		activeRequests.Add(ctx, -1, metric.WithAttributes(attrs...))
	}

	fmt.Println("Recorded 100 simulated requests with metrics.")
	fmt.Println("Metrics will be exported on the next periodic flush.")

	// Force a flush so we see output
	if err := mp.ForceFlush(ctx); err != nil {
		log.Printf("Error flushing metrics: %v", err)
	}
}
```

### OTLP Exporter for Production

For production, you send telemetry to an OpenTelemetry Collector or a backend that speaks OTLP (OpenTelemetry Protocol).

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initOTLPTracer() (*sdktrace.TracerProvider, error) {
	ctx := context.Background()

	// The OTLP exporter sends spans to an OpenTelemetry Collector
	// or any OTLP-compatible backend (Honeycomb, Grafana Tempo, etc.)
	endpoint := os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
	if endpoint == "" {
		endpoint = "localhost:4318" // default OTLP HTTP port
	}

	exporter, err := otlptracehttp.New(ctx,
		otlptracehttp.WithEndpoint(endpoint),
		otlptracehttp.WithInsecure(), // Use WithTLSCredentials in production
	)
	if err != nil {
		return nil, fmt.Errorf("creating OTLP exporter: %w", err)
	}

	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("checkout-service"),
			semconv.ServiceVersionKey.String("v2.34.1"),
			semconv.DeploymentEnvironmentKey.String(
				getEnv("DEPLOY_ENV", "development"),
			),
			attribute.String("team", "platform"),
			attribute.String("region", getEnv("AWS_REGION", "us-east-1")),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter,
			sdktrace.WithMaxQueueSize(4096),
			sdktrace.WithMaxExportBatchSize(512),
			sdktrace.WithBatchTimeout(5*time.Second),
		),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.ParentBased(
			sdktrace.TraceIDRatioBased(0.1), // Sample 10% of new traces
		)),
	)

	otel.SetTracerProvider(tp)
	return tp, nil
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

func main() {
	tp, err := initOTLPTracer()
	if err != nil {
		log.Fatalf("Failed to initialize OTLP tracer: %v", err)
	}
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		if err := tp.Shutdown(ctx); err != nil {
			log.Printf("Error shutting down tracer: %v", err)
		}
	}()

	tracer := otel.Tracer("checkout-service")
	ctx, span := tracer.Start(context.Background(), "main")
	defer span.End()

	span.SetAttributes(attribute.String("startup", "complete"))
	fmt.Println("OTLP tracer initialized. Spans will be sent to", os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT"))
	_ = ctx
}
```

---

## 6. Instrumenting HTTP Servers

### Tracing Middleware

The most important place to instrument is the HTTP server entry point. Every incoming request should create a span automatically.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// responseWriter wraps http.ResponseWriter to capture the status code.
type responseWriter struct {
	http.ResponseWriter
	statusCode  int
	wroteHeader bool
	bytesWritten int
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if !rw.wroteHeader {
		rw.statusCode = code
		rw.wroteHeader = true
		rw.ResponseWriter.WriteHeader(code)
	}
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// TracingMiddleware creates a span for every incoming HTTP request.
// It extracts trace context from incoming headers (for distributed tracing)
// and injects rich attributes into the span.
func TracingMiddleware(serviceName string) func(http.Handler) http.Handler {
	tracer := otel.Tracer(serviceName)
	propagator := otel.GetTextMapPropagator()

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Extract trace context from incoming request headers.
			// If this request was initiated by another service, the trace_id
			// and parent span_id will be in the headers.
			ctx := propagator.Extract(r.Context(), propagation.HeaderCarrier(r.Header))

			// Create a new span for this request.
			// The span name should be the route pattern, not the raw URL
			// (to avoid high-cardinality span names).
			spanName := fmt.Sprintf("%s %s", r.Method, r.URL.Path)
			ctx, span := tracer.Start(ctx, spanName,
				trace.WithSpanKind(trace.SpanKindServer),
				trace.WithAttributes(
					semconv.HTTPMethodKey.String(r.Method),
					semconv.HTTPTargetKey.String(r.URL.Path),
					semconv.HTTPSchemeKey.String(r.URL.Scheme),
					attribute.String("http.host", r.Host),
					attribute.String("http.user_agent", r.UserAgent()),
					attribute.String("http.client_ip", r.RemoteAddr),
					attribute.Int64("http.request_content_length", r.ContentLength),
				),
			)
			defer span.End()

			// Wrap the response writer to capture status code and bytes written
			wrapped := newResponseWriter(w)

			// Serve the request with the traced context
			next.ServeHTTP(wrapped, r.WithContext(ctx))

			// After the handler completes, add response attributes
			span.SetAttributes(
				semconv.HTTPStatusCodeKey.Int(wrapped.statusCode),
				attribute.Int("http.response_content_length", wrapped.bytesWritten),
			)

			// Mark span as error if status code indicates a server error
			if wrapped.statusCode >= 500 {
				span.SetStatus(codes.Error, fmt.Sprintf("HTTP %d", wrapped.statusCode))
			}
		})
	}
}

// --- Sample application using the middleware ---

func initTracer() (*sdktrace.TracerProvider, error) {
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, err
	}

	res := resource.NewWithAttributes(
		semconv.SchemaURL,
		semconv.ServiceNameKey.String("order-service"),
		semconv.ServiceVersionKey.String("v1.0.0"),
	)

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
	)

	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	))

	return tp, nil
}

type Order struct {
	ID     string  `json:"id"`
	UserID string  `json:"user_id"`
	Total  float64 `json:"total"`
	Items  int     `json:"items"`
}

func handleGetOrder(w http.ResponseWriter, r *http.Request) {
	// The span is already created by the middleware.
	// We can add business-specific attributes to it.
	span := trace.SpanFromContext(r.Context())
	span.SetAttributes(
		attribute.String("order.id", "ord_123"),
		attribute.String("user.id", "usr_abc"),
	)

	// Simulate database lookup
	time.Sleep(20 * time.Millisecond)

	order := Order{
		ID:     "ord_123",
		UserID: "usr_abc",
		Total:  149.99,
		Items:  3,
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(order)
}

func handleCreateOrder(w http.ResponseWriter, r *http.Request) {
	span := trace.SpanFromContext(r.Context())

	var order Order
	if err := json.NewDecoder(r.Body).Decode(&order); err != nil {
		span.SetStatus(codes.Error, "invalid request body")
		span.RecordError(err)
		http.Error(w, `{"error":"invalid request body"}`, http.StatusBadRequest)
		return
	}

	span.SetAttributes(
		attribute.Float64("order.total", order.Total),
		attribute.Int("order.items", order.Items),
	)

	// Simulate processing
	time.Sleep(50 * time.Millisecond)

	order.ID = "ord_" + fmt.Sprintf("%d", time.Now().UnixMilli())

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(order)
}

func handleHealth(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status":"ok"}`))
}

func main() {
	tp, err := initTracer()
	if err != nil {
		log.Fatalf("Failed to initialize tracer: %v", err)
	}
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/orders/{id}", handleGetOrder)
	mux.HandleFunc("POST /api/orders", handleCreateOrder)
	mux.HandleFunc("GET /health", handleHealth)

	// Wrap with tracing middleware
	tracedMux := TracingMiddleware("order-service")(mux)

	server := &http.Server{
		Addr:         ":8080",
		Handler:      tracedMux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	fmt.Println("Server starting on :8080")
	fmt.Println("Try: curl http://localhost:8080/api/orders/ord_123")
	fmt.Println("Try: curl -X POST -d '{\"total\":99.99,\"items\":2}' http://localhost:8080/api/orders")
	log.Fatal(server.ListenAndServe())
}
```

### Using otelhttp (The Official Contrib Package)

Instead of writing your own middleware, you can use the official `otelhttp` package which handles many edge cases:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

func initTracer() (*sdktrace.TracerProvider, error) {
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, err
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("product-service"),
		)),
	)

	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp, nil
}

type Product struct {
	ID    string  `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}

func handleGetProduct(w http.ResponseWriter, r *http.Request) {
	span := trace.SpanFromContext(r.Context())
	span.SetAttributes(attribute.String("product.id", "prod_001"))

	product := Product{
		ID:    "prod_001",
		Name:  "Go in Action",
		Price: 39.99,
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(product)
}

func handleListProducts(w http.ResponseWriter, r *http.Request) {
	span := trace.SpanFromContext(r.Context())
	span.SetAttributes(attribute.Int("product.count", 3))

	products := []Product{
		{ID: "prod_001", Name: "Go in Action", Price: 39.99},
		{ID: "prod_002", Name: "Observability Engineering", Price: 49.99},
		{ID: "prod_003", Name: "Distributed Systems", Price: 59.99},
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(products)
}

func main() {
	tp, err := initTracer()
	if err != nil {
		log.Fatal(err)
	}
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	mux := http.NewServeMux()

	// otelhttp.NewHandler wraps each handler with automatic span creation.
	// It handles:
	// - Span creation with HTTP semantic conventions
	// - Context propagation from incoming headers
	// - Status code and error recording
	// - Request/response size tracking
	// - Metric recording (request count, duration, size)
	mux.Handle("GET /api/products/{id}",
		otelhttp.NewHandler(
			http.HandlerFunc(handleGetProduct),
			"GET /api/products/{id}", // operation name
		),
	)

	mux.Handle("GET /api/products",
		otelhttp.NewHandler(
			http.HandlerFunc(handleListProducts),
			"GET /api/products",
		),
	)

	// You can also wrap the entire mux for blanket instrumentation
	handler := otelhttp.NewHandler(mux, "product-service",
		otelhttp.WithMessageEvents(otelhttp.ReadEvents, otelhttp.WriteEvents),
	)

	server := &http.Server{
		Addr:    ":8081",
		Handler: handler,
	}

	fmt.Println("Product service starting on :8081")
	log.Fatal(server.ListenAndServe())
}
```

### Node.js Comparison

```javascript
// Node.js with Express + OpenTelemetry auto-instrumentation
// Auto-instrumentation handles this automatically -- no middleware needed.
// But you can also use manual instrumentation:

const { trace } = require('@opentelemetry/api');

app.use((req, res, next) => {
  const span = trace.getActiveSpan();
  if (span) {
    span.setAttribute('user.id', req.user?.id);
    span.setAttribute('http.route', req.route?.path);
  }
  next();
});

// In Go, you must explicitly instrument. This is more work
// but gives you full visibility into what is being traced.
```

---

## 7. Instrumenting HTTP Clients

### Propagating Trace Context

When your service calls another service, you need to propagate the trace context so the downstream service's spans become children of your span. This is how distributed traces are formed.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// TracedHTTPClient wraps an http.Client with tracing.
type TracedHTTPClient struct {
	client     *http.Client
	tracer     trace.Tracer
	propagator propagation.TextMapPropagator
}

// NewTracedHTTPClient creates a new HTTP client with tracing.
func NewTracedHTTPClient(serviceName string) *TracedHTTPClient {
	return &TracedHTTPClient{
		client: &http.Client{
			Timeout: 30 * time.Second,
		},
		tracer:     otel.Tracer(serviceName),
		propagator: otel.GetTextMapPropagator(),
	}
}

// Do performs an HTTP request with tracing.
// It creates a client span and propagates trace context to the downstream service.
func (c *TracedHTTPClient) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
	// Create a client span
	ctx, span := c.tracer.Start(ctx, fmt.Sprintf("HTTP %s %s", req.Method, req.URL.Path),
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			semconv.HTTPMethodKey.String(req.Method),
			semconv.HTTPURLKey.String(req.URL.String()),
			attribute.String("http.host", req.URL.Host),
			attribute.String("net.peer.name", req.URL.Hostname()),
		),
	)
	defer span.End()

	// Inject trace context into the outgoing request headers.
	// This is what connects the client span to the server span
	// in the downstream service.
	c.propagator.Inject(ctx, propagation.HeaderCarrier(req.Header))

	// Execute the request
	resp, err := c.client.Do(req.WithContext(ctx))
	if err != nil {
		span.SetStatus(codes.Error, err.Error())
		span.RecordError(err)
		return nil, err
	}

	// Record response attributes
	span.SetAttributes(
		semconv.HTTPStatusCodeKey.Int(resp.StatusCode),
		attribute.Int64("http.response_content_length", resp.ContentLength),
	)

	if resp.StatusCode >= 400 {
		span.SetStatus(codes.Error, fmt.Sprintf("HTTP %d", resp.StatusCode))
	}

	return resp, nil
}

// Get is a convenience method for GET requests.
func (c *TracedHTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("creating request: %w", err)
	}
	return c.Do(ctx, req)
}

// --- Example usage ---

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("api-gateway"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	client := NewTracedHTTPClient("api-gateway")
	tracer := otel.Tracer("api-gateway")

	// Start a parent span (simulating an incoming request)
	ctx, span := tracer.Start(context.Background(), "GET /api/dashboard",
		trace.WithSpanKind(trace.SpanKindServer),
	)
	defer span.End()

	// Call downstream services -- each creates a child span
	// and propagates trace context via HTTP headers

	// In a real scenario, these would be real service URLs.
	// For demonstration, we show the pattern.
	fmt.Println("=== Demonstrating HTTP Client Instrumentation ===")
	fmt.Println()
	fmt.Println("The traced client automatically:")
	fmt.Println("1. Creates a client span (SpanKindClient)")
	fmt.Println("2. Injects traceparent header into the request")
	fmt.Println("3. Records status code and errors on the span")
	fmt.Println()

	// Show what the traceparent header looks like
	req, _ := http.NewRequestWithContext(ctx, "GET", "http://order-service:8080/api/orders", nil)
	otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

	fmt.Printf("Outgoing headers:\n")
	for key, values := range req.Header {
		for _, v := range values {
			fmt.Printf("  %s: %s\n", key, v)
		}
	}

	// The traceparent header format:
	// traceparent: 00-{trace_id}-{span_id}-{trace_flags}
	// e.g., traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

	// Use the traced client to call a real endpoint
	resp, err := client.Get(ctx, "https://httpbin.org/get")
	if err != nil {
		fmt.Printf("Request failed (expected if no network): %v\n", err)
		return
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	var result map[string]any
	json.Unmarshal(body, &result)

	fmt.Printf("\nResponse status: %d\n", resp.StatusCode)
	if headers, ok := result["headers"].(map[string]any); ok {
		if tp, ok := headers["Traceparent"]; ok {
			fmt.Printf("Server received traceparent: %s\n", tp)
		}
	}

	_ = log.Default() // suppress unused import
}
```

### Using otelhttp Transport

The official `otelhttp` package provides a transport wrapper that instruments all outgoing HTTP requests automatically:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("api-gateway"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	// Create an HTTP client with the otelhttp transport.
	// This automatically instruments ALL outgoing requests.
	client := &http.Client{
		Transport: otelhttp.NewTransport(http.DefaultTransport),
		Timeout:   30 * time.Second,
	}

	tracer := otel.Tracer("api-gateway")

	// Start a parent span
	ctx, span := tracer.Start(context.Background(), "FetchUserDashboard",
		trace.WithSpanKind(trace.SpanKindInternal),
	)
	defer span.End()

	// Every request made with this client will:
	// 1. Create a child span automatically
	// 2. Inject traceparent headers
	// 3. Record status code, duration, errors

	req, _ := http.NewRequestWithContext(ctx, "GET", "https://httpbin.org/get", nil)
	resp, err := client.Do(req)
	if err != nil {
		fmt.Printf("Request error: %v\n", err)
		return
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Printf("Status: %d, Body length: %d bytes\n", resp.StatusCode, len(body))
	fmt.Println("The otelhttp transport created a child span automatically.")
}
```

---

## 8. Instrumenting Database Calls

### Wrapping Database Queries with Spans

Database calls are one of the most common bottlenecks in any service. Instrumenting them lets you see exactly which queries are slow, which ones fail, and how they contribute to overall request latency.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// TracedDB wraps a *sql.DB with OpenTelemetry tracing.
type TracedDB struct {
	db     *sql.DB
	tracer trace.Tracer
	dbName string
	dbSystem string
}

// NewTracedDB creates a new traced database wrapper.
func NewTracedDB(db *sql.DB, dbName, dbSystem string) *TracedDB {
	return &TracedDB{
		db:       db,
		tracer:   otel.Tracer("database"),
		dbName:   dbName,
		dbSystem: dbSystem,
	}
}

// commonAttributes returns attributes common to all database operations.
func (t *TracedDB) commonAttributes() []attribute.KeyValue {
	return []attribute.KeyValue{
		semconv.DBSystemKey.String(t.dbSystem),
		semconv.DBNameKey.String(t.dbName),
	}
}

// QueryContext executes a query with tracing.
func (t *TracedDB) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
	attrs := append(t.commonAttributes(),
		semconv.DBStatementKey.String(query),
		semconv.DBOperationKey.String(extractOperation(query)),
		attribute.Int("db.parameter_count", len(args)),
	)

	ctx, span := t.tracer.Start(ctx, "db.query",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(attrs...),
	)
	defer span.End()

	start := time.Now()
	rows, err := t.db.QueryContext(ctx, query, args...)
	duration := time.Since(start)

	span.SetAttributes(attribute.Int64("db.duration_ms", duration.Milliseconds()))

	if err != nil {
		span.SetStatus(codes.Error, err.Error())
		span.RecordError(err)
		return nil, err
	}

	return rows, nil
}

// QueryRowContext executes a query that returns at most one row, with tracing.
func (t *TracedDB) QueryRowContext(ctx context.Context, query string, args ...any) *TracedRow {
	attrs := append(t.commonAttributes(),
		semconv.DBStatementKey.String(query),
		semconv.DBOperationKey.String(extractOperation(query)),
		attribute.Int("db.parameter_count", len(args)),
	)

	ctx, span := t.tracer.Start(ctx, "db.query_row",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(attrs...),
	)

	start := time.Now()
	row := t.db.QueryRowContext(ctx, query, args...)
	duration := time.Since(start)

	span.SetAttributes(attribute.Int64("db.duration_ms", duration.Milliseconds()))

	return &TracedRow{row: row, span: span}
}

// ExecContext executes a statement with tracing.
func (t *TracedDB) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
	attrs := append(t.commonAttributes(),
		semconv.DBStatementKey.String(query),
		semconv.DBOperationKey.String(extractOperation(query)),
		attribute.Int("db.parameter_count", len(args)),
	)

	ctx, span := t.tracer.Start(ctx, "db.exec",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(attrs...),
	)
	defer span.End()

	start := time.Now()
	result, err := t.db.ExecContext(ctx, query, args...)
	duration := time.Since(start)

	span.SetAttributes(attribute.Int64("db.duration_ms", duration.Milliseconds()))

	if err != nil {
		span.SetStatus(codes.Error, err.Error())
		span.RecordError(err)
		return nil, err
	}

	rowsAffected, _ := result.RowsAffected()
	span.SetAttributes(attribute.Int64("db.rows_affected", rowsAffected))

	return result, nil
}

// TracedRow wraps sql.Row to properly end the span.
type TracedRow struct {
	row  *sql.Row
	span trace.Span
}

// Scan scans the row and ends the span.
func (tr *TracedRow) Scan(dest ...any) error {
	defer tr.span.End()

	err := tr.row.Scan(dest...)
	if err != nil {
		if err == sql.ErrNoRows {
			tr.span.SetAttributes(attribute.Bool("db.no_rows", true))
		} else {
			tr.span.SetStatus(codes.Error, err.Error())
			tr.span.RecordError(err)
		}
	}
	return err
}

// extractOperation pulls the SQL operation from the query (SELECT, INSERT, etc.)
func extractOperation(query string) string {
	for i, c := range query {
		if c == ' ' || c == '\t' || c == '\n' {
			return query[:i]
		}
	}
	return query
}

// --- Traced Transaction ---

// TracedTx wraps a sql.Tx with tracing.
type TracedTx struct {
	tx     *sql.Tx
	tracer trace.Tracer
	span   trace.Span
	ctx    context.Context
	dbName string
	dbSystem string
}

// BeginTx starts a traced transaction.
func (t *TracedDB) BeginTx(ctx context.Context, opts *sql.TxOptions) (*TracedTx, error) {
	ctx, span := t.tracer.Start(ctx, "db.transaction",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(t.commonAttributes()...),
	)

	tx, err := t.db.BeginTx(ctx, opts)
	if err != nil {
		span.SetStatus(codes.Error, err.Error())
		span.RecordError(err)
		span.End()
		return nil, err
	}

	return &TracedTx{
		tx:       tx,
		tracer:   t.tracer,
		span:     span,
		ctx:      ctx,
		dbName:   t.dbName,
		dbSystem: t.dbSystem,
	}, nil
}

// ExecContext executes a statement within the transaction.
func (ttx *TracedTx) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
	_, span := ttx.tracer.Start(ctx, "db.tx.exec",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			semconv.DBSystemKey.String(ttx.dbSystem),
			semconv.DBNameKey.String(ttx.dbName),
			semconv.DBStatementKey.String(query),
			semconv.DBOperationKey.String(extractOperation(query)),
		),
	)
	defer span.End()

	result, err := ttx.tx.ExecContext(ctx, query, args...)
	if err != nil {
		span.SetStatus(codes.Error, err.Error())
		span.RecordError(err)
	}
	return result, err
}

// Commit commits the transaction and ends the transaction span.
func (ttx *TracedTx) Commit() error {
	defer ttx.span.End()
	err := ttx.tx.Commit()
	if err != nil {
		ttx.span.SetStatus(codes.Error, err.Error())
		ttx.span.RecordError(err)
	} else {
		ttx.span.SetAttributes(attribute.String("db.tx.result", "commit"))
	}
	return err
}

// Rollback rolls back the transaction and ends the transaction span.
func (ttx *TracedTx) Rollback() error {
	defer ttx.span.End()
	err := ttx.tx.Rollback()
	ttx.span.SetAttributes(attribute.String("db.tx.result", "rollback"))
	if err != nil && err != sql.ErrTxDone {
		ttx.span.SetStatus(codes.Error, err.Error())
	}
	return err
}

// --- Example usage ---

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("order-service"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	// In a real application, you would open a real database connection:
	//   db, err := sql.Open("postgres", "postgres://...")
	//   tracedDB := NewTracedDB(db, "orders_db", "postgresql")

	fmt.Println("=== Database Instrumentation Pattern ===")
	fmt.Println()
	fmt.Println("TracedDB wraps *sql.DB and automatically:")
	fmt.Println("  1. Creates a span for every query/exec")
	fmt.Println("  2. Records the SQL statement (be careful with PII!)")
	fmt.Println("  3. Records the operation type (SELECT, INSERT, etc.)")
	fmt.Println("  4. Records duration and rows affected")
	fmt.Println("  5. Records errors with stack context")
	fmt.Println()
	fmt.Println("TracedTx wraps transactions with a parent span:")
	fmt.Println("  1. Transaction span wraps all queries in the tx")
	fmt.Println("  2. Commit/rollback recorded on the span")
	fmt.Println("  3. Individual queries are child spans")
	fmt.Println()
	fmt.Println("Usage:")
	fmt.Println("  tracedDB := NewTracedDB(db, \"orders_db\", \"postgresql\")")
	fmt.Println("  rows, err := tracedDB.QueryContext(ctx, \"SELECT * FROM orders WHERE user_id = $1\", userID)")
	fmt.Println("  tx, err := tracedDB.BeginTx(ctx, nil)")
	fmt.Println("  tx.ExecContext(ctx, \"INSERT INTO orders ...\", ...)")
	fmt.Println("  tx.Commit()")

	_ = log.Default() // suppress unused import
}
```

### Security Note: SQL Statement Recording

Recording full SQL statements in spans is incredibly useful for debugging but raises security concerns. Never record queries containing plaintext passwords, credit card numbers, or other PII.

```go
// sanitizeQuery removes potential PII from SQL statements.
// In production, consider using parameterized query templates
// instead of the actual query with values.
func sanitizeQuery(query string) string {
	// Option 1: Record only the query template (recommended)
	// "SELECT * FROM users WHERE email = $1" -- safe
	// "SELECT * FROM users WHERE email = 'user@example.com'" -- NOT safe

	// Option 2: Truncate long queries
	const maxLen = 500
	if len(query) > maxLen {
		return query[:maxLen] + "...[truncated]"
	}
	return query
}
```

---

## 9. Custom Spans and Events

### Business Logic Instrumentation

Beyond automatic boundary instrumentation, you often need to instrument business-critical operations to understand _why_ they behave the way they do.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"math/rand"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("checkout-service")

// Order represents a customer order.
type Order struct {
	ID         string
	UserID     string
	Items      []OrderItem
	Total      float64
	Currency   string
	PromoCode  string
	ShippingTo string
}

// OrderItem represents an item in an order.
type OrderItem struct {
	ProductID string
	Quantity  int
	Price     float64
	Warehouse string
}

// ProcessCheckout instruments the entire checkout flow.
func ProcessCheckout(ctx context.Context, order *Order) error {
	ctx, span := tracer.Start(ctx, "ProcessCheckout",
		trace.WithAttributes(
			attribute.String("order.id", order.ID),
			attribute.String("user.id", order.UserID),
			attribute.Float64("order.total", order.Total),
			attribute.String("order.currency", order.Currency),
			attribute.Int("order.item_count", len(order.Items)),
			attribute.String("order.shipping_to", order.ShippingTo),
		),
	)
	defer span.End()

	// Add promo code info if present
	if order.PromoCode != "" {
		span.SetAttributes(attribute.String("order.promo_code", order.PromoCode))
	}

	// Step 1: Validate inventory
	if err := validateInventory(ctx, order); err != nil {
		span.SetStatus(codes.Error, "inventory validation failed")
		span.RecordError(err)
		return fmt.Errorf("inventory validation: %w", err)
	}

	// Step 2: Apply pricing rules
	finalTotal, err := applyPricing(ctx, order)
	if err != nil {
		span.SetStatus(codes.Error, "pricing calculation failed")
		span.RecordError(err)
		return fmt.Errorf("pricing: %w", err)
	}
	span.SetAttributes(attribute.Float64("order.final_total", finalTotal))

	// Step 3: Process payment
	paymentID, err := processPayment(ctx, order.UserID, finalTotal, order.Currency)
	if err != nil {
		span.SetStatus(codes.Error, "payment processing failed")
		span.RecordError(err)
		return fmt.Errorf("payment: %w", err)
	}
	span.SetAttributes(attribute.String("payment.id", paymentID))

	// Step 4: Reserve inventory
	reservationIDs, err := reserveInventory(ctx, order)
	if err != nil {
		// Payment succeeded but reservation failed -- need to refund
		span.SetStatus(codes.Error, "inventory reservation failed after payment")
		span.RecordError(err)

		// Add an event recording the compensation action
		span.AddEvent("compensation.refund_initiated",
			trace.WithAttributes(
				attribute.String("payment.id", paymentID),
				attribute.String("reason", "inventory_reservation_failed"),
			),
		)

		return fmt.Errorf("reservation: %w", err)
	}
	span.SetAttributes(attribute.StringSlice("reservation.ids", reservationIDs))

	// Step 5: Send confirmation
	sendConfirmation(ctx, order)

	span.SetAttributes(attribute.String("order.status", "completed"))
	return nil
}

func validateInventory(ctx context.Context, order *Order) error {
	ctx, span := tracer.Start(ctx, "ValidateInventory")
	defer span.End()

	warehouses := make(map[string]int)
	for _, item := range order.Items {
		warehouses[item.Warehouse]++
	}

	span.SetAttributes(
		attribute.Int("inventory.item_count", len(order.Items)),
		attribute.Int("inventory.warehouse_count", len(warehouses)),
	)

	// Simulate validation with a random failure
	time.Sleep(15 * time.Millisecond)

	// Add events for each warehouse checked
	for wh, count := range warehouses {
		span.AddEvent("inventory.warehouse_checked",
			trace.WithAttributes(
				attribute.String("warehouse", wh),
				attribute.Int("item_count", count),
				attribute.Bool("available", true),
			),
		)
	}

	return nil
}

func applyPricing(ctx context.Context, order *Order) (float64, error) {
	ctx, span := tracer.Start(ctx, "ApplyPricing")
	defer span.End()

	time.Sleep(5 * time.Millisecond)

	discount := 0.0
	if order.PromoCode != "" {
		discount = order.Total * 0.1 // 10% discount
		span.AddEvent("pricing.promo_applied",
			trace.WithAttributes(
				attribute.String("promo_code", order.PromoCode),
				attribute.Float64("discount_amount", discount),
				attribute.Float64("discount_percent", 10.0),
			),
		)
	}

	finalTotal := order.Total - discount
	span.SetAttributes(
		attribute.Float64("pricing.original_total", order.Total),
		attribute.Float64("pricing.discount", discount),
		attribute.Float64("pricing.final_total", finalTotal),
	)

	return finalTotal, nil
}

func processPayment(ctx context.Context, userID string, amount float64, currency string) (string, error) {
	_, span := tracer.Start(ctx, "ProcessPayment",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			attribute.String("payment.user_id", userID),
			attribute.Float64("payment.amount", amount),
			attribute.String("payment.currency", currency),
			attribute.String("payment.provider", "stripe"),
		),
	)
	defer span.End()

	// Simulate payment processing
	time.Sleep(100 * time.Millisecond)

	paymentID := fmt.Sprintf("pay_%d", rand.Int63n(1_000_000))
	span.SetAttributes(
		attribute.String("payment.id", paymentID),
		attribute.String("payment.status", "succeeded"),
	)

	return paymentID, nil
}

func reserveInventory(ctx context.Context, order *Order) ([]string, error) {
	ctx, span := tracer.Start(ctx, "ReserveInventory")
	defer span.End()

	var reservationIDs []string
	for _, item := range order.Items {
		resID, err := reserveItem(ctx, item)
		if err != nil {
			span.SetStatus(codes.Error, "item reservation failed")
			span.RecordError(err, trace.WithAttributes(
				attribute.String("failed_product_id", item.ProductID),
				attribute.String("failed_warehouse", item.Warehouse),
			))
			return nil, err
		}
		reservationIDs = append(reservationIDs, resID)
	}

	span.SetAttributes(attribute.Int("reservation.count", len(reservationIDs)))
	return reservationIDs, nil
}

func reserveItem(ctx context.Context, item OrderItem) (string, error) {
	_, span := tracer.Start(ctx, "ReserveItem",
		trace.WithAttributes(
			attribute.String("item.product_id", item.ProductID),
			attribute.Int("item.quantity", item.Quantity),
			attribute.String("item.warehouse", item.Warehouse),
			attribute.Float64("item.price", item.Price),
		),
	)
	defer span.End()

	time.Sleep(20 * time.Millisecond)
	resID := fmt.Sprintf("res_%d", rand.Int63n(1_000_000))
	span.SetAttributes(attribute.String("reservation.id", resID))
	return resID, nil
}

func sendConfirmation(ctx context.Context, order *Order) {
	_, span := tracer.Start(ctx, "SendConfirmation",
		trace.WithAttributes(
			attribute.String("confirmation.order_id", order.ID),
			attribute.String("confirmation.user_id", order.UserID),
			attribute.String("confirmation.channel", "email"),
		),
	)
	defer span.End()

	time.Sleep(10 * time.Millisecond)
	span.AddEvent("confirmation.sent")
}

// --- Main ---

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("checkout-service"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	order := &Order{
		ID:         "ord_12345",
		UserID:     "usr_abc",
		Total:      299.98,
		Currency:   "USD",
		PromoCode:  "SAVE10",
		ShippingTo: "DE",
		Items: []OrderItem{
			{ProductID: "prod_001", Quantity: 1, Price: 149.99, Warehouse: "eu-west-1"},
			{ProductID: "prod_002", Quantity: 1, Price: 149.99, Warehouse: "us-east-1"},
		},
	}

	err := ProcessCheckout(context.Background(), order)
	if err != nil {
		fmt.Printf("Checkout failed: %v\n", err)
	} else {
		fmt.Println("Checkout completed successfully")
	}

	_ = errors.New // suppress unused import
}
```

### Span Events vs Child Spans

A common question: when should you create a child span vs add an event to the current span?

```go
// Use a CHILD SPAN when:
// - The operation has meaningful duration
// - It crosses a boundary (HTTP call, DB query, cache lookup)
// - It might fail independently
// - You want to see it in the trace waterfall

ctx, span := tracer.Start(ctx, "db.query")
// ... do work ...
span.End()

// Use an EVENT when:
// - It is a point-in-time occurrence (no duration)
// - It is a notable happening within a span
// - You want to annotate a span with extra context

span.AddEvent("cache.miss",
    trace.WithAttributes(
        attribute.String("cache.key", "user:abc:profile"),
    ),
)

span.AddEvent("retry.attempt",
    trace.WithAttributes(
        attribute.Int("retry.number", 2),
        attribute.String("retry.reason", "timeout"),
    ),
)
```

---

## 10. Log Correlation

### Connecting Logs to Traces

The power of observability comes from _correlation_ -- the ability to jump from a log line to its trace, or from a trace to its logs. The bridge is the `trace_id` and `span_id`.

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// LoggerWithTrace extracts trace context from the context and adds
// trace_id and span_id to the logger. This is the fundamental
// correlation mechanism between logs and traces.
func LoggerWithTrace(ctx context.Context, logger *slog.Logger) *slog.Logger {
	span := trace.SpanFromContext(ctx)
	if !span.SpanContext().IsValid() {
		return logger
	}

	return logger.With(
		slog.String("trace_id", span.SpanContext().TraceID().String()),
		slog.String("span_id", span.SpanContext().SpanID().String()),
		slog.Bool("trace_sampled", span.SpanContext().TraceFlags().IsSampled()),
	)
}

// TraceContextHandler is a custom slog.Handler that automatically
// injects trace context into every log record.
type TraceContextHandler struct {
	inner slog.Handler
}

// NewTraceContextHandler wraps an existing handler with trace context injection.
func NewTraceContextHandler(inner slog.Handler) *TraceContextHandler {
	return &TraceContextHandler{inner: inner}
}

func (h *TraceContextHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *TraceContextHandler) Handle(ctx context.Context, record slog.Record) error {
	// Automatically inject trace context from the context
	span := trace.SpanFromContext(ctx)
	if span.SpanContext().IsValid() {
		record.AddAttrs(
			slog.String("trace_id", span.SpanContext().TraceID().String()),
			slog.String("span_id", span.SpanContext().SpanID().String()),
		)
	}
	return h.inner.Handle(ctx, record)
}

func (h *TraceContextHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &TraceContextHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *TraceContextHandler) WithGroup(name string) slog.Handler {
	return &TraceContextHandler{inner: h.inner.WithGroup(name)}
}

// --- Example usage ---

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("order-service"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	// --- Approach 1: Manual logger enrichment ---
	fmt.Println("=== Approach 1: Manual LoggerWithTrace ===")

	baseLogger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelDebug,
	}))

	tracer := otel.Tracer("order-service")
	ctx, span := tracer.Start(context.Background(), "ProcessOrder")

	// Create a trace-enriched logger
	logger := LoggerWithTrace(ctx, baseLogger)

	// Every log line now includes trace_id and span_id
	logger.Info("starting order processing",
		slog.String("order_id", "ord_12345"),
	)
	logger.Debug("validating inventory",
		slog.Int("item_count", 3),
	)
	logger.Info("order completed",
		slog.String("order_id", "ord_12345"),
		slog.Duration("duration", 250*time.Millisecond),
	)

	span.End()

	fmt.Println()

	// --- Approach 2: Automatic injection via custom handler ---
	fmt.Println("=== Approach 2: TraceContextHandler (automatic) ===")

	autoLogger := slog.New(
		NewTraceContextHandler(
			slog.NewJSONHandler(os.Stdout, nil),
		),
	)

	ctx2, span2 := tracer.Start(context.Background(), "ProcessPayment")

	// trace_id and span_id are injected automatically
	// just pass the context to the log call
	autoLogger.InfoContext(ctx2, "charging card",
		slog.String("payment_id", "pay_789"),
		slog.Float64("amount", 149.99),
	)

	autoLogger.InfoContext(ctx2, "payment succeeded",
		slog.String("payment_id", "pay_789"),
	)

	// Without context, no trace IDs are added (graceful degradation)
	autoLogger.Info("server startup complete",
		slog.String("version", "v2.34.1"),
	)

	span2.End()
}
```

### Correlation in Practice

With trace_id in your logs, you can:

```
1. See an error in your logs:
   {"level":"ERROR","msg":"payment failed","trace_id":"4bf92f...","error":"card declined"}

2. Copy the trace_id and search your tracing backend:
   Jaeger/Tempo/Honeycomb: search by trace_id = "4bf92f..."

3. See the full distributed trace:
   api-gateway → order-service → payment-service (FAILED)
                               → inventory-service (OK)

4. Click on the payment-service span to see its logs:
   All log lines with trace_id="4bf92f..." in this service

5. Understand the full context:
   Who was the user? What was in their cart? Which payment method?
   What was the error message from Stripe?
```

### Node.js Comparison

```javascript
// Node.js: pino + OpenTelemetry log correlation
const { trace, context } = require('@opentelemetry/api');
const pino = require('pino');

const logger = pino();

function logWithTrace(ctx, msg, fields = {}) {
  const span = trace.getSpan(context.active());
  if (span) {
    const spanContext = span.spanContext();
    fields.trace_id = spanContext.traceId;
    fields.span_id = spanContext.spanId;
  }
  logger.info(fields, msg);
}

// Go's approach is more explicit: you pass ctx to LoggerWithTrace
// or use a custom slog.Handler with InfoContext(ctx, ...).
// Node.js uses context.active() which relies on AsyncLocalStorage.
```

---

## 11. Context Propagation

### W3C Trace Context

The W3C Trace Context specification defines a standard HTTP header format for propagating trace context across service boundaries. OpenTelemetry uses this by default.

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^-^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^-^^^^^^^^^^^^^^^^-^^
             |  |                                |                |
             |  trace-id (32 hex chars)          span-id (16 hex) trace-flags
             version                                              (01 = sampled)
```

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/trace"
)

func main() {
	// Set up propagators. This is critical -- without this,
	// trace context will not cross service boundaries.
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{}, // W3C Trace Context (traceparent, tracestate)
			propagation.Baggage{},      // W3C Baggage (key=value pairs)
		),
	)

	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(exporter))
	otel.SetTracerProvider(tp)

	tracer := otel.Tracer("demo")
	propagator := otel.GetTextMapPropagator()

	// --- Injection: adding trace context to outgoing requests ---
	ctx, span := tracer.Start(context.Background(), "parent-operation")
	defer span.End()

	// Create an HTTP request and inject trace context
	req, _ := http.NewRequestWithContext(ctx, "GET", "http://downstream:8080/api/data", nil)
	propagator.Inject(ctx, propagation.HeaderCarrier(req.Header))

	fmt.Println("=== Injected Headers (Outgoing Request) ===")
	for key, values := range req.Header {
		for _, v := range values {
			fmt.Printf("  %s: %s\n", key, v)
		}
	}

	// --- Extraction: reading trace context from incoming requests ---
	fmt.Println("\n=== Extraction (Incoming Request) ===")

	// Simulate receiving a request with trace context
	incomingHeaders := http.Header{}
	incomingHeaders.Set("traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01")
	incomingHeaders.Set("tracestate", "vendor1=value1")

	// Extract trace context from headers
	extractedCtx := propagator.Extract(context.Background(), propagation.HeaderCarrier(incomingHeaders))

	// Start a new span as a child of the extracted context
	_, childSpan := tracer.Start(extractedCtx, "child-operation")
	defer childSpan.End()

	fmt.Printf("  Parent trace_id: %s\n", childSpan.SpanContext().TraceID().String())
	fmt.Printf("  Child span_id:   %s\n", childSpan.SpanContext().SpanID().String())
	fmt.Printf("  Is remote:       %v\n", trace.SpanFromContext(extractedCtx).SpanContext().IsRemote())
}
```

### Baggage: Propagating Business Context

Baggage lets you propagate key-value pairs across service boundaries. Unlike span attributes (which are local to a span), baggage is propagated to _all_ downstream services.

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/baggage"
	"go.opentelemetry.io/otel/propagation"
)

func main() {
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{},
			propagation.Baggage{},
		),
	)

	// --- Creating baggage ---
	userID, _ := baggage.NewMember("user.id", "usr_a8f3k29x")
	tenantID, _ := baggage.NewMember("tenant.id", "tenant_acme")
	region, _ := baggage.NewMember("request.region", "eu-west-1")

	bag, _ := baggage.New(userID, tenantID, region)

	// Attach baggage to the context
	ctx := baggage.ContextWithBaggage(context.Background(), bag)

	// --- Injecting baggage into HTTP headers ---
	req, _ := http.NewRequestWithContext(ctx, "GET", "http://downstream:8080/api/data", nil)
	otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

	fmt.Println("=== Outgoing Headers with Baggage ===")
	for key, values := range req.Header {
		for _, v := range values {
			fmt.Printf("  %s: %s\n", key, v)
		}
	}

	// --- Extracting baggage from incoming requests ---
	fmt.Println("\n=== Extracting Baggage ===")

	// Simulate receiving a request with baggage
	incomingHeaders := http.Header{}
	incomingHeaders.Set("baggage", "user.id=usr_a8f3k29x,tenant.id=tenant_acme,request.region=eu-west-1")

	extractedCtx := otel.GetTextMapPropagator().Extract(
		context.Background(),
		propagation.HeaderCarrier(incomingHeaders),
	)

	extractedBag := baggage.FromContext(extractedCtx)
	for _, member := range extractedBag.Members() {
		fmt.Printf("  %s = %s\n", member.Key(), member.Value())
	}

	// --- Use cases for baggage ---
	fmt.Println("\n=== Baggage Use Cases ===")
	fmt.Println("  1. user.id -- track requests per user across all services")
	fmt.Println("  2. tenant.id -- multi-tenant isolation and billing")
	fmt.Println("  3. feature.flags -- propagate feature flag state")
	fmt.Println("  4. request.priority -- let downstream services prioritize")
	fmt.Println()
	fmt.Println("WARNING: Baggage is sent to ALL downstream services via HTTP headers.")
	fmt.Println("Never put sensitive data (passwords, tokens) in baggage.")
	fmt.Println("Keep baggage small -- it adds overhead to every request.")
}
```

### Propagating Across Goroutines

In Go, context propagation across goroutines is straightforward -- you pass the context. But you need to be careful about span lifecycle.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

func initTracer() *sdktrace.TracerProvider {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("parallel-service"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	return tp
}

func main() {
	tp := initTracer()
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	tracer := otel.Tracer("parallel-service")

	// Parent span for the overall operation
	ctx, parentSpan := tracer.Start(context.Background(), "FetchDashboard")
	defer parentSpan.End()

	// Fan out to multiple goroutines, each with its own child span.
	// Pass the context so child spans are linked to the parent.
	var wg sync.WaitGroup
	results := make(chan string, 3)

	tasks := []struct {
		name     string
		duration time.Duration
	}{
		{"FetchUserProfile", 50 * time.Millisecond},
		{"FetchRecentOrders", 100 * time.Millisecond},
		{"FetchRecommendations", 75 * time.Millisecond},
	}

	for _, task := range tasks {
		wg.Add(1)
		go func(name string, dur time.Duration) {
			defer wg.Done()

			// Create a child span in the goroutine.
			// The parent context carries the parent span,
			// so this span becomes a child automatically.
			_, span := tracer.Start(ctx, name,
				trace.WithAttributes(
					attribute.String("task.name", name),
				),
			)
			defer span.End()

			// Simulate work
			time.Sleep(dur)
			span.SetAttributes(attribute.String("task.status", "completed"))
			results <- fmt.Sprintf("%s completed in %v", name, dur)
		}(task.name, task.duration)
	}

	// Wait for all goroutines to finish
	go func() {
		wg.Wait()
		close(results)
	}()

	for result := range results {
		fmt.Println(result)
	}

	parentSpan.SetAttributes(attribute.Int("tasks.completed", len(tasks)))

	fmt.Println("\nAll tasks completed. The trace shows parallel execution:")
	fmt.Println("  FetchDashboard (parent)")
	fmt.Println("  ├── FetchUserProfile (child, concurrent)")
	fmt.Println("  ├── FetchRecentOrders (child, concurrent)")
	fmt.Println("  └── FetchRecommendations (child, concurrent)")

	// IMPORTANT: Do NOT create a span in one goroutine and End() it in another.
	// Each goroutine should create and end its own spans.
	// It is safe to pass the parent context to goroutines -- the parent span
	// can outlive the child spans.
}
```

---

## 12. Sampling Strategies

### Why Sample?

At high traffic volumes, tracing every request is prohibitively expensive. If your service handles 10,000 requests per second and each request generates 20 spans, that is 200,000 spans per second. At ~1KB per span, that is 200MB/s of telemetry data. You need to sample.

The goal of sampling is to keep enough data to answer your questions while discarding enough to stay within budget.

### Head-Based vs Tail-Based Sampling

**Head-based sampling** decides at the start of a trace whether to sample it. It is simple and efficient but cannot make decisions based on outcomes (you do not know yet if the request will be slow or error).

**Tail-based sampling** waits until the trace is complete, then decides whether to keep it. It can keep all error traces and all slow traces while discarding routine ones. It requires a collector that buffers complete traces.

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

func main() {
	// --- Head-based sampling strategies ---

	fmt.Println("=== Sampling Strategies ===")
	fmt.Println()

	// 1. AlwaysSample: Sample 100% (development only)
	fmt.Println("1. AlwaysSample: every trace is sampled")
	fmt.Println("   Use for: development, testing")
	fmt.Println("   sdktrace.WithSampler(sdktrace.AlwaysSample())")
	fmt.Println()

	// 2. NeverSample: Sample 0%
	fmt.Println("2. NeverSample: no traces are sampled")
	fmt.Println("   Use for: disabling tracing entirely")
	fmt.Println("   sdktrace.WithSampler(sdktrace.NeverSample())")
	fmt.Println()

	// 3. TraceIDRatioBased: Sample a fraction of traces
	fmt.Println("3. TraceIDRatioBased: sample a fixed percentage")
	fmt.Println("   Use for: production baseline sampling")
	fmt.Println("   sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1)) // 10%")
	fmt.Println()

	// 4. ParentBased: Respect the parent's sampling decision
	fmt.Println("4. ParentBased: follow the parent span's decision")
	fmt.Println("   If parent was sampled, child is sampled.")
	fmt.Println("   If no parent, fall back to a root sampler.")
	fmt.Println("   This is the RECOMMENDED production strategy.")
	fmt.Println("   sdktrace.WithSampler(sdktrace.ParentBased(")
	fmt.Println("       sdktrace.TraceIDRatioBased(0.1),")
	fmt.Println("   ))")
	fmt.Println()

	// Demonstrate ParentBased + TraceIDRatioBased
	demonstrateSampling()
}

func demonstrateSampling() {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("sampling-demo"),
		)),
		// ParentBased: if an incoming request has a sampled parent,
		// always sample. If no parent, use 10% ratio sampling.
		sdktrace.WithSampler(sdktrace.ParentBased(
			sdktrace.TraceIDRatioBased(0.1), // 10% of new traces
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.TraceContext{})
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	tracer := otel.Tracer("sampling-demo")

	// Simulate 100 requests and count how many are sampled
	sampled := 0
	notSampled := 0

	for i := 0; i < 100; i++ {
		_, span := tracer.Start(context.Background(), "request")
		if span.SpanContext().IsSampled() {
			sampled++
		} else {
			notSampled++
		}
		span.End()
	}

	fmt.Printf("\n=== Sampling Results (100 requests, 10%% ratio) ===\n")
	fmt.Printf("  Sampled:     %d\n", sampled)
	fmt.Printf("  Not sampled: %d\n", notSampled)
	fmt.Printf("  Actual rate: %.0f%%\n", float64(sampled)/100*100)
}
```

### Custom Sampler

For more sophisticated sampling, you can implement a custom sampler. A common pattern is to always sample errors and slow requests.

```go
package main

import (
	"fmt"
	"strings"

	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/trace"
)

// PrioritySampler implements a custom sampling strategy:
// - Always sample health check endpoints at 0% (never)
// - Always sample error responses at 100%
// - Sample everything else at a configurable rate
type PrioritySampler struct {
	defaultRate float64
	description string
}

// NewPrioritySampler creates a new priority sampler.
func NewPrioritySampler(defaultRate float64) *PrioritySampler {
	return &PrioritySampler{
		defaultRate: defaultRate,
		description: fmt.Sprintf("PrioritySampler{rate=%.2f}", defaultRate),
	}
}

// ShouldSample makes a sampling decision for each span.
func (s *PrioritySampler) ShouldSample(params sdktrace.SamplingParameters) sdktrace.SamplingResult {
	// Never sample health checks
	if isHealthCheck(params.Name) {
		return sdktrace.SamplingResult{
			Decision:   sdktrace.Drop,
			Tracestate: trace.SpanContextFromContext(params.ParentContext).TraceState(),
		}
	}

	// If parent was sampled, always sample (maintain trace continuity)
	parentSpanContext := trace.SpanContextFromContext(params.ParentContext)
	if parentSpanContext.IsValid() && parentSpanContext.IsSampled() {
		return sdktrace.SamplingResult{
			Decision:   sdktrace.RecordAndSample,
			Tracestate: parentSpanContext.TraceState(),
		}
	}

	// For root spans, use ratio-based sampling
	// The ratio-based sampler makes deterministic decisions based on trace_id,
	// so all spans in the same trace get the same decision.
	ratioSampler := sdktrace.TraceIDRatioBased(s.defaultRate)
	return ratioSampler.ShouldSample(params)
}

// Description returns a description of the sampler.
func (s *PrioritySampler) Description() string {
	return s.description
}

func isHealthCheck(name string) bool {
	lower := strings.ToLower(name)
	return strings.Contains(lower, "/health") ||
		strings.Contains(lower, "/ready") ||
		strings.Contains(lower, "/live")
}

func main() {
	sampler := NewPrioritySampler(0.1)
	fmt.Printf("Sampler: %s\n\n", sampler.Description())

	fmt.Println("Usage:")
	fmt.Println("  tp := sdktrace.NewTracerProvider(")
	fmt.Println("      sdktrace.WithSampler(NewPrioritySampler(0.1)),")
	fmt.Println("  )")
	fmt.Println()
	fmt.Println("Behavior:")
	fmt.Println("  GET /health       -> Never sampled (0%)")
	fmt.Println("  GET /ready        -> Never sampled (0%)")
	fmt.Println("  GET /api/orders   -> 10% sampled (root spans)")
	fmt.Println("  Child of sampled  -> Always sampled (100%)")
	fmt.Println()
	fmt.Println("For tail-based sampling (keep errors, keep slow requests),")
	fmt.Println("configure the OpenTelemetry Collector's tail_sampling processor.")
	fmt.Println("The SDK-side sampler cannot make tail-based decisions because")
	fmt.Println("it does not know the final outcome when the span starts.")
}
```

### Tail-Based Sampling with the Collector

Tail-based sampling happens in the OpenTelemetry Collector, not in your application. Your application sends all (or a large percentage of) spans, and the collector decides which complete traces to keep.

```yaml
# otel-collector-config.yaml
# This is a Collector configuration, not Go code.
# It shows how tail-based sampling works in practice.

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  tail_sampling:
    decision_wait: 10s          # Wait 10s for all spans to arrive
    num_traces: 100000          # Buffer up to 100k traces
    policies:
      # Always keep error traces
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Always keep slow traces (>1s)
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000

      # Keep 10% of everything else
      - name: probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling]
      exporters: [otlp]
```

---

## 13. Instrumentation Best Practices

### Span Naming Conventions

Span names should be low-cardinality and descriptive. They are used for grouping and searching.

```go
package main

import "fmt"

func main() {
	fmt.Println("=== Span Naming Best Practices ===")
	fmt.Println()

	// BAD: High cardinality (unique per request)
	bad := []string{
		"GET /api/users/abc123",          // user ID in name
		"GET /api/orders/ord_789/items",  // order ID in name
		"SELECT * FROM users WHERE id=5", // query with values
		"http://api.example.com/v2/data", // full URL
	}
	fmt.Println("BAD span names (high cardinality):")
	for _, name := range bad {
		fmt.Printf("  %s\n", name)
	}

	fmt.Println()

	// GOOD: Low cardinality (same for all similar operations)
	good := []string{
		"GET /api/users/{id}",           // parameterized route
		"GET /api/orders/{id}/items",    // parameterized route
		"db.users.select",              // operation + table
		"http.client GET api.example.com", // method + host only
	}
	fmt.Println("GOOD span names (low cardinality):")
	for _, name := range good {
		fmt.Printf("  %s\n", name)
	}

	fmt.Println()

	// Naming conventions by span kind
	conventions := map[string][]string{
		"HTTP Server": {
			"GET /api/users/{id}",
			"POST /api/orders",
			"PUT /api/users/{id}/preferences",
		},
		"HTTP Client": {
			"HTTP GET order-service",
			"HTTP POST payment-service",
		},
		"Database": {
			"db.users.select",
			"db.orders.insert",
			"db.inventory.update",
			"db.transaction",
		},
		"Cache": {
			"cache.get",
			"cache.set",
			"cache.delete",
		},
		"Message Queue": {
			"queue.publish orders.created",
			"queue.consume orders.created",
		},
		"Internal": {
			"ValidateOrder",
			"CalculateShipping",
			"ProcessPayment",
		},
	}

	fmt.Println("Naming conventions by span kind:")
	for kind, names := range conventions {
		fmt.Printf("\n  %s:\n", kind)
		for _, name := range names {
			fmt.Printf("    %s\n", name)
		}
	}
}
```

### What to Add to Spans

```go
package main

import (
	"context"
	"fmt"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/trace"
)

func main() {
	exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
	tp := sdktrace.NewTracerProvider(sdktrace.WithBatcher(exporter))
	otel.SetTracerProvider(tp)
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}()

	tracer := otel.Tracer("best-practices")
	ctx := context.Background()

	// --- Example: Well-instrumented span ---
	_, span := tracer.Start(ctx, "POST /api/orders",
		trace.WithSpanKind(trace.SpanKindServer),
	)

	// 1. Add attributes at creation time (known upfront)
	span.SetAttributes(
		attribute.String("http.method", "POST"),
		attribute.String("http.route", "/api/orders"),
	)

	// 2. Add attributes as you learn them
	span.SetAttributes(
		attribute.String("user.id", "usr_abc"),
		attribute.String("user.plan", "premium"),
	)

	// 3. Add business context
	span.SetAttributes(
		attribute.String("order.id", "ord_123"),
		attribute.Float64("order.total", 149.99),
		attribute.Int("order.item_count", 3),
		attribute.Bool("order.is_first_order", false),
		attribute.String("order.promo_code", "SAVE10"),
	)

	// 4. Record events for notable occurrences
	span.AddEvent("inventory.checked",
		trace.WithAttributes(
			attribute.Bool("all_available", true),
			attribute.Int("items_checked", 3),
		),
	)

	// 5. Record errors properly
	err := fmt.Errorf("payment gateway timeout after 5s")
	span.RecordError(err,
		trace.WithAttributes(
			attribute.String("payment.gateway", "stripe"),
			attribute.Int("payment.retry_count", 2),
		),
	)
	span.SetStatus(codes.Error, err.Error())

	// 6. Add result attributes at the end
	span.SetAttributes(
		attribute.Int("http.status_code", 500),
		attribute.String("http.response_content_type", "application/json"),
	)

	span.End()

	fmt.Println("=== Attribute Guidelines ===")
	fmt.Println()
	fmt.Println("ALWAYS add:")
	fmt.Println("  - Request identity: method, route, status code")
	fmt.Println("  - User identity: user_id (high cardinality is OK!)")
	fmt.Println("  - Business context: order_id, product_id, etc.")
	fmt.Println("  - Error details: error message, error type")
	fmt.Println("  - Performance: relevant counts and durations")
	fmt.Println()
	fmt.Println("NEVER add:")
	fmt.Println("  - Passwords, API keys, tokens")
	fmt.Println("  - Credit card numbers, SSNs")
	fmt.Println("  - Full request/response bodies (too large)")
	fmt.Println("  - PII unless required and properly handled")
	fmt.Println()
	fmt.Println("BE CAREFUL with:")
	fmt.Println("  - SQL statements (may contain PII)")
	fmt.Println("  - Email addresses (PII)")
	fmt.Println("  - IP addresses (may be PII in some jurisdictions)")
	fmt.Println("  - Request headers (may contain auth tokens)")
}
```

### Recording Errors

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"net"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("error-examples")

// handleError demonstrates proper error recording on spans.
func handleError(ctx context.Context, err error) {
	span := trace.SpanFromContext(ctx)

	// Always record the error on the span
	span.RecordError(err)

	// Categorize the error for better filtering
	switch {
	case errors.Is(err, context.Canceled):
		// Client disconnected -- not really an error
		span.SetAttributes(attribute.String("error.kind", "cancelled"))
		span.SetStatus(codes.Ok, "client cancelled")

	case errors.Is(err, context.DeadlineExceeded):
		// Timeout -- operational error
		span.SetAttributes(attribute.String("error.kind", "timeout"))
		span.SetStatus(codes.Error, "deadline exceeded")

	case errors.Is(err, sql.ErrNoRows):
		// Not found -- expected in some cases
		span.SetAttributes(attribute.String("error.kind", "not_found"))
		// Depending on context, this might not be an error
		span.SetStatus(codes.Ok, "not found")

	default:
		// Check if it is a network error
		var netErr net.Error
		if errors.As(err, &netErr) {
			span.SetAttributes(
				attribute.String("error.kind", "network"),
				attribute.Bool("error.temporary", netErr.Timeout()),
			)
		}
		span.SetStatus(codes.Error, err.Error())
	}
}

func main() {
	fmt.Println("=== Error Recording Best Practices ===")
	fmt.Println()
	fmt.Println("1. Always call span.RecordError(err) to attach the error to the span")
	fmt.Println("2. Always call span.SetStatus(codes.Error, msg) to mark the span as errored")
	fmt.Println("3. Categorize errors with attributes (error.kind) for filtering")
	fmt.Println("4. Not all errors are span errors:")
	fmt.Println("   - sql.ErrNoRows might be expected (404)")
	fmt.Println("   - context.Canceled means client disconnected")
	fmt.Println("5. Add context to errors: retry count, which service failed, etc.")
	fmt.Println("6. Use errors.Is() and errors.As() for proper error matching")
}
```

---

## 14. Building an Observability Library

### Wrapping OpenTelemetry for Your Organization

In a real organization with multiple services, you do not want every team to set up OpenTelemetry from scratch. You create a shared library that provides sensible defaults.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"log/slog"
	"net/http"
	"os"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/metric"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// ============================================================
// observe package -- your organization's observability library
// ============================================================

// Config holds configuration for the observability library.
type Config struct {
	ServiceName    string
	ServiceVersion string
	Environment    string
	SampleRate     float64
	OTLPEndpoint   string // empty = use stdout exporter
}

// DefaultConfig returns a Config populated from environment variables.
func DefaultConfig(serviceName string) Config {
	return Config{
		ServiceName:    serviceName,
		ServiceVersion: getEnv("SERVICE_VERSION", "dev"),
		Environment:    getEnv("ENVIRONMENT", "development"),
		SampleRate:     1.0, // 100% in dev
		OTLPEndpoint:   os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT"),
	}
}

// Provider holds all observability providers and offers a clean API.
type Provider struct {
	TracerProvider *sdktrace.TracerProvider
	Tracer         trace.Tracer
	Logger         *slog.Logger
	config         Config
}

// Init initializes the observability provider.
// Call Shutdown() when your application exits.
func Init(cfg Config) (*Provider, error) {
	// Create resource
	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String(cfg.ServiceName),
			semconv.ServiceVersionKey.String(cfg.ServiceVersion),
			semconv.DeploymentEnvironmentKey.String(cfg.Environment),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	// Create exporter
	// In production, this would use OTLP; for demo, we use stdout
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, fmt.Errorf("creating exporter: %w", err)
	}

	// Create sampler
	sampler := sdktrace.ParentBased(
		sdktrace.TraceIDRatioBased(cfg.SampleRate),
	)

	// Create TracerProvider
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sampler),
	)

	// Set global providers
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{},
			propagation.Baggage{},
		),
	)

	// Create trace-aware logger
	logger := slog.New(
		NewTraceContextHandler(
			slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
				Level: slog.LevelInfo,
			}),
		),
	)

	return &Provider{
		TracerProvider: tp,
		Tracer:         otel.Tracer(cfg.ServiceName),
		Logger:         logger,
		config:         cfg,
	}, nil
}

// Shutdown gracefully shuts down all providers.
func (p *Provider) Shutdown(ctx context.Context) error {
	return p.TracerProvider.Shutdown(ctx)
}

// TraceContextHandler automatically injects trace context into slog records.
type TraceContextHandler struct {
	inner slog.Handler
}

func NewTraceContextHandler(inner slog.Handler) *TraceContextHandler {
	return &TraceContextHandler{inner: inner}
}

func (h *TraceContextHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *TraceContextHandler) Handle(ctx context.Context, record slog.Record) error {
	span := trace.SpanFromContext(ctx)
	if span.SpanContext().IsValid() {
		record.AddAttrs(
			slog.String("trace_id", span.SpanContext().TraceID().String()),
			slog.String("span_id", span.SpanContext().SpanID().String()),
		)
	}
	return h.inner.Handle(ctx, record)
}

func (h *TraceContextHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &TraceContextHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *TraceContextHandler) WithGroup(name string) slog.Handler {
	return &TraceContextHandler{inner: h.inner.WithGroup(name)}
}

// --- HTTP Middleware ---

// responseWriter wraps http.ResponseWriter to capture status code.
type responseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// HTTPMiddleware returns middleware that creates spans and logs for HTTP requests.
func (p *Provider) HTTPMiddleware() func(http.Handler) http.Handler {
	propagator := otel.GetTextMapPropagator()

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// Extract incoming trace context
			ctx := propagator.Extract(r.Context(), propagation.HeaderCarrier(r.Header))

			// Create server span
			ctx, span := p.Tracer.Start(ctx, fmt.Sprintf("%s %s", r.Method, r.URL.Path),
				trace.WithSpanKind(trace.SpanKindServer),
				trace.WithAttributes(
					semconv.HTTPMethodKey.String(r.Method),
					semconv.HTTPTargetKey.String(r.URL.Path),
					attribute.String("http.host", r.Host),
					attribute.String("http.user_agent", r.UserAgent()),
				),
			)
			defer span.End()

			// Wrap response writer
			wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

			// Serve
			next.ServeHTTP(wrapped, r.WithContext(ctx))

			// Record response attributes
			duration := time.Since(start)
			span.SetAttributes(
				semconv.HTTPStatusCodeKey.Int(wrapped.statusCode),
				attribute.Int("http.response_content_length", wrapped.bytesWritten),
			)

			if wrapped.statusCode >= 500 {
				span.SetStatus(codes.Error, fmt.Sprintf("HTTP %d", wrapped.statusCode))
			}

			// Correlated log line
			level := slog.LevelInfo
			if wrapped.statusCode >= 500 {
				level = slog.LevelError
			} else if wrapped.statusCode >= 400 {
				level = slog.LevelWarn
			}

			p.Logger.LogAttrs(ctx, level, "request completed",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", wrapped.statusCode),
				slog.Duration("duration", duration),
				slog.Int("bytes", wrapped.bytesWritten),
			)
		})
	}
}

// HTTPClient returns an http.Client with automatic tracing.
func (p *Provider) HTTPClient() *http.Client {
	return &http.Client{
		Timeout: 30 * time.Second,
		Transport: &tracedTransport{
			base:       http.DefaultTransport,
			tracer:     p.Tracer,
			propagator: otel.GetTextMapPropagator(),
		},
	}
}

type tracedTransport struct {
	base       http.RoundTripper
	tracer     trace.Tracer
	propagator propagation.TextMapPropagator
}

func (t *tracedTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	ctx, span := t.tracer.Start(req.Context(),
		fmt.Sprintf("HTTP %s %s", req.Method, req.URL.Host),
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			semconv.HTTPMethodKey.String(req.Method),
			semconv.HTTPURLKey.String(req.URL.String()),
		),
	)
	defer span.End()

	t.propagator.Inject(ctx, propagation.HeaderCarrier(req.Header))
	req = req.WithContext(ctx)

	resp, err := t.base.RoundTrip(req)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, err
	}

	span.SetAttributes(semconv.HTTPStatusCodeKey.Int(resp.StatusCode))
	return resp, nil
}

// --- Helpers ---

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

// ============================================================
// Application code -- how teams USE the library
// ============================================================

func main() {
	// Initialize observability with one function call
	obs, err := Init(DefaultConfig("order-service"))
	if err != nil {
		log.Fatalf("Failed to initialize observability: %v", err)
	}
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		obs.Shutdown(ctx)
	}()

	// Use the middleware
	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/orders", func(w http.ResponseWriter, r *http.Request) {
		// Get the trace-correlated logger
		obs.Logger.InfoContext(r.Context(), "fetching orders",
			slog.String("user_id", "usr_abc"),
		)

		// Create a child span for business logic
		ctx, span := obs.Tracer.Start(r.Context(), "FetchOrders")
		defer span.End()

		span.SetAttributes(attribute.String("user.id", "usr_abc"))

		// Simulate DB query
		time.Sleep(20 * time.Millisecond)

		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`[{"id":"ord_1","total":99.99}]`))
		_ = ctx
	})

	handler := obs.HTTPMiddleware()(mux)

	fmt.Println("=== Observability Library Usage ===")
	fmt.Println()
	fmt.Println("Teams get traces, metrics, and correlated logs with minimal setup:")
	fmt.Println()
	fmt.Println("  obs, err := observe.Init(observe.DefaultConfig(\"my-service\"))")
	fmt.Println("  defer obs.Shutdown(ctx)")
	fmt.Println()
	fmt.Println("  handler := obs.HTTPMiddleware()(mux)  // automatic server spans")
	fmt.Println("  client := obs.HTTPClient()            // automatic client spans")
	fmt.Println("  obs.Logger.InfoContext(ctx, \"msg\")     // correlated logs")
	fmt.Println("  obs.Tracer.Start(ctx, \"operation\")     // custom spans")
	fmt.Println()
	fmt.Println("Server starting on :8080...")

	server := &http.Server{
		Addr:    ":8080",
		Handler: handler,
	}
	log.Fatal(server.ListenAndServe())

	_ = metric.NewFloat64Histogram // suppress unused import
}
```

---

## 15. Real-World Example: Fully Instrumented HTTP API

### Complete Bookstore API

This is a complete, runnable API that demonstrates every observability pattern from this chapter: traces, metrics, correlated logs, context propagation, and proper error handling.

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"log/slog"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"sync"
	"syscall"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/metric"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
	"go.opentelemetry.io/otel/trace"
)

// ============================================================
// Domain Models
// ============================================================

type Book struct {
	ID       string  `json:"id"`
	Title    string  `json:"title"`
	Author   string  `json:"author"`
	Price    float64 `json:"price"`
	InStock  int     `json:"in_stock"`
	Category string  `json:"category"`
}

type CreateBookRequest struct {
	Title    string  `json:"title"`
	Author   string  `json:"author"`
	Price    float64 `json:"price"`
	InStock  int     `json:"in_stock"`
	Category string  `json:"category"`
}

type ErrorResponse struct {
	Error   string `json:"error"`
	Code    string `json:"code"`
	TraceID string `json:"trace_id,omitempty"`
}

// ============================================================
// In-Memory Store (simulates a database)
// ============================================================

type BookStore struct {
	mu    sync.RWMutex
	books map[string]Book
}

func NewBookStore() *BookStore {
	return &BookStore{
		books: map[string]Book{
			"book_001": {ID: "book_001", Title: "Observability Engineering", Author: "Charity Majors", Price: 49.99, InStock: 15, Category: "engineering"},
			"book_002": {ID: "book_002", Title: "The Go Programming Language", Author: "Donovan & Kernighan", Price: 39.99, InStock: 23, Category: "programming"},
			"book_003": {ID: "book_003", Title: "Designing Data-Intensive Applications", Author: "Martin Kleppmann", Price: 44.99, InStock: 8, Category: "engineering"},
		},
	}
}

func (s *BookStore) GetAll(ctx context.Context) ([]Book, error) {
	tracer := otel.Tracer("bookstore")
	_, span := tracer.Start(ctx, "db.books.select_all",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			attribute.String("db.system", "memory"),
			attribute.String("db.operation", "SELECT"),
			attribute.String("db.table", "books"),
		),
	)
	defer span.End()

	// Simulate database latency
	time.Sleep(time.Duration(10+rand.Intn(20)) * time.Millisecond)

	s.mu.RLock()
	defer s.mu.RUnlock()

	books := make([]Book, 0, len(s.books))
	for _, b := range s.books {
		books = append(books, b)
	}

	span.SetAttributes(attribute.Int("db.rows_returned", len(books)))
	return books, nil
}

func (s *BookStore) GetByID(ctx context.Context, id string) (Book, error) {
	tracer := otel.Tracer("bookstore")
	_, span := tracer.Start(ctx, "db.books.select_by_id",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			attribute.String("db.system", "memory"),
			attribute.String("db.operation", "SELECT"),
			attribute.String("db.table", "books"),
			attribute.String("db.query_parameter.id", id),
		),
	)
	defer span.End()

	time.Sleep(time.Duration(5+rand.Intn(10)) * time.Millisecond)

	s.mu.RLock()
	defer s.mu.RUnlock()

	book, exists := s.books[id]
	if !exists {
		span.SetAttributes(attribute.Bool("db.no_rows", true))
		return Book{}, fmt.Errorf("book %s not found", id)
	}

	span.SetAttributes(attribute.Int("db.rows_returned", 1))
	return book, nil
}

func (s *BookStore) Create(ctx context.Context, book Book) error {
	tracer := otel.Tracer("bookstore")
	_, span := tracer.Start(ctx, "db.books.insert",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			attribute.String("db.system", "memory"),
			attribute.String("db.operation", "INSERT"),
			attribute.String("db.table", "books"),
		),
	)
	defer span.End()

	time.Sleep(time.Duration(15+rand.Intn(25)) * time.Millisecond)

	s.mu.Lock()
	defer s.mu.Unlock()

	s.books[book.ID] = book
	span.SetAttributes(attribute.Int("db.rows_affected", 1))
	return nil
}

func (s *BookStore) Delete(ctx context.Context, id string) error {
	tracer := otel.Tracer("bookstore")
	_, span := tracer.Start(ctx, "db.books.delete",
		trace.WithSpanKind(trace.SpanKindClient),
		trace.WithAttributes(
			attribute.String("db.system", "memory"),
			attribute.String("db.operation", "DELETE"),
			attribute.String("db.table", "books"),
			attribute.String("db.query_parameter.id", id),
		),
	)
	defer span.End()

	time.Sleep(time.Duration(10+rand.Intn(15)) * time.Millisecond)

	s.mu.Lock()
	defer s.mu.Unlock()

	if _, exists := s.books[id]; !exists {
		span.SetAttributes(attribute.Bool("db.no_rows", true))
		return fmt.Errorf("book %s not found", id)
	}

	delete(s.books, id)
	span.SetAttributes(attribute.Int("db.rows_affected", 1))
	return nil
}

// ============================================================
// Observability Setup
// ============================================================

type TraceContextLogHandler struct {
	inner slog.Handler
}

func NewTraceContextLogHandler(inner slog.Handler) *TraceContextLogHandler {
	return &TraceContextLogHandler{inner: inner}
}

func (h *TraceContextLogHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *TraceContextLogHandler) Handle(ctx context.Context, record slog.Record) error {
	span := trace.SpanFromContext(ctx)
	if span.SpanContext().IsValid() {
		record.AddAttrs(
			slog.String("trace_id", span.SpanContext().TraceID().String()),
			slog.String("span_id", span.SpanContext().SpanID().String()),
		)
	}
	return h.inner.Handle(ctx, record)
}

func (h *TraceContextLogHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &TraceContextLogHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *TraceContextLogHandler) WithGroup(name string) slog.Handler {
	return &TraceContextLogHandler{inner: h.inner.WithGroup(name)}
}

func initTelemetry() (*sdktrace.TracerProvider, *slog.Logger, error) {
	// Tracer
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, nil, fmt.Errorf("creating trace exporter: %w", err)
	}

	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("bookstore-api"),
			semconv.ServiceVersionKey.String("v1.0.0"),
			semconv.DeploymentEnvironmentKey.String("development"),
		),
	)
	if err != nil {
		return nil, nil, fmt.Errorf("creating resource: %w", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
	)

	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{},
			propagation.Baggage{},
		),
	)

	// Logger with trace correlation
	logger := slog.New(
		NewTraceContextLogHandler(
			slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
				Level: slog.LevelDebug,
			}),
		),
	)

	return tp, logger, nil
}

// ============================================================
// Middleware
// ============================================================

type statusResponseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
	wroteHeader  bool
}

func (w *statusResponseWriter) WriteHeader(code int) {
	if !w.wroteHeader {
		w.statusCode = code
		w.wroteHeader = true
		w.ResponseWriter.WriteHeader(code)
	}
}

func (w *statusResponseWriter) Write(b []byte) (int, error) {
	n, err := w.ResponseWriter.Write(b)
	w.bytesWritten += n
	return n, err
}

func tracingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	tracer := otel.Tracer("bookstore-api")
	propagator := otel.GetTextMapPropagator()

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// Extract trace context from incoming headers
			ctx := propagator.Extract(r.Context(), propagation.HeaderCarrier(r.Header))

			// Create server span
			spanName := fmt.Sprintf("%s %s", r.Method, r.URL.Path)
			ctx, span := tracer.Start(ctx, spanName,
				trace.WithSpanKind(trace.SpanKindServer),
				trace.WithAttributes(
					semconv.HTTPMethodKey.String(r.Method),
					semconv.HTTPTargetKey.String(r.URL.Path),
					attribute.String("http.host", r.Host),
					attribute.String("http.user_agent", r.UserAgent()),
					attribute.String("http.client_ip", r.RemoteAddr),
				),
			)
			defer span.End()

			// Wrap response writer
			wrapped := &statusResponseWriter{ResponseWriter: w, statusCode: 200}

			// Add trace ID to response headers for debugging
			if span.SpanContext().IsValid() {
				wrapped.Header().Set("X-Trace-ID", span.SpanContext().TraceID().String())
			}

			// Serve the request
			next.ServeHTTP(wrapped, r.WithContext(ctx))

			// Record response on span
			duration := time.Since(start)
			span.SetAttributes(
				semconv.HTTPStatusCodeKey.Int(wrapped.statusCode),
				attribute.Int("http.response_content_length", wrapped.bytesWritten),
			)

			if wrapped.statusCode >= 500 {
				span.SetStatus(codes.Error, fmt.Sprintf("HTTP %d", wrapped.statusCode))
			}

			// Correlated log line
			level := slog.LevelInfo
			if wrapped.statusCode >= 500 {
				level = slog.LevelError
			} else if wrapped.statusCode >= 400 {
				level = slog.LevelWarn
			}

			logger.LogAttrs(ctx, level, "request completed",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", wrapped.statusCode),
				slog.Duration("duration", duration),
				slog.Int("bytes", wrapped.bytesWritten),
				slog.String("client_ip", r.RemoteAddr),
			)
		})
	}
}

func recoveryMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if rec := recover(); rec != nil {
					span := trace.SpanFromContext(r.Context())
					err := fmt.Errorf("panic: %v", rec)
					span.RecordError(err)
					span.SetStatus(codes.Error, "panic recovered")

					logger.ErrorContext(r.Context(), "panic recovered",
						slog.Any("panic", rec),
						slog.String("method", r.Method),
						slog.String("path", r.URL.Path),
					)

					writeJSON(w, http.StatusInternalServerError, ErrorResponse{
						Error: "internal server error",
						Code:  "INTERNAL_ERROR",
					})
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// ============================================================
// Handlers
// ============================================================

type BookHandler struct {
	store  *BookStore
	logger *slog.Logger
}

func (h *BookHandler) ListBooks(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)

	h.logger.DebugContext(ctx, "listing all books")

	books, err := h.store.GetAll(ctx)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, "failed to list books")
		h.logger.ErrorContext(ctx, "failed to list books", slog.Any("error", err))
		writeJSON(w, http.StatusInternalServerError, ErrorResponse{
			Error:   "failed to retrieve books",
			Code:    "STORE_ERROR",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	span.SetAttributes(attribute.Int("books.count", len(books)))
	h.logger.InfoContext(ctx, "books listed", slog.Int("count", len(books)))

	writeJSON(w, http.StatusOK, books)
}

func (h *BookHandler) GetBook(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)

	bookID := r.PathValue("id")
	span.SetAttributes(attribute.String("book.id", bookID))

	h.logger.DebugContext(ctx, "fetching book", slog.String("book_id", bookID))

	book, err := h.store.GetByID(ctx, bookID)
	if err != nil {
		span.SetAttributes(attribute.Bool("book.not_found", true))
		h.logger.WarnContext(ctx, "book not found",
			slog.String("book_id", bookID),
		)
		writeJSON(w, http.StatusNotFound, ErrorResponse{
			Error:   fmt.Sprintf("book %s not found", bookID),
			Code:    "NOT_FOUND",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	span.SetAttributes(
		attribute.String("book.title", book.Title),
		attribute.String("book.category", book.Category),
		attribute.Float64("book.price", book.Price),
	)

	writeJSON(w, http.StatusOK, book)
}

func (h *BookHandler) CreateBook(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)

	var req CreateBookRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, "invalid request body")
		h.logger.WarnContext(ctx, "invalid request body", slog.Any("error", err))
		writeJSON(w, http.StatusBadRequest, ErrorResponse{
			Error:   "invalid request body",
			Code:    "INVALID_INPUT",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	// Validate
	if req.Title == "" || req.Author == "" || req.Price <= 0 {
		span.AddEvent("validation.failed",
			trace.WithAttributes(
				attribute.Bool("has_title", req.Title != ""),
				attribute.Bool("has_author", req.Author != ""),
				attribute.Bool("valid_price", req.Price > 0),
			),
		)
		writeJSON(w, http.StatusBadRequest, ErrorResponse{
			Error:   "title, author, and a positive price are required",
			Code:    "VALIDATION_ERROR",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	book := Book{
		ID:       fmt.Sprintf("book_%s", strconv.FormatInt(time.Now().UnixNano(), 36)),
		Title:    req.Title,
		Author:   req.Author,
		Price:    req.Price,
		InStock:  req.InStock,
		Category: req.Category,
	}

	span.SetAttributes(
		attribute.String("book.id", book.ID),
		attribute.String("book.title", book.Title),
		attribute.String("book.category", book.Category),
		attribute.Float64("book.price", book.Price),
	)

	if err := h.store.Create(ctx, book); err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, "failed to create book")
		h.logger.ErrorContext(ctx, "failed to create book", slog.Any("error", err))
		writeJSON(w, http.StatusInternalServerError, ErrorResponse{
			Error:   "failed to create book",
			Code:    "STORE_ERROR",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	h.logger.InfoContext(ctx, "book created",
		slog.String("book_id", book.ID),
		slog.String("title", book.Title),
	)

	writeJSON(w, http.StatusCreated, book)
}

func (h *BookHandler) DeleteBook(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)

	bookID := r.PathValue("id")
	span.SetAttributes(attribute.String("book.id", bookID))

	h.logger.InfoContext(ctx, "deleting book", slog.String("book_id", bookID))

	if err := h.store.Delete(ctx, bookID); err != nil {
		span.SetAttributes(attribute.Bool("book.not_found", true))
		h.logger.WarnContext(ctx, "book not found for deletion",
			slog.String("book_id", bookID),
		)
		writeJSON(w, http.StatusNotFound, ErrorResponse{
			Error:   fmt.Sprintf("book %s not found", bookID),
			Code:    "NOT_FOUND",
			TraceID: span.SpanContext().TraceID().String(),
		})
		return
	}

	h.logger.InfoContext(ctx, "book deleted", slog.String("book_id", bookID))
	w.WriteHeader(http.StatusNoContent)
}

func (h *BookHandler) HealthCheck(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{
		"status":  "ok",
		"service": "bookstore-api",
		"version": "v1.0.0",
	})
}

// ============================================================
// Helpers
// ============================================================

func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// ============================================================
// Main
// ============================================================

func main() {
	// Initialize telemetry
	tp, logger, err := initTelemetry()
	if err != nil {
		log.Fatalf("Failed to initialize telemetry: %v", err)
	}

	// Create store and handler
	store := NewBookStore()
	handler := &BookHandler{
		store:  store,
		logger: logger,
	}

	// Set up routes
	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/books", handler.ListBooks)
	mux.HandleFunc("GET /api/books/{id}", handler.GetBook)
	mux.HandleFunc("POST /api/books", handler.CreateBook)
	mux.HandleFunc("DELETE /api/books/{id}", handler.DeleteBook)
	mux.HandleFunc("GET /health", handler.HealthCheck)

	// Apply middleware (order matters: outermost runs first)
	var finalHandler http.Handler = mux
	finalHandler = tracingMiddleware(logger)(finalHandler)
	finalHandler = recoveryMiddleware(logger)(finalHandler)

	// Create server
	server := &http.Server{
		Addr:         ":8080",
		Handler:      finalHandler,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	// Graceful shutdown
	done := make(chan os.Signal, 1)
	signal.Notify(done, os.Interrupt, syscall.SIGTERM)

	go func() {
		logger.Info("server starting",
			slog.String("addr", server.Addr),
			slog.String("service", "bookstore-api"),
			slog.String("version", "v1.0.0"),
		)
		if err := server.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server failed", slog.Any("error", err))
			os.Exit(1)
		}
	}()

	fmt.Println("=== Bookstore API (Fully Instrumented) ===")
	fmt.Println()
	fmt.Println("Endpoints:")
	fmt.Println("  GET    /api/books       - List all books")
	fmt.Println("  GET    /api/books/{id}  - Get a book by ID")
	fmt.Println("  POST   /api/books       - Create a new book")
	fmt.Println("  DELETE /api/books/{id}  - Delete a book")
	fmt.Println("  GET    /health          - Health check")
	fmt.Println()
	fmt.Println("Try:")
	fmt.Println("  curl http://localhost:8080/api/books")
	fmt.Println("  curl http://localhost:8080/api/books/book_001")
	fmt.Println("  curl -X POST -H 'Content-Type: application/json' \\")
	fmt.Println("    -d '{\"title\":\"Go in Action\",\"author\":\"William Kennedy\",\"price\":39.99,\"in_stock\":10,\"category\":\"programming\"}' \\")
	fmt.Println("    http://localhost:8080/api/books")
	fmt.Println("  curl -X DELETE http://localhost:8080/api/books/book_001")
	fmt.Println()
	fmt.Println("Each request produces:")
	fmt.Println("  1. A distributed trace (printed to stdout as JSON)")
	fmt.Println("  2. Correlated log lines (with trace_id and span_id)")
	fmt.Println("  3. An X-Trace-ID response header for debugging")
	fmt.Println()
	fmt.Println("Press Ctrl+C to stop.")

	// Wait for shutdown signal
	<-done
	logger.Info("shutting down server")

	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer shutdownCancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		logger.Error("server shutdown error", slog.Any("error", err))
	}

	// Flush remaining spans
	if err := tp.Shutdown(shutdownCtx); err != nil {
		logger.Error("tracer shutdown error", slog.Any("error", err))
	}

	logger.Info("server stopped")

	_ = metric.Float64Counter.Add // suppress unused import
}
```

### What This Example Demonstrates

Every request to this API produces:

1. **A server span** with HTTP semantic conventions (method, path, status, duration)
2. **Database child spans** for each store operation (with query type, table, rows affected)
3. **Correlated log lines** with `trace_id` and `span_id` in every log record
4. **Error recording** on spans when things go wrong
5. **Span events** for notable occurrences (validation failures)
6. **Business context attributes** on spans (book.id, book.title, book.category)
7. **Trace ID in response headers** (`X-Trace-ID`) for client-side debugging
8. **Trace ID in error responses** so support teams can look up traces
9. **Recovery middleware** that records panics on spans
10. **Proper status propagation** -- 4xx logged as WARN, 5xx logged as ERROR

---

## 16. Key Takeaways

1. **Observability is not monitoring.** Monitoring answers pre-defined questions ("is error rate above 1%?"). Observability lets you ask arbitrary questions you did not anticipate ("why are German premium users on the new checkout flow seeing slow responses from warehouse eu-west-1?"). You need both, but observability is what saves you when monitoring's dashboards are all green and users are still complaining.

2. **High cardinality and high dimensionality are essential.** Traditional metrics with labels like `method` and `status` cannot answer questions about specific users, specific orders, or specific traces. You need fields with millions of unique values (user_id, trace_id, order_id) and you need many of them per event (20-50 fields). This is why structured events are the core unit of observability.

3. **The three pillars (logs, metrics, traces) are necessary but insufficient alone.** Each pillar has strengths and weaknesses. Logs are rich but hard to aggregate. Metrics are cheap but low-cardinality. Traces show causality but are expensive to store. The power comes from correlating them -- a trace_id in your logs lets you jump from a log line to its full distributed trace.

4. **Instrument at the boundaries.** The highest-value instrumentation points are where your system interacts with the outside world: incoming HTTP requests, outgoing HTTP calls, database queries, cache operations, message queue interactions. Instrumenting these boundaries gives you the most insight for the least effort.

5. **OpenTelemetry is the standard.** It is vendor-neutral, well-maintained, and supported by every major observability backend. Use it. The Go SDK provides TracerProvider for traces and MeterProvider for metrics. The contrib packages provide pre-built instrumentation for common libraries.

6. **Context propagation is the glue.** Distributed traces only work if trace context flows across service boundaries. In Go, this means passing `context.Context` everywhere and using W3C Trace Context headers (`traceparent`) for inter-service communication. Without propagation, each service produces isolated spans that cannot be assembled into a trace.

7. **Sampling is a necessity, not an optimization.** At scale, tracing every request is too expensive. Use `ParentBased(TraceIDRatioBased(rate))` for head-based sampling. Use the OpenTelemetry Collector's tail_sampling processor to keep 100% of errors and slow requests while sampling routine traffic at a lower rate.

8. **Go requires explicit instrumentation.** Unlike Node.js, which has auto-instrumentation that hooks into module loading, Go requires you to explicitly add tracing middleware, wrap HTTP clients, and instrument database calls. This is more work but gives you full control over what gets traced and what attributes are recorded.

9. **Build an observability library for your organization.** Do not make every team set up OpenTelemetry from scratch. Create a shared package that provides `Init()`, `Shutdown()`, `HTTPMiddleware()`, `HTTPClient()`, and a trace-correlated logger. This ensures consistent instrumentation across all services.

10. **Log correlation is trivial and transformative.** Adding `trace_id` and `span_id` to every log line costs almost nothing but lets you jump between logs and traces instantly. Use a custom `slog.Handler` that extracts trace context from `context.Context` and injects it into every log record.

11. **Error recording requires nuance.** Not all errors deserve `span.SetStatus(codes.Error)`. A 404 is usually not a server error -- it is expected behavior. A `context.Canceled` means the client disconnected. Categorize errors and record them appropriately on spans.

12. **Always record the trace ID in error responses.** When you return an error to a client, include the trace_id in the response body. This lets support teams look up the exact trace for a customer's problem, dramatically reducing time-to-diagnosis.

13. **Span names must be low-cardinality.** Use `GET /api/users/{id}`, not `GET /api/users/abc123`. High-cardinality span names make your tracing backend unusable because every unique name creates a separate entry in the service map.

14. **Shut down providers gracefully.** The `TracerProvider.Shutdown()` call flushes any remaining spans in the batch processor's queue. Without it, you lose the last few seconds of telemetry data, which is often the most important -- the data from right before a crash.

15. **Start with traces, add metrics second.** Traces give you more insight per unit of effort than metrics. Once you have traces working, add metrics for things you want to alert on and dashboard. Do not build a metrics-only system and call it observability.

---

## 17. Practice Exercises

### Exercise 1: Console Trace Exporter

Build a minimal tracing setup that:
- Creates a TracerProvider with a stdout exporter
- Creates a parent span called "ProcessOrder"
- Creates three child spans: "ValidateOrder" (20ms), "ChargePayment" (100ms), "SendConfirmation" (10ms)
- Adds at least 5 attributes to each span
- Runs and produces readable JSON trace output

### Exercise 2: HTTP Tracing Middleware

Build an HTTP server with tracing middleware that:
- Creates a span for every request with HTTP semantic conventions
- Records status code, duration, and response size on the span
- Propagates trace context from incoming headers
- Adds the trace ID to response headers as `X-Trace-ID`
- Test with at least 3 endpoints and verify spans appear in stdout

### Exercise 3: Distributed Trace Across Two Services

Build two HTTP services that demonstrate distributed tracing:
- Service A (port 8080) receives requests and calls Service B
- Service B (port 8081) processes the request and returns a response
- Both services create spans and propagate trace context via HTTP headers
- Verify that both services' spans share the same trace_id
- Print the complete trace showing the parent-child relationship

### Exercise 4: Log Correlation

Build a system that demonstrates log-trace correlation:
- Create a custom `slog.Handler` that injects `trace_id` and `span_id` from context
- Build an HTTP server that uses this logger
- For each request, emit at least 3 log lines at different points (start, middle, end)
- Verify that all log lines for a single request share the same trace_id
- Write a script that groups log lines by trace_id and prints them together

### Exercise 5: Custom Sampler

Implement a custom sampler that:
- Never samples health check endpoints (`/health`, `/ready`, `/live`)
- Always samples requests with an `X-Force-Sample: true` header
- Samples everything else at a configurable rate (default 10%)
- Write tests that verify each sampling behavior with 1000 test spans

### Exercise 6: Database Instrumentation

Build a `TracedDB` wrapper that:
- Wraps `*sql.DB` with automatic span creation for `Query`, `QueryRow`, and `Exec`
- Records the SQL operation type, table name, and duration
- Records rows affected for write operations
- Supports traced transactions (`BeginTx`, `Commit`, `Rollback`)
- Use SQLite (via `modernc.org/sqlite`) to test with a real database
- Create a simple CRUD API that uses TracedDB for all database operations

### Exercise 7: Structured Event Builder

Implement the structured event pattern from Section 4:
- Create an `EventBuilder` that accumulates fields throughout a request
- Store the builder in `context.Context` via middleware
- Add fields from different middleware layers (auth, rate limit, etc.)
- Add fields from handler and service code
- Emit the complete event as JSON when the request completes
- Include at least 20 fields in the final event

### Exercise 8: Metrics Integration

Add OpenTelemetry metrics to an HTTP server:
- `http_request_total` counter (by method, route, status)
- `http_request_duration_ms` histogram (by method, route)
- `http_request_in_flight` gauge
- `http_response_size_bytes` histogram
- Export metrics to stdout using the periodic reader
- Generate some traffic and verify the metric output

### Exercise 9: End-to-End Observability Stack

Build a complete observability stack:
- Write a `docker-compose.yml` with: your Go API, OpenTelemetry Collector, Jaeger (for traces), Prometheus (for metrics)
- Configure the Collector to receive OTLP and export to Jaeger and Prometheus
- Instrument your Go API with traces and metrics via OTLP exporter
- Generate traffic with a script that hits various endpoints
- Verify traces appear in Jaeger and metrics appear in Prometheus

### Exercise 10: Observability Library

Build a reusable observability library (`pkg/observe`) that:
- Provides `Init(Config)` to set up all providers
- Provides `Shutdown(ctx)` to gracefully flush and close
- Provides `HTTPMiddleware()` with tracing, logging, and metrics
- Provides `HTTPClient()` with automatic trace propagation
- Provides a trace-correlated `*slog.Logger`
- Reads configuration from environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, etc.)
- Use this library to build two microservices that call each other, and verify end-to-end trace propagation

---

*Next chapter: Chapter 41 will cover advanced observability topics including SLOs (Service Level Objectives), error budgets, alert design, and production debugging workflows.*
