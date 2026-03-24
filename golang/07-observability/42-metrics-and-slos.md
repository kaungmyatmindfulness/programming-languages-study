# Chapter 42: Metrics & SLOs

Production systems are not binary. They do not simply "work" or "not work." They exist on a spectrum of health -- responding a little slower under load, dropping a fraction of a percent of requests during a deployment, consuming more memory as the cache warms. Logs tell you what happened to individual requests. Traces tell you why a specific request was slow. Metrics tell you what is happening to your system as a whole, right now, and over time. They are the vital signs of your software.

This chapter covers application metrics and Service Level Objectives (SLOs) in Go, drawing from the principles in "Observability Engineering" by Charity Majors, Liz Fong-Jones, and George Miranda (Chapters 13-18). We will instrument a Go service with Prometheus, implement the RED and USE monitoring methods, define SLIs and SLOs in code, build error budget tracking, and wire up multi-window burn rate alerting. Every example is complete, runnable Go code that you can copy into a project and start using immediately.

If Chapter 28 gave you structured logging (the microscope), this chapter gives you metrics and SLOs (the dashboard and the contract). Together, they form the foundation of production observability.

---

## Prerequisites

Before diving into this chapter, you should be comfortable with:

- **Chapter 12: JSON, HTTP, and REST APIs** -- All examples use HTTP servers and handlers
- **Chapter 16: Context Package** -- Metrics collection uses context for request-scoped data
- **Chapter 25: Middleware Patterns** -- We build metrics middleware for HTTP handlers
- **Chapter 28: Structured Logging & Observability** -- This chapter extends the observability story
- **Chapter 30: Graceful Shutdown & Production Patterns** -- Production services need both metrics and clean shutdown

You should also have:

- Go 1.21 or later installed
- Basic familiarity with HTTP status codes and latency concepts
- A conceptual understanding of what Prometheus is (a time-series database that scrapes metrics endpoints)

---

## Table of Contents

1. [Why Metrics?](#1-why-metrics)
2. [Metric Types](#2-metric-types)
3. [Prometheus Client in Go](#3-prometheus-client-in-go)
4. [Instrumenting HTTP Handlers](#4-instrumenting-http-handlers)
5. [Custom Business Metrics](#5-custom-business-metrics)
6. [Database Metrics](#6-database-metrics)
7. [Go Runtime Metrics](#7-go-runtime-metrics)
8. [The /metrics Endpoint](#8-the-metrics-endpoint)
9. [RED Method](#9-red-method)
10. [USE Method](#10-use-method)
11. [SLI (Service Level Indicators)](#11-sli-service-level-indicators)
12. [SLO (Service Level Objectives)](#12-slo-service-level-objectives)
13. [Error Budget Policies](#13-error-budget-policies)
14. [Multi-Window Burn Rate Alerts](#14-multi-window-burn-rate-alerts)
15. [Building an SLO Dashboard](#15-building-an-slo-dashboard)
16. [Real-World Example: Fully Instrumented API with SLOs](#16-real-world-example-fully-instrumented-api-with-slos)
17. [Key Takeaways](#17-key-takeaways)
18. [Practice Exercises](#18-practice-exercises)

---

## 1. Why Metrics?

### The Limits of Logs

In Chapter 28, we built a complete structured logging system. Logs are indispensable for debugging individual requests, but they have fundamental limitations when you need to understand system-wide behavior:

```
Question: "What is the P99 latency of our /api/orders endpoint over the last hour?"
```

To answer this with logs, you would need to:
1. Query all log lines for that endpoint in the last hour (potentially millions)
2. Extract the duration field from each line
3. Sort all durations and compute the 99th percentile
4. Repeat this every time someone asks

This is expensive, slow, and does not scale. Metrics solve this problem by pre-aggregating data at collection time.

### What Metrics Give You

Metrics are numeric measurements collected at regular intervals. They answer questions about system behavior over time:

| Question | Metric Type |
|----------|-------------|
| How many requests per second are we serving? | Counter (rate) |
| What is the current memory usage? | Gauge |
| What is the P95 request latency? | Histogram |
| How many database connections are in use? | Gauge |
| What percentage of requests are errors? | Counter (ratio) |
| Are we meeting our latency targets? | SLI/SLO |

### The Three Pillars

Observability rests on three pillars, each serving a different purpose:

```
Logs    → What happened to THIS request?
Traces  → Why was THIS request slow?
Metrics → What is happening to the SYSTEM?
```

Metrics are unique because they are pre-aggregated. A histogram metric tracking request duration does not store every individual request time -- it stores counts in buckets (e.g., "how many requests took 0-10ms, 10-50ms, 50-100ms, ..."). This means:

- **Constant storage**: A metric takes the same amount of space whether you serve 10 or 10 million requests per second
- **Fast queries**: Computing P99 latency is a simple bucket lookup, not a sort over millions of values
- **Long retention**: You can store years of metrics data affordably
- **Real-time alerting**: Prometheus can evaluate alert rules every 15 seconds

### Metrics vs Events

"Observability Engineering" draws a clear distinction between metrics and events. Events are individual occurrences with arbitrary dimensions (what we capture in logs and traces). Metrics are aggregated summaries of events over time. Both are necessary:

- Use **events** (logs/traces) when you need to debug a specific incident
- Use **metrics** when you need to understand trends, set alerts, or define SLOs

### A Simple Mental Model

Think of metrics as the dashboard of a car. You do not need to know the exact state of every piston to drive safely -- you need a speedometer, fuel gauge, temperature gauge, and engine warning light. Metrics are the speedometer; logs are the OBD-II diagnostic port.

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// SimpleMetric demonstrates the concept of a metric: a named numeric
// value that changes over time and can be aggregated.
type SimpleMetric struct {
	Name   string
	Values []float64
}

func (m *SimpleMetric) Record(v float64) {
	m.Values = append(m.Values, v)
}

func (m *SimpleMetric) Count() int {
	return len(m.Values)
}

func (m *SimpleMetric) Sum() float64 {
	var sum float64
	for _, v := range m.Values {
		sum += v
	}
	return sum
}

func (m *SimpleMetric) Average() float64 {
	if len(m.Values) == 0 {
		return 0
	}
	return m.Sum() / float64(len(m.Values))
}

func (m *SimpleMetric) Percentile(p float64) float64 {
	if len(m.Values) == 0 {
		return 0
	}
	// Simple percentile: sort a copy and pick the value at position p*len
	sorted := make([]float64, len(m.Values))
	copy(sorted, m.Values)
	// Insertion sort for simplicity
	for i := 1; i < len(sorted); i++ {
		key := sorted[i]
		j := i - 1
		for j >= 0 && sorted[j] > key {
			sorted[j+1] = sorted[j]
			j--
		}
		sorted[j+1] = key
	}
	idx := int(p * float64(len(sorted)-1))
	return sorted[idx]
}

func main() {
	latency := &SimpleMetric{Name: "request_duration_ms"}

	// Simulate 1000 requests with varying latencies
	for i := 0; i < 1000; i++ {
		// Most requests: 10-50ms. Some: 100-500ms. A few: 1000-3000ms.
		var duration float64
		r := rand.Float64()
		switch {
		case r < 0.90:
			duration = 10 + rand.Float64()*40
		case r < 0.98:
			duration = 100 + rand.Float64()*400
		default:
			duration = 1000 + rand.Float64()*2000
		}
		latency.Record(duration)
	}

	fmt.Printf("Metric: %s\n", latency.Name)
	fmt.Printf("  Count:   %d\n", latency.Count())
	fmt.Printf("  Average: %.2fms\n", latency.Average())
	fmt.Printf("  P50:     %.2fms\n", latency.Percentile(0.50))
	fmt.Printf("  P90:     %.2fms\n", latency.Percentile(0.90))
	fmt.Printf("  P95:     %.2fms\n", latency.Percentile(0.95))
	fmt.Printf("  P99:     %.2fms\n", latency.Percentile(0.99))
	fmt.Printf("  Rate:    %.0f req/sec (simulated over 1s)\n", float64(latency.Count())/1.0)

	_ = time.Now() // suppress unused import
}
```

Output (values will vary):
```
Metric: request_duration_ms
  Count:   1000
  Average: 72.34ms
  P50:     28.51ms
  P90:     48.12ms
  P95:     287.44ms
  P99:     2134.67ms
```

Notice the gap between average (72ms) and P99 (2134ms). The average hides the pain of the slowest 1% of users. This is why percentile metrics matter, and it is one of the core insights from "Observability Engineering."

---

## 2. Metric Types

Prometheus defines four core metric types. Understanding when to use each is essential for effective instrumentation.

### Counter

A counter is a cumulative value that only goes up (or resets to zero on process restart). Use counters for things you count: requests served, errors encountered, bytes transferred.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

// Counter is a monotonically increasing value.
// It only goes up. On process restart, it resets to zero.
// Prometheus handles the reset detection automatically.
type Counter struct {
	name  string
	value atomic.Int64
}

func NewCounter(name string) *Counter {
	return &Counter{name: name}
}

func (c *Counter) Inc() {
	c.value.Add(1)
}

func (c *Counter) Add(delta int64) {
	if delta < 0 {
		panic("counter cannot decrease")
	}
	c.value.Add(delta)
}

func (c *Counter) Value() int64 {
	return c.value.Load()
}

func main() {
	requestsTotal := NewCounter("http_requests_total")
	errorsTotal := NewCounter("http_errors_total")

	// Simulate traffic
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			requestsTotal.Inc()
			if i%10 == 0 { // 10% error rate
				errorsTotal.Inc()
			}
		}(i)
	}
	wg.Wait()

	fmt.Printf("%s = %d\n", requestsTotal.name, requestsTotal.Value())
	fmt.Printf("%s = %d\n", errorsTotal.name, errorsTotal.Value())
	fmt.Printf("Error rate: %.1f%%\n",
		float64(errorsTotal.Value())/float64(requestsTotal.Value())*100)
}
```

**When to use a Counter:**
- Total HTTP requests
- Total errors
- Total bytes sent/received
- Total items processed
- Total database queries

**Key insight:** You never query a counter's raw value directly. You always query its *rate* -- how fast it is increasing. In Prometheus, `rate(http_requests_total[5m])` gives you requests per second over the last 5 minutes.

### Gauge

A gauge is a value that can go up and down. Use gauges for things you measure at a point in time: current temperature, current memory usage, number of active goroutines.

```go
package main

import (
	"fmt"
	"math/rand"
	"sync/atomic"
	"time"
)

// Gauge is a value that can go up and down.
// It represents a snapshot of some current state.
type Gauge struct {
	name  string
	value atomic.Int64
}

func NewGauge(name string) *Gauge {
	return &Gauge{name: name}
}

func (g *Gauge) Set(v int64) {
	g.value.Store(v)
}

func (g *Gauge) Inc() {
	g.value.Add(1)
}

func (g *Gauge) Dec() {
	g.value.Add(-1)
}

func (g *Gauge) Value() int64 {
	return g.value.Load()
}

func main() {
	activeConns := NewGauge("active_connections")
	queueDepth := NewGauge("job_queue_depth")

	// Simulate connections opening and closing
	for i := 0; i < 20; i++ {
		if rand.Float64() < 0.7 {
			activeConns.Inc()
		} else {
			activeConns.Dec()
		}
		// Simulate queue depth fluctuating
		queueDepth.Set(int64(rand.Intn(100)))
	}

	fmt.Printf("%s = %d\n", activeConns.name, activeConns.Value())
	fmt.Printf("%s = %d\n", queueDepth.name, queueDepth.Value())

	_ = time.Now() // suppress unused import
}
```

**When to use a Gauge:**
- Active connections/goroutines
- Queue depth
- Memory usage
- CPU temperature
- Number of items in a cache
- Disk space remaining

**Common mistake:** Using a gauge when you should use a counter. If you are tracking "total requests," use a counter, not a gauge. Gauges can miss spikes between scrapes -- if the value jumps from 0 to 100 and back to 0 between two Prometheus scrapes (every 15 seconds), you will never see it.

### Histogram

A histogram tracks the distribution of values. It pre-sorts observations into configurable buckets. Use histograms for things where you care about percentiles: request latency, response size, queue wait time.

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"strings"
)

// Histogram divides observations into buckets to track distribution.
// Each bucket counts observations less than or equal to its upper bound.
// Buckets are cumulative: the +Inf bucket always equals the total count.
type Histogram struct {
	name    string
	buckets []float64 // upper bounds
	counts  []int64   // count for each bucket
	sum     float64
	count   int64
}

func NewHistogram(name string, buckets []float64) *Histogram {
	return &Histogram{
		name:    name,
		buckets: buckets,
		counts:  make([]int64, len(buckets)),
	}
}

func (h *Histogram) Observe(v float64) {
	h.sum += v
	h.count++
	for i, bound := range h.buckets {
		if v <= bound {
			h.counts[i]++
		}
	}
}

func (h *Histogram) String() string {
	var sb strings.Builder
	fmt.Fprintf(&sb, "%s (count=%d, sum=%.2f, avg=%.2f)\n",
		h.name, h.count, h.sum, h.sum/float64(h.count))
	for i, bound := range h.buckets {
		bar := strings.Repeat("=", int(float64(h.counts[i])/float64(h.count)*50))
		fmt.Fprintf(&sb, "  le=%.3f: %6d %s\n", bound, h.counts[i], bar)
	}
	fmt.Fprintf(&sb, "  le=+Inf:  %6d\n", h.count)
	return sb.String()
}

// DefaultBuckets mirrors Prometheus default histogram buckets.
var DefaultBuckets = []float64{
	0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10,
}

func main() {
	latency := NewHistogram("http_request_duration_seconds", DefaultBuckets)

	// Simulate 10000 requests
	for i := 0; i < 10000; i++ {
		var duration float64
		r := rand.Float64()
		switch {
		case r < 0.70:
			duration = 0.005 + rand.Float64()*0.020 // 5-25ms
		case r < 0.90:
			duration = 0.025 + rand.Float64()*0.075 // 25-100ms
		case r < 0.98:
			duration = 0.1 + rand.Float64()*0.4 // 100-500ms
		default:
			duration = 0.5 + rand.Float64()*4.5 // 500ms-5s
		}
		latency.Observe(duration)
	}

	fmt.Println(latency)

	// Estimate P50 and P99 from buckets
	// This is how Prometheus computes histogram_quantile()
	fmt.Println("Estimated percentiles from histogram buckets:")
	for _, p := range []float64{0.50, 0.90, 0.95, 0.99} {
		v := estimateQuantile(latency, p)
		fmt.Printf("  P%.0f: %.3fs (%.1fms)\n", p*100, v, v*1000)
	}

	_ = math.Inf(1) // suppress unused import
}

// estimateQuantile estimates a quantile from histogram buckets
// using linear interpolation, the same way Prometheus does.
func estimateQuantile(h *Histogram, q float64) float64 {
	target := q * float64(h.count)
	prevCount := int64(0)
	prevBound := 0.0

	for i, bound := range h.buckets {
		if float64(h.counts[i]) >= target {
			// Linear interpolation within this bucket
			fraction := (target - float64(prevCount)) / float64(h.counts[i]-prevCount)
			return prevBound + fraction*(bound-prevBound)
		}
		prevCount = h.counts[i]
		prevBound = bound
	}
	return h.buckets[len(h.buckets)-1]
}
```

Output (values will vary):
```
http_request_duration_seconds (count=10000, sum=289.45, avg=0.03)
  le=0.005:    704 ===
  le=0.010:   2331 ===========
  le=0.025:   6987 ==================================
  le=0.050:   7010 ==================================
  le=0.100:   8983 ============================================
  le=0.250:   9407 ===============================================
  le=0.500:   9805 ================================================
  le=1.000:   9868 =================================================
  le=2.500:   9947 =================================================
  le=5.000:  10000 ==================================================
  le=10.000: 10000 ==================================================
  le=+Inf:   10000

Estimated percentiles from histogram buckets:
  P50: 0.015s (14.6ms)
  P90: 0.087s (86.5ms)
  P95: 0.194s (193.9ms)
  P99: 1.256s (1255.8ms)
```

**When to use a Histogram:**
- Request latency (the most common use case)
- Response body size
- Queue wait time
- Batch job duration
- Any measurement where percentiles matter

**Bucket selection matters.** Prometheus default buckets (5ms to 10s) work well for HTTP request durations. For other use cases, choose buckets that align with your SLO targets. If your SLO says "P99 latency under 200ms," make sure you have a bucket boundary at or near 200ms.

### Summary

A summary is similar to a histogram but computes quantiles on the client side. Summaries are rarely used in modern Prometheus setups because they cannot be aggregated across instances. Prefer histograms in almost all cases.

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"sort"
)

// Summary computes exact quantiles on the client side.
// Unlike histograms, summaries cannot be aggregated across instances.
// This is why Prometheus documentation recommends histograms in most cases.
type Summary struct {
	name       string
	values     []float64
	quantiles  []float64
	maxSamples int
}

func NewSummary(name string, quantiles []float64, maxSamples int) *Summary {
	return &Summary{
		name:       name,
		quantiles:  quantiles,
		maxSamples: maxSamples,
	}
}

func (s *Summary) Observe(v float64) {
	s.values = append(s.values, v)
	if len(s.values) > s.maxSamples {
		// Drop oldest samples (simplistic windowing)
		s.values = s.values[len(s.values)-s.maxSamples:]
	}
}

func (s *Summary) Quantile(q float64) float64 {
	if len(s.values) == 0 {
		return math.NaN()
	}
	sorted := make([]float64, len(s.values))
	copy(sorted, s.values)
	sort.Float64s(sorted)
	idx := int(q * float64(len(sorted)-1))
	return sorted[idx]
}

func (s *Summary) Report() {
	fmt.Printf("%s (samples=%d)\n", s.name, len(s.values))
	for _, q := range s.quantiles {
		fmt.Printf("  quantile=%.2f: %.4f\n", q, s.Quantile(q))
	}
}

func main() {
	s := NewSummary("rpc_duration_seconds",
		[]float64{0.5, 0.9, 0.95, 0.99},
		10000,
	)

	for i := 0; i < 10000; i++ {
		duration := 0.01 + rand.ExpFloat64()*0.05
		s.Observe(duration)
	}

	s.Report()
}
```

**Histogram vs Summary -- which to use:**

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Aggregatable across instances? | Yes | No |
| Exact quantiles? | No (estimation from buckets) | Yes |
| Client CPU cost? | Low | Higher |
| Can define quantiles after collection? | Yes | No |
| Recommended by Prometheus? | Yes | Rarely |

**Rule of thumb:** Use histograms. Use summaries only when you need exact quantiles from a single instance and cannot tolerate the approximation from histogram buckets.

---

## 3. Prometheus Client in Go

The `prometheus/client_golang` library is the official Go client for Prometheus. It provides the four metric types, a default registry, and an HTTP handler for the `/metrics` endpoint.

### Installation

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
go get github.com/prometheus/client_golang/prometheus/promauto
```

### Defining Metrics

Metrics are typically defined as package-level variables. This is one of the few cases in Go where package-level mutable state is appropriate -- metrics are global by nature.

```go
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// Define metrics as package-level variables.
// Use descriptive names following the convention:
//   <namespace>_<subsystem>_<name>_<unit>
// Examples:
//   myapp_http_requests_total
//   myapp_http_request_duration_seconds
//   myapp_db_connections_active

var (
	// Counter: total HTTP requests, labeled by method, path, and status code.
	// Labels let you break down a single metric into dimensions.
	httpRequestsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "requests_total",
			Help:      "Total number of HTTP requests processed.",
		},
		[]string{"method", "path", "status"},
	)

	// Histogram: request duration in seconds, labeled by method and path.
	// Buckets are chosen to match our SLO targets.
	httpRequestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "request_duration_seconds",
			Help:      "HTTP request duration in seconds.",
			Buckets:   []float64{0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2.5, 5, 10},
		},
		[]string{"method", "path"},
	)

	// Gauge: number of currently active connections.
	activeConnections = prometheus.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "active_connections",
			Help:      "Number of currently active HTTP connections.",
		},
	)

	// Histogram: response size in bytes.
	httpResponseSize = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "response_size_bytes",
			Help:      "HTTP response size in bytes.",
			Buckets:   prometheus.ExponentialBuckets(100, 10, 6), // 100, 1K, 10K, 100K, 1M, 10M
		},
		[]string{"method", "path"},
	)
)

func init() {
	// Register all metrics with the default Prometheus registry.
	// MustRegister panics if registration fails (duplicate metric name, etc).
	// This is intentional -- a registration failure is a programmer error
	// and should be caught at startup, not at runtime.
	prometheus.MustRegister(
		httpRequestsTotal,
		httpRequestDuration,
		activeConnections,
		httpResponseSize,
	)
}

func main() {
	// Handlers
	http.HandleFunc("/api/hello", func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		activeConnections.Inc()
		defer activeConnections.Dec()

		// Simulate work
		time.Sleep(time.Duration(10+time.Now().UnixNano()%50) * time.Millisecond)

		w.WriteHeader(http.StatusOK)
		body := []byte(`{"message": "hello, world"}`)
		w.Write(body)

		duration := time.Since(start).Seconds()
		httpRequestsTotal.WithLabelValues(r.Method, "/api/hello", "200").Inc()
		httpRequestDuration.WithLabelValues(r.Method, "/api/hello").Observe(duration)
		httpResponseSize.WithLabelValues(r.Method, "/api/hello").Observe(float64(len(body)))
	})

	// Expose metrics at /metrics for Prometheus to scrape
	http.Handle("/metrics", promhttp.Handler())

	fmt.Println("Server listening on :8080")
	fmt.Println("  GET /api/hello   - sample endpoint")
	fmt.Println("  GET /metrics     - Prometheus metrics")
	http.ListenAndServe(":8080", nil)
}
```

### Using promauto for Automatic Registration

If you find the `init()` + `MustRegister()` pattern verbose, `promauto` registers metrics automatically:

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// promauto.NewCounterVec automatically registers with the default registry.
// No init() or MustRegister() needed.
var (
	requestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "myapp_requests_total",
			Help: "Total requests by method and status.",
		},
		[]string{"method", "status"},
	)

	requestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "myapp_request_duration_seconds",
			Help:    "Request duration in seconds.",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"method"},
	)
)

func main() {
	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("Metrics server on :2112")
	http.ListenAndServe(":2112", nil)
}
```

### Custom Registry

For testing or when running multiple metric sources in the same process, use a custom registry instead of the default global one:

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	// Create a custom registry -- does NOT include default Go runtime metrics.
	registry := prometheus.NewRegistry()

	// Create metrics
	requestsTotal := prometheus.NewCounter(prometheus.CounterOpts{
		Name: "custom_requests_total",
		Help: "Total requests in custom registry.",
	})

	// Register with our custom registry, not the default one
	registry.MustRegister(requestsTotal)

	// Optionally add default Go collector and process collector
	registry.MustRegister(prometheus.NewGoCollector())
	registry.MustRegister(prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{}))

	// Handler for our custom registry
	http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{
		EnableOpenMetrics: true,
	}))

	fmt.Println("Custom registry metrics on :2112")
	http.ListenAndServe(":2112", nil)
}
```

### Label Cardinality -- The Most Common Mistake

Labels are powerful but dangerous. Every unique combination of label values creates a new time series in Prometheus. This is called **cardinality**.

```go
package main

import "fmt"

func main() {
	// GOOD: Low cardinality labels
	// method: GET, POST, PUT, DELETE, PATCH (5 values)
	// status: 200, 201, 400, 401, 403, 404, 500 (7 values)
	// path: /api/users, /api/orders, /api/products (10 values)
	// Total time series: 5 * 7 * 10 = 350 -- perfectly fine

	// BAD: High cardinality labels
	// user_id: could be millions of unique values
	// request_id: unique per request -- infinite cardinality
	// ip_address: up to 4 billion IPv4 addresses
	// Total time series: MILLIONS -- will crash Prometheus

	// RULE OF THUMB:
	// - Labels with fewer than ~100 unique values: SAFE
	// - Labels with 100-1000 unique values: BE CAREFUL
	// - Labels with 1000+ unique values: DANGER -- use logs instead

	// If you need per-user metrics, use logs/traces, not metric labels.
	// Metrics are for aggregate behavior. Logs are for individual events.

	fmt.Println("Label cardinality rules:")
	fmt.Println("  method, status, endpoint: GOOD (low cardinality)")
	fmt.Println("  user_id, request_id, ip: BAD (high cardinality)")
	fmt.Println("  region, datacenter, version: GOOD (low cardinality)")
	fmt.Println("  email, session_token: BAD (high cardinality)")
}
```

---

## 4. Instrumenting HTTP Handlers

The most common instrumentation pattern is HTTP middleware that automatically records request metrics for every handler.

### Response Writer Wrapper

To capture the HTTP status code and response size, we need to wrap `http.ResponseWriter`:

```go
package main

import (
	"fmt"
	"net/http"
)

// responseWriter wraps http.ResponseWriter to capture the status code
// and bytes written. This is necessary because the standard ResponseWriter
// does not expose the status code after WriteHeader is called.
type responseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
	wroteHeader  bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK, // default if WriteHeader is never called
	}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.wroteHeader {
		return // prevent double WriteHeader calls
	}
	rw.statusCode = code
	rw.wroteHeader = true
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.wroteHeader {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// Flush implements http.Flusher if the underlying writer supports it.
// This is important for streaming responses and Server-Sent Events.
func (rw *responseWriter) Flush() {
	if f, ok := rw.ResponseWriter.(http.Flusher); ok {
		f.Flush()
	}
}

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
		w.Write([]byte(`{"id": 42}`))
	})

	// Wrap and call
	recorder := newResponseWriter(nil) // nil for demo, would be a real ResponseWriter
	_ = handler
	_ = recorder

	fmt.Println("responseWriter wraps http.ResponseWriter to capture:")
	fmt.Println("  - Status code (default 200 if WriteHeader not called)")
	fmt.Println("  - Bytes written")
	fmt.Println("  - Whether header was already written (prevents double writes)")
}
```

### Complete Metrics Middleware

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// --- Metrics ---

var (
	httpRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "requests_total",
			Help:      "Total HTTP requests.",
		},
		[]string{"method", "path", "status"},
	)

	httpRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "request_duration_seconds",
			Help:      "HTTP request duration in seconds.",
			Buckets:   []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2.5, 5},
		},
		[]string{"method", "path"},
	)

	httpResponseSize = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "response_size_bytes",
			Help:      "HTTP response size in bytes.",
			Buckets:   prometheus.ExponentialBuckets(100, 10, 6),
		},
		[]string{"method", "path"},
	)

	httpActiveRequests = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "http",
			Name:      "active_requests",
			Help:      "Number of currently active HTTP requests.",
		},
		[]string{"method"},
	)
)

// --- Response Writer Wrapper ---

type responseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
	wroteHeader  bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{
		ResponseWriter: w,
		statusCode:     http.StatusOK,
	}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.wroteHeader {
		return
	}
	rw.statusCode = code
	rw.wroteHeader = true
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.wroteHeader {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// --- Middleware ---

// MetricsMiddleware records HTTP metrics for every request.
// It wraps the response writer to capture status code and response size,
// and records timing, counts, and in-flight request gauges.
func MetricsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		// Normalize the path to prevent cardinality explosion.
		// In a real app, map actual paths to route patterns:
		//   /api/users/123 -> /api/users/:id
		path := normalizePath(r.URL.Path)

		// Track active requests
		httpActiveRequests.WithLabelValues(r.Method).Inc()
		defer httpActiveRequests.WithLabelValues(r.Method).Dec()

		// Wrap the response writer
		wrapped := newResponseWriter(w)

		// Call the next handler
		next.ServeHTTP(wrapped, r)

		// Record metrics after the handler completes
		duration := time.Since(start).Seconds()
		status := strconv.Itoa(wrapped.statusCode)

		httpRequestsTotal.WithLabelValues(r.Method, path, status).Inc()
		httpRequestDuration.WithLabelValues(r.Method, path).Observe(duration)
		httpResponseSize.WithLabelValues(r.Method, path).Observe(float64(wrapped.bytesWritten))
	})
}

// normalizePath maps concrete URL paths to route patterns.
// This prevents label cardinality explosion from path parameters.
// In production, your router framework can often provide the matched
// route pattern directly (e.g., chi.RouteContext, gorilla mux.CurrentRoute).
func normalizePath(path string) string {
	// Simple normalization -- in production, use your router's matched pattern.
	// This example just returns the path as-is for known routes.
	knownPaths := map[string]bool{
		"/api/users":    true,
		"/api/orders":   true,
		"/api/products": true,
		"/health":       true,
		"/metrics":      true,
	}
	if knownPaths[path] {
		return path
	}
	return "/other" // Catch-all for unrecognized paths
}

// --- Handlers ---

func usersHandler(w http.ResponseWriter, r *http.Request) {
	// Simulate variable latency
	time.Sleep(time.Duration(10+rand.Intn(90)) * time.Millisecond)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]`))
}

