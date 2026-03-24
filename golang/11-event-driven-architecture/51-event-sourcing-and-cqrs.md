# Chapter 51: Event Sourcing and CQRS in Go

## Table of Contents

1. [Event Sourcing Fundamentals](#1-event-sourcing-fundamentals)
2. [Building an Event Store in Go](#2-building-an-event-store-in-go)
3. [Aggregates and Command Handling](#3-aggregates-and-command-handling)
4. [Event Versioning and Schema Evolution](#4-event-versioning-and-schema-evolution)
5. [Snapshots for Performance Optimization](#5-snapshots-for-performance-optimization)
6. [CQRS Pattern](#6-cqrs-pattern)
7. [Separate Read and Write Models](#7-separate-read-and-write-models)
8. [Building Projections](#8-building-projections)
9. [Eventual Consistency Handling](#9-eventual-consistency-handling)
10. [Saga Pattern for Distributed Transactions](#10-saga-pattern-for-distributed-transactions)
11. [Process Managers: Choreography vs Orchestration](#11-process-managers-choreography-vs-orchestration)
12. [Outbox Pattern for Reliable Event Publishing](#12-outbox-pattern-for-reliable-event-publishing)
13. [Complete Example: E-Commerce Order System](#13-complete-example-e-commerce-order-system)
14. [Testing Event-Sourced Systems](#14-testing-event-sourced-systems)
15. [When to Use (and When NOT to Use) Event Sourcing](#15-when-to-use-and-when-not-to-use-event-sourcing)
16. [Common Pitfalls and Lessons Learned](#16-common-pitfalls-and-lessons-learned)

---

## 1. Event Sourcing Fundamentals

### What Is Event Sourcing?

In traditional systems, you store the **current state** of an entity in a database row. When something changes, you overwrite the old state with new state. The history of how that entity reached its current state is lost forever.

**Event sourcing** flips this model entirely. Instead of storing current state, you store every **event** (state change) that has ever happened to an entity. The current state is derived by replaying those events from the beginning.

Think of it like a bank account. A traditional system stores your balance: `$1,542.37`. An event-sourced system stores every deposit and withdrawal that ever occurred. The balance is a **projection** -- a derived value you compute by replaying the transaction history.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Event** | An immutable fact that something happened in the past. Always named in past tense (e.g., `OrderPlaced`, `ItemAdded`). |
| **Event Store** | An append-only log that persists events. Events are never updated or deleted. |
| **Aggregate** | A domain object whose state is rebuilt by replaying its events. The consistency boundary. |
| **Projection** | A read-optimized view built by processing events. Can be rebuilt at any time from the event log. |
| **Stream** | A sequence of events belonging to a single aggregate instance (e.g., `order-abc123`). |
| **Snapshot** | A cached state of an aggregate at a particular version, used to avoid replaying all events. |

### The Fundamental Event Type

```go
package eventsourcing

import (
    "encoding/json"
    "time"

    "github.com/google/uuid"
)

// Event represents a single domain event -- an immutable fact that something happened.
type Event struct {
    // Unique identifier for this event.
    ID string `json:"id"`

    // The type/name of the event, e.g. "OrderPlaced", "ItemAdded".
    Type string `json:"type"`

    // The aggregate this event belongs to, e.g. "Order", "Account".
    AggregateType string `json:"aggregate_type"`

    // The unique ID of the specific aggregate instance.
    AggregateID string `json:"aggregate_id"`

    // The version (sequence number) of this event within the aggregate's stream.
    // Starts at 1 and increments by 1 for each new event.
    Version int `json:"version"`

    // The serialized event payload (domain-specific data).
    Data json.RawMessage `json:"data"`

    // Optional metadata (causation ID, correlation ID, user info, etc.).
    Metadata json.RawMessage `json:"metadata"`

    // When the event occurred.
    Timestamp time.Time `json:"timestamp"`
}

// NewEvent creates a new event with a generated UUID and the current timestamp.
func NewEvent(
    eventType string,
    aggregateType string,
    aggregateID string,
    version int,
    data interface{},
    metadata map[string]string,
) (Event, error) {
    dataBytes, err := json.Marshal(data)
    if err != nil {
        return Event{}, err
    }

    metaBytes, err := json.Marshal(metadata)
    if err != nil {
        return Event{}, err
    }

    return Event{
        ID:            uuid.New().String(),
        Type:          eventType,
        AggregateType: aggregateType,
        AggregateID:   aggregateID,
        Version:       version,
        Data:          dataBytes,
        Metadata:      metaBytes,
        Timestamp:     time.Now().UTC(),
    }, nil
}
```

### Events as Source of Truth

In an event-sourced system, the event log is the **single source of truth**. Everything else -- database tables, search indexes, caches -- is a derived projection that can be rebuilt.

```
Traditional:                 Event Sourced:

  Command                      Command
     |                            |
     v                            v
  [Validate]                  [Validate]
     |                            |
     v                            v
  UPDATE row                  APPEND event(s) to log
  SET balance = 950              |
                                 v
                           [Project into read models]
                                 |
                           +-----+------+
                           |            |
                           v            v
                       SQL table    Search index
                       (balance)    (transactions)
```

### Why Events Must Be Immutable

Events represent facts about the past. You cannot change the past. If an event was wrong, you publish a **compensating event** (e.g., `OrderCancelled` to undo an `OrderPlaced`), not modify or delete the original.

This immutability gives you:

- **Complete audit trail** -- every state change is recorded.
- **Temporal queries** -- "what was the state at 3pm yesterday?"
- **Debugging** -- replay events to reproduce any bug.
- **Rebuildable projections** -- delete a read model and rebuild it from the event log.

---

## 2. Building an Event Store in Go

### Event Store Interface

Every event store implementation must support a minimal set of operations: appending events and loading them back.

```go
package eventsourcing

import "context"

// EventStore is the interface that all event store implementations must satisfy.
type EventStore interface {
    // AppendEvents atomically appends events to a stream.
    // expectedVersion is used for optimistic concurrency control:
    //   -1 means "stream should not exist" (new aggregate)
    //    0 means "no concurrency check"
    //   >0 means "stream must be at exactly this version"
    AppendEvents(ctx context.Context, streamID string, expectedVersion int, events []Event) error

    // LoadEvents loads all events for a given stream, in order.
    LoadEvents(ctx context.Context, streamID string) ([]Event, error)

    // LoadEventsFrom loads events for a stream starting from a given version (inclusive).
    LoadEventsFrom(ctx context.Context, streamID string, fromVersion int) ([]Event, error)

    // LoadAllEvents loads all events across all streams, ordered globally.
    // Used by projections that need to process every event in the system.
    LoadAllEvents(ctx context.Context, fromPosition int64) ([]Event, error)
}
```

### In-Memory Event Store

An in-memory implementation is invaluable for unit testing and prototyping.

```go
package eventsourcing

import (
    "context"
    "fmt"
    "sync"
)

// InMemoryEventStore stores events in memory. Suitable for tests and prototypes.
type InMemoryEventStore struct {
    mu       sync.RWMutex
    streams  map[string][]Event  // streamID -> events
    allEvents []Event            // global ordered log
}

func NewInMemoryEventStore() *InMemoryEventStore {
    return &InMemoryEventStore{
        streams: make(map[string][]Event),
    }
}

func (s *InMemoryEventStore) AppendEvents(
    ctx context.Context,
    streamID string,
    expectedVersion int,
    events []Event,
) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    stream := s.streams[streamID]
    currentVersion := len(stream)

    // Optimistic concurrency check.
    switch {
    case expectedVersion == -1 && currentVersion > 0:
        return fmt.Errorf("stream %s already exists (version %d)", streamID, currentVersion)
    case expectedVersion > 0 && currentVersion != expectedVersion:
        return fmt.Errorf(
            "concurrency conflict on stream %s: expected version %d, got %d",
            streamID, expectedVersion, currentVersion,
        )
    // expectedVersion == 0 means "no check"
    }

    for i := range events {
        events[i].Version = currentVersion + i + 1
    }

    s.streams[streamID] = append(stream, events...)
    s.allEvents = append(s.allEvents, events...)

    return nil
}

func (s *InMemoryEventStore) LoadEvents(
    ctx context.Context,
    streamID string,
) ([]Event, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    stream, ok := s.streams[streamID]
    if !ok {
        return nil, nil
    }

    // Return a copy to prevent mutation.
    result := make([]Event, len(stream))
    copy(result, stream)
    return result, nil
}

func (s *InMemoryEventStore) LoadEventsFrom(
    ctx context.Context,
    streamID string,
    fromVersion int,
) ([]Event, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    stream, ok := s.streams[streamID]
    if !ok {
        return nil, nil
    }

    if fromVersion > len(stream) {
        return nil, nil
    }

    // Versions are 1-based, slice indices are 0-based.
    result := make([]Event, len(stream)-(fromVersion-1))
    copy(result, stream[fromVersion-1:])
    return result, nil
}

func (s *InMemoryEventStore) LoadAllEvents(
    ctx context.Context,
    fromPosition int64,
) ([]Event, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    if fromPosition >= int64(len(s.allEvents)) {
        return nil, nil
    }

    result := make([]Event, int64(len(s.allEvents))-fromPosition)
    copy(result, s.allEvents[fromPosition:])
    return result, nil
}
```

### PostgreSQL-Backed Event Store

For production use, PostgreSQL is a solid choice. Its SERIALIZABLE isolation and NOTIFY/LISTEN make it a good fit.

```sql
-- Schema for a PostgreSQL event store.

CREATE TABLE events (
    -- Global sequence number, monotonically increasing.
    global_position BIGSERIAL PRIMARY KEY,

    -- Unique event ID (UUID).
    event_id        UUID NOT NULL UNIQUE,

    -- Stream identifier, e.g. "Order-abc123".
    stream_id       TEXT NOT NULL,

    -- Position within the stream (1-based).
    stream_version  INT NOT NULL,

    -- Event type name, e.g. "OrderPlaced".
    event_type      TEXT NOT NULL,

    -- Serialized event payload (JSON).
    data            JSONB NOT NULL,

    -- Optional metadata (correlation IDs, user info, etc.).
    metadata        JSONB DEFAULT '{}',

    -- When the event was stored.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Ensure no two events share the same position in a stream.
    UNIQUE (stream_id, stream_version)
);

-- Index for loading events by stream.
CREATE INDEX idx_events_stream_id ON events (stream_id, stream_version);

-- Index for global event ordering (used by projections).
CREATE INDEX idx_events_global_position ON events (global_position);

-- Index for loading events by type (useful for certain projections).
CREATE INDEX idx_events_event_type ON events (event_type);

-- Notification channel for new events.
CREATE OR REPLACE FUNCTION notify_new_event()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('new_event', json_build_object(
        'global_position', NEW.global_position,
        'stream_id', NEW.stream_id,
        'event_type', NEW.event_type
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER events_notify
    AFTER INSERT ON events
    FOR EACH ROW
    EXECUTE FUNCTION notify_new_event();
```

```go
package eventsourcing

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"

    "github.com/lib/pq"
)

// PostgresEventStore implements EventStore backed by PostgreSQL.
type PostgresEventStore struct {
    db *sql.DB
}

func NewPostgresEventStore(db *sql.DB) *PostgresEventStore {
    return &PostgresEventStore{db: db}
}

func (s *PostgresEventStore) AppendEvents(
    ctx context.Context,
    streamID string,
    expectedVersion int,
    events []Event,
) error {
    tx, err := s.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    // Check current stream version for optimistic concurrency.
    var currentVersion int
    err = tx.QueryRowContext(ctx,
        "SELECT COALESCE(MAX(stream_version), 0) FROM events WHERE stream_id = $1",
        streamID,
    ).Scan(&currentVersion)
    if err != nil {
        return fmt.Errorf("check version: %w", err)
    }

    switch {
    case expectedVersion == -1 && currentVersion > 0:
        return fmt.Errorf("stream %s already exists", streamID)
    case expectedVersion > 0 && currentVersion != expectedVersion:
        return &ConcurrencyError{
            StreamID:        streamID,
            ExpectedVersion: expectedVersion,
            ActualVersion:   currentVersion,
        }
    }

    // Insert events.
    stmt, err := tx.PrepareContext(ctx, `
        INSERT INTO events (event_id, stream_id, stream_version, event_type, data, metadata, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
    `)
    if err != nil {
        return fmt.Errorf("prepare insert: %w", err)
    }
    defer stmt.Close()

    for i, event := range events {
        version := currentVersion + i + 1
        _, err = stmt.ExecContext(ctx,
            event.ID,
            streamID,
            version,
            event.Type,
            event.Data,
            event.Metadata,
            event.Timestamp,
        )
        if err != nil {
            // Check for unique constraint violation (concurrent write).
            if pqErr, ok := err.(*pq.Error); ok && pqErr.Code == "23505" {
                return &ConcurrencyError{
                    StreamID:        streamID,
                    ExpectedVersion: expectedVersion,
                    ActualVersion:   currentVersion,
                }
            }
            return fmt.Errorf("insert event: %w", err)
        }
    }

    return tx.Commit()
}

func (s *PostgresEventStore) LoadEvents(
    ctx context.Context,
    streamID string,
) ([]Event, error) {
    return s.LoadEventsFrom(ctx, streamID, 1)
}

func (s *PostgresEventStore) LoadEventsFrom(
    ctx context.Context,
    streamID string,
    fromVersion int,
) ([]Event, error) {
    rows, err := s.db.QueryContext(ctx, `
        SELECT event_id, stream_id, stream_version, event_type, data, metadata, created_at
        FROM events
        WHERE stream_id = $1 AND stream_version >= $2
        ORDER BY stream_version ASC
    `, streamID, fromVersion)
    if err != nil {
        return nil, fmt.Errorf("query events: %w", err)
    }
    defer rows.Close()

    var events []Event
    for rows.Next() {
        var e Event
        err := rows.Scan(
            &e.ID,
            &e.AggregateID,
            &e.Version,
            &e.Type,
            &e.Data,
            &e.Metadata,
            &e.Timestamp,
        )
        if err != nil {
            return nil, fmt.Errorf("scan event: %w", err)
        }
        events = append(events, e)
    }
    return events, rows.Err()
}

func (s *PostgresEventStore) LoadAllEvents(
    ctx context.Context,
    fromPosition int64,
) ([]Event, error) {
    rows, err := s.db.QueryContext(ctx, `
        SELECT event_id, stream_id, stream_version, event_type, data, metadata, created_at
        FROM events
        WHERE global_position > $1
        ORDER BY global_position ASC
    `, fromPosition)
    if err != nil {
        return nil, fmt.Errorf("query all events: %w", err)
    }
    defer rows.Close()

    var events []Event
    for rows.Next() {
        var e Event
        err := rows.Scan(
            &e.ID,
            &e.AggregateID,
            &e.Version,
            &e.Type,
            &e.Data,
            &e.Metadata,
            &e.Timestamp,
        )
        if err != nil {
            return nil, fmt.Errorf("scan event: %w", err)
        }
        events = append(events, e)
    }
    return events, rows.Err()
}

// SubscribeToEvents listens for new events via PostgreSQL LISTEN/NOTIFY.
func (s *PostgresEventStore) SubscribeToEvents(
    ctx context.Context,
    connStr string,
    handler func(notification EventNotification),
) error {
    listener := pq.NewListener(connStr, 0, 0, nil)
    if err := listener.Listen("new_event"); err != nil {
        return fmt.Errorf("listen: %w", err)
    }
    defer listener.Close()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case n := <-listener.Notify:
            if n == nil {
                continue
            }
            var notification EventNotification
            if err := json.Unmarshal([]byte(n.Extra), &notification); err != nil {
                continue
            }
            handler(notification)
        }
    }
}

// EventNotification is the payload sent over PostgreSQL NOTIFY.
type EventNotification struct {
    GlobalPosition int64  `json:"global_position"`
    StreamID       string `json:"stream_id"`
    EventType      string `json:"event_type"`
}

// ConcurrencyError is returned when optimistic concurrency checks fail.
type ConcurrencyError struct {
    StreamID        string
    ExpectedVersion int
    ActualVersion   int
}

func (e *ConcurrencyError) Error() string {
    return fmt.Sprintf(
        "concurrency conflict on stream %s: expected version %d, actual %d",
        e.StreamID, e.ExpectedVersion, e.ActualVersion,
    )
}
```

### EventStoreDB Client

EventStoreDB is a purpose-built database for event sourcing. Here is how to use the official Go client.

```go
package eventsourcing

import (
    "context"
    "encoding/json"
    "fmt"
    "io"

    "github.com/EventStore/EventStore-Client-Go/v4/esdb"
    "github.com/google/uuid"
)

// ESDBEventStore wraps EventStoreDB as an EventStore implementation.
type ESDBEventStore struct {
    client *esdb.Client
}

func NewESDBEventStore(connectionString string) (*ESDBEventStore, error) {
    settings, err := esdb.ParseConnectionString(connectionString)
    if err != nil {
        return nil, fmt.Errorf("parse connection string: %w", err)
    }

    client, err := esdb.NewClient(settings)
    if err != nil {
        return nil, fmt.Errorf("create client: %w", err)
    }

    return &ESDBEventStore{client: client}, nil
}

func (s *ESDBEventStore) AppendEvents(
    ctx context.Context,
    streamID string,
    expectedVersion int,
    events []Event,
) error {
    eventData := make([]esdb.EventData, len(events))
    for i, e := range events {
        eventData[i] = esdb.EventData{
            EventID:     uuid.MustParse(e.ID),
            EventType:   e.Type,
            ContentType: esdb.ContentTypeJson,
            Data:        e.Data,
            Metadata:    e.Metadata,
        }
    }

    var opts esdb.AppendToStreamOptions

    switch {
    case expectedVersion == -1:
        opts.ExpectedRevision = esdb.NoStream{}
    case expectedVersion == 0:
        opts.ExpectedRevision = esdb.Any{}
    default:
        opts.ExpectedRevision = esdb.Revision(uint64(expectedVersion - 1))
    }

    _, err := s.client.AppendToStream(ctx, streamID, opts, eventData...)
    if err != nil {
        return fmt.Errorf("append to stream: %w", err)
    }
    return nil
}

func (s *ESDBEventStore) LoadEvents(
    ctx context.Context,
    streamID string,
) ([]Event, error) {
    stream, err := s.client.ReadStream(ctx, streamID, esdb.ReadStreamOptions{
        Direction: esdb.Forwards,
        From:      esdb.Start{},
    }, ^uint64(0)) // Read all events.
    if err != nil {
        return nil, fmt.Errorf("read stream: %w", err)
    }
    defer stream.Close()

    var events []Event
    for {
        resolved, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, fmt.Errorf("recv event: %w", err)
        }

        original := resolved.Event
        events = append(events, Event{
            ID:          original.EventID.String(),
            Type:        original.EventType,
            AggregateID: streamID,
            Version:     int(original.EventNumber) + 1, // ESDB is 0-based.
            Data:        json.RawMessage(original.Data),
            Metadata:    json.RawMessage(original.UserMetadata),
            Timestamp:   original.CreatedDate,
        })
    }

    return events, nil
}

func (s *ESDBEventStore) Close() error {
    return s.client.Close()
}
```

---

## 3. Aggregates and Command Handling

### The Aggregate Pattern

An aggregate is the central pattern in event sourcing. It:

1. Receives **commands** (requests to do something).
2. Validates business rules against current state.
3. Produces **events** (facts about what happened).
4. Applies events to update its own state.

```go
package eventsourcing

// Aggregate is the interface that all event-sourced aggregates implement.
type Aggregate interface {
    // AggregateID returns the unique ID of this aggregate instance.
    AggregateID() string

    // AggregateType returns the type name, e.g. "Order".
    AggregateType() string

    // Version returns the current version (number of events applied so far).
    Version() int

    // ApplyEvent updates the aggregate's internal state based on an event.
    // This method must be a pure function: no side effects, no I/O.
    ApplyEvent(event Event) error

    // UncommittedEvents returns the events that have been produced but not yet persisted.
    UncommittedEvents() []Event

    // ClearUncommittedEvents clears the uncommitted events after persistence.
    ClearUncommittedEvents()
}
```

### Base Aggregate Implementation

A reusable base struct provides the boilerplate shared across all aggregates.

```go
package eventsourcing

import (
    "encoding/json"
    "time"

    "github.com/google/uuid"
)

// BaseAggregate provides shared aggregate functionality.
// Embed this in your concrete aggregate types.
type BaseAggregate struct {
    id                string
    aggregateType     string
    version           int
    uncommittedEvents []Event
}

func NewBaseAggregate(id, aggregateType string) BaseAggregate {
    return BaseAggregate{
        id:            id,
        aggregateType: aggregateType,
    }
}

func (a *BaseAggregate) AggregateID() string   { return a.id }
func (a *BaseAggregate) AggregateType() string  { return a.aggregateType }
func (a *BaseAggregate) Version() int           { return a.version }

func (a *BaseAggregate) UncommittedEvents() []Event {
    return a.uncommittedEvents
}

func (a *BaseAggregate) ClearUncommittedEvents() {
    a.uncommittedEvents = nil
}

// RaiseEvent creates a new event and appends it to the uncommitted list.
// The caller must also call ApplyEvent on the concrete aggregate to update state.
func (a *BaseAggregate) RaiseEvent(eventType string, data interface{}) (Event, error) {
    dataBytes, err := json.Marshal(data)
    if err != nil {
        return Event{}, err
    }

    event := Event{
        ID:            uuid.New().String(),
        Type:          eventType,
        AggregateType: a.aggregateType,
        AggregateID:   a.id,
        Version:       a.version + 1,
        Data:          dataBytes,
        Timestamp:     time.Now().UTC(),
    }

    a.version++
    a.uncommittedEvents = append(a.uncommittedEvents, event)
    return event, nil
}

// SetVersion sets the aggregate version (used when loading from events).
func (a *BaseAggregate) SetVersion(v int) {
    a.version = v
}
```

### A Concrete Aggregate: Order

```go
package order

import (
    "encoding/json"
    "errors"
    "fmt"
    "time"

    es "myapp/eventsourcing"
)

// --- Domain Events ---

type OrderPlaced struct {
    OrderID    string    `json:"order_id"`
    CustomerID string    `json:"customer_id"`
    PlacedAt   time.Time `json:"placed_at"`
}

type ItemAdded struct {
    ProductID string  `json:"product_id"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

type ItemRemoved struct {
    ProductID string `json:"product_id"`
}

type OrderConfirmed struct {
    ConfirmedAt time.Time `json:"confirmed_at"`
}

type OrderShipped struct {
    ShippedAt  time.Time `json:"shipped_at"`
    TrackingID string    `json:"tracking_id"`
}

type OrderCancelled struct {
    CancelledAt time.Time `json:"cancelled_at"`
    Reason      string    `json:"reason"`
}

// --- Order Aggregate ---

type Status string

const (
    StatusDraft     Status = "draft"
    StatusConfirmed Status = "confirmed"
    StatusShipped   Status = "shipped"
    StatusCancelled Status = "cancelled"
)

type LineItem struct {
    ProductID string
    Name      string
    Quantity  int
    Price     float64
}

// Order is an event-sourced aggregate representing a customer order.
type Order struct {
    es.BaseAggregate

    CustomerID string
    Status     Status
    Items      map[string]LineItem // keyed by ProductID
    PlacedAt   time.Time
}

func NewOrder(id string) *Order {
    return &Order{
        BaseAggregate: es.NewBaseAggregate(id, "Order"),
        Items:         make(map[string]LineItem),
    }
}

// --- Command Handlers (methods on the aggregate) ---

func (o *Order) Place(customerID string) error {
    if o.Status != "" {
        return errors.New("order has already been placed")
    }

    event, err := o.RaiseEvent("OrderPlaced", OrderPlaced{
        OrderID:    o.AggregateID(),
        CustomerID: customerID,
        PlacedAt:   time.Now().UTC(),
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) AddItem(productID, name string, quantity int, price float64) error {
    if o.Status != StatusDraft {
        return fmt.Errorf("cannot add items to order in status %s", o.Status)
    }
    if quantity <= 0 {
        return errors.New("quantity must be positive")
    }
    if price < 0 {
        return errors.New("price cannot be negative")
    }

    event, err := o.RaiseEvent("ItemAdded", ItemAdded{
        ProductID: productID,
        Name:      name,
        Quantity:  quantity,
        Price:     price,
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) RemoveItem(productID string) error {
    if o.Status != StatusDraft {
        return fmt.Errorf("cannot remove items from order in status %s", o.Status)
    }
    if _, ok := o.Items[productID]; !ok {
        return fmt.Errorf("product %s not in order", productID)
    }

    event, err := o.RaiseEvent("ItemRemoved", ItemRemoved{
        ProductID: productID,
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) Confirm() error {
    if o.Status != StatusDraft {
        return fmt.Errorf("cannot confirm order in status %s", o.Status)
    }
    if len(o.Items) == 0 {
        return errors.New("cannot confirm an empty order")
    }

    event, err := o.RaiseEvent("OrderConfirmed", OrderConfirmed{
        ConfirmedAt: time.Now().UTC(),
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) Ship(trackingID string) error {
    if o.Status != StatusConfirmed {
        return fmt.Errorf("cannot ship order in status %s", o.Status)
    }

    event, err := o.RaiseEvent("OrderShipped", OrderShipped{
        ShippedAt:  time.Now().UTC(),
        TrackingID: trackingID,
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) Cancel(reason string) error {
    if o.Status == StatusShipped {
        return errors.New("cannot cancel a shipped order")
    }
    if o.Status == StatusCancelled {
        return errors.New("order is already cancelled")
    }

    event, err := o.RaiseEvent("OrderCancelled", OrderCancelled{
        CancelledAt: time.Now().UTC(),
        Reason:      reason,
    })
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}

func (o *Order) TotalAmount() float64 {
    var total float64
    for _, item := range o.Items {
        total += float64(item.Quantity) * item.Price
    }
    return total
}

// --- Event Application ---

// ApplyEvent updates the Order's internal state based on a domain event.
// This must be a pure function with no side effects.
func (o *Order) ApplyEvent(event es.Event) error {
    switch event.Type {
    case "OrderPlaced":
        var data OrderPlaced
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        o.CustomerID = data.CustomerID
        o.Status = StatusDraft
        o.PlacedAt = data.PlacedAt

    case "ItemAdded":
        var data ItemAdded
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        o.Items[data.ProductID] = LineItem{
            ProductID: data.ProductID,
            Name:      data.Name,
            Quantity:  data.Quantity,
            Price:     data.Price,
        }

    case "ItemRemoved":
        var data ItemRemoved
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        delete(o.Items, data.ProductID)

    case "OrderConfirmed":
        o.Status = StatusConfirmed

    case "OrderShipped":
        o.Status = StatusShipped

    case "OrderCancelled":
        o.Status = StatusCancelled

    default:
        return fmt.Errorf("unknown event type: %s", event.Type)
    }

    o.SetVersion(event.Version)
    return nil
}
```

### The Repository Pattern

A repository abstracts the mechanics of loading and saving aggregates via the event store.

```go
package eventsourcing

import (
    "context"
    "fmt"
)

// AggregateFactory is a function that creates a new, empty aggregate instance.
type AggregateFactory func(id string) Aggregate

// Repository provides load/save operations for aggregates.
type Repository struct {
    store   EventStore
    factory AggregateFactory
}

func NewRepository(store EventStore, factory AggregateFactory) *Repository {
    return &Repository{
        store:   store,
        factory: factory,
    }
}

// Load reconstitutes an aggregate from its event history.
func (r *Repository) Load(ctx context.Context, aggregateID string) (Aggregate, error) {
    agg := r.factory(aggregateID)

    streamID := fmt.Sprintf("%s-%s", agg.AggregateType(), aggregateID)
    events, err := r.store.LoadEvents(ctx, streamID)
    if err != nil {
        return nil, fmt.Errorf("load events: %w", err)
    }

    for _, event := range events {
        if err := agg.ApplyEvent(event); err != nil {
            return nil, fmt.Errorf("apply event %s (v%d): %w", event.Type, event.Version, err)
        }
    }

    return agg, nil
}

// Save persists the aggregate's uncommitted events to the event store.
func (r *Repository) Save(ctx context.Context, agg Aggregate) error {
    uncommitted := agg.UncommittedEvents()
    if len(uncommitted) == 0 {
        return nil
    }

    streamID := fmt.Sprintf("%s-%s", agg.AggregateType(), agg.AggregateID())
    expectedVersion := agg.Version() - len(uncommitted)

    if err := r.store.AppendEvents(ctx, streamID, expectedVersion, uncommitted); err != nil {
        return fmt.Errorf("append events: %w", err)
    }

    agg.ClearUncommittedEvents()
    return nil
}
```

### Command Bus

A command bus dispatches commands to the appropriate handler, providing a clean entry point.

```go
package eventsourcing

import (
    "context"
    "fmt"
)

// Command is a marker interface for commands.
type Command interface {
    CommandName() string
}

// CommandHandler processes a command.
type CommandHandler func(ctx context.Context, cmd Command) error

// CommandBus routes commands to their registered handlers.
type CommandBus struct {
    handlers map[string]CommandHandler
}

func NewCommandBus() *CommandBus {
    return &CommandBus{
        handlers: make(map[string]CommandHandler),
    }
}

func (b *CommandBus) Register(commandName string, handler CommandHandler) {
    b.handlers[commandName] = handler
}

func (b *CommandBus) Dispatch(ctx context.Context, cmd Command) error {
    handler, ok := b.handlers[cmd.CommandName()]
    if !ok {
        return fmt.Errorf("no handler registered for command: %s", cmd.CommandName())
    }
    return handler(ctx, cmd)
}
```

**Example usage with the Order aggregate:**

```go
package order

import (
    "context"

    es "myapp/eventsourcing"
)

// --- Commands ---

type PlaceOrderCommand struct {
    OrderID    string
    CustomerID string
}

func (c PlaceOrderCommand) CommandName() string { return "PlaceOrder" }

type AddItemCommand struct {
    OrderID   string
    ProductID string
    Name      string
    Quantity  int
    Price     float64
}

func (c AddItemCommand) CommandName() string { return "AddItem" }

type ConfirmOrderCommand struct {
    OrderID string
}

func (c ConfirmOrderCommand) CommandName() string { return "ConfirmOrder" }

// --- Command Handler Registration ---

func RegisterCommandHandlers(bus *es.CommandBus, repo *es.Repository) {
    bus.Register("PlaceOrder", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(PlaceOrderCommand)
        order := NewOrder(c.OrderID)
        if err := order.Place(c.CustomerID); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })

    bus.Register("AddItem", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(AddItemCommand)
        agg, err := repo.Load(ctx, c.OrderID)
        if err != nil {
            return err
        }
        order := agg.(*Order)
        if err := order.AddItem(c.ProductID, c.Name, c.Quantity, c.Price); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })

    bus.Register("ConfirmOrder", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(ConfirmOrderCommand)
        agg, err := repo.Load(ctx, c.OrderID)
        if err != nil {
            return err
        }
        order := agg.(*Order)
        if err := order.Confirm(); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })
}
```

---

## 4. Event Versioning and Schema Evolution

### The Problem

Events are stored forever. Over months and years, the shape of your events will change: fields get added, removed, renamed, or their types change. You need a strategy to handle old event formats alongside new ones.

### Strategies Overview

| Strategy | Description | Complexity |
|----------|-------------|------------|
| **Weak schema** | Use `map[string]interface{}` and tolerate missing fields | Low |
| **Upcasting** | Transform old events to new format at read time | Medium |
| **Versioned types** | Use `OrderPlacedV1`, `OrderPlacedV2`, etc. | Medium |
| **Event migration** | Rewrite the entire event store (dangerous) | High |
| **Copy-transform** | Create new stream with transformed events | High |

### Upcasting: The Recommended Approach

Upcasting transforms an old event format into the current format at the moment you read it from the store. The original bytes are never modified -- the transformation happens in memory.

```go
package eventsourcing

import "encoding/json"

// Upcaster transforms an event from an older version to a newer version.
type Upcaster interface {
    // CanUpcast returns true if this upcaster handles the given event type and version.
    CanUpcast(eventType string, schemaVersion int) bool

    // Upcast transforms the event data from one schema version to the next.
    Upcast(eventType string, data json.RawMessage) (json.RawMessage, int, error)
}

// UpcasterChain applies upcasters in sequence until the event is at the latest version.
type UpcasterChain struct {
    upcasters []Upcaster
}

func NewUpcasterChain(upcasters ...Upcaster) *UpcasterChain {
    return &UpcasterChain{upcasters: upcasters}
}

func (c *UpcasterChain) Upcast(eventType string, data json.RawMessage, schemaVersion int) (json.RawMessage, error) {
    currentData := data
    currentVersion := schemaVersion

    for {
        upcasted := false
        for _, u := range c.upcasters {
            if u.CanUpcast(eventType, currentVersion) {
                var err error
                currentData, currentVersion, err = u.Upcast(eventType, currentData)
                if err != nil {
                    return nil, err
                }
                upcasted = true
                break
            }
        }
        if !upcasted {
            break // No more upcasters apply; we are at the latest version.
        }
    }
    return currentData, nil
}
```

### Concrete Upcaster Example

Suppose `OrderPlaced` originally had a flat `address` string, but we later split it into structured fields.

```go
package order

import (
    "encoding/json"

    es "myapp/eventsourcing"
)

// V1: { "customer_id": "c1", "address": "123 Main St, Springfield, IL 62704" }
// V2: { "customer_id": "c1", "shipping_address": { "street": "...", "city": "...", "state": "...", "zip": "..." } }

type OrderPlacedV1toV2Upcaster struct{}

func (u *OrderPlacedV1toV2Upcaster) CanUpcast(eventType string, version int) bool {
    return eventType == "OrderPlaced" && version == 1
}

func (u *OrderPlacedV1toV2Upcaster) Upcast(
    eventType string,
    data json.RawMessage,
) (json.RawMessage, int, error) {
    var v1 struct {
        CustomerID string `json:"customer_id"`
        Address    string `json:"address"`
    }
    if err := json.Unmarshal(data, &v1); err != nil {
        return nil, 0, err
    }

    // In production, you would parse the address string properly.
    v2 := map[string]interface{}{
        "customer_id": v1.CustomerID,
        "shipping_address": map[string]string{
            "street": v1.Address, // Simplified -- real code would parse.
            "city":   "",
            "state":  "",
            "zip":    "",
        },
    }

    result, err := json.Marshal(v2)
    if err != nil {
        return nil, 0, err
    }
    return result, 2, nil
}

// Register the upcaster chain.
func NewOrderUpcasterChain() *es.UpcasterChain {
    return es.NewUpcasterChain(
        &OrderPlacedV1toV2Upcaster{},
        // Add more upcasters here as the schema evolves:
        // &OrderPlacedV2toV3Upcaster{},
    )
}
```

### Versioned Event Types

An alternative is to use separate event types per version. The aggregate's `ApplyEvent` handles all versions.

```go
func (o *Order) ApplyEvent(event es.Event) error {
    switch event.Type {
    case "OrderPlaced":
        // Latest version.
        var data OrderPlaced
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        o.CustomerID = data.CustomerID
        o.Status = StatusDraft

    case "OrderPlacedV1":
        // Legacy format -- handle directly.
        var data struct {
            CustomerID string `json:"customer_id"`
            Address    string `json:"address"`
        }
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        o.CustomerID = data.CustomerID
        o.Status = StatusDraft
        // Legacy address field is ignored or stored differently.

    // ... other event types
    }
    return nil
}
```

> **Tip:** Upcasting is generally preferred over versioned types because it keeps the aggregate's `ApplyEvent` logic clean -- it only needs to handle the latest format.

---

## 5. Snapshots for Performance Optimization

### The Problem

When an aggregate has thousands of events, replaying them all on every load becomes expensive. Snapshots cache the aggregate state at a point in time so you only need to replay events after the snapshot.

### Snapshot Store

```go
package eventsourcing

import (
    "context"
    "encoding/json"
    "time"
)

// Snapshot represents a cached aggregate state.
type Snapshot struct {
    AggregateID   string          `json:"aggregate_id"`
    AggregateType string          `json:"aggregate_type"`
    Version       int             `json:"version"`
    State         json.RawMessage `json:"state"`
    CreatedAt     time.Time       `json:"created_at"`
}

// SnapshotStore persists and retrieves snapshots.
type SnapshotStore interface {
    SaveSnapshot(ctx context.Context, snapshot Snapshot) error
    LoadSnapshot(ctx context.Context, aggregateID string) (*Snapshot, error)
}

// Snapshottable is an interface for aggregates that support snapshotting.
type Snapshottable interface {
    Aggregate

    // ToSnapshot serializes the aggregate's current state.
    ToSnapshot() (json.RawMessage, error)

    // FromSnapshot restores the aggregate's state from a snapshot.
    FromSnapshot(data json.RawMessage) error
}
```

### In-Memory Snapshot Store

```go
package eventsourcing

import (
    "context"
    "sync"
)

type InMemorySnapshotStore struct {
    mu        sync.RWMutex
    snapshots map[string]Snapshot
}

func NewInMemorySnapshotStore() *InMemorySnapshotStore {
    return &InMemorySnapshotStore{
        snapshots: make(map[string]Snapshot),
    }
}

func (s *InMemorySnapshotStore) SaveSnapshot(ctx context.Context, snap Snapshot) error {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.snapshots[snap.AggregateID] = snap
    return nil
}

func (s *InMemorySnapshotStore) LoadSnapshot(ctx context.Context, id string) (*Snapshot, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    snap, ok := s.snapshots[id]
    if !ok {
        return nil, nil
    }
    return &snap, nil
}
```

### PostgreSQL Snapshot Store

```sql
CREATE TABLE snapshots (
    aggregate_id   TEXT PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    version        INT NOT NULL,
    state          JSONB NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```go
package eventsourcing

import (
    "context"
    "database/sql"
    "fmt"
)

type PostgresSnapshotStore struct {
    db *sql.DB
}

func NewPostgresSnapshotStore(db *sql.DB) *PostgresSnapshotStore {
    return &PostgresSnapshotStore{db: db}
}

func (s *PostgresSnapshotStore) SaveSnapshot(ctx context.Context, snap Snapshot) error {
    _, err := s.db.ExecContext(ctx, `
        INSERT INTO snapshots (aggregate_id, aggregate_type, version, state, created_at)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (aggregate_id) DO UPDATE SET
            version = EXCLUDED.version,
            state = EXCLUDED.state,
            created_at = EXCLUDED.created_at
    `, snap.AggregateID, snap.AggregateType, snap.Version, snap.State, snap.CreatedAt)
    if err != nil {
        return fmt.Errorf("save snapshot: %w", err)
    }
    return nil
}

func (s *PostgresSnapshotStore) LoadSnapshot(
    ctx context.Context,
    aggregateID string,
) (*Snapshot, error) {
    var snap Snapshot
    err := s.db.QueryRowContext(ctx, `
        SELECT aggregate_id, aggregate_type, version, state, created_at
        FROM snapshots
        WHERE aggregate_id = $1
    `, aggregateID).Scan(
        &snap.AggregateID,
        &snap.AggregateType,
        &snap.Version,
        &snap.State,
        &snap.CreatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("load snapshot: %w", err)
    }
    return &snap, nil
}
```

### Snapshot-Aware Repository

```go
package eventsourcing

import (
    "context"
    "fmt"
    "time"
)

// SnapshotRepository extends Repository with snapshot support.
type SnapshotRepository struct {
    eventStore    EventStore
    snapshotStore SnapshotStore
    factory       AggregateFactory
    snapshotEvery int // Take a snapshot every N events.
}

func NewSnapshotRepository(
    eventStore EventStore,
    snapshotStore SnapshotStore,
    factory AggregateFactory,
    snapshotEvery int,
) *SnapshotRepository {
    return &SnapshotRepository{
        eventStore:    eventStore,
        snapshotStore: snapshotStore,
        factory:       factory,
        snapshotEvery: snapshotEvery,
    }
}

func (r *SnapshotRepository) Load(ctx context.Context, aggregateID string) (Aggregate, error) {
    agg := r.factory(aggregateID)
    streamID := fmt.Sprintf("%s-%s", agg.AggregateType(), aggregateID)

    // Try to load from snapshot first.
    snap, err := r.snapshotStore.LoadSnapshot(ctx, aggregateID)
    if err != nil {
        return nil, fmt.Errorf("load snapshot: %w", err)
    }

    fromVersion := 1
    if snap != nil {
        if sa, ok := agg.(Snapshottable); ok {
            if err := sa.FromSnapshot(snap.State); err != nil {
                return nil, fmt.Errorf("restore from snapshot: %w", err)
            }
            fromVersion = snap.Version + 1
        }
    }

    // Replay events from after the snapshot.
    events, err := r.eventStore.LoadEventsFrom(ctx, streamID, fromVersion)
    if err != nil {
        return nil, fmt.Errorf("load events: %w", err)
    }

    for _, event := range events {
        if err := agg.ApplyEvent(event); err != nil {
            return nil, fmt.Errorf("apply event: %w", err)
        }
    }

    return agg, nil
}

func (r *SnapshotRepository) Save(ctx context.Context, agg Aggregate) error {
    uncommitted := agg.UncommittedEvents()
    if len(uncommitted) == 0 {
        return nil
    }

    streamID := fmt.Sprintf("%s-%s", agg.AggregateType(), agg.AggregateID())
    expectedVersion := agg.Version() - len(uncommitted)

    if err := r.eventStore.AppendEvents(ctx, streamID, expectedVersion, uncommitted); err != nil {
        return fmt.Errorf("append events: %w", err)
    }
    agg.ClearUncommittedEvents()

    // Take a snapshot if enough events have accumulated.
    if sa, ok := agg.(Snapshottable); ok && r.snapshotEvery > 0 {
        if agg.Version()%r.snapshotEvery == 0 {
            state, err := sa.ToSnapshot()
            if err != nil {
                // Log but do not fail -- snapshot is an optimization, not critical.
                return nil
            }
            _ = r.snapshotStore.SaveSnapshot(ctx, Snapshot{
                AggregateID:   agg.AggregateID(),
                AggregateType: agg.AggregateType(),
                Version:       agg.Version(),
                State:         state,
                CreatedAt:     time.Now().UTC(),
            })
        }
    }

    return nil
}
```

### Making the Order Aggregate Snapshottable

```go
package order

import "encoding/json"

// orderSnapshot is the serializable form of an Order's state.
type orderSnapshot struct {
    CustomerID string              `json:"customer_id"`
    Status     Status              `json:"status"`
    Items      map[string]LineItem `json:"items"`
    PlacedAt   string              `json:"placed_at"`
    Version    int                 `json:"version"`
}

func (o *Order) ToSnapshot() (json.RawMessage, error) {
    snap := orderSnapshot{
        CustomerID: o.CustomerID,
        Status:     o.Status,
        Items:      o.Items,
        PlacedAt:   o.PlacedAt.Format("2006-01-02T15:04:05Z"),
        Version:    o.Version(),
    }
    return json.Marshal(snap)
}

func (o *Order) FromSnapshot(data json.RawMessage) error {
    var snap orderSnapshot
    if err := json.Unmarshal(data, &snap); err != nil {
        return err
    }

    o.CustomerID = snap.CustomerID
    o.Status = snap.Status
    o.Items = snap.Items
    o.SetVersion(snap.Version)
    return nil
}
```

> **Warning:** Snapshot formats can also evolve over time. You may need snapshot versioning and migration just like you need event versioning. Keep this in mind before committing to a snapshot schema.

---

## 6. CQRS Pattern

### What Is CQRS?

**Command Query Responsibility Segregation** separates the models used for writing data (commands) from the models used for reading data (queries). In a traditional CRUD application, the same model handles both reads and writes. CQRS acknowledges that read and write workloads have fundamentally different requirements.

```
           Traditional (CRUD)                 CQRS
         ┌────────────────────┐     ┌────────────────────────────┐
         │     Application    │     │        Application         │
         │                    │     │                            │
         │   ┌────────────┐   │     │  ┌──────┐    ┌─────────┐  │
         │   │ Same Model │   │     │  │Write │    │  Read   │  │
         │   │            │   │     │  │Model │    │  Model  │  │
         │   │ Read+Write │   │     │  │      │    │         │  │
         │   └─────┬──────┘   │     │  └──┬───┘    └────┬────┘  │
         │         │          │     │     │             │       │
         │   ┌─────┴──────┐   │     │  ┌──┴───┐   ┌────┴────┐  │
         │   │  Database   │   │     │  │Event │   │  Read   │  │
         │   └─────────────┘   │     │  │Store │   │   DB    │  │
         └────────────────────┘     │  └──────┘   └─────────┘  │
                                    └────────────────────────────┘
```

### Why CQRS?

| Benefit | Explanation |
|---------|-------------|
| **Independent scaling** | Scale read and write sides independently. Reads often outnumber writes 100:1. |
| **Optimized read models** | Tailor read models to specific query needs -- denormalized, pre-computed. |
| **Simpler write model** | Write side focuses only on business rules, not on how data is queried. |
| **Independent technology** | Write side can use an event store; read side can use SQL, Elasticsearch, Redis. |
| **Better performance** | Reads do not contend with writes for locks or connections. |

### CQRS Without Event Sourcing

It is important to understand that CQRS and event sourcing are independent patterns. You can use CQRS without event sourcing, and vice versa. However, they complement each other extremely well because event sourcing naturally produces the stream of events that CQRS needs to update its read models.

---

## 7. Separate Read and Write Models

### Write Model

The write model is the event-sourced aggregate. It accepts commands, enforces invariants, and produces events. We already built this in Sections 3-5.

### Read Model

The read model is a denormalized, query-optimized view of the data. It is updated by processing events from the event store.

```go
package readmodel

import "time"

// OrderSummary is a read model optimized for listing orders.
// It is flat and denormalized -- fast to query, no joins.
type OrderSummary struct {
    OrderID    string    `json:"order_id"    db:"order_id"`
    CustomerID string    `json:"customer_id" db:"customer_id"`
    Status     string    `json:"status"      db:"status"`
    ItemCount  int       `json:"item_count"  db:"item_count"`
    TotalAmount float64  `json:"total_amount" db:"total_amount"`
    PlacedAt   time.Time `json:"placed_at"   db:"placed_at"`
    UpdatedAt  time.Time `json:"updated_at"  db:"updated_at"`
}

// OrderDetail is a read model optimized for viewing a single order.
type OrderDetail struct {
    OrderID    string          `json:"order_id"`
    CustomerID string          `json:"customer_id"`
    Status     string          `json:"status"`
    Items      []OrderItemView `json:"items"`
    TotalAmount float64        `json:"total_amount"`
    PlacedAt   time.Time       `json:"placed_at"`
    ConfirmedAt *time.Time     `json:"confirmed_at,omitempty"`
    ShippedAt   *time.Time     `json:"shipped_at,omitempty"`
    TrackingID  string         `json:"tracking_id,omitempty"`
}

// OrderItemView is a read model for individual order items.
type OrderItemView struct {
    ProductID string  `json:"product_id"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
    Subtotal  float64 `json:"subtotal"`
}

// CustomerOrderHistory is a read model for a customer's order history.
type CustomerOrderHistory struct {
    CustomerID  string         `json:"customer_id"`
    TotalOrders int            `json:"total_orders"`
    TotalSpent  float64        `json:"total_spent"`
    Orders      []OrderSummary `json:"orders"`
}
```

### Query Service

```go
package readmodel

import (
    "context"
    "database/sql"
    "fmt"
)

// QueryService provides read-only access to projections.
type QueryService struct {
    db *sql.DB
}

func NewQueryService(db *sql.DB) *QueryService {
    return &QueryService{db: db}
}

func (s *QueryService) GetOrderSummary(ctx context.Context, orderID string) (*OrderSummary, error) {
    var o OrderSummary
    err := s.db.QueryRowContext(ctx, `
        SELECT order_id, customer_id, status, item_count, total_amount, placed_at, updated_at
        FROM order_summaries
        WHERE order_id = $1
    `, orderID).Scan(
        &o.OrderID, &o.CustomerID, &o.Status,
        &o.ItemCount, &o.TotalAmount, &o.PlacedAt, &o.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("query order summary: %w", err)
    }
    return &o, nil
}

func (s *QueryService) ListOrdersByCustomer(
    ctx context.Context,
    customerID string,
    limit, offset int,
) ([]OrderSummary, error) {
    rows, err := s.db.QueryContext(ctx, `
        SELECT order_id, customer_id, status, item_count, total_amount, placed_at, updated_at
        FROM order_summaries
        WHERE customer_id = $1
        ORDER BY placed_at DESC
        LIMIT $2 OFFSET $3
    `, customerID, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("query orders: %w", err)
    }
    defer rows.Close()

    var orders []OrderSummary
    for rows.Next() {
        var o OrderSummary
        if err := rows.Scan(
            &o.OrderID, &o.CustomerID, &o.Status,
            &o.ItemCount, &o.TotalAmount, &o.PlacedAt, &o.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan order: %w", err)
        }
        orders = append(orders, o)
    }
    return orders, rows.Err()
}

func (s *QueryService) GetOrderDetail(ctx context.Context, orderID string) (*OrderDetail, error) {
    var o OrderDetail
    err := s.db.QueryRowContext(ctx, `
        SELECT order_id, customer_id, status, total_amount, placed_at,
               confirmed_at, shipped_at, tracking_id
        FROM order_details
        WHERE order_id = $1
    `, orderID).Scan(
        &o.OrderID, &o.CustomerID, &o.Status, &o.TotalAmount,
        &o.PlacedAt, &o.ConfirmedAt, &o.ShippedAt, &o.TrackingID,
    )
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("query order detail: %w", err)
    }

    // Load items.
    rows, err := s.db.QueryContext(ctx, `
        SELECT product_id, name, quantity, price, subtotal
        FROM order_detail_items
        WHERE order_id = $1
    `, orderID)
    if err != nil {
        return nil, fmt.Errorf("query items: %w", err)
    }
    defer rows.Close()

    for rows.Next() {
        var item OrderItemView
        if err := rows.Scan(&item.ProductID, &item.Name, &item.Quantity, &item.Price, &item.Subtotal); err != nil {
            return nil, fmt.Errorf("scan item: %w", err)
        }
        o.Items = append(o.Items, item)
    }

    return &o, rows.Err()
}
```

---

## 8. Building Projections

### Projection Fundamentals

A projection reads events and builds or updates a read model. It is the bridge between the event store (write side) and the read database (read side).

```go
package projection

import (
    "context"

    es "myapp/eventsourcing"
)

// Projection processes events to build read models.
type Projection interface {
    // Name returns a unique identifier for this projection.
    Name() string

    // Handle processes a single event.
    Handle(ctx context.Context, event es.Event) error
}
```

### Synchronous Projections

A synchronous projection updates the read model within the same transaction that appends events. This provides strong consistency but couples the write and read paths.

```go
package projection

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "time"

    es "myapp/eventsourcing"
    "myapp/order"
)

// OrderSummaryProjection builds the order_summaries read model.
type OrderSummaryProjection struct {
    db *sql.DB
}

func NewOrderSummaryProjection(db *sql.DB) *OrderSummaryProjection {
    return &OrderSummaryProjection{db: db}
}

func (p *OrderSummaryProjection) Name() string { return "order_summary" }

func (p *OrderSummaryProjection) Handle(ctx context.Context, event es.Event) error {
    switch event.Type {
    case "OrderPlaced":
        return p.handleOrderPlaced(ctx, event)
    case "ItemAdded":
        return p.handleItemAdded(ctx, event)
    case "ItemRemoved":
        return p.handleItemRemoved(ctx, event)
    case "OrderConfirmed":
        return p.handleStatusChange(ctx, event, "confirmed")
    case "OrderShipped":
        return p.handleStatusChange(ctx, event, "shipped")
    case "OrderCancelled":
        return p.handleStatusChange(ctx, event, "cancelled")
    default:
        return nil // Ignore events we do not care about.
    }
}

func (p *OrderSummaryProjection) handleOrderPlaced(ctx context.Context, event es.Event) error {
    var data order.OrderPlaced
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    _, err := p.db.ExecContext(ctx, `
        INSERT INTO order_summaries (order_id, customer_id, status, item_count, total_amount, placed_at, updated_at)
        VALUES ($1, $2, 'draft', 0, 0, $3, $4)
    `, data.OrderID, data.CustomerID, data.PlacedAt, time.Now().UTC())
    return err
}

func (p *OrderSummaryProjection) handleItemAdded(ctx context.Context, event es.Event) error {
    var data order.ItemAdded
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    _, err := p.db.ExecContext(ctx, `
        UPDATE order_summaries
        SET item_count = item_count + $1,
            total_amount = total_amount + ($1 * $2),
            updated_at = $3
        WHERE order_id = $4
    `, data.Quantity, data.Price, time.Now().UTC(), event.AggregateID)
    return err
}

func (p *OrderSummaryProjection) handleItemRemoved(ctx context.Context, event es.Event) error {
    // In a real system you would need to look up the item's quantity and price
    // to decrement correctly. Here we simplify for brevity.
    _, err := p.db.ExecContext(ctx, `
        UPDATE order_summaries
        SET item_count = item_count - 1,
            updated_at = $1
        WHERE order_id = $2
    `, time.Now().UTC(), event.AggregateID)
    return err
}

func (p *OrderSummaryProjection) handleStatusChange(
    ctx context.Context,
    event es.Event,
    status string,
) error {
    _, err := p.db.ExecContext(ctx, `
        UPDATE order_summaries
        SET status = $1, updated_at = $2
        WHERE order_id = $3
    `, status, time.Now().UTC(), event.AggregateID)
    return err
}
```

### Synchronous Projection Manager

The synchronous projection manager runs projections inside the event store transaction.

```go
package projection

import (
    "context"

    es "myapp/eventsourcing"
)

// SyncProjectionManager runs projections synchronously after events are appended.
type SyncProjectionManager struct {
    projections []Projection
    store       es.EventStore
}

func NewSyncProjectionManager(store es.EventStore, projections ...Projection) *SyncProjectionManager {
    return &SyncProjectionManager{
        projections: projections,
        store:       store,
    }
}

// ProjectEvents runs all projections for the given events.
// Call this immediately after appending events to the store.
func (m *SyncProjectionManager) ProjectEvents(ctx context.Context, events []es.Event) error {
    for _, event := range events {
        for _, p := range m.projections {
            if err := p.Handle(ctx, event); err != nil {
                return fmt.Errorf("projection %s failed on event %s: %w", p.Name(), event.ID, err)
            }
        }
    }
    return nil
}
```

### Asynchronous Projections

Asynchronous projections process events in the background, typically using a polling or subscription mechanism. They provide better write performance at the cost of eventual consistency.

```go
package projection

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    es "myapp/eventsourcing"
)

// AsyncProjectionEngine is a background worker that continuously processes
// events and feeds them to projections.
type AsyncProjectionEngine struct {
    store       es.EventStore
    checkpointDB *sql.DB
    projections []Projection
    pollInterval time.Duration
}

func NewAsyncProjectionEngine(
    store es.EventStore,
    checkpointDB *sql.DB,
    pollInterval time.Duration,
    projections ...Projection,
) *AsyncProjectionEngine {
    return &AsyncProjectionEngine{
        store:        store,
        checkpointDB: checkpointDB,
        projections:  projections,
        pollInterval: pollInterval,
    }
}

// Run starts the projection engine. It blocks until the context is cancelled.
func (e *AsyncProjectionEngine) Run(ctx context.Context) error {
    for _, p := range e.projections {
        go e.runProjection(ctx, p)
    }

    <-ctx.Done()
    return ctx.Err()
}

func (e *AsyncProjectionEngine) runProjection(ctx context.Context, p Projection) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }

        if err := e.processNewEvents(ctx, p); err != nil {
            log.Printf("projection %s error: %v", p.Name(), err)
            // Back off on error.
            time.Sleep(5 * time.Second)
            continue
        }

        // Poll interval when no new events.
        time.Sleep(e.pollInterval)
    }
}

func (e *AsyncProjectionEngine) processNewEvents(ctx context.Context, p Projection) error {
    checkpoint, err := e.loadCheckpoint(ctx, p.Name())
    if err != nil {
        return fmt.Errorf("load checkpoint: %w", err)
    }

    events, err := e.store.LoadAllEvents(ctx, checkpoint)
    if err != nil {
        return fmt.Errorf("load events: %w", err)
    }

    for _, event := range events {
        if err := p.Handle(ctx, event); err != nil {
            return fmt.Errorf("handle event %s: %w", event.ID, err)
        }

        // Save checkpoint after each event (for at-least-once delivery).
        // For better performance, batch checkpoint updates.
        if err := e.saveCheckpoint(ctx, p.Name(), checkpoint+1); err != nil {
            return fmt.Errorf("save checkpoint: %w", err)
        }
        checkpoint++
    }

    return nil
}

func (e *AsyncProjectionEngine) loadCheckpoint(ctx context.Context, name string) (int64, error) {
    var pos int64
    err := e.checkpointDB.QueryRowContext(ctx,
        "SELECT position FROM projection_checkpoints WHERE name = $1", name,
    ).Scan(&pos)
    if err == sql.ErrNoRows {
        return 0, nil
    }
    return pos, err
}

func (e *AsyncProjectionEngine) saveCheckpoint(ctx context.Context, name string, position int64) error {
    _, err := e.checkpointDB.ExecContext(ctx, `
        INSERT INTO projection_checkpoints (name, position, updated_at)
        VALUES ($1, $2, NOW())
        ON CONFLICT (name) DO UPDATE SET position = EXCLUDED.position, updated_at = NOW()
    `, name, position)
    return err
}
```

### SQL Schema for Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    name       TEXT PRIMARY KEY,
    position   BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Projection Rebuild

One of the biggest advantages of event sourcing is the ability to rebuild projections from scratch. This is useful when you fix a bug in a projection, add a new field, or create an entirely new read model.

```go
// Rebuild deletes the existing read model and replays all events through the projection.
func (e *AsyncProjectionEngine) Rebuild(ctx context.Context, p Projection) error {
    log.Printf("Rebuilding projection: %s", p.Name())

    // Reset checkpoint to 0.
    if err := e.saveCheckpoint(ctx, p.Name(), 0); err != nil {
        return fmt.Errorf("reset checkpoint: %w", err)
    }

    // Process all events from the beginning.
    return e.processNewEvents(ctx, p)
}
```

> **Tip:** In production, you would typically rebuild into a new table (e.g., `order_summaries_v2`), then swap the table name atomically. This avoids downtime and allows the old read model to continue serving queries during the rebuild.

### Synchronous vs Asynchronous: When to Use Which

| Criteria | Synchronous | Asynchronous |
|----------|-------------|--------------|
| **Consistency** | Strong (read-after-write) | Eventual |
| **Write latency** | Higher (projection overhead) | Lower |
| **Failure isolation** | Projection failure blocks writes | Projection failure does not affect writes |
| **Complexity** | Lower | Higher (checkpoints, polling, error handling) |
| **Scalability** | Limited by write transaction | Independent scaling |
| **Best for** | Critical read models that must be immediately consistent | Analytics, search indexes, notifications |

---

## 9. Eventual Consistency Handling

### Understanding the Gap

With asynchronous projections, there is a time window between when an event is written and when the read model reflects it. This is the "consistency gap" and you must design your system to handle it.

### Strategies

#### 1. Causal Consistency via Version Checks

The write side returns the new version number. The client includes it in subsequent read requests, and the read side waits until it has caught up.

```go
package readmodel

import (
    "context"
    "fmt"
    "time"
)

// VersionAwareQueryService waits for the read model to catch up before returning.
type VersionAwareQueryService struct {
    inner     *QueryService
    checkpointFn func(ctx context.Context) (int64, error)
}

func NewVersionAwareQueryService(
    inner *QueryService,
    checkpointFn func(ctx context.Context) (int64, error),
) *VersionAwareQueryService {
    return &VersionAwareQueryService{
        inner:        inner,
        checkpointFn: checkpointFn,
    }
}

// WaitForVersion blocks until the read model has processed events up to the given position.
func (s *VersionAwareQueryService) WaitForVersion(
    ctx context.Context,
    requiredPosition int64,
    timeout time.Duration,
) error {
    deadline := time.Now().Add(timeout)

    for time.Now().Before(deadline) {
        current, err := s.checkpointFn(ctx)
        if err != nil {
            return fmt.Errorf("check position: %w", err)
        }
        if current >= requiredPosition {
            return nil
        }
        // Brief pause before checking again.
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(50 * time.Millisecond):
        }
    }

    return fmt.Errorf("read model did not catch up to position %d within %v", requiredPosition, timeout)
}
```

#### 2. Command Response with Created Resource

After a write, return the relevant data directly from the command handler instead of forcing a read.

```go
package api

import (
    "encoding/json"
    "net/http"
)

// CommandResult carries data from the write side back to the API caller
// so they do not need to immediately read from the eventually-consistent read model.
type CommandResult struct {
    AggregateID string `json:"aggregate_id"`
    Version     int    `json:"version"`
    Status      string `json:"status"`
}

func handlePlaceOrder(w http.ResponseWriter, r *http.Request) {
    // ... parse request, dispatch command ...

    // Return enough data so the client can render the UI without reading.
    result := CommandResult{
        AggregateID: orderID,
        Version:     order.Version(),
        Status:      "draft",
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(result)
}
```

#### 3. UI Optimistic Updates

The frontend optimistically updates its local state after a successful command, without waiting for the read model. On the next full page load, the read model will have caught up.

```
Client                 Server
  |                      |
  |--- PlaceOrder ------>|
  |                      |--- Append events
  |<-- 201 Created ------|
  |                      |--- (async) update read model
  |--- (optimistic UI    |
  |    update locally)   |
  |                      |
  |--- GET /orders ----->|  (later, read model is consistent)
  |<-- [order list] -----|
```

---

## 10. Saga Pattern for Distributed Transactions

### The Problem

In a distributed system, a single business operation often spans multiple aggregates or services. Traditional database transactions cannot span service boundaries. The saga pattern coordinates these multi-step processes.

### Saga Fundamentals

A saga is a sequence of local transactions. Each step either succeeds and triggers the next step, or fails and triggers compensating actions to undo previous steps.

```go
package saga

import (
    "context"
    "fmt"

    es "myapp/eventsourcing"
)

// SagaStep represents a single step in a saga.
type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context, data map[string]interface{}) error
    Compensate func(ctx context.Context, data map[string]interface{}) error
}

// Saga orchestrates a multi-step business process with compensation.
type Saga struct {
    name  string
    steps []SagaStep
}

func NewSaga(name string) *Saga {
    return &Saga{name: name}
}

func (s *Saga) AddStep(step SagaStep) {
    s.steps = append(s.steps, step)
}

// Execute runs the saga. If any step fails, it compensates all previously
// completed steps in reverse order.
func (s *Saga) Execute(ctx context.Context, data map[string]interface{}) error {
    completedSteps := make([]int, 0, len(s.steps))

    for i, step := range s.steps {
        if err := step.Execute(ctx, data); err != nil {
            // Step failed. Compensate in reverse order.
            compensationErr := s.compensate(ctx, completedSteps, data)
            if compensationErr != nil {
                return fmt.Errorf(
                    "saga %s: step %s failed (%w), compensation also failed: %v",
                    s.name, step.Name, err, compensationErr,
                )
            }
            return fmt.Errorf("saga %s: step %s failed (compensated): %w", s.name, step.Name, err)
        }
        completedSteps = append(completedSteps, i)
    }

    return nil
}

func (s *Saga) compensate(ctx context.Context, completedSteps []int, data map[string]interface{}) error {
    var errs []error
    // Compensate in reverse order.
    for i := len(completedSteps) - 1; i >= 0; i-- {
        step := s.steps[completedSteps[i]]
        if step.Compensate != nil {
            if err := step.Compensate(ctx, data); err != nil {
                errs = append(errs, fmt.Errorf("compensate %s: %w", step.Name, err))
            }
        }
    }
    if len(errs) > 0 {
        return fmt.Errorf("compensation errors: %v", errs)
    }
    return nil
}
```

### Order Fulfillment Saga

A concrete example: placing an order requires reserving inventory, charging payment, and sending a confirmation email.

```go
package saga

import (
    "context"
    "fmt"
)

// Services that the saga interacts with.
type InventoryService interface {
    Reserve(ctx context.Context, orderID string, items []OrderItem) error
    Release(ctx context.Context, orderID string) error
}

type PaymentService interface {
    Charge(ctx context.Context, orderID, customerID string, amount float64) (string, error)
    Refund(ctx context.Context, transactionID string) error
}

type NotificationService interface {
    SendOrderConfirmation(ctx context.Context, orderID, customerID string) error
}

type OrderItem struct {
    ProductID string
    Quantity  int
}

// NewOrderFulfillmentSaga creates the saga for order fulfillment.
func NewOrderFulfillmentSaga(
    inventory InventoryService,
    payment PaymentService,
    notification NotificationService,
) *Saga {
    saga := NewSaga("OrderFulfillment")

    saga.AddStep(SagaStep{
        Name: "ReserveInventory",
        Execute: func(ctx context.Context, data map[string]interface{}) error {
            orderID := data["order_id"].(string)
            items := data["items"].([]OrderItem)
            return inventory.Reserve(ctx, orderID, items)
        },
        Compensate: func(ctx context.Context, data map[string]interface{}) error {
            orderID := data["order_id"].(string)
            return inventory.Release(ctx, orderID)
        },
    })

    saga.AddStep(SagaStep{
        Name: "ChargePayment",
        Execute: func(ctx context.Context, data map[string]interface{}) error {
            orderID := data["order_id"].(string)
            customerID := data["customer_id"].(string)
            amount := data["amount"].(float64)
            txID, err := payment.Charge(ctx, orderID, customerID, amount)
            if err != nil {
                return err
            }
            data["transaction_id"] = txID // Pass to later steps and compensation.
            return nil
        },
        Compensate: func(ctx context.Context, data map[string]interface{}) error {
            txID, ok := data["transaction_id"].(string)
            if !ok {
                return nil // No transaction to refund.
            }
            return payment.Refund(ctx, txID)
        },
    })

    saga.AddStep(SagaStep{
        Name: "SendConfirmation",
        Execute: func(ctx context.Context, data map[string]interface{}) error {
            orderID := data["order_id"].(string)
            customerID := data["customer_id"].(string)
            return notification.SendOrderConfirmation(ctx, orderID, customerID)
        },
        Compensate: nil, // Email cannot be unsent; no compensation needed.
    })

    return saga
}
```

### Event-Driven Saga (Persistent State)

For long-running sagas that must survive process restarts, persist the saga state as events.

```go
package saga

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    es "myapp/eventsourcing"
)

// SagaState tracks the progress of a saga instance.
type SagaState struct {
    es.BaseAggregate

    SagaType       string                 `json:"saga_type"`
    Status         string                 `json:"status"` // "running", "completed", "compensating", "failed"
    CurrentStep    int                    `json:"current_step"`
    Data           map[string]interface{} `json:"data"`
    CompletedSteps []string               `json:"completed_steps"`
    StartedAt      time.Time              `json:"started_at"`
}

// Saga events.
type SagaStarted struct {
    SagaType string                 `json:"saga_type"`
    Data     map[string]interface{} `json:"data"`
}

type SagaStepCompleted struct {
    StepName string `json:"step_name"`
    StepIndex int   `json:"step_index"`
}

type SagaStepFailed struct {
    StepName string `json:"step_name"`
    Error    string `json:"error"`
}

type SagaCompleted struct{}

type SagaCompensated struct{}

func NewSagaState(id string) *SagaState {
    return &SagaState{
        BaseAggregate: es.NewBaseAggregate(id, "Saga"),
        Data:          make(map[string]interface{}),
    }
}

func (s *SagaState) ApplyEvent(event es.Event) error {
    switch event.Type {
    case "SagaStarted":
        var data SagaStarted
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        s.SagaType = data.SagaType
        s.Status = "running"
        s.Data = data.Data
        s.StartedAt = event.Timestamp

    case "SagaStepCompleted":
        var data SagaStepCompleted
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        s.CompletedSteps = append(s.CompletedSteps, data.StepName)
        s.CurrentStep = data.StepIndex + 1

    case "SagaStepFailed":
        s.Status = "compensating"

    case "SagaCompleted":
        s.Status = "completed"

    case "SagaCompensated":
        s.Status = "failed"
    }

    s.SetVersion(event.Version)
    return nil
}
```

---

## 11. Process Managers: Choreography vs Orchestration

### Choreography

In choreography, there is no central coordinator. Each service listens for events and decides what to do next. The "process" emerges from the collective behavior.

```
OrderPlaced ─────> Inventory Service ─────> InventoryReserved
                                                    │
                                                    v
                   Payment Service <──── (listens)
                          │
                          v
                   PaymentCharged ────> Notification Service
                                                    │
                                                    v
                                           EmailSent
```

```go
package choreography

import (
    "context"
    "encoding/json"
    "log"

    es "myapp/eventsourcing"
)

// InventoryEventHandler reacts to order events.
type InventoryEventHandler struct {
    inventoryService InventoryService
    eventPublisher   EventPublisher
}

func (h *InventoryEventHandler) Handle(ctx context.Context, event es.Event) error {
    switch event.Type {
    case "OrderConfirmed":
        var data struct {
            OrderID string `json:"order_id"`
        }
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }

        // Reserve inventory and publish the result.
        if err := h.inventoryService.Reserve(ctx, data.OrderID); err != nil {
            return h.eventPublisher.Publish(ctx, "InventoryReservationFailed", map[string]interface{}{
                "order_id": data.OrderID,
                "reason":   err.Error(),
            })
        }

        return h.eventPublisher.Publish(ctx, "InventoryReserved", map[string]interface{}{
            "order_id": data.OrderID,
        })

    case "OrderCancelled":
        var data struct {
            OrderID string `json:"order_id"`
        }
        if err := json.Unmarshal(event.Data, &data); err != nil {
            return err
        }
        return h.inventoryService.Release(ctx, data.OrderID)
    }
    return nil
}

type EventPublisher interface {
    Publish(ctx context.Context, eventType string, data interface{}) error
}
```

### Orchestration

In orchestration, a central **process manager** coordinates the steps, telling each service what to do and reacting to their responses.

```go
package orchestration

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    es "myapp/eventsourcing"
)

// OrderFulfillmentProcessManager orchestrates the order fulfillment process.
type OrderFulfillmentProcessManager struct {
    repo             *es.Repository
    commandBus       *es.CommandBus
    inventoryService InventoryService
    paymentService   PaymentService
}

// Handle reacts to events and sends commands to advance the process.
func (pm *OrderFulfillmentProcessManager) Handle(ctx context.Context, event es.Event) error {
    switch event.Type {
    case "OrderConfirmed":
        return pm.onOrderConfirmed(ctx, event)
    case "InventoryReserved":
        return pm.onInventoryReserved(ctx, event)
    case "InventoryReservationFailed":
        return pm.onInventoryFailed(ctx, event)
    case "PaymentCharged":
        return pm.onPaymentCharged(ctx, event)
    case "PaymentFailed":
        return pm.onPaymentFailed(ctx, event)
    }
    return nil
}

func (pm *OrderFulfillmentProcessManager) onOrderConfirmed(ctx context.Context, event es.Event) error {
    var data struct {
        OrderID    string  `json:"order_id"`
        CustomerID string  `json:"customer_id"`
        Amount     float64 `json:"amount"`
    }
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    log.Printf("[ProcessManager] Order %s confirmed, reserving inventory", data.OrderID)
    return pm.inventoryService.Reserve(ctx, data.OrderID)
}

func (pm *OrderFulfillmentProcessManager) onInventoryReserved(ctx context.Context, event es.Event) error {
    var data struct {
        OrderID string `json:"order_id"`
    }
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    log.Printf("[ProcessManager] Inventory reserved for order %s, charging payment", data.OrderID)

    // Look up order to get payment details.
    agg, err := pm.repo.Load(ctx, data.OrderID)
    if err != nil {
        return err
    }
    // Charge payment...
    _ = agg
    return nil
}

func (pm *OrderFulfillmentProcessManager) onInventoryFailed(ctx context.Context, event es.Event) error {
    var data struct {
        OrderID string `json:"order_id"`
        Reason  string `json:"reason"`
    }
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    log.Printf("[ProcessManager] Inventory failed for order %s: %s, cancelling", data.OrderID, data.Reason)
    return pm.commandBus.Dispatch(ctx, CancelOrderCommand{
        OrderID: data.OrderID,
        Reason:  fmt.Sprintf("inventory unavailable: %s", data.Reason),
    })
}

func (pm *OrderFulfillmentProcessManager) onPaymentCharged(ctx context.Context, event es.Event) error {
    var data struct {
        OrderID string `json:"order_id"`
    }
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    log.Printf("[ProcessManager] Payment charged for order %s, fulfillment complete", data.OrderID)
    return nil // Order is ready to ship.
}

func (pm *OrderFulfillmentProcessManager) onPaymentFailed(ctx context.Context, event es.Event) error {
    var data struct {
        OrderID string `json:"order_id"`
        Reason  string `json:"reason"`
    }
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return err
    }

    log.Printf("[ProcessManager] Payment failed for order %s, releasing inventory", data.OrderID)
    // Release inventory, then cancel order.
    if err := pm.inventoryService.Release(ctx, data.OrderID); err != nil {
        log.Printf("[ProcessManager] WARNING: failed to release inventory for %s: %v", data.OrderID, err)
    }
    return pm.commandBus.Dispatch(ctx, CancelOrderCommand{
        OrderID: data.OrderID,
        Reason:  fmt.Sprintf("payment failed: %s", data.Reason),
    })
}

type CancelOrderCommand struct {
    OrderID string
    Reason  string
}

func (c CancelOrderCommand) CommandName() string { return "CancelOrder" }

type InventoryService interface {
    Reserve(ctx context.Context, orderID string) error
    Release(ctx context.Context, orderID string) error
}
```

### Comparison

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Coupling** | Loose (services only know events) | Tighter (process manager knows all steps) |
| **Visibility** | Hard to see the full process | Full process visible in one place |
| **Complexity** | Distributed complexity | Centralized complexity |
| **Debugging** | Harder (follow events across services) | Easier (single place to inspect) |
| **Change impact** | Adding a step = new subscriber | Adding a step = modify process manager |
| **Best for** | Simple flows, few steps | Complex flows, many conditional branches |

> **Tip:** In practice, most production systems use a mix. Simple reactive behaviors use choreography; complex multi-step processes use orchestration.

---

## 12. Outbox Pattern for Reliable Event Publishing

### The Problem

After appending events to the event store, you need to publish them to a message broker (Kafka, RabbitMQ, NATS). A naive approach -- write to DB then publish to broker -- can fail between the two operations, leading to lost events.

```
1. BEGIN TRANSACTION
2.   INSERT events into event_store     -- succeeds
3. COMMIT
4. Publish to Kafka                     -- FAILS! (network error)
   --> Events saved but never published
```

### The Solution: Transactional Outbox

Write events to both the event store and an outbox table in the same database transaction. A separate relay process polls the outbox and publishes to the broker.

```
1. BEGIN TRANSACTION
2.   INSERT events into event_store
3.   INSERT events into outbox_table
4. COMMIT
5. (background) Relay polls outbox, publishes to Kafka, marks as published
```

### Schema

```sql
CREATE TABLE outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL,
    event_type      TEXT NOT NULL,
    aggregate_type  TEXT NOT NULL,
    aggregate_id    TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at    TIMESTAMPTZ,      -- NULL until successfully published.
    retry_count     INT NOT NULL DEFAULT 0,
    next_retry_at   TIMESTAMPTZ
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at)
    WHERE published_at IS NULL;
```

### Outbox Writer

```go
package outbox

import (
    "context"
    "database/sql"
    "fmt"

    es "myapp/eventsourcing"
)

// Writer inserts events into the outbox within the same transaction
// that appends them to the event store.
type Writer struct {
    db *sql.DB
}

func NewWriter(db *sql.DB) *Writer {
    return &Writer{db: db}
}

// WriteInTx inserts events into the outbox table using the provided transaction.
func (w *Writer) WriteInTx(ctx context.Context, tx *sql.Tx, events []es.Event) error {
    stmt, err := tx.PrepareContext(ctx, `
        INSERT INTO outbox (event_id, event_type, aggregate_type, aggregate_id, payload, metadata)
        VALUES ($1, $2, $3, $4, $5, $6)
    `)
    if err != nil {
        return fmt.Errorf("prepare outbox insert: %w", err)
    }
    defer stmt.Close()

    for _, event := range events {
        _, err := stmt.ExecContext(ctx,
            event.ID,
            event.Type,
            event.AggregateType,
            event.AggregateID,
            event.Data,
            event.Metadata,
        )
        if err != nil {
            return fmt.Errorf("insert outbox event: %w", err)
        }
    }
    return nil
}
```

### Outbox Relay (Publisher)

```go
package outbox

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "time"
)

// MessagePublisher is the interface for publishing events to an external broker.
type MessagePublisher interface {
    Publish(ctx context.Context, topic string, key string, value []byte) error
}

// Relay polls the outbox table and publishes unpublished events to the broker.
type Relay struct {
    db            *sql.DB
    publisher     MessagePublisher
    pollInterval  time.Duration
    batchSize     int
    maxRetries    int
}

func NewRelay(
    db *sql.DB,
    publisher MessagePublisher,
    pollInterval time.Duration,
    batchSize int,
) *Relay {
    return &Relay{
        db:           db,
        publisher:    publisher,
        pollInterval: pollInterval,
        batchSize:    batchSize,
        maxRetries:   5,
    }
}

// Run starts the relay loop. Blocks until context is cancelled.
func (r *Relay) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        published, err := r.publishBatch(ctx)
        if err != nil {
            log.Printf("outbox relay error: %v", err)
            time.Sleep(5 * time.Second)
            continue
        }

        if published == 0 {
            // No messages to publish; sleep before polling again.
            time.Sleep(r.pollInterval)
        }
    }
}

func (r *Relay) publishBatch(ctx context.Context) (int, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT id, event_id, event_type, aggregate_type, aggregate_id, payload, metadata
        FROM outbox
        WHERE published_at IS NULL
          AND (next_retry_at IS NULL OR next_retry_at <= NOW())
          AND retry_count < $1
        ORDER BY id ASC
        LIMIT $2
        FOR UPDATE SKIP LOCKED
    `, r.maxRetries, r.batchSize)
    if err != nil {
        return 0, fmt.Errorf("query outbox: %w", err)
    }
    defer rows.Close()

    count := 0
    for rows.Next() {
        var (
            id            int64
            eventID       string
            eventType     string
            aggregateType string
            aggregateID   string
            payload       json.RawMessage
            metadata      json.RawMessage
        )

        if err := rows.Scan(&id, &eventID, &eventType, &aggregateType, &aggregateID, &payload, &metadata); err != nil {
            return count, fmt.Errorf("scan outbox row: %w", err)
        }

        message, _ := json.Marshal(map[string]interface{}{
            "event_id":       eventID,
            "event_type":     eventType,
            "aggregate_type": aggregateType,
            "aggregate_id":   aggregateID,
            "data":           payload,
            "metadata":       metadata,
        })

        topic := fmt.Sprintf("events.%s", aggregateType)
        if err := r.publisher.Publish(ctx, topic, aggregateID, message); err != nil {
            // Mark for retry with exponential backoff.
            r.markForRetry(ctx, id)
            log.Printf("failed to publish outbox event %d: %v", id, err)
            continue
        }

        // Mark as published.
        if err := r.markPublished(ctx, id); err != nil {
            log.Printf("failed to mark outbox event %d as published: %v", id, err)
        }
        count++
    }

    return count, rows.Err()
}

func (r *Relay) markPublished(ctx context.Context, id int64) error {
    _, err := r.db.ExecContext(ctx,
        "UPDATE outbox SET published_at = NOW() WHERE id = $1", id,
    )
    return err
}

func (r *Relay) markForRetry(ctx context.Context, id int64) {
    _, _ = r.db.ExecContext(ctx, `
        UPDATE outbox
        SET retry_count = retry_count + 1,
            next_retry_at = NOW() + (INTERVAL '1 second' * POWER(2, retry_count))
        WHERE id = $1
    `, id)
}
```

### Outbox Cleanup

Old published events should be cleaned up periodically.

```go
// CleanupPublished removes outbox entries that were published more than `olderThan` ago.
func (r *Relay) CleanupPublished(ctx context.Context, olderThan time.Duration) (int64, error) {
    result, err := r.db.ExecContext(ctx, `
        DELETE FROM outbox
        WHERE published_at IS NOT NULL
          AND published_at < NOW() - $1::interval
    `, olderThan.String())
    if err != nil {
        return 0, err
    }
    return result.RowsAffected()
}
```

> **Warning:** The outbox pattern guarantees **at-least-once** delivery. Consumers must be idempotent -- processing the same event twice must produce the same result.

---

## 13. Complete Example: E-Commerce Order System

This section ties everything together into a cohesive e-commerce system.

### Domain Events

```go
package domain

import "time"

// All domain events for the order system.

type OrderCreated struct {
    OrderID    string `json:"order_id"`
    CustomerID string `json:"customer_id"`
    CreatedAt  time.Time `json:"created_at"`
}

type OrderItemAdded struct {
    OrderID   string  `json:"order_id"`
    ProductID string  `json:"product_id"`
    SKU       string  `json:"sku"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    UnitPrice float64 `json:"unit_price"`
}

type OrderItemQuantityChanged struct {
    OrderID   string `json:"order_id"`
    ProductID string `json:"product_id"`
    OldQty    int    `json:"old_qty"`
    NewQty    int    `json:"new_qty"`
}

type OrderItemRemoved struct {
    OrderID   string `json:"order_id"`
    ProductID string `json:"product_id"`
}

type ShippingAddressSet struct {
    OrderID string `json:"order_id"`
    Street  string `json:"street"`
    City    string `json:"city"`
    State   string `json:"state"`
    ZipCode string `json:"zip_code"`
    Country string `json:"country"`
}

type OrderSubmitted struct {
    OrderID     string    `json:"order_id"`
    SubmittedAt time.Time `json:"submitted_at"`
    TotalAmount float64   `json:"total_amount"`
}

type PaymentAuthorized struct {
    OrderID       string  `json:"order_id"`
    TransactionID string  `json:"transaction_id"`
    Amount        float64 `json:"amount"`
}

type PaymentDeclined struct {
    OrderID string `json:"order_id"`
    Reason  string `json:"reason"`
}

type OrderApproved struct {
    OrderID    string    `json:"order_id"`
    ApprovedAt time.Time `json:"approved_at"`
}

type OrderShipped struct {
    OrderID    string    `json:"order_id"`
    TrackingID string    `json:"tracking_id"`
    Carrier    string    `json:"carrier"`
    ShippedAt  time.Time `json:"shipped_at"`
}

type OrderDelivered struct {
    OrderID     string    `json:"order_id"`
    DeliveredAt time.Time `json:"delivered_at"`
}

type OrderCancelled struct {
    OrderID     string    `json:"order_id"`
    CancelledAt time.Time `json:"cancelled_at"`
    Reason      string    `json:"reason"`
}

type OrderRefunded struct {
    OrderID       string    `json:"order_id"`
    TransactionID string    `json:"transaction_id"`
    Amount        float64   `json:"amount"`
    RefundedAt    time.Time `json:"refunded_at"`
}
```

### The Order Aggregate (Complete)

```go
package domain

import (
    "encoding/json"
    "errors"
    "fmt"
    "time"

    es "myapp/eventsourcing"
)

type OrderStatus string

const (
    OrderStatusDraft     OrderStatus = "draft"
    OrderStatusSubmitted OrderStatus = "submitted"
    OrderStatusApproved  OrderStatus = "approved"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

type Address struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    State   string `json:"state"`
    ZipCode string `json:"zip_code"`
    Country string `json:"country"`
}

type Item struct {
    ProductID string  `json:"product_id"`
    SKU       string  `json:"sku"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    UnitPrice float64 `json:"unit_price"`
}

type Order struct {
    es.BaseAggregate

    CustomerID      string
    Status          OrderStatus
    Items           map[string]*Item
    ShippingAddress *Address
    TransactionID   string
    TrackingID      string
    Carrier         string
    CreatedAt       time.Time
    SubmittedAt     *time.Time
    ShippedAt       *time.Time
}

func NewOrder(id string) *Order {
    return &Order{
        BaseAggregate: es.NewBaseAggregate(id, "Order"),
        Items:         make(map[string]*Item),
    }
}

// --- Commands ---

func (o *Order) Create(customerID string) error {
    if o.Status != "" {
        return errors.New("order already exists")
    }
    return o.raise("OrderCreated", OrderCreated{
        OrderID:    o.AggregateID(),
        CustomerID: customerID,
        CreatedAt:  time.Now().UTC(),
    })
}

func (o *Order) AddItem(productID, sku, name string, quantity int, unitPrice float64) error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot modify order in status %s", o.Status)
    }
    if quantity <= 0 {
        return errors.New("quantity must be positive")
    }
    if unitPrice < 0 {
        return errors.New("unit price cannot be negative")
    }
    if _, exists := o.Items[productID]; exists {
        return fmt.Errorf("product %s already in order, use ChangeItemQuantity", productID)
    }
    return o.raise("OrderItemAdded", OrderItemAdded{
        OrderID:   o.AggregateID(),
        ProductID: productID,
        SKU:       sku,
        Name:      name,
        Quantity:  quantity,
        UnitPrice: unitPrice,
    })
}

func (o *Order) ChangeItemQuantity(productID string, newQty int) error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot modify order in status %s", o.Status)
    }
    item, exists := o.Items[productID]
    if !exists {
        return fmt.Errorf("product %s not in order", productID)
    }
    if newQty <= 0 {
        return errors.New("use RemoveItem to remove, quantity must be positive")
    }
    if item.Quantity == newQty {
        return nil // No change, no event.
    }
    return o.raise("OrderItemQuantityChanged", OrderItemQuantityChanged{
        OrderID:   o.AggregateID(),
        ProductID: productID,
        OldQty:    item.Quantity,
        NewQty:    newQty,
    })
}

func (o *Order) RemoveItem(productID string) error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot modify order in status %s", o.Status)
    }
    if _, exists := o.Items[productID]; !exists {
        return fmt.Errorf("product %s not in order", productID)
    }
    return o.raise("OrderItemRemoved", OrderItemRemoved{
        OrderID:   o.AggregateID(),
        ProductID: productID,
    })
}

func (o *Order) SetShippingAddress(street, city, state, zip, country string) error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot modify order in status %s", o.Status)
    }
    return o.raise("ShippingAddressSet", ShippingAddressSet{
        OrderID: o.AggregateID(),
        Street:  street,
        City:    city,
        State:   state,
        ZipCode: zip,
        Country: country,
    })
}

func (o *Order) Submit() error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot submit order in status %s", o.Status)
    }
    if len(o.Items) == 0 {
        return errors.New("cannot submit an empty order")
    }
    if o.ShippingAddress == nil {
        return errors.New("shipping address is required")
    }
    return o.raise("OrderSubmitted", OrderSubmitted{
        OrderID:     o.AggregateID(),
        SubmittedAt: time.Now().UTC(),
        TotalAmount: o.TotalAmount(),
    })
}

func (o *Order) AuthorizePayment(transactionID string, amount float64) error {
    if o.Status != OrderStatusSubmitted {
        return fmt.Errorf("cannot authorize payment for order in status %s", o.Status)
    }
    return o.raise("PaymentAuthorized", PaymentAuthorized{
        OrderID:       o.AggregateID(),
        TransactionID: transactionID,
        Amount:        amount,
    })
}

func (o *Order) DeclinePayment(reason string) error {
    if o.Status != OrderStatusSubmitted {
        return fmt.Errorf("cannot decline payment for order in status %s", o.Status)
    }
    return o.raise("PaymentDeclined", PaymentDeclined{
        OrderID: o.AggregateID(),
        Reason:  reason,
    })
}

func (o *Order) Approve() error {
    if o.Status != OrderStatusSubmitted {
        return fmt.Errorf("cannot approve order in status %s", o.Status)
    }
    if o.TransactionID == "" {
        return errors.New("payment must be authorized before approval")
    }
    return o.raise("OrderApproved", OrderApproved{
        OrderID:    o.AggregateID(),
        ApprovedAt: time.Now().UTC(),
    })
}

func (o *Order) Ship(trackingID, carrier string) error {
    if o.Status != OrderStatusApproved {
        return fmt.Errorf("cannot ship order in status %s", o.Status)
    }
    return o.raise("OrderShipped", OrderShipped{
        OrderID:    o.AggregateID(),
        TrackingID: trackingID,
        Carrier:    carrier,
        ShippedAt:  time.Now().UTC(),
    })
}

func (o *Order) MarkDelivered() error {
    if o.Status != OrderStatusShipped {
        return fmt.Errorf("cannot mark delivered for order in status %s", o.Status)
    }
    return o.raise("OrderDelivered", OrderDelivered{
        OrderID:     o.AggregateID(),
        DeliveredAt: time.Now().UTC(),
    })
}

func (o *Order) Cancel(reason string) error {
    switch o.Status {
    case OrderStatusShipped, OrderStatusDelivered, OrderStatusCancelled:
        return fmt.Errorf("cannot cancel order in status %s", o.Status)
    }
    return o.raise("OrderCancelled", OrderCancelled{
        OrderID:     o.AggregateID(),
        CancelledAt: time.Now().UTC(),
        Reason:      reason,
    })
}

func (o *Order) TotalAmount() float64 {
    var total float64
    for _, item := range o.Items {
        total += float64(item.Quantity) * item.UnitPrice
    }
    return total
}

// --- Event Application ---

func (o *Order) ApplyEvent(event es.Event) error {
    switch event.Type {
    case "OrderCreated":
        var d OrderCreated
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.CustomerID = d.CustomerID
        o.Status = OrderStatusDraft
        o.CreatedAt = d.CreatedAt

    case "OrderItemAdded":
        var d OrderItemAdded
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.Items[d.ProductID] = &Item{
            ProductID: d.ProductID,
            SKU:       d.SKU,
            Name:      d.Name,
            Quantity:  d.Quantity,
            UnitPrice: d.UnitPrice,
        }

    case "OrderItemQuantityChanged":
        var d OrderItemQuantityChanged
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        if item, ok := o.Items[d.ProductID]; ok {
            item.Quantity = d.NewQty
        }

    case "OrderItemRemoved":
        var d OrderItemRemoved
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        delete(o.Items, d.ProductID)

    case "ShippingAddressSet":
        var d ShippingAddressSet
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.ShippingAddress = &Address{
            Street: d.Street, City: d.City,
            State: d.State, ZipCode: d.ZipCode, Country: d.Country,
        }

    case "OrderSubmitted":
        var d OrderSubmitted
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.Status = OrderStatusSubmitted
        o.SubmittedAt = &d.SubmittedAt

    case "PaymentAuthorized":
        var d PaymentAuthorized
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.TransactionID = d.TransactionID

    case "PaymentDeclined":
        // Status remains submitted; the process manager will handle retry or cancellation.

    case "OrderApproved":
        o.Status = OrderStatusApproved

    case "OrderShipped":
        var d OrderShipped
        if err := json.Unmarshal(event.Data, &d); err != nil {
            return err
        }
        o.Status = OrderStatusShipped
        o.TrackingID = d.TrackingID
        o.Carrier = d.Carrier
        o.ShippedAt = &d.ShippedAt

    case "OrderDelivered":
        o.Status = OrderStatusDelivered

    case "OrderCancelled":
        o.Status = OrderStatusCancelled

    default:
        // Ignore unknown events for forward compatibility.
    }

    o.SetVersion(event.Version)
    return nil
}

// raise is a helper to create and apply an event in one step.
func (o *Order) raise(eventType string, data interface{}) error {
    event, err := o.RaiseEvent(eventType, data)
    if err != nil {
        return err
    }
    return o.ApplyEvent(event)
}
```

### HTTP API Layer

```go
package api

import (
    "encoding/json"
    "net/http"

    es "myapp/eventsourcing"
    "myapp/domain"
    "myapp/readmodel"
)

type OrderAPI struct {
    commandBus   *es.CommandBus
    queryService *readmodel.QueryService
}

func NewOrderAPI(bus *es.CommandBus, qs *readmodel.QueryService) *OrderAPI {
    return &OrderAPI{commandBus: bus, queryService: qs}
}

func (a *OrderAPI) RegisterRoutes(mux *http.ServeMux) {
    // Commands (write side).
    mux.HandleFunc("POST /orders", a.createOrder)
    mux.HandleFunc("POST /orders/{id}/items", a.addItem)
    mux.HandleFunc("POST /orders/{id}/submit", a.submitOrder)

    // Queries (read side).
    mux.HandleFunc("GET /orders/{id}", a.getOrder)
    mux.HandleFunc("GET /customers/{id}/orders", a.listCustomerOrders)
}

type CreateOrderRequest struct {
    CustomerID string `json:"customer_id"`
}

func (a *OrderAPI) createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    cmd := CreateOrderCommand{
        CustomerID: req.CustomerID,
    }

    if err := a.commandBus.Dispatch(r.Context(), cmd); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{
        "order_id": cmd.OrderID,
        "status":   "draft",
    })
}

type AddItemRequest struct {
    ProductID string  `json:"product_id"`
    SKU       string  `json:"sku"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    UnitPrice float64 `json:"unit_price"`
}

func (a *OrderAPI) addItem(w http.ResponseWriter, r *http.Request) {
    orderID := r.PathValue("id")

    var req AddItemRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    cmd := AddItemToOrderCommand{
        OrderID:   orderID,
        ProductID: req.ProductID,
        SKU:       req.SKU,
        Name:      req.Name,
        Quantity:  req.Quantity,
        UnitPrice: req.UnitPrice,
    }

    if err := a.commandBus.Dispatch(r.Context(), cmd); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "item_added"})
}

func (a *OrderAPI) submitOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.PathValue("id")

    cmd := SubmitOrderCommand{OrderID: orderID}
    if err := a.commandBus.Dispatch(r.Context(), cmd); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "submitted"})
}

func (a *OrderAPI) getOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.PathValue("id")

    detail, err := a.queryService.GetOrderDetail(r.Context(), orderID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if detail == nil {
        http.Error(w, "order not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(detail)
}

func (a *OrderAPI) listCustomerOrders(w http.ResponseWriter, r *http.Request) {
    customerID := r.PathValue("id")

    orders, err := a.queryService.ListOrdersByCustomer(r.Context(), customerID, 50, 0)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(orders)
}

// --- Command definitions ---

type CreateOrderCommand struct {
    OrderID    string
    CustomerID string
}
func (c CreateOrderCommand) CommandName() string { return "CreateOrder" }

type AddItemToOrderCommand struct {
    OrderID   string
    ProductID string
    SKU       string
    Name      string
    Quantity  int
    UnitPrice float64
}
func (c AddItemToOrderCommand) CommandName() string { return "AddItemToOrder" }

type SubmitOrderCommand struct {
    OrderID string
}
func (c SubmitOrderCommand) CommandName() string { return "SubmitOrder" }
```

### Wiring It All Together

```go
package main

import (
    "context"
    "database/sql"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    _ "github.com/lib/pq"

    es "myapp/eventsourcing"
    "myapp/api"
    "myapp/domain"
    "myapp/outbox"
    "myapp/projection"
    "myapp/readmodel"
)

func main() {
    // Database connections.
    writeDB, err := sql.Open("postgres", os.Getenv("WRITE_DB_URL"))
    if err != nil {
        log.Fatalf("connect to write DB: %v", err)
    }
    defer writeDB.Close()

    readDB, err := sql.Open("postgres", os.Getenv("READ_DB_URL"))
    if err != nil {
        log.Fatalf("connect to read DB: %v", err)
    }
    defer readDB.Close()

    // Event store.
    eventStore := es.NewPostgresEventStore(writeDB)

    // Repository.
    repo := es.NewRepository(eventStore, func(id string) es.Aggregate {
        return domain.NewOrder(id)
    })

    // Command bus.
    commandBus := es.NewCommandBus()
    registerOrderCommands(commandBus, repo)

    // Projections.
    orderSummaryProjection := projection.NewOrderSummaryProjection(readDB)
    projectionEngine := projection.NewAsyncProjectionEngine(
        eventStore,
        readDB,
        100*time.Millisecond,
        orderSummaryProjection,
    )

    // Query service.
    queryService := readmodel.NewQueryService(readDB)

    // HTTP API.
    orderAPI := api.NewOrderAPI(commandBus, queryService)
    mux := http.NewServeMux()
    orderAPI.RegisterRoutes(mux)

    // Start background processes.
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go func() {
        if err := projectionEngine.Run(ctx); err != nil {
            log.Printf("projection engine stopped: %v", err)
        }
    }()

    // HTTP server.
    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    go func() {
        log.Println("Starting server on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("HTTP server error: %v", err)
        }
    }()

    // Graceful shutdown.
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh

    log.Println("Shutting down...")
    cancel()

    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer shutdownCancel()
    server.Shutdown(shutdownCtx)
}

func registerOrderCommands(bus *es.CommandBus, repo *es.Repository) {
    bus.Register("CreateOrder", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(api.CreateOrderCommand)
        order := domain.NewOrder(c.OrderID)
        if err := order.Create(c.CustomerID); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })

    bus.Register("AddItemToOrder", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(api.AddItemToOrderCommand)
        agg, err := repo.Load(ctx, c.OrderID)
        if err != nil {
            return err
        }
        order := agg.(*domain.Order)
        if err := order.AddItem(c.ProductID, c.SKU, c.Name, c.Quantity, c.UnitPrice); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })

    bus.Register("SubmitOrder", func(ctx context.Context, cmd es.Command) error {
        c := cmd.(api.SubmitOrderCommand)
        agg, err := repo.Load(ctx, c.OrderID)
        if err != nil {
            return err
        }
        order := agg.(*domain.Order)
        if err := order.Submit(); err != nil {
            return err
        }
        return repo.Save(ctx, order)
    })
}
```

---

## 14. Testing Event-Sourced Systems

### Why Event Sourcing Is Excellent for Testing

Event sourcing makes testing significantly easier than traditional architectures:

1. **Given-When-Then** maps directly to events-commands-events.
2. **No database needed** for domain logic tests.
3. **State is deterministic** -- the same events always produce the same state.
4. **Test data is natural** -- just lists of events.

### Testing the Aggregate: Given-When-Then

```go
package domain_test

import (
    "encoding/json"
    "testing"
    "time"

    es "myapp/eventsourcing"
    "myapp/domain"
)

// Helper: create an event for testing.
func makeEvent(t *testing.T, eventType string, version int, data interface{}) es.Event {
    t.Helper()
    dataBytes, err := json.Marshal(data)
    if err != nil {
        t.Fatalf("marshal event data: %v", err)
    }
    return es.Event{
        ID:        "test-event-id",
        Type:      eventType,
        AggregateType: "Order",
        AggregateID:   "order-1",
        Version:   version,
        Data:      dataBytes,
        Timestamp: time.Now().UTC(),
    }
}

// Helper: apply a list of "given" events to an order.
func givenEvents(t *testing.T, order *domain.Order, events []es.Event) {
    t.Helper()
    for _, e := range events {
        if err := order.ApplyEvent(e); err != nil {
            t.Fatalf("apply given event %s: %v", e.Type, err)
        }
    }
}

func TestOrder_Create_Success(t *testing.T) {
    // Given: no prior events (fresh aggregate).
    order := domain.NewOrder("order-1")

    // When: create the order.
    err := order.Create("customer-123")

    // Then: no error and one uncommitted event.
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }
    events := order.UncommittedEvents()
    if len(events) != 1 {
        t.Fatalf("expected 1 event, got %d", len(events))
    }
    if events[0].Type != "OrderCreated" {
        t.Errorf("expected OrderCreated, got %s", events[0].Type)
    }
    if order.Status != domain.OrderStatusDraft {
        t.Errorf("expected status draft, got %s", order.Status)
    }
}

func TestOrder_Create_AlreadyExists(t *testing.T) {
    // Given: an order that already exists.
    order := domain.NewOrder("order-1")
    givenEvents(t, order, []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID:    "order-1",
            CustomerID: "customer-123",
            CreatedAt:  time.Now().UTC(),
        }),
    })

    // When: try to create again.
    err := order.Create("customer-456")

    // Then: error.
    if err == nil {
        t.Fatal("expected error, got nil")
    }
}

