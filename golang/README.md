# Go (Golang) Complete Study Guide

A comprehensive study guide covering Go from beginner to expert level, with in-depth explanations, Node.js comparisons, and hands-on exercises.

**Total: 22 chapters | ~46,000 lines | ~1.4 MB of content**

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

---

## How to Use This Guide

1. **Beginners**: Start from Chapter 1 and work through sequentially
2. **Node.js developers**: Each chapter has dedicated Go vs Node.js comparison sections
3. **Experienced Go developers**: Jump to Advanced/Expert chapters for deep dives
4. **Every chapter includes**: Code examples, visual diagrams, key takeaways, and practice exercises
