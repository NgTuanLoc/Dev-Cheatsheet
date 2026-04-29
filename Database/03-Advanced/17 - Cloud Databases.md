---
tags: [database, advanced, cloud, postgresql]
aliases: [RDS, Aurora, Azure Database, Cosmos DB, Cloud SQL]
level: Advanced
---

# Cloud Databases

> **One-liner**: Managed databases trade tuning knobs and root for predictable operations — backups, HA, patching, security baselines come for free; the cost is vendor lock-in and per-hour billing.

---

## Quick Reference

| Service | Provider | Engine |
|---------|----------|--------|
| **Azure Database for PostgreSQL** (Flexible Server) | Azure | Postgres |
| **Azure SQL Database / Managed Instance** | Azure | SQL Server |
| **Cosmos DB** | Azure | multi-API: NoSQL, Mongo, Cassandra, Gremlin, Table |
| **Amazon RDS** | AWS | Postgres / MySQL / MariaDB / Oracle / SQL Server |
| **Amazon Aurora** | AWS | Postgres- / MySQL-compatible; cloud-native storage |
| **DynamoDB** | AWS | NoSQL key-value/doc; massive scale |
| **Cloud SQL** | GCP | Postgres / MySQL / SQL Server |
| **AlloyDB** | GCP | Postgres-compatible; columnar accelerator |
| **Spanner** | GCP | globally distributed, strongly consistent SQL |
| **Neon / Supabase / Crunchy Bridge** | independents | hosted Postgres |

| Operational concern | Cloud handles | You still own |
|---------------------|---------------|---------------|
| Backups, PITR | yes | retention policy |
| HA, failover | yes | application retry on failover |
| OS patching | yes | DB version upgrades (you trigger) |
| TLS, encryption-at-rest | yes (defaults on) | KMS key strategy, app TLS verification |
| Indexing, schema, queries | no | all on you |
| Capacity planning | partly (autoscale) | budget + access patterns |
| Network / VPC | partly | private endpoints, peering |

---

## Core Concept

A managed cloud DB is a **service-shaped DB**. The provider owns the OS, the binaries, replication, backups, and the on-call rotation that responds to disk-full alerts. You own the data model, the queries, and most of the security configuration.

Three families of cloud Postgres-compatible DBs:

1. **Lift-and-shift managed** (RDS Postgres, Cloud SQL, Azure Database for PostgreSQL) — vanilla Postgres on managed VMs. Familiar tools work; some `superuser` features are off-limits.
2. **Cloud-native re-architected** (Aurora, AlloyDB) — Postgres compute on a custom storage engine that replicates synchronously across multiple AZs at the storage layer. Faster for some workloads; less predictable in pricing.
3. **Globally distributed** (Spanner, CockroachDB Cloud, Yugabyte) — spans regions with strong consistency. The killer feature is geo-distribution; the cost is unfamiliar performance characteristics.

For NoSQL, the canonical cloud picks are:
- **DynamoDB** — fastest at scale; design for access patterns up front
- **Cosmos DB** — multi-API, multi-region, tunable consistency (5 levels)
- **Firestore / Datastore** — GCP's document store

Pick by where your team and traffic are. Cross-cloud migrations are painful; multi-cloud is rarely worth it for the DB tier.

---

## Syntax & API

### Provision Postgres on Azure (Flexible Server)
```bash
az postgres flexible-server create \
    --resource-group myRG \
    --name myshop \
    --location eastus \
    --admin-user shopadmin --admin-password 'StrongP@ss!' \
    --sku-name Standard_D2ds_v4 \
    --tier GeneralPurpose \
    --storage-size 64 \
    --version 16 \
    --high-availability ZoneRedundant \
    --backup-retention 14 \
    --geo-redundant-backup Enabled

# Allow your IP
az postgres flexible-server firewall-rule create \
    --resource-group myRG --name myshop \
    --rule-name "my-ip" --start-ip-address $(curl -s ifconfig.me) --end-ip-address $(curl -s ifconfig.me)

# Connect
psql "host=myshop.postgres.database.azure.com user=shopadmin dbname=postgres sslmode=require"
```

### RDS Postgres on AWS
```bash
aws rds create-db-instance \
    --db-instance-identifier myshop \
    --db-instance-class db.t4g.medium \
    --engine postgres --engine-version 16.2 \
    --allocated-storage 50 \
    --master-username shopadmin --master-user-password 'StrongP@ss!' \
    --vpc-security-group-ids sg-12345 \
    --backup-retention-period 14 \
    --multi-az \
    --storage-encrypted
```

### Cosmos DB (NoSQL API) — multi-region, tunable consistency
```bash
az cosmosdb create \
    --resource-group myRG --name shop-cosmos \
    --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
    --locations regionName=westeurope failoverPriority=1 \
    --default-consistency-level Session \
    --enable-multiple-write-locations false
```

```javascript
// Cosmos DB SQL API
const { CosmosClient } = require('@azure/cosmos');
const client = new CosmosClient({ endpoint, key });
const container = client.database('shop').container('orders');

// Strongly consistent read for this op
const { resource } = await container.item(id, partitionKey)
    .read({ consistencyLevel: 'Strong' });
```

