# WHMCS SMS Notifications Workflow

## Purpose
Set up and manage SMS notifications in WHMCS.

## Prerequisites
- SMS provider account
- SMS credits purchased
- WHMCS notification module enabled

## Supported Providers

### 1. Twilio
1. Create Twilio account
2. Get Account SID and Auth Token
3. Purchase phone number
4. Note SMS-enabled number

### 2. Nexmo/Vonage
1. Create Vonage account
2. Get API Key and Secret
3. Rent SMS-enabled number

### 3. ClickSend
1. Create ClickSend account
2. Get username and API key

### 4. msg91
1. Create msg91 account
2. Get authkey
3. Configure sender ID

## Configuration

### Step 1: Enable SMS Provider
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Find SMS section
4. Select provider

### Step 2: Enter Credentials
```
Twilio Configuration:
- Account SID: ACxxxxxxxx
- Auth Token: xxxxxxxx
- From Number: +1234567890

Nexmo Configuration:
- API Key: xxxxxxxx
- API Secret: xxxxxxxx
- From Number: +1234567890
```

### Step 3: Test Connection
1. Click "Test Connection"
2. Enter test phone number
3. Send test SMS
4. Verify delivery

## SMS Notification Setup

### Step 1: Create SMS Template
1. Navigate to: Configuration > System > Notifications
2. Select "Templates" tab
3. Create SMS-specific template

### Step 2: Character Limits
- Standard SMS: 160 characters
- Long SMS: Multiple segments (153 chars/segment)
- Keep messages concise

### Step 3: Variables for SMS
```
{$client_name}
{$client_phone}
{$alert_message}
{$invoice_amount}
{$due_date}
{$order_number}
```

## SMS Best Practices

### Message Design
1. Keep under 160 characters
2. Include clear call-to-action
3. Add link to more info
4. Include sender identification

### Example Messages
```
Invoice #{$invoice_num} due on {$due_date}. Pay now: {$payment_url}

Your order #{$order_num} has shipped. Track: {$tracking_url}

Service {$product_name} expires in {$days} days. Renew: {$renewal_url}
```

## Cost Management

### Budget Controls
1. Set daily/monthly limits
2. Enable only for important events
3. Use short codes for routing

### Monitoring
1. Track SMS usage
2. Review costs
3. Optimize message length

## Troubleshooting

### SMS Not Sending
- Check provider credits
- Verify phone number format
- Check API credentials
- Review error logs

### Delivery Issues
- Verify international format (+1-xxx-xxx-xxxx)
- Check blocked numbers
- Review spam filters

## Related Workflows
- whmcs-notification-create
- whmcs-notification-providers
- whmcs-notification-schedule