---
tags: [database, intermediate, schema-design, postgresql]
aliases: [1NF, 2NF, 3NF, BCNF, Denormalization]
level: Intermediate
---

# Normalization

> **One-liner**: Normalization is splitting tables to remove redundancy and update anomalies; denormalization is selectively reintroducing duplication for query speed.

---

## Quick Reference

| Form | Rule | Test |
|------|------|------|
| **1NF** | Each cell holds one atomic value; rows are unique | "Is this field a list/struct?" |
| **2NF** | 1NF + no partial dependency on a composite PK | "Does a non-key column depend on only PART of the PK?" |
| **3NF** | 2NF + no transitive dependency between non-key columns | "Does column A depend on column B which depends on PK?" |
| **BCNF** | Stricter 3NF — every determinant is a candidate key | rare to hit in practice once 3NF is reached |
| **Denormalization** | Intentionally violate 3NF for query speed | "Is the join cost real and measured?" |

---

## Core Concept

Normalization is a discipline for **avoiding three anomalies** in relational data:

- **Insert anomaly**: can't add a fact without inventing unrelated data
- **Update anomaly**: same fact stored in many rows; one update misses copies → inconsistency
- **Delete anomaly**: deleting a row destroys an unrelated fact

Each "normal form" closes a class of these by **splitting** tables and using **foreign keys** to reconnect.

- **1NF** — atomic columns, no repeating groups. (`tags TEXT` storing `'a,b,c'` violates it; an array column or junction table fixes it.)
- **2NF** — relevant only with composite PKs: every non-key column must depend on the *whole* PK, not just part.
- **3NF** — non-key columns must depend only on the PK, not on each other. (Storing `city, country` where `country` is determined by `city` violates it; move `city → country` to a `cities` lookup table.)
- **BCNF** — like 3NF, but every functional dependency's left side must be a candidate key. Most well-modeled 3NF schemas are already BCNF.

3NF is the practical target. **Denormalize** only when measurements show that joins are the bottleneck and the redundant column is rarely written.

---

## Syntax & API

### From unnormalized to 3NF (e-commerce example)

#### Bad — flat / 1NF violation
```sql
-- Comma-list of items in a single column = NOT 1NF
CREATE TABLE bad_orders (
    id          INT PRIMARY KEY,
    user_email  TEXT,
    user_name   TEXT,           -- duplicated for every order
    items       TEXT             -- "shoes:2,hat:1" — atomic? no
);
```

#### 1NF: atomic columns, separate row per item
```sql
CREATE TABLE orders_1nf (
    order_id    INT,
    item_name   TEXT,
    quantity    INT,
    user_email  TEXT,
    user_name   TEXT,
    PRIMARY KEY (order_id, item_name)
);
-- Better — but user_name still duplicated for every (order, item) row
```

#### 2NF: split partial dependencies
```sql
-- order_id alone determines user_email and user_name → move out
CREATE TABLE orders_2nf (
    order_id    INT PRIMARY KEY,
    user_email  TEXT,
    user_name   TEXT
);
CREATE TABLE order_items_2nf (
    order_id    INT,
    item_name   TEXT,
    quantity    INT,
    PRIMARY KEY (order_id, item_name)
);
```

#### 3NF: split transitive dependencies
```sql
-- user_email determines user_name → split into users
CREATE TABLE users_3nf (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email  TEXT UNIQUE NOT NULL,
    name   TEXT NOT NULL
);
CREATE TABLE orders_3nf (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users_3nf(id)
);
CREATE TABLE order_items_3nf (
    order_id   INT NOT NULL REFERENCES orders_3nf(id) ON DELETE CASCADE,
    product_id INT NOT NULL REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)
);
```

### Denormalize when measured
```sql
-- 3NF view — joins orders + items + products
SELECT o.id, p.name, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;

-- Denormalized snapshot column — capture price AT order time
ALTER TABLE order_items ADD COLUMN unit_price NUMERIC(10, 2) NOT NULL;
-- Even if product price changes later, history is preserved.
-- This is "intentional redundancy" — desirable here.
```

---

## Common Patterns

```sql
-- Pattern: lookup tables for low-cardinality enums
CREATE TABLE order_statuses (
    code  TEXT PRIMARY KEY,
    label TEXT NOT NULL
);
INSERT INTO order_statuses VALUES
    ('pending', 'Pending'), ('paid','Paid'), ('shipped','Shipped');

ALTER TABLE orders
    ADD COLUMN status_code TEXT NOT NULL REFERENCES order_statuses(code);
```

```sql
-- Pattern: snapshot pricing (intentional denormalization for history)
CREATE TABLE order_items (
    order_id    INT,
    product_id  INT,
    quantity    INT,
    unit_price  NUMERIC(10, 2) NOT NULL,    -- captured at order time
    PRIMARY KEY (order_id, product_id)
);
```

```sql
-- Pattern: materialized aggregate (denormalized cache)
CREATE MATERIALIZED VIEW user_revenue AS
SELECT user_id, SUM(total) AS revenue, MAX(placed_at) AS last_order
FROM orders GROUP BY user_id;

REFRESH MATERIALIZED VIEW CONCURRENTLY user_revenue;
```

---

## Gotchas & Tips

- **3NF is the goal, not the limit** — going past BCNF rarely matters in practice.
- **Snapshots (price, address) are NOT denormalization violations** — they're capturing history. The "current" version still lives in the parent table.
- **Don't denormalize prematurely** — JOINs with proper indexes are very fast on Postgres. Profile first.
- **OLAP / analytics workloads often denormalize on purpose** — star schemas (see [[12 - Data Warehousing]]) trade write effort for query speed.
- **Avoid lookups for true two-state booleans** — `is_active BOOLEAN` is fine; don't model it as `state_id INT REFERENCES states`.
- **Composite PKs aren't normalization violations** — junction tables need them.
- **JSONB is a normalization escape hatch** — useful for sparse/optional fields, terrible for relational queries. Don't put obvious relational data in JSONB.
- **Check schema after every ALTER** — `\d table_name` to see constraints, indexes, FKs.

---

## See Also

- [[04 - Schema Design Basics]]
- [[05 - Keys and Relationships]]
- [[12 - Data Warehousing]]
- [[18 - JSON and JSONB]]