func TestOrder_AddItem_Success(t *testing.T) {
    // Given: a draft order.
    order := domain.NewOrder("order-1")
    givenEvents(t, order, []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID:    "order-1",
            CustomerID: "customer-123",
            CreatedAt:  time.Now().UTC(),
        }),
    })

    // When: add an item.
    err := order.AddItem("prod-1", "SKU-001", "Widget", 2, 19.99)

    // Then: item is in the order.
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }
    if len(order.Items) != 1 {
        t.Fatalf("expected 1 item, got %d", len(order.Items))
    }
    item := order.Items["prod-1"]
    if item.Quantity != 2 || item.UnitPrice != 19.99 {
        t.Errorf("unexpected item: %+v", item)
    }
}

func TestOrder_Submit_EmptyOrder(t *testing.T) {
    // Given: a draft order with no items.
    order := domain.NewOrder("order-1")
    givenEvents(t, order, []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID:    "order-1",
            CustomerID: "customer-123",
            CreatedAt:  time.Now().UTC(),
        }),
    })

    // When: try to submit.
    err := order.Submit()

    // Then: error about empty order.
    if err == nil {
        t.Fatal("expected error for empty order, got nil")
    }
}

func TestOrder_Submit_NoShippingAddress(t *testing.T) {
    // Given: a draft order with items but no shipping address.
    order := domain.NewOrder("order-1")
    givenEvents(t, order, []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID:    "order-1",
            CustomerID: "customer-123",
            CreatedAt:  time.Now().UTC(),
        }),
        makeEvent(t, "OrderItemAdded", 2, domain.OrderItemAdded{
            OrderID:   "order-1",
            ProductID: "prod-1",
            SKU:       "SKU-001",
            Name:      "Widget",
            Quantity:  1,
            UnitPrice: 9.99,
        }),
    })

    // When: try to submit.
    err := order.Submit()

    // Then: error about missing address.
    if err == nil {
        t.Fatal("expected error for missing address, got nil")
    }
}

