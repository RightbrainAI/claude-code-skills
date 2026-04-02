# Image Generation Tasks

This guide covers creating Rightbrain tasks that generate images. Image generation requires specific model selection, prompt patterns, and configuration to produce quality results.

---

## Task Configuration

Image generation tasks use `output_modality: "image"` and require a model with image output capabilities.

```json
{
  "name": "My Image Task",
  "description": "Generates images from text input",
  "enabled": true,
  "output_modality": "image",
  "output_format": null,

  "system_prompt": "...",
  "user_prompt": "...",

  "llm_model_id": "MODEL_UUID_WITH_IMAGE_OUTPUT",
  "llm_config": {
    "temperature": 0.7
  },

  "image_required": false,
  "input_processors": []
}
```

**Key points:**
- `output_modality` must be `"image"`
- `output_format` must be `null` (not applicable for image output)
- Model must have `supports_image_output: true`
- Temperature 0.6-0.8 works well for creative image generation

---

## Model Selection

When fetching models, filter for those with `supports_image_output: true`:

| Model | Best For | Notes |
|-------|----------|-------|
| **Gemini 2.5 Flash Image** | Fast, efficient generation | Good balance of speed and quality |
| **Gemini 3 Pro Image** | Higher fidelity output | Slower but more detailed |
| **GPT Image 1.5** | Better instruction following | Best for complex prompts |

**Finding image models:**
```bash
curl -s "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/model" \
  -H "Authorization: Bearer {API_KEY}" | \
  jq '.[] | select(.supports_image_output == true) | {id, name, alias}'
```

---

## Preventing Text in Generated Images

Image generation models often add unwanted text, labels, or annotations. This is a known challenge because models learn from images that frequently contain text.

### The Problem

Simple instructions like "no text" or "without words" often fail because:
- Models prioritize positive descriptions over negatives
- "Text" is a broad concept the model may interpret differently
- Training data contains many images with text

### Techniques That Work

#### 1. Identity Framing (Most Effective)

Frame the generation as if created by a specific type of professional who wouldn't include text:

```
"You are a fine art photographer... think like a photographer, not a graphic designer"
"You are creating museum-quality abstract art..."
"You are an editorial photographer for a luxury magazine..."
```

**Why it works:** The persona establishes implicit constraints about what that professional would or wouldn't include.

#### 2. Explicit Negatives with Synonyms

Use comprehensive lists covering all text-related concepts:

```
CRITICAL RULES:
- NEVER include any text, letters, words, numbers, labels, or typography
- NEVER include captions, annotations, watermarks, or signatures
- Create purely visual, abstract representations
```

**Why it works:** Covers multiple interpretations of "text" to close loopholes.

#### 3. Style Anti-Patterns

Explicitly exclude categories of content that typically contain text:

```
## Style Reference
Think: abstract 3D art, cinema stills, editorial photography.
NOT: infographics, presentations, educational diagrams, marketing materials,
     annotated photos, labeled charts, user interfaces.
```

**Why it works:** Eliminates entire genres that inherently include text.

#### 4. Positive Redirection

Focus on describing what you want in rich detail. The more specific your positive description, the less room for unwanted elements:

```
"Professional product photograph, clean studio backdrop, soft diffuse lighting,
commercial photography style, minimalist composition, single focal point,
gallery exhibition quality"
```

**Why it works:** Detailed positive prompts naturally exclude unwanted elements.

---

## Prompt Structure for Image Tasks

### System Prompt Template

```
You are a [visual role] creating [style] images for [purpose].

Your aesthetic:
- [Visual principle 1 - composition]
- [Visual principle 2 - color/lighting]
- [Visual principle 3 - mood/atmosphere]

CRITICAL RULES:
- NEVER include any text, letters, words, numbers, labels, or typography
- NEVER include logos, watermarks, or UI elements
- Create purely visual, abstract representations
- Think like a [persona], not a graphic designer
```

### User Prompt Template

