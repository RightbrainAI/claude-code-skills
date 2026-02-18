# Generation Task Examples

Generation tasks create content based on inputs. They're characterized by:
- Higher temperature (0.5-0.8) for creativity
- Often use `text` modality instead of `json`
- Focus on style, tone, and structure
- May include style guides or examples in prompts

---

## Example 1: Customer Support Email Response (JSON Output)

Generate professional support responses with structured metadata.

```json
{
  "name": "Support Email Generator",
  "description": "Generate helpful customer support email responses",
  "enabled": true,
  "output_modality": "json",

  "system_prompt": "You are a customer support specialist known for clear, empathetic communication.\n\nYou write responses that:\n- Acknowledge the customer's frustration or situation\n- Provide clear, actionable solutions\n- Maintain a warm but professional tone\n- Are concise yet complete\n\nYou never blame the customer or make excuses. You focus on solutions.",

  "user_prompt": "## Goal\nGenerate a professional support email response.\n\n## Context\n{customer_name}: Customer's name\n{issue_summary}: Summary of the customer's issue\n{resolution}: How the issue was resolved or next steps\n{customer_tier}: Customer tier (free, pro, enterprise)\n\n## Response Guidelines\n- Start with a personalized greeting\n- Acknowledge their specific issue\n- Explain the resolution clearly\n- Provide next steps if any\n- Offer additional help\n- Sign off professionally\n\n## Tone Adjustments\n- Enterprise customers: More formal, emphasize priority handling\n- Free tier: Friendly but efficient",

  "llm_model_id": "MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.6
  },

  "output_format": {
    "subject_line": "str",
    "email_body": "str",
    "internal_notes": "str",
    "suggested_follow_up_days": "int",
    "escalation_recommended": "bool"
  },

  "image_required": false,
  "input_processors": []
}
```

**Example output:**
```json
{
  "subject_line": "Re: Your billing question - Resolved",
  "email_body": "Hi Sarah,\n\nThank you for reaching out about the duplicate charge on your account.\n\nI've reviewed your billing history and can confirm that the second charge was processed in error. I've initiated a full refund of $49.99, which should appear on your statement within 3-5 business days.\n\nIs there anything else I can help you with?\n\nBest regards,\nSupport Team",
  "internal_notes": "Duplicate charge confirmed. Refund processed via Stripe.",
  "suggested_follow_up_days": 5,
  "escalation_recommended": false
}
```

**Key patterns:**
- JSON output for structured responses with metadata
- Moderate temperature (0.6) for natural but consistent tone
- Separate internal notes from customer-facing content

---

## Example 2: Blog Post Generator (Text Output)

Generate SEO-optimized blog content as free-form text.

```json
{
  "name": "Blog Post Writer",
  "description": "Generate engaging, SEO-friendly blog posts",
  "enabled": true,
  "output_modality": "text",
  "output_format": null,

  "system_prompt": "You are a content marketing specialist who writes engaging, informative blog posts.\n\nYour writing style:\n- Clear and accessible to a general audience\n- Uses short paragraphs and bullet points for readability\n- Includes practical examples and actionable advice\n- Naturally incorporates keywords without stuffing\n- Engages readers with questions and direct address\n\nYou write in a conversational but authoritative tone.",

  "user_prompt": "## Goal\nWrite a blog post on the given topic.\n\n## Topic\n{topic}: The main subject of the blog post\n{target_keywords}: Keywords to naturally incorporate\n{word_count}: Target word count\n{audience}: Target reader persona\n{tone}: Desired tone (professional, casual, technical, etc.)\n\n## Structure Requirements\n1. Compelling headline with primary keyword\n2. Hook introduction (2-3 sentences)\n3. 3-5 main sections with H2 headings\n4. Practical examples or tips in each section\n5. Conclusion with call-to-action\n\n## SEO Guidelines\n- Include primary keyword in first 100 words\n- Use variations of keywords naturally\n- Write meta description (150-160 characters) at the end",

  "llm_model_id": "MODEL_UUID_HERE",
  "llm_config": {
    "temperature": 0.7
  },

  "image_required": false,
  "input_processors": []
}
```

**Key patterns:**
- `output_modality: "text"` for long-form content
- `output_format: null` required for text modality
- Higher temperature (0.7) for more creative, varied output
- Detailed structure guidance in prompt

---

## Best Practices for Generation Tasks

1. **Adjust temperature** based on creativity needs (0.5-0.8 typical)
2. **Provide style guidance** in the system prompt
3. **Set clear constraints** - length, tone, structure
4. **Use `text` modality** for long-form content (no JSON structure)
5. **Use `json` modality** when you need metadata alongside content
6. **Include examples** of good output when possible
7. **Test variations** - run the same input multiple times to check consistency
