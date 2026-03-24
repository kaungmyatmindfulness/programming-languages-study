# Message Queues in Go: Kafka, NATS, and RabbitMQ

## Table of Contents

1. [Message Queue Fundamentals](#1-message-queue-fundamentals)
2. [Apache Kafka with Go](#2-apache-kafka-with-go)
3. [NATS and NATS JetStream](#3-nats-and-nats-jetstream)
4. [RabbitMQ with Go](#4-rabbitmq-with-go)
5. [Comparing Kafka vs NATS vs RabbitMQ](#5-comparing-kafka-vs-nats-vs-rabbitmq)
6. [Message Serialization](#6-message-serialization)
7. [Error Handling and Dead Letter Queues](#7-error-handling-and-dead-letter-queues)
8. [Idempotent Consumers](#8-idempotent-consumers)
9. [Graceful Shutdown of Consumers](#9-graceful-shutdown-of-consumers)
10. [Testing Message-Driven Systems](#10-testing-message-driven-systems)
11. [Production Monitoring and Observability](#11-production-monitoring-and-observability)

---

## 1. Message Queue Fundamentals

### What Is a Message Queue?

A message queue is middleware that enables asynchronous communication between services. Instead of
services calling each other directly (synchronous RPC), a producing service places a message onto a
queue, and one or more consuming services pick it up later. This decoupling provides:

- **Temporal decoupling**: Producer and consumer do not need to be online at the same time.
- **Spatial decoupling**: They do not need to know each other's network address.
- **Load leveling**: Bursts of messages are absorbed by the queue; consumers process at their own pace.
- **Fault isolation**: A failing consumer does not crash the producer.

### Core Concepts

#### Producers

A **producer** (or publisher) is any service that creates a message and sends it to the messaging
system. Producers are responsible for:

- Serializing domain events or commands into a wire format (JSON, Protobuf, Avro).
- Choosing a destination (topic, exchange, subject).
- Optionally choosing a partition key to control ordering.

#### Consumers

A **consumer** (or subscriber) reads messages from the messaging system and processes them. Key
responsibilities include:

- Deserializing the message.
- Processing the business logic.
- Acknowledging the message (so the broker knows it was handled).
- Handling failures (retries, dead-letter queues).

#### Topics and Queues

| Concept | Description |
|---------|-------------|
| **Topic** | A named category or feed of messages. Producers write to a topic; multiple consumers can independently read from it (fan-out). Used by Kafka and NATS. |
| **Queue** | A first-in-first-out buffer. Each message is delivered to exactly one consumer (competing consumers). Used by RabbitMQ. |
| **Subject** | NATS terminology for a topic. Supports hierarchical wildcards (`orders.*`, `orders.>`). |
| **Exchange** | RabbitMQ concept. Receives messages from producers and routes them to queues based on bindings and routing keys. |

#### Partitions

Partitions are a Kafka concept (and partially mirrored by NATS JetStream). A topic is split into
ordered, append-only logs called partitions. Benefits:

- **Parallelism**: Each partition can be consumed by a different consumer in the group.
- **Ordering**: Messages within a single partition are strictly ordered.
- **Scalability**: More partitions = more consumers working in parallel.

```
Topic: orders (3 partitions)

Partition 0: [msg1] [msg4] [msg7] ...
Partition 1: [msg2] [msg5] [msg8] ...
Partition 2: [msg3] [msg6] [msg9] ...
```

#### Consumer Groups

A **consumer group** is a set of consumers that cooperate to consume a topic. Each partition is
assigned to exactly one consumer in the group. If a consumer fails, its partitions are
**rebalanced** to remaining consumers.

```
Consumer Group "order-service"
  Consumer A -> Partition 0, Partition 1
  Consumer B -> Partition 2

(If Consumer B dies, Consumer A picks up Partition 2)
```

#### Message Delivery Guarantees

| Guarantee | Description |
|-----------|-------------|
| **At-most-once** | Message may be lost, but never delivered twice. Fire-and-forget. |
| **At-least-once** | Message is never lost, but may be delivered more than once. Requires idempotent consumers. |
| **Exactly-once** | Message is delivered and processed exactly once. Hard to achieve; requires broker + consumer cooperation. |

> **Tip**: In practice, most systems target **at-least-once** delivery combined with **idempotent
> consumers**. True exactly-once is expensive and often unnecessary.

---

## 2. Apache Kafka with Go

Apache Kafka is a distributed event streaming platform designed for high throughput, durability, and
horizontal scalability. Messages are persisted to disk and replicated across brokers.

### Go Client Libraries

| Library | Description | Pros | Cons |
|---------|-------------|------|------|
| `segmentio/kafka-go` | Pure Go implementation | No CGo dependency, easy cross-compilation | Slightly fewer features |
| `confluent-kafka-go` | Wrapper around librdkafka (C) | Battle-tested, full feature set | Requires CGo, harder to cross-compile |

### 2.1 Producer with segmentio/kafka-go

#### Installation

```bash
go get github.com/segmentio/kafka-go
```

#### Synchronous Producer

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    // Create a writer (producer) with explicit configuration
    writer := &kafka.Writer{
        Addr:         kafka.TCP("localhost:9092"),
        Topic:        "orders",
        Balancer:     &kafka.LeastBytes{}, // Partition balancing strategy
        RequiredAcks: kafka.RequireAll,     // Wait for all ISR replicas
        MaxAttempts:  3,                    // Retry up to 3 times
        BatchTimeout: 10 * time.Millisecond,
        WriteTimeout: 10 * time.Second,
    }
    defer writer.Close()

    // Produce a single message synchronously
    err := writer.WriteMessages(context.Background(),
        kafka.Message{
            Key:   []byte("order-123"),
            Value: []byte(`{"id":"order-123","amount":99.99,"status":"created"}`),
            Headers: []kafka.Header{
                {Key: "event-type", Value: []byte("OrderCreated")},
                {Key: "correlation-id", Value: []byte("abc-def-ghi")},
            },
            Time: time.Now(),
        },
    )
    if err != nil {
        log.Fatalf("failed to write message: %v", err)
    }
    fmt.Println("Message written successfully")
}
```

#### Asynchronous Producer with Batching

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    writer := &kafka.Writer{
        Addr:  kafka.TCP("localhost:9092"),
        Topic: "events",

        // Batching configuration
        BatchSize:    100,                   // Batch up to 100 messages
        BatchBytes:   1048576,               // Or up to 1MB
        BatchTimeout: 50 * time.Millisecond, // Or flush every 50ms

        // Compression reduces network I/O and disk usage
        Compression: kafka.Snappy, // Options: Gzip, Snappy, Lz4, Zstd

        // Async mode: WriteMessages returns immediately
        Async: true,

        // Completion callback for async writes
        Completion: func(messages []kafka.Message, err error) {
            if err != nil {
                log.Printf("async write failed for %d messages: %v",
                    len(messages), err)
                return
            }
            for _, msg := range messages {
                fmt.Printf("delivered: partition=%d offset=%d key=%s\n",
                    msg.Partition, msg.Offset, string(msg.Key))
            }
        },

        // Error logger
        ErrorLogger: kafka.LoggerFunc(func(msg string, args ...interface{}) {
            log.Printf("[kafka-error] "+msg, args...)
        }),
    }
    defer writer.Close()

    // Fire off many messages concurrently
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            err := writer.WriteMessages(context.Background(),
                kafka.Message{
                    Key:   []byte(fmt.Sprintf("key-%d", i%10)),
                    Value: []byte(fmt.Sprintf(`{"event_id":%d}`, i)),
                },
            )
            if err != nil {
                log.Printf("write error: %v", err)
            }
        }(i)
    }

    wg.Wait()
    fmt.Println("All messages submitted")
}
```

> **Warning**: In `Async` mode, `WriteMessages` returns `nil` even if the message has not been
> acknowledged by the broker yet. Use the `Completion` callback to track actual delivery.

#### Compression Comparison

| Algorithm | CPU Usage | Compression Ratio | Use Case |
|-----------|-----------|-------------------|----------|
| **None** | Lowest | 1:1 | Low-latency, small messages |
| **Gzip** | Highest | Best (~70% reduction) | Archival, bandwidth-constrained |
| **Snappy** | Low | Good (~50% reduction) | General purpose, balanced |
| **LZ4** | Lowest of compressed | Good (~55% reduction) | High-throughput, low-latency |
| **Zstd** | Medium | Excellent (~65% reduction) | Modern default, great ratio |

### 2.2 Consumer Groups, Offsets, and Rebalancing

#### Basic Consumer Group

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/segmentio/kafka-go"
)

func main() {
    // Create a reader (consumer) that belongs to a consumer group
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        GroupID: "order-processing-group", // Consumer group ID
        Topic:   "orders",

        // Start from the earliest message if no committed offset exists
        StartOffset: kafka.FirstOffset,

        // Commit offsets automatically every 5 seconds
        CommitInterval: 0, // 0 = manual commit (recommended)

        // Max bytes to fetch in a single request
        MaxBytes: 10e6, // 10MB

        // Session timeout for consumer group membership
        SessionTimeout: 30000, // 30 seconds (in milliseconds for kafka-go v0.4+)

        // Heartbeat interval
        HeartbeatInterval: 3000, // 3 seconds
    })
    defer reader.Close()

    // Handle graceful shutdown
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigCh
        fmt.Println("\nShutting down consumer...")
        cancel()
    }()

    fmt.Println("Consumer started, waiting for messages...")

    for {
        // FetchMessage does NOT auto-commit the offset
        msg, err := reader.FetchMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                break // Context cancelled, graceful shutdown
            }
            log.Printf("fetch error: %v", err)
            continue
        }

        fmt.Printf("Received: topic=%s partition=%d offset=%d key=%s value=%s\n",
            msg.Topic, msg.Partition, msg.Offset,
            string(msg.Key), string(msg.Value))

        // Process the message (your business logic here)
        if err := processOrder(msg.Value); err != nil {
            log.Printf("processing error for offset %d: %v", msg.Offset, err)
            // Decide: retry, send to DLQ, or skip
            continue
        }

        // Manually commit the offset AFTER successful processing
        // This ensures at-least-once delivery
        if err := reader.CommitMessages(ctx, msg); err != nil {
            log.Printf("commit error: %v", err)
        }
    }

    fmt.Println("Consumer stopped")
}

func processOrder(data []byte) error {
    // Your business logic here
    fmt.Printf("  Processing order: %s\n", string(data))
    return nil
}
```

#### Understanding Rebalancing

When consumers join or leave a group, Kafka **rebalances** partition assignments. During a
rebalance:

1. All consumers in the group pause consumption.
2. The group coordinator reassigns partitions.
3. Consumers resume from the last committed offset.

```go
// kafka-go handles rebalancing internally. You can observe it through logs.
// With confluent-kafka-go, you get explicit rebalance callbacks:

// Using confluent-kafka-go for fine-grained rebalance control:
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers":  "localhost:9092",
        "group.id":           "order-processing-group",
        "auto.offset.reset":  "earliest",
        "enable.auto.commit": false,

        // Partition assignment strategy
        "partition.assignment.strategy": "cooperative-sticky",

        // Session and heartbeat
        "session.timeout.ms":   30000,
        "heartbeat.interval.ms": 3000,

        // Max poll interval: if processing takes longer, consumer is evicted
        "max.poll.interval.ms": 300000, // 5 minutes
    })
    if err != nil {
        log.Fatalf("failed to create consumer: %v", err)
    }
    defer consumer.Close()

    // Subscribe with rebalance callback
    err = consumer.SubscribeTopics([]string{"orders"}, func(c *kafka.Consumer, event kafka.Event) error {
        switch e := event.(type) {
        case kafka.AssignedPartitions:
            fmt.Printf("Rebalance: assigned partitions: %v\n", e.Partitions)
            // Optionally seek to specific offsets here
            return c.Assign(e.Partitions)

        case kafka.RevokedPartitions:
            fmt.Printf("Rebalance: revoked partitions: %v\n", e.Partitions)
            // Commit offsets for revoked partitions before giving them up
            commitOffsets, _ := c.Committed(e.Partitions, 5000)
            fmt.Printf("  Committed offsets: %v\n", commitOffsets)
            return c.Unassign()
        }
        return nil
    })
    if err != nil {
        log.Fatalf("subscribe error: %v", err)
    }

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    run := true
    for run {
        select {
        case <-sigCh:
            run = false
        default:
            ev := consumer.Poll(100) // 100ms timeout
            if ev == nil {
                continue
            }
            switch e := ev.(type) {
            case *kafka.Message:
                fmt.Printf("Received: %s = %s [partition %d, offset %d]\n",
                    string(e.Key), string(e.Value),
                    e.TopicPartition.Partition, e.TopicPartition.Offset)

                // Process message...

                // Commit offset
                _, err := consumer.CommitMessage(e)
                if err != nil {
                    log.Printf("commit error: %v", err)
                }

            case kafka.Error:
                log.Printf("Kafka error: %v\n", e)
                if e.Code() == kafka.ErrAllBrokersDown {
                    run = false
                }
            }
        }
    }
}
```

### 2.3 Exactly-Once Semantics (EOS) in Kafka

Kafka supports exactly-once semantics through **idempotent producers** and **transactions**.

#### Idempotent Producer

An idempotent producer ensures that retries do not result in duplicate messages. The broker
deduplicates based on producer ID and sequence number.

```go
// With confluent-kafka-go:
producer, err := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers": "localhost:9092",
    "enable.idempotence": true,          // Enable idempotent producer
    "acks":               "all",          // Required for idempotence
    "max.in.flight.requests.per.connection": 5, // Max 5 for idempotence
    "retries":            2147483647,     // Max retries
})
```

#### Transactional Producer (Read-Process-Write)

```go
package main

