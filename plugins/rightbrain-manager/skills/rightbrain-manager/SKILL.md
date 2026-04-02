---
name: rightbrain-manager
description: |
  Complete manager for the Rightbrain API. Create and manage Tasks, TaskAgents, Skills, Integrations, and MCP Servers.
  Trigger with "rightbrain tasks", "rightbrain agent", "rightbrain skill", "rightbrain integration",
  "create task", "create agent", "run agent", "browse tasks", "manage skills", "connect integration",
  "switch to staging", "use staging", "staging environment", "use production".
allowed-tools: Bash(curl *), Bash(npx rightbrain*), Bash(node *), Bash(cat *), Bash([ *), Bash(mkdir *), Bash(jq *)
---

# Rightbrain Platform Manager

Manage the four Rightbrain primitives and MCP Servers from the command line.

## The Four Primitives

| Primitive | What It Is | When to Use |
|-----------|-----------|-------------|
| **Task** | A stateless AI function: input in, structured output out | Single-step operations: classify, extract, generate, analyze |
| **TaskAgent** | An autonomous agent that orchestrates multiple tools | Multi-step workflows, tool chaining, conversations |
| **Skill** | A declarative instruction bundle for agents | Teaching agents domain knowledge, policies, procedures |
| **Integration** | An OAuth connection to external services | Connecting agents to Google Workspace (Sheets, Docs, Gmail, etc.) |
| **MCP Server** | An external tool provider via Model Context Protocol | Connecting agents to databases, internal APIs, SaaS platforms |

**Decision guide:**
- Need a single AI call with structured output? --> **Task**
- Need multi-step reasoning or tool orchestration? --> **TaskAgent**
- Need to teach an agent domain expertise? --> **Skill** (attached to an agent)
- Need to read/write Google Workspace data? --> **Integration** (attached to an agent)
- Need to call tools on an external server? --> **MCP Server** (attached to an agent)
- Not sure which primitive to use? --> See [Architecture Guide](references/architecture-guide.md)

---

## Phase 0: Authentication & Setup

### Environment Selection

The skill supports two environments. Default is **production**.

| Environment | API Base URL | When to use |
|---|---|---|
| **Production** | `https://app.rightbrain.ai/api/v1` | Default. Live data, real credits. |
| **Staging** | `https://stag.leftbrain.me/api/v1` | Testing new features, development work. |

If the user says "use staging", "staging", or "stag" → set `API_BASE` to `https://stag.leftbrain.me/api/v1` and confirm: "Switched to **staging** environment (stag.leftbrain.me)"

If the user says "use production", "production", or "prod" → set `API_BASE` to `https://app.rightbrain.ai/api/v1` and confirm: "Using **production** environment (app.rightbrain.ai)"

If unspecified, default to production: `API_BASE=https://app.rightbrain.ai/api/v1`

Store `API_BASE` in context and use it for ALL subsequent curl commands.

**Exit option:** At any point, if the user says "cancel", "exit", or "never mind", exit gracefully.

### Step 1: Check for Existing Credentials

```bash
[ -f ~/.rightbrain/credentials.json ] && echo "CREDENTIALS_FILE_EXISTS=yes" || echo "CREDENTIALS_FILE_EXISTS=no"
```

If no credentials file, run first-time login:
```bash
npx rightbrain@latest login --non-interactive
```
This opens the browser to sign in. If login fails, tell the user to try `npx rightbrain@latest login` manually.

### Step 2: Load and Validate Token

```bash
node -e "
const c = JSON.parse(require('fs').readFileSync(process.env.HOME + '/.rightbrain/credentials.json', 'utf8'));
const token = c.access_token || '';
const expiresAt = c.expires_at || 0;
const expired = expiresAt && Date.now() >= expiresAt - 300000 ? 'yes' : 'no';
console.log('TOKEN_SET=' + (token ? 'yes' : 'no'));
console.log('EXPIRED=' + expired);
console.log('ORG_ID=' + (c.org_id || ''));
console.log('PROJECT_ID=' + (c.project_id || ''));
"
```

If TOKEN_SET=no or EXPIRED=yes, re-authenticate via `npx rightbrain@latest login --non-interactive`, then re-read.

Extract the access token:
```bash
node -e "console.log(JSON.parse(require('fs').readFileSync(process.env.HOME + '/.rightbrain/credentials.json', 'utf8')).access_token)"
```

Store as `ACCESS_TOKEN` for all subsequent API calls.

### Step 3: Validate Token & Select Context

Test the token by fetching organizations:
```bash
curl -s -X GET "{API_BASE}/org" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

If 401, re-authenticate. If 200, proceed.

**If ORG_ID and PROJECT_ID already set** from credentials: use them directly, confirm to user, skip selection.

**Otherwise:** Select org (auto-select if only one, ask if multiple), then fetch and select project the same way.

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Step 4: Prefetch Models

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Store model list in context. Key fields: `id`, `name`, `supports_vision`, `supports_image_output`.

---

## Phase 1: Choose Action

Present the main action menu:

- **Create new task** -- Design a new Rightbrain task from scratch
- **Browse existing tasks** -- View, run, update, or export tasks
- **Import task from file** -- Load a task configuration from JSON
- **Create agent** -- Build a TaskAgent with tools, skills, and integrations
- **Manage agents** -- List, update, run, or delete agents
- **Manage skills** -- Browse catalog, create, or update skills
- **Connect integration** -- Set up a Google Workspace integration
- **Add MCP server** -- Connect an external tool provider

Route to the appropriate reference file based on selection.

---

## Task Quick-Start

For full task workflows, see [Tasks Reference](references/tasks.md).

### Create a Task

1. Discover what the user wants (see Phase 1-3 in tasks reference)
2. Design the output schema
3. Deploy:

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Email Classifier",
  "description": "Classifies emails by intent and urgency",
  "enabled": true,
  "output_modality": "json",
  "system_prompt": "You are an email analyst specializing in business communication.",
  "user_prompt": "## Goal\nClassify the email by intent.\n\n## Input\n{email_content}: The email to classify\n\n## Instructions\n1. Identify the primary intent\n2. Assess urgency\n3. Extract action items",
  "llm_model_id": "model-uuid",
  "llm_config": {"temperature": 0.2},
  "output_format": {
    "intent": {"type": "str", "options": ["inquiry", "complaint", "request", "feedback"]},
    "urgency": {"type": "str", "options": ["low", "medium", "high"]},
    "action_items": {"type": "list", "item_type": "str"}
  }
}
EOF
```

### Run a Task

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "task_input": {
    "email_content": "Hi, I need to cancel my subscription by Friday..."
  }
}
EOF
```

**Critical:** All inputs must be wrapped in `task_input`.

---

## Agent Quick-Start

For full agent workflows, see [Agents Reference](references/agents.md).

### Create an Agent

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Research Assistant",
  "description": "Searches and summarizes findings",
  "instruction": "You are a research assistant. Given a topic, search for relevant information and produce a structured summary.",
  "mode": "agentic",
  "llm_model_id": "model-uuid",
  "task_tools": [
    {"task_id": "search-task-uuid", "is_output_formatter": false},
    {"task_id": "summary-task-uuid", "is_output_formatter": true}
  ],
  "skills": [],
  "integrations": [],
  "mcp_servers": [],
  "memory_strategy": "sliding_window",
  "memory_config": {"max_events": 100},
  "max_turns": 15
}
EOF
```

