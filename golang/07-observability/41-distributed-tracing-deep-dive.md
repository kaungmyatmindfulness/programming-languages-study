# Chapter 41: Distributed Tracing Deep Dive

> **Level: Advanced / Production** | **Prerequisites: Chapters 10 (Goroutines), 11 (Channels & Select), 12 (JSON, HTTP, REST APIs), 16 (Context Package), 17 (Advanced Concurrency), 21 (Building Microservices), 28 (Structured Logging & Observability)**

When a user clicks "Place Order" and nothing happens, who do you call? The frontend team says the request was sent. The API gateway shows a 504. The order service logs say it made a call to the inventory service. The inventory service has no record of that call. The payment service did charge the card. The email service never sent a confirmation.

You have seven services, thirty log streams, and no way to connect them. This is the problem distributed tracing solves.

This chapter builds a distributed tracing system from absolute first principles in Go. We will implement every component: trace and span IDs, context propagation, the W3C Trace Context standard, span exporters, batch processing, sampling strategies, middleware instrumentation, database tracing, goroutine propagation, error recording, trace analysis, and a complete multi-service example. The concepts are drawn from "Observability Engineering" by Charity Majors, Liz Fong-Jones, and George Miranda (Chapters 8-12), implemented entirely in Go.

By the end of this chapter, you will understand not just how to use OpenTelemetry, but how it works under the hood -- because you will have built your own tracer.

---

## Prerequisites

Before starting this chapter, you should be comfortable with:

- **Chapter 10: Goroutines and Concurrency Basics** -- Traces span across goroutines
- **Chapter 11: Channels and Select** -- Batch exporters use channels and tickers
- **Chapter 12: JSON, HTTP, and REST APIs** -- Trace context propagates via HTTP headers
- **Chapter 16: Context Package** -- The spine of trace propagation
- **Chapter 17: Advanced Concurrency** -- sync.Mutex, WaitGroup, worker pools for exporters
- **Chapter 21: Building Microservices** -- Tracing is a microservices observability primitive
- **Chapter 28: Structured Logging & Observability** -- Tracing extends structured logging with causal relationships

---

## Table of Contents

