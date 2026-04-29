---
tags: [database, intermediate, postgresql, schema-design]
aliases: [JSONB, JSON Path, jsonb_path_query]
level: Intermediate
---

# JSON and JSONB

> **One-liner**: `JSONB` is binary, indexable, fast-to-query JSON storage in Postgres — perfect for sparse/optional/evolving fields, terrible for relational data dressed up as documents.

---

## Quick Reference

| Type | Storage | Use |
|------|---------|-----|
| `JSON` | text | preserves whitespace + key order; slower to query |
| `JSONB` | binary tree | indexable, dedupes keys, fast — almost always preferred |

| Operator | Meaning | Example |
|----------|---------|---------|
| `->` | get child as JSON | `data->'addr'` |
| `->>` | get child as text | `data->>'name'` |
| `#>` | path → JSON | `data #> '{addr,city}'` |
| `#>>` | path → text | `data #>> '{addr,city}'` |
| `?` | does key exist? | `data ? 'name'` |
| `?|` | any of these keys exists? | `data ?| ARRAY['a','b']` |
| `?&` | all of these keys exist? | `data ?& ARRAY['a','b']` |
| `@>` | left contains right? | `data @> '{"a":1}'` |
| `<@` | right contains left? | `'{"a":1}' <@ data` |
| `@?` | jsonpath returns any? (PG 12+) | `data @? '$.tags[*] ? (@ == "x")'` |
| `@@` | jsonpath as predicate | `data @@ '$.age > 18'` |

| Function | Use |
|----------|-----|
| `jsonb_array_elements(j)` | rows from array |
| `jsonb_each(j)` / `jsonb_each_text(j)` | rows of (key, value) |
| `jsonb_object_agg(k, v)` | aggregate to object |
| `jsonb_set(j, path, value)` | update path (returns new JSONB) |
| `jsonb_path_query(j, path)` | jsonpath query |
| `to_jsonb(row)` / `row_to_json(row)` | row → JSON |

---

## Core Concept

`JSONB` stores parsed, deduplicated, sorted-by-key JSON in a binary tree. Reads and queries are fast; storage is slightly larger than text JSON. Use `JSONB` unless you have an explicit need to preserve formatting/order (almost never).

When to use JSONB:

- **Sparse / optional attributes** — products with vendor-specific specs, events with arbitrary payloads
- **External / passthrough payloads** — webhook bodies, audit logs, raw API responses
- **Evolving schemas before you commit** — prototype shape with JSONB, normalize later

When **not** to use JSONB:

- Data that has a clear relational shape (users, orders) — model it as columns, not nested JSON
- Anything you'd want to JOIN frequently
- Anything where you'd add indexes for many keys (Postgres can do it; relational still cleaner)

A **GIN index** makes `@>` containment and `?` key-existence queries fast. Use `jsonb_path_ops` for smaller, faster `@>` -only indexes.

---

## Syntax & API

### Setup
```sql
CREATE TABLE products (
    id        INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku       TEXT NOT NULL UNIQUE,
    name      TEXT NOT NULL,
    metadata  JSONB
);

INSERT INTO products (sku, name, metadata) VALUES
    ('LP-1', 'Laptop',
     '{"brand":"Dell","specs":{"ram":16,"ssd":512},"tags":["work","portable"]}'),
    ('PH-1', 'Phone',
     '{"brand":"Pixel","specs":{"ram":8},"tags":["mobile"]}');
```

### Read fields
```sql
SELECT sku,
       metadata->>'brand'                  AS brand,           -- text
       metadata->'specs'                   AS specs_json,      -- jsonb
       (metadata#>>'{specs,ram}')::INT     AS ram_gb,
       metadata->'tags'                    AS tags
FROM products;
```

### Filter by JSON
```sql
-- Containment (whole subtree match)
SELECT * FROM products WHERE metadata @> '{"brand":"Dell"}';

-- Key existence
SELECT * FROM products WHERE metadata ? 'tags';

-- Numeric comparison via cast
SELECT * FROM products
WHERE (metadata->'specs'->>'ram')::INT >= 16;

-- Array element
SELECT * FROM products
WHERE metadata->'tags' ? 'work';                 -- 'work' is in the tags array
```

