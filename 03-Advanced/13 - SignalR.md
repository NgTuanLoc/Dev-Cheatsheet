---
tags: [dotnet, advanced, aspnetcore]
aliases: [SignalR, Hubs, WebSockets, Real-time]
level: Advanced
---

# SignalR

> **One-liner**: SignalR is the ASP.NET Core abstraction for **real-time bidirectional** communication — it negotiates the best transport (**WebSockets** → SSE → Long Polling) and lets servers call methods on connected clients via **Hubs**.

---

## Quick Reference

| Concept | API |
|---------|-----|
| Hub | `class ChatHub : Hub` |
| Strongly-typed hub | `Hub<TClient>` (compile-time client method names) |
| Map endpoint | `app.MapHub<ChatHub>("/chat")` |
| Send to all | `Clients.All.SendAsync("Method", arg)` |
| Send to caller | `Clients.Caller.SendAsync(...)` |
| Send to others | `Clients.Others.SendAsync(...)` |
| Send to group | `Clients.Group("room1").SendAsync(...)` |
| Send to user | `Clients.User(userId).SendAsync(...)` |
| Group management | `Groups.AddToGroupAsync(connId, "room1")` |
| Connection lifecycle | `OnConnectedAsync`, `OnDisconnectedAsync` |
| Streaming → client | `IAsyncEnumerable<T>` return |
| Streaming ← client | `IAsyncEnumerable<T>` parameter |
| Auth | `[Authorize]` on hub or method |
| Scale-out | Redis backplane, Azure SignalR Service |

---

## Core Concept

A **Hub** is the server-side entry point: clients call methods on the hub, and the hub calls methods on clients. Under the hood, SignalR uses WebSockets when both sides support them, falling back to Server-Sent Events or Long Polling. JavaScript, .NET, Java, Swift, and Python clients are all supported.

The mental model is **method calls in both directions**. The server writes `Clients.Group("room").SendAsync("MessageReceived", text)` — every connected client in that group receives a `MessageReceived` invocation. The client similarly invokes hub methods (`connection.invoke("SendMessage", text)`).

A **connection ID** is per WebSocket connection (changes on reconnect). A **user ID** is stable per authenticated user (across multiple devices/tabs). Use **groups** for arbitrary topic-based routing (e.g., chat rooms, document IDs).

For multiple ASP.NET instances, you need a **backplane** so that messages from one instance reach clients connected to another. The two production options: **Redis** backplane or **Azure SignalR Service** (managed). Without one, scale-out simply doesn't work.

---

## Syntax & API

### Hub
```csharp
public sealed class ChatHub : Hub
{
    public Task SendMessage(string room, string text) =>
        Clients.Group(room).SendAsync("MessageReceived", Context.UserIdentifier, text);

    public async Task JoinRoom(string room)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);
        await Clients.Group(room).SendAsync("UserJoined", Context.UserIdentifier);
    }

    public override async Task OnConnectedAsync()
    {
        await Clients.Caller.SendAsync("Welcome");
        await base.OnConnectedAsync();
    }
}
```

### Configuration
```csharp
var b = WebApplication.CreateBuilder(args);
b.Services.AddSignalR(o =>
{
    o.EnableDetailedErrors = b.Environment.IsDevelopment();
    o.MaximumReceiveMessageSize = 64 * 1024;
    o.ClientTimeoutInterval = TimeSpan.FromSeconds(60);
    o.KeepAliveInterval = TimeSpan.FromSeconds(15);
});

var app = b.Build();
app.MapHub<ChatHub>("/chat");
app.Run();
```

### Strongly-typed hub
```csharp
public interface IChatClient
{
    Task MessageReceived(string user, string text);
    Task UserJoined(string user);
}

public sealed class ChatHub : Hub<IChatClient>
{
    public Task SendMessage(string room, string text) =>
        Clients.Group(room).MessageReceived(Context.UserIdentifier!, text);
}
```

### Authentication
```csharp
[Authorize]
public sealed class ChatHub : Hub
{
    public Task Whisper(string targetUserId, string text) =>
        Clients.User(targetUserId).SendAsync("Whisper", Context.UserIdentifier, text);
}
```

### .NET client
```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://api.example.com/chat", o => o.AccessTokenProvider = () => Task.FromResult(token))
    .WithAutomaticReconnect()
    .Build();

connection.On<string, string>("MessageReceived", (user, text) =>
    Console.WriteLine($"{user}: {text}"));

await connection.StartAsync();
await connection.InvokeAsync("JoinRoom", "general");
await connection.InvokeAsync("SendMessage", "general", "hello!");
```

