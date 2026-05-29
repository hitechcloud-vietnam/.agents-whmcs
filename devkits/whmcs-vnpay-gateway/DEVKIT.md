# WHMCS VNPay Gateway Module - DEVKIT

## Module Information
- **Name**: VNPay Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: Vietnam payment gateway integration for WHMCS

## Installation
1. Copy to `/modules/gateways/vnpay/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## vnpay.php
```php
<?php
/**
 * WHMCS VNPay Payment Gateway
 * 
 * @package WHMCS
 * @copyright Copyright (c) 2024
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway configuration
 */
function vnpay_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'VNPay'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'VNPay Payment Gateway for Vietnam'
        ],
        'vnp_TmnCode' => [
            'FriendlyName' => 'Terminal ID (TmnCode)',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'VNPay Terminal ID from merchant portal'
        ],
        'vnp_HashSecret' => [
            'FriendlyName' => 'Hash Secret',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'VNPay secure hash key'
        ],
        'vnp_Url' => [
            'FriendlyName' => 'API URL',
            'Type' => 'text',
            'Size' => '100',
            'Default' => 'https://sandbox.vnpayment.vn/paymentv2/vpcpay.html',
            'Description' => 'VNPay API endpoint URL'
        ],
        'vnp_ReturnUrl' => [
            'FriendlyName' => 'Return URL',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Callback URL after payment'
        ],
        'vnp_TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox mode'
        ]
    ];
}

/**
 * Capture payment
 */
function vnpay_capture($params)
{
    $orderId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    
    $vnp_TmnCode = $params['vnp_TmnCode'];
    $vnp_HashSecret = $params['vnp_HashSecret'];
    $vnp_Url = $params['vnp_Url'];
    $vnp_ReturnUrl = $params['vnp_ReturnUrl'];
    
    // Create payment URL
    $vnp_Params = [
        'vnp_Version' => '2.1.0',
        'vnp_Command' => 'pay',
        'vnp_TmnCode' => $vnp_TmnCode,
        'vnp_Locale' => 'vn',
        'vnp_CurrCode' => $currency,
        'vnp_TxnRef' => $orderId . '_' . time(),
        'vnp_OrderInfo' => 'Payment for Invoice #' . $orderId,
        'vnp_OrderType' => 'billpayment',
        'vnp_Amount' => $amount * 100, // Convert to VND cents
        'vnp_ReturnUrl' => $vnp_ReturnUrl,
        'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1',
        'vnp_CreateDate' => date('YmdHis')
    ];
    
    // Sort parameters and create hash
    ksort($vnp_Params);
    
    $hashData = '';
    $i = 0;
    foreach ($vnp_Params as $key => $value) {
        if ($i == 1) {
            $hashData .= '&';
        }
        $hashData .= $key . '=' . urlencode($value);
        $i = 1;
    }
    
    $vnp_SecureHash = hash_hmac('sha512', $hashData, $vnp_HashSecret);
    $vnp_Params['vnp_SecureHash'] = $vnp_SecureHash;
    
    // Build redirect URL
    $redirectUrl = $vnp_Url . '?' . http_build_query($vnp_Params);
    
    return [
        'status' => 'pending',
        'declined' => false,
        'pending' => true,
        'rawsuccess' => true,
        'redirecturl' => $redirectUrl,
        'reference' => $vnp_Params['vnp_TxnRef']
    ];
}

/**
 * Validate payment callback
 */
