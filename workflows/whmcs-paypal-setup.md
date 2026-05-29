# WHMCS PayPal Setup Workflow

## Description
Configure PayPal payment gateway for WHMCS.

## Prerequisites
- PayPal Business account
- API credentials
- SSL certificate

## Steps

### Step 1: Get PayPal API Credentials
```bash
# 1. Go to PayPal Dashboard
# 2. Settings > Account Settings > API Access
# 3. Choose NVP/SOAP or REST API
# 4. Generate API credentials
```

### Step 2: Configure PayPal Gateway
```php
<?php
// modules/gateways/paypal_com/paypal_com.php

function paypal_com_MetaData()
{
    return [
        'DisplayName' => 'PayPal',
        'APIVersion' => '1.1',
    ];
}

function paypal_com_config()
{
    return [
        'FriendlyName' => ['value' => 'PayPal'],
        'APIUsername' => ['Type' => 'text', 'Label' => 'API Username'],
        'APIPassword' => ['Type' => 'password', 'Label' => 'API Password'],
        'APISignature' => ['Type' => 'password', 'Label' => 'API Signature'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
    ];
}

function paypal_com_link($params)
{
    $environment = $params['environment'] === 'sandbox' 
        ? 'sandbox' 
        : 'live';
    
    $baseUrl = $environment === 'sandbox'
        ? 'https://api-3t.sandbox.paypal.com/nvp'
        : 'https://api-3t.paypal.com/nvp';
    
    $returnUrl = $params['returnurl'];
    $cancelUrl = $params['returnurl'] . '&cancel=true';
    
    $data = [
        'USER' => $params['APIUsername'],
        'PWD' => $params['APIPassword'],
        'SIGNATURE' => $params['APISignature'],
        'VERSION' => '124',
        'METHOD' => 'SetExpressCheckout',
        'PAYMENTREQUEST_0_AMT' => $params['amount'],
        'PAYMENTREQUEST_0_CURRENCYCODE' => $params['currency'],
        'PAYMENTREQUEST_0_INVNUM' => $params['invoiceid'],
        'RETURNURL' => $returnUrl,
        'CANCELURL' => $cancelUrl,
        'PAYMENTREQUEST_0_PAYMENTACTION' => 'Sale',
    ];
    
    $ch = curl_init($baseUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query($data),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    
    $response = curl_exec($ch);
    parse_str($response, $result);
    curl_close($ch);
    
    if ($result['ACK'] === 'Success') {
        $redirectUrl = $environment === 'sandbox'
            ? 'https://www.sandbox.paypal.com/checkoutnow?token=' . $result['TOKEN']
            : 'https://www.paypal.com/checkoutnow?token=' . $result['TOKEN'];
        
        return '<a href="' . $redirectUrl . '" class="btn btn-paypal">
            Pay with PayPal
        </a>';
    }
    
    return 'PayPal payment error';
}
```

### Step 3: Create Callback Handler
```php
<?php
// modules/gateways/callback/paypal_com.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('paypal_com');

// Get PayPal response
$token = $_GET['token'];
$payerId = $_GET['PayerID'];

// Complete payment
$data = [
    'USER' => $gateway['APIUsername'],
    'PWD' => $gateway['APIPassword'],
    'SIGNATURE' => $gateway['APISignature'],
    'VERSION' => '124',
    'METHOD' => 'DoExpressCheckoutPayment',
    'TOKEN' => $token,
    'PAYERID' => $payerId,
    'PAYMENTREQUEST_0_AMT' => $_GET['amount'],
    'PAYMENTREQUEST_0_CURRENCYCODE' => $_GET['currency'],
    'PAYMENTREQUEST_0_PAYMENTACTION' => 'Sale',
];

// Process payment...

// Redirect back to WHMCS
header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $invoiceId);
```

## PayPal Configuration Steps
1. Get API credentials from PayPal
2. Configure gateway in WHMCS
3. Set IPN URL if needed
4. Test with sandbox
5. Switch to production

## Tags
- paypal
- payment
- gateway
- express-checkout