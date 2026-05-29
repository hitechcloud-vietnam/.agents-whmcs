# WHMCS MoMo Setup Workflow

## Description
Configure MoMo payment gateway for WHMCS (Vietnam).

## Prerequisites
- MoMo merchant account
- Partner Code and Secret Key
- WHMCS installation

## Steps

### Step 1: MoMo Account Setup
```bash
# Register at: https://momosv2.apigateway.shop/
# Get merchant credentials:
# - Partner Code
# - Access Key
# - Secret Key
# - API URL
```

### Step 2: Create MoMo Gateway
```php
<?php
// modules/gateways/momo/momo.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function momo_MetaData()
{
    return [
        'DisplayName' => 'MoMo',
        'APIVersion' => '1.0',
    ];
}

function momo_config()
{
    return [
        'FriendlyName' => ['value' => 'MoMo'],
        'partnerCode' => ['Type' => 'text', 'Label' => 'Partner Code'],
        'accessKey' => ['Type' => 'text', 'Label' => 'Access Key'],
        'secretKey' => ['Type' => 'password', 'Label' => 'Secret Key'],
        'momoEndpoint' => ['Type' => 'text', 'Label' => 'API URL'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
    ];
}

function momo_link($params)
{
    $endpoint = $params['environment'] === 'sandbox'
        ? 'https://test.payment.momo.vn/gw_payment/transactionProcessor'
        : $params['momoEndpoint'];
    
    $orderId = $params['invoiceid'] . '_' . time();
    $amount = (int)$params['amount'];
    $orderInfo = 'Thanh toan hoa don #' . $params['invoiceid'];
    $returnUrl = $params['systemurl'] . '/modules/gateways/callback/momo.php';
    $notifyUrl = $params['systemurl'] . '/modules/gateways/callback/momo_ipn.php';
    
    $requestId = time() . '';
    $requestType = 'captureWallet';
    
    $rawData = 'partnerCode=' . $params['partnerCode'] .
        '&accessKey=' . $params['accessKey'] .
        '&requestId=' . $requestId .
        '&amount=' . $amount .
        '&orderId=' . $orderId .
        '&orderInfo=' . $orderInfo .
        '&returnUrl=' . $returnUrl .
        '&notifyUrl=' . $notifyUrl .
        '&extraData=';
    
    $signature = hash_hmac('sha256', $rawData, $params['secretKey']);
    
    $data = [
        'partnerCode' => $params['partnerCode'],
        'accessKey' => $params['accessKey'],
        'requestId' => $requestId,
        'amount' => $amount,
        'orderId' => $orderId,
        'orderInfo' => $orderInfo,
        'returnUrl' => $returnUrl,
        'notifyUrl' => $notifyUrl,
        'extraData' => '',
        'requestType' => $requestType,
        'signature' => $signature,
    ];
    
    $ch = curl_init($endpoint);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    ]);
    
    $response = curl_exec($ch);
    $result = json_decode($response, true);
    curl_close($ch);
    
    if (isset($result['payUrl'])) {
        return '<a href="' . $result['payUrl'] . '" class="btn btn-momo">
            Thanh toan qua MoMo
        </a>';
    }
    
    return 'MoMo payment error';
}
```

### Step 3: Create Callback Handler
```php
<?php
// modules/gateways/callback/momo.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('momo');

// Get response
$partnerCode = $_GET['partnerCode'] ?? '';
$orderId = $_GET['orderId'] ?? '';
$requestId = $_GET['requestId'] ?? '';
$amount = $_GET['amount'] ?? 0;
$transId = $_GET['transId'] ?? '';
$resultCode = $_GET['resultCode'] ?? '';

// Extract invoice ID
$invoiceId = (int)explode('_', $orderId)[0];

if ($resultCode == '0') {
    // Payment successful
    addInvoicePayment($invoiceId, $transId, $amount, 0, 'momo');
    logTransaction('momo', $_GET, 'Successful');
} else {
    // Payment failed
    logTransaction('momo', $_GET, 'Failed: ' . $resultCode);
}

// Redirect to invoice
header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $invoiceId);
```

### Step 4: IPN Handler (Optional)
```php
<?php
// modules/gateways/callback/momo_ipn.php

// IPN handling for MoMo
// Similar to callback but for background processing
```

## MoMo Result Codes
| Code | Description |
|------|-------------|
| 0 | Transaction successful |
| 1001 | Invalid signature |
| 1002 | Amount too large |
| 1003 | Invalid merchant |
| 1006 | User canceled |

## Tags
- momo
- payment
- vietnam
- gateway