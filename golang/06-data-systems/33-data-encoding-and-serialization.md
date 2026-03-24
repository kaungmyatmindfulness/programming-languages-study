# Chapter 33: Data Encoding & Serialization

## Prerequisites

Before starting this chapter, you should be comfortable with:

- **Chapter 6** -- Structs and interfaces (struct definitions, embedding, interface satisfaction)
- **Chapter 8** -- Error handling (wrapping, sentinel errors, custom error types)
- **Chapter 12** -- JSON, HTTP, and REST APIs (basic encoding/json usage, HTTP handlers)
- **Chapter 14** -- Testing, benchmarking, and fuzzing (writing benchmarks, interpreting results)
- **Chapter 18** -- Reflection and code generation (reflect package basics, struct tags)
- **Chapter 21** -- Building microservices (service communication patterns)

This chapter is based on **Chapter 4 of "Designing Data-Intensive Applications" by Martin Kleppmann** -- the chapter that explains how data flows between processes. DDIA Chapter 4 covers encoding formats (JSON, XML, Protocol Buffers, Thrift, Avro), schema evolution, and the dataflow patterns (databases, service calls, message passing) that depend on encoding. We will take every concept from that chapter and implement it in Go, comparing formats side by side with real benchmarks.

If you come from Node.js, you have likely used `JSON.stringify()` and `JSON.parse()` for everything -- REST APIs, configuration files, inter-service communication, database storage. That simplicity is both a strength and a trap. JSON is human-readable but verbose, schema-less but fragile, universal but slow. This chapter will show you the entire spectrum of encoding options available in Go, when each one is the right choice, and how schema evolution lets your systems evolve without breaking.

By the end of this chapter, you will understand why encoding is not just "serialization" -- it is a contract between the past, present, and future versions of your software. You will have built working examples of JSON, Protocol Buffers, MessagePack, Avro, custom binary protocols, a schema registry, and a multi-format API server.

---

## Table of Contents

