---
tags: [database, intermediate, migrations, postgresql, dotnet]
aliases: [Schema Migrations, EF Core Migrations, Flyway, DbUp]
level: Intermediate
---

# Database Migrations

> **One-liner**: Migrations are versioned, ordered SQL scripts that evolve a database's schema in lockstep with the application — checked into git like code.

---

## Quick Reference

| Tool | Stack | Style |
|------|-------|-------|
| **EF Core Migrations** | .NET | Auto-generated from C# model diff |
| **FluentMigrator** | .NET | Hand-written C# fluent DSL |
| **DbUp** | .NET | Plain SQL files in order, idempotent |
| **Flyway** | Java/Polyglot | Versioned `V1__name.sql` files |
| **Liquibase** | Java/Polyglot | XML/YAML/SQL changelogs |
| **Sqitch** | Polyglot | Verify scripts + dependency model |
| **Alembic** | Python | Auto-generated diff (SQLAlchemy) |
| **goose / migrate** | Go | Plain SQL up/down |

| Concept | Meaning |
|---------|---------|
| **Migration** | One ordered, immutable script (forward) |
| **Up / Down** | Apply / revert (down is optional in many tools) |
| **Idempotent** | Re-runnable without changing the result |
| **Schema history table** | Records which migrations have been applied |
| **Forward-only** | Never edit applied migrations; create a new one |

---

## Core Concept

A migration system tracks which SQL has been applied to a database, so multiple developers and environments evolve the schema in the same order. The DB has a small **history table** (`__EFMigrationsHistory`, `flyway_schema_history`, etc.); the tool runs scripts whose IDs aren't in that table, in order.

Two big philosophies:

1. **Code-first generation** (EF Core, Alembic) — you change the model, the tool diffs against the snapshot, and writes migration code. Fast for app-driven work; can produce surprising SQL.
2. **SQL-first** (Flyway, DbUp, goose) — you write the SQL. Slower to author, but explicit and reviewable.

**Forward-only** is the iron rule: never edit a migration after it's been applied to a shared environment. Make a new one to fix mistakes. Editing in-place leads to environments diverging silently.

For zero-downtime deploys, schema changes need to be **backwards-compatible**: deploy migration → deploy app code that uses new schema → cleanup migration. See [[08 - Schema Evolution]].

---

## Syntax & API

### EF Core (.NET) — quick path
```bash
dotnet tool install -g dotnet-ef                # one-time

# Add a migration based on model changes
dotnet ef migrations add AddUserPhone

# Apply pending migrations
dotnet ef database update

# Roll back to a previous migration
dotnet ef database update PreviousMigrationName

# Generate SQL script (for review / production deploy)
dotnet ef migrations script
dotnet ef migrations script PreviousMigration  # incremental
dotnet ef migrations script --idempotent       # safe to re-apply

# Remove the latest migration (only if NOT applied)
dotnet ef migrations remove
```

### EF Core migration (auto-generated)
```csharp
public partial class AddUserPhone : Migration
{
    protected override void Up(MigrationBuilder b)
    {
        b.AddColumn<string>(
            name: "phone",
            table: "users",
            type: "text",
            nullable: true);

        b.CreateIndex(
            name: "ix_users_phone",
            table: "users",
            column: "phone");
    }

    protected override void Down(MigrationBuilder b)
    {
        b.DropIndex(name: "ix_users_phone", table: "users");
        b.DropColumn(name: "phone", table: "users");
    }
}
```

### DbUp (plain SQL files)
```csharp
// Program.cs
var upgrader = DeployChanges.To
    .PostgresqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
    .WithTransactionPerScript()
    .LogToConsole()
    .Build();

var result = upgrader.PerformUpgrade();
if (!result.Successful) { Console.Error.WriteLine(result.Error); return 1; }
```

```sql
-- Scripts/0001_create_users.sql
CREATE TABLE users (
    id    INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name  TEXT NOT NULL
);
```

