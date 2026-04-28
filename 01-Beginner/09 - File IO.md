---
tags: [dotnet, beginner, csharp]
aliases: [File IO, Streams]
level: Beginner
---

# File IO

> **One-liner**: `System.IO` provides `File`/`Directory` for high-level operations and `Stream`/`StreamReader`/`StreamWriter` for buffered I/O — always prefer **async** methods on the network/disk hot path.

---

## Quick Reference

| Task | API |
|------|-----|
| Read whole file → string | `File.ReadAllText(path)` |
| Read whole file → bytes | `File.ReadAllBytes(path)` |
| Read lines (lazy) | `File.ReadLines(path)` |
| Read all lines (eager) | `File.ReadAllLines(path)` |
| Write string → file | `File.WriteAllText(path, text)` |
| Append text | `File.AppendAllText(path, text)` |
| Open stream (read) | `File.OpenRead(path)` |
| Open stream (write) | `File.OpenWrite(path)` |
| Copy / Move / Delete | `File.Copy / Move / Delete` |
| Exists | `File.Exists(path)` / `Directory.Exists` |
| List files | `Directory.GetFiles(path, "*.txt")` |
| List subdirs | `Directory.GetDirectories(path)` |
| Combine paths | `Path.Combine("a", "b")` |
| Get extension | `Path.GetExtension(path)` |
| Temp file | `Path.GetTempFileName()` |
| Async variants | `*Async`: `ReadAllTextAsync`, etc. |

---

## Core Concept

The simple `File.ReadAllText`/`WriteAllText` APIs are fine for small config files. For anything larger or in a server, use **streams** — they're buffered, memory-efficient, and have async variants.

`StreamReader`/`StreamWriter` wrap a `Stream` with text encoding (UTF-8 by default in modern .NET). For binary, use `BinaryReader`/`BinaryWriter` or just read raw bytes.

**Always use `Path.Combine`** — manually concatenating with `/` or `\` breaks cross-platform.

**Always use `using`** with streams — they hold OS file handles that must be released.

---

## Syntax & API

### Quick reads/writes
```csharp
// Whole file at once
string text = await File.ReadAllTextAsync("notes.txt");
byte[] bytes = await File.ReadAllBytesAsync("data.bin");
string[] lines = await File.ReadAllLinesAsync("log.txt");

await File.WriteAllTextAsync("out.txt", "Hello");
await File.WriteAllBytesAsync("out.bin", bytes);
await File.AppendAllTextAsync("log.txt", $"{DateTime.UtcNow:o}\n");
```

### Streaming a large file
```csharp
// Read line-by-line — doesn't load whole file
await foreach (var line in File.ReadLinesAsync("huge.csv"))
{
    Process(line);
}

// Or with explicit StreamReader
using var reader = new StreamReader("huge.csv");
string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    Process(line);
}
```

### Writing a stream
```csharp
using var writer = new StreamWriter("out.csv");
await writer.WriteLineAsync("name,age");
await writer.WriteLineAsync("Alice,30");
// disposed → flushed and closed
```

### Binary
```csharp
using var stream = File.Create("data.bin");
using var writer = new BinaryWriter(stream);
writer.Write(42);
writer.Write(3.14);
writer.Write("Hello");

using var rstream = File.OpenRead("data.bin");
using var reader = new BinaryReader(rstream);
int n = reader.ReadInt32();
double d = reader.ReadDouble();
string s = reader.ReadString();
```

### Paths
```csharp
string root = "/var/data";
string sub  = Path.Combine(root, "logs", "2025.txt");  // /var/data/logs/2025.txt

string ext = Path.GetExtension("a.tar.gz");            // ".gz"
string name = Path.GetFileNameWithoutExtension("a.txt"); // "a"
string dir = Path.GetDirectoryName("/var/log/x.txt");  // "/var/log"
string full = Path.GetFullPath("./relative");          // absolute

string temp = Path.GetTempFileName();                  // creates empty temp file
string tempDir = Path.GetTempPath();
```

### Directories
```csharp
Directory.CreateDirectory("./out/logs");               // recursive, no-op if exists

foreach (var f in Directory.EnumerateFiles("./logs", "*.log", SearchOption.AllDirectories))
{
    Console.WriteLine(f);
}

if (Directory.Exists("./old"))
    Directory.Delete("./old", recursive: true);
```

### Stream copying
```csharp
using var src = File.OpenRead("source.dat");
using var dst = File.Create("copy.dat");
await src.CopyToAsync(dst);
```

---

## Common Patterns

```csharp
// Pattern: write atomically (temp file + rename)
public async Task SaveAtomicAsync(string path, string content)
{
    string tmp = path + ".tmp";
    await File.WriteAllTextAsync(tmp, content);
    File.Move(tmp, path, overwrite: true);
}
```

```csharp
// Pattern: read-process-write large file with constant memory
using var input  = File.OpenText("in.csv");
using var output = File.CreateText("out.csv");

string? line;
while ((line = await input.ReadLineAsync()) != null)
{
    var transformed = Transform(line);
    await output.WriteLineAsync(transformed);
}
```

---

## Gotchas & Tips

- **Always `await ...Async()` methods** in server code — sync I/O blocks a thread pool thread.
- **`File.ReadAllLines` eagerly loads everything** — for large files use `ReadLines` (lazy `IEnumerable<string>`).
- **Default encoding in .NET 6+ is UTF-8 (no BOM)** — explicitly pass `Encoding.UTF8` if you need a BOM.
- **Path separators** — use `Path.Combine`, `Path.DirectorySeparatorChar`, never hardcode `/` or `\`.
- **Don't trust `File.Exists` to gate operations** — by the time you open the file, it may be gone (TOCTOU race). Just open and catch `FileNotFoundException`.
- **`FileStream` buffer size matters** — defaults are fine for most use; for huge reads, pass a larger buffer or `useAsync: true`.

---

## See Also

- [[06 - Async and Await]]
- [[10 - IDisposable and Resource Mgmt]]
- [[09 - Channels and Pipelines]]
