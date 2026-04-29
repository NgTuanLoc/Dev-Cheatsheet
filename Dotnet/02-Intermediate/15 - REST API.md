---
tags: [dotnet, intermediate, aspnetcore]
aliases: [REST, Web API, OpenAPI]
level: Intermediate
---

# REST API

> **One-liner**: REST conventions + ASP.NET Core idioms — resource-noun URLs, proper HTTP verbs/status codes, problem details for errors, OpenAPI for discoverability, and versioning for evolution.

---

## Quick Reference

| Verb | Idempotent | Safe | Returns | Use for |
|------|-----------|------|---------|---------|
| `GET` | yes | yes | 200 + body / 404 | read |
| `HEAD` | yes | yes | headers only | check existence |
| `POST` | no | no | 201 + Location / 200 | create / non-idempotent action |
| `PUT` | yes | no | 200 / 204 | replace whole resource |
| `PATCH` | no (typically) | no | 200 / 204 | partial update |
| `DELETE` | yes | no | 204 | remove |

| Status | Meaning |
|--------|---------|
| 200 OK | request succeeded with body |
| 201 Created | resource created (with `Location` header) |
| 204 No Content | success, no body |
| 400 Bad Request | malformed / validation error |
| 401 Unauthorized | not authenticated |
| 403 Forbidden | authenticated but not allowed |
| 404 Not Found | resource missing |
| 409 Conflict | state conflict (e.g. version) |
| 422 Unprocessable Entity | semantic validation error |
| 429 Too Many Requests | rate-limited |
| 500 Internal Server Error | bug |
| 503 Service Unavailable | overloaded / dependency down |

---

## Core Concept

REST = Representational State Transfer. The URL is a **noun** (resource), the HTTP **verb** is the action. Stateless: each request carries everything needed.

URL conventions:
```
GET    /users           — list
GET    /users/42        — one
POST   /users           — create
PUT    /users/42        — replace
PATCH  /users/42        — modify
DELETE /users/42        — remove
GET    /users/42/orders — sub-resource
```

For errors, ASP.NET Core ships **RFC 7807 ProblemDetails**: a JSON document with `type`, `title`, `status`, `detail`, `instance`. Use it instead of ad-hoc error shapes.

**OpenAPI** (formerly Swagger) is a JSON/YAML schema describing every endpoint. ASP.NET Core auto-generates it; clients can be code-generated from it.

---

## Syntax & API

### CRUD with minimal APIs
```csharp
var users = app.MapGroup("/api/v1/users").WithTags("Users").WithOpenApi();

users.MapGet("",          ListUsers);
users.MapGet("{id:int}",  GetUser);
users.MapPost("",         CreateUser);
users.MapPut("{id:int}",  ReplaceUser);
users.MapPatch("{id:int}", PatchUser);
users.MapDelete("{id:int}", DeleteUser);

static async Task<Ok<List<User>>> ListUsers(IUserService s) =>
    TypedResults.Ok(await s.ListAsync());

static async Task<Results<Ok<User>, NotFound>> GetUser(int id, IUserService s)
{
    var u = await s.GetByIdAsync(id);
    return u is null ? TypedResults.NotFound() : TypedResults.Ok(u);
}

static async Task<Created<User>> CreateUser(CreateUserDto dto, IUserService s)
{
    var u = await s.CreateAsync(dto);
    return TypedResults.Created($"/api/v1/users/{u.Id}", u);
}

static async Task<NoContent> DeleteUser(int id, IUserService s)
{
    await s.DeleteAsync(id);
    return TypedResults.NoContent();
}
```

### Validation
```csharp
public record CreateUserDto(
    [Required, EmailAddress] string Email,
    [Required, MinLength(2)] string Name,
    [Range(13, 150)]         int    Age);

// Auto-validation requires:
// builder.Services.AddControllers().AddDataAnnotations();
// In minimal APIs use FluentValidation or hand-roll
```

### ProblemDetails for errors
```csharp
builder.Services.AddProblemDetails();        // .NET 8+
app.UseExceptionHandler();
app.UseStatusCodePages();

// In a handler
return TypedResults.Problem(
    title: "Insufficient funds",
    detail: $"Balance {balance} < requested {amount}",
    statusCode: StatusCodes.Status409Conflict);
```

