# Go (Golang) Complete Study Guide

A comprehensive study guide covering Go from beginner to expert level, with in-depth explanations, Node.js comparisons, and hands-on exercises. Includes real-world patterns from production APIs, distributed systems concepts from *Designing Data-Intensive Applications*, and observability engineering practices.

**Total: 43 chapters | ~147,000 lines | ~4.3 MB of content**

---

## Beginner (Chapters 1-7)

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction & Setup](01-introduction-and-setup.md) | History, why Go, Go vs Node.js overview, installation, toolchain, IDE setup |
| 02 | [Variables, Types & Constants](02-variables-types-and-constants.md) | Declaration, basic types, strings, type conversion, constants, iota, scope |
| 03 | [Control Flow](03-control-flow.md) | if/else, switch, for loops, range, labels, defer |
| 04 | [Functions & Methods](04-functions-and-methods.md) | Multiple returns, variadic, closures, methods, receivers, init(), error pattern |
| 05 | [Arrays, Slices & Maps](05-arrays-slices-and-maps.md) | Arrays, slice internals, maps, strings/bytes/runes, common patterns |
| 06 | [Structs & Interfaces](06-structs-and-interfaces.md) | Structs, embedding, tags, implicit interfaces, composition vs inheritance |
| 07 | [Pointers & Memory](07-pointers-and-memory.md) | Pointers, pass-by-value, nil, new vs &T{}, stack vs heap, escape analysis |

## Intermediate (Chapters 8-13)

| # | Chapter | Topics |
|---|---------|--------|
| 08 | [Error Handling](08-error-handling.md) | Error values, wrapping, sentinel errors, custom errors, panic/recover |
| 09 | [Packages & Modules](09-packages-and-modules.md) | Packages, imports, Go modules, dependencies, standard library, workspaces |
| 10 | [Goroutines & Concurrency](10-goroutines-and-concurrency-basics.md) | Goroutines, scheduler, WaitGroup, Mutex, race conditions, Go vs Node.js event loop |
| 11 | [Channels & Select](11-channels-and-select.md) | Channels, buffered/unbuffered, select, fan-out/fan-in, pipelines, deadlocks |
| 12 | [JSON, HTTP & REST APIs](12-json-http-and-rest-apis.md) | JSON encoding, HTTP client/server, REST API, middleware, routing, validation |
| 13 | [File I/O & CLI Tools](13-file-io-and-cli-tools.md) | Reading/writing files, io.Reader/Writer, CLI tools, os/exec, cross-compilation |

## Advanced (Chapters 14-18)

| # | Chapter | Topics |
|---|---------|--------|
| 14 | [Testing & Benchmarking](14-testing-benchmarking-and-fuzzing.md) | Table-driven tests, subtests, coverage, benchmarks, fuzzing, mocking |
| 15 | [Generics](15-generics.md) | Type parameters, constraints, type sets, generic types, slices/maps packages |
| 16 | [Context Package](16-context-package.md) | Cancellation, timeouts, values, HTTP context, propagation, best practices |
| 17 | [Advanced Concurrency](17-advanced-concurrency.md) | sync.Pool, atomics, errgroup, singleflight, pipelines, rate limiting |
| 18 | [Reflection & Code Gen](18-reflection-and-code-generation.md) | reflect package, struct tags, go generate, code generation tools |

## Expert (Chapters 19-22)

| # | Chapter | Topics |
|---|---------|--------|
| 19 | [Runtime Internals](19-go-runtime-internals.md) | G-M-P scheduler, stack management, garbage collector, memory allocator, netpoller |
| 20 | [Performance & Profiling](20-performance-optimization-and-profiling.md) | pprof, benchmarking, memory/CPU optimization, GC tuning, tracing |
| 21 | [Building Microservices](21-building-microservices.md) | gRPC, databases, message queues, observability, Docker, Kubernetes |
| 22 | [System Programming & CGo](22-system-programming-cgo-and-advanced-patterns.md) | CGo, syscalls, unsafe, embed, design patterns, Go proverbs, final comparison |

## Real-World API Patterns (Chapters 23-32)

