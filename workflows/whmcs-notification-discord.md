# WHMCS Discord Notifications Workflow

## Purpose
Configure Discord webhook notifications in WHMCS.

## Prerequisites
- Discord server admin access
- Create webhook in channel

## Setup Steps

### Step 1: Create Discord Webhook
1. Open Discord server
2. Click gear icon on channel
3. Select "Integrations"
4. Click "Create Webhook"
5. Name webhook (e.g., "WHMCS")
6. Copy webhook URL

### Step 2: Configure in WHMCS
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Find Discord
4. Enable Discord
5. Paste webhook URL
6. Set bot name and avatar
7. Save

### Step 3: Create Notification Template
1. Create notification
2. Select Discord channel
3. Format using Discord embeds
4. Save

## Discord Embed Format

### Basic Embed
```json
{
  "embeds": [
    {
      "title": "Invoice Alert",
      "description": "Invoice #{$invoice_num} is overdue",
      "color": 15158332,
      "fields": [
        {"name": "Amount", "value": "{$amount}", "inline": true},
        {"name": "Client", "value": "{$client_name}", "inline": true}
      ],
      "url": "{$invoice_url}"
    }
  ]
}
```

### Rich Notification
```json
{
  "embeds": [
    {
      "title": "New Order #{$order_num}",
      "description": "{$client_name} placed an order",
      "color": 3066993,
      "fields": [
        {"name": "Products", "value": "{$order_products}"},
        {"name": "Total", "value": "{$order_total}", "inline": true},
        {"name": "Date", "value": "{$order_date}", "inline": true}
      ],
      "footer": {"text": "WHMCS Notification"},
      "timestamp": "{$event_timestamp}"
    }
  ]
}
```

## Color Codes
```
15277667 - Red (Error/Danger)
3066993 - Green (Success)
3447003 - Blue (Info)
15105570 - Orange (Warning)
9807270 - Gray (Neutral)
```

## Variables
```
{$client_name}
{$client_email}
{$order_num}
{$order_total}
{$invoice_num}
{$invoice_amount}
{$alert_title}
{$alert_message}
{$event_timestamp}
```

## Advanced Features

### Role Mentions
```json
{
  "content": "<@&ROLE_ID> New alert!",
  "embeds": [...]
}
```

### Button Actions (using components)
```json
{
  "embeds": [...],
  "components": [
    {
      "type": 1,
      "components": [
        {
          "type": 2,
          "label": "View Invoice",
          "style": 5,
          "url": "{$invoice_url}"
        }
      ]
    }
  ]
}
```

## Testing

### Send Test Notification
1. Click "Send Test" in WHMCS
2. Check Discord channel
3. Verify embed format
4. Test buttons/links

## Troubleshooting

### Webhook Not Working
- Verify webhook URL
- Check channel permissions
- Test with curl:
```bash
curl -H "Content-Type: application/json" \
  -d '{"content": "Test"}' \
  YOUR_WEBHOOK_URL
```

## Related Workflows
- whmcs-notification-providers
- whmcs-notification-create
- whmcs-notification-slack