import (
    "fmt"
    "log"

    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
    // Create a transactional producer
    producer, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "transactional.id":  "order-processor-txn-1", // Unique per instance
        "enable.idempotence": true,
        "acks":               "all",
    })
    if err != nil {
        log.Fatalf("failed to create producer: %v", err)
    }
    defer producer.Close()

    // Initialize transactions (call once at startup)
    if err := producer.InitTransactions(nil); err != nil {
        log.Fatalf("init transactions failed: %v", err)
    }

    // Create a consumer for the read side
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers":  "localhost:9092",
        "group.id":           "order-processor",
        "auto.offset.reset":  "earliest",
        "enable.auto.commit": false,
        "isolation.level":    "read_committed", // Only read committed messages
    })
    if err != nil {
        log.Fatalf("failed to create consumer: %v", err)
    }
    defer consumer.Close()

    consumer.SubscribeTopics([]string{"raw-orders"}, nil)

    for {
        ev := consumer.Poll(100)
        if ev == nil {
            continue
        }

        msg, ok := ev.(*kafka.Message)
        if !ok {
            continue
        }

        // Begin transaction
        if err := producer.BeginTransaction(); err != nil {
            log.Printf("begin transaction error: %v", err)
            continue
        }

        // Process the message and produce output
        outputValue := fmt.Sprintf(`{"processed": %s}`, string(msg.Value))
        outputTopic := "processed-orders"

        err = producer.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{
                Topic:     &outputTopic,
                Partition: kafka.PartitionAny,
            },
            Key:   msg.Key,
            Value: []byte(outputValue),
        }, nil)
        if err != nil {
            producer.AbortTransaction(nil)
            log.Printf("produce error, aborting transaction: %v", err)
            continue
        }

        // Send consumer offsets as part of the transaction
        // This is the key to exactly-once: offset commit + produce are atomic
        consumerMetadata, err := consumer.GetConsumerGroupMetadata()
        if err != nil {
            producer.AbortTransaction(nil)
            log.Printf("get metadata error: %v", err)
            continue
        }

        offsets := []kafka.TopicPartition{msg.TopicPartition}
        offsets[0].Offset++ // Commit the NEXT offset

        err = producer.SendOffsetsToTransaction(nil, offsets, consumerMetadata)
        if err != nil {
            producer.AbortTransaction(nil)
            log.Printf("send offsets error: %v", err)
            continue
        }

        // Commit the transaction (atomic: produce + offset commit)
        if err := producer.CommitTransaction(nil); err != nil {
            log.Printf("commit transaction error: %v", err)
            producer.AbortTransaction(nil)
            continue
        }

        fmt.Printf("Transaction committed for offset %d\n", msg.TopicPartition.Offset)
    }
}
```

> **Warning**: Exactly-once semantics only apply within the Kafka ecosystem (Kafka-to-Kafka). If
> your consumer writes to an external database, you need application-level idempotency.

### 2.4 Topic Administration

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "strconv"

    "github.com/segmentio/kafka-go"
)

func main() {
    // Connect to the controller broker
    conn, err := kafka.Dial("tcp", "localhost:9092")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    controller, err := conn.Controller()
    if err != nil {
        log.Fatal(err)
    }

    controllerConn, err := kafka.Dial("tcp",
        net.JoinHostPort(controller.Host, strconv.Itoa(controller.Port)))
    if err != nil {
        log.Fatal(err)
    }
    defer controllerConn.Close()

    // Create a topic
    err = controllerConn.CreateTopics(
        kafka.TopicConfig{
            Topic:             "orders",
            NumPartitions:     6,
            ReplicationFactor: 3,
            ConfigEntries: []kafka.ConfigEntry{
                {ConfigName: "retention.ms", ConfigValue: "604800000"}, // 7 days
                {ConfigName: "cleanup.policy", ConfigValue: "delete"},
                {ConfigName: "min.insync.replicas", ConfigValue: "2"},
            },
        },
    )
    if err != nil {
        log.Printf("create topic error: %v", err)
    }

    // List topics
    partitions, err := controllerConn.ReadPartitions()
    if err != nil {
        log.Fatal(err)
    }

    topicMap := make(map[string]int)
    for _, p := range partitions {
        topicMap[p.Topic]++
    }

    for topic, count := range topicMap {
        fmt.Printf("Topic: %s, Partitions: %d\n", topic, count)
    }

    // Delete a topic
    err = controllerConn.DeleteTopics("old-topic")
    if err != nil {
        log.Printf("delete topic error: %v", err)
    }

    // Check consumer group lag using the Reader stats
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        GroupID: "order-processing-group",
        Topic:   "orders",
    })
    defer reader.Close()

    stats := reader.Stats()
    fmt.Printf("Consumer lag: %d messages\n", stats.Lag)
    fmt.Printf("Messages read: %d\n", stats.Messages)

    _ = context.Background() // placeholder for extended operations
}
```

---

## 3. NATS and NATS JetStream

NATS is a lightweight, high-performance messaging system designed for cloud-native applications.
It has two layers:

- **Core NATS**: Pure pub/sub with at-most-once delivery. Extremely fast.
- **JetStream**: Persistence layer built on top of core NATS. Provides at-least-once and
  exactly-once delivery, streams, consumers, and key-value stores.

### Installation

```bash
go get github.com/nats-io/nats.go
```

### 3.1 Core NATS

#### Basic Publish/Subscribe

```go
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/nats-io/nats.go"
)

func main() {
    // Connect to NATS
    nc, err := nats.Connect(
        "nats://localhost:4222",
        nats.Name("order-service"),           // Client name for monitoring
        nats.ReconnectWait(2*time.Second),    // Wait 2s between reconnect attempts
        nats.MaxReconnects(60),               // Try 60 times before giving up
        nats.PingInterval(20*time.Second),    // Ping server every 20s
        nats.MaxPingsOutstanding(3),          // Disconnect after 3 missed pongs
        nats.DisconnectErrHandler(func(nc *nats.Conn, err error) {
            log.Printf("Disconnected: %v", err)
        }),
        nats.ReconnectHandler(func(nc *nats.Conn) {
            log.Printf("Reconnected to %s", nc.ConnectedUrl())
        }),
        nats.ClosedHandler(func(nc *nats.Conn) {
            log.Printf("Connection closed: %v", nc.LastError())
        }),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    // Subscribe to a subject
    sub, err := nc.Subscribe("orders.created", func(msg *nats.Msg) {
        fmt.Printf("Received on %s: %s\n", msg.Subject, string(msg.Data))
    })
    if err != nil {
        log.Fatal(err)
    }
    defer sub.Unsubscribe()

    // Wildcard subscriptions
    // '*' matches a single token: orders.* matches orders.created, orders.updated
    nc.Subscribe("orders.*", func(msg *nats.Msg) {
        fmt.Printf("Wildcard match on %s: %s\n", msg.Subject, string(msg.Data))
    })

    // '>' matches one or more tokens: orders.> matches orders.created, orders.us.east
    nc.Subscribe("orders.>", func(msg *nats.Msg) {
        fmt.Printf("Catch-all on %s: %s\n", msg.Subject, string(msg.Data))
    })

    // Publish a message
    err = nc.Publish("orders.created", []byte(`{"id":"order-123","amount":99.99}`))
    if err != nil {
        log.Fatal(err)
    }

    // Flush to ensure all published messages have been sent
    nc.Flush()
    if err := nc.LastError(); err != nil {
        log.Fatal(err)
    }

    time.Sleep(1 * time.Second) // Wait for messages to arrive
}
```

#### Request/Reply Pattern

```go
package main

import (
    "fmt"
    "log"
    "time"

    "github.com/nats-io/nats.go"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    // --- Responder (service) ---
    nc.Subscribe("inventory.check", func(msg *nats.Msg) {
        fmt.Printf("Received request: %s\n", string(msg.Data))

        // Simulate inventory check
        response := fmt.Sprintf(`{"product_id":"prod-1","available":true,"quantity":42}`)

        // Reply to the requestor
        if err := msg.Respond([]byte(response)); err != nil {
            log.Printf("respond error: %v", err)
        }
    })

    // --- Requestor (client) ---
    // Synchronous request with timeout
    reply, err := nc.Request("inventory.check",
        []byte(`{"product_id":"prod-1"}`),
        2*time.Second, // Timeout
    )
    if err != nil {
        if err == nats.ErrTimeout {
            log.Println("Request timed out -- is the responder running?")
        } else {
            log.Fatal(err)
        }
    } else {
        fmt.Printf("Reply: %s\n", string(reply.Data))
    }
}
```

#### Queue Groups (Load Balancing)

Queue groups allow multiple subscribers to share the load. Each message is delivered to exactly
one subscriber in the group.

```go
package main

import (
    "fmt"
    "log"
    "sync"
    "time"

    "github.com/nats-io/nats.go"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    var wg sync.WaitGroup

    // Start 3 competing consumers in queue group "workers"
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        workerID := i

        // QueueSubscribe: only ONE subscriber in the group gets each message
        nc.QueueSubscribe("tasks.process", "workers", func(msg *nats.Msg) {
            fmt.Printf("Worker %d processing: %s\n", workerID, string(msg.Data))
        })
    }

    // Publish 10 tasks -- they will be distributed among the 3 workers
    for i := 0; i < 10; i++ {
        nc.Publish("tasks.process", []byte(fmt.Sprintf(`{"task_id":%d}`, i)))
    }
    nc.Flush()

    time.Sleep(1 * time.Second)
    fmt.Println("All tasks distributed")
}
```

### 3.2 NATS JetStream

JetStream adds persistence, replay, and exactly-once semantics on top of core NATS.

#### Creating Streams and Publishing

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    // Create JetStream context (new API as of nats.go v1.31+)
    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()

    // Create or update a stream
    stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
        Name:     "ORDERS",
        Subjects: []string{"orders.>"},  // Capture all order subjects

        // Storage: file (persistent) or memory (fast but volatile)
        Storage: jetstream.FileStorage,

        // Retention policy
        Retention: jetstream.LimitsPolicy, // Keep messages until limits are hit
        // Alternatives:
        // jetstream.InterestPolicy - delete when no consumers are interested
        // jetstream.WorkQueuePolicy - delete after acknowledgment

        // Limits
        MaxMsgs:          1000000,          // Max messages in stream
        MaxBytes:          1073741824,       // Max 1GB
        MaxAge:            7 * 24 * time.Hour, // Max age: 7 days
        MaxMsgsPerSubject: 100000,          // Max per subject

        // Replication (for clustered NATS)
        Replicas: 3,

        // Deduplication window
        Duplicates: 2 * time.Minute,

        // Discard policy when limits are reached
        Discard: jetstream.DiscardOld, // Discard oldest messages
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Stream created: %s\n", stream.CachedInfo().Config.Name)

    // Publish messages with acknowledgment
    ack, err := js.Publish(ctx, "orders.created",
        []byte(`{"id":"order-123","amount":99.99}`),
        jetstream.WithMsgID("order-123-created"), // For deduplication
    )
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Published: stream=%s seq=%d\n", ack.Stream, ack.Sequence)

    // Publish with headers
    msg := nats.NewMsg("orders.updated")
    msg.Data = []byte(`{"id":"order-123","status":"shipped"}`)
    msg.Header.Set("Nats-Msg-Id", "order-123-shipped") // Dedup ID
    msg.Header.Set("Content-Type", "application/json")
    msg.Header.Set("X-Correlation-Id", "abc-def")

    ack, err = js.PublishMsg(ctx, msg)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Published with headers: seq=%d duplicate=%v\n",
        ack.Sequence, ack.Duplicate)

    // Async publish for higher throughput
    futures := make([]jetstream.PubAckFuture, 0, 100)
    for i := 0; i < 100; i++ {
        future, err := js.PublishAsync(
            "orders.created",
            []byte(fmt.Sprintf(`{"id":"order-%d"}`, i)),
        )
        if err != nil {
            log.Printf("async publish error: %v", err)
            continue
        }
        futures = append(futures, future)
    }

    // Wait for all async publishes to be acknowledged
    select {
    case <-js.PublishAsyncComplete():
        fmt.Println("All async publishes acknowledged")
    case <-time.After(10 * time.Second):
        log.Println("Timed out waiting for async publishes")
    }

    // Check each future for errors
    for _, f := range futures {
        select {
        case ack := <-f.Ok():
            _ = ack // Successfully published
        case err := <-f.Err():
            log.Printf("async publish failed: %v", err)
        }
    }
}
```

#### Consumers and Durable Subscriptions

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Create a durable consumer
    // Durable consumers survive restarts and remember their position
    consumer, err := js.CreateOrUpdateConsumer(ctx, "ORDERS", jetstream.ConsumerConfig{
        Name:    "order-processor",
        Durable: "order-processor", // Durable name survives reconnects

        // Delivery policy: where to start consuming
        DeliverPolicy: jetstream.DeliverAllPolicy, // From the beginning
        // Alternatives:
        // jetstream.DeliverNewPolicy       - only new messages
        // jetstream.DeliverLastPolicy       - last message per subject
        // jetstream.DeliverByStartSequencePolicy - from a specific sequence
        // jetstream.DeliverByStartTimePolicy     - from a specific time

        // Acknowledgment policy
        AckPolicy: jetstream.AckExplicitPolicy, // Must explicitly ack
        AckWait:   30 * time.Second,             // Redeliver if not acked within 30s

        // Max delivery attempts before sending to dead letter
        MaxDeliver: 5,

        // Filter to specific subjects
        FilterSubject: "orders.created",

        // Max number of unacknowledged messages
        MaxAckPending: 1000,

        // Replay policy
        ReplayPolicy: jetstream.ReplayInstantPolicy, // Deliver as fast as possible

        // Backoff for redeliveries
        BackOff: []time.Duration{
            1 * time.Second,
            5 * time.Second,
            30 * time.Second,
            1 * time.Minute,
            5 * time.Minute,
        },
    })
    if err != nil {
        log.Fatal(err)
    }

    // --- Pull-based consumption (recommended for most use cases) ---
    fmt.Println("Starting pull consumer...")

    // Consume with auto-fetch
    consumeCtx, err := consumer.Consume(func(msg jetstream.Msg) {
        metadata, _ := msg.Metadata()
        fmt.Printf("Received: subject=%s seq=%d delivered=%d data=%s\n",
            msg.Subject(),
            metadata.Sequence.Stream,
            metadata.NumDelivered,
            string(msg.Data()))

        // Process the message
        if err := processMessage(msg.Data()); err != nil {
            // Negative acknowledge: message will be redelivered
            // with backoff delay
            msg.Nak()
            return
        }

        // Acknowledge successful processing
        msg.Ack()

        // Other acknowledgment options:
        // msg.NakWithDelay(5 * time.Second)  // Nak with custom delay
        // msg.InProgress()                    // Reset ack timer (still processing)
        // msg.Term()                          // Terminate: do not redeliver
    }, jetstream.PullMaxMessages(10)) // Fetch 10 at a time
    if err != nil {
        log.Fatal(err)
    }
    defer consumeCtx.Stop()

    // --- Alternative: Batch fetch ---
    go func() {
        for {
            // Fetch a batch of messages
            batch, err := consumer.Fetch(10, jetstream.FetchMaxWait(5*time.Second))
            if err != nil {
                log.Printf("fetch error: %v", err)
                continue
            }

            for msg := range batch.Messages() {
                fmt.Printf("Batch received: %s\n", string(msg.Data()))
                msg.Ack()
            }

            if batch.Error() != nil {
                log.Printf("batch error: %v", batch.Error())
            }
        }
    }()

    // Wait for shutdown signal
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh
    cancel()
    fmt.Println("Consumer stopped")
}

func processMessage(data []byte) error {
    fmt.Printf("  Processing: %s\n", string(data))
    return nil
}
```