```sql
-- Scripts/0002_add_phone.sql
ALTER TABLE users ADD COLUMN phone TEXT;
CREATE INDEX ix_users_phone ON users(phone);
```

### Flyway (engine-agnostic)
```text
db/migration/
    V1__create_users.sql
    V2__add_phone.sql
    R__refresh_views.sql        -- "repeatable" — re-runs when changed
```

```bash
flyway -url=jdbc:postgresql://localhost:5432/shop -user=postgres -password=secret migrate
flyway info                                                                           # status
flyway validate                                                                       # check checksums
```

### FluentMigrator (.NET) — explicit DSL
```csharp
[Migration(202604290001)]
public class AddUserPhone : Migration
{
    public override void Up()
    {
        Alter.Table("users").AddColumn("phone").AsString().Nullable();
        Create.Index("ix_users_phone").OnTable("users").OnColumn("phone");
    }

    public override void Down()
    {
        Delete.Index("ix_users_phone").OnTable("users");
        Delete.Column("phone").FromTable("users");
    }
}
```

### Idempotent SQL pattern (for hand-written scripts)
```sql
CREATE TABLE IF NOT EXISTS users (...);

ALTER TABLE users ADD COLUMN IF NOT EXISTS phone TEXT;

DO $$ BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_indexes WHERE indexname = 'ix_users_phone') THEN
        CREATE INDEX ix_users_phone ON users(phone);
    END IF;
END $$;
```

---

## Common Patterns

```sql
-- Pattern: zero-downtime add NOT NULL column
-- Step 1 (migration A): add nullable column with default
ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';

-- Step 2 (deploy): app starts writing the new column

-- Step 3 (migration B, after backfill done): make it required
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
ALTER TABLE users ALTER COLUMN status DROP DEFAULT;
```

```sql
-- Pattern: rename via expand/contract (so old code keeps working during deploy)
-- Migration 1: add new column
ALTER TABLE users ADD COLUMN full_name TEXT;
UPDATE users SET full_name = name;
-- Migration 2 (after app reads from full_name and writes to both)
-- Migration 3: drop old column
ALTER TABLE users DROP COLUMN name;
```

```bash
# Pattern: separate generate vs apply for production
dotnet ef migrations script --idempotent --output deploy.sql
# → review deploy.sql in PR
# → apply via DBA / pipeline tool, not the app's runtime EF
```

---

## Gotchas & Tips

- **Never edit an applied migration** — always create a new one. Editing means production and dev diverge.
- **Don't generate migrations against an empty model on an existing DB** — EF Core will try to create everything from scratch. Use `dotnet ef dbcontext scaffold` to reverse-engineer first.
- **Long-running migrations lock tables** — adding a NOT NULL column with a default scans every row. Postgres 11+ does this in O(1) for new columns; older versions don't.
- **`CREATE INDEX` blocks writes** — use `CREATE INDEX CONCURRENTLY`, but EF Core doesn't emit it by default. Hand-edit migrations or use raw SQL.
- **Test migrations on production-sized data** — what's instant on 1k rows can be hours on 100M rows.
- **Down migrations rot fast** — most teams treat migrations as forward-only; rollback by writing a *new* corrective migration.
- **Snapshot file conflicts in EF Core** — `<Project>ModelSnapshot.cs` is the source of truth; merge conflicts there are real bugs. Resolve by re-generating, not hand-merging.
- **Don't run `database update` from the app at startup in production** — race conditions when multiple instances start. Run it as a one-shot job in the deploy pipeline.
- **Postgres DDL is transactional** (with a few exceptions like `CREATE INDEX CONCURRENTLY`). Wrap multi-step migrations in a transaction so partial failures roll back.
- **Pre-commit checks** — `dotnet ef migrations script` emits SQL. Diff it in PRs to catch surprises.

---

## See Also

- [[08 - Schema Evolution]]
- [[14 - ADO.NET and Dapper]]
- [[02 - Transactions and ACID]]
- [[16 - Database Security]]
