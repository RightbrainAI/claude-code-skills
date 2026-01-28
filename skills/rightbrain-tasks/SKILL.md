---
name: rightbrain-tasks
description: |
  Manage Rightbrain tasks - create, browse, run, update, export, and import task configurations.
  Use when user wants to work with Rightbrain AI tasks.
  Trigger with "rightbrain tasks", "rightbrain task", "manage tasks", "create task", "run task", or "browse tasks".
---

# Rightbrain Task Manager

A comprehensive skill for managing Rightbrain tasks. Create new tasks following best practices, browse and run existing tasks, update configurations, and export/import task definitions.

## Core Principles

### 1. Stateless Execution
Every task run is independent—no memory of previous runs. All context must be passed as input.

### 2. Single Responsibility
One task does one thing well. If your description contains "and then" or "also", split it into multiple tasks.

### 3. Structured Outputs
Tasks return predictable, typed outputs. Use `output_format` to define the exact structure.

---

## Workflow

### Context Tracking

Throughout this workflow, you must track these values across phases:
- `API_KEY` - The user's Rightbrain API key (from env var or user input)
- `ORG_ID` - Selected organization UUID
- `PROJECT_ID` - Selected project UUID
- `TASK_ID` - Currently selected task UUID (when applicable)

These values are used in all subsequent API calls. Store them in your reasoning context as you progress through the workflow.

---

### Phase 0: Authentication & Setup

Before any API operations, establish credentials and select the working context.

**Exit option:** At any point in this phase, if the user says "cancel", "exit", or "never mind", exit the skill gracefully with: "No problem! Run this skill again when you're ready to work with Rightbrain tasks."

#### Step 1: Check for API Credentials

First, check for existing credentials in the environment:

```bash
[ -n "$RIGHTBRAIN_API_KEY" ] && echo "API key configured" || echo "API key not set"
```

**If environment variable exists (output shows "API key configured"):**
- Use the existing key
- Proceed to validation

**If environment variable is NOT set:**
- Ask user for their API key:
  ```
  To interact with the Rightbrain API, I need your API key.

  You can find it at: https://app.rightbrain.ai/preferences?tab=api-clients

  Please paste your API key:
  ```

**Security reminder:** Tell the user their API key will only be used for this session and won't be stored.

**API key usage in commands:**
- If using environment variable: Use `$RIGHTBRAIN_API_KEY` directly in curl commands to prevent shell history exposure
- If user provided key manually: Store in your context and substitute directly into commands (the key will appear in shell history, but this is unavoidable for session-based keys)

#### Step 2: Validate Credentials & Fetch Organizations

Validate the API key by fetching organizations:

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

**If 401 response:**
```
API key is invalid or expired. Please check your key at:
https://app.rightbrain.ai/preferences?tab=api-clients
```
Ask user to re-enter their API key.

**If 200 response:** Credentials valid - proceed with organization data.

#### Step 3: Select Organization

Parse the organization list from the response.

**If 0 organizations:**
```
You don't have any organizations yet.
Create one at: https://app.rightbrain.ai
```
Exit skill gracefully.

**If 1 organization:**
Auto-select and confirm:
```
Using your organization: {org_name} ({org_id})
```

**If multiple organizations:**
Use AskUserQuestion:
- Question: "Which organization do you want to work with?"
- Header: "Organization"
- Options: List each org as "{name}" with description showing the org ID

#### Step 4: Select Project

Fetch projects for the selected organization:

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

**If 0 projects:**
```
This organization has no projects yet.
Create one at: https://app.rightbrain.ai
```
Exit skill gracefully.

**If 1 project:**
Auto-select and confirm:
```
Using your project: {project_name} ({project_id})
```

**If multiple projects:**
Use AskUserQuestion:
- Question: "Which project do you want to work with?"
- Header: "Project"
- Options: List each project as "{name}" with description showing the project ID

#### Step 5: Prefetch Available Models (for Create/Import flows)

After selecting a project, fetch available models so they're ready for task creation or import:

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer $RIGHTBRAIN_API_KEY" \
  -H "Content-Type: application/json"
