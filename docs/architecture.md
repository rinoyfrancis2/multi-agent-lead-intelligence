# Architecture — Multi-Agent Lead Intelligence System

## System Overview

A multi-agent system built in n8n that automates the full lead intelligence pipeline: research, scoring, personalised outreach, and CRM enrichment. Five specialist agents coordinate through a shared Postgres state table, with human-in-the-loop approval before any CRM write.

---

## High-Level Flow

```
Lead arrives (Webhook / Form / CRM trigger)
        |
        v
 ┌─────────────────────┐
 │  Orchestrator Agent  │  (GPT-4o)
 │  Reads lead data,    │
 │  delegates in order  │
 └────────┬────────────┘
          |
          v
 ┌─────────────────────┐
 │   Research Agent     │  (Gemini Pro + SerpAPI)
 │   Scrapes company,   │
 │   LinkedIn, news     │
 └────────┬────────────┘
          | writes research_output → Postgres
          v
 ┌─────────────────────┐      ┌──────────────────────┐
 │   Scoring Agent      │ ←── │  RAG Knowledge Base   │
 │   Scores lead 1-10   │     │  (Supabase pgvector)  │
 │   against ICP        │     │  ICP criteria, product │
 └────────┬────────────┘      │  info, competitors     │
          | writes score → Postgres    └──────────────────────┘
          v
 ┌─────────────────────┐
 │ Personalisation Agent│  (GPT-4o)
 │ Writes tailored      │
 │ outreach email       │
 └────────┬────────────┘
          | writes outreach_draft → Postgres
          v
 ┌─────────────────────┐
 │   HITL Approval      │
 │   Wait node + Email  │
 │   Human reviews draft│
 └────────┬────────────┘
          | approved
          v
 ┌─────────────────────┐
 │    CRM Agent         │  (GPT-4o + API)
 │    Writes enriched   │
 │    data to HubSpot   │
 └─────────────────────┘
```

---

## Agent Detail

### 1. Orchestrator Agent
- **LLM**: GPT-4o
- **n8n node**: AI Agent (main workflow entry point)
- **Trigger**: Webhook / Form submission / CRM event
- **Role**: Receives raw lead data, creates a session in Postgres, then calls each specialist agent in sequence via AI Agent Tool nodes
- **System prompt**: Defines the exact order and conditions for calling each sub-agent
- **Key decision**: The Orchestrator does NOT make business decisions — it delegates and routes

### 2. Research Agent
- **LLM**: OpenAI GPT-4o-mini (switched from Gemini Pro — free tier quota exhausted)
- **Tools**: Tavily (web search) — switched from SerpAPI (better free tier). HTTP Request tool removed — caused context window overflow with raw HTML.
- **Input**: Lead name, company name, domain from Postgres `lead_data`
- **Output**: Structured research summary written to Postgres `research_output`
- **What it researches**:
  - Company overview — what they do, size, industry
  - Recent news — funding rounds, product launches, leadership changes
  - AI automation fit signals
- **Note**: Currently uses INSERT (not UPDATE) — will switch to UPDATE when Orchestrator is built. Orchestrator will INSERT the initial row, Research Agent will UPDATE it.

### 3. Scoring Agent
- **LLM**: Claude Sonnet (Anthropic)
- **Tools**: Postgres query tool (queries `p01_documents` for ICP criteria — local pgvector, not Supabase)
- **Input**: `research_output` from Postgres + ICP criteria from RAG
- **Output**: Score (1-10) + reasoning written to Postgres `score` + `score_reasoning`
- **Code node**: Parses agent output to extract integer score and reasoning text separately
- **Max iterations**: Set to 25 (default 10 was too low)
- **Scoring criteria** (retrieved via RAG):
  - Company size match
  - Industry alignment
  - Technology stack fit
  - Budget indicators
  - Timing signals (hiring, funding, expansion)
- **Why Claude**: Strong reasoning and structured output — scoring requires careful multi-criteria analysis
- **Why RAG**: ICP criteria can change without touching the workflow — just update the knowledge base documents

