# 🔧 IT Support Bot Troubleshooting Guide

Quick solutions to common issues you may encounter with IT Support Bot.

## 🚨 Quick Diagnostics

### System Health Check Checklist

Run through this checklist first:

```
[ ] n8n workflow is Active (green toggle in top-right)
[ ] All nodes show green checkmarks (no red errors)
[ ] All 7 credentials are configured
[ ] Slack Event Subscriptions shows verified webhook
[ ] Google Drive folder has documents
[ ] Supabase tables exist and have data
[ ] OpenAI account has credits
```

---

## 📱 Slack Integration Issues

### Problem: Bot Not Responding to Messages

#### Symptoms
- Send DM to bot in Slack
- No response within 10 seconds
- No "typing..." indicator

#### Diagnostic Steps

**1. Check Workflow Status**
```
Go to n8n → Workflows → Check if toggle is Active (green)
```

**2. Verify Webhook Configuration**
```
n8n: Click "Receive DMs" node → Copy webhook URL
Slack: api.slack.com/apps → Event Subscriptions → Verify URL matches
```

**3. Test Webhook Directly**
```bash
curl -X POST "https://your-n8n.com/webhook/YOUR_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "event": {
      "type": "message",
      "channel": "D12345",
      "user": "U12345",
      "text": "test message",
      "ts": "1234567890.123456"
    },
    "type": "event_callback"
  }'
```

**Expected Response**: 200 OK

**4. Check n8n Execution History**
```
n8n → Executions (left sidebar)
Look for recent executions of "IT Support Bot Slack AI Agent"
If no executions → webhook not triggering
If executions show errors → click to see error details
```

#### Solutions

**Solution A: Webhook URL Changed**
- n8n webhook URLs can change when workflow is copied or imported
- Re-copy the current webhook URL from n8n
- Update in Slack Event Subscriptions
- Click "Retry" in Slack to verify

**Solution B: Slack Event Subscription Expired**
- Go to Slack app settings
- Event Subscriptions → Re-verify the webhook URL
- May need to reinstall app to workspace

**Solution C: Bot Not Invited to Conversation**
- In Slack, go to DMs with the bot
- Bot should appear in "Apps" section
- If not: Reinstall app from Slack app settings

**Solution D: Firewall/Network Issues**
- If self-hosted n8n, ensure port is open
- Check n8n logs for connection errors
- Verify DNS resolves correctly

---

### Problem: Bot Responds with "I Don't Have Access"

#### Symptoms
- Bot responds but says it can't access information
- Error mentions permissions or authentication

#### Solutions

**Check Slack Bot Scopes**
```
api.slack.com/apps → OAuth & Permissions → Scopes

Required scopes:
✓ chat:write
✓ channels:history
✓ im:history
✓ im:read
✓ im:write
✓ users:read
```

**If scopes changed**: Reinstall app to workspace

---

### Problem: Bot Responds to Its Own Messages (Loop)

#### Symptoms
- Bot sends message
- Bot replies to itself
- Creates infinite loop

#### Solution

Check the **"Check if Bot"** node:
```javascript
// Should filter out bot messages
const isBotMessage = $json.event.bot_id !== undefined || 
                     $json.event.subtype === 'bot_message';
return [{ json: { isBot: isBotMessage } }];
```

If filter not working:
1. Click "Check if Bot" node
2. Verify condition: `{{ $json.isBot === true }}`
3. Ensure "true" path goes to "No Operation" node

---

## 📄 Document Ingestion Issues

### Problem: Documents Not Being Indexed

#### Symptoms
- Upload file to Google Drive
- Wait 5+ minutes
- Query bot about document content
- Bot says "I don't have information about that"

#### Diagnostic Steps

**1. Check Google Drive Trigger**
```
n8n → Click "File Created" or "File Updated" node
Status should be: Active (webhook registered)
```

**2. Verify Folder ID**
```javascript
// In Google Drive trigger node
folderToWatch: {
  value: "YOUR_FOLDER_ID"  // Should match your actual folder
}
```

**3. Test Service Account Access**
```
Google Drive → Your folder → Share settings
Should show: your-service-account@project.iam.gserviceaccount.com
```

