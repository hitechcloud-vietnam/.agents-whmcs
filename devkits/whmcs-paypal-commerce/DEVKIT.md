# WHMCS PayPal Commerce Gateway Module - DEVKIT

## Module Information
- **Name**: PayPal Commerce
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: PayPal Commerce Platform integration for WHMCS

## Installation
1. Copy to `/modules/gateways/paypal_commerce/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## paypal_commerce.php
```php
<?php
/**
 * WHMCS PayPal Commerce Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function paypal_commerce_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'PayPal Commerce'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'PayPal Commerce Platform for split payments'
        ],
        'client_id' => [
            'FriendlyName' => 'Client ID',
            'Type' => 'text',
            'Size' => '80',
            'Description' => 'PayPal App Client ID'
        ],
        'client_secret' => [
            'FriendlyName' => 'Client Secret',
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'PayPal App Client Secret'
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'sandbox' => 'Sandbox',
                'live' => 'Live'
            ],
            'Default' => 'sandbox'
        ],
        'webhook_id' => [
            'FriendlyName' => 'Webhook ID',
            'Type' => 'text',
            'Size' => '80',
            'Description' => 'PayPal webhook ID for IPN'
        ],
        'receiver_email' => [
            'FriendlyName' => 'Primary Receiver Email',
            'Type' => 'text',
            'Size' => '80',
            'Description' => 'Primary PayPal email for receiving payments'
        ]
    ];
}

function paypal_commerce_capture($params)
{
    $clientId = $params['client_id'];
    $clientSecret = $params['client_secret'];
    $environment = $params['environment'];
    
    $baseUrl = $environment == 'live' 
        ? 'https://api-m.paypal.com' 
        : 'https://api-m.sandbox.paypal.com';
    
    // Get access token
    $ch = curl_init($baseUrl . '/v1/oauth2/token');
    curl_setopt($ch, CURLOPT_USERPWD, $clientId . ':' . $clientSecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, 'grant_type=client_credentials');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/x-www-form-urlencoded']);
    
    $tokenResponse = curl_exec($ch);
    curl_close($ch);
    
    $tokenData = json_decode($tokenResponse, true);
    $accessToken = $tokenData['access_token'] ?? '';
    
    if (empty($accessToken)) {
        return ['status' => 'failed', 'error' => 'Failed to obtain access token'];
    }
    
    // Create order
    $orderData = [
        'intent' => 'CAPTURE',
        'purchase_units' => [
            [
                'reference_id' => 'INV_' . $params['invoiceid'],
                'description' => 'Invoice #' . $params['invoiceid'],
                'amount' => [
                    'currency_code' => $params['currency'],
                    'value' => number_format($params['amount'], 2, '.', '')
                ]
            ]
        ]
    ];
    
    $ch = curl_init($baseUrl . '/v2/checkout/orders');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($orderData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $accessToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        // Find approval URL
        $approveUrl = '';
        foreach ($result['links'] as $link) {
            if ($link['rel'] == 'approve') {
                $approveUrl = $link['href'];
                break;
            }
        }
        
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'redirecturl' => $approveUrl,
            'reference' => $result['id']
        ];
    } else {
        return [
            'status' => 'failed',
            'error' => $result['message'] ?? 'Failed to create order'
        ];
    }
}

function paypal_commerce_callback($params)
{
    $clientId = $params['client_id'];
    $clientSecret = $params['client_secret'];
    $environment = $params['environment'];
    
    $baseUrl = $environment == 'live' 
        ? 'https://api-m.paypal.com' 
        : 'https://api-m.sandbox.paypal.com';
    
    $tokenResponse = file_get_contents($baseUrl . '/v1/oauth2/token');
    // Note: In production, implement proper OAuth flow
    
    // Get order ID from return URL
    $orderId = $_GET['token'] ?? $_GET['orderID'] ?? '';
    
    if (empty($orderId)) {
        return ['status' => 'error', 'rawdata' => 'No order ID received'];
    }
    
    // Get access token
    $ch = curl_init($baseUrl . '/v1/oauth2/token');
    curl_setopt($ch, CURLOPT_USERPWD, $clientId . ':' . $clientSecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, 'grant_type=client_credentials');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/x-www-form-urlencoded']);
    
    $tokenResponse = curl_exec($ch);
    curl_close($ch);
    
    $tokenData = json_decode($tokenResponse, true);
    $accessToken = $tokenData['access_token'] ?? '';
    
    // Capture the order
    $ch = curl_init($baseUrl . '/v2/checkout/orders/' . $orderId . '/capture');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $accessToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['status']) && $result['status'] == 'COMPLETED') {
        $purchaseUnit = $result['purchase_units'][0] ?? [];
        $payments = $purchaseUnit['payments']['captures'][0] ?? [];
        
        return [
            'status' => 'success',
            'transid' => $payments['id'] ?? $result['id'],
            'amount' => $result['purchase_units'][0]['amount']['value'] ?? 0,
            'rawdata' => json_encode($result)
        ];
    } else {
        return [
            'status' => 'declined',
            'rawdata' => json_encode($result)
        ];
    }
}

function paypal_commerce_refund($params)
{
    $clientId = $params['client_id'];
    $clientSecret = $params['client_secret'];
    $environment = $params['environment'];
    
    $baseUrl = $environment == 'live' 
        ? 'https://api-m.paypal.com' 
        : 'https://api-m.sandbox.paypal.com';
    
    // Get access token
    $ch = curl_init($baseUrl . '/v1/oauth2/token');
    curl_setopt($ch, CURLOPT_USERPWD, $clientId . ':' . $clientSecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, 'grant_type=client_credentials');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/x-www-form-urlencoded']);
    
    $tokenResponse = curl_exec($ch);
    curl_close($ch);
    
    $tokenData = json_decode($tokenResponse, true);
    $accessToken = $tokenData['access_token'] ?? '';
    
    // Refund capture
    $refundData = [
        'amount' => [
            'currency_code' => $params['currency'],
            'value' => number_format($params['amount'], 2, '.', '')
        ]
    ];
    
    $ch = curl_init($baseUrl . '/v2/payments/captures/' . $params['transactionId'] . '/refund');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($refundData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json',
        'Authorization: Bearer ' . $accessToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['id'])) {
        return ['status' => 'success', 'refund_id' => $result['id']];
    } else {
        return ['status' => 'failed', 'error' => $result['message'] ?? 'Refund failed'];
    }
}

function paypal_commerce_query($params)
{
    $clientId = $params['client_id'];
    $clientSecret = $params['client_secret'];
    $environment = $params['environment'];
    
    $baseUrl = $environment == 'live' 
        ? 'https://api-m.paypal.com' 
        : 'https://api-m.sandbox.paypal.com';
    
    // Get access token
    $ch = curl_init($baseUrl . '/v1/oauth2/token');
    curl_setopt($ch, CURLOPT_USERPWD, $clientId . ':' . $clientSecret);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, 'grant_type=client_credentials');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/x-www-form-urlencoded']);
    
    $tokenResponse = curl_exec($ch);
    curl_close($ch);
    
    $tokenData = json_decode($tokenResponse, true);
    $accessToken = $tokenData['access_token'] ?? '';
    
    // Get order details
    $ch = curl_init($baseUrl . '/v2/checkout/orders/' . $params['reference']);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $accessToken
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
    if ($vars['paymentmethod'] == 'paypal_commerce') {
        logActivity("PayPal Commerce payment confirmed for Invoice #" . $vars['invoiceid']);
    }
});

add_hook('DailyCronJob', 1, function($vars) {
    PayPalHelper::syncTransactions();
});

class PayPalHelper
{
    public static function syncTransactions()
    {
        // Sync PayPal transactions with WHMCS
    }
}
```