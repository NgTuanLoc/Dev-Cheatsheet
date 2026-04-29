---
tags: [database, beginner, schema-design, postgresql]
aliases: [Schema Design, Tables and Naming]
level: Beginner
---

# Schema Design Basics

> **One-liner**: A schema is the blueprint of your database — tables, columns, types, and the relationships between them. Good schemas make queries simple and bad data impossible.

---

## Quick Reference

| Item | Convention |
|------|------------|
| Table names | plural, snake_case: `users`, `order_items` |
| Column names | singular, snake_case: `user_id`, `created_at` |
| Primary key | `id` (surrogate) or natural key with `UNIQUE` constraint |
| Foreign key | `<referenced_singular>_id`: `user_id` references `users(id)` |
| Boolean | `is_<adjective>`: `is_active`, `is_deleted` |
| Timestamps | `created_at`, `updated_at`, `deleted_at` (TIMESTAMPTZ) |
| Junction table | both names alphabetized: `posts_tags` not `tags_posts` |
| Avoid | reserved words (`user`, `order`, `select`), CamelCase |

---

## Core Concept

A **table** is a list of rows; a **row** is a fact (one user, one order, one event). A **column** is an attribute of that fact, with a fixed type. A **schema** is a collection of tables and how they relate.

Three principles for beginners:

1. **One thing per table.** A `users` table holds users — not their orders, not their addresses (those are their own tables, joined by foreign keys).
2. **Identity matters.** Every row needs a stable, unique identifier — usually a synthetic `id` (surrogate key). Natural keys (email, SSN) can change; IDs shouldn't.
3. **Names should read like sentences.** `SELECT email FROM users WHERE id = 1` reads naturally. `SELECT Email FROM tbl_User WHERE UserID = 1` does not.

A separate **schema** namespace (Postgres concept, not the same word) groups tables — `public.users`, `audit.events`. The default is `public`. You usually only split schemas in larger systems (auth, billing, audit).

---

## Syntax & API

### Creating a clean table
```sql
CREATE TABLE users (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email        TEXT        NOT NULL UNIQUE,
    name         TEXT        NOT NULL,
    is_active    BOOLEAN     NOT NULL DEFAULT true,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ
);

CREATE TABLE addresses (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id      INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    line1        TEXT NOT NULL,
    city         TEXT NOT NULL,
    country      TEXT NOT NULL,
    is_primary   BOOLEAN NOT NULL DEFAULT false
);
```

### Schema namespaces
```sql
CREATE SCHEMA audit;

CREATE TABLE audit.events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    actor_id    INT,
    action      TEXT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Set the search path (where unqualified names are looked up)
SET search_path = public, audit;

SELECT * FROM events;          -- finds audit.events
```

### Modifying a schema (DDL)
```sql
ALTER TABLE users ADD COLUMN phone TEXT;
ALTER TABLE users ALTER COLUMN name SET NOT NULL;
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users DROP COLUMN phone;

ALTER TABLE users RENAME TO accounts;
DROP TABLE accounts;          -- gone — irreversible
```

### Look at a table's structure
```sql
\d users               -- psql shortcut
\d+ users              -- with extra detail (storage, comments)

-- SQL standard
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';
```

---

## Common Patterns

```sql
-- Pattern: surrogate id + natural unique key
CREATE TABLE products (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku    TEXT NOT NULL UNIQUE          -- natural key (stable, externally meaningful)
);
-- Apps reference id; humans/integrations use sku.
```

```sql
-- Pattern: soft delete with deleted_at
CREATE TABLE posts (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT NOT NULL,
    body        TEXT NOT NULL,
    deleted_at  TIMESTAMPTZ            -- NULL = live, set = deleted
);

-- Filter live rows
CREATE VIEW posts_live AS
    SELECT * FROM posts WHERE deleted_at IS NULL;
```

```sql
-- Pattern: junction table for many-to-many
CREATE TABLE posts_tags (
    post_id  INT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id   INT NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)        -- composite PK prevents duplicates
);
```

---

## Gotchas & Tips

- **Don't name a table `user`** — it's a reserved word in SQL. Use `users`, `accounts`, or `app_user`.
- **Plural table, singular column** is the most common convention; pick one and be consistent.
- **`id` everywhere is fine** — `users.id`, `orders.id`, etc. The qualifier (`users.id`) makes it clear in joins.
- **CamelCase requires quoting** — `"UserId"` is case-sensitive; `UserId` (unquoted) becomes `userid`. Use snake_case to avoid the trap.
- **Don't reuse columns across tables for unrelated meaning** — `status` should mean roughly the same kind of thing if used in multiple tables. Otherwise rename: `order_status`, `payment_status`.
- **Booleans should default explicitly** — `NOT NULL DEFAULT false` is clearer than nullable booleans (which give three states: true/false/unknown).
- **Add `created_at` to almost every table** — costs little, helps debugging massively.
- **Resist over-modeling early** — start with the columns you need. Add normalization (lookup tables, etc.) when repetition becomes painful.

---

## See Also

- [[03 - Data Types]]
- [[05 - Keys and Relationships]]
- [[09 - Constraints]]
- [[01 - Normalization]]
