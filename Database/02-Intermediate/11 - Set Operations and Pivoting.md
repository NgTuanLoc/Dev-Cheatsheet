---
tags: [database, intermediate, sql, postgresql]
aliases: [UNION, INTERSECT, EXCEPT, PIVOT, Crosstab]
level: Intermediate
---

# Set Operations and Pivoting

> **One-liner**: Set operations combine rows from multiple queries (`UNION`, `INTERSECT`, `EXCEPT`); pivoting reshapes rows into columns and back.

---

## Quick Reference

| Operator | Meaning | Notes |
|----------|---------|-------|
| `UNION` | rows from A ∪ B, deduplicated | adds a sort/hash for dedup → slower |
| `UNION ALL` | rows from A and B, with duplicates | the fast one — use unless you need dedup |
| `INTERSECT` | rows in both A and B | dedup'd by default; `INTERSECT ALL` keeps multiplicity |
| `EXCEPT` | rows in A not in B (set difference) | `EXCEPT ALL` keeps multiplicity |
| `crosstab(...)` | pivot rows → columns (extension `tablefunc`) | for fixed column lists |
| `FILTER (WHERE …)` aggregate | conditional aggregate per column | the modern pivot trick |

| Rule for set operations | |
|--------------------------|---|
| All branches must have same **number of columns** | error otherwise |
| Columns matched by **position**, not name | first column in A == first column in B |
| Resulting column types must be compatible | implicit cast or explicit CAST |

---

## Core Concept

Set operations stack queries vertically: each branch produces rows with the same shape, and the operator decides how to combine them.

`UNION` is the most common but the slowest — it deduplicates the entire result. `UNION ALL` is the right default unless you actually need dedup.

`INTERSECT` and `EXCEPT` are less common. Most "rows in A but not B" situations are clearer with `LEFT JOIN … WHERE B IS NULL` or `NOT EXISTS`, so the planner can choose better.

**Pivoting** transforms long-format data (one row per fact) into wide-format (one column per category). Postgres has two ways:

- The **modern idiom** — `SUM(x) FILTER (WHERE col = 'A')` — works for any aggregate, no extension needed.
- The **classic** `crosstab` function (extension `tablefunc`) — built for "true" pivot, but rigid and awkward.

For unpivoting (wide → long) use `UNION ALL` or `unnest` over an array of values.

---

## Syntax & API

### UNION ALL (fastest)
```sql
SELECT id, email FROM customers
UNION ALL
SELECT id, email FROM partners;
-- Mixes rows; doesn't dedup
```

### UNION (deduplicates)
```sql
SELECT email FROM customers
UNION
SELECT email FROM partners;
-- Same email from both shows up once
```

### INTERSECT — common rows
```sql
SELECT email FROM customers
INTERSECT
SELECT email FROM partners;
-- Emails present in both lists
```

### EXCEPT — set difference
```sql
SELECT email FROM customers
EXCEPT
SELECT email FROM partners;
-- In customers but not in partners
```

### Order operators / parens
```sql
SELECT … FROM a
UNION ALL
SELECT … FROM b
ORDER BY 1
LIMIT 10;
-- ORDER BY / LIMIT applies to the whole combined set, not per branch.

(SELECT … FROM a ORDER BY 1 LIMIT 10)
UNION ALL
(SELECT … FROM b ORDER BY 1 LIMIT 10);
-- Each branch limited separately.
```

### Modern pivot via FILTER
```sql
-- Source: orders(id, country, status, total)
SELECT
    country,
    SUM(total) FILTER (WHERE status = 'paid')      AS paid,
    SUM(total) FILTER (WHERE status = 'pending')   AS pending,
    SUM(total) FILTER (WHERE status = 'cancelled') AS cancelled,
    COUNT(*)   FILTER (WHERE status = 'paid')      AS paid_count
FROM orders
GROUP BY country;
```

### Pivot with CASE (older style — works in any SQL dialect)
```sql
SELECT
    country,
    SUM(CASE WHEN status='paid' THEN total ELSE 0 END)    AS paid,
    SUM(CASE WHEN status='pending' THEN total ELSE 0 END) AS pending
FROM orders
GROUP BY country;
```

### crosstab (Postgres tablefunc extension)
```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Source row format: (row_key, category, value)
-- Output: one row per row_key, columns per category
SELECT * FROM crosstab(
    $$SELECT country, status, SUM(total)
      FROM orders
      GROUP BY country, status
      ORDER BY country, status$$,
    $$VALUES ('paid'),('pending'),('cancelled')$$
) AS ct (country TEXT, paid NUMERIC, pending NUMERIC, cancelled NUMERIC);
```

### Unpivot (wide → long) with VALUES
```sql
-- Source: monthly(item, jan, feb, mar)
SELECT item, month, value
FROM monthly,
LATERAL (VALUES ('jan', jan), ('feb', feb), ('mar', mar)) v(month, value);
```

### Unpivot via unnest of an array
```sql
SELECT id, unnest(ARRAY[col1, col2, col3]) AS val FROM t;
```

---

## Common Patterns

```sql
-- Pattern: dashboard "by category, this period vs last"
SELECT
    category,
    SUM(amount) FILTER (WHERE month = '2026-04')        AS this_month,
    SUM(amount) FILTER (WHERE month = '2026-03')        AS last_month
FROM revenue
GROUP BY category;
```

```sql
-- Pattern: combine results from several similar tables (sharded → merged)
SELECT * FROM events_2024
UNION ALL
SELECT * FROM events_2025
UNION ALL
SELECT * FROM events_2026;
-- Or: declarative partitioning (PG 10+) — see [[01 - Sharding and Partitioning]]
```

```sql
-- Pattern: "rows in primary not in replica" diff
SELECT id FROM primary.users
EXCEPT
SELECT id FROM replica.users;
```

---

## Gotchas & Tips

- **`UNION` deduplicates by default — slow** for big results. Always ask: do I need dedup? If not, `UNION ALL`.
- **Columns matched by position** — `SELECT a,b UNION SELECT b,a` mixes columns silently. Use the same explicit column list in every branch.
- **Compatible types required** — `SELECT 'foo' UNION SELECT 1` fails. Cast explicitly.
- **`ORDER BY` and `LIMIT` apply to the whole** unless you parenthesize each branch.
- **`NULL`s are equal in set ops** — unlike `=` semantics. `INTERSECT` will treat two `NULL`s as a match.
- **`FILTER` is preferred over `CASE` for conditional aggregates** — clearer, sometimes faster.
- **Pivot column lists are static** — for dynamic pivot, build SQL in the app or use JSONB aggregation:
  ```sql
  SELECT country, jsonb_object_agg(status, total) FROM ... GROUP BY country;
  ```
- **`crosstab` is rigid** — column types and order must match exactly. The `FILTER` idiom is friendlier.
- **Heavy unpivot? Consider not storing wide.** If you frequently unpivot, the table is probably modeled wrong.

---

## See Also

- [[07 - Aggregations and Grouping]]
- [[10 - Window Functions]]
- [[18 - JSON and JSONB]]
- [[01 - Sharding and Partitioning]]
