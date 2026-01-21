# Workflow: Email Inbox Triage with AI

## Purpose
Automatically analyze and categorize unread emails from Microsoft 365 inbox using Azure OpenAI GPT-5.2. Emails are sorted into actionable folders and a summary digest is sent to Microsoft Teams, helping manage inbox overload efficiently.

## Trigger
- **Type**: Scheduled (Cron)
- **Frequency**: Twice daily (morning and afternoon)
- **Catchup Mode**: Manual execution for processing backlog of 1000+ emails, 50 emails per run

## Inputs
- **Source**: Microsoft 365 / Outlook inbox via Microsoft Graph API
- **Filter**: Unread emails only
- **Batch Size**: 50 emails per execution
- **Data Used**: Subject line and email body (attachments excluded)

## Categories
The AI will classify each email into one of four categories:

1. **Urgent**: Emails requiring immediate attention. Any of these criteria trigger urgent:
   - Marked with HIGH importance flag
   - Contains explicit deadlines
   - Time-sensitive language or requests
   - Urgent or critical matters
2. **Action-Internal**: Emails from @dairylandpower.com requiring a response or action (not urgent)
3. **Action-External**: Emails from outside dairylandpower.com requiring a response or action (not urgent)
4. **Informational**: Newsletters, FYIs, updates, system-generated emails - no action required

### Special Rule: System-Generated Emails
Automated emails that don't require action are always categorized as **Informational**, including:
- Calendar updates and meeting notifications
- Delivery receipts and read receipts
- Out-of-office replies
- Subscription confirmations
- System alerts that are FYI only

## Processing Steps
1. **Trigger**: Schedule fires or manual execution
2. **Fetch Emails**: Get up to 50 unread emails from Outlook inbox
3. **Loop Through Emails**: Process each email individually
4. **AI Analysis**: For each email:
   - Extract sender domain
   - Send subject + body to Azure OpenAI GPT-5.2
   - Prompt AI to categorize and summarize
   - AI returns: category, confidence, summary
5. **Handle Uncertainty**: If AI confidence is low, skip email (leave in inbox)
6. **Move to Folder**: Move categorized emails to appropriate Outlook folder
7. **Aggregate Results**: Collect all categorizations for summary
8. **Generate Digest**: AI creates comprehensive summary of processed emails
9. **Send to Teams**: Send digest to user's Microsoft Teams chat

## Outputs

### Outlook Folders (created if not exist)
- `Inbox/Action-Internal`
- `Inbox/Action-External`
- `Inbox/Informational`
- `Inbox/Urgent`

### Microsoft Teams Chat Message
Full digest sent directly to user's Teams chat (not a channel) including:
- Count per category
- List of urgent items with sender and subject
- List of action-required items (internal and external)
- HTML formatting for readability

> Uses n8n Microsoft Teams node with chatMessage resource for direct delivery to user.

### Email Handling
- Emails moved to categorized folders
- **NOT** marked as read (preserved for user review)
- Uncertain categorizations left in inbox untouched

## Error Handling
| Scenario | Behavior |
|----------|----------|
| AI categorization uncertain | Skip email, leave in inbox |
| Microsoft Graph API error | Retry up to 3 times, then log error |
| Azure OpenAI timeout | Skip email, continue with next |
| Teams webhook failure | Log error, continue processing |
| Empty inbox | Send "No unread emails" message to Teams |

## Dependencies

### Credentials Required (Pre-configured)
- **Microsoft 365**: OAuth2 credentials for Outlook access
- **Azure OpenAI**: API key and endpoint for GPT-5.2
- **Microsoft Teams**: OAuth2 credentials for Teams chat messaging

### External Services
- Microsoft Graph API (Outlook)
- Azure OpenAI Service
- Microsoft Teams Graph API (Chat)

## Constraints

### Rate Limits
- Microsoft Graph: 10,000 requests per 10 minutes (not a concern at 50 emails)
- Azure OpenAI: Varies by deployment, typically 60 RPM for GPT-5.2
- Processing includes 1-second delays between emails to avoid throttling

### Performance
- Expected processing time: 2-5 minutes per 50-email batch
- AI analysis: ~2-3 seconds per email

### Security
- Email content sent to Azure OpenAI (within Azure tenant)
- No PII stored in workflow logs
- Teams OAuth credentials managed securely via n8n credential store

## Test Scenarios

1. **Happy Path**: 10 unread emails, mix of categories → all categorized and moved
2. **Empty Inbox**: No unread emails → Teams message says "No emails to process"
3. **Mixed Confidence**: Some emails unclear → uncertain ones stay in inbox
4. **API Timeout**: Azure OpenAI slow → email skipped, others continue
5. **Bulk Processing**: 50 emails in catchup mode → all processed within timeout
6. **Internal vs External**: Emails from @dairylandpower.com → Action-Internal folder

## Configuration Values

```json
{
  "internalDomain": "dairylandpower.com",
  "batchSize": 50,
  "folders": {
    "internal": "Action-Internal",
    "external": "Action-External",
    "informational": "Informational",
    "urgent": "Urgent"
  },
  "schedule": "0 8,14 * * 1-5"
}
```

## Microsoft Teams Chat Setup

User needs to:
1. In n8n, add **Microsoft Teams OAuth2 API** credentials
2. Authenticate with Microsoft 365 account
3. In the **Send to Teams Chat** node, select the target chat
4. The workflow sends HTML-formatted digest messages directly to the user's chat
