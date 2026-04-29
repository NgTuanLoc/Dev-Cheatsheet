---
tags: [database, intermediate, sql, postgresql]
aliases: [View, Materialized View, Updatable View]
level: Intermediate
---

# Views

> **One-liner**: A view is a saved query that looks like a table; a materialized view is a saved *result* — fast to read, refreshed on demand.

---

## Quick Reference

| Type | Storage | Read cost | Refresh |
|------|---------|-----------|---------|
| **View** | Just a saved query | Re-computed every read | N/A |
| **Materialized view** | Snapshot of result | One physical scan | `REFRESH MATERIALIZED VIEW` |
| **Updatable view** | Plain view that allows INSERT/UPDATE/DELETE | passthrough to base | N/A |

| Statement | Effect |
|-----------|--------|
| `CREATE VIEW v AS SELECT …` | regular view |
| `CREATE OR REPLACE VIEW v AS …` | redefine (must keep column shape) |
| `CREATE MATERIALIZED VIEW mv AS SELECT … [WITH NO DATA]` | materialized; `WITH NO DATA` defers loading |
| `REFRESH MATERIALIZED VIEW mv` | re-run, blocks reads |
| `REFRESH MATERIALIZED VIEW CONCURRENTLY mv` | refresh without blocking reads (needs unique index) |
| `DROP VIEW v` / `DROP MATERIALIZED VIEW mv` | remove |

---

## Core Concept

A **view** is a query saved by name. Selecting from a view is identical to running its underlying query — there's no caching. Use views to:

- Hide complexity (a 5-table join becomes one name)
- Provide a stable API while the underlying schema evolves
- Restrict access to specific columns/rows (combined with permissions or RLS)

A **materialized view** stores the result. Reads are instant scans of pre-computed data. Writes happen via explicit `REFRESH`. Use when:

- The query is expensive (analytics, aggregates over millions of rows)
- The data can be slightly stale (refresh nightly, hourly, or on-demand)

`REFRESH … CONCURRENTLY` requires a unique index on the materialized view and avoids locking out readers. It's the production-friendly default.

---

## Syntax & API

### Plain view
```sql
CREATE VIEW user_revenue AS
SELECT
    u.id,
    u.email,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total), 0) AS revenue
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.email;

-- Use like a table
SELECT * FROM user_revenue WHERE revenue > 1000 ORDER BY revenue DESC;
```

### Replace (preserving columns)
```sql
CREATE OR REPLACE VIEW user_revenue AS
SELECT
    u.id,
    u.email,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total), 0) AS revenue,
    MAX(o.placed_at) AS last_order_at      -- new column → must be appended
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.email;
```

### Updatable view (Postgres auto-allows INSERT/UPDATE/DELETE if simple)
```sql
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

INSERT INTO active_users (email, name)         -- writes to users
VALUES ('x@y.com', 'X');

UPDATE active_users SET name = 'Y' WHERE id = 1;

-- Add WITH CHECK OPTION to prevent rows that escape the view
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL
WITH CHECK OPTION;
-- Now: UPDATE active_users SET deleted_at=now() FAILS (would leave the view)
```

### Materialized view
```sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
    placed_at::DATE AS day,
    COUNT(*)        AS orders,
    SUM(total)      AS revenue
FROM orders
GROUP BY placed_at::DATE;

-- Reads are fast — already computed
SELECT * FROM daily_revenue WHERE day >= CURRENT_DATE - 30;

-- Refresh — blocks reads
REFRESH MATERIALIZED VIEW daily_revenue;

-- Concurrent refresh requires a unique index
CREATE UNIQUE INDEX uq_daily_revenue_day ON daily_revenue (day);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
```

### Materialized view with deferred load
```sql
CREATE MATERIALIZED VIEW expensive_report AS
SELECT … FROM huge_join … WITH NO DATA;
-- Empty placeholder; populate later (e.g. nightly job)

REFRESH MATERIALIZED VIEW expensive_report;
```

### Index on a materialized view
```sql
CREATE INDEX idx_daily_revenue_day ON daily_revenue (day);
-- Materialized views accept indexes like ordinary tables
```

---

## Common Patterns

```sql
-- Pattern: stable API view over evolving schema
CREATE VIEW v_users_v1 AS
SELECT id, email, full_name AS name, created_at FROM users;
-- App targets v_users_v1; underlying renames don't break consumers.
```

```sql
-- Pattern: scheduled refresh via cron / pg_cron
SELECT cron.schedule('refresh-daily', '0 1 * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue$$);
```

```sql
-- Pattern: layered views — readable composition
CREATE VIEW v_active_users   AS SELECT * FROM users WHERE deleted_at IS NULL;
CREATE VIEW v_recent_orders  AS SELECT * FROM orders WHERE placed_at > now() - INTERVAL '30 days';
CREATE VIEW v_active_recent  AS
    SELECT u.*, COUNT(o.id) AS recent_orders
    FROM v_active_users u
    LEFT JOIN v_recent_orders o ON o.user_id = u.id
    GROUP BY u.id;
```

```sql
-- Pattern: row-level access via SECURITY BARRIER view
CREATE VIEW v_my_orders WITH (security_barrier=true) AS
SELECT * FROM orders WHERE user_id = current_setting('app.current_user_id')::INT;
-- Can't be tricked by leaky-where injection (qual ordering)
```

---

## Gotchas & Tips

- **Plain views are not faster** — they re-run the query every time. They're for *readability*, not perf.
- **Updatable views need to be "simple"** — single base table, no aggregates, no GROUP BY/HAVING/DISTINCT/UNION. Use `INSTEAD OF` triggers for complex cases.
- **`WITH CHECK OPTION` matters for updatable views** — without it, `UPDATE` can produce rows that don't satisfy the view's `WHERE`.
- **`REFRESH MATERIALIZED VIEW`** without CONCURRENTLY takes an `ACCESS EXCLUSIVE` lock — readers block. Always create the unique index and use CONCURRENTLY in production.
- **CONCURRENTLY refresh costs more I/O** — it does a full diff. Plain refresh is faster but blocks reads.
- **Stale data is the tradeoff** — set refresh cadence to match how stale users tolerate.
- **Don't materialize what's already fast** — only worth it for queries taking seconds+.
- **Schema changes in base tables** can break views. `pg_depend` shows dependencies.
- **Views don't persist across `pg_dump --schema-only` if recursive** — be careful with view-on-view chains during migrations.

---

## See Also

- [[06 - Query Optimization]]
- [[12 - Database Migrations]]
- [[09 - Performance Tuning]]
- [[12 - Data Warehousing]]
