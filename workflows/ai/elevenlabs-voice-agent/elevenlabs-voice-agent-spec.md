# Workflow: ElevenLabs Voice Agent (Atlas)

## Purpose

Atlas is a conversational voice agent backend for ElevenLabs that orchestrates multiple AI tools through a central routing agent. ElevenLabs sends voice transcriptions to n8n via webhook, where the Atlas agent routes requests to specialized sub-workflows (child agents) for task execution. The system is designed for electric cooperative operations with domain-specific tools like substation visit logging.

## Architecture

```
┌─────────────────┐      Webhook POST       ┌─────────────────────────────────────┐
│   ElevenLabs    │ ──────────────────────► │           Atlas Main Agent          │
│   Voice Agent   │ ◄────────────────────── │     (gpt-4.1-mini on Azure OpenAI)  │
└─────────────────┘      JSON Response      └─────────────────┬───────────────────┘
                                                              │
                    ┌─────────────────────────────────────────┼─────────────────────────────────────────┐
                    │                    │                    │                    │                    │
                    ▼                    ▼                    ▼                    ▼                    ▼
         ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
         │ Substation Visit │ │  Content Creator │ │    Web Search    │ │   Email Agent    │ │  Calendar Agent  │
         │  Logger Agent    │ │  (Blog Pipeline) │ │   (Serper API)   │ │    (disabled)    │ │    (disabled)    │
         │  (SharePoint)    │ │  (MCP-exposed)   │ │                  │ │                  │ │                  │
         └──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘
```

## Current Workflows

### 1. Atlas (Main Orchestrator)
- **Workflow ID**: `7Ogez9TGTDjBunpx`
- **Trigger**: Webhook POST at `/webhook/atlas`
- **LLM**: Azure OpenAI `gpt-4.1-mini`
- **Response**: Responds to webhook with agent output
- **Tags**: "Voice Agent", "Jarvis"

### 2. Substation Visit Logger Agent
- **Workflow ID**: `xJFUtQN3Du56ZNTf`
- **Trigger**: Execute Workflow Trigger (sub-workflow)
- **LLM**: Azure OpenAI `gpt-4.1-nano`
- **Data Store**: Microsoft SharePoint (Substation Visits Log Demo)
- **Inputs**: `query`, `phoneNumber`

### 3. VW Blog Automation (Content Creator)
- **Workflow ID**: `e9JBsSC1AGKbHU2n`
- **Trigger**: Webhook + Chat Trigger
- **MCP Exposed**: Yes (`availableInMCP: true`)
- **Pipeline**: Researcher → Copywriter → Content Creator → Image Gen → Email
- **LLMs**: gpt-5-mini (research), o3 (copywriting), gpt-4.1 (parsing)

## Triggers

### Primary: Webhook (Current)
- **Path**: `/webhook/atlas`
- **Method**: POST
- **Authentication**: None currently (recommend adding Bearer token)
- **Headers**:
  - `phonenumber` - Caller's phone number for identification

### Future: MCP Server Trigger (Research Complete)

**n8n MCP Server Trigger** (`@n8n/n8n-nodes-langchain.mcpTrigger`):
- **Path**: `/mcp/atlas`
- **Transport**: SSE (Server-Sent Events) or HTTP Streamable
- **Authentication**: None, Bearer Auth, or Header Auth
- **Purpose**: Expose n8n tools as MCP endpoint for ElevenLabs

**ElevenLabs MCP Support** (Officially Available as of April 2025):
- Supports both SSE and HTTP streamable transport
- Configuration via MCP server integrations dashboard
- Approval control modes: Always Ask, Fine-Grained, No Approval
- **Limitation**: Not available for Zero Retention Mode or HIPAA compliance users

**MCP Configuration for ElevenLabs**:
```
Name: Atlas Voice Agent
Server URL: https://<n8n-host>/mcp/atlas
Secret Token: <bearer-token> (optional but recommended)
Transport: SSE or HTTP Streamable
```

## Inputs

### Webhook Payload from ElevenLabs
```json
{
  "headers": {
    "phonenumber": "+1234567890"
  },
  "body": {
    "query": "User's voice transcription text"
  }
}
```

## Available Tools

### Active Tools

| Tool | Type | Description | Backend |
|------|------|-------------|---------|
| `substationVisitLoggerAgent` | Sub-workflow | Log substation entry/exit | SharePoint |
| `contentCreator` | HTTP Request | Generate blog posts | VW Blog Automation workflow |
| `web_search` | HTTP Request | Search the internet | Serper API |
| `Calculator` | Built-in | Basic calculations | n8n native |

### Disabled Tools (Ready to Enable)