function vnpay_callback($params)
{
    $vnp_SecureHash = $_GET['vnp_SecureHash'] ?? '';
    unset($_GET['vnp_SecureHash']);
    unset($_GET['module']);
    
    // Verify hash
    $vnp_HashSecret = $params['vnp_HashSecret'];
    
    ksort($_GET);
    $hashData = '';
    $i = 0;
    foreach ($_GET as $key => $value) {
        if ($i == 1) {
            $hashData .= '&';
        }
        $hashData .= $key . '=' . urlencode($value);
        $i = 1;
    }
    
    $secureHash = hash_hmac('sha512', $hashData, $vnp_HashSecret);
    
    if ($secureHash === $vnp_SecureHash) {
        $vnp_ResponseCode = $_GET['vnp_ResponseCode'] ?? '';
        
        if ($vnp_ResponseCode == '00') {
            // Payment successful
            $invoiceId = explode('_', $_GET['vnp_TxnRef'])[0];
            $transactionId = $_GET['vnp_TransactionNo'] ?? '';
            
            return [
                'status' => 'success',
                'transid' => $transactionId,
                'amount' => ($_GET['vnp_Amount'] ?? 0) / 100,
                'rawdata' => json_encode($_GET)
            ];
        } else {
            return [
                'status' => 'declined',
                'rawdata' => json_encode($_GET)
            ];
        }
    } else {
        return [
            'status' => 'error',
            'rawdata' => 'Invalid hash signature'
        ];
    }
}

/**
 * Refund payment
 */
function vnpay_refund($params)
{
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    
    $vnp_TmnCode = $params['vnp_TmnCode'];
    $vnp_HashSecret = $params['vnp_HashSecret'];
    
    // Build refund request
    $vnp_Params = [
        'vnp_Version' => '2.1.0',
        'vnp_Command' => 'refund',
        'vnp_TmnCode' => $vnp_TmnCode,
        'vnp_TxnRef' => $transactionId,
        'vnp_Amount' => $amount * 100,
        'vnp_TransactionType' => '02', // Full refund
        'vnp_CreateDate' => date('YmdHis'),
        'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1'
    ];
    
    ksort($vnp_Params);
    
    $hashData = '';
    $i = 0;
    foreach ($vnp_Params as $key => $value) {
        if ($i == 1) {
            $hashData .= '&';
        }
        $hashData .= $key . '=' . urlencode($value);
        $i = 1;
    }
    
    $vnp_SecureHash = hash_hmac('sha512', $hashData, $vnp_HashSecret);
    $vnp_Params['vnp_SecureHash'] = $vnp_SecureHash;
    
    // Call refund API
    $ch = curl_init('https://sandbox.vnpayment.vn/merchant_webapi/api/v1/refund');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($vnp_Params));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/x-www-form-urlencoded']);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($result['vnp_ResponseCode'] == '00') {
        return ['status' => 'success', 'refund_id' => $result['vnp_TransactionNo']];
    } else {
        return ['status' => 'failed', 'error' => $result['vnp_Message'] ?? 'Refund failed'];
    }
}

/**
 * Check transaction status
 */
function vnpay_query($params)
{
    $transactionId = $params['reference'];
    
    $vnp_TmnCode = $params['vnp_TmnCode'];
    $vnp_HashSecret = $params['vnp_HashSecret'];
    
    $vnp_Params = [
        'vnp_Version' => '2.1.0',
        'vnp_Command' => 'querydr',
        'vnp_TmnCode' => $vnp_TmnCode,
        'vnp_TxnRef' => $transactionId,
        'vnp_TransactionDate' => date('YmdHis'),
        'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1',
        'vnp_CreateDate' => date('YmdHis')
    ];
    
    ksort($vnp_Params);
    
    $hashData = '';
    $i = 0;
    foreach ($vnp_Params as $key => $value) {
        if ($i == 1) {
            $hashData .= '&';
        }
        $hashData .= $key . '=' . urlencode($value);
        $i = 1;
    }
    
    $vnp_SecureHash = hash_hmac('sha512', $hashData, $vnp_HashSecret);
    
    $ch = curl_init('https://sandbox.vnpayment.vn/merchant_webapi/api/v1/querydr?' . http_build_query($vnp_Params) . '&vnp_SecureHash=' . $vnp_SecureHash);
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

/**
 * Hook: InvoicePaid - Sync to VNPay records
 */
add_hook('InvoicePaid', 1, function($vars) {
    $gateway = 'vnpay';
    
    if ($vars['paymentmethod'] == $gateway) {
        logActivity("VNPay payment confirmed for Invoice #" . $vars['invoiceid']);
    }
});
```