func ordersHandler(w http.ResponseWriter, r *http.Request) {
	time.Sleep(time.Duration(20+rand.Intn(180)) * time.Millisecond)

	// Simulate occasional errors
	if rand.Float64() < 0.05 {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte(`{"error":"database timeout"}`))
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`[{"id":1,"total":99.99}]`))
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status":"ok"}`))
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/api/users", usersHandler)
	mux.HandleFunc("/api/orders", ordersHandler)
	mux.HandleFunc("/health", healthHandler)
	mux.Handle("/metrics", promhttp.Handler())

	// Wrap the entire mux with metrics middleware
	instrumented := MetricsMiddleware(mux)

	fmt.Println("Server listening on :8080")
	fmt.Println("Endpoints:")
	fmt.Println("  GET /api/users   - list users")
	fmt.Println("  GET /api/orders  - list orders (5% error rate)")
	fmt.Println("  GET /health      - health check")
	fmt.Println("  GET /metrics     - Prometheus metrics")
	fmt.Println()
	fmt.Println("Try: curl localhost:8080/api/users && curl localhost:8080/metrics")

	http.ListenAndServe(":8080", instrumented)
}
```

After sending a few requests and hitting `/metrics`, you will see output like:

```
# HELP myapp_http_requests_total Total HTTP requests.
# TYPE myapp_http_requests_total counter
myapp_http_requests_total{method="GET",path="/api/users",status="200"} 5
myapp_http_requests_total{method="GET",path="/api/orders",status="200"} 4
myapp_http_requests_total{method="GET",path="/api/orders",status="500"} 1

# HELP myapp_http_request_duration_seconds HTTP request duration in seconds.
# TYPE myapp_http_request_duration_seconds histogram
myapp_http_request_duration_seconds_bucket{method="GET",path="/api/users",le="0.025"} 1
myapp_http_request_duration_seconds_bucket{method="GET",path="/api/users",le="0.05"} 3
myapp_http_request_duration_seconds_bucket{method="GET",path="/api/users",le="0.1"} 5
...
```

---

## 5. Custom Business Metrics

Beyond HTTP metrics, you should track domain-specific events that matter to the business.

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// --- Business Metrics ---
// These track domain-specific events, not infrastructure concerns.
// They answer questions like "How many users signed up today?"
// and "What is the average order value?"

var (
	// User lifecycle events
	userSignupsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "business",
			Name:      "user_signups_total",
			Help:      "Total user signups by source.",
		},
		[]string{"source"}, // "organic", "referral", "ad_campaign"
	)

	userLoginsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "business",
			Name:      "user_logins_total",
			Help:      "Total user logins by method.",
		},
		[]string{"method"}, // "password", "oauth_google", "oauth_github"
	)

	// Commerce events
	ordersTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "business",
			Name:      "orders_total",
			Help:      "Total orders by status.",
		},
		[]string{"status"}, // "completed", "cancelled", "refunded"
	)

	orderValueDollars = promauto.NewHistogram(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "business",
			Name:      "order_value_dollars",
			Help:      "Order value distribution in dollars.",
			Buckets:   []float64{5, 10, 25, 50, 100, 250, 500, 1000, 5000},
		},
	)

	// Payment processing
	paymentDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "payments",
			Name:      "processing_duration_seconds",
			Help:      "Payment processing duration in seconds.",
			Buckets:   []float64{0.1, 0.5, 1, 2, 5, 10, 30},
		},
		[]string{"provider"}, // "stripe", "paypal"
	)

	paymentErrorsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "payments",
			Name:      "errors_total",
			Help:      "Total payment processing errors.",
		},
		[]string{"provider", "error_type"}, // "stripe"+"card_declined", "paypal"+"timeout"
	)

	// Inventory
	inventoryLevel = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "inventory",
			Name:      "current_level",
			Help:      "Current inventory level by product category.",
		},
		[]string{"category"},
	)

	// Feature flags / experiments
	featureFlagEvaluations = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "features",
			Name:      "flag_evaluations_total",
			Help:      "Feature flag evaluation count.",
		},
		[]string{"flag", "result"}, // "new_checkout"+"enabled", "new_checkout"+"disabled"
	)
)

// --- Service Layer with Instrumented Business Logic ---

type OrderService struct{}

type Order struct {
	ID       string
	UserID   string
	Amount   float64
	Provider string
}

func (s *OrderService) ProcessOrder(ctx context.Context, order Order) error {
	// Track the order
	ordersTotal.WithLabelValues("processing").Inc()

	// Process payment with timing
	start := time.Now()
	err := s.processPayment(ctx, order)
	duration := time.Since(start).Seconds()
	paymentDuration.WithLabelValues(order.Provider).Observe(duration)

	if err != nil {
		paymentErrorsTotal.WithLabelValues(order.Provider, "processing_failed").Inc()
		ordersTotal.WithLabelValues("failed").Inc()
		return fmt.Errorf("payment failed: %w", err)
	}

	// Track successful order
	ordersTotal.WithLabelValues("completed").Inc()
	orderValueDollars.Observe(order.Amount)

	// Update inventory
	inventoryLevel.WithLabelValues("electronics").Dec()

	return nil
}

func (s *OrderService) processPayment(ctx context.Context, order Order) error {
	// Simulate payment processing
	time.Sleep(time.Duration(100+rand.Intn(900)) * time.Millisecond)

	// Simulate 5% failure rate
	if rand.Float64() < 0.05 {
		return fmt.Errorf("card declined")
	}
	return nil
}

// --- User Service ---

type UserService struct{}

func (s *UserService) Signup(ctx context.Context, email, source string) error {
	userSignupsTotal.WithLabelValues(source).Inc()

	// Evaluate feature flag for new users
	if rand.Float64() < 0.5 {
		featureFlagEvaluations.WithLabelValues("new_onboarding_flow", "enabled").Inc()
	} else {
		featureFlagEvaluations.WithLabelValues("new_onboarding_flow", "disabled").Inc()
	}

	return nil
}

func (s *UserService) Login(ctx context.Context, email, method string) error {
	userLoginsTotal.WithLabelValues(method).Inc()
	return nil
}

// --- Inventory Background Worker ---

func updateInventoryMetrics() {
	// In production, this would query your database.
	// Run periodically (e.g., every 30 seconds) to update gauge values.
	categories := map[string]float64{
		"electronics": float64(rand.Intn(1000)),
		"clothing":    float64(rand.Intn(5000)),
		"books":       float64(rand.Intn(10000)),
	}
	for category, level := range categories {
		inventoryLevel.WithLabelValues(category).Set(level)
	}
}

func main() {
	// Start inventory poller
	go func() {
		ticker := time.NewTicker(30 * time.Second)
		defer ticker.Stop()
		updateInventoryMetrics() // Initial update
		for range ticker.C {
			updateInventoryMetrics()
		}
	}()

	orderSvc := &OrderService{}
	userSvc := &UserService{}

	// Simulate some traffic in the background
	go func() {
		for {
			ctx := context.Background()

			// Simulate signups
			sources := []string{"organic", "referral", "ad_campaign"}
			userSvc.Signup(ctx, "test@example.com", sources[rand.Intn(len(sources))])

			// Simulate logins
			methods := []string{"password", "oauth_google", "oauth_github"}
			userSvc.Login(ctx, "test@example.com", methods[rand.Intn(len(methods))])

			// Simulate orders
			providers := []string{"stripe", "paypal"}
			order := Order{
				ID:       fmt.Sprintf("ord_%d", rand.Int()),
				UserID:   "user_123",
				Amount:   10 + rand.Float64()*490,
				Provider: providers[rand.Intn(len(providers))],
			}
			orderSvc.ProcessOrder(ctx, order)

			time.Sleep(time.Duration(100+rand.Intn(400)) * time.Millisecond)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("Business metrics server on :2112")
	fmt.Println("Visit http://localhost:2112/metrics to see business metrics")
	http.ListenAndServe(":2112", nil)
}
```