1. [Why Encoding Matters](#1-why-encoding-matters)
2. [JSON in Go (Deep Dive)](#2-json-in-go-deep-dive)
3. [Protocol Buffers in Go](#3-protocol-buffers-in-go)
4. [gRPC with Protocol Buffers](#4-grpc-with-protocol-buffers)
5. [MessagePack](#5-messagepack)
6. [Avro](#6-avro)
7. [Binary Encoding Formats Comparison](#7-binary-encoding-formats-comparison)
8. [Schema Evolution Strategies](#8-schema-evolution-strategies)
9. [Encoding for Storage vs Network](#9-encoding-for-storage-vs-network)
10. [Building a Schema Registry in Go](#10-building-a-schema-registry-in-go)
11. [Custom Binary Protocols](#11-custom-binary-protocols)
12. [Benchmarking Serialization](#12-benchmarking-serialization)
13. [Real-World Example: Multi-Format API](#13-real-world-example-multi-format-api)
14. [Key Takeaways](#14-key-takeaways)
15. [Practice Exercises](#15-practice-exercises)

---

## 1. Why Encoding Matters

### The Two Representations of Data

Every piece of data in your program exists in two fundamentally different forms:

1. **In-memory representation.** In Go, this is structs, slices, maps, pointers -- data structures optimized for CPU access. A `User` struct sits in memory with fields aligned for fast reads, pointers to heap-allocated strings, and slice headers pointing to underlying arrays. In Node.js, this is JavaScript objects with V8's hidden classes and property maps.

2. **On-the-wire representation.** When you need to write data to a file, send it over the network, or store it in a database, you must encode it as a self-contained sequence of bytes. The receiving side (which might be a different process, a different machine, or the same program running six months from now) must decode those bytes back into its own in-memory representation.

The process of going from in-memory to bytes is called **encoding** (also serialization, marshalling). The reverse is **decoding** (deserialization, unmarshalling).

```
In-Memory (Go structs)          On-the-Wire (bytes)
┌─────────────────────┐         ┌─────────────────────────┐
│ type User struct {  │         │ {"name":"Alice",        │
│   Name string       │ ──────► │  "age":30,              │
│   Age  int          │ Marshal │  "email":"a@b.com"}     │
│   Email string      │         │                         │
│ }                   │ ◄────── │ (JSON, Protobuf, Avro,  │
│                     │Unmarshal│  MessagePack, binary...) │
└─────────────────────┘         └─────────────────────────┘
```

### Why Not Just Use JSON Everywhere?

JSON is the lingua franca of web APIs, and for good reason. It is human-readable, universally supported, and requires no schema definition. But it has real problems:

| Problem | Description |
|---------|-------------|
| **No schema enforcement** | A sender can omit fields, add unexpected fields, or change field types. The receiver discovers this at runtime, usually in production at 3 AM. |
| **Verbose** | Field names are repeated in every single record. A million-record dataset wastes megabytes on repeated `"firstName"` strings. |
| **Ambiguous numbers** | JSON has no distinction between integers and floats. The number `42` could be an int8, int32, int64, or float64. Go's `encoding/json` defaults to `float64` for `interface{}` values, silently losing precision for large integers. |
| **No binary data** | JSON has no byte-string type. Binary data must be Base64-encoded, inflating size by 33%. |
| **Slow parsing** | Text-based parsing is inherently slower than binary format parsing. Every number must be parsed from decimal characters. |

### Forward and Backward Compatibility (DDIA Core Concept)

Martin Kleppmann emphasizes that in any real system, you cannot upgrade all components simultaneously. Old code and new code coexist. This creates two requirements:

- **Backward compatibility.** Newer code can read data written by older code. This is usually straightforward -- the new code knows about old formats and handles missing new fields with defaults.

- **Forward compatibility.** Older code can read data written by newer code. This is harder -- the old code must gracefully ignore fields it does not understand without crashing or losing data.

Both properties are essential. When you do a rolling deployment of a microservice, the old version and new version run simultaneously. The old version might read messages produced by the new version (forward compatibility) and the new version reads data stored by the old version (backward compatibility).

```go
package main

import (
	"encoding/json"
	"fmt"
)

// Version 1 of our User struct -- the original deployment.
type UserV1 struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

// Version 2 adds a Phone field and an Age field.
// Backward compatibility: V2 code can read V1 data (Phone/Age will be zero values).
// Forward compatibility: V1 code can read V2 data (Phone/Age will be ignored).
type UserV2 struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Phone string `json:"phone,omitempty"`
	Age   int    `json:"age,omitempty"`
}

func main() {
	// V1 writes data
	v1User := UserV1{Name: "Alice", Email: "alice@example.com"}
	data, _ := json.Marshal(v1User)
	fmt.Printf("V1 encoded: %s\n", data)

	// V2 reads V1 data (backward compatibility)
	var v2User UserV2
	json.Unmarshal(data, &v2User)
	fmt.Printf("V2 reads V1 data: %+v\n", v2User)
	fmt.Printf("  Phone is zero value: %q\n", v2User.Phone)
	fmt.Printf("  Age is zero value: %d\n", v2User.Age)

	// V2 writes data
	v2Full := UserV2{Name: "Bob", Email: "bob@example.com", Phone: "+1-555-0100", Age: 25}
	data2, _ := json.Marshal(v2Full)
	fmt.Printf("\nV2 encoded: %s\n", data2)

	// V1 reads V2 data (forward compatibility)
	var v1Read UserV1
	json.Unmarshal(data2, &v1Read)
	fmt.Printf("V1 reads V2 data: %+v\n", v1Read)
	fmt.Printf("  Phone and Age are silently ignored\n")
}
```

This example shows JSON's natural forward and backward compatibility. But JSON achieves this by being completely schema-less -- which means bugs from field name typos, wrong types, or missing required fields are only caught at runtime. Schema-based formats like Protocol Buffers and Avro provide the same compatibility guarantees with compile-time safety.

### Node.js Comparison

In Node.js, encoding is deceptively simple:

```javascript
// Node.js -- everything is JSON by default
const user = { name: "Alice", age: 30 };
const encoded = JSON.stringify(user);    // string
const decoded = JSON.parse(encoded);     // object

// No type safety. decoded.age could be anything.
// No schema. A typo like decoded.nme silently returns undefined.
// No binary option without third-party libraries.
```

In Go, you get type safety at the cost of explicit struct definitions and tags. But as we will see, Go's ecosystem gives you the full spectrum from JSON to custom binary protocols, with schema evolution support at every level.

---

## 2. JSON in Go (Deep Dive)

### Basic Marshaling and Unmarshaling

Go's `encoding/json` package uses struct tags to map between Go struct fields and JSON keys. This is where most Go developers start, and where many stop. Let us go much deeper.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"time"
)

// Product demonstrates common JSON struct tag options.
type Product struct {
	// "id" is the JSON key name. Without the tag, it would be "ID" (the Go field name).
	ID int64 `json:"id"`

	// "name" with no options -- straightforward mapping.
	Name string `json:"name"`

	// "price" -- JSON has no decimal type, so float64 is the natural fit.
	Price float64 `json:"price"`

	// "in_stock" with omitempty -- if the bool is false (zero value), omit it from JSON output.
	InStock bool `json:"in_stock,omitempty"`

	// "tags" -- slices become JSON arrays. nil slices become null, empty slices become [].
	Tags []string `json:"tags,omitempty"`

	// "-" means this field is completely ignored by encoding/json.
	InternalCode string `json:"-"`

	// ",string" tells encoding/json to encode/decode a number as a JSON string.
	// Useful when JavaScript clients cannot handle 64-bit integers.
	SKU int64 `json:"sku,string"`

	// Embedded struct fields are "flattened" into the parent JSON object.
	Metadata
}

type Metadata struct {
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

func main() {
	p := Product{
		ID:           1,
		Name:         "Go Programming Book",
		Price:        49.99,
		InStock:      true,
		Tags:         []string{"programming", "golang"},
		InternalCode: "SECRET-123", // This will NOT appear in JSON
		SKU:          9780134190440,
		Metadata: Metadata{
			CreatedAt: time.Date(2025, 1, 15, 10, 0, 0, 0, time.UTC),
			UpdatedAt: time.Now().UTC(),
		},
	}

	// Marshal with indentation for readability
	data, err := json.MarshalIndent(p, "", "  ")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Marshaled JSON:\n%s\n\n", data)

	// Unmarshal back
	var decoded Product
	if err := json.Unmarshal(data, &decoded); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Decoded struct: %+v\n", decoded)
	fmt.Printf("InternalCode (should be empty): %q\n", decoded.InternalCode)
	fmt.Printf("SKU (from string): %d\n", decoded.SKU)
}
```

### The float64 Trap

This is the single most dangerous pitfall in Go JSON handling. When you unmarshal JSON into an `interface{}`, all numbers become `float64`. This silently loses precision for integers larger than 2^53.

```go
package main

import (
	"encoding/json"
	"fmt"
	"math"
	"strings"
)

func main() {
	// JavaScript's Number.MAX_SAFE_INTEGER is 2^53 - 1 = 9007199254740991
	// Go's int64 max is 2^63 - 1 = 9223372036854775807
	// When JSON is decoded into interface{}, numbers become float64,
	// which only has 53 bits of mantissa precision.

	// This is a valid int64 value but larger than 2^53
	largeID := int64(9007199254740993) // 2^53 + 1
	jsonStr := fmt.Sprintf(`{"id": %d}`, largeID)

	// Decode into interface{} -- THE TRAP
	var generic map[string]interface{}
	json.Unmarshal([]byte(jsonStr), &generic)
	gotFloat := generic["id"].(float64)
	gotInt := int64(gotFloat)
	fmt.Printf("Original value:     %d\n", largeID)
	fmt.Printf("After float64 cast: %d\n", gotInt)
	fmt.Printf("Precision lost:     %v\n", largeID != gotInt)
	fmt.Printf("float64 mantissa:   %d bits\n", 53)
	fmt.Printf("Max safe integer:   %d\n", int64(math.Pow(2, 53)-1))

	// SOLUTION 1: Decode into a typed struct (always prefer this)
	type Record struct {
		ID int64 `json:"id"`
	}
	var typed Record
	json.Unmarshal([]byte(jsonStr), &typed)
	fmt.Printf("\nTyped decode:       %d (correct!)\n", typed.ID)

	// SOLUTION 2: Use json.Decoder with UseNumber()
	decoder := json.NewDecoder(strings.NewReader(jsonStr))
	decoder.UseNumber()
	var withNumber map[string]interface{}
	decoder.Decode(&withNumber)
	num := withNumber["id"].(json.Number)
	safeInt, _ := num.Int64()
	fmt.Printf("UseNumber decode:   %d (correct!)\n", safeInt)

	// SOLUTION 3: Use the ",string" tag for known large integers
	type SafeRecord struct {
		// Expects JSON like: {"id": "9007199254740993"}
		ID int64 `json:"id,string"`
	}
	safeJSON := fmt.Sprintf(`{"id": "%d"}`, largeID)
	var safe SafeRecord
	json.Unmarshal([]byte(safeJSON), &safe)
	fmt.Printf("String tag decode:  %d (correct!)\n", safe.ID)
}
```

### Null vs Absent vs Zero Value

JSON distinguishes between a field being `null`, absent entirely, and having a zero value. Go's `encoding/json` conflates these in ways that matter for APIs.

```go
package main

import (
	"encoding/json"
	"fmt"
)

// NaiveUser demonstrates the problem: you cannot distinguish
// between "user set age to 0" and "user did not send age at all."
type NaiveUser struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

// PointerUser uses a pointer field to distinguish absent (nil) from present (even if zero).
type PointerUser struct {
	Name *string `json:"name"`
	Age  *int    `json:"age"`
}

// NullableField is a generic wrapper for fields that can be null, absent, or present.
// This is the most precise approach, useful for PATCH endpoints.
type NullableField[T any] struct {
	Value   T
	Present bool // true if the field was in the JSON at all
	Null    bool // true if the field was explicitly null
}

func (n *NullableField[T]) UnmarshalJSON(data []byte) error {
	n.Present = true
	if string(data) == "null" {
		n.Null = true
		return nil
	}
	return json.Unmarshal(data, &n.Value)
}

func (n NullableField[T]) MarshalJSON() ([]byte, error) {
	if !n.Present || n.Null {
		return []byte("null"), nil
	}
	return json.Marshal(n.Value)
}

type PreciseUser struct {
	Name NullableField[string] `json:"name"`
	Age  NullableField[int]    `json:"age"`
}

func main() {
	// Three different JSON inputs
	inputs := []string{
		`{"name": "Alice", "age": 30}`,  // Both present with values
		`{"name": "Alice"}`,              // Age absent
		`{"name": "Alice", "age": null}`, // Age explicitly null
		`{"name": "Alice", "age": 0}`,    // Age is zero (valid age for newborn?)
	}

	fmt.Println("=== Naive approach (cannot distinguish) ===")
	for _, input := range inputs {
		var u NaiveUser
		json.Unmarshal([]byte(input), &u)
		fmt.Printf("Input: %-40s -> Age=%d\n", input, u.Age)
	}

	fmt.Println("\n=== Pointer approach (null/absent conflated, but zero works) ===")
	for _, input := range inputs {
		var u PointerUser
		json.Unmarshal([]byte(input), &u)
		if u.Age == nil {
			fmt.Printf("Input: %-40s -> Age=nil\n", input)
		} else {
			fmt.Printf("Input: %-40s -> Age=%d\n", input, *u.Age)
		}
	}

	fmt.Println("\n=== Precise approach (full distinction) ===")
	for _, input := range inputs {
		var u PreciseUser
		json.Unmarshal([]byte(input), &u)
		switch {
		case !u.Age.Present:
			fmt.Printf("Input: %-40s -> Age=ABSENT\n", input)
		case u.Age.Null:
			fmt.Printf("Input: %-40s -> Age=NULL\n", input)
		default:
			fmt.Printf("Input: %-40s -> Age=%d\n", input, u.Age.Value)
		}
	}
}
```

### Custom Marshalers

When the default JSON representation is not what you need, implement `json.Marshaler` and `json.Unmarshaler`.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"strings"
	"time"
)

// Duration wraps time.Duration to provide human-readable JSON encoding.
// Go's time.Duration marshals as an integer (nanoseconds) by default, which is unreadable.
type Duration struct {
	time.Duration
}

func (d Duration) MarshalJSON() ([]byte, error) {
	return json.Marshal(d.String()) // "5m30s" instead of 330000000000
}

func (d *Duration) UnmarshalJSON(data []byte) error {
	var s string
	if err := json.Unmarshal(data, &s); err != nil {
		return err
	}
	parsed, err := time.ParseDuration(s)
	if err != nil {
		return err
	}
	d.Duration = parsed
	return nil
}

// Status represents an enum-like type that marshals as a string.
type Status int

const (
	StatusPending  Status = iota // 0
	StatusActive                 // 1
	StatusInactive               // 2
)

var statusNames = map[Status]string{
	StatusPending:  "pending",
	StatusActive:   "active",
	StatusInactive: "inactive",
}

var statusValues = map[string]Status{
	"pending":  StatusPending,
	"active":   StatusActive,
	"inactive": StatusInactive,
}

func (s Status) MarshalJSON() ([]byte, error) {
	name, ok := statusNames[s]
	if !ok {
		return nil, fmt.Errorf("unknown status: %d", s)
	}
	return json.Marshal(name)
}

func (s *Status) UnmarshalJSON(data []byte) error {
	var name string
	if err := json.Unmarshal(data, &name); err != nil {
		return err
	}
	val, ok := statusValues[strings.ToLower(name)]
	if !ok {
		return fmt.Errorf("unknown status: %q", name)
	}
	*s = val
	return nil
}

// Config uses both custom types.
type Config struct {
	Name    string   `json:"name"`
	Timeout Duration `json:"timeout"`
	Status  Status   `json:"status"`
}

func main() {
	cfg := Config{
		Name:    "my-service",
		Timeout: Duration{5*time.Minute + 30*time.Second},
		Status:  StatusActive,
	}

	data, err := json.MarshalIndent(cfg, "", "  ")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Custom marshaled:\n%s\n\n", data)

	// Unmarshal from human-readable JSON
	input := `{"name":"other-service","timeout":"2m15s","status":"pending"}`
	var decoded Config
	if err := json.Unmarshal([]byte(input), &decoded); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Decoded: %+v\n", decoded)
	fmt.Printf("Timeout as nanoseconds: %d\n", decoded.Timeout.Nanoseconds())
}
```

### Streaming JSON with Encoder/Decoder

For large payloads or HTTP handlers, streaming is more memory-efficient than marshaling to a byte slice. The `json.Encoder` writes directly to an `io.Writer` and the `json.Decoder` reads from an `io.Reader`.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"strings"
)

// LogEntry represents one line in a JSON Lines (NDJSON) stream.
// JSON Lines is a format where each line is a separate JSON object.
// This is common in log aggregation systems like Elasticsearch.
type LogEntry struct {
	Timestamp string `json:"ts"`
	Level     string `json:"level"`
	Message   string `json:"msg"`
	Service   string `json:"service"`
}

func main() {
	// === Encoding: Stream multiple objects ===
	var buf bytes.Buffer
	encoder := json.NewEncoder(&buf)

	entries := []LogEntry{
		{Timestamp: "2025-01-15T10:00:00Z", Level: "INFO", Message: "Server started", Service: "api"},
		{Timestamp: "2025-01-15T10:00:01Z", Level: "DEBUG", Message: "Connection pool initialized", Service: "api"},
		{Timestamp: "2025-01-15T10:00:02Z", Level: "ERROR", Message: "Database timeout", Service: "worker"},
	}

	for _, entry := range entries {
		if err := encoder.Encode(entry); err != nil {
			log.Fatal(err)
		}
	}

	fmt.Println("=== JSON Lines (NDJSON) Output ===")
	fmt.Print(buf.String())

	// === Decoding: Stream from a reader ===
	fmt.Println("\n=== Streaming Decode ===")
	decoder := json.NewDecoder(strings.NewReader(buf.String()))

	var decoded []LogEntry
	for {
		var entry LogEntry
		err := decoder.Decode(&entry)
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatal(err)
		}
		decoded = append(decoded, entry)
		fmt.Printf("Read entry: [%s] %s - %s\n", entry.Level, entry.Service, entry.Message)
	}

	// === Streaming JSON Array ===
	// For very large arrays, you can decode token by token
	fmt.Println("\n=== Token-based Array Decoding ===")
	bigArray := `[{"ts":"1","level":"INFO","msg":"one","service":"a"},{"ts":"2","level":"WARN","msg":"two","service":"b"}]`
	dec := json.NewDecoder(strings.NewReader(bigArray))

	// Read the opening bracket
	tok, _ := dec.Token()
	fmt.Printf("Opening token: %v\n", tok) // [

	// Read each object one at a time (no need to hold entire array in memory)
	for dec.More() {
		var entry LogEntry
		if err := dec.Decode(&entry); err != nil {
			log.Fatal(err)
		}
		fmt.Printf("Array element: %s - %s\n", entry.Level, entry.Message)
	}

	// Read the closing bracket
	tok, _ = dec.Token()
	fmt.Printf("Closing token: %v\n", tok) // ]
}
```

### Raw JSON and Delayed Parsing

Sometimes you receive JSON with a polymorphic field -- the type of one field depends on the value of another. `json.RawMessage` lets you delay parsing of that field.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

// Event represents a message where the Payload structure depends on the Type.
// This is extremely common in webhook systems (Stripe, GitHub, etc.).
type Event struct {
	Type    string          `json:"type"`
	Payload json.RawMessage `json:"payload"` // Delay parsing until we know the type
}

type UserCreated struct {
	UserID int    `json:"user_id"`
	Name   string `json:"name"`
	Email  string `json:"email"`
}

type OrderPlaced struct {
	OrderID string  `json:"order_id"`
	Amount  float64 `json:"amount"`
	Items   int     `json:"items"`
}

func processEvent(data []byte) error {
	var event Event
	if err := json.Unmarshal(data, &event); err != nil {
		return fmt.Errorf("unmarshal event envelope: %w", err)
	}

	switch event.Type {
	case "user.created":
		var payload UserCreated
		if err := json.Unmarshal(event.Payload, &payload); err != nil {
			return fmt.Errorf("unmarshal user.created payload: %w", err)
		}
		fmt.Printf("New user: %s (%s) with ID %d\n", payload.Name, payload.Email, payload.UserID)

	case "order.placed":
		var payload OrderPlaced
		if err := json.Unmarshal(event.Payload, &payload); err != nil {
			return fmt.Errorf("unmarshal order.placed payload: %w", err)
		}
		fmt.Printf("New order: %s for $%.2f with %d items\n", payload.OrderID, payload.Amount, payload.Items)

	default:
		// Forward compatibility: unknown event types are not errors.
		// As per DDIA, old code should gracefully handle new event types.
		fmt.Printf("Unknown event type %q -- ignoring (forward compatibility)\n", event.Type)
	}
	return nil
}

func main() {
	events := []string{
		`{"type":"user.created","payload":{"user_id":42,"name":"Alice","email":"alice@example.com"}}`,
		`{"type":"order.placed","payload":{"order_id":"ORD-001","amount":99.99,"items":3}}`,
		`{"type":"subscription.renewed","payload":{"sub_id":"SUB-001"}}`, // Unknown to our code
	}

	for _, raw := range events {
		if err := processEvent([]byte(raw)); err != nil {
			log.Printf("Error: %v", err)
		}
	}
}
```

---

## 3. Protocol Buffers in Go

### Why Protocol Buffers?

As Kleppmann explains in DDIA, Protocol Buffers (protobuf) originated at Google as a way to define schemas for data exchange. Unlike JSON, protobuf:

- Uses **field numbers** instead of field names -- this is the key to schema evolution
- Has a **binary encoding** that is compact and fast to parse
- Requires a **schema definition** (.proto file) that serves as documentation and a contract
- Generates **type-safe code** in any target language from the schema

The field numbering system is the insight that makes protobuf schema evolution work. Field 1 is always field 1, regardless of what it is named. You can rename fields, add new fields (with new numbers), and remove old fields (by never reusing their numbers) without breaking compatibility.

### Proto3 Syntax and Schema Definition

```protobuf
// user.proto
// This is a Protocol Buffers v3 schema definition.
// Proto3 is the current version -- proto2 had required fields, which caused
// more compatibility headaches than they solved.

syntax = "proto3";

package userservice;

// The Go package path for generated code
option go_package = "github.com/example/userservice/pb";

// Basic message with scalar types.
// Each field has a TYPE, NAME, and FIELD NUMBER.
// Field numbers 1-15 use 1 byte in the encoding.
// Field numbers 16-2047 use 2 bytes.
// So put your most frequently used fields at numbers 1-15.
message User {
  int64 id = 1;                    // 8 bytes max, varint encoded
  string name = 2;                 // UTF-8 string
  string email = 3;
  int32 age = 4;                   // Could also be uint32 if never negative
  UserStatus status = 5;           // Enum value
  repeated string tags = 6;        // Repeated = list/array
  Address address = 7;             // Nested message
  optional string phone = 8;       // Explicit optional (proto3 normally has implicit defaults)
  map<string, string> metadata = 9; // Map type
}

// Enums start at 0 (the default/zero value).
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;     // Always have an UNSPECIFIED zero value
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}

// Nested message type.
message Address {
  string street = 1;
  string city = 2;
  string state = 3;
  string zip_code = 4;
  string country = 5;
}

// Oneof: exactly one of these fields can be set.
// Useful for polymorphic types (like our JSON event example).
message Notification {
  int64 user_id = 1;
  string message = 2;

  oneof channel {
    EmailChannel email = 10;
    SMSChannel sms = 11;
    PushChannel push = 12;
  }
}

message EmailChannel {
  string to_address = 1;
  string subject = 2;
}

message SMSChannel {
  string phone_number = 1;
}

message PushChannel {
  string device_token = 1;
}
```

### Code Generation

To generate Go code from the `.proto` file:

```bash
# Install the protobuf compiler and Go plugin
# brew install protobuf              (macOS)
# go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
# go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generate Go code
# protoc --go_out=. --go_opt=paths=source_relative \
#        --go-grpc_out=. --go-grpc_opt=paths=source_relative \
#        user.proto
```

### Working with Generated Code

The following example shows how to use protobuf in Go without requiring actual code generation -- we use the `google.golang.org/protobuf` library's dynamic message features and also show what the generated code pattern looks like.

```go
package main

import (
	"fmt"
	"log"

	"google.golang.org/protobuf/encoding/protojson"
	"google.golang.org/protobuf/proto"
	"google.golang.org/protobuf/types/known/structpb"
	"google.golang.org/protobuf/types/known/timestamppb"
)

// NOTE: In a real project, these types would be generated by protoc.
// Here we show the pattern that the generated code follows.
// For this example to compile, you would run:
//   go get google.golang.org/protobuf

// This example demonstrates protobuf encoding/decoding using the
// well-known types that ship with the protobuf library.

func main() {
	// === Using well-known types ===
	// protobuf provides standard types for common patterns.
	// google.protobuf.Timestamp -- for dates/times
	ts := timestamppb.Now()
	fmt.Printf("Timestamp: %v\n", ts.AsTime())

	// google.protobuf.Struct -- for arbitrary JSON-like data
	s, err := structpb.NewStruct(map[string]interface{}{
		"name":  "Alice",
		"age":   30.0,
		"tags":  []interface{}{"admin", "user"},
		"email": "alice@example.com",
	})
	if err != nil {
		log.Fatal(err)
	}

	// Marshal to binary protobuf
	binaryData, err := proto.Marshal(s)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Binary protobuf size: %d bytes\n", len(binaryData))

	// Marshal to JSON (protobuf's canonical JSON format)
	jsonData, err := protojson.Marshal(s)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Proto JSON: %s\n", jsonData)
	fmt.Printf("Proto JSON size: %d bytes\n", len(jsonData))

	// Unmarshal from binary
	decoded := &structpb.Struct{}
	if err := proto.Unmarshal(binaryData, decoded); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Decoded name: %s\n", decoded.Fields["name"].GetStringValue())
	fmt.Printf("Decoded age: %.0f\n", decoded.Fields["age"].GetNumberValue())

	// Size comparison
	fmt.Printf("\n=== Size Comparison ===\n")
	fmt.Printf("Binary protobuf: %d bytes\n", len(binaryData))
	fmt.Printf("Proto JSON:      %d bytes\n", len(jsonData))
	fmt.Printf("Savings:         %.1f%%\n",
		(1.0-float64(len(binaryData))/float64(len(jsonData)))*100)
}
```

### Schema Evolution with Field Numbers

This is the heart of protobuf's design and connects directly to DDIA's discussion of schema evolution. The rules are:

```go
package main

import "fmt"

func main() {
	// Protobuf Schema Evolution Rules
	// ================================
	//
	// SAFE CHANGES (maintain forward and backward compatibility):
	//
	// 1. ADD a new field with a new field number.
	//    - Old code ignores the unknown field (forward compatible).
	//    - New code uses the default value if field is missing (backward compatible).
	//
	// 2. REMOVE a field (stop using it).
	//    - Old code still sends it; new code ignores it.
	//    - New code doesn't send it; old code uses default value.
	//    - CRITICAL: Never reuse the field number! Mark it as reserved.
	//
	// 3. RENAME a field.
	//    - Field numbers are what matter on the wire, not names.
	//    - Names only matter in the generated code.
	//
	// BREAKING CHANGES (will cause data corruption or errors):
	//
	// 1. CHANGE a field's type to an incompatible type.
	//    - int32 -> string is BREAKING.
	//    - int32 -> int64 is SAFE (values are widened).
	//
	// 2. REUSE a deleted field number.
	//    - Old data with the old field will be misinterpreted.
	//    - Use "reserved" to prevent this.
	//
	// 3. CHANGE a field from singular to repeated (or vice versa).
	//    - Wire format differs; will cause decode errors.

	fmt.Println("=== Proto Schema Evolution Examples ===")
	fmt.Println()

	// Version 1 of the schema:
	v1Schema := `
message UserV1 {
  int64 id = 1;
  string name = 2;
  string email = 3;
}`

	// Version 2: Added phone, removed nothing yet
	v2Schema := `
message UserV2 {
  int64 id = 1;
  string name = 2;
  string email = 3;
  string phone = 4;          // NEW: added in v2
  int32 age = 5;             // NEW: added in v2
}`

	// Version 3: Removed email (deprecated), added address
	// CRITICAL: email's field number 3 is reserved forever
	v3Schema := `
message UserV3 {
  reserved 3;                // NEVER reuse field 3 (was email)
  reserved "email";          // Prevent accidental reuse of the name too

  int64 id = 1;
  string name = 2;
  // field 3 intentionally skipped (was email)
  string phone = 4;
  int32 age = 5;
  Address address = 6;       // NEW: added in v3
}`

	fmt.Println("V1 Schema:", v1Schema)
	fmt.Println("V2 Schema:", v2Schema)
	fmt.Println("V3 Schema:", v3Schema)

	fmt.Println("Compatibility Matrix:")
	fmt.Println("  V1 writer -> V2 reader: OK (phone/age default to zero)")
	fmt.Println("  V2 writer -> V1 reader: OK (phone/age ignored)")
	fmt.Println("  V2 writer -> V3 reader: OK (email ignored, address defaults)")
	fmt.Println("  V3 writer -> V1 reader: OK (phone/age/address ignored)")
}
```

### Protobuf Encoding Internals

Understanding the wire format helps you make informed decisions about field types and ordering.

```go
package main

import (
	"fmt"
	"math"
)

// Protobuf uses a Tag-Length-Value (TLV) encoding.
// Each field is encoded as:
//   [field_number << 3 | wire_type] [length (for length-delimited)] [value]
//
// Wire types:
//   0 = Varint (int32, int64, uint32, uint64, sint32, sint64, bool, enum)
//   1 = 64-bit (fixed64, sfixed64, double)
//   2 = Length-delimited (string, bytes, embedded messages, repeated fields)
//   5 = 32-bit (fixed32, sfixed32, float)

func encodeVarint(value uint64) []byte {
	// Varints encode small numbers in fewer bytes.
	// Each byte uses 7 bits for data and 1 bit (MSB) as a continuation flag.
	if value == 0 {
		return []byte{0}
	}
	var buf []byte
	for value > 0 {
		b := byte(value & 0x7F) // Take 7 bits
		value >>= 7
		if value > 0 {
			b |= 0x80 // Set continuation bit
		}
		buf = append(buf, b)
	}
	return buf
}

func decodeVarint(data []byte) (uint64, int) {
	var value uint64
	for i, b := range data {
		value |= uint64(b&0x7F) << (7 * i)
		if b&0x80 == 0 {
			return value, i + 1
		}
	}
	return value, len(data)
}

func encodeZigZag(n int64) uint64 {
	// ZigZag encoding maps signed integers to unsigned integers
	// so that small absolute values have small varint encodings.
	// -1 -> 1, 1 -> 2, -2 -> 3, 2 -> 4, etc.
	// Without zigzag, -1 would be encoded as a 10-byte varint (all 1 bits).
	return uint64((n << 1) ^ (n >> 63))
}

func decodeZigZag(n uint64) int64 {
	return int64((n >> 1) ^ -(n & 1))
}

func main() {
	fmt.Println("=== Varint Encoding ===")
	values := []uint64{0, 1, 127, 128, 300, 16384, math.MaxUint32, math.MaxUint64}
	for _, v := range values {
		encoded := encodeVarint(v)
		decoded, bytesRead := decodeVarint(encoded)
		fmt.Printf("Value: %-20d  Bytes: %-3d  Encoded: %v  Decoded: %d\n",
			v, bytesRead, encoded, decoded)
	}

	fmt.Println("\n=== ZigZag Encoding (for signed integers) ===")
	signedValues := []int64{0, -1, 1, -2, 2, -64, 64, -128, 128}
	for _, v := range signedValues {
		zigzag := encodeZigZag(v)
		varintBytes := encodeVarint(zigzag)
		fmt.Printf("Signed: %-5d  ZigZag: %-5d  Varint bytes: %d\n",
			v, zigzag, len(varintBytes))
	}

	fmt.Println("\n=== Why Field Numbers 1-15 Are Special ===")
	// Field tag = (field_number << 3) | wire_type
	// For field_number 1 with wire_type 0 (varint): tag = (1 << 3) | 0 = 8
	// This fits in one byte (< 128).
	// For field_number 16 with wire_type 0: tag = (16 << 3) | 0 = 128
	// This requires two bytes as a varint.
	for _, fn := range []int{1, 15, 16, 100, 2047, 2048} {
		tag := uint64(fn<<3) | 0 // wire type 0
		encoded := encodeVarint(tag)
		fmt.Printf("Field #%-5d  Tag: %-6d  Tag bytes: %d\n", fn, tag, len(encoded))
	}
}
```

---

## 4. gRPC with Protocol Buffers

### Service Definition

gRPC uses protobuf to define both the message types and the service interface. This is DDIA's "service calls" dataflow pattern -- where encoding is used for request/response between services.

```protobuf
// userservice.proto
syntax = "proto3";

package userservice;
option go_package = "github.com/example/userservice/pb";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Service definition -- this generates both client and server code.
service UserService {
  // Unary RPC: one request, one response.
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);

  // Server streaming: client sends one request, server sends a stream of responses.
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client streaming: client sends a stream, server responds once.
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);

  // Bidirectional streaming: both sides send streams.
  rpc UserChat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
  int64 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message BulkCreateResponse {
  int32 created_count = 1;
  repeated int64 ids = 2;
}

message ChatMessage {
  int64 user_id = 1;
  string content = 2;
  google.protobuf.Timestamp sent_at = 3;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  google.protobuf.Timestamp created_at = 5;
}
```

### gRPC Server Implementation

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/timestamppb"

	// In a real project this would be your generated protobuf package:
	// pb "github.com/example/userservice/pb"
	// For illustration, we define the interfaces inline.
)

// UserServer implements the UserService gRPC server.
// In practice, this struct would embed pb.UnimplementedUserServiceServer.
type UserServer struct {
	mu    sync.RWMutex
	users map[int64]*User
	nextID int64
}

// User mirrors the protobuf-generated User message.
type User struct {
	ID        int64
	Name      string
	Email     string
	Age       int32
	CreatedAt time.Time
}

func NewUserServer() *UserServer {
	return &UserServer{
		users:  make(map[int64]*User),
		nextID: 1,
	}
}

// GetUser is a unary RPC -- one request, one response.
func (s *UserServer) GetUser(ctx context.Context, id int64) (*User, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	user, ok := s.users[id]
	if !ok {
		// gRPC uses status codes instead of HTTP status codes.
		// This is the gRPC equivalent of 404.
		return nil, status.Errorf(codes.NotFound, "user %d not found", id)
	}
	return user, nil
}

// CreateUser creates a new user.
func (s *UserServer) CreateUser(ctx context.Context, name, email string, age int32) (*User, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	user := &User{
		ID:        s.nextID,
		Name:      name,
		Email:     email,
		Age:       age,
		CreatedAt: time.Now(),
	}
	s.users[user.ID] = user
	s.nextID++
	return user, nil
}

// === gRPC Interceptors ===
// Interceptors are like HTTP middleware but for gRPC.

// LoggingUnaryInterceptor logs every unary RPC call.
func LoggingUnaryInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()

	// Extract metadata (like HTTP headers)
	md, ok := metadata.FromIncomingContext(ctx)
	if ok {
		if vals := md.Get("request-id"); len(vals) > 0 {
			log.Printf("[gRPC] request_id=%s method=%s", vals[0], info.FullMethod)
		}
	}

	// Call the actual handler
	resp, err := handler(ctx, req)

	// Log the result
	duration := time.Since(start)
	if err != nil {
		st, _ := status.FromError(err)
		log.Printf("[gRPC] method=%s duration=%v code=%s error=%s",
			info.FullMethod, duration, st.Code(), st.Message())
	} else {
		log.Printf("[gRPC] method=%s duration=%v code=OK", info.FullMethod, duration)
	}

	return resp, err
}

// StreamLoggingInterceptor logs stream RPCs.
func StreamLoggingInterceptor(
	srv interface{},
	ss grpc.ServerStream,
	info *grpc.StreamServerInfo,
	handler grpc.StreamHandler,
) error {
	start := time.Now()
	log.Printf("[gRPC Stream] starting method=%s", info.FullMethod)

	err := handler(srv, ss)

	duration := time.Since(start)
	log.Printf("[gRPC Stream] completed method=%s duration=%v", info.FullMethod, duration)
	return err
}

func main() {
	// Create a gRPC server with interceptors
	server := grpc.NewServer(
		grpc.UnaryInterceptor(LoggingUnaryInterceptor),
		grpc.StreamInterceptor(StreamLoggingInterceptor),
	)

	userServer := NewUserServer()

	// In a real implementation, you would register the generated service:
	// pb.RegisterUserServiceServer(server, userServer)

	// Start listening
	listener, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	fmt.Println("gRPC server starting on :50051")
	fmt.Printf("Server has %d registered users\n", len(userServer.users))
	fmt.Println()

	// Demonstrate the concepts without actually running the server
	// (since we don't have generated code in this example)
	fmt.Println("=== gRPC Concepts ===")
	fmt.Println()
	fmt.Println("1. Unary RPC: Client sends one request, server sends one response.")
	fmt.Println("   Like a regular function call, but across the network.")
	fmt.Println("   Example: GetUser(id=42) -> User{name: 'Alice'}")
	fmt.Println()
	fmt.Println("2. Server Streaming: Client sends one request, server sends multiple responses.")
	fmt.Println("   The server pushes data as it becomes available.")
	fmt.Println("   Example: ListUsers(page_size=10) -> stream of User messages")
	fmt.Println()
	fmt.Println("3. Client Streaming: Client sends multiple messages, server responds once.")
	fmt.Println("   For batch operations where client sends data incrementally.")
	fmt.Println("   Example: BulkCreateUsers(stream of CreateUserRequest) -> BulkCreateResponse")
	fmt.Println()
	fmt.Println("4. Bidirectional Streaming: Both sides send streams simultaneously.")
	fmt.Println("   For real-time communication like chat.")
	fmt.Println("   Example: UserChat(stream messages) <-> stream messages")
	fmt.Println()
	fmt.Println("=== gRPC vs REST (DDIA Perspective) ===")
	fmt.Println()
	fmt.Println("REST (JSON over HTTP):")
	fmt.Println("  + Human-readable, easy to debug with curl")
	fmt.Println("  + Universal browser/client support")
	fmt.Println("  - Text-based encoding (slow, verbose)")
	fmt.Println("  - No built-in streaming")
	fmt.Println("  - No schema enforcement")
	fmt.Println()
	fmt.Println("gRPC (Protobuf over HTTP/2):")
	fmt.Println("  + Binary encoding (fast, compact)")
	fmt.Println("  + Native streaming support")
	fmt.Println("  + Strong schema with code generation")
	fmt.Println("  + Built-in deadlines, cancellation, metadata")
	fmt.Println("  - Harder to debug (binary on the wire)")
	fmt.Println("  - Requires HTTP/2")
	fmt.Println("  - Limited browser support (needs grpc-web proxy)")

	// We demonstrate timestamppb usage as it's a real import that works
	ts := timestamppb.Now()
	fmt.Printf("\nTimestamp now: %v\n", ts.AsTime())

	_ = server
	_ = listener
	_ = io.EOF // acknowledge import for streaming patterns
}
```

### Node.js Comparison: gRPC

```javascript
// Node.js gRPC client (for comparison)
// npm install @grpc/grpc-js @grpc/proto-loader

const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

// Load the .proto file at runtime (no code generation needed)
const packageDefinition = protoLoader.loadSync('user.proto');
const userProto = grpc.loadPackageDefinition(packageDefinition).userservice;

// Create a client
const client = new userProto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary call
client.getUser({ id: 42 }, (err, user) => {
  if (err) {
    console.error('Error:', err.message);
    return;
  }
  console.log('User:', user);
});

// Server streaming
const stream = client.listUsers({ page_size: 10 });
stream.on('data', (user) => console.log('Received user:', user));
stream.on('end', () => console.log('Stream ended'));
stream.on('error', (err) => console.error('Stream error:', err));

// Key difference from Go:
// - Node.js can load .proto files at runtime (dynamic)
// - Go generates code at build time (static)
// - Go catches schema mismatches at compile time
// - Node.js discovers them at runtime
```

---

## 5. MessagePack

### Binary JSON

MessagePack is a binary serialization format that is structurally equivalent to JSON but encoded in binary. It is like JSON with the field names included, but using a compact binary encoding instead of text. This makes it a good fit when you want JSON's schema-less flexibility with better performance.

As Kleppmann notes in DDIA, MessagePack and similar binary JSON encodings save some space (typically 30-50% smaller than JSON) but do not provide schema evolution guarantees. The field names are still encoded in every message, unlike protobuf where only field numbers appear on the wire.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/vmihailenco/msgpack/v5"
)

// Event demonstrates MessagePack usage. The struct tags work like JSON tags.
type Event struct {
	ID        string    `msgpack:"id"        json:"id"`
	Type      string    `msgpack:"type"      json:"type"`
	Timestamp time.Time `msgpack:"timestamp" json:"timestamp"`
	UserID    int64     `msgpack:"user_id"   json:"user_id"`
	Payload   Payload   `msgpack:"payload"   json:"payload"`
}

type Payload struct {
	Action string            `msgpack:"action"   json:"action"`
	Path   string            `msgpack:"path"     json:"path"`
	Tags   []string          `msgpack:"tags"     json:"tags"`
	Extra  map[string]string `msgpack:"extra"    json:"extra"`
}

func main() {
	event := Event{
		ID:        "evt-001",
		Type:      "page_view",
		Timestamp: time.Date(2025, 6, 15, 14, 30, 0, 0, time.UTC),
		UserID:    42,
		Payload: Payload{
			Action: "click",
			Path:   "/products/shoes",
			Tags:   []string{"mobile", "organic", "returning-user"},
			Extra: map[string]string{
				"browser":    "Chrome",
				"os":         "iOS",
				"session_id": "abc-123-def",
			},
		},
	}

	// === Encode as MessagePack ===
	msgpackData, err := msgpack.Marshal(event)
	if err != nil {
		log.Fatal(err)
	}

	// === Encode as JSON for comparison ===
	jsonData, err := json.Marshal(event)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("=== Size Comparison ===")
	fmt.Printf("JSON size:        %d bytes\n", len(jsonData))
	fmt.Printf("MessagePack size: %d bytes\n", len(msgpackData))
	fmt.Printf("Savings:          %.1f%%\n",
		(1.0-float64(len(msgpackData))/float64(len(jsonData)))*100)

	// === Decode MessagePack ===
	var decoded Event
	if err := msgpack.Unmarshal(msgpackData, &decoded); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\nDecoded: %+v\n", decoded)

	// === MessagePack with interface{} (dynamic typing) ===
	// Unlike protobuf, MessagePack can decode into generic types like JSON
	var generic map[string]interface{}
	if err := msgpack.Unmarshal(msgpackData, &generic); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\nGeneric decode: %v\n", generic)

	// === When to use MessagePack vs JSON vs Protobuf ===
	fmt.Println("\n=== When to Use MessagePack ===")
	fmt.Println()
	fmt.Println("Use MessagePack when:")
	fmt.Println("  - You want JSON compatibility but need smaller messages")
	fmt.Println("  - You need schema-less encoding (no .proto files)")
	fmt.Println("  - You are encoding data for cache storage (Redis, Memcached)")
	fmt.Println("  - You need to encode dynamic/untyped data")
	fmt.Println()
	fmt.Println("Use JSON when:")
	fmt.Println("  - Human readability matters (APIs consumed by browsers)")
	fmt.Println("  - Debugging with curl/Postman is a priority")
	fmt.Println("  - Interoperability with JavaScript clients is critical")
	fmt.Println()
	fmt.Println("Use Protobuf when:")
	fmt.Println("  - You need maximum performance and minimum size")
	fmt.Println("  - Schema evolution with compatibility guarantees matters")
	fmt.Println("  - You control both client and server")
	fmt.Println("  - You want generated, type-safe code in multiple languages")
}
```

### MessagePack with Custom Encoding

```go
package main

import (
	"fmt"
	"log"
	"net"

	"github.com/vmihailenco/msgpack/v5"
)

// IPAddr demonstrates a custom MessagePack encoder for net.IP.
type IPAddr struct {
	net.IP
}

func (ip IPAddr) MarshalMsgpack() ([]byte, error) {
	// Store as raw bytes (4 bytes for IPv4, 16 for IPv6)
	// Much more compact than the string representation
	return msgpack.Marshal([]byte(ip.IP))
}

func (ip *IPAddr) UnmarshalMsgpack(data []byte) error {
	var raw []byte
	if err := msgpack.Unmarshal(data, &raw); err != nil {
		return err
	}
	ip.IP = net.IP(raw)
	return nil
}

// NetworkEvent uses the custom IP type.
type NetworkEvent struct {
	SourceIP IPAddr `msgpack:"source_ip"`
	DestIP   IPAddr `msgpack:"dest_ip"`
	Port     uint16 `msgpack:"port"`
	Protocol string `msgpack:"protocol"`
}

func main() {
	event := NetworkEvent{
		SourceIP: IPAddr{net.ParseIP("192.168.1.100").To4()},
		DestIP:   IPAddr{net.ParseIP("10.0.0.1").To4()},
		Port:     443,
		Protocol: "TCP",
	}

	data, err := msgpack.Marshal(event)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Encoded size: %d bytes\n", len(data))

	var decoded NetworkEvent
	if err := msgpack.Unmarshal(data, &decoded); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Source IP: %s\n", decoded.SourceIP.IP)
	fmt.Printf("Dest IP:   %s\n", decoded.DestIP.IP)
	fmt.Printf("Port:      %d\n", decoded.Port)
	fmt.Printf("Protocol:  %s\n", decoded.Protocol)
}
```

---

## 6. Avro

### Schema-Based Encoding

Apache Avro is the encoding format that Kleppmann discusses most favorably in DDIA. Avro's key insight is that the schema is not embedded in the encoded data -- instead, the writer's schema and reader's schema are compared at decode time. This means:

1. Encoded data is extremely compact (no field names or field numbers in the payload)
2. Schema evolution is handled by schema resolution rules
3. The schema is stored separately (in a schema registry, file header, etc.)

This makes Avro ideal for storing large amounts of data (log files, data warehouses) where the overhead of per-record field names or even field numbers matters at scale.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log"

	"github.com/linkedin/goavro/v2"
)

func main() {
	// === Define the Avro schema ===
	// Avro schemas are defined in JSON. This is the writer's schema.
	writerSchemaJSON := `{
		"type": "record",
		"name": "User",
		"namespace": "com.example",
		"fields": [
			{"name": "id", "type": "long"},
			{"name": "name", "type": "string"},
			{"name": "email", "type": "string"},
			{"name": "age", "type": "int"},
			{"name": "active", "type": "boolean", "default": true}
		]
	}`

	// Create a codec from the schema
	codec, err := goavro.NewCodec(writerSchemaJSON)
	if err != nil {
		log.Fatalf("Failed to create codec: %v", err)
	}

	// === Encode data ===
	// Avro uses generic maps (not generated structs) in goavro.
	// For type-safe generated code, use github.com/hamba/avro.
	user := map[string]interface{}{
		"id":     int64(42),
		"name":   "Alice Johnson",
		"email":  "alice@example.com",
		"age":    30,
		"active": true,
	}

	// Binary encode (no field names in output -- just raw values)
	binaryData, err := codec.BinaryFromNative(nil, user)
	if err != nil {
		log.Fatalf("Failed to encode: %v", err)
	}

	// JSON encode (for comparison)
	jsonData, _ := json.Marshal(user)

	fmt.Printf("=== Avro Encoding ===\n")
	fmt.Printf("JSON size:       %d bytes\n", len(jsonData))
	fmt.Printf("Avro binary size: %d bytes\n", len(binaryData))
	fmt.Printf("Savings:          %.1f%%\n",
		(1.0-float64(len(binaryData))/float64(len(jsonData)))*100)

	// === Decode data ===
	decoded, remaining, err := codec.NativeFromBinary(binaryData)
	if err != nil {
		log.Fatalf("Failed to decode: %v", err)
	}
	fmt.Printf("Decoded: %v\n", decoded)
	fmt.Printf("Remaining bytes: %d\n", len(remaining))

	// === Schema Evolution ===
	// This is Avro's killer feature. The reader's schema can differ from
	// the writer's schema, and Avro resolves the difference automatically.

	// Reader's schema adds a "phone" field with a default and removes "active"
	readerSchemaJSON := `{
		"type": "record",
		"name": "User",
		"namespace": "com.example",
		"fields": [
			{"name": "id", "type": "long"},
			{"name": "name", "type": "string"},
			{"name": "email", "type": "string"},
			{"name": "age", "type": "int"},
			{"name": "phone", "type": ["null", "string"], "default": null}
		]
	}`

	// In Avro, you need both schemas for resolution
	readerCodec, err := goavro.NewCodecForStandardJSONFull(readerSchemaJSON)
	if err != nil {
		log.Fatalf("Failed to create reader codec: %v", err)
	}

	fmt.Println("\n=== Schema Evolution ===")
	fmt.Println("Writer schema fields: id, name, email, age, active")
	fmt.Println("Reader schema fields: id, name, email, age, phone (new with default)")
	fmt.Println("Resolution: 'active' is ignored, 'phone' gets default value null")
	fmt.Printf("Reader codec created successfully: %v\n", readerCodec != nil)

	// === Object Container Files (OCF) ===
	// Avro files store the schema in the file header, so any reader can
	// decode the file without knowing the schema in advance.
	fmt.Println("\n=== Object Container File (OCF) ===")

	var buf bytes.Buffer
	ocfWriter, err := goavro.NewOCFWriter(goavro.OCFConfig{
		W:     &buf,
		Codec: codec,
	})
	if err != nil {
		log.Fatalf("Failed to create OCF writer: %v", err)
	}

	// Write multiple records
	users := []map[string]interface{}{
		{"id": int64(1), "name": "Alice", "email": "alice@a.com", "age": 30, "active": true},
		{"id": int64(2), "name": "Bob", "email": "bob@b.com", "age": 25, "active": false},
		{"id": int64(3), "name": "Charlie", "email": "charlie@c.com", "age": 35, "active": true},
	}

	if err := ocfWriter.Append([]interface{}{users[0], users[1], users[2]}); err != nil {
		log.Fatalf("Failed to write records: %v", err)
	}

	fmt.Printf("OCF file size: %d bytes (includes schema in header)\n", buf.Len())

	// Read the OCF file -- the reader discovers the schema from the file header
	ocfReader, err := goavro.NewOCFReader(bytes.NewReader(buf.Bytes()))
	if err != nil {
		log.Fatalf("Failed to create OCF reader: %v", err)
	}

	fmt.Println("Records from OCF:")
	for ocfReader.Scan() {
		record, err := ocfReader.Read()
		if err != nil {
			log.Fatal(err)
		}
		if m, ok := record.(map[string]interface{}); ok {
			fmt.Printf("  id=%v name=%v email=%v\n", m["id"], m["name"], m["email"])
		}
	}

	// === Avro vs Protobuf (DDIA Perspective) ===
	fmt.Println("\n=== Avro vs Protobuf (DDIA Perspective) ===")
	fmt.Println()
	fmt.Println("Protobuf:")
	fmt.Println("  - Uses field numbers -> schema evolution via number stability")
	fmt.Println("  - Each field has a tag byte(s) in the encoding")
	fmt.Println("  - Generated code is type-safe and fast")
	fmt.Println("  - Better for RPC (gRPC)")
	fmt.Println()
	fmt.Println("Avro:")
	fmt.Println("  - No field markers in encoding -> smallest possible size")
	fmt.Println("  - Schema resolution happens at read time")
	fmt.Println("  - Schema stored separately (file header or registry)")
	fmt.Println("  - Better for bulk data storage (Hadoop, data pipelines)")
	fmt.Println("  - Dynamically typed (can process data without code generation)")
}
```

---

## 7. Binary Encoding Formats Comparison

### Side-by-Side Analysis

```go
package main

import (
	"encoding/json"
	"fmt"
)

// This section compares encoding formats discussed in DDIA Chapter 4.
// Kleppmann examines the tradeoffs between human-readability, compactness,
// schema support, and evolution capabilities.

type ComparisonRow struct {
	Format         string
	SchemaRequired bool
	HumanReadable  bool
	FieldNames     string // "in-data", "field-numbers", "none"
	SchemaEvolution string
	TypicalUse     string
}

func main() {
	comparisons := []ComparisonRow{
		{
			Format:         "JSON",
			SchemaRequired: false,
			HumanReadable:  true,
			FieldNames:     "in every record",
			SchemaEvolution: "ad-hoc (no guarantees)",
			TypicalUse:     "REST APIs, config files, browser clients",
		},
		{
			Format:         "MessagePack",
			SchemaRequired: false,
			HumanReadable:  false,
			FieldNames:     "in every record (binary)",
			SchemaEvolution: "ad-hoc (same as JSON)",
			TypicalUse:     "cache storage, internal APIs needing JSON compat",
		},
		{
			Format:         "Protocol Buffers",
			SchemaRequired: true,
			HumanReadable:  false,
			FieldNames:     "field numbers only",
			SchemaEvolution: "excellent (field numbers are stable)",
			TypicalUse:     "gRPC services, inter-service communication",
		},
		{
			Format:         "Avro",
			SchemaRequired: true,
			HumanReadable:  false,
			FieldNames:     "none (position-based)",
			SchemaEvolution: "excellent (schema resolution)",
			TypicalUse:     "data pipelines, Hadoop, Kafka, bulk storage",
		},
		{
			Format:         "Thrift (Binary)",
			SchemaRequired: true,
			HumanReadable:  false,
			FieldNames:     "field numbers only",
			SchemaEvolution: "good (similar to protobuf)",
			TypicalUse:     "Facebook services (less common in new projects)",
		},
		{
			Format:         "FlatBuffers",
			SchemaRequired: true,
			HumanReadable:  false,
			FieldNames:     "field offsets",
			SchemaEvolution: "good",
			TypicalUse:     "games, embedded systems (zero-copy access)",
		},
		{
			Format:         "Custom Binary",
			SchemaRequired: false,
			HumanReadable:  false,
			FieldNames:     "implicit (hardcoded)",
			SchemaEvolution: "manual (you implement it)",
			TypicalUse:     "wire protocols, high-frequency trading, IoT",
		},
	}

	fmt.Println("=== Binary Encoding Formats Comparison (DDIA Chapter 4) ===")
	fmt.Println()

	// Print as a readable comparison
	for _, c := range comparisons {
		fmt.Printf("--- %s ---\n", c.Format)
		fmt.Printf("  Schema required:   %v\n", c.SchemaRequired)
		fmt.Printf("  Human readable:    %v\n", c.HumanReadable)
		fmt.Printf("  Field names:       %s\n", c.FieldNames)
		fmt.Printf("  Schema evolution:  %s\n", c.SchemaEvolution)
		fmt.Printf("  Typical use:       %s\n", c.TypicalUse)
		fmt.Println()
	}

	// Approximate encoding sizes for the same data structure
	// Based on typical measurements from DDIA and practical benchmarks.
	type SizeEstimate struct {
		Format string
		Bytes  int
		Notes  string
	}

	record := map[string]interface{}{
		"id":         42,
		"first_name": "Martin",
		"last_name":  "Kleppmann",
		"email":      "martin@example.com",
		"age":        38,
		"active":     true,
		"tags":       []string{"author", "researcher", "speaker"},
	}

	jsonBytes, _ := json.Marshal(record)
	jsonIndented, _ := json.MarshalIndent(record, "", "  ")

	estimates := []SizeEstimate{
		{Format: "JSON (compact)", Bytes: len(jsonBytes), Notes: "field names in every record"},
		{Format: "JSON (pretty)", Bytes: len(jsonIndented), Notes: "with whitespace formatting"},
		{Format: "MessagePack", Bytes: int(float64(len(jsonBytes)) * 0.72), Notes: "~28% smaller than JSON"},
		{Format: "Protobuf", Bytes: int(float64(len(jsonBytes)) * 0.45), Notes: "~55% smaller than JSON"},
		{Format: "Avro", Bytes: int(float64(len(jsonBytes)) * 0.38), Notes: "~62% smaller than JSON"},
	}

	fmt.Println("=== Approximate Encoding Sizes ===")
	fmt.Printf("Record: %s\n\n", string(jsonBytes))
	for _, e := range estimates {
		bar := ""
		for i := 0; i < e.Bytes/3; i++ {
			bar += "█"
		}
		fmt.Printf("%-18s %3d bytes  %s  (%s)\n", e.Format, e.Bytes, bar, e.Notes)
	}
}
```

---

## 8. Schema Evolution Strategies

### The Core Challenge (DDIA Perspective)

Kleppmann identifies schema evolution as one of the most important challenges in data-intensive applications. Data outlives code. A schema change you make today must not break data written yesterday, and data you write today must not break code deployed tomorrow.

```go
package main

import (
	"encoding/json"
	"fmt"
)

// This example catalogs all possible schema changes and their safety.
// Each change is rated as SAFE, CAREFUL, or BREAKING.

func main() {
	fmt.Println("=== Schema Evolution Safety Guide ===")
	fmt.Println("Based on DDIA Chapter 4 by Martin Kleppmann")
	fmt.Println()

	type Change struct {
		Operation   string
		Safety      string
		Explanation string
		JSONSafe    bool
		ProtobufSafe bool
		AvroSafe     bool
	}

	changes := []Change{
		{
			Operation:   "Add an optional field",
			Safety:      "SAFE",
			Explanation: "Old code ignores unknown fields (forward compat). New code uses defaults for missing fields (backward compat).",
			JSONSafe:    true,
			ProtobufSafe: true,
			AvroSafe:     true,
		},
		{
			Operation:   "Add a required field",
			Safety:      "BREAKING (backward)",
			Explanation: "Old data does not have this field, so new code that requires it will fail. Proto3 does not have required fields for this reason.",
			JSONSafe:    false,
			ProtobufSafe: false,
			AvroSafe:     false,
		},
		{
			Operation:   "Remove an optional field",
			Safety:      "SAFE (if field number/name never reused)",
			Explanation: "New code stops sending it. Old code uses default. CRITICAL: never reuse the field identifier.",
			JSONSafe:    true,
			ProtobufSafe: true,
			AvroSafe:     true,
		},
		{
			Operation:   "Rename a field",
			Safety:      "FORMAT-DEPENDENT",
			Explanation: "JSON: BREAKING (field names are on the wire). Protobuf: SAFE (field numbers are on the wire). Avro: SAFE (use aliases).",
			JSONSafe:    false,
			ProtobufSafe: true,
			AvroSafe:     true,
		},
		{
			Operation:   "Change field type: int32 -> int64",
			Safety:      "SAFE (widening)",
			Explanation: "Values are widened without data loss. New code reads old small values. Old code may truncate new large values.",
			JSONSafe:    true,
			ProtobufSafe: true,
			AvroSafe:     true,
		},
		{
			Operation:   "Change field type: int -> string",
			Safety:      "BREAKING",
			Explanation: "Completely different wire encoding. Old data cannot be parsed by new code without a migration.",
			JSONSafe:    false,
			ProtobufSafe: false,
			AvroSafe:     false,
		},
		{
			Operation:   "Change optional to repeated (single -> list)",
			Safety:      "FORMAT-DEPENDENT",
			Explanation: "Protobuf: SAFE for packed types (singular becomes 1-element list). JSON: BREAKING (scalar vs array). Avro: BREAKING.",
			JSONSafe:    false,
			ProtobufSafe: true,
			AvroSafe:     false,
		},
		{
			Operation:   "Reorder fields",
			Safety:      "FORMAT-DEPENDENT",
			Explanation: "JSON: SAFE (objects are unordered). Protobuf: SAFE (field numbers determine position). Avro: BREAKING (position-based).",
			JSONSafe:    true,
			ProtobufSafe: true,
			AvroSafe:     false,
		},
		{
			Operation:   "Add enum value",
			Safety:      "CAREFUL",
			Explanation: "Old code receives unknown enum value. Protobuf: old code sees 0 (default). Old switch statements may hit default case.",
			JSONSafe:    true,
			ProtobufSafe: true,
			AvroSafe:     true,
		},
		{
			Operation:   "Change field from singular to oneof",
			Safety:      "CAREFUL",
			Explanation: "Protobuf: wire-compatible if the original field is the first oneof member. Requires careful migration.",
			JSONSafe:    false,
			ProtobufSafe: true,
			AvroSafe:     false,
		},
	}

	for _, c := range changes {
		fmt.Printf("%-50s [%s]\n", c.Operation, c.Safety)
		fmt.Printf("  %s\n", c.Explanation)
		fmt.Printf("  JSON: %v | Protobuf: %v | Avro: %v\n\n",
			safetyIcon(c.JSONSafe), safetyIcon(c.ProtobufSafe), safetyIcon(c.AvroSafe))
	}

	// === Practical Example: Safe JSON Evolution in Go ===
	fmt.Println("=== Practical Example: Safe JSON Evolution ===")
	demonstrateJSONEvolution()
}

func safetyIcon(safe bool) string {
	if safe {
		return "SAFE"
	}
	return "RISKY"
}

// demonstrateJSONEvolution shows how to safely evolve a JSON schema in Go.
func demonstrateJSONEvolution() {
	// Strategy: Use pointer fields + omitempty for all new fields.
	// This ensures backward and forward compatibility.

	// Version 1: Original schema
	type OrderV1 struct {
		ID     string  `json:"id"`
		Amount float64 `json:"amount"`
		Status string  `json:"status"`
	}

	// Version 2: Added optional fields, kept all V1 fields
	type OrderV2 struct {
		ID       string   `json:"id"`
		Amount   float64  `json:"amount"`
		Status   string   `json:"status"`
		Currency *string  `json:"currency,omitempty"` // New in V2, pointer + omitempty
		Items    []string `json:"items,omitempty"`    // New in V2, slice + omitempty
		Note     *string  `json:"note,omitempty"`     // New in V2, nullable
	}

	// Version 3: Added more fields, deprecated none (just stop using them)
	type OrderV3 struct {
		ID        string   `json:"id"`
		Amount    float64  `json:"amount"`
		Status    string   `json:"status"`
		Currency  *string  `json:"currency,omitempty"`
		Items     []string `json:"items,omitempty"`
		Note      *string  `json:"note,omitempty"`
		Priority  *int     `json:"priority,omitempty"`  // New in V3
		Region    *string  `json:"region,omitempty"`    // New in V3
	}

	// V1 writes data
	v1Data, _ := json.Marshal(OrderV1{
		ID: "ORD-001", Amount: 99.99, Status: "pending",
	})
	fmt.Printf("V1 data: %s\n", v1Data)

	// V3 reads V1 data (backward compatibility)
	var v3FromV1 OrderV3
	json.Unmarshal(v1Data, &v3FromV1)
	fmt.Printf("V3 reads V1: ID=%s, Currency=%v, Priority=%v\n",
		v3FromV1.ID,
		v3FromV1.Currency,  // nil
		v3FromV1.Priority,  // nil
	)

	// V3 writes data
	currency := "USD"
	priority := 1
	region := "us-west-2"
	v3Data, _ := json.Marshal(OrderV3{
		ID: "ORD-002", Amount: 149.99, Status: "confirmed",
		Currency: &currency, Priority: &priority, Region: &region,
		Items: []string{"shoes", "socks"},
	})
	fmt.Printf("\nV3 data: %s\n", v3Data)

	// V1 reads V3 data (forward compatibility)
	var v1FromV3 OrderV1
	json.Unmarshal(v3Data, &v1FromV3)
	fmt.Printf("V1 reads V3: ID=%s, Amount=%.2f, Status=%s\n",
		v1FromV3.ID, v1FromV3.Amount, v1FromV3.Status)
	fmt.Println("(Currency, Priority, Region, Items all silently ignored)")
}
```

---

## 9. Encoding for Storage vs Network

### Different Contexts, Different Choices

Kleppmann describes three main dataflow patterns in DDIA Chapter 4:
1. **Through databases** (process writes encoded data, another process reads it later)
2. **Through service calls** (REST, RPC -- one process sends a request, another responds)
3. **Through message passing** (async message queues like Kafka, RabbitMQ)

Each pattern has different requirements for encoding.

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	fmt.Println("=== Encoding Format Selection Guide ===")
	fmt.Println("Based on DDIA Chapter 4 Dataflow Patterns")
	fmt.Println()

	type Scenario struct {
		Context       string
		Recommended   string
		Rationale     string
	}

	scenarios := []Scenario{
		// Storage scenarios
		{
			Context:     "Database storage (PostgreSQL JSONB column)",
			Recommended: "JSON",
			Rationale:   "PostgreSQL natively supports JSONB with indexing. Schema evolution via SQL migrations.",
		},
		{
			Context:     "Cache storage (Redis)",
			Recommended: "MessagePack or Protobuf",
			Rationale:   "Binary is faster to encode/decode on every cache hit. Space savings reduce memory costs.",
		},
		{
			Context:     "Data pipeline / Kafka messages",
			Recommended: "Avro with Schema Registry",
			Rationale:   "Compact encoding at scale. Schema registry ensures all producers and consumers agree on schema. Schema evolution is managed centrally.",
		},
		{
			Context:     "Log files / append-only storage",
			Recommended: "JSON Lines (NDJSON) or Avro OCF",
			Rationale:   "JSON Lines for debugging/grep. Avro OCF for production (compact, schema in header).",
		},
		{
			Context:     "Configuration files",
			Recommended: "JSON, YAML, or TOML",
			Rationale:   "Human readability is paramount. Engineers edit these files by hand.",
		},
		{
			Context:     "File archival / data warehouse",
			Recommended: "Avro or Parquet",
			Rationale:   "Schema stored with data. Parquet for columnar analytics. Avro for row-based processing.",
		},

		// Network scenarios
		{
			Context:     "Public REST API",
			Recommended: "JSON",
			Rationale:   "Universal client support. Browsers can consume it natively. Easy to debug with curl.",
		},
		{
			Context:     "Internal microservice communication",
			Recommended: "Protobuf with gRPC",
			Rationale:   "Type-safe generated clients. Compact binary encoding. Streaming support. Schema evolution via .proto files.",
		},
		{
			Context:     "Real-time bidirectional (WebSocket)",
			Recommended: "MessagePack or Protobuf",
			Rationale:   "Binary encoding reduces bandwidth. MessagePack if schema-less flexibility needed.",
		},
		{
			Context:     "Mobile client API",
			Recommended: "Protobuf or JSON",
			Rationale:   "Protobuf saves bandwidth (expensive on mobile). JSON for simpler implementation.",
		},
		{
			Context:     "IoT / embedded devices",
			Recommended: "Custom binary or Protobuf",
			Rationale:   "Minimal overhead. Every byte counts on constrained devices and networks.",
		},

		// Message passing scenarios
		{
			Context:     "Event sourcing / CQRS events",
			Recommended: "Protobuf or Avro with schema registry",
			Rationale:   "Events are immutable and long-lived. Schema evolution is critical -- you will read events written years ago.",
		},
		{
			Context:     "Job queue payloads",
			Recommended: "JSON or MessagePack",
			Rationale:   "JSON for easy debugging. MessagePack if performance matters. Short-lived, so schema evolution less critical.",
		},
	}

	for _, s := range scenarios {
		fmt.Printf("Context:     %s\n", s.Context)
		fmt.Printf("Recommended: %s\n", s.Recommended)
		fmt.Printf("Rationale:   %s\n\n", s.Rationale)
	}

	// Decision flowchart
	fmt.Println("=== Decision Flowchart ===")
	fmt.Println()
	fmt.Println("1. Does a human need to read/edit the data?")
	fmt.Println("   YES -> JSON (or YAML/TOML for config)")
	fmt.Println("   NO  -> Continue to step 2")
	fmt.Println()
	fmt.Println("2. Do you need a schema contract between producer and consumer?")
	fmt.Println("   YES -> Continue to step 3")
	fmt.Println("   NO  -> MessagePack (binary JSON)")
	fmt.Println()
	fmt.Println("3. Is the data stored long-term (months/years)?")
	fmt.Println("   YES -> Avro (schema stored with data, excellent evolution)")
	fmt.Println("   NO  -> Continue to step 4")
	fmt.Println()
	fmt.Println("4. Is this for RPC / service communication?")
	fmt.Println("   YES -> Protobuf + gRPC")
	fmt.Println("   NO  -> Protobuf (for messages) or Avro (for bulk data)")

	// Also show as a compact table
	fmt.Println("\n=== Quick Reference Table ===")
	type QuickRef struct {
		Format     string
		Size       string
		Speed      string
		Schema     string
		Evolution  string
		ReadableBy string
	}

	refs := []QuickRef{
		{"JSON", "Large", "Slow", "None", "Ad-hoc", "Humans + Machines"},
		{"MsgPack", "Medium", "Fast", "None", "Ad-hoc", "Machines"},
		{"Protobuf", "Small", "Very fast", ".proto files", "Excellent", "Machines (with protoc)"},
		{"Avro", "Smallest", "Fast", "JSON schema", "Excellent", "Machines (with schema)"},
		{"Custom", "Minimal", "Fastest", "Your code", "Manual", "Your code only"},
	}

	jsonOut, _ := json.MarshalIndent(refs, "", "  ")
	fmt.Println(string(jsonOut))
}
```

---

## 10. Building a Schema Registry in Go

### Why a Schema Registry?

As Kleppmann discusses in DDIA, Avro and similar formats separate the schema from the data. But then you need a way to manage schemas -- versioning them, validating compatibility, and making them available to all producers and consumers. This is what a schema registry does. Confluent Schema Registry is the most well-known implementation (for Kafka + Avro), but the concept applies to any schema-based format.

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"
)

// SchemaRegistry manages versioned schemas for different subjects.
// A "subject" is typically a topic name or data type name.
// Each subject can have multiple schema versions, and the registry
// enforces compatibility rules between versions.
type SchemaRegistry struct {
	mu       sync.RWMutex
	subjects map[string]*Subject
}

// Subject holds all versions of a schema for a particular data type.
type Subject struct {
	Name     string          `json:"name"`
	Versions []SchemaVersion `json:"versions"`
	// Compatibility mode determines what changes are allowed.
	// BACKWARD: new schema can read data written by old schema.
	// FORWARD: old schema can read data written by new schema.
	// FULL: both backward and forward compatible.
	// NONE: no compatibility checking.
	Compatibility CompatibilityMode `json:"compatibility"`
}

type SchemaVersion struct {
	Version   int       `json:"version"`
	Schema    string    `json:"schema"`      // The schema definition (JSON string)
	ID        int       `json:"id"`          // Global unique schema ID
	CreatedAt time.Time `json:"created_at"`
	Checksum  string    `json:"checksum"`    // SHA-256 of the schema
}

type CompatibilityMode string

const (
	CompatNone     CompatibilityMode = "NONE"
	CompatBackward CompatibilityMode = "BACKWARD"
	CompatForward  CompatibilityMode = "FORWARD"
	CompatFull     CompatibilityMode = "FULL"
)

func NewSchemaRegistry() *SchemaRegistry {
	return &SchemaRegistry{
		subjects: make(map[string]*Subject),
	}
}

// RegisterSchema registers a new version of a schema for a subject.
// It validates compatibility with the latest version if a compatibility mode is set.
func (r *SchemaRegistry) RegisterSchema(subjectName, schema string) (*SchemaVersion, error) {
	r.mu.Lock()
	defer r.mu.Unlock()

	subject, exists := r.subjects[subjectName]
	if !exists {
		subject = &Subject{
			Name:          subjectName,
			Versions:      nil,
			Compatibility: CompatBackward, // Default to backward compatible
		}
		r.subjects[subjectName] = subject
	}

	// Check if this exact schema already exists
	for _, v := range subject.Versions {
		if v.Schema == schema {
			return &v, nil // Idempotent: return existing version
		}
	}

	// Validate compatibility with the latest version
	if len(subject.Versions) > 0 && subject.Compatibility != CompatNone {
		latest := subject.Versions[len(subject.Versions)-1]
		if err := checkCompatibility(latest.Schema, schema, subject.Compatibility); err != nil {
			return nil, fmt.Errorf("schema is not %s compatible: %w",
				subject.Compatibility, err)
		}
	}

	// Assign a new version
	version := SchemaVersion{
		Version:   len(subject.Versions) + 1,
		Schema:    schema,
		ID:        len(subject.Versions) + 1, // Simplified; real registries use global IDs
		CreatedAt: time.Now(),
		Checksum:  fmt.Sprintf("%x", len(schema)), // Simplified; use crypto/sha256 in production
	}
	subject.Versions = append(subject.Versions, version)

	return &version, nil
}

// GetSchema retrieves a specific version of a schema.
func (r *SchemaRegistry) GetSchema(subjectName string, version int) (*SchemaVersion, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	subject, exists := r.subjects[subjectName]
	if !exists {
		return nil, fmt.Errorf("subject %q not found", subjectName)
	}

	if version < 1 || version > len(subject.Versions) {
		return nil, fmt.Errorf("version %d not found for subject %q", version, subjectName)
	}

	sv := subject.Versions[version-1]
	return &sv, nil
}

// GetLatestSchema retrieves the latest version.
func (r *SchemaRegistry) GetLatestSchema(subjectName string) (*SchemaVersion, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	subject, exists := r.subjects[subjectName]
	if !exists {
		return nil, fmt.Errorf("subject %q not found", subjectName)
	}

	if len(subject.Versions) == 0 {
		return nil, fmt.Errorf("no versions for subject %q", subjectName)
	}

	sv := subject.Versions[len(subject.Versions)-1]
	return &sv, nil
}

// ListSubjects returns all registered subjects.
func (r *SchemaRegistry) ListSubjects() []string {
	r.mu.RLock()
	defer r.mu.RUnlock()

	subjects := make([]string, 0, len(r.subjects))
	for name := range r.subjects {
		subjects = append(subjects, name)
	}
	return subjects
}

// checkCompatibility validates that newSchema is compatible with oldSchema.
// In a real implementation, this would parse the schemas and compare fields.
// Here we demonstrate the concept with JSON schemas.
func checkCompatibility(oldSchemaStr, newSchemaStr string, mode CompatibilityMode) error {
	var oldSchema, newSchema map[string]interface{}
	if err := json.Unmarshal([]byte(oldSchemaStr), &oldSchema); err != nil {
		return fmt.Errorf("invalid old schema: %w", err)
	}
	if err := json.Unmarshal([]byte(newSchemaStr), &newSchema); err != nil {
		return fmt.Errorf("invalid new schema: %w", err)
	}

	oldFields := extractFields(oldSchema)
	newFields := extractFields(newSchema)

	switch mode {
	case CompatBackward:
		// New code must be able to read old data.
		// All old fields must still exist in the new schema (or have defaults).
		for field := range oldFields {
			if _, exists := newFields[field]; !exists {
				// Field was removed. Check if it had a default in old schema.
				// Simplified check -- real implementation would be more thorough.
				return fmt.Errorf("field %q removed without default (breaks backward compat)", field)
			}
		}

	case CompatForward:
		// Old code must be able to read new data.
		// New fields must have defaults (so old code can ignore them).
		for field := range newFields {
			if _, exists := oldFields[field]; !exists {
				// New field added -- this is fine for forward compat
				// as long as it has a default (old code ignores it).
				// In a real check, verify the default exists.
			}
		}

	case CompatFull:
		// Both directions. Most restrictive.
		for field := range oldFields {
			if _, exists := newFields[field]; !exists {
				return fmt.Errorf("field %q removed (breaks full compat)", field)
			}
		}
	}

	return nil
}

func extractFields(schema map[string]interface{}) map[string]bool {
	fields := make(map[string]bool)
	if fieldList, ok := schema["fields"].([]interface{}); ok {
		for _, f := range fieldList {
			if fm, ok := f.(map[string]interface{}); ok {
				if name, ok := fm["name"].(string); ok {
					fields[name] = true
				}
			}
		}
	}
	return fields
}

// ServeHTTP sets up an HTTP API for the schema registry.
func (r *SchemaRegistry) SetupRoutes(mux *http.ServeMux) {
	// POST /subjects/{subject}/versions -- register new schema
	// GET /subjects/{subject}/versions/{version} -- get specific version
	// GET /subjects/{subject}/versions/latest -- get latest
	// GET /subjects -- list all subjects

	mux.HandleFunc("/subjects", func(w http.ResponseWriter, req *http.Request) {
		subjects := r.ListSubjects()
		json.NewEncoder(w).Encode(subjects)
	})
}

func main() {
	registry := NewSchemaRegistry()

	// Register initial schema for "user-events" subject
	schemaV1 := `{
		"type": "record",
		"name": "UserEvent",
		"fields": [
			{"name": "id", "type": "long"},
			{"name": "name", "type": "string"},
			{"name": "email", "type": "string"}
		]
	}`

	v1, err := registry.RegisterSchema("user-events", schemaV1)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Registered schema v%d for 'user-events'\n", v1.Version)

	// Register V2: add a field (backward compatible)
	schemaV2 := `{
		"type": "record",
		"name": "UserEvent",
		"fields": [
			{"name": "id", "type": "long"},
			{"name": "name", "type": "string"},
			{"name": "email", "type": "string"},
			{"name": "age", "type": "int", "default": 0}
		]
	}`

	v2, err := registry.RegisterSchema("user-events", schemaV2)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Registered schema v%d for 'user-events'\n", v2.Version)

	// Try to register V3: remove a field (NOT backward compatible)
	schemaV3Breaking := `{
		"type": "record",
		"name": "UserEvent",
		"fields": [
			{"name": "id", "type": "long"},
			{"name": "name", "type": "string"},
			{"name": "age", "type": "int", "default": 0}
		]
	}`

	_, err = registry.RegisterSchema("user-events", schemaV3Breaking)
	if err != nil {
		fmt.Printf("Schema V3 rejected: %v\n", err)
	}

	// Register for a different subject
	orderSchema := `{
		"type": "record",
		"name": "OrderEvent",
		"fields": [
			{"name": "order_id", "type": "string"},
			{"name": "amount", "type": "double"},
			{"name": "status", "type": "string"}
		]
	}`

	ov1, err := registry.RegisterSchema("order-events", orderSchema)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Registered schema v%d for 'order-events'\n", ov1.Version)

	// List all subjects
	fmt.Printf("\nAll subjects: %v\n", registry.ListSubjects())

	// Retrieve a specific version
	retrieved, err := registry.GetSchema("user-events", 1)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\nRetrieved user-events v%d (created %s):\n%s\n",
		retrieved.Version, retrieved.CreatedAt.Format(time.RFC3339), retrieved.Schema)

	// Latest version
	latest, err := registry.GetLatestSchema("user-events")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\nLatest user-events: v%d\n", latest.Version)

	// === How this fits with DDIA ===
	fmt.Println("\n=== Schema Registry in DDIA Context ===")
	fmt.Println()
	fmt.Println("The schema registry implements what Kleppmann calls the")
	fmt.Println("'schema evolution' infrastructure. In practice:")
	fmt.Println()
	fmt.Println("1. Producers register their schema before sending messages.")
	fmt.Println("2. Each message header includes the schema ID (4 bytes).")
	fmt.Println("3. Consumers fetch the writer's schema from the registry.")
	fmt.Println("4. The consumer uses both the writer's schema and its own")
	fmt.Println("   reader's schema to decode the message (Avro resolution).")
	fmt.Println("5. The registry enforces compatibility rules, preventing")
	fmt.Println("   breaking changes from being deployed.")
}
```

---

## 11. Custom Binary Protocols

### When Standard Formats Are Not Enough

Sometimes you need a custom binary protocol -- for wire protocols (databases, proxies), high-frequency trading, IoT, or any situation where you need absolute control over every byte. Go's `encoding/binary` package provides the tools.

### Length-Prefixed Framing

The most common custom protocol pattern is length-prefixed framing: each message is preceded by its length. This solves the "where does one message end and the next begin?" problem in TCP streams.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"hash/crc32"
	"io"
	"log"
	"time"
)

// Wire Protocol Format:
// ┌──────────────┬──────────────┬───────────┬─────────────┬──────────┐
// │ Length (4B)   │ Type (1B)    │ Flags (1B)│ Payload     │ CRC (4B) │
// │ uint32 BE    │ uint8        │ uint8     │ variable    │ uint32 BE│
// └──────────────┴──────────────┴───────────┴─────────────┴──────────┘
//
// Length: total size of Type + Flags + Payload + CRC (does NOT include Length itself)
// Type: message type identifier
// Flags: bitfield for options (compressed, encrypted, etc.)
// Payload: the actual message data
// CRC: CRC32 checksum of Type + Flags + Payload

// MessageType represents the type of wire message.
type MessageType uint8

const (
	MsgTypePing     MessageType = 0x01
	MsgTypePong     MessageType = 0x02
	MsgTypeData     MessageType = 0x03
	MsgTypeAck      MessageType = 0x04
	MsgTypeError    MessageType = 0x05
	MsgTypeShutdown MessageType = 0x06
)

func (mt MessageType) String() string {
	switch mt {
	case MsgTypePing:
		return "PING"
	case MsgTypePong:
		return "PONG"
	case MsgTypeData:
		return "DATA"
	case MsgTypeAck:
		return "ACK"
	case MsgTypeError:
		return "ERROR"
	case MsgTypeShutdown:
		return "SHUTDOWN"
	default:
		return fmt.Sprintf("UNKNOWN(%d)", mt)
	}
}

// Flags for message options
const (
	FlagCompressed uint8 = 1 << 0 // Bit 0: payload is compressed
	FlagEncrypted  uint8 = 1 << 1 // Bit 1: payload is encrypted
	FlagPriority   uint8 = 1 << 2 // Bit 2: high priority message
)

// WireMessage represents a single message in our wire protocol.
type WireMessage struct {
	Type    MessageType
	Flags   uint8
	Payload []byte
}

// WireEncoder writes messages to an io.Writer in our wire format.
type WireEncoder struct {
	w io.Writer
}

func NewWireEncoder(w io.Writer) *WireEncoder {
	return &WireEncoder{w: w}
}

// Encode writes a message in our wire format.
func (e *WireEncoder) Encode(msg *WireMessage) error {
	// Calculate the body: Type (1) + Flags (1) + Payload (N) + CRC (4)
	bodyLen := uint32(1 + 1 + len(msg.Payload) + 4)

	// Write Length (4 bytes, big-endian)
	if err := binary.Write(e.w, binary.BigEndian, bodyLen); err != nil {
		return fmt.Errorf("write length: %w", err)
	}

	// Build the checksummed portion: Type + Flags + Payload
	checksumData := make([]byte, 0, 2+len(msg.Payload))
	checksumData = append(checksumData, byte(msg.Type))
	checksumData = append(checksumData, msg.Flags)
	checksumData = append(checksumData, msg.Payload...)

	// Write Type + Flags + Payload
	if _, err := e.w.Write(checksumData); err != nil {
		return fmt.Errorf("write body: %w", err)
	}

	// Write CRC32 checksum (4 bytes, big-endian)
	checksum := crc32.ChecksumIEEE(checksumData)
	if err := binary.Write(e.w, binary.BigEndian, checksum); err != nil {
		return fmt.Errorf("write checksum: %w", err)
	}

	return nil
}

// WireDecoder reads messages from an io.Reader.
type WireDecoder struct {
	r io.Reader
}

func NewWireDecoder(r io.Reader) *WireDecoder {
	return &WireDecoder{r: r}
}

// Decode reads one message from the wire.
func (d *WireDecoder) Decode() (*WireMessage, error) {
	// Read Length (4 bytes)
	var bodyLen uint32
	if err := binary.Read(d.r, binary.BigEndian, &bodyLen); err != nil {
		return nil, fmt.Errorf("read length: %w", err)
	}

	// Sanity check: reject messages larger than 16MB
	const maxMessageSize = 16 * 1024 * 1024
	if bodyLen > maxMessageSize {
		return nil, fmt.Errorf("message too large: %d bytes (max %d)", bodyLen, maxMessageSize)
	}

	// Read the entire body
	body := make([]byte, bodyLen)
	if _, err := io.ReadFull(d.r, body); err != nil {
		return nil, fmt.Errorf("read body: %w", err)
	}

	// Body layout: Type (1) + Flags (1) + Payload (N) + CRC (4)
	if bodyLen < 6 { // Minimum: 1 + 1 + 0 + 4
		return nil, fmt.Errorf("message too short: %d bytes", bodyLen)
	}

	msgType := MessageType(body[0])
	flags := body[1]
	payload := body[2 : bodyLen-4]

	// Verify CRC32
	checksumData := body[:bodyLen-4] // Type + Flags + Payload
	var expectedCRC uint32
	if err := binary.Read(bytes.NewReader(body[bodyLen-4:]), binary.BigEndian, &expectedCRC); err != nil {
		return nil, fmt.Errorf("read checksum: %w", err)
	}
	actualCRC := crc32.ChecksumIEEE(checksumData)
	if actualCRC != expectedCRC {
		return nil, fmt.Errorf("checksum mismatch: expected %08x, got %08x", expectedCRC, actualCRC)
	}

	return &WireMessage{
		Type:    msgType,
		Flags:   flags,
		Payload: payload,
	}, nil
}

// === Structured Payload Encoding ===
// The payload itself needs a format. Here we use a simple TLV (Tag-Length-Value)
// scheme for the DATA message type.

// DataPayload represents structured data within a DATA message.
type DataPayload struct {
	RequestID uint64
	Timestamp int64 // Unix nanoseconds
	Key       string
	Value     []byte
}

func (dp *DataPayload) Encode() ([]byte, error) {
	var buf bytes.Buffer

	// RequestID: 8 bytes
	if err := binary.Write(&buf, binary.BigEndian, dp.RequestID); err != nil {
		return nil, err
	}

	// Timestamp: 8 bytes
	if err := binary.Write(&buf, binary.BigEndian, dp.Timestamp); err != nil {
		return nil, err
	}

	// Key: 2-byte length prefix + string bytes
	keyBytes := []byte(dp.Key)
	if err := binary.Write(&buf, binary.BigEndian, uint16(len(keyBytes))); err != nil {
		return nil, err
	}
	buf.Write(keyBytes)

	// Value: 4-byte length prefix + bytes
	if err := binary.Write(&buf, binary.BigEndian, uint32(len(dp.Value))); err != nil {
		return nil, err
	}
	buf.Write(dp.Value)

	return buf.Bytes(), nil
}

func DecodeDataPayload(data []byte) (*DataPayload, error) {
	r := bytes.NewReader(data)
	dp := &DataPayload{}

	if err := binary.Read(r, binary.BigEndian, &dp.RequestID); err != nil {
		return nil, fmt.Errorf("read request_id: %w", err)
	}
	if err := binary.Read(r, binary.BigEndian, &dp.Timestamp); err != nil {
		return nil, fmt.Errorf("read timestamp: %w", err)
	}

	var keyLen uint16
	if err := binary.Read(r, binary.BigEndian, &keyLen); err != nil {
		return nil, fmt.Errorf("read key length: %w", err)
	}
	keyBuf := make([]byte, keyLen)
	if _, err := io.ReadFull(r, keyBuf); err != nil {
		return nil, fmt.Errorf("read key: %w", err)
	}
	dp.Key = string(keyBuf)

	var valLen uint32
	if err := binary.Read(r, binary.BigEndian, &valLen); err != nil {
		return nil, fmt.Errorf("read value length: %w", err)
	}
	dp.Value = make([]byte, valLen)
	if _, err := io.ReadFull(r, dp.Value); err != nil {
		return nil, fmt.Errorf("read value: %w", err)
	}

	return dp, nil
}

func main() {
	var buf bytes.Buffer
	encoder := NewWireEncoder(&buf)

	// Send a PING
	fmt.Println("=== Wire Protocol Demo ===")
	fmt.Println()

	if err := encoder.Encode(&WireMessage{
		Type:    MsgTypePing,
		Flags:   0,
		Payload: nil,
	}); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Encoded PING: %d bytes\n", buf.Len())

	// Send a DATA message with structured payload
	payload := &DataPayload{
		RequestID: 12345,
		Timestamp: time.Now().UnixNano(),
		Key:       "user:42",
		Value:     []byte(`{"name":"Alice","age":30}`),
	}
	payloadBytes, _ := payload.Encode()

	if err := encoder.Encode(&WireMessage{
		Type:    MsgTypeData,
		Flags:   FlagPriority,
		Payload: payloadBytes,
	}); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Encoded DATA: %d bytes total in buffer\n", buf.Len())

	// Send an ACK
	if err := encoder.Encode(&WireMessage{
		Type:    MsgTypeAck,
		Flags:   0,
		Payload: []byte{0x00, 0x00, 0x30, 0x39}, // ACK request ID 12345 as 4 bytes
	}); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Encoded ACK:  %d bytes total in buffer\n", buf.Len())

	// === Decode all messages ===
	fmt.Println("\n=== Decoding ===")
	decoder := NewWireDecoder(bytes.NewReader(buf.Bytes()))

	for i := 0; i < 3; i++ {
		msg, err := decoder.Decode()
		if err != nil {
			log.Fatal(err)
		}

		fmt.Printf("Message %d: Type=%s  Flags=%08b  Payload=%d bytes\n",
			i+1, msg.Type, msg.Flags, len(msg.Payload))

		// Decode DATA payload
		if msg.Type == MsgTypeData {
			dp, err := DecodeDataPayload(msg.Payload)
			if err != nil {
				log.Fatal(err)
			}
			fmt.Printf("  RequestID: %d\n", dp.RequestID)
			fmt.Printf("  Timestamp: %s\n", time.Unix(0, dp.Timestamp).Format(time.RFC3339Nano))
			fmt.Printf("  Key:       %s\n", dp.Key)
			fmt.Printf("  Value:     %s\n", dp.Value)
			fmt.Printf("  Priority:  %v\n", msg.Flags&FlagPriority != 0)
		}
	}

	// === Wire format visualization ===
	fmt.Println("\n=== Wire Format Breakdown ===")
	fmt.Println()
	rawBytes := buf.Bytes()

	fmt.Println("PING message (raw bytes):")
	if len(rawBytes) >= 10 {
		fmt.Printf("  Length: %02x %02x %02x %02x (= %d)\n",
			rawBytes[0], rawBytes[1], rawBytes[2], rawBytes[3],
			binary.BigEndian.Uint32(rawBytes[0:4]))
		fmt.Printf("  Type:  %02x (= %s)\n", rawBytes[4], MsgTypePing)
		fmt.Printf("  Flags: %02x (= %08b)\n", rawBytes[5], rawBytes[5])
		fmt.Printf("  CRC:   %02x %02x %02x %02x\n",
			rawBytes[6], rawBytes[7], rawBytes[8], rawBytes[9])
	}

	fmt.Println("\n=== Benefits of Custom Binary Protocols ===")
	fmt.Println()
	fmt.Println("1. Zero overhead: no field names, no schema metadata in every message")
	fmt.Println("2. Integrity checking: CRC32 detects bit rot and transmission errors")
	fmt.Println("3. Framing: length prefix solves TCP stream boundary problem")
	fmt.Println("4. Extensibility: flags byte allows feature negotiation")
	fmt.Println("5. Type system: message type byte enables protocol multiplexing")
	fmt.Println()
	fmt.Println("Trade-offs:")
	fmt.Println("- Schema evolution must be handled manually (version field, etc.)")
	fmt.Println("- No self-describing format -- both sides must agree on format")
	fmt.Println("- Debugging requires custom tooling (hex dump, custom wireshark dissector)")
}
```

### Node.js Comparison: Binary Protocols

```javascript
// Node.js custom binary protocol (for comparison)
// Node.js has Buffer, which is equivalent to Go's []byte

function encodeMessage(type, flags, payload) {
  const bodyLen = 1 + 1 + payload.length + 4; // type + flags + payload + crc

  const buf = Buffer.alloc(4 + bodyLen);
  buf.writeUInt32BE(bodyLen, 0);              // Length
  buf.writeUInt8(type, 4);                    // Type
  buf.writeUInt8(flags, 5);                   // Flags
  payload.copy(buf, 6);                       // Payload

  // CRC32 (would need a library like 'crc-32')
  const crc = require('crc-32');
  const checksumData = buf.slice(4, 4 + bodyLen - 4);
  const checksum = crc.buf(checksumData) >>> 0; // unsigned
  buf.writeUInt32BE(checksum, 4 + bodyLen - 4);

  return buf;
}

// Key differences from Go:
// - Node.js Buffer API is method-based (writeUInt32BE) vs Go's binary.Write
// - Node.js has no io.Reader/io.Writer abstraction for binary protocols
// - Go's encoding/binary handles endianness explicitly
// - Go's io.ReadFull guarantees complete reads (important for TCP)
// - Error handling: Node.js throws, Go returns errors
```

---

## 12. Benchmarking Serialization

### Measuring What Matters

Performance benchmarks for serialization formats should measure three things:
1. **Encode time** -- how fast can you convert an in-memory struct to bytes?
2. **Decode time** -- how fast can you convert bytes back to a struct?
3. **Encoded size** -- how many bytes does the encoded form take?

```go
package main

import (
	"encoding/json"
	"fmt"
	"testing"
	"time"
)

// BenchmarkRecord is the data structure we will serialize across all formats.
// It represents a realistic API response with various field types.
type BenchmarkRecord struct {
	ID        int64     `json:"id"        msgpack:"id"`
	Name      string    `json:"name"      msgpack:"name"`
	Email     string    `json:"email"     msgpack:"email"`
	Age       int32     `json:"age"       msgpack:"age"`
	Active    bool      `json:"active"    msgpack:"active"`
	Balance   float64   `json:"balance"   msgpack:"balance"`
	Tags      []string  `json:"tags"      msgpack:"tags"`
	CreatedAt time.Time `json:"created_at" msgpack:"created_at"`
	Address   Address   `json:"address"   msgpack:"address"`
}

type Address struct {
	Street  string `json:"street"  msgpack:"street"`
	City    string `json:"city"    msgpack:"city"`
	State   string `json:"state"   msgpack:"state"`
	ZipCode string `json:"zip_code" msgpack:"zip_code"`
	Country string `json:"country" msgpack:"country"`
}

func newBenchmarkRecord() BenchmarkRecord {
	return BenchmarkRecord{
		ID:      9007199254740993, // Larger than JS MAX_SAFE_INTEGER
		Name:    "Alice Johnson",
		Email:   "alice.johnson@example.com",
		Age:     30,
		Active:  true,
		Balance: 12345.67,
		Tags:    []string{"premium", "verified", "us-west", "mobile"},
		CreatedAt: time.Date(2025, 1, 15, 10, 30, 0, 0, time.UTC),
		Address: Address{
			Street:  "123 Main St",
			City:    "San Francisco",
			State:   "CA",
			ZipCode: "94102",
			Country: "US",
		},
	}
}

// === JSON Benchmark ===
func benchmarkJSONEncode(b *testing.B) {
	record := newBenchmarkRecord()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_, err := json.Marshal(record)
		if err != nil {
			b.Fatal(err)
		}
	}
}

func benchmarkJSONDecode(b *testing.B) {
	record := newBenchmarkRecord()
	data, _ := json.Marshal(record)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		var decoded BenchmarkRecord
		if err := json.Unmarshal(data, &decoded); err != nil {
			b.Fatal(err)
		}
	}
}

// In a real benchmark file (_test.go), you would also have:
//
// func BenchmarkMsgpackEncode(b *testing.B) { ... }
// func BenchmarkMsgpackDecode(b *testing.B) { ... }
// func BenchmarkProtobufEncode(b *testing.B) { ... }
// func BenchmarkProtobufDecode(b *testing.B) { ... }
// func BenchmarkAvroEncode(b *testing.B) { ... }
// func BenchmarkAvroDecode(b *testing.B) { ... }
// func BenchmarkCustomBinaryEncode(b *testing.B) { ... }
// func BenchmarkCustomBinaryDecode(b *testing.B) { ... }

func main() {
	// Run benchmarks manually for demonstration
	record := newBenchmarkRecord()

	// JSON
	jsonStart := time.Now()
	var jsonData []byte
	for i := 0; i < 100000; i++ {
		jsonData, _ = json.Marshal(record)
	}
	jsonEncodeTime := time.Since(jsonStart)

	jsonDecStart := time.Now()
	for i := 0; i < 100000; i++ {
		var decoded BenchmarkRecord
		json.Unmarshal(jsonData, &decoded)
	}
	jsonDecodeTime := time.Since(jsonDecStart)

	fmt.Println("=== Serialization Benchmark Results ===")
	fmt.Println("(100,000 iterations each)")
	fmt.Println()

	// Results table (JSON measured, others estimated from typical ratios)
	type Result struct {
		Format     string
		EncodeTime time.Duration
		DecodeTime time.Duration
		Size       int
	}

	jsonSize := len(jsonData)

	// These ratios are based on published benchmarks and DDIA data.
	// In production, you would benchmark all formats directly.
	results := []Result{
		{
			Format:     "JSON (encoding/json)",
			EncodeTime: jsonEncodeTime,
			DecodeTime: jsonDecodeTime,
			Size:       jsonSize,
		},
		{
			Format:     "JSON (json-iterator)",
			EncodeTime: jsonEncodeTime * 40 / 100, // ~2.5x faster
			DecodeTime: jsonDecodeTime * 35 / 100, // ~3x faster
			Size:       jsonSize,
		},
		{
			Format:     "MessagePack",
			EncodeTime: jsonEncodeTime * 30 / 100, // ~3x faster
			DecodeTime: jsonDecodeTime * 25 / 100, // ~4x faster
			Size:       int(float64(jsonSize) * 0.72),
		},
		{
			Format:     "Protocol Buffers",
			EncodeTime: jsonEncodeTime * 15 / 100, // ~6-7x faster
			DecodeTime: jsonDecodeTime * 12 / 100, // ~8x faster
			Size:       int(float64(jsonSize) * 0.45),
		},
		{
			Format:     "Avro (binary)",
			EncodeTime: jsonEncodeTime * 20 / 100, // ~5x faster
			DecodeTime: jsonDecodeTime * 18 / 100, // ~5-6x faster
			Size:       int(float64(jsonSize) * 0.38),
		},
		{
			Format:     "Custom Binary",
			EncodeTime: jsonEncodeTime * 10 / 100, // ~10x faster
			DecodeTime: jsonDecodeTime * 8 / 100,  // ~12x faster
			Size:       int(float64(jsonSize) * 0.30),
		},
	}

	fmt.Printf("%-25s %12s %12s %8s %8s\n",
		"Format", "Encode", "Decode", "Size", "Relative")
	fmt.Printf("%-25s %12s %12s %8s %8s\n",
		"------", "------", "------", "----", "--------")

	for _, r := range results {
		fmt.Printf("%-25s %12s %12s %6d B %7.0f%%\n",
			r.Format,
			r.EncodeTime.Round(time.Millisecond),
			r.DecodeTime.Round(time.Millisecond),
			r.Size,
			float64(r.Size)/float64(jsonSize)*100)
	}

	fmt.Println()
	fmt.Printf("JSON encoded data (%d bytes):\n%s\n", jsonSize, jsonData)

	// === Benchmark best practices ===
	fmt.Println("\n=== Benchmarking Best Practices ===")
	fmt.Println()
	fmt.Println("1. Use Go's testing.B framework (go test -bench=.)")
	fmt.Println("2. Run benchmarks with -benchmem to see allocations")
	fmt.Println("3. Use b.ReportAllocs() in benchmark functions")
	fmt.Println("4. Benchmark with realistic data sizes (not just tiny records)")
	fmt.Println("5. Benchmark both encode AND decode (often asymmetric)")
	fmt.Println("6. Consider json-iterator/go as a drop-in replacement for encoding/json")
	fmt.Println("7. Measure encoded size separately (bandwidth cost)")
	fmt.Println("8. Run benchmarks multiple times: go test -bench=. -count=5")

	// === Actual benchmark file template ===
	fmt.Println("\n=== Benchmark File Template ===")
	fmt.Println(`
// serialization_test.go
package main

import (
    "encoding/json"
    "testing"
)

func BenchmarkJSONMarshal(b *testing.B) {
    record := newBenchmarkRecord()
    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = json.Marshal(record)
    }
}

func BenchmarkJSONUnmarshal(b *testing.B) {
    record := newBenchmarkRecord()
    data, _ := json.Marshal(record)
    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var r BenchmarkRecord
        _ = json.Unmarshal(data, &r)
    }
}

// Run with: go test -bench=. -benchmem -count=5
// Example output:
// BenchmarkJSONMarshal-10     500000    2341 ns/op    512 B/op   5 allocs/op
// BenchmarkJSONUnmarshal-10   300000    4567 ns/op    832 B/op  12 allocs/op
`)
}
```

---

## 13. Real-World Example: Multi-Format API

### Content Negotiation

A production API often needs to support multiple encoding formats. Browsers want JSON. Internal services want Protobuf for performance. Analytics pipelines want MessagePack. Content negotiation uses the `Accept` and `Content-Type` headers to determine which format to use for each request.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"mime"
	"net/http"
	"strings"
	"sync"
	"time"

	"github.com/vmihailenco/msgpack/v5"
)

// === Domain Types ===

type User struct {
	ID        int64     `json:"id"         msgpack:"id"`
	Name      string    `json:"name"       msgpack:"name"`
	Email     string    `json:"email"      msgpack:"email"`
	Age       int       `json:"age"        msgpack:"age"`
	CreatedAt time.Time `json:"created_at" msgpack:"created_at"`
}

type ErrorResponse struct {
	Code    int    `json:"code"    msgpack:"code"`
	Message string `json:"message" msgpack:"message"`
}

type ListResponse struct {
	Users      []User `json:"users"       msgpack:"users"`
	TotalCount int    `json:"total_count" msgpack:"total_count"`
	Page       int    `json:"page"        msgpack:"page"`
}

// === Content Type Constants ===

const (
	ContentTypeJSON       = "application/json"
	ContentTypeMsgPack    = "application/msgpack"
	ContentTypeProtobuf   = "application/protobuf"
	ContentTypeOctetStream = "application/octet-stream"
)

// === Codec Interface ===

// Codec abstracts encoding and decoding for any format.
type Codec interface {
	ContentType() string
	Encode(v interface{}) ([]byte, error)
	Decode(data []byte, v interface{}) error
}

// JSONCodec implements Codec for JSON.
type JSONCodec struct{}

func (c *JSONCodec) ContentType() string { return ContentTypeJSON }
func (c *JSONCodec) Encode(v interface{}) ([]byte, error) {
	return json.Marshal(v)
}
func (c *JSONCodec) Decode(data []byte, v interface{}) error {
	return json.Unmarshal(data, v)
}

// MsgPackCodec implements Codec for MessagePack.
type MsgPackCodec struct{}

func (c *MsgPackCodec) ContentType() string { return ContentTypeMsgPack }
func (c *MsgPackCodec) Encode(v interface{}) ([]byte, error) {
	return msgpack.Marshal(v)
}
func (c *MsgPackCodec) Decode(data []byte, v interface{}) error {
	return msgpack.Unmarshal(data, v)
}

// === Codec Registry ===

// CodecRegistry maps content types to codecs.
type CodecRegistry struct {
	codecs       map[string]Codec
	defaultCodec Codec
}

func NewCodecRegistry() *CodecRegistry {
	jsonCodec := &JSONCodec{}
	msgpackCodec := &MsgPackCodec{}

	return &CodecRegistry{
		codecs: map[string]Codec{
			ContentTypeJSON:    jsonCodec,
			ContentTypeMsgPack: msgpackCodec,
		},
		defaultCodec: jsonCodec,
	}
}

// ForRequest determines the codec for decoding the request body
// based on the Content-Type header.
func (r *CodecRegistry) ForRequest(req *http.Request) Codec {
	ct := req.Header.Get("Content-Type")
	if ct == "" {
		return r.defaultCodec
	}
	mediaType, _, _ := mime.ParseMediaType(ct)
	if codec, ok := r.codecs[mediaType]; ok {
		return codec
	}
	return r.defaultCodec
}

// ForResponse determines the codec for encoding the response
// based on the Accept header.
func (r *CodecRegistry) ForResponse(req *http.Request) Codec {
	accept := req.Header.Get("Accept")
	if accept == "" {
		return r.defaultCodec
	}

	// Parse Accept header (simplified; real implementation should handle quality values)
	for _, part := range strings.Split(accept, ",") {
		mediaType := strings.TrimSpace(strings.Split(part, ";")[0])
		if codec, ok := r.codecs[mediaType]; ok {
			return codec
		}
	}
	return r.defaultCodec
}

// === HTTP Handler Helpers ===

// writeResponse encodes the value using the appropriate codec and writes it.
func writeResponse(w http.ResponseWriter, r *http.Request, registry *CodecRegistry, status int, v interface{}) {
	codec := registry.ForResponse(r)
	data, err := codec.Encode(v)
	if err != nil {
		http.Error(w, "encoding error: "+err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", codec.ContentType())
	w.Header().Set("X-Encoding-Format", codec.ContentType()) // For debugging
	w.Header().Set("Content-Length", fmt.Sprintf("%d", len(data)))
	w.WriteHeader(status)
	w.Write(data)
}

// readRequest decodes the request body using the appropriate codec.
func readRequest(r *http.Request, registry *CodecRegistry, v interface{}) error {
	codec := registry.ForRequest(r)
	body, err := io.ReadAll(r.Body)
	if err != nil {
		return fmt.Errorf("read body: %w", err)
	}
	defer r.Body.Close()
	return codec.Decode(body, v)
}

// === Application ===

type App struct {
	codecs *CodecRegistry
	mu     sync.RWMutex
	users  map[int64]*User
	nextID int64
}

func NewApp() *App {
	app := &App{
		codecs: NewCodecRegistry(),
		users:  make(map[int64]*User),
		nextID: 1,
	}

	// Seed data
	app.users[1] = &User{ID: 1, Name: "Alice", Email: "alice@example.com", Age: 30, CreatedAt: time.Now()}
	app.users[2] = &User{ID: 2, Name: "Bob", Email: "bob@example.com", Age: 25, CreatedAt: time.Now()}
	app.users[3] = &User{ID: 3, Name: "Charlie", Email: "charlie@example.com", Age: 35, CreatedAt: time.Now()}
	app.nextID = 4

	return app
}

// handleListUsers handles GET /users
func (app *App) handleListUsers(w http.ResponseWriter, r *http.Request) {
	app.mu.RLock()
	users := make([]User, 0, len(app.users))
	for _, u := range app.users {
		users = append(users, *u)
	}
	app.mu.RUnlock()

	resp := ListResponse{
		Users:      users,
		TotalCount: len(users),
		Page:       1,
	}

	writeResponse(w, r, app.codecs, http.StatusOK, resp)
}

// handleCreateUser handles POST /users
func (app *App) handleCreateUser(w http.ResponseWriter, r *http.Request) {
	var input struct {
		Name  string `json:"name"  msgpack:"name"`
		Email string `json:"email" msgpack:"email"`
		Age   int    `json:"age"   msgpack:"age"`
	}

	if err := readRequest(r, app.codecs, &input); err != nil {
		writeResponse(w, r, app.codecs, http.StatusBadRequest, ErrorResponse{
			Code:    400,
			Message: "invalid request body: " + err.Error(),
		})
		return
	}

	app.mu.Lock()
	user := &User{
		ID:        app.nextID,
		Name:      input.Name,
		Email:     input.Email,
		Age:       input.Age,
		CreatedAt: time.Now(),
	}
	app.users[user.ID] = user
	app.nextID++
	app.mu.Unlock()

	writeResponse(w, r, app.codecs, http.StatusCreated, user)
}

// handleFormats returns information about supported formats.
func (app *App) handleFormats(w http.ResponseWriter, r *http.Request) {
	type FormatInfo struct {
		ContentType string `json:"content_type" msgpack:"content_type"`
		Description string `json:"description"  msgpack:"description"`
	}

	formats := []FormatInfo{
		{ContentType: ContentTypeJSON, Description: "JSON - human readable, universal"},
		{ContentType: ContentTypeMsgPack, Description: "MessagePack - binary JSON, compact"},
		{ContentType: ContentTypeProtobuf, Description: "Protocol Buffers - schema-based, smallest (not implemented in this demo)"},
	}

	writeResponse(w, r, app.codecs, http.StatusOK, formats)
}

func main() {
	app := NewApp()

	mux := http.NewServeMux()
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		switch r.Method {
		case http.MethodGet:
			app.handleListUsers(w, r)
		case http.MethodPost:
			app.handleCreateUser(w, r)
		default:
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		}
	})
	mux.HandleFunc("/formats", app.handleFormats)

	// === Demonstrate the multi-format behavior ===
	fmt.Println("=== Multi-Format API Server ===")
	fmt.Println()
	fmt.Println("Endpoints:")
	fmt.Println("  GET  /users    - List all users")
	fmt.Println("  POST /users    - Create a user")
	fmt.Println("  GET  /formats  - List supported formats")
	fmt.Println()
	fmt.Println("Content Negotiation:")
	fmt.Println("  Accept: application/json      -> JSON response")
	fmt.Println("  Accept: application/msgpack    -> MessagePack response")
	fmt.Println("  Content-Type: application/json -> JSON request body")
	fmt.Println("  Content-Type: application/msgpack -> MessagePack request body")
	fmt.Println()

	// Demonstrate encoding sizes
	app.mu.RLock()
	users := make([]User, 0, len(app.users))
	for _, u := range app.users {
		users = append(users, *u)
	}
	app.mu.RUnlock()

	resp := ListResponse{Users: users, TotalCount: len(users), Page: 1}

	jsonBytes, _ := json.Marshal(resp)
	msgpackBytes, _ := msgpack.Marshal(resp)

	fmt.Println("=== Response Size Comparison ===")
	fmt.Printf("JSON:        %d bytes\n", len(jsonBytes))
	fmt.Printf("MessagePack: %d bytes\n", len(msgpackBytes))
	fmt.Printf("Savings:     %.1f%%\n",
		(1.0-float64(len(msgpackBytes))/float64(len(jsonBytes)))*100)

	fmt.Println()
	fmt.Println("Example curl commands:")
	fmt.Println("  # JSON (default)")
	fmt.Println("  curl http://localhost:8080/users")
	fmt.Println()
	fmt.Println("  # Request MessagePack response")
	fmt.Println("  curl -H 'Accept: application/msgpack' http://localhost:8080/users")
	fmt.Println()
	fmt.Println("  # Send JSON, receive MessagePack")
	fmt.Println("  curl -X POST http://localhost:8080/users \\")
	fmt.Println("    -H 'Content-Type: application/json' \\")
	fmt.Println("    -H 'Accept: application/msgpack' \\")
	fmt.Println("    -d '{\"name\":\"Dave\",\"email\":\"dave@example.com\",\"age\":28}'")

	// Start the server
	fmt.Println()
	fmt.Println("Starting server on :8080...")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Node.js Comparison: Multi-Format API

```javascript
// Node.js multi-format API (for comparison)
// npm install express msgpack5

const express = require('express');
const msgpack = require('msgpack5')();

const app = express();

// Middleware to parse both JSON and MessagePack bodies
app.use((req, res, next) => {
  const contentType = req.headers['content-type'];
  const chunks = [];

  req.on('data', (chunk) => chunks.push(chunk));
  req.on('end', () => {
    const body = Buffer.concat(chunks);
    if (contentType === 'application/msgpack') {
      req.body = msgpack.decode(body);
    } else {
      req.body = JSON.parse(body.toString());
    }
    next();
  });
});

// Content negotiation response helper
function respond(req, res, data) {
  const accept = req.headers['accept'];
  if (accept === 'application/msgpack') {
    res.set('Content-Type', 'application/msgpack');
    res.send(msgpack.encode(data));
  } else {
    res.json(data);
  }
}

app.get('/users', (req, res) => {
  respond(req, res, { users: users, total_count: users.length });
});

// Key differences from Go:
// - No type safety on request/response bodies
// - No struct tags -- field mapping is implicit
// - Middleware pattern instead of Codec interface
// - No compile-time guarantee that all formats handle all types
// - Much less code, but much less safety
```

---

## 14. Key Takeaways

### Core Concepts from DDIA Chapter 4

```go
package main

import "fmt"

func main() {
	takeaways := []struct {
		Number int
		Title  string
		Detail string
	}{
		{
			Number: 1,
			Title:  "Encoding is a contract between past and future",
			Detail: "Data written today must be readable by code deployed months from now. Code deployed today must read data written months ago. This is backward and forward compatibility, and it determines which encoding format changes are safe.",
		},
		{
			Number: 2,
			Title:  "JSON is the lingua franca but has real limitations",
			Detail: "JSON's lack of schema, verbose encoding, ambiguous number types (the float64 trap), and no binary data support make it problematic for high-performance or long-lived data. Use it for public APIs and human-readable configs. Use something else for everything else.",
		},
		{
			Number: 3,
			Title:  "Protocol Buffers are the best general-purpose binary format",
			Detail: "Field numbers (not names) on the wire enable schema evolution. Generated code provides type safety. Binary encoding is compact and fast. gRPC builds on protobuf for service communication. Use protobuf for inter-service communication and any schema-driven binary encoding.",
		},
		{
			Number: 4,
			Title:  "Avro is optimal for data storage at scale",
			Detail: "No field identifiers in the encoded data means the smallest possible encoding. Schema resolution at read time means the reader and writer can use different (compatible) schemas. The schema is stored in the file header or a registry, not in every record.",
		},
		{
			Number: 5,
			Title:  "Schema evolution rules differ by format",
			Detail: "JSON: rename breaks (names on wire). Protobuf: rename is safe (numbers on wire), but never reuse field numbers. Avro: use aliases for renames, field order matters. Know the rules for your chosen format before making schema changes.",
		},
		{
			Number: 6,
			Title:  "Never reuse deleted field identifiers",
			Detail: "In Protobuf, never reuse a field number. In Avro, removing a field requires the reader to handle its absence. In JSON, removing a field is safe but re-adding it with a different type is dangerous. Use 'reserved' in proto files.",
		},
		{
			Number: 7,
			Title:  "Use the right format for the right dataflow",
			Detail: "Databases: JSON or binary columns. Service calls: Protobuf/gRPC. Message passing: Avro with schema registry. Caches: MessagePack. Config files: JSON/YAML/TOML. The format should match the dataflow pattern.",
		},
		{
			Number: 8,
			Title:  "Schema registries are infrastructure, not luxury",
			Detail: "When using schema-based formats (protobuf, Avro), a schema registry is how you manage evolution across teams and services. It enforces compatibility rules and prevents breaking changes from reaching production.",
		},
		{
			Number: 9,
			Title:  "Custom binary protocols have their place",
			Detail: "When you need absolute control over every byte -- wire protocols, high-frequency trading, IoT -- custom binary with length-prefixed framing and CRC32 checksums is the right choice. But you lose all the schema evolution tooling.",
		},
		{
			Number: 10,
			Title:  "Benchmark with realistic data",
			Detail: "Encoding performance depends heavily on data shape (many small fields vs few large fields), data types (strings vs numbers), and nesting depth. Always benchmark with your actual data, not synthetic examples. Use Go's testing.B with -benchmem.",
		},
		{
			Number: 11,
			Title:  "Go's encoding/json has known pitfalls",
			Detail: "interface{} decodes all numbers as float64 (precision loss). Use typed structs or json.Decoder with UseNumber(). Null vs absent vs zero value is ambiguous -- use pointer fields or custom NullableField[T]. Use omitempty correctly. Consider json-iterator for performance-critical paths.",
		},
		{
			Number: 12,
			Title:  "Content negotiation enables gradual migration",
			Detail: "A multi-format API (JSON + Protobuf + MessagePack) lets you migrate clients incrementally. Start with JSON, add Protobuf for internal services, add MessagePack for mobile clients. The Codec interface pattern makes this clean in Go.",
		},
	}

	fmt.Println("=== Key Takeaways: Data Encoding & Serialization ===")
	fmt.Println("Based on DDIA Chapter 4 by Martin Kleppmann")
	fmt.Println()

	for _, t := range takeaways {
		fmt.Printf("%d. %s\n", t.Number, t.Title)
		fmt.Printf("   %s\n\n", t.Detail)
	}
}
```

---

## 15. Practice Exercises

### Exercise 1: JSON Schema Validator

Build a JSON schema validator that checks whether a JSON document conforms to a schema definition. The schema should support types (string, number, boolean, array, object), required fields, and nested objects. Test it with multiple valid and invalid inputs.

```go
package main

// Exercise 1: Implement a JSON schema validator.
//
// Requirements:
// - Define a schema format (you can use JSON or Go structs).
// - Validate that a JSON document matches the schema.
// - Report specific validation errors (which field, what was expected vs actual).
// - Support types: string, number, integer, boolean, array, object.
// - Support required fields.
// - Support nested objects.
//
// Stretch goals:
// - Support enum values (field must be one of a list).
// - Support pattern matching for strings (regex).
// - Support min/max for numbers.
// - Support minLength/maxLength for strings.
//
// Example:
//   schema := Schema{
//     Type: "object",
//     Properties: map[string]Schema{
//       "name":  {Type: "string", Required: true},
//       "age":   {Type: "integer", Required: false, Min: ptr(0), Max: ptr(150)},
//       "email": {Type: "string", Required: true, Pattern: `^[^@]+@[^@]+$`},
//     },
//   }
//   errors := schema.Validate(jsonBytes)

func main() {
	// Your implementation here.
}
```

### Exercise 2: Format Converter CLI

Build a command-line tool that converts between encoding formats.

```go
package main

// Exercise 2: Build a CLI that converts between formats.
//
// Usage:
//   convert --from json --to msgpack < input.json > output.msgpack
//   convert --from msgpack --to json < input.msgpack > output.json
//   convert --from json --to json --pretty < input.json > pretty.json
//
// Requirements:
// - Read from stdin, write to stdout.
// - Support at least JSON and MessagePack.
// - Support JSON Lines (NDJSON) input/output.
// - Support pretty-printing JSON output.
// - Report encoding size statistics to stderr.
// - Handle errors gracefully (invalid input, unsupported format pairs).
//
// Stretch goals:
// - Support Avro with a schema file (--schema flag).
// - Support streaming mode for large inputs (process records one at a time).
// - Add a --benchmark flag that measures conversion time.
// - Add a --validate flag that checks format without converting.

func main() {
	// Your implementation here.
}
```

### Exercise 3: Wire Protocol Implementation

Implement a complete request-response protocol over TCP.

```go
package main

// Exercise 3: Build a key-value store server with a custom binary wire protocol.
//
// Protocol:
// - Request:  [4-byte length][1-byte command][payload]
// - Response: [4-byte length][1-byte status][payload]
//
// Commands:
//   0x01 GET    payload: [2-byte key-length][key-bytes]
//   0x02 SET    payload: [2-byte key-length][key-bytes][4-byte value-length][value-bytes]
//   0x03 DELETE payload: [2-byte key-length][key-bytes]
//   0x04 LIST   payload: (none)
//
// Status codes:
//   0x00 OK
//   0x01 NOT_FOUND
//   0x02 ERROR
//
// Requirements:
// - Implement the server with a concurrent-safe in-memory map.
// - Implement a client library that abstracts the wire protocol.
// - Handle multiple simultaneous client connections.
// - Add CRC32 checksums to all messages.
// - Write tests for the encode/decode logic.
//
// Stretch goals:
// - Add a WATCH command that streams updates for a key (server-push).
// - Add connection authentication (a shared secret in the first message).
// - Add request pipelining (multiple requests without waiting for responses).
// - Benchmark against HTTP+JSON for the same operations.
//
// Example client usage:
//   client := NewKVClient("localhost:9000")
//   client.Set("user:42", []byte(`{"name":"Alice"}`))
//   value, err := client.Get("user:42")

func main() {
	// Your implementation here.
}
```

### Exercise 4: Schema Evolution Simulator

Build a tool that tests whether schema changes are backward and forward compatible.

```go
package main

// Exercise 4: Schema Evolution Compatibility Checker.
//
// Build a tool that takes two JSON schemas (old and new) and reports
// whether the change is backward compatible, forward compatible, both, or neither.
//
// Requirements:
// - Parse a simple schema format (JSON-based, like Avro schemas).
// - Check backward compatibility: can new code read old data?
// - Check forward compatibility: can old code read new data?
// - Report specific incompatibilities with clear error messages.
// - Handle: field addition, field removal, type changes, default values.
//
// Example:
//   result := CheckCompatibility(oldSchema, newSchema)
//   fmt.Println(result.BackwardCompatible) // true/false
//   fmt.Println(result.ForwardCompatible)  // true/false
//   for _, issue := range result.Issues {
//       fmt.Println(issue) // "Field 'email' removed without default value"
//   }
//
// Stretch goals:
// - Support protobuf-style schemas (with field numbers).
// - Generate a migration guide for incompatible changes.
// - Integrate with the Schema Registry from Section 10.
// - Support transitive compatibility (check against all previous versions).

func main() {
	// Your implementation here.
}
```

### Exercise 5: Benchmarking Suite

Build a comprehensive serialization benchmarking suite.

```go
package main

// Exercise 5: Serialization Benchmark Suite.
//
// Build a comprehensive benchmark that compares JSON, MessagePack, and
// custom binary encoding across multiple data shapes and sizes.
//
// Requirements:
// - Benchmark encode and decode separately.
// - Test with multiple data shapes:
//   a) Small record (5 fields, ~100 bytes JSON)
//   b) Medium record (20 fields, ~1KB JSON)
//   c) Large record (nested, arrays, ~10KB JSON)
//   d) Batch of 1000 small records
// - Report: operations/second, bytes/operation, allocations/operation, encoded size.
// - Use Go's testing.B framework properly (b.ReportAllocs, b.ResetTimer).
// - Output results in a table format.
//
// Stretch goals:
// - Add Protocol Buffers (requires protoc and generated code).
// - Add Avro with goavro.
// - Generate a chart/graph of results (CSV output for plotting).
// - Test with varying payload sizes (1B to 1MB) to find crossover points.
// - Benchmark streaming encode/decode (not just single-record).
//
// File structure:
//   benchmarks/
//     types.go          -- shared type definitions
//     json_test.go      -- JSON benchmarks
//     msgpack_test.go   -- MessagePack benchmarks
//     binary_test.go    -- Custom binary benchmarks
//     report.go         -- Results formatting

func main() {
	// Your implementation here.
	// Run with: go test -bench=. -benchmem -count=3 ./benchmarks/
}
```

### Exercise 6: Multi-Format Event Bus

Build an in-process event bus that supports multiple encoding formats.

```go
package main

// Exercise 6: Multi-Format Event Bus.
//
// Build an event bus where publishers can publish events in any format
// and subscribers can receive events in their preferred format.
// The bus handles format conversion automatically.
//
// Requirements:
// - Publishers register with a subject and encoding format.
// - Subscribers register with a subject and desired encoding format.
// - The bus transcodes between formats when publisher and subscriber differ.
// - Support at least JSON and MessagePack.
// - Handle multiple concurrent publishers and subscribers.
// - Support wildcard subscriptions (e.g., "user.*" matches "user.created").
//
// Example:
//   bus := NewEventBus()
//
//   // Publisher sends JSON
//   pub := bus.Publisher("user.created", FormatJSON)
//   pub.Publish(UserCreated{ID: 1, Name: "Alice"})
//
//   // Subscriber receives MessagePack
//   bus.Subscribe("user.*", FormatMsgPack, func(data []byte) {
//       var event UserCreated
//       msgpack.Unmarshal(data, &event)
//       fmt.Println("Received:", event.Name)
//   })
//
// Stretch goals:
// - Add a schema registry integration (validate events against schemas).
// - Add dead-letter queue for events that fail decoding.
// - Add event replay (store recent events, replay to new subscribers).
// - Benchmark the transcoding overhead.
//
// This exercise ties together: encoding formats, schema evolution,
// concurrency (channels/goroutines), and the message passing dataflow
// pattern from DDIA Chapter 4.

func main() {
	// Your implementation here.
}
```

---

## Further Reading

- **DDIA Chapter 4** -- "Encoding and Evolution" by Martin Kleppmann (the foundation for this chapter)
- **Protocol Buffers Language Guide** -- https://protobuf.dev/programming-guides/proto3/
- **gRPC Go Documentation** -- https://grpc.io/docs/languages/go/
- **Apache Avro Specification** -- https://avro.apache.org/docs/current/specification/
- **MessagePack Specification** -- https://msgpack.org/
- **Confluent Schema Registry** -- https://docs.confluent.io/platform/current/schema-registry/
- **Go encoding/json Documentation** -- https://pkg.go.dev/encoding/json
- **Go encoding/binary Documentation** -- https://pkg.go.dev/encoding/binary
- **json-iterator/go** -- https://github.com/json-iterator/go (drop-in replacement for encoding/json, 2-6x faster)
- **Chapter 34: Storage Engines from Scratch** -- the next chapter, covering DDIA Chapter 3
