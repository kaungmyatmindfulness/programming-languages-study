# Chapter 44: gRPC and Protocol Buffers in Go

## Table of Contents

1. [Introduction](#introduction)
2. [What Are Protocol Buffers?](#what-are-protocol-buffers)
3. [Protocol Buffers vs JSON](#protocol-buffers-vs-json)
4. [Defining .proto Files](#defining-proto-files)
5. [Code Generation with protoc](#code-generation-with-protoc)
6. [What Is gRPC?](#what-is-grpc)
7. [Building a gRPC Server](#building-a-grpc-server)
8. [Building a gRPC Client](#building-a-grpc-client)
9. [Streaming RPCs](#streaming-rpcs)
10. [Interceptors](#interceptors)
11. [Error Handling with gRPC Status Codes](#error-handling-with-grpc-status-codes)
12. [Deadlines and Timeouts](#deadlines-and-timeouts)
13. [Metadata: Headers and Trailers](#metadata-headers-and-trailers)
14. [gRPC-Gateway: REST + gRPC](#grpc-gateway-rest--grpc)
15. [Health Checking and Reflection](#health-checking-and-reflection)
16. [Testing gRPC Services](#testing-grpc-services)
17. [Performance Considerations](#performance-considerations)
18. [Best Practices](#best-practices)
19. [Summary](#summary)

---

## Introduction

Modern distributed systems need fast, type-safe, and language-agnostic communication between services. gRPC, built on top of Protocol Buffers and HTTP/2, has become the standard for service-to-service communication in microservice architectures. This chapter teaches you how to define service contracts with Protocol Buffers, generate Go code, build production-grade gRPC servers and clients, handle errors, implement middleware, and test everything thoroughly.

By the end of this chapter you will be able to:

- Define service contracts with `.proto` files
- Generate Go code from those contracts
- Build gRPC servers supporting all four RPC patterns
- Write clients that consume those services
- Add cross-cutting concerns (logging, auth) via interceptors
- Handle errors idiomatically with gRPC status codes
- Manage deadlines, timeouts, and metadata
- Expose gRPC services as REST APIs via gRPC-Gateway
- Test gRPC services without a real network using `bufconn`

---

## What Are Protocol Buffers?

Protocol Buffers (protobuf) is a language-neutral, platform-neutral, extensible mechanism for serializing structured data. Developed at Google, it is smaller, faster, and more strongly typed than JSON or XML.

### Core Concepts

Protocol Buffers work in three steps:

1. **Define** your data structures and services in a `.proto` file.
2. **Compile** the `.proto` file using the `protoc` compiler to generate source code in your target language.
3. **Use** the generated code to serialize, deserialize, and communicate.

```
┌─────────────┐      protoc       ┌──────────────────┐
│  .proto file │ ───────────────> │  Generated Go Code │
│  (contract)  │                  │  (structs, client, │
└─────────────┘                  │   server stubs)    │
                                  └──────────────────┘
```

### Why Protocol Buffers?

| Feature              | Description                                                |
|----------------------|------------------------------------------------------------|
| **Strongly Typed**   | Every field has a concrete type. No runtime guessing.      |
| **Compact**          | Binary encoding is 3-10x smaller than JSON.                |
| **Fast**             | Serialization/deserialization is 20-100x faster than JSON. |
| **Schema Evolution** | Add or remove fields without breaking existing clients.    |
| **Code Generation**  | Generates idiomatic code for 12+ languages.                |
| **Self-Documenting** | The `.proto` file IS the contract and documentation.       |

---

## Protocol Buffers vs JSON

Understanding when to choose protobuf over JSON (and vice versa) is critical for architectural decisions.

### Encoding Comparison

```
JSON representation (83 bytes as UTF-8):
{
  "id": 12345,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "active": true
}

Protobuf binary representation (~45 bytes):
08 B9 60 12 0D 41 6C 69 63 65 20 4A 6F 68 6E 73
6F 6E 1A 11 61 6C 69 63 65 40 65 78 61 6D 70 6C
65 2E 63 6F 6D 20 01
```

### Feature Comparison Table

| Aspect                 | Protocol Buffers                    | JSON                              |
|------------------------|-------------------------------------|-----------------------------------|
| **Format**             | Binary                              | Text (UTF-8)                      |
| **Size**               | 3-10x smaller                       | Larger (verbose keys)             |
| **Speed**              | 20-100x faster serialization        | Slower (text parsing)             |
| **Schema**             | Required (.proto file)              | Optional (JSON Schema)            |
| **Type Safety**        | Compile-time                        | Runtime                           |
| **Human Readable**     | No (binary)                         | Yes                               |
| **Browser Support**    | Requires gRPC-Web or gateway        | Native                            |
| **Debugging**          | Harder (need tools)                 | Easy (readable in logs)           |
| **Schema Evolution**   | Built-in (field numbers)            | Manual (versioning)               |
| **Language Support**   | Code generation for 12+ languages   | Universal                         |
| **Streaming**          | Native (HTTP/2 streams)             | Requires WebSocket/SSE            |
| **Bidirectional**      | Native                              | Requires WebSocket                |

### When to Use What

**Choose Protocol Buffers / gRPC when:**
- Service-to-service communication in a backend
- Low latency and high throughput are critical
- You need strong contracts between teams
- You want streaming (real-time data feeds, chat, etc.)
- You have polyglot services (Go, Java, Python, etc.)

**Choose JSON / REST when:**
- Browser clients consume the API directly
- You need human-readable payloads for debugging
- The API is public-facing and needs maximum compatibility
- Rapid prototyping where schema overhead is unwanted
- Third-party integrations expect REST

> **Tip:** You do not have to choose one exclusively. gRPC-Gateway (covered later) lets you serve both gRPC and REST from the same Go server.

---

## Defining .proto Files

The `.proto` file is the single source of truth for your API contract. Everything -- messages, services, enums -- is defined here.

### Syntax and Package Declaration

Always use `syntax = "proto3";` for new projects. Proto2 is legacy.

```protobuf
// user_service.proto
syntax = "proto3";

package userservice.v1;

option go_package = "github.com/yourorg/project/gen/userservice/v1;userservicev1";
```

The `go_package` option tells the Go code generator where to place the output. The part after the semicolon is the Go package name.

### Scalar Types

Proto3 scalar types map directly to Go types:

| Proto Type   | Go Type    | Default Value | Notes                              |
|-------------|------------|---------------|------------------------------------|
| `double`    | `float64`  | 0             |                                    |
| `float`     | `float32`  | 0             |                                    |
| `int32`     | `int32`    | 0             | Variable-length encoding           |
| `int64`     | `int64`    | 0             | Variable-length encoding           |
| `uint32`    | `uint32`   | 0             |                                    |
| `uint64`    | `uint64`   | 0             |                                    |
| `sint32`    | `int32`    | 0             | Efficient for negative numbers     |
| `sint64`    | `int64`    | 0             | Efficient for negative numbers     |
| `fixed32`   | `uint32`   | 0             | Always 4 bytes; efficient if > 2^28|
| `fixed64`   | `uint64`   | 0             | Always 8 bytes; efficient if > 2^56|
| `sfixed32`  | `int32`    | 0             | Always 4 bytes                     |
| `sfixed64`  | `int64`    | 0             | Always 8 bytes                     |
| `bool`      | `bool`     | false         |                                    |
| `string`    | `string`   | ""            | Must be valid UTF-8                |
| `bytes`     | `[]byte`   | nil           | Arbitrary byte sequences           |

### Messages

Messages are the primary data structure. They compile to Go structs.

```protobuf
// A user in the system.
message User {
  // Unique identifier for the user.
  int64 id = 1;

  // Display name.
  string name = 2;

  // Email address. Must be unique.
  string email = 3;

  // When the user was created.
  google.protobuf.Timestamp created_at = 4;

  // The user's role in the system.
  Role role = 5;

  // Mailing address (optional; may be nil).
  Address address = 6;

  // Tags associated with the user.
  repeated string tags = 7;

  // Arbitrary key-value metadata.
  map<string, string> metadata = 8;
}
```

> **Important:** Field numbers (1, 2, 3...) are part of the binary encoding. Once assigned, they must NEVER be changed or reused. Numbers 1-15 use one byte on the wire; 16-2047 use two bytes. Reserve 1-15 for frequently-set fields.

### Nested Messages

```protobuf
message Address {
  string street = 1;
  string city = 2;
  string state = 3;
  string zip_code = 4;
  string country = 5;
}

message Order {
  int64 id = 1;
  int64 user_id = 2;

  // Nested message definition
  message LineItem {
    string product_id = 1;
    int32 quantity = 2;
    int64 price_cents = 3;
  }

  repeated LineItem items = 3;

  Address shipping_address = 4;
  Address billing_address = 5;
}
```

### Enums

Enums define a set of named constants. The first value MUST be 0 and serves as the default.

```protobuf
enum Role {
  ROLE_UNSPECIFIED = 0;  // Always have an UNSPECIFIED as 0
  ROLE_ADMIN = 1;
  ROLE_EDITOR = 2;
  ROLE_VIEWER = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}
```

> **Warning:** Always prefix enum values with the enum name (e.g., `ROLE_ADMIN`, not `ADMIN`). Enum values share a namespace in proto3, so `ADMIN` in two different enums would conflict.

### Oneof Fields

`oneof` enforces that at most one of a group of fields is set. Useful for polymorphic data.

```protobuf
message NotificationPreference {
  string user_id = 1;

  oneof channel {
    EmailPreference email = 2;
    SMSPreference sms = 3;
    PushPreference push = 4;
  }
}

message EmailPreference {
  string address = 1;
  bool html_enabled = 2;
}

message SMSPreference {
  string phone_number = 1;
}

message PushPreference {
  string device_token = 1;
  string platform = 2;  // "ios" or "android"
}
```

In Go, `oneof` generates an interface and concrete types:

```go
// Generated code (simplified)
type NotificationPreference struct {
    UserId  string
    Channel isNotificationPreference_Channel // interface
}

type NotificationPreference_Email struct {
    Email *EmailPreference
}

type NotificationPreference_Sms struct {
    Sms *SMSPreference
}

type NotificationPreference_Push struct {
    Push *PushPreference
}

// Usage:
pref := &pb.NotificationPreference{
    UserId: "user-123",
    Channel: &pb.NotificationPreference_Email{
        Email: &pb.EmailPreference{
            Address:     "alice@example.com",
            HtmlEnabled: true,
        },
    },
}

// Type switch to read:
switch ch := pref.Channel.(type) {
case *pb.NotificationPreference_Email:
    fmt.Println("Email:", ch.Email.Address)
case *pb.NotificationPreference_Sms:
    fmt.Println("SMS:", ch.Sms.PhoneNumber)
case *pb.NotificationPreference_Push:
    fmt.Println("Push:", ch.Push.DeviceToken)
}
```

### Repeated Fields (Lists)

`repeated` creates a slice in Go.

```protobuf
message SearchResponse {
  repeated User users = 1;       // becomes []*User in Go
  int32 total_count = 2;
  string next_page_token = 3;
}
```

### Map Fields

```protobuf
message Config {
  map<string, string> settings = 1;    // becomes map[string]string in Go
  map<string, int32> feature_flags = 2; // becomes map[string]int32 in Go
}
```

> **Warning:** Map fields cannot be `repeated`. The iteration order is not guaranteed. Use a repeated message with key/value fields if you need ordering.

### Well-Known Types

Google provides commonly-needed message types. Import them like this:

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/any.proto";
import "google/protobuf/field_mask.proto";
import "google/protobuf/struct.proto";

message Event {
  string id = 1;
  google.protobuf.Timestamp occurred_at = 2;
  google.protobuf.Duration ttl = 3;
  google.protobuf.StringValue optional_note = 4;  // nil-able string
  google.protobuf.FieldMask update_mask = 5;       // partial updates
  google.protobuf.Any payload = 6;                 // arbitrary message
}
```

| Well-Known Type   | Go Type                          | Use Case                                |
|-------------------|----------------------------------|-----------------------------------------|
| `Timestamp`       | `*timestamppb.Timestamp`         | Points in time (UTC)                    |
| `Duration`        | `*durationpb.Duration`           | Time spans                              |
| `Empty`           | `*emptypb.Empty`                 | RPCs with no request/response body      |
| `StringValue`     | `*wrapperspb.StringValue`        | Nullable string (distinguish "" from nil)|
| `FieldMask`       | `*fieldmaskpb.FieldMask`         | Partial updates (PATCH semantics)       |
| `Any`             | `*anypb.Any`                     | Polymorphic payloads                    |
| `Struct`          | `*structpb.Struct`               | Dynamic JSON-like data                  |

### Service Definitions

Services define the RPC methods. This is where gRPC comes in.

```protobuf
service UserService {
  // Unary RPC: one request, one response.
  rpc GetUser(GetUserRequest) returns (GetUserResponse);

  // Unary RPC: create a user.
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);

  // Unary RPC: update a user (partial update with field mask).
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);

  // Unary RPC: delete a user.
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

  // Server streaming: stream a list of users matching a filter.
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client streaming: upload many user records at once.
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateUsersResponse);

  // Bidirectional streaming: real-time user sync.
  rpc SyncUsers(stream SyncRequest) returns (stream SyncResponse);
}
```

### Request and Response Messages

Follow naming conventions: `<MethodName>Request` and `<MethodName>Response`.

```protobuf
message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  Role role = 3;
  repeated string tags = 4;
}

message CreateUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  int64 id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;  // e.g., "role=ADMIN"
}

message BulkCreateUsersResponse {
  int32 created_count = 1;
  int32 failed_count = 2;
  repeated string errors = 3;
}

message SyncRequest {
  User user = 1;
  SyncAction action = 2;
}

message SyncResponse {
  User user = 1;
  SyncAction action = 2;
  bool success = 3;
}

enum SyncAction {
  SYNC_ACTION_UNSPECIFIED = 0;
  SYNC_ACTION_CREATE = 1;
  SYNC_ACTION_UPDATE = 2;
  SYNC_ACTION_DELETE = 3;
}
```

### Reserved Fields and Numbers

When you remove a field, reserve its number and name to prevent accidental reuse.

```protobuf
message User {
  reserved 6, 9 to 11;
  reserved "middle_name", "nickname";

  int64 id = 1;
  string name = 2;
  string email = 3;
  // field 6 (middle_name) was removed in v2
  // fields 9-11 were removed in v3
}
```

### Full Example .proto File

Here is a complete, production-style `.proto` file bringing everything together:

```protobuf
// proto/userservice/v1/user_service.proto
syntax = "proto3";

package userservice.v1;

option go_package = "github.com/yourorg/project/gen/userservice/v1;userservicev1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";

// ─── Enums ───────────────────────────────────────────────

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_ADMIN = 1;
  ROLE_EDITOR = 2;
  ROLE_VIEWER = 3;
}

// ─── Messages ────────────────────────────────────────────

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  Role role = 4;
  repeated string tags = 5;
  map<string, string> metadata = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  Role role = 3;
  repeated string tags = 4;
  map<string, string> metadata = 5;
}

message CreateUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  int64 id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

// ─── Service ─────────────────────────────────────────────

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

---

## Code Generation with protoc

### Installing the Tools

You need three things:

1. **`protoc`** -- the Protocol Buffer compiler
2. **`protoc-gen-go`** -- generates Go message types
3. **`protoc-gen-go-grpc`** -- generates Go gRPC service stubs

```bash
# Install protoc (macOS)
brew install protobuf

# Install protoc (Linux -- download from GitHub releases)
# https://github.com/protocolbuffers/protobuf/releases

# Install Go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Ensure $GOPATH/bin is in your PATH
export PATH="$PATH:$(go env GOPATH)/bin"
```

Verify the installation:

```bash
protoc --version
# libprotoc 28.x

which protoc-gen-go
# /Users/you/go/bin/protoc-gen-go

which protoc-gen-go-grpc
# /Users/you/go/bin/protoc-gen-go-grpc
```

### Project Layout

A recommended project structure:

```
project/
├── cmd/
│   ├── server/
│   │   └── main.go
│   └── client/
│       └── main.go
├── gen/                          # Generated code (committed or gitignored)
│   └── userservice/
│       └── v1/
│           ├── user_service.pb.go
│           └── user_service_grpc.pb.go
├── internal/
│   └── server/
│       └── user_service.go       # Server implementation
├── proto/
│   └── userservice/
│       └── v1/
│           └── user_service.proto
├── buf.yaml                      # Optional: Buf configuration
├── buf.gen.yaml                  # Optional: Buf generation config
├── go.mod
└── go.sum
```

### Running protoc Directly

```bash
protoc \
  --proto_path=proto \
  --go_out=gen \
  --go_opt=paths=source_relative \
  --go-grpc_out=gen \
  --go-grpc_opt=paths=source_relative \
  proto/userservice/v1/user_service.proto
```

This produces two files:
- `gen/userservice/v1/user_service.pb.go` -- message types, serialization
- `gen/userservice/v1/user_service_grpc.pb.go` -- gRPC client and server interfaces

### Using Buf (Recommended)

[Buf](https://buf.build) is a modern replacement for raw `protoc` invocations. It handles dependency management, linting, and breaking change detection.

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
```

```bash
# Generate code
buf generate

# Lint proto files
buf lint

# Check for breaking changes against the main branch
buf breaking --against '.git#branch=main'
```

### Using a Makefile

```makefile
# Makefile
.PHONY: proto proto-lint proto-breaking clean

PROTO_DIR := proto
GEN_DIR := gen

proto:
	@echo "Generating protobuf code..."
	buf generate

proto-lint:
	buf lint

proto-breaking:
	buf breaking --against '.git#branch=main'

clean:
	rm -rf $(GEN_DIR)
```

### Understanding the Generated Code

The message types file (`*.pb.go`) generates Go structs:

```go
// Generated: user_service.pb.go (simplified)
type User struct {
    state         protoimpl.MessageState
    sizeCache     protoimpl.SizeCache
    unknownFields protoimpl.UnknownFields

    Id        int64                  `protobuf:"varint,1,opt,name=id,proto3" json:"id,omitempty"`
    Name      string                 `protobuf:"bytes,2,opt,name=name,proto3" json:"name,omitempty"`
    Email     string                 `protobuf:"bytes,3,opt,name=email,proto3" json:"email,omitempty"`
    Role      Role                   `protobuf:"varint,4,opt,name=role,proto3,enum=userservice.v1.Role" json:"role,omitempty"`
    Tags      []string               `protobuf:"bytes,5,rep,name=tags,proto3" json:"tags,omitempty"`
    Metadata  map[string]string      `protobuf:"bytes,6,rep,name=metadata,proto3" json:"metadata,omitempty"`
    CreatedAt *timestamppb.Timestamp `protobuf:"bytes,7,opt,name=created_at,json=createdAt,proto3" json:"created_at,omitempty"`
    UpdatedAt *timestamppb.Timestamp `protobuf:"bytes,8,opt,name=updated_at,json=updatedAt,proto3" json:"updated_at,omitempty"`
}
```

The gRPC file (`*_grpc.pb.go`) generates:

```go
// Generated: user_service_grpc.pb.go (simplified)

// Server interface -- you implement this
type UserServiceServer interface {
    GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
    CreateUser(context.Context, *CreateUserRequest) (*CreateUserResponse, error)
    UpdateUser(context.Context, *UpdateUserRequest) (*UpdateUserResponse, error)
    DeleteUser(context.Context, *DeleteUserRequest) (*emptypb.Empty, error)
    ListUsers(*ListUsersRequest, UserService_ListUsersServer) error
    mustEmbedUnimplementedUserServiceServer()
}

// Client interface -- generated for you
type UserServiceClient interface {
    GetUser(ctx context.Context, in *GetUserRequest, opts ...grpc.CallOption) (*GetUserResponse, error)
    CreateUser(ctx context.Context, in *CreateUserRequest, opts ...grpc.CallOption) (*CreateUserResponse, error)
    UpdateUser(ctx context.Context, in *UpdateUserRequest, opts ...grpc.CallOption) (*UpdateUserResponse, error)
    DeleteUser(ctx context.Context, in *DeleteUserRequest, opts ...grpc.CallOption) (*emptypb.Empty, error)
    ListUsers(ctx context.Context, in *ListUsersRequest, opts ...grpc.CallOption) (UserService_ListUsersClient, error)
}
```

---

## What Is gRPC?

gRPC (gRPC Remote Procedure Calls) is a high-performance RPC framework built on:

- **HTTP/2** -- multiplexing, header compression, bidirectional streaming
- **Protocol Buffers** -- efficient binary serialization
- **Strongly-typed contracts** -- `.proto` files as the source of truth

### The Four RPC Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                     gRPC RPC Patterns                       │
├──────────────────────┬──────────────────────────────────────┤
│ Unary                │  Client ──req──> Server              │
│                      │  Client <──res── Server              │
├──────────────────────┼──────────────────────────────────────┤
│ Server Streaming     │  Client ──req──> Server              │
│                      │  Client <──res── Server              │
│                      │  Client <──res── Server              │
│                      │  Client <──res── Server              │
├──────────────────────┼──────────────────────────────────────┤
│ Client Streaming     │  Client ──req──> Server              │
│                      │  Client ──req──> Server              │
│                      │  Client ──req──> Server              │
│                      │  Client <──res── Server              │
├──────────────────────┼──────────────────────────────────────┤
│ Bidirectional        │  Client ──req──> Server              │
│ Streaming            │  Client <──res── Server              │
│                      │  Client ──req──> Server              │
│                      │  Client <──res── Server              │
│                      │  (independent streams, any order)    │
└──────────────────────┴──────────────────────────────────────┘
```

### Go Dependencies

```bash
go mod init github.com/yourorg/project

go get google.golang.org/grpc
go get google.golang.org/protobuf
go get google.golang.org/grpc/credentials/insecure
```

---

## Building a gRPC Server

### Step 1: Implement the Service Interface

The generated code includes an `UnimplementedUserServiceServer` that you embed in your implementation. This ensures forward compatibility -- if new methods are added to the proto service, your server compiles without implementing them immediately (they return "unimplemented" errors).

```go
// internal/server/user_service.go
package server

import (
    "context"
    "fmt"
    "sync"
    "sync/atomic"
    "time"

    pb "github.com/yourorg/project/gen/userservice/v1"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/emptypb"
    "google.golang.org/protobuf/types/known/timestamppb"
)

// UserServiceServer implements the generated UserServiceServer interface.
type UserServiceServer struct {
    // Embedding this struct ensures forward compatibility.
    // New methods added to the proto will have default "unimplemented" behavior.
    pb.UnimplementedUserServiceServer

    mu    sync.RWMutex
    users map[int64]*pb.User
    nextID atomic.Int64
}

// NewUserServiceServer creates a new server with an empty user store.
func NewUserServiceServer() *UserServiceServer {
    s := &UserServiceServer{
        users: make(map[int64]*pb.User),
    }
    s.nextID.Store(1)
    return s
}
```

### Step 2: Implement Unary RPCs

```go
// GetUser retrieves a user by ID.
func (s *UserServiceServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {
    // Validate the request.
    if req.GetId() <= 0 {
        return nil, status.Errorf(codes.InvalidArgument, "id must be positive, got %d", req.GetId())
    }

    s.mu.RLock()
    defer s.mu.RUnlock()

    user, ok := s.users[req.GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
    }

    return &pb.GetUserResponse{User: user}, nil
}

// CreateUser creates a new user.
func (s *UserServiceServer) CreateUser(
    ctx context.Context,
    req *pb.CreateUserRequest,
) (*pb.CreateUserResponse, error) {
    // Validate required fields.
    if req.GetName() == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    if req.GetEmail() == "" {
        return nil, status.Error(codes.InvalidArgument, "email is required")
    }

    // Check for context cancellation before proceeding.
    if err := ctx.Err(); err != nil {
        return nil, status.FromContextError(err).Err()
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    // Check for duplicate email.
    for _, u := range s.users {
        if u.Email == req.GetEmail() {
            return nil, status.Errorf(codes.AlreadyExists, "email %q already in use", req.GetEmail())
        }
    }

    now := timestamppb.New(time.Now())
    user := &pb.User{
        Id:        s.nextID.Add(1) - 1,
        Name:      req.GetName(),
        Email:     req.GetEmail(),
        Role:      req.GetRole(),
        Tags:      req.GetTags(),
        Metadata:  req.GetMetadata(),
        CreatedAt: now,
        UpdatedAt: now,
    }

    s.users[user.Id] = user

    return &pb.CreateUserResponse{User: user}, nil
}

// UpdateUser performs a partial update using a field mask.
func (s *UserServiceServer) UpdateUser(
    ctx context.Context,
    req *pb.UpdateUserRequest,
) (*pb.UpdateUserResponse, error) {
    if req.GetUser() == nil {
        return nil, status.Error(codes.InvalidArgument, "user is required")
    }
    if req.GetUser().GetId() <= 0 {
        return nil, status.Error(codes.InvalidArgument, "user.id is required")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    existing, ok := s.users[req.GetUser().GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetUser().GetId())
    }

    // Apply field mask. If no mask, update all fields.
    paths := req.GetUpdateMask().GetPaths()
    if len(paths) == 0 {
        paths = []string{"name", "email", "role", "tags", "metadata"}
    }

    for _, path := range paths {
        switch path {
        case "name":
            existing.Name = req.GetUser().GetName()
        case "email":
            existing.Email = req.GetUser().GetEmail()
        case "role":
            existing.Role = req.GetUser().GetRole()
        case "tags":
            existing.Tags = req.GetUser().GetTags()
        case "metadata":
            existing.Metadata = req.GetUser().GetMetadata()
        default:
            return nil, status.Errorf(codes.InvalidArgument, "unknown field path: %q", path)
        }
    }

    existing.UpdatedAt = timestamppb.New(time.Now())

    return &pb.UpdateUserResponse{User: existing}, nil
}

// DeleteUser removes a user by ID.
func (s *UserServiceServer) DeleteUser(
    ctx context.Context,
    req *pb.DeleteUserRequest,
) (*emptypb.Empty, error) {
    if req.GetId() <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.users[req.GetId()]; !ok {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
    }

    delete(s.users, req.GetId())
    return &emptypb.Empty{}, nil
}
```

### Step 3: Start the gRPC Server

```go
// cmd/server/main.go
package main

import (
    "flag"
    "fmt"
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"

    pb "github.com/yourorg/project/gen/userservice/v1"
    "github.com/yourorg/project/internal/server"
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

func main() {
    port := flag.Int("port", 50051, "The server port")
    flag.Parse()

    // Create a TCP listener.
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    // Create a gRPC server with options.
    grpcServer := grpc.NewServer(
        grpc.MaxRecvMsgSize(4 * 1024 * 1024), // 4MB max receive
        grpc.MaxSendMsgSize(4 * 1024 * 1024), // 4MB max send
    )

    // Register our service implementation.
    userService := server.NewUserServiceServer()
    pb.RegisterUserServiceServer(grpcServer, userService)

    // Register reflection for tools like grpcurl and grpcui.
    reflection.Register(grpcServer)

    // Graceful shutdown on SIGINT/SIGTERM.
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        sig := <-sigCh
        log.Printf("Received signal %v, shutting down gracefully...", sig)
        grpcServer.GracefulStop()
    }()

    log.Printf("gRPC server listening on :%d", *port)
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

---

## Building a gRPC Client

### Basic Client Connection

```go
// cmd/client/main.go
package main

import (
    "context"
    "flag"
    "log"
    "time"

    pb "github.com/yourorg/project/gen/userservice/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    addr := flag.String("addr", "localhost:50051", "the server address")
    flag.Parse()

    // Establish a connection.
    // In production, use TLS credentials instead of insecure.
    conn, err := grpc.NewClient(*addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()

    // Create a typed client from the generated code.
    client := pb.NewUserServiceClient(conn)

    // Use a context with timeout for each RPC call.
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // ── Create a user ────────────────────────────────────
    createResp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Alice Johnson",
        Email: "alice@example.com",
        Role:  pb.Role_ROLE_ADMIN,
        Tags:  []string{"engineering", "team-lead"},
        Metadata: map[string]string{
            "department": "platform",
        },
    })
    if err != nil {
        log.Fatalf("CreateUser failed: %v", err)
    }
    log.Printf("Created user: %v", createResp.GetUser())

    // ── Get the user ─────────────────────────────────────
    getResp, err := client.GetUser(ctx, &pb.GetUserRequest{
        Id: createResp.GetUser().GetId(),
    })
    if err != nil {
        log.Fatalf("GetUser failed: %v", err)
    }
    log.Printf("Got user: %v", getResp.GetUser())

    // ── Update the user ──────────────────────────────────
    updateResp, err := client.UpdateUser(ctx, &pb.UpdateUserRequest{
        User: &pb.User{
            Id:   createResp.GetUser().GetId(),
            Name: "Alice J.",
        },
        UpdateMask: &fieldmaskpb.FieldMask{
            Paths: []string{"name"},
        },
    })
    if err != nil {
        log.Fatalf("UpdateUser failed: %v", err)
    }
    log.Printf("Updated user: %v", updateResp.GetUser())

    // ── Delete the user ──────────────────────────────────
    _, err = client.DeleteUser(ctx, &pb.DeleteUserRequest{
        Id: createResp.GetUser().GetId(),
    })
    if err != nil {
        log.Fatalf("DeleteUser failed: %v", err)
    }
    log.Println("User deleted successfully")
}
```

### Connection Best Practices

```go
// Production-grade connection setup
func newGRPCConn(ctx context.Context, target string) (*grpc.ClientConn, error) {
    opts := []grpc.DialOption{
        // Transport security
        grpc.WithTransportCredentials(insecure.NewCredentials()), // Dev only!
        // For production:
        // grpc.WithTransportCredentials(credentials.NewTLS(tlsConfig)),

        // Default call options applied to every RPC
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(4 * 1024 * 1024),
            grpc.MaxCallSendMsgSize(4 * 1024 * 1024),
            grpc.WaitForReady(true), // Wait for the connection to be ready
        ),

        // Keepalive configuration
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time:                10 * time.Second, // Ping if no activity for 10s
            Timeout:             3 * time.Second,  // Wait 3s for ping ack
            PermitWithoutStream: false,             // No pings without active RPCs
        }),
    }

    conn, err := grpc.NewClient(target, opts...)
    if err != nil {
        return nil, fmt.Errorf("grpc.NewClient(%q): %w", target, err)
    }

    return conn, nil
}
```

> **Tip:** `grpc.NewClient` does NOT block or establish a TCP connection. The actual connection happens lazily on the first RPC. This is by design -- gRPC handles reconnection and load balancing internally.

---

## Streaming RPCs

Streaming is one of gRPC's killer features. Let us implement all three streaming patterns.

### Proto Definitions for Streaming

```protobuf
service ChatService {
  // Server streaming: subscribe to messages in a room
  rpc SubscribeToRoom(SubscribeRequest) returns (stream ChatMessage);

  // Client streaming: upload a batch of messages
  rpc SendMessageBatch(stream ChatMessage) returns (BatchResponse);

  // Bidirectional streaming: real-time chat
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message SubscribeRequest {
  string room_id = 1;
  google.protobuf.Timestamp since = 2;
}

message ChatMessage {
  string id = 1;
  string room_id = 2;
  string sender_id = 3;
  string content = 4;
  google.protobuf.Timestamp sent_at = 5;
}

message BatchResponse {
  int32 received_count = 1;
}
```

### Server Streaming: Server Implementation

The server sends multiple responses for a single client request.

```go
// SubscribeToRoom streams messages to the client as they arrive.
func (s *ChatServer) SubscribeToRoom(
    req *pb.SubscribeRequest,
    stream pb.ChatService_SubscribeToRoomServer,
) error {
    if req.GetRoomId() == "" {
        return status.Error(codes.InvalidArgument, "room_id is required")
    }

    log.Printf("Client subscribed to room %s", req.GetRoomId())

    // Create a channel for this subscriber.
    msgCh := s.subscribe(req.GetRoomId())
    defer s.unsubscribe(req.GetRoomId(), msgCh)

    for {
        select {
        case msg := <-msgCh:
            // Send each message to the client.
            if err := stream.Send(msg); err != nil {
                // Client disconnected or error occurred.
                log.Printf("Error sending to subscriber: %v", err)
                return err
            }
        case <-stream.Context().Done():
            // Client cancelled or deadline exceeded.
            log.Printf("Client disconnected from room %s: %v",
                req.GetRoomId(), stream.Context().Err())
            return nil
        }
    }
}
```

### Server Streaming: Client Implementation

```go
func subscribeToRoom(ctx context.Context, client pb.ChatServiceClient, roomID string) error {
    // Open the stream.
    stream, err := client.SubscribeToRoom(ctx, &pb.SubscribeRequest{
        RoomId: roomID,
        Since:  timestamppb.Now(),
    })
    if err != nil {
        return fmt.Errorf("SubscribeToRoom: %w", err)
    }

    // Read messages until the stream closes or context is cancelled.
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            log.Println("Stream ended (server closed)")
            return nil
        }
        if err != nil {
            // Check if it was a cancellation.
            st, ok := status.FromError(err)
            if ok && st.Code() == codes.Canceled {
                log.Println("Stream cancelled by client")
                return nil
            }
            return fmt.Errorf("stream.Recv: %w", err)
        }

        fmt.Printf("[%s] %s: %s\n",
            msg.GetRoomId(),
            msg.GetSenderId(),
            msg.GetContent(),
        )
    }
}
```

### Client Streaming: Server Implementation

The server receives multiple messages from the client and returns a single response.

```go
// SendMessageBatch receives a stream of messages from the client.
func (s *ChatServer) SendMessageBatch(
    stream pb.ChatService_SendMessageBatchServer,
) error {
    var count int32

    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            // Client finished sending. Return the response.
            log.Printf("Batch complete: received %d messages", count)
            return stream.SendAndClose(&pb.BatchResponse{
                ReceivedCount: count,
            })
        }
        if err != nil {
            return status.Errorf(codes.Internal, "recv error: %v", err)
        }

        // Process each message.
        if err := s.processMessage(msg); err != nil {
            return status.Errorf(codes.Internal, "processing message %s: %v", msg.GetId(), err)
        }
        count++

        // Periodically check context.
        if err := stream.Context().Err(); err != nil {
            return status.FromContextError(err).Err()
        }
    }
}
```

### Client Streaming: Client Implementation

```go
func sendMessageBatch(ctx context.Context, client pb.ChatServiceClient, messages []*pb.ChatMessage) error {
    // Open the stream.
    stream, err := client.SendMessageBatch(ctx)
    if err != nil {
        return fmt.Errorf("SendMessageBatch: %w", err)
    }

    // Send all messages.
    for _, msg := range messages {
        if err := stream.Send(msg); err != nil {
            return fmt.Errorf("stream.Send: %w", err)
        }
    }

    // Close the send side and receive the response.
    resp, err := stream.CloseAndRecv()
    if err != nil {
        return fmt.Errorf("CloseAndRecv: %w", err)
    }

    log.Printf("Server received %d messages", resp.GetReceivedCount())
    return nil
}
```

### Bidirectional Streaming: Server Implementation

Both client and server send messages independently on their own stream.

```go
// Chat handles bidirectional streaming for real-time chat.
func (s *ChatServer) Chat(stream pb.ChatService_ChatServer) error {
    // Get the user ID from metadata (set by client interceptor).
    md, ok := metadata.FromIncomingContext(stream.Context())
    if !ok {
        return status.Error(codes.Unauthenticated, "missing metadata")
    }
    userIDs := md.Get("user-id")
    if len(userIDs) == 0 {
        return status.Error(codes.Unauthenticated, "missing user-id in metadata")
    }
    userID := userIDs[0]

    log.Printf("User %s connected to chat", userID)

    // Register this stream for broadcasting.
    s.mu.Lock()
    s.chatStreams[userID] = stream
    s.mu.Unlock()

    defer func() {
        s.mu.Lock()
        delete(s.chatStreams, userID)
        s.mu.Unlock()
        log.Printf("User %s disconnected from chat", userID)
    }()

    // Read messages from this client and broadcast to others.
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        // Set the sender.
        msg.SenderId = userID
        msg.SentAt = timestamppb.Now()

        // Broadcast to all connected clients (except sender).
        s.mu.RLock()
        for uid, s := range s.chatStreams {
            if uid != userID {
                // Non-blocking send: if a client is slow, skip them.
                if err := s.Send(msg); err != nil {
                    log.Printf("Error sending to %s: %v", uid, err)
                }
            }
        }
        s.mu.RUnlock()
    }
}
```

### Bidirectional Streaming: Client Implementation

```go
func chat(ctx context.Context, client pb.ChatServiceClient) error {
    // Open the bidirectional stream.
    stream, err := client.Chat(ctx)
    if err != nil {
        return fmt.Errorf("Chat: %w", err)
    }

    // Use an errgroup to manage the send and receive goroutines.
    var wg sync.WaitGroup

    // Receive goroutine: print incoming messages.
    wg.Add(1)
    go func() {
        defer wg.Done()
        for {
            msg, err := stream.Recv()
            if err == io.EOF {
                log.Println("Server closed the stream")
                return
            }
            if err != nil {
                log.Printf("Recv error: %v", err)
                return
            }
            fmt.Printf("[%s] %s: %s\n",
                msg.GetRoomId(), msg.GetSenderId(), msg.GetContent())
        }
    }()

    // Send goroutine: read from stdin and send.
    wg.Add(1)
    go func() {
        defer wg.Done()
        scanner := bufio.NewScanner(os.Stdin)
        for scanner.Scan() {
            text := scanner.Text()
            if text == "/quit" {
                if err := stream.CloseSend(); err != nil {
                    log.Printf("CloseSend error: %v", err)
                }
                return
            }
            if err := stream.Send(&pb.ChatMessage{
                RoomId:  "general",
                Content: text,
            }); err != nil {
                log.Printf("Send error: %v", err)
                return
            }
        }
    }()

    wg.Wait()
    return nil
}
```

---

## Interceptors

Interceptors are gRPC's middleware mechanism. They allow you to add cross-cutting concerns like logging, authentication, metrics, and tracing without modifying service logic.

There are four types of interceptors:

| Type                       | Side   | Pattern               |
|----------------------------|--------|-----------------------|
| Unary Server Interceptor   | Server | Single request/response |
| Stream Server Interceptor  | Server | Streaming RPCs        |
| Unary Client Interceptor   | Client | Single request/response |
| Stream Client Interceptor  | Client | Streaming RPCs        |

### Unary Server Interceptor: Logging

```go
// loggingUnaryInterceptor logs every unary RPC call.
func loggingUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // Extract metadata for request ID if available.
    md, _ := metadata.FromIncomingContext(ctx)
    requestID := "unknown"
    if ids := md.Get("x-request-id"); len(ids) > 0 {
        requestID = ids[0]
    }

    // Call the actual handler.
    resp, err := handler(ctx, req)

    // Log the result.
    duration := time.Since(start)
    statusCode := codes.OK
    if err != nil {
        if st, ok := status.FromError(err); ok {
            statusCode = st.Code()
        }
    }

    log.Printf("RPC %s | status=%s | duration=%s | request_id=%s",
        info.FullMethod, statusCode, duration, requestID)

    return resp, err
}
```

### Unary Server Interceptor: Authentication

```go
// authUnaryInterceptor validates bearer tokens on incoming requests.
func authUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Skip auth for certain methods (e.g., health checks).
    if isPublicMethod(info.FullMethod) {
        return handler(ctx, req)
    }

    // Extract the authorization token from metadata.
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    authHeaders := md.Get("authorization")
    if len(authHeaders) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing authorization header")
    }

    token := authHeaders[0]
    if !strings.HasPrefix(token, "Bearer ") {
        return nil, status.Error(codes.Unauthenticated, "invalid authorization format; expected 'Bearer <token>'")
    }
    token = strings.TrimPrefix(token, "Bearer ")

    // Validate the token (e.g., JWT verification).
    claims, err := validateToken(token)
    if err != nil {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Inject the claims into the context for downstream handlers.
    ctx = context.WithValue(ctx, claimsKey{}, claims)

    return handler(ctx, req)
}

type claimsKey struct{}

func isPublicMethod(method string) bool {
    publicMethods := map[string]bool{
        "/grpc.health.v1.Health/Check": true,
        "/grpc.reflection.v1.ServerReflection/ServerReflectionInfo": true,
    }
    return publicMethods[method]
}
```

### Stream Server Interceptor: Logging

```go
// loggingStreamInterceptor logs streaming RPC calls.
func loggingStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()

    log.Printf("Stream started: %s (client_stream=%v, server_stream=%v)",
        info.FullMethod, info.IsClientStream, info.IsServerStream)

    // Wrap the stream to count messages.
    wrapped := &countingServerStream{ServerStream: ss}

    // Call the actual handler.
    err := handler(srv, wrapped)

    duration := time.Since(start)
    statusCode := codes.OK
    if err != nil {
        if st, ok := status.FromError(err); ok {
            statusCode = st.Code()
        }
    }

    log.Printf("Stream ended: %s | status=%s | duration=%s | sent=%d | recv=%d",
        info.FullMethod, statusCode, duration,
        wrapped.sentCount, wrapped.recvCount)

    return err
}

// countingServerStream wraps grpc.ServerStream to count messages.
type countingServerStream struct {
    grpc.ServerStream
    sentCount int64
    recvCount int64
}

func (s *countingServerStream) SendMsg(m interface{}) error {
    err := s.ServerStream.SendMsg(m)
    if err == nil {
        atomic.AddInt64(&s.sentCount, 1)
    }
    return err
}

func (s *countingServerStream) RecvMsg(m interface{}) error {
    err := s.ServerStream.RecvMsg(m)
    if err == nil {
        atomic.AddInt64(&s.recvCount, 1)
    }
    return err
}
```

### Stream Server Interceptor: Authentication

```go
// authStreamInterceptor validates auth for streaming RPCs.
func authStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    if isPublicMethod(info.FullMethod) {
        return handler(srv, ss)
    }

    md, ok := metadata.FromIncomingContext(ss.Context())
    if !ok {
        return status.Error(codes.Unauthenticated, "missing metadata")
    }

    authHeaders := md.Get("authorization")
    if len(authHeaders) == 0 {
        return status.Error(codes.Unauthenticated, "missing authorization header")
    }

    token := strings.TrimPrefix(authHeaders[0], "Bearer ")
    claims, err := validateToken(token)
    if err != nil {
        return status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Wrap the stream with an authenticated context.
    wrapped := &authenticatedServerStream{
        ServerStream: ss,
        ctx:          context.WithValue(ss.Context(), claimsKey{}, claims),
    }

    return handler(srv, wrapped)
}

type authenticatedServerStream struct {
    grpc.ServerStream
    ctx context.Context
}

func (s *authenticatedServerStream) Context() context.Context {
    return s.ctx
}
```

### Client Interceptors

```go
// loggingUnaryClientInterceptor logs outgoing unary RPCs.
func loggingUnaryClientInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()

    err := invoker(ctx, method, req, reply, cc, opts...)

    duration := time.Since(start)
    statusCode := codes.OK
    if err != nil {
        if st, ok := status.FromError(err); ok {
            statusCode = st.Code()
        }
    }

    log.Printf("Client RPC %s | status=%s | duration=%s",
        method, statusCode, duration)

    return err
}

// authUnaryClientInterceptor injects the auth token into outgoing requests.
func authUnaryClientInterceptor(tokenSource TokenSource) grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        token, err := tokenSource.Token(ctx)
        if err != nil {
            return status.Errorf(codes.Unauthenticated, "failed to get token: %v", err)
        }

        // Inject the token into outgoing metadata.
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)

        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

