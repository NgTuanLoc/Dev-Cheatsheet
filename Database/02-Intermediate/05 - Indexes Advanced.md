---
tags: [database, intermediate, indexing, postgresql, performance]
aliases: [GIN, GiST, BRIN, Partial Index, Covering Index]
level: Intermediate
---

# Indexes Advanced

> **One-liner**: Beyond the default B-tree, Postgres has specialized index types for arrays, JSON, full-text, geometry, and huge time-series — pick the one that matches your query.

---

## Quick Reference

| Index type | Best for |
|------------|----------|
| **B-tree** (default) | equality, ranges, ORDER BY, prefix `LIKE 'abc%'` |
| **GIN** | arrays (`@>`), JSONB (`?`, `@>`), full-text (`tsvector`), trigram |
| **GiST** | ranges (`tsrange`), geometry, exclusion constraints |
| **BRIN** | huge tables where rows are stored in roughly the same order as the indexed value (timestamps, append-only logs) |
| **Hash** | strict equality only — rarely used; B-tree is usually fine |

| Variant | Use |
|---------|-----|
| **Composite** | multiple columns; left-prefix rule |
| **Covering** (`INCLUDE`) | extra columns in leaves → index-only scan |
| **Partial** | `WHERE` filter on the index → smaller, faster |
| **Expression** | index on a function (`LOWER(email)`) |
| **Unique** | enforces uniqueness; can be partial too |
| **CONCURRENTLY** | build/drop without blocking writes |

---

## Core Concept

Different access patterns want different index structures:

- **B-tree** — sorted tree; fast equality, ranges, sorts. Fits 95% of cases.
- **GIN** (Generalized Inverted Index) — like a search index: term → list of rows. Perfect for "does this row's array/JSONB/tsvector contain X?"
- **GiST** (Generalized Search Tree) — flexible spatial-ish index. Used for ranges (`tsrange`), geometry (PostGIS), and exclusion constraints.
- **BRIN** (Block Range INdex) — tiny index storing min/max per range of pages. Useless for random access, perfect for billion-row append-only tables where the timestamp is naturally ordered.

**Composite** indexes follow the **left-prefix rule**: an index on `(a, b, c)` helps queries filtering by `a`, `(a, b)`, or `(a, b, c)` — but not by `b` alone or `c` alone.

A **covering index** stores extra columns in the leaves so a query can be answered without touching the heap (an "index-only scan"). Massive speedup for hot read paths.

A **partial index** only covers rows matching a `WHERE` — tiny and selective when most rows are uninteresting (e.g., only "pending" jobs).

---

## Syntax & API

### B-tree composite + ORDER BY
```sql
-- Query: WHERE user_id = ? ORDER BY placed_at DESC LIMIT 10
CREATE INDEX idx_orders_user_placed
    ON orders (user_id, placed_at DESC);
-- Order direction in the index matches the ORDER BY → no sort step
```

### Covering index (Postgres 11+)
```sql
CREATE INDEX idx_orders_user_cover
    ON orders (user_id) INCLUDE (total, placed_at);

-- Query that only reads user_id, total, placed_at can skip the heap
SELECT total, placed_at FROM orders WHERE user_id = 5;
-- EXPLAIN shows "Index Only Scan"
```

### Partial index
```sql
-- 99% of jobs are completed; only 'pending' is queried often
CREATE INDEX idx_jobs_pending
    ON jobs (created_at)
    WHERE status = 'pending';

-- Used by:
SELECT * FROM jobs WHERE status = 'pending' ORDER BY created_at LIMIT 1;
-- Index is tiny because it only contains pending rows
```

### Expression index
```sql
-- Case-insensitive lookups
CREATE INDEX idx_users_email_lower
    ON users (LOWER(email));

-- MUST use the same expression in the query
SELECT * FROM users WHERE LOWER(email) = LOWER('A@B.com');
```

### GIN on JSONB
```sql
CREATE INDEX idx_products_metadata_gin
    ON products USING gin (metadata);

-- Helps `?`, `@>`, `?|`, `?&` operators
SELECT * FROM products WHERE metadata @> '{"brand":"Dell"}';
SELECT * FROM products WHERE metadata ? 'gpu';

-- More efficient for one-key lookups: jsonb_path_ops
CREATE INDEX idx_products_metadata_path
    ON products USING gin (metadata jsonb_path_ops);
-- smaller, faster for `@>` only
```

