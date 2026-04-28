---
tags: [dotnet, advanced, aspnetcore]
aliases: [gRPC, Protobuf, gRPC-Web, Streaming RPC]
level: Advanced
---

# gRPC

> **One-liner**: gRPC is a contract-first RPC framework using **Protobuf** definitions over HTTP/2 — fast, strongly typed, supports streaming in both directions, and is the right choice for **internal service-to-service** calls.

---

## Quick Reference

| File | Purpose |
|------|---------|
| `*.proto` | Service + message definitions |
| `Grpc.AspNetCore` | Server SDK |
| `Grpc.Net.Client` | Client SDK |
| `Grpc.Tools` | Code generation at build time |
| `Google.Protobuf` | Runtime |

| RPC type | Direction | Use case |
|----------|-----------|----------|
| Unary | one req → one resp | classic RPC call |
| Server streaming | one req → many resp | live feed |
| Client streaming | many req → one resp | upload/aggregate |
| Bidirectional | many ↔ many | chat-like |

| Feature | API |
|---------|-----|
| Define service | `service Foo { rpc Bar (X) returns (Y); }` |
| Generated server | `class FooService : Foo.FooBase { override Bar(...) }` |
| Generated client | `var client = new Foo.FooClient(channel)` |
| Channel | `GrpcChannel.ForAddress("https://...")` |
| Authentication | metadata header `Bearer <token>` |
| Deadline | `client.Bar(req, deadline: DateTime.UtcNow.AddSeconds(5))` |
| Status | `RpcException` carries `StatusCode` |
| Reflection | `MapGrpcReflectionService()` (for tools like grpcurl) |
| gRPC-Web | browser-friendly: `MapGrpcService<T>().EnableGrpcWeb()` |

---

## Core Concept

You start with a `.proto` contract. The Protobuf compiler generates strongly-typed message classes and service stubs in C# (and any other supported language). The server implements the abstract base class; clients call the typed client. The wire format is binary Protobuf over HTTP/2 — much smaller and faster than JSON.

gRPC's killer features are **streaming** (you can `IAsyncEnumerable`-style push from server, client, or both at once) and **schema evolution** (Protobuf is forward and backward compatible if you follow rules: add new fields with new tags, never reuse a tag).

The price: it's **HTTP/2-only**, which means browsers can't call gRPC services directly. For browser support, use **gRPC-Web** (proxies HTTP/1.1 to HTTP/2) or expose a separate REST gateway. For service-to-service, gRPC is hands-down the better choice.

---

## Syntax & API

### Project setup
```bash
dotnet new grpc -n Shop.Catalog
dotnet add Shop.Catalog package Grpc.AspNetCore
```

### .proto definition
```protobuf
// Protos/catalog.proto
syntax = "proto3";

option csharp_namespace = "Shop.Catalog";

package catalog;

service Catalog {
  rpc GetProduct (GetProductRequest) returns (Product);
  rpc ListProducts (ListProductsRequest) returns (stream Product);     // server-streaming
  rpc Upload (stream ProductDraft) returns (UploadResult);              // client-streaming
  rpc Chat (stream ChatMsg) returns (stream ChatMsg);                   // bidirectional
}

message GetProductRequest { int32 id = 1; }
message Product {
  int32 id = 1;
  string name = 2;
  double price = 3;
  repeated string tags = 4;
}

message ListProductsRequest {
  int32 page_size = 1;
  string filter = 2;
}

message ProductDraft { string name = 1; double price = 2; }
message UploadResult { int32 count = 1; }

message ChatMsg { string user = 1; string text = 2; }
```

```xml
<!-- csproj -->
<ItemGroup>
  <Protobuf Include="Protos\catalog.proto" GrpcServices="Server" />
</ItemGroup>
```

### Server implementation
```csharp
public sealed class CatalogService(ICatalogRepo repo) : Catalog.CatalogBase
{
    public override async Task<Product> GetProduct(GetProductRequest req, ServerCallContext ctx)
    {
        var p = await repo.GetAsync(req.Id, ctx.CancellationToken)
                ?? throw new RpcException(new Status(StatusCode.NotFound, $"product {req.Id}"));
        return new Product { Id = p.Id, Name = p.Name, Price = (double)p.Price };
    }

    public override async Task ListProducts(ListProductsRequest req, IServerStreamWriter<Product> stream, ServerCallContext ctx)
    {
        await foreach (var p in repo.SearchAsync(req.Filter, ctx.CancellationToken))
            await stream.WriteAsync(new Product { Id = p.Id, Name = p.Name, Price = (double)p.Price });
    }

    public override async Task<UploadResult> Upload(IAsyncStreamReader<ProductDraft> stream, ServerCallContext ctx)
    {
        int n = 0;
        await foreach (var d in stream.ReadAllAsync(ctx.CancellationToken))
        {
            await repo.AddAsync(new Product(d.Name, (decimal)d.Price), ctx.CancellationToken);
            n++;
        }
        return new UploadResult { Count = n };
    }

    public override async Task Chat(
        IAsyncStreamReader<ChatMsg> input, IServerStreamWriter<ChatMsg> output, ServerCallContext ctx)
    {
        await foreach (var msg in input.ReadAllAsync(ctx.CancellationToken))
            await output.WriteAsync(new ChatMsg { User = "echo", Text = msg.Text });
    }
}
```

