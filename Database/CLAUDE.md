# CLAUDE.md — Database Topic

Topic-specific guidance for the `Database/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured database teacher-authored cheatsheet — organized from beginner to advanced, intended for students learning relational databases (primarily **PostgreSQL**) and adjacent NoSQL/distributed-database concepts.

---

## SQL Dialect Policy

**PostgreSQL is the primary dialect.** All SQL examples are PostgreSQL-flavored unless explicitly noted. When a topic differs significantly across engines (SQL Server, MySQL, SQLite), call it out in a callout sub-section:

````markdown
### Dialect notes

- **PostgreSQL**: `RETURNING`, `JSONB`, `LATERAL`, `ON CONFLICT`
- **SQL Server**: uses `OUTPUT`, `JSON` is text-only, no `LATERAL` (use APPLY)
- **MySQL**: limited CTE recursion, no `RETURNING` until 10.5+ (MariaDB)
````

When showing engine-specific syntax, use the appropriate code-fence language tag (`sql` is the default; `psql` only for psql meta-commands like `\d`, `\dt`).

---

## Folder Structure

```
Database/
├── CLAUDE.md                    ← this file
├── 00-Index/
│   ├── Master Index.md          ← single entry point, links to all DB notes
│   └── Learning Path.md         ← recommended reading order per level
├── 01-Beginner/                 (10 notes)
├── 02-Intermediate/             (20 notes)
└── 03-Advanced/                 (18 notes)
```

---

## Tag Taxonomy (Database-specific)

Always include `database` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `database` | applied to every database note |
| `sql` | SQL language topics |
| `postgresql` | Postgres-specific feature or syntax |
| `mysql` / `sqlserver` / `sqlite` | engine-specific call-outs |
| `nosql` | document, key-value, column, graph stores |
| `mongodb` / `redis` / `cassandra` / `neo4j` / `elasticsearch` | NoSQL specifics |
| `transactions` | ACID, isolation, locking |
| `indexing` | indexes, query plans, statistics |
| `query-optimization` | execution plans, sargable, tuning |
| `schema-design` | normalization, modeling, constraints |
| `migrations` | schema evolution, EF Core, Flyway, DbUp |
| `replication` | streaming, logical, physical |
| `sharding` | partitioning, distribution |
| `availability` | HA, failover, recovery |
| `security` | auth, encryption, RLS |
| `performance` | tuning, profiling |
| `cloud` | Azure SQL, RDS, Aurora, Cosmos DB |
| `data-warehousing` | OLAP, star schema, ETL |
| `dotnet` | .NET data-access tie-ins (ADO.NET, Dapper, EF Core) |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `sql` | all SQL (default — PostgreSQL flavor unless noted) |
| `psql` | psql meta-commands only (`\d`, `\dt`, `\timing`) |
| `bash` | CLI: `pg_dump`, `pg_restore`, `psql`, Docker, EF tooling |
| `json` | JSONB sample documents, MongoDB queries |
| `csharp` | .NET data-access examples (ADO.NET, Dapper, EF Core) |
| `yaml` | docker-compose for local DB, Kubernetes manifests |
| `mermaid` | diagrams |

Examples must be **runnable** — student should be able to paste into `psql` or a fresh database and execute.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Database Overview | RDBMS vs NoSQL, ACID vs BASE, OLTP vs OLAP, common engines (PostgreSQL, MySQL, SQL Server, SQLite, Mongo, Redis) |
| 02 | SQL Fundamentals | SELECT, INSERT, UPDATE, DELETE, WHERE, ORDER BY, LIMIT/OFFSET, DISTINCT |
| 03 | Data Types | INT/BIGINT, NUMERIC/DECIMAL, TEXT/VARCHAR, DATE/TIMESTAMP, BOOLEAN, UUID, JSONB; choosing the right type |
| 04 | Schema Design Basics | Tables, columns, rows, naming conventions, surrogate vs natural keys |
| 05 | Keys and Relationships | Primary key, foreign key, composite keys, 1:1, 1:N, N:N (junction tables) |
| 06 | Joins | INNER, LEFT, RIGHT, FULL OUTER, CROSS, SELF; visual explanation |
| 07 | Aggregations and Grouping | COUNT, SUM, AVG, MIN, MAX, GROUP BY, HAVING vs WHERE |
| 08 | Subqueries and CTEs | Scalar/correlated subqueries, EXISTS, IN, WITH clauses, recursive CTEs |
| 09 | Constraints | NOT NULL, UNIQUE, CHECK, DEFAULT, FK with ON DELETE/UPDATE |
| 10 | Indexes Basics | What an index is, B-tree default, when indexes help vs hurt |

### Intermediate (02-Intermediate) — 21 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Normalization | 1NF, 2NF, 3NF, BCNF, denormalization tradeoffs |
| 02 | Transactions and ACID | BEGIN/COMMIT/ROLLBACK, savepoints, atomicity guarantees |
| 03 | Isolation Levels | Read Committed (Postgres default), Repeatable Read, Serializable; phenomena |
| 04 | Locking and Concurrency | Pessimistic vs optimistic, row/table locks, FOR UPDATE/SHARE, deadlocks |
| 05 | Indexes Advanced | Composite, covering, partial, expression, GIN/GiST/BRIN, fragmentation |
| 06 | Query Optimization | EXPLAIN/EXPLAIN ANALYZE, statistics, sargable predicates, anti-patterns |
| 07 | Views | Regular views, materialized views, REFRESH MATERIALIZED VIEW CONCURRENTLY |
| 08 | Stored Procedures and Functions | PL/pgSQL functions, procedures, parameters, return types |
| 09 | Triggers | BEFORE/AFTER, statement vs row, when to avoid, audit-trail patterns |
| 10 | Window Functions | ROW_NUMBER, RANK, DENSE_RANK, LAG/LEAD, PARTITION BY, running totals |
| 11 | Set Operations and Pivoting | UNION/UNION ALL, INTERSECT, EXCEPT, crosstab |
| 12 | Database Migrations | EF Core migrations, FluentMigrator, DbUp, Flyway, idempotent scripts |
| 13 | Connection Management | Connection strings, pooling (PgBouncer, Npgsql pool), async I/O, leaks |
| 14 | ADO.NET and Dapper | Npgsql, parameterized queries, Dapper micro-ORM patterns |
| 15 | Backup and Restore | pg_dump, pg_restore, base backups, PITR, WAL archiving, RTO vs RPO |
| 16 | Database Security | Roles, GRANT/REVOKE, row-level security (RLS), least privilege |
| 17 | Encryption | TLS in transit, pgcrypto, column-level encryption, secrets handling |
| 18 | JSON and JSONB | jsonb operators, indexing JSONB with GIN, when to use vs separate table |
| 19 | Full-Text Search | tsvector, tsquery, GIN index, ranking, language config |
| 20 | NoSQL Fundamentals | Document (Mongo), key-value (Redis), wide-column (Cassandra), graph (Neo4j) |
| 21 | B-Tree Internals | B+tree structure, sargability, prefix vs leading-wildcard `LIKE`, function-wrapped columns, implicit-cast traps, `text_pattern_ops`, fixes (`pg_trgm`, FTS, reverse-string) |

### Advanced (03-Advanced) — 18 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Sharding and Partitioning | Postgres declarative partitioning, hash/range/list, partition pruning, Citus |
| 02 | Replication | Streaming replication, logical replication, publication/subscription, lag |
| 03 | High Availability and Failover | Patroni, repmgr, automatic failover, quorum, split-brain |
| 04 | CAP and PACELC | Consistency vs availability vs partition tolerance; latency tradeoffs |
| 05 | Distributed Transactions | 2PC, Saga (orchestration vs choreography), Outbox pattern, idempotency |
| 06 | CQRS and Event Sourcing | Read/write model split, event store, projections, replay, snapshots |
| 07 | Temporal and Audit Tables | System-versioned patterns, history tables, point-in-time queries |
| 08 | Schema Evolution | Zero-downtime migrations, expand/contract, dual-write, backfills |
| 09 | Performance Tuning | pg_stat_statements, plan regression, parameter sniffing, autovacuum |
| 10 | Deadlocks and Blocking | Detection (pg_locks, pg_stat_activity), retry strategies |
| 11 | Bulk Operations | COPY, batched INSERTs, ON CONFLICT, MERGE (PG 15+) |
| 12 | Data Warehousing | OLTP vs OLAP, star and snowflake schema, fact/dimension, SCD |
| 13 | ETL and CDC | Extract/Transform/Load, change data capture, Debezium, logical decoding |
| 14 | Time-Series Databases | TimescaleDB, hypertables, retention/downsampling, continuous aggregates |
| 15 | Graph Databases | Neo4j, Cypher basics, traversals, when graph beats relational |
| 16 | Search Engines | Elasticsearch/OpenSearch, inverted indexes, analyzers, relevance scoring |
| 17 | Cloud Databases | Azure Database for PostgreSQL, AWS RDS/Aurora, Cosmos DB, managed-service tradeoffs |
| 18 | Vector Databases | pgvector, Pinecone, Qdrant; embeddings, similarity search, RAG patterns |

**Total: 49 notes** (10 beginner + 21 intermediate + 18 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| Database Overview | graph (engine taxonomy + when-to-use map) |
| Joins | graph LR (Venn-style or set diagram) |
| Subqueries and CTEs | flowchart (recursive CTE traversal) |
| Indexes Basics | graph TD (B-tree structure) |
| Transactions and ACID | stateDiagram-v2 (transaction lifecycle) |
| Isolation Levels | flowchart (decision tree per phenomenon) |
| Locking and Concurrency | sequenceDiagram (deadlock scenario) |
| Query Optimization | flowchart (EXPLAIN plan reading) |
| Window Functions | flowchart (PARTITION BY visualization) |
| NoSQL Fundamentals | graph (taxonomy + use-case fit) |
| Sharding and Partitioning | graph (partition strategies) |
| Replication | graph (topology) |
| High Availability and Failover | sequenceDiagram (failover handshake) |
| Distributed Transactions | sequenceDiagram (Saga + Outbox) |
| CQRS and Event Sourcing | graph (read/write split + projections) |
| ETL and CDC | sequenceDiagram (CDC pipeline) |
| Data Warehousing | erDiagram (star schema) |
| Vector Databases | graph (embedding + similarity search) |
