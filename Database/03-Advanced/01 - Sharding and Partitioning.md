---
tags: [database, advanced, postgresql, sharding, performance]
aliases: [Partitioning, Declarative Partitioning, Citus, Horizontal Scaling]
level: Advanced
---

# Sharding and Partitioning

> **One-liner**: Partitioning splits a table into pieces *within* one server; sharding splits it *across* servers. Both reduce per-query work; both add operational cost.

---

## Quick Reference

| Concept | Postgres |
|---------|----------|
| **Partitioning** | one table, multiple physical sub-tables, one server |
| **Sharding** | data spread across multiple servers (extension: Citus) |
| Partition strategies | `RANGE`, `LIST`, `HASH` |
| Partition pruning | planner skips partitions that can't match — automatic with right `WHERE` |
| Default partition | catches values not in any partition |
| Detach / attach | live partition swap (`ALTER TABLE … DETACH/ATTACH PARTITION`) |
| Citus | distributed Postgres extension; `CREATE TABLE … DISTRIBUTED BY` |
| Vitess / pg_partman / Foreign Tables | other sharding/partition tooling |

| Strategy | Best for |
|----------|----------|
| **RANGE** | time-series (one partition per month) |
| **LIST** | discrete values (per-region, per-tenant) |
| **HASH** | even distribution when no natural key (Citus default) |

---

## Core Concept

**Partitioning** divides a logical table into physical chunks. Postgres's declarative partitioning (PG 10+) presents one parent table; reads/writes route to the right child based on the partition key. Benefits:

- **Pruning** — `WHERE created_at > '2026-01-01'` only touches recent partitions
- **Cheap drops** — `DETACH PARTITION events_2020` is O(1) vs `DELETE`
- **Smaller indexes** per partition (faster, easier to maintain)
- **Parallel maintenance** — VACUUM, autovacuum per partition

**Sharding** is partitioning across machines. Postgres core doesn't shard, but:

- **Citus** (extension) — distributes tables across worker nodes; coordinator routes queries
- **Foreign Data Wrappers** + custom routing — manual setup, more control
- App-level sharding — your code picks the shard

Sharding is a major step. It complicates joins (cross-shard = expensive or impossible), transactions, and operations. Reach for it when:
- One server can no longer hold the data
- One server can no longer absorb the writes
- Tail-latency on the largest table dominates

Until then, partition + scale up.

---

## Syntax & API

### Range partitioning by time
```sql
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id     INT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    payload     JSONB,
    PRIMARY KEY (id, occurred_at)        -- partition key MUST be in PK
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_q1 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE events_2026_q2 PARTITION OF events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Indexes on the parent are propagated to existing & future partitions (PG 11+)
CREATE INDEX ON events (user_id);
```

### List partitioning by region
```sql
CREATE TABLE orders (
    id      INT GENERATED ALWAYS AS IDENTITY,
    region  TEXT NOT NULL,
    total   NUMERIC,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US', 'CA', 'MX');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('DE', 'FR', 'UK');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

### Hash partitioning for even distribution
```sql
CREATE TABLE sessions (
    user_id     INT NOT NULL,
    session_id  UUID NOT NULL,
    started_at  TIMESTAMPTZ,
    PRIMARY KEY (user_id, session_id)
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_p0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_p1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_p2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_p3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Drop old data instantly
```sql
-- Compared to DELETE FROM events WHERE occurred_at < '2024-01-01'
ALTER TABLE events DETACH PARTITION events_2023_q4;
DROP TABLE events_2023_q4;
-- O(1) metadata change + table drop. No row-by-row deletion.
```

### Verify pruning in EXPLAIN
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events
WHERE occurred_at >= '2026-04-01' AND occurred_at < '2026-05-01';

-- Plan should mention only events_2026_q2 (other partitions pruned)
```

### pg_partman — auto-create partitions on schedule
```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table => 'public.events',
    p_control      => 'occurred_at',
    p_type         => 'native',
    p_interval     => 'monthly',
    p_premake      => 4
);

-- Run periodically (cron / pg_cron) to maintain partitions
SELECT partman.run_maintenance();
```

### Citus — distributed Postgres
```sql
CREATE EXTENSION citus;
SELECT citus_add_node('worker1', 5432);
SELECT citus_add_node('worker2', 5432);

CREATE TABLE orders (id INT, user_id INT NOT NULL, total NUMERIC, PRIMARY KEY (id, user_id));

-- Distribute by user_id; co-locate orders + users on the same shard
SELECT create_distributed_table('orders', 'user_id');
SELECT create_distributed_table('users', 'id', colocate_with => 'orders');

-- Reference table: replicated to every worker (small lookup tables)
CREATE TABLE products (id INT PRIMARY KEY, name TEXT);
SELECT create_reference_table('products');

-- Queries routing handled by coordinator
SELECT u.name, SUM(o.total)
FROM users u JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
```

---

## Common Patterns

```sql
-- Pattern: time-series with monthly partitions + retention
SELECT partman.run_maintenance();
-- Drops old partitions per `retention` setting on the parent.
```

```sql
-- Pattern: per-tenant LIST partition for hot tenants + DEFAULT for the rest
CREATE TABLE bills PARTITION BY LIST (tenant_id);
CREATE TABLE bills_t_acme  PARTITION OF bills FOR VALUES IN (1);
CREATE TABLE bills_t_globex PARTITION OF bills FOR VALUES IN (2);
CREATE TABLE bills_default PARTITION OF bills DEFAULT;
-- Hot tenants have their own files/indexes; rest pooled.
```

```sql
-- Pattern: bulk-load via attach
CREATE TABLE events_2026_q3_staging (LIKE events INCLUDING ALL);
COPY events_2026_q3_staging FROM '/tmp/q3.csv';
-- Add CHECK matching the partition bounds before attach (avoids re-scan)
ALTER TABLE events_2026_q3_staging ADD CONSTRAINT pk
    CHECK (occurred_at >= '2026-07-01' AND occurred_at < '2026-10-01');
ALTER TABLE events ATTACH PARTITION events_2026_q3_staging
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
```

---

## Gotchas & Tips

- **Partition key must be in the PK** — Postgres needs it to route inserts. Use composite PKs (`PRIMARY KEY (id, partition_key)`).
- **Pruning needs predicate on the partition key** — `WHERE id = ?` doesn't prune if you partition by date. Make the partition key part of every important query.
- **Foreign keys to a partitioned table need PG 12+** — earlier versions don't support FKs into partitioned tables.
- **Don't over-partition** — too many small partitions slow planning. Aim for partitions of 10M–100M rows.
- **Default partition is a footgun** — rows that don't fit any explicit partition land there. Then you can't add a new explicit partition that overlaps without first detaching default.
- **Indexes on partitioned tables must be created on the parent** (PG 11+) to apply to all children. `CREATE INDEX ON parent (col)` cascades.
- **Detach/Attach without `LOCK` issues** — `ALTER TABLE ... DETACH PARTITION ... CONCURRENTLY` (PG 14+) avoids `ACCESS EXCLUSIVE`.
- **Sharding ≠ partitioning** — partitioning is one-server; sharding is multi-server. Citus does both.
- **Cross-shard joins are slow** — Citus co-locates by distribution key; queries that touch multiple shards pay network costs.
- **Pre-create future partitions** — first INSERT into a non-existent partition fails. Use `pg_partman` or cron.
- **VACUUM per partition** is parallelizable — autovacuum handles them independently. Bigger total, smaller chunks.

---

## See Also

- [[02 - Replication]]
- [[09 - Performance Tuning]]
- [[14 - Time-Series Databases]]
- [[12 - Data Warehousing]]
