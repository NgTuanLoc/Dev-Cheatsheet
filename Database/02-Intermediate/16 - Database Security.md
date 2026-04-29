---
tags: [database, intermediate, security, postgresql]
aliases: [Roles, GRANT, REVOKE, RLS, Row-Level Security]
level: Intermediate
---

# Database Security

> **One-liner**: Defense in depth — least-privilege roles, parameterized queries, row-level security, and TLS — so a leaked credential or SQL injection doesn't expose the whole database.

---

## Quick Reference

| Object | Purpose |
|--------|---------|
| **Role** | A login or group; users and groups are both roles |
| `CREATE ROLE name LOGIN PASSWORD '…'` | a login user |
| `CREATE ROLE app_readonly NOLOGIN` | a group (no login) |
| `GRANT … ON … TO role` | give privileges |
| `REVOKE … FROM role` | take them away |
| `ALTER DEFAULT PRIVILEGES` | apply privileges to *future* objects |
| `pg_hba.conf` | host-based auth — who can connect from where, with what method |
| **RLS** (`ALTER TABLE … ENABLE ROW LEVEL SECURITY`) | per-row access control |
| `SECURITY DEFINER` function | runs with the function owner's privileges |

| Privilege keywords | |
|--------------------|---|
| `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER` | per table |
| `USAGE`, `CREATE` | on schemas |
| `EXECUTE` | on functions |
| `ALL PRIVILEGES` | shorthand |

---

## Core Concept

Postgres has three security perimeters:

1. **Network / connection** — `pg_hba.conf` decides who can authenticate, from where, with what method (`md5`, `scram-sha-256`, `cert`, `peer`). TLS encrypts the wire.
2. **Authorization (roles + privileges)** — every object has an owner; privileges are granted to roles. Apply **least privilege**: an app role doesn't need `SUPERUSER` or `CREATE`.
3. **Row-level (RLS)** — beyond table grants, RLS policies filter which *rows* a role sees. Critical for multi-tenant apps.

The first defense against SQL injection is **parameterized queries** (covered in [[14 - ADO.NET and Dapper]]). RLS, role separation, and audit are layers behind that.

A **role** in Postgres is both user and group — a unifying concept. Production typical setup: a `db_owner` role owns the schema/migrations, and an `app` role with only `SELECT/INSERT/UPDATE/DELETE` on specific tables runs the app.

---

## Syntax & API

### Roles and grants
```sql
-- Create a login role
CREATE ROLE app LOGIN PASSWORD 'secret';
ALTER ROLE app PASSWORD 'newsecret';      -- rotate
ALTER ROLE app VALID UNTIL '2026-12-31';  -- expire

-- Group role (no LOGIN; users granted INTO it inherit privileges)
CREATE ROLE app_read NOLOGIN;
GRANT app_read TO alice, bob;

-- Schema + table privileges
GRANT USAGE ON SCHEMA public TO app_read;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;

-- Apply to FUTURE tables too (otherwise new tables won't have grants)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_read;

-- Remove
REVOKE SELECT ON users FROM app_read;
DROP ROLE alice;
```

### Read-only / read-write app roles
```sql
CREATE ROLE app_rw LOGIN PASSWORD '…';
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_rw;
```

### Inspect privileges
```sql
\du                           -- roles
\l                            -- DBs + their owners
\dp users                     -- privileges on table users
\dn+ public                   -- schema privileges

SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'users';
```

### pg_hba.conf — connection rules
```text
# TYPE   DATABASE   USER         ADDRESS         METHOD
hostssl  all        all          0.0.0.0/0       scram-sha-256
host     all        all          10.0.0.0/8      scram-sha-256
hostssl  shop       app          ::1/128         cert clientcert=verify-full
local    all        postgres                     peer
```

```bash
# Reload after edits
sudo -u postgres pg_ctl reload
```

### Row-Level Security (RLS) — multi-tenant
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;     -- apply even to owners

