# WHMCS Payment Gateway Developer Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

Payment gateways allow WHMCS to process payments through various payment providers. This guide covers the complete development process for creating custom payment gateway modules.

---

## Gateway Module Structure

```
modules/gateways/
  your_gateway/
    callback.php           # Payment callback handler
    your_gateway.php      # Main module file
    logo.png              # Gateway logo (80x80px)
```

## Main Gateway File

```php
<?php
/**
 * Gateway Module Definition
 * 
 * @package WHMCS
 * @subpackage GatewayModule
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module meta data
 * 
 * @return array Module metadata
 */
function your_gateway_MetaData()
{
    return [
        'DisplayName'      => 'Your Gateway Name',
        'APIVersion'       => '1.1',
        'disable_gateway'  => false,
        'gatewayFields'    => [
            'api_key' => [
                'Type'        => 'password',
                'FriendlyName'=> 'API Key',
                'Size'        => '40',
            ],
            'secret_key' => [
                'Type'        => 'password',
                'FriendlyName'=> 'Secret Key',
                'Size'        => '40',
            ],
            'test_mode' => [
                'Type'        => 'yesno',
                'FriendlyName'=> 'Test Mode',
            ],
        ],
    ];
}

/**
 * Define configuration fields
 * 
 * @return array Configuration fields
 */
function your_gateway_config()
{
    return [
        'FriendlyName' => [
            'Type'    => 'System',
            'Value'   => 'Your Gateway Name',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type'         => 'password',
            'Size'         => '40',
        ],
        'secret_key' => [
            'FriendlyName' => 'Secret Key',
            'Type'         => 'password',
            'Size'         => '40',
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type'         => 'yesno',
        ],
    ];
}

/**
 * Link to external payment page
 * 
 * @param array $params Payment parameters
 * @return string HTML form or redirect URL
 */
function your_gateway_link($params)
{
    // Get configuration
    $apiKey      = $params['apiKey'];
    $secretKey   = $params['secretKey'];
    $testMode    = $params['test_mode'];
    
    // Get invoice details
    $invoiceId   = $params['invoiceid'];
    $description = $params['description'];
    $amount      = $params['amount'];
    $currency    = $params['currency'];
    
    // Build API endpoint
    $endpoint = $testMode 
        ? 'https://sandbox.yourgateway.com/checkout' 
        : 'https://api.yourgateway.com/checkout';
    
    // Create checkout session
    $sessionData = [
        'order_id'    => $invoiceId,
        'amount'      => $amount,
        'currency'    => $currency,
        'description' => $description,
        'return_url'  => $params['return_url'],
        ' cancel_url' => $params['cancel_url'],
        'notify_url'  => $params['systemurl'] . 'modules/gateways/your_gateway/callback.php',
    ];
    
    // Generate signature
    $signature = hash_hmac('sha256', json_encode($sessionData), $secretKey);
    
    // Return checkout form
    $form = '<form action="' . $endpoint . '" method="POST">';
    $form .= '<input type="hidden" name="data" value="' . base64_encode(json_encode($sessionData)) . '">';
    $form .= '<input type="hidden" name="signature" value="' . $signature . '">';
    $form .= '<input type="submit" value="' . $params['lang pay_now'] . '">';
    $form .= '</form>';
    
    return $form;
}
```

## Callback Handler

```php
<?php
/**
 * Payment Gateway Callback Handler
 * 
 * @package WHMCS
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Verify gateway activation
use WHMCS\Module\Gateway;

if (!function_exists('your_gateway_config')) {
    require_once __DIR__ . '/your_gateway.php';
}

// Retrieve POST data
$payload    = $_POST['data'] ?? '';
$signature  = $_POST['signature'] ?? '';

// Decode payload
$transactionData = json_decode(base64_decode($payload), true);
$invoiceId = $transactionData['order_id'] ?? 0;

// Verify signature
$secretKey   = Gateway::getAvailablePaymentGateways()['your_gateway']['secret_key'] ?? '';
$expectedSig = hash_hmac('sha256', $payload, $secretKey);

if (!hash_equals($expectedSig, $signature)) {
    logTransaction('your_gateway', $_POST, 'Invalid Signature');
    http_response_code(403);
    exit;
}

// Process transaction
$transactionId = $transactionData['transaction_id'] ?? '';
$amount        = $transactionData['amount'] ?? 0;
$status        = $transactionData['status'] ?? '';

// Verify invoice exists
$invoiceId = checkCbInvoiceID($invoiceId, 'your_gateway');

// Check payment status
if ($status === 'completed' || $status === 'approved') {
    // Check if already processed
    checkCbTransID($transactionId);
    
    // Add payment
    $amount = format_as_currency($amount);
    addInvoicePayment($invoiceId, $transactionId, $amount, '', 'your_gateway');
    
    logTransaction('your_gateway', $transactionData, 'Successful');
    
    // Redirect to thank you page
    header('Location: ' . $params['return_url'] . '&paymentsuccess=true');
    exit;
} elseif ($status === 'pending') {
    logTransaction('your_gateway', $transactionData, 'Pending');
    header('Location: ' . $params['return_url'] . '&paymentpending=true');
    exit;
} else {
    logTransaction('your_gateway', $transactionData, 'Failed');
    header('Location: ' . $params['return_url'] . '&paymentfailed=true');
    exit;
}
```

## Refund Handler