1. [What Is a Trace?](#1-what-is-a-trace)
2. [Building a Tracer from Scratch](#2-building-a-tracer-from-scratch)
3. [W3C Trace Context Standard](#3-w3c-trace-context-standard)
4. [Span Exporters](#4-span-exporters)
5. [Batch Processing and Export](#5-batch-processing-and-export)
6. [Sampling Strategies](#6-sampling-strategies)
7. [Trace Context Propagation Across Services](#7-trace-context-propagation-across-services)
8. [Instrumenting Middleware](#8-instrumenting-middleware)
9. [Database Tracing](#9-database-tracing)
10. [Async Operations and Goroutine Tracing](#10-async-operations-and-goroutine-tracing)
11. [Error Recording and Exception Tracking](#11-error-recording-and-exception-tracking)
12. [Trace Analysis Patterns](#12-trace-analysis-patterns)
13. [Building a Trace Visualizer](#13-building-a-trace-visualizer)
14. [Integration with Jaeger and Tempo](#14-integration-with-jaeger-and-tempo)
15. [Real-World Example: Multi-Service Tracing](#15-real-world-example-multi-service-tracing)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. What Is a Trace?

### The Core Problem

In a monolith, a request enters one process and exits the same process. You can trace it with a stack trace, a debugger, or a profiler. In a distributed system, a single user request might touch ten services, three databases, two caches, and a message queue. No single service has the full picture.

A **trace** is the complete record of a single request's journey through a distributed system. It is a directed acyclic graph (DAG) of operations called **spans**, connected by parent-child relationships.

```
User Request: GET /api/orders/123
│
├── api-gateway: route-request (12ms)
│   ├── auth-service: validate-token (3ms)
│   └── order-service: get-order (8ms)
│       ├── cache: redis-get (1ms) [cache miss]
│       ├── db: SELECT orders WHERE id=123 (4ms)
│       └── user-service: get-user (2ms)
│           └── db: SELECT users WHERE id=456 (1ms)
```

Each line in that tree is a **span**. The entire tree is a **trace**.

### Span Anatomy

A span represents a single unit of work within a trace. It carries timing information, metadata, status, and relationships.

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"time"
)

// TraceID is a 16-byte globally unique identifier for a trace.
// All spans in the same trace share the same TraceID.
type TraceID [16]byte

func (t TraceID) String() string {
	return hex.EncodeToString(t[:])
}

// SpanID is an 8-byte identifier unique within a trace.
type SpanID [8]byte

func (s SpanID) String() string {
	return hex.EncodeToString(s[:])
}

// IsZero returns true if the SpanID is all zeros (no parent).
func (s SpanID) IsZero() bool {
	for _, b := range s {
		if b != 0 {
			return false
		}
	}
	return true
}

// SpanStatus represents the outcome of the operation.
type SpanStatus int

const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

func (s SpanStatus) String() string {
	switch s {
	case StatusOK:
		return "OK"
	case StatusError:
		return "ERROR"
	default:
		return "UNSET"
	}
}

// SpanKind classifies the relationship of a span to its remote parent/child.
type SpanKind int

const (
	SpanKindInternal SpanKind = iota // Default, local operation
	SpanKindServer                   // Server-side of an RPC
	SpanKindClient                   // Client-side of an RPC
	SpanKindProducer                 // Message queue producer
	SpanKindConsumer                 // Message queue consumer
)

func (k SpanKind) String() string {
	switch k {
	case SpanKindServer:
		return "SERVER"
	case SpanKindClient:
		return "CLIENT"
	case SpanKindProducer:
		return "PRODUCER"
	case SpanKindConsumer:
		return "CONSUMER"
	default:
		return "INTERNAL"
	}
}

// SpanEvent is a timestamped annotation on a span.
// Events record things that happen at a specific moment during a span's lifetime.
type SpanEvent struct {
	Name       string
	Timestamp  time.Time
	Attributes map[string]any
}

// SpanLink connects a span to another span in the same or different trace.
// Links are used for batch operations or fan-in scenarios where a span
// is causally related to multiple other spans.
type SpanLink struct {
	TraceID    TraceID
	SpanID     SpanID
	Attributes map[string]any
}

// Span is the fundamental unit of work in a trace.
type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID // Zero value means root span
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	Events      []SpanEvent
	Links       []SpanLink
	ServiceName string
}

// Duration returns how long the span lasted.
func (s *Span) Duration() time.Duration {
	if s.EndTime.IsZero() {
		return time.Since(s.StartTime)
	}
	return s.EndTime.Sub(s.StartTime)
}

// IsRoot returns true if this span has no parent.
func (s *Span) IsRoot() bool {
	return s.ParentID.IsZero()
}

// SetAttribute adds a key-value pair to the span.
func (s *Span) SetAttribute(key string, value any) {
	if s.Attributes == nil {
		s.Attributes = make(map[string]any)
	}
	s.Attributes[key] = value
}

// AddEvent records a timestamped event on the span.
func (s *Span) AddEvent(name string, attrs map[string]any) {
	s.Events = append(s.Events, SpanEvent{
		Name:       name,
		Timestamp:  time.Now(),
		Attributes: attrs,
	})
}

// SetStatus sets the span's outcome status.
func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.Status = status
	s.StatusMsg = msg
}

// End marks the span as complete by recording the end time.
func (s *Span) End() {
	s.EndTime = time.Now()
}

// generateTraceID creates a new random TraceID.
func generateTraceID() TraceID {
	var id TraceID
	_, _ = rand.Read(id[:])
	return id
}

// generateSpanID creates a new random SpanID.
func generateSpanID() SpanID {
	var id SpanID
	_, _ = rand.Read(id[:])
	return id
}

func main() {
	// Create a root span (the first span in a trace)
	traceID := generateTraceID()

	rootSpan := &Span{
		TraceID:     traceID,
		SpanID:      generateSpanID(),
		Name:        "GET /api/orders/123",
		Kind:        SpanKindServer,
		StartTime:   time.Now(),
		ServiceName: "api-gateway",
		Attributes: map[string]any{
			"http.method":      "GET",
			"http.url":         "/api/orders/123",
			"http.user_agent":  "Mozilla/5.0",
			"net.peer.ip":      "10.0.0.42",
		},
	}

	// Simulate work
	time.Sleep(2 * time.Millisecond)

	// Create a child span
	dbSpan := &Span{
		TraceID:     traceID, // Same trace
		SpanID:      generateSpanID(),
		ParentID:    rootSpan.SpanID, // Parent is the root span
		Name:        "SELECT orders WHERE id = $1",
		Kind:        SpanKindClient,
		StartTime:   time.Now(),
		ServiceName: "api-gateway",
		Attributes: map[string]any{
			"db.system":    "postgresql",
			"db.statement": "SELECT * FROM orders WHERE id = $1",
			"db.name":      "orders_db",
		},
	}

	time.Sleep(4 * time.Millisecond)

	// Add an event to the DB span
	dbSpan.AddEvent("query.plan", map[string]any{
		"plan":      "Index Scan using orders_pkey",
		"rows_read": 1,
	})

	dbSpan.SetStatus(StatusOK, "")
	dbSpan.End()

	// Complete the root span
	rootSpan.SetAttribute("http.status_code", 200)
	rootSpan.SetStatus(StatusOK, "")
	rootSpan.End()

	// Print span details
	fmt.Println("=== Trace ===")
	fmt.Printf("Trace ID: %s\n\n", traceID)

	for _, span := range []*Span{rootSpan, dbSpan} {
		fmt.Printf("Span: %s\n", span.Name)
		fmt.Printf("  Span ID:    %s\n", span.SpanID)
		fmt.Printf("  Parent ID:  %s\n", span.ParentID)
		fmt.Printf("  Kind:       %s\n", span.Kind)
		fmt.Printf("  Service:    %s\n", span.ServiceName)
		fmt.Printf("  Duration:   %s\n", span.Duration())
		fmt.Printf("  Status:     %s\n", span.Status)
		fmt.Printf("  Is Root:    %v\n", span.IsRoot())
		fmt.Printf("  Attributes: %v\n", span.Attributes)
		fmt.Printf("  Events:     %d\n", len(span.Events))
		fmt.Println()
	}
}
```

### Trace Trees and Parent-Child Relationships

Every span except the root has a parent. This creates a tree structure:

```
Root Span (no parent)
├── Child Span A (parent = Root)
│   ├── Grandchild Span A1 (parent = A)
│   └── Grandchild Span A2 (parent = A)
└── Child Span B (parent = Root)
    └── Grandchild Span B1 (parent = B)
```

Key properties of the trace tree:

- **TraceID** is shared by all spans in the tree. It is the correlation key.
- **SpanID** is unique within the trace. It identifies a single operation.
- **ParentID** points to the parent span. A zero value means root.
- The tree can be arbitrarily deep (gateway -> service -> database -> connection pool).
- Siblings can overlap in time (parallel operations).

### The Critical Path

The **critical path** is the longest chain of sequentially dependent spans in a trace. It determines the total latency of the request. If you want to make a request faster, you must optimize operations on the critical path -- optimizing anything else is wasted effort.

```go
package main

import (
	"fmt"
	"time"
)

// SimpleSpan is a minimal span for critical path analysis.
type SimpleSpan struct {
	ID        string
	ParentID  string
	Name      string
	Start     time.Duration // Offset from trace start
	Duration  time.Duration
	Children  []*SimpleSpan
}

// End returns the end time offset from trace start.
func (s *SimpleSpan) End() time.Duration {
	return s.Start + s.Duration
}

// findCriticalPath returns the spans on the critical path.
// The critical path is the chain of spans that determines the total trace duration.
func findCriticalPath(root *SimpleSpan) []*SimpleSpan {
	if len(root.Children) == 0 {
		return []*SimpleSpan{root}
	}

	// Find the child whose subtree ends latest -- that child is on the critical path
	var latestChild *SimpleSpan
	var latestEnd time.Duration
	for _, child := range root.Children {
		childEnd := findSubtreeEnd(child)
		if childEnd > latestEnd {
			latestEnd = childEnd
			latestChild = child
		}
	}

	// Recurse into the latest-ending child
	childPath := findCriticalPath(latestChild)
	return append([]*SimpleSpan{root}, childPath...)
}

// findSubtreeEnd returns the latest end time in a subtree.
func findSubtreeEnd(span *SimpleSpan) time.Duration {
	latest := span.End()
	for _, child := range span.Children {
		childEnd := findSubtreeEnd(child)
		if childEnd > latest {
			latest = childEnd
		}
	}
	return latest
}

func main() {
	// Simulate a trace:
	//
	// api-gateway (0-120ms)
	// ├── auth-service (5-20ms)            15ms
	// └── order-service (20-115ms)         95ms
	//     ├── cache-lookup (22-24ms)       2ms
	//     ├── db-query (25-70ms)           45ms   <-- critical path
	//     └── user-service (30-90ms)       60ms   <-- critical path winner
	//         └── db-query (35-85ms)       50ms   <-- critical path

	userDB := &SimpleSpan{
		ID: "7", Name: "user-db-query", Start: 35 * time.Millisecond, Duration: 50 * time.Millisecond,
	}
	userService := &SimpleSpan{
		ID: "6", Name: "user-service", Start: 30 * time.Millisecond, Duration: 60 * time.Millisecond,
		Children: []*SimpleSpan{userDB},
	}
	orderDB := &SimpleSpan{
		ID: "5", Name: "order-db-query", Start: 25 * time.Millisecond, Duration: 45 * time.Millisecond,
	}
	cacheLookup := &SimpleSpan{
		ID: "4", Name: "cache-lookup", Start: 22 * time.Millisecond, Duration: 2 * time.Millisecond,
	}
	orderService := &SimpleSpan{
		ID: "3", Name: "order-service", Start: 20 * time.Millisecond, Duration: 95 * time.Millisecond,
		Children: []*SimpleSpan{cacheLookup, orderDB, userService},
	}
	authService := &SimpleSpan{
		ID: "2", Name: "auth-service", Start: 5 * time.Millisecond, Duration: 15 * time.Millisecond,
	}
	gateway := &SimpleSpan{
		ID: "1", Name: "api-gateway", Start: 0, Duration: 120 * time.Millisecond,
		Children: []*SimpleSpan{authService, orderService},
	}

	criticalPath := findCriticalPath(gateway)

	fmt.Println("=== Critical Path Analysis ===")
	fmt.Println()
	for i, span := range criticalPath {
		indent := ""
		for j := 0; j < i; j++ {
			indent += "  "
		}
		fmt.Printf("%s%s (%s - %s, duration: %s)\n",
			indent, span.Name,
			span.Start, span.End(), span.Duration)
	}

	fmt.Println()
	fmt.Println("To reduce total latency, optimize these spans.")
	fmt.Println("Optimizing auth-service or cache-lookup will NOT help --")
	fmt.Println("they are not on the critical path.")
}
```

Output:

```
=== Critical Path Analysis ===

api-gateway (0s - 120ms, duration: 120ms)
  order-service (20ms - 115ms, duration: 95ms)
    user-service (30ms - 90ms, duration: 60ms)
      user-db-query (35ms - 85ms, duration: 50ms)

To reduce total latency, optimize these spans.
Optimizing auth-service or cache-lookup will NOT help --
they are not on the critical path.
```

---

## 2. Building a Tracer from Scratch

### The Tracer Architecture

A tracer is the factory that creates spans. It manages:

1. **ID generation** -- Creating unique TraceIDs and SpanIDs
2. **Context propagation** -- Storing and retrieving spans from `context.Context`
3. **Sampling** -- Deciding which traces to record
4. **Exporting** -- Sending completed spans to a backend

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// ---------- ID Types ----------

type TraceID [16]byte

func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

func (t TraceID) IsZero() bool {
	for _, b := range t {
		if b != 0 {
			return false
		}
	}
	return true
}

type SpanID [8]byte

func (s SpanID) String() string { return hex.EncodeToString(s[:]) }

func (s SpanID) IsZero() bool {
	for _, b := range s {
		if b != 0 {
			return false
		}
	}
	return true
}

// ---------- Span ----------

type SpanStatus int

const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int

const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
	SpanKindProducer
	SpanKindConsumer
)

type SpanEvent struct {
	Name       string
	Timestamp  time.Time
	Attributes map[string]any
}

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	Events      []SpanEvent
	ServiceName string

	mu     sync.Mutex
	ended  bool
	tracer *Tracer // back-reference to the tracer that created this span
}

func (s *Span) Duration() time.Duration {
	if s.EndTime.IsZero() {
		return time.Since(s.StartTime)
	}
	return s.EndTime.Sub(s.StartTime)
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil {
		s.Attributes = make(map[string]any)
	}
	s.Attributes[key] = value
}

func (s *Span) AddEvent(name string, attrs map[string]any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Events = append(s.Events, SpanEvent{
		Name:       name,
		Timestamp:  time.Now(),
		Attributes: attrs,
	})
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) RecordError(err error) {
	if err == nil {
		return
	}
	s.SetStatus(StatusError, err.Error())
	s.AddEvent("exception", map[string]any{
		"exception.type":    fmt.Sprintf("%T", err),
		"exception.message": err.Error(),
	})
}

// End finalizes the span and exports it.
func (s *Span) End() {
	s.mu.Lock()
	if s.ended {
		s.mu.Unlock()
		return // End is idempotent
	}
	s.ended = true
	s.EndTime = time.Now()
	s.mu.Unlock()

	// Report the completed span to the tracer
	if s.tracer != nil && s.tracer.exporter != nil {
		_ = s.tracer.exporter.ExportSpans(context.Background(), []*Span{s})
	}
}

// ---------- SpanExporter Interface ----------

type SpanExporter interface {
	ExportSpans(ctx context.Context, spans []*Span) error
	Shutdown(ctx context.Context) error
}

// ---------- Sampler Interface ----------

type SamplingDecision int

const (
	Drop     SamplingDecision = iota // Do not record this trace
	RecordAndSample                  // Record and export this trace
)

type Sampler interface {
	ShouldSample(traceID TraceID, name string) SamplingDecision
}

// AlwaysOnSampler records every trace.
type AlwaysOnSampler struct{}

func (a *AlwaysOnSampler) ShouldSample(_ TraceID, _ string) SamplingDecision {
	return RecordAndSample
}

// ---------- Context Keys ----------

type contextKey struct{}

var spanContextKey = contextKey{}

// spanFromContext retrieves the current span from context.
func spanFromContext(ctx context.Context) *Span {
	span, _ := ctx.Value(spanContextKey).(*Span)
	return span
}

// contextWithSpan stores a span in the context.
func contextWithSpan(ctx context.Context, span *Span) context.Context {
	return context.WithValue(ctx, spanContextKey, span)
}

// ---------- ID Generation ----------

func generateTraceID() TraceID {
	var id TraceID
	_, _ = rand.Read(id[:])
	return id
}

func generateSpanID() SpanID {
	var id SpanID
	_, _ = rand.Read(id[:])
	return id
}

// ---------- Tracer ----------

type Tracer struct {
	serviceName string
	exporter    SpanExporter
	sampler     Sampler
}

// NewTracer creates a tracer with the given options.
func NewTracer(serviceName string, exporter SpanExporter, sampler Sampler) *Tracer {
	if sampler == nil {
		sampler = &AlwaysOnSampler{}
	}
	return &Tracer{
		serviceName: serviceName,
		exporter:    exporter,
		sampler:     sampler,
	}
}

// StartSpan creates a new span.
// If the context already contains a span, the new span becomes its child.
// If not, a new trace is started with a new TraceID.
func (t *Tracer) StartSpan(ctx context.Context, name string, kind ...SpanKind) (context.Context, *Span) {
	// Determine parent context
	parentSpan := spanFromContext(ctx)

	var traceID TraceID
	var parentID SpanID

	if parentSpan != nil {
		// Inherit the trace ID from the parent
		traceID = parentSpan.TraceID
		parentID = parentSpan.SpanID
	} else {
		// No parent -- start a new trace
		traceID = generateTraceID()
	}

	// Check sampling decision (only for root spans)
	if parentSpan == nil {
		decision := t.sampler.ShouldSample(traceID, name)
		if decision == Drop {
			// Return a no-op span that records nothing
			noopSpan := &Span{
				TraceID:   traceID,
				SpanID:    generateSpanID(),
				ParentID:  parentID,
				Name:      name,
				StartTime: time.Now(),
				// No tracer back-reference -- End() will not export
			}
			return contextWithSpan(ctx, noopSpan), noopSpan
		}
	}

	spanKind := SpanKindInternal
	if len(kind) > 0 {
		spanKind = kind[0]
	}

	span := &Span{
		TraceID:     traceID,
		SpanID:      generateSpanID(),
		ParentID:    parentID,
		Name:        name,
		Kind:        spanKind,
		StartTime:   time.Now(),
		ServiceName: t.serviceName,
		Attributes:  make(map[string]any),
		tracer:      t,
	}

	return contextWithSpan(ctx, span), span
}

// ---------- Console Exporter ----------

type ConsoleExporter struct {
	mu sync.Mutex
}

func (e *ConsoleExporter) ExportSpans(_ context.Context, spans []*Span) error {
	e.mu.Lock()
	defer e.mu.Unlock()
	for _, s := range spans {
		parent := "ROOT"
		if !s.ParentID.IsZero() {
			parent = s.ParentID.String()
		}
		fmt.Printf("[SPAN] trace=%s span=%s parent=%s name=%q service=%s duration=%s attrs=%v\n",
			s.TraceID, s.SpanID, parent, s.Name, s.ServiceName, s.Duration(), s.Attributes)
	}
	return nil
}

func (e *ConsoleExporter) Shutdown(_ context.Context) error {
	return nil
}

// ---------- Main: Demonstrate the Tracer ----------

func main() {
	exporter := &ConsoleExporter{}
	tracer := NewTracer("order-service", exporter, nil)

	// Start a root span
	ctx := context.Background()
	ctx, rootSpan := tracer.StartSpan(ctx, "HandleGetOrder", SpanKindServer)
	rootSpan.SetAttribute("http.method", "GET")
	rootSpan.SetAttribute("http.url", "/api/orders/123")

	// Start a child span for database query
	ctx2, dbSpan := tracer.StartSpan(ctx, "db.Query", SpanKindClient)
	dbSpan.SetAttribute("db.system", "postgresql")
	dbSpan.SetAttribute("db.statement", "SELECT * FROM orders WHERE id = $1")
	time.Sleep(5 * time.Millisecond) // simulate DB query
	dbSpan.SetStatus(StatusOK, "")
	dbSpan.End()

	// Start another child span for a downstream service call
	_, serviceSpan := tracer.StartSpan(ctx, "user-service.GetUser", SpanKindClient)
	serviceSpan.SetAttribute("rpc.service", "user-service")
	serviceSpan.SetAttribute("rpc.method", "GetUser")
	time.Sleep(3 * time.Millisecond)
	serviceSpan.SetStatus(StatusOK, "")
	serviceSpan.End()

	// Complete the root span
	rootSpan.SetAttribute("http.status_code", 200)
	rootSpan.SetStatus(StatusOK, "")
	rootSpan.End()

	_ = ctx2 // ctx2 would be passed to deeper operations
}
```

### How Context Propagation Works

The key insight is that `context.Context` forms a chain. When you call `tracer.StartSpan(ctx, ...)`, the tracer:

1. Looks for an existing span in `ctx` via `ctx.Value(spanContextKey)`
2. If found, the new span becomes a **child** (inherits TraceID, sets ParentID)
3. If not found, the new span is a **root** (generates new TraceID)
4. Returns a new context with the new span stored in it

This means context must flow through your entire call chain:

```go
func handleRequest(ctx context.Context) {
    ctx, span := tracer.StartSpan(ctx, "handleRequest")
    defer span.End()

    // ctx now contains this span
    result := queryDatabase(ctx)  // pass ctx forward
    enrichResult(ctx, result)     // pass ctx forward
}

func queryDatabase(ctx context.Context) *Result {
    ctx, span := tracer.StartSpan(ctx, "queryDatabase")
    defer span.End()
    // This span is automatically a child of "handleRequest"
    // because ctx contains the parent span
    return &Result{}
}
```

If you forget to pass context, the chain breaks and a new trace is started instead.

---

## 3. W3C Trace Context Standard

### Why a Standard Matters

Without a standard, every tracing system (Jaeger, Zipkin, Datadog, AWS X-Ray) uses different HTTP headers to propagate trace context. If service A uses Jaeger and service B uses Zipkin, the trace breaks at the boundary. The W3C Trace Context standard (https://www.w3.org/TR/trace-context/) defines two HTTP headers that all tracing systems can agree on:

- **`traceparent`** -- The core trace identification
- **`tracestate`** -- Vendor-specific supplementary data

### The traceparent Header

Format: `{version}-{trace-id}-{parent-id}-{trace-flags}`

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │   │                                │                │
             │   │                                │                └── flags (01 = sampled)
             │   │                                └── parent span id (8 bytes / 16 hex chars)
             │   └── trace id (16 bytes / 32 hex chars)
             └── version (always 00)
```

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"strings"
)

// TraceID is a 16-byte identifier.
type TraceID [16]byte

func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

func (t TraceID) IsZero() bool {
	for _, b := range t {
		if b != 0 {
			return false
		}
	}
	return true
}

// SpanID is an 8-byte identifier.
type SpanID [8]byte

func (s SpanID) String() string { return hex.EncodeToString(s[:]) }

func (s SpanID) IsZero() bool {
	for _, b := range s {
		if b != 0 {
			return false
		}
	}
	return true
}

// TraceFlags contains trace options.
type TraceFlags byte

const (
	FlagsSampled TraceFlags = 0x01 // Bit 0: sampled
)

func (f TraceFlags) IsSampled() bool {
	return f&FlagsSampled != 0
}

func (f TraceFlags) String() string {
	return fmt.Sprintf("%02x", byte(f))
}

// TraceContext holds the parsed trace context from W3C headers.
type TraceContext struct {
	Version    byte
	TraceID    TraceID
	SpanID     SpanID // The parent span ID from the perspective of the receiver
	TraceFlags TraceFlags
	TraceState string // Raw tracestate header value
}

// String formats the TraceContext as a traceparent header value.
func (tc *TraceContext) String() string {
	return fmt.Sprintf("%02x-%s-%s-%s",
		tc.Version,
		tc.TraceID,
		tc.SpanID,
		tc.TraceFlags,
	)
}

// IsSampled returns whether this trace is sampled.
func (tc *TraceContext) IsSampled() bool {
	return tc.TraceFlags.IsSampled()
}

// Errors for parsing.
var (
	ErrInvalidTraceParent = errors.New("invalid traceparent header")
	ErrInvalidTraceID     = errors.New("invalid trace-id: must be 32 hex chars and non-zero")
	ErrInvalidSpanID      = errors.New("invalid parent-id: must be 16 hex chars and non-zero")
	ErrUnsupportedVersion = errors.New("unsupported traceparent version")
)

// ParseTraceParent parses a W3C traceparent header.
// Format: {version}-{trace-id}-{parent-id}-{trace-flags}
// Example: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
func ParseTraceParent(header string) (*TraceContext, error) {
	header = strings.TrimSpace(header)

	parts := strings.Split(header, "-")
	if len(parts) < 4 {
		return nil, fmt.Errorf("%w: expected 4 parts, got %d", ErrInvalidTraceParent, len(parts))
	}

	// Parse version
	if len(parts[0]) != 2 {
		return nil, fmt.Errorf("%w: version must be 2 hex chars", ErrInvalidTraceParent)
	}
	versionBytes, err := hex.DecodeString(parts[0])
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidTraceParent, err)
	}
	version := versionBytes[0]

	// Version 255 is invalid per spec
	if version == 0xff {
		return nil, fmt.Errorf("%w: version ff is invalid", ErrUnsupportedVersion)
	}

	// For version 00, there must be exactly 4 parts
	if version == 0x00 && len(parts) != 4 {
		return nil, fmt.Errorf("%w: version 00 must have exactly 4 parts", ErrInvalidTraceParent)
	}

	// Parse trace-id (32 hex chars = 16 bytes)
	if len(parts[1]) != 32 {
		return nil, fmt.Errorf("%w: trace-id must be 32 hex chars, got %d", ErrInvalidTraceID, len(parts[1]))
	}
	traceIDBytes, err := hex.DecodeString(parts[1])
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidTraceID, err)
	}
	var traceID TraceID
	copy(traceID[:], traceIDBytes)
	if traceID.IsZero() {
		return nil, fmt.Errorf("%w: all zeros", ErrInvalidTraceID)
	}

	// Parse parent-id (16 hex chars = 8 bytes)
	if len(parts[2]) != 16 {
		return nil, fmt.Errorf("%w: parent-id must be 16 hex chars, got %d", ErrInvalidSpanID, len(parts[2]))
	}
	spanIDBytes, err := hex.DecodeString(parts[2])
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidSpanID, err)
	}
	var spanID SpanID
	copy(spanID[:], spanIDBytes)
	if spanID.IsZero() {
		return nil, fmt.Errorf("%w: all zeros", ErrInvalidSpanID)
	}

	// Parse trace-flags (2 hex chars = 1 byte)
	if len(parts[3]) != 2 {
		return nil, fmt.Errorf("%w: trace-flags must be 2 hex chars", ErrInvalidTraceParent)
	}
	flagsBytes, err := hex.DecodeString(parts[3])
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidTraceParent, err)
	}
	flags := TraceFlags(flagsBytes[0])

	return &TraceContext{
		Version:    version,
		TraceID:    traceID,
		SpanID:     spanID,
		TraceFlags: flags,
	}, nil
}

// ParseTraceState parses the tracestate header.
// tracestate is a list of key=value pairs separated by commas.
// Example: "rojo=00f067aa0ba902b7,congo=t61rcWkgMzE"
func ParseTraceState(header string) map[string]string {
	state := make(map[string]string)
	if header == "" {
		return state
	}
	pairs := strings.Split(header, ",")
	for _, pair := range pairs {
		pair = strings.TrimSpace(pair)
		parts := strings.SplitN(pair, "=", 2)
		if len(parts) == 2 {
			state[strings.TrimSpace(parts[0])] = strings.TrimSpace(parts[1])
		}
	}
	return state
}

// FormatTraceState formats a tracestate map back to a header value.
func FormatTraceState(state map[string]string) string {
	pairs := make([]string, 0, len(state))
	for k, v := range state {
		pairs = append(pairs, k+"="+v)
	}
	return strings.Join(pairs, ",")
}

// NewTraceContext creates a brand-new trace context (for root spans).
func NewTraceContext(sampled bool) *TraceContext {
	var traceID TraceID
	var spanID SpanID
	_, _ = rand.Read(traceID[:])
	_, _ = rand.Read(spanID[:])

	flags := TraceFlags(0)
	if sampled {
		flags = FlagsSampled
	}

	return &TraceContext{
		Version:    0x00,
		TraceID:    traceID,
		SpanID:     spanID,
		TraceFlags: flags,
	}
}

func main() {
	// Parse a valid traceparent
	header := "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
	tc, err := ParseTraceParent(header)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	fmt.Println("=== Parsed traceparent ===")
	fmt.Printf("Version:     %02x\n", tc.Version)
	fmt.Printf("Trace ID:    %s\n", tc.TraceID)
	fmt.Printf("Parent ID:   %s\n", tc.SpanID)
	fmt.Printf("Flags:       %s\n", tc.TraceFlags)
	fmt.Printf("Is Sampled:  %v\n", tc.IsSampled())
	fmt.Printf("Roundtrip:   %s\n", tc)
	fmt.Printf("Match:       %v\n", tc.String() == header)
	fmt.Println()

	// Parse tracestate
	traceState := "rojo=00f067aa0ba902b7,congo=t61rcWkgMzE"
	state := ParseTraceState(traceState)
	fmt.Println("=== Parsed tracestate ===")
	for k, v := range state {
		fmt.Printf("  %s = %s\n", k, v)
	}
	fmt.Println()

	// Create a new trace context
	newTC := NewTraceContext(true)
	fmt.Println("=== New Trace Context ===")
	fmt.Printf("traceparent: %s\n", newTC)
	fmt.Println()

	// Test error cases
	errorCases := []string{
		"",
		"00",
		"00-0000-0000-00",
		"ff-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
		"00-00000000000000000000000000000000-00f067aa0ba902b7-01", // zero trace-id
		"00-4bf92f3577b34da6a3ce929d0e0e4736-0000000000000000-01", // zero span-id
	}

	fmt.Println("=== Error Cases ===")
	for _, tc := range errorCases {
		_, err := ParseTraceParent(tc)
		fmt.Printf("  %q -> %v\n", tc, err)
	}
}
```

---

## 4. Span Exporters

### The Exporter Interface

Exporters are responsible for sending completed spans somewhere -- the console, a file, an HTTP endpoint (Jaeger, Tempo), or a collector (OTLP). The interface is simple: you give it a batch of spans, it sends them.

```go
package main

import (
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"sync"
	"time"
)

// ---------- Types ----------

type TraceID [16]byte

func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte

func (s SpanID) String() string { return hex.EncodeToString(s[:]) }

func (s SpanID) IsZero() bool {
	for _, b := range s {
		if b != 0 {
			return false
		}
	}
	return true
}

type SpanStatus int

const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int

const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type SpanEvent struct {
	Name       string         `json:"name"`
	Timestamp  time.Time      `json:"timestamp"`
	Attributes map[string]any `json:"attributes,omitempty"`
}

type Span struct {
	TraceID     TraceID        `json:"-"`
	SpanID      SpanID         `json:"-"`
	ParentID    SpanID         `json:"-"`
	Name        string         `json:"operationName"`
	Kind        SpanKind       `json:"kind"`
	StartTime   time.Time      `json:"startTime"`
	EndTime     time.Time      `json:"endTime"`
	Status      SpanStatus     `json:"status"`
	StatusMsg   string         `json:"statusMessage,omitempty"`
	Attributes  map[string]any `json:"attributes,omitempty"`
	Events      []SpanEvent    `json:"events,omitempty"`
	ServiceName string         `json:"serviceName"`
}

func (s *Span) Duration() time.Duration {
	return s.EndTime.Sub(s.StartTime)
}

// ---------- SpanExporter Interface ----------

type SpanExporter interface {
	ExportSpans(ctx context.Context, spans []*Span) error
	Shutdown(ctx context.Context) error
}

// ---------- Console Exporter ----------
// Writes spans to stdout in a human-readable format.

type ConsoleExporter struct {
	writer io.Writer
	mu     sync.Mutex
	pretty bool
}

func NewConsoleExporter(pretty bool) *ConsoleExporter {
	return &ConsoleExporter{
		writer: os.Stdout,
		pretty: pretty,
	}
}

type consoleSpan struct {
	TraceID     string         `json:"traceId"`
	SpanID      string         `json:"spanId"`
	ParentID    string         `json:"parentId,omitempty"`
	Name        string         `json:"name"`
	Kind        string         `json:"kind"`
	Service     string         `json:"service"`
	StartTime   string         `json:"startTime"`
	EndTime     string         `json:"endTime"`
	DurationMs  float64        `json:"durationMs"`
	Status      string         `json:"status"`
	StatusMsg   string         `json:"statusMessage,omitempty"`
	Attributes  map[string]any `json:"attributes,omitempty"`
	Events      []SpanEvent    `json:"events,omitempty"`
}

func kindString(k SpanKind) string {
	switch k {
	case SpanKindServer:
		return "SERVER"
	case SpanKindClient:
		return "CLIENT"
	default:
		return "INTERNAL"
	}
}

func statusString(s SpanStatus) string {
	switch s {
	case StatusOK:
		return "OK"
	case StatusError:
		return "ERROR"
	default:
		return "UNSET"
	}
}

func (e *ConsoleExporter) ExportSpans(_ context.Context, spans []*Span) error {
	e.mu.Lock()
	defer e.mu.Unlock()

	for _, s := range spans {
		cs := consoleSpan{
			TraceID:    s.TraceID.String(),
			SpanID:     s.SpanID.String(),
			Name:       s.Name,
			Kind:       kindString(s.Kind),
			Service:    s.ServiceName,
			StartTime:  s.StartTime.Format(time.RFC3339Nano),
			EndTime:    s.EndTime.Format(time.RFC3339Nano),
			DurationMs: float64(s.Duration().Microseconds()) / 1000.0,
			Status:     statusString(s.Status),
			StatusMsg:  s.StatusMsg,
			Attributes: s.Attributes,
			Events:     s.Events,
		}
		if !s.ParentID.IsZero() {
			cs.ParentID = s.ParentID.String()
		}

		var data []byte
		var err error
		if e.pretty {
			data, err = json.MarshalIndent(cs, "", "  ")
		} else {
			data, err = json.Marshal(cs)
		}
		if err != nil {
			return fmt.Errorf("console exporter: marshal error: %w", err)
		}

		_, err = fmt.Fprintf(e.writer, "%s\n", data)
		if err != nil {
			return fmt.Errorf("console exporter: write error: %w", err)
		}
	}
	return nil
}

func (e *ConsoleExporter) Shutdown(_ context.Context) error {
	return nil
}

// ---------- JSON File Exporter ----------
// Writes spans to a JSON Lines file for later analysis.

type JSONFileExporter struct {
	mu   sync.Mutex
	file *os.File
	enc  *json.Encoder
}

func NewJSONFileExporter(path string) (*JSONFileExporter, error) {
	f, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		return nil, fmt.Errorf("open trace file: %w", err)
	}
	return &JSONFileExporter{
		file: f,
		enc:  json.NewEncoder(f),
	}, nil
}

func (e *JSONFileExporter) ExportSpans(_ context.Context, spans []*Span) error {
	e.mu.Lock()
	defer e.mu.Unlock()

	for _, s := range spans {
		record := map[string]any{
			"traceId":    s.TraceID.String(),
			"spanId":     s.SpanID.String(),
			"parentId":   s.ParentID.String(),
			"name":       s.Name,
			"service":    s.ServiceName,
			"startTime":  s.StartTime.UnixNano(),
			"endTime":    s.EndTime.UnixNano(),
			"durationMs": float64(s.Duration().Microseconds()) / 1000.0,
			"status":     statusString(s.Status),
			"attributes": s.Attributes,
		}
		if err := e.enc.Encode(record); err != nil {
			return fmt.Errorf("json file exporter: encode error: %w", err)
		}
	}
	return nil
}

func (e *JSONFileExporter) Shutdown(_ context.Context) error {
	return e.file.Close()
}

// ---------- OTLP HTTP Exporter (Simplified) ----------
// Sends spans to an OpenTelemetry Collector via HTTP/JSON.

type OTLPHTTPExporter struct {
	endpoint string
	client   *http.Client
	headers  map[string]string
}

func NewOTLPHTTPExporter(endpoint string, headers map[string]string) *OTLPHTTPExporter {
	return &OTLPHTTPExporter{
		endpoint: endpoint,
		client: &http.Client{
			Timeout: 10 * time.Second,
		},
		headers: headers,
	}
}

// otlpPayload is a simplified OTLP trace export format.
type otlpPayload struct {
	ResourceSpans []otlpResourceSpan `json:"resourceSpans"`
}

type otlpResourceSpan struct {
	Resource  otlpResource    `json:"resource"`
	ScopeSpans []otlpScopeSpan `json:"scopeSpans"`
}

type otlpResource struct {
	Attributes []otlpKeyValue `json:"attributes"`
}

type otlpScopeSpan struct {
	Spans []otlpSpan `json:"spans"`
}

type otlpSpan struct {
	TraceID            string         `json:"traceId"`
	SpanID             string         `json:"spanId"`
	ParentSpanID       string         `json:"parentSpanId,omitempty"`
	Name               string         `json:"name"`
	Kind               int            `json:"kind"`
	StartTimeUnixNano  int64          `json:"startTimeUnixNano,string"`
	EndTimeUnixNano    int64          `json:"endTimeUnixNano,string"`
	Attributes         []otlpKeyValue `json:"attributes,omitempty"`
	Status             otlpStatus     `json:"status"`
}

type otlpKeyValue struct {
	Key   string    `json:"key"`
	Value otlpValue `json:"value"`
}

type otlpValue struct {
	StringValue string `json:"stringValue,omitempty"`
	IntValue    string `json:"intValue,omitempty"`
}

type otlpStatus struct {
	Code    int    `json:"code"`
	Message string `json:"message,omitempty"`
}

func (e *OTLPHTTPExporter) ExportSpans(ctx context.Context, spans []*Span) error {
	if len(spans) == 0 {
		return nil
	}

	// Group spans by service name
	byService := make(map[string][]*Span)
	for _, s := range spans {
		byService[s.ServiceName] = append(byService[s.ServiceName], s)
	}

	payload := otlpPayload{}

	for service, serviceSpans := range byService {
		var otlpSpans []otlpSpan
		for _, s := range serviceSpans {
			os := otlpSpan{
				TraceID:           s.TraceID.String(),
				SpanID:            s.SpanID.String(),
				Name:              s.Name,
				Kind:              int(s.Kind) + 1, // OTLP kind is 1-indexed
				StartTimeUnixNano: s.StartTime.UnixNano(),
				EndTimeUnixNano:   s.EndTime.UnixNano(),
				Status:            otlpStatus{Code: int(s.Status)},
			}
			if !s.ParentID.IsZero() {
				os.ParentSpanID = s.ParentID.String()
			}
			for k, v := range s.Attributes {
				os.Attributes = append(os.Attributes, otlpKeyValue{
					Key:   k,
					Value: otlpValue{StringValue: fmt.Sprintf("%v", v)},
				})
			}
			otlpSpans = append(otlpSpans, os)
		}

		payload.ResourceSpans = append(payload.ResourceSpans, otlpResourceSpan{
			Resource: otlpResource{
				Attributes: []otlpKeyValue{
					{Key: "service.name", Value: otlpValue{StringValue: service}},
				},
			},
			ScopeSpans: []otlpScopeSpan{
				{Spans: otlpSpans},
			},
		})
	}

	data, err := json.Marshal(payload)
	if err != nil {
		return fmt.Errorf("otlp exporter: marshal error: %w", err)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost, e.endpoint, bytes.NewReader(data))
	if err != nil {
		return fmt.Errorf("otlp exporter: create request error: %w", err)
	}
	req.Header.Set("Content-Type", "application/json")
	for k, v := range e.headers {
		req.Header.Set(k, v)
	}

	resp, err := e.client.Do(req)
	if err != nil {
		return fmt.Errorf("otlp exporter: request error: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 400 {
		body, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("otlp exporter: server returned %d: %s", resp.StatusCode, string(body))
	}

	return nil
}

func (e *OTLPHTTPExporter) Shutdown(_ context.Context) error {
	return nil
}

// ---------- Multi Exporter ----------
// Sends spans to multiple exporters (fan-out).

type MultiExporter struct {
	exporters []SpanExporter
}

func NewMultiExporter(exporters ...SpanExporter) *MultiExporter {
	return &MultiExporter{exporters: exporters}
}

func (e *MultiExporter) ExportSpans(ctx context.Context, spans []*Span) error {
	var firstErr error
	for _, exp := range e.exporters {
		if err := exp.ExportSpans(ctx, spans); err != nil && firstErr == nil {
			firstErr = err
		}
	}
	return firstErr
}

func (e *MultiExporter) Shutdown(ctx context.Context) error {
	var firstErr error
	for _, exp := range e.exporters {
		if err := exp.Shutdown(ctx); err != nil && firstErr == nil {
			firstErr = err
		}
	}
	return firstErr
}

// ---------- Demo ----------

func main() {
	// Create a console exporter with pretty printing
	console := NewConsoleExporter(true)

	// Create test spans
	var traceID TraceID
	_, _ = rand.Read(traceID[:])
	var rootSpanID, childSpanID SpanID
	_, _ = rand.Read(rootSpanID[:])
	_, _ = rand.Read(childSpanID[:])

	now := time.Now()
	spans := []*Span{
		{
			TraceID:     traceID,
			SpanID:      rootSpanID,
			Name:        "GET /api/orders",
			Kind:        SpanKindServer,
			StartTime:   now,
			EndTime:     now.Add(50 * time.Millisecond),
			Status:      StatusOK,
			ServiceName: "api-gateway",
			Attributes: map[string]any{
				"http.method":      "GET",
				"http.status_code": 200,
			},
		},
		{
			TraceID:     traceID,
			SpanID:      childSpanID,
			ParentID:    rootSpanID,
			Name:        "db.Query",
			Kind:        SpanKindClient,
			StartTime:   now.Add(5 * time.Millisecond),
			EndTime:     now.Add(35 * time.Millisecond),
			Status:      StatusOK,
			ServiceName: "api-gateway",
			Attributes: map[string]any{
				"db.system":    "postgresql",
				"db.statement": "SELECT * FROM orders LIMIT 20",
			},
		},
	}

	fmt.Println("=== Console Exporter Output ===")
	if err := console.ExportSpans(context.Background(), spans); err != nil {
		fmt.Printf("Error: %v\n", err)
	}
}
```

---

## 5. Batch Processing and Export

### Why Batching Matters

Exporting each span individually is extremely wasteful. If your service handles 10,000 requests per second and each request creates 5 spans, that is 50,000 export calls per second. Batching groups spans together and exports them periodically or when a size threshold is reached, reducing network overhead by orders of magnitude.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// ---------- Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	Attributes  map[string]any
	ServiceName string
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- SpanExporter Interface ----------

type SpanExporter interface {
	ExportSpans(ctx context.Context, spans []*Span) error
	Shutdown(ctx context.Context) error
}

// ---------- Counting Exporter (for testing) ----------

type CountingExporter struct {
	mu         sync.Mutex
	BatchCount int
	SpanCount  int
}

func (e *CountingExporter) ExportSpans(_ context.Context, spans []*Span) error {
	e.mu.Lock()
	defer e.mu.Unlock()
	e.BatchCount++
	e.SpanCount += len(spans)
	fmt.Printf("  [export] batch #%d: %d spans (total: %d spans)\n",
		e.BatchCount, len(spans), e.SpanCount)
	return nil
}

func (e *CountingExporter) Shutdown(_ context.Context) error { return nil }

// ---------- Batch Exporter ----------

// BatchExporter collects spans in a queue and periodically exports them
// in batches to a delegate exporter. This reduces network calls and
// improves throughput.
type BatchExporter struct {
	delegate SpanExporter

	// Configuration
	maxBatchSize  int           // Maximum spans per export call
	maxQueueSize  int           // Maximum spans waiting in the queue
	flushInterval time.Duration // How often to flush
	exportTimeout time.Duration // Timeout for each export call

	// Internal state
	queue    chan *Span
	stopCh   chan struct{}
	doneCh   chan struct{}
	mu       sync.Mutex
	stopped  bool

	// Metrics
	droppedCount int64
}

// BatchExporterConfig holds configuration for a BatchExporter.
type BatchExporterConfig struct {
	MaxBatchSize  int
	MaxQueueSize  int
	FlushInterval time.Duration
	ExportTimeout time.Duration
}

// DefaultBatchConfig returns sensible defaults.
func DefaultBatchConfig() BatchExporterConfig {
	return BatchExporterConfig{
		MaxBatchSize:  512,
		MaxQueueSize:  2048,
		FlushInterval: 5 * time.Second,
		ExportTimeout: 30 * time.Second,
	}
}

// NewBatchExporter creates and starts a batch exporter.
func NewBatchExporter(delegate SpanExporter, config BatchExporterConfig) *BatchExporter {
	if config.MaxBatchSize <= 0 {
		config.MaxBatchSize = 512
	}
	if config.MaxQueueSize <= 0 {
		config.MaxQueueSize = 2048
	}
	if config.FlushInterval <= 0 {
		config.FlushInterval = 5 * time.Second
	}
	if config.ExportTimeout <= 0 {
		config.ExportTimeout = 30 * time.Second
	}

	be := &BatchExporter{
		delegate:      delegate,
		maxBatchSize:  config.MaxBatchSize,
		maxQueueSize:  config.MaxQueueSize,
		flushInterval: config.FlushInterval,
		exportTimeout: config.ExportTimeout,
		queue:         make(chan *Span, config.MaxQueueSize),
		stopCh:        make(chan struct{}),
		doneCh:        make(chan struct{}),
	}

	go be.processLoop()
	return be
}

// ExportSpans adds spans to the queue. This is non-blocking.
// If the queue is full, spans are dropped (backpressure).
func (be *BatchExporter) ExportSpans(_ context.Context, spans []*Span) error {
	be.mu.Lock()
	if be.stopped {
		be.mu.Unlock()
		return fmt.Errorf("batch exporter is shut down")
	}
	be.mu.Unlock()

	for _, s := range spans {
		select {
		case be.queue <- s:
			// Successfully queued
		default:
			// Queue is full -- drop the span (backpressure)
			be.mu.Lock()
			be.droppedCount++
			be.mu.Unlock()
		}
	}
	return nil
}

// processLoop runs in a goroutine. It collects spans from the queue
// and exports them when the batch is full or the flush interval elapses.
func (be *BatchExporter) processLoop() {
	defer close(be.doneCh)

	ticker := time.NewTicker(be.flushInterval)
	defer ticker.Stop()

	batch := make([]*Span, 0, be.maxBatchSize)

	for {
		select {
		case span := <-be.queue:
			batch = append(batch, span)
			if len(batch) >= be.maxBatchSize {
				be.exportBatch(batch)
				batch = make([]*Span, 0, be.maxBatchSize)
			}

		case <-ticker.C:
			if len(batch) > 0 {
				be.exportBatch(batch)
				batch = make([]*Span, 0, be.maxBatchSize)
			}

		case <-be.stopCh:
			// Drain remaining spans from the queue
			for {
				select {
				case span := <-be.queue:
					batch = append(batch, span)
					if len(batch) >= be.maxBatchSize {
						be.exportBatch(batch)
						batch = make([]*Span, 0, be.maxBatchSize)
					}
				default:
					// Queue is empty
					if len(batch) > 0 {
						be.exportBatch(batch)
					}
					return
				}
			}
		}
	}
}

// exportBatch sends a batch of spans to the delegate exporter.
func (be *BatchExporter) exportBatch(batch []*Span) {
	ctx, cancel := context.WithTimeout(context.Background(), be.exportTimeout)
	defer cancel()

	if err := be.delegate.ExportSpans(ctx, batch); err != nil {
		fmt.Printf("  [batch] export error: %v\n", err)
	}
}

// Shutdown stops the batch exporter and flushes remaining spans.
func (be *BatchExporter) Shutdown(ctx context.Context) error {
	be.mu.Lock()
	if be.stopped {
		be.mu.Unlock()
		return nil
	}
	be.stopped = true
	be.mu.Unlock()

	close(be.stopCh)

	// Wait for the process loop to finish or context to expire
	select {
	case <-be.doneCh:
		return be.delegate.Shutdown(ctx)
	case <-ctx.Done():
		return fmt.Errorf("batch exporter shutdown timed out: %w", ctx.Err())
	}
}

// DroppedCount returns how many spans were dropped due to backpressure.
func (be *BatchExporter) DroppedCount() int64 {
	be.mu.Lock()
	defer be.mu.Unlock()
	return be.droppedCount
}

// ---------- Demo ----------

func main() {
	counter := &CountingExporter{}

	// Create a batch exporter with a small batch size and fast flush for demo
	batcher := NewBatchExporter(counter, BatchExporterConfig{
		MaxBatchSize:  10,
		MaxQueueSize:  100,
		FlushInterval: 200 * time.Millisecond,
		ExportTimeout: 5 * time.Second,
	})

	// Generate 55 spans rapidly
	fmt.Println("=== Adding 55 spans ===")
	now := time.Now()
	for i := 0; i < 55; i++ {
		var traceID TraceID
		var spanID SpanID
		_, _ = rand.Read(traceID[:])
		_, _ = rand.Read(spanID[:])

		span := &Span{
			TraceID:     traceID,
			SpanID:      spanID,
			Name:        fmt.Sprintf("operation-%d", i),
			StartTime:   now,
			EndTime:     now.Add(time.Duration(i) * time.Millisecond),
			Status:      StatusOK,
			ServiceName: "test-service",
		}
		_ = batcher.ExportSpans(context.Background(), []*Span{span})
	}

	// Wait for all batches to be flushed
	fmt.Println("\n=== Waiting for flush ===")
	time.Sleep(500 * time.Millisecond)

	// Shutdown and flush remaining
	fmt.Println("\n=== Shutting down ===")
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := batcher.Shutdown(ctx); err != nil {
		fmt.Printf("Shutdown error: %v\n", err)
	}

	fmt.Printf("\nFinal: %d batches, %d spans, %d dropped\n",
		counter.BatchCount, counter.SpanCount, batcher.DroppedCount())
}
```

### Retry with Exponential Backoff

Production exporters must handle transient failures. Here is a retry wrapper:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"math"
	"sync/atomic"
	"time"
)

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }
type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
type SpanStatus int
const StatusOK SpanStatus = 1

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	Attributes  map[string]any
	ServiceName string
}

type SpanExporter interface {
	ExportSpans(ctx context.Context, spans []*Span) error
	Shutdown(ctx context.Context) error
}

// RetryExporter wraps another exporter with retry logic.
type RetryExporter struct {
	delegate    SpanExporter
	maxRetries  int
	initialWait time.Duration
	maxWait     time.Duration
}

func NewRetryExporter(delegate SpanExporter, maxRetries int) *RetryExporter {
	return &RetryExporter{
		delegate:    delegate,
		maxRetries:  maxRetries,
		initialWait: 100 * time.Millisecond,
		maxWait:     10 * time.Second,
	}
}

func (r *RetryExporter) ExportSpans(ctx context.Context, spans []*Span) error {
	var lastErr error
	for attempt := 0; attempt <= r.maxRetries; attempt++ {
		err := r.delegate.ExportSpans(ctx, spans)
		if err == nil {
			return nil
		}
		lastErr = err

		// Do not retry if context is cancelled
		if ctx.Err() != nil {
			return fmt.Errorf("export cancelled: %w", lastErr)
		}

		// Check if error is retryable
		if !isRetryable(err) {
			return err
		}

		// Exponential backoff: 100ms, 200ms, 400ms, 800ms, ...
		wait := time.Duration(float64(r.initialWait) * math.Pow(2, float64(attempt)))
		if wait > r.maxWait {
			wait = r.maxWait
		}

		fmt.Printf("  [retry] attempt %d failed: %v (retrying in %s)\n", attempt+1, err, wait)

		select {
		case <-time.After(wait):
		case <-ctx.Done():
			return fmt.Errorf("export cancelled during backoff: %w", ctx.Err())
		}
	}
	return fmt.Errorf("export failed after %d retries: %w", r.maxRetries, lastErr)
}

func (r *RetryExporter) Shutdown(ctx context.Context) error {
	return r.delegate.Shutdown(ctx)
}

// isRetryable determines if an error should be retried.
func isRetryable(err error) bool {
	// In a real system, you would check for specific error types:
	// - HTTP 429 (rate limited)
	// - HTTP 503 (service unavailable)
	// - Network timeouts
	// - Connection refused
	var retryErr *RetryableError
	return errors.As(err, &retryErr)
}

// RetryableError is an error that the retry logic should attempt again.
type RetryableError struct {
	Err error
}

func (e *RetryableError) Error() string { return e.Err.Error() }
func (e *RetryableError) Unwrap() error { return e.Err }

// ---------- Flaky Exporter (for testing) ----------

type FlakyExporter struct {
	callCount atomic.Int32
	failUntil int32
}

func (f *FlakyExporter) ExportSpans(_ context.Context, spans []*Span) error {
	count := f.callCount.Add(1)
	if count <= f.failUntil {
		return &RetryableError{Err: fmt.Errorf("connection refused (attempt %d)", count)}
	}
	fmt.Printf("  [flaky] export succeeded on attempt %d: %d spans\n", count, len(spans))
	return nil
}

func (f *FlakyExporter) Shutdown(_ context.Context) error { return nil }

// ---------- Demo ----------

func main() {
	// Create a flaky exporter that fails the first 3 attempts
	flaky := &FlakyExporter{failUntil: 3}
	retrier := NewRetryExporter(flaky, 5)

	var traceID TraceID
	var spanID SpanID
	_, _ = rand.Read(traceID[:])
	_, _ = rand.Read(spanID[:])

	spans := []*Span{{
		TraceID:     traceID,
		SpanID:      spanID,
		Name:        "test-operation",
		StartTime:   time.Now(),
		EndTime:     time.Now().Add(10 * time.Millisecond),
		Status:      StatusOK,
		ServiceName: "demo",
	}}

	fmt.Println("=== Retry Exporter Demo ===")
	err := retrier.ExportSpans(context.Background(), spans)
	if err != nil {
		fmt.Printf("Final error: %v\n", err)
	} else {
		fmt.Println("Export succeeded!")
	}
}
```

---

## 6. Sampling Strategies

### Why Sample?

High-traffic services can produce millions of spans per minute. Sending all of them to a tracing backend is expensive (network bandwidth, storage, processing). Sampling lets you keep a representative subset of traces while controlling cost.

The key insight from "Observability Engineering": sampling should be a knob, not a switch. You want to keep 100% of interesting traces (errors, slow requests) and a sample of normal traces.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"math"
	"sync"
	"time"
)

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }

// ---------- Sampling Decision ----------

type SamplingDecision int

const (
	Drop            SamplingDecision = iota // Do not record
	RecordAndSample                         // Record and export
)

func (d SamplingDecision) String() string {
	if d == RecordAndSample {
		return "SAMPLE"
	}
	return "DROP"
}

// ---------- Sampler Interface ----------

type SamplingParams struct {
	TraceID  TraceID
	Name     string
	Kind     int
	ParentSampled *bool // nil if no parent, pointer to parent's sampling decision
}

type Sampler interface {
	ShouldSample(params SamplingParams) SamplingDecision
	Description() string
}

// ---------- AlwaysOn Sampler ----------
// Records every single trace. Use in development or low-traffic systems.

type AlwaysOnSampler struct{}

func (s *AlwaysOnSampler) ShouldSample(_ SamplingParams) SamplingDecision {
	return RecordAndSample
}

func (s *AlwaysOnSampler) Description() string { return "AlwaysOnSampler" }

// ---------- AlwaysOff Sampler ----------
// Drops every trace. Useful for disabling tracing entirely.

type AlwaysOffSampler struct{}

func (s *AlwaysOffSampler) ShouldSample(_ SamplingParams) SamplingDecision {
	return Drop
}

func (s *AlwaysOffSampler) Description() string { return "AlwaysOffSampler" }

// ---------- Probability Sampler ----------
// Samples a deterministic fraction of traces based on TraceID.
// If rate=0.1, 10% of traces are sampled.
// The decision is deterministic: the same TraceID always produces the
// same decision, ensuring all spans in a trace are consistently sampled.

type ProbabilitySampler struct {
	rate      float64
	threshold uint64
}

func NewProbabilitySampler(rate float64) *ProbabilitySampler {
	if rate <= 0 {
		rate = 0
	}
	if rate >= 1 {
		rate = 1
	}
	return &ProbabilitySampler{
		rate:      rate,
		threshold: uint64(rate * float64(math.MaxUint64)),
	}
}

func (s *ProbabilitySampler) ShouldSample(params SamplingParams) SamplingDecision {
	if s.rate >= 1 {
		return RecordAndSample
	}
	if s.rate <= 0 {
		return Drop
	}

	// Use the last 8 bytes of TraceID as a deterministic hash.
	// This ensures all services make the same decision for the same trace.
	hash := binary.BigEndian.Uint64(params.TraceID[8:])
	if hash < s.threshold {
		return RecordAndSample
	}
	return Drop
}

func (s *ProbabilitySampler) Description() string {
	return fmt.Sprintf("ProbabilitySampler{rate=%.4f}", s.rate)
}

// ---------- Rate Limiting Sampler ----------
// Samples at most N traces per second. Uses a token bucket algorithm.

type RateLimitingSampler struct {
	maxPerSecond float64
	tokens       float64
	lastRefill   time.Time
	mu           sync.Mutex
}

func NewRateLimitingSampler(maxPerSecond float64) *RateLimitingSampler {
	return &RateLimitingSampler{
		maxPerSecond: maxPerSecond,
		tokens:       maxPerSecond, // Start full
		lastRefill:   time.Now(),
	}
}

func (s *RateLimitingSampler) ShouldSample(_ SamplingParams) SamplingDecision {
	s.mu.Lock()
	defer s.mu.Unlock()

	// Refill tokens based on elapsed time
	now := time.Now()
	elapsed := now.Sub(s.lastRefill).Seconds()
	s.tokens += elapsed * s.maxPerSecond
	if s.tokens > s.maxPerSecond {
		s.tokens = s.maxPerSecond
	}
	s.lastRefill = now

	// Try to consume a token
	if s.tokens >= 1 {
		s.tokens--
		return RecordAndSample
	}
	return Drop
}

func (s *RateLimitingSampler) Description() string {
	return fmt.Sprintf("RateLimitingSampler{max=%.0f/s}", s.maxPerSecond)
}

// ---------- Parent-Based Sampler ----------
// Defers to the parent's sampling decision if a parent exists.
// For root spans (no parent), delegates to a configurable root sampler.
// This ensures that once a trace is sampled, all child spans are also sampled.

type ParentBasedSampler struct {
	root Sampler // Used when there is no parent
}

func NewParentBasedSampler(root Sampler) *ParentBasedSampler {
	return &ParentBasedSampler{root: root}
}

func (s *ParentBasedSampler) ShouldSample(params SamplingParams) SamplingDecision {
	if params.ParentSampled != nil {
		// Parent exists -- follow its decision
		if *params.ParentSampled {
			return RecordAndSample
		}
		return Drop
	}
	// No parent -- this is a root span, use root sampler
	return s.root.ShouldSample(params)
}

func (s *ParentBasedSampler) Description() string {
	return fmt.Sprintf("ParentBasedSampler{root=%s}", s.root.Description())
}

// ---------- Composite Sampler ----------
// Samples if ANY of the delegate samplers say yes.
// Useful for "always sample errors" + "sample 10% of normal traffic".

type CompositeSampler struct {
	samplers []Sampler
}

func NewCompositeSampler(samplers ...Sampler) *CompositeSampler {
	return &CompositeSampler{samplers: samplers}
}

func (s *CompositeSampler) ShouldSample(params SamplingParams) SamplingDecision {
	for _, sampler := range s.samplers {
		if sampler.ShouldSample(params) == RecordAndSample {
			return RecordAndSample
		}
	}
	return Drop
}

func (s *CompositeSampler) Description() string {
	return "CompositeSampler"
}

// ---------- Name-Based Sampler ----------
// Always samples spans with specific operation names (e.g., health checks get dropped).

type NameBasedSampler struct {
	rules    map[string]SamplingDecision
	fallback Sampler
}

func NewNameBasedSampler(rules map[string]SamplingDecision, fallback Sampler) *NameBasedSampler {
	return &NameBasedSampler{rules: rules, fallback: fallback}
}

func (s *NameBasedSampler) ShouldSample(params SamplingParams) SamplingDecision {
	if decision, ok := s.rules[params.Name]; ok {
		return decision
	}
	return s.fallback.ShouldSample(params)
}

func (s *NameBasedSampler) Description() string {
	return "NameBasedSampler"
}

// ---------- Demo ----------

func generateTraceID() TraceID {
	var id TraceID
	_, _ = rand.Read(id[:])
	return id
}

func testSampler(name string, sampler Sampler, count int) {
	sampled := 0
	for i := 0; i < count; i++ {
		decision := sampler.ShouldSample(SamplingParams{
			TraceID: generateTraceID(),
			Name:    "test-operation",
		})
		if decision == RecordAndSample {
			sampled++
		}
	}
	fmt.Printf("%-40s: %d/%d sampled (%.1f%%)\n",
		name, sampled, count, float64(sampled)/float64(count)*100)
}

func main() {
	_ = context.Background() // used in real scenarios

	const trials = 10000

	fmt.Println("=== Sampling Strategy Comparison ===")
	fmt.Println()

	testSampler("AlwaysOn", &AlwaysOnSampler{}, trials)
	testSampler("AlwaysOff", &AlwaysOffSampler{}, trials)
	testSampler("Probability(10%)", NewProbabilitySampler(0.10), trials)
	testSampler("Probability(50%)", NewProbabilitySampler(0.50), trials)
	testSampler("Probability(1%)", NewProbabilitySampler(0.01), trials)
	testSampler("RateLimiting(100/s)", NewRateLimitingSampler(100), trials)

	// Parent-based with 10% root sampling
	parentBased := NewParentBasedSampler(NewProbabilitySampler(0.10))
	testSampler("ParentBased(root=10%)", parentBased, trials)

	// Test parent-based with a sampled parent
	sampledTrue := true
	sampledParent := 0
	for i := 0; i < 1000; i++ {
		d := parentBased.ShouldSample(SamplingParams{
			TraceID:       generateTraceID(),
			Name:          "child-op",
			ParentSampled: &sampledTrue,
		})
		if d == RecordAndSample {
			sampledParent++
		}
	}
	fmt.Printf("%-40s: %d/%d sampled (%.1f%%) [parent=sampled]\n",
		"ParentBased(child, parent=true)", sampledParent, 1000, float64(sampledParent)/10.0)

	// Name-based: drop health checks, sample everything else at 50%
	nameBased := NewNameBasedSampler(
		map[string]SamplingDecision{
			"GET /healthz": Drop,
			"GET /readyz":  Drop,
		},
		NewProbabilitySampler(0.50),
	)
	fmt.Println()
	fmt.Println("=== Name-Based Sampler ===")
	healthSampled := 0
	normalSampled := 0
	for i := 0; i < 1000; i++ {
		d := nameBased.ShouldSample(SamplingParams{TraceID: generateTraceID(), Name: "GET /healthz"})
		if d == RecordAndSample { healthSampled++ }
		d = nameBased.ShouldSample(SamplingParams{TraceID: generateTraceID(), Name: "GET /api/orders"})
		if d == RecordAndSample { normalSampled++ }
	}
	fmt.Printf("  Health checks: %d/1000 sampled\n", healthSampled)
	fmt.Printf("  Normal ops:    %d/1000 sampled\n", normalSampled)
}
```

### Tail-Based Sampling

Head-based sampling (all the samplers above) decides whether to sample at the beginning of a trace, before you know how it will turn out. Tail-based sampling waits until the trace is complete, then decides based on the outcome:

- Did the trace contain an error? Keep it.
- Was the total duration above a threshold? Keep it.
- Did it touch a specific service? Keep it.

Tail-based sampling requires a collector that buffers complete traces before making a decision. Here is a simplified in-process version:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	Attributes  map[string]any
	ServiceName string
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

type SpanExporter interface {
	ExportSpans(ctx context.Context, spans []*Span) error
	Shutdown(ctx context.Context) error
}

// TailSamplingPolicy decides whether to keep a complete trace.
type TailSamplingPolicy interface {
	ShouldKeep(spans []*Span) bool
	Name() string
}

// ErrorPolicy keeps traces that contain at least one error span.
type ErrorPolicy struct{}

func (p *ErrorPolicy) ShouldKeep(spans []*Span) bool {
	for _, s := range spans {
		if s.Status == StatusError {
			return true
		}
	}
	return false
}

func (p *ErrorPolicy) Name() string { return "error-policy" }

// LatencyPolicy keeps traces where the root span exceeds a threshold.
type LatencyPolicy struct {
	Threshold time.Duration
}

func (p *LatencyPolicy) ShouldKeep(spans []*Span) bool {
	for _, s := range spans {
		if s.Duration() > p.Threshold {
			return true
		}
	}
	return false
}

func (p *LatencyPolicy) Name() string {
	return fmt.Sprintf("latency-policy(>%s)", p.Threshold)
}

// AttributePolicy keeps traces where any span has a specific attribute value.
type AttributePolicy struct {
	Key   string
	Value any
}

func (p *AttributePolicy) ShouldKeep(spans []*Span) bool {
	for _, s := range spans {
		if v, ok := s.Attributes[p.Key]; ok && v == p.Value {
			return true
		}
	}
	return false
}

func (p *AttributePolicy) Name() string {
	return fmt.Sprintf("attribute-policy(%s=%v)", p.Key, p.Value)
}

// TailSamplingCollector buffers traces and applies tail sampling policies.
type TailSamplingCollector struct {
	mu       sync.Mutex
	traces   map[TraceID][]*Span
	policies []TailSamplingPolicy
	delegate SpanExporter
	timeout  time.Duration // How long to wait for a trace to complete
	stopCh   chan struct{}
	doneCh   chan struct{}
}

func NewTailSamplingCollector(
	delegate SpanExporter,
	timeout time.Duration,
	policies ...TailSamplingPolicy,
) *TailSamplingCollector {
	c := &TailSamplingCollector{
		traces:   make(map[TraceID][]*Span),
		policies: policies,
		delegate: delegate,
		timeout:  timeout,
		stopCh:   make(chan struct{}),
		doneCh:   make(chan struct{}),
	}
	go c.evictionLoop()
	return c
}

func (c *TailSamplingCollector) AddSpan(span *Span) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.traces[span.TraceID] = append(c.traces[span.TraceID], span)
}

// evictionLoop periodically checks for traces that are old enough to evaluate.
func (c *TailSamplingCollector) evictionLoop() {
	defer close(c.doneCh)
	ticker := time.NewTicker(c.timeout / 2)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			c.evaluateTraces()
		case <-c.stopCh:
			c.evaluateTraces()
			return
		}
	}
}

func (c *TailSamplingCollector) evaluateTraces() {
	c.mu.Lock()
	now := time.Now()
	toEvaluate := make(map[TraceID][]*Span)

	for traceID, spans := range c.traces {
		// Check if the trace is old enough to evaluate
		oldestEnd := now
		for _, s := range spans {
			if !s.EndTime.IsZero() && s.EndTime.Before(oldestEnd) {
				oldestEnd = s.EndTime
			}
		}
		if now.Sub(oldestEnd) >= c.timeout {
			toEvaluate[traceID] = spans
			delete(c.traces, traceID)
		}
	}
	c.mu.Unlock()

	// Evaluate outside the lock
	for traceID, spans := range toEvaluate {
		keep := false
		var matchedPolicy string
		for _, policy := range c.policies {
			if policy.ShouldKeep(spans) {
				keep = true
				matchedPolicy = policy.Name()
				break
			}
		}

		if keep {
			fmt.Printf("[tail-sample] KEEP trace %s (%d spans, matched: %s)\n",
				traceID, len(spans), matchedPolicy)
			_ = c.delegate.ExportSpans(context.Background(), spans)
		} else {
			fmt.Printf("[tail-sample] DROP trace %s (%d spans)\n", traceID, len(spans))
		}
	}
}

func (c *TailSamplingCollector) Shutdown() {
	close(c.stopCh)
	<-c.doneCh
}

// ---------- Printing Exporter ----------

type PrintExporter struct{}

func (e *PrintExporter) ExportSpans(_ context.Context, spans []*Span) error {
	for _, s := range spans {
		fmt.Printf("  [exported] %s (status=%d, duration=%s)\n", s.Name, s.Status, s.Duration())
	}
	return nil
}

func (e *PrintExporter) Shutdown(_ context.Context) error { return nil }

// ---------- Demo ----------

func main() {
	collector := NewTailSamplingCollector(
		&PrintExporter{},
		100*time.Millisecond,
		&ErrorPolicy{},
		&LatencyPolicy{Threshold: 50 * time.Millisecond},
	)

	now := time.Now().Add(-200 * time.Millisecond) // In the past so they evaluate immediately

	// Trace 1: Normal, fast -- should be DROPPED
	var t1 TraceID
	_, _ = rand.Read(t1[:])
	collector.AddSpan(&Span{
		TraceID: t1, Name: "fast-ok", StartTime: now, EndTime: now.Add(10 * time.Millisecond),
		Status: StatusOK, ServiceName: "svc-a",
	})

	// Trace 2: Contains an error -- should be KEPT (error policy)
	var t2 TraceID
	_, _ = rand.Read(t2[:])
	collector.AddSpan(&Span{
		TraceID: t2, Name: "has-error", StartTime: now, EndTime: now.Add(20 * time.Millisecond),
		Status: StatusError, ServiceName: "svc-b",
	})

	// Trace 3: Slow -- should be KEPT (latency policy)
	var t3 TraceID
	_, _ = rand.Read(t3[:])
	collector.AddSpan(&Span{
		TraceID: t3, Name: "slow-op", StartTime: now, EndTime: now.Add(100 * time.Millisecond),
		Status: StatusOK, ServiceName: "svc-c",
	})

	fmt.Println("=== Tail Sampling Demo ===")
	time.Sleep(300 * time.Millisecond)
	collector.Shutdown()
}
```

---

## 7. Trace Context Propagation Across Services

### The Propagation Problem

When Service A calls Service B over HTTP, the trace context must travel across the network boundary. Without propagation, Service B starts a brand-new trace, and the connection between A and B is lost forever.

Propagation has two sides:

- **Injection**: Service A injects trace context into outgoing HTTP headers
- **Extraction**: Service B extracts trace context from incoming HTTP headers

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"strings"
)

// ---------- Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }
func (t TraceID) IsZero() bool {
	for _, b := range t { if b != 0 { return false } }
	return true
}

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type TraceFlags byte
const FlagsSampled TraceFlags = 0x01
func (f TraceFlags) IsSampled() bool { return f&FlagsSampled != 0 }

// SpanContext carries the immutable trace identification data.
type SpanContext struct {
	TraceID    TraceID
	SpanID     SpanID
	TraceFlags TraceFlags
	TraceState string
}

func (sc SpanContext) IsValid() bool {
	return !sc.TraceID.IsZero() && !sc.SpanID.IsZero()
}

// ---------- Propagator Interface ----------

// Propagator defines how trace context is injected into and extracted from carriers.
type Propagator interface {
	// Inject sets trace context into the carrier (e.g., HTTP headers).
	Inject(ctx context.Context, carrier Carrier)
	// Extract reads trace context from the carrier and returns an enriched context.
	Extract(ctx context.Context, carrier Carrier) context.Context
	// Fields returns the header names this propagator uses.
	Fields() []string
}

// Carrier abstracts the transport mechanism (HTTP headers, gRPC metadata, etc.)
type Carrier interface {
	Get(key string) string
	Set(key, value string)
	Keys() []string
}

// ---------- HTTP Header Carrier ----------

type HTTPHeaderCarrier http.Header

func (h HTTPHeaderCarrier) Get(key string) string {
	return http.Header(h).Get(key)
}

func (h HTTPHeaderCarrier) Set(key, value string) {
	http.Header(h).Set(key, value)
}

func (h HTTPHeaderCarrier) Keys() []string {
	keys := make([]string, 0, len(h))
	for k := range h {
		keys = append(keys, k)
	}
	return keys
}

// ---------- Map Carrier (for message queues) ----------

type MapCarrier map[string]string

func (m MapCarrier) Get(key string) string { return m[key] }
func (m MapCarrier) Set(key, value string) { m[key] = value }
func (m MapCarrier) Keys() []string {
	keys := make([]string, 0, len(m))
	for k := range m { keys = append(keys, k) }
	return keys
}

// ---------- W3C TraceContext Propagator ----------

const (
	traceparentHeader = "traceparent"
	tracestateHeader  = "tracestate"
)

type W3CTraceContextPropagator struct{}

func (p *W3CTraceContextPropagator) Fields() []string {
	return []string{traceparentHeader, tracestateHeader}
}

func (p *W3CTraceContextPropagator) Inject(ctx context.Context, carrier Carrier) {
	sc := spanContextFromContext(ctx)
	if !sc.IsValid() {
		return
	}

	// Format: 00-{trace-id}-{span-id}-{flags}
	traceparent := fmt.Sprintf("00-%s-%s-%02x",
		sc.TraceID, sc.SpanID, byte(sc.TraceFlags))
	carrier.Set(traceparentHeader, traceparent)

	if sc.TraceState != "" {
		carrier.Set(tracestateHeader, sc.TraceState)
	}
}

func (p *W3CTraceContextPropagator) Extract(ctx context.Context, carrier Carrier) context.Context {
	traceparent := carrier.Get(traceparentHeader)
	if traceparent == "" {
		return ctx
	}

	sc, err := parseTraceparent(traceparent)
	if err != nil {
		return ctx // Invalid header, ignore
	}

	sc.TraceState = carrier.Get(tracestateHeader)

	return contextWithSpanContext(ctx, sc)
}

func parseTraceparent(header string) (SpanContext, error) {
	parts := strings.Split(strings.TrimSpace(header), "-")
	if len(parts) < 4 {
		return SpanContext{}, fmt.Errorf("invalid traceparent: expected 4 parts")
	}

	if parts[0] == "ff" {
		return SpanContext{}, fmt.Errorf("invalid version ff")
	}

	traceIDBytes, err := hex.DecodeString(parts[1])
	if err != nil || len(traceIDBytes) != 16 {
		return SpanContext{}, fmt.Errorf("invalid trace-id")
	}

	spanIDBytes, err := hex.DecodeString(parts[2])
	if err != nil || len(spanIDBytes) != 8 {
		return SpanContext{}, fmt.Errorf("invalid span-id")
	}

	flagsBytes, err := hex.DecodeString(parts[3])
	if err != nil || len(flagsBytes) != 1 {
		return SpanContext{}, fmt.Errorf("invalid flags")
	}

	var sc SpanContext
	copy(sc.TraceID[:], traceIDBytes)
	copy(sc.SpanID[:], spanIDBytes)
	sc.TraceFlags = TraceFlags(flagsBytes[0])

	if sc.TraceID.IsZero() || sc.SpanID.IsZero() {
		return SpanContext{}, fmt.Errorf("zero trace-id or span-id")
	}

	return sc, nil
}

// ---------- Context Helpers ----------

type spanContextKeyType struct{}
var spanContextKey = spanContextKeyType{}

func spanContextFromContext(ctx context.Context) SpanContext {
	sc, _ := ctx.Value(spanContextKey).(SpanContext)
	return sc
}

func contextWithSpanContext(ctx context.Context, sc SpanContext) context.Context {
	return context.WithValue(ctx, spanContextKey, sc)
}

// ---------- Demo ----------

func main() {
	propagator := &W3CTraceContextPropagator{}

	// Simulate Service A: create a span context and inject into headers
	var traceID TraceID
	var spanID SpanID
	_, _ = rand.Read(traceID[:])
	_, _ = rand.Read(spanID[:])

	serviceACtx := contextWithSpanContext(context.Background(), SpanContext{
		TraceID:    traceID,
		SpanID:     spanID,
		TraceFlags: FlagsSampled,
		TraceState: "myvendor=abc123",
	})

	// Inject into HTTP headers (what Service A does before calling Service B)
	headers := make(http.Header)
	propagator.Inject(serviceACtx, HTTPHeaderCarrier(headers))

	fmt.Println("=== Service A: Injected Headers ===")
	fmt.Printf("traceparent: %s\n", headers.Get("traceparent"))
	fmt.Printf("tracestate:  %s\n", headers.Get("tracestate"))
	fmt.Println()

	// Simulate Service B: extract from headers
	serviceBCtx := propagator.Extract(context.Background(), HTTPHeaderCarrier(headers))
	extractedSC := spanContextFromContext(serviceBCtx)

	fmt.Println("=== Service B: Extracted Context ===")
	fmt.Printf("Trace ID:    %s\n", extractedSC.TraceID)
	fmt.Printf("Span ID:     %s\n", extractedSC.SpanID)
	fmt.Printf("Sampled:     %v\n", extractedSC.TraceFlags.IsSampled())
	fmt.Printf("TraceState:  %s\n", extractedSC.TraceState)
	fmt.Printf("Valid:       %v\n", extractedSC.IsValid())
	fmt.Printf("IDs match:   %v\n", extractedSC.TraceID == traceID && extractedSC.SpanID == spanID)
	fmt.Println()

	// Simulate message queue propagation
	mqHeaders := MapCarrier{}
	propagator.Inject(serviceACtx, mqHeaders)

	fmt.Println("=== Message Queue: Injected Headers ===")
	for k, v := range mqHeaders {
		fmt.Printf("  %s: %s\n", k, v)
	}

	mqCtx := propagator.Extract(context.Background(), mqHeaders)
	mqSC := spanContextFromContext(mqCtx)
	fmt.Printf("Extracted from MQ: trace=%s span=%s\n", mqSC.TraceID, mqSC.SpanID)
}
```

---

## 8. Instrumenting Middleware

### Automatic HTTP Server and Client Instrumentation

In a real application, you do not want to manually start and end spans in every handler. Middleware does it automatically -- for every incoming request and every outgoing HTTP call.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"sync"
	"time"
)

// ---------- Core Types (abbreviated for focus on middleware) ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }
func (t TraceID) IsZero() bool {
	for _, b := range t { if b != 0 { return false } }
	return true
}

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int
const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	ServiceName string
	mu          sync.Mutex
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil {
		s.Attributes = make(map[string]any)
	}
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) End() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.EndTime = time.Now()
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Context helpers ----------

type contextKey struct{ name string }
var spanCtxKey = &contextKey{"span"}

func spanFromContext(ctx context.Context) *Span {
	s, _ := ctx.Value(spanCtxKey).(*Span)
	return s
}

func contextWithSpan(ctx context.Context, span *Span) context.Context {
	return context.WithValue(ctx, spanCtxKey, span)
}

// ---------- Simple Tracer ----------

type Tracer struct {
	serviceName string
	mu          sync.Mutex
	spans       []*Span // collect for demonstration
}

func NewTracer(service string) *Tracer {
	return &Tracer{serviceName: service}
}

func (t *Tracer) StartSpan(ctx context.Context, name string, kind SpanKind) (context.Context, *Span) {
	parent := spanFromContext(ctx)

	var traceID TraceID
	var parentID SpanID

	if parent != nil {
		traceID = parent.TraceID
		parentID = parent.SpanID
	} else {
		_, _ = rand.Read(traceID[:])
	}

	var spanID SpanID
	_, _ = rand.Read(spanID[:])

	span := &Span{
		TraceID:     traceID,
		SpanID:      spanID,
		ParentID:    parentID,
		Name:        name,
		Kind:        kind,
		StartTime:   time.Now(),
		ServiceName: t.serviceName,
		Attributes:  make(map[string]any),
	}

	t.mu.Lock()
	t.spans = append(t.spans, span)
	t.mu.Unlock()

	return contextWithSpan(ctx, span), span
}

func (t *Tracer) Spans() []*Span {
	t.mu.Lock()
	defer t.mu.Unlock()
	return append([]*Span{}, t.spans...)
}

// ---------- Response Writer Wrapper ----------

// responseWriter wraps http.ResponseWriter to capture the status code.
type responseWriter struct {
	http.ResponseWriter
	statusCode    int
	bytesWritten  int64
	headerWritten bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
}

func (rw *responseWriter) WriteHeader(code int) {
	if !rw.headerWritten {
		rw.statusCode = code
		rw.headerWritten = true
	}
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += int64(n)
	return n, err
}

// ---------- Trace Middleware (Server Side) ----------

// TraceMiddleware automatically creates a server span for every incoming request.
// It extracts trace context from the incoming traceparent header and creates
// a child span, or starts a new trace if no header is present.
func TraceMiddleware(tracer *Tracer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx := r.Context()

			// Extract trace context from incoming headers
			traceparent := r.Header.Get("traceparent")
			if traceparent != "" {
				sc, err := parseTraceparentSimple(traceparent)
				if err == nil {
					// Create a "remote parent" span in context
					remoteSpan := &Span{
						TraceID: sc.traceID,
						SpanID:  sc.spanID,
					}
					ctx = contextWithSpan(ctx, remoteSpan)
				}
			}

			// Generate span name from HTTP method and path
			spanName := fmt.Sprintf("%s %s", r.Method, r.URL.Path)

			// Start a server span
			ctx, span := tracer.StartSpan(ctx, spanName, SpanKindServer)

			// Record HTTP attributes (following OpenTelemetry semantic conventions)
			span.SetAttribute("http.method", r.Method)
			span.SetAttribute("http.url", r.URL.String())
			span.SetAttribute("http.target", r.URL.Path)
			span.SetAttribute("http.host", r.Host)
			span.SetAttribute("http.scheme", "http")
			span.SetAttribute("http.user_agent", r.UserAgent())
			span.SetAttribute("net.peer.ip", r.RemoteAddr)

			if r.ContentLength > 0 {
				span.SetAttribute("http.request_content_length", r.ContentLength)
			}

			// Inject trace context into response headers
			// (so the caller can correlate its client span with our server span)
			w.Header().Set("traceparent", fmt.Sprintf("00-%s-%s-01",
				span.TraceID, span.SpanID))

			// Wrap response writer to capture status code
			wrapped := newResponseWriter(w)

			// Call the actual handler
			next.ServeHTTP(wrapped, r.WithContext(ctx))

			// Record response attributes
			span.SetAttribute("http.status_code", wrapped.statusCode)
			span.SetAttribute("http.response_content_length", wrapped.bytesWritten)

			// Set span status based on HTTP status code
			if wrapped.statusCode >= 400 {
				span.SetStatus(StatusError, fmt.Sprintf("HTTP %d", wrapped.statusCode))
			} else {
				span.SetStatus(StatusOK, "")
			}

			span.End()
		})
	}
}

// ---------- Tracing HTTP Client Transport ----------

// TracingTransport wraps an http.RoundTripper to automatically
// create client spans for every outgoing HTTP request.
type TracingTransport struct {
	base   http.RoundTripper
	tracer *Tracer
}

func NewTracingTransport(tracer *Tracer, base http.RoundTripper) *TracingTransport {
	if base == nil {
		base = http.DefaultTransport
	}
	return &TracingTransport{base: base, tracer: tracer}
}

func (t *TracingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	ctx := req.Context()

	// Create a client span
	spanName := fmt.Sprintf("HTTP %s %s", req.Method, req.URL.Path)
	ctx, span := t.tracer.StartSpan(ctx, spanName, SpanKindClient)

	// Record outgoing request attributes
	span.SetAttribute("http.method", req.Method)
	span.SetAttribute("http.url", req.URL.String())
	span.SetAttribute("net.peer.name", req.URL.Host)

	// Inject trace context into outgoing headers
	req = req.Clone(ctx) // Clone to avoid mutating the original
	req.Header.Set("traceparent", fmt.Sprintf("00-%s-%s-01",
		span.TraceID, span.SpanID))

	// Make the actual request
	resp, err := t.base.RoundTrip(req)

	if err != nil {
		span.SetStatus(StatusError, err.Error())
		span.End()
		return nil, err
	}

	// Record response attributes
	span.SetAttribute("http.status_code", resp.StatusCode)

	if resp.StatusCode >= 400 {
		span.SetStatus(StatusError, fmt.Sprintf("HTTP %d", resp.StatusCode))
	} else {
		span.SetStatus(StatusOK, "")
	}

	span.End()
	return resp, nil
}

// ---------- Simple traceparent parser ----------

type simpleTraceContext struct {
	traceID TraceID
	spanID  SpanID
	sampled bool
}

func parseTraceparentSimple(header string) (simpleTraceContext, error) {
	parts := strings.Split(header, "-")
	if len(parts) < 4 {
		return simpleTraceContext{}, fmt.Errorf("invalid")
	}
	traceBytes, err := hex.DecodeString(parts[1])
	if err != nil || len(traceBytes) != 16 {
		return simpleTraceContext{}, fmt.Errorf("invalid trace-id")
	}
	spanBytes, err := hex.DecodeString(parts[2])
	if err != nil || len(spanBytes) != 8 {
		return simpleTraceContext{}, fmt.Errorf("invalid span-id")
	}
	var tc simpleTraceContext
	copy(tc.traceID[:], traceBytes)
	copy(tc.spanID[:], spanBytes)
	tc.sampled = parts[3] == "01"
	return tc, nil
}

// ---------- Demo ----------

func main() {
	tracer := NewTracer("my-api")

	// Create a test server with trace middleware
	mux := http.NewServeMux()

	mux.HandleFunc("/api/orders", func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		span := spanFromContext(ctx)
		if span != nil {
			span.SetAttribute("handler", "orders")
		}

		// Simulate some work
		time.Sleep(5 * time.Millisecond)

		// Simulate a database call within the handler
		_, dbSpan := tracer.StartSpan(ctx, "db.Query SELECT orders", SpanKindClient)
		dbSpan.SetAttribute("db.system", "postgresql")
		time.Sleep(3 * time.Millisecond)
		dbSpan.SetStatus(StatusOK, "")
		dbSpan.End()

		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"orders": [{"id": 1, "item": "widget"}]}`))
	})

	mux.HandleFunc("/api/error", func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(`{"error": "something went wrong"}`))
	})

	// Wrap with trace middleware
	handler := TraceMiddleware(tracer)(mux)

	// Start test server
	server := httptest.NewServer(handler)
	defer server.Close()

	// Create a tracing HTTP client
	client := &http.Client{
		Transport: NewTracingTransport(tracer, nil),
	}

	// Make requests
	fmt.Println("=== Making Requests ===")
	fmt.Println()

	// Request 1: successful
	resp, err := client.Get(server.URL + "/api/orders")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	body, _ := io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Printf("Response 1: %d - %s\n", resp.StatusCode, string(body))

	// Request 2: error
	resp, err = client.Get(server.URL + "/api/error")
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	body, _ = io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Printf("Response 2: %d - %s\n", resp.StatusCode, string(body))

	// Print all collected spans
	fmt.Println()
	fmt.Println("=== Collected Spans ===")
	for _, s := range tracer.Spans() {
		kind := "INTERNAL"
		if s.Kind == SpanKindServer {
			kind = "SERVER"
		} else if s.Kind == SpanKindClient {
			kind = "CLIENT"
		}
		parent := "ROOT"
		if !s.ParentID.IsZero() {
			parent = s.ParentID.String()[:8] + "..."
		}
		status := "UNSET"
		if s.Status == StatusOK {
			status = "OK"
		} else if s.Status == StatusError {
			status = "ERROR"
		}
		fmt.Printf("[%s] %s (parent=%s, status=%s, duration=%s)\n",
			kind, s.Name, parent, status, s.Duration())
	}
}
```

---

## 9. Database Tracing

### Why Trace Database Calls?

Database queries are often the biggest source of latency in web applications. Tracing database calls tells you:

- Which queries are slow
- How many queries each request makes (N+1 query detection)
- Whether connection pool contention is causing delays
- Which transactions are holding locks

```go
package main

import (
	"context"
	"crypto/rand"
	"database/sql"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// ---------- Trace Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int
const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	ServiceName string
	mu          sync.Mutex
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil { s.Attributes = make(map[string]any) }
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) End() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.EndTime = time.Now()
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Tracer ----------

type contextKey struct{}
var spanKey = contextKey{}

func spanFromContext(ctx context.Context) *Span {
	s, _ := ctx.Value(spanKey).(*Span)
	return s
}

type Tracer struct {
	service string
	mu      sync.Mutex
	spans   []*Span
}

func NewTracer(service string) *Tracer {
	return &Tracer{service: service}
}

func (t *Tracer) StartSpan(ctx context.Context, name string, kind SpanKind) (context.Context, *Span) {
	parent := spanFromContext(ctx)
	var traceID TraceID
	var parentID SpanID
	if parent != nil {
		traceID = parent.TraceID
		parentID = parent.SpanID
	} else {
		_, _ = rand.Read(traceID[:])
	}
	var spanID SpanID
	_, _ = rand.Read(spanID[:])
	span := &Span{
		TraceID: traceID, SpanID: spanID, ParentID: parentID,
		Name: name, Kind: kind, StartTime: time.Now(),
		ServiceName: t.service, Attributes: make(map[string]any),
	}
	t.mu.Lock()
	t.spans = append(t.spans, span)
	t.mu.Unlock()
	return context.WithValue(ctx, spanKey, span), span
}

// ---------- Traced Database Wrapper ----------

// TracedDB wraps database operations with tracing.
// It follows OpenTelemetry semantic conventions for database spans.
type TracedDB struct {
	tracer   *Tracer
	dbSystem string // "postgresql", "mysql", "sqlite", etc.
	dbName   string
	dbHost   string
	dbPort   int

	// Simulated operations (in real code, this wraps *sql.DB)
}

// NewTracedDB creates a new traced database wrapper.
func NewTracedDB(tracer *Tracer, system, name, host string, port int) *TracedDB {
	return &TracedDB{
		tracer:   tracer,
		dbSystem: system,
		dbName:   name,
		dbHost:   host,
		dbPort:   port,
	}
}

// setDBAttributes adds standard database span attributes.
func (db *TracedDB) setDBAttributes(span *Span, operation, statement string) {
	span.SetAttribute("db.system", db.dbSystem)
	span.SetAttribute("db.name", db.dbName)
	span.SetAttribute("db.operation", operation)
	span.SetAttribute("db.statement", statement)
	span.SetAttribute("net.peer.name", db.dbHost)
	span.SetAttribute("net.peer.port", db.dbPort)
}

// QueryContext executes a query and traces it.
func (db *TracedDB) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
	spanName := fmt.Sprintf("db.Query %s", truncateQuery(query, 50))
	ctx, span := db.tracer.StartSpan(ctx, spanName, SpanKindClient)
	defer span.End()

	db.setDBAttributes(span, extractOperation(query), query)
	span.SetAttribute("db.args_count", len(args))

	// Simulate the query
	time.Sleep(5 * time.Millisecond)

	// In real code:
	// rows, err := db.inner.QueryContext(ctx, query, args...)
	// if err != nil {
	//     span.SetStatus(StatusError, err.Error())
	//     return nil, err
	// }
	// span.SetStatus(StatusOK, "")
	// return rows, nil

	span.SetAttribute("db.rows_affected", 3) // simulated
	span.SetStatus(StatusOK, "")

	_ = ctx // would be passed to inner DB

	return nil, nil
}

// ExecContext executes a statement (INSERT, UPDATE, DELETE) and traces it.
func (db *TracedDB) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
	spanName := fmt.Sprintf("db.Exec %s", truncateQuery(query, 50))
	ctx, span := db.tracer.StartSpan(ctx, spanName, SpanKindClient)
	defer span.End()

	db.setDBAttributes(span, extractOperation(query), query)
	span.SetAttribute("db.args_count", len(args))

	// Simulate the execution
	time.Sleep(3 * time.Millisecond)

	span.SetAttribute("db.rows_affected", 1)
	span.SetStatus(StatusOK, "")

	_ = ctx
	return nil, nil
}

// BeginTxContext starts a transaction and traces it.
func (db *TracedDB) BeginTxContext(ctx context.Context, opts *sql.TxOptions) (*TracedTx, error) {
	ctx, span := db.tracer.StartSpan(ctx, "db.BeginTransaction", SpanKindClient)
	db.setDBAttributes(span, "BEGIN", "BEGIN TRANSACTION")

	if opts != nil {
		span.SetAttribute("db.tx_isolation", fmt.Sprintf("%d", opts.Isolation))
		span.SetAttribute("db.tx_read_only", opts.ReadOnly)
	}

	return &TracedTx{
		db:       db,
		ctx:      ctx,
		beginSpan: span,
	}, nil
}

// ---------- Traced Transaction ----------

type TracedTx struct {
	db        *TracedDB
	ctx       context.Context
	beginSpan *Span
}

func (tx *TracedTx) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
	// Use the transaction's context to maintain span hierarchy
	return tx.db.QueryContext(tx.ctx, query, args...)
}

