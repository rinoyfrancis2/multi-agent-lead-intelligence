# Build Guide — Multi-Agent Lead Intelligence System

Step-by-step manual build instructions for n8n. Complete each step fully before moving to the next. Update `progress.md` after each step.

---

## Step 1 — Set Up Postgres Shared State Table

**Where:** Local Docker Postgres (`n8n-postgres-1` container)

Connect to psql:
```bash
docker exec -it n8n-postgres-1 psql -U <YOUR_USER> -d <YOUR_DB>
```

Run this SQL:

```sql
CREATE TABLE p01_p01_lead_sessions (
    id SERIAL PRIMARY KEY,
    lead_data JSONB,
    research_output TEXT,
    score INTEGER,
    score_reasoning TEXT,
    outreach_draft TEXT,
    crm_status TEXT,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Note:** Table uses `p01_` prefix for multi-project naming consistency.

**Verify:** Run `\dt p01_p01_lead_sessions` in psql → confirm table exists with all columns.

---

## Step 2 — Set Up RAG Knowledge Base

**Where:** Local Docker Postgres with pgvector (`n8n-postgres-1` container)

### 2a. Enable pgvector and create documents table

Connect to psql and run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

```sql
CREATE TABLE p01_documents (
    id SERIAL PRIMARY KEY,
    text TEXT,
    embedding vector(1536),
    metadata JSONB
);
```

**Note:** Table uses `p01_` prefix. Columns must be `text`, `embedding`, `metadata` — required by n8n PGVector node.

### 2b. Prepare these documents as plain text

**ICP Criteria document:**
```
Ideal Customer Profile:
- Company size: 10-500 employees
- Industries: SaaS, fintech, e-commerce, marketing agencies
- Uses: CRM tools (HubSpot, Salesforce, Pipedrive)
- Budget: £5,000-£50,000 for automation projects
- Pain points: manual lead processing, slow outreach, inconsistent follow-up
- Green flags: recently funded, hiring sales roles, expanding to new markets
- Red flags: under 5 employees, no online presence, government/public sector
```

**Product/Service Info document:**
```
Services Offered:
- AI automation workflow design and build (n8n, Make)
- Agentic AI systems — multi-agent orchestration for business processes
- LLM integration — OpenAI, Claude, Gemini API setup and prompt engineering
- RAG pipeline setup — document ingestion, vector search, contextual retrieval
- CRM automation — lead enrichment, scoring, automated outreach
```

**Competitor Notes document:**
```
Competitors:
- Clay.com — lead enrichment platform, no custom agent logic
- Apollo.io — outbound sales tool, template-based outreach
- Instantly.ai — cold email automation, no research/scoring layer
Our edge: fully custom multi-agent pipeline with RAG-driven scoring, not a template tool
```

### 2c. Build a loader workflow in n8n

1. Create new workflow: `P01 - RAG Document Loader`
2. **Node 1:** Manual Trigger
3. **Node 2:** Code node — paste all 3 documents as an array returning `{ json: { content: "..." } }` for each
4. **Node 3:** Postgres PGVector Store node
   - Mode: Insert
   - Table: `p01_documents`
   - Connect **OpenAI Embeddings** sub-node (`text-embedding-3-small`)
   - Connect **Default Data Loader** sub-node → set text field to `{{ $json.content }}`

**Verify:** Run the workflow. Check psql → `SELECT id, LEFT(text, 50) FROM p01_documents;` → should show 3 rows.

---

## Step 3 — Create Credentials in n8n

**Where:** n8n → Settings → Credentials

Create these credentials (if not already set up):

| Credential | For | Notes |
|---|---|---|
| Postgres | Shared state DB (p01_p01_lead_sessions, p01_documents) | Host: container name in docker-compose |
| Google Gemini | Research Agent | API key from Google AI Studio |
| Anthropic (Claude) | Scoring Agent | API key from console.anthropic.com. Test may fail with "resource not found" — known n8n bug, save anyway. |
| OpenAI | Personalisation Agent + Embeddings | API key from platform.openai.com |
| Tavily | Research Agent web search (replaces Tavily) | API key from tavily.com — 1,000 free searches/month, better for AI agents than Tavily |
| Gmail | HITL approval emails | Connect via OAuth in n8n |
| Airtable Personal Access Token | CRM Agent destination | Scopes: data.records:read, data.records:write, schema.bases:read |

**Verify:** Test each credential connection in n8n.

---

## Step 4 — Build the Research Agent

**Where:** n8n → New Workflow → Name: `Research Agent`

### Nodes in order:

**Node 1: "When Called by Another Workflow" trigger**
- This receives the session ID from the Orchestrator

**Node 2: Postgres (Read)**
- Operation: Select Rows
- Table: `p01_lead_sessions`
- Column: `id`
- Value: `{{ $json.session_id }}` (from trigger input)

**Node 3: AI Agent**
- LLM: Google Gemini Pro
- System prompt:
```
You are a lead research specialist. Given a lead's name, company, and domain, research them thoroughly.