### Registering Interceptors

```go
// Server with interceptors
grpcServer := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryUnaryInterceptor,   // Outermost: catch panics
        loggingUnaryInterceptor,    // Log all requests
        authUnaryInterceptor,       // Authenticate
    ),
    grpc.ChainStreamInterceptor(
        recoveryStreamInterceptor,
        loggingStreamInterceptor,
        authStreamInterceptor,
    ),
)

// Client with interceptors
conn, err := grpc.NewClient(target,
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithChainUnaryInterceptor(
        loggingUnaryClientInterceptor,
        authUnaryClientInterceptor(myTokenSource),
    ),
    grpc.WithChainStreamInterceptor(
        loggingStreamClientInterceptor,
    ),
)
```

### Recovery Interceptor (Panic Handler)

```go
// recoveryUnaryInterceptor catches panics and returns Internal errors.
func recoveryUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("PANIC in %s: %v\n%s", info.FullMethod, r, debug.Stack())
            err = status.Errorf(codes.Internal, "internal server error")
        }
    }()
    return handler(ctx, req)
}

// recoveryStreamInterceptor catches panics in streaming RPCs.
func recoveryStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) (err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("PANIC in %s: %v\n%s", info.FullMethod, r, debug.Stack())
            err = status.Errorf(codes.Internal, "internal server error")
        }
    }()
    return handler(srv, ss)
}
```

