# WHMCS VNPay Setup Workflow

## Description
Configure VNPay payment gateway for WHMCS (Vietnam).

## Prerequisites
- VNPay merchant account
- Terminal ID and Secure Key
- WHMCS installation

## Steps

### Step 1: VNPay Account Setup
```bash
# Register at: https://vnpayment.vn
# Get merchant credentials:
# - Merchant ID (vnp_TmnCode)
# - Secure Key (vnp_HashSecret)
# - API URL
```

### Step 2: Create VNPay Gateway
```php
<?php
// modules/gateways/vnpay/vnpay.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function vnpay_MetaData()
{
    return [
        'DisplayName' => 'VNPay',
        'APIVersion' => '1.0',
    ];
}

function vnpay_config()
{
    return [
        'FriendlyName' => ['value' => 'VNPay'],
        'vnp_TmnCode' => ['Type' => 'text', 'Label' => 'Merchant Code'],
        'vnp_HashSecret' => ['Type' => 'password', 'Label' => 'Secure Key'],
        'vnp_Url' => ['Type' => 'text', 'Label' => 'API URL'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
    ];
}

function vnpay_link($params)
{
    $config = $params;
    
    $vnp_Url = $params['environment'] === 'sandbox'
        ? 'https://sandbox.vnpayment.vn/paymentv2/vpcpay.html'
        : 'params['vnp_Url'];
    
    $vnp_Returnurl = $params['systemurl'] . '/modules/gateways/callback/vnpay.php';
    
    // Create payment data
    $startTime = date('YmdHis');
    $expire = date('YmdHis', strtotime('+15 minutes'));
    
    $vnp_TxnRef = $params['invoiceid'] . '_' . time();
    $vnp_Amount = (int)($params['amount'] * 100); // VND in cents
    $vnp_Locale = 'vn';
    $vnp_BankCode = '';
    
    $inputData = [
        'vnp_Version' => '2.1.0',
        'vnp_Command' => 'pay',
        'vnp_TmnCode' => $params['vnp_TmnCode'],
        'vnp_Amount' => $vnp_Amount,
        'vnp_CreateDate' => $startTime,
        'vnp_CurrCode' => 'VND',
        'vnp_ExpireDate' => $expire,
        'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'],
        'vnp_Locale' => $vnp_Locale,
        'vnp_OrderInfo' => 'Thanh toan hoa don #' . $params['invoiceid'],
        'vnp_OrderType' => 'billpayment',
        'vnp_ReturnUrl' => $vnp_Returnurl,
        'vnp_TxnRef' => $vnp_TxnRef,
    ];
    
    ksort($inputData);
    $query = [];
    $i = 0;
    $hashData = '';
    
    foreach ($inputData as $key => $value) {
        if ($i == 1) {
            $hashData .= '&' . urlencode($key) . '=' . urlencode($value);
        } else {
            $hashData .= urlencode($key) . '=' . urlencode($value);
            $i = 1;
        }
        $query[] = urlencode($key) . '=' . urlencode($value);
    }
    
    $vnp_SecureHash = hash_hmac('sha512', $hashData, $params['vnp_HashSecret']);
    $vnp_Url .= '?' . implode('&', $query) . '&vnp_SecureHash=' . $vnp_SecureHash;
    
    // Store transaction reference
    $_SESSION['vnpay_txnref'] = $vnp_TxnRef;
    
    return '<a href="' . $vnp_Url . '" class="btn btn-vnpay">
        Thanh toan qua VNPay
    </a>';
}
```

### Step 3: Create Callback Handler
```php
<?php
// modules/gateways/callback/vnpay.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('vnpay');

// Get response
$vnp_ResponseCode = $_GET['vnp_ResponseCode'] ?? '';
$vnp_TxnRef = $_GET['vnp_TxnRef'] ?? '';
$vnp_Amount = $_GET['vnp_Amount'] ?? 0;
$vnp_SecureHash = $_GET['vnp_SecureHash'] ?? '';

// Extract invoice ID
$invoiceId = (int)explode('_', $vnp_TxnRef)[0];

// Verify checksum
if ($vnp_ResponseCode === '00') {
    // Payment successful
    addInvoicePayment($invoiceId, $vnp_TxnRef, $vnp_Amount / 100, 0, 'vnpay');
    logTransaction('vnpay', $_GET, 'Successful');
} else {
    // Payment failed
    logTransaction('vnpay', $_GET, 'Failed: ' . $vnp_ResponseCode);
}

// Redirect to invoice
header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $invoiceId);
```

### Step 4: Test Integration
```bash
# Configure in test mode
# Process test payment
# Verify callback received
# Check transaction logging
```

## VNPay Response Codes
| Code | Description |
|------|-------------|
| 00 | Transaction successful |
| 07 | Suspicious transaction |
| 09 | Bank declining |
| 10 | Card expired |
| 11 | Insufficient funds |
| 24 | Card not supported |

## Tags
- vnpay
- payment
- vietnam
- gateway