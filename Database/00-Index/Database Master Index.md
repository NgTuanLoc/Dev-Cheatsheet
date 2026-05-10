---
tags: [database, index]
aliases: [Database Index, DB Home]
---

# Database Master Index — Database Cheatsheet

> **Single entry point** for the Database topic. Browse by level or jump straight to a topic. Primary dialect: **PostgreSQL**.

---

## Start Here

- [[Database Learning Path]] — recommended reading order
- New to databases? Begin with [[01 - Database Overview]]

---

## 01 — Beginner (foundations)

Database fundamentals and core SQL. Master these before moving on.

| # | Note | What you'll learn |
|---|------|-------------------|
| 01 | [[01 - Database Overview]] | RDBMS vs NoSQL, ACID, OLTP/OLAP, engine taxonomy |
| 02 | [[02 - SQL Fundamentals]] | SELECT, INSERT, UPDATE, DELETE, WHERE, ORDER BY |
| 03 | [[03 - Data Types]] | INT, NUMERIC, TEXT, TIMESTAMP, UUID, JSONB |
| 04 | [[04 - Schema Design Basics]] | Tables, naming, surrogate vs natural keys |
| 05 | [[05 - Keys and Relationships]] | PK, FK, composite, 1:1, 1:N, N:N |
| 06 | [[06 - Joins]] | INNER, LEFT, RIGHT, FULL, CROSS, SELF |
| 07 | [[07 - Aggregations and Grouping]] | COUNT/SUM/AVG, GROUP BY, HAVING |
| 08 | [[08 - Subqueries and CTEs]] | Subqueries, EXISTS, WITH, recursive CTEs |
| 09 | [[09 - Constraints]] | NOT NULL, UNIQUE, CHECK, DEFAULT, FK actions |
| 10 | [[10 - Indexes Basics]] | B-tree, when indexes help vs hurt |

---

## 02 — Intermediate (real-world data work)

Schema design, query tuning, transactions, .NET integration.

### Schema and modeling
| # | Note |
|---|------|
| 01 | [[01 - Normalization]] |
| 04 | [[04 - Locking and Concurrency]] |
| 11 | [[11 - Set Operations and Pivoting]] |

### Transactions and concurrency
| # | Note |
|---|------|
| 02 | [[02 - Transactions and ACID]] |
| 03 | [[03 - Isolation Levels]] |
| 04 | [[04 - Locking and Concurrency]] |

### Performance and indexing
| # | Note |
|---|------|
| 05 | [[05 - Indexes Advanced]] |
| 06 | [[06 - Query Optimization]] |
| 07 | [[07 - Views]] |
| 10 | [[10 - Window Functions]] |
| 21 | [[21 - B-Tree Internals]] |

### Server-side code
| # | Note |
|---|------|
| 08 | [[08 - Stored Procedures and Functions]] |
| 09 | [[09 - Triggers]] |

### Operations and integration
| # | Note |
|---|------|
| 12 | [[12 - Database Migrations]] |
| 13 | [[13 - Connection Management]] |
| 14 | [[14 - ADO.NET and Dapper]] |
| 15 | [[15 - Backup and Restore]] |
| 16 | [[16 - Database Security]] |
| 17 | [[17 - Encryption]] |

### Specialized data
| # | Note |
|---|------|
| 18 | [[18 - JSON and JSONB]] |
| 19 | [[19 - Full-Text Search]] |
| 20 | [[20 - NoSQL Fundamentals]] |

---

## 03 — Advanced (scale, distribute, observe)

Production-grade concerns: distribution, performance tuning, schema evolution, cloud, AI.

### Distribution and scaling
| # | Note |
|---|------|
| 01 | [[01 - Sharding and Partitioning]] |
| 02 | [[02 - Replication]] |
| 03 | [[03 - High Availability and Failover]] |
| 04 | [[04 - CAP and PACELC]] |

### Distributed patterns
| # | Note |
|---|------|
| 05 | [[05 - Distributed Transactions]] |
| 06 | [[06 - CQRS and Event Sourcing]] |
| 07 | [[07 - Temporal and Audit Tables]] |
| 08 | [[08 - Schema Evolution]] |

### Performance and operations
| # | Note |
|---|------|
| 09 | [[09 - Performance Tuning]] |
| 10 | [[10 - Deadlocks and Blocking]] |
| 11 | [[11 - Bulk Operations]] |

### Analytics and pipelines
| # | Note |
|---|------|
| 12 | [[12 - Data Warehousing]] |
| 13 | [[13 - ETL and CDC]] |

### Specialized stores
| # | Note |
|---|------|
| 14 | [[14 - Time-Series Databases]] |
| 15 | [[15 - Graph Databases]] |
| 16 | [[16 - Search Engines]] |
| 17 | [[17 - Cloud Databases]] |
| 18 | [[18 - Vector Databases]] |

---

## Browse by tag

Use Obsidian's tag pane to filter:

- `#postgresql` — Postgres-specific features
- `#sql` — pure SQL language topics
- `#nosql` — document, key-value, column, graph stores
- `#transactions` — ACID, isolation, locking
- `#indexing` `#query-optimization` — performance
- `#schema-design` `#migrations` — modeling and evolution
- `#replication` `#sharding` `#availability` — distribution
- `#security` — auth, encryption, RLS
- `#cloud` — managed-service specifics
- `#dotnet` — .NET data-access tie-ins
