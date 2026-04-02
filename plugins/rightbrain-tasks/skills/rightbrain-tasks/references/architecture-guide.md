# Architecture Guide: Choosing the Right Primitive

## Core Distinction

**Tasks** are single-step LLM operations with structured pipelines. **TaskAgents** are reasoning orchestrators that dynamically decide which Tasks to invoke.

---

## Task Capabilities

A Task executes this pipeline:

```
Input → [Input Processors] → [RAG] → [LLM + MCP Tools] → Structured Output
```

Tasks excel at:
- **File processing** — audio transcription, PDF extraction, image analysis
- **Context retrieval** — RAG with vector search, query rewriting, source attribution
- **Deterministic output** — typed JSON schemas, images, audio, CSV, PDF
- **A/B testing** — weighted revision traffic splitting
- **Predictable cost** — fixed LLM cost per invocation, 2-10s typical execution

## TaskAgent Capabilities

TaskAgents add reasoning-based orchestration:
- **Multi-turn dialogue** — session memory across conversation turns
- **Dynamic execution** — LLM decides which tools to call and in what order
- **Streaming responses** — real-time SSE event stream
- **Tool composition** — combines Tasks, MCP servers, Skills, and Integrations
- **Parallel execution** — independent tools can run simultaneously in agentic mode

## Cost Model

| | Tasks | TaskAgents |
|---|---|---|
| **Cost structure** | Fixed per invocation | Accumulates across reasoning steps |
| **Predictability** | High — one LLM call per run | Variable — scales with conversation complexity |
| **Typical tokens** | 500-5,000 | 2,000-100,000+ |
| **When cost matters** | High-volume batch processing | Interactive or complex workflows |

---

## The Four Primitives Decision Matrix

| You need... | Use this | Why |
|---|---|---|
| A structured AI operation with typed output | **Task** | Single-step, versioned, testable, predictable cost |
| External API, database, or service access | **MCP Server** | Connect any service via Model Context Protocol |
| Domain expertise, brand voice, or procedures | **Skill** | Declarative instructions the LLM reads and follows |
| Google Workspace read/write (Sheets, Docs, etc.) | **Integration** | Platform-managed OAuth, zero infrastructure |
| Multi-step reasoning that combines the above | **TaskAgent** | Orchestrates all primitives with LLM decision-making |

### When to combine primitives

The most powerful agents combine multiple primitive types:

| Pattern | Primitives | Example |
|---|---|---|
| Read → Analyse → Write | Integration + Task + Integration | Read transcript from Docs, extract themes via Task, write to Sheets |
| Research → Format | MCP + Skill | Search Notion via MCP, format response using brand voice Skill |
| Classify → Route → Respond | Task + Task + Task | Classify intent, look up knowledge, draft response |
| Read → Analyse → Design → Log | Integration + Task + Skill + Integration | Full post-call intelligence hub |

---

## Selection Heuristic

### Use Tasks when:
- Single-path workflow (invoice extraction, transcription, RAG lookups, image generation)
- Need deterministic, typed output
- High volume / batch processing where cost predictability matters
- Want A/B testing via revision weights

### Use TaskAgents when:
- Multi-step reasoning (repo analysis → ecosystem research → report)
- Conditional routing (different paths based on input)
- Back-and-forth conversation (support chat, interactive assistance)
- Need to combine multiple Tasks, Skills, Integrations, or MCP servers
- Want streaming real-time responses

### Use Skills when:
- You have institutional knowledge to encode (brand voice, escalation procedures, design guidelines)
- Multiple agents need the same expertise
- You want to version and iterate on knowledge independently of agent config
- The knowledge shapes HOW the agent responds, not WHAT data it accesses

### Use Integrations when:
- You need Google Workspace access (Sheets, Docs, Slides, Calendar, Gmail)
- You want platform-managed OAuth (no infrastructure to maintain)
- You need the agent to read from or write to Google services at runtime

### Use MCP Servers when:
- You need to connect to external services not covered by Integrations
- You have a custom internal API, Notion workspace, GitHub, or other tool server
- You want the agent to call external tools directly

---

## Architecture Pattern

Optimal systems compose **focused Tasks** (each with dedicated RAG, processors, and tools) orchestrated by **lightweight TaskAgents** that use reasoning-optimized models for coordination.

```
TaskAgent (GPT-4o, concise instruction)
  ├── Task: classify-intent (GPT-4o-mini, low temp, JSON output)
  ├── Task: extract-entities (GPT-4o, structured output)
  ├── Task: draft-response (Claude Sonnet, text output)
  ├── Skill: brand-voice (markdown instructions)
  ├── Integration: Google Sheets (3 filtered tools)
  └── MCP: Notion (search + fetch tools)
```

### Key insight from testing

Agent instructions should be **concise and goal-oriented**. A one-sentence instruction like:

> "You are a content strategist. Read transcripts from Google Docs and push content ideas to Google Sheets."

...works as well as a 40-line instruction with explicit tool usage details. The LLM figures out tool mechanics from the tool descriptions. Focus the instruction on the **what** and **why**, not the **how**.

### Avoid in instructions

- **Curly braces** — `{DOCUMENT_ID}` in instructions gets treated as a session state variable by ADK. Use plain text descriptions instead.
- **Tool usage details** — Don't tell the agent which API parameters to pass. The tool schemas handle this.
- **Step-by-step tool sequences** — The LLM determines the optimal order. Only specify sequence when order genuinely matters.
