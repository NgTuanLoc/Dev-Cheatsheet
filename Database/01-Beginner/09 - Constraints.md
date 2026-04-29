---
tags: [database, beginner, schema-design, postgresql]
aliases: [NOT NULL, UNIQUE, CHECK, FOREIGN KEY]
level: Beginner
---

# Constraints

> **One-liner**: Constraints are rules the database enforces on your data — they make bad data impossible, not just unlikely.

---

## Quick Reference

| Constraint | Purpose | Example |
|------------|---------|---------|
| `NOT NULL` | Column must have a value | `email TEXT NOT NULL` |
| `UNIQUE` | No duplicates (NULLs are allowed multiple times) | `UNIQUE (email)` |
| `PRIMARY KEY` | `UNIQUE` + `NOT NULL` + identity | `PRIMARY KEY (id)` |
| `FOREIGN KEY` (FK) | Must reference existing PK in another table | `REFERENCES users(id)` |
| `CHECK` | Custom boolean rule per row | `CHECK (price >= 0)` |
| `DEFAULT` | Value used when none provided | `DEFAULT now()` |
| `EXCLUSION` | Generalized uniqueness (no overlapping ranges, etc.) | `EXCLUDE USING gist (room_id WITH =, period WITH &&)` |

| FK action | Behavior on parent change |
|-----------|---------------------------|
| `RESTRICT` (default) | Reject if children exist |
| `CASCADE` | Apply same change to children |
| `SET NULL` | Null the FK column in children |
| `SET DEFAULT` | Set the FK to its default |
| `NO ACTION` | Like `RESTRICT`, checked at end of statement |

---

## Core Concept

Constraints push validation **into** the database, where every client and every code path is forced through them — application bugs, ad-hoc SQL, ETL imports all obey. The DB rejects bad data with an error.

Five everyday constraints cover most needs:
1. `NOT NULL` — column always has a value
2. `UNIQUE` — column (or combination) is unique across rows
3. `PRIMARY KEY` — the row identifier (`UNIQUE` + `NOT NULL`)
4. `FOREIGN KEY` — references another table's PK; prevents orphans
5. `CHECK` — arbitrary per-row predicate (`age >= 0`, `start_date < end_date`)

A constraint can be defined **inline** on a column or as a **table-level** constraint (required for composite PKs/FKs/UNIQUEs). Naming constraints (`CONSTRAINT my_name`) makes errors and migrations cleaner.

---

## Syntax & API

### Inline column-level
```sql
CREATE TABLE products (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku    TEXT NOT NULL UNIQUE,
    name   TEXT NOT NULL,
    price  NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock  INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
);
```

### Table-level (named, composite)
```sql
CREATE TABLE order_items (
    order_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,

    CONSTRAINT pk_order_items     PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_oi_order        FOREIGN KEY (order_id)
        REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_oi_product      FOREIGN KEY (product_id)
        REFERENCES products(id),
    CONSTRAINT ck_oi_quantity_pos CHECK (quantity > 0),
    CONSTRAINT ck_oi_price_pos    CHECK (unit_price >= 0)
);
```

### Adding constraints later
```sql
ALTER TABLE users
    ADD CONSTRAINT uq_users_email UNIQUE (email);

ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT;

ALTER TABLE products
    ADD CONSTRAINT ck_price_positive CHECK (price > 0);

-- Add NOT NULL (Postgres rewrites the table if old rows have NULLs)
ALTER TABLE users
    ALTER COLUMN name SET NOT NULL;
```

### Removing constraints
```sql
ALTER TABLE users DROP CONSTRAINT uq_users_email;
ALTER TABLE users ALTER COLUMN name DROP NOT NULL;
```

### Conditional UNIQUE (partial index)
```sql
-- Email must be unique only among NON-deleted users
CREATE UNIQUE INDEX uq_users_email_live
    ON users (email)
    WHERE deleted_at IS NULL;
```

### CHECK across columns
```sql
CREATE TABLE bookings (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    starts_at   TIMESTAMPTZ NOT NULL,
    ends_at     TIMESTAMPTZ NOT NULL,
    CONSTRAINT ck_booking_range CHECK (ends_at > starts_at)
);
```

### EXCLUSION constraint (overlapping ranges)
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE reservations (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INT NOT NULL,
    period  TSTZRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, period WITH &&)  -- no overlap per room
);
```

---

## Common Patterns

```sql
-- Pattern: domain-via-CHECK (poor man's enum)
CREATE TABLE orders (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled'))
);
-- Adding a value = ALTER (no need to recreate type, unlike ENUM)
```

```sql
-- Pattern: at-most-one row matching a condition
-- Each user can have at most one PRIMARY address
CREATE UNIQUE INDEX uq_addresses_primary
    ON addresses (user_id)
    WHERE is_primary = true;
```

```sql
-- Pattern: deferrable FK (check at COMMIT, not per-statement)
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    DEFERRABLE INITIALLY DEFERRED;
-- Useful when inserting interrelated rows in any order in one transaction
```

---

## Gotchas & Tips

- **`UNIQUE` allows multiple NULLs** — Postgres treats NULLs as distinct. To enforce "unique non-null", use a partial unique index `WHERE col IS NOT NULL`.
- **`PRIMARY KEY` implies both `UNIQUE` and `NOT NULL`** — don't add them redundantly.
- **`CHECK` must be deterministic and side-effect-free** — `CHECK (created_at <= now())` works on insert but won't be re-checked later.
- **Foreign keys auto-create an index? No.** Postgres creates the FK constraint, but **you** must create the index on the FK column (the parent's PK is auto-indexed). Missing FK indexes cause slow deletes/updates.
- **Validation should be centralized** — put it in the DB. Application validation is a nice UX touch, but the DB is the truth.
- **Constraint names matter** — Postgres auto-names them `tablename_columnname_key` etc. Naming explicitly (`CONSTRAINT uq_users_email`) makes errors and dropping easier.
- **Adding `NOT NULL` to a populated table is expensive** — it has to rewrite. Add the column with a `DEFAULT`, backfill, then `SET NOT NULL`. (Postgres 11+ does fast non-null defaults for new columns.)
- **`DEFERRABLE` is rarely needed** — most apps don't need it; useful for circular FKs or bulk loads.
- **Don't enforce business logic in `CHECK` for things that change** — "user must be 18+" is fine, "stock must be > order_count" is not (it depends on other tables).

---

## See Also

- [[03 - Data Types]]
- [[04 - Schema Design Basics]]
- [[05 - Keys and Relationships]]
- [[10 - Indexes Basics]]
