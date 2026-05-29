# WHMCS API Webhooks

## Overview

Webhooks allow WHMCS to notify external systems about events in real-time. This guide covers webhook configuration, handling, and security.

## Webhook Configuration

### Registering Webhooks

```php
<?php
// In WHMCS hooks file
add_hook('ClientCreated', 1, function($vars) {
    $webhook = new WhmcsWebhookNotifier();
    $webhook->trigger('client.created', [
        'client_id' => $vars['client_id'],
        'email' => $vars['email'],
        'first_name' => $vars['firstname'],
        'last_name' => $vars['lastname'],
    ]);
});
```

### Webhook Manager Class

```php
<?php
class WhmcsWebhookNotifier {
    private string $signingSecret;
    private int $timeout = 30;
    private int $maxRetries = 3;
    
    public function __construct(string $signingSecret = null)
    {
        $this->signingSecret = $signingSecret ?? getenv('WEBHOOK_SIGNING_SECRET');
    }
    
    public function trigger(string $event, array $payload, array $webhookUrls = []): array
    {
        $webhookUrls = $webhookUrls ?: $this->getRegisteredWebhooks($event);
        
        $results = [];
        foreach ($webhookUrls as $url) {
            $results[$url] = $this->send($url, $event, $payload);
        }
        
        return $results;
    }
    
    private function send(string $url, string $event, array $payload): array
    {
        $timestamp = time();
        $body = json_encode([
            'event' => $event,
            'timestamp' => $timestamp,
            'data' => $payload,
        ]);
        
        $signature = $this->generateSignature($body, $timestamp);
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature: ' . $signature,
                'X-Webhook-Timestamp: ' . $timestamp,
                'X-Webhook-Event: ' . $event,
            ],
            CURLOPT_TIMEOUT => $this->timeout,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode,
            'response' => $response,
            'error' => $error,
        ];
    }
    
    private function generateSignature(string $payload, int $timestamp): string
    {
        $signature = hash_hmac('sha256', $timestamp . '.' . $payload, $this->signingSecret);
        return 'sha256=' . $signature;
    }
    
    private function getRegisteredWebhooks(string $event): array
    {
        return Capsule::table('webhooks')
            ->where('event', $event)
            ->where('enabled', 1)
            ->pluck('url')
            ->toArray();
    }
}
```

### Webhook Receiver

```php
<?php
class WhmcsWebhookReceiver {
    private string $signingSecret;
    
    public function __construct(string $signingSecret)
    {
        $this->signingSecret = $signingSecret;
    }
    
    public function verify(?string $payload, ?string $signature, ?string $timestamp): bool
    {
        if (!$payload || !$signature || !$timestamp) {
            return false;
        }
        
        // Check timestamp is within acceptable window (5 minutes)
        if (abs(time() - (int) $timestamp) > 300) {
            return false;
        }
        
        $expectedSignature = 'sha256=' . hash_hmac(
            'sha256',
            $timestamp . '.' . $payload,
            $this->signingSecret
        );
        
        return hash_equals($expectedSignature, $signature);
    }
    
    public function parse(string $payload): ?array
    {
        $data = json_decode($payload, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            return null;
        }
        
        return $data;
    }
    
    public function handle(string $payload, string $signature, string $timestamp): WebhookResponse
    {
        if (!$this->verify($payload, $signature, $timestamp)) {
            return new WebhookResponse(401, 'Invalid signature');
        }
        
        $data = $this->parse($payload);
        
        if (!$data) {
            return new WebhookResponse(400, 'Invalid JSON payload');
        }
        
        try {
            $this->processEvent($data['event'], $data['data']);
            return new WebhookResponse(200, 'OK');
        } catch (Exception $e) {
            return new WebhookResponse(500, $e->getMessage());
        }
    }
    
    private function processEvent(string $event, array $data): void
    {
        $handlers = $this->getHandlers($event);
        
        foreach ($handlers as $handler) {
            $handler($data);
        }
    }
    
    private function getHandlers(string $event): array
    {
        return [
            'client.created' => [$this, 'handleClientCreated'],
            'invoice.paid' => [$this, 'handleInvoicePaid'],
            'service.created' => [$this, 'handleServiceCreated'],
        ];
    }
    
    private function handleClientCreated(array $data): void
    {
        // Process new client
        Log::info('New client created: ' . $data['client_id']);
    }
    
    private function handleInvoicePaid(array $data): void
    {
        // Process paid invoice
        Log::info('Invoice paid: ' . $data['invoice_id']);
    }
    
    private function handleServiceCreated(array $data): void
    {
        // Process new service
        Log::info('Service created: ' . $data['service_id']);
    }
}

class WebhookResponse {
    public function __construct(
        public readonly int $statusCode,
        public readonly string $message
    ) {}
}
```

## Available Webhook Events

### Client Events

```php
<?php
$clientEvents = [
    'ClientCreated',
    'ClientUpdated',
    'ClientDeleted',
    'ClientLogin',
    'ClientPasswordChange',
    'ClientEmailVerification',
];
```

### Invoice Events

```php
<?php
$invoiceEvents = [
    'InvoiceCreated',
    'InvoicePaid',
    'InvoiceCancelled',
    'InvoiceRefunded',
    'InvoiceOverdue',
    'InvoiceDeleted',
];
```

### Order Events

```php
<?php
$orderEvents = [
    'OrderCreated',
    'OrderAccepted',
    'OrderRejected',
    'OrderCancelled',
    'OrderFraud',
    'OrderPaid',
];
```

### Service Events

```php
<?php
$serviceEvents = [
    'ServiceCreated',
    'ServiceUpdated',
    'ServiceTerminated',
    'ServiceSuspended',
    'ServiceUnsuspended',
    'ServiceRenewed',
];
```

## Database Schema

```sql
CREATE TABLE `mod_webhooks` (
    `id` INT NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(255) NOT NULL,
    `url` VARCHAR(500) NOT NULL,
    `event` VARCHAR(100) NOT NULL,
    `secret` VARCHAR(255) NOT NULL,
    `enabled` TINYINT(1) NOT NULL DEFAULT 1,
    `retry_count` INT NOT NULL DEFAULT 3,
    `timeout` INT NOT NULL DEFAULT 30,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_event_enabled` (`event`, `enabled`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_webhook_logs` (
    `id` INT NOT NULL AUTO_INCREMENT,
    `webhook_id` INT NOT NULL,
    `event` VARCHAR(100) NOT NULL,
    `payload` TEXT NOT NULL,
    `response_code` INT NULL,
    `response_body` TEXT NULL,
    `attempts` INT NOT NULL DEFAULT 1,
    `status` ENUM('pending', 'success', 'failed') NOT NULL,
    `created_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`),
    INDEX `idx_webhook_status` (`webhook_id`, `status`),
    FOREIGN KEY (`webhook_id`) REFERENCES `mod_webhooks`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Best Practices

1. **Always verify signatures** - Validate webhook authenticity
2. **Handle idempotently** - Use event IDs to prevent duplicate processing
3. **Respond quickly** - Return 200 immediately, process asynchronously
4. **Log all webhooks** - Store for debugging and replay
5. **Implement retries** - Queue failed webhooks for retry
6. **Use HTTPS** - Never send webhooks over unencrypted connections

## Related Documentation

- [WHMCS API Authentication](/docs/whmcs-api-authentication.md)
- [WHMCS Cron Hooks](/docs/whmcs-cron-hooks.md)
- [WHMCS Client Hooks](/docs/whmcs-client-hooks.md)