### 4. Personalisation Agent
- **LLM**: GPT-4o
- **Input**: `research_output` + `score` + score reasoning from Postgres
- **Output**: Personalised cold outreach email written to Postgres `outreach_draft`
- **Personalisation signals used**:
  - Company-specific pain points from research
  - Industry-relevant value propositions
  - Recent company events as conversation hooks
  - Score-adjusted tone (high score = more direct ask, low score = softer nurture)
- **Why GPT-4o**: Strong natural language generation for persuasive, human-sounding copy

### 5. CRM Agent
- **LLM**: GPT-4o
- **Tools**: HTTP Request (HubSpot / Airtable API)
- **Input**: All fields from Postgres session
- **Output**: Created/updated CRM contact with enriched data
- **Fields written to CRM**:
  - Original lead data
  - Research summary
  - Lead score + reasoning
  - Approved outreach draft
  - Session timestamp and status
- **Only runs after HITL approval**

---

## Communication Pattern — Postgres Shared State

Agents do NOT talk to each other directly. All inter-agent communication flows through a shared Postgres table.

```
┌──────────────────────────────────────────────────────────────┐
│                    lead_sessions table                        │
├──────────┬──────────────┬───────┬────────────────┬──────────┤
│ lead_data│research_output│ score │ outreach_draft  │  status  │
├──────────┼──────────────┼───────┼────────────────┼──────────┤
│ Orch     │ Research     │Scoring│ Personalisation │ CRM      │
│ writes → │ writes →     │writes→│ writes →        │ reads all│
└──────────┴──────────────┴───────┴────────────────┴──────────┘
```

**Why this pattern:**
- Each agent can be tested independently — just populate its input column
- Full audit trail — every step is persisted
- Debugging is trivial — check which column is empty to find the failing agent
- Supports parallel execution in future (Research + another agent could run simultaneously)

### Table Schema

```sql
CREATE TABLE p01_lead_sessions (
    id SERIAL PRIMARY KEY,
    lead_data JSONB,           -- Raw lead info (name, email, company, source)
    research_output TEXT,       -- Research Agent's structured summary
    score INTEGER,             -- Scoring Agent's 1-10 score (integer, parsed by Code node)
    score_reasoning TEXT,      -- Why this score (full text from Claude)
    outreach_draft TEXT,       -- Personalisation Agent's email draft
    status VARCHAR(50) DEFAULT 'pending',  -- pending → research_done → scoring_done → personalisation_done → approved/rejected
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```
**Note**: Table renamed to `p01_lead_sessions` for multi-project naming consistency.

**Status progression**: `pending` → `researching` → `scored` → `drafted` → `awaiting_approval` → `approved` → `completed`

---

## RAG Knowledge Base (Supabase pgvector)

### Purpose
Gives the Scoring Agent business context that can't be hardcoded into prompts. When ICP criteria change, you update documents — not workflows.

### Documents to Embed
| Document | Content | Used By |
|---|---|---|
| ICP Criteria | Company size ranges, target industries, tech stack requirements, budget thresholds | Scoring Agent |
| Product/Service Info | What we offer, pricing tiers, key differentiators | Personalisation Agent |
| Competitor Notes | Who else is in this space, how we differ | Personalisation Agent |
| Past Outreach Examples | Successful cold emails that got replies | Personalisation Agent |

### Embedding Setup
- **Model**: OpenAI `text-embedding-3-small`
- **Storage**: Supabase pgvector
- **Retrieval**: n8n Supabase Vector Store node with similarity search
- **Chunk size**: ~500 tokens per chunk (standard for this type of content)

---

## Human-in-the-Loop (HITL) Approval Gate

### Why HITL Here
Fully automated outreach is risky — one bad email damages the brand. The HITL gate sits after the Personalisation Agent and before the CRM Agent.