```

Store the model list in context. Key fields to note:
- `id`: UUID for `llm_model_id`
- `name`: Human-readable name (e.g., "GPT-4o", "Claude 3.5 Sonnet")
- `supports_vision`: Required for image tasks

This prefetch prevents users from designing a task only to discover their preferred model isn't available.

---

### Phase 1: Choose Action

After selecting organization and project, present the main action menu.

Use AskUserQuestion:
- Question: "What would you like to do?"
- Header: "Action"
- Options:
  - **Create new task** - Design a new Rightbrain task from scratch
  - **Browse existing tasks** - View, run, update, or export tasks
  - **Import task from file** - Load a task configuration from JSON file

**If "Create new task":** Proceed to [Task Design Workflow](#task-design-workflow-create-new-task)

**If "Browse existing tasks":** Proceed to [Task Browser](#task-browser)

**If "Import task from file":** Proceed to [Import Task Configuration](#import-task-configuration)

---

## Task Browser

### Fetch Tasks

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task?page_limit=50" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

### Display Task List

**If 0 tasks:**
```
This project has no tasks yet. Would you like to create one?
```
Offer to proceed to task creation.

**If tasks exist:**
Format each task as a summary for display:

```
{name}
  {description (first 80 chars)}...
  Modality: {output_modality} | Status: {enabled ? "Enabled" : "Disabled"}
```

Use AskUserQuestion to let user select a task:
- Question: "Select a task to manage:"
- Header: "Task"
- Options: List each task with name as label and summary as description
- Include a final option: "Create new task" - Start fresh with a new task design

### Task Actions

After selecting a task, present the action menu:

Use AskUserQuestion:
- Question: "What would you like to do with '{task_name}'?"
- Header: "Task action"
- Options:
  - **Run task** - Execute with input values
  - **Update task** - Modify prompts, model, or output format
  - **Export task** - Save configuration to file
  - **View details** - See full task configuration
  - **Back** - Return to task list

---

## Run Task Flow

### Step 1: Fetch Task Details

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

### Step 2: Parse Input Variables

Extract the active revision from the response (look for revision with `active: true`).

Parse the `user_prompt` for `{variable_name}` patterns using regex:
```
\{([a-zA-Z_][a-zA-Z0-9_]*)\}
```

Exclude common template patterns that aren't user inputs (like `{variable_1}` placeholders in documentation).

### Step 3: Collect Input Values

For each input variable found, ask the user to provide a value:

```
The task requires the following inputs:

{variable_name}: [inferred description from prompt context]

Please provide a value for {variable_name}:
```

### Step 4: Execute Task

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}/run" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "task_input": {
    "variable_1": "user_provided_value",
    "variable_2": "user_provided_value"
  }
}
EOF
```

**Critical:** All input parameters must be wrapped in a `task_input` object.

**JSON payload best practices:**

1. **Always use heredoc syntax** (`<< 'EOF'`) for JSON payloads - this is the most reliable method and handles special characters correctly.

2. **Avoid inline `-d '...'`** - Shell escaping can cause "Invalid JSON" errors with quotes or special characters.

3. **If `jq` is available**, it provides the safest JSON construction for complex values:
   ```bash
   # Alternative: Safe JSON construction with jq (if available)
   jq -n \
     --arg var1 "$USER_VALUE_1" \
     --arg var2 "$USER_VALUE_2" \
     '{task_input: {variable_1: $var1, variable_2: $var2}}' | \
   curl -s -X POST "..." -H "..." -d @-
   ```
   Note: `jq` may not be installed in all environments - heredoc is the reliable fallback.

### Step 5: Display Results

On success, display:
- The task output (`response` field)
- Token usage summary
- Credits consumed
- Link to view in dashboard: `https://app.rightbrain.ai/task/{task_id}/run?id={task_run_id}`

On failure, show the error and suggest fixes based on the error type.

### Step 6: Post-Run Menu

Use AskUserQuestion:
- Question: "Task completed. What next?"
- Header: "Next action"
- Options:
  - **Run again** - Same task with new inputs
  - **Different task** - Return to task browser
  - **Done** - Exit skill

**If "Run again":** Return to Step 3 (Collect Input Values)

