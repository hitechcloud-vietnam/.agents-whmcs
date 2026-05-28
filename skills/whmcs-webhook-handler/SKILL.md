# WHMCS Webhook Handler Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building and handling webhooks in WHMCS modules.

## When to Use

- Receiving external callbacks
- Processing payment notifications
- Handling domain registrar webhooks

## Webhook Structure

```
modules/addons/{module}/
└── webhook.php              ← Webhook endpoint
```

## Building Webhooks

### Basic Webhook Handler

```php
<?php
// modules/addons/{module}/webhook.php
if (!defined("WHMCS")) { die("Direct access denied"); }

// Get webhook data
$payload = file_get_contents('php://input');
$data = json_decode($payload, true) ?? $_POST;

// Validate webhook
$secret = getWebhookSecret();
if (!validateWebhookSignature($payload, $_SERVER['HTTP_X_SIGNATURE'], $secret)) {
    http_response_code(401);
    exit('Unauthorized');
}

// Process webhook
$event = $data['event'] ?? '';

switch ($event) {
    case 'payment.completed':
        handlePaymentCompleted($data);
        break;
    case 'service.created':
        handleServiceCreated($data);
        break;
    case 'domain.renewed':
        handleDomainRenewed($data);
        break;
    default:
        logUnknownEvent($event, $data);
}

// Always respond quickly
http_response_code(200);
echo json_encode(['status' => 'received']);
```

### Signature Validation

```php
function validateWebhookSignature(string $payload, ?string $signature, string $secret): bool {
    if (empty($signature)) {
        return false;
    }

    $expected = hash_hmac('sha256', $payload, $secret);
    return hash_equals($expected, $signature);
}
```

### Event Processing

```php
function handlePaymentCompleted(array $data): void {
    $transactionId = $data['transaction_id'] ?? '';
    $invoiceId = $data['invoice_id'] ?? 0;
    $amount = $data['amount'] ?? 0;

    // Prevent duplicates
    if (Capsule::table('mod_{module}_webhooks')
        ->where('event_id', $data['id'])
        ->exists()) {
        return;
    }

    // Process payment
    addInvoicePayment($invoiceId, $transactionId, $amount, 0, '{Module}');

    // Log webhook
    Capsule::table('mod_{module}_webhooks')->insert([
        'event_id' => $data['id'],
        'event_type' => 'payment.completed',
        'payload' => json_encode($data),
        'processed_at' => date('Y-m-d H:i:s'),
    ]);
}

function logUnknownEvent(string $event, array $data): void {
    Capsule::table('mod_{module}_webhooks')->insert([
        'event_id' => $data['id'] ?? '',
        'event_type' => $event,
        'payload' => json_encode($data),
        'status' => 'unknown_event',
        'processed_at' => date('Y-m-d H:i:s'),
    ]);

    logActivity('Unknown webhook event: ' . $event);
}
```

## WHMCS Webhook System

```php
// Register custom webhook
add_hook('CustomWebhook', 1, function($vars) {
    return handleCustomWebhook($vars);
});

// WHMCS webhook endpoints
// /api/v1/webhooks/...
```

## Checklist

- [ ] Webhook endpoint file
- [ ] Signature validation
- [ ] Event routing
- [ ] Idempotent processing
- [ ] Error handling
- [ ] Logging

---

**Related Skills:**
- whmcs-callback-handler
- whmcs-hooks-development
- whmcs-security-hardening