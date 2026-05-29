# WHMCS Notification Providers Workflow

## Purpose
Configure notification providers for delivering alerts.

## Available Providers

### 1. Email Notifications
- Built-in provider
- No additional setup
- Uses WHMCS email system

### 2. Admin Notification Center
- In-app notifications
- Admin dashboard
- Notification bell icon

### 3. Slack
1. Create Slack App
2. Enable incoming webhooks
3. Get webhook URL
4. Configure in WHMCS

### 4. Discord
1. Create Discord server
2. Create webhook in channel
3. Get webhook URL
4. Configure in WHMCS

### 5. Telegram
1. Create Telegram bot
2. Get bot token
3. Create channel/group
4. Add bot to channel
5. Get chat ID

### 6. Webhooks
- Custom HTTP endpoints
- JSON payload
- Authentication support

### 7. SMS (Third-party)
- Twilio
- Nexmo/Vonage
- Configure API credentials

## Configuration Steps

### Slack Setup
1. Go to: https://api.slack.com/apps
2. Create new app
3. Enable Incoming Webhooks
4. Add webhook to workspace
5. Copy webhook URL
6. Paste into WHMCS

### Discord Setup
1. Server Settings > Webhooks
2. Create webhook
3. Name it (e.g., "WHMCS Alerts")
4. Copy webhook URL
5. Paste into WHMCS

### Telegram Setup
1. Start chat with @BotFather
2. Create new bot
3. Copy bot token
4. Create channel/group
5. Add bot as admin
6. Get chat ID via @userinfobot

### Webhook Configuration
```
Webhook URL: https://your-app.com/webhook
Method: POST
Headers:
  - Content-Type: application/json
  - Authorization: Bearer token
Payload Format:
{
  "event": "{$event_type}",
  "data": {$event_data},
  "timestamp": "{$date}"
}
```

## Provider Settings

### Enable/Disable Providers
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Toggle providers on/off

### Test Providers
1. Click "Test" next to provider
2. Send test notification
3. Verify delivery

## Related Workflows
- whmcs-notification-slack
- whmcs-notification-discord
- whmcs-notification-telegram
- whmcs-notification-webhook