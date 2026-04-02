# Output Formats Reference

Complete guide to Rightbrain task output configuration.

## Output Modalities

Tasks can output different types of content based on `output_modality`:

| Modality | Description | `output_format` Requirement |
|----------|-------------|----------------------------|
| `json` | Structured JSON data | **Required** - must define fields |
| `text` | Plain text content | **Must be null** |
| `image` | Generated images | **Must be null** |
| `csv` | Tabular CSV data | **Must be null** |
| `pdf` | PDF documents | **Must be null** |
| `audio` | Audio content | **Must be null** |

### JSON Modality (Most Common)

For structured data extraction, classification, and analysis:

```json
{
  "output_modality": "json",
  "output_format": {
    "field_name": "type"
  }
}
```

### Text Modality

For long-form content generation:

```json
{
  "output_modality": "text",
  "output_format": null
}
```

### Image Modality

For image generation tasks (requires vision-capable model):

```json
{
  "output_modality": "image",
  "output_format": null
}
```

---

## Output Format Types

The `output_format` object defines the structure of JSON responses.

### Primitive Types

| Type | Aliases | Description | Example Value |
|------|---------|-------------|---------------|
| `str` | `string` | Text content | `"Hello world"` |
| `int` | `integer` | Whole numbers | `42` |
| `number` | - | Decimal numbers | `3.14159` |
| `bool` | `boolean` | True/false | `true` |

### Shorthand Notation

Simple types can use shorthand:

```json
{
  "output_format": {
    "title": "str",
    "count": "int",
    "price": "number",
    "is_valid": "bool"
  }
}
```

### Expanded Notation

For additional options, use the expanded format:

```json
{
  "output_format": {
    "category": {
      "type": "str",
      "description": "The primary category classification"
    }
  }
}
```

---

## Enum Values (Constrained Choices)

Use `options` to restrict values to a specific set:

```json
{
  "output_format": {
    "sentiment": {
      "type": "str",
      "options": ["positive", "negative", "neutral"]
    },
    "priority": {
      "type": "str",
      "options": ["low", "medium", "high", "critical"]
    }
  }
}
```

**Response will be exactly one of the specified options.**

---

## Lists

Arrays of values use `type: "list"` with `item_type`:

### Simple Lists

```json
{
  "output_format": {
    "tags": {
      "type": "list",
      "item_type": "str"
    },
    "scores": {
      "type": "list",
      "item_type": "int"
    }
  }
}
```

**Output:**
```json
{
  "tags": ["important", "urgent", "customer"],
  "scores": [85, 92, 78]
}
```

### List of Objects

For complex nested structures:

```json
{
  "output_format": {
    "line_items": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "description": "str",
        "quantity": "int",
        "unit_price": "number"
      }
    }
  }
}
```

**Output:**
```json
{
  "line_items": [
    {"description": "Widget A", "quantity": 5, "unit_price": 19.99},
    {"description": "Widget B", "quantity": 2, "unit_price": 29.99}
  ]
}
```

---

## Nested Objects

For hierarchical data structures:

```json
{
  "output_format": {
    "contact": {
      "type": "object",
      "nested_structure": {
        "name": "str",
        "email": "str",
        "phone": "str"
      }
    },
    "address": {
      "type": "object",
      "nested_structure": {
        "street": "str",
        "city": "str",
        "country": "str",
        "postal_code": "str"
      }
    }
  }
}
```

**Output:**
```json
{
  "contact": {
    "name": "John Smith",
    "email": "john@example.com",
    "phone": "+1-555-0123"
  },
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "country": "USA",
    "postal_code": "94102"
  }
}
```

---

## Complex Examples

### Invoice Extraction

```json
{
  "output_format": {
    "vendor_name": "str",
    "invoice_number": "str",
    "invoice_date": "str",
    "due_date": "str",
    "subtotal": "number",
    "tax": "number",
    "total": "number",
    "currency": {
      "type": "str",
      "options": ["USD", "EUR", "GBP", "CAD", "AUD"]
    },
    "line_items": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "description": "str",
        "quantity": "int",
        "unit_price": "number",
        "total": "number"
      }
    },
    "payment_terms": "str",
    "notes": "str"
  }
}
```