### Run an Agent

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "message": "Research quantum computing developments in 2026",
  "session_id": null
}
EOF
```

Response streams via SSE. Pass `session_id` from a previous run to continue the conversation.

**Agent modes:**
- **Agentic**: LLM chooses tools, supports all four primitives, parallel execution
- **Sequential**: Fixed pipeline, task tools only (min 2), no skills/integrations/MCP

---

## Skills Quick-Start

For full skills workflows, see [Skills Catalog Reference](references/skills-catalog.md).

### Browse Skills

```bash
curl -s -X GET "{API_BASE}/skill/available" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Create a Skill

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "slug": "compliance-checker",
  "display_name": "Compliance Checker",
  "description": "Validates agent actions against GDPR and internal compliance policies",
  "instructions": "## When to Use\nBefore any action that involves personal data.\n\n## Steps\n1. Identify personal data fields\n2. Check consent status\n3. Apply data minimization\n\n## Constraints\n- Never process data without verified consent\n- Log all compliance decisions",
  "reference_docs": {},
  "skill_metadata": {"author": "security-team"}
}
EOF
```

Skills are declarative instructions, not executable tools. They teach agents domain knowledge. Agentic mode only.

---

## Integrations Quick-Start

For full integration workflows, see [Integrations Reference](references/integrations.md).

### Connect Google Sheets

```bash
# 1. Create the integration
curl -s -X POST "{API_BASE}/integration" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{"type": "google_sheets"}
EOF