func (tx *TracedTx) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
	return tx.db.ExecContext(tx.ctx, query, args...)
}

func (tx *TracedTx) Commit() error {
	_, span := tx.db.tracer.StartSpan(tx.ctx, "db.Commit", SpanKindClient)
	tx.db.setDBAttributes(span, "COMMIT", "COMMIT")
	time.Sleep(1 * time.Millisecond)
	span.SetStatus(StatusOK, "")
	span.End()

	tx.beginSpan.SetStatus(StatusOK, "committed")
	tx.beginSpan.End()
	return nil
}

func (tx *TracedTx) Rollback() error {
	_, span := tx.db.tracer.StartSpan(tx.ctx, "db.Rollback", SpanKindClient)
	tx.db.setDBAttributes(span, "ROLLBACK", "ROLLBACK")
	time.Sleep(1 * time.Millisecond)
	span.SetStatus(StatusOK, "")
	span.End()

	tx.beginSpan.SetStatus(StatusError, "rolled back")
	tx.beginSpan.End()
	return nil
}

// ---------- Helpers ----------

func truncateQuery(query string, maxLen int) string {
	if len(query) <= maxLen {
		return query
	}
	return query[:maxLen] + "..."
}

func extractOperation(query string) string {
	// Extract the first word (SELECT, INSERT, UPDATE, DELETE, etc.)
	for i, ch := range query {
		if ch == ' ' || ch == '\t' || ch == '\n' {
			return query[:i]
		}
	}
	return query
}