### JavaScript client
```javascript
const conn = new signalR.HubConnectionBuilder()
    .withUrl("/chat", { accessTokenFactory: () => token })
    .withAutomaticReconnect()
    .build();

conn.on("MessageReceived", (user, text) => render(user, text));
await conn.start();
await conn.invoke("SendMessage", "general", "hi");
```

### Streaming from server to client
```csharp
public sealed class StockHub : Hub
{
    public async IAsyncEnumerable<decimal> Prices(string symbol,
        [EnumeratorCancellation] CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            yield return await _quotes.GetPriceAsync(symbol, ct);
            await Task.Delay(1000, ct);
        }
    }
}

// Client
await foreach (var price in connection.StreamAsync<decimal>("Prices", "MSFT", ct))
    Console.WriteLine(price);
```

### Streaming from client to server
```csharp
public async Task UploadStream(IAsyncEnumerable<string> stream)
{
    await foreach (var chunk in stream)
        await _store.AppendAsync(chunk);
}
```

### Sending from outside the hub (controllers, services)
```csharp
public sealed class NotificationService(IHubContext<ChatHub, IChatClient> hubContext)
{
    public Task BroadcastAsync(string text) =>
        hubContext.Clients.All.MessageReceived("system", text);
}
```

### Redis backplane
```csharp
b.Services.AddSignalR()
    .AddStackExchangeRedis(b.Configuration.GetConnectionString("Redis")!,
        o => o.Configuration.ChannelPrefix = RedisChannel.Literal("signalr:"));
```

### Azure SignalR Service
```csharp
b.Services.AddSignalR().AddAzureSignalR(b.Configuration.GetConnectionString("AzureSignalR"));
```

### Custom user identifier (e.g., from JWT sub claim)
```csharp
public sealed class UserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext c) =>
        c.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
}
b.Services.AddSingleton<IUserIdProvider, UserIdProvider>();
```

---

## Common Patterns

```csharp
// Pattern: presence — track connected user IDs
public sealed class PresenceTracker
{
    private readonly ConcurrentDictionary<string, HashSet<string>> _online = new();

    public void Add(string userId, string connectionId) =>
        _online.AddOrUpdate(userId,
            _ => new HashSet<string> { connectionId },
            (_, s) => { lock (s) s.Add(connectionId); return s; });

    public bool Remove(string userId, string connectionId)
    {
        if (!_online.TryGetValue(userId, out var s)) return false;
        lock (s) s.Remove(connectionId);
        if (s.Count == 0) _online.TryRemove(userId, out _);
        return true;
    }

    public bool IsOnline(string userId) => _online.ContainsKey(userId);
}
```

```csharp
// Pattern: rejoin groups after reconnect
public override async Task OnConnectedAsync()
{
    var rooms = await _store.GetRoomsForUserAsync(Context.UserIdentifier!);
    foreach (var r in rooms) await Groups.AddToGroupAsync(Context.ConnectionId, r);
    await base.OnConnectedAsync();
}
```

```csharp
// Pattern: rate-limit per connection
public async Task SendMessage(string text)
{
    if (!await _limiter.AllowAsync(Context.ConnectionId))
        throw new HubException("Slow down.");
    // ...
}
```

---

## Gotchas & Tips

- **Method names are strings unless you use a strongly-typed Hub**. A typo on either side fails silently.
- **Hubs are transient** — don't store state on the hub instance; use a singleton service for shared state.
- **`OnDisconnectedAsync` runs on graceful disconnect** — abrupt drops time out via `ClientTimeoutInterval`. Plan for stale state.
- **`Context.UserIdentifier` is nullable** — until authenticated. Default uses `NameIdentifier` claim; override with `IUserIdProvider`.
- **Groups are not persisted** — on rejoin, you re-add. The server has no memory across reconnect.
- **Use `[Authorize]`** at the Hub class level. Don't rely on the SignalR endpoint URL being "internal".
- **Backplane required for scale-out** — without Redis or Azure SignalR, instance A's `Clients.All.SendAsync` doesn't reach instance B's clients.
- **Sticky sessions** are still required for Long Polling fallback even *with* a backplane. WebSockets don't need them.
- **Big payloads break things** — `MaximumReceiveMessageSize` defaults to 32 KB. Don't push 5 MB messages over SignalR; use a separate file API.
- **Hub method exceptions become `HubException` on client** — the `Message` propagates only in detailed-errors mode. Throw `HubException` for client-friendly errors.
- **Streaming uses cancellation** — pass and respect the `CancellationToken`, otherwise streams leak when clients disconnect.

---

## See Also

- [[14 - ASP.NET Core Basics]]
- [[05 - Security and Auth]]
- [[09 - Channels and Pipelines]]
- [[14 - gRPC]]
