---
tags: [database, intermediate, postgresql, dotnet]
aliases: [Npgsql, Dapper, Micro-ORM, ADO.NET]
level: Intermediate
---

# ADO.NET and Dapper

> **One-liner**: ADO.NET is the raw .NET data API; Dapper is a thin micro-ORM that adds parameter binding and result mapping while staying close to the SQL.

---

## Quick Reference

| Layer | What it gives you | Cost |
|-------|-------------------|------|
| **ADO.NET (Npgsql)** | Connections, commands, parameters, readers | Verbose; you map by hand |
| **Dapper** | `Query<T>`, `QuerySingle<T>`, `Execute` over IDbConnection | Tiny — same SQL, less boilerplate |
| **EF Core** | Full ORM: entities, change tracking, LINQ, migrations | More features, more abstraction (see Dotnet topic) |

| Dapper extension | What it does |
|------------------|--------------|
| `Query<T>(sql, params)` | Buffered list |
| `QueryAsync<T>(sql, params)` | Same, async |
| `QueryFirst[OrDefault]Async<T>` | One row |
| `QuerySingle[OrDefault]Async<T>` | Exactly one row (throws if more) |
| `ExecuteAsync(sql, params)` | INSERT/UPDATE/DELETE; returns rows-affected |
| `ExecuteScalarAsync<T>` | Single value |
| `QueryMultipleAsync` | Multi-resultset |
| `Query<A, B, T>(sql, map, splitOn)` | Multi-table mapping |

---

## Core Concept

**ADO.NET** is the bottom layer: `IDbConnection`, `IDbCommand`, `IDbDataReader`. Postgres's provider is **Npgsql**. You write SQL, bind parameters, read rows column-by-column. Maximum control, maximum boilerplate.

**Dapper** is a static-extension-method library on top of `IDbConnection`. It generates a parameter-binding/mapper IL once per query shape, caches it, and runs nearly as fast as hand-written ADO.NET. You still write SQL — Dapper just removes the boilerplate.

Use Dapper when:
- You want SQL-first control (complex joins, CTEs, window functions)
- The mapping is mostly straightforward (column → property)
- You don't need change tracking or migrations from the same library

Use EF Core when:
- You want code-first models, migrations, and LINQ over an ORM
- The domain is well-structured and joins are routine

Many apps use both: EF Core for CRUD + migrations, Dapper for hot read paths and complex queries.

---

## Syntax & API

### Setup
```bash
dotnet add package Npgsql
dotnet add package Dapper
```

### Raw ADO.NET — read
```csharp
await using var conn = new NpgsqlConnection(connStr);
await conn.OpenAsync(ct);

await using var cmd = new NpgsqlCommand(
    "SELECT id, email FROM users WHERE country = @c", conn);
cmd.Parameters.AddWithValue("c", "US");

var users = new List<User>();
await using var rdr = await cmd.ExecuteReaderAsync(ct);
while (await rdr.ReadAsync(ct))
{
    users.Add(new User
    {
        Id    = rdr.GetInt32(0),
        Email = rdr.GetString(1)
    });
}
```

### Same with Dapper
```csharp
using Dapper;

await using var conn = new NpgsqlConnection(connStr);
var users = await conn.QueryAsync<User>(
    "SELECT id, email FROM users WHERE country = @c",
    new { c = "US" });
// Returns IEnumerable<User>; column→property mapped by name (case-insensitive)
```

### Insert returning
```csharp
var sql = @"INSERT INTO users (email, name)
            VALUES (@Email, @Name)
            RETURNING id";
var id = await conn.QuerySingleAsync<int>(sql, new { Email="a@b.com", Name="Alice" });
```

### Update / delete
```csharp
var rows = await conn.ExecuteAsync(
    "UPDATE users SET name = @Name WHERE id = @Id",
    new { Id = 1, Name = "Renamed" });
```

### Transactions
```csharp
await using var conn = new NpgsqlConnection(connStr);
await conn.OpenAsync(ct);
await using var tx = await conn.BeginTransactionAsync(ct);
try
{
    await conn.ExecuteAsync(
        "UPDATE accounts SET balance = balance - @a WHERE id = @i",
        new { a = 100m, i = 1 }, tx);
    await conn.ExecuteAsync(
        "UPDATE accounts SET balance = balance + @a WHERE id = @i",
        new { a = 100m, i = 2 }, tx);
    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

### Multi-mapping (parent + child in one query)
```csharp
var sql = @"
    SELECT u.id, u.email, o.id, o.total
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.id = @id";

