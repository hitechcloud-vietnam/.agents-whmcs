# WHMCS Callback Handler Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building secure and reliable payment gateway callback handlers (IPN listeners).

## When to Use

- Processing payment gateway callbacks
- Handling webhook notifications
- Verifying payment status updates

## Callback Structure

```
modules/gateways/callback/
└── {module}.php
```

## Building Callbacks

### Standard Callback Pattern

```php
<?php
// modules/gateways/callback/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

// Get raw input for signature verification
$rawInput = file_get_contents('php://input');
$response = $_POST;

if (empty($response)) {
    // Try JSON payload
    $response = json_decode($rawInput, true) ?? [];
}

// Validate callback
$config = getGatewayConfig();
$isValid = validateCallback($response, $config);

if (!$isValid) {
    logTransaction('{Module}', $response, 'Invalid Signature');
    http_response_code(400);
    exit('Invalid callback');
}

// Process payment
processPayment($response);

// Always respond with 200 to prevent retries
http_response_code(200);
exit('OK');
```

### Signature Verification

```php
function validateCallback(array $response, array $config): bool {
    $signature = $response['signature'] ?? '';

    // Rebuild signature for verification
    $data = [
        'merchant_id' => $response['merchant_id'] ?? '',
        'order_id' => $response['order_id'] ?? '',
        'amount' => $response['amount'] ?? '',
        'status' => $response['status'] ?? '',
    ];

    ksort($data);
    $signData = http_build_query($data);
    $expected = strtoupper(hash_hmac('sha256', $signData, $config['api_key']));

    return hash_equals($expected, $signature);
}
```

### Payment Processing

```php
function processPayment(array $response): void {
    $status = $response['status'] ?? '';
    $orderId = $response['order_id'] ?? '';
    $transactionId = $response['transaction_id'] ?? '';
    $amount = ($response['amount'] ?? 0) / 100; // Convert from cents

    // Extract invoice ID from order ID
    preg_match('/^INV(\d+)_/', $orderId, $matches);
    $invoiceId = (int) ($matches[1] ?? 0);

    if ($invoiceId <= 0) {
        logTransaction('{Module}', $response, 'Invalid Invoice ID');
        return;
    }

    // Check for duplicate
    if (isTransactionProcessed($transactionId)) {
        logTransaction('{Module}', $response, 'Duplicate Transaction');
        return;
    }

    switch ($status) {
        case 'completed':
        case 'success':
            // Payment successful
            addInvoicePayment(
                $invoiceId,
                $transactionId,
                $amount,
                0, // fees
                '{Module}'
            );
            logTransaction('{Module}', $response, 'Success');
            break;

        case 'pending':
            logTransaction('{Module}', $response, 'Pending');
            break;

        case 'failed':
        case 'cancelled':
            logTransaction('{Module}', $response, 'Failed');
            break;

        default:
            logTransaction('{Module}', $response, 'Unknown Status: ' . $status);
    }

    // Mark as processed
    markTransactionProcessed($transactionId);
}
```

### IP Whitelist

```php
function validateIpAddress(array $allowedIps): bool {
    $clientIp = $_SERVER['REMOTE_ADDR'];

    // Check for proxy headers
    if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
        $clientIp = $_SERVER['HTTP_X_FORWARDED_FOR'];
    }

    return in_array($clientIp, $allowedIps);
}

// Common payment provider IPs
$allowedIps = [
    '103.89.12.0/24',  // Example provider
    '10.0.0.0/8',      // Internal
];
```

### Database Tracking

```php
function isTransactionProcessed(string $transId): bool {
    return Capsule::table('mod_{module}_transactions')
        ->where('transaction_id', $transId)
        ->exists();
}

function markTransactionProcessed(string $transId, array $data = []): void {
    Capsule::table('mod_{module}_transactions')->insert([
        'transaction_id' => $transId,
        'invoice_id' => extractInvoiceId($data),
        'amount' => ($data['amount'] ?? 0) / 100,
        'status' => $data['status'] ?? 'unknown',
        'raw_response' => json_encode($data),
        'processed_at' => date('Y-m-d H:i:s'),
    ]);
}
```

## Checklist

- [ ] Handle both POST and JSON payloads
- [ ] Signature verification
- [ ] IP whitelist (if available)
- [ ] Duplicate transaction prevention
- [ ] All status cases handled
- [ ] logTransaction() for all outcomes
- [ ] Always return 200 to prevent retries
- [ ] Timeout handling

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-security-hardening
- whmcs-payment-validation