### Wiring
```csharp
var b = WebApplication.CreateBuilder(args);
b.Services.AddGrpc(o => o.EnableDetailedErrors = b.Environment.IsDevelopment());

var app = b.Build();
app.MapGrpcService<CatalogService>();
app.MapGrpcReflectionService();   // optional — exposes service list for tools
app.Run();
```

### Client (.proto with GrpcServices="Client")
```csharp
using var channel = GrpcChannel.ForAddress("https://catalog");
var client = new Catalog.CatalogClient(channel);

// unary
var product = await client.GetProductAsync(new GetProductRequest { Id = 42 });

// server streaming
using var call = client.ListProducts(new ListProductsRequest { Filter = "books" });
await foreach (var p in call.ResponseStream.ReadAllAsync())
    Console.WriteLine(p.Name);

// client streaming
using var up = client.Upload();
foreach (var d in drafts) await up.RequestStream.WriteAsync(d);
await up.RequestStream.CompleteAsync();
var result = await up;
```

### HttpClientFactory + DI
```csharp
b.Services.AddGrpcClient<Catalog.CatalogClient>(o => o.Address = new Uri("https://catalog"))
    .AddStandardResilienceHandler();
```

### Auth: send a Bearer token
```csharp
var meta = new Metadata { { "Authorization", $"Bearer {token}" } };
await client.GetProductAsync(req, meta);

// or via interceptor
public sealed class AuthInterceptor(ITokenService tokens) : Interceptor
{
    public override AsyncUnaryCall<TResp> AsyncUnaryCall<TReq, TResp>(
        TReq request, ClientInterceptorContext<TReq, TResp> context,
        AsyncUnaryCallContinuation<TReq, TResp> continuation)
    {
        var headers = context.Options.Headers ?? new Metadata();
        headers.Add("Authorization", $"Bearer {tokens.Get()}");
        return continuation(request, context.WithOptions(context.Options.WithHeaders(headers)));
    }
}
```

### Deadlines / cancellation
```csharp
var deadline = DateTime.UtcNow.AddSeconds(5);
try
{
    var p = await client.GetProductAsync(req, deadline: deadline);
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.DeadlineExceeded)
{
    // handle timeout
}
```

### gRPC-Web (browser)
```csharp
app.UseGrpcWeb(new GrpcWebOptions { DefaultEnabled = true });
app.MapGrpcService<CatalogService>().EnableGrpcWeb().RequireCors("any");
```

---

## Common Patterns

```csharp
// Pattern: well-known types
import "google/protobuf/timestamp.proto";

message Order {
  int32 id = 1;
  google.protobuf.Timestamp placed_at = 2;
}
```

```csharp
// Pattern: domain mapping (proto on the wire, domain inside)
public static Product ToProto(this Domain.Product p) =>
    new() { Id = p.Id, Name = p.Name, Price = (double)p.Price };

public static Domain.Product ToDomain(this Product p) =>
    new(p.Id, p.Name, (decimal)p.Price);
```

```csharp
// Pattern: RpcException with rich status
throw new RpcException(new Status(StatusCode.FailedPrecondition, "insufficient stock"),
    new Metadata { { "available", available.ToString() } });
```

```protobuf
// Pattern: schema evolution — add fields, never reuse tags
message Product {
  int32 id = 1;
  string name = 2;
  double price = 3;
  // v2 — added; old clients ignore unknown fields
  string sku = 4;
  // reserved — never reuse tag 5 even if you remove the field
  reserved 5;
}
```

---

## Gotchas & Tips

- **HTTP/2 is required end-to-end** — Kestrel supports it, but reverse proxies must too. Nginx needs `grpc_pass`; ALB/ACA/IIS have specific configs. HTTP/1.1 between hops breaks gRPC.
- **`GrpcChannel` is heavy — reuse it.** Create once per target, share across calls. `IHttpClientFactory.AddGrpcClient` does this.
- **TLS by default in .NET** — `https://` only unless you configure `GrpcChannelOptions.HttpHandler` for plaintext (mostly for tests).
- **Always set deadlines** — without one, a hanging server hangs callers indefinitely. Cascade them across hops.
- **`RpcException` is the only failure type** — server `throw new InvalidOperation()` becomes a generic `Internal` status with no detail.
- **Streams must be drained** — abandoning a server-streaming call without iterating leaks a connection until cancelled.
- **gRPC-Web is for browsers only** — adds latency. For service-to-service, use plain gRPC over HTTP/2.
- **JSON transcoding** (`.NET 7+`): `[HttpRule]` annotations expose REST endpoints from the same proto for browser-friendly clients without writing two services.
- **Versioning**: never delete or repurpose a tag. Use `reserved` to lock retired tags. Add new fields freely (zero/null defaults are fine).
- **Don't use `int64` if you target JS** — Protobuf's int64 round-trips as string in JSON because JS numbers lose precision.
- **gRPC is hard to debug with curl** — use **`grpcurl`** with reflection enabled, or Postman's gRPC support.

---

## See Also

- [[04 - Microservices]]
- [[15 - REST API]]
- [[05 - Security and Auth]]
- [[14 - ASP.NET Core Basics]]
