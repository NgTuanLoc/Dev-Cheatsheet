---
tags: [database, intermediate, postgresql, devops]
aliases: [pg_dump, pg_restore, PITR, WAL Archiving, Base Backup]
level: Intermediate
---

# Backup and Restore

> **One-liner**: A backup you haven't restored isn't a backup; pick a strategy that meets your RTO and RPO, then practice the restore quarterly.

---

## Quick Reference

| Tool | What it does |
|------|--------------|
| `pg_dump` | logical backup — SQL or custom format, single DB |
| `pg_dumpall` | logical backup of cluster (all DBs + globals: roles, tablespaces) |
| `pg_restore` | restore custom-format dumps |
| `pg_basebackup` | physical base backup of the cluster |
| WAL archiving + base backup | enables **Point-In-Time Recovery** (PITR) |
| `pgBackRest`, `Barman`, `WAL-G` | production-grade orchestration on top of base + WAL |

| Term | Meaning |
|------|---------|
| **RTO** (Recovery Time Objective) | how long until back online |
| **RPO** (Recovery Point Objective) | how much data loss is acceptable |
| **Logical backup** | SQL representing rows; portable; slow on huge DBs |
| **Physical backup** | byte-for-byte copy of files; fast; tied to PG version + arch |
| **PITR** | restore to any timestamp using base + replayed WAL |
| **WAL** | Write-Ahead Log — every change recorded before commit |

---

## Core Concept

Two families:

- **Logical** (`pg_dump`) — exports schema + data as SQL or a custom binary archive. Selective (one DB, one schema, one table). Restorable to a different Postgres version. Slower and larger than physical for big databases.
- **Physical** (`pg_basebackup` + WAL archiving) — copies the data directory while the server is running. Restoring is byte-for-byte. Combined with archived WAL, lets you replay forward to any point in time (**PITR**).

**RTO** is "how fast can I be back?" — physical restores are usually faster (no parsing).
**RPO** is "how much data can I lose?" — without WAL archiving you lose everything since the last dump. With continuous WAL archiving, near-zero.

The bare minimum for any production DB:
1. Daily `pg_dump` to off-site storage
2. WAL archiving to off-site storage
3. **Test restores**, regularly

Without (3), you have a *hope*, not a backup.

---

## Syntax & API

### Logical backup — pg_dump
```bash
# Single DB → custom format (recommended; supports parallel + selective restore)
pg_dump -h localhost -U postgres -d shop -F c -f shop.dump

# Plain SQL (text)
pg_dump -d shop -F p -f shop.sql

# Schema only / data only
pg_dump -d shop -F c --schema-only -f schema.dump
pg_dump -d shop -F c --data-only -f data.dump

# Specific tables
pg_dump -d shop -F c -t users -t orders -f partial.dump

# Compressed parallel dump (PG 9.3+)
pg_dump -d shop -F d -j 4 -f shop_dir/         # directory format, 4 workers
```

### pg_dumpall — entire cluster
```bash
# Includes roles, tablespaces, all DBs (large; slower than per-DB)
pg_dumpall -h localhost -U postgres -f cluster.sql

# Just globals (roles + tablespaces) — pair with per-DB pg_dumps
pg_dumpall --globals-only -f globals.sql
```

### Restore
```bash
# Custom format → pg_restore
createdb shop_restored
pg_restore -d shop_restored -F c -j 4 shop.dump

# Plain SQL → psql
psql -d shop_restored -f shop.sql

# Selective restore (objects you want)
pg_restore -d shop_restored -t users -t orders shop.dump

# Restore to a different DB / different machine
pg_restore -h other-host -U postgres -d shop_new -F c shop.dump
```

### Physical base backup
```bash
# Run while the source DB is online
pg_basebackup \
    -h primary -U replicator -D /var/backups/base \
    -F t -X fetch -P -R
# -F t  → tar format
# -X fetch → include WAL files needed
# -P    → progress
# -R    → write standby.signal (turn into a replica/restore base)
```

