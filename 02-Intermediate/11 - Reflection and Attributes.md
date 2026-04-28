---
tags: [dotnet, intermediate, csharp]
aliases: [Reflection, Attributes, Metadata]
level: Intermediate
---

# Reflection and Attributes

> **One-liner**: **Reflection** lets code inspect and invoke types/members at runtime; **attributes** are metadata you decorate code with — both power frameworks (DI, serializers, ORMs, test runners).

---

## Quick Reference

| API | Purpose |
|-----|---------|
| `typeof(T)` | Get `Type` at compile time |
| `obj.GetType()` | Get runtime type |
| `Type.GetProperties()` | List public instance properties |
| `Type.GetMethods()` | List methods |
| `Type.GetMethod("X")` | One method by name |
| `Type.GetCustomAttributes<T>()` | Read attribute metadata |
| `Activator.CreateInstance(t)` | New instance via reflection |
| `MethodInfo.Invoke(obj, args)` | Call a method |
| `PropertyInfo.GetValue / SetValue` | Read/write property |
| `Assembly.GetExecutingAssembly()` | Current DLL |
| `Assembly.GetTypes()` | All types in assembly |
| `Type.IsAssignableTo(...)` | Is-a check |
| `[AttributeUsage(AttributeTargets...)]` | Restrict where attribute applies |

---

## Core Concept

Every assembly carries **metadata** describing its types, members, and attributes. Reflection is the API to read it. With reflection you can build generic frameworks: a JSON serializer that handles any type, a DI container that auto-wires constructors, a test runner that finds `[Test]` methods.

**Attributes** are user-defined classes inheriting `Attribute` that decorate code (`[MyAttr]`). They're stored in metadata; nothing executes at compile time — code reads them via reflection (or, increasingly, via [[15 - Source Generators]]).

Reflection is **slow** compared to direct calls (10–100×). For hot paths, cache `MethodInfo`/`PropertyInfo` lookups, or build delegates with `Expression.Compile`, or use source generators to avoid reflection entirely.

---

## Syntax & API

### Inspect a type
```csharp
Type t = typeof(User);
// or: Type t = user.GetType();

Console.WriteLine(t.FullName);                    // Full name with namespace
Console.WriteLine(t.IsClass);                     // true
Console.WriteLine(t.BaseType?.Name);              // "Object"

foreach (var p in t.GetProperties())
    Console.WriteLine($"{p.PropertyType.Name} {p.Name}");

foreach (var m in t.GetMethods(BindingFlags.Public | BindingFlags.Instance))
    Console.WriteLine($"{m.ReturnType.Name} {m.Name}({string.Join(", ", m.GetParameters().Select(x => x.ParameterType.Name))})");
```

### Read/write properties
```csharp
var p = t.GetProperty("Name")!;
string? name = (string?)p.GetValue(user);
p.SetValue(user, "Alice");
```

### Invoke a method
```csharp
var m = t.GetMethod("Greet")!;
var result = m.Invoke(user, new object?[] { "Hello" });
```

### Create instance dynamically
```csharp
var instance = Activator.CreateInstance(typeof(User), "Alice", 30);
// or:
var ctor = typeof(User).GetConstructor(new[] { typeof(string), typeof(int) })!;
var instance2 = ctor.Invoke(new object[] { "Alice", 30 });
```

### Define a custom attribute
```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class, AllowMultiple = false)]
public sealed class RoleAttribute : Attribute
{
    public string Role { get; }
    public RoleAttribute(string role) => Role = role;
}

// Apply
[Role("Admin")]
public class AdminController { }

// Read
var attr = typeof(AdminController).GetCustomAttribute<RoleAttribute>();
Console.WriteLine(attr?.Role);                    // "Admin"
```

### Common built-in attributes
```csharp
[Obsolete("Use V2 instead", error: false)]
public void OldMethod() { }

[Serializable]
public class Dto { }

[DebuggerDisplay("{Name} ({Age})")]
public class Person { /* ... */ }

[CallerMemberName] string member = ""    // injects caller name automatically
```

### Generic reflection
```csharp
Type listOpen = typeof(List<>);                  // unbound
Type listClosed = listOpen.MakeGenericType(typeof(int));
var list = Activator.CreateInstance(listClosed); // List<int> instance

// Method
var addOpen = listClosed.GetMethod("Add")!;
addOpen.Invoke(list, new object?[] { 42 });
```

### Scan assembly for plugins
```csharp
var pluginTypes = AppDomain.CurrentDomain
    .GetAssemblies()
    .SelectMany(a => a.GetTypes())
    .Where(t => typeof(IPlugin).IsAssignableFrom(t)
             && !t.IsAbstract
             && !t.IsInterface);

foreach (var t in pluginTypes)
    services.AddSingleton(typeof(IPlugin), t);
```

### Build fast delegate (avoid Invoke cost)
```csharp
PropertyInfo p = typeof(User).GetProperty("Age")!;
var getter = (Func<User, int>)Delegate.CreateDelegate(
    typeof(Func<User, int>), p.GetGetMethod()!);

int age = getter(user);                          // ~native speed
```

---

## Common Patterns

```csharp
// Pattern: validation via attributes
[AttributeUsage(AttributeTargets.Property)]
public class RequiredAttribute : Attribute { }

public class User
{
    [Required] public string Name { get; set; } = "";
    [Required] public string Email { get; set; } = "";
    public int? Age { get; set; }
}

public IEnumerable<string> Validate(object obj)
{
    foreach (var p in obj.GetType().GetProperties())
    {
        if (p.GetCustomAttribute<RequiredAttribute>() != null
            && p.GetValue(obj) is null or "")
        {
            yield return $"{p.Name} is required";
        }
    }
}
```

```csharp
// Pattern: register all implementations of an interface
public static IServiceCollection AddAllOf<TInterface>(this IServiceCollection services)
{
    var impls = typeof(TInterface).Assembly.GetTypes()
        .Where(t => typeof(TInterface).IsAssignableFrom(t) && !t.IsAbstract);

    foreach (var t in impls)
        services.AddTransient(typeof(TInterface), t);

    return services;
}
```

---

## Gotchas & Tips

- **Reflection is slow** — `MethodInfo.Invoke` allocates an `object[]` for args, boxes value types, and walks lookup tables. Cache and reuse.
- **Trim/AOT-unsafe** — `dotnet publish` with `<PublishTrimmed>true</PublishTrimmed>` or `<PublishAot>true</PublishAot>` may strip types reflection looks for. Source generators are the modern alternative — see [[15 - Source Generators]] and [[16 - Native Interop and AOT]].
- **`GetProperties()` doesn't include private** unless you pass `BindingFlags.NonPublic | BindingFlags.Instance`.
- **`Activator.CreateInstance` requires a public parameterless constructor by default** — use the `ConstructorInfo.Invoke` overload for parameters.
- **Avoid string-based reflection in production code** — use `nameof(User.Name)` so the compiler catches typos and renames update them.
- **Attributes have no behavior** — they're just metadata. Behavior comes from the code that reads them.
- **`AttributeUsage` matters** — without `AllowMultiple = true`, you can only apply your attribute once per target.
- **`[Conditional("DEBUG")]`** removes calls in non-DEBUG builds — useful for assertions.

---

## See Also

- [[15 - Source Generators]]
- [[16 - Native Interop and AOT]]
- [[12 - Serialization]]
- [[13 - Dependency Injection]]