| Tool | Type | Description | Status |
|------|------|-------------|--------|
| `emailAgent` | Sub-workflow | Email actions | Needs workflow mapping |
| `calendarAgent` | Sub-workflow | Calendar actions | Needs workflow mapping |
| `contactAgent` | Sub-workflow | Contact lookup | Needs workflow mapping |

## Processing Flow

### Atlas Main Agent Flow
1. Receive webhook from ElevenLabs with transcribed query
2. Extract phone number from headers for caller identification
3. Route query to appropriate tool via AI Agent reasoning:
   - Substation visits → `substationVisitLoggerAgent`
   - Content creation → `contentCreator`
   - Information search → `web_search`
   - Calculations → `Calculator`
4. Execute tool and get response
5. Return response via webhook to ElevenLabs

### Substation Visit Logger Flow
1. Receive query and phone number from Atlas
2. Parse intent (entry, exit, or check visits)
3. Execute appropriate tool:
   - `entry_log` → Create SharePoint list item
   - `exit_log` → Update SharePoint list item with exit time
   - `check_visits` → Query active visits for caller
4. Return confirmation message with timestamp (CST format)

## Memory Integration (Research Complete)

### Current State
The Atlas system prompt references "Memory MCP" for Zep integration:
> "Use this tool to keep Zep state in sync for every voice call."

However, no memory node is currently connected.

### Zep Memory Node Status: DEPRECATED

**Critical Finding**: The n8n Zep Memory node (`@n8n/n8n-nodes-langchain.memoryZep`) is **deprecated** and will be removed in a future version. It only works with:
- Zep Cloud
- Zep Community Edition <= v0.27.2

**Recommendation**: Do NOT implement Zep integration. Use an alternative memory solution instead.

### Recommended Alternatives

| Memory Node | Backend | Session Key | TTL Support | Best For |
|-------------|---------|-------------|-------------|----------|
| **Postgres Chat Memory** | PostgreSQL | Custom or from trigger | No | Persistent storage, audit trails |
| **Redis Chat Memory** | Redis | Custom or from trigger | Yes (configurable) | Fast access, auto-expiring sessions |
| **MongoDB Chat Memory** | MongoDB | Custom or from trigger | No | Document-based storage |

### Recommended Implementation: Postgres Chat Memory

**Why Postgres**:
1. Already have Azure infrastructure for persistent storage
2. Auto-creates table if not exists (`n8n_chat_histories`)
3. Custom session key support (use phone number)
4. No additional infrastructure required (can use Azure Database for PostgreSQL)

**Configuration**:
```
Node: @n8n/n8n-nodes-langchain.memoryPostgresChat
Version: 1.3
Session ID Type: "Define below" (customKey)
Session Key: {{ $json.headers.phonenumber }}
Table Name: atlas_chat_histories
```

### Alternative: Redis Chat Memory

**Why Redis** (if faster access needed):
1. Session TTL support (auto-expire old conversations)
2. Faster read/write for high-volume scenarios
3. Azure Cache for Redis available

**Configuration**:
```
Node: @n8n/n8n-nodes-langchain.memoryRedisChat
Version: 1.5
Session ID Type: "Define below" (customKey)
Session Key: {{ $json.headers.phonenumber }}
Session TTL: 86400 (24 hours) or 0 (never expire)
```

### Memory Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   ElevenLabs    │────►│   Atlas Agent    │────►│ Postgres/Redis  │
│   (phonenumber) │     │   + Memory Node  │     │  Chat History   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        Session Key: phonenumber
                        Stores: user + assistant messages
                        Retrieves: conversation context
