---
tags: [azure, index]
aliases: [Azure Index, Azure Home]
---

# Azure Master Index — Cloud Cheatsheet

> **Single entry point** for the Azure topic. Browse by level or jump straight to a service area.

---

## Start Here

- [[Azure Learning Path]] — recommended reading order
- New to Azure? Begin with [[01 - Azure Overview]]

---

## 01 — Beginner (foundations)

Cloud fundamentals, Azure-specific concepts, and the resource hierarchy.

| # | Note | What you'll learn |
|---|------|-------------------|
| 01 | [[01 - Azure Overview]] | Cloud models, regions, AZs, shared responsibility |
| 02 | [[02 - Azure Portal and CLI]] | Portal, `az` CLI, Cloud Shell, PowerShell |
| 03 | [[03 - Subscriptions Resource Groups and Tags]] | Management hierarchy and tagging discipline |
| 04 | [[04 - Identity with Microsoft Entra ID]] | Tenants, users, service principals, MI intro |
| 05 | [[05 - Compute Options Overview]] | VM vs App Service vs Functions vs ACA vs AKS |
| 06 | [[06 - Storage Basics]] | Storage accounts, redundancy, blob/file/queue/table |
| 07 | [[07 - Networking Basics]] | VNet, subnet, NSG, public IP, DNS |
| 08 | [[08 - Database Options]] | Azure SQL, Postgres, Cosmos, Redis — when to pick |
| 09 | [[09 - Cost Management]] | Pricing calculator, budgets, reservations |
| 10 | [[10 - IaC with ARM and Bicep]] | Bicep basics, deployment, what-if |

---

## 02 — Intermediate (production-ready services)

Deep dives into the most-used services across compute, data, messaging, and networking.

### Compute and containers
| # | Note |
|---|------|
| 01 | [[01 - App Service Deep Dive]] |
| 02 | [[02 - Azure Functions]] |
| 03 | [[03 - Container Apps]] |
| 04 | [[04 - AKS Basics]] |
| 05 | [[05 - Container Registry]] |

### Storage and data
| # | Note |
|---|------|
| 06 | [[06 - Blob Storage Advanced]] |
| 07 | [[07 - Azure SQL Database]] |
| 08 | [[08 - Cosmos DB]] |
| 09 | [[09 - Azure Postgres Flexible Server]] |
| 10 | [[10 - Azure Cache for Redis]] |

### Messaging and events
| # | Note |
|---|------|
| 11 | [[11 - Service Bus]] |
| 12 | [[12 - Event Grid]] |
| 13 | [[13 - Event Hubs]] |
| 14 | [[14 - Storage Queues vs Service Bus]] |

### Security and identity
| # | Note |
|---|------|
| 15 | [[15 - Key Vault]] |
| 16 | [[16 - Managed Identity]] |

### Networking
| # | Note |
|---|------|
| 17 | [[17 - VNet and Subnets]] |
| 18 | [[18 - Application Gateway and Load Balancer]] |
| 19 | [[19 - Azure Front Door and CDN]] |

### IaC
| # | Note |
|---|------|
| 20 | [[20 - Bicep Modules]] |

---

## 03 — Advanced (architecture and operations)

Architecture frameworks, production-grade patterns, observability, security, cost, and AI.

### Architecture and governance
| # | Note |
|---|------|
| 01 | [[01 - Well-Architected Framework]] |
| 02 | [[02 - Landing Zones]] |
| 09 | [[09 - RBAC and Azure Policy]] |

### Compute architectures
| # | Note |
|---|------|
| 03 | [[03 - AKS Production Patterns]] |
| 04 | [[04 - Serverless Architectures]] |
| 05 | [[05 - Microservices on Azure]] |

### Observability
| # | Note |
|---|------|
| 06 | [[06 - Distributed Tracing with OpenTelemetry]] |
| 07 | [[07 - Azure Monitor and Log Analytics]] |
| 08 | [[08 - Application Insights Deep Dive]] |

### Security
| # | Note |
|---|------|
| 10 | [[10 - Defender for Cloud and Sentinel]] |
| 12 | [[12 - Private Endpoints and Zero Trust]] |

### Networking and resilience
| # | Note |
|---|------|
| 11 | [[11 - Hub and Spoke Networking]] |
| 13 | [[13 - Multi-Region HA]] |
| 14 | [[14 - Disaster Recovery]] |

### Delivery and cost
| # | Note |
|---|------|
| 15 | [[15 - CI-CD on Azure]] |
| 16 | [[16 - Cost Optimization at Scale]] |

### Modern workloads
| # | Note |
|---|------|
| 17 | [[17 - Event-Driven Architecture]] |
| 18 | [[18 - AI and Azure OpenAI]] |

---

## Quick Topic Routes

- **Choosing compute**: [[05 - Compute Options Overview]] → [[01 - App Service Deep Dive]] / [[02 - Azure Functions]] / [[03 - Container Apps]] / [[04 - AKS Basics]]
- **Choosing data**: [[08 - Database Options]] → [[07 - Azure SQL Database]] / [[08 - Cosmos DB]] / [[09 - Azure Postgres Flexible Server]]
- **Security mindset**: [[04 - Identity with Microsoft Entra ID]] → [[16 - Managed Identity]] → [[15 - Key Vault]] → [[12 - Private Endpoints and Zero Trust]]
- **Cost discipline**: [[09 - Cost Management]] → [[16 - Cost Optimization at Scale]]
- **Going to prod**: [[01 - Well-Architected Framework]] → [[02 - Landing Zones]] → [[13 - Multi-Region HA]] → [[15 - CI-CD on Azure]]
