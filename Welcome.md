---
tags: [home]
aliases: [Home, Start]
---

# Welcome

A structured, teacher-authored cheatsheet vault covering multiple technical topics from beginner to advanced.

## Pick a topic

| Topic | Index | Path | Start here |
| ----- | ----- | ---- | ---------- |
| .NET Core | [[Dotnet Master Index]] | [[Dotnet Learning Path]] | [[01 - Dotnet Overview]] |
| Database (PostgreSQL-primary) | [[Database Master Index]] | [[Database Learning Path]] | [[01 - Database Overview]] |
| React (modern, hooks + RSC) | [[React Master Index]] | [[React Learning Path]] | [[01 - React Overview]] |
| Angular (standalone, signals) | [[Angular Master Index]] | [[Angular Learning Path]] | [[01 - Angular Overview]] |
| Azure (cloud platform) | [[Azure Master Index]] | [[Azure Learning Path]] | [[01 - Azure Overview]] |
| Business (developer-facing) | [[Business Master Index]] | [[Business Learning Path]] | [[01 - Business Domain Overview]] |

---

## How notes are organized

Every note follows the same structure:

1. **Quick Reference** — scannable cheatsheet table
2. **Core Concept** — plain-English explanation
3. **Diagram** *(when helpful)* — Mermaid visual
4. **Syntax & API** — compilable code
5. **Common Patterns** — real-world usage
6. **Gotchas & Tips** — pitfalls
7. **See Also** — related notes

---

## Vault structure

```text
/
├── Dotnet/          ← .NET Core (48 notes)
│   ├── 01-Beginner/    (10 notes — language fundamentals)
│   ├── 02-Intermediate/ (20 notes — async, DI, web, data, runtime)
│   └── 03-Advanced/    (18 notes — architecture, performance, infra)
├── Database/        ← Databases (48 notes, PostgreSQL-primary)
│   ├── 01-Beginner/    (10 notes — SQL fundamentals)
│   ├── 02-Intermediate/ (20 notes — schema, transactions, tuning)
│   └── 03-Advanced/    (18 notes — distribution, analytics, AI)
├── React/           ← React (48 notes, hooks + RSC + frameworks)
│   ├── 01-Beginner/    (10 notes — JSX, components, state, effects)
│   ├── 02-Intermediate/ (20 notes — hooks deep dive, forms, routing, testing)
│   └── 03-Advanced/    (18 notes — internals, RSC, Next/Remix, perf, design systems)
├── Angular/         ← Angular (48 notes, standalone + signals + RxJS)
│   ├── 01-Beginner/    (10 notes — components, templates, DI basics)
│   ├── 02-Intermediate/ (20 notes — signals, RxJS, forms, routing, testing)
│   └── 03-Advanced/    (18 notes — internals, zoneless, SSR, NgRx, monorepo)
├── Azure/           ← Azure (48 notes, cloud platform)
│   ├── 01-Beginner/    (10 notes — resources, identity, compute/storage/networking basics)
│   ├── 02-Intermediate/ (20 notes — App Service, Functions, AKS, SQL/Cosmos, messaging, IaC)
│   └── 03-Advanced/    (18 notes — landing zones, observability, security, multi-region, AI)
└── Business/        ← Business domains (24 notes — finance, commerce, logistics)
    ├── 01-Beginner/    ( 8 notes — domain primers)
    ├── 02-Intermediate/( 8 notes — workflows, integrations, money math)
    └── 03-Advanced/    ( 8 notes — regulated and distributed-system concerns)
```

For shared authoring conventions see `CLAUDE.md` at vault root; per-topic rules live in each topic folder's own `CLAUDE.md`.
