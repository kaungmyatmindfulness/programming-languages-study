# Chapter 45: WebSockets and Real-Time Communication

HTTP was designed as a request-response protocol: the client asks, the server answers, and the connection sits idle or closes. This model works beautifully for fetching web pages and REST APIs, but it breaks down when you need the server to push data to the client the instant something happens -- a new chat message, a stock price tick, a live dashboard update, or a multiplayer game state change. Polling the server every second wastes bandwidth and adds latency. Long polling is a hack that ties up connections. What you need is a persistent, full-duplex communication channel between client and server. That is exactly what WebSockets provide.

This chapter covers the full spectrum of real-time communication in Go. We start with the WebSocket protocol itself -- the upgrade handshake, frame structure, and opcodes. We then build progressively more sophisticated servers using both the battle-tested `gorilla/websocket` library and the modern `nhooyr.io/websocket` alternative. We implement the Hub pattern for multi-client broadcasting, add authentication, scale across multiple server instances with Redis pub/sub, and compare WebSockets against Server-Sent Events and long polling. By the end, you will have the knowledge to build production-grade real-time systems in Go.

---

## Prerequisites

Before working through this chapter, you should be comfortable with:

- **Chapter 6: Structs and Interfaces** -- WebSocket handlers rely on interface-driven design
- **Chapter 10: Goroutines and Concurrency Basics** -- Every WebSocket connection runs in its own goroutine
- **Chapter 11: Channels and Select** -- Channels coordinate message broadcasting
- **Chapter 12: JSON, HTTP, and REST APIs** -- WebSockets upgrade from HTTP connections
- **Chapter 16: Context Package** -- Connection lifecycle management uses contexts
- **Chapter 17: Advanced Concurrency** -- sync primitives protect shared state
- **Chapter 27: Authentication and Authorization** -- Securing WebSocket connections
- **Chapter 30: Graceful Shutdown** -- Draining WebSocket connections on server shutdown

---

## Table of Contents

