# Extraction Task Examples

Extraction tasks pull structured data from unstructured content. They're characterized by:
- Very low temperature (0.0-0.2) for accuracy
- Complex nested output structures
- Often use image input or document processors
- Focus on precision over creativity

---

## Example 1: Invoice Processing (Image Input)

Extract structured data from invoice documents.

```json
{
  "name": "Invoice Extractor",
  "description": "Extract structured data from invoice images or PDFs",
  "enabled": true,
  "output_modality": "json",

  "system_prompt": "You are a document processing specialist with expertise in invoice analysis.\n\nYou extract data with high precision, handling various invoice formats from different countries and industries. You understand common invoice layouts, date formats, and currency conventions.\n\nConstraints:\n- Use null for fields that cannot be found\n- Do not infer or calculate values - extract only what is explicitly stated\n- Flag ambiguous values for human review",

  "user_prompt": "## Goal\nExtract all relevant data from this invoice document.\n\n## Document\nAnalyze the provided invoice image.\n\n## Fields to Extract\n- Vendor/supplier information (name, address, tax ID)\n- Invoice metadata (number, date, due date, PO number)\n- Line items with descriptions, quantities, and prices\n- Totals (subtotal, tax, discounts, total)\n\n## Extraction Rules\n- Dates should be in YYYY-MM-DD format\n- Currency amounts should be numbers without symbols\n- If tax rate is shown as percentage, convert to decimal (e.g., 20% = 0.20)",

  "llm_model_id": "VISION_MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.0
  },

  "output_format": {
    "vendor": {
      "type": "object",
      "nested_structure": {
        "name": "str",
        "address": "str",
        "tax_id": "str"
      }
    },
    "invoice_number": "str",
    "invoice_date": "str",
    "due_date": "str",
    "currency": {
      "type": "str",
      "options": ["USD", "EUR", "GBP", "CAD", "AUD", "OTHER"]
    },
    "line_items": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "description": "str",
        "quantity": "number",
        "unit_price": "number",
        "total": "number"
      }
    },
    "subtotal": "number",
    "tax_amount": "number",
    "total_amount": "number",
    "extraction_confidence": {
      "type": "str",
      "options": ["high", "medium", "low"]
    },
    "requires_review": "bool"
  },

  "image_required": true,
  "optimise_images": true,
  "input_processors": []
}
```

**Key patterns:**
- `image_required: true` for vision-based extraction
- Nested objects for grouped data (vendor, line items)
- Confidence and review flags for quality control

---

## Example 2: Contract Entity Extraction (Document Processor)

Extract key entities and clauses from contracts using document processing.

```json
{
  "name": "Contract Entity Extractor",
  "description": "Extract parties, dates, terms, and key clauses from contracts",
  "enabled": true,
  "output_modality": "json",

  "system_prompt": "You are a legal document analyst specializing in contract review.\n\nYou extract key information from contracts with high accuracy, identifying parties, obligations, and important clauses. You understand legal terminology and contract structures.\n\nConstraints:\n- Extract only explicitly stated information\n- Do not provide legal advice or interpretations\n- Flag unusual or potentially problematic clauses",

  "user_prompt": "## Goal\nExtract key entities, dates, and clauses from this contract.\n\n## Contract Text\n{contract_text}\n\n## Information to Extract\n1. **Parties**: Names, roles, and signatories\n2. **Dates**: Effective date, termination date, renewal dates\n3. **Financial Terms**: Amounts, payment schedules\n4. **Key Clauses**: Termination, liability, confidentiality\n\n## Extraction Guidelines\n- Identify the contract type (NDA, SaaS, Employment, etc.)\n- Note any auto-renewal provisions\n- Flag one-sided or unusual terms",

  "llm_model_id": "MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.0
  },

  "output_format": {
    "contract_type": {
      "type": "str",
      "options": ["nda", "saas_agreement", "employment", "consulting", "license", "other"]
    },
    "parties": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "name": "str",
        "role": {
          "type": "str",
          "options": ["party_a", "party_b", "vendor", "customer", "employer", "employee"]
        },
        "signatory_name": "str"
      }
    },
    "effective_date": "str",
    "termination_date": "str",
    "auto_renewal": "bool",
    "key_clauses": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "clause_type": {
          "type": "str",
          "options": ["termination", "liability", "confidentiality", "ip_ownership", "non_compete", "other"]
        },
        "summary": "str",
        "section_reference": "str"
      }
    },
    "flags": {
      "type": "list",
      "item_type": "object",
      "nested_structure": {
        "issue": "str",
        "severity": {
          "type": "str",
          "options": ["info", "warning", "critical"]
        }
      }
    }
  },

  "image_required": false,
  "input_processors": [
    {
      "param_name": "contract_text",
      "input_processor": "document_content_extractor",
      "config": {
        "filename": "contract.pdf",
        "strategy": "hi_res"
      }
    }
  ]
}
```

**Key patterns:**
- `document_content_extractor` input processor for PDF/DOC files
- `param_name` matches the variable in `user_prompt`
- Complex nested structures for multi-part extractions

---

## Best Practices for Extraction Tasks

1. **Use temperature 0.0** for maximum accuracy
2. **Handle missing data** - instruct the model to use null, not invent values
3. **Include validation fields** - confidence scores, review flags
4. **Use document processors** for PDF/DOC inputs
5. **Enable vision** for image-based extraction
6. **Define precise formats** - date formats, number formats, etc.
