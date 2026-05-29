# WHMCS Webhook Automation Workflow

## Overview
This workflow implements webhook-based automation for real-time integration with external systems.

## Prerequisites
- WHMCS v7.10+ for native webhooks
- External service endpoint URL
- API authentication credentials (if required)

## Step-by-Step Process

### Step 1: Understand Webhook Architecture
```
WHMCS Event → Webhook Trigger → HTTP POST → External Endpoint → Processing
                    ↓
            Retry Queue → Dead Letter Queue (if failed)
```

### Step 2: Configure WHMCS Native Webhooks

#### Access Webhook Configuration
1. Log into WHMCS admin area
2. Navigate to **Configuration > System Settings > Webhooks**
3. Click **Create New Webhook**

#### Webhook Settings
```
Webhook Configuration:
- Name: [External System Sync]
- URL: [https://api.example.com/webhooks/whmcs]
- Method: [POST]
- Format: [JSON]

Authentication:
- Type: [Bearer Token]
- Token: [your-secret-token]

Headers (optional):
- X-Custom-Header: custom-value
- X-Webhook-Source: whmcs
```

### Step 3: Configure Webhook Triggers

#### Select Events to Trigger
Check events to subscribe:
- [x] Client Created
- [x] Client Updated
- [x] Service Created
- [x] Service Suspended
- [x] Service Terminated
- [x] Invoice Created
- [x] Invoice Paid
- [x] Invoice Overdue
- [x] Order Placed
- [x] Ticket Created

### Step 4: Implement Custom Webhook Handler

#### Create Webhook Module
```php
<?php
// /includes/hooks/webhook_handler.php

/**
 * Custom Webhook Event Handlers
 */

add_hook('WebhookPayloadSent', 1, function($vars) {
    // Log all webhook deliveries
    logWebhookDelivery($vars);

    // Track delivery status
    if ($vars['success']) {
        incrementMetric('webhooks_sent_success');
    } else {
        incrementMetric('webhooks_sent_failed');
        handleWebhookFailure($vars);
    }
});

function logWebhookDelivery($vars)
{
    Capsule::table('mod_webhook_log')->insert([
        'webhook_id' => $vars['webhook_id'],
        'event_type' => $vars['event_type'],
        'payload' => json_encode($vars['payload']),
        'endpoint' => $vars['endpoint'],
        'response_code' => $vars['response_code'] ?? null,
        'success' => $vars['success'] ? 1 : 0,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

function handleWebhookFailure($vars)
{
    // Queue for retry
    if ($vars['retry_count'] < 3) {
        queueWebhookRetry($vars);
    } else {
        // Move to dead letter queue
        moveToDeadLetterQueue($vars);
        notifyWebhookFailure($vars);
    }
}
```

### Step 5: Create Webhook Event Listeners

#### Client Webhook Events
```php
// Client created webhook data
add_hook('ClientAdd', 1, function($vars) {
    $webhookPayload = [
        'event' => 'client.created',
        'timestamp' => date('c'),
        'data' => [
            'client_id' => $vars['userid'],
            'email' => $vars['email'],
            'first_name' => $vars['firstname'],
            'last_name' => $vars['lastname'],
            'company' => $vars['companyname'] ?? null,
            'created_at' => date('c')
        ]
    ];

    dispatchWebhook($webhookPayload, 'client_events');
});

// Client updated webhook data
add_hook('ClientEdit', 1, function($vars) {
    $webhookPayload = [
        'event' => 'client.updated',
        'timestamp' => date('c'),
        'data' => [
            'client_id' => $vars['userid'],
            'changes' => $vars['changes'] ?? [],
            'updated_at' => date('c')
        ]
    ];

    dispatchWebhook($webhookPayload, 'client_events');
});
```