---

## Error Handling with gRPC Status Codes

gRPC uses a well-defined set of status codes, similar to HTTP but more specific. Proper error handling is critical for building robust services.

### Status Code Reference

| Code                  | Number | HTTP Equiv | When to Use                                          |
|-----------------------|--------|------------|------------------------------------------------------|
| `OK`                  | 0      | 200        | Success                                              |
| `Cancelled`           | 1      | 499        | Client cancelled the request                         |
| `Unknown`             | 2      | 500        | Unknown error (catch-all)                            |
| `InvalidArgument`     | 3      | 400        | Bad request data                                     |
| `DeadlineExceeded`    | 4      | 504        | Timeout                                              |
| `NotFound`            | 5      | 404        | Resource not found                                   |
| `AlreadyExists`       | 6      | 409        | Resource already exists                              |
| `PermissionDenied`    | 7      | 403        | Authenticated but not authorized                     |
| `ResourceExhausted`   | 8      | 429        | Rate limiting, quota exceeded                        |
| `FailedPrecondition`  | 9      | 400        | System not in required state                         |
| `Aborted`             | 10     | 409        | Concurrency conflict (retry is appropriate)          |
| `OutOfRange`          | 11     | 400        | Seek past end of file, etc.                          |
| `Unimplemented`       | 12     | 501        | Method not implemented                               |
| `Internal`            | 13     | 500        | Server bug                                           |
| `Unavailable`         | 14     | 503        | Service temporarily unavailable (retry)              |
| `DataLoss`            | 15     | 500        | Unrecoverable data loss                              |
| `Unauthenticated`     | 16     | 401        | Missing or invalid credentials                       |