func TestOrder_Cancel_ShippedOrder(t *testing.T) {
    // Given: a shipped order.
    order := domain.NewOrder("order-1")
    givenEvents(t, order, []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID: "order-1", CustomerID: "c1", CreatedAt: time.Now().UTC(),
        }),
        makeEvent(t, "OrderItemAdded", 2, domain.OrderItemAdded{
            OrderID: "order-1", ProductID: "p1", SKU: "S1", Name: "W", Quantity: 1, UnitPrice: 10,
        }),
        makeEvent(t, "ShippingAddressSet", 3, domain.ShippingAddressSet{
            OrderID: "order-1", Street: "123 Main", City: "NY", State: "NY", ZipCode: "10001", Country: "US",
        }),
        makeEvent(t, "OrderSubmitted", 4, domain.OrderSubmitted{
            OrderID: "order-1", SubmittedAt: time.Now().UTC(), TotalAmount: 10,
        }),
        makeEvent(t, "PaymentAuthorized", 5, domain.PaymentAuthorized{
            OrderID: "order-1", TransactionID: "tx-1", Amount: 10,
        }),
        makeEvent(t, "OrderApproved", 6, domain.OrderApproved{
            OrderID: "order-1", ApprovedAt: time.Now().UTC(),
        }),
        makeEvent(t, "OrderShipped", 7, domain.OrderShipped{
            OrderID: "order-1", TrackingID: "TRK-1", Carrier: "UPS", ShippedAt: time.Now().UTC(),
        }),
    })

    // When: try to cancel.
    err := order.Cancel("changed my mind")

    // Then: error -- cannot cancel shipped order.
    if err == nil {
        t.Fatal("expected error cancelling shipped order, got nil")
    }
}
```

### Testing Aggregate Reconstitution

```go
func TestOrder_Reconstitution(t *testing.T) {
    // Simulate loading from event store by replaying events.
    order := domain.NewOrder("order-1")
    events := []es.Event{
        makeEvent(t, "OrderCreated", 1, domain.OrderCreated{
            OrderID: "order-1", CustomerID: "c-1", CreatedAt: time.Now().UTC(),
        }),
        makeEvent(t, "OrderItemAdded", 2, domain.OrderItemAdded{
            OrderID: "order-1", ProductID: "p-1", SKU: "S1",
            Name: "Widget", Quantity: 3, UnitPrice: 10.0,
        }),
        makeEvent(t, "OrderItemAdded", 3, domain.OrderItemAdded{
            OrderID: "order-1", ProductID: "p-2", SKU: "S2",
            Name: "Gadget", Quantity: 1, UnitPrice: 25.0,
        }),
        makeEvent(t, "OrderItemRemoved", 4, domain.OrderItemRemoved{
            OrderID: "order-1", ProductID: "p-1",
        }),
    }

    for _, e := range events {
        if err := order.ApplyEvent(e); err != nil {
            t.Fatalf("apply event: %v", err)
        }
    }

    // Verify state.
    if order.Version() != 4 {
        t.Errorf("expected version 4, got %d", order.Version())
    }
    if order.CustomerID != "c-1" {
        t.Errorf("expected customer c-1, got %s", order.CustomerID)
    }
    if len(order.Items) != 1 {
        t.Errorf("expected 1 item, got %d", len(order.Items))
    }
    if _, ok := order.Items["p-2"]; !ok {
        t.Error("expected product p-2 to be in order")
    }
    if order.TotalAmount() != 25.0 {
        t.Errorf("expected total 25.0, got %.2f", order.TotalAmount())
    }
}
```

### Testing Projections

```go
package projection_test

