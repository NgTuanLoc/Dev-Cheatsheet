---
tags: [database, advanced, postgresql, performance, query-optimization]
aliases: [pg_stat_statements, Autovacuum, Bloat, Plan Cache]
level: Advanced
---

# Performance Tuning

> **One-liner**: Tuning is a loop — measure with `pg_stat_statements`, fix the worst query (usually with an index or a rewrite), tune autovacuum and configuration only when measurement justifies it.

---

## Quick Reference

| Tool | What it tells you |
|------|-------------------|
| `pg_stat_statements` | top queries by time/calls/rows |
| `pg_stat_user_tables` | per-table read/write/scan/vacuum stats |
| `pg_stat_user_indexes` | per-index usage (or non-usage) |
| `pg_stat_activity` | running queries, locks, waits |
| `pg_locks` | who's blocking whom |
| `EXPLAIN (ANALYZE, BUFFERS)` | one query's actual plan |
| `auto_explain` | log slow queries' plans automatically |
| `pg_buffercache` | what's hot in shared buffers |
| `pgBadger` | log analyzer (built from CSV logs) |
| `pgmustard / explain.dalibo.com` | plan visualizers |

| Key config | Typical value |
|------------|---------------|
| `shared_buffers` | 25% of RAM (cap ~16GB) |
| `effective_cache_size` | 50–75% of RAM (planner hint) |
| `work_mem` | 16–64 MB per operation per connection |
| `maintenance_work_mem` | 256 MB – 2 GB for VACUUM, CREATE INDEX |
| `max_connections` | low (use a pooler), e.g. 100–200 |
| `random_page_cost` | 1.1 on SSD (default 4 assumes spinning disks) |
| `effective_io_concurrency` | 200 on SSD/NVMe |
| `default_statistics_target` | 100 default; raise to 1000 for skewed columns |
| `autovacuum_vacuum_scale_factor` | 0.2 default; lower (0.05) for big tables |

---

## Core Concept

Performance work is a hierarchy. Spend time on each level proportional to its impact:

1. **Query / index** — 90% of wins. Find the worst query, profile, add an index or rewrite.
2. **Schema** — table layout, partitioning, normalization choices.
3. **Autovacuum / statistics** — bad stats give bad plans; bloated tables run slow.
4. **Configuration** — `shared_buffers`, `work_mem`, `random_page_cost`. Do this *after* the first three; the defaults rarely cause real-world problems if the basics are right.
5. **Hardware / OS** — last resort. Most "we need bigger box" requests dissolve under proper indexing.

The loop: identify slowest queries (`pg_stat_statements`) → `EXPLAIN ANALYZE` → diagnose → fix → measure again. Without a baseline, you're guessing.

---

## Syntax & API

### Top-N slow queries
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- shared_preload_libraries = 'pg_stat_statements'  in postgresql.conf

SELECT
    LEFT(query, 100) AS query,
    calls,
    ROUND(total_exec_time::NUMERIC, 0)              AS total_ms,
    ROUND(mean_exec_time::NUMERIC, 2)               AS mean_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::NUMERIC, 1) AS pct,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Reset stats periodically (after a fix)
```sql
SELECT pg_stat_statements_reset();
SELECT pg_stat_reset();         -- table/index stats
```

### auto_explain — log slow query plans
```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze      = on
auto_explain.log_buffers      = on
```

Now anything > 500ms gets its full plan in the log.

### Find unused indexes (drop candidates)
```sql
SELECT s.schemaname, s.relname AS table, s.indexrelname AS index,
       pg_size_pretty(pg_relation_size(i.indexrelid)) AS size,
       s.idx_scan
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0 AND NOT i.indisunique AND NOT i.indisprimary
ORDER BY pg_relation_size(i.indexrelid) DESC;
```

### Find missing indexes (sequential scans on big tables)
```sql
SELECT relname, seq_scan, seq_tup_read, idx_scan,
       pg_size_pretty(pg_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND pg_relation_size(relid) > 100*1024*1024   -- > 100MB
ORDER BY seq_tup_read DESC;
```

