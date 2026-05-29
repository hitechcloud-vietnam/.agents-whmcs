# WHMCS Custom Gateway Module - DEVKIT

## Module Information
- **Name**: Custom Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Template for building custom payment gateways

## Installation
1. Copy to `/modules/gateways/custom_gateway/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## custom_gateway.php
```php
<?php
/**
 * WHMCS Custom Payment Gateway Template
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway configuration
 */
function custom_gateway_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Custom Gateway'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Custom payment gateway integration'
        ],
        'api_url' => [
            'FriendlyName' => 'API URL',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Payment gateway API URL'
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '80'
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '80'
        ],
        'merchant_id' => [
            'FriendlyName' => 'Merchant ID',
            'Type' => 'text',
            'Size' => '50'
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox/test mode'
        ],
        'timeout' => [
            'FriendlyName' => 'Request Timeout',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30'
        ]
    ];
}

/**
 * Capture payment - redirect to payment page
 */
function custom_gateway_capture($params)
{
    $apiUrl = $params['api_url'];
    $apiKey = $params['api_key'];
    $merchantId = $params['merchant_id'];
    $testMode = $params['test_mode'];
    
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $clientEmail = $params['clientdetails']['email'] ?? '';
    
    // Prepare payment request
    $payload = [
        'merchant_id' => $merchantId,
        'order_id' => 'INV_' . $invoiceId . '_' . time(),
        'amount' => $amount,
        'currency' => $currency,
        'customer_email' => $clientEmail,
        'return_url' => $params['returnurl'],
        'cancel_url' => $params['cancelurl'],
        'callback_url' => $params['systemurl'] . '/modules/gateways/callback/custom_gateway.php'
    ];
    
    // Add signature
    $payload['signature'] = custom_gateway_create_signature($payload, $params['api_secret']);
    
    // Make API request
    $ch = curl_init($apiUrl . '/create_payment');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $apiKey
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['payment_url'])) {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'redirecturl' => $result['payment_url'],
            'reference' => $result['order_id'] ?? $payload['order_id']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => $result['error'] ?? 'Failed to create payment'
    ];
}

/**
 * Process callback from payment gateway
 */
function custom_gateway_callback($params)
{
    // Verify callback signature
    $signature = $_POST['signature'] ?? $_GET['signature'] ?? '';
    
    $callbackData = [
        'order_id' => $_POST['order_id'] ?? $_GET['order_id'] ?? '',
        'status' => $_POST['status'] ?? $_GET['status'] ?? '',
        'amount' => $_POST['amount'] ?? $_GET['amount'] ?? 0,
        'transaction_id' => $_POST['transaction_id'] ?? $_GET['transaction_id'] ?? ''
    ];
    
    $expectedSignature = custom_gateway_create_signature($callbackData, $params['api_secret']);
    
    if ($signature !== $expectedSignature) {
        return ['status' => 'error', 'rawdata' => 'Invalid signature'];
    }
    
    // Process based on status
    $status = strtolower($callbackData['status']);
    
    if ($status == 'completed' || $status == 'success') {
        return [
            'status' => 'success',
            'transid' => $callbackData['transaction_id'],
            'amount' => $callbackData['amount'],
            'rawdata' => json_encode($callbackData)
        ];
    }
    
    if ($status == 'pending' || $status == 'processing') {
        return ['status' => 'pending'];
    }
    
    return [
        'status' => 'declined',
        'rawdata' => json_encode($callbackData)
    ];
}

/**
 * Refund payment
 */
function custom_gateway_refund($params)
{
    $apiUrl = $params['api_url'];
    $apiKey = $params['api_key'];
    
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    $invoiceId = $params['invoiceid'];
    
    $payload = [
        'original_transaction_id' => $transactionId,
        'refund_amount' => $amount,
        'reason' => 'Customer request - Invoice #' . $invoiceId
    ];
    
    $payload['signature'] = custom_gateway_create_signature($payload, $params['api_secret']);
    
    $ch = curl_init($apiUrl . '/refund');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $apiKey
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['refund_id'])) {
        return [
            'status' => 'success',
            'refund_id' => $result['refund_id']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => $result['error'] ?? 'Refund failed'
    ];
}

/**
 * Query transaction status
 */
function custom_gateway_query($params)
{
    $apiUrl = $params['api_url'];
    $apiKey = $params['api_key'];
    
    $transactionId = $params['reference'];
    
    $ch = curl_init($apiUrl . '/transaction/' . $transactionId);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiKey
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}

/**
 * Create HMAC signature for API requests
 */
function custom_gateway_create_signature($data, $secret)
{
    // Sort and encode data
    ksort($data);
    
    $encodedData = [];
    foreach ($data as $key => $value) {
        if ($key !== 'signature') {
            $encodedData[] = $key . '=' . $value;
        }
    }
    
    $stringToSign = implode('&', $encodedData);
    
    return hash_hmac('sha256', $stringToSign, $secret);
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('InvoicePaid', 1, function($vars) {
    if ($vars['paymentmethod'] == 'custom_gateway') {
        logActivity("Custom Gateway payment confirmed: " . ($vars['transid'] ?? 'N/A'));
    }
});
```