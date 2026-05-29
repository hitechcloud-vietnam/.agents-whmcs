# WHMCS Callback Processing Workflow

## Description
Process payment gateway callbacks and IPN (Instant Payment Notifications).

## Prerequisites
- Payment gateway with callback support
- Correctly configured callback URL

## Steps

### Step 1: Create Callback Endpoint
```php
<?php
// modules/gateways/callback/payment_gateway.php

define('WHMCS', true);
require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/../../../includes/gatewayfunctions.php';

$gateway = getGatewayVariables('payment_gateway');

if (!$gateway['type']) {
    die('Gateway not active');
}
```

### Step 2: Validate Callback
```php
<?php
function validateCallback($gateway, $data)
{
    // IP whitelist check
    $allowedIPs = ['1.2.3.4', '5.6.7.8']; // From gateway documentation
    $clientIP = $_SERVER['REMOTE_ADDR'];
    
    if (!in_array($clientIP, $allowedIPs)) {
        logTransaction($gateway['paymentmethod'], $data, 'Invalid IP: ' . $clientIP);
        http_response_code(403);
        die('Unauthorized');
    }
    
    // Signature verification
    $expectedSig = calculateSignature($data, $gateway['secret']);
    if ($data['signature'] !== $expectedSig) {
        logTransaction($gateway['paymentmethod'], $data, 'Invalid signature');
        http_response_code(401);
        die('Invalid signature');
    }
    
    return true;
}
```

### Step 3: Process Payment Callback
```php
<?php
function processPaymentCallback($gateway, $data)
{
    // Extract key data
    $invoiceId = $data['invoice_id'];
    $amount = $data['amount'];
    $transactionId = $data['transaction_id'];
    $status = $data['status'];
    
    // Validate amount
    $invoice = Capsule::table('tblinvoices')->find($invoiceId);
    if (!$invoice) {
        logTransaction($gateway['paymentmethod'], $data, 'Invoice not found');
        return false;
    }
    
    if (abs($amount - $invoice->total) > 0.01) {
        logTransaction($gateway['paymentmethod'], $data, 'Amount mismatch');
        return false;
    }
    
    // Check for duplicate
    $existing = Capsule::table('tblaccounts')
        ->where('transid', $transactionId)
        ->first();
    
    if ($existing) {
        logTransaction($gateway['paymentmethod'], $data, 'Duplicate transaction');
        return true;
    }
    
    // Process based on status
    if ($status === 'completed') {
        addInvoicePayment($invoiceId, $transactionId, $amount, 0, 'payment_gateway');
        logTransaction($gateway['paymentmethod'], $data, 'Success');
    } elseif ($status === 'pending') {
        logTransaction($gateway['paymentmethod'], $data, 'Pending');
    } else {
        logTransaction($gateway['paymentmethod'], $data, 'Failed: ' . $status);
    }
    
    return true;
}
```

### Step 4: Handle Different Gateway Formats

**Format 1: POST Data**
```php
<?php
$postData = $_POST;
processPaymentCallback($gateway, $postData);
```

**Format 2: JSON Body**
```php
<?php
$jsonData = json_decode(file_get_contents('php://input'), true);
processPaymentCallback($gateway, $jsonData);
```

**Format 3: Query Parameters**
```php
<?php
$queryData = $_GET;
processPaymentCallback($gateway, $queryData);
```

### Step 5: Send Response
```php
<?php
// Always send a response
if ($success) {
    http_response_code(200);
    echo 'OK';
} else {
    http_response_code(400);
    echo 'Error: ' . $errorMessage;
}
```

### Step 6: Test Callbacks
```php
<?php
// Test script
// curl -X POST -d "invoice_id=1&amount=100&status=completed" \
//    https://yourwhmcs.com/modules/gateways/callback/gateway.php
```

## Callback Testing
```bash
# Test with curl
curl -X POST https://yourdomain.com/modules/gateways/callback/gateway.php \
  -d "invoice_id=123" \
  -d "amount=99.99" \
  -d "status=completed" \
  -d "transaction_id=TEST123"

# Check logs
tail -f /var/www/whmcs/logs/gatewaylog.log
```

## Common Issues
- Missing IP whitelisting
- Incorrect signature verification
- Duplicate transaction handling
- Amount validation
- Currency conversion

## Tags
- callback
- ipn
- webhook
- integration