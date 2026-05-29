# WHMCS Razorpay Gateway Module - DEVKIT

## Module Information
- **Name**: Razorpay Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Razorpay payment gateway for India and international payments

## Installation
1. Copy to `/modules/gateways/razorpay/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## razorpay.php
```php
<?php
/**
 * WHMCS Razorpay Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function razorpay_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Razorpay'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Razorpay Payment Gateway for India'
        ],
        'key_id' => [
            'FriendlyName' => 'Key ID',
            'Type' => 'text',
            'Size' => '80',
            'Description' => 'Razorpay Key ID'
        ],
        'key_secret' => [
            'FriendlyName' => 'Key Secret',
            'Type' => 'password',
            'Size' => '80'
        ],
        'webhook_secret' => [
            'FriendlyName' => 'Webhook Secret',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'For webhook signature verification'
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable test mode'
        ]
    ];
}

function razorpay_capture($params)
{
    $keyId = $params['key_id'];
    $keySecret = $params['key_secret'];
    
    $invoiceId = $params['invoiceid'];
    $amount = (int)round($params['amount'] * 100); // Convert to paise
    $currency = $params['currency'];
    $email = $params['clientdetails']['email'] ?? '';
    
    // Create order
    $payload = [
        'amount' => $amount,
        'currency' => $currency,
        'receipt' => 'INV_' . $invoiceId,
        'notes' => [
            'invoice_id' => $invoiceId,
            'WHMCS' => true
        ]
    ];
    
    $ch = curl_init('https://api.razorpay.com/v1/orders');
    curl_setopt($ch, CURLOPT_USERPWD, $keyId . ':' . $keySecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'order_id' => $result['id'],
            'amount' => $result['amount'],
            'reference' => $result['id']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => $result['error']['description'] ?? 'Failed to create order'
    ];
}

function razorpay_callback($params)
{
    $webhookSecret = $params['webhook_secret'];
    
    $payload = file_get_contents('php://input');
    $signature = $_SERVER['HTTP_X_RAZORPAY_SIGNATURE'] ?? '';
    
    // Verify webhook signature
    $expectedSignature = hash_hmac('sha256', $payload, $webhookSecret);
    
    if ($signature !== $expectedSignature) {
        return ['status' => 'error', 'rawdata' => 'Invalid signature'];
    }
    
    $event = json_decode($payload, true);
    
    if ($event['event'] == 'payment.captured') {
        $payment = $event['payload']['payment']['entity'];
        
        return [
            'status' => 'success',
            'transid' => $payment['id'],
            'amount' => $payment['amount'] / 100,
            'rawdata' => json_encode($event)
        ];
    }
    
    if ($event['event'] == 'payment.failed') {
        return [
            'status' => 'declined',
            'rawdata' => json_encode($event)
        ];
    }
    
    return ['status' => 'pending'];
}

function razorpay_refund($params)
{
    $keyId = $params['key_id'];
    $keySecret = $params['key_secret'];
    
    $transactionId = $params['transactionId'];
    $amount = (int)round($params['amount'] * 100);
    
    $payload = [
        'amount' => $amount,
        'receipt' => 'REF_' . time()
    ];
    
    $ch = curl_init('https://api.razorpay.com/v1/payments/' . $transactionId . '/refund');
    curl_setopt($ch, CURLOPT_USERPWD, $keyId . ':' . $keySecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        return [
            'status' => 'success',
            'refund_id' => $result['id']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => $result['error']['description'] ?? 'Refund failed'
    ];
}

function razorpay_query($params)
{
    $keyId = $params['key_id'];
    $keySecret = $params['key_secret'];
    
    $transactionId = $params['reference'];
    
    $ch = curl_init('https://api.razorpay.com/v1/payments/' . $transactionId);
    curl_setopt($ch, CURLOPT_USERPWD, $keyId . ':' . $keySecret);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('InvoicePaid', 1, function($vars) {
    if ($vars['paymentmethod'] == 'razorpay') {
        logActivity("Razorpay payment confirmed: " . ($vars['transid'] ?? 'N/A'));
    }
});
```