### Bloat estimation
```sql
-- Quick approximation
SELECT relname,
       n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;
```

### Autovacuum tuning per table
```sql
-- High-churn table → vacuum more aggressively
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,    -- 5% of rows dead → vacuum
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_vacuum_cost_limit = 1000       -- speed it up
);
```

### Manual vacuum / analyze
```sql
VACUUM ANALYZE orders;
VACUUM (VERBOSE, ANALYZE) orders;
VACUUM (FULL) orders;     -- reclaims disk; takes ACCESS EXCLUSIVE — danger!
```

### Active queries / waits
```sql
SELECT pid, now() - query_start AS run_for, state, wait_event_type, wait_event,
       LEFT(query, 80) AS q
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Find a long-running idle-in-transaction holder of locks
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND xact_start < now() - INTERVAL '1 minute';
```

### Cache hit ratio
```sql
SELECT relname,
       100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0) AS hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC LIMIT 10;
-- Hit ratio < ~99% on a warm cache hints at cold tables or undersized shared_buffers
```

### Useful settings
```sql
-- Per-session work_mem boost for a heavy report
SET work_mem = '256MB';

-- Set per-statement timeout
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SET idle_in_transaction_session_timeout = '60s';
```

---

## Common Patterns

```sql
-- Pattern: find the most-blocked queries
SELECT bl.pid AS blocked_pid, bl.query AS blocked_query,
       kl.pid AS blocking_pid, kl.query AS blocking_query,
       now() - bl.query_start AS waited_for
FROM pg_stat_activity bl
JOIN pg_stat_activity kl ON kl.pid = ANY(pg_blocking_pids(bl.pid));
```

```sql
-- Pattern: index advisor proxy (HypoPG — hypothetical indexes)
CREATE EXTENSION hypopg;
SELECT hypopg_create_index('CREATE INDEX ON orders (user_id, placed_at)');
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY placed_at DESC LIMIT 10;
SELECT hypopg_reset();
-- Test plans without paying real index build/storage cost
```

```sql
-- Pattern: refresh stats aggressively for skewed columns
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
-- Helps the planner pick better with rare values
```

```text
Pattern: queue depth & throughput SLO
- Latency p99 (per query) and queue depth (active queries) are the two metrics that matter
- p99 spike → individual slow query (find with pg_stat_statements)
- depth spike → contention (locks) or under-provisioned pool/CPU
```

---

## Gotchas & Tips

- **`SELECT *` is the most overlooked perf bug** — wider columns, no covering index applies, planner can't push down projection.
- **Cold caches lie** — first run after restart is slow. `EXPLAIN (ANALYZE, BUFFERS)` shows `read` vs `hit`; warm before benchmarking.
- **`work_mem` is *per operation*, not per session** — a single query can use multiples (per sort, per hash). Don't crank it cluster-wide.
- **Long idle-in-transaction is the #1 hidden problem** — holds locks, prevents vacuuming. `idle_in_transaction_session_timeout` saves you.
- **`VACUUM FULL` is rarely the answer** — it takes `ACCESS EXCLUSIVE`. Use `pg_repack` for online cleanup.
- **`CREATE INDEX` (non-concurrent) blocks writes** — always `CONCURRENTLY` in production.
- **Plan-cache surprises** — prepared statements may pick a generic plan that's worse than parameter-aware. `plan_cache_mode = force_custom_plan` for problem queries.
- **Connection pooling > raising `max_connections`** — Postgres performs poorly past ~200 backends. Use PgBouncer.
- **Replication lag from heavy reads** — long queries on replicas pause WAL replay. Use `hot_standby_feedback = on` and tune `max_standby_streaming_delay`.
- **Autovacuum is your friend** — never disable it. Tune per table for hot ones.
- **Index rebuild after big DELETEs** — `REINDEX INDEX CONCURRENTLY` reclaims space.
- **Track `pg_stat_io` (PG 16+)** — granular I/O stats by backend type and context.

---

## See Also

- [[06 - Query Optimization]]
- [[05 - Indexes Advanced]]
- [[10 - Deadlocks and Blocking]]
- [[01 - Sharding and Partitioning]]