### Creating and Returning Errors

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Simple error
return nil, status.Error(codes.NotFound, "user not found")

// Formatted error
return nil, status.Errorf(codes.InvalidArgument,
    "name must be between 1 and 100 characters, got %d", len(name))
```

### Rich Error Details

gRPC supports attaching structured error details using the `errdetails` package.

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Validation error with field-level details.
func validateCreateUserRequest(req *pb.CreateUserRequest) error {
    var violations []*errdetails.BadRequest_FieldViolation

    if req.GetName() == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "name",
            Description: "name is required",
        })
    }
    if len(req.GetName()) > 100 {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "name",
            Description: "name must be at most 100 characters",
        })
    }
    if req.GetEmail() == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "email",
            Description: "email is required",
        })
    }

    if len(violations) > 0 {
        st := status.New(codes.InvalidArgument, "validation failed")
        detailed, err := st.WithDetails(&errdetails.BadRequest{
            FieldViolations: violations,
        })
        if err != nil {
            // If WithDetails fails, return the plain status.
            return st.Err()
        }
        return detailed.Err()
    }

    return nil
}

// Rate limiting error with retry info.
func rateLimitError(retryAfter time.Duration) error {
    st := status.New(codes.ResourceExhausted, "rate limit exceeded")
    detailed, err := st.WithDetails(&errdetails.RetryInfo{
        RetryDelay: durationpb.New(retryAfter),
    })
    if err != nil {
        return st.Err()
    }
    return detailed.Err()
}
```

