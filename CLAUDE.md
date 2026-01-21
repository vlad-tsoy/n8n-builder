# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This project builds and manages n8n workflows programmatically using the n8n MCP (Model Context Protocol) tools available through Docker MCP. The connected n8n instance is hosted on Azure Container Apps.

## Workflow Organization

Each workflow is stored in its own folder within a category folder for clean separation of artifacts.

### Folder Structure
```
workflows/
├── <category>/
│   └── <workflow-name>/
│       ├── <workflow-name>.json        # Main workflow definition
│       ├── <workflow-name>-spec.md     # Requirements specification
│       ├── <workflow-name>-prompt.md   # AI agent system prompt (if applicable)
│       ├── <workflow-name>-README.md   # Setup and usage instructions
│       └── <workflow-name>-schema.json # JSON schemas (if applicable)
```

### Categories
- `productivity` - Task management, scheduling, reminders
- `automation` - Process automation, data sync
- `communication` - Slack, email, notifications
- `integration` - API connections, data pipelines
- `ai` - AI agents, LLM workflows, chatbots
- `monitoring` - Alerts, health checks, logging
- `data` - ETL, transformations, reporting

**Example**: A Slack notification automation:
```
workflows/communication/slack-alerts/
├── slack-alerts.json
├── slack-alerts-spec.md
└── slack-alerts-README.md
```

## Workflow Specification Process

**Every new workflow MUST have a spec.md file created through a requirements interview.**

### Creating a New Workflow Spec

Before building any workflow, conduct a thorough requirements interview using the `AskUserQuestion` tool:

1. **Start the interview** - Use `AskUserQuestion` to gather requirements iteratively
2. **Ask in-depth questions** - Cover all aspects of the workflow:
   - **Trigger**: What initiates this workflow? (webhook, schedule, event, manual)
   - **Data sources**: Where does input data come from? What format? What volume?
   - **Processing logic**: What transformations, conditions, or business rules apply?
   - **Error handling**: What happens on failure? Retry logic? Fallback behavior?
   - **Output/Actions**: What should the workflow produce or trigger?
   - **Authentication**: What services need credentials? OAuth, API keys, tokens?
   - **Rate limits**: Are there API rate limits to consider? Throttling needs?
   - **Idempotency**: Should repeated executions produce the same result?
   - **State management**: Does the workflow need to track state between runs?
   - **Testing strategy**: How will this be tested? Test data available?
   - **Monitoring**: What metrics matter? Alerting thresholds?
   - **Security**: Sensitive data handling? PII considerations?
   - **Dependencies**: External services? Failure modes if they're down?
   - **Scale**: Expected execution frequency? Peak load scenarios?
   - **Edge cases**: What unusual inputs or conditions might occur?

3. **Continue until complete** - Keep asking follow-up questions until all requirements are clear
4. **Create workflow folder** - Create the workflow's dedicated folder:
   ```
   workflows/<category>/<workflow-name>/
   ```
5. **Write the spec** - Save the completed specification to:
   ```
   workflows/<category>/<workflow-name>/<workflow-name>-spec.md
   ```

### Spec File Format

The spec.md file should contain:
```markdown
# Workflow: <Name>

## Purpose
<One paragraph description>

## Trigger
<What starts this workflow>

## Inputs
<Data sources, formats, schemas>

## Processing Steps
<Ordered list of operations>

## Outputs
<What the workflow produces>

## Error Handling
<Failure modes and recovery>

## Dependencies
<External services, credentials needed>

## Constraints
<Rate limits, security, performance requirements>

## Test Scenarios
<How to verify correct behavior>
```

## Workflow Artifacts

All artifacts for a workflow are stored in its dedicated folder:

### Standard Artifacts
| File | Purpose |
|------|---------|
| `<name>.json` | Main workflow definition |
| `<name>-spec.md` | Requirements specification (required) |
| `<name>-README.md` | Setup and usage instructions |
| `<name>-prompt.md` | AI agent system prompt (for AI workflows) |
| `<name>-schema.json` | JSON schemas for structured output |
| `<name>-template.md` | Email/message templates |
| `<name>-config.json` | Environment-specific configuration |

