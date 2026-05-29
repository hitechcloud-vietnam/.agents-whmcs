# WHMCS 2Checkout Gateway Module - DEVKIT

## Module Information
- **Name**: 2Checkout Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: 2Checkout (Verifone) payment gateway integration

## Installation
1. Copy to `/modules/gateways/twocheckout/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## twocheckout.php
```php
<?php
/**
 * WHMCS 2Checkout Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function twocheckout_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '2Checkout'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => '2Checkout Payment Gateway'
        ],
        'account_number' => [
            'FriendlyName' => 'Account Number',
            'Type' => 'text',
            'Size' => '50',
            'Description' => '2Checkout Account Number'
        ],
        'secret_key' => [
            'FriendlyName' => 'Secret Key',
            'Type' => 'password',
            'Size' => '80'
        ],
        'publishable_key' => [
            'FriendlyName' => 'Publishable Key',
            'Type' => 'text',
            'Size' => '80'
        ],
        'secret_word' => [
            'FriendlyName' => 'Secret Word',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'For URL validation'
        ],
        'sandbox' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox environment'
        ],
        'demo' => [
            'FriendlyName' => 'Demo Mode',
            'Type' => 'yesno',
            'Description' => 'Enable demo mode (no real charges)'
        ]
    ];
}

function twocheckout_capture($params)
{
    $accountNumber = $params['account_number'];
    $publishableKey = $params['publishable_key'];
    $secretKey = $params['secret_key'];
    
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $email = $params['clientdetails']['email'] ?? '';
    
    $mode = $params['sandbox'] ? 'sandbox' : 'production';
    
    // Create payment token
    $tokenPayload = [
        'publiKey' => $publishableKey,
        'order' => [
            'currency' => $currency,
            'amount' => number_format($amount, 2, '.', ''),
            'orderExt' => [
                'customUuid' => 'INV_' . $invoiceId . '_' . time()
            ]
        ]
    ];
    
    $ch = curl_init('https://apis.2checkout.com/tokens');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($tokenPayload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $tokenResult = json_decode($response, true);
    
    if (isset($tokenResult['token'])) {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'token' => $tokenResult['token'],
            'reference' => $tokenResult['token']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => $tokenResult['error'] ?? 'Failed to create token'
    ];
}

function twocheckout_callback($params)
{
    $secretWord = $params['secret_word'];
    $orderId = $_GET['order_id'] ?? $_POST['order_id'] ?? '';
    $total = $_GET['total'] ?? $_POST['total'] ?? 0;
    $key = $_GET['key'] ?? $_POST['key'] ?? '';
    
    // Validate MD5 hash
    if ($params['demo']) {
        $expectedKey = strtoupper(md5($secretWord));
    } else {
        $expectedKey = strtoupper(md5($params['account_number'] . $orderId . $total . $secretWord));
    }
    
    if ($key !== $expectedKey && $key !== strtoupper(md5($params['account_number'] . $_GET['ORDERID'] . $_GET['TOTAL'] . $secretWord))) {
        return ['status' => 'error', 'rawdata' => 'Invalid hash'];
    }
    
    $invoiceId = str_replace('INV_', '', $_GET['order_id'] ?? $_GET['ORDERID'] ?? '');
    $invoiceId = explode('_', $invoiceId)[0];
    
    return [
        'status' => 'success',
        'transid' => $_GET['transactionId'] ?? $_GET['TRXNIDENTIFIER'] ?? $orderId,
        'amount' => $total,
        'rawdata' => json_encode($_REQUEST)
    ];
}

function twocheckout_refund($params)
{
    $accountNumber = $params['account_number'];
    $secretKey = $params['secret_key'];
    
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    
    $payload = [
        'sale_id' => $transactionId,
        'amount' => number_format($amount, 2, '.', ''),
        'currency' => $params['currency']
    ];
    
    // Build authorization header
    $auth = base64_encode($accountNumber . ':' . $secretKey);
    
    $ch = curl_init('https://api.2checkout.com/orders');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Basic ' . $auth
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

function twocheckout_query($params)
{
    $accountNumber = $params['account_number'];
    $secretKey = $params['secret_key'];
    
    $transactionId = $params['reference'];
    $auth = base64_encode($accountNumber . ':' . $secretKey);
    
    $ch = curl_init('https://api.2checkout.com/orders/' . $transactionId);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Basic ' . $auth
    ]);
    
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
    if ($vars['paymentmethod'] == 'twocheckout') {
        logActivity("2Checkout payment confirmed: " . ($vars['transid'] ?? 'N/A'));
    }
});
```