# Chapter 21: Building Microservices Architecture

Microservices architecture decomposes a monolithic application into small, independent, loosely coupled services that communicate over the network. Each service owns its own data, deploys independently, and can be written in a different language -- though in practice, organizations pick one or two languages for consistency. Go has become the dominant language for this domain, and this chapter explores exactly why, how to build production-grade microservices in Go, and how every aspect compares to doing the same work in Node.js.

This is not a theoretical overview. We will build real service code, wire up gRPC, connect message queues, add observability, configure graceful shutdown, and package everything for Kubernetes. Every section addresses production concerns that textbooks often skip.

---

## Table of Contents

1. [Why Go for Microservices?](#1-why-go-for-microservices)
2. [Project Structure](#2-project-structure)
3. [gRPC and Protocol Buffers](#3-grpc-and-protocol-buffers)
4. [Service Discovery](#4-service-discovery)
5. [API Gateway Pattern](#5-api-gateway-pattern)
6. [Database Patterns](#6-database-patterns)
7. [Message Queues and Event-Driven Architecture](#7-message-queues-and-event-driven-architecture)
8. [Observability](#8-observability)
9. [Configuration Management](#9-configuration-management)
10. [Graceful Shutdown](#10-graceful-shutdown)
11. [Health Checks and Readiness](#11-health-checks-and-readiness)
12. [Docker and Deployment](#12-docker-and-deployment)
13. [Go vs Node.js Microservices Comparison](#13-go-vs-nodejs-microservices-comparison)
14. [Key Takeaways](#14-key-takeaways)
15. [Practice Exercises](#15-practice-exercises)

---

## 1. Why Go for Microservices?

### The Cloud-Native Language

Go did not accidentally become the language of cloud infrastructure. Docker, Kubernetes, Prometheus, Terraform, etcd, Consul, Istio, CockroachDB, InfluxDB, Traefik, NATS -- the entire cloud-native ecosystem -- are written in Go. This is not a coincidence. It is the result of specific engineering properties that make Go uniquely suited for networked, distributed systems.

### Small Binary Size

A Go HTTP server compiles to a single static binary, typically 5-15 MB. There is no runtime to install, no node_modules, no JVM. You copy the binary to a server and it runs.

```go
// This entire HTTP server compiles to ~6MB
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello from Go microservice")
    })
    http.ListenAndServe(":8080", nil)
}
```

```bash
$ go build -o server main.go
$ ls -lh server
-rwxr-xr-x  1 user  staff  6.2M  server

# With ldflags to strip debug info:
$ go build -ldflags="-s -w" -o server main.go
$ ls -lh server
-rwxr-xr-x  1 user  staff  4.1M  server
```

Compare with Node.js where even a minimal Express server pulls in 50+ MB of node_modules, plus the ~60MB Node.js runtime.

### Fast Startup Time

Go services start in milliseconds. This matters enormously in containerized environments where services scale up and down constantly.

```
Go microservice startup:      ~10-50ms
Node.js microservice startup: ~200-800ms (module loading, JIT warmup)
Java microservice startup:    ~2-10 seconds (JVM, Spring Boot)
```

**WHY this matters:** In Kubernetes, when a pod crashes or new replicas are needed to handle a traffic spike, Go services are ready to serve requests almost instantly. Node.js services need time to parse and compile JavaScript. Java services need the JVM to warm up. Those seconds translate directly into failed requests during scale-up events.

### Low Memory Footprint

A Go HTTP service at rest uses 5-15 MB of memory. A Node.js Express service at rest uses 30-80 MB. When you are running hundreds of microservices, this difference is the difference between needing 10 servers and needing 40.

```
+-----------------------------------------+
|     Memory Usage Per Service Instance   |
+-----------------------------------------+
| Go          | ██░░░░░░░░░░░ ~10 MB      |
| Node.js     | ██████░░░░░░░ ~50 MB      |
| Java/Spring | ████████████░ ~200 MB     |
+-----------------------------------------+
```

### Built-In Concurrency

Go's goroutines and channels are the foundation of its microservice strength. Each incoming HTTP request runs in its own goroutine. A single Go service can handle tens of thousands of concurrent connections on a single core, with goroutines costing only ~2KB of stack space each.

```go
// Each request gets its own goroutine automatically
// 10,000 concurrent requests = 10,000 goroutines = ~20MB of stack space
// The same in Node.js uses a single thread with an event loop
```

Node.js handles concurrency through its event loop and async/await model. This works well for I/O-bound tasks, but a single CPU-heavy request blocks the entire event loop. Go's goroutines are preemptively scheduled across all CPU cores.

### Strong Standard Library

Go's standard library includes production-ready HTTP servers, JSON encoding, TLS, cryptography, SQL database drivers, templating, testing, and profiling. You can build a serious microservice with zero third-party dependencies.

```go
// Production HTTP server using ONLY the standard library
package main

import (
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    mux := http.NewServeMux()
    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })
    mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        json.NewEncoder(w).Encode(map[string]string{"id": id, "name": "Alice"})
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    go func() {
        logger.Info("starting server", "addr", server.Addr)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("server error", "err", err)
            os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    server.Shutdown(ctx)
    logger.Info("server stopped")
}
```

In Node.js, you need Express or Fastify for routing, a signal handler library, and separate packages for structured logging. Go ships it all in the standard library.

---

## 2. Project Structure

### The Standard Go Project Layout

Go does not enforce a project structure, but a strong convention has emerged for microservices. The key principle is **separation of concerns**: your business logic should not know about HTTP, gRPC, or databases.

```
order-service/
├── cmd/
│   └── server/
│       └── main.go              # Entry point -- wires everything together
├── internal/
│   ├── domain/
│   │   ├── order.go             # Domain entities (pure structs, no dependencies)
│   │   └── order_repository.go  # Repository interface (not implementation)
│   ├── service/
│   │   └── order_service.go     # Business logic (depends on domain interfaces)
│   ├── handler/
│   │   ├── http.go              # HTTP handlers (adapts HTTP to service calls)
│   │   └── grpc.go              # gRPC handlers (adapts gRPC to service calls)
│   ├── repository/
│   │   ├── postgres.go          # PostgreSQL implementation of repository
│   │   └── memory.go            # In-memory implementation (for testing)
│   └── middleware/
│       ├── logging.go           # HTTP middleware for request logging
│       ├── auth.go              # Authentication middleware
│       └── recovery.go          # Panic recovery middleware
├── pkg/
│   └── pagination/
│       └── pagination.go        # Reusable utilities (importable by other services)
├── api/
│   └── proto/
│       └── order.proto          # Protocol Buffer definitions
├── migrations/
│   ├── 001_create_orders.up.sql
│   └── 001_create_orders.down.sql
├── config/
│   └── config.go                # Configuration loading
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── go.mod
└── go.sum
```

### WHY This Structure?

**`cmd/`** -- Contains the `main` package. Each subdirectory is a separate binary. A service might have `cmd/server/` for the main service and `cmd/migrate/` for database migrations. The `main.go` file is thin -- it reads config, creates dependencies, wires them together, and starts the server.

**`internal/`** -- This is a Go-enforced boundary. Code inside `internal/` cannot be imported by any code outside this module. This prevents other services from reaching into your implementation details. This is one of Go's most powerful features for maintaining clean architecture -- the compiler enforces your boundaries.

**`pkg/`** -- Code that is safe for other projects to import. Use this sparingly. If you are unsure, put it in `internal/`.

**`domain/`** -- Pure business entities and interfaces. No imports from infrastructure packages. This is the core of hexagonal architecture -- your domain does not know whether it is being served over HTTP, gRPC, or carrier pigeon.

**`service/`** -- Business logic layer. Depends only on domain interfaces, not concrete implementations. This is where your rules live.

**`handler/`** -- Adapters that translate protocol-specific requests (HTTP, gRPC) into service calls. Thin and mechanical.

**`repository/`** -- Adapters that implement domain interfaces using specific databases. Swappable.

### Hexagonal Architecture in Go

The hexagonal architecture (also called ports and adapters) maps naturally to Go interfaces:

```
                    ┌─────────────────────────────────────┐
                    │           HTTP Handler               │
                    │         (adapter - inbound)          │
                    └──────────────┬──────────────────────┘
                                   │ calls
                    ┌──────────────▼──────────────────────┐
                    │         Service Layer                │
                    │       (business logic)               │
                    │   depends on interfaces, not         │
                    │   concrete implementations           │
                    └──────────────┬──────────────────────┘
                                   │ calls via interface
                    ┌──────────────▼──────────────────────┐
                    │      Repository Interface            │
                    │        (port - outbound)             │
                    └──────────────┬──────────────────────┘
                                   │ implemented by
                    ┌──────────────▼──────────────────────┐
                    │   PostgreSQL Repository              │
                    │      (adapter - outbound)            │
                    └─────────────────────────────────────┘
```

Here is how this looks in code:

```go
// internal/domain/order.go -- Domain entity (no external dependencies)
package domain

import "time"

type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusConfirmed OrderStatus = "confirmed"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusCancelled OrderStatus = "cancelled"
)

type Order struct {
    ID         string
    CustomerID string
    Items      []OrderItem
    Status     OrderStatus
    Total      int64  // cents -- never use float for money
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type OrderItem struct {
    ProductID string
    Quantity  int
    PriceEach int64  // cents
}
```

```go
// internal/domain/order_repository.go -- Port (interface)
package domain

import "context"

// OrderRepository defines what the service layer needs from persistence.
// It does not mention PostgreSQL, MongoDB, or any specific technology.
type OrderRepository interface {
    Create(ctx context.Context, order *Order) error
    GetByID(ctx context.Context, id string) (*Order, error)
    ListByCustomer(ctx context.Context, customerID string, limit, offset int) ([]*Order, error)
    UpdateStatus(ctx context.Context, id string, status OrderStatus) error
}
```

```go
// internal/service/order_service.go -- Business logic
package service

import (
    "context"
    "fmt"
    "time"

    "github.com/google/uuid"
    "mycompany/order-service/internal/domain"
)

type OrderService struct {
    repo   domain.OrderRepository  // depends on interface, not concrete type
    events EventPublisher          // another interface
}

func NewOrderService(repo domain.OrderRepository, events EventPublisher) *OrderService {
    return &OrderService{repo: repo, events: events}
}

func (s *OrderService) CreateOrder(ctx context.Context, customerID string, items []domain.OrderItem) (*domain.Order, error) {
    // Business logic: calculate total
    var total int64
    for _, item := range items {
        total += item.PriceEach * int64(item.Quantity)
    }

    if total <= 0 {
        return nil, fmt.Errorf("order total must be positive")
    }

    order := &domain.Order{
        ID:         uuid.New().String(),
        CustomerID: customerID,
        Items:      items,
        Status:     domain.OrderStatusPending,
        Total:      total,
        CreatedAt:  time.Now(),
        UpdatedAt:  time.Now(),
    }

    if err := s.repo.Create(ctx, order); err != nil {
        return nil, fmt.Errorf("creating order: %w", err)
    }

    // Publish event (fire and forget -- the event publisher handles retries)
    s.events.Publish(ctx, "order.created", order)

    return order, nil
}
```

### Comparison with Node.js Project Structure

```
# Node.js microservice (typical structure)
order-service/
├── src/
│   ├── index.ts                 # Entry point
│   ├── routes/
│   │   └── orderRoutes.ts       # Express routes
│   ├── controllers/
│   │   └── orderController.ts   # Request handling
│   ├── services/
│   │   └── orderService.ts      # Business logic
│   ├── models/
│   │   └── order.ts             # Prisma/TypeORM model
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── errorHandler.ts
│   └── utils/
│       └── logger.ts
├── prisma/
│   └── schema.prisma
├── package.json
├── tsconfig.json
├── Dockerfile
└── node_modules/                # 50-500MB of dependencies
```

Key differences:

| Aspect | Go | Node.js |
|---|---|---|
| Enforced boundaries | `internal/` is compiler-enforced | Convention only -- nothing stops imports |
| Dependencies | `go.mod` lists ~5-20 direct deps | `package.json` often has 20-50 direct deps |
| Dependency size | `vendor/` or module cache, small | `node_modules/` can be 500MB+ |
| Interface enforcement | Compile-time via interfaces | Runtime or via TypeScript types |
| Build artifact | Single binary | Entire source + node_modules |

**WHY Go's `internal/` directory matters:** In a Node.js project, any file can import any other file. You rely on developer discipline to maintain architectural boundaries. In Go, the compiler physically prevents code outside your module from importing `internal/` packages. When you have 50 microservices and 30 developers, compiler-enforced boundaries prevent architectural decay.

---

## 3. gRPC and Protocol Buffers

### WHY gRPC Over REST for Inter-Service Communication

REST with JSON is the default for public APIs. But for communication **between** microservices, gRPC offers significant advantages:

```
┌──────────────┐   gRPC (binary, typed)   ┌──────────────┐
│   Order      │◄────────────────────────►│   Payment    │
│   Service    │                           │   Service    │
└──────┬───────┘                           └──────────────┘
       │
       │  REST/JSON (human-readable)
       ▼
┌──────────────┐
│   Public     │
│   Clients    │
│  (browsers,  │
│   mobile)    │
└──────────────┘
```

```
+-------------------+---------------------------+---------------------------+
| Feature           | REST/JSON                 | gRPC/Protobuf             |
+-------------------+---------------------------+---------------------------+
| Serialization     | Text (JSON) -- slow       | Binary (protobuf) -- fast |
| Schema            | Optional (OpenAPI)        | Required (.proto files)   |
| Code generation   | Optional                  | Built-in                  |
| Streaming         | Websockets (bolt-on)      | Native bidirectional      |
| Performance       | ~1-5ms serialization      | ~0.1-0.5ms serialization  |
| Type safety       | Runtime validation        | Compile-time types        |
| HTTP version      | HTTP/1.1 typically        | HTTP/2 always             |
| Browser support   | Universal                 | Requires proxy (grpc-web) |
+-------------------+---------------------------+---------------------------+
```

**WHY binary protocol matters:** In a microservices system, services constantly talk to each other. An order request might trigger calls to the inventory service, payment service, notification service, and shipping service. JSON serialization/deserialization at every hop adds up. Protobuf is 3-10x faster to serialize and produces messages 3-10x smaller than JSON.

### Protocol Buffer Definition

Protocol Buffers (protobuf) define the schema for your service. They are language-neutral -- the same `.proto` file generates Go, Node.js, Java, Python, and other clients.

```protobuf
// api/proto/order/v1/order.proto
syntax = "proto3";

package order.v1;

option go_package = "mycompany/order-service/gen/order/v1;orderv1";

import "google/protobuf/timestamp.proto";

// The Order service definition
service OrderService {
    // Unary RPC -- one request, one response
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);

    // Server streaming -- one request, stream of responses
    rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusUpdate);

    // Client streaming -- stream of requests, one response
    rpc BulkCreateOrders(stream CreateOrderRequest) returns (BulkCreateOrdersResponse);

    // Bidirectional streaming -- stream both ways
    rpc OrderChat(stream OrderChatMessage) returns (stream OrderChatMessage);
}

message CreateOrderRequest {
    string customer_id = 1;
    repeated OrderItemRequest items = 2;
}

message OrderItemRequest {
    string product_id = 1;
    int32 quantity = 2;
}

message CreateOrderResponse {
    Order order = 1;
}

message GetOrderRequest {
    string order_id = 1;
}

message GetOrderResponse {
    Order order = 1;
}

message Order {
    string id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;
    OrderStatus status = 4;
    int64 total_cents = 5;
    google.protobuf.Timestamp created_at = 6;
    google.protobuf.Timestamp updated_at = 7;
}

message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    int64 price_each_cents = 3;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_CANCELLED = 4;
}

message WatchOrderStatusRequest {
    string order_id = 1;
}

message OrderStatusUpdate {
    string order_id = 1;
    OrderStatus old_status = 2;
    OrderStatus new_status = 3;
    google.protobuf.Timestamp updated_at = 4;
}

message BulkCreateOrdersResponse {
    int32 created_count = 1;
    repeated string order_ids = 2;
}

message OrderChatMessage {
    string order_id = 1;
    string sender = 2;
    string message = 3;
    google.protobuf.Timestamp sent_at = 4;
}
```

### Code Generation

Install the tools and generate Go code from the proto definition:

```bash
# Install protoc compiler and Go plugins
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generate Go code
$ protoc --go_out=. --go_opt=paths=source_relative \
         --go-grpc_out=. --go-grpc_opt=paths=source_relative \
         api/proto/order/v1/order.proto
```

This generates two files:
- `order.pb.go` -- Message types (structs, serialization)
- `order_grpc.pb.go` -- Service interface and client/server stubs

### Implementing the gRPC Server

```go
// internal/handler/grpc.go
package handler

import (
    "context"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/timestamppb"

    orderv1 "mycompany/order-service/gen/order/v1"
    "mycompany/order-service/internal/domain"
    "mycompany/order-service/internal/service"
)

// OrderGRPCHandler implements the generated gRPC interface.
// The generated interface is orderv1.OrderServiceServer.
type OrderGRPCHandler struct {
    orderv1.UnimplementedOrderServiceServer  // Forward compatibility
    svc *service.OrderService
}

func NewOrderGRPCHandler(svc *service.OrderService) *OrderGRPCHandler {
    return &OrderGRPCHandler{svc: svc}
}

func (h *OrderGRPCHandler) CreateOrder(
    ctx context.Context,
    req *orderv1.CreateOrderRequest,
) (*orderv1.CreateOrderResponse, error) {
    // Validate request
    if req.CustomerId == "" {
        return nil, status.Error(codes.InvalidArgument, "customer_id is required")
    }
    if len(req.Items) == 0 {
        return nil, status.Error(codes.InvalidArgument, "at least one item is required")
    }

    // Convert protobuf types to domain types
    items := make([]domain.OrderItem, len(req.Items))
    for i, item := range req.Items {
        items[i] = domain.OrderItem{
            ProductID: item.ProductId,
            Quantity:  int(item.Quantity),
        }
    }

    // Call business logic
    order, err := h.svc.CreateOrder(ctx, req.CustomerId, items)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "creating order: %v", err)
    }

    // Convert domain types to protobuf response
    return &orderv1.CreateOrderResponse{
        Order: domainOrderToProto(order),
    }, nil
}

func (h *OrderGRPCHandler) GetOrder(
    ctx context.Context,
    req *orderv1.GetOrderRequest,
) (*orderv1.GetOrderResponse, error) {
    order, err := h.svc.GetOrder(ctx, req.OrderId)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "order not found: %v", err)
    }

    return &orderv1.GetOrderResponse{
        Order: domainOrderToProto(order),
    }, nil
}

// Server streaming -- sends multiple responses for one request
func (h *OrderGRPCHandler) WatchOrderStatus(
    req *orderv1.WatchOrderStatusRequest,
    stream orderv1.OrderService_WatchOrderStatusServer,
) error {
    ctx := stream.Context()

    // Subscribe to status changes (simplified -- real implementation uses pub/sub)
    updates := h.svc.SubscribeToOrderUpdates(ctx, req.OrderId)

    for {
        select {
        case <-ctx.Done():
            return nil // Client disconnected
        case update, ok := <-updates:
            if !ok {
                return nil // Channel closed
            }
            if err := stream.Send(&orderv1.OrderStatusUpdate{
                OrderId:   update.OrderID,
                NewStatus: orderv1.OrderStatus(update.NewStatus),
                UpdatedAt: timestamppb.New(update.UpdatedAt),
            }); err != nil {
                return err
            }
        }
    }
}

func domainOrderToProto(o *domain.Order) *orderv1.Order {
    items := make([]*orderv1.OrderItem, len(o.Items))
    for i, item := range o.Items {
        items[i] = &orderv1.OrderItem{
            ProductId:     item.ProductID,
            Quantity:      int32(item.Quantity),
            PriceEachCents: item.PriceEach,
        }
    }
    return &orderv1.Order{
        Id:         o.ID,
        CustomerId: o.CustomerID,
        Items:      items,
        Status:     orderv1.OrderStatus(orderv1.OrderStatus_value[string(o.Status)]),
        TotalCents: o.Total,
        CreatedAt:  timestamppb.New(o.CreatedAt),
        UpdatedAt:  timestamppb.New(o.UpdatedAt),
    }
}
```

### gRPC Interceptors (Middleware)

gRPC interceptors are the equivalent of HTTP middleware. They wrap every RPC call for logging, authentication, metrics, etc.

```go
// internal/middleware/grpc_logging.go
package middleware

import (
    "context"
    "log/slog"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/status"
)

// UnaryLoggingInterceptor logs every unary RPC call with duration and status.
func UnaryLoggingInterceptor(logger *slog.Logger) grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        start := time.Now()

        // Call the actual handler
        resp, err := handler(ctx, req)

        // Log the result
        duration := time.Since(start)
        st, _ := status.FromError(err)

        logger.Info("gRPC call",
            "method", info.FullMethod,
            "duration_ms", duration.Milliseconds(),
            "status", st.Code().String(),
        )

        return resp, err
    }
}

// UnaryAuthInterceptor validates authentication tokens.
func UnaryAuthInterceptor(validator TokenValidator) grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        // Skip auth for health checks
        if info.FullMethod == "/grpc.health.v1.Health/Check" {
            return handler(ctx, req)
        }

        // Extract token from metadata
        token, err := extractToken(ctx)
        if err != nil {
            return nil, status.Error(codes.Unauthenticated, "missing token")
        }

        // Validate token
        claims, err := validator.Validate(token)
        if err != nil {
            return nil, status.Error(codes.Unauthenticated, "invalid token")
        }

        // Add claims to context
        ctx = context.WithValue(ctx, claimsKey{}, claims)
        return handler(ctx, req)
    }
}
```

### Wiring Up the gRPC Server

```go
// cmd/server/main.go
package main

import (
    "log/slog"
    "net"
    "os"

    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"

    orderv1 "mycompany/order-service/gen/order/v1"
    "mycompany/order-service/internal/handler"
    "mycompany/order-service/internal/middleware"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Create gRPC server with interceptors (middleware chain)
    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            middleware.UnaryLoggingInterceptor(logger),
            middleware.UnaryRecoveryInterceptor(),
            middleware.UnaryAuthInterceptor(tokenValidator),
        ),
        grpc.ChainStreamInterceptor(
            middleware.StreamLoggingInterceptor(logger),
        ),
    )

    // Register service implementations
    orderHandler := handler.NewOrderGRPCHandler(orderService)
    orderv1.RegisterOrderServiceServer(grpcServer, orderHandler)

    // Enable reflection for development tools (grpcurl, grpcui)
    reflection.Register(grpcServer)

    // Start listening
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        logger.Error("failed to listen", "err", err)
        os.Exit(1)
    }

    logger.Info("gRPC server starting", "addr", ":50051")
    if err := grpcServer.Serve(lis); err != nil {
        logger.Error("gRPC server failed", "err", err)
        os.Exit(1)
    }
}
```

### Go gRPC vs Node.js gRPC

**Go gRPC:**

```go
// Go gRPC client -- calling another service
conn, err := grpc.Dial("payment-service:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil {
    return fmt.Errorf("connecting to payment service: %w", err)
}
defer conn.Close()

client := paymentv1.NewPaymentServiceClient(conn)

resp, err := client.ChargeCustomer(ctx, &paymentv1.ChargeRequest{
    CustomerId: order.CustomerID,
    AmountCents: order.Total,
    OrderId:    order.ID,
})
if err != nil {
    st, _ := status.FromError(err)
    return fmt.Errorf("payment failed: %s: %s", st.Code(), st.Message())
}
```

**Node.js gRPC:**

```javascript
// Node.js gRPC client -- same operation
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const packageDefinition = protoLoader.loadSync("payment.proto");
const paymentProto = grpc.loadPackageDefinition(packageDefinition);

const client = new paymentProto.payment.v1.PaymentService(
    "payment-service:50051",
    grpc.credentials.createInsecure()
);

// Callback-based API (can be promisified)
client.chargeCustomer(
    {
        customerId: order.customerId,
        amountCents: order.total,
        orderId: order.id,
    },
    (err, response) => {
        if (err) {
            console.error("Payment failed:", err.message);
            return;
        }
        console.log("Payment succeeded:", response);
    }
);
```

Key differences:
- Go generates strongly typed client code at compile time. Node.js loads proto files at runtime (or uses `grpc-tools` for code generation).
- Go's gRPC uses native HTTP/2. Node.js also uses HTTP/2 via the `@grpc/grpc-js` package but with more overhead.
- Go gRPC streaming is trivially handled with goroutines. Node.js streaming uses Node streams.
- Go gRPC benchmarks at 2-5x the throughput of Node.js gRPC for typical microservice payloads.

---

## 4. Service Discovery

### The Problem

In a dynamic environment like Kubernetes, service instances come and go. IP addresses change. You cannot hardcode addresses.

```
How does Order Service find Payment Service?

                ┌─────────────────────────────────────────────┐
                │              Kubernetes Cluster              │
                │                                             │
                │  ┌───────────┐         ┌───────────┐        │
                │  │  Order    │   ???   │  Payment   │       │
                │  │  Service  │────────►│  Service   │       │
                │  │ 10.0.1.5  │         │ 10.0.2.?? │       │
                │  └───────────┘         │ 10.0.3.?? │       │
                │                        │ 10.0.4.?? │       │
                │                        └───────────┘        │
                └─────────────────────────────────────────────┘
```

### DNS-Based Discovery (Kubernetes Default)

Kubernetes provides built-in DNS-based service discovery. Every Kubernetes Service gets a DNS name. This is the simplest and most common approach.

```go
// In Kubernetes, services are discoverable by their DNS name.
// "payment-service" resolves to the cluster IP for that service.
// Kubernetes handles load balancing across pods.

conn, err := grpc.Dial(
    "payment-service.default.svc.cluster.local:50051",  // Full DNS name
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)

// Or simply:
conn, err := grpc.Dial(
    "payment-service:50051",  // Short name works within the same namespace
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

**WHY DNS-based discovery is usually enough:** Kubernetes DNS resolves service names to virtual IPs that are load-balanced across all healthy pods. For most microservices, this is all you need. You do not need Consul or etcd just for service discovery if you are running in Kubernetes.

### Consul-Based Discovery

For more advanced scenarios (multi-datacenter, non-Kubernetes environments, health-check-aware routing), Consul provides service discovery with health checking.

```go
// Using Consul for service discovery
package discovery

import (
    "fmt"
    "math/rand"

    consul "github.com/hashicorp/consul/api"
)

type ConsulDiscovery struct {
    client *consul.Client
}

func NewConsulDiscovery(addr string) (*ConsulDiscovery, error) {
    config := consul.DefaultConfig()
    config.Address = addr

    client, err := consul.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("creating consul client: %w", err)
    }

    return &ConsulDiscovery{client: client}, nil
}

// Register registers this service instance with Consul.
func (d *ConsulDiscovery) Register(name, id, host string, port int) error {
    return d.client.Agent().ServiceRegister(&consul.AgentServiceRegistration{
        ID:      id,
        Name:    name,
        Address: host,
        Port:    port,
        Check: &consul.AgentServiceCheck{
            GRPC:                           fmt.Sprintf("%s:%d", host, port),
            Interval:                       "10s",
            DeregisterCriticalServiceAfter: "30s",
        },
    })
}

// Discover finds a healthy instance of the named service.
func (d *ConsulDiscovery) Discover(name string) (string, error) {
    entries, _, err := d.client.Health().Service(name, "", true, nil)
    if err != nil {
        return "", fmt.Errorf("querying consul: %w", err)
    }

    if len(entries) == 0 {
        return "", fmt.Errorf("no healthy instances of %s", name)
    }

    // Simple random load balancing
    entry := entries[rand.Intn(len(entries))]
    addr := fmt.Sprintf("%s:%d", entry.Service.Address, entry.Service.Port)
    return addr, nil
}

// Deregister removes this service instance from Consul.
func (d *ConsulDiscovery) Deregister(id string) error {
    return d.client.Agent().ServiceDeregister(id)
}
```

### etcd-Based Discovery

etcd is a distributed key-value store used by Kubernetes itself. It can also be used for service discovery with watches.

```go
// Using etcd for service discovery with watching
package discovery

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
)

type ServiceInstance struct {
    ID      string `json:"id"`
    Address string `json:"address"`
    Port    int    `json:"port"`
}

type EtcdDiscovery struct {
    client *clientv3.Client
    prefix string
}

func NewEtcdDiscovery(endpoints []string, prefix string) (*EtcdDiscovery, error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        return nil, err
    }
    return &EtcdDiscovery{client: client, prefix: prefix}, nil
}

// Register adds a service with a TTL lease (auto-removes if service dies).
func (d *EtcdDiscovery) Register(ctx context.Context, serviceName string, instance ServiceInstance) error {
    key := fmt.Sprintf("%s/%s/%s", d.prefix, serviceName, instance.ID)
    value, _ := json.Marshal(instance)

    // Create a lease that expires in 15 seconds
    lease, err := d.client.Grant(ctx, 15)
    if err != nil {
        return err
    }

    // Put the service registration with the lease
    _, err = d.client.Put(ctx, key, string(value), clientv3.WithLease(lease.ID))
    if err != nil {
        return err
    }

    // Keep the lease alive -- sends heartbeats every ~5 seconds
    ch, err := d.client.KeepAlive(ctx, lease.ID)
    if err != nil {
        return err
    }

    // Drain the keep-alive responses in the background
    go func() {
        for range ch {
            // Keep-alive response received
        }
    }()

    return nil
}

// Watch monitors changes to a service's registered instances.
func (d *EtcdDiscovery) Watch(ctx context.Context, serviceName string) <-chan []ServiceInstance {
    prefix := fmt.Sprintf("%s/%s/", d.prefix, serviceName)
    updates := make(chan []ServiceInstance, 1)

    go func() {
        defer close(updates)
        watchCh := d.client.Watch(ctx, prefix, clientv3.WithPrefix())
        for watchResp := range watchCh {
            for range watchResp.Events {
                // On any change, re-fetch all instances
                instances, _ := d.GetInstances(ctx, serviceName)
                updates <- instances
            }
        }
    }()

    return updates
}
```

---

## 5. API Gateway Pattern

### WHY an API Gateway?

An API gateway sits at the edge of your microservices system, providing a single entry point for external clients. It handles cross-cutting concerns so individual services do not have to.

```
                           ┌─────────────────┐
                           │   API Gateway    │
     Clients ─────────────►│                  │
     (browsers, mobile)    │  - Routing       │
                           │  - Auth          │
                           │  - Rate Limiting │
                           │  - TLS           │
                           │  - CORS          │
                           │  - Request ID    │
                           └──┬────┬────┬────┘
                              │    │    │
                    ┌─────────┘    │    └──────────┐
                    ▼              ▼               ▼
              ┌──────────┐  ┌──────────┐   ┌──────────┐
              │  Order   │  │  User    │   │ Product  │
              │ Service  │  │ Service  │   │ Service  │
              └──────────┘  └──────────┘   └──────────┘
```

### Building an API Gateway in Go

```go
// cmd/gateway/main.go
package main

import (
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "net/http/httputil"
    "net/url"
    "os"
    "strings"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// --- Reverse Proxy ---

type ServiceProxy struct {
    routes map[string]*httputil.ReverseProxy
    logger *slog.Logger
}

func NewServiceProxy(logger *slog.Logger) *ServiceProxy {
    return &ServiceProxy{
        routes: make(map[string]*httputil.ReverseProxy),
        logger: logger,
    }
}

func (sp *ServiceProxy) AddRoute(prefix string, target string) error {
    u, err := url.Parse(target)
    if err != nil {
        return err
    }

    proxy := httputil.NewSingleHostReverseProxy(u)
    proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
        sp.logger.Error("proxy error",
            "target", target,
            "path", r.URL.Path,
            "err", err,
        )
        http.Error(w, "Service unavailable", http.StatusBadGateway)
    }

    sp.routes[prefix] = proxy
    return nil
}

func (sp *ServiceProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    for prefix, proxy := range sp.routes {
        if strings.HasPrefix(r.URL.Path, prefix) {
            // Strip the prefix before forwarding
            r.URL.Path = strings.TrimPrefix(r.URL.Path, prefix)
            if r.URL.Path == "" {
                r.URL.Path = "/"
            }
            proxy.ServeHTTP(w, r)
            return
        }
    }
    http.NotFound(w, r)
}

// --- Rate Limiter Middleware ---

type RateLimiter struct {
    visitors map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(rps float64, burst int) *RateLimiter {
    rl := &RateLimiter{
        visitors: make(map[string]*rate.Limiter),
        rate:     rate.Limit(rps),
        burst:    burst,
    }

    // Cleanup stale entries every minute
    go rl.cleanup()
    return rl
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, exists := rl.visitors[ip]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.visitors[ip] = limiter
    }
    return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        if forwarded := r.Header.Get("X-Forwarded-For"); forwarded != "" {
            ip = strings.Split(forwarded, ",")[0]
        }

        limiter := rl.getLimiter(strings.TrimSpace(ip))
        if !limiter.Allow() {
            w.Header().Set("Retry-After", "1")
            http.Error(w, "Too many requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func (rl *RateLimiter) cleanup() {
    for {
        time.Sleep(time.Minute)
        rl.mu.Lock()
        // In production, track last-seen time and evict old entries
        rl.visitors = make(map[string]*rate.Limiter)
        rl.mu.Unlock()
    }
}

// --- Auth Middleware ---

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Skip auth for health checks
        if r.URL.Path == "/health" || r.URL.Path == "/ready" {
            next.ServeHTTP(w, r)
            return
        }

        authHeader := r.Header.Get("Authorization")
        if !strings.HasPrefix(authHeader, "Bearer ") {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        token := strings.TrimPrefix(authHeader, "Bearer ")
        claims, err := validateJWT(token) // Your JWT validation logic
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // Add user info to context for downstream services
        ctx := context.WithValue(r.Context(), "user_id", claims.UserID)
        r.Header.Set("X-User-ID", claims.UserID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// --- Request ID Middleware ---

func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateUUID()
        }

        r.Header.Set("X-Request-ID", requestID)
        w.Header().Set("X-Request-ID", requestID)

        next.ServeHTTP(w, r)
    })
}

// --- Logging Middleware ---

type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *statusRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}

func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            recorder := &statusRecorder{ResponseWriter: w, statusCode: 200}

            next.ServeHTTP(recorder, r)

            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "status", recorder.statusCode,
                "duration_ms", time.Since(start).Milliseconds(),
                "remote_addr", r.RemoteAddr,
                "request_id", r.Header.Get("X-Request-ID"),
            )
        })
    }
}

// --- Wiring it all together ---

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    proxy := NewServiceProxy(logger)
    proxy.AddRoute("/api/orders", "http://order-service:8080")
    proxy.AddRoute("/api/users", "http://user-service:8080")
    proxy.AddRoute("/api/products", "http://product-service:8080")

    rateLimiter := NewRateLimiter(100, 200) // 100 requests/sec, burst of 200

    // Middleware chain (outermost first)
    var handler http.Handler = proxy
    handler = AuthMiddleware(handler)
    handler = rateLimiter.Middleware(handler)
    handler = RequestIDMiddleware(handler)
    handler = LoggingMiddleware(logger)(handler)

    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
    }

    logger.Info("API gateway starting", "addr", server.Addr)
    if err := server.ListenAndServe(); err != nil {
        logger.Error("server failed", "err", err)
    }
}
```

### Comparison with Node.js API Gateways

Node.js developers typically use Express Gateway, Kong (Lua/Nginx), or build custom gateways with Express/Fastify:

```javascript
// Node.js API gateway (Express)
const express = require("express");
const { createProxyMiddleware } = require("http-proxy-middleware");
const rateLimit = require("express-rate-limit");

const app = express();

// Rate limiting
app.use(
    rateLimit({
        windowMs: 60 * 1000, // 1 minute
        max: 100,
    })
);

// Proxy routes
app.use(
    "/api/orders",
    createProxyMiddleware({ target: "http://order-service:8080" })
);
app.use(
    "/api/users",
    createProxyMiddleware({ target: "http://user-service:8080" })
);

app.listen(8080);
```

The Node.js version is more concise for simple cases. The Go version gives you more control and significantly better performance under load. In production, most organizations use a dedicated gateway like Kong, Envoy, or Traefik rather than building from scratch in either language.

---

## 6. Database Patterns

### database/sql -- The Standard Library

Go's `database/sql` package provides a generic interface for SQL databases. It is not an ORM -- it is a thin abstraction over database drivers. This design is intentional: it gives you control without hiding what is happening.

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/lib/pq"  // PostgreSQL driver -- blank import registers the driver
)

func main() {
    // Open a database connection pool (not a single connection)
    db, err := sql.Open("postgres",
        "host=localhost port=5432 user=app password=secret dbname=orders sslmode=disable",
    )
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // CRITICAL: Configure the connection pool
    db.SetMaxOpenConns(25)                  // Max concurrent connections
    db.SetMaxIdleConns(10)                  // Max idle connections in the pool
    db.SetConnMaxLifetime(5 * time.Minute)  // Max time a connection can be reused
    db.SetConnMaxIdleTime(1 * time.Minute)  // Max time a connection can sit idle

    // Verify the connection works
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        log.Fatal("database unreachable:", err)
    }
}
```

**WHY `sql.Open` does not actually connect:** `sql.Open` validates the driver name and data source format, but it does not open a connection. Connections are opened lazily when you first query. This is by design -- the pool manages connections transparently. Call `db.Ping()` to verify the database is actually reachable.

**WHY connection pool tuning matters:** Without `SetMaxOpenConns`, Go will open unlimited connections, potentially overwhelming your database. Without `SetConnMaxLifetime`, connections might go stale behind a load balancer. These settings are critical in production.

### CRUD Operations with database/sql

```go
// internal/repository/postgres.go
package repository

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    "mycompany/order-service/internal/domain"
)

type PostgresOrderRepository struct {
    db *sql.DB
}

func NewPostgresOrderRepository(db *sql.DB) *PostgresOrderRepository {
    return &PostgresOrderRepository{db: db}
}

// Create inserts a new order with its items in a transaction.
func (r *PostgresOrderRepository) Create(ctx context.Context, order *domain.Order) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }
    // defer tx.Rollback() is safe even after tx.Commit() succeeds --
    // Rollback on a committed transaction is a no-op.
    defer tx.Rollback()

    // Insert the order
    _, err = tx.ExecContext(ctx,
        `INSERT INTO orders (id, customer_id, status, total_cents, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        order.ID, order.CustomerID, order.Status, order.Total,
        order.CreatedAt, order.UpdatedAt,
    )
    if err != nil {
        return fmt.Errorf("inserting order: %w", err)
    }

    // Insert order items
    stmt, err := tx.PrepareContext(ctx,
        `INSERT INTO order_items (order_id, product_id, quantity, price_each_cents)
         VALUES ($1, $2, $3, $4)`,
    )
    if err != nil {
        return fmt.Errorf("preparing item insert: %w", err)
    }
    defer stmt.Close()

    for _, item := range order.Items {
        _, err = stmt.ExecContext(ctx,
            order.ID, item.ProductID, item.Quantity, item.PriceEach,
        )
        if err != nil {
            return fmt.Errorf("inserting order item: %w", err)
        }
    }

    // Commit the transaction
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }

    return nil
}

// GetByID retrieves an order with all its items.
func (r *PostgresOrderRepository) GetByID(ctx context.Context, id string) (*domain.Order, error) {
    order := &domain.Order{}

    err := r.db.QueryRowContext(ctx,
        `SELECT id, customer_id, status, total_cents, created_at, updated_at
         FROM orders WHERE id = $1`,
        id,
    ).Scan(
        &order.ID, &order.CustomerID, &order.Status,
        &order.Total, &order.CreatedAt, &order.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("order %s not found", id)
    }
    if err != nil {
        return nil, fmt.Errorf("querying order: %w", err)
    }

    // Fetch order items
    rows, err := r.db.QueryContext(ctx,
        `SELECT product_id, quantity, price_each_cents
         FROM order_items WHERE order_id = $1`,
        id,
    )
    if err != nil {
        return nil, fmt.Errorf("querying order items: %w", err)
    }
    defer rows.Close()

    for rows.Next() {
        var item domain.OrderItem
        if err := rows.Scan(&item.ProductID, &item.Quantity, &item.PriceEach); err != nil {
            return nil, fmt.Errorf("scanning order item: %w", err)
        }
        order.Items = append(order.Items, item)
    }

    // IMPORTANT: Always check rows.Err() after the loop.
    // The loop might have ended due to an error, not because it ran out of rows.
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterating order items: %w", err)
    }

    return order, nil
}

// ListByCustomer returns paginated orders for a customer.
func (r *PostgresOrderRepository) ListByCustomer(
    ctx context.Context,
    customerID string,
    limit, offset int,
) ([]*domain.Order, error) {
    rows, err := r.db.QueryContext(ctx,
        `SELECT id, customer_id, status, total_cents, created_at, updated_at
         FROM orders WHERE customer_id = $1
         ORDER BY created_at DESC
         LIMIT $2 OFFSET $3`,
        customerID, limit, offset,
    )
    if err != nil {
        return nil, fmt.Errorf("querying orders: %w", err)
    }
    defer rows.Close()

    var orders []*domain.Order
    for rows.Next() {
        order := &domain.Order{}
        if err := rows.Scan(
            &order.ID, &order.CustomerID, &order.Status,
            &order.Total, &order.CreatedAt, &order.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scanning order: %w", err)
        }
        orders = append(orders, order)
    }

    return orders, rows.Err()
}

// UpdateStatus changes an order's status atomically.
func (r *PostgresOrderRepository) UpdateStatus(
    ctx context.Context,
    id string,
    status domain.OrderStatus,
) error {
    result, err := r.db.ExecContext(ctx,
        `UPDATE orders SET status = $1, updated_at = $2 WHERE id = $3`,
        status, time.Now(), id,
    )
    if err != nil {
        return fmt.Errorf("updating order status: %w", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("checking rows affected: %w", err)
    }
    if rowsAffected == 0 {
        return fmt.Errorf("order %s not found", id)
    }

    return nil
}
```

### Using sqlx -- database/sql with Less Boilerplate

`sqlx` extends `database/sql` with struct scanning and named queries. It does not hide SQL -- it just removes the repetitive `Scan` calls.

```go
import "github.com/jmoiron/sqlx"

// With database/sql: manual scanning
err := row.Scan(&order.ID, &order.CustomerID, &order.Status, &order.Total, ...)

// With sqlx: automatic struct scanning using `db` tags
type OrderRow struct {
    ID         string    `db:"id"`
    CustomerID string    `db:"customer_id"`
    Status     string    `db:"status"`
    TotalCents int64     `db:"total_cents"`
    CreatedAt  time.Time `db:"created_at"`
    UpdatedAt  time.Time `db:"updated_at"`
}

var order OrderRow
err := db.GetContext(ctx, &order, "SELECT * FROM orders WHERE id = $1", id)

// Named queries
_, err := db.NamedExecContext(ctx,
    `INSERT INTO orders (id, customer_id, status, total_cents)
     VALUES (:id, :customer_id, :status, :total_cents)`,
    order,
)
```

### Using GORM -- Full ORM

GORM is the most popular Go ORM. It provides ActiveRecord-style database operations. Opinions on GORM are polarized in the Go community: some teams love the productivity boost, others find it hides too much.

```go
import "gorm.io/gorm"

// Define models with GORM tags
type Order struct {
    ID         string      `gorm:"primaryKey;type:uuid"`
    CustomerID string      `gorm:"index;not null"`
    Items      []OrderItem `gorm:"foreignKey:OrderID"`
    Status     string      `gorm:"default:pending"`
    TotalCents int64
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type OrderItem struct {
    ID            uint   `gorm:"primaryKey"`
    OrderID       string `gorm:"index"`
    ProductID     string
    Quantity      int
    PriceEachCents int64
}

// CRUD with GORM
func CreateOrder(db *gorm.DB, order *Order) error {
    return db.Create(order).Error  // Inserts order + items in a transaction
}

func GetOrder(db *gorm.DB, id string) (*Order, error) {
    var order Order
    err := db.Preload("Items").First(&order, "id = ?", id).Error
    return &order, err
}

func ListOrders(db *gorm.DB, customerID string, page, pageSize int) ([]Order, error) {
    var orders []Order
    err := db.Where("customer_id = ?", customerID).
        Offset((page - 1) * pageSize).
        Limit(pageSize).
        Order("created_at DESC").
        Find(&orders).Error
    return orders, err
}
```

### Go database/sql vs Node.js Database Libraries

```
+------------------+----------------------------+----------------------------+
| Aspect           | Go                         | Node.js                    |
+------------------+----------------------------+----------------------------+
| Standard library | database/sql (built-in)    | None (need pg, mysql2)     |
| Connection pool  | Built into database/sql    | pg-pool, knex pool         |
| Raw SQL          | database/sql, sqlx         | pg, mysql2, knex.raw()     |
| Query builder    | squirrel, goqu             | Knex.js                    |
| Simple ORM       | sqlx + squirrel            | Prisma, Drizzle            |
| Full ORM         | GORM, ent                  | TypeORM, Sequelize         |
| Migrations       | golang-migrate, goose      | Prisma migrate, knex       |
| Type safety      | Compile-time (with sqlc)   | Runtime (or Prisma types)  |
+------------------+----------------------------+----------------------------+
```

**Node.js Prisma equivalent:**

```javascript
// Node.js with Prisma -- closest equivalent to GORM
const { PrismaClient } = require("@prisma/client");
const prisma = new PrismaClient();

// Create
const order = await prisma.order.create({
    data: {
        customerId: "cust-123",
        status: "pending",
        totalCents: 5000,
        items: {
            create: [
                { productId: "prod-1", quantity: 2, priceEachCents: 2500 },
            ],
        },
    },
    include: { items: true },
});

// Read with relations
const order = await prisma.order.findUnique({
    where: { id: "order-123" },
    include: { items: true },
});

// List with pagination
const orders = await prisma.order.findMany({
    where: { customerId: "cust-123" },
    skip: 0,
    take: 10,
    orderBy: { createdAt: "desc" },
});
```

**WHY many Go developers prefer raw SQL over ORMs:** Go's culture values explicitness. When you write `SELECT id, name FROM users WHERE active = true`, everyone on the team knows exactly what query hits the database. With an ORM, the generated query might include unexpected JOINs, N+1 queries, or suboptimal plans. Tools like `sqlc` give you type-safe Go code generated from raw SQL -- the best of both worlds.

---

## 7. Message Queues and Event-Driven Architecture

### WHY Event-Driven Architecture?

In a synchronous microservices system, one slow service blocks the entire request chain. Event-driven architecture decouples services using message queues:

```
Synchronous (fragile):
  Client → Order → Payment → Inventory → Notification → Client
  (If Payment is slow, everything waits. If Notification is down, order fails.)

Event-driven (resilient):
  Client → Order → "order.created" event
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Payment    Inventory  Notification
          (async)    (async)    (async)

  Order returns immediately. Each consumer processes at its own pace.
  If Notification is down, its messages queue up and are processed when
  it comes back.
```

### Working with NATS

NATS is a lightweight, high-performance messaging system popular in the Go ecosystem (NATS itself is written in Go).

```go
package messaging

import (
    "context"
    "encoding/json"
    "fmt"
    "log/slog"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

// --- Publisher ---

type NATSPublisher struct {
    js     jetstream.JetStream
    logger *slog.Logger
}

func NewNATSPublisher(nc *nats.Conn, logger *slog.Logger) (*NATSPublisher, error) {
    js, err := jetstream.New(nc)
    if err != nil {
        return nil, fmt.Errorf("creating jetstream context: %w", err)
    }

    return &NATSPublisher{js: js, logger: logger}, nil
}

func (p *NATSPublisher) Publish(ctx context.Context, subject string, data interface{}) error {
    payload, err := json.Marshal(data)
    if err != nil {
        return fmt.Errorf("marshaling event: %w", err)
    }

    ack, err := p.js.Publish(ctx, subject, payload)
    if err != nil {
        return fmt.Errorf("publishing to %s: %w", subject, err)
    }

    p.logger.Debug("event published",
        "subject", subject,
        "sequence", ack.Sequence,
    )
    return nil
}

// --- Consumer ---

type NATSConsumer struct {
    js     jetstream.JetStream
    logger *slog.Logger
}

func NewNATSConsumer(nc *nats.Conn, logger *slog.Logger) (*NATSConsumer, error) {
    js, err := jetstream.New(nc)
    if err != nil {
        return nil, fmt.Errorf("creating jetstream context: %w", err)
    }
    return &NATSConsumer{js: js, logger: logger}, nil
}

// Subscribe starts consuming messages from a stream.
// The handler processes each message. Return nil to acknowledge, error to NAK.
func (c *NATSConsumer) Subscribe(
    ctx context.Context,
    streamName string,
    consumerName string,
    handler func(ctx context.Context, msg []byte) error,
) error {
    // Get or create the consumer
    cons, err := c.js.CreateOrUpdateConsumer(ctx, streamName, jetstream.ConsumerConfig{
        Durable:       consumerName,
        AckPolicy:     jetstream.AckExplicitPolicy,
        MaxDeliver:    5,                    // Retry up to 5 times
        AckWait:       30 * time.Second,     // Time to process before redelivery
        FilterSubject: streamName + ".>",
    })
    if err != nil {
        return fmt.Errorf("creating consumer: %w", err)
    }

    // Consume messages
    iter, err := cons.Messages()
    if err != nil {
        return fmt.Errorf("creating message iterator: %w", err)
    }

    go func() {
        <-ctx.Done()
        iter.Stop()
    }()

    for {
        msg, err := iter.Next()
        if err != nil {
            if ctx.Err() != nil {
                return nil // Context cancelled -- clean shutdown
            }
            c.logger.Error("error getting next message", "err", err)
            continue
        }

        if err := handler(ctx, msg.Data()); err != nil {
            c.logger.Error("handler error",
                "subject", msg.Subject(),
                "err", err,
            )
            msg.Nak() // Negative acknowledgment -- will be redelivered
        } else {
            msg.Ack()
        }
    }
}
```

### Working with Kafka

```go
package messaging

import (
    "context"
    "encoding/json"
    "fmt"
    "log/slog"

    "github.com/segmentio/kafka-go"
)

type KafkaPublisher struct {
    writer *kafka.Writer
    logger *slog.Logger
}

func NewKafkaPublisher(brokers []string, logger *slog.Logger) *KafkaPublisher {
    writer := &kafka.Writer{
        Addr:         kafka.TCP(brokers...),
        Balancer:     &kafka.LeastBytes{},
        BatchSize:    100,
        BatchTimeout: 10 * time.Millisecond,
        RequiredAcks: kafka.RequireAll,  // Wait for all replicas
        Async:        false,             // Synchronous for reliability
    }

    return &KafkaPublisher{writer: writer, logger: logger}
}

func (p *KafkaPublisher) Publish(ctx context.Context, topic string, key string, data interface{}) error {
    value, err := json.Marshal(data)
    if err != nil {
        return fmt.Errorf("marshaling message: %w", err)
    }

    err = p.writer.WriteMessages(ctx, kafka.Message{
        Topic: topic,
        Key:   []byte(key),    // Key determines partition (same key = same partition = ordering)
        Value: value,
    })
    if err != nil {
        return fmt.Errorf("writing to kafka: %w", err)
    }

    return nil
}

func (p *KafkaPublisher) Close() error {
    return p.writer.Close()
}

type KafkaConsumer struct {
    reader *kafka.Reader
    logger *slog.Logger
}

func NewKafkaConsumer(brokers []string, topic, groupID string, logger *slog.Logger) *KafkaConsumer {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:     brokers,
        Topic:       topic,
        GroupID:     groupID,           // Consumer group for load balancing
        MinBytes:    1,
        MaxBytes:    10e6,              // 10MB
        StartOffset: kafka.LastOffset,  // Start from latest if no offset stored
    })

    return &KafkaConsumer{reader: reader, logger: logger}
}

func (c *KafkaConsumer) Consume(ctx context.Context, handler func(ctx context.Context, key, value []byte) error) error {
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                return nil // Context cancelled
            }
            return fmt.Errorf("fetching message: %w", err)
        }

        if err := handler(ctx, msg.Key, msg.Value); err != nil {
            c.logger.Error("handler error",
                "topic", msg.Topic,
                "partition", msg.Partition,
                "offset", msg.Offset,
                "err", err,
            )
            // In production: implement dead-letter queue or retry logic
            continue
        }

        // Commit the offset only after successful processing
        if err := c.reader.CommitMessages(ctx, msg); err != nil {
            c.logger.Error("commit error", "err", err)
        }
    }
}