// ---------- Demo ----------

func main() {
	tracer := NewTracer("order-service")
	db := NewTracedDB(tracer, "postgresql", "orders_db", "db.internal", 5432)

	// Simulate a request handler
	ctx := context.Background()
	ctx, rootSpan := tracer.StartSpan(ctx, "GET /api/orders/123", SpanKindServer)

	// Simple query
	fmt.Println("=== Database Tracing Demo ===")
	fmt.Println()

	_, _ = db.QueryContext(ctx, "SELECT * FROM orders WHERE id = $1", 123)

	// Transaction
	tx, _ := db.BeginTxContext(ctx, nil)
	_, _ = tx.ExecContext(ctx, "UPDATE orders SET status = $1 WHERE id = $2", "shipped", 123)
	_, _ = tx.ExecContext(ctx, "INSERT INTO order_events (order_id, event) VALUES ($1, $2)", 123, "shipped")
	_ = tx.Commit()

	rootSpan.SetStatus(StatusOK, "")
	rootSpan.End()

	// Print all spans
	fmt.Println("=== Spans ===")
	for _, s := range tracer.spans {
		parent := "ROOT"
		if !s.ParentID.IsZero() {
			parent = s.ParentID.String()[:8] + "..."
		}
		kind := "INT"
		if s.Kind == SpanKindServer { kind = "SVR" }
		if s.Kind == SpanKindClient { kind = "CLI" }
		fmt.Printf("[%s] %-50s parent=%s duration=%s\n",
			kind, s.Name, parent, s.Duration())
		if stmt, ok := s.Attributes["db.statement"]; ok {
			fmt.Printf("       db.statement=%s\n", stmt)
		}
	}
}
```

---

## 10. Async Operations and Goroutine Tracing

### The Goroutine Propagation Problem

In Go, work frequently moves between goroutines. The critical rule is: **always pass context when spawning goroutines**. If you do not, the child goroutine loses its trace context and starts a disconnected trace.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// ---------- Core Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int
const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	Attributes  map[string]any
	ServiceName string
	mu          sync.Mutex
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil { s.Attributes = make(map[string]any) }
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
}

func (s *Span) End() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.EndTime = time.Now()
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Tracer ----------

type contextKey struct{}
var spanKey = contextKey{}

func spanFromContext(ctx context.Context) *Span {
	s, _ := ctx.Value(spanKey).(*Span)
	return s
}

type Tracer struct {
	service string
	mu      sync.Mutex
	spans   []*Span
}

func NewTracer(service string) *Tracer { return &Tracer{service: service} }

func (t *Tracer) StartSpan(ctx context.Context, name string, kind SpanKind) (context.Context, *Span) {
	parent := spanFromContext(ctx)
	var traceID TraceID
	var parentID SpanID
	if parent != nil {
		traceID = parent.TraceID
		parentID = parent.SpanID
	} else {
		_, _ = rand.Read(traceID[:])
	}
	var spanID SpanID
	_, _ = rand.Read(spanID[:])
	span := &Span{
		TraceID: traceID, SpanID: spanID, ParentID: parentID,
		Name: name, Kind: kind, StartTime: time.Now(),
		ServiceName: t.service, Attributes: make(map[string]any),
	}
	t.mu.Lock()
	t.spans = append(t.spans, span)
	t.mu.Unlock()
	return context.WithValue(ctx, spanKey, span), span
}

// ---------- Pattern 1: Fan-Out with WaitGroup ----------

func processBatch(ctx context.Context, tracer *Tracer, items []string) {
	ctx, span := tracer.StartSpan(ctx, "process-batch", SpanKindInternal)
	defer span.End()
	span.SetAttribute("batch.size", len(items))

	var wg sync.WaitGroup

	for _, item := range items {
		wg.Add(1)
		go func(item string) {
			defer wg.Done()
			// Pass ctx to maintain trace parent-child relationship
			processItem(ctx, tracer, item)
		}(item)
	}

	wg.Wait()
	span.SetStatus(StatusOK, "")
}

func processItem(ctx context.Context, tracer *Tracer, item string) {
	// This span is a child of "process-batch" because ctx carries
	// the parent span
	ctx, span := tracer.StartSpan(ctx, "process-item", SpanKindInternal)
	defer span.End()
	span.SetAttribute("item.name", item)

	// Simulate variable-duration work
	time.Sleep(time.Duration(len(item)) * time.Millisecond)
	span.SetStatus(StatusOK, "")

	_ = ctx
}

// ---------- Pattern 2: Fan-Out / Fan-In with Results ----------

type Result struct {
	Item  string
	Value int
	Err   error
}

func fetchAll(ctx context.Context, tracer *Tracer, urls []string) []Result {
	ctx, span := tracer.StartSpan(ctx, "fetch-all", SpanKindInternal)
	defer span.End()
	span.SetAttribute("urls.count", len(urls))

	results := make([]Result, len(urls))
	var wg sync.WaitGroup

	for i, url := range urls {
		wg.Add(1)
		go func(i int, url string) {
			defer wg.Done()
			childCtx, childSpan := tracer.StartSpan(ctx, "fetch-url", SpanKindClient)
			defer childSpan.End()
			childSpan.SetAttribute("http.url", url)

			// Simulate HTTP fetch
			time.Sleep(10 * time.Millisecond)

			results[i] = Result{Item: url, Value: len(url)}
			childSpan.SetStatus(StatusOK, "")

			_ = childCtx
		}(i, url)
	}

	wg.Wait()
	span.SetStatus(StatusOK, "")
	return results
}

// ---------- Pattern 3: Worker Pool with Traced Jobs ----------

type Job struct {
	ID   int
	Data string
}

func workerPool(ctx context.Context, tracer *Tracer, jobs []Job, workers int) {
	ctx, span := tracer.StartSpan(ctx, "worker-pool", SpanKindInternal)
	defer span.End()
	span.SetAttribute("pool.workers", workers)
	span.SetAttribute("pool.jobs", len(jobs))

	jobCh := make(chan Job, len(jobs))
	var wg sync.WaitGroup

	// Start workers
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for job := range jobCh {
				// Each job gets its own span, parented to the pool span
				_, jobSpan := tracer.StartSpan(ctx, fmt.Sprintf("job-%d", job.ID), SpanKindInternal)
				jobSpan.SetAttribute("worker.id", workerID)
				jobSpan.SetAttribute("job.id", job.ID)
				jobSpan.SetAttribute("job.data", job.Data)

				// Simulate work
				time.Sleep(5 * time.Millisecond)

				jobSpan.SetStatus(StatusOK, "")
				jobSpan.End()
			}
		}(w)
	}

	// Submit jobs
	for _, job := range jobs {
		jobCh <- job
	}
	close(jobCh)

	wg.Wait()
	span.SetStatus(StatusOK, "")
}

// ---------- Pattern 4: Pipeline with Context ----------

func pipeline(ctx context.Context, tracer *Tracer, data []int) []int {
	ctx, span := tracer.StartSpan(ctx, "pipeline", SpanKindInternal)
	defer span.End()
	span.SetAttribute("input.size", len(data))

	// Stage 1: Filter
	filtered := pipelineStage(ctx, tracer, "filter", data, func(x int) (int, bool) {
		return x, x%2 == 0
	})

	// Stage 2: Transform
	transformed := pipelineStage(ctx, tracer, "transform", filtered, func(x int) (int, bool) {
		return x * x, true
	})

	span.SetAttribute("output.size", len(transformed))
	span.SetStatus(StatusOK, "")
	return transformed
}

func pipelineStage(ctx context.Context, tracer *Tracer, name string, data []int, fn func(int) (int, bool)) []int {
	_, span := tracer.StartSpan(ctx, "pipeline-stage-"+name, SpanKindInternal)
	defer span.End()
	span.SetAttribute("stage.name", name)
	span.SetAttribute("stage.input_size", len(data))

	var result []int
	for _, x := range data {
		if y, ok := fn(x); ok {
			result = append(result, y)
		}
	}

	span.SetAttribute("stage.output_size", len(result))
	span.SetStatus(StatusOK, "")
	return result
}

// ---------- Anti-Pattern: Lost Context ----------

func antiPatternLostContext(tracer *Tracer) {
	ctx := context.Background()
	ctx, parentSpan := tracer.StartSpan(ctx, "parent-operation", SpanKindServer)
	defer parentSpan.End()

	// WRONG: Using context.Background() loses the trace
	go func() {
		wrongCtx := context.Background() // This breaks the trace!
		_, childSpan := tracer.StartSpan(wrongCtx, "LOST-child", SpanKindInternal)
		defer childSpan.End()
		time.Sleep(2 * time.Millisecond)
		childSpan.SetStatus(StatusOK, "")
	}()

	// RIGHT: Pass the parent context to maintain the trace
	go func() {
		_, childSpan := tracer.StartSpan(ctx, "CONNECTED-child", SpanKindInternal)
		defer childSpan.End()
		time.Sleep(2 * time.Millisecond)
		childSpan.SetStatus(StatusOK, "")
	}()

	time.Sleep(10 * time.Millisecond)
	parentSpan.SetStatus(StatusOK, "")
}

// ---------- Demo ----------

func main() {
	tracer := NewTracer("async-demo")

	fmt.Println("=== Pattern 1: Fan-Out ===")
	ctx := context.Background()
	processBatch(ctx, tracer, []string{"alpha", "beta", "gamma", "delta"})

	fmt.Println("=== Pattern 2: Fan-Out/Fan-In ===")
	results := fetchAll(ctx, tracer, []string{
		"https://api.example.com/users",
		"https://api.example.com/orders",
		"https://api.example.com/products",
	})
	for _, r := range results {
		fmt.Printf("  %s -> %d\n", r.Item, r.Value)
	}

	fmt.Println("\n=== Pattern 3: Worker Pool ===")
	workerPool(ctx, tracer, []Job{
		{ID: 1, Data: "task-a"},
		{ID: 2, Data: "task-b"},
		{ID: 3, Data: "task-c"},
		{ID: 4, Data: "task-d"},
	}, 2)

	fmt.Println("\n=== Pattern 4: Pipeline ===")
	result := pipeline(ctx, tracer, []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
	fmt.Printf("  Pipeline result: %v\n", result)

	fmt.Println("\n=== Anti-Pattern: Lost Context ===")
	antiPatternLostContext(tracer)

	// Print span summary
	fmt.Println("\n=== All Spans ===")
	for _, s := range tracer.spans {
		parent := "ROOT"
		if !s.ParentID.IsZero() {
			parent = s.ParentID.String()[:8] + "..."
		}
		fmt.Printf("  trace=%s..  %-30s parent=%s duration=%s\n",
			s.TraceID.String()[:8], s.Name, parent, s.Duration())
	}
}
```

