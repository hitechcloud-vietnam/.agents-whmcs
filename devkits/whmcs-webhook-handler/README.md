# WHMCS Webhook Handler Module

A comprehensive webhook integration handler with event routing, payload transformation, validation, and retry logic.

## Features

- Webhook registration and management
- Event-based triggering
- Payload transformation
- Conditional filtering
- Automatic retry with exponential backoff
- Payload signing for verification
- Delivery status tracking
- Comprehensive logging
- Statistics and analytics

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/webhookhandler/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Webhook Handler"
   - Click **Activate**
   - Configure settings

3. Add cron job for processing deliveries:
   ```
   * * * * * php -q /path/to/whmcs/crons/cron.php /crons/webhook-delivery.php
   ```

## Usage

### Registering Webhooks

```php
// Register a new webhook
$result = webhookhandler_RegisterWebhook(array(
    'name' => 'Slack Notifications',
    'description' => 'Send notifications to Slack',
    'event_type' => 'invoice.created',
    'endpoint_url' => 'https://hooks.slack.com/services/xxx',
    'method' => 'POST',
    'headers' => array(
        'Authorization' => 'Bearer token123',
    ),
    'filters' => array(
        array('field' => 'total', 'operator' => 'greater_than', 'value' => 100),
    ),
    'transformations' => array(
        array('type' => 'rename_field', 'from' => 'id', 'to' => 'invoice_id'),
        array('type' => 'add_field', 'field' => 'source', 'value' => 'whmcs'),
    ),
));

$webhookKey = $result['webhook_key'];
$secret = $result['secret']; // Use this to verify signatures
```

### Managing Webhooks

```php
// Get all webhooks
$webhooks = webhookhandler_GetWebhooks(array('active_only' => true));

// Get specific webhook
$webhook = webhookhandler_GetWebhook($webhookKey);

// Update webhook
webhookhandler_UpdateWebhook($webhookKey, array(
    'endpoint_url' => 'https://new-endpoint.com/webhook',
    'is_active' => true,
));

// Delete webhook
webhookhandler_DeleteWebhook($webhookKey);
```

### Triggering Events

```php
// Trigger a webhook event
$result = webhookhandler_TriggerEvent('invoice.created', array(
    'id' => 12345,
    'total' => 99.99,
    'client_name' => 'John Doe',
    'items' => array(
        array('description' => 'Hosting Plan', 'amount' => 79.99),
        array('description' => 'SSL Certificate', 'amount' => 19.99),
    ),
));

$eventId = $result['event_id'];
```

### Event Types

You can trigger webhooks for any custom event type. Common examples:

- `invoice.created`
- `invoice.paid`
- `invoice.overdue`
- `service.created`
- `service.suspended`
- `service.cancelled`
- `domain.registered`
- `ticket.created`
- `ticket.reply`

### Filtering

```php
// Filters control when webhooks are sent
$filters = array(
    // Only if total > 100
    array('field' => 'total', 'operator' => 'greater_than', 'value' => 100),
    
    // Only for specific status
    array('field' => 'status', 'operator' => 'equals', 'value' => 'active'),
    
    // Only if contains string
    array('field' => 'name', 'operator' => 'contains', 'value' => 'premium'),
    
    // Multiple values
    array('field' => 'type', 'operator' => 'in', 'value' => array('hosting', 'vps')),
);
```

### Transformations

```php
// Transform payload before sending
$transformations = array(
    // Rename fields
    array('type' => 'rename_field', 'from' => 'id', 'to' => 'invoice_id'),
    
    // Add static fields
    array('type' => 'add_field', 'field' => 'source', 'value' => 'whmcs'),
    
    // Remove fields
    array('type' => 'remove_field', 'field' => 'internal_data'),
    
    // Map values
    array(
        'type' => 'map_values',
        'field' => 'status',
        'mapping' => array(
            'pending' => 'waiting',
            'active' => 'running',
            'suspended' => 'paused',
        ),
    ),
);
```

### Payload Verification

```php
// Verify incoming webhook signature
$headers = getallheaders();
$payload = file_get_contents('php://input');
$data = json_decode($payload, true);

$timestamp = $headers['X-Webhook-Timestamp'];
$signature = $headers['X-Webhook-Signature'];

$expectedSignature = hash_hmac('sha256', $timestamp . '.' . $payload, $secret);

if ($signature !== $expectedSignature) {
    http_response_code(401);
    exit('Invalid signature');
}
```

### Delivery Management

```php
// Get delivery status
$delivery = webhookhandler_GetDelivery($deliveryId);

// Retry failed delivery
webhookhandler_RetryDelivery($deliveryId);

// Process pending deliveries (for cron)
$results = webhookhandler_ProcessDeliveries(50);
```

### Statistics

```php
// Get webhook statistics
$stats = webhookhandler_GetStats($webhookKey);
// Returns: total, pending, success, failed, retrying, success_rate, avg_duration_ms

// Get global statistics
$allStats = webhookhandler_GetStats();
```

## Delivery Headers

Each webhook delivery includes these headers:

| Header | Description |
|--------|-------------|
| Content-Type | application/json |
| User-Agent | WHMCSDevKit-WebhookHandler/1.0 |
| X-Webhook-Event | Event type |
| X-Webhook-Key | Webhook identifier |
| X-Webhook-Signature | HMAC signature |
| X-Webhook-Timestamp | Unix timestamp |
| X-Delivery-ID | Unique delivery ID |

## Retry Policy

- Failed deliveries are retried automatically
- Default max attempts: 3
- Default retry delay: 60 seconds
- Retry delay can be configured per webhook

## Database Tables

- `mod_webhookhandler_webhooks` - Webhook definitions
- `mod_webhookhandler_deliveries` - Delivery attempts
- `mod_webhookhandler_events` - Event log

## API Functions

| Function | Description |
|----------|-------------|
| `webhookhandler_RegisterWebhook()` | Register new webhook |
| `webhookhandler_GetWebhooks()` | Get all webhooks |
| `webhookhandler_GetWebhook()` | Get webhook by key |
| `webhookhandler_UpdateWebhook()` | Update webhook |
| `webhookhandler_DeleteWebhook()` | Delete webhook |
| `webhookhandler_TriggerEvent()` | Trigger event |
| `webhookhandler_ProcessDeliveries()` | Process pending deliveries |
| `webhookhandler_DeliverWebhook()` | Deliver specific webhook |
| `webhookhandler_GetDelivery()` | Get delivery status |
| `webhookhandler_RetryDelivery()` | Retry failed delivery |
| `webhookhandler_GetStats()` | Get statistics |

## Version History

- **1.0.0** - Initial release
  - Webhook registration
  - Event triggering
  - Payload filtering
  - Payload transformation
  - Automatic retry
  - Signature verification
  - Statistics
