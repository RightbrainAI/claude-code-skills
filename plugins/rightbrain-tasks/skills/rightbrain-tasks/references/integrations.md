# Integrations Reference

Detailed workflows for connecting and managing Google Workspace integrations with Rightbrain agents.

> **Prerequisites:** Authentication and context (API_BASE, ACCESS_TOKEN, ORG_ID, PROJECT_ID) must be established before using these workflows. See SKILL.md Phase 0.

---

## Overview

Integrations connect Rightbrain agents to external services via OAuth. Currently supported: Google Workspace applications. Each integration exposes tools that agents can call to read and write data in the connected service.

**Available integration types:**
| Type | Service | Notes |
|------|---------|-------|
| `google_sheets` | Google Sheets | 17 tools, ~936k tokens in schemas. **Tool filtering mandatory.** |
| `google_docs` | Google Docs | Document read/write |
| `google_slides` | Google Slides | Presentation management |
| `google_calendar` | Google Calendar | Event management |
| `google_gmail` | Google Gmail | Email read/send |

---

## API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Browse catalog | GET | `/integration/catalog` |
| Create integration | POST | `/integration` |
| Get integration | GET | `/integration/{id}` |
| List integrations | GET | `/integration` |
| Authorize (OAuth) | POST | `/integration/{id}/authorize` |
| Check auth status | GET | `/integration/{id}/auth` |
| Disconnect | POST | `/integration/{id}/disconnect` |
| Delete integration | DELETE | `/integration/{id}` |

---

## Setup Workflow

### Step 1: Browse the Catalog

```bash
curl -s -X GET "{API_BASE}/integration/catalog" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

The catalog lists all available integration types with their descriptions and required scopes.

### Step 2: Create an Integration

```bash
curl -s -X POST "{API_BASE}/integration" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "type": "google_sheets"
}
EOF
```

The response includes the integration `id` needed for authorization.

### Step 3: Authorize via OAuth

```bash
curl -s -X POST "{API_BASE}/integration/{INTEGRATION_ID}/authorize" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

This returns an authorization URL. The user must open this URL in their browser to complete the OAuth flow with Google. After granting access, the integration is connected.

**Important:** This step requires browser interaction -- it cannot be completed entirely via CLI.

Tell the user:
```
I need to connect to Google. Please open this URL in your browser to authorize access:

{authorization_url}

Let me know once you've completed the authorization.
```

### Step 4: Verify Auth Status

After the user completes OAuth:

```bash
curl -s -X GET "{API_BASE}/integration/{INTEGRATION_ID}/auth" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Check that the status shows the integration is authorized and active.

---

## Attach to Agent

When creating or updating an agent, include the integrations array:

```json
"integrations": [
  {
    "project_integration_id": "integration-uuid",
    "allowed_tool_ids": ["values_get", "values_update", "values_append"]
  }
]
```

### Tool Filtering (CRITICAL for Google Sheets)

Google Sheets exposes 17 tools with schemas totaling approximately 936,000 tokens. Sending all of them to the agent wastes context and degrades performance.

**Always use `allowed_tool_ids` for Google Sheets.** Expose only the 3-4 tools the agent actually needs.

**Common Google Sheets tool selections:**

| Use Case | Recommended Tools |
|----------|-------------------|
| Read data | `values_get` |
| Write data | `values_update`, `values_append` |
| Read + Write | `values_get`, `values_update`, `values_append` |
| Full CRUD | `values_get`, `values_update`, `values_append`, `values_clear` |

**For other Google integrations** (Docs, Slides, Calendar, Gmail), tool filtering is optional but still recommended if the agent only needs a subset of capabilities.

### Finding Tool IDs

To see available tools for an integration, check the integration details:

```bash
curl -s -X GET "{API_BASE}/integration/{INTEGRATION_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

The response includes an `available_tools` list with each tool's `id` and `description`.

---

## Disconnect and Reconnect

### Disconnect

Revoke the OAuth connection without deleting the integration:

```bash
curl -s -X POST "{API_BASE}/integration/{INTEGRATION_ID}/disconnect" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

The integration remains configured but agents can no longer use it until re-authorized.

### Reconnect

Simply run the authorize flow again (Step 3 above).

### Delete

Permanently remove the integration and all its configuration:

```bash
curl -s -X DELETE "{API_BASE}/integration/{INTEGRATION_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Any agents using this integration will lose access to its tools on their next run.

---

## Update Semantics on Agents

The `integrations` array on agents follows the same semantics as all array fields:
- **Omitted** = no change
- **Empty list `[]`** = remove all integrations
- **Non-empty list** = full replacement

To add an integration without removing existing ones:
1. GET the current agent configuration
2. Append the new integration to the existing `integrations` array
3. POST the complete array

---

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| 401 on authorize | Token expired | Re-authenticate with `npx rightbrain@latest login` |
| OAuth flow fails | User denied access | Re-run authorize, ensure correct Google account |
| Integration not authorized | OAuth not completed | Run authorize flow (Step 3) |
| Tool call fails at runtime | Token revoked | Disconnect and re-authorize |
| Context too large | Too many tools exposed | Use `allowed_tool_ids` to filter (especially Sheets) |

---

## Best Practices

1. **Always filter Google Sheets tools** -- The 936k token schema will consume most of the agent's context. Only expose what you need.

2. **Check auth before attaching** -- Verify the integration is authorized before attaching it to an agent. An unauthorized integration will cause runtime failures.

3. **Use descriptive agent instructions** -- When an agent has integrations, mention the service in the instruction so the agent knows it has access: "You have access to the team's Google Sheets spreadsheet for reading sales data."

4. **Handle disconnection gracefully** -- If a user revokes Google access externally, the integration will fail. Check auth status if agents report tool errors.

5. **One integration per service type** -- Create one Google Sheets integration and share it across agents via `project_integration_id`, rather than creating separate integrations per agent.
