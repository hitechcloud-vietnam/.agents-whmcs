# WHMCS API Webhook Handler

## Skill Description
Build robust webhook handlers for WHMCS modules to receive and process external events, validate incoming requests, and maintain reliable event processing with retry mechanisms.

## Prerequisites
- WHMCS 8.0+ installation
- PHP 7.4+ with JSON support
- Basic understanding of HTTP webhooks
- Database access for event logging

## Step-by-Step Implementation

### 1. Webhook Handler Class
```php
<?php
// includes/webhook/WebhookHandler.php

namespace WHMCS\Module\YourModule\Webhook;

class WebhookHandler
{
    private $db;
    private $secretKey;
    private $supportedEvents = [
        'invoice.created',
        'invoice.paid',
        'invoice.cancelled',
        'order.created',
        'order.completed',
        'ticket.created',
        'ticket.reply',
        'user.created',
        'user.updated'
    ];

    public function __construct(string $secretKey = '')
    {
        global $db;
        $this->db = $db;
        $this->secretKey = $secretKey ?: $this->getWebhookSecret();
    }

    private function getWebhookSecret(): string
    {
        $config = getWHMCSConfig();
        return $config['webhook_secret'] ?? '';
    }

    public function handle(array $payload, array $headers): array
    {
        // Validate webhook signature
        if (!$this->validateSignature($payload, $headers)) {
            return [
                'success' => false,
                'error' => 'Invalid signature',
                'http_code' => 401
            ];
        }

        // Parse event type
        $eventType = $payload['event'] ?? null;

        if (!$eventType || !in_array($eventType, $this->supportedEvents)) {
            return [
                'success' => false,
                'error' => 'Unsupported event type',
                'http_code' => 400
            ];
        }

        // Store webhook for processing
        $webhookId = $this->storeWebhook($payload, $headers);

        // Process webhook asynchronously
        $this->queueForProcessing($webhookId);

        return [
            'success' => true,
            'webhook_id' => $webhookId,
            'message' => 'Webhook received'
        ];
    }

    private function validateSignature(array $payload, array $headers): bool
    {
        if (empty($this->secretKey)) {
            return true; // Skip validation if no secret configured
        }

        $signature = $headers['X-Webhook-Signature'] ?? $headers['x-webhook-signature'] ?? '';

        if (empty($signature)) {
            logModuleCall('YourModule', 'webhook_invalid', $payload, 'Missing signature header');
            return false;
        }

        $payloadJson = json_encode($payload);
        $expectedSignature = hash_hmac('sha256', $payloadJson, $this->secretKey);

        return hash_equals($expectedSignature, $signature);
    }

    private function storeWebhook(array $payload, array $headers): int
    {
        $eventType = $payload['event'] ?? 'unknown';
        $eventId = $payload['id'] ?? uniqid('wh_');

        $this->db->insert('mod_yourmodule_webhooks', [
            'event_id' => $eventId,
            'event_type' => $eventType,
            'payload' => json_encode($payload),
            'headers' => json_encode($headers),
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? ''
        ]);

        return $this->db->getLastInsertID();
    }

    private function queueForProcessing(int $webhookId): void
    {
        // Add to processing queue
        $this->db->insert('mod_yourmodule_webhook_queue', [
            'webhook_id' => $webhookId,
            'status' => 'queued',
            'queued_at' => date('Y-m-d H:i:s'),
            'attempts' => 0
        ]);
    }

    public function processWebhook(int $webhookId): array
    {
        $webhook = $this->db->select(
            "SELECT * FROM mod_yourmodule_webhooks WHERE id = ?",
            [$webhookId]
        );

        if (empty($webhook)) {
            return ['success' => false, 'error' => 'Webhook not found'];
        }

        $webhook = $webhook[0];
        $payload = json_decode($webhook['payload'], true);

        try {
            $result = $this->dispatchEvent($webhook['event_type'], $payload);

            $this->db->update('mod_yourmodule_webhooks', [
                'status' => 'processed',
                'processed_at' => date('Y-m-d H:i:s'),
                'result' => json_encode($result)
            ], 'id = ?', [$webhookId]);

            return ['success' => true, 'result' => $result];

        } catch (\Exception $e) {
            $this->db->update('mod_yourmodule_webhooks', [
                'status' => 'failed',
                'error_message' => $e->getMessage(),
                'attempts' => $webhook['attempts'] + 1
            ], 'id = ?', [$webhookId]);

            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function dispatchEvent(string $eventType, array $payload): mixed
    {
        $handlers = [
            'invoice.paid' => [$this, 'handleInvoicePaid'],
            'order.created' => [$this, 'handleOrderCreated'],
            'ticket.created' => [$this, 'handleTicketCreated'],
        ];

        if (isset($handlers[$eventType])) {
            return call_user_func($handlers[$eventType], $payload);
        }

        // Generic event handler
        return ['handled' => true, 'event' => $eventType];
    }

    private function handleInvoicePaid(array $payload): array
    {
        $invoiceId = $payload['invoice_id'] ?? 0;

        // Your custom logic here
        logActivity("Custom: Invoice #{$invoiceId} paid webhook processed");

        return [
            'invoice_id' => $invoiceId,
            'amount' => $payload['amount'] ?? 0,
            'handled' => true
        ];
    }

    private function handleOrderCreated(array $payload): array
    {
        $orderId = $payload['order_id'] ?? 0;

        // Your custom logic here
        logActivity("Custom: Order #{$orderId} created webhook processed");

        return [
            'order_id' => $orderId,
            'handled' => true
        ];
    }

    private function handleTicketCreated(array $payload): array
    {
        $ticketId = $payload['ticket_id'] ?? 0;

        // Your custom logic here
        logActivity("Custom: Ticket #{$ticketId} created webhook processed");

        return [
            'ticket_id' => $ticketId,
            'handled' => true
        ];
    }
}
```