### Handling Errors on the Client

```go
func handleGRPCError(err error) {
    if err == nil {
        return
    }

    // Extract the gRPC status.
    st, ok := status.FromError(err)
    if !ok {
        log.Printf("Non-gRPC error: %v", err)
        return
    }

    log.Printf("gRPC error: code=%s message=%q", st.Code(), st.Message())

    // Check for rich error details.
    for _, detail := range st.Details() {
        switch d := detail.(type) {
        case *errdetails.BadRequest:
            log.Println("Validation errors:")
            for _, v := range d.GetFieldViolations() {
                log.Printf("  - field %q: %s", v.GetField(), v.GetDescription())
            }
        case *errdetails.RetryInfo:
            delay := d.GetRetryDelay().AsDuration()
            log.Printf("Retry after: %s", delay)
        case *errdetails.DebugInfo:
            log.Printf("Debug info: %s", d.GetDetail())
        }
    }

    // Take action based on the status code.
    switch st.Code() {
    case codes.NotFound:
        log.Println("Resource not found; check the ID")
    case codes.Unauthenticated:
        log.Println("Need to re-authenticate")
    case codes.PermissionDenied:
        log.Println("Insufficient permissions")
    case codes.Unavailable:
        log.Println("Service unavailable; retry with backoff")
    case codes.DeadlineExceeded:
        log.Println("Request timed out; consider increasing deadline")
    }
}
```

### Wrapping Internal Errors

Never leak internal error details to clients. Wrap them appropriately.

```go
func (s *UserServiceServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {
    user, err := s.db.FindUserByID(ctx, req.GetId())
    if err != nil {
        // Log the full internal error for debugging.
        log.Printf("ERROR GetUser(%d): %v", req.GetId(), err)

        // Return a sanitized gRPC error to the client.
        if errors.Is(err, sql.ErrNoRows) {
            return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
        }

        // Generic internal error -- never expose DB error messages.
        return nil, status.Error(codes.Internal, "failed to retrieve user")
    }

    return &pb.GetUserResponse{User: toProtoUser(user)}, nil
}
```

---

## Deadlines and Timeouts

Deadlines prevent RPCs from hanging forever. In Go's gRPC, deadlines are propagated through `context.Context` and are respected at every hop in a microservice chain.

### Setting Deadlines on the Client

```go
// Timeout: relative duration from now.
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
if err != nil {
    st, _ := status.FromError(err)
    if st.Code() == codes.DeadlineExceeded {
        log.Println("Request timed out after 5 seconds")
    }
}

// Deadline: absolute point in time.
deadline := time.Now().Add(10 * time.Second)
ctx, cancel = context.WithDeadline(context.Background(), deadline)
defer cancel()

resp, err = client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

### Checking Deadlines on the Server

```go
func (s *UserServiceServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {
    // Check if the deadline has already passed.
    if ctx.Err() == context.DeadlineExceeded {
        return nil, status.Error(codes.DeadlineExceeded, "deadline already exceeded")
    }

    // Check remaining time.
    if deadline, ok := ctx.Deadline(); ok {
        remaining := time.Until(deadline)
        if remaining < 100*time.Millisecond {
            log.Printf("WARNING: only %s remaining before deadline", remaining)
        }
    }

    // Long-running operation: periodically check context.
    user, err := s.db.FindUserByID(ctx, req.GetId())
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, status.Error(codes.DeadlineExceeded, "deadline exceeded during DB query")
        }
        return nil, status.Errorf(codes.Internal, "db error: %v", err)
    }

    return &pb.GetUserResponse{User: toProtoUser(user)}, nil
}
```

### Deadline Propagation in Microservice Chains

When Service A calls Service B, the deadline propagates automatically through the context:

```
Client (5s deadline)
  --> Service A (sees ~5s remaining)
    --> Service B (sees ~4.8s remaining, accounting for network + processing)
      --> Service C (sees ~4.5s remaining)
```

```go
// Service A's handler: the deadline automatically propagates.
func (s *ServiceA) ProcessOrder(
    ctx context.Context,
    req *pb.ProcessOrderRequest,
) (*pb.ProcessOrderResponse, error) {
    // ctx already carries the client's deadline.
    // When we call Service B, it inherits the same deadline.
    userResp, err := s.userClient.GetUser(ctx, &pb.GetUserRequest{
        Id: req.GetUserId(),
    })
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }

    // You can also create a tighter sub-deadline.
    subCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    inventoryResp, err := s.inventoryClient.CheckStock(subCtx, &pb.CheckStockRequest{
        ProductId: req.GetProductId(),
    })
    if err != nil {
        return nil, fmt.Errorf("check stock: %w", err)
    }

    // ...process order...
    return &pb.ProcessOrderResponse{}, nil
}
```

> **Warning:** Always pass the incoming `ctx` (or a derived context) to downstream calls. Creating a fresh `context.Background()` breaks deadline propagation and is a common bug.

### Recommended Timeout Strategy

| RPC Type                  | Suggested Timeout | Rationale                           |
|---------------------------|-------------------|-------------------------------------|
| Simple DB lookup          | 1-5s              | Fast, bounded operation             |
| Aggregation across services| 5-15s            | Multiple hops, but bounded          |
| File upload/download      | 30-120s           | Large payloads, depends on size     |
| Streaming (long-lived)    | No timeout        | Use keepalive and cancellation instead|
| Background job trigger    | 5-10s             | Just triggers, does not wait for completion|

---

## Metadata: Headers and Trailers

Metadata is gRPC's equivalent of HTTP headers. It carries key-value pairs alongside RPC requests and responses.

### Metadata Anatomy

```
Client Request:
  ├── Headers (metadata sent before the request body)
  │     ├── authorization: Bearer eyJhbGciOi...
  │     ├── x-request-id: req-abc-123
  │     └── x-trace-id: trace-xyz-789
  ├── Request Message (protobuf body)
  └── (end of stream)

Server Response:
  ├── Headers (metadata sent before the response body)
  │     ├── x-request-id: req-abc-123
  │     └── x-ratelimit-remaining: 42
  ├── Response Message (protobuf body)
  └── Trailers (metadata sent after the response body)
        ├── x-processing-time-ms: 23
        └── x-server-id: server-05
```

### Sending Metadata from the Client

```go
import "google.golang.org/grpc/metadata"

// Method 1: Create metadata and attach to context.
md := metadata.New(map[string]string{
    "authorization": "Bearer my-token",
    "x-request-id":  "req-abc-123",
})
ctx := metadata.NewOutgoingContext(context.Background(), md)

// Method 2: Append to existing outgoing metadata.
ctx = metadata.AppendToOutgoingContext(ctx,
    "x-trace-id", "trace-xyz-789",
    "x-custom-header", "some-value",
)

// Make the RPC call with the metadata-carrying context.
resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

### Reading Metadata on the Server

```go
func (s *UserServiceServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {
    // Extract incoming metadata.
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Internal, "failed to get metadata")
    }

    // Read specific values.
    requestIDs := md.Get("x-request-id")
    if len(requestIDs) > 0 {
        log.Printf("Request ID: %s", requestIDs[0])
    }

    // Read all metadata keys.
    for key, values := range md {
        log.Printf("Metadata: %s = %v", key, values)
    }

    // ...handle request...
    return &pb.GetUserResponse{User: user}, nil
}
```

### Sending Response Headers and Trailers from the Server

```go
func (s *UserServiceServer) GetUser(
    ctx context.Context,
    req *pb.GetUserRequest,
) (*pb.GetUserResponse, error) {
    start := time.Now()

    // Send response headers before the response.
    header := metadata.Pairs(
        "x-request-id", extractRequestID(ctx),
        "x-ratelimit-remaining", "42",
    )
    if err := grpc.SendHeader(ctx, header); err != nil {
        log.Printf("Failed to send header: %v", err)
    }

    // ...process request...
    user, err := s.findUser(ctx, req.GetId())
    if err != nil {
        return nil, err
    }

    // Set trailers (sent after the response).
    trailer := metadata.Pairs(
        "x-processing-time-ms", fmt.Sprintf("%d", time.Since(start).Milliseconds()),
        "x-server-id", s.serverID,
    )
    grpc.SetTrailer(ctx, trailer)

    return &pb.GetUserResponse{User: user}, nil
}
```

### Receiving Headers and Trailers on the Client

```go
// Declare variables to capture headers and trailers.
var header, trailer metadata.MD

resp, err := client.GetUser(ctx,
    &pb.GetUserRequest{Id: 42},
    grpc.Header(&header),     // Capture response headers
    grpc.Trailer(&trailer),   // Capture response trailers
)

if err != nil {
    // Trailers are still available even on error.
    log.Printf("Error: %v", err)
    if serverIDs := trailer.Get("x-server-id"); len(serverIDs) > 0 {
        log.Printf("Error came from server: %s", serverIDs[0])
    }
    return
}

// Read response headers.
if remaining := header.Get("x-ratelimit-remaining"); len(remaining) > 0 {
    log.Printf("Rate limit remaining: %s", remaining[0])
}

// Read response trailers.
if duration := trailer.Get("x-processing-time-ms"); len(duration) > 0 {
    log.Printf("Server processing time: %sms", duration[0])
}
```

### Metadata for Streaming RPCs