-- A user can only see their own rows
CREATE POLICY orders_self ON orders
    FOR SELECT
    USING (user_id = current_setting('app.user_id')::INT);

CREATE POLICY orders_self_insert ON orders
    FOR INSERT
    WITH CHECK (user_id = current_setting('app.user_id')::INT);

-- App sets the user before each query (per-tx)
SET LOCAL app.user_id = 42;

-- Now SELECTs only return user 42's rows; INSERTs require user_id = 42
```

### Security-definer function (escalate privileges narrowly)
```sql
-- Grant a callable that does something the caller couldn't otherwise do
CREATE OR REPLACE FUNCTION admin_promote_user(p_id INT)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER                         -- runs as owner, not caller
SET search_path = public                 -- avoid PATH attacks
AS $$
BEGIN
    UPDATE users SET role = 'admin' WHERE id = p_id;
END;
$$;

REVOKE ALL ON FUNCTION admin_promote_user(INT) FROM PUBLIC;
GRANT  EXECUTE ON FUNCTION admin_promote_user(INT) TO support_role;
```

### Audit who did what
```sql
-- Built-in: log_statement = 'mod' → log all writes
-- Better: pgAudit extension for fine-grained logging
CREATE EXTENSION pgaudit;
-- postgresql.conf:
-- pgaudit.log = 'write, ddl'
```

---

## Common Patterns

```sql
-- Pattern: separation of migration role and app role
--   migrator: owns the schema, runs CREATE/ALTER/DROP
--   app:      only DML on tables, no DDL
CREATE ROLE migrator LOGIN PASSWORD '…';
ALTER SCHEMA public OWNER TO migrator;

CREATE ROLE app LOGIN PASSWORD '…';
GRANT USAGE ON SCHEMA public TO app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app;
```

```sql
-- Pattern: tenant scoping with current_setting
ALTER TABLE bills ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_iso ON bills
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- App per-request:
-- BEGIN; SET LOCAL app.tenant_id = '...'; ... COMMIT;
```

```bash
# Pattern: rotate passwords without downtime
-- 1. Issue new password to app
ALTER ROLE app PASSWORD 'newsecret';
-- 2. Deploy app config with new password
-- 3. Postgres still accepts old password until cache expires (re-auth)
-- For real rotation use SCRAM with two creds via auth proxies (PgBouncer + auth_query)
```

---

## Gotchas & Tips

- **Default `postgres` superuser is for ops only** — never use it for the app. Make a dedicated app role.
- **`ALTER DEFAULT PRIVILEGES` is per-grantor + schema** — it only affects objects created **by that role** afterwards. If migrations run as another role, set defaults for that role too.
- **Granting on schema isn't enough** — you also need `USAGE ON SCHEMA` and grants on individual tables/sequences/functions.
- **RLS doesn't apply to owners by default** — use `FORCE ROW LEVEL SECURITY` if the app role also owns the tables.
- **`SECURITY DEFINER` functions are dangerous** — `SET search_path` inside them, otherwise a malicious caller can hijack.
- **Don't store secrets in connection strings in source** — use environment variables, secret managers (Azure Key Vault, AWS Secrets Manager, Vault).
- **TLS is opt-in** — confirm with `SHOW ssl;` on the server. Set `sslmode=require` (or stronger) in connection strings.
- **Audit before incidents** — `log_connections`, `log_disconnections`, `log_statement = 'mod'` cost little and help forensics.
- **Lock down `pg_hba.conf`** — `host all all 0.0.0.0/0 trust` is the worst possible line.
- **Use `scram-sha-256`** — `md5` is legacy. Modern Postgres defaults to SCRAM.
- **Periodically `REASSIGN OWNED BY`** when removing users — orphaned objects break dumps.

---

## See Also

- [[14 - ADO.NET and Dapper]]
- [[17 - Encryption]]
- [[09 - Triggers]]
- [[17 - Cloud Databases]]
