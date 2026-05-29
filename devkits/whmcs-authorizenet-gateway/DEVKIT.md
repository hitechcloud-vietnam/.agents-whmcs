# WHMCS Authorize.net Gateway Module - DEVKIT

## Module Information
- **Name**: Authorize.net Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Authorize.net payment gateway integration

## Installation
1. Copy to `/modules/gateways/authorizenet/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## authorizenet.php
```php
<?php
/**
 * WHMCS Authorize.net Payment Gateway
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function authorizenet_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Authorize.net'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Authorize.net Payment Gateway'
        ],
        'api_login' => [
            'FriendlyName' => 'API Login ID',
            'Type' => 'text',
            'Size' => '50'
        ],
        'transaction_key' => [
            'FriendlyName' => 'Transaction Key',
            'Type' => 'password',
            'Size' => '80'
        ],
        'sandbox' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
            'Description' => 'Use sandbox environment'
        ],
        'client_key' => [
            'FriendlyName' => 'Client Key',
            'Type' => 'textarea',
            'Description' => 'For Accept.js integration'
        ]
    ];
}

function authorizenet_capture($params)
{
    $apiLogin = $params['api_login'];
    $transactionKey = $params['transaction_key'];
    $sandbox = $params['sandbox'];
    
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    $email = $params['clientdetails']['email'] ?? '';
    
    $baseUrl = $sandbox 
        ? 'https://apitest.authorize.net/xml/v1/request.api' 
        : 'https://api.authorize.net/xml/v1/request.api';
    
    // Create customer profile
    $customerProfile = [
        'merchantAuthentication' => [
            'name' => $apiLogin,
            'transactionKey' => $transactionKey
        ],
        'profile' => [
            'email' => $email
        ]
    ];
    
    // For simplicity, we'll use direct payment
    $paymentNonce = $_POST['authorizenet_nonce'] ?? '';
    
    if (empty($paymentNonce)) {
        return [
            'status' => 'failed',
            'error' => 'No payment token provided'
        ];
    }
    
    $payload = [
        'createTransactionRequest' => [
            'merchantAuthentication' => [
                'name' => $apiLogin,
                'transactionKey' => $transactionKey
            ],
            'transactionRequest' => [
                'transactionType' => 'authCaptureTransaction',
                'amount' => number_format($amount, 2, '.', ''),
                'currencyCode' => $currency,
                'payment' => [
                    'opaqueData' => [
                        'dataDescriptor' => 'COMMON.ACCEPT.INAPP.PAYMENT',
                        'dataValue' => $paymentNonce
                    ]
                ],
                'lineItems' => [
                    'lineItem' => [
                        'itemId' => '1',
                        'name' => 'Invoice #' . $invoiceId,
                        'quantity' => '1',
                        'unitPrice' => number_format($amount, 2, '.', '')
                    ]
                ],
                'order' => [
                    'invoiceNumber' => 'INV_' . $invoiceId
                ]
            ]
        ]
    ];
    
    $ch = curl_init($baseUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['transactionResponse']) && $result['transactionResponse']['responseCode'] == '1') {
        return [
            'status' => 'success',
            'transid' => $result['transactionResponse']['transId'],
            'amount' => $amount,
            'rawdata' => $response
        ];
    }
    
    $errorCode = $result['transactionResponse']['errors'][0]['errorCode'] ?? '';
    $errorMsg = $result['transactionResponse']['errors'][0]['errorText'] ?? 'Transaction failed';
    
    return [
        'status' => 'declined',
        'error' => $errorMsg,
        'rawdata' => $response
    ];
}

function authorizenet_callback($params)
{
    // Authorize.net uses direct response for SIM
    // This handles the relay response
    
    $transactionId = $_POST['x_trans_id'] ?? '';
    $amount = $_POST['x_amount'] ?? 0;
    $responseCode = $_POST['x_response_code'] ?? '';
    
    if ($responseCode == '1') {
        return [
            'status' => 'success',
            'transid' => $transactionId,
            'amount' => $amount,
            'rawdata' => json_encode($_POST)
        ];
    }
    
    return [
        'status' => 'declined',
        'rawdata' => json_encode($_POST)
    ];
}

function authorizenet_refund($params)
{
    $apiLogin = $params['api_login'];
    $transactionKey = $params['transaction_key'];
    $sandbox = $params['sandbox'];
    
    $baseUrl = $sandbox 
        ? 'https://apitest.authorize.net/xml/v1/request.api' 
        : 'https://api.authorize.net/xml/v1/request.api';
    
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    $cardNumber = $params['cardnum'] ?? 'XXXX';
    
    $payload = [
        'createTransactionRequest' => [
            'merchantAuthentication' => [
                'name' => $apiLogin,
                'transactionKey' => $transactionKey
            ],
            'transactionRequest' => [
                'transactionType' => 'refundTransaction',
                'amount' => number_format($amount, 2, '.', ''),
                'payment' => [
                    'creditCard' => [
                        'cardNumber' => $cardNumber,
                        'expirationDate' => 'XXXX'
                    ]
                ],
                'refTransId' => $transactionId
            ]
        ]
    ];
    
    $ch = curl_init($baseUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['transactionResponse']) && $result['transactionResponse']['responseCode'] == '1') {
        return [
            'status' => 'success',
            'refund_id' => $result['transactionResponse']['transId']
        ];
    }
    
    return [
        'status' => 'failed',
        'error' => 'Refund failed'
    ];
}

function authorizenet_query($params)
{
    $apiLogin = $params['api_login'];
    $transactionKey = $params['transaction_key'];
    $sandbox = $params['sandbox'];
    
    $baseUrl = $sandbox 
        ? 'https://apitest.authorize.net/xml/v1/request.api' 
        : 'https://api.authorize.net/xml/v1/request.api';
    
    $transactionId = $params['reference'];
    
    $payload = [
        'getTransactionDetailsRequest' => [
            'merchantAuthentication' => [
                'name' => $apiLogin,
                'transactionKey' => $transactionKey
            ],
            'transId' => $transactionId
        ]
    ];
    
    $ch = curl_init($baseUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json'
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
    if ($vars['paymentmethod'] == 'authorizenet') {
        logActivity("Authorize.net payment confirmed: " . ($vars['transid'] ?? 'N/A'));
    }
});
```