```go
// Server: sending headers and trailers on a stream.
func (s *ChatServer) SubscribeToRoom(
    req *pb.SubscribeRequest,
    stream pb.ChatService_SubscribeToRoomServer,
) error {
    // Send initial headers.
    header := metadata.Pairs("x-room-id", req.GetRoomId())
    if err := stream.SendHeader(header); err != nil {
        return err
    }

    // ...stream messages...

    // Set trailers (sent when the stream ends).
    stream.SetTrailer(metadata.Pairs(
        "x-messages-sent", fmt.Sprintf("%d", messageCount),
    ))

    return nil
}

// Client: reading headers and trailers from a stream.
func readStream(ctx context.Context, client pb.ChatServiceClient) error {
    stream, err := client.SubscribeToRoom(ctx, &pb.SubscribeRequest{
        RoomId: "general",
    })
    if err != nil {
        return err
    }

    // Read headers (blocks until headers arrive).
    header, err := stream.Header()
    if err != nil {
        return fmt.Errorf("reading headers: %w", err)
    }
    log.Printf("Room ID from header: %v", header.Get("x-room-id"))

    // Read messages.
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        fmt.Printf("Message: %s\n", msg.GetContent())
    }

    // Read trailers (available after stream ends).
    trailer := stream.Trailer()
    log.Printf("Messages sent: %v", trailer.Get("x-messages-sent"))

    return nil
}
```

> **Tip:** Metadata keys are case-insensitive and are lowercased by gRPC. Keys starting with `grpc-` are reserved. Binary values can be sent using keys ending in `-bin` (they are base64-encoded automatically).

---

## gRPC-Gateway: REST + gRPC

gRPC-Gateway generates a reverse proxy that translates RESTful JSON API calls into gRPC calls. This lets you serve both REST and gRPC from the same service.

### How It Works

```
Browser/curl          gRPC-Gateway Proxy         gRPC Server
    │                       │                        │
    │  POST /v1/users       │                        │
    │  {"name": "Alice"}    │                        │
    │ ──────────────────>   │                        │
    │                       │  CreateUser(req)       │
    │                       │ ────────────────────>  │
    │                       │                        │
    │                       │  CreateUserResponse    │
    │                       │ <────────────────────  │
    │  200 OK               │                        │
    │  {"user": {...}}      │                        │
    │ <──────────────────   │                        │
```

### Step 1: Annotate the .proto File

```protobuf
syntax = "proto3";

package userservice.v1;

import "google/api/annotations.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";
import "google/protobuf/timestamp.proto";

option go_package = "github.com/yourorg/project/gen/userservice/v1;userservicev1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }

  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"
    };
  }

  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse) {
    option (google.api.http) = {
      patch: "/v1/users/{user.id}"
      body: "user"
    };
  }

  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      delete: "/v1/users/{id}"
    };
  }

  rpc ListUsers(ListUsersRequest) returns (stream User) {
    option (google.api.http) = {
      get: "/v1/users"
    };
  }
}
```

### Step 2: Install the Gateway Plugin

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

### Step 3: Generate Gateway Code

Using Buf:

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/gateway
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc-ecosystem/openapiv2
    out: gen/openapiv2
```

Or with raw protoc:

```bash
protoc \
  --proto_path=proto \
  --proto_path=third_party \
  --go_out=gen --go_opt=paths=source_relative \
  --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
  --grpc-gateway_out=gen --grpc-gateway_opt=paths=source_relative \
  --openapiv2_out=gen/openapiv2 \
  proto/userservice/v1/user_service.proto
```

This generates `user_service.pb.gw.go` with the REST-to-gRPC proxy.

### Step 4: Serve Both gRPC and REST

```go
// cmd/server/main.go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "net"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    pb "github.com/yourorg/project/gen/userservice/v1"
    "github.com/yourorg/project/internal/server"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/reflection"
)

func main() {
    grpcPort := flag.Int("grpc-port", 50051, "gRPC server port")
    httpPort := flag.Int("http-port", 8080, "HTTP gateway port")
    flag.Parse()

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // ── Start gRPC server ────────────────────────────────
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *grpcPort))
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    grpcServer := grpc.NewServer()
    userService := server.NewUserServiceServer()
    pb.RegisterUserServiceServer(grpcServer, userService)
    reflection.Register(grpcServer)

    go func() {
        log.Printf("gRPC server listening on :%d", *grpcPort)
        if err := grpcServer.Serve(lis); err != nil {
            log.Fatalf("gRPC server failed: %v", err)
        }
    }()

    // ── Start HTTP gateway ───────────────────────────────
    mux := runtime.NewServeMux(
        // Customize JSON marshaling to use original proto field names.
        runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{
            MarshalOptions: protojson.MarshalOptions{
                UseProtoNames:   true,
                EmitUnpopulated: true,
            },
            UnmarshalOptions: protojson.UnmarshalOptions{
                DiscardUnknown: true,
            },
        }),
    )

    opts := []grpc.DialOption{
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    }

    err = pb.RegisterUserServiceHandlerFromEndpoint(ctx, mux,
        fmt.Sprintf("localhost:%d", *grpcPort), opts)
    if err != nil {
        log.Fatalf("failed to register gateway: %v", err)
    }

    httpServer := &http.Server{
        Addr:    fmt.Sprintf(":%d", *httpPort),
        Handler: mux,
    }

    go func() {
        log.Printf("HTTP gateway listening on :%d", *httpPort)
        if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("HTTP server failed: %v", err)
        }
    }()

    // ── Graceful shutdown ────────────────────────────────
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh

    log.Println("Shutting down...")
    httpServer.Shutdown(ctx)
    grpcServer.GracefulStop()
}
```

### Using the REST API

```bash
# Create a user
curl -X POST http://localhost:8080/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com", "role": "ROLE_ADMIN"}'

# Get a user
curl http://localhost:8080/v1/users/1

# Update a user
curl -X PATCH http://localhost:8080/v1/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Johnson"}'

# Delete a user
curl -X DELETE http://localhost:8080/v1/users/1

# List users
curl "http://localhost:8080/v1/users?page_size=10"
```

### Serving on a Single Port with cmux

For simplicity, you can multiplex gRPC and HTTP on the same port:

```go
import "github.com/soheilhy/cmux"

func main() {
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal(err)
    }

    // Create a connection multiplexer.
    m := cmux.New(lis)

    // gRPC connections match HTTP/2 with the gRPC content type.
    grpcLis := m.MatchWithWriters(
        cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"),
    )
    // Everything else is HTTP.
    httpLis := m.Match(cmux.Any())

    // Start both servers.
    go grpcServer.Serve(grpcLis)
    go httpServer.Serve(httpLis)

    log.Println("Serving gRPC and HTTP on :8080")
    m.Serve()
}
```

---

## Health Checking and Reflection

### gRPC Health Checking Protocol

The standard health check service lets load balancers and orchestrators (like Kubernetes) probe your service.

```go
import (
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

func main() {
    grpcServer := grpc.NewServer()

    // Register your services.
    pb.RegisterUserServiceServer(grpcServer, userService)

    // Register the standard health service.
    healthServer := health.NewServer()
    healthpb.RegisterHealthServer(grpcServer, healthServer)

    // Set the serving status for specific services.
    healthServer.SetServingStatus(
        "userservice.v1.UserService",
        healthpb.HealthCheckResponse_SERVING,
    )

    // Set overall health.
    healthServer.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)

    // During shutdown, mark as not serving.
    go func() {
        <-shutdownCh
        healthServer.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
        healthServer.Shutdown() // Terminates all Watch streams
    }()

    // ...start server...
}
```

### Health Check Statuses

| Status          | Meaning                                          |
|-----------------|--------------------------------------------------|
| `UNKNOWN`       | Default state; service status is not known.      |
| `SERVING`       | Service is healthy and ready to handle requests. |
| `NOT_SERVING`   | Service is unhealthy; do not send requests.      |
| `SERVICE_UNKNOWN` | The requested service name is not registered.  |

### Checking Health from the Client

```go
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

func checkHealth(ctx context.Context, conn *grpc.ClientConn) error {
    healthClient := healthpb.NewHealthClient(conn)

    // Check a specific service.
    resp, err := healthClient.Check(ctx, &healthpb.HealthCheckRequest{
        Service: "userservice.v1.UserService",
    })
    if err != nil {
        return fmt.Errorf("health check failed: %w", err)
    }

    if resp.GetStatus() != healthpb.HealthCheckResponse_SERVING {
        return fmt.Errorf("service is not healthy: %s", resp.GetStatus())
    }

    log.Println("Service is healthy")
    return nil
}

// Watch health changes (server streaming).
func watchHealth(ctx context.Context, conn *grpc.ClientConn) error {
    healthClient := healthpb.NewHealthClient(conn)

    stream, err := healthClient.Watch(ctx, &healthpb.HealthCheckRequest{
        Service: "userservice.v1.UserService",
    })
    if err != nil {
        return err
    }

    for {
        resp, err := stream.Recv()
        if err != nil {
            return err
        }
        log.Printf("Health status changed: %s", resp.GetStatus())
    }
}
```

### Using grpc-health-probe

```bash
# Install
go install github.com/grpc-ecosystem/grpc-health-probe@latest

# Check health
grpc-health-probe -addr=localhost:50051
# Output: status: SERVING

# Check a specific service
grpc-health-probe -addr=localhost:50051 -service=userservice.v1.UserService
```

### Kubernetes Liveness and Readiness Probes

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      containers:
        - name: user-service
          image: yourorg/user-service:latest
          ports:
            - containerPort: 50051
              name: grpc
          livenessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            grpc:
              port: 50051
              service: "userservice.v1.UserService"
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Server Reflection

Reflection lets tools like `grpcurl` and `grpcui` discover your services and methods at runtime without needing the `.proto` files.

```go
import "google.golang.org/grpc/reflection"

func main() {
    grpcServer := grpc.NewServer()
    pb.RegisterUserServiceServer(grpcServer, userService)

    // Enable reflection. Remove this in production if you do not want
    // services to be discoverable.
    reflection.Register(grpcServer)

    // ...
}
```

### Using grpcurl (like curl for gRPC)

```bash
# Install
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# List all services
grpcurl -plaintext localhost:50051 list
# Output:
# grpc.health.v1.Health
# grpc.reflection.v1.ServerReflection
# userservice.v1.UserService

# Describe a service
grpcurl -plaintext localhost:50051 describe userservice.v1.UserService
# Output:
# userservice.v1.UserService is a service:
# service UserService {
#   rpc CreateUser ( .userservice.v1.CreateUserRequest ) returns ( .userservice.v1.CreateUserResponse );
#   rpc DeleteUser ( .userservice.v1.DeleteUserRequest ) returns ( .google.protobuf.Empty );
#   rpc GetUser ( .userservice.v1.GetUserRequest ) returns ( .userservice.v1.GetUserResponse );
#   ...
# }

# Call an RPC
grpcurl -plaintext -d '{"name": "Alice", "email": "alice@example.com", "role": "ROLE_ADMIN"}' \
  localhost:50051 userservice.v1.UserService/CreateUser

# Call with metadata
grpcurl -plaintext \
  -H "authorization: Bearer my-token" \
  -d '{"id": 1}' \
  localhost:50051 userservice.v1.UserService/GetUser
```

> **Warning:** Disable reflection in production environments where service discovery should not be public. Alternatively, protect it behind authentication.

---

## Testing gRPC Services

### The bufconn Approach

`bufconn` provides an in-memory listener that avoids real network I/O. This makes tests fast, portable, and free from port conflicts.

```go
// internal/server/user_service_test.go
package server_test

import (
    "context"
    "log"
    "net"
    "testing"
    "time"

    pb "github.com/yourorg/project/gen/userservice/v1"
    "github.com/yourorg/project/internal/server"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
    "google.golang.org/grpc/test/bufconn"
)

const bufSize = 1024 * 1024

