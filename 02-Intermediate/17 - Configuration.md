---
tags: [dotnet, intermediate, aspnetcore]
aliases: [appsettings, IConfiguration, IOptions]
level: Intermediate
---

# Configuration

> **One-liner**: .NET layers configuration sources (JSON, env vars, secrets, CLI args) into a single `IConfiguration`; bind sections to strongly-typed `IOptions<T>` for type safety and validation.

---

## Quick Reference

| Source | Order (later wins) |
|--------|--------------------|
| `appsettings.json` | 1 |
| `appsettings.{Environment}.json` | 2 |
| User Secrets (Development only) | 3 |
| Environment variables | 4 |
| Command-line args | 5 |

| API | Purpose |
|-----|---------|
| `IConfiguration["Smtp:Host"]` | Indexed access |
| `IConfiguration.GetValue<T>("...")` | Typed value |
| `IConfiguration.GetSection("Smtp")` | Sub-tree |
| `IConfiguration.GetConnectionString("Db")` | `ConnectionStrings:Db` shortcut |
| `services.Configure<TOptions>(section)` | Bind to options |
| `IOptions<T>` | Read once at startup |
| `IOptionsSnapshot<T>` | Per-scope (request) re-read |
| `IOptionsMonitor<T>` | Live with change notifications |

---

## Core Concept

Configuration is a **layered key-value store**. Keys use `:` (or `__` in env vars) for nesting. Sources are added in order; **later sources override earlier ones**. So `appsettings.json` provides defaults; environment variables override per environment; CLI args override everything.

You read it via `IConfiguration` (string-keyed) or, better, via the **options pattern** — define a POCO, bind a config section to it, inject `IOptions<TPoco>`. This gives type safety, IntelliSense, and central validation.

**Secrets** (DB passwords, API keys) should never live in committed JSON. In development use **User Secrets** (`dotnet user-secrets`); in production use environment variables or a secret manager (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault).

---

## Syntax & API

### appsettings.json
```json
{
  "Logging": {
    "LogLevel": { "Default": "Information" }
  },
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=shop;Username=app"
  },
  "Smtp": {
    "Host": "smtp.example.com",
    "Port": 587,
    "EnableSsl": true
  },
  "FeatureFlags": {
    "NewCheckout": false
  }
}
```

### appsettings.Development.json (overrides)
```json
{
  "Smtp": { "Host": "localhost" },
  "Logging": { "LogLevel": { "Default": "Debug" } }
}
```

### Environment variables (override)
```bash
# Underscores represent nesting in env vars
export Smtp__Host=smtp.prod.example.com
export ConnectionStrings__Default="Host=db.prod;..."

# Per-environment
export ASPNETCORE_ENVIRONMENT=Production
```

### Reading raw values
```csharp
var cfg = builder.Configuration;

string? host = cfg["Smtp:Host"];
int port = cfg.GetValue<int>("Smtp:Port", 25);                   // default 25
bool flag = cfg.GetValue<bool>("FeatureFlags:NewCheckout");
string? cs = cfg.GetConnectionString("Default");

IConfigurationSection smtp = cfg.GetSection("Smtp");
string? h = smtp["Host"];
```

### Options pattern
```csharp
public class SmtpOptions
{
    public const string SectionName = "Smtp";
    [Required] public string Host { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; }
    public bool EnableSsl { get; set; }
}

builder.Services
    .AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection(SmtpOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public class Mailer
{
    private readonly SmtpOptions _opts;
    public Mailer(IOptions<SmtpOptions> opts) => _opts = opts.Value;
}
```

### Reload on change
```csharp
public class Mailer
{
    private SmtpOptions _opts;

    public Mailer(IOptionsMonitor<SmtpOptions> monitor)
    {
        _opts = monitor.CurrentValue;
        monitor.OnChange(v => _opts = v);
    }
}
// JSON file watcher reloads automatically
```

### User Secrets (development)
```bash
cd MyApp
dotnet user-secrets init
dotnet user-secrets set "Smtp:Password" "supersecret"
dotnet user-secrets list
dotnet user-secrets remove "Smtp:Password"
```
Stored at `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` (Windows) or `~/.microsoft/usersecrets/<id>/secrets.json`. Never committed.

### Custom source — Azure Key Vault example
```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

### Validate at startup with eager binding
```csharp
builder.Services
    .AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .Validate(o => o.Port > 0, "Port must be positive")
    .ValidateOnStart();        // throws on app start if invalid
```

---

## Common Patterns

```csharp
// Pattern: per-environment overrides
// appsettings.json — defaults
// appsettings.Development.json — local dev
// appsettings.Staging.json     — staging
// appsettings.Production.json  — prod (often only in CI/CD)
```

```csharp
// Pattern: feature flag via config
public class CheckoutController(IOptionsMonitor<FeatureFlags> flags) : ControllerBase
{
    [HttpPost]
    public IActionResult Place() =>
        flags.CurrentValue.NewCheckout ? PlaceV2() : PlaceV1();
}
```

```csharp
// Pattern: sectioned binding for nested config
public record ApiOptions(string BaseUrl, ApiAuthOptions Auth);
public record ApiAuthOptions(string ClientId, string ClientSecret);

builder.Services.Configure<ApiOptions>(builder.Configuration.GetSection("Api"));
```

---

## Gotchas & Tips

- **Env var nesting uses `__` (double underscore)**, not `:` — `Smtp__Host`, not `Smtp:Host`.
- **`appsettings.json` is loaded by case-insensitive key** — `Smtp:Host` and `smtp:host` are the same.
- **Don't commit secrets** — `appsettings.Development.json` is committed; if you put a real password there, it's in git history forever.
- **`IOptions<T>` is singleton** — value snapshotted at first resolve. For request-scoped reload use `IOptionsSnapshot<T>`.
- **`IOptionsMonitor<T>` is the only way to react to live changes** — file changes trigger callbacks.
- **Don't bind to mutable static state** — bind to a class with a private setter or `init`.
- **JSON arrays use index keys** — `Hosts:0`, `Hosts:1`, etc. Comma-delimited isn't supported by default.
- **CLI args override** — `dotnet run -- Smtp:Host=test` works, useful in CI.
- **Use `[ConfigurationKeyName("name")]`** to map a property name different from the JSON key.

---

## See Also

- [[14 - ASP.NET Core Basics]]
- [[13 - Dependency Injection]]
- [[18 - Logging]]
- [[18 - CI-CD and DevOps]]