**Example**: An AI customer support workflow:
```
workflows/ai/customer-support/
├── customer-support.json
├── customer-support-spec.md
├── customer-support-README.md
├── customer-support-prompt.md
└── customer-support-schema.json
```

## n8n MCP Tools Architecture

All n8n operations are performed through MCP tools with the `mcp__MCP_DOCKER__` prefix. There are no local build/test commands - this is a workflow orchestration project.

### Tool Categories

**Discovery** - Find nodes and understand capabilities:
- `search_nodes({query: "slack"})` - Full-text search across 525 nodes
- `list_nodes({category: "trigger"})` - List by category (trigger, transform, output, input, AI)
- `list_ai_tools()` - List 263 AI-capable nodes

**Configuration** - Get node schemas and properties:
- `get_node_essentials("nodes-base.slack")` - **ALWAYS call first** (5KB, shows required fields)
- `get_node_info("nodes-base.slack")` - Complete schema (100KB+, use only if essentials insufficient)
- `search_node_properties("nodes-base.slack", "auth")` - Find specific properties

**Validation** - Check configurations before deployment:
- `validate_node_minimal("nodes-base.slack", config)` - Quick required fields check
- `validate_node_operation("nodes-base.slack", config)` - Full validation with fixes
- `validate_workflow(workflow)` - Complete workflow validation

**Workflow Management** - CRUD operations on n8n instance:
- `n8n_create_workflow({name, nodes, connections})` - Creates workflow (inactive by default)
- `n8n_update_partial_workflow({id, operations})` - Surgical diff updates (preferred)
- `n8n_update_full_workflow({id, nodes, connections})` - Full replacement
- `n8n_get_workflow(id)` - Retrieve workflow
- `n8n_list_workflows()` - List all workflows
- `n8n_validate_workflow({id})` - Validate existing workflow
- `n8n_autofix_workflow({id})` - Auto-fix common errors

**Execution** - Trigger and monitor workflows:
- `n8n_trigger_webhook_workflow({webhookUrl, data})` - Trigger via webhook
- `n8n_get_execution({id, mode: "preview"})` - Get execution details
- `n8n_list_executions({workflowId})` - List executions

**Templates** - Pre-built workflow patterns:
- `search_templates("chatbot")` - Search by keyword
- `get_template(templateId)` - Get complete workflow JSON
- `list_node_templates(["n8n-nodes-base.httpRequest"])` - Find templates using specific nodes

## Critical Patterns

### Node Type Prefixes
All node types require package prefix:
- Core nodes: `n8n-nodes-base.webhook`, `n8n-nodes-base.httpRequest`, `n8n-nodes-base.slack`
- AI nodes: `@n8n/n8n-nodes-langchain.openAi`, `@n8n/n8n-nodes-langchain.agent`

### Workflow Creation Structure

**IMPORTANT**: The `connections` object references nodes by their **`name`** property, NOT by their `id`.

```javascript
n8n_create_workflow({
  name: "Workflow Name",
  nodes: [
    {
      id: "unique_id",           // Internal identifier
      name: "Display Name",      // Used in connections!
      type: "n8n-nodes-base.webhook",
      typeVersion: 1,
      position: [250, 300],
      parameters: { httpMethod: "POST", path: "my-webhook" }
    },
    {
      id: "another_id",
      name: "Process Data",
      type: "n8n-nodes-base.set",
      typeVersion: 3.4,
      position: [500, 300],
      parameters: { ... }
    }
  ],
  connections: {
    "Display Name": {            // <-- Use node NAME, not id
      "main": [[{ node: "Process Data", type: "main", index: 0 }]]  // <-- Target by NAME
    }
  }
})
```

