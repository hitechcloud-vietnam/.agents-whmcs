# WHMCS Payment Gateway Workflow
# Version: 1.0 | Created: 2026-05-28

---

## Overview

This workflow guides the creation of WHMCS payment gateway modules for Vietnamese and international payment providers.

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md` (Technical Reference)
2. Read `Core_exapm_whmcs/sample-gateway-module/`
3. Read `.agents/docs/WHMCS_GATEWAY_TYPES.md`
4. Read `.agents/docs/VIETNAMESE_BILLING_REFERENCE.md` (for Vietnamese gateways)
5. Identify API documentation for the target provider

---

## Gateway Types

| Type | Description | Files |
|------|-------------|-------|
| Standard | Redirect to external payment page | `{module}.php` |
| Merchant | On-site card capture + redirect | `{module}.php` + callback |
| Remote Bank | Bank simulation/gateway | `{module}.php` + callback |
| Tokenization | Store card token for future charges | `{module}.php` |
| Remote Input | iframe hosted form | `{module}.php` + callback |

---

## Module Structure

```
modules/gateways/
├── {module}.php              ← Main gateway file
└── callback/
    └── {module}.php          ← IPN/callback handler
```

---

## Step-by-Step Development

### Step 1: Create Main Gateway File

Create `{module}.php`:

```php
<?php
/**
 * {Module} Payment Gateway for WHMCS
 * Version: 1.0.0
 * Author: HiTechCloud
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Gateway Configuration
 */
function {module}_config(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '{Payment Provider Name}',
        ],
        'description' => [
            'Type' => 'System',
            'Value' => '{Description of the payment method}',
        ],
        'apiEndpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type' => 'text',
            'Size' => '50',
            'Default' => 'https://api.paymentprovider.vn',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '20',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'testMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable for testing with sandbox environment',
        ],
    ];
}

/**
 * Payment Link (redirect to payment page)
 */
function {module}_link(array $params): string {
    // Configuration
    $apiEndpoint = $params['apiEndpoint'];
    $merchantId = $params['merchantId'];
    $apiKey = $params['apiKey'];
    $testMode = $params['testMode'];

    // Invoice data
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $returnUrl = $params['returnurl'];
    $cancelUrl = $params['cancelurl'];

    // Build payment request
    $orderId = 'INV' . $invoiceId . '_' . time();
    $amountInt = (int) round($amount * 100); // VND in cents

    $payload = [
        'merchant_id' => $merchantId,
        'order_id' => $orderId,
        'amount' => $amountInt,
        'currency' => 'VND',
        'return_url' => $returnUrl,
        'cancel_url' => $cancelUrl,
        'description' => 'Payment for Invoice #' . $invoiceId,
        'timestamp' => time(),
    ];

    // Generate signature
    $signature = generateSignature($payload, $apiKey);

    // Build form
    $form = '<form action="' . $apiEndpoint . '/payment" method="POST">';
    $form .= '<input type="hidden" name="merchant_id" value="' . htmlspecialchars($merchantId) . '">';
    $form .= '<input type="hidden" name="order_id" value="' . htmlspecialchars($orderId) . '">';
    $form .= '<input type="hidden" name="amount" value="' . $amountInt . '">';
    $form .= '<input type="hidden" name="currency" value="VND">';
    $form .= '<input type="hidden" name="return_url" value="' . htmlspecialchars($returnUrl) . '">';
    $form .= '<input type="hidden" name="cancel_url" value="' . htmlspecialchars($cancelUrl) . '">';
    $form .= '<input type="hidden" name="signature" value="' . htmlspecialchars($signature) . '">';

    $form .= '<button type="submit" class="btn btn-success">';
    $form .= 'Pay with {Payment Provider}';
    $form .= '</button>';
    $form .= '</form>';

    return $form;
}

/**
 * Refund (if supported)
 */
function {module}_refund(array $params): array {
    $transactionId = $params['transid'];
    $amount = $params['amount'];
    $apiKey = $params['apiKey'];
    $apiEndpoint = $params['apiEndpoint'];

    try {
        $result = refundTransaction($apiEndpoint, $apiKey, $transactionId, $amount);

        return [
            'status' => 'success',
            'transid' => $result['refund_id'],
            'raw' => json_encode($result),
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'failed',
            'error' => $e->getMessage(),
            'raw' => '',
        ];
    }
}

// Helper function: Generate signature
function generateSignature(array $data, string $apiKey): string {
    ksort($data);
    $signData = implode('&', array_map(
        fn($k, $v) => "$k=$v",
        array_keys($data),
        array_values($data)
    ));

    return hash_hmac('sha256', $signData, $apiKey);
}