Find and summarise:
1. What the company does (products/services)
2. Company size and industry
3. Recent news (funding, launches, leadership changes)
4. Technology stack they use (if findable)
5. Key decision makers

Return a structured research summary. Be factual — if you can't find something, say so.
```
- Connect these tools to the agent:
  - **HTTP Request tool** — for web scraping (agent provides the URL)
  - **Tavily tool** — or use HTTP Request pointed at `https://serpapi.com/search?api_key=YOUR_KEY&q=QUERY`

**Node 4: Postgres (Update)**
- Table: `p01_lead_sessions`
- Set: `research_output` = `{{ $json.output }}` (AI Agent output)
- Set: `status` = `researching`
- Where: `id` = session ID

**Verify:** Run manually with a test session ID. Check Postgres → `research_output` column should be populated.

---

## Step 5 — Build the Scoring Agent

**Where:** n8n → New Workflow → Name: `Scoring Agent`

### Nodes in order:

**Node 1: "When Called by Another Workflow" trigger**

**Node 2: Postgres (Read)**
- Read `research_output` from `p01_lead_sessions` where `id` = session ID

**Node 3: Supabase Vector Store (Retrieve)**
- Query: Use the research output text as the similarity search query
- Top K: 3 (retrieve top 3 matching document chunks)
- This pulls your ICP criteria from the RAG knowledge base

**Node 4: AI Agent**
- LLM: Claude Sonnet (Anthropic credential)
- System prompt:
```
You are a lead scoring specialist. You receive company research and ICP (Ideal Customer Profile) criteria.

Score this lead from 1 to 10 based on how well they match the ICP.

Return EXACTLY this format:
SCORE: [number 1-10]
REASONING: [2-3 sentences explaining the score]

Scoring guide:
- 8-10: Strong match, prioritise immediately
- 5-7: Moderate match, worth pursuing
- 3-4: Weak match, low priority
- 1-2: Poor match, do not pursue
```
- Input: Combine the research output + RAG retrieved documents

**Node 5: Code node (parse score)**
```javascript
const response = $input.first().json.output;
const scoreMatch = response.match(/SCORE:\s*(\d+)/);
const reasoningMatch = response.match(/REASONING:\s*(.*)/s);

return [{
  json: {
    score: scoreMatch ? parseInt(scoreMatch[1]) : 0,
    reasoning: reasoningMatch ? reasoningMatch[1].trim() : response
  }
}];
```

**Node 6: Postgres (Update)**
- Set: `score` = `{{ $json.score }}`
- Set: `score_reasoning` = `{{ $json.reasoning }}`
- Set: `status` = `scored`
- Where: `id` = session ID

**Verify:** Test with same lead. Check Postgres → `score` and `score_reasoning` populated.

---

## Step 6 — Build the Personalisation Agent

**Where:** n8n → New Workflow → Name: `Personalisation Agent`

### Nodes in order:

**Node 1: "When Called by Another Workflow" trigger**

**Node 2: Postgres (Read)**
- Read `lead_data`, `research_output`, `score`, `score_reasoning` from `p01_lead_sessions`
- Where: `id` = session ID

**Node 3: AI Agent**
- LLM: GPT-4o (OpenAI credential)
- System prompt:
```
You are an outreach copywriter. Using the company research and lead score, write a personalised cold email.

Rules:
- Keep it under 150 words
- Open with something specific to their company (from the research)
- Reference a pain point relevant to their industry
- If score is 8+: be direct with a meeting ask
- If score is 5-7: softer approach, offer value first (free resource, insight)
- If score is below 5: nurture tone, share relevant content
- No generic phrases like "I hope this finds you well"
- Sign off as: Rinoy Francis, AI Automation Engineer

Return the email with Subject line and Body separately.
```
- Input: Pass all the Postgres data (research, score, reasoning, lead data)

**Node 4: Postgres (Update)**
- Set: `outreach_draft` = AI Agent output
- Set: `status` = `drafted`
- Where: `id` = session ID

**Verify:** Check Postgres → `outreach_draft` should contain a personalised email with subject and body.

---

## Step 7 — Build the HITL Approval Gate

**Where:** This will be part of the Orchestrator workflow (Step 9), but set up the approval webhook first.

### Create a new workflow: `HITL Approval Handler`

**Node 1: Webhook Trigger**
- Method: GET
- Path: `/lead-approval`
- This receives the approve/reject click

**Node 2: Postgres (Update)**
- If query param `action` = `approve`: set `status` = `approved`
- If query param `action` = `reject`: set `status` = `rejected`
- Where: `id` = query param `id`

**Node 3: Respond to Webhook**
- Approve: Return "Lead approved. CRM update in progress."
- Reject: Return "Lead rejected. No action taken."

### Approval email format (used in Orchestrator):

```
Subject: Lead Approval Required: [Company Name]

LEAD: [Name] at [Company]
SCORE: [X]/10
REASONING: [Score reasoning]

DRAFT EMAIL:
[Outreach draft]

---
✅ APPROVE: https://your-n8n-url/webhook/lead-approval?action=approve&id=[session_id]
❌ REJECT: https://your-n8n-url/webhook/lead-approval?action=reject&id=[session_id]
```

