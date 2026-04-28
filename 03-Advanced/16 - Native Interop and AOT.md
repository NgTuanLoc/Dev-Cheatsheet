---
tags: [dotnet, advanced, performance, csharp]
aliases: [P/Invoke, DllImport, LibraryImport, Native AOT, Trimming]
level: Advanced
---

# Native Interop and AOT

> **One-liner**: **P/Invoke** lets managed C# call native libraries (use the modern source-gen **`[LibraryImport]`** instead of legacy **`[DllImport]`**), and **Native AOT** publishes a self-contained native binary with no JIT — fast startup, small footprint, but with strict trimming and reflection limits.

---

## Quick Reference

| Interop API | When |
|-------------|------|
| `[LibraryImport]` (.NET 7+) | Modern P/Invoke, source-generated marshalling, AOT-safe |
| `[DllImport]` | Legacy P/Invoke (still works; doesn't generate optimized marshalling) |
| `Marshal` class | Manual memory marshalling helpers |
| `unsafe` + pointers | Direct pointer manipulation |
| `NativeMemory.Alloc` | Native heap allocation (bypasses GC) |
| `GCHandle` | Pin a managed object across native call boundaries |
| `SafeHandle` | RAII wrapper for native resources |
| `[UnmanagedCallersOnly]` | Expose a C# method as a callback for native code |

| AOT API | What |
|---------|------|
| `<PublishAot>true</PublishAot>` | Publish as Native AOT |
| `<TrimMode>full</TrimMode>` | Aggressive trimming |
| `<InvariantGlobalization>true</InvariantGlobalization>` | Drop globalization data |
| `[DynamicallyAccessedMembers(...)]` | Tell trimmer what reflection needs |
| `[RequiresUnreferencedCode("...")]` | Mark methods incompatible with trimming |
| `[RequiresDynamicCode("...")]` | Mark methods incompatible with AOT |
| `dotnet publish -c Release -r linux-x64` | Required: must specify RID for AOT |

---

## Core Concept

P/Invoke is the bridge between managed C# and native C/C++ libraries (`libc`, `Win32`, `OpenGL`, your own `.so/.dll`). You declare a static method with `[LibraryImport("libname")]`, the runtime resolves the symbol at first call and marshals arguments according to the signature. **`[LibraryImport]`** is a **source-gen** version (.NET 7+) — the marshalling code is generated at compile time, removing IL stubs, working under AOT, and reporting issues as build errors.

**Native AOT** compiles your app ahead-of-time into a single platform-native executable. There is **no JIT, no reflection over arbitrary types, no dynamic code generation** at runtime. This means: ~50ms cold start (vs 500ms+ for JIT), ~10–30 MB binary (vs ~100+ MB framework-dependent), zero runtime per-app installation. The cost: features that depend on `Reflection.Emit`, `Activator.CreateInstance` on unknown types, or runtime expression compilation **don't work**.

The trimmer is the gatekeeper: it walks the call graph from `Main` and removes anything unused. **Reflection breaks this analysis** — the trimmer can't see what `Type.GetMethod("Foo")` actually does. Source generators (`System.Text.Json`, `[LoggerMessage]`, MVC source-gen, EF Core compiled models) are how the .NET ecosystem stays AOT-compatible: they replace reflection with statically-known code.

---

## Syntax & API

### LibraryImport (modern P/Invoke)
```csharp
public static partial class Native
{
    [LibraryImport("libc", StringMarshalling = StringMarshalling.Utf8)]
    public static partial int getpid();

    [LibraryImport("kernel32.dll", EntryPoint = "GetCurrentThreadId")]
    public static partial uint GetCurrentThreadId();

    [LibraryImport("user32.dll", EntryPoint = "MessageBoxW", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBox(IntPtr hWnd, string text, string caption, uint type);
}
```

### Struct marshalling
```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Point
{
    public int X;
    public int Y;
}

public static partial class Native
{
    [LibraryImport("mylib")]
    public static partial Point GetCenter();

    [LibraryImport("mylib")]
    public static partial int Distance(in Point a, in Point b);
}
```

### Pinning + spans
```csharp
public static unsafe int Read(byte[] buffer, int offset, int count)
{
    fixed (byte* p = &buffer[offset])
    {
        return Native.read_buf(p, count);
    }
}

[LibraryImport("mylib")]
public static unsafe partial int read_buf(byte* buf, int len);
```

### Native heap allocation
```csharp
unsafe
{
    byte* mem = (byte*)NativeMemory.Alloc(1024);
    try
    {
        // use mem...
    }
    finally
    {
        NativeMemory.Free(mem);
    }
}
```

### SafeHandle (resource RAII)
```csharp
public sealed class FileHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public FileHandle() : base(true) { }

    [LibraryImport("kernel32.dll", EntryPoint = "CloseHandle")]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool CloseHandle(IntPtr h);

    protected override bool ReleaseHandle() => CloseHandle(handle);
}
```

### Function pointer / callback
```csharp
[UnmanagedCallersOnly(CallConvs = new[] { typeof(CallConvCdecl) })]
public static int OnEvent(int code) => code * 2;

// Pass to native
[LibraryImport("mylib")]
public static unsafe partial void SetCallback(delegate* unmanaged[Cdecl]<int, int> fn);

unsafe { Native.SetCallback(&OnEvent); }
```

### Native AOT publish
```bash
dotnet publish -c Release -r linux-x64
# → bin/Release/net9.0/linux-x64/publish/MyApp  (single binary, no JIT)
```

```xml
<!-- csproj -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
  <StripSymbols>true</StripSymbols>
  <OptimizationPreference>Size</OptimizationPreference>
</PropertyGroup>
```

### Trim warnings
```csharp
[RequiresUnreferencedCode("This uses reflection over arbitrary types")]
public static T Deserialize<T>(string json) =>
    JsonSerializer.Deserialize<T>(json)!;   // analyzer warns

[RequiresDynamicCode("Uses Expression.Compile")]
public Func<T, bool> Compile<T>(Expression<Func<T, bool>> expr) => expr.Compile();
```

### Dynamically accessed members
```csharp
public static T Create<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)] T>()
    where T : new() => new T();
```

### AOT-friendly minimal API
```csharp
var b = WebApplication.CreateSlimBuilder(args);   // AOT-tailored builder
b.Services.ConfigureHttpJsonOptions(o => o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = b.Build();
app.MapGet("/users", () => new[] { new User(1, "Alice") });
app.Run();

[JsonSerializable(typeof(User[]))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

### Errno / last-error
```csharp
[LibraryImport("libc", SetLastError = true)]
public static partial int open(string path, int flags);

int fd = Native.open("/tmp/x", 0);
if (fd < 0) throw new Win32Exception(Marshal.GetLastPInvokeError());
```

---

## Common Patterns

```csharp
// Pattern: load native library by RID
[LibraryImport("libssl")]   // resolved per platform: libssl.so.3, libssl.dylib, libssl.dll
public static partial int SSL_init();
```

```csharp
// Pattern: zero-alloc string marshalling using ReadOnlySpan<byte>
[LibraryImport("mylib", EntryPoint = "process")]
public static unsafe partial int Process(byte* utf8, int len);

ReadOnlySpan<byte> utf8 = "hello"u8;
fixed (byte* p = utf8) Native.Process(p, utf8.Length);
```

```csharp
// Pattern: AOT-safe DI registration (no scan)
b.Services.AddSingleton<IUserService, UserService>();
b.Services.AddSingleton<IOrderService, OrderService>();
// ... avoid AddScrutor/Assembly.Scan in AOT
```

```csharp
// Pattern: forward-compat trimmer hint with feature switch
[FeatureSwitchDefinition("Sample.UseReflection")]
public static bool UseReflection => false;

if (UseReflection) { /* trimmed away in AOT */ }
```

---

## Gotchas & Tips

- **Use `[LibraryImport]` over `[DllImport]`** — the source generator gives better errors, AOT compatibility, and lets you skip the `unsafe` boilerplate the marshaller used to generate at runtime.
- **`partial` is required** for `[LibraryImport]` methods because the body is generated.
- **String marshalling** must be explicit: `StringMarshalling.Utf8` for Linux/macOS C APIs, `Utf16` for Win32 wide-string APIs.
- **Pinning is mandatory** when passing managed buffers to native code that retains the pointer past the call. Use `fixed`, `GCHandle.Pinned`, or pinned-heap arrays.
- **Always free what you allocate** — `NativeMemory.Free`, `Marshal.FreeHGlobal`, native library cleanup. The GC ignores native memory.
- **AOT means no plugins, no Reflection.Emit** — evaluate carefully if your stack works (EF, MVC, gRPC, AutoMapper, etc.). Many libraries now ship "AOT-ready" markers.
- **Trim warnings are errors in production** — fix them, don't suppress. They mark genuine runtime failures.
- **Test the AOT build in CI** — JIT runs hide AOT-only failures (`PlatformNotSupportedException` at runtime).
- **`InvariantGlobalization` is harsh but small** — turning it on drops ~30 MB of ICU data; comes back as `CultureNotFoundException` for non-invariant cultures.
- **`-r <RID>` is mandatory for AOT** — there is no portable AOT binary.
- **LLVM-level optimizations** — `OptimizationPreference=Size` shrinks; `Speed` is faster at runtime. Pick based on your target (cold-start CLI vs server).
- **First call is the slowest P/Invoke** — symbol resolution + marshalling stub setup. Don't microbenchmark cold paths.

---

## See Also

- [[15 - Source Generators]]
- [[09 - Memory Management and GC]]
- [[08 - Span and Memory Types]]
- [[06 - Performance Optimization]]
