# SpaceKiwi — Etsy Customer Service Agent

An agentic n8n workflow that triages incoming Etsy customer messages using Claude AI, routes them by priority, and auto-responds to simple questions — so the store owner only gets pinged when it actually matters.

Built with **Claude Code + n8n MCP** as the development workflow: Claude designed, debugged, and iterated on the workflow entirely through the n8n API without touching the n8n UI.

---

## What it does

| Message type | Behaviour |
|---|---|
| Rush order / deadline / occasion | 🚨 Immediate Slack alert to store owner within 1 hour |
| Mixed message (urgent + simple question) | Answers the simple part instantly, alerts owner for the urgent part |
| Exchange / return / custom order | Draft response queued for owner batch review |
| Simple FAQ / product question | Auto-replies using product catalogue and shop policies |
| Low-confidence / ambiguous | Escalates to owner with full context |

### Key features
- **Conversation memory** — each session's history is stored and retrieved, so follow-up messages have full context
- **Partial auto-response** — mixed-intent messages answer the answerable parts immediately while escalating the rest
- **Priority isolation** — follow-up low-priority questions in an active urgent session are auto-answered rather than re-alerting the store owner
- **Safety gates** — financial topics, damage claims, and refund requests are never auto-responded

---

## Stack

- **n8n** (self-hosted via Docker) — workflow orchestration
- **Claude Haiku** (`claude-haiku-4-5`) — message classification and intent detection
- **Claude Sonnet** (`claude-sonnet-4-6`) — auto-reply generation
- **Google Docs** — shop FAQ and policies source
- **Google Sheets** — audit log and conversation history
- **Slack** — store owner alerts

---

## Workflow

```
Chat Trigger / Webhook
  → Validate & Parse Input
  → Fetch Listing Context
  → Fetch FAQ and Policies (Google Docs)
  → Load Conversation History (Google Sheets)
  → Build AI Context
  → Claude: Classify Message
  → Parse Classification
  → Log to Audit Sheet
  → Route by Priority
      ├── HIGH  → Format Alert → Slack → Chat Response
      ├── MEDIUM → Build Draft → Slack (batch review)
      ├── LOW   → Safety Gate → Generate Auto-Reply → Respond
      └── escalate → Format Escalation → Slack → Chat Response
```

---

## Setup

### Prerequisites
- Docker
- n8n self-hosted (`docker run -p 5678:5678 n8nio/n8n`)
- Anthropic API key
- Google Cloud project with Docs + Sheets OAuth2 enabled
- Slack app with `chat:write` scope

### 1. Clone and configure MCP
```bash
git clone https://github.com/YOUR_USERNAME/n8n-etsy-agentic-workflow.git
cd n8n-etsy-agentic-workflow
cp .mcp.json.example .mcp.json
# Edit .mcp.json and add your N8N_API_KEY
```

### 2. Import the workflow
In the n8n UI: **Workflows → Import from file** → select `workflows/etsy-customer-service-agent-v2.json`

Or via API:
```bash
curl -X POST http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d @workflows/etsy-customer-service-agent-v2.json
```

### 3. Set up credentials in n8n
Go to **Settings → Credentials** and create:
- `Anthropic API` — your Anthropic API key
- `Google Docs account` — OAuth2 (link to your FAQ doc)
- `Google Sheets account` — OAuth2 (link to your audit/history spreadsheet)
- `Slack OAuth2 API` — your Slack bot token

### 4. Configure environment variables
In your n8n Docker `.env`:
```
ETSY_SHOP_NAME=YourShopName
ETSY_SELLER_NAME=YourName
```

### 5. Google Sheets structure
Your spreadsheet needs two tabs:

**Sheet1** (audit log) — columns:
`Timestamp | Message ID | Priority | Listing | Message | Category | Sentiment | Confidence | Draft Response/Our Response | Financial Topic | Status | Conversation ID`

### 6. Activate and test
Activate the workflow in n8n, then open the chat interface at:
`http://localhost:5678/webhook/YOUR_WEBHOOK_ID/chat`

---

## Development approach

This project was built using **Claude Code with the n8n MCP server** — Claude had direct API access to the running n8n instance and could create, update, validate, and debug workflows through conversation, without manual UI interaction.

The `CLAUDE.md` file in this repo contains the project instructions used to guide Claude throughout development.