### Lead Scoring

```json
{
  "output_format": {
    "score": {
      "type": "int",
      "description": "Score from 0-100"
    },
    "tier": {
      "type": "str",
      "options": ["hot", "warm", "cold"]
    },
    "signals": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "signal_type": {
          "type": "str",
          "options": ["positive", "negative", "neutral"]
        },
        "description": "str",
        "weight": "number"
      }
    },
    "recommended_action": "str",
    "talking_points": {
      "type": "list",
      "item_type": "str"
    }
  }
}
```

### Content Analysis

```json
{
  "output_format": {
    "summary": "str",
    "sentiment": {
      "type": "str",
      "options": ["positive", "negative", "neutral", "mixed"]
    },
    "confidence": "number",
    "key_topics": {
      "type": "list",
      "item_type": "str"
    },
    "entities": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "name": "str",
        "type": {
          "type": "str",
          "options": ["person", "organization", "location", "product", "other"]
        }
      }
    },
    "action_items": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "action": "str",
        "assignee": "str",
        "priority": {
          "type": "str",
          "options": ["low", "medium", "high"]
        }
      }
    }
  }
}
```

---

## Validation Rules

### Output Format Keys

All keys must match: `^[a-zA-Z0-9_.-]{1,64}$`

| Valid | Invalid | Reason |
|-------|---------|--------|
| `customer_name` | `customer name` | No spaces |
| `line_items` | `line-items!` | No special chars except `_.-` |
| `price.usd` | `price/usd` | No forward slashes |
| `v2_response` | `2_response` | Must start with letter |

### Nested Structure Rules

1. `nested_structure` is only valid when `type` is `list` or `object`
2. When `type` is `list` with `item_type: "object"`, `nested_structure` is **required**
3. When `type` is `object`, `nested_structure` is **required**

```json
// ✅ Valid - object type with nested_structure
{
  "contact": {
    "type": "object",
    "nested_structure": {"name": "str", "email": "str"}
  }
}

// ❌ Invalid - object type without nested_structure
{
  "contact": {
    "type": "object"
  }
}

// ✅ Valid - list of objects with nested_structure
{
  "items": {
    "type": "list",
    "item_type": "object",
    "nested_structure": {"name": "str", "price": "number"}
  }
}

// ❌ Invalid - list of objects without nested_structure
{
  "items": {
    "type": "list",
    "item_type": "object"
  }
}
```

### Item Type Rules

`item_type` is only valid when `type` is `list`:

```json
// ✅ Valid
{"tags": {"type": "list", "item_type": "str"}}

// ❌ Invalid - item_type on non-list
{"name": {"type": "str", "item_type": "str"}}
```

---

## Common Patterns

### Optional Fields

All fields are technically optional in the LLM output. Use descriptions to indicate required vs optional:

```json
{
  "output_format": {
    "name": {
      "type": "str",
      "description": "Customer name (required)"
    },
    "nickname": {
      "type": "str",
      "description": "Optional nickname, null if not provided"
    }
  }
}
```

### Boolean Flags

Use for yes/no decisions:

```json
{
  "output_format": {
    "is_spam": "bool",
    "contains_pii": "bool",
    "requires_review": "bool"
  }
}
```

### Scores and Ratings

```json
{
  "output_format": {
    "quality_score": {
      "type": "int",
      "description": "Score from 1-10"
    },
    "confidence": {
      "type": "number",
      "description": "Confidence level from 0.0 to 1.0"
    }
  }
}
```

### Reasoning Fields

Include reasoning for transparency:

```json
{
  "output_format": {
    "decision": {
      "type": "str",
      "options": ["approve", "reject", "review"]
    },
    "reasoning": {
      "type": "str",
      "description": "Explanation for the decision"
    },
    "factors": {
      "type": "list",
      "item_type": "str",
      "description": "Key factors that influenced the decision"
    }
  }
}
```