import (
    "context"
    "database/sql"
    "testing"
    "time"

    es "myapp/eventsourcing"
    "myapp/domain"
    "myapp/projection"
)

func TestOrderSummaryProjection(t *testing.T) {
    // Use an in-memory SQLite or test PostgreSQL database.
    db := setupTestDB(t) // Assume this creates a test DB with the schema.
    defer db.Close()

    proj := projection.NewOrderSummaryProjection(db)
    ctx := context.Background()

    // Simulate processing events.
    events := []es.Event{
        makeEvent(t, "OrderPlaced", 1, domain.OrderCreated{
            OrderID: "order-1", CustomerID: "c-1", CreatedAt: time.Now().UTC(),
        }),
        makeEvent(t, "ItemAdded", 2, domain.OrderItemAdded{
            OrderID: "order-1", ProductID: "p-1", Name: "Widget", Quantity: 2, UnitPrice: 15.0,
        }),
    }

    for _, e := range events {
        if err := proj.Handle(ctx, e); err != nil {
            t.Fatalf("handle event: %v", err)
        }
    }

    // Query the read model and verify.
    var itemCount int
    var totalAmount float64
    err := db.QueryRow(
        "SELECT item_count, total_amount FROM order_summaries WHERE order_id = $1",
        "order-1",
    ).Scan(&itemCount, &totalAmount)
    if err != nil {
        t.Fatalf("query: %v", err)
    }

    if itemCount != 2 {
        t.Errorf("expected item_count 2, got %d", itemCount)
    }
    if totalAmount != 30.0 {
        t.Errorf("expected total_amount 30.0, got %.2f", totalAmount)
    }
}
```

### Testing the Event Store

```go
package eventsourcing_test

