# Tasks Reference

Detailed workflows for creating, browsing, running, updating, exporting, and importing Rightbrain Tasks.

> **Prerequisites:** Authentication and context (ACCESS_TOKEN, ORG_ID, PROJECT_ID) must be established before using these workflows. See SKILL.md Phase 0.

---

## Task Browser

### Fetch Tasks

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task?page_limit=50" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Display Task List

**If 0 tasks:**
```
This project has no tasks yet. Would you like to create one?
```
Offer to proceed to task creation.

**If tasks exist:**
Format each task as a summary:

```
{name}
  {description (first 80 chars)}...
  Modality: {output_modality} | Status: {enabled ? "Enabled" : "Disabled"}
```

Use AskUserQuestion to let the user select a task:
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
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Step 2: Parse Input Variables

Extract the active revision from the response (look for revision with `active: true`).

Parse the `user_prompt` for `{variable_name}` patterns using regex:
```
\{([a-zA-Z_][a-zA-Z0-9_]*)\}
```

Exclude common template patterns that are not user inputs.

### Step 3: Collect Input Values

For each input variable found, ask the user to provide a value:

```
The task requires the following inputs:

{variable_name}: [inferred description from prompt context]

Please provide a value for {variable_name}:
```

### Step 4: Execute Task

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}/run" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
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

1. **Always use heredoc syntax** (`<< 'EOF'`) for JSON payloads - handles special characters correctly.
2. **Avoid inline `-d '...'`** - Shell escaping can cause "Invalid JSON" errors.
3. **If `jq` is available**, use it for safe JSON construction with complex values:
   ```bash
   jq -n \
     --arg var1 "$USER_VALUE_1" \
     --arg var2 "$USER_VALUE_2" \
     '{task_input: {variable_1: $var1, variable_2: $var2}}' | \
   curl -s -X POST "..." -H "..." -d @-
   ```

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

---

## Update Existing Task

### Step 1: Fetch Current Configuration

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Step 2: Display Current Configuration

Extract the active revision and show:

```
Current Task Configuration:
---
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

### Step 4: Submit Update

Updates create a new task revision. The new revision is **NOT** automatically active.

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "system_prompt": "updated prompt...",
  "user_prompt": "updated prompt...",
  "output_format": {},
  "llm_model_id": "model-uuid",
  "llm_config": {"temperature": 0.5}
}
EOF
```

Only include fields that were changed. The response includes the new revision with `"active": false`.

### Step 5: Activate the New Revision

**Option A: Activate the revision (recommended for permanent changes)**

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "active_revisions": [
    {"task_revision_id": "{NEW_REVISION_ID}", "weight": 1}
  ]
}
EOF
```

The `weight` field supports traffic splitting (e.g., 0.5 for 50% of traffic).

**Option B: Run with specific revision (for testing)**

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}/run?revision_id={NEW_REVISION_ID}" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
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

Then immediately run the task with the new revision to verify changes.

---

## Export Task Configuration

### Step 1: Fetch Task Details

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task/{TASK_ID}?include_tests=false" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
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
- `llm_model_id` -> Convert to `llm_model_name` for portability
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
  "output_format": {},
  "llm_model_name": "gpt-4o",
  "llm_config": {"temperature": 0.3},
  "input_processors": [],
  "image_required": false
}
```

### Step 4: Write Export File

Ask user for export path:
```
Where should I save the task configuration?
Default: ./tasks/{task-name-kebab}.json
```

Convert task name to kebab-case for filename (lowercase, spaces to hyphens). Create directory if needed, then write the file.

---

## Import Task Configuration

### Step 1: Read Import File

Ask user for file path, then read and parse the JSON file.

### Step 2: Validate Configuration

1. Check `_export_version` for compatibility (currently "1.0")
2. Validate required fields: `name`, `system_prompt`, `user_prompt`, `output_modality`
3. Validate `output_format` matches modality rules

