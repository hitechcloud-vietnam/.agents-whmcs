# WHMCS Telegram Notifications Workflow

## Purpose
Configure Telegram bot notifications in WHMCS.

## Prerequisites
- Telegram account
- Bot created via @BotFather

## Setup Steps

### Step 1: Create Telegram Bot
1. Open Telegram
2. Search for @BotFather
3. Send /newbot
4. Give bot a name
5. Give bot a username
6. Copy the HTTP API token

### Step 2: Set Up Channel/Group
1. Create channel or group
2. Add bot as admin with permissions
3. Get chat ID

### Step 3: Get Chat ID
Option A: Direct message bot
1. Send message to bot
2. Visit: https://api.telegram.org/bot<TOKEN>/getUpdates
3. Find "chat":{"id":xxxxx}

Option B: Add to group
1. Add bot to group
2. Send /start in group
3. Check getUpdates for group ID (negative number)

### Step 4: Configure in WHMCS
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Find Telegram
4. Enable Telegram
5. Enter bot token
6. Enter chat ID
7. Save

## Telegram Message Format

### Simple Message
```json
{
  "text": "Invoice #{$invoice_num} is overdue\nAmount: {$amount}\nDue: {$due_date}"
}
```

### With Inline Keyboard
```json
{
  "text": "*Invoice Alert*\nInvoice #{$invoice_num} for {$amount} is overdue.",
  "parse_mode": "Markdown",
  "reply_markup": {
    "inline_keyboard": [
      [
        {"text": "View Invoice", "url": "{$invoice_url}"},
        {"text": "Pay Now", "url": "{$payment_url}"}
      ]
    ]
  }
}
```

## Variables
```
{$client_name}
{$order_num}
{$invoice_num}
{$invoice_amount}
{$alert_title}
{$alert_message}
{$action_url}
```

## Formatting

### Markdown Support
```
*bold*
_italic_
[link text](url)
`code`
```

## Bot Configuration

### Set Bot Info
```
/setname - Change bot name
/setdescription - Set description
/setabouttext - Set about text
/setuserpic - Set profile picture
```

### Security
- Keep bot token secure
- Use bot for notifications only
- Restrict chat access if needed

## Testing

### Send Test Message
1. Click "Send Test" in WHMCS
2. Check Telegram
3. Verify message format
4. Test button actions

## Troubleshooting

### Bot Not Responding
- Verify token is correct
- Check bot is admin in channel
- Get updates via API:
```
https://api.telegram.org/bot<TOKEN>/getUpdates
```

### Chat ID Not Found
- Make sure bot is in channel
- Send message to bot first
- Check response in getUpdates

## Related Workflows
- whmcs-notification-providers
- whmcs-notification-create
- whmcs-notification-channels