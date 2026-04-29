---
tags: [claude-code, index]
aliases: [Claude Code Index, CC Home]
---

# Claude Code Master Index — AI Agent Cheatsheet

> **Single entry point** for the Claude Code topic. Browse by level, or jump straight to a feature. Audience: developers using Claude Code on real projects.

---

## Start Here

- [[Claude Code Learning Path]] — recommended reading order
- New to Claude Code? Begin with [[01 - Claude Code Overview]]
- Already installed? Skip to [[03 - Slash Commands Basics]] and [[06 - CLAUDE.md Files]]

---

## 01 — Beginner (get productive fast)

The minimum you need to do real work. Ten short notes — read in order on day one.

| # | Note | What you'll learn |
|---|------|-------------------|
| 01 | [[01 - Claude Code Overview]] | What Claude Code is, surfaces, model lineup |
| 02 | [[02 - Installation and Setup]] | Install, login, first session, where settings live |
| 03 | [[03 - Slash Commands Basics]] | Built-in `/help`, `/clear`, `/init`, `/memory`, `/agents`, `/review` |
| 04 | [[04 - Tools and Capabilities]] | Read, Edit, Bash, Grep, Glob, Task — what each does |
| 05 | [[05 - Permissions and Safety]] | Permission prompts, allow/deny, modes |
| 06 | [[06 - CLAUDE.md Files]] | Project memory, scope, what to include |
| 07 | [[07 - Effective Prompting]] | Prompt patterns, scope, brief vs detailed |
| 08 | [[08 - Plan Mode]] | When to plan first vs jump in |
| 09 | [[09 - Common Workflows]] | Bug fix / feature / refactor / review recipes |
| 10 | [[10 - Tips and Pitfalls]] | Antipatterns, context budget, when to stop |

---

## 02 — Intermediate (extend and customize)

Make Claude fit *your* workflow. Customize commands, add hooks, plug in MCP servers.

### Customization
| # | Note |
|---|------|
| 01 | [[01 - Subagents]] |
| 02 | [[02 - Custom Slash Commands]] |
| 03 | [[03 - settings.json]] |
| 04 | [[04 - Hooks]] |
| 05 | [[05 - MCP Servers Using]] |
| 06 | [[06 - Memory System]] |
| 07 | [[07 - Output Styles]] |

### Workflows
| # | Note |
|---|------|
| 08 | [[08 - Git and PR Workflow]] |
| 09 | [[09 - Code Review with Claude]] |
| 10 | [[10 - TDD Workflow]] |
| 11 | [[11 - Background Tasks]] |
| 12 | [[12 - Worktrees and Parallel Work]] |
| 13 | [[13 - IDE Integration]] |
| 14 | [[14 - Multi-file Refactoring]] |
| 15 | [[15 - Model and Cost Optimization]] |

---

## 03 — Advanced (orchestrate and build)

Build your own agents, hooks, MCP servers — and orchestrate multi-agent workflows.

### Building
| # | Note |
|---|------|
| 01 | [[01 - Building Custom Agents]] |
| 04 | [[04 - Building Custom Hooks]] |
| 05 | [[05 - Building MCP Servers]] |
| 06 | [[06 - Claude Agent SDK]] |

### Orchestration and operations
| # | Note |
|---|------|
| 02 | [[02 - Agent Orchestration]] |
| 03 | [[03 - Multi-Agent Reviews]] |
| 07 | [[07 - CI-CD Integration]] |
| 08 | [[08 - Debugging and Observability]] |
| 09 | [[09 - Security and Sandboxing]] |
| 10 | [[10 - Prompt Engineering for Code]] |

---

## Browse by tag

Use Obsidian's tag pane to filter:

- `#slash-commands` — built-in and custom commands
- `#subagents` `#orchestration` — agent design and patterns
- `#hooks` — automation around tool calls
- `#mcp` — Model Context Protocol
- `#memory` — CLAUDE.md, persistent memory
- `#settings` — `settings.json`, allowedTools, env
- `#workflow` — TDD, code review, git, PRs
- `#sdk` `#api` — Claude Agent SDK, Anthropic API
- `#ide` — VS Code, JetBrains
- `#cost` — model selection, caching
- `#security` — sandboxing, permissions, safety
