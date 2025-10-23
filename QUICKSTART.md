# Quick Start Guide

Get your Slack IT Knowledge Base RAG Agent running in **30 minutes**.

## Prerequisites Checklist

- [ ] n8n instance (cloud or self-hosted)
- [ ] Slack workspace (admin access)
- [ ] Google Cloud Platform account
- [ ] Supabase account
- [ ] OpenAI API key
- [ ] LlamaIndex account

---

## Setup Steps

### 1. Create API Accounts (10 min)

**Slack Bot:**
1. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create New App
2. Add bot scopes: `chat:write`, `im:history`, `im:read`, `im:write`
3. Install to workspace → Copy Bot Token (`xoxb-...`)

**Google Drive:**
1. [console.cloud.google.com](https://console.cloud.google.com) → New Project
2. Enable Google Drive API
3. Create Service Account → Download JSON key
4. Share IT documentation folder with service account email

**Supabase:**
1. [supabase.com](https://supabase.com) → New Project
2. Copy URL and anon key from Settings → API

**OpenAI:**
1. [platform.openai.com](https://platform.openai.com) → API Keys → Create

**LlamaParse:**
1. [cloud.llamaindex.ai](https://cloud.llamaindex.ai) → API Keys → Create

### 2. Configure Supabase (5 min)

Run this SQL in Supabase SQL Editor:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,
  metadata JSONB,
  embedding VECTOR(1536)
);

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE TABLE chat_memory (
  id BIGSERIAL PRIMARY KEY,
  session_id TEXT NOT NULL,
  sender TEXT NOT NULL,
  message TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON chat_memory(session_id, created_at DESC);
```

### 3. Import Workflow (5 min)

1. Download `workflows/slack-it-rag-agent.json`
2. Open n8n → Workflows → Import from File
3. Workflow imports with 43 nodes

### 4. Add Credentials (5 min)

Configure each credential in n8n:

| Service | Credential Type | Value |
|---------|----------------|-------|
| Slack | OAuth Token | `xoxb-...` |
| Google Drive | Service Account JSON | Upload JSON file |
| Supabase | API Key | URL + anon key |
| PostgreSQL | Connection String | Supabase DB string |
| OpenAI | API Key | `sk-...` |
| LlamaParse | Bearer Token | `llx-...` |

### 5. Connect Webhook (5 min)

1. Activate workflow in n8n
2. Copy webhook URL from "Receive DMs" node
3. Go to Slack → Event Subscriptions
4. Enable + paste webhook URL
5. Subscribe to: `message.im`
6. Save → Reinstall app

---

## Test Your Setup

**Test 1: Document Indexing**
1. Upload a text file to Google Drive folder
2. Content: "The WiFi password is Test123"
3. Wait 1 minute
4. Check Supabase: `SELECT COUNT(*) FROM documents;`

**Test 2: Slack Bot**
1. Open Slack → DM the bot
2. Send: "What is the WiFi password?"
3. Should respond with the password + source

---

## Common Issues

**Bot not responding?**
- Check workflow is Active
- Verify webhook URL in Slack

**Documents not indexing?**
- Verify service account has folder access
- Check LlamaParse API key is valid

**Poor responses?**
- Increase Top K to 5 in vector search node
- Lower similarity threshold to 0.5

---

## Next Steps

1. Add your IT documentation to Google Drive
2. Customize system prompt in AI Agent node
3. Review [full documentation](README.md)
4. Check [troubleshooting guide](docs/TROUBLESHOOTING.md)

---

**Estimated monthly cost**: $10-50 (moderate usage)

For detailed instructions, see [docs/SETUP.md](docs/SETUP.md)