If validation fails, show specific error and exit.

### Step 3: Resolve Model

The export contains `llm_model_name`. Fetch available models and find matching UUID:

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Search by name (case-insensitive match on `name` or `internal_name`).

**If model NOT found:** Use AskUserQuestion to let user select an alternative model.

### Step 4: Check for Name Conflicts

Fetch existing tasks and check if name already exists. If duplicate, offer:
- **Use different name** - I'll provide a new name
- **Auto-rename** - Create as "{name} (imported)"
- **Cancel import** - Don't import

### Step 5: Create Task

Transform import config to TaskCreate format:
- Replace `llm_model_name` with resolved `llm_model_id`
- Remove export metadata fields (`_export_version`, `_exported_at`, `_source_project`, `_notes`)

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "name": "Task Name",
  "description": "...",
  "enabled": true,
  "system_prompt": "...",
  "user_prompt": "...",
  "output_modality": "json",
  "output_format": {},
  "llm_model_id": "resolved-model-uuid",
  "llm_config": {},
  "input_processors": [],
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

Note: If this task used RAG or task forwarding, reconfigure those settings in the dashboard.
```

---

## Task Design Workflow (Create New Task)

### Phase 1: Discovery

Start by understanding what the user wants to accomplish.

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
Bad:  "Classify the email AND extract key dates AND draft a reply"
Good: Task 1: "Classify email intent"
      Task 2: "Extract dates from email"
      Task 3: "Draft reply based on classification"
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
- The processor uses this value to perform its action
- The result REPLACES the original value in that same variable
- `query_context` must reference the SAME variable as `param_name`

### Phase 3: Prompt Engineering

Generate well-structured prompts. See [Prompt Patterns Reference](prompt-patterns.md) for detailed templates.

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

### Phase 4: Output Schema Design

Define structured outputs. See [Output Formats Reference](output-formats.md) for complete documentation.

**Quick reference:**
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
  "llm_config": {"temperature": 0.3},
  "output_format": {
    "field_1": "str",
    "field_2": "int"
  },
  "image_required": false,
  "input_processors": []
}
```

### Phase 7: Deploy to Rightbrain API

**Step 1: Fetch available models** (skip if already prefetched)

```bash
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

**Step 2: Create the task**

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/task" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
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
- 401 errors: Session expired - re-run `npx rightbrain@latest login`
- 403 errors: Permission issue

**Step 4: Offer to run the task**

Ask if the user wants to test the task immediately. If yes, proceed to [Run Task Flow](#run-task-flow).

---

## API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| List tasks | GET | `/org/{org_id}/project/{project_id}/task` |
| Get task | GET | `/org/{org_id}/project/{project_id}/task/{task_id}` |
| Create task | POST | `/org/{org_id}/project/{project_id}/task` |
| Update task | POST | `/org/{org_id}/project/{project_id}/task/{task_id}` |
| Run task | POST | `/org/{org_id}/project/{project_id}/task/{task_id}/run` |

---

## Task Types Quick Reference

| Pattern | Example | Key Features |
|---------|---------|--------------|
| **Classification** | Sentiment analysis, ticket routing | Enum outputs, low temperature |
| **Extraction** | Invoice parsing, entity recognition | Structured output, lists |
| **Generation** | Email drafting, content creation | Higher temperature, text output |
| **Analysis** | Risk assessment, competitive intel | Multiple output fields |
| **Validation** | Compliance check, quality review | Boolean outputs |
| **Image** | Social media visuals, product photos | `output_modality: "image"`, `supports_image_output` models |

---

## Related References

- [Task Components Reference](task-components.md) - Complete schema documentation
- [Prompt Patterns Reference](prompt-patterns.md) - Prompt templates and examples
- [Output Formats Reference](output-formats.md) - Type system and validation
- [Image Generation Guide](image-generation.md) - Image tasks and text-prevention techniques
