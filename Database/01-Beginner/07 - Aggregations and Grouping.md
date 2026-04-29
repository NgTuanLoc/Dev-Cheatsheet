---
tags: [database, beginner, sql, postgresql]
aliases: [GROUP BY, Aggregate Functions, HAVING]
level: Beginner
---

# Aggregations and Grouping

> **One-liner**: Aggregations collapse many rows into summary values; `GROUP BY` does it per category, `HAVING` filters those groups.

---

## Quick Reference

| Function | What it does |
|----------|--------------|
| `COUNT(*)` | Number of rows |
| `COUNT(col)` | Number of non-NULL values |
| `COUNT(DISTINCT col)` | Distinct non-NULL values |
| `SUM(col)` | Total |
| `AVG(col)` | Average (returns NUMERIC) |
| `MIN(col)` / `MAX(col)` | Extremes |
| `STRING_AGG(col, sep)` | Concatenate with separator |
| `ARRAY_AGG(col)` | Collect into an array |
| `BOOL_OR / BOOL_AND` | Aggregate booleans |
| `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col)` | Median (and other percentiles) |

| Clause | Purpose |
|--------|---------|
| `GROUP BY col1, col2` | Collapse rows sharing the same `(col1, col2)` |
| `HAVING …` | Filter groups (after aggregation) |
| `WHERE …` | Filter rows (before aggregation) |

---

## Core Concept

An **aggregate function** takes a set of rows and returns a single value. Without `GROUP BY`, an aggregate collapses the entire result to one row.

`GROUP BY` partitions the rows by the listed columns; the aggregate runs once per partition. Every column in `SELECT` must be either:
1. listed in `GROUP BY`, or
2. inside an aggregate function

`WHERE` filters rows **before** grouping; `HAVING` filters groups **after** grouping. Aggregate functions can appear in `HAVING` but not in `WHERE`:

```
WHERE total > 100      -- per-row predicate
HAVING SUM(total) > 1000  -- per-group predicate
```

`NULL` values are skipped by aggregates (except `COUNT(*)`). `AVG(salary)` divides only by non-NULL salaries.

---

## Syntax & API

### One number for the whole table
```sql
SELECT COUNT(*) AS row_count, AVG(total) AS avg_order, MAX(total) AS biggest
FROM orders;
```

### Group by a single column
```sql
SELECT user_id, COUNT(*) AS orders, SUM(total) AS revenue
FROM orders
GROUP BY user_id
ORDER BY revenue DESC;
```

### Group by multiple columns
```sql
SELECT
    DATE_TRUNC('month', placed_at)::DATE AS month,
    country,
    COUNT(*) AS orders,
    SUM(total) AS revenue
FROM orders
JOIN users USING (user_id)            -- if needed
GROUP BY month, country
ORDER BY month, revenue DESC;
```

### `HAVING` for group-level filters
```sql
-- Users who placed more than 5 orders
SELECT user_id, COUNT(*) AS n
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;
```

### `COUNT(DISTINCT …)`
```sql
-- How many distinct customers ordered today?
SELECT COUNT(DISTINCT user_id) AS unique_customers
FROM orders
WHERE placed_at::DATE = CURRENT_DATE;
```

### `STRING_AGG` and `ARRAY_AGG`
```sql
SELECT user_id, STRING_AGG(item_name, ', ' ORDER BY item_name) AS items
FROM order_items
GROUP BY user_id;

SELECT user_id, ARRAY_AGG(DISTINCT product_id) AS products
FROM order_items
GROUP BY user_id;
```

### `FILTER` clause — aggregate a subset
```sql
SELECT
    user_id,
    COUNT(*)                                      AS total_orders,
    COUNT(*) FILTER (WHERE total > 100)          AS big_orders,
    SUM(total) FILTER (WHERE placed_at > now() - INTERVAL '30 days') AS spend_30d
FROM orders
GROUP BY user_id;
```

### Percentiles
```sql
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total) AS median,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY total) AS p95
FROM orders;
```

---

## Common Patterns

```sql
-- Pattern: orders per user, including users with zero orders
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

```sql
-- Pattern: top N per group (window function — see 10 - Window Functions)
SELECT user_id, id, total
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rk
    FROM orders
) t
WHERE rk <= 3;
```

```sql
-- Pattern: rollup / subtotals
SELECT
    COALESCE(country, 'TOTAL') AS country,
    SUM(total) AS revenue
FROM orders JOIN users USING (user_id)
GROUP BY ROLLUP (country);
```

---

## Gotchas & Tips

- **Every non-aggregated `SELECT` column must be in `GROUP BY`** — Postgres enforces this strictly. (MySQL historically didn't, leading to surprising results.)
- **`COUNT(*)` vs `COUNT(col)`** — `COUNT(*)` includes rows with NULLs; `COUNT(col)` skips NULLs in that column. Different answers when the column is nullable.
- **`AVG` ignores NULLs** — if you want NULLs counted as zero, `AVG(COALESCE(col, 0))`.
- **`HAVING` without `GROUP BY` works** — it treats the whole table as one group. Rare but valid.
- **`GROUP BY` aliases** — Postgres lets you `GROUP BY 1, 2` (column positions) or `GROUP BY my_alias`. Convenient but harder to refactor.
- **`SELECT DISTINCT` vs `GROUP BY`** — equivalent for plain deduplication. `GROUP BY` is needed when you also aggregate.
- **Beware `COUNT(*) > 0` patterns** — `EXISTS` is usually faster because it short-circuits.
- **Aggregating after a join can double-count** — joining one parent to two child tables multiplies counts. Aggregate one side first in a subquery.

---

## See Also

- [[02 - SQL Fundamentals]]
- [[06 - Joins]]
- [[08 - Subqueries and CTEs]]
- [[10 - Window Functions]]