---

## 11. Error Recording and Exception Tracking

### How Errors Should Be Recorded in Spans

An error in a span is not just a status code. Rich error recording includes the error type, message, stack trace, and any relevant context. OpenTelemetry defines semantic conventions for recording exceptions.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"runtime"
	"strings"
	"sync"
	"time"
)

// ---------- Core Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanEvent struct {
	Name       string
	Timestamp  time.Time
	Attributes map[string]any
}

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	Events      []SpanEvent
	ServiceName string
	mu          sync.Mutex
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil { s.Attributes = make(map[string]any) }
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) AddEvent(name string, attrs map[string]any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Events = append(s.Events, SpanEvent{
		Name: name, Timestamp: time.Now(), Attributes: attrs,
	})
}

func (s *Span) End() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.EndTime = time.Now()
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Error Recording Methods ----------

// RecordError records an error as a span event with exception details.
// This follows OpenTelemetry semantic conventions for exception recording.
func (s *Span) RecordError(err error, opts ...ErrorOption) {
	if err == nil {
		return
	}

	config := errorConfig{}
	for _, opt := range opts {
		opt(&config)
	}

	attrs := map[string]any{
		"exception.type":    fmt.Sprintf("%T", err),
		"exception.message": err.Error(),
	}

	// Capture stack trace if requested
	if config.withStackTrace {
		attrs["exception.stacktrace"] = captureStackTrace(3) // skip 3 frames
	}

	// Add any additional attributes
	for k, v := range config.extraAttrs {
		attrs[k] = v
	}

	// Unwrap nested errors and record the chain
	var chain []string
	var current error = err
	for current != nil {
		chain = append(chain, fmt.Sprintf("%T: %s", current, current.Error()))
		current = errors.Unwrap(current)
	}
	if len(chain) > 1 {
		attrs["exception.chain"] = strings.Join(chain, " -> ")
	}

	s.AddEvent("exception", attrs)

	// Optionally set the span status to error
	if config.setStatusError {
		s.SetStatus(StatusError, err.Error())
	}
}

