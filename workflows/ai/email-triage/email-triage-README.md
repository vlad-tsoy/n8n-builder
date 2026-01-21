# Email Inbox Triage with AI

Automatically analyze and categorize unread emails from Microsoft 365 inbox using Azure OpenAI GPT-4o.

## Files

| File | Description |
|------|-------------|
| `email-triage.json` | Main workflow JSON - import into n8n |
| `email-triage-spec.md` | Full requirements specification |
| `email-triage-prompt.md` | AI agent system prompt |

## Quick Start

### 1. Import Workflow

1. Open your n8n instance
2. Go to **Workflows** > **Add Workflow** > **Import from File**
3. Select `email-triage.json`

### 2. Create Outlook Folders

Create these folders in your Outlook inbox:
- `Urgent`
- `Action-Internal`
- `Action-External`
- `Informational`

### 3. Configure Credentials

#### Microsoft 365 / Outlook
1. In n8n, go to **Credentials** > **Add Credential** > **Microsoft Outlook OAuth2 API**
2. Follow the OAuth flow to connect your account

#### Azure OpenAI
1. In n8n, go to **Credentials** > **Add Credential** > **Azure OpenAI API**
2. Add your:
   - API Key
   - Resource Name
   - API Version (e.g., `2024-02-15-preview`)

### 4. Configure Microsoft Teams

This workflow sends digest summaries directly to your Teams chat (not a channel).

#### Add Microsoft Teams Credentials
1. In n8n, go to **Credentials** > **Add Credential** > **Microsoft Teams OAuth2 API**
2. Follow the OAuth flow to connect your Microsoft 365 account
3. Grant the required permissions for chat messaging

#### Configure the Send Node
1. Open the **Send to Teams Chat** node in the workflow
2. Select your Microsoft Teams credential
3. For **Chat**, click the dropdown and select your personal chat (usually shows as your name or "Me")
4. The message content is automatically populated from the summary generator

### 5. Update Workflow Settings

In the workflow, you may need to update:

1. **Get Unread Emails** node - Select your inbox folder
2. **Move to X Folder** nodes - Select/create the destination folders
3. **Azure OpenAI GPT-4o** node - Enter your deployment name (model name)

### 6. Test & Activate

1. Use **Manual Trigger (Catchup Mode)** to test with a few emails
2. Review the Teams message to verify categorization
3. Once working, activate the workflow for scheduled runs

## Schedule

The workflow runs:
- **8:00 AM** and **2:00 PM** on weekdays (Mon-Fri)
- Cron expression: `0 8,14 * * 1-5`

Modify in the **Schedule Trigger** node if needed.

## Catchup Mode

For your initial 1000+ email backlog:

1. Keep the schedule trigger disabled initially
2. Use **Manual Trigger (Catchup Mode)** repeatedly
3. Each run processes up to 50 emails
4. Run until backlog is cleared, then enable schedule

## Categories

| Category | Description | Folder |
|----------|-------------|--------|
| Urgent | Time-sensitive, deadlines | Urgent |
| Action-Internal | Response needed from @dairylandpower.com | Action-Internal |
| Action-External | Response needed from external | Action-External |
| Informational | FYI only, no action | Informational |

## Customization

### Change Internal Domain

In **Prepare Email Data** node, update:
```javascript
.endsWith('@dairylandpower.com')
```
to your domain.

### Modify Categories

1. Update the **AI Categorize Email** system prompt
2. Update the **Route by Category** switch conditions
3. Add/modify the folder move nodes

### Adjust Batch Size

In **Get Unread Emails** node, change the `limit` parameter (default: 50).

## Troubleshooting

### Emails not categorizing correctly

1. Check AI Agent output in execution logs
2. Adjust system prompt for edge cases
3. Ensure sender domain matching is correct

### Teams notification not sending

1. Verify Microsoft Teams OAuth2 credentials are configured
2. Check that the correct chat is selected in the Send to Teams Chat node
3. Review the Microsoft Teams node output in execution logs for errors
4. Ensure your Microsoft 365 account has permissions for chat messaging

### Folders not found

1. Ensure folders exist in Outlook first
2. In move nodes, select folders from dropdown (not by name)
