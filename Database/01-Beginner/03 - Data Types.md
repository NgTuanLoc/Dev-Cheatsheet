---
tags: [database, beginner, postgresql, schema-design]
aliases: [PostgreSQL Types, Column Types]
level: Beginner
---

# Data Types

> **One-liner**: Pick the narrowest type that fits your data — narrower types use less storage, less memory, and let indexes work harder.

---

## Quick Reference

| Category | PostgreSQL type | Use for |
|----------|-----------------|---------|
| Integer | `SMALLINT` (2B), `INTEGER` / `INT` (4B), `BIGINT` (8B) | counts, IDs, ages |
| Auto-increment | `SERIAL`, `BIGSERIAL` (legacy) or `GENERATED ALWAYS AS IDENTITY` (preferred) | surrogate PKs |
| Decimal | `NUMERIC(p, s)` / `DECIMAL` | money, exact decimals |
| Float | `REAL` (4B), `DOUBLE PRECISION` (8B) | scientific, NOT money |
| Text | `TEXT` (preferred), `VARCHAR(n)`, `CHAR(n)` | strings |
| Boolean | `BOOLEAN` | true/false/null |
| Date/time | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` | timestamps |
| UUID | `UUID` | distributed/non-sequential IDs |
| JSON | `JSONB` (preferred), `JSON` | semi-structured data |
| Binary | `BYTEA` | raw bytes |
| Array | `TYPE[]` (e.g. `INT[]`) | small arrays |
| Enum | `CREATE TYPE … AS ENUM (…)` | fixed value sets |
| Network | `INET`, `CIDR`, `MACADDR` | IPs, MAC addresses |
| Geometric | `POINT`, `POLYGON`, … (PostGIS for real GIS) | coordinates |

---

## Core Concept

PostgreSQL has rich, **strict** types: assigning `'abc'` to an `INTEGER` column fails. This is a feature — bad data never sneaks in.

Three rules of thumb:

1. **Narrow over wide** — `INT` over `BIGINT` unless you'll exceed ~2 billion rows. `SMALLINT` for ages or quantities.
2. **Exact over approximate for money** — `NUMERIC(18, 2)`, never `FLOAT` or `DOUBLE`. Floating-point arithmetic introduces rounding errors (`0.1 + 0.2 ≠ 0.3` in binary float).
3. **`TIMESTAMPTZ` over `TIMESTAMP`** — `TIMESTAMPTZ` stores UTC and converts on read. `TIMESTAMP` stores wall-clock with no timezone — surprising on DST or cross-region apps.

For text, use `TEXT` everywhere unless you have a real reason for a length limit. In Postgres, `TEXT`, `VARCHAR(n)`, and `VARCHAR` perform identically — there's no storage win for short types.

---

## Syntax & API

### Common column types
```sql
CREATE TABLE products (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku          TEXT NOT NULL UNIQUE,
    name         TEXT NOT NULL,
    description  TEXT,
    price        NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock        INTEGER NOT NULL DEFAULT 0,
    is_active    BOOLEAN NOT NULL DEFAULT true,
    tags         TEXT[],
    metadata     JSONB,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ
);
```

### UUID primary key
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- for gen_random_uuid()

CREATE TABLE orders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL,
    placed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Date / time
```sql
SELECT
    now()                          AS current_ts,    -- with timezone
    CURRENT_DATE                   AS today,
    now() - INTERVAL '7 days'      AS week_ago,
    now() AT TIME ZONE 'UTC'       AS in_utc,
    EXTRACT(YEAR FROM now())       AS year_part;
```

### Enums
```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

CREATE TABLE orders (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status  order_status NOT NULL DEFAULT 'pending'
);

-- Add a new value (must be last by default)
ALTER TYPE order_status ADD VALUE 'refunded';
```

### Arrays
```sql
INSERT INTO products (sku, name, tags) VALUES
    ('SKU1', 'Coffee', ARRAY['drink', 'hot', 'caffeinated']);

-- Query
SELECT * FROM products WHERE 'hot' = ANY(tags);
SELECT * FROM products WHERE tags && ARRAY['drink'];   -- overlap
```

### JSONB
```sql
INSERT INTO products (sku, name, metadata) VALUES
    ('SKU2', 'Laptop', '{"brand":"Dell","ram":16}');

SELECT name, metadata->>'brand' AS brand
FROM products
WHERE metadata->>'brand' = 'Dell'
  AND (metadata->>'ram')::INT >= 8;
```

---

## Common Patterns

```sql
-- Pattern: surrogate vs natural primary key
-- Prefer IDENTITY over SERIAL (modern Postgres)
CREATE TABLE users (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email  TEXT NOT NULL UNIQUE     -- natural key as a UNIQUE
);
```

```sql
-- Pattern: money with 4 fractional digits for currencies that need them
price NUMERIC(18, 4)  -- enough for crypto / FX
```

```sql
-- Pattern: timezone-aware audit timestamps
created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at  TIMESTAMPTZ
```

---

## Gotchas & Tips

- **Never use `FLOAT`/`DOUBLE` for money** — round-trip rounding errors. Use `NUMERIC`.
- **`TIMESTAMP WITHOUT TIME ZONE` is a trap** — it stores no zone info. Use `TIMESTAMPTZ` (UTC under the hood, converted on display) almost always.
- **`SERIAL` is legacy** — use `GENERATED ALWAYS AS IDENTITY`. SERIAL has subtle issues with permissions and `pg_upgrade`.
- **`CHAR(n)` pads with spaces** — almost never what you want. Use `TEXT`.
- **`VARCHAR(n)` is not faster than `TEXT`** in Postgres — the length check is the only difference. Use `TEXT` + a `CHECK` constraint if you need a max length.
- **Timezone of the SERVER is irrelevant for `TIMESTAMPTZ`** — it's stored in UTC. Display conversion uses the client's `timezone` setting.
- **Enums are sticky** — adding values is easy, removing is hard (you have to recreate the type). Consider a lookup table if values churn.
- **Arrays vs separate table** — arrays are fine for short, unbounded-but-small lists (tags, days-of-week). For relational data (users → orders), use a separate table.
- **`JSONB` indexes with GIN** — efficient for `?`, `@>`, `->>` filters. Plain `JSON` is text-only and slower.

---

## See Also

- [[04 - Schema Design Basics]]
- [[09 - Constraints]]
- [[18 - JSON and JSONB]]
- [[03 - Isolation Levels]]