### What Business Metrics Tell You

With these metrics in Prometheus, you can build dashboards that answer:

- **Revenue health:** `rate(myapp_business_orders_total{status="completed"}[5m])` -- orders per second
- **Payment reliability:** `rate(myapp_payments_errors_total[5m]) / rate(myapp_business_orders_total[5m])` -- payment error rate
- **User growth:** `rate(myapp_business_user_signups_total[1d])` -- signups per day by source
- **Inventory alerts:** `myapp_inventory_current_level{category="electronics"} < 100` -- low stock alert
- **A/B test exposure:** `rate(myapp_features_flag_evaluations_total{flag="new_onboarding_flow"}[1h])` -- flag evaluation rates

---

## 6. Database Metrics

Database operations are often the primary source of latency in web applications. Instrumenting them is essential.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// --- Database Metrics ---

var (
	// Query duration by operation type
	dbQueryDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "query_duration_seconds",
			Help:      "Database query duration in seconds.",
			Buckets:   []float64{0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 5},
		},
		[]string{"operation", "table"},
	)

	// Query errors
	dbQueryErrorsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "query_errors_total",
			Help:      "Total database query errors.",
		},
		[]string{"operation", "table", "error_type"},
	)

	// Connection pool stats
	dbConnectionsActive = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "connections_active",
			Help:      "Number of active database connections.",
		},
	)

	dbConnectionsIdle = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "connections_idle",
			Help:      "Number of idle database connections.",
		},
	)

	dbConnectionsMaxOpen = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "connections_max_open",
			Help:      "Maximum number of open connections allowed.",
		},
	)

	dbConnectionWaitTotal = promauto.NewCounter(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "connection_wait_total",
			Help:      "Total number of times a connection was waited for.",
		},
	)

	dbConnectionWaitDuration = promauto.NewHistogram(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "connection_wait_duration_seconds",
			Help:      "Time spent waiting for a database connection.",
			Buckets:   []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5},
		},
	)

	// Transaction metrics
	dbTransactionDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "myapp",
			Subsystem: "db",
			Name:      "transaction_duration_seconds",
			Help:      "Database transaction duration in seconds.",
			Buckets:   []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1, 5, 10},
		},
		[]string{"result"}, // "commit", "rollback"
	)
)

// --- Instrumented Database Wrapper ---

// InstrumentedDB wraps *sql.DB and records metrics for every operation.
type InstrumentedDB struct {
	db *sql.DB
}

func NewInstrumentedDB(db *sql.DB) *InstrumentedDB {
	return &InstrumentedDB{db: db}
}

// QueryContext executes a query and records metrics.
func (idb *InstrumentedDB) QueryContext(ctx context.Context, table, query string, args ...interface{}) (*sql.Rows, error) {
	start := time.Now()
	rows, err := idb.db.QueryContext(ctx, query, args...)
	duration := time.Since(start).Seconds()

	dbQueryDuration.WithLabelValues("select", table).Observe(duration)
	if err != nil {
		dbQueryErrorsTotal.WithLabelValues("select", table, classifyDBError(err)).Inc()
	}
	return rows, err
}

// ExecContext executes a statement and records metrics.
func (idb *InstrumentedDB) ExecContext(ctx context.Context, operation, table, query string, args ...interface{}) (sql.Result, error) {
	start := time.Now()
	result, err := idb.db.ExecContext(ctx, query, args...)
	duration := time.Since(start).Seconds()

	dbQueryDuration.WithLabelValues(operation, table).Observe(duration)
	if err != nil {
		dbQueryErrorsTotal.WithLabelValues(operation, table, classifyDBError(err)).Inc()
	}
	return result, err
}

// classifyDBError maps database errors to categories for metric labels.
// Keep cardinality low by grouping errors into known categories.
func classifyDBError(err error) string {
	if err == nil {
		return "none"
	}
	errStr := err.Error()
	switch {
	case contains(errStr, "connection refused"):
		return "connection_refused"
	case contains(errStr, "timeout"):
		return "timeout"
	case contains(errStr, "deadlock"):
		return "deadlock"
	case contains(errStr, "duplicate key"):
		return "duplicate_key"
	case contains(errStr, "foreign key"):
		return "foreign_key_violation"
	default:
		return "unknown"
	}
}

func contains(s, substr string) bool {
	return len(s) >= len(substr) && searchString(s, substr)
}

func searchString(s, substr string) bool {
	for i := 0; i <= len(s)-len(substr); i++ {
		if s[i:i+len(substr)] == substr {
			return true
		}
	}
	return false
}

// --- Connection Pool Stats Collector ---

// DBStatsCollector periodically reads sql.DB.Stats() and updates gauges.
// Call this in a goroutine at startup.
func DBStatsCollector(db *sql.DB, interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for range ticker.C {
		stats := db.Stats()
		dbConnectionsActive.Set(float64(stats.InUse))
		dbConnectionsIdle.Set(float64(stats.Idle))
		dbConnectionsMaxOpen.Set(float64(stats.MaxOpenConnections))

		// Note: WaitCount and WaitDuration are cumulative.
		// In a real implementation, you would track deltas.
		// For simplicity, we just set the counter value directly.
	}
}

// --- Alternative: Prometheus Collector Interface ---

// DBStatsPrometheusCollector implements the prometheus.Collector interface
// to export sql.DB stats. This is the recommended approach because
// Prometheus will call Collect() at scrape time, ensuring fresh data.
type DBStatsPrometheusCollector struct {
	db *sql.DB

	// Descriptors -- defined once, returned in Describe()
	maxOpenDesc  *prometheus.Desc
	openDesc     *prometheus.Desc
	inUseDesc    *prometheus.Desc
	idleDesc     *prometheus.Desc
	waitDesc     *prometheus.Desc
	waitDurDesc  *prometheus.Desc
}

func NewDBStatsCollector(db *sql.DB, dbName string) *DBStatsPrometheusCollector {
	labels := prometheus.Labels{"db": dbName}
	return &DBStatsPrometheusCollector{
		db: db,
		maxOpenDesc: prometheus.NewDesc(
			"myapp_db_max_open_connections",
			"Maximum number of open connections to the database.",
			nil, labels,
		),
		openDesc: prometheus.NewDesc(
			"myapp_db_open_connections",
			"Number of established connections (in-use + idle).",
			nil, labels,
		),
		inUseDesc: prometheus.NewDesc(
			"myapp_db_in_use_connections",
			"Number of connections currently in use.",
			nil, labels,
		),
		idleDesc: prometheus.NewDesc(
			"myapp_db_idle_connections",
			"Number of idle connections.",
			nil, labels,
		),
		waitDesc: prometheus.NewDesc(
			"myapp_db_wait_count_total",
			"Total number of connections waited for.",
			nil, labels,
		),
		waitDurDesc: prometheus.NewDesc(
			"myapp_db_wait_duration_seconds_total",
			"Total time blocked waiting for a new connection.",
			nil, labels,
		),
	}
}

func (c *DBStatsPrometheusCollector) Describe(ch chan<- *prometheus.Desc) {
	ch <- c.maxOpenDesc
	ch <- c.openDesc
	ch <- c.inUseDesc
	ch <- c.idleDesc
	ch <- c.waitDesc
	ch <- c.waitDurDesc
}

func (c *DBStatsPrometheusCollector) Collect(ch chan<- prometheus.Metric) {
	stats := c.db.Stats()

	ch <- prometheus.MustNewConstMetric(c.maxOpenDesc, prometheus.GaugeValue, float64(stats.MaxOpenConnections))
	ch <- prometheus.MustNewConstMetric(c.openDesc, prometheus.GaugeValue, float64(stats.OpenConnections))
	ch <- prometheus.MustNewConstMetric(c.inUseDesc, prometheus.GaugeValue, float64(stats.InUse))
	ch <- prometheus.MustNewConstMetric(c.idleDesc, prometheus.GaugeValue, float64(stats.Idle))
	ch <- prometheus.MustNewConstMetric(c.waitDesc, prometheus.CounterValue, float64(stats.WaitCount))
	ch <- prometheus.MustNewConstMetric(c.waitDurDesc, prometheus.CounterValue, stats.WaitDuration.Seconds())
}

func main() {
	// In production, you would open a real database connection:
	//   db, _ := sql.Open("postgres", "postgresql://...")
	//   prometheus.MustRegister(NewDBStatsCollector(db, "primary"))
	//   idb := NewInstrumentedDB(db)

	// For this demo, we simulate metrics updates
	go func() {
		for {
			table := []string{"users", "orders", "products"}[rand.Intn(3)]
			op := []string{"select", "insert", "update"}[rand.Intn(3)]
			duration := 0.001 + rand.Float64()*0.1
			dbQueryDuration.WithLabelValues(op, table).Observe(duration)

			if rand.Float64() < 0.02 {
				errType := []string{"timeout", "deadlock", "connection_refused"}[rand.Intn(3)]
				dbQueryErrorsTotal.WithLabelValues(op, table, errType).Inc()
			}

			time.Sleep(50 * time.Millisecond)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("Database metrics server on :2112")
	http.ListenAndServe(":2112", nil)
}
```

---

## 7. Go Runtime Metrics

Go provides rich runtime metrics that help you understand the health of your process: memory usage, garbage collection, goroutine count, and more.

### Using the runtime/metrics Package

Go 1.16 introduced the `runtime/metrics` package, which provides a stable API for reading runtime metrics. This is preferred over directly reading `runtime.MemStats` because the metrics are well-defined and versioned.

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/metrics"
	"strings"
	"time"
)

func main() {
	// List all available runtime metrics
	descs := metrics.All()
	fmt.Printf("Go runtime exposes %d metrics:\n\n", len(descs))

	// Show a curated list of the most useful metrics
	interesting := []string{
		"/gc/cycles/total:gc-cycles",
		"/gc/heap/allocs:bytes",
		"/gc/heap/frees:bytes",
		"/gc/heap/goal:bytes",
		"/gc/pauses:seconds",
		"/memory/classes/heap/objects:bytes",
		"/memory/classes/heap/stacks:bytes",
		"/memory/classes/total:bytes",
		"/sched/goroutines:goroutines",
		"/sched/latencies:seconds",
	}

	// Read all interesting metrics
	samples := make([]metrics.Sample, len(interesting))
	for i, name := range interesting {
		samples[i].Name = name
	}
	metrics.Read(samples)

	fmt.Println("Key runtime metrics:")
	fmt.Println(strings.Repeat("-", 60))

	for _, s := range samples {
		switch s.Value.Kind() {
		case metrics.KindUint64:
			fmt.Printf("  %-45s %d\n", s.Name, s.Value.Uint64())
		case metrics.KindFloat64:
			fmt.Printf("  %-45s %.4f\n", s.Name, s.Value.Float64())
		case metrics.KindFloat64Histogram:
			h := s.Value.Float64Histogram()
			fmt.Printf("  %-45s (histogram, %d buckets)\n", s.Name, len(h.Buckets))
		case metrics.KindBad:
			fmt.Printf("  %-45s (unsupported)\n", s.Name)
		}
	}

	// Also show runtime.NumGoroutine() and runtime.GOMAXPROCS(0)
	fmt.Println()
	fmt.Printf("runtime.NumGoroutine():  %d\n", runtime.NumGoroutine())
	fmt.Printf("runtime.GOMAXPROCS(0):   %d\n", runtime.GOMAXPROCS(0))
	fmt.Printf("runtime.NumCPU():        %d\n", runtime.NumCPU())
	fmt.Printf("runtime.Version():       %s\n", runtime.Version())

	_ = time.Now() // suppress unused import
}
```

### Exposing Runtime Metrics as Prometheus Gauges

The default `prometheus.NewGoCollector()` already exposes many Go runtime metrics, but you can build a custom collector for specific metrics you care about:

```go
package main

import (
	"fmt"
	"net/http"
	"runtime"
	"runtime/metrics"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	goGoroutines = promauto.NewGauge(prometheus.GaugeOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "goroutines",
		Help:      "Number of goroutines that currently exist.",
	})

	goHeapBytes = promauto.NewGauge(prometheus.GaugeOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "heap_alloc_bytes",
		Help:      "Bytes allocated on the heap and still in use.",
	})

	goGCPauseDuration = promauto.NewHistogram(prometheus.HistogramOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "gc_pause_duration_seconds",
		Help:      "Distribution of GC pause durations.",
		Buckets:   []float64{0.00001, 0.00005, 0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05},
	})

	goGCCycles = promauto.NewCounter(prometheus.CounterOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "gc_cycles_total",
		Help:      "Total number of completed GC cycles.",
	})

	goHeapGoal = promauto.NewGauge(prometheus.GaugeOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "heap_goal_bytes",
		Help:      "Target heap size for the next GC cycle.",
	})

	goTotalAllocBytes = promauto.NewCounter(prometheus.CounterOpts{
		Namespace: "myapp",
		Subsystem: "runtime",
		Name:      "alloc_bytes_total",
		Help:      "Total cumulative bytes allocated (even if freed).",
	})
)

// collectRuntimeMetrics reads Go runtime metrics and updates Prometheus gauges.
// Run this in a goroutine with a ticker.
func collectRuntimeMetrics() {
	samples := []metrics.Sample{
		{Name: "/gc/cycles/total:gc-cycles"},
		{Name: "/memory/classes/heap/objects:bytes"},
		{Name: "/gc/heap/goal:bytes"},
		{Name: "/gc/heap/allocs:bytes"},
		{Name: "/sched/goroutines:goroutines"},
	}
	metrics.Read(samples)

	for _, s := range samples {
		switch s.Name {
		case "/sched/goroutines:goroutines":
			goGoroutines.Set(float64(s.Value.Uint64()))
		case "/memory/classes/heap/objects:bytes":
			goHeapBytes.Set(float64(s.Value.Uint64()))
		case "/gc/heap/goal:bytes":
			goHeapGoal.Set(float64(s.Value.Uint64()))
		}
	}
}

// runtimeMetricsLoop collects runtime metrics every interval.
func runtimeMetricsLoop(interval time.Duration, stop <-chan struct{}) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			collectRuntimeMetrics()
		case <-stop:
			return
		}
	}
}

func main() {
	stop := make(chan struct{})
	go runtimeMetricsLoop(10*time.Second, stop)

	// Generate some allocations to make metrics interesting
	go func() {
		for {
			data := make([]byte, 1024*1024) // 1MB allocation
			_ = data
			runtime.GC() // force GC for demo purposes
			time.Sleep(time.Second)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())
	fmt.Println("Runtime metrics server on :2112")
	http.ListenAndServe(":2112", nil)
}
```

### Key Runtime Metrics to Monitor

| Metric | What It Tells You |
|--------|-------------------|
| `goroutines` | Goroutine leak detection -- should be stable under steady load |
| `heap_alloc_bytes` | Memory usage -- sudden increases may indicate a leak |
| `gc_pause_duration` | GC latency impact -- long pauses affect tail latency |
| `gc_cycles_total` | GC frequency -- too frequent means excessive allocation |
| `heap_goal_bytes` | GC target -- shows when the runtime expects to trigger GC |

**Goroutine leak detection tip:** If your goroutine count grows monotonically under steady-state load, you have a goroutine leak. Set an alert for `myapp_runtime_goroutines > 10000` (adjust threshold to your baseline).

---

## 8. The /metrics Endpoint

The `/metrics` endpoint is where Prometheus scrapes your application's metrics. Setting it up correctly is crucial.

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	requestsTotal = promauto.NewCounter(prometheus.CounterOpts{
		Name: "myapp_requests_total",
		Help: "Total requests.",
	})

	buildInfo = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "myapp_build_info",
			Help: "Build information. Value is always 1. Labels carry the info.",
		},
		[]string{"version", "commit", "build_date", "go_version"},
	)
)

