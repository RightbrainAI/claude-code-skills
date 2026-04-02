# TaskAgents Reference

Detailed workflows for creating, running, and managing Rightbrain TaskAgents.

> **Prerequisites:** Authentication and context (ACCESS_TOKEN, ORG_ID, PROJECT_ID) must be established before using these workflows. See SKILL.md Phase 0.

---

## Overview

A TaskAgent is an autonomous AI agent that orchestrates multiple tools (Tasks, Skills, Integrations, MCP Servers) to accomplish complex goals. Agents support multi-turn conversations, memory strategies, and streaming execution.

**Two modes:**
- **Agentic** (default): LLM decides which tools to call, supports all tool types, parallel execution
- **Sequential**: Fixed pipeline, task tools only, no skills/integrations/MCP, minimum 2 task tools

---

## API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Create agent | POST | `/org/{org_id}/project/{project_id}/task-agent` |
| List agents | GET | `/org/{org_id}/project/{project_id}/task-agent` |
| Get agent | GET | `/org/{org_id}/project/{project_id}/task-agent/{id}` |
| Update agent | POST | `/org/{org_id}/project/{project_id}/task-agent/{id}` |
| Delete agent | DELETE | `/org/{org_id}/project/{project_id}/task-agent/{id}` |
| Run agent | POST | `/org/{org_id}/project/{project_id}/task-agent/{id}/run` |
| List runs | GET | `/org/{org_id}/project/{project_id}/task-agent/{id}/run` |
| Get run events | GET | `/org/{org_id}/project/{project_id}/task-agent/{id}/run/{run_id}/events` |

---

## Create Agent

### Step 1: Choose Mode

Ask the user what they need:
- **Agentic mode** -- The agent decides which tools to call based on the user's message. Supports all tool types. Best for open-ended tasks, research, or multi-step workflows where the path is not predetermined.
- **Sequential mode** -- A fixed pipeline of task tools executed in order. Best for deterministic, repeatable workflows. Requires minimum 2 task tools. Does not support skills, integrations, or MCP servers.

### Step 2: Build Configuration

Gather the required information:
- **name**: Human-readable agent name
- **description**: What the agent does (shown in listings)
- **instruction**: System prompt for the agent -- keep it concise and goal-oriented
- **llm_model_id**: Model UUID (fetch from `/model` endpoint)
- **mode**: `"agentic"` or `"sequential"`

### Step 3: Attach Tools

Depending on mode, attach the appropriate tool types:

**Task tools** (both modes):
```json
"task_tools": [
  {"task_id": "uuid-of-task", "is_output_formatter": false},
  {"task_id": "uuid-of-formatter-task", "is_output_formatter": true}
]
```
- `is_output_formatter: true` -- This task formats the agent's final output. Only one formatter allowed.

**Skills** (agentic only):
```json
"skills": [
  {"skill_id": "uuid-of-skill", "skill_revision_id": null}
]
```
- `skill_revision_id: null` = follow latest active revision
- `skill_revision_id: "uuid"` = pin to specific version

**Integrations** (agentic only):
```json
"integrations": [
  {"project_integration_id": "uuid", "allowed_tool_ids": ["tool-1", "tool-2"]}
]
```
- `allowed_tool_ids: null` = all tools exposed (use with caution for Google Sheets)
- `allowed_tool_ids: ["..."]` = only expose listed tools

**MCP Servers** (agentic only):
```json
"mcp_servers": [
  {"task_mcp_server_id": "uuid", "allowed_tool_ids": null}
]
```

### Step 4: Configure Memory

```json
"memory_strategy": "sliding_window",
"memory_config": {"max_events": 100}
```

Available strategies:
- **sliding_window** -- Keeps the last N events. Simple, predictable token usage.
- **compaction** -- Summarizes older events to save tokens while preserving context. Better for long sessions.

### Step 5: Set Limits

```json
"max_turns": 15
```

`max_turns` limits how many tool-call rounds the agent can make per run. Prevents runaway execution. Default is reasonable for most use cases; increase for complex multi-step workflows.

### Step 6: Deploy

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Research Assistant",
  "description": "Searches the web and summarizes findings",
  "instruction": "You are a research assistant. Given a topic, search for relevant information, analyze findings, and produce a structured summary.",
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

