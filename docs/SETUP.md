# 🛠️ IT Support Bot Setup Guide

Complete step-by-step guide to get IT Support Bot running in your environment.

## 📋 Prerequisites Checklist

Before starting, ensure you have:
- [ ] n8n instance (cloud or self-hosted)
- [ ] Slack workspace admin access
- [ ] Google Cloud Platform account
- [ ] Supabase account (free tier works)
- [ ] OpenAI API account with credits
- [ ] LlamaIndex account for LlamaParse

## ⏱️ Estimated Setup Time
- **First-time setup**: 45-60 minutes
- **With existing integrations**: 20-30 minutes

---

## 🎯 Step 1: Slack App Configuration

### 1.1 Create Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **"Create New App"**
3. Choose **"From scratch"**
4. Enter:
   - **App Name**: `IT Support Bot`
   - **Workspace**: Select your workspace
5. Click **"Create App"**

### 1.2 Configure Bot Permissions

1. In the left sidebar, click **"OAuth & Permissions"**
2. Scroll to **"Scopes"** → **"Bot Token Scopes"**
3. Add the following scopes:
   ```
   chat:write          # Send messages
   channels:history    # Read channel messages
   im:history         # Read DM history
   im:read            # Access to DMs
   im:write           # Write to DMs
   users:read         # Read user information
   ```
4. Scroll up and click **"Install to Workspace"**
5. Authorize the app
6. **Copy the Bot User OAuth Token** (starts with `xoxb-`)
   - Save it as `SLACK_BOT_TOKEN` in your `.env` file

### 1.3 Enable Event Subscriptions (After n8n Setup)

*You'll return to this step after setting up the n8n webhook*

---

## 🗄️ Step 2: Google Drive Setup

### 2.1 Create Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click **"Select a project"** → **"New Project"**
3. Enter project name: `slack-it-rag-agent-docs`
4. Click **"Create"**

### 2.2 Enable Google Drive API

1. In the search bar, type "Google Drive API"
2. Click **"Enable"**
3. Wait for confirmation

### 2.3 Create Service Account (Recommended)

1. Navigate to **"IAM & Admin"** → **"Service Accounts"**
2. Click **"Create Service Account"**
3. Enter:
   - **Name**: `slack-it-rag-agent-service`
   - **Description**: `Service account for IT Support Bot n8n workflow`
4. Click **"Create and Continue"**
5. Skip role assignment (click **"Continue"**)
6. Click **"Done"**

### 2.4 Generate Service Account Key

1. Click on the newly created service account
2. Go to **"Keys"** tab
3. Click **"Add Key"** → **"Create new key"**
4. Choose **JSON** format
5. Click **"Create"**
6. **Save the downloaded JSON file** securely
   - Rename it to `google-credentials.json`
   - Place in your project root
   - This file contains your credentials

### 2.5 Share Google Drive Folder