*Inspired by [krafty-core](https://github.com/user/krafty-core), a production Go API for IT certification exam prep.*

| # | Chapter | Topics |
|---|---------|--------|
| 23 | [Configuration & Environment](23-configuration-and-environment.md) | 12-factor app, .env/godotenv, config structs, validation, secrets, functional options |
| 24 | [Building a Custom HTTP Router](24-building-a-custom-http-router.md) | Path params, route matching, middleware chains, route groups, Chi/Gin/Echo comparison |
| 25 | [Middleware Patterns in Practice](25-middleware-patterns-in-practice.md) | Recovery, CORS, rate limiting, logging, security headers, body limits, request ID, auth |
| 26 | [Repository Pattern & Database](26-repository-pattern-and-database.md) | pgx, connection pools, goose migrations, CRUD, upserts, JSONB, transactions, mocking |
| 27 | [Authentication & Authorization](27-authentication-and-authorization.md) | JWT/Ed25519, OIDC, token exchange, cookies, refresh tokens, RBAC, auth middleware |
| 28 | [Structured Logging & Observability](28-structured-logging-and-observability.md) | slog, log levels, request IDs, response writer wrapping, query redaction, log correlation |
| 29 | [Request Validation & API Responses](29-request-validation-and-api-responses.md) | go-playground/validator, custom types, standard response envelope, error codes, pagination |
| 30 | [Graceful Shutdown & Production](30-graceful-shutdown-and-production-patterns.md) | Signal handling, health checks, server timeouts, connection draining, Docker, Kubernetes |
| 31 | [Concurrency in Web Applications](31-concurrency-in-web-applications.md) | Rate limiters, parallel queries, background tasks, worker pools, race condition debugging |
| 32 | [Building a Production API](32-building-a-production-api.md) | **Capstone** — complete Task Management API combining all patterns from Ch.23-31 |

## Designing Data-Intensive Applications (Chapters 33-39)

*Concepts from Martin Kleppmann's DDIA, implemented in Go.*

| # | Chapter | Topics |
|---|---------|--------|
| 33 | [Data Encoding & Serialization](33-data-encoding-and-serialization.md) | JSON deep dive, Protocol Buffers, gRPC, MessagePack, Avro, schema evolution, binary protocols |
| 34 | [Storage Engines from Scratch](34-storage-engines-from-scratch.md) | Append-only logs, hash indexes, SSTables, LSM trees, B-trees, WAL, Bloom filters |
| 35 | [Data Replication Patterns](35-data-replication-patterns.md) | Leader-follower, multi-leader, leaderless, quorums, vector clocks, CRDTs, Merkle trees |
| 36 | [Partitioning & Sharding](36-partitioning-and-sharding.md) | Key range, hash partitioning, consistent hashing, rebalancing, scatter-gather, routing |
| 37 | [Transactions & Consistency](37-transactions-and-consistency.md) | ACID, isolation levels, MVCC, 2PL, SSI, two-phase commit, saga pattern, Raft consensus |
| 38 | [Distributed Systems Challenges](38-distributed-systems-challenges.md) | Network faults, clock problems, Lamport/vector clocks, leader election, CAP, gossip protocols |
| 39 | [Batch & Stream Processing](39-batch-and-stream-processing.md) | MapReduce, event sourcing, CQRS, stream processing, windowing, CDC, backpressure |

## Observability Engineering (Chapters 40-43)

*Concepts from Charity Majors et al.'s Observability Engineering, implemented in Go.*

| # | Chapter | Topics |
|---|---------|--------|
| 40 | [Observability Fundamentals](40-observability-fundamentals.md) | Three pillars, structured events, OpenTelemetry, instrumentation, context propagation, sampling |
| 41 | [Distributed Tracing Deep Dive](41-distributed-tracing-deep-dive.md) | Spans, W3C Trace Context, exporters, batch processing, sampling strategies, trace visualization |
| 42 | [Metrics & SLOs](42-metrics-and-slos.md) | Prometheus, counters/gauges/histograms, RED/USE methods, SLIs, SLOs, error budgets, burn rates |
| 43 | [Observability Pipelines](43-observability-pipelines.md) | Log/metrics/trace collection, pipeline processing, fan-out routing, backpressure, data reduction |

---

## How to Use This Guide

1. **Beginners**: Start from Chapter 1 and work through sequentially
2. **Node.js developers**: Each chapter has dedicated Go vs Node.js comparison sections
3. **Experienced Go developers**: Jump to Advanced/Expert chapters for deep dives
4. **Building real APIs**: Chapters 23-32 walk through production patterns from a real codebase
5. **Distributed systems**: Chapters 33-39 implement DDIA concepts hands-on in Go
6. **Observability**: Chapters 40-43 cover modern instrumentation and monitoring practices
7. **Every chapter includes**: Code examples, visual diagrams, key takeaways, and practice exercises