**4. Check Supabase Table**
```sql
-- Run in Supabase SQL Editor
SELECT 
  id,
  metadata->>'drive_file_id' as file_id,
  metadata->>'file_name' as name,
  created_at
FROM documents
ORDER BY created_at DESC
LIMIT 10;
```

**No results?** → Document never reached Supabase

#### Solutions

**Solution A: Service Account Permissions**
```
1. Go to Google Drive folder
2. Click "Share"
3. Add service account email
4. Set to "Editor" (not "Viewer" - needs to read metadata)
5. Click "Send"
```

**Solution B: LlamaParse API Issue**

Check LlamaParse quota:
```bash
curl https://api.cloud.llamaindex.ai/api/v1/parsing/account \
  -H "Authorization: Bearer YOUR_LLAMAPARSE_KEY"
```

Response shows daily limit and usage. Free tier: 1,000 pages/day.

**Solution C: Trigger Polling Interval**
```
Google Drive triggers poll every minute by default.
If immediate testing needed:
1. Use "Execute Workflow" for manual run
2. Or: Increase polling frequency (not recommended for production)
```

**Solution D: Re-register Webhook**
```
1. Deactivate workflow (toggle off)
2. Wait 10 seconds
3. Activate workflow (toggle on)
4. This re-registers the Google Drive webhook
```

---

### Problem: LlamaParse Fails with 429 Error

#### Symptoms
- n8n execution shows error in "Upload File" node
- Error: "429 Too Many Requests"

#### Solutions

**Check Rate Limits**
- Free tier: 1,000 pages/day
- Upgrade at cloud.llamaindex.ai for higher limits

**Adjust Wait Time**
```
In "Wait to stay within service limits" node:
Increase from 5 seconds to 10 seconds
```

**Batch Processing**
```
Add "Split In Batches" node before LlamaParse:
- Batch Size: 1
- Time Between Batches: 10 seconds
```

---

### Problem: Large PDFs Not Processing

#### Symptoms
- Small documents work fine
- PDFs over 50 pages fail or time out

#### Solutions

**Option 1: Increase Timeout**
```javascript
// In HTTP Request node (Upload File)
"timeout": 300000  // 5 minutes (300 seconds)
```

**Option 2: Split Documents**
Use a PDF splitter before LlamaParse to break into smaller chunks.

**Option 3: Upgrade LlamaParse Plan**
Higher tiers have faster processing and larger file limits.

---

## 🔍 Search & Retrieval Issues

### Problem: Bot Can't Find Relevant Information

#### Symptoms
- Documents are indexed (checked in Supabase)
- Bot responds "I don't have information about that"
- Information IS in the documents

#### Diagnostic Steps

**1. Test Vector Search Manually**
```sql
-- In Supabase SQL Editor
SELECT 
  content,
  metadata,
  similarity
FROM match_documents(
  (SELECT embedding FROM documents LIMIT 1),  -- Use a sample embedding
  0.5,  -- Lower threshold for testing
  10    -- More results
);
```

**2. Check Embedding Dimensions**
```sql
SELECT 
  vector_dims(embedding) as dimensions
FROM documents
LIMIT 1;
```

Should be **1536** for OpenAI ada-002.

#### Solutions

**Solution A: Lower Similarity Threshold**
```
In "Supabase Vector Store" retrieval node:
Change "Similarity Threshold" from 0.7 to 0.5
(Lower = more permissive matching)
```

**Solution B: Increase Top K**
```
In vector retrieval node:
Change "Top K" from 3 to 5 or 10
(Retrieves more documents for context)
```

**Solution C: Improve Chunking**
```
In "Recursive Character Text Splitter":
- Reduce Chunk Size from 1000 to 500
  (Smaller chunks = more precise matching)
- Increase Overlap from 200 to 300
  (Better context preservation)
```

**Solution D: Re-index Documents**
```sql
-- Delete existing vectors
DELETE FROM documents;

-- Re-upload documents to Google Drive
-- Workflow will re-process everything
```

---

### Problem: Bot Returns Incorrect Information

#### Symptoms
- Bot responds confidently
- Information is wrong or outdated
- Cites non-existent sources

#### Solutions

**Solution A: Add Response Validation**