// Helper function: Refund transaction
function refundTransaction(string $endpoint, string $apiKey, string $transId, float $amount): array {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $endpoint . '/refund',
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_POSTFIELDS => json_encode([
            'transaction_id' => $transId,
            'amount' => (int) round($amount * 100),
        ]),
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'Authorization: Bearer ' . $apiKey,
        ],
    ]);

    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}
```

### Step 2: Create Callback Handler

Create `callback/{module}.php`:

```php
<?php
/**
 * {Module} Payment Gateway Callback Handler
 * Version: 1.0.0
 * Author: HiTechCloud
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Get callback data
$response = $_POST; // or $_GET depending on provider

// Validate callback signature
$apiKey = ''; // Get from gateway configuration
$receivedSignature = $response['signature'] ?? '';

// Rebuild and verify signature
$payload = [
    'merchant_id' => $response['merchant_id'] ?? '',
    'order_id' => $response['order_id'] ?? '',
    'amount' => $response['amount'] ?? '',
    'status' => $response['status'] ?? '',
];
$expectedSignature = generateSignature($payload, $apiKey);

if ($receivedSignature !== $expectedSignature) {
    logTransaction('{Module}', $response, 'Invalid Signature');
    die('Invalid signature');
}

// Process based on status
$status = $response['status'] ?? '';
$orderId = $response['order_id'] ?? '';
$transactionId = $response['transaction_id'] ?? '';
$amount = ($response['amount'] ?? 0) / 100;

// Extract invoice ID from order ID
preg_match('/^INV(\d+)_/', $orderId, $matches);
$invoiceId = $matches[1] ?? 0;

switch ($status) {
    case 'completed':
    case 'success':
        // Payment successful
        addInvoicePayment(
            $invoiceId,
            $transactionId,
            $amount,
            0, // fees
            '{module}'
        );
        logTransaction('{Module}', $response, 'Success');
        break;

    case 'pending':
        // Payment pending
        logTransaction('{Module}', $response, 'Pending');
        break;

    case 'failed':
    case 'cancelled':
        // Payment failed
        logTransaction('{Module}', $response, 'Failed');
        break;

    default:
        logTransaction('{Module}', $response, 'Unknown Status: ' . $status);
}

// Redirect back to WHMCS
$systemUrl = \WHMCS\Config\Setting::getValue('SystemURL');
header('Location: ' . $systemUrl . '/viewinvoice.php?id=' . $invoiceId);

// Helper function: Generate signature
function generateSignature(array $data, string $apiKey): string {
    ksort($data);
    $signData = implode('&', array_map(
        fn($k, $v) => "$k=$v",
        array_keys($data),
        array_values($data)
    ));

    return hash_hmac('sha256', $signData, $apiKey);
}
```

---

## Vietnamese Payment Gateway Patterns

### payos_v2 Pattern

```php
// For Vietnamese payment gateways like payOS, VNPay, etc.

function {module}_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'payOS'],
        'apiKey' => ['FriendlyName' => 'API Key', 'Type' => 'text'],
        'checksumKey' => ['FriendlyName' => 'Checksum Key', 'Type' => 'password'],
        'clientId' => ['FriendlyName' => 'Client ID', 'Type' => 'text'],
        'testMode' => ['FriendlyName' => 'Sandbox Mode', 'Type' => 'yesno'],
    ];
}

// Signature generation for payOS
function payosSignature(array $data, string $checksumKey): string {
    $dataStr = implode('|', $data);
    return strtoupper(hash_hmac('sha256', $dataStr, $checksumKey));
}
```

---

## Checklist

- [ ] Gateway config function with all required fields
- [ ] link() function returning payment form
- [ ] refund() function (if supported)
- [ ] Callback handler with signature validation
- [ ] IP whitelist for callback (if provider provides IPs)
- [ ] logTransaction() calls for all status changes
- [ ] Test with sandbox environment

---

## Security Checklist

- [ ] Validate callback signature before processing
- [ ] IP whitelist for known provider IPs
- [ ] Replay attack prevention (check transaction ID not already processed)
- [ ] Sanitize all user inputs
- [ ] Use HTTPS for all API calls
- [ ] Log all transactions for debugging

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Callback not processing | Check IP whitelist and signature validation |
| Duplicate payments | Check transaction ID before processing |
| Amount mismatch | Verify currency conversion (VND often in cents) |
| Return URL not working | Check SSL and URL encoding |

---

Last updated: 2026-05-28