#### Key-Value Store

JetStream includes a built-in distributed key-value store, useful for configuration, caching, and
lightweight state management.

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()

    // Create a key-value bucket
    kv, err := js.CreateOrUpdateKeyValue(ctx, jetstream.KeyValueConfig{
        Bucket:      "service-config",
        Description: "Service configuration store",
        History:     10, // Keep 10 revisions per key
        TTL:         0,  // No expiry (0 = infinite)
    })
    if err != nil {
        log.Fatal(err)
    }

    // Put a value
    rev, err := kv.Put(ctx, "database.url", []byte("postgres://localhost:5432/mydb"))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Put: revision=%d\n", rev)

    // Get a value
    entry, err := kv.Get(ctx, "database.url")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Get: key=%s value=%s revision=%d\n",
        entry.Key(), string(entry.Value()), entry.Revision())

    // Update with optimistic concurrency (CAS - Compare And Swap)
    newRev, err := kv.Update(ctx, "database.url",
        []byte("postgres://primary:5432/mydb"), entry.Revision())
    if err != nil {
        log.Fatal(err) // Fails if revision has changed
    }
    fmt.Printf("Updated: revision=%d\n", newRev)

    // Create (fails if key already exists)
    _, err = kv.Create(ctx, "feature.new-ui", []byte("true"))
    if err != nil {
        log.Printf("create error (may already exist): %v", err)
    }

    // Watch for changes
    watcher, err := kv.WatchAll(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer watcher.Stop()

    // Read initial values and subsequent updates
    go func() {
        for entry := range watcher.Updates() {
            if entry == nil {
                fmt.Println("  Initial values loaded")
                continue
            }
            fmt.Printf("  Watch: key=%s value=%s op=%s\n",
                entry.Key(), string(entry.Value()), entry.Operation())
        }
    }()

    // Delete a key
    err = kv.Delete(ctx, "feature.new-ui")
    if err != nil {
        log.Printf("delete error: %v", err)
    }

    // List all keys
    keys, err := kv.ListKeys(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Keys in bucket:")
    for key := range keys.Keys() {
        fmt.Printf("  - %s\n", key)
    }

    // History for a key
    history, err := kv.History(ctx, "database.url")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("History for database.url:")
    for _, entry := range history {
        fmt.Printf("  rev=%d value=%s\n", entry.Revision(), string(entry.Value()))
    }
}
```

#### Object Store

JetStream also provides an object store for larger binary blobs.

```go
package main

import (
    "bytes"
    "context"
    "fmt"
    "io"
    "log"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()

    // Create an object store bucket
    obs, err := js.CreateOrUpdateObjectStore(ctx, jetstream.ObjectStoreConfig{
        Bucket:      "attachments",
        Description: "Order attachments",
        MaxBytes:    1073741824, // 1GB max
    })
    if err != nil {
        log.Fatal(err)
    }

    // Store an object
    data := []byte("This is a large document content...")
    _, err = obs.Put(ctx, jetstream.ObjectMeta{
        Name:        "order-123/invoice.pdf",
        Description: "Invoice for order 123",
        Headers:     nats.Header{"Content-Type": {"application/pdf"}},
    }, bytes.NewReader(data))
    if err != nil {
        log.Fatal(err)
    }

    // Retrieve an object
    result, err := obs.Get(ctx, "order-123/invoice.pdf")
    if err != nil {
        log.Fatal(err)
    }
    defer result.Close()

    content, err := io.ReadAll(result)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Retrieved: %s\n", string(content))

    // Get object info
    info, err := obs.GetInfo(ctx, "order-123/invoice.pdf")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Object: name=%s size=%d\n", info.Name, info.Size)
}
```

---

## 4. RabbitMQ with Go

RabbitMQ is a traditional message broker implementing the AMQP 0-9-1 protocol. It excels at
complex routing, message acknowledgments, and plugin-based extensibility.

### Installation

```bash
go get github.com/rabbitmq/amqp091-go
```

### 4.1 Connection Management

```go
package main

import (
    "fmt"
    "log"
    "sync"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

// RabbitMQ recommends ONE connection per application, with multiple channels.
// Connections are expensive (TCP + TLS + AMQP handshake).
// Channels are lightweight and can be created/destroyed freely.

// ConnectionManager handles RabbitMQ connections with auto-reconnect.
type ConnectionManager struct {
    url        string
    conn       *amqp.Connection
    mu         sync.RWMutex
    closeCh    chan struct{}
    reconnectCh chan struct{}
}

func NewConnectionManager(url string) *ConnectionManager {
    cm := &ConnectionManager{
        url:         url,
        closeCh:     make(chan struct{}),
        reconnectCh: make(chan struct{}, 1),
    }
    return cm
}

func (cm *ConnectionManager) Connect() error {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    conn, err := amqp.DialConfig(cm.url, amqp.Config{
        Heartbeat: 10 * time.Second,
        Locale:    "en_US",
        Properties: amqp.Table{
            "connection_name": "order-service",
        },
    })
    if err != nil {
        return fmt.Errorf("dial: %w", err)
    }

    cm.conn = conn

    // Monitor connection closure and trigger reconnect
    go func() {
        closeErr := <-conn.NotifyClose(make(chan *amqp.Error, 1))
        if closeErr != nil {
            log.Printf("Connection closed: %v", closeErr)
            cm.reconnect()
        }
    }()

    log.Println("Connected to RabbitMQ")
    return nil
}

func (cm *ConnectionManager) reconnect() {
    for {
        select {
        case <-cm.closeCh:
            return
        default:
        }

        log.Println("Attempting to reconnect...")
        if err := cm.Connect(); err != nil {
            log.Printf("Reconnect failed: %v, retrying in 5s", err)
            time.Sleep(5 * time.Second)
            continue
        }

        // Signal that reconnection happened
        select {
        case cm.reconnectCh <- struct{}{}:
        default:
        }
        return
    }
}

func (cm *ConnectionManager) Channel() (*amqp.Channel, error) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()

    if cm.conn == nil || cm.conn.IsClosed() {
        return nil, fmt.Errorf("connection is closed")
    }
    return cm.conn.Channel()
}

func (cm *ConnectionManager) Close() {
    close(cm.closeCh)
    cm.mu.Lock()
    defer cm.mu.Unlock()
    if cm.conn != nil && !cm.conn.IsClosed() {
        cm.conn.Close()
    }
}
```

### 4.2 Exchanges, Queues, Bindings, and Routing Keys

RabbitMQ's routing model is built around three concepts:

```
Producer --> Exchange --[binding/routing key]--> Queue --> Consumer
```

#### Exchange Types

| Exchange Type | Routing Behavior |
|---------------|-----------------|
| **direct** | Routes to queues whose binding key exactly matches the routing key. |
| **fanout** | Routes to ALL bound queues (ignores routing key). Like broadcast. |
| **topic** | Routes based on wildcard matching. `*` = one word, `#` = zero or more words. |
| **headers** | Routes based on message headers instead of routing key. |

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    // --- Declare Exchanges ---

    // Direct exchange
    err = ch.ExchangeDeclare(
        "orders.direct", // name
        "direct",        // type
        true,            // durable (survives broker restart)
        false,           // auto-delete
        false,           // internal
        false,           // no-wait
        nil,             // arguments
    )
    if err != nil {
        log.Fatal(err)
    }

    // Topic exchange
    err = ch.ExchangeDeclare(
        "events.topic",
        "topic",
        true,
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // Fanout exchange (for broadcasting)
    err = ch.ExchangeDeclare(
        "notifications.fanout",
        "fanout",
        true,
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // --- Declare Queues ---

    // Durable queue for order processing
    orderQueue, err := ch.QueueDeclare(
        "order-processing", // name
        true,               // durable
        false,              // auto-delete
        false,              // exclusive (single consumer only)
        false,              // no-wait
        amqp.Table{
            "x-max-length":          10000,             // Max messages in queue
            "x-message-ttl":         int32(86400000),   // Message TTL: 24 hours
            "x-queue-type":          "quorum",          // Quorum queue (replicated)
            "x-dead-letter-exchange": "dlx.orders",     // Dead letter exchange
            "x-dead-letter-routing-key": "orders.dead", // Dead letter routing key
        },
    )
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Queue declared: %s (%d messages)\n", orderQueue.Name, orderQueue.Messages)

    // --- Bind Queues to Exchanges ---

    // Direct binding: route "order.created" messages to order-processing queue
    err = ch.QueueBind(
        "order-processing", // queue
        "order.created",    // routing key (binding key)
        "orders.direct",    // exchange
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // Topic binding: route all order events to order-processing
    err = ch.QueueBind(
        "order-processing",
        "orders.*",       // Wildcard: orders.created, orders.updated, etc.
        "events.topic",
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // Another topic binding: route all US region events
    err = ch.QueueBind(
        "order-processing",
        "orders.*.us.#",  // orders.created.us, orders.created.us.east, etc.
        "events.topic",
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    // --- Publish Messages ---

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Publish to direct exchange
    err = ch.PublishWithContext(ctx,
        "orders.direct",  // exchange
        "order.created",  // routing key
        true,             // mandatory (return if unroutable)
        false,            // immediate (deprecated, use false)
        amqp.Publishing{
            ContentType:  "application/json",
            DeliveryMode: amqp.Persistent, // Survive broker restart
            Timestamp:    time.Now(),
            MessageId:    "msg-001",
            CorrelationId: "corr-abc",
            Headers: amqp.Table{
                "event-type": "OrderCreated",
                "version":    int32(1),
            },
            Body: []byte(`{"id":"order-123","amount":99.99}`),
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    // Publish to topic exchange
    err = ch.PublishWithContext(ctx,
        "events.topic",
        "orders.updated.us.east",
        false,
        false,
        amqp.Publishing{
            ContentType:  "application/json",
            DeliveryMode: amqp.Persistent,
            Body:         []byte(`{"id":"order-123","status":"shipped"}`),
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    // Publish to fanout exchange (routing key is ignored)
    err = ch.PublishWithContext(ctx,
        "notifications.fanout",
        "",    // routing key ignored for fanout
        false,
        false,
        amqp.Publishing{
            ContentType: "application/json",
            Body:        []byte(`{"message":"System maintenance at midnight"}`),
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Messages published successfully")
}
```

### 4.3 Dead Letter Queues

Dead letter queues (DLQs) capture messages that cannot be processed successfully.

Messages are dead-lettered when:
- The consumer rejects them (nack with `requeue: false`).
- The message TTL expires.
- The queue reaches its maximum length.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

func setupDeadLetterInfrastructure(ch *amqp.Channel) error {
    // Step 1: Create the dead letter exchange
    err := ch.ExchangeDeclare(
        "dlx.orders",   // Dead letter exchange name
        "direct",
        true,
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        return fmt.Errorf("declare DLX: %w", err)
    }

    // Step 2: Create the dead letter queue
    _, err = ch.QueueDeclare(
        "orders.dead-letter", // DLQ name
        true,
        false,
        false,
        false,
        amqp.Table{
            // Optionally, set a TTL on dead letters for automatic cleanup
            "x-message-ttl": int32(604800000), // 7 days
        },
    )
    if err != nil {
        return fmt.Errorf("declare DLQ: %w", err)
    }

    // Step 3: Bind DLQ to DLX
    err = ch.QueueBind(
        "orders.dead-letter",
        "orders.dead",   // routing key
        "dlx.orders",    // exchange
        false,
        nil,
    )
    if err != nil {
        return fmt.Errorf("bind DLQ: %w", err)
    }

    // Step 4: Create the main queue with DLX configuration
    _, err = ch.QueueDeclare(
        "orders.processing",
        true,
        false,
        false,
        false,
        amqp.Table{
            "x-dead-letter-exchange":    "dlx.orders",
            "x-dead-letter-routing-key": "orders.dead",
            // Messages are dead-lettered after 3 delivery attempts
            // (handled in consumer logic, not a native RabbitMQ feature
            //  for classic queues; quorum queues support delivery-limit)
            "x-queue-type":     "quorum",
            "x-delivery-limit": int32(3), // Quorum queue feature
        },
    )
    if err != nil {
        return fmt.Errorf("declare main queue: %w", err)
    }

    return nil
}

// DLQ consumer for manual inspection and retry
func consumeDeadLetters(ch *amqp.Channel) {
    msgs, err := ch.Consume(
        "orders.dead-letter",
        "dlq-inspector",
        false, // manual ack
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    for msg := range msgs {
        // Inspect dead letter headers
        xDeath, ok := msg.Headers["x-death"].([]interface{})
        if ok && len(xDeath) > 0 {
            deathInfo := xDeath[0].(amqp.Table)
            fmt.Printf("Dead letter info:\n")
            fmt.Printf("  Queue: %s\n", deathInfo["queue"])
            fmt.Printf("  Reason: %s\n", deathInfo["reason"])
            fmt.Printf("  Count: %d\n", deathInfo["count"])
            fmt.Printf("  Exchange: %s\n", deathInfo["exchange"])
            fmt.Printf("  Routing keys: %v\n", deathInfo["routing-keys"])
            fmt.Printf("  Time: %v\n", deathInfo["time"])
        }

        fmt.Printf("Dead letter body: %s\n", string(msg.Body))

        // Options:
        // 1. Log and acknowledge (discard)
        // 2. Republish to original queue (retry)
        // 3. Store in database for manual investigation
        // 4. Send alert to operations team

        msg.Ack(false)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    if err := setupDeadLetterInfrastructure(ch); err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()
    _ = ctx

    go consumeDeadLetters(ch)

    // Keep running
    select {}
}
```

### 4.4 Publisher Confirms

Publisher confirms ensure the broker has received and persisted the message.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    // Enable publisher confirms on this channel
    if err := ch.Confirm(false); err != nil {
        log.Fatalf("channel could not be put into confirm mode: %v", err)
    }

    // --- Strategy 1: Individual confirms (simple, slower) ---
    confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1))

    ctx := context.Background()
    err = ch.PublishWithContext(ctx,
        "",                 // default exchange
        "order-processing", // routing key = queue name
        false,
        false,
        amqp.Publishing{
            DeliveryMode: amqp.Persistent,
            Body:         []byte(`{"id":"order-1"}`),
        },
    )
    if err != nil {
        log.Fatal(err)
    }

    // Wait for confirm
    select {
    case confirm := <-confirms:
        if confirm.Ack {
            fmt.Printf("Message confirmed: delivery tag=%d\n", confirm.DeliveryTag)
        } else {
            fmt.Printf("Message NACKED: delivery tag=%d\n", confirm.DeliveryTag)
        }
    case <-time.After(5 * time.Second):
        log.Println("Confirm timeout")
    }

    // --- Strategy 2: Batch confirms (balanced) ---
    batchConfirms := ch.NotifyPublish(make(chan amqp.Confirmation, 100))

    for i := 0; i < 100; i++ {
        ch.PublishWithContext(ctx,
            "",
            "order-processing",
            false,
            false,
            amqp.Publishing{
                DeliveryMode: amqp.Persistent,
                Body:         []byte(fmt.Sprintf(`{"id":"order-%d"}`, i)),
            },
        )
    }

    // Collect all confirms
    confirmed := 0
    nacked := 0
    timeout := time.After(10 * time.Second)

    for confirmed+nacked < 100 {
        select {
        case confirm := <-batchConfirms:
            if confirm.Ack {
                confirmed++
            } else {
                nacked++
            }
        case <-timeout:
            log.Printf("Timeout: confirmed=%d nacked=%d pending=%d",
                confirmed, nacked, 100-confirmed-nacked)
            break
        }
    }

    fmt.Printf("Batch result: confirmed=%d nacked=%d\n", confirmed, nacked)

    // --- Strategy 3: Async confirms with tracking (fastest) ---
    asyncConfirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1000))
    pendingConfirms := &sync.Map{} // delivery tag -> message for retry

    // Background goroutine to process confirms
    go func() {
        for confirm := range asyncConfirms {
            if confirm.Ack {
                pendingConfirms.Delete(confirm.DeliveryTag)
            } else {
                // Message was nacked; retrieve and retry
                if msg, ok := pendingConfirms.LoadAndDelete(confirm.DeliveryTag); ok {
                    log.Printf("NACK for tag %d, retrying: %s",
                        confirm.DeliveryTag, msg)
                    // Retry logic here
                }
            }
        }
    }()

    // Publish with tracking
    for i := 0; i < 1000; i++ {
        body := fmt.Sprintf(`{"id":"order-%d"}`, i)

        // Get the next delivery tag (channel sequence number)
        // Note: in practice you'd track this more carefully
        ch.PublishWithContext(ctx,
            "",
            "order-processing",
            false,
            false,
            amqp.Publishing{
                DeliveryMode: amqp.Persistent,
                Body:         []byte(body),
            },
        )
        pendingConfirms.Store(uint64(i+1), body) // Tags start at 1
    }

    time.Sleep(5 * time.Second) // Wait for confirms to arrive
    fmt.Println("Async publishing complete")
}
```

### 4.5 Consumer Acknowledgments

```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()

    // Set QoS (prefetch count)
    // This limits how many unacknowledged messages the consumer holds at once.
    // Critical for:
    // - Preventing one slow consumer from hogging all messages
    // - Enabling fair dispatch among multiple consumers
    err = ch.Qos(
        10,    // prefetch count: max 10 unacked messages at a time
        0,     // prefetch size: 0 = no limit on total byte size
        false, // global: false = per-consumer, true = per-channel
    )
    if err != nil {
        log.Fatal(err)
    }

    msgs, err := ch.Consume(
        "order-processing", // queue
        "consumer-1",       // consumer tag (unique identifier)
        false,              // auto-ack: false = manual acknowledgment
        false,              // exclusive: false = allow multiple consumers
        false,              // no-local: not supported by RabbitMQ
        false,              // no-wait
        nil,                // args
    )
    if err != nil {
        log.Fatal(err)
    }

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        for msg := range msgs {
            fmt.Printf("Received: %s (delivery tag: %d)\n",
                string(msg.Body), msg.DeliveryTag)

            err := processMessage(msg.Body)

            if err == nil {
                // ACK: message processed successfully, remove from queue
                msg.Ack(false) // false = ack only this message
                // msg.Ack(true)  // true = ack this AND all prior unacked messages
                //                //        (batch ack, use carefully)
                fmt.Println("  -> ACK")

            } else if isRetryable(err) {
                // NACK with requeue: put message back in the queue
                // WARNING: can cause infinite loops if the message always fails
                msg.Nack(false, true) // (multiple=false, requeue=true)
                fmt.Printf("  -> NACK (requeue): %v\n", err)

            } else {
                // NACK without requeue: send to dead letter queue (if configured)
                msg.Nack(false, false) // (multiple=false, requeue=false)
                fmt.Printf("  -> NACK (dead letter): %v\n", err)

                // Alternative: Reject (same as Nack for a single message)
                // msg.Reject(false) // requeue=false -> dead letter
            }
        }
    }()

    <-sigCh
    fmt.Println("Shutting down consumer...")
}

func processMessage(body []byte) error {
    // Your business logic
    return nil
}

func isRetryable(err error) bool {
    // Determine if the error is transient and worth retrying
    return false
}
```

### 4.6 Priority Queues

```go
// Declare a priority queue
_, err := ch.QueueDeclare(
    "orders.priority",
    true,
    false,
    false,
    false,
    amqp.Table{
        "x-max-priority": int32(10), // Priority range: 0-10
    },
)

// Publish with priority
ch.PublishWithContext(ctx,
    "",
    "orders.priority",
    false,
    false,
    amqp.Publishing{
        Priority: 9, // High priority (0 = lowest, 10 = highest)
        Body:     []byte(`{"id":"vip-order","priority":"urgent"}`),
    },
)

// Normal priority message
ch.PublishWithContext(ctx,
    "",
    "orders.priority",
    false,
    false,
    amqp.Publishing{
        Priority: 1, // Low priority
        Body:     []byte(`{"id":"regular-order","priority":"normal"}`),
    },
)
```

> **Tip**: Priority queues add overhead. Use a maximum of 5-10 priority levels. For most use
> cases, 2-3 levels (low, normal, high) are sufficient.

---

## 5. Comparing Kafka vs NATS vs RabbitMQ

### Feature Comparison

| Feature | Kafka | NATS JetStream | RabbitMQ |
|---------|-------|----------------|----------|
| **Protocol** | Custom binary | Custom text/binary | AMQP 0-9-1 |
| **Messaging Model** | Log-based pub/sub | Pub/sub + queue groups | Exchange/queue routing |
| **Ordering** | Per-partition | Per-stream/per-subject | Per-queue |
| **Delivery Guarantee** | At-least-once, exactly-once | At-least-once, exactly-once (limited) | At-most-once, at-least-once |
| **Persistence** | Always (append-only log) | Configurable (file/memory) | Configurable (transient/persistent) |
| **Replay** | Yes (seek to any offset) | Yes (by sequence, time, or subject) | No (messages deleted after ack) |
| **Consumer Groups** | Native | Queue groups + durable consumers | Competing consumers on same queue |
| **Routing** | Topic + partition key | Subject hierarchy + wildcards | Exchanges (direct, topic, fanout, headers) |
| **Dead Letter** | Application-level | Yes (max deliver + advisory subjects) | Native (DLX) |
| **Transactions** | Yes (producer transactions) | Limited (dedup window) | Yes (AMQP transactions, publisher confirms) |
| **Clustering** | Zookeeper/KRaft | Built-in RAFT | Quorum queues / classic mirroring |
| **Schema Registry** | Confluent Schema Registry | No built-in (use external) | No built-in (use external) |

### Performance Characteristics

| Metric | Kafka | NATS JetStream | RabbitMQ |
|--------|-------|----------------|----------|
| **Throughput** | Very high (millions msg/s) | High (hundreds of thousands msg/s) | Moderate (tens of thousands msg/s) |
| **Latency (p99)** | Low (5-15ms) | Very low (sub-ms to 2ms) | Low (1-10ms) |
| **Message Size** | Up to 1MB (default) | Up to 1MB (default, configurable) | No hard limit (but >128MB not recommended) |
| **Storage Overhead** | Higher (replication + compaction) | Moderate | Lower (messages deleted after ack) |

### When to Use What

| Use Case | Best Choice | Why |
|----------|-------------|-----|
| **Event sourcing / event log** | Kafka | Append-only log, replay capability, long retention |
| **High-throughput data pipelines** | Kafka | Designed for massive throughput, batch processing |
| **Microservice request/reply** | NATS | Ultra-low latency, native request/reply pattern |
| **IoT / edge computing** | NATS | Lightweight, leaf nodes, adaptive deployment |
| **Complex routing** | RabbitMQ | Flexible exchange types, binding patterns |
| **Task queues / work distribution** | RabbitMQ | Built-in QoS, priority queues, DLQ support |
| **Cloud-native / Kubernetes** | NATS | Small footprint, built-in clustering, easy to operate |
| **Legacy AMQP integration** | RabbitMQ | AMQP 0-9-1 protocol support |
| **Stream processing** | Kafka | Kafka Streams, ksqlDB, consumer group semantics |

### Decision Flowchart

```
Do you need message replay / event sourcing?
  YES -> Kafka or NATS JetStream
  NO  -> Do you need complex routing (topic patterns, headers)?
           YES -> RabbitMQ
           NO  -> Do you need ultra-low latency?
                    YES -> Core NATS (at-most-once) or NATS JetStream
                    NO  -> Any of the three will work; choose based on
                           operational familiarity
```

---

## 6. Message Serialization

### JSON (Simple, Human-Readable)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "time"
)

// OrderEvent represents a domain event
type OrderEvent struct {
    EventID   string    `json:"event_id"`
    EventType string    `json:"event_type"`
    Timestamp time.Time `json:"timestamp"`
    Payload   Order     `json:"payload"`
}

type Order struct {
    ID       string   `json:"id"`
    UserID   string   `json:"user_id"`
    Items    []Item   `json:"items"`
    Total    float64  `json:"total"`
    Currency string   `json:"currency"`
}

type Item struct {
    ProductID string  `json:"product_id"`
    Name      string  `json:"name"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

func main() {
    event := OrderEvent{
        EventID:   "evt-123",
        EventType: "OrderCreated",
        Timestamp: time.Now(),
        Payload: Order{
            ID:     "order-456",
            UserID: "user-789",
            Items: []Item{
                {ProductID: "prod-1", Name: "Widget", Quantity: 2, Price: 29.99},
            },
            Total:    59.98,
            Currency: "USD",
        },
    }

    // Serialize
    data, err := json.Marshal(event)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("JSON size: %d bytes\n", len(data))
    // Send `data` to your message queue

    // Deserialize
    var received OrderEvent
    if err := json.Unmarshal(data, &received); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Received event: %s for order %s\n",
        received.EventType, received.Payload.ID)
}
```

### Protocol Buffers (Fast, Compact, Strongly Typed)

First, define the proto file:

```protobuf
// proto/events/order.proto
syntax = "proto3";

package events;
option go_package = "myapp/proto/events";

import "google/protobuf/timestamp.proto";

message OrderEvent {
    string event_id = 1;
    string event_type = 2;
    google.protobuf.Timestamp timestamp = 3;
    Order payload = 4;
}

message Order {
    string id = 1;
    string user_id = 2;
    repeated Item items = 3;
    double total = 4;
    string currency = 5;
}

message Item {
    string product_id = 1;
    string name = 2;
    int32 quantity = 3;
    double price = 4;
}
```

```go
package main

import (
    "fmt"
    "log"

    "google.golang.org/protobuf/proto"
    "google.golang.org/protobuf/types/known/timestamppb"

    eventspb "myapp/proto/events"
)

func main() {
    event := &eventspb.OrderEvent{
        EventId:   "evt-123",
        EventType: "OrderCreated",
        Timestamp: timestamppb.Now(),
        Payload: &eventspb.Order{
            Id:     "order-456",
            UserId: "user-789",
            Items: []*eventspb.Item{
                {ProductId: "prod-1", Name: "Widget", Quantity: 2, Price: 29.99},
            },
            Total:    59.98,
            Currency: "USD",
        },
    }

    // Serialize (much smaller than JSON)
    data, err := proto.Marshal(event)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Protobuf size: %d bytes\n", len(data))
    // Typically 40-60% smaller than JSON

    // Deserialize
    received := &eventspb.OrderEvent{}
    if err := proto.Unmarshal(data, received); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Received event: %s for order %s\n",
        received.EventType, received.Payload.Id)
}
```

### Avro (Schema Evolution, Kafka-Native)

```go
package main

import (
    "fmt"
    "log"

    "github.com/linkedin/goavro/v2"
)

func main() {
    // Define Avro schema
    schemaJSON := `{
        "type": "record",
        "name": "OrderEvent",
        "namespace": "com.myapp.events",
        "fields": [
            {"name": "event_id", "type": "string"},
            {"name": "event_type", "type": "string"},
            {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"},
            {"name": "order_id", "type": "string"},
            {"name": "user_id", "type": "string"},
            {"name": "total", "type": "double"},
            {"name": "currency", "type": "string", "default": "USD"},
            {"name": "status", "type": {
                "type": "enum",
                "name": "OrderStatus",
                "symbols": ["CREATED", "PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
            }}
        ]
    }`

    codec, err := goavro.NewCodec(schemaJSON)
    if err != nil {
        log.Fatal(err)
    }

    // Create a record
    record := map[string]interface{}{
        "event_id":   "evt-123",
        "event_type": "OrderCreated",
        "timestamp":  int64(1700000000000),
        "order_id":   "order-456",
        "user_id":    "user-789",
        "total":      59.98,
        "currency":   "USD",
        "status":     goavro.Union("OrderStatus", "CREATED"),
    }

    // Encode to binary (very compact)
    binary, err := codec.BinaryFromNative(nil, record)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Avro binary size: %d bytes\n", len(binary))

    // Decode from binary
    native, _, err := codec.NativeFromBinary(binary)
    if err != nil {
        log.Fatal(err)
    }

    decoded := native.(map[string]interface{})
    fmt.Printf("Decoded: event_type=%s order_id=%s\n",
        decoded["event_type"], decoded["order_id"])

    // Avro also supports JSON encoding (for debugging)
    jsonBytes, err := codec.TextualFromNative(nil, record)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Avro JSON: %s\n", string(jsonBytes))
}
```

### Serialization Format Comparison

| Format | Size | Speed | Schema | Human-Readable | Schema Evolution |
|--------|------|-------|--------|----------------|-----------------|
| **JSON** | Largest | Slowest | Optional (JSON Schema) | Yes | Manual |
| **Protobuf** | Smallest | Fastest | Required (.proto) | No | Excellent (field numbers) |
| **Avro** | Small | Fast | Required (JSON schema) | No | Excellent (schema registry) |
| **MessagePack** | Small | Fast | None | No | None |

> **Tip**: For Kafka, use Avro with Confluent Schema Registry for automatic schema evolution
> and compatibility checking. For NATS and RabbitMQ, Protobuf or JSON are common choices.

### Envelope Pattern

Wrap messages in a standard envelope for consistent handling across services:

```go
package messaging

import (
    "encoding/json"
    "time"

    "github.com/google/uuid"
)

// MessageEnvelope is a standard wrapper for all messages.
type MessageEnvelope struct {
    // Metadata
    MessageID     string            `json:"message_id"`
    CorrelationID string            `json:"correlation_id"`
    CausationID   string            `json:"causation_id"`
    EventType     string            `json:"event_type"`
    Source        string            `json:"source"`
    Timestamp     time.Time         `json:"timestamp"`
    Version       int               `json:"version"`
    Headers       map[string]string `json:"headers,omitempty"`

    // Payload as raw JSON (deferred deserialization)
    Payload json.RawMessage `json:"payload"`
}

// NewEnvelope creates a new message envelope.
func NewEnvelope(eventType string, source string, payload interface{}) (*MessageEnvelope, error) {
    payloadBytes, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }

    return &MessageEnvelope{
        MessageID:     uuid.New().String(),
        CorrelationID: uuid.New().String(),
        EventType:     eventType,
        Source:        source,
        Timestamp:     time.Now().UTC(),
        Version:       1,
        Payload:       payloadBytes,
    }, nil
}

// DecodePayload deserializes the payload into the given target.
func (e *MessageEnvelope) DecodePayload(target interface{}) error {
    return json.Unmarshal(e.Payload, target)
}

// WithCorrelation sets the correlation and causation IDs for tracing.
func (e *MessageEnvelope) WithCorrelation(correlationID, causationID string) *MessageEnvelope {
    e.CorrelationID = correlationID
    e.CausationID = causationID
    return e
}
```

---

## 7. Error Handling and Dead Letter Queues

### Retry Strategies

```go
package messaging

import (
    "context"
    "fmt"
    "log"
    "math"
    "math/rand"
    "time"
)

// RetryConfig defines retry behavior.
type RetryConfig struct {
    MaxRetries     int
    InitialBackoff time.Duration
    MaxBackoff     time.Duration
    BackoffFactor  float64
    Jitter         bool // Add randomness to prevent thundering herd
}

// DefaultRetryConfig returns a sensible default configuration.
func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxRetries:     5,
        InitialBackoff: 100 * time.Millisecond,
        MaxBackoff:     30 * time.Second,
        BackoffFactor:  2.0,
        Jitter:         true,
    }
}