In "Validate Response" code node:
```javascript
const response = $input.first().json.output;

// Check for hallucination indicators
if (response.includes("I believe") || 
    response.includes("probably") ||
    !response.includes("Source:")) {
  return [{
    json: {
      response: "I'm not confident about this information. Please contact IT at support@company.com for accurate details."
    }
  }];
}

return [{ json: { response } }];
```

**Solution B: Improve System Prompt**

In "AI Agent" node, update system message:
```
You MUST follow these rules:
1. ONLY answer using information from the search results
2. ALWAYS cite the specific document name as "Source: [filename]"
3. If the search results don't contain the answer, say "I don't have information about that in my knowledge base"
4. NEVER make up information or guess
5. If unsure, direct the user to contact IT at support@company.com
```

**Solution C: Clean Up Knowledge Base**
```
Remove outdated documents from Google Drive folder
Bot can only be as accurate as its knowledge base
```

---

## 🧠 Memory & Context Issues

### Problem: Bot Doesn't Remember Previous Messages

#### Symptoms
- Ask: "What is the WiFi password?"
- Bot answers correctly
- Ask: "What did I just ask?"
- Bot says "I don't have any previous conversation history"

#### Diagnostic Steps

**1. Check Chat Memory Table**
```sql
SELECT * FROM chat_memory
ORDER BY created_at DESC
LIMIT 20;
```

**Expected**: Recent messages from your conversation

**2. Verify PostgreSQL Connection**
```
In "Postgres Chat Memory" node → Credentials → Test connection
```

**3. Check Session ID**
```
In "Edit Fields" node, verify:
sessionId: {{ $json.event.channel }}_{{ $json.event.user }}
```

#### Solutions

**Solution A: PostgreSQL Credentials Wrong**
```
Re-enter credentials with correct:
- Host: db.xxxxx.supabase.co
- Port: 5432
- Database: postgres
- Username: postgres
- Password: YOUR_DATABASE_PASSWORD
- SSL: Enabled
```

**Solution B: Memory Not Persisting**
```javascript
// In "Set" node before AI Agent, add:
{
  "sessionId": "{{ $json.event.channel }}_{{ $json.event.user }}",
  "timestamp": "{{ $now }}"
}
```

**Solution C: Table Doesn't Exist**
```sql
-- Create the table if missing
CREATE TABLE IF NOT EXISTS chat_memory (
  id BIGSERIAL PRIMARY KEY,
  session_id TEXT NOT NULL,
  sender TEXT NOT NULL,
  message TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

### Problem: Memory Too Long / Context Window Exceeded

#### Symptoms
- Long conversations fail
- Error: "Context length exceeded"

#### Solutions

**Solution A: Limit Memory Length**

Add a Code node before memory node:
```javascript
// Keep only last 10 messages
const sessionId = $json.sessionId;

// Query database to get recent messages
const recentMessages = await getRecentMessages(sessionId, 10);

return [{ json: { messages: recentMessages } }];
```

**Solution B: Implement Conversation Summarization**
```javascript
// Every 20 messages, summarize conversation
if (messageCount > 20) {
  const summary = await summarizeConversation(sessionId);
  clearOldMessages(sessionId);
  saveContextSummary(sessionId, summary);
}
```

---

## ⚡ Performance Issues

### Problem: Bot Responses Are Slow (>10 seconds)

#### Symptoms
- User sends message
- "Typing..." indicator shows for 10+ seconds
- Eventually gets response

#### Diagnostic Steps

**Check n8n Execution Time**
```
n8n → Executions → Click on recent execution
Look at total execution time and per-node times
```

**Identify Bottlenecks:**
- LlamaParse upload: 5-30 seconds (during ingestion)
- Vector search: <1 second
- OpenAI API: 2-5 seconds
- Total should be: <8 seconds

#### Solutions

**Solution A: Reduce Vector Search Time**
```
Lower "Top K" from 10 to 3
(Fewer documents = faster search)
```

**Solution B: Use GPT-3.5 Instead of GPT-4**
```
In "OpenAI Chat Model" node:
Model: gpt-3.5-turbo (much faster, slightly less capable)
```

**Solution C: Implement Response Streaming**
```
Not currently supported in n8n's OpenAI node,
but consider using HTTP Request node with streaming enabled
```

**Solution D: Upgrade n8n Instance**
```
For n8n Cloud: Upgrade plan
For Self-hosted: Increase CPU/RAM
```

---

## 💰 Cost Management Issues

### Problem: Unexpected OpenAI Costs

#### Symptoms
- Monthly bill higher than expected
- Usage dashboard shows spikes

#### Diagnostic Steps

**Check Token Usage:**
```
OpenAI Dashboard → Usage
Look for:
- API requests/day
- Tokens/request
- Model being used (GPT-4 is 10x more expensive than GPT-3.5)
```

#### Solutions

**Solution A: Set OpenAI Usage Limits**
```
OpenAI Dashboard → Billing → Usage limits
Set hard cap: e.g., $50/month
```

**Solution B: Reduce Context Window**
```javascript
// In AI Agent, limit conversation history
const maxHistoryMessages = 5;  // Down from 10
```

**Solution C: Switch to Cheaper Embeddings**
```
Current: text-embedding-ada-002 ($0.0001/1K tokens)
Consider: Opensource alternatives with similar quality
```

**Solution D: Implement Caching**
```javascript
// Cache common queries
const commonQueries = {
  "wifi password": "The WiFi password is...",
  "reset password": "To reset your password..."
};