func (c *KafkaConsumer) Close() error {
    return c.reader.Close()
}
```

### Using Messages in the Order Service

```go
// internal/service/order_events.go
package service

import "context"

// EventPublisher is the interface the service depends on.
// It does not know if it is NATS, Kafka, or RabbitMQ.
type EventPublisher interface {
    Publish(ctx context.Context, subject string, data interface{}) error
}

// Order event types
type OrderCreatedEvent struct {
    OrderID    string `json:"order_id"`
    CustomerID string `json:"customer_id"`
    TotalCents int64  `json:"total_cents"`
}

type OrderStatusChangedEvent struct {
    OrderID   string `json:"order_id"`
    OldStatus string `json:"old_status"`
    NewStatus string `json:"new_status"`
}
```

```go
// Payment service consuming order events
func handleOrderCreated(ctx context.Context, data []byte) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal(data, &event); err != nil {
        return fmt.Errorf("unmarshaling event: %w", err)
    }

    // Process payment
    log.Printf("Processing payment for order %s: %d cents",
        event.OrderID, event.TotalCents)

    // ... charge the customer ...

    return nil
}
```

### Comparison with Node.js Message Queue Clients

```javascript
// Node.js Kafka consumer (using kafkajs)
const { Kafka } = require("kafkajs");

