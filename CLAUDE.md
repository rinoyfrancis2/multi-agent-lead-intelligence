# CLAUDE.md — Project 01: Multi-Agent Lead Intelligence System

## Project Overview

A multi-agent n8n system that automates lead research, scoring, personalised outreach, and CRM updates end-to-end. A lead arrives (from a form, webhook, or CRM trigger) and a team of 4 specialist agents autonomously research the company, score the opportunity against your ICP, write a personalised outreach message, and push everything into your CRM — with zero manual work.

**Why it's impressive:**
- Real multi-agent orchestration — not just one big agent
- Directly solves a problem every sales team has
- Combines RAG + Postgres communication pattern
- Human-in-the-loop approval before CRM write

---

## Agent Architecture

| # | Agent | Responsibility | LLM / Tool |
|---|---|---|---|
| 1 | Orchestrator Agent | Receives lead, reads context, delegates to specialists in sequence | GPT-4o |
| 2 | Research Agent | Scrapes company website, LinkedIn, recent news for context | Gemini Pro + SerpAPI |
| 3 | Scoring Agent | Scores lead 1–10 against ICP criteria from RAG knowledge base | Claude Sonnet |
| 4 | Personalisation Agent | Writes tailored outreach using research + score context | GPT-4o |
| 5 | CRM Agent | Writes enriched lead data + message to HubSpot / Airtable | GPT-4o + API |

---

## Shared Architecture Pattern

All 4 portfolio projects share this core pattern:

```
Orchestrator Agent (main n8n workflow)
    ├── Specialist Agent 1 (Agent Tool node)
    ├── Specialist Agent 2 (Agent Tool node)
    ├── Specialist Agent 3 (Agent Tool node)
    └── Specialist Agent 4 (Agent Tool node)

Communication: Postgres shared state table
Context: RAG via Supabase pgvector
LLMs: OpenAI GPT-4o, Claude Sonnet, Gemini Pro (swappable per agent)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n | Primary orchestration platform (self-hosted on Docker) |
| OpenAI GPT-4o | Orchestrator, Personalisation, and CRM agents |
| Claude Sonnet | Scoring Agent |
| Gemini Pro | Research Agent |
| SerpAPI / Tavily | Web search for lead research |
| PostgreSQL | Shared state layer — agents write outputs, next agent reads |
| Supabase pgvector | RAG knowledge base (ICP criteria, product info, competitor notes) |
| OpenAI Embeddings | text-embedding-3-small for RAG |
| HubSpot / Airtable API | CRM destination |
| n8n Webhook Trigger | Lead intake trigger |

---

## Postgres Shared State Table

```sql
CREATE TABLE lead_sessions (
    id SERIAL PRIMARY KEY,
    lead_data JSONB,
    research_output TEXT,
    score INTEGER,
    outreach_draft TEXT,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Step-by-Step Build Guide

### Step 01 — Set up Postgres shared state table
Create the `lead_sessions` table above. This is where agents communicate — each agent reads the previous agent's output and writes its own.

### Step 02 — Build your RAG knowledge base
Load your ICP criteria, product info, and competitor notes into Supabase pgvector. Use OpenAI embeddings (`text-embedding-3-small`).

### Step 03 — Create the Research Agent workflow
n8n workflow: Webhook trigger → AI Agent node (Gemini Pro) with SerpAPI tool + web scrape tool → Write output to Postgres `lead_sessions.research_output`.

### Step 04 — Create the Scoring Agent workflow
AI Agent node (Claude Sonnet) reads research from Postgres, queries RAG for ICP criteria, returns score 1–10 with reasoning. Writes to `lead_sessions.score`.

### Step 05 — Create the Personalisation Agent workflow
AI Agent node (GPT-4o) reads research + score from Postgres. System prompt: write a personalised cold email. Writes draft to `lead_sessions.outreach_draft`.

### Step 06 — Create the CRM Agent workflow
Reads all outputs from Postgres, formats payload, calls HubSpot/Airtable API to create/update contact with enriched data.

### Step 07 — Build the Orchestrator Agent
Main workflow with Chat/Webhook trigger. Add each specialist as an AI Agent Tool node. System prompt defines when to call each agent.

### Step 08 — Add human-in-the-loop approval
After Personalisation Agent, add a Wait node + email approval step. Only CRM Agent runs after human approves the outreach draft.

### Step 09 — Test end to end
Send a test lead via webhook. Check each Postgres column gets populated in sequence. Verify CRM receives the final enriched record.

---

## Key Implementation Notes

- **Orchestrator system prompt is critical**: Describe WHEN to use each agent tool clearly. Example: "Use Research Agent first for any new lead. Then always run Scoring Agent. Only run CRM Agent after human approval." The LLM follows these instructions to route correctly.
- **Each specialist is an Agent Tool node** in n8n — not a separate workflow. They are sub-agents connected to the Orchestrator.
- **Postgres is the communication bus** — agents don't talk to each other directly. Agent A writes to Postgres, Agent B reads from Postgres.
- **RAG gives business context** — without it, the Scoring Agent has no criteria to score against.

---

## n8n Node Types Used

| Node | Purpose |
|---|---|
| Webhook Trigger | Lead intake |
| AI Agent | Orchestrator + each specialist agent |
| AI Agent Tool | Connecting specialists to Orchestrator |
| Postgres | Read/write shared state |
| Supabase Vector Store | RAG retrieval |
| HTTP Request | SerpAPI calls, CRM API calls |
| Wait | Human-in-the-loop pause |
| Send Email | Approval notification |

---

## File Conventions

| File type | Naming convention |
|---|---|
| n8n workflow exports | `workflow_[purpose].json` (e.g., `workflow_orchestrator.json`, `workflow_research_agent.json`) |
| Documentation | `docs/[topic].md` |
| Architecture diagrams | `docs/architecture_[name].png` |

---

## RAG Knowledge Base Contents (to prepare)

- Ideal Customer Profile (ICP) criteria document
- Product/service descriptions
- Competitor analysis notes
- Past successful outreach examples (for Personalisation Agent context)

---

## Parent Project Reference

Full career context, CV details, and overall portfolio strategy:
`/Users/rinoyfrancis/Rinoy Cluade Lab/LInkedin mady mentor/CLAUDE.md`