// CalculateBackoff returns the delay for the nth retry.
func (rc RetryConfig) CalculateBackoff(attempt int) time.Duration {
    backoff := float64(rc.InitialBackoff) * math.Pow(rc.BackoffFactor, float64(attempt))
    if backoff > float64(rc.MaxBackoff) {
        backoff = float64(rc.MaxBackoff)
    }

    if rc.Jitter {
        // Add +/- 25% jitter
        jitter := backoff * 0.25
        backoff = backoff - jitter + (rand.Float64() * 2 * jitter)
    }

    return time.Duration(backoff)
}

// MessageHandler processes a message and returns an error if it fails.
type MessageHandler func(ctx context.Context, msg []byte) error

// RetryableHandler wraps a handler with retry logic.
func RetryableHandler(handler MessageHandler, config RetryConfig) MessageHandler {
    return func(ctx context.Context, msg []byte) error {
        var lastErr error

        for attempt := 0; attempt <= config.MaxRetries; attempt++ {
            if attempt > 0 {
                backoff := config.CalculateBackoff(attempt - 1)
                log.Printf("Retry attempt %d/%d after %v",
                    attempt, config.MaxRetries, backoff)

                select {
                case <-time.After(backoff):
                case <-ctx.Done():
                    return fmt.Errorf("context cancelled during retry: %w", ctx.Err())
                }
            }

            lastErr = handler(ctx, msg)
            if lastErr == nil {
                return nil
            }

            // Check if error is retryable
            if !isRetryableError(lastErr) {
                return fmt.Errorf("non-retryable error: %w", lastErr)
            }

            log.Printf("Attempt %d failed: %v", attempt+1, lastErr)
        }

        return fmt.Errorf("max retries (%d) exceeded: %w", config.MaxRetries, lastErr)
    }
}