const kafka = new Kafka({ brokers: ["kafka:9092"] });
const consumer = kafka.consumer({ groupId: "payment-service" });

await consumer.connect();
await consumer.subscribe({ topic: "orders", fromBeginning: false });

await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
        const event = JSON.parse(message.value.toString());
        console.log(`Processing payment for order ${event.order_id}`);
        // ... process payment ...
    },
});
```

Key differences:
- Go's Kafka libraries (`segmentio/kafka-go`, `confluent-kafka-go`) are compiled and faster than Node.js's `kafkajs`.
- Go consumers naturally handle concurrent message processing with goroutines. Node.js processes messages sequentially unless you add explicit concurrency.
- Go's strong typing catches event schema mismatches at compile time. Node.js discovers them at runtime.

---

## 8. Observability

Observability is the ability to understand a running system's internal state from its external outputs. For microservices, this means three pillars: **logs**, **traces**, and **metrics**.

```
                    The Three Pillars of Observability

    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │    Logs      │    │   Traces     │    │   Metrics    │
    │              │    │              │    │              │
    │ What         │    │ Request flow │    │ Aggregated   │
    │ happened     │    │ across       │    │ measurements │
    │ (discrete    │    │ services     │    │ over time    │
    │  events)     │    │ (causal      │    │ (counters,   │
    │              │    │  chains)     │    │  histograms) │
    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
           │                   │                    │
           ▼                   ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │  Loki /      │    │   Jaeger /   │    │  Prometheus  │
    │  ELK Stack   │    │   Tempo      │    │  + Grafana   │
    └──────────────┘    └──────────────┘    └──────────────┘