import (
    "context"
    "testing"

    es "myapp/eventsourcing"
)

func TestInMemoryEventStore_AppendAndLoad(t *testing.T) {
    store := es.NewInMemoryEventStore()
    ctx := context.Background()

    events := []es.Event{
        {ID: "e1", Type: "OrderCreated", Data: []byte(`{"order_id":"o1"}`)},
        {ID: "e2", Type: "ItemAdded", Data: []byte(`{"product_id":"p1"}`)},
    }

    // Append to a new stream.
    err := store.AppendEvents(ctx, "Order-o1", -1, events)
    if err != nil {
        t.Fatalf("append: %v", err)
    }

    // Load back.
    loaded, err := store.LoadEvents(ctx, "Order-o1")
    if err != nil {
        t.Fatalf("load: %v", err)
    }

    if len(loaded) != 2 {
        t.Fatalf("expected 2 events, got %d", len(loaded))
    }
    if loaded[0].Version != 1 || loaded[1].Version != 2 {
        t.Errorf("unexpected versions: %d, %d", loaded[0].Version, loaded[1].Version)
    }
}

func TestInMemoryEventStore_ConcurrencyConflict(t *testing.T) {
    store := es.NewInMemoryEventStore()
    ctx := context.Background()

    // Create stream.
    store.AppendEvents(ctx, "Order-o1", -1, []es.Event{
        {ID: "e1", Type: "OrderCreated", Data: []byte(`{}`)},
    })

    // Try to append with wrong expected version.
    err := store.AppendEvents(ctx, "Order-o1", 5, []es.Event{
        {ID: "e2", Type: "ItemAdded", Data: []byte(`{}`)},
    })

    if err == nil {
        t.Fatal("expected concurrency error, got nil")
    }
}