### OpenAPI / Swagger
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new() { Title = "Shop API", Version = "v1" });
});

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();   // /swagger
}
```

### Versioning
```csharp
// Microsoft.AspNetCore.Mvc.Versioning package
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;
});

app.MapGet("/api/v{version:apiVersion}/users", ...);
```

### Pagination
```csharp
public record PagedResult<T>(IReadOnlyList<T> Items, int Page, int Size, int Total);

app.MapGet("/users", async (int page = 1, int size = 20, IUserService s) =>
{
    var (items, total) = await s.PageAsync(page, size);
    return TypedResults.Ok(new PagedResult<User>(items, page, size, total));
});
```

### Filtering / sorting
```csharp
// /users?name=alice&minAge=18&sort=-createdAt
app.MapGet("/users", async (
    string? name,
    int?    minAge,
    string? sort,
    IUserService s) =>
{
    var query = new UserQuery(name, minAge, sort);
    return TypedResults.Ok(await s.SearchAsync(query));
});
```

### Idempotency keys (POST safety)
```csharp
// Client sends: Idempotency-Key: 7c9c8e..
// Server stores key+result for 24h; replays return same response

app.MapPost("/payments",
    async ([FromHeader(Name="Idempotency-Key")] string key,
           PaymentDto dto, IPayments p) =>
    {
        if (await p.TryReplayAsync(key) is { } prev) return prev;
        var result = await p.ChargeAsync(dto);
        await p.RememberAsync(key, result);
        return TypedResults.Ok(result);
    });
```

---

## Common Patterns

```csharp
// Pattern: ETag for optimistic concurrency
app.MapPut("/users/{id:int}", async (
    int id, UpdateUserDto dto,
    [FromHeader(Name = "If-Match")] string ifMatch,
    IUserService s) =>
{
    var current = await s.GetByIdAsync(id);
    if (current is null)              return Results.NotFound();
    if (current.RowVersion != ifMatch) return Results.StatusCode(412);  // Precondition Failed

    var updated = await s.UpdateAsync(id, dto);
    return TypedResults.Ok(updated);
});
```

```csharp
// Pattern: Hateoas-style links (simple)
public record UserDto(int Id, string Name, IDictionary<string,string> Links);

return TypedResults.Ok(new UserDto(u.Id, u.Name, new Dictionary<string,string>
{
    ["self"]   = $"/users/{u.Id}",
    ["orders"] = $"/users/{u.Id}/orders"
}));
```

```csharp
// Pattern: rate limiting (.NET 7+)
builder.Services.AddRateLimiter(o => o.AddFixedWindowLimiter("api", lim =>
{
    lim.PermitLimit = 100;
    lim.Window = TimeSpan.FromMinutes(1);
}));
app.UseRateLimiter();
api.MapGet("/heavy", ...).RequireRateLimiting("api");
```

---

## Gotchas & Tips

- **`200` for delete is wrong** — use `204 No Content`. Reserve 200 for responses with bodies.
- **Don't return entities directly** — use DTOs to control shape; entities couple your API to your DB schema.
- **Always `Location` header on 201** — `Created("/users/{id}", body)`.
- **Use plurals** for collection resources: `/users`, not `/user`.
- **Filter / sort / paginate** at the database, not in memory — pass to EF Core.
- **Avoid verb-in-URL** (`/users/getAll`) — that's RPC, not REST. Reserve verbs for actions that aren't CRUD (`/orders/{id}/cancel` is acceptable).
- **`PUT` replaces, `PATCH` modifies** — semantics matter for caches and proxies.
- **Don't expose internal IDs** unnecessarily — use opaque identifiers (Guids) for public APIs.
- **Standardize error shape** with ProblemDetails — ad-hoc `{error:"..."}` shapes fragment client code.
- **Version from day 1** — even if it's just `/api/v1`. Adding versioning later breaks clients.

---

## See Also

- [[14 - ASP.NET Core Basics]]
- [[19 - Middleware]]
- [[05 - Security and Auth]]
- [[16 - Entity Framework Core]]
