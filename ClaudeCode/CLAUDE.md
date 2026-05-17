# CLAUDE.md — Claude Code Topic

Topic-specific guidance for the `ClaudeCode/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured cheatsheet for **using Claude Code (and AI coding agents in general) effectively** — installation, prompting, slash commands, subagents, hooks, MCP servers, settings, and orchestration. Organized beginner → intermediate → advanced, intended for developers who want to get real productivity from an AI coding agent.

This is a **how-to-use** vault, not an internal reference for how Claude is built. The audience is engineers using Claude Code on real projects.

---

## Folder Structure

```
ClaudeCode/
├── CLAUDE.md                    ← this file
├── 00-Index/
│   ├── Claude Code Master Index.md   ← single entry point
│   └── Claude Code Learning Path.md  ← recommended reading order
├── 01-Beginner/                 (10 notes — get productive fast)
├── 02-Intermediate/             (15 notes — extend and customize)
└── 03-Advanced/                 (10 notes — orchestrate and build)
```

---

## Tag Taxonomy (Claude Code-specific)

Always include `claude-code` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `claude-code` | applied to every note in this topic |
| `cli` | terminal usage, command-line patterns |
| `prompting` | prompt engineering, effective questions |
| `slash-commands` | built-in or custom `/commands` |
| `subagents` | sub-agent design and usage |
| `hooks` | event hooks, automation |
| `mcp` | Model Context Protocol servers |
| `memory` | CLAUDE.md, persistent memory, context |
| `settings` | settings.json, allowedTools, env |
| `workflow` | TDD, code review, git, PRs |
| `orchestration` | multi-agent, parallel agents |
| `sdk` | Claude Agent SDK / Anthropic SDK |
| `api` | Claude API (Anthropic API) |
| `ide` | VS Code, JetBrains, IDE extensions |
| `cost` | model selection, caching, budget |
| `security` | sandboxing, permissions, safety |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `bash` | shell commands, `claude` CLI invocations, install steps |
| `json` | `settings.json`, `mcp.json`, hook config |
| `markdown` | `CLAUDE.md` snippets, slash command files, agent files |
| `yaml` | GitHub Actions integrations, MCP server descriptors |
| `python` / `typescript` | Agent SDK / Anthropic SDK examples |
| `mermaid` | diagrams |

Examples must be **runnable as written** — student should be able to paste into a terminal or settings file and have it work.

When showing CLI invocations, use the `claude` binary name (or `cc` if the user has aliased it). Do NOT invent URLs or version numbers — refer to "the docs" or "your settings file" in prose.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Claude Code Overview | What Claude Code is, surfaces (CLI / desktop / IDE / web), models (Opus / Sonnet / Haiku) |
| 02 | Installation and Setup | Install, login, first session, where settings live |
| 03 | Slash Commands Basics | `/help`, `/clear`, `/init`, `/memory`, `/agents`, `/review`, `/config`, `/cost` |
| 04 | Tools and Capabilities | Read, Edit, Write, Glob, Grep, Bash, Task, WebFetch, WebSearch — what each does |
| 05 | Permissions and Safety | Permission prompts, allow/deny, modes, dangerous-action prompts |
| 06 | CLAUDE.md Files | Project memory, scope, what to put in vs out, hierarchy |
| 07 | Effective Prompting | Specificity, scope, context, "don't peek" patterns, brief vs detailed |
| 08 | Plan Mode | When to use, how to enter/exit, ExitPlanMode, plan-then-execute pattern |
| 09 | Common Workflows | Bug fix, feature add, refactor, code review, debugging session |
| 10 | Tips and Pitfalls | Prompt antipatterns, context window, sharing tool output, when to stop |

### Intermediate (02-Intermediate) — 15 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Subagents | What they are, when to use, fork vs subagent_type, parallel agents |
| 02 | Custom Slash Commands | `.claude/commands/*.md`, args, frontmatter, scope |
| 03 | settings.json | Schema, allowedTools, env, model, scope (user/project/local) |
| 04 | Hooks | PreToolUse / PostToolUse / Stop, blocking vs non-blocking, examples |
| 05 | MCP Servers Using | Adding servers, configuring, popular servers (filesystem, github, browser) |
| 06 | Memory System | Auto memory, types (user/feedback/project/reference), MEMORY.md |
| 07 | Output Styles | Customizing default behavior, terse vs verbose, persona |
| 08 | Git and PR Workflow | Commits, branches, conventional commits, gh CLI, PR creation |
| 09 | Code Review with Claude | `/review`, agent-driven review, severity levels, security review |
| 10 | TDD Workflow | RED-GREEN-REFACTOR with Claude, test-first prompting |
| 11 | Background Tasks | `/loop`, `/schedule`, when to use each, dynamic vs cron |
| 12 | Worktrees and Parallel Work | Worktree isolation, parallel forks, when to isolate |
| 13 | IDE Integration | VS Code extension, JetBrains, terminal-in-IDE patterns |
| 14 | Multi-file Refactoring | Large-scale changes, plan-mode-first, incremental commits |
| 15 | Model and Cost Optimization | Picking Opus / Sonnet / Haiku, caching, context budget |
| 16 | Popular Claude GitHub Repos | Ecosystem cheat sheet: official SDKs, MCP, Cline/aider/goose, awesome-lists |

### Advanced (03-Advanced) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Building Custom Agents | Agent file format, tools list, model, system prompt, when to invoke |
| 02 | Agent Orchestration | Parallel, sequential, fork, multi-perspective review, split-role agents |
| 03 | Multi-Agent Reviews | Factual / senior / security / consistency reviewers, aggregation |
| 04 | Building Custom Hooks | Shell scripts, JSON I/O contract, PreToolUse pattern matching |
| 05 | Building MCP Servers | Server SDK, tools/resources/prompts, transport, testing |
| 06 | Claude Agent SDK | Programmatic agent creation, tool definitions, message loop |
| 07 | CI-CD Integration | GitHub Actions, automated reviews, PR comment bots |
| 08 | Debugging and Observability | Verbose mode, transcripts, telemetry, troubleshooting |
| 09 | Security and Sandboxing | Tool allowlists, network policies, secret leakage, prompt injection |
| 10 | Prompt Engineering for Code | Patterns: scoping, constraints, examples, anti-leakage |

**Total: 36 notes** (10 beginner + 16 intermediate + 10 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **should** include one:

| Note | Diagram type |
|------|--------------|
| Claude Code Overview | graph (surfaces + model lineup) |
| Tools and Capabilities | graph (tool taxonomy by category) |
| Permissions and Safety | flowchart (permission decision flow) |
| CLAUDE.md Files | graph (memory hierarchy: user / project / local) |
| Plan Mode | stateDiagram-v2 (Plan → Execute → Done) |
| Subagents | flowchart (fork vs subagent_type vs main thread) |
| Hooks | sequenceDiagram (PreToolUse → tool → PostToolUse) |
| MCP Servers | graph (Claude ↔ MCP server ↔ external tool) |
| Memory System | graph (memory types + MEMORY.md index) |
| Background Tasks | flowchart (loop vs schedule decision tree) |
| Building Custom Agents | classDiagram or graph (agent file structure) |
| Agent Orchestration | sequenceDiagram (parallel agent fan-out) |
| Building MCP Servers | graph (server lifecycle + transport) |
| CI-CD Integration | sequenceDiagram (PR → action → review comment) |

---

## Authoring Notes Specific to This Topic

- **Avoid hallucinating commands or flags.** When unsure, write "see `claude --help`" or "see your `settings.json` schema" rather than invent.
- **Do not invent URLs.** External documentation is referred to in prose, not as a clickable link, unless the URL is well-known and stable.
- **Examples should be platform-agnostic** where possible. When platform-specific, label clearly: "On Windows...", "On macOS/Linux...".
- **Keep cheatsheet style** — most notes will lean heavier on tables, code blocks, and bulleted lists than long prose. The Quick Reference table is the most-used surface.
- **Cross-link aggressively** — Claude Code features compose (memory + hooks + agents). Use `[[Note Title]]` to chain related notes.