func main() {
	// Publish build info as a metric.
	// This is a common Prometheus pattern: a gauge that is always 1,
	// with labels carrying the information.
	// In Grafana: myapp_build_info shows you which version is deployed.
	version := getEnv("APP_VERSION", "dev")
	commit := getEnv("APP_COMMIT", "unknown")
	buildDate := getEnv("APP_BUILD_DATE", "unknown")
	buildInfo.WithLabelValues(version, commit, buildDate, "go1.22").Set(1)

	// --- Option 1: Metrics on the same port as the API ---
	// Simple, but /metrics is exposed on the same port as your API.
	// Fine for internal services. Risky for public-facing services.
	mux := http.NewServeMux()
	mux.HandleFunc("/api/hello", func(w http.ResponseWriter, r *http.Request) {
		requestsTotal.Inc()
		w.Write([]byte("hello"))
	})
	mux.Handle("/metrics", promhttp.Handler())

	// --- Option 2: Metrics on a separate port (recommended) ---
	// Run the metrics endpoint on a different port so it is not
	// exposed through your public load balancer.
	metricsMux := http.NewServeMux()
	metricsMux.Handle("/metrics", promhttp.HandlerFor(
		prometheus.DefaultGatherer,
		promhttp.HandlerOpts{
			// EnableOpenMetrics enables the OpenMetrics format,
			// which supports exemplars and other advanced features.
			EnableOpenMetrics: true,
			// Timeout prevents slow scrapes from blocking.
			Timeout: 10 * time.Second,
		},
	))

	// Health check on the metrics port (for Kubernetes probes)
	metricsMux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	})

	apiServer := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
	}

	metricsServer := &http.Server{
		Addr:         ":9090",
		Handler:      metricsMux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Start both servers
	go func() {
		fmt.Println("API server on :8080")
		if err := apiServer.ListenAndServe(); err != http.ErrServerClosed {
			fmt.Fprintf(os.Stderr, "API server error: %v\n", err)
			os.Exit(1)
		}
	}()

	go func() {
		fmt.Println("Metrics server on :9090")
		if err := metricsServer.ListenAndServe(); err != http.ErrServerClosed {
			fmt.Fprintf(os.Stderr, "Metrics server error: %v\n", err)
			os.Exit(1)
		}
	}()

	// Graceful shutdown
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	<-sigCh

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	apiServer.Shutdown(ctx)
	metricsServer.Shutdown(ctx)
	fmt.Println("Servers shut down gracefully")
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

### What Prometheus Sees

When Prometheus scrapes `/metrics`, it receives text in the Prometheus exposition format:

```
# HELP myapp_requests_total Total requests.
# TYPE myapp_requests_total counter
myapp_requests_total 42

# HELP myapp_build_info Build information. Value is always 1. Labels carry the info.
# TYPE myapp_build_info gauge
myapp_build_info{version="1.2.3",commit="abc123",build_date="2025-03-15",go_version="go1.22"} 1

# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8

# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 2.062336e+06
```

### Prometheus Configuration

In your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'myapp'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9090']
    # Or with service discovery in Kubernetes:
    # kubernetes_sd_configs:
    #   - role: pod
    # relabel_configs:
    #   - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    #     action: keep
    #     regex: true
```

---

## 9. RED Method

The RED method (Rate, Errors, Duration) is a monitoring methodology for request-driven services. It was popularized by Tom Wilkie (Grafana Labs) and focuses on the three signals that matter most for services handling requests.

### The Three Signals

- **Rate**: How many requests per second is the service handling?
- **Errors**: How many of those requests are failing?
- **Duration**: How long do the requests take?

If you can only have three dashboards for a service, these are the three.

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// REDMetrics encapsulates the three RED signals for a service.
// Every request-driven service should have these three metrics.
type REDMetrics struct {
	// Rate: requests per second (derived from this counter)
	RequestsTotal *prometheus.CounterVec

	// Errors: error rate (derived from requests with error status codes)
	// Note: errors are tracked via the "status" label on RequestsTotal.
	// A separate error counter is redundant if you label request status.

	// Duration: request latency distribution
	RequestDuration *prometheus.HistogramVec
}

func NewREDMetrics(namespace, subsystem string) *REDMetrics {
	return &REDMetrics{
		RequestsTotal: promauto.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: subsystem,
				Name:      "requests_total",
				Help:      "Total requests by method, path, and status code.",
			},
			[]string{"method", "path", "status_code"},
		),
		RequestDuration: promauto.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: namespace,
				Subsystem: subsystem,
				Name:      "request_duration_seconds",
				Help:      "Request duration in seconds.",
				Buckets:   []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2.5, 5},
			},
			[]string{"method", "path"},
		),
	}
}

// Observe records a request to the RED metrics.
func (r *REDMetrics) Observe(method, path string, statusCode int, duration time.Duration) {
	r.RequestsTotal.WithLabelValues(method, path, strconv.Itoa(statusCode)).Inc()
	r.RequestDuration.WithLabelValues(method, path).Observe(duration.Seconds())
}

// IsError returns true if the status code indicates a server error.
// Client errors (4xx) are generally NOT counted as service errors
// because they indicate bad input, not a broken service.
func IsError(statusCode int) bool {
	return statusCode >= 500
}

// REDMiddleware creates HTTP middleware that records RED metrics.
func REDMiddleware(red *REDMetrics) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			wrapped := &statusRecorder{ResponseWriter: w, statusCode: 200}

			next.ServeHTTP(wrapped, r)

			duration := time.Since(start)
			red.Observe(r.Method, r.URL.Path, wrapped.statusCode, duration)
		})
	}
}

type statusRecorder struct {
	http.ResponseWriter
	statusCode int
}

func (sr *statusRecorder) WriteHeader(code int) {
	sr.statusCode = code
	sr.ResponseWriter.WriteHeader(code)
}

func main() {
	red := NewREDMetrics("myapp", "api")

	mux := http.NewServeMux()
	mux.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(5+rand.Intn(45)) * time.Millisecond)
		if rand.Float64() < 0.02 {
			w.WriteHeader(http.StatusInternalServerError)
			w.Write([]byte(`{"error":"internal"}`))
			return
		}
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`[{"id":1}]`))
	})

	mux.HandleFunc("/api/orders", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(20+rand.Intn(180)) * time.Millisecond)
		if rand.Float64() < 0.05 {
			w.WriteHeader(http.StatusInternalServerError)
			w.Write([]byte(`{"error":"database timeout"}`))
			return
		}
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`[{"id":1,"total":99.99}]`))
	})

	mux.Handle("/metrics", promhttp.Handler())

	wrapped := REDMiddleware(red)(mux)

	fmt.Println("RED Method metrics server on :8080")
	fmt.Println()
	fmt.Println("PromQL queries for RED dashboards:")
	fmt.Println("  Rate:     rate(myapp_api_requests_total[5m])")
	fmt.Println("  Errors:   rate(myapp_api_requests_total{status_code=~\"5..\"}[5m])")
	fmt.Println("  Duration: histogram_quantile(0.99, rate(myapp_api_request_duration_seconds_bucket[5m]))")
	fmt.Println()
	fmt.Println("  Error %:  rate(myapp_api_requests_total{status_code=~\"5..\"}[5m])")
	fmt.Println("            / rate(myapp_api_requests_total[5m]) * 100")

	http.ListenAndServe(":8080", wrapped)
}
```

### RED Dashboard Queries (PromQL)

```
# Request rate by endpoint
sum(rate(myapp_api_requests_total[5m])) by (path)

# Error rate percentage by endpoint
sum(rate(myapp_api_requests_total{status_code=~"5.."}[5m])) by (path)
/
sum(rate(myapp_api_requests_total[5m])) by (path)
* 100

# P50 latency by endpoint
histogram_quantile(0.50, sum(rate(myapp_api_request_duration_seconds_bucket[5m])) by (le, path))

# P99 latency by endpoint
histogram_quantile(0.99, sum(rate(myapp_api_request_duration_seconds_bucket[5m])) by (le, path))
```

---

## 10. USE Method

The USE method (Utilization, Saturation, Errors) is a monitoring methodology for resource-oriented systems. It was created by Brendan Gregg and focuses on infrastructure resources like CPU, memory, disk, and network.

While RED applies to services (things that handle requests), USE applies to resources (things that are consumed).

### The Three Signals

- **Utilization**: What percentage of the resource's capacity is being used?
- **Saturation**: How much work is queued because the resource is full?
- **Errors**: How many error events has the resource produced?

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"sync"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// USEMetrics tracks Utilization, Saturation, and Errors for a resource.
type USEMetrics struct {
	// Utilization: fraction of capacity in use (0.0 to 1.0)
	Utilization *prometheus.GaugeVec

	// Saturation: amount of work queued beyond capacity
	Saturation *prometheus.GaugeVec

	// Errors: error events on the resource
	Errors *prometheus.CounterVec
}

func NewUSEMetrics(namespace, subsystem string) *USEMetrics {
	return &USEMetrics{
		Utilization: promauto.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Subsystem: subsystem,
				Name:      "utilization_ratio",
				Help:      "Resource utilization as a ratio (0.0 = idle, 1.0 = fully used).",
			},
			[]string{"resource"},
		),
		Saturation: promauto.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: namespace,
				Subsystem: subsystem,
				Name:      "saturation",
				Help:      "Resource saturation (queued work beyond capacity).",
			},
			[]string{"resource"},
		),
		Errors: promauto.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: namespace,
				Subsystem: subsystem,
				Name:      "errors_total",
				Help:      "Total resource errors.",
			},
			[]string{"resource", "error_type"},
		),
	}
}

// --- Worker Pool with USE Metrics ---
// A worker pool is a classic example of a resource you monitor with USE.

type WorkerPool struct {
	name     string
	workers  int
	queue    chan func()
	use      *USEMetrics
	active   int
	mu       sync.Mutex
}

func NewWorkerPool(name string, workers, queueSize int, use *USEMetrics) *WorkerPool {
	wp := &WorkerPool{
		name:    name,
		workers: workers,
		queue:   make(chan func(), queueSize),
		use:     use,
	}

	// Start worker goroutines
	for i := 0; i < workers; i++ {
		go wp.worker()
	}

	// Start metrics reporter
	go wp.reportMetrics()

	return wp
}

func (wp *WorkerPool) worker() {
	for task := range wp.queue {
		wp.mu.Lock()
		wp.active++
		wp.mu.Unlock()

		func() {
			defer func() {
				if r := recover(); r != nil {
					wp.use.Errors.WithLabelValues(wp.name, "panic").Inc()
				}
				wp.mu.Lock()
				wp.active--
				wp.mu.Unlock()
			}()
			task()
		}()
	}
}

func (wp *WorkerPool) Submit(task func()) error {
	select {
	case wp.queue <- task:
		return nil
	default:
		wp.use.Errors.WithLabelValues(wp.name, "queue_full").Inc()
		return fmt.Errorf("worker pool %s: queue full", wp.name)
	}
}

func (wp *WorkerPool) reportMetrics() {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		wp.mu.Lock()
		active := wp.active
		wp.mu.Unlock()

		queueLen := len(wp.queue)
		queueCap := cap(wp.queue)

		// Utilization: fraction of workers currently busy
		utilization := float64(active) / float64(wp.workers)
		wp.use.Utilization.WithLabelValues(wp.name).Set(utilization)

		// Saturation: items queued (waiting for a worker)
		saturation := float64(queueLen) / float64(queueCap)
		wp.use.Saturation.WithLabelValues(wp.name).Set(saturation)
	}
}

// --- Connection Pool USE Example ---

type ConnectionPoolMetrics struct {
	use      *USEMetrics
	name     string
	maxConns int
	inUse    int
	waiting  int
	mu       sync.Mutex
}

func NewConnectionPoolMetrics(name string, maxConns int, use *USEMetrics) *ConnectionPoolMetrics {
	return &ConnectionPoolMetrics{
		name:     name,
		maxConns: maxConns,
		use:      use,
	}
}

func (cp *ConnectionPoolMetrics) Update(inUse, idle, waiting int) {
	cp.mu.Lock()
	defer cp.mu.Unlock()

	// Utilization: fraction of max connections in use
	utilization := float64(inUse) / float64(cp.maxConns)
	cp.use.Utilization.WithLabelValues(cp.name).Set(utilization)

	// Saturation: number of goroutines waiting for a connection
	cp.use.Saturation.WithLabelValues(cp.name).Set(float64(waiting))
}

func main() {
	use := NewUSEMetrics("myapp", "resources")

	// Create a worker pool
	pool := NewWorkerPool("email_sender", 5, 100, use)

	// Simulate work
	go func() {
		for {
			pool.Submit(func() {
				// Simulate sending an email
				time.Sleep(time.Duration(50+rand.Intn(200)) * time.Millisecond)
			})
			time.Sleep(time.Duration(10+rand.Intn(40)) * time.Millisecond)
		}
	}()

	// Simulate connection pool metrics
	dbPool := NewConnectionPoolMetrics("postgres", 25, use)
	go func() {
		for {
			inUse := 5 + rand.Intn(20)
			idle := 25 - inUse
			waiting := 0
			if rand.Float64() < 0.1 {
				waiting = rand.Intn(10)
			}
			dbPool.Update(inUse, idle, waiting)
			time.Sleep(5 * time.Second)
		}
	}()

	http.Handle("/metrics", promhttp.Handler())

	fmt.Println("USE Method metrics server on :2112")
	fmt.Println()
	fmt.Println("USE dashboard queries:")
	fmt.Println("  Utilization:  myapp_resources_utilization_ratio")
	fmt.Println("  Saturation:   myapp_resources_saturation")
	fmt.Println("  Errors:       rate(myapp_resources_errors_total[5m])")

	http.ListenAndServe(":2112", nil)
}
```

### When to Use RED vs USE

| Methodology | Applies To | Examples |
|------------|-----------|---------|
| RED | Services (request handlers) | HTTP APIs, gRPC services, GraphQL resolvers |
| USE | Resources (infrastructure) | CPU, memory, disk, connection pools, worker pools, queues |

Most applications need both. RED for your API endpoints, USE for your connection pools, worker pools, and memory.

---

## 11. SLI (Service Level Indicators)

An SLI (Service Level Indicator) is a quantitative measure of some aspect of the level of service provided. It is the raw measurement that feeds into an SLO. "Observability Engineering" emphasizes that good SLIs should be defined from the user's perspective, not the system's perspective.

### What Makes a Good SLI?

A good SLI has two components:
- **Good events**: Events that met the quality threshold
- **Valid events**: All events that should be counted (excluding things like health checks or load balancer probes)

The SLI value is: `SLI = good events / valid events`

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// RequestEvent represents a single request observed by the system.
// This is the raw event from which SLIs are computed.
type RequestEvent struct {
	Method     string
	Path       string
	StatusCode int
	Duration   time.Duration
	Timestamp  time.Time
	UserID     string
	Error      error
}

// SLI defines a Service Level Indicator.
// It specifies how to classify events as "good" or "bad" and which
// events are "valid" (should be counted at all).
type SLI struct {
	Name        string
	Description string

	// Good returns true if this event met the quality threshold.
	Good func(event *RequestEvent) bool

	// Valid returns true if this event should be counted.
	// Events that are not valid are excluded from both numerator and denominator.
	// Example: exclude health checks, load balancer probes, rate-limited requests.
	Valid func(event *RequestEvent) bool
}

// Evaluate computes the SLI value (good/valid ratio) for a set of events.
func (s *SLI) Evaluate(events []*RequestEvent) SLIResult {
	var good, valid int64
	for _, e := range events {
		if s.Valid(e) {
			valid++
			if s.Good(e) {
				good++
			}
		}
	}
	ratio := 0.0
	if valid > 0 {
		ratio = float64(good) / float64(valid)
	}
	return SLIResult{
		SLIName:    s.Name,
		Good:       good,
		Valid:      valid,
		Ratio:      ratio,
		Percentage: ratio * 100,
	}
}

type SLIResult struct {
	SLIName    string
	Good       int64
	Valid      int64
	Ratio      float64
	Percentage float64
}

func (r SLIResult) String() string {
	return fmt.Sprintf("SLI[%s]: %d/%d good (%.3f%%) ",
		r.SLIName, r.Good, r.Valid, r.Percentage)
}

// --- Define Common SLIs ---

// Availability SLI: Was the request successful (non-5xx)?
var availabilitySLI = SLI{
	Name:        "availability",
	Description: "Proportion of requests that did not result in a server error (5xx).",
	Good: func(e *RequestEvent) bool {
		return e.StatusCode < 500
	},
	Valid: func(e *RequestEvent) bool {
		// Exclude health checks and metrics endpoint
		return e.Path != "/health" && e.Path != "/metrics" &&
			// Exclude rate-limited requests (429)
			e.StatusCode != 429
	},
}

// Latency SLI: Was the request fast enough?
var latencySLI = SLI{
	Name:        "latency",
	Description: "Proportion of requests that completed within 200ms.",
	Good: func(e *RequestEvent) bool {
		return e.Duration < 200*time.Millisecond
	},
	Valid: func(e *RequestEvent) bool {
		// Only count successful requests for latency SLI.
		// If the request errored, latency is not meaningful.
		return e.StatusCode < 500 &&
			e.Path != "/health" && e.Path != "/metrics" &&
			e.StatusCode != 429
	},
}

// Correctness SLI: Did the response have the right content?
// This requires application-specific validation.
var correctnessSLI = SLI{
	Name:        "correctness",
	Description: "Proportion of responses that returned correct data.",
	Good: func(e *RequestEvent) bool {
		// In a real system, you would check response body, checksums, etc.
		// Here we simulate: any non-error response is "correct"
		return e.StatusCode >= 200 && e.StatusCode < 400 && e.Error == nil
	},
	Valid: func(e *RequestEvent) bool {
		return e.StatusCode != 429 &&
			e.Path != "/health" && e.Path != "/metrics"
	},
}

// Freshness SLI: For read endpoints, is the data up-to-date?
// Common for services backed by eventually consistent stores.
var freshnessSLI = SLI{
	Name:        "freshness",
	Description: "Proportion of read requests returning data less than 5 seconds old.",
	Good: func(e *RequestEvent) bool {
		// In a real system, you would compare response data timestamp
		// to the current time. Simulated here.
		return rand.Float64() > 0.01 // 99% of reads are fresh
	},
	Valid: func(e *RequestEvent) bool {
		return e.Method == "GET" && e.StatusCode < 500
	},
}

func main() {
	// Generate simulated request events
	events := generateEvents(100000)

	// Evaluate all SLIs
	slis := []SLI{availabilitySLI, latencySLI, correctnessSLI, freshnessSLI}

	fmt.Println("SLI Evaluation Results")
	fmt.Println("=" + repeat("=", 59))
	for _, sli := range slis {
		result := sli.Evaluate(events)
		fmt.Printf("  %-15s %7d/%7d good  = %7.3f%%\n",
			result.SLIName, result.Good, result.Valid, result.Percentage)
	}
	fmt.Println()
	fmt.Printf("Total events: %d\n", len(events))
}

func generateEvents(n int) []*RequestEvent {
	paths := []string{"/api/users", "/api/orders", "/api/products", "/health", "/metrics"}
	events := make([]*RequestEvent, n)

	for i := 0; i < n; i++ {
		path := paths[rand.Intn(len(paths))]
		statusCode := 200

		r := rand.Float64()
		switch {
		case r < 0.005:
			statusCode = 500 // 0.5% server errors
		case r < 0.010:
			statusCode = 502
		case r < 0.015:
			statusCode = 503
		case r < 0.03:
			statusCode = 429 // rate limited
		case r < 0.05:
			statusCode = 404
		case r < 0.07:
			statusCode = 400
		}

		// Latency: most fast, some slow
		var duration time.Duration
		lr := rand.Float64()
		switch {
		case lr < 0.80:
			duration = time.Duration(5+rand.Intn(45)) * time.Millisecond
		case lr < 0.95:
			duration = time.Duration(50+rand.Intn(150)) * time.Millisecond
		case lr < 0.99:
			duration = time.Duration(200+rand.Intn(800)) * time.Millisecond
		default:
			duration = time.Duration(1000+rand.Intn(4000)) * time.Millisecond
		}

		events[i] = &RequestEvent{
			Method:     "GET",
			Path:       path,
			StatusCode: statusCode,
			Duration:   duration,
			Timestamp:  time.Now().Add(-time.Duration(rand.Intn(3600)) * time.Second),
		}
	}
	return events
}

func repeat(s string, n int) string {
	result := ""
	for i := 0; i < n; i++ {
		result += s
	}
	return result
}
```