**Verify:** Open the approve URL in browser → should update Postgres status.

---

## Step 8 — Build the CRM Agent

**Where:** n8n → New Workflow → Name: `CRM Agent`

### Nodes in order:

**Node 1: "When Called by Another Workflow" trigger**

**Node 2: Postgres (Read)**
- Read ALL fields from `p01_lead_sessions` where `id` = session ID

**Node 3: Code node (format CRM payload)**
```javascript
const session = $input.first().json;
const leadData = typeof session.lead_data === 'string'
  ? JSON.parse(session.lead_data)
  : session.lead_data;

return [{
  json: {
    name: leadData.name,
    email: leadData.email,
    company: leadData.company,
    research_summary: session.research_output,
    lead_score: session.score,
    score_reasoning: session.score_reasoning,
    outreach_draft: session.outreach_draft,
    status: 'approved',
    source: 'Lead Intelligence System'
  }
}];
```

**Node 4: Airtable node (or HubSpot HTTP Request)**
- For **Airtable** (recommended to start):
  - Create an Airtable base with columns matching the fields above
  - Use Airtable node → Create Record
  - Map each field
- For **HubSpot**:
  - HTTP Request → POST to `https://api.hubapi.com/crm/v3/objects/contacts`
  - Map fields to HubSpot contact properties

**Node 5: Postgres (Update)**
- Set: `crm_status` = `written`
- Set: `status` = `completed`
- Where: `id` = session ID

**Verify:** Run with a test session. Check Airtable/HubSpot → enriched contact should appear.

---

## Step 9 — Build the Orchestrator (Main Workflow)

**Where:** n8n → New Workflow → Name: `Lead Intelligence Orchestrator`

This is the master workflow that ties everything together.

### Nodes in order:

**Node 1: Webhook Trigger**
- Method: POST
- Path: `/lead-intake`
- This is the entry point for all leads

**Node 2: Postgres (Insert)**
- Table: `p01_lead_sessions`
- Set: `lead_data` = `{{ JSON.stringify($json) }}` (entire webhook body)
- Set: `status` = `pending`
- Return: the new row `id`

**Node 3: Execute Workflow → `Research Agent`**
- Pass: `{ "session_id": [id from step 2] }`

**Node 4: Execute Workflow → `Scoring Agent`**
- Pass: `{ "session_id": [id] }`

**Node 5: Execute Workflow → `Personalisation Agent`**
- Pass: `{ "session_id": [id] }`

**Node 6: Postgres (Read)**
- Read the full session to get data for the approval email

**Node 7: Send Email (Gmail/SMTP)**
- To: `rinoyfrancis2job@gmail.com`
- Subject: `Lead Approval Required: {{ $json.company }}`
- Body: Use the approval email format from Step 7

**Node 8: Wait node**
- Resume: On webhook call
- Webhook path will be unique per execution

**Alternative approach for HITL:** Instead of Wait node, use a polling pattern:
- After sending the email, end the Orchestrator
- Create a separate workflow triggered by the HITL Approval Handler (Step 7)
- When approval webhook fires and status = `approved`, that workflow calls the CRM Agent

**Node 9: IF node**
- Condition: Check if status from Wait/webhook = approved
- True → Node 10
- False → Postgres Update status = `rejected` → End

**Node 10: Execute Workflow → `CRM Agent`**
- Pass: `{ "session_id": [id] }`

**Node 11: Respond to Webhook**
- Return: `{ "status": "completed", "session_id": [id] }`

**Verify:** Activate the workflow. The webhook URL is now live.

---

## Step 10 — End-to-End Test

Send a test lead to your webhook:

```bash
curl -X POST https://your-n8n-url/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Smith",
    "email": "john@testcompany.com",
    "company": "TestCompany Ltd",
    "domain": "testcompany.com"
  }'
```

### Checklist — verify each stage in Postgres:

| Check | Column | Expected |
|---|---|---|
| Orchestrator created session | `lead_data` | JSON with name, email, company, domain |
| Research Agent ran | `research_output` | Structured company research summary |
| Scoring Agent ran | `score` | Number 1-10 |
| Scoring Agent ran | `score_reasoning` | 2-3 sentence explanation |
| Personalisation Agent ran | `outreach_draft` | Personalised email with subject + body |
| Status before approval | `status` | `drafted` or `awaiting_approval` |
| Approval email received | Your inbox | Email with approve/reject links |
| After clicking approve | `status` | `approved` → `completed` |
| CRM Agent ran | `crm_status` | `written` |
| CRM record created | Airtable/HubSpot | Full enriched contact record |

### If something fails:

1. Check n8n execution log for the failing workflow
2. Check which Postgres column is empty — that tells you which agent failed
3. Test that agent's workflow independently with a hardcoded session ID
4. Check credentials are valid (API keys not expired)

---

## After All Steps Complete

- Export all workflows as JSON into the `workflows/` folder
- Take screenshots of the n8n canvas for each workflow
- Update `progress.md` to mark project as complete
- Push to GitHub with README and architecture docs
