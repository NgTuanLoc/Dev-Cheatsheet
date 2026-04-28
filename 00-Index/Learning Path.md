---
tags: [dotnet, index, learning-path]
aliases: [Roadmap, Curriculum]
---

# Learning Path

> **Recommended order** to study the vault. Stages assume zero .NET background and build to production-grade depth.

---

## Stage 1 — Language Fluency (1–2 weeks)

**Goal**: Read and write idiomatic C# without consulting docs for basic syntax.

1. [[01 - Dotnet Overview]] — orient the ecosystem
2. [[02 - CSharp Basics]]
3. [[03 - Control Flow]]
4. [[04 - Methods]]
5. [[05 - OOP Fundamentals]]
6. [[06 - Collections]]
7. [[07 - Strings]]
8. [[08 - Exception Handling]]
9. [[09 - File IO]]
10. [[10 - LINQ Basics]]

**Checkpoint**: Build a console app that reads a CSV, filters/transforms with LINQ, writes results.

---

## Stage 2 — Modern C# Idioms (1 week)

**Goal**: Write code that reviewers won't reject in a 2025 codebase.

1. [[01 - OOP Advanced]]
2. [[02 - Generics]]
3. [[03 - Delegates and Events]]
4. [[04 - Lambda and Functional]]
5. [[05 - LINQ Advanced]]

**Checkpoint**: Refactor your console app to use generic repositories and event-driven flow.

---

## Stage 3 — Async, Threading, Memory (1–2 weeks)

**Goal**: Avoid deadlocks, race conditions, and leaks. The single biggest source of production bugs.

1. [[06 - Async and Await]] ← **read first**
2. [[07 - Threading and Concurrency]]
3. [[08 - Synchronization Primitives]]
4. [[09 - Memory Management and GC]]
5. [[10 - IDisposable and Resource Mgmt]]

**Checkpoint**: Build a parallel file processor with cancellation and proper disposal.

---

## Stage 4 — Runtime Services (3–5 days)

1. [[11 - Reflection and Attributes]]
2. [[12 - Serialization]]

---

## Stage 5 — Web & Data (2 weeks)

**Goal**: Ship a real REST API with persistence.

1. [[13 - Dependency Injection]]
2. [[14 - ASP.NET Core Basics]]
3. [[19 - Middleware]]
4. [[15 - REST API]]
5. [[17 - Configuration]]
6. [[18 - Logging]]
7. [[16 - Entity Framework Core]]
8. [[20 - Testing]]

**Checkpoint**: A REST API with EF Core, structured logging, integration tests.

---

## Stage 6 — Architecture (1–2 weeks)

1. [[01 - Design Patterns]]
2. [[02 - Clean Architecture]]
3. [[03 - Domain-Driven Design]]
4. [[05 - Security and Auth]]
5. [[04 - Microservices]]

---

## Stage 7 — Performance & Memory (1 week)

1. [[06 - Performance Optimization]]
2. [[07 - Memory Leaks and Profiling]]
3. [[08 - Span and Memory Types]]
4. [[09 - Channels and Pipelines]]
5. [[10 - Parallel and Dataflow]]

---

## Stage 8 — Infrastructure (1 week)

1. [[11 - Caching]]
2. [[12 - Background Services]]
3. [[13 - SignalR]]
4. [[14 - gRPC]]

---

## Stage 9 — Compiler & Deployment (3–5 days)

1. [[15 - Source Generators]]
2. [[16 - Native Interop and AOT]]
3. [[17 - Docker and Containers]]
4. [[18 - CI-CD and DevOps]]

---

## Tips for using this path

- **Don't skip Stage 3** — async/threading/memory mistakes cause 80% of production incidents.
- Build the checkpoint project at each stage. Reading without coding doesn't stick.
- Revisit earlier notes as you advance — concepts deepen with experience.
- Use [[Master Index]] to jump around; this path is recommended, not mandatory.