```

## Dependencies

### External Services
| Service | Purpose | Credentials |
|---------|---------|-------------|
| Azure OpenAI | LLM inference | `dpcazaichat-aillm` |
| Microsoft SharePoint | Substation visit data | SharePoint OAuth |
| Serper API | Web search | Header auth |
| Brave Search | News search | API key |
| Azure Blob Storage | Image storage | Shared key |
| Microsoft Outlook | Email delivery | OAuth |
| PostgreSQL (recommended) | Conversation memory | Connection string |
| Redis (alternative) | Conversation memory | Connection string |

### Azure OpenAI Deployments
| Model | Use Case |
|-------|----------|
| `gpt-4.1-mini` | Atlas main agent |
| `gpt-4.1-nano` | Substation logger |
| `gpt-5-mini` | Research, topic ID, image prompts |
| `o3` | Copywriting |
| `gpt-4.1` | Structured output parsing |
| `gpt-image-1` | Image generation |

## Error Handling

### Current Implementation
- **Substation Logger**: Has error output path → "Try Again" response
- **Content Creator**: Retry on fail enabled
- **Blog Pipeline**: Multiple agents with `onError: continueErrorOutput`

### Recommended Enhancements
1. Graceful fallback messages for voice agent
2. Teams/email notifications for critical failures
3. Retry with exponential backoff for transient errors

## Constraints

### Timezone
- All timestamps in CST (UTC-06:00)
- Timezone configured in workflow settings: `America/Chicago`

### Security (Current Gaps)
- [ ] Webhook has no authentication - **HIGH PRIORITY**
- [ ] API keys in workflow JSON should move to n8n credentials
- [ ] Phone number passed in headers (ensure HTTPS)

### Performance
- Tool call latency should be < 2 seconds for voice UX
- Content creation is async (delivered via email)

## Test Scenarios

### Substation Entry
```bash
curl -X POST https://<n8n-host>/webhook/atlas \
  -H "Content-Type: application/json" \
  -H "phonenumber: +1234567890" \
  -d '{"query": "I am entering substation Echo-5 for routine maintenance"}'
```

Expected: Entry logged with timestamp, confirmation returned

### Substation Exit
```bash
curl -X POST https://<n8n-host>/webhook/atlas \
  -H "Content-Type: application/json" \
  -H "phonenumber: +1234567890" \
  -d '{"query": "I am leaving the substation now"}'
```

Expected: Exit time updated, confirmation with entry/exit times

### Content Creation
```bash
curl -X POST https://<n8n-host>/webhook/atlas \
  -H "Content-Type: application/json" \
  -d '{"query": "Write an article about grid modernization"}'
```

Expected: Immediate acknowledgment, article delivered via email later

## Enhancement Roadmap (Updated with Research)

### Phase 1: Security & Stability
- [ ] Add Bearer token authentication to webhook
- [ ] Move API keys to n8n credentials
- [ ] Add comprehensive error handling
- [ ] Enable Teams notifications for failures

### Phase 2: Memory Integration (REVISED - Zep Deprecated)
- [ ] **DO NOT use Zep** - node is deprecated
- [ ] Provision Azure Database for PostgreSQL (or Azure Cache for Redis)
- [ ] Add Postgres Chat Memory node to Atlas workflow
- [ ] Configure session key using `phonenumber` header
- [ ] Test conversation persistence across calls
- [ ] Update Atlas system prompt to remove Zep references

### Phase 3: Tool Expansion
- [ ] Enable and configure Email Agent
- [ ] Enable and configure Calendar Agent
- [ ] Add customer lookup tool (CRM integration)
- [ ] Add knowledge base search (Supabase Vector Store or Pinecone)

### Phase 4: MCP Migration (OPTIONAL - Research Complete)

**Recommendation**: MCP migration is **optional** but provides benefits for complex tool orchestration.

**Current Webhook Advantages**:
- Simpler architecture
- Direct request/response pattern
- Already working

**MCP Server Trigger Advantages**:
- Built-in tool schema discovery
- ElevenLabs native MCP support
- Fine-grained approval controls
- Better for multi-tool scenarios

**Migration Decision Criteria**:
- If adding 5+ tools → Consider MCP
- If tools need approval workflows → Use MCP
- If keeping it simple → Stay with Webhook

**If Migrating to MCP**:
- [ ] Create MCP Server Trigger workflow at `/mcp/atlas`
- [ ] Configure Bearer Auth credentials
- [ ] Expose tools via toolWorkflow nodes
- [ ] Configure ElevenLabs MCP integration dashboard
- [ ] Test with "Always Ask" approval mode first
- [ ] Gradually enable "Fine-Grained" approval for trusted tools

## Comparison: Webhook vs MCP

| Aspect | Current Webhook | MCP Server Trigger |
|--------|-----------------|-------------------|
| Transport | HTTP POST | SSE / HTTP Streamable |
| Authentication | None (needs adding) | Bearer Auth / Header Auth |
| Tool Discovery | Manual (in prompt) | Automatic schema |
| Approval Control | N/A | Always Ask / Fine-Grained / None |
| Complexity | Low | Medium |
| ElevenLabs Support | Via custom tools | Native integration |
| Best For | Simple queries | Complex tool orchestration |

## Files

| File | Purpose |
|------|---------|
| `Atlas.json` | Main orchestrator workflow |
| `Substation Visit Logger Agent (1).json` | Substation logging sub-workflow |
| `VW Blog Automation LinkedIn.json` | Content creation pipeline |
| `elevenlabs-voice-agent-spec.md` | This specification |
| `elevenlabs-voice-agent-README.md` | Setup instructions |