// newTestServer creates an in-memory gRPC server and returns a client connection.
func newTestServer(t *testing.T) (pb.UserServiceClient, func()) {
    t.Helper()

    // Create an in-memory listener.
    lis := bufconn.Listen(bufSize)

    // Create and configure the gRPC server.
    grpcServer := grpc.NewServer()
    userService := server.NewUserServiceServer()
    pb.RegisterUserServiceServer(grpcServer, userService)

    // Start serving in a goroutine.
    go func() {
        if err := grpcServer.Serve(lis); err != nil {
            log.Printf("Server exited with error: %v", err)
        }
    }()

    // Create a client connection using the in-memory listener.
    conn, err := grpc.NewClient(
        "passthrough://bufnet",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx)
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        t.Fatalf("Failed to create client: %v", err)
    }

    client := pb.NewUserServiceClient(conn)

    // Return a cleanup function.
    cleanup := func() {
        conn.Close()
        grpcServer.GracefulStop()
        lis.Close()
    }

    return client, cleanup
}
```

### Unit Tests for Unary RPCs

```go
func TestCreateUser(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Test successful creation.
    resp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Alice Johnson",
        Email: "alice@example.com",
        Role:  pb.Role_ROLE_ADMIN,
        Tags:  []string{"engineering"},
    })
    if err != nil {
        t.Fatalf("CreateUser failed: %v", err)
    }

    if resp.GetUser().GetName() != "Alice Johnson" {
        t.Errorf("got name %q, want %q", resp.GetUser().GetName(), "Alice Johnson")
    }
    if resp.GetUser().GetEmail() != "alice@example.com" {
        t.Errorf("got email %q, want %q", resp.GetUser().GetEmail(), "alice@example.com")
    }
    if resp.GetUser().GetRole() != pb.Role_ROLE_ADMIN {
        t.Errorf("got role %v, want %v", resp.GetUser().GetRole(), pb.Role_ROLE_ADMIN)
    }
    if resp.GetUser().GetId() <= 0 {
        t.Errorf("expected positive ID, got %d", resp.GetUser().GetId())
    }
    if resp.GetUser().GetCreatedAt() == nil {
        t.Error("created_at should not be nil")
    }
}

func TestCreateUser_Validation(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    tests := []struct {
        name     string
        req      *pb.CreateUserRequest
        wantCode codes.Code
        wantMsg  string
    }{
        {
            name:     "missing name",
            req:      &pb.CreateUserRequest{Email: "test@example.com"},
            wantCode: codes.InvalidArgument,
            wantMsg:  "name is required",
        },
        {
            name:     "missing email",
            req:      &pb.CreateUserRequest{Name: "Test"},
            wantCode: codes.InvalidArgument,
            wantMsg:  "email is required",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := client.CreateUser(ctx, tt.req)
            if err == nil {
                t.Fatal("expected an error, got nil")
            }

            st, ok := status.FromError(err)
            if !ok {
                t.Fatalf("expected gRPC status error, got: %v", err)
            }
            if st.Code() != tt.wantCode {
                t.Errorf("got code %v, want %v", st.Code(), tt.wantCode)
            }
            if st.Message() != tt.wantMsg {
                t.Errorf("got message %q, want %q", st.Message(), tt.wantMsg)
            }
        })
    }
}

func TestGetUser_NotFound(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    _, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 999})
    if err == nil {
        t.Fatal("expected error for non-existent user")
    }

    st, ok := status.FromError(err)
    if !ok {
        t.Fatalf("expected gRPC status, got: %v", err)
    }
    if st.Code() != codes.NotFound {
        t.Errorf("got code %v, want NotFound", st.Code())
    }
}
```

### Testing the Full CRUD Lifecycle

```go
func TestUserCRUDLifecycle(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 1. Create
    createResp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Bob Smith",
        Email: "bob@example.com",
        Role:  pb.Role_ROLE_EDITOR,
    })
    if err != nil {
        t.Fatalf("CreateUser: %v", err)
    }
    userID := createResp.GetUser().GetId()

    // 2. Read
    getResp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: userID})
    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }
    if getResp.GetUser().GetName() != "Bob Smith" {
        t.Errorf("got name %q, want %q", getResp.GetUser().GetName(), "Bob Smith")
    }

    // 3. Update
    updateResp, err := client.UpdateUser(ctx, &pb.UpdateUserRequest{
        User: &pb.User{
            Id:   userID,
            Name: "Robert Smith",
        },
        UpdateMask: &fieldmaskpb.FieldMask{Paths: []string{"name"}},
    })
    if err != nil {
        t.Fatalf("UpdateUser: %v", err)
    }
    if updateResp.GetUser().GetName() != "Robert Smith" {
        t.Errorf("got name %q, want %q", updateResp.GetUser().GetName(), "Robert Smith")
    }
    // Verify email was NOT changed.
    if updateResp.GetUser().GetEmail() != "bob@example.com" {
        t.Errorf("email should not have changed, got %q", updateResp.GetUser().GetEmail())
    }

    // 4. Delete
    _, err = client.DeleteUser(ctx, &pb.DeleteUserRequest{Id: userID})
    if err != nil {
        t.Fatalf("DeleteUser: %v", err)
    }

    // 5. Verify deletion
    _, err = client.GetUser(ctx, &pb.GetUserRequest{Id: userID})
    st, ok := status.FromError(err)
    if !ok || st.Code() != codes.NotFound {
        t.Errorf("expected NotFound after deletion, got: %v", err)
    }
}
```

### Testing Duplicate Email Constraint

```go
func TestCreateUser_DuplicateEmail(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Create first user.
    _, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Alice",
        Email: "alice@example.com",
    })
    if err != nil {
        t.Fatalf("first CreateUser: %v", err)
    }

    // Try to create another user with the same email.
    _, err = client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Alice Clone",
        Email: "alice@example.com",
    })
    if err == nil {
        t.Fatal("expected error for duplicate email")
    }

    st, ok := status.FromError(err)
    if !ok {
        t.Fatalf("expected gRPC status, got: %v", err)
    }
    if st.Code() != codes.AlreadyExists {
        t.Errorf("got code %v, want AlreadyExists", st.Code())
    }
}
```

### Testing Server Streaming

```go
func TestListUsers_Streaming(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Seed some users.
    names := []string{"Alice", "Bob", "Charlie", "Diana", "Eve"}
    for i, name := range names {
        _, err := client.CreateUser(ctx, &pb.CreateUserRequest{
            Name:  name,
            Email: fmt.Sprintf("%s@example.com", strings.ToLower(name)),
            Role:  pb.Role(i%3 + 1),
        })
        if err != nil {
            t.Fatalf("CreateUser(%s): %v", name, err)
        }
    }

    // Open the stream.
    stream, err := client.ListUsers(ctx, &pb.ListUsersRequest{
        PageSize: 10,
    })
    if err != nil {
        t.Fatalf("ListUsers: %v", err)
    }

    // Collect all streamed users.
    var users []*pb.User
    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            t.Fatalf("stream.Recv: %v", err)
        }
        users = append(users, user)
    }

    if len(users) != len(names) {
        t.Errorf("got %d users, want %d", len(users), len(names))
    }
}
```

### Testing with Interceptors

```go
func newTestServerWithAuth(t *testing.T) (pb.UserServiceClient, func()) {
    t.Helper()

    lis := bufconn.Listen(bufSize)

    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            authUnaryInterceptor,
        ),
    )

    userService := server.NewUserServiceServer()
    pb.RegisterUserServiceServer(grpcServer, userService)

    go func() {
        grpcServer.Serve(lis)
    }()

    conn, err := grpc.NewClient(
        "passthrough://bufnet",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx)
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        t.Fatalf("Failed to dial: %v", err)
    }

    client := pb.NewUserServiceClient(conn)
    cleanup := func() {
        conn.Close()
        grpcServer.GracefulStop()
        lis.Close()
    }

    return client, cleanup
}

func TestAuth_MissingToken(t *testing.T) {
    client, cleanup := newTestServerWithAuth(t)
    defer cleanup()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Call without auth metadata.
    _, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
    if err == nil {
        t.Fatal("expected auth error")
    }

    st, _ := status.FromError(err)
    if st.Code() != codes.Unauthenticated {
        t.Errorf("got code %v, want Unauthenticated", st.Code())
    }
}

func TestAuth_ValidToken(t *testing.T) {
    client, cleanup := newTestServerWithAuth(t)
    defer cleanup()

    // Inject a valid token.
    ctx := metadata.AppendToOutgoingContext(
        context.Background(),
        "authorization", "Bearer valid-test-token",
    )
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // First create a user (assuming the token is valid).
    _, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Test User",
        Email: "test@example.com",
    })
    // The test passes as long as we do not get Unauthenticated.
    if err != nil {
        st, _ := status.FromError(err)
        if st.Code() == codes.Unauthenticated {
            t.Fatalf("should have passed auth: %v", err)
        }
    }
}
```

### Testing Deadlines

```go
func TestGetUser_DeadlineExceeded(t *testing.T) {
    client, cleanup := newTestServer(t)
    defer cleanup()

    // Use an already-expired context.
    ctx, cancel := context.WithTimeout(context.Background(), 0)
    defer cancel()

    _, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
    if err == nil {
        t.Fatal("expected deadline exceeded error")
    }

    st, ok := status.FromError(err)
    if !ok {
        t.Fatalf("expected gRPC status: %v", err)
    }
    if st.Code() != codes.DeadlineExceeded {
        t.Errorf("got code %v, want DeadlineExceeded", st.Code())
    }
}
```

### Benchmarking gRPC

```go
func BenchmarkGetUser(b *testing.B) {
    client, cleanup := newTestServer(&testing.T{})
    defer cleanup()

    ctx := context.Background()

    // Seed a user.
    resp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:  "Benchmark User",
        Email: "bench@example.com",
    })
    if err != nil {
        b.Fatal(err)
    }
    userID := resp.GetUser().GetId()

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, err := client.GetUser(ctx, &pb.GetUserRequest{Id: userID})
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}
```

---

## Performance Considerations

### Connection Management

```go
// DO: Reuse connections. A single grpc.ClientConn multiplexes RPCs over HTTP/2.
conn, _ := grpc.NewClient(target, opts...)
client := pb.NewUserServiceClient(conn)
// Use 'client' for all RPCs. Do NOT create a new connection per RPC.

// DON'T: Create a new connection for every call.
// This is extremely wasteful -- each connection has TCP + TLS handshake overhead.
```

### Message Size Optimization

```go
// Server: set maximum message sizes
grpcServer := grpc.NewServer(
    grpc.MaxRecvMsgSize(16 * 1024 * 1024), // 16MB (default is 4MB)
    grpc.MaxSendMsgSize(16 * 1024 * 1024),
)

// Client: set per-call message size
resp, err := client.GetLargeData(ctx, req,
    grpc.MaxCallRecvMsgSize(32*1024*1024),
)
```

### Keepalive Configuration

```go
import "google.golang.org/grpc/keepalive"

// Server keepalive
grpcServer := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     5 * time.Minute,  // Close idle connections after 5m
        MaxConnectionAge:      30 * time.Minute,  // Close connections after 30m (for load balancing)
        MaxConnectionAgeGrace: 10 * time.Second,  // Grace period to finish RPCs
        Time:                  1 * time.Minute,   // Ping clients every 1m if idle
        Timeout:               20 * time.Second,  // Wait 20s for ping ack
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             30 * time.Second, // Minimum allowed ping interval from clients
        PermitWithoutStream: false,            // No pings without active streams
    }),
)

// Client keepalive
conn, _ := grpc.NewClient(target,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                30 * time.Second, // Ping server every 30s if idle
        Timeout:             10 * time.Second, // Wait 10s for ping ack
        PermitWithoutStream: false,
    }),
)
```

### Compression

```go
import "google.golang.org/grpc/encoding/gzip"

// Client: enable gzip compression for a specific call.
resp, err := client.GetLargeData(ctx, req,
    grpc.UseCompressor(gzip.Name),
)