**If "Different task":** Return to [Task Browser](#task-browser)

**If "Done":** Exit skill with summary of what was accomplished

---

## Update Existing Task

### Step 1: Fetch Current Configuration

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

### Step 2: Display Current Configuration

Extract the active revision and show:

```
Current Task Configuration:
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Name: {name}
Description: {description}
Model: {llm_model.name}
Temperature: {llm_config.temperature}
Output Modality: {output_modality}

System Prompt:
{system_prompt}

User Prompt:
{user_prompt}

Output Format:
{output_format as formatted JSON}
```

### Step 3: Collect Updates

Use AskUserQuestion:
- Question: "What would you like to update?"
- Header: "Update"
- Options:
  - **System prompt** - Modify the AI's role and behavior
  - **User prompt** - Modify instructions and input variables
  - **Output format** - Change the structured output schema
  - **Model settings** - Change model or temperature
  - **Multiple fields** - I'll specify several changes
  - **Cancel** - Return without changes

Based on selection, guide user through the specific update:
- For prompts: Show current value, ask for new value
- For model: Fetch available models and let user choose
- For output format: Guide through schema modification

### Step 4: Submit Update

**Important:** Updates create a new task revision. The new revision is **NOT** automatically active—you must explicitly activate it or run with the specific revision ID.

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "system_prompt": "updated prompt...",
  "user_prompt": "updated prompt...",
  "output_format": {...},
  "llm_model_id": "model-uuid",
  "llm_config": {"temperature": 0.5}
}
EOF
```

Only include fields that were changed.

The response will include the new revision with `"active": false`. Note the new `revision_id`.

### Step 5: Activate the New Revision

After creating a new revision, you have two options:

**Option A: Activate the revision (recommended for permanent changes)**

Update the task with `active_revisions` to make the new revision active:

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "active_revisions": [
    {"task_revision_id": "{NEW_REVISION_ID}", "weight": 1}
  ]
}
EOF
```

This makes the new revision the default for all future runs. The `weight` field supports traffic splitting (e.g., 0.5 for 50% of traffic).

**Option B: Run with specific revision (for testing)**

Use the `revision_id` query parameter to test before activating:

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}/run?revision_id={NEW_REVISION_ID}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "task_input": {
    "variable_1": "value"
  }
}
EOF
```

**Why this matters:**
- Creating a new revision does NOT automatically make it active
- Without activating or specifying `revision_id`, the task runs using the previously active revision
- Use Option B to test, then Option A to make permanent

### Step 6: Confirm Update

On success:
```
Task updated successfully!
New revision ID: {revision_id}

Testing the update now with the new revision...
```

Then immediately run the task with the new revision to verify the changes work as expected.

---

## Export Task Configuration

### Step 1: Fetch Task Details

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}?include_tests=false" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

### Step 2: Prepare Export

Extract the active revision configuration and transform for portability.

**Fields to EXCLUDE (security/portability):**
- `access_token` - Security risk if exported
- `id`, `task_id` - Will be regenerated on import
- `created`, `modified` - Timestamps
- `collection_id` - RAG collection (project-specific)
- `task_forwarder_id` - Task forwarding (project-specific)

**Fields to INCLUDE:**
- `name`, `description`, `enabled`
- `system_prompt`, `user_prompt`
- `output_modality`, `output_format`
- `llm_model_id` → Convert to `llm_model_name` for portability
- `llm_config`
- `input_processors`
- `image_required`

### Step 3: Generate Export JSON

```json
{
  "_export_version": "1.0",
  "_exported_at": "2026-01-23T12:00:00Z",
  "_source_project": "Project Name",
  "_notes": "RAG and task forwarding settings excluded - reconfigure after import if needed",

  "name": "Task Name",
  "description": "Task description",
  "enabled": true,
  "output_modality": "json",
  "system_prompt": "...",
  "user_prompt": "...",
  "output_format": {...},
  "llm_model_name": "gpt-4o",
  "llm_config": {"temperature": 0.3},
  "input_processors": [...],
  "image_required": false
}
```

### Step 4: Write Export File

Ask user for export path:
```
Where should I save the task configuration?
Default: ./tasks/{task-name-kebab}.json
```

Convert task name to kebab-case for filename (lowercase, spaces to hyphens).

Create directory if needed (`./tasks/`), then write the file.

Confirm:
```
Task exported successfully!
File: ./tasks/{filename}.json