### Flow
1. Personalisation Agent writes draft to Postgres
2. n8n sends approval email to Rinoy with the draft + lead summary + score
3. n8n Wait node pauses the workflow
4. Rinoy clicks Approve or Reject link in email
5. **Approve** → CRM Agent runs, lead status → `completed`
6. **Reject** → Status → `rejected`, no CRM write

### n8n Implementation
- **Wait node**: Set to "On webhook call" — resumes when approval link is clicked
- **Send Email node**: Sends formatted approval email with lead context
- **IF node**: After Wait, routes to CRM Agent (approved) or end (rejected)

---

## Data Flow Diagram

```
                    ┌─────────┐
                    │  Lead   │
                    │  Input  │
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │Orchestr.│
                    │  Agent  │
                    └────┬────┘
                         │ creates session
                    ┌────▼────────────────────────────────┐
                    │         POSTGRES lead_sessions       │
                    │                                      │
    ┌───────────────│  lead_data: {...}                    │
    │               │  research_output: null               │
    │               │  score: null                         │
    │               │  outreach_draft: null                │
    │               │  status: "pending"                   │
    │               └─────────────────────────────────────┘
    │                        │
    │  ┌─────────────────────▼──────────┐
    │  │        Research Agent           │
    │  │  reads: lead_data              │
    │  │  tools: SerpAPI, HTTP scrape   │
    │  │  writes: research_output       │
    │  └─────────────────────┬──────────┘
    │                        │
    │  ┌─────────────────────▼──────────┐    ┌──────────────┐
    │  │        Scoring Agent            │◄───│ RAG: pgvector│
    │  │  reads: research_output        │    │ ICP criteria  │
    │  │  writes: score, score_reasoning│    └──────────────┘
    │  └─────────────────────┬──────────┘
    │                        │
    │  ┌─────────────────────▼──────────┐
    │  │    Personalisation Agent        │
    │  │  reads: research + score       │
    │  │  writes: outreach_draft        │
    │  └─────────────────────┬──────────┘
    │                        │
    │               ┌────────▼────────┐
    │               │  HITL APPROVAL  │
    │               │  Email → Wait   │
    │               └───┬─────────┬───┘
    │              Reject│        │Approve
    │                   ▼         ▼
    │              [END]    ┌──────────┐
    │                       │CRM Agent │
    │                       │→ HubSpot │
    │                       └──────────┘
    │
    └── All agents read/write via this same Postgres table
```

---

## Technology Decisions

| Decision | Choice | Reason |
|---|---|---|
| Orchestration | n8n (self-hosted) | Already running on Docker, visual debugging, credential management built in |
| Agent communication | Postgres table, not direct | Testability, audit trail, independent agent testing |
| Multi-LLM | Different LLM per agent | Each agent's task suits a different model's strength |
| RAG storage | Local Postgres pgvector (`p01_documents`) | Already running in Docker, avoided Supabase dependency |
| CRM | Airtable (pending setup) | Lightweight demo CRM, free tier sufficient |
| HITL method | Gmail approval email | Simple, works on mobile, no extra UI needed |
| Web search | Tavily | Better free tier than SerpAPI |
| Research Agent LLM | GPT-4o-mini (was Gemini Pro) | Gemini free tier quota exhausted during build |
| HTTP scraping tool | Removed | Raw HTML caused context window overflow in Research Agent |
| Score parsing | Code node (regex) | Agent returns text, Code node extracts integer score + reasoning separately |

---

## Portfolio Value

This project demonstrates:
- **Multi-agent orchestration** — 5 agents with clear separation of concerns
- **Shared state pattern** — Postgres as inter-agent communication bus
- **RAG integration** — vector search providing business context to agents
- **Multi-LLM routing** — right model for the right task (Gemini for research, Claude for reasoning, GPT-4o for generation)
- **Human-in-the-loop** — production-grade approval gate
- **Real business value** — every sales team needs lead intelligence automation

---

## Diagram Reference

Visual architecture diagram: [architecture.excalidraw](architecture.excalidraw)

Open in Excalidraw (https://excalidraw.com) or the VS Code Excalidraw extension.
