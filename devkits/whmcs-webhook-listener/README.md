# WHMCS Webhook Listener DevKit

A comprehensive webhook listener module for WHMCS that handles incoming webhook events and triggers external endpoints.

## Features

- Handle all WHMCS webhook events
- Signature validation (HMAC-SHA256)
- IP whitelist filtering
- Async and sync processing modes
- Automatic retry mechanism
- Comprehensive logging
- Admin interface for management

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_webhook_listener/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Access the webhook endpoint at:
   ```
   https://your-whmcs.com/modules/addons/whmcs_webhook_listener/webhook.php
   ```

## Configuration

### Admin Settings

1. **Webhook Secret**: Used for validating incoming webhook signatures
2. **Allowed IPs**: Comma-separated list of IPs allowed to send webhooks
3. **Async Processing**: Enable to process webhooks via cron instead of immediately
4. **Max Retries**: Number of retry attempts for failed webhooks

### Event Types

Supported WHMCS events:
- `ClientAdd` - New client created
- `ClientEdit` - Client updated
- `ClientDelete` - Client deleted
- `AfterModuleCreate` - Service provisioned
- `AfterModuleSuspend` - Service suspended
- `AfterModuleUnsuspend` - Service unsuspended
- `AfterModuleTerminate` - Service terminated
- `InvoicePaid` - Invoice paid
- `InvoiceCancelled` - Invoice cancelled
- `InvoiceCreated` - Invoice created
- `TicketOpen` - Ticket opened
- `TicketReply` - Ticket replied
- `TicketClose` - Ticket closed
- `AcceptOrder` - Order accepted
- `AfterServiceChangePackage` - Package changed
- `AfterRegistrarRegistration` - Domain registered
- `AfterRegistrarRenewal` - Domain renewed
- `AfterRegistrarTransfer` - Domain transferred
- `DailyCronJob` - Daily cron runs

## Usage

### Registering a Webhook

1. Go to Admin > Addon Modules > Webhook Listener
2. Click "Add Webhook Endpoint"
3. Select the event type
4. Enter the webhook URL
5. Add custom headers if needed
6. Save

### Sending Webhooks from WHMCS

Configure WHMCS to send webhooks to your endpoint:

1. Go to Configuration > System Settings > Webhooks
2. Add a new webhook
3. Set the URL to your webhook endpoint
4. Select events to trigger

### Processing Webhooks

#### Sync Mode (Default)
Webhooks are processed immediately when received.

#### Async Mode
Webhooks are queued and processed via cron job:

```php
// Add to hooks.php
add_hook('DailyCronJob', 1, function($vars) {
    $processor = new \WebhookListener\EventProcessor();
    $processed = $processor->processPendingEvents();
});
```

## Webhook Payload Format

```json
{
    "event": "ClientAdd",
    "timestamp": "2024-05-28T10:30:00Z",
    "data": {
        "userid": 12345,
        "email": "client@example.com",
        "firstname": "John",
        "lastname": "Doe"
    }
}
```

## Signature Validation

To validate webhook authenticity, configure a webhook secret:

1. Set the secret in admin settings
2. WHMCS will sign requests with HMAC-SHA256
3. Header: `X-WHMCS-Signature` or `X-Signature`

Verify in your receiving endpoint:

```php
$signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'];
$payload = file_get_contents('php://input');
$expected = hash_hmac('sha256', $payload, $secret);

if (hash_equals($expected, $signature)) {
    // Valid webhook
}
```

## API Endpoints

### Webhook Endpoint
```
POST /modules/addons/whmcs_webhook_listener/webhook.php
```

### Admin Actions
```
GET  /admin/addonmodules.php?module=whmcs_webhook_listener
POST /admin/addonmodules.php?module=whmcs_webhook_listener
```

## Logging

All webhook activity is logged in:
- `mod_{module}_webhook_logs` - Request/response logs
- `mod_{module}_webhook_events` - Event processing status

View logs in Admin interface or query directly:

```php
$logs = Capsule::table('mod_whmcs_webhook_listener_webhook_logs')
    ->orderBy('created_at', 'desc')
    ->limit(100)
    ->get();
```

## Error Handling

Failed webhooks are automatically retried up to the configured max retries. Each retry is scheduled with exponential backoff.

Failed events can be manually retriggered from the admin interface.

## Security

- Always validate webhook signatures
- Use IP whitelisting when possible
- Store secrets securely
- Log all webhook activity for auditing
- Use HTTPS for webhook endpoints

## File Structure

```
whmcs-webhook-listener/
├── webhook-listener.php    # Main module file
├── webhook.php             # Webhook endpoint
├── lib/
│   ├── EventProcessor.php  # Event processing logic
│   ├── WebhookValidator.php # Request validation
│   └── EventHandlers.php   # Event handler implementations
└── templates/
    └── webhook_config.tpl  # Admin configuration
```

## Requirements

- WHMCS 7.0+
- PHP 7.4+
- cURL extension

## Support

For issues and feature requests, please contact the developer.