# 2. Authorize (returns URL for browser OAuth)
curl -s -X POST "{API_BASE}/integration/{INTEGRATION_ID}/authorize" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

**CRITICAL:** Google Sheets has 17 tools (~936k tokens). Always use `allowed_tool_ids` when attaching to agents:
```json
{"project_integration_id": "uuid", "allowed_tool_ids": ["values_get", "values_update", "values_append"]}
```

Available types: `google_sheets`, `google_docs`, `google_slides`, `google_calendar`, `google_gmail`

---

## MCP Servers Quick-Start

For full MCP workflows, see [MCP Servers Reference](references/mcp-servers.md).

### Add an MCP Server

```bash
# 1. Discover tools
curl -s -X POST "{API_BASE}/task_mcp_server/discover" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{"url": "https://mcp.example.com/sse", "transport": "sse"}
EOF

# 2. Create server
curl -s -X POST "{API_BASE}/task_mcp_server" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{"name": "My MCP Server", "url": "https://mcp.example.com/sse", "transport": "sse"}
EOF
```

Attach to agent: `{"task_mcp_server_id": "uuid", "allowed_tool_ids": null}`

If the server requires OAuth, use `GET /task_mcp_server/authorize?url=...` first.

---

## Common API Patterns

### Base URL
`{API_BASE}`

### Authentication
All requests require:
```
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
```

### JSON Payloads
Always use heredoc syntax for reliability:
```bash
curl -s -X POST "{API_BASE}/..." \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{"key": "value"}
EOF
```

### Array Update Semantics (Agents)
For `task_tools`, `skills`, `integrations`, `mcp_servers`:
- **Omitted** = no change
- **Empty list `[]`** = clear all
- **Non-empty list** = full replacement (not merge)

---

## Error Handling

| Status | Cause | Solution |
|--------|-------|----------|
| 400 | Validation error | Check error message, fix input |
| 400 | `duplicate_task_name` | Use a unique name |
| 401 | Invalid/expired session | Re-run `npx rightbrain@latest login` |
| 403 | No permission | Verify access in dashboard |
| 404 | Resource not found | Check org/project/resource IDs |
| 429 | Rate limit | Wait 30 seconds, retry |
| 500 | Server error | Retry after a few seconds |

Add timeouts for long requests: `--connect-timeout 10 --max-time 30`

---

## Validation Rules

- **Task name:** Alphanumeric, hyphens, spaces. Max 63 chars. Unique per project.
- **Output format keys:** Alphanumeric, underscore, dot, hyphen. 1-64 chars.
- **Modality rules:** `json` requires output_format; all others require `null`.
- **Sequential agents:** Minimum 2 task tools, no skills/integrations/MCP.
- **Agent instructions:** Avoid curly braces `{like_this}` -- ADK treats them as session state variables.

See [Task Components Reference](references/task-components.md) for complete validation documentation.

---

## References

**Workflow guides:**
- [Tasks Reference](references/tasks.md) - Create, browse, run, update, export, import tasks
- [Agents Reference](references/agents.md) - Create, run, manage TaskAgents
- [Skills Catalog Reference](references/skills-catalog.md) - Browse, create, manage skills
- [Integrations Reference](references/integrations.md) - Google Workspace connections
- [MCP Servers Reference](references/mcp-servers.md) - External tool providers
- [Architecture Guide](references/architecture-guide.md) - Choosing the right primitive, cost model, composition patterns

**Task-specific references:**
- [Task Components](references/task-components.md) - Complete schema documentation
- [Prompt Patterns](references/prompt-patterns.md) - Prompt templates and examples
- [Output Formats](references/output-formats.md) - Type system and validation
- [Image Generation](references/image-generation.md) - Image tasks and text-prevention

**Templates:**
- [Task Template](assets/task-template.json) - Complete task configuration
- [Agent Template](assets/agent-template.json) - Complete agent configuration with all primitive types
- [Export Schema Example](assets/export-schema-example.json) - Example export file
- [Classification Examples](assets/examples/classification.md)
- [Extraction Examples](assets/examples/extraction.md)
- [Generation Examples](assets/examples/generation.md)
