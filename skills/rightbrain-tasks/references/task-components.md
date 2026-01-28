# Task Components Reference

Complete schema reference for Rightbrain task configuration.

## Task Create Schema

The `TaskCreate` schema combines task-level and revision-level fields:

```json
{
  // Task-level fields
  "name": "string",
  "description": "string | null",
  "enabled": "boolean",
  "output_modality": "json | image | audio | pdf | text | csv",
  "public": "boolean",
  "exposed_to_agents": "boolean",
  "task_run_visibility": "owner_only | editors_and_owners | all_viewers",

  // Revision-level fields (defines task behavior)
  "system_prompt": "string",
  "user_prompt": "string",
  "llm_model_id": "UUID",
  "llm_config": "object",
  "output_format": "object | null",
  "image_required": "boolean",
  "optimise_images": "boolean",
  "input_processors": "array",
  "rag": "object | null",
  "task_forwarder_id": "UUID | null",
  "fallback_llm_model_id": "UUID | null",
  "fallback_llm_config": "object | null",
  "annotation": "string | null"
}
```

---

## Field Reference

### Task-Level Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | - | Task identifier. Pattern: `^[a-zA-Z0-9\- ]+$`, max 63 chars |
| `description` | string | No | null | Internal reference description |
| `enabled` | boolean | Yes | - | When true, task is callable |
| `output_modality` | enum | No | "json" | Output type: json, image, audio, pdf, text, csv |
| `public` | boolean | No | false | When true, accessible by any user |
| `exposed_to_agents` | boolean | No | false | When true, accessible via unauthenticated agent endpoint |
| `task_run_visibility` | enum | No | "owner_only" | Who can view task runs |

### Revision-Level Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `system_prompt` | string | Yes | - | Sets LLM context and role |
| `user_prompt` | string | Yes | - | Task instructions with `{variable}` placeholders |
| `llm_model_id` | UUID | Yes | - | Primary model to use |
| `llm_config` | object | No | {} | Model configuration (e.g., temperature) |
| `output_format` | object | Conditional | null | Required for json modality, null for others |
| `image_required` | boolean | No | false | Require image in task run request |
| `optimise_images` | boolean | No | true | Auto-optimize images before processing |
| `input_processors` | array | No | [] | Pre-processing transformations |
| `rag` | object | No | null | RAG collection configuration |
| `task_forwarder_id` | UUID | No | null | Webhook/email forwarding |
| `fallback_llm_model_id` | UUID | No | null | Fallback model if primary fails |
| `fallback_llm_config` | object | No | null | Fallback model configuration |
| `annotation` | string | No | null | Note about this revision (max 255 chars) |

---

## Input Processors

Pre-process input values before they reach the LLM.

### Configuration

```json
{
  "input_processors": [
    {
      "param_name": "variable_name",
      "input_processor": "processor_type",
      "config": {}
    }
  ]
}
```

**Important - Data Flow:**
- `param_name` serves as BOTH input AND output
- User passes a value for `param_name`
- Processor transforms this value and replaces it with the result
- The variable in `user_prompt` receives the processed output

### Available Processors

#### `url_fetcher`
Fetches HTML content from a URL and converts to markdown.

```json
{
  "param_name": "website_content",
  "input_processor": "url_fetcher",
  "config": {
    "fallback_text": "Unable to fetch URL content"
  }
}
```

**Usage:** User passes `{"website_content": "https://example.com"}` → variable receives fetched markdown.

| Config | Type | Default | Description |
|--------|------|---------|-------------|
| `fallback_text` | string | null | Text to return on error |

#### `document_content_extractor`
Extracts text from PDF, DOC, and other document formats.

```json
{
  "param_name": "document_text",
  "input_processor": "document_content_extractor",
  "config": {
    "filename": "contract.pdf",
    "strategy": "fast"
  }
}
```

**Usage:** User uploads file with matching filename → variable receives extracted text.

| Config | Type | Default | Options | Description |
|--------|------|---------|---------|-------------|
| `filename` | string | Required | - | Target filename to extract |
| `strategy` | string | "fast" | fast, hi_res, ocr_only, auto | Extraction strategy |

#### `perplexity_search`
Performs web search and returns results.

```json
{
  "param_name": "research_results",
  "input_processor": "perplexity_search",
  "config": {
    "max_results": 5,
    "query_context": "{research_results} latest news and updates",
    "fallback_text": "Search unavailable"
  }
}
```

**Usage:** User passes `{"research_results": "OpenAI"}` → Perplexity searches for "OpenAI latest news and updates" → variable receives search results.

**Critical:** `query_context` must reference the SAME variable as `param_name`. The placeholder controls where the user's input appears in the search query.

| Config | Type | Default | Description |
|--------|------|---------|-------------|
| `max_results` | int | 5 | Number of results (1-100) |
| `query_context` | string | null | Query template - must use same variable as `param_name` |
| `fallback_text` | string | null | Text to return on error |

---

## RAG Configuration

Connect tasks to knowledge collections for domain-specific context.

