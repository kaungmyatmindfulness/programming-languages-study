# Chapter 43: Observability Pipelines

Production systems generate staggering volumes of telemetry data -- logs, metrics, and traces -- from hundreds of services running across thousands of containers. The raw data is useless if it sits in isolation on the machines that produced it. An **observability pipeline** is the infrastructure that collects that data at its source, processes it in flight (filtering, enriching, transforming, sampling), routes it to the right storage backends, and does all of this without dropping data or falling over when the load spikes.

This chapter draws on concepts from Chapters 19-24 of "Observability Engineering" by Charity Majors, Liz Fong-Jones, and George Miranda. Those chapters argue that observability is not just about having logs and metrics -- it is about building the **data infrastructure** that makes those signals queryable, cost-effective, and reliable. We will implement that infrastructure in Go, using Go's concurrency primitives to build a working telemetry pipeline from scratch.

By the end of this chapter, you will have built a complete mini observability pipeline: receivers that ingest logs, metrics, and traces; processors that filter, transform, enrich, and sample the data; a router that fans out to multiple exporters; buffering with backpressure; and pipeline self-monitoring. This is the same architecture used by production systems like the OpenTelemetry Collector, Vector, and Fluent Bit.

---

## Prerequisites

Before working through this chapter, you should be comfortable with:

- **Chapter 6: Structs and Interfaces** -- Every component in the pipeline is defined by an interface
- **Chapter 10: Goroutines and Concurrency Basics** -- The pipeline is inherently concurrent
- **Chapter 11: Channels and Select** -- Channels are the connective tissue between pipeline stages
- **Chapter 16: Context Package** -- Cancellation propagates through every stage
- **Chapter 17: Advanced Concurrency** -- errgroup, sync primitives, worker pools
- **Chapter 28: Structured Logging & Observability** -- The telemetry data we are transporting
- **Chapter 30: Graceful Shutdown** -- Draining pipelines cleanly on shutdown

---

## Table of Contents

