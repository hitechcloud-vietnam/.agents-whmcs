# WHMCS Payment Gateway Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building WHMCS payment gateway modules for various payment providers.

## When to Use

- Creating new payment gateway integration
- Converting existing payment processor to WHMCS module
- Adding support for Vietnamese payment providers (VNPay, MoMo, payOS, etc.)

## Gateway Types

| Type | Pattern | Use Case |
|------|---------|----------|
| Standard | Redirect to payment page | Bank transfer, e-wallet |
| Merchant | Card capture + process | Credit card on-site |
| Tokenization | Store token + charge later | Subscriptions |
| Remote Input | iframe hosted form | PCI-compliant cards |

## Building Steps

### Step 1: Determine Gateway Type

```php
// Standard redirect gateway - most common
function gateway_link(array $params): string {
    // Return HTML form redirecting to payment page
}

// Merchant gateway - card capture
function gateway_capture_form(array $params): array {
    // Return card capture form HTML
}

function gateway_capture(array $params): array {
    // Process payment and return result
}

// Tokenization - store card for future
function gateway_capture_token(array $params): array {
    // Store card token
}
```

### Step 2: Create Standard Gateway

```php
<?php
// modules/gateways/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => '{Provider Name}'],
        'description' => ['Type' => 'System', 'Value' => '{Description}'],
        'apiEndpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type' => 'text',
            'Size' => '60',
            'Default' => 'https://api.provider.com',
        ],
        'merchantId' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '30',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'testMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}

function {module}_link(array $params): string {
    $endpoint = $params['apiEndpoint'];
    $merchantId = $params['merchantId'];
    $apiKey = $params['apiKey'];

    $orderId = 'INV' . $params['invoiceid'] . '_' . time();
    $amount = (int) round($params['amount'] * 100); // VND
    $returnUrl = $params['returnurl'];
    $cancelUrl = $params['cancelurl'];

    $payload = [
        'merchant_id' => $merchantId,
        'order_id' => $orderId,
        'amount' => $amount,
        'currency' => 'VND',
        'return_url' => $returnUrl,
        'cancel_url' => $cancelUrl,
    ];

    $signature = generateSignature($payload, $apiKey);
    $payload['signature'] = $signature;

    $form = '<form action="' . $endpoint . '/payment" method="POST">';
    foreach ($payload as $key => $value) {
        $form .= '<input type="hidden" name="' . $key . '" value="' . htmlspecialchars($value) . '">';
    }
    $form .= '<button type="submit" class="btn btn-success">Pay Now</button>';
    $form .= '</form>';

    return $form;
}

function {module}_refund(array $params): array {
    try {
        $apiKey = $params['apiKey'];
        $endpoint = $params['apiEndpoint'];
        $transId = $params['transid'];
        $amount = (int) round($params['amount'] * 100);

        $result = refundPayment($endpoint, $apiKey, $transId, $amount);

        return [
            'status' => 'success',
            'transid' => $result['refund_id'],
        ];
    } catch (\Exception $e) {
        return ['status' => 'failed', 'error' => $e->getMessage()];
    }
}

function generateSignature(array $data, string $key): string {
    ksort($data);
    $signData = http_build_query($data);
    return strtoupper(hash_hmac('sha256', $signData, $key));
}

function refundPayment(string $endpoint, string $key, string $transId, int $amount): array {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $endpoint . '/refund',
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'transaction_id' => $transId,
            'amount' => $amount,
        ]),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $key,
            'Content-Type: application/json',
        ],
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
    return json_decode($response, true) ?? [];
}
```

### Step 3: Create Callback Handler

```php
<?php
// modules/gateways/callback/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

$response = $_POST;
$apiKey = ''; // Get from gateway settings

$receivedSignature = $response['signature'] ?? '';
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

$status = $response['status'] ?? '';
preg_match('/^INV(\d+)_/', $response['order_id'] ?? '', $matches);
$invoiceId = $matches[1] ?? 0;
$transactionId = $response['transaction_id'] ?? '';
$amount = ($response['amount'] ?? 0) / 100;

if ($status === 'completed' || $status === 'success') {
    addInvoicePayment($invoiceId, $transactionId, $amount, 0, '{module}');
    logTransaction('{Module}', $response, 'Success');
} elseif ($status === 'pending') {
    logTransaction('{Module}', $response, 'Pending');
} else {
    logTransaction('{Module}', $response, 'Failed');
}

$systemUrl = \WHMCS\Config\Setting::getValue('SystemURL');
header('Location: ' . $systemUrl . '/viewinvoice.php?id=' . $invoiceId);
```

## Vietnamese Payment Patterns

### VNPay
```php
// VNPay signature pattern
function vnpaySignature(array $data, string $secret): string {
    $signData = implode('|', [
        $data['vnp_TxnRef'],
        $data['vnp_Amount'],
        $data['vnp_BankCode'] ?? '',
        $data['vnp_BankTranNo'] ?? '',
        $data['vnp_CardType'] ?? '',
        $data['vnp_OrderInfo'],
        $data['vnp_PayDate'],
        $data['vnp_ResponseCode'],
        $data['vnp_TmnCode'],
        $data['vnp_TransactionNo'] ?? '',
        $data['vnp_TransactionStatus'],
        $data['vnp_VnpSecureHash'],
    ]);
    return strtoupper(hash_hmac('sha512', $signData, $secret));
}
```

### payOS
```php
// payOS signature pattern
function payosSignature(array $data, string $checksumKey): string {
    $dataStr = implode('|', array_values($data));
    return hash_hmac('sha256', $dataStr, $checksumKey);
}
```

## Checklist

- [ ] config() with all required fields
- [ ] link() returns HTML form
- [ ] refund() processes refunds
- [ ] callback/{module}.php handles IPN
- [ ] Signature validation
- [ ] IP whitelist (if available)
- [ ] logTransaction() for all status
- [ ] Test mode support

---

**Related Skills:**
- whmcs-gateway-security
- whmcs-callback-handler
- whmcs-vietnamese-payments