### WAL archiving (in postgresql.conf)
```ini
wal_level         = replica           # or 'logical' if needed
archive_mode      = on
archive_command   = 'test ! -f /archive/%f && cp %p /archive/%f'
restore_command   = 'cp /archive/%f %p'   # for restore-side recovery.conf
```

```bash
# Keep base + WAL — forms the PITR substrate
ls /var/backups/base/   # base backup
ls /archive/            # 0000000100000000000000A1, …
```

### Point-In-Time Recovery (PITR)
```bash
# 1. Stop the destination cluster, restore base
tar -xf base.tar.gz -C /var/lib/postgresql/16/main
chown -R postgres:postgres /var/lib/postgresql/16/main

# 2. Provide WAL via restore_command
echo "restore_command = 'cp /archive/%f %p'"            >  /var/lib/postgresql/16/main/recovery.signal
echo "restore_command = 'cp /archive/%f %p'"            >> /etc/postgresql/16/main/postgresql.conf
echo "recovery_target_time = '2026-04-29 11:30:00 UTC'" >> /etc/postgresql/16/main/postgresql.conf

# 3. Start; Postgres replays WAL to the target time, then sits in recovery.
sudo systemctl start postgresql

# 4. Promote (exit recovery)
psql -c "SELECT pg_promote();"
```

### Verify backups
```bash
# List contents of a custom-format dump
pg_restore -l shop.dump | less

# Test restore on a scratch DB (the only real test)
createdb shop_test && pg_restore -d shop_test shop.dump && \
psql -d shop_test -c "SELECT count(*) FROM users;"
dropdb shop_test
```

---

## Common Patterns

```bash
# Pattern: nightly logical + continuous WAL
0 2 * * *  pg_dump -d shop -F c -f /backup/shop_$(date +\%F).dump
# WAL archiving runs continuously via archive_command (above)
```

```bash
# Pattern: encrypted off-site backup
pg_dump -d shop -F c | \
  gpg --encrypt --recipient backup@example.com | \
  aws s3 cp - s3://my-backups/shop/$(date +%F).dump.gpg
```

```bash
# Pattern: pgBackRest (production orchestration)
# /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=4
[shop]
pg1-path=/var/lib/postgresql/16/main

pgbackrest --stanza=shop backup --type=full
pgbackrest --stanza=shop backup --type=diff
pgbackrest --stanza=shop restore --target-time='2026-04-29 11:30:00 UTC'
```

```sql
-- Pattern: drop unused objects before dump (smaller backups)
DROP TABLE temp_*;       -- discard one-shot scratch tables
VACUUM (FULL) huge_table; -- rare; rewrites table compactly
```

---

## Gotchas & Tips

- **Untested backup = no backup.** Schedule quarterly drills: restore to scratch, run a smoke query, confirm row counts.
- **`pg_dump` doesn't capture roles/tablespaces** — combine with `pg_dumpall --globals-only` or use `pg_basebackup` for everything.
- **Custom format (`-F c`) > plain SQL** — supports parallel restore, selective objects, smaller files.
- **Don't restore a `pg_basebackup` to a different PG major version** — physical backups are version-locked. Logical (`pg_dump`) handles upgrades.
- **`vacuumdb --analyze` after a big logical restore** — stats are missing post-restore; queries plan poorly until you analyze.
- **WAL archiving is not optional in production** — without it, RPO equals "since last full dump" which is hours of loss.
- **Encrypt backups** — they contain everything. At rest and in transit.
- **`archive_command` must be reliable** — a failed command makes WAL pile up on the primary. Monitor disk and exit codes.
- **Test PITR specifically** — base + WAL gives you nothing if you've never proven recovery works for your cluster.
- **Keep multiple generations** — daily for 7 days, weekly for 4 weeks, monthly for 12 months is typical.
- **Use `pgBackRest`/`Barman`/`WAL-G` instead of hand-rolling** — they handle parallelism, compression, encryption, retention, and restore tooling.

---

## See Also

- [[02 - Replication]]
- [[03 - High Availability and Failover]]
- [[16 - Database Security]]
- [[17 - Cloud Databases]]