func TestInMemoryEventStore_LoadEventsFrom(t *testing.T) {
    store := es.NewInMemoryEventStore()
    ctx := context.Background()

    events := []es.Event{
        {ID: "e1", Type: "A", Data: []byte(`{}`)},
        {ID: "e2", Type: "B", Data: []byte(`{}`)},
        {ID: "e3", Type: "C", Data: []byte(`{}`)},
    }
    store.AppendEvents(ctx, "stream-1", -1, events)

    // Load from version 2.
    loaded, err := store.LoadEventsFrom(ctx, "stream-1", 2)
    if err != nil {
        t.Fatalf("load from: %v", err)
    }
    if len(loaded) != 2 {
        t.Fatalf("expected 2 events, got %d", len(loaded))
    }
    if loaded[0].Type != "B" || loaded[1].Type != "C" {
        t.Errorf("unexpected types: %s, %s", loaded[0].Type, loaded[1].Type)
    }
}
```

### Test Fixtures: Event Builder

For complex test scenarios, a builder pattern keeps tests readable.

```go
package testutil

import (
    "encoding/json"
    "time"

    es "myapp/eventsourcing"
    "github.com/google/uuid"
)

// EventBuilder helps construct test events fluently.
type EventBuilder struct {
    event es.Event
}

func NewEventBuilder() *EventBuilder {
    return &EventBuilder{
        event: es.Event{
            ID:        uuid.New().String(),
            Timestamp: time.Now().UTC(),
        },
    }
}

