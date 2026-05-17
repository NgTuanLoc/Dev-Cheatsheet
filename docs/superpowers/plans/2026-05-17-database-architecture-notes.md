# Database Architecture Notes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three architecture deep-dive notes to the Database topic — one each for PostgreSQL, SQL Server, and MongoDB — explaining internal architecture, how each engine works, key differences/similarities across all three, and use cases. Wire them into the Master Index, CLAUDE.md coverage map, and Learning Path.

**Architecture:** Three sibling notes placed in `Database/02-Intermediate/` as files `22`, `23`, `24`. Each follows the canonical Note Template (`_Templates/Note Template.md`), targets the same depth/style as `21 - B-Tree Internals.md`, and contains a **Comparison Matrix** section that puts all three engines side-by-side using identical column order so a reader landing on any one note gets the cross-engine view. Cross-links via Obsidian wiki-links bind the three notes together.

**Tech Stack:** Markdown + YAML frontmatter, Mermaid (`graph`, `sequenceDiagram`, `flowchart`) for architecture diagrams, code-fences in `sql` / `bash` / `csharp` / `json` per the topic's [Code-Block Conventions](../../../Database/CLAUDE.md).

---

## File Structure

Files to **create**:

| Path | Responsibility |
|------|----------------|
| `Database/02-Intermediate/22 - PostgreSQL Architecture.md` | Postmaster + backend processes, shared memory, MVCC, WAL, vacuum, planner, replication topology, use cases, cross-engine comparison |
| `Database/02-Intermediate/23 - SQL Server Architecture.md` | SQLOS, thread scheduling, buffer pool + plan cache, pages/extents, transaction log, lock manager, Always On AGs, columnstore, use cases, cross-engine comparison |
| `Database/02-Intermediate/24 - MongoDB Architecture.md` | mongod, WiredTiger, BSON + collections, replica sets + oplog, sharded cluster (mongos + config servers), read/write concerns, transactions, use cases, cross-engine comparison |

Files to **modify**:

| Path | Change |
|------|--------|
| `Database/00-Index/Database Master Index.md` | Add 3 rows under "Specialized data" (or a new "Engine architectures" sub-section) linking the new notes |
| `Database/00-Index/Database Learning Path.md` | Insert a new "Stage 6.5 — Engine Internals" between Stage 6 and Stage 7, referencing the three notes |
| `Database/CLAUDE.md` | Update intermediate count `21 → 24`, append 3 rows to the intermediate coverage-map table, add the 3 architecture-diagram entries to the Diagram Coverage Targets table |

No existing files are deleted. No code/tests to run — this is documentation, so "verification" steps are template-conformance and link-resolution checks via `Grep`/`Read`, not automated tests.

---

## Authoring Conventions (read once, apply to every note)

Before writing any of the three notes, internalize these rules. They come from the vault's root `CLAUDE.md` and `Database/CLAUDE.md`:

1. **Template order is fixed** — frontmatter → `# Title` → one-liner blockquote → `## Quick Reference` table → `## Core Concept` (≤300 words) → `## Diagram` (Mermaid) → `## Syntax & API` → `## Common Patterns` → `## Gotchas & Tips` → `## See Also`.
2. **Every code block has a language tag.** Allowed for these notes: `sql`, `bash`, `csharp`, `json`, `mermaid`. No bare ` ``` ` fences.
3. **Internal links use `[[Note Title]]`** (no path, no extension). Examples: `[[20 - NoSQL Fundamentals]]`, `[[02 - Transactions and ACID]]`. Wiki-links to the two sibling architecture notes are required in each note's `## See Also`.
4. **Quick Reference table is mandatory** — it is the scannable cheatsheet anchor. ~8–12 rows.
5. **Core Concept ≤ 300 words.** Put depth in sub-sections beneath. Use analogies, not jargon-without-definition.
6. **One Mermaid diagram per note, minimum.** Architecture notes benefit from multiple: a high-level component diagram and a sequenceDiagram showing a write path. Cap each diagram at ~15 nodes; split if larger.
7. **Cross-engine consistency** — all three notes must use the **same column order** in their Comparison Matrix sub-section so they line up visually. Column order: `PostgreSQL | SQL Server | MongoDB`.
8. **No emojis** in note bodies unless quoted from an external source.

---

## Comparison Matrix — Shared Source of Truth

Every architecture note ends `## Common Patterns` with a sub-section `### Comparison with the other two engines` containing the **same** table. Producing the table once and copying it into all three notes guarantees consistency. The canonical table:

```markdown
| Dimension | PostgreSQL | SQL Server | MongoDB |
|-----------|------------|------------|---------|
| **Model** | Relational (SQL) | Relational (T-SQL) | Document (BSON) |
| **Process model** | Process-per-connection (postmaster forks a backend) | Single process, user-mode scheduler (SQLOS) with thread pool | Single process (`mongod`), thread-per-connection |
| **Storage unit** | 8 KB pages, heap files + TOAST for large values | 8 KB pages grouped into 64 KB extents (`.mdf`/`.ndf`) | BSON documents in collections; WiredTiger files |
| **Storage engine** | Heap + B-tree/GIN/GiST/BRIN indexes | Pluggable (rowstore, columnstore, in-memory Hekaton) | WiredTiger (default; B-tree or LSM) |
| **Concurrency** | MVCC via heap-tuple versioning (no read locks) | Locking by default; optional MVCC via Snapshot Isolation / RCSI | MVCC via WiredTiger, document-level locking |
| **Durability** | WAL (Write-Ahead Log) + fsync at commit | Transaction log (`.ldf`) + WAL discipline | Journal (per-write) + oplog (per-replica-set) |
| **Default isolation** | Read Committed | Read Committed (locking) | Snapshot (within a transaction); read-uncommitted-ish outside |
| **Transactions** | Full ACID, any statement set | Full ACID, any statement set | Multi-document since 4.0 (replica set) / 4.2 (sharded) |
| **Replication** | Streaming (physical WAL) + Logical (decoded) | Always On Availability Groups (sync/async) + log shipping | Replica set: primary + secondaries replay the oplog |
| **HA / failover** | Patroni / repmgr / pg_auto_failover (external) | Built-in AG automatic failover with WSFC quorum | Built-in via replica set election (Raft-like) |
| **Horizontal scale** | Manual partitioning + Citus extension | Partitioned tables; sharding via SQL Sharding/Synapse | First-class sharding via `mongos` + config servers |
| **Query planner** | Cost-based, plans cached per prepared statement | Cost-based, plans cached aggressively in the plan cache | Rule-based + cost-based hybrid; plan cache per query shape |
| **Extension model** | First-class extensions (PostGIS, pgvector, TimescaleDB, pg_trgm) | Built-in feature surface + CLR assemblies; few "extensions" in the Postgres sense | Limited; some features behind Atlas-only flags |
| **Strongest fit** | OLTP + mixed analytics, JSON-and-SQL, GIS, embeddings, open-source TCO | Windows/.NET enterprises, BI stack (SSAS/SSRS/SSIS), Azure SQL hybrid | Flexible schema, hierarchical/polymorphic data, content/CMS, IoT |
| **Weakest fit** | Massive horizontal scale without extensions | Cross-platform OSS preferences; licensing cost at scale | Cross-document joins/transactions at scale; strict relational integrity |
```

