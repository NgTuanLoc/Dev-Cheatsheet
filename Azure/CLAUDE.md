# CLAUDE.md — Azure Topic

Topic-specific guidance for the `Azure/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured Microsoft Azure teacher-authored cheatsheet/notebook — organized from beginner to advanced, intended for students learning cloud fundamentals on Azure and the broader platform (compute, storage, networking, identity, observability, IaC, AI).

---

## Scope & Version Policy

- **Azure portal as it stands 2025+** is the baseline; commands target the latest Azure CLI and Bicep.
- **Microsoft Entra ID** is used throughout — `Azure AD` is mentioned only as a historical name.
- **Bicep is the IaC default**; ARM JSON is shown only when Bicep can't express something or as a historical reference.
- **Azure CLI (`az`)** is the preferred tool over PowerShell `Az` modules. Both work; we standardize for consistency.
- **Cloud-agnostic where possible** — patterns are explained in vendor-neutral terms, then mapped to Azure services.
- **Cost transparency** — every advanced note flags pricing levers; cloud notes that ignore cost are misleading.

When a topic differs significantly across SKUs (Consumption vs Premium for Functions; DTU vs vCore for SQL), call it out in a callout sub-section.

---

## Folder Structure

```
Azure/
├── CLAUDE.md                       ← this file
├── 00-Index/
│   ├── Azure Master Index.md       ← single entry point, links to all Azure notes
│   └── Azure Learning Path.md      ← recommended reading order per level
├── 01-Beginner/                    (10 notes)
├── 02-Intermediate/                (20 notes)
└── 03-Advanced/                    (18 notes)
```

---

## Tag Taxonomy (Azure-specific)

Always include `azure` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `azure` | applied to every Azure note |
| `iaas` | infrastructure-as-a-service (VMs, disks, networking) |
| `paas` | platform-as-a-service (App Service, SQL DB, Cosmos) |
| `serverless` | Functions, Logic Apps, Container Apps consumption tier |
| `compute` | VMs, App Service, Functions, Container Apps, AKS |
| `storage` | Blob, File, Queue, Table, Disks |
| `networking` | VNet, NSG, peering, gateways, Front Door |
| `database` | Azure SQL, Postgres/MySQL Flexible, Cosmos DB |
| `identity` | Microsoft Entra ID, RBAC, managed identity |
| `security` | Key Vault, Defender, encryption, private endpoints |
| `monitoring` | Azure Monitor, App Insights, Log Analytics, KQL |
| `iac` | Bicep, ARM, Terraform on Azure |
| `devops` | Azure DevOps, GitHub Actions, deployment patterns |
| `cost` | pricing, budgets, reservations, FinOps |
| `architecture` | Well-Architected, landing zones, reference architectures |
| `ai` | Azure OpenAI, AI Foundry, ML, cognitive services |
| `messaging` | Service Bus, Event Grid, Event Hubs, Storage Queues |
| `containers` | AKS, Container Apps, Container Registry |
| `dr` | disaster recovery, geo-redundancy, backup |
| `dotnet` | .NET-on-Azure tie-ins (App Service, Functions, SDK) |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `bash` | Azure CLI commands (`az ...`), shell, Cloud Shell |
| `powershell` | `Az` PowerShell modules where bash CLI is awkward |
| `bicep` | Bicep templates |
| `json` | ARM templates, `azuredeploy.parameters.json`, REST samples |
| `csharp` | Azure SDK for .NET examples |
| `yaml` | GitHub Actions, Azure Pipelines YAML, Helm values |
| `kql` | Kusto Query Language (Log Analytics, App Insights) |
| `sql` | T-SQL for Azure SQL Database |
| `mermaid` | diagrams |

Examples must be **runnable end-to-end** in a personal subscription — every `az` command should work without imagined parameters. Prefer real default values to `<placeholders>` where it doesn't hurt clarity.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Azure Overview | Cloud models (IaaS/PaaS/SaaS), Azure vs AWS/GCP, regions, AZs, datacenter geography, shared responsibility |
| 02 | Azure Portal and CLI | Portal UI, Cloud Shell, `az` CLI install/auth, Azure PowerShell, ARM REST |
| 03 | Subscriptions Resource Groups and Tags | Management hierarchy (tenant → MG → sub → RG → resource), scope, tags for cost/ownership |
| 04 | Identity with Microsoft Entra ID | Tenants, users, groups, service principals, managed identities (intro), role assignments |
| 05 | Compute Options Overview | VM vs App Service vs Functions vs Container Apps vs AKS — when to pick which |
| 06 | Storage Basics | Storage accounts, Blob/File/Queue/Table services, redundancy (LRS/ZRS/GRS), access tiers |
| 07 | Networking Basics | VNet, subnet, NSG, public IP, DNS, address spaces, peering intro |
| 08 | Database Options | Azure SQL DB, Azure Database for Postgres/MySQL, Cosmos DB, Cache for Redis — picking the engine |
| 09 | Cost Management | Pricing calculator, cost analysis, budgets, alerts, reservations vs savings plans, free tier |
| 10 | IaC with ARM and Bicep | What IaC is, ARM JSON, Bicep syntax, modules, deployment, what-if |

### Intermediate (02-Intermediate) — 20 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | App Service Deep Dive | Plans, deployment slots, scaling rules, custom domains, TLS, identity, networking integration |
| 02 | Azure Functions | Hosting plans (Consumption/Premium/Dedicated), triggers, bindings, Durable Functions, isolated worker |
| 03 | Container Apps | KEDA scaling, revisions, ingress, environments, secrets, Dapr integration |
| 04 | AKS Basics | Cluster + node pools, system vs user pools, kubectl access, ingress, image pull from ACR |
| 05 | Container Registry | ACR tiers, ACR Tasks, content trust, geo-replication, vulnerability scanning |
| 06 | Blob Storage Advanced | Lifecycle management, SAS tokens, soft delete, versioning, immutability, change feed |
| 07 | Azure SQL Database | DTU vs vCore, Hyperscale, elastic pools, geo-replication, failover groups |
| 08 | Cosmos DB | APIs (NoSQL, Mongo, Cassandra, Gremlin, Table), partitioning, RU/s, consistency levels |
| 09 | Azure Postgres Flexible Server | Compute tiers, HA modes, read replicas, extensions, connection pooling with PgBouncer |
| 10 | Azure Cache for Redis | Tiers, clustering, persistence, geo-replication, Redis 6 ACLs |
| 11 | Service Bus | Queues, topics + subscriptions, sessions, dead-letter, duplicate detection, scheduled |
| 12 | Event Grid | Push-based, system vs custom topics, event schemas, filtering, retries, dead-letter |
| 13 | Event Hubs | Pull-based streaming, partitions, consumer groups, checkpointing, Capture |
| 14 | Storage Queues vs Service Bus | When to use the cheap simple queue vs the rich broker |
| 15 | Key Vault | Secrets, keys, certificates, access policies vs RBAC, network restrictions, soft delete |
| 16 | Managed Identity | System-assigned vs user-assigned, federated identity credentials (workload identity), token flow |
| 17 | VNet and Subnets | Peering, service endpoints, private endpoints, NSG rules, ASG, route tables |
| 18 | Application Gateway and Load Balancer | L4 vs L7, WAF, path-based routing, backend pools, health probes |
| 19 | Azure Front Door and CDN | Global edge routing, caching rules, WAF, custom domains, multi-origin |
| 20 | Bicep Modules | Module patterns, parameters, outputs, registries, reusable building blocks |

### Advanced (03-Advanced) — 18 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Well-Architected Framework | Five pillars (reliability, security, cost, ops, performance) + how to use them in reviews |
| 02 | Landing Zones | Enterprise-scale baseline, management groups, policy, network topology, identity |
| 03 | AKS Production Patterns | Cluster autoscaler, node pool strategies, network plugins (Azure CNI vs Overlay), ingress, mTLS |
| 04 | Serverless Architectures | Functions + Service Bus + Cosmos DB patterns, cold-start mitigation, durable orchestrations |
| 05 | Microservices on Azure | Picking ACA vs AKS vs App Service, service discovery, communication, deployment |
| 06 | Distributed Tracing with OpenTelemetry | OTel SDK in .NET, exporter to App Insights, sampling, span links, correlation |
| 07 | Azure Monitor and Log Analytics | Workspaces, data sources, KQL queries, dashboards, alerts, action groups |
| 08 | Application Insights Deep Dive | Auto-instrumentation, sampling, dependency tracking, snapshot debugger, live metrics |
| 09 | RBAC and Azure Policy | Built-in vs custom roles, scope, deny assignments, policy initiative, remediation |
| 10 | Defender for Cloud and Sentinel | CSPM, CWPP, attack paths, JIT, regulatory compliance, Sentinel SIEM/SOAR |
| 11 | Hub and Spoke Networking | Hub-spoke topology, vWAN, firewall, segmentation, transitive routing |
| 12 | Private Endpoints and Zero Trust | PaaS without public IPs, private DNS zones, network isolation patterns |
| 13 | Multi-Region HA | Active-active vs active-passive, traffic manager vs Front Door, data replication strategies |
| 14 | Disaster Recovery | RPO/RTO, Azure Site Recovery, geo-redundant storage, backup vault, failover drills |
| 15 | CI-CD on Azure | GitHub Actions + OIDC federation, Azure Pipelines, environments, approvals, blue/green |
| 16 | Cost Optimization at Scale | Reservations, savings plans, autoscaling, rightsizing, hibernation, FinOps practices |
| 17 | Event-Driven Architecture | Event Grid + Event Hubs + Functions composition, idempotency, ordering, replays |
| 18 | AI and Azure OpenAI | OpenAI Service, deployment types, content filters, AI Foundry, RAG patterns with Cognitive Search |

**Total: 48 notes** (10 beginner + 20 intermediate + 18 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| Azure Overview | graph (regions, paired regions, AZs hierarchy) |
| Subscriptions Resource Groups and Tags | graph (tenant → MG → sub → RG → resource hierarchy) |
| Identity with Microsoft Entra ID | sequenceDiagram (OIDC handshake with Entra ID) |
| Compute Options Overview | flowchart (decision tree: which compute service) |
| Storage Basics | graph (storage account → 4 services → redundancy options) |
| Networking Basics | graph (VNet topology with subnets/NSG/peering) |
| Database Options | flowchart (decision tree: which DB service) |
| IaC with ARM and Bicep | sequenceDiagram (Bicep → ARM JSON → ARM API → resources) |
| App Service Deep Dive | graph (plan → app → slots → scaling) |
| Azure Functions | flowchart (trigger types + binding directions) |
| Container Apps | graph (environment → app → revisions → ingress) |
| AKS Basics | graph (control plane + node pools + kubectl) |
| Service Bus | sequenceDiagram (sender → queue/topic → receivers) |
| Event Grid | sequenceDiagram (publisher → topic → handler with retry) |
| Event Hubs | graph (producer → partitions → consumer groups) |
| Key Vault | sequenceDiagram (app → MI → Entra → Key Vault token flow) |
| Managed Identity | sequenceDiagram (workload → IMDS → Entra → token → resource) |
| VNet and Subnets | graph (hub-spoke + peering) |
| Application Gateway and Load Balancer | graph (Front Door → AGW → backend pools) |
| Bicep Modules | flowchart (main → modules → deployment scopes) |
| Well-Architected Framework | graph (five pillars) |
| Landing Zones | graph (enterprise-scale topology) |
| AKS Production Patterns | graph (production AKS topology) |
| Serverless Architectures | sequenceDiagram (HTTP → queue → durable → output) |
| Distributed Tracing with OpenTelemetry | sequenceDiagram (span propagation across services) |
| Azure Monitor and Log Analytics | graph (data sources → workspace → KQL → alerts/dashboards) |
| Hub and Spoke Networking | graph (hub + spokes + firewall) |
| Private Endpoints and Zero Trust | sequenceDiagram (client → private DNS → private endpoint → PaaS) |
| Multi-Region HA | graph (active-active topology with traffic manager / Front Door) |
| Disaster Recovery | flowchart (RPO/RTO targets → backup/replication choice) |
| Event-Driven Architecture | graph (producers → broker → consumers) |
| AI and Azure OpenAI | sequenceDiagram (RAG flow: query → search → LLM → response) |
