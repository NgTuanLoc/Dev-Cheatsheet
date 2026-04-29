---
tags: [database, advanced, postgresql, performance]
aliases: [COPY, BULK INSERT, ON CONFLICT, MERGE, Bulk Loading]
level: Advanced
---

# Bulk Operations

> **One-liner**: For thousands+ rows, `COPY` beats `INSERT` by 10–100×; `INSERT … ON CONFLICT` and `MERGE` (PG 15+) handle upserts cleanly; batch deletes in chunks to avoid bloated WAL.

---

## Quick Reference

| Operation | Best tool |
|-----------|-----------|
| Insert N rows from app | `COPY` (binary or text); fall back to multi-row `INSERT` |
| Insert from CSV | `COPY FROM 'file.csv' CSV HEADER` |
| Upsert (insert or update) | `INSERT … ON CONFLICT (key) DO UPDATE SET …` |
| Multi-target upsert | `MERGE` (PG 15+) |
| Bulk update | `UPDATE … FROM (VALUES …)` or `WITH … UPDATE …` |
| Bulk delete | chunked DELETE; or partition + DROP for huge ranges |
| Bulk transform | `INSERT … SELECT` from staging table |
| Foreign data load | `COPY` from staging schema; minimize indexes during load |

| `COPY` formats | |
|----------------|---|
| `TEXT` | default; tab-separated by default |
| `CSV` | with header, quoting, escaping |
| `BINARY` | fastest; tightly typed; not human-readable |

---

## Core Concept

`INSERT` row-by-row pays per-row overhead: parse, plan, transaction tracking, WAL. `COPY` streams bytes into the table with minimal per-row work. The speed difference is huge — a million-row load takes minutes via `INSERT`, seconds via `COPY`.

For **upsert** (insert if new, update if existing), Postgres has had `INSERT … ON CONFLICT` since 9.5. PG 15 added `MERGE` for the SQL-standard syntax (more powerful for multi-action: insert + update + delete in one).

For **bulk delete** of huge ranges, deleting in one statement creates a giant WAL record and bloats the table. Two better strategies:

1. **Partition** the table; `DETACH PARTITION` + `DROP TABLE` is O(1) (see [[01 - Sharding and Partitioning]]).
2. **Chunk** the delete: small batches with explicit transactions; vacuum periodically.

For loading data warehouses (millions to billions of rows), drop indexes/constraints during load, `COPY` into a staging table, validate, then attach or `INSERT … SELECT` into the live table.

---

## Syntax & API

### COPY from a CSV file
```sql
-- Server-side (file readable by Postgres user)
COPY users (email, name)
FROM '/var/lib/postgresql/imports/users.csv'
WITH (FORMAT CSV, HEADER true, DELIMITER ',');

-- Client-side via psql (\copy reads from local file)
\copy users (email, name) FROM 'users.csv' WITH (FORMAT CSV, HEADER true);
```

### COPY from STDIN (programmatic)
```bash
psql -c "COPY users (email, name) FROM STDIN WITH (FORMAT CSV)" < users.csv
```

### Npgsql binary COPY (fastest)
```csharp
await using var importer = await conn.BeginBinaryImportAsync(
    "COPY users (email, name) FROM STDIN (FORMAT BINARY)", ct);

foreach (var u in inputs)
{
    await importer.StartRowAsync(ct);
    await importer.WriteAsync(u.Email, NpgsqlDbType.Text, ct);
    await importer.WriteAsync(u.Name,  NpgsqlDbType.Text, ct);
}
ulong rows = await importer.CompleteAsync(ct);
```

### Multi-row INSERT (fallback when COPY isn't available)
```sql
INSERT INTO users (email, name) VALUES
    ('a@b.com', 'Alice'),
    ('c@d.com', 'Carol'),
    ('e@f.com', 'Eve');
-- A single INSERT with N rows is much faster than N separate INSERTs
-- but still slower than COPY for large N.
```

### Upsert with ON CONFLICT
```sql
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice 2.0')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name, updated_at = now();

-- Bulk upsert
INSERT INTO users (email, name)
SELECT * FROM (VALUES
    ('a@b.com', 'Alice'),
    ('c@d.com', 'Carol')
) AS v(email, name)
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

-- Skip on conflict (no update)
INSERT INTO users (email, name) VALUES ('a@b.com','x')
ON CONFLICT (email) DO NOTHING;

-- WHERE clause to skip same-value updates (avoid no-op write storms)
ON CONFLICT (email) DO UPDATE
    SET name = EXCLUDED.name
    WHERE users.name IS DISTINCT FROM EXCLUDED.name;
```

### MERGE (PG 15+)
```sql
MERGE INTO users u
USING (
    SELECT * FROM (VALUES
        ('a@b.com', 'Alice'),
        ('c@d.com', 'Carol')
    ) AS v(email, name)
) src ON src.email = u.email
WHEN MATCHED AND u.name IS DISTINCT FROM src.name
    THEN UPDATE SET name = src.name, updated_at = now()
WHEN NOT MATCHED
    THEN INSERT (email, name) VALUES (src.email, src.name);
-- MERGE supports DELETE / DO NOTHING branches too
```