### Partial Update Operations
Use `n8n_update_partial_workflow` for surgical changes:
- `addNode`, `removeNode`, `updateNode`, `moveNode`, `enableNode`, `disableNode`
- `addConnection`, `removeConnection`, `rewireConnection`, `cleanStaleConnections`
- `updateSettings`, `updateName`, `addTag`, `removeTag`
- `activateWorkflow`, `deactivateWorkflow`

### Smart Connection Parameters for IF/Switch Nodes
```javascript
// IF node - use branch instead of sourceIndex
{type: "addConnection", source: "IF", target: "Handler", branch: "true"}  // or "false"

// Switch node - use case instead of sourceIndex
{type: "addConnection", source: "Switch", target: "Handler", case: 0}  // case number
```

### AI Connection Types
For AI Agent workflows, specify `sourceOutput`:
- `ai_languageModel` - Connect LLMs to AI Agents
- `ai_tool` - Connect tools to AI Agents
- `ai_memory` - Connect memory systems
- `ai_embedding` - Connect embedding models
- `ai_vectorStore` - Connect vector stores

```javascript
{type: "addConnection", source: "OpenAI", target: "AI Agent", sourceOutput: "ai_languageModel"}
```

### Property Removal
Use `undefined` to remove properties (not `null`):
```javascript
{type: "updateNode", nodeName: "HTTP Request", updates: {continueOnFail: undefined, onError: "continueErrorOutput"}}
```

## Workflow Best Practices

1. **Always validate before deployment**: Use `validate_workflow` or `validateOnly: true`
2. **Get essentials first**: Call `get_node_essentials` before `get_node_info`
3. **Prefer partial updates**: Use `n8n_update_partial_workflow` over full updates
4. **Include intent**: Add `intent` parameter to partial updates for better responses
5. **Clean stale connections**: Run `cleanStaleConnections` after node renames/deletions
6. **Use atomic mode**: Default behavior ensures all operations succeed or none apply
7. **Connect AI models first**: Language model must connect before AI Agent is enabled

## Documentation Access

Get detailed tool documentation:
```javascript
tools_documentation({topic: "tool_name", depth: "full"})
tools_documentation({topic: "ai_agents_guide"})  // AI workflow patterns
tools_documentation({topic: "javascript_code_node_guide"})  // Code node help
```

## Publishing Workflows to n8n

After developing a workflow locally, publish it to the connected n8n instance.

### Method 1: Direct MCP Publishing (Preferred)

Use the MCP tools to create or update workflows directly:

```javascript
// Create new workflow
n8n_create_workflow({
  name: "My Workflow",
  nodes: [...],
  connections: {...}
})

// Update existing workflow by ID
n8n_update_partial_workflow({
  id: "workflow_id",
  operations: [...]
})
```

**Post-creation steps:**
1. Workflow is created **inactive** by default
2. Configure credentials in n8n UI (credentials cannot be set via API)
3. Test with manual trigger before activating
4. Use `n8n_update_partial_workflow` with `activateWorkflow` operation to enable

### Method 2: Manual Import (Fallback)

If MCP creation fails, import the JSON file manually:

1. Open n8n instance in browser
2. Go to **Workflows** > **Add Workflow** > **Import from File**
3. Select the `<workflow-name>.json` file from `workflows/<category>/<workflow-name>/`
4. Configure credentials for each node that requires authentication
5. Test and activate

### Post-Publish Checklist

After publishing any workflow:

- [ ] Configure all required credentials (OAuth, API keys)
- [ ] Create any referenced folders (e.g., Outlook folders for email workflows)
- [ ] Set environment variables if workflow uses `$env.*` expressions
- [ ] Test with manual trigger using sample data
- [ ] Verify error handling paths work correctly
- [ ] Activate workflow for scheduled/webhook triggers
- [ ] Document the workflow ID in the local README for future reference

### Workflow Activation

```javascript
// Activate after testing
n8n_update_partial_workflow({
  id: "workflow_id",
  operations: [{type: "activateWorkflow"}]
})

// Deactivate for maintenance
n8n_update_partial_workflow({
  id: "workflow_id",
  operations: [{type: "deactivateWorkflow"}]
})
```