```

### Structured Logging with slog

Go 1.21 added `log/slog` to the standard library, providing structured logging without third-party dependencies.

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

func main() {
    // JSON handler for production (machine-readable)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        // AddSource adds file:line to every log entry
        AddSource: true,
    }))

    // Set as default logger
    slog.SetDefault(logger)

    // Basic structured logging
    logger.Info("server starting",
        "port", 8080,
        "version", "1.2.3",
        "environment", "production",
    )
    // Output: {"time":"2025-01-15T10:30:00Z","level":"INFO","source":{"function":"main.main",
    //   "file":"main.go","line":20},"msg":"server starting","port":8080,"version":"1.2.3",
    //   "environment":"production"}

    // Logger with persistent attributes (context)
    requestLogger := logger.With(
        "request_id", "req-abc-123",
        "user_id", "user-456",
    )
    requestLogger.Info("processing order", "order_id", "ord-789")
    // Every log from requestLogger includes request_id and user_id

    // Group related attributes
    logger.Info("database query",
        slog.Group("query",
            slog.String("sql", "SELECT * FROM orders"),
            slog.Duration("duration", 15*time.Millisecond),
            slog.Int("rows", 42),
        ),
    )

    // Error logging with error value
    logger.Error("failed to process order",
        "order_id", "ord-789",
        "err", err,
        "retry_count", 3,
    )

    // Logging levels
    logger.Debug("detailed debug info")  // Hidden at Info level
    logger.Info("normal operation")
    logger.Warn("degraded performance", "latency_ms", 500)
    logger.Error("operation failed", "err", err)
}
```