Output (values will vary):
```
SLI Evaluation Results
============================================================
  availability     93287/  94003 good  =  99.238%
  latency          71482/  93287 good  =  76.620%
  correctness      88301/  94003 good  =  93.934%
  freshness        55678/  56234 good  =  99.011%

Total events: 100000
```

### SLIs with Prometheus

In production, SLIs are computed from Prometheus metrics, not from in-memory event arrays. Here is how to implement SLIs using Prometheus counters:

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// SLI metrics track good and valid events as separate counters.
// The SLI ratio is computed in PromQL: good / valid
var (
	sliRequestsValid = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "sli",
			Name:      "requests_valid_total",
			Help:      "Total valid requests for SLI calculation.",
		},
		[]string{"sli"},
	)

	sliRequestsGood = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "myapp",
			Subsystem: "sli",
			Name:      "requests_good_total",
			Help:      "Total good requests for SLI calculation.",
		},
		[]string{"sli"},
	)
)

// SLIRecorder evaluates SLIs for each request and updates Prometheus counters.
type SLIRecorder struct {
	availabilityThreshold int           // Max acceptable status code (exclusive)
	latencyThreshold      time.Duration // Max acceptable latency
}

func NewSLIRecorder(latencyThreshold time.Duration) *SLIRecorder {
	return &SLIRecorder{
		availabilityThreshold: 500,
		latencyThreshold:      latencyThreshold,
	}
}

func (s *SLIRecorder) Record(statusCode int, duration time.Duration, path string) {
	// Skip non-user-facing paths
	if path == "/health" || path == "/metrics" || path == "/readyz" {
		return
	}
	// Skip rate-limited requests
	if statusCode == 429 {
		return
	}

	// Availability SLI
	sliRequestsValid.WithLabelValues("availability").Inc()
	if statusCode < s.availabilityThreshold {
		sliRequestsGood.WithLabelValues("availability").Inc()
	}

	// Latency SLI (only for successful requests)
	if statusCode < s.availabilityThreshold {
		sliRequestsValid.WithLabelValues("latency").Inc()
		if duration < s.latencyThreshold {
			sliRequestsGood.WithLabelValues("latency").Inc()
		}
	}
}

// SLIMiddleware integrates SLI recording into the HTTP middleware chain.
func SLIMiddleware(recorder *SLIRecorder) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			wrapped := &statusWriter{ResponseWriter: w, code: 200}

			next.ServeHTTP(wrapped, r)

			duration := time.Since(start)
			recorder.Record(wrapped.code, duration, r.URL.Path)
		})
	}
}

type statusWriter struct {
	http.ResponseWriter
	code int
}

func (sw *statusWriter) WriteHeader(code int) {
	sw.code = code
	sw.ResponseWriter.WriteHeader(code)
}

func main() {
	recorder := NewSLIRecorder(200 * time.Millisecond)

	mux := http.NewServeMux()
	mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(time.Duration(10+rand.Intn(190)) * time.Millisecond)
		if rand.Float64() < 0.01 {
			w.WriteHeader(500)
			return
		}
		w.WriteHeader(200)
		w.Write([]byte(`{"data":"ok"}`))
	})
	mux.Handle("/metrics", promhttp.Handler())

	wrapped := SLIMiddleware(recorder)(mux)

	fmt.Println("SLI metrics server on :8080")
	fmt.Println()
	fmt.Println("PromQL for SLI ratios:")
	fmt.Println("  Availability SLI:")
	fmt.Println("    rate(myapp_sli_requests_good_total{sli=\"availability\"}[5m])")
	fmt.Println("    / rate(myapp_sli_requests_valid_total{sli=\"availability\"}[5m])")
	fmt.Println()
	fmt.Println("  Latency SLI:")
	fmt.Println("    rate(myapp_sli_requests_good_total{sli=\"latency\"}[5m])")
	fmt.Println("    / rate(myapp_sli_requests_valid_total{sli=\"latency\"}[5m])")

	http.ListenAndServe(":8080", wrapped)
}
```

---

## 12. SLO (Service Level Objectives)

An SLO (Service Level Objective) is a target value for an SLI over a time window. It is the contract between your service and its users. "Observability Engineering" argues that SLOs are the single most important concept in production operations because they define what "reliable enough" means.

### The Error Budget

If your SLO is 99.9% availability over 30 days, your error budget is 0.1% -- that is how much "unreliability" you are allowed. Over 30 days, that works out to:

```
30 days * 24 hours * 60 minutes * 0.001 = 43.2 minutes of allowed downtime
```

Or, in terms of requests: if you serve 1 million requests per day, your error budget allows 1,000 failed requests per day.

```go
package main

import (
	"fmt"
	"math"
	"time"
)

// SLO represents a Service Level Objective: a target percentage
// for an SLI over a rolling time window.
type SLO struct {
	Name        string
	Description string
	Target      float64       // e.g., 0.999 = 99.9%
	Window      time.Duration // e.g., 30 * 24 * time.Hour
}

// ErrorBudget returns the fraction of events that are allowed to be "bad."
func (s *SLO) ErrorBudget() float64 {
	return 1 - s.Target
}

// ErrorBudgetMinutes returns the allowed downtime in minutes for the window.
func (s *SLO) ErrorBudgetMinutes() float64 {
	return s.Window.Minutes() * s.ErrorBudget()
}

// ErrorBudgetRemaining computes how much of the error budget has been consumed.
// Returns a value between 0.0 (budget exhausted) and 1.0 (budget untouched).
// Can go negative if the budget is overdrawn.
func (s *SLO) ErrorBudgetRemaining(good, total int64) float64 {
	if total == 0 {
		return 1.0 // No data means budget is untouched
	}
	actual := float64(good) / float64(total)
	budgetUsed := (1 - actual) / s.ErrorBudget()
	return 1 - budgetUsed
}

// BurnRate computes how fast the error budget is being consumed.
// A burn rate of 1.0 means the budget will be exactly exhausted at the end of the window.
// A burn rate of 2.0 means the budget will be exhausted in half the window.
// A burn rate of 0.5 means only half the budget will be used over the full window.
func (s *SLO) BurnRate(good, total int64) float64 {
	if total == 0 {
		return 0
	}
	errorRate := 1 - (float64(good) / float64(total))
	allowedErrorRate := s.ErrorBudget()
	if allowedErrorRate == 0 {
		return math.Inf(1) // 100% target, any error is infinite burn
	}
	return errorRate / allowedErrorRate
}

// Status returns a human-readable status based on error budget remaining.
func (s *SLO) Status(good, total int64) string {
	remaining := s.ErrorBudgetRemaining(good, total)
	burnRate := s.BurnRate(good, total)
	switch {
	case remaining > 0.75:
		return fmt.Sprintf("HEALTHY (%.1f%% budget remaining, burn rate %.2f)", remaining*100, burnRate)
	case remaining > 0.25:
		return fmt.Sprintf("WARNING (%.1f%% budget remaining, burn rate %.2f)", remaining*100, burnRate)
	case remaining > 0:
		return fmt.Sprintf("DANGER  (%.1f%% budget remaining, burn rate %.2f)", remaining*100, burnRate)
	default:
		return fmt.Sprintf("VIOLATED (%.1f%% budget remaining, burn rate %.2f)", remaining*100, burnRate)
	}
}

func main() {
	// Define SLOs
	slos := []SLO{
		{
			Name:        "API Availability",
			Description: "99.9% of requests succeed (non-5xx) over 30 days",
			Target:      0.999,
			Window:      30 * 24 * time.Hour,
		},
		{
			Name:        "API Latency",
			Description: "99.0% of requests complete within 200ms over 30 days",
			Target:      0.990,
			Window:      30 * 24 * time.Hour,
		},
		{
			Name:        "Data Freshness",
			Description: "99.5% of read requests return data <5s old over 7 days",
			Target:      0.995,
			Window:      7 * 24 * time.Hour,
		},
	}

	// Print SLO details
	fmt.Println("Service Level Objectives")
	fmt.Println("========================")
	for _, slo := range slos {
		fmt.Printf("\n%s\n", slo.Name)
		fmt.Printf("  Description:     %s\n", slo.Description)
		fmt.Printf("  Target:          %.1f%%\n", slo.Target*100)
		fmt.Printf("  Error budget:    %.2f%% (%.1f minutes in %d-day window)\n",
			slo.ErrorBudget()*100, slo.ErrorBudgetMinutes(), int(slo.Window.Hours()/24))
	}

	// Simulate different scenarios
	fmt.Println("\n\nScenario Analysis")
	fmt.Println("=================")

	scenarios := []struct {
		name  string
		good  int64
		total int64
	}{
		{"Normal operation", 999700, 1000000},   // 99.97% good
		{"Minor degradation", 999000, 1000000},  // 99.9% good
		{"SLO boundary", 998500, 1000000},       // 99.85% good
		{"Budget exhausted", 997000, 1000000},    // 99.7% good
		{"Major incident", 990000, 1000000},      // 99.0% good
	}

	for _, scenario := range scenarios {
		fmt.Printf("\n--- %s (%d/%d = %.3f%%) ---\n",
			scenario.name, scenario.good, scenario.total,
			float64(scenario.good)/float64(scenario.total)*100)

		for _, slo := range slos {
			status := slo.Status(scenario.good, scenario.total)
			remaining := slo.ErrorBudgetRemaining(scenario.good, scenario.total)
			fmt.Printf("  %-20s %s\n", slo.Name+":", status)
			_ = remaining
		}
	}

	// Error budget over time
	fmt.Println("\n\nError Budget Consumption Over Time")
	fmt.Println("===================================")
	slo := slos[0] // API Availability 99.9%

	fmt.Printf("SLO: %s (target %.1f%%, 30-day window)\n\n", slo.Name, slo.Target*100)
	fmt.Println("Day  | Good     | Total    | Budget Remaining | Burn Rate | Status")
	fmt.Println("-----|----------|----------|------------------|-----------|-------")

	totalGood := int64(0)
	totalAll := int64(0)
	dailyRequests := int64(100000)

	for day := 1; day <= 30; day++ {
		// Simulate daily traffic with varying error rates
		var dailyGood int64
		switch {
		case day == 15: // Bad day -- incident
			dailyGood = dailyRequests - 500 // 0.5% errors
		case day == 16: // Recovery
			dailyGood = dailyRequests - 200
		default:
			dailyGood = dailyRequests - 50 // Normal: 0.05% errors
		}

		totalGood += dailyGood
		totalAll += dailyRequests

		remaining := slo.ErrorBudgetRemaining(totalGood, totalAll)
		burnRate := slo.BurnRate(totalGood, totalAll)
		status := "OK"
		if remaining < 0.25 {
			status = "DANGER"
		} else if remaining < 0.5 {
			status = "WARNING"
		}
		if day == 1 || day == 15 || day == 16 || day == 30 || day%5 == 0 {
			fmt.Printf("%-4d | %8d | %8d | %14.1f%%  | %9.2f | %s\n",
				day, totalGood, totalAll, remaining*100, burnRate, status)
		}
	}
}
```

Output:
```
Service Level Objectives
========================

API Availability
  Description:     99.9% of requests succeed (non-5xx) over 30 days
  Target:          99.9%
  Error budget:    0.10% (43.2 minutes in 30-day window)

API Latency
  Description:     99.0% of requests complete within 200ms over 30 days
  Target:          99.0%
  Error budget:    1.00% (432.0 minutes in 30-day window)

Data Freshness
  Description:     99.5% of read requests return data <5s old over 7 days
  Target:          99.5%
  Error budget:    0.50% (50.4 minutes in 7-day window)
```

### Choosing SLO Targets

"Observability Engineering" offers clear guidance on choosing targets:

1. **Start with user expectations.** If users expect a page to load in under 1 second, your latency SLO should reflect that.
2. **Do not set targets higher than you can achieve.** If your service currently runs at 99.5% availability, do not set a 99.99% SLO.
3. **Do not set targets at 100%.** A 100% SLO means zero error budget, which means you can never deploy, never do maintenance, and never have a single failed request.
4. **Review and adjust quarterly.** SLOs are not set in stone. Tighten them as your service matures.

Common SLO targets:

| Service Type | Availability | Latency (P99) |
|-------------|-------------|---------------|
| Internal API | 99.5% | 1s |
| User-facing API | 99.9% | 200ms |
| Critical payment API | 99.95% | 500ms |
| Static content CDN | 99.99% | 50ms |

---

## 13. Error Budget Policies

An error budget policy defines what happens when the error budget is consumed. Without a policy, SLOs are just numbers on a dashboard. With a policy, SLOs drive engineering decisions.

```go
package main

import (
	"fmt"
	"time"
)

// ErrorBudgetPolicy defines actions to take at different budget consumption levels.
type ErrorBudgetPolicy struct {
	SLOName     string
	Target      float64
	Window      time.Duration
	Thresholds  []PolicyThreshold
}

// PolicyThreshold defines an action when the budget drops below a certain level.
type PolicyThreshold struct {
	BudgetRemaining float64 // Trigger when budget drops below this (0.0 to 1.0)
	Action          string
	Description     string
	AutomatedAction func() error // Optional automated action
}

// Evaluate checks the current error budget against all thresholds.
func (p *ErrorBudgetPolicy) Evaluate(good, total int64) []TriggeredAction {
	if total == 0 {
		return nil
	}

	actual := float64(good) / float64(total)
	budgetUsed := (1 - actual) / (1 - p.Target)
	remaining := 1 - budgetUsed

	var triggered []TriggeredAction
	for _, t := range p.Thresholds {
		if remaining <= t.BudgetRemaining {
			triggered = append(triggered, TriggeredAction{
				Threshold:       t,
				BudgetRemaining: remaining,
				Good:            good,
				Total:           total,
			})
		}
	}
	return triggered
}

type TriggeredAction struct {
	Threshold       PolicyThreshold
	BudgetRemaining float64
	Good            int64
	Total           int64
}

func main() {
	// Define a comprehensive error budget policy
	policy := ErrorBudgetPolicy{
		SLOName: "API Availability",
		Target:  0.999,
		Window:  30 * 24 * time.Hour,
		Thresholds: []PolicyThreshold{
			{
				BudgetRemaining: 0.75,
				Action:          "INFORM",
				Description:     "25% of error budget consumed. Post in #reliability channel.",
			},
			{
				BudgetRemaining: 0.50,
				Action:          "REVIEW",
				Description: "50% of error budget consumed. " +
					"Review recent deployments and changes. " +
					"Require extra review for risky deployments.",
			},
			{
				BudgetRemaining: 0.25,
				Action:          "SLOW_DOWN",
				Description: "75% of error budget consumed. " +
					"Freeze non-critical deployments. " +
					"Focus engineering effort on reliability work. " +
					"Page on-call SRE for awareness.",
			},
			{
				BudgetRemaining: 0.0,
				Action:          "FREEZE",
				Description: "Error budget exhausted. " +
					"All deployments frozen except reliability fixes. " +
					"Incident review required. " +
					"Leadership escalation. " +
					"All engineering effort redirected to reliability.",
			},
			{
				BudgetRemaining: -0.5,
				Action:          "ESCALATE",
				Description: "Error budget 150% consumed. SLO significantly violated. " +
					"VP-level escalation. Customer communication required. " +
					"Post-mortem with action items due within 48 hours.",
			},
		},
	}

	// Simulate different budget states
	scenarios := []struct {
		name  string
		good  int64
		total int64
	}{
		{"Healthy", 999900, 1000000},           // 99.99% -> 90% remaining
		{"25% consumed", 999750, 1000000},      // 99.975% -> 75% remaining
		{"50% consumed", 999500, 1000000},       // 99.95% -> 50% remaining
		{"75% consumed", 999250, 1000000},       // 99.925% -> 25% remaining
		{"Budget exhausted", 999000, 1000000},   // 99.9% -> 0% remaining
		{"SLO violated", 998000, 1000000},       // 99.8% -> -100% remaining
	}

	fmt.Printf("Error Budget Policy: %s (target %.1f%%)\n", policy.SLOName, policy.Target*100)
	fmt.Println("=" + repeatStr("=", 69))

	for _, s := range scenarios {
		fmt.Printf("\n--- %s (%.3f%% success) ---\n", s.name,
			float64(s.good)/float64(s.total)*100)

		triggered := policy.Evaluate(s.good, s.total)
		if len(triggered) == 0 {
			fmt.Println("  No actions triggered. Budget healthy.")
		}
		for _, t := range triggered {
			fmt.Printf("  [%s] Budget: %.1f%% | %s\n",
				t.Threshold.Action, t.BudgetRemaining*100, t.Threshold.Description)
		}
	}

	// Show the practical impact of different SLO targets
	fmt.Println("\n\nError Budget Comparison (30-day window, 1M requests/day)")
	fmt.Println("=" + repeatStr("=", 69))
	targets := []float64{0.99, 0.999, 0.9999}
	for _, target := range targets {
		budget := 1 - target
		dailyBudget := budget * 1000000 // 1M requests per day
		monthlyMinutes := budget * 30 * 24 * 60
		fmt.Printf("  %.2f%% -> %.0f bad req/day, %.1f minutes downtime/month\n",
			target*100, dailyBudget, monthlyMinutes)
	}
}

