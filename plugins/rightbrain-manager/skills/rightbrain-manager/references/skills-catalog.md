# Skills Catalog Reference

Detailed workflows for browsing, creating, and managing Rightbrain Skills.

> **Prerequisites:** Authentication and context (API_BASE, ACCESS_TOKEN, ORG_ID, PROJECT_ID) must be established before using these workflows. See SKILL.md Phase 0.

---

## Overview

Skills are declarative instruction bundles that teach an agent how to handle specific domains or tasks. They are NOT executable tools -- they provide knowledge and procedures that the agent follows.

**Key concepts:**
- **Progressive disclosure**: `list_skills` shows descriptions; `load_skill` loads full instructions. This keeps the agent's context lean until a skill is actually needed.
- **Revisions are immutable**: Each edit creates a new revision. The active revision is a mutable pointer.
- **Skills only work in agentic mode**: Sequential mode agents cannot use skills.

---

## API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Browse global catalog | GET | `/skills` |
| Available skills (global + project) | GET | `/skill/available` |
| Create project skill | POST | `/org/{org_id}/project/{project_id}/skill` |
| List project skills | GET | `/org/{org_id}/project/{project_id}/skill` |
| Get skill | GET | `/org/{org_id}/project/{project_id}/skill/{skill_id}` |
| Update skill | POST | `/org/{org_id}/project/{project_id}/skill/{skill_id}` |
| Delete skill | DELETE | `/org/{org_id}/project/{project_id}/skill/{skill_id}` |
| Create revision | POST | `/org/{org_id}/project/{project_id}/skill/{skill_id}/revision` |
| Activate revision | POST | `/org/{org_id}/project/{project_id}/skill/{skill_id}/revision/{rev_id}/activate` |
| List revisions | GET | `/org/{org_id}/project/{project_id}/skill/{skill_id}/revision` |

---

## Browse Available Skills

### Global Catalog

Browse skills available to all Rightbrain users:

```bash
curl -s -X GET "{API_BASE}/skills" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

Filter by source, tag, or search:

```bash
# Filter by source
curl -s -X GET "{API_BASE}/skills?source=rightbrain" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"

# Search by keyword
curl -s -X GET "{API_BASE}/skills?search=customer+support" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

### Combined Available Skills

Get both global and project-specific skills in one call:

```bash
curl -s -X GET "{API_BASE}/skill/available" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

---

## Create a Project Skill

### Step 1: Design the Skill

Gather the required information:
- **slug**: Kebab-case identifier (e.g., `customer-onboarding`)
- **display_name**: Human-readable name (e.g., "Customer Onboarding")
- **description**: When to use this skill -- this is what the agent sees when deciding whether to load it
- **instructions**: Full markdown instructions the agent follows when the skill is loaded
- **reference_docs**: Optional additional documents (map of filename to content)
- **skill_metadata**: Optional metadata (e.g., author, version notes)

### Step 2: Write Effective Instructions

Skill instructions should be:
- **Actionable**: Tell the agent what to DO, not just what to know
- **Structured**: Use markdown headers, numbered steps, decision trees
- **Self-contained**: Include all context needed -- the agent has no other reference

**Good structure:**
```markdown
## When to Use
[Conditions that trigger this skill]

## Steps
1. [First action]
2. [Decision point]
   - If A: [action]
   - If B: [action]
3. [Final action]

## Constraints
- [What NOT to do]
- [Boundaries]

## Examples
[Concrete examples of correct behavior]
```

### Step 3: Deploy

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "slug": "customer-onboarding",
  "display_name": "Customer Onboarding",
  "description": "Guides agents through the customer onboarding process including account setup, welcome messaging, and initial configuration",
  "instructions": "## When to Use\nUse this skill when a new customer needs to be onboarded.\n\n## Steps\n1. Verify the customer's account details\n2. Send a personalized welcome message\n3. Configure default settings based on their plan tier\n\n## Constraints\n- Never skip the verification step\n- Always confirm with the customer before making changes",
  "reference_docs": {
    "plan-tiers.md": "# Plan Tiers\n\n## Starter\n- 100 tasks/month\n- 1 project\n\n## Pro\n- 10,000 tasks/month\n- Unlimited projects"
  },
  "skill_metadata": {
    "author": "customer-success-team"
  }
}
EOF
```

The response includes the skill `id` and an initial revision that is automatically activated.

---

## Revision Workflow

Skills use an immutable revision system. Each change creates a new revision; the active revision determines what agents see.

### Create a New Revision

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill/{SKILL_ID}/revision" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << 'EOF'
{
  "instructions": "## Updated Instructions\n...",
  "reference_docs": {"updated-doc.md": "new content"},
  "annotation": "Added error handling guidance"
}
EOF
```

The new revision is created but NOT active yet.

### Activate a Revision

```bash
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill/{SKILL_ID}/revision/{REVISION_ID}/activate" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

This makes the specified revision the active one. All agents using `skill_revision_id: null` (follow latest) will immediately pick up the change.

### Revert to a Previous Revision

Simply activate the older revision ID. Revisions are never deleted.

```bash
# List revisions to find the one you want
curl -s -X GET "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill/{SKILL_ID}/revision" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"

# Activate the desired revision
curl -s -X POST "{API_BASE}/org/{ORG_ID}/project/{PROJECT_ID}/skill/{SKILL_ID}/revision/{OLD_REVISION_ID}/activate" \
  -H "Authorization: Bearer {ACCESS_TOKEN}" \
  -H "Content-Type: application/json"
```

---

## Attach Skills to Agents

When creating or updating an agent, include the skills array:

```json
"skills": [
  {"skill_id": "skill-uuid-1", "skill_revision_id": null},
  {"skill_id": "skill-uuid-2", "skill_revision_id": "specific-revision-uuid"}
]
```

**Version pinning:**
- `skill_revision_id: null` -- Follow the latest active revision. The agent always gets the most current version. Best for skills under active development.
- `skill_revision_id: "uuid"` -- Pin to a specific revision. The agent always gets this exact version regardless of what is active. Best for production stability.

**Update semantics** (same as all agent array fields):
- Omit the field = no change
- Empty list `[]` = remove all skills
- Non-empty list = full replacement

---

## Skill Design Best Practices

1. **Description is critical** -- The agent uses the description to decide whether to load the skill. Make it specific: "Handles GDPR compliance checks for EU customer data" is better than "Compliance skill."

2. **Keep instructions focused** -- One skill = one domain. If your instructions cover multiple unrelated topics, split into separate skills.

3. **Use reference_docs for static content** -- Put lookup tables, policy documents, and reference data in `reference_docs` rather than inlining them in instructions. This keeps instructions readable.

4. **Test with the agent** -- Run the agent with the skill attached and verify it loads and follows the instructions correctly. Check run events to see `skills_activated`.

5. **Version thoughtfully** -- Use annotations on revisions to track what changed. Pin production agents to specific revisions; let development agents follow latest.

---

## Common Patterns

### Policy Enforcement Skill
Instructions define rules and guardrails. The agent checks its actions against the skill before responding.

### Domain Expert Skill
Instructions contain specialized knowledge (medical terminology, legal frameworks, industry jargon). The agent loads this when the conversation enters that domain.

### Workflow Skill
Instructions define a multi-step process with decision points. The agent follows the steps in order, making decisions at each branch.

### Style Guide Skill
Instructions define tone, formatting, and terminology standards. The agent applies these to all its outputs when the skill is loaded.
