---
tags: [azure, index]
aliases: [Azure Path]
---

# Azure Learning Path

A recommended reading order. The Beginner section is sequential — skip nothing. Intermediate and Advanced can be read by interest area.

---

## Stage 1 — Foundations (read in order)

1. [[01 - Azure Overview]] — what Azure is, regions, the shared-responsibility model.
2. [[02 - Azure Portal and CLI]] — get hands on the portal and `az` CLI.
3. [[03 - Subscriptions Resource Groups and Tags]] — the hierarchy that organizes everything else.
4. [[04 - Identity with Microsoft Entra ID]] — every resource lives in a tenant; learn the identity model.
5. [[05 - Compute Options Overview]] — the question "where does my code run?" answered.
6. [[06 - Storage Basics]] — durable storage primitives.
7. [[07 - Networking Basics]] — VNets, subnets, NSGs.
8. [[08 - Database Options]] — picking the right managed database.
9. [[09 - Cost Management]] — keep a personal subscription from surprises.
10. [[10 - IaC with ARM and Bicep]] — never click your way to production.

After Stage 1 you should be able to: deploy a small web app with a database, secure it with Entra ID, and tear it down without leaving stray resources.

---

## Stage 2 — Production-grade services (read by track)

Pick the track most relevant to your work; read others later.

### Track A — Web and APIs
- [[01 - App Service Deep Dive]]
- [[02 - Azure Functions]]
- [[18 - Application Gateway and Load Balancer]]
- [[19 - Azure Front Door and CDN]]
- [[15 - Key Vault]]
- [[16 - Managed Identity]]

### Track B — Containers and microservices
- [[03 - Container Apps]]
- [[04 - AKS Basics]]
- [[05 - Container Registry]]
- [[15 - Key Vault]]
- [[16 - Managed Identity]]

### Track C — Data
- [[07 - Azure SQL Database]]
- [[08 - Cosmos DB]]
- [[09 - Azure Postgres Flexible Server]]
- [[10 - Azure Cache for Redis]]
- [[06 - Blob Storage Advanced]]

### Track D — Eventing and messaging
- [[11 - Service Bus]]
- [[12 - Event Grid]]
- [[13 - Event Hubs]]
- [[14 - Storage Queues vs Service Bus]]

### Track E — IaC and networking
- [[20 - Bicep Modules]]
- [[17 - VNet and Subnets]]

---

## Stage 3 — Architecture and operations

Read after you've shipped at least one production workload.

### Architecture first
1. [[01 - Well-Architected Framework]] — the lens for every design decision.
2. [[02 - Landing Zones]] — how enterprises arrange Stage 1 concepts at scale.
3. [[09 - RBAC and Azure Policy]] — governance and guardrails.

### Then pick depth tracks
- **Operations**: [[06 - Distributed Tracing with OpenTelemetry]] → [[07 - Azure Monitor and Log Analytics]] → [[08 - Application Insights Deep Dive]] → [[15 - CI-CD on Azure]]
- **Resilience**: [[13 - Multi-Region HA]] → [[14 - Disaster Recovery]]
- **Security**: [[10 - Defender for Cloud and Sentinel]] → [[11 - Hub and Spoke Networking]] → [[12 - Private Endpoints and Zero Trust]]
- **Architecture choices**: [[03 - AKS Production Patterns]] → [[04 - Serverless Architectures]] → [[05 - Microservices on Azure]]
- **Cost and modernization**: [[16 - Cost Optimization at Scale]] → [[17 - Event-Driven Architecture]] → [[18 - AI and Azure OpenAI]]

---

## How to use this path

- **Build something along the way.** Each beginner note has runnable `az` commands; do them in a personal subscription.
- **Delete what you create.** Always tag practice resources with `purpose=learning` and prune weekly.
- **Don't memorize SKUs.** Memorize the trade-off questions; SKUs change every quarter.
- **Read the official docs alongside.** This vault is a cheatsheet, not a substitute for `learn.microsoft.com`.
