---
tags: [claude-code, index]
aliases: [Claude Code Path, CC Roadmap]
---

# Claude Code Learning Path

> Recommended reading order. Skip ahead if you already know an area, but the order is designed so each note builds on the previous.

---

## Day 1 — Get productive (≈ 1–2 hours)

**Goal**: install, run a first session, understand permission prompts, write a project `CLAUDE.md`.

1. [[01 - Claude Code Overview]] — what it is, what surfaces it runs on
2. [[02 - Installation and Setup]] — install + first login
3. [[05 - Permissions and Safety]] — read this BEFORE running real tasks
4. [[03 - Slash Commands Basics]] — `/help`, `/clear`, `/init`, `/memory`
5. [[04 - Tools and Capabilities]] — what Claude can actually do
6. [[06 - CLAUDE.md Files]] — pin project context that survives across sessions
7. [[07 - Effective Prompting]] — how to ask without back-and-forth
8. [[08 - Plan Mode]] — when to plan first
9. [[09 - Common Workflows]] — recipes for bug fix, feature, refactor
10. [[10 - Tips and Pitfalls]] — what *not* to do

---

## Week 1 — Customize for your project (≈ 4–6 hours)

**Goal**: tailor Claude to your codebase with custom commands, hooks, and memory.

1. [[03 - settings.json]] — pick a permission mode, allowed tools
2. [[06 - Memory System]] — auto-memory + manual memory
3. [[02 - Custom Slash Commands]] — your own `/foo` for repeated tasks
4. [[01 - Subagents]] — specialize work to a sub-agent
5. [[04 - Hooks]] — auto-format on edit, block dangerous commands
6. [[07 - Output Styles]] — terse / verbose / persona
7. [[15 - Model and Cost Optimization]] — pick the right model for the task

---

## Week 2 — Integrate with your workflow (≈ 4–6 hours)

**Goal**: use Claude inside your daily git / PR / TDD flow.

1. [[08 - Git and PR Workflow]] — commits, branches, PRs
2. [[10 - TDD Workflow]] — write tests first, let Claude implement
3. [[09 - Code Review with Claude]] — `/review` and dedicated review agents
4. [[14 - Multi-file Refactoring]] — large changes safely
5. [[12 - Worktrees and Parallel Work]] — isolate experimental work
6. [[13 - IDE Integration]] — VS Code / JetBrains
7. [[11 - Background Tasks]] — `/loop` and `/schedule`

---

## Month 1+ — Extend and orchestrate (deep dives)

**Goal**: build your own agents, hooks, MCP servers; orchestrate multi-agent workflows.

1. [[01 - Building Custom Agents]] — write your first specialized agent
2. [[02 - Agent Orchestration]] — fork, parallel, sequential
3. [[03 - Multi-Agent Reviews]] — split-role critique panels
4. [[04 - Building Custom Hooks]] — shell scripts triggered by tool calls
5. [[05 - Building MCP Servers]] — expose tools to Claude
6. [[06 - Claude Agent SDK]] — programmatic agents
7. [[10 - Prompt Engineering for Code]] — patterns that move the needle
8. [[09 - Security and Sandboxing]] — running agents safely
9. [[07 - CI-CD Integration]] — agents in GitHub Actions
10. [[08 - Debugging and Observability]] — when things go sideways

---

## Quick paths by goal

| If you want to… | Read |
|------------------|------|
| Just start using it today | Day 1 list above |
| Stop being asked for permission constantly | [[05 - Permissions and Safety]] → [[03 - settings.json]] |
| Reuse a complex multi-step task | [[02 - Custom Slash Commands]] |
| Specialize a task to a smarter prompt | [[01 - Subagents]] → [[01 - Building Custom Agents]] |
| Auto-format on every edit | [[04 - Hooks]] |
| Save context across sessions | [[06 - CLAUDE.md Files]] → [[06 - Memory System]] |
| Run a recurring task | [[11 - Background Tasks]] |
| Add new capabilities (DBs, APIs) | [[05 - MCP Servers Using]] → [[05 - Building MCP Servers]] |
| Cut your bill | [[15 - Model and Cost Optimization]] |
| Run Claude in CI | [[07 - CI-CD Integration]] |