### Index it
```sql
-- General GIN — supports @>, ?, ?|, ?&
CREATE INDEX idx_products_meta_gin
    ON products USING gin (metadata);

-- Smaller, faster, @> only
CREATE INDEX idx_products_meta_path
    ON products USING gin (metadata jsonb_path_ops);

-- Indexed expression for a specific scalar path
CREATE INDEX idx_products_brand
    ON products ((metadata->>'brand'));

-- Now queries on metadata->>'brand' use a B-tree index
```

### Update — `jsonb_set`
```sql
-- Add or change a deep path
UPDATE products
SET metadata = jsonb_set(metadata, '{specs,gpu}', '"RTX 4080"', true)
WHERE sku = 'LP-1';

-- Concatenate (merge) two JSONB
UPDATE products
SET metadata = metadata || '{"warranty_years":2}'::jsonb
WHERE sku = 'LP-1';

-- Remove a key
UPDATE products
SET metadata = metadata - 'tags'
WHERE sku = 'LP-1';

-- Remove a deep path
UPDATE products
SET metadata = metadata #- '{specs,ssd}'
WHERE sku = 'LP-1';
```

### Aggregate / build JSON from rows
```sql
-- Object: { sku → name }
SELECT jsonb_object_agg(sku, name) FROM products;

-- Array of objects
SELECT jsonb_agg(jsonb_build_object('sku', sku, 'name', name))
FROM products;

-- Whole row → JSON
SELECT to_jsonb(p) FROM products p WHERE id = 1;
```

### Expand JSON → rows
```sql
-- Each tag as its own row
SELECT p.sku, t.tag
FROM products p,
     jsonb_array_elements_text(p.metadata->'tags') AS t(tag);

-- Each (key, value) pair
SELECT p.sku, kv.key, kv.value
FROM products p,
     jsonb_each(p.metadata) AS kv;
```

### jsonpath (PG 12+)
```sql
-- Strict jsonpath
SELECT sku
FROM products
WHERE metadata @? '$.specs.ram ? (@ >= 16)';

-- Extract a value via path
SELECT jsonb_path_query_first(metadata, '$.specs.ram') FROM products;
```

---

## Common Patterns

```sql
-- Pattern: log table for arbitrary event payloads
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_type  TEXT NOT NULL,
    payload     JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);

-- Find login events with a specific user_agent fragment
SELECT * FROM events
WHERE event_type = 'login'
  AND payload @> '{"ua":"Firefox"}';
```

```sql
-- Pattern: validate JSONB shape with a CHECK
ALTER TABLE products ADD CONSTRAINT ck_meta_brand_text
    CHECK (jsonb_typeof(metadata->'brand') = 'string');
```

```sql
-- Pattern: indexed JSONB for hot keys + relational columns for important fields
-- Don't over-do JSONB. If you query brand often, lift it to a real column.
ALTER TABLE products ADD COLUMN brand TEXT
    GENERATED ALWAYS AS (metadata->>'brand') STORED;
CREATE INDEX idx_products_brand ON products(brand);
```

---

## Gotchas & Tips

- **Use `JSONB`, not `JSON`** — virtually always. Plain `JSON` is for "store this exact text" rare cases.
- **Indexable, but not free** — GIN indexes are slow to update. Heavy-write tables hurt.
- **`->` returns JSON, `->>` returns text** — confusing one for the other gives bad implicit casts.
- **Casting numeric inside JSON** — `(j->>'x')::INT` is fine; just remember to cast.
- **`jsonb_set(j, path, val, create_missing)`** — `create_missing` defaults to true; set false to update only existing keys.
- **`||` for merge is shallow** — nested objects are replaced, not deep-merged. Postgres has no built-in deep merge; write a function or use `jsonb_set` repeatedly.
- **Sorting on JSONB is order-insensitive at the top level** — `'{"a":1,"b":2}' = '{"b":2,"a":1}'` is true.
- **NULL in JSON ≠ SQL NULL** — `'null'::jsonb` is a JSON null value (`jsonb_typeof = 'null'`). `j IS NULL` checks SQL null.
- **Don't put primary keys in JSONB** — you give up FK enforcement and indexability.
- **Schema-on-read costs add up** — every query that does `j->>'k'` pays parse cost. For very hot queries lift the field to a generated column.
- **Validate input** — Postgres won't reject ill-formed JSON (it's already JSON), but it will accept any shape. Validate in app or with `CHECK` constraints.

---

## See Also

- [[03 - Data Types]]
- [[05 - Indexes Advanced]]
- [[19 - Full-Text Search]]
- [[20 - NoSQL Fundamentals]]
