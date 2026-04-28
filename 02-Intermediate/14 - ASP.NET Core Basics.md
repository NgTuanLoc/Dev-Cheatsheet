---
tags: [dotnet, intermediate, aspnetcore]
aliases: [ASP.NET Core, Minimal API, WebApplication]
level: Intermediate
---

# ASP.NET Core Basics

> **One-liner**: ASP.NET Core is a cross-platform web framework — `WebApplication.CreateBuilder` builds the host, registers DI services, configures middleware, and starts Kestrel; **minimal APIs** are the lightweight modern style for HTTP endpoints.

---

## Quick Reference

| Component | Role |
|-----------|------|
| `WebApplicationBuilder` | Configure services + config + logging |
| `WebApplication` | Run the app, define routes/middleware |
| `Kestrel` | Built-in cross-platform HTTP server |
| `Middleware` | Per-request pipeline (auth, logging, ...) |
| `Endpoints` | Routed HTTP handlers |
| `IHostedService` | Background tasks (see [[12 - Background Services]]) |

| Map verb | Sugar |
|----------|-------|
| `MapGet(path, handler)` | HTTP GET |
| `MapPost`, `MapPut`, `MapDelete`, `MapPatch` | other verbs |
| `MapGroup("/api/v1")` | shared prefix + filters |
| `MapControllers()` | enable controllers |
| `WithTags`, `WithName`, `WithOpenApi` | metadata |
| `RequireAuthorization()` | gate endpoint |

---

## Core Concept

ASP.NET Core uses a **host**: a long-running process with DI, configuration, logging, and lifetime management. The host owns one or more servers (typically Kestrel) listening for HTTP.

The **request pipeline** is built from middleware: each component sees the request, may short-circuit or call the next. Order matters — `UseAuthentication` must come before `UseAuthorization`. See [[19 - Middleware]].

**Endpoints** are the leaf of the pipeline. They're either:
- **Minimal API**: `app.MapGet("/x", handler)` — terse, fast, no controller class needed
- **Controllers**: classes with `[HttpGet]` etc. — better for big APIs and complex routing

---

## Syntax & API

### Minimal API skeleton
```csharp
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// Middleware (order matters)
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Endpoints
app.MapGet("/", () => "Hello");

app.MapGet("/users/{id:int}", async (int id, IUserService svc) =>
{
    var user = await svc.GetByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
})
.WithName("GetUser")
.Produces<User>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);

app.Run();
```

### Route patterns
```csharp
app.MapGet("/orders/{id:guid}", (Guid id) => $"Order {id}");
app.MapGet("/files/{*path}", (string path) => $"File {path}");      // catch-all
app.MapGet("/items", (int page = 1, int size = 20) => $"page {page}");

// Constraints
// {id:int}, {id:guid}, {id:long}, {name:alpha}, {age:range(0,150)}
```

### Binding sources
```csharp
app.MapPost("/users", async (
    [FromBody] CreateUserDto dto,
    [FromServices] IUserService svc,
    [FromHeader(Name = "X-Trace")] string? trace,
    HttpContext ctx) =>
{
    var user = await svc.CreateAsync(dto);
    return Results.Created($"/users/{user.Id}", user);
});
```

### Results helpers
```csharp
Results.Ok(value);                          // 200 + JSON
Results.Created(uri, value);                // 201
Results.NoContent();                        // 204
Results.BadRequest(value);                  // 400
Results.NotFound();                         // 404
Results.Conflict();                         // 409
Results.Problem("oops", statusCode: 500);   // RFC 7807
Results.File(stream, "application/pdf", "report.pdf");
Results.Stream(stream);
TypedResults.Ok(value);                     // strongly-typed (preferred)
```

### Route groups
```csharp
var api = app.MapGroup("/api/v1")
    .RequireAuthorization()
    .WithOpenApi();

api.MapGet("/users",        async (IUserService s) => await s.ListAsync());
api.MapGet("/users/{id:int}", async (int id, IUserService s) => await s.GetByIdAsync(id));
api.MapPost("/users",       async (CreateUserDto d, IUserService s) => await s.CreateAsync(d));
```

### Controller-based
```csharp
builder.Services.AddControllers();
// ...
app.MapControllers();
```

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _svc;
    public UsersController(IUserService svc) => _svc = svc;

    [HttpGet("{id:int}")]
    [ProducesResponseType<User>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<User>> GetById(int id)
    {
        var u = await _svc.GetByIdAsync(id);
        return u is null ? NotFound() : Ok(u);
    }
}
```

### Configuration access
```csharp
var connStr = builder.Configuration.GetConnectionString("Default");
var port    = builder.Configuration["Smtp:Port"];
```

### Environments
```csharp
if (app.Environment.IsDevelopment())   { /* dev only */ }
if (app.Environment.IsStaging())       { /* ... */ }
if (app.Environment.IsProduction())    { /* ... */ }
// ASPNETCORE_ENVIRONMENT controls this
```

---

## Common Patterns

```csharp
// Pattern: minimal API + extension method to organize endpoints
public static class UserEndpoints
{
    public static IEndpointRouteBuilder MapUserApi(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/users").WithTags("Users");

        g.MapGet("",            ListUsers);
        g.MapGet("{id:int}",    GetUser);
        g.MapPost("",           CreateUser);
        g.MapDelete("{id:int}", DeleteUser);
        return app;
    }

    static async Task<IResult> ListUsers(IUserService s) =>
        TypedResults.Ok(await s.ListAsync());
    // ...
}

// In Program.cs:
app.MapUserApi();
```

```csharp
// Pattern: global exception handling
app.UseExceptionHandler(eh => eh.Run(async ctx =>
{
    var ex = ctx.Features.Get<IExceptionHandlerPathFeature>()?.Error;
    await Results.Problem(detail: ex?.Message, statusCode: 500).ExecuteAsync(ctx);
}));
```

---

## Gotchas & Tips

- **Order matters in the pipeline** — `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints. Misordering is a common source of "401 even though I'm logged in".
- **Don't `await` on `app.Run()` followed by more code** — `Run` blocks until shutdown.
- **Use `TypedResults`** instead of `Results` in minimal APIs — gives strongly-typed return + auto-OpenAPI.
- **Bind your DTO with `[FromBody]` attribute** if framework can't infer (e.g. when type ambiguity matters).
- **Avoid blocking** in handlers — `Task` everywhere, no `.Result` / `.Wait()`.
- **`HttpContext` is per-request** — never store in a singleton or capture in a long-lived task.
- **`IHttpContextAccessor`** lets services read the current `HttpContext` — register with `AddHttpContextAccessor()`. Use sparingly.
- **Kestrel is the default server** but you can run behind IIS/Nginx — use `UseForwardedHeaders` so client IPs are correct.
- **Hot reload** — `dotnet watch run` rebuilds + restarts on save.

---

## See Also

- [[15 - REST API]]
- [[19 - Middleware]]
- [[13 - Dependency Injection]]
- [[17 - Configuration]]
- [[18 - Logging]]
