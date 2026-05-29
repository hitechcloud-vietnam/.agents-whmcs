# WHMCS Slack Notifications Workflow

## Purpose
Configure Slack webhook notifications in WHMCS.

## Prerequisites
- Slack workspace admin access
- Webhook URL for channel

## Setup Steps

### Step 1: Create Slack App
1. Go to: https://api.slack.com/apps
2. Click "Create New App"
3. Select "From scratch"
4. Name app (e.g., "WHMCS Notifications")
5. Select workspace

### Step 2: Enable Incoming Webhooks
1. In left sidebar, click "Incoming Webhooks"
2. Toggle "Activate Incoming Webhooks" to On
3. Click "Add New Webhook to Workspace"
4. Select channel for notifications
5. Click "Allow"

### Step 3: Copy Webhook URL
1. Copy the webhook URL
2. Format: https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX

### Step 4: Configure in WHMCS
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Find Slack
4. Enable Slack
5. Paste webhook URL
6. Set channel name
7. Save

### Step 5: Configure Slack Template
1. Create notification
2. Select Slack channel
3. Format message for Slack
4. Use Slack markdown

## Slack Message Format

### Basic Message
```json
{
  "text": "Invoice #{$invoice_num} is overdue",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Invoice Alert*\nInvoice #{$invoice_num} for {$amount} is now overdue."
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "View Invoice"},
          "url": "{$invoice_url}"
        }
      ]
    }
  ]
}
```

### Rich Notification Example
```json
{
  "text": "New Order: {$order_num}",
  "attachments": [
    {
      "color": "#36a64f",
      "fields": [
        {"title": "Customer", "value": "{$client_name}", "short": true},
        {"title": "Amount", "value": "{$order_total}", "short": true}
      ]
    }
  ]
}
```

## Slack Variables
```
{$client_name}
{$client_email}
{$order_num}
{$order_total}
{$invoice_num}
{$alert_title}
{$alert_message}
```

## Channel Routing

### Route by Event Type
```
- #alerts: Critical notifications
- #orders: New order notifications
- #support: Ticket notifications
- #billing: Invoice notifications
```

### Route by Priority
```
- High priority: #critical-alerts
- Medium priority: #notifications
- Low priority: #general
```

## Testing

### Test Webhook
1. Click "Send Test" in WHMCS
2. Check Slack channel
3. Verify message format
4. Check links work

## Troubleshooting

### Webhook Not Working
- Verify webhook URL
- Check Slack app permissions
- Test with curl:
```bash
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Test"}' \
  YOUR_WEBHOOK_URL
```

## Related Workflows
- whmcs-notification-providers
- whmcs-notification-create
- whmcs-notification-discord