// Client: enable gzip compression for all calls via interceptor or default options.
conn, _ := grpc.NewClient(target,
    grpc.WithDefaultCallOptions(
        grpc.UseCompressor(gzip.Name),
    ),
)

// Server: gzip decompression is enabled by default.
// Importing the package registers the codec:
import _ "google.golang.org/grpc/encoding/gzip"
```

### Connection Pooling for High Throughput

A single `grpc.ClientConn` uses one HTTP/2 connection, which maxes out at ~100 concurrent streams by default. For extremely high throughput, use a connection pool:

```go
type ConnPool struct {
    conns []*grpc.ClientConn
    next  atomic.Uint64
}

func NewConnPool(target string, size int, opts ...grpc.DialOption) (*ConnPool, error) {
    pool := &ConnPool{
        conns: make([]*grpc.ClientConn, size),
    }
    for i := 0; i < size; i++ {
        conn, err := grpc.NewClient(target, opts...)
        if err != nil {
            // Clean up already-created connections.
            for j := 0; j < i; j++ {
                pool.conns[j].Close()
            }
            return nil, fmt.Errorf("creating connection %d: %w", i, err)
        }
        pool.conns[i] = conn
    }
    return pool, nil
}

// Get returns the next connection in round-robin fashion.
func (p *ConnPool) Get() *grpc.ClientConn {
    idx := p.next.Add(1) - 1
    return p.conns[idx%uint64(len(p.conns))]
}

func (p *ConnPool) Close() {
    for _, conn := range p.conns {
        conn.Close()
    }
}
```

### Proto Message Optimization Tips

```protobuf
// DO: Use appropriate integer types.
// If values are always positive and < 2^28, use fixed32 instead of uint64.
message Stats {
  fixed32 request_count = 1;  // 4 bytes always (efficient for values > 2^28)
  uint32 small_count = 2;     // variable length (efficient for small values)
  sint32 temperature = 3;     // efficient for negative numbers
}

// DO: Put frequently-set fields in positions 1-15 (one byte on wire).
message Event {
  string id = 1;          // Frequently set -- 1 byte tag
  int64 timestamp = 2;    // Frequently set -- 1 byte tag
  string type = 3;        // Frequently set -- 1 byte tag
  // ...
  string debug_info = 16; // Rarely set -- 2 byte tag (OK)
  string trace_id = 17;   // Rarely set -- 2 byte tag (OK)
}
```

---

## Best Practices

### 1. API Design

```
Rule                                      Example
─────────────────────────────────────────────────────────────────────
Use versioned packages                    package userservice.v1;
Use clear method names                    GetUser, not Fetch or Read
Separate request/response per method      GetUserRequest, GetUserResponse
Use field masks for partial updates       UpdateUserRequest.update_mask
Use page tokens for pagination            ListUsersRequest.page_token
Always have UNSPECIFIED = 0 for enums     ROLE_UNSPECIFIED = 0
Use google well-known types               Timestamp, Duration, FieldMask
Prefix enum values with the enum name     ROLE_ADMIN, not ADMIN
```

### 2. Proto File Organization

```
proto/
├── userservice/
│   └── v1/
│       ├── user_service.proto     # Service definition
│       ├── user.proto             # Message definitions
│       └── enums.proto            # Shared enums
├── orderservice/
│   └── v1/
│       └── order_service.proto
└── common/
    └── v1/
        ├── pagination.proto       # Shared pagination messages
        └── money.proto            # Shared money type
```

### 3. Error Handling Checklist

- Always return gRPC status errors, never raw Go errors.
- Use the most specific status code (not just Internal for everything).
- Include actionable error messages.
- Never expose internal details (stack traces, SQL queries) in error messages.
- Log internal errors server-side; return sanitized messages to clients.
- Use rich error details (`errdetails`) for field-level validation errors.
- Document which error codes each RPC can return.

### 4. Context and Deadline Guidelines

- Always pass `context.Context` through the call chain. Never use `context.Background()` inside a handler (use the incoming context).
- Set deadlines on every client call.
- Check `ctx.Err()` in long-running server operations.
- Do not set deadlines on long-lived streaming RPCs -- use keepalive and cancellation instead.
- Propagate the incoming context to downstream RPCs.

### 5. Interceptor Ordering

Apply interceptors from outermost to innermost:

```
Request flow:  Recovery -> Logging -> Auth -> Rate Limit -> Handler
Response flow: Handler -> Rate Limit -> Auth -> Logging -> Recovery
```

```go
grpc.ChainUnaryInterceptor(
    recoveryInterceptor,    // 1st: catch panics from everything below
    loggingInterceptor,     // 2nd: log all requests (including auth failures)
    authInterceptor,        // 3rd: authenticate
    rateLimitInterceptor,   // 4th: rate limit authenticated users
)
```

### 6. Testing Strategy

| Layer              | What to Test                                | Tool              |
|--------------------|---------------------------------------------|-------------------|
| Unit (handler)     | Business logic, validation, error codes     | bufconn           |
| Integration        | Full server with interceptors + middleware  | bufconn           |
| Contract           | Proto compatibility / breaking changes      | `buf breaking`    |
| Load               | Throughput, latency under load              | `ghz` tool        |
| End-to-end         | Full deployment with real network           | grpcurl / custom  |

### 7. Security Checklist

```
[ ] Use TLS in production (never insecure credentials)
[ ] Validate all input fields on the server
[ ] Implement authentication (JWT, mTLS, or API keys)
[ ] Implement authorization (check permissions per RPC)
[ ] Set maximum message sizes
[ ] Set deadlines on all RPCs
[ ] Rate limit to prevent abuse
[ ] Disable reflection in production (or protect it)
[ ] Use the recovery interceptor to catch panics
[ ] Sanitize error messages (no internal details)
```

### 8. Observability

```go
// Integrate with OpenTelemetry for tracing and metrics.
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

grpcServer := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

conn, err := grpc.NewClient(target,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

Key metrics to track:

| Metric                          | Type      | Description                        |
|---------------------------------|-----------|------------------------------------|
| `grpc_server_handled_total`     | Counter   | Total RPCs completed by status code|
| `grpc_server_handling_seconds`  | Histogram | RPC latency distribution           |
| `grpc_server_msg_received_total`| Counter   | Messages received (streaming)      |
| `grpc_server_msg_sent_total`    | Counter   | Messages sent (streaming)          |
| `grpc_client_handled_total`     | Counter   | Client-side RPCs by status code    |
| `grpc_client_handling_seconds`  | Histogram | Client-perceived latency           |

### 9. Schema Evolution Rules

| Change                      | Safe?  | Notes                                       |
|-----------------------------|--------|---------------------------------------------|
| Add a new field             | Yes    | Old clients ignore unknown fields           |
| Remove a field              | Yes*   | Reserve the number and name                 |
| Rename a field              | Yes    | Wire format uses numbers, not names         |
| Change field number         | NO     | Breaks binary compatibility                 |
| Change field type           | NO*    | Some changes are compatible (int32 -> int64)|
| Add an enum value           | Yes    | Old clients see it as the numeric value     |
| Remove an enum value        | Yes*   | Reserve the number and name                 |
| Add a service method        | Yes    | Old clients do not call it                  |
| Remove a service method     | NO     | Breaks clients that call it                 |
| Change method signature     | NO     | Breaks clients                              |

Items marked * require using `reserved` to prevent reuse.

### 10. Common Pitfalls

**Pitfall 1: Forgetting to close streams**
```go
// BAD: leaks resources if you stop reading early.
stream, _ := client.ListUsers(ctx, req)
msg, _ := stream.Recv() // Read only one message, never close.

// GOOD: always drain or cancel.
stream, _ := client.ListUsers(ctx, req)
for {
    msg, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        // Handle error
        break
    }
    // process msg
}
```

**Pitfall 2: Using the wrong context in handlers**
```go
// BAD: creates a new context without the client's deadline.
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    freshCtx := context.Background() // WRONG -- loses deadline + metadata.
    result, err := s.db.Query(freshCtx, ...)
}

// GOOD: use the incoming context.
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    result, err := s.db.Query(ctx, ...) // Deadline and cancellation propagate.
}
```

**Pitfall 3: Not handling io.EOF in streams**
```go
// BAD: treats EOF as an error.
msg, err := stream.Recv()
if err != nil {
    return fmt.Errorf("recv failed: %w", err) // EOF is not an error!
}

// GOOD: handle EOF as normal stream termination.
msg, err := stream.Recv()
if err == io.EOF {
    return nil // Stream ended normally.
}
if err != nil {
    return fmt.Errorf("recv failed: %w", err)
}
```

**Pitfall 4: Returning raw Go errors from handlers**
```go
// BAD: raw errors become status.Unknown on the client.
return nil, fmt.Errorf("user not found") // Client gets codes.Unknown

// GOOD: use status errors.
return nil, status.Error(codes.NotFound, "user not found") // Client gets codes.NotFound
```

**Pitfall 5: Blocking forever without a deadline**
```go
// BAD: no deadline -- hangs forever if server is slow.
ctx := context.Background()
resp, err := client.GetUser(ctx, req)

// GOOD: always set a deadline.
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

---

## Summary

This chapter covered gRPC and Protocol Buffers end to end:

| Topic                     | Key Takeaway                                                        |
|---------------------------|---------------------------------------------------------------------|
| Protocol Buffers          | Binary format; 3-10x smaller and 20-100x faster than JSON          |
| .proto files              | Single source of truth for your API contract                        |
| Code generation           | `protoc` or `buf generate` produces Go structs and gRPC stubs       |
| Unary RPCs                | One request, one response -- the most common pattern                |
| Server streaming          | One request, many responses -- lists, feeds, subscriptions          |
| Client streaming          | Many requests, one response -- uploads, batches                     |
| Bidi streaming            | Independent streams in both directions -- chat, sync                |
| Interceptors              | Middleware for logging, auth, recovery, metrics                     |
| Error handling            | Use `status.Error` with specific codes; use `errdetails` for detail |
| Deadlines                 | Always set via context; propagate through call chains               |
| Metadata                  | Headers and trailers for auth tokens, request IDs, tracing          |
| gRPC-Gateway              | Serve REST and gRPC from the same service                           |
| Health checking            | Standard protocol for Kubernetes and load balancers                 |
| Reflection                | Runtime service discovery for tools like grpcurl                    |
| Testing with bufconn      | Fast, in-memory testing without real network I/O                    |
| Performance               | Reuse connections, use keepalive, compress large payloads           |

### Quick Reference: Essential Imports

```go
import (
    // gRPC core
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"

    // Error details
    "google.golang.org/genproto/googleapis/rpc/errdetails"

    // Well-known types
    "google.golang.org/protobuf/types/known/durationpb"
    "google.golang.org/protobuf/types/known/emptypb"
    "google.golang.org/protobuf/types/known/fieldmaskpb"
    "google.golang.org/protobuf/types/known/timestamppb"
    "google.golang.org/protobuf/types/known/wrapperspb"

    // Health checking
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"

    // Reflection
    "google.golang.org/grpc/reflection"

    // Testing
    "google.golang.org/grpc/test/bufconn"

    // Keepalive
    "google.golang.org/grpc/keepalive"

    // Compression
    _ "google.golang.org/grpc/encoding/gzip"

    // OpenTelemetry (observability)
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)
```

### What to Study Next

- **Chapter 45:** TLS and mTLS for securing gRPC connections
- **Chapter 46:** Load balancing strategies (client-side, proxy, service mesh)
- **Chapter 47:** gRPC with Kubernetes (service discovery, Envoy, Istio)
- **OpenTelemetry integration:** Distributed tracing across gRPC services
- **Buf Schema Registry:** Managing proto schemas across teams
- **Connect-Go:** An alternative gRPC-compatible framework with simpler HTTP semantics