// Error classification
type PermanentError struct {
    Err error
}

func (e *PermanentError) Error() string { return e.Err.Error() }
func (e *PermanentError) Unwrap() error { return e.Err }

type TransientError struct {
    Err error
}

func (e *TransientError) Error() string { return e.Err.Error() }
func (e *TransientError) Unwrap() error { return e.Err }

func isRetryableError(err error) bool {
    // Permanent errors should NOT be retried
    if _, ok := err.(*PermanentError); ok {
        return false
    }
    // Everything else (including TransientError) is retryable
    return true
}
```

### Generic Dead Letter Queue Handler

```go
package messaging

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
)

// DeadLetterMessage wraps the original message with failure metadata.
type DeadLetterMessage struct {
    OriginalMessage json.RawMessage `json:"original_message"`
    OriginalTopic   string          `json:"original_topic"`
    OriginalKey     string          `json:"original_key"`
    ErrorMessage    string          `json:"error_message"`
    ErrorType       string          `json:"error_type"`
    FailedAt        time.Time       `json:"failed_at"`
    RetryCount      int             `json:"retry_count"`
    ConsumerGroup   string          `json:"consumer_group"`
    ConsumerID      string          `json:"consumer_id"`
}

// DeadLetterPublisher sends failed messages to a dead letter topic/queue.
type DeadLetterPublisher interface {
    PublishDeadLetter(ctx context.Context, dlm DeadLetterMessage) error
}

// MessageProcessor is a high-level message processor with DLQ support.
type MessageProcessor struct {
    handler     MessageHandler
    retryConfig RetryConfig
    dlq         DeadLetterPublisher
    topic       string
    consumerID  string
}

func NewMessageProcessor(
    handler MessageHandler,
    retryConfig RetryConfig,
    dlq DeadLetterPublisher,
    topic string,
    consumerID string,
) *MessageProcessor {
    return &MessageProcessor{
        handler:     handler,
        retryConfig: retryConfig,
        dlq:         dlq,
        topic:       topic,
        consumerID:  consumerID,
    }
}

// Process handles a message with retries and dead-letter fallback.
func (mp *MessageProcessor) Process(ctx context.Context, key string, value []byte) error {
    retryHandler := RetryableHandler(mp.handler, mp.retryConfig)

    err := retryHandler(ctx, value)
    if err == nil {
        return nil // Success
    }

    // All retries exhausted or permanent error; send to DLQ
    log.Printf("Sending message to DLQ: key=%s error=%v", key, err)

    dlm := DeadLetterMessage{
        OriginalMessage: value,
        OriginalTopic:   mp.topic,
        OriginalKey:     key,
        ErrorMessage:    err.Error(),
        ErrorType:       fmt.Sprintf("%T", err),
        FailedAt:        time.Now().UTC(),
        RetryCount:      mp.retryConfig.MaxRetries,
        ConsumerGroup:   "order-processing-group",
        ConsumerID:      mp.consumerID,
    }

    if dlqErr := mp.dlq.PublishDeadLetter(ctx, dlm); dlqErr != nil {
        // Critical: could not even send to DLQ
        log.Printf("CRITICAL: failed to publish to DLQ: %v (original error: %v)",
            dlqErr, err)
        return fmt.Errorf("DLQ publish failed: %w (original: %v)", dlqErr, err)
    }

    // Message is in DLQ; consider this "handled" from the consumer's perspective
    return nil
}
```

---

## 8. Idempotent Consumers

In at-least-once delivery systems, the same message may be delivered multiple times. Consumers
must be **idempotent**: processing the same message twice must produce the same result as
processing it once.

### Strategy 1: Deduplication Table

```go
package messaging

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "time"
)

// IdempotencyStore tracks processed message IDs.
type IdempotencyStore struct {
    db *sql.DB
}