func (b *EventBuilder) Type(t string) *EventBuilder {
    b.event.Type = t
    return b
}

func (b *EventBuilder) AggregateID(id string) *EventBuilder {
    b.event.AggregateID = id
    return b
}

func (b *EventBuilder) Version(v int) *EventBuilder {
    b.event.Version = v
    return b
}

func (b *EventBuilder) Data(data interface{}) *EventBuilder {
    bytes, _ := json.Marshal(data)
    b.event.Data = bytes
    return b
}

func (b *EventBuilder) Build() es.Event {
    return b.event
}

// Usage in tests:
//
//   event := testutil.NewEventBuilder().
//       Type("OrderCreated").
//       AggregateID("order-1").
//       Version(1).
//       Data(domain.OrderCreated{OrderID: "order-1", CustomerID: "c-1"}).
//       Build()
```

### Behavior-Driven Test Helper

```go
package testutil

import (
    "testing"

    es "myapp/eventsourcing"
)

// AggregateTestCase provides a Given-When-Then harness for aggregate tests.
type AggregateTestCase struct {
    t         *testing.T
    aggregate es.Aggregate
    givenEvts []es.Event
    whenErr   error
}

func Given(t *testing.T, agg es.Aggregate, events ...es.Event) *AggregateTestCase {
    t.Helper()
    for _, e := range events {
        if err := agg.ApplyEvent(e); err != nil {
            t.Fatalf("apply given event: %v", err)
        }
    }
    agg.ClearUncommittedEvents()
    return &AggregateTestCase{t: t, aggregate: agg, givenEvts: events}
}