type errorConfig struct {
	withStackTrace bool
	setStatusError bool
	extraAttrs     map[string]any
}

type ErrorOption func(*errorConfig)

// WithStackTrace includes a Go stack trace in the error recording.
func WithStackTrace() ErrorOption {
	return func(c *errorConfig) { c.withStackTrace = true }
}

// WithStatusError also sets the span status to Error.
func WithStatusError() ErrorOption {
	return func(c *errorConfig) { c.setStatusError = true }
}

// WithErrorAttributes adds extra attributes to the error event.
func WithErrorAttributes(attrs map[string]any) ErrorOption {
	return func(c *errorConfig) { c.extraAttrs = attrs }
}

// captureStackTrace returns a string representation of the current goroutine's stack.
func captureStackTrace(skip int) string {
	var pcs [32]uintptr
	n := runtime.Callers(skip, pcs[:])
	frames := runtime.CallersFrames(pcs[:n])

	var sb strings.Builder
	for {
		frame, more := frames.Next()
		fmt.Fprintf(&sb, "%s\n\t%s:%d\n", frame.Function, frame.File, frame.Line)
		if !more {
			break
		}
	}
	return sb.String()
}

// ---------- Panic Recovery with Tracing ----------

// RecoverWithSpan recovers from panics and records them in the span.
// Use this at the top of any traced function that might panic.
func RecoverWithSpan(span *Span) {
	if r := recover(); r != nil {
		// Convert panic value to error
		var err error
		switch v := r.(type) {
		case error:
			err = v
		case string:
			err = errors.New(v)
		default:
			err = fmt.Errorf("panic: %v", v)
		}

		span.RecordError(err,
			WithStackTrace(),
			WithStatusError(),
			WithErrorAttributes(map[string]any{
				"exception.escaped": true, // Indicates the error originated from a panic
			}),
		)
		span.End()
	}
}

// ---------- Custom Error Types ----------

// AppError is a domain-specific error with additional context.
type AppError struct {
	Code    string
	Message string
	Cause   error
	Context map[string]any
}

func (e *AppError) Error() string {
	if e.Cause != nil {
		return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.Cause)
	}
	return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e *AppError) Unwrap() error { return e.Cause }

// RecordAppError records an AppError with its full context.
func RecordAppError(span *Span, appErr *AppError) {
	attrs := map[string]any{
		"error.code": appErr.Code,
	}
	for k, v := range appErr.Context {
		attrs["error.context."+k] = v
	}
	span.RecordError(appErr, WithStatusError(), WithStackTrace(), WithErrorAttributes(attrs))
}

// ---------- Tracer ----------

type contextKey struct{}
var spanKey = contextKey{}

func spanFromContext(ctx context.Context) *Span {
	s, _ := ctx.Value(spanKey).(*Span)
	return s
}

type Tracer struct {
	service string
	mu      sync.Mutex
	spans   []*Span
}

func NewTracer(service string) *Tracer { return &Tracer{service: service} }

func (t *Tracer) StartSpan(ctx context.Context, name string) (context.Context, *Span) {
	parent := spanFromContext(ctx)
	var traceID TraceID
	var parentID SpanID
	if parent != nil {
		traceID = parent.TraceID
		parentID = parent.SpanID
	} else {
		_, _ = rand.Read(traceID[:])
	}
	var spanID SpanID
	_, _ = rand.Read(spanID[:])
	span := &Span{
		TraceID: traceID, SpanID: spanID, ParentID: parentID,
		Name: name, StartTime: time.Now(), ServiceName: t.service,
		Attributes: make(map[string]any),
	}
	t.mu.Lock()
	t.spans = append(t.spans, span)
	t.mu.Unlock()
	return context.WithValue(ctx, spanKey, span), span
}

// ---------- Demo Functions ----------

func riskyOperation(ctx context.Context, tracer *Tracer) {
	ctx, span := tracer.StartSpan(ctx, "risky-operation")
	defer RecoverWithSpan(span) // Will catch panics and record them

	// Simulate a panic
	var data []int
	_ = data[5] // index out of range panic

	_ = ctx
}

func databaseOperation(ctx context.Context, tracer *Tracer) error {
	ctx, span := tracer.StartSpan(ctx, "db-operation")
	defer span.End()

	// Simulate a database error
	dbErr := fmt.Errorf("connection refused: dial tcp 10.0.0.5:5432: i/o timeout")
	wrappedErr := fmt.Errorf("query failed: %w", dbErr)

	span.RecordError(wrappedErr, WithStatusError(), WithStackTrace())
	return wrappedErr
}

func businessLogicError(ctx context.Context, tracer *Tracer) error {
	ctx, span := tracer.StartSpan(ctx, "create-order")
	defer span.End()

	appErr := &AppError{
		Code:    "INSUFFICIENT_FUNDS",
		Message: "user does not have enough balance",
		Cause:   fmt.Errorf("balance 50.00 < required 150.00"),
		Context: map[string]any{
			"user_id":   "usr_abc123",
			"balance":   50.00,
			"required":  150.00,
			"order_id":  "ord_xyz789",
		},
	}

	RecordAppError(span, appErr)
	return appErr
}

// ---------- Main ----------

func main() {
	tracer := NewTracer("error-demo")
	ctx := context.Background()

	fmt.Println("=== Error Recording Demo ===")
	fmt.Println()

	// 1. Panic recovery
	fmt.Println("--- Panic Recovery ---")
	func() {
		defer func() { recover() }() // outer recovery so main does not crash
		riskyOperation(ctx, tracer)
	}()

	// 2. Wrapped database error
	fmt.Println("--- Database Error ---")
	if err := databaseOperation(ctx, tracer); err != nil {
		fmt.Printf("Got error: %v\n", err)
	}

	// 3. Business logic error
	fmt.Println("--- Business Logic Error ---")
	if err := businessLogicError(ctx, tracer); err != nil {
		fmt.Printf("Got error: %v\n", err)
	}

	// Print all spans with their events
	fmt.Println()
	fmt.Println("=== Span Summary ===")
	for _, s := range tracer.spans {
		status := "UNSET"
		if s.Status == StatusOK { status = "OK" }
		if s.Status == StatusError { status = "ERROR" }
		fmt.Printf("\nSpan: %s (status=%s)\n", s.Name, status)
		if s.StatusMsg != "" {
			fmt.Printf("  Status message: %s\n", s.StatusMsg)
		}
		for _, event := range s.Events {
			fmt.Printf("  Event: %s\n", event.Name)
			for k, v := range event.Attributes {
				val := fmt.Sprintf("%v", v)
				if len(val) > 100 {
					val = val[:100] + "..."
				}
				fmt.Printf("    %s = %s\n", k, val)
			}
		}
	}
}
```

---

## 12. Trace Analysis Patterns

### Using Traces for Debugging and Optimization

Traces are not just pretty pictures. They are structured data that enable powerful analysis patterns. This section shows how to compute useful metrics from trace data programmatically.

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sort"
	"strings"
	"time"
)

// ---------- Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	Attributes  map[string]any
	ServiceName string
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Service Dependency Map ----------

// ServiceEdge represents a call from one service to another.
type ServiceEdge struct {
	From  string
	To    string
	Count int
	AvgMs float64
	MaxMs float64
	Errors int
}

// BuildServiceDependencyMap analyzes spans to determine which services
// call which other services, how often, and how fast.
func BuildServiceDependencyMap(spans []*Span) []ServiceEdge {
	// Index spans by SpanID for parent lookup
	byID := make(map[SpanID]*Span)
	for _, s := range spans {
		byID[s.SpanID] = s
	}

	// For each span, if its parent is in a different service, that is an edge
	edgeMap := make(map[string]*ServiceEdge) // "from->to" -> edge

	for _, s := range spans {
		if s.ParentID.IsZero() {
			continue
		}
		parent, ok := byID[s.ParentID]
		if !ok {
			continue
		}
		if parent.ServiceName == s.ServiceName {
			continue // Same service, not an edge
		}

		key := parent.ServiceName + "->" + s.ServiceName
		edge, ok := edgeMap[key]
		if !ok {
			edge = &ServiceEdge{From: parent.ServiceName, To: s.ServiceName}
			edgeMap[key] = edge
		}
		edge.Count++
		ms := float64(s.Duration().Microseconds()) / 1000.0
		edge.AvgMs = (edge.AvgMs*float64(edge.Count-1) + ms) / float64(edge.Count)
		if ms > edge.MaxMs {
			edge.MaxMs = ms
		}
		if s.Status == StatusError {
			edge.Errors++
		}
	}

	edges := make([]ServiceEdge, 0, len(edgeMap))
	for _, e := range edgeMap {
		edges = append(edges, *e)
	}
	sort.Slice(edges, func(i, j int) bool {
		return edges[i].From < edges[j].From || (edges[i].From == edges[j].From && edges[i].To < edges[j].To)
	})
	return edges
}

// ---------- Latency Breakdown ----------

// LatencyBreakdown shows how time is distributed across services in a trace.
type LatencyBreakdown struct {
	Service     string
	TotalTime   time.Duration
	SelfTime    time.Duration // Time not spent in child spans
	SpanCount   int
	Percentage  float64
}

// ComputeLatencyBreakdown analyzes a trace to show where time is spent.
func ComputeLatencyBreakdown(spans []*Span) []LatencyBreakdown {
	if len(spans) == 0 {
		return nil
	}

	// Group by service
	byService := make(map[string][]time.Duration)

	// Index spans by SpanID
	byID := make(map[SpanID]*Span)
	for _, s := range spans {
		byID[s.SpanID] = s
	}

	// Compute self time for each span
	// Self time = span duration - sum of direct child durations
	childDurations := make(map[SpanID]time.Duration) // parent -> total child duration
	for _, s := range spans {
		if !s.ParentID.IsZero() {
			childDurations[s.ParentID] += s.Duration()
		}
	}

	for _, s := range spans {
		selfTime := s.Duration() - childDurations[s.SpanID]
		if selfTime < 0 {
			selfTime = 0 // Can happen with overlapping children
		}
		byService[s.ServiceName] = append(byService[s.ServiceName], selfTime)
	}

	// Find total trace duration
	var traceStart, traceEnd time.Time
	for _, s := range spans {
		if traceStart.IsZero() || s.StartTime.Before(traceStart) {
			traceStart = s.StartTime
		}
		if s.EndTime.After(traceEnd) {
			traceEnd = s.EndTime
		}
	}
	totalDuration := traceEnd.Sub(traceStart)

	// Build breakdown
	var breakdown []LatencyBreakdown
	for service, durations := range byService {
		var total time.Duration
		for _, d := range durations {
			total += d
		}
		pct := 0.0
		if totalDuration > 0 {
			pct = float64(total) / float64(totalDuration) * 100
		}
		breakdown = append(breakdown, LatencyBreakdown{
			Service:    service,
			SelfTime:   total,
			SpanCount:  len(durations),
			Percentage: pct,
		})
	}

	sort.Slice(breakdown, func(i, j int) bool {
		return breakdown[i].SelfTime > breakdown[j].SelfTime
	})
	return breakdown
}

// ---------- N+1 Query Detection ----------

// DetectNPlusOne finds patterns where the same query is executed many times
// under the same parent span (classic N+1 query pattern).
type NPlusOneReport struct {
	ParentSpan string
	Query      string
	Count      int
	TotalMs    float64
}

func DetectNPlusOne(spans []*Span, threshold int) []NPlusOneReport {
	// Group DB spans by parent and query
	type key struct {
		parentID SpanID
		query    string
	}
	groups := make(map[key][]time.Duration)
	parentNames := make(map[SpanID]string)

	for _, s := range spans {
		parentNames[s.SpanID] = s.Name
	}

	for _, s := range spans {
		stmt, ok := s.Attributes["db.statement"]
		if !ok {
			continue
		}
		query := fmt.Sprintf("%v", stmt)
		k := key{parentID: s.ParentID, query: query}
		groups[k] = append(groups[k], s.Duration())
	}

	var reports []NPlusOneReport
	for k, durations := range groups {
		if len(durations) >= threshold {
			var total time.Duration
			for _, d := range durations {
				total += d
			}
			reports = append(reports, NPlusOneReport{
				ParentSpan: parentNames[k.parentID],
				Query:      k.query,
				Count:      len(durations),
				TotalMs:    float64(total.Microseconds()) / 1000.0,
			})
		}
	}

	sort.Slice(reports, func(i, j int) bool {
		return reports[i].Count > reports[j].Count
	})
	return reports
}

// ---------- Generate Test Data ----------

func generateTestTrace() []*Span {
	var traceID TraceID
	_, _ = rand.Read(traceID[:])

	genID := func() SpanID {
		var id SpanID
		_, _ = rand.Read(id[:])
		return id
	}

	now := time.Now()
	gatewayID := genID()
	authID := genID()
	orderID := genID()
	orderDBID := genID()
	userSvcID := genID()
	userDBID := genID()
	cacheID := genID()

	// Also simulate N+1 queries
	listHandlerID := genID()
	var nPlusOneSpans []*Span
	for i := 0; i < 25; i++ {
		nPlusOneSpans = append(nPlusOneSpans, &Span{
			TraceID: traceID, SpanID: genID(), ParentID: listHandlerID,
			Name: "db.Query", ServiceName: "order-service",
			StartTime: now.Add(time.Duration(50+i*3) * time.Millisecond),
			EndTime:   now.Add(time.Duration(52+i*3) * time.Millisecond),
			Status: StatusOK,
			Attributes: map[string]any{
				"db.statement": "SELECT * FROM users WHERE id = $1",
				"db.system":    "postgresql",
			},
		})
	}

	spans := []*Span{
		{TraceID: traceID, SpanID: gatewayID, Name: "GET /api/orders", ServiceName: "api-gateway",
			StartTime: now, EndTime: now.Add(120 * time.Millisecond), Status: StatusOK},
		{TraceID: traceID, SpanID: authID, ParentID: gatewayID, Name: "validate-token", ServiceName: "auth-service",
			StartTime: now.Add(2 * time.Millisecond), EndTime: now.Add(8 * time.Millisecond), Status: StatusOK},
		{TraceID: traceID, SpanID: orderID, ParentID: gatewayID, Name: "get-orders", ServiceName: "order-service",
			StartTime: now.Add(10 * time.Millisecond), EndTime: now.Add(110 * time.Millisecond), Status: StatusOK},
		{TraceID: traceID, SpanID: cacheID, ParentID: orderID, Name: "cache.Get", ServiceName: "order-service",
			StartTime: now.Add(12 * time.Millisecond), EndTime: now.Add(14 * time.Millisecond), Status: StatusOK,
			Attributes: map[string]any{"cache.hit": false}},
		{TraceID: traceID, SpanID: orderDBID, ParentID: orderID, Name: "db.Query", ServiceName: "order-service",
			StartTime: now.Add(15 * time.Millisecond), EndTime: now.Add(60 * time.Millisecond), Status: StatusOK,
			Attributes: map[string]any{"db.statement": "SELECT * FROM orders WHERE user_id = $1", "db.system": "postgresql"}},
		{TraceID: traceID, SpanID: userSvcID, ParentID: orderID, Name: "get-user", ServiceName: "user-service",
			StartTime: now.Add(62 * time.Millisecond), EndTime: now.Add(90 * time.Millisecond), Status: StatusOK},
		{TraceID: traceID, SpanID: userDBID, ParentID: userSvcID, Name: "db.Query", ServiceName: "user-service",
			StartTime: now.Add(64 * time.Millisecond), EndTime: now.Add(88 * time.Millisecond), Status: StatusOK,
			Attributes: map[string]any{"db.statement": "SELECT * FROM users WHERE id = $1", "db.system": "postgresql"}},
		// N+1 query scenario
		{TraceID: traceID, SpanID: listHandlerID, ParentID: gatewayID, Name: "list-users-for-orders", ServiceName: "order-service",
			StartTime: now.Add(45 * time.Millisecond), EndTime: now.Add(130 * time.Millisecond), Status: StatusOK},
	}

	spans = append(spans, nPlusOneSpans...)
	return spans
}

// ---------- Main ----------

func main() {
	spans := generateTestTrace()

	// 1. Service Dependency Map
	fmt.Println("=== Service Dependency Map ===")
	edges := BuildServiceDependencyMap(spans)
	fmt.Printf("%-20s %-20s %6s %8s %8s %6s\n", "FROM", "TO", "COUNT", "AVG(ms)", "MAX(ms)", "ERRORS")
	fmt.Println(strings.Repeat("-", 80))
	for _, e := range edges {
		fmt.Printf("%-20s %-20s %6d %8.2f %8.2f %6d\n",
			e.From, e.To, e.Count, e.AvgMs, e.MaxMs, e.Errors)
	}

	// 2. Latency Breakdown
	fmt.Println("\n=== Latency Breakdown by Service ===")
	breakdown := ComputeLatencyBreakdown(spans)
	fmt.Printf("%-20s %10s %6s %8s\n", "SERVICE", "SELF TIME", "SPANS", "PCT")
	fmt.Println(strings.Repeat("-", 50))
	for _, b := range breakdown {
		fmt.Printf("%-20s %10s %6d %7.1f%%\n",
			b.Service, b.SelfTime, b.SpanCount, b.Percentage)
	}

	// 3. N+1 Query Detection
	fmt.Println("\n=== N+1 Query Detection (threshold: 5) ===")
	reports := DetectNPlusOne(spans, 5)
	if len(reports) == 0 {
		fmt.Println("  No N+1 patterns detected.")
	} else {
		for _, r := range reports {
			fmt.Printf("  WARNING: Query executed %d times under span %q\n", r.Count, r.ParentSpan)
			fmt.Printf("    Query: %s\n", r.Query)
			fmt.Printf("    Total time: %.2fms\n", r.TotalMs)
			fmt.Println()
		}
	}
}
```