func NewIdempotencyStore(db *sql.DB) (*IdempotencyStore, error) {
    // Create the deduplication table
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS processed_messages (
            message_id    VARCHAR(255) PRIMARY KEY,
            processed_at  TIMESTAMP NOT NULL DEFAULT NOW(),
            consumer_id   VARCHAR(255) NOT NULL,
            result        TEXT
        )
    `)
    if err != nil {
        return nil, fmt.Errorf("create table: %w", err)
    }

    return &IdempotencyStore{db: db}, nil
}

// IsProcessed checks if a message has already been processed.
func (s *IdempotencyStore) IsProcessed(ctx context.Context, messageID string) (bool, error) {
    var exists bool
    err := s.db.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_messages WHERE message_id = $1)",
        messageID,
    ).Scan(&exists)
    return exists, err
}

// MarkProcessed records that a message has been processed.
func (s *IdempotencyStore) MarkProcessed(
    ctx context.Context, messageID, consumerID, result string,
) error {
    _, err := s.db.ExecContext(ctx,
        `INSERT INTO processed_messages (message_id, consumer_id, result)
         VALUES ($1, $2, $3)
         ON CONFLICT (message_id) DO NOTHING`,
        messageID, consumerID, result,
    )
    return err
}

// Cleanup removes old entries to prevent the table from growing forever.
func (s *IdempotencyStore) Cleanup(ctx context.Context, olderThan time.Duration) (int64, error) {
    result, err := s.db.ExecContext(ctx,
        "DELETE FROM processed_messages WHERE processed_at < $1",
        time.Now().Add(-olderThan),
    )
    if err != nil {
        return 0, err
    }
    return result.RowsAffected()
}

// IdempotentHandler wraps a handler with deduplication.
func IdempotentHandler(
    store *IdempotencyStore,
    consumerID string,
    handler func(ctx context.Context, messageID string, data []byte) (string, error),
) MessageHandler {
    return func(ctx context.Context, data []byte) error {
        // Extract message ID from envelope
        var envelope MessageEnvelope
        if err := json.Unmarshal(data, &envelope); err != nil {
            return &PermanentError{Err: fmt.Errorf("unmarshal envelope: %w", err)}
        }

        messageID := envelope.MessageID
        if messageID == "" {
            return &PermanentError{Err: errors.New("message has no ID")}
        }

        // Check if already processed
        processed, err := store.IsProcessed(ctx, messageID)
        if err != nil {
            return &TransientError{Err: fmt.Errorf("check idempotency: %w", err)}
        }
        if processed {
            // Already processed; skip (idempotent)
            log.Printf("Skipping duplicate message: %s", messageID)
            return nil
        }

        // Process the message
        result, err := handler(ctx, messageID, envelope.Payload)
        if err != nil {
            return err
        }

        // Mark as processed
        if err := store.MarkProcessed(ctx, messageID, consumerID, result); err != nil {
            // Non-critical: worst case, we process it again (still idempotent)
            log.Printf("Warning: failed to mark message %s as processed: %v",
                messageID, err)
        }

        return nil
    }
}
```

### Strategy 2: Transactional Outbox (Process + Mark Atomically)

```go
package messaging

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
)

// TransactionalProcessor processes messages within a database transaction,
// ensuring the business operation and dedup marker are atomic.
type TransactionalProcessor struct {
    db *sql.DB
}

func NewTransactionalProcessor(db *sql.DB) *TransactionalProcessor {
    return &TransactionalProcessor{db: db}
}

// ProcessOrder is an example of an idempotent, transactional message handler.
func (tp *TransactionalProcessor) ProcessOrder(
    ctx context.Context, messageID string, data []byte,
) error {
    tx, err := tp.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return &TransientError{Err: fmt.Errorf("begin tx: %w", err)}
    }
    defer tx.Rollback() // No-op if committed

    // Step 1: Check and insert dedup marker (within same transaction)
    var exists bool
    err = tx.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_messages WHERE message_id = $1 FOR UPDATE)",
        messageID,
    ).Scan(&exists)
    if err != nil {
        return &TransientError{Err: fmt.Errorf("check dedup: %w", err)}
    }
    if exists {
        log.Printf("Duplicate message %s, skipping", messageID)
        return nil // Already processed
    }

    // Step 2: Parse the order
    var order Order
    if err := json.Unmarshal(data, &order); err != nil {
        return &PermanentError{Err: fmt.Errorf("unmarshal order: %w", err)}
    }

    // Step 3: Execute business logic within the transaction
    _, err = tx.ExecContext(ctx,
        `INSERT INTO orders (id, user_id, total, currency, status)
         VALUES ($1, $2, $3, $4, 'confirmed')`,
        order.ID, order.UserID, order.Total, order.Currency,
    )
    if err != nil {
        return &TransientError{Err: fmt.Errorf("insert order: %w", err)}
    }

    // Step 4: Mark message as processed (same transaction)
    _, err = tx.ExecContext(ctx,
        `INSERT INTO processed_messages (message_id, consumer_id, result)
         VALUES ($1, $2, $3)`,
        messageID, "order-processor", "confirmed",
    )
    if err != nil {
        return &TransientError{Err: fmt.Errorf("mark processed: %w", err)}
    }

    // Step 5: Commit atomically
    if err := tx.Commit(); err != nil {
        return &TransientError{Err: fmt.Errorf("commit: %w", err)}
    }

    return nil
}
```

### Strategy 3: Natural Idempotency

Sometimes the operation itself is naturally idempotent:

```go
// GOOD: Idempotent - setting to an absolute value
// "Set order status to SHIPPED" is safe to execute multiple times
db.Exec("UPDATE orders SET status = 'SHIPPED' WHERE id = $1", orderID)

// BAD: Not idempotent - relative operation
// "Increment balance by 100" will double-charge on redelivery
db.Exec("UPDATE accounts SET balance = balance + 100 WHERE id = $1", accountID)

// FIX: Make relative operations idempotent with a transaction ID
db.Exec(`
    INSERT INTO balance_transactions (transaction_id, account_id, amount)
    VALUES ($1, $2, 100)
    ON CONFLICT (transaction_id) DO NOTHING
`, messageID, accountID)

db.Exec(`
    UPDATE accounts
    SET balance = (SELECT COALESCE(SUM(amount), 0) FROM balance_transactions WHERE account_id = $1)
    WHERE id = $1
`, accountID)
```

---

## 9. Graceful Shutdown of Consumers

Graceful shutdown is critical to avoid message loss or duplicate processing. The pattern:
1. Catch OS signals (SIGINT, SIGTERM).
2. Stop fetching new messages.
3. Finish processing in-flight messages.
4. Commit offsets / acknowledge messages.
5. Close connections.

### Complete Graceful Shutdown Example

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    "github.com/segmentio/kafka-go"
)

// Consumer is a Kafka consumer with graceful shutdown support.
type Consumer struct {
    reader     *kafka.Reader
    handler    func(ctx context.Context, msg kafka.Message) error
    workerPool int
    wg         sync.WaitGroup
    messages   chan kafka.Message
}

func NewConsumer(brokers []string, topic, groupID string, workerPool int,
    handler func(ctx context.Context, msg kafka.Message) error,
) *Consumer {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:        brokers,
        GroupID:        groupID,
        Topic:          topic,
        StartOffset:    kafka.FirstOffset,
        CommitInterval: 0, // Manual commit
        MaxBytes:       10e6,
    })

    return &Consumer{
        reader:     reader,
        handler:    handler,
        workerPool: workerPool,
        messages:   make(chan kafka.Message, workerPool*2),
    }
}

// Run starts the consumer and blocks until shutdown.
func (c *Consumer) Run(ctx context.Context) error {
    // Start worker pool
    for i := 0; i < c.workerPool; i++ {
        c.wg.Add(1)
        go c.worker(ctx, i)
    }

    log.Printf("Consumer started with %d workers", c.workerPool)

    // Fetch messages in the main goroutine
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                // Context cancelled: stop fetching, but let workers finish
                log.Println("Context cancelled, stopping message fetch")
                break
            }
            log.Printf("Fetch error: %v", err)
            continue
        }

        select {
        case c.messages <- msg:
        case <-ctx.Done():
            // Could not send to worker; redelivery will happen on restart
            log.Printf("Shutdown before processing: partition=%d offset=%d",
                msg.Partition, msg.Offset)
            break
        }
    }

    // Close the messages channel so workers know to stop
    close(c.messages)

    // Wait for all workers to finish processing their current message
    log.Println("Waiting for in-flight messages to complete...")
    done := make(chan struct{})
    go func() {
        c.wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All workers finished")
    case <-time.After(30 * time.Second):
        log.Println("WARNING: Timed out waiting for workers (30s)")
    }

    // Close the reader (leaves the consumer group)
    if err := c.reader.Close(); err != nil {
        return fmt.Errorf("close reader: %w", err)
    }

    return nil
}

func (c *Consumer) worker(ctx context.Context, id int) {
    defer c.wg.Done()

    for msg := range c.messages {
        log.Printf("Worker %d processing: partition=%d offset=%d",
            id, msg.Partition, msg.Offset)

        // Use a timeout context for processing
        processCtx, cancel := context.WithTimeout(ctx, 60*time.Second)

        if err := c.handler(processCtx, msg); err != nil {
            log.Printf("Worker %d: handler error for offset %d: %v",
                id, msg.Offset, err)
            cancel()
            // Do NOT commit; message will be redelivered
            continue
        }
        cancel()

        // Commit offset after successful processing
        if err := c.reader.CommitMessages(context.Background(), msg); err != nil {
            log.Printf("Worker %d: commit error for offset %d: %v",
                id, msg.Offset, err)
        }
    }

    log.Printf("Worker %d stopped", id)
}

func main() {
    // Setup signal handling
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        sig := <-sigCh
        log.Printf("Received signal: %v", sig)
        cancel() // Trigger graceful shutdown
    }()

    // Create and run consumer
    consumer := NewConsumer(
        []string{"localhost:9092"},
        "orders",
        "order-processing-group",
        5, // 5 worker goroutines
        func(ctx context.Context, msg kafka.Message) error {
            // Simulate processing
            time.Sleep(100 * time.Millisecond)
            fmt.Printf("Processed: key=%s value=%s\n",
                string(msg.Key), string(msg.Value))
            return nil
        },
    )

    if err := consumer.Run(ctx); err != nil {
        log.Fatalf("Consumer error: %v", err)
    }

    log.Println("Consumer shut down cleanly")
}
```

### Graceful Shutdown for NATS JetStream

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

func main() {
    nc, err := nats.Connect("nats://localhost:4222",
        nats.DrainTimeout(30*time.Second), // Allow 30s for drain
    )
    if err != nil {
        log.Fatal(err)
    }

    js, err := jetstream.New(nc)
    if err != nil {
        log.Fatal(err)
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    consumer, err := js.CreateOrUpdateConsumer(ctx, "ORDERS", jetstream.ConsumerConfig{
        Name:      "order-processor",
        Durable:   "order-processor",
        AckPolicy: jetstream.AckExplicitPolicy,
        AckWait:   30 * time.Second,
    })
    if err != nil {
        log.Fatal(err)
    }

    var wg sync.WaitGroup

    // Start consuming
    consumeCtx, err := consumer.Consume(func(msg jetstream.Msg) {
        wg.Add(1)
        defer wg.Done()

        fmt.Printf("Processing: %s\n", string(msg.Data()))

        // Extend the ack deadline while processing
        msg.InProgress()

        // Simulate work
        time.Sleep(2 * time.Second)

        msg.Ack()
    })
    if err != nil {
        log.Fatal(err)
    }

    // Wait for shutdown signal
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh

    log.Println("Shutting down...")

    // Step 1: Stop accepting new messages
    consumeCtx.Stop()

    // Step 2: Wait for in-flight messages to finish
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All in-flight messages processed")
    case <-time.After(30 * time.Second):
        log.Println("Timeout waiting for in-flight messages")
    }

    // Step 3: Drain the NATS connection (flushes pending data)
    nc.Drain()

    log.Println("Shutdown complete")
}
```

### Graceful Shutdown for RabbitMQ

```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }

    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }

    ch.Qos(10, 0, false)

    msgs, err := ch.Consume("order-processing", "consumer-1", false, false, false, false, nil)
    if err != nil {
        log.Fatal(err)
    }

    var wg sync.WaitGroup

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    // Process messages until signal
    go func() {
        for msg := range msgs {
            wg.Add(1)
            go func(d amqp.Delivery) {
                defer wg.Done()
                fmt.Printf("Processing: %s\n", string(d.Body))
                time.Sleep(1 * time.Second) // Simulate work
                d.Ack(false)
            }(msg)
        }
    }()

    <-sigCh
    log.Println("Shutting down...")

    // Step 1: Cancel the consumer (stop receiving new messages)
    // The channel name must match what was passed to Consume
    if err := ch.Cancel("consumer-1", false); err != nil {
        log.Printf("Cancel error: %v", err)
    }

    // Step 2: Wait for in-flight messages
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All in-flight messages acknowledged")
    case <-time.After(30 * time.Second):
        log.Println("Timeout waiting for in-flight messages")
    }

    // Step 3: Close channel and connection
    ch.Close()
    conn.Close()

    log.Println("Shutdown complete")
}
```

---

## 10. Testing Message-Driven Systems

### Unit Testing with Interfaces

Design your code around interfaces so message brokers can be swapped out in tests.

```go
package messaging

import (
    "context"
)

// Publisher is the interface for producing messages.
type Publisher interface {
    Publish(ctx context.Context, topic string, key string, value []byte) error
    Close() error
}

// Subscriber is the interface for consuming messages.
type Subscriber interface {
    Subscribe(ctx context.Context, topic string, handler MessageHandler) error
    Close() error
}

// Message represents a generic message.
type Message struct {
    Topic   string
    Key     string
    Value   []byte
    Headers map[string]string
}
```

### In-Memory Broker for Tests

