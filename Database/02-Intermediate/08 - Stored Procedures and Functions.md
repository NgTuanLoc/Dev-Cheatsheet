---
tags: [database, intermediate, postgresql]
aliases: [PL/pgSQL, FUNCTION, PROCEDURE, RPC]
level: Intermediate
---

# Stored Procedures and Functions

> **One-liner**: Functions and procedures let you bundle SQL (and procedural logic) in the database — useful for set-based operations, but easy to overuse.

---

## Quick Reference

| Object | Returns | Can `COMMIT`/`ROLLBACK` inside? | Called via |
|--------|---------|---------------------------------|------------|
| **Function** | a value, set, or table | No (PG 11+: only `PROCEDURE` can) | `SELECT fn(args)` or in queries |
| **Procedure** | nothing (output via OUT params) | Yes | `CALL proc(args)` |
| **Trigger function** | special — returns row | No | invoked by triggers |

| Postgres languages | Use |
|--------------------|-----|
| `LANGUAGE sql` | Pure SQL body — fastest, inline-able by planner |
| `LANGUAGE plpgsql` | Procedural — variables, loops, exception handlers |
| `LANGUAGE plpython3u`, `plperl`, etc. | Custom languages (require extension; use sparingly) |

---

## Core Concept

A **function** computes and returns a value. It can return a scalar, a row, a set of rows, or an arbitrary table. Functions are usable in `SELECT`, `WHERE`, etc.:

```sql
SELECT name, full_name(first, last) FROM people;
SELECT * FROM top_users(10);
```

A **procedure** is for side effects with no return value. It can manage its own transactions (`COMMIT`/`ROLLBACK` inside the body) — useful for batch jobs.

The default body language is **PL/pgSQL** — Postgres's own procedural language with variables, control flow, and exception handlers. For pure SQL bodies, prefer `LANGUAGE sql` (faster, may be inlined).

Functions are great for **set-based logic** (operate on many rows in one call). They're not so great for hiding application logic — they're harder to test, version, and observe than app code. Use deliberately.

---

## Syntax & API

### Pure SQL function (preferred when possible)
```sql
CREATE OR REPLACE FUNCTION full_name(first TEXT, last TEXT)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE                        -- no side effects, deterministic
PARALLEL SAFE
AS $$
    SELECT first || ' ' || last
$$;

SELECT full_name('Alice', 'Smith');     -- 'Alice Smith'
```

### PL/pgSQL function with logic
```sql
CREATE OR REPLACE FUNCTION transfer(p_from INT, p_to INT, p_amount NUMERIC)
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    src_balance NUMERIC;
BEGIN
    -- Lock rows in PK order to avoid deadlock
    PERFORM 1 FROM accounts WHERE id IN (p_from, p_to)
        ORDER BY id FOR UPDATE;

    SELECT balance INTO src_balance FROM accounts WHERE id = p_from;
    IF src_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: % < %', src_balance, p_amount
            USING ERRCODE = 'P0001';
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to;
END;
$$;

SELECT transfer(1, 2, 100);
```

### Set-returning function (TABLE)
```sql
CREATE OR REPLACE FUNCTION top_users(p_n INT)
RETURNS TABLE (id INT, email TEXT, revenue NUMERIC)
LANGUAGE sql STABLE
AS $$
    SELECT u.id, u.email, COALESCE(SUM(o.total), 0) AS revenue
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    GROUP BY u.id, u.email
    ORDER BY revenue DESC
    LIMIT p_n
$$;

SELECT * FROM top_users(10);
```

### Function returning a record / row type
```sql
CREATE OR REPLACE FUNCTION find_user(p_email TEXT)
RETURNS users
LANGUAGE sql STABLE
AS $$
    SELECT * FROM users WHERE email = p_email LIMIT 1
$$;
```

