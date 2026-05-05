# CLAUDE.md — Vault-Wide Conventions

This file provides **shared** guidance for the entire Obsidian vault. Topic-specific rules (.NET, Database, etc.) live in per-topic `CLAUDE.md` files at each topic root.

## What This Vault Is

A multi-topic, structured, teacher-authored cheatsheet/notebook. Each top-level folder is one technical topic, organized **beginner → intermediate → advanced**, intended for students.

### Topics

| Folder | Topic | Per-topic instructions |
|--------|-------|------------------------|
| `Dotnet/` | .NET Core (C#, ASP.NET Core, EF Core, runtime) | [Dotnet/CLAUDE.md](./Dotnet/CLAUDE.md) |
| `Database/` | Databases (PostgreSQL-primary, SQL & NoSQL) | [Database/CLAUDE.md](./Database/CLAUDE.md) |
| `React/` | React (hooks, RSC, Next.js, Remix, ecosystem) | [React/CLAUDE.md](./React/CLAUDE.md) |

---

## Top-Level Vault Structure

```
/
├── Welcome.md              ← topic router (entry point)
├── CLAUDE.md               ← this file (shared rules)
├── _Templates/
│   └── Note Template.md    ← canonical note structure (shared)
├── Dotnet/
│   ├── CLAUDE.md           ← .NET-specific rules + coverage map
│   ├── 00-Index/
│   ├── 01-Beginner/
│   ├── 02-Intermediate/
│   └── 03-Advanced/
├── Database/
│   ├── CLAUDE.md           ← Database-specific rules + coverage map
│   ├── 00-Index/
│   ├── 01-Beginner/
│   ├── 02-Intermediate/
│   └── 03-Advanced/
└── React/
    ├── CLAUDE.md           ← React-specific rules + coverage map
    ├── 00-Index/
    ├── 01-Beginner/
    ├── 02-Intermediate/
    └── 03-Advanced/
```

Adding a new topic: create `<Topic>/`, mirror the four-folder structure, write a `<Topic>/CLAUDE.md` with the topic's coverage map, and link it from `Welcome.md` and this file.

---

## Diagrams — Mermaid (text-based, native Obsidian rendering)

Diagrams are written as **Mermaid** code blocks directly in markdown. Obsidian renders them natively — no images, no binary files, fully version-controlled.

**Use a diagram when it clarifies more than prose can.** Don't add diagrams for the sake of it. Each note gets a `## Diagram` section *only if* a diagram genuinely helps.

### Diagram type → use case mapping

| Mermaid type | Use for |
|--------------|---------|
| `flowchart` | request lifecycle, control flow, decision trees, pipelines |
| `sequenceDiagram` | async flow, request/response, handshakes, transactions |
| `classDiagram` | OOP hierarchies, interface relationships, design patterns |
| `stateDiagram-v2` | lifecycle states, GC generations, transaction states |
| `erDiagram` | EF Core entities, database schemas, star schema |
| `graph LR/TD` | architecture diagrams, topology, dependency graphs |

### Diagram authoring rules

- Always wrap in a fenced block with language `mermaid`.
- Keep diagrams under ~15 nodes — split into multiple if larger.
- Use meaningful labels, not single letters (except sequence-diagram participants where short aliases are conventional).
- Place the `## Diagram` section **after Core Concept, before Syntax & API** — students should see the picture before the code.

---

## Note Template (canonical format for every file)

Every note must follow this exact structure. Sections that don't apply can be omitted, but order must be preserved. The canonical file is `_Templates/Note Template.md`.

````markdown
---
tags: [<topic>, <level>, <category>]
aliases: []
level: Beginner | Intermediate | Advanced
---

# <Topic Title>

> **One-liner**: What this is and why it matters in one sentence.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| ... | ... |

---

## Core Concept

Plain-English explanation. No jargon without definition. Analogies encouraged. Max 3–4 short paragraphs.

---

## Diagram

*(Optional — include only when a diagram clarifies more than prose. Use Mermaid.)*

---

## Syntax & API

```<lang>
// Minimal working example
```

---

## Common Patterns

---

## Gotchas & Tips

---

## See Also

- [[Related Note 1]]
````

---

## Shared Authoring Rules

These apply across all topics. Topic-specific rules (e.g., default SQL dialect, language-specific examples) live in the per-topic `CLAUDE.md`.

- Every code block must specify a language. Common languages: `csharp`, `sql`, `bash`, `json`, `xml`, `yaml`, `mermaid`. Never plain fenced blocks.
- All internal links use Obsidian wiki-link syntax: `[[Note Title]]`. Obsidian resolves by note title across the vault, so paths are not required.
- The **Quick Reference** table must exist on every note — it is the cheatsheet anchor students scan first.
- Keep **Core Concept** prose below 300 words; put depth in sub-sections.
- Add a `## Diagram` section only when it clarifies more than prose; place it after Core Concept.
- Number prefixes on filenames (`01 -`, `02 -`) control display order in the Obsidian file explorer.
- Each topic's `00-Index/Master Index.md` must be updated whenever a new note is added in that topic.
- `.obsidian/` — do not edit manually.
- `_Templates/` is shared across topics — change with care.

---

## Shared Tag Taxonomy

Each topic adds its own tags on top of these.

| Tag | Meaning |
|-----|---------|
| `beginner` / `intermediate` / `advanced` | difficulty level |
| `index` | index/router pages |
| `home` | top-level landing pages |

See per-topic `CLAUDE.md` for topic-specific tags (`csharp`, `aspnetcore`, `postgresql`, `sql`, `react`, `hooks`, `nextjs`, etc.).
