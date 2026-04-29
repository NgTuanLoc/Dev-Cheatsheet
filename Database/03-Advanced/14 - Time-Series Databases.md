---
tags: [database, advanced, postgresql, performance]
aliases: [TimescaleDB, InfluxDB, Hypertable, Continuous Aggregate]
level: Advanced
---

# Time-Series Databases

> **One-liner**: Time-series workloads are append-only, time-ordered, and overwhelmingly aggregated — purpose-built engines (TimescaleDB, InfluxDB) win over plain RDBMS only at scale.

---

## Quick Reference

| Engine | Type | Notes |
|--------|------|-------|
| **TimescaleDB** | Postgres extension | full SQL + hypertables + continuous aggregates |
| **InfluxDB** | purpose-built | Flux/InfluxQL; tight monitoring focus |
| **Prometheus** | metric DB + scraper | pull-based; not for long-term storage |
| **VictoriaMetrics** | Prometheus-compatible store | very fast, durable |
| **QuestDB / ClickHouse** | columnar; great for time-series + analytics | SQL-based |
| **Postgres + partitioning** | works fine up to ~tens of millions of points / table | start here |

| Concept | Meaning |
|---------|---------|
| **Hypertable** (Timescale) | logical table over many time-partitioned chunks |
| **Chunk** | one partition (e.g., one day's data) |
| **Retention policy** | auto-drop old chunks |
| **Continuous aggregate** | incrementally maintained materialized view |
| **Downsampling** | replace raw points with rolled-up averages over windows |
| **Compression** | columnar + delta encoding for old chunks |

---

## Core Concept

A time-series workload looks like:
- Data is **append-only**, sorted by time
- The vast majority of points are **never read raw** — they're aggregated (avg, min/max, p99) over windows
- Old data is **truncated** or **downsampled** rather than mutated
- Cardinality (distinct (metric, label) combinations) explodes if not designed carefully

Plain Postgres handles this up to medium scale (tens of millions of points/table) with native partitioning + indexes. **TimescaleDB** is a Postgres *extension* that adds:

- **Hypertables** — auto-partitioned by time (and optionally space)
- **Continuous aggregates** — materialized views that incrementally update as new data arrives
- **Compression** — convert old chunks to columnar+delta-encoded storage (10–100× smaller)
- **Retention policies** — auto-drop chunks older than X
- All while keeping plain SQL, joins to relational tables, and the Postgres ecosystem

For pure metrics monitoring, **Prometheus + VictoriaMetrics** is the standard. For application-side time-series with rich joins, TimescaleDB is the sweet spot. For pure billion-row analytics, ClickHouse.

---

## Syntax & API

### Plain Postgres + partitioning (start here)
```sql
CREATE TABLE measurements (
    sensor_id   INT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    value       NUMERIC NOT NULL,
    PRIMARY KEY (sensor_id, occurred_at)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE measurements_2026_q2 PARTITION OF measurements
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX ON measurements (sensor_id, occurred_at DESC);

-- pg_partman handles auto-creation + retention; see [[01 - Sharding and Partitioning]]
```

### TimescaleDB hypertable
```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE measurements (
    sensor_id   INT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    value       NUMERIC NOT NULL
);

-- Convert into a hypertable; chunks default to 7-day intervals
SELECT create_hypertable('measurements', 'occurred_at',
    chunk_time_interval => INTERVAL '1 day');

-- Insert rows like a normal table
INSERT INTO measurements VALUES (1, now(), 23.5);

-- Query like a normal table
SELECT date_trunc('hour', occurred_at) AS hour,
       AVG(value) AS avg_v
FROM measurements
WHERE sensor_id = 1
  AND occurred_at > now() - INTERVAL '7 days'
GROUP BY 1
ORDER BY 1;
```

### Continuous aggregate (auto-refreshed materialized view)
```sql
CREATE MATERIALIZED VIEW measurements_hourly
WITH (timescaledb.continuous) AS
SELECT sensor_id,
       time_bucket(INTERVAL '1 hour', occurred_at) AS bucket,
       AVG(value) AS avg_v,
       MIN(value) AS min_v,
       MAX(value) AS max_v,
       COUNT(*)   AS n
FROM measurements
GROUP BY sensor_id, bucket;

-- Refresh policy: keep up to 1 hour ago, refresh every 10 minutes
SELECT add_continuous_aggregate_policy('measurements_hourly',
    start_offset => INTERVAL '1 day',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '10 minutes');

-- Query the aggregate (instant)
SELECT * FROM measurements_hourly
WHERE sensor_id = 1
  AND bucket > now() - INTERVAL '7 days';
```

### Compression
```sql
-- Enable on old chunks
ALTER TABLE measurements SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',
    timescaledb.compress_orderby   = 'occurred_at DESC'
);

-- Auto-compress chunks older than 7 days
SELECT add_compression_policy('measurements', INTERVAL '7 days');
```

### Retention policy
```sql
-- Drop chunks older than 1 year
SELECT add_retention_policy('measurements', INTERVAL '1 year');
```

### Useful functions
```sql
-- time_bucket: like date_trunc but with arbitrary intervals
SELECT time_bucket(INTERVAL '5 minutes', occurred_at) AS bucket,
       AVG(value)
FROM measurements
WHERE occurred_at > now() - INTERVAL '1 hour'
GROUP BY bucket
ORDER BY bucket;

-- last(value, occurred_at) — last value within group
SELECT sensor_id, last(value, occurred_at) AS latest
FROM measurements
GROUP BY sensor_id;

-- approximate percentiles (Timescale toolkit)
SELECT approx_percentile(0.95, percentile_agg(value)) AS p95
FROM measurements
WHERE occurred_at > now() - INTERVAL '1 day';
```

### InfluxDB (line protocol + Flux)
```text
# Line protocol — push to /api/v2/write
cpu,host=server-1,region=us-east usage=23.5,load=1.2 1714400000000000000
```

```javascript
// Flux query
from(bucket: "metrics")
    |> range(start: -1h)
    |> filter(fn: (r) => r._measurement == "cpu" and r.host == "server-1")
    |> aggregateWindow(every: 1m, fn: mean)
```

### Prometheus PromQL (read-only example)
```text
# Average CPU usage per host over last 5 minutes
avg_over_time(cpu_usage{host=~"web-.*"}[5m])

# 95th percentile request latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

---

## Common Patterns

```sql
-- Pattern: hypertable + relational dimensions
-- Big data lives in hypertable; sensor metadata stays in normal Postgres tables
CREATE TABLE sensors (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    TEXT NOT NULL,
    location TEXT
);

SELECT s.name, AVG(m.value)
FROM measurements m JOIN sensors s ON s.id = m.sensor_id
WHERE m.occurred_at > now() - INTERVAL '1 hour'
GROUP BY s.name;
-- TimescaleDB joins hypertables with regular tables seamlessly
```

```sql
-- Pattern: hot+cold tiers with continuous aggregates
-- Raw points kept for 7 days (hot)
SELECT add_retention_policy('measurements', INTERVAL '7 days');

-- Hourly aggregates kept forever (cold)
-- (continuous aggregate has its own retention)
```

```sql
-- Pattern: real-time + historical via single SQL
SELECT * FROM measurements_hourly WHERE bucket >= now() - INTERVAL '7 days'
UNION ALL
SELECT sensor_id, time_bucket(INTERVAL '1 hour', occurred_at) AS bucket,
       AVG(value), MIN(value), MAX(value), COUNT(*)
FROM measurements WHERE occurred_at > now() - INTERVAL '1 hour'
GROUP BY 1, 2;
```

---

## Gotchas & Tips

- **High cardinality kills time-series engines** — InfluxDB and Prometheus both struggle when (metric × labels) explodes. Beware unbounded labels (user_id, request_id).
- **Don't store huge text in time-series** — values should be numeric. For events, log differently.
- **Index design** — `(sensor_id, occurred_at DESC)` is the canonical pattern; queries filter by ID and order by time.
- **Compression is read-only** — once a chunk is compressed in Timescale, you can't update individual rows in it (decompress first; expensive).
- **Continuous aggregates must be incrementally refreshable** — every aggregate must be associative. AVG works, percentile_disc doesn't (use approx_percentile).
- **Hypertable chunks shouldn't be too small** — too many chunks slow planning. Aim for chunks 50MB–1GB each.
- **Drop chunks for retention, not DELETE** — `DROP TABLE` is O(1) per chunk; `DELETE FROM measurements WHERE …` is O(N) and bloats.
- **Don't use Prometheus for "real" data** — it's a metric scraper with limited durability. Pair with VictoriaMetrics or Mimir for long-term.
- **Backups need awareness** — TimescaleDB chunks make `pg_dump` huge but it works. `pg_basebackup` + WAL is faster.
- **Cardinality budget** — write a query: `SELECT count(*) FROM (SELECT DISTINCT sensor_id, label1, label2 FROM measurements) t;`. If it's growing without bound, redesign.
- **Don't normalize timestamps to LOCAL** — store as `TIMESTAMPTZ`. Local-time-of-day queries can be derived.

---

## See Also

- [[01 - Sharding and Partitioning]]
- [[07 - Views]]
- [[12 - Data Warehousing]]
- [[09 - Performance Tuning]]
