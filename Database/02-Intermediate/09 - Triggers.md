---
tags: [database, intermediate, postgresql]
aliases: [TRIGGER, BEFORE, AFTER, INSTEAD OF]
level: Intermediate
---

# Triggers

> **One-liner**: A trigger runs a function automatically on `INSERT`/`UPDATE`/`DELETE` — invisible side effects that are powerful and easy to abuse.

---

## Quick Reference

| Timing | When it runs | Typical use |
|--------|--------------|-------------|
| `BEFORE` | Before the row change | normalize/derive columns, reject bad data |
| `AFTER` | After the row change | audit log, propagate to other tables |
| `INSTEAD OF` | Replaces the change (views only) | make complex views writable |

| Granularity | Fires |
|-------------|-------|
| `FOR EACH ROW` | Once per affected row; sees `OLD`, `NEW` |
| `FOR EACH STATEMENT` | Once per statement; sees no row data (PG 10+: transition tables) |

| Special variables (in trigger function) | |
|------------------------------------------|---|
| `NEW` | The new row (INSERT/UPDATE) |
| `OLD` | The previous row (UPDATE/DELETE) |
| `TG_OP` | `'INSERT'` / `'UPDATE'` / `'DELETE'` |
| `TG_TABLE_NAME` | the table name |

---

## Core Concept

A trigger is a **callback** the DB fires for matching events. You write a *trigger function* (returning `TRIGGER`) and bind it to a table with `CREATE TRIGGER`.

Two big patterns:

- **`BEFORE` row trigger** — modify or validate the row about to be written. Setting `NEW.updated_at = now()` or rejecting a row with `RAISE EXCEPTION` belongs here.
- **`AFTER` row trigger** — react to a write that already happened. Audit logging, denormalized counter updates, sending notifications.

`INSTEAD OF` triggers exist only on views; they let you write through views that aren't auto-updatable (e.g., aggregating views).

Statement-level triggers fire once per statement regardless of row count — useful for refresh/notify/audit summaries when row-count doesn't matter.

Triggers are **invisible**: a developer reading the app code won't know they exist. They run on every write, costing performance, and can cascade. Use sparingly and document loudly.

---

## Syntax & API

### `BEFORE` trigger — auto-update timestamp
```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### `AFTER` trigger — write to an audit table
```sql
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    op          TEXT NOT NULL,
    row_id      INT,
    old_data    JSONB,
    new_data    JSONB,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE OR REPLACE FUNCTION log_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO audit_log (table_name, op, row_id, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        COALESCE(NEW.id, OLD.id),
        CASE TG_OP WHEN 'INSERT' THEN NULL ELSE to_jsonb(OLD) END,
        CASE TG_OP WHEN 'DELETE' THEN NULL ELSE to_jsonb(NEW) END
    );
    RETURN NULL;             -- AFTER trigger ignores return value
END;
$$;

CREATE TRIGGER trg_users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION log_changes();
```

### Validation that aborts
```sql
CREATE OR REPLACE FUNCTION reject_negative_balance()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.balance < 0 THEN
        RAISE EXCEPTION 'Negative balance not allowed (id=%)', NEW.id
            USING ERRCODE = 'P0001';
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_accounts_no_negative
BEFORE INSERT OR UPDATE ON accounts
FOR EACH ROW
EXECUTE FUNCTION reject_negative_balance();
-- (Better in this case: a CHECK constraint. Use triggers when CHECK can't express it.)
```

### `INSTEAD OF` on a view
```sql
CREATE VIEW user_summary AS
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- This view isn't naturally updatable
CREATE OR REPLACE FUNCTION user_summary_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE users SET name = NEW.name WHERE id = NEW.id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_user_summary_update
INSTEAD OF UPDATE ON user_summary
FOR EACH ROW
EXECUTE FUNCTION user_summary_update();
```

### Statement-level with transition tables (PG 10+)
```sql
CREATE OR REPLACE FUNCTION log_bulk_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO bulk_audit (n_changed, occurred_at)
    SELECT COUNT(*), now() FROM new_table;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_users_bulk
AFTER UPDATE ON users
REFERENCING NEW TABLE AS new_table OLD TABLE AS old_table
FOR EACH STATEMENT
EXECUTE FUNCTION log_bulk_changes();
```

### Inspect / drop
```sql
\d users                              -- shows triggers
SELECT tgname, tgenabled FROM pg_trigger WHERE tgrelid = 'users'::regclass;

DROP TRIGGER trg_users_updated_at ON users;
ALTER TABLE users DISABLE TRIGGER trg_users_audit;     -- temporarily off
ALTER TABLE users ENABLE TRIGGER trg_users_audit;
```

---

## Common Patterns

```sql
-- Pattern: maintain a denormalized counter
CREATE OR REPLACE FUNCTION bump_post_comment_count()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts SET comment_count = comment_count + 1 WHERE id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts SET comment_count = comment_count - 1 WHERE id = OLD.post_id;
    END IF;
    RETURN NULL;
END $$;

CREATE TRIGGER trg_comments_count
AFTER INSERT OR DELETE ON comments
FOR EACH ROW
EXECUTE FUNCTION bump_post_comment_count();
```

```sql
-- Pattern: NOTIFY on row change for live subscriptions
CREATE OR REPLACE FUNCTION notify_change()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    PERFORM pg_notify('orders_changes', NEW.id::TEXT);
    RETURN NULL;
END $$;

CREATE TRIGGER trg_orders_notify
AFTER INSERT OR UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION notify_change();
-- Listeners: LISTEN orders_changes;
```

```sql
-- Pattern: temporal table (history)
CREATE TABLE users_history (
    LIKE users INCLUDING ALL,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_to   TIMESTAMPTZ
);

CREATE OR REPLACE FUNCTION snapshot_users()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO users_history SELECT OLD.*, OLD.updated_at, now();
    RETURN NEW;
END $$;

CREATE TRIGGER trg_users_history
BEFORE UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION snapshot_users();
```

---

## Gotchas & Tips

- **Triggers are invisible** — a maintainer reading the app won't see them. Document in CLAUDE.md / README and prefer obvious solutions when possible.
- **`AFTER` triggers fire even if a later constraint fails** — actually they fire at the right time, but a tx rollback unwinds them. They run within the same tx.
- **`BEFORE` triggers can mutate `NEW`** — `NEW.col := …` is the canonical way to default/normalize.
- **Return matters in BEFORE row triggers** — return `NULL` to skip the row entirely; return `NEW` (possibly modified) to allow it. AFTER triggers ignore return.
- **CHECK constraints beat triggers for simple validation** — they're declarative, the planner sees them, no PL/pgSQL overhead.
- **Cascades are easy to write, hard to debug** — trigger on table A updates B, which has a trigger updating C, … Track depth.
- **Triggers don't fire on `TRUNCATE` by default** — separate event: `CREATE TRIGGER … BEFORE TRUNCATE …`.
- **`COPY` and bulk loads fire row triggers row-by-row** — slow. Use statement-level when possible, or disable triggers during bulk loads.
- **Test triggers thoroughly** — they fire under every code path, including future ones nobody anticipated.
- **For audit, prefer logical replication or CDC** ([[13 - ETL and CDC]]) for high-volume systems — triggers don't scale forever.

---

## See Also

- [[08 - Stored Procedures and Functions]]
- [[09 - Constraints]]
- [[07 - Temporal and Audit Tables]]
- [[13 - ETL and CDC]]