#### Service Webhook Events
```php
// Service lifecycle webhooks
add_hook('AfterServiceCreate', 1, function($vars) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    $webhookPayload = [
        'event' => 'service.created',
        'timestamp' => date('c'),
        'data' => [
            'service_id' => $vars['serviceid'],
            'client_id' => $vars['userid'],
            'product_id' => $service->packageid,
            'domain' => $service->domain,
            'billing_cycle' => $service->billingcycle,
            'next_due_date' => $service->nextduedate,
            'status' => $service->domainstatus
        ]
    ];

    dispatchWebhook($webhookPayload, 'service_events');
});

add_hook('ServiceSuspended', 1, function($vars) {
    $webhookPayload = [
        'event' => 'service.suspended',
        'timestamp' => date('c'),
        'data' => [
            'service_id' => $vars['serviceid'],
            'client_id' => $vars['userid'],
            'reason' => $vars['suspendreason'] ?? 'Payment overdue',
            'suspended_at' => date('c')
        ]
    ];

    dispatchWebhook($webhookPayload, 'service_events');
});

add_hook('ServiceTerminated', 1, function($vars) {
    $webhookPayload = [
        'event' => 'service.terminated',
        'timestamp' => date('c'),
        'data' => [
            'service_id' => $vars['serviceid'],
            'client_id' => $vars['userid'],
            'terminated_at' => date('c')
        ]
    ];

    dispatchWebhook($webhookPayload, 'service_events');
});
```

#### Invoice Webhook Events
```php
add_hook('InvoiceCreated', 1, function($vars) {
    $invoice = localApi('GetInvoice', ['invoiceid' => $vars['invoiceid']]);

    $webhookPayload = [
        'event' => 'invoice.created',
        'timestamp' => date('c'),
        'data' => [
            'invoice_id' => $vars['invoiceid'],
            'client_id' => $invoice['userid'],
            'total' => $invoice['total'],
            'balance' => $invoice['balance'],
            'due_date' => $invoice['duedate'],
            'status' => $invoice['status']
        ]
    ];

    dispatchWebhook($webhookPayload, 'invoice_events');
});

add_hook('InvoicePaid', 1, function($vars) {
    $invoice = localApi('GetInvoice', ['invoiceid' => $vars['invoiceid']]);

    $webhookPayload = [
        'event' => 'invoice.paid',
        'timestamp' => date('c'),
        'data' => [
            'invoice_id' => $vars['invoiceid'],
            'client_id' => $invoice['userid'],
            'amount_paid' => $invoice['total'],
            'payment_method' => $invoice['paymentmethod'],
            'paid_at' => $invoice['datepaid']
        ]
    ];

    dispatchWebhook($webhookPayload, 'invoice_events');
});
```

### Step 6: Implement Webhook Dispatcher
```php
<?php

function dispatchWebhook($payload, $channel = 'default')
{
    $webhooks = getWebhooksForChannel($channel);

    foreach ($webhooks as $webhook) {
        if (!$webhook['enabled']) {
            continue;
        }

        // Build request
        $headers = [
            'Content-Type: application/json',
            'X-Webhook-Event: ' . $payload['event'],
            'X-Webhook-Timestamp: ' . $payload['timestamp'],
            'X-Webhook-Signature: ' . generateSignature($payload, $webhook['secret'])
        ];

        // Add custom headers
        if (!empty($webhook['custom_headers'])) {
            $customHeaders = json_decode($webhook['custom_headers'], true);
            foreach ($customHeaders as $key => $value) {
                $headers[] = "$key: $value";
            }
        }

        // Make HTTP request
        $ch = curl_init($webhook['url']);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        // Log result
        logWebhookDelivery([
            'webhook_id' => $webhook['id'],
            'endpoint' => $webhook['url'],
            'payload' => $payload,
            'response_code' => $httpCode,
            'response' => $response,
            'success' => $httpCode >= 200 && $httpCode < 300,
            'error' => $error
        ]);
    }
}

function generateSignature($payload, $secret)
{
    $signature = hash_hmac('sha256', json_encode($payload), $secret);
    return $signature;
}
```