```php
/**
 * Process refunds
 * 
 * @param array $params Refund parameters
 * @return array Refund result
 */
function your_gateway_refund($params)
{
    $apiKey       = $params['apiKey'];
    $secretKey    = $params['secretKey'];
    $testMode     = $params['test_mode'];
    $refundId     = $params['refund_id'];
    $transactionId = $params['transaction_id'];
    $amount       = $params['amount'];
    $invoiceId    = $params['invoiceid'];
    
    // Build refund request
    $endpoint = $testMode
        ? 'https://sandbox.yourgateway.com/refunds'
        : 'https://api.yourgateway.com/refunds';
    
    $data = [
        'original_transaction' => $transactionId,
        'amount'                => $amount,
        'reason'                => 'Customer request',
    ];
    
    // Execute refund via API
    $ch = curl_init($endpoint);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($data),
        CURLOPT_HTTPHEADER     => [
            'Authorization: Bearer ' . $apiKey,
            'Content-Type: application/json',
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 30,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($httpCode === 200 && ($result['status'] ?? '') === 'refunded') {
        return [
            'status'  => 'success',
            'refund_id' => $result['refund_id'],
            'rawdata' => $result,
        ];
    }
    
    return [
        'status'  => 'failed',
        'rawdata' => $result,
        'message' => $result['error']['message'] ?? 'Refund failed',
    ];
}
```

## Three-Step Credit Card Processing

```php
/**
 * Capture step (for 3-step processing)
 * 
 * @param array $params Capture parameters
 * @return array Capture result
 */
function your_gateway_capture($params)
{
    $apiKey      = $params['apiKey'];
    $secretKey   = $params['secretKey'];
    $testMode    = $params['test_mode'];
    $transId     = $params['trans_id'];
    $amount      = $params['amount'];
    
    // Build capture request
    $endpoint = $testMode
        ? 'https://sandbox.yourgateway.com/capture'
        : 'https://api.yourgateway.com/capture';
    
    // Authenticate
    $auth = base64_encode($apiKey . ':' . $secretKey);
    
    $data = [
        'transaction_id' => $transId,
        'amount'          => $amount,
    ];
    
    $ch = curl_init($endpoint);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => http_build_query($data),
        CURLOPT_HTTPHEADER     => [
            'Authorization: Basic ' . $auth,
            'Content-Type: application/x-www-form-urlencoded',
        ],
        CURLOPT_RETURNTRANSFER => true,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, curl_info(CURLINFO_HTTP_CODE));
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($httpCode === 200) {
        return [
            'status'      => 'success',
            'trans_id'    => $result['transaction_id'],
            'capture_id'  => $result['capture_id'],
            'amount'      => $result['amount'],
        ];
    }
    
    return [
        'status'  => 'failed',
        'message' => $result['error']['message'] ?? 'Capture failed',
    ];
}
```

## Remote Input Gateway

```php
/**
 * Remote input configuration for credit card processing
 * 
 * @param array $params Configuration parameters
 * @return array Remote input fields
 */
function your_gateway_remote_input($params)
{
    return [
        'card_num' => [
            'id'          => 'cardNumber',
            'name'        => 'card_number',
            'type'        => 'cardnum',
            'label'       => 'Card Number',
            'placeholder'=> '4111111111111111',
            'required'    => true,
            'autocomplete'=> 'cc-number',
        ],
        'card_cvv' => [
            'id'          => 'cardCvv',
            'name'        => 'card_cvv',
            'type'        => 'cvv',
            'label'       => 'CVV',
            'placeholder'=> '123',
            'required'    => true,
            'autocomplete'=> 'cc-csc',
        ],
        'card_expiry' [
            'id'          => 'cardExpiry',
            'name'        => 'card_expiry',
            'type'        => 'cardexpiry',
            'label'       => 'Expiry Date',
            'placeholder'=> 'MM/YY',
            'required'    => true,
            'autocomplete'=> 'cc-exp',
        ],
    ];
}
```

## Installation Checklist

1. Create gateway directory in `/modules/gateways/`
2. Create main module file with all required functions
3. Implement callback handler for payment notifications
4. Add gateway logo (80x80px, PNG format)
5. Test in sandbox environment
6. Test callback with various scenarios
7. Implement proper error logging
8. Test refund functionality
9. Document configuration requirements

## Testing Checklist

- [ ] Successful payment flow
- [ ] Cancelled payment handling
- [ ] Failed payment handling
- [ ] Pending payment handling
- [ ] Duplicate callback prevention
- [ ] Refund processing
- [ ] Signature verification
- [ ] Webhook security
- [ ] Test mode vs live mode
- [ ] Currency handling
- [ ] Partial refund handling

## Security Best Practices

- Validate all callback signatures using HMAC
- Use constant-time comparison for signatures
- Verify transaction amounts before updating invoices
- Implement idempotency checks for callbacks
- Store API credentials securely
- Use SSL/TLS for all API communications
- Log all transaction attempts
- Implement rate limiting for callbacks

---

## Related Skills and Workflows

- `module-security-standards` - Gateway security requirements
- `module-testing-strategies` - Testing approaches for payment modules
- `module-error-handling-guide` - Error handling patterns
- `module-logging-guide` - Transaction logging best practices
- `payment-gateway-developer-guide` - Gateway module creation
- `webhook-events-reference` - Webhook event handling

## Resources

- [WHMCS Module Development Docs](https://developers.whmcs.com/payment-gateways/)
- [Gateway Hook Reference](/module-hook-reference)