```
## Task
Create [type of visual] that captures [goal/essence].

## Content to Visualize
{input_variable}

## Visual Direction
Extract [what to focus on] and represent through:
- [Visual element 1]
- [Visual element 2]
- [Mood/atmosphere]

## Strict Requirements
- NO TEXT of any kind - no letters, words, labels, or numbers anywhere
- [Composition requirement]
- [Color/style requirement]

## Style Reference
Think: [positive examples with specific styles].
NOT: [negative examples to avoid].
```

---

## Complete Example: LinkedIn Post to Image

This task generates abstract visuals to accompany LinkedIn posts:

```json
{
  "name": "LinkedIn Post to Image",
  "description": "Generates a visually compelling abstract image to accompany a LinkedIn post",
  "enabled": true,
  "output_modality": "image",
  "output_format": null,

  "system_prompt": "You are a visual designer creating abstract, editorial-style images for professional social media.\n\nYour aesthetic:\n- Minimalist 3D renders with dramatic lighting\n- Abstract geometric forms and shapes\n- Rich, moody color palettes (deep blues, charcoal, copper accents)\n- Clean negative space\n- Editorial magazine quality\n\nCRITICAL RULES:\n- NEVER include any text, letters, words, numbers, labels, or typography\n- NEVER include logos, watermarks, or UI elements\n- Create purely visual, abstract representations\n- Think like a fine art photographer, not a graphic designer",

  "user_prompt": "## Task\nCreate an abstract visual that captures the emotional essence of this content.\n\n## Content to Visualize\n{linkedin_post}\n\n## Visual Direction\nExtract ONE core metaphor or feeling from the content and represent it through:\n- Abstract 3D geometric shapes\n- Dramatic lighting and shadows\n- Symbolic objects (no literal interpretations)\n- Professional, moody atmosphere\n\n## Strict Requirements\n- NO TEXT of any kind - no letters, words, labels, or numbers anywhere in the image\n- NO logos or branding\n- Purely abstract and symbolic\n- Square 1:1 aspect ratio\n- Dark, sophisticated color palette\n- High contrast for social media feeds\n\n## Style Reference\nThink: abstract 3D art, cinema stills, editorial photography, conceptual illustration.\nNOT: infographics, presentations, stock photos, marketing materials.",

  "llm_model_id": "MODEL_UUID_WITH_IMAGE_OUTPUT",
  "llm_config": {
    "temperature": 0.8
  },

  "image_required": false,
  "input_processors": []
}
```

**Key design decisions:**
- Higher temperature (0.8) for creative variation
- Multiple layers of text prevention (identity framing, explicit negatives, anti-patterns)
- Specific style references to guide the aesthetic
- Square aspect ratio optimized for LinkedIn

---

## Running Image Tasks

Image tasks return a download URL instead of text content:

```json
{
  "task_id": "...",
  "task_revision_id": "...",
  "response": {
    "image_name": "uuid.png",
    "content_type": "image/png",
    "download_url": "/org/{org_id}/project/{project_id}/task/{task_id}/run/{run_id}/file/{filename}"
  }
}
```

**Downloading the generated image:**
```bash
curl -s "{API_BASE}{download_url}" \
  -H "Authorization: Bearer {API_KEY}" \
  -o output.png
```

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Text appears in image | Insufficient negative prompts | Add identity framing + explicit negatives + anti-patterns |
| Wrong style/aesthetic | Vague style description | Add specific style references and negative examples |
| Low quality output | Wrong model or temperature | Use higher-fidelity model, adjust temperature |
| Task fails | Model doesn't support image output | Filter for `supports_image_output: true` |
| Literal interpretations | Prompt too descriptive | Focus on metaphors and abstract concepts |

---

## Best Practices Summary

1. **Model selection:** Always verify `supports_image_output: true`
2. **Text prevention:** Layer multiple techniques (identity + negatives + anti-patterns)
3. **Temperature:** Use 0.6-0.8 for creative variation
4. **Prompts:** Be specific about style, vague about subject interpretation
5. **Testing:** Generate multiple images to verify consistency
6. **Iteration:** If text appears, strengthen the anti-text language