### Step 7: Set Up Webhook Retry Logic
```php
<?php

function queueWebhookRetry($vars)
{
    $delay = calculateRetryDelay($vars['retry_count']);

    Capsule::table('mod_webhook_retry_queue')->insert([
        'webhook_log_id' => $vars['webhook_log_id'],
        'payload' => $vars['payload'],
        'endpoint' => $vars['endpoint'],
        'retry_count' => $vars['retry_count'] + 1,
        'next_retry_at' => date('Y-m-d H:i:s', time() + $delay),
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

function calculateRetryDelay($retryCount)
{
    $delays = [
        1 => 60,      // 1 minute
        2 => 300,     // 5 minutes
        3 => 900,     // 15 minutes
        4 => 3600,    // 1 hour
        5 => 14400    // 4 hours
    ];

    return $delays[$retryCount] ?? 86400;
}
```

#### Process Retry Queue (Cron Task)
```php
add_hook('CronJobHourly', 1, function($vars) {
    $pendingRetries = Capsule::table('mod_webhook_retry_queue')
        ->where('next_retry_at', '<=', date('Y-m-d H:i:s'))
        ->where('retry_count', '<', 5)
        ->get();

    foreach ($pendingRetries as $retry) {
        $result = sendWebhookRequest($retry->endpoint, $retry->payload);

        if ($result['success']) {
            Capsule::table('mod_webhook_retry_queue')
                ->where('id', $retry->id)
                ->delete();

            logWebhookDelivery([
                'webhook_id' => $retry->webhook_id,
                'endpoint' => $retry->endpoint,
                'success' => true,
                'note' => 'Retry successful'
            ]);
        } else {
            Capsule::table('mod_webhook_retry_queue')
                ->where('id', $retry->id)
                ->update([
                    'retry_count' => $retry->retry_count + 1,
                    'next_retry_at' => date('Y-m-d H:i:s', time() + calculateRetryDelay($retry->retry_count + 1))
                ]);
        }
    }
});
```

### Step 8: Implement Webhook Verification (Receiving)

If WHMCS receives webhooks from external systems:
```php
<?php
// /includes/hooks/webhook_receiver.php

add_hook('WebhooksAuthenticate', 1, function($vars) {
    $signature = $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '';

    // Verify HMAC signature
    $expectedSignature = hash_hmac('sha256', file_get_contents('php://input'), $secret);

    if (!hash_equals($expectedSignature, $signature)) {
        throw new Exception('Invalid webhook signature');
    }

    return true;
});
```

### Step 9: Monitor Webhook Performance

#### Dashboard Query
```php
function getWebhookStats($days = 7)
{
    $stats = Capsule::select("
        SELECT
            DATE(created_at) as date,
            event_type,
            COUNT(*) as total,
            SUM(success) as successful,
            AVG(response_time) as avg_response_time
        FROM mod_webhook_log
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
        GROUP BY DATE(created_at), event_type
        ORDER BY date DESC
    ", [$days]);

    return $stats;
}
```

## Best Practices

1. **Always verify signatures** - Authenticate incoming webhooks
2. **Use idempotency keys** - Prevent duplicate processing
3. **Implement timeouts** - Don't block on external services
4. **Log everything** - Enable detailed webhook logging
5. **Set up monitoring** - Alert on webhook failures
6. **Test locally** - Use tunneling for webhook testing

## Testing Webhooks

### Local Testing with ngrok
```bash
# Start ngrok tunnel
ngrok http 80

# Use the forwarded URL as webhook endpoint
```

### Send Test Webhook
```php
// WHMCS Admin > Configuration > Webhooks > Test
```

## Related Workflows
- [WHMCS API Integration](./whmcs-api-integration.md)
- [WHMCS Event-Driven Automation](./whmcs-event-driven-automation.md)
- [WHMCS Third-Party Sync](./whmcs-third-party-sync.md)