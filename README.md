# Rightbrain Claude Code Skills

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skills-blueviolet)](https://claude.ai/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Official Claude Code skills for [Rightbrain AI](https://rightbrain.ai) - the platform for building, deploying, and managing AI tasks.

## What's Included

| Skill | Description |
|-------|-------------|
| **rightbrain-tasks** | Complete task manager for the Rightbrain API. Create, browse, run, update, export, and import AI tasks. |

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/RightbrainAI/claude-code-skills.git
```

### 2. Copy skill to Claude Code

```bash
cp -r claude-code-skills/skills/rightbrain-tasks ~/.claude/skills/
```

### 3. Set your API key

```bash
export RIGHTBRAIN_API_KEY="your-api-key"
```

That's it! Restart Claude Code and the skill will be available.

## Skill: rightbrain-tasks

A comprehensive skill for managing Rightbrain AI tasks directly from Claude Code.

### Features

- **Create tasks** - Design new AI tasks with guided prompts, output schemas, and model selection
- **Browse & run tasks** - Execute existing tasks and view results
- **Update tasks** - Modify prompts, models, temperature, and output formats
- **Export/Import** - Save task configurations to files and import them to other projects

### Trigger Phrases

Say any of these to activate the skill:

- "rightbrain tasks"
- "rightbrain task"
- "manage tasks"
- "create task"
- "run task"
- "browse tasks"

### Task Types Supported

| Type | Use Case | Example |
|------|----------|---------|
| **Classification** | Categorize inputs | Sentiment analysis, ticket routing |
| **Extraction** | Pull structured data | Invoice parsing, entity recognition |
| **Generation** | Create content | Email drafting, blog writing |
| **Analysis** | Evaluate and score | Lead scoring, risk assessment |
| **Image** | Generate visuals | Social media images, product photos |

## Getting Your API Key

1. Sign up or log in at [app.rightbrain.ai](https://app.rightbrain.ai)
2. Go to **Settings** → **API Clients**
3. Click **Create API Key**
4. Copy the key immediately (it's only shown once)
5. Store it securely

### Setting the API Key

**Option A: Environment variable (recommended)**

Add to your shell profile (`~/.bashrc`, `~/.zshrc`, etc.):

```bash
export RIGHTBRAIN_API_KEY="rb_your_key_here"
```

**Option B: Enter when prompted**

If no environment variable is set, the skill will ask for your API key when you use it.

## Documentation

- **Full Documentation:** [docs.rightbrain.ai](https://docs.rightbrain.ai/)
- **LLM-Friendly Docs:** [docs.rightbrain.ai/llms-full.txt](https://docs.rightbrain.ai/llms-full.txt)
- **API Reference:** [docs.rightbrain.ai/api-reference](https://docs.rightbrain.ai/api-reference)

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI installed
- A [Rightbrain](https://rightbrain.ai) account with API access

## Repository Structure

```
claude-code-skills/
├── README.md
├── LICENSE
└── skills/
    └── rightbrain-tasks/
        ├── SKILL.md                    # Main skill file
        ├── assets/
        │   ├── task-template.json      # Complete task template
        │   ├── export-schema-example.json
        │   └── examples/
        │       ├── classification.md
        │       ├── extraction.md
        │       └── generation.md
        └── references/
            ├── task-components.md      # Schema reference
            ├── output-formats.md       # Type system docs
            ├── prompt-patterns.md      # Prompt templates
            └── image-generation.md     # Image task guide
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Built with love by [Rightbrain AI](https://rightbrain.ai)
