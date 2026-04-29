---
tags: [database, advanced, postgresql, schema-design]
aliases: [System-Versioned, History Table, Audit Log, Time Travel]
level: Advanced
---

# Temporal and Audit Tables

> **One-liner**: Temporal tables answer "what did this row look like on Tuesday?"; audit tables answer "who changed this row, and why?".

---

## Quick Reference

| Need | Pattern |
|------|---------|
| Point-in-time row state | history table + `valid_from` / `valid_to` |
| Append-only change log | audit table per row change (INSERT/UPDATE/DELETE) |
| Schema-aware audit (operator + reason) | trigger captures `current_user`, app context |
| Per-row history kept inline | system-versioned tables (extension or hand-rolled) |
| Soft delete only | `deleted_at` column |
| Just need "who/when" — no values | application-level audit log + IDs |

| Term | Meaning |
|------|---------|
| **Application time** | when the fact was true in the world |
| **System time** | when the fact was recorded in the DB |
| **Bitemporal** | both — "as of system time X, the row's address was Y from app date W" |

---

## Core Concept

Two related but distinct problems:

1. **Temporal** — capture *every state* a row has had, queryable by time. Used for compliance ("show me Bob's address as of 2025-04-01"), undo, and reproducible reporting.
2. **Audit** — capture *every change* with metadata: who did it, when, from where, optionally why. Used for security forensics and compliance.

