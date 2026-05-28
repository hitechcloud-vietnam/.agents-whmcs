# WHMCS Webhook Builder Module

Webhook builder and manager for automation and integrations.

## Features

- Webhook creation and management
- Trigger types (events, schedules, conditions)
- HTTP methods (GET, POST, PUT, DELETE, PATCH)
- Custom headers and authentication
- Body templates (JSON, XML, form-data)
- Variable substitution
- Retry logic with backoff
- Webhook history and logs
- Payload transformation
- Filter conditions
- Multiple endpoints per webhook
- Encryption for sensitive data
- HMAC signature verification
- Rate limiting
- Webhook monitoring

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/webhookbuilder/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Create webhooks

## Usage

```php
// Create webhook
$result = webhookbuilder_CreateWebhook(array(
    'webhook_name' => 'Order Created Notification',
    'webhook_key' => 'order_created_notify',
    'trigger_type' => 'event',
    'trigger_event' => 'AfterModuleCreate',
    'endpoint_url' => 'https://example.com/webhook',
    'method' => 'POST',
    'headers' => array(
        'Content-Type' => 'application/json',
        'Authorization' => 'Bearer {{api_key}}'
    ),
    'body_template' => '{"order_id": "{{order.id}}", "status": "{{status}}"}',
    'retry_enabled' => true,
    'max_retries' => 3,
    'timeout_seconds' => 30
));

// Manually trigger webhook
$result = webhookbuilder_TriggerWebhook($webhookId, array(
    'order_id' => 123,
    'status' => 'active'
));

// Get webhook status
$status = webhookbuilder_GetWebhookStatus($webhookId);

// Test webhook
$test = webhookbuilder_TestWebhook($webhookId);
// Returns: success, status_code, response_body, duration_ms

// Get webhook history
$history = webhookbuilder_GetWebhookHistory($webhookId, 50);
// Returns: trigger_time, status, response_code, duration_ms

// Update webhook
webhookbuilder_UpdateWebhook($webhookId, array(
    'endpoint_url' => 'https://new-endpoint.com/webhook',
    'body_template' => '{"updated": true}'
));

// Delete webhook (soft delete)
webhookbuilder_DeleteWebhook($webhookId);

// Pause/Resume webhook
webhookbuilder_PauseWebhook($webhookId);
webhookbuilder_ResumeWebhook($webhookId);

// Add variable substitution
webhookbuilder_AddVariable($webhookId, 'customer_email', 'client.email');

// Create scheduled webhook
webhookbuilder_CreateScheduledWebhook(array(
    'webhook_name' => 'Daily Report',
    'webhook_key' => 'daily_report',
    'schedule' => '0 9 * * *',
    'endpoint_url' => 'https://api.example.com/report',
    'body_template' => '{"date": "{{date}}", "stats": {{stats}}}'
));

// Get webhook logs
$logs = webhookbuilder_GetLogs($webhookId, array(
    'status' => 'failed',
    'from_date' => '2026-05-01'
));

// Retry failed webhook
webhookbuilder_RetryWebhook($logId);

// Add webhook filter
webhookbuilder_AddFilter($webhookId, array(
    'field' => 'service.amount',
    'operator' => '>',
    'value' => 100
));

// Create conditional webhook
webhookbuilder_CreateConditionalWebhook(array(
    'webhook_name' => 'High Value Order Alert',
    'trigger_event' => 'OrderCreated',
    'conditions' => array(
        array('field' => 'amount', 'operator' => '>=', 'value' => 1000)
    ),
    'endpoint_url' => 'https://alerts.example.com/high-value'
));
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableWebhookLogging | yesno | yes | Log all webhook calls |
| MaxRetries | text | 3 | Default max retries |
| TimeoutSeconds | text | 30 | Default timeout |
| RetryBackoff | dropdown | exponential | Retry backoff type |
| EnableEncryption | yesno | yes | Enable payload encryption |
| HmacSecret | password | - | HMAC signature secret |
| RateLimitPerHour | text | 1000 | Rate limit per hour |
| EnableMonitoring | yesno | yes | Enable monitoring |
| AlertOnFailure | yesno | yes | Alert on failures |

## Trigger Types

| Type | Description |
|------|-------------|
| event | Triggered by WHMCS events |
| schedule | Triggered by cron schedule |
| manual | Manually triggered |
| conditional | Triggered by conditions |

## Supported Events

| Event | Description |
|-------|-------------|
| ClientAdd | New client created |
| ClientEdit | Client edited |
| ClientDelete | Client deleted |
| AfterModuleCreate | Service created |
| AfterModuleSuspend | Service suspended |
| AfterModuleTerminate | Service terminated |
| InvoicePaid | Invoice paid |
| InvoiceCreated | Invoice created |
| OrderCreated | Order placed |
|TicketOpen | Ticket opened |
| TicketReply | Ticket replied |

## HTTP Methods

| Method | Use Case |
|-------|----------|
| GET | Retrieve data |
| POST | Create resources |
| PUT | Full update |
| PATCH | Partial update |
| DELETE | Remove resources |

## Database Tables

- `mod_webhookbuilder_webhooks` - Webhook definitions
- `mod_webhookbuilder_logs` - Execution logs
- `mod_webhookbuilder_variables` - Variable mappings
- `mod_webhookbuilder_filters` - Condition filters
- `mod_webhookbuilder_schedules` - Schedules
- `mod_webhookbuilder_destinations` - Multiple endpoints

## API Functions

| Function | Description |
|----------|-------------|
| `webhookbuilder_CreateWebhook()` | Create webhook |
| `webhookbuilder_GetWebhook()` | Get webhook details |
| `webhookbuilder_GetWebhooks()` | List webhooks |
| `webhookbuilder_UpdateWebhook()` | Update webhook |
| `webhookbuilder_DeleteWebhook()` | Delete webhook |
| `webhookbuilder_PauseWebhook()` | Pause webhook |
| `webhookbuilder_ResumeWebhook()` | Resume webhook |
| `webhookbuilder_TriggerWebhook()` | Manual trigger |
| `webhookbuilder_TestWebhook()` | Test webhook |
| `webhookbuilder_GetWebhookStatus()` | Get status |
| `webhookbuilder_GetWebhookHistory()` | Get history |
| `webhookbuilder_GetLogs()` | Get detailed logs |
| `webhookbuilder_RetryWebhook()` | Retry failed |
| `webhookbuilder_AddVariable()` | Add variable |
| `webhookbuilder_AddFilter()` | Add condition |
| `webhookbuilder_CreateScheduledWebhook()` | Create schedule |
