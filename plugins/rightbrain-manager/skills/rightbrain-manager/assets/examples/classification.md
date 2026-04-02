# Classification Task Examples

Classification tasks categorize inputs into predefined categories. They're characterized by:
- Low temperature (0.0-0.3) for consistency
- Enum-constrained outputs
- Optional confidence scores

---

## Example 1: Support Ticket Routing (Simple)

Classify customer support tickets for department routing.

```json
{
  "name": "Support Ticket Router",
  "description": "Classify incoming support tickets for automatic routing",
  "enabled": true,
  "output_modality": "json",

  "system_prompt": "You are a customer support triage specialist.\n\nYou quickly and accurately categorize support tickets to ensure they reach the right team. You prioritize customer experience and understand that misrouted tickets cause delays.\n\nWhen uncertain, you flag for human review rather than guess.",

  "user_prompt": "## Goal\nClassify this support ticket for routing to the appropriate team.\n\n## Ticket Content\n{ticket_subject}: Email subject line\n{ticket_body}: Full ticket content\n{customer_tier}: Customer subscription tier (free, pro, enterprise)\n\n## Departments\n- **billing**: Payment issues, invoices, refunds, subscription changes\n- **technical**: Bugs, errors, integration problems, performance issues\n- **account**: Login issues, password resets, profile updates, permissions\n- **sales**: Upgrade questions, feature inquiries, enterprise contracts\n- **success**: Onboarding help, best practices, training requests\n\n## Routing Rules\n- Enterprise customers with technical issues get priority flag\n- Tickets mentioning \"cancel\" go to billing regardless of other content\n- Feature requests from paying customers go to sales",

  "llm_model_id": "MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.0
  },

  "output_format": {
    "department": {
      "type": "str",
      "options": ["billing", "technical", "account", "sales", "success"]
    },
    "priority": {
      "type": "str",
      "options": ["low", "normal", "high", "urgent"]
    },
    "confidence": {
      "type": "number",
      "description": "Confidence in classification from 0.0 to 1.0"
    },
    "requires_human_review": "bool",
    "summary": {
      "type": "str",
      "description": "One-sentence summary of the issue"
    }
  },

  "image_required": false,
  "input_processors": []
}
```

**Key patterns:**
- Enum outputs for consistent categorization
- Confidence score for uncertainty handling
- Human review flag for edge cases

---

## Example 2: Lead Scoring (Complex Output Structure)

Evaluate B2B leads for sales-readiness with detailed breakdown.

```json
{
  "name": "Lead Scoring",
  "description": "Score and tier B2B leads based on company fit and engagement signals",
  "enabled": true,
  "output_modality": "json",

  "system_prompt": "You are a B2B sales analyst specializing in lead qualification for enterprise SaaS products.\n\nYour expertise includes:\n- Evaluating company fit based on firmographic data\n- Identifying buying signals and timing indicators\n- Assessing decision-maker engagement\n\nYou provide data-driven assessments with clear reasoning.",

  "user_prompt": "## Goal\nScore the sales-readiness of this lead based on available data.\n\n## Lead Information\n{company_name}: Company name\n{industry}: Primary industry\n{employee_count}: Number of employees\n{recent_activity}: Recent engagement, funding, or news\n\n## Scoring Criteria\n- **Company fit (40%)**: Size, industry, and tech stack alignment\n- **Timing signals (30%)**: Recent funding, hiring, or tech initiatives\n- **Engagement level (30%)**: Content consumption, demo requests, or direct outreach\n\n## Instructions\n1. Evaluate company fit against ideal customer profile\n2. Identify positive and negative timing signals\n3. Assess engagement quality and intent\n4. Calculate overall score and assign tier",

  "llm_model_id": "MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.2
  },

  "output_format": {
    "score": {
      "type": "int",
      "description": "Overall lead score from 0-100"
    },
    "tier": {
      "type": "str",
      "options": ["hot", "warm", "cold"]
    },
    "fit_score": "int",
    "timing_score": "int",
    "engagement_score": "int",
    "positive_signals": {
      "type": "list",
      "item_type": "str"
    },
    "concerns": {
      "type": "list",
      "item_type": "str"
    },
    "recommended_action": "str",
    "talking_points": {
      "type": "list",
      "item_type": "str"
    }
  },

  "image_required": false,
  "input_processors": []
}
```

**Key patterns:**
- Multi-dimensional scoring with component breakdown
- Lists for variable-length outputs (signals, concerns)
- Actionable recommendations alongside classification

---

## Best Practices for Classification Tasks

1. **Use low temperature** (0.0-0.3) for consistent results
2. **Define clear categories** with examples in the prompt
3. **Include confidence scores** to identify uncertain classifications
4. **Add human review flags** for edge cases
5. **Use enum constraints** (`options`) for categorical outputs
6. **Handle edge cases** explicitly in the prompt