func repeatStr(s string, n int) string {
	result := ""
	for i := 0; i < n; i++ {
		result += s
	}
	return result
}
```

### Automating Error Budget Actions

In a production system, error budget policy actions should be automated:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"time"
)

// ErrorBudgetEnforcer periodically checks error budgets and takes action.
type ErrorBudgetEnforcer struct {
	logger *slog.Logger
	// In production, these would be real integrations
	slackNotifier  func(channel, message string) error
	pagerduty      func(severity, message string) error
	deploymentGate func(allow bool) error
}

func NewErrorBudgetEnforcer(logger *slog.Logger) *ErrorBudgetEnforcer {
	return &ErrorBudgetEnforcer{
		logger: logger,
		slackNotifier: func(channel, message string) error {
			fmt.Printf("[Slack -> %s] %s\n", channel, message)
			return nil
		},
		pagerduty: func(severity, message string) error {
			fmt.Printf("[PagerDuty %s] %s\n", severity, message)
			return nil
		},
		deploymentGate: func(allow bool) error {
			if allow {
				fmt.Println("[Deploy Gate] Deployments ENABLED")
			} else {
				fmt.Println("[Deploy Gate] Deployments FROZEN")
			}
			return nil
		},
	}
}

// CheckAndEnforce evaluates the error budget and takes automated actions.
func (e *ErrorBudgetEnforcer) CheckAndEnforce(ctx context.Context, sloName string, budgetRemaining float64) {
	e.logger.Info("error budget check",
		"slo", sloName,
		"budget_remaining_pct", fmt.Sprintf("%.1f%%", budgetRemaining*100),
	)

	switch {
	case budgetRemaining > 0.75:
		// All clear -- ensure deployments are enabled
		e.deploymentGate(true)

	case budgetRemaining > 0.50:
		// Informational alert
		e.slackNotifier("#reliability",
			fmt.Sprintf("SLO %s: %.0f%% error budget consumed. Review recent changes.",
				sloName, (1-budgetRemaining)*100))
		e.deploymentGate(true)

	case budgetRemaining > 0.25:
		// Warning -- slow down deployments
		e.slackNotifier("#reliability",
			fmt.Sprintf("WARNING: SLO %s: %.0f%% error budget consumed. Freezing non-critical deploys.",
				sloName, (1-budgetRemaining)*100))
		e.pagerduty("warning",
			fmt.Sprintf("SLO %s error budget at %.0f%%", sloName, budgetRemaining*100))
		e.deploymentGate(false) // Freeze non-critical

	case budgetRemaining > 0:
		// Danger -- freeze everything
		e.slackNotifier("#incidents",
			fmt.Sprintf("DANGER: SLO %s: Only %.0f%% error budget remaining. ALL deploys frozen.",
				sloName, budgetRemaining*100))
		e.pagerduty("critical",
			fmt.Sprintf("SLO %s error budget nearly exhausted: %.0f%% remaining", sloName, budgetRemaining*100))
		e.deploymentGate(false)

	default:
		// Budget exhausted
		e.slackNotifier("#incidents",
			fmt.Sprintf("SLO VIOLATION: %s error budget exhausted. Leadership escalation required.",
				sloName))
		e.pagerduty("critical",
			fmt.Sprintf("SLO %s VIOLATED. Budget overdrawn by %.0f%%", sloName, -budgetRemaining*100))
		e.deploymentGate(false)
	}
}

func main() {
	logger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
	enforcer := NewErrorBudgetEnforcer(logger)

	ctx := context.Background()

	// Simulate error budget dropping over time
	scenarios := []struct {
		day       int
		remaining float64
	}{
		{1, 0.95},
		{5, 0.80},
		{10, 0.60},
		{15, 0.40},
		{20, 0.15},
		{25, -0.20},
	}

	for _, s := range scenarios {
		fmt.Printf("\n=== Day %d ===\n", s.day)
		enforcer.CheckAndEnforce(ctx, "API Availability", s.remaining)
	}

	_ = time.Now() // suppress unused import
}
```

---

## 14. Multi-Window Burn Rate Alerts

Traditional threshold alerts ("alert if error rate > 1%") are noisy and miss slow burns. Multi-window burn rate alerts, recommended by Google's SRE book and "Observability Engineering," solve this by detecting both fast burns (short intense incidents) and slow burns (gradual degradation).

### How Burn Rate Alerting Works

The burn rate is how fast you are consuming your error budget relative to the SLO window. A burn rate of 1 means you will exactly exhaust your budget at the end of the window. A burn rate of 14.4 means you will exhaust your entire 30-day budget in just 50 hours.

Multi-window alerts use two windows: a long window for detection and a short window for confirmation. Both must fire to trigger the alert.

```go
package main

import (
	"fmt"
	"math"
	"time"
)

// BurnRateAlert defines a multi-window burn rate alert.
// Both the long window and short window must show the burn rate
// exceeding the threshold for the alert to fire.
type BurnRateAlert struct {
	Name        string
	LongWindow  time.Duration
	ShortWindow time.Duration
	BurnRate    float64 // threshold
	Severity    string
	// What fraction of the error budget this alert detects
	BudgetConsumed float64
}

// Recommended multi-window burn rate alerts for a 30-day SLO window.
// These are derived from the Google SRE Workbook.
//
// The math:
//   burn_rate = (budget_consumed / detection_time) * slo_window
//   For 2% budget in 1 hour: burn_rate = (0.02 / 1h) * 720h = 14.4
//   For 5% budget in 6 hours: burn_rate = (0.05 / 6h) * 720h = 6.0
//   For 10% budget in 3 days: burn_rate = (0.10 / 72h) * 720h = 1.0
var recommendedAlerts = []BurnRateAlert{
	{
		Name:           "fast_burn_critical",
		LongWindow:     1 * time.Hour,
		ShortWindow:    5 * time.Minute,
		BurnRate:       14.4,
		Severity:       "critical",
		BudgetConsumed: 0.02, // 2% of budget in 1 hour
	},
	{
		Name:           "fast_burn_high",
		LongWindow:     6 * time.Hour,
		ShortWindow:    30 * time.Minute,
		BurnRate:       6.0,
		Severity:       "critical",
		BudgetConsumed: 0.05, // 5% of budget in 6 hours
	},
	{
		Name:           "slow_burn_warning",
		LongWindow:     3 * 24 * time.Hour, // 3 days
		ShortWindow:    6 * time.Hour,
		BurnRate:       1.0,
		Severity:       "warning",
		BudgetConsumed: 0.10, // 10% of budget in 3 days
	},
}

// BurnRateCalculator computes burn rates from metrics.
type BurnRateCalculator struct {
	SLOTarget float64
}

// Calculate computes the burn rate from good and total event counts
// within a time window.
func (c *BurnRateCalculator) Calculate(good, total int64) float64 {
	if total == 0 {
		return 0
	}
	errorRate := 1 - (float64(good) / float64(total))
	allowedErrorRate := 1 - c.SLOTarget
	if allowedErrorRate == 0 {
		return math.Inf(1)
	}
	return errorRate / allowedErrorRate
}

// TimeToExhaustion returns how long until the error budget is fully consumed
// at the current burn rate.
func (c *BurnRateCalculator) TimeToExhaustion(burnRate float64, sloWindow time.Duration) time.Duration {
	if burnRate <= 0 {
		return time.Duration(math.MaxInt64) // infinite
	}
	hours := sloWindow.Hours() / burnRate
	return time.Duration(hours * float64(time.Hour))
}

// AlertEvaluator checks if burn rate alerts should fire.
type AlertEvaluator struct {
	calc   *BurnRateCalculator
	alerts []BurnRateAlert
}

func NewAlertEvaluator(sloTarget float64, alerts []BurnRateAlert) *AlertEvaluator {
	return &AlertEvaluator{
		calc:   &BurnRateCalculator{SLOTarget: sloTarget},
		alerts: alerts,
	}
}

// WindowMetrics represents the good/total counts for a specific time window.
type WindowMetrics struct {
	Good  int64
	Total int64
}

// Evaluate checks all alerts and returns those that are firing.
func (ae *AlertEvaluator) Evaluate(longWindowMetrics, shortWindowMetrics map[time.Duration]WindowMetrics) []FiringAlert {
	var firing []FiringAlert

	for _, alert := range ae.alerts {
		longMetrics, okLong := longWindowMetrics[alert.LongWindow]
		shortMetrics, okShort := shortWindowMetrics[alert.ShortWindow]

		if !okLong || !okShort {
			continue
		}

		longBurnRate := ae.calc.Calculate(longMetrics.Good, longMetrics.Total)
		shortBurnRate := ae.calc.Calculate(shortMetrics.Good, shortMetrics.Total)

		// Both windows must exceed the threshold
		if longBurnRate >= alert.BurnRate && shortBurnRate >= alert.BurnRate {
			tte := ae.calc.TimeToExhaustion(longBurnRate, 30*24*time.Hour)
			firing = append(firing, FiringAlert{
				Alert:            alert,
				LongBurnRate:     longBurnRate,
				ShortBurnRate:    shortBurnRate,
				TimeToExhaustion: tte,
			})
		}
	}

	return firing
}

type FiringAlert struct {
	Alert            BurnRateAlert
	LongBurnRate     float64
	ShortBurnRate    float64
	TimeToExhaustion time.Duration
}

func main() {
	fmt.Println("Multi-Window Burn Rate Alert Configuration")
	fmt.Println("SLO: 99.9% availability over 30 days")
	fmt.Println("Error budget: 0.1% = 43.2 minutes")
	fmt.Println()
	fmt.Println("Alert Configuration:")
	fmt.Println("--------------------------------------------------------------------")
	fmt.Printf("%-25s | %-12s | %-12s | %-6s | %-8s\n",
		"Alert", "Long Window", "Short Window", "Rate", "Severity")
	fmt.Println("--------------------------------------------------------------------")
	for _, a := range recommendedAlerts {
		fmt.Printf("%-25s | %-12s | %-12s | %-6.1f | %-8s\n",
			a.Name, a.LongWindow, a.ShortWindow, a.BurnRate, a.Severity)
	}

	// Simulate different scenarios
	evaluator := NewAlertEvaluator(0.999, recommendedAlerts)

	scenarios := []struct {
		name string
		// Metrics for different windows
		longWindows  map[time.Duration]WindowMetrics
		shortWindows map[time.Duration]WindowMetrics
	}{
		{
			name: "Normal operation (0.01% error rate)",
			longWindows: map[time.Duration]WindowMetrics{
				1 * time.Hour:          {Good: 99990, Total: 100000},
				6 * time.Hour:          {Good: 599940, Total: 600000},
				3 * 24 * time.Hour:     {Good: 7199280, Total: 7200000},
			},
			shortWindows: map[time.Duration]WindowMetrics{
				5 * time.Minute:  {Good: 8332, Total: 8333},
				30 * time.Minute: {Good: 49995, Total: 50000},
				6 * time.Hour:    {Good: 599940, Total: 600000},
			},
		},
		{
			name: "Major incident (5% error rate for 30 min)",
			longWindows: map[time.Duration]WindowMetrics{
				1 * time.Hour:          {Good: 95000, Total: 100000}, // 5% errors
				6 * time.Hour:          {Good: 595000, Total: 600000},
				3 * 24 * time.Hour:     {Good: 7195000, Total: 7200000},
			},
			shortWindows: map[time.Duration]WindowMetrics{
				5 * time.Minute:  {Good: 7917, Total: 8333},  // 5% errors
				30 * time.Minute: {Good: 47500, Total: 50000}, // 5% errors
				6 * time.Hour:    {Good: 595000, Total: 600000},
			},
		},
		{
			name: "Slow degradation (0.15% error rate over 3 days)",
			longWindows: map[time.Duration]WindowMetrics{
				1 * time.Hour:          {Good: 99850, Total: 100000},
				6 * time.Hour:          {Good: 599100, Total: 600000},
				3 * 24 * time.Hour:     {Good: 7189200, Total: 7200000}, // 0.15% error
			},
			shortWindows: map[time.Duration]WindowMetrics{
				5 * time.Minute:  {Good: 8320, Total: 8333},
				30 * time.Minute: {Good: 49925, Total: 50000},
				6 * time.Hour:    {Good: 599100, Total: 600000}, // 0.15% error
			},
		},
	}

	for _, s := range scenarios {
		fmt.Printf("\n\n=== Scenario: %s ===\n", s.name)
		firing := evaluator.Evaluate(s.longWindows, s.shortWindows)

		if len(firing) == 0 {
			fmt.Println("  No alerts firing.")
		} else {
			for _, f := range firing {
				fmt.Printf("  ALERT [%s] %s\n", f.Alert.Severity, f.Alert.Name)
				fmt.Printf("    Long window burn rate:  %.2f (threshold: %.1f)\n",
					f.LongBurnRate, f.Alert.BurnRate)
				fmt.Printf("    Short window burn rate: %.2f (threshold: %.1f)\n",
					f.ShortBurnRate, f.Alert.BurnRate)
				fmt.Printf("    Time to budget exhaustion: %s\n", f.TimeToExhaustion.Round(time.Minute))
			}
		}
	}

	// Show the PromQL for these alerts
	fmt.Println("\n\nPromQL Alert Rules:")
	fmt.Println("===================")
	fmt.Println(`
# Fast burn: 2% budget consumed in 1 hour
# Burn rate 14.4x, long window 1h, short window 5m
- alert: SLOHighErrorRate_Fast
  expr: |
    (
      1 - (rate(myapp_sli_requests_good_total{sli="availability"}[1h])
           / rate(myapp_sli_requests_valid_total{sli="availability"}[1h]))
    ) / 0.001 > 14.4
    AND
    (
      1 - (rate(myapp_sli_requests_good_total{sli="availability"}[5m])
           / rate(myapp_sli_requests_valid_total{sli="availability"}[5m]))
    ) / 0.001 > 14.4
  labels:
    severity: critical
  annotations:
    summary: "High error rate burning 2% of 30d error budget per hour"

# Slow burn: 10% budget consumed in 3 days
# Burn rate 1.0x, long window 3d, short window 6h
- alert: SLOHighErrorRate_Slow
  expr: |
    (
      1 - (rate(myapp_sli_requests_good_total{sli="availability"}[3d])
           / rate(myapp_sli_requests_valid_total{sli="availability"}[3d]))
    ) / 0.001 > 1.0
    AND
    (
      1 - (rate(myapp_sli_requests_good_total{sli="availability"}[6h])
           / rate(myapp_sli_requests_valid_total{sli="availability"}[6h]))
    ) / 0.001 > 1.0
  labels:
    severity: warning
  annotations:
    summary: "Elevated error rate burning 10% of 30d error budget per 3 days"`)
}
```

### Why Multi-Window?

The dual-window approach solves two problems:

1. **The long window** detects sustained issues. A 1-hour window with a burn rate of 14.4 means the error rate is consistently high enough to consume 2% of the 30-day budget in just 1 hour.

2. **The short window** confirms the issue is current. Without it, the alert would keep firing long after the incident ended, because the long window still includes the bad period.

Both must fire simultaneously. This dramatically reduces false positives compared to simple threshold alerts.

---

## 15. Building an SLO Dashboard

A complete SLO dashboard shows the current status of all SLOs, error budget consumption, and trend information. Here is how to build one in Go that serves a JSON API for a frontend dashboard.