```json
{
  "rag": {
    "collection_id": "UUID-of-collection",
    "rag_param": "context"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `collection_id` | UUID | Yes | Knowledge collection to query |
| `rag_param` | string | Yes | Parameter name to inject retrieved context |

**Note:** RAG is a paid tier feature.

---

## LLM Configuration

Common `llm_config` options:

```json
{
  "llm_config": {
    "temperature": 0.3,
    "max_tokens": 1000,
    "top_p": 1.0
  }
}
```

| Option | Type | Range | Description |
|--------|------|-------|-------------|
| `temperature` | number | 0.0 - 2.0 | Randomness (0 = deterministic) |
| `max_tokens` | int | 1 - model limit | Max response length |
| `top_p` | number | 0.0 - 1.0 | Nucleus sampling |

### Temperature Guidelines

| Task Type | Temperature | Reasoning |
|-----------|-------------|-----------|
| Classification | 0.0 - 0.3 | Consistent, repeatable results |
| Extraction | 0.0 - 0.2 | Accurate data capture |
| Analysis | 0.3 - 0.5 | Balanced reasoning |
| Generation | 0.5 - 0.8 | Creative variation |
| Brainstorming | 0.8 - 1.0 | Maximum creativity |

---

## Task Forwarders

Forward task results to external systems.

### Webhook Forwarder

```json
{
  "task_forwarder_id": "UUID",
  "config": {
    "destination_url": "https://your-webhook.com/endpoint"
  },
  "config_sensitive": {
    "signing_key": "your-secret-key"
  }
}
```

### Email Forwarder

```json
{
  "task_forwarder_id": "UUID",
  "config": {
    "recipient_emails": ["team@example.com"],
    "subject_template": "Task Result: {task_name}",
    "body_template": "Result: {response}"
  }
}
```

---

## Validation Summary

### Name Validation
- Pattern: `^[a-zA-Z0-9\- ]+$`
- Max length: 63 characters
- **Must be unique within the project**

### Output Modality Rules
| Modality | `output_format` |
|----------|-----------------|
| json | Required (at least 1 field) |
| image | Must be null |
| text | Must be null |
| csv | Must be null |
| pdf | Must be null |
| audio | Must be null |

### Output Format Key Validation
- Pattern: `^[a-zA-Z0-9_.-]{1,64}$`

### Input Processor Validation
- `param_name` must exist in `user_prompt` variables

### Revision Constraints
- `test: true` cannot be set on task creation (tasks need at least one active revision)

---

## Task Run Schema

To execute a task, POST to `/org/{org_id}/project/{project_id}/task/{task_id}/run`.

### Request Payload

**Critical:** All input parameters must be wrapped in a `task_input` object:

```json
{
  "task_input": {
    "variable_1": "value for variable_1",
    "variable_2": "value for variable_2"
  }
}
```

**Common mistake:** Passing parameters at root level will fail:
```json
// ❌ WRONG - will return validation error
{
  "variable_1": "value",
  "variable_2": "value"
}

// ✅ CORRECT - wrapped in task_input
{
  "task_input": {
    "variable_1": "value",
    "variable_2": "value"
  }
}
```

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `task_input` | object | **Required.** Key-value pairs matching `user_prompt` variables |
| `reference` | string | Optional reference ID for tracking |
| `context_id` | UUID | Optional context for conversation continuity |
| `reporting_group_id` | UUID | Optional grouping for analytics |

### Response Schema

```json
{
  "id": "task-run-uuid",
  "task_id": "task-uuid",
  "task_revision_id": "revision-uuid",
  "response": {
    // Structured output matching output_format
  },
  "run_data": {
    "submitted": {
      // Original inputs
    },
    "variable_name_preprocessed": "original value before processing"
  },
  "files": [],
  "is_error": false,
  "llm_retries": 0,
  "used_fallback_model": false,
  "input_tokens": 1234,
  "output_tokens": 567,
  "total_tokens": 1801,
  "charged_credits": "10",
  "created": "2024-01-01T00:00:00Z"
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique task run identifier |
| `response` | object/string | Task output (structure depends on `output_modality`) |
| `run_data.submitted` | object | Final inputs after processor transformation |
| `is_error` | boolean | Whether the run encountered an error |
| `used_fallback_model` | boolean | Whether fallback model was used |
| `input_tokens` | int | Tokens in the prompt |
| `output_tokens` | int | Tokens in the response |
| `charged_credits` | string | Credits consumed for this run |

### Example: Running a Task with Perplexity Search

For a task with `perplexity_search` input processor:

```bash
curl -s -X POST "https://app.rightbrain.ai/api/v1/org/{org_id}/project/{project_id}/task/{task_id}/run" \
  -H "Authorization: Bearer {api_key}" \
  -H "Content-Type: application/json" \
  -d '{
    "task_input": {
      "competitor_name": "Langdock",
      "research_results": "Langdock"
    }
  }'
```

In the response:
- `run_data.submitted.research_results` contains the Perplexity search results
- `run_data.research_results_preprocessed` contains the original value ("Langdock")

### Dashboard URLs

| View | URL Format |
|------|------------|
| Task overview | `https://app.rightbrain.ai/task/{task_id}/run` |
| Specific task run | `https://app.rightbrain.ai/task/{task_id}/run?id={task_run_id}` |

**Example:**
```
https://app.rightbrain.ai/task/019be741-6b7c-6d21-5821-d122703f975e/run?id=019be743-aa40-e3e7-9966-2aae7848af1e
```