```go
package messaging_test

import (
    "context"
    "sync"
)

// InMemoryBroker is a test double for message brokers.
type InMemoryBroker struct {
    mu          sync.RWMutex
    messages    map[string][]Message      // topic -> messages
    subscribers map[string][]MessageHandler // topic -> handlers
    published   []Message                  // all published messages (for assertions)
}

type Message struct {
    Topic   string
    Key     string
    Value   []byte
    Headers map[string]string
}

type MessageHandler func(ctx context.Context, msg []byte) error

func NewInMemoryBroker() *InMemoryBroker {
    return &InMemoryBroker{
        messages:    make(map[string][]Message),
        subscribers: make(map[string][]MessageHandler),
    }
}

func (b *InMemoryBroker) Publish(ctx context.Context, topic, key string, value []byte) error {
    b.mu.Lock()
    defer b.mu.Unlock()

    msg := Message{Topic: topic, Key: key, Value: value}
    b.messages[topic] = append(b.messages[topic], msg)
    b.published = append(b.published, msg)

    // Deliver to subscribers immediately
    for _, handler := range b.subscribers[topic] {
        if err := handler(ctx, value); err != nil {
            return err
        }
    }

    return nil
}

func (b *InMemoryBroker) Subscribe(_ context.Context, topic string, handler MessageHandler) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.subscribers[topic] = append(b.subscribers[topic], handler)
    return nil
}

func (b *InMemoryBroker) Close() error { return nil }

// Test helper methods

func (b *InMemoryBroker) PublishedMessages(topic string) []Message {
    b.mu.RLock()
    defer b.mu.RUnlock()
    return b.messages[topic]
}

func (b *InMemoryBroker) AllPublished() []Message {
    b.mu.RLock()
    defer b.mu.RUnlock()
    return b.published
}

func (b *InMemoryBroker) Reset() {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.messages = make(map[string][]Message)
    b.published = nil
}
```

### Writing Unit Tests

```go
package order_test

import (
    "context"
    "encoding/json"
    "testing"

    "myapp/messaging"
)

// OrderService is the service under test.
type OrderService struct {
    publisher messaging.Publisher
}

func NewOrderService(pub messaging.Publisher) *OrderService {
    return &OrderService{publisher: pub}
}

func (s *OrderService) CreateOrder(ctx context.Context, order Order) error {
    // Business logic...

    // Publish event
    eventData, _ := json.Marshal(OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  order.UserID,
        Total:   order.Total,
    })

    return s.publisher.Publish(ctx, "orders.created", order.ID, eventData)
}

type Order struct {
    ID     string
    UserID string
    Total  float64
}

type OrderCreatedEvent struct {
    OrderID string  `json:"order_id"`
    UserID  string  `json:"user_id"`
    Total   float64 `json:"total"`
}

func TestCreateOrder_PublishesEvent(t *testing.T) {
    broker := NewInMemoryBroker()
    svc := NewOrderService(broker)

    order := Order{ID: "order-1", UserID: "user-1", Total: 99.99}
    err := svc.CreateOrder(context.Background(), order)
    if err != nil {
        t.Fatalf("CreateOrder failed: %v", err)
    }

    // Assert that the event was published
    messages := broker.PublishedMessages("orders.created")
    if len(messages) != 1 {
        t.Fatalf("expected 1 message, got %d", len(messages))
    }

    // Verify the event content
    var event OrderCreatedEvent
    if err := json.Unmarshal(messages[0].Value, &event); err != nil {
        t.Fatalf("unmarshal event: %v", err)
    }

    if event.OrderID != "order-1" {
        t.Errorf("expected order ID 'order-1', got %q", event.OrderID)
    }
    if event.Total != 99.99 {
        t.Errorf("expected total 99.99, got %f", event.Total)
    }
}

func TestCreateOrder_HandlesPublishError(t *testing.T) {
    broker := &FailingPublisher{err: fmt.Errorf("broker unavailable")}
    svc := NewOrderService(broker)

    err := svc.CreateOrder(context.Background(), Order{ID: "order-1"})
    if err == nil {
        t.Fatal("expected error when publish fails")
    }
}
```

### Integration Testing with Testcontainers

```go
package integration_test

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/segmentio/kafka-go"
    "github.com/testcontainers/testcontainers-go"
    tcKafka "github.com/testcontainers/testcontainers-go/modules/kafka"
)

func TestKafkaIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }

    ctx := context.Background()

    // Start a Kafka container
    kafkaContainer, err := tcKafka.Run(ctx,
        "confluentinc/confluent-local:7.5.0",
        tcKafka.WithClusterID("test-cluster"),
    )
    if err != nil {
        t.Fatalf("failed to start Kafka container: %v", err)
    }
    defer kafkaContainer.Terminate(ctx)

    // Get the broker address
    brokers, err := kafkaContainer.Brokers(ctx)
    if err != nil {
        t.Fatalf("failed to get brokers: %v", err)
    }

    topic := "test-orders"

    // Create topic
    conn, err := kafka.Dial("tcp", brokers[0])
    if err != nil {
        t.Fatal(err)
    }
    err = conn.CreateTopics(kafka.TopicConfig{
        Topic:             topic,
        NumPartitions:     3,
        ReplicationFactor: 1,
    })
    conn.Close()
    if err != nil {
        t.Fatal(err)
    }

    // Produce messages
    writer := &kafka.Writer{
        Addr:  kafka.TCP(brokers...),
        Topic: topic,
    }
    defer writer.Close()

    for i := 0; i < 10; i++ {
        err := writer.WriteMessages(ctx, kafka.Message{
            Key:   []byte(fmt.Sprintf("key-%d", i)),
            Value: []byte(fmt.Sprintf(`{"id":%d}`, i)),
        })
        if err != nil {
            t.Fatalf("write error: %v", err)
        }
    }

    // Consume and verify
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:     brokers,
        Topic:       topic,
        GroupID:     "test-group",
        StartOffset: kafka.FirstOffset,
    })
    defer reader.Close()

    received := 0
    readCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    for received < 10 {
        msg, err := reader.FetchMessage(readCtx)
        if err != nil {
            t.Fatalf("fetch error after %d messages: %v", received, err)
        }
        reader.CommitMessages(ctx, msg)
        received++
    }

    if received != 10 {
        t.Errorf("expected 10 messages, got %d", received)
    }
}
```

### Integration Testing with NATS

```go
package integration_test

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
)

func TestNATSJetStreamIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()

    // Start NATS container with JetStream enabled
    req := testcontainers.ContainerRequest{
        Image:        "nats:latest",
        Cmd:          []string{"-js", "-sd", "/data"},
        ExposedPorts: []string{"4222/tcp"},
        WaitingFor:   wait.ForLog("Server is ready").WithStartupTimeout(30 * time.Second),
    }

    natsContainer, err := testcontainers.GenericContainer(ctx,
        testcontainers.GenericContainerRequest{
            ContainerRequest: req,
            Started:          true,
        },
    )
    if err != nil {
        t.Fatal(err)
    }
    defer natsContainer.Terminate(ctx)

    endpoint, err := natsContainer.Endpoint(ctx, "")
    if err != nil {
        t.Fatal(err)
    }

    // Connect to NATS
    nc, err := nats.Connect(fmt.Sprintf("nats://%s", endpoint))
    if err != nil {
        t.Fatal(err)
    }
    defer nc.Close()

    js, err := jetstream.New(nc)
    if err != nil {
        t.Fatal(err)
    }

    // Create stream
    _, err = js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
        Name:     "TEST",
        Subjects: []string{"test.>"},
    })
    if err != nil {
        t.Fatal(err)
    }

    // Publish
    for i := 0; i < 5; i++ {
        _, err := js.Publish(ctx, "test.orders",
            []byte(fmt.Sprintf(`{"id":%d}`, i)))
        if err != nil {
            t.Fatal(err)
        }
    }

    // Consume
    consumer, err := js.CreateOrUpdateConsumer(ctx, "TEST", jetstream.ConsumerConfig{
        Durable:   "test-consumer",
        AckPolicy: jetstream.AckExplicitPolicy,
    })
    if err != nil {
        t.Fatal(err)
    }

    received := 0
    batch, err := consumer.Fetch(5, jetstream.FetchMaxWait(5*time.Second))
    if err != nil {
        t.Fatal(err)
    }

    for msg := range batch.Messages() {
        msg.Ack()
        received++
    }

    if received != 5 {
        t.Errorf("expected 5 messages, got %d", received)
    }
}
```

### Testing Consumer Handlers in Isolation

```go
package handler_test

import (
    "context"
    "encoding/json"
    "errors"
    "testing"
)

type OrderHandler struct {
    orders map[string]Order // In-memory store for testing
}

func NewOrderHandler() *OrderHandler {
    return &OrderHandler{orders: make(map[string]Order)}
}

func (h *OrderHandler) HandleOrderCreated(ctx context.Context, data []byte) error {
    var event struct {
        OrderID string  `json:"order_id"`
        UserID  string  `json:"user_id"`
        Total   float64 `json:"total"`
    }
    if err := json.Unmarshal(data, &event); err != nil {
        return err
    }
    if event.OrderID == "" {
        return errors.New("order_id is required")
    }
    h.orders[event.OrderID] = Order{
        ID:     event.OrderID,
        UserID: event.UserID,
        Total:  event.Total,
        Status: "created",
    }
    return nil
}

type Order struct {
    ID     string
    UserID string
    Total  float64
    Status string
}

func TestHandleOrderCreated(t *testing.T) {
    handler := NewOrderHandler()

    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {
            name:    "valid order",
            input:   `{"order_id":"o1","user_id":"u1","total":99.99}`,
            wantErr: false,
        },
        {
            name:    "missing order ID",
            input:   `{"user_id":"u1","total":99.99}`,
            wantErr: true,
        },
        {
            name:    "invalid JSON",
            input:   `not json`,
            wantErr: true,
        },
        {
            name:    "empty payload",
            input:   `{}`,
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := handler.HandleOrderCreated(context.Background(), []byte(tt.input))
            if (err != nil) != tt.wantErr {
                t.Errorf("HandleOrderCreated() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}

func TestHandleOrderCreated_Idempotency(t *testing.T) {
    handler := NewOrderHandler()

    data := []byte(`{"order_id":"o1","user_id":"u1","total":99.99}`)

    // Process the same message twice
    err1 := handler.HandleOrderCreated(context.Background(), data)
    err2 := handler.HandleOrderCreated(context.Background(), data)

    if err1 != nil || err2 != nil {
        t.Fatalf("expected no errors, got err1=%v err2=%v", err1, err2)
    }

    // State should be the same regardless of how many times processed
    order, exists := handler.orders["o1"]
    if !exists {
        t.Fatal("order not found")
    }
    if order.Total != 99.99 {
        t.Errorf("expected total 99.99, got %f", order.Total)
    }
}
```

---

## 11. Production Monitoring and Observability

### Metrics with Prometheus

```go
package monitoring

import (
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

// Metrics for message queue monitoring.
var (
    // Producer metrics
    messagesPublished = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mq_messages_published_total",
            Help: "Total number of messages published",
        },
        []string{"topic", "status"}, // status: "success" or "error"
    )

    publishLatency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "mq_publish_duration_seconds",
            Help:    "Time taken to publish a message",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 12), // 1ms to 4s
        },
        []string{"topic"},
    )

    // Consumer metrics
    messagesConsumed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mq_messages_consumed_total",
            Help: "Total number of messages consumed",
        },
        []string{"topic", "consumer_group", "status"},
    )

    consumeLatency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "mq_consume_processing_duration_seconds",
            Help:    "Time taken to process a consumed message",
            Buckets: prometheus.ExponentialBuckets(0.01, 2, 10), // 10ms to 10s
        },
        []string{"topic", "consumer_group"},
    )

    consumerLag = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "mq_consumer_lag_messages",
            Help: "Number of messages behind the latest offset",
        },
        []string{"topic", "consumer_group", "partition"},
    )

    // Error metrics
    messagesDeadLettered = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mq_messages_dead_lettered_total",
            Help: "Total number of messages sent to dead letter queue",
        },
        []string{"topic", "consumer_group", "error_type"},
    )

    messageRetries = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mq_message_retries_total",
            Help: "Total number of message processing retries",
        },
        []string{"topic", "consumer_group"},
    )

    // Connection metrics
    connectionStatus = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "mq_connection_status",
            Help: "Current connection status (1=connected, 0=disconnected)",
        },
        []string{"broker", "client_id"},
    )
)

// InstrumentedPublisher wraps a publisher with metrics.
type InstrumentedPublisher struct {
    inner Publisher
    topic string
}

func NewInstrumentedPublisher(inner Publisher, topic string) *InstrumentedPublisher {
    return &InstrumentedPublisher{inner: inner, topic: topic}
}

func (p *InstrumentedPublisher) Publish(ctx context.Context, key string, value []byte) error {
    start := time.Now()

    err := p.inner.Publish(ctx, p.topic, key, value)

    duration := time.Since(start)
    publishLatency.WithLabelValues(p.topic).Observe(duration.Seconds())

    if err != nil {
        messagesPublished.WithLabelValues(p.topic, "error").Inc()
        return err
    }

    messagesPublished.WithLabelValues(p.topic, "success").Inc()
    return nil
}

// InstrumentedHandler wraps a message handler with metrics.
func InstrumentedHandler(
    topic, consumerGroup string,
    handler MessageHandler,
) MessageHandler {
    return func(ctx context.Context, data []byte) error {
        start := time.Now()

        err := handler(ctx, data)

        duration := time.Since(start)
        consumeLatency.WithLabelValues(topic, consumerGroup).Observe(duration.Seconds())

        status := "success"
        if err != nil {
            status = "error"
        }
        messagesConsumed.WithLabelValues(topic, consumerGroup, status).Inc()

        return err
    }
}
```