Postgres has no built-in `SYSTEM VERSIONING` (unlike SQL Server's *temporal tables*), but you can hand-roll it cleanly with two tables and a trigger:

- The **base** table holds the current state.
- The **history** table holds previous versions, each with `valid_from` / `valid_to`.

For audit, a single append-only `audit_log` with `JSONB` `before`/`after` snapshots is usually sufficient. Triggers capture them automatically.

For full history-as-truth, see [[06 - CQRS and Event Sourcing]] — events *are* the change log.

Be deliberate: storing every row version doubles write cost and storage. Decide table-by-table.

---

## Syntax & API

### History table + trigger
```sql
CREATE TABLE products (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku         TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    valid_from  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products_history (
    LIKE products,
    valid_to   TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE products_history
    ADD CONSTRAINT pk_history PRIMARY KEY (id, valid_from);

CREATE OR REPLACE FUNCTION snapshot_product()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO products_history
        SELECT OLD.*, now();         -- store old version with valid_to = now()
    NEW.valid_from := now();         -- new "current" starts now
    RETURN NEW;
END $$;

CREATE TRIGGER trg_products_history
BEFORE UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION snapshot_product();
```

### As-of query (point-in-time)
```sql
-- Product 42's state as of 2026-04-01
SELECT * FROM products
WHERE id = 42 AND valid_from <= '2026-04-01'
UNION ALL
SELECT id, sku, name, price, valid_from
FROM products_history
WHERE id = 42
  AND valid_from <= '2026-04-01'
  AND valid_to   >  '2026-04-01'
ORDER BY valid_from DESC LIMIT 1;
```

### A cleaner reusable view per table
```sql
CREATE OR REPLACE FUNCTION products_as_of(at TIMESTAMPTZ)
RETURNS SETOF products LANGUAGE sql STABLE AS $$
    SELECT id, sku, name, price, valid_from
    FROM products
    WHERE valid_from <= at
    UNION ALL
    SELECT id, sku, name, price, valid_from
    FROM products_history
    WHERE valid_from <= at AND valid_to > at
$$;

SELECT * FROM products_as_of('2026-04-01') WHERE id = 42;
```

### Generic audit log table
```sql
CREATE TABLE audit_log (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name   TEXT NOT NULL,
    op           TEXT NOT NULL,            -- 'INSERT' / 'UPDATE' / 'DELETE'
    row_pk       JSONB,
    before       JSONB,
    after        JSONB,
    changed_by   TEXT,
    changed_from INET,
    reason       TEXT,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_table_time ON audit_log (table_name, occurred_at DESC);
CREATE INDEX idx_audit_pk ON audit_log USING gin (row_pk jsonb_path_ops);
```

### Audit trigger (generic, reusable)
```sql
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    v_pk    JSONB;
    v_before JSONB;
    v_after  JSONB;
BEGIN
    v_pk := CASE TG_OP
                WHEN 'DELETE' THEN to_jsonb(OLD) -> 'id'
                ELSE to_jsonb(NEW) -> 'id' END;
    v_before := CASE TG_OP WHEN 'INSERT' THEN NULL ELSE to_jsonb(OLD) END;
    v_after  := CASE TG_OP WHEN 'DELETE' THEN NULL ELSE to_jsonb(NEW) END;

    INSERT INTO audit_log (table_name, op, row_pk, before, after,
                           changed_by, changed_from, reason)
    VALUES (TG_TABLE_NAME, TG_OP,
            jsonb_build_object('id', v_pk),
            v_before, v_after,
            current_setting('app.user_email', true),     -- nullable in PG 9.5+
            inet_client_addr(),
            current_setting('app.reason', true));

    RETURN NULL;
END $$;

CREATE TRIGGER trg_audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_changes();
```

App sets context at the start of each tx:
```sql
BEGIN;
SET LOCAL app.user_email = 'alice@example.com';
SET LOCAL app.reason = 'support ticket #1234';
UPDATE users SET name = 'Bob' WHERE id = 1;
COMMIT;
```

### Diff between two history rows
```sql
WITH a AS (SELECT to_jsonb(p) AS d FROM products_history p WHERE id=42 AND valid_from = '2026-01-01'),
     b AS (SELECT to_jsonb(p) AS d FROM products_history p WHERE id=42 AND valid_from = '2026-04-01')
SELECT key, a.d->key AS old, b.d->key AS new
FROM jsonb_object_keys((SELECT d FROM a)) AS key, a, b
WHERE a.d->key IS DISTINCT FROM b.d->key;
```

### Retention / archive
```sql
-- Move history older than 7 years to a separate cold table
INSERT INTO products_history_archive
SELECT * FROM products_history WHERE valid_to < now() - INTERVAL '7 years';
DELETE FROM products_history WHERE valid_to < now() - INTERVAL '7 years';
```

---

## Common Patterns

```sql
-- Pattern: bitemporal — both system and application time
CREATE TABLE addresses (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     INT NOT NULL,
    line1       TEXT NOT NULL,
    -- application time: when the fact is/was true in the world
    valid_from  DATE NOT NULL,
    valid_to    DATE,
    -- system time: when this row was recorded
    sys_from    TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to      TIMESTAMPTZ,
    EXCLUDE USING gist (user_id WITH =, daterange(valid_from, valid_to) WITH &&)
);
```

```sql
-- Pattern: append-only "facts" table, no UPDATE
CREATE TABLE measurements (
    sensor_id   INT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    value       NUMERIC NOT NULL,
    PRIMARY KEY (sensor_id, occurred_at)
);
-- Never UPDATE; corrections are new rows with later occurred_at
```

```sql
-- Pattern: lazy audit via logical decoding (no triggers)
-- Use Debezium or similar to read WAL → write to audit store
-- Pros: zero impact on app, captures everything
-- Cons: more moving parts; see [[13 - ETL and CDC]]
```

---

## Gotchas & Tips

- **History tables double writes** — every UPDATE costs an INSERT into history. Important for high-write tables.
- **Triggers are invisible** — document loudly. A maintainer should not be surprised by a magic history table.
- **Use generic audit (table_name + JSONB)** for many tables — one schema, one trigger function. Per-table mirroring duplicates effort.
- **Don't put PII in audit logs** if you can't comply with deletion requests — encrypt sensitive fields or mask.
- **Index `audit_log` by table + time** and by `row_pk` (GIN on JSONB) — those are the common access paths.
- **Read-only history** — `REVOKE INSERT, UPDATE, DELETE ON products_history FROM app` (allow only the trigger function via `SECURITY DEFINER` if needed).
- **Schema changes in base tables propagate to history** — keep them in sync; use a migration step that ALTERs both.
- **System-versioned tables in standard SQL** — Postgres 17+ has limited support; until then, hand-roll or use extensions like `temporal_tables`.
- **Backups still matter** — history isn't a substitute for [[15 - Backup and Restore]].
- **`current_setting('app.x', true)`** — second arg `true` returns NULL if unset (PG 9.6+), so triggers don't fail on missing context.

---

## See Also

- [[09 - Triggers]]
- [[06 - CQRS and Event Sourcing]]
- [[13 - ETL and CDC]]
- [[16 - Database Security]]