var lookup = new Dictionary<int, User>();
await conn.QueryAsync<User, Order, User>(sql,
    (u, o) =>
    {
        if (!lookup.TryGetValue(u.Id, out var user))
        {
            user = u;
            user.Orders = new List<Order>();
            lookup[u.Id] = user;
        }
        if (o is not null) user.Orders.Add(o);
        return user;
    },
    new { id = 1 },
    splitOn: "id");          // boundary column: second "id" starts Order

var user = lookup.Values.SingleOrDefault();
```

### IN-clause expansion
```csharp
var ids = new[] { 1, 2, 3, 4 };
var users = await conn.QueryAsync<User>(
    "SELECT * FROM users WHERE id = ANY(@Ids)",
    new { Ids = ids });
// Postgres-friendly: pass array, use ANY(@Ids). Dapper also supports
// "WHERE id IN @Ids" — but ANY(array) is faster and parameter-friendly.
```

### Bulk insert via COPY (raw Npgsql)
```csharp
await using var importer = await conn.BeginBinaryImportAsync(
    "COPY users (email, name) FROM STDIN (FORMAT BINARY)", ct);
foreach (var u in inputs)
{
    await importer.StartRowAsync(ct);
    await importer.WriteAsync(u.Email, NpgsqlDbType.Text, ct);
    await importer.WriteAsync(u.Name,  NpgsqlDbType.Text, ct);
}
await importer.CompleteAsync(ct);
// Orders of magnitude faster than INSERT loops.
```

### JSONB round-trip
```csharp
// Send a CLR object as JSONB
await conn.ExecuteAsync(
    "INSERT INTO events (payload) VALUES (@p::jsonb)",
    new { p = JsonSerializer.Serialize(new { user = "alice", action = "login" }) });

// Read JSONB
var json = await conn.QuerySingleAsync<string>("SELECT payload FROM events WHERE id = @i", new { i = 1 });
```

---

## Common Patterns

```csharp
// Pattern: thin repository over Dapper
public sealed class UserRepository(NpgsqlDataSource db)
{
    public async Task<User?> ByIdAsync(int id, CancellationToken ct)
    {
        await using var conn = await db.OpenConnectionAsync(ct);
        return await conn.QuerySingleOrDefaultAsync<User>(
            "SELECT id, email, name FROM users WHERE id = @id",
            new { id });
    }

    public async Task<int> CreateAsync(User u, CancellationToken ct)
    {
        await using var conn = await db.OpenConnectionAsync(ct);
        return await conn.QuerySingleAsync<int>(
            @"INSERT INTO users (email, name) VALUES (@Email, @Name) RETURNING id",
            u);
    }
}
```

```csharp
// Pattern: optimistic concurrency via version column
var rows = await conn.ExecuteAsync(@"
    UPDATE users
       SET name = @Name, version = version + 1
     WHERE id = @Id AND version = @Version",
    new { Id = u.Id, u.Name, u.Version });
if (rows == 0) throw new ConcurrencyException();
```

```csharp
// Pattern: dynamic query with whitelisted ORDER BY
var allowedSort = new HashSet<string> { "id", "email", "created_at" };
if (!allowedSort.Contains(sortBy)) sortBy = "id";

var sql = $"SELECT * FROM users ORDER BY {sortBy} LIMIT @lim";
var rows = await conn.QueryAsync<User>(sql, new { lim = 25 });
// String interpolation only on validated input; values still parameterized.
```

---

## Gotchas & Tips

- **Always parameterize** — `$"WHERE x = '{v}'"` is SQL injection. Dapper anonymous objects bind by member name.
- **Anonymous-object names map to `@names`** — `new { Id = 1 }` → `@Id`. Case-insensitive in Postgres but match in C# for clarity.
- **Dapper buffers by default** — for huge result sets pass `buffered: false` and stream.
- **`QuerySingle` vs `QueryFirst`** — `Single` throws on multiple rows; `First` doesn't. Use `Single` to assert uniqueness.
- **`ExecuteAsync` returns rows-affected** — check it for optimistic concurrency or "did I update anything?".
- **Don't reuse `IDbConnection` across threads** — open one per logical operation; the pool gives you fresh ones cheaply.
- **`splitOn` boundaries** in multi-mapping must match actual column order; off-by-one is the #1 mistake.
- **Use `NpgsqlDataSource`** — Npgsql 7+ recommends it; binds type maps once and pools nicely.
- **Bulk inserts: `COPY` >> `INSERT` loops** — even >> a single multi-row INSERT for thousands of rows.
- **JSON columns** — Dapper reads them as `string` by default. Wrap with `JsonSerializer` or use a custom `SqlMapper.TypeHandler`.
- **No magic in Dapper** — if SQL has a typo, you'll see a Postgres error. The library is intentionally thin.

---

## See Also

- [[13 - Connection Management]]
- [[02 - Transactions and ACID]]
- [[12 - Database Migrations]]
- [[18 - JSON and JSONB]]
