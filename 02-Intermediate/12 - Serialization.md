---
tags: [dotnet, intermediate, csharp]
aliases: [JSON, XML, System.Text.Json, Newtonsoft]
level: Intermediate
---

# Serialization

> **One-liner**: `System.Text.Json` is the modern, fast, allocation-aware JSON library bundled with .NET; **source-generated** serializers (`[JsonSerializable]`) make it AOT-friendly and even faster — `Newtonsoft.Json` remains for legacy/feature-rich scenarios.

---

## Quick Reference

| API | Purpose |
|-----|---------|
| `JsonSerializer.Serialize(obj)` | Object → string |
| `JsonSerializer.SerializeAsync(stream, obj)` | Object → stream (async) |
| `JsonSerializer.Deserialize<T>(json)` | String → object |
| `JsonSerializer.DeserializeAsync<T>(stream)` | Stream → object |
| `JsonSerializerOptions` | Configure naming, ignore-null, indent, converters |
| `[JsonPropertyName("name")]` | Custom JSON property name |
| `[JsonIgnore]` | Skip a property |
| `[JsonConverter(typeof(...))]` | Custom converter |
| `[JsonConstructor]` | Pick constructor for deserialize |
| `JsonSerializerContext` (source gen) | AOT-safe, faster, smaller |
| `XmlSerializer` | XML — legacy but built-in |
| `MessagePack` / `Protobuf` | Binary, compact, schema-based |

---

## Core Concept

JSON is the default wire format for HTTP APIs. .NET ships `System.Text.Json` (STJ), a high-performance UTF-8 reader/writer that beats `Newtonsoft.Json` on speed and allocations. STJ uses **System.Text** types throughout (`Utf8JsonReader`, `Utf8JsonWriter`) — it never converts to UTF-16 strings unless you ask.

By default STJ:
- Uses **camelCase** in ASP.NET Core, **PascalCase** standalone (configurable)
- Is **case-insensitive on read** in ASP.NET Core
- Skips read-only properties on write
- Doesn't include null values unless asked

For AOT/trimming and maximum perf, **source-generated** serializers (`JsonSerializerContext`) compile per-type code at build time — no reflection.

For non-JSON: XML still ships (`XmlSerializer`), but for compact wire formats use **MessagePack** or **Protobuf** (see [[14 - gRPC]]).

---

## Syntax & API

### Basic JSON
```csharp
using System.Text.Json;

var user = new User("Alice", 30);

string json = JsonSerializer.Serialize(user);
// {"Name":"Alice","Age":30}

User? back = JsonSerializer.Deserialize<User>(json);
```

### Options
```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = true,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Converters = { new JsonStringEnumConverter() }
};

string json = JsonSerializer.Serialize(user, options);
```

### Annotations
```csharp
public class User
{
    [JsonPropertyName("user_name")]
    public string Name { get; set; } = "";

    [JsonIgnore]
    public string Password { get; set; } = "";

    [JsonPropertyOrder(1)]
    public int Age { get; set; }

    [JsonConverter(typeof(JsonStringEnumConverter))]
    public Role Role { get; set; }
}
```

### Custom converter
```csharp
public sealed class DateOnlyJsonConverter : JsonConverter<DateOnly>
{
    public override DateOnly Read(ref Utf8JsonReader reader, Type _, JsonSerializerOptions __)
        => DateOnly.Parse(reader.GetString()!);

    public override void Write(Utf8JsonWriter writer, DateOnly value, JsonSerializerOptions _)
        => writer.WriteStringValue(value.ToString("yyyy-MM-dd"));
}
```

### Streaming (async, large JSON)
```csharp
// Write
await using var fs = File.Create("users.json");
await JsonSerializer.SerializeAsync(fs, users);

// Read
await using var fs2 = File.OpenRead("users.json");
var loaded = await JsonSerializer.DeserializeAsync<List<User>>(fs2);
```

### Source generator (AOT-safe + fast)
```csharp
using System.Text.Json.Serialization;

[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = false)]
public partial class AppJsonContext : JsonSerializerContext { }

// Usage
string json = JsonSerializer.Serialize(user, AppJsonContext.Default.User);
User? u = JsonSerializer.Deserialize(json, AppJsonContext.Default.User);
```

### Polymorphism (.NET 7+)
```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "$type")]
[JsonDerivedType(typeof(Dog), typeDiscriminator: "dog")]
[JsonDerivedType(typeof(Cat), typeDiscriminator: "cat")]
public abstract class Animal { public string Name { get; set; } = ""; }

public class Dog : Animal { public string Breed { get; set; } = ""; }
public class Cat : Animal { public bool Indoor { get; set; } }

Animal a = new Dog { Name = "Rex", Breed = "Husky" };
string json = JsonSerializer.Serialize(a);
// {"$type":"dog","Breed":"Husky","Name":"Rex"}
```

### XML
```csharp
using System.Xml.Serialization;

var ser = new XmlSerializer(typeof(User));

await using var fs = File.Create("user.xml");
ser.Serialize(fs, user);

await using var fs2 = File.OpenRead("user.xml");
var loaded = (User?)ser.Deserialize(fs2);
```

### Newtonsoft.Json (when STJ doesn't fit)
```csharp
using Newtonsoft.Json;

string json = JsonConvert.SerializeObject(user, Formatting.Indented);
User? back = JsonConvert.DeserializeObject<User>(json);
```

---

## Common Patterns

```csharp
// Pattern: web API client with STJ
public async Task<User?> GetUserAsync(int id)
{
    using var resp = await _http.GetAsync($"/users/{id}");
    resp.EnsureSuccessStatusCode();
    return await resp.Content.ReadFromJsonAsync<User>();
}

await _http.PostAsJsonAsync("/users", newUser);
```

```csharp
// Pattern: deserialize unknown JSON with JsonElement
using var doc = JsonDocument.Parse(json);
JsonElement root = doc.RootElement;
string name = root.GetProperty("name").GetString()!;
int age = root.GetProperty("age").GetInt32();
```

```csharp
// Pattern: roundtrip-safe DateTime — UTC + ISO 8601
var options = new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter() }
};
// .NET serializes DateTime in ISO 8601 by default; ensure UTC end-to-end
```

---

## Gotchas & Tips

- **Default STJ is camelCase only in ASP.NET Core** — in console/library code it's PascalCase. Be explicit with `JsonNamingPolicy.CamelCase`.
- **Read-only properties aren't deserialized** unless they're init-only or have a setter.
- **Records work great** — STJ uses the primary constructor automatically.
- **Reference loops** crash STJ by default — use `ReferenceHandler.IgnoreCycles` or `Preserve`.
- **`JsonSerializerOptions` is expensive to create** — create once, cache, reuse. Don't allocate per request.
- **Newtonsoft is slower and allocates more** — but it has features STJ lacks (`JObject`/`JArray` flexibility, `[JsonExtensionData]`, custom contract resolvers). Use it where you need them.
- **Don't expose entities directly** — use DTOs to control serialization shape and avoid leaking domain internals.
- **Big payloads → use streaming** (`SerializeAsync`/`DeserializeAsync` over a `Stream`). Avoid loading 100 MB into a string.
- **For AOT/trimmed apps**, you must use source-generated context — reflection-based serialization will fail at runtime.

---

## See Also

- [[09 - File IO]]
- [[15 - REST API]]
- [[15 - Source Generators]]
- [[14 - gRPC]]