1. [The Observability Pipeline](#1-the-observability-pipeline)
2. [Telemetry Data Model](#2-telemetry-data-model)
3. [Building a Log Collector](#3-building-a-log-collector)
4. [Building a Metrics Collector](#4-building-a-metrics-collector)
5. [Pipeline Processing](#5-pipeline-processing)
6. [Fan-Out Routing](#6-fan-out-routing)
7. [Buffering and Backpressure](#7-buffering-and-backpressure)
8. [Building a Log Aggregator](#8-building-a-log-aggregator)
9. [Trace Correlation](#9-trace-correlation)
10. [OpenTelemetry Collector Architecture](#10-opentelemetry-collector-architecture)
11. [Building a Mini OTel Collector](#11-building-a-mini-otel-collector)
12. [Metric Aggregation](#12-metric-aggregation)
13. [Data Reduction Strategies](#13-data-reduction-strategies)
14. [Pipeline Monitoring](#14-pipeline-monitoring)
15. [Real-World Example: Complete Observability Pipeline](#15-real-world-example-complete-observability-pipeline)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. The Observability Pipeline

### Why Pipelines Matter

Without a pipeline, every application sends telemetry directly to its storage backend. A Go service writes logs to Elasticsearch, pushes metrics to Prometheus, and exports traces to Jaeger. This works for one service. It falls apart at scale:

```
Without a Pipeline (direct shipping):
┌─────────┐    ┌──────────────┐
│ Service A├───►│ Elasticsearch│
└─────────┘    └──────────────┘
┌─────────┐    ┌──────────────┐
│ Service B├───►│ Elasticsearch│
└─────────┘    └──────────────┘
┌─────────┐    ┌──────────────┐
│ Service C├───►│ Elasticsearch│
└─────────┘    └──────────────┘

Problems:
- Every service needs credentials for every backend
- No central place to filter, sample, or transform
- Changing backends requires redeploying every service
- No buffering: if Elasticsearch is down, logs are lost
- No cost control: every log line ships at full fidelity
```

With a pipeline, services emit telemetry to a local agent, the agent forwards to a central pipeline, and the pipeline handles all the complexity:

```
With a Pipeline:
┌─────────┐     ┌───────┐     ┌──────────┐     ┌──────────────┐
│ Service A├────►│       │     │          ├────►│ Elasticsearch│
└─────────┘     │ Agent │────►│ Pipeline │     └──────────────┘
┌─────────┐     │       │     │          ├────►┌──────────────┐
│ Service B├────►│       │     │          │     │   Grafana    │
└─────────┘     └───────┘     │          ├────►└──────────────┘
┌─────────┐     ┌───────┐     │          │     ┌──────────────┐
│ Service C├────►│ Agent │────►│          ├────►│    S3 (cold) │
└─────────┘     └───────┘     └──────────┘     └──────────────┘

Benefits:
- Services only know about the local agent
- Central filtering, sampling, enrichment
- Backend changes require zero application changes
- Buffering protects against downstream outages
- Sampling and aggregation control costs
```

### The Four Stages

Every observability pipeline follows the same pattern:

1. **Collection** -- Gather telemetry from sources (files, HTTP endpoints, OTLP receivers, syslog)
2. **Processing** -- Filter, transform, enrich, sample, and aggregate the data in flight
3. **Routing** -- Decide where each piece of data should go based on its type, content, or labels
4. **Export** -- Deliver the data to storage backends (Elasticsearch, Prometheus, Jaeger, S3, etc.)

Let us build each stage in Go.

```go
package main

import (
	"context"
	"fmt"
)

// Stage represents a pipeline stage.
type Stage string

const (
	StageCollect  Stage = "collect"
	StageProcess  Stage = "process"
	StageRoute    Stage = "route"
	StageExport   Stage = "export"
)

// PipelineStage is the fundamental unit of a pipeline.
// Data flows: Collect -> Process -> Route -> Export
type PipelineStage interface {
	Name() string
	Stage() Stage
	Start(ctx context.Context) error
	Stop(ctx context.Context) error
}

func main() {
	stages := []struct {
		name        string
		stage       Stage
		description string
	}{
		{"file_tailer", StageCollect, "Tails log files and emits log entries"},
		{"metrics_scraper", StageCollect, "Scrapes /metrics endpoints on interval"},
		{"otlp_receiver", StageCollect, "Accepts OTLP over gRPC/HTTP"},
		{"filter", StageProcess, "Drops telemetry matching exclusion rules"},
		{"enricher", StageProcess, "Adds Kubernetes metadata to telemetry"},
		{"sampler", StageProcess, "Probabilistically drops traces to reduce volume"},
		{"type_router", StageRoute, "Routes logs to Loki, metrics to Prometheus, traces to Tempo"},
		{"elasticsearch", StageExport, "Sends logs to Elasticsearch"},
		{"prometheus_remote", StageExport, "Sends metrics via Prometheus remote write"},
	}

	fmt.Println("Observability Pipeline Stages:")
	fmt.Println("==============================")
	for _, s := range stages {
		fmt.Printf("  [%s] %-20s -- %s\n", s.stage, s.name, s.description)
	}
}
```

Output:

```
Observability Pipeline Stages:
==============================
  [collect] file_tailer          -- Tails log files and emits log entries
  [collect] metrics_scraper      -- Scrapes /metrics endpoints on interval
  [collect] otlp_receiver        -- Accepts OTLP over gRPC/HTTP
  [process] filter               -- Drops telemetry matching exclusion rules
  [process] enricher             -- Adds Kubernetes metadata to telemetry
  [process] sampler              -- Probabilistically drops traces to reduce volume
  [route] type_router            -- Routes logs to Loki, metrics to Prometheus, traces to Tempo
  [export] elasticsearch         -- Sends logs to Elasticsearch
  [export] prometheus_remote     -- Sends metrics via Prometheus remote write
```

---

## 2. Telemetry Data Model

### The Three Pillars as Go Types

Observability data comes in three forms: logs (events), metrics (measurements), and traces (request flows). A pipeline must handle all three with a unified data model. This is exactly what the OpenTelemetry Protocol (OTLP) does -- it defines a common envelope that wraps any telemetry type.

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

// TelemetryType identifies the kind of telemetry data.
type TelemetryType string

const (
	TelemetryLog    TelemetryType = "log"
	TelemetryMetric TelemetryType = "metric"
	TelemetryTrace  TelemetryType = "trace"
)

// Resource describes the entity that produced the telemetry.
// This follows the OpenTelemetry resource semantic conventions.
type Resource struct {
	ServiceName    string            `json:"service_name"`
	ServiceVersion string            `json:"service_version"`
	Environment    string            `json:"environment"`
	HostName       string            `json:"host_name"`
	Attributes     map[string]string `json:"attributes,omitempty"`
}

// TelemetryData is the universal envelope for all telemetry.
// Every piece of data flowing through the pipeline is wrapped in this struct.
type TelemetryData struct {
	Type      TelemetryType `json:"type"`
	Resource  Resource      `json:"resource"`
	Timestamp time.Time     `json:"timestamp"`
	Payload   any           `json:"payload"`
	TraceID   string        `json:"trace_id,omitempty"`
	SpanID    string        `json:"span_id,omitempty"`
}

// --- Log Payload ---

// LogSeverity represents the severity level of a log entry.
type LogSeverity string

const (
	SeverityDebug LogSeverity = "DEBUG"
	SeverityInfo  LogSeverity = "INFO"
	SeverityWarn  LogSeverity = "WARN"
	SeverityError LogSeverity = "ERROR"
	SeverityFatal LogSeverity = "FATAL"
)

// LogPayload is the payload for log-type telemetry.
type LogPayload struct {
	Severity   LogSeverity       `json:"severity"`
	Body       string            `json:"body"`
	Attributes map[string]string `json:"attributes,omitempty"`
}

// --- Metric Payload ---

// MetricType classifies the metric.
type MetricType string

const (
	MetricCounter   MetricType = "counter"
	MetricGauge     MetricType = "gauge"
	MetricHistogram MetricType = "histogram"
)

// MetricPayload is the payload for metric-type telemetry.
type MetricPayload struct {
	Name        string            `json:"name"`
	Description string            `json:"description,omitempty"`
	Unit        string            `json:"unit,omitempty"`
	MetricType  MetricType        `json:"metric_type"`
	Value       float64           `json:"value"`
	Labels      map[string]string `json:"labels,omitempty"`
}

// --- Trace Payload ---

// SpanKind describes the relationship between the span and its parent.
type SpanKind string

const (
	SpanKindServer   SpanKind = "server"
	SpanKindClient   SpanKind = "client"
	SpanKindInternal SpanKind = "internal"
)

// SpanStatus is the status of a span.
type SpanStatus string

const (
	SpanStatusOK    SpanStatus = "ok"
	SpanStatusError SpanStatus = "error"
)

// TracePayload is the payload for trace-type telemetry.
type TracePayload struct {
	TraceID      string            `json:"trace_id"`
	SpanID       string            `json:"span_id"`
	ParentSpanID string            `json:"parent_span_id,omitempty"`
	OperationName string           `json:"operation_name"`
	Kind         SpanKind          `json:"kind"`
	StartTime    time.Time         `json:"start_time"`
	EndTime      time.Time         `json:"end_time"`
	Status       SpanStatus        `json:"status"`
	Attributes   map[string]string `json:"attributes,omitempty"`
}

func main() {
	now := time.Now()
	resource := Resource{
		ServiceName:    "order-service",
		ServiceVersion: "1.4.2",
		Environment:    "production",
		HostName:       "pod-order-7b9f4",
		Attributes: map[string]string{
			"k8s.namespace": "default",
			"k8s.pod.uid":   "abc-123",
		},
	}

	// Create one of each telemetry type.
	samples := []TelemetryData{
		{
			Type:      TelemetryLog,
			Resource:  resource,
			Timestamp: now,
			TraceID:   "4bf92f3577b34da6a3ce929d0e0e4736",
			Payload: LogPayload{
				Severity: SeverityInfo,
				Body:     "Order created successfully",
				Attributes: map[string]string{
					"order_id":    "ORD-9842",
					"customer_id": "CUST-111",
				},
			},
		},
		{
			Type:      TelemetryMetric,
			Resource:  resource,
			Timestamp: now,
			Payload: MetricPayload{
				Name:       "http_requests_total",
				MetricType: MetricCounter,
				Value:      1,
				Labels: map[string]string{
					"method": "POST",
					"path":   "/api/orders",
					"status": "201",
				},
			},
		},
		{
			Type:      TelemetryTrace,
			Resource:  resource,
			Timestamp: now,
			TraceID:   "4bf92f3577b34da6a3ce929d0e0e4736",
			SpanID:    "00f067aa0ba902b7",
			Payload: TracePayload{
				TraceID:       "4bf92f3577b34da6a3ce929d0e0e4736",
				SpanID:        "00f067aa0ba902b7",
				OperationName: "POST /api/orders",
				Kind:          SpanKindServer,
				StartTime:     now.Add(-50 * time.Millisecond),
				EndTime:       now,
				Status:        SpanStatusOK,
				Attributes: map[string]string{
					"http.method":      "POST",
					"http.status_code": "201",
					"http.url":         "/api/orders",
				},
			},
		},
	}

	for _, sample := range samples {
		data, err := json.MarshalIndent(sample, "", "  ")
		if err != nil {
			fmt.Printf("Error: %v\n", err)
			continue
		}
		fmt.Printf("--- %s ---\n%s\n\n", sample.Type, string(data))
	}
}
```

Output (timestamps and IDs will vary):

```
--- log ---
{
  "type": "log",
  "resource": {
    "service_name": "order-service",
    "service_version": "1.4.2",
    "environment": "production",
    "host_name": "pod-order-7b9f4",
    "attributes": {
      "k8s.namespace": "default",
      "k8s.pod.uid": "abc-123"
    }
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {
    "severity": "INFO",
    "body": "Order created successfully",
    "attributes": {
      "customer_id": "CUST-111",
      "order_id": "ORD-9842"
    }
  },
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}

--- metric ---
{
  "type": "metric",
  "resource": { ... },
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {
    "name": "http_requests_total",
    "metric_type": "counter",
    "value": 1,
    "labels": {
      "method": "POST",
      "path": "/api/orders",
      "status": "201"
    }
  }
}

--- trace ---
{
  "type": "trace",
  "resource": { ... },
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id": "00f067aa0ba902b7",
    "operation_name": "POST /api/orders",
    "kind": "server",
    "start_time": "2025-01-15T10:29:59.95Z",
    "end_time": "2025-01-15T10:30:00Z",
    "status": "ok",
    "attributes": { ... }
  },
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

### Why a Unified Envelope

The `TelemetryData` struct is the currency of the pipeline. Every processor, router, and exporter works with this single type. This means you can build a processor that enriches logs, metrics, and traces with Kubernetes metadata -- one implementation, not three. It also means your router can make decisions based on `data.Type`, `data.Resource.Environment`, or any attribute in the payload.

---

## 3. Building a Log Collector

### Tailing Files

The most fundamental collection task is tailing log files -- reading new lines as they are appended, just like `tail -f`. A production log collector must track its position in the file, survive restarts, and handle log rotation.

```go
package main

import (
	"bufio"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"os"
	"strings"
	"sync"
	"time"
)

// TelemetryType identifies the kind of telemetry data.
type TelemetryType string

const (
	TelemetryLog TelemetryType = "log"
)

// Resource describes the entity that produced the telemetry.
type Resource struct {
	ServiceName    string            `json:"service_name"`
	ServiceVersion string            `json:"service_version"`
	Environment    string            `json:"environment"`
	HostName       string            `json:"host_name"`
	Attributes     map[string]string `json:"attributes,omitempty"`
}

// TelemetryData is the universal envelope.
type TelemetryData struct {
	Type      TelemetryType `json:"type"`
	Resource  Resource      `json:"resource"`
	Timestamp time.Time     `json:"timestamp"`
	Payload   any           `json:"payload"`
}

// LogSeverity represents log severity.
type LogSeverity string

const (
	SeverityInfo  LogSeverity = "INFO"
	SeverityWarn  LogSeverity = "WARN"
	SeverityError LogSeverity = "ERROR"
)

// LogPayload carries log data.
type LogPayload struct {
	Severity   LogSeverity       `json:"severity"`
	Body       string            `json:"body"`
	Attributes map[string]string `json:"attributes,omitempty"`
}

// LogEntry represents a single parsed log line.
type LogEntry struct {
	Timestamp time.Time
	Severity  LogSeverity
	Message   string
	Fields    map[string]string
	Raw       string
}

// LogSource is the interface for anything that produces log entries.
type LogSource interface {
	Tail(ctx context.Context) <-chan LogEntry
	Name() string
}

// FileSource tails a single log file.
type FileSource struct {
	path     string
	position int64
	mu       sync.Mutex
}

// NewFileSource creates a file source starting from the end of the file.
func NewFileSource(path string) *FileSource {
	return &FileSource{path: path}
}

func (fs *FileSource) Name() string {
	return fmt.Sprintf("file:%s", fs.path)
}

// Tail opens the file, seeks to the last known position, and emits new lines.
func (fs *FileSource) Tail(ctx context.Context) <-chan LogEntry {
	ch := make(chan LogEntry, 100)

	go func() {
		defer close(ch)

		file, err := os.Open(fs.path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Failed to open %s: %v\n", fs.path, err)
			return
		}
		defer file.Close()

		// Seek to last known position (0 = start of file for first run).
		fs.mu.Lock()
		pos := fs.position
		fs.mu.Unlock()

		if pos > 0 {
			if _, err := file.Seek(pos, io.SeekStart); err != nil {
				fmt.Fprintf(os.Stderr, "Seek failed: %v\n", err)
				return
			}
		}

		scanner := bufio.NewScanner(file)
		pollInterval := 250 * time.Millisecond

		for {
			select {
			case <-ctx.Done():
				return
			default:
			}

			if scanner.Scan() {
				line := scanner.Text()
				if line == "" {
					continue
				}

				entry := parseLine(line)

				// Update position.
				newPos, _ := file.Seek(0, io.SeekCurrent)
				fs.mu.Lock()
				fs.position = newPos
				fs.mu.Unlock()

				select {
				case ch <- entry:
				case <-ctx.Done():
					return
				}
			} else {
				// No new data -- wait and retry.
				select {
				case <-time.After(pollInterval):
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	return ch
}

// parseLine parses a structured JSON log line, falling back to plain text.
func parseLine(line string) LogEntry {
	entry := LogEntry{
		Timestamp: time.Now(),
		Severity:  SeverityInfo,
		Raw:       line,
	}

	// Try JSON first.
	var parsed map[string]any
	if err := json.Unmarshal([]byte(line), &parsed); err == nil {
		if msg, ok := parsed["msg"].(string); ok {
			entry.Message = msg
		} else if msg, ok := parsed["message"].(string); ok {
			entry.Message = msg
		}
		if lvl, ok := parsed["level"].(string); ok {
			entry.Severity = mapSeverity(lvl)
		}
		if ts, ok := parsed["time"].(string); ok {
			if t, err := time.Parse(time.RFC3339, ts); err == nil {
				entry.Timestamp = t
			}
		}
		entry.Fields = make(map[string]string)
		for k, v := range parsed {
			if k != "msg" && k != "message" && k != "level" && k != "time" {
				entry.Fields[k] = fmt.Sprintf("%v", v)
			}
		}
	} else {
		// Plain text fallback.
		entry.Message = line
	}

	return entry
}

func mapSeverity(level string) LogSeverity {
	switch strings.ToUpper(level) {
	case "WARN", "WARNING":
		return SeverityWarn
	case "ERROR", "ERR":
		return SeverityError
	default:
		return SeverityInfo
	}
}

// Exporter sends telemetry data somewhere.
type Exporter interface {
	Export(ctx context.Context, data *TelemetryData) error
	Name() string
}

// StdoutExporter prints telemetry to stdout.
type StdoutExporter struct{}

func (e *StdoutExporter) Name() string { return "stdout" }

func (e *StdoutExporter) Export(ctx context.Context, data *TelemetryData) error {
	out, _ := json.Marshal(data)
	fmt.Println(string(out))
	return nil
}

// LogCollector ties together sources, processing, and export.
type LogCollector struct {
	sources  []LogSource
	exporter Exporter
	resource Resource
}

func NewLogCollector(resource Resource, exporter Exporter, sources ...LogSource) *LogCollector {
	return &LogCollector{
		sources:  sources,
		exporter: exporter,
		resource: resource,
	}
}

// Run starts tailing all sources and forwarding to the exporter.
func (lc *LogCollector) Run(ctx context.Context) error {
	var wg sync.WaitGroup

	for _, src := range lc.sources {
		wg.Add(1)
		go func(s LogSource) {
			defer wg.Done()
			ch := s.Tail(ctx)
			for entry := range ch {
				td := &TelemetryData{
					Type:      TelemetryLog,
					Resource:  lc.resource,
					Timestamp: entry.Timestamp,
					Payload: LogPayload{
						Severity:   entry.Severity,
						Body:       entry.Message,
						Attributes: entry.Fields,
					},
				}
				if err := lc.exporter.Export(ctx, td); err != nil {
					fmt.Fprintf(os.Stderr, "Export error: %v\n", err)
				}
			}
		}(src)
	}

	wg.Wait()
	return nil
}

func main() {
	// Create a temporary log file to tail.
	tmpFile, err := os.CreateTemp("", "app-*.log")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create temp file: %v\n", err)
		return
	}
	defer os.Remove(tmpFile.Name())

	// Write some log lines.
	logLines := []string{
		`{"time":"2025-01-15T10:00:01Z","level":"INFO","msg":"Server starting","port":8080}`,
		`{"time":"2025-01-15T10:00:02Z","level":"INFO","msg":"Connected to database","host":"db.internal"}`,
		`{"time":"2025-01-15T10:00:05Z","level":"WARN","msg":"Slow query detected","duration_ms":1500,"query":"SELECT * FROM orders"}`,
		`{"time":"2025-01-15T10:00:10Z","level":"ERROR","msg":"Connection refused","service":"payment-api","retry":3}`,
	}
	for _, line := range logLines {
		fmt.Fprintln(tmpFile, line)
	}
	tmpFile.Close()

	// Set up the collector.
	resource := Resource{
		ServiceName:    "order-service",
		ServiceVersion: "1.4.2",
		Environment:    "production",
		HostName:       "pod-order-7b9f4",
	}

	source := NewFileSource(tmpFile.Name())
	exporter := &StdoutExporter{}
	collector := NewLogCollector(resource, exporter, source)

	// Run with a timeout so the program exits.
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	fmt.Println("=== Log Collector Output ===")
	collector.Run(ctx)
	fmt.Println("=== Done ===")
}
```

### How It Works

1. `FileSource.Tail` opens the file, seeks to the last known position, and scans for new lines.
2. When no new lines are available, it polls every 250ms (production collectors use `fsnotify` for efficiency).
3. Each line is parsed -- JSON structured logs are decomposed into fields; plain text is kept as-is.
4. The `LogCollector` wraps each `LogEntry` in a `TelemetryData` envelope and hands it to the exporter.
5. Multiple sources can run concurrently, each in its own goroutine.
6. The `context.Context` propagates cancellation, ensuring clean shutdown.

---

## 4. Building a Metrics Collector

### Pull-Based Scraping

Prometheus popularized the pull model: services expose a `/metrics` endpoint, and a scraper periodically fetches it. Let us build a scraper that collects metrics from multiple targets.

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"net/http"
	"strconv"
	"strings"
	"sync"
	"time"
)

// MetricType classifies the metric.
type MetricType string

const (
	MetricCounter   MetricType = "counter"
	MetricGauge     MetricType = "gauge"
	MetricHistogram MetricType = "histogram"
)

// Metric is a single scraped metric data point.
type Metric struct {
	Name      string            `json:"name"`
	Type      MetricType        `json:"type"`
	Value     float64           `json:"value"`
	Labels    map[string]string `json:"labels,omitempty"`
	Timestamp time.Time         `json:"timestamp"`
}

// ScrapeTarget identifies a metrics endpoint to scrape.
type ScrapeTarget struct {
	URL    string            `json:"url"`
	Labels map[string]string `json:"labels,omitempty"` // extra labels added to all metrics
}

// MetricsStore receives scraped metrics.
type MetricsStore interface {
	Store(metrics []Metric) error
}

// InMemoryStore holds metrics in memory for demonstration.
type InMemoryStore struct {
	mu      sync.Mutex
	metrics []Metric
}

func (s *InMemoryStore) Store(metrics []Metric) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.metrics = append(s.metrics, metrics...)
	return nil
}

func (s *InMemoryStore) All() []Metric {
	s.mu.Lock()
	defer s.mu.Unlock()
	result := make([]Metric, len(s.metrics))
	copy(result, s.metrics)
	return result
}

// MetricsScraper scrapes metrics from targets on an interval.
type MetricsScraper struct {
	targets  []ScrapeTarget
	interval time.Duration
	store    MetricsStore
	client   *http.Client
}

func NewMetricsScraper(interval time.Duration, store MetricsStore, targets ...ScrapeTarget) *MetricsScraper {
	return &MetricsScraper{
		targets:  targets,
		interval: interval,
		store:    store,
		client: &http.Client{
			Timeout: 5 * time.Second,
		},
	}
}

// Scrape fetches metrics from a single target.
func (s *MetricsScraper) Scrape(ctx context.Context, target ScrapeTarget) ([]Metric, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, target.URL, nil)
	if err != nil {
		return nil, fmt.Errorf("creating request: %w", err)
	}

	resp, err := s.client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("scraping %s: %w", target.URL, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("scraping %s: status %d", target.URL, resp.StatusCode)
	}

	// Parse Prometheus exposition format (simplified).
	var metrics []Metric
	scanner := bufio.NewScanner(resp.Body)
	now := time.Now()
	metricTypes := make(map[string]MetricType)

	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())

		// Skip empty lines.
		if line == "" {
			continue
		}

		// Parse TYPE comments.
		if strings.HasPrefix(line, "# TYPE ") {
			parts := strings.Fields(line)
			if len(parts) >= 4 {
				metricTypes[parts[2]] = MetricType(parts[3])
			}
			continue
		}

		// Skip other comments.
		if strings.HasPrefix(line, "#") {
			continue
		}

		// Parse metric line: metric_name{label="value"} 123.45
		m, err := parsePrometheusLine(line, metricTypes, now)
		if err != nil {
			continue // skip unparseable lines
		}

		// Merge target labels.
		for k, v := range target.Labels {
			if _, exists := m.Labels[k]; !exists {
				m.Labels[k] = v
			}
		}

		metrics = append(metrics, m)
	}

	return metrics, scanner.Err()
}

// parsePrometheusLine parses a single Prometheus exposition format line.
func parsePrometheusLine(line string, types map[string]MetricType, ts time.Time) (Metric, error) {
	m := Metric{
		Timestamp: ts,
		Labels:    make(map[string]string),
	}

	// Split into name{labels} value
	var nameAndLabels, valueStr string

	// Check for labels.
	if idx := strings.Index(line, "{"); idx >= 0 {
		m.Name = line[:idx]
		endIdx := strings.Index(line, "}")
		if endIdx < 0 {
			return m, fmt.Errorf("malformed labels in: %s", line)
		}
		labelStr := line[idx+1 : endIdx]
		valueStr = strings.TrimSpace(line[endIdx+1:])

		// Parse labels.
		for _, pair := range splitLabels(labelStr) {
			parts := strings.SplitN(pair, "=", 2)
			if len(parts) == 2 {
				m.Labels[parts[0]] = strings.Trim(parts[1], `"`)
			}
		}
	} else {
		parts := strings.Fields(line)
		if len(parts) < 2 {
			return m, fmt.Errorf("malformed line: %s", line)
		}
		nameAndLabels = parts[0]
		valueStr = parts[1]
		m.Name = nameAndLabels
	}

	val, err := strconv.ParseFloat(valueStr, 64)
	if err != nil {
		return m, fmt.Errorf("parsing value: %w", err)
	}
	m.Value = val

	// Look up type.
	baseName := m.Name
	// Strip suffixes like _total, _bucket, _sum, _count.
	for _, suffix := range []string{"_total", "_bucket", "_sum", "_count"} {
		baseName = strings.TrimSuffix(baseName, suffix)
	}
	if t, ok := types[baseName]; ok {
		m.Type = t
	} else if t, ok := types[m.Name]; ok {
		m.Type = t
	} else {
		m.Type = MetricGauge // default
	}

	return m, nil
}

// splitLabels splits a Prometheus label string, respecting quoted values.
func splitLabels(s string) []string {
	var result []string
	var current strings.Builder
	inQuotes := false

	for _, ch := range s {
		switch {
		case ch == '"':
			inQuotes = !inQuotes
			current.WriteRune(ch)
		case ch == ',' && !inQuotes:
			result = append(result, strings.TrimSpace(current.String()))
			current.Reset()
		default:
			current.WriteRune(ch)
		}
	}
	if current.Len() > 0 {
		result = append(result, strings.TrimSpace(current.String()))
	}
	return result
}

// Run starts the periodic scrape loop.
func (s *MetricsScraper) Run(ctx context.Context) {
	// Scrape immediately on start.
	s.scrapeAll(ctx)

	ticker := time.NewTicker(s.interval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			s.scrapeAll(ctx)
		}
	}
}

func (s *MetricsScraper) scrapeAll(ctx context.Context) {
	var wg sync.WaitGroup

	for _, target := range s.targets {
		wg.Add(1)
		go func(t ScrapeTarget) {
			defer wg.Done()
			metrics, err := s.Scrape(ctx, t)
			if err != nil {
				fmt.Printf("[scraper] error scraping %s: %v\n", t.URL, err)
				return
			}
			if err := s.store.Store(metrics); err != nil {
				fmt.Printf("[scraper] error storing metrics from %s: %v\n", t.URL, err)
			}
			fmt.Printf("[scraper] collected %d metrics from %s\n", len(metrics), t.URL)
		}(target)
	}

	wg.Wait()
}

func main() {
	// Start a fake metrics endpoint.
	mux := http.NewServeMux()
	mux.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/plain")
		fmt.Fprint(w, `# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/api/users",status="200"} 1027
http_requests_total{method="POST",path="/api/orders",status="201"} 342
http_requests_total{method="GET",path="/api/orders",status="500"} 5
# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.01"} 500
http_request_duration_seconds_bucket{le="0.05"} 900
http_request_duration_seconds_bucket{le="0.1"} 980
http_request_duration_seconds_bucket{le="0.5"} 1000
http_request_duration_seconds_sum 45.2
http_request_duration_seconds_count 1000
# HELP process_resident_memory_bytes Resident memory size
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 5.2428e+07
# HELP go_goroutines Number of goroutines
# TYPE go_goroutines gauge
go_goroutines 42
`)
	})

	server := &http.Server{Addr: ":19090", Handler: mux}
	go server.ListenAndServe()
	defer server.Close()

	// Give the server a moment to start.
	time.Sleep(50 * time.Millisecond)

	store := &InMemoryStore{}
	scraper := NewMetricsScraper(
		5*time.Second,
		store,
		ScrapeTarget{
			URL: "http://localhost:19090/metrics",
			Labels: map[string]string{
				"job":      "order-service",
				"instance": "localhost:19090",
			},
		},
	)

	// Run for a short duration.
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	scraper.Run(ctx)

	// Print collected metrics.
	fmt.Println("\n=== Collected Metrics ===")
	for _, m := range store.All() {
		fmt.Printf("  %-45s type=%-10s value=%.2f labels=%v\n", m.Name, m.Type, m.Value, m.Labels)
	}
}
```

Output:

```
[scraper] collected 10 metrics from http://localhost:19090/metrics

=== Collected Metrics ===
  http_requests_total                           type=counter    value=1027.00 labels=map[instance:localhost:19090 job:order-service method:GET path:/api/users status:200]
  http_requests_total                           type=counter    value=342.00  labels=map[instance:localhost:19090 job:order-service method:POST path:/api/orders status:201]
  http_requests_total                           type=counter    value=5.00    labels=map[instance:localhost:19090 job:order-service method:GET path:/api/orders status:500]
  http_request_duration_seconds_bucket          type=histogram  value=500.00  labels=map[instance:localhost:19090 job:order-service le:0.01]
  ...
  process_resident_memory_bytes                 type=gauge      value=52428000.00 labels=map[instance:localhost:19090 job:order-service]
  go_goroutines                                 type=gauge      value=42.00   labels=map[instance:localhost:19090 job:order-service]
```

### Push vs Pull

Pull (scraping) is what Prometheus uses. The collector periodically fetches metrics. Push is what StatsD and OTLP use. The application sends metrics to the collector whenever it wants. Both work. Pull is simpler for discovery (you just need a list of targets). Push is better for short-lived processes (batch jobs, lambdas) that may exit before the next scrape.

---

## 5. Pipeline Processing

### The Processor Interface

Every processing step implements the same interface. It receives a `TelemetryData`, possibly modifies it, and returns it. Returning `nil` means "drop this data." This design lets you chain processors into a pipeline.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"math/big"
	"regexp"
	"strings"
	"time"
)

// --- Data model (reused from section 2) ---

type TelemetryType string

const (
	TelemetryLog    TelemetryType = "log"
	TelemetryMetric TelemetryType = "metric"
	TelemetryTrace  TelemetryType = "trace"
)

type Resource struct {
	ServiceName    string            `json:"service_name"`
	ServiceVersion string            `json:"service_version"`
	Environment    string            `json:"environment"`
	HostName       string            `json:"host_name"`
	Attributes     map[string]string `json:"attributes,omitempty"`
}

type TelemetryData struct {
	Type      TelemetryType `json:"type"`
	Resource  Resource      `json:"resource"`
	Timestamp time.Time     `json:"timestamp"`
	Payload   any           `json:"payload"`
	TraceID   string        `json:"trace_id,omitempty"`
	SpanID    string        `json:"span_id,omitempty"`
}

type LogSeverity string

const (
	SeverityDebug LogSeverity = "DEBUG"
	SeverityInfo  LogSeverity = "INFO"
	SeverityWarn  LogSeverity = "WARN"
	SeverityError LogSeverity = "ERROR"
)

type LogPayload struct {
	Severity   LogSeverity       `json:"severity"`
	Body       string            `json:"body"`
	Attributes map[string]string `json:"attributes,omitempty"`
}

// --- Processor interface ---

// Processor transforms telemetry data in flight.
// Return nil to drop the data. Return an error to signal failure.
type Processor interface {
	Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error)
	Name() string
}

// --- Filter Processor ---

// FilterCondition defines a rule for dropping telemetry.
type FilterCondition struct {
	Field    string // e.g., "resource.service_name", "payload.severity", "type"
	Operator string // "equals", "contains", "matches"
	Value    string
}

// FilterProcessor drops telemetry that matches any condition.
type FilterProcessor struct {
	conditions []FilterCondition
	mode       string // "drop" = drop matching, "keep" = keep only matching
}

func NewFilterProcessor(mode string, conditions ...FilterCondition) *FilterProcessor {
	return &FilterProcessor{conditions: conditions, mode: mode}
}

func (f *FilterProcessor) Name() string { return "filter" }

func (f *FilterProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	matches := f.matchesAny(data)

	switch f.mode {
	case "drop":
		if matches {
			return nil, nil // drop
		}
		return data, nil
	case "keep":
		if matches {
			return data, nil
		}
		return nil, nil // drop
	default:
		return data, nil
	}
}

func (f *FilterProcessor) matchesAny(data *TelemetryData) bool {
	for _, cond := range f.conditions {
		fieldValue := extractField(data, cond.Field)
		switch cond.Operator {
		case "equals":
			if fieldValue == cond.Value {
				return true
			}
		case "contains":
			if strings.Contains(fieldValue, cond.Value) {
				return true
			}
		case "matches":
			if matched, _ := regexp.MatchString(cond.Value, fieldValue); matched {
				return true
			}
		}
	}
	return false
}

// extractField reads a field from TelemetryData using a dot-separated path.
func extractField(data *TelemetryData, field string) string {
	switch field {
	case "type":
		return string(data.Type)
	case "resource.service_name":
		return data.Resource.ServiceName
	case "resource.environment":
		return data.Resource.Environment
	case "resource.host_name":
		return data.Resource.HostName
	case "trace_id":
		return data.TraceID
	}

	// Check payload-specific fields.
	if lp, ok := data.Payload.(LogPayload); ok {
		switch field {
		case "payload.severity":
			return string(lp.Severity)
		case "payload.body":
			return lp.Body
		}
		if strings.HasPrefix(field, "payload.attributes.") {
			key := strings.TrimPrefix(field, "payload.attributes.")
			return lp.Attributes[key]
		}
	}

	return ""
}

// --- Transform Processor ---

// TransformRule defines a field transformation.
type TransformRule struct {
	Action string // "set_field", "rename_field", "delete_field", "lowercase", "uppercase"
	Field  string
	Value  string // for set_field
}

// TransformProcessor modifies telemetry fields.
type TransformProcessor struct {
	rules []TransformRule
}

func NewTransformProcessor(rules ...TransformRule) *TransformProcessor {
	return &TransformProcessor{rules: rules}
}

func (t *TransformProcessor) Name() string { return "transform" }

func (t *TransformProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	for _, rule := range t.rules {
		switch rule.Action {
		case "set_field":
			applySetField(data, rule.Field, rule.Value)
		case "lowercase":
			val := extractField(data, rule.Field)
			applySetField(data, rule.Field, strings.ToLower(val))
		case "uppercase":
			val := extractField(data, rule.Field)
			applySetField(data, rule.Field, strings.ToUpper(val))
		}
	}
	return data, nil
}

func applySetField(data *TelemetryData, field, value string) {
	switch field {
	case "resource.environment":
		data.Resource.Environment = value
	case "resource.host_name":
		data.Resource.HostName = value
	}
	if data.Resource.Attributes == nil {
		data.Resource.Attributes = make(map[string]string)
	}
	if strings.HasPrefix(field, "resource.attributes.") {
		key := strings.TrimPrefix(field, "resource.attributes.")
		data.Resource.Attributes[key] = value
	}
}

// --- Enrich Processor ---

// LookupSource provides key-value lookups for enrichment.
type LookupSource interface {
	Lookup(key string) (map[string]string, bool)
}

// StaticLookup is a simple in-memory lookup table.
type StaticLookup struct {
	data map[string]map[string]string
}

func (s *StaticLookup) Lookup(key string) (map[string]string, bool) {
	val, ok := s.data[key]
	return val, ok
}

// EnrichProcessor adds metadata to telemetry based on lookups.
type EnrichProcessor struct {
	lookupField string       // field to use as lookup key
	source      LookupSource
	targetField string       // prefix for enrichment fields
}

func NewEnrichProcessor(lookupField, targetField string, source LookupSource) *EnrichProcessor {
	return &EnrichProcessor{
		lookupField: lookupField,
		source:      source,
		targetField: targetField,
	}
}

func (e *EnrichProcessor) Name() string { return "enrich" }

func (e *EnrichProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	key := extractField(data, e.lookupField)
	if key == "" {
		return data, nil
	}

	enrichment, found := e.source.Lookup(key)
	if !found {
		return data, nil
	}

	if data.Resource.Attributes == nil {
		data.Resource.Attributes = make(map[string]string)
	}
	for k, v := range enrichment {
		data.Resource.Attributes[e.targetField+"."+k] = v
	}

	return data, nil
}

// --- Sampling Processor ---

// SamplingProcessor probabilistically drops telemetry.
type SamplingProcessor struct {
	rate float64 // 0.0 to 1.0; 0.1 = keep 10%
}

func NewSamplingProcessor(rate float64) *SamplingProcessor {
	return &SamplingProcessor{rate: rate}
}

func (s *SamplingProcessor) Name() string { return fmt.Sprintf("sampler(%.0f%%)", s.rate*100) }

func (s *SamplingProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	// Always keep errors -- never sample out errors.
	if lp, ok := data.Payload.(LogPayload); ok {
		if lp.Severity == SeverityError {
			return data, nil
		}
	}

	// Use crypto/rand for unbiased sampling.
	n, err := rand.Int(rand.Reader, big.NewInt(1000))
	if err != nil {
		return data, nil // on error, keep the data
	}
	threshold := int64(s.rate * 1000)
	if n.Int64() < threshold {
		return data, nil // keep
	}
	return nil, nil // drop
}

// --- Pipeline ---

// Pipeline chains multiple processors together.
type Pipeline struct {
	processors []Processor
}

func NewPipeline(processors ...Processor) *Pipeline {
	return &Pipeline{processors: processors}
}

func (p *Pipeline) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	for _, proc := range p.processors {
		var err error
		data, err = proc.Process(ctx, data)
		if err != nil {
			return nil, fmt.Errorf("processor %s: %w", proc.Name(), err)
		}
		if data == nil {
			return nil, nil // filtered out
		}
	}
	return data, nil
}

func main() {
	// Build a pipeline: filter -> enrich -> transform -> sample
	pipeline := NewPipeline(
		// Drop debug logs.
		NewFilterProcessor("drop",
			FilterCondition{Field: "payload.severity", Operator: "equals", Value: "DEBUG"},
		),
		// Enrich with team ownership data.
		NewEnrichProcessor("resource.service_name", "team", &StaticLookup{
			data: map[string]map[string]string{
				"order-service":   {"name": "commerce", "oncall": "alice@example.com", "tier": "1"},
				"payment-service": {"name": "payments", "oncall": "bob@example.com", "tier": "1"},
				"email-service":   {"name": "notifications", "oncall": "carol@example.com", "tier": "3"},
			},
		}),
		// Normalize environment names.
		NewTransformProcessor(
			TransformRule{Action: "lowercase", Field: "resource.environment"},
		),
		// Sample 50% of non-error traffic.
		NewSamplingProcessor(0.5),
	)

	// Create test telemetry.
	testData := []TelemetryData{
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "order-service", Environment: "PRODUCTION", HostName: "pod-1"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityDebug, Body: "Entering handler", Attributes: map[string]string{}},
		},
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "order-service", Environment: "PRODUCTION", HostName: "pod-1"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityInfo, Body: "Order created", Attributes: map[string]string{"order_id": "ORD-123"}},
		},
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "payment-service", Environment: "STAGING", HostName: "pod-2"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityError, Body: "Payment failed: card declined", Attributes: map[string]string{"customer_id": "C-456"}},
		},
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "email-service", Environment: "Production", HostName: "pod-3"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityInfo, Body: "Email sent", Attributes: map[string]string{"to": "user@example.com"}},
		},
	}

	ctx := context.Background()
	fmt.Println("=== Pipeline Processing ===")
	fmt.Printf("Input: %d telemetry items\n\n", len(testData))

	kept := 0
	for i, td := range testData {
		result, err := pipeline.Process(ctx, &td)
		if err != nil {
			fmt.Printf("[%d] ERROR: %v\n", i, err)
			continue
		}
		if result == nil {
			lp, _ := td.Payload.(LogPayload)
			fmt.Printf("[%d] DROPPED: severity=%s body=%q\n", i, lp.Severity, lp.Body)
			continue
		}
		kept++
		out, _ := json.MarshalIndent(result, "    ", "  ")
		fmt.Printf("[%d] KEPT:\n    %s\n", i, string(out))
	}

	// Generate a random hex string for visual variety.
	b := make([]byte, 4)
	rand.Read(b)
	_ = hex.EncodeToString(b)

	fmt.Printf("\nResult: %d/%d items passed through the pipeline\n", kept, len(testData))
}
```

Output (sampling is random, so exact results vary):

```
=== Pipeline Processing ===
Input: 4 telemetry items

[0] DROPPED: severity=DEBUG body="Entering handler"
[1] KEPT:
    {
      "type": "log",
      "resource": {
        "service_name": "order-service",
        "environment": "production",
        "host_name": "pod-1",
        "attributes": {
          "team.name": "commerce",
          "team.oncall": "alice@example.com",
          "team.tier": "1"
        }
      },
      ...
    }
[2] KEPT:
    {
      "type": "log",
      "resource": {
        "service_name": "payment-service",
        "environment": "staging",
        "host_name": "pod-2",
        "attributes": {
          "team.name": "payments",
          "team.oncall": "bob@example.com",
          "team.tier": "1"
        }
      },
      ...
    }
[3] DROPPED: severity=INFO body="Email sent"

Result: 2/4 items passed through the pipeline
```

### Key Design Points

- **Filter first**: The filter processor runs before enrichment and sampling. No point enriching data you are going to drop.
- **Errors are never sampled**: The sampling processor always keeps error-level logs. You never want to lose error signals.
- **Processors are composable**: Each processor does one thing. The pipeline chains them. You can rearrange, add, or remove processors without touching the others.
- **Nil means "drop"**: Returning `nil` from a processor signals that the data should be dropped. The pipeline short-circuits immediately.

---

## 6. Fan-Out Routing

### Sending Data to Multiple Destinations

After processing, data needs to go somewhere -- often to multiple destinations. Error logs go to PagerDuty and Elasticsearch. Metrics go to Prometheus. Traces go to Jaeger. A router matches data against rules and fans out to the appropriate exporters concurrently.

```go
package main

import (
	"context"
	"fmt"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// --- Minimal data model ---

type TelemetryType string

const (
	TelemetryLog    TelemetryType = "log"
	TelemetryMetric TelemetryType = "metric"
	TelemetryTrace  TelemetryType = "trace"
)

type Resource struct {
	ServiceName string
	Environment string
}

type TelemetryData struct {
	Type      TelemetryType
	Resource  Resource
	Timestamp time.Time
	Payload   any
}

type LogSeverity string

const (
	SeverityInfo  LogSeverity = "INFO"
	SeverityError LogSeverity = "ERROR"
)

type LogPayload struct {
	Severity   LogSeverity
	Body       string
	Attributes map[string]string
}

// --- Exporter interface ---

type Exporter interface {
	Export(ctx context.Context, data *TelemetryData) error
	Name() string
}

// CountingExporter counts exports for demonstration.
type CountingExporter struct {
	name  string
	count atomic.Int64
	mu    sync.Mutex
	last  []string
}

func NewCountingExporter(name string) *CountingExporter {
	return &CountingExporter{name: name}
}

func (e *CountingExporter) Name() string { return e.name }

func (e *CountingExporter) Export(ctx context.Context, data *TelemetryData) error {
	e.count.Add(1)
	desc := fmt.Sprintf("type=%s service=%s", data.Type, data.Resource.ServiceName)
	e.mu.Lock()
	e.last = append(e.last, desc)
	e.mu.Unlock()
	return nil
}

func (e *CountingExporter) Stats() (int64, []string) {
	e.mu.Lock()
	defer e.mu.Unlock()
	items := make([]string, len(e.last))
	copy(items, e.last)
	return e.count.Load(), items
}

// --- Route ---

// Route maps a match function to an exporter.
type Route struct {
	Name     string
	Match    func(*TelemetryData) bool
	Exporter Exporter
}

// --- Router ---

// Router evaluates routes and fans out to matching exporters concurrently.
type Router struct {
	routes       []Route
	defaultRoute *Route // for data that matches no route
}

func NewRouter(routes []Route, defaultRoute *Route) *Router {
	return &Router{routes: routes, defaultRoute: defaultRoute}
}

func (r *Router) Route(ctx context.Context, data *TelemetryData) error {
	var matched []Route

	for _, route := range r.routes {
		if route.Match(data) {
			matched = append(matched, route)
		}
	}

	// If nothing matched, use the default route.
	if len(matched) == 0 && r.defaultRoute != nil {
		matched = append(matched, *r.defaultRoute)
	}

	if len(matched) == 0 {
		return nil // no route, drop silently
	}

	// Fan out to all matched routes concurrently.
	var wg sync.WaitGroup
	errs := make(chan error, len(matched))

	for _, route := range matched {
		wg.Add(1)
		go func(rt Route) {
			defer wg.Done()
			if err := rt.Exporter.Export(ctx, data); err != nil {
				errs <- fmt.Errorf("route %s: %w", rt.Name, err)
			}
		}(route)
	}

	wg.Wait()
	close(errs)

	// Collect errors.
	var errMsgs []string
	for err := range errs {
		errMsgs = append(errMsgs, err.Error())
	}
	if len(errMsgs) > 0 {
		return fmt.Errorf("routing errors: %s", strings.Join(errMsgs, "; "))
	}

	return nil
}

func main() {
	// Create exporters.
	elasticsearch := NewCountingExporter("elasticsearch")
	prometheus := NewCountingExporter("prometheus-remote-write")
	jaeger := NewCountingExporter("jaeger")
	pagerduty := NewCountingExporter("pagerduty")
	s3Archive := NewCountingExporter("s3-archive")

	// Build routing rules.
	router := NewRouter(
		[]Route{
			{
				Name: "logs-to-elasticsearch",
				Match: func(data *TelemetryData) bool {
					return data.Type == TelemetryLog
				},
				Exporter: elasticsearch,
			},
			{
				Name: "metrics-to-prometheus",
				Match: func(data *TelemetryData) bool {
					return data.Type == TelemetryMetric
				},
				Exporter: prometheus,
			},
			{
				Name: "traces-to-jaeger",
				Match: func(data *TelemetryData) bool {
					return data.Type == TelemetryTrace
				},
				Exporter: jaeger,
			},
			{
				Name: "errors-to-pagerduty",
				Match: func(data *TelemetryData) bool {
					if data.Type != TelemetryLog {
						return false
					}
					if lp, ok := data.Payload.(LogPayload); ok {
						return lp.Severity == SeverityError
					}
					return false
				},
				Exporter: pagerduty,
			},
			{
				Name: "production-to-archive",
				Match: func(data *TelemetryData) bool {
					return data.Resource.Environment == "production"
				},
				Exporter: s3Archive,
			},
		},
		nil, // no default route
	)

	// Create test data.
	testData := []TelemetryData{
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "order-service", Environment: "production"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityInfo, Body: "Order created"},
		},
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "payment-service", Environment: "production"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityError, Body: "Payment failed"},
		},
		{
			Type:      TelemetryMetric,
			Resource:  Resource{ServiceName: "order-service", Environment: "production"},
			Timestamp: time.Now(),
			Payload:   nil,
		},
		{
			Type:      TelemetryTrace,
			Resource:  Resource{ServiceName: "order-service", Environment: "staging"},
			Timestamp: time.Now(),
			Payload:   nil,
		},
		{
			Type:      TelemetryLog,
			Resource:  Resource{ServiceName: "email-service", Environment: "staging"},
			Timestamp: time.Now(),
			Payload:   LogPayload{Severity: SeverityInfo, Body: "Email sent"},
		},
	}

	ctx := context.Background()
	fmt.Println("=== Fan-Out Routing ===")
	for i, td := range testData {
		if err := router.Route(ctx, &td); err != nil {
			fmt.Printf("[%d] Routing error: %v\n", i, err)
		}
	}

	// Print stats.
	fmt.Println("\n=== Routing Results ===")
	exporters := []*CountingExporter{elasticsearch, prometheus, jaeger, pagerduty, s3Archive}
	for _, exp := range exporters {
		count, items := exp.Stats()
		fmt.Printf("%-25s received %d items\n", exp.Name(), count)
		for _, item := range items {
			fmt.Printf("  -> %s\n", item)
		}
	}
}
```

Output:

```
=== Fan-Out Routing ===

=== Routing Results ===
elasticsearch             received 3 items
  -> type=log service=order-service
  -> type=log service=payment-service
  -> type=log service=email-service
prometheus-remote-write   received 1 items
  -> type=metric service=order-service
jaeger                    received 1 items
  -> type=trace service=order-service
pagerduty                 received 1 items
  -> type=log service=payment-service
s3-archive                received 3 items
  -> type=log service=order-service
  -> type=log service=payment-service
  -> type=metric service=order-service
```

Notice that the error log from `payment-service` in production matched three routes: `logs-to-elasticsearch`, `errors-to-pagerduty`, and `production-to-archive`. This is the power of fan-out routing -- a single piece of data can go to multiple destinations.

---

## 7. Buffering and Backpressure

### The Problem

When a downstream system is slow or temporarily unavailable, you have three choices:

1. **Block** -- Stop accepting new data until the downstream recovers. Simple, but causes backpressure that propagates all the way to the source.
2. **Drop newest** -- Accept data into the buffer until it is full, then drop new data. Good for metrics (you can always get the next scrape).
3. **Drop oldest** -- When the buffer is full, evict the oldest data to make room. Good for logs where recent data is more valuable.

A buffered exporter wraps any exporter with a channel-based buffer and a configurable overflow strategy.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// --- Minimal data model ---

type TelemetryType string

const TelemetryLog TelemetryType = "log"

type TelemetryData struct {
	Type      TelemetryType
	Timestamp time.Time
	Payload   any
	sequence  int // for tracking in demo
}

// --- Exporter interface ---

type Exporter interface {
	Export(ctx context.Context, data *TelemetryData) error
	Name() string
}

// --- Overflow strategy ---

type OverflowStrategy string

const (
	OverflowBlock      OverflowStrategy = "block"
	OverflowDropNewest OverflowStrategy = "drop_newest"
	OverflowDropOldest OverflowStrategy = "drop_oldest"
)

// --- Buffered Exporter ---

type BufferedExporter struct {
	delegate Exporter
	buffer   chan *TelemetryData
	overflow OverflowStrategy
	dropped  atomic.Int64
	exported atomic.Int64
	wg       sync.WaitGroup
	workers  int
}

func NewBufferedExporter(delegate Exporter, bufferSize int, overflow OverflowStrategy, workers int) *BufferedExporter {
	if workers < 1 {
		workers = 1
	}
	return &BufferedExporter{
		delegate: delegate,
		buffer:   make(chan *TelemetryData, bufferSize),
		overflow: overflow,
		workers:  workers,
	}
}

func (b *BufferedExporter) Name() string {
	return fmt.Sprintf("buffered(%s)", b.delegate.Name())
}

// Start launches worker goroutines that drain the buffer.
func (b *BufferedExporter) Start(ctx context.Context) {
	for i := 0; i < b.workers; i++ {
		b.wg.Add(1)
		go func(workerID int) {
			defer b.wg.Done()
			for {
				select {
				case data, ok := <-b.buffer:
					if !ok {
						return // channel closed
					}
					if err := b.delegate.Export(ctx, data); err != nil {
						fmt.Printf("[worker-%d] export error: %v\n", workerID, err)
					} else {
						b.exported.Add(1)
					}
				case <-ctx.Done():
					// Drain remaining items.
					for data := range b.buffer {
						b.delegate.Export(context.Background(), data)
						b.exported.Add(1)
					}
					return
				}
			}
		}(i)
	}
}

// Export adds data to the buffer using the configured overflow strategy.
func (b *BufferedExporter) Export(ctx context.Context, data *TelemetryData) error {
	switch b.overflow {
	case OverflowBlock:
		// Block until space is available or context is cancelled.
		select {
		case b.buffer <- data:
			return nil
		case <-ctx.Done():
			return ctx.Err()
		}

	case OverflowDropNewest:
		// Try to send; if buffer is full, drop the new data.
		select {
		case b.buffer <- data:
			return nil
		default:
			b.dropped.Add(1)
			return nil // silently dropped
		}

	case OverflowDropOldest:
		// Try to send; if buffer is full, remove the oldest item and retry.
		select {
		case b.buffer <- data:
			return nil
		default:
			// Drain one old item.
			select {
			case <-b.buffer:
				b.dropped.Add(1)
			default:
			}
			// Try again.
			select {
			case b.buffer <- data:
			default:
				b.dropped.Add(1) // still full, drop new
			}
			return nil
		}
	}

	return nil
}

// Stop closes the buffer and waits for workers to finish.
func (b *BufferedExporter) Stop() {
	close(b.buffer)
	b.wg.Wait()
}

func (b *BufferedExporter) Stats() (exported, dropped int64) {
	return b.exported.Load(), b.dropped.Load()
}

// --- Slow exporter for testing ---

type SlowExporter struct {
	name     string
	delay    time.Duration
	exported atomic.Int64
}

func (s *SlowExporter) Name() string { return s.name }

func (s *SlowExporter) Export(ctx context.Context, data *TelemetryData) error {
	time.Sleep(s.delay)
	s.exported.Add(1)
	return nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Simulate a slow backend (10ms per export).
	slowBackend := &SlowExporter{name: "slow-elasticsearch", delay: 10 * time.Millisecond}

	// Test each overflow strategy.
	strategies := []OverflowStrategy{OverflowDropNewest, OverflowDropOldest, OverflowBlock}

	for _, strategy := range strategies {
		fmt.Printf("\n=== Strategy: %s ===\n", strategy)

		buffered := NewBufferedExporter(slowBackend, 10, strategy, 2)
		buffered.Start(ctx)

		// Flood the buffer with 100 items.
		start := time.Now()
		for i := 0; i < 100; i++ {
			td := &TelemetryData{
				Type:      TelemetryLog,
				Timestamp: time.Now(),
				sequence:  i,
			}

			// For "block" strategy, use a short timeout to avoid hanging.
			exportCtx := ctx
			if strategy == OverflowBlock {
				var exportCancel context.CancelFunc
				exportCtx, exportCancel = context.WithTimeout(ctx, 50*time.Millisecond)
				defer exportCancel()
			}

			buffered.Export(exportCtx, td)
		}
		elapsed := time.Since(start)

		// Give workers time to process.
		time.Sleep(200 * time.Millisecond)
		buffered.Stop()

		exported, dropped := buffered.Stats()
		fmt.Printf("  Sent 100 items in %v\n", elapsed.Round(time.Millisecond))
		fmt.Printf("  Exported: %d\n", exported)
		fmt.Printf("  Dropped:  %d\n", dropped)
		fmt.Printf("  In-flight (lost): %d\n", 100-exported-dropped)
	}
}
```

Output (approximate, timing-dependent):

```
=== Strategy: drop_newest ===
  Sent 100 items in 0ms
  Exported: 12
  Dropped:  88
  In-flight (lost): 0

=== Strategy: drop_oldest ===
  Sent 100 items in 0ms
  Exported: 12
  Dropped:  88
  In-flight (lost): 0

=== Strategy: block ===
  Sent 100 items in 485ms
  Exported: 100
  Dropped:  0
  In-flight (lost): 0
```

### Understanding the Trade-offs

- **drop_newest**: The sender never blocks. Fast producers do not slow down. But you lose the most recent data when the buffer fills up. Best for metrics where staleness matters less.
- **drop_oldest**: The sender never blocks. You keep the freshest data. Best for logs where recent events are more actionable during an incident.
- **block**: Nothing is dropped, but the sender slows down to the pace of the consumer. Backpressure propagates upstream. Best when data loss is unacceptable and you can tolerate slower ingestion.

---

## 8. Building a Log Aggregator

### Collecting from Multiple Sources

A log aggregator receives logs from many services, deduplicates them, batches them for efficiency, and forwards them to storage. This is what systems like Fluentd, Fluent Bit, and Loki's Promtail do.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"sync"
	"time"
)

// --- Data model ---

type LogEntry struct {
	Service   string
	Timestamp time.Time
	Level     string
	Message   string
	Fields    map[string]string
}

// fingerprint creates a hash for deduplication.
func (le LogEntry) fingerprint() string {
	raw := fmt.Sprintf("%s|%s|%s|%s", le.Service, le.Timestamp.Format(time.RFC3339Nano), le.Level, le.Message)
	h := sha256.Sum256([]byte(raw))
	return hex.EncodeToString(h[:8])
}

// --- Batch ---

type Batch struct {
	Entries   []LogEntry
	CreatedAt time.Time
}

// --- Log Aggregator ---

type LogAggregator struct {
	input       chan LogEntry
	batchSize   int
	flushInterval time.Duration
	output      func(Batch) // callback for completed batches
	seen        map[string]time.Time
	seenMu      sync.Mutex
	dedupWindow time.Duration
}

func NewLogAggregator(batchSize int, flushInterval, dedupWindow time.Duration, output func(Batch)) *LogAggregator {
	return &LogAggregator{
		input:         make(chan LogEntry, 1000),
		batchSize:     batchSize,
		flushInterval: flushInterval,
		output:        output,
		seen:          make(map[string]time.Time),
		dedupWindow:   dedupWindow,
	}
}

// Submit sends a log entry to the aggregator.
func (la *LogAggregator) Submit(entry LogEntry) {
	la.input <- entry
}

// Run starts the aggregation loop.
func (la *LogAggregator) Run(ctx context.Context) {
	var batch []LogEntry
	flushTicker := time.NewTicker(la.flushInterval)
	defer flushTicker.Stop()

	cleanupTicker := time.NewTicker(la.dedupWindow)
	defer cleanupTicker.Stop()

	flush := func() {
		if len(batch) == 0 {
			return
		}
		la.output(Batch{
			Entries:   batch,
			CreatedAt: time.Now(),
		})
		batch = nil
	}

	for {
		select {
		case entry := <-la.input:
			// Deduplication check.
			fp := entry.fingerprint()
			la.seenMu.Lock()
			if lastSeen, exists := la.seen[fp]; exists && time.Since(lastSeen) < la.dedupWindow {
				la.seenMu.Unlock()
				continue // duplicate, skip
			}
			la.seen[fp] = time.Now()
			la.seenMu.Unlock()

			batch = append(batch, entry)
			if len(batch) >= la.batchSize {
				flush()
			}

		case <-flushTicker.C:
			flush()

		case <-cleanupTicker.C:
			// Clean old dedup entries.
			la.seenMu.Lock()
			now := time.Now()
			for fp, ts := range la.seen {
				if now.Sub(ts) > la.dedupWindow {
					delete(la.seen, fp)
				}
			}
			la.seenMu.Unlock()

		case <-ctx.Done():
			flush() // flush remaining
			return
		}
	}
}

func main() {
	var mu sync.Mutex
	var batches []Batch
	totalEntries := 0

	aggregator := NewLogAggregator(
		5,                    // batch size
		100*time.Millisecond, // flush interval
		1*time.Second,        // dedup window
		func(b Batch) {
			mu.Lock()
			batches = append(batches, b)
			totalEntries += len(b.Entries)
			mu.Unlock()
		},
	)

	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	go aggregator.Run(ctx)

	// Simulate logs from multiple services.
	services := []string{"api-gateway", "order-service", "payment-service"}
	now := time.Now()

	submitted := 0
	for i := 0; i < 20; i++ {
		entry := LogEntry{
			Service:   services[i%len(services)],
			Timestamp: now.Add(time.Duration(i) * time.Millisecond),
			Level:     "INFO",
			Message:   fmt.Sprintf("Request processed #%d", i),
			Fields:    map[string]string{"request_id": fmt.Sprintf("req-%d", i)},
		}
		aggregator.Submit(entry)
		submitted++
	}

	// Submit duplicates.
	for i := 0; i < 5; i++ {
		entry := LogEntry{
			Service:   "order-service",
			Timestamp: now,              // same timestamp as earlier
			Level:     "INFO",
			Message:   "Request processed #0", // same message
			Fields:    map[string]string{"request_id": "req-0"},
		}
		aggregator.Submit(entry)
		submitted++
	}

	// Wait for context to expire.
	<-ctx.Done()
	time.Sleep(50 * time.Millisecond) // let final flush complete

	mu.Lock()
	defer mu.Unlock()

	fmt.Println("=== Log Aggregator Results ===")
	fmt.Printf("Submitted:     %d entries (including %d intentional duplicates)\n", submitted, 5)
	fmt.Printf("Batches:       %d\n", len(batches))
	fmt.Printf("Total entries: %d (after deduplication)\n", totalEntries)
	fmt.Println()

	for i, b := range batches {
		fmt.Printf("Batch %d (%d entries):\n", i+1, len(b.Entries))
		for _, e := range b.Entries {
			fmt.Printf("  [%s] %s: %s\n", e.Level, e.Service, e.Message)
		}
	}
}
```

Output:

```
=== Log Aggregator Results ===
Submitted:     25 entries (including 5 intentional duplicates)
Batches:       4
Total entries: 20 (after deduplication)

Batch 1 (5 entries):
  [INFO] api-gateway: Request processed #0
  [INFO] order-service: Request processed #1
  [INFO] payment-service: Request processed #2
  [INFO] api-gateway: Request processed #3
  [INFO] order-service: Request processed #4
Batch 2 (5 entries):
  [INFO] payment-service: Request processed #5
  ...
```

### Key Design Points

- **Batching** reduces the number of network calls to the backend. Instead of sending one log at a time, you send batches of 5 (or 100, or 1000 in production).
- **Flush interval** ensures logs are delivered even when the batch is not full. You do not want to wait forever for 5 more logs when the system is quiet.
- **Deduplication** uses a fingerprint (hash of service + timestamp + message) with a time window. The same log arriving twice within 1 second is dropped.
- **Dedup cleanup** periodically removes old fingerprints to prevent unbounded memory growth.

---

## 9. Trace Correlation

### Connecting Traces, Logs, and Metrics

The real power of observability comes from correlating signals. When you see an error in a trace, you want to find the logs that happened during that trace. When you see a latency spike in a metric, you want to find the traces that caused it. The key is the **trace ID** -- a shared identifier that links all three signals.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"time"
)

// TraceContext carries trace correlation identifiers.
type TraceContext struct {
	TraceID string `json:"trace_id"`
	SpanID  string `json:"span_id"`
}

// NewTraceContext generates a new trace context with random IDs.
func NewTraceContext() TraceContext {
	return TraceContext{
		TraceID: randomHex(16), // 128-bit trace ID
		SpanID:  randomHex(8),  // 64-bit span ID
	}
}

func randomHex(bytes int) string {
	b := make([]byte, bytes)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// NewChildSpan creates a new span ID under the same trace.
func (tc TraceContext) NewChildSpan() TraceContext {
	return TraceContext{
		TraceID: tc.TraceID,
		SpanID:  randomHex(8),
	}
}

// CorrelatedEvent represents any telemetry event with trace correlation.
type CorrelatedEvent struct {
	Type       string       `json:"type"` // "log", "metric", "span"
	TraceCtx   TraceContext `json:"trace_context"`
	Timestamp  time.Time    `json:"timestamp"`
	Service    string       `json:"service"`
	Data       any          `json:"data"`
}

// EventStore collects all correlated events.
type EventStore struct {
	events []CorrelatedEvent
}

func (es *EventStore) Add(event CorrelatedEvent) {
	es.events = append(es.events, event)
}

// FindByTraceID returns all events for a given trace.
func (es *EventStore) FindByTraceID(traceID string) []CorrelatedEvent {
	var result []CorrelatedEvent
	for _, e := range es.events {
		if e.TraceCtx.TraceID == traceID {
			result = append(result, e)
		}
	}
	return result
}

// Correlate groups events by trace ID.
func (es *EventStore) Correlate() map[string][]CorrelatedEvent {
	grouped := make(map[string][]CorrelatedEvent)
	for _, e := range es.events {
		grouped[e.TraceCtx.TraceID] = append(grouped[e.TraceCtx.TraceID], e)
	}
	return grouped
}

// Simulate a request flowing through services.
func simulateRequest(store *EventStore) {
	// API Gateway creates the root trace.
	rootCtx := NewTraceContext()
	now := time.Now()

	// 1. API Gateway receives the request.
	store.Add(CorrelatedEvent{
		Type: "span", TraceCtx: rootCtx, Timestamp: now,
		Service: "api-gateway",
		Data: map[string]any{
			"operation": "POST /api/orders",
			"kind":      "server",
		},
	})
	store.Add(CorrelatedEvent{
		Type: "log", TraceCtx: rootCtx, Timestamp: now,
		Service: "api-gateway",
		Data: map[string]any{
			"level":   "INFO",
			"message": "Received order request",
			"user_id": "user-42",
		},
	})
	store.Add(CorrelatedEvent{
		Type: "metric", TraceCtx: rootCtx, Timestamp: now,
		Service: "api-gateway",
		Data: map[string]any{
			"name":  "http_requests_total",
			"value": 1,
			"labels": map[string]string{
				"method": "POST",
				"path":   "/api/orders",
			},
		},
	})

	// 2. API Gateway calls Order Service.
	orderCtx := rootCtx.NewChildSpan()
	store.Add(CorrelatedEvent{
		Type: "span", TraceCtx: orderCtx, Timestamp: now.Add(2 * time.Millisecond),
		Service: "order-service",
		Data: map[string]any{
			"operation": "CreateOrder",
			"kind":      "server",
			"parent_span_id": rootCtx.SpanID,
		},
	})
	store.Add(CorrelatedEvent{
		Type: "log", TraceCtx: orderCtx, Timestamp: now.Add(3 * time.Millisecond),
		Service: "order-service",
		Data: map[string]any{
			"level":   "INFO",
			"message": "Creating order for user-42",
		},
	})

	// 3. Order Service calls Payment Service.
	paymentCtx := orderCtx.NewChildSpan()
	store.Add(CorrelatedEvent{
		Type: "span", TraceCtx: paymentCtx, Timestamp: now.Add(5 * time.Millisecond),
		Service: "payment-service",
		Data: map[string]any{
			"operation": "ChargeCard",
			"kind":      "server",
			"parent_span_id": orderCtx.SpanID,
		},
	})
	store.Add(CorrelatedEvent{
		Type: "log", TraceCtx: paymentCtx, Timestamp: now.Add(6 * time.Millisecond),
		Service: "payment-service",
		Data: map[string]any{
			"level":   "WARN",
			"message": "Payment gateway slow, retrying",
		},
	})
	store.Add(CorrelatedEvent{
		Type: "metric", TraceCtx: paymentCtx, Timestamp: now.Add(10 * time.Millisecond),
		Service: "payment-service",
		Data: map[string]any{
			"name":  "payment_duration_seconds",
			"value": 0.45,
		},
	})
	store.Add(CorrelatedEvent{
		Type: "log", TraceCtx: paymentCtx, Timestamp: now.Add(11 * time.Millisecond),
		Service: "payment-service",
		Data: map[string]any{
			"level":   "INFO",
			"message": "Payment processed successfully",
		},
	})

	// 4. Order Service completes.
	store.Add(CorrelatedEvent{
		Type: "log", TraceCtx: orderCtx, Timestamp: now.Add(15 * time.Millisecond),
		Service: "order-service",
		Data: map[string]any{
			"level":    "INFO",
			"message":  "Order created successfully",
			"order_id": "ORD-9842",
		},
	})
}

func main() {
	store := &EventStore{}

	// Simulate two requests.
	simulateRequest(store)
	simulateRequest(store)

	// Correlate events by trace ID.
	correlated := store.Correlate()

	fmt.Println("=== Trace Correlation ===")
	fmt.Printf("Total events: %d\n", len(store.events))
	fmt.Printf("Unique traces: %d\n\n", len(correlated))

	traceNum := 1
	for traceID, events := range correlated {
		fmt.Printf("Trace %d: %s (%d events)\n", traceNum, traceID, len(events))

		// Count by type.
		typeCounts := make(map[string]int)
		services := make(map[string]bool)
		for _, e := range events {
			typeCounts[e.Type]++
			services[e.Service] = true
		}
		fmt.Printf("  Types:    spans=%d logs=%d metrics=%d\n",
			typeCounts["span"], typeCounts["log"], typeCounts["metric"])

		svcList := make([]string, 0, len(services))
		for s := range services {
			svcList = append(svcList, s)
		}
		fmt.Printf("  Services: %v\n", svcList)

		fmt.Println("  Timeline:")
		for _, e := range events {
			dataBytes, _ := json.Marshal(e.Data)
			fmt.Printf("    %s [%s] %s: %s\n",
				e.Timestamp.Format("15:04:05.000"),
				e.Type,
				e.Service,
				string(dataBytes),
			)
		}
		fmt.Println()
		traceNum++
	}

	// Demonstrate finding all events for a specific trace.
	// Grab the first trace ID.
	var firstTraceID string
	for id := range correlated {
		firstTraceID = id
		break
	}

	fmt.Printf("=== Lookup: trace_id=%s ===\n", firstTraceID)
	events := store.FindByTraceID(firstTraceID)
	for _, e := range events {
		fmt.Printf("  [%s] %s - %v\n", e.Type, e.Service, e.Data)
	}
}
```

### Why This Matters

In a production incident, you start with a symptom -- a metric showing elevated error rates. You click through to find the traces with errors. Each trace shows you the request path across services. Each span links to the logs that were emitted during that span's execution. Without trace correlation, you are searching through millions of log lines manually. With it, you follow the thread from symptom to root cause in minutes.

---

## 10. OpenTelemetry Collector Architecture

### How the OTel Collector Works

The OpenTelemetry Collector is the reference implementation of an observability pipeline. Understanding its architecture helps you understand any pipeline system. It has three core component types, wired together in a pipeline:

```
┌──────────────────────────────────────────────────────────┐
│                 OpenTelemetry Collector                    │
│                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │ Receiver │   │ Receiver │   │ Receiver │  (ingestion) │
│  │  (OTLP)  │   │(Prometheus│   │ (Jaeger) │             │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘             │
│       │              │              │                    │
│       └──────────────┼──────────────┘                    │
│                      ▼                                   │
│              ┌───────────────┐                            │
│              │   Processor   │                            │
│              │   Pipeline    │  (filter, transform,       │
│              │               │   enrich, batch, sample)   │
│              └───────┬───────┘                            │
│                      │                                   │
│       ┌──────────────┼──────────────┐                    │
│       ▼              ▼              ▼                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │ Exporter │   │ Exporter │   │ Exporter │  (delivery)  │
│  │  (OTLP)  │   │(Prometheus│   │ (Jaeger) │             │
│  └──────────┘   └──────────┘   └──────────┘             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Component Roles

**Receivers** accept telemetry data from external sources. They translate from wire formats (OTLP/gRPC, Prometheus exposition format, Jaeger Thrift) into the internal data model.

**Processors** transform data in flight. The OTel Collector ships with processors for batching, filtering, attribute manipulation, tail-based sampling, and more. They are chained sequentially.

**Exporters** send data to backends. They translate from the internal data model back to wire formats and handle retries, batching, and connection management.

**Pipelines** connect receivers to processors to exporters. You can define multiple pipelines -- one for logs, one for metrics, one for traces -- each with its own set of processors.

```go
package main

import (
	"fmt"
)

// ComponentType identifies the role of a pipeline component.
type ComponentType string

const (
	ReceiverComponent  ComponentType = "receiver"
	ProcessorComponent ComponentType = "processor"
	ExporterComponent  ComponentType = "exporter"
)

// ComponentConfig describes a pipeline component.
type ComponentConfig struct {
	Name   string        `json:"name"`
	Type   ComponentType `json:"type"`
	Config map[string]any `json:"config"`
}

// PipelineConfig describes a full pipeline.
type PipelineConfig struct {
	Name       string            `json:"name"`
	SignalType string            `json:"signal_type"` // logs, metrics, traces
	Receivers  []ComponentConfig `json:"receivers"`
	Processors []ComponentConfig `json:"processors"`
	Exporters  []ComponentConfig `json:"exporters"`
}

func main() {
	// This mirrors an OTel Collector configuration.
	pipelines := []PipelineConfig{
		{
			Name:       "logs",
			SignalType: "logs",
			Receivers: []ComponentConfig{
				{Name: "otlp", Type: ReceiverComponent, Config: map[string]any{
					"protocols": map[string]any{
						"grpc": map[string]any{"endpoint": "0.0.0.0:4317"},
						"http": map[string]any{"endpoint": "0.0.0.0:4318"},
					},
				}},
				{Name: "filelog", Type: ReceiverComponent, Config: map[string]any{
					"include": []string{"/var/log/app/*.log"},
				}},
			},
			Processors: []ComponentConfig{
				{Name: "batch", Type: ProcessorComponent, Config: map[string]any{
					"timeout":    "5s",
					"send_batch_size": 1000,
				}},
				{Name: "filter", Type: ProcessorComponent, Config: map[string]any{
					"logs": map[string]any{
						"exclude": map[string]any{
							"match_type": "strict",
							"bodies":     []string{"health check"},
						},
					},
				}},
			},
			Exporters: []ComponentConfig{
				{Name: "elasticsearch", Type: ExporterComponent, Config: map[string]any{
					"endpoints": []string{"https://es.internal:9200"},
					"index":     "logs-{yyyy.MM.dd}",
				}},
			},
		},
		{
			Name:       "metrics",
			SignalType: "metrics",
			Receivers: []ComponentConfig{
				{Name: "otlp", Type: ReceiverComponent, Config: map[string]any{}},
				{Name: "prometheus", Type: ReceiverComponent, Config: map[string]any{
					"config": map[string]any{
						"scrape_configs": []map[string]any{
							{"job_name": "app", "scrape_interval": "15s"},
						},
					},
				}},
			},
			Processors: []ComponentConfig{
				{Name: "batch", Type: ProcessorComponent, Config: map[string]any{
					"timeout": "10s",
				}},
			},
			Exporters: []ComponentConfig{
				{Name: "prometheusremotewrite", Type: ExporterComponent, Config: map[string]any{
					"endpoint": "https://prometheus.internal/api/v1/write",
				}},
			},
		},
		{
			Name:       "traces",
			SignalType: "traces",
			Receivers: []ComponentConfig{
				{Name: "otlp", Type: ReceiverComponent, Config: map[string]any{}},
				{Name: "jaeger", Type: ReceiverComponent, Config: map[string]any{
					"protocols": map[string]any{
						"thrift_http": map[string]any{"endpoint": "0.0.0.0:14268"},
					},
				}},
			},
			Processors: []ComponentConfig{
				{Name: "batch", Type: ProcessorComponent, Config: map[string]any{}},
				{Name: "tail_sampling", Type: ProcessorComponent, Config: map[string]any{
					"policies": []map[string]any{
						{"name": "errors", "type": "status_code", "status_code": map[string]any{"status_codes": []string{"ERROR"}}},
						{"name": "slow", "type": "latency", "latency": map[string]any{"threshold_ms": 500}},
						{"name": "probabilistic", "type": "probabilistic", "probabilistic": map[string]any{"sampling_percentage": 10}},
					},
				}},
			},
			Exporters: []ComponentConfig{
				{Name: "otlp", Type: ExporterComponent, Config: map[string]any{
					"endpoint": "tempo.internal:4317",
				}},
			},
		},
	}

	fmt.Println("=== OTel Collector Pipeline Configuration ===\n")
	for _, p := range pipelines {
		fmt.Printf("Pipeline: %s (signal: %s)\n", p.Name, p.SignalType)
		fmt.Printf("  Receivers:  ")
		for i, r := range p.Receivers {
			if i > 0 {
				fmt.Print(", ")
			}
			fmt.Print(r.Name)
		}
		fmt.Println()
		fmt.Printf("  Processors: ")
		for i, proc := range p.Processors {
			if i > 0 {
				fmt.Print(" -> ")
			}
			fmt.Print(proc.Name)
		}
		fmt.Println()
		fmt.Printf("  Exporters:  ")
		for i, e := range p.Exporters {
			if i > 0 {
				fmt.Print(", ")
			}
			fmt.Print(e.Name)
		}
		fmt.Println("\n")
	}
}
```

Output:

```
=== OTel Collector Pipeline Configuration ===

Pipeline: logs (signal: logs)
  Receivers:  otlp, filelog
  Processors: batch -> filter
  Exporters:  elasticsearch

Pipeline: metrics (signal: metrics)
  Receivers:  otlp, prometheus
  Processors: batch
  Exporters:  prometheusremotewrite

Pipeline: traces (signal: traces)
  Receivers:  otlp, jaeger
  Processors: batch -> tail_sampling
  Exporters:  otlp
```

---

## 11. Building a Mini OTel Collector

### A Simplified But Functional Collector

Let us build a miniature version of the OTel Collector. It has receivers (data in), processors (data transformation), exporters (data out), and a pipeline that wires them together. This is the centerpiece of the chapter.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// =====================================
// Data Model
// =====================================

type TelemetryType string

const (
	TelemetryLog    TelemetryType = "log"
	TelemetryMetric TelemetryType = "metric"
	TelemetryTrace  TelemetryType = "trace"
)

type Resource struct {
	ServiceName    string            `json:"service_name"`
	ServiceVersion string            `json:"service_version"`
	Environment    string            `json:"environment"`
	HostName       string            `json:"host_name"`
	Attributes     map[string]string `json:"attributes,omitempty"`
}

type TelemetryData struct {
	Type      TelemetryType `json:"type"`
	Resource  Resource      `json:"resource"`
	Timestamp time.Time     `json:"timestamp"`
	Payload   any           `json:"payload"`
	TraceID   string        `json:"trace_id,omitempty"`
	SpanID    string        `json:"span_id,omitempty"`
}

type LogPayload struct {
	Severity string            `json:"severity"`
	Body     string            `json:"body"`
	Attrs    map[string]string `json:"attributes,omitempty"`
}

type MetricPayload struct {
	Name   string            `json:"name"`
	Value  float64           `json:"value"`
	Labels map[string]string `json:"labels,omitempty"`
}

// =====================================
// Interfaces
// =====================================

// Receiver produces telemetry data.
type Receiver interface {
	Receive(ctx context.Context) <-chan TelemetryData
	Name() string
}

// Processor transforms telemetry data.
type Processor interface {
	Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error)
	Name() string
}

// Exporter sends telemetry data to a backend.
type Exporter interface {
	Export(ctx context.Context, data *TelemetryData) error
	Name() string
	Shutdown(ctx context.Context) error
}

// =====================================
// Receivers
// =====================================

// SimulatedLogReceiver generates fake log entries for testing.
type SimulatedLogReceiver struct {
	serviceName string
	interval    time.Duration
}

func NewSimulatedLogReceiver(serviceName string, interval time.Duration) *SimulatedLogReceiver {
	return &SimulatedLogReceiver{serviceName: serviceName, interval: interval}
}

func (r *SimulatedLogReceiver) Name() string {
	return fmt.Sprintf("simulated-log(%s)", r.serviceName)
}

func (r *SimulatedLogReceiver) Receive(ctx context.Context) <-chan TelemetryData {
	ch := make(chan TelemetryData, 50)

	go func() {
		defer close(ch)
		ticker := time.NewTicker(r.interval)
		defer ticker.Stop()

		messages := []struct {
			severity string
			body     string
		}{
			{"INFO", "Request handled successfully"},
			{"INFO", "Cache hit for user session"},
			{"WARN", "Slow database query detected"},
			{"ERROR", "Connection timeout to downstream service"},
			{"INFO", "Background job completed"},
			{"DEBUG", "Entering request handler"},
		}

		seq := 0
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				msg := messages[seq%len(messages)]
				seq++

				td := TelemetryData{
					Type: TelemetryLog,
					Resource: Resource{
						ServiceName: r.serviceName,
						Environment: "production",
						HostName:    fmt.Sprintf("pod-%s-0", r.serviceName),
					},
					Timestamp: time.Now(),
					Payload: LogPayload{
						Severity: msg.severity,
						Body:     msg.body,
						Attrs:    map[string]string{"seq": fmt.Sprintf("%d", seq)},
					},
				}

				select {
				case ch <- td:
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	return ch
}

// SimulatedMetricReceiver generates fake metrics.
type SimulatedMetricReceiver struct {
	serviceName string
	interval    time.Duration
}

func NewSimulatedMetricReceiver(serviceName string, interval time.Duration) *SimulatedMetricReceiver {
	return &SimulatedMetricReceiver{serviceName: serviceName, interval: interval}
}

func (r *SimulatedMetricReceiver) Name() string {
	return fmt.Sprintf("simulated-metric(%s)", r.serviceName)
}

func (r *SimulatedMetricReceiver) Receive(ctx context.Context) <-chan TelemetryData {
	ch := make(chan TelemetryData, 50)

	go func() {
		defer close(ch)
		ticker := time.NewTicker(r.interval)
		defer ticker.Stop()

		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				metrics := []MetricPayload{
					{Name: "http_requests_total", Value: float64(rand.Intn(100)), Labels: map[string]string{"method": "GET", "status": "200"}},
					{Name: "http_request_duration_seconds", Value: rand.Float64() * 2, Labels: map[string]string{"method": "GET"}},
					{Name: "go_goroutines", Value: float64(50 + rand.Intn(50))},
				}

				for _, mp := range metrics {
					td := TelemetryData{
						Type: TelemetryMetric,
						Resource: Resource{
							ServiceName: r.serviceName,
							Environment: "production",
						},
						Timestamp: time.Now(),
						Payload:   mp,
					}
					select {
					case ch <- td:
					case <-ctx.Done():
						return
					}
				}
			}
		}
	}()

	return ch
}

// =====================================
// Processors
// =====================================

// FilterProcessor drops telemetry matching a condition.
type FilterProcessor struct {
	dropMatching func(*TelemetryData) bool
}

func NewDropDebugProcessor() *FilterProcessor {
	return &FilterProcessor{
		dropMatching: func(data *TelemetryData) bool {
			if lp, ok := data.Payload.(LogPayload); ok {
				return lp.Severity == "DEBUG"
			}
			return false
		},
	}
}

func (f *FilterProcessor) Name() string { return "filter" }

func (f *FilterProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if f.dropMatching(data) {
		return nil, nil
	}
	return data, nil
}

// AttributeProcessor adds attributes to the resource.
type AttributeProcessor struct {
	attributes map[string]string
}

func NewAttributeProcessor(attrs map[string]string) *AttributeProcessor {
	return &AttributeProcessor{attributes: attrs}
}

func (a *AttributeProcessor) Name() string { return "attributes" }

func (a *AttributeProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Resource.Attributes == nil {
		data.Resource.Attributes = make(map[string]string)
	}
	for k, v := range a.attributes {
		data.Resource.Attributes[k] = v
	}
	return data, nil
}

// =====================================
// Exporters
// =====================================

// ConsoleExporter prints to stdout.
type ConsoleExporter struct {
	prefix   string
	count    atomic.Int64
	mu       sync.Mutex
	lastN    []string
	maxLastN int
}

func NewConsoleExporter(prefix string) *ConsoleExporter {
	return &ConsoleExporter{prefix: prefix, maxLastN: 20}
}

func (e *ConsoleExporter) Name() string { return fmt.Sprintf("console(%s)", e.prefix) }

func (e *ConsoleExporter) Export(ctx context.Context, data *TelemetryData) error {
	e.count.Add(1)

	var summary string
	switch p := data.Payload.(type) {
	case LogPayload:
		summary = fmt.Sprintf("[%s] %s: %s", p.Severity, data.Resource.ServiceName, p.Body)
	case MetricPayload:
		summary = fmt.Sprintf("%s=%0.2f service=%s", p.Name, p.Value, data.Resource.ServiceName)
	default:
		summary = fmt.Sprintf("type=%s service=%s", data.Type, data.Resource.ServiceName)
	}

	e.mu.Lock()
	if len(e.lastN) < e.maxLastN {
		e.lastN = append(e.lastN, summary)
	}
	e.mu.Unlock()

	return nil
}

func (e *ConsoleExporter) Shutdown(ctx context.Context) error {
	return nil
}

func (e *ConsoleExporter) Report() {
	e.mu.Lock()
	defer e.mu.Unlock()
	fmt.Printf("\n  Exporter %s: %d items\n", e.Name(), e.count.Load())
	for _, s := range e.lastN {
		fmt.Printf("    %s\n", s)
	}
}

// =====================================
// Pipeline
// =====================================

type Pipeline struct {
	processors []Processor
}

func NewPipeline(processors ...Processor) *Pipeline {
	return &Pipeline{processors: processors}
}

func (p *Pipeline) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	for _, proc := range p.processors {
		var err error
		data, err = proc.Process(ctx, data)
		if err != nil {
			return nil, fmt.Errorf("processor %s: %w", proc.Name(), err)
		}
		if data == nil {
			return nil, nil
		}
	}
	return data, nil
}

// =====================================
// Router
// =====================================

type Route struct {
	Name     string
	Match    func(*TelemetryData) bool
	Exporter Exporter
}

type Router struct {
	routes []Route
}

func NewRouter(routes ...Route) *Router {
	return &Router{routes: routes}
}

func (r *Router) Route(ctx context.Context, data *TelemetryData) error {
	var wg sync.WaitGroup
	for _, route := range r.routes {
		if route.Match(data) {
			wg.Add(1)
			go func(rt Route) {
				defer wg.Done()
				rt.Exporter.Export(ctx, data)
			}(route)
		}
	}
	wg.Wait()
	return nil
}

// =====================================
// Collector -- ties it all together
// =====================================

type Collector struct {
	receivers  []Receiver
	pipeline   *Pipeline
	router     *Router
	exporters  []Exporter

	received   atomic.Int64
	processed  atomic.Int64
	dropped    atomic.Int64
	exported   atomic.Int64
}

func NewCollector(receivers []Receiver, pipeline *Pipeline, router *Router, exporters []Exporter) *Collector {
	return &Collector{
		receivers: receivers,
		pipeline:  pipeline,
		router:    router,
		exporters: exporters,
	}
}

func (c *Collector) Start(ctx context.Context) error {
	var wg sync.WaitGroup

	for _, receiver := range c.receivers {
		wg.Add(1)
		go func(r Receiver) {
			defer wg.Done()
			ch := r.Receive(ctx)
			for data := range ch {
				c.received.Add(1)

				// Process through the pipeline.
				processed, err := c.pipeline.Process(ctx, &data)
				if err != nil {
					continue
				}
				if processed == nil {
					c.dropped.Add(1)
					continue
				}
				c.processed.Add(1)

				// Route to exporters.
				if err := c.router.Route(ctx, processed); err != nil {
					continue
				}
				c.exported.Add(1)
			}
		}(receiver)
	}

	// Wait for context cancellation.
	<-ctx.Done()
	wg.Wait()

	return c.Shutdown(context.Background())
}

func (c *Collector) Shutdown(ctx context.Context) error {
	var errs []string
	for _, exp := range c.exporters {
		if err := exp.Shutdown(ctx); err != nil {
			errs = append(errs, fmt.Sprintf("%s: %v", exp.Name(), err))
		}
	}
	if len(errs) > 0 {
		return fmt.Errorf("shutdown errors: %s", strings.Join(errs, "; "))
	}
	return nil
}

func (c *Collector) Stats() (received, processed, dropped, exported int64) {
	return c.received.Load(), c.processed.Load(), c.dropped.Load(), c.exported.Load()
}

func main() {
	// Create receivers.
	receivers := []Receiver{
		NewSimulatedLogReceiver("order-service", 20*time.Millisecond),
		NewSimulatedLogReceiver("payment-service", 30*time.Millisecond),
		NewSimulatedMetricReceiver("order-service", 50*time.Millisecond),
	}

	// Create processors.
	pipeline := NewPipeline(
		NewDropDebugProcessor(),
		NewAttributeProcessor(map[string]string{
			"cluster":     "us-east-1",
			"collector":   "mini-otel",
			"pipeline_id": "main",
		}),
	)

	// Create exporters.
	logExporter := NewConsoleExporter("logs")
	metricExporter := NewConsoleExporter("metrics")
	allExporter := NewConsoleExporter("archive")

	exporters := []Exporter{logExporter, metricExporter, allExporter}

	// Create router.
	router := NewRouter(
		Route{
			Name:     "logs",
			Match:    func(d *TelemetryData) bool { return d.Type == TelemetryLog },
			Exporter: logExporter,
		},
		Route{
			Name:     "metrics",
			Match:    func(d *TelemetryData) bool { return d.Type == TelemetryMetric },
			Exporter: metricExporter,
		},
		Route{
			Name:     "archive-all",
			Match:    func(d *TelemetryData) bool { return true },
			Exporter: allExporter,
		},
	)

	// Build the collector.
	collector := NewCollector(receivers, pipeline, router, exporters)

	// Run for 500ms.
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	fmt.Println("=== Mini OTel Collector ===")
	fmt.Println("Starting collector with:")
	for _, r := range receivers {
		fmt.Printf("  Receiver:  %s\n", r.Name())
	}
	for _, p := range pipeline.processors {
		fmt.Printf("  Processor: %s\n", p.Name())
	}
	for _, e := range exporters {
		fmt.Printf("  Exporter:  %s\n", e.Name())
	}
	fmt.Println("\nRunning for 500ms...\n")

	collector.Start(ctx)

	// Print stats.
	received, processed, dropped, exported := collector.Stats()
	fmt.Println("\n=== Collector Stats ===")
	fmt.Printf("  Received:  %d\n", received)
	fmt.Printf("  Processed: %d (passed pipeline)\n", processed)
	fmt.Printf("  Dropped:   %d (filtered by pipeline)\n", dropped)
	fmt.Printf("  Exported:  %d (routed to exporters)\n", exported)

	// Print exporter reports.
	fmt.Println("\n=== Exporter Reports ===")
	logExporter.Report()
	metricExporter.Report()
	allExporter.Report()

	// Show a sample item with enriched attributes.
	sample := TelemetryData{
		Type: TelemetryLog,
		Resource: Resource{
			ServiceName: "order-service",
			Environment: "production",
		},
		Timestamp: time.Now(),
		Payload: LogPayload{
			Severity: "INFO",
			Body:     "Sample item showing enrichment",
		},
	}
	result, _ := pipeline.Process(context.Background(), &sample)
	if result != nil {
		out, _ := json.MarshalIndent(result, "", "  ")
		fmt.Printf("\n=== Sample Enriched Item ===\n%s\n", string(out))
	}
}
```

Output (approximate):

```
=== Mini OTel Collector ===
Starting collector with:
  Receiver:  simulated-log(order-service)
  Receiver:  simulated-log(payment-service)
  Receiver:  simulated-metric(order-service)
  Processor: filter
  Processor: attributes
  Exporter:  console(logs)
  Exporter:  console(metrics)
  Exporter:  console(archive)

Running for 500ms...

=== Collector Stats ===
  Received:  72
  Processed: 60 (passed pipeline)
  Dropped:   12 (filtered by pipeline)
  Exported:  60 (routed to exporters)

=== Exporter Reports ===
  Exporter console(logs): 30 items
    [INFO] order-service: Request handled successfully
    [INFO] order-service: Cache hit for user session
    [WARN] order-service: Slow database query detected
    [ERROR] order-service: Connection timeout to downstream service
    ...

  Exporter console(metrics): 30 items
    http_requests_total=42.00 service=order-service
    http_request_duration_seconds=1.23 service=order-service
    go_goroutines=67.00 service=order-service
    ...

  Exporter console(archive): 60 items
    ...

=== Sample Enriched Item ===
{
  "type": "log",
  "resource": {
    "service_name": "order-service",
    "environment": "production",
    "attributes": {
      "cluster": "us-east-1",
      "collector": "mini-otel",
      "pipeline_id": "main"
    }
  },
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {
    "severity": "INFO",
    "body": "Sample item showing enrichment",
  }
}
```

---

## 12. Metric Aggregation

### Pre-Aggregating Metrics Before Export

Sending every raw metric data point to a backend is expensive. If 100 pods each emit `http_requests_total` every 15 seconds, that is 400 data points per minute for a single metric. Pre-aggregation combines those into a single data point per label set, dramatically reducing storage costs.

```go
package main

import (
	"fmt"
	"sort"
	"strings"
	"sync"
	"time"
)

// MetricType classifies the metric.
type MetricType string

const (
	MetricCounter   MetricType = "counter"
	MetricGauge     MetricType = "gauge"
	MetricHistogram MetricType = "histogram"
)

// MetricSample is a single data point.
type MetricSample struct {
	Name      string
	Type      MetricType
	Value     float64
	Labels    map[string]string
	Timestamp time.Time
}

// labelKey creates a canonical string key from labels for grouping.
func labelKey(name string, labels map[string]string) string {
	keys := make([]string, 0, len(labels))
	for k := range labels {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	var b strings.Builder
	b.WriteString(name)
	b.WriteString("{")
	for i, k := range keys {
		if i > 0 {
			b.WriteString(",")
		}
		b.WriteString(k)
		b.WriteString("=")
		b.WriteString(labels[k])
	}
	b.WriteString("}")
	return b.String()
}

// AggregatedMetric holds the aggregation result.
type AggregatedMetric struct {
	Name      string
	Type      MetricType
	Labels    map[string]string
	Sum       float64
	Count     int
	Min       float64
	Max       float64
	Last      float64
	Timestamp time.Time
}

// MetricAggregator pre-aggregates metrics by label set.
type MetricAggregator struct {
	mu          sync.Mutex
	aggregations map[string]*AggregatedMetric
	flushInterval time.Duration
}

func NewMetricAggregator(flushInterval time.Duration) *MetricAggregator {
	return &MetricAggregator{
		aggregations:  make(map[string]*AggregatedMetric),
		flushInterval: flushInterval,
	}
}

// Add records a new metric sample.
func (ma *MetricAggregator) Add(sample MetricSample) {
	key := labelKey(sample.Name, sample.Labels)

	ma.mu.Lock()
	defer ma.mu.Unlock()

	agg, exists := ma.aggregations[key]
	if !exists {
		ma.aggregations[key] = &AggregatedMetric{
			Name:      sample.Name,
			Type:      sample.Type,
			Labels:    sample.Labels,
			Sum:       sample.Value,
			Count:     1,
			Min:       sample.Value,
			Max:       sample.Value,
			Last:      sample.Value,
			Timestamp: sample.Timestamp,
		}
		return
	}

	agg.Sum += sample.Value
	agg.Count++
	agg.Last = sample.Value
	agg.Timestamp = sample.Timestamp
	if sample.Value < agg.Min {
		agg.Min = sample.Value
	}
	if sample.Value > agg.Max {
		agg.Max = sample.Value
	}
}

// Flush returns all aggregated metrics and resets the state.
func (ma *MetricAggregator) Flush() []AggregatedMetric {
	ma.mu.Lock()
	defer ma.mu.Unlock()

	result := make([]AggregatedMetric, 0, len(ma.aggregations))
	for _, agg := range ma.aggregations {
		result = append(result, *agg)
	}

	// Reset.
	ma.aggregations = make(map[string]*AggregatedMetric)
	return result
}

// ResolvedValue returns the appropriate value based on metric type.
func (am *AggregatedMetric) ResolvedValue() float64 {
	switch am.Type {
	case MetricCounter:
		return am.Sum // counters are summed
	case MetricGauge:
		return am.Last // gauges use the latest value
	case MetricHistogram:
		if am.Count > 0 {
			return am.Sum / float64(am.Count) // average for histograms
		}
		return 0
	default:
		return am.Last
	}
}

func main() {
	aggregator := NewMetricAggregator(15 * time.Second)

	// Simulate metrics from 5 pods over multiple scrape intervals.
	now := time.Now()
	pods := []string{"pod-1", "pod-2", "pod-3", "pod-4", "pod-5"}

	// Each pod reports the same metrics.
	for scrape := 0; scrape < 4; scrape++ {
		ts := now.Add(time.Duration(scrape) * 15 * time.Second)
		for _, pod := range pods {
			aggregator.Add(MetricSample{
				Name:      "http_requests_total",
				Type:      MetricCounter,
				Value:     float64(100 + scrape*10),
				Labels:    map[string]string{"method": "GET", "status": "200", "pod": pod},
				Timestamp: ts,
			})
			aggregator.Add(MetricSample{
				Name:      "http_request_duration_seconds",
				Type:      MetricHistogram,
				Value:     0.05 + float64(scrape)*0.01,
				Labels:    map[string]string{"method": "GET", "pod": pod},
				Timestamp: ts,
			})
			aggregator.Add(MetricSample{
				Name:      "go_goroutines",
				Type:      MetricGauge,
				Value:     float64(40 + scrape*5),
				Labels:    map[string]string{"pod": pod},
				Timestamp: ts,
			})
		}
	}

	// Flush and display results.
	aggregated := aggregator.Flush()

	fmt.Println("=== Metric Aggregation Results ===")
	fmt.Printf("Input:  %d scrapes x %d pods x 3 metrics = %d raw samples\n", 4, len(pods), 4*len(pods)*3)
	fmt.Printf("Output: %d aggregated metric series\n\n", len(aggregated))

	// Sort for consistent display.
	sort.Slice(aggregated, func(i, j int) bool {
		return aggregated[i].Name < aggregated[j].Name
	})

	for _, agg := range aggregated {
		fmt.Printf("  %-40s type=%-10s samples=%-3d resolved=%.2f  min=%.2f max=%.2f sum=%.2f\n",
			labelKey(agg.Name, agg.Labels),
			agg.Type,
			agg.Count,
			agg.ResolvedValue(),
			agg.Min,
			agg.Max,
			agg.Sum,
		)
	}

	// Show the reduction ratio.
	fmt.Printf("\nReduction ratio: %d raw -> %d aggregated (%.0f%% reduction)\n",
		4*len(pods)*3, len(aggregated),
		(1-float64(len(aggregated))/float64(4*len(pods)*3))*100)
}
```

Output:

```
=== Metric Aggregation Results ===
Input:  4 scrapes x 5 pods x 3 metrics = 60 raw samples
Output: 15 aggregated metric series

  go_goroutines{pod=pod-1}                   type=gauge      samples=4   resolved=55.00  min=40.00 max=55.00 sum=190.00
  go_goroutines{pod=pod-2}                   type=gauge      samples=4   resolved=55.00  min=40.00 max=55.00 sum=190.00
  ...
  http_request_duration_seconds{method=GET,pod=pod-1} type=histogram  samples=4   resolved=0.07  min=0.05 max=0.08 sum=0.26
  ...
  http_requests_total{method=GET,pod=pod-1,status=200} type=counter    samples=4   resolved=460.00  min=100.00 max=130.00 sum=460.00
  ...

Reduction ratio: 60 raw -> 15 aggregated (75% reduction)
```

### Delta vs Cumulative

- **Cumulative** counters always increase. Each scrape returns the total count since the process started. To compute the rate, you subtract consecutive values: `rate = (value_t - value_t-1) / interval`. This is what Prometheus uses.
- **Delta** counters report only the change since the last report. The aggregator must sum the deltas. This is what StatsD and some OTLP implementations use.

Our aggregator uses cumulative semantics for counters (summing all reported values). In production, you would track the previous value per series and compute deltas when needed.

---

## 13. Data Reduction Strategies

### Controlling Costs Without Losing Signal

Observability data is expensive. A medium-sized company with 200 services can easily produce 10TB of telemetry per day. Data reduction strategies cut that volume while preserving the signal you need for debugging.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"strings"
	"time"
)

// --- Minimal data model ---

type TelemetryType string

const (
	TelemetryLog    TelemetryType = "log"
	TelemetryMetric TelemetryType = "metric"
	TelemetryTrace  TelemetryType = "trace"
)

type TelemetryData struct {
	Type      TelemetryType
	Service   string
	Timestamp time.Time
	Severity  string
	Body      string
	TraceID   string
	Labels    map[string]string
	Value     float64
}

// Processor transforms or drops telemetry.
type Processor interface {
	Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error)
	Name() string
}

// ========================================
// Strategy 1: Head-Based Sampling
// ========================================

// HeadSampler drops a percentage of traces at the start,
// using a deterministic hash of the trace ID so all spans
// in a trace are either kept or dropped together.
type HeadSampler struct {
	rate float64 // 0.0 to 1.0
}

func NewHeadSampler(rate float64) *HeadSampler {
	return &HeadSampler{rate: rate}
}

func (s *HeadSampler) Name() string { return fmt.Sprintf("head-sampler(%.0f%%)", s.rate*100) }

func (s *HeadSampler) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Type != TelemetryTrace {
		return data, nil // only sample traces
	}
	if data.TraceID == "" {
		return data, nil
	}

	// Deterministic: hash the trace ID so all spans in a trace get the same decision.
	h := sha256.Sum256([]byte(data.TraceID))
	hashVal := binary.BigEndian.Uint32(h[:4])
	threshold := uint32(s.rate * float64(^uint32(0)))

	if hashVal <= threshold {
		return data, nil // keep
	}
	return nil, nil // drop
}

// ========================================
// Strategy 2: Severity-Based Filtering
// ========================================

// SeverityFilter keeps only logs at or above a minimum severity.
type SeverityFilter struct {
	minLevel int
	levels   map[string]int
}

func NewSeverityFilter(minSeverity string) *SeverityFilter {
	levels := map[string]int{
		"DEBUG": 0,
		"INFO":  1,
		"WARN":  2,
		"ERROR": 3,
		"FATAL": 4,
	}
	return &SeverityFilter{
		minLevel: levels[strings.ToUpper(minSeverity)],
		levels:   levels,
	}
}

func (f *SeverityFilter) Name() string { return "severity-filter" }

func (f *SeverityFilter) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Type != TelemetryLog {
		return data, nil
	}
	level, ok := f.levels[strings.ToUpper(data.Severity)]
	if !ok {
		return data, nil
	}
	if level >= f.minLevel {
		return data, nil
	}
	return nil, nil // drop
}

// ========================================
// Strategy 3: Top-K Metric Filtering
// ========================================

// TopKFilter keeps only the most important metrics.
type TopKFilter struct {
	allowedMetrics map[string]bool
}

func NewTopKFilter(metrics ...string) *TopKFilter {
	allowed := make(map[string]bool)
	for _, m := range metrics {
		allowed[m] = true
	}
	return &TopKFilter{allowedMetrics: allowed}
}

func (f *TopKFilter) Name() string { return "topk-filter" }

func (f *TopKFilter) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Type != TelemetryMetric {
		return data, nil
	}
	metricName := data.Labels["__name__"]
	if metricName == "" {
		metricName = data.Body
	}
	if f.allowedMetrics[metricName] {
		return data, nil
	}
	return nil, nil
}

// ========================================
// Strategy 4: Rate Limiting
// ========================================

// RateLimiter limits the number of items per second per key.
type RateLimiter struct {
	maxPerSecond int
	keyFunc      func(*TelemetryData) string
	counts       map[string]*window
}

type window struct {
	count int
	start time.Time
}

func NewRateLimiter(maxPerSecond int, keyFunc func(*TelemetryData) string) *RateLimiter {
	return &RateLimiter{
		maxPerSecond: maxPerSecond,
		keyFunc:      keyFunc,
		counts:       make(map[string]*window),
	}
}

func (rl *RateLimiter) Name() string { return "rate-limiter" }

func (rl *RateLimiter) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	key := rl.keyFunc(data)
	now := time.Now()

	w, exists := rl.counts[key]
	if !exists || now.Sub(w.start) > time.Second {
		rl.counts[key] = &window{count: 1, start: now}
		return data, nil
	}

	w.count++
	if w.count > rl.maxPerSecond {
		return nil, nil // drop
	}
	return data, nil
}

// ========================================
// Strategy 5: Content-Based Deduplication
// ========================================

// Deduplicator suppresses identical messages within a time window.
type Deduplicator struct {
	windowDuration time.Duration
	seen           map[string]time.Time
}

func NewDeduplicator(window time.Duration) *Deduplicator {
	return &Deduplicator{
		windowDuration: window,
		seen:           make(map[string]time.Time),
	}
}

func (d *Deduplicator) Name() string { return "deduplicator" }

func (d *Deduplicator) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	key := fmt.Sprintf("%s:%s:%s", data.Service, data.Severity, data.Body)
	now := time.Now()

	if lastSeen, exists := d.seen[key]; exists {
		if now.Sub(lastSeen) < d.windowDuration {
			return nil, nil // duplicate
		}
	}
	d.seen[key] = now
	return data, nil
}

func main() {
	ctx := context.Background()

	// Define all reduction strategies.
	strategies := []struct {
		name      string
		processor Processor
		data      []TelemetryData
	}{
		{
			name:      "Head-Based Trace Sampling (10%)",
			processor: NewHeadSampler(0.10),
			data: func() []TelemetryData {
				var data []TelemetryData
				for i := 0; i < 100; i++ {
					data = append(data, TelemetryData{
						Type:    TelemetryTrace,
						Service: "api",
						TraceID: fmt.Sprintf("trace-%04d", i),
						Body:    "GET /api/users",
					})
				}
				return data
			}(),
		},
		{
			name:      "Severity Filtering (WARN+)",
			processor: NewSeverityFilter("WARN"),
			data: []TelemetryData{
				{Type: TelemetryLog, Severity: "DEBUG", Body: "entering handler"},
				{Type: TelemetryLog, Severity: "DEBUG", Body: "parsing request"},
				{Type: TelemetryLog, Severity: "INFO", Body: "request received"},
				{Type: TelemetryLog, Severity: "INFO", Body: "response sent"},
				{Type: TelemetryLog, Severity: "INFO", Body: "cache hit"},
				{Type: TelemetryLog, Severity: "WARN", Body: "slow query"},
				{Type: TelemetryLog, Severity: "ERROR", Body: "connection refused"},
			},
		},
		{
			name:      "Rate Limiting (5/sec per service)",
			processor: NewRateLimiter(5, func(d *TelemetryData) string { return d.Service }),
			data: func() []TelemetryData {
				var data []TelemetryData
				for i := 0; i < 20; i++ {
					data = append(data, TelemetryData{
						Type:    TelemetryLog,
						Service: "chatty-service",
						Body:    fmt.Sprintf("log line %d", i),
					})
				}
				return data
			}(),
		},
		{
			name:      "Deduplication (1s window)",
			processor: NewDeduplicator(1 * time.Second),
			data: []TelemetryData{
				{Type: TelemetryLog, Service: "api", Severity: "ERROR", Body: "connection refused"},
				{Type: TelemetryLog, Service: "api", Severity: "ERROR", Body: "connection refused"},
				{Type: TelemetryLog, Service: "api", Severity: "ERROR", Body: "connection refused"},
				{Type: TelemetryLog, Service: "api", Severity: "ERROR", Body: "connection refused"},
				{Type: TelemetryLog, Service: "api", Severity: "ERROR", Body: "timeout"},
				{Type: TelemetryLog, Service: "api", Severity: "INFO", Body: "recovery complete"},
			},
		},
	}

	fmt.Println("=== Data Reduction Strategies ===\n")

	for _, s := range strategies {
		kept := 0
		for _, d := range s.data {
			d := d
			result, _ := s.processor.Process(ctx, &d)
			if result != nil {
				kept++
			}
		}
		reduction := float64(len(s.data)-kept) / float64(len(s.data)) * 100
		fmt.Printf("%-40s input=%-4d kept=%-4d dropped=%-4d reduction=%.0f%%\n",
			s.name, len(s.data), kept, len(s.data)-kept, reduction)
	}

	// Show combined effect.
	fmt.Println("\n=== Combined Pipeline ===")
	fmt.Println("Applying all strategies in sequence:\n")

	// Build combined test data.
	var allData []TelemetryData
	for i := 0; i < 50; i++ {
		allData = append(allData, TelemetryData{
			Type: TelemetryLog, Service: "api", Severity: "DEBUG",
			Body: fmt.Sprintf("debug %d", i),
		})
	}
	for i := 0; i < 30; i++ {
		allData = append(allData, TelemetryData{
			Type: TelemetryLog, Service: "api", Severity: "INFO",
			Body: fmt.Sprintf("info %d", i),
		})
	}
	for i := 0; i < 10; i++ {
		allData = append(allData, TelemetryData{
			Type: TelemetryLog, Service: "api", Severity: "ERROR",
			Body: "connection refused", // duplicates
		})
	}
	for i := 0; i < 10; i++ {
		allData = append(allData, TelemetryData{
			Type: TelemetryLog, Service: "api", Severity: "ERROR",
			Body: fmt.Sprintf("unique error %d", i),
		})
	}

	processors := []Processor{
		NewSeverityFilter("WARN"),
		NewDeduplicator(1 * time.Second),
		NewRateLimiter(10, func(d *TelemetryData) string { return d.Service }),
	}

	fmt.Printf("Input: %d items\n", len(allData))
	remaining := len(allData)

	for _, proc := range processors {
		kept := 0
		var keptData []TelemetryData
		for _, d := range allData {
			d := d
			result, _ := proc.Process(ctx, &d)
			if result != nil {
				kept++
				keptData = append(keptData, d)
			}
		}
		fmt.Printf("  After %-25s: %d -> %d (dropped %d)\n", proc.Name(), remaining, kept, remaining-kept)
		remaining = kept
		allData = keptData
	}
	fmt.Printf("\nFinal: %d items (from %d original)\n", remaining, 100)
}
```

Output:

```
=== Data Reduction Strategies ===

Head-Based Trace Sampling (10%)            input=100  kept=11   dropped=89   reduction=89%
Severity Filtering (WARN+)                 input=7    kept=2    dropped=5    reduction=71%
Rate Limiting (5/sec per service)          input=20   kept=5    dropped=15   reduction=75%
Deduplication (1s window)                  input=6    kept=3    dropped=3    reduction=50%

=== Combined Pipeline ===
Applying all strategies in sequence:

Input: 100 items
  After severity-filter          : 100 -> 20 (dropped 80)
  After deduplicator             : 20 -> 11 (dropped 9)
  After rate-limiter             : 11 -> 10 (dropped 1)

Final: 10 items (from 100 original)
```

### The Cost Equation

For a company ingesting 10TB/day of logs:
- At $0.50/GB for Elasticsearch, that is **$5,000/day** or **$150,000/month**.
- Dropping DEBUG logs (typically 60% of volume) saves **$90,000/month**.
- Sampling 10% of traces saves **$135,000/month** on trace storage.
- Deduplication during incidents (when the same error repeats thousands of times) can cut incident-time costs by 90%.

The pipeline is the control point for all of this.

---

## 14. Pipeline Monitoring

### Observing the Observer

An observability pipeline that cannot be observed itself is a liability. If the pipeline silently drops data, you will not know until you try to debug an incident and the logs are missing. Pipeline monitoring tracks:

- **Throughput**: Items received, processed, exported per second.
- **Latency**: How long each stage takes.
- **Errors**: Failed exports, parsing errors, connection issues.
- **Drops**: Items dropped by filtering, sampling, or buffer overflow.
- **Dead letter queue**: Items that could not be delivered to any backend.

```go
package main

import (
	"context"
	"fmt"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// --- Minimal data model ---

type TelemetryData struct {
	Type    string
	Service string
	Body    string
}

// ========================================
// Pipeline Metrics
// ========================================

// PipelineMetrics tracks operational metrics for the pipeline itself.
type PipelineMetrics struct {
	received   atomic.Int64
	processed  atomic.Int64
	dropped    atomic.Int64
	exported   atomic.Int64
	errors     atomic.Int64

	stageLatency sync.Map // stage name -> *LatencyTracker
}

// LatencyTracker tracks processing latency for a stage.
type LatencyTracker struct {
	mu    sync.Mutex
	total time.Duration
	count int64
	min   time.Duration
	max   time.Duration
}

func NewLatencyTracker() *LatencyTracker {
	return &LatencyTracker{min: time.Hour} // start with a very high min
}

func (lt *LatencyTracker) Record(d time.Duration) {
	lt.mu.Lock()
	defer lt.mu.Unlock()
	lt.total += d
	lt.count++
	if d < lt.min {
		lt.min = d
	}
	if d > lt.max {
		lt.max = d
	}
}

func (lt *LatencyTracker) Stats() (avg, min, max time.Duration, count int64) {
	lt.mu.Lock()
	defer lt.mu.Unlock()
	if lt.count == 0 {
		return 0, 0, 0, 0
	}
	return lt.total / time.Duration(lt.count), lt.min, lt.max, lt.count
}

func (pm *PipelineMetrics) RecordLatency(stage string, d time.Duration) {
	tracker, _ := pm.stageLatency.LoadOrStore(stage, NewLatencyTracker())
	tracker.(*LatencyTracker).Record(d)
}

// ========================================
// Dead Letter Queue
// ========================================

// DeadLetterQueue stores items that failed to export.
type DeadLetterQueue struct {
	mu    sync.Mutex
	items []DeadLetterItem
	limit int
}

type DeadLetterItem struct {
	Data      *TelemetryData
	Error     error
	Exporter  string
	Timestamp time.Time
	Retries   int
}

func NewDeadLetterQueue(limit int) *DeadLetterQueue {
	return &DeadLetterQueue{limit: limit}
}

func (dlq *DeadLetterQueue) Add(item DeadLetterItem) {
	dlq.mu.Lock()
	defer dlq.mu.Unlock()

	if len(dlq.items) >= dlq.limit {
		// Remove oldest.
		dlq.items = dlq.items[1:]
	}
	dlq.items = append(dlq.items, item)
}

func (dlq *DeadLetterQueue) Len() int {
	dlq.mu.Lock()
	defer dlq.mu.Unlock()
	return len(dlq.items)
}

func (dlq *DeadLetterQueue) Drain() []DeadLetterItem {
	dlq.mu.Lock()
	defer dlq.mu.Unlock()
	items := dlq.items
	dlq.items = nil
	return items
}

// ========================================
// Monitored Pipeline
// ========================================

// Processor is a processing stage.
type Processor interface {
	Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error)
	Name() string
}

// Exporter sends data to a backend.
type Exporter interface {
	Export(ctx context.Context, data *TelemetryData) error
	Name() string
}

// PassthroughProcessor always passes data through (for testing).
type PassthroughProcessor struct{ name string }

func (p *PassthroughProcessor) Name() string { return p.name }
func (p *PassthroughProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	// Simulate some work.
	time.Sleep(time.Microsecond * 10)
	return data, nil
}

// DropProcessor drops data matching a condition.
type DropProcessor struct {
	name string
	drop func(*TelemetryData) bool
}

func (p *DropProcessor) Name() string { return p.name }
func (p *DropProcessor) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if p.drop(data) {
		return nil, nil
	}
	return data, nil
}

// ReliableExporter always succeeds.
type ReliableExporter struct{ name string }

func (e *ReliableExporter) Name() string { return e.name }
func (e *ReliableExporter) Export(ctx context.Context, data *TelemetryData) error {
	time.Sleep(time.Microsecond * 50)
	return nil
}

// UnreliableExporter fails periodically.
type UnreliableExporter struct {
	name     string
	counter  atomic.Int64
	failRate int // fail every N items
}

func (e *UnreliableExporter) Name() string { return e.name }
func (e *UnreliableExporter) Export(ctx context.Context, data *TelemetryData) error {
	n := e.counter.Add(1)
	if int(n)%e.failRate == 0 {
		return fmt.Errorf("connection refused")
	}
	return nil
}

// MonitoredPipeline wraps processors and exporters with metrics collection.
type MonitoredPipeline struct {
	processors []Processor
	exporters  []Exporter
	metrics    *PipelineMetrics
	dlq        *DeadLetterQueue
}

func NewMonitoredPipeline(processors []Processor, exporters []Exporter, dlqLimit int) *MonitoredPipeline {
	return &MonitoredPipeline{
		processors: processors,
		exporters:  exporters,
		metrics:    &PipelineMetrics{},
		dlq:        NewDeadLetterQueue(dlqLimit),
	}
}

func (mp *MonitoredPipeline) Process(ctx context.Context, data *TelemetryData) {
	mp.metrics.received.Add(1)

	// Run through processors.
	current := data
	for _, proc := range mp.processors {
		start := time.Now()
		result, err := proc.Process(ctx, current)
		mp.metrics.RecordLatency("proc:"+proc.Name(), time.Since(start))

		if err != nil {
			mp.metrics.errors.Add(1)
			return
		}
		if result == nil {
			mp.metrics.dropped.Add(1)
			return
		}
		current = result
	}
	mp.metrics.processed.Add(1)

	// Export to all exporters.
	for _, exp := range mp.exporters {
		start := time.Now()
		err := exp.Export(ctx, current)
		mp.metrics.RecordLatency("exp:"+exp.Name(), time.Since(start))

		if err != nil {
			mp.metrics.errors.Add(1)
			mp.dlq.Add(DeadLetterItem{
				Data:      current,
				Error:     err,
				Exporter:  exp.Name(),
				Timestamp: time.Now(),
			})
		} else {
			mp.metrics.exported.Add(1)
		}
	}
}

func (mp *MonitoredPipeline) Report() {
	fmt.Println("=== Pipeline Health Report ===")
	fmt.Printf("  Received:   %d\n", mp.metrics.received.Load())
	fmt.Printf("  Processed:  %d\n", mp.metrics.processed.Load())
	fmt.Printf("  Dropped:    %d\n", mp.metrics.dropped.Load())
	fmt.Printf("  Exported:   %d\n", mp.metrics.exported.Load())
	fmt.Printf("  Errors:     %d\n", mp.metrics.errors.Load())
	fmt.Printf("  DLQ size:   %d\n", mp.dlq.Len())

	fmt.Println("\n  Stage Latencies:")
	mp.metrics.stageLatency.Range(func(key, value any) bool {
		name := key.(string)
		tracker := value.(*LatencyTracker)
		avg, min, max, count := tracker.Stats()
		fmt.Printf("    %-30s avg=%-10s min=%-10s max=%-10s samples=%d\n",
			name, avg.Round(time.Microsecond), min.Round(time.Microsecond),
			max.Round(time.Microsecond), count)
		return true
	})

	// Show DLQ items.
	items := mp.dlq.Drain()
	if len(items) > 0 {
		fmt.Printf("\n  Dead Letter Queue (%d items):\n", len(items))
		limit := 5
		if len(items) < limit {
			limit = len(items)
		}
		for _, item := range items[:limit] {
			fmt.Printf("    exporter=%s error=%q data.body=%q\n",
				item.Exporter, item.Error.Error(), item.Data.Body)
		}
		if len(items) > limit {
			fmt.Printf("    ... and %d more\n", len(items)-limit)
		}
	}
}

func main() {
	pipeline := NewMonitoredPipeline(
		[]Processor{
			&PassthroughProcessor{name: "enrich"},
			&DropProcessor{
				name: "filter-debug",
				drop: func(d *TelemetryData) bool {
					return strings.Contains(d.Body, "DEBUG")
				},
			},
		},
		[]Exporter{
			&ReliableExporter{name: "elasticsearch"},
			&UnreliableExporter{name: "s3-archive", failRate: 7},
		},
		100, // DLQ limit
	)

	ctx := context.Background()

	// Push 100 items through the pipeline.
	for i := 0; i < 100; i++ {
		severity := "INFO"
		if i%10 == 0 {
			severity = "DEBUG"
		}
		if i%15 == 0 {
			severity = "ERROR"
		}

		data := &TelemetryData{
			Type:    "log",
			Service: "order-service",
			Body:    fmt.Sprintf("[%s] Log message #%d", severity, i),
		}
		pipeline.Process(ctx, data)
	}

	pipeline.Report()
}
```

Output:

```
=== Pipeline Health Report ===
  Received:   100
  Processed:  90
  Dropped:    10
  Exported:   167
  Errors:     13
  DLQ size:   13

  Stage Latencies:
    proc:enrich                    avg=12us       min=10us       max=45us       samples=100
    proc:filter-debug              avg=0s         min=0s         max=1us        samples=90
    exp:elasticsearch              avg=52us       min=50us       max=120us      samples=90
    exp:s3-archive                 avg=0s         min=0s         max=1us        samples=90

  Dead Letter Queue (13 items):
    exporter=s3-archive error="connection refused" data.body="[INFO] Log message #6"
    exporter=s3-archive error="connection refused" data.body="[INFO] Log message #13"
    exporter=s3-archive error="connection refused" data.body="[ERROR] Log message #20"
    exporter=s3-archive error="connection refused" data.body="[INFO] Log message #27"
    exporter=s3-archive error="connection refused" data.body="[INFO] Log message #34"
    ... and 8 more
```

### Why Dead Letter Queues Matter

Without a DLQ, failed exports silently disappear. During an incident, those might be the exact error logs you need. With a DLQ, failed items are preserved and can be retried later, inspected for debugging, or forwarded to a different backend. Production pipelines typically persist the DLQ to disk so items survive pipeline restarts.

---

## 15. Real-World Example: Complete Observability Pipeline

### Putting It All Together

This final example builds a complete observability pipeline that:
1. Receives logs, metrics, and traces from simulated services.
2. Filters out debug logs and health checks.
3. Enriches data with team ownership metadata.
4. Samples traces at 20%.
5. Routes logs to one exporter, metrics to another, and errors to a third.
6. Buffers exports to handle slow backends.
7. Monitors its own health.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"math/rand"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

// =====================================================================
// TELEMETRY DATA MODEL
// =====================================================================

type TelemetryType string

const (
	TypeLog    TelemetryType = "log"
	TypeMetric TelemetryType = "metric"
	TypeTrace  TelemetryType = "trace"
)

type Resource struct {
	ServiceName string            `json:"service_name"`
	Environment string            `json:"environment"`
	Attributes  map[string]string `json:"attributes,omitempty"`
}

type TelemetryData struct {
	Type      TelemetryType `json:"type"`
	Resource  Resource      `json:"resource"`
	Timestamp time.Time     `json:"timestamp"`
	TraceID   string        `json:"trace_id,omitempty"`
	// Log fields
	Severity string `json:"severity,omitempty"`
	Body     string `json:"body,omitempty"`
	// Metric fields
	MetricName  string            `json:"metric_name,omitempty"`
	MetricValue float64           `json:"metric_value,omitempty"`
	Labels      map[string]string `json:"labels,omitempty"`
	// Trace fields
	SpanName string        `json:"span_name,omitempty"`
	Duration time.Duration `json:"duration,omitempty"`
}

// =====================================================================
// INTERFACES
// =====================================================================

type Receiver interface {
	Receive(ctx context.Context) <-chan TelemetryData
	Name() string
}

type Processor interface {
	Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error)
	Name() string
}

type Exporter interface {
	Export(ctx context.Context, batch []*TelemetryData) error
	Name() string
}

// =====================================================================
// RECEIVERS
// =====================================================================

type ServiceSimulator struct {
	serviceName string
	interval    time.Duration
	rng         *rand.Rand
}

func NewServiceSimulator(name string, interval time.Duration) *ServiceSimulator {
	return &ServiceSimulator{
		serviceName: name,
		interval:    interval,
		rng:         rand.New(rand.NewSource(time.Now().UnixNano())),
	}
}

func (s *ServiceSimulator) Name() string {
	return fmt.Sprintf("simulator(%s)", s.serviceName)
}

func (s *ServiceSimulator) Receive(ctx context.Context) <-chan TelemetryData {
	ch := make(chan TelemetryData, 100)

	go func() {
		defer close(ch)
		ticker := time.NewTicker(s.interval)
		defer ticker.Stop()

		seq := 0
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				seq++

				// Emit a log.
				severity, body := s.randomLog(seq)
				traceID := fmt.Sprintf("trace-%s-%04d", s.serviceName, seq)

				select {
				case ch <- TelemetryData{
					Type:      TypeLog,
					Resource:  Resource{ServiceName: s.serviceName, Environment: "production"},
					Timestamp: time.Now(),
					Severity:  severity,
					Body:      body,
					TraceID:   traceID,
				}:
				case <-ctx.Done():
					return
				}

				// Emit metrics.
				select {
				case ch <- TelemetryData{
					Type:        TypeMetric,
					Resource:    Resource{ServiceName: s.serviceName, Environment: "production"},
					Timestamp:   time.Now(),
					MetricName:  "http_requests_total",
					MetricValue: float64(s.rng.Intn(50) + 1),
					Labels:      map[string]string{"method": "GET", "status": "200"},
				}:
				case <-ctx.Done():
					return
				}

				// Emit a trace span.
				select {
				case ch <- TelemetryData{
					Type:      TypeTrace,
					Resource:  Resource{ServiceName: s.serviceName, Environment: "production"},
					Timestamp: time.Now(),
					TraceID:   traceID,
					SpanName:  "handle_request",
					Duration:  time.Duration(s.rng.Intn(500)) * time.Millisecond,
				}:
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	return ch
}

func (s *ServiceSimulator) randomLog(seq int) (string, string) {
	roll := s.rng.Intn(100)
	switch {
	case roll < 5:
		return "ERROR", fmt.Sprintf("Connection timeout to database (seq=%d)", seq)
	case roll < 10:
		return "WARN", fmt.Sprintf("Slow query detected: 1200ms (seq=%d)", seq)
	case roll < 20:
		return "DEBUG", fmt.Sprintf("Entering handler (seq=%d)", seq)
	case roll < 25:
		return "INFO", "Health check OK"
	default:
		return "INFO", fmt.Sprintf("Request processed successfully (seq=%d)", seq)
	}
}

// =====================================================================
// PROCESSORS
// =====================================================================

// DebugFilter drops DEBUG logs and health checks.
type DebugFilter struct{}

func (f *DebugFilter) Name() string { return "debug-filter" }
func (f *DebugFilter) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Type == TypeLog {
		if data.Severity == "DEBUG" {
			return nil, nil
		}
		if strings.Contains(strings.ToLower(data.Body), "health check") {
			return nil, nil
		}
	}
	return data, nil
}

// TeamEnricher adds team ownership metadata.
type TeamEnricher struct {
	teams map[string]map[string]string
}

func NewTeamEnricher() *TeamEnricher {
	return &TeamEnricher{
		teams: map[string]map[string]string{
			"order-service":   {"team": "commerce", "tier": "1", "oncall": "alice@example.com"},
			"payment-service": {"team": "payments", "tier": "1", "oncall": "bob@example.com"},
			"search-service":  {"team": "discovery", "tier": "2", "oncall": "carol@example.com"},
			"email-service":   {"team": "comms", "tier": "3", "oncall": "dave@example.com"},
		},
	}
}

func (e *TeamEnricher) Name() string { return "team-enricher" }
func (e *TeamEnricher) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	info, ok := e.teams[data.Resource.ServiceName]
	if !ok {
		return data, nil
	}
	if data.Resource.Attributes == nil {
		data.Resource.Attributes = make(map[string]string)
	}
	for k, v := range info {
		data.Resource.Attributes[k] = v
	}
	return data, nil
}

// TraceSampler samples traces at a given rate using deterministic hashing.
type TraceSampler struct {
	rate float64
}

func NewTraceSampler(rate float64) *TraceSampler {
	return &TraceSampler{rate: rate}
}

func (s *TraceSampler) Name() string { return fmt.Sprintf("trace-sampler(%.0f%%)", s.rate*100) }
func (s *TraceSampler) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	if data.Type != TypeTrace {
		return data, nil
	}
	if data.TraceID == "" {
		return data, nil
	}
	h := sha256.Sum256([]byte(data.TraceID))
	hashVal := binary.BigEndian.Uint32(h[:4])
	threshold := uint32(s.rate * float64(^uint32(0)))
	if hashVal <= threshold {
		return data, nil
	}
	return nil, nil
}

// =====================================================================
// PIPELINE
// =====================================================================

type Pipeline struct {
	processors []Processor
}

func NewPipeline(procs ...Processor) *Pipeline {
	return &Pipeline{processors: procs}
}

func (p *Pipeline) Process(ctx context.Context, data *TelemetryData) (*TelemetryData, error) {
	for _, proc := range p.processors {
		var err error
		data, err = proc.Process(ctx, data)
		if err != nil {
			return nil, err
		}
		if data == nil {
			return nil, nil
		}
	}
	return data, nil
}

// =====================================================================
// EXPORTERS (with batching)
// =====================================================================

type BatchExporter struct {
	name      string
	buffer    chan *TelemetryData
	batchSize int
	flush     time.Duration
	count     atomic.Int64
	mu        sync.Mutex
	batches   int
	wg        sync.WaitGroup
}

func NewBatchExporter(name string, batchSize int, flush time.Duration) *BatchExporter {
	return &BatchExporter{
		name:      name,
		buffer:    make(chan *TelemetryData, batchSize*10),
		batchSize: batchSize,
		flush:     flush,
	}
}

func (e *BatchExporter) Name() string { return e.name }

func (e *BatchExporter) Export(ctx context.Context, batch []*TelemetryData) error {
	for _, item := range batch {
		select {
		case e.buffer <- item:
			e.count.Add(1)
		case <-ctx.Done():
			return ctx.Err()
		}
	}
	return nil
}

// ExportSingle exports a single item (for the pipeline to call).
func (e *BatchExporter) ExportSingle(ctx context.Context, data *TelemetryData) error {
	select {
	case e.buffer <- data:
		e.count.Add(1)
		return nil
	case <-ctx.Done():
		return ctx.Err()
	default:
		return nil // drop if buffer full
	}
}

func (e *BatchExporter) Start(ctx context.Context) {
	e.wg.Add(1)
	go func() {
		defer e.wg.Done()
		ticker := time.NewTicker(e.flush)
		defer ticker.Stop()

		var batch []*TelemetryData

		for {
			select {
			case item, ok := <-e.buffer:
				if !ok {
					if len(batch) > 0 {
						e.mu.Lock()
						e.batches++
						e.mu.Unlock()
					}
					return
				}
				batch = append(batch, item)
				if len(batch) >= e.batchSize {
					e.mu.Lock()
					e.batches++
					e.mu.Unlock()
					batch = nil
				}

			case <-ticker.C:
				if len(batch) > 0 {
					e.mu.Lock()
					e.batches++
					e.mu.Unlock()
					batch = nil
				}

			case <-ctx.Done():
				for {
					select {
					case _, ok := <-e.buffer:
						if !ok {
							return
						}
					default:
						if len(batch) > 0 {
							e.mu.Lock()
							e.batches++
							e.mu.Unlock()
						}
						return
					}
				}
			}
		}
	}()
}

func (e *BatchExporter) Stop() {
	close(e.buffer)
	e.wg.Wait()
}

func (e *BatchExporter) Stats() (count int64, batches int) {
	e.mu.Lock()
	defer e.mu.Unlock()
	return e.count.Load(), e.batches
}

// =====================================================================
// ROUTER
// =====================================================================

type Route struct {
	Name     string
	Match    func(*TelemetryData) bool
	Exporter *BatchExporter
}

type Router struct {
	routes []Route
}

func NewRouter(routes ...Route) *Router {
	return &Router{routes: routes}
}

func (r *Router) Route(ctx context.Context, data *TelemetryData) {
	for _, route := range r.routes {
		if route.Match(data) {
			route.Exporter.ExportSingle(ctx, data)
		}
	}
}

// =====================================================================
// PIPELINE STATS
// =====================================================================

type PipelineStats struct {
	received  atomic.Int64
	processed atomic.Int64
	dropped   atomic.Int64
	routed    atomic.Int64
}

// =====================================================================
// COMPLETE COLLECTOR
// =====================================================================

type Collector struct {
	receivers  []Receiver
	pipeline   *Pipeline
	router     *Router
	exporters  []*BatchExporter
	stats      PipelineStats
}

func NewCollector(
	receivers []Receiver,
	pipeline *Pipeline,
	router *Router,
	exporters []*BatchExporter,
) *Collector {
	return &Collector{
		receivers: receivers,
		pipeline:  pipeline,
		router:    router,
		exporters: exporters,
	}
}

func (c *Collector) Run(ctx context.Context) {
	// Start exporters.
	for _, exp := range c.exporters {
		exp.Start(ctx)
	}

	// Start receivers.
	var wg sync.WaitGroup
	for _, recv := range c.receivers {
		wg.Add(1)
		go func(r Receiver) {
			defer wg.Done()
			ch := r.Receive(ctx)
			for data := range ch {
				c.stats.received.Add(1)

				result, err := c.pipeline.Process(ctx, &data)
				if err != nil {
					continue
				}
				if result == nil {
					c.stats.dropped.Add(1)
					continue
				}
				c.stats.processed.Add(1)

				c.router.Route(ctx, result)
				c.stats.routed.Add(1)
			}
		}(recv)
	}

	<-ctx.Done()
	wg.Wait()

	// Stop exporters.
	for _, exp := range c.exporters {
		exp.Stop()
	}
}

func main() {
	fmt.Println("=====================================================")
	fmt.Println("  COMPLETE OBSERVABILITY PIPELINE")
	fmt.Println("=====================================================")
	fmt.Println()

	// --- Receivers ---
	receivers := []Receiver{
		NewServiceSimulator("order-service", 15*time.Millisecond),
		NewServiceSimulator("payment-service", 20*time.Millisecond),
		NewServiceSimulator("search-service", 25*time.Millisecond),
		NewServiceSimulator("email-service", 30*time.Millisecond),
	}

	// --- Processors ---
	pipeline := NewPipeline(
		&DebugFilter{},
		NewTeamEnricher(),
		NewTraceSampler(0.20), // keep 20% of traces
	)

	// --- Exporters ---
	logExporter := NewBatchExporter("log-store", 50, 100*time.Millisecond)
	metricExporter := NewBatchExporter("metric-store", 100, 200*time.Millisecond)
	traceExporter := NewBatchExporter("trace-store", 20, 100*time.Millisecond)
	errorExporter := NewBatchExporter("error-alerter", 10, 50*time.Millisecond)
	archiveExporter := NewBatchExporter("cold-archive", 200, 500*time.Millisecond)

	allExporters := []*BatchExporter{
		logExporter, metricExporter, traceExporter, errorExporter, archiveExporter,
	}

	// --- Router ---
	router := NewRouter(
		Route{
			Name:     "logs",
			Match:    func(d *TelemetryData) bool { return d.Type == TypeLog },
			Exporter: logExporter,
		},
		Route{
			Name:     "metrics",
			Match:    func(d *TelemetryData) bool { return d.Type == TypeMetric },
			Exporter: metricExporter,
		},
		Route{
			Name:     "traces",
			Match:    func(d *TelemetryData) bool { return d.Type == TypeTrace },
			Exporter: traceExporter,
		},
		Route{
			Name: "errors",
			Match: func(d *TelemetryData) bool {
				return d.Type == TypeLog && d.Severity == "ERROR"
			},
			Exporter: errorExporter,
		},
		Route{
			Name:     "archive-everything",
			Match:    func(d *TelemetryData) bool { return true },
			Exporter: archiveExporter,
		},
	)

	// --- Build and run ---
	collector := NewCollector(receivers, pipeline, router, allExporters)

	fmt.Println("Configuration:")
	fmt.Printf("  Receivers:  %d services\n", len(receivers))
	for _, r := range receivers {
		fmt.Printf("    - %s\n", r.Name())
	}
	fmt.Printf("  Processors: %d stages\n", len(pipeline.processors))
	for _, p := range pipeline.processors {
		fmt.Printf("    - %s\n", p.Name())
	}
	fmt.Printf("  Exporters:  %d destinations\n", len(allExporters))
	for _, e := range allExporters {
		fmt.Printf("    - %s\n", e.Name())
	}
	fmt.Println()

	duration := 1 * time.Second
	fmt.Printf("Running pipeline for %v...\n\n", duration)

	ctx, cancel := context.WithTimeout(context.Background(), duration)
	defer cancel()

	start := time.Now()
	collector.Run(ctx)
	elapsed := time.Since(start)

	// --- Report ---
	fmt.Println("=====================================================")
	fmt.Println("  PIPELINE RESULTS")
	fmt.Println("=====================================================")
	fmt.Printf("\n  Duration:    %v\n", elapsed.Round(time.Millisecond))
	fmt.Printf("  Received:    %d items\n", collector.stats.received.Load())
	fmt.Printf("  Processed:   %d items (passed all processors)\n", collector.stats.processed.Load())
	fmt.Printf("  Dropped:     %d items (filtered/sampled)\n", collector.stats.dropped.Load())
	fmt.Printf("  Routed:      %d items (sent to exporters)\n", collector.stats.routed.Load())

	// Throughput.
	rps := float64(collector.stats.received.Load()) / elapsed.Seconds()
	fmt.Printf("  Throughput:  %.0f items/sec\n", rps)

	fmt.Println("\n  Exporter Breakdown:")
	for _, exp := range allExporters {
		count, batches := exp.Stats()
		fmt.Printf("    %-20s items=%-6d batches=%d\n", exp.Name(), count, batches)
	}

	// Data reduction summary.
	received := collector.stats.received.Load()
	dropped := collector.stats.dropped.Load()
	if received > 0 {
		reductionPct := float64(dropped) / float64(received) * 100
		fmt.Printf("\n  Data Reduction: %.1f%% (%d/%d items dropped before export)\n",
			reductionPct, dropped, received)
	}

	fmt.Println("\n=====================================================")
	fmt.Println("  Pipeline shutdown complete.")
	fmt.Println("=====================================================")
}
```

Output (approximate, numbers will vary due to timing and randomness):

```
=====================================================
  COMPLETE OBSERVABILITY PIPELINE
=====================================================

Configuration:
  Receivers:  4 services
    - simulator(order-service)
    - simulator(payment-service)
    - simulator(search-service)
    - simulator(email-service)
  Processors: 3 stages
    - debug-filter
    - team-enricher
    - trace-sampler(20%)
  Exporters:  5 destinations
    - log-store
    - metric-store
    - trace-store
    - error-alerter
    - cold-archive

Running pipeline for 1s...

=====================================================
  PIPELINE RESULTS
=====================================================

  Duration:    1.003s
  Received:    462 items
  Processed:   318 items (passed all processors)
  Dropped:     144 items (filtered/sampled)
  Routed:      318 items (sent to exporters)
  Throughput:  461 items/sec

  Exporter Breakdown:
    log-store            items=130    batches=3
    metric-store         items=154    batches=2
    trace-store          items=34     batches=2
    error-alerter        items=8      batches=1
    cold-archive         items=318    batches=2

  Data Reduction: 31.2% (144/462 items dropped before export)

=====================================================
  Pipeline shutdown complete.
=====================================================
```

### What We Built

This pipeline demonstrates every concept from the chapter working together:

1. **Four simulated services** produce logs, metrics, and traces concurrently (Section 3-4).
2. **Three processors** in sequence filter debug logs, enrich with team metadata, and sample traces (Section 5).
3. **Five routes** fan out to different exporters based on data type and severity (Section 6).
4. **Batch exporters** buffer data before "sending" to backends (Section 7-8).
5. **Pipeline stats** track received, processed, dropped, and routed counts (Section 14).
6. **Data reduction** through filtering and sampling cut volume by ~30% before export (Section 13).

Every component communicates via Go channels. Every goroutine respects `context.Context` for cancellation. The pipeline shuts down gracefully, draining all buffers before exiting.

---

## 16. Key Takeaways

1. **Pipelines are infrastructure, not just code.** An observability pipeline is as critical as your database. If the pipeline drops data during an incident, you lose your ability to debug.

2. **The Receiver-Processor-Exporter pattern is universal.** Whether you use the OTel Collector, Vector, Fluentd, or build your own, the architecture is the same: collect, process, route, export.

3. **Go channels are natural pipeline connectors.** Each stage reads from a channel and writes to the next. Buffered channels provide backpressure. `select` handles timeouts and cancellation. This is exactly what Go's concurrency model was designed for.

4. **Filter early, enrich late.** Drop unwanted data as early as possible to avoid wasting CPU on enrichment and serialization for data you will throw away.

5. **Never sample errors.** Your sampling processor should always have a carve-out for error-level logs and error-status traces. These are the exact signals you need during incidents.

6. **Buffering strategy depends on the data type.** Metrics can tolerate `drop_oldest` because you will get fresh data on the next scrape. Logs during incidents should never be dropped -- use `block` or `drop_oldest` (keeping recent logs). Traces benefit from head-based sampling (decided at the start) so you keep complete traces rather than random fragments.

7. **Batching is essential for throughput.** Sending one item at a time to a backend is extremely inefficient. Batch 100-1000 items per HTTP request to amortize connection overhead.

8. **Pre-aggregate metrics to control costs.** If 100 pods report the same metric, aggregate before export. You reduce cardinality (and storage cost) by orders of magnitude.

9. **Deduplication saves money during incidents.** When a service is in a crash loop, it may emit the same error log thousands of times per second. Deduplication (by content hash within a time window) prevents this from overwhelming your storage.

10. **Observe the observer.** Your pipeline must emit its own metrics: items received, items dropped, export latency, error rates, DLQ size. If you cannot monitor your monitoring infrastructure, you are flying blind.

11. **Dead letter queues prevent data loss.** When an exporter fails, failed items go to a DLQ rather than being silently dropped. The DLQ can be retried, inspected, or forwarded to a fallback backend.

12. **Trace correlation connects the dots.** A trace ID that flows through logs, metrics, and spans lets you go from "error rate is elevated" to "here is the exact request that failed, here are all its logs, and here is the database query that timed out" in minutes.

13. **Context propagation is non-negotiable.** Every function in the pipeline takes `context.Context`. This enables graceful shutdown (cancellation propagates through every goroutine), timeout enforcement, and request-scoped values.

14. **Data reduction is a business decision.** Choosing what to keep and what to drop is not a technical decision -- it is a cost-vs-observability trade-off. The pipeline gives you the knobs to make that trade-off explicitly rather than losing data accidentally.

15. **Test your pipeline under load.** A pipeline that works at 100 items/sec may fall over at 10,000 items/sec. Benchmark your processors, test with realistic volume, and understand where the bottlenecks are before production traffic hits.

---

## 17. Practice Exercises

### Exercise 1: Disk-Backed Buffer

Build a `DiskBackedBuffer` exporter that:
- Buffers telemetry data to a local file when the in-memory buffer is full.
- Reads from the disk buffer when the in-memory buffer has capacity.
- Survives process restarts -- on startup, it reads the disk buffer and resumes.
- Implements a maximum disk usage limit (e.g., 100MB).
- Write a test that fills the buffer beyond in-memory capacity, restarts the exporter, and verifies all data is eventually delivered.

### Exercise 2: Tail-Based Sampling

Build a `TailBasedSampler` processor that:
- Collects all spans for a trace before making a sampling decision.
- Keeps all traces that contain at least one error span.
- Keeps all traces with total duration exceeding a threshold (e.g., 500ms).
- Samples the remaining traces at a configurable rate (e.g., 10%).
- Has a configurable wait window (e.g., 30 seconds) to collect all spans before deciding.
- Write tests with complete trace data showing that error traces and slow traces are always kept.

### Exercise 3: Prometheus Remote Write Exporter

Build an exporter that:
- Accepts `MetricPayload` items.
- Batches them into Prometheus Remote Write format (protocol buffer encoding).
- Sends them via HTTP POST to a configurable endpoint.
- Implements retry with exponential backoff on failure.
- Tracks export latency, success rate, and retry counts.
- Write a test using `httptest.Server` to simulate the remote write endpoint and verify correct formatting.

### Exercise 4: Multi-Tenant Pipeline

Extend the pipeline to support multi-tenancy:
- Each telemetry item has a `tenant_id` field.
- The router routes items to tenant-specific exporters.
- Each tenant has configurable rate limits and sampling rates.
- A "noisy neighbor" protection mechanism drops data from tenants exceeding their quota.
- Write a test that simulates 5 tenants with different quotas and verifies that no tenant can consume more than its allocated throughput.

### Exercise 5: Pipeline Configuration from YAML

Build a pipeline configuration system that:
- Reads pipeline configuration from a YAML file (receivers, processors, exporters, routes).
- Dynamically constructs the pipeline from the configuration.
- Supports hot-reloading: watching the config file for changes and reconfiguring the pipeline without downtime.
- Validates the configuration (no dangling references, valid processor chains).
- Write tests that load different configurations and verify the pipeline behaves accordingly.

### Exercise 6: Log-to-Metric Converter

Build a processor that converts log patterns into metrics:
- Counts occurrences of log patterns (e.g., "connection refused" -> `connection_errors_total`).
- Extracts numeric values from log bodies (e.g., "query took 1234ms" -> `query_duration_milliseconds{} 1234`).
- Supports configurable regex patterns for extraction.
- Emits the generated metrics alongside the original logs (one input produces two outputs).
- Write tests with sample log lines and verify correct metric extraction.

### Exercise 7: Distributed Pipeline

Build a two-tier pipeline:
- **Agent tier**: Runs on each host, tails local log files, collects local metrics, and forwards to the gateway.
- **Gateway tier**: Receives data from multiple agents, aggregates, deduplicates, and exports to backends.
- Agents communicate with the gateway over HTTP or gRPC.
- The gateway handles agent registration and health checking.
- If the gateway is unavailable, agents buffer locally and retry.
- Write an integration test that starts 3 agents and 1 gateway, sends data, and verifies it arrives at the final exporter.

### Exercise 8: Anomaly Detection Processor

Build a processor that detects anomalies in metric streams:
- Maintains a sliding window of metric values per series (e.g., last 100 data points).
- Computes a rolling mean and standard deviation.
- Flags values that are more than 3 standard deviations from the mean.
- Adds an `anomaly=true` attribute to flagged telemetry.
- Optionally emits an additional alert-type event for anomalies.
- Write tests with known data sequences that include clear anomalies and verify detection.

### Exercise 9: Pipeline Benchmarking

Write comprehensive benchmarks for the pipeline:
- Benchmark each processor individually (filter, transform, enrich, sample) with `testing.B`.
- Benchmark the full pipeline with 1, 5, and 10 processors chained.
- Benchmark the router with 1, 5, and 20 routes.
- Benchmark the buffered exporter under different overflow strategies.
- Compare throughput with 1 worker vs 4 workers vs 16 workers.
- Report allocations per operation. Identify which processors allocate and which are zero-alloc.

### Exercise 10: End-to-End Observability

Build a complete system:
- A simple HTTP API (3 endpoints: create order, get order, list orders).
- The API emits structured logs (via `slog`), Prometheus metrics (via a `/metrics` endpoint), and traces (with trace IDs in log fields).
- A pipeline collects all three signal types from the API.
- The pipeline enriches, filters, and routes the data.
- Build a simple query tool that, given a trace ID, retrieves all correlated logs, metrics, and spans from the pipeline's exporters.
- Write an integration test that sends HTTP requests to the API, queries the pipeline for the trace ID, and verifies all three signal types are present and correlated.