### 2. Retry Handler
```php
<?php
// includes/webhook/WebhookRetryHandler.php

namespace WHMCS\Module\YourModule\Webhook;

class WebhookRetryHandler
{
    private $db;
    private $maxAttempts = 5;
    private $retryDelays = [60, 300, 900, 3600, 14400]; // 1m, 5m, 15m, 1h, 4h

    public function __construct()
    {
        global $db;
        $this->db = $db;
    }

    public function processRetryQueue(): array
    {
        $pending = $this->db->select(
            "SELECT * FROM mod_yourmodule_webhook_queue
             WHERE status = 'queued'
             AND attempts < ?
             AND (next_retry_at IS NULL OR next_retry_at <= NOW())
             ORDER BY queued_at ASC
             LIMIT 10",
            [$this->maxAttempts]
        );

        $results = [];

        foreach ($pending as $job) {
            $results[] = $this->processJob($job);
        }

        return $results;
    }

    private function processJob(array $job): array
    {
        $webhookHandler = new WebhookHandler();

        $this->db->update('mod_yourmodule_webhook_queue', [
            'status' => 'processing',
            'attempts' => $job['attempts'] + 1,
            'last_attempt_at' => date('Y-m-d H:i:s')
        ], 'id = ?', [$job['id']]);

        $result = $webhookHandler->processWebhook($job['webhook_id']);

        if ($result['success']) {
            $this->db->update('mod_yourmodule_webhook_queue', [
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s')
            ], 'id = ?', [$job['id']]);

            return ['job_id' => $job['id'], 'status' => 'completed'];

        } else {
            $nextRetry = $this->calculateNextRetry($job['attempts'] + 1);

            $this->db->update('mod_yourmodule_webhook_queue', [
                'status' => $job['attempts'] + 1 >= $this->maxAttempts ? 'failed' : 'queued',
                'next_retry_at' => $nextRetry ? date('Y-m-d H:i:s', $nextRetry) : null,
                'last_error' => $result['error'] ?? 'Unknown error'
            ], 'id = ?', [$job['id']]);

            return [
                'job_id' => $job['id'],
                'status' => 'retry_scheduled',
                'next_retry' => $nextRetry
            ];
        }
    }

    private function calculateNextRetry(int $attempt): ?int
    {
        if ($attempt >= $this->maxAttempts) {
            return null;
        }

        $delayIndex = min($attempt - 1, count($this->retryDelays) - 1);
        $delay = $this->retryDelays[$delayIndex];

        return time() + $delay;
    }
}
```

### 3. Webhook Registration Hook
```php
<?php
// hooks.php

add_hook('AdminAreaFooter', 1, function($vars) {
    // Add webhook configuration to admin area
});

add_hook('WebClientAreaPage', 1, function($vars) {
    // Process incoming webhooks
});
```

### 4. Database Tables
```sql
-- Webhooks storage table
CREATE TABLE IF NOT EXISTS mod_yourmodule_webhooks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_id VARCHAR(255) UNIQUE,
    event_type VARCHAR(100) NOT NULL,
    payload LONGTEXT NOT NULL,
    headers TEXT,
    status ENUM('pending', 'processing', 'processed', 'failed') DEFAULT 'pending',
    ip_address VARCHAR(45),
    created_at DATETIME NOT NULL,
    processed_at DATETIME,
    result TEXT,
    error_message TEXT,
    attempts INT DEFAULT 0,
    INDEX idx_event_type (event_type),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Processing queue table
CREATE TABLE IF NOT EXISTS mod_yourmodule_webhook_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    webhook_id INT NOT NULL,
    status ENUM('queued', 'processing', 'completed', 'failed') DEFAULT 'queued',
    queued_at DATETIME NOT NULL,
    last_attempt_at DATETIME,
    next_retry_at DATETIME,
    completed_at DATETIME,
    attempts INT DEFAULT 0,
    last_error TEXT,
    FOREIGN KEY (webhook_id) REFERENCES mod_yourmodule_webhooks(id) ON DELETE CASCADE,
    INDEX idx_status (status),
    INDEX idx_next_retry (next_retry_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Duplicate webhook processing | Use idempotency keys and check before processing |
| Webhook timeout in client | Process asynchronously and return 200 immediately |
| Missing webhook retries | Implement exponential backoff with max retries |
| Invalid payload handling | Validate all expected fields before processing |
| Webhook replay attacks | Store and check event IDs to prevent replay |

## Security Considerations

1. **Always validate signatures** - Verify HMAC signature before processing
2. **Use HTTPS endpoints** - Never accept webhooks over HTTP
3. **Log all incoming webhooks** - Store for auditing and debugging
4. **Implement idempotency** - Prevent duplicate processing of same event
5. **Rate limit per source IP** - Prevent webhook flooding attacks
6. **Validate payload structure** - Check all required fields exist

## Testing Checklist

- [ ] Test webhook with valid signature
- [ ] Test webhook with invalid signature
- [ ] Test webhook with missing signature
- [ ] Test all supported event types
- [ ] Test unsupported event handling
- [ ] Test retry mechanism
- [ ] Test max retry limit
- [ ] Test idempotency with duplicate events
- [ ] Test webhook logging
- [ ] Test asynchronous processing

## Reference Links

- [WHMCS Webhook System](https://developers.whmcs.com/webhooks/)
- [Webhook Security Best Practices](https://developer.squareup.com/blog/ heartbeat-based-protocols-for-webhook-safety)
- [HTTP Webhooks Guide](https://webhooks.io/)