### GIN for full-text search
```sql
ALTER TABLE articles ADD COLUMN search_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;

CREATE INDEX idx_articles_search ON articles USING gin (search_tsv);

SELECT id, ts_rank(search_tsv, q) AS rank
FROM articles, plainto_tsquery('english', 'postgres index') q
WHERE search_tsv @@ q
ORDER BY rank DESC LIMIT 20;
```

### GIN with pg_trgm for substring search
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_users_name_trgm
    ON users USING gin (name gin_trgm_ops);

-- Now leading-wildcard LIKE can use an index
SELECT * FROM users WHERE name ILIKE '%alic%';
```

### GiST for ranges and exclusion
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE reservations (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INT NOT NULL,
    period TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, period WITH &&)   -- GiST + exclusion
);
```

### BRIN for huge time-series
```sql
-- 1B rows, append-only events ordered by occurred_at
CREATE INDEX idx_events_occurred_brin
    ON events USING brin (occurred_at);

-- Storage cost: ~kilobytes (vs gigabytes for B-tree)
-- Queries: SELECT * FROM events WHERE occurred_at > now() - INTERVAL '1 day';
```

### Concurrent build / drop
```sql
CREATE INDEX CONCURRENTLY idx_orders_user ON orders (user_id);
DROP INDEX CONCURRENTLY idx_orders_user_old;
-- Slower, but no ACCESS EXCLUSIVE on the table — writes keep flowing
```

### Inspect usage
```sql
-- Index sizes
SELECT relname, pg_size_pretty(pg_relation_size(oid))
FROM pg_class WHERE relkind = 'i' ORDER BY pg_relation_size(oid) DESC;

-- Unused indexes (good drop candidates)
SELECT relname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

---

## Common Patterns

```sql
-- Pattern: indexed soft-delete view
CREATE INDEX idx_users_active ON users (id) WHERE deleted_at IS NULL;
-- All queries filter on deleted_at; index only contains live rows
```

```sql
-- Pattern: case-insensitive UNIQUE
CREATE UNIQUE INDEX uq_users_email_lower ON users (LOWER(email));
-- 'A@B.com' and 'a@b.com' collide
```

```sql
-- Pattern: index-only scan for top-N read paths
CREATE INDEX idx_orders_status_date_cover
    ON orders (status, placed_at DESC)
    INCLUDE (id, total)
    WHERE status IN ('pending','paid');
-- Combines partial + covering + composite
```

---

## Gotchas & Tips

- **GIN inserts/updates are slower** — they update many index entries per row. Mostly-read tables are fine; write-heavy ones less so.
- **`gin_trgm_ops` lets `LIKE '%foo%'` use an index** — no other index type supports leading wildcards.
- **BRIN is useless if data is shuffled** — the assumption is on-disk order ≈ value order. Holds for `CREATE TABLE … PARTITION BY RANGE` on time, breaks if you reorganize.
- **Covering only helps if the query reads only those columns** — `SELECT *` defeats it.
- **Partial index conditions must match the WHERE in queries** — `WHERE status='pending'` index won't be used for `WHERE status IN ('pending','paid')`.
- **Expression must match exactly** — `LOWER(email)` and `lower(email)` are the same; `email ILIKE 'X'` won't hit a `LOWER(email)` index.
- **Multiple single-column indexes can be combined** — Postgres can do BitmapAnd / BitmapOr to merge results. Often a composite is better, but not always.
- **Always `CREATE INDEX CONCURRENTLY` in prod** — non-concurrent takes an `ACCESS EXCLUSIVE` lock that blocks all writes.
- **Re-build to fight bloat** — `REINDEX INDEX CONCURRENTLY idx_name;` after large delete/update churn.
- **Keep `pg_stat_user_indexes` open** — drop indexes with `idx_scan = 0` after a representative production window.

---

## See Also

- [[10 - Indexes Basics]]
- [[06 - Query Optimization]]
- [[18 - JSON and JSONB]]
- [[19 - Full-Text Search]]