---

## 13. Building a Trace Visualizer

### ASCII Waterfall Diagram

Tracing backends like Jaeger show a waterfall view of spans. Here we build a simple ASCII version that shows the timeline, nesting, and duration of each span.

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"sort"
	"strings"
	"time"
)

// ---------- Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	ServiceName string
	Attributes  map[string]any
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Trace Tree ----------

type traceNode struct {
	span     *Span
	children []*traceNode
	depth    int
}

// buildTree constructs a tree from a flat list of spans.
func buildTree(spans []*Span) []*traceNode {
	nodeMap := make(map[SpanID]*traceNode)
	var roots []*traceNode

	// Create nodes
	for _, s := range spans {
		nodeMap[s.SpanID] = &traceNode{span: s}
	}

	// Link parents to children
	for _, s := range spans {
		node := nodeMap[s.SpanID]
		if s.ParentID.IsZero() {
			roots = append(roots, node)
		} else if parent, ok := nodeMap[s.ParentID]; ok {
			parent.children = append(parent.children, node)
		} else {
			// Orphan span (parent not in this set) -- treat as root
			roots = append(roots, node)
		}
	}

	// Sort children by start time
	var sortChildren func(node *traceNode)
	sortChildren = func(node *traceNode) {
		sort.Slice(node.children, func(i, j int) bool {
			return node.children[i].span.StartTime.Before(node.children[j].span.StartTime)
		})
		for _, child := range node.children {
			sortChildren(child)
		}
	}
	for _, root := range roots {
		sortChildren(root)
	}

	// Set depths
	var setDepth func(node *traceNode, depth int)
	setDepth = func(node *traceNode, depth int) {
		node.depth = depth
		for _, child := range node.children {
			setDepth(child, depth+1)
		}
	}
	for _, root := range roots {
		setDepth(root, 0)
	}

	return roots
}

// ---------- ASCII Visualizer ----------

// PrintTrace renders a trace as an ASCII waterfall diagram.
func PrintTrace(spans []*Span) {
	if len(spans) == 0 {
		fmt.Println("(empty trace)")
		return
	}

	roots := buildTree(spans)

	// Find trace time boundaries
	traceStart := spans[0].StartTime
	traceEnd := spans[0].EndTime
	for _, s := range spans {
		if s.StartTime.Before(traceStart) {
			traceStart = s.StartTime
		}
		if s.EndTime.After(traceEnd) {
			traceEnd = s.EndTime
		}
	}
	traceDuration := traceEnd.Sub(traceStart)

	// Print header
	fmt.Printf("Trace ID: %s\n", spans[0].TraceID)
	fmt.Printf("Duration: %s\n", traceDuration)
	fmt.Printf("Spans:    %d\n", len(spans))
	fmt.Println()

	// Waterfall width
	const barWidth = 50
	const nameWidth = 40

	// Print scale
	fmt.Printf("%-*s |", nameWidth, "")
	for i := 0; i <= barWidth; i += barWidth / 4 {
		t := time.Duration(float64(traceDuration) * float64(i) / float64(barWidth))
		label := fmt.Sprintf("%.0fms", float64(t.Microseconds())/1000.0)
		fmt.Printf("%-*s", barWidth/4, label)
	}
	fmt.Println()
	fmt.Printf("%-*s |%s|\n", nameWidth, "", strings.Repeat("-", barWidth))

	// Flatten tree into display order
	var ordered []*traceNode
	var flatten func(node *traceNode)
	flatten = func(node *traceNode) {
		ordered = append(ordered, node)
		for _, child := range node.children {
			flatten(child)
		}
	}
	for _, root := range roots {
		flatten(root)
	}

	// Print each span
	for _, node := range ordered {
		s := node.span

		// Build name with indentation
		indent := strings.Repeat("  ", node.depth)
		connector := ""
		if node.depth > 0 {
			connector = "|-- "
		}
		displayName := fmt.Sprintf("%s%s%s [%s]", indent, connector, s.Name, s.ServiceName)
		if len(displayName) > nameWidth {
			displayName = displayName[:nameWidth-3] + "..."
		}

		// Calculate bar position
		startOffset := s.StartTime.Sub(traceStart)
		duration := s.Duration()

		startPos := int(float64(startOffset) / float64(traceDuration) * float64(barWidth))
		barLen := int(float64(duration) / float64(traceDuration) * float64(barWidth))
		if barLen < 1 {
			barLen = 1
		}
		if startPos+barLen > barWidth {
			barLen = barWidth - startPos
		}

		// Build the bar
		bar := strings.Repeat(" ", startPos)
		statusChar := "="
		if s.Status == StatusError {
			statusChar = "X"
		}
		bar += strings.Repeat(statusChar, barLen)
		bar += strings.Repeat(" ", barWidth-startPos-barLen)

		// Duration label
		durationLabel := fmt.Sprintf(" %.1fms", float64(s.Duration().Microseconds())/1000.0)

		fmt.Printf("%-*s |%s|%s\n", nameWidth, displayName, bar, durationLabel)
	}

	// Print summary
	fmt.Printf("%-*s |%s|\n", nameWidth, "", strings.Repeat("-", barWidth))
}

// PrintTraceTree renders just the tree structure without the waterfall.
func PrintTraceTree(spans []*Span) {
	roots := buildTree(spans)

	var printNode func(node *traceNode, prefix string, isLast bool)
	printNode = func(node *traceNode, prefix string, isLast bool) {
		s := node.span
		connector := "|-- "
		if isLast {
			connector = "`-- "
		}
		if node.depth == 0 {
			connector = ""
		}

		status := " "
		if s.Status == StatusError {
			status = "!"
		}

		fmt.Printf("%s%s%s[%s] %s (%s)%s\n",
			prefix, connector, status, s.ServiceName, s.Name,
			s.Duration(), status)

		childPrefix := prefix
		if node.depth > 0 {
			if isLast {
				childPrefix += "    "
			} else {
				childPrefix += "|   "
			}
		}

		for i, child := range node.children {
			printNode(child, childPrefix, i == len(node.children)-1)
		}
	}

	for _, root := range roots {
		printNode(root, "", true)
	}
}

// ---------- Generate Test Trace ----------

