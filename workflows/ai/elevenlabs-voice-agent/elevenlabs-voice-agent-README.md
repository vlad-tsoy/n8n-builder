# ElevenLabs Voice Agent (Atlas)

Voice agent backend for ElevenLabs that routes requests to specialized AI tools for electric cooperative operations.

## Quick Start

### n8n Instance
- **Host**: Azure Container Apps
- **URL**: `https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io`

### Webhook Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/webhook/atlas` | Main voice agent entry point |
| `/webhook/0422ccfe-788c-4236-a81d-b90d81699cc4` | Content creator (blog pipeline) |

## Architecture

```
ElevenLabs → Webhook → Atlas Agent → Child Tools → Response
                           │
                           ├── Substation Logger (SharePoint)
                           ├── Content Creator (Blog Pipeline)
                           ├── Web Search (Serper API)
                           └── Calculator
```

## Existing Workflows

### Atlas (Main Orchestrator)
- **ID**: `7Ogez9TGTDjBunpx`
- **Status**: Active
- **LLM**: Azure OpenAI `gpt-4.1-mini`

Routes incoming voice queries to appropriate tools based on intent.

### Substation Visit Logger
- **ID**: `xJFUtQN3Du56ZNTf`
- **Status**: Active
- **LLM**: Azure OpenAI `gpt-4.1-nano`
- **Data Store**: SharePoint (Substation Visits Log Demo)

Logs entry/exit times for substation visits with timezone-aware timestamps (CST).

### VW Blog Automation
- **ID**: `e9JBsSC1AGKbHU2n`
- **Status**: Active
- **MCP Exposed**: Yes

Multi-agent content pipeline: Research → Write → Generate Image → Email.

## Credentials Required

| Credential | Node | Purpose |
|------------|------|---------|
| `Atlas Webhook Auth` | Webhook | Bearer token authentication |
| `dpcazaichat-aillm` | Azure OpenAI | LLM inference |
| `Azure OpenAI Image Gen API` | Create Image (HTTP Request) | Image generation API auth (NEW) |
| `Microsoft SharePoint account` | SharePoint | Substation data |
| `Serper API` | HTTP Request Tool | Web search |
| `Brave Search account` | Brave Search | News search |
| `n8nstprodkkn0` | Azure Storage | Image storage |
| `vladimir.tsoy@dairylandpower.com` | Outlook | Email delivery |

### Creating Azure OpenAI Image Gen API Credential

1. Go to **Credentials** → **Add Credential** → **Header Auth**
2. Configure:
   - **Name**: `Azure OpenAI Image Gen API`
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer <your-azure-openai-api-key>`
3. Save the credential

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `N8N_BASE_URL` | Base URL for internal webhook calls | `https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io` |

## Testing

> **Note**: All requests require the `Authorization: Bearer <token>` header after authentication is configured.

### Test Substation Entry
```bash
curl -X POST https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io/webhook/atlas \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-atlas-webhook-token>" \
  -H "phonenumber: +1234567890" \
  -d '{"query": "I am entering substation Echo-5"}'
```

### Test Web Search
```bash
curl -X POST https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io/webhook/atlas \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-atlas-webhook-token>" \
  -d '{"query": "What is the current weather in Wisconsin?"}'
```

### Test Content Creation
```bash
curl -X POST https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io/webhook/atlas \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-atlas-webhook-token>" \
  -d '{"query": "Write a blog post about renewable energy trends"}'
```

## ElevenLabs Configuration

Configure your ElevenLabs agent to call the webhook:

```
Webhook URL: https://n8n-ca-prod-kkn0-n8n.purplesmoke-e39cf5ad.eastus2.azurecontainerapps.io/webhook/atlas
Method: POST
Headers:
  - Content-Type: application/json
  - Authorization: Bearer <your-atlas-webhook-token>  # REQUIRED for authentication
  - phonenumber: {{caller_phone_number}}
Body:
  {
    "query": "{{transcribed_text}}"
  }