```go
package main

import (
	"encoding/json"
	"fmt"
	"math"
	"math/rand"
	"net/http"
	"sync"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// --- SLO Data Model ---

type SLODefinition struct {
	Name        string        `json:"name"`
	Description string        `json:"description"`
	SLIName     string        `json:"sli_name"`
	Target      float64       `json:"target"`      // e.g., 0.999
	Window      time.Duration `json:"window"`       // e.g., 30 days
}

type SLOStatus struct {
	SLO               SLODefinition `json:"slo"`
	CurrentSLI        float64       `json:"current_sli"`          // actual SLI value
	Good              int64         `json:"good_events"`
	Valid             int64         `json:"valid_events"`
	ErrorBudgetTotal  float64       `json:"error_budget_total"`   // allowed bad events
	ErrorBudgetUsed   float64       `json:"error_budget_used"`    // bad events that occurred
	BudgetRemaining   float64       `json:"budget_remaining_pct"` // 0-100
	BurnRate          float64       `json:"burn_rate"`
	TimeToExhaustion  string        `json:"time_to_exhaustion"`
	Status            string        `json:"status"`               // HEALTHY, WARNING, DANGER, VIOLATED
	WindowStart       time.Time     `json:"window_start"`
	WindowEnd         time.Time     `json:"window_end"`
}

// --- SLO Tracker ---

// SLOTracker maintains running counters for SLI events and computes SLO status.
// In production, these counters would come from Prometheus queries.
// This in-memory version is useful for services that need SLO awareness
// without depending on an external Prometheus server.
type SLOTracker struct {
	mu         sync.RWMutex
	slos       []SLODefinition
	// counters: sliName -> {good, valid}
	good       map[string]int64
	valid      map[string]int64
	windowStart time.Time
}

func NewSLOTracker(slos []SLODefinition) *SLOTracker {
	return &SLOTracker{
		slos:        slos,
		good:        make(map[string]int64),
		valid:       make(map[string]int64),
		windowStart: time.Now(),
	}
}

func (t *SLOTracker) RecordGood(sliName string) {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.good[sliName]++
	t.valid[sliName]++
}

func (t *SLOTracker) RecordBad(sliName string) {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.valid[sliName]++
}

func (t *SLOTracker) GetStatus() []SLOStatus {
	t.mu.RLock()
	defer t.mu.RUnlock()

	var statuses []SLOStatus
	now := time.Now()

	for _, slo := range t.slos {
		good := t.good[slo.SLIName]
		valid := t.valid[slo.SLIName]

		status := SLOStatus{
			SLO:         slo,
			Good:        good,
			Valid:       valid,
			WindowStart: t.windowStart,
			WindowEnd:   t.windowStart.Add(slo.Window),
		}

		if valid > 0 {
			status.CurrentSLI = float64(good) / float64(valid)
			bad := float64(valid - good)
			totalBudget := float64(valid) * (1 - slo.Target)

			status.ErrorBudgetTotal = totalBudget
			status.ErrorBudgetUsed = bad

			if totalBudget > 0 {
				status.BudgetRemaining = (1 - bad/totalBudget) * 100
			}

			errorRate := 1 - status.CurrentSLI
			allowedErrorRate := 1 - slo.Target
			if allowedErrorRate > 0 {
				status.BurnRate = errorRate / allowedErrorRate
			}

			if status.BurnRate > 0 {
				hoursRemaining := slo.Window.Hours() / status.BurnRate
				tte := time.Duration(hoursRemaining * float64(time.Hour))
				if tte > 0 && tte < 10*365*24*time.Hour {
					status.TimeToExhaustion = tte.Round(time.Minute).String()
				} else {
					status.TimeToExhaustion = "infinite"
				}
			} else {
				status.TimeToExhaustion = "infinite"
			}
		} else {
			status.BudgetRemaining = 100
			status.TimeToExhaustion = "infinite"
		}

		// Determine status
		switch {
		case status.BudgetRemaining > 75:
			status.Status = "HEALTHY"
		case status.BudgetRemaining > 50:
			status.Status = "CAUTION"
		case status.BudgetRemaining > 25:
			status.Status = "WARNING"
		case status.BudgetRemaining > 0:
			status.Status = "DANGER"
		default:
			status.Status = "VIOLATED"
		}

		statuses = append(statuses, status)
	}

	_ = now
	return statuses
}

// --- Prometheus Metrics for the Dashboard ---

var (
	sloBudgetRemaining = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "slo",
			Name:      "error_budget_remaining_ratio",
			Help:      "Remaining error budget as a ratio (0.0 to 1.0).",
		},
		[]string{"slo"},
	)

	sloBurnRate = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "slo",
			Name:      "burn_rate",
			Help:      "Current error budget burn rate.",
		},
		[]string{"slo"},
	)

	sloCurrentValue = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "slo",
			Name:      "current_value",
			Help:      "Current SLI value (0.0 to 1.0).",
		},
		[]string{"slo"},
	)

	sloViolated = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "myapp",
			Subsystem: "slo",
			Name:      "violated",
			Help:      "Whether the SLO is currently violated (1=yes, 0=no).",
		},
		[]string{"slo"},
	)
)

// updatePrometheusMetrics syncs the SLO tracker state to Prometheus gauges.
func updatePrometheusMetrics(tracker *SLOTracker) {
	for _, status := range tracker.GetStatus() {
		name := status.SLO.Name
		sloBudgetRemaining.WithLabelValues(name).Set(status.BudgetRemaining / 100)
		sloBurnRate.WithLabelValues(name).Set(status.BurnRate)
		sloCurrentValue.WithLabelValues(name).Set(status.CurrentSLI)
		if status.Status == "VIOLATED" {
			sloViolated.WithLabelValues(name).Set(1)
		} else {
			sloViolated.WithLabelValues(name).Set(0)
		}
	}
}

// --- HTTP Handlers ---

func dashboardHandler(tracker *SLOTracker) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		statuses := tracker.GetStatus()

		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("Cache-Control", "no-cache")

		response := struct {
			Timestamp time.Time   `json:"timestamp"`
			SLOs      []SLOStatus `json:"slos"`
		}{
			Timestamp: time.Now(),
			SLOs:      statuses,
		}

		enc := json.NewEncoder(w)
		enc.SetIndent("", "  ")
		enc.Encode(response)
	}
}

func dashboardHTMLHandler(tracker *SLOTracker) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		statuses := tracker.GetStatus()

		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		fmt.Fprint(w, `<!DOCTYPE html>
<html><head><title>SLO Dashboard</title>
<meta http-equiv="refresh" content="10">
<style>
body { font-family: monospace; background: #1a1a2e; color: #e0e0e0; padding: 20px; }
h1 { color: #00d4ff; }
.slo { border: 1px solid #333; border-radius: 8px; padding: 16px; margin: 12px 0; }
.HEALTHY { border-color: #00c853; }
.CAUTION { border-color: #ffd600; }
.WARNING { border-color: #ff9100; }
.DANGER { border-color: #ff1744; }
.VIOLATED { border-color: #ff1744; background: #2a0a0a; }
.status { font-size: 1.2em; font-weight: bold; }
.metric { display: inline-block; margin-right: 24px; }
.value { font-size: 1.4em; }
</style></head><body>
<h1>SLO Dashboard</h1>
<p>Last updated: ` + time.Now().Format(time.RFC3339) + `</p>`)

		for _, s := range statuses {
			statusColor := map[string]string{
				"HEALTHY": "#00c853", "CAUTION": "#ffd600",
				"WARNING": "#ff9100", "DANGER": "#ff1744", "VIOLATED": "#ff1744",
			}
			color := statusColor[s.Status]
			fmt.Fprintf(w, `
<div class="slo %s">
  <div class="status" style="color:%s">%s: %s</div>
  <p>%s</p>
  <div class="metric">SLI: <span class="value">%.3f%%</span> (target: %.1f%%)</div>
  <div class="metric">Budget: <span class="value">%.1f%%</span> remaining</div>
  <div class="metric">Burn Rate: <span class="value">%.2f</span></div>
  <div class="metric">Events: <span class="value">%d/%d</span> good</div>
  <div class="metric">Exhaustion: <span class="value">%s</span></div>
</div>`,
				s.Status, color, s.SLO.Name, s.Status, s.SLO.Description,
				s.CurrentSLI*100, s.SLO.Target*100,
				s.BudgetRemaining, s.BurnRate,
				s.Good, s.Valid, s.TimeToExhaustion)
		}

		fmt.Fprint(w, `</body></html>`)
	}
}

func main() {
	// Define SLOs
	slos := []SLODefinition{
		{
			Name:        "API Availability",
			Description: "99.9% of API requests return non-5xx responses",
			SLIName:     "availability",
			Target:      0.999,
			Window:      30 * 24 * time.Hour,
		},
		{
			Name:        "API Latency",
			Description: "99% of API requests complete within 200ms",
			SLIName:     "latency",
			Target:      0.99,
			Window:      30 * 24 * time.Hour,
		},
		{
			Name:        "Payment Success",
			Description: "99.95% of payment attempts succeed",
			SLIName:     "payment_success",
			Target:      0.9995,
			Window:      7 * 24 * time.Hour,
		},
	}

	tracker := NewSLOTracker(slos)

	// Simulate traffic in the background
	go func() {
		for {
			// Availability: 0.2% error rate (above 0.1% budget)
			if rand.Float64() < 0.998 {
				tracker.RecordGood("availability")
			} else {
				tracker.RecordBad("availability")
			}

			// Latency: 1.5% too slow (above 1% budget)
			if rand.Float64() < 0.985 {
				tracker.RecordGood("latency")
			} else {
				tracker.RecordBad("latency")
			}

			// Payment: 0.03% failure (within 0.05% budget)
			if rand.Float64() < 0.9997 {
				tracker.RecordGood("payment_success")
			} else {
				tracker.RecordBad("payment_success")
			}

			time.Sleep(time.Millisecond)
		}
	}()

	// Update Prometheus metrics periodically
	go func() {
		ticker := time.NewTicker(15 * time.Second)
		defer ticker.Stop()
		for range ticker.C {
			updatePrometheusMetrics(tracker)
		}
	}()

	mux := http.NewServeMux()
	mux.HandleFunc("/", dashboardHTMLHandler(tracker))
	mux.HandleFunc("/api/slo", dashboardHandler(tracker))
	mux.Handle("/metrics", promhttp.Handler())

	fmt.Println("SLO Dashboard server on :8080")
	fmt.Println("  http://localhost:8080/        - HTML dashboard (auto-refreshes)")
	fmt.Println("  http://localhost:8080/api/slo  - JSON API")
	fmt.Println("  http://localhost:8080/metrics  - Prometheus metrics")

	_ = math.Inf(1) // suppress unused import
	http.ListenAndServe(":8080", mux)
}
```

The JSON API response looks like:

```json
{
  "timestamp": "2025-03-15T10:30:00Z",
  "slos": [
    {
      "slo": {
        "name": "API Availability",
        "description": "99.9% of API requests return non-5xx responses",
        "target": 0.999,
        "window": 2592000000000000
      },
      "current_sli": 0.9981,
      "good_events": 502345,
      "valid_events": 503300,
      "error_budget_total": 503.3,
      "error_budget_used": 955,
      "budget_remaining_pct": -89.7,
      "burn_rate": 1.897,
      "time_to_exhaustion": "380h0m",
      "status": "VIOLATED"
    }
  ]
}
```

---

## 16. Real-World Example: Fully Instrumented API with SLOs

This final example ties everything together: a complete HTTP API with Prometheus metrics, RED method monitoring, SLI/SLO tracking, error budget computation, and burn rate alerting. This is the kind of instrumentation you would find in a production Go service.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"sync"
	"sync/atomic"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// =============================================================
// Section 1: Metrics Definitions
// =============================================================

// RED metrics for the HTTP layer
var (
	httpRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "orderapi",
			Subsystem: "http",
			Name:      "requests_total",
			Help:      "Total HTTP requests by method, route, and status code.",
		},
		[]string{"method", "route", "status_code"},
	)

	httpRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "orderapi",
			Subsystem: "http",
			Name:      "request_duration_seconds",
			Help:      "HTTP request duration distribution in seconds.",
			Buckets:   []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1, 2.5, 5},
		},
		[]string{"method", "route"},
	)

	httpResponseSize = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "orderapi",
			Subsystem: "http",
			Name:      "response_size_bytes",
			Help:      "HTTP response body size in bytes.",
			Buckets:   prometheus.ExponentialBuckets(100, 10, 6),
		},
		[]string{"method", "route"},
	)

	httpActiveRequests = promauto.NewGauge(
		prometheus.GaugeOpts{
			Namespace: "orderapi",
			Subsystem: "http",
			Name:      "active_requests",
			Help:      "Number of in-flight HTTP requests.",
		},
	)
)

// SLI metrics
var (
	sliValid = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "orderapi",
			Subsystem: "sli",
			Name:      "valid_total",
			Help:      "Total valid events for SLI calculation.",
		},
		[]string{"sli"},
	)

	sliGood = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "orderapi",
			Subsystem: "sli",
			Name:      "good_total",
			Help:      "Total good events for SLI calculation.",
		},
		[]string{"sli"},
	)
)

// Business metrics
var (
	ordersCreated = promauto.NewCounter(
		prometheus.CounterOpts{
			Namespace: "orderapi",
			Subsystem: "business",
			Name:      "orders_created_total",
			Help:      "Total orders successfully created.",
		},
	)

	orderValue = promauto.NewHistogram(
		prometheus.HistogramOpts{
			Namespace: "orderapi",
			Subsystem: "business",
			Name:      "order_value_dollars",
			Help:      "Order value distribution in dollars.",
			Buckets:   []float64{10, 25, 50, 100, 250, 500, 1000},
		},
	)
)

// Database metrics
var (
	dbQueryDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Namespace: "orderapi",
			Subsystem: "db",
			Name:      "query_duration_seconds",
			Help:      "Database query duration in seconds.",
			Buckets:   []float64{0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1},
		},
		[]string{"operation"},
	)

	dbErrors = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Namespace: "orderapi",
			Subsystem: "db",
			Name:      "errors_total",
			Help:      "Total database errors.",
		},
		[]string{"operation"},
	)
)

// Build info
var (
	buildInfoMetric = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Namespace: "orderapi",
			Name:      "build_info",
			Help:      "Build information.",
		},
		[]string{"version", "commit"},
	)
)

// =============================================================
// Section 2: Response Writer Wrapper
// =============================================================

type responseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
	wroteHeader  bool
}

func newResponseWriter(w http.ResponseWriter) *responseWriter {
	return &responseWriter{ResponseWriter: w, statusCode: 200}
}

func (rw *responseWriter) WriteHeader(code int) {
	if rw.wroteHeader {
		return
	}
	rw.statusCode = code
	rw.wroteHeader = true
	rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
	if !rw.wroteHeader {
		rw.WriteHeader(http.StatusOK)
	}
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// =============================================================
// Section 3: Middleware Stack
// =============================================================

// Route resolver maps URL paths to normalized route patterns.
// This prevents high cardinality from path parameters.
var routePatterns = map[string]string{
	"/api/orders":  "/api/orders",
	"/api/users":   "/api/users",
	"/api/health":  "/api/health",
}

func resolveRoute(path string) string {
	if pattern, ok := routePatterns[path]; ok {
		return pattern
	}
	// Check prefixes for parameterized routes
	if len(path) > len("/api/orders/") && path[:len("/api/orders/")] == "/api/orders/" {
		return "/api/orders/:id"
	}
	if len(path) > len("/api/users/") && path[:len("/api/users/")] == "/api/users/" {
		return "/api/users/:id"
	}
	return "/other"
}

// MetricsMiddleware records RED metrics and SLI events for every request.
func MetricsMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			route := resolveRoute(r.URL.Path)
			wrapped := newResponseWriter(w)

			httpActiveRequests.Inc()
			defer httpActiveRequests.Dec()

			next.ServeHTTP(wrapped, r)

			duration := time.Since(start)
			status := strconv.Itoa(wrapped.statusCode)

			// RED metrics
			httpRequestsTotal.WithLabelValues(r.Method, route, status).Inc()
			httpRequestDuration.WithLabelValues(r.Method, route).Observe(duration.Seconds())
			httpResponseSize.WithLabelValues(r.Method, route).Observe(float64(wrapped.bytesWritten))

			// SLI recording (skip health checks)
			if route != "/api/health" && route != "/metrics" {
				// Availability SLI
				sliValid.WithLabelValues("availability").Inc()
				if wrapped.statusCode < 500 {
					sliGood.WithLabelValues("availability").Inc()
				}

				// Latency SLI (only for successful requests)
				if wrapped.statusCode < 500 {
					sliValid.WithLabelValues("latency").Inc()
					if duration < 200*time.Millisecond {
						sliGood.WithLabelValues("latency").Inc()
					}
				}
			}

			// Structured log
			logger.Info("request completed",
				"method", r.Method,
				"route", route,
				"status", wrapped.statusCode,
				"duration_ms", duration.Milliseconds(),
				"bytes", wrapped.bytesWritten,
			)
		})
	}
}

// RecoveryMiddleware catches panics and records them as 500 errors.
func RecoveryMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if err := recover(); err != nil {
					logger.Error("panic recovered",
						"error", fmt.Sprintf("%v", err),
						"method", r.Method,
						"path", r.URL.Path,
					)
					w.WriteHeader(http.StatusInternalServerError)
					json.NewEncoder(w).Encode(map[string]string{
						"error": "internal server error",
					})
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// =============================================================
// Section 4: Domain Models and Simulated Database
// =============================================================

type Order struct {
	ID        string    `json:"id"`
	UserID    string    `json:"user_id"`
	Items     []string  `json:"items"`
	Total     float64   `json:"total"`
	Status    string    `json:"status"`
	CreatedAt time.Time `json:"created_at"`
}

type User struct {
	ID    string `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

// In-memory database (for demo purposes)
type Database struct {
	mu     sync.RWMutex
	orders map[string]*Order
	users  map[string]*User
	nextID atomic.Int64
}

func NewDatabase() *Database {
	db := &Database{
		orders: make(map[string]*Order),
		users:  make(map[string]*User),
	}
	// Seed data
	db.users["user_1"] = &User{ID: "user_1", Name: "Alice", Email: "alice@example.com"}
	db.users["user_2"] = &User{ID: "user_2", Name: "Bob", Email: "bob@example.com"}
	return db
}

func (db *Database) CreateOrder(ctx context.Context, order *Order) error {
	start := time.Now()
	defer func() {
		dbQueryDuration.WithLabelValues("insert").Observe(time.Since(start).Seconds())
	}()

	// Simulate database latency
	time.Sleep(time.Duration(2+rand.Intn(15)) * time.Millisecond)

	// Simulate occasional database errors
	if rand.Float64() < 0.005 {
		dbErrors.WithLabelValues("insert").Inc()
		return fmt.Errorf("database connection timeout")
	}

	db.mu.Lock()
	defer db.mu.Unlock()

	id := fmt.Sprintf("ord_%d", db.nextID.Add(1))
	order.ID = id
	order.CreatedAt = time.Now()
	order.Status = "pending"
	db.orders[id] = order
	return nil
}

func (db *Database) GetOrder(ctx context.Context, id string) (*Order, error) {
	start := time.Now()
	defer func() {
		dbQueryDuration.WithLabelValues("select").Observe(time.Since(start).Seconds())
	}()

	time.Sleep(time.Duration(1+rand.Intn(5)) * time.Millisecond)

	db.mu.RLock()
	defer db.mu.RUnlock()

	order, ok := db.orders[id]
	if !ok {
		return nil, fmt.Errorf("order not found: %s", id)
	}
	return order, nil
}

func (db *Database) ListOrders(ctx context.Context) ([]*Order, error) {
	start := time.Now()
	defer func() {
		dbQueryDuration.WithLabelValues("select").Observe(time.Since(start).Seconds())
	}()

	time.Sleep(time.Duration(5+rand.Intn(20)) * time.Millisecond)

	if rand.Float64() < 0.002 {
		dbErrors.WithLabelValues("select").Inc()
		return nil, fmt.Errorf("query timeout")
	}

	db.mu.RLock()
	defer db.mu.RUnlock()

	orders := make([]*Order, 0, len(db.orders))
	for _, o := range db.orders {
		orders = append(orders, o)
	}
	return orders, nil
}

func (db *Database) ListUsers(ctx context.Context) ([]*User, error) {
	start := time.Now()
	defer func() {
		dbQueryDuration.WithLabelValues("select").Observe(time.Since(start).Seconds())
	}()

	time.Sleep(time.Duration(1+rand.Intn(5)) * time.Millisecond)

	db.mu.RLock()
	defer db.mu.RUnlock()

	users := make([]*User, 0, len(db.users))
	for _, u := range db.users {
		users = append(users, u)
	}
	return users, nil
}

// =============================================================
// Section 5: HTTP Handlers
// =============================================================

type API struct {
	db     *Database
	logger *slog.Logger
}

func (a *API) handleListOrders(w http.ResponseWriter, r *http.Request) {
	orders, err := a.db.ListOrders(r.Context())
	if err != nil {
		a.logger.Error("failed to list orders", "error", err)
		w.WriteHeader(http.StatusInternalServerError)
		json.NewEncoder(w).Encode(map[string]string{"error": "failed to list orders"})
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(orders)
}

func (a *API) handleCreateOrder(w http.ResponseWriter, r *http.Request) {
	var req struct {
		UserID string   `json:"user_id"`
		Items  []string `json:"items"`
		Total  float64  `json:"total"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{"error": "invalid request body"})
		return
	}

	order := &Order{
		UserID: req.UserID,
		Items:  req.Items,
		Total:  req.Total,
	}

	if err := a.db.CreateOrder(r.Context(), order); err != nil {
		a.logger.Error("failed to create order", "error", err)
		w.WriteHeader(http.StatusInternalServerError)
		json.NewEncoder(w).Encode(map[string]string{"error": "failed to create order"})
		return
	}

	// Business metrics
	ordersCreated.Inc()
	orderValue.Observe(order.Total)

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(order)
}