You can import this task to any project using the import function.
```

---

## Import Task Configuration

### Step 1: Read Import File

Ask user for file path:
```
Please provide the path to the task configuration file:
```

Read and parse the JSON file.

### Step 2: Validate Configuration

1. Check `_export_version` for compatibility (currently "1.0")
2. Validate required fields exist: `name`, `system_prompt`, `user_prompt`, `output_modality`
3. Validate `output_format` matches modality rules

If validation fails, show specific error and exit.

### Step 3: Resolve Model

The export contains `llm_model_name`. Fetch available models and find matching UUID:

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

Search for model by name (case-insensitive match on `name` or `internal_name`).

**If model found:** Use its `id` as `llm_model_id`

**If model NOT found:**
```
The model '{llm_model_name}' is not available in this project.
```
Use AskUserQuestion to let user select an alternative model from available options.

### Step 4: Check for Name Conflicts

Fetch existing tasks and check if name already exists:

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task?name={task_name}" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

**If duplicate exists:**
Use AskUserQuestion:
- Question: "A task named '{name}' already exists. What should I do?"
- Header: "Conflict"
- Options:
  - **Use different name** - I'll provide a new name
  - **Auto-rename** - Create as "{name} (imported)"
  - **Cancel import** - Don't import this task

### Step 5: Create Task

Transform import config to TaskCreate format:
- Replace `llm_model_name` with resolved `llm_model_id`
- Remove export metadata fields (`_export_version`, `_exported_at`, `_source_project`, `_notes`)

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Task Name",
  "description": "...",
  "enabled": true,
  "system_prompt": "...",
  "user_prompt": "...",
  "output_modality": "json",
  "output_format": {...},
  "llm_model_id": "resolved-model-uuid",
  "llm_config": {...},
  "input_processors": [...],
  "image_required": false
}
EOF
```

### Step 6: Confirm Import

On success:
```
Task imported successfully!
Task ID: {task_id}

View and configure in dashboard: https://app.rightbrain.ai/task/{task_id}/run
(Access token available in task settings)

Note: If this task used RAG or task forwarding, you'll need to reconfigure those settings in the dashboard.
```

---

## Task Design Workflow (Create New Task)

This is the original task creation workflow for designing new tasks from scratch.

### Phase 1: Discovery

Start by understanding what the user wants to accomplish:

**Questions to ask:**
1. "What problem are you trying to solve?"
2. "What inputs will you have available?"
3. "What outputs do you need to make decisions?"

**Validation checks:**
| Red Flag | Problem | Solution |
|----------|---------|----------|
| "Keep track of..." | Requires state | Pass all context as input each run |
| "Remember the last..." | Needs memory | Use external storage, pass context as input |
| "Update the database..." | Side effects | Return data, handle storage separately |
| "Do X, then Y, then Z" | Multiple responsibilities | Split into separate composable tasks |
| "Analyze this" (vague) | Inconsistent results | Provide specific, directive instructions |

**Single responsibility check:**
If the task goal contains "and then" or "also", suggest splitting:
```
❌ "Classify the email AND extract key dates AND draft a reply"
✅ Task 1: "Classify email intent"
✅ Task 2: "Extract dates from email"
✅ Task 3: "Draft reply based on classification"
```

### Phase 2: Input Design

Define what the task receives. Inputs are defined in `user_prompt` using `{variable_name}` syntax.

**Input types:**

| Input Type | Implementation | Use Case |
|------------|----------------|----------|
| Text | `{variable_name}` in user_prompt | Customer message, document text |
| Image | `image_required: true` | Photo analysis, document scanning |
| PDF/DOC | `document_content_extractor` processor | Contract review, invoice parsing |
| URL | `url_fetcher` processor | Web scraping, link analysis |
| Web Search | `perplexity_search` processor | Research, fact-checking |

**Input processor configuration:**
```json
"input_processors": [
  {
    "param_name": "website_content",
    "input_processor": "url_fetcher"
  },
  {
    "param_name": "research_results",
    "input_processor": "perplexity_search",
    "config": {
      "max_results": 5,
      "query_context": "{research_results} latest news and features"
    }
  }
]
```

**Important - Input Processor Data Flow:**
- `param_name` is BOTH the input AND output variable
- The user passes a value for `param_name` (e.g., `{"research_results": "OpenAI"}`)
- The processor uses this value to perform its action (search, fetch, extract)
- The result REPLACES the original value in that same variable
- `query_context` must reference the SAME variable as `param_name`

**Example:** For a competitor analysis task:
```json
{
  "param_name": "research_results",
  "input_processor": "perplexity_search",
  "config": {
    "query_context": "{research_results} AI platform features pricing"
  }
}
```
User calls: `{"competitor_name": "OpenAI", "research_results": "OpenAI"}`
- `{competitor_name}` stays as "OpenAI" (for display in prompt)
- `{research_results}` becomes the Perplexity search results

### Phase 3: Prompt Engineering

Generate well-structured prompts using proven patterns.

**System prompt structure:**
```
You are a [role/expert type] specializing in [domain].

Your expertise includes:
- [Capability 1]
- [Capability 2]

Constraints:
- [Constraint 1]
- [Constraint 2]
```