1. Open [Google Drive](https://drive.google.com)
2. Create or navigate to your IT documentation folder
3. Right-click → **"Share"**
4. Add the service account email (from the JSON file, looks like: `slack-it-rag-agent-service@project-name.iam.gserviceaccount.com`)
5. Set permission to **"Viewer"** or **"Editor"** (if it needs to modify files)
6. **Copy the Folder ID** from the URL:
   ```
   https://drive.google.com/drive/folders/11IZVdhbW7gxJseyJMLuMkwZCnWxlVoij
                                            ↑ This is your FOLDER_ID
   ```
7. Save it as `GOOGLE_DRIVE_FOLDER_ID` in your `.env` file

---

## 🗃️ Step 3: Supabase Setup

### 3.1 Create Supabase Project

1. Go to [supabase.com](https://supabase.com)
2. Sign in and click **"New project"**
3. Enter:
   - **Name**: `slack-it-rag-agent-kb`
   - **Database Password**: Generate a strong password (save it!)
   - **Region**: Choose closest to your users
4. Click **"Create new project"**
5. Wait 2-3 minutes for provisioning

### 3.2 Get API Credentials

1. Go to **Project Settings** → **API**
2. Copy the following:
   - **URL**: `https://xxxxx.supabase.co`
   - **anon/public key**: Long string starting with `eyJ...`
3. Save to `.env`:
   ```
   SUPABASE_URL=https://xxxxx.supabase.co
   SUPABASE_ANON_KEY=eyJ...
   ```

### 3.3 Get Database Connection String

1. Go to **Project Settings** → **Database**
2. Scroll to **Connection string** → **URI**
3. Copy the string (replace `[YOUR-PASSWORD]` with your actual password)
4. Save to `.env`:
   ```
   SUPABASE_POSTGRES_CONNECTION=postgresql://postgres:your-password@db.xxxxx.supabase.co:5432/postgres
   ```

### 3.4 Create Database Tables

1. Go to **SQL Editor** in Supabase dashboard
2. Click **"New query"**
3. Paste and run the following SQL:

```sql
-- Enable pgvector extension for vector similarity search
CREATE EXTENSION IF NOT EXISTS vector;

-- Create documents table for vector storage
CREATE TABLE documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  embedding VECTOR(1536)  -- OpenAI ada-002 uses 1536 dimensions
);

-- Create index for faster similarity search
-- Note: This may take a few seconds
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Create function for similarity search
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding VECTOR(1536),
  match_threshold FLOAT,
  match_count INT
)
RETURNS TABLE (
  id BIGINT,
  content TEXT,
  metadata JSONB,
  similarity FLOAT
)
LANGUAGE SQL STABLE
AS $$
  SELECT
    documents.id,
    documents.content,
    documents.metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE 1 - (documents.embedding <=> query_embedding) > match_threshold
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
$$;

-- Create chat memory table for conversation history
CREATE TABLE chat_memory (
  id BIGSERIAL PRIMARY KEY,
  session_id TEXT NOT NULL,
  sender TEXT NOT NULL CHECK (sender IN ('user', 'assistant')),
  message TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Create index for faster session lookups
CREATE INDEX idx_chat_memory_session ON chat_memory(session_id, created_at DESC);

-- Create cleanup function (optional - runs weekly to delete old conversations)
CREATE OR REPLACE FUNCTION cleanup_old_chat_memory()
RETURNS void
LANGUAGE SQL
AS $$
  DELETE FROM chat_memory 
  WHERE created_at < NOW() - INTERVAL '30 days';
$$;

-- Grant necessary permissions
GRANT ALL ON documents TO postgres;
GRANT ALL ON chat_memory TO postgres;
GRANT ALL ON SEQUENCE documents_id_seq TO postgres;
GRANT ALL ON SEQUENCE chat_memory_id_seq TO postgres;
```

4. Click **"Run"**
5. Verify tables were created by going to **Table Editor**

---

## 🤖 Step 4: OpenAI Setup

### 4.1 Get API Key

1. Go to [platform.openai.com](https://platform.openai.com)
2. Sign in or create an account
3. Go to **API keys** in the left sidebar
4. Click **"Create new secret key"**
5. Enter name: `slack-it-rag-agent`
6. Copy the key (starts with `sk-`)
7. Save to `.env`:
   ```
   OPENAI_API_KEY=sk-your-key-here
   ```

### 4.2 Add Credit/Billing

1. Go to **Billing** → **Payment methods**
2. Add a payment method
3. Set usage limits (recommended: $50/month for testing)

### 4.3 Check Model Access

Ensure you have access to:
- `gpt-4` (for agent reasoning)
- `text-embedding-ada-002` (for embeddings)

---

## 📄 Step 5: LlamaParse Setup

### 5.1 Create Account

1. Go to [cloud.llamaindex.ai](https://cloud.llamaindex.ai)
2. Sign up with GitHub or email
3. Free tier includes **1,000 pages/day**

### 5.2 Generate API Key

1. Go to **API Keys** in dashboard
2. Click **"Create API Key"**
3. Copy the key (starts with `llx-`)
4. Save to `.env`:
   ```
   LLAMAPARSE_API_KEY=llx-your-key-here
   ```

---

## 🔧 Step 6: Import Workflow to n8n

### 6.1 Access n8n

**For n8n Cloud:**
1. Go to [app.n8n.cloud](https://app.n8n.cloud)
2. Sign in to your account

**For Self-Hosted:**
1. Access your n8n instance (e.g., `http://localhost:5678`)

### 6.2 Import Workflow

1. Click **"Workflows"** in the left sidebar
2. Click **"Add Workflow"** (+ icon)
3. Click the three-dot menu (⋯) → **"Import from File"**
4. Select `workflows/slack-it-rag-agent.json`
5. Wait for import to complete

### 6.3 Review Imported Workflow

You should see 43 nodes connected in a flow. Don't panic if you see red error indicators - we'll fix those by adding credentials.

---

## 🔑 Step 7: Configure n8n Credentials

n8n uses a credential system for secure API access. You'll need to create credentials for each service.

### 7.1 Slack API Credential

1. Find any **Slack** node (e.g., "Send Initial Message")
2. Click the node
3. Under **Credentials**, click the dropdown
4. Select **"Create New"**
5. Choose **"Slack OAuth2 API"** or **"Slack API"**
6. Enter:
   - **Access Token**: Your `SLACK_BOT_TOKEN` from Step 1
7. Click **"Save"**
8. Apply to all Slack nodes

### 7.2 Google Drive Credential

1. Find a **Google Drive** node
2. Click **Credentials** → **"Create New"**
3. Choose **"Google Service Account"**
4. Paste the contents of your `google-credentials.json` file
5. Click **"Save"**

### 7.3 Supabase Credential

1. Find a **Supabase** node
2. Click **Credentials** → **"Create New"**
3. Enter:
   - **Host**: Your Supabase URL (e.g., `xxxxx.supabase.co`)
   - **API Key**: Your `SUPABASE_ANON_KEY`
4. Click **"Save"**

### 7.4 PostgreSQL Credential (for Chat Memory)

1. Find the **"Postgres Chat Memory"** node
2. Click **Credentials** → **"Create New"**
3. Choose **"Postgres"**
4. Enter connection details from your `SUPABASE_POSTGRES_CONNECTION`:
   - **Host**: `db.xxxxx.supabase.co`
   - **Database**: `postgres`
   - **User**: `postgres`
   - **Password**: Your database password
   - **Port**: `5432`
   - **SSL**: Enable (for Supabase)
5. Click **"Save"**

### 7.5 OpenAI Credential

1. Find an **OpenAI** node (e.g., "OpenAI Chat Model")
2. Click **Credentials** → **"Create New"**
3. Enter:
   - **API Key**: Your `OPENAI_API_KEY`
4. Click **"Save"**

### 7.6 HTTP Bearer Auth (for LlamaParse)

1. Find **"Upload File1"** (HTTP Request node)
2. Click **Credentials** → **"Create New"**
3. Choose **"HTTP Header Auth"** or **"Bearer Auth"**
4. Enter:
   - **Token**: Your `LLAMAPARSE_API_KEY`
5. Click **"Save"**
6. Apply to all LlamaParse HTTP Request nodes

---

## 🔗 Step 8: Configure Webhook

### 8.1 Activate Test Webhook

1. Find the **"Receive DMs"** webhook node
2. Click on it
3. Click **"Listen for Test Event"** or toggle workflow to **Active**
4. The webhook URL will appear (format: `https://your-n8n.com/webhook/xxxxx`)
5. **Copy this URL**

### 8.2 Add Webhook to Slack

1. Return to [api.slack.com/apps](https://api.slack.com/apps)
2. Select your IT Support Bot app
3. Go to **"Event Subscriptions"** in the left sidebar
4. Toggle **"Enable Events"** to ON
5. Paste your n8n webhook URL in **"Request URL"**
6. Wait for verification (should show green checkmark)
7. Scroll to **"Subscribe to bot events"**
8. Click **"Add Bot User Event"**
9. Add: `message.im` (for direct messages)
10. Click **"Save Changes"**
11. Slack will prompt to reinstall - click **"Reinstall App"**

---

## ✅ Step 9: Test the Workflow

### 9.1 Activate Workflow

1. In n8n, toggle the workflow to **Active** (top-right switch)
2. Ensure all nodes show green checkmarks (no red errors)

### 9.2 Test Knowledge Base Ingestion

1. Add a test document to your Google Drive folder
   - Create a simple text file: `test-doc.txt`
   - Content: "The WiFi password is SecurePassword123"
2. Wait 1 minute (trigger polls every minute)
3. Check n8n execution history for successful run
4. Verify in Supabase:
   ```sql
   SELECT COUNT(*) FROM documents;
   ```

### 9.3 Test Slack Bot

1. Open Slack
2. Find **IT Support Bot** in the Apps section
3. Send a direct message: `What is the WiFi password?`
4. You should receive a response within 3-5 seconds

**Expected Response:**
```
Based on the documentation, the WiFi password is SecurePassword123.

Source: test-doc.txt
```

### 9.4 Test Conversation Memory

1. Send: `What did I just ask you?`
2. Bot should remember: "You asked about the WiFi password."

---

## 🎨 Step 10: Customization

### 10.1 Customize System Prompt

1. Open the **AI Agent** node
2. Find the **System Message** field
3. Edit to match your organization's tone:

```
You are IT Support Bot, the AI assistant for [Your Company Name].

Your role:
- Answer IT questions using our knowledge base
- Be friendly, professional, and concise
- Always cite your sources
- If unsure, guide users to contact IT at support@company.com

Company-specific info:
- IT Helpdesk Hours: Mon-Fri, 9 AM - 5 PM EST
- Emergency Contact: +1-555-IT-HELP
```

### 10.2 Adjust Vector Search

In **Supabase Vector Store** retrieval node:
- **Top K**: `5` (retrieves more documents, better context)
- **Similarity Threshold**: `0.7` (0.0-1.0, higher = stricter matching)

### 10.3 Customize Response Format

In the **Validate Response** code node, you can format responses:

```javascript
// Add company branding to responses
const response = $input.first().json.output;
const formattedResponse = `
${response}

---
📚 Need more help? Visit https://support.company.com
📧 Email: it@company.com
`;

return [{ json: { response: formattedResponse } }];
```

---

## 🐛 Troubleshooting Common Issues

### Issue 1: Webhook Not Receiving Messages

**Symptoms**: Bot doesn't respond in Slack

**Solutions**:
- [ ] Verify workflow is **Active**
- [ ] Check webhook URL is correctly configured in Slack Event Subscriptions
- [ ] Ensure bot was invited to the channel/DM
- [ ] Test webhook manually:
  ```bash
  curl -X POST https://your-n8n.com/webhook/xxxxx \
    -H "Content-Type: application/json" \
    -d '{"event":{"type":"message","text":"test"}}'
  ```

### Issue 2: Documents Not Indexing

**Symptoms**: Bot says "I don't have information about that"

**Solutions**:
- [ ] Check Google Drive trigger is active
- [ ] Verify service account has access to folder
- [ ] Test LlamaParse API:
  ```bash
  curl -X POST https://api.cloud.llamaindex.ai/api/v1/parsing/upload \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -F "file=@test.pdf"
  ```
- [ ] Check Supabase documents table:
  ```sql
  SELECT * FROM documents LIMIT 5;
  ```

### Issue 3: Bot Responses Are Slow

**Symptoms**: Takes >10 seconds to respond

**Solutions**:
- [ ] Reduce **Top K** in vector retrieval (try 3)
- [ ] Check OpenAI API status
- [ ] Monitor n8n execution time in workflow history
- [ ] Consider upgrading n8n instance resources

### Issue 4: Memory Not Working

**Symptoms**: Bot doesn't remember previous messages

**Solutions**:
- [ ] Verify PostgreSQL credential is correct
- [ ] Check `chat_memory` table exists
- [ ] Test connection:
  ```sql
  SELECT * FROM chat_memory ORDER BY created_at DESC LIMIT 10;
  ```
- [ ] Ensure session IDs are consistent (check "Edit Fields" node)

---

## 📊 Monitoring & Maintenance

### Daily Checks
- [ ] Review n8n execution history for errors
- [ ] Monitor Supabase storage usage
- [ ] Check OpenAI API usage/costs

### Weekly Tasks
- [ ] Review most asked questions (analyze `chat_memory`)
- [ ] Update knowledge base documents
- [ ] Clean up old chat history (optional)

### Monthly Tasks
- [ ] Rotate API keys
- [ ] Review and optimize system prompt
- [ ] Analyze response quality metrics
- [ ] Update workflow dependencies

---

## 🎓 Next Steps

Once your bot is running smoothly:

1. **Add Analytics**: Integrate with logging tools (e.g., LogDNA, Datadog)
2. **Create Dashboard**: Build Grafana dashboard for metrics
3. **Expand Knowledge Base**: Add more documents systematically
4. **A/B Test Prompts**: Experiment with different system messages
5. **Add Escalation**: Create fallback to human agents for complex queries
6. **Multi-Language**: Add translation for global teams

---

## 🆘 Getting Help

If you're stuck:

1. **Check n8n Logs**: Look for error messages in execution history
2. **n8n Community**: [community.n8n.io](https://community.n8n.io)
3. **GitHub Issues**: Open an issue in this repository
4. **LinkedIn**: Connect with me for questions about this specific setup

---

**🎉 Congratulations! Your IT Support Bot is now live!**

Time to populate your knowledge base and let the bot handle those repetitive IT questions! 🚀