**Resemblances to call out (in each note's Core Concept or Gotchas):**
- All three use **B-tree** as the primary index data structure (B+tree variants).
- All three implement **MVCC** in some form: Postgres natively, SQL Server opt-in (RCSI/SI), MongoDB via WiredTiger.
- All three use **write-ahead durability** (WAL / transaction log / journal+oplog).
- All three support **secondary indexes**, **server-side scripting** (PL/pgSQL, T-SQL, MongoDB aggregation/JavaScript), and **role-based access control**.
- All three offer **TLS in transit** and **encryption at rest** (TDE in SQL Server, `pgcrypto`/cluster TDE in Postgres, encrypted storage engine in MongoDB).

**Key differences to emphasize:**
- **Data model**: relational vs document — this is the gating choice. JSON-in-Postgres ≠ MongoDB; understand the tradeoffs.
- **Concurrency primitive**: Postgres assumes MVCC always; SQL Server assumes locking unless you opt in; MongoDB MVCC is per-document, not per-row-with-cross-row-snapshot.
- **Scale-out model**: Postgres = vertical first, extensions for horizontal; SQL Server = vertical-first with feature-level partitioning; MongoDB = sharding is first-class.
- **Failover**: external (Postgres) vs built-in cluster (SQL Server AGs, MongoDB replica sets).

---

## Task 1: Create `22 - PostgreSQL Architecture.md`

**Files:**
- Create: `Database/02-Intermediate/22 - PostgreSQL Architecture.md`
- Reference: `_Templates/Note Template.md` (structure), `Database/02-Intermediate/21 - B-Tree Internals.md` (depth/style benchmark)

- [ ] **Step 1.1: Write frontmatter + title + one-liner**

```markdown
---
tags: [database, intermediate, postgresql, architecture, mvcc, wal, replication]
aliases: [Postgres Architecture, PG Internals, Postgres Internals]
level: Intermediate
---

# PostgreSQL Architecture

> **One-liner**: PostgreSQL is a process-per-connection RDBMS that achieves concurrency through MVCC (every write creates a new row version) and durability through the WAL — a single sequential log every change is appended to before it touches the heap.
```

- [ ] **Step 1.2: Write the Quick Reference table**

Must contain 10–12 rows covering the cheatsheet facts a reader needs in seconds. Use this exact starter (extend if useful, do not shorten):

```markdown
---

## Quick Reference

| Item | Value / Fact |
|------|--------------|
| Process model | `postmaster` forks one OS process per client connection |
| Concurrency control | MVCC — readers never block writers, writers never block readers |
| Durability mechanism | Write-Ahead Log (WAL) — flushed before commit returns |
| Default page size | 8 KB |
| Storage layout | Heap files per table + per-index B-tree files + TOAST for >2 KB values |
| Default index type | B-tree (B+tree variant) |
| Background workers | `checkpointer`, `bgwriter`, `walwriter`, `autovacuum launcher/workers`, `archiver`, `logical replication workers` |
| Default isolation | Read Committed |
| Replication | Streaming (physical, byte-for-byte WAL) or Logical (decoded change stream) |
| Pluggable storage engine | No — heap-only since v12 introduced the API but no in-tree alternative |
| Extensibility | First-class: PostGIS, pgvector, TimescaleDB, pg_trgm, pg_stat_statements, citus |
| Inspect runtime | `pg_stat_activity`, `pg_stat_bgwriter`, `pg_stat_replication`, `pg_stat_wal` |

---
```

- [ ] **Step 1.3: Write the Core Concept (≤300 words)**

Constraint: ≤300 words. Cover, in this order:
1. **Process model** — `postmaster` is the parent process; on each connection it forks a backend process; backends share memory but isolate connection state.
2. **Shared memory** — `shared_buffers` (page cache), WAL buffers, lock table, proc array. Backends talk to disk via this shared cache.
3. **MVCC in one paragraph** — every `UPDATE` writes a new tuple version; old version stays until no transaction can see it; autovacuum reclaims dead tuples. The "no read locks" property comes from this.
4. **WAL** — every change is first written to the WAL (sequential, fast); the heap is updated lazily; on commit, WAL is fsynced; on crash, replay replays WAL since the last checkpoint.
5. **Why it scales the way it does** — vertical first; sharding via Citus or app-level partitioning; replication via WAL streaming.

Style: short paragraphs, no bullet lists in Core Concept.

- [ ] **Step 1.4: Write the Diagram section with two Mermaid diagrams**

First diagram — process and memory model. Second diagram — write path (sequence diagram from client through WAL to heap).

```markdown
---

## Diagram

### Process and Memory Model

\`\`\`mermaid
graph TD
    Client1[Client conn 1] --> BE1[Backend process 1]
    Client2[Client conn 2] --> BE2[Backend process 2]
    Client3[Client conn 3] --> BE3[Backend process 3]

    Postmaster[postmaster<br/>parent process]
    Postmaster -.forks.-> BE1
    Postmaster -.forks.-> BE2
    Postmaster -.forks.-> BE3

    subgraph SharedMem[Shared Memory]
        Buffers[shared_buffers<br/>page cache]
        WALBuf[WAL buffers]
        LockTable[Lock table]
        ProcArray[Proc array]
    end

    BE1 --> Buffers
    BE2 --> Buffers
    BE3 --> Buffers
    BE1 --> WALBuf
    BE2 --> WALBuf
    BE3 --> WALBuf

    BGWriter[bgwriter] --> Buffers
    Checkpointer[checkpointer] --> Buffers
    WALWriter[walwriter] --> WALBuf
    Autovacuum[autovacuum] --> Buffers

    Buffers --> Heap[(Heap files<br/>per table)]
    WALBuf --> WAL[(WAL segment files)]
    WAL --> Archive[(WAL archive)]
\`\`\`

### Write Path — `UPDATE users SET name=$1 WHERE id=$2`

\`\`\`mermaid
sequenceDiagram
    participant C as Client
    participant BE as Backend
    participant SB as shared_buffers
    participant WB as WAL buffer
    participant DSK as Disk
    C->>BE: BEGIN; UPDATE...
    BE->>SB: Load page (cache miss → read from disk)
    BE->>SB: Insert NEW tuple version<br/>mark OLD tuple xmax=tx_id
    BE->>WB: Append WAL record
    C->>BE: COMMIT
    BE->>WB: WAL flush (fsync)
    WB->>DSK: WAL durable
    BE-->>C: OK
    Note over SB,DSK: Heap page stays dirty in cache;<br/>checkpointer flushes later
\`\`\`

---
```

(Note: when authoring, replace `\`\`\`mermaid` with three plain backticks — they are escaped here only because this plan file itself is markdown.)

- [ ] **Step 1.5: Write the Syntax & API section**

Cover three sub-sections with runnable code:

1. **Inspect the running cluster** — `pg_stat_activity`, `pg_stat_bgwriter`, `pg_stat_wal`, `\l+` in psql, `SHOW shared_buffers`, `SHOW max_wal_size`.
2. **Watch MVCC in action** — open two sessions, demonstrate that session A sees its own update while session B sees the old row until commit; then run `VACUUM (VERBOSE)` to show tuple cleanup.
3. **Inspect WAL and checkpoint behavior** — `pg_current_wal_lsn()`, `pg_walfile_name()`, `pg_switch_wal()`, force a `CHECKPOINT`.

Provide complete snippets for each — do not write "show an example here". Example anchor for sub-section 2:

```sql
-- Session A
BEGIN;
UPDATE users SET name = 'Alicia' WHERE id = 1;
-- (don't commit yet)

-- Session B (parallel session, same DB)
SELECT name FROM users WHERE id = 1;
-- → 'Alice'  (sees pre-update snapshot — MVCC)

-- Session A
COMMIT;

-- Session B
SELECT name FROM users WHERE id = 1;
-- → 'Alicia'  (sees new committed version)

-- Inspect tuple versions (xmin/xmax)
SELECT id, name, xmin, xmax FROM users WHERE id = 1;
```

- [ ] **Step 1.6: Write the Common Patterns section**

Cover these sub-sections with runnable code or concrete steps:

1. **Tuning checkpoint behavior** — `checkpoint_timeout`, `max_wal_size`, `checkpoint_completion_target`; why bursty I/O hurts and how spreading checkpoints helps.
2. **Autovacuum tuning** — `autovacuum_vacuum_scale_factor`, per-table overrides via `ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.05)`.
3. **Streaming replication setup** — minimal `primary_conninfo` + `pg_basebackup` + `standby.signal` recipe.
4. **Logical replication setup** — `CREATE PUBLICATION` + `CREATE SUBSCRIPTION` minimal example.
5. **`### Comparison with the other two engines`** — **paste the canonical Comparison Matrix table verbatim** from the "Comparison Matrix — Shared Source of Truth" section of this plan. Do not rewrite it.

- [ ] **Step 1.7: Write the Gotchas & Tips section**

Bullet list, 8–12 items. Required entries (write at least these):

- MVCC means `UPDATE` is conceptually `DELETE + INSERT` at the row level → heavily updated tables bloat without autovacuum, even if row count is stable.
- `VACUUM` does **not** return disk to the OS (only marks space reusable); only `VACUUM FULL` rewrites and shrinks, but it takes an exclusive lock. Use `pg_repack` for online compaction.
- Connection-per-process is expensive (~10 MB+ per backend). Use **PgBouncer** in transaction-pooling mode for high-concurrency apps. Cross-link `[[13 - Connection Management]]`.
- `synchronous_commit = off` trades durability for throughput — you can lose committed transactions on crash. Know what you're trading.
- WAL is the **only** source of truth for replication and PITR. Lose the WAL → lose the ability to recover.
- `pg_stat_statements` is the single highest-ROI extension for performance troubleshooting. Enable it in every cluster.
- Default `shared_buffers = 128 MB` is a development default; tune to ~25% of RAM in production.
- `max_connections` interacts with shared memory sizing — every backend reserves slots in `shared_buffers`, `proc array`, and lock table. Don't set `max_connections = 5000` casually.
- Long-running transactions hold back `xmin` → autovacuum can't clean up newer dead tuples → table bloat. Monitor `pg_stat_activity` for transactions older than minutes.
- The planner uses statistics from `ANALYZE`. After bulk loads, run `ANALYZE` manually.

- [ ] **Step 1.8: Write the See Also section**

```markdown
---

## See Also

- [[01 - Database Overview]] — engine taxonomy and where Postgres fits
- [[02 - Transactions and ACID]] — the MVCC visibility rules in depth
- [[03 - Isolation Levels]] — Read Committed, Repeatable Read, Serializable in Postgres
- [[04 - Locking and Concurrency]] — row locks, advisory locks, deadlocks
- [[05 - Indexes Advanced]] — GIN, GiST, BRIN, partial, covering
- [[09 - Performance Tuning]] — `pg_stat_statements`, autovacuum, plan regressions
- [[13 - Connection Management]] — PgBouncer, Npgsql pooling
- [[15 - Backup and Restore]] — `pg_basebackup`, PITR, WAL archiving
- [[02 - Replication]] — streaming and logical replication
- [[21 - B-Tree Internals]] — the default index data structure
- [[23 - SQL Server Architecture]] — sibling architecture note
- [[24 - MongoDB Architecture]] — sibling architecture note
```

- [ ] **Step 1.9: Self-check the note**

Open the file. Verify in order:
1. Frontmatter has `tags`, `aliases`, `level`.
2. Sections appear in canonical order: title → one-liner → Quick Reference → Core Concept → Diagram → Syntax & API → Common Patterns → Gotchas & Tips → See Also.
3. Every code fence has a language tag (`sql`, `bash`, `csharp`, `mermaid`).
4. Word count of Core Concept is ≤300 words (paste into a counter or eyeball — 4 short paragraphs ≈ 250 words).
5. Comparison Matrix table is present in Common Patterns and matches the canonical version in this plan letter-for-letter.
6. All `[[Note Title]]` links resolve to files that exist. Verify with:

```bash
grep -oE '\[\[[^]]+\]\]' "Database/02-Intermediate/22 - PostgreSQL Architecture.md"
```

For each match, confirm a file with that title exists in `Database/`. `[[23 - SQL Server Architecture]]` and `[[24 - MongoDB Architecture]]` won't exist yet — that's fine, they'll be created in Tasks 2 and 3.

- [ ] **Step 1.10: Commit**

```bash
git add "Database/02-Intermediate/22 - PostgreSQL Architecture.md"
git commit -m "docs(database): add 22 - PostgreSQL Architecture note"
```

---

## Task 2: Create `23 - SQL Server Architecture.md`

**Files:**
- Create: `Database/02-Intermediate/23 - SQL Server Architecture.md`
- Reference: Task 1's note (same template), `Database/CLAUDE.md` "Dialect notes" guidance — call out that examples use T-SQL.

- [ ] **Step 2.1: Write frontmatter + title + one-liner**

```markdown
---
tags: [database, intermediate, sqlserver, architecture, sqlos, locking, dotnet]
aliases: [SQL Server Internals, MSSQL Architecture, T-SQL Architecture]
level: Intermediate
---

# SQL Server Architecture

> **One-liner**: SQL Server is a single-process RDBMS that runs its own cooperative scheduler (SQLOS) on top of the OS, uses a buffer pool + plan cache for hot data and parsed queries, defaults to pessimistic locking, and writes durability through a per-database transaction log.
```

- [ ] **Step 2.2: Write the Quick Reference table**

```markdown
---

## Quick Reference

| Item | Value / Fact |
|------|--------------|
| Process model | Single `sqlservr.exe` process; SQLOS schedules thread-like "tasks" via fibers/workers |
| Concurrency control | Locking by default; optional MVCC via Read Committed Snapshot (RCSI) or Snapshot Isolation |
| Durability mechanism | Per-database transaction log (`.ldf`) — WAL discipline; flushed before commit returns |
| Default page size | 8 KB; 8 pages = one 64 KB **extent** |
| Storage files | `.mdf` primary data, `.ndf` secondary data, `.ldf` transaction log |
| Storage engines | Rowstore (default), Columnstore (analytics), In-Memory OLTP (Hekaton) |
| Default index type | B-tree (clustered + nonclustered) |
| Default isolation | Read Committed (locking flavor) |
| High availability | Always On Availability Groups (sync/async commit), Failover Cluster Instances, log shipping |
| Query optimizer | Cost-based; plan cache aggressively reused; "parameter sniffing" tradeoffs |
| Editions | Express (free, 10 GB), Standard, Enterprise, Developer (free for dev), Azure SQL (managed) |
| Inspect runtime | `sys.dm_exec_requests`, `sys.dm_os_wait_stats`, `sys.dm_exec_query_stats` |

---
```

- [ ] **Step 2.3: Write the Core Concept (≤300 words)**

Cover, in this order:
1. **SQLOS** — SQL Server runs a cooperative user-mode scheduler inside one process. Each logical CPU gets a scheduler; workers (lightweight tasks) yield cooperatively. This is why CPU pressure shows up as `SOS_SCHEDULER_YIELD` waits.
2. **Memory architecture** — buffer pool holds 8 KB pages; plan cache holds compiled query plans; lock manager tracks lock owners; columnstore object pool for compressed segments.
3. **Storage** — every database has at least one data file (`.mdf`) and one log file (`.ldf`). Pages live on data files; the log is sequential.
4. **Concurrency by default = locking** — readers acquire shared locks unless RCSI is on. This is the most important behavioral difference from Postgres.
5. **Transaction log** — WAL semantics: log writes must hit disk before commit returns; checkpoints flush dirty data pages; recovery replays the log.

- [ ] **Step 2.4: Write the Diagram section with two Mermaid diagrams**

First diagram — process, SQLOS, memory layout. Second diagram — write path (sequence diagram showing log-buffer flush ordering).

```markdown
---

## Diagram

### Process, SQLOS, and Memory

\`\`\`mermaid
graph TD
    Client1[Client conn 1] --> Net[TDS endpoint]
    Client2[Client conn 2] --> Net
    Client3[Client conn 3] --> Net

    Net --> SQLOS[SQLOS scheduler<br/>cooperative workers]

    subgraph SQLServer[sqlservr.exe — single process]
        SQLOS --> BufferPool[Buffer pool<br/>8 KB pages]
        SQLOS --> PlanCache[Plan cache<br/>compiled queries]
        SQLOS --> LockMgr[Lock manager]
        SQLOS --> LogBuffer[Log buffer]
    end

    BufferPool --> MDF[(`.mdf` / `.ndf`<br/>data files)]
    LogBuffer --> LDF[(`.ldf` transaction log)]
    LDF --> Backup[(Log backups<br/>for PITR)]
\`\`\`

### Write Path — `UPDATE Users SET Name=@n WHERE Id=@id`

\`\`\`mermaid
sequenceDiagram
    participant C as Client
    participant Q as Query exec
    participant BP as Buffer pool
    participant LB as Log buffer
    participant LDF as `.ldf` on disk
    participant MDF as `.mdf` on disk
    C->>Q: BEGIN TRAN; UPDATE...
    Q->>BP: Load page (read from MDF if cold)
    Q->>BP: Modify row in-place (dirty page)
    Q->>LB: Write log record (LSN assigned)
    C->>Q: COMMIT
    Q->>LB: Flush log records up to commit LSN
    LB->>LDF: fsync — durable
    Q-->>C: OK
    Note over BP,MDF: Dirty page stays in pool;<br/>CHECKPOINT flushes later
\`\`\`

---
```

- [ ] **Step 2.5: Write the Syntax & API section**

Use `sql` fences (T-SQL) and `csharp` for .NET integration. Three sub-sections:

1. **Inspect SQLOS and memory** — `sys.dm_os_schedulers`, `sys.dm_os_wait_stats`, `sys.dm_os_memory_clerks`, `sys.dm_exec_requests`.
2. **Demonstrate locking vs RCSI** — show a `SELECT` blocked by an uncommitted `UPDATE` under default Read Committed (locking), then enable `ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON` and show the same `SELECT` returning the pre-update value without blocking.
3. **.NET connection example** — `Microsoft.Data.SqlClient` with parameterized query and `TransactionScope`.

Concrete example anchor:

```sql
-- Enable Read Committed Snapshot Isolation (database-wide)
ALTER DATABASE Shop SET READ_COMMITTED_SNAPSHOT ON;

-- Session A
BEGIN TRAN;
UPDATE Users SET Name = N'Alicia' WHERE Id = 1;
-- (don't commit)

-- Session B (under RCSI)
SELECT Name FROM Users WHERE Id = 1;
-- → N'Alice'  (snapshot read, no blocking)

-- Without RCSI, Session B would block on Session A's exclusive lock.
```

```csharp
using Microsoft.Data.SqlClient;

await using var conn = new SqlConnection("Server=.;Database=Shop;Integrated Security=true;TrustServerCertificate=true");
await conn.OpenAsync();

await using var cmd = new SqlCommand("UPDATE Users SET Name=@n WHERE Id=@id", conn);
cmd.Parameters.Add("@n",  SqlDbType.NVarChar, 100).Value = "Alicia";
cmd.Parameters.Add("@id", SqlDbType.Int).Value          = 1;
await cmd.ExecuteNonQueryAsync();
```

- [ ] **Step 2.6: Write the Common Patterns section**

1. **Always On Availability Groups** — sketch a 3-replica AG (1 primary sync + 1 secondary sync + 1 secondary async); call out the WSFC quorum requirement.
2. **Columnstore for analytics** — `CREATE CLUSTERED COLUMNSTORE INDEX` on a fact table; explain segment compression and batch-mode execution at a high level.
3. **In-Memory OLTP (Hekaton)** — `CREATE TABLE ... WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA)`; latch-free, optimistic concurrency, lock-free hash/range indexes.
4. **Plan cache and parameter sniffing** — `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, plan guides — call out the sniffing pitfall briefly.
5. **`### Comparison with the other two engines`** — **paste the canonical Comparison Matrix table** verbatim.

- [ ] **Step 2.7: Write the Gotchas & Tips section**

8–12 bullets. Required entries:

- Default isolation is **locking** Read Committed — heavy read traffic can block writers. Enable RCSI to get MVCC-like behavior, but it adds version-store pressure on `tempdb`.
- **`tempdb` is shared** across the entire instance — RCSI/SI versions, sort spills, temp tables, and table variables all live there. Right-size and split into multiple files to avoid allocation-page contention.
- The **plan cache** can serve a bad plan for non-representative parameters ("parameter sniffing"). Symptom: same query is fast for some inputs, slow for others. Mitigations: `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, query hints, or upgrade the cardinality estimator.
- **Editions matter for licensing and feature gating.** Standard edition caps memory and lacks Always On AG (only AlwaysOn basic). Enterprise unlocks everything. Verify features against your edition before designing for them.
- `sys.dm_os_wait_stats` is your first stop for "why is it slow" — wait categories tell you whether it's CPU (`SOS_SCHEDULER_YIELD`), I/O (`PAGEIOLATCH_*`), locks (`LCK_M_*`), or log writes (`WRITELOG`).
- **Backup chain matters.** Full + differential + log backups form a chain — break it and you lose the ability to restore to point-in-time. Set `RECOVERY MODEL = FULL` for production OLTP.
- **TDE** (Transparent Data Encryption) protects data at rest at the file level; **Always Encrypted** protects columns end-to-end so the server never sees plaintext.
- The `WITH (NOLOCK)` hint = `READ UNCOMMITTED`. It can return dirty reads, duplicate rows, or missing rows from page splits. Don't use it as a "make it fast" knob; use RCSI instead.
- T-SQL is rich but proprietary. `MERGE` has historical bugs — many shops have a blanket ban on production `MERGE`; use explicit `UPDATE` + `INSERT` instead.
- **`COMPATIBILITY_LEVEL`** governs which version of the cardinality estimator is used. Upgrading the server doesn't automatically upgrade the optimizer behavior — plan accordingly.

- [ ] **Step 2.8: Write the See Also section**

```markdown
---

## See Also

- [[01 - Database Overview]] — engine taxonomy
- [[02 - Transactions and ACID]]
- [[03 - Isolation Levels]] — RCSI and Snapshot Isolation specifics
- [[04 - Locking and Concurrency]] — SQL Server's pessimistic default
- [[14 - ADO.NET and Dapper]] — .NET integration
- [[17 - Cloud Databases]] — Azure SQL Database / Managed Instance
- [[22 - PostgreSQL Architecture]] — sibling architecture note
- [[24 - MongoDB Architecture]] — sibling architecture note
```

- [ ] **Step 2.9: Self-check the note**

Same checklist as Step 1.9. Confirm Comparison Matrix is byte-identical to the version in Task 1 (compare with `diff` or eyeball). Run:

```bash
grep -oE '\[\[[^]]+\]\]' "Database/02-Intermediate/23 - SQL Server Architecture.md"
```

`[[24 - MongoDB Architecture]]` won't exist yet — that's fine.

- [ ] **Step 2.10: Commit**

```bash
git add "Database/02-Intermediate/23 - SQL Server Architecture.md"
git commit -m "docs(database): add 23 - SQL Server Architecture note"
```

---

## Task 3: Create `24 - MongoDB Architecture.md`

**Files:**
- Create: `Database/02-Intermediate/24 - MongoDB Architecture.md`
- Reference: Tasks 1 and 2 for template adherence; `02-Intermediate/20 - NoSQL Fundamentals.md` for the existing Mongo snippets (don't duplicate; cross-link).

- [ ] **Step 3.1: Write frontmatter + title + one-liner**

```markdown
---
tags: [database, intermediate, nosql, mongodb, architecture, replication, sharding]
aliases: [Mongo Architecture, MongoDB Internals, mongod]
level: Intermediate
---

# MongoDB Architecture

> **One-liner**: MongoDB is a document database built around `mongod` and the WiredTiger storage engine; replica sets provide HA via an oplog-driven primary-secondary topology, and sharded clusters provide horizontal scale through a `mongos` router fronting per-shard replica sets.
```

- [ ] **Step 3.2: Write the Quick Reference table**

```markdown
---

## Quick Reference

| Item | Value / Fact |
|------|--------------|
| Process | `mongod` (data node), `mongos` (sharding router), `mongocfg`/config-server replicas |
| Document format | BSON — binary JSON with extended types (`ObjectId`, `Date`, `Decimal128`, `Binary`) |
| Default storage engine | WiredTiger |
| Concurrency control | MVCC via WiredTiger; document-level locking |
| Durability mechanism | Journal (per-write, group-commit) + oplog (per-replica-set) |
| Default page / block | 32 KB internal page in WiredTiger; documents up to 16 MB |
| Replica set | 1 primary + N secondaries (+ optional arbiter); Raft-like election |
| Sharding | `mongos` router + config servers + N shard replica sets; chunks split on shard key |
| Default index type | B-tree (single, compound, multikey, text, geo, hashed, wildcard) |
| Transactions | Multi-document since 4.0 (replica set), 4.2 (sharded); default scope is single-document |
| Read concerns | `local`, `available`, `majority`, `linearizable`, `snapshot` |
| Write concerns | `{ w: 1 }` … `{ w: "majority", j: true, wtimeout: 5000 }` |
| Inspect runtime | `db.serverStatus()`, `db.currentOp()`, `rs.status()`, `sh.status()` |

---
```

- [ ] **Step 3.3: Write the Core Concept (≤300 words)**

Cover, in this order:
1. **The data unit is the document.** Collections hold BSON documents up to 16 MB. No schema by default; schemas live in the application or in `$jsonSchema` validators.
2. **WiredTiger** — pluggable storage engine, default since 3.2. Supports B-tree and LSM-tree; default is B-tree. Document-level locking and MVCC mean concurrent writes to different documents in the same collection don't block.
3. **Replica set** — primary handles writes; secondaries replay the **oplog** (a capped collection that logs every write). On primary failure, secondaries hold an election (Raft-like). Clients are oplog-aware and automatically reconnect to the new primary.
4. **Sharding** — horizontal scale built in. A `mongos` router accepts client traffic, asks the config servers where each shard key lives, and routes the query. Each shard is itself a replica set. Chunks (ranges of shard-key values) auto-balance.
5. **Read/write concerns are tunable** — `w: "majority", j: true` is the safe default. You can trade durability/latency on a per-operation basis.

- [ ] **Step 3.4: Write the Diagram section with two Mermaid diagrams**

First — replica set topology with oplog replication. Second — sharded cluster topology with query routing.

```markdown
---

## Diagram

### Replica Set

\`\`\`mermaid
graph TD
    Client[Client driver] --> Primary[(Primary<br/>mongod)]
    Primary -.oplog tail.-> Sec1[(Secondary 1)]
    Primary -.oplog tail.-> Sec2[(Secondary 2)]

    Primary --> Journal1[(Journal)]
    Sec1 --> Journal2[(Journal)]
    Sec2 --> Journal3[(Journal)]

    Primary --> WT1[WiredTiger files]
    Sec1 --> WT2[WiredTiger files]
    Sec2 --> WT3[WiredTiger files]

    Sec1 -.election heartbeat.-> Sec2
    Sec1 -.election heartbeat.-> Primary
    Sec2 -.election heartbeat.-> Primary
\`\`\`

### Sharded Cluster

\`\`\`mermaid
graph TD
    App[App driver] --> Mongos1[mongos router]
    App --> Mongos2[mongos router]

    Mongos1 --> Cfg[(Config server<br/>replica set)]
    Mongos2 --> Cfg

    Mongos1 --> Shard1[(Shard A<br/>replica set)]
    Mongos1 --> Shard2[(Shard B<br/>replica set)]
    Mongos1 --> Shard3[(Shard C<br/>replica set)]
    Mongos2 --> Shard1
    Mongos2 --> Shard2
    Mongos2 --> Shard3

    Balancer[Balancer<br/>chunk migrations] -.-> Shard1
    Balancer -.-> Shard2
    Balancer -.-> Shard3
\`\`\`

---
```

- [ ] **Step 3.5: Write the Syntax & API section**

Use `json` for shell commands (Mongo shell is JS-based; the existing `20 - NoSQL Fundamentals` note uses `javascript` — match that for consistency). Provide three sub-sections:

1. **Inspect runtime** — `db.serverStatus()`, `rs.status()`, `sh.status()`, `db.currentOp()`, `db.collection.stats()`.
2. **Set up a replica set locally** — three `mongod --replSet rs0` processes + `rs.initiate()`.
3. **A multi-document transaction** — `client.startSession()` → `session.startTransaction()` → operations → `session.commitTransaction()` in the Node/.NET driver.

Concrete example anchor:

```javascript
// rs.initiate on the first node
rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "mongo1:27017" },
        { _id: 1, host: "mongo2:27017" },
        { _id: 2, host: "mongo3:27017" }
    ]
})

rs.status()       // see primary/secondary roles, oplog position
db.printReplicationInfo()
```

```csharp
// .NET driver — multi-document transaction
using MongoDB.Driver;

var client = new MongoClient("mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0");
using var session = await client.StartSessionAsync();

session.StartTransaction(new TransactionOptions(
    readConcern: ReadConcern.Snapshot,
    writeConcern: WriteConcern.WMajority));

try {
    var orders   = client.GetDatabase("shop").GetCollection<BsonDocument>("orders");
    var accounts = client.GetDatabase("shop").GetCollection<BsonDocument>("accounts");

    await accounts.UpdateOneAsync(session, Builders<BsonDocument>.Filter.Eq("_id", 1),
        Builders<BsonDocument>.Update.Inc("balance", -100));
    await orders.InsertOneAsync(session, new BsonDocument {
        { "user_id", 1 }, { "total", 100 }
    });

    await session.CommitTransactionAsync();
} catch {
    await session.AbortTransactionAsync();
    throw;
}
```

- [ ] **Step 3.6: Write the Common Patterns section**

1. **Choosing a shard key** — high-cardinality, low-frequency-update, monotonically-non-increasing (avoid hot shards on `_id` or `created_at` without hashing).
2. **Read preference** — `primary`, `primaryPreferred`, `secondary`, `nearest`; how secondaries serve reads at the cost of staleness.
3. **Write concerns and journal** — table of `w` + `j` combinations and their durability/latency tradeoffs.
4. **Schema-design patterns** — embed vs reference; the "extended reference" pattern; bucketing time-series.
5. **`### Comparison with the other two engines`** — **paste the canonical Comparison Matrix table** verbatim.

- [ ] **Step 3.7: Write the Gotchas & Tips section**

8–12 bullets. Required entries:

- The 16 MB document limit is hard. Designs that "just append to an array forever" eventually hit it. Use bucketing or separate collections.
- `$lookup` exists but is not a JOIN equivalent — it streams from one collection per outer document; for OLTP joins, embed or denormalize instead.
- Multi-document transactions exist but are **expensive**. Snapshot reads pull the WiredTiger transaction view; long-running transactions hold snapshots and pressure cache. Keep transactions short (<60 s default limit).
- A **hot shard key** (e.g., `_id` with `ObjectId` — monotonically increasing) routes all writes to one shard. Use a **hashed** shard key or composite shard key including a high-cardinality field.
- `w: 1` (the historical default in some drivers) acknowledges only the primary. A primary crash before replication loses the write. Use `w: "majority"` for durability.
- **Chunk migration** during the balancer's work can spike CPU and I/O. Schedule balancer windows in maintenance hours for high-traffic clusters.
- The oplog is a **capped collection** sized by `--oplogSize` (default 5% of disk). Long secondary downtime that exceeds oplog retention forces an initial sync (full data copy).
- Aggregation pipelines are powerful but stages run in order — `$match` first, then `$project`, then `$group`, etc. A pipeline that puts `$group` before `$match` materializes far more data than needed.
- Indexes pay write cost like in any RDBMS. Multikey indexes on large arrays multiply key writes per document. Audit unused indexes via `db.collection.aggregate([{$indexStats:{}}])`.
- The "schemaless" property is real at the storage layer but a liability at the application layer. Use **schema validators** (`$jsonSchema`) on production collections.
- Read concern `majority` requires majority commit point — if the cluster loses majority, `majority` reads stall. Don't confuse "read concern majority" with "read preference primary".
- Atlas (managed) and self-hosted differ in operational surface (encryption-at-rest, search, change streams) — verify before designing.

- [ ] **Step 3.8: Write the See Also section**

```markdown
---

## See Also

- [[01 - Database Overview]] — engine taxonomy
- [[18 - JSON and JSONB]] — Postgres's document story
- [[20 - NoSQL Fundamentals]] — document/key-value/wide-column/graph overview
- [[04 - CAP and PACELC]] — consistency vs availability tradeoffs
- [[02 - Replication]] — replication concepts; Mongo replica-set version
- [[01 - Sharding and Partitioning]] — Mongo's first-class sharding model
- [[22 - PostgreSQL Architecture]] — sibling architecture note
- [[23 - SQL Server Architecture]] — sibling architecture note
```

- [ ] **Step 3.9: Self-check the note**

Same checklist as Step 1.9. Final cross-engine check: open all three architecture notes side-by-side and verify their Comparison Matrix tables are **byte-identical**:

```bash
diff <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/22 - PostgreSQL Architecture.md") \
     <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/23 - SQL Server Architecture.md")

diff <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/23 - SQL Server Architecture.md") \
     <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/24 - MongoDB Architecture.md")
```

Both `diff` invocations should produce **no output**. If they do, fix the divergence.

- [ ] **Step 3.10: Commit**

```bash
git add "Database/02-Intermediate/24 - MongoDB Architecture.md"
git commit -m "docs(database): add 24 - MongoDB Architecture note"
```

---

## Task 4: Update Database Master Index

**Files:**
- Modify: `Database/00-Index/Database Master Index.md`

- [ ] **Step 4.1: Locate the "Specialized data" sub-section under "02 — Intermediate"**

Open `Database/00-Index/Database Master Index.md`. Find the block:

```markdown
### Specialized data
| # | Note |
|---|------|
| 18 | [[18 - JSON and JSONB]] |
| 19 | [[19 - Full-Text Search]] |
| 20 | [[20 - NoSQL Fundamentals]] |
```

- [ ] **Step 4.2: Insert a new "Engine architectures" sub-section after "Specialized data"**

Use the Edit tool. Replace the "Specialized data" block with:

```markdown
### Specialized data
| # | Note |
|---|------|
| 18 | [[18 - JSON and JSONB]] |
| 19 | [[19 - Full-Text Search]] |
| 20 | [[20 - NoSQL Fundamentals]] |

### Engine architectures
| # | Note |
|---|------|
| 21 | [[21 - B-Tree Internals]] |
| 22 | [[22 - PostgreSQL Architecture]] |
| 23 | [[23 - SQL Server Architecture]] |
| 24 | [[24 - MongoDB Architecture]] |
```

Then **remove** the old standalone `21 - B-Tree Internals` row from the "Performance and indexing" sub-section above (it now lives under "Engine architectures"). The "Performance and indexing" block should end at note 10:

```markdown
### Performance and indexing
| # | Note |
|---|------|
| 05 | [[05 - Indexes Advanced]] |
| 06 | [[06 - Query Optimization]] |
| 07 | [[07 - Views]] |
| 10 | [[10 - Window Functions]] |
```

- [ ] **Step 4.3: Verify the file**

```bash
grep -n "22 - PostgreSQL Architecture\|23 - SQL Server Architecture\|24 - MongoDB Architecture\|21 - B-Tree Internals" "Database/00-Index/Database Master Index.md"
```

Expected: exactly four matches, all under the "Engine architectures" sub-section. No duplicate `21 - B-Tree Internals` row elsewhere.

- [ ] **Step 4.4: Commit**

```bash
git add "Database/00-Index/Database Master Index.md"
git commit -m "docs(database): index engine-architecture notes under new section"
```

---

## Task 5: Update Database Learning Path

**Files:**
- Modify: `Database/00-Index/Database Learning Path.md`

- [ ] **Step 5.1: Add a new stage between Stage 6 and Stage 7**

Open `Database/00-Index/Database Learning Path.md`. Find the line `## Stage 7 — Distribution and Scaling (1–2 weeks)` and insert immediately **before** it:

```markdown
## Stage 6.5 — Engine Internals (3–5 days)

**Goal**: Understand how the three engines you're most likely to meet in industry actually work — so the words "MVCC", "transaction log", "oplog", "shard key", "buffer pool" point at concrete machinery instead of folklore.

1. [[22 - PostgreSQL Architecture]] — process model, MVCC, WAL, vacuum, planner
2. [[23 - SQL Server Architecture]] — SQLOS, buffer pool, plan cache, transaction log, Always On AGs
3. [[24 - MongoDB Architecture]] — WiredTiger, replica sets + oplog, sharding via mongos

**Checkpoint**: Without re-reading the notes, sketch the write path of a single `UPDATE` for each engine on a whiteboard — including where durability is guaranteed and what fails on a crash.

---
```

- [ ] **Step 5.2: Verify**

```bash
grep -n "Stage 6.5\|Stage 7" "Database/00-Index/Database Learning Path.md"
```

Expected output shows `Stage 6.5` appearing immediately before `Stage 7`.

- [ ] **Step 5.3: Commit**

```bash
git add "Database/00-Index/Database Learning Path.md"
git commit -m "docs(database): add Stage 6.5 — Engine Internals to learning path"
```

---

## Task 6: Update `Database/CLAUDE.md` Coverage Map

**Files:**
- Modify: `Database/CLAUDE.md`

- [ ] **Step 6.1: Update the intermediate count**

Find the heading `### Intermediate (02-Intermediate) — 21 notes`. Change to:

```markdown
### Intermediate (02-Intermediate) — 24 notes
```

- [ ] **Step 6.2: Append three rows to the intermediate coverage-map table**

The table currently ends with row 21. Add three rows after it:

```markdown
| 22 | PostgreSQL Architecture | Postmaster + backend processes, shared memory, MVCC, WAL, vacuum, planner, replication topology, comparison with SQL Server and MongoDB |
| 23 | SQL Server Architecture | SQLOS scheduler, buffer pool + plan cache, pages and extents, transaction log, locking vs RCSI, Always On AGs, columnstore, Hekaton, comparison matrix |
| 24 | MongoDB Architecture | mongod, WiredTiger, BSON + collections, replica sets and oplog, sharded cluster (mongos + config servers), read/write concerns, transactions, comparison matrix |
```

- [ ] **Step 6.3: Update the total**

Find the line `**Total: 49 notes** (10 beginner + 21 intermediate + 18 advanced)` and update to:

```markdown
**Total: 52 notes** (10 beginner + 24 intermediate + 18 advanced)
```

- [ ] **Step 6.4: Add three rows to the Diagram Coverage Targets table**

Find the table under `## Diagram Coverage Targets`. Append:

```markdown
| PostgreSQL Architecture | graph + sequenceDiagram (process model + write path) |
| SQL Server Architecture | graph + sequenceDiagram (SQLOS + write path) |
| MongoDB Architecture | graph (replica set + sharded cluster) |
```

- [ ] **Step 6.5: Verify**

```bash
grep -nE "Intermediate \(02-Intermediate\)|Total: 52 notes|22 \| PostgreSQL Architecture|23 \| SQL Server Architecture|24 \| MongoDB Architecture" "Database/CLAUDE.md"
```

Expected: 5 matching lines.

- [ ] **Step 6.6: Commit**

```bash
git add "Database/CLAUDE.md"
git commit -m "docs(database): update coverage map for engine-architecture notes"
```

---

## Task 7: Final Cross-Reference Verification

**Files:**
- Modify: none (verification only); only commit if a fix is needed.

- [ ] **Step 7.1: Verify all sibling wiki-links resolve**

```bash
grep -nE "\[\[(22 - PostgreSQL Architecture|23 - SQL Server Architecture|24 - MongoDB Architecture)\]\]" \
    "Database/02-Intermediate/22 - PostgreSQL Architecture.md" \
    "Database/02-Intermediate/23 - SQL Server Architecture.md" \
    "Database/02-Intermediate/24 - MongoDB Architecture.md"
```

Expected:
- `22 - PostgreSQL Architecture.md` contains `[[23 - SQL Server Architecture]]` and `[[24 - MongoDB Architecture]]` in `## See Also`.
- `23 - SQL Server Architecture.md` contains `[[22 - PostgreSQL Architecture]]` and `[[24 - MongoDB Architecture]]`.
- `24 - MongoDB Architecture.md` contains `[[22 - PostgreSQL Architecture]]` and `[[23 - SQL Server Architecture]]`.

If any link is missing, add it via `Edit` and commit.

- [ ] **Step 7.2: Verify the canonical Comparison Matrix is byte-identical across all three notes**

```bash
diff <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/22 - PostgreSQL Architecture.md") \
     <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/23 - SQL Server Architecture.md") \
&& \
diff <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/23 - SQL Server Architecture.md") \
     <(awk '/^### Comparison with the other two engines/,/^---$/' "Database/02-Intermediate/24 - MongoDB Architecture.md")
```

Both `diff`s must exit silently (no output, exit code 0). If either prints differences, edit the divergent file to match the canonical table from this plan, then re-run.

- [ ] **Step 7.3: Verify every code fence has a language tag**

```bash
grep -n '^```$' \
    "Database/02-Intermediate/22 - PostgreSQL Architecture.md" \
    "Database/02-Intermediate/23 - SQL Server Architecture.md" \
    "Database/02-Intermediate/24 - MongoDB Architecture.md"
```

Expected: no output (matches mean a `` ``` `` line with nothing after it, i.e., a bare fence). Bare **closing** fences are allowed — the regex matches both. Inspect any hits: a closing ` ``` ` is fine; an **opening** ` ``` ` without a language tag is a violation and must be fixed.