```

### Authentication Setup

#### Step 1: Generate a Secure Token

Choose one of these methods:

**Option A - OpenSSL (recommended):**
```bash
openssl rand -base64 32
# Output: K7xP2mN9qR4sT6vW8yB1cD3eF5gH7jL9mN0pQ2rS4tU=
```

**Option B - Python:**
```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
# Output: aB3cD4eF5gH6iJ7kL8mN9oP0qR1sT2uV3wX4yZ5
```

**Option C - UUID (simpler, less entropy):**
```bash
uuidgen
# Output: 550e8400-e29b-41d4-a716-446655440000
```

> **Security Note**: Use at least 32 characters. Store the token securely - you'll need it for both n8n and ElevenLabs configuration.

#### Step 2: Create Credential in n8n

1. Go to **Credentials** → **Add Credential** → **Header Auth**
2. Configure:
   - **Name**: `Atlas Webhook Auth`
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer <your-generated-token>`

   Example: `Bearer K7xP2mN9qR4sT6vW8yB1cD3eF5gH7jL9mN0pQ2rS4tU=`

3. Save the credential

#### Step 3: Configure Atlas Webhook

1. Open the Atlas workflow
2. Click on the **Webhook** node
3. Under **Authentication**, select **Header Auth**
4. Select the `Atlas Webhook Auth` credential
5. Save the workflow

#### Step 4: Configure ElevenLabs

In your ElevenLabs agent webhook settings, add the header:
```
Authorization: Bearer <your-token>
```

Use the same token value (without the "Bearer " prefix in the ElevenLabs field if it adds it automatically, or with the full value if it expects the complete header).

## Planned Enhancements

### Phase 1: Security (Priority) ✅
- [x] Add Bearer token authentication to Atlas webhook
- [x] Secure API keys in n8n credentials (Serper API moved to credentials)
- [x] Parameterize hardcoded webhook URLs (uses `$env.N8N_BASE_URL`)

### Phase 2: Memory (Postgres/Redis - Zep is DEPRECATED)

> **Important**: The n8n Zep Memory node is deprecated and should NOT be used.
> Use Postgres Chat Memory or Redis Chat Memory instead.

- [ ] Provision Azure Database for PostgreSQL
- [ ] Add Postgres Chat Memory node to Atlas
- [ ] Configure session key using `phonenumber` header
- [ ] Test conversation persistence

### Phase 3: Tool Expansion
- [ ] Enable Email Agent
- [ ] Enable Calendar Agent
- [ ] Add customer lookup

### Phase 4: MCP Migration (Optional)

> **Research Complete**: ElevenLabs officially supports MCP (as of April 2025).
> Migration is optional - webhook is simpler, MCP is better for complex tool orchestration.

- [ ] Create MCP Server Trigger at `/mcp/atlas`
- [ ] Configure in ElevenLabs MCP integrations dashboard
- [ ] Test with approval controls

## Files

| File | Purpose |
|------|---------|
| `Atlas.json` | Main orchestrator |
| `Substation Visit Logger Agent (1).json` | Substation tool |
| `VW Blog Automation LinkedIn.json` | Content pipeline |
| `elevenlabs-voice-agent-spec.md` | Detailed specification |
| `elevenlabs-voice-agent-README.md` | This file |

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| No response from webhook | Check workflow is active in n8n |
| SharePoint errors | Verify OAuth token not expired |
| Tool not found | Check sub-workflow IDs in Atlas |
| Slow response | Check Azure OpenAI quotas |

### Debug Mode

Enable execution logging in n8n to see full request/response flow.

## Related Documentation

- [n8n AI Agent Documentation](https://docs.n8n.io/integrations/langchain/)
- [n8n MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/)
- [ElevenLabs Conversational AI](https://elevenlabs.io/docs/conversational-ai)
- [ElevenLabs MCP Integration](https://elevenlabs.io/docs/agents-platform/customization/tools/mcp)
- [ElevenLabs MCP Server (GitHub)](https://github.com/elevenlabs/elevenlabs-mcp)