1. [WebSocket Protocol Fundamentals](#1-websocket-protocol-fundamentals)
2. [The gorilla/websocket Library](#2-the-gorillawebsocket-library)
3. [The nhooyr.io/websocket Library](#3-the-nhooyriowebsocket-library)
4. [Building a WebSocket Echo Server](#4-building-a-websocket-echo-server)
5. [Connection Lifecycle Management](#5-connection-lifecycle-management)
6. [Handling Concurrent Reads and Writes](#6-handling-concurrent-reads-and-writes)
7. [The Hub Pattern for Multi-Client Communication](#7-the-hub-pattern-for-multi-client-communication)
8. [Building a Chat Room Server](#8-building-a-chat-room-server)
9. [Authentication with WebSockets](#9-authentication-with-websockets)
10. [Scaling WebSockets with Redis Pub/Sub](#10-scaling-websockets-with-redis-pubsub)
11. [Server-Sent Events (SSE)](#11-server-sent-events-sse)
12. [Long Polling vs WebSockets vs SSE](#12-long-polling-vs-websockets-vs-sse)
13. [Reconnection Strategies](#13-reconnection-strategies)
14. [Load Testing WebSocket Servers](#14-load-testing-websocket-servers)
15. [Production Considerations](#15-production-considerations)
16. [Real-World Example: Live Dashboard System](#16-real-world-example-live-dashboard-system)
17. [Key Takeaways](#17-key-takeaways)
18. [Practice Exercises](#18-practice-exercises)

---

## 1. WebSocket Protocol Fundamentals

### The Problem with HTTP for Real-Time

HTTP/1.1 follows a strict request-response cycle. The client sends a request, the server sends a response, and that transaction is complete. If the server has new data for the client five seconds later, it has no way to push it -- it must wait for the client to ask.

```
Traditional HTTP:

Client                    Server
  │                         │
  │── GET /messages ───────►│   Client polls
  │◄── 200 OK (2 msgs) ────│
  │                         │
  │    ... 5 seconds ...    │   New message arrives on server
  │                         │   Server cannot notify client!
  │                         │
  │── GET /messages ───────►│   Client polls again
  │◄── 200 OK (3 msgs) ────│   5 second delay before client sees it
  │                         │
```

### The WebSocket Upgrade Handshake

WebSocket connections start as regular HTTP requests. The client sends a special `Upgrade` header, and if the server agrees, the connection is "upgraded" from HTTP to the WebSocket protocol. After the upgrade, both sides can send messages at any time over the same TCP connection.

```
WebSocket Handshake:

Client                                  Server
  │                                       │
  │── GET /ws HTTP/1.1 ─────────────────►│
  │   Host: example.com                   │
  │   Upgrade: websocket                  │
  │   Connection: Upgrade                 │
  │   Sec-WebSocket-Key: dGhlIHN...       │
  │   Sec-WebSocket-Version: 13           │
  │                                       │
  │◄── HTTP/1.1 101 Switching Protocols ──│
  │    Upgrade: websocket                 │
  │    Connection: Upgrade                │
  │    Sec-WebSocket-Accept: s3pPL...     │
  │                                       │
  │◄════════ Full-Duplex Channel ════════►│
  │                                       │
  │── "Hello" ──────────────────────────►│   Client sends anytime
  │◄── "World" ──────────────────────────│   Server sends anytime
  │◄── "Update: price=42.50" ───────────│   Server pushes data
  │── "Subscribe: AAPL" ───────────────►│   Client sends anytime
  │                                       │
```

The `Sec-WebSocket-Key` is a base64-encoded random value. The server concatenates it with a magic GUID (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`), takes the SHA-1 hash, and base64-encodes the result to produce `Sec-WebSocket-Accept`. This proves the server understands the WebSocket protocol -- it is not a security mechanism.

### WebSocket Frame Structure

After the handshake, data is transmitted in **frames**. Each frame has a small header followed by payload data:

```
WebSocket Frame Format:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |           (16/64)             |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - -+-------------------------------+
|                               | Masking-key, if MASK set to 1 |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------+ - - - - - - - - - - - - - - -+
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

Key fields:

| Field | Size | Description |
|-------|------|-------------|
| FIN | 1 bit | 1 if this is the final fragment of a message |
| Opcode | 4 bits | Frame type (see table below) |
| MASK | 1 bit | 1 if payload is masked (client-to-server must be masked) |
| Payload length | 7 bits | 0-125: actual length. 126: next 2 bytes are length. 127: next 8 bytes are length |
| Masking key | 4 bytes | XOR key for unmasking payload (only if MASK=1) |
| Payload | variable | The actual message data |

### Opcodes

| Opcode | Name | Description |
|--------|------|-------------|
| 0x0 | Continuation | Continues a fragmented message |
| 0x1 | Text | UTF-8 encoded text data |
| 0x2 | Binary | Arbitrary binary data |
| 0x8 | Close | Initiates connection close |
| 0x9 | Ping | Heartbeat request (server or client) |
| 0xA | Pong | Heartbeat response |

> **Key Insight:** Text frames (0x1) must contain valid UTF-8. Binary frames (0x2) can contain anything. In Go, you typically use text frames for JSON messages and binary frames for protocol buffers or raw data.

### Connection Close Protocol

WebSocket has a graceful close handshake. Either side sends a Close frame with an optional status code and reason. The other side responds with its own Close frame, and then both sides close the TCP connection.

```
Close Handshake:

Client                              Server
  │                                   │
  │── Close (1000, "done") ─────────►│   Client initiates close
  │◄── Close (1000, "goodbye") ──────│   Server acknowledges
  │                                   │
  │    TCP connection closes          │
```

Common close codes:

| Code | Meaning |
|------|---------|
| 1000 | Normal closure |
| 1001 | Going away (e.g., server shutting down) |
| 1002 | Protocol error |
| 1003 | Unsupported data type |
| 1006 | Abnormal closure (no close frame received) |
| 1008 | Policy violation |
| 1009 | Message too big |
| 1011 | Unexpected server error |

---

## 2. The gorilla/websocket Library

The `gorilla/websocket` package has been the de facto standard for WebSockets in Go for nearly a decade. While the original Gorilla organization archived its repositories, the community has forked and continues to maintain it. It provides a complete, low-level WebSocket implementation with fine-grained control.

### Installation

```bash
go get github.com/gorilla/websocket
```

### Core Types

The library centers around two types:

1. **`websocket.Upgrader`** -- upgrades an HTTP connection to a WebSocket connection
2. **`websocket.Conn`** -- represents the WebSocket connection itself

### Basic Server Setup

```go
package main

import (
    "log"
    "net/http"

    "github.com/gorilla/websocket"
)

// Upgrader configures the upgrade from HTTP to WebSocket.
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,

    // CheckOrigin controls which origins are allowed to connect.
    // In production, always validate the origin!
    CheckOrigin: func(r *http.Request) bool {
        // For development, allow all origins.
        return true
    },
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    // Upgrade the HTTP connection to a WebSocket connection.
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade error: %v", err)
        return
    }
    defer conn.Close()

    for {
        // ReadMessage blocks until a message is received.
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                log.Printf("read error: %v", err)
            }
            break
        }

        log.Printf("received: %s", message)

        // Echo the message back.
        if err := conn.WriteMessage(messageType, message); err != nil {
            log.Printf("write error: %v", err)
            break
        }
    }
}

func main() {
    http.HandleFunc("/ws", wsHandler)
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Upgrader Configuration Options

```go
upgrader := websocket.Upgrader{
    // Buffer sizes for the connection. Larger buffers use more memory
    // but can handle larger messages without fragmentation.
    ReadBufferSize:  4096,
    WriteBufferSize: 4096,

    // HandshakeTimeout specifies the duration for the handshake to complete.
    HandshakeTimeout: 10 * time.Second,

    // Subprotocols specifies the server's supported protocols in order
    // of preference. If the client requests subprotocols, the first
    // match is selected.
    Subprotocols: []string{"chat", "binary"},

    // EnableCompression allows the server to negotiate per-message
    // compression (RFC 7692).
    EnableCompression: true,

    // CheckOrigin prevents cross-site request forgery.
    // Return true to allow the connection.
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return origin == "https://myapp.com"
    },

    // Error is called when the upgrade fails. The default writes an
    // HTTP error response.
    Error: func(w http.ResponseWriter, r *http.Request, status int, reason error) {
        log.Printf("WebSocket upgrade error: status=%d reason=%v", status, reason)
        http.Error(w, reason.Error(), status)
    },
}
```

### Reading and Writing Messages

gorilla/websocket provides two levels of API for reading and writing:

**Simple API (ReadMessage/WriteMessage):**

```go
// Read a complete message.
messageType, data, err := conn.ReadMessage()

// Write a complete message.
err = conn.WriteMessage(websocket.TextMessage, []byte("hello"))
```

**Streaming API (NextReader/NextWriter):**

```go
// Read with streaming -- useful for large messages.
messageType, reader, err := conn.NextReader()
if err != nil {
    return err
}
// reader is an io.Reader -- you can use io.Copy, json.NewDecoder, etc.
var msg ChatMessage
if err := json.NewDecoder(reader).Decode(&msg); err != nil {
    return err
}

// Write with streaming.
writer, err := conn.NextWriter(websocket.TextMessage)
if err != nil {
    return err
}
if err := json.NewEncoder(writer).Encode(response); err != nil {
    writer.Close()
    return err
}
// You MUST close the writer to flush the frame.
if err := writer.Close(); err != nil {
    return err
}
```

### JSON Helpers

gorilla/websocket has built-in JSON helpers:

```go
// Read JSON message.
var msg ChatMessage
if err := conn.ReadJSON(&msg); err != nil {
    log.Printf("json read error: %v", err)
    return
}

// Write JSON message.
response := ChatMessage{
    User:    "server",
    Content: "message received",
    Time:    time.Now(),
}
if err := conn.WriteJSON(response); err != nil {
    log.Printf("json write error: %v", err)
    return
}
```

---

## 3. The nhooyr.io/websocket Library

`nhooyr.io/websocket` (often called "nhooyr websocket") is a more modern WebSocket library that embraces Go idioms like `context.Context`, `io.Reader`/`io.Writer`, and `net/http` integration more tightly. It has a smaller API surface and handles many edge cases automatically.

### Installation

```bash
go get nhooyr.io/websocket
```

### Key Differences from gorilla/websocket

| Feature | gorilla/websocket | nhooyr.io/websocket |
|---------|-------------------|---------------------|
| Context support | No (manual) | First-class `context.Context` |
| API style | Low-level, manual | High-level, idiomatic |
| Concurrent writes | Manual synchronization | Built-in write mutex |
| Close handling | Manual close frame | Automatic negotiation |
| Compression | Opt-in | Opt-in with context-based control |
| WASM support | No | Yes (compiles to WebAssembly) |
| Maintenance | Community-maintained fork | Actively maintained |

### Basic Server with nhooyr.io/websocket

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"

    "nhooyr.io/websocket"
    "nhooyr.io/websocket/wsjson"
)

func wsHandler(w http.ResponseWriter, r *http.Request) {
    // Accept upgrades the HTTP connection to a WebSocket connection.
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        // OriginPatterns controls allowed origins using glob patterns.
        OriginPatterns: []string{"myapp.com", "*.myapp.com"},

        // Subprotocols lists supported subprotocols.
        Subprotocols: []string{"chat"},

        // CompressionMode controls per-message compression.
        CompressionMode: websocket.CompressionContextTakeover,
    })
    if err != nil {
        log.Printf("accept error: %v", err)
        return
    }
    defer conn.CloseNow()

    // Create a context that cancels when the connection closes.
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Minute)
    defer cancel()

    for {
        // Read blocks until a message arrives or the context is cancelled.
        typ, data, err := conn.Read(ctx)
        if err != nil {
            log.Printf("read error: %v", err)
            return
        }

        log.Printf("received (%v): %s", typ, data)

        // Echo it back.
        if err := conn.Write(ctx, typ, data); err != nil {
            log.Printf("write error: %v", err)
            return
        }
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/ws", wsHandler)

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }
    log.Println("Server starting on :8080")
    log.Fatal(srv.ListenAndServe())
}
```

### JSON Reading and Writing with nhooyr

```go
import "nhooyr.io/websocket/wsjson"

type Message struct {
    Type    string `json:"type"`
    Content string `json:"content"`
    From    string `json:"from"`
}

// Read JSON.
var msg Message
if err := wsjson.Read(ctx, conn, &msg); err != nil {
    return fmt.Errorf("reading json: %w", err)
}

// Write JSON.
response := Message{
    Type:    "response",
    Content: "got it",
    From:    "server",
}
if err := wsjson.Write(ctx, conn, response); err != nil {
    return fmt.Errorf("writing json: %w", err)
}
```

### Graceful Close with nhooyr

```go
// Graceful close sends a close frame and waits for acknowledgment.
conn.Close(websocket.StatusNormalClosure, "session ended")

// Immediate close -- does not wait for acknowledgment.
// Use in defer statements.
conn.CloseNow()
```

---

## 4. Building a WebSocket Echo Server

An echo server is the "Hello World" of WebSocket programming. It reads every message from the client and sends it right back. Despite its simplicity, it demonstrates the core read/write loop.

### gorilla/websocket Echo Server

```go
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin:     func(r *http.Request) bool { return true },
}

func echoHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade: %v", err)
        return
    }
    defer conn.Close()

    // Set connection limits.
    conn.SetReadLimit(512) // Max message size in bytes.
    conn.SetReadDeadline(time.Now().Add(60 * time.Second))

    // Reset the read deadline on every pong.
    conn.SetPongHandler(func(appData string) error {
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsCloseError(err,
                websocket.CloseNormalClosure,
                websocket.CloseGoingAway,
            ) {
                log.Printf("client disconnected normally")
            } else {
                log.Printf("read error: %v", err)
            }
            return
        }

        log.Printf("echo: %s", message)

        conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
        if err := conn.WriteMessage(messageType, message); err != nil {
            log.Printf("write error: %v", err)
            return
        }
    }
}

func main() {
    http.HandleFunc("/ws", echoHandler)
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        http.ServeFile(w, r, "index.html")
    })
    log.Println("Echo server on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Minimal HTML Test Client

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head><title>WebSocket Echo Test</title></head>
<body>
<h1>WebSocket Echo</h1>
<input type="text" id="msg" placeholder="Type a message..." />
<button onclick="send()">Send</button>
<pre id="log"></pre>
<script>
    const ws = new WebSocket("ws://" + window.location.host + "/ws");
    const logEl = document.getElementById("log");

    ws.onopen = () => log("Connected");
    ws.onclose = (e) => log(`Disconnected: code=${e.code} reason=${e.reason}`);
    ws.onerror = (e) => log("Error: " + e);
    ws.onmessage = (e) => log("Received: " + e.data);

    function send() {
        const msg = document.getElementById("msg").value;
        ws.send(msg);
        log("Sent: " + msg);
    }

    function log(msg) {
        logEl.textContent += msg + "\n";
    }
</script>
</body>
</html>
```

### nhooyr.io/websocket Echo Server

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"

    "nhooyr.io/websocket"
)

func echoHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns: []string{"*"},
    })
    if err != nil {
        log.Printf("accept: %v", err)
        return
    }
    defer conn.CloseNow()

    // Limit message size to prevent abuse.
    conn.SetReadLimit(512)

    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Minute)
    defer cancel()

    for {
        typ, data, err := conn.Read(ctx)
        if err != nil {
            log.Printf("read: %v", err)
            return
        }

        if err := conn.Write(ctx, typ, data); err != nil {
            log.Printf("write: %v", err)
            return
        }
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/ws", echoHandler)
    log.Println("nhooyr echo server on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 5. Connection Lifecycle Management

Managing WebSocket connection lifecycles correctly is critical. A WebSocket connection can stay open for hours or days. You need to handle timeouts, detect dead connections, and clean up resources.

### The Connection Lifecycle

```
Connection Lifecycle:

  ┌──────────┐
  │  HTTP     │ Client sends HTTP Upgrade request
  │  Request  │
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  Upgrade  │ Server upgrades to WebSocket
  │  Handshake│
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  Open     │ Connection established
  │           │ Start read/write loops
  └────┬─────┘
       │
       ▼
  ┌──────────────────────────────────┐
  │  Active Communication            │
  │                                  │
  │  ┌─────────┐    ┌─────────┐     │
  │  │  Read    │    │  Write  │     │
  │  │  Loop    │    │  Loop   │     │
  │  └────┬────┘    └────┬────┘     │
  │       │              │          │
  │  Ping/Pong heartbeats every N   │
  │  seconds to detect dead peers   │
  │                                  │
  └────────────┬─────────────────────┘
               │
               ▼
  ┌──────────────────┐
  │  Close            │
  │  - Client close   │  Client sends Close frame
  │  - Server close   │  Server sends Close frame
  │  - Network error  │  TCP connection drops
  │  - Timeout        │  Read/write deadline exceeded
  └──────────────────┘
```

### Ping/Pong Heartbeats

WebSocket has built-in ping/pong frames for keep-alive. The server sends a Ping, and the client automatically responds with a Pong. If no Pong arrives within the deadline, the connection is considered dead.

```go
const (
    // Time allowed to write a message to the peer.
    writeWait = 10 * time.Second

    // Time allowed to read the next pong message from the peer.
    pongWait = 60 * time.Second

    // Send pings to peer with this period. Must be less than pongWait.
    pingPeriod = (pongWait * 9) / 10 // 54 seconds

    // Maximum message size allowed from peer.
    maxMessageSize = 8192
)

func serveWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade: %v", err)
        return
    }

    // Configure connection.
    conn.SetReadLimit(maxMessageSize)
    conn.SetReadDeadline(time.Now().Add(pongWait))
    conn.SetPongHandler(func(string) error {
        // Client responded to our ping -- reset the read deadline.
        conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    // Start a goroutine to send pings.
    done := make(chan struct{})
    go func() {
        ticker := time.NewTicker(pingPeriod)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                conn.SetWriteDeadline(time.Now().Add(writeWait))
                if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                    return // Connection is dead.
                }
            case <-done:
                return
            }
        }
    }()

    // Read loop.
    defer func() {
        close(done) // Stop the ping goroutine.
        conn.Close()
    }()

    for {
        _, message, err := conn.ReadMessage()
        if err != nil {
            break
        }
        log.Printf("message: %s", message)
    }
}
```

### Handling Close Frames

```go
// Set a close handler that logs the close reason.
conn.SetCloseHandler(func(code int, text string) error {
    log.Printf("close received: code=%d text=%s", code, text)

    // Send back a close frame.
    message := websocket.FormatCloseMessage(code, "")
    conn.WriteControl(
        websocket.CloseMessage,
        message,
        time.Now().Add(writeWait),
    )
    return nil
})
```

### Detecting Unexpected Disconnections

When a client disconnects without sending a Close frame (browser tab closed, network failure), the next read returns an error. Use `IsUnexpectedCloseError` to distinguish normal from abnormal disconnections:

```go
for {
    _, message, err := conn.ReadMessage()
    if err != nil {
        if websocket.IsUnexpectedCloseError(err,
            websocket.CloseGoingAway,
            websocket.CloseNormalClosure,
        ) {
            // This is an actual error -- log it.
            log.Printf("unexpected close: %v", err)
        }
        // Normal close or expected error -- just return.
        return
    }
    processMessage(message)
}
```

---

## 6. Handling Concurrent Reads and Writes

> **Warning:** In gorilla/websocket, a `*websocket.Conn` supports **one concurrent reader and one concurrent writer**. Multiple goroutines must NOT call `WriteMessage` simultaneously. Multiple goroutines must NOT call `ReadMessage` simultaneously. You CAN have one goroutine reading and a different goroutine writing at the same time.

This is the single most common source of bugs in Go WebSocket code.

### The Problem

```go
// WRONG: Multiple goroutines writing simultaneously.
func handleConnection(conn *websocket.Conn) {
    // Goroutine 1 writes.
    go func() {
        for msg := range broadcastCh {
            conn.WriteMessage(websocket.TextMessage, msg) // DATA RACE!
        }
    }()

    // Goroutine 2 writes (e.g., pings).
    go func() {
        for range time.Tick(30 * time.Second) {
            conn.WriteMessage(websocket.PingMessage, nil) // DATA RACE!
        }
    }()
}
```

### Solution 1: Dedicated Write Goroutine with Channel

The standard pattern is to have a single goroutine own all writes. Other goroutines send messages through a channel.

```go
type Client struct {
    conn *websocket.Conn
    send chan []byte
}

func NewClient(conn *websocket.Conn) *Client {
    return &Client{
        conn: conn,
        send: make(chan []byte, 256), // Buffered to prevent blocking senders.
    }
}

// writePump is the single goroutine that writes to the connection.
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // The channel was closed -- send a close frame.
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            // Get a writer for the next message.
            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Drain any queued messages into the same frame
            // to reduce syscall overhead.
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte("\n"))
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// readPump is the single goroutine that reads from the connection.
func (c *Client) readPump(handler func([]byte)) {
    defer c.conn.Close()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        handler(message)
    }
}
```

### Solution 2: Write Mutex

If you prefer not to use channels, protect writes with a mutex:

```go
type SafeConn struct {
    conn    *websocket.Conn
    writeMu sync.Mutex
}

func (sc *SafeConn) WriteMessage(messageType int, data []byte) error {
    sc.writeMu.Lock()
    defer sc.writeMu.Unlock()
    return sc.conn.WriteMessage(messageType, data)
}

func (sc *SafeConn) WriteJSON(v interface{}) error {
    sc.writeMu.Lock()
    defer sc.writeMu.Unlock()
    return sc.conn.WriteJSON(v)
}

func (sc *SafeConn) WritePing() error {
    sc.writeMu.Lock()
    defer sc.writeMu.Unlock()
    return sc.conn.WriteMessage(websocket.PingMessage, nil)
}
```

> **Tip:** The channel-based approach (Solution 1) is preferred in production because it naturally provides backpressure -- if the send channel fills up, you know the client is slow and can disconnect it. The mutex approach can block fast senders while a slow write completes.

### nhooyr.io/websocket Concurrency

nhooyr.io/websocket handles concurrent writes internally with a mutex, so you do not need to synchronize writes yourself:

```go
// nhooyr.io/websocket: concurrent writes are safe out of the box.
go func() {
    for msg := range broadcastCh {
        conn.Write(ctx, websocket.MessageText, msg) // Safe!
    }
}()
go func() {
    // Pings are handled automatically by nhooyr.io/websocket.
    // No manual ping goroutine needed.
}()
```

However, concurrent reads are still NOT safe -- only one goroutine should call `Read` at a time.

---

## 7. The Hub Pattern for Multi-Client Communication

The Hub (or Broker) pattern is the backbone of any multi-client WebSocket application. A single Hub goroutine coordinates all client connections, handles registration/unregistration, and broadcasts messages.

```
Hub Pattern Architecture:

                    ┌──────────────────────────┐
                    │          Hub              │
                    │                           │
  ┌─────────┐      │  register    ┌─────────┐ │
  │ Client A ├─────►├────────────►│         │ │
  └─────────┘      │              │ clients │ │
                    │  unregister  │   map   │ │
  ┌─────────┐      │◄────────────┤         │ │
  │ Client B ├─────►├            │└─────────┘ │
  └─────────┘      │              │           │
                    │  broadcast   │           │
  ┌─────────┐      │─────────────►│ sends to │ │
  │ Client C ├─────►├             │ all      │ │
  └─────────┘      │              │ clients  │ │
                    │              │           │
                    └──────────────────────────┘
```

### Hub Implementation

```go
package main

import (
    "log"
    "sync"
)

// Hub maintains the set of active clients and broadcasts messages.
type Hub struct {
    // Registered clients.
    clients map[*Client]bool

    // Inbound messages from clients to broadcast.
    broadcast chan []byte

    // Register requests from clients.
    register chan *Client

    // Unregister requests from clients.
    unregister chan *Client

    mu sync.RWMutex
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

// Run starts the hub's event loop. This must run in its own goroutine.
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
            log.Printf("client registered, total: %d", len(h.clients))

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
                log.Printf("client unregistered, total: %d", len(h.clients))
            }

        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                    // Message queued successfully.
                default:
                    // Client's send buffer is full -- disconnect it.
                    close(client.send)
                    delete(h.clients, client)
                    log.Printf("slow client removed, total: %d", len(h.clients))
                }
            }
        }
    }
}

// ClientCount returns the number of connected clients.
func (h *Hub) ClientCount() int {
    h.mu.RLock()
    defer h.mu.RUnlock()
    return len(h.clients)
}
```

### Client Implementation (Integrated with Hub)

```go
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

const (
    writeWait  = 10 * time.Second
    pongWait   = 60 * time.Second
    pingPeriod = (pongWait * 9) / 10
    maxMsgSize = 8192
)

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMsgSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                log.Printf("read error: %v", err)
            }
            break
        }

        // Broadcast the message to all clients.
        c.hub.broadcast <- message
    }
}

func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Batch queued messages.
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte("\n"))
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// serveWS handles new WebSocket connections.
func serveWS(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade: %v", err)
        return
    }

    client := &Client{
        hub:  hub,
        conn: conn,
        send: make(chan []byte, 256),
    }
    hub.register <- client

    // Start the read and write pumps in separate goroutines.
    // All sends to this connection go through writePump.
    go client.writePump()
    go client.readPump()
}
```

### Wiring It Together

```go
func main() {
    hub := NewHub()
    go hub.Run()

    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        serveWS(hub, w, r)
    })

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        http.ServeFile(w, r, "index.html")
    })

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 8. Building a Chat Room Server

Let us build a full-featured chat server with named rooms, user identities, and structured JSON messages.

### Message Types

```go
package main

import "time"

// MessageType represents different kinds of messages in the system.
type MessageType string

const (
    MsgTypeChat     MessageType = "chat"
    MsgTypeJoin     MessageType = "join"
    MsgTypeLeave    MessageType = "leave"
    MsgTypeSystem   MessageType = "system"
    MsgTypeRoomList MessageType = "room_list"
    MsgTypeError    MessageType = "error"
)

// IncomingMessage is what the client sends to the server.
type IncomingMessage struct {
    Type    MessageType `json:"type"`
    Content string      `json:"content"`
    Room    string      `json:"room,omitempty"`
}

// OutgoingMessage is what the server sends to the client.
type OutgoingMessage struct {
    Type      MessageType `json:"type"`
    Content   string      `json:"content"`
    From      string      `json:"from"`
    Room      string      `json:"room"`
    Timestamp time.Time   `json:"timestamp"`
    Online    int         `json:"online,omitempty"`
}
```

### Room Implementation

```go
package main

import (
    "log"
    "sync"
)

// Room represents a chat room that clients can join.
type Room struct {
    name    string
    clients map[*ChatClient]bool
    mu      sync.RWMutex

    join      chan *ChatClient
    leave     chan *ChatClient
    broadcast chan OutgoingMessage
    done      chan struct{}
}

func NewRoom(name string) *Room {
    return &Room{
        name:      name,
        clients:   make(map[*ChatClient]bool),
        join:      make(chan *ChatClient),
        leave:     make(chan *ChatClient),
        broadcast: make(chan OutgoingMessage, 256),
        done:      make(chan struct{}),
    }
}

func (r *Room) Run() {
    for {
        select {
        case client := <-r.join:
            r.mu.Lock()
            r.clients[client] = true
            count := len(r.clients)
            r.mu.Unlock()

            log.Printf("[%s] %s joined (%d online)", r.name, client.username, count)

            // Notify everyone that a new user joined.
            r.broadcast <- OutgoingMessage{
                Type:    MsgTypeJoin,
                Content: client.username + " joined the room",
                From:    "system",
                Room:    r.name,
                Online:  count,
            }

        case client := <-r.leave:
            r.mu.Lock()
            if _, ok := r.clients[client]; ok {
                delete(r.clients, client)
            }
            count := len(r.clients)
            r.mu.Unlock()

            log.Printf("[%s] %s left (%d online)", r.name, client.username, count)

            r.broadcast <- OutgoingMessage{
                Type:    MsgTypeLeave,
                Content: client.username + " left the room",
                From:    "system",
                Room:    r.name,
                Online:  count,
            }

        case msg := <-r.broadcast:
            r.mu.RLock()
            for client := range r.clients {
                select {
                case client.send <- msg:
                default:
                    // Client is slow -- mark for removal.
                    go func(c *ChatClient) {
                        r.leave <- c
                    }(client)
                }
            }
            r.mu.RUnlock()

        case <-r.done:
            return
        }
    }
}

func (r *Room) Stop() {
    close(r.done)
}

func (r *Room) ClientCount() int {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return len(r.clients)
}
```

### Room Manager

```go
package main

import "sync"

// RoomManager manages all chat rooms.
type RoomManager struct {
    rooms map[string]*Room
    mu    sync.RWMutex
}

func NewRoomManager() *RoomManager {
    return &RoomManager{
        rooms: make(map[string]*Room),
    }
}

// GetOrCreate returns an existing room or creates a new one.
func (rm *RoomManager) GetOrCreate(name string) *Room {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    if room, ok := rm.rooms[name]; ok {
        return room
    }

    room := NewRoom(name)
    rm.rooms[name] = room
    go room.Run()
    return room
}

// List returns all active room names.
func (rm *RoomManager) List() []string {
    rm.mu.RLock()
    defer rm.mu.RUnlock()

    names := make([]string, 0, len(rm.rooms))
    for name := range rm.rooms {
        names = append(names, name)
    }
    return names
}

// Remove removes a room if it is empty.
func (rm *RoomManager) Remove(name string) {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    if room, ok := rm.rooms[name]; ok {
        if room.ClientCount() == 0 {
            room.Stop()
            delete(rm.rooms, name)
        }
    }
}
```

### Chat Client

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

type ChatClient struct {
    username    string
    conn        *websocket.Conn
    send        chan OutgoingMessage
    currentRoom *Room
    roomManager *RoomManager
}

func NewChatClient(
    username string,
    conn *websocket.Conn,
    rm *RoomManager,
) *ChatClient {
    return &ChatClient{
        username:    username,
        conn:        conn,
        send:        make(chan OutgoingMessage, 64),
        roomManager: rm,
    }
}

func (c *ChatClient) readPump() {
    defer func() {
        if c.currentRoom != nil {
            c.currentRoom.leave <- c
        }
        c.conn.Close()
    }()

    c.conn.SetReadLimit(4096)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, raw, err := c.conn.ReadMessage()
        if err != nil {
            break
        }

        var msg IncomingMessage
        if err := json.Unmarshal(raw, &msg); err != nil {
            c.sendError("invalid message format")
            continue
        }

        c.handleMessage(msg)
    }
}

func (c *ChatClient) handleMessage(msg IncomingMessage) {
    switch msg.Type {
    case MsgTypeChat:
        if c.currentRoom == nil {
            c.sendError("you must join a room first")
            return
        }
        c.currentRoom.broadcast <- OutgoingMessage{
            Type:      MsgTypeChat,
            Content:   msg.Content,
            From:      c.username,
            Room:      c.currentRoom.name,
            Timestamp: time.Now(),
        }

    case MsgTypeJoin:
        if msg.Room == "" {
            c.sendError("room name is required")
            return
        }

        // Leave current room if in one.
        if c.currentRoom != nil {
            c.currentRoom.leave <- c
        }

        // Join the new room.
        room := c.roomManager.GetOrCreate(msg.Room)
        c.currentRoom = room
        room.join <- c

    case MsgTypeRoomList:
        rooms := c.roomManager.List()
        content, _ := json.Marshal(rooms)
        c.send <- OutgoingMessage{
            Type:      MsgTypeRoomList,
            Content:   string(content),
            From:      "system",
            Timestamp: time.Now(),
        }

    default:
        c.sendError("unknown message type: " + string(msg.Type))
    }
}

func (c *ChatClient) sendError(text string) {
    c.send <- OutgoingMessage{
        Type:      MsgTypeError,
        Content:   text,
        From:      "system",
        Timestamp: time.Now(),
    }
}

func (c *ChatClient) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.conn.WriteJSON(msg); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

### Chat Server Main

```go
package main

import (
    "log"
    "net/http"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin:     func(r *http.Request) bool { return true },
}

func main() {
    rm := NewRoomManager()

    // Create a default room.
    rm.GetOrCreate("general")

    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        // Get username from query parameter (in production, use auth).
        username := r.URL.Query().Get("username")
        if username == "" {
            http.Error(w, "username required", http.StatusBadRequest)
            return
        }

        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("upgrade: %v", err)
            return
        }

        client := NewChatClient(username, conn, rm)
        go client.writePump()
        go client.readPump()
    })

    log.Println("Chat server on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Chat Client HTML

```html
<!DOCTYPE html>
<html>
<head><title>Go Chat</title></head>
<body>
<div id="login">
    <input type="text" id="username" placeholder="Username" />
    <button onclick="connect()">Connect</button>
</div>
<div id="chat" style="display:none;">
    <div>
        Room: <input type="text" id="roomName" value="general" />
        <button onclick="joinRoom()">Join Room</button>
    </div>
    <pre id="messages" style="height:400px;overflow:auto;border:1px solid #ccc;padding:10px;"></pre>
    <input type="text" id="msgInput" placeholder="Type a message..."
           onkeydown="if(event.key==='Enter')sendMsg()" />
    <button onclick="sendMsg()">Send</button>
</div>
<script>
let ws;
function connect() {
    const username = document.getElementById("username").value;
    if (!username) return alert("Enter a username");

    ws = new WebSocket(`ws://${location.host}/ws?username=${username}`);
    ws.onopen = () => {
        document.getElementById("login").style.display = "none";
        document.getElementById("chat").style.display = "block";
        joinRoom(); // Auto-join default room.
    };
    ws.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        const el = document.getElementById("messages");
        const time = new Date(msg.timestamp).toLocaleTimeString();
        el.textContent += `[${time}] ${msg.from}: ${msg.content}\n`;
        el.scrollTop = el.scrollHeight;
    };
    ws.onclose = () => appendMsg("Disconnected");
}

function joinRoom() {
    const room = document.getElementById("roomName").value;
    ws.send(JSON.stringify({type: "join", room: room}));
}

function sendMsg() {
    const input = document.getElementById("msgInput");
    ws.send(JSON.stringify({type: "chat", content: input.value}));
    input.value = "";
}

function appendMsg(text) {
    document.getElementById("messages").textContent += text + "\n";
}
</script>
</body>
</html>
```

---

## 9. Authentication with WebSockets

WebSocket connections start as HTTP requests, which means you can authenticate during the upgrade handshake using the same mechanisms you use for HTTP: cookies, bearer tokens, or query parameters.

### Strategy Comparison

| Strategy | Approach | Pros | Cons |
|----------|----------|------|------|
| Query parameter token | `ws://host/ws?token=xxx` | Simple | Token visible in logs and URL |
| Cookie-based | Session cookie sent on upgrade | Automatic, standard | Requires same-origin or CORS |
| First-message auth | Client sends auth message after connect | No URL exposure | Brief unauthenticated window |
| Custom header | Subprotocol or header-based | Clean | JavaScript WebSocket API cannot set arbitrary headers |

### Token-Based Authentication (Query Parameter)

```go
package main

import (
    "errors"
    "log"
    "net/http"
    "strings"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/websocket"
)

var jwtSecret = []byte("your-secret-key-change-in-production")

// Claims represents JWT claims.
type Claims struct {
    UserID   string `json:"user_id"`
    Username string `json:"username"`
    jwt.RegisteredClaims
}

// validateToken parses and validates a JWT token.
func validateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, errors.New("unexpected signing method")
            }
            return jwtSecret, nil
        },
    )
    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }
    return claims, nil
}

// generateToken creates a JWT for testing.
func generateToken(userID, username string) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    // Extract token from query parameter.
    token := r.URL.Query().Get("token")
    if token == "" {
        // Also check Authorization header (for non-browser clients).
        auth := r.Header.Get("Authorization")
        if strings.HasPrefix(auth, "Bearer ") {
            token = strings.TrimPrefix(auth, "Bearer ")
        }
    }

    if token == "" {
        http.Error(w, "authentication required", http.StatusUnauthorized)
        return
    }

    // Validate the token BEFORE upgrading.
    claims, err := validateToken(token)
    if err != nil {
        http.Error(w, "invalid token: "+err.Error(), http.StatusUnauthorized)
        return
    }

    // Token is valid -- upgrade the connection.
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade: %v", err)
        return
    }
    defer conn.Close()

    log.Printf("authenticated user: %s (ID: %s)", claims.Username, claims.UserID)

    // Now handle the WebSocket connection with the authenticated user.
    for {
        _, msg, err := conn.ReadMessage()
        if err != nil {
            break
        }
        log.Printf("[%s] %s", claims.Username, msg)
    }
}

func main() {
    // Token generation endpoint (for testing).
    http.HandleFunc("/token", func(w http.ResponseWriter, r *http.Request) {
        username := r.URL.Query().Get("username")
        if username == "" {
            http.Error(w, "username required", http.StatusBadRequest)
            return
        }

        token, err := generateToken("user-123", username)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }

        w.Header().Set("Content-Type", "text/plain")
        w.Write([]byte(token))
    })

    http.HandleFunc("/ws", wsHandler)
    log.Println("Auth server on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Cookie-Based Authentication

```go
func wsHandlerWithCookie(w http.ResponseWriter, r *http.Request) {
    // Read the session cookie.
    cookie, err := r.Cookie("session_token")
    if err != nil {
        http.Error(w, "no session cookie", http.StatusUnauthorized)
        return
    }

    // Validate the session (look up in database, Redis, etc.).
    session, err := sessionStore.Get(cookie.Value)
    if err != nil {
        http.Error(w, "invalid session", http.StatusUnauthorized)
        return
    }

    // Upgrade after authentication.
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }
    defer conn.Close()

    log.Printf("cookie-authenticated user: %s", session.Username)

    // Handle connection...
}
```

### First-Message Authentication Pattern

In some cases, the client cannot pass authentication during the upgrade handshake (e.g., when using a JavaScript WebSocket API that cannot set custom headers). The client connects first, then sends an authentication message:

```go
func wsHandlerFirstMessage(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }
    defer conn.Close()

    // Set a short deadline for the auth message.
    conn.SetReadDeadline(time.Now().Add(10 * time.Second))

    // Read the first message -- it must be an auth message.
    var authMsg struct {
        Type  string `json:"type"`
        Token string `json:"token"`
    }
    if err := conn.ReadJSON(&authMsg); err != nil {
        conn.WriteJSON(map[string]string{
            "type":  "error",
            "error": "expected auth message",
        })
        return
    }

    if authMsg.Type != "auth" {
        conn.WriteJSON(map[string]string{
            "type":  "error",
            "error": "first message must be auth",
        })
        return
    }

    claims, err := validateToken(authMsg.Token)
    if err != nil {
        conn.WriteJSON(map[string]string{
            "type":  "error",
            "error": "authentication failed",
        })
        return
    }

    // Auth successful -- notify the client.
    conn.WriteJSON(map[string]interface{}{
        "type":     "auth_success",
        "username": claims.Username,
    })

    // Reset deadline for normal operation.
    conn.SetReadDeadline(time.Now().Add(pongWait))

    // Now handle authenticated messages.
    for {
        _, msg, err := conn.ReadMessage()
        if err != nil {
            break
        }
        log.Printf("[%s] %s", claims.Username, msg)
    }
}
```

> **Warning:** With first-message auth, the connection is briefly unauthenticated. Make sure no sensitive data is sent before the auth message is validated. The short read deadline (10 seconds) ensures that unauthenticated connections are cleaned up quickly.

---

## 10. Scaling WebSockets with Redis Pub/Sub

A single WebSocket server can handle thousands of connections. But when you run multiple server instances behind a load balancer, a message sent to one server must reach clients connected to other servers. Redis Pub/Sub solves this by providing a cross-instance message bus.

```
Scaling Architecture:

  ┌───────────────────────────────────────────────────┐
  │                  Load Balancer                     │
  └──────┬────────────────┬────────────────┬──────────┘
         │                │                │
    ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
    │ Server 1│      │ Server 2│      │ Server 3│
    │         │      │         │      │         │
    │ Client A│      │ Client C│      │ Client E│
    │ Client B│      │ Client D│      │ Client F│
    └────┬────┘      └────┬────┘      └────┬────┘
         │                │                │
         └────────┬───────┴────────┬───────┘
                  │                │
             ┌────▼────────────────▼────┐
             │       Redis Pub/Sub      │
             │                          │
             │  Channel: "chat:general" │
             │  Channel: "chat:random"  │
             └──────────────────────────┘

When Client A sends a message:
1. Server 1 receives it via WebSocket
2. Server 1 publishes to Redis channel "chat:general"
3. Redis delivers to all subscribers (Servers 1, 2, 3)
4. Each server broadcasts to its local clients
5. Clients B, C, D, E, F all receive the message
```

### Redis-Backed Hub

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "sync"

    "github.com/redis/go-redis/v9"
)

// RedisHub extends the Hub pattern with Redis pub/sub for multi-instance support.
type RedisHub struct {
    rdb     *redis.Client
    clients map[*Client]bool
    mu      sync.RWMutex

    register   chan *Client
    unregister chan *Client
    publish    chan OutgoingMessage

    ctx    context.Context
    cancel context.CancelFunc
}

func NewRedisHub(redisAddr string) *RedisHub {
    rdb := redis.NewClient(&redis.Options{
        Addr: redisAddr,
    })

    ctx, cancel := context.WithCancel(context.Background())

    return &RedisHub{
        rdb:        rdb,
        clients:    make(map[*Client]bool),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        publish:    make(chan OutgoingMessage, 256),
        ctx:        ctx,
        cancel:     cancel,
    }
}

// channelName returns the Redis channel name for a room.
func channelName(room string) string {
    return fmt.Sprintf("ws:room:%s", room)
}

// Run starts the hub's main event loop.
func (h *RedisHub) Run() {
    go h.subscribeLoop()

    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            log.Printf("registered, total=%d", len(h.clients))

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()

        case msg := <-h.publish:
            // Publish to Redis so all server instances receive it.
            data, err := json.Marshal(msg)
            if err != nil {
                log.Printf("marshal error: %v", err)
                continue
            }
            channel := channelName(msg.Room)
            if err := h.rdb.Publish(h.ctx, channel, data).Err(); err != nil {
                log.Printf("redis publish error: %v", err)
            }

        case <-h.ctx.Done():
            return
        }
    }
}

// subscribeLoop subscribes to Redis channels and delivers messages to local clients.
func (h *RedisHub) subscribeLoop() {
    // Subscribe to all room channels using pattern subscribe.
    pubsub := h.rdb.PSubscribe(h.ctx, "ws:room:*")
    defer pubsub.Close()

    ch := pubsub.Channel()
    for msg := range ch {
        var outMsg OutgoingMessage
        if err := json.Unmarshal([]byte(msg.Payload), &outMsg); err != nil {
            log.Printf("unmarshal error: %v", err)
            continue
        }

        // Deliver to all local clients in the room.
        h.mu.RLock()
        for client := range h.clients {
            if client.currentRoom == outMsg.Room {
                select {
                case client.send <- []byte(msg.Payload):
                default:
                    // Slow client -- will be cleaned up.
                }
            }
        }
        h.mu.RUnlock()
    }
}

// Publish sends a message through Redis to all instances.
func (h *RedisHub) Publish(msg OutgoingMessage) {
    h.publish <- msg
}

// Stop gracefully shuts down the hub.
func (h *RedisHub) Stop() {
    h.cancel()
    h.rdb.Close()
}
```

### Client Adapted for Redis Hub

```go
type Client struct {
    hub         *RedisHub
    conn        *websocket.Conn
    send        chan []byte
    username    string
    currentRoom string
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMsgSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, raw, err := c.conn.ReadMessage()
        if err != nil {
            break
        }

        var msg IncomingMessage
        if err := json.Unmarshal(raw, &msg); err != nil {
            continue
        }

        switch msg.Type {
        case MsgTypeChat:
            // Publish through Redis -- all instances will receive it.
            c.hub.Publish(OutgoingMessage{
                Type:      MsgTypeChat,
                Content:   msg.Content,
                From:      c.username,
                Room:      c.currentRoom,
                Timestamp: time.Now(),
            })

        case MsgTypeJoin:
            c.currentRoom = msg.Room
            c.hub.Publish(OutgoingMessage{
                Type:      MsgTypeJoin,
                Content:   c.username + " joined",
                From:      "system",
                Room:      msg.Room,
                Timestamp: time.Now(),
            })
        }
    }
}
```

### Connection Tracking Across Instances

To know how many users are online across all instances, use Redis:

```go
// TrackConnection adds the user to a Redis set for the room.
func (h *RedisHub) TrackConnection(room, userID string) error {
    key := fmt.Sprintf("ws:online:%s", room)
    return h.rdb.SAdd(h.ctx, key, userID).Err()
}

// UntrackConnection removes the user from the Redis set.
func (h *RedisHub) UntrackConnection(room, userID string) error {
    key := fmt.Sprintf("ws:online:%s", room)
    return h.rdb.SRem(h.ctx, key, userID).Err()
}

// OnlineCount returns the number of unique users in a room.
func (h *RedisHub) OnlineCount(room string) (int64, error) {
    key := fmt.Sprintf("ws:online:%s", room)
    return h.rdb.SCard(h.ctx, key).Result()
}

// OnlineUsers returns the list of user IDs online in a room.
func (h *RedisHub) OnlineUsers(room string) ([]string, error) {
    key := fmt.Sprintf("ws:online:%s", room)
    return h.rdb.SMembers(h.ctx, key).Result()
}
```

> **Tip:** Use Redis `SETEX` or add expiry to the online set entries, or use Redis Streams with consumer groups for more robust cross-instance state. A heartbeat mechanism (each server periodically refreshes its entries) prevents stale entries when a server crashes without cleanup.

---

## 11. Server-Sent Events (SSE)

Server-Sent Events is a simpler alternative to WebSockets when you only need **server-to-client** communication. SSE uses plain HTTP with a `text/event-stream` content type. The connection stays open, and the server pushes events as they occur.

### SSE vs WebSocket

```
SSE (unidirectional):

Client ──── GET /events ────► Server
       ◄──── event: data ─────
       ◄──── event: data ─────
       ◄──── event: data ─────

WebSocket (bidirectional):

Client ◄════════════════════► Server
       ── message ──────────►
       ◄── message ──────────
       ── message ──────────►
```

### SSE Protocol Format

SSE messages follow a simple text format:

```
event: message
data: {"user": "alice", "text": "hello"}
id: 42

event: userJoined
data: {"user": "bob"}
id: 43

: this is a comment (ignored by the client)

retry: 5000
```

Each field:
- `event:` -- The event type (default: "message")
- `data:` -- The payload (can span multiple lines)
- `id:` -- Event ID (used for reconnection: the client sends `Last-Event-ID` on reconnect)
- `retry:` -- Reconnection interval in milliseconds

### SSE Server in Go

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// SSEBroker manages SSE client connections and broadcasts events.
type SSEBroker struct {
    clients    map[chan string]bool
    register   chan chan string
    unregister chan chan string
    broadcast  chan string
}

func NewSSEBroker() *SSEBroker {
    return &SSEBroker{
        clients:    make(map[chan string]bool),
        register:   make(chan chan string),
        unregister: make(chan chan string),
        broadcast:  make(chan string, 256),
    }
}

func (b *SSEBroker) Run() {
    for {
        select {
        case client := <-b.register:
            b.clients[client] = true
            log.Printf("SSE client registered, total: %d", len(b.clients))

        case client := <-b.unregister:
            if _, ok := b.clients[client]; ok {
                delete(b.clients, client)
                close(client)
                log.Printf("SSE client unregistered, total: %d", len(b.clients))
            }

        case event := <-b.broadcast:
            for client := range b.clients {
                select {
                case client <- event:
                default:
                    // Slow client -- remove it.
                    delete(b.clients, client)
                    close(client)
                }
            }
        }
    }
}

// ServeHTTP handles SSE connections.
func (b *SSEBroker) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Check that the client supports streaming.
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    // Set SSE headers.
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")

    // Create a channel for this client.
    messageChan := make(chan string, 64)
    b.register <- messageChan

    // Clean up when the client disconnects.
    defer func() {
        b.unregister <- messageChan
    }()

    // Use the request context to detect client disconnection.
    ctx := r.Context()

    // Send events as they arrive.
    for {
        select {
        case msg, ok := <-messageChan:
            if !ok {
                return
            }
            // Write the SSE event.
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()

        case <-ctx.Done():
            // Client disconnected.
            return
        }
    }
}

func main() {
    broker := NewSSEBroker()
    go broker.Run()

    // SSE endpoint.
    http.Handle("/events", broker)

    // Endpoint to send events (for testing).
    http.HandleFunc("/send", func(w http.ResponseWriter, r *http.Request) {
        msg := r.URL.Query().Get("msg")
        if msg == "" {
            msg = "hello at " + time.Now().Format(time.RFC3339)
        }
        broker.broadcast <- msg
        w.Write([]byte("sent"))
    })

    // Serve a test page.
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprint(w, `
<!DOCTYPE html>
<html>
<body>
<h1>SSE Demo</h1>
<pre id="log"></pre>
<script>
const es = new EventSource("/events");
es.onmessage = (e) => {
    document.getElementById("log").textContent += e.data + "\n";
};
es.onerror = () => {
    document.getElementById("log").textContent += "Connection error\n";
};
</script>
</body>
</html>`)
    })

    log.Println("SSE server on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### SSE with Named Events and IDs

```go
// SSEEvent represents a full SSE event with optional fields.
type SSEEvent struct {
    ID    string
    Event string
    Data  string
    Retry int // milliseconds
}

// WriteSSE writes a formatted SSE event to a ResponseWriter.
func WriteSSE(w http.ResponseWriter, event SSEEvent) {
    if event.ID != "" {
        fmt.Fprintf(w, "id: %s\n", event.ID)
    }
    if event.Event != "" {
        fmt.Fprintf(w, "event: %s\n", event.Event)
    }
    if event.Retry > 0 {
        fmt.Fprintf(w, "retry: %d\n", event.Retry)
    }
    fmt.Fprintf(w, "data: %s\n\n", event.Data)
}

// Usage in handler:
func sseHandler(w http.ResponseWriter, r *http.Request) {
    flusher := w.(http.Flusher)
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")

    // Tell the client to reconnect after 3 seconds if disconnected.
    WriteSSE(w, SSEEvent{Retry: 3000})
    flusher.Flush()

    eventID := 0
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            eventID++
            WriteSSE(w, SSEEvent{
                ID:    fmt.Sprintf("%d", eventID),
                Event: "tick",
                Data:  fmt.Sprintf(`{"time": "%s", "seq": %d}`, time.Now().Format(time.RFC3339), eventID),
            })
            flusher.Flush()

        case <-r.Context().Done():
            return
        }
    }
}
```

### SSE Reconnection with Last-Event-ID

The browser automatically reconnects SSE connections and sends the `Last-Event-ID` header. Your server can use this to replay missed events:

```go
func sseWithReplayHandler(w http.ResponseWriter, r *http.Request) {
    flusher := w.(http.Flusher)
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")

    // Check if the client is reconnecting.
    lastID := r.Header.Get("Last-Event-ID")
    if lastID != "" {
        log.Printf("client reconnecting from event ID: %s", lastID)

        // Replay missed events from your event store.
        missedEvents, err := eventStore.GetAfter(lastID)
        if err != nil {
            log.Printf("replay error: %v", err)
        }

        for _, event := range missedEvents {
            WriteSSE(w, event)
        }
        flusher.Flush()
    }

    // Continue with live events...
}
```

---

## 12. Long Polling vs WebSockets vs SSE

### Long Polling

Long polling is a workaround for real-time communication over plain HTTP. The client sends a request, and the server holds it open until data is available (or a timeout expires). Once the client gets a response, it immediately sends another request.

```go
package main

import (
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

// LongPollBroker manages long-polling clients.
type LongPollBroker struct {
    mu       sync.Mutex
    messages []string
    version  int
    waiters  []chan struct{}
}

func NewLongPollBroker() *LongPollBroker {
    return &LongPollBroker{}
}

// Publish adds a message and notifies waiting clients.
func (b *LongPollBroker) Publish(msg string) {
    b.mu.Lock()
    b.messages = append(b.messages, msg)
    b.version++
    waiters := b.waiters
    b.waiters = nil
    b.mu.Unlock()

    // Wake up all waiting long-poll requests.
    for _, ch := range waiters {
        close(ch)
    }
}

// Poll blocks until new messages arrive after the given version,
// or until the timeout expires.
func (b *LongPollBroker) Poll(afterVersion int, timeout time.Duration) ([]string, int) {
    b.mu.Lock()

    // If there are already new messages, return immediately.
    if b.version > afterVersion {
        msgs := b.messages[afterVersion:]
        ver := b.version
        b.mu.Unlock()
        return msgs, ver
    }

    // No new messages -- wait.
    ch := make(chan struct{})
    b.waiters = append(b.waiters, ch)
    b.mu.Unlock()

    select {
    case <-ch:
        // New message arrived.
        b.mu.Lock()
        msgs := b.messages[afterVersion:]
        ver := b.version
        b.mu.Unlock()
        return msgs, ver
    case <-time.After(timeout):
        // Timeout -- return empty.
        return nil, afterVersion
    }
}

func main() {
    broker := NewLongPollBroker()

    http.HandleFunc("/poll", func(w http.ResponseWriter, r *http.Request) {
        // Client sends its last known version.
        var afterVersion int
        if v := r.URL.Query().Get("version"); v != "" {
            fmt.Sscanf(v, "%d", &afterVersion)
        }

        // Block for up to 30 seconds waiting for new messages.
        messages, newVersion := broker.Poll(afterVersion, 30*time.Second)

        json.NewEncoder(w).Encode(map[string]interface{}{
            "messages": messages,
            "version":  newVersion,
        })
    })

    http.HandleFunc("/send", func(w http.ResponseWriter, r *http.Request) {
        msg := r.URL.Query().Get("msg")
        broker.Publish(msg)
        w.Write([]byte("ok"))
    })

    http.ListenAndServe(":8080", nil)
}
```

### Comprehensive Comparison

| Feature | Long Polling | WebSocket | SSE |
|---------|-------------|-----------|-----|
| **Direction** | Client-initiated (simulated push) | Full duplex | Server to client |
| **Protocol** | HTTP | WebSocket (ws://, wss://) | HTTP |
| **Connection** | New HTTP request each cycle | Single persistent TCP | Single persistent HTTP |
| **Latency** | Medium (depends on poll interval) | Low (immediate) | Low (immediate) |
| **Overhead** | High (HTTP headers each request) | Low (2-byte frame header) | Low (small text format) |
| **Browser support** | Universal | Universal (modern) | Universal (except old IE) |
| **Proxy/CDN friendly** | Yes | Sometimes problematic | Yes |
| **Auto-reconnect** | Manual | Manual | Built-in |
| **Event IDs** | Manual | Manual | Built-in |
| **Binary data** | Base64 encoding needed | Native support | Base64 encoding needed |
| **Max connections** | Limited by HTTP pool | Independent | 6 per domain (HTTP/1.1) |
| **Complexity** | Low | Medium | Low |
| **Use case** | Fallback, simple needs | Chat, games, collaboration | Notifications, feeds, dashboards |

### Decision Flowchart

```
Do you need bidirectional communication?
├── YES → Do you need low latency?
│         ├── YES → Use WebSockets
│         └── NO  → WebSockets or Long Polling
└── NO  → Server-to-client only?
          ├── YES → Do proxies/firewalls block WebSockets?
          │         ├── YES → Use SSE
          │         └── NO  → SSE (simpler) or WebSocket (more flexible)
          └── NO  → Use regular HTTP requests
```

### Practical Guidelines

When to use **WebSockets**:
- Chat applications
- Real-time collaborative editing
- Multiplayer games
- Live trading platforms
- Any scenario where the client sends frequent messages to the server

When to use **SSE**:
- Live dashboards and monitoring
- Social media feeds
- Notification streams
- Event logs
- Any scenario where the server pushes updates and the client rarely or never sends data

When to use **Long Polling**:
- As a fallback when WebSockets and SSE are unavailable
- Very simple notification systems
- When you need maximum compatibility with corporate proxies
- When the update frequency is low (minutes, not seconds)

---

## 13. Reconnection Strategies

Network connections drop. Servers restart. Load balancers rotate. Your client-side code must handle disconnections gracefully and reconnect automatically.

### Exponential Backoff with Jitter

The most important reconnection pattern is exponential backoff with jitter. Without it, when a server restarts, all clients reconnect simultaneously (a "thundering herd"), potentially overloading the server before it can stabilize.

```go
package main

import (
    "context"
    "log"
    "math"
    "math/rand"
    "time"

    "nhooyr.io/websocket"
    "nhooyr.io/websocket/wsjson"
)

// ReconnectConfig controls reconnection behavior.
type ReconnectConfig struct {
    InitialDelay time.Duration // First retry delay.
    MaxDelay     time.Duration // Maximum delay cap.
    MaxRetries   int           // 0 = unlimited.
    Multiplier   float64       // Backoff multiplier (typically 2.0).
    JitterFactor float64       // Random jitter factor (0.0 to 1.0).
}

var DefaultReconnectConfig = ReconnectConfig{
    InitialDelay: 1 * time.Second,
    MaxDelay:     60 * time.Second,
    MaxRetries:   0, // Unlimited.
    Multiplier:   2.0,
    JitterFactor: 0.5,
}

// calculateDelay returns the delay for a given attempt number.
func (c ReconnectConfig) calculateDelay(attempt int) time.Duration {
    delay := float64(c.InitialDelay) * math.Pow(c.Multiplier, float64(attempt))
    if delay > float64(c.MaxDelay) {
        delay = float64(c.MaxDelay)
    }

    // Add jitter: delay * (1 +/- jitterFactor * random).
    jitter := delay * c.JitterFactor * (2*rand.Float64() - 1)
    delay += jitter

    if delay < 0 {
        delay = float64(c.InitialDelay)
    }

    return time.Duration(delay)
}

// ResilientClient wraps a WebSocket connection with automatic reconnection.
type ResilientClient struct {
    url     string
    config  ReconnectConfig
    onMsg   func(ctx context.Context, data []byte)
    onState func(state string)
}

func NewResilientClient(url string, config ReconnectConfig) *ResilientClient {
    return &ResilientClient{
        url:    url,
        config: config,
    }
}

func (rc *ResilientClient) OnMessage(handler func(ctx context.Context, data []byte)) {
    rc.onMsg = handler
}

func (rc *ResilientClient) OnStateChange(handler func(state string)) {
    rc.onState = handler
}

// Connect starts the connection loop. It blocks until the context is cancelled.
func (rc *ResilientClient) Connect(ctx context.Context) error {
    attempt := 0

    for {
        if rc.onState != nil {
            rc.onState("connecting")
        }

        err := rc.connectOnce(ctx)
        if err == nil {
            // Connection closed normally (server sent close frame).
            attempt = 0
        }

        // Check if context is cancelled.
        if ctx.Err() != nil {
            return ctx.Err()
        }

        // Calculate backoff delay.
        if rc.config.MaxRetries > 0 && attempt >= rc.config.MaxRetries {
            return fmt.Errorf("max retries (%d) exceeded", rc.config.MaxRetries)
        }

        delay := rc.config.calculateDelay(attempt)
        attempt++

        if rc.onState != nil {
            rc.onState(fmt.Sprintf("reconnecting in %v (attempt %d)", delay, attempt))
        }

        log.Printf("reconnecting in %v (attempt %d)", delay, attempt)

        select {
        case <-time.After(delay):
            // Continue to reconnect.
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}

func (rc *ResilientClient) connectOnce(ctx context.Context) error {
    conn, _, err := websocket.Dial(ctx, rc.url, nil)
    if err != nil {
        return fmt.Errorf("dial: %w", err)
    }
    defer conn.CloseNow()

    if rc.onState != nil {
        rc.onState("connected")
    }
    log.Println("connected")

    // Read messages until error.
    for {
        _, data, err := conn.Read(ctx)
        if err != nil {
            return err
        }
        if rc.onMsg != nil {
            rc.onMsg(ctx, data)
        }
    }
}
```

### Usage

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    client := NewResilientClient("ws://localhost:8080/ws", DefaultReconnectConfig)

    client.OnMessage(func(ctx context.Context, data []byte) {
        log.Printf("received: %s", data)
    })

    client.OnStateChange(func(state string) {
        log.Printf("state: %s", state)
    })

    // This blocks forever (or until context is cancelled).
    if err := client.Connect(ctx); err != nil {
        log.Fatalf("client error: %v", err)
    }
}
```

### Backoff Delay Progression

| Attempt | Base Delay | With Jitter (approx) |
|---------|-----------|----------------------|
| 0 | 1s | 0.5s - 1.5s |
| 1 | 2s | 1.0s - 3.0s |
| 2 | 4s | 2.0s - 6.0s |
| 3 | 8s | 4.0s - 12.0s |
| 4 | 16s | 8.0s - 24.0s |
| 5 | 32s | 16.0s - 48.0s |
| 6+ | 60s (cap) | 30.0s - 60.0s |

### JavaScript Client-Side Reconnection

For browser clients, implement reconnection in JavaScript:

```javascript
class ResilientWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.initialDelay = options.initialDelay || 1000;
        this.maxDelay = options.maxDelay || 60000;
        this.multiplier = options.multiplier || 2;
        this.jitter = options.jitter || 0.5;
        this.attempt = 0;
        this.handlers = { message: [], open: [], close: [], error: [] };
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
            console.log("Connected");
            this.attempt = 0; // Reset on successful connection.
            this.handlers.open.forEach(h => h());
        };

        this.ws.onclose = (event) => {
            console.log(`Disconnected: code=${event.code}`);
            this.handlers.close.forEach(h => h(event));

            if (event.code !== 1000) {
                // Abnormal close -- reconnect.
                this.scheduleReconnect();
            }
        };

        this.ws.onerror = (error) => {
            console.error("WebSocket error:", error);
            this.handlers.error.forEach(h => h(error));
        };

        this.ws.onmessage = (event) => {
            this.handlers.message.forEach(h => h(event));
        };
    }

    scheduleReconnect() {
        let delay = this.initialDelay * Math.pow(this.multiplier, this.attempt);
        delay = Math.min(delay, this.maxDelay);

        // Add jitter.
        const jitterRange = delay * this.jitter;
        delay += (Math.random() * 2 - 1) * jitterRange;

        this.attempt++;
        console.log(`Reconnecting in ${Math.round(delay)}ms (attempt ${this.attempt})`);

        setTimeout(() => this.connect(), delay);
    }

    send(data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(data);
        }
    }

    on(event, handler) {
        if (this.handlers[event]) {
            this.handlers[event].push(handler);
        }
    }

    close() {
        if (this.ws) {
            this.ws.close(1000, "client closing");
        }
    }
}

// Usage:
const ws = new ResilientWebSocket("ws://localhost:8080/ws");
ws.on("message", (e) => console.log("Message:", e.data));
ws.on("open", () => console.log("Connected!"));
ws.connect();
```

---

## 14. Load Testing WebSocket Servers

Load testing WebSocket servers requires specialized tools because standard HTTP benchmarking tools (like `wrk` or `ab`) do not support persistent WebSocket connections.

### Go-Based Load Tester

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "sync"
    "sync/atomic"
    "time"

    "nhooyr.io/websocket"
)

type LoadTestResult struct {
    TotalConnections   int64
    SuccessConnections int64
    FailedConnections  int64
    MessagesSent       int64
    MessagesReceived   int64
    Errors             int64
    Duration           time.Duration
    AvgConnectTime     time.Duration
    MaxConnectTime     time.Duration
    MinConnectTime     time.Duration
}

type LoadTester struct {
    url            string
    numClients     int
    duration       time.Duration
    messageRate    time.Duration // How often each client sends a message.
    rampUpDuration time.Duration

    // Atomic counters.
    connected    int64
    failed       int64
    msgSent      int64
    msgReceived  int64
    errors       int64
    connectTimes []time.Duration
    mu           sync.Mutex
}

func NewLoadTester(url string, numClients int, duration time.Duration) *LoadTester {
    return &LoadTester{
        url:            url,
        numClients:     numClients,
        duration:       duration,
        messageRate:    1 * time.Second,
        rampUpDuration: 10 * time.Second,
    }
}

func (lt *LoadTester) Run(ctx context.Context) LoadTestResult {
    start := time.Now()
    ctx, cancel := context.WithTimeout(ctx, lt.duration)
    defer cancel()

    var wg sync.WaitGroup

    // Calculate ramp-up delay between each client connection.
    rampDelay := lt.rampUpDuration / time.Duration(lt.numClients)

    log.Printf("Starting load test: %d clients, %v duration, %v ramp-up",
        lt.numClients, lt.duration, lt.rampUpDuration)

    for i := 0; i < lt.numClients; i++ {
        wg.Add(1)
        go func(clientID int) {
            defer wg.Done()
            lt.runClient(ctx, clientID)
        }(i)

        // Stagger connections to avoid thundering herd.
        if rampDelay > 0 {
            time.Sleep(rampDelay)
        }
    }

    wg.Wait()
    elapsed := time.Since(start)

    result := LoadTestResult{
        TotalConnections:   int64(lt.numClients),
        SuccessConnections: atomic.LoadInt64(&lt.connected),
        FailedConnections:  atomic.LoadInt64(&lt.failed),
        MessagesSent:       atomic.LoadInt64(&lt.msgSent),
        MessagesReceived:   atomic.LoadInt64(&lt.msgReceived),
        Errors:             atomic.LoadInt64(&lt.errors),
        Duration:           elapsed,
    }

    // Calculate connect time stats.
    lt.mu.Lock()
    if len(lt.connectTimes) > 0 {
        var total time.Duration
        result.MinConnectTime = lt.connectTimes[0]
        for _, d := range lt.connectTimes {
            total += d
            if d > result.MaxConnectTime {
                result.MaxConnectTime = d
            }
            if d < result.MinConnectTime {
                result.MinConnectTime = d
            }
        }
        result.AvgConnectTime = total / time.Duration(len(lt.connectTimes))
    }
    lt.mu.Unlock()

    return result
}

func (lt *LoadTester) runClient(ctx context.Context, id int) {
    connectStart := time.Now()

    conn, _, err := websocket.Dial(ctx, lt.url, nil)
    if err != nil {
        atomic.AddInt64(&lt.failed, 1)
        return
    }
    defer conn.CloseNow()

    connectDuration := time.Since(connectStart)
    lt.mu.Lock()
    lt.connectTimes = append(lt.connectTimes, connectDuration)
    lt.mu.Unlock()

    atomic.AddInt64(&lt.connected, 1)

    // Start a reader goroutine.
    go func() {
        for {
            _, _, err := conn.Read(ctx)
            if err != nil {
                return
            }
            atomic.AddInt64(&lt.msgReceived, 1)
        }
    }()

    // Send messages at the configured rate.
    ticker := time.NewTicker(lt.messageRate)
    defer ticker.Stop()

    msg := fmt.Sprintf(`{"client": %d, "type": "ping"}`, id)

    for {
        select {
        case <-ticker.C:
            err := conn.Write(ctx, websocket.MessageText, []byte(msg))
            if err != nil {
                atomic.AddInt64(&lt.errors, 1)
                return
            }
            atomic.AddInt64(&lt.msgSent, 1)

        case <-ctx.Done():
            conn.Close(websocket.StatusNormalClosure, "test complete")
            return
        }
    }
}

func (r LoadTestResult) String() string {
    return fmt.Sprintf(`
=== WebSocket Load Test Results ===
Duration:             %v
Total Connections:    %d
Successful:           %d
Failed:               %d
Messages Sent:        %d
Messages Received:    %d
Errors:               %d
Avg Connect Time:     %v
Min Connect Time:     %v
Max Connect Time:     %v
Messages/sec (sent):  %.1f
Messages/sec (recv):  %.1f
===================================`,
        r.Duration,
        r.TotalConnections,
        r.SuccessConnections,
        r.FailedConnections,
        r.MessagesSent,
        r.MessagesReceived,
        r.Errors,
        r.AvgConnectTime,
        r.MinConnectTime,
        r.MaxConnectTime,
        float64(r.MessagesSent)/r.Duration.Seconds(),
        float64(r.MessagesReceived)/r.Duration.Seconds(),
    )
}

func main() {
    url := flag.String("url", "ws://localhost:8080/ws", "WebSocket URL")
    clients := flag.Int("clients", 100, "Number of concurrent clients")
    duration := flag.Duration("duration", 30*time.Second, "Test duration")
    flag.Parse()

    tester := NewLoadTester(*url, *clients, *duration)
    result := tester.Run(context.Background())
    fmt.Println(result)
}
```

### Running the Load Test

```bash
# Start your WebSocket server first, then:
go run loadtest.go -url ws://localhost:8080/ws -clients 1000 -duration 60s
```

### External Load Testing Tools

Several external tools are purpose-built for WebSocket load testing:

**websocket-bench** (Go-based):
```bash
go install github.com/myzhan/boomer@latest
# Or use a dedicated WebSocket benchmarking tool.
```

**Artillery** (Node.js-based, supports WebSockets):
```yaml
# artillery-config.yml
config:
  target: "ws://localhost:8080"
  phases:
    - duration: 60
      arrivalRate: 10     # 10 new users per second
      rampTo: 100         # ramp up to 100 users/second
  ws:
    subprotocols:
      - "chat"

scenarios:
  - engine: ws
    flow:
      - send: '{"type":"join","room":"loadtest"}'
      - think: 1
      - loop:
          - send: '{"type":"chat","content":"load test message"}'
          - think: 0.5
        count: 100
```

```bash
artillery run artillery-config.yml
```

### Metrics to Monitor During Load Tests

| Metric | What to Watch | Red Flag |
|--------|---------------|----------|
| Connection success rate | Percentage of successful upgrades | Below 99% |
| Message latency (p50/p95/p99) | Time from send to receive | p99 > 500ms |
| Memory usage | Server RSS memory | Grows without bound |
| Goroutine count | `runtime.NumGoroutine()` | Grows without bound |
| CPU usage | Server CPU | Sustained > 80% |
| File descriptors | `ulimit -n` vs active connections | Approaching limit |
| Write buffer fill rate | Channel buffer occupancy | Consistently full |
| Disconnect rate | Unexpected disconnections per second | Increasing over time |

---

## 15. Production Considerations

### Connection Limits and Resource Management

```go
package main

import (
    "log"
    "net/http"
    "sync"
    "sync/atomic"
    "time"

    "github.com/gorilla/websocket"
    "golang.org/x/time/rate"
)

// ConnectionManager enforces connection limits and tracks active connections.
type ConnectionManager struct {
    maxConnections   int64
    activeConns      int64
    totalConns       int64
    rejectedConns    int64

    // Per-IP rate limiting.
    ipLimiters map[string]*rate.Limiter
    ipMu       sync.RWMutex
    ipRate     rate.Limit
    ipBurst    int
}

func NewConnectionManager(maxConns int64) *ConnectionManager {
    return &ConnectionManager{
        maxConnections: maxConns,
        ipLimiters:     make(map[string]*rate.Limiter),
        ipRate:         rate.Every(time.Second), // 1 connection per second per IP.
        ipBurst:        5,                       // Burst of 5 connections.
    }
}

func (cm *ConnectionManager) Allow(ip string) bool {
    // Check global connection limit.
    if atomic.LoadInt64(&cm.activeConns) >= cm.maxConnections {
        atomic.AddInt64(&cm.rejectedConns, 1)
        return false
    }

    // Check per-IP rate limit.
    cm.ipMu.Lock()
    limiter, ok := cm.ipLimiters[ip]
    if !ok {
        limiter = rate.NewLimiter(cm.ipRate, cm.ipBurst)
        cm.ipLimiters[ip] = limiter
    }
    cm.ipMu.Unlock()

    if !limiter.Allow() {
        atomic.AddInt64(&cm.rejectedConns, 1)
        return false
    }

    return true
}

func (cm *ConnectionManager) Add() {
    atomic.AddInt64(&cm.activeConns, 1)
    atomic.AddInt64(&cm.totalConns, 1)
}

func (cm *ConnectionManager) Remove() {
    atomic.AddInt64(&cm.activeConns, -1)
}

func (cm *ConnectionManager) Stats() map[string]int64 {
    return map[string]int64{
        "active":   atomic.LoadInt64(&cm.activeConns),
        "total":    atomic.LoadInt64(&cm.totalConns),
        "rejected": atomic.LoadInt64(&cm.rejectedConns),
    }
}

// CleanupIPLimiters periodically removes old IP limiters to prevent memory leaks.
func (cm *ConnectionManager) CleanupIPLimiters(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for range ticker.C {
        cm.ipMu.Lock()
        // In a real system, track last access time and evict stale entries.
        // For simplicity, just clear old entries periodically.
        if len(cm.ipLimiters) > 10000 {
            cm.ipLimiters = make(map[string]*rate.Limiter)
        }
        cm.ipMu.Unlock()
    }
}
```

### Production Upgrader Configuration

```go
var upgrader = websocket.Upgrader{
    // Buffer sizes affect memory per connection.
    // 1024 bytes each for a chat app; increase for large payloads.
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,

    // Handshake timeout prevents slow clients from tying up resources.
    HandshakeTimeout: 10 * time.Second,

    // Always validate origins in production.
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        switch origin {
        case "https://myapp.com", "https://www.myapp.com":
            return true
        default:
            log.Printf("rejected origin: %s", origin)
            return false
        }
    },

    // Enable per-message compression for text-heavy protocols.
    EnableCompression: true,
}
```

### Timeout and Deadline Configuration

```go
const (
    // writeWait: time allowed to write a message.
    // Should be longer than the expected write time but short enough
    // to detect dead connections.
    writeWait = 10 * time.Second

    // pongWait: time allowed to read the next pong.
    // This is effectively the "connection alive" timeout.
    // 60 seconds is a good default; reduce for faster detection.
    pongWait = 60 * time.Second

    // pingPeriod: interval between pings.
    // MUST be less than pongWait to allow the pong to arrive.
    // Using 9/10 of pongWait gives a comfortable margin.
    pingPeriod = (pongWait * 9) / 10

    // maxMessageSize: prevent a single client from consuming
    // excessive memory with a huge message.
    maxMessageSize = 64 * 1024 // 64 KB for chat, increase for file transfer.
)
```

### Message Rate Limiting

```go
// RateLimitedClient wraps a client with per-client rate limiting.
type RateLimitedClient struct {
    *Client
    limiter *rate.Limiter
}

func NewRateLimitedClient(client *Client, msgsPerSecond float64, burst int) *RateLimitedClient {
    return &RateLimitedClient{
        Client:  client,
        limiter: rate.NewLimiter(rate.Limit(msgsPerSecond), burst),
    }
}

func (rl *RateLimitedClient) readPump() {
    defer func() {
        rl.hub.unregister <- rl.Client
        rl.conn.Close()
    }()

    rl.conn.SetReadLimit(maxMessageSize)
    rl.conn.SetReadDeadline(time.Now().Add(pongWait))
    rl.conn.SetPongHandler(func(string) error {
        rl.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := rl.conn.ReadMessage()
        if err != nil {
            break
        }

        // Enforce rate limit.
        if !rl.limiter.Allow() {
            // Client is sending too fast -- send a warning.
            warning := []byte(`{"type":"error","content":"rate limit exceeded"}`)
            select {
            case rl.send <- warning:
            default:
            }
            continue // Drop the message.
        }

        rl.hub.broadcast <- message
    }
}
```

### Graceful Shutdown

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    hub := NewHub()
    go hub.Run()

    mux := http.NewServeMux()
    mux.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        serveWS(hub, w, r)
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,

        // These timeouts apply to the initial HTTP phase
        // (before the WebSocket upgrade).
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server in a goroutine.
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for interrupt signal.
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down...")

    // Create a deadline for shutdown.
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Notify all connected clients that the server is going away.
    hub.NotifyShutdown()

    // Give clients time to disconnect gracefully.
    time.Sleep(5 * time.Second)

    // Shut down the HTTP server (stops accepting new connections).
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("shutdown error: %v", err)
    }

    log.Println("Server stopped")
}

// Add to Hub:
func (h *Hub) NotifyShutdown() {
    shutdownMsg := []byte(`{"type":"system","content":"server shutting down"}`)
    for client := range h.clients {
        select {
        case client.send <- shutdownMsg:
        default:
        }
        // Send a Close frame with "Going Away" status.
        client.conn.WriteControl(
            websocket.CloseMessage,
            websocket.FormatCloseMessage(websocket.CloseGoingAway, "server shutdown"),
            time.Now().Add(writeWait),
        )
    }
}
```

### Monitoring and Observability

```go
package main

import (
    "encoding/json"
    "net/http"
    "runtime"
    "sync/atomic"
    "time"
)

// WSMetrics tracks WebSocket server metrics.
type WSMetrics struct {
    ConnectionsActive   int64 `json:"connections_active"`
    ConnectionsTotal    int64 `json:"connections_total"`
    ConnectionsRejected int64 `json:"connections_rejected"`
    MessagesSent        int64 `json:"messages_sent"`
    MessagesReceived    int64 `json:"messages_received"`
    BytesSent           int64 `json:"bytes_sent"`
    BytesReceived       int64 `json:"bytes_received"`
    Errors              int64 `json:"errors"`
    Goroutines          int   `json:"goroutines"`
    UptimeSeconds       int64 `json:"uptime_seconds"`
}

var (
    metrics   WSMetrics
    startTime = time.Now()
)

func metricsHandler(w http.ResponseWriter, r *http.Request) {
    snapshot := WSMetrics{
        ConnectionsActive:   atomic.LoadInt64(&metrics.ConnectionsActive),
        ConnectionsTotal:    atomic.LoadInt64(&metrics.ConnectionsTotal),
        ConnectionsRejected: atomic.LoadInt64(&metrics.ConnectionsRejected),
        MessagesSent:        atomic.LoadInt64(&metrics.MessagesSent),
        MessagesReceived:    atomic.LoadInt64(&metrics.MessagesReceived),
        BytesSent:           atomic.LoadInt64(&metrics.BytesSent),
        BytesReceived:       atomic.LoadInt64(&metrics.BytesReceived),
        Errors:              atomic.LoadInt64(&metrics.Errors),
        Goroutines:          runtime.NumGoroutine(),
        UptimeSeconds:       int64(time.Since(startTime).Seconds()),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(snapshot)
}

// Usage -- instrument your WebSocket handler:
func instrumentedReadPump(c *Client) {
    defer func() {
        atomic.AddInt64(&metrics.ConnectionsActive, -1)
        c.hub.unregister <- c
        c.conn.Close()
    }()

    atomic.AddInt64(&metrics.ConnectionsActive, 1)
    atomic.AddInt64(&metrics.ConnectionsTotal, 1)

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            atomic.AddInt64(&metrics.Errors, 1)
            break
        }
        atomic.AddInt64(&metrics.MessagesReceived, 1)
        atomic.AddInt64(&metrics.BytesReceived, int64(len(message)))

        c.hub.broadcast <- message
    }
}
```

### Operating System Tuning

When running a WebSocket server with many concurrent connections, you need to tune OS-level settings:

```bash
# Check current file descriptor limit.
ulimit -n

# Increase for the current session (requires root for values > 65535).
ulimit -n 100000

# Permanent change on Linux: edit /etc/security/limits.conf
# youruser soft nofile 100000
# youruser hard nofile 100000

# On Linux, also increase system-wide limits:
# echo "fs.file-max = 200000" >> /etc/sysctl.conf
# sysctl -p

# Check current number of open file descriptors for a process:
ls /proc/<pid>/fd | wc -l
```

> **Warning:** Each WebSocket connection consumes at least one file descriptor. If your server handles 10,000 connections, you need at least 10,000 file descriptors, plus some for the server itself (logs, databases, etc.). Set the limit to at least 2x your expected maximum connections.

### Memory Budget Per Connection

Understanding memory usage helps with capacity planning:

| Component | Memory (approx) |
|-----------|-----------------|
| TCP socket buffers (OS) | ~8 KB (4 KB read + 4 KB write) |
| gorilla/websocket read buffer | 1-4 KB (configurable) |
| gorilla/websocket write buffer | 1-4 KB (configurable) |
| Send channel buffer (256 msgs x ~256 bytes) | ~64 KB |
| Goroutine stack (read pump) | ~8 KB initial, grows as needed |
| Goroutine stack (write pump) | ~8 KB initial, grows as needed |
| Application state per client | Variable |
| **Total per connection** | **~100-200 KB typical** |

With 200 KB per connection:
- 10,000 connections = ~2 GB
- 50,000 connections = ~10 GB
- 100,000 connections = ~20 GB

To reduce memory, consider:
- Smaller buffer sizes (`ReadBufferSize`, `WriteBufferSize`)
- Smaller send channel buffers
- Buffer pooling with `sync.Pool`
- Compressing messages to reduce payload sizes

### Buffer Pooling

```go
// Use a buffer pool to reduce GC pressure for message processing.
var bufferPool = sync.Pool{
    New: func() interface{} {
        buf := make([]byte, 0, 1024)
        return &buf
    },
}

func processMessage(conn *websocket.Conn) error {
    _, reader, err := conn.NextReader()
    if err != nil {
        return err
    }

    bufPtr := bufferPool.Get().(*[]byte)
    buf := *bufPtr
    buf = buf[:0] // Reset length, keep capacity.
    defer func() {
        *bufPtr = buf
        bufferPool.Put(bufPtr)
    }()

    // Read into pooled buffer.
    buf, err = io.ReadAll(reader)
    if err != nil {
        return err
    }

    // Process the message...
    return nil
}
```

### Connection Upgrade Buffer Pooling

gorilla/websocket supports custom buffer pools for the upgrader:

```go
// BufferPool reuses buffers for WebSocket frame reading/writing.
type wsBufferPool struct {
    pool sync.Pool
}

func newWSBufferPool() *wsBufferPool {
    return &wsBufferPool{
        pool: sync.Pool{
            New: func() interface{} {
                buf := make([]byte, 4096)
                return &buf
            },
        },
    }
}

func (p *wsBufferPool) Get() interface{} {
    return p.pool.Get()
}

func (p *wsBufferPool) Put(v interface{}) {
    p.pool.Put(v)
}

// Use in the upgrader.
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    WriteBufferPool: newWSBufferPool(),
}
```

### Reverse Proxy Configuration

When running behind NGINX or another reverse proxy, ensure WebSocket upgrades are forwarded correctly:

**NGINX configuration:**

```nginx
upstream websocket_backend {
    # Use ip_hash for sticky sessions -- ensures a client
    # reconnects to the same backend server.
    ip_hash;

    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 443 ssl;
    server_name myapp.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location /ws {
        proxy_pass http://websocket_backend;

        # Required for WebSocket upgrade.
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Pass client IP.
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;

        # Timeouts -- must be longer than your ping interval.
        proxy_read_timeout 86400s;  # 24 hours.
        proxy_send_timeout 86400s;

        # Buffer settings.
        proxy_buffering off;
    }
}
```

> **Warning:** The `proxy_read_timeout` must be longer than your ping interval. If the proxy times out the connection between pings, clients will experience unexpected disconnections. With a 54-second ping period, set the proxy timeout to at least 120 seconds. Using 86400 seconds (24 hours) is a safe default.

---

## 16. Real-World Example: Live Dashboard System

Let us combine everything into a practical example: a live metrics dashboard that pushes real-time system metrics to connected browser clients.

### Architecture

```
Live Dashboard Architecture:

  ┌─────────────┐     ┌───────────────┐
  │ Metric       │     │               │
  │ Collector    ├────►│               │
  │ (goroutine)  │     │  Dashboard    │     ┌──────────┐
  └─────────────┘     │  Server       ├────►│ Browser 1│
                      │               │     └──────────┘
  ┌─────────────┐     │  - WebSocket  │     ┌──────────┐
  │ Alert        │     │    Hub       ├────►│ Browser 2│
  │ Checker     ├────►│  - SSE       │     └──────────┘
  │ (goroutine)  │     │    Fallback  │     ┌──────────┐
  └─────────────┘     │  - Auth      ├────►│ Browser 3│
                      │  - Metrics   │     └──────────┘
                      └───────────────┘
```

### Complete Dashboard Server

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "math/rand"
    "net/http"
    "os"
    "os/signal"
    "runtime"
    "sync"
    "sync/atomic"
    "syscall"
    "time"

    "github.com/gorilla/websocket"
)

// --- Metric Types ---

type MetricSnapshot struct {
    Timestamp    time.Time         `json:"timestamp"`
    CPU          float64           `json:"cpu_percent"`
    MemoryUsed   uint64            `json:"memory_used_bytes"`
    MemoryTotal  uint64            `json:"memory_total_bytes"`
    Goroutines   int               `json:"goroutines"`
    Connections  int               `json:"ws_connections"`
    RequestRate  float64           `json:"requests_per_second"`
    ErrorRate    float64           `json:"errors_per_second"`
    CustomGauges map[string]float64 `json:"custom_gauges,omitempty"`
}

type Alert struct {
    ID        string    `json:"id"`
    Level     string    `json:"level"` // "info", "warning", "critical"
    Message   string    `json:"message"`
    Metric    string    `json:"metric"`
    Value     float64   `json:"value"`
    Threshold float64   `json:"threshold"`
    Timestamp time.Time `json:"timestamp"`
}

type DashboardMessage struct {
    Type string      `json:"type"` // "metrics", "alert", "system"
    Data interface{} `json:"data"`
}

// --- Dashboard Hub ---

type DashboardHub struct {
    clients    map[*DashboardClient]bool
    register   chan *DashboardClient
    unregister chan *DashboardClient
    broadcast  chan DashboardMessage
    mu         sync.RWMutex

    // Track connection stats.
    totalConnections int64
}

func NewDashboardHub() *DashboardHub {
    return &DashboardHub{
        clients:    make(map[*DashboardClient]bool),
        register:   make(chan *DashboardClient),
        unregister: make(chan *DashboardClient),
        broadcast:  make(chan DashboardMessage, 64),
    }
}

func (h *DashboardHub) Run(ctx context.Context) {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            atomic.AddInt64(&h.totalConnections, 1)
            log.Printf("dashboard client connected (total: %d)", h.ClientCount())

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()
            log.Printf("dashboard client disconnected (total: %d)", h.ClientCount())

        case msg := <-h.broadcast:
            data, err := json.Marshal(msg)
            if err != nil {
                log.Printf("marshal error: %v", err)
                continue
            }

            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- data:
                default:
                    // Slow client -- close it in a separate goroutine
                    // to avoid holding the lock.
                    go func(c *DashboardClient) {
                        h.unregister <- c
                    }(client)
                }
            }
            h.mu.RUnlock()

        case <-ctx.Done():
            // Graceful shutdown -- close all clients.
            h.mu.Lock()
            for client := range h.clients {
                close(client.send)
                delete(h.clients, client)
            }
            h.mu.Unlock()
            return
        }
    }
}

func (h *DashboardHub) ClientCount() int {
    h.mu.RLock()
    defer h.mu.RUnlock()
    return len(h.clients)
}

func (h *DashboardHub) Broadcast(msg DashboardMessage) {
    select {
    case h.broadcast <- msg:
    default:
        log.Println("broadcast channel full, dropping message")
    }
}

// --- Dashboard Client ---

type DashboardClient struct {
    hub  *DashboardHub
    conn *websocket.Conn
    send chan []byte
}

const (
    dashWriteWait  = 10 * time.Second
    dashPongWait   = 60 * time.Second
    dashPingPeriod = (dashPongWait * 9) / 10
    dashMaxMsgSize = 1024
)

func (c *DashboardClient) writePump() {
    ticker := time.NewTicker(dashPingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(dashWriteWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Batch queued messages.
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte("\n"))
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(dashWriteWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

func (c *DashboardClient) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(dashMaxMsgSize)
    c.conn.SetReadDeadline(time.Now().Add(dashPongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(dashPongWait))
        return nil
    })

    for {
        _, _, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        // Dashboard is mostly server-to-client.
        // Client messages could be subscription changes, etc.
    }
}

// --- Metric Collector ---

type MetricCollector struct {
    hub       *DashboardHub
    interval  time.Duration
    requestCt int64
    errorCt   int64
}

func NewMetricCollector(hub *DashboardHub, interval time.Duration) *MetricCollector {
    return &MetricCollector{
        hub:      hub,
        interval: interval,
    }
}

func (mc *MetricCollector) Run(ctx context.Context) {
    ticker := time.NewTicker(mc.interval)
    defer ticker.Stop()

    var lastRequestCt, lastErrorCt int64
    lastTime := time.Now()

    for {
        select {
        case <-ticker.C:
            now := time.Now()
            elapsed := now.Sub(lastTime).Seconds()

            currentReqs := atomic.LoadInt64(&mc.requestCt)
            currentErrs := atomic.LoadInt64(&mc.errorCt)

            var memStats runtime.MemStats
            runtime.ReadMemStats(&memStats)

            snapshot := MetricSnapshot{
                Timestamp:   now,
                CPU:         simulateCPU(), // Replace with real metrics.
                MemoryUsed:  memStats.Alloc,
                MemoryTotal: memStats.Sys,
                Goroutines:  runtime.NumGoroutine(),
                Connections: mc.hub.ClientCount(),
                RequestRate: float64(currentReqs-lastRequestCt) / elapsed,
                ErrorRate:   float64(currentErrs-lastErrorCt) / elapsed,
                CustomGauges: map[string]float64{
                    "queue_depth":    float64(len(mc.hub.broadcast)),
                    "heap_objects":   float64(memStats.HeapObjects),
                    "gc_pause_ns":   float64(memStats.PauseNs[(memStats.NumGC+255)%256]),
                },
            }

            mc.hub.Broadcast(DashboardMessage{
                Type: "metrics",
                Data: snapshot,
            })

            lastRequestCt = currentReqs
            lastErrorCt = currentErrs
            lastTime = now

        case <-ctx.Done():
            return
        }
    }
}

func (mc *MetricCollector) RecordRequest() {
    atomic.AddInt64(&mc.requestCt, 1)
}

func (mc *MetricCollector) RecordError() {
    atomic.AddInt64(&mc.errorCt, 1)
}

// simulateCPU returns a simulated CPU percentage for demonstration.
func simulateCPU() float64 {
    return 20.0 + rand.Float64()*30.0
}

// --- Alert Checker ---

type AlertChecker struct {
    hub      *DashboardHub
    interval time.Duration
    rules    []AlertRule
}

type AlertRule struct {
    Metric    string
    Threshold float64
    Level     string
    Message   string
}

func NewAlertChecker(hub *DashboardHub) *AlertChecker {
    return &AlertChecker{
        hub:      hub,
        interval: 10 * time.Second,
        rules: []AlertRule{
            {Metric: "cpu_percent", Threshold: 80.0, Level: "warning", Message: "High CPU usage"},
            {Metric: "cpu_percent", Threshold: 95.0, Level: "critical", Message: "Critical CPU usage"},
            {Metric: "memory_percent", Threshold: 90.0, Level: "warning", Message: "High memory usage"},
            {Metric: "error_rate", Threshold: 10.0, Level: "warning", Message: "Elevated error rate"},
        },
    }
}

func (ac *AlertChecker) Run(ctx context.Context) {
    ticker := time.NewTicker(ac.interval)
    defer ticker.Stop()

    alertID := 0
    for {
        select {
        case <-ticker.C:
            // In a real system, check actual metrics against rules.
            cpu := simulateCPU()
            for _, rule := range ac.rules {
                if rule.Metric == "cpu_percent" && cpu > rule.Threshold {
                    alertID++
                    ac.hub.Broadcast(DashboardMessage{
                        Type: "alert",
                        Data: Alert{
                            ID:        fmt.Sprintf("alert-%d", alertID),
                            Level:     rule.Level,
                            Message:   rule.Message,
                            Metric:    rule.Metric,
                            Value:     cpu,
                            Threshold: rule.Threshold,
                            Timestamp: time.Now(),
                        },
                    })
                }
            }

        case <-ctx.Done():
            return
        }
    }
}

// --- HTTP Handlers ---

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin:     func(r *http.Request) bool { return true },
}

func serveDashboardWS(hub *DashboardHub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade: %v", err)
        return
    }

    client := &DashboardClient{
        hub:  hub,
        conn: conn,
        send: make(chan []byte, 64),
    }

    hub.register <- client

    go client.writePump()
    go client.readPump()
}

// --- SSE Fallback ---

func serveDashboardSSE(hub *DashboardHub, w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")

    messageChan := make(chan []byte, 64)

    // Register as a pseudo-client. In production, you would adapt
    // the hub to support SSE clients directly.
    client := &DashboardClient{
        hub:  hub,
        send: messageChan,
    }
    hub.register <- client
    defer func() {
        hub.unregister <- client
    }()

    ctx := r.Context()
    for {
        select {
        case msg, ok := <-messageChan:
            if !ok {
                return
            }
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()
        case <-ctx.Done():
            return
        }
    }
}

// --- Main ---

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    hub := NewDashboardHub()
    go hub.Run(ctx)

    collector := NewMetricCollector(hub, 2*time.Second)
    go collector.Run(ctx)

    alertChecker := NewAlertChecker(hub)
    go alertChecker.Run(ctx)

    mux := http.NewServeMux()

    // WebSocket endpoint.
    mux.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        collector.RecordRequest()
        serveDashboardWS(hub, w, r)
    })

    // SSE fallback endpoint.
    mux.HandleFunc("/events", func(w http.ResponseWriter, r *http.Request) {
        collector.RecordRequest()
        serveDashboardSSE(hub, w, r)
    })

    // Health check.
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]interface{}{
            "status":      "ok",
            "connections": hub.ClientCount(),
            "goroutines":  runtime.NumGoroutine(),
        })
    })

    // Dashboard page.
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/html")
        fmt.Fprint(w, dashboardHTML)
    })

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
    }

    go func() {
        log.Println("Dashboard server on :8080")
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server: %v", err)
        }
    }()

    // Wait for shutdown signal.
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down...")
    cancel() // Stop hub, collector, alert checker.

    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer shutdownCancel()
    srv.Shutdown(shutdownCtx)

    log.Println("Server stopped")
}

const dashboardHTML = `<!DOCTYPE html>
<html>
<head>
<title>Live Dashboard</title>
<style>
    body { font-family: monospace; background: #1a1a2e; color: #e0e0e0; padding: 20px; }
    h1 { color: #00d4ff; }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; }
    .card { background: #16213e; border: 1px solid #0f3460; border-radius: 8px; padding: 15px; }
    .card h3 { margin: 0 0 10px 0; color: #00d4ff; font-size: 12px; text-transform: uppercase; }
    .card .value { font-size: 28px; font-weight: bold; color: #e94560; }
    .alert { padding: 10px; margin: 5px 0; border-radius: 4px; }
    .alert.warning { background: #f39c1233; border-left: 4px solid #f39c12; }
    .alert.critical { background: #e7452233; border-left: 4px solid #e74522; }
    #log { max-height: 300px; overflow: auto; background: #0a0a1a; padding: 10px; border-radius: 4px; }
    .status { display: inline-block; width: 10px; height: 10px; border-radius: 50%; margin-right: 5px; }
    .status.connected { background: #2ecc71; }
    .status.disconnected { background: #e74c3c; }
</style>
</head>
<body>
<h1><span class="status disconnected" id="status"></span> Live Dashboard</h1>
<div class="grid">
    <div class="card"><h3>CPU</h3><div class="value" id="cpu">--</div></div>
    <div class="card"><h3>Memory</h3><div class="value" id="memory">--</div></div>
    <div class="card"><h3>Goroutines</h3><div class="value" id="goroutines">--</div></div>
    <div class="card"><h3>Connections</h3><div class="value" id="connections">--</div></div>
    <div class="card"><h3>Req/s</h3><div class="value" id="reqrate">--</div></div>
    <div class="card"><h3>Err/s</h3><div class="value" id="errrate">--</div></div>
</div>
<h2>Alerts</h2>
<div id="alerts"></div>
<h2>Event Log</h2>
<pre id="log"></pre>

<script>
let attempt = 0;
function connect() {
    const ws = new WebSocket("ws://" + location.host + "/ws");
    const statusEl = document.getElementById("status");

    ws.onopen = () => {
        statusEl.className = "status connected";
        attempt = 0;
        appendLog("Connected");
    };

    ws.onclose = () => {
        statusEl.className = "status disconnected";
        const delay = Math.min(1000 * Math.pow(2, attempt), 30000);
        attempt++;
        appendLog("Disconnected. Reconnecting in " + (delay/1000) + "s...");
        setTimeout(connect, delay);
    };

    ws.onmessage = (e) => {
        // Handle batched messages (newline-separated).
        e.data.split("\n").forEach(line => {
            if (!line.trim()) return;
            try {
                const msg = JSON.parse(line);
                if (msg.type === "metrics") updateMetrics(msg.data);
                if (msg.type === "alert") showAlert(msg.data);
            } catch(err) { /* skip parse errors */ }
        });
    };
}

function updateMetrics(m) {
    document.getElementById("cpu").textContent = m.cpu_percent.toFixed(1) + "%";
    document.getElementById("memory").textContent = formatBytes(m.memory_used_bytes);
    document.getElementById("goroutines").textContent = m.goroutines;
    document.getElementById("connections").textContent = m.ws_connections;
    document.getElementById("reqrate").textContent = m.requests_per_second.toFixed(1);
    document.getElementById("errrate").textContent = m.errors_per_second.toFixed(1);
}

function showAlert(a) {
    const div = document.createElement("div");
    div.className = "alert " + a.level;
    div.textContent = "[" + a.level.toUpperCase() + "] " + a.message +
        " (" + a.metric + "=" + a.value.toFixed(1) + ", threshold=" + a.threshold + ")";
    const container = document.getElementById("alerts");
    container.insertBefore(div, container.firstChild);
    if (container.children.length > 20) container.removeChild(container.lastChild);
}

function formatBytes(b) {
    if (b < 1024) return b + " B";
    if (b < 1024*1024) return (b/1024).toFixed(1) + " KB";
    return (b/1024/1024).toFixed(1) + " MB";
}

function appendLog(text) {
    const el = document.getElementById("log");
    el.textContent += new Date().toLocaleTimeString() + " " + text + "\n";
    el.scrollTop = el.scrollHeight;
}

connect();
</script>
</body>
</html>`
```

---

## 17. Key Takeaways

1. **WebSocket protocol basics:** WebSockets upgrade from HTTP via a handshake, then provide full-duplex communication over a single TCP connection using a lightweight frame format with opcodes for text, binary, ping/pong, and close.

2. **Library choice:** `gorilla/websocket` gives you low-level control and is battle-tested. `nhooyr.io/websocket` is more idiomatic Go with built-in context support and safe concurrent writes. Both are production-ready.

3. **Concurrent write safety:** In gorilla/websocket, only one goroutine may write at a time. Use a dedicated write goroutine with a channel (preferred) or protect writes with a mutex. nhooyr.io/websocket handles this internally.

4. **The Hub pattern is essential:** A central Hub goroutine manages client registration, unregistration, and broadcasting. This pattern scales to thousands of clients on a single server and is the foundation for chat, notifications, and dashboards.

5. **Ping/pong keeps connections alive:** Send periodic pings and set read deadlines. If a pong does not arrive in time, the connection is dead. Without this, you will accumulate ghost connections that waste resources.

6. **Authenticate before or during the upgrade:** Validate tokens, cookies, or credentials during the HTTP upgrade request -- before the connection is established. The first-message pattern works when HTTP-level auth is not possible, but introduces a brief unauthenticated window.

7. **Redis pub/sub scales horizontally:** When running multiple WebSocket server instances, use Redis pub/sub (or NATS, Kafka, etc.) to broadcast messages across all instances. Sticky sessions via `ip_hash` can supplement this for simpler cases.

8. **SSE is simpler for server-to-client:** If your application only pushes data from server to client (dashboards, notifications), SSE is simpler to implement, proxies love it, and the browser handles reconnection automatically with `Last-Event-ID` replay.

9. **Reconnection with exponential backoff and jitter:** Clients must reconnect automatically. Without jitter, a server restart causes a thundering herd of simultaneous reconnections.

10. **Production tuning matters:** Set appropriate buffer sizes, message size limits, per-client rate limits, connection limits, and OS-level file descriptor limits. Monitor goroutine count, memory usage, and connection metrics. Configure reverse proxies with adequate timeouts.

---

## 18. Practice Exercises

### Exercise 1: Typing Indicator
Add "user is typing..." indicators to the chat room server. When a user starts typing, broadcast a `typing` event to other clients in the room. Include a timeout so the indicator disappears if no messages are sent within 3 seconds.

### Exercise 2: Message History
Add persistent message history to the chat server using an in-memory ring buffer (or SQLite). When a client joins a room, send the last 50 messages. When a client reconnects, use a `lastMessageID` to send only missed messages.

### Exercise 3: Binary Protocol
Replace JSON messaging with Protocol Buffers. Define `.proto` messages for chat events, use binary WebSocket frames, and benchmark the throughput difference compared to JSON.

### Exercise 4: Presence System
Build a presence system that tracks which users are online in each room across multiple server instances. Use Redis sorted sets with heartbeat timestamps. Display the online user list in real time and handle stale entries when a server crashes.

### Exercise 5: File Transfer
Implement file transfer over WebSockets. The sender breaks a file into chunks (e.g., 16 KB each), sends them as binary frames with metadata (filename, chunk index, total chunks), and the receiver reassembles the file. Add progress reporting and resumable transfers.

### Exercise 6: WebSocket Gateway
Build a WebSocket-to-HTTP gateway. The WebSocket server receives JSON commands from clients (e.g., `{"method": "GET", "path": "/api/users"}`), forwards them as HTTP requests to a backend REST API, and streams the responses back over the WebSocket connection.

### Exercise 7: Load Test Analysis
Write a load test that measures p50, p95, and p99 latency for WebSocket messages. Start 1000 clients, have each send a timestamped message every second, measure round-trip time, and generate a histogram of latencies.

### Exercise 8: SSE with Replay
Implement an SSE server that stores the last 1000 events in a ring buffer. When a client reconnects with `Last-Event-ID`, replay all events since that ID. Write a test that simulates disconnection and verifies no events are lost.

### Exercise 9: Multi-Room Dashboard
Extend the live dashboard to support multiple "channels" (e.g., per-service metrics). Clients subscribe to specific channels and only receive metrics for those channels. Implement channel subscription via WebSocket messages.

### Exercise 10: Graceful Migration
Implement graceful WebSocket migration for zero-downtime deployments. When a server receives SIGTERM, it sends a custom "redirect" message to all connected clients with the URL of a healthy server instance. Clients disconnect and reconnect to the new server. Track migration progress and report when all clients have migrated.
