---
tags: [database, index, learning-path]
aliases: [DB Roadmap, DB Curriculum]
---

# Database Learning Path

> **Recommended order** to study the Database topic. Stages assume zero database background and build to production-grade depth. Primary dialect: PostgreSQL.

---

## Stage 1 — SQL Fluency (1–2 weeks)

**Goal**: Read and write everyday SQL queries without consulting docs.

1. [[01 - Database Overview]] — orient the landscape
2. [[02 - SQL Fundamentals]]
3. [[03 - Data Types]]
4. [[04 - Schema Design Basics]]
5. [[05 - Keys and Relationships]]
6. [[06 - Joins]]
7. [[07 - Aggregations and Grouping]]
8. [[08 - Subqueries and CTEs]]
9. [[09 - Constraints]]
10. [[10 - Indexes Basics]]

**Checkpoint**: Design a small e-commerce schema (users, orders, products), populate it, and write 10 analytical queries with joins and aggregates.

---

## Stage 2 — Schema Design and Concurrency (1 week)

**Goal**: Model real domains and reason about transactions correctly.

1. [[01 - Normalization]]
2. [[02 - Transactions and ACID]]
3. [[03 - Isolation Levels]]
4. [[04 - Locking and Concurrency]]
5. [[09 - Constraints]] *(revisit with deeper eyes)*

**Checkpoint**: Implement a money-transfer flow with explicit transactions and the right isolation level. Reason about which anomalies your code allows.

---

## Stage 3 — Performance and Indexing (1–2 weeks)

**Goal**: Make slow queries fast. Read execution plans.

1. [[05 - Indexes Advanced]]
2. [[06 - Query Optimization]]
3. [[10 - Window Functions]]
4. [[07 - Views]]
5. [[11 - Set Operations and Pivoting]]

**Checkpoint**: Take a slow query, run `EXPLAIN ANALYZE`, identify the bottleneck, add the right index, prove improvement.

---

## Stage 4 — Server-Side Code and Triggers (3–5 days)

1. [[08 - Stored Procedures and Functions]]
2. [[09 - Triggers]]

---

## Stage 5 — Operations and .NET Integration (1 week)

**Goal**: Connect a real .NET app to PostgreSQL and ship it.

1. [[13 - Connection Management]]
2. [[14 - ADO.NET and Dapper]]
3. [[12 - Database Migrations]]
4. [[15 - Backup and Restore]]
5. [[16 - Database Security]]
6. [[17 - Encryption]]

**Checkpoint**: A .NET API backed by Postgres with EF Core migrations, role-based access, automated backup script, and TLS in transit.

---

## Stage 6 — Specialized Data (3–5 days)

1. [[18 - JSON and JSONB]]
2. [[19 - Full-Text Search]]
3. [[20 - NoSQL Fundamentals]]

---

## Stage 7 — Distribution and Scaling (1–2 weeks)

**Goal**: Understand the tradeoffs when one machine isn't enough.

1. [[02 - Replication]] ← read first
2. [[01 - Sharding and Partitioning]]
3. [[03 - High Availability and Failover]]
4. [[04 - CAP and PACELC]]

---

## Stage 8 — Distributed Patterns (1 week)

1. [[05 - Distributed Transactions]]
2. [[06 - CQRS and Event Sourcing]]
3. [[07 - Temporal and Audit Tables]]
4. [[08 - Schema Evolution]]

---

## Stage 9 — Production Performance (3–5 days)

1. [[09 - Performance Tuning]]
2. [[10 - Deadlocks and Blocking]]
3. [[11 - Bulk Operations]]

---

## Stage 10 — Analytics and Pipelines (1 week)

1. [[12 - Data Warehousing]]
2. [[13 - ETL and CDC]]

---

## Stage 11 — Specialized Stores (1 week)

1. [[14 - Time-Series Databases]]
2. [[15 - Graph Databases]]
3. [[16 - Search Engines]]
4. [[17 - Cloud Databases]]
5. [[18 - Vector Databases]]

---

## Tips for using this path

- **Don't skip Stage 2 and 3** — most production database bugs are isolation-level or missing-index issues.
- Spin up a local Postgres in Docker and run every example. Reading without executing doesn't stick.
- Use `EXPLAIN ANALYZE` constantly — develop intuition for what slow looks like.
- Revisit earlier notes as you advance — concepts deepen with experience.
- Use [[Database Master Index]] to jump around; this path is recommended, not mandatory.