func (tc *AggregateTestCase) When(action func() error) *AggregateTestCase {
    tc.t.Helper()
    tc.whenErr = action()
    return tc
}

func (tc *AggregateTestCase) ThenExpectEvents(expectedTypes ...string) *AggregateTestCase {
    tc.t.Helper()
    if tc.whenErr != nil {
        tc.t.Fatalf("expected success but got error: %v", tc.whenErr)
    }
    uncommitted := tc.aggregate.UncommittedEvents()
    if len(uncommitted) != len(expectedTypes) {
        tc.t.Fatalf("expected %d events, got %d", len(expectedTypes), len(uncommitted))
    }
    for i, expected := range expectedTypes {
        if uncommitted[i].Type != expected {
            tc.t.Errorf("event[%d]: expected type %s, got %s", i, expected, uncommitted[i].Type)
        }
    }
    return tc
}

func (tc *AggregateTestCase) ThenExpectError(contains string) *AggregateTestCase {
    tc.t.Helper()
    if tc.whenErr == nil {
        tc.t.Fatal("expected error, got nil")
    }
    if contains != "" {
        // Optionally check error message content.
        if !containsString(tc.whenErr.Error(), contains) {
            tc.t.Errorf("expected error containing %q, got %q", contains, tc.whenErr.Error())
        }
    }
    return tc
}

func containsString(s, substr string) bool {
    return len(s) >= len(substr) && (s == substr || len(s) > 0 && containsSubstring(s, substr))
}

func containsSubstring(s, sub string) bool {
    for i := 0; i <= len(s)-len(sub); i++ {
        if s[i:i+len(sub)] == sub {
            return true
        }
    }
    return false
}

// Usage:
//
// func TestOrder_Submit(t *testing.T) {
//     order := domain.NewOrder("o1")
//     testutil.Given(t, order,
//         makeEvent(t, "OrderCreated", 1, ...),
//         makeEvent(t, "OrderItemAdded", 2, ...),
//         makeEvent(t, "ShippingAddressSet", 3, ...),
//     ).When(func() error {
//         return order.Submit()
//     }).ThenExpectEvents("OrderSubmitted")
// }
```

---

## 15. When to Use (and When NOT to Use) Event Sourcing

### Good Fit: Use Event Sourcing When...

| Scenario | Why |
|----------|-----|
| **Audit trail is critical** | Financial systems, healthcare, compliance. The event log IS the audit trail. |
| **Temporal queries needed** | "What was the portfolio value on March 15?" Replay events up to that date. |
| **Complex domain with many state transitions** | Order management, insurance claims, supply chain. The state machine is naturally expressed as events. |
| **Event-driven architecture** | If you are already publishing domain events, storing them becomes natural. |
| **High write throughput** | Append-only writes are fast. No read-modify-write cycles. |
| **Need to rebuild read models** | Adding a new dashboard or search feature? Replay the event log into a new projection. |
| **Debugging production issues** | Replay the exact sequence of events that led to a bug. |

### Bad Fit: Do NOT Use Event Sourcing When...

| Scenario | Why |
|----------|-----|
| **Simple CRUD** | A blog, a settings page, a user profile. The overhead is not justified. |
| **Bulk data import** | Importing millions of rows as individual events is slow and wasteful. |
| **Queries on current state are the main use case** | If you rarely need history, the projection overhead is pure cost. |
| **Team lacks experience** | Event sourcing has a steep learning curve. Starting with it on a new team is risky. |
| **Strict consistency requirements everywhere** | Eventual consistency from async projections requires careful UI/UX design. |
| **Rapid prototyping** | Event sourcing is slower to build initially. Use CRUD to validate the idea first. |
| **Schema changes frequently** | Very early-stage products change domain models rapidly. Event versioning adds friction. |
| **Privacy regulations (GDPR right to erasure)** | Immutable event logs and "delete my data" are in fundamental tension. Requires crypto-shredding or tombstoning. |

### The Hybrid Approach

Most production systems do not event-source everything. A common and pragmatic approach:

- **Event-source** the core domain aggregates (orders, accounts, claims).
- **Use traditional CRUD** for supporting entities (user preferences, configuration).
- **Use CQRS** broadly, even without event sourcing, for read-heavy services.

```
┌─────────────────────────────────────────────┐
│            Application Boundary              │
│                                              │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │ Order Service │    │ User Preferences │   │
│  │              │    │   Service         │   │
│  │ Event Sourced│    │ Traditional CRUD  │   │
│  │ + CQRS       │    │                   │   │
│  └──────────────┘    └──────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │        Reporting Service              │   │
│  │   CQRS read models (no event sourcing)│   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### GDPR Considerations

When using event sourcing with personal data, you need a strategy for data erasure.

**Crypto-Shredding:** Encrypt personal data in events with a per-user key. To "delete" the data, delete the encryption key. The events remain but the personal data becomes unreadable.

```go
package gdpr

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/json"
    "fmt"
    "io"
)

// EncryptedEventData wraps event data with encryption.
type EncryptedEventData struct {
    KeyID      string `json:"key_id"`      // Reference to the encryption key.
    Ciphertext []byte `json:"ciphertext"`
    Nonce      []byte `json:"nonce"`
}

// KeyStore manages per-user encryption keys.
type KeyStore interface {
    GetKey(keyID string) ([]byte, error)
    DeleteKey(keyID string) error // Called for GDPR erasure.
}

func EncryptEventData(data interface{}, key []byte, keyID string) (json.RawMessage, error) {
    plaintext, err := json.Marshal(data)
    if err != nil {
        return nil, err
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }

    ciphertext := gcm.Seal(nil, nonce, plaintext, nil)

    encrypted := EncryptedEventData{
        KeyID:      keyID,
        Ciphertext: ciphertext,
        Nonce:      nonce,
    }

    return json.Marshal(encrypted)
}

func DecryptEventData(raw json.RawMessage, keyStore KeyStore) (json.RawMessage, error) {
    var encrypted EncryptedEventData
    if err := json.Unmarshal(raw, &encrypted); err != nil {
        return nil, err
    }

    key, err := keyStore.GetKey(encrypted.KeyID)
    if err != nil {
        return nil, fmt.Errorf("key not found (data may have been erased): %w", err)
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    plaintext, err := gcm.Open(nil, encrypted.Nonce, encrypted.Ciphertext, nil)
    if err != nil {
        return nil, err
    }

    return plaintext, nil
}
```

---

## 16. Common Pitfalls and Lessons Learned

### Pitfall 1: Too Many Events Per Aggregate

**Problem:** An aggregate with millions of events is slow to load.

**Solution:** Use snapshots (Section 5), or reconsider your aggregate boundaries. If an aggregate grows unbounded, it may be too coarse-grained. For example, a "ShoppingCart" aggregate that accumulates events over years should probably be split into per-session or per-order aggregates.

### Pitfall 2: Large Event Payloads

**Problem:** Storing entire documents (e.g., a 50KB PDF) as event data bloats the event store.

**Solution:** Store references (IDs, URLs) in events, not the full content. Put large blobs in object storage (S3) and reference them by key.

```go
// Bad: large payload in event.
type DocumentUploaded struct {
    DocumentID string `json:"document_id"`
    Content    []byte `json:"content"` // 50KB+ in every event!
}

// Good: reference to external storage.
type DocumentUploaded struct {
    DocumentID string `json:"document_id"`
    StorageKey string `json:"storage_key"` // "s3://bucket/docs/abc123.pdf"
    SizeBytes  int64  `json:"size_bytes"`
    MimeType   string `json:"mime_type"`
}
```

### Pitfall 3: Treating Events as Commands

**Problem:** Events named like commands ("CreateOrder" instead of "OrderCreated") indicate confused modeling. Events describe what happened, not what should happen.

**Solution:** Always use past tense for events. Commands are imperative ("CreateOrder"), events are declarative ("OrderCreated").

| Wrong (Command-style) | Right (Event-style) |
|------------------------|---------------------|
| CreateOrder | OrderCreated |
| UpdateAddress | AddressUpdated |
| DeleteItem | ItemRemoved |
| ProcessPayment | PaymentProcessed |

### Pitfall 4: Not Handling Idempotency

**Problem:** Processing the same event twice in a projection doubles the count, adds duplicate rows, etc.

**Solution:** Track which events have been processed (using event IDs or global positions) and skip duplicates.

```go
func (p *SomeProjection) Handle(ctx context.Context, event es.Event) error {
    // Check if already processed.
    var exists bool
    err := p.db.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_events WHERE event_id = $1 AND projection = $2)",
        event.ID, p.Name(),
    ).Scan(&exists)
    if err != nil {
        return err
    }
    if exists {
        return nil // Already processed, skip.
    }

    // Process the event...

    // Mark as processed.
    _, err = p.db.ExecContext(ctx,
        "INSERT INTO processed_events (event_id, projection) VALUES ($1, $2)",
        event.ID, p.Name(),
    )
    return err
}
```

### Pitfall 5: Querying the Event Store Directly

**Problem:** Writing queries against the event store to answer business questions. The event store is not designed for this -- it is an append-only log, not a query engine.

**Solution:** Build projections for every query you need. The event store's job is to store and replay events, nothing more.

### Pitfall 6: Not Planning for Projection Rebuilds

**Problem:** You need to fix a bug in a projection, but rebuilding takes hours because you did not plan for it.

**Solution:**
- Build projection rebuild tooling from day one.
- Test that projections can be rebuilt from scratch.
- Consider blue/green projections (build new, then swap).
- Partition events by time for faster partial rebuilds.

### Pitfall 7: Coupling Aggregates via Events

**Problem:** Aggregate A reads events from Aggregate B's stream to make decisions. This creates hidden coupling.

**Solution:** Aggregates should only read their own events. Use sagas or process managers to coordinate between aggregates.

```go
// BAD: Order aggregate reads from Inventory stream.
func (o *Order) CanShip(inventoryEvents []es.Event) bool {
    // This tightly couples Order to Inventory's event schema.
}

// GOOD: Process manager coordinates.
// OrderFulfillmentProcessManager listens to both Order and Inventory events
// and sends commands to the appropriate aggregate.
```

### Pitfall 8: Over-Engineering from Day One

**Problem:** Building a full-blown CQRS/ES infrastructure when you have five users and one developer.

**Solution:** Start simple:
1. Begin with a single database and synchronous projections.
2. Add async projections when you need independent scaling.
3. Add the outbox pattern when you need reliable cross-service messaging.
4. Add EventStoreDB when you outgrow PostgreSQL's event storage.

### Pitfall 9: Ignoring Event Ordering

**Problem:** Events processed out of order produce incorrect projections.

**Solution:**
- Within a single stream, events are always ordered by version. Never process them out of order.
- Across streams, use the global position to maintain a total order.
- When using message brokers, ensure ordering guarantees (Kafka partitions by aggregate ID, for example).

### Pitfall 10: Forgetting About Concurrency

**Problem:** Two users edit the same order simultaneously. Without optimistic concurrency, the second write silently overwrites the first.

**Solution:** Always use the expected version parameter when appending events. On conflict, reload the aggregate and retry or return an error to the user.

```go
func (h *Handler) AddItemWithRetry(ctx context.Context, cmd AddItemCommand) error {
    const maxRetries = 3

    for attempt := 0; attempt < maxRetries; attempt++ {
        agg, err := h.repo.Load(ctx, cmd.OrderID)
        if err != nil {
            return err
        }
        order := agg.(*Order)

        if err := order.AddItem(cmd.ProductID, cmd.Name, cmd.Qty, cmd.Price); err != nil {
            return err // Business rule violation, do not retry.
        }

        err = h.repo.Save(ctx, order)
        if err == nil {
            return nil // Success.
        }

        // Check if it is a concurrency error.
        var concErr *es.ConcurrencyError
        if !errors.As(err, &concErr) {
            return err // Not a concurrency error, do not retry.
        }

        // Concurrency conflict -- retry with fresh state.
        log.Printf("Concurrency conflict on order %s, retrying (attempt %d)", cmd.OrderID, attempt+1)
    }

    return fmt.Errorf("failed after %d retries due to concurrency conflicts", maxRetries)
}
```

---

## Summary

### Key Takeaways

1. **Events are immutable facts.** They represent what happened, never what should happen. Name them in past tense.

2. **The event log is the source of truth.** Everything else -- read models, search indexes, caches -- is derived and rebuildable.

3. **Aggregates enforce business rules.** They accept commands, validate invariants, and produce events. Keep them focused on a single consistency boundary.

4. **CQRS separates reads from writes.** The write model (event-sourced aggregate) and the read model (denormalized projection) serve different purposes and can scale independently.

5. **Projections bridge the gap.** Synchronous projections give strong consistency; asynchronous projections give better write performance at the cost of eventual consistency.

6. **Event versioning is inevitable.** Plan for it from the start. Upcasting is the cleanest approach.

7. **Snapshots are an optimization.** Only add them when aggregate replay becomes a measurable performance problem.

8. **The outbox pattern ensures reliability.** Never rely on "write to DB then publish to broker" -- use a transactional outbox.

9. **Sagas coordinate multi-aggregate processes.** Choose choreography for simple flows, orchestration for complex ones.

10. **Test with Given-When-Then.** Event sourcing makes domain logic testing natural and database-free.

### Architecture Decision Record

Before adopting event sourcing, answer these questions:

| Question | If Yes... | If No... |
|----------|-----------|----------|
| Do you need a complete audit trail? | Strong signal for ES | Audit logging may suffice |
| Do you need temporal queries? | ES is a natural fit | Not needed |
| Is the domain event-driven? | ES aligns well | Consider CRUD |
| Is the team experienced with ES? | Go for it | Start with a bounded context, learn first |
| Do you have high write throughput? | Append-only stores scale well | Traditional DB is fine |
| Do you need to evolve read models? | Rebuildable projections are powerful | Schema migrations work fine |

### Recommended Libraries and Tools

| Tool | Purpose |
|------|---------|
| **EventStoreDB** | Purpose-built event store database |
| **PostgreSQL** | Excellent general-purpose event store backend |
| **Apache Kafka** | Event streaming for cross-service communication |
| **NATS JetStream** | Lightweight event streaming with persistence |
| **github.com/EventStore/EventStore-Client-Go** | Official Go client for EventStoreDB |
| **github.com/ThreeDotsLabs/watermill** | Go library for event-driven applications |
| **github.com/looplab/eventhorizon** | CQRS/ES framework for Go |

---

*Event sourcing and CQRS are powerful patterns, but they are not silver bullets. Apply them where they genuinely solve a problem, keep the implementation as simple as possible, and invest in good tooling for projection rebuilds and event versioning from the start.*