func generateTestTrace() []*Span {
	var traceID TraceID
	_, _ = rand.Read(traceID[:])

	genID := func() SpanID {
		var id SpanID
		_, _ = rand.Read(id[:])
		return id
	}

	now := time.Now()

	gatewayID := genID()
	authID := genID()
	orderSvcID := genID()
	cacheID := genID()
	orderDBID := genID()
	userSvcID := genID()
	userDBID := genID()
	emailID := genID()

	return []*Span{
		{TraceID: traceID, SpanID: gatewayID,
			Name: "GET /api/orders/123", ServiceName: "api-gateway",
			StartTime: now, EndTime: now.Add(150 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: authID, ParentID: gatewayID,
			Name: "validate-token", ServiceName: "auth-service",
			StartTime: now.Add(2 * time.Millisecond), EndTime: now.Add(10 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: orderSvcID, ParentID: gatewayID,
			Name: "get-order", ServiceName: "order-service",
			StartTime: now.Add(12 * time.Millisecond), EndTime: now.Add(140 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: cacheID, ParentID: orderSvcID,
			Name: "cache.Get orders:123", ServiceName: "order-service",
			StartTime: now.Add(14 * time.Millisecond), EndTime: now.Add(16 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: orderDBID, ParentID: orderSvcID,
			Name: "SELECT * FROM orders", ServiceName: "order-service",
			StartTime: now.Add(18 * time.Millisecond), EndTime: now.Add(65 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: userSvcID, ParentID: orderSvcID,
			Name: "get-user", ServiceName: "user-service",
			StartTime: now.Add(67 * time.Millisecond), EndTime: now.Add(105 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: userDBID, ParentID: userSvcID,
			Name: "SELECT * FROM users", ServiceName: "user-service",
			StartTime: now.Add(69 * time.Millisecond), EndTime: now.Add(100 * time.Millisecond), Status: StatusOK},

		{TraceID: traceID, SpanID: emailID, ParentID: orderSvcID,
			Name: "send-notification", ServiceName: "email-service",
			StartTime: now.Add(107 * time.Millisecond), EndTime: now.Add(135 * time.Millisecond), Status: StatusError},
	}
}

func main() {
	spans := generateTestTrace()

	fmt.Println("========== TRACE TREE ==========")
	fmt.Println()
	PrintTraceTree(spans)

	fmt.Println()
	fmt.Println("========== WATERFALL DIAGRAM ==========")
	fmt.Println()
	PrintTrace(spans)
}
```

This produces output like:

```
========== TRACE TREE ==========

 [api-gateway] GET /api/orders/123 (150ms)
|--  [auth-service] validate-token (8ms)
|--  [order-service] get-order (128ms)
|   |--  [order-service] cache.Get orders:123 (2ms)
|   |--  [order-service] SELECT * FROM orders (47ms)
|   |--  [user-service] get-user (38ms)
|   |   `--  [user-service] SELECT * FROM users (31ms)
|   `-- ![email-service] send-notification (28ms)!

========== WATERFALL DIAGRAM ==========

Trace ID: ...
Duration: 150ms
Spans:    8

                                         |0ms         37ms        75ms        112ms       150ms
                                         |--------------------------------------------------|
GET /api/orders/123 [api-gateway]        |==================================================| 150.0ms
  |-- validate-token [auth-service]      |=                                                 | 8.0ms
  |-- get-order [order-service]          | ===============================================  | 128.0ms
    |-- cache.Get orders:123 [order-s... |  =                                               | 2.0ms
    |-- SELECT * FROM orders [order-s... |  ===============                                 | 47.0ms
    |-- get-user [user-service]          |                  ============                    | 38.0ms
      |-- SELECT * FROM users [user-s... |                   ==========                     | 31.0ms
    |-- send-notification [email-serv... |                                  XXXXXXXXX       | 28.0ms
                                         |--------------------------------------------------|
```

---

## 14. Integration with Jaeger and Tempo

### Sending Traces to Jaeger

Jaeger is one of the most popular open-source distributed tracing backends. It accepts spans via the OpenTelemetry Protocol (OTLP) or its own Thrift format. The easiest way to send traces to Jaeger is using OTLP over HTTP.

```go
package main

import (
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
	"time"
)

// ---------- Core Types ----------

type TraceID [16]byte
func (t TraceID) String() string { return hex.EncodeToString(t[:]) }

type SpanID [8]byte
func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s { if b != 0 { return false } }
	return true
}

type SpanStatus int
const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int
const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	ServiceName string
	mu          sync.Mutex
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil { s.Attributes = make(map[string]any) }
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) End() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.EndTime = time.Now()
}

// ---------- OTLP Exporter for Jaeger ----------

// JaegerOTLPExporter sends spans to Jaeger using the OTLP HTTP protocol.
// Jaeger supports OTLP natively since version 1.35.
//
// To run Jaeger locally for testing:
//
//   docker run -d --name jaeger \
//     -p 16686:16686 \
//     -p 4318:4318 \
//     jaegertracing/all-in-one:latest
//
// - Port 4318: OTLP HTTP receiver
// - Port 16686: Jaeger UI
type JaegerOTLPExporter struct {
	endpoint string
	client   *http.Client
}

func NewJaegerOTLPExporter(endpoint string) *JaegerOTLPExporter {
	if endpoint == "" {
		endpoint = "http://localhost:4318/v1/traces"
	}
	return &JaegerOTLPExporter{
		endpoint: endpoint,
		client:   &http.Client{Timeout: 10 * time.Second},
	}
}

// OTLP JSON structures (subset needed for trace export)
type otlpExportRequest struct {
	ResourceSpans []otlpResourceSpans `json:"resourceSpans"`
}

type otlpResourceSpans struct {
	Resource   otlpResource    `json:"resource"`
	ScopeSpans []otlpScopeSpans `json:"scopeSpans"`
}

type otlpResource struct {
	Attributes []otlpKV `json:"attributes"`
}

type otlpScopeSpans struct {
	Scope *otlpScope  `json:"scope,omitempty"`
	Spans []otlpSpan  `json:"spans"`
}

type otlpScope struct {
	Name    string `json:"name"`
	Version string `json:"version"`
}

type otlpSpan struct {
	TraceID            string   `json:"traceId"`
	SpanID             string   `json:"spanId"`
	ParentSpanID       string   `json:"parentSpanId,omitempty"`
	Name               string   `json:"name"`
	Kind               int      `json:"kind"`
	StartTimeUnixNano  string   `json:"startTimeUnixNano"`
	EndTimeUnixNano    string   `json:"endTimeUnixNano"`
	Attributes         []otlpKV `json:"attributes,omitempty"`
	Status             *otlpStatus `json:"status,omitempty"`
}

type otlpKV struct {
	Key   string    `json:"key"`
	Value otlpValue `json:"value"`
}

type otlpValue struct {
	StringValue *string `json:"stringValue,omitempty"`
	IntValue    *string `json:"intValue,omitempty"`
	BoolValue   *bool   `json:"boolValue,omitempty"`
}

type otlpStatus struct {
	Code    int    `json:"code"`
	Message string `json:"message,omitempty"`
}

func makeStringValue(s string) otlpValue {
	return otlpValue{StringValue: &s}
}

// Export sends spans to Jaeger via OTLP HTTP.
func (e *JaegerOTLPExporter) Export(ctx context.Context, spans []*Span) error {
	if len(spans) == 0 {
		return nil
	}

	// Group spans by service
	byService := make(map[string][]*Span)
	for _, s := range spans {
		byService[s.ServiceName] = append(byService[s.ServiceName], s)
	}

	req := otlpExportRequest{}

	for service, serviceSpans := range byService {
		var otlpSpans []otlpSpan

		for _, s := range serviceSpans {
			os := otlpSpan{
				TraceID:           s.TraceID.String(),
				SpanID:            s.SpanID.String(),
				Name:              s.Name,
				Kind:              mapKind(s.Kind),
				StartTimeUnixNano: fmt.Sprintf("%d", s.StartTime.UnixNano()),
				EndTimeUnixNano:   fmt.Sprintf("%d", s.EndTime.UnixNano()),
			}

			if !s.ParentID.IsZero() {
				os.ParentSpanID = s.ParentID.String()
			}

			for k, v := range s.Attributes {
				str := fmt.Sprintf("%v", v)
				os.Attributes = append(os.Attributes, otlpKV{
					Key:   k,
					Value: makeStringValue(str),
				})
			}

			if s.Status != StatusUnset {
				os.Status = &otlpStatus{
					Code:    int(s.Status),
					Message: s.StatusMsg,
				}
			}

			otlpSpans = append(otlpSpans, os)
		}

		serviceName := service
		req.ResourceSpans = append(req.ResourceSpans, otlpResourceSpans{
			Resource: otlpResource{
				Attributes: []otlpKV{
					{Key: "service.name", Value: makeStringValue(serviceName)},
				},
			},
			ScopeSpans: []otlpScopeSpans{
				{
					Scope: &otlpScope{Name: "custom-tracer", Version: "1.0.0"},
					Spans: otlpSpans,
				},
			},
		})
	}

	body, err := json.Marshal(req)
	if err != nil {
		return fmt.Errorf("marshal OTLP request: %w", err)
	}

	httpReq, err := http.NewRequestWithContext(ctx, http.MethodPost, e.endpoint, bytes.NewReader(body))
	if err != nil {
		return fmt.Errorf("create HTTP request: %w", err)
	}
	httpReq.Header.Set("Content-Type", "application/json")

	resp, err := e.client.Do(httpReq)
	if err != nil {
		return fmt.Errorf("send to Jaeger: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 400 {
		return fmt.Errorf("Jaeger returned %d", resp.StatusCode)
	}

	return nil
}

func mapKind(k SpanKind) int {
	// OTLP kind enum: 0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER
	switch k {
	case SpanKindInternal:
		return 1
	case SpanKindServer:
		return 2
	case SpanKindClient:
		return 3
	default:
		return 0
	}
}

// ---------- Tempo Configuration ----------

// GrafanaTempoExporter sends traces to Grafana Tempo.
// Tempo accepts OTLP over HTTP on port 4318 (same protocol as Jaeger).
//
// To run Tempo locally:
//
//   docker run -d --name tempo \
//     -p 3200:3200 \
//     -p 4318:4318 \
//     grafana/tempo:latest
//
// To query traces, use Grafana connected to Tempo as a data source.
type GrafanaTempoExporter struct {
	*JaegerOTLPExporter // Same OTLP protocol, different endpoint
}

func NewGrafanaTempoExporter(endpoint string) *GrafanaTempoExporter {
	if endpoint == "" {
		endpoint = "http://localhost:4318/v1/traces"
	}
	return &GrafanaTempoExporter{
		JaegerOTLPExporter: NewJaegerOTLPExporter(endpoint),
	}
}

// ---------- Demo ----------

func main() {
	// Create a Jaeger exporter (will fail if Jaeger is not running, which is fine for demo)
	exporter := NewJaegerOTLPExporter("http://localhost:4318/v1/traces")

	// Generate sample trace
	var traceID TraceID
	_, _ = rand.Read(traceID[:])

	genID := func() SpanID {
		var id SpanID
		_, _ = rand.Read(id[:])
		return id
	}

	now := time.Now()
	rootID := genID()
	dbID := genID()
	cacheID := genID()

	spans := []*Span{
		{
			TraceID: traceID, SpanID: rootID,
			Name: "GET /api/users/42", Kind: SpanKindServer,
			StartTime: now, EndTime: now.Add(85 * time.Millisecond),
			Status: StatusOK, ServiceName: "user-api",
			Attributes: map[string]any{
				"http.method": "GET", "http.status_code": 200,
				"http.url": "/api/users/42",
			},
		},
		{
			TraceID: traceID, SpanID: cacheID, ParentID: rootID,
			Name: "cache.Get user:42", Kind: SpanKindClient,
			StartTime: now.Add(2 * time.Millisecond), EndTime: now.Add(4 * time.Millisecond),
			Status: StatusOK, ServiceName: "user-api",
			Attributes: map[string]any{"cache.hit": "false", "cache.system": "redis"},
		},
		{
			TraceID: traceID, SpanID: dbID, ParentID: rootID,
			Name: "SELECT * FROM users WHERE id = $1", Kind: SpanKindClient,
			StartTime: now.Add(5 * time.Millisecond), EndTime: now.Add(75 * time.Millisecond),
			Status: StatusOK, ServiceName: "user-api",
			Attributes: map[string]any{
				"db.system": "postgresql", "db.name": "users_db",
				"db.statement": "SELECT * FROM users WHERE id = $1",
			},
		},
	}

	fmt.Println("=== Sending Trace to Jaeger via OTLP ===")
	fmt.Printf("Trace ID: %s\n", traceID)
	fmt.Printf("Spans:    %d\n", len(spans))
	fmt.Println()

	err := exporter.Export(context.Background(), spans)
	if err != nil {
		fmt.Printf("Export failed (expected if Jaeger is not running): %v\n", err)
		fmt.Println()
		fmt.Println("To see this work, start Jaeger with:")
		fmt.Println("  docker run -d --name jaeger \\")
		fmt.Println("    -p 16686:16686 \\")
		fmt.Println("    -p 4318:4318 \\")
		fmt.Println("    jaegertracing/all-in-one:latest")
		fmt.Println()
		fmt.Println("Then visit http://localhost:16686 to see the trace.")
	} else {
		fmt.Println("Export successful!")
		fmt.Println("View the trace at: http://localhost:16686")
	}

	// Print the OTLP JSON that would be sent
	fmt.Println()
	fmt.Println("=== OTLP JSON Payload (what gets sent) ===")
	byService := make(map[string][]*Span)
	for _, s := range spans {
		byService[s.ServiceName] = append(byService[s.ServiceName], s)
	}

	req := otlpExportRequest{}
	for service, serviceSpans := range byService {
		var otlpSpans []otlpSpan
		for _, s := range serviceSpans {
			os := otlpSpan{
				TraceID:           s.TraceID.String(),
				SpanID:            s.SpanID.String(),
				Name:              s.Name,
				Kind:              mapKind(s.Kind),
				StartTimeUnixNano: fmt.Sprintf("%d", s.StartTime.UnixNano()),
				EndTimeUnixNano:   fmt.Sprintf("%d", s.EndTime.UnixNano()),
			}
			if !s.ParentID.IsZero() {
				os.ParentSpanID = s.ParentID.String()
			}
			otlpSpans = append(otlpSpans, os)
		}
		req.ResourceSpans = append(req.ResourceSpans, otlpResourceSpans{
			Resource: otlpResource{
				Attributes: []otlpKV{{Key: "service.name", Value: makeStringValue(service)}},
			},
			ScopeSpans: []otlpScopeSpans{{Spans: otlpSpans}},
		})
	}
	data, _ := json.MarshalIndent(req, "", "  ")
	fmt.Println(string(data))
}
```

---

## 15. Real-World Example: Multi-Service Tracing

### Complete Example: Three Services with Full Distributed Tracing

This is the capstone example. We build three HTTP services that communicate with each other, all sharing a single distributed trace. This demonstrates every concept from the chapter in a realistic scenario.

- **API Gateway** (port 8080) -- receives user requests, calls Order Service
- **Order Service** (port 8081) -- handles order logic, calls User Service and queries DB
- **User Service** (port 8082) -- handles user lookups

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"strings"
	"sync"
	"time"
)

// ============================================================
// TRACING LIBRARY (would normally be a separate package)
// ============================================================

// ---------- ID Types ----------

type TraceID [16]byte

func (t TraceID) String() string { return hex.EncodeToString(t[:]) }
func (t TraceID) IsZero() bool {
	for _, b := range t {
		if b != 0 {
			return false
		}
	}
	return true
}

type SpanID [8]byte

func (s SpanID) String() string { return hex.EncodeToString(s[:]) }
func (s SpanID) IsZero() bool {
	for _, b := range s {
		if b != 0 {
			return false
		}
	}
	return true
}

func generateTraceID() TraceID {
	var id TraceID
	_, _ = rand.Read(id[:])
	return id
}

func generateSpanID() SpanID {
	var id SpanID
	_, _ = rand.Read(id[:])
	return id
}

// ---------- Span ----------

type SpanStatus int

const (
	StatusUnset SpanStatus = iota
	StatusOK
	StatusError
)

type SpanKind int

const (
	SpanKindInternal SpanKind = iota
	SpanKindServer
	SpanKindClient
)

type Span struct {
	TraceID     TraceID
	SpanID      SpanID
	ParentID    SpanID
	Name        string
	Kind        SpanKind
	StartTime   time.Time
	EndTime     time.Time
	Status      SpanStatus
	StatusMsg   string
	Attributes  map[string]any
	ServiceName string
	mu          sync.Mutex
	ended       bool
	onEnd       func(*Span) // callback when span ends
}

func (s *Span) SetAttribute(key string, value any) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.Attributes == nil {
		s.Attributes = make(map[string]any)
	}
	s.Attributes[key] = value
}

func (s *Span) SetStatus(status SpanStatus, msg string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Status = status
	s.StatusMsg = msg
}

func (s *Span) End() {
	s.mu.Lock()
	if s.ended {
		s.mu.Unlock()
		return
	}
	s.ended = true
	s.EndTime = time.Now()
	onEnd := s.onEnd
	s.mu.Unlock()

	if onEnd != nil {
		onEnd(s)
	}
}

func (s *Span) Duration() time.Duration { return s.EndTime.Sub(s.StartTime) }

// ---------- Context ----------

type contextKeyType struct{}

var ctxKey = contextKeyType{}

func SpanFromContext(ctx context.Context) *Span {
	s, _ := ctx.Value(ctxKey).(*Span)
	return s
}

func ContextWithSpan(ctx context.Context, span *Span) context.Context {
	return context.WithValue(ctx, ctxKey, span)
}

// ---------- Tracer ----------

type Tracer struct {
	serviceName string
	mu          sync.Mutex
	allSpans    []*Span
}

func NewTracer(serviceName string) *Tracer {
	return &Tracer{serviceName: serviceName}
}

func (t *Tracer) StartSpan(ctx context.Context, name string, kind SpanKind) (context.Context, *Span) {
	parent := SpanFromContext(ctx)

	var traceID TraceID
	var parentID SpanID

	if parent != nil {
		traceID = parent.TraceID
		parentID = parent.SpanID
	} else {
		traceID = generateTraceID()
	}

	span := &Span{
		TraceID:     traceID,
		SpanID:      generateSpanID(),
		ParentID:    parentID,
		Name:        name,
		Kind:        kind,
		StartTime:   time.Now(),
		ServiceName: t.serviceName,
		Attributes:  make(map[string]any),
		onEnd: func(s *Span) {
			// Log the span when it ends
			parent := "ROOT"
			if !s.ParentID.IsZero() {
				parent = s.ParentID.String()[:12] + "..."
			}
			status := "OK"
			if s.Status == StatusError {
				status = "ERR"
			}
			fmt.Printf("[%s] %-35s trace=%s.. span=%s.. parent=%-16s %s %s\n",
				s.ServiceName,
				s.Name,
				s.TraceID.String()[:12],
				s.SpanID.String()[:12],
				parent,
				s.Duration(),
				status,
			)

			// Collect for visualization
			t.mu.Lock()
			t.allSpans = append(t.allSpans, s)
			t.mu.Unlock()
		},
	}

	return ContextWithSpan(ctx, span), span
}

func (t *Tracer) Spans() []*Span {
	t.mu.Lock()
	defer t.mu.Unlock()
	return append([]*Span{}, t.allSpans...)
}

// ---------- W3C Propagation ----------

func InjectTraceContext(ctx context.Context, headers http.Header) {
	span := SpanFromContext(ctx)
	if span == nil {
		return
	}
	traceparent := fmt.Sprintf("00-%s-%s-01", span.TraceID, span.SpanID)
	headers.Set("traceparent", traceparent)
}

func ExtractTraceContext(ctx context.Context, headers http.Header) context.Context {
	traceparent := headers.Get("traceparent")
	if traceparent == "" {
		return ctx
	}

	parts := strings.Split(traceparent, "-")
	if len(parts) < 4 {
		return ctx
	}

	traceBytes, err := hex.DecodeString(parts[1])
	if err != nil || len(traceBytes) != 16 {
		return ctx
	}
	spanBytes, err := hex.DecodeString(parts[2])
	if err != nil || len(spanBytes) != 8 {
		return ctx
	}

	// Create a remote parent span
	remoteSpan := &Span{}
	copy(remoteSpan.TraceID[:], traceBytes)
	copy(remoteSpan.SpanID[:], spanBytes)

	return ContextWithSpan(ctx, remoteSpan)
}

// ---------- Middleware ----------

type responseWriter struct {
	http.ResponseWriter
	statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func TraceMiddleware(tracer *Tracer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Extract incoming trace context
			ctx := ExtractTraceContext(r.Context(), r.Header)

			// Start server span
			spanName := fmt.Sprintf("%s %s", r.Method, r.URL.Path)
			ctx, span := tracer.StartSpan(ctx, spanName, SpanKindServer)

			span.SetAttribute("http.method", r.Method)
			span.SetAttribute("http.url", r.URL.String())
			span.SetAttribute("http.host", r.Host)

			wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

			next.ServeHTTP(wrapped, r.WithContext(ctx))

			span.SetAttribute("http.status_code", wrapped.statusCode)
			if wrapped.statusCode >= 400 {
				span.SetStatus(StatusError, fmt.Sprintf("HTTP %d", wrapped.statusCode))
			} else {
				span.SetStatus(StatusOK, "")
			}
			span.End()
		})
	}
}

// ---------- Tracing HTTP Client ----------

type TracingClient struct {
	base   *http.Client
	tracer *Tracer
}

func NewTracingClient(tracer *Tracer) *TracingClient {
	return &TracingClient{
		base:   &http.Client{Timeout: 10 * time.Second},
		tracer: tracer,
	}
}

func (c *TracingClient) Get(ctx context.Context, url string) (*http.Response, error) {
	spanName := fmt.Sprintf("HTTP GET %s", url)
	ctx, span := c.tracer.StartSpan(ctx, spanName, SpanKindClient)
	defer span.End()

	span.SetAttribute("http.method", "GET")
	span.SetAttribute("http.url", url)

	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		span.SetStatus(StatusError, err.Error())
		return nil, err
	}

	// Inject trace context into outgoing request
	InjectTraceContext(ctx, req.Header)

	resp, err := c.base.Do(req)
	if err != nil {
		span.SetStatus(StatusError, err.Error())
		return nil, err
	}

	span.SetAttribute("http.status_code", resp.StatusCode)
	if resp.StatusCode >= 400 {
		span.SetStatus(StatusError, fmt.Sprintf("HTTP %d", resp.StatusCode))
	} else {
		span.SetStatus(StatusOK, "")
	}
	return resp, nil
}

// ============================================================
// SERVICE IMPLEMENTATIONS
// ============================================================

// ---------- User Service (port 8082) ----------

func startUserService(tracer *Tracer) *http.Server {
	mux := http.NewServeMux()

	mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		// Simulate database query
		_, dbSpan := tracer.StartSpan(ctx, "db.Query SELECT users", SpanKindClient)
		dbSpan.SetAttribute("db.system", "postgresql")
		dbSpan.SetAttribute("db.statement", "SELECT * FROM users WHERE id = $1")
		time.Sleep(8 * time.Millisecond) // Simulate DB latency
		dbSpan.SetStatus(StatusOK, "")
		dbSpan.End()

		userID := strings.TrimPrefix(r.URL.Path, "/users/")
		resp := map[string]any{
			"id":    userID,
			"name":  "Alice Johnson",
			"email": "alice@example.com",
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(resp)
	})

	handler := TraceMiddleware(tracer)(mux)
	server := &http.Server{Addr: ":8082", Handler: handler}
	go func() {
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Printf("User service error: %v", err)
		}
	}()
	return server
}

// ---------- Order Service (port 8081) ----------

func startOrderService(tracer *Tracer) *http.Server {
	client := NewTracingClient(tracer)
	mux := http.NewServeMux()

	mux.HandleFunc("/orders/", func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		orderID := strings.TrimPrefix(r.URL.Path, "/orders/")

		// Step 1: Check cache
		_, cacheSpan := tracer.StartSpan(ctx, "cache.Get", SpanKindClient)
		cacheSpan.SetAttribute("cache.system", "redis")
		cacheSpan.SetAttribute("cache.key", "order:"+orderID)
		time.Sleep(1 * time.Millisecond)
		cacheSpan.SetAttribute("cache.hit", false)
		cacheSpan.SetStatus(StatusOK, "")
		cacheSpan.End()

		// Step 2: Query database
		_, dbSpan := tracer.StartSpan(ctx, "db.Query SELECT orders", SpanKindClient)
		dbSpan.SetAttribute("db.system", "postgresql")
		dbSpan.SetAttribute("db.statement", "SELECT * FROM orders WHERE id = $1")
		time.Sleep(15 * time.Millisecond) // Simulate DB latency
		dbSpan.SetStatus(StatusOK, "")
		dbSpan.End()

		// Step 3: Call User Service to get user details
		userResp, err := client.Get(ctx, "http://localhost:8082/users/42")
		if err != nil {
			http.Error(w, "user service unavailable", http.StatusBadGateway)
			return
		}
		defer userResp.Body.Close()
		var user map[string]any
		json.NewDecoder(userResp.Body).Decode(&user)

		// Step 4: Build response
		resp := map[string]any{
			"id":     orderID,
			"status": "shipped",
			"total":  99.99,
			"user":   user,
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(resp)
	})

	handler := TraceMiddleware(tracer)(mux)
	server := &http.Server{Addr: ":8081", Handler: handler}
	go func() {
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Printf("Order service error: %v", err)
		}
	}()
	return server
}

// ---------- API Gateway (port 8080) ----------

func startAPIGateway(tracer *Tracer) *http.Server {
	client := NewTracingClient(tracer)
	mux := http.NewServeMux()

	mux.HandleFunc("/api/orders/", func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		// Step 1: Validate auth token
		_, authSpan := tracer.StartSpan(ctx, "auth.ValidateToken", SpanKindInternal)
		authSpan.SetAttribute("auth.method", "JWT")
		time.Sleep(3 * time.Millisecond)
		authSpan.SetStatus(StatusOK, "")
		authSpan.End()

		// Step 2: Call Order Service
		orderID := strings.TrimPrefix(r.URL.Path, "/api/orders/")
		orderResp, err := client.Get(ctx, "http://localhost:8081/orders/"+orderID)
		if err != nil {
			http.Error(w, "order service unavailable", http.StatusBadGateway)
			return
		}
		defer orderResp.Body.Close()

		// Forward the response
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(orderResp.StatusCode)
		io.Copy(w, orderResp.Body)
	})

	handler := TraceMiddleware(tracer)(mux)
	server := &http.Server{Addr: ":8080", Handler: handler}
	go func() {
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Printf("API Gateway error: %v", err)
		}
	}()
	return server
}

// ============================================================
// MAIN
// ============================================================

func main() {
	// Create separate tracers for each service
	// In production, each service runs in its own process with its own tracer.
	// Here we run them all in one process for demonstration.
	gatewayTracer := NewTracer("api-gateway")
	orderTracer := NewTracer("order-service")
	userTracer := NewTracer("user-service")

	// Start all three services
	fmt.Println("Starting services...")
	userServer := startUserService(userTracer)
	orderServer := startOrderService(orderTracer)
	gatewayServer := startAPIGateway(gatewayTracer)

	// Wait for services to start
	time.Sleep(100 * time.Millisecond)
	fmt.Println("All services started.")
	fmt.Println()

	// Make a request to the API Gateway
	fmt.Println("=== Making Request: GET /api/orders/123 ===")
	fmt.Println()

	resp, err := http.Get("http://localhost:8080/api/orders/123")
	if err != nil {
		log.Fatalf("Request failed: %v", err)
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Println()
	fmt.Println("=== Response ===")
	fmt.Printf("Status: %d\n", resp.StatusCode)

	var prettyJSON map[string]any
	json.Unmarshal(body, &prettyJSON)
	formatted, _ := json.MarshalIndent(prettyJSON, "", "  ")
	fmt.Println(string(formatted))

	// Wait a moment for all spans to be recorded
	time.Sleep(50 * time.Millisecond)

	// Collect all spans from all tracers
	fmt.Println()
	fmt.Println("=== Trace Summary ===")

	var allSpans []*Span
	allSpans = append(allSpans, gatewayTracer.Spans()...)
	allSpans = append(allSpans, orderTracer.Spans()...)
	allSpans = append(allSpans, userTracer.Spans()...)

	fmt.Printf("Total spans: %d\n", len(allSpans))
	fmt.Printf("Services: api-gateway (%d), order-service (%d), user-service (%d)\n",
		len(gatewayTracer.Spans()), len(orderTracer.Spans()), len(userTracer.Spans()))

	if len(allSpans) > 0 {
		fmt.Printf("Trace ID: %s\n", allSpans[0].TraceID)
	}

	// Shutdown all services
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	gatewayServer.Shutdown(ctx)
	orderServer.Shutdown(ctx)
	userServer.Shutdown(ctx)
}
```

To run this example:

```bash
go run main.go
```

You will see output showing the trace propagating across all three services:

```
Starting services...
All services started.

=== Making Request: GET /api/orders/123 ===

[api-gateway]    auth.ValidateToken                  trace=abc123def4.. span=... parent=...           3ms OK
[order-service]  cache.Get                           trace=abc123def4.. span=... parent=...           1ms OK
[order-service]  db.Query SELECT orders               trace=abc123def4.. span=... parent=...          15ms OK
[user-service]   db.Query SELECT users                trace=abc123def4.. span=... parent=...           8ms OK
[user-service]   GET /users/42                        trace=abc123def4.. span=... parent=...          10ms OK
[order-service]  HTTP GET http://localhost:8082/us... trace=abc123def4.. span=... parent=...          12ms OK
[order-service]  GET /orders/123                      trace=abc123def4.. span=... parent=...          32ms OK
[api-gateway]    HTTP GET http://localhost:8081/or... trace=abc123def4.. span=... parent=...          35ms OK
[api-gateway]    GET /api/orders/123                  trace=abc123def4.. span=... parent=...          40ms OK

=== Response ===
Status: 200
{
  "id": "123",
  "status": "shipped",
  "total": 99.99,
  "user": {
    "email": "alice@example.com",
    "id": "42",
    "name": "Alice Johnson"
  }
}

=== Trace Summary ===
Total spans: 9
Services: api-gateway (3), order-service (4), user-service (2)
Trace ID: abc123def456...
```

Notice that every span shares the same Trace ID, even though they are created by different tracers in different services. This is the power of distributed tracing -- a single request, visible across every service it touches.

---

## 16. Key Takeaways

1. **A trace is a tree of spans.** Every span has a TraceID (shared by the whole trace) and a SpanID (unique to that span). Parent-child relationships form the tree.

2. **Context is the backbone of tracing in Go.** `context.Context` carries the current span through function calls. Always pass context. Never use `context.Background()` inside a traced function.

3. **The W3C Trace Context standard** defines the `traceparent` header (`00-{traceID}-{spanID}-{flags}`). It ensures interoperability between different tracing systems.

4. **Propagation has two sides**: injection (adding headers to outgoing requests) and extraction (reading headers from incoming requests). Without propagation, each service sees isolated traces.

5. **Batch exporters are essential for production.** Exporting each span individually creates too much overhead. Batch them with size and time limits, and handle backpressure by dropping spans when the queue is full.

6. **Sampling controls cost.** Head-based sampling (probability, rate-limiting, parent-based) decides at trace start. Tail-based sampling waits until the trace completes and keeps interesting traces (errors, high latency).

7. **The critical path** determines total request latency. Optimizing spans not on the critical path does not reduce end-to-end latency. Trace analysis should start with critical path identification.

8. **Middleware handles 80% of instrumentation.** HTTP server middleware creates server spans. HTTP client transport wrappers create client spans. Database wrappers create client spans. Most code never touches the tracer directly.

9. **Spans should follow OpenTelemetry semantic conventions** for attribute naming (`http.method`, `db.system`, `db.statement`, etc.). This ensures your traces are queryable in any backend.

10. **Error recording should be rich.** Record the error type, message, stack trace, and chain of wrapped errors. Use span events for exceptions, not just status codes.

11. **Goroutine tracing requires explicit context passing.** Go's goroutines do not inherit context automatically. When you `go func()`, you must capture the context variable in the closure.

12. **N+1 queries become obvious with tracing.** When you see 25 identical database spans under a single parent, the N+1 problem is immediately visible without any code analysis.

13. **Service dependency maps emerge from traces.** By analyzing parent-child relationships across service boundaries, you can automatically discover which services depend on which.

14. **The tracer, exporter, and sampler are the three pillars.** The tracer creates spans. The exporter sends them somewhere. The sampler decides which traces to keep. Everything else (middleware, propagators, analyzers) builds on these three.

---

## 17. Practice Exercises

### Exercise 1: Add gRPC Propagation

Extend the propagation system from Section 7 to support gRPC metadata. Create a `GRPCMetadataCarrier` that implements the `Carrier` interface by reading and writing gRPC metadata. Test it with a simple gRPC server and client.

### Exercise 2: Build a Probabilistic Tail Sampler

Extend the tail-based sampler from Section 6 with a combined policy: keep 100% of error traces, keep 100% of traces slower than 500ms, and keep 10% of normal traces. Implement this as a `CombinedTailPolicy` that takes a probability sampler as a fallback.

### Exercise 3: Span Attribute Sanitization

Database span attributes can accidentally include sensitive data (passwords, tokens, PII). Build an `AttributeSanitizer` that wraps a `SpanExporter` and redacts sensitive values based on configurable rules. For example, any attribute containing "password", "token", or "secret" in its key should have its value replaced with `[REDACTED]`.

### Exercise 4: Distributed Trace Assembler

Build a `TraceAssembler` that collects spans from multiple services (simulated as separate in-memory stores) and assembles complete traces. It should:
- Detect incomplete traces (missing spans)
- Calculate total trace duration
- Identify orphan spans (spans whose parent is not in the trace)
- Return a `TraceReport` with all this metadata

### Exercise 5: Latency Histogram per Operation

Build a `LatencyHistogram` that aggregates span durations by operation name across many traces. It should compute P50, P90, P95, and P99 latencies. Use this to build a simple "service dashboard" that shows the slowest operations.

### Exercise 6: Add a Zipkin Exporter

Implement a `ZipkinExporter` that formats spans in the Zipkin v2 JSON format and sends them to a Zipkin server. The Zipkin format is different from OTLP -- research the format at https://zipkin.io/zipkin-api/ and implement the conversion.

### Exercise 7: Context Deadline Tracing

Build middleware that automatically records context deadline/timeout information in spans. When a context has a deadline, the span should include attributes `context.deadline`, `context.timeout_ms`, and an event `context.deadline.exceeded` if the deadline is exceeded during the span's lifetime.

### Exercise 8: Build a Live Trace Dashboard

Create a terminal-based dashboard (using ANSI escape codes) that shows:
- Active traces (in-progress)
- Recent completed traces with their duration
- Error rate per service
- Throughput (traces per second)

Update the display in real-time as spans are exported.

### Exercise 9: Trace-Based Testing

Build a test helper that captures all spans created during a test and provides assertion methods:
```go
func TestOrderHandler(t *testing.T) {
    recorder := tracetest.NewRecorder()
    tracer := NewTracer("test", recorder)
    // ... run test ...
    recorder.AssertSpanExists(t, "db.Query SELECT orders")
    recorder.AssertNoErrors(t)
    recorder.AssertMaxDuration(t, "cache.Get", 10*time.Millisecond)
}
```

### Exercise 10: Implement Baggage Propagation

W3C Baggage (https://www.w3.org/TR/baggage/) is a companion standard to Trace Context that propagates arbitrary key-value pairs across services. Implement a `BaggagePropagator` that reads and writes the `baggage` header, and integrate it with the tracer so that baggage is automatically propagated alongside trace context.