### Context-Aware Logging

In microservices, every log line should carry a trace ID and request ID for correlation:

```go
// internal/middleware/logging.go
package middleware

import (
    "context"
    "log/slog"
    "net/http"
)

type contextKey string

const loggerKey contextKey = "logger"

// InjectLogger creates a request-scoped logger with trace context.
func InjectLogger(baseLogger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            requestID := r.Header.Get("X-Request-ID")
            traceID := r.Header.Get("X-Trace-ID")

            // Create a logger enriched with request context
            logger := baseLogger.With(
                "request_id", requestID,
                "trace_id", traceID,
                "method", r.Method,
                "path", r.URL.Path,
            )

            // Store in context
            ctx := context.WithValue(r.Context(), loggerKey, logger)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// LoggerFromContext retrieves the request-scoped logger.
func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}
```

Usage in a handler:

```go
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    logger := middleware.LoggerFromContext(r.Context())

    logger.Info("creating order")

    order, err := h.svc.CreateOrder(r.Context(), req)
    if err != nil {
        logger.Error("order creation failed", "err", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    logger.Info("order created", "order_id", order.ID)
}
```

### Alternative: zerolog for High-Performance Logging

For services with extreme logging throughput (millions of log lines per second), zerolog is zero-allocation:

```go
import "github.com/rs/zerolog"

logger := zerolog.New(os.Stdout).With().Timestamp().Logger()

logger.Info().
    Str("order_id", "ord-789").
    Int64("total_cents", 5000).
    Dur("duration", elapsed).
    Msg("order created")
```

### Distributed Tracing with OpenTelemetry

When a single request passes through 5 services, you need to trace the entire journey. OpenTelemetry (OTel) is the industry standard.

```go
package tracing

import (
    "context"
    "fmt"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
    "go.opentelemetry.io/otel/trace"
)

// InitTracer sets up the OpenTelemetry tracing pipeline.
func InitTracer(ctx context.Context, serviceName, version, otelEndpoint string) (func(), error) {
    // Create OTLP exporter (sends traces to Jaeger, Tempo, etc.)
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint(otelEndpoint),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, fmt.Errorf("creating exporter: %w", err)
    }

    // Define the resource (identifies this service)
    res, err := resource.Merge(
        resource.Default(),
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(version),
            attribute.String("environment", "production"),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("creating resource: %w", err)
    }

    // Create the tracer provider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1)), // Sample 10% of traces
    )

    // Set as global tracer provider
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    // Return a cleanup function
    cleanup := func() {
        tp.Shutdown(context.Background())
    }
    return cleanup, nil
}
```

Using traces in your service:

