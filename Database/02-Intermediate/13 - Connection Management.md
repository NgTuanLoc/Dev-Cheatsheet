---
tags: [database, intermediate, postgresql, dotnet, performance]
aliases: [Connection Pool, PgBouncer, Npgsql Pool, Connection String]
level: Intermediate
---

# Connection Management

> **One-liner**: A connection is expensive — pool them, keep them short, and use a transaction-aware proxy (PgBouncer) when serverless or microservices push the count too high.

---

## Quick Reference

| Concept | Postgres / .NET fact |
|---------|----------------------|
| Each connection = one OS process in Postgres | ~5–10 MB RAM each |
| Default `max_connections` | 100 |
| Pool max in Npgsql | `Maximum Pool Size=100` (default) |
| PgBouncer modes | `session` / `transaction` / `statement` |
| Connection string spec | key/value pairs separated by `;` |
| Idle timeout | both pool-side and Postgres-side configurable |
| Async I/O | always — `await OpenAsync`, never `Open` |

---

## Core Concept

A **connection** is a TCP session + an authenticated Postgres backend process. Opening one is expensive (TCP handshake, TLS, auth, plus a fork on the server). You almost never want to open and close on every query.

A **connection pool** keeps a set of open connections idle and hands them to callers. Npgsql has a built-in pool — pooling is enabled by default. Each unique connection string gets its own pool.

When you have **many short-lived clients** (serverless, lambdas, container scale-out), even pools can't keep up — you'd open too many backends. **PgBouncer** is a connection multiplexer: many client connections share fewer real backends. In **transaction mode** it allocates a backend per transaction, releasing it on commit — extremely efficient but disallows session-scoped features (`SET LOCAL`, prepared statements without proper handling, advisory locks).

The right pool size is small: a Postgres backend is CPU/disk/RAM-bound. The classic formula is `pool_size ≈ ((cores × 2) + spindles)` per node. Most apps need 10–30 active connections, not 200.

---

## Syntax & API

### Connection string (Npgsql)
```text
Host=localhost;Port=5432;Database=shop;Username=app;Password=secret;
SSL Mode=Require;Trust Server Certificate=true;
Pooling=true;
Minimum Pool Size=0;
Maximum Pool Size=20;
Connection Idle Lifetime=300;
Command Timeout=30;
Application Name=shop-api;
```

### Open / dispose properly (.NET)
```csharp
await using var conn = new NpgsqlConnection(connStr);
await conn.OpenAsync(ct);

await using var cmd = new NpgsqlCommand("SELECT email FROM users WHERE id = @id", conn);
cmd.Parameters.AddWithValue("id", 42);

await using var reader = await cmd.ExecuteReaderAsync(ct);
while (await reader.ReadAsync(ct))
{
    Console.WriteLine(reader.GetString(0));
}
// Disposed in reverse order: reader → cmd → conn (returned to pool)
```

### Always parameterize
```csharp
// WRONG — SQL injection
var sql = $"SELECT * FROM users WHERE email = '{email}'";

// RIGHT — parameter
var cmd = new NpgsqlCommand("SELECT * FROM users WHERE email = @e", conn);
cmd.Parameters.AddWithValue("e", email);
```

### NpgsqlDataSource (Npgsql 7+ — preferred)
```csharp
// Program.cs
var dataSource = NpgsqlDataSource.Create(connStr);
builder.Services.AddSingleton(dataSource);

// Usage
public class UserRepo(NpgsqlDataSource db)
{
    public async Task<string?> GetEmailAsync(int id, CancellationToken ct)
    {
        await using var cmd = db.CreateCommand("SELECT email FROM users WHERE id = $1");
        cmd.Parameters.AddWithValue(id);
        return (string?)await cmd.ExecuteScalarAsync(ct);
    }
}
// DataSource owns the pool; no need to manage connections manually for one-shot queries.
```

### EF Core lifetime
```csharp
builder.Services.AddDbContext<ShopContext>(opts =>
    opts.UseNpgsql(connStr));
// Default: Scoped (per HTTP request). DbContext is NOT thread-safe.
// For parallel work use AddDbContextFactory<T>().
```

