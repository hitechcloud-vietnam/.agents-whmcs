# WHMCS MoMo Gateway Module - DEVKIT

## Module Information
- **Name**: MoMo Payment Gateway
- **Version**: 1.0.0
- **Type**: Gateway Module
- **Description**: MoMo e-wallet payment gateway for Vietnam

## Installation
1. Copy to `/modules/gateways/momo/`
2. Activate via WHMCS Admin > Configuration > Payment Gateways

## momo.php
```php
<?php
/**
 * WHMCS MoMo Payment Gateway
 * 
 * @package WHMCS
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Gateway configuration
 */
function momo_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'MoMo'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'MoMo e-wallet Payment Gateway'
        ],
        'momo_endpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type' => 'text',
            'Size' => '100',
            'Default' => 'https://test-payment.momo.vn/v2/gateway/api/create',
            'Description' => 'MoMo API endpoint'
        ],
        'momo_partner_code' => [
            'FriendlyName' => 'Partner Code',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'MoMo partner code'
        ],
        'momo_access_key' => [
            'FriendlyName' => 'Access Key',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'MoMo access key'
        ],
        'momo_secret_key' => [
            'FriendlyName' => 'Secret Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'MoMo secret key'
        ],
        'momo_store_id' => [
            'FriendlyName' => 'Store ID',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'MoMo store ID'
        ],
        'momo_public_key' => [
            'FriendlyName' => 'Public Key',
            'Type' => 'textarea',
            'Description' => 'MoMo public key for encryption'
        ],
        'momo_ipn_url' => [
            'FriendlyName' => 'IPN URL',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Instant Payment Notification URL'
        ]
    ];
}

/**
 * Capture payment
 */
function momo_capture($params)
{
    $invoiceId = $params['invoiceid'];
    $amount = $params['amount'];
    $currency = $params['currency'];
    
    $endpoint = $params['momo_endpoint'];
    $partnerCode = $params['momo_partner_code'];
    $accessKey = $params['momo_access_key'];
    $secretKey = $params['momo_secret_key'];
    $storeId = $params['momo_store_id'];
    
    $orderId = $partnerCode . date('YmdHis') . $invoiceId;
    $requestId = $partnerCode . date('YmdHis');
    
    $requestData = [
        'partnerCode' => $partnerCode,
        'partnerName' => 'WHMCS',
        'storeId' => $storeId,
        'requestId' => $requestId,
        'amount' => (string)round($amount),
        'orderId' => $orderId,
        'orderInfo' => 'Invoice #' . $invoiceId,
        'redirectUrl' => $params['returnurl'],
        'ipnUrl' => $params['momo_ipn_url'],
        'lang' => 'vi',
        'extraData' => json_encode(['invoice_id' => $invoiceId])
    ];
    
    // Create signature
    $rawSignature = "accessKey=" . $accessKey . "&amount=" . $requestData['amount'] . 
                    "&extraData=" . $requestData['extraData'] . "&ipnUrl=" . $requestData['ipnUrl'] . 
                    "&orderId=" . $orderId . "&orderInfo=" . $requestData['orderInfo'] . 
                    "&partnerCode=" . $partnerCode . "&redirectUrl=" . $requestData['redirectUrl'] . 
                    "&requestId=" . $requestId;
    
    $signature = hash_hmac('sha256', $rawSignature, $secretKey);
    $requestData['signature'] = $signature;
    
    // Send request to MoMo
    $ch = curl_init($endpoint);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($requestData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['payUrl']) && $result['payUrl']) {
        return [
            'status' => 'pending',
            'declined' => false,
            'pending' => true,
            'rawsuccess' => true,
            'redirecturl' => $result['payUrl'],
            'reference' => $orderId
        ];
    } else {
        return [
            'status' => 'failed',
            'rawdata' => $result
        ];
    }
}

/**
 * Validate IPN callback
 */
function momo_callback($params)
{
    $secretKey = $params['momo_secret_key'];
    
    // Get response data
    $data = json_decode(file_get_contents('php://input'), true);
    
    if (empty($data)) {
        return ['status' => 'error', 'rawdata' => 'No data received'];
    }
    
    // Verify signature
    $rawSignature = "amount=" . ($data['amount'] ?? '') . 
                    "&extraData=" . ($data['extraData'] ?? '') . 
                    "&message=" . ($data['message'] ?? '') . 
                    "&orderId=" . ($data['orderId'] ?? '') . 
                    "&partnerCode=" . ($data['partnerCode'] ?? '') . 
                    "&payType=" . ($data['payType'] ?? '') . 
                    "&requestId=" . ($data['requestId'] ?? '') . 
                    "&responseTime=" . ($data['responseTime'] ?? '') . 
                    "&resultCode=" . ($data['resultCode'] ?? '');
    
    $signature = hash_hmac('sha256', $rawSignature, $secretKey);
    
    if ($signature !== ($data['signature'] ?? '')) {
        return ['status' => 'error', 'rawdata' => 'Invalid signature'];
    }
    
    // Process result
    $resultCode = $data['resultCode'] ?? -1;
    
    if ($resultCode == 0) {
        $extraData = json_decode($data['extraData'] ?? '{}', true);
        $invoiceId = $extraData['invoice_id'] ?? 0;
        
        return [
            'status' => 'success',
            'transid' => $data['transId'] ?? '',
            'amount' => $data['amount'] ?? 0,
            'rawdata' => json_encode($data)
        ];
    } else {
        return [
            'status' => 'declined',
            'rawdata' => json_encode($data)
        ];
    }
}

/**
 * Refund payment
 */
function momo_refund($params)
{
    $transactionId = $params['transactionId'];
    $amount = $params['amount'];
    
    $endpoint = str_replace('/create', '/refund', $params['momo_endpoint']);
    $partnerCode = $params['momo_partner_code'];
    $accessKey = $params['momo_access_key'];
    $secretKey = $params['momo_secret_key'];
    
    $requestId = $partnerCode . date('YmdHis');
    
    $refundData = [
        'partnerCode' => $partnerCode,
        'partnerName' => 'WHMCS',
        'requestId' => $requestId,
        'orderId' => $transactionId,
        'amount' => (string)round($amount),
        'transId' => $transactionId,
        'lang' => 'vi'
    ];
    
    // Create signature
    $rawSignature = "accessKey=" . $accessKey . "&amount=" . $refundData['amount'] . 
                    "&orderId=" . $refundData['orderId'] . "&partnerCode=" . $partnerCode . 
                    "&requestId=" . $requestId . "&transId=" . $transactionId;
    
    $signature = hash_hmac('sha256', $rawSignature, $secretKey);
    $refundData['signature'] = $signature;
    
    // Send refund request
    $ch = curl_init($endpoint);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($refundData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (($result['resultCode'] ?? -1) == 0) {
        return ['status' => 'success', 'refund_id' => $result['transId'] ?? ''];
    } else {
        return ['status' => 'failed', 'error' => $result['message'] ?? 'Refund failed'];
    }
}

/**
 * Query transaction status
 */
function momo_query($params)
{
    $transactionId = $params['reference'];
    
    $endpoint = str_replace('/create', '/query', $params['momo_endpoint']);
    $partnerCode = $params['momo_partner_code'];
    $accessKey = $params['momo_access_key'];
    $secretKey = $params['momo_secret_key'];
    
    $requestId = $partnerCode . date('YmdHis');
    
    $queryData = [
        'partnerCode' => $partnerCode,
        'requestId' => $requestId,
        'orderId' => $transactionId,
        'lang' => 'vi'
    ];
    
    $rawSignature = "accessKey=" . $accessKey . "&orderId=" . $transactionId . 
                    "&partnerCode=" . $partnerCode . "&requestId=" . $requestId;
    
    $signature = hash_hmac('sha256', $rawSignature, $secretKey);
    $queryData['signature'] = $signature;
    
    $ch = curl_init($endpoint);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($queryData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
    
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
    if ($vars['paymentmethod'] == 'momo') {
        logActivity("MoMo payment confirmed for Invoice #" . $vars['invoiceid']);
    }
});
```