```go
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Start a new span
    ctx, span := otel.Tracer("order-service").Start(ctx, "CreateOrder")
    defer span.End()

    // Add attributes to the span
    span.SetAttributes(
        attribute.String("customer_id", req.CustomerID),
        attribute.Int("item_count", len(req.Items)),
    )

    // Validate (sub-span)
    ctx, validateSpan := otel.Tracer("order-service").Start(ctx, "ValidateOrder")
    if err := s.validate(req); err != nil {
        validateSpan.RecordError(err)
        validateSpan.End()
        return nil, err
    }
    validateSpan.End()

    // Save to database (sub-span)
    ctx, dbSpan := otel.Tracer("order-service").Start(ctx, "SaveOrder")
    order, err := s.repo.Create(ctx, &Order{...})
    if err != nil {
        dbSpan.RecordError(err)
        dbSpan.End()
        return nil, err
    }
    dbSpan.End()

    // Publish event (the trace context propagates automatically)
    s.events.Publish(ctx, "order.created", order)

    span.SetAttributes(attribute.String("order_id", order.ID))
    return order, nil
}
```

The trace context propagates across service boundaries through HTTP headers or gRPC metadata. When the payment service receives the event, it continues the same trace.

### Metrics with Prometheus

```go
package metrics

import (
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Counter: total requests
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    // Histogram: request duration
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )

    // Gauge: active connections
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )

    // Business metrics
    ordersCreatedTotal = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
    )

    orderTotalCents = promauto.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "order_total_cents",
            Help:    "Distribution of order totals in cents",
            Buckets: []float64{100, 500, 1000, 5000, 10000, 50000, 100000},
        },
    )
)

// MetricsMiddleware records HTTP metrics for every request.
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        activeConnections.Inc()
        defer activeConnections.Dec()

        recorder := &statusRecorder{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(recorder, r)

        duration := time.Since(start).Seconds()
        statusStr := fmt.Sprintf("%d", recorder.statusCode)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, statusStr).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

// ExposeMetrics returns an HTTP handler that serves Prometheus metrics.
func ExposeMetrics() http.Handler {
    return promhttp.Handler()
}
```

Mount the metrics endpoint:

```go
mux := http.NewServeMux()
mux.Handle("/metrics", metrics.ExposeMetrics())  // Prometheus scrapes this
mux.Handle("/", metrics.MetricsMiddleware(appHandler))
```

### Go vs Node.js Observability

```
+--------------------+-------------------------------+-------------------------------+
| Aspect             | Go                            | Node.js                       |
+--------------------+-------------------------------+-------------------------------+
| Structured logging | slog (stdlib), zerolog         | Pino, Winston                 |
| Log performance    | Zero-alloc (zerolog)           | Pino is fast but has allocs   |
| Tracing            | OTel Go SDK                    | OTel JS SDK                   |
| Metrics            | prometheus/client_golang       | prom-client                   |
| CPU profiling      | Built-in (pprof)               | --inspect, clinic.js          |
| Memory profiling   | Built-in (pprof)               | --inspect, heapdump           |
| Goroutine/thread   | pprof goroutine dump           | No equivalent                 |
+--------------------+-------------------------------+-------------------------------+
```

Go has a major observability advantage: `net/http/pprof` is built into the standard library and gives you CPU profiles, memory profiles, goroutine dumps, and block profiles in production with near-zero overhead. Node.js requires external tools and typically cannot profile safely in production.

---

## 9. Configuration Management

### The 12-Factor App Approach

The 12-Factor App methodology (authored at Heroku) states that configuration should come from the environment, not from code or config files. Go makes this straightforward.

### A Complete Configuration Package

```go
// config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

type Config struct {
    // Server
    ServerPort         int
    ServerReadTimeout  time.Duration
    ServerWriteTimeout time.Duration

    // Database
    DatabaseURL          string
    DatabaseMaxOpenConns int
    DatabaseMaxIdleConns int

    // gRPC
    GRPCPort int

    // Message Queue
    NATSUrl   string
    KafkaBrokers []string

    // Observability
    OTelEndpoint string
    LogLevel     string

    // Feature Flags
    EnableNewCheckout bool
}

// Load reads configuration from environment variables.
// It returns an error if any required variable is missing.
func Load() (*Config, error) {
    cfg := &Config{}
    var errs []string

    // Required variables
    cfg.DatabaseURL = mustEnv("DATABASE_URL", &errs)

    // Optional variables with defaults
    cfg.ServerPort = envInt("SERVER_PORT", 8080)
    cfg.ServerReadTimeout = envDuration("SERVER_READ_TIMEOUT", 5*time.Second)
    cfg.ServerWriteTimeout = envDuration("SERVER_WRITE_TIMEOUT", 10*time.Second)
    cfg.DatabaseMaxOpenConns = envInt("DATABASE_MAX_OPEN_CONNS", 25)
    cfg.DatabaseMaxIdleConns = envInt("DATABASE_MAX_IDLE_CONNS", 10)
    cfg.GRPCPort = envInt("GRPC_PORT", 50051)
    cfg.NATSUrl = envString("NATS_URL", "nats://localhost:4222")
    cfg.KafkaBrokers = envStringSlice("KAFKA_BROKERS", []string{"localhost:9092"})
    cfg.OTelEndpoint = envString("OTEL_ENDPOINT", "localhost:4317")
    cfg.LogLevel = envString("LOG_LEVEL", "info")
    cfg.EnableNewCheckout = envBool("ENABLE_NEW_CHECKOUT", false)

    if len(errs) > 0 {
        return nil, fmt.Errorf("missing required config: %v", errs)
    }

    return cfg, nil
}

// Helper functions

func mustEnv(key string, errs *[]string) string {
    val := os.Getenv(key)
    if val == "" {
        *errs = append(*errs, key)
    }
    return val
}

func envString(key, fallback string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return fallback
}

func envInt(key string, fallback int) int {
    if val := os.Getenv(key); val != "" {
        if i, err := strconv.Atoi(val); err == nil {
            return i
        }
    }
    return fallback
}

func envBool(key string, fallback bool) bool {
    if val := os.Getenv(key); val != "" {
        if b, err := strconv.ParseBool(val); err == nil {
            return b
        }
    }
    return fallback
}

func envDuration(key string, fallback time.Duration) time.Duration {
    if val := os.Getenv(key); val != "" {
        if d, err := time.ParseDuration(val); err == nil {
            return d
        }
    }
    return fallback
}

func envStringSlice(key string, fallback []string) []string {
    if val := os.Getenv(key); val != "" {
        return strings.Split(val, ",")
    }
    return fallback
}
```

### Using Viper for Complex Configuration

Viper is the most popular Go configuration library. It supports environment variables, config files (YAML, JSON, TOML), remote config stores (etcd, Consul), and live reloading.

```go
package config

import (
    "fmt"
    "strings"

    "github.com/spf13/viper"
)

func LoadWithViper() (*Config, error) {
    v := viper.New()

    // Config file (optional -- environment variables override)
    v.SetConfigName("config")
    v.SetConfigType("yaml")
    v.AddConfigPath(".")
    v.AddConfigPath("/etc/order-service/")

    // Environment variables override config file
    v.AutomaticEnv()
    v.SetEnvPrefix("ORDER_SVC")                     // ORDER_SVC_SERVER_PORT
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_")) // server.port -> SERVER_PORT

    // Defaults
    v.SetDefault("server.port", 8080)
    v.SetDefault("server.read_timeout", "5s")
    v.SetDefault("database.max_open_conns", 25)
    v.SetDefault("log.level", "info")

    // Read config file (not an error if it does not exist)
    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("reading config: %w", err)
        }
    }

    cfg := &Config{
        ServerPort:           v.GetInt("server.port"),
        ServerReadTimeout:    v.GetDuration("server.read_timeout"),
        ServerWriteTimeout:   v.GetDuration("server.write_timeout"),
        DatabaseURL:          v.GetString("database.url"),
        DatabaseMaxOpenConns: v.GetInt("database.max_open_conns"),
        GRPCPort:             v.GetInt("grpc.port"),
        LogLevel:             v.GetString("log.level"),
    }

    if cfg.DatabaseURL == "" {
        return nil, fmt.Errorf("database.url is required")
    }

    return cfg, nil
}
```

Corresponding YAML config file:

```yaml
# config.yaml
server:
  port: 8080
  read_timeout: 5s
  write_timeout: 10s

database:
  url: postgres://localhost:5432/orders
  max_open_conns: 25
  max_idle_conns: 10

grpc:
  port: 50051

log:
  level: info
```

### WHY Environment Variables Over Config Files

Config files create a problem: you need different files for development, staging, and production, and those files must be managed, versioned, and deployed separately. Environment variables are set by the deployment environment (Kubernetes, Docker, Heroku) and follow the service:

```yaml
# Kubernetes deployment -- environment variables
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: order-service
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: url
        - name: SERVER_PORT
          value: "8080"
        - name: LOG_LEVEL
          value: "info"
        - name: NATS_URL
          value: "nats://nats.messaging:4222"
```

---

## 10. Graceful Shutdown

### WHY Graceful Shutdown Matters

When Kubernetes sends a SIGTERM to your pod (during scaling, deployment, or node maintenance), you have a grace period (default 30 seconds) to finish in-flight work before SIGKILL terminates the process. Without graceful shutdown:

- In-flight HTTP requests return errors to clients
- Database transactions are left incomplete
- Messages are lost or double-processed
- WebSocket connections drop without notice

### Complete Graceful Shutdown Implementation

```go
// cmd/server/main.go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log/slog"
    "net"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    "google.golang.org/grpc"
    _ "github.com/lib/pq"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        logger.Error("failed to load config", "err", err)
        os.Exit(1)
    }

    // --- Initialize dependencies ---

    // Database
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        logger.Error("failed to open database", "err", err)
        os.Exit(1)
    }
    db.SetMaxOpenConns(cfg.DatabaseMaxOpenConns)
    db.SetMaxIdleConns(cfg.DatabaseMaxIdleConns)
    db.SetConnMaxLifetime(5 * time.Minute)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        logger.Error("database unreachable", "err", err)
        os.Exit(1)
    }
    logger.Info("database connected")

    // Create service layer (business logic)
    orderRepo := repository.NewPostgresOrderRepository(db)
    orderService := service.NewOrderService(orderRepo, eventPublisher)

    // --- Start servers ---

    // HTTP server
    httpHandler := handler.NewOrderHTTPHandler(orderService)
    httpServer := &http.Server{
        Addr:         fmt.Sprintf(":%d", cfg.ServerPort),
        Handler:      httpHandler,
        ReadTimeout:  cfg.ServerReadTimeout,
        WriteTimeout: cfg.ServerWriteTimeout,
        IdleTimeout:  120 * time.Second,
    }

    // gRPC server
    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            middleware.UnaryLoggingInterceptor(logger),
            middleware.UnaryRecoveryInterceptor(),
        ),
    )
    grpcHandler := handler.NewOrderGRPCHandler(orderService)
    orderv1.RegisterOrderServiceServer(grpcServer, grpcHandler)

    grpcLis, err := net.Listen("tcp", fmt.Sprintf(":%d", cfg.GRPCPort))
    if err != nil {
        logger.Error("failed to listen for gRPC", "err", err)
        os.Exit(1)
    }

    // Start HTTP server in a goroutine
    go func() {
        logger.Info("HTTP server starting", "addr", httpServer.Addr)
        if err := httpServer.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("HTTP server error", "err", err)
        }
    }()

    // Start gRPC server in a goroutine
    go func() {
        logger.Info("gRPC server starting", "port", cfg.GRPCPort)
        if err := grpcServer.Serve(grpcLis); err != nil {
            logger.Error("gRPC server error", "err", err)
        }
    }()

    // --- Graceful shutdown ---

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    logger.Info("shutdown signal received", "signal", sig.String())

    // Create a deadline for the shutdown process
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer shutdownCancel()

    // Shutdown in order: stop accepting new requests, then drain existing ones
    var wg sync.WaitGroup

    // 1. Stop HTTP server (stops accepting new connections, waits for in-flight requests)
    wg.Add(1)
    go func() {
        defer wg.Done()
        logger.Info("shutting down HTTP server")
        if err := httpServer.Shutdown(shutdownCtx); err != nil {
            logger.Error("HTTP shutdown error", "err", err)
        }
        logger.Info("HTTP server stopped")
    }()

    // 2. Stop gRPC server (gracefully completes in-flight RPCs)
    wg.Add(1)
    go func() {
        defer wg.Done()
        logger.Info("shutting down gRPC server")
        grpcServer.GracefulStop()
        logger.Info("gRPC server stopped")
    }()

    // Wait for servers to drain
    wg.Wait()

    // 3. Close message queue connections
    if eventPublisher != nil {
        logger.Info("closing event publisher")
        eventPublisher.Close()
    }

    // 4. Close database connections (after all queries are done)
    logger.Info("closing database")
    if err := db.Close(); err != nil {
        logger.Error("database close error", "err", err)
    }

    logger.Info("shutdown complete")
}
```