- [ ] **Step 7.4: Open each note in Obsidian and visually verify mermaid renders**

(If Obsidian isn't available, paste each `mermaid` block into https://mermaid.live to confirm it parses. Common syntax errors: stray `<br/>` inside node labels with double quotes, unmatched `[`/`]`/`(`/`)`, reserved keywords as node IDs.)

If a diagram fails to render, fix the syntax in-place and commit.

- [ ] **Step 7.5: Final commit (only if Steps 7.1–7.4 produced fixes)**

```bash
git add "Database/02-Intermediate/22 - PostgreSQL Architecture.md" \
        "Database/02-Intermediate/23 - SQL Server Architecture.md" \
        "Database/02-Intermediate/24 - MongoDB Architecture.md"
git commit -m "docs(database): fix cross-link / matrix / diagram issues from final review"
```

If no fixes were needed, skip the commit.

---

## Self-Review Notes (from the plan author)

**Spec coverage**: The user asked for (a) three markdown files explaining architecture and how each engine works, (b) key differences and similarities, (c) use cases. Each of Tasks 1–3 covers (a) and (c) per-engine; the canonical Comparison Matrix in Common Patterns covers (b) in all three notes; the resemblances and key-differences bullets in this plan's "Comparison Matrix — Shared Source of Truth" section give the engineer the talking points to weave into Core Concept / Gotchas where appropriate.

**Placeholders**: None — every Step contains the actual frontmatter, table content, or section requirements. The Mermaid diagrams are provided as complete code blocks (escaped only in this plan file). The Comparison Matrix table is fully written and is intended to be copied verbatim.

**Type / name consistency**: The three filenames (`22 - PostgreSQL Architecture.md`, `23 - SQL Server Architecture.md`, `24 - MongoDB Architecture.md`) appear identically in all index / coverage / cross-link updates. Wiki-link spellings match filenames exactly.

**Convention conformance**: All notes follow `_Templates/Note Template.md`. Code-fence languages comply with `Database/CLAUDE.md` Code-Block Conventions. Tag taxonomy uses `database` + level + engine-specific tags as enumerated.