## Connected Instance

- **API URL**: Configured via environment variables (N8N_API_URL, N8N_API_KEY)
- **Health Check**: `n8n_health_check()` to verify connectivity
- **Diagnostics**: `n8n_diagnostic({verbose: true})` for troubleshooting

## Additional MCP Tools

Beyond Docker MCP for n8n operations, this project has access to Firecrawl MCP and Playwright MCP for web research and testing.

### Firecrawl MCP - Web Scraping & Search

Use Firecrawl for web content extraction and search. Tools have `mcp__firecrawl__` prefix.

**When to use**:
- Researching documentation, APIs, or external services for workflow integration
- Extracting structured data from websites for workflow requirements
- Finding up-to-date information about n8n nodes, integrations, or third-party APIs

**Primary Tools**:

| Tool | Use Case | Example |
|------|----------|---------|
| `firecrawl_search` | Web search with optional scraping | Research an API's webhook format |
| `firecrawl_scrape` | Extract content from a single URL | Get documentation page content |
| `firecrawl_map` | Discover all URLs on a site | Find all endpoints in an API docs site |
| `firecrawl_extract` | Extract structured data with schema | Pull pricing info, feature lists |

**Best Practices**:
- Use `firecrawl_search` without `formats` first, then `firecrawl_scrape` specific pages
- Add `maxAge` parameter for faster cached responses
- Use `firecrawl_map` before `firecrawl_crawl` to discover URLs
- Prefer `firecrawl_agent` for complex multi-source research tasks

```javascript
// Research an integration
firecrawl_search({query: "ElevenLabs n8n webhook integration", limit: 5})

// Get specific documentation
firecrawl_scrape({url: "https://docs.example.com/webhooks", formats: ["markdown"]})

// Extract structured data
firecrawl_extract({
  urls: ["https://example.com/pricing"],
  prompt: "Extract pricing tiers and features",
  schema: {type: "object", properties: {tiers: {type: "array"}}}
})
```

### Playwright MCP - Browser Automation & Testing

Use Playwright for testing webhooks, inspecting web UIs, and verifying workflow outputs. Tools have `mcp__playwright__` prefix.

**When to use**:
- Testing webhook endpoints that require browser interaction
- Verifying workflow outputs in web interfaces (n8n UI, connected apps)
- Debugging OAuth flows or web-based authentication
- Capturing screenshots for documentation

**Primary Tools**:

| Tool | Use Case |
|------|----------|
| `browser_navigate` | Open a URL |
| `browser_snapshot` | Get accessibility tree (better than screenshot for actions) |
| `browser_click` | Click elements by ref from snapshot |
| `browser_type` | Type into form fields |
| `browser_take_screenshot` | Capture visual state |
| `browser_console_messages` | Check for JavaScript errors |
| `browser_network_requests` | Inspect API calls made by page |

**Best Practices**:
- Always use `browser_snapshot` before interacting - it provides element refs
- Use `browser_snapshot` over `browser_take_screenshot` for actionable context
- Close browser with `browser_close` when done
- Check `browser_console_messages` for errors after page loads

```javascript
// Test a workflow webhook UI
browser_navigate({url: "https://n8n-instance/webhook/test-path"})
browser_snapshot({})  // Get element refs
browser_type({element: "Query input", ref: "input1", text: "test query"})
browser_click({element: "Submit button", ref: "button1"})
browser_take_screenshot({filename: "webhook-response.png"})
```

### Tool Selection Guide

| Task | Primary Tool | Fallback |
|------|--------------|----------|
| Research external APIs/docs | Firecrawl search/scrape | WebFetch |
| Test webhook responses | Playwright browser | curl via Bash |
| Verify n8n workflow output | Playwright + n8n MCP | n8n execution API |
| Extract structured web data | Firecrawl extract | Firecrawl scrape + parse |
| Debug OAuth/login flows | Playwright browser | Manual testing |
| Check real-time data | Firecrawl with maxAge:0 | WebSearch |