### How http.Server.Shutdown Works

`Shutdown` does the following, in order:

1. Closes the listener (stops accepting new connections)
2. Closes all idle connections
3. Waits for all active connections to become idle (i.e., in-flight requests complete)
4. Returns when all connections are closed or the context deadline is reached

If the context deadline passes before all connections close, `Shutdown` returns an error and remaining connections are forcefully closed. This is why you set a 30-second shutdown timeout -- it gives in-flight requests time to complete without waiting forever.

### Node.js Comparison: Graceful Shutdown

```javascript
// Node.js graceful shutdown (Express)
const server = app.listen(8080);

const shutdown = async (signal) => {
    console.log(`Received ${signal}, shutting down gracefully`);

    // Stop accepting new connections
    server.close(() => {
        console.log("HTTP server closed");
    });

    // Close database
    await prisma.$disconnect();

    // Close message queue
    await kafkaConsumer.disconnect();

    // Force exit after timeout
    setTimeout(() => {
        console.error("Forced shutdown after timeout");
        process.exit(1);
    }, 30000);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

Key differences:
- Go's `http.Server.Shutdown` has built-in support for draining connections. Node.js's `server.close()` stops accepting new connections but does not wait for in-flight requests unless you track them manually.
- Go handles shutdown with `sync.WaitGroup` for coordination. Node.js uses `setTimeout` as a fallback, which is less precise.
- Go's context-based timeout is cleaner than a manual `setTimeout`.

---

## 11. Health Checks and Readiness

### WHY Two Different Probes?

Kubernetes uses two types of health checks:

- **Liveness probe:** "Is this process stuck?" If it fails, Kubernetes restarts the pod.
- **Readiness probe:** "Can this process handle traffic?" If it fails, Kubernetes removes the pod from the service's load balancer but does NOT restart it.

```
                        ┌─────────────────────────────┐
                        │        Kubernetes            │
                        │                             │
                        │   /healthz (liveness)        │
                        │   "Is the process alive?"    │
                        │   Fail → restart the pod     │
                        │                             │
                        │   /readyz (readiness)        │
                        │   "Can it serve traffic?"    │
                        │   Fail → remove from LB      │
                        │          (no restart)         │
                        └─────────────────────────────┘

Example scenarios:
  - Service is alive but database is down → liveness: OK, readiness: FAIL
    (Kubernetes removes from LB, does not restart -- restarting would not fix the DB)
  - Service is stuck in a deadlock → liveness: FAIL (timeout)
    (Kubernetes restarts the pod)
  - Service is starting up and warming cache → liveness: OK, readiness: FAIL
    (Kubernetes waits until readiness passes before sending traffic)
```

### Implementation

```go
// internal/health/health.go
package health

import (
    "context"
    "database/sql"
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

type Status string

const (
    StatusUp   Status = "up"
    StatusDown Status = "down"
)

type HealthChecker struct {
    db       *sql.DB
    checks   map[string]CheckFunc
    mu       sync.RWMutex
    ready    bool
}

type CheckFunc func(ctx context.Context) error

type HealthResponse struct {
    Status Status            `json:"status"`
    Checks map[string]string `json:"checks,omitempty"`
}

func NewHealthChecker(db *sql.DB) *HealthChecker {
    h := &HealthChecker{
        db:     db,
        checks: make(map[string]CheckFunc),
    }

    // Register default checks
    h.AddCheck("database", h.checkDatabase)

    return h
}

func (h *HealthChecker) AddCheck(name string, check CheckFunc) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.checks[name] = check
}

func (h *HealthChecker) SetReady(ready bool) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.ready = ready
}

// LivenessHandler responds to /healthz.
// This should be lightweight -- it only checks if the process is alive.
func (h *HealthChecker) LivenessHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(HealthResponse{Status: StatusUp})
    }
}

// ReadinessHandler responds to /readyz.
// This checks all dependencies (database, cache, message queue).
func (h *HealthChecker) ReadinessHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        h.mu.RLock()
        ready := h.ready
        checks := make(map[string]CheckFunc, len(h.checks))
        for k, v := range h.checks {
            checks[k] = v
        }
        h.mu.RUnlock()

        if !ready {
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(HealthResponse{
                Status: StatusDown,
                Checks: map[string]string{"ready": "service not ready"},
            })
            return
        }

        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        resp := HealthResponse{
            Status: StatusUp,
            Checks: make(map[string]string),
        }
        allHealthy := true

        for name, check := range checks {
            if err := check(ctx); err != nil {
                resp.Checks[name] = err.Error()
                allHealthy = false
            } else {
                resp.Checks[name] = "ok"
            }
        }

        w.Header().Set("Content-Type", "application/json")
        if !allHealthy {
            resp.Status = StatusDown
            w.WriteHeader(http.StatusServiceUnavailable)
        } else {
            w.WriteHeader(http.StatusOK)
        }
        json.NewEncoder(w).Encode(resp)
    }
}

func (h *HealthChecker) checkDatabase(ctx context.Context) error {
    return h.db.PingContext(ctx)
}
```

Register the health endpoints:

```go
healthChecker := health.NewHealthChecker(db)

mux.HandleFunc("GET /healthz", healthChecker.LivenessHandler())
mux.HandleFunc("GET /readyz", healthChecker.ReadinessHandler())

// After all initialization is complete:
healthChecker.SetReady(true)

// During shutdown, mark as not ready BEFORE stopping the server:
// This gives Kubernetes time to remove the pod from the load balancer
// before the server stops accepting connections.
healthChecker.SetReady(false)
time.Sleep(5 * time.Second)  // Wait for Kubernetes to notice
httpServer.Shutdown(ctx)
```

### Kubernetes Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5      # Wait 5s before first check
          periodSeconds: 10            # Check every 10s
          timeoutSeconds: 3            # Timeout after 3s
          failureThreshold: 3          # Restart after 3 consecutive failures
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2          # Remove from LB after 2 failures
          successThreshold: 1          # Add back after 1 success
```

---

## 12. Docker and Deployment

### Multi-Stage Docker Build (Go's Killer Feature)

This is where Go's compiled, statically-linked binary model truly shines. A multi-stage Docker build compiles the Go binary in one stage and copies just the binary into a minimal final image.

```dockerfile
# Dockerfile for Go microservice

# --- Stage 1: Build ---
FROM golang:1.23-alpine AS builder

# Install git (needed for go mod download if using private repos)
RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Copy go.mod and go.sum first (Docker layer caching)
# These change rarely, so this layer is cached most of the time.
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build a statically linked binary
# CGO_ENABLED=0: no C dependencies -- pure Go binary
# -ldflags="-s -w": strip debug info and symbol table (smaller binary)
# -o /app/server: output binary location
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=1.2.3 -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -o /app/server \
    ./cmd/server/

# --- Stage 2: Runtime ---
FROM scratch

# Copy CA certificates (needed for HTTPS calls to external services)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the binary
COPY --from=builder /app/server /server

# Copy migrations if needed
COPY --from=builder /app/migrations /migrations

# Non-root user (security best practice)
# scratch does not have useradd, so we copy the user from the builder
COPY --from=builder /etc/passwd /etc/passwd
USER nobody

EXPOSE 8080 50051

ENTRYPOINT ["/server"]
```

**WHY `FROM scratch`?** The `scratch` image is completely empty -- no shell, no OS, no libraries. Since Go compiles to a statically linked binary, it does not need any of those. The result is the smallest possible Docker image.

```bash
$ docker build -t order-service:latest .
$ docker images order-service
REPOSITORY      TAG       SIZE
order-service   latest    8.2MB    # <-- The entire service is 8MB
```

### Alpine-Based Image (When You Need a Shell)

Sometimes you need a shell for debugging or `ca-certificates` for TLS:

```dockerfile
# If you need a shell for debugging
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/server /server
USER nobody
ENTRYPOINT ["/server"]
```

This adds ~7MB for Alpine, resulting in a ~15MB image.

### Docker Image Size Comparison

```
+------------------------------------------------------+
|         Docker Image Size Comparison                  |
+------------------------------------------------------+
|                                                      |
| Go (scratch)         ██░░░░░░░░░░░░░░░░  ~8 MB      |
| Go (Alpine)          ███░░░░░░░░░░░░░░░  ~15 MB     |
| Go (Distroless)      ██░░░░░░░░░░░░░░░░  ~12 MB     |
|                                                      |
| Node.js (Alpine)     ████████░░░░░░░░░░  ~120 MB    |
| Node.js (slim)       ██████████░░░░░░░░  ~180 MB    |
| Node.js (default)    █████████████████░  ~350 MB    |
| Node.js (w/ devDeps) ███████████████████ ~900 MB    |
|                                                      |
| Java (JRE Alpine)    ██████████░░░░░░░░  ~200 MB    |
| Java (Spring Boot)   ████████████████░░  ~400 MB    |
+------------------------------------------------------+
```

**WHY image size matters at scale:**
- **Pull time:** When Kubernetes scales from 3 to 30 pods, it must pull the image on each new node. An 8MB image pulls in under a second. A 350MB image takes 10-30 seconds.
- **Storage:** 50 microservices x 30 versions retained x 350MB = 525GB of registry storage. With Go: 12GB.
- **Security surface:** A `scratch` image has zero CVEs because it contains zero packages. A Node.js image inherits vulnerabilities from the base OS, Node.js runtime, and all npm packages.

### Node.js Docker Comparison

```dockerfile
# Dockerfile for Node.js microservice (optimized)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build  # TypeScript compilation

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
EXPOSE 8080
CMD ["node", "dist/index.js"]
# Result: ~120-180MB (Node.js runtime + node_modules + source)
```

Even with an optimized multi-stage build, the Node.js image is 10-20x larger than the Go image because:
1. Node.js needs the V8 runtime (~60MB)
2. `node_modules/` is typically 30-100MB
3. Alpine base image adds ~7MB

### Kubernetes Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: order-service
        image: registry.example.com/order-service:1.2.3
        ports:
        - name: http
          containerPort: 8080
        - name: grpc
          containerPort: 50051
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db
              key: url
        - name: LOG_LEVEL
          value: "info"
        - name: NATS_URL
          value: "nats://nats.messaging:4222"
        resources:
          requests:
            cpu: "100m"        # 0.1 CPU cores
            memory: "64Mi"     # Go services need very little memory
          limits:
            cpu: "500m"
            memory: "128Mi"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 50051
    targetPort: 50051
```

**Notice the resource requests:** Go services request 64Mi of memory. A Node.js service typically needs 256Mi-512Mi. A Java/Spring service needs 512Mi-1Gi. This means you can run 4-8x more Go pods per cluster node compared to Node.js, and 8-16x more compared to Java.

### Makefile for Development Workflow

```makefile
# Makefile
.PHONY: build run test lint proto docker

# Build the binary
build:
	go build -ldflags="-s -w" -o bin/server ./cmd/server/

# Run locally
run:
	go run ./cmd/server/

# Run tests
test:
	go test -v -race -count=1 ./...

# Run tests with coverage
test-coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

# Lint
lint:
	golangci-lint run ./...

# Generate protobuf code
proto:
	protoc --go_out=. --go_opt=paths=source_relative \
	       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
	       api/proto/**/*.proto

# Build Docker image
docker:
	docker build -t order-service:latest .

# Run with Docker Compose (full stack)
up:
	docker-compose up -d

down:
	docker-compose down

# Database migrations
migrate-up:
	migrate -path migrations -database "$(DATABASE_URL)" up

migrate-down:
	migrate -path migrations -database "$(DATABASE_URL)" down 1
```

---

## 13. Go vs Node.js Microservices Comparison

### Complete Side-by-Side: A Simple Microservice

**Go -- Complete microservice with health checks, graceful shutdown, and structured logging:**

```go
package main

