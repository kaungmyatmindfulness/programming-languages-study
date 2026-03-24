# Clean Architecture and Domain-Driven Design in Go

## Table of Contents

1. [Introduction](#introduction)
2. [Clean Architecture Principles](#clean-architecture-principles)
3. [Hexagonal Architecture (Ports and Adapters)](#hexagonal-architecture-ports-and-adapters)
4. [Project Structure for Clean Architecture](#project-structure-for-clean-architecture)
5. [Domain-Driven Design Tactical Patterns in Go](#domain-driven-design-tactical-patterns-in-go)
6. [Bounded Contexts and Context Mapping](#bounded-contexts-and-context-mapping)
7. [Anti-Corruption Layer](#anti-corruption-layer)
8. [Ubiquitous Language in Code](#ubiquitous-language-in-code)
9. [Error Handling in the Domain Layer](#error-handling-in-the-domain-layer)
10. [Testing Each Layer Independently](#testing-each-layer-independently)
11. [When Clean Architecture / DDD Is Overkill](#when-clean-architecture--ddd-is-overkill)
12. [Complete Example: Order Management System](#complete-example-order-management-system)
13. [Common Mistakes and Pragmatic Shortcuts](#common-mistakes-and-pragmatic-shortcuts)
14. [Summary](#summary)

---

## Introduction

Clean Architecture (Robert C. Martin, 2012) and Domain-Driven Design (Eric Evans, 2003) are
complementary approaches to structuring software. Clean Architecture gives you a blueprint for
organizing dependencies so that the core business logic is shielded from infrastructure concerns.
DDD gives you a vocabulary and set of tactical patterns for modelling complex business domains.

Go is particularly well-suited to these patterns because:

- Interfaces are implicitly satisfied, making dependency inversion natural.
- The type system is simple yet expressive enough for rich domain models.
- Packages enforce boundaries at compile time.
- There is no inheritance hierarchy to fight against; composition is idiomatic.

This chapter teaches you how to combine both approaches in idiomatic Go, with full working
examples that go beyond toy demos.

---

## Clean Architecture Principles

### The Core Idea

Clean Architecture arranges code in concentric layers. The **Dependency Rule** states:

> Source code dependencies must point inward only. Nothing in an inner layer can know anything
> about an outer layer.

```
+-----------------------------------------------------+
|                   Frameworks & Drivers               |
|  (HTTP server, database driver, message broker)      |
|                                                      |
|   +---------------------------------------------+   |
|   |            Interface Adapters                |   |
|   |  (Controllers, Presenters, Gateways)         |   |
|   |                                              |   |
|   |   +-------------------------------------+    |   |
|   |   |         Application Layer           |    |   |
|   |   |  (Use Cases / Application Services) |    |   |
|   |   |                                     |    |   |
|   |   |   +-----------------------------+   |    |   |
|   |   |   |       Domain Layer          |   |    |   |
|   |   |   |  (Entities, Value Objects,  |   |    |   |
|   |   |   |   Domain Services, Events)  |   |    |   |
|   |   |   +-----------------------------+   |    |   |
|   |   +-------------------------------------+    |   |
|   +---------------------------------------------+   |
+-----------------------------------------------------+
```

### The Four Layers

| Layer | Responsibility | Go Package Example | Changes When... |
|---|---|---|---|
| **Domain** | Core business rules, entities, value objects | `internal/domain/order` | Business rules change |
| **Application** | Orchestrates use cases, coordinates domain objects | `internal/application/order` | Workflows change |
| **Interface Adapters** | Translates between external formats and application | `internal/interfaces/http` | API contract changes |
| **Frameworks & Drivers** | Concrete implementations of external dependencies | `internal/infrastructure/postgres` | Technology choices change |

### The Dependency Rule in Go

In Go, the dependency rule is enforced through **interfaces defined in inner layers** and
**implementations provided by outer layers**.

```go
// domain/order/repository.go  -- INNER LAYER defines the interface
package order

import "context"

// Repository is defined in the domain layer.
// It describes what the domain NEEDS, not how it is implemented.
type Repository interface {
    FindByID(ctx context.Context, id OrderID) (*Order, error)
    Save(ctx context.Context, order *Order) error
    NextID(ctx context.Context) (OrderID, error)
}
```

```go
// infrastructure/postgres/order_repo.go  -- OUTER LAYER implements it
package postgres

import (
    "context"
    "database/sql"

    "myapp/internal/domain/order"
)

type OrderRepository struct {
    db *sql.DB
}

func NewOrderRepository(db *sql.DB) *OrderRepository {
    return &OrderRepository{db: db}
}

func (r *OrderRepository) FindByID(ctx context.Context, id order.OrderID) (*order.Order, error) {
    // SQL query, row scanning, mapping to domain entity
    row := r.db.QueryRowContext(ctx, "SELECT id, customer_id, status, total FROM orders WHERE id = $1", string(id))
    // ... scan and reconstruct domain object
    return reconstructOrder(row)
}

func (r *OrderRepository) Save(ctx context.Context, o *order.Order) error {
    // Persist the order using SQL
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO orders (id, customer_id, status, total)
         VALUES ($1, $2, $3, $4)
         ON CONFLICT (id) DO UPDATE SET status = $3, total = $4`,
        o.ID(), o.CustomerID(), o.Status(), o.Total().Amount(),
    )
    return err
}

func (r *OrderRepository) NextID(ctx context.Context) (order.OrderID, error) {
    // Generate a new unique ID (could use UUID)
    return order.NewOrderID(), nil
}
```

### Why This Matters

- **Testability**: You can test domain logic with no database, no HTTP server, no filesystem.
- **Flexibility**: Swap Postgres for MongoDB by writing a new adapter; zero changes to domain code.
- **Longevity**: Business rules outlive frameworks. Your domain layer can survive a full rewrite
  of the infrastructure layer.

> **Tip**: Think of the dependency rule as a compile-time guarantee. If your `domain` package
> imports `database/sql`, you have violated the rule and the compiler should have stopped you
> (via linting or import path discipline).

---

## Hexagonal Architecture (Ports and Adapters)

Hexagonal Architecture (Alistair Cockburn, 2005) is another name for a very similar concept.
The terminology differs but the intent is identical to Clean Architecture.

### Ports

A **port** is an interface that defines how the application interacts with the outside world.
There are two kinds:

| Port Type | Direction | Example |
|---|---|---|
| **Driving (Primary)** | Outside -> Application | `OrderService` interface that an HTTP handler calls |
| **Driven (Secondary)** | Application -> Outside | `OrderRepository` interface that the app calls |

### Adapters

An **adapter** is a concrete implementation of a port.

| Adapter Type | Implements | Example |
|---|---|---|
| **Driving Adapter** | Calls primary ports | HTTP handler, gRPC server, CLI command |
| **Driven Adapter** | Implements secondary ports | Postgres repository, Redis cache, SMTP mailer |

### Hexagonal Architecture in Go

```go
// --- PRIMARY PORT (what the outside world can ask us to do) ---
// application/order_service.go
package application

import "context"

type PlaceOrderCommand struct {
    CustomerID string
    Items      []OrderItemDTO
}

type OrderItemDTO struct {
    ProductID string
    Quantity  int
}

// OrderService is the primary port -- the entry point for use cases.
type OrderService interface {
    PlaceOrder(ctx context.Context, cmd PlaceOrderCommand) (string, error)
    CancelOrder(ctx context.Context, orderID string) error
    GetOrder(ctx context.Context, orderID string) (*OrderDTO, error)
}
```

```go
// --- SECONDARY PORT (what we need from the outside world) ---
// domain/order/repository.go
package order

import "context"

type Repository interface {
    FindByID(ctx context.Context, id OrderID) (*Order, error)
    Save(ctx context.Context, order *Order) error
}
```

```go
// --- SECONDARY PORT (notification) ---
// domain/notification/notifier.go
package notification

import "context"

type Notifier interface {
    NotifyOrderPlaced(ctx context.Context, orderID string, customerEmail string) error
}
```

```go
// --- DRIVING ADAPTER (HTTP handler calls the primary port) ---
// interfaces/http/order_handler.go
package http

import (
    "encoding/json"
    "net/http"

    "myapp/internal/application"
)

type OrderHandler struct {
    service application.OrderService
}

func NewOrderHandler(s application.OrderService) *OrderHandler {
    return &OrderHandler{service: s}
}

func (h *OrderHandler) PlaceOrder(w http.ResponseWriter, r *http.Request) {
    var cmd application.PlaceOrderCommand
    if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    orderID, err := h.service.PlaceOrder(r.Context(), cmd)
    if err != nil {
        // Map domain errors to HTTP status codes (covered later)
        handleError(w, err)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{"order_id": orderID})
}
```

```go
// --- DRIVEN ADAPTER (implements the secondary notification port) ---
// infrastructure/email/notifier.go
package email

import (
    "context"
    "fmt"
    "net/smtp"
)

type SMTPNotifier struct {
    host     string
    port     int
    from     string
    password string
}

func NewSMTPNotifier(host string, port int, from, password string) *SMTPNotifier {
    return &SMTPNotifier{host: host, port: port, from: from, password: password}
}

func (n *SMTPNotifier) NotifyOrderPlaced(ctx context.Context, orderID, customerEmail string) error {
    subject := fmt.Sprintf("Order %s Confirmed", orderID)
    body := fmt.Sprintf("Your order %s has been placed successfully.", orderID)
    msg := fmt.Sprintf("Subject: %s\r\n\r\n%s", subject, body)

    addr := fmt.Sprintf("%s:%d", n.host, n.port)
    auth := smtp.PlainAuth("", n.from, n.password, n.host)
    return smtp.SendMail(addr, auth, n.from, []string{customerEmail}, []byte(msg))
}
```

### Wiring It Together

The `main` function (or a composition root) is the only place that knows about all concrete types:

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"

    _ "github.com/lib/pq"

    appService "myapp/internal/application"
    emailAdapter "myapp/internal/infrastructure/email"
    pgAdapter "myapp/internal/infrastructure/postgres"
    httpAdapter "myapp/internal/interfaces/http"
)

func main() {
    // Infrastructure
    db, err := sql.Open("postgres", "postgres://localhost/myapp?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Driven adapters (secondary)
    orderRepo := pgAdapter.NewOrderRepository(db)
    notifier := emailAdapter.NewSMTPNotifier("smtp.example.com", 587, "noreply@example.com", "secret")

    // Application service (implements primary port)
    orderService := appService.NewOrderServiceImpl(orderRepo, notifier)

    // Driving adapter (primary)
    handler := httpAdapter.NewOrderHandler(orderService)

    // Routes
    mux := http.NewServeMux()
    mux.HandleFunc("POST /orders", handler.PlaceOrder)

    log.Println("Listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

> **Key Insight**: The `main` function is the outermost layer. It is the only file that imports
> every package. This is expected and desirable.

---

## Project Structure for Clean Architecture

### Recommended Layout

```
myapp/
├── cmd/
│   ├── server/          # HTTP server entry point
│   │   └── main.go
│   ├── worker/          # Background worker entry point
│   │   └── main.go
│   └── cli/             # CLI tool entry point
│       └── main.go
├── internal/
│   ├── domain/                  # INNERMOST LAYER
│   │   ├── order/
│   │   │   ├── order.go         # Order aggregate root (entity)
│   │   │   ├── order_item.go    # OrderItem entity
│   │   │   ├── money.go         # Money value object
│   │   │   ├── order_status.go  # OrderStatus value object (enum)
│   │   │   ├── events.go        # Domain events
│   │   │   ├── repository.go    # Repository interface (port)
│   │   │   ├── errors.go        # Domain-specific errors
│   │   │   └── service.go       # Domain service (pricing rules, etc.)
│   │   ├── customer/
│   │   │   ├── customer.go
│   │   │   ├── email.go         # Email value object
│   │   │   └── repository.go
│   │   ├── product/
│   │   │   ├── product.go
│   │   │   └── repository.go
│   │   └── shared/              # Shared kernel (cross-aggregate types)
│   │       ├── id.go
│   │       └── events.go
│   ├── application/             # USE CASE LAYER
│   │   ├── order/
│   │   │   ├── place_order.go       # PlaceOrder use case
│   │   │   ├── cancel_order.go      # CancelOrder use case
│   │   │   ├── get_order.go         # GetOrder query
│   │   │   └── dto.go               # Data transfer objects
│   │   └── ports/                   # Ports used by application layer
│   │       ├── event_publisher.go   # Event publishing port
│   │       └── payment_gateway.go   # Payment port
│   ├── infrastructure/          # OUTER LAYER (driven adapters)
│   │   ├── postgres/
│   │   │   ├── order_repo.go
│   │   │   ├── customer_repo.go
│   │   │   └── migrations/
│   │   ├── redis/
│   │   │   └── cache.go
│   │   ├── rabbitmq/
│   │   │   └── event_publisher.go
│   │   ├── stripe/
│   │   │   └── payment_gateway.go
│   │   └── email/
│   │       └── notifier.go
│   └── interfaces/              # OUTER LAYER (driving adapters)
│       ├── http/
│       │   ├── order_handler.go
│       │   ├── customer_handler.go
│       │   ├── middleware.go
│       │   ├── router.go
│       │   └── errors.go       # HTTP error mapping
│       ├── grpc/
│       │   ├── order_server.go
│       │   └── proto/
│       └── cli/
│           └── commands.go
├── pkg/                         # Public shared libraries (if any)
│   └── clock/
│       └── clock.go             # Abstraction for time.Now()
├── go.mod
└── go.sum
```

### Package Dependency Rules

The following import rules must be enforced (via linting or code review):

| Package | Can Import | Must NOT Import |
|---|---|---|
| `domain/*` | Only stdlib, `domain/shared` | `application`, `infrastructure`, `interfaces` |
| `application/*` | `domain/*`, stdlib | `infrastructure`, `interfaces` |
| `infrastructure/*` | `domain/*`, `application/ports`, stdlib, third-party libs | `interfaces` |
| `interfaces/*` | `application/*`, `domain/*` (for errors), stdlib, third-party libs | `infrastructure` |
| `cmd/*` | Everything | (nothing restricted -- this is the composition root) |

> **Tip**: Use the `depguard` linter (part of `golangci-lint`) to enforce these import rules
> automatically:
>
> ```yaml
> # .golangci.yml
> linters-settings:
>   depguard:
>     rules:
>       domain-layer:
>         files:
>           - "**/internal/domain/**"
>         deny:
>           - pkg: "myapp/internal/infrastructure"
>             desc: "Domain must not depend on infrastructure"
>           - pkg: "myapp/internal/interfaces"
>             desc: "Domain must not depend on interfaces"
>           - pkg: "myapp/internal/application"
>             desc: "Domain must not depend on application layer"
> ```

### The `internal/` Convention

Go's `internal` directory has special compiler-enforced visibility. Packages under `internal/`
cannot be imported by code outside the parent of `internal/`. This is perfect for Clean
Architecture because it prevents external consumers from reaching into your infrastructure layer.

---

## Domain-Driven Design Tactical Patterns in Go

### Entities

An **entity** is an object defined primarily by its identity, not its attributes. Two orders
with the same items but different IDs are different orders.

Key rules for entities in Go:

1. Make the struct fields unexported.
2. Provide a constructor that enforces invariants.
3. Expose behavior through methods, not getters/setters.
4. The entity decides whether a state transition is valid.

```go
// domain/order/order.go
package order

import (
    "errors"
    "time"
)

// OrderID is a strongly-typed identifier.
type OrderID string

func NewOrderID() OrderID {
    // In production, use a UUID library
    return OrderID(generateUUID())
}

// Order is the aggregate root entity.
type Order struct {
    id          OrderID
    customerID  string
    items       []OrderItem
    status      OrderStatus
    total       Money
    placedAt    time.Time
    cancelledAt *time.Time

    // Domain events collected during the lifetime of this aggregate
    events []DomainEvent
}

// NewOrder is the constructor. It enforces all creation-time invariants.
func NewOrder(id OrderID, customerID string, items []OrderItem) (*Order, error) {
    if customerID == "" {
        return nil, ErrCustomerIDRequired
    }
    if len(items) == 0 {
        return nil, ErrOrderMustHaveItems
    }

    o := &Order{
        id:         id,
        customerID: customerID,
        items:      items,
        status:     StatusPending,
        placedAt:   time.Now(),
    }
    o.recalculateTotal()

    // Record a domain event
    o.recordEvent(OrderPlacedEvent{
        OrderID:    string(id),
        CustomerID: customerID,
        Total:      o.total.Amount(),
        OccurredAt: o.placedAt,
    })

    return o, nil
}

// AddItem adds an item to the order. Only allowed while order is pending.
func (o *Order) AddItem(item OrderItem) error {
    if o.status != StatusPending {
        return ErrCannotModifyNonPendingOrder
    }

    // Check for duplicate product -- merge quantity if same product
    for i, existing := range o.items {
        if existing.ProductID() == item.ProductID() {
            o.items[i] = existing.WithAdditionalQuantity(item.Quantity())
            o.recalculateTotal()
            return nil
        }
    }

    o.items = append(o.items, item)
    o.recalculateTotal()
    return nil
}

// Cancel transitions the order to cancelled status.
func (o *Order) Cancel(reason string) error {
    if o.status == StatusShipped {
        return ErrCannotCancelShippedOrder
    }
    if o.status == StatusCancelled {
        return ErrOrderAlreadyCancelled
    }

    now := time.Now()
    o.status = StatusCancelled
    o.cancelledAt = &now

    o.recordEvent(OrderCancelledEvent{
        OrderID:    string(o.id),
        Reason:     reason,
        OccurredAt: now,
    })

    return nil
}

// Confirm transitions from pending to confirmed.
func (o *Order) Confirm() error {
    if o.status != StatusPending {
        return ErrInvalidStatusTransition
    }
    o.status = StatusConfirmed

    o.recordEvent(OrderConfirmedEvent{
        OrderID:    string(o.id),
        OccurredAt: time.Now(),
    })

    return nil
}

func (o *Order) recalculateTotal() {
    var total int64
    for _, item := range o.items {
        total += item.Subtotal().Amount()
    }
    o.total = NewMoney(total, "USD")
}

func (o *Order) recordEvent(event DomainEvent) {
    o.events = append(o.events, event)
}

// --- Read-only accessors ---

func (o *Order) ID() OrderID        { return o.id }
func (o *Order) CustomerID() string  { return o.customerID }
func (o *Order) Status() OrderStatus { return o.status }
func (o *Order) Total() Money        { return o.total }
func (o *Order) PlacedAt() time.Time { return o.placedAt }

// Items returns a copy of the items slice to prevent external mutation.
func (o *Order) Items() []OrderItem {
    copied := make([]OrderItem, len(o.items))
    copy(copied, o.items)
    return copied
}

// PullEvents returns and clears all accumulated domain events.
// This is called by the application layer after saving the aggregate.
func (o *Order) PullEvents() []DomainEvent {
    events := o.events
    o.events = nil
    return events
}
```

### Value Objects

A **value object** is defined by its attributes, not by identity. Two `Money` values of
$10.00 USD are equal and interchangeable. Value objects should be **immutable**.

```go
// domain/order/money.go
package order

import "fmt"

// Money is a value object representing a monetary amount.
// It is immutable -- all operations return a new Money.
type Money struct {
    amount   int64  // in smallest currency unit (cents)
    currency string
}

func NewMoney(amount int64, currency string) Money {
    return Money{amount: amount, currency: currency}
}

func (m Money) Amount() int64    { return m.amount }
func (m Money) Currency() string { return m.currency }

// Add returns a new Money that is the sum of m and other.
func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, fmt.Errorf("cannot add %s to %s: currency mismatch", m.currency, other.currency)
    }
    return NewMoney(m.amount+other.amount, m.currency), nil
}

// Multiply returns a new Money multiplied by a quantity.
func (m Money) Multiply(quantity int) Money {
    return NewMoney(m.amount*int64(quantity), m.currency)
}

// Equals checks value equality.
func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}

// String returns a human-readable representation.
func (m Money) String() string {
    dollars := m.amount / 100
    cents := m.amount % 100
    return fmt.Sprintf("%s %d.%02d", m.currency, dollars, cents)
}
```

```go
// domain/order/order_status.go
package order

// OrderStatus is a value object implemented as a typed string.
type OrderStatus string

const (
    StatusPending   OrderStatus = "PENDING"
    StatusConfirmed OrderStatus = "CONFIRMED"
    StatusShipped   OrderStatus = "SHIPPED"
    StatusDelivered OrderStatus = "DELIVERED"
    StatusCancelled OrderStatus = "CANCELLED"
)

// IsTerminal returns true if no further transitions are possible.
func (s OrderStatus) IsTerminal() bool {
    return s == StatusDelivered || s == StatusCancelled
}

// CanTransitionTo checks if a transition from s to target is valid.
func (s OrderStatus) CanTransitionTo(target OrderStatus) bool {
    allowed := map[OrderStatus][]OrderStatus{
        StatusPending:   {StatusConfirmed, StatusCancelled},
        StatusConfirmed: {StatusShipped, StatusCancelled},
        StatusShipped:   {StatusDelivered},
        StatusDelivered: {},
        StatusCancelled: {},
    }
    for _, t := range allowed[s] {
        if t == target {
            return true
        }
    }
    return false
}
```

```go
// domain/customer/email.go
package customer

import (
    "fmt"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

// Email is a value object that guarantees a valid email format.
type Email struct {
    address string
}

// NewEmail creates an Email value object, returning an error if the format is invalid.
func NewEmail(address string) (Email, error) {
    if !emailRegex.MatchString(address) {
        return Email{}, fmt.Errorf("invalid email address: %q", address)
    }
    return Email{address: address}, nil
}

func (e Email) String() string { return e.address }
func (e Email) Address() string { return e.address }

// Equals checks value equality.
func (e Email) Equals(other Email) bool {
    return e.address == other.address
}
```

### Key Differences Between Entities and Value Objects

| Aspect | Entity | Value Object |
|---|---|---|
| Identity | Has a unique ID | Defined by its attributes |
| Mutability | Can change state through methods | Immutable (operations return new values) |
| Equality | Compared by ID | Compared by all attributes |
| Lifecycle | Created, modified, persisted | Created, passed around, replaced |
| Example | `Order`, `Customer`, `Product` | `Money`, `Email`, `Address`, `OrderStatus` |

### Aggregates and Aggregate Roots

An **aggregate** is a cluster of entities and value objects that are treated as a single unit for
data changes. The **aggregate root** is the single entity through which all modifications to the
aggregate must pass.

Rules:
1. External objects may only hold a reference to the aggregate root.
2. All changes go through the root.
3. Aggregates are the unit of persistence (save/load the whole aggregate).
4. Aggregates are the transactional boundary.

```go
// Order is the aggregate root.
// OrderItem is an entity within the aggregate.
// Money is a value object within the aggregate.

// domain/order/order_item.go
package order

// OrderItem is an entity within the Order aggregate.
// It has its own identity (line number or product reference),
// but it cannot exist outside of an Order.
type OrderItem struct {
    productID   string
    productName string
    unitPrice   Money
    quantity    int
}

func NewOrderItem(productID, productName string, unitPrice Money, quantity int) (OrderItem, error) {
    if productID == "" {
        return OrderItem{}, ErrProductIDRequired
    }
    if quantity <= 0 {
        return OrderItem{}, ErrInvalidQuantity
    }
    return OrderItem{
        productID:   productID,
        productName: productName,
        unitPrice:   unitPrice,
        quantity:    quantity,
    }, nil
}

func (i OrderItem) ProductID() string   { return i.productID }
func (i OrderItem) ProductName() string { return i.productName }
func (i OrderItem) UnitPrice() Money    { return i.unitPrice }
func (i OrderItem) Quantity() int       { return i.quantity }

func (i OrderItem) Subtotal() Money {
    return i.unitPrice.Multiply(i.quantity)
}

// WithAdditionalQuantity returns a new OrderItem with increased quantity.
func (i OrderItem) WithAdditionalQuantity(qty int) OrderItem {
    return OrderItem{
        productID:   i.productID,
        productName: i.productName,
        unitPrice:   i.unitPrice,
        quantity:    i.quantity + qty,
    }
}
```

> **Warning**: Do not let external code reach into the aggregate and modify `OrderItem` directly.
> Always go through the `Order` aggregate root:
>
> ```go
> // WRONG: reaching into the aggregate
> order.Items()[0].quantity = 5
>
> // RIGHT: going through the aggregate root
> order.AddItem(newItem)
> ```

### Aggregate Design Guidelines

| Guideline | Rationale |
|---|---|
| Keep aggregates small | Large aggregates cause contention and performance issues |
| Reference other aggregates by ID only | Avoid loading the entire object graph |
| Use eventual consistency between aggregates | Use domain events to synchronize |
| One aggregate per transaction | Avoid distributed transactions |

### Domain Events

**Domain events** represent something meaningful that happened in the domain. They are named in
past tense using ubiquitous language.

```go
// domain/order/events.go
package order

import "time"

// DomainEvent is the marker interface for all domain events.
type DomainEvent interface {
    EventName() string
    OccurredAt() time.Time
}

// OrderPlacedEvent is raised when a new order is created.
type OrderPlacedEvent struct {
    OrderID    string
    CustomerID string
    Total      int64
    occurredAt time.Time
}

func (e OrderPlacedEvent) EventName() string      { return "order.placed" }
func (e OrderPlacedEvent) OccurredAt() time.Time   { return e.occurredAt }

// OrderConfirmedEvent is raised when an order is confirmed.
type OrderConfirmedEvent struct {
    OrderID    string
    occurredAt time.Time
}

func (e OrderConfirmedEvent) EventName() string    { return "order.confirmed" }
func (e OrderConfirmedEvent) OccurredAt() time.Time { return e.occurredAt }

// OrderCancelledEvent is raised when an order is cancelled.
type OrderCancelledEvent struct {
    OrderID    string
    Reason     string
    occurredAt time.Time
}

func (e OrderCancelledEvent) EventName() string    { return "order.cancelled" }
func (e OrderCancelledEvent) OccurredAt() time.Time { return e.occurredAt }
```

```go
// application/ports/event_publisher.go
package ports

import (
    "context"

    "myapp/internal/domain/order"
)

// EventPublisher is a port for publishing domain events to the outside world.
type EventPublisher interface {
    Publish(ctx context.Context, events []order.DomainEvent) error
}
```

```go
// infrastructure/rabbitmq/event_publisher.go
package rabbitmq

import (
    "context"
    "encoding/json"
    "fmt"

    amqp "github.com/rabbitmq/amqp091-go"

    "myapp/internal/domain/order"
)

type EventPublisher struct {
    channel *amqp.Channel
    exchange string
}

func NewEventPublisher(ch *amqp.Channel, exchange string) *EventPublisher {
    return &EventPublisher{channel: ch, exchange: exchange}
}

func (p *EventPublisher) Publish(ctx context.Context, events []order.DomainEvent) error {
    for _, event := range events {
        body, err := json.Marshal(event)
        if err != nil {
            return fmt.Errorf("marshaling event %s: %w", event.EventName(), err)
        }

        err = p.channel.PublishWithContext(ctx,
            p.exchange,
            event.EventName(), // routing key
            false,             // mandatory
            false,             // immediate
            amqp.Publishing{
                ContentType: "application/json",
                Body:        body,
            },
        )
        if err != nil {
            return fmt.Errorf("publishing event %s: %w", event.EventName(), err)
        }
    }
    return nil
}
```

### The Repository Pattern

The repository is an abstraction that makes the aggregate appear as if it were an in-memory
collection. It hides persistence mechanics from the domain.

```go
// domain/order/repository.go
package order

import "context"

// Repository provides access to the Order aggregate persistence.
// This interface is defined in the DOMAIN layer.
type Repository interface {
    // FindByID retrieves an order by its ID.
    // Returns ErrOrderNotFound if the order does not exist.
    FindByID(ctx context.Context, id OrderID) (*Order, error)

    // FindByCustomerID retrieves all orders for a customer.
    FindByCustomerID(ctx context.Context, customerID string) ([]*Order, error)

    // Save persists the order (create or update).
    Save(ctx context.Context, order *Order) error

    // Delete removes an order from persistence.
    Delete(ctx context.Context, id OrderID) error
}
```

> **Tip**: The repository interface belongs in the domain layer, but its implementation
> belongs in the infrastructure layer. This is the dependency inversion at work.

**Repository Anti-Patterns to Avoid:**

```go
// WRONG: Generic repository (loses domain meaning)
type Repository[T any] interface {
    FindByID(ctx context.Context, id string) (T, error)
    FindAll(ctx context.Context) ([]T, error)
    Save(ctx context.Context, entity T) error
    Delete(ctx context.Context, id string) error
}

// WRONG: Leaking persistence details into the domain
type OrderRepository interface {
    FindBySQL(ctx context.Context, query string) ([]*Order, error)  // SQL leaks!
    FindByFilter(ctx context.Context, filter bson.M) ([]*Order, error) // MongoDB leaks!
}

// RIGHT: Domain-specific query methods with business meaning
type Repository interface {
    FindByID(ctx context.Context, id OrderID) (*Order, error)
    FindPendingOrdersOlderThan(ctx context.Context, age time.Duration) ([]*Order, error)
    FindByCustomerID(ctx context.Context, customerID string) ([]*Order, error)
}
```

### Domain Services

A **domain service** contains business logic that does not naturally belong to a single entity or
value object. It operates on domain concepts and has no knowledge of infrastructure.

```go
// domain/order/service.go
package order

// PricingService calculates pricing with business rules
// that span multiple entities (e.g., discount logic).
type PricingService struct {
    // No infrastructure dependencies! Only domain types.
}

func NewPricingService() *PricingService {
    return &PricingService{}
}

// CalculateOrderTotal applies discount rules and returns the final total.
func (s *PricingService) CalculateOrderTotal(items []OrderItem, customerTier string) Money {
    var subtotal int64
    for _, item := range items {
        subtotal += item.Subtotal().Amount()
    }

    discount := s.calculateDiscount(subtotal, customerTier)
    finalAmount := subtotal - discount

    return NewMoney(finalAmount, "USD")
}

func (s *PricingService) calculateDiscount(subtotal int64, customerTier string) int64 {
    switch customerTier {
    case "GOLD":
        return subtotal * 10 / 100 // 10% discount
    case "SILVER":
        return subtotal * 5 / 100 // 5% discount
    default:
        return 0
    }
}
```

```go
// domain/order/availability_service.go
package order

// AvailabilityChecker is an interface defined in the domain
// for checking product availability. The implementation will
// be in the infrastructure layer.
type AvailabilityChecker interface {
    IsAvailable(productID string, quantity int) (bool, error)
}

// OrderValidationService is a domain service that validates
// whether an order can be placed.
type OrderValidationService struct {
    checker AvailabilityChecker
}

func NewOrderValidationService(checker AvailabilityChecker) *OrderValidationService {
    return &OrderValidationService{checker: checker}
}

func (s *OrderValidationService) ValidateOrder(items []OrderItem) error {
    for _, item := range items {
        available, err := s.checker.IsAvailable(item.ProductID(), item.Quantity())
        if err != nil {
            return err
        }
        if !available {
            return NewInsufficientStockError(item.ProductID(), item.Quantity())
        }
    }
    return nil
}
```

### Application Services vs Domain Services

This distinction confuses many developers. Here is the clear difference:

| Aspect | Domain Service | Application Service |
|---|---|---|
| **Lives in** | `domain/` layer | `application/` layer |
| **Knows about** | Domain types only | Domain types + ports |
| **Does** | Business rules that span entities | Orchestrates a use case workflow |
| **Example** | Pricing calculation, order validation | Place order, cancel order |
| **Has side effects** | No (pure business logic) | Yes (calls repos, publishes events) |
| **Calls infrastructure** | Never directly | Through ports/interfaces |

```go
// application/order/place_order.go
package order

import (
    "context"
    "fmt"

    domain "myapp/internal/domain/order"
    "myapp/internal/application/ports"
)

// PlaceOrderUseCase is an APPLICATION SERVICE.
// It orchestrates the workflow of placing an order.
type PlaceOrderUseCase struct {
    orderRepo      domain.Repository
    pricingService *domain.PricingService
    validator      *domain.OrderValidationService
    eventPublisher ports.EventPublisher
}

func NewPlaceOrderUseCase(
    repo domain.Repository,
    pricing *domain.PricingService,
    validator *domain.OrderValidationService,
    publisher ports.EventPublisher,
) *PlaceOrderUseCase {
    return &PlaceOrderUseCase{
        orderRepo:      repo,
        pricingService: pricing,
        validator:      validator,
        eventPublisher: publisher,
    }
}

type PlaceOrderCommand struct {
    CustomerID string
    Items      []PlaceOrderItemDTO
}

type PlaceOrderItemDTO struct {
    ProductID   string
    ProductName string
    UnitPrice   int64
    Currency    string
    Quantity    int
}

type PlaceOrderResult struct {
    OrderID string
    Total   int64
}

// Execute runs the "Place Order" use case.
func (uc *PlaceOrderUseCase) Execute(ctx context.Context, cmd PlaceOrderCommand) (*PlaceOrderResult, error) {
    // 1. Convert DTOs to domain objects
    items := make([]domain.OrderItem, 0, len(cmd.Items))
    for _, dto := range cmd.Items {
        item, err := domain.NewOrderItem(
            dto.ProductID,
            dto.ProductName,
            domain.NewMoney(dto.UnitPrice, dto.Currency),
            dto.Quantity,
        )
        if err != nil {
            return nil, fmt.Errorf("invalid order item: %w", err)
        }
        items = append(items, item)
    }

    // 2. Validate using DOMAIN SERVICE
    if err := uc.validator.ValidateOrder(items); err != nil {
        return nil, err
    }

    // 3. Create the aggregate (business rules enforced in constructor)
    orderID := domain.NewOrderID()
    order, err := domain.NewOrder(orderID, cmd.CustomerID, items)
    if err != nil {
        return nil, err
    }

    // 4. Persist using REPOSITORY
    if err := uc.orderRepo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("saving order: %w", err)
    }

    // 5. Publish domain events
    events := order.PullEvents()
    if err := uc.eventPublisher.Publish(ctx, events); err != nil {
        // Log but don't fail -- event publishing can be retried
        // In production, use an outbox pattern for reliability
        fmt.Printf("warning: failed to publish events: %v\n", err)
    }

    return &PlaceOrderResult{
        OrderID: string(orderID),
        Total:   order.Total().Amount(),
    }, nil
}
```

---

## Bounded Contexts and Context Mapping

### What Is a Bounded Context?

A **bounded context** is a boundary within which a particular domain model is defined and
applicable. The same real-world concept (e.g., "customer") may have different representations
in different contexts.

```
+-------------------+     +-------------------+     +-------------------+
|  Order Context    |     |  Billing Context  |     |  Shipping Context |
|                   |     |                   |     |                   |
| Order             |     | Invoice           |     | Shipment          |
| OrderItem         |     | Payment           |     | Package           |
| Customer (ID+name)|     | Customer (ID+     |     | Customer (ID+     |
|                   |     |   billing addr)   |     |   shipping addr)  |
+-------------------+     +-------------------+     +-------------------+
```

In Go, bounded contexts are mapped to top-level directories or even separate modules:

```
myapp/
├── internal/
│   ├── ordering/          # Bounded Context: Order Management
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── interfaces/
│   ├── billing/           # Bounded Context: Billing
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── interfaces/
│   └── shipping/          # Bounded Context: Shipping
│       ├── domain/
│       ├── application/
│       ├── infrastructure/
│       └── interfaces/
```

Or in a larger organization, each bounded context might be a separate Go module or microservice:

```
github.com/company/ordering-service/
github.com/company/billing-service/
github.com/company/shipping-service/
```

### Context Mapping Patterns

| Pattern | Description | When to Use |
|---|---|---|
| **Shared Kernel** | Two contexts share a common subset of the domain model | Tightly coupled teams, same codebase |
| **Customer/Supplier** | Upstream context provides what downstream needs | Upstream team accommodates downstream |
| **Conformist** | Downstream conforms to upstream's model as-is | No influence over upstream |
| **Anti-Corruption Layer** | Downstream translates upstream's model to its own | Protect your domain from external models |
| **Published Language** | A well-documented shared schema (e.g., protobuf) | API-first communication |
| **Open Host Service** | Upstream provides a general-purpose API | Multiple consumers with varied needs |

### Example: Context Communication via Events

```go
// ordering/domain/events.go
package domain

import "time"

// OrderPlacedEvent is published by the Ordering context.
type OrderPlacedEvent struct {
    OrderID    string    `json:"order_id"`
    CustomerID string    `json:"customer_id"`
    TotalCents int64     `json:"total_cents"`
    Currency   string    `json:"currency"`
    OccurredAt time.Time `json:"occurred_at"`
}
```

```go
// billing/application/event_handler.go
package application

import (
    "context"
    "encoding/json"
    "fmt"

    billingDomain "myapp/internal/billing/domain"
)

// OrderPlacedExternalEvent represents the event FROM the ordering context.
// This is NOT the same type as the ordering context's event -- it is
// our own representation of what we received.
type OrderPlacedExternalEvent struct {
    OrderID    string `json:"order_id"`
    CustomerID string `json:"customer_id"`
    TotalCents int64  `json:"total_cents"`
    Currency   string `json:"currency"`
}

type BillingEventHandler struct {
    invoiceService *billingDomain.InvoiceService
}

func (h *BillingEventHandler) HandleOrderPlaced(ctx context.Context, payload []byte) error {
    var event OrderPlacedExternalEvent
    if err := json.Unmarshal(payload, &event); err != nil {
        return fmt.Errorf("unmarshaling order placed event: %w", err)
    }

    // Translate the external event into our domain's language
    return h.invoiceService.CreateInvoiceForOrder(ctx,
        event.OrderID,
        event.CustomerID,
        billingDomain.NewAmount(event.TotalCents, event.Currency),
    )
}
```

---

## Anti-Corruption Layer

An **Anti-Corruption Layer (ACL)** is a translation boundary that prevents an external system's
model from leaking into your domain. This is crucial when integrating with legacy systems,
third-party APIs, or other bounded contexts.

### Structure of an ACL

```
Your Domain  <-->  ACL (translator)  <-->  External System
```

### Example: Integrating with a Legacy Inventory System

```go
// domain/order/availability_checker.go (your port)
package order

// AvailabilityChecker is your domain's interface.
// It speaks your domain language.
type AvailabilityChecker interface {
    IsAvailable(productID string, quantity int) (bool, error)
}
```

```go
// infrastructure/legacy/inventory_client.go (the raw external client)
package legacy

import (
    "encoding/xml"
    "fmt"
    "net/http"
)

// LegacyInventoryResponse is the horrible XML response from the legacy system.
// This type should NEVER leak into your domain.
type LegacyInventoryResponse struct {
    XMLName    xml.Name `xml:"InventoryCheckResult"`
    ItemCode   string   `xml:"item_code"`
    QtyOnHand  int      `xml:"qty_on_hand"`
    QtyReserved int     `xml:"qty_reserved"`
    StatusCode string   `xml:"status_code"` // "A" = active, "D" = discontinued
    Warehouse  string   `xml:"wh_loc"`
}

type LegacyInventoryClient struct {
    baseURL string
    client  *http.Client
}

func NewLegacyInventoryClient(baseURL string) *LegacyInventoryClient {
    return &LegacyInventoryClient{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

func (c *LegacyInventoryClient) CheckStock(itemCode string) (*LegacyInventoryResponse, error) {
    url := fmt.Sprintf("%s/api/stock?item=%s", c.baseURL, itemCode)
    resp, err := c.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result LegacyInventoryResponse
    if err := xml.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    return &result, nil
}
```

```go
// infrastructure/legacy/acl_availability_adapter.go (the ACL)
package legacy

import (
    "fmt"

    "myapp/internal/domain/order"
)

// AvailabilityAdapter is the Anti-Corruption Layer.
// It translates between the legacy system's model and your domain's model.
type AvailabilityAdapter struct {
    legacyClient *LegacyInventoryClient
    skuMapper    *SKUMapper // Maps your product IDs to legacy item codes
}

func NewAvailabilityAdapter(client *LegacyInventoryClient, mapper *SKUMapper) *AvailabilityAdapter {
    return &AvailabilityAdapter{
        legacyClient: client,
        skuMapper:    mapper,
    }
}

// IsAvailable implements order.AvailabilityChecker.
// This is where the translation happens.
func (a *AvailabilityAdapter) IsAvailable(productID string, quantity int) (bool, error) {
    // Step 1: Translate our product ID to legacy item code
    legacyCode, err := a.skuMapper.ToLegacyCode(productID)
    if err != nil {
        return false, fmt.Errorf("mapping product %s to legacy code: %w", productID, err)
    }

    // Step 2: Call the legacy system
    result, err := a.legacyClient.CheckStock(legacyCode)
    if err != nil {
        return false, fmt.Errorf("checking legacy inventory: %w", err)
    }

    // Step 3: Translate the legacy response to our domain's concept
    if result.StatusCode == "D" {
        return false, nil // Discontinued = not available
    }
    availableQty := result.QtyOnHand - result.QtyReserved
    return availableQty >= quantity, nil
}

// Compile-time check that AvailabilityAdapter implements the domain interface.
var _ order.AvailabilityChecker = (*AvailabilityAdapter)(nil)
```

```go
// infrastructure/legacy/sku_mapper.go
package legacy

import "fmt"

// SKUMapper translates between your product IDs and legacy system codes.
// This mapping might come from a database or configuration file.
type SKUMapper struct {
    mappings map[string]string // productID -> legacyItemCode
}

func NewSKUMapper(mappings map[string]string) *SKUMapper {
    return &SKUMapper{mappings: mappings}
}

func (m *SKUMapper) ToLegacyCode(productID string) (string, error) {
    code, ok := m.mappings[productID]
    if !ok {
        return "", fmt.Errorf("no legacy mapping for product %s", productID)
    }
    return code, nil
}
```

> **Tip**: The ACL is particularly valuable during migrations. You can swap the legacy system
> for a new one by changing only the adapter, with zero changes to your domain code.

---

## Ubiquitous Language in Code

**Ubiquitous language** means that the code uses the same terms that domain experts use. If the
business says "place an order," your code should have `PlaceOrder`, not `CreateOrder` or
`InsertOrder`.

### Good vs Bad Naming

| Business Term | Good (Ubiquitous Language) | Bad (Technical Jargon) |
|---|---|---|
| Place an order | `PlaceOrder()` | `CreateOrder()`, `InsertOrder()` |
| Cancel an order | `CancelOrder()` | `DeleteOrder()`, `SetStatusCancelled()` |
| Ship an order | `ShipOrder()` | `UpdateOrderStatus("shipped")` |
| Customer tier | `CustomerTier` | `CustomerLevel`, `CustomerType` |
| Order line item | `OrderItem` | `OrderDetail`, `OrderLine` |
| Apply discount | `ApplyDiscount()` | `SetDiscount()`, `UpdatePrice()` |

### Ubiquitous Language in Method Names

```go
// GOOD: Methods read like domain operations
func (o *Order) Place() error { ... }
func (o *Order) Confirm() error { ... }
func (o *Order) Ship(trackingNumber string) error { ... }
func (o *Order) Deliver() error { ... }
func (o *Order) Cancel(reason string) error { ... }
func (o *Order) Refund() error { ... }

// BAD: Generic CRUD methods
func (o *Order) Update(status string) error { ... }
func (o *Order) SetStatus(s OrderStatus) { ... }
func (o *Order) Save() error { ... }
```

### Ubiquitous Language in Package Names

```go
// GOOD: packages named after domain concepts
import "myapp/internal/domain/order"
import "myapp/internal/domain/customer"
import "myapp/internal/domain/inventory"
import "myapp/internal/domain/shipping"

// BAD: packages named after technical concerns
import "myapp/internal/models"
import "myapp/internal/entities"
import "myapp/internal/types"
```

### Ubiquitous Language in Error Messages

```go
// GOOD: domain-meaningful errors
var ErrOrderAlreadyShipped = errors.New("order has already been shipped")
var ErrInsufficientStock = errors.New("insufficient stock for requested quantity")
var ErrCustomerSuspended = errors.New("customer account is suspended")

// BAD: technical errors
var ErrInvalidState = errors.New("invalid state transition")
var ErrConstraintViolation = errors.New("constraint violation")
```

---

## Error Handling in the Domain Layer

Domain errors are a critical part of the domain model. They represent business rule violations and
should be distinguishable from infrastructure errors.

### Defining Domain Errors

```go
// domain/order/errors.go
package order

import (
    "errors"
    "fmt"
)

// Sentinel errors for common domain violations.
var (
    ErrOrderNotFound              = errors.New("order not found")
    ErrCustomerIDRequired         = errors.New("customer ID is required")
    ErrOrderMustHaveItems         = errors.New("order must have at least one item")
    ErrCannotModifyNonPendingOrder = errors.New("cannot modify a non-pending order")
    ErrCannotCancelShippedOrder   = errors.New("cannot cancel an order that has been shipped")
    ErrOrderAlreadyCancelled      = errors.New("order has already been cancelled")
    ErrInvalidStatusTransition    = errors.New("invalid order status transition")
    ErrProductIDRequired          = errors.New("product ID is required")
    ErrInvalidQuantity            = errors.New("quantity must be positive")
)

// InsufficientStockError is a rich domain error with context.
type InsufficientStockError struct {
    ProductID         string
    RequestedQuantity int
}

func NewInsufficientStockError(productID string, requested int) *InsufficientStockError {
    return &InsufficientStockError{
        ProductID:         productID,
        RequestedQuantity: requested,
    }
}

func (e *InsufficientStockError) Error() string {
    return fmt.Sprintf("insufficient stock for product %s (requested %d)",
        e.ProductID, e.RequestedQuantity)
}

// OrderLimitExceededError is raised when a customer exceeds their order limit.
type OrderLimitExceededError struct {
    CustomerID   string
    CurrentCount int
    MaxAllowed   int
}

func (e *OrderLimitExceededError) Error() string {
    return fmt.Sprintf("customer %s has reached the maximum order limit (%d/%d)",
        e.CustomerID, e.CurrentCount, e.MaxAllowed)
}
```

### Classifying Domain Errors

A useful pattern is to classify domain errors so that outer layers can decide how to handle them:

```go
// domain/shared/errors.go
package shared

// DomainErrorType classifies domain errors for outer layers.
type DomainErrorType int

const (
    ErrorTypeUnknown      DomainErrorType = iota
    ErrorTypeNotFound                     // Resource not found
    ErrorTypeValidation                   // Input validation failed
    ErrorTypeConflict                     // Business rule conflict
    ErrorTypeAuthorization                // Not allowed
)

// DomainError wraps a domain error with a classification.
type DomainError struct {
    Type    DomainErrorType
    Message string
    Err     error
}

func (e *DomainError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *DomainError) Unwrap() error { return e.Err }

// Constructor helpers
func NewNotFoundError(msg string) *DomainError {
    return &DomainError{Type: ErrorTypeNotFound, Message: msg}
}

func NewValidationError(msg string, err error) *DomainError {
    return &DomainError{Type: ErrorTypeValidation, Message: msg, Err: err}
}

func NewConflictError(msg string) *DomainError {
    return &DomainError{Type: ErrorTypeConflict, Message: msg}
}
```

### Mapping Domain Errors to HTTP Status Codes

```go
// interfaces/http/errors.go
package http

import (
    "errors"
    "net/http"

    "myapp/internal/domain/order"
    "myapp/internal/domain/shared"
)

func handleError(w http.ResponseWriter, err error) {
    // Check for classified domain errors first
    var domainErr *shared.DomainError
    if errors.As(err, &domainErr) {
        switch domainErr.Type {
        case shared.ErrorTypeNotFound:
            http.Error(w, domainErr.Message, http.StatusNotFound)
        case shared.ErrorTypeValidation:
            http.Error(w, domainErr.Message, http.StatusBadRequest)
        case shared.ErrorTypeConflict:
            http.Error(w, domainErr.Message, http.StatusConflict)
        case shared.ErrorTypeAuthorization:
            http.Error(w, domainErr.Message, http.StatusForbidden)
        default:
            http.Error(w, "internal server error", http.StatusInternalServerError)
        }
        return
    }

    // Check for specific domain sentinel errors
    switch {
    case errors.Is(err, order.ErrOrderNotFound):
        http.Error(w, err.Error(), http.StatusNotFound)
    case errors.Is(err, order.ErrCustomerIDRequired),
        errors.Is(err, order.ErrOrderMustHaveItems),
        errors.Is(err, order.ErrInvalidQuantity):
        http.Error(w, err.Error(), http.StatusBadRequest)
    case errors.Is(err, order.ErrCannotCancelShippedOrder),
        errors.Is(err, order.ErrOrderAlreadyCancelled):
        http.Error(w, err.Error(), http.StatusConflict)
    default:
        // Don't leak internal errors to clients
        http.Error(w, "internal server error", http.StatusInternalServerError)
    }
}
```

> **Warning**: Never expose infrastructure errors (database connection failures, etc.) to clients.
> Only domain errors should produce meaningful error messages in HTTP responses.

---

## Testing Each Layer Independently

One of the greatest benefits of Clean Architecture is testability. Each layer can be tested in
isolation by mocking its dependencies.

### Testing the Domain Layer

Domain layer tests require **no mocks at all** because the domain has no external dependencies.

```go
// domain/order/order_test.go
package order_test

import (
    "testing"

    "myapp/internal/domain/order"
)

func TestNewOrder_RequiresCustomerID(t *testing.T) {
    _, err := order.NewOrder(order.NewOrderID(), "", []order.OrderItem{})
    if err != order.ErrCustomerIDRequired {
        t.Errorf("expected ErrCustomerIDRequired, got %v", err)
    }
}

func TestNewOrder_RequiresAtLeastOneItem(t *testing.T) {
    _, err := order.NewOrder(order.NewOrderID(), "cust-1", []order.OrderItem{})
    if err != order.ErrOrderMustHaveItems {
        t.Errorf("expected ErrOrderMustHaveItems, got %v", err)
    }
}

func TestNewOrder_CalculatesTotal(t *testing.T) {
    item1, _ := order.NewOrderItem("prod-1", "Widget", order.NewMoney(1000, "USD"), 2)
    item2, _ := order.NewOrderItem("prod-2", "Gadget", order.NewMoney(2500, "USD"), 1)

    o, err := order.NewOrder(order.NewOrderID(), "cust-1", []order.OrderItem{item1, item2})
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    // 2 * $10.00 + 1 * $25.00 = $45.00 = 4500 cents
    expectedTotal := order.NewMoney(4500, "USD")
    if !o.Total().Equals(expectedTotal) {
        t.Errorf("expected total %v, got %v", expectedTotal, o.Total())
    }
}

func TestOrder_Cancel_ShippedOrder_Fails(t *testing.T) {
    o := createShippedOrder(t) // helper that creates an order in Shipped status

    err := o.Cancel("changed my mind")
    if err != order.ErrCannotCancelShippedOrder {
        t.Errorf("expected ErrCannotCancelShippedOrder, got %v", err)
    }
}

func TestOrder_Cancel_PendingOrder_Succeeds(t *testing.T) {
    item, _ := order.NewOrderItem("prod-1", "Widget", order.NewMoney(1000, "USD"), 1)
    o, _ := order.NewOrder(order.NewOrderID(), "cust-1", []order.OrderItem{item})

    err := o.Cancel("changed my mind")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if o.Status() != order.StatusCancelled {
        t.Errorf("expected status CANCELLED, got %s", o.Status())
    }
}

func TestOrder_PlacedEvent_IsRaised(t *testing.T) {
    item, _ := order.NewOrderItem("prod-1", "Widget", order.NewMoney(1000, "USD"), 1)
    o, _ := order.NewOrder(order.NewOrderID(), "cust-1", []order.OrderItem{item})

    events := o.PullEvents()
    if len(events) != 1 {
        t.Fatalf("expected 1 event, got %d", len(events))
    }

    placedEvent, ok := events[0].(order.OrderPlacedEvent)
    if !ok {
        t.Fatalf("expected OrderPlacedEvent, got %T", events[0])
    }
    if placedEvent.OrderID != string(o.ID()) {
        t.Errorf("expected event order ID %s, got %s", o.ID(), placedEvent.OrderID)
    }
}

func TestMoney_Add_DifferentCurrencies_Fails(t *testing.T) {
    usd := order.NewMoney(1000, "USD")
    eur := order.NewMoney(1000, "EUR")

    _, err := usd.Add(eur)
    if err == nil {
        t.Error("expected error when adding different currencies")
    }
}

func TestOrderStatus_CanTransitionTo(t *testing.T) {
    tests := []struct {
        from     order.OrderStatus
        to       order.OrderStatus
        expected bool
    }{
        {order.StatusPending, order.StatusConfirmed, true},
        {order.StatusPending, order.StatusCancelled, true},
        {order.StatusPending, order.StatusShipped, false},
        {order.StatusConfirmed, order.StatusShipped, true},
        {order.StatusShipped, order.StatusCancelled, false},
        {order.StatusDelivered, order.StatusCancelled, false},
    }

    for _, tt := range tests {
        t.Run(string(tt.from)+"->"+string(tt.to), func(t *testing.T) {
            result := tt.from.CanTransitionTo(tt.to)
            if result != tt.expected {
                t.Errorf("expected %v, got %v", tt.expected, result)
            }
        })
    }
}
```

### Testing the Application Layer

Application layer tests mock the ports (repository, event publisher, etc.).

```go
// application/order/place_order_test.go
package order_test

import (
    "context"
    "testing"

    appOrder "myapp/internal/application/order"
    domain "myapp/internal/domain/order"
)

// --- Mock implementations ---

type mockOrderRepository struct {
    saved  []*domain.Order
    findFn func(ctx context.Context, id domain.OrderID) (*domain.Order, error)
}

func (m *mockOrderRepository) FindByID(ctx context.Context, id domain.OrderID) (*domain.Order, error) {
    if m.findFn != nil {
        return m.findFn(ctx, id)
    }
    return nil, domain.ErrOrderNotFound
}

func (m *mockOrderRepository) FindByCustomerID(ctx context.Context, customerID string) ([]*domain.Order, error) {
    return nil, nil
}

func (m *mockOrderRepository) Save(ctx context.Context, o *domain.Order) error {
    m.saved = append(m.saved, o)
    return nil
}

func (m *mockOrderRepository) Delete(ctx context.Context, id domain.OrderID) error {
    return nil
}

type mockAvailabilityChecker struct {
    available bool
}

func (m *mockAvailabilityChecker) IsAvailable(productID string, quantity int) (bool, error) {
    return m.available, nil
}

type mockEventPublisher struct {
    published []domain.DomainEvent
}

func (m *mockEventPublisher) Publish(ctx context.Context, events []domain.DomainEvent) error {
    m.published = append(m.published, events...)
    return nil
}

// --- Tests ---

func TestPlaceOrder_Success(t *testing.T) {
    repo := &mockOrderRepository{}
    checker := &mockAvailabilityChecker{available: true}
    publisher := &mockEventPublisher{}
    pricing := domain.NewPricingService()
    validator := domain.NewOrderValidationService(checker)

    uc := appOrder.NewPlaceOrderUseCase(repo, pricing, validator, publisher)

    cmd := appOrder.PlaceOrderCommand{
        CustomerID: "cust-123",
        Items: []appOrder.PlaceOrderItemDTO{
            {
                ProductID:   "prod-1",
                ProductName: "Widget",
                UnitPrice:   1000,
                Currency:    "USD",
                Quantity:    2,
            },
        },
    }

    result, err := uc.Execute(context.Background(), cmd)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    // Verify order was saved
    if len(repo.saved) != 1 {
        t.Fatalf("expected 1 saved order, got %d", len(repo.saved))
    }

    // Verify result
    if result.OrderID == "" {
        t.Error("expected non-empty order ID")
    }
    if result.Total != 2000 { // 2 * $10.00
        t.Errorf("expected total 2000, got %d", result.Total)
    }

    // Verify events were published
    if len(publisher.published) != 1 {
        t.Fatalf("expected 1 published event, got %d", len(publisher.published))
    }
}

func TestPlaceOrder_OutOfStock_Fails(t *testing.T) {
    repo := &mockOrderRepository{}
    checker := &mockAvailabilityChecker{available: false} // out of stock
    publisher := &mockEventPublisher{}
    pricing := domain.NewPricingService()
    validator := domain.NewOrderValidationService(checker)

    uc := appOrder.NewPlaceOrderUseCase(repo, pricing, validator, publisher)

    cmd := appOrder.PlaceOrderCommand{
        CustomerID: "cust-123",
        Items: []appOrder.PlaceOrderItemDTO{
            {ProductID: "prod-1", ProductName: "Widget", UnitPrice: 1000, Currency: "USD", Quantity: 2},
        },
    }

    _, err := uc.Execute(context.Background(), cmd)
    if err == nil {
        t.Fatal("expected error for out-of-stock item")
    }

    // Verify nothing was saved
    if len(repo.saved) != 0 {
        t.Error("expected no saved orders when stock is unavailable")
    }
}
```

### Testing the Infrastructure Layer (Integration Tests)

Infrastructure tests verify that your adapters correctly communicate with external systems.
These are integration tests and typically require the real dependency (database, etc.) running.

```go
// infrastructure/postgres/order_repo_test.go
//go:build integration

package postgres_test

import (
    "context"
    "database/sql"
    "os"
    "testing"

    _ "github.com/lib/pq"

    domain "myapp/internal/domain/order"
    "myapp/internal/infrastructure/postgres"
)

func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL not set, skipping integration test")
    }

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        t.Fatalf("opening database: %v", err)
    }

    // Clean up after test
    t.Cleanup(func() {
        db.Exec("DELETE FROM order_items")
        db.Exec("DELETE FROM orders")
        db.Close()
    })

    return db
}

func TestOrderRepository_SaveAndFind(t *testing.T) {
    db := setupTestDB(t)
    repo := postgres.NewOrderRepository(db)
    ctx := context.Background()

    // Create a domain order
    item, _ := domain.NewOrderItem("prod-1", "Widget", domain.NewMoney(1000, "USD"), 2)
    orderID := domain.NewOrderID()
    order, err := domain.NewOrder(orderID, "cust-123", []domain.OrderItem{item})
    if err != nil {
        t.Fatalf("creating order: %v", err)
    }

    // Save
    if err := repo.Save(ctx, order); err != nil {
        t.Fatalf("saving order: %v", err)
    }

    // Find
    found, err := repo.FindByID(ctx, orderID)
    if err != nil {
        t.Fatalf("finding order: %v", err)
    }

    // Verify
    if found.ID() != orderID {
        t.Errorf("expected ID %s, got %s", orderID, found.ID())
    }
    if found.CustomerID() != "cust-123" {
        t.Errorf("expected customer cust-123, got %s", found.CustomerID())
    }
    if found.Status() != domain.StatusPending {
        t.Errorf("expected status PENDING, got %s", found.Status())
    }
    if len(found.Items()) != 1 {
        t.Fatalf("expected 1 item, got %d", len(found.Items()))
    }
}

func TestOrderRepository_FindByID_NotFound(t *testing.T) {
    db := setupTestDB(t)
    repo := postgres.NewOrderRepository(db)

    _, err := repo.FindByID(context.Background(), "nonexistent-id")
    if err != domain.ErrOrderNotFound {
        t.Errorf("expected ErrOrderNotFound, got %v", err)
    }
}
```

### Testing the Interface Layer (HTTP Handlers)

```go
// interfaces/http/order_handler_test.go
package http_test

import (
    "bytes"
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "myapp/internal/application"
    handler "myapp/internal/interfaces/http"
)

// mockOrderService implements the application.OrderService interface.
type mockOrderService struct {
    placeOrderFn func(ctx context.Context, cmd application.PlaceOrderCommand) (string, error)
}

func (m *mockOrderService) PlaceOrder(ctx context.Context, cmd application.PlaceOrderCommand) (string, error) {
    return m.placeOrderFn(ctx, cmd)
}

func (m *mockOrderService) CancelOrder(ctx context.Context, orderID string) error {
    return nil
}

func (m *mockOrderService) GetOrder(ctx context.Context, orderID string) (*application.OrderDTO, error) {
    return nil, nil
}

func TestOrderHandler_PlaceOrder_Success(t *testing.T) {
    svc := &mockOrderService{
        placeOrderFn: func(ctx context.Context, cmd application.PlaceOrderCommand) (string, error) {
            return "ord-123", nil
        },
    }

    h := handler.NewOrderHandler(svc)

    body := `{
        "customer_id": "cust-1",
        "items": [{"product_id": "prod-1", "quantity": 2}]
    }`
    req := httptest.NewRequest(http.MethodPost, "/orders", bytes.NewBufferString(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    h.PlaceOrder(rec, req)

    if rec.Code != http.StatusCreated {
        t.Errorf("expected status 201, got %d", rec.Code)
    }

    var resp map[string]string
    json.NewDecoder(rec.Body).Decode(&resp)
    if resp["order_id"] != "ord-123" {
        t.Errorf("expected order_id ord-123, got %s", resp["order_id"])
    }
}

func TestOrderHandler_PlaceOrder_InvalidBody(t *testing.T) {
    svc := &mockOrderService{}
    h := handler.NewOrderHandler(svc)

    req := httptest.NewRequest(http.MethodPost, "/orders", bytes.NewBufferString("not json"))
    rec := httptest.NewRecorder()

    h.PlaceOrder(rec, req)

    if rec.Code != http.StatusBadRequest {
        t.Errorf("expected status 400, got %d", rec.Code)
    }
}
```

### Testing Pyramid for Clean Architecture

```
         /\
        /  \
       / E2E \        <- Few: full system tests (HTTP -> DB -> Events)
      /--------\
     / Integration \   <- Some: infrastructure adapter tests (real DB)
    /--------------\
   /  Application   \  <- Many: use case tests with mocked ports
  /------------------\
 /     Domain         \ <- Many: pure unit tests, no mocks needed
/______________________\
```

---

## When Clean Architecture / DDD Is Overkill

Not every project benefits from the full ceremony of Clean Architecture and DDD. Here is a
decision framework:

### Signs You DO Need Clean Architecture / DDD

- Complex business logic with many rules and edge cases
- Multiple external systems to integrate with
- Long-lived project (years of expected maintenance)
- Large team (3+ developers)
- Business logic that changes frequently
- Need to support multiple delivery mechanisms (HTTP, gRPC, CLI, events)

### Signs It Is Overkill

- Simple CRUD application
- Prototype or throwaway code
- Small team (1-2 developers) with a well-understood domain
- Project with a short expected lifetime
- Performance-critical paths where the abstraction layers add unacceptable overhead
- Configuration management tools or scripts

### The Pragmatic Spectrum

| Complexity | Recommended Approach |
|---|---|
| Simple CRUD | Flat structure, handlers call DB directly |
| Moderate business logic | Service layer + repository pattern |
| Complex domain | Clean Architecture with DDD tactical patterns |
| Very complex, multiple teams | Full DDD with bounded contexts, event-driven |

### Example: When a Flat Structure Is Fine

```go
// For a simple TODO app, this is perfectly fine:
// cmd/server/main.go
package main

import (
    "database/sql"
    "encoding/json"
    "net/http"
)

func main() {
    db, _ := sql.Open("sqlite3", "todos.db")

    http.HandleFunc("POST /todos", func(w http.ResponseWriter, r *http.Request) {
        var todo struct {
            Title string `json:"title"`
        }
        json.NewDecoder(r.Body).Decode(&todo)
        db.Exec("INSERT INTO todos (title) VALUES (?)", todo.Title)
        w.WriteHeader(http.StatusCreated)
    })

    http.HandleFunc("GET /todos", func(w http.ResponseWriter, r *http.Request) {
        rows, _ := db.Query("SELECT id, title, done FROM todos")
        defer rows.Close()
        // ... scan and return JSON
    })

    http.ListenAndServe(":8080", nil)
}
```

> **Tip**: You can always refactor toward Clean Architecture later as the application grows.
> Start simple, extract layers when complexity demands it. The key insight from DDD is
> "start with the domain model in conversations with domain experts" -- you can do that
> without the full architectural ceremony.

---

## Complete Example: Order Management System

Let us build a complete, working order management system that demonstrates all of these
concepts together.

### Project Structure

```
ordersystem/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── domain/
│   │   ├── order/
│   │   │   ├── order.go
│   │   │   ├── order_item.go
│   │   │   ├── money.go
│   │   │   ├── status.go
│   │   │   ├── events.go
│   │   │   ├── repository.go
│   │   │   ├── errors.go
│   │   │   └── pricing_service.go
│   │   └── customer/
│   │       ├── customer.go
│   │       ├── email.go
│   │       └── repository.go
│   ├── application/
│   │   └── order/
│   │       ├── place_order.go
│   │       ├── cancel_order.go
│   │       ├── get_order.go
│   │       ├── list_orders.go
│   │       └── dto.go
│   ├── infrastructure/
│   │   ├── memory/
│   │   │   ├── order_repo.go
│   │   │   └── customer_repo.go
│   │   └── logging/
│   │       └── event_publisher.go
│   └── interfaces/
│       └── http/
│           ├── router.go
│           ├── order_handler.go
│           └── error_mapper.go
├── go.mod
└── go.sum
```

### Domain Layer

```go
// internal/domain/order/money.go
package order

import "fmt"

type Money struct {
    amount   int64
    currency string
}

func NewMoney(amount int64, currency string) Money {
    return Money{amount: amount, currency: currency}
}

func ZeroMoney(currency string) Money {
    return Money{amount: 0, currency: currency}
}

func (m Money) Amount() int64    { return m.amount }
func (m Money) Currency() string { return m.currency }

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, fmt.Errorf("cannot add %s and %s", m.currency, other.currency)
    }
    return NewMoney(m.amount+other.amount, m.currency), nil
}

func (m Money) Multiply(qty int) Money {
    return NewMoney(m.amount*int64(qty), m.currency)
}

func (m Money) IsPositive() bool { return m.amount > 0 }

func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}

func (m Money) String() string {
    return fmt.Sprintf("%s %.2f", m.currency, float64(m.amount)/100)
}
```

```go
// internal/domain/order/status.go
package order

type Status string

const (
    StatusPending   Status = "PENDING"
    StatusConfirmed Status = "CONFIRMED"
    StatusShipped   Status = "SHIPPED"
    StatusDelivered Status = "DELIVERED"
    StatusCancelled Status = "CANCELLED"
)

var validTransitions = map[Status][]Status{
    StatusPending:   {StatusConfirmed, StatusCancelled},
    StatusConfirmed: {StatusShipped, StatusCancelled},
    StatusShipped:   {StatusDelivered},
    StatusDelivered: {},
    StatusCancelled: {},
}

func (s Status) CanTransitionTo(target Status) bool {
    for _, allowed := range validTransitions[s] {
        if allowed == target {
            return true
        }
    }
    return false
}

func (s Status) IsTerminal() bool {
    return s == StatusDelivered || s == StatusCancelled
}

func (s Status) String() string { return string(s) }
```

```go
// internal/domain/order/errors.go
package order

import (
    "errors"
    "fmt"
)

var (
    ErrNotFound                = errors.New("order not found")
    ErrCustomerIDRequired      = errors.New("customer ID is required")
    ErrMustHaveItems           = errors.New("order must contain at least one item")
    ErrCannotModifyOrder       = errors.New("order cannot be modified in its current state")
    ErrCannotCancelOrder       = errors.New("order cannot be cancelled in its current state")
    ErrAlreadyCancelled        = errors.New("order is already cancelled")
    ErrInvalidStatusTransition = errors.New("invalid status transition")
    ErrProductIDRequired       = errors.New("product ID is required")
    ErrInvalidQuantity         = errors.New("quantity must be greater than zero")
    ErrInvalidPrice            = errors.New("unit price must be positive")
)

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}
```

```go
// internal/domain/order/events.go
package order

import "time"

type DomainEvent interface {
    EventName() string
    OccurredOn() time.Time
}

type OrderPlaced struct {
    ID         string
    CustomerID string
    Total      int64
    Currency   string
    Timestamp  time.Time
}

func (e OrderPlaced) EventName() string    { return "order.placed" }
func (e OrderPlaced) OccurredOn() time.Time { return e.Timestamp }

type OrderCancelled struct {
    ID        string
    Reason    string
    Timestamp time.Time
}

func (e OrderCancelled) EventName() string    { return "order.cancelled" }
func (e OrderCancelled) OccurredOn() time.Time { return e.Timestamp }

type OrderConfirmed struct {
    ID        string
    Timestamp time.Time
}

func (e OrderConfirmed) EventName() string    { return "order.confirmed" }
func (e OrderConfirmed) OccurredOn() time.Time { return e.Timestamp }
```

```go
// internal/domain/order/order_item.go
package order

type Item struct {
    productID   string
    productName string
    unitPrice   Money
    quantity    int
}

func NewItem(productID, productName string, unitPrice Money, quantity int) (Item, error) {
    if productID == "" {
        return Item{}, ErrProductIDRequired
    }
    if quantity <= 0 {
        return Item{}, ErrInvalidQuantity
    }
    if !unitPrice.IsPositive() {
        return Item{}, ErrInvalidPrice
    }
    return Item{
        productID:   productID,
        productName: productName,
        unitPrice:   unitPrice,
        quantity:    quantity,
    }, nil
}

func (i Item) ProductID() string   { return i.productID }
func (i Item) ProductName() string { return i.productName }
func (i Item) UnitPrice() Money    { return i.unitPrice }
func (i Item) Quantity() int       { return i.quantity }

func (i Item) Subtotal() Money {
    return i.unitPrice.Multiply(i.quantity)
}

func (i Item) WithQuantity(qty int) Item {
    return Item{
        productID:   i.productID,
        productName: i.productName,
        unitPrice:   i.unitPrice,
        quantity:    qty,
    }
}
```

```go
// internal/domain/order/order.go
package order

import (
    "fmt"
    "time"

    "github.com/google/uuid"
)

type ID string

func NewID() ID {
    return ID(uuid.New().String())
}

// Order is the aggregate root.
type Order struct {
    id          ID
    customerID  string
    items       []Item
    status      Status
    total       Money
    placedAt    time.Time
    confirmedAt *time.Time
    cancelledAt *time.Time
    cancelReason string

    events []DomainEvent
}

// Place creates a new order. This is the only way to create an Order.
func Place(id ID, customerID string, items []Item) (*Order, error) {
    if customerID == "" {
        return nil, ErrCustomerIDRequired
    }
    if len(items) == 0 {
        return nil, ErrMustHaveItems
    }

    now := time.Now()
    o := &Order{
        id:         id,
        customerID: customerID,
        items:      make([]Item, len(items)),
        status:     StatusPending,
        placedAt:   now,
    }
    copy(o.items, items)
    o.recalculateTotal()

    o.raise(OrderPlaced{
        ID:         string(id),
        CustomerID: customerID,
        Total:      o.total.Amount(),
        Currency:   o.total.Currency(),
        Timestamp:  now,
    })

    return o, nil
}

// Reconstitute rebuilds an Order from persistence without triggering events
// or validation. This is used by the repository layer only.
func Reconstitute(
    id ID,
    customerID string,
    items []Item,
    status Status,
    total Money,
    placedAt time.Time,
    confirmedAt *time.Time,
    cancelledAt *time.Time,
    cancelReason string,
) *Order {
    return &Order{
        id:           id,
        customerID:   customerID,
        items:        items,
        status:       status,
        total:        total,
        placedAt:     placedAt,
        confirmedAt:  confirmedAt,
        cancelledAt:  cancelledAt,
        cancelReason: cancelReason,
    }
}

// Confirm transitions the order to confirmed status.
func (o *Order) Confirm() error {
    if !o.status.CanTransitionTo(StatusConfirmed) {
        return fmt.Errorf("%w: cannot transition from %s to CONFIRMED", ErrInvalidStatusTransition, o.status)
    }
    now := time.Now()
    o.status = StatusConfirmed
    o.confirmedAt = &now

    o.raise(OrderConfirmed{
        ID:        string(o.id),
        Timestamp: now,
    })
    return nil
}

// Cancel transitions the order to cancelled status.
func (o *Order) Cancel(reason string) error {
    if o.status == StatusCancelled {
        return ErrAlreadyCancelled
    }
    if !o.status.CanTransitionTo(StatusCancelled) {
        return fmt.Errorf("%w: cannot cancel order in %s status", ErrCannotCancelOrder, o.status)
    }

    now := time.Now()
    o.status = StatusCancelled
    o.cancelledAt = &now
    o.cancelReason = reason

    o.raise(OrderCancelled{
        ID:        string(o.id),
        Reason:    reason,
        Timestamp: now,
    })
    return nil
}

// AddItem adds an item to a pending order.
func (o *Order) AddItem(item Item) error {
    if o.status != StatusPending {
        return ErrCannotModifyOrder
    }

    // Merge if same product
    for i, existing := range o.items {
        if existing.ProductID() == item.ProductID() {
            o.items[i] = existing.WithQuantity(existing.Quantity() + item.Quantity())
            o.recalculateTotal()
            return nil
        }
    }

    o.items = append(o.items, item)
    o.recalculateTotal()
    return nil
}

// RemoveItem removes an item by product ID from a pending order.
func (o *Order) RemoveItem(productID string) error {
    if o.status != StatusPending {
        return ErrCannotModifyOrder
    }

    for i, item := range o.items {
        if item.ProductID() == productID {
            o.items = append(o.items[:i], o.items[i+1:]...)
            o.recalculateTotal()
            return nil
        }
    }
    return fmt.Errorf("product %s not found in order", productID)
}

func (o *Order) recalculateTotal() {
    total := ZeroMoney("USD")
    for _, item := range o.items {
        sum, _ := total.Add(item.Subtotal())
        total = sum
    }
    o.total = total
}

func (o *Order) raise(event DomainEvent) {
    o.events = append(o.events, event)
}

// PullEvents returns accumulated events and clears the internal list.
func (o *Order) PullEvents() []DomainEvent {
    events := o.events
    o.events = nil
    return events
}

// --- Accessors ---

func (o *Order) ID() ID              { return o.id }
func (o *Order) CustomerID() string   { return o.customerID }
func (o *Order) Status() Status       { return o.status }
func (o *Order) Total() Money         { return o.total }
func (o *Order) PlacedAt() time.Time  { return o.placedAt }
func (o *Order) CancelReason() string { return o.cancelReason }

func (o *Order) Items() []Item {
    copied := make([]Item, len(o.items))
    copy(copied, o.items)
    return copied
}

func (o *Order) ItemCount() int {
    total := 0
    for _, item := range o.items {
        total += item.Quantity()
    }
    return total
}
```

```go
// internal/domain/order/repository.go
package order

import "context"

type Repository interface {
    FindByID(ctx context.Context, id ID) (*Order, error)
    FindByCustomerID(ctx context.Context, customerID string) ([]*Order, error)
    FindAll(ctx context.Context) ([]*Order, error)
    Save(ctx context.Context, order *Order) error
    Delete(ctx context.Context, id ID) error
}
```

```go
// internal/domain/order/pricing_service.go
package order

// PricingService is a DOMAIN SERVICE that encapsulates pricing rules.
type PricingService struct{}

func NewPricingService() *PricingService {
    return &PricingService{}
}

// ApplyDiscount calculates the discounted total based on customer tier.
func (s *PricingService) ApplyDiscount(total Money, customerTier string) Money {
    var discountPercent int64
    switch customerTier {
    case "PLATINUM":
        discountPercent = 15
    case "GOLD":
        discountPercent = 10
    case "SILVER":
        discountPercent = 5
    default:
        discountPercent = 0
    }

    discountAmount := total.Amount() * discountPercent / 100
    return NewMoney(total.Amount()-discountAmount, total.Currency())
}
```

```go
// internal/domain/customer/email.go
package customer

import (
    "fmt"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

type Email struct {
    address string
}

func NewEmail(address string) (Email, error) {
    if !emailRegex.MatchString(address) {
        return Email{}, fmt.Errorf("invalid email: %q", address)
    }
    return Email{address: address}, nil
}

func (e Email) String() string  { return e.address }
func (e Email) Address() string { return e.address }
```

```go
// internal/domain/customer/customer.go
package customer

import "time"

type ID string

type Customer struct {
    id        ID
    name      string
    email     Email
    tier      string // "STANDARD", "SILVER", "GOLD", "PLATINUM"
    createdAt time.Time
}

func NewCustomer(id ID, name string, email Email) *Customer {
    return &Customer{
        id:        id,
        name:      name,
        email:     email,
        tier:      "STANDARD",
        createdAt: time.Now(),
    }
}

func ReconstructCustomer(id ID, name string, email Email, tier string, createdAt time.Time) *Customer {
    return &Customer{id: id, name: name, email: email, tier: tier, createdAt: createdAt}
}

func (c *Customer) ID() ID         { return c.id }
func (c *Customer) Name() string   { return c.name }
func (c *Customer) Email() Email   { return c.email }
func (c *Customer) Tier() string   { return c.tier }
```

```go
// internal/domain/customer/repository.go
package customer

import "context"

type Repository interface {
    FindByID(ctx context.Context, id ID) (*Customer, error)
}
```

### Application Layer

```go
// internal/application/order/dto.go
package order

import "time"

// --- Command DTOs (input) ---

type PlaceOrderCommand struct {
    CustomerID string              `json:"customer_id"`
    Items      []PlaceOrderItemDTO `json:"items"`
}

type PlaceOrderItemDTO struct {
    ProductID   string `json:"product_id"`
    ProductName string `json:"product_name"`
    UnitPrice   int64  `json:"unit_price"`
    Currency    string `json:"currency"`
    Quantity    int    `json:"quantity"`
}

type CancelOrderCommand struct {
    OrderID string `json:"order_id"`
    Reason  string `json:"reason"`
}

// --- Result DTOs (output) ---

type OrderDTO struct {
    ID           string         `json:"id"`
    CustomerID   string         `json:"customer_id"`
    CustomerName string         `json:"customer_name,omitempty"`
    Status       string         `json:"status"`
    Items        []OrderItemDTO `json:"items"`
    TotalCents   int64          `json:"total_cents"`
    Currency     string         `json:"currency"`
    PlacedAt     time.Time      `json:"placed_at"`
}

type OrderItemDTO struct {
    ProductID   string `json:"product_id"`
    ProductName string `json:"product_name"`
    UnitPrice   int64  `json:"unit_price"`
    Quantity    int    `json:"quantity"`
    Subtotal    int64  `json:"subtotal"`
}

type PlaceOrderResult struct {
    OrderID    string `json:"order_id"`
    TotalCents int64  `json:"total_cents"`
}
```

```go
// internal/application/order/place_order.go
package order

import (
    "context"
    "fmt"

    "myapp/internal/domain/customer"
    domain "myapp/internal/domain/order"
)

// EventPublisher is a port for publishing domain events.
type EventPublisher interface {
    Publish(ctx context.Context, events []domain.DomainEvent) error
}

// PlaceOrderUseCase orchestrates the "place order" workflow.
type PlaceOrderUseCase struct {
    orderRepo    domain.Repository
    customerRepo customer.Repository
    pricing      *domain.PricingService
    events       EventPublisher
}

func NewPlaceOrderUseCase(
    orderRepo domain.Repository,
    customerRepo customer.Repository,
    pricing *domain.PricingService,
    events EventPublisher,
) *PlaceOrderUseCase {
    return &PlaceOrderUseCase{
        orderRepo:    orderRepo,
        customerRepo: customerRepo,
        pricing:      pricing,
        events:       events,
    }
}

func (uc *PlaceOrderUseCase) Execute(ctx context.Context, cmd PlaceOrderCommand) (*PlaceOrderResult, error) {
    // 1. Look up the customer to get their tier
    cust, err := uc.customerRepo.FindByID(ctx, customer.ID(cmd.CustomerID))
    if err != nil {
        return nil, fmt.Errorf("finding customer: %w", err)
    }

    // 2. Convert DTOs to domain items
    items := make([]domain.Item, 0, len(cmd.Items))
    for _, dto := range cmd.Items {
        item, err := domain.NewItem(
            dto.ProductID,
            dto.ProductName,
            domain.NewMoney(dto.UnitPrice, dto.Currency),
            dto.Quantity,
        )
        if err != nil {
            return nil, fmt.Errorf("invalid item %s: %w", dto.ProductID, err)
        }
        items = append(items, item)
    }

    // 3. Create the order aggregate
    orderID := domain.NewID()
    o, err := domain.Place(orderID, cmd.CustomerID, items)
    if err != nil {
        return nil, err
    }

    // 4. Apply tier-based discount (domain service)
    _ = uc.pricing.ApplyDiscount(o.Total(), cust.Tier())
    // In a real system you would update the order total here

    // 5. Persist the order
    if err := uc.orderRepo.Save(ctx, o); err != nil {
        return nil, fmt.Errorf("saving order: %w", err)
    }

    // 6. Publish domain events
    if err := uc.events.Publish(ctx, o.PullEvents()); err != nil {
        // Log but do not fail the use case; use outbox pattern in production
        fmt.Printf("warning: failed to publish events: %v\n", err)
    }

    return &PlaceOrderResult{
        OrderID:    string(orderID),
        TotalCents: o.Total().Amount(),
    }, nil
}
```

```go
// internal/application/order/cancel_order.go
package order

import (
    "context"
    "fmt"

    domain "myapp/internal/domain/order"
)

type CancelOrderUseCase struct {
    orderRepo domain.Repository
    events    EventPublisher
}

func NewCancelOrderUseCase(repo domain.Repository, events EventPublisher) *CancelOrderUseCase {
    return &CancelOrderUseCase{orderRepo: repo, events: events}
}

func (uc *CancelOrderUseCase) Execute(ctx context.Context, cmd CancelOrderCommand) error {
    // 1. Load the aggregate
    o, err := uc.orderRepo.FindByID(ctx, domain.ID(cmd.OrderID))
    if err != nil {
        return fmt.Errorf("finding order: %w", err)
    }

    // 2. Execute the domain operation
    if err := o.Cancel(cmd.Reason); err != nil {
        return err
    }

    // 3. Persist
    if err := uc.orderRepo.Save(ctx, o); err != nil {
        return fmt.Errorf("saving order: %w", err)
    }

    // 4. Publish events
    if err := uc.events.Publish(ctx, o.PullEvents()); err != nil {
        fmt.Printf("warning: failed to publish events: %v\n", err)
    }

    return nil
}
```

```go
// internal/application/order/get_order.go
package order

import (
    "context"
    "fmt"

    "myapp/internal/domain/customer"
    domain "myapp/internal/domain/order"
)

type GetOrderUseCase struct {
    orderRepo    domain.Repository
    customerRepo customer.Repository
}

func NewGetOrderUseCase(orderRepo domain.Repository, customerRepo customer.Repository) *GetOrderUseCase {
    return &GetOrderUseCase{orderRepo: orderRepo, customerRepo: customerRepo}
}

func (uc *GetOrderUseCase) Execute(ctx context.Context, orderID string) (*OrderDTO, error) {
    o, err := uc.orderRepo.FindByID(ctx, domain.ID(orderID))
    if err != nil {
        return nil, err
    }

    // Optionally enrich with customer info
    customerName := ""
    cust, err := uc.customerRepo.FindByID(ctx, customer.ID(o.CustomerID()))
    if err == nil {
        customerName = cust.Name()
    }

    return mapOrderToDTO(o, customerName), nil
}

func mapOrderToDTO(o *domain.Order, customerName string) *OrderDTO {
    items := make([]OrderItemDTO, 0, len(o.Items()))
    for _, item := range o.Items() {
        items = append(items, OrderItemDTO{
            ProductID:   item.ProductID(),
            ProductName: item.ProductName(),
            UnitPrice:   item.UnitPrice().Amount(),
            Quantity:    item.Quantity(),
            Subtotal:    item.Subtotal().Amount(),
        })
    }

    return &OrderDTO{
        ID:           string(o.ID()),
        CustomerID:   o.CustomerID(),
        CustomerName: customerName,
        Status:       o.Status().String(),
        Items:        items,
        TotalCents:   o.Total().Amount(),
        Currency:     o.Total().Currency(),
        PlacedAt:     o.PlacedAt(),
    }
}
```

```go
// internal/application/order/list_orders.go
package order

import (
    "context"
    "fmt"

    domain "myapp/internal/domain/order"
)

type ListOrdersUseCase struct {
    orderRepo domain.Repository
}

func NewListOrdersUseCase(repo domain.Repository) *ListOrdersUseCase {
    return &ListOrdersUseCase{orderRepo: repo}
}

func (uc *ListOrdersUseCase) Execute(ctx context.Context) ([]OrderDTO, error) {
    orders, err := uc.orderRepo.FindAll(ctx)
    if err != nil {
        return nil, fmt.Errorf("listing orders: %w", err)
    }

    dtos := make([]OrderDTO, 0, len(orders))
    for _, o := range orders {
        dtos = append(dtos, *mapOrderToDTO(o, ""))
    }
    return dtos, nil
}
```

### Infrastructure Layer

```go
// internal/infrastructure/memory/order_repo.go
package memory

import (
    "context"
    "sync"

    "myapp/internal/domain/order"
)

// OrderRepository is an in-memory implementation of order.Repository.
// Useful for development, testing, and prototyping.
type OrderRepository struct {
    mu     sync.RWMutex
    orders map[order.ID]*order.Order
}

func NewOrderRepository() *OrderRepository {
    return &OrderRepository{
        orders: make(map[order.ID]*order.Order),
    }
}

func (r *OrderRepository) FindByID(_ context.Context, id order.ID) (*order.Order, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    o, ok := r.orders[id]
    if !ok {
        return nil, order.ErrNotFound
    }
    return o, nil
}

func (r *OrderRepository) FindByCustomerID(_ context.Context, customerID string) ([]*order.Order, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    var result []*order.Order
    for _, o := range r.orders {
        if o.CustomerID() == customerID {
            result = append(result, o)
        }
    }
    return result, nil
}

func (r *OrderRepository) FindAll(_ context.Context) ([]*order.Order, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    result := make([]*order.Order, 0, len(r.orders))
    for _, o := range r.orders {
        result = append(result, o)
    }
    return result, nil
}

func (r *OrderRepository) Save(_ context.Context, o *order.Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    r.orders[o.ID()] = o
    return nil
}

func (r *OrderRepository) Delete(_ context.Context, id order.ID) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    delete(r.orders, id)
    return nil
}

// Compile-time verification
var _ order.Repository = (*OrderRepository)(nil)
```

```go
// internal/infrastructure/memory/customer_repo.go
package memory

import (
    "context"
    "fmt"
    "sync"

    "myapp/internal/domain/customer"
)

type CustomerRepository struct {
    mu        sync.RWMutex
    customers map[customer.ID]*customer.Customer
}

func NewCustomerRepository() *CustomerRepository {
    return &CustomerRepository{
        customers: make(map[customer.ID]*customer.Customer),
    }
}

func (r *CustomerRepository) FindByID(_ context.Context, id customer.ID) (*customer.Customer, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    c, ok := r.customers[id]
    if !ok {
        return nil, fmt.Errorf("customer %s not found", id)
    }
    return c, nil
}

// Seed adds a customer for testing/demo purposes.
func (r *CustomerRepository) Seed(c *customer.Customer) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.customers[c.ID()] = c
}

var _ customer.Repository = (*CustomerRepository)(nil)
```

```go
// internal/infrastructure/logging/event_publisher.go
package logging

import (
    "context"
    "log"

    "myapp/internal/domain/order"
)

// EventPublisher logs events to stdout. In production, this would
// publish to RabbitMQ, Kafka, or similar.
type EventPublisher struct {
    logger *log.Logger
}

func NewEventPublisher(logger *log.Logger) *EventPublisher {
    return &EventPublisher{logger: logger}
}

func (p *EventPublisher) Publish(_ context.Context, events []order.DomainEvent) error {
    for _, event := range events {
        p.logger.Printf("[EVENT] %s occurred at %s\n", event.EventName(), event.OccurredOn())
    }
    return nil
}
```

### Interface Layer (HTTP Handlers)

```go
// internal/interfaces/http/error_mapper.go
package http

import (
    "errors"
    "net/http"

    "myapp/internal/domain/order"
)

// mapErrorToStatus converts domain/application errors to HTTP status codes.
func mapErrorToStatus(err error) int {
    switch {
    case errors.Is(err, order.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, order.ErrCustomerIDRequired),
        errors.Is(err, order.ErrMustHaveItems),
        errors.Is(err, order.ErrProductIDRequired),
        errors.Is(err, order.ErrInvalidQuantity),
        errors.Is(err, order.ErrInvalidPrice):
        return http.StatusBadRequest
    case errors.Is(err, order.ErrCannotModifyOrder),
        errors.Is(err, order.ErrCannotCancelOrder),
        errors.Is(err, order.ErrAlreadyCancelled),
        errors.Is(err, order.ErrInvalidStatusTransition):
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

```go
// internal/interfaces/http/order_handler.go
package http

import (
    "encoding/json"
    "net/http"

    appOrder "myapp/internal/application/order"
)

type OrderHandler struct {
    placeOrder  *appOrder.PlaceOrderUseCase
    cancelOrder *appOrder.CancelOrderUseCase
    getOrder    *appOrder.GetOrderUseCase
    listOrders  *appOrder.ListOrdersUseCase
}

func NewOrderHandler(
    place *appOrder.PlaceOrderUseCase,
    cancel *appOrder.CancelOrderUseCase,
    get *appOrder.GetOrderUseCase,
    list *appOrder.ListOrdersUseCase,
) *OrderHandler {
    return &OrderHandler{
        placeOrder:  place,
        cancelOrder: cancel,
        getOrder:    get,
        listOrders:  list,
    }
}

func (h *OrderHandler) HandlePlaceOrder(w http.ResponseWriter, r *http.Request) {
    var cmd appOrder.PlaceOrderCommand
    if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid request body"})
        return
    }

    result, err := h.placeOrder.Execute(r.Context(), cmd)
    if err != nil {
        status := mapErrorToStatus(err)
        writeJSON(w, status, map[string]string{"error": err.Error()})
        return
    }

    writeJSON(w, http.StatusCreated, result)
}

func (h *OrderHandler) HandleCancelOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.PathValue("id")
    if orderID == "" {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "order ID is required"})
        return
    }

    var body struct {
        Reason string `json:"reason"`
    }
    json.NewDecoder(r.Body).Decode(&body)

    cmd := appOrder.CancelOrderCommand{
        OrderID: orderID,
        Reason:  body.Reason,
    }

    if err := h.cancelOrder.Execute(r.Context(), cmd); err != nil {
        status := mapErrorToStatus(err)
        writeJSON(w, status, map[string]string{"error": err.Error()})
        return
    }

    writeJSON(w, http.StatusOK, map[string]string{"status": "cancelled"})
}

func (h *OrderHandler) HandleGetOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.PathValue("id")
    if orderID == "" {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "order ID is required"})
        return
    }

    result, err := h.getOrder.Execute(r.Context(), orderID)
    if err != nil {
        status := mapErrorToStatus(err)
        writeJSON(w, status, map[string]string{"error": err.Error()})
        return
    }

    writeJSON(w, http.StatusOK, result)
}

func (h *OrderHandler) HandleListOrders(w http.ResponseWriter, r *http.Request) {
    results, err := h.listOrders.Execute(r.Context())
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "failed to list orders"})
        return
    }

    writeJSON(w, http.StatusOK, results)
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

```go
// internal/interfaces/http/router.go
package http

import "net/http"

func NewRouter(handler *OrderHandler) *http.ServeMux {
    mux := http.NewServeMux()

    mux.HandleFunc("POST /api/orders", handler.HandlePlaceOrder)
    mux.HandleFunc("GET /api/orders", handler.HandleListOrders)
    mux.HandleFunc("GET /api/orders/{id}", handler.HandleGetOrder)
    mux.HandleFunc("POST /api/orders/{id}/cancel", handler.HandleCancelOrder)

    return mux
}
```

### Composition Root (main.go)

```go
// cmd/server/main.go
package main

import (
    "log"
    "net/http"
    "os"

    "myapp/internal/domain/customer"
    domainOrder "myapp/internal/domain/order"

    appOrder "myapp/internal/application/order"

    "myapp/internal/infrastructure/logging"
    "myapp/internal/infrastructure/memory"

    httpAdapter "myapp/internal/interfaces/http"
)

func main() {
    logger := log.New(os.Stdout, "[ordersystem] ", log.LstdFlags)

    // --- Infrastructure (driven adapters) ---
    orderRepo := memory.NewOrderRepository()
    customerRepo := memory.NewCustomerRepository()
    eventPublisher := logging.NewEventPublisher(logger)

    // Seed demo data
    email, _ := customer.NewEmail("alice@example.com")
    alice := customer.ReconstructCustomer("cust-1", "Alice Johnson", email, "GOLD", time.Now())
    customerRepo.Seed(alice)

    // --- Domain services ---
    pricingService := domainOrder.NewPricingService()

    // --- Application services (use cases) ---
    placeOrder := appOrder.NewPlaceOrderUseCase(orderRepo, customerRepo, pricingService, eventPublisher)
    cancelOrder := appOrder.NewCancelOrderUseCase(orderRepo, eventPublisher)
    getOrder := appOrder.NewGetOrderUseCase(orderRepo, customerRepo)
    listOrders := appOrder.NewListOrdersUseCase(orderRepo)

    // --- Interface adapters (driving adapters) ---
    handler := httpAdapter.NewOrderHandler(placeOrder, cancelOrder, getOrder, listOrders)
    router := httpAdapter.NewRouter(handler)

    // --- Start server ---
    addr := ":8080"
    logger.Printf("Starting server on %s", addr)
    if err := http.ListenAndServe(addr, router); err != nil {
        logger.Fatalf("Server failed: %v", err)
    }
}
```

### Testing the Complete Example

```bash
# Place an order
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "cust-1",
    "items": [
      {
        "product_id": "prod-1",
        "product_name": "Mechanical Keyboard",
        "unit_price": 12999,
        "currency": "USD",
        "quantity": 1
      },
      {
        "product_id": "prod-2",
        "product_name": "Mouse Pad",
        "unit_price": 1999,
        "currency": "USD",
        "quantity": 2
      }
    ]
  }'

# Response: {"order_id":"abc-123","total_cents":16997}

# Get the order
curl http://localhost:8080/api/orders/abc-123

# List all orders
curl http://localhost:8080/api/orders

# Cancel the order
curl -X POST http://localhost:8080/api/orders/abc-123/cancel \
  -H "Content-Type: application/json" \
  -d '{"reason": "Changed my mind"}'
```

---

## Common Mistakes and Pragmatic Shortcuts

### Mistake 1: Anemic Domain Model

The domain entities are just data containers with getters/setters, and all business logic lives
in services.

```go
// WRONG: Anemic domain model
type Order struct {
    ID         string
    CustomerID string
    Status     string // Just a string, no validation
    Items      []OrderItem
    Total      float64 // float for money -- another mistake
}

// "Service" that contains all the logic
type OrderService struct {
    repo OrderRepository
}

func (s *OrderService) CancelOrder(id string) error {
    order, _ := s.repo.FindByID(id)
    if order.Status == "SHIPPED" {
        return errors.New("cannot cancel")
    }
    order.Status = "CANCELLED" // Direct mutation
    return s.repo.Save(order)
}
```

```go
// RIGHT: Rich domain model
type Order struct {
    id     ID
    status Status
    // ... unexported fields
}

func (o *Order) Cancel(reason string) error {
    if !o.status.CanTransitionTo(StatusCancelled) {
        return ErrCannotCancelOrder
    }
    o.status = StatusCancelled
    o.raise(OrderCancelled{...})
    return nil
}
```

### Mistake 2: Leaking Infrastructure into the Domain

```go
// WRONG: Domain entity knows about SQL
package order

import "database/sql"

type Order struct {
    ID   sql.NullString // SQL type in the domain!
    // ...
}

func (o *Order) Save(db *sql.DB) error { // Persistence logic in the entity!
    _, err := db.Exec("INSERT INTO orders ...")
    return err
}
```

```go
// RIGHT: Domain entity is pure
package order

type Order struct {
    id ID // Domain type
}

// Repository interface in domain; implementation elsewhere
type Repository interface {
    Save(ctx context.Context, order *Order) error
}
```

### Mistake 3: Over-Engineering Simple CRUD

If your "domain logic" is just validation and CRUD, you do not need Clean Architecture.

```go
// Over-engineered for a simple user profile:
// domain/user/user.go
// domain/user/repository.go
// application/user/update_profile.go
// infrastructure/postgres/user_repo.go
// interfaces/http/user_handler.go

// Just do this instead:
func handleUpdateProfile(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    json.NewDecoder(r.Body).Decode(&req)
    _, err := db.Exec("UPDATE users SET name=$1, email=$2 WHERE id=$3", req.Name, req.Email, userID)
    // ...
}
```

### Mistake 4: Wrong Aggregate Boundaries

```go
// WRONG: Order aggregate contains full Customer and Product entities
type Order struct {
    id       OrderID
    customer *Customer    // Loading the entire customer graph!
    items    []*Product   // Loading entire product entities!
}

// RIGHT: Reference other aggregates by ID
type Order struct {
    id         OrderID
    customerID string     // Just the reference
    items      []Item     // Item contains productID, not *Product
}
```

### Mistake 5: Not Using the Reconstitute Pattern

When loading entities from the database, you need to bypass validation and event recording:

```go
// WRONG: Using the constructor when loading from DB
func (r *PostgresRepo) FindByID(ctx context.Context, id order.ID) (*order.Order, error) {
    row := r.db.QueryRow(...)
    // ...
    // This will trigger validation and raise domain events!
    return order.Place(id, customerID, items)
}

// RIGHT: Using a reconstitution function
func (r *PostgresRepo) FindByID(ctx context.Context, id order.ID) (*order.Order, error) {
    row := r.db.QueryRow(...)
    // ...
    // Bypasses validation and events -- just rebuilds the state
    return order.Reconstitute(id, customerID, items, status, total, placedAt, confirmedAt, cancelledAt, reason), nil
}
```

### Mistake 6: Circular Dependencies Between Packages

```go
// WRONG: order depends on customer, customer depends on order
// domain/order/order.go
import "myapp/internal/domain/customer"

// domain/customer/customer.go
import "myapp/internal/domain/order" // Circular!

// RIGHT: Use interfaces or a shared package
// domain/order/order.go references customerID as a string
// If cross-aggregate queries are needed, use the application layer
```

### Pragmatic Shortcuts

Here are legitimate shortcuts for teams that want the benefits of Clean Architecture without
the full ceremony:

#### Shortcut 1: Collapse Application and Domain into One Package

For small bounded contexts, having separate `domain/` and `application/` directories is
unnecessary overhead.

```
internal/
├── order/           # Combined domain + application
│   ├── order.go     # Entity + value objects
│   ├── service.go   # Use cases (application service)
│   ├── repository.go # Port
│   └── errors.go
├── postgres/        # Infrastructure
│   └── order_repo.go
└── api/             # Interface
    └── handler.go
```

#### Shortcut 2: Skip Domain Events If You Do Not Need Them

Domain events add complexity. If your use cases are simple request-response with no async
processing, skip them until you need them.

#### Shortcut 3: Use Table-Driven Tests Instead of Mocks

For simple use cases, table-driven tests with an in-memory repository can be cleaner than
complex mock setups:

```go
func TestCancelOrder(t *testing.T) {
    tests := []struct {
        name        string
        setupOrder  func() *order.Order
        wantErr     error
    }{
        {
            name: "cancel pending order succeeds",
            setupOrder: func() *order.Order {
                item, _ := order.NewItem("p1", "Widget", order.NewMoney(100, "USD"), 1)
                o, _ := order.Place(order.NewID(), "c1", []order.Item{item})
                return o
            },
            wantErr: nil,
        },
        {
            name: "cancel shipped order fails",
            setupOrder: func() *order.Order {
                item, _ := order.NewItem("p1", "Widget", order.NewMoney(100, "USD"), 1)
                o, _ := order.Place(order.NewID(), "c1", []order.Item{item})
                o.Confirm()
                // Assume Ship() exists
                return o
            },
            wantErr: order.ErrCannotCancelOrder,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            o := tt.setupOrder()
            err := o.Cancel("test reason")
            if !errors.Is(err, tt.wantErr) {
                t.Errorf("expected %v, got %v", tt.wantErr, err)
            }
        })
    }
}
```

#### Shortcut 4: Use Constructor Functions Judiciously

Not every value object needs a full constructor with validation. For internal types that are
always created by trusted code, a simple struct literal is fine:

```go
// This is fine for internal types created within the aggregate
item := OrderItem{
    productID: "prod-1",
    quantity:  2,
}

// But for types at the aggregate boundary, always validate
func NewEmail(address string) (Email, error) {
    // Validation required here because this is called from outside
}
```

#### Shortcut 5: Accept Minor Impurity in the Domain Layer

If your domain entity needs a UUID, it is acceptable to import a UUID library in the domain
layer. The rule is about not depending on infrastructure frameworks (databases, HTTP, messaging),
not about zero external dependencies.

```go
// This is acceptable in the domain layer:
import "github.com/google/uuid"

func NewOrderID() OrderID {
    return OrderID(uuid.New().String())
}
```

### Decision Matrix: What Goes Where?

| Question | Domain | Application | Infrastructure | Interface |
|---|---|---|---|---|
| "Is this a business rule?" | Yes | | | |
| "Does this orchestrate a workflow?" | | Yes | | |
| "Does this talk to a database?" | | | Yes | |
| "Does this parse an HTTP request?" | | | | Yes |
| "Does this validate business invariants?" | Yes | | | |
| "Does this validate input format?" | | | | Yes |
| "Does this send emails?" | | | Yes | |
| "Does this define what data is needed?" | Yes (interface) | | Yes (impl) | |
| "Does this map DTOs to domain objects?" | | Yes | | |
| "Does this map domain errors to HTTP codes?" | | | | Yes |

---

## Summary

| Concept | Key Takeaway |
|---|---|
| **Clean Architecture** | Dependencies point inward; domain has zero external dependencies |
| **Hexagonal Architecture** | Ports (interfaces) defined inside, adapters (implementations) outside |
| **Entities** | Identity-based, mutable through controlled methods, enforce invariants |
| **Value Objects** | Attribute-based, immutable, operations return new instances |
| **Aggregates** | Transactional boundary; all changes through the root |
| **Domain Events** | Past-tense facts that happened; collected and published after save |
| **Repository Pattern** | Abstract persistence; interface in domain, implementation in infrastructure |
| **Domain Services** | Business logic that spans entities; no infrastructure deps |
| **Application Services** | Orchestrate use cases; coordinate domain + ports |
| **Bounded Contexts** | Separate models for separate areas of the business |
| **Anti-Corruption Layer** | Translate external models to protect your domain |
| **Ubiquitous Language** | Code speaks the same language as domain experts |
| **Domain Errors** | Classify errors so outer layers can map them appropriately |
| **Testing** | Domain: pure unit tests; Application: mocked ports; Infra: integration tests |
| **Pragmatism** | Use only what you need; start simple and evolve toward complexity |

The goal is not to follow every pattern religiously. The goal is to make your business logic
easy to understand, easy to test, and easy to change independently of the technology choices
surrounding it. Start with the domain model, get the language right, enforce the dependency rule,
and add layers of sophistication only when the complexity of your domain demands it.
