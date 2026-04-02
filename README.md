# Rightbrain Claude Code Skills

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skills-blueviolet)](https://claude.ai/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Official Claude Code plugin for [Rightbrain AI](https://rightbrain.ai) — the platform for embedding AI into existing tech stacks via a single API call.

## What is Rightbrain?

Rightbrain lets you build production AI features without infrastructure. The platform provides four primitives that work together:

| Primitive | What it is | Example |
|-----------|-----------|---------|
| **Task** | A stateless AI function with typed input/output | Classify sentiment, extract invoices, generate images |
| **TaskAgent** | An orchestrator that reasons about which tasks to call | Support triage, content pipeline, deal intelligence |
| **Skill** | A declarative instruction bundle for agents | Brand voice guide, escalation procedures, design guidelines |
| **Integration** | A platform-managed OAuth connection | Google Sheets, Docs, Slides, Calendar, Gmail |

Tasks are composed into **TaskAgents**, which combine them with **Skills** (domain expertise), **Integrations** (Google Workspace), and **MCP Servers** (external APIs) to build complex AI workflows.

[Learn more](https://docs.rightbrain.ai/docs/getting-started/introduction) | [Developer Blog](https://blog.leftbrain.me)

## What's Included

| Plugin | Description |
|--------|-------------|
| **rightbrain-manager** | Complete manager for the Rightbrain API. Create and manage Tasks, TaskAgents, Skills, Integrations, and MCP Servers. |

## Quick Start

### Option A: Install via Claude Code (recommended)

Run these commands inside Claude Code:

```
/plugin marketplace add https://github.com/RightbrainAI/claude-code-skills
/plugin install rightbrain-manager@rightbrain-skills
```

That's it! The skill will handle authentication automatically when you first use it.

### Option B: Manual install

```bash
git clone https://github.com/RightbrainAI/claude-code-skills.git
cp -r claude-code-skills/plugins/rightbrain-manager/skills/rightbrain-manager ~/.claude/skills/
npx rightbrain@latest login
```

Restart Claude Code and the skill will be available.

## Skill: rightbrain-manager

A comprehensive skill for managing the full Rightbrain platform from Claude Code.

### Features

- **Tasks** — Create, browse, run, update, export, and import AI tasks with guided workflows
- **TaskAgents** — Build agents that orchestrate multiple tasks, skills, integrations, and MCP servers
- **Skills** — Browse the catalog, create custom project skills, manage revisions
- **Integrations** — Connect Google Workspace (Sheets, Docs, Slides, Calendar, Gmail) via OAuth
- **MCP Servers** — Connect external tool providers (Notion, GitHub, custom APIs)
- **Environment switching** — Work against production or staging with a single command

### Trigger Phrases

Say any of these to activate the skill:

| Category | Triggers |
|----------|----------|
| **Tasks** | "rightbrain tasks", "create task", "run task", "browse tasks" |
| **Agents** | "rightbrain agent", "create agent", "run agent" |
| **Skills** | "rightbrain skill", "manage skills" |
| **Integrations** | "rightbrain integration", "connect integration" |
| **Environment** | "use staging", "switch to staging", "use production" |

### Environment Support

The skill supports two environments:

| Environment | API Base | When to use |
|-------------|----------|-------------|
| **Production** | `app.rightbrain.ai` | Default. Live data, real credits. |
| **Staging** | `stag.leftbrain.me` | Testing new features, development work. |

Switch with: "use staging" or "use production"

### Architecture Guide

Not sure which primitive to use? The skill includes a built-in [architecture guide](plugins/rightbrain-manager/skills/rightbrain-manager/references/architecture-guide.md) with:

- **Selection heuristic** — Tasks for single-step, Agents for multi-step reasoning
- **Cost model comparison** — predictable (Tasks) vs variable (Agents)
- **Composition patterns** — how to combine primitives for complex workflows
- **Key insight** — concise agent instructions work as well as verbose ones; the LLM figures out tool mechanics from descriptions

### Reference Files

| Reference | What it covers |
|-----------|----------------|
| [Tasks](plugins/rightbrain-manager/skills/rightbrain-manager/references/tasks.md) | Full task lifecycle: create, run, update, export, import |
| [Agents](plugins/rightbrain-manager/skills/rightbrain-manager/references/agents.md) | Agent creation, modes, tool attachment, sessions, observability |
| [Skills](plugins/rightbrain-manager/skills/rightbrain-manager/references/skills-catalog.md) | Catalog, custom skills, revision workflow, agent attachment |
| [Integrations](plugins/rightbrain-manager/skills/rightbrain-manager/references/integrations.md) | Google Workspace OAuth, tool filtering, agent attachment |
| [MCP Servers](plugins/rightbrain-manager/skills/rightbrain-manager/references/mcp-servers.md) | Server connection, auth, tool discovery |
| [Architecture](plugins/rightbrain-manager/skills/rightbrain-manager/references/architecture-guide.md) | Choosing primitives, cost model, patterns |
| [Prompt Patterns](plugins/rightbrain-manager/skills/rightbrain-manager/references/prompt-patterns.md) | Prompt templates and examples |
| [Output Formats](plugins/rightbrain-manager/skills/rightbrain-manager/references/output-formats.md) | Type system and validation rules |
| [Task Components](plugins/rightbrain-manager/skills/rightbrain-manager/references/task-components.md) | Complete schema documentation |
| [Image Generation](plugins/rightbrain-manager/skills/rightbrain-manager/references/image-generation.md) | Image tasks and text-prevention |

## Authentication

1. Run `npx rightbrain@latest login` (or the skill prompts you automatically)
2. Sign in via your browser — don't have an account? Sign up for free during this step
3. Select your organization and project
4. Credentials stored securely in `~/.rightbrain/credentials.json`

## Documentation

- **Full Documentation:** [docs.rightbrain.ai](https://docs.rightbrain.ai/)
- **LLM-Friendly Docs:** [docs.rightbrain.ai/llms-full.txt](https://docs.rightbrain.ai/llms-full.txt)
- **API Reference:** [docs.rightbrain.ai/api-reference](https://docs.rightbrain.ai/api-reference)
- **Developer Blog:** [blog.leftbrain.me](https://blog.leftbrain.me)

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI installed
- [Node.js](https://nodejs.org/) (for `npx rightbrain@latest login`)
- A [Rightbrain](https://rightbrain.ai) account

## Repository Structure

```
claude-code-skills/
├── .claude-plugin/
│   └── marketplace.json                # Plugin marketplace catalog
├── README.md
├── LICENSE
└── plugins/
    └── rightbrain-manager/             # Plugin root
        ├── .claude-plugin/
        │   └── plugin.json             # Plugin manifest
        └── skills/
            └── rightbrain-manager/     # Skill directory
                ├── SKILL.md            # Main skill hub (< 500 lines)
                ├── assets/
                │   ├── agent-template.json
                │   ├── task-template.json
                │   ├── export-schema-example.json
                │   └── examples/
                │       ├── classification.md
                │       ├── extraction.md
                │       └── generation.md
                └── references/
                    ├── architecture-guide.md
                    ├── agents.md
                    ├── skills-catalog.md
                    ├── integrations.md
                    ├── mcp-servers.md
                    ├── tasks.md
                    ├── task-components.md
                    ├── output-formats.md
                    ├── prompt-patterns.md
                    └── image-generation.md
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Built with love by [Rightbrain AI](https://rightbrain.ai)