### Procedure with transaction control
```sql
CREATE OR REPLACE PROCEDURE archive_old_orders()
LANGUAGE plpgsql
AS $$
DECLARE
    cutoff TIMESTAMPTZ := now() - INTERVAL '5 years';
BEGIN
    LOOP
        WITH chunk AS (
            DELETE FROM orders
            WHERE placed_at < cutoff
            RETURNING *
        )
        INSERT INTO orders_archive SELECT * FROM chunk;

        EXIT WHEN NOT FOUND;
        COMMIT;                          -- only allowed in procedures
    END LOOP;
END;
$$;

CALL archive_old_orders();
```

### Function with default and named parameters
```sql
CREATE OR REPLACE FUNCTION search_users(
    p_query   TEXT,
    p_limit   INT DEFAULT 10,
    p_offset  INT DEFAULT 0
)
RETURNS SETOF users
LANGUAGE sql STABLE
AS $$
    SELECT * FROM users
    WHERE name ILIKE '%' || p_query || '%'
    ORDER BY id
    LIMIT p_limit OFFSET p_offset
$$;

SELECT * FROM search_users('alice', p_limit => 5);
```

### Volatility flags (matter for the planner)
```sql
-- IMMUTABLE: deterministic, no side effects, no DB reads → may be cached
-- STABLE:    deterministic within a transaction; reads DB allowed
-- VOLATILE:  default — assume anything (writes, random, time-dependent)
```

### Drop / list
```sql
DROP FUNCTION transfer(INT, INT, NUMERIC);
\df+ transfer        -- psql: detail
SELECT proname FROM pg_proc WHERE proname LIKE 'transfer%';
```

---

## Common Patterns

```sql
-- Pattern: validation function used in CHECK
CREATE FUNCTION is_positive(n NUMERIC) RETURNS BOOLEAN
LANGUAGE sql IMMUTABLE AS $$ SELECT n > 0 $$;

ALTER TABLE products ADD CONSTRAINT ck_price CHECK (is_positive(price));
```

```sql
-- Pattern: encapsulate complex insert with returning
CREATE FUNCTION create_order(p_user INT, p_items JSONB)
RETURNS INT
LANGUAGE plpgsql AS $$
DECLARE
    v_order_id INT;
BEGIN
    INSERT INTO orders (user_id) VALUES (p_user) RETURNING id INTO v_order_id;
    INSERT INTO order_items (order_id, product_id, quantity)
    SELECT v_order_id, (i->>'product_id')::INT, (i->>'quantity')::INT
    FROM jsonb_array_elements(p_items) AS i;
    RETURN v_order_id;
END $$;
```

```sql
-- Pattern: table-valued function as a parameterized view
CREATE FUNCTION active_orders(p_since TIMESTAMPTZ)
RETURNS SETOF orders
LANGUAGE sql STABLE AS $$
    SELECT * FROM orders WHERE placed_at >= p_since AND status <> 'cancelled'
$$;
```

---

## Gotchas & Tips

- **Function names overload by argument signature** — `fn(INT)` and `fn(TEXT)` are different. Drop with `DROP FUNCTION fn(INT)`.
- **Volatility hints affect plan caching** — mark IMMUTABLE/STABLE when accurate; otherwise the planner can't cache or inline.
- **`LANGUAGE sql` may be inlined** by the planner; PL/pgSQL is opaque. Pure SQL when possible.
- **PL/pgSQL is *not* application code** — no string concatenation for queries (SQL injection!). Use parameterized `EXECUTE … USING …` if you must build dynamic SQL.
- **Versioning is hard** — DB code lives outside your repo unless you check it in (use [[12 - Database Migrations]]).
- **Testing is harder** — apps test with mocks; functions need a live DB. Use frameworks like pgTAP if you go heavy.
- **Procedures are PG 11+** — older Postgres only had functions; people simulated procedures with `RETURNS VOID`.
- **Don't put critical business logic in DB without a strong reason** — observability, deployability, and team familiarity usually favor app code. DB-side shines for set-based ops and atomic invariants.
- **Use `RAISE NOTICE` for debugging** — it shows in psql / log output. Remove before prod.
- **Encrypt secrets, not in function bodies** — bodies are visible to anyone with `pg_proc` read access.

---

## See Also

- [[09 - Triggers]]
- [[02 - Transactions and ACID]]
- [[12 - Database Migrations]]
- [[16 - Database Security]]
