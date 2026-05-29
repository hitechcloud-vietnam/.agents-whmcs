# WHMCS Webhook Debugging Workflow

## Overview
This workflow guides you through debugging webhook issues in WHMCS modules.

## Prerequisites
- Webhook logs
- ngrok or similar
- Request inspection tools

## Step-by-Step Guide

### Step 1: Enable Webhook Logging
```php
// In webhook handler
public function handleWebhook(Request $request): Response
{
    $payload = $request->getContent();
    $signature = $request->header('X-Webhook-Signature');

    // Log incoming webhook
    logModuleCall(
        'yourmodule',
        'webhook_received',
        $payload,
        "Signature: $signature"
    );

    // ... process webhook
}
```

### Step 2: Test Webhook Locally
```bash
# Start ngrok
ngrok http 80

# Configure WHMCS webhook URL to ngrok URL
# e.g., https://abc123.ngrok.io/modules/addons/yourmodule/webhook.php

# Send test webhook
curl -X POST "https://abc123.ngrok.io/modules/addons/yourmodule/webhook.php" \
    -H "Content-Type: application/json" \
    -d '{"event": "test", "data": {"id": 1}}'
```

### Step 3: Check Webhook Delivery
```bash
# Check WHMCS webhook configuration
# Settings > System Settings > Module Settings > Your Module

# Verify webhook URL is accessible
curl -v "https://your-webhook-url.com/webhook.php"
```

### Step 4: Common Webhook Issues
```php
// Issue: Signature verification fails
// Fix: Check secret key and hashing algorithm

// Issue: Webhook not triggering
// Fix: Verify WHMCS event hooks are registered

// Issue: Processing errors
// Fix: Check webhook handler exceptions are caught
```

## Webhook Debugging Checklist

### Investigation
- [ ] Webhook logs checked
- [ ] Request received (or not)
- [ ] Signature verified
- [ ] Processing errors found

### Resolution
- [ ] URL configured correctly
- [ ] Signature validated
- [ ] Error handling improved
- [ ] Retry mechanism added
