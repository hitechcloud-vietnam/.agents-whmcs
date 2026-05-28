# WHMCS Webhook Integration Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing webhook integrations.

## When to Use

- Third-party integrations
- Event notifications
- Automated workflows

## Webhook Structure

```php
class WebhookHandler {
    private string $secret;

    public function handleIncoming(string $payload, string $signature): array {
        if (!$this->verifySignature($payload, $signature)) {
            throw new \Exception('Invalid signature');
        }

        $data = json_decode($payload, true);
        return $this->processEvent($data);
    }

    private function verifySignature(string $payload, string $signature): bool {
        $expected = hash_hmac('sha256', $payload, $this->secret);
        return hash_equals($expected, $signature);
    }

    private function processEvent(array $data): array {
        $event = $data['event'] ?? 'unknown';

        return match ($event) {
            'order.created' => $this->handleOrderCreated($data),
            'order.updated' => $this->handleOrderUpdated($data),
            'payment.received' => $this->handlePayment($data),
            'service.created' => $this->handleServiceCreated($data),
            default => $this->handleUnknown($data),
        };
    }
}
```

## Outgoing Webhooks

```php
function sendWebhook(string $url, string $event, array $data): bool {
    $payload = json_encode([
        'event' => $event,
        'timestamp' => time(),
        'data' => $data,
    ]);

    $signature = hash_hmac('sha256', $payload, $this->secret);

    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => $payload,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'X-Webhook-Signature: ' . $signature,
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
    ]);

    $result = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    return $httpCode >= 200 && $httpCode < 300;
}
```

## Webhook Retry Logic

```php
function retryWebhook(int $webhookId): void {
    $webhook = Capsule::table('mod_webhook_queue')->where('id', $webhookId)->first();

    if ($webhook->attempts >= 5) {
        Capsule::table('mod_webhook_queue')
            ->where('id', $webhookId)
            ->update(['status' => 'failed']);
        return;
    }

    $result = $this->sendWebhook($webhook->url, $webhook->event, json_decode($webhook->payload, true));

    Capsule::table('mod_webhook_queue')
        ->where('id', $webhookId)
        ->update([
            'attempts' => $webhook->attempts + 1,
            'last_attempt' => date('Y-m-d H:i:s'),
            'last_result' => $result ? 'success' : 'failed',
        ]);

    if (!$result && $webhook->attempts < 3) {
        // Schedule retry
        $this->scheduleRetry($webhookId, pow(2, $webhook->attempts) * 60);
    }
}
```

---

**Related Skills:**
- whmcs-webhook-handler
- whmcs-api-integration
