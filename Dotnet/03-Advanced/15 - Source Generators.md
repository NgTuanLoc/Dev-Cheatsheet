---
tags: [dotnet, advanced, csharp, performance]
aliases: [Roslyn Source Generators, Incremental Generators, JSON Source Gen, Regex Source Gen]
level: Advanced
---

# Source Generators

> **One-liner**: Compile-time C# generators that produce additional code into the user's compilation — used by .NET itself for **JSON serialization**, **regex**, **logging**, and **observability** to eliminate reflection and enable AOT.

---

## Quick Reference

| Built-in generator | What it generates |
|--------------------|-------------------|
| `[JsonSerializable]` | System.Text.Json source-gen contract → no reflection |
| `[GeneratedRegex(...)]` | Compiled regex matcher at build time |
| `[LoggerMessage(...)]` | Strongly-typed, alloc-free `ILogger` extensions |
| `[LibraryImport(...)]` | Replaces `[DllImport]` with marshalling source |
| `[GeneratedComInterface]` | COM interop |
| `[StringSyntax]` analyzer | (analyzer, not generator — but related Roslyn tooling) |

| Custom generator API | Use |
|----------------------|-----|
| `IIncrementalGenerator` | Modern incremental generator (preferred) |
| `ISourceGenerator` | Original v1 API — superseded |
| `IncrementalValuesProvider<T>` | Incremental input pipeline |
| `RegisterSourceOutput` | Emit generated code |
| `ForAttributeWithMetadataName(...)` | Filter syntax to attributed types |
| `Microsoft.CodeAnalysis.CSharp` | Roslyn API for `Compilation`, `Symbols`, `Syntax` |

---

## Core Concept

A source generator is a class library that the C# compiler invokes during build. It receives the user's source, can inspect syntax + symbols, and emits new C# files that get compiled together. Generators **never modify** existing code — they only add — which keeps the model predictable and IDE-friendly.

Modern generators implement **`IIncrementalGenerator`**: you describe a *pipeline* (`SyntaxProvider → ForAttributeWithMetadataName → CombineWith → Select`), and the framework caches each stage so a typing-keystroke recompile only re-runs what changed. This is the difference between "Roslyn cooks for 3 seconds" and "Roslyn is invisible".

The .NET team uses generators heavily: **`System.Text.Json`** source-gen replaces reflection with hand-written-style serializers (faster, AOT-safe). **`[GeneratedRegex]`** compiles patterns at build time. **`[LoggerMessage]`** emits zero-allocation, strongly-typed log methods. The pattern: anywhere you'd reach for reflection in a hot path, prefer source-gen.

---

## Syntax & API

### System.Text.Json source-gen
```csharp
[JsonSerializable(typeof(User))]
[JsonSerializable(typeof(List<User>))]
public partial class AppJsonContext : JsonSerializerContext { }

// Use
string json = JsonSerializer.Serialize(user, AppJsonContext.Default.User);
var users = JsonSerializer.Deserialize(json, AppJsonContext.Default.ListUser);

// In ASP.NET Core (.NET 8+)
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

### GeneratedRegex
```csharp
public partial class EmailValidator
{
    [GeneratedRegex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailRegex();

    public static bool IsValid(string s) => EmailRegex().IsMatch(s);
}
```

### LoggerMessage
```csharp
public static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
        Message = "Order {OrderId} placed by user {UserId} (total {Total})")]
    public static partial void OrderPlaced(this ILogger logger, int orderId, int userId, decimal total);

    [LoggerMessage(EventId = 1002, Level = LogLevel.Warning,
        Message = "Slow request to {Endpoint}: {Ms} ms")]
    public static partial void SlowRequest(this ILogger logger, string endpoint, long ms);
}

// Usage — zero-allocation, structured
logger.OrderPlaced(orderId, userId, total);
```

### LibraryImport (replaces DllImport)
```csharp
public static partial class Native
{
    [LibraryImport("libc", StringMarshalling = StringMarshalling.Utf8)]
    public static partial int getpid();