### DynamoDB — designed for access patterns
```bash
# Single-table design (typical)
aws dynamodb create-table \
    --table-name shop \
    --attribute-definitions AttributeName=PK,AttributeType=S AttributeName=SK,AttributeType=S \
    --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST
```

```csharp
// AWS SDK for .NET — query by partition key
var resp = await client.QueryAsync(new QueryRequest {
    TableName = "shop",
    KeyConditionExpression = "PK = :pk AND begins_with(SK, :prefix)",
    ExpressionAttributeValues = {
        [":pk"] = new AttributeValue("USER#42"),
        [":prefix"] = new AttributeValue("ORDER#")
    }
});
```

### Aurora — same as RDS API; pay attention to:
```text
- Storage scales 10GB → 128TB automatically
- Read replicas share storage (no replication lag for state, only WAL apply)
- Aurora Serverless v2 scales compute by ACU
- Global Database = cross-region read replicas with sub-second lag
- I/O-Optimized vs Standard pricing — pick based on read:write ratio
```

### Connection patterns
```text
Always-on host endpoints:
  - <name>.<region>.rds.amazonaws.com
  - <name>.postgres.database.azure.com
  - <name>:<region>:<instance> (Cloud SQL)

Always require TLS:
  sslmode=verify-full + ca-bundle.pem

For Aurora and Cosmos DB:
  Use the writer endpoint for writes, reader endpoints for reads
  Or use a connection pooler (RDS Proxy, PgBouncer) for serverless apps
```

### Inspect / manage
```sql
-- Most cloud Postgres restricts SUPERUSER
-- Use rds_superuser / azure_pg_admin / cloudsqlsuperuser instead
GRANT rds_superuser TO myadmin;            -- AWS
GRANT azure_pg_admin TO myadmin;           -- Azure

-- See enabled extensions allow-list
SELECT * FROM pg_available_extensions ORDER BY name;
```

### Networking — private endpoints
```bash
# AWS PrivateLink for RDS
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-12345 \
    --service-name com.amazonaws.us-east-1.rds \
    --vpc-endpoint-type Interface

# Azure Private Endpoint for Postgres
az network private-endpoint create \
    --resource-group myRG --name pg-private \
    --vnet-name myVnet --subnet mySubnet \
    --private-connection-resource-id <postgres-resource-id> \
    --group-ids postgresqlServer --connection-name pg-conn
```

---

## Common Patterns

```text
Pattern: serverless app + RDS Proxy / connection pooler
- Lambda / Functions create thousands of cold connections
- RDS Proxy / PgBouncer multiplexes them down to a sustainable pool
- See [[13 - Connection Management]]
```

```text
Pattern: read replica for analytics
- Reports / dashboards on a replica
- Primary handles OLTP
- Lag tolerance per workload
```

```text
Pattern: blue/green deployment for upgrades
- AWS RDS Blue/Green Deployments
- Azure Flexible Server uses Logical Replication for major version upgrades
- Test on green, switch traffic, retire blue
```

```text
Pattern: per-region Cosmos DB / DynamoDB Global Tables
- Multi-master, last-writer-wins (or app-defined merge)
- Sub-second cross-region replication
- Local reads everywhere; eventual consistency between regions
```

---

## Gotchas & Tips

- **Read the SLA carefully** — "99.99%" is monthly; that's still 4 minutes/month of allowed downtime. Distinguish DB SLA from app SLA.
- **Cloud `superuser` is restricted** — many extensions, file-system access, and OS-level tools are off-limits. Plan for this.
- **Egress is the silent cost** — moving 1TB out of your cloud is expensive. Keep replicas, backups, and analytics in-region.
- **Backups are configured, not created** — verify retention, point-in-time recovery, and test restores. Provider defaults are sane, not always sufficient.
- **Storage scales automatically; compute does not** (mostly) — DB instances need manual resizing or scheduled maintenance. Aurora Serverless / Cosmos / DynamoDB can autoscale compute.
- **TLS is on by default — verify-full it from the client** — don't accept self-signed in production.
- **Maintenance windows can cause failover** — set them to off-peak; expect brief outages during minor version patches.
- **`pg_dump` from RDS works**, but very large dumps are I/O-bound. Use snapshot exports to S3 for huge backups.
- **Cosmos DB consistency levels** — Session is the practical sweet spot; Strong is expensive across regions.
- **DynamoDB billing models matter** — On-Demand vs Provisioned; the wrong choice costs 5–10×.
- **Don't confuse "Multi-AZ" with "read replica"** — in RDS, Multi-AZ is for HA (sync, not used for reads); Replicas are async + readable.
- **Aurora I/O-Optimized** has flat I/O pricing — better for I/O-heavy workloads even though base hourly is higher.
- **Rate-limit your migrations** — `dotnet ef database update` from a function in production can throttle a small instance.

---

## See Also

- [[03 - High Availability and Failover]]
- [[02 - Replication]]
- [[15 - Backup and Restore]]
- [[16 - Database Security]]