### Distributed Tracing with OpenTelemetry

```go
package tracing

import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("message-queue")

// HeaderCarrier implements propagation.TextMapCarrier for message headers.
type HeaderCarrier map[string]string

func (c HeaderCarrier) Get(key string) string          { return c[key] }
func (c HeaderCarrier) Set(key string, value string)    { c[key] = value }
func (c HeaderCarrier) Keys() []string {
    keys := make([]string, 0, len(c))
    for k := range c {
        keys = append(keys, k)
    }
    return keys
}

// InjectTraceContext injects the current trace context into message headers.
func InjectTraceContext(ctx context.Context, headers map[string]string) {
    carrier := HeaderCarrier(headers)
    otel.GetTextMapPropagator().Inject(ctx, carrier)
}

// ExtractTraceContext extracts trace context from message headers.
func ExtractTraceContext(ctx context.Context, headers map[string]string) context.Context {
    carrier := HeaderCarrier(headers)
    return otel.GetTextMapPropagator().Extract(ctx, carrier)
}

// TracedPublish creates a span for message publishing.
func TracedPublish(
    ctx context.Context,
    topic string,
    publish func(ctx context.Context, headers map[string]string) error,
) error {
    ctx, span := tracer.Start(ctx, topic+" publish",
        trace.WithSpanKind(trace.SpanKindProducer),
        trace.WithAttributes(
            attribute.String("messaging.system", "kafka"),
            attribute.String("messaging.destination", topic),
            attribute.String("messaging.operation", "publish"),
        ),
    )
    defer span.End()

    headers := make(map[string]string)
    InjectTraceContext(ctx, headers)

    if err := publish(ctx, headers); err != nil {
        span.RecordError(err)
        return err
    }

    return nil
}

// TracedConsume creates a span for message consumption.
func TracedConsume(
    ctx context.Context,
    topic string,
    headers map[string]string,
    handler func(ctx context.Context) error,
) error {
    // Extract parent trace from message headers
    ctx = ExtractTraceContext(ctx, headers)

    ctx, span := tracer.Start(ctx, topic+" process",
        trace.WithSpanKind(trace.SpanKindConsumer),
        trace.WithAttributes(
            attribute.String("messaging.system", "kafka"),
            attribute.String("messaging.destination", topic),
            attribute.String("messaging.operation", "process"),
        ),
    )
    defer span.End()

    if err := handler(ctx); err != nil {
        span.RecordError(err)
        return err
    }

    return nil
}
```

### Structured Logging

```go
package logging

import (
    "context"
    "log/slog"
    "time"
)

// LoggingHandler wraps a message handler with structured logging.
func LoggingHandler(
    logger *slog.Logger,
    topic string,
    consumerGroup string,
    handler func(ctx context.Context, data []byte) error,
) func(ctx context.Context, data []byte) error {
    return func(ctx context.Context, data []byte) error {
        start := time.Now()

        logger.InfoContext(ctx, "processing message",
            slog.String("topic", topic),
            slog.String("consumer_group", consumerGroup),
            slog.Int("payload_size", len(data)),
        )

        err := handler(ctx, data)

        duration := time.Since(start)

        if err != nil {
            logger.ErrorContext(ctx, "message processing failed",
                slog.String("topic", topic),
                slog.String("consumer_group", consumerGroup),
                slog.Duration("duration", duration),
                slog.String("error", err.Error()),
            )
        } else {
            logger.InfoContext(ctx, "message processed successfully",
                slog.String("topic", topic),
                slog.String("consumer_group", consumerGroup),
                slog.Duration("duration", duration),
            )
        }

        return err
    }
}
```

### Health Checks

```go
package health

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// HealthStatus represents the health of a component.
type HealthStatus string

const (
    StatusHealthy   HealthStatus = "healthy"
    StatusDegraded  HealthStatus = "degraded"
    StatusUnhealthy HealthStatus = "unhealthy"
)

// ComponentHealth tracks the health of a single component.
type ComponentHealth struct {
    Status    HealthStatus `json:"status"`
    Message   string       `json:"message,omitempty"`
    CheckedAt time.Time    `json:"checked_at"`
}

// HealthChecker manages health checks for message queue components.
type HealthChecker struct {
    mu         sync.RWMutex
    components map[string]*ComponentHealth
    checks     map[string]func(ctx context.Context) error
}

func NewHealthChecker() *HealthChecker {
    return &HealthChecker{
        components: make(map[string]*ComponentHealth),
        checks:     make(map[string]func(ctx context.Context) error),
    }
}

// RegisterCheck adds a health check function for a component.
func (hc *HealthChecker) RegisterCheck(name string, check func(ctx context.Context) error) {
    hc.mu.Lock()
    defer hc.mu.Unlock()
    hc.checks[name] = check
    hc.components[name] = &ComponentHealth{
        Status:    StatusHealthy,
        CheckedAt: time.Now(),
    }
}

// RunChecks executes all registered health checks.
func (hc *HealthChecker) RunChecks(ctx context.Context) map[string]*ComponentHealth {
    hc.mu.Lock()
    defer hc.mu.Unlock()

    for name, check := range hc.checks {
        checkCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        err := check(checkCtx)
        cancel()

        health := &ComponentHealth{
            CheckedAt: time.Now(),
        }

        if err != nil {
            health.Status = StatusUnhealthy
            health.Message = err.Error()
        } else {
            health.Status = StatusHealthy
        }

        hc.components[name] = health
    }

    // Return a copy
    result := make(map[string]*ComponentHealth, len(hc.components))
    for k, v := range hc.components {
        copy := *v
        result[k] = &copy
    }
    return result
}

// OverallStatus returns the worst status among all components.
func (hc *HealthChecker) OverallStatus() HealthStatus {
    hc.mu.RLock()
    defer hc.mu.RUnlock()

    worst := StatusHealthy
    for _, health := range hc.components {
        if health.Status == StatusUnhealthy {
            return StatusUnhealthy
        }
        if health.Status == StatusDegraded {
            worst = StatusDegraded
        }
    }
    return worst
}

// HTTPHandler returns an HTTP handler for the health endpoint.
func (hc *HealthChecker) HTTPHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        results := hc.RunChecks(ctx)

        overall := hc.OverallStatus()

        response := struct {
            Status     HealthStatus                  `json:"status"`
            Components map[string]*ComponentHealth   `json:"components"`
        }{
            Status:     overall,
            Components: results,
        }

        w.Header().Set("Content-Type", "application/json")
        if overall == StatusUnhealthy {
            w.WriteHeader(http.StatusServiceUnavailable)
        }
        json.NewEncoder(w).Encode(response)
    }
}

// Example health checks for each broker:

func KafkaHealthCheck(brokers []string) func(ctx context.Context) error {
    return func(ctx context.Context) error {
        // Try to connect to a broker and fetch metadata
        conn, err := kafka.DialContext(ctx, "tcp", brokers[0])
        if err != nil {
            return fmt.Errorf("kafka dial: %w", err)
        }
        defer conn.Close()

        _, err = conn.Brokers()
        if err != nil {
            return fmt.Errorf("kafka brokers: %w", err)
        }
        return nil
    }
}

func NATSHealthCheck(nc *nats.Conn) func(ctx context.Context) error {
    return func(ctx context.Context) error {
        if !nc.IsConnected() {
            return fmt.Errorf("NATS not connected, status: %s",
                nc.Status().String())
        }
        // Use RTT as a health indicator
        rtt, err := nc.RTT()
        if err != nil {
            return fmt.Errorf("NATS RTT: %w", err)
        }
        if rtt > 1*time.Second {
            return fmt.Errorf("NATS high latency: %v", rtt)
        }
        return nil
    }
}

func RabbitMQHealthCheck(conn *amqp.Connection) func(ctx context.Context) error {
    return func(ctx context.Context) error {
        if conn.IsClosed() {
            return fmt.Errorf("RabbitMQ connection is closed")
        }
        // Try to create and close a channel as a health check
        ch, err := conn.Channel()
        if err != nil {
            return fmt.Errorf("RabbitMQ channel: %w", err)
        }
        ch.Close()
        return nil
    }
}
```

### Key Metrics to Monitor

| Metric | What It Tells You | Alert When |
|--------|-------------------|------------|
| **Consumer lag** | How far behind consumers are | Lag growing over time |
| **Publish rate** | Messages produced per second | Sudden drops or spikes |
| **Consume rate** | Messages consumed per second | Rate drops significantly |
| **Processing latency (p99)** | How long message processing takes | Exceeds SLA threshold |
| **Error rate** | Failed message processing | Above baseline (e.g., >1%) |
| **Dead letter rate** | Messages going to DLQ | Any increase warrants investigation |
| **Rebalance frequency** | Consumer group stability | Too frequent (unstable consumers) |
| **Connection count** | Active broker connections | Drops to zero or exceeds pool size |
| **Queue depth** | Messages waiting in queue | Growing unboundedly |
| **Disk usage** | Broker storage utilization | Approaching capacity |

### Alerting Rules (Prometheus Example)

```yaml
# prometheus-alerts.yml
groups:
  - name: message-queue-alerts
    rules:
      # Consumer lag is growing
      - alert: HighConsumerLag
        expr: mq_consumer_lag_messages > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag is high"
          description: "Consumer group {{ $labels.consumer_group }} on topic {{ $labels.topic }} has {{ $value }} messages lag"

      # Messages going to dead letter queue
      - alert: DeadLetterMessages
        expr: rate(mq_messages_dead_lettered_total[5m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Messages are being dead-lettered"
          description: "{{ $labels.consumer_group }} is sending messages to DLQ at {{ $value }}/s"

      # High processing latency
      - alert: HighProcessingLatency
        expr: histogram_quantile(0.99, rate(mq_consume_processing_duration_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Message processing latency is high"
          description: "p99 processing latency for {{ $labels.consumer_group }} is {{ $value }}s"

      # Consumer group is not consuming
      - alert: ConsumerGroupStalled
        expr: rate(mq_messages_consumed_total[10m]) == 0 AND mq_consumer_lag_messages > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Consumer group has stopped consuming"
          description: "{{ $labels.consumer_group }} has not consumed any messages in the last 10 minutes"

      # High error rate
      - alert: HighMessageErrorRate
        expr: >
          rate(mq_messages_consumed_total{status="error"}[5m])
          / rate(mq_messages_consumed_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High message processing error rate"
          description: "Error rate for {{ $labels.consumer_group }} is {{ $value | humanizePercentage }}"
```

### Grafana Dashboard Queries

```
# Consumer lag per partition
mq_consumer_lag_messages{consumer_group="order-processing-group"}

# Messages per second (published)
rate(mq_messages_published_total{status="success"}[1m])

# Messages per second (consumed)
rate(mq_messages_consumed_total{status="success"}[1m])

# Processing latency p50, p95, p99
histogram_quantile(0.50, rate(mq_consume_processing_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(mq_consume_processing_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(mq_consume_processing_duration_seconds_bucket[5m]))

# Dead letter rate
rate(mq_messages_dead_lettered_total[5m])

# Error rate percentage
rate(mq_messages_consumed_total{status="error"}[5m])
/ rate(mq_messages_consumed_total[5m]) * 100
```

---

## Quick Reference

### Choosing a Message Queue

```
Need event replay or event sourcing?  ------>  Kafka
Need ultra-low latency pub/sub?       ------>  Core NATS
Need persistence + low latency?       ------>  NATS JetStream
Need complex routing patterns?        ------>  RabbitMQ
Need distributed KV store?            ------>  NATS JetStream
Need exactly-once within the broker?  ------>  Kafka (transactions)
Need lightweight / edge deployment?   ------>  NATS
```

### Essential Go Packages

| Package | Install |
|---------|---------|
| segmentio/kafka-go | `go get github.com/segmentio/kafka-go` |
| confluent-kafka-go | `go get github.com/confluentinc/confluent-kafka-go/v2` |
| nats.go | `go get github.com/nats-io/nats.go` |
| nats.go/jetstream | `go get github.com/nats-io/nats.go/jetstream` |
| amqp091-go | `go get github.com/rabbitmq/amqp091-go` |
| testcontainers-go | `go get github.com/testcontainers/testcontainers-go` |

### Production Checklist

- [ ] **Idempotent consumers**: Dedup table or natural idempotency
- [ ] **Dead letter queues**: Configured for all consumer queues
- [ ] **Graceful shutdown**: Signal handling, drain in-flight messages
- [ ] **Retry with backoff**: Exponential backoff with jitter
- [ ] **Monitoring**: Lag, throughput, latency, error rate metrics
- [ ] **Health checks**: Broker connectivity, consumer group status
- [ ] **Distributed tracing**: Trace context propagated through message headers
- [ ] **Structured logging**: Correlation IDs, message metadata in logs
- [ ] **Serialization schema**: Versioned schemas with backward compatibility
- [ ] **Alerting**: Lag growth, DLQ activity, stalled consumers, high error rate
- [ ] **Capacity planning**: Partition count, retention policy, disk sizing
- [ ] **Security**: TLS encryption, SASL authentication, ACLs