**User prompt structure:**
```
## Goal
[One clear sentence describing what to accomplish]

## Context
[Background information about the task domain]

## Input
{variable_1}: [Description of what this variable contains]
{variable_2}: [Description of what this variable contains]

## Instructions
1. [Step 1]
2. [Step 2]
3. [Step 3]

## Output Requirements
- [Requirement 1]
- [Requirement 2]
```

See [Prompt Patterns Reference](references/prompt-patterns.md) for detailed templates.

### Phase 4: Output Schema Design

Define structured outputs based on the task type. See [Output Formats Reference](references/output-formats.md) for complete documentation.

**Quick reference - Output modalities:**
| Modality | When to Use | `output_format` |
|----------|-------------|-----------------|
| `json` | Structured data, classification | **Required** - define schema |
| `text` | Long-form content | Must be `null` |
| `image` / `audio` / `csv` / `pdf` | Media generation | Must be `null` |

**Common patterns:**

```json
// Enum (constrained choices)
"sentiment": {"type": "str", "options": ["positive", "negative", "neutral"]}

// Nested list
"items": {"type": "list", "item_type": "object", "nested_structure": {"name": "str", "value": "number"}}
```

### Phase 5: Model & Configuration

**Model selection guidance:**
| Task Type | Recommended Model | Temperature |
|-----------|-------------------|-------------|
| Classification | Fast model (e.g., GPT-4o-mini) | 0.0 - 0.3 |
| Extraction | Accurate model (e.g., GPT-4o) | 0.0 - 0.2 |
| Analysis | Reasoning model (e.g., Claude 3.5) | 0.3 - 0.5 |
| Generation | Creative model | 0.5 - 0.8 |
| Image Output | Image model (e.g., Gemini Flash Image) | 0.6 - 0.8 |

**Fallback model:** Configure `fallback_llm_model_id` for resilience.

**Vision tasks (image input):** Use models with `supports_vision: true`.

**Image generation tasks (image output):** Use models with `supports_image_output: true`. See [Image Generation Guide](references/image-generation.md).

### Phase 6: Generate Configuration

Output a complete, API-ready JSON configuration:

```json
{
  "name": "Task Name",
  "description": "What this task does",
  "enabled": true,
  "output_modality": "json",
  "system_prompt": "...",
  "user_prompt": "...",
  "llm_model_id": "UUID-of-selected-model",
  "llm_config": {
    "temperature": 0.3
  },
  "output_format": {
    "field_1": "str",
    "field_2": "int"
  },
  "image_required": false,
  "input_processors": []
}
```

### Phase 7: Deploy to Rightbrain API

Since we already have credentials from Phase 0, proceed directly to deployment.

**Step 1: Fetch available models**

```bash
curl -s -X GET "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json"
```

Parse the response to find the best model for the task type. Look for:
- `id`: The model UUID to use in `llm_model_id`
- `name`: Human-readable model name
- `supports_vision`: Required for image tasks

**Step 2: Create the task**

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{ORG_ID}/project/{PROJECT_ID}/task" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Task Name",
  ...complete task configuration...
}
EOF
```

**Step 3: Report results**

On success, tell the user:
- Task ID (for future reference)
- Task access token (for running the task)
- How to run the task via API
- Link to view task in dashboard: `https://app.rightbrain.ai/task/{task_id}/run`

On failure, parse the error and help fix it:
- 400 errors: Show validation message, suggest fix
- 400 `duplicate_task_name`: Use a unique task name (add version suffix like "v2")
- 401 errors: API key issue
- 403 errors: Permission issue

**Step 4: Offer to run the task**