### PgBouncer in front (transaction mode)
```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
shop = host=localhost port=5432 dbname=shop

[pgbouncer]
listen_addr  = *
listen_port  = 6432
auth_type    = md5
auth_file    = /etc/pgbouncer/userlist.txt
pool_mode    = transaction       # session | transaction | statement
max_client_conn          = 5000  # apps connecting to PgBouncer
default_pool_size        = 25    # backends to Postgres per (db,user)
reserve_pool_size        = 5
server_idle_timeout      = 600
```

App connects to port 6432 instead of 5432; the rest of the connection string is the same.

### Inspect activity / pool
```sql
-- Live backends, what they're doing
SELECT pid, usename, application_name, state, wait_event,
       NOW() - query_start AS running_for, LEFT(query, 80) AS q
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Backend count vs limit
SELECT count(*) FILTER (WHERE state IS NOT NULL) AS active,
       (SELECT setting::INT FROM pg_settings WHERE name='max_connections') AS max
FROM pg_stat_activity;
```

```bash
# PgBouncer admin console
psql -h pgbouncer -p 6432 pgbouncer
# Inside:
SHOW POOLS;
SHOW CLIENTS;
SHOW STATS;
```

---

## Common Patterns

```csharp
// Pattern: command timeout per query, not just per connection
await using var cmd = new NpgsqlCommand("SELECT ...", conn);
cmd.CommandTimeout = 5;            // seconds; overrides connection-level default
```

```csharp
// Pattern: cancellation through the whole stack
public async Task<List<User>> GetActiveAsync(CancellationToken ct)
{
    await using var conn = await db.OpenConnectionAsync(ct);
    await using var cmd  = new NpgsqlCommand("SELECT id, email FROM users WHERE is_active", conn);
    await using var rdr  = await cmd.ExecuteReaderAsync(ct);
    var list = new List<User>();
    while (await rdr.ReadAsync(ct))
        list.Add(new User { Id = rdr.GetInt32(0), Email = rdr.GetString(1) });
    return list;
}
```

```ini
# Pattern: separate pools per workload
# read-only replica connection
Host=replica.local;Database=shop;Application Name=shop-api-read;
# primary
Host=primary.local;Database=shop;Application Name=shop-api-write;
# Two distinct connection strings → two distinct pools
```

---

## Gotchas & Tips

- **Don't `Open()` on the hot path** — pooled connections, but not opening, also costs round-trips for `SELECT version()` resets if `Reset On Open` is on.
- **Always `await using`** — leaked connections silently hold backend slots and bloat the pool.
- **Pool is per connection string** — adding `Application Name=foo` makes a *different* pool. Useful for distinguishing read/write paths.
- **Postgres `max_connections` is hard ceiling** — going beyond it errors out new connections. Pool limit ≤ `max_connections − reserved_for_superuser`.
- **DBs hate idle-in-transaction backends** — they hold locks and snapshots. Set `idle_in_transaction_session_timeout = 60s` to auto-kill stragglers.
- **Use `PgBouncer transaction mode` for serverless / many small clients** — but don't use prepared statements without `Server Compatibility Mode=NoTypeLoading` or pgbouncer 1.21+ prepared-statement support.
- **Don't reuse a connection across threads** — Npgsql's pool is thread-safe, but a single `NpgsqlConnection` is not.
- **`NpgsqlDataSource` (v7+) is the modern API** — handles the pool for you, supports binding type maps, and is the recommended DI pattern.
- **Watch for connection leaks under exceptions** — `await using` is critical. A throw in the middle without proper using → leaked connection.
- **Keep transactions short** — open conn, BEGIN, do work, COMMIT, return conn. No user think-time inside.

---

## See Also

- [[14 - ADO.NET and Dapper]]
- [[02 - Transactions and ACID]]
- [[09 - Performance Tuning]]
- [[17 - Cloud Databases]]
