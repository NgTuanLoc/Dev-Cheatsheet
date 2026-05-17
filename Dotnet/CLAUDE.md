# CLAUDE.md — .NET Topic

Topic-specific guidance for the `Dotnet/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured .NET Core teacher-authored cheatsheet/notebook — organized from beginner to advanced, intended for students learning the .NET ecosystem.

---

## Folder Structure

```
Dotnet/
├── CLAUDE.md                    ← this file
├── 00-Index/
│   ├── Master Index.md          ← single entry point, links to all .NET notes
│   └── Learning Path.md         ← recommended reading order per level
├── 01-Beginner/                 (10 notes)
├── 02-Intermediate/             (20 notes)
└── 03-Advanced/                 (18 notes)
```

---

## Tag Taxonomy (.NET-specific)

Always include `dotnet` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `dotnet` | applied to every .NET note |
| `csharp` | pure language topics |
| `aspnetcore` | web framework topics |
| `efcore` | Entity Framework Core |
| `linq` | LINQ-specific |
| `async` | async/await, Task |
| `concurrency` | threading, locks, parallelism |
| `memory` | GC, IDisposable, leaks, Span<T> |
| `architecture` | design patterns, DDD, clean arch |
| `testing` | unit/integration/e2e |
| `devops` | Docker, CI/CD |
| `performance` | optimization, profiling, benchmarking |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `csharp` | all C# code |
| `bash` | CLI commands (`dotnet ef`, `dotnet build`) |
| `json` | `appsettings.json`, project files where readable as JSON |
| `xml` | `.csproj`, configuration |
| `yaml` | GitHub Actions, docker-compose |
| `mermaid` | diagrams |

Examples must be **minimal and compilable** — student should be able to paste into a fresh project.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Dotnet Overview | .NET vs .NET Core vs .NET Framework, CLR, SDK vs Runtime, CLI basics |
| 02 | CSharp Basics | Variables, data types, type inference, nullable, constants |
| 03 | Control Flow | if/else, switch expressions, for/foreach/while, pattern matching basics |
| 04 | Methods | Parameters, return types, overloading, optional/named params, ref/out |
| 05 | OOP Fundamentals | Classes, objects, constructors, properties, access modifiers, records |
| 06 | Collections | Array, List<T>, Dictionary<K,V>, HashSet, Queue, Stack |
| 07 | Strings | Interpolation, verbatim, common methods, StringBuilder |
| 08 | Exception Handling | try/catch/finally, custom exceptions, global handlers |
| 09 | File IO | File, Directory, Path, StreamReader/Writer, async IO |
| 10 | LINQ Basics | Where, Select, OrderBy, GroupBy, First/Single, ToList |

### Intermediate (02-Intermediate) — 20 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | OOP Advanced | Inheritance, abstract, interfaces, polymorphism, sealed, covariance |
| 02 | Generics | Generic classes/methods, constraints, covariance/contravariance |
| 03 | Delegates and Events | Action, Func, Predicate, EventHandler, multicast |
| 04 | Lambda and Functional | Closures, expression trees, functional patterns |
| 05 | LINQ Advanced | Join, Aggregate, let, query syntax vs method syntax, deferred execution |
| 06 | Async and Await | Task, async/await, ConfigureAwait, cancellation, ValueTask |
| 07 | Threading and Concurrency | Thread, ThreadPool, Task vs Thread, parallelism vs concurrency, race conditions |
| 08 | Synchronization Primitives | lock, Monitor, Mutex, Semaphore, ReaderWriterLockSlim, Interlocked, ConcurrentDictionary |
| 09 | Memory Management and GC | Stack vs heap, value vs reference, generations 0/1/2, LOH, GC modes, finalizers |
| 10 | IDisposable and Resource Mgmt | using statement, using declaration, IAsyncDisposable, dispose pattern, finalizer fallback |
| 11 | Reflection and Attributes | Type, MethodInfo, custom attributes, AttributeUsage, runtime metadata |
| 12 | Serialization | System.Text.Json, Newtonsoft.Json, XML, source-generated serializers, custom converters |
| 13 | Dependency Injection | IServiceCollection, lifetimes, constructor injection, keyed services |
| 14 | ASP.NET Core Basics | Program.cs, minimal APIs, routing, filters, model binding |
| 15 | REST API | Controllers, action results, status codes, versioning, OpenAPI |
| 16 | Entity Framework Core | DbContext, migrations, relationships, querying, tracking |
| 17 | Configuration | appsettings.json, IOptions<T>, environment overrides, secrets |
| 18 | Logging | ILogger, log levels, structured logging, Serilog |
| 19 | Middleware | Pipeline, custom middleware, short-circuiting |
| 20 | Testing | xUnit, Moq, FluentAssertions, TestServer, integration tests |

### Advanced (03-Advanced) — 19 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Design Patterns | Repository, CQRS, Mediator (MediatR), Factory, Decorator |
| 02 | Clean Architecture | Layers, dependency rule, use cases, ports & adapters |
| 03 | Domain-Driven Design | Aggregates, value objects, domain events, bounded contexts |
| 04 | Microservices | Service decomposition, API gateway, service discovery |
| 05 | Security and Auth | JWT, OAuth2/OIDC, ASP.NET Core Identity, HTTPS, CORS |
| 06 | Performance Optimization | Benchmarking (BenchmarkDotNet), ArrayPool, ObjectPool, hot paths |
| 07 | Memory Leaks and Profiling | Common leak causes, dotMemory, dotnet-dump, dotnet-counters, ETW |
| 08 | Span and Memory Types | Span<T>, ReadOnlySpan<T>, Memory<T>, stackalloc, zero-allocation parsing |
| 09 | Channels and Pipelines | System.Threading.Channels, System.IO.Pipelines |
| 10 | Parallel and Dataflow | Parallel.For, Parallel.ForEachAsync, PLINQ, TPL Dataflow blocks |
| 11 | Caching | IMemoryCache, IDistributedCache, Redis, cache-aside pattern |
| 12 | Background Services | IHostedService, BackgroundService, Hangfire, Quartz.NET |
| 13 | SignalR | Hubs, groups, real-time streaming, scaling |
| 14 | gRPC | Protobuf, service definitions, client/server, streaming |
| 15 | Source Generators | Roslyn analyzers, incremental generators, JSON source-gen, regex source-gen |
| 16 | Native Interop and AOT | P/Invoke, DllImport, LibraryImport, Native AOT publishing, trimming |
| 17 | Docker and Containers | Dockerfile, multi-stage builds, docker-compose, health checks |
| 18 | CI-CD and DevOps | GitHub Actions, build/test/publish pipeline, environment promotion |
| 19 | Production Monitoring and Diagnostics | Observability pillars (logs/metrics/traces), OpenTelemetry, Serilog, health checks, dotnet-counters/trace/dump/gcdump, pattern catalog (latency, leaks, CPU, threadpool, connection pool, deadlock, GC pressure, 5xx, cold start, log flood) |

**Total: 49 notes** (10 beginner + 20 intermediate + 19 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| Async and Await | sequenceDiagram (await flow + state machine) |
| Threading and Concurrency | flowchart (Thread vs Task vs ThreadPool) |
| Synchronization Primitives | flowchart (decision tree: which primitive to pick) |
| Memory Management and GC | stateDiagram-v2 (Gen 0 → Gen 1 → Gen 2 → LOH) |
| IDisposable and Resource Mgmt | flowchart (dispose pattern + finalizer fallback) |
| Memory Leaks and Profiling | flowchart (common leak sources) |
| Span and Memory Types | graph (stack vs heap vs pinned memory) |
| Channels and Pipelines | sequenceDiagram (producer/consumer) |
| Parallel and Dataflow | graph (Dataflow block topology) |
| Middleware | flowchart (request pipeline) |
| Dependency Injection | graph (DI container resolution) |
| Entity Framework Core | erDiagram (sample model) |
| Clean Architecture | graph (layer dependencies) |
| Microservices | graph (service topology + gateway) |
| Security and Auth | sequenceDiagram (OAuth2/JWT handshake) |
| Design Patterns | classDiagram (per pattern) |
| Production Monitoring and Diagnostics | graph (observability stack — SDKs → exporters → backends → dashboards) |