### Bulk update from a VALUES list
```sql
UPDATE products p
SET price = v.new_price
FROM (VALUES
    ('SKU1', 9.99),
    ('SKU2', 19.99)
) AS v(sku, new_price)
WHERE p.sku = v.sku;
```

### Bulk update from a staging table
```sql
CREATE TEMP TABLE updates (sku TEXT PRIMARY KEY, new_price NUMERIC);
COPY updates FROM '/tmp/updates.csv' CSV HEADER;

UPDATE products p
SET price = u.new_price
FROM updates u
WHERE p.sku = u.sku;
```

### Chunked delete
```sql
DO $$
DECLARE
    n INT;
BEGIN
    LOOP
        DELETE FROM events
        WHERE id IN (
            SELECT id FROM events
            WHERE occurred_at < now() - INTERVAL '1 year'
            LIMIT 10000
        );
        GET DIAGNOSTICS n = ROW_COUNT;
        EXIT WHEN n = 0;
        COMMIT;          -- in plpgsql procedures (PG 11+)
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

### Drop indexes for big loads, recreate after
```sql
BEGIN;
ALTER TABLE big_table DISABLE TRIGGER USER;     -- skip triggers
DROP INDEX idx_big_table_x;
DROP INDEX idx_big_table_y;
COPY big_table FROM '/tmp/big.csv' CSV;
CREATE INDEX CONCURRENTLY idx_big_table_x ON big_table (x);
CREATE INDEX CONCURRENTLY idx_big_table_y ON big_table (y);
ALTER TABLE big_table ENABLE TRIGGER USER;
COMMIT;
-- (CONCURRENTLY can't run inside tx — manage in steps)
```

### UNLOGGED tables for staging
```sql
CREATE UNLOGGED TABLE staging_events (LIKE events INCLUDING DEFAULTS);
COPY staging_events FROM '/tmp/events.csv' CSV;
-- UNLOGGED skips WAL → much faster, but not crash-safe
INSERT INTO events SELECT * FROM staging_events;
DROP TABLE staging_events;
```

---

## Common Patterns

```sql
-- Pattern: idempotent bulk import (re-runnable)
BEGIN;
CREATE TEMP TABLE staging (LIKE products);
\copy staging FROM 'products.csv' CSV HEADER;

INSERT INTO products
SELECT * FROM staging
ON CONFLICT (sku) DO UPDATE
    SET name = EXCLUDED.name,
        price = EXCLUDED.price,
        updated_at = now()
    WHERE products.* IS DISTINCT FROM EXCLUDED.*;
COMMIT;
```

```sql
-- Pattern: COPY into partitioned table — direct insert into specific partition
-- Postgres routes COPY based on partition key, but COPY direct to a leaf is fastest
COPY events_2026_q2 FROM '/tmp/q2.csv' CSV;     -- skip routing
```

```sql
-- Pattern: parallel COPY with multiple workers
-- Split file by line count
split -l 1000000 huge.csv huge_part_

-- Run multiple psql sessions in parallel
parallel -j 4 'psql -d shop -c "\copy events FROM {} CSV HEADER"' ::: huge_part_*
```

---

## Gotchas & Tips

- **`COPY` parses much less than INSERT** — types are determined by the column, no per-row planning.
- **Binary format > text** for repeated loads — but binary is type-strict and version-bound.
- **`COPY` runs as a single transaction** — a single bad row fails the whole load. Split into chunks if needed.
- **Disable triggers and constraints during pre-validated bulk loads** — re-enable + validate after. Saves 50%+.
- **Drop secondary indexes** for big loads → recreate `CONCURRENTLY` after. Way faster.
- **`UPDATE` rewrites rows** — full row is duplicated (MVCC), index entries updated. Bulk updates can bloat heavily; vacuum after.
- **`ON CONFLICT DO UPDATE` writes even on no-op match** — add `WHERE old IS DISTINCT FROM new` to skip identical updates.
- **`MERGE` (PG 15) doesn't dedupe source rows** — duplicates in source cause "command cannot affect row a second time" errors. Pre-dedupe.
- **Chunked deletes need vacuum** — long chains of dead rows slow scans. `VACUUM` after each chunk or rely on autovacuum.
- **`UNLOGGED` is fast but not durable** — restart truncates them. Great for staging; never for primary data.
- **Network is often the bottleneck** for client-side `\copy` — use server-side `COPY` with files local to PG when possible.
- **For huge loads, partition first** — load into the right partition; downstream operations stay scoped.

---

## See Also

- [[14 - ADO.NET and Dapper]]
- [[01 - Sharding and Partitioning]]
- [[13 - ETL and CDC]]
- [[09 - Performance Tuning]]
