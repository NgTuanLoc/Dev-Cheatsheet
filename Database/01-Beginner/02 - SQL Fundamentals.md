---
tags: [database, beginner, sql, postgresql]
aliases: [Basic SQL, CRUD]
level: Beginner
---

# SQL Fundamentals

> **One-liner**: SQL is a declarative language for querying relational databases — you describe **what** you want, the engine figures out **how**.

---

## Quick Reference

| Operation | Statement | Example |
|-----------|-----------|---------|
| Read | `SELECT … FROM … WHERE …` | `SELECT email FROM users WHERE id = 1` |
| Insert | `INSERT INTO … VALUES …` | `INSERT INTO users (email) VALUES ('a@b.com')` |
| Update | `UPDATE … SET … WHERE …` | `UPDATE users SET email='x' WHERE id=1` |
| Delete | `DELETE FROM … WHERE …` | `DELETE FROM users WHERE id=1` |
| Sort | `ORDER BY col [ASC|DESC]` | `ORDER BY created_at DESC` |
| Limit | `LIMIT n OFFSET m` | `LIMIT 10 OFFSET 20` |
| Distinct | `SELECT DISTINCT col` | `SELECT DISTINCT country FROM users` |
| Filter | `WHERE`, `AND`, `OR`, `NOT`, `IN`, `BETWEEN`, `LIKE`, `IS NULL` | see below |

---

## Core Concept

SQL has six **clause** keywords that almost always appear in this logical order:

```
SELECT  → projection (which columns)
FROM    → source (which table)
WHERE   → row filter (predicates)
GROUP BY → group rows
HAVING  → group filter
ORDER BY → sort
LIMIT   → cap rows
```

You write them in roughly that order, but the engine **executes** them in a different order: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`. That's why aliases defined in `SELECT` aren't visible in `WHERE`.

SQL is **declarative**: `SELECT name FROM users WHERE age > 18` doesn't tell the engine to scan, hash, or sort — only what answer you want. The query planner picks the strategy.

---

## Syntax & API

### Reading rows
```sql
-- All columns
SELECT * FROM users;

-- Specific columns + alias
SELECT id, email AS contact_email FROM users;

-- Filter
SELECT * FROM users WHERE age >= 18 AND country = 'US';

-- IN, BETWEEN, LIKE
SELECT * FROM users
WHERE country IN ('US', 'CA', 'UK')
  AND created_at BETWEEN '2024-01-01' AND '2024-12-31'
  AND email LIKE '%@gmail.com';

-- NULL is special — never use = NULL
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Sorting and limiting
```sql
SELECT * FROM users
ORDER BY created_at DESC, id ASC
LIMIT 10 OFFSET 20;          -- skip 20, take 10 (paging)
```

### Inserting
```sql
-- Single row, all columns
INSERT INTO users (email, age) VALUES ('a@b.com', 30);

-- Multiple rows
INSERT INTO users (email, age) VALUES
    ('a@b.com', 30),
    ('c@d.com', 25);

-- Postgres: return inserted row(s)
INSERT INTO users (email) VALUES ('e@f.com')
RETURNING id, created_at;
```

### Updating
```sql
UPDATE users
SET email = 'new@example.com', updated_at = now()
WHERE id = 42;

-- ALWAYS include WHERE — without it, you update every row
```

### Deleting
```sql
DELETE FROM users WHERE id = 42;

-- Soft delete (more common in production)
UPDATE users SET deleted_at = now() WHERE id = 42;
```

### Distinct
```sql
SELECT DISTINCT country FROM users;
SELECT DISTINCT country, status FROM users;   -- distinct combination
```

---

## Common Patterns

```sql
-- Pattern: paging
SELECT id, email FROM users
ORDER BY id
LIMIT 25 OFFSET (page - 1) * 25;

-- Pattern: keyset pagination (faster on big tables)
SELECT id, email FROM users
WHERE id > :last_seen_id
ORDER BY id
LIMIT 25;
```

```sql
-- Pattern: upsert (Postgres ON CONFLICT)
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

```sql
-- Pattern: count and exists
SELECT COUNT(*) FROM users WHERE country = 'US';

-- EXISTS is often faster than COUNT > 0
SELECT EXISTS(SELECT 1 FROM users WHERE email = 'a@b.com');
```

---

## Gotchas & Tips

- **`NULL` is not equal to anything, including itself** — `NULL = NULL` is `NULL` (treated as false). Use `IS NULL` / `IS NOT NULL`.
- **`UPDATE` and `DELETE` without `WHERE` modify every row.** Postgres has no built-in safety; some clients (DataGrip, DBeaver) warn. Wrap risky changes in a transaction: `BEGIN; UPDATE …; -- inspect; COMMIT;`
- **`SELECT *` is fine for ad-hoc, bad for app code** — column order or count changes break consumers. Always list columns in production queries.
- **`OFFSET` gets slow** on big tables — the engine still reads and discards skipped rows. Prefer keyset pagination (`WHERE id > :last`).
- **`LIKE` is case-sensitive in Postgres**; use `ILIKE` for case-insensitive. Leading-wildcard `LIKE '%foo'` can't use a normal index.
- **Identifiers are folded to lowercase** unless quoted: `SELECT Id` reads column `id`, but `SELECT "Id"` is case-sensitive.
- **String literals use single quotes**, never double: `'hello'` is a string, `"hello"` is an identifier. Escape a single quote by doubling: `'O''Brien'`.

---

## See Also

- [[06 - Joins]]
- [[07 - Aggregations and Grouping]]
- [[09 - Constraints]]
- [[02 - Transactions and ACID]]