On success, note the returned agent `id` for running.

---

## Run Agent

### New Session

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "message": "Research the latest developments in quantum computing"
}
EOF
```

The response streams via **Server-Sent Events (SSE)**. Each event contains:
- `event_type`: The kind of event (e.g., `tool_call`, `tool_result`, `text_chunk`, `done`)
- `data`: Event payload

The final response includes a `session_id` and `run_id`.

### Continue Existing Session (Multi-turn)

Pass the `session_id` from a previous run to continue the conversation:

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "message": "Now compare that with classical computing approaches",
  "session_id": "session-uuid-from-previous-run"
}
EOF
```

The agent retains context from previous turns based on the configured memory strategy.

---

## Update Agent

Updates use POST (not PATCH). Only include fields you want to change.

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "instruction": "Updated system prompt here",
  "max_turns": 20
}
EOF
```

**Array field semantics:**
- **Omitted** = unchanged (field not in payload)
- **Empty list `[]`** = clear all items
- **Non-empty list** = full replacement (not merge)

This applies to `task_tools`, `skills`, `integrations`, and `mcp_servers`.

Example -- add a new skill while keeping existing task tools:
1. GET the current agent to see existing `task_tools`
2. POST with `skills` array only (omit `task_tools` to leave them unchanged)

---

## Run Observability

### List Runs

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Each run includes:
- `id`: Run UUID
- `session_id`: Session this run belongs to
- `status`: `completed`, `failed`, `running`
- `created`: Timestamp
- `phase_timings`: Breakdown of time spent in each phase

### Get Run Events

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}/run/{RUN_ID}/events" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Events show the full trace of agent execution:
- Which tools were called and in what order
- Tool inputs and outputs
- `skills_activated`: Which skills the agent loaded
- `integration_tools_invoked`: Which integration tools were called
- Phase timings for performance analysis

---

## Delete Agent

```bash
curl -s -X DELETE "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task-agent/{AGENT_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

This is permanent. All run history for the agent is also deleted.

---

## Agent Instruction Best Practices

1. **Be concise and goal-oriented** -- The LLM figures out tool mechanics from tool descriptions. Do not repeat tool documentation in the instruction.
2. **Avoid curly braces `{like_this}` in instructions** -- ADK treats them as session state variables and will attempt to interpolate them.
3. **Specify output expectations** -- Tell the agent what format the final answer should be in.
4. **Set boundaries** -- Clarify what the agent should NOT do (e.g., "Do not modify any data, only read").

**Good instruction example:**
```
You are a competitive analysis assistant. Given a company name, research their
products, pricing, and recent news. Produce a structured comparison against our
offerings. Focus on factual, verifiable information only.
```

**Bad instruction example:**
```
You are an assistant. You have access to {tool_1} and {tool_2}. When the user
asks something, first call {tool_1} to search, then call {tool_2} to format.
Always call tools in this order. The output of {tool_1} should be passed to
{tool_2} as input.
```

---

## Output Formatters

An output formatter is a task tool with `is_output_formatter: true`. The agent calls this as its final step to produce structured output.

**How it works:**
1. Agent does its reasoning and tool calls
2. Agent calls the formatter task with its collected results
3. The formatter task's output format defines the agent's final output schema

This is useful when you need the agent to return structured JSON matching a specific schema, regardless of which tools it used internally.

**Constraints:**
- Only one output formatter per agent
- The formatter task must have `output_modality: "json"` with a defined `output_format`

---

## Common Patterns

### Research Agent
- 1 search task (web/perplexity) + 1 summary formatter task
- Agentic mode, sliding_window memory
- max_turns: 10

### Data Processing Pipeline
- Sequential mode with 3+ task tools
- Input -> Clean -> Transform -> Format
- Each task's output feeds the next

### Customer Service Agent
- Agentic mode with skills for tone/policy guidelines
- Integration with Google Gmail for email access
- Multi-turn sessions for ongoing conversations

### Document Analysis Agent
- Agentic mode with Google Docs integration
- Task tools for extraction and classification
- Output formatter for structured report