    [LibraryImport("user32.dll", EntryPoint = "MessageBoxW", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBox(IntPtr hWnd, string text, string caption, int type);
}
```

### Custom generator — incremental
```csharp
[Generator]
public sealed class HelloGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext ctx)
    {
        // 1. Filter to types annotated with [Hello]
        var classes = ctx.SyntaxProvider
            .ForAttributeWithMetadataName(
                "Sample.HelloAttribute",
                predicate: static (node, _) => node is ClassDeclarationSyntax,
                transform: static (gctx, _) =>
                {
                    var c = (ClassDeclarationSyntax)gctx.TargetNode;
                    return new ClassInfo(c.Identifier.Text,
                        ((INamedTypeSymbol)gctx.TargetSymbol).ContainingNamespace.ToDisplayString());
                });

        // 2. Emit one file per type
        ctx.RegisterSourceOutput(classes, (spc, info) =>
        {
            var src = $$"""
                namespace {{info.Namespace}};

                partial class {{info.Name}}
                {
                    public string Hello() => "Hello from {{info.Name}}!";
                }
                """;
            spc.AddSource($"{info.Name}.g.cs", src);
        });
    }

    private record ClassInfo(string Name, string Namespace);
}
```

### Generator project file
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.10.0" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.11.0" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

### Reference the generator from a consumer project
```xml
<ItemGroup>
  <ProjectReference Include="..\HelloGen\HelloGen.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

### View generated source
```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>$(BaseIntermediateOutputPath)Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

---

## Common Patterns

```csharp
// Pattern: marker attribute lives in user code (or via embedded source)
ctx.RegisterPostInitializationOutput(spc => spc.AddSource(
    "HelloAttribute.g.cs",
    """
    namespace Sample;
    [System.AttributeUsage(System.AttributeTargets.Class)]
    internal sealed class HelloAttribute : System.Attribute { }
    """));
```

```csharp
// Pattern: DTO mapper generator
[Generator]
public sealed class MapperGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext ctx)
    {
        var pairs = ctx.SyntaxProvider.ForAttributeWithMetadataName(
            "MapToAttribute",
            (n, _) => n is ClassDeclarationSyntax,
            (gctx, _) => /* extract source/target type pair */);
        ctx.RegisterSourceOutput(pairs, (spc, info) => spc.AddSource(
            $"{info.SourceName}_To_{info.TargetName}.g.cs",
            EmitMapper(info)));
    }
}
```

```csharp
// Pattern: combine multiple inputs
var combined = classes.Combine(ctx.CompilationProvider);
ctx.RegisterSourceOutput(combined, (spc, t) => Emit(spc, t.Left, t.Right));
```

---

## Gotchas & Tips

- **Generators target `netstandard2.0`** — they run inside the compiler host. Use polyfill packages for newer features.
- **Make types `record` or `IEquatable<T>`** in your pipeline — incremental caching depends on equality. A non-equatable model busts the cache and recompiles everything.
- **Don't use `Compilation.GetTypeByMetadataName` in the predicate** — it forces full semantic eval. Use `ForAttributeWithMetadataName` (Roslyn 4.4+) for fast attribute discovery.
- **Generators see source as it's typed** — emit **`partial`** classes and methods so users can extend the generated code.
- **No File I/O at generation time** — the host may run inside an IDE sandbox. Read files via `AdditionalFiles` if you must.
- **Diagnostics are first-class** — `spc.ReportDiagnostic(...)` to surface user errors with squiggles.
- **Test with `CSharpGeneratorDriver`** — `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing` package + xUnit.
- **Source-gen JSON is not a free swap** — features like `JsonExtensionData`, polymorphism, and converter discovery may need configuration. Test before disabling reflection.
- **`[LoggerMessage]` is the cheapest log call** in .NET — beats `_log.LogInformation(string, object[])` because it skips boxing and parameter array allocation.
- **`GeneratedRegex` is faster than runtime `new Regex(...)`** for cold start — pre-compiles to IL at build time.
- **Don't add expensive logic to the predicate** — it runs on every keystroke. Move heavy work to `transform` (still incremental but less hot).

---

## See Also

- [[11 - Reflection and Attributes]]
- [[12 - Serialization]]
- [[18 - Logging]]
- [[16 - Native Interop and AOT]]