import (
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type Order struct {
    ID         string `json:"id"`
    CustomerID string `json:"customer_id"`
    Total      int64  `json:"total"`
    Status     string `json:"status"`
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    mux := http.NewServeMux()

    mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })

    mux.HandleFunc("GET /api/orders/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        logger.Info("fetching order", "id", id)

        order := Order{
            ID: id, CustomerID: "cust-1", Total: 5000, Status: "pending",
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(order)
    })

    mux.HandleFunc("POST /api/orders", func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            CustomerID string `json:"customer_id"`
            Total      int64  `json:"total"`
        }
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "invalid request", http.StatusBadRequest)
            return
        }

        logger.Info("creating order", "customer_id", req.CustomerID, "total", req.Total)

        order := Order{
            ID: "ord-123", CustomerID: req.CustomerID, Total: req.Total, Status: "pending",
        }

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(order)
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    go func() {
        logger.Info("server starting", "addr", ":8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("server error", "err", err)
            os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    logger.Info("shutting down")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    server.Shutdown(ctx)
    logger.Info("server stopped")
}
// Dependencies: ZERO (standard library only)
// Binary size: ~6MB
// Docker image: ~8MB (scratch)
// Memory at rest: ~8MB
// Startup time: ~10ms
```

**Node.js -- Same microservice:**

```javascript
// package.json dependencies: express, pino, pino-pretty
const express = require("express");
const pino = require("pino");

const logger = pino({ level: "info" });
const app = express();
app.use(express.json());

app.get("/healthz", (req, res) => {
    res.json({ status: "ok" });
});

app.get("/api/orders/:id", (req, res) => {
    const { id } = req.params;
    logger.info({ id }, "fetching order");

    const order = {
        id,
        customer_id: "cust-1",
        total: 5000,
        status: "pending",
    };

    res.json(order);
});

app.post("/api/orders", (req, res) => {
    const { customer_id, total } = req.body;
    logger.info({ customer_id, total }, "creating order");

    const order = {
        id: "ord-123",
        customer_id,
        total,
        status: "pending",
    };

    res.status(201).json(order);
});

const server = app.listen(8080, () => {
    logger.info("server starting on :8080");
});

const shutdown = (signal) => {
    logger.info(`Received ${signal}, shutting down`);
    server.close(() => {
        logger.info("server stopped");
        process.exit(0);
    });
    setTimeout(() => {
        logger.error("forced shutdown");
        process.exit(1);
    }, 30000);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));

// Dependencies: express, pino (+ transitive deps)
// node_modules size: ~5MB (minimal)
// Docker image: ~120MB (Node.js Alpine)
// Memory at rest: ~40MB
// Startup time: ~200ms
```

### Performance Comparison

```
+-------------------------------+------------------+------------------+
| Metric                        | Go               | Node.js          |
+-------------------------------+------------------+------------------+
| Requests/sec (simple JSON)    | 50,000-100,000   | 15,000-30,000    |
| P99 latency (simple JSON)     | 1-3ms            | 5-15ms           |
| Memory per service instance   | 8-15 MB          | 40-80 MB         |
| Docker image size             | 5-20 MB          | 100-350 MB       |
| Startup time                  | 10-50 ms         | 200-800 ms       |
| Binary/artifact size          | 5-15 MB          | 5-500 MB (deps)  |
| CPU under load                | All cores (auto) | 1 core (default) |
| Max concurrent connections    | 100K+ (goroutines)| 10K+ (event loop)|
| gRPC throughput               | ~80,000 req/s    | ~20,000 req/s    |
+-------------------------------+------------------+------------------+
```

### When to Choose Go vs Node.js for Microservices

**Choose Go when:**
- Building infrastructure or platform services
- High throughput / low latency is critical
- The service is CPU-intensive (data processing, cryptography)
- You need tiny Docker images for fast scaling
- Memory efficiency matters (running many services per node)
- You want minimal dependencies and a single binary artifact
- Team is building Kubernetes operators or cloud-native tooling

**Choose Node.js when:**
- The service is a BFF (Backend for Frontend) with complex JSON transformations
- Rapid prototyping and iteration speed is the priority
- The team is primarily frontend developers
- The service is I/O bound with simple logic (API proxy, aggregator)
- You are building real-time features (Socket.IO is mature)
- Sharing code between frontend and backend (isomorphic/universal JS)

**Choose either when:**
- Building standard CRUD REST APIs
- The service is I/O bound (database queries, HTTP calls)
- Team competency is the deciding factor

### Architecture Diagram: Production Microservices System

```
                         ┌─────────────────┐
                         │   Load Balancer  │
                         │   (Ingress)      │
                         └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │   API Gateway    │
                         │   (Go)           │
                         │   - Auth         │
                         │   - Rate Limit   │
                         │   - Routing      │
                         └──┬────┬────┬─────┘
                            │    │    │
              ┌─────────────┘    │    └──────────────┐
              │                  │                    │
     ┌────────▼──────┐  ┌───────▼───────┐   ┌───────▼───────┐
     │ Order Service │  │ User Service  │   │Product Service│
     │ (Go)          │  │ (Go)          │   │ (Go)          │
     │               │  │               │   │               │
     │ HTTP + gRPC   │  │ HTTP + gRPC   │   │ HTTP + gRPC   │
     └───┬──────┬────┘  └───┬───────────┘   └───┬───────────┘
         │      │            │                    │
         │      │     ┌──────▼──────┐      ┌─────▼──────┐
         │      │     │  PostgreSQL │      │  PostgreSQL │
         │      │     │  (users)    │      │  (products) │
         │      │     └─────────────┘      └────────────┘
         │      │
    ┌────▼──┐  │    ┌───────────────────────────────────┐
    │Postgres│  └───►│  NATS / Kafka (Message Queue)     │
    │(orders)│       │                                   │
    └────────┘       └──────┬─────────────┬──────────────┘
                            │             │
                   ┌────────▼──────┐ ┌────▼──────────────┐
                   │ Payment Svc   │ │ Notification Svc  │
                   │ (Go)          │ │ (Go)              │
                   └───────────────┘ └───────────────────┘

     ┌──────────────────────────────────────────────────────┐
     │                  Observability                        │
     │  Prometheus (metrics) → Grafana (dashboards)          │
     │  Jaeger/Tempo (traces)                                │
     │  Loki (logs)                                          │
     └──────────────────────────────────────────────────────┘
```

---

## 14. Key Takeaways

1. **Go dominates cloud-native infrastructure for good reasons.** Small binaries, fast startup, low memory, built-in concurrency, and a strong standard library make it the natural choice for microservices. Docker, Kubernetes, and most of the CNCF ecosystem are proof.

2. **Use `internal/` to enforce architectural boundaries.** Unlike Node.js where any file can import any other, Go's `internal/` directory is compiler-enforced. Put your domain, services, handlers, and repositories in `internal/` so other projects cannot couple to your implementation.

3. **Use gRPC for inter-service communication, REST for public APIs.** gRPC with Protocol Buffers gives you type safety, code generation, streaming, and 3-10x better performance than JSON. Keep REST/JSON for browser-facing endpoints.

4. **Kubernetes DNS is sufficient for most service discovery needs.** Do not reach for Consul or etcd unless you have specific requirements like multi-datacenter routing or health-check-aware discovery outside Kubernetes.

5. **database/sql's connection pool is production-ready but must be configured.** Always set `MaxOpenConns`, `MaxIdleConns`, and `ConnMaxLifetime`. The defaults (unlimited connections, no lifetime) are dangerous in production.

6. **Event-driven architecture decouples services.** Use NATS, Kafka, or RabbitMQ for asynchronous communication between services. Design your event publishers and consumers behind interfaces so you can swap implementations.

7. **Observability requires all three pillars: logs, traces, and metrics.** Use `slog` (standard library) for structured logging, OpenTelemetry for distributed tracing, and Prometheus for metrics. Go's built-in `pprof` gives you CPU and memory profiling in production with near-zero overhead.

8. **Configuration should come from the environment.** Follow 12-Factor App principles. Use environment variables as the primary source, with Viper if you need config file support. Never hardcode connection strings or secrets.

9. **Graceful shutdown is not optional in production.** Handle SIGTERM, stop accepting new connections, drain in-flight requests, close message queue consumers, then close database connections. Go's `http.Server.Shutdown` handles the HTTP drain automatically.

10. **Implement both liveness and readiness probes.** Liveness checks if the process is alive (keep it simple). Readiness checks if the service can handle traffic (check database, cache, dependencies). Mark the service as not-ready before shutting down.

11. **Multi-stage Docker builds with `scratch` are Go's killer feature.** An 8MB Docker image with zero CVEs, pulling in under a second, is a massive operational advantage over 120-350MB Node.js images.

12. **Go's resource efficiency translates to real cost savings.** Running 64Mi per pod instead of 256Mi means 4x more services per cluster node. At scale (hundreds of services, thousands of pods), this saves significant infrastructure cost.

---

## 15. Practice Exercises

### Exercise 1: Basic Microservice

Build a complete "Product Service" microservice with:
- CRUD endpoints for products (name, description, price, stock)
- Structured logging with `slog`
- Graceful shutdown
- Health check endpoints (`/healthz` and `/readyz`)
- Configuration from environment variables
- Zero third-party dependencies (standard library only)

### Exercise 2: gRPC Service

Define a Protocol Buffer schema for an `InventoryService` with:
- `CheckStock(product_id) -> quantity`
- `ReserveStock(product_id, quantity) -> success/failure`
- `WatchStock(product_id) -> stream of stock updates` (server streaming)

Generate the Go code, implement the server, and write a client that calls all three RPCs.

### Exercise 3: Database Integration

Extend Exercise 1 with a PostgreSQL database:
- Use `database/sql` with proper connection pool configuration
- Implement the repository pattern (define an interface, implement it for PostgreSQL and for in-memory)
- Write database migrations
- Handle transactions for operations that modify multiple tables
- Write tests using the in-memory repository implementation

### Exercise 4: Event-Driven Communication

Build two services that communicate via NATS JetStream:
- **Order Service:** Creates orders and publishes `order.created` events
- **Inventory Service:** Consumes `order.created` events and reserves stock

Ensure:
- Messages are not lost if the Inventory Service is temporarily down
- Failed message processing is retried
- Duplicate processing is handled (idempotency)

### Exercise 5: Full Observability

Add complete observability to any service from the previous exercises:
- Structured JSON logging with request IDs
- Prometheus metrics: request count, request duration histogram, active connections gauge, business metrics (orders created, revenue)
- OpenTelemetry tracing with spans for HTTP handlers, database queries, and external service calls
- A `/metrics` endpoint for Prometheus scraping

### Exercise 6: Docker and Deployment

Create a multi-stage Dockerfile for your service:
- Build stage using `golang:1.23-alpine`
- Final stage using `scratch`
- Verify the final image is under 20MB
- Write a `docker-compose.yml` that runs your service, PostgreSQL, NATS, Prometheus, and Grafana
- Write a Kubernetes deployment manifest with proper resource limits, health probes, and environment variable configuration

### Exercise 7: API Gateway

Build a simple API gateway that:
- Routes `/api/orders/*` to the Order Service
- Routes `/api/products/*` to the Product Service
- Implements per-IP rate limiting (100 requests/minute)
- Adds a `X-Request-ID` header to every request
- Logs every request with method, path, status code, and duration
- Validates JWT tokens and passes user info to downstream services

### Exercise 8: Go vs Node.js Benchmark

Build the same simple JSON API in both Go (standard library) and Node.js (Express):
- `GET /api/items` -- returns a list of 100 items
- `POST /api/items` -- creates an item

Benchmark both with a load testing tool (like `hey` or `wrk`):
- Compare requests per second
- Compare P50, P95, P99 latency
- Compare memory usage under load
- Compare Docker image sizes
- Document your findings

### Exercise 9: Complete Microservices System

Build a minimal e-commerce system with three services:
- **Product Service** -- manages product catalog
- **Order Service** -- creates and manages orders, calls Product Service via gRPC to check stock
- **Notification Service** -- consumes order events and logs notifications

Requirements:
- gRPC for synchronous inter-service calls
- NATS for asynchronous events
- Each service has its own database
- Graceful shutdown in all services
- Health checks in all services
- Structured logging with correlated request IDs across services
- Docker Compose file to run the entire system
- Makefile for common development tasks
