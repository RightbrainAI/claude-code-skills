# MCP Servers Reference

Detailed workflows for discovering, creating, and managing MCP (Model Context Protocol) servers in Rightbrain.

> **Prerequisites:** Authentication and context (ACCESS_TOKEN, ORG_ID, PROJECT_ID) must be established before using these workflows. See SKILL.md Phase 0.

---

## Overview

MCP Servers let you connect external tool providers to Rightbrain agents using the Model Context Protocol standard. This enables agents to call tools hosted outside of Rightbrain -- databases, internal APIs, SaaS platforms, or any service exposing an MCP endpoint.

**Key concepts:**
- **Discovery**: Probe an MCP server URL to see what tools it offers before creating it
- **Authorization**: Some MCP servers require OAuth -- Rightbrain handles the flow
- **Tool filtering**: Like integrations, you can restrict which tools an agent can access
- **Agentic mode only**: MCP servers are not available in sequential mode agents

---

## API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Discover tools | POST | `/task_mcp_server/discover` |
| Create server | POST | `/task_mcp_server` |
| List servers | GET | `/task_mcp_server` |
| Get server | GET | `/task_mcp_server/{id}` |
| Update server | POST | `/task_mcp_server/{id}` |
| Delete server | DELETE | `/task_mcp_server/{id}` |
| Authorize (OAuth) | GET | `/task_mcp_server/authorize` |

---

## Setup Workflow

### Step 1: Discover Available Tools

Before creating a server, probe the endpoint to see what tools it offers:

```bash
curl -s -X POST "{API_BASE}/task_mcp_server/discover" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "url": "https://mcp.example.com/sse",
  "transport": "sse"
}
EOF
```

The response lists all tools the server exposes, including:
- `name`: Tool identifier
- `description`: What the tool does
- `input_schema`: Expected parameters

Review the tools to decide which ones are relevant for your agent.

**Transport types:**
- `sse` -- Server-Sent Events (most common for remote servers)
- `streamable_http` -- HTTP-based streaming

### Step 2: Authorize (If Required)

Some MCP servers require OAuth authorization. If the discover response indicates auth is needed:

```bash
curl -s -X GET "{API_BASE}/task_mcp_server/authorize?url=https://mcp.example.com/sse" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

This returns an authorization URL. The user must open it in their browser to complete OAuth.

Tell the user:
```
This MCP server requires authorization. Please open this URL in your browser:

{authorization_url}

Let me know once you've completed the authorization.
```

### Step 3: Create the Server

```bash
curl -s -X POST "{API_BASE}/task_mcp_server" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "My MCP Server",
  "url": "https://mcp.example.com/sse",
  "transport": "sse"
}
EOF
```

The response includes the server `id` (referred to as `task_mcp_server_id` when attaching to agents).

---

## Attach to Agent

When creating or updating an agent, include the mcp_servers array:

```json
"mcp_servers": [
  {
    "task_mcp_server_id": "server-uuid",
    "allowed_tool_ids": null
  }
]
```

### Tool Filtering

Like integrations, you can restrict which tools the agent can access:

```json
"mcp_servers": [
  {
    "task_mcp_server_id": "server-uuid",
    "allowed_tool_ids": ["tool_name_1", "tool_name_2"]
  }
]
```

- `allowed_tool_ids: null` = expose all tools from the server
- `allowed_tool_ids: [...]` = only expose listed tools

**When to filter:**
- The server exposes many tools but the agent only needs a few
- You want to prevent the agent from calling sensitive/destructive tools
- The total tool schemas would consume too much context

---

## Update Server

```bash
curl -s -X POST "{API_BASE}/task_mcp_server/{SERVER_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Updated Server Name"
}
EOF
```

---

## Delete Server

```bash
curl -s -X DELETE "{API_BASE}/task_mcp_server/{SERVER_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Any agents using this server will lose access to its tools on their next run.

---

## Update Semantics on Agents

The `mcp_servers` array on agents follows the same semantics as all array fields:
- **Omitted** = no change
- **Empty list `[]`** = remove all MCP servers
- **Non-empty list** = full replacement

---

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| Discovery fails | Server unreachable or wrong URL | Verify the URL is correct and the server is running |
| Auth required | Server needs OAuth | Run the authorize flow (Step 2) |
| Tool call timeout | Server slow to respond | Check server health; consider increasing agent timeout |
| Connection refused | Server down or firewall | Verify server is accessible from Rightbrain's infrastructure |
| Invalid transport | Wrong transport type | Try `sse` or `streamable_http` |

---

## Best Practices

1. **Always discover before creating** -- The discover step shows you exactly what tools are available, preventing surprises at runtime.

2. **Filter tools for focused agents** -- If the server exposes 20 tools but your agent only needs 3, use `allowed_tool_ids` to keep the agent's context clean.

3. **Test the server independently** -- Before attaching to an agent, verify the MCP server responds correctly to tool calls using the discover endpoint.

4. **Monitor for availability** -- MCP servers are external dependencies. If agent runs start failing, check server availability first.

5. **Use descriptive names** -- Name the server after the service it connects to (e.g., "Notion MCP" or "Internal CRM") so it is easy to identify in agent configurations.

6. **Re-discover after server updates** -- If the MCP server adds or removes tools, re-run discovery to see the current tool list before updating agent tool filters.