if (commonQueries[userQuery.toLowerCase()]) {
  return commonQueries[userQuery];
}
// Otherwise, call AI Agent
```

---

## 🔒 Security Issues

### Problem: Workflow Credentials Exposed

#### Symptoms
- Accidentally committed `.env` to GitHub
- Credentials visible in workflow JSON

#### Immediate Actions

**1. Revoke ALL Compromised Keys**
```
Slack: Regenerate OAuth token
OpenAI: Delete and create new API key
Supabase: Reset API keys
Google: Revoke service account key, create new one
LlamaParse: Regenerate API key
```

**2. Remove from Git History**
```bash
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch .env" \
  --prune-empty --tag-name-filter cat -- --all

git push origin --force --all
```

**3. Update .gitignore**
```
echo ".env" >> .gitignore
echo "credentials.json" >> .gitignore
git add .gitignore
git commit -m "Add sensitive files to gitignore"
```

**4. Monitor for Unauthorized Usage**
```
Check all service dashboards for unusual activity:
- OpenAI: Unexpected API calls
- Supabase: Unknown connections
- Slack: Unauthorized messages
```

---

## 🆘 Last Resort: Complete Reset

If nothing works, start fresh:

```bash
# 1. Backup your data
# Export Supabase documents table
# Download Google Drive files

# 2. Delete workflow in n8n

# 3. Delete all credentials in n8n

# 4. Drop and recreate Supabase tables
DROP TABLE documents;
DROP TABLE chat_memory;
# Re-run setup SQL from SETUP.md

# 5. Re-import workflow

# 6. Configure credentials from scratch

# 7. Test with simple query
```

---

## 📞 Getting Additional Help

If you're still stuck:

### Community Resources
- **n8n Community Forum**: [community.n8n.io](https://community.n8n.io)
- **n8n Discord**: [n8n.io/discord](https://n8n.io/discord)
- **GitHub Issues**: Open an issue in this repository

### Debugging Tips

**Enable Detailed Logging:**
```
In each node → Settings → Always Output Data: ON
Re-run workflow and check execution details
```

**Share Debug Info (Without Credentials):**
```
- n8n version
- Node.js version (if self-hosted)
- Workflow execution time per node
- Error messages (redact any tokens/keys)
- Steps already attempted
```

---

## 📊 Monitoring Recommendations

### Set Up Proactive Monitoring

**1. n8n Execution Alerts**
```
Create a workflow that:
- Triggers when main workflow fails
- Sends alert to Slack/Email
```

**2. Supabase Database Monitoring**
```sql
-- Check database size weekly
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**3. OpenAI Cost Tracking**
```
Set up daily/weekly alerts for:
- Total API spend
- Requests per day
- Average tokens per request
```

**4. Health Check Workflow**
```
Create a scheduled workflow that:
- Runs every 6 hours
- Tests: DB connection, API endpoints, vector search
- Reports status to monitoring dashboard
```

---

**Remember**: Most issues are credential/configuration related. Always double-check credentials first! 🔑