Ask if the user wants to test the task immediately. If yes, proceed to [Run Task Flow](#run-task-flow).

---

## API Reference

### Base URL
- **Production**: `https://app.rightbrain.ai/api/v1`

### Endpoints Used

| Action | Method | Endpoint | Notes |
|--------|--------|----------|-------|
| List organizations | GET | `/org` | Returns user's organizations |
| List projects | GET | `/org/{org_id}/project` | Returns projects in org |
| List models | GET | `/org/{org_id}/project/{project_id}/model` | Returns available models |
| List tasks | GET | `/org/{org_id}/project/{project_id}/task` | Returns tasks in project |
| Get task | GET | `/org/{org_id}/project/{project_id}/task/{task_id}` | Returns task with revisions |
| Create task | POST | `/org/{org_id}/project/{project_id}/task` | Body: TaskCreate JSON |
| Update task | POST | `/org/{org_id}/project/{project_id}/task/{task_id}` | Creates new revision |
| Run task | POST | `/org/{org_id}/project/{project_id}/task/{task_id}/run` | Body: `{"task_input": {...}}` |

### Task Run Payload Format

**Critical:** Task inputs must be wrapped in `task_input`:

```json
{
  "task_input": {
    "variable_1": "value1",
    "variable_2": "value2"
  }
}
```

Passing parameters at the root level will return a validation error.

### Authentication

All requests require:
```
Authorization: Bearer {api_key}
Content-Type: application/json
```

---

## Error Handling

### HTTP Status Errors

| Status | Cause | Solution |
|--------|-------|----------|
| 400 | Validation error | Check error message, fix input |
| 400 | `duplicate_task_name` | Task name already exists - use unique name |
| 401 | Invalid/expired API key | Check key at dashboard, regenerate if needed |
| 403 | No permission on resource | Verify access in dashboard |
| 404 | Resource not found | Verify org/project/task IDs are correct |
| 429 | Rate limit exceeded | Wait 30 seconds before retrying |
| 500 | Server error | Retry after a few seconds |

### Network Errors

| Error Type | Cause | Solution |
|------------|-------|----------|
| Connection timeout | Network issue or server unreachable | Check internet connection, retry in 10 seconds |
| Connection refused | Server down or wrong URL | Verify API URL is correct, check Rightbrain status |
| SSL/TLS error | Certificate issue | Never use `-k` or `--insecure` flags - report to Rightbrain support |
| Empty response | Server returned no data | Retry request, check API status if persistent |

**Timeout handling:** Add `--connect-timeout 10 --max-time 30` to curl commands for long-running requests:

```bash
curl -s --connect-timeout 10 --max-time 30 -X POST "..." \
  -H "Authorization: Bearer $RIGHTBRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '...'
```

**Retry strategy:** For 429 and 5xx errors, wait and retry up to 3 times with exponential backoff (10s, 30s, 60s).

---

## Validation Rules

See [Task Components Reference](references/task-components.md) for complete validation rules.

**Key constraints:**
- **Task name:** Alphanumeric, hyphens, spaces only. Max 63 chars. Must be unique in project.
- **Output format keys:** Alphanumeric, underscore, dot, hyphen. 1-64 chars.
- **Modality rules:** `json` requires output_format; all others require `null`.
- **Input processors:** `param_name` must exist as `{variable}` in user_prompt.

---

## Quick Reference

### Task Types & Examples

| Pattern | Example | Key Features |
|---------|---------|--------------|
| **Classification** | Sentiment analysis, ticket routing | Enum outputs, low temperature |
| **Extraction** | Invoice parsing, entity recognition | Structured output, lists |
| **Generation** | Email drafting, content creation | Higher temperature, text output |
| **Analysis** | Risk assessment, competitive intel | Multiple output fields |
| **Validation** | Compliance check, quality review | Boolean outputs |
| **Image** | Social media visuals, product photos | `output_modality: "image"`, `supports_image_output` models |

### Common Configurations

**Classification task:**
```json
{
  "output_modality": "json",
  "output_format": {
    "category": {"type": "str", "options": ["A", "B", "C"]},
    "confidence": "number",
    "reasoning": "str"
  }
}
```

**Extraction task:**
```json
{
  "output_modality": "json",
  "image_required": true,
  "output_format": {
    "extracted_data": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "field": "str",
        "value": "str"
      }
    }
  }
}
```

**Generation task:**
```json
{
  "output_modality": "text",
  "output_format": null,
  "llm_config": {"temperature": 0.7}
}
```

**Image generation task:**
```json
{
  "output_modality": "image",
  "output_format": null,
  "llm_model_id": "MODEL_WITH_supports_image_output_TRUE",
  "llm_config": {"temperature": 0.8}
}
```

See [Image Generation Guide](references/image-generation.md) for text-prevention techniques and complete examples.

---

## References

- [Task Components Reference](references/task-components.md) - Complete schema documentation
- [Prompt Patterns Reference](references/prompt-patterns.md) - Prompt templates and examples
- [Output Formats Reference](references/output-formats.md) - Type system and validation
- [Image Generation Guide](references/image-generation.md) - Image tasks and text-prevention techniques
- [Task Template](assets/task-template.json) - Complete configuration template
- [Export Schema Example](assets/export-schema-example.json) - Example export file format
- [Classification Examples](assets/examples/classification.md)
- [Extraction Examples](assets/examples/extraction.md)
- [Generation Examples](assets/examples/generation.md)