func (a *API) handleListUsers(w http.ResponseWriter, r *http.Request) {
	users, err := a.db.ListUsers(r.Context())
	if err != nil {
		a.logger.Error("failed to list users", "error", err)
		w.WriteHeader(http.StatusInternalServerError)
		json.NewEncoder(w).Encode(map[string]string{"error": "failed to list users"})
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(users)
}

func (a *API) handleHealth(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{
		"status": "ok",
		"time":   time.Now().Format(time.RFC3339),
	})
}

// =============================================================
// Section 6: SLO Dashboard Endpoint
// =============================================================

type SLODashboardResponse struct {
	Timestamp time.Time         `json:"timestamp"`
	SLOs      []SLOStatusReport `json:"slos"`
}

type SLOStatusReport struct {
	Name           string  `json:"name"`
	Target         float64 `json:"target"`
	Description    string  `json:"description"`
	Note           string  `json:"note"`
}

func (a *API) handleSLODashboard(w http.ResponseWriter, r *http.Request) {
	// In production, you would query Prometheus here using the Prometheus
	// HTTP API (e.g., /api/v1/query). For this example, we provide the
	// PromQL queries that power the dashboard.
	response := SLODashboardResponse{
		Timestamp: time.Now(),
		SLOs: []SLOStatusReport{
			{
				Name:        "API Availability",
				Target:      0.999,
				Description: "99.9% of requests return non-5xx",
				Note: "PromQL: rate(orderapi_sli_good_total{sli='availability'}[30d]) " +
					"/ rate(orderapi_sli_valid_total{sli='availability'}[30d])",
			},
			{
				Name:        "API Latency",
				Target:      0.99,
				Description: "99% of requests complete within 200ms",
				Note: "PromQL: rate(orderapi_sli_good_total{sli='latency'}[30d]) " +
					"/ rate(orderapi_sli_valid_total{sli='latency'}[30d])",
			},
		},
	}

	w.Header().Set("Content-Type", "application/json")
	enc := json.NewEncoder(w)
	enc.SetIndent("", "  ")
	enc.Encode(response)
}

// =============================================================
// Section 7: Traffic Simulator (for demo purposes)
// =============================================================

func simulateTraffic(ctx context.Context, baseURL string, logger *slog.Logger) {
	client := &http.Client{Timeout: 5 * time.Second}
	ticker := time.NewTicker(50 * time.Millisecond) // ~20 RPS
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			go func() {
				// 70% GET orders, 20% POST orders, 10% GET users
				r := rand.Float64()
				var req *http.Request
				var err error

				switch {
				case r < 0.70:
					req, err = http.NewRequest("GET", baseURL+"/api/orders", nil)
				case r < 0.90:
					body := fmt.Sprintf(
						`{"user_id":"user_%d","items":["item_%d"],"total":%.2f}`,
						rand.Intn(2)+1, rand.Intn(100), 10+rand.Float64()*490,
					)
					req, err = http.NewRequestWithContext(ctx, "POST", baseURL+"/api/orders",
						http.NoBody)
					if err == nil {
						req, err = http.NewRequest("POST", baseURL+"/api/orders",
							stringReader(body))
						req.Header.Set("Content-Type", "application/json")
					}
				default:
					req, err = http.NewRequest("GET", baseURL+"/api/users", nil)
				}

				if err != nil {
					return
				}

				resp, err := client.Do(req)
				if err != nil {
					return
				}
				resp.Body.Close()
			}()
		}
	}
}

type stringReaderType struct {
	s string
	i int
}

func (sr *stringReaderType) Read(p []byte) (int, error) {
	if sr.i >= len(sr.s) {
		return 0, fmt.Errorf("EOF")
	}
	n := copy(p, sr.s[sr.i:])
	sr.i += n
	return n, nil
}

func stringReader(s string) *stringReaderType {
	return &stringReaderType{s: s}
}

// =============================================================
// Section 8: Main -- Wire Everything Together
// =============================================================

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	// Publish build info
	buildInfoMetric.WithLabelValues("1.0.0", "abc123").Set(1)

	// Initialize database
	db := NewDatabase()

	// Initialize API handlers
	api := &API{db: db, logger: logger}

	// Build route table
	mux := http.NewServeMux()
	mux.HandleFunc("/api/orders", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodGet:
			api.handleListOrders(w, r)
		case http.MethodPost:
			api.handleCreateOrder(w, r)
		default:
			w.WriteHeader(http.StatusMethodNotAllowed)
		}
	})
	mux.HandleFunc("/api/users", api.handleListUsers)
	mux.HandleFunc("/api/health", api.handleHealth)
	mux.HandleFunc("/api/slo", api.handleSLODashboard)

	// Apply middleware (innermost first)
	var handler http.Handler = mux
	handler = MetricsMiddleware(logger)(handler)
	handler = RecoveryMiddleware(logger)(handler)

	// Metrics endpoint on a separate mux (best practice)
	metricsMux := http.NewServeMux()
	metricsMux.Handle("/metrics", promhttp.Handler())

	// API server
	apiServer := &http.Server{
		Addr:         ":8080",
		Handler:      handler,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Metrics server
	metricsServer := &http.Server{
		Addr:         ":9090",
		Handler:      metricsMux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Start servers
	go func() {
		logger.Info("API server starting", "addr", ":8080")
		if err := apiServer.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("API server error", "error", err)
			os.Exit(1)
		}
	}()

	go func() {
		logger.Info("Metrics server starting", "addr", ":9090")
		if err := metricsServer.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("Metrics server error", "error", err)
			os.Exit(1)
		}
	}()

	// Start traffic simulator
	trafficCtx, trafficCancel := context.WithCancel(context.Background())
	go simulateTraffic(trafficCtx, "http://localhost:8080", logger)

	logger.Info("service started",
		"api_addr", ":8080",
		"metrics_addr", ":9090",
		"endpoints", []string{
			"GET  /api/orders",
			"POST /api/orders",
			"GET  /api/users",
			"GET  /api/health",
			"GET  /api/slo",
			"GET  :9090/metrics",
		},
	)

	// Wait for shutdown signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	sig := <-sigCh

	logger.Info("shutdown signal received", "signal", sig.String())

	// Graceful shutdown
	trafficCancel()
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer shutdownCancel()

	if err := apiServer.Shutdown(shutdownCtx); err != nil {
		logger.Error("API server shutdown error", "error", err)
	}
	if err := metricsServer.Shutdown(shutdownCtx); err != nil {
		logger.Error("Metrics server shutdown error", "error", err)
	}

	logger.Info("service stopped")
}
```

### What This Example Gives You

After running this service and letting it generate traffic for a few minutes, scraping `/metrics` on port 9090 will show:

```
# RED metrics
orderapi_http_requests_total{method="GET",route="/api/orders",status_code="200"} 2847
orderapi_http_requests_total{method="POST",route="/api/orders",status_code="201"} 812
orderapi_http_requests_total{method="GET",route="/api/users",status_code="200"} 405
orderapi_http_request_duration_seconds_bucket{method="GET",route="/api/orders",le="0.025"} 2431

# SLI metrics
orderapi_sli_valid_total{sli="availability"} 4064
orderapi_sli_good_total{sli="availability"} 4052
orderapi_sli_valid_total{sli="latency"} 4052
orderapi_sli_good_total{sli="latency"} 3891

# Business metrics
orderapi_business_orders_created_total 812
orderapi_business_order_value_dollars_sum 203456.78

# Database metrics
orderapi_db_query_duration_seconds_bucket{operation="select",le="0.01"} 3100
orderapi_db_errors_total{operation="select"} 3

# Build info
orderapi_build_info{version="1.0.0",commit="abc123"} 1
```

### PromQL Alert Rules for This Service

```yaml
groups:
  - name: orderapi-slo-alerts
    rules:
      # Availability SLO: 99.9% over 30 days
      # Fast burn: 2% budget in 1 hour
      - alert: OrderAPI_AvailabilityBudget_FastBurn
        expr: |
          (
            1 - (rate(orderapi_sli_good_total{sli="availability"}[1h])
                 / rate(orderapi_sli_valid_total{sli="availability"}[1h]))
          ) / 0.001 > 14.4
          AND
          (
            1 - (rate(orderapi_sli_good_total{sli="availability"}[5m])
                 / rate(orderapi_sli_valid_total{sli="availability"}[5m]))
          ) / 0.001 > 14.4
        for: 2m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "Order API availability SLO: fast burn detected"
          description: "Burning >2% of 30-day error budget per hour"

      # Slow burn: 10% budget in 3 days
      - alert: OrderAPI_AvailabilityBudget_SlowBurn
        expr: |
          (
            1 - (rate(orderapi_sli_good_total{sli="availability"}[3d])
                 / rate(orderapi_sli_valid_total{sli="availability"}[3d]))
          ) / 0.001 > 1.0
          AND
          (
            1 - (rate(orderapi_sli_good_total{sli="availability"}[6h])
                 / rate(orderapi_sli_valid_total{sli="availability"}[6h]))
          ) / 0.001 > 1.0
        for: 15m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Order API availability SLO: slow burn detected"
          description: "Burning >10% of 30-day error budget per 3 days"

      # Latency SLO: 99% under 200ms over 30 days
      - alert: OrderAPI_LatencyBudget_FastBurn
        expr: |
          (
            1 - (rate(orderapi_sli_good_total{sli="latency"}[1h])
                 / rate(orderapi_sli_valid_total{sli="latency"}[1h]))
          ) / 0.01 > 14.4
          AND
          (
            1 - (rate(orderapi_sli_good_total{sli="latency"}[5m])
                 / rate(orderapi_sli_valid_total{sli="latency"}[5m]))
          ) / 0.01 > 14.4
        for: 2m
        labels:
          severity: critical
          slo: latency
        annotations:
          summary: "Order API latency SLO: fast burn detected"

      # High error rate (non-SLO, traditional alert)
      - alert: OrderAPI_HighErrorRate
        expr: |
          sum(rate(orderapi_http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(orderapi_http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Order API error rate >5%"
```

---

## 17. Key Takeaways

1. **Metrics complement logs and traces.** Logs tell you about individual events. Metrics tell you about aggregate system behavior. You need both.

2. **Use the four Prometheus metric types correctly.** Counters for things that only go up (request counts). Gauges for point-in-time values (active connections). Histograms for distributions (latency). Avoid summaries unless you have a specific reason.

3. **Watch label cardinality.** Every unique label combination creates a new time series. User IDs, request IDs, and email addresses as labels will destroy your Prometheus instance. Keep label cardinality under ~100 per label.

4. **Instrument with the RED method for services.** Rate, Errors, Duration -- these three signals cover most request-driven monitoring needs.

5. **Instrument with the USE method for resources.** Utilization, Saturation, Errors -- apply this to connection pools, worker pools, CPU, memory, and disk.

6. **SLIs should reflect user experience.** An availability SLI should count what users see (non-5xx responses), not what your health check reports. Exclude internal traffic, rate-limited requests, and other non-user events from SLI calculations.

7. **SLOs are contracts, not aspirations.** Set targets you can actually meet. Start with lower targets (99.5%) and tighten over time. Never set 100% -- it means zero error budget and zero ability to deploy.

8. **Error budgets drive engineering decisions.** When the budget is healthy, deploy freely. When it is depleted, freeze changes and invest in reliability. This creates a principled balance between velocity and reliability.

9. **Multi-window burn rate alerts replace threshold alerts.** Two windows (long for detection, short for confirmation) dramatically reduce false positives compared to "alert if error rate > X%."

10. **Separate your metrics endpoint from your API.** Run `/metrics` on a different port so it is not exposed through your public load balancer and so a metrics scrape does not compete with user traffic.

11. **Normalize paths in metric labels.** Map `/api/users/123` to `/api/users/:id` to prevent cardinality explosion. Use your router's matched pattern when available.

12. **Publish build info as a metric.** A gauge that is always 1 with version/commit labels lets you correlate deployments with metric changes in Grafana.

---

## 18. Practice Exercises

### Exercise 1: Implement Custom Histogram Buckets

The Prometheus default buckets (`DefBuckets`) are designed for HTTP request durations. Create a new histogram with custom buckets appropriate for:
- A payment processing service where latency ranges from 500ms to 30 seconds
- A cache lookup where latency ranges from 10 microseconds to 10 milliseconds
- A file upload endpoint where response sizes range from 1KB to 1GB

For each, write the `HistogramOpts` with appropriate bucket boundaries that give good resolution around your SLO targets.

### Exercise 2: Build a Rate Limiter with Metrics

Implement an HTTP rate limiter middleware that:
- Tracks the current rate of requests per client IP (as a metric)
- Tracks the number of rate-limited (429) responses (as a counter)
- Tracks the percentage of requests being rate-limited (a derived metric)
- Ensures rate-limited requests are excluded from availability SLI calculations

### Exercise 3: Implement the Four Golden Signals

Google's SRE book defines four golden signals: Latency, Traffic, Errors, and Saturation. Implement a middleware and background collector that exposes all four for an HTTP service. Specifically:
- **Latency**: Separate histograms for successful vs failed requests (failed requests that are fast are interesting differently from slow failures)
- **Traffic**: Requests per second, also broken down by endpoint
- **Errors**: 5xx errors, also track explicit application errors that return 200 but contain error payloads
- **Saturation**: Goroutine count, memory pressure, connection pool utilization

### Exercise 4: Multi-SLO Error Budget Calculator

Build a CLI tool that:
- Reads SLO definitions from a YAML file
- Accepts good/valid event counts as command-line arguments (simulating a Prometheus query)
- Computes error budget remaining, burn rate, and time to exhaustion for each SLO
- Outputs a formatted table showing the status of all SLOs
- Exits with a non-zero status code if any SLO is violated (for CI/CD integration)

### Exercise 5: Prometheus Collector for External Dependencies

Implement a `prometheus.Collector` that monitors external dependency health:
- HTTP health checks against 3 external services (configurable URLs)
- Track response time, success/failure, and consecutive failures
- Expose as gauges: `dependency_up` (0 or 1), `dependency_latency_seconds`, `dependency_consecutive_failures`
- Run health checks every 30 seconds in a background goroutine

### Exercise 6: SLO-Aware Deployment Gate

Build an HTTP service that acts as a deployment gate:
- Exposes a `/can-deploy` endpoint that returns `{"allowed": true}` or `{"allowed": false}`
- Queries the SLO dashboard (or uses in-memory SLO tracking) to check error budget
- Blocks deployments when any SLO's error budget is below 25%
- Allows deployments tagged as "reliability-fix" regardless of budget status
- Logs every deploy decision with SLO status details

### Exercise 7: Alertmanager Integration

Build a Go service that:
- Receives webhook alerts from Prometheus Alertmanager (POST /alerts)
- Parses the alert payload (alert name, severity, labels, annotations)
- Routes critical alerts to a simulated PagerDuty integration
- Routes warning alerts to a simulated Slack integration
- Tracks alert counts by severity as Prometheus metrics
- Implements alert deduplication (do not re-notify for the same alert within 1 hour)

### Exercise 8: Full Observability Stack with Docker Compose

Create a `docker-compose.yml` that runs:
1. Your instrumented Go API (from Section 16)
2. Prometheus (configured to scrape your API)
3. Grafana (with pre-configured dashboards for RED metrics and SLOs)
4. A load generator that produces realistic traffic patterns

Write the Prometheus configuration, Grafana dashboard JSON, and the Grafana datasource provisioning. The goal is a one-command (`docker-